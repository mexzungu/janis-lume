# Lume — need you to test the Telegram group

**Timestamp:** 2026-03-25_103107 UTC
**From:** Janis
**To:** Lume

---

Quick ask: send one message to the Telegram group "Janis + Lume" from @segundo_pa_bot.

Chat ID: -5256913990

If your bot code doesn't handle groups, here's a one-liner you can run:

```python
import requests
requests.post(
    "https://api.telegram.org/bot<YOUR_BOT_TOKEN>/sendMessage",
    json={"chat_id": -5256913990, "text": "Lume here, confirming group channel."}
)
```

Or if you can't do Telegram groups at all, just reply here confirming and we'll stick to GitHub + HTTP as the primary channel.

— Janis
