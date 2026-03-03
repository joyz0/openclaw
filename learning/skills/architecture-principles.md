# OpenClaw 编程架构思想合集

> 本文档解构 OpenClaw 项目的核心架构范式，为未来 AI 编程工具提供可复用的架构模式参考。

---

## 目录

### 技术架构篇

1. [配置系统设计](#1-配置系统设计)
2. [CLI 系统架构](#2-cli-系统架构)
3. [插件化架构](#3-插件化架构)
4. [网关通信协议](#4-网关通信协议)
5. [会话管理](#5-会话管理)
6. [重试与容错](#6-重试与容错)
7. [类型安全与 Schema 驱动](#7-类型安全与-schema-驱动)
8. [日志与可观测性](#8-日志与可观测性)
9. [安全与认证](#9-安全与认证)
10. [模块边界与依赖注入](#10-模块边界与依赖注入)
11. [媒体处理架构](#11-媒体处理架构)
12. [内存管理系统](#12-内存管理系统)

### 项目管理篇

13. [测试系统架构](#13-测试系统架构)
14. [部署与发布系统](#14-部署与发布系统)
15. [CI/CD 与工作流](#15-cicd-与工作流)
16. [国际化系统](#16-国际化系统)
17. [文档系统](#17-文档系统)
18. [项目管理最佳实践](#18-项目管理最佳实践)

---

## 快速索引

### 常见问题快速定位

| 问题               | 章节                                                   |
| ------------------ | ------------------------------------------------------ |
| 如何添加新通道？   | [3. 插件化架构](#3-插件化架构)                         |
| 如何配置会话？     | [5. 会话管理](#5-会话管理)                             |
| 如何处理媒体文件？ | [11. 媒体处理架构](#11-媒体处理架构)                   |
| 如何设计重试机制？ | [6. 重试与容错](#6-重试与容错)                         |
| 如何定义 Schema？  | [7. 类型安全与 Schema 驱动](#7-类型安全与-schema-驱动) |
| 如何调试网关问题？ | [附录 B.6](#b2-开发实践)                               |
| 如何优化性能？     | [附录 B.7-8](#b3-性能优化)                             |
| 如何备份配置？     | [附录 B.9](#b4-安全与运维)                             |

### 关键决策记录

- [为什么选择 TypeBox？](#7-类型安全与-schema-驱动)
- [为什么使用 pnpm？](#1831-包管理器选择)
- [插件 vs 核心模块](#b1-架构设计)
- [会话作用域选择](#b1-架构设计)

### 检查清单

- [添加新插件](#c1-添加新插件检查清单)
- [发布前检查](#c2-发布前检查清单)
- [故障排查](#c3-故障排查检查清单)

---

## 1. 配置系统设计

### 1.1 核心理念

**配置即代码，但安全优先**

- 使用 JSON5 支持注释和 trailing commas，提升可读性
- 严格的 Schema 验证：未知 key 会导致启动失败（除 `$schema` 外）
- 支持环境变量替换：`${VAR_NAME}` 语法
- 支持 Secret 引用：env/file/exec 三种来源
- 配置文件分片：`$include` 支持模块化组织

### 1.2 架构层次

```
┌─────────────────────────────────────┐
│         配置读取层 (io.ts)          │
│  - 读取 JSON5 文件                    │
│  - 环境变量替换                      │
│  - $include 解析                      │
│  - 备份轮转                          │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│       配置合并层 (merge-*.ts)       │
│  - 深度合并 include 文件               │
│  - 应用运行时覆盖                    │
│  - 路径标准化                        │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│       默认值层 (defaults.ts)        │
│  - 按模块应用默认值                  │
│  - 条件默认值（基于现有配置）         │
│  - 迁移旧配置格式                    │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│       验证层 (validation.ts)        │
│  - Zod Schema 全量验证                │
│  - 插件 Schema 合并验证                │
│  - 生成验证错误报告                  │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│       运行时层 (runtime.ts)         │
│  - 配置热重载                        │
│  - 配置快照管理                      │
│  - 配置审计日志                      │
└─────────────────────────────────────┘
```

### 1.2.1 配置加载流程

```mermaid
flowchart TD
    A[开始] --> B[读取 JSON5 文件]
    B --> C[解析 $include]
    C --> D[环境变量替换]
    D --> E[应用默认值]
    E --> F[Schema 验证]
    F --> G{验证通过？}
    G -->|是 | H[保存配置快照]
    G -->|否 | I[输出错误报告]
    I --> J[退出]
    H --> K[启动热重载监听]
    K --> L[配置就绪]

    style F fill:#e1f5ff
    style G fill:#fff3cd
    style H fill:#d4edda
    style I fill:#f8d7da
```

### 1.3 关键模式

#### 1.3.1 配置 IO 模式

```typescript
// 读取配置时捕获快照，用于后续写操作的对比
export type ConfigFileSnapshot = {
  hash: string; // SHA256 哈希
  raw: string; // 原始内容
  parsed: unknown; // 解析后的对象
  exists: boolean; // 文件是否存在
  mtime: number; // 修改时间
};

// 写配置时使用快照做乐观锁
export async function writeConfigFile(
  configPath: string,
  config: OpenClawConfig,
  options: {
    baseHash?: string; // 期望的哈希，防止并发写冲突
    envSnapshot?: Record<string, string>;
  }
): Promise<void>;
```

#### 1.3.2 环境变量替换模式

```typescript
// 支持 ${VAR} 语法，仅匹配大写字母
const ENV_VAR_RE = /\$\{([A-Z_][A-Z0-9_]*)\}/g;

// 递归替换所有字符串值中的环境变量
function resolveConfigEnvVars(
  config: unknown,
  env: Record<string, string | undefined>,
  missingVars: Set<string>
): unknown;
```

#### 1.3.3 配置热重载模式

```typescript
type ReloadMode = "hybrid" | "hot" | "restart" | "off";

// hybrid: 安全变更热应用，关键变更自动重启
// hot: 安全变更热应用，需重启的仅警告
// restart: 任何变更都重启
// off: 禁用热重载
```

### 1.4 最佳实践

1. **Schema 先行**：先定义 Zod Schema，再生成类型和验证逻辑
2. **错误诊断友好**：验证失败时提供精确的路径和修复建议
3. **配置审计**：记录每次配置变更的时间、哈希、执行命令
4. **备份策略**：写配置前轮转旧文件（保留最近 N 个版本）
5. **路径标准化**：统一解析 `~` 和环境变量路径

### 1.5 相关章节

- [类型安全与 Schema 驱动](#7-类型安全与-schema-驱动) - Schema 定义方法
- [插件化架构](#3-插件化架构) - 插件配置合并
- [会话管理](#5-会话管理) - 会话配置结构

---

## 2. CLI 系统架构

### 2.1 核心理念

**命令即 API，一致性优先**

- 基于 Commander.js 构建声明式命令树
- 统一的上下文传递机制
- Pre-action 钩子实现横切关注点
- 支持 RPC 模式供远程调用

### 2.2 架构层次

```
┌─────────────────────────────────────┐
│         命令注册层 (program.ts)      │
│  - 创建 Commander 实例                 │
│  - 注册所有子命令                    │
│  - 配置帮助信息                      │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│       上下文层 (context.ts)         │
│  - 解析程序版本                      │
│  - 加载配置路径                      │
│  - 初始化日志器                      │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│      Pre-action 钩子层 (preaction.ts)│
│  - 版本检查                          │
│  - 配置验证                          │
│  - 诊断模式检测                      │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│        命令实现层 (commands/*.ts)    │
│  - 业务逻辑                          │
│  - 参数解析                          │
│  - 结果输出                          │
└─────────────────────────────────────┘
```

### 2.3 关键模式

#### 2.3.1 命令注册模式

```typescript
export function buildProgram(): Command {
  const program = new Command();
  const ctx = createProgramContext();

  // 设置上下文
  setProgramContext(program, ctx);

  // 配置帮助信息
  configureProgramHelp(program, ctx);

  // 注册 pre-action 钩子
  registerPreActionHooks(program, ctx.programVersion);

  // 注册所有子命令
  registerProgramCommands(program, ctx, argv);

  return program;
}
```

#### 2.3.2 上下文传递模式

```typescript
export type ProgramContext = {
  programVersion: string;
  configPath: string;
  logger: Logger;
  startTime: number;
};

// 通过 Command 的 store 传递上下文
export function setProgramContext(program: Command, ctx: ProgramContext): void;

export function getProgramContext(cmd: Command): ProgramContext;
```

#### 2.3.3 Pre-action 钩子模式

```typescript
// 在所有命令执行前运行的横切逻辑
program.hook("preAction", (thisCommand, actionCommand) => {
  const ctx = getProgramContext(thisCommand);

  // 1. 版本兼容性检查
  checkVersionCompatibility(ctx.programVersion);

  // 2. 诊断模式检测
  if (isDiagnosticCommand(actionCommand.name())) {
    skipConfigValidation();
  }

  // 3. 配置加载（非诊断命令）
  if (!isDiagnosticCommand(actionCommand.name())) {
    loadConfigIfNeeded(ctx);
  }
});
```

### 2.4 命令分类

1. **诊断命令**：不依赖配置，用于故障排查
   - `doctor`, `health`, `logs`, `status`
2. **配置命令**：读写配置
   - `configure`, `config get/set/unset`
3. **通道命令**：管理消息通道
   - `channels status`, `pairing approve`
4. **会话命令**：管理会话状态
   - `sessions list`, `sessions cleanup`
5. **网关命令**：网关运维
   - `gateway call`, `gateway run`

### 2.5 最佳实践

1. **命令命名一致性**：使用 `noun verb` 格式（如 `sessions list`）
2. **参数验证**：在命令入口处验证所有参数
3. **错误处理**：统一错误格式，提供修复建议
4. **输出格式化**：支持 `--json` 模式供脚本调用
5. **帮助信息**：提供示例和常见问题链接

### 2.6 相关章节

- [配置系统设计](#1-配置系统设计) - 配置加载逻辑
- [日志与可观测性](#8-日志与可观测性) - 日志器初始化
- [部署与发布系统](#14-部署与发布系统) - CLI 发布流程

---

## 3. 插件化架构

### 3.1 核心理念

**隔离优先，稳定 API**

- 插件与核心代码完全隔离
- 通过 SDK 提供编译时类型
- 通过 Runtime 提供运行时能力
- 禁止插件直接导入核心模块

### 3.2 架构层次

```
┌─────────────────────────────────────┐
│         插件 SDK (plugin-sdk/)      │
│  - 类型定义                          │
│  - Config Schema 构建器                │
│  - 工具函数                          │
│  - 文档链接助手                      │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│       插件 Runtime (runtime.ts)     │
│  - 通道操作 API                      │
│  - 日志 API                          │
│  - 状态管理 API                      │
│  - 媒体处理 API                      │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│        插件加载器 (loader.ts)       │
│  - 发现插件                          │
│  - 验证 Manifest                     │
│  - 注入 Runtime                      │
│  - 注册钩子和工具                    │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│         插件实现 (extensions/*)      │
│  - 通道连接器                        │
│  - 自定义工具                        │
│  - 自动化钩子                        │
└─────────────────────────────────────┘
```

### 3.2.1 插件生命周期

```mermaid
stateDiagram-v2
    [*] --> Discovering: 启动网关
    Discovering --> Validating: 找到插件
    Validating --> Loading: Manifest 验证通过
    Loading --> Running: 注入 Runtime
    Running --> Registering: 注册钩子/工具
    Registering --> Active: 插件就绪

    Active --> Error: 运行时错误
    Error --> Active: 恢复
    Error --> [*]: 禁用插件

    Active --> Unloading: 网关关闭
    Unloading --> [*]: 清理完成
```

### 3.3 关键模式

#### 3.3.1 插件 API 模式

```typescript
export type OpenClawPluginApi = {
  // 编译时 SDK（稳定，语义化版本）
  sdk: {
    types: ChannelPlugin;
    config: {
      buildSchema: (schema: object) => object;
      setAccountEnabled: (config, channel, accountId, enabled) => void;
    };
    pairing: {
      APPROVED_MESSAGE: string;
      formatApproveHint: (code: string) => string;
    };
  };

  // 运行时 API（版本化，每发布版本更新）
  runtime: PluginRuntime;

  // 插件元数据
  manifest: PluginManifest;
};
```

#### 3.3.2 Runtime 注入模式

```typescript
// 插件加载时注入 runtime
export async function loadPlugin(
  pluginPath: string,
  api: OpenClawPluginApi
): Promise<PluginModule> {
  // 使用 jiti 动态加载插件
  const jiti = createJiti(__filename, {
    alias: {
      "openclaw/plugin-sdk": pluginSdkPath,
    },
  });

  // 加载插件模块
  const module = jiti(pluginPath) as PluginModule;

  // 注入 API
  module.default(api);

  return module;
}
```

#### 3.3.3 插件注册模式

```typescript
export type PluginRegistry = {
  // 按 ID 索引
  plugins: Map<string, PluginRecord>;

  // 按通道类型索引
  byChannelType: Map<string, string[]>;

  // 钩子注册表
  hooks: {
    beforeSend: HookRegistry;
    afterReceive: HookRegistry;
  };

  // 工具注册表
  tools: Map<string, ToolDefinition>;
};
```

### 3.4 最佳实践

1. **API 稳定性**：SDK 语义化版本，Runtime 每版本编号
2. **最小权限**：插件只能访问显式注入的 API
3. **错误隔离**：插件错误不影响核心功能
4. **版本兼容**：插件声明所需 runtime 版本范围
5. **文档链接**：每个插件提供 docs.openclaw.ai 链接

### 3.5 相关章节

- [配置系统设计](#1-配置系统设计) - 插件配置 Schema 合并
- [模块边界与依赖注入](#10-模块边界与依赖注入) - 模块隔离
- [CI/CD 与工作流](#15-cicd-与工作流) - 插件发布流程

---

## 4. 网关通信协议

### 4.1 核心理念

**Schema 即真理，类型驱动**

- TypeBox 定义协议 Schema
- 自动生成 JSON Schema
- 自动生成 Swift 客户端模型
- 运行时验证所有消息

### 4.2 协议层次

```
┌─────────────────────────────────────┐
│      Schema 定义层 (schema.ts)       │
│  - TypeBox 类型定义                   │
│  - 协议版本号                         │
│  - 方法/事件枚举                     │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│      验证器层 (index.ts)            │
│  - 生成 AJV 验证器                    │
│  - 导出 TypeScript 类型                │
│  - 生成 JSON Schema                  │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│      服务器层 (server.ts)           │
│  - WebSocket 连接管理                  │
│  - 消息验证                          │
│  - 路由分发                          │
│  - 事件广播                          │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│      客户端层 (client.ts)           │
│  - 连接管理                          │
│  - 请求/响应跟踪                     │
│  - 事件订阅                          │
└─────────────────────────────────────┘
```

### 4.2.1 连接流程

````mermaid
sequenceDiagram
    participant Client
    participant Gateway

    Client->>Gateway: connect request
    Gateway->>Gateway: validate auth
    alt 已配对设备
        Gateway-->>Client: connection established
    else 新设备
        Gateway-->>Client: requires pairing + code
        Note over Gateway: 等待用户批准
    end

    loop 心跳
        Client->>Gateway: ping
        Gateway-->>Client: pong
    end

    Client->>Gateway: send message
    Gateway-->>Client: ack

### 4.3 关键模式

#### 4.3.1 帧结构模式

```typescript
// 所有消息都是这三种帧之一
type GatewayFrame =
  | RequestFrame // { type: "req", id, method, params }
  | ResponseFrame // { type: "res", id, ok, payload | error }
  | EventFrame; // { type: "event", event, payload, seq? }

// 第一个消息必须是 connect
type ConnectRequest = {
  type: "req";
  id: string;
  method: "connect";
  params: {
    minProtocol: number;
    maxProtocol: number;
    client: ClientInfo;
  };
};
````

#### 4.3.2 协议版本协商

```typescript
// 客户端声明支持的范围
{
  minProtocol: 2,
  maxProtocol: 3
}

// 服务器选择共同支持的版本
{
  protocol: 3,  // 使用最高共同版本
  server: { version, connId },
  features: { methods, events }
}
```

#### 4.3.3 幂等键模式

```typescript
// 副作用操作需要幂等键
type SendParams = {
  idempotencyKey: string; // 必需
  channel: string;
  to: string;
  text: string;
};

// 服务器缓存已处理的幂等键（短期）
const idempotencyCache = new LRUCache<string, any>({
  max: 10000,
  ttl: 3600_000, // 1 小时
});
```

### 4.4 最佳实践

1. **Schema 优先开发**：先改 Schema，再改实现
2. **严格验证**：所有入站消息必须通过 AJV 验证
3. **向后兼容**：未知字段保留，不抛出错误
4. **版本协商**：客户端声明范围，服务器选择版本
5. **幂等设计**：副作用操作必须支持重试

### 4.5 相关章节

- [类型安全与 Schema 驱动](#7-类型安全与-schema-驱动) - TypeBox 使用方法
- [重试与容错](#6-重试与容错) - 幂等重试机制
- [安全与认证](#9-安全与认证) - 连接认证流程

---

## 5. 会话管理

### 5.1 核心理念

**会话即状态，隔离优先**

- 每个会话有独立的上下文
- 支持按用户/频道/群组隔离
- 自动维护会话生命周期
- 透明的压缩和归档

### 5.2 会话键模式

```typescript
// 会话键决定隔离级别
type SessionKey =
  | `agent:${agentId}:main` // 所有 DM 共享
  | `agent:${agentId}:dm:${peerId}` // 按用户隔离
  | `agent:${agentId}:${channel}:dm:${peerId}` // 按频道 + 用户
  | `agent:${agentId}:${channel}:group:${groupId}` // 群组
  | `agent:${agentId}:${channel}:channel:${roomId}`; // 频道房间
```

### 5.3 关键模式

#### 5.3.1 会话作用域配置

```typescript
{
  session: {
    // DM 会话作用域
    dmScope: "per-channel-peer",  // main | per-peer | per-channel-peer

    // 身份链接（同一用户跨渠道共享会话）
    identityLinks: {
      alice: ["telegram:123", "discord:456"]
    },

    // 重置策略
    reset: {
      mode: "daily",      // daily | idle | never
      atHour: 4,          // 每天 4 点重置
      idleMinutes: 120,   // 120 分钟无活动重置
    },
  }
}
```

#### 5.3.2 会话存储结构

```typescript
// sessions.json - 会话索引
{
  "agent:main:whatsapp:dm:+123": {
    sessionId: "sess_abc123",
    updatedAt: 1709372800000,
    inputTokens: 1000,
    outputTokens: 500,
    totalTokens: 1500,
    origin: {
      label: "Alice",
      provider: "whatsapp",
      from: "+123",
      to: "+456"
    }
  }
}

// sess_abc123.jsonl - 对话记录
{"type":"user","content":"Hello","timestamp":1709372800000}
{"type":"assistant","content":"Hi there!","timestamp":1709372801000}
```

#### 5.3.3 会话维护模式

```typescript
{
  session: {
    maintenance: {
      mode: "enforce",     // warn | enforce
      pruneAfter: "30d",   // 30 天后清理
      maxEntries: 500,     // 最多 500 个会话
      rotateBytes: "10mb", // 索引文件超过 10MB 轮转
      maxDiskBytes: "1gb", // 磁盘预算
      highWaterBytes: "800mb" // 高水位线
    }
  }
}
```

### 5.4 最佳实践

1. **安全隔离**：多用户场景使用 `per-channel-peer`
2. **身份链接**：跨渠道识别同一用户
3. **自动维护**：生产环境使用 `enforce` 模式
4. **磁盘预算**：设置 `maxDiskBytes` 防止无限增长
5. **归档策略**：保留重要会话，归档旧会话

### 5.5 相关章节

- [配置系统设计](#1-配置系统设计) - 会话配置结构
- [内存管理系统](#12-内存管理系统) - 会话数据存储
- [日志与可观测性](#8-日志与可观测性) - 会话追踪上下文

---

## 6. 重试与容错

### 6.1 核心理念

**失败是常态，优雅降级**

- 指数退避 + 抖动
- 基于错误类型的智能重试
- 超时控制
- 熔断机制

### 6.2 关键模式

#### 6.2.1 通用重试模式

```typescript
type RetryConfig = {
  attempts: number;      // 最大尝试次数
  minDelayMs: number;    // 最小延迟
  maxDelayMs: number;    // 最大延迟
  jitter: number;        // 抖动比例 (0-1)
};

async function retryAsync<T>(
  fn: () => Promise<T>,
  options: RetryOptions
): Promise<T> {
  const config = resolveRetryConfig(DEFAULT_CONFIG, options);

  for (let attempt = 1; attempt <= config.attempts; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (attempt >= config.attempts) throw;
      if (!options.shouldRetry?.(err, attempt)) throw;

      // 计算延迟（指数退避 + 抖动）
      const baseDelay = config.minDelayMs * 2 ** (attempt - 1);
      const delay = applyJitter(
        Math.min(baseDelay, config.maxDelayMs),
        config.jitter
      );

      await sleep(delay);
    }
  }
}

function applyJitter(delayMs: number, jitter: number): number {
  const offset = (Math.random() * 2 - 1) * jitter;
  return Math.max(0, Math.round(delayMs * (1 + offset)));
}
```

#### 6.2.2 HTTP 重试模式

```typescript
// 基于 HTTP 状态码的重试策略
function shouldRetryHttpError(status: number): boolean {
  return (
    status === 408 || // Request Timeout
    status === 429 || // Too Many Requests
    status === 500 || // Internal Server Error
    status === 502 || // Bad Gateway
    status === 503 || // Service Unavailable
    status === 504 // Gateway Timeout
  );
}

// 解析 Retry-After 头
function parseRetryAfterMs(response: Response): number | undefined {
  const retryAfter = response.headers.get("Retry-After");
  if (!retryAfter) return undefined;

  // 可能是秒数或 HTTP 日期
  const seconds = parseInt(retryAfter, 10);
  if (!isNaN(seconds)) return seconds * 1000;

  const date = new Date(retryAfter);
  if (!isNaN(date.getTime())) {
    return date.getTime() - Date.now();
  }

  return undefined;
}
```

#### 6.2.3 熔断器模式

```typescript
type CircuitBreakerState = "closed" | "open" | "half-open";

class CircuitBreaker {
  private state: CircuitBreakerState = "closed";
  private failureCount = 0;
  private lastFailureTime: number | null = null;

  constructor(
    private threshold: number, // 失败阈值
    private timeout: number, // 熔断超时
    private halfOpenMaxCalls: number // 半开状态最大调用数
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === "open") {
      if (Date.now() - this.lastFailureTime! > this.timeout) {
        this.state = "half-open";
        this.failureCount = 0;
      } else {
        throw new Error("Circuit breaker is open");
      }
    }

    try {
      const result = await fn();

      if (this.state === "half-open") {
        this.failureCount++;
        if (this.failureCount >= this.halfOpenMaxCalls) {
          this.state = "closed";
          this.failureCount = 0;
        }
      }

      return result;
    } catch (err) {
      this.failureCount++;
      this.lastFailureTime = Date.now();

      if (this.failureCount >= this.threshold) {
        this.state = "open";
      }

      throw err;
    }
  }
}
```

### 6.3 最佳实践

1. **默认重试**：所有外部调用默认启用重试
2. **智能退避**：指数退避 + 抖动避免雪崩
3. **错误分类**：区分可重试和不可重试错误
4. **熔断保护**：防止故障服务被持续调用
5. **超时控制**：所有异步操作设置超时

### 6.4 性能考量

- **重试风暴**：大量并发请求同时重试可能导致服务雪崩
  - 解决：使用抖动（jitter）和指数退避
  - 避免：固定延迟重试或无限制重试
- **内存占用**：熔断器状态和重试队列占用内存
  - 建议：设置合理的缓存大小和 TTL
  - 监控：跟踪熔断器状态和重试次数

### 6.5 相关章节

- [网关通信协议](#4-网关通信协议) - 幂等键设计
- [日志与可观测性](#8-日志与可观测性) - 错误追踪

---

## 7. 类型安全与 Schema 驱动

### 7.1 核心理念

**类型即文档，验证即安全**

- TypeBox 定义运行时 Schema
- Zod 用于配置验证
- TypeScript 提供编译时类型
- 自动生成 API 文档

### 7.2 关键模式

#### 7.2.1 TypeBox 协议定义

```typescript
import { Type, type Static } from "@sinclair/typebox";

// 定义基础类型
const NonEmptyString = Type.String({ minLength: 1 });

// 定义协议 Schema
export const ConnectParamsSchema = Type.Object(
  {
    minProtocol: Type.Number(),
    maxProtocol: Type.Number(),
    client: Type.Object({
      id: NonEmptyString,
      displayName: NonEmptyString,
      version: NonEmptyString,
      platform: NonEmptyString,
      mode: Type.Union([
        Type.Literal("ui"),
        Type.Literal("cli"),
        Type.Literal("node"),
      ]),
    }),
  },
  { additionalProperties: false }
);

// 导出 TypeScript 类型
export type ConnectParams = Static<typeof ConnectParamsSchema>;

// 导出到协议注册表
export const ProtocolSchemas = {
  ConnectParams: ConnectParamsSchema,
  // ... 其他 schema
};
```

#### 7.2.2 Zod 配置验证

```typescript
import { z } from "zod";

// 定义配置 Schema
export const OpenClawConfigSchema = z.object({
  agents: z.object({
    defaults: z.object({
      workspace: z.string(),
      model: z.object({
        primary: z.string(),
        fallbacks: z.array(z.string()).optional(),
      }),
    }),
  }),
  channels: z.object({
    whatsapp: z
      .object({
        enabled: z.boolean(),
        dmPolicy: z.enum(["pairing", "allowlist", "open", "disabled"]),
      })
      .optional(),
  }),
});

// 验证并推断类型
export type OpenClawConfig = z.infer<typeof OpenClawConfigSchema>;

// 验证函数
export function validateConfigObject(
  config: unknown,
  plugins?: PluginRegistry
):
  | { valid: true; config: OpenClawConfig }
  | { valid: false; errors: ValidationError[] } {
  const result = OpenClawConfigSchema.safeParse(config);

  if (!result.success) {
    return {
      valid: false,
      errors: result.error.errors.map((err) => ({
        path: err.path.join("."),
        message: err.message,
        code: err.code,
      })),
    };
  }

  return { valid: true, config: result.data };
}
```

### 7.3 最佳实践

1. **Schema 单一来源**：TypeBox 同时用于验证和类型推导
2. **严格模式**：`additionalProperties: false` 防止未知字段
3. **错误友好**：验证错误包含精确路径和修复建议
4. **版本化**：协议 Schema 包含版本号
5. **代码生成**：从 Schema 自动生成 Swift/TypeScript 类型

### 7.4 反模式示例

```typescript
// ❌ 错误：Schema 和类型分离（容易导致不一致）
interface MyType {
  name: string;
  age: number;
}

const schema = z.object({
  name: z.string(),
  agee: z.number(), // 拼写错误！TypeScript 无法发现
});

// ✅ 正确：从 Schema 推导类型
const schema = z.object({
  name: z.string(),
  age: z.number(),
});
type MyType = z.infer<typeof schema>; // 类型自动同步
```

### 7.5 性能考量

- **验证开销**：运行时验证会增加延迟
  - 典型开销：每个对象 0.1-1ms
  - 优化：批量验证、缓存验证结果
- **Schema 大小**：过大的 Schema 会影响编译速度
  - 建议：按模块拆分 Schema
  - 避免：单个 Schema 文件超过 1000 行

### 7.6 相关章节

- [网关通信协议](#4-网关通信协议) - TypeBox 协议定义
- [配置系统设计](#1-配置系统设计) - Zod 配置验证

---

## 8. 日志与可观测性

### 8.1 核心理念

**结构化日志，分级追踪**

- 统一日志格式
- 子系统隔离
- 敏感信息脱敏
- 诊断上下文

### 8.2 关键模式

#### 8.2.1 子系统日志模式

```typescript
// 创建带子系统的日志器
export function createSubsystemLogger(
  subsystem: string,
  parent?: Logger
): Logger {
  return {
    debug: (msg, ...args) =>
      log('debug', subsystem, msg, args),
    info: (msg, ...args) =>
      log('info', subsystem, msg, args),
    warn: (msg, ...args) =>
      log('warn', subsystem, msg, args),
    error: (msg, ...args) =>
      log('error', subsystem, msg, args),
    verbose: (msg, ...args) =>
      log('verbose', subsystem, msg, args)
  };
}

// 日志格式
{
  ts: "2026-03-03T10:00:00.000Z",
  level: "info",
  subsystem: "gateway",
  message: "WebSocket connection established",
  context: {
    connId: "ws-123",
    client: "macos"
  }
}
```

#### 8.2.2 敏感信息脱敏

```typescript
// 脱敏规则
const REDACT_PATTERNS = [
  { name: "api_key", regex: /sk-[a-zA-Z0-9]{32,}/g },
  { name: "token", regex: /Bearer [a-zA-Z0-9._-]+/g },
  { name: "password", regex: /password[=:]\s*\S+/gi },
];

export function redactSensitiveInfo(
  message: string,
  rules: RedactPattern[] = REDACT_PATTERNS
): string {
  let result = message;
  for (const rule of rules) {
    result = result.replace(rule.regex, `[REDACTED_${rule.name}]`);
  }
  return result;
}
```

#### 8.2.3 诊断上下文模式

```typescript
// 日志上下文
type LogContext = {
  // 请求追踪
  requestId?: string;
  sessionId?: string;

  // 用户上下文
  userId?: string;
  channelId?: string;

  // 性能指标
  durationMs?: number;
  tokensUsed?: number;

  // 错误信息
  errorCode?: string;
  errorMessage?: string;
};

// 在日志中携带上下文
logger.info("Agent run completed", {
  requestId: "req_123",
  sessionId: "sess_456",
  durationMs: 1500,
  tokensUsed: 2048,
});
```

### 8.3 最佳实践

1. **结构化日志**：JSON 格式，便于机器解析
2. **分级日志**：debug/info/warn/error/verbose
3. **敏感脱敏**：自动脱敏 API 密钥、令牌、密码
4. **上下文携带**：日志包含追踪 ID 和会话 ID
5. **性能追踪**：记录关键操作的耗时和指标

---

## 9. 安全与认证

### 9.1 核心理念

**零信任，最小权限**

- 所有连接需要认证
- 设备配对机制
- 本地信任自动批准
- 远程连接显式授权

### 9.2 关键模式

#### 9.2.1 设备配对模式

```typescript
// 配对流程
type PairingFlow = {
  // 1. 设备生成密钥对
  keypair: { publicKey: string; privateKey: string };

  // 2. 首次连接（未配对）
  connect: {
    deviceId: string;
    publicKey: string;
    signature: string; // 对 challenge 的签名
  };

  // 3. 网关返回配对码
  response: {
    requiresPairing: true;
    pairingCode: string; // 6 位数字
  };

  // 4. 用户在网关批准配对码
  approve: {
    pairingCode: string;
  };

  // 5. 获取设备令牌
  token: {
    deviceToken: string; // JWT
    expiresAt: number;
  };
};
```

#### 9.2.2 连接认证模式

```typescript
// 连接请求
{
  type: "req",
  id: "c1",
  method: "connect",
  params: {
    client: {
      id: "macos",
      displayName: "macOS App",
      version: "2026.2.26",
      platform: "macos 15.1",
      deviceId: "dev_abc123",
      publicKey: "pk_..."
    },
    auth: {
      token: "jwt_token",  // 已配对设备
      signature: "sig_..." // 对 challenge 的签名
    }
  }
}

// 网关验证
1. 验证 signature（挑战 - 响应）
2. 检查 deviceToken（已配对设备）
3. 检查 deviceId 是否在配对存储中
4. 本地连接自动批准（可选）
5. 远程连接需要显式配对
```

#### 9.2.3 访问控制模式

```typescript
// DM 访问策略
type DMPolicy =
  | 'pairing'    // 需要配对码批准
  | 'allowlist'  // 仅允许列表中的用户
  | 'open'       // 允许所有用户
  | 'disabled';  // 禁用所有 DM

// 配置示例
{
  channels: {
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+1234567890"]  // 允许的号码
    },
    telegram: {
      dmPolicy: "pairing"  // 默认策略
    }
  }
}
```

### 9.3 最佳实践

1. **配对优先**：默认使用 pairing 策略
2. **本地信任**：本地连接自动批准（可选）
3. **签名验证**：所有连接请求必须签名
4. **令牌过期**：设备令牌设置过期时间
5. **审计日志**：记录所有配对和认证事件

---

## 10. 模块边界与依赖注入

### 10.1 核心理念

**明确边界，可测试优先**

- 模块间通过接口通信
- 依赖通过构造函数注入
- 避免循环依赖
- 便于单元测试

### 10.2 关键模式

#### 10.2.1 依赖注入模式

```typescript
// 定义依赖接口
interface Dependencies {
  config: OpenClawConfig;
  logger: Logger;
  httpClient: HttpClient;
  db: Database;
}

// 使用依赖的类
class AgentService {
  constructor(private deps: Dependencies) {}

  async run(message: string): Promise<string> {
    this.deps.logger.info("Running agent", { message });

    const config = this.deps.config.agents.defaults;
    const response = await this.deps.httpClient.post("/agent", {
      message,
      model: config.model.primary,
    });

    await this.deps.db.save({ message, response });

    return response;
  }
}

// 创建依赖
const deps: Dependencies = {
  config: await loadConfig(),
  logger: createSubsystemLogger("agent"),
  httpClient: createHttpClient(),
  db: createDatabase(),
};

// 注入依赖
const agentService = new AgentService(deps);
```

#### 10.2.2 工厂模式

```typescript
// 依赖工厂
export function createDefaultDeps(
  overrides?: Partial<Dependencies>
): Dependencies {
  return {
    config: loadConfig(),
    logger: createSubsystemLogger("default"),
    httpClient: createHttpClient(),
    db: createDatabase(),
    ...overrides,
  };
}

// 测试中使用 mock 依赖
const testDeps = createDefaultDeps({
  httpClient: {
    post: vi.fn().mockResolvedValue({ data: "mock" }),
  },
  db: {
    save: vi.fn(),
  },
});
```

#### 10.2.3 模块边界模式

```typescript
// 公共 API（导出给其他模块使用）
export {
  AgentService,
  type AgentOptions,
  type AgentResult,
} from "./agent-service.js";

// 内部实现（不导出）
import { InternalHelper } from "./internal-helper.js";

// 边界检查脚本（CI 运行）
// scripts/check-module-boundaries.mjs
// 确保 extensions/** 不导入 src/**
```

### 10.3 最佳实践

1. **接口隔离**：模块间通过接口通信，不直接依赖实现
2. **依赖注入**：所有外部依赖通过构造函数注入
3. **工厂函数**：提供默认依赖工厂便于测试
4. **边界检查**：CI 检查模块边界违规
5. **循环依赖检测**：使用工具检测循环依赖

---

---

## 11. 媒体处理架构

### 11.1 核心理念

**统一接口，按需处理**

- 所有媒体通过统一接口获取和存储
- 自动识别 MIME 类型并验证
- 支持本地和远程媒体源
- 透明的格式转换和优化

### 11.2 架构层次

```
┌─────────────────────────────────────┐
│      媒体获取层 (fetch.ts)          │
│  - HTTP 下载                          │
│  - 本地文件读取                      │
│  - Base64 解析                        │
│  - 流式处理                          │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│      类型识别层 (mime.ts)           │
│  - Magic Number 识别                  │
│  - 文件扩展名验证                    │
│  - MIME 类型映射                      │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│      处理层 (image-ops.ts)          │
│  - 图片缩放/裁剪                     │
│  - 格式转换（WebP/PNG/JPEG）          │
│  - 质量优化                          │
│  - 音频转码（OPUS/MP3）               │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│      存储层 (store.ts)              │
│  - 临时文件管理                      │
│  - 媒体服务器托管                    │
│  - CDN 集成（可选）                   │
│  - 垃圾回收                          │
└─────────────────────────────────────┘
```

### 11.3 关键模式

#### 11.3.1 媒体获取模式

```typescript
interface MediaFetchResult {
  path: string; // 本地文件路径
  contentType?: string; // MIME 类型
  size: number; // 文件大小（字节）
  url?: string; // 原始 URL（如果是远程）
}

// 统一获取接口
async function fetchMedia(
  source: string | Buffer,
  options: {
    timeoutMs?: number;
    maxBytes?: number;
    allowedTypes?: string[];
  }
): Promise<MediaFetchResult>;
```

#### 11.3.2 类型识别模式

```typescript
// 基于 magic number 识别
async function fileTypeFromBuffer(
  buffer: Uint8Array
): Promise<{ mime: string; ext: string } | undefined>;

// 验证和标准化 MIME 类型
function normalizeMimeType(mime: string): string {
  // image/jpeg -> image/jpeg
  // image/jpg  -> image/jpeg
  // text/plain -> application/txt (标准化)
}
```

#### 11.3.3 图片处理模式

```typescript
interface ImageProcessingOptions {
  maxDimension?: number; // 最大边长
  quality?: number; // 质量 (0-100)
  format?: "jpeg" | "png" | "webp";
  preserveAnimation?: boolean;
}

async function processImage(
  input: string | Buffer,
  options: ImageProcessingOptions
): Promise<Buffer>;
```

#### 11.3.4 媒体服务器模式

```typescript
// 内置 HTTP 服务器托管媒体文件
{
  media: {
    server: {
      enabled: true,
      port: 18790,
      bind: '127.0.0.1',
      path: '~/.openclaw/media'
    }
  }
}

// 生成的 URL: http://127.0.0.1:18790/<filename>
```

### 11.4 最佳实践

1. **大小限制**：设置 `maxBytes` 防止大文件攻击
2. **类型白名单**：只允许预期的 MIME 类型
3. **超时控制**：所有媒体获取设置超时
4. **临时文件清理**：定期清理未使用的媒体文件
5. **流式处理**：大文件使用流式避免内存溢出

---

## 12. 内存管理系统

### 12.1 核心理念

**多后端支持，透明抽象**

- 统一的内存存储接口
- 支持多种后端（SQLite/内存/远程）
- 自动过期和清理
- 向量检索优化

### 12.2 架构层次

```
┌─────────────────────────────────────┐
│      存储接口层 (manager.ts)        │
│  - 统一的 CRUD 接口                   │
│  - 事务支持                          │
│  - 批量操作                          │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│      后端抽象层 (sqlite.ts)         │
│  - SQLite 后端（默认）                 │
│  - 内存后端（测试）                  │
│  - 远程 HTTP 后端（可选）              │
│  - 向量数据库集成（sqlite-vec）       │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│      嵌入生成层 (embeddings.ts)     │
│  - 多种嵌入模型支持                  │
│  - 批量生成优化                      │
│  - 缓存复用                          │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│      查询优化层 (mmr.ts)            │
│  - 最大边际相关性（MMR）              │
│  - 相似度搜索                        │
│  - 混合检索（关键词 + 向量）           │
└─────────────────────────────────────┘
```

### 12.3 关键模式

#### 12.3.1 存储接口模式

```typescript
interface MemoryStore {
  // 基本操作
  insert(doc: MemoryDocument): Promise<void>;
  delete(id: string): Promise<void>;
  update(id: string, doc: Partial<MemoryDocument>): Promise<void>;

  // 查询操作
  search(query: string, options: SearchOptions): Promise<MemoryDocument[]>;
  similar(docId: string, limit: number): Promise<MemoryDocument[]>;

  // 管理操作
  clear(): Promise<void>;
  stats(): Promise<MemoryStats>;
}
```

#### 12.3.2 嵌入生成模式

```typescript
interface EmbeddingProvider {
  model: string;
  dimension: number;

  // 批量生成（优化 API 调用）
  generateBatch(texts: string[]): Promise<number[][]>;

  // 缓存键生成
  cacheKey(text: string): string;
}

// 多提供商支持
const providers = {
  openai: createOpenAIEmbedder(),
  gemini: createGeminiEmbedder(),
  voyage: createVoyageEmbedder(),
  "node-llama": createLocalEmbedder(),
};
```

#### 12.3.3 MMR 检索模式

```typescript
// 最大边际相关性：平衡相关性和多样性
function maximalMarginalRelevance(
  query: number[],
  docs: MemoryDocument[],
  lambda: number = 0.5, // 0=纯相关性，1=纯多样性
  k: number
): MemoryDocument[] {
  const selected: MemoryDocument[] = [];
  const remaining = [...docs];

  while (selected.length < k && remaining.length > 0) {
    // 选择最高 MMR 分数的文档
    const best = selectBestMMR(query, remaining, selected, lambda);
    selected.push(best);
    remove(remaining, best);
  }

  return selected;
}
```

#### 12.3.4 过期清理模式

```typescript
{
  memory: {
    retention: {
      maxDocuments: 10000,      // 最大文档数
      maxAge: '30d',            // 最大年龄
      maxSizeBytes: '1gb',      // 最大磁盘占用
      cleanupInterval: '1h'     // 清理间隔
    }
  }
}
```

### 12.4 最佳实践

1. **批量操作**：嵌入生成使用批量 API 减少调用次数
2. **缓存策略**：缓存已生成的嵌入避免重复计算
3. **混合检索**：结合关键词（BM25）和向量检索
4. **定期清理**：设置 retention 策略防止无限增长
5. **监控指标**：跟踪存储大小、查询延迟、命中率

---

## 13. 测试系统架构

### 13.1 核心理念

**分层测试，自动化优先**

- 单元测试为基础
- 集成测试验证模块交互
- E2E 测试模拟真实场景
- Docker 隔离测试环境

### 13.2 测试金字塔

```
           ╱╲
          ╱  ╲         E2E 测试（Docker）
         ╱────╲        - 完整场景
        ╱      ╲       - 多服务交互
       ╱────────╲
      ╱          ╲     集成测试
     ╱            ╲    - 模块间交互
    ╱──────────────╲   - 真实依赖
   ╱                ╲
  ╱──────────────────╲  单元测试
                      ╲ - 纯函数
                       ╲- Mock 外部依赖
```

### 13.3 关键模式

#### 13.3.1 测试分类模式

```typescript
// 单元测试（快速，隔离）
describe("retry logic", () => {
  it("should retry with exponential backoff", async () => {
    const mockFn = vi
      .fn()
      .mockRejectedValueOnce(new Error("fail"))
      .mockResolvedValueOnce("success");

    const result = await retryAsync(mockFn, { attempts: 3 });
    expect(result).toBe("success");
    expect(mockFn).toHaveBeenCalledTimes(2);
  });
});

// 集成测试（中等速度，真实依赖）
describe("config loading", () => {
  it("should load config with env substitution", async () => {
    process.env.TEST_API_KEY = "secret123";
    const config = await loadConfigFromFixture("env-substitution.json5");
    expect(config.gateway.auth.token).toBe("secret123");
  });
});

// E2E 测试（慢速，完整环境）
// scripts/e2e/onboard-docker.sh
// 1. 构建 Docker 镜像
// 2. 运行 onboarding
// 3. 验证配置生成
// 4. 清理容器
```

#### 13.3.2 测试环境隔离模式

```bash
# Docker 测试容器
docker run --rm \
  -e OPENAI_API_KEY=fake_key \
  -e TELEGRAM_BOT_TOKEN=fake_token \
  openclaw:test \
  pnpm test:docker:all

# 测试后自动清理
scripts/test-cleanup-docker.sh
```

#### 13.3.3 测试覆盖率模式

```json
{
  "test:coverage": "vitest run --coverage",
  "test:coverage:thresholds": {
    "lines": 70,
    "branches": 70,
    "functions": 70,
    "statements": 70
  }
}
```

#### 13.3.4 实时测试模式

```bash
# 使用真实 API 密钥（可选）
OPENCLAW_LIVE_TEST=1 pnpm test:live

# Docker 中的实时测试
pnpm test:docker:live-models
pnpm test:docker:live-gateway
```

### 13.4 最佳实践

1. **测试命名**：描述行为而非函数（`should retry when fails`）
2. **隔离性**：测试间不共享状态
3. **可重复性**：使用固定随机种子
4. **快速反馈**：单元测试 < 100ms
5. **覆盖率门槛**：至少 70% 覆盖率

---

## 14. 部署与发布系统

### 14.1 核心理念

**多平台支持，自动化发布**

- macOS/iOS/Android/CLI/Docker 多平台
- 自动化打包和签名
- 版本一致性检查
- 回滚机制

### 14.2 平台矩阵

| 平台    | 打包方式    | 分发渠道               | 自动化程度 |
| ------- | ----------- | ---------------------- | ---------- |
| macOS   | .app + .dmg | Sparkle + Homebrew     | 完全自动化 |
| iOS     | .ipa        | TestFlight + App Store | 半自动化   |
| Android | .apk        | GitHub Releases        | 完全自动化 |
| CLI     | npm         | npm registry           | 完全自动化 |
| Docker  | 多架构镜像  | Docker Hub             | 完全自动化 |

### 14.3 关键模式

#### 14.3.1 版本一致性模式

```typescript
// scripts/release-check.ts
// 检查所有版本文件一致性
const versionLocations = {
  "package.json": "version",
  "apps/android/app/build.gradle.kts": "versionName",
  "apps/ios/Sources/Info.plist": "CFBundleShortVersionString",
  "apps/macos/Sources/Info.plist": "CFBundleShortVersionString",
  "docs/install/updating.md": "pinned version",
};

// 所有版本必须一致才能发布
```

#### 14.3.2 macOS 打包模式

```bash
# scripts/package-mac-app.sh
# 1. 构建 Swift 项目
xcodebuild -project apps/macos/OpenClaw.xcodeproj \
  -scheme OpenClaw \
  -configuration Release \
  build

# 2. 代码签名
codesign --deep --force --sign "Developer ID" \
  dist/OpenClaw.app

# 3. 公证（Notarization）
xcrun notarytool submit dist/OpenClaw.zip \
  --apple-id "$APPLE_ID" \
  --team-id "$TEAM_ID" \
  --password "$APP_SPECIFIC_PASSWORD"

# 4. 创建 DMG
create-dmg dist/OpenClaw.app dist/
```

#### 14.3.3 npm 发布模式

```bash
# 使用 1Password 管理 OTP
tmux new -d -s release
eval "$(op signin --account my.1password.com)"
OTP=$(op read 'op://Private/Npmjs/one-time password?attribute=otp')

# 发布
npm publish --access public --otp="$OTP"

# 验证
npm view openclaw version --userconfig "$(mktemp)"

# 清理
tmux kill-session -t release
```

#### 14.3.4 Docker 发布模式

```dockerfile
# Dockerfile
FROM node:22-alpine

# 安装依赖
RUN npm install -g openclaw@latest

# 配置工作目录
WORKDIR /workspace

# 暴露网关端口
EXPOSE 18789

# 启动命令
CMD ["openclaw", "gateway", "run"]
```

```bash
# 多架构构建
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t openclaw/openclaw:latest \
  --push .
```

#### 14.3.5 发布检查清单

```typescript
// scripts/release-check.ts
const releaseChecklist = [
  "所有测试通过",
  "版本号一致性",
  "CHANGELOG 已更新",
  "文档已更新",
  "macOS 签名有效",
  "npm 包可安装",
  "Docker 镜像可运行",
  "安装脚本测试通过",
];
```

### 14.4 最佳实践

1. **版本锁定**：使用 lockfile 确保依赖一致性
2. **自动化测试**：发布前自动运行所有测试
3. **渐进发布**：先发布 beta，再发布 stable
4. **回滚计划**：保留上一个稳定版本
5. **发布说明**：自动生成 changelog

---

## 15. CI/CD 与工作流

### 15.1 核心理念

**智能检测，并行执行**

- 基于变更范围跳过无关任务
- 多平台并行测试
- 缓存优化构建速度
- 失败自动重试

### 15.2 工作流架构

```
┌─────────────────────────────────────┐
│      变更检测层 (changed-scope)     │
│  - 检测文档变更（跳过重型任务）      │
│  - 检测代码变更范围                  │
│  - 决定运行哪些测试                  │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│      并行执行层                     │
│  ├─ Node.js 测试（Linux）            │
│  ├─ macOS 构建测试                   │
│  ├─ Android 构建测试                 │
│  └─ Docker E2E 测试                   │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│      缓存优化层 (setup-pnpm-store)  │
│  - pnpm store 缓存                   │
│  - Docker 层缓存                      │
│  - 构建产物缓存                      │
└─────────────────────────────────────┘
```

### 15.3 关键工作流

#### 15.3.1 CI 工作流（ci.yml）

```yaml
# 智能变更检测
- name: Detect docs-only changes
  uses: ./.github/actions/detect-docs-changes

# 文档变更：只运行 lint/format
# 代码变更：运行完整测试套件

# 并发控制
concurrency:
  group: ci-${{ workflow }}-${{ pr.number || ref }}
  cancel-in-progress: true  # PR 更新取消旧任务
```

#### 15.3.2 安装烟测（install-smoke.yml）

```yaml
# 验证安装脚本可用性
- name: Run installer docker tests
  env:
    CLAWDBOT_INSTALL_URL: https://openclaw.ai/install.sh
  run: pnpm test:install:smoke
# 测试场景：
# 1. root 用户安装
# 2. 非 root 用户安装
# 3. 上一个版本安装
```

#### 15.3.3 Docker 发布（docker-release.yml）

```yaml
# 多架构构建
- name: Build and push Docker image
  uses: docker/build-push-action@v5
  with:
    platforms: linux/amd64,linux/arm64
    tags: |
      openclaw/openclaw:latest
      openclaw/openclaw:${{ version }}
    push: true
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

#### 15.3.4 自动化响应（auto-response.yml）

```yaml
# 自动标记 PR
- name: Label PR
  uses: actions/labeler@v4

# 自动回复 Issue
- name: Auto-respond to Issue
  uses: ./.github/actions/auto-respond
```

### 15.4 GitHub Actions 最佳实践

#### 15.4.1 自定义 Action 复用

```yaml
# .github/actions/setup-pnpm-store-cache/action.yml
name: Setup pnpm Store Cache
inputs:
  pnpm-version:
    required: true
  cache-key-suffix:
    default: ""
runs:
  using: composite
  steps:
    - name: Setup pnpm
      uses: pnpm/action-setup@v2
      with:
        version: ${{ inputs.pnpm-version }}

    - name: Cache pnpm store
      uses: actions/cache@v3
      with:
        path: ~/.pnpm-store
        key: pnpm-store-${{ runner.os }}-${{ hashFiles('pnpm-lock.yaml') }}
```

#### 15.4.2 并发控制

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}
```

#### 15.4.3 失败重试

```yaml
- name: Flaky test retry
  uses: nick-fields/retry@v2
  with:
    timeout_minutes: 10
    max_attempts: 3
    command: pnpm test:live
```

### 15.5 最佳实践

1. **快速失败**：先运行快速测试，失败立即停止
2. **缓存策略**：缓存依赖和构建产物
3. **并行化**：独立任务并行执行
4. **资源优化**：根据变更范围跳过任务
5. **可观测性**：详细的日志和错误信息

---

## 16. 国际化系统

### 16.1 核心理念

**翻译记忆，术语统一**

- 翻译记忆库（TMX）避免重复翻译
- 术语表保证一致性
- 自动化翻译流程
- 人工审核关键内容

### 16.2 架构层次

```
┌─────────────────────────────────────┐
│      文档提取层 (docs/.i18n/)       │
│  - 提取英文文档                      │
│  - 生成翻译单元                      │
│  - 更新翻译记忆库                    │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│      翻译引擎层 (scripts/docs-i18n) │
│  - 调用翻译 API（DeepL/OpenAI）       │
│  - 应用翻译记忆                      │
│  - 应用术语表                        │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│      质量检查层                     │
│  - 链接检查                          │
│  - 代码块检查                        │
│  - 格式检查                          │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│      生成层                         │
│  - 生成 zh-CN 文档                    │
│  - 更新导航结构                      │
│  - 同步索引                          │
└─────────────────────────────────────┘
```

### 16.3 关键模式

#### 16.3.1 翻译记忆库模式

```typescript
// docs/.i18n/zh-CN.tm.jsonl
{"source":"Configuration","target":"配置"}
{"source":"Gateway","target":"网关"}
{"source":"Session","target":"会话"}

// 翻译时优先匹配记忆库
function translateWithMemory(
  source: string,
  memory: TranslationMemory
): string | null {
  return memory.get(source)?.target || null;
}
```

#### 16.3.2 术语表模式

```json
// docs/.i18n/glossary.zh-CN.json
{
  "Gateway": "网关",
  "Channel": "通道",
  "Plugin": "插件",
  "Session": "会话",
  "Memory": "内存",
  "Webhook": "Webhook",
  "Pairing": "配对"
}

// 强制使用术语表翻译
function applyGlossary(
  translation: string,
  glossary: Glossary
): string {
  for (const [en, zh] of Object.entries(glossary)) {
    translation = translation.replace(
      new RegExp(`\\b${en}\\b`, 'g'),
      zh
    );
  }
  return translation;
}
```

#### 16.3.3 自动化流程模式

```bash
# scripts/docs-i18n
# 1. 提取英文文档
extract_docs docs/*.md

# 2. 更新翻译记忆
update_translation_memory

# 3. 调用翻译 API
translate_with_deepl \
  --memory docs/.i18n/zh-CN.tm.jsonl \
  --glossary docs/.i18n/glossary.zh-CN.json

# 4. 生成中文文档
generate_docs docs/zh-CN/

# 5. 人工审核（可选）
# 检查关键章节翻译质量
```

### 16.4 最佳实践

1. **术语先行**：先定义术语表再翻译
2. **记忆复用**：避免相同内容重复翻译
3. **机器 + 人工**：机器翻译初稿，人工审核关键内容
4. **增量更新**：只翻译变更的文档
5. **格式保持**：保持 Markdown 格式和链接完整

---

## 17. 文档系统

### 17.1 核心理念

**文档即代码，自动化生成**

- 文档与代码同库管理
- 自动化链接检查
- 自动化拼写检查
- 版本化文档

### 17.2 架构层次

```
┌─────────────────────────────────────┐
│      文档组织层 (docs/)             │
│  - 按功能模块分目录                  │
│  - 统一的 frontmatter                │
│  - 内部链接规范                      │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│      构建层 (Mintlify)              │
│  - Markdown → HTML                  │
│  - 搜索索引生成                      │
│  - SEO 优化                           │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│      质量检查层                     │
│  - 链接审计（docs-link-check）       │
│  - 拼写检查（cspell）                │
│  - 格式检查（markdownlint）          │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│      部署层                         │
│  - Vercel/Mintlify 部署              │
│  - CDN 缓存                           │
│  - 多语言路由                        │
└─────────────────────────────────────┘
```

### 17.3 关键模式

#### 17.3.1 文档结构模式

```
docs/
├── concepts/          # 核心概念
│   ├── architecture.md
│   ├── session.md
│   └── models.md
├── gateway/           # 网关相关
│   ├── configuration.md
│   ├── security/
│   └── protocol.md
├── channels/          # 通道相关
│   ├── whatsapp.md
│   ├── telegram.md
│   └── discord.md
├── automation/        # 自动化
│   ├── cron.md
│   └── hooks.md
├── reference/         # API 参考
│   ├── cli.md
│   └── config.md
└── help/              # 帮助
    ├── troubleshooting.md
    └── faq.md
```

#### 17.3.2 Frontmatter 模式

```markdown
---
title: "Gateway Architecture"
summary: "WebSocket gateway architecture"
read_when:
  - Working on gateway protocol
  - Working on clients
tags:
  - architecture
  - gateway
---
```

#### 17.3.3 链接规范模式

```markdown
<!-- 内部链接：根相对路径，无 .md 后缀 -->

[Config](/configuration)
[Hooks](/configuration#hooks)

<!-- 外部链接：完整 URL -->

[TypeBox](https://github.com/sinclairzx81/typebox)

<!-- 避免的链接格式 -->

❌ [Config](./gateway/configuration.md) <!-- 相对路径 + 后缀 -->
✅ [Config](/gateway/configuration) <!-- 根相对路径 -->
```

#### 17.3.4 质量检查模式

```bash
# 链接审计
pnpm docs:check-links
# 检查所有内部链接是否有效
# 报告断链和无效锚点

# 拼写检查
pnpm docs:spellcheck
# 使用 cspell 检查拼写错误
# 支持自定义词典

# Markdown 格式检查
pnpm lint:docs
# 使用 markdownlint 检查格式
# 统一的文档风格
```

#### 17.3.5 API 文档自动生成

```typescript
// 从 TypeBox Schema 生成文档
// scripts/protocol-gen.ts
const schema = ConnectParamsSchema;
const markdown = `
## ConnectParams

\`\`\`typescript
${generateTypeScriptInterface(schema)}
\`\`\`

### Properties

${generatePropertiesTable(schema)}
`;
```

### 17.4 最佳实践

1. **单一来源**：文档只在一个地方定义
2. **自动化检查**：CI 自动检查链接和拼写
3. **版本控制**：文档与代码版本同步
4. **搜索友好**：清晰的标题和摘要
5. **示例驱动**：每个概念配示例代码

---

## 18. 项目管理最佳实践

### 18.1 代码组织

#### 18.1.1 目录结构约定

```
openclaw/
├── src/                 # 源代码
│   ├── gateway/        # 网关核心
│   ├── channels/       # 通道实现
│   ├── plugins/        # 插件系统
│   └── infra/          # 基础设施
├── extensions/          # 扩展插件
│   ├── bluebubbles/
│   ├── matrix/
│   └── voice-call/
├── apps/                # 客户端应用
│   ├── macos/
│   ├── ios/
│   └── android/
├── docs/                # 文档
├── scripts/             # 构建/部署脚本
├── .github/             # CI/CD 配置
└── tests/               # 测试（部分与源码同目录）
```

#### 18.1.2 文件命名规范

```typescript
// 测试文件：*.test.ts
retry.test.ts;
config.test.ts;

// 类型定义：types.ts 或 *.types.ts
types.ts;
config.types.ts;

// 实现文件：功能名.ts
config.ts;
gateway.ts;

// 工具函数：utils.ts 或功能 -utils.ts
path - utils.ts;
string - utils.ts;
```

### 18.2 Git 工作流

#### 18.2.1 分支策略

```
main              # 主分支，随时可发布
├── feature/*     # 功能分支
├── bugfix/*      # 修复分支
├── release/*     # 发布分支
└── hotfix/*      # 紧急修复
```

#### 18.2.2 提交信息规范

```bash
# Conventional Commits
<type>(<scope>): <subject>

# 类型
feat:     新功能
fix:      Bug 修复
docs:     文档更新
style:    格式调整
refactor: 重构
test:     测试
chore:    构建/工具

# 示例
feat(channels): add WhatsApp group mention support
fix(config): resolve env substitution edge case
docs(gateway): update authentication guide
```

#### 18.2.3 PR 流程

```
1. 创建功能分支
   git checkout -b feature/my-feature

2. 开发并提交
   git commit -m "feat: add feature"

3. 运行测试
   pnpm test
   pnpm lint

4. 推送并创建 PR
   git push origin feature/my-feature

5. Code Review
   - 至少 1 人批准
   - 所有 CI 检查通过

6. 合并到 main
   - Squash merge
   - 删除功能分支
```

### 18.3 依赖管理

#### 18.3.1 包管理器选择

```json
{
  "packageManager": "pnpm@10.23.0",
  "pnpm": {
    "minimumReleaseAge": 2880, // 包年龄限制（安全）
    "overrides": {
      "hono": "4.11.10" // 强制版本（安全修复）
    }
  }
}
```

#### 18.3.2 依赖分类

```json
{
  "dependencies": {
    // 运行时依赖
    "@sinclair/typebox": "0.34.48",
    "commander": "^14.0.3"
  },
  "devDependencies": {
    // 开发时依赖
    "vitest": "^4.0.18",
    "typescript": "^5.9.3"
  },
  "peerDependencies": {
    // 对等依赖
    "@napi-rs/canvas": "^0.1.89"
  },
  "optionalDependencies": {
    // 可选依赖
    "@discordjs/opus": "^0.10.0"
  }
}
```

### 18.4 质量管理

#### 18.4.1 代码检查清单

```bash
# 预检查
pnpm format:check    # 格式检查
pnpm lint            # Lint 检查
pnpm tsgo            # 类型检查
pnpm test            # 单元测试

# 构建检查
pnpm build           # 构建验证
pnpm protocol:check  # 协议一致性

# 文档检查
pnpm docs:check-links
pnpm docs:spellcheck
```

#### 18.4.2 发布检查清单

```bash
# 发布前检查
pnpm release:check

# 检查项：
# ✓ 所有测试通过
# ✓ 版本号一致性
# ✓ CHANGELOG 更新
# ✓ 文档更新
# ✓ 无 TODO/FIXME
# ✓ 性能回归检查
```

### 18.5 团队协作

#### 18.5.1 Code Review 指南

```markdown
## Reviewer 职责

- [ ] 代码正确性
- [ ] 测试覆盖
- [ ] 文档更新
- [ ] 性能影响
- [ ] 安全风险

## Review 时间

- 小 PR (< 200 行): 24 小时内
- 中 PR (200-500 行): 48 小时内
- 大 PR (> 500 行): 拆分或安排专门 review 时间
```

#### 18.5.2 Issue 管理

```markdown
## Issue 模板

### Bug Report

- 重现步骤
- 预期行为
- 实际行为
- 环境信息

### Feature Request

- 使用场景
- 期望功能
- 替代方案
```

### 18.6 知识管理

#### 18.6.1 决策记录（ADR）

```markdown
# ADR-001: 选择 TypeBox 作为协议 Schema

## 状态

Accepted

## 背景

需要统一的协议定义方式，同时支持：

- TypeScript 类型推导
- 运行时验证
- JSON Schema 生成
- Swift 代码生成

## 决策

使用 TypeBox 作为单一来源

## 后果

### 正面

- 类型安全
- 减少重复代码
- 自动生成文档

### 负面

- 学习曲线
- 额外的构建步骤
```

#### 18.6.2 技能文档

```markdown
## AGENTS.md / skills/

为 AI 助手提供的指南：

- 项目结构
- 编码规范
- 常用命令
- 陷阱和注意事项
```

### 18.7 最佳实践

1. **文档化一切**：决策、流程、陷阱
2. **自动化重复工作**：构建、测试、部署
3. **小步快跑**：小 PR，频繁合并
4. **质量内建**：测试先行，持续集成
5. **知识共享**：定期技术分享，文档更新

---

## 总结

OpenClaw 项目的架构思想可以概括为以下核心原则：

### 技术架构原则

1. **类型驱动**：Schema 定义一切，类型即文档
2. **安全优先**：零信任、最小权限、严格验证
3. **插件隔离**：稳定 API、运行时注入、禁止直接导入
4. **优雅容错**：重试、退避、熔断、降级
5. **可观测性**：结构化日志、诊断上下文、性能追踪
6. **模块化**：明确边界、依赖注入、可测试优先
7. **媒体统一**：统一接口、按需处理、透明优化
8. **内存抽象**：多后端支持、智能检索、自动清理

### 项目管理原则

1. **测试分层**：单元 → 集成 → E2E，自动化优先
2. **持续集成**：智能检测、并行执行、快速反馈
3. **多平台部署**：自动化打包、版本一致、渐进发布
4. **文档即代码**：自动化检查、版本控制、搜索友好
5. **国际化**：翻译记忆、术语统一、人机协作
6. **质量内建**：预检查清单、发布检查、持续改进
7. **知识管理**：决策记录、技能文档、团队协作

这些架构模式不仅适用于 OpenClaw，也可以作为通用范式应用于其他项目。

---

**相关文档**：

- [Gateway Architecture](/concepts/architecture)
- [Configuration Reference](/gateway/configuration-reference)
- [Plugin SDK Refactor](/refactor/plugin-sdk)
- [TypeBox](/concepts/typebox)
- [Session Management](/concepts/session)
- [Testing Guide](/help/testing)
- [CI/CD Workflows](/.github/workflows)
- [Release Process](/reference/RELEASING.md)

---

## 附录 A：术语表

| 术语    | 英文                            | 定义                                           |
| ------- | ------------------------------- | ---------------------------------------------- |
| 网关    | Gateway                         | WebSocket 网关服务，连接客户端和 AI 服务       |
| 通道    | Channel                         | 消息通道实现（如 WhatsApp、Telegram、Discord） |
| 插件    | Plugin                          | 扩展功能模块，通过 SDK 与核心隔离              |
| 会话    | Session                         | 用户对话上下文，包含历史记录和状态             |
| 配对    | Pairing                         | 设备认证流程，生成配对码批准连接               |
| 幂等键  | Idempotency Key                 | 防止重复操作的唯一标识符                       |
| Schema  | Schema                          | 数据结构定义，用于验证和类型推导               |
| TypeBox | TypeBox                         | TypeScript 运行时类型系统库                    |
| Zod     | Zod                             | TypeScript Schema 验证库                       |
| 熔断器  | Circuit Breaker                 | 保护系统免于连续失败的容错模式                 |
| 退避    | Backoff                         | 重试延迟递增策略                               |
| 抖动    | Jitter                          | 随机化延迟避免同步重试                         |
| 记忆库  | Memory                          | 向量数据库支持的长期记忆存储                   |
| 嵌入    | Embedding                       | 文本的向量表示，用于相似度搜索                 |
| MMR     | Maximal Marginal Relevance      | 最大边际相关性，平衡相关性和多样性             |
| ADR     | Architecture Decision Record    | 架构决策记录文档                               |
| E2E     | End-to-End                      | 端到端测试，模拟完整用户场景                   |
| CI/CD   | Continuous Integration/Delivery | 持续集成/持续部署                              |

---

## 附录 B：常见问题（FAQ）

### B.1 架构设计

#### Q1: 什么时候应该使用插件而不是核心模块？

**A**: 当功能满足以下条件时使用插件：

- 可选功能，非核心需求
- 需要独立发布周期
- 第三方开发者可能扩展
- 需要与核心代码隔离（如实验性功能）

**核心模块**适用于：

- 基础功能（配置、日志、网关）
- 所有用户都需要的功能
- 性能敏感的核心路径

#### Q2: 如何选择会话作用域？

**A**: 根据隔离需求和隐私级别：

| 作用域             | 适用场景            | 示例                          |
| ------------------ | ------------------- | ----------------------------- |
| `main`             | 单用户、个人助理    | 个人使用的 AI 助手            |
| `per-peer`         | 多用户 DM、基础隔离 | 家庭共享设备                  |
| `per-channel-peer` | 严格隔离、隐私敏感  | 企业多租户场景                |
| `group`            | 群组会话            | WhatsApp 群组、Discord 服务器 |

#### Q3: 为什么选择 TypeBox 而不是纯 TypeScript 接口？

**A**: TypeBox 提供：

- **运行时验证**：TypeScript 类型只在编译时有效，TypeBox 可在运行时验证
- **单一来源**：从 Schema 同时生成类型和验证逻辑，避免不一致
- **代码生成**：可生成 JSON Schema、Swift 模型等
- **文档生成**：自动从 Schema 生成 API 文档

**代价**：

- 学习曲线
- 运行时验证开销（通常 < 1ms/对象）

### B.2 开发实践

#### Q4: 如何添加新的消息通道？

**A**: 推荐两种方式：

**方式 1：插件（推荐）**

```bash
# 1. 创建插件目录
mkdir extensions/my-channel

# 2. 初始化插件
cd extensions/my-channel
npm init -y

# 3. 实现 ChannelPlugin 接口
# 4. 注册到配置
```

**方式 2：核心模块**

```bash
# 1. 在 src/channels 创建新目录
mkdir src/channels/my-channel

# 2. 实现通道接口
# 3. 更新通道注册表
# 4. 添加配置 Schema
```

**选择建议**：优先使用插件，除非是官方核心通道。

#### Q5: 配置修改后如何生效？

**A**: 三种方式：

1. **热重载**（如果启用）：

   ```bash
   openclaw configure reload
   ```

2. **软重启**（保持会话）：

   ```bash
   openclaw gateway restart
   ```

3. **硬重启**（完整重启）：

   ```bash
   # 停止网关
   pkill -f openclaw-gateway

   # 重新启动
   openclaw gateway run
   ```

#### Q6: 如何调试网关连接问题？

**A**: 诊断步骤：

```bash
# 1. 检查网关状态
openclaw gateway status

# 2. 查看日志
openclaw logs --tail 100

# 3. 测试连接
openclaw health --deep

# 4. 检查配置
openclaw config get --json | jq '.gateway'

# 5. 网络诊断
netstat -an | grep 18789
```

### B.3 性能优化

#### Q7: 系统内存占用过高怎么办？

**A**: 优化建议：

1. **限制会话大小**：

   ```json5
   {
     session: {
       maintenance: {
         maxEntries: 500,
         maxDiskBytes: "1gb",
       },
     },
   }
   ```

2. **配置记忆清理**：

   ```json5
   {
     memory: {
       retention: {
         maxDocuments: 10000,
         maxAge: "30d",
       },
     },
   }
   ```

3. **减少并发连接**：

   ```json5
   {
     gateway: {
       maxConnections: 10,
     },
   }
   ```

4. **监控工具**：
   ```bash
   # 实时监控内存
   top -pid $(pgrep -f openclaw-gateway)
   ```

#### Q8: 如何优化响应速度？

**A**: 性能调优方向：

1. **模型选择**：使用更快的模型（如 `gpt-4o-mini` 代替 `gpt-4`）
2. **启用缓存**：缓存常用响应和嵌入
3. **减少上下文**：限制会话历史长度
4. **批量操作**：嵌入生成使用批量 API
5. **本地模型**：使用 `node-llama` 本地推理（延迟更低）

### B.4 安全与运维

#### Q9: 如何备份和恢复配置？

**A**: 备份：

```bash
# 1. 备份配置文件
cp ~/.openclaw/config.json5 ~/.openclaw/config.backup.$(date +%Y%m%d).json5

# 2. 备份会话
tar -czf sessions-backup.tar.gz ~/.openclaw/sessions/

# 3. 备份凭证
cp -r ~/.openclaw/credentials ~/.openclaw/credentials.backup
```

恢复：

```bash
# 1. 恢复配置
cp ~/.openclaw/config.backup.20260303.json5 ~/.openclaw/config.json5

# 2. 恢复会话
tar -xzf sessions-backup.tar.gz -C ~/

# 3. 重启网关
openclaw gateway restart
```

#### Q10: 设备配对失败如何处理？

**A**: 排查步骤：

1. **检查配对码是否正确**（6 位数字）
2. **确认网络连通性**：
   ```bash
   ping <gateway-host>
   ```
3. **查看网关日志**：
   ```bash
   openclaw logs --grep pairing
   ```
4. **重置配对状态**：
   ```bash
   openclaw pairing reset --device <device-id>
   ```
5. **重新生成配对码**：
   ```bash
   openclaw pairing generate
   ```

---

## 附录 C：检查清单

### C.1 添加新插件检查清单

- [ ] 定义插件 Manifest（`name`, `version`, `channelTypes`）
- [ ] 实现 `ChannelPlugin` 接口
- [ ] 添加配置 Schema（使用 `sdk.config.buildSchema`）
- [ ] 实现消息发送/接收逻辑
- [ ] 处理媒体附件（如有）
- [ ] 编写单元测试（覆盖率 > 70%）
- [ ] 更新插件文档
- [ ] 通过模块边界检查
- [ ] 测试与网关的集成
- [ ] 发布到 npm（如果是独立插件）

### C.2 发布前检查清单

- [ ] 所有测试通过（`pnpm test`）
- [ ] 类型检查通过（`pnpm tsgo`）
- [ ] Lint 检查通过（`pnpm lint`）
- [ ] 格式检查通过（`pnpm format:check`）
- [ ] 构建验证（`pnpm build`）
- [ ] 版本号一致性检查（`pnpm release:check`）
- [ ] CHANGELOG 已更新
- [ ] 文档已更新（包括中英文）
- [ ] 无 TODO/FIXME 遗留
- [ ] 性能回归检查
- [ ] 安全审计（依赖漏洞扫描）

### C.3 故障排查检查清单

**网关无法启动**：

- [ ] 检查配置文件语法（JSON5 格式）
- [ ] 验证环境变量是否设置
- [ ] 检查端口是否被占用（`ss -ltnp | grep 18789`）
- [ ] 查看日志错误信息
- [ ] 运行 `openclaw doctor` 诊断

**消息发送失败**：

- [ ] 检查通道是否启用
- [ ] 验证通道配置（token、API key）
- [ ] 检查网络连接
- [ ] 查看通道特定日志
- [ ] 测试通道连接状态（`channels status --probe`）

**会话数据丢失**：

- [ ] 检查会话文件是否存在
- [ ] 验证会话键格式是否正确
- [ ] 查看会话维护日志
- [ ] 检查磁盘空间
- [ ] 恢复备份（如有）

---

**文档版本**：2026.3.3  
**最后更新**：2026-03-03  
**维护者**：OpenClaw Team
