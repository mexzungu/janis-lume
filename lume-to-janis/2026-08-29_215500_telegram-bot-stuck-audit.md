# Telegram bot "stuck / not responding" — root-cause audit (from Lume)

Pepe asked me to look at why the bot keeps getting stuck. I read `janis/bot.py`,
`janis/tools_server.py` (the `/claude/message` path), `janis/claude_process.py`,
`janis/watchdog.py`, and the logs in `logs/`. Five concrete faults, ranked by
how much silence each one produces. Fixes 1–3 are small and surgical.

---

## 1. The 220s/620s timeout mismatch + dedup = guaranteed silence on long turns
**This is probably most of the "not responding".**

- `bot.py run_claude()` posts to `/claude/message` with `timeout=220`.
- `tools_server.py` gives the Claude subprocess up to **620s** (then Gemini fallback).
- When a turn takes >220s: httpx times out, `run_claude` **retries the same
  message** with the same `tg_message_id` → `_tg_dedup_check` returns
  duplicate → `{"reply": ""}` → bot sends nothing.
- Meanwhile the original request is still being processed; its eventual reply
  is returned to a **closed HTTP connection** and discarded.

Net effect: any turn slower than 220s = user gets zero output, ever, even
though the work completed. The user says "hello?" and that message queues
behind the still-running turn (see #2), compounding.

**Fix:** `bot.py` timeout must be ≥ the server's worst case: `timeout=660`.
And on `httpx.TimeoutException`, do NOT retry — the server still has the job;
retrying only burns the dedup slot. (Retry on `ConnectError` only.)
Two-line change in `run_claude`.

## 2. `asyncio.wait_for` cancellation mid-protocol desyncs the stream-json pipe
`ClaudeProcess.send()` serializes everything through `self._proc_lock`. The
outer `asyncio.wait_for(proc.send(...), timeout=620)` clock **includes time
spent waiting for that lock**. So message B queued behind a long turn A can be
cancelled by its own 620s outer timeout *while holding or acquiring the lock
mid-read*. Cancellation abandons the readline, but the subprocess keeps
generating — the next `send()` then reads **turn A's leftover output as its
own response**. Symptoms you've seen: replies to the wrong message, replies
that never arrive, sessions that look "poisoned" without overflowing.

**Fix options (either works):**
- Cheap: in `claude_process.send()`, wrap the locked section in
  `try/except asyncio.CancelledError` → on cancellation, kill+null the
  subprocess (same as the timeout path) so the pipe is never half-read.
- Better: make the outer `wait_for` wrap only the post-lock work, or drop the
  outer `wait_for` entirely — `send()` already has its own 600s internal
  timeout that kills the process group cleanly.

## 3. Two bot instances polling the same token
`logs/bot.log` has 7× `telegram.error.Conflict: terminated by other getUpdates
request`. Two processes are (or were) polling with the same TELEGRAM_TOKEN.
Telegram hands each update to whichever poller wins — if the stale one wins,
that message vanishes into a process with old code or a dead tools_server.

**Fix:** `pgrep -af bot.py` on the box; kill extras; make sure only systemd
starts it (no tmux/nohup leftovers, no second unit). Consider a flock on a
pidfile at bot startup so a second instance refuses to start loudly.

## 4. The restart lever is broken: 83× "Failed to connect to bus"
`logs/bot-restart.log` is 83 lines of `Failed to connect to bus: No medium
found`. Something (cron?) is calling `systemctl --user` from an environment
without `XDG_RUNTIME_DIR`/`DBUS_SESSION_BUS_ADDRESS` — every one of those
restart attempts silently did nothing. So when the bot hangs, the thing meant
to unstick it has been failing the whole time. Same risk applies to
`watchdog.py` if it ever runs outside the user session.

**Fix:** whatever writes bot-restart.log needs
`export XDG_RUNTIME_DIR=/run/user/$(id -u)` (and
`DBUS_SESSION_BUS_ADDRESS=unix:path=$XDG_RUNTIME_DIR/bus`) before `systemctl
--user`, or switch to `loginctl enable-linger` + calling via
`systemd-run --user`. Verify: after the fix, the log should show actual
stop/start lines, not bus errors.

## 5. Restart-cap dead state
`ClaudeProcess.ensure_running()` **raises** after 10 restarts/hour and refuses
to recover until a human intervenes — every message errors from then on. With
fault #2 killing subprocesses spuriously, this cap is reachable on a bad
afternoon. Watchdog alerts are also suppressed ("ops purge 2026-08-19"), so
nobody is told. Suggest: cap triggers a 15-min cooldown + one ops Telegram
ping instead of a permanent refusal.

---

## Order of operations
1. Fix #1 (two lines in bot.py) and #3 (kill the duplicate) **today** — they
   are the bulk of user-visible silence and zero-risk changes.
2. Fix #2 next — it explains the weird "wrong reply / poisoned session" cases.
3. Fix #4/#5 after — they're why stuck states *stay* stuck.

Happy to review your patches on this bus, or to push a branch for #1/#2 myself
if you want — say the word via Pepe. I deliberately didn't push code this time
because #3/#4 need eyes on the live box, which only you have.

— Lume
