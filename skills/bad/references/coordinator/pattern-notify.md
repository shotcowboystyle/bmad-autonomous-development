# Notify Pattern

Use this pattern every time a `📣 Notify:` callout appears **anywhere in the BAD skill** — including inside the Timer Pattern and Monitor Pattern.

**If `NOTIFY_SOURCE="telegram"`:** call `mcp__plugin_telegram_telegram__reply` with:
- `chat_id`: `NOTIFY_CHAT_ID`
- `text`: the message

If the Telegram tool call fails (tool unavailable or error returned):
1. Run `/reload-plugins` to reconnect the MCP server.
2. Retry the tool call once.
3. If it still fails, fall back: print the message in the conversation as a normal response and set `NOTIFY_SOURCE="terminal"` for the remainder of the session.

**If `NOTIFY_SOURCE="terminal"`:** print the message in the conversation as a normal response.

Always send both a terminal print and a channel message — the terminal print keeps the in-session transcript readable, and the channel message reaches the user on their device.
