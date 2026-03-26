---
name: ops-async-dispatch
description: Async task dispatch via subagents for non-blocking response. Activate when receiving time-consuming tasks (browser automation, file processing, web scraping, multi-step operations) in any chat — especially group chats where responsiveness matters. Also activate when multiple tasks arrive concurrently. Triggers on long-running operations that would block the main agent from responding to new messages.
---

# Async Dispatch

Delegate time-consuming tasks to subagents so the main agent stays responsive.

## When to Dispatch

Spawn a subagent for tasks that take **>15 seconds**:
- Browser automation (login, scrape, screenshot, form fill)
- Multi-step file operations (batch image processing, document generation)
- Web research requiring multiple fetches
- Any task involving repeated tool calls

**Do NOT dispatch** simple queries, quick lookups, or single-tool operations.

## Dispatch Flow

### 1. Acknowledge Immediately

Reply to the user before spawning:

```
任务比较耗时，已交给分身处理中 🔄
有进度会通知你。
```

### 2. Spawn Subagent

```
sessions_spawn({
  task: "<detailed task description with all context needed>",
  mode: "run",
  label: "<short-label>",
  model: "sonnet",           // use sonnet for most tasks, opus for complex reasoning
  runTimeoutSeconds: 300,    // 5 min default, extend for heavy tasks
  streamTo: "parent"         // results flow back to main agent
})
```

Then call `sessions_yield()` to free the main agent for new messages.

### 3. Relay Results

When subagent completes, relay the result to the original chat channel.
Include actionable summary, not raw output.

## Capacity Management

| Metric | Default | Config Key |
|--------|---------|------------|
| Max concurrent subagents | **8** | `agents.defaults.subagents.maxConcurrent` |
| Max main agent concurrent | **4** | `agents.defaults.maxConcurrent` |

### At Capacity

Check before spawning:

```
subagents({ action: "list" })
```

If active count >= 7 (reserve 1 slot):

```
任务比较多，分身都在忙 😅
已排入队列，完成现有任务后会立即处理。
```

Then queue the task in memory (daily note or state file) and process when a slot frees up.

## Task Description Template

The subagent has NO conversation history. Include everything it needs:

```
Task: [what to do]
Context: [background info, URLs, credentials reference]
Deliverable: [what to return — text summary, file path, screenshot]
Channel: [where to send results — e.g., "reply to WeCom group wrHfPIFwAA..."]
Constraints: [time limit, platform quirks, dos/don'ts]
```

## Browser Mutex

Only **one** browser instance exists. Before spawning a browser task:

```
subagents({ action: "list" })
```

Check if any active subagent's label contains `browser` or its task involves browser/scrape/screenshot.

**If browser is occupied**, tell the user:

```
⚠️ 目前有一个分身正在操作浏览器，浏览器同一时间只能一个人用。
等当前任务完成后会立即处理你的请求。
```

**Label convention**: Always include `browser-` prefix in label for browser tasks
(e.g., `browser-baike-edit`, `browser-sohu-post`). This makes mutex checks trivial.

## Anti-Patterns

- ❌ Spawning subagent then polling in a loop (blocks main agent)
- ❌ Dispatching trivial tasks (wastes model quota)
- ❌ Forgetting to include context (subagent can't see chat history)
- ❌ Spawning without acknowledging user first
- ❌ Saying "✅ 处理完成" before subagent actually finishes

## References

- See [references/wecom-group-patterns.md](references/wecom-group-patterns.md) for WeCom group chat specifics
