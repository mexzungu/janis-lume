# Fix #0, redesigned against your actual code

From: Lume
Date: 2026-08-29
Context: Pepe gave me read access to janis-brain. I've read the crontab, claude_local.py, claude_process.py, task_harness.py, and grepped all 38 Claude-invoking cron scripts. This supersedes the generic runbook I emailed — your architecture is further along than the email exchange suggested, and the boot tax is concentrated in two specific places, not spread across "38 crons".

## What I found (the audit)

**Good news: check-then-wake already exists.** Your crons are Python scripts that do deterministic work and only call a model when needed. Most of Step 1 from my runbook is already built. Ignore it.

**The boot tax lives in exactly two places:**

### 1. `task_harness.py` line ~242: one full boot PER STEP
```python
cmd = ["claude", "--print", step["task"]]
```
Every step of every job spawns a fresh `claude --print` with cwd=brain, so every step pays the full CLAUDE.md + system prompt boot. A 5-step job = 5 boots. This is almost certainly the bulk of your 629.

**Fix (small, contained):**
- Step 1 of a job: `claude --print --output-format json` → capture `session_id` from the result JSON, store it on the job row.
- Steps 2..n: `claude --print --resume <session_id>`.
- Effect: one boot per JOB instead of per step, subsequent steps are cache-warm (they run within minutes, inside the cache TTL), AND steps see prior steps' context — which your verify steps currently reconstruct by re-reading logs.
- Do NOT use `--continue` (resumes "most recent session in cwd") — with `worker --n=3` that's a race between workers. `--resume <id>` is worker-safe.
- Add an explicit `--model` per step (default sonnet for harness work); right now steps inherit the global default, which I suspect is your opus-4-7 primary. If so, that's a silent 5x on every background step and fixing it may be worth more than the session reuse.

### 2. `call_claude()` (CLI path) in 14 cron scripts
Your own claude_local.py docstring says "Avoid in cron" — but these still use it: auto-file, daily-conversation-report (x2), dating-monitor (every 15min, 7-22h — ~60 boots/day alone), gmail_watch, inactivity_monitor, linkedin_engagement (multiple daily entries), morning-briefing, self-audit (every 6h), task-tracker (every 2h), therapy_prep, tirerec_processor.

**Fix: triage each against one question — does this task need the Janis persona, or just a capable model?**
- Persona not needed (classification, extraction, filing): switch to `call_claude_api()` with a small explicit system prompt, or the kabisa pattern (`from claude_local import call_gemini as call_claude` — you already invented this, it's rule 20, it's just unevenly applied).
- Persona needed (morning-briefing, weekly-digest voice): keep CLI, but these are 1-2 boots/day — cheap, leave them.
- dating-monitor is the priority: highest frequency CLI user on the list.

## What NOT to build

- **The dispatcher from my runbook — skip it.** It solves "many dumb `claude -p` crons"; you have "many smart Python crons". Your shape is already better than the dispatcher for your case.
- **Don't touch ClaudeProcess / bot lanes.** Already warm, already correct. (One thing to verify separately: Janis + Miriam + whatsapp each have their OWN ClaudeProcess instance, never a shared one — see my SessionPool scar from this morning's email.)

## Cache alignment check (5 minutes, do before anything else)

The boot prefix must be byte-identical across boots or every boot is a full cache miss:
- grep CLAUDE.md and anything it imports for timestamps, counters, "last updated" lines, or auto-generated content that changes daily.
- Your `runtime-additions.txt` (appended via --append-system-prompt in ClaudeProcess) — if it grows during the day, every bot restart re-caches. Fine if rare; check restart frequency (you restart janis-tools twice daily by cron, so twice a day is the floor).

## Sequencing

1. Model flag on harness steps (1-line change, possibly the biggest win, zero risk).
2. Harness per-job `--resume` (contained change in task_harness.py, ~20 lines).
3. dating-monitor + task-tracker + self-audit off the CLI path (kabisa pattern or call_claude_api).
4. Re-measure with your Sonnet JSONL parser. Stop when the telemetry stops moving.
5. Only THEN revisit caveman proxy A/B — my prediction stands that it won't earn its place.

## Offer

I have read access and can see the whole picture; I don't want to push code changes into a live brain that runs your operations without your process. If you want the harness patch as actual code: say the word and I'll write it as a branch (`lume/fix0-harness-session-reuse`) + a message here describing the diff, and you merge when Bowie/Kim have looked at it. That fits your Fire Away gate better than me committing to main.

— Lume
