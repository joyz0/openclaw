# OpenClaw 的 Telegram 多用户支持

**一个 OpenClaw 服务可以对接多个 Telegram 用户**，它提供了丰富的多用户支持机制：

## 多账户配置

OpenClaw 支持配置多个 Telegram 账户，每个账户有独立的 bot token：

```json5
{
  channels: {
    telegram: {
      accounts: {
        default: {
          botToken: 'YOUR_BOT_TOKEN_1',
        },
        work: {
          botToken: 'YOUR_BOT_TOKEN_2',
          name: 'Work Bot',
        },
      },
    },
  },
}
```

## 用户访问控制

OpenClaw 提供了精细的用户访问控制机制：

### 1. 私信访问控制

- **dmPolicy**：控制私信访问模式
  - `"pairing"`：需要用户先发送 `/start` 命令
  - `"allowFrom"`：只允许指定用户
  - `"open"`：允许所有用户

### 2. 用户允许列表

- **allowFrom**：指定允许与机器人交互的用户 ID 列表

```json5
{
  channels: {
    telegram: {
      dmPolicy: 'allowFrom',
      allowFrom: ['123456789', '987654321'],
    },
  },
}
```

### 3. 群组访问控制

- **groups**：针对不同群组的配置
- **requireMention**：是否需要 @ 提及才响应
- **groupPolicy**：群组访问模式
- **groupAllowFrom**：群组中允许的用户列表

```json5
{
  channels: {
    telegram: {
      groups: {
        '-1001234567890': { requireMention: false }, // 特定群组无需提及
        '*': { requireMention: true }, // 其他群组需要提及
      },
    },
  },
}
```

## 每用户个性化配置

OpenClaw 支持为特定用户设置个性化配置：

```json5
{
  channels: {
    telegram: {
      dms: {
        '123456789': {
          historyLimit: 50, // 为特定用户设置历史消息限制
        },
      },
    },
  },
}
```

## 群组激活模式

在群组中，OpenClaw 可以配置不同的激活模式：

- **默认**：只响应 @ 提及
- **特定群组**：配置为始终响应
- **所有群组**：对所有群组始终响应

## 总结

OpenClaw 不仅支持对接多个 Telegram 用户，还提供了：

- **多账户支持**：可以配置多个 Telegram 机器人账户
- **精细的访问控制**：基于用户 ID 的访问限制
- **群组管理**：针对不同群组的差异化配置
- **个性化设置**：为特定用户提供定制化配置

这种灵活的设计使得 OpenClaw 可以在各种场景下使用，从个人助手到团队协作工具，都能很好地适应不同的用户需求。
