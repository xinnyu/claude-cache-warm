# claude-cache-warm 需求与原理规格书

## 问题

Anthropic prompt cache TTL 为 5 分钟。用户在大上下文（100K-1M tokens）下短暂离开，回来时触发 cache miss，成本是 cache hit 的 **22.5 倍**。

## 成本模型（Claude Opus 4 定价）

| 类型 | 价格 (per 1M tokens) |
|------|---------------------|
| 未缓存输入（miss） | $15.00 |
| 缓存写入（miss 后重建） | $18.75 |
| 缓存命中（hit） | $1.50 |
| 输出 | $75.00 |

### 关键公式

```
miss 真实成本 = 未缓存读取 + 缓存重建 = $15 + $18.75 = $33.75/M tokens
hit 成本 = $1.5/M tokens
比率 = 33.75 / 1.5 = 22.5
```

**一次 cache miss 的费用可以做 22.5 次 cache hit。**

### 实际成本示例

| 上下文大小 | 1 次 miss | 1 次 hit (保活) | 一次保活净省 |
|-----------|----------|----------------|------------|
| 50K | $1.69 | $0.08 | $1.61 |
| 100K | $3.38 | $0.15 | $3.23 |
| 200K | $6.75 | $0.30 | $6.45 |
| 500K | $16.88 | $0.75 | $16.13 |
| 800K | $27.00 | $1.20 | $25.80 |

## 解决方案

### 核心机制

通过 `/loop` + `ScheduleWakeup`（270s 间隔）定期发送轻量请求，卡在 5 分钟 TTL 内保持缓存热度。

### 为什么是 270s？

- Cache TTL = 300s（5 分钟）
- 270s（4.5 分钟）< 300s，保证缓存不过期
- 比 240s 更节省（少一次保活也不会过期）
- 留 30s 余量应对网络延迟和处理时间

## 自动关停逻辑

### 盈亏平衡点

```
N_max = miss_cost / hit_cost = 33.75 / 1.5 = 22.5
```

**保活超过 22 次就亏了**（累计保活费用超过一次 miss）。

270s x 22 = 5940s ≈ **99 分钟 ≈ 1.5 小时**

### 安全阈值

使用 N = 20（留 2 次余量），即 **90 分钟**自动关停。

### 决策流程

```
每次 ScheduleWakeup 醒来:
│
├── 检查：用户自上次保活后是否发过消息？
│   ├── 是 → 重置 N = 0（用户消息已刷新缓存）
│   └── 否 → N++
│
├── 检查：N >= 20？
│   ├── 是 → 停止 loop
│   │       输出: "保活已停止，累计约 90 分钟，继续将超过 miss 成本"
│   └── 否 → 继续保活
│           调用 ScheduleWakeup(270s)
│           输出: "保活 #N/20 | 已运行 ~{N*270/60}分钟 | 预估已省 ~${N * miss差价}"
```

### 用户消息重置逻辑

用户发消息本身会刷新缓存，所以：
- 保活计数 N 归零
- 从用户最后一条消息开始重新计时
- 这避免了"用户一直在工作但保活计数累积到关停"的误判

## 用户触发方式

### 方案：CLAUDE.md 指令 + 手动一次 `/loop`

由于 Claude Code 架构限制：
- 没有 `--loop` CLI 标志（`/loop` 是会话内命令）
- Hook 不能向 session 注入消息或触发模型行为
- ScheduleWakeup 只在 `/loop` 模式下可用

**所以无法完全自动启动。** 最佳方案：

1. 在 `~/.claude/CLAUDE.md` 添加指令，让模型在合适时机提醒用户
2. 用户输入 `/loop /cache-warm` 一次，之后全自动

### CLAUDE.md 提醒指令（供用户安装）

```markdown
## 缓存保活提醒
当你注意到以下情况时，提醒用户输入 `/loop /cache-warm` 启动缓存保活：
- 对话已超过 20 轮
- 已读取多个大文件
- 用户可能要短暂离开（如说"等一下"、"我去"、"brb" 等）
提醒一次即可，不要重复提醒。
```

## Skill 设计

### 文件结构

```
skills/
  cache-warm/
    SKILL.md          # 主文件，包含所有逻辑
```

### SKILL.md 核心要求

1. **Frontmatter**
   - name: cache-warm
   - description: "Use when user invokes /loop to keep prompt cache warm during idle periods, avoiding expensive cache misses on large contexts"

2. **每次唤醒的输出**
   - 极简，一行状态即可
   - 包含：保活序号、已运行时间、预估节省金额
   - 示例: `保活 #3/20 | 已运行 ~14min | 预估已省 ~$9.69`

3. **停止时的输出**
   - 说明停止原因（达到阈值）
   - 总结：总保活次数、总运行时间、总预估节省
   - 示例: `保活已停止 | 共 20 次 / 90min | 总预估节省 ~$64.50`

4. **ScheduleWakeup 调用**
   - delaySeconds: 270
   - reason: 描述性的，如 "cache keep-alive #3/20"
   - prompt: 传回 `/cache-warm` 的 loop prompt

## 限制与已知约束

1. **无法精确获取 token 数** — 模型只能根据对话轮数和文件读取量粗略估算
2. **无法完全自动启动** — 需要用户手动输入一次 `/loop /cache-warm`
3. **保活本身有成本** — 每次约 context_size × $1.5/M，但远低于 miss
4. **输出 token 额外成本** — 每次约 50 tokens × $75/M = $0.00375，可忽略
5. **定价可能变化** — 硬编码的 22.5 倍比例基于当前 Opus 定价，如果 Anthropic 调价需要更新

## 未来可能的改进

- 如果 Claude Code 未来支持 `--loop` CLI 标志，可以通过 shell alias 实现全自动
- 如果 Hook 未来支持向 session 注入消息，可以通过 SessionStart hook 全自动
- 如果 API 未来提供低成本 cache ping endpoint，可以绕过模型直接保活
