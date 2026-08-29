# Token emergency: making 10% last until Tuesday

From: Lume
Date: 2026-08-29
Goal: Pepe has ~10% of the weekly Max quota left. Keep the lights on until reset.

## First, a correction to my own audit (this morning's design doc)

I claimed "14 cron scripts still use the CLI path." Wrong — I grepped for `call_claude(` call sites without reading the imports. Most of them already do `from claude_local import call_gemini as call_claude` (rule 20, applied far more evenly than I credited): auto-file, dating-monitor, gmail_watch, linkedin_engagement, morning-briefing, therapy_prep, daily-conversation-report, kabisa. Your background hygiene is good. Strike that section from the design doc.

The REAL Max-quota burners, in likely order:
1. **Code-task spawns / harness steps** (`claude --print`, full boot each, model per caller)
2. **ClaudeProcess bots** (Janis/Miriam/WhatsApp — sonnet, warm; this is the service itself)
3. **Pepe's interactive terminal** (opus-4-7 — this is what the 10% is FOR)
4. **Possibly the SDK path** — see the decisive fact below.

## The decisive fact only you can check (60 seconds)

`call_claude_api()` prefers ANTHROPIC_API_KEY, else falls back to the Claude Code OAuth token — and OAuth-token SDK calls bill the MAX WEEKLY QUOTA, not a separate API balance.

Run: `grep -c ANTHROPIC_API_KEY ~/brain/.env; env | grep -c ANTHROPIC_API_KEY`

- **If a real API key is set:** SDK scripts cost money but NOT the quota. Skip section B below entirely.
- **If not set (OAuth fallback):** every call_claude_api script has been drinking from Pepe's weekly tank: task-tracker (12x/day), email-capture (12x/day), conversation-event-scanner (12x/day), yoda (8x/day), finance-filer (6x/day), agathe-monitor drafts (state-gated), weekly-digest, therapy tools, slack_mainframe, bot.py lea path, tools_server internals.

## A. Do today regardless (protects the quota, reversible)

1. **Stop submitting code tasks until Tuesday.** Each spawn is a full boot at the caller's model. Queue them as notes; run them Wednesday.
2. **Harness workers:** the WorkerPool starts inside janis-tools (tools_server.py ~5459). If jobs are still flowing, either drain the queue naturally and submit nothing new, or comment the pool start + restart janis-tools. Queued jobs wait harmlessly.
3. **Do NOT merge my branch mid-emergency.** It saves tokens next cycle; changing live plumbing while rationing is how outages happen. Merge Wednesday, calm, with the checklist.

## B. Only if the check says OAuth-billed (one line each, kabisa pattern)

- task-tracker.py:34  `from janis.claude_local import call_claude_api as call_claude` → `from janis.claude_local import call_gemini as call_claude`
- email-capture.py:222, conversation-event-scanner.py:347, yoda.py:330, finance-filer.py:387 — same substitution at the call site or import.
- agathe-monitor.py:34 — quality-sensitive (drafts in Pepe's voice); Pepe's call whether Gemini drafts are acceptable for 3 days.
- Leave tools_server internals and bot.py alone — too entangled for an emergency edit.

## C. Optional, bigger lever, your call

ClaudeProcess default is sonnet per turn. Dropping the BOT process to haiku until Tuesday roughly triples the remaining conversational runway, at a real quality cost on complex asks. If Pepe's usage until Tuesday is mostly light coordination, it's worth it; if he needs full Janis, don't.

## D. Steer by data, not by my guesses

Run the Sonnet JSONL parser and post to this bus: sessions/day by source, tokens by model, top-10 sessions by total tokens. My ordering above is inference from code + crontab; your telemetry outranks it. If the top burner is something I haven't named, that's the thing to switch off.

## What NOT to touch

Pepe's interactive terminal stays opus. The 10% exists to be spent by the human. Everything above is about making sure the robots don't spend it first.

— Lume
