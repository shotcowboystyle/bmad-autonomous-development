# BAD Module Setup

Standalone module self-registration for BMad Autonomous Development. This file is loaded when:
- The user passes `setup`, `configure`, or `install` as an argument
- The module is not yet registered in `{project-root}/_bmad/bad/config.yaml`

## Overview

Registers BAD into a project. Writes to:
- **`{project-root}/_bmad/config.yaml`** — shared project config (universal settings + harness-specific settings)
- **`{project-root}/_bmad/config.user.yaml`** — personal settings (gitignored): `user_name`, `communication_language`, and any `user_setting: true` variable
- **`{project-root}/_bmad/module-help.csv`** — registers BAD capabilities for the help system

Both config scripts use an anti-zombie pattern — existing `bad` entries are removed before writing fresh ones, so stale values never persist.

`{project-root}` is a **literal token** in config values — never substitute it with an actual path.

## Step 1: Check Existing Config

1. Read `./assets/module.yaml` for module metadata.
2. Check if `{project-root}/_bmad/config.yaml` has a `bad` section — if so, inform the user this is a reconfiguration and show existing values as defaults.
3. Check for inline args (e.g. `accept all defaults`, `--headless`, or `MAX_PARALLEL_STORIES=5`) — map any provided values to config keys, use defaults for the rest, skip prompting for those keys.

## Step 2: Detect Installed Harnesses

Check for the presence of harness directories at the project root:

| Directory | Harness |
|---|---|
| `.claude/` | `claude-code` |
| `.cursor/` | `cursor` |
| `.github/skills/` | `github-copilot` (use `/skills/` subfolder to avoid false positive on bare `.github/`) |
| `.codex/` | `openai-codex` |
| `.gemini/` | `gemini` |
| `.windsurf/` | `windsurf` |
| `.cline/` | `cline` |

Store all detected harnesses. Determine the **current harness** from this skill's own file path — whichever harness directory contains this running skill is the current harness. Use the current harness to drive the question branch in Step 3.

## Step 3: Collect Configuration

Show defaults in brackets. Present all values together so the user can respond once with only what they want to change. Never say "press enter" or "leave blank".

**Default priority** (highest wins): existing config values > `./assets/module.yaml` defaults.

### Core Config (only if not yet set)

Only collect if no core keys exist in `config.yaml` or `config.user.yaml`:

- `user_name` (default: BMad) — written exclusively to `config.user.yaml`
- `communication_language` and `document_output_language` (default: English — ask as a single language question, both keys get the same answer) — `communication_language` written exclusively to `config.user.yaml`
- `output_folder` (default: `{project-root}/_bmad-output`) — written to root of `config.yaml`, shared across all modules

### Universal BAD Config

Read from `./assets/module.yaml` and present as a grouped block:

- `max_parallel_stories` — Max stories to run in a single batch [3]
- `worktree_base_path` — Root directory for git worktrees, relative to repo root [.worktrees]
- `auto_pr_merge` — Auto-merge batch PRs sequentially after each batch? [No]
- `run_ci_locally` — Skip GitHub Actions and run CI locally by default? [No]
- `wait_timer_seconds` — Seconds to wait between batches before re-checking PR status [3600]
- `retro_timer_seconds` — Seconds before auto-running retrospective after epic completion [600]
- `context_compaction_threshold` — Context window % at which to compact/summarise context [80]

### Harness-Specific Config

Run once for the **current harness**. If multiple harnesses are detected, also offer to configure each additional harness in sequence after the current one — label each section clearly.

When configuring multiple harnesses, model and threshold variables are stored with a harness prefix (e.g. `claude_model_standard`, `cursor_model_standard`) so they coexist. Universal variables are shared and asked only once.

#### Claude Code (`claude-code`)

Present as **"Claude Code settings"**:

- `model_standard` — Model for story creation, dev, and PR steps
  - Choose: `sonnet` (default), `haiku`
- `model_quality` — Model for code review step
  - Choose: `opus` (default), `sonnet`
- `api_five_hour_threshold` — 5-hour API usage % at which to pause [80]
- `api_seven_day_threshold` — 7-day API usage % at which to pause [95]

Automatically write `timer_support: true` — no prompt needed.

#### All Other Harnesses

Present as **"{HarnessName} settings"**:

- `model_standard` — Model for story creation, dev, and PR steps (e.g. `fast`, `gpt-4o-mini`, `flash`)
- `model_quality` — Model for code review step (e.g. `best`, `o1`, `pro`)
- `api_usage_threshold` — API usage % at which to pause for rate limits [80]

Automatically write `timer_support: false` — no prompt needed. BAD will use prompt-based continuation instead of native timers on this harness.

## Step 4: Write Files

Write a temp JSON file with collected answers structured as:
```json
{
  "core": { "user_name": "...", "document_output_language": "...", "output_folder": "..." },
  "bad": {
    "max_parallel_stories": "3",
    "worktree_base_path": ".worktrees",
    "auto_pr_merge": false,
    "run_ci_locally": false,
    "wait_timer_seconds": "3600",
    "retro_timer_seconds": "600",
    "context_compaction_threshold": "80",
    "timer_support": true,
    "model_standard": "sonnet",
    "model_quality": "opus",
    "api_five_hour_threshold": "80",
    "api_seven_day_threshold": "95"
  }
}
```

Omit `core` key if core config already exists. Run both scripts in parallel:

```bash
python3 ./scripts/merge-config.py \
  --config-path "{project-root}/_bmad/config.yaml" \
  --user-config-path "{project-root}/_bmad/config.user.yaml" \
  --module-yaml ./assets/module.yaml \
  --answers {temp-file}

python3 ./scripts/merge-help-csv.py \
  --target "{project-root}/_bmad/module-help.csv" \
  --source ./assets/module-help.csv \
  --module-code bad
```

If either exits non-zero, surface the error and stop.

Run `./scripts/merge-config.py --help` or `./scripts/merge-help-csv.py --help` for full usage.

## Step 5: Create Directories

After writing config, create the worktree base directory at the resolved path of `{project-root}/{worktree_base_path}` if it does not exist. Use the actual resolved path for filesystem operations only — config values must continue to use the literal `{project-root}` token.

Also create `output_folder` and any other `{project-root}/`-prefixed values from the config that don't exist on disk.

## Step 6: Confirm and Greet

Display what was written: config values set, user settings written, help entries registered, fresh install vs reconfiguration.

Then display the module greeting:

> BAD is ready. Run /bad to start. Pass KEY=VALUE args to override config at runtime (e.g. /bad MAX_PARALLEL_STORIES=2).

## Return to Skill

Setup is complete. Resume normal BAD activation — load config from the freshly written files and proceed with whatever the user originally intended.
