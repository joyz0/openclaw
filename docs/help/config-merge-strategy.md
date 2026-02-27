# OpenClaw 配置文件合并策略及默认值文档

## 配置层次结构

OpenClaw 采用层次化的配置结构，主要包括以下层级：

1. **全局配置**：最外层的配置项，适用于整个系统
2. **agents.defaults**：所有 agent 的默认配置
3. **agents.list**：具体 agent 的配置列表，每个 agent 可以覆盖默认配置

## 配置合并策略

### 1. 基本合并规则

OpenClaw 的配置合并采用**覆盖式继承**策略：

- **优先级**：`agents.list` 中的具体 agent 配置 > `agents.defaults` > 系统默认值
- **合并方式**：当 agent 配置中存在某个配置项时，会完全覆盖默认配置中的对应项
- **部分覆盖**：agent 配置只需指定需要覆盖的部分，未指定的部分会继承默认配置

### 2. 工具配置合并

对于工具配置（`tools`），合并策略如下：

- **全局工具配置**：最外层的 `tools` 配置为全局默认值
- **agent 工具配置**：`agents.list[].tools` 会覆盖全局工具配置
- **工具配置继承**：如果 agent 没有指定 `tools` 配置，则会使用 `agents.defaults.tools`（如果存在），否则使用全局 `tools` 配置

### 3. 模型配置合并

- **模型默认值**：系统为模型配置提供默认值，如 `contextWindow`、`maxTokens` 等
- **模型配置继承**：`agents.defaults.models` 中的配置会应用到所有 agent
- **agent 模型配置**：`agents.list[].model` 可以覆盖默认模型配置

## 默认值说明

### 1. 系统默认值

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `agents.defaults.maxConcurrent` | 10 | 最大并发 agent 数 |
| `agents.defaults.subagents.maxConcurrent` | 5 | 最大并发子 agent 数 |
| `agents.defaults.compaction.mode` | "safeguard" | 上下文压缩模式 |
| `agents.defaults.contextPruning.mode` | "cache-ttl" | 上下文修剪模式 |
| `agents.defaults.contextPruning.ttl` | "1h" | 上下文缓存时间 |
| `agents.defaults.heartbeat.every` | "30m" (API Key) / "1h" (OAuth) | 心跳间隔 |
| `logging.redactSensitive` | "tools" | 敏感信息脱敏级别 |
| `messages.ackReactionScope` | "group-mentions" | 确认反应作用域 |

### 2. 模型默认值

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `models.providers[].models[].reasoning` | false | 是否启用推理 |
| `models.providers[].models[].input` | ["text"] | 支持的输入类型 |
| `models.providers[].models[].cost` | { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 } | 模型成本 |
| `models.providers[].models[].contextWindow` | 4096 | 上下文窗口大小 |
| `models.providers[].models[].maxTokens` | 8192 | 最大令牌数 |

### 3. 模型别名默认值

| 别名 | 实际模型 |
|------|----------|
| opus | anthropic/claude-opus-4-6 |
| sonnet | anthropic/claude-sonnet-4-6 |
| gpt | openai/gpt-5.2 |
| gpt-mini | openai/gpt-5-mini |
| gemini | google/gemini-3-pro-preview |
| gemini-flash | google/gemini-3-flash-preview |

## 配置验证

OpenClaw 在加载配置时会进行以下验证：

1. **agent ID 唯一性**：`agents.list` 中的每个 agent 必须有唯一的 ID
2. **配置格式验证**：所有配置项必须符合预期格式
3. **必填项验证**：确保必要的配置项存在
4. **值范围验证**：确保配置值在有效范围内

## 配置示例

### 基本配置结构

```json
{
  "tools": {
    "exec": {
      "enabled": true,
      "safeBins": ["echo", "ls"]
    }
  },
  "agents": {
    "defaults": {
      "model": "gpt",
      "tools": {
        "exec": {
          "enabled": false
        }
      }
    },
    "list": [
      {
        "id": "default",
        "default": true,
        "identity": {
          "name": "OpenClaw"
        },
        "tools": {
          "exec": {
            "enabled": true,
            "safeBins": ["echo", "ls", "cat"]
          }
        }
      },
      {
        "id": "restricted",
        "identity": {
          "name": "Restricted Agent"
        }
        // 继承 defaults.tools 配置
      }
    ]
  }
}
```

### 工具配置合并示例

在上述示例中：

1. **全局 tools 配置**：`exec.enabled = true`，`safeBins = ["echo", "ls"]`
2. **agents.defaults.tools**：`exec.enabled = false`（覆盖全局配置）
3. **agents.list[0].tools**：`exec.enabled = true`，`safeBins = ["echo", "ls", "cat"]`（覆盖默认配置）
4. **agents.list[1].tools**：未指定，继承 `agents.defaults.tools` 配置

最终生效的配置：
- `default` agent：`exec.enabled = true`，`safeBins = ["echo", "ls", "cat"]`
- `restricted` agent：`exec.enabled = false`，`safeBins = ["echo", "ls"]`（继承全局配置）

## 常见问题解答

### Q: agents.list 下的配置与最外层的配置是否会自动合并？

A: 是的，OpenClaw 会自动处理配置合并。具体来说：

1. **全局配置**作为基础
2. **agents.defaults** 中的配置会覆盖全局配置
3. **agents.list** 中每个 agent 的配置会覆盖 `agents.defaults` 中的对应配置

### Q: 如果我在 agents.list 中没有指定 tools 配置，会发生什么？

A: 如果 agent 没有指定 `tools` 配置，系统会：

1. 首先检查 `agents.defaults.tools` 是否存在
2. 如果存在，使用 `agents.defaults.tools` 配置
3. 如果不存在，使用最外层的 `tools` 配置
4. 如果都不存在，使用系统默认值

### Q: 如何查看最终生效的配置？

A: 你可以使用 `openclaw config show` 命令查看当前生效的配置，包括所有默认值和合并后的结果。

## 最佳实践

1. **使用默认配置**：尽量使用 `agents.defaults` 来设置通用配置，减少重复
2. **最小化覆盖**：在 `agents.list` 中只覆盖需要修改的配置项
3. **保持一致性**：为相似功能的 agent 使用相同的配置模式
4. **测试配置**：在修改配置后，使用 `openclaw config validate` 验证配置的有效性

## 配置迁移

OpenClaw 会自动处理配置迁移，包括：

- 从旧版配置格式迁移到新格式
- 处理废弃的配置项
- 应用新的默认值

如果遇到配置迁移问题，可以使用 `openclaw doctor` 命令检查和修复配置。