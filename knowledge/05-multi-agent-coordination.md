# 05 - 多 Agent 协调与编排系统

Claude Code 实现了一套完整的多 Agent 协调系统，涵盖 IDE Bridge 通信、Agent Swarm 编排、任务管理、Skill/Plugin 扩展以及跨会话消息传递。本文档从架构、协议、关键数据结构三个层面深入剖析。

---

## 目录

1. [IDE Bridge 架构](#1-ide-bridge-架构)
2. [多 Agent 协调器模式](#2-多-agent-协调器模式)
3. [Agent Swarm 团队系统](#3-agent-swarm-团队系统)
4. [任务管理子系统](#4-任务管理子系统)
5. [Skill 系统架构](#5-skill-系统架构)
6. [Plugin 架构](#6-plugin-架构)
7. [Agent 间通信协议](#7-agent-间通信协议)
8. [子 Agent 生成机制](#8-子-agent-生成机制)
9. [会话管理与认证](#9-会话管理与认证)
10. [可复用的多 Agent 设计模式](#10-可复用的多-agent-设计模式)

---

## 1. IDE Bridge 架构

**核心文件:** `src/bridge/`

### 1.1 整体架构

Bridge 是 Claude Code CLI 与 IDE 扩展 (VSCode/JetBrains) / claude.ai 网页端之间的通信层。它实现了两种运行模式：

- **Standalone Bridge** (`bridgeMain.ts` ~112K): `claude remote-control` 命令启动的独立进程，轮询服务器获取工作，为每个会话 spawn 子 CLI 进程
- **REPL Bridge** (`replBridge.ts` ~98K): 嵌入在交互式 REPL 中的 bridge，通过 `/remote-control` 激活，复用当前会话

### 1.2 消息协议

Bridge 使用基于 WebSocket/SSE 的双向协议，消息类型定义在 `bridgeMessaging.ts`：

```typescript
// SDK 消息类型判断 — 区分控制消息和数据消息
export function isSDKMessage(value: unknown): value is SDKMessage {
  return value !== null && typeof value === 'object'
    && 'type' in value && typeof value.type === 'string'
}

// 控制请求 — 服务器发给客户端的操作指令
export function isSDKControlRequest(value: unknown): value is SDKControlRequest {
  return value !== null && typeof value === 'object'
    && 'type' in value && value.type === 'control_request'
    && 'request_id' in value && 'request' in value
}

// 控制响应 — 客户端返回给服务器的操作结果
export function isSDKControlResponse(value: unknown): value is SDKControlResponse {
  return value !== null && typeof value === 'object'
    && 'type' in value && value.type === 'control_response'
    && 'response' in value
}
```

**消息流过滤** — 只有真实的用户/助手消息才会转发到 bridge：

```typescript
export function isEligibleBridgeMessage(m: Message): boolean {
  if ((m.type === 'user' || m.type === 'assistant') && m.isVirtual) return false
  return m.type === 'user' || m.type === 'assistant'
    || (m.type === 'system' && m.subtype === 'local_command')
}
```

### 1.3 Standalone Bridge 主循环

`bridgeMain.ts` 实现了一个完整的工作轮询-分发循环：

```
注册环境 → 轮询工作 → 解码 WorkSecret → spawn 子进程 → 监控活动 → 清理
    ↑                                                           |
    └───── 多会话模式: 返回轮询 ←──────────────────────────────┘
```

关键数据结构：

```typescript
export type BridgeConfig = {
  dir: string
  machineName: string
  branch: string
  gitRepoUrl: string | null
  maxSessions: number
  spawnMode: SpawnMode // 'single-session' | 'worktree' | 'same-dir'
  bridgeId: string
  workerType: string // 'claude_code' | 'claude_code_assistant'
  environmentId: string
  apiBaseUrl: string
  sessionIngressUrl: string
  sessionTimeoutMs?: number
}

export type SessionHandle = {
  sessionId: string
  done: Promise<SessionDoneStatus>
  kill(): void
  forceKill(): void
  activities: SessionActivity[] // 最近 ~10 条活动的环形缓冲区
  currentActivity: SessionActivity | null
  accessToken: string
  lastStderr: string[]
  writeStdin(data: string): void
  updateAccessToken(token: string): void
}
```

**三种 Spawn 模式:**

| 模式 | 行为 | 场景 |
|------|------|------|
| `single-session` | 一个会话结束后 bridge 退出 | 默认的 `claude remote-control` |
| `worktree` | 每个会话获得独立 git worktree | 多会话并行开发 |
| `same-dir` | 所有会话共享 cwd | 简单多会话 |

### 1.4 控制请求处理

服务器可以通过 `control_request` 控制本地会话。必须在 ~10-14 秒内响应，否则服务器断开连接：

```typescript
export function handleServerControlRequest(
  request: SDKControlRequest,
  handlers: ServerControlRequestHandlers,
): void {
  switch (request.request.subtype) {
    case 'initialize':    // 初始化能力协商
    case 'set_model':     // 远程切换模型
    case 'set_max_thinking_tokens':  // 调整思考 token 限制
    case 'set_permission_mode':      // 设置权限模式
    case 'interrupt':     // 中断当前执行
    default:              // 未知类型返回 error 响应
  }
}
```

### 1.5 Echo 去重 — BoundedUUIDSet

为防止 WebSocket 消息回传被重复处理，使用 FIFO 环形缓冲区去重：

```typescript
export class BoundedUUIDSet {
  private readonly ring: (string | undefined)[]
  private readonly set = new Set<string>()
  private writeIdx = 0

  add(uuid: string): void {
    if (this.set.has(uuid)) return
    const evicted = this.ring[this.writeIdx]
    if (evicted !== undefined) this.set.delete(evicted)
    this.ring[this.writeIdx] = uuid
    this.set.add(uuid)
    this.writeIdx = (this.writeIdx + 1) % this.capacity
  }
}
```

### 1.6 权限桥接

Bridge 将 CLI 的权限请求转发到 web UI，用户在浏览器中审批：

```typescript
type BridgePermissionCallbacks = {
  sendRequest(requestId, toolName, input, toolUseId, description, ...): void
  sendResponse(requestId, response: BridgePermissionResponse): void
  cancelRequest(requestId): void
  onResponse(requestId, handler): () => void // 返回取消订阅函数
}

type BridgePermissionResponse = {
  behavior: 'allow' | 'deny'
  updatedInput?: Record<string, unknown>
  updatedPermissions?: PermissionUpdate[]
}
```

---

## 2. 多 Agent 协调器模式

**核心文件:** `src/coordinator/coordinatorMode.ts`

### 2.1 协调器模式开关

通过 feature flag + 环境变量双重控制：

```typescript
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}
```

恢复会话时自动匹配模式：

```typescript
export function matchSessionMode(
  sessionMode: 'coordinator' | 'normal' | undefined,
): string | undefined {
  if (currentIsCoordinator === sessionIsCoordinator) return undefined
  // 翻转环境变量
  if (sessionIsCoordinator) process.env.CLAUDE_CODE_COORDINATOR_MODE = '1'
  else delete process.env.CLAUDE_CODE_COORDINATOR_MODE
}
```

### 2.2 协调器系统 Prompt

协调器有专门的系统 prompt，定义了分层工作流：

```
Research (Workers, 并行) → Synthesis (Coordinator) → Implementation (Workers) → Verification (Workers)
```

核心原则：
- **协调器负责合成** — 不允许 "based on your findings" 这种把理解推给 worker 的写法
- **并行是超能力** — 独立的 worker 应尽可能并发
- **读写隔离** — 研究任务并行，写操作按文件集串行
- **Continue vs Spawn** — 根据上下文重叠度选择复用 worker 还是新建

### 2.3 Worker 上下文注入

```typescript
export function getCoordinatorUserContext(
  mcpClients: ReadonlyArray<{ name: string }>,
  scratchpadDir?: string,
): { [k: string]: string } {
  // 告诉协调器 worker 有哪些工具
  let content = `Workers spawned via the Agent tool have access to: ${workerTools}`
  // 注入 MCP 服务器信息
  if (mcpClients.length > 0) { content += `\n\nWorkers also have MCP tools from: ${serverNames}` }
  // 注入 Scratchpad 共享目录
  if (scratchpadDir && isScratchpadGateEnabled()) {
    content += `\n\nScratchpad: ${scratchpadDir}\nWorkers can read/write here without permission prompts.`
  }
}
```

---

## 3. Agent Swarm 团队系统

**核心文件:** `src/utils/swarm/`

### 3.1 团队文件结构

团队状态通过 JSON 文件 (`~/.claude/teams/{team-name}/config.json`) 持久化：

```typescript
export type TeamFile = {
  name: string
  description?: string
  createdAt: number
  leadAgentId: string
  leadSessionId?: string // 用于团队发现
  hiddenPaneIds?: string[]
  teamAllowedPaths?: TeamAllowedPath[] // 团队成员可免审编辑的路径
  members: Array<{
    agentId: string      // "name@teamName" 格式
    name: string
    agentType?: string
    model?: string
    color?: string
    planModeRequired?: boolean
    joinedAt: number
    tmuxPaneId: string
    cwd: string
    worktreePath?: string
    sessionId?: string
    subscriptions: string[]
    backendType?: BackendType // 'tmux' | 'iterm2' | 'in-process'
    isActive?: boolean
    mode?: PermissionMode
  }>
}
```

### 3.2 三种执行后端

```typescript
export type BackendType = 'tmux' | 'iterm2' | 'in-process'
```

**后端检测与回退链:**

```
iTerm2 + it2 CLI 可用? → iterm2
  ↓ 否
tmux 可用? → tmux
  ↓ 否
in-process (AsyncLocalStorage 隔离)
```

```typescript
export type PaneBackend = {
  readonly type: BackendType
  readonly displayName: string
  readonly supportsHideShow: boolean
  isAvailable(): Promise<boolean>
  isRunningInside(): Promise<boolean>
  createTeammatePaneInSwarmView(name, color): Promise<CreatePaneResult>
  sendCommandToPane(paneId, command): Promise<void>
  killPane(paneId): Promise<boolean>
  hidePane(paneId): Promise<boolean>
  showPane(paneId, targetWindowOrPane): Promise<boolean>
}

export type TeammateExecutor = {
  readonly type: BackendType
  isAvailable(): Promise<boolean>
  spawn(config: TeammateSpawnConfig): Promise<TeammateSpawnResult>
  sendMessage(agentId, message): Promise<void>
  terminate(agentId, reason?): Promise<boolean>
  kill(agentId): Promise<boolean>
  isActive(agentId): Promise<boolean>
}
```

### 3.3 In-Process 团队成员生成

最轻量的后端 — 同进程、AsyncLocalStorage 隔离：

```typescript
export async function spawnInProcessTeammate(
  config: InProcessSpawnConfig,
  context: SpawnContext,
): Promise<InProcessSpawnOutput> {
  // 1. 生成确定性 agent ID
  const agentId = formatAgentId(name, teamName)  // "researcher@my-team"
  const taskId = generateTaskId('in_process_teammate')

  // 2. 创建独立 AbortController（不受 leader 中断影响）
  const abortController = createAbortController()

  // 3. 创建 TeammateContext（用于 AsyncLocalStorage）
  const teammateContext = createTeammateContext({
    agentId, agentName: name, teamName, color,
    planModeRequired, parentSessionId, abortController,
  })

  // 4. 创建任务状态并注册
  const taskState: InProcessTeammateTaskState = {
    ...createTaskStateBase(taskId, 'in_process_teammate', description),
    type: 'in_process_teammate',
    status: 'running',
    identity: { agentId, agentName: name, teamName, color, planModeRequired, parentSessionId },
    prompt, model, abortController,
    permissionMode: planModeRequired ? 'plan' : 'default',
    isIdle: false, shutdownRequested: false,
    pendingUserMessages: [], messages: [],
  }

  // 5. 注册清理钩子
  const unregisterCleanup = registerCleanup(async () => {
    abortController.abort()
  })
  registerTask(taskState, setAppState)
}
```

### 3.4 团队成员系统 Prompt 附加

每个团队成员的系统 prompt 会附加通信指令：

```typescript
export const TEAMMATE_SYSTEM_PROMPT_ADDENDUM = `
# Agent Teammate Communication
IMPORTANT: You are running as an agent in a team.
- Use SendMessage tool with \`to: "<name>"\` to send messages to specific teammates
- Use SendMessage tool with \`to: "*"\` sparingly for team-wide broadcasts
Just writing a response in text is NOT visible to others on your team.
`
```

### 3.5 团队生命周期管理

```
TeamCreate → 注册团队文件 → 分配颜色 → 初始化任务目录
  ↓
spawn 团队成员 (InProcessBackend | PaneBackend)
  ↓
TeamDelete → 检查活跃成员 → 清理 worktree → 删除团队/任务目录
  ↓
会话退出时自动清理未显式删除的团队 (gracefulShutdown)
```

```typescript
// 自动清理机制 — 防止 orphan 团队目录
export function registerTeamForSessionCleanup(teamName: string): void {
  getSessionCreatedTeams().add(teamName)
}

export async function cleanupSessionTeams(): Promise<void> {
  // 先 kill pane (防止 orphan 进程)，再删目录
  await Promise.allSettled(teams.map(name => killOrphanedTeammatePanes(name)))
  await Promise.allSettled(teams.map(name => cleanupTeamDirectories(name)))
}
```

---

## 4. 任务管理子系统

**核心文件:** `src/tasks/`

### 4.1 任务类型联合

```typescript
export type TaskState =
  | LocalShellTaskState    // 本地 shell 命令
  | LocalAgentTaskState    // 本地子 agent
  | RemoteAgentTaskState   // 远程 CCR agent
  | InProcessTeammateTaskState  // 进程内团队成员
  | LocalWorkflowTaskState // 本地工作流
  | MonitorMcpTaskState    // MCP 监控
  | DreamTaskState         // 自动记忆整理 (auto-dream)
```

### 4.2 LocalAgentTask — 子 Agent 任务

最核心的任务类型，管理通过 AgentTool 生成的子 agent：

```typescript
export type LocalAgentTaskState = TaskStateBase & {
  type: 'local_agent'
  agentId: string
  prompt: string
  selectedAgent?: AgentDefinition
  agentType: string
  model?: string
  abortController?: AbortController
  error?: string
  result?: AgentToolResult
  progress?: AgentProgress    // tool 使用次数、token 消耗、最近活动
  retrieved: boolean
  messages?: Message[]        // 会话历史
  isBackgrounded: boolean     // 前台 vs 后台
  pendingMessages: string[]   // SendMessage 排队
  retain: boolean             // UI 是否持有（阻止回收）
  diskLoaded: boolean         // 是否已从磁盘加载
  evictAfter?: number         // 过期回收时间戳
}
```

**进度追踪模式:**

```typescript
export type AgentProgress = {
  toolUseCount: number
  tokenCount: number
  lastActivity?: ToolActivity
  recentActivities?: ToolActivity[]
  summary?: string
}

// 从 assistant 消息中更新进度
export function updateProgressFromMessage(
  tracker: ProgressTracker, message: Message, ...
): void {
  // input_tokens 是累积的，取最新值
  tracker.latestInputTokens = usage.input_tokens + cache_creation + cache_read
  // output_tokens 是每轮的，累加
  tracker.cumulativeOutputTokens += usage.output_tokens
  // 记录最近 5 个 tool_use 活动
  for (const content of message.message.content) {
    if (content.type === 'tool_use') tracker.toolUseCount++
  }
}
```

### 4.3 InProcessTeammateTask — 进程内团队成员

运行在同一 Node.js 进程中，通过 AsyncLocalStorage 实现上下文隔离：

```typescript
export type InProcessTeammateTaskState = TaskStateBase & {
  type: 'in_process_teammate'
  identity: TeammateIdentity   // agentId, agentName, teamName, color, ...
  prompt: string
  model?: string
  abortController?: AbortController       // kill 整个 teammate
  currentWorkAbortController?: AbortController  // 仅中断当前轮次
  awaitingPlanApproval: boolean
  permissionMode: PermissionMode
  messages?: Message[]           // UI 展示用，上限 50 条
  pendingUserMessages: string[]  // 用户在 transcript view 中输入的消息队列
  isIdle: boolean
  shutdownRequested: boolean
  onIdleCallbacks?: Array<() => void>  // 通知 leader
}

// 内存保护 — 限制 UI 消息副本数量
export const TEAMMATE_MESSAGES_UI_CAP = 50
```

### 4.4 RemoteAgentTask — 远程 Agent 任务

远程在 CCR (Claude Code Runner) 环境中执行的任务：

```typescript
export type RemoteAgentTaskState = TaskStateBase & {
  type: 'remote_agent'
  remoteTaskType: RemoteTaskType  // 'remote-agent' | 'ultraplan' | 'ultrareview' | 'autofix-pr' | 'background-pr'
  remoteTaskMetadata?: RemoteTaskMetadata  // PR 号、仓库等
  sessionId: string
  command: string
  title: string
  todoList: TodoList
  log: SDKMessage[]
  isLongRunning?: boolean
  pollStartedAt: number
  isRemoteReview?: boolean
  reviewProgress?: { stage, bugsFound, bugsVerified, bugsRefuted }
  isUltraplan?: boolean
  ultraplanPhase?: 'needs_input' | 'plan_ready'
}

// 完成检查器 — 外部轮询判断任务是否完成
export type RemoteTaskCompletionChecker = (
  metadata: RemoteTaskMetadata | undefined
) => Promise<string | null>
```

### 4.5 DreamTask — 自动记忆整理

后台运行的记忆合并 agent，纯 UI 跟踪：

```typescript
export type DreamTaskState = TaskStateBase & {
  type: 'dream'
  phase: 'starting' | 'updating'
  sessionsReviewing: number
  filesTouched: string[]   // 不完整反映，仅捕获 Edit/Write tool_use
  turns: DreamTurn[]       // 最近 30 轮
  abortController?: AbortController
  priorMtime: number       // kill 时回滚锁
}
```

### 4.6 通知机制

Agent 完成时通过 XML notification 通知父 agent：

```xml
<task-notification>
  <task-id>{agentId}</task-id>
  <status>completed|failed|killed</status>
  <summary>{description}</summary>
  <result>{agent 最终文本响应}</result>
  <usage>
    <total_tokens>N</total_tokens>
    <tool_uses>N</tool_uses>
    <duration_ms>N</duration_ms>
  </usage>
</task-notification>
```

---

## 5. Skill 系统架构

**核心文件:** `src/skills/`

### 5.1 Skill 来源层级

```typescript
export type LoadedFrom =
  | 'commands_DEPRECATED'  // 旧版 commands 目录
  | 'skills'               // .claude/skills/ 目录
  | 'plugin'               // 通过 plugin 安装
  | 'managed'              // 企业托管配置
  | 'bundled'              // CLI 内置
  | 'mcp'                  // MCP 服务器提供
```

Skill 搜索路径（按优先级）：

```typescript
export function getSkillsPath(
  source: SettingSource | 'plugin',
  dir: 'skills' | 'commands',
): string {
  switch (source) {
    case 'policySettings': return join(getManagedFilePath(), '.claude', dir)
    case 'userSettings':   return join(getClaudeConfigHomeDir(), dir)
    case 'projectSettings': return `.claude/${dir}`
    case 'plugin':          return 'plugin'
  }
}
```

### 5.2 Skill Frontmatter 解析

每个 skill 是一个 Markdown 文件，通过 YAML frontmatter 定义元数据：

```typescript
export function parseSkillFrontmatterFields(frontmatter, markdownContent, resolvedName): {
  displayName: string | undefined
  description: string
  allowedTools: string[]        // 限制可用工具
  argumentHint: string          // 参数提示
  argumentNames: string[]       // 参数名称列表
  whenToUse: string             // 模型触发条件
  version: string
  model: string                 // 可指定 'inherit'
  disableModelInvocation: boolean  // 仅人工触发
  userInvocable: boolean         // 是否在 /skill 列表显示
  hooks: HooksSettings           // 关联的钩子
  executionContext: 'fork'       // 在独立上下文中执行
  agent: string                  // 绑定到特定 agent 类型
  effort: EffortValue            // 推理努力程度
  shell: FrontmatterShell        // shell 命令执行配置
}
```

### 5.3 Bundled Skills 注册

内置 skill 在启动时程序化注册：

```typescript
export type BundledSkillDefinition = {
  name: string
  description: string
  aliases?: string[]
  whenToUse?: string
  allowedTools?: string[]
  model?: string
  disableModelInvocation?: boolean
  userInvocable?: boolean
  isEnabled?: () => boolean
  hooks?: HooksSettings
  context?: 'inline' | 'fork'    // inline = 在当前上下文, fork = 隔离子 agent
  agent?: string
  files?: Record<string, string>  // 附带的参考文件
  getPromptForCommand: (args, context) => Promise<ContentBlockParam[]>
}

// 注册示例
export function initBundledSkills(): void {
  registerUpdateConfigSkill()   // /update-config
  registerKeybindingsSkill()    // /keybindings
  registerVerifySkill()         // /verify
  registerDebugSkill()          // /debug
  registerSimplifySkill()       // /simplify
  registerBatchSkill()          // /batch
  registerStuckSkill()          // /stuck
  // Feature-gated:
  if (feature('AGENT_TRIGGERS')) registerLoopSkill()       // /loop
  if (feature('AGENT_TRIGGERS_REMOTE')) registerScheduleRemoteAgentsSkill() // /schedule
  if (feature('BUILDING_CLAUDE_APPS')) registerClaudeApiSkill()  // /claude-api
}
```

### 5.4 Skill 执行 — Inline vs Fork

```typescript
// Skill 可以在两种上下文中执行：
// inline: 在当前对话上下文中直接注入 prompt
// fork:   在隔离的子 agent 中执行，有独立 token 预算
```

**附带文件的安全提取:**

```typescript
// O_NOFOLLOW | O_EXCL 防止符号链接攻击
const SAFE_WRITE_FLAGS = fsConstants.O_WRONLY | fsConstants.O_CREAT | fsConstants.O_EXCL | O_NOFOLLOW

// 路径遍历验证
function resolveSkillFilePath(baseDir: string, relPath: string): string {
  const normalized = normalize(relPath)
  if (isAbsolute(normalized) || normalized.split(pathSep).includes('..')) {
    throw new Error(`bundled skill file path escapes skill dir: ${relPath}`)
  }
  return join(baseDir, normalized)
}
```

### 5.5 Skill Token 预算管理

Skill 列表占用上下文窗口的 ~1%：

```typescript
export const SKILL_BUDGET_CONTEXT_PERCENT = 0.01
export const MAX_LISTING_DESC_CHARS = 250  // 每条描述上限

export function formatCommandsWithinBudget(commands, contextWindowTokens?): string {
  // 1. 尝试完整描述
  // 2. 超预算? bundled skill 保持全描述，其他截断
  // 3. 极端情况? 非 bundled 只显示名称
}
```

---

## 6. Plugin 架构

**核心文件:** `src/plugins/`

### 6.1 Built-in Plugin 注册

```typescript
export type BuiltinPluginDefinition = {
  name: string
  description: string
  version: string
  defaultEnabled?: boolean
  isAvailable?: () => boolean   // 环境检测
  skills?: BundledSkillDefinition[]
  hooks?: HooksSettings
  mcpServers?: McpServerConfig[]
}

// 注册
export function registerBuiltinPlugin(definition: BuiltinPluginDefinition): void {
  BUILTIN_PLUGINS.set(definition.name, definition)
}

// 插件 ID 格式: {name}@builtin
export function isBuiltinPluginId(pluginId: string): boolean {
  return pluginId.endsWith('@builtin')
}
```

### 6.2 Plugin 启用/禁用

用户可以通过 `/plugin` UI 切换：

```typescript
export function getBuiltinPlugins(): { enabled: LoadedPlugin[], disabled: LoadedPlugin[] } {
  for (const [name, definition] of BUILTIN_PLUGINS) {
    if (definition.isAvailable && !definition.isAvailable()) continue
    const userSetting = settings?.enabledPlugins?.[pluginId]
    const isEnabled = userSetting !== undefined ? userSetting === true : (definition.defaultEnabled ?? true)
    // 分配到 enabled 或 disabled 列表
  }
}
```

### 6.3 Plugin 提供的 Skill

```typescript
export function getBuiltinPluginSkillCommands(): Command[] {
  const { enabled } = getBuiltinPlugins()
  for (const plugin of enabled) {
    const definition = BUILTIN_PLUGINS.get(plugin.name)
    if (!definition?.skills) continue
    for (const skill of definition.skills) {
      commands.push(skillDefinitionToCommand(skill))
    }
  }
}
```

---

## 7. Agent 间通信协议

**核心文件:** `src/tools/SendMessageTool/`

### 7.1 消息类型

```typescript
const StructuredMessage = z.discriminatedUnion('type', [
  z.object({
    type: z.literal('shutdown_request'),
    reason: z.string().optional(),
  }),
  z.object({
    type: z.literal('shutdown_response'),
    request_id: z.string(),
    approve: semanticBoolean(),
  }),
  z.object({
    type: z.literal('plan_approval_response'),
    request_id: z.string(),
    approve: semanticBoolean(),
    feedback: z.string().optional(),
  }),
])

// 输入 schema
const inputSchema = z.object({
  to: z.string(),     // 团队成员名 | "*" 广播 | "uds:path" | "bridge:session_id"
  summary: z.string().optional(),  // 5-10 词摘要
  message: z.union([z.string(), StructuredMessage]),
})
```

### 7.2 跨会话通信 (UDS)

通过 Unix Domain Socket 实现同机器上不同 Claude Code 会话的通信：

```typescript
// feature('UDS_INBOX') 启用时：
// "uds:/tmp/cc-socks/1234.sock" — 本地 Claude 会话
// "bridge:session_01AbCd..." — Remote Control 远程会话

// 消息包装为 <cross-session-message from="..."> XML
// 接收方回复时，复制 from 属性到 to 字段
```

### 7.3 消息路由

```
SendMessage(to="researcher", message="...")
  ↓
是否为 InProcessTeammate? → findTeammateTaskByAgentId → 直接投递到 pendingMessages
  ↓ 否
是否为 LocalAgent? → queuePendingMessage → 在 tool-round 边界消费
  ↓ 否
是否为 Pane (tmux/iterm2)? → writeToMailbox → 文件系统邮箱
  ↓ 否
是否为 UDS/Bridge? → 通过 socket/bridge 发送
  ↓ 否
是否为 "*" (广播)? → 遍历所有团队成员逐一发送
```

### 7.4 Shutdown 协议

```
Leader → shutdown_request → Teammate
                              ↓
                          approve: true → gracefulShutdown() → 进程退出
                          approve: false + reason → Leader 收到拒绝
```

---

## 8. 子 Agent 生成机制

**核心文件:** `src/tools/AgentTool/`

### 8.1 Agent 定义

```typescript
export type BaseAgentDefinition = {
  agentType: string        // "Explore", "code-reviewer", ...
  whenToUse: string        // 触发条件描述
  tools?: string[]         // 工具白名单
  disallowedTools?: string[]  // 工具黑名单
  skills?: string[]        // 预加载的 skill
  mcpServers?: AgentMcpServerSpec[]  // agent 专属 MCP 服务器
  hooks?: HooksSettings
  color?: AgentColorName
  model?: string           // 可指定 'inherit'
  effort?: EffortValue
  permissionMode?: PermissionMode  // 'default' | 'plan' | 'bubble'
  maxTurns?: number
  background?: boolean
  isolation?: 'worktree' | 'remote'
}

// 内置 agent
export type BuiltInAgentDefinition = BaseAgentDefinition & {
  source: 'built-in'
  getSystemPrompt: () => string
}

// 用户自定义 agent (从 .claude/agents/*.md 加载)
export type CustomAgentDefinition = BaseAgentDefinition & {
  source: SettingSource
  prompt: string
  initialPrompt?: string
  memory?: AgentMemoryScope
}
```

### 8.2 内置 Agent 列表

```typescript
export function getBuiltInAgents(): AgentDefinition[] {
  // 协调器模式 → 返回 coordinator 专用 agents
  if (isCoordinatorMode()) return getCoordinatorAgents()

  const agents = [
    GENERAL_PURPOSE_AGENT,     // 通用
    STATUSLINE_SETUP_AGENT,    // 状态栏设置
  ]
  if (areExplorePlanAgentsEnabled()) {
    agents.push(EXPLORE_AGENT, PLAN_AGENT)  // 探索和规划
  }
  if (isNonSdkEntrypoint) agents.push(CLAUDE_CODE_GUIDE_AGENT)  // 使用指南
  if (feature('VERIFICATION_AGENT')) agents.push(VERIFICATION_AGENT)  // 验证
  return agents
}
```

### 8.3 Fork 子 Agent

Fork 是一种特殊的子 agent 模式 — 继承父 agent 的完整上下文和系统 prompt：

```typescript
export function isForkSubagentEnabled(): boolean {
  if (feature('FORK_SUBAGENT')) {
    if (isCoordinatorMode()) return false     // 与协调器互斥
    if (getIsNonInteractiveSession()) return false  // SDK 模式不支持
    return true
  }
  return false
}

export const FORK_AGENT = {
  agentType: 'fork',
  tools: ['*'],           // 继承父 agent 的全部工具
  maxTurns: 200,
  model: 'inherit',       // 使用父 agent 的模型（共享缓存）
  permissionMode: 'bubble', // 权限弹出到父终端
  source: 'built-in',
  getSystemPrompt: () => '',  // 不使用 — fork 直接传入父 agent 的渲染后 prompt
}
```

**Fork vs 普通 Agent 的区别：**

| 特性 | Fork | 普通 Agent |
|------|------|-----------|
| 上下文 | 继承父 agent 完整对话 | 从零开始 |
| Prompt | 指令式（不需要解释背景） | 需要完整上下文描述 |
| Prompt Cache | 共享父 agent 的缓存 | 无法复用 |
| 递归 | 检测 `FORK_BOILERPLATE_TAG` 禁止递归 | 允许嵌套 |

### 8.4 Agent 运行核心 (runAgent)

`runAgent.ts` (~35K) 是所有子 agent 执行的入口：

```typescript
// 简化的 runAgent 流程:
async function runAgent(agentDefinition, prompt, context, override?): Promise<AgentToolResult> {
  // 1. 初始化 agent 专属 MCP 服务器（如果有）
  const { clients, tools, cleanup } = await initializeAgentMcpServers(agentDefinition, parentClients)

  // 2. 解析工具集（白名单/黑名单/MCP 合并）
  const resolvedTools = resolveAgentTools(agentDefinition, tools)

  // 3. 构建系统 prompt
  const systemPrompt = override?.systemPrompt ?? agentDefinition.getSystemPrompt()

  // 4. 创建子 agent 上下文（独立的 file state cache、transcript 路径等）
  const subagentContext = createSubagentContext(context, {
    agentId, tools: resolvedTools, systemPrompt, ...
  })

  // 5. 执行查询循环（复用核心 query() 引擎）
  const result = await query(messages, subagentContext)

  // 6. 清理并返回结果
  cleanup()
  return result
}
```

### 8.5 Agent 列表注入模式

为减少 prompt cache 失效，agent 列表可作为附件消息而非内嵌在工具描述中：

```typescript
export function shouldInjectAgentListInMessages(): boolean {
  // 开启后，agent 列表通过 attachment message 注入
  // 工具描述变为静态，MCP/plugin 变更不再触发 cache bust
  return getFeatureValue_CACHED_MAY_BE_STALE('tengu_agent_list_attach', false)
}
```

---

## 9. 会话管理与认证

### 9.1 JWT Token 管理

Bridge 使用 JWT 进行会话认证，内置 token 自动刷新机制：

```typescript
export function createTokenRefreshScheduler({
  getAccessToken, onRefresh, label, refreshBufferMs = 5 * 60 * 1000,
}): {
  schedule: (sessionId, token) => void
  scheduleFromExpiresIn: (sessionId, expiresInSeconds) => void
  cancel: (sessionId) => void
  cancelAll: () => void
} {
  // 核心机制:
  // 1. 解码 JWT payload 获取 exp 声明
  // 2. 在过期前 5 分钟触发刷新
  // 3. 刷新失败最多重试 3 次
  // 4. 成功后安排 30 分钟的跟进刷新（防止长会话过期）
  // 5. 使用 generation counter 防止并发 doRefresh 冲突
}
```

### 9.2 会话创建与归档

```typescript
// 创建会话
export async function createBridgeSession({
  environmentId, title, events, gitRepoUrl, branch, signal, ...
}): Promise<string | null> {
  // POST /v1/sessions
  // 包含 git 上下文（source 和 outcome）、模型信息、权限模式
}

// 归档会话
export async function archiveBridgeSession(sessionId, opts?): Promise<void> {
  // POST /v1/sessions/{id}/archive
  // 幂等 — 已归档返回 409，安全重复调用
}

// 更新标题
export async function updateBridgeSessionTitle(sessionId, title, opts?): Promise<void> {
  // PATCH /v1/sessions/{id}
  // 最佳努力，错误被吞掉
}
```

### 9.3 WorkSecret 解码

服务器通过 `work.secret` 传递会话配置：

```typescript
export type WorkSecret = {
  version: number
  session_ingress_token: string    // 会话级 JWT
  api_base_url: string
  sources: Array<{
    type: string
    git_info?: { type, repo, ref?, token? }
  }>
  auth: Array<{ type: string, token: string }>
  claude_code_args?: Record<string, string>
  mcp_config?: unknown
  environment_variables?: Record<string, string>
  use_code_sessions?: boolean
}
```

---

## 10. 可复用的多 Agent 设计模式

### 10.1 环形缓冲区去重 (BoundedUUIDSet)

适用于任何需要"记住最近 N 项"的场景：

```typescript
// O(1) 查询，O(capacity) 空间，自动淘汰最旧条目
class BoundedUUIDSet {
  private readonly ring: (string | undefined)[]
  private readonly set = new Set<string>()
  private writeIdx = 0
  // add(uuid) — 环形覆盖 + set 同步
  // has(uuid) — Set O(1) 查询
}
```

### 10.2 Generation Counter 防并发

用于 token 刷新调度器，但可泛化到任何异步操作：

```typescript
// 问题：异步 doRefresh 完成时，可能已被新的 schedule 取代
// 方案：每次 schedule/cancel 递增 generation，doRefresh 完成后检查
const generations = new Map<string, number>()

function nextGeneration(id: string): number {
  const gen = (generations.get(id) ?? 0) + 1
  generations.set(id, gen)
  return gen
}

async function doRefresh(id: string, gen: number): Promise<void> {
  await someAsyncWork()
  if (generations.get(id) !== gen) return // stale, 跳过
  // 继续处理...
}
```

### 10.3 Coordinator-Worker 分层

协调器模式的核心模式 — 可用于任何多 agent 编排：

```
Coordinator（合成者）
├── 接收用户请求
├── 分解为独立子任务
├── 并行分发给 Worker
├── 等待 <task-notification> 回报
├── 合成理解，制定下一步
└── 向用户报告

Worker（执行者）
├── 接收自包含 prompt（包含文件路径、行号、具体指令）
├── 自主执行（无法看到 Coordinator 的对话）
├── 自我验证（运行测试、类型检查）
└── 返回结果通知
```

### 10.4 三层执行后端抽象

```
TeammateExecutor (统一接口)
├── InProcessBackend    — AsyncLocalStorage, 零开销
├── PaneBackendExecutor — 包装 PaneBackend
│   ├── TmuxBackend    — tmux split-pane
│   └── ITermBackend   — iTerm2 native pane
└── 自动检测回退链
```

### 10.5 文件系统邮箱

Pane-based 团队成员使用文件系统进行消息传递（`teammateMailbox.ts`）：

```
~/.claude/teams/{team}/{agent}.inbox
├── writeToMailbox(agentId, message)   — 追加 JSON 行
├── readMailbox(agentId)               — 读取所有未处理消息
└── markMessageAsReadByIndex(...)      — 标记已读
```

### 10.6 Skill 懒加载 + 安全提取

```typescript
// 延迟提取：首次调用时写入磁盘
// 进程级缓存：memoize promise 防止并发提取
let extractionPromise: Promise<string | null> | undefined
getPromptForCommand = async (args, ctx) => {
  extractionPromise ??= extractBundledSkillFiles(name, files)
  const extractedDir = await extractionPromise
  return prependBaseDir(blocks, extractedDir)
}

// 安全写入：O_NOFOLLOW | O_EXCL 防止符号链接攻击
// 路径验证：normalize + 禁止 .. 防止目录遍历
```

### 10.7 Task 状态机

```
pending → running → completed
                  → failed
                  → killed

// 后台任务标识：
isBackgroundTask(task): task is running|pending 且 isBackgrounded !== false

// 回收机制：
evictAfter timestamp → 过期后 UI 隐藏 + 状态清理
retain flag → 阻止回收（UI 正在查看时）
```

### 10.8 Agent 通知作为 User Message

Agent 完成的通知被包装为 `user` 类型消息注入到对话中，这样 LLM 可以自然地处理。使用 XML 标签 `<task-notification>` 作为区分标记，避免与真实用户消息混淆。

---

## 关键架构决策总结

| 决策 | 原因 |
|------|------|
| Bridge 消息过滤 `isEligibleBridgeMessage` | 只转发真实用户/助手消息，过滤内部 REPL chatter |
| Agent 列表从工具描述移到附件消息 | 减少 10.2% 的 prompt cache bust |
| In-process teammate 使用 AsyncLocalStorage | 避免进程间通信开销，同时保持上下文隔离 |
| Fork agent 继承 `renderedSystemPrompt` bytes | 字节精确匹配，保证 prompt cache 命中 |
| Team 文件持久化到磁盘 | 跨进程共享状态，pane-based teammate 需要文件发现 |
| Skill 占上下文 1% 预算 | 平衡发现性和 token 效率 |
| 控制请求必须 10-14 秒内响应 | 服务器会断开连接，未知类型返回 error 而非沉默 |
| DreamTask 的 `filesTouched` 标注为不完整 | 只捕获 Edit/Write tool_use，miss bash 写入 |
| `TEAMMATE_MESSAGES_UI_CAP = 50` | 防止鲸鱼会话（292 agents, 36.8GB）的内存爆炸 |
