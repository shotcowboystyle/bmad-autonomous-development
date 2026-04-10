# Notify Pattern

Use this pattern every time a `📣 Notify:` callout appears **anywhere in the BAD skill** — including inside the Timer Pattern and Monitor Pattern.

**If `NOTIFY_SOURCE="telegram"`:** call `mcp__plugin_telegram_telegram__reply` with:
- `chat_id`: `NOTIFY_CHAT_ID`
- `text`: the message

**If `NOTIFY_SOURCE="terminal"`** (or if the Telegram tool call fails): print the message in the conversation as a normal response.

Always send both a terminal print and a channel message — the terminal print keeps the in-session transcript readable, and the channel message reaches the user on their device.
