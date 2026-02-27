# Telegram接入指南

本文档详细介绍如何将OpenClaw接入到Telegram，并说明几个典型场景下的内部执行逻辑。

## 1. 从零接入Telegram

### 1.1 创建Telegram机器人

1. **打开Telegram应用**，搜索并启动 `@BotFather` 机器人
2. **创建新机器人**：
   - 发送 `/newbot` 命令
   - 为机器人设置一个名称（如 "OpenClaw Assistant"）
   - 为机器人设置一个唯一的用户名（必须以 `_bot` 结尾，如 "openclaw_assistant_bot"）
3. **获取API令牌**：
   - `@BotFather` 会生成一个API令牌，类似于 `1234567890:ABCdefGhIjKlMnOpQrStUvWxYz123456`
   - 保存此令牌，后续配置需要使用
4. **配置机器人权限**：
   - 发送 `/setprivacy` 命令，选择你的机器人，设置为 `DISABLED`（允许机器人读取所有消息）
   - 发送 `/setcommands` 命令，为机器人添加命令菜单（可选）

### 1.2 配置OpenClaw

1. **启动OpenClaw**：
   ```bash
   openclaw start
   ```

2. **运行配置向导**：
   ```bash
   openclaw configure
   ```

3. **选择Telegram通道**：
   - 在配置向导中，选择 "Channels" → "Telegram"
   - 输入之前获取的API令牌
   - 配置其他选项（如代理设置、消息处理方式等）

4. **验证连接**：
   ```bash
   openclaw status
   ```
   确保Telegram通道显示为 "connected"

### 1.3 开始对话

1. **在Telegram中搜索**你创建的机器人
2. **发送 `/start` 命令** 激活机器人
3. **开始聊天**：向机器人发送消息，OpenClaw会处理并回复

## 2. 内部执行逻辑说明

### 2.1 场景一：用户发送"与我视频聊天"信息

**执行逻辑**：

1. **消息接收**：
   - Telegram机器人接收到消息
   - `src/telegram/bot-handlers.ts` 中的处理器捕获消息
   - 消息被传递给 `createTelegramMessageProcessor` 进行处理

2. **消息解析**：
   - 系统识别出用户请求视频聊天功能
   - 检查是否有视频聊天相关的技能或插件

3. **技能匹配**：
   - 系统查找与视频聊天相关的技能
   - 可能匹配到 `voice-call` 技能

4. **执行流程**：
   - 加载 `voice-call` 技能
   - 检查系统是否支持视频通话
   - 检查必要的依赖是否安装
   - 初始化视频通话设置

5. **响应生成**：
   - 向用户发送视频聊天邀请
   - 提供连接说明或直接发起视频通话

### 2.2 场景二：用户发送"帮我查看我的github账户的kitz-platform项目还剩多少open状态的issues，登录信息在/auth/github.md中"信息

**执行逻辑**：

1. **消息接收**：
   - Telegram机器人接收到消息
   - `src/telegram/bot-handlers.ts` 中的处理器捕获消息
   - 消息被传递给 `createTelegramMessageProcessor` 进行处理

2. **消息解析**：
   - 系统识别出用户请求GitHub操作
   - 提取关键词："github"、"kitz-platform"、"open issues"
   - 识别出登录信息位置："/auth/github.md"

3. **技能匹配**：
   - 系统匹配到 `github` 技能（位于 `skills/github/`）
   - 可能同时匹配到 `gh-issues` 技能

4. **执行流程**：
   - 加载 `github` 技能
   - 读取 `/auth/github.md` 文件获取登录信息
   - 使用 `gh` CLI 工具执行GitHub操作
   - 运行命令：`gh issue list --repo kitz-platform --state open`

5. **响应生成**：
   - 解析 `gh` CLI 的输出
   - 统计并格式化 open 状态的 issues 数量
   - 向用户发送结果

### 2.3 场景三：用户发送"北京时间每天早上8点向我问好"信息

**执行逻辑**：

1. **消息接收**：
   - Telegram机器人接收到消息
   - `src/telegram/bot-handlers.ts` 中的处理器捕获消息
   - 消息被传递给 `createTelegramMessageProcessor` 进行处理

2. **消息解析**：
   - 系统识别出用户请求定时任务
   - 提取关键词："每天早上8点"、"向我问好"、"北京时间"

3. **技能匹配**：
   - 系统识别出需要使用定时任务功能
   - 匹配到 `cron` 相关功能（位于 `src/cron/`）

4. **执行流程**：
   - 加载 `cron` 模块
   - 创建定时任务：
     - 时间设置：每天 8:00（北京时间）
     - 任务内容：发送问候消息
     - 目标：当前聊天会话
   - 保存任务到定时任务存储

5. **响应生成**：
   - 确认定时任务已创建
   - 向用户发送任务创建成功的消息
   - 说明任务的执行时间和内容

## 3. 技术实现细节

### 3.1 Telegram接入实现

OpenClaw 使用 `grammy` 库实现 Telegram 机器人功能：

- **核心文件**：`src/telegram/bot.ts`、`src/telegram/bot-handlers.ts`
- **消息处理**：`src/telegram/bot-message.ts`
- **账户管理**：`src/telegram/accounts.ts`
- **消息发送**：`src/telegram/send.ts`

### 3.2 通道管理

通道管理由 `src/channels/` 目录下的文件实现：

- **插件系统**：`src/channels/plugins/`
- **账户管理**：`src/channels/account-action-gate.ts`
- **消息处理**：`src/channels/message-actions.ts`

### 3.3 技能系统

技能系统由 `skills/` 目录下的各个技能实现：

- **GitHub技能**：`skills/github/`
- **定时任务**：`src/cron/`
- **技能加载**：`src/agents/skills/`

### 3.4 消息处理流程

1. **消息接收**：通道插件接收消息
2. **消息解析**：提取意图和参数
3. **技能匹配**：找到适合处理该消息的技能
4. **技能执行**：执行技能逻辑
5. **响应生成**：生成并发送响应

## 4. 常见问题排查

### 4.1 Telegram机器人不响应

- 检查API令牌是否正确
- 确认 `privacy` 设置是否为 `DISABLED`
- 检查网络连接和代理设置
- 查看OpenClaw日志：`openclaw logs`

### 4.2 GitHub操作失败

- 检查 `gh` CLI 是否安装：`gh --version`
- 确认GitHub认证状态：`gh auth status`
- 验证 `/auth/github.md` 文件是否存在且格式正确
- 检查网络连接和API访问权限

### 4.3 定时任务不执行

- 检查系统时间是否正确
- 确认定时任务是否已创建：`openclaw cron list`
- 检查定时任务存储权限
- 查看定时任务日志：`openclaw logs --filter cron`

## 5. 高级配置

### 5.1 Telegram机器人高级设置

在 `~/.openclaw/config.json` 文件中，可以修改以下Telegram相关配置：

```json
{
  "channels": {
    "telegram": {
      "accounts": {
        "default": {
          "token": "YOUR_API_TOKEN",
          "config": {
            "network": {
              "proxy": "socks5://localhost:1080"  // 代理设置
            },
            "commands": {
              "native": true  // 启用原生命令
            }
          }
        }
      }
    }
  }
}
```

### 5.2 自定义技能

可以通过创建自定义技能扩展OpenClaw功能：

```bash
openclaw skills create my-skill
```

## 6. 总结

通过本文档的指南，你可以成功将OpenClaw接入到Telegram，并了解系统在处理不同类型请求时的内部执行逻辑。OpenClaw的模块化设计使得它能够灵活处理各种场景，从简单的文本聊天到复杂的GitHub操作和定时任务。

随着你对系统的熟悉，你可以进一步探索OpenClaw的高级功能，如自定义技能、多通道集成、AI模型配置等，以满足更复杂的使用需求。
