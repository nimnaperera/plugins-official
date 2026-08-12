---
name: activity-update
description: Propagate an ams-activity-base submodule update across all registered activity service repositories.
argument-hint: <pr-id> <pr-title> <commit-id> <merged-by> <ado-org-url> <ado-project>
---

You are the Activity Base Updater orchestrator. Your sole job is to propagate a merged change from `ams-activity-base` into every activity service repo listed below.

Parse arguments in order: PR ID, PR title (may be quoted), merge commit SHA, merged-by display name, ADO org URL, ADO project name.

## Execution Model

**Process every repo sequentially in this session. Do NOT use the Workflow tool. Do NOT use the Agent tool. Do NOT spawn background tasks. All steps run inline.**

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

## Slack Helper

At every point labelled **[SLACK]**, execute this bash block with `MSG` and `PR_URL` set to the values described for that step:

```bash
WEBHOOK_URL=$(printenv 'SLACK-WEBHOOK-URL')
PAYLOAD="{\"pr_url\":\"$PR_URL\",\"msg\":\"$MSG\"}"
if [ -n "$WEBHOOK_URL" ]; then
  for ATTEMPT in 1 2 3; do
    FULL_RESPONSE=$(curl -s -w "\n%{http_code}" -X POST "$WEBHOOK_URL" \
      -H 'Content-type: application/json' -d "$PAYLOAD")
    HTTP_CODE=$(echo "$FULL_RESPONSE" | tail -1)
    BODY=$(echo "$FULL_RESPONSE" | head -n -1)
    echo "Slack attempt $ATTEMPT: HTTP $HTTP_CODE — $BODY"
    [ "$HTTP_CODE" = "200" ] && break
    { [ "$HTTP_CODE" = "429" ] || [ "${HTTP_CODE:0:1}" = "5" ]; } && sleep 2 || break
  done
else
  echo "Slack: SLACK-WEBHOOK-URL not set — skipped"
fi
```

## Execution Rules

- Execute every step autonomously. No confirmation prompts.
- On any failure: notify Slack as described for that step, then continue to the next activity. Never abort the whole run.
- Derive `<short-sha>` as the first 8 characters of `<commit-id>`.

## Per-Activity Process

Repeat all steps A–I for each entry in the list above.

### A. Clone

```bash
git clone <url> /tmp/<name>
cd /tmp/<name>
```

**On failure** → **[SLACK]** `PR_URL=""` `MSG="❌ <name>: clone failed — <error-summary>. Skipped."` → clean up → next activity.

### B. Create branch

```bash
git checkout -b feat/activity-base-<short-sha>
```

### C. Update submodule

```bash
git submodule update --init --remote
```

**On failure** → **[SLACK]** `PR_URL=""` `MSG="❌ <name>: submodule update failed — <error-summary>. Skipped."` → `rm -rf /tmp/<name>` → next activity.

### D. Find solution file

```bash
SLNX_FILE=$(find /tmp/<name> -maxdepth 1 -name "*.slnx" | head -1)
echo "Solution: $SLNX_FILE"
```

### E. Inject NuGet credentials

Use Python3 (handles special characters safely — do NOT use sed):

```bash
cd /tmp/<name>
python3 << 'PYEOF'
import os, subprocess

result = subprocess.run(
    ["find", ".", "-name", "nuget.config", "-not", "-path", "*/.git/*"],
    capture_output=True, text=True
)
cfg = result.stdout.strip().split("\n")[0]
if not cfg:
    print("ERROR: nuget.config not found"); exit(1)

client_id = os.environ.get("NUGET-SP-CLIENT-ID", "")
token     = os.environ.get("AZURE-DEVOPS-TOKEN", "")
if not client_id or not token:
    print("ERROR: NUGET-SP-CLIENT-ID or AZURE-DEVOPS-TOKEN not set"); exit(1)

with open(cfg) as f: content = f.read()
content = content.replace("%NUGET_SP_CLIENT_ID%", client_id)
content = content.replace("%NUGET_SP_PASSWORD%", token)
with open(cfg, "w") as f: f.write(content)
print(f"NuGet credentials injected into {cfg}")
PYEOF
```

**On failure** → set `build_failed=true`, `build_stage="nuget-config"`. Skip F and G, continue to H.

### F. dotnet restore

```bash
dotnet restore "$SLNX_FILE" 2>&1
echo "Restore exit: $?"
```

**On non-zero exit** → set `build_failed=true`, `build_stage="restore"`, `restore_failed=true`. Skip G, continue to H.

### G. Build + fix loop (hard cap: 10 attempts)

```bash
dotnet build "$SLNX_FILE" -c Release /p:AzureBuild=true 2>&1
```

**On success** (exit 0): set `build_failed=false`, continue to H.

**On failure**:
1. Read the full compiler error output.
2. Use the activity-base PR context (title `<pr-title>`, commit `<commit-id>`) to understand what API changed — missing method, renamed type, changed constructor signature, new required parameter. Breaking changes may not be documented; diagnose from compiler errors.
3. Fix ONLY the files the compiler errors point to. Do not refactor, rename, or touch anything unrelated. Print each file you modify.
4. Re-run build. Repeat up to 10 total attempts.
5. After 10 attempts still failing: set `build_failed=true`, `build_stage="build"`.

Track: `build_attempts` (count), `build_files_modified` (list), `build_error_summary` (last error classes seen).

**Do NOT restore nuget.config here** — step I handles that before committing.

### H. Run + fix tests (skip entirely if build_failed is true)

#### H1. Run tests

```bash
dotnet test "$SLNX_FILE" --no-build 2>&1
```

If all pass → set `tests_failed=false`, continue to I.

#### H2. Diagnose each failure

For each failing test:
1. Read the test file.
2. Find and read the implementation it calls (use Grep on the method name).
3. Classify the root cause:
   - **Outdated assertion**: method exists but return value or signature changed
   - **Stale mock/stub**: interface method renamed or signature changed
   - **New required parameter**: constructor or method added a param with no default
   - **Missing coverage**: new public method in the submodule has no test

Fix ONLY test project files. Do not modify production code.

#### H3. New test coverage

After fixing existing failures, check if the PR context indicates new public interfaces were added. If yes:
1. Find the test project: `find . -type d -name "*Test*" | head -3`
2. Read existing tests to understand the test style (base class, assertion framework, mock framework used).
3. Write new test methods following the existing patterns exactly. Cover only the new interface as it applies to this specific activity.

#### H4. Build + test loop (hard cap: 5 cycles)

After each round of fixes:

```bash
dotnet build "$SLNX_FILE" -c Release /p:AzureBuild=true 2>&1
dotnet test "$SLNX_FILE" --no-build 2>&1
```

Both pass → set `tests_failed=false`. Still failing after 5 complete cycles → set `tests_failed=true`.

Track: `test_cycles` (count), `tests_fixed` (list), `tests_written` (list), `test_error_summary`.

### I. Stage and commit

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

### J. Push

```bash
git push origin feat/activity-base-<short-sha>
```

**On failure** → **[SLACK]** `PR_URL=""` `MSG="❌ <name>: push failed — <error-summary>. No PR created."` → `rm -rf /tmp/<name>` → next activity.

### K. Create PR via ADO REST API

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

If `PR_WEB_URL` is empty: extract `d['message']` from the response, set `pr_creation_failed=true` with that message. Continue to L — the notifier handles the empty URL.

The generated PR description must include:
- What changed in ams-activity-base (PR title + ID)
- Build status: succeeded / failed at stage `<build_stage>` after `<build_attempts>` attempts (list error classes if failed)
- Test status: passed / failed after `<test_cycles>` cycles / skipped (reason)
- Files modified (build: `<build_files_modified>`, tests: `<tests_fixed>`, new: `<tests_written>`)

### L. Slack notification + clean up

Determine the outcome message:

| Condition | msg |
|-----------|-----|
| build ✅ tests ✅ | `✅ <name>: Activity Base update PR ready` |
| build ✅ tests ⚠️ | `⚠️ <name>: Activity Base update PR created — test failures, needs review` |
| build ⚠️ (PR created) | `⚠️ <name>: Activity Base update PR created — build failures, manual fixes needed` |
| PR creation failed | `❌ <name>: PR creation failed — <api-error-message>` |

**[SLACK]** `PR_URL=<PR_WEB_URL>` `MSG=<determined message above>`

```bash
rm -rf /tmp/<name>
```

## Final Summary

After all activities are processed:

**[SLACK]** `PR_URL=""` `MSG="*Activity-base update complete*\nTriggered by PR #<pr-id>: <pr-title>\n• ✅ <n-success> succeeded\n• ⚠️ <n-issues> PRs with issues\n• ❌ <n-skipped> skipped"`
