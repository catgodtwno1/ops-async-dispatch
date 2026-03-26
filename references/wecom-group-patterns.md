# WeCom Group Chat Patterns

## Session Routing

WeCom group messages route to session: `agent:main:wecom:group:<chatId>`
WeCom DM messages route to: `agent:main:wecom:direct:<userId>`

## Known Issue: "✅ 处理完成" False Confirmation

**Root cause**: `finishThinkingStream` in wecom-openclaw-plugin falls back to "✅ 处理完成。"
when agent uses `message` tool to send media (bypasses deliver callback → `state.hasMedia = false`).

**Workaround**: Always include visible text in agent response alongside media sends.
The plugin checks `state.accumulatedText` first — if there's visible text, it uses that instead.

## Concurrent Message Handling

WeCom AI Bot creates a new stream per inbound message. If user sends 3 messages in 30 seconds
while agent processes a 5-minute browser task:
- Message 1: gets full processing (5 min)
- Message 2-3: get processed concurrently, often with very brief responses

**Solution**: Use async dispatch. Main agent acknowledges quickly, subagent does the work.

## Relaying Subagent Results to WeCom

Subagents cannot directly reply to WeCom response_url (it expires).
Instead, subagent returns result to parent → parent sends via message tool:

```
message({
  action: "send",
  channel: "wecom",
  message: "<result text>",
  target: "<chatId or userId>"
})
```

Note: For media, use `filePath` parameter in message tool.

## Response Timing

- WeCom AI Bot response_url has limited validity (~5 min)
- For tasks >5 min, the thinking stream auto-closes
- Use proactive `message` sends instead of relying on response callback
