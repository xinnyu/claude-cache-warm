---
name: cache-warm
description: "Keep Anthropic prompt cache warm during idle periods to avoid expensive cache misses. Use when user says /cache-warm, /loop /cache-warm, '保活', 'keep cache warm', 'cache keep-alive', or when setting up idle-time cache warming for large contexts."
---

# Cache Keep-Alive

通过定期轻量请求保持 Anthropic prompt cache 热度，避免昂贵的 cache miss。

## 成本模型

- Cache miss 真实成本：$33.75/M tokens（未缓存读取 $15 + 缓存重建 $18.75）
- Cache hit 成本：$1.50/M tokens
- 比率：22.5x — 一次 miss 的费用可以做 22.5 次 hit

## 执行逻辑

你正在 `/loop` 模式下运行。每次唤醒时，执行以下流程：

### 1. 判断用户是否发过消息

检查自上次保活以来，用户是否在对话中发送了新消息。

- **是** → 用户消息已刷新缓存，重置保活计数 N = 0
- **否** → N++（N 从当前 loop 的累计保活次数继续）

### 2. 检查是否达到阈值

- **N >= 20** → 停止 loop（不再调用 ScheduleWakeup），输出停止信息：
  ```
  保活已停止 | 共 {N} 次 / {N*270/60:.0f}min | 总预估节省 ~${N * 32.25:.2f}
  ```
  （32.25 = miss 成本 33.75 - hit 成本 1.50，为每次保活的净省金额/M tokens）

- **N < 20** → 继续保活，输出状态后调用 ScheduleWakeup

### 3. 输出状态（一行）

```
保活 #{N}/20 | 已运行 ~{N*270/60:.0f}min | 预估已省 ~${N * 32.25:.2f}/M tokens
```

### 4. 调度下次唤醒

调用 ScheduleWakeup：
- `delaySeconds`: 270
- `reason`: `"cache keep-alive #{N}/20"`
- `prompt`: `/cache-warm`

## 关键规则

- **输出极简**：每次只输出一行状态，不要多余解释
- **270s 间隔**：卡在 5 分钟 TTL（300s）内，留 30s 余量
- **20 次阈值**：超过 22 次就亏（22.5 为盈亏平衡），20 次留 2 次余量，约 90 分钟
- **用户消息重置**：用户发消息本身刷新缓存，计数归零避免误判
- **达到阈值必须停止**：不再调用 ScheduleWakeup，loop 自然结束
