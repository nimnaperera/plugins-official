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
  { "name": "ams-activity-pdf-embedder", "url": "https://compello@dev.azure.com/compello/Send%20-%20Azure/_git/ams-activity-pdf-embedder" }
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

`MSG` frequently carries compiler output with quotes and newlines, so build the payload with Python3 rather than string-interpolating it — an unescaped quote produces invalid JSON and the notification is silently lost.

```bash
WEBHOOK_URL=$(printenv 'SLACK-WEBHOOK-URL')
PAYLOAD=$(MSG="$MSG" PR_URL="$PR_URL" python3 -c "
import json, os
print(json.dumps({'pr_url': os.environ.get('PR_URL',''), 'msg': os.environ.get('MSG','')}))
")
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

## Pre-Flight Check (run ONCE before the activity loop)

Abort the entire run if this fails — every activity would fail identically, so failing fast beats 25 wasted clones.

```bash
MISSING=""
for VAR in 'AZURE-DEVOPS-TOKEN' 'NUGET-SP-CLIENT-ID'; do
  [ -z "$(printenv "$VAR")" ] && MISSING="$MISSING $VAR"
done
for CMD in git dotnet python3 curl; do
  command -v "$CMD" >/dev/null 2>&1 || MISSING="$MISSING $CMD"
done
[ -z "$(printenv 'SLACK-WEBHOOK-URL')" ] && echo "WARN: SLACK-WEBHOOK-URL not set — notifications will be skipped"
if [ -n "$MISSING" ]; then
  echo "PREFLIGHT FAILED — missing:$MISSING"; exit 1
fi
echo "Preflight OK"
```

**On failure** → **[SLACK]** `PR_URL=""` `MSG="❌ Activity-base update aborted — preflight failed:<missing>"` → stop. Do NOT process any activity.

## Execution Rules

- Execute every step autonomously. No confirmation prompts.
- On any failure: notify Slack as described for that step, then continue to the next activity. Never abort the whole run (the sole exception is a failed pre-flight check).
- Derive `<short-sha>` as the first 8 characters of `<commit-id>`.
- All `dotnet` commands must carry `-p:Platform="Any CPU"`. ADO hosts export `Platform=azuredevops`, and a `.slnx` with no explicit configurations inherits it, producing an invalid `Debug|azuredevops` solution configuration.

### Shell state does NOT persist between Bash calls

Each Bash invocation is a fresh shell. A variable assigned in one block is **empty** in the next, so `dotnet restore "$SLNX_FILE"` in a later block would silently run against an empty path.

Two consequences you must respect:

1. Step D writes the per-activity state to `/tmp/<name>.env`. **Every later block that needs it must begin with this exact preamble:**

   ```bash
   cd /tmp/<name> && . /tmp/<name>.env
   ```

2. When a value is produced and consumed by the same step (step K's `AUTH`, `PR_BODY`, `PR_WEB_URL`), keep the producing and consuming commands **in a single Bash call**. Do not split them across calls.

The state file holds paths only — never write a credential into it.

### Credentials use hyphenated names — always read them with `printenv`

`AZURE-DEVOPS-TOKEN`, `NUGET-SP-CLIENT-ID`, and `SLACK-WEBHOOK-URL` contain hyphens, which are **not valid shell identifiers**. Consequences:

- `$AZURE-DEVOPS-TOKEN` never works — bash parses it as `$AZURE` followed by the literal `-DEVOPS-TOKEN`.
- `export AZURE-DEVOPS-TOKEN=...` is rejected with `not a valid identifier`.
- The only correct reads are `$(printenv 'AZURE-DEVOPS-TOKEN')` in bash and `os.environ.get("AZURE-DEVOPS-TOKEN")` in Python. Both read the process environment directly and work correctly.

Do not "simplify" these to `$VAR` form. It silently yields an empty string, which turns the credential gate into a no-op and makes restore fail for an unrelated-looking reason.

## Per-Activity Process

Repeat all steps A–L for each entry in the list above.

### A. Clone

`rm -rf` first so a leftover directory from an aborted earlier run cannot poison this one.

A commit identity must be set explicitly. A fresh container has none, and `git commit` then fails at step I with `Author identity unknown` after all the build and test work is already done.

```bash
rm -rf /tmp/<name>
git clone <url> /tmp/<name>
cd /tmp/<name>
git config user.email "support.send@amili.no"
git config user.name "Activity Base Updater"
```

**On failure** → **[SLACK]** `PR_URL=""` `MSG="❌ <name>: clone failed — <error-summary>. Skipped."` → clean up → next activity.

### B. Create branch (idempotent)

Reuse the branch if a previous run already created it, otherwise create it.

```bash
BRANCH=feat/activity-base-<short-sha>
git checkout "$BRANCH" 2>/dev/null || git checkout -b "$BRANCH"
git rev-parse --abbrev-ref HEAD
```

**On failure** → **[SLACK]** `PR_URL=""` `MSG="❌ <name>: branch checkout failed — <error-summary>. Skipped."` → `rm -rf /tmp/<name>` → next activity.

### C. Update submodule — pin to the exact merged commit

Do NOT use `--remote`: it follows the submodule's remote HEAD, so any commit merged after `<commit-id>` would be propagated instead of the one this run is for.

```bash
git submodule update --init
SUB_PATH=$(git config --file .gitmodules --get-regexp path | awk '{print $2}' | head -1)
echo "Submodule path: $SUB_PATH"
git -C "$SUB_PATH" fetch origin
git -C "$SUB_PATH" checkout <commit-id>
git -C "$SUB_PATH" rev-parse HEAD
```

Verify the printed SHA equals `<commit-id>`. If it does not, treat this step as failed.

**On failure** → **[SLACK]** `PR_URL=""` `MSG="❌ <name>: submodule update failed — <error-summary>. Skipped."` → `rm -rf /tmp/<name>` → next activity.

### D. Find solution file and write the state file

The solution is **not** at the repo root — it typically lives under `appcode/`. Do not add `-maxdepth 1`; it finds nothing.

This step also writes `/tmp/<name>.env`, which every later step sources (see *Shell state* above).

```bash
cd /tmp/<name>
SLNX_FILE=$(find . -name "*.slnx" -not -path "*/.git/*" | head -1)
if [ -z "$SLNX_FILE" ]; then
  echo "ERROR: no .slnx solution file found"; exit 1
fi
SUB_PATH=$(git config --file .gitmodules --get-regexp path | awk '{print $2}' | head -1)

cat > /tmp/<name>.env << EOF
SLNX_FILE='$SLNX_FILE'
SUB_PATH='$SUB_PATH'
BRANCH='feat/activity-base-<short-sha>'
EOF

cat /tmp/<name>.env
```

Confirm all three values printed non-empty (`SUB_PATH` may legitimately be empty only if the repo has no submodule, which would itself be a failure for this workflow).

**On failure (no solution found)** → **[SLACK]** `PR_URL=""` `MSG="❌ <name>: no .slnx solution file found. Skipped."` → `rm -rf /tmp/<name> /tmp/<name>.env` → next activity. Do not attempt restore or build without a solution path.

### E. Inject NuGet credentials

Use Python3, never `sed` — tokens routinely contain `/`, `+`, and `=`, which break `sed` substitution. Discover the files with `pathlib` rather than a `find` subprocess: one less external dependency and no shell quoting to get wrong.

A repo normally has **two** `nuget.config` files (`appcode/` and the submodule). Only the one carrying placeholders needs injecting; the loop below skips the other rather than failing on it.

```bash
cd /tmp/<name>
python3 << 'PYEOF'
import os, sys
from pathlib import Path

cfgs = [p for p in Path(".").rglob("nuget.config") if ".git" not in p.parts]
if not cfgs:
    sys.exit("ERROR: nuget.config not found")

client_id = os.environ.get("NUGET-SP-CLIENT-ID", "")
token     = os.environ.get("AZURE-DEVOPS-TOKEN", "")
if not client_id or not token:
    sys.exit("ERROR: NUGET-SP-CLIENT-ID or AZURE-DEVOPS-TOKEN not set")

injected = 0
for cfg in cfgs:
    content = cfg.read_text()
    if "%NUGET_SP_CLIENT_ID%" not in content and "%NUGET_SP_PASSWORD%" not in content:
        print(f"Skipped {cfg} — no placeholders")
        continue
    content = content.replace("%NUGET_SP_CLIENT_ID%", client_id)
    content = content.replace("%NUGET_SP_PASSWORD%", token)
    cfg.write_text(content)
    print(f"NuGet credentials injected into {cfg}")
    injected += 1

if injected == 0:
    sys.exit("ERROR: no nuget.config contained credential placeholders")
print(f"Injected into {injected} file(s)")
PYEOF
```

`sys.exit("message")` prints to stderr and exits non-zero, so a failure here is visible in the tool output rather than silently continuing.

**On failure** → set `build_failed=true`, `build_stage="nuget-config"`. Skip F and G, continue to H.

### F. dotnet restore

```bash
cd /tmp/<name> && . /tmp/<name>.env
dotnet restore "$SLNX_FILE" -p:Platform="Any CPU" 2>&1
echo "Restore exit: $?"
```

**On non-zero exit** → set `build_failed=true`, `build_stage="restore"`, `restore_failed=true`. Skip G, continue to H.

### G. Build + fix loop (hard cap: 10 attempts)

```bash
cd /tmp/<name> && . /tmp/<name>.env
dotnet build "$SLNX_FILE" -c Release -p:Platform="Any CPU" /p:AzureBuild=true 2>&1
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

`-c Release` must match the build configuration — without it the test runner looks for `Debug` assemblies and reports "no test assemblies found".

```bash
cd /tmp/<name> && . /tmp/<name>.env
dotnet test "$SLNX_FILE" -c Release -p:Platform="Any CPU" --no-build 2>&1
```

If all pass → set `tests_failed=false`, continue to I.

If the runner reports **no test project / no test assemblies**, that is not a pass. Set `tests_failed=false` but record `test_error_summary="no test project found"` so the PR description states tests were absent rather than green.

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
cd /tmp/<name> && . /tmp/<name>.env
dotnet build "$SLNX_FILE" -c Release -p:Platform="Any CPU" /p:AzureBuild=true 2>&1
dotnet test "$SLNX_FILE" -c Release -p:Platform="Any CPU" --no-build 2>&1
```

Both pass → set `tests_failed=false`. Still failing after 5 complete cycles → set `tests_failed=true`.

Track: `test_cycles` (count), `tests_fixed` (list), `tests_written` (list), `test_error_summary`.

### I. Stage and commit

Restore every modified `nuget.config` before staging, then **verify no credential reached the index**. This is a hard gate: if the verification fails, abort this activity rather than push a commit containing a live token.

```bash
cd /tmp/<name> && . /tmp/<name>.env

# 1. Revert credential injection in the parent repo
git diff --name-only | grep "nuget.config" | xargs -r git checkout --

# 2. Revert credential injection inside the submodule (separate git repo)
if [ -n "$SUB_PATH" ]; then
  git -C "$SUB_PATH" diff --name-only | grep "nuget.config" | xargs -r git -C "$SUB_PATH" checkout --
fi

git add -A

# 3. HARD GATE — no placeholder-substituted credential may be staged
TOKEN=$(printenv 'AZURE-DEVOPS-TOKEN')
CLIENT_ID=$(printenv 'NUGET-SP-CLIENT-ID')

# An empty needle would make grep match every line and abort every activity.
# Treat missing credentials as a gate failure, never as "nothing to check".
if [ -z "$TOKEN" ] || [ -z "$CLIENT_ID" ]; then
  git reset
  echo "FATAL: credentials unavailable — cannot verify staged diff is clean"
  exit 1
fi

STAGED=$(git diff --cached)
LEAK=0
printf '%s' "$STAGED" | grep -qF -- "$TOKEN"     && { echo "FATAL: AZURE-DEVOPS-TOKEN present in staged diff"; LEAK=1; }
printf '%s' "$STAGED" | grep -qF -- "$CLIENT_ID" && { echo "FATAL: NUGET-SP-CLIENT-ID present in staged diff"; LEAK=1; }

# Backstop: any staged nuget.config must still carry the literal placeholders.
if git diff --cached --name-only | grep -q "nuget.config"; then
  git diff --cached -- '*nuget.config' | grep -q '%NUGET_SP_PASSWORD%' \
    || { echo "FATAL: staged nuget.config lost its placeholder — credentials likely substituted"; LEAK=1; }
fi

if [ "$LEAK" = "1" ]; then
  git reset
  echo "Staging aborted — credentials would have been committed"
  exit 1
fi
echo "Credential gate passed"
```

In the normal case `nuget.config` is fully reverted and therefore not staged at all, so the backstop is inert. It only engages if a `nuget.config` change *is* staged, where the placeholder must still be present.

**If the credential gate fails** → **[SLACK]** `PR_URL=""` `MSG="❌ <name>: credential leak detected in staged diff — commit aborted, no PR created."` → `rm -rf /tmp/<name>` → next activity. Never bypass this gate.

Then commit:

```bash
cd /tmp/<name>
git commit -m "chore: update ams-activity-base submodule

Activity-base PR #<pr-id>: <pr-title>
Submodule commit: <commit-id>
Merged by: <merged-by>"
git --no-pager log --stat -1
```

If `git commit` reports "nothing to commit", the submodule was already at `<commit-id>` and no build fixes were needed. Set `no_changes=true`, skip J and K, and report `ℹ️ <name>: already up to date — no PR needed` at step L.

### J. Push

Skip entirely if `no_changes=true`.

`--force-with-lease` is required because step B may have reused a branch from an earlier partial run. It refuses to overwrite work pushed by anyone else, so a concurrent change is reported as a failure rather than silently discarded.

```bash
cd /tmp/<name> && . /tmp/<name>.env
git push --set-upstream --force-with-lease origin "$BRANCH"
```

**On failure** → **[SLACK]** `PR_URL=""` `MSG="❌ <name>: push failed — <error-summary>. No PR created."` → `rm -rf /tmp/<name>` → next activity.

### K. Create PR via ADO REST API

Do NOT use `az` CLI — it is not authenticated in this container. Use the PAT from `AZURE-DEVOPS-TOKEN` directly.

**Never interpolate the generated description into the `-d` string.** The description is multi-line prose containing quotes, newlines, and backticks — inlining it produces malformed JSON, and `curl` then fails with an opaque error while the run reports "PR creation failed" for the wrong reason. Build the payload with Python3 so it is always valid JSON.

First write the description to its own file. Use a quoted heredoc so backticks and `$` in the prose are not expanded:

```bash
cat > /tmp/pr-desc-<name>.md << 'DESCEOF'
<generated-description>
DESCEOF
wc -l /tmp/pr-desc-<name>.md
```

Then create the PR. **This must be a single Bash call** — `AUTH`, `PROJECT_ENCODED`, and `PR_BODY` do not survive across calls, and a lost `AUTH` would send an unauthenticated request that fails for a misleading reason.

```bash
cd /tmp/<name> && . /tmp/<name>.env

TOKEN=$(printenv 'AZURE-DEVOPS-TOKEN')
AUTH="Authorization: Basic $(printf ':%s' "$TOKEN" | base64 | tr -d '\n')"
PROJECT_ENCODED=$(python3 -c "import urllib.parse, sys; print(urllib.parse.quote(sys.argv[1], safe=''))" "<ado-project>")
BASE="<ado-org-url>/${PROJECT_ENCODED}/_apis/git/repositories/<name>/pullrequests"

python3 - << 'PYEOF' > /tmp/pr-payload-<name>.json
import json
with open("/tmp/pr-desc-<name>.md") as f:
    desc = f.read()
print(json.dumps({
    "title": "chore: update ams-activity-base submodule (#<pr-id>)",
    "description": desc,
    "sourceRefName": "refs/heads/feat/activity-base-<short-sha>",
    "targetRefName": "refs/heads/main",
}))
PYEOF

PR_RESPONSE=$(curl -s -w "\n%{http_code}" -X POST "${BASE}?api-version=7.1" \
  -H "$AUTH" -H "Content-Type: application/json" \
  --data-binary @/tmp/pr-payload-<name>.json)
HTTP_CODE=$(echo "$PR_RESPONSE" | tail -1)
PR_BODY=$(echo "$PR_RESPONSE" | head -n -1)
echo "PR create: HTTP $HTTP_CODE"

PR_WEB_URL=$(printf '%s' "$PR_BODY" | python3 -c "
import sys, json
try:
    d = json.load(sys.stdin)
except Exception:
    print(''); raise SystemExit
print(d.get('_links', {}).get('web', {}).get('href', ''))
")

# ADO returns 409 + TF401179 when an active PR already exists for this source->target.
# That is success for our purposes — recover the existing PR URL instead of failing.
if [ -z "$PR_WEB_URL" ]; then
  EXISTING=$(curl -s -H "$AUTH" \
    "${BASE}?searchCriteria.sourceRefName=refs/heads/${BRANCH}&searchCriteria.status=active&api-version=7.1")
  PR_WEB_URL=$(printf '%s' "$EXISTING" | python3 -c "
import sys, json
try:
    d = json.load(sys.stdin)
except Exception:
    print(''); raise SystemExit
vals = d.get('value', [])
print(vals[0].get('_links', {}).get('web', {}).get('href', '') if vals else '')
")
  [ -n "$PR_WEB_URL" ] && echo "Reused existing PR: $PR_WEB_URL"
fi

# Surface the API error message when both attempts came up empty.
if [ -z "$PR_WEB_URL" ]; then
  printf '%s' "$PR_BODY" | python3 -c "
import sys, json
try:
    print('API error:', json.load(sys.stdin).get('message', '<no message>'))
except Exception:
    print('API error: unparseable response')
"
fi

echo "PR_WEB_URL=$PR_WEB_URL"
rm -f /tmp/pr-payload-<name>.json /tmp/pr-desc-<name>.md
```

Record the printed `PR_WEB_URL` — you need it for the Slack step, and it will not survive into the next Bash call.

If `PR_WEB_URL` is empty: set `pr_creation_failed=true` with the printed API error message plus the HTTP code. Continue to L — the notifier handles the empty URL.

The generated PR description must include:
- What changed in ams-activity-base (PR title + ID)
- Build status: succeeded / failed at stage `<build_stage>` after `<build_attempts>` attempts (list error classes if failed)
- Test status: passed / failed after `<test_cycles>` cycles / skipped (reason)
- Files modified (build: `<build_files_modified>`, tests: `<tests_fixed>`, new: `<tests_written>`)

Never include credential values or environment variable contents in the description.

### L. Slack notification + clean up

Determine the outcome message:

Evaluate top to bottom and use the first row that matches.

| Condition | msg |
|-----------|-----|
| `no_changes=true` | `ℹ️ <name>: already up to date — no PR needed` |
| PR creation failed | `❌ <name>: PR creation failed — <api-error-message>` |
| build ⚠️ (PR created) | `⚠️ <name>: Activity Base update PR created — build failures, manual fixes needed` |
| build ✅ tests ⚠️ | `⚠️ <name>: Activity Base update PR created — test failures, needs review` |
| build ✅ tests ✅ | `✅ <name>: Activity Base update PR ready` |

**[SLACK]** `PR_URL=<PR_WEB_URL>` `MSG=<determined message above>`

Clean up unconditionally — the clone holds a credential-injected `nuget.config` on disk, so it must not be left behind even when the activity failed.

```bash
rm -rf /tmp/<name> /tmp/<name>.env /tmp/pr-payload-<name>.json /tmp/pr-desc-<name>.md
```

## Final Summary

After all activities are processed:

**[SLACK]** `PR_URL=""` `MSG="*Activity-base update complete*\nTriggered by PR #<pr-id>: <pr-title>\n• ✅ <n-success> succeeded\n• ⚠️ <n-issues> PRs with issues\n• ℹ️ <n-uptodate> already up to date\n• ❌ <n-skipped> skipped"`

Every activity must land in exactly one bucket, and the four counts must sum to the total number of activities in the list. If they do not, report the discrepancy rather than adjusting a count to make it balance.
