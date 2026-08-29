# Addendum: usage-threshold notifications (from Lume)

Pepe wants a Telegram ping as usage crosses each of:
25, 50, 60, 70, 80, 85, 90, 93, 95, 98 %.

**Two limits, two ladders.** Max plans have (a) the 5-hour rolling window
and (b) a WEEKLY cap — Pepe's resets **Tuesdays 16:00 CET** (his words,
2026-08-29). Tonight's 98% was the weekly one. Track BOTH: same threshold
ladder, separate state, messages labelled "5-h window" vs "weekly".
The weekly ladder is the one that really matters — 93% weekly on a
Thursday means degraded mode for days, so early warnings (25/50%) are
what let Pepe pace himself.

## Reading the number — the hard part

You're on a subscription (5-hour rolling windows), not metered API, so
there's no official "percent used" API. Options, best first:

1. **`ccusage` (npm, community standard).** `npx ccusage@latest blocks
   --json` parses the local `~/.claude/projects/**/*.jsonl` transcripts
   and reports the current 5-h block with token totals and, with
   `--token-limit`, a percentage. Calibrate the limit once: note what
   ccusage says at the moment Claude actually cuts you off, hardcode that.
2. **Claude Code statusline JSON** — recent versions pass usage info to
   statusline scripts; if your version includes plan-window fields, you
   can read the same number `/usage` shows. Check before building on it.
3. **Fallback:** count tokens yourself from the JSONLs (that's all ccusage
   does anyway).

Don't block on perfect accuracy — ±5% is fine for a warning ladder.

## The loop

`loops/usage-watch/run.py`, cron every 5 min. Cheap (no API calls), so it
does NOT check the degraded flag — it must keep running when degraded.

```python
THRESHOLDS = [25, 50, 60, 70, 80, 85, 90, 93, 95, 98]
STATE = Path.home() / "brain/state/usage-notified.json"
# state = {"window_id": "...", "notified": [25, 50]}

pct, window_id = read_usage()          # from ccusage --json
state = load(STATE)
if state["window_id"] != window_id:    # new 5-h window → reset ladder
    state = {"window_id": window_id, "notified": []}
due = [t for t in THRESHOLDS if t <= pct and t not in state["notified"]]
if due:
    tg_send(f"Token usage: {pct:.0f}% of this 5-h window "
            f"(crossed {', '.join(map(str, due))}%). "
            f"Resets at {window_end_local}.")
    state["notified"] += due
save(STATE, state)
```

Details that matter:

- **One message per crossing, batched.** If usage jumps 40→82 between
  polls, send ONE message noting 50/60/70/80 crossed — not four pings.
- **Reset per window.** The `window_id` (block start time from ccusage) is
  the reset key. New window → clean ladder → the 25% ping fires again.
- **Always include the reset time.** "Resets at 03:15" is the actually
  useful part of the message.
- **At ≥90%, same message suggests action:** "Consider pausing loops /
  degraded mode will trigger at limit."
- **Tie-in (Pepe confirmed, 2026-08-29):** at 93% on EITHER limit,
  pre-emptively `enter("93% usage")` from the degraded-mode spec — don't
  wait for the hard limit error. Notification and entry fire together:
  "Crossed 93% (weekly) — entering degraded mode (loops paused, Haiku
  replies). Resets Tuesday 16:00."
- **Weekly window math:** `window_id` for the weekly ladder = the most
  recent Tuesday 16:00 Europe/Madrid. Recovery probe note: when degraded
  mode was entered on weekly %, the probe succeeding (5-h window rolled)
  is NOT recovery — stay degraded until past the Tuesday boundary or
  weekly % drops below 93. Store which limit triggered it in the flag
  file's reason text and check accordingly.
- **Pacing line (weekly messages only):** include "on pace to run out
  ~Sunday" — linear projection from window start, rough is fine. This is
  the single most useful sentence in the whole system.

## Build order

1. Get `read_usage()` returning a believable number (test against what
   `/usage` shows) — this is 80% of the work.
2. Ladder loop + state file (30 min).
3. Wire tg_send through your existing notification path (NOT suppressed —
   see the ops-purge warning in the degraded-mode spec).

— Lume
