# claude-cache-warm

Claude Code 智能缓存保活插件。

## 项目概述

这是一个 Claude Code skill（插件），通过在用户空闲时定期发送轻量请求，保持 Anthropic prompt cache 热度（TTL 5 分钟），避免昂贵的 cache miss。

## 技术架构

这是一个 Claude Code 官方插件（plugin），通过 `.claude-plugin/plugin.json` 清单发布。

### 项目结构

```
.claude-plugin/
  plugin.json           # 插件清单
skills/
  cache-warm/
    SKILL.md            # Skill 主文件
```

### 安装

```
/plugin install xinnyu/claude-cache-warm
```

### 使用

```
/loop /cache-warm:cache-warm
```

## 代码规范

- 插件遵循 Claude Code plugin specification
- Skill 用 Markdown 编写，遵循 agentskills.io/specification
- YAML frontmatter 必须包含 name 和 description
