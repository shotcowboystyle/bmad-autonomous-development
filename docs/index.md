# BAD — BMad Autonomous Development

> 🤖 Autonomous development orchestrator for the BMad Method. Runs fully autonomous parallel multi-agent pipelines through the full story lifecycle (create → dev → review → PR) driven by your sprint backlog and dependency graph.

## Overview

BAD is a [BMad Method](https://docs.bmad-method.org/) module that automates your entire sprint execution. A lightweight coordinator orchestrates the pipeline — it never reads files or writes code itself. **Every unit of work is delegated to a dedicated subagent with a fresh context window**, keeping each agent fully focused on its single task.

## Requirements

- [BMad Method](https://docs.bmad-method.org/) installed in your project
- A sprint plan with epics, stories, and `sprint-status.yaml`
- Git + GitHub CLI (`gh`) installed and authenticated:
  1. `brew install gh`
  2. `gh auth login`
  3. Add to your `.zshrc` so BAD's subagents can connect to GitHub:
     ```bash
     export GITHUB_PERSONAL_ACCESS_TOKEN=$(gh auth token)
     ```

## Installation

```bash
npx skills add https://github.com/stephenleo/bmad-autonomous-development
```

Then run setup in your project:

```
/bad setup
```

## Usage

```
/bad
```

BAD can also be triggered naturally: *"run BAD"*, *"kick off the sprint"*, *"automate the sprint"*, *"start autonomous development"*, *"run the pipeline"*, *"start the dev pipeline"*

Run with optional runtime overrides:

```
/bad MAX_PARALLEL_STORIES=2 AUTO_PR_MERGE=true MODEL_STANDARD=opus
```

## Pipeline

Once your epics and stories are planned, BAD takes over:

1. *(`MODEL_STANDARD` subagent)* Builds a dependency graph from your sprint backlog — maps story dependencies, syncs GitHub PR status, and identifies what's ready to work on
2. Picks ready stories from the graph, respecting epic ordering and dependencies
3. Runs up to `MAX_PARALLEL_STORIES` stories simultaneously — each in its own isolated git worktree — each through a sequential 4-step pipeline. **Every step runs in a dedicated subagent with a fresh context window**, keeping the coordinator lean and each agent fully focused on its single task:
   - **Step 1** *(`MODEL_STANDARD` subagent)* — `bmad-create-story`: generates the story spec
   - **Step 2** *(`MODEL_STANDARD` subagent)* — `bmad-dev-story`: implements the code
   - **Step 3** *(`MODEL_QUALITY` subagent)* — `bmad-code-review`: reviews and fixes the implementation
   - **Step 4** *(`MODEL_STANDARD` subagent)* — commit, push, open PR, monitor CI, fix any failing checks, resolve code review comments, and resolve merge conflicts
4. *(`MODEL_STANDARD` subagent)* Optionally auto-merges batch PRs sequentially (lowest story number first), resolving any conflicts
5. Waits, then loops back for the next batch — until the entire sprint is done

## Configuration

BAD is configured at install time (`/bad setup`) and stores settings in `_bmad/bad/config.yaml`. All values can be overridden at runtime with `KEY=VALUE` args.

| Variable | Config Key | Default | Description |
|---|---|---|---|
| `MAX_PARALLEL_STORIES` | `max_parallel_stories` | `3` | Stories to run per batch |
| `WORKTREE_BASE_PATH` | `worktree_base_path` | `.worktrees` | Base directory for per-story git worktrees (relative to repo root) |
| `MODEL_STANDARD` | `model_standard` | `sonnet` | Model for create, dev, and PR steps |
| `MODEL_QUALITY` | `model_quality` | `opus` | Model for code review step |
| `AUTO_PR_MERGE` | `auto_pr_merge` | `false` | Auto-merge PRs sequentially after each batch |
| `RUN_CI_LOCALLY` | `run_ci_locally` | `false` | Run CI locally instead of GitHub Actions |
| `WAIT_TIMER_SECONDS` | `wait_timer_seconds` | `3600` | Seconds to wait between batches |
| `RETRO_TIMER_SECONDS` | `retro_timer_seconds` | `600` | Seconds before auto-retrospective after epic completion |
| `CONTEXT_COMPACTION_THRESHOLD` | `context_compaction_threshold` | `80` | Context window % at which to compact context |
| `TIMER_SUPPORT` | `timer_support` | `true` | Use native platform timers; `false` for prompt-based continuation |
| `API_FIVE_HOUR_THRESHOLD` | `api_five_hour_threshold` | `80` | (Claude Code) 5-hour usage % at which to pause |
| `API_SEVEN_DAY_THRESHOLD` | `api_seven_day_threshold` | `95` | (Claude Code) 7-day usage % at which to pause |
| `API_USAGE_THRESHOLD` | `api_usage_threshold` | `80` | (Other harnesses) Generic usage % at which to pause |

## Agent Harness Support

BAD is harness-agnostic. Setup detects your installed harnesses by checking for their directories at the project root (`.claude/` for Claude Code, `.cursor/` for Cursor, `.github/skills/` for GitHub Copilot, etc.) and configures platform-specific settings accordingly:

- **Claude Code** — native timer support (`CronCreate`), Claude model names (`sonnet`/`opus`/`haiku`), 5-hour and 7-day rate limit thresholds
- **Other harnesses** — prompt-based continuation, free-text model names, single generic usage threshold

In multi-harness projects, setup runs once per detected harness and stores per-harness model settings (e.g. `claude_model_standard`, `cursor_model_standard`).

## Reconfigure

To update your configuration at any time:

```
/bad configure
```

## License

MIT © 2026 Marie Stephen Leo
