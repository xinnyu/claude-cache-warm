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
