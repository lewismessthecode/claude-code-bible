# 02 - 工具系统 (Tool System)

Claude Code 的工具系统是其核心能力层，定义了 AI 如何与外部世界交互。本文档深入分析工具接口契约、权限模型、各工具实现模式和安全设计。

---

## 1. 工具接口契约 (Tool Interface Contract)

**核心文件**: `src/Tool.ts`

### 1.1 Tool 类型定义

每个工具必须实现 `Tool` 类型（约 700 行），这是一个泛型类型：

```typescript
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  readonly name: string
  readonly inputSchema: Input

  // === 核心方法 ===
  call(args, context, canUseTool, parentMessage, onProgress?): Promise<ToolResult<Output>>
  checkPermissions(input, context): Promise<PermissionResult>

  // === 元信息方法 ===
  description(input, options): Promise<string>
  prompt(options): Promise<string>
  userFacingName(input): string

  // === 行为标记方法 ===
  isEnabled(): boolean
  isConcurrencySafe(input): boolean
  isReadOnly(input): boolean
  isDestructive?(input): boolean

  // === 渲染方法 ===
  renderToolUseMessage(input, options): React.ReactNode
  renderToolResultMessage?(content, progressMessages, options): React.ReactNode
  mapToolResultToToolResultBlockParam(content, toolUseID): ToolResultBlockParam

  // === 可选高级方法 ===
  validateInput?(input, context): Promise<ValidationResult>
  preparePermissionMatcher?(input): Promise<(pattern: string) => boolean>
  getPath?(input): string
  backfillObservableInput?(input): void
  isSearchOrReadCommand?(input): { isSearch, isRead, isList? }
  interruptBehavior?(): 'cancel' | 'block'
  toAutoClassifierInput(input): unknown

  maxResultSizeChars: number
  strict?: boolean
  shouldDefer?: boolean
  alwaysLoad?: boolean
  aliases?: string[]
  searchHint?: string
}
```

### 1.2 buildTool 工厂函数 —— 消除样板代码

`buildTool` 是所有工具的工厂函数，提供 fail-closed 默认值：

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,    // 默认不安全（保守）
  isReadOnly: () => false,            // 默认假定有写操作
  isDestructive: () => false,
  checkPermissions: (input) => Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: () => '',    // 默认跳过分类器
  userFacingName: () => '',
}

export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

**设计亮点**：
- `ToolDef` 类型让 `buildTool` 的入参可以省略带默认值的方法
- `BuiltTool<D>` 类型精确反映 `{...defaults, ...def}` 的合并语义
- 所有 60+ 工具都通过 `buildTool` 创建，默认值集中在一处

### 1.3 ToolUseContext —— 执行上下文

`ToolUseContext` 是传递给每个 `call()` 方法的巨大上下文对象（~60 字段），包含：

```typescript
export type ToolUseContext = {
  // 配置
  options: {
    tools: Tools
    commands: Command[]
    mainLoopModel: string
    mcpClients: MCPServerConnection[]
    thinkingConfig: ThinkingConfig
    maxBudgetUsd?: number
  }

  // 控制
  abortController: AbortController
  messages: Message[]

  // 状态访问
  readFileState: FileStateCache        // 文件读取缓存（防止过时写入）
  getAppState(): AppState
  setAppState(f: (prev) => AppState): void

  // UI 交互
  setToolJSX?: SetToolJSXFn            // 显示工具自定义 JSX
  addNotification?: (notif) => void
  setStreamMode?: (mode: SpinnerMode) => void

  // 代理标识
  agentId?: AgentId                    // 仅子代理设置
  agentType?: string
  toolUseId?: string

  // 文件历史
  updateFileHistoryState: (updater) => void
  updateAttributionState: (updater) => void

  // 限制
  fileReadingLimits?: { maxTokens?, maxSizeBytes? }
  globLimits?: { maxResults? }
}
```

### 1.4 ToolResult —— 返回值

```typescript
export type ToolResult<T> = {
  data: T                               // 工具输出数据
  newMessages?: Message[]               // 注入到对话的额外消息
  contextModifier?: (ctx) => ctx        // 修改后续执行上下文（仅非并发安全工具）
  mcpMeta?: { _meta?, structuredContent? }  // MCP 协议透传
}
```

---

## 2. 工具注册与加载 (Tool Registration)

**核心文件**: `src/tools.ts`

### 2.1 工具发现与加载模式

工具通过 `getAllBaseTools()` 注册，使用条件加载实现特性门控：

```typescript
export function getAllBaseTools(): Tools {
  return [
    // 始终加载的核心工具
    AgentTool, BashTool, FileReadTool, FileEditTool, FileWriteTool,
    NotebookEditTool, WebFetchTool, WebSearchTool, TodoWriteTool,
    AskUserQuestionTool, SkillTool, ExitPlanModeV2Tool, EnterPlanModeTool,
    TaskStopTool, BriefTool,

    // 条件加载 - 嵌入式搜索工具
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),

    // 特性门控 - Bun 的 feature() 做编译时 DCE
    ...(SleepTool ? [SleepTool] : []),
    ...cronTools,                        // feature('AGENT_TRIGGERS')
    ...(WebBrowserTool ? [WebBrowserTool] : []),

    // 环境门控
    ...(process.env.USER_TYPE === 'ant' ? [ConfigTool, TungstenTool] : []),
    ...(isEnvTruthy(process.env.ENABLE_LSP_TOOL) ? [LSPTool] : []),

    // 功能门控
    ...(isTodoV2Enabled() ? [TaskCreateTool, TaskGetTool, TaskUpdateTool, TaskListTool] : []),
    ...(isWorktreeModeEnabled() ? [EnterWorktreeTool, ExitWorktreeTool] : []),
    ...(isAgentSwarmsEnabled() ? [getTeamCreateTool(), getTeamDeleteTool()] : []),

    // MCP 资源工具
    ListMcpResourcesTool, ReadMcpResourceTool,

    // 工具搜索（用于超多工具时延迟加载）
    ...(isToolSearchEnabledOptimistic() ? [ToolSearchTool] : []),
  ]
}
```

### 2.2 工具池组装

工具经过多层过滤才到达模型：

```
getAllBaseTools()
  → filterToolsByDenyRules()      // 按 deny 规则移除
  → isEnabled() 过滤              // 检查每个工具是否启用
  → REPL 模式过滤                 // REPL 模式隐藏原始工具
  → assembleToolPool()            // 合并 MCP 工具
    → 按名称排序（内置在前，MCP 在后）
    → uniqBy('name')              // 去重，内置优先
```

```typescript
export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools,
): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)
  const byName = (a, b) => a.name.localeCompare(b.name)
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',
  )
}
```

### 2.3 延迟加载与工具搜索

当工具数量很多时，使用 `shouldDefer` 标记工具为延迟加载：

- 延迟的工具以 `defer_loading: true` 发送给 API，模型看到名称但没有完整 schema
- 模型需先调用 `ToolSearchTool` 来"发现"工具，才能正确调用
- `searchHint` 字段辅助关键词匹配

---

## 3. 权限模型 (Permission Model)

### 3.1 三层权限架构

权限检查遵循分层决策链：

```
层 1: 工具级 deny 规则 → 工具直接从列表移除，模型看不到
层 2: 工具自带 checkPermissions() → 工具特定的安全检查
层 3: hasPermissionsToUseTool() → 通用权限系统
      → 3a: 精确/前缀匹配 allow 规则
      → 3b: 精确/前缀匹配 deny 规则
      → 3c: 精确/前缀匹配 ask 规则
      → 3d: 权限模式检查 (default/plan/bypassPermissions)
      → 3e: 分类器检查 (bash classifier)
层 4: useCanUseTool hook → 最终决策
      → 交互式提示 / 自动批准 / 拒绝
```

### 3.2 权限规则系统

权限规则由 `ToolPermissionContext` 管理：

```typescript
export type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode                              // 'default' | 'plan' | 'bypassPermissions'
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource      // { localSettings: string[], ... }
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  shouldAvoidPermissionPrompts?: boolean             // 后台代理自动拒绝
}>
```

规则来源按优先级：
1. `policySettings` — 企业管理员策略
2. `localSettings` — 用户本地 settings.json
3. `userSettings` — 用户全局设置
4. `projectSettings` — 项目级 CLAUDE.md 设置
5. `session` — 会话级临时授权
6. `cliArg` — 命令行参数
7. `command` — 命令级规则

规则格式示例：
```
"Bash"                    → 整个 Bash 工具
"Bash(git commit:*)"      → git commit 及其子命令
"Bash(npm run test)"      → 精确匹配
"Read(/home/user/**)"     → 路径通配符
"mcp__server1"            → 整个 MCP 服务器
```

### 3.3 useCanUseTool 流程

`useCanUseTool.tsx` 是权限检查的终端执行点：

```
1. hasPermissionsToUseTool() → allow/deny/ask
2. 如果 allow → 直接放行，记录决策
3. 如果 deny → 直接拒绝，通知用户
4. 如果 ask:
   a. 协调器模式：handleCoordinatorPermission()
   b. Swarm 工作者：handleSwarmWorkerPermission()
   c. Bash 分类器：peekSpeculativeClassifierCheck()
      → 提前启动的投机检查，2 秒超时
   d. 交互式：handleInteractivePermission()
      → 显示权限对话框，用户选择
```

### 3.4 Bash 分类器 (Bash Classifier)

Bash 工具有额外的 AI 分类器层：

```typescript
// 投机启动：在权限检查开始前就启动分类器
startSpeculativeClassifierCheck(command, permissionContext, signal)

// 三种行为的分类器：
classifyBashCommand(command, 'allow', allowDescriptions)  // 自动允许
classifyBashCommand(command, 'deny', denyDescriptions)    // 自动拒绝
classifyBashCommand(command, 'ask', askDescriptions)       // 需要询问
```

---

## 4. BashTool 深入分析

**文件**: `src/tools/BashTool/` (~156K 字节，15+ 文件)

### 4.1 架构总览

BashTool 是最复杂的工具，包含：

```
BashTool.tsx            → 主入口，call() 实现，命令语义分析
bashPermissions.ts      → 权限检查逻辑（1200+ 行）
bashSecurity.ts         → 安全验证（命令注入检测）
readOnlyValidation.ts   → 只读命令白名单验证
pathValidation.ts       → 路径安全验证
sedValidation.ts        → sed 命令安全验证
modeValidation.ts       → 权限模式检查
shouldUseSandbox.ts     → 沙箱决策
commandSemantics.ts     → 退出码语义解释
sedEditParser.ts        → sed 编辑命令解析
destructiveCommandWarning.ts → 破坏性命令警告
```

### 4.2 输入 Schema

```typescript
const fullInputSchema = z.strictObject({
  command: z.string(),
  timeout: semanticNumber(z.number().optional()),
  description: z.string().optional(),
  run_in_background: semanticBoolean(z.boolean().optional()),
  dangerouslyDisableSandbox: semanticBoolean(z.boolean().optional()),
  _simulatedSedEdit: z.object({          // 内部字段，不暴露给模型
    filePath: z.string(),
    newContent: z.string(),
  }).optional(),
})

// 对模型隐藏 _simulatedSedEdit（防止绕过权限检查）
const inputSchema = fullInputSchema.omit({ _simulatedSedEdit: true })
```

### 4.3 命令执行流程

```
1. validateInput() → 拦截 sleep N 模式
2. checkPermissions() → bashToolHasPermission()
   a. splitCommand_DEPRECATED() → 拆分复合命令
   b. 每个子命令：
      - stripSafeWrappers() → 剥离安全包装器 (timeout, nohup, nice, etc.)
      - 检查 allow/deny/ask 规则
      - checkReadOnlyConstraints() → 只读命令白名单
      - bashCommandIsSafeAsync() → 安全性分析
      - checkPathConstraints() → 路径验证
      - checkSedConstraints() → sed 安全验证
   c. 合并所有子命令的权限决策
3. call()
   a. 检查 _simulatedSedEdit（sed 预览路径）
   b. runShellCommand() → 异步生成器
      - exec(command, signal, 'bash', options)
      - 进度轮询（每秒）
      - 自动后台化（超时后）
      - 用户手动后台化（Ctrl+B）
   c. 后处理：
      - trackGitOperations() → 跟踪 git 操作
      - interpretCommandResult() → 退出码语义解释
      - SandboxManager.annotateStderrWithSandboxFailures()
      - extractClaudeCodeHints() → 提取 CLI 提示
      - stripEmptyLines() → 清理输出
      - resizeShellImageOutput() → 图像输出压缩
      - 大输出持久化到 tool-results 目录
```

### 4.4 安全机制：stripSafeWrappers

防止命令包装器绕过权限检查：

```typescript
export function stripSafeWrappers(command: string): string {
  // 阶段 1：剥离安全环境变量（NODE_ENV=prod npm run build → npm run build）
  // 只处理 SAFE_ENV_VARS 白名单中的变量
  // SECURITY: 永不剥离 PATH, LD_PRELOAD, NODE_PATH 等危险变量

  // 阶段 2：剥离安全包装器命令
  const SAFE_WRAPPER_PATTERNS = [
    /^timeout .../ ,    // timeout 30s command
    /^time .../,        // time command
    /^nice .../,        // nice -n 10 command
    /^stdbuf .../,      // stdbuf -o0 command
    /^nohup .../,       // nohup command
  ]

  // SECURITY: 使用 [ \t]+ 而非 \s+，\s 匹配换行符会导致跨行剥离
  // SECURITY: 消费 -- 终止符，否则 nohup -- rm 会以 -- 作为 baseCmd
}
```

### 4.5 安全机制：bashSecurity.ts

多层命令注入检测：

```typescript
// 危险模式检测
const COMMAND_SUBSTITUTION_PATTERNS = [
  { pattern: /\$\(/, message: '$() command substitution' },
  { pattern: /\$\{/, message: '${} parameter substitution' },
  { pattern: /<\(/,  message: 'process substitution <()' },
  { pattern: />\(/,  message: 'process substitution >()' },
  { pattern: /=\(/,  message: 'Zsh process substitution =()' },
  { pattern: /\$\[/, message: '$[] legacy arithmetic expansion' },
]

// Zsh 特定危险命令
const ZSH_DANGEROUS_COMMANDS = new Set([
  'zmodload',   // 加载模块（可能获得文件I/O、网络能力）
  'emulate',    // -c flag 是 eval 等价
  'sysopen', 'sysread', 'syswrite',  // 文件描述符操作
  'zpty',       // 伪终端命令执行
  'ztcp',       // TCP 连接（数据泄露）
])

// 安全检查编号系统（避免日志中出现字符串）
const BASH_SECURITY_CHECK_IDS = {
  INCOMPLETE_COMMANDS: 1,
  JQ_SYSTEM_FUNCTION: 2,
  OBFUSCATED_FLAGS: 4,
  SHELL_METACHARACTERS: 5,
  DANGEROUS_VARIABLES: 6,
  NEWLINES: 7,
  CONTROL_CHARACTERS: 17,
  UNICODE_WHITESPACE: 18,
  // ... 23 种检查
}
```

### 4.6 安全机制：readOnlyValidation.ts

只读命令白名单验证系统：

```typescript
// 统一命令验证配置
type CommandConfig = {
  safeFlags: Record<string, FlagArgType>  // 安全 flag 及其值类型
  regex?: RegExp                           // 额外正则验证
  additionalCommandIsDangerousCallback?: (cmd, args) => boolean
  respectsDoubleDash?: boolean            // 是否支持 -- 终止
}

// 白名单命令包括：
// GIT_READ_ONLY_COMMANDS: git status, git log, git diff, git show, etc.
// GH_READ_ONLY_COMMANDS: gh pr view, gh issue list, etc.
// DOCKER_READ_ONLY_COMMANDS: docker ps, docker images, etc.
// RIPGREP_READ_ONLY_COMMANDS: rg pattern
// EXTERNAL_READONLY_COMMANDS: cat, head, tail, ls, tree, etc.
```

### 4.7 沙箱集成

```typescript
function shouldUseSandbox(input: BashToolInput): boolean {
  // 检查命令是否应在沙箱中运行
  // 考虑因素：
  // 1. dangerouslyDisableSandbox 输入参数
  // 2. 动态配置中禁用的命令
  // 3. 用户设置中排除的命令
  // 4. 沙箱全局开关
}

// 沙箱违规标注
SandboxManager.annotateStderrWithSandboxFailures(command, stdout)
```

### 4.8 后台执行

```typescript
// 三种后台化路径：
// 1. 显式：run_in_background: true → 立即后台化
// 2. 超时：命令超时后自动后台化
// 3. 助手模式：超过 15 秒的阻塞命令自动后台化

async function* runShellCommand({input, ...}) {
  // 使用 AsyncGenerator 实现流式进度
  const shellCommand = await exec(command, signal, 'bash', {
    timeout: timeoutMs,
    onProgress(lastLines, allLines, totalLines, totalBytes, isIncomplete) {
      resolveProgress()  // 唤醒生成器产出进度
    },
    shouldUseSandbox: shouldUseSandbox(input),
    shouldAutoBackground,
  })

  // 进度循环
  while (true) {
    const progressSignal = createProgressSignal()
    const result = await Promise.race([resultPromise, progressSignal])
    if (result !== null) {
      // 命令完成
      return result
    }
    yield { type: 'progress', output, fullOutput, elapsedTimeSeconds, ... }
  }
}
```

---

## 5. 文件工具模式 (File Tools)

### 5.1 FileReadTool

**特点**：
- 支持多种文件类型：文本、图像、PDF、Jupyter notebook
- `maxResultSizeChars: Infinity` — 永不持久化（避免循环读取）
- 使用 `FileStateCache` 缓存文件修改时间
- 图像自动压缩和尺寸调整
- PDF 支持页范围读取（最多 20 页/次）
- 行号格式化（`cat -n` 风格）
- offset/limit 分段读取大文件
- 注册 `fileReadListeners` 通知其他服务

```typescript
// 输出是联合类型
type Output =
  | { type: 'text', file: { content, numLines, startLine, totalLines } }
  | { type: 'image', file: { base64, type, originalSize, dimensions } }
  | { type: 'notebook', file: { filePath, cells } }
  | { type: 'pdf', file: { filePath, base64, originalSize } }
  | { type: 'parts', file: { filePath, count, outputDir } }
  | { type: 'file_unchanged', file: { filePath } }
```

**文件不变检测**：如果文件自上次读取后未修改，返回 `file_unchanged` 存根而非完整内容，节省 token。

**安全考量**：
- 阻止设备文件路径（`/dev/zero`, `/dev/random` 等会挂起进程）
- UNC 路径检查（防止 Windows NTLM 凭据泄露）
- 读取权限通过 `checkReadPermissionForTool()` 检查

### 5.2 FileEditTool

**特点**：
- 基于字符串替换的精确编辑（非 diff/patch）
- `old_string` 必须在文件中唯一
- 支持 `replace_all` 批量替换
- 文件历史追踪（undo 支持）
- 编码检测和行尾符保持
- VS Code 通知
- Git diff 生成用于 UI 显示

```typescript
// 输入
type FileEditInput = {
  file_path: string
  old_string: string
  new_string: string
  replace_all?: boolean
}

// 验证逻辑
validateInput() {
  // 1. 检查 team memory 中的密钥
  // 2. old_string !== new_string
  // 3. 路径 deny 规则检查
  // 4. 文件大小限制（1 GiB）
  // 5. 文件是否已被外部修改（stale write 检测）
  // 6. old_string 是否在文件中存在
  // 7. 唯一性检查（非 replace_all 模式）
  // 8. settings.json 特殊验证
}
```

**防止过时写入机制**：

```typescript
// FileStateCache 追踪每次读取的时间戳
readFileState.set(filePath, {
  content: newContent,
  timestamp: getFileModificationTime(filePath),
  offset, limit,
})

// 编辑前检查文件是否在 Claude 读取后被修改
if (fileModTime > lastReadTime) {
  return FILE_UNEXPECTEDLY_MODIFIED_ERROR
}
```

### 5.3 FileWriteTool

- 创建新文件或完全覆写
- 与 FileEditTool 共享权限检查路径
- 自动创建父目录

---

## 6. 搜索工具模式 (Search Tools)

### 6.1 GlobTool

- 包装 `glob()` 工具函数
- 结果按修改时间排序
- 默认限制 100 个文件
- 纯只读，并发安全

```typescript
// 简洁的实现模式
export const GlobTool = buildTool({
  name: GLOB_TOOL_NAME,
  isConcurrencySafe: () => true,
  isReadOnly: () => true,
  isSearchOrReadCommand: () => ({ isSearch: true, isRead: false }),

  async call({ pattern, path }, context) {
    const searchPath = path ? expandPath(path) : getCwd()
    const results = await glob(pattern, searchPath, {
      maxResults: context.globLimits?.maxResults || 100,
    })
    return { data: { filenames: results, numFiles: results.length, ... } }
  }
})
```

### 6.2 GrepTool

- 包装 `ripgrep` (rg) 工具
- 支持多种输出模式：content, files_with_matches, count
- 支持上下文行（-A, -B, -C）、行号（-n）
- head_limit（默认 250）和 offset 分页
- 自动排除版本控制目录（.git, .svn 等）
- 插件缓存排除模式

---

## 7. AgentTool —— 子代理生成

**文件**: `src/tools/AgentTool/` (20+ 文件)

### 7.1 架构

AgentTool 是工具系统中最复杂的工具之一，负责生成隔离的子代理：

```typescript
const baseInputSchema = z.object({
  description: z.string(),          // 3-5 词任务描述
  prompt: z.string(),               // 详细任务指令
  subagent_type: z.string().optional(),  // 代理类型 (Explore, Plan, etc.)
  model: z.enum(['sonnet', 'opus', 'haiku']).optional(),
  run_in_background: z.boolean().optional(),

  // 多代理参数
  name: z.string().optional(),       // 可寻址名称
  team_name: z.string().optional(),
  mode: permissionModeSchema().optional(),

  // 隔离模式
  isolation: z.enum(['worktree', 'remote']).optional(),
  cwd: z.string().optional(),
})
```

### 7.2 内置代理类型

```
src/tools/AgentTool/built-in/
  generalPurposeAgent.ts      → 通用代理（默认）
  exploreAgent.ts             → 代码探索代理
  planAgent.ts                → 计划制定代理
  verificationAgent.ts        → 验证代理
  claudeCodeGuideAgent.ts     → Claude Code 指南代理
```

### 7.3 子代理执行流程

```typescript
// runAgent.ts 中的核心流程
async function runAgent(params): Promise<AgentToolResult> {
  // 1. 创建子代理上下文
  const subagentContext = createSubagentContext(parentContext, {
    agentId, agentType, readFileState, tools, ...
  })

  // 2. 初始化代理特定 MCP 服务器
  const { clients, tools: mcpTools } = await initializeAgentMcpServers(
    agentDefinition, parentClients
  )

  // 3. 组装工具池
  const agentTools = assembleToolPool(permissionContext, mcpTools)
  const filteredTools = resolveAgentTools(agentDefinition, agentTools)

  // 4. 构建系统提示
  const systemPrompt = await buildEffectiveSystemPrompt(agentDefinition, ...)

  // 5. 执行查询循环（递归调用 query()）
  const result = await query({
    messages: [createUserMessage({ content: prompt })],
    systemPrompt,
    tools: filteredTools,
    ...subagentContext,
  })

  // 6. 清理
  await cleanup()
  return result
}
```

### 7.4 隔离模式

- **Worktree 隔离**: 创建临时 git worktree，代理在隔离副本中工作
- **Remote 隔离**: 在远程 CCR 环境中启动（仅内部用户）
- **后台执行**: 代理可在后台运行，通过通知系统报告完成

### 7.5 Fork 子代理

Fork 子代理模式共享父代理的提示缓存：

```typescript
// forkSubagent.ts
export function buildForkedMessages(
  parentMessages: Message[],
  prompt: string,
  cacheSafeParams?: CacheSafeParams,
): Message[] {
  // 使用父代理的消息历史，附加新提示
  // 共享 renderedSystemPrompt 以重用缓存
}
```

---

## 8. 工具执行流水线

**核心文件**: `src/services/tools/toolExecution.ts`

### 8.1 runToolUse —— 入口点

```typescript
export async function* runToolUse(
  toolUse: ToolUseBlock,
  assistantMessage: AssistantMessage,
  canUseTool: CanUseToolFn,
  toolUseContext: ToolUseContext,
): AsyncGenerator<MessageUpdateLazy, void> {
  // 1. 查找工具（优先活跃工具，回退到别名）
  let tool = findToolByName(tools, toolName)
  if (!tool) {
    const fallbackTool = findToolByName(getAllBaseTools(), toolName)
    if (fallbackTool?.aliases?.includes(toolName)) tool = fallbackTool
  }

  // 2. 检查中止信号
  if (abortController.signal.aborted) { yield cancelMessage; return }

  // 3. 流式执行
  for await (const update of streamedCheckPermissionsAndCallTool(...)) {
    yield update
  }
}
```

### 8.2 checkPermissionsAndCallTool —— 完整流水线

```
1. Zod schema 验证 → tool.inputSchema.safeParse(input)
   → 如果失败，检查 ToolSearch 提示（延迟加载的工具可能没有 schema）

2. validateInput() → 工具自定义验证
   → 返回 { result: false, message, errorCode }

3. 投机启动 Bash 分类器（并行运行）

4. Pre-tool hooks → runPreToolUseHooks()
   → 钩子可修改输入或阻止执行

5. 权限检查 → canUseTool()
   → allow / deny / ask 三路决策

6. 执行工具 → tool.call(validatedInput, context, canUseTool, msg, onProgress)

7. Post-tool hooks → runPostToolUseHooks()
   → 钩子可修改输出

8. 结果处理：
   → mapToolResultToToolResultBlockParam() → API 格式
   → processToolResultBlock() → 大结果持久化
   → newMessages 注入
   → contextModifier 应用
```

### 8.3 大结果持久化

当工具输出超过 `maxResultSizeChars` 时：

```typescript
processToolResultBlock(result, tool.maxResultSizeChars)
// 1. 检查结果字符数
// 2. 如果超限，写入 ~/.claude/tool-results/<uuid>.txt
// 3. 生成预览 + 文件路径消息
// 4. 模型可用 FileRead 读取完整输出
```

---

## 9. 其他重要工具

### 9.1 SkillTool

技能执行工具，通过 `runAgent()` 在子代理中执行技能指令：

```typescript
// 查找技能命令
const command = await findCommand(skillName, getAllCommands(context))

// 执行技能提示
const result = await runAgent({
  messages: [createUserMessage({ content: skillPrompt })],
  tools: forkedTools,
  ...
})
```

### 9.2 MCPTool

MCP 工具模板，在 `mcpClient.ts` 中被克隆并覆写：

```typescript
export const MCPTool = buildTool({
  isMcp: true,
  name: 'mcp',  // 被覆写为 mcp__server__tool

  // 以下方法全部在 mcpClient.ts 中覆写：
  call(), description(), prompt(), userFacingName(), isOpenWorld()

  checkPermissions: () => ({ behavior: 'passthrough' }),  // 透传到通用权限
})
```

### 9.3 Task 工具族

```
TaskCreateTool  → 创建任务（status: pending）
TaskGetTool     → 获取任务详情
TaskUpdateTool  → 更新任务状态/内容
TaskListTool    → 列出所有任务
TaskStopTool    → 停止后台任务
```

### 9.4 消息工具

```
SendMessageTool  → 向团队成员发消息
TeamCreateTool   → 创建代理团队
TeamDeleteTool   → 删除代理团队
```

### 9.5 计划模式工具

```
EnterPlanModeTool  → 进入计划模式（权限切换到 plan）
ExitPlanModeV2Tool → 退出计划模式（恢复权限）
```

### 9.6 Worktree 工具

```
EnterWorktreeTool  → 创建 git worktree 并切换
ExitWorktreeTool   → 退出 worktree（保留或删除）
```

### 9.7 定时工具

```
CronCreateTool    → 创建定时任务（cron 表达式）
CronDeleteTool    → 删除定时任务
CronListTool      → 列出定时任务
SleepTool         → 延时（feature-gated）
RemoteTriggerTool → 远程触发器
```

### 9.8 配置与杂项工具

```
ConfigTool         → 修改配置（仅内部用户）
BriefTool          → 控制输出简洁度
SyntheticOutputTool → 合成输出（内部使用）
AskUserQuestionTool → 向用户提问
NotebookEditTool    → Jupyter notebook 编辑
```

---

## 10. 可复用模式总结

### 10.1 buildTool 工厂模式

**模式**: 使用泛型工厂函数创建具有默认值的类型安全工具。

```typescript
// 定义
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return { ...TOOL_DEFAULTS, userFacingName: () => def.name, ...def }
}

// 使用
export const MyTool = buildTool({
  name: 'MyTool',
  maxResultSizeChars: 100_000,
  get inputSchema() { return inputSchema() },
  async call(input, context) { ... },
  // 省略 isEnabled, isReadOnly 等 → 使用默认值
} satisfies ToolDef<InputSchema, Output>)
```

**可复用之处**: 适用于任何需要大量可选接口方法、且需要安全默认值的插件系统。

### 10.2 lazySchema 延迟加载模式

```typescript
const inputSchema = lazySchema(() => z.strictObject({
  command: z.string(),
  timeout: z.number().optional(),
}))

// 使用
get inputSchema() { return inputSchema() }
```

**可复用之处**: 避免模块加载时创建所有 Zod schema（尤其是有跨模块依赖时）。

### 10.3 条件工具加载与 DCE

```typescript
// Bun 编译时特性检查
const SleepTool = feature('PROACTIVE')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null

// 运行时条件
...(isWorktreeModeEnabled() ? [EnterWorktreeTool, ExitWorktreeTool] : [])
```

**可复用之处**: 结合编译时 DCE 和运行时门控实现功能标志。

### 10.4 分层权限模式

```
层 1: 静态过滤（编译/配置时移除工具）
层 2: 工具自检（每个工具的 checkPermissions）
层 3: 通用权限引擎（规则匹配、分类器）
层 4: 交互式确认（UI 提示）
```

**可复用之处**: 适用于任何需要灵活权限控制的系统。

### 10.5 AsyncGenerator 进度报告

```typescript
async function* runShellCommand(params): AsyncGenerator<Progress, Result, void> {
  // 使用 Promise.race 实现非阻塞进度报告
  while (true) {
    const progressSignal = createProgressSignal()
    const result = await Promise.race([resultPromise, progressSignal])
    if (result !== null) return result
    yield { type: 'progress', output, elapsedTime, ... }
  }
}

// 消费
for await (const progress of runShellCommand(params)) {
  onProgress(progress)
}
const finalResult = generator.return // 从 return 获取最终结果
```

**可复用之处**: 长时间运行操作的进度报告模式。

### 10.6 安全验证白名单模式

```typescript
type CommandConfig = {
  safeFlags: Record<string, FlagArgType>,
  regex?: RegExp,
  additionalCommandIsDangerousCallback?: (cmd, args) => boolean,
  respectsDoubleDash?: boolean,
}

// 使用配置驱动的验证，而非硬编码
const FD_SAFE_FLAGS: Record<string, FlagArgType> = {
  '-h': 'none', '--help': 'none',
  '-d': 'number', '--max-depth': 'number',
  '-t': 'string', '--type': 'string',
  // SECURITY: -x/--exec deliberately excluded
}
```

**可复用之处**: 命令行工具安全验证的声明式配置。

### 10.7 结果大小管理

```typescript
// 每个工具声明最大结果大小
maxResultSizeChars: 30_000     // BashTool
maxResultSizeChars: 100_000    // FileEditTool, GlobTool, GrepTool
maxResultSizeChars: Infinity   // FileReadTool（自有限制）

// 超限时持久化到文件
processToolResultBlock(result, maxChars)
// → 写入 ~/.claude/tool-results/<uuid>.txt
// → 返回预览 + 文件路径
```

### 10.8 UI 与逻辑分离

每个工具目录通常包含：
```
MyTool/
  MyTool.ts    → 核心逻辑（call, validateInput, checkPermissions）
  prompt.ts    → 提示词和描述
  UI.tsx       → 渲染方法（renderToolUseMessage, renderToolResultMessage）
  constants.ts → 常量
  types.ts     → 类型定义（如有）
```

---

## 11. 工具生命周期时序图

```
模型输出 tool_use block
    │
    ▼
runToolUse()
    │
    ├── 查找工具（名称 + 别名回退）
    │
    ├── Zod 验证 inputSchema
    │
    ├── tool.validateInput()
    │       └── 工具特定验证
    │
    ├── [BashTool] 投机启动分类器
    │
    ├── PreToolUse hooks
    │       └── 钩子可修改输入或阻止
    │
    ├── canUseTool() 权限检查
    │       ├── hasPermissionsToUseTool()
    │       │     ├── allow 规则匹配
    │       │     ├── deny 规则匹配
    │       │     ├── ask 规则匹配
    │       │     ├── 权限模式检查
    │       │     └── 分类器检查
    │       │
    │       ├── [如果 ask] 交互式提示
    │       │     ├── 协调器路径
    │       │     ├── Swarm 工作者路径
    │       │     └── 用户确认路径
    │       │
    │       └── allow / deny 决策
    │
    ├── tool.call(input, context, ...)
    │       ├── 执行核心逻辑
    │       ├── 通过 onProgress 报告进度
    │       └── 返回 ToolResult
    │
    ├── PostToolUse hooks
    │       └── 钩子可修改输出
    │
    ├── mapToolResultToToolResultBlockParam()
    │       └── 转换为 API 格式
    │
    ├── processToolResultBlock()
    │       └── 大结果持久化
    │
    └── 返回 MessageUpdateLazy
            ├── tool_result 消息
            ├── newMessages（附加消息）
            └── contextModifier（上下文修改）
```
