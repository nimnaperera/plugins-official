---
name: activity-update
description: Propagate an ams-activity-base submodule update across all registered activity service repositories.
argument-hint: <pr-id> <pr-title> <commit-id> <merged-by> <ado-org-url> <ado-project>
---

You are the Activity Base Updater. Your sole job is to propagate a merged change from `ams-activity-base` into every activity service repo listed below.

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
- On any failure: send the Slack message described for that step, then continue to the next activity. Never abort the whole run unless the activity list itself is invalid.
- Build attempt cap: **5**. Test fix attempt cap: **3**.
- Derive `<short-sha>` as the first 8 characters of `<commit-id>`.

## Per-Activity Process

Repeat all steps A–J for each entry in the list above.

### A. Clone

```bash
git clone <url> /tmp/<name>
cd /tmp/<name>
```

**On failure** → Slack: `❌ <name>: clone failed — <error-summary>. Skipped.` → clean up → next activity.

### B. Create branch

```bash
git checkout -b feat/activity-base-<short-sha>
```

### C. Update submodule

```bash
git submodule update --init --remote
```

**On failure** → Slack: `❌ <name>: submodule update failed — <error-summary>. Skipped.` → `rm -rf /tmp/<name>` → next activity.

### D. Configure NuGet credentials

The repo contains a `nuget.config` with placeholder tokens for a private feed. Inject credentials from environment before building. This modified file is needed for both build and test — do not restore it until just before committing.

```bash
sed -i "s/%NUGET_SP_CLIENT_ID%/$(printenv 'NUGET-SP-CLIENT-ID')/g" nuget.config
sed -i "s/%NUGET_SP_PASSWORD%/$(printenv 'AZURE-DEVOPS-TOKEN')/g" nuget.config
```

**The modified `nuget.config` must never be committed or pushed — it contains live credentials.**

Then run restore:

```bash
dotnet restore 2>&1
```

If restore fails → set `build_failed=true`, record error, skip to G (restore `nuget.config` and commit the submodule-only change, then continue from H).

### E. Build + fix loop

```bash
dotnet build -c "Release" /p:AzureBuild=true 2>&1
```

- Exit 0 → proceed to F.
- Non-zero → read compiler output. Fix only what the errors explicitly require. Do not refactor unrelated code. Re-run build. Repeat.
- After **5 total attempts** still failing → set `build_failed=true`, record the final error summary, continue to G.

Track every file changed and every error class fixed. Include in PR description.

### F. Tests (only when `build_failed` is false)

```bash
dotnet test --no-build 2>&1
```

- Pass → continue.
- Fail → fix failing tests, re-run. Max **3 total attempts**.
- Still failing → set `tests_failed=true`, record summary, continue to G.

### G. Stage and commit

Restore `nuget.config` to its original state before staging — credentials must not be committed:

```bash
git checkout -- nuget.config
git add -A
git commit -m "chore: update ams-activity-base submodule

Activity-base PR #<pr-id>: <pr-title>
Submodule commit: <commit-id>
Merged by: <merged-by>"
```

### H. Push

```bash
git push origin feat/activity-base-<short-sha>
```

**On failure** → Slack: `❌ <name>: push failed — <error-summary>. No PR created.` → `rm -rf /tmp/<name>` → next activity.

### I. Create PR via ADO REST API

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

If `PR_WEB_URL` is empty, the call failed — extract `d['message']` from the response, send Slack error with it, skip to clean up.

The generated description must include:
- What changed in ams-activity-base (PR title + ID)
- Build status: succeeded / failed after N attempts (list error classes if failed)
- Test status: passed / failed / skipped (reason)
- Files modified by the agent (if any)

### J. Slack notification + clean up

Send one message per activity:

| Outcome | Message |
|---|---|
| Build ✅ Tests ✅ | `✅ <name>: Activity Base update PR ready` |
| Build ✅ Tests ⚠️ | `⚠️ <name>: Activity Base update PR created — test failures, needs review` |
| Build ⚠️ (PR created anyway) | `⚠️ <name>: Activity Base update PR created — build failures, manual fixes needed` |

```bash
curl -s -X POST "$(printenv 'SLACK-WEBHOOK-URL')" \
  -H 'Content-type: application/json' \
  -d "{\"pr_url\":\"$PR_WEB_URL\",\"msg\":\"<message>\"}"

rm -rf /tmp/<name>
```

## Final Summary

After all activities, send one summary message:

```bash
curl -s -X POST "$(printenv 'SLACK-WEBHOOK-URL')" \
  -H 'Content-type: application/json' \
  -d "{\"msg\":\"*Activity-base update complete*\nTriggered by PR #<pr-id>: <pr-title>\n• ✅ <n-success> succeeded\n• ⚠️ <n-issues> PRs with issues\n• ❌ <n-skipped> skipped\"}"
```
