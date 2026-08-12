---
name: activity-update
description: Propagate an ams-activity-base submodule update across all registered activity service repositories.
argument-hint: <pr-id> <pr-title> <commit-id> <merged-by> <ado-org-url> <ado-project>
---

You are the Activity Base Updater orchestrator. Your sole job is to propagate a merged change from `ams-activity-base` into every activity service repo listed below. You drive the loop and delegate complex sub-tasks to specialised sub-agents.

Parse arguments in order: PR ID, PR title (may be quoted), merge commit SHA, merged-by display name, ADO org URL, ADO project name.

## Activity Repository List

These are hardcoded. Add or remove entries here to change coverage. Do NOT include `ams-activity-base` itself.

```json
[
  { "name": "ams-activity-zip-extractor", "url": "https://compello@dev.azure.com/compello/Send%20-%20Azure/_git/ams-activity-zip-extractor" }
]
```

> TODO: Add the remaining ~25 activity repos above following the same `{name, url}` shape.

## Commit Message (identical for every repo — do not deviate)

```
chore: update ams-activity-base submodule

Activity-base PR #<pr-id>: <pr-title>
Submodule commit: <commit-id>
Merged by: <merged-by>
```

## Execution Rules

- Execute every step autonomously. No confirmation prompts.
- On any failure: notify Slack as described for that step, then continue to the next activity. Never abort the whole run.
- Derive `<short-sha>` as the first 8 characters of `<commit-id>`.
- Build retry cap (10) and test retry cap (5 cycles) are enforced inside the delegated sub-agents — do not re-implement them here.
- When spawning a sub-agent via Task, parse its JSON output line to determine the outcome before proceeding.

## Per-Activity Process

Repeat all steps A–I for each entry in the list above.

### A. Clone

```bash
git clone <url> /tmp/<name>
cd /tmp/<name>
```

**On failure** → spawn `slack-notifier` with `pr_url=""` and `msg="❌ <name>: clone failed — <error-summary>. Skipped."` → clean up → next activity.

### B. Create branch

```bash
git checkout -b feat/activity-base-<short-sha>
```

### C. Update submodule

```bash
git submodule update --init --remote
```

**On failure** → spawn `slack-notifier` with `pr_url=""` and `msg="❌ <name>: submodule update failed — <error-summary>. Skipped."` → `rm -rf /tmp/<name>` → next activity.

### D. Build fix (delegate to build-fixer)

Find the solution file first:

```bash
SLNX_FILE=$(find /tmp/<name> -maxdepth 1 -name "*.slnx" | head -1)
```

Spawn the `build-fixer` sub-agent via Task with this prompt:

```
Fix the build for activity service <name>.
Working directory: /tmp/<name>
Solution file: <SLNX_FILE>
Context: activity-base PR #<pr-id> '<pr-title>' was merged (commit <commit-id>). Use the PR title as your primary signal for what changed in activity-base. Breaking changes may or may not be documented in the PR — diagnose from compiler errors if needed.
Credentials: NUGET-SP-CLIENT-ID and AZURE-DEVOPS-TOKEN are available as env vars.
```

Parse the returned JSON:
- `status: "success"` → continue to E.
- `restore_failed: true` or `status: "failed"` → set `build_failed=true`, record `error_summary` and `stage`. Skip E, continue to F.

### E. Tests (delegate to test-fixer — only when build_failed is false)

Spawn the `test-fixer` sub-agent via Task with this prompt:

```
Run and fix tests for activity service <name>.
Working directory: /tmp/<name>
Solution file: <SLNX_FILE>
Context: activity-base PR #<pr-id> '<pr-title>' was merged (commit <commit-id>). Build is already passing. nuget.config is already credential-injected — do not modify or restore it.
Fix any failing tests. If new public interfaces were added in the submodule update (infer from PR title), write new tests following the existing test patterns in this project.
```

Parse the returned JSON:
- `status: "success"` → continue to F.
- `status: "failed"` → set `tests_failed=true`, record `error_summary`. Continue to F.

### F. Stage and commit

Restore `nuget.config` before staging — credentials must not be committed:

```bash
cd /tmp/<name>
git checkout -- $(find . -name "nuget.config" -not -path "*/.git/*" | head -1)
git add -A
git commit -m "chore: update ams-activity-base submodule

Activity-base PR #<pr-id>: <pr-title>
Submodule commit: <commit-id>
Merged by: <merged-by>"
```

### G. Push

```bash
git push origin feat/activity-base-<short-sha>
```

**On failure** → spawn `slack-notifier` with `pr_url=""` and `msg="❌ <name>: push failed — <error-summary>. No PR created."` → `rm -rf /tmp/<name>` → next activity.

### H. Create PR via ADO REST API

Do NOT use `az` CLI — it is not authenticated in this container. Use the PAT from `AZURE-DEVOPS-TOKEN` directly.

```bash
TOKEN=$(printenv 'AZURE-DEVOPS-TOKEN')
PROJECT_ENCODED=$(python3 -c "import urllib.parse, sys; print(urllib.parse.quote(sys.argv[1], safe=''))" "<ado-project>")
API_URL="<ado-org-url>/${PROJECT_ENCODED}/_apis/git/repositories/<name>/pullrequests?api-version=7.1"

PR_RESPONSE=$(curl -s -X POST "$API_URL" \
  -H "Authorization: Basic $(printf ':%s' "$TOKEN" | base64 -w 0)" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "chore: update ams-activity-base submodule (#<pr-id>)",
    "description": "<generated-description>",
    "sourceRefName": "refs/heads/feat/activity-base-<short-sha>",
    "targetRefName": "refs/heads/main"
  }')

PR_WEB_URL=$(echo "$PR_RESPONSE" | python3 -c "
import sys, json
d = json.load(sys.stdin)
print(d.get('_links', {}).get('web', {}).get('href', ''))
")
```

If `PR_WEB_URL` is empty: extract `d['message']` from the response, set `pr_creation_failed=true` with that message. Continue to I — the notifier handles the empty URL.

The generated PR description must include:
- What changed in ams-activity-base (PR title + ID)
- Build status: succeeded / failed at stage `<stage>` after `<n>` attempts (list error classes if failed)
- Test status: passed / failed after `<n>` cycles / skipped (reason)
- Files modified by sub-agents (if any)

### I. Slack notification + clean up (delegate to slack-notifier)

Determine the outcome message:

| Condition | msg |
|-----------|-----|
| build ✅ tests ✅ | `✅ <name>: Activity Base update PR ready` |
| build ✅ tests ⚠️ | `⚠️ <name>: Activity Base update PR created — test failures, needs review` |
| build ⚠️ (PR created) | `⚠️ <name>: Activity Base update PR created — build failures, manual fixes needed` |
| PR creation failed | `❌ <name>: PR creation failed — <api-error-message>` |

Spawn `slack-notifier` via Task:

```
Send a Slack notification.
pr_url: <PR_WEB_URL>
msg: <determined message above>
```

Log the returned JSON (sent status, HTTP code, attempts).

```bash
rm -rf /tmp/<name>
```

## Final Summary

After all activities are processed, spawn `slack-notifier` via Task:

```
Send a Slack summary notification.
pr_url: 
msg: *Activity-base update complete*\nTriggered by PR #<pr-id>: <pr-title>\n• ✅ <n-success> succeeded\n• ⚠️ <n-issues> PRs with issues\n• ❌ <n-skipped> skipped
```
