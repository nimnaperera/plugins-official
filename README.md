# AMS Plugins

A curated marketplace of Claude plugins for [Amili SEND](https://amili.no/) — automation tools for Amili activity services.

---

## Available Plugins

| Plugin | Version | Description | Category |
|--------|---------|-------------|----------|
| [activity-updater](./plugins/activity-updater) | 1.0.0 | Propagates `ams-activity-base` submodule changes across all registered activity service repositories. Handles build/test fix loops, creates ADO PRs, and posts Slack notifications. | automation |

---

## Adding This Marketplace

Add this marketplace to the-agent using the `plugin marketplace add` command:

**From GitHub:**

```
claude plugin marketplace add nimnaperera/plugins-official
```

**From a remote `marketplace.json` URL:**

```
claude plugin marketplace add https://raw.githubusercontent.com/nimnaperera/plugins-official/main/.claude-plugin/marketplace.json
```

**From a local directory path:**

```
claude plugin marketplace add ./path/to/ams-plugins
```

---

## Installation

Once the marketplace is added, install a plugin via:

```
/plugin install activity-updater@ams-plugins
```

Or browse via `/plugin > Discover`.

---

## Plugins

### activity-updater

Propagates a merged `ams-activity-base` change into every registered activity service repository autonomously — no manual intervention required per repo.

**Command:** `/activity-update`

**Arguments:**

```
/activity-update <pr-id> <pr-title> <commit-id> <merged-by> <ado-org-url> <ado-project>
```

| Argument | Description |
|----------|-------------|
| `pr-id` | The merged PR number in `ams-activity-base` |
| `pr-title` | Title of the merged PR (quote if it contains spaces) |
| `commit-id` | Full merge commit SHA |
| `merged-by` | Display name of the person who merged the PR |
| `ado-org-url` | Azure DevOps organisation URL (e.g. `https://dev.azure.com/compello`) |
| `ado-project` | ADO project name (e.g. `Send - Azure`) |

**Example:**

```
/activity-update 142 "Add invoice attachment support" a3f9c21b4d6e7890 "Jane Smith" https://dev.azure.com/compello "Send - Azure"
```

**What it does — per repository:**

| Step | Action |
|------|--------|
| A | Clone the activity repo to `/tmp/<name>` |
| B | Create branch `feat/activity-base-<short-sha>` |
| C | Run `git submodule update --init --remote --merge` |
| D | Run `dotnet build`; auto-fix compiler errors up to **5 attempts** |
| E | Run `dotnet test --no-build`; auto-fix test failures up to **3 attempts** |
| F | Stage and commit with the standardised commit message |
| G | Push the branch |
| H | Create an ADO pull request targeting `main` |
| I | Post a Slack notification and clean up `/tmp/<name>` |

After all repos are processed, a final Slack summary reports success / issues / skipped counts.

**Slack notification outcomes:**

| Build | Tests | Slack message |
|-------|-------|---------------|
| ✅ | ✅ | `✅ <name>: Activity Base update PR ready` |
| ✅ | ⚠️ | `⚠️ <name>: Activity Base update PR created — test failures, needs review` |
| ⚠️ | skipped | `⚠️ <name>: Activity Base update PR created — build failures, manual fixes needed` |
| ❌ | — | `❌ <name>: <step> failed — <error>. Skipped.` |

**Required environment variable:**

```
SLACK_WEBHOOK_URL   Incoming webhook URL for the target Slack channel
```

**Adding / removing activity repositories:**

Edit the hardcoded list in [`plugins/activity-updater/commands/activity-update.md`](./plugins/activity-updater/commands/activity-update.md):

```json
[
  { "name": "ams-activity-deliver-print", "url": "https://compello@dev.azure.com/compello/Send%20-%20Azure/_git/ams-activity-deliver-print" }
]
```

Add each new repo as `{ "name": "<repo-name>", "url": "<ado-clone-url>" }`. Do not include `ams-activity-base` itself.

---

## Repository Structure

```
ams-plugins/
├── .claude-plugin/
│   └── marketplace.json          # Marketplace metadata
└── plugins/
    └── activity-updater/
        ├── .claude-plugin/
        │   └── plugin.json       # Plugin metadata
        └── commands/
            └── activity-update.md  # Command definition
```

---

## Owner

**Visma Amili AS** — [support.send@amili.no](mailto:support.send@amili.no)

---

## License

MIT
