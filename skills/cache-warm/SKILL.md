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

### 1. 确定当前保活计数 N

回看对话历史，统计自**最近一条用户消息**之后，你输出了多少条 `保活 #X/20` 状态行。这个数量就是当前 N。

为什么这样做：没有外部存储，对话历史本身就是状态。每条保活输出都是一条记录。

- 如果最近的用户消息之后没有保活记录 → N = 0（首次或刚重置）
- 如果有 K 条保活记录 → N = K

然后 N++（本次计入）。

### 2. 检查是否达到阈值

- **N >= 20** → 停止 loop（不再调用 ScheduleWakeup），输出停止信息：
  ```
  保活已停止 | 共 {N} 次 / {N*270/60:.0f}min | 总预估节省 ~${N * 32.25:.2f}/M tokens
  ```
  （32.25 = miss 成本 33.75 - hit 成本 1.50，为每次保活的净省金额/M tokens）

- **N < 20** → 继续保活，输出状态后调用 ScheduleWakeup

### 3. 输出状态（一行）

```
保活 #{N}/20 | 已运行 ~{N*270/60:.0f}min | 预估已省 ~${N * 32.25:.2f}/M tokens
```

这行输出同时也是下次唤醒时的计数依据 — 对话历史就是持久化存储。

### 4. 调度下次唤醒

调用 ScheduleWakeup：
- `delaySeconds`: 270
- `reason`: `"cache keep-alive #{N}/20"`
- `prompt`: `/cache-warm`

## 关键规则

- **输出极简**：每次只输出一行状态，不要多余解释
- **270s 间隔**：卡在 5 分钟 TTL（300s）内，留 30s 余量
- **20 次阈值**：超过 22 次就亏（22.5 为盈亏平衡），20 次留 2 次余量，约 90 分钟
- **用户消息重置**：用户发消息本身刷新缓存，只计最近用户消息之后的保活记录
- **对话历史即状态**：N 通过回看自己之前输出的 `保活 #X/20` 行来确定，无需外部存储
- **达到阈值必须停止**：不再调用 ScheduleWakeup，loop 自然结束
