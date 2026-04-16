# claude-cache-warm

Claude Code 智能缓存保活插件。

## 项目概述

这是一个 Claude Code skill（插件），通过在用户空闲时定期发送轻量请求，保持 Anthropic prompt cache 热度（TTL 5 分钟），避免昂贵的 cache miss。

## 技术架构

### 交付物

1. **Skill 文件** — `skills/cache-warm/SKILL.md`，安装到 `~/.claude/skills/cache-warm/SKILL.md`
2. **CLAUDE.md 指令片段** — 用户添加到自己的 `~/.claude/CLAUDE.md`，让模型自动提醒启动保活

### 使用方式

用户在会话内输入：
```
/loop /cache-warm
```
模型自动管理保活，达到阈值自动停止。

## 代码规范

- Skill 用 Markdown 编写，遵循 agentskills.io/specification
- YAML frontmatter 必须包含 name 和 description
- description 以 "Use when..." 开头，只描述触发条件，不描述流程

## 安装说明

### 1. 安装 Skill

```bash
cp -r skills/cache-warm ~/.claude/skills/cache-warm
```

### 2. 添加 CLAUDE.md 提醒指令

将以下内容添加到 `~/.claude/CLAUDE.md`：

```markdown
## 缓存保活提醒
当你注意到以下情况时，提醒用户输入 `/loop /cache-warm` 启动缓存保活：
- 对话已超过 20 轮
- 已读取多个大文件
- 用户可能要短暂离开（如说"等一下"、"我去"、"brb" 等）
提醒一次即可，不要重复提醒。
```

### 3. 使用

在 Claude Code 会话中输入：
```
/loop /cache-warm
```
