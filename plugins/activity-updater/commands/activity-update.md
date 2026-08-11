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
  { "name": "ams-activity-deliver-print", "url": "https://compello@dev.azure.com/compello/Send%20-%20Azure/_git/ams-activity-deliver-print" }
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

Repeat all steps A–I for each entry in the list above.

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
git submodule update --init --remote --merge
```

**On failure** → Slack: `❌ <name>: submodule update failed — <error-summary>. Skipped.` → `rm -rf /tmp/<name>` → next activity.

### D. Build + fix loop

```bash
dotnet build 2>&1
```

- Exit 0 → proceed to E.
- Non-zero → read compiler output. Fix only what the errors explicitly require. Do not refactor unrelated code. Re-run build. Repeat.
- After **5 total attempts** still failing → set `build_failed=true`, record the final error summary, continue to F.

Track every file changed and every error class fixed. Include in PR description.

### E. Tests (only when `build_failed` is false)

```bash
dotnet test --no-build 2>&1
```

- Pass → continue.
- Fail → fix failing tests, re-run. Max **3 total attempts**.
- Still failing → set `tests_failed=true`, record summary, continue to F.

### F. Stage and commit

```bash
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

**On failure** → Slack: `❌ <name>: push failed — <error-summary>. No PR created.` → `rm -rf /tmp/<name>` → next activity.

### H. Create PR

```bash
az repos pr create \
  --org "<ado-org-url>" \
  --project "<ado-project>" \
  --repository "<name>" \
  --source-branch "feat/activity-base-<short-sha>" \
  --target-branch "main" \
  --title "chore: update ams-activity-base submodule (#<pr-id>)" \
  --description "<generated-description>"
```

PR description must include:
- What changed in ams-activity-base (PR title + ID)
- Build status: succeeded / failed after N attempts (list error classes if failed)
- Test status: passed / failed / skipped (reason)
- Files modified by the agent (if any)

Capture the PR web URL from the command output.

### I. Slack notification + clean up

Send one message per activity:

| Outcome | Message |
|---|---|
| Build ✅ Tests ✅ | `✅ <name>: Activity Base update PR ready` |
| Build ✅ Tests ⚠️ | `⚠️ <name>: Activity Base update PR created — test failures, needs review` |
| Build ⚠️ (PR created anyway) | `⚠️ <name>: Activity Base update PR created — build failures, manual fixes needed` |

```bash
curl -s -X POST "$SLACK_WEBHOOK_URL" \
  -H 'Content-type: application/json' \
  -d "{\"pr_url\":\"<pr-url>\",\"msg\":\"<message>\"}"

rm -rf /tmp/<name>
```

## Final Summary

After all activities, send one summary message:

```bash
curl -s -X POST "$SLACK_WEBHOOK_URL" \
  -H 'Content-type: application/json' \
  -d "{\"msg\":\"*Activity-base update complete*\nTriggered by PR #<pr-id>: <pr-title>\n• ✅ <n-success> succeeded\n• ⚠️ <n-issues> PRs with issues\n• ❌ <n-skipped> skipped\"}"
```
