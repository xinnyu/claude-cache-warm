---
name: cache-warm
description: "Keep Anthropic prompt cache warm during idle periods to avoid expensive cache misses. Use when user says /cache-warm, /loop /cache-warm, '保活', 'keep cache warm', 'cache keep-alive', or when setting up idle-time cache warming for large contexts."
---

# Cache Keep-Alive

每 270s 发一次轻量请求，保持 prompt cache 热度（TTL 5min），避免 22.5x 的 cache miss 成本。

## 每次唤醒执行

### Step 1 — 计数 N

回看对话历史，数所有 `[cw:` 开头的 assistant 输出行，得到 K。

但如果中间出现了**真实用户消息**（不是 `/cache-warm` 这个 loop prompt），则只从那条用户消息之后开始数。loop prompt `/cache-warm` 不算用户消息，跳过它。

本次 N = K + 1。

### Step 2 — 执行

**如果 N >= 20（达到阈值）：**
- 输出 `[cw:stop] 保活结束，共 ~90min`
- 不调用 ScheduleWakeup，loop 自然结束

**如果 N < 20（继续保活）：**
- 输出 `[cw:N/20] ~{N*4.5:.0f}min`（如 `[cw:3/20] ~14min`）
- 调用 ScheduleWakeup(delaySeconds=270, reason="cache-warm N/20", prompt="/cache-warm")

输出只用上述格式，不要多余文字。每多一个输出 token = $75/M 的浪费。
