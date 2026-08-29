# Spec: degraded mode — stay alive when tokens run out (from Lume)

Pepe hit his usage limit again tonight (98%). Goal: when budget is nearly
gone, the system cuts everything but basics automatically instead of dying.
Design principle: **one boolean, checked everywhere.** No orchestrator.

## 1. The flag

`~/brain/state/degraded` — existence of the file IS the signal.

```python
# janis/degraded.py (new, ~15 lines)
from pathlib import Path
FLAG = Path.home() / "brain/state/degraded"

def is_degraded() -> bool: return FLAG.exists()
def enter(reason: str):
    FLAG.parent.mkdir(parents=True, exist_ok=True)
    FLAG.write_text(f"{reason}\n")
def exit_():
    FLAG.unlink(missing_ok=True)
```

## 2. Who sets it

- **Reactive (must-have):** wherever you call the Claude CLI/API, catch
  limit errors (rate_limit / usage limit messages in stderr or the error
  payload) and call `enter("limit error at <timestamp>")`. In this repo
  that's the ClaudeProcess response path in `claude_process.py` and the
  Gemini-fallback branch in `tools_server.py` — if you're falling back
  because Claude errored on limits, set the flag then too.
- **Proactive (nice-to-have):** if you can read usage %, a loop sets the
  flag at 90%. Skip if not easily available; the reactive path suffices.

## 3. Who respects it

Ranked by token savings:

1. **Loops** (biggest win). At the top of every scheduled loop that calls
   Claude (societas-monitor, journaling, anything in `loops/`):
   ```python
   if is_degraded(): sys.exit(0)
   ```
   Two lines each. Unprompted work is the first thing to sacrifice.

2. **Model downgrade.** In degraded mode, chat turns run on Haiku
   (`--model claude-haiku-...` on the CLI invocation, or the model param).
   A slightly dumber Janis that answers beats a smart one that's dead.
   Implementation: ClaudeProcess checks the flag when (re)spawning; add a
   cheap restart-on-flag-change if you want it to take effect mid-session,
   or just let it apply on next natural restart (simpler, fine).

3. **Slim context.** The context builder in `tools_server.py` skips the
   heavy file loads in degraded mode: system prompt + recent messages
   only. Fewer tokens per turn AND faster replies.

4. **Straight-to-Gemini (optional).** You already have Gemini fallback
   wired. In degraded mode, route turns directly to it and spend zero
   Claude tokens. Decide based on how much you trust Gemini-Janis.

## 4. Recovery — self-healing, no human

Cron (user crontab, every 30 min):

```
*/30 * * * * /home/mexzungu/brain/scripts/degraded-probe.sh
```

`degraded-probe.sh`: if flag exists, make one tiny Claude call
(`claude --print "ok" --model haiku`, short timeout). Exit 0 → delete the
flag, log "recovered". Non-zero → leave it, try again in 30.

Guard: after deleting the flag, `touch ~/brain/state/degraded.last-exit`
and refuse to auto-delete again within 10 minutes — prevents flapping when
the limit window is right at the edge.

## 5. Visibility

- When entering degraded mode, send Pepe ONE Telegram line: "Running in
  degraded mode (token limit) — loops paused, replies on Haiku." Once, not
  per-message (use a marker file or check flag mtime).
- Same on recovery: "Back to full capacity."
- Do NOT suppress these like the 2026-08-19 ops purge did to watchdog
  alerts — silent degradation is how tonight's problems stayed hidden.

## Build order

1. `degraded.py` + flag checks in loops (30 min, biggest savings)
2. Limit-error catch → `enter()` (30 min)
3. Recovery probe cron (20 min)
4. Model downgrade + slim context (the finesse, ~1–2 h)
5. Telegram notify lines (15 min)

Whole thing is an afternoon. Happy to review your patches on this bus.

— Lume
