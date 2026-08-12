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

### C2. Early exit if `main` already has this update

**This is the single largest time saving in the whole run.** Without it, a repo that is already up to date still pays a full restore, build, and test before step I discovers there is nothing to commit. Across ten activities on a re-run, that is ten wasted pipelines.

Compare against the **remote** `main`, not local `HEAD`: if a previous run committed the bump locally but failed to push, `HEAD` looks current while no PR exists, and skipping would silently drop the work.

Compare against `FETCH_HEAD`, not `origin/main`. `git fetch origin main` reliably sets `FETCH_HEAD`, but does not always update the `refs/remotes/origin/main` tracking ref — comparing against a stale tracking ref makes this check never fire.

```bash
cd /tmp/<name>
SUB_PATH=$(git config --file .gitmodules --get-regexp path | awk '{print $2}' | head -1)
git fetch -q origin main
if git diff --quiet FETCH_HEAD -- "$SUB_PATH"; then
  echo "SKIP: remote main already pins $SUB_PATH at this commit — nothing to do"
  exit 3
fi
echo "Update needed — remote main differs at $SUB_PATH"
git diff --submodule=short FETCH_HEAD -- "$SUB_PATH" | grep Subproject
```

**On exit 3** → set `no_changes=true`. Skip steps D through K entirely and report `ℹ️ <name>: already up to date — no PR needed` at step L. Do not restore, build, or test.

### C3. Extract what actually changed in activity-base

Do not diagnose breaking changes from compiler errors alone. The exact diff between the commit `main` currently pins and `<commit-id>` is available locally, and it is **ground truth** about which APIs moved. A compiler error tells you a call site broke; this diff tells you what it should become.

The old SHA is what remote `main` pins — the same `FETCH_HEAD` from C2, which persists in `.git` across Bash calls.

```bash
cd /tmp/<name> && . /tmp/<name>.env
git fetch -q origin main
OLD_SUB=$(git rev-parse FETCH_HEAD:"$SUB_PATH")
NEW_SUB=$(git -C "$SUB_PATH" rev-parse HEAD)
DIFF_FILE=/tmp/<name>-base-diff.md
echo "activity-base: $OLD_SUB -> $NEW_SUB"

{
  echo "# ams-activity-base changes: ${OLD_SUB:0:8} -> ${NEW_SUB:0:8}"
  echo
  echo "## Files changed"
  git -C "$SUB_PATH" diff --stat "$OLD_SUB" "$NEW_SUB"
  echo
  echo "## Added / deleted / renamed"
  git -C "$SUB_PATH" diff --name-status -M "$OLD_SUB" "$NEW_SUB"
  echo
  echo "## Public API surface changes"
  echo '```diff'
  git -C "$SUB_PATH" diff -U0 "$OLD_SUB" "$NEW_SUB" -- '*.cs' \
    | grep -E '^[+-]' | grep -vE '^(\+\+\+|---)' \
    | grep -E '(public|protected|internal|interface|abstract|virtual|override|record|enum|class|struct|required)' \
    | head -200
  echo '```'
  echo
  echo "## Package reference changes"
  echo '```diff'
  git -C "$SUB_PATH" diff -U0 "$OLD_SUB" "$NEW_SUB" -- '*.csproj' 'Directory.Packages.props' \
    | grep -E '^[+-].*(PackageReference|PackageVersion)' | head -40
  echo '```'
} > "$DIFF_FILE"

echo "BASE_DIFF='$DIFF_FILE'" >> /tmp/<name>.env
cat "$DIFF_FILE"
```

`-U0` with the signature grep is deliberate: it yields only changed declaration lines, so a large refactor still produces a short, high-signal summary instead of thousands of context lines. If the API section hits the 200-line cap, read the full diff for the specific type you are fixing rather than dumping everything:

```bash
git -C "$SUB_PATH" diff "$OLD_SUB" "$NEW_SUB" -- '*/TypeName.cs'
```

**Read `$BASE_DIFF` before attempting any fix in step G or H.** It is written once per activity and persists for the whole run, so re-read it rather than re-deriving it on each attempt.

**If a package reference changed**, expect `NU1004` from `--locked-mode` in F3 and a `packages.lock.json` update that must be committed at step I.

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

> **Every `dotnet` command in steps F, G, and H must be run with the Bash tool's `timeout` set to `600000` (10 minutes), its maximum.**
>
> The Bash tool defaults to a **2-minute** timeout. A cold NuGet restore against the Compello feed routinely exceeds that, so with the default the call is killed mid-restore, the step is reported as failed, and the whole run gets rescheduled — the timeout is not the feed being slow, it is the tool budget being too small.

#### F1. Warm up the toolchain and park the NuGet caches on the shared volume

The first `dotnet` invocation may trigger an SDK download (the executor provisions .NET on demand via mise). Do that here so it cannot consume the restore's budget. Run with `timeout: 600000`.

The executor sets `NUGET_PACKAGES` onto the shared tenant volume, but **not** `NUGET_HTTP_CACHE_PATH`. That second cache holds the downloaded `.nupkg` files and defaults to a container-local path, so without this it is discarded after every run and every activity re-downloads from the feed. Point it at the same volume — all ten activities share the `AMS.*` packages, so the first restore warms the cache for the rest.

```bash
cd /tmp/<name> && . /tmp/<name>.env
dotnet --version
dotnet --list-sdks

# Persist the .nupkg HTTP cache next to the package cache, whatever volume that is on.
if [ -n "${NUGET_PACKAGES:-}" ]; then
  export NUGET_HTTP_CACHE_PATH="$(dirname "$NUGET_PACKAGES")/nuget-http-cache"
  mkdir -p "$NUGET_HTTP_CACHE_PATH"
  echo "export NUGET_HTTP_CACHE_PATH='$NUGET_HTTP_CACHE_PATH'" >> /tmp/<name>.env
  echo "NuGet http cache: $NUGET_HTTP_CACHE_PATH"
else
  echo "WARNING: NUGET_PACKAGES unset — no shared runtime volume. Every restore will be cold."
fi
echo "NUGET_PACKAGES=${NUGET_PACKAGES:-<unset>}"
```

If that warning appears, the shared volume is not mounted and **no plugin-side change can make restores fast** — fix the mount instead. See *Speed checklist* at the end of this document.

#### F2. Probe feed auth before restoring

A bad or expired credential makes NuGet retry with backoff until the tool budget runs out, so an auth failure looks exactly like a timeout. This probe turns that into an instant, unambiguous error. It prints only the HTTP status — never the token.

```bash
CID=$(printenv 'NUGET-SP-CLIENT-ID')
TOK=$(printenv 'AZURE-DEVOPS-TOKEN')
CODE=$(curl -s -o /dev/null -w '%{http_code}' --max-time 30 -u "$CID:$TOK" \
  "https://compello.pkgs.visualstudio.com/_packaging/Compello/nuget/v3/index.json")
echo "Compello feed auth probe: HTTP $CODE"
case "$CODE" in
  200) echo "Feed reachable and credentials accepted" ;;
  401|403) echo "FATAL: feed rejected the credentials (HTTP $CODE) — restore would hang, not fail fast"; exit 1 ;;
  000) echo "FATAL: no network route to the Compello feed"; exit 1 ;;
  *) echo "WARNING: unexpected status $CODE — continuing, restore may still work" ;;
esac
```

**On 401/403/000** → set `build_failed=true`, `build_stage="nuget-auth"`. Skip F3 and G, continue to H. Report the HTTP code in Slack — this is a credential problem, not a build problem, and retrying will never fix it.

#### F3. Restore, retrying on timeout

Run with `timeout: 600000`. **A restore killed by the timeout is worth retrying**, because every package that finished downloading stays in `NUGET_PACKAGES` (the executor points this at the shared tenant volume). Each attempt therefore starts from a warmer cache and gets further than the last.

Every project in these repos ships a `packages.lock.json`, so try `--locked-mode` first: it restores straight from the lock file and skips dependency graph resolution entirely, which is the bulk of restore time on a warm cache.

```bash
cd /tmp/<name> && . /tmp/<name>.env
echo "NUGET_PACKAGES=${NUGET_PACKAGES:-<default>}"
dotnet restore "$SLNX_FILE" -p:Platform="Any CPU" --locked-mode 2>&1
echo "Restore exit: $?"
```

`--locked-mode` **fails by design** (`NU1004`, "the packages lock file is inconsistent") when the submodule bump changed a package version — that is the lock file doing its job, not an error to work around. When you see `NU1004`, drop `--locked-mode` and restore normally; that run legitimately needs to update the lock, and the updated `packages.lock.json` must be committed with the rest of the change at step I.

Retry rules — make each attempt a **separate** Bash call with `timeout: 600000`, never a loop inside one block (three attempts in one block would exceed the 10-minute cap):

| Attempt | Command change |
|---|---|
| 1 | with `--locked-mode` (fast path) |
| 2 | drop `--locked-mode` — required if `NU1004`, harmless otherwise |
| 3 | add `--disable-parallel` — serialises downloads, clearing feed throttling and connection resets |
| 4 | add `--disable-parallel -v n` — normal verbosity so the package or source that stalls is named |

Give up after **4** attempts.

**On non-zero exit after 3 attempts** → set `build_failed=true`, `build_stage="restore"`, `restore_failed=true`. Skip G, continue to H. Record in `build_error_summary` whether the failure was a timeout (attempts exhausted) or a real NuGet error, so the PR description distinguishes an infrastructure problem from a broken dependency.

### G. Build + fix loop (hard cap: 10 attempts)

Run every build with `timeout: 600000`. Add `--no-restore` so the build cannot silently re-enter restore and spend its whole budget there — F3 already restored, and a build that genuinely needs a restore should surface that as an error rather than hiding it as a timeout.

```bash
cd /tmp/<name> && . /tmp/<name>.env
dotnet build "$SLNX_FILE" -c Release -p:Platform="Any CPU" /p:AzureBuild=true --no-restore 2>&1
```

If a build fails with `NU1101`/`NU1102` (package not found) or otherwise insists assets are missing, re-run F3 once, then resume the loop. Do not add `--no-restore` blindly if the submodule bump introduced a genuinely new package reference — in that case a restore is legitimately required.

**On success** (exit 0): set `build_failed=false`, continue to H.

**On failure**:

1. Read the full compiler error output.
2. **Read `$BASE_DIFF` (written in C3).** This is the authoritative account of what changed in activity-base. Map each compiler error to the diff entry that caused it before editing anything — the diff tells you what the call site should become, which the error alone does not.
3. Match the error to its cause and apply the corresponding fix:

   | Compiler error | Look for in `$BASE_DIFF` | Fix |
   |---|---|---|
   | `CS0117` / `CS1061` — member does not exist | a removed `public` member with a similar added one | use the renamed member |
   | `CS1501` / `CS7036` — wrong argument count / missing required arg | added parameter on a changed signature | pass the new argument; use `CancellationToken.None` only if no token is in scope |
   | `CS9035` — required member not set | added `required` property | set it at every construction site |
   | `CS0535` — interface member not implemented | added interface member | implement it, including in test mocks and stubs |
   | `CS0246` — type not found | deleted or moved type | update the namespace, or the replacement type if renamed |
   | `CS0019` / enum mismatch | renamed enum member | use the new member name |
   | `NU1101` / `NU1102` | package reference change | re-run F3 without `--locked-mode` |

4. Fix ONLY the files the compiler errors point to. Do not refactor, rename, or touch anything unrelated. Print each file you modify.
5. Re-run build. Repeat up to 10 total attempts.
6. **If two consecutive attempts produce the same error**, stop editing and re-read `$BASE_DIFF` plus the changed base type in full (`git -C "$SUB_PATH" show <new-sha> -- path/to/Type.cs`). Repeating a failing edit burns attempts; the signature you need is in the submodule source.
7. After 10 attempts still failing: set `build_failed=true`, `build_stage="build"`. Record which error classes never resolved.

**Never work around a breaking change by deleting a call, stubbing a method to throw, or commenting code out.** Those produce a green build that ships broken behaviour. If the correct fix is genuinely unclear, leave the build failing and let step L report it for manual review — a failed build with an accurate Slack message is far better than a passing build that silently drops functionality.

Track: `build_attempts` (count), `build_files_modified` (list), `build_error_summary` (last error classes seen).

**Do NOT restore nuget.config here** — step I handles that before committing.

### H. Run + fix tests (skip entirely if build_failed is true)

#### H1. Run tests

`-c Release` must match the build configuration — without it the test runner looks for `Debug` assemblies and reports "no test assemblies found".

Run with `timeout: 600000`. `--no-build` already prevents a hidden restore here.

```bash
cd /tmp/<name> && . /tmp/<name>.env
dotnet test "$SLNX_FILE" -c Release -p:Platform="Any CPU" --no-build 2>&1
```

If all pass → set `tests_failed=false`, continue to I.

If the runner reports **no test project / no test assemblies**, that is not a pass. Set `tests_failed=false` but record `test_error_summary="no test project found"` so the PR description states tests were absent rather than green.

#### H2. Diagnose each failure

**Read `$BASE_DIFF` first.** A test failing after a submodule bump almost always traces to a specific entry in it, and a behavioural change in the base is a very different fix from a stale assertion.

For each failing test:
1. Read the test file.
2. Find and read the implementation it calls (use Grep on the method name).
3. Cross-reference `$BASE_DIFF` and classify the root cause:

   | Symptom | Cause in `$BASE_DIFF` | Fix |
   |---|---|---|
   | Assertion expects an old value | renamed enum member or changed return type | update the expected value |
   | Mock does not satisfy the interface | added interface member | add the member to the mock setup |
   | Constructor call fails to compile | added required parameter or `required` property | supply it in the arrange step |
   | Test passes but asserts nothing meaningful | behaviour moved in the base | assert against the new behaviour, not the old shape |

4. **Decide whether the test or the expectation is wrong.** If `$BASE_DIFF` shows the base deliberately changed behaviour, the test's expectation is outdated and should be updated. If the diff shows no relevant behavioural change, the failure is likely a real regression in this activity — do not "fix" the test to make it pass; report it.

Fix ONLY test project files. Do not modify production code.

**Never weaken a test to make it pass** — no deleting assertions, no `Assert.True(true)`, no `[Skip]`, no loosening an exact assertion to a null check. If a test cannot be made to pass honestly, leave it failing; step L reports it as needing review. A weakened test is worse than a failing one because it hides the regression permanently.

#### H3. New test coverage

After fixing existing failures, check the **Added / deleted / renamed** and **Public API surface changes** sections of `$BASE_DIFF` for new public types or members. Those are the only things needing new coverage — no guessing from the PR title.

1. Find the test project: `find . -type d -name "*Test*" | head -3`
2. Read existing tests to understand the test style (base class, assertion framework, mock framework used).
3. Write new test methods following the existing patterns exactly.

Cover only what this activity actually uses. A new base interface that this activity never references needs no test here — it belongs in the base repo's own suite. Grep for the new type first:

```bash
cd /tmp/<name> && . /tmp/<name>.env
grep -rl "NewTypeName" --include=*.cs . | grep -v "$SUB_PATH" || echo "not referenced by this activity — no test needed"
```

#### H4. Build + test loop (hard cap: 5 cycles)

After each round of fixes. Run with `timeout: 600000` — this block is a build *and* a test run, so it is the most likely of all steps to exceed the 2-minute default.

```bash
cd /tmp/<name> && . /tmp/<name>.env
dotnet build "$SLNX_FILE" -c Release -p:Platform="Any CPU" /p:AzureBuild=true --no-restore 2>&1 \
  && dotnet test "$SLNX_FILE" -c Release -p:Platform="Any CPU" --no-build 2>&1
```

`&&` rather than two separate lines: if the build fails there is nothing to test, and running `dotnet test --no-build` anyway produces a confusing "assemblies not found" error that masks the real compiler error.

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

## Speed Checklist

Ordered by impact. The first item dwarfs everything the plugin itself can do.

**1. Confirm the shared runtime volume is mounted.** If it is not, the .NET SDK is re-downloaded and every package is re-fetched on *every* run, and no plugin-side tuning helps. The executor logs one of these lines:

- `Runtime cache root: /workspace/runtimes` — mounted, caches persist
- `No shared runtime volume mounted — falling back to per-repo cache` — lost per repo
- `No persistent volume available` — lost per run, worst case

Grep the executor output for `[runtimes]`. If it is not the first line, fix the mount before tuning anything else.

**2. Early exit (step C2)** skips restore, build, and test for activities already on `main`. On a re-run this is close to the entire cost of the run.

**3. Warm caches are shared across activities.** All ten activities depend on the same `Amili.Send2.*` packages, so activity 1 pays the download and activities 2–10 hit the cache. Do not shuffle the repo order between runs; a stable order keeps the warm path predictable.

**4. `--locked-mode` restore and `--no-restore` builds** keep the per-activity fast path free of redundant dependency resolution.

**Not worth doing:** shallow clones. These repos are ~1.7 MB with single-digit commit counts, so `--depth 1` saves a fraction of a second and complicates the submodule pinning in step C.

**Deliberately not enabled: processing activities in parallel.** It is the largest remaining lever — ten sequential pipelines become roughly one — but this command mandates sequential inline execution, and parallel activities would contend on the same NuGet cache and provisioning lock. If you want it, say so explicitly; it needs the execution model at the top of this file changed, not just a flag.
