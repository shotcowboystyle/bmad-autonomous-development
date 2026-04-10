# Timer Pattern

Both the retrospective and post-batch wait timers use this pattern. The caller supplies the duration, fire prompt, option labels, and actions.

Behaviour depends on `TIMER_SUPPORT`:

---

## If `TIMER_SUPPORT=true` (native platform timers)

**Step 1 — compute target cron expression** (convert seconds to minutes: `SECONDS ÷ 60`):
```bash
# macOS
date -v +{N}M '+%M %H %d %m *'
# Linux
date -d '+{N} minutes' '+%M %H %d %m *'
```
Save as `CRON_EXPR`. Save `TIMER_START=$(date +%s)`.

**Step 2 — create the one-shot timer** via `CronCreate`:
- `cron`: expression from Step 1
- `recurring`: `false`
- `prompt`: the caller-supplied fire prompt

Save the returned job ID as `JOB_ID`.

**Step 3 — print the options menu** (always all three options):
> Timer running (job: {JOB_ID}). I'll act in {N} minutes.
>
> - **[C] Continue** — {C label}
> - **[S] Stop** — {S label}
> - **[M] {N} Modify timer to {N} minutes** — shorten or extend the countdown

📣 **Notify** (see `references/notify-pattern.md`) with the same options so the user can respond from their device:
```
⏱ Timer set — {N} minutes (job: {JOB_ID})

[C] {C label}
[S] {S label}
[M] <minutes> — modify countdown
```

Wait for whichever arrives first — user reply or fired prompt. On any human reply, print elapsed time first:
```bash
ELAPSED=$(( $(date +%s) - TIMER_START ))
echo "⏱ Time elapsed: $((ELAPSED / 60))m $((ELAPSED % 60))s"
```

- **[C]** → `CronDelete(JOB_ID)`, run the [C] action
- **[S]** → `CronDelete(JOB_ID)`, run the [S] action
- **[M] N** → `CronDelete(JOB_ID)`, recompute cron for N minutes from now, `CronCreate` again with same fire prompt, update `JOB_ID` and `TIMER_START`, print updated countdown, then 📣 **Notify**:
  ```
  ⏱ Timer updated — {N} minutes (job: {JOB_ID})

  [C] {C label}
  [S] {S label}
  [M] <minutes> — modify countdown
  ```
- **FIRED (no prior reply)** → run the [C] action automatically

---

## If `TIMER_SUPPORT=false` (prompt-based continuation)

Save `TIMER_START=$(date +%s)`. No native timer is created — print the options menu immediately and wait for user reply:

> Waiting {N} minutes before continuing. Reply when ready.
>
> - **[C] Continue** — {C label}
> - **[S] Stop** — {S label}
> - **[M] N** — remind me after N minutes (reply with `[M] <minutes>`)

📣 **Notify** (see `references/notify-pattern.md`) with the same options.

On any human reply, print elapsed time first:
```bash
ELAPSED=$(( $(date +%s) - TIMER_START ))
echo "⏱ Time elapsed: $((ELAPSED / 60))m $((ELAPSED % 60))s"
```

- **[C]** → run the [C] action
- **[S]** → run the [S] action
- **[M] N** → update `TIMER_START`, print updated wait message, 📣 **Notify**, and wait again
