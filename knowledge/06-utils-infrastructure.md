# 06 - Utils & Infrastructure (工具库与基础设施)

> 基于 `src/utils/`(298 文件 + 31 子目录)、`src/bootstrap/`、`src/entrypoints/`、`src/migrations/`、`src/schemas/`、`src/memdir/`、`src/context/`、`src/types/` 的源码分析。

---

## 目录

1. [Bootstrap 全局单例模式](#1-bootstrap-全局单例模式)
2. [启动流程 (Startup Flow)](#2-启动流程)
3. [配置系统 (Configuration System)](#3-配置系统)
4. [Settings 多源合并系统](#4-settings-多源合并系统)
5. [安全存储 (Secure Storage)](#5-安全存储)
6. [权限系统 (Permission System)](#6-权限系统)
7. [Bash AST 安全分析](#7-bash-ast-安全分析)
8. [Git 工具库](#8-git-工具库)
9. [Token 管理与上下文分析](#9-token-管理与上下文分析)
10. [CLAUDE.md 与持久化记忆系统](#10-claudemd-与持久化记忆系统)
11. [配置迁移 (Migrations)](#11-配置迁移)
12. [Zod Schema 验证](#12-zod-schema-验证)
13. [取消/中止机制 (Abort Controller)](#13-取消中止机制)
14. [Signal 事件原语](#14-signal-事件原语)
15. [Startup Profiler 启动性能分析](#15-startup-profiler)
16. [Cleanup Registry 清理注册表](#16-cleanup-registry)
17. [Telemetry 遥测集成](#17-telemetry-遥测集成)
18. [Thinking 模式支持](#18-thinking-模式支持)
19. [可复用的基础设施模式](#19-可复用的基础设施模式)

---

## 1. Bootstrap 全局单例模式

**关键文件**: `src/bootstrap/state.ts` (55KB, 唯一文件)

### 设计原理 — 叶模块隔离

Bootstrap 是整个依赖图的叶节点 (leaf module)，**禁止从 `src/` 导入任何模块**（除极少数通过显式 eslint-disable 标记的例外）。这是防止循环依赖的核心架构约束。

```typescript
// bootstrap/state.ts 顶部注释
// DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE

// 自定义 ESLint 规则 custom-rules/bootstrap-isolation 强制执行:
// 只允许导入外部包和 type-only 导入
```

### State 单例结构

全局状态通过一个私有 `STATE` 对象持有，通过细粒度 getter/setter 暴露：

```typescript
type State = {
  // === 身份与路径 ===
  originalCwd: string           // 启动时的工作目录
  projectRoot: string           // 稳定的项目根目录，不随 EnterWorktree 变化
  cwd: string                   // 当前工作目录（可变）
  sessionId: SessionId          // UUID 格式的会话 ID

  // === 成本与性能指标 ===
  totalCostUSD: number
  totalAPIDuration: number
  totalToolDuration: number
  turnHookDurationMs: number    // 单轮钩子耗时
  turnToolCount: number         // 单轮工具调用数

  // === 模型配置 ===
  modelUsage: { [modelName: string]: ModelUsage }
  mainLoopModelOverride: ModelSetting | undefined
  initialMainLoopModel: ModelSetting
  modelStrings: ModelStrings | null

  // === 遥测 (OpenTelemetry) ===
  meter: Meter | null
  sessionCounter: AttributedCounter | null
  loggerProvider: LoggerProvider | null
  tracerProvider: BasicTracerProvider | null

  // === 缓存优化粘性开关 ===
  afkModeHeaderLatched: boolean | null   // AFK 模式 header，一旦启用不再关闭
  fastModeHeaderLatched: boolean | null  // 快速模式 header 粘性开关
  promptCache1hEligible: boolean | null  // 1小时 prompt cache 资格判定

  // === 会话级标志（不持久化到磁盘）===
  sessionBypassPermissionsMode: boolean
  sessionTrustAccepted: boolean
  scheduledTasksEnabled: boolean
  sessionCronTasks: SessionCronTask[]
  sessionCreatedTeams: Set<string>       // 用于 gracefulShutdown 清理

  // ... 更多字段（总计 ~100 个字段）
}

const STATE: State = getInitialState()

// 初始化时解析符号链接确保路径一致性
function getInitialState(): State {
  let resolvedCwd = ''
  const rawCwd = cwd()
  try {
    resolvedCwd = realpathSync(rawCwd).normalize('NFC')
  } catch {
    resolvedCwd = rawCwd.normalize('NFC')
  }
  // ...
}
```

### 关键设计模式

1. **粘性开关 (Latching)**：某些 beta header 一旦启用就永不关闭，避免中途切换导致 prompt cache 失效
2. **SessionId 原子切换**：`switchSession()` 同时更新 sessionId 和 sessionProjectDir，防止两者不同步
3. **Signal 通知**：使用 `createSignal` 原语在 sessionId 变化时通知订阅者（如 PID 文件管理）
4. **弱引用控制**：使用 `resetStateForTests()` 在测试间完全重置状态

```typescript
// 原子切换会话
export function switchSession(
  sessionId: SessionId,
  projectDir: string | null = null,
): void {
  STATE.planSlugCache.delete(STATE.sessionId)  // 清理旧会话缓存
  STATE.sessionId = sessionId
  STATE.sessionProjectDir = projectDir
  sessionSwitched.emit(sessionId)               // 通知订阅者
}
```

---

## 2. 启动流程

**关键文件**: `src/entrypoints/cli.tsx`、`src/entrypoints/init.ts`

### cli.tsx — 入口分发器

cli.tsx 是进程入口，采用**快速路径 (fast-path)** 模式，通过动态 import 最小化模块加载：

```typescript
async function main(): Promise<void> {
  const args = process.argv.slice(2)

  // 快速路径 1: --version — 零模块加载
  if (args.length === 1 && (args[0] === '--version' || args[0] === '-v')) {
    console.log(`${MACRO.VERSION} (Claude Code)`)
    return
  }

  // 快速路径 2: --dump-system-prompt — 仅加载 config + prompts
  if (feature('DUMP_SYSTEM_PROMPT') && args[0] === '--dump-system-prompt') { ... }

  // 快速路径 3: --daemon-worker — 精简工作进程，不加载 config/analytics
  if (feature('DAEMON') && args[0] === '--daemon-worker') { ... }

  // 快速路径 4: remote-control / bridge 模式
  if (feature('BRIDGE_MODE') && args[0] === 'remote-control') { ... }

  // 快速路径 5: daemon 子命令
  if (feature('DAEMON') && args[0] === 'daemon') { ... }

  // 快速路径 6: 后台会话管理 (ps/logs/attach/kill/--bg)
  if (feature('BG_SESSIONS') && ...) { ... }

  // 默认路径：加载完整 CLI
  const { main: cliMain } = await import('../main.tsx')
  await cliMain()
}
```

**关键设计**:
- `feature()` 来自 `bun:bundle`，在构建时内联布尔值，使 Bun 可以对未启用的特性进行死代码消除 (DCE)
- `MACRO.VERSION` 在构建时内联，--version 路径实现了真正的零运行时开销
- 顶层副作用（如 `COREPACK_ENABLE_AUTO_PIN = '0'`）放在文件最开头执行

### init.ts — 初始化编排器

`init()` 使用 `memoize` 确保只执行一次：

```typescript
export const init = memoize(async (): Promise<void> => {
  // 1. 验证配置文件并启用配置系统
  enableConfigs()

  // 2. 应用安全的环境变量（信任对话框之前只应用安全变量）
  applySafeConfigEnvironmentVariables()

  // 3. 应用 CA 证书（必须在首次 TLS 握手前，Bun 启动时缓存证书存储）
  applyExtraCACertsFromConfig()

  // 4. 注册优雅关闭处理器
  setupGracefulShutdown()

  // 5. 异步初始化（不阻塞启动）
  void Promise.all([
    import('../services/analytics/firstPartyEventLogger.js'),
    import('../services/analytics/growthbook.js'),
  ]).then(([fp, gb]) => {
    fp.initialize1PEventLogging()
    gb.onGrowthBookRefresh(() => fp.reinitialize1PEventLoggingIfConfigChanged())
  })

  // 6. 检测环境与配置
  void populateOAuthAccountInfoIfNeeded()
  void initJetBrainsDetection()
  void detectCurrentRepository()

  // 7. 网络配置 (mTLS, proxy)
  configureGlobalMTLS()
  configureGlobalAgents()

  // 8. API 预连接 — 在 action-handler ~100ms 工作期间完成 TCP+TLS 握手
  preconnectAnthropicApi()

  // 9. 清理注册
  registerCleanup(shutdownLspServerManager)
  registerCleanup(async () => {
    const { cleanupSessionTeams } = await import('../utils/swarm/teamHelpers.js')
    await cleanupSessionTeams()
  })
})
```

### Telemetry 延迟初始化

遥测在信任对话框通过后才初始化，延迟加载 ~400KB 的 OpenTelemetry + protobuf 模块：

```typescript
export function initializeTelemetryAfterTrust(): void {
  if (isEligibleForRemoteManagedSettings()) {
    // 等待远程设置加载完成后再初始化遥测
    void waitForRemoteManagedSettingsToLoad()
      .then(async () => {
        applyConfigEnvironmentVariables()
        await doInitializeTelemetry()
      })
  } else {
    void doInitializeTelemetry()
  }
}

async function setMeterState(): Promise<void> {
  // 延迟导入 ~400KB 的 OpenTelemetry + protobuf
  const { initializeTelemetry } = await import('../utils/telemetry/instrumentation.js')
  const meter = await initializeTelemetry()
  // ...
}
```

---

## 3. 配置系统

**关键文件**: `src/utils/config.ts`

### 双层配置架构

Claude Code 有两个独立的配置系统：

| 层级 | 文件 | 用途 |
|------|------|------|
| **GlobalConfig** | `~/.claude.json` | 运行时状态（历史、OAuth tokens、上次会话指标） |
| **ProjectConfig** | `.claude/config.json` | 项目级配置（允许的工具、MCP 服务器、信任状态） |
| **SettingsJson** | `settings.json`（多源） | 行为配置（权限规则、钩子、模型选择、环境变量） |

### ProjectConfig 结构

```typescript
export type ProjectConfig = {
  allowedTools: string[]           // 已允许的工具列表
  mcpContextUris: string[]         // MCP 上下文 URI
  mcpServers?: Record<string, McpServerConfig>

  // 上次会话指标
  lastAPIDuration?: number
  lastCost?: number
  lastLinesAdded?: number
  lastModelUsage?: Record<string, { inputTokens, outputTokens, ... }>

  // 信任对话框
  hasTrustDialogAccepted?: boolean
  hasCompletedProjectOnboarding?: boolean

  // Worktree 会话管理
  activeWorktreeSession?: {
    originalCwd: string
    worktreePath: string
    worktreeName: string
    sessionId: string
  }
}
```

### 重入防护

配置读取有重入防护，防止 `getConfig → logEvent → getGlobalConfig → getConfig` 无限递归：

```typescript
let insideGetConfig = false

export function getGlobalConfig(): GlobalConfig {
  if (insideGetConfig) {
    return DEFAULT_GLOBAL_CONFIG  // 短路返回默认值
  }
  insideGetConfig = true
  try {
    // ... 实际读取逻辑
  } finally {
    insideGetConfig = false
  }
}
```

---

## 4. Settings 多源合并系统

**关键文件**: `src/utils/settings/settings.ts`、`src/utils/settings/types.ts`、`src/utils/settings/constants.ts`

### 五层设置源

设置从多个来源加载，后加载的覆盖先加载的：

```typescript
export const SETTING_SOURCES = [
  'userSettings',      // ~/.claude/settings.json — 全局个人设置
  'projectSettings',   // .claude/settings.json — 项目共享设置
  'localSettings',     // .claude/settings.local.json — 本地 gitignored 设置
  'flagSettings',      // --settings CLI 参数
  'policySettings',    // managed-settings.json 或远程 API — 企业管控
] as const
```

### 企业管控设置 (Managed Settings)

支持 systemd 风格的 drop-in 目录：

```typescript
export function loadManagedFileSettings(): { settings, errors } {
  // 1. 加载基础文件 managed-settings.json（最低优先级）
  const { settings } = parseSettingsFile(getManagedSettingsFilePath())

  // 2. 加载 drop-in 目录 managed-settings.d/*.json
  //    按字母排序，后文件覆盖前文件
  //    例如: 10-otel.json, 20-security.json
  const entries = readdirSync(dropInDir)
    .filter(d => d.name.endsWith('.json') && !d.name.startsWith('.'))
    .sort()

  for (const name of entries) {
    merged = mergeWith(merged, settings, settingsMergeCustomizer)
  }
}
```

### SettingsJson Schema (Zod 验证)

所有设置通过 Zod v4 schema 验证：

```typescript
export const PermissionsSchema = lazySchema(() =>
  z.object({
    allow: z.array(PermissionRuleSchema()).optional(),
    deny: z.array(PermissionRuleSchema()).optional(),
    ask: z.array(PermissionRuleSchema()).optional(),
    defaultMode: z.enum(PERMISSION_MODES).optional(),
    disableBypassPermissionsMode: z.enum(['disable']).optional(),
    additionalDirectories: z.array(z.string()).optional(),
  }).passthrough()
)

// lazySchema 模式 — 延迟创建 schema，避免模块加载时的循环依赖
export function lazySchema<T>(factory: () => T): () => T {
  let cached: T | undefined
  return () => cached ?? (cached = factory())
}
```

---

## 5. 安全存储

**关键文件**: `src/utils/secureStorage/`

### 平台适配架构

```typescript
// index.ts — 平台选择
export function getSecureStorage(): SecureStorage {
  if (process.platform === 'darwin') {
    return createFallbackStorage(macOsKeychainStorage, plainTextStorage)
  }
  return plainTextStorage  // Linux/Windows 降级
}

// fallbackStorage.ts — 自动降级包装
export function createFallbackStorage(
  primary: SecureStorage,
  fallback: SecureStorage,
): SecureStorage {
  // 尝试 primary，失败时自动切换 fallback
}
```

### macOS Keychain 集成

使用系统 `security` 命令操作钥匙串：

```typescript
export const macOsKeychainStorage = {
  read(): SecureStorageData | null {
    // TTL 缓存 — 避免频繁 spawn security 进程
    if (Date.now() - prev.cachedAt < KEYCHAIN_CACHE_TTL_MS) {
      return prev.data
    }
    // execSync 调用 security find-generic-password
    const result = execSyncWithDefaults_DEPRECATED(
      `security find-generic-password -a "${username}" -w -s "${storageServiceName}"`
    )
    // ...
  },

  // 异步读取 — 防止并发重复读取
  async readAsync(): Promise<SecureStorageData | null> {
    if (keychainCacheState.readInFlight) {
      return keychainCacheState.readInFlight  // 合并并发请求
    }
    // ...
  },

  update(data: SecureStorageData): { success: boolean; warning?: string } {
    // security find-generic-password 的 stdin 有 4096 字节限制
    // 超过限制会静默截断导致数据损坏
    const SECURITY_STDIN_LINE_LIMIT = 4096 - 64  // 64B 安全余量
    // ...
  }
}
```

**Stale-while-error 策略**：keychain 读取失败时继续使用过期缓存数据，防止单次 `security` 进程故障导致全局 "Not logged in"。

---

## 6. 权限系统

**关键文件**: `src/utils/permissions/` (24 文件)

### 权限决策流程

```
用户/模型请求工具调用
    │
    ├─ 1. 检查 deny 规则 → 命中则直接拒绝
    │
    ├─ 2. 检查 allow 规则 → 命中则直接允许
    │
    ├─ 3. 内置安全检查
    │   ├─ filesystem.ts: 危险文件/目录保护
    │   ├─ pathValidation.ts: 路径遍历检测
    │   └─ dangerousPatterns.ts: 危险命令模式
    │
    ├─ 4. Auto Mode (YOLO) 分类器
    │   └─ yoloClassifier.ts: LLM 分类器决定是否自动批准
    │
    └─ 5. 交互式提示用户
```

### 文件系统保护

```typescript
// filesystem.ts
export const DANGEROUS_FILES = [
  '.gitconfig', '.gitmodules', '.bashrc', '.bash_profile',
  '.zshrc', '.zprofile', '.profile', '.ripgreprc',
  '.mcp.json', '.claude.json',
] as const

export const DANGEROUS_DIRECTORIES = [
  '.git', '.vscode', '.idea', '.claude',
] as const

// 路径大小写归一化 — 防止 .cLauDe/Settings.locaL.json 绕过
export function normalizeCaseForComparison(path: string): string {
  return path.toLowerCase()
}
```

### 拒绝追踪 (Denial Tracking)

```typescript
// denialTracking.ts — 追踪分类器拒绝次数
export type DenialTrackingState = {
  consecutiveDenials: number  // 连续拒绝次数
  totalDenials: number        // 总拒绝次数
}

export const DENIAL_LIMITS = {
  maxConsecutive: 3,   // 连续 3 次拒绝 → 降级为交互式提示
  maxTotal: 20,        // 总共 20 次拒绝 → 降级为交互式提示
}

// 不可变更新模式
export function recordDenial(state: DenialTrackingState): DenialTrackingState {
  return {
    ...state,
    consecutiveDenials: state.consecutiveDenials + 1,
    totalDenials: state.totalDenials + 1,
  }
}

export function recordSuccess(state: DenialTrackingState): DenialTrackingState {
  if (state.consecutiveDenials === 0) return state  // 短路避免分配
  return { ...state, consecutiveDenials: 0 }
}
```

### YOLO 分类器 (Auto Mode)

```typescript
// yoloClassifier.ts
// 使用 LLM side-query 分类工具调用是否安全
export function classifyYoloAction(
  tool: Tool,
  toolInput: AnyObject,
  context: ToolPermissionContext,
  messages: Message[],
): Promise<YoloClassifierResult> {
  // 构建分类提示词 — 包含工具名、输入、CLAUDE.md 内容、权限规则
  // 调用小模型进行分类
  // 返回: allow | deny | ask (需要用户确认)
}
```

---

## 7. Bash AST 安全分析

**关键文件**: `src/utils/bash/ast.ts`、`src/utils/bash/parser.ts`、`src/utils/bash/ParsedCommand.ts`

### 设计原则 — 失败即关闭 (Fail-Closed)

```typescript
/**
 * AST-based bash command analysis using tree-sitter.
 *
 * 关键设计属性: FAIL-CLOSED — 我们从不解释不理解的结构。
 * 如果 tree-sitter 产生了我们没有显式白名单的节点类型，
 * 整个命令被分类为 'too-complex'，必须通过权限提示流程。
 *
 * 这不是沙箱。它只回答一个问题:
 * "能否为这个字符串中的每个简单命令生成可信的 argv[]？"
 */
```

### 解析结果类型

```typescript
export type ParseForSecurityResult =
  | { kind: 'simple'; commands: SimpleCommand[] }
  | { kind: 'too-complex'; reason: string; nodeType?: string }
  | { kind: 'parse-unavailable' }

export type SimpleCommand = {
  argv: string[]                     // argv[0] 是命令名
  envVars: { name: string; value: string }[]  // 前导环境变量赋值
  redirects: Redirect[]             // 重定向
  text: string                      // 原始源码片段
}
```

### 解析中止处理 (PARSE_ABORTED)

```typescript
// parser.ts

// SECURITY: 解析超时/节点数超限的哨兵值
// 恶意输入可以触发中止: (( a[0][0]... )) 约 2800 个下标就会命中超时
// 调用方必须视为 fail-closed，不能路由到 legacy 路径
export const PARSE_ABORTED = Symbol('parse-aborted')

export async function parseCommandRaw(command: string): Promise<Node | null | typeof PARSE_ABORTED> {
  if (!command || command.length > MAX_COMMAND_LENGTH) return null  // 10000 字符上限

  await ensureParserInitialized()
  const mod = getParserModule()
  if (!mod) return null

  try {
    const result = mod.parse(command)
    if (result === null) {
      // 模块已加载但解析失败 → 超时/节点预算用完
      logEvent('tengu_tree_sitter_parse_abort', { cmdLength: command.length })
      return PARSE_ABORTED
    }
    return result
  } catch {
    return PARSE_ABORTED
  }
}
```

### ParsedCommand — 双实现模式

```typescript
// ParsedCommand.ts
export const ParsedCommand = {
  async parse(command: string): Promise<IParsedCommand | null> {
    // 1. 尝试 tree-sitter（完整 AST 分析）
    if (treeSitterAvailable) {
      const data = await parseCommand(command)
      if (data) return buildParsedCommandFromRoot(command, data.rootNode)
    }
    // 2. 降级到正则 + shell-quote（已废弃但保留）
    return new RegexParsedCommand_DEPRECATED(command)
  }
}

// 单条目缓存 — 相同命令重复解析只做一次 native parse
let lastCmd: string | undefined
let lastResult: Promise<IParsedCommand | null> | undefined
```

### 安全环境变量白名单

```typescript
// ast.ts — 只有 shell 自动设置的变量才允许通过 $VAR 引用
const SAFE_ENV_VARS = new Set([
  'HOME', 'PWD', 'OLDPWD', 'USER', 'LOGNAME', 'SHELL',
  'PATH', 'HOSTNAME', 'UID', 'TMPDIR', 'BASH_VERSION', 'SHLVL',
])

// 裸引用的 $VAR 在 bash 中会经历 word-splitting 和 glob 展开
// VAR="-rf /" && rm $VAR → bash 执行 rm -rf / (两个参数)
// 但 argv 分析会得到 ['rm', '-rf /'] (一个参数) — 不安全
const BARE_VAR_UNSAFE_RE = /[ \t\n*?[]/
```

---

## 8. Git 工具库

**关键文件**: `src/utils/git/gitFilesystem.ts`、`src/utils/git/gitConfigParser.ts`

### 文件系统级 Git 状态读取

避免频繁 spawn git 子进程，直接读取 .git 文件：

```typescript
// gitFilesystem.ts

/**
 * 解析 .git 目录 — 支持 worktree/submodule 场景
 * - 正常仓库: .git 是目录
 * - Worktree/Submodule: .git 是文件，内容为 "gitdir: <path>"
 */
export async function resolveGitDir(startPath?: string): Promise<string | null> {
  const gitPath = join(root, '.git')
  const st = await stat(gitPath)
  if (st.isFile()) {
    // Worktree: .git 文件包含 gitdir 指针
    const content = (await readFile(gitPath, 'utf-8')).trim()
    if (content.startsWith('gitdir:')) {
      return resolve(root, content.slice('gitdir:'.length).trim())
    }
  }
  return gitPath  // 正常仓库
}
```

### ref 名称安全验证

```typescript
/**
 * 验证 ref/branch 名称安全性:
 * - 防止路径遍历 (..)
 * - 防止参数注入 (前导 -)
 * - 防止 shell 元字符注入 (backtick, $, ;, |, &)
 *
 * 白名单: ASCII 字母数字 + / . _ + - @
 */
export function isSafeRefName(name: string): boolean {
  if (!name || name.startsWith('-') || name.startsWith('/')) return false
  // ...允许列表检查...
}
```

---

## 9. Token 管理与上下文分析

**关键文件**: `src/utils/tokens.ts`、`src/utils/context.ts`、`src/utils/analyzeContext.ts`、`src/utils/tokenBudget.ts`

### 上下文窗口大小计算

```typescript
// context.ts
export const MODEL_CONTEXT_WINDOW_DEFAULT = 200_000

export function getContextWindowForModel(model: string, betas?: string[]): number {
  // 优先级:
  // 1. CLAUDE_CODE_MAX_CONTEXT_TOKENS 环境变量 (ant-only)
  // 2. [1m] 后缀 → 1,000,000
  // 3. 模型能力查询 → 动态值
  // 4. 默认 200,000

  if (has1mContext(model)) return 1_000_000
  const cap = getModelCapability(model)
  if (cap?.max_input_tokens >= 100_000) return cap.max_input_tokens
  return MODEL_CONTEXT_WINDOW_DEFAULT
}

// 输出 token 限制 — 有自适应升级机制
export const CAPPED_DEFAULT_MAX_TOKENS = 8_000   // 初始值，覆盖 >99% 的请求
export const ESCALATED_MAX_TOKENS = 64_000        // 命中上限后重试使用
```

### tokenCountWithEstimation — 上下文大小的权威计算

```typescript
// tokens.ts

/**
 * 计算当前上下文窗口大小（tokens）
 * 这是检查阈值（自动压缩、会话记忆等）的 **权威函数**
 *
 * 算法:
 * 1. 找到最后一个 API 响应的 usage（包含 input + output + cache tokens）
 * 2. 处理并行工具调用的消息拆分：回溯到同一 message.id 的第一个记录
 * 3. 估算该响应之后新增消息的 token 数
 */
export function tokenCountWithEstimation(messages: readonly Message[]): number {
  let i = messages.length - 1
  while (i >= 0) {
    const usage = getTokenUsage(messages[i])
    if (usage) {
      // 回溯处理并行工具调用拆分的消息
      const responseId = getAssistantMessageId(messages[i])
      if (responseId) {
        let j = i - 1
        while (j >= 0) {
          if (getAssistantMessageId(messages[j]) === responseId) i = j
          else if (getAssistantMessageId(messages[j]) !== undefined) break
          j--
        }
      }
      return getTokenCountFromUsage(usage) +
        roughTokenCountEstimationForMessages(messages.slice(i + 1))
    }
    i--
  }
  return roughTokenCountEstimationForMessages(messages)
}
```

### Token Budget — 用户指定预算

```typescript
// tokenBudget.ts — 解析用户在消息中嵌入的 token 预算

// 支持格式:
// "+500k" (消息开头或结尾)
// "use 2M tokens" / "spend 500k tokens"
const SHORTHAND_START_RE = /^\s*\+(\d+(?:\.\d+)?)\s*(k|m|b)\b/i
const VERBOSE_RE = /\b(?:use|spend)\s+(\d+(?:\.\d+)?)\s*(k|m|b)\s*tokens?\b/i

export function parseTokenBudget(text: string): number | null {
  const startMatch = text.match(SHORTHAND_START_RE)
  if (startMatch) return parseBudgetMatch(startMatch[1]!, startMatch[2]!)
  // ...
}
```

### analyzeContext — 上下文窗口可视化分析

```typescript
// analyzeContext.ts — 用于 /context 命令的详细分析

interface ContextCategory {
  name: string      // 如 "System prompt", "Tools", "Messages"
  tokens: number
  color: keyof Theme
  isDeferred?: boolean  // 延迟加载的 token 不计入使用量
}

// 计算每个组件占用的 token 数:
// - 系统提示词
// - 工具定义（内置 + MCP）
// - 消息历史
// - CLAUDE.md 文件
// - Autocompact 缓冲区
```

---

## 10. CLAUDE.md 与持久化记忆系统

**关键文件**: `src/utils/claudemd.ts`、`src/memdir/memdir.ts`、`src/memdir/paths.ts`

### CLAUDE.md 加载顺序

优先级从低到高（后加载覆盖前加载）：

```
1. /etc/claude-code/CLAUDE.md              — 企业全局指令（managed memory）
2. ~/.claude/CLAUDE.md + ~/.claude/rules/   — 用户全局指令
3. 项目 CLAUDE.md, .claude/CLAUDE.md,       — 项目级指令（版本控制内）
   .claude/rules/*.md
4. CLAUDE.local.md                          — 项目本地指令（gitignored）
```

### @include 指令系统

```typescript
// claudemd.ts
// 支持 @path、@./relative、@~/home、@/absolute 语法
// 仅在叶文本节点中工作（不在代码块内）
// 使用 marked.Lexer 解析 markdown 结构
// 有循环引用检测
// 不存在的文件静默忽略

const TEXT_FILE_EXTENSIONS = new Set([
  '.md', '.txt', '.json', '.yaml', '.yml', '.toml', '.xml',
  '.js', '.ts', '.tsx', '.py', '.go', '.rs', '.java', '.kt',
  '.c', '.cpp', '.h', '.cs', // ... 更多
])
```

### Auto Memory (memdir) 系统

```typescript
// memdir.ts
export const ENTRYPOINT_NAME = 'MEMORY.md'
export const MAX_ENTRYPOINT_LINES = 200
export const MAX_ENTRYPOINT_BYTES = 25_000

// MEMORY.md 截断策略 — 先按行截断，再按字节截断
export function truncateEntrypointContent(raw: string): EntrypointTruncation {
  const contentLines = raw.trim().split('\n')
  const wasLineTruncated = contentLines.length > MAX_ENTRYPOINT_LINES
  const wasByteTruncated = raw.trim().length > MAX_ENTRYPOINT_BYTES

  if (wasLineTruncated) {
    truncated = contentLines.slice(0, MAX_ENTRYPOINT_LINES).join('\n')
  }
  if (truncated.length > MAX_ENTRYPOINT_BYTES) {
    // 在字节限制前的最后一个换行处截断，不切断行
    const cutAt = truncated.lastIndexOf('\n', MAX_ENTRYPOINT_BYTES)
    truncated = truncated.slice(0, cutAt > 0 ? cutAt : MAX_ENTRYPOINT_BYTES)
  }
}
```

### Memory 目录路径解析

```typescript
// paths.ts
export function isAutoMemoryEnabled(): boolean {
  // 优先级链:
  // 1. CLAUDE_CODE_DISABLE_AUTO_MEMORY 环境变量
  // 2. CLAUDE_CODE_SIMPLE (--bare 模式) → 关闭
  // 3. CCR 无持久存储 → 关闭
  // 4. settings.json 中的 autoMemoryEnabled
  // 5. 默认: 启用
}

export function getMemoryBaseDir(): string {
  return process.env.CLAUDE_CODE_REMOTE_MEMORY_DIR || getClaudeConfigHomeDir()
}
```

---

## 11. 配置迁移

**关键文件**: `src/migrations/`

### 迁移模式

每个迁移文件是一个自包含函数，幂等执行：

```typescript
// migrateSonnet45ToSonnet46.ts — 典型迁移示例
export function migrateSonnet45ToSonnet46(): void {
  // 1. 前置条件检查
  if (getAPIProvider() !== 'firstParty') return
  if (!isProSubscriber() && !isMaxSubscriber()) return

  // 2. 检查当前值是否需要迁移
  const model = getSettingsForSource('userSettings')?.model
  if (model !== 'claude-sonnet-4-5-20250929' &&
      model !== 'claude-sonnet-4-5-20250929[1m]') return

  // 3. 执行迁移 — 从固定版本改为别名
  const has1m = model.endsWith('[1m]')
  updateSettingsForSource('userSettings', {
    model: has1m ? 'sonnet[1m]' : 'sonnet',
  })

  // 4. 记录迁移通知标记（跳过新用户）
  if (config.numStartups > 1) {
    saveGlobalConfig(current => ({
      ...current,
      sonnet45To46MigrationTimestamp: Date.now(),
    }))
  }

  // 5. 分析事件
  logEvent('tengu_sonnet45_to_46_migration', { from_model: model, has_1m: has1m })
}
```

迁移文件命名约定:
- `migrateFennecToOpus.ts` — 模型重命名
- `migrateLegacyOpusToCurrent.ts` — 版本更新
- `migrateAutoUpdatesToSettings.ts` — 配置结构变更
- `resetProToOpusDefault.ts` — 订阅级别变更时重置默认值

---

## 12. Zod Schema 验证

**关键文件**: `src/schemas/hooks.ts`、`src/utils/settings/types.ts`

### 钩子 Schema — 带鉴别联合

```typescript
// schemas/hooks.ts — 打破 settings/types.ts 和 plugins/schemas.ts 的循环依赖

const BashCommandHookSchema = z.object({
  type: z.literal('command'),
  command: z.string(),
  if: IfConditionSchema(),        // 权限规则语法过滤
  shell: z.enum(SHELL_TYPES).optional(),
  timeout: z.number().positive().optional(),
  statusMessage: z.string().optional(),
  once: z.boolean().optional(),    // 只执行一次
  async: z.boolean().optional(),   // 后台执行
  asyncRewake: z.boolean().optional(),  // 后台执行，exit code 2 时唤醒模型
})

const PromptHookSchema = z.object({
  type: z.literal('prompt'),
  prompt: z.string(),             // LLM 提示词，$ARGUMENTS 为占位符
  model: z.string().optional(),
})

const HttpHookSchema = z.object({
  type: z.literal('http'),
  url: z.string().url(),
})

// lazySchema 模式 — 延迟构造以打破循环依赖
export const HookCommandSchema = lazySchema(() =>
  z.discriminatedUnion('type', [BashCommandHookSchema, PromptHookSchema, HttpHookSchema])
)
```

### lazySchema 工具函数

```typescript
// 解决 Zod schema 的循环依赖问题
// Schema A 引用 Schema B，B 引用 A → 包装成惰性求值
export function lazySchema<T>(factory: () => T): () => T {
  let cached: T | undefined
  return () => cached ?? (cached = factory())
}
```

---

## 13. 取消/中止机制

**关键文件**: `src/utils/abortController.ts`

### 父子 AbortController 与内存安全

```typescript
/**
 * 创建子 AbortController：父中止时子跟随中止，子中止不影响父
 *
 * 内存安全设计:
 * 1. WeakRef 防止父持有对废弃子的强引用
 * 2. 子被 abort 时自动清理父监听器
 * 3. 支持 GC — 若子被丢弃但未 abort，仍可被回收
 */
export function createChildAbortController(
  parent: AbortController,
  maxListeners?: number,
): AbortController {
  const child = createAbortController(maxListeners)

  // 快速路径: 父已中止
  if (parent.signal.aborted) {
    child.abort(parent.signal.reason)
    return child
  }

  // WeakRef 防止内存泄漏
  const weakChild = new WeakRef(child)
  const weakParent = new WeakRef(parent)
  const handler = propagateAbort.bind(weakParent, weakChild)

  parent.signal.addEventListener('abort', handler, { once: true })

  // 自动清理: 子 abort 时移除父监听器
  child.signal.addEventListener(
    'abort',
    removeAbortHandler.bind(weakParent, new WeakRef(handler)),
    { once: true },
  )

  return child
}

// 模块级函数 — 避免每次调用分配闭包
function propagateAbort(
  this: WeakRef<AbortController>,
  weakChild: WeakRef<AbortController>,
): void {
  const parent = this.deref()
  weakChild.deref()?.abort(parent?.signal.reason)
}
```

**关键优化**: `propagateAbort` 和 `removeAbortHandler` 是模块级函数用 `.bind()` 调用，而非闭包，避免每次创建子控制器时分配新的函数对象。

---

## 14. Signal 事件原语

**关键文件**: `src/utils/signal.ts`

替代了之前散落在 ~15 处的手写 listener set 样板代码：

```typescript
export type Signal<Args extends unknown[] = []> = {
  subscribe: (listener: (...args: Args) => void) => () => void
  emit: (...args: Args) => void
  clear: () => void
}

export function createSignal<Args extends unknown[] = []>(): Signal<Args> {
  const listeners = new Set<(...args: Args) => void>()
  return {
    subscribe(listener) {
      listeners.add(listener)
      return () => { listeners.delete(listener) }  // 返回取消订阅函数
    },
    emit(...args) {
      for (const listener of listeners) listener(...args)
    },
    clear() {
      listeners.clear()
    },
  }
}

// 使用示例 (bootstrap/state.ts):
const sessionSwitched = createSignal<[id: SessionId]>()
export const onSessionSwitch = sessionSwitched.subscribe
// 内部触发: sessionSwitched.emit(sessionId)
```

**与 Store 的区别**: Signal 没有快照/getState，仅通知"某事发生了"。适用于事件通知，不适用于状态管理。

---

## 15. Startup Profiler

**关键文件**: `src/utils/startupProfiler.ts`

### 两种模式

```typescript
// 模式 1: 采样日志 — 100% 内部用户, 0.5% 外部用户
const STATSIG_SAMPLE_RATE = 0.005
const STATSIG_LOGGING_SAMPLED =
  process.env.USER_TYPE === 'ant' || Math.random() < STATSIG_SAMPLE_RATE

// 模式 2: 详细分析 — CLAUDE_CODE_PROFILE_STARTUP=1
const DETAILED_PROFILING = isEnvTruthy(process.env.CLAUDE_CODE_PROFILE_STARTUP)

// 不被采样的用户零开销
const SHOULD_PROFILE = DETAILED_PROFILING || STATSIG_LOGGING_SAMPLED

export function profileCheckpoint(name: string): void {
  if (!SHOULD_PROFILE) return  // 零开销短路

  performance.mark(name)
  if (DETAILED_PROFILING) {
    memorySnapshots.push(process.memoryUsage())
  }
}

// 预定义的阶段 — 用于 Statsig 日志
const PHASE_DEFINITIONS = {
  import_time: ['cli_entry', 'main_tsx_imports_loaded'],
  init_time: ['init_function_start', 'init_function_end'],
  settings_time: ['eagerLoadSettings_start', 'eagerLoadSettings_end'],
  total_time: ['cli_entry', 'main_after_run'],
}
```

---

## 16. Cleanup Registry

**关键文件**: `src/utils/cleanupRegistry.ts`

从 `gracefulShutdown.ts` 分离出来，避免循环依赖：

```typescript
const cleanupFunctions = new Set<() => Promise<void>>()

export function registerCleanup(cleanupFn: () => Promise<void>): () => void {
  cleanupFunctions.add(cleanupFn)
  return () => cleanupFunctions.delete(cleanupFn)  // 返回注销函数
}

export async function runCleanupFunctions(): Promise<void> {
  await Promise.all(Array.from(cleanupFunctions).map(fn => fn()))
}
```

使用场景:
```typescript
// init.ts
registerCleanup(shutdownLspServerManager)
registerCleanup(async () => {
  const { cleanupSessionTeams } = await import('../utils/swarm/teamHelpers.js')
  await cleanupSessionTeams()
})
```

---

## 17. Telemetry 遥测集成

**关键文件**: `src/utils/telemetry/` (延迟加载)、`src/bootstrap/state.ts`

### 延迟加载策略

OpenTelemetry 模块 (~400KB + gRPC ~700KB) 通过动态 import 延迟加载：

```typescript
// init.ts — 遥测仅在信任对话框通过后初始化
async function setMeterState(): Promise<void> {
  // 延迟导入: ~400KB OpenTelemetry + protobuf
  const { initializeTelemetry } = await import('../utils/telemetry/instrumentation.js')
  // gRPC 导出器进一步延迟加载 (~700KB @grpc/grpc-js)
  const meter = await initializeTelemetry()
  // ...
}
```

### AttributedCounter 模式

```typescript
// bootstrap/state.ts — 每次计数自动附加当前遥测属性
type AttributedCounter = {
  add(value: number, additionalAttributes?: Attributes): void
}

const createAttributedCounter = (name: string, options: MetricOptions): AttributedCounter => {
  const counter = meter?.createCounter(name, options)
  return {
    add(value: number, additionalAttributes: Attributes = {}) {
      // 每次 add 时获取最新属性，确保中途变化的属性被捕获
      const currentAttributes = getTelemetryAttributes()
      counter?.add(value, { ...currentAttributes, ...additionalAttributes })
    },
  }
}

// 预定义的计数器
export type State = {
  sessionCounter: AttributedCounter | null
  locCounter: AttributedCounter | null
  prCounter: AttributedCounter | null
  commitCounter: AttributedCounter | null
  costCounter: AttributedCounter | null
  tokenCounter: AttributedCounter | null
  codeEditToolDecisionCounter: AttributedCounter | null
  activeTimeCounter: AttributedCounter | null
}
```

---

## 18. Thinking 模式支持

**关键文件**: `src/utils/thinking.ts`

### 配置类型

```typescript
export type ThinkingConfig =
  | { type: 'adaptive' }             // 自适应思考（模型决定）
  | { type: 'enabled'; budgetTokens: number }  // 固定预算
  | { type: 'disabled' }             // 禁用
```

### 模型能力检测

```typescript
// 提供商感知的思考能力检测
export function modelSupportsThinking(model: string): boolean {
  // 1. 检查第三方模型覆盖
  const supported3P = get3PModelCapabilityOverride(model, 'thinking')
  if (supported3P !== undefined) return supported3P

  // 2. 1P 和 Foundry: 所有 Claude 4+ 模型
  if (provider === 'firstParty' || provider === 'foundry') {
    return !canonical.includes('claude-3-')
  }

  // 3. 3P (Bedrock/Vertex): 仅 Opus 4+ 和 Sonnet 4+
  return canonical.includes('sonnet-4') || canonical.includes('opus-4')
}

export function modelSupportsAdaptiveThinking(model: string): boolean {
  // 仅 4.6 版本模型支持自适应思考
  if (canonical.includes('opus-4-6') || canonical.includes('sonnet-4-6')) return true
  // 1P/Foundry 对未知模型默认启用（新模型都训练了自适应思考）
  return provider === 'firstParty' || provider === 'foundry'
}
```

### Ultrathink

```typescript
// 构建时 feature gate + 运行时 GrowthBook gate
export function isUltrathinkEnabled(): boolean {
  if (!feature('ULTRATHINK')) return false  // 外部构建完全消除
  return getFeatureValue_CACHED_MAY_BE_STALE('tengu_turtle_carbon', true)
}

export function hasUltrathinkKeyword(text: string): boolean {
  return /\bultrathink\b/i.test(text)
}
```

---

## 19. 可复用的基础设施模式

### 模式 1: 叶模块隔离 (Leaf Module Isolation)

**用途**: 防止循环依赖
**实现**: 禁止从项目内部导入，只允许外部包和 type-only 导入
**应用**: `bootstrap/state.ts`、`cleanupRegistry.ts`

```typescript
// ESLint 自定义规则强制执行
// custom-rules/bootstrap-isolation
```

### 模式 2: 延迟导入 (Lazy Import)

**用途**: 减少启动时间，延迟加载大型依赖
**实现**: `await import()` 或 `require()` 在运行时条件分支中

```typescript
// 400KB OpenTelemetry 模块仅在遥测初始化时加载
const { initializeTelemetry } = await import('../utils/telemetry/instrumentation.js')

// feature gate + require 组合
const teamMemPaths = feature('TEAMMEM')
  ? (require('../memdir/teamMemPaths.js') as typeof import('../memdir/teamMemPaths.js'))
  : null
```

### 模式 3: 构建时特性开关 (Build-time Feature Flags)

**用途**: 外部构建中完全消除代码（DCE）
**实现**: `bun:bundle` 的 `feature()` 函数

```typescript
// feature() 在构建时内联为 true/false
// Bun 可以对 false 分支做死代码消除
if (feature('TREE_SITTER_BASH')) {
  // 仅在启用时包含在构建中
}
```

### 模式 4: 失败即关闭 (Fail-Closed)

**用途**: 安全关键路径
**实现**: 白名单 + 未知类型拒绝

```typescript
// Bash AST 分析: 未知节点类型 → too-complex → 需要用户确认
// Keychain 读取: 解析失败 → stale-while-error（不返回 null）
// 配置解析: 无效配置 → ConfigParseError → 显示修复对话框
```

### 模式 5: 粘性开关 (Latching)

**用途**: 避免中途切换破坏缓存
**实现**: 一旦启用永不关闭

```typescript
// bootstrap/state.ts
afkModeHeaderLatched: boolean | null    // null = 未触发
fastModeHeaderLatched: boolean | null   // 首次启用后粘住
cacheEditingHeaderLatched: boolean | null
```

### 模式 6: 不可变状态更新

**用途**: 状态追踪无副作用
**实现**: 返回新对象而非修改现有对象

```typescript
// denialTracking.ts
export function recordDenial(state: DenialTrackingState): DenialTrackingState {
  return {
    ...state,
    consecutiveDenials: state.consecutiveDenials + 1,
    totalDenials: state.totalDenials + 1,
  }
}

export function recordSuccess(state: DenialTrackingState): DenialTrackingState {
  if (state.consecutiveDenials === 0) return state  // 无变化短路
  return { ...state, consecutiveDenials: 0 }
}
```

### 模式 7: 单条目缓存 (Single-Entry Cache)

**用途**: 避免重复昂贵操作，同时防止内存泄漏
**实现**: 只缓存最近一个结果

```typescript
// ParsedCommand.ts
let lastCmd: string | undefined
let lastResult: Promise<IParsedCommand | null> | undefined

export const ParsedCommand = {
  parse(command: string): Promise<IParsedCommand | null> {
    if (command === lastCmd && lastResult !== undefined) return lastResult
    lastCmd = command
    lastResult = doParse(command)
    return lastResult
  },
}
```

### 模式 8: WeakRef 内存管理

**用途**: 父子关系中防止强引用导致内存泄漏
**实现**: `WeakRef` + `{once: true}` 事件监听

```typescript
// abortController.ts
const weakChild = new WeakRef(child)
const weakParent = new WeakRef(parent)
parent.signal.addEventListener('abort', handler, { once: true })
```

### 模式 9: 重入防护 (Re-entrancy Guard)

**用途**: 防止递归调用无限循环
**实现**: 布尔标志 + 短路返回

```typescript
// config.ts
let insideGetConfig = false
export function getGlobalConfig(): GlobalConfig {
  if (insideGetConfig) return DEFAULT_GLOBAL_CONFIG
  insideGetConfig = true
  try { ... } finally { insideGetConfig = false }
}
```

### 模式 10: Stale-While-Error

**用途**: 暂时性故障时保持可用性
**实现**: 失败时继续使用过期缓存数据

```typescript
// macOsKeychainStorage.ts
if (prev.data !== null) {
  logForDebugging('[keychain] read failed; serving stale cache', { level: 'warn' })
  keychainCacheState.cache = { data: prev.data, cachedAt: Date.now() }
  return prev.data
}
```

---

## 架构总览图

```
┌─────────────────────────────────────────────────┐
│                  cli.tsx (入口)                    │
│  快速路径: --version, --daemon-worker, bridge     │
│  默认路径: → main.tsx                             │
└──────────────────────┬──────────────────────────┘
                       │
              ┌────────▼────────┐
              │    init.ts      │
              │  (memoized)     │
              │  配置 → 网络 →  │
              │  预连接 → 清理  │
              └────────┬────────┘
                       │
    ┌──────────────────┼──────────────────┐
    │                  │                  │
┌───▼───┐    ┌────────▼────────┐   ┌─────▼──────┐
│bootstrap│   │   Settings      │   │  Security   │
│state.ts │   │  多源合并       │   │  Keychain   │
│(叶模块) │   │  Zod 验证      │   │  Permissions│
│ 100+字段│   │  企业管控      │   │  Bash AST   │
└────┬────┘   └────────┬────────┘   └─────┬──────┘
     │                 │                   │
     │     ┌───────────┼───────────┐       │
     │     │           │           │       │
  ┌──▼─────▼──┐  ┌─────▼────┐  ┌──▼───────▼──┐
  │  Tokens   │  │ CLAUDE.md │  │  Git Utils  │
  │  Context  │  │  Memory   │  │  Filesystem │
  │  Budget   │  │  memdir   │  │  Worktree   │
  └───────────┘  └──────────┘   └─────────────┘
```
