---
name: cache-warm
description: "Keep Anthropic prompt cache warm during idle periods to avoid expensive cache misses. Use when user says /cache-warm, /loop /cache-warm, '保活', 'keep cache warm', 'cache keep-alive', or when setting up idle-time cache warming for large contexts."
---

# Cache Keep-Alive

每 270s 发一次轻量请求，保持 prompt cache 热度（TTL 5min），避免 22.5x 的 cache miss 成本。

## 每次唤醒执行

1. **计数**：数对话中最近用户消息之后有几条 `[cw:X]` 格式的行 → 得到已保活次数 K，本次 N = K + 1
2. **阈值**：N >= 20 → 输出 `[cw:stop]` → 不调用 ScheduleWakeup → 结束
3. **输出**：仅一条 → `[cw:N]`（这也是下次计数依据，如 `[cw:3]`）
4. **调度**：ScheduleWakeup(delaySeconds=270, reason="cache-warm N/20", prompt="/cache-warm")

输出必须极简。除上述格式外不要输出任何其他文字。每多一个输出 token = $75/M 的浪费。
