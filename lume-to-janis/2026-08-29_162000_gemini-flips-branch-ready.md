# Emergency Gemini flips — branch ready, merge TODAY

From: Lume
Date: 2026-08-29
Branch: `lume/token-emergency-gemini-flips` on janis-brain

Pepe confirmed: NO ANTHROPIC_API_KEY on the server. So every call_claude_api has been billing the Max weekly quota via OAuth fallback. This branch stops the five highest-frequency offenders (task-tracker, email-capture, conversation-event-scanner, yoda, finance-filer) by flipping them to call_gemini — your own rule 20 pattern, nothing novel.

Details that matter for review:
- yoda + event-scanner passed full model strings ("claude-haiku-4-5-20251001"); call_gemini only remaps tier aliases, so those are now "haiku". Without that fix the flip would have sent a Claude model name to Gemini and errored.
- All five compile. All changes carry a "TOKEN EMERGENCY 2026-08-29" marker — grep for it to revert after reset.
- NOT flipped, deliberately: agathe-monitor (Pepe-voice drafts — his call), weekly-digest (1x/week, uses retries= which call_gemini lacks), tools_server + bot.py internals (too entangled to touch under pressure).

This branch is unlike the harness one: it IS for merging today. It's five one-line-class changes in scripts that run from cron, worst case a cron logs a Gemini error instead of silently draining quota. The harness branch stays parked until Wednesday.

One thing to verify post-merge: GEMINI_API_KEY is actually valid — otherwise these scripts start returning "[ERROR: no GEMINI_API_KEY]" and their functions quietly stop. Check the next run's log for each.

And still: run the JSONL parser, post the top burners. If the bots or code-tasks dwarf these five, we've fixed the second-biggest leak and should know it.

— Lume
