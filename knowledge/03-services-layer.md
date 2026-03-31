# 03 - Services Layer 服务层深度分析

> 源码路径: `src/services/`
> 文件数: ~100+ TypeScript 文件, 20+ 子目录
> 核心职责: API 客户端、MCP 集成、上下文压缩、认证、分析、企业管理

---

## 目录

1. [整体架构概览](#1-整体架构概览)
2. [API 客户端架构](#2-api-客户端架构)
3. [MCP 集成系统](#3-mcp-集成系统)
4. [上下文压缩与 Token 管理](#4-上下文压缩与-token-管理)
5. [认证与 OAuth 系统](#5-认证与-oauth-系统)
6. [分析与特性开关系统](#6-分析与特性开关系统)
7. [LSP 集成](#7-lsp-集成)
8. [插件系统](#8-插件系统)
9. [企业级服务](#9-企业级服务)
10. [团队记忆同步](#10-团队记忆同步)
11. [自动记忆提取](#11-自动记忆提取)
12. [可复用的服务模式](#12-可复用的服务模式)

---

## 1. 整体架构概览

### 服务层目录结构

```
src/services/
├── api/                    # Anthropic API 客户端 (~300K 代码)
│   ├── claude.ts           # 核心查询引擎 (~122K) - 流式/非流式 API 调用
│   ├── client.ts           # SDK 客户端工厂 (15.8K) - 多提供商支持
│   ├── errors.ts           # 错误分类与消息 (40.8K)
│   ├── withRetry.ts        # 重试策略引擎 (27.6K)
│   ├── logging.ts          # API 日志与遥测 (23.6K)
│   ├── bootstrap.ts        # 引导 API 请求
│   ├── filesApi.ts         # 文件上传 API
│   ├── usage.ts            # 用量查询
│   ├── sessionIngress.ts   # 会话日志持久化
│   └── promptCacheBreakDetection.ts  # 缓存命中率分析
├── mcp/                    # MCP 协议集成 (~250K 代码)
│   ├── client.ts           # MCP 客户端 (~116K) - 服务器连接/工具发现
│   ├── auth.ts             # OAuth/XAA 认证 (~86K)
│   ├── MCPConnectionManager.tsx  # React 连接管理上下文
│   ├── types.ts            # Zod schema 配置类型
│   ├── config.ts           # 多源配置合并
│   └── xaa.ts / xaaIdpLogin.ts  # 跨应用访问
├── compact/                # 上下文压缩子系统
│   ├── compact.ts          # 全量压缩算法
│   ├── autoCompact.ts      # 自动触发逻辑
│   ├── microCompact.ts     # 工具结果微压缩
│   ├── apiMicrocompact.ts  # API 端上下文管理
│   ├── grouping.ts         # 消息分组 (按 API 轮次)
│   └── prompt.ts           # 压缩摘要提示词
├── oauth/                  # OAuth 2.0 客户端
├── analytics/              # GrowthBook + 遥测
├── lsp/                    # Language Server Protocol
├── plugins/                # 插件/市场管理
├── policyLimits/           # 组织级策略限制
├── remoteManagedSettings/  # 远程配置同步
├── extractMemories/        # 自动记忆提取
├── teamMemorySync/         # 团队记忆同步
├── tokenEstimation.ts      # Token 估算算法
└── ...其他辅助服务
```

### 设计原则

1. **多提供商抽象**: 单一 `getAnthropicClient()` 工厂支持 Anthropic 1P / Bedrock / Vertex / Foundry
2. **Fail-Open 模式**: 企业服务 (policyLimits, remoteManagedSettings) 失败时不阻塞核心功能
3. **渐进式增强**: 通过 `feature()` 编译时门控实现功能的选择性启用
4. **会话稳定性**: Beta header 锁存 (latch) 机制避免缓存键变化导致的缓存失效
5. **异步生成器 (AsyncGenerator)**: 核心 API 调用通过 `yield` 实现流式事件传递

---

## 2. API 客户端架构

### 2.1 客户端工厂 — 多提供商统一入口

`src/services/api/client.ts` 提供统一的客户端创建入口，支持 4 个 API 提供商:

```typescript
// src/services/api/client.ts:88-316
export async function getAnthropicClient({
  apiKey, maxRetries, model, fetchOverride, source,
}: { ... }): Promise<Anthropic> {
  const defaultHeaders: { [key: string]: string } = {
    'x-app': 'cli',
    'User-Agent': getUserAgent(),
    'X-Claude-Code-Session-Id': getSessionId(),
    ...customHeaders,
  }

  await checkAndRefreshOAuthTokenIfNeeded()

  const ARGS = {
    defaultHeaders,
    maxRetries,
    timeout: parseInt(process.env.API_TIMEOUT_MS || String(600 * 1000), 10),
    dangerouslyAllowBrowser: true,
    fetchOptions: getProxyFetchOptions({ forAnthropicAPI: true }),
  }

  // 提供商分支 — 按优先级检测环境变量
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK)) {
    const { AnthropicBedrock } = await import('@anthropic-ai/bedrock-sdk')
    return new AnthropicBedrock(bedrockArgs) as unknown as Anthropic
  }
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY)) {
    const { AnthropicFoundry } = await import('@anthropic-ai/foundry-sdk')
    return new AnthropicFoundry(foundryArgs) as unknown as Anthropic
  }
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX)) {
    const { AnthropicVertex } = await import('@anthropic-ai/vertex-sdk')
    return new AnthropicVertex(vertexArgs) as unknown as Anthropic
  }

  // 默认: 直接 API 调用
  return new Anthropic({
    apiKey: isClaudeAISubscriber() ? null : apiKey || getAnthropicApiKey(),
    authToken: isClaudeAISubscriber() ? getClaudeAIOAuthTokens()?.accessToken : undefined,
    ...ARGS,
  })
}
```

**关键设计点**:
- 动态导入 SDK (`await import(...)`) 避免未使用的 SDK 进入打包体积
- `as unknown as Anthropic` 类型断言统一返回类型 (Bedrock/Vertex SDK 类型略有不同)
- 自定义 `buildFetch()` 注入 `x-client-request-id` header 用于超时场景的日志关联
- Bedrock 支持 Bearer token 认证和 AWS STS 凭证刷新

### 2.2 核心查询引擎 — queryModel

`src/services/api/claude.ts` 的 `queryModel()` 是整个系统最核心的函数 (~700 行)，负责:

```typescript
// src/services/api/claude.ts:1017-1027
async function* queryModel(
  messages: Message[],
  systemPrompt: SystemPrompt,
  thinkingConfig: ThinkingConfig,
  tools: Tools,
  signal: AbortSignal,
  options: Options,
): AsyncGenerator<StreamEvent | AssistantMessage | SystemAPIErrorMessage, void> {
```

**关键流程**:

1. **Off-switch 检查** (L1028-1049): GrowthBook 动态配置可以关闭 Opus 模型
2. **Beta headers 组装** (L1071): `getMergedBetas()` 合并模型级别和功能级别的 beta
3. **工具搜索过滤** (L1120-1172): 仅发送已发现的延迟加载工具，减少 token 消耗
4. **工具 Schema 构建** (L1235-1246): 并行生成所有工具的 API schema
5. **消息标准化** (L1266): `normalizeMessagesForAPI()` 处理历史消息格式
6. **Beta Header 锁存** (L1405-1456): 一旦发送某个 beta header，整个会话持续发送避免缓存键变化
7. **paramsFromContext 闭包** (L1538-1729): 构建完整的 API 请求参数
8. **流式处理** (L1940-2200): 逐块累积 content blocks，处理 thinking/text/tool_use/connector_text

**Options 类型 — 控制 API 调用的完整参数**:

```typescript
// src/services/api/claude.ts:676-707
export type Options = {
  getToolPermissionContext: () => Promise<ToolPermissionContext>
  model: string
  toolChoice?: BetaToolChoiceTool | BetaToolChoiceAuto
  isNonInteractiveSession: boolean
  fallbackModel?: string            // 529 过载时的降级模型
  querySource: QuerySource           // 调用来源标记 (repl_main_thread, agent:*, compact...)
  mcpTools: Tools                    // MCP 服务器提供的工具
  effortValue?: EffortValue          // 推理努力级别
  fastMode?: boolean                 // 快速模式
  taskBudget?: { total: number; remaining?: number }  // API 端 token 预算
  advisorModel?: string              // Advisor 双模型
  enablePromptCaching?: boolean
  skipCacheWrite?: boolean
  outputFormat?: BetaJSONOutputFormat // 结构化输出
  // ... 更多参数
}
```

### 2.3 流式处理架构

流式处理使用原始 SSE 流而非 SDK 的 `BetaMessageStream`，避免 O(n^2) 的部分 JSON 解析:

```typescript
// src/services/api/claude.ts:1818-1836
// Use raw stream instead of BetaMessageStream to avoid O(n² partial JSON parsing
const result = await anthropic.beta.messages
  .create(
    { ...params, stream: true },
    { signal, ...(clientRequestId && { headers: { ... } }) },
  )
  .withResponse()

stream = result.data  // Stream<BetaRawMessageStreamEvent>
```

**流式事件处理状态机**:

```
message_start → 初始化 usage, 捕获 research metadata
content_block_start → 创建新的 content block (text/thinking/tool_use/server_tool_use)
content_block_delta → 增量拼接 (text_delta/input_json_delta/thinking_delta/signature_delta)
content_block_stop → 构建 AssistantMessage, yield 出去
message_delta → 更新 usage, stop_reason
message_stop → 完成
```

**流式空闲超时看门狗** (L1870-1928):
- 可选启用 (`CLAUDE_ENABLE_STREAM_WATCHDOG`)
- 默认 90s 无数据即 abort 流，防止静默断连
- 45s 时发出警告日志

### 2.4 Prompt 缓存策略

三层缓存控制:

```typescript
// src/services/api/claude.ts:358-374
export function getCacheControl({ scope, querySource } = {}) {
  return {
    type: 'ephemeral',
    ...(should1hCacheTTL(querySource) && { ttl: '1h' }),
    ...(scope === 'global' && { scope }),
  }
}
```

- **Ephemeral (默认 5min)**: 标准缓存
- **1h TTL**: 付费用户 + GrowthBook allowlist 匹配的 querySource
- **Global scope**: 系统提示在所有用户间共享缓存 (避免 MCP 工具变化导致失效)

**缓存键稳定性机制**:
- Beta header 锁存: 一旦发送 `afk_mode` / `fast_mode` / `cache_editing`，整个会话不变
- 思维清除锁存: 超过 1h cache TTL 才触发 thinking clear
- MCP 工具变化时自动切换 global→none 缓存策略

### 2.5 重试策略引擎

`src/services/api/withRetry.ts` 实现了精细的重试逻辑:

```typescript
// src/services/api/withRetry.ts:170-178
export async function* withRetry<T>(
  getClient: () => Promise<Anthropic>,
  operation: (client: Anthropic, attempt: number, context: RetryContext) => Promise<T>,
  options: RetryOptions,
): AsyncGenerator<SystemAPIErrorMessage, T> {
```

**重试决策矩阵**:

| 错误类型 | 处理策略 |
|---------|---------|
| 429 Rate Limit | Fast mode: 短 retry-after 等待; 长 retry-after 降速冷却 |
| 529 Overloaded | 最多 3 次，之后降级到 fallbackModel (Sonnet) |
| 401 Unauthorized | 刷新 OAuth token 后重试 |
| 403 Token Revoked | 调用 `handleOAuth401Error()` 强制刷新 |
| ECONNRESET/EPIPE | 禁用 keep-alive 后重试 |
| Context overflow (400) | 减少 max_tokens 后重试 |
| 非前台查询 529 | 立即放弃 (减少后端放大) |

**前台查询源白名单**:
```typescript
// src/services/api/withRetry.ts:62-82
const FOREGROUND_529_RETRY_SOURCES = new Set([
  'repl_main_thread', 'sdk', 'agent:custom', 'agent:default',
  'compact', 'hook_agent', 'auto_mode', ...
])
```
非白名单的查询 (summaries, titles, classifiers) 遇到 529 直接放弃，避免级联放大。

**持久重试模式** (CLAUDE_CODE_UNATTENDED_RETRY):
- 无限重试 429/529，最大退避 5 分钟
- 每 30 秒发送心跳 keep-alive
- 6 小时 reset-after 上限

### 2.6 错误分类系统

`src/services/api/errors.ts` 提供结构化的错误分类:

```typescript
// src/services/api/errors.ts:85-96
export function parsePromptTooLongTokenCounts(rawMessage: string) {
  const match = rawMessage.match(
    /prompt is too long[^0-9]*(\d+)\s*tokens?\s*>\s*(\d+)/i,
  )
  return {
    actualTokens: match ? parseInt(match[1]!, 10) : undefined,
    limitTokens: match ? parseInt(match[2]!, 10) : undefined,
  }
}
```

**错误分类层次**:
1. `isPromptTooLongMessage` → 触发自动压缩
2. `isMediaSizeError` → 剥离图片后重试
3. `getPromptTooLongTokenGap` → 计算需要释放的 token 数量
4. `startsWithApiErrorPrefix` → API 错误 vs 用户消息

### 2.7 非流式降级

当流式请求失败时，自动降级到非流式:

```typescript
// src/services/api/claude.ts:818-917
export async function* executeNonStreamingRequest(
  clientOptions, retryOptions, paramsFromContext, onAttempt, captureRequest,
): AsyncGenerator<SystemAPIErrorMessage, BetaMessage> {
  const fallbackTimeoutMs = getNonstreamingFallbackTimeoutMs()
  // 远程会话 120s (避免容器超时), 本地 300s
}
```

---

## 3. MCP 集成系统

### 3.1 配置类型体系

`src/services/mcp/types.ts` 使用 Zod schema 定义 MCP 服务器配置:

```typescript
// src/services/mcp/types.ts:9-27
export const ConfigScopeSchema = lazySchema(() =>
  z.enum(['local', 'user', 'project', 'dynamic', 'enterprise', 'claudeai', 'managed'])
)

export const TransportSchema = lazySchema(() =>
  z.enum(['stdio', 'sse', 'sse-ide', 'http', 'ws', 'sdk'])
)
```

**传输类型**:
- `stdio`: 子进程通过 stdin/stdout 通信 (最常见)
- `sse`: Server-Sent Events 远程连接
- `http`: Streamable HTTP (MCP 2025 规范)
- `ws`: WebSocket (IDE 扩展)
- `sdk`: 进程内 SDK 控制传输

### 3.2 MCP 客户端核心

`src/services/mcp/client.ts` (~116K) 管理 MCP 服务器的完整生命周期:

**错误类型层次**:
```typescript
// src/services/mcp/client.ts:152-186
export class McpAuthError extends Error { ... }       // OAuth 认证失败
class McpSessionExpiredError extends Error { ... }     // 会话过期 (404 + -32001)
export class McpToolCallError extends TelemetrySafeError { ... }  // 工具调用返回 isError: true
```

**传输创建工厂**:
- `StdioClientTransport` → 本地子进程
- `SSEClientTransport` → SSE 远程
- `StreamableHTTPClientTransport` → HTTP Streamable
- `WebSocketTransport` → WebSocket
- `SdkControlClientTransport` → 进程内

**工具发现 → Tool 对象转换**:
```
MCP ListTools → MCPTool 实例化 → Tool[] 返回给 queryModel
MCP ListResources → ServerResource[] → ListMcpResourcesTool / ReadMcpResourceTool
```

### 3.3 MCP OAuth 认证

`src/services/mcp/auth.ts` (~86K) 实现完整的 OAuth 2.0 + XAA 跨应用访问:

```typescript
// src/services/mcp/auth.ts:198-210 — 自定义 fetch 带超时
function createAuthFetch(): FetchLike {
  return async (url, init?) => {
    const timeoutSignal = AbortSignal.timeout(AUTH_REQUEST_TIMEOUT_MS) // 30s
    // ...
  }
}
```

**OAuth 错误规范化** (处理非标准服务器如 Slack):
```typescript
// src/services/mcp/auth.ts:147-151
const NONSTANDARD_INVALID_GRANT_ALIASES = new Set([
  'invalid_refresh_token',
  'expired_refresh_token',
  'token_expired',
])
```
将非标准错误码映射为 RFC 6749 的 `invalid_grant`，统一刷新/失效逻辑。

**ClaudeAuthProvider**:
- 实现 `OAuthClientProvider` 接口
- Token 存储在系统密钥链 (macOS Keychain / Windows Credential Manager)
- 文件锁 (`lockfile.ts`) 防止并发 OAuth 刷新
- XAA (Cross-App Access): IdP → MCP server 的 token 交换

### 3.4 MCPConnectionManager (React)

```typescript
// src/services/mcp/MCPConnectionManager.tsx
export function MCPConnectionManager({ children, dynamicMcpConfig, isStrictMcpConfig }) {
  const { reconnectMcpServer, toggleMcpServer } = useManageMCPConnections(
    dynamicMcpConfig, isStrictMcpConfig
  )
  return (
    <MCPConnectionContext.Provider value={{ reconnectMcpServer, toggleMcpServer }}>
      {children}
    </MCPConnectionContext.Provider>
  )
}
```
React 上下文提供热重连和开关功能，使用 React Compiler 优化 (`_c(6)` memo 插槽)。

---

## 4. 上下文压缩与 Token 管理

### 4.1 自动压缩触发

`src/services/compact/autoCompact.ts` 控制何时触发压缩:

```typescript
// src/services/compact/autoCompact.ts:63-66
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000
export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
export const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000
export const MANUAL_COMPACT_BUFFER_TOKENS = 3_000
```

**阈值计算**:
```
有效上下文窗口 = getContextWindowForModel(model) - max(maxOutputTokens, 20_000)
自动压缩阈值 = 有效上下文窗口 - 13_000
警告阈值 = 阈值 - 20_000
错误阈值 = 阈值 - 20_000
阻塞限制 = 有效上下文窗口 - 3_000
```

**电路断路器**:
```typescript
// src/services/compact/autoCompact.ts:70-71
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```
连续失败 3 次后停止尝试，避免无限循环浪费 API 调用。

### 4.2 压缩策略管道

`autoCompactIfNeeded()` 的决策流程:

```
1. 检查 DISABLE_COMPACT 环境变量
2. 检查电路断路器 (连续失败次数)
3. shouldAutoCompact() — token 计数 vs 阈值
4. 优先尝试 Session Memory Compaction (更精细)
5. 降级到 compactConversation() (全量摘要)
```

### 4.3 全量压缩算法

`src/services/compact/compact.ts` 的 `compactConversation()`:

1. **消息预处理**: `stripImagesFromMessages()` 移除图片/文档 (替换为 `[image]` 标记)
2. **分组**: `groupMessagesByApiRound()` 按 API 轮次分组
3. **摘要生成**: 使用 forked agent (共享 prompt cache) 生成对话摘要
4. **后处理**: 恢复近期文件状态、重新注入 skills/attachments

**CompactionResult 结构**:
```typescript
// src/services/compact/compact.ts:299-310
export interface CompactionResult {
  boundaryMarker: SystemMessage      // 压缩边界标记
  summaryMessages: UserMessage[]     // 摘要消息
  attachments: AttachmentMessage[]   // 重新注入的附件
  hookResults: HookResultMessage[]   // 钩子结果
  messagesToKeep?: Message[]         // 保留的尾部消息
  preCompactTokenCount?: number      // 压缩前 token 数
  postCompactTokenCount?: number     // 压缩后 token 数
}
```

### 4.4 微压缩 (MicroCompact)

`src/services/compact/microCompact.ts` 在不触发全量压缩的前提下减少 token:

**可压缩的工具结果**:
```typescript
const COMPACTABLE_TOOLS = new Set([
  FILE_READ_TOOL_NAME, ...SHELL_TOOL_NAMES, GREP_TOOL_NAME,
  GLOB_TOOL_NAME, WEB_SEARCH_TOOL_NAME, WEB_FETCH_TOOL_NAME,
  FILE_EDIT_TOOL_NAME, FILE_WRITE_TOOL_NAME,
])
```

**时间衰减策略** (`timeBasedMCConfig.ts`):
- 旧的工具结果 → `[Old tool result content cleared]`
- 保留最近的窗口 (默认最后 40K tokens)

### 4.5 API 端上下文管理

`src/services/compact/apiMicrocompact.ts` 利用 API 端的 `context_management` 功能:

```typescript
// src/services/compact/apiMicrocompact.ts:35-57
export type ContextEditStrategy =
  | {
      type: 'clear_tool_uses_20250919'
      trigger?: { type: 'input_tokens'; value: number }
      keep?: { type: 'tool_uses'; value: number }
      clear_tool_inputs?: boolean | string[]
      exclude_tools?: string[]
    }
  | {
      type: 'clear_thinking_20251015'
      keep: { type: 'thinking_turns'; value: number } | 'all'
    }
```

服务端自动清除旧工具结果和思维块，减少客户端压缩负担。

### 4.6 Token 估算算法

`src/services/tokenEstimation.ts` 提供多层 token 计数:

**层级 1: 粗略估算** (无 API 调用):
```typescript
// src/services/tokenEstimation.ts:203-208
export function roughTokenCountEstimation(
  content: string,
  bytesPerToken: number = 4,
): number {
  return Math.round(content.length / bytesPerToken)
}
```

**文件类型感知** (JSON 更密集):
```typescript
// src/services/tokenEstimation.ts:215-224
export function bytesPerTokenForFileType(fileExtension: string): number {
  switch (fileExtension) {
    case 'json': case 'jsonl': case 'jsonc':
      return 2  // JSON 单字符 token 多
    default:
      return 4  // 通用比率
  }
}
```

**层级 2: API 精确计数**:
```typescript
// src/services/tokenEstimation.ts:124-200
export async function countMessagesTokensWithAPI(
  messages, tools,
): Promise<number | null> {
  return withTokenCountVCR(messages, tools, async () => {
    // 使用 anthropic.beta.messages.countTokens()
    // Bedrock: 使用 CountTokensCommand
  })
}
```

**层级 3: Haiku 降级计数** (精确但更便宜):
```typescript
// src/services/tokenEstimation.ts:251-325
export async function countTokensViaHaikuFallback(messages, tools) {
  // 使用 Haiku 模型的 usage.input_tokens 作为代理
  // Vertex global 端点无 Haiku → 降级到 Sonnet
  // Bedrock + thinking blocks → 降级到 Sonnet (Haiku 3.5 不支持 thinking)
}
```

---

## 5. 认证与 OAuth 系统

### 5.1 OAuth 2.0 PKCE 流程

`src/services/oauth/client.ts`:

```typescript
// src/services/oauth/client.ts:46-100
export function buildAuthUrl({
  codeChallenge, state, port, isManual,
  loginWithClaudeAi, inferenceOnly, orgUUID, loginHint, loginMethod,
}) {
  const authUrl = new URL(
    loginWithClaudeAi
      ? getOauthConfig().CLAUDE_AI_AUTHORIZE_URL
      : getOauthConfig().CONSOLE_AUTHORIZE_URL
  )
  authUrl.searchParams.append('code_challenge', codeChallenge)
  authUrl.searchParams.append('code_challenge_method', 'S256')
  authUrl.searchParams.append('scope', scopesToUse.join(' '))
}
```

**双认证路径**:
- Console 用户: API key (`ANTHROPIC_API_KEY`)
- Claude.ai 订阅者: OAuth 2.0 + PKCE (本地 HTTP 回调服务器)

**Scope 体系**:
- `claude_ai:inference` — 推理 API 访问
- `claude_ai:profile` — 用量查询
- `ALL_OAUTH_SCOPES` — 完整权限

### 5.2 Token 生命周期管理

```
登录 → buildAuthUrl() → 浏览器授权 → 回调 → exchangeCodeForToken()
     → saveApiKey() (持久化)
     → checkAndRefreshOAuthTokenIfNeeded() (每次 API 调用前检查)
     → isOAuthTokenExpired() → refreshToken → handleOAuth401Error()
```

---

## 6. 分析与特性开关系统

### 6.1 GrowthBook 集成

`src/services/analytics/growthbook.ts`:

```typescript
// src/services/analytics/growthbook.ts:31-47
export type GrowthBookUserAttributes = {
  id: string
  sessionId: string
  deviceID: string
  platform: 'win32' | 'darwin' | 'linux'
  apiBaseUrlHost?: string
  organizationUUID?: string
  accountUUID?: string
  userType?: string           // 'ant' | 'external'
  subscriptionType?: string
  rateLimitTier?: string
  firstTokenTime?: number
  appVersion?: string
}
```

**功能**:
- `getFeatureValue_CACHED_MAY_BE_STALE()` — 非阻塞读取 (使用缓存值)
- `getDynamicConfig_BLOCKS_ON_INIT()` — 阻塞等待初始化完成
- Remote eval 支持 (服务端评估)
- 实验曝光日志 (exposure logging)
- 刷新监听器 (`onGrowthBookRefresh()`)

**反模式防护**:
```typescript
// 使用 CACHED_MAY_BE_STALE 时可能读到旧值
// 关键路径使用 BLOCKS_ON_INIT 确保最新
const config = await getDynamicConfig_BLOCKS_ON_INIT('tengu-off-switch', { activated: false })
```

### 6.2 事件分析管道

`src/services/analytics/index.ts`:

```typescript
// src/services/analytics/index.ts:95-100
export function attachAnalyticsSink(newSink: AnalyticsSink): void {
  if (sink !== null) return  // 幂等
  sink = newSink
  // 异步排空预初始化事件队列
  queueMicrotask(() => { /* drain eventQueue */ })
}
```

**PII 保护机制**:
- `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` — 类型标记强制开发者验证
- `_PROTO_*` 前缀键 → 仅发送到 PII 标记的 BQ 列
- `stripProtoFields()` — 在 Datadog 等通用存储前移除

**Sink 架构**:
```
logEvent() → eventQueue (pre-init) / sink.logEvent() (post-init)
           ↓
     analytics/sink.ts → Datadog (DD)
                       → firstPartyEventLogger (1P BQ)
```

---

## 7. LSP 集成

### 7.1 LSP Server Manager

`src/services/lsp/LSPServerManager.ts`:

```typescript
// src/services/lsp/LSPServerManager.ts:16-43
export type LSPServerManager = {
  initialize(): Promise<void>
  shutdown(): Promise<void>
  getServerForFile(filePath: string): LSPServerInstance | undefined
  ensureServerStarted(filePath: string): Promise<LSPServerInstance | undefined>
  sendRequest<T>(filePath: string, method: string, params: unknown): Promise<T | undefined>
  openFile(filePath: string, content: string): Promise<void>
  changeFile(filePath: string, content: string): Promise<void>
  saveFile(filePath: string): Promise<void>
  closeFile(filePath: string): Promise<void>
  isFileOpen(filePath: string): boolean
}
```

**设计**: 工厂函数 + 闭包封装状态 (避免 class):

```typescript
// src/services/lsp/LSPServerManager.ts:59-65
export function createLSPServerManager(): LSPServerManager {
  const servers: Map<string, LSPServerInstance> = new Map()
  const extensionMap: Map<string, string[]> = new Map()
  const openedFiles: Map<string, string> = new Map()
  // ... 闭包方法
}
```

**文件路由**: 基于文件扩展名将请求路由到正确的语言服务器。

### 7.2 全局单例管理

`src/services/lsp/manager.ts`:
- 使用代际计数器 (`initializationGeneration`) 防止过期的初始化 Promise 更新状态
- `initializationState: 'not-started' | 'pending' | 'success' | 'failed'`
- 被动反馈 (`passiveFeedback.ts`) 注册 LSP 通知处理器

---

## 8. 插件系统

### 8.1 后台安装管理器

`src/services/plugins/PluginInstallationManager.ts`:

```typescript
// src/services/plugins/PluginInstallationManager.ts:60-99
export async function performBackgroundPluginInstallations(setAppState) {
  const declared = getDeclaredMarketplaces()
  const materialized = await loadKnownMarketplacesConfig()
  const diff = diffMarketplaces(declared, materialized)

  // 初始化 pending 状态
  setAppState(prev => ({
    ...prev,
    plugins: {
      ...prev.plugins,
      installationStatus: {
        marketplaces: pendingNames.map(name => ({ name, status: 'pending' })),
        plugins: [],
      },
    },
  }))

  // 后台 reconcile
  await reconcileMarketplaces(declared, {
    onProgress: (name, status) => updateMarketplaceStatus(setAppState, name, status),
  })
}
```

**协调器模式**:
1. `getDeclaredMarketplaces()` → 读取 settings 中声明的市场
2. `loadKnownMarketplacesConfig()` → 读取本地已安装状态
3. `diffMarketplaces()` → 计算需要新安装/更新的
4. `reconcileMarketplaces()` → 克隆/更新仓库
5. `refreshActivePlugins()` → 新安装后立即刷新

---

## 9. 企业级服务

### 9.1 策略限制 (Policy Limits)

`src/services/policyLimits/index.ts`:

```typescript
// src/services/policyLimits/index.ts:56-59
const CACHE_FILENAME = 'policy-limits.json'
const FETCH_TIMEOUT_MS = 10000   // 10 秒
const POLLING_INTERVAL_MS = 60 * 60 * 1000  // 1 小时轮询
```

**Fail-Open 模式**:
- API 不可用时继续运行，不施加限制
- ETag 缓存 + 后台轮询减少网络流量
- 本地文件缓存 (`~/.claude/policy-limits.json`) 支持离线启动

**合格用户**: Console API key 用户 + OAuth Team/Enterprise 订阅者

### 9.2 远程管理设置 (Remote Managed Settings)

`src/services/remoteManagedSettings/index.ts`:

与 Policy Limits 完全相同的架构模式:
- Checksum 校验减少传输
- 1 小时后台轮询
- 30 秒加载超时防止死锁
- 安全检查 (`securityCheck.tsx`) 验证远程设置的安全性

```typescript
// src/services/remoteManagedSettings/index.ts:51-56
const SETTINGS_TIMEOUT_MS = 10000
const DEFAULT_MAX_RETRIES = 5
const POLLING_INTERVAL_MS = 60 * 60 * 1000
```

---

## 10. 团队记忆同步

### 10.1 同步架构

`src/services/teamMemorySync/index.ts`:

```
API 路径:
  GET  /api/claude_code/team_memory?repo={owner/repo}             → 完整数据
  GET  /api/claude_code/team_memory?repo={owner/repo}&view=hashes → 仅哈希
  PUT  /api/claude_code/team_memory?repo={owner/repo}             → 上传 (upsert)
```

**同步语义**:
- **Pull**: 服务端覆盖本地 (server wins per-key)
- **Push**: 仅上传哈希不同的条目 (增量上传)
- **删除不传播**: 删除本地文件不会删除服务端数据

**安全保护**:
```typescript
// src/services/teamMemorySync/index.ts:72-89
const MAX_FILE_SIZE_BYTES = 250_000    // 单文件上限
const MAX_PUT_BODY_BYTES = 200_000     // 网关限制 (超大 PUT 拆分)
const MAX_RETRIES = 3
const MAX_CONFLICT_RETRIES = 2
```

- `secretScanner.ts` — 推送前扫描密钥
- `teamMemSecretGuard.ts` — 阻止包含密钥的文件
- 超大 PUT 自动拆分为多个顺序请求

### 10.2 状态管理

```typescript
// src/services/teamMemorySync/index.ts:100
export type SyncState = {
  // 每个会话创建一个，传入所有同步函数
  // 包含 ETag 跟踪、watcher 抑制等
}
```

使用显式状态对象而非模块级变量，确保测试隔离。

---

## 11. 自动记忆提取

### 11.1 触发时机与机制

`src/services/extractMemories/extractMemories.ts`:

```
触发: 每次完整查询循环结束 (最终响应无工具调用)
     → handleStopHooks (stopHooks.ts)
     → extractMemories
```

**实现模式**:
- 使用 `runForkedAgent()` — 父进程的完美分叉，共享 prompt cache
- 状态通过闭包封装 (`initExtractMemories()` 返回闭包)
- 允许的工具限定: Bash, Read, Write, Edit, Glob, Grep, REPL

**记忆目录**: `~/.claude/projects/<path>/memory/`

---

## 12. 可复用的服务模式

### 模式 1: AsyncGenerator 流式管道

```typescript
async function* serviceOperation(): AsyncGenerator<StatusEvent, FinalResult> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await operation()
    } catch (error) {
      yield { type: 'status', message: `Retry ${attempt}...` }
    }
  }
}
```
用于: API 调用、MCP 操作、压缩。好处: 调用者可以在等待最终结果时处理中间状态。

### 模式 2: Fail-Open 后台服务

```typescript
// 初始化加载 Promise + 超时
let loadingPromise: Promise<void> | null = null
function initializeLoadingPromise() {
  loadingPromise = new Promise(resolve => {
    loadingResolve = resolve
    setTimeout(() => { resolve(); loadingResolve = null }, 30_000)
  })
}

// 后台轮询 + ETag 缓存
let pollingId: ReturnType<typeof setInterval> | null = null
function startPolling() {
  pollingId = setInterval(fetchWithETag, 60 * 60 * 1000)
  registerCleanup(() => clearInterval(pollingId))
}
```
用于: policyLimits, remoteManagedSettings。特点: 30s 超时防死锁，1h 轮询，文件缓存。

### 模式 3: 闭包工厂 (替代 class)

```typescript
export function createManager(): Manager {
  const state: Map<string, Instance> = new Map()
  return {
    initialize: async () => { /* 用 state */ },
    getForFile: (path) => state.get(path),
    shutdown: async () => { state.clear() },
  }
}
```
用于: LSPServerManager, cachedMCState。好处: 自然封装、无 `this` 绑定问题。

### 模式 4: 会话稳定锁存

```typescript
let headerLatched = false
function ensureHeaderLatched(condition: boolean) {
  if (!headerLatched && condition) {
    headerLatched = true
    setHeaderLatched(true)
  }
  return headerLatched
}
```
用于: fast_mode, afk_mode, cache_editing beta headers。目的: 避免 prompt cache 键在会话中途变化。

### 模式 5: 编译时功能门控

```typescript
import { feature } from 'bun:bundle'

// 外部构建完全删除此代码块
if (feature('CACHED_MICROCOMPACT')) {
  const { isCachedMicrocompactEnabled } = await import('./cachedMicrocompact.js')
  // ...
}

// 条件 require (非顶层 import，避免初始化循环)
const module = feature('KAIROS')
  ? require('../sessionTranscript/sessionTranscript.js')
  : null
```
用于: 内部功能隔离。好处: 死代码消除减少外部构建体积。

### 模式 6: 电路断路器

```typescript
const MAX_FAILURES = 3
if (tracking.consecutiveFailures >= MAX_FAILURES) {
  return { wasCompacted: false }  // 停止尝试
}
// 失败时递增
return { consecutiveFailures: prevFailures + 1 }
// 成功时重置
return { consecutiveFailures: 0 }
```
用于: autoCompact, retry 逻辑。防止无限循环浪费资源。

### 模式 7: PII 安全的遥测标记

```typescript
// 编译时强制验证 — 类型标记
type Safe = AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS // = never
logEvent('event', {
  model: options.model as Safe,  // 开发者必须显式断言这不是敏感数据
})
// _PROTO_ 前缀自动路由到 PII 专用存储
logEvent('event', {
  _PROTO_user_email: email as AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED,
})
```

---

## 关键数据流图

```
用户输入
  │
  ▼
queryModel() ─── paramsFromContext() ─── getAnthropicClient()
  │                     │                      │
  │                     ▼                      ▼
  │              系统提示构建           Anthropic / Bedrock / Vertex
  │              工具 Schema
  │              Beta headers
  │              缓存控制
  │
  ▼
withRetry() ─── 重试/降级/快速模式冷却
  │
  ▼
SSE Stream ─── 事件累积 ─── AssistantMessage
  │                              │
  ▼                              ▼
StreamEvent yield         Tool execution
  │                              │
  ▼                              ▼
UI 渲染                   Tool results → 下一轮 queryModel()
                                │
                           autoCompactIfNeeded() ─── 超限 → compactConversation()
```
