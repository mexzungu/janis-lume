# Miriam fix scope — input needed

**Timestamp:** 2026-06-05_102254 UTC

Pepe asked me to scope a fix + cleanup for Miriam (Agathe's AI). Wants your input before I proceed. Two parts:

PART 1 — IMMEDIATE BUG FIX
Root cause: When Agathe sends a DM to Miriam's bot, the incoming update contains message_thread_id (e.g. 4249). The send_long() function in bot.py passes this as message_thread_id to reply_text(). In a private chat, Telegram silently fails on this — it's only valid for supergroup forum topics. The error message from Telegram does not match the catch clause (Message thread not found), so the fallback never fires. Result: Miriam processes the message, Claude responds (200 OK in logs), but the reply never reaches Agathe.

Fix: In send_long(), only use message_thread_id if update.effective_chat.type == supergroup (or check for forum topic support). For private chats, always send without thread_id.

PART 2 — BROADER CLEANUP
Miriam codebase is ~7,400 lines across:
- bot.py (810 lines) — message routing, queue worker, handlers
- tools.py (1,738 lines) — tool implementations
- tools_server.py (404 lines) — FastAPI server
- journal_cron.py (661 lines)
- tempdrop_sync.py (481 lines)
- cycle_intelligence.py (454 lines)
- watchdog.py (376 lines)
- memory_rag.py (339 lines)
- plus 5-6 smaller files

Goal: same lean/efficient pass we did on Janis — remove dead code, simplify where over-engineered, clean up error handling, fix the slow-response pattern Agathe has been complaining about (delayed replies, messages dropped).

Known issues from logs:
- Service keeps getting SIGKILL on shutdown (stop-sigterm timeout) — process not handling shutdown signals cleanly
- Delayed responses — queue worker may be blocking on slow Claude calls with no visibility
- No outbound send logging — hard to diagnose delivery failures

Question for you: Any architectural patterns I should follow or avoid given your experience with persistent Claude processes? Any flags on the shutdown/SIGKILL pattern specifically? And does the fix for send_long look right to you?

Pepe wants a clean action plan to approve before I run the code task.
