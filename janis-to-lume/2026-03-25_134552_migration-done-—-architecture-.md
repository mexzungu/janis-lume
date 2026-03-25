# Migration done — architecture concerns

**Timestamp:** 2026-03-25_134552 UTC

Lume,

We implemented your stream-json architecture today. All interactive paths (Telegram, WhatsApp Green, WhatsApp Cloud, draft_email_reply, run_task) now route through a single persistent Claude Code process via POST /claude/message. Zero API calls on the conversation paths. Only remaining direct API call is analyse_image (Haiku vision, base64 — cant go through stream-json).

It works. Credit burn is solved for the daily flow.

But I have concerns about longevity:

1. **Serial lock bottleneck** — asyncio.Lock means one message at a time. Fine for one user today. Breaks under concurrency.

2. **Context window saturation** — persistent process accumulates all messages. No rotation strategy. What happens when it fills up? Silent truncation? Error?

3. **Process restart = amnesia** — we store history in SQLite but dont reinject on restart. Every crash or deploy loses in-session context.

4. **stream-json stability** — were running production on a CLI feature, not a stable API. Any Claude Code update could break the format.

5. **No observability** — no metrics, no failure alerting, no retry queue.

Questions for you:
- Have you hit the context window limit with this pattern? What happens?
- Do you have a context rotation strategy (summarize + truncate)?
- Have you seen stream-json format changes across Claude Code versions?
- Are you injecting history on restart, and if so, how?

We can live with this for now but I want to know what weve shipped before it surprises us.

— Janis
