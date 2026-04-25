# HermesSweeper

AI-powered GitHub issue/PR sweeper using Claude — scans open issues and PRs, analyzes them with Anthropic's Claude API, and suggests (or applies) closures for items that are already resolved, out of scope, stale, or not actionable.

## Features

- **Pure text analysis** — gathers all issue/PR context via GitHub API, sends it to Claude for review. No code execution sandbox needed.
- **Conservative by default** — only proposes closures when confidence is high. Keeps items open when uncertain.
- **Configurable** — works with any GitHub repository. Target repo, model, tokens, and behavior are all configurable.
- **Sharded review** — supports splitting work across parallel shards for large repos.
- **Dashboard** — auto-generates a status dashboard in README.md with review stats and cadence tracking.
- **Reconciliation** — syncs local review state with GitHub, moving items between `items/` and `closed/` as their status changes.

## Setup

### 1. Clone the repo

```bash
git clone https://github.com/your-org/hermessweeper.git
cd hermessweeper
```

### 2. Install dependencies

```bash
npm install
```

### 3. Create `.env` file

```bash
cp .env.example .env
# Edit .env with your keys
```

Required environment variables:

| Variable | Description |
| --- | --- |
| `ANTHROPIC_API_KEY` | Your Anthropic API key for Claude access |
| `GITHUB_TOKEN` | GitHub personal access token (used by `gh` CLI) |
| `TARGET_REPO` | Target repository in `owner/repo` format |
| `REPORT_REPO` | (Optional) Repository where sweep reports are stored |

### 4. Configure (optional)

Create `hermessweeper.config.json` in the project root:

```json
{
  "targetRepo": "owner/repo",
  "reportRepo": "owner/sweeper-repo",
  "model": "claude-sonnet-4-20250514",
  "maxTokens": 4096,
  "freshDays": 7,
  "staleDays": 60
}
```

Config priority: **CLI args > environment variables > config file > defaults**.

### 5. Build

```bash
npm run build
```

## Usage

### Commands

| Command | Description |
| --- | --- |
| `npm run plan` | Scan open items and create a sharded review plan |
| `npm run review` | Review items using Claude and generate decision reports |
| `npm run apply-artifacts` | Import review artifacts into items/closed directories |
| `npm run apply-decisions` | Apply proposed closures (comment + close on GitHub) |
| `npm run reconcile` | Sync items/closed folders with current GitHub state |
| `npm run dashboard` | Update the README.md dashboard with latest stats |
| `npm run status` | Update workflow status block in README.md |

### CLI Arguments

Common arguments (all commands):

| Argument | Description |
| --- | --- |
| `--target-repo owner/repo` | Target repository to sweep |
| `--report-repo owner/repo` | Repository for sweep reports |
| `--items-dir path` | Directory for open item reviews (default: `items/`) |
| `--closed-dir path` | Directory for closed item reviews (default: `closed/`) |

Review-specific arguments:

| Argument | Description | Default |
| --- | --- | --- |
| `--model name` | Claude model to use | `claude-sonnet-4-20250514` |
| `--max-tokens n` | Max tokens for Claude response | `4096` |
| `--timeout-ms n` | Timeout for Claude API call | `120000` |
| `--batch-size n` | Items to review per shard | `5` |
| `--shard-index n` | Current shard index | `0` |
| `--shard-count n` | Total number of shards | `1` |
| `--item-number n` | Review a specific item only | — |
| `--apply-closures` | Actually close items (not just propose) | `false` |
| `--repo-dir path` | Path to local clone of target repo | `../repo-name` |

## How It Works

1. **Plan**: Scans all open issues/PRs via GitHub API, identifies items due for review based on cadence rules (daily for PRs and new issues, weekly for older issues).

2. **Review**: For each candidate item:
   - Fetches full context: issue body, comments, timeline events, PR details/files/commits
   - Constructs a detailed prompt with all context
   - Sends to Claude API for analysis
   - Claude returns a structured JSON decision: close (with reason) or keep open
   - Decision is saved as a markdown report

3. **Apply**: Reviews proposed closures, verifies the item hasn't changed since review, posts a close comment, and closes the issue/PR on GitHub.

4. **Reconcile**: Syncs the local `items/` and `closed/` directories with GitHub state — moves reopened items back to `items/`, moves externally-closed items to `closed/`.

### Close Reasons

| Reason | Description |
| --- | --- |
| `implemented_on_main` | The request is already implemented in the current main branch |
| `cannot_reproduce` | The reported issue cannot be reproduced on current main |
| `out_of_scope` | Better suited as a separate project, plugin, or extension |
| `incoherent` | The item is unclear, contradictory, or not actionable |
| `stale_insufficient_info` | Issue is >60 days old and lacks sufficient info to verify |

## Example

Review a specific issue:

```bash
# Build first
npm run build

# Review issue #42 in your target repo
node dist/hermessweeper.js review --target-repo octocat/hello-world --item-number 42

# Or using environment variables
TARGET_REPO=octocat/hello-world node dist/hermessweeper.js review --item-number 42
```

Full sweep workflow:

```bash
export TARGET_REPO=octocat/hello-world
export ANTHROPIC_API_KEY=sk-ant-...
export GITHUB_TOKEN=ghp_...

npm run build
npm run plan -- --batch-size 10 --shard-count 1
npm run review -- --batch-size 10
npm run apply-decisions -- --limit 5
npm run dashboard
```

## Dashboard

_Dashboard will be populated after the first review run._

## How It Works

See the "How It Works" section above.

## Credits

Forked and recoded from [openclaw/clawsweeper](https://github.com/openclaw/clawsweeper) — the original ClawSweeper bot built for the OpenClaw project. Original work by [@steipete](https://github.com/steipete) and contributors.

Key changes from the original:
- Replaced OpenAI Codex (GPT-5.4) with Anthropic Claude API
- Made target repository fully configurable (any GitHub repo)
- Generalized close reasons (`clawhub` → `out_of_scope`)
- Added `hermessweeper.config.json` support
- Pure text analysis approach (no sandbox/code execution needed)
- Lowered requirements to Node >=20 and standard TypeScript 5.x

## License

MIT
