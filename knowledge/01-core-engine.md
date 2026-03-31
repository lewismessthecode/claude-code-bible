# 01 - 核心引擎 (Core Engine)

> Claude Code 的核心引擎层：CLI 入口、查询循环、工具抽象、命令系统、成本追踪和上下文收集。

---

## 目录

1. [总体架构概览](#1-总体架构概览)
2. [启动优化与并行预取 (main.tsx)](#2-启动优化与并行预取)
3. [CLI 参数解析与 Commander.js 模式](#3-cli-参数解析与-commanderjs-模式)
4. [QueryEngine — 会话生命周期管理](#4-queryengine--会话生命周期管理)
5. [query() — 核心查询循环与状态机](#5-query--核心查询循环与状态机)
6. [Tool 接口与抽象体系](#6-tool-接口与抽象体系)
7. [工具注册与条件加载 (tools.ts)](#7-工具注册与条件加载)
8. [命令注册系统 (commands.ts)](#8-命令注册系统)
9. [成本与 Token 追踪](#9-成本与-token-追踪)
10. [上下文收集系统 (context.ts)](#10-上下文收集系统)
11. [Feature Flag 与死代码消除](#11-feature-flag-与死代码消除)
12. [流式处理与错误恢复策略](#12-流式处理与错误恢复策略)
13. [消息规范化与 API 协议](#13-消息规范化与-api-协议)
14. [可复用的工程模式总结](#14-可复用的工程模式总结)

---

## 1. 总体架构概览

Claude Code 核心引擎的数据流如下：

```
用户输入
  │
  ▼
main.tsx (CLI入口, Commander.js解析, 并行预取)
  │
  ▼
QueryEngine (会话状态管理, 多轮对话持久化)
  │
  ▼
query() (核心循环: API调用 → 流式响应 → 工具执行 → 自动压缩 → 递归)
  │    ├── callModel() — 流式 API 调用
  │    ├── runTools() / StreamingToolExecutor — 工具并发执行
  │    ├── autocompact / reactiveCompact — Token 压缩
  │    └── stopHooks / tokenBudget — 终止条件
  │
  ▼
Tool 接口 (validateInput → checkPermissions → call → mapResult)
  │
  ▼
tools.ts (工具注册表, 特性门控, deny-rule 过滤)
```

**关键设计原则：**
- 启动速度至上：模块评估期间就并行发起子进程（MDM、keychain），不等待 imports 完成
- 不可变消息流：API-bound 的消息对象不可变（保护 prompt cache），仅在 yield 给 SDK 时克隆
- Generator 驱动：`query()` 和 `QueryEngine.submitMessage()` 都是 `AsyncGenerator`，实现背压感知的流式处理
- 特性门控消除：`feature('FLAG')` 在编译期由 `bun:bundle` 静态求值，未激活分支被 DCE 移除

---

## 2. 启动优化与并行预取

`main.tsx` 的前 20 行是整个项目最关键的性能优化区域。它利用了 ES module 的**评估顺序保证**：import 语句之间的副作用在下一个 import 前执行。

### 2.1 三阶段并行预取

```typescript
// 文件: src/main.tsx:1-20
// 阶段1: 启动计时（必须第一个执行）
import { profileCheckpoint } from './utils/startupProfiler.js';
profileCheckpoint('main_tsx_entry');

// 阶段2: 启动 MDM 子进程（plutil/reg query）— 与后续 ~135ms imports 并行
import { startMdmRawRead } from './utils/settings/mdm/rawRead.js';
startMdmRawRead();

// 阶段3: macOS keychain 预取（OAuth + legacy API key 并行读取）
// 避免 isRemoteManagedSettingsEligible() 中的串行 sync spawn (~65ms)
import { ensureKeychainPrefetchCompleted, startKeychainPrefetch }
  from './utils/secureStorage/keychainPrefetch.js';
startKeychainPrefetch();
```

**为什么这样做？** 普通写法会在 `init()` 中串行执行这些操作。而这里利用了模块评估的空闲窗口——后面还有 ~135ms 的 import 评估时间，子进程在这段时间里并行完成。

**可复用模式：** 在任何 CLI 应用中，如果有需要子进程/网络IO的启动逻辑，可以在 import 语句之间穿插执行。

### 2.2 延迟预取 (Deferred Prefetches)

```typescript
// 文件: src/main.tsx:388-431
export function startDeferredPrefetches(): void {
  // 跳过纯启动性能基准测试
  if (isEnvTruthy(process.env.CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER) || isBareMode()) {
    return;
  }

  // 进程级预取（用户还在打字时执行，首次 API 调用时消费）
  void initUser();
  void getUserContext();
  prefetchSystemContextIfSafe();  // 安全门控：仅在信任建立后预取 git 数据
  void getRelevantTips();

  // 云凭证预取
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK)) {
    void prefetchAwsCredentialsAndBedRockInfoIfSafe();
  }

  // 文件计数（用于 token 估算）
  void countFilesRoundedRg(getCwd(), AbortSignal.timeout(3000), []);

  // 分析和特性标志
  void initializeAnalyticsGates();
  void refreshModelCapabilities();
  void settingsChangeDetector.initialize();
}
```

**关键洞察：** 此函数在 REPL 首次渲染**之后**调用。这些预取不阻塞首屏，但利用了"用户打字"的空闲窗口来预热缓存。`void` 前缀表示 fire-and-forget，不等待结果。

### 2.3 安全优先的预取

```typescript
// 文件: src/main.tsx:360-380
function prefetchSystemContextIfSafe(): void {
  const isNonInteractiveSession = getIsNonInteractiveSession();

  // 非交互模式：信任是隐式的
  if (isNonInteractiveSession) {
    void getSystemContext();
    return;
  }

  // 交互模式：仅在信任已建立时预取
  // Git 命令可通过 hooks 和 config 执行任意代码
  const hasTrust = checkHasTrustDialogAccepted();
  if (hasTrust) {
    void getSystemContext();
  }
  // 否则不预取——等待信任建立
}
```

**安全模式：** git 命令可以通过 `core.fsmonitor` 和 `diff.external` 执行任意代码。在不受信任的目录中预取 git status 等于允许攻击者执行代码。

---

## 3. CLI 参数解析与 Commander.js 模式

### 3.1 preAction Hook 模式

```typescript
// 文件: src/main.tsx:907-967
program.hook('preAction', async thisCommand => {
  // 等待模块评估期间启动的异步子进程
  // 几乎免费——子进程在 ~135ms imports 期间已完成
  await Promise.all([
    ensureMdmSettingsLoaded(),
    ensureKeychainPrefetchCompleted()
  ]);

  await init();

  // 附加日志 sink（子命令不调用 setup()，需要在这里初始化）
  const { initSinks } = await import('./utils/sinks.js');
  initSinks();

  // 处理跨子命令的 --plugin-dir 选项
  const pluginDir = thisCommand.getOptionValue('pluginDir');
  if (Array.isArray(pluginDir) && pluginDir.length > 0) {
    setInlinePlugins(pluginDir);
    clearPluginCache('preAction: --plugin-dir inline plugins');
  }

  runMigrations();

  // 非阻塞远程设置加载
  void loadRemoteManagedSettings();
  void loadPolicyLimits();
});
```

**为什么用 preAction 而非直接初始化？** 因为 `--help` 不需要初始化。preAction hook 仅在实际执行命令时运行，避免了显示帮助信息时的不必要开销。

### 3.2 早期 argv 处理模式

主函数中有大量在 Commander 解析之前的 argv 预处理：

```typescript
// 文件: src/main.tsx:612-642
// cc:// URL 处理——在 Commander 解析前重写 argv
if (feature('DIRECT_CONNECT')) {
  const rawCliArgs = process.argv.slice(2);
  const ccIdx = rawCliArgs.findIndex(
    a => a.startsWith('cc://') || a.startsWith('cc+unix://')
  );
  if (ccIdx !== -1 && _pendingConnect) {
    const ccUrl = rawCliArgs[ccIdx]!;
    const { parseConnectUrl } = await import('./server/parseConnectUrl.js');
    const parsed = parseConnectUrl(ccUrl);

    if (rawCliArgs.includes('-p') || rawCliArgs.includes('--print')) {
      // Headless: 重写为内部 `open` 子命令
      process.argv = [process.argv[0]!, process.argv[1]!, 'open', ccUrl, ...stripped];
    } else {
      // Interactive: 剥离 cc:// URL，让主命令处理
      _pendingConnect.url = parsed.serverUrl;
      _pendingConnect.authToken = parsed.authToken;
      process.argv = [process.argv[0]!, process.argv[1]!, ...stripped];
    }
  }
}
```

**模式：** 对于需要在常规命令框架之外处理的特殊协议（URL scheme、SSH 等），先预处理 argv 再交给 Commander。

### 3.3 迁移版本控制

```typescript
// 文件: src/main.tsx:325-352
const CURRENT_MIGRATION_VERSION = 11;

function runMigrations(): void {
  if (getGlobalConfig().migrationVersion !== CURRENT_MIGRATION_VERSION) {
    migrateAutoUpdatesToSettings();
    migrateBypassPermissionsAcceptedToSettings();
    migrateSonnet45ToSonnet46();
    migrateOpusToOpus1m();
    // ... 更多迁移

    saveGlobalConfig(prev =>
      prev.migrationVersion === CURRENT_MIGRATION_VERSION
        ? prev  // 防止并发写入
        : { ...prev, migrationVersion: CURRENT_MIGRATION_VERSION }
    );
  }
  // 异步迁移——fire-and-forget
  migrateChangelogFromConfig().catch(() => {});
}
```

**模式：** 版本号 + 幂等迁移函数。同步迁移门控，异步迁移 fire-and-forget。

---

## 4. QueryEngine — 会话生命周期管理

### 4.1 配置与状态

```typescript
// 文件: src/QueryEngine.ts:130-173
export type QueryEngineConfig = {
  cwd: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  agents: AgentDefinition[]
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  initialMessages?: Message[]
  readFileCache: FileStateCache
  customSystemPrompt?: string
  appendSystemPrompt?: string
  userSpecifiedModel?: string
  fallbackModel?: string
  thinkingConfig?: ThinkingConfig
  maxTurns?: number
  maxBudgetUsd?: number
  taskBudget?: { total: number }
  jsonSchema?: Record<string, unknown>  // 结构化输出 schema
  verbose?: boolean
  replayUserMessages?: boolean
  snipReplay?: (msg: Message, store: Message[]) =>
    { messages: Message[]; executed: boolean } | undefined
}
```

**一个 QueryEngine 对应一个会话。** 每次 `submitMessage()` 是同一会话中的一个新"轮次"。状态（消息历史、文件缓存、用量统计）在轮次间持久化。

### 4.2 submitMessage — 核心提交流程

```typescript
// 文件: src/QueryEngine.ts:209-213
async *submitMessage(
  prompt: string | ContentBlockParam[],
  options?: { uuid?: string; isMeta?: boolean },
): AsyncGenerator<SDKMessage, void, unknown> {
```

关键步骤：

1. **重置轮次状态**：清空 discoveredSkillNames
2. **获取系统提示**：`fetchSystemPromptParts()` — 包含工具描述、CLAUDE.md 等
3. **权限拦截代理**：包装 `canUseTool` 以追踪权限拒绝
4. **处理用户输入**：`processUserInput()` — 处理 `/slash` 命令、附件等
5. **持久化记录**：在 API 调用前先写入 transcript（防止进程被杀导致丢失）
6. **调用 query()**：进入核心循环
7. **SDK 消息转换**：将内部消息格式转换为 SDK 协议

### 4.3 权限拦截模式

```typescript
// 文件: src/QueryEngine.ts:244-271
const wrappedCanUseTool: CanUseToolFn = async (
  tool, input, toolUseContext, assistantMessage, toolUseID, forceDecision,
) => {
  const result = await canUseTool(
    tool, input, toolUseContext, assistantMessage, toolUseID, forceDecision,
  );

  // 追踪拒绝用于 SDK 报告
  if (result.behavior !== 'allow') {
    this.permissionDenials.push({
      tool_name: sdkCompatToolName(tool.name),
      tool_use_id: toolUseID,
      tool_input: input,
    });
  }
  return result;
};
```

**模式：** 装饰器/代理模式包装权限检查函数，在不修改原始逻辑的情况下收集拒绝事件。

### 4.4 Transcript 持久化策略

```typescript
// 文件: src/QueryEngine.ts:450-463
// 在 API 调用前先持久化用户消息
// 原因：如果进程被杀（如 cowork 中用户点击 Stop），
// transcript 中至少有用户消息，--resume 不会失败
if (persistSession && messagesFromUserInput.length > 0) {
  const transcriptPromise = recordTranscript(messages);
  if (isBareMode()) {
    void transcriptPromise;  // --bare: fire-and-forget (~4ms SSD, ~30ms disk contention)
  } else {
    await transcriptPromise;
    if (isEnvTruthy(process.env.CLAUDE_CODE_EAGER_FLUSH)) {
      await flushSessionStorage();
    }
  }
}
```

**模式：** 分级持久化——交互模式同步写入保证可恢复性，bare/脚本模式异步写入保证性能。

---

## 5. query() — 核心查询循环与状态机

### 5.1 循环结构

`query()` 是一个 `while(true)` 循环，通过 `state = next; continue` 实现状态转换。

```typescript
// 文件: src/query.ts:200-217
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined  // 上一次迭代为何 continue
}
```

### 5.2 迭代内流水线

每次循环迭代的执行顺序：

```
1. applyToolResultBudget    — 工具结果大小预算
2. snipCompact              — 历史截断压缩（feature: HISTORY_SNIP）
3. microcompact             — 微压缩（缓存编辑）
4. contextCollapse          — 上下文折叠（feature: CONTEXT_COLLAPSE）
5. autocompact              — 自动压缩（当 token 超阈值时）
6. calculateTokenWarningState — 阻塞限制检查
7. callModel (streaming)    — 流式 API 调用
8. StreamingToolExecutor    — 流式工具执行（与 streaming 并行）
9. stopHooks                — 停止钩子评估
10. tokenBudget check       — Token 预算检查
11. getAttachmentMessages   — 附件注入（内存、技能发现）
12. 递归：state = next; continue
```

### 5.3 流式工具执行

```typescript
// 文件: src/query.ts:561-568
const useStreamingToolExecution = config.gates.streamingToolExecution;
let streamingToolExecutor = useStreamingToolExecution
  ? new StreamingToolExecutor(
      toolUseContext.options.tools,
      canUseTool,
      toolUseContext,
    )
  : null;
```

当模型流式返回 tool_use block 时，`StreamingToolExecutor` 立即开始执行工具，不等待整个响应完成：

```typescript
// 文件: src/query.ts:838-862
if (streamingToolExecutor && !toolUseContext.abortController.signal.aborted) {
  for (const toolBlock of msgToolUseBlocks) {
    streamingToolExecutor.addTool(toolBlock, message);
  }
}

// 收集已完成的结果
for (const result of streamingToolExecutor.getCompletedResults()) {
  if (result.message) {
    yield result.message;
    toolResults.push(
      ...normalizeMessagesForAPI([result.message], toolUseContext.options.tools)
        .filter(_ => _.type === 'user'),
    );
  }
}
```

**性能优势：** 模型输出期间工具就已开始执行，总体轮次延迟降低。

### 5.4 自动压缩 (Auto-Compact)

```typescript
// 文件: src/query.ts:454-543
const { compactionResult, consecutiveFailures } = await deps.autocompact(
  messagesForQuery, toolUseContext,
  {
    systemPrompt, userContext, systemContext,
    toolUseContext,
    forkContextMessages: messagesForQuery,
  },
  querySource, tracking, snipTokensFreed,
);

if (compactionResult) {
  // 追踪 task_budget 消耗
  if (params.taskBudget) {
    const preCompactContext = finalContextTokensFromLastResponse(messagesForQuery);
    taskBudgetRemaining = Math.max(
      0,
      (taskBudgetRemaining ?? params.taskBudget.total) - preCompactContext,
    );
  }

  // 重置追踪状态
  tracking = {
    compacted: true,
    turnId: deps.uuid(),
    turnCounter: 0,
    consecutiveFailures: 0,
  };

  const postCompactMessages = buildPostCompactMessages(compactionResult);
  for (const message of postCompactMessages) {
    yield message;
  }
  messagesForQuery = postCompactMessages;
}
```

**三层压缩策略：**
1. **snipCompact** (HISTORY_SNIP) — 截断旧历史，最轻量
2. **microcompact** — 缓存编辑级别，精确删除已处理的工具结果
3. **autocompact** — 完整的对话摘要，使用 Haiku 模型生成摘要
4. **reactiveCompact** — 响应式压缩，当 API 返回 prompt-too-long 错误时触发

---

## 6. Tool 接口与抽象体系

### 6.1 完整的 Tool 接口

```typescript
// 文件: src/Tool.ts:362-590
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  readonly name: string
  aliases?: string[]           // 向后兼容的别名
  searchHint?: string          // ToolSearch 关键词匹配
  readonly inputSchema: Input  // Zod schema
  readonly inputJSONSchema?: ToolInputJSONSchema  // MCP 工具直接用 JSON Schema
  outputSchema?: z.ZodType<unknown>
  maxResultSizeChars: number   // 超过则持久化到磁盘

  // 核心生命周期
  call(args, context, canUseTool, parentMessage, onProgress?): Promise<ToolResult<Output>>
  description(input, options): Promise<string>
  isEnabled(): boolean
  isReadOnly(input): boolean
  isDestructive?(input): boolean
  isConcurrencySafe(input): boolean

  // 权限
  validateInput?(input, context): Promise<ValidationResult>
  checkPermissions(input, context): Promise<PermissionResult>
  preparePermissionMatcher?(input): Promise<(pattern: string) => boolean>

  // 中断行为
  interruptBehavior?(): 'cancel' | 'block'

  // UI 渲染
  prompt(options): Promise<string>
  userFacingName(input): string
  renderToolResultMessage?(content, progressMessages, options): React.ReactNode
  getActivityDescription?(input): string | null  // Spinner 显示文本
  isSearchOrReadCommand?(input): { isSearch: boolean; isRead: boolean; isList?: boolean }

  // 观察者相关
  backfillObservableInput?(input): void  // SDK/hook 可见的派生字段
  toAutoClassifierInput(input): unknown  // 安全分类器输入

  // 延迟加载
  readonly shouldDefer?: boolean    // ToolSearch 延迟加载
  readonly alwaysLoad?: boolean     // 始终加载（不延迟）

  // MCP 元数据
  mcpInfo?: { serverName: string; toolName: string }
}
```

### 6.2 ToolResult 与上下文修改

```typescript
// 文件: src/Tool.ts:321-336
export type ToolResult<T> = {
  data: T
  newMessages?: (UserMessage | AssistantMessage | AttachmentMessage | SystemMessage)[]
  // contextModifier 仅对非并发安全的工具有效
  contextModifier?: (context: ToolUseContext) => ToolUseContext
  /** MCP protocol metadata */
  mcpMeta?: {
    _meta?: Record<string, unknown>
    structuredContent?: Record<string, unknown>
  }
}
```

**关键设计：** `contextModifier` 允许工具在执行后修改上下文（如改变 cwd），但仅对 `isConcurrencySafe() === false` 的工具有效，因为并发执行的工具修改上下文会产生竞态。

### 6.3 ToolUseContext — 贯穿全局的上下文对象

```typescript
// 文件: src/Tool.ts:158-300
export type ToolUseContext = {
  options: {
    commands: Command[]
    tools: Tools
    mainLoopModel: string
    thinkingConfig: ThinkingConfig
    mcpClients: MCPServerConnection[]
    mcpResources: Record<string, ServerResource[]>
    agentDefinitions: AgentDefinitionsResult
    maxBudgetUsd?: number
    customSystemPrompt?: string
    appendSystemPrompt?: string
    refreshTools?: () => Tools  // 动态刷新工具列表
  }
  abortController: AbortController
  readFileState: FileStateCache    // 文件状态缓存（去重、变更检测）
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  setAppStateForTasks?: (...)      // 始终到达 root store（子代理也能注册基础设施）
  handleElicitation?: (...)        // MCP elicitation 处理
  messages: Message[]              // 当前消息列表
  agentId?: AgentId                // 仅子代理设置
  contentReplacementState?: ContentReplacementState  // 工具结果预算状态
  toolDecisions?: Map<string, {...}>  // 权限决策缓存
  queryTracking?: QueryChainTracking  // 查询链追踪
}
```

---

## 7. 工具注册与条件加载

### 7.1 基础工具列表

```typescript
// 文件: src/tools.ts:193-251
export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    TaskOutputTool,
    BashTool,
    // 当内嵌搜索工具可用时，Glob/Grep 不需要
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    ExitPlanModeV2Tool,
    FileReadTool,
    FileEditTool,
    FileWriteTool,
    NotebookEditTool,
    WebFetchTool,
    TodoWriteTool,
    WebSearchTool,
    // 条件工具...
    ...(isWorktreeModeEnabled() ? [EnterWorktreeTool, ExitWorktreeTool] : []),
    ...(isAgentSwarmsEnabled() ? [getTeamCreateTool(), getTeamDeleteTool()] : []),
    // ToolSearch 本身也是条件加载
    ...(isToolSearchEnabledOptimistic() ? [ToolSearchTool] : []),
  ]
}
```

### 7.2 三层工具池组装

```typescript
// 文件: src/tools.ts:345-367
export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools,
): Tools {
  const builtInTools = getTools(permissionContext);
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext);

  // 分区排序保证 prompt cache 稳定性：
  // 内置工具作为连续前缀排序，MCP 工具在后面排序
  // 如果混合排序，新 MCP 工具会插入内置工具之间，
  // 导致所有下游 cache key 失效
  const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name);
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',  // 内置工具优先（uniqBy 保留首次出现）
  );
}
```

**Prompt Cache 稳定性：** 这是一个关键优化。工具描述是系统提示的一部分，发送给 API。如果工具顺序变化，prompt cache 失效会导致 12x 的输入 token 成本增加。

### 7.3 Deny Rule 过滤

```typescript
// 文件: src/tools.ts:262-269
export function filterToolsByDenyRules<T extends { name: string; mcpInfo?: {...} }>(
  tools: readonly T[],
  permissionContext: ToolPermissionContext,
): T[] {
  return tools.filter(tool => !getDenyRuleForTool(permissionContext, tool));
}
```

**过滤时机：** 工具在**发送给模型之前**就被过滤，不仅仅是在调用时。这样模型不会看到被禁止的工具，避免浪费 token 尝试调用它们。

### 7.4 条件加载与死代码消除

```typescript
// 文件: src/tools.ts:14-53
// 进程环境变量门控（运行时）
const REPLTool = process.env.USER_TYPE === 'ant'
  ? require('./tools/REPLTool/REPLTool.js').REPLTool
  : null;

// Feature flag 门控（编译时 DCE）
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null;

// 数组展开门控
const cronTools = feature('AGENT_TRIGGERS')
  ? [
      require('./tools/ScheduleCronTool/CronCreateTool.js').CronCreateTool,
      require('./tools/ScheduleCronTool/CronDeleteTool.js').CronDeleteTool,
      require('./tools/ScheduleCronTool/CronListTool.js').CronListTool,
    ]
  : [];
```

---

## 8. 命令注册系统

### 8.1 Command 类型体系

```typescript
// 文件: src/types/command.ts:25-57, 74-78, 144-186
// 三种命令类型：

// 1. prompt — 生成提示发送给模型
type PromptCommand = {
  type: 'prompt'
  progressMessage: string
  contentLength: number  // 用于 token 估算
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
  context?: 'inline' | 'fork'  // fork = 子代理执行
  hooks?: HooksSettings         // 技能激活时注册的 hooks
  getPromptForCommand(args, context): Promise<ContentBlockParam[]>
}

// 2. local — 本地执行，返回文本结果
type LocalCommand = {
  type: 'local'
  supportsNonInteractive: boolean
  load: () => Promise<LocalCommandModule>  // 懒加载
}

// 3. local-jsx — 本地执行，返回 React 组件
type LocalJSXCommand = {
  type: 'local-jsx'
  load: () => Promise<LocalJSXCommandModule>  // 懒加载
}
```

### 8.2 命令注册与懒加载

```typescript
// 文件: src/commands.ts:189-202
// 重型模块的懒加载 shim
const usageReport: Command = {
  type: 'prompt',
  name: 'insights',
  description: 'Generate a report analyzing your Claude Code sessions',
  contentLength: 0,
  progressMessage: 'analyzing your sessions',
  source: 'builtin',
  async getPromptForCommand(args, context) {
    // insights.ts 是 113KB (3200行)，延迟到实际调用时才导入
    const real = (await import('./commands/insights.js')).default;
    if (real.type !== 'prompt') throw new Error('unreachable');
    return real.getPromptForCommand(args, context);
  },
};
```

### 8.3 多源技能合并

```typescript
// 文件: src/commands.ts:353-398
async function getSkills(cwd: string): Promise<{...}> {
  const [skillDirCommands, pluginSkills] = await Promise.all([
    getSkillDirCommands(cwd).catch(err => { logError(toError(err)); return []; }),
    getPluginSkills().catch(err => { logError(toError(err)); return []; }),
  ]);

  const bundledSkills = getBundledSkills();       // 同步，启动时注册
  const builtinPluginSkills = getBuiltinPluginSkillCommands();

  return { skillDirCommands, pluginSkills, bundledSkills, builtinPluginSkills };
}
```

**四个命令来源：**
1. 内置命令 (hardcoded in `COMMANDS()`)
2. 技能目录命令 (`~/.claude/skills/`, `.claude/skills/`)
3. 插件技能
4. 内置插件技能

---

## 9. 成本与 Token 追踪

### 9.1 状态管理

成本状态存储在 `bootstrap/state.ts` 的全局状态中，`cost-tracker.ts` 是外观层：

```typescript
// 文件: src/cost-tracker.ts:49-69
export {
  getTotalCostUSD as getTotalCost,
  getTotalDuration,
  getTotalAPIDuration,
  getTotalInputTokens,
  getTotalOutputTokens,
  getTotalCacheReadInputTokens,
  getTotalCacheCreationInputTokens,
  getTotalWebSearchRequests,
  getModelUsage,
  // ...
}
```

### 9.2 每次 API 调用的成本累积

```typescript
// 文件: src/cost-tracker.ts:278-323
export function addToTotalSessionCost(
  cost: number,
  usage: Usage,
  model: string,
): number {
  const modelUsage = addToTotalModelUsage(cost, usage, model);
  addToTotalCostState(cost, modelUsage, model);

  // OpenTelemetry 计数器
  const attrs = isFastModeEnabled() && usage.speed === 'fast'
    ? { model, speed: 'fast' }
    : { model };
  getCostCounter()?.add(cost, attrs);
  getTokenCounter()?.add(usage.input_tokens, { ...attrs, type: 'input' });
  getTokenCounter()?.add(usage.output_tokens, { ...attrs, type: 'output' });

  // 递归处理 advisor（顾问模型）的用量
  let totalCost = cost;
  for (const advisorUsage of getAdvisorUsage(usage)) {
    const advisorCost = calculateUSDCost(advisorUsage.model, advisorUsage);
    totalCost += addToTotalSessionCost(advisorCost, advisorUsage, advisorUsage.model);
  }
  return totalCost;
}
```

### 9.3 会话恢复时的成本恢复

```typescript
// 文件: src/cost-tracker.ts:87-137
export function getStoredSessionCosts(sessionId: string): StoredCostState | undefined {
  const projectConfig = getCurrentProjectConfig();

  // 仅在 session ID 匹配时恢复（防止跨会话成本混淆）
  if (projectConfig.lastSessionId !== sessionId) {
    return undefined;
  }

  return {
    totalCostUSD: projectConfig.lastCost ?? 0,
    totalAPIDuration: projectConfig.lastAPIDuration ?? 0,
    // ... 恢复所有累积状态
    modelUsage: Object.fromEntries(
      Object.entries(projectConfig.lastModelUsage).map(([model, usage]) => [
        model,
        {
          ...usage,
          contextWindow: getContextWindowForModel(model, getSdkBetas()),
          maxOutputTokens: getModelMaxOutputTokens(model).default,
        },
      ]),
    ),
  };
}
```

### 9.4 格式化显示

```typescript
// 文件: src/cost-tracker.ts:228-244
export function formatTotalCost(): string {
  return chalk.dim(
    `Total cost:            ${formatCost(getTotalCostUSD())}${hasUnknownModelCost() ? ' (costs may be inaccurate)' : ''}\n` +
    `Total duration (API):  ${formatDuration(getTotalAPIDuration())}\n` +
    `Total duration (wall): ${formatDuration(getTotalDuration())}\n` +
    `Total code changes:    ${getTotalLinesAdded()} lines added, ${getTotalLinesRemoved()} lines removed\n` +
    `${formatModelUsage()}`
  );
}
```

---

## 10. 上下文收集系统

### 10.1 系统上下文 (Git 状态)

```typescript
// 文件: src/context.ts:116-150
export const getSystemContext = memoize(
  async (): Promise<{ [k: string]: string }> => {
    // CCR（Cloud Code Runner）或禁用 git 指令时跳过
    const gitStatus = isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) ||
      !shouldIncludeGitInstructions()
      ? null
      : await getGitStatus();

    return {
      ...(gitStatus && { gitStatus }),
      // 缓存破坏器（仅内部使用）
      ...(feature('BREAK_CACHE_COMMAND') && injection
        ? { cacheBreaker: `[CACHE_BREAKER: ${injection}]` }
        : {}),
    };
  },
);
```

### 10.2 用户上下文 (CLAUDE.md)

```typescript
// 文件: src/context.ts:155-189
export const getUserContext = memoize(
  async (): Promise<{ [k: string]: string }> => {
    // 三种禁用 CLAUDE.md 的方式：
    // 1. CLAUDE_CODE_DISABLE_CLAUDE_MDS 环境变量
    // 2. --bare 模式（除非有 --add-dir）
    const shouldDisableClaudeMd =
      isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_CLAUDE_MDS) ||
      (isBareMode() && getAdditionalDirectoriesForClaudeMd().length === 0);

    const claudeMd = shouldDisableClaudeMd
      ? null
      : getClaudeMds(filterInjectedMemoryFiles(await getMemoryFiles()));

    // 缓存给 auto-mode 分类器使用（避免循环依赖）
    setCachedClaudeMdContent(claudeMd || null);

    return {
      ...(claudeMd && { claudeMd }),
      currentDate: `Today's date is ${getLocalISODate()}.`,
    };
  },
);
```

### 10.3 Git 状态并行采集

```typescript
// 文件: src/context.ts:60-110
const [branch, mainBranch, status, log, userName] = await Promise.all([
  getBranch(),
  getDefaultBranch(),
  execFileNoThrow(gitExe(), ['--no-optional-locks', 'status', '--short'], {...})
    .then(({ stdout }) => stdout.trim()),
  execFileNoThrow(gitExe(), ['--no-optional-locks', 'log', '--oneline', '-n', '5'], {...})
    .then(({ stdout }) => stdout.trim()),
  execFileNoThrow(gitExe(), ['config', 'user.name'], {...})
    .then(({ stdout }) => stdout.trim()),
]);

// 截断过长的 status
const truncatedStatus = status.length > MAX_STATUS_CHARS  // 2000 chars
  ? status.substring(0, MAX_STATUS_CHARS) + '\n... (truncated)'
  : status;
```

**关键细节：** `--no-optional-locks` 防止 git 操作获取索引锁，避免与用户的并发 git 操作冲突。

### 10.4 memoize + cache.clear 模式

```typescript
// 文件: src/context.ts:29-34
export function setSystemPromptInjection(value: string | null): void {
  systemPromptInjection = value;
  // 注入变化时立即清除上下文缓存
  getUserContext.cache.clear?.();
  getSystemContext.cache.clear?.();
}
```

**模式：** 用 lodash `memoize` 包装异步函数实现"首次调用懒加载 + 后续缓存"。当底层数据变化时，通过 `cache.clear()` 手动失效。

---

## 11. Feature Flag 与死代码消除

### 11.1 bun:bundle 的 feature() 机制

```typescript
// 文件: src/main.tsx:21, src/query.ts:105
import { feature } from 'bun:bundle';
```

`feature()` 在 Bun 编译时被静态求值。在 external 构建中，`feature('COORDINATOR_MODE')` 被替换为 `false`，然后 DCE 移除整个分支及其 `require()`。

### 11.2 三种条件加载模式

```typescript
// 模式1: 顶层条件 require（编译时 DCE）
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js') : null;

// 模式2: 环境变量门控（运行时，不能 DCE）
const REPLTool = process.env.USER_TYPE === 'ant'
  ? require('./tools/REPLTool/REPLTool.js').REPLTool : null;

// 模式3: 函数级延迟 require（运行时，避免循环依赖）
const getTeammateUtils = () =>
  require('./utils/teammate.js') as typeof import('./utils/teammate.js');
```

### 11.3 ESLint 规则协作

```typescript
// eslint-disable-next-line custom-rules/no-top-level-side-effects
profileCheckpoint('main_tsx_entry');

/* eslint-disable @typescript-eslint/no-require-imports */
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js') : null;
/* eslint-enable @typescript-eslint/no-require-imports */
```

自定义 ESLint 规则 `no-top-level-side-effects` 防止意外的模块级副作用。顶层预取是有意为之，需要显式禁用。

---

## 12. 流式处理与错误恢复策略

### 12.1 错误扣留机制 (Withholding)

```typescript
// 文件: src/query.ts:799-825
// 扣留可恢复的错误，直到确认恢复是否可行
// 过早 yield 会导致 SDK 消费者终止会话
let withheld = false;
if (feature('CONTEXT_COLLAPSE')) {
  if (contextCollapse?.isWithheldPromptTooLong(message, isPromptTooLongMessage, querySource)) {
    withheld = true;
  }
}
if (reactiveCompact?.isWithheldPromptTooLong(message)) {
  withheld = true;
}
if (mediaRecoveryEnabled && reactiveCompact?.isWithheldMediaSizeError(message)) {
  withheld = true;
}
if (isWithheldMaxOutputTokens(message)) {
  withheld = true;
}
if (!withheld) {
  yield yieldMessage;
}
```

**为什么扣留？** 如果立即 yield 一个 prompt-too-long 错误，SDK 消费者（如 desktop app）会终止会话。但实际上系统可能通过压缩恢复。扣留让恢复逻辑有机会运行。

### 12.2 Fallback 模型切换

```typescript
// 文件: src/query.ts:894-951
catch (innerError) {
  if (innerError instanceof FallbackTriggeredError && fallbackModel) {
    currentModel = fallbackModel;
    attemptWithFallback = true;

    // 为孤立消息生成墓碑
    for (const msg of assistantMessages) {
      yield { type: 'tombstone' as const, message: msg };
    }

    // 清空状态
    assistantMessages.length = 0;
    toolResults.length = 0;
    toolUseBlocks.length = 0;
    needsFollowUp = false;

    // Thinking 签名是模型绑定的：protected-thinking 块
    // 重放到无保护的 fallback 模型会 400 错误
    if (process.env.USER_TYPE === 'ant') {
      messagesForQuery = stripSignatureBlocks(messagesForQuery);
    }

    yield createSystemMessage(
      `Switched to ${renderModelName(innerError.fallbackModel)} due to high demand`
    );
    continue;
  }
  throw innerError;
}
```

### 12.3 max_output_tokens 恢复策略

```typescript
// 文件: src/query.ts:1188-1256
// 策略1: 升级到 64k tokens（单次）
if (capEnabled && maxOutputTokensOverride === undefined) {
  state = { ...state, maxOutputTokensOverride: ESCALATED_MAX_TOKENS,
    transition: { reason: 'max_output_tokens_escalate' } };
  continue;
}

// 策略2: 注入恢复消息（最多3次）
if (maxOutputTokensRecoveryCount < MAX_OUTPUT_TOKENS_RECOVERY_LIMIT) {
  const recoveryMessage = createUserMessage({
    content: `Output token limit hit. Resume directly — no apology, no recap. ` +
      `Pick up mid-thought. Break remaining work into smaller pieces.`,
    isMeta: true,
  });
  state = {
    messages: [...messagesForQuery, ...assistantMessages, recoveryMessage],
    maxOutputTokensRecoveryCount: maxOutputTokensRecoveryCount + 1,
    transition: { reason: 'max_output_tokens_recovery', attempt: count + 1 },
    // ...
  };
  continue;
}

// 策略3: 恢复耗尽，surface 错误
yield lastMessage;
```

### 12.4 Token Budget 自动继续

```typescript
// 文件: src/query.ts:1308-1355
if (feature('TOKEN_BUDGET')) {
  const decision = checkTokenBudget(
    budgetTracker!, toolUseContext.agentId,
    getCurrentTurnTokenBudget(), getTurnOutputTokens(),
  );

  if (decision.action === 'continue') {
    incrementBudgetContinuationCount();
    state = {
      messages: [...messagesForQuery, ...assistantMessages,
        createUserMessage({ content: decision.nudgeMessage, isMeta: true })],
      transition: { reason: 'token_budget_continuation' },
      // ...
    };
    continue;
  }
}
```

---

## 13. 消息规范化与 API 协议

### 13.1 不可变消息原则

```typescript
// 文件: src/query.ts:747-787
// 在 yield 前克隆消息，为 SDK 观察者添加派生字段
// 原始 message 保持不变——它会被 push 到 assistantMessages 并流回 API
// 变更会破坏 prompt caching（字节不匹配）
let yieldMessage: typeof message = message;
if (message.type === 'assistant') {
  let clonedContent: typeof message.message.content | undefined;
  for (let i = 0; i < message.message.content.length; i++) {
    const block = message.message.content[i]!;
    if (block.type === 'tool_use' && typeof block.input === 'object') {
      const tool = findToolByName(toolUseContext.options.tools, block.name);
      if (tool?.backfillObservableInput) {
        const inputCopy = { ...block.input };
        tool.backfillObservableInput(inputCopy);
        // 仅当 backfill 添加了新字段时才克隆（覆盖会破坏 transcript hash）
        const addedFields = Object.keys(inputCopy).some(k => !(k in block.input));
        if (addedFields) {
          clonedContent ??= [...message.message.content];
          clonedContent[i] = { ...block, input: inputCopy };
        }
      }
    }
  }
  if (clonedContent) {
    yieldMessage = { ...message, message: { ...message.message, content: clonedContent } };
  }
}
```

### 13.2 Thinking 规则

```typescript
// 文件: src/query.ts:151-163 (注释)
/**
 * Thinking 规则：
 * 1. 包含 thinking/redacted_thinking 块的消息必须在 max_thinking_length > 0 的查询中
 * 2. thinking 块不能是消息中的最后一个块
 * 3. thinking 块必须在助手轨迹持续期间保留
 *    （单次 turn，或包含 tool_use 时延续到后续 tool_result 和下一个 assistant 消息）
 *
 * 不遵守这些规则的惩罚：一整天的调试和揪头发。
 */
```

---

## 14. 可复用的工程模式总结

### 14.1 Import 间隙预取

**场景：** CLI 启动有大量 import 需要评估
**模式：** 在 import 语句之间穿插副作用启动子进程/IO
**收益：** 利用 ~135ms 的模块评估时间完成并行工作

### 14.2 分级持久化

**场景：** 需要平衡数据安全性和性能
**模式：** 交互模式 await 写入，脚本模式 fire-and-forget
**代码：** `isBareMode() ? void promise : await promise`

### 14.3 Generator 状态机

**场景：** 复杂的多步骤流程需要流式输出
**模式：** `while(true) { ...process...; state = next; continue; }` + `yield`
**优势：** 背压控制、自然的中断点、清晰的状态转换

### 14.4 Prompt Cache 感知排序

**场景：** 动态工具列表可能破坏 API 缓存
**模式：** 内置工具和外部工具分区排序，内置在前
**收益：** 避免 12x 输入 token 成本增加

### 14.5 错误扣留+恢复

**场景：** 可恢复的 API 错误不应过早暴露给消费者
**模式：** 标记 withheld，尝试恢复，恢复失败才 yield 错误
**适用：** prompt-too-long, max-output-tokens, media-size 错误

### 14.6 条件 require + feature() DCE

**场景：** 特定构建不需要某些模块
**模式：** `feature('X') ? require('...') : null` — 编译时移除
**对比：** 环境变量门控仅运行时生效，不能 DCE

### 14.7 Lazy require 避免循环依赖

**场景：** `A → B → ... → A` 的导入环
**模式：** `const getModule = () => require('./module.js') as typeof import('./module.js')`
**时机：** 在实际需要时调用函数，而非模块评估时

### 14.8 memoize + 手动失效

**场景：** 昂贵的异步计算结果需要缓存，但底层数据可变
**模式：** `lodash memoize` + 变更时 `fn.cache.clear()`
**典型：** `getSystemContext`, `getUserContext`

### 14.9 Decorator/Proxy 权限追踪

**场景：** 需要在不修改核心逻辑的情况下收集遥测数据
**模式：** 包装函数，调用原始实现后记录额外信息
**代码：** `wrappedCanUseTool` in QueryEngine

### 14.10 Settings 路径稳定化

**场景：** 临时文件路径影响 API prompt cache
**模式：** 用内容哈希替代随机 UUID 生成临时文件路径
**代码参考：** `main.tsx:454` — `generateTempFilePath('claude-settings', '.json', { contentHash })`

```typescript
// 文件: src/main.tsx:449-455
// 用内容哈希代替随机 UUID，避免每次子进程更改工具描述
// 从而导致 API prompt cache 失效，引起 12x 输入 token 成本惩罚
settingsPath = generateTempFilePath('claude-settings', '.json', {
  contentHash: trimmedSettings
});
```
