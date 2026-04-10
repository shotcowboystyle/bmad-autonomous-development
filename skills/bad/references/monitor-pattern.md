# Monitor Pattern

Use this pattern when `MONITOR_SUPPORT=true`. It covers two use cases in BAD: CI status polling (Step 4) and PR-merge watching (Phase 4 Branch B). The caller supplies the poll script and the reaction logic.

> **Requires Claude Code v2.1.98+.** Uses the same Bash permission rules. Not available on Amazon Bedrock, Google Vertex AI, or Microsoft Azure Foundry — set `MONITOR_SUPPORT=false` on those platforms.

## How it works

1. **Write a poll script** — a `while true; do ...; sleep N; done` loop that emits one line per status change to stdout.
2. **Start Monitor** — pass the script to the Monitor tool. Claude receives each stdout line as a live event and can react immediately without blocking the conversation.
3. **React to events** — on each line, apply the caller's reaction logic (e.g. CI green → proceed; PR merged → continue).
4. **Stop Monitor** — call stop/cancel on the Monitor handle when done (success, failure, or user override).

## CI status polling (Step 4)

Poll script (run inside the Step 4 subagent):
```bash
while true; do
  gh run view --json status,conclusion 2>&1
  sleep 30
done
```

React to each output line:
- `"conclusion":"success"` → stop Monitor, proceed to step 5
- `"conclusion":"failure"` or `"conclusion":"cancelled"` → stop Monitor, diagnose, fix, push, restart Monitor
- Billing/spending limit text in output → stop Monitor, run Local CI Fallback

## PR-merge watching (Phase 4 Branch B)

Poll script (run by the coordinator):
```bash
while true; do
  gh pr list --json number,mergedAt \
    --jq '.[] | select(.mergedAt != null) | "MERGED: #\(.number)"'
  sleep 60
done
```

React to each output line:
- `MERGED: #N` → CronDelete the fallback timer, stop Monitor, run Pre-Continuation Checks, re-run Phase 0

## If `MONITOR_SUPPORT=false`

- **CI polling:** use the manual `gh run view` loop in Step 4 (see Step 4 fallback path in SKILL.md).
- **PR-merge watching:** use the CronCreate-only Timer Pattern in Phase 4 Branch B (see fallback path in SKILL.md).
