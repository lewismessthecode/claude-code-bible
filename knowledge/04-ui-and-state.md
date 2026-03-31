# 04 - UI 与状态管理系统

## 目录

1. [终端 UI 架构总览](#1-终端-ui-架构总览)
2. [自定义 Ink 渲染引擎](#2-自定义-ink-渲染引擎)
3. [状态管理系统](#3-状态管理系统)
4. [REPL 屏幕架构](#4-repl-屏幕架构)
5. [组件体系与设计系统](#5-组件体系与设计系统)
6. [权限 UI 流程](#6-权限-ui-流程)
7. [消息渲染系统](#7-消息渲染系统)
8. [Spinner 与进度显示](#8-spinner-与进度显示)
9. [键绑定系统](#9-键绑定系统)
10. [Vim 模式实现](#10-vim-模式实现)
11. [主题与输出样式系统](#11-主题与输出样式系统)
12. [虚拟滚动与全屏布局](#12-虚拟滚动与全屏布局)
13. [可复用架构模式](#13-可复用架构模式)

---

## 1. 终端 UI 架构总览

### 技术栈层次

Claude Code 的终端 UI 基于 **React + 自定义 Ink fork** 构建,实现了浏览器级别的组件化终端界面:

```
┌─────────────────────────────────────────────────────┐
│  React 19 (Concurrent Mode)                          │
│    └─ react-reconciler (ConcurrentRoot)              │
│        └─ 自定义 Ink DOM (ink-box, ink-text, ...)    │
│            └─ Yoga 布局引擎 (Flexbox)                │
│                └─ Screen Buffer → ANSI Diff → TTY    │
└─────────────────────────────────────────────────────┘
```

### 关键文件索引

| 层级 | 文件 | 职责 |
|------|------|------|
| 入口 | `ink/root.ts` | 创建 Ink 实例,挂载 React 树 |
| 协调器 | `ink/reconciler.ts` | React Reconciler 宿主配置 |
| 渲染器 | `ink/renderer.ts` | Yoga 布局 → Screen Buffer |
| 核心类 | `ink/ink.tsx` (246K!) | Ink 主类,帧调度、输入、选择 |
| DOM | `ink/dom.ts` | 虚拟 DOM 节点定义 |
| 终端写入 | `ink/log-update.ts` | ANSI diff 输出到 stdout |
| 屏幕 | `ink/screen.ts` | 双缓冲 Screen Buffer |
| 选择 | `ink/selection.ts` | 鼠标文本选择(alt screen) |
| 事件 | `ink/events/dispatcher.ts` | DOM 事件捕获/冒泡 |
| 焦点 | `ink/focus.ts` | FocusManager(类 DOM) |

### 整体渲染流水线

```
stdin 输入
  → parse-keypress.ts 解析为 ParsedKey
  → InputEvent 创建
  → Dispatcher.dispatchDiscrete() 发送到 React 树
  → React 更新状态 → Reconciler commit
  → resetAfterCommit → onComputeLayout (Yoga)
  → onRender → renderer.ts 生成 Screen
  → log-update.ts ANSI diff 写入 stdout
```

---

## 2. 自定义 Ink 渲染引擎

### 为什么 fork Ink

Claude Code 没有使用原版 Ink,而是完全 fork 并深度定制了整个渲染引擎。主要改造包括:

1. **升级到 React 19 + ConcurrentRoot** — 原版 Ink 使用 LegacyRoot
2. **自定义事件系统** — 实现了类 DOM 的捕获/冒泡事件分发
3. **alt-screen 支持** — 全屏终端模式 + 鼠标跟踪
4. **双缓冲 + ANSI diff** — 增量渲染避免闪烁
5. **文本选择系统** — 鼠标拖选 + 复制
6. **虚拟滚动** — 大量消息的高性能滚动
7. **搜索高亮** — 终端内全文搜索

### React Reconciler 宿主配置

`ink/reconciler.ts` 是 React 与终端的桥梁,关键实现:

```typescript
// 节点类型映射
type ElementNames =
  | 'ink-root'    // 根节点
  | 'ink-box'     // Flexbox 容器 (<Box>)
  | 'ink-text'    // 文本节点 (<Text>)
  | 'ink-virtual-text'  // Text 内嵌套的 Text
  | 'ink-link'    // 超链接
  | 'ink-progress' // 进度条
  | 'ink-raw-ansi' // 原始 ANSI 序列

// 创建实例
createInstance(type, props, root, hostContext, fiber) {
  // Text 不能包含 Box
  if (hostContext.isInsideText && type === 'ink-box') {
    throw new Error(`<Box> can't be nested inside <Text>`)
  }
  // Text 嵌套自动转换为 virtual-text
  const nodeType = type === 'ink-text' && hostContext.isInsideText
    ? 'ink-virtual-text' : type
  const node = createNode(nodeType)
  // 应用 props(style, event handlers, attributes)
  for (const [key, value] of Object.entries(props)) {
    applyProp(node, key, value)
  }
  return node
}

// React 19 commitUpdate — 接收 old/new props 而非 updatePayload
commitUpdate(node, type, oldProps, newProps) {
  const props = diff(oldProps, newProps)
  const style = diff(oldProps.style, newProps.style)
  // 增量更新属性和样式
}

// 提交后触发布局和渲染
resetAfterCommit(rootNode) {
  rootNode.onComputeLayout?.()  // Yoga 布局
  rootNode.onRender?.()         // 渲染到终端
}
```

### DOM 节点结构

```typescript
type DOMElement = {
  nodeName: ElementNames
  attributes: Record<string, DOMNodeAttribute>
  childNodes: DOMNode[]
  parentNode: DOMElement | undefined
  yogaNode?: LayoutNode        // Yoga 布局节点
  style: Styles                // Flexbox 样式
  dirty: boolean               // 需要重新渲染
  isHidden?: boolean           // 隐藏/显示
  _eventHandlers?: Record<string, unknown>  // 事件处理器

  // 滚动状态(overflow: scroll)
  scrollTop?: number
  pendingScrollDelta?: number  // 平滑滚动累积
  scrollHeight?: number
  stickyScroll?: boolean       // 自动跟随底部
  scrollAnchor?: { el: DOMElement; offset: number }  // 锚点滚动

  // 焦点管理(仅 root)
  focusManager?: FocusManager
}
```

### 事件系统

仿浏览器的 W3C 事件模型:

```typescript
// ink/events/dispatcher.ts
class Dispatcher {
  // 捕获 → 冒泡双阶段分发
  dispatch(target, event) {
    event._setTarget(target)
    const listeners = collectListeners(target, event)
    processDispatchQueue(listeners, event)
    return !event.defaultPrevented
  }

  // 离散事件(键盘、点击)— 同步高优先级
  dispatchDiscrete(target, event) {
    return this.discreteUpdates(
      (t, e) => this.dispatch(t, e),
      target, event, undefined, undefined
    )
  }

  // 连续事件(滚动、resize)— 可中断
  dispatchContinuous(target, event) { ... }
}

// 事件优先级映射(镜像 react-dom)
function getEventPriority(eventType) {
  switch (eventType) {
    case 'keydown': case 'click': case 'focus':
      return DiscreteEventPriority   // 同步刷新
    case 'resize': case 'scroll':
      return ContinuousEventPriority // 可批处理
    default:
      return DefaultEventPriority
  }
}
```

### 焦点管理

类 DOM 的 FocusManager,支持 tab 导航和焦点栈:

```typescript
class FocusManager {
  activeElement: DOMElement | null = null
  private focusStack: DOMElement[] = []  // 恢复栈

  focus(node) {
    if (node === this.activeElement) return
    const previous = this.activeElement
    if (previous) {
      this.focusStack.push(previous)  // 入栈以便恢复
      this.dispatchFocusEvent(previous, new FocusEvent('blur', node))
    }
    this.activeElement = node
    this.dispatchFocusEvent(node, new FocusEvent('focus', previous))
  }

  // 节点移除时自动恢复焦点到栈顶
  handleNodeRemoved(node, root) {
    this.focusStack = this.focusStack.filter(n => isInTree(n, root))
    while (this.focusStack.length > 0) {
      const candidate = this.focusStack.pop()!
      if (isInTree(candidate, root)) {
        this.activeElement = candidate
        return
      }
    }
  }
}
```

### 渲染器与双缓冲

```typescript
// ink/renderer.ts
function createRenderer(node, stylePool) {
  let output: Output | undefined  // 复用 Output 对象(charCache 持久化)

  return (options) => {
    const { frontFrame, backFrame, terminalWidth, terminalRows } = options
    const width = Math.floor(node.yogaNode.getComputedWidth())
    // alt-screen: 钳制高度到 terminalRows
    const height = options.altScreen ? terminalRows : yogaHeight

    // 复用或创建 Output
    if (output) output.reset(width, height, screen)
    else output = new Output({ width, height, stylePool, screen })

    // 渲染 DOM 树到 Output,利用 prevScreen 做 blit 优化
    renderNodeToOutput(node, output, {
      prevScreen: options.prevFrameContaminated ? undefined : prevScreen
    })

    return {
      screen: output.get(),
      viewport: { width: terminalWidth, height: terminalRows },
      cursor: { x: 0, y: screen.height, visible: !isTTY || height === 0 }
    }
  }
}
```

### Ink 主类(帧调度)

`ink/ink.tsx` (246K) 是整个渲染引擎的核心,管理:

- **帧调度** — throttle 渲染,16ms 帧间隔
- **双缓冲** — frontFrame/backFrame 交换
- **输入处理** — stdin → keypress 解析 → 事件分发
- **文本选择** — 鼠标事件处理 + selection overlay
- **搜索高亮** — 全屏搜索匹配渲染
- **alt-screen 管理** — 进入/退出备用屏幕
- **console 拦截** — patch console.log 避免输出混乱

---

## 3. 状态管理系统

### 自制轻量 Store

不依赖任何状态管理库,实现了一个 35 行的 Zustand-like store:

```typescript
// state/store.ts
export function createStore<T>(initialState, onChange?) {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    getState: () => state,
    setState: (updater) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return   // 引用相等跳过
      state = next
      onChange?.({ newState: next, oldState: prev })  // 变更钩子
      for (const listener of listeners) listener()    // 通知订阅者
    },
    subscribe: (listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    }
  }
}
```

### AppState — 全局状态定义

`state/AppStateStore.ts` 定义了 `AppState` 类型,是整个应用的状态枢纽。核心字段分类:

```typescript
export type AppState = DeepImmutable<{
  // === 会话设置 ===
  settings: SettingsJson
  verbose: boolean
  mainLoopModel: ModelSetting
  fastMode?: boolean
  effortValue?: EffortValue
  thinkingEnabled: boolean | undefined

  // === UI 显示状态 ===
  expandedView: 'none' | 'tasks' | 'teammates'
  isBriefOnly: boolean
  statusLineText: string | undefined
  spinnerTip?: string
  footerSelection: FooterItem | null
  activeOverlays: ReadonlySet<string>

  // === 权限系统 ===
  toolPermissionContext: ToolPermissionContext

  // === 远程/Bridge ===
  remoteSessionUrl: string | undefined
  remoteConnectionStatus: 'connecting' | 'connected' | ...
  replBridgeEnabled: boolean
  replBridgeConnected: boolean

  // === 推测执行 ===
  speculation: SpeculationState
  promptSuggestion: { text, promptId, shownAt, ... }

  // === 初始消息 ===
  initialMessage: { message, clearContext?, mode? } | null
}> & {
  // === 可变状态(排除 DeepImmutable) ===
  tasks: { [taskId: string]: TaskState }
  agentNameRegistry: Map<string, AgentId>
  foregroundedTaskId?: string
  viewingAgentTaskId?: string

  // === MCP ===
  mcp: { clients, tools, commands, resources }

  // === 插件 ===
  plugins: { enabled, disabled, commands, errors }

  // === 团队/Swarm ===
  teamContext?: { teamName, teammates, ... }
  inbox: { messages: Array<...> }

  // === 文件历史/归因 ===
  fileHistory: FileHistoryState
  attribution: AttributionState
}
```

### React Context 集成

```typescript
// state/AppState.tsx
export const AppStoreContext = React.createContext<AppStateStore | null>(null)

// Provider 包裹整个应用
function AppStateProvider({ children, initialState, onChangeAppState }) {
  const [store] = useState(
    () => createStore(initialState ?? getDefaultAppState(), onChangeAppState)
  )
  // 设置变更监听
  useSettingsChange(source => applySettingsChange(source, store.setState))

  return (
    <AppStoreContext.Provider value={store}>
      <HasAppStateContext.Provider value={true}>
        <MailboxProvider>
          <VoiceProvider>{children}</VoiceProvider>
        </MailboxProvider>
      </HasAppStateContext.Provider>
    </AppStoreContext.Provider>
  )
}

// 消费状态的 Hook — 带 selector 的 useSyncExternalStore
export function useAppState<T>(selector: (s: AppState) => T): T {
  const store = useContext(AppStoreContext)!
  return useSyncExternalStore(
    store.subscribe,
    () => selector(store.getState())
  )
}

// 获取 setState
export function useSetAppState() {
  return useContext(AppStoreContext)!.setState
}
```

### 状态变更副作用

`state/onChangeAppState.ts` 集中处理状态变更的副作用:

```typescript
function onChangeAppState({ newState, oldState }) {
  // 1. 权限模式变更 → 同步到 CCR/SDK
  if (prevMode !== newMode) {
    notifySessionMetadataChanged({ permission_mode: newExternal })
    notifyPermissionModeChanged(newMode)
  }

  // 2. 模型变更 → 保存到设置文件
  if (newState.mainLoopModel !== oldState.mainLoopModel) {
    updateSettingsForSource('userSettings', { model: newState.mainLoopModel })
    setMainLoopModelOverride(newState.mainLoopModel)
  }

  // 3. expandedView 变更 → 持久化到 globalConfig
  if (newState.expandedView !== oldState.expandedView) {
    saveGlobalConfig(current => ({
      ...current,
      showExpandedTodos: newState.expandedView === 'tasks',
      showSpinnerTree: newState.expandedView === 'teammates',
    }))
  }

  // 4. 设置变更 → 清除认证缓存
  if (newState.settings !== oldState.settings) {
    clearApiKeyHelperCache()
    clearAwsCredentialsCache()
  }
}
```

### Selectors

纯函数的状态派生,保持关注点分离:

```typescript
// 获取当前正在查看的队友任务
function getViewedTeammateTask(appState) {
  const task = appState.tasks[appState.viewingAgentTaskId]
  return isInProcessTeammateTask(task) ? task : undefined
}

// 判断用户输入应路由到哪个 agent
function getActiveAgentForInput(appState): ActiveAgentForInput {
  const viewedTask = getViewedTeammateTask(appState)
  if (viewedTask) return { type: 'viewed', task: viewedTask }
  return { type: 'leader' }
}
```

---

## 4. REPL 屏幕架构

### REPL.tsx — 874K 的超级组件

`screens/REPL.tsx` 是整个应用的主界面,集成了几乎所有功能模块。它是一个巨大的 React 函数组件(约 12000 行编译后代码),组织为:

### 组件层次结构

```
<REPL>
  <KeybindingSetup>              // 键绑定系统初始化
    <AlternateScreen>            // alt-screen 全屏模式
      <FullscreenLayout>         // 全屏布局(ScrollBox + 状态栏)
        <ScrollBox stickyScroll> // 可滚动主内容区
          <VirtualMessageList>   // 虚拟滚动消息列表
            <Messages>           // 消息渲染(递归)
          </VirtualMessageList>
        </ScrollBox>
        <StatusLine>             // 底部状态栏
        <PromptInput>            // 用户输入框
      </FullscreenLayout>
    </AlternateScreen>

    // 叠加层(对话框、选择器)
    <PermissionRequest>          // 权限确认对话框
    <ModelPicker>                // 模型选择器
    <ThemePicker>                // 主题选择器
    <MessageSelector>            // 消息重播选择器
    <GlobalSearchDialog>         // 全局搜索
    <ExportDialog>               // 会话导出
    <FeedbackSurvey>             // 反馈调查
    // ... 更多对话框
  </KeybindingSetup>
</REPL>
```

### REPL 的核心职责

1. **消息状态管理** — `messages` 数组是对话的核心数据
2. **查询循环** — 接收用户输入 → 调用 `query()` → 流式更新消息
3. **工具权限** — `useCanUseTool` hook 决定是否弹出权限对话框
4. **会话生命周期** — 加载/恢复/导出/分叉会话
5. **MCP 集成** — 管理 MCP 服务器连接和工具
6. **快捷键** — 全局键绑定处理
7. **命令队列** — 处理 `/command` 和排队消息

### 查询循环（核心交互流程）

```
用户输入 → handleSubmit()
  → 创建 UserMessage
  → addToHistory()
  → query({ messages, tools, onMessage, ... })
    → 流式回调 handleMessageFromStream()
      → setMessages(prev => [...prev, newMessage])
      → 工具调用 → useCanUseTool → 权限检查
        → 允许 → 执行工具 → 结果追加到消息
        → 拒绝 → 拒绝消息追加
    → 完成 → 记录分析 → 保存会话
```

### 三个屏幕组件

| 屏幕 | 文件 | 大小 | 用途 |
|------|------|------|------|
| REPL | `screens/REPL.tsx` | 874K | 主交互循环 |
| Doctor | `screens/Doctor.tsx` | 71K | 环境诊断(/doctor) |
| ResumeConversation | `screens/ResumeConversation.tsx` | 58K | 会话恢复(--resume) |

---

## 5. 组件体系与设计系统

### 设计系统组件

`components/design-system/` 提供基础 UI 原语:

| 组件 | 用途 |
|------|------|
| `ThemedBox` | 带主题色的 Box |
| `ThemedText` | 带主题色的 Text |
| `ThemeProvider` | 主题上下文提供者 |
| `Dialog` | 模态对话框(标题 + 内容 + 按钮) |
| `Divider` | 水平分隔线 |
| `FuzzyPicker` | 模糊搜索选择器 |
| `ListItem` | 列表项(带缩进/图标) |
| `Tabs` | 标签页切换 |
| `Pane` | 面板容器 |
| `ProgressBar` | 进度条 |
| `StatusIcon` | 状态图标(成功/失败/等待) |
| `KeyboardShortcutHint` | 快捷键提示文本 |
| `LoadingState` | 加载状态占位 |
| `Byline` | 模型/Agent 署名行 |

### Ink 基础组件

`ink/components/` 是 Ink 框架级组件:

| 组件 | 用途 |
|------|------|
| `Box` | Flexbox 容器(对应 `<div>`) |
| `Text` | 文本渲染(对应 `<span>`) |
| `ScrollBox` | 可滚动容器(虚拟视口) |
| `AlternateScreen` | 终端备用屏幕包裹器 |
| `Button` | 可点击按钮 |
| `Link` | 终端超链接(OSC 8) |
| `Spacer` | Flex 空间填充 |
| `RawAnsi` | 原始 ANSI 序列注入 |
| `App` | 顶层 Ink 应用包裹器 |

### 组件组织模式

```
components/
├── design-system/     # 原子级设计系统组件
├── permissions/       # 权限对话框(每个工具一个子目录)
├── messages/          # 消息类型渲染组件
├── diff/              # Diff 显示组件
├── StructuredDiff/    # 结构化 Diff
├── tasks/             # 任务显示
├── teams/             # 团队显示
├── Spinner/           # Spinner 子组件
├── PromptInput/       # 用户输入框组件
├── settings/          # 设置面板
├── mcp/               # MCP 相关 UI
├── grove/             # 文件树浏览
├── skills/            # 技能 UI
├── ui/                # 通用 UI 组件
└── *.tsx              # 顶级功能组件
```

---

## 6. 权限 UI 流程

### 权限请求的数据流

```
AI 调用工具
  → useCanUseTool hook
    → hasPermissionsToUseTool() 检查静态规则
      → allow → 直接执行
      → deny → 返回拒绝消息
      → ask → 进入交互流程

交互流程:
  → handleInteractivePermission()
    → 入队: setToolUseConfirmQueue([...queue, item])
    → <PermissionRequest> 组件渲染
    → 用户操作(y/n/Always allow/...)
    → resolve(decision)
    → 可能: persistPermissionUpdate() 保存规则
```

### PermissionRequest 组件路由

根据工具类型选择不同的权限 UI 组件:

```typescript
function permissionComponentForTool(tool) {
  switch (tool) {
    case BashTool:      return BashPermissionRequest
    case FileEditTool:  return FileEditPermissionRequest
    case FileWriteTool: return FileWritePermissionRequest
    case WebFetchTool:  return WebFetchPermissionRequest
    case GlobTool:
    case GrepTool:
    case FileReadTool:  return FilesystemPermissionRequest
    case SkillTool:     return SkillPermissionRequest
    default:            return FallbackPermissionRequest
  }
}
```

### 权限对话框结构

每个权限请求组件遵循统一模式:

```
┌─ PermissionPrompt ─────────────────────────────────┐
│  PermissionRequestTitle (工具名 + 描述)              │
│  ───────────────────────────────────────────────     │
│  [具体权限内容]                                      │
│    - BashPermissionRequest: 显示命令内容              │
│    - FileEditPermissionRequest: 显示文件差异          │
│    - FilesystemPermissionRequest: 显示文件路径         │
│  ───────────────────────────────────────────────     │
│  PermissionExplanation (展开/折叠的规则解释)           │
│  ───────────────────────────────────────────────     │
│  [操作按钮] y=允许 n=拒绝 a=始终允许 ...              │
└─────────────────────────────────────────────────────┘
```

---

## 7. 消息渲染系统

### 消息类型

消息是 REPL 的核心数据,每种类型有独立的渲染组件:

| 消息角色 | 子类型 | 渲染组件 |
|----------|--------|----------|
| user | 文本输入 | `UserTextMessage` |
| assistant | 文本回复 | `Markdown` 组件 |
| assistant | 工具调用 | `ToolUseLoader` / 工具专用组件 |
| assistant | 思考 | 折叠的思考块 |
| system | 系统消息 | `CompactSummary` |
| progress | 进度更新 | 内联进度指示 |

### Messages 组件（消息列表）

`components/Messages.tsx` (144K) 是消息列表的核心渲染器:

- 遍历 `messages` 数组,为每条消息选择合适的渲染组件
- 处理消息折叠/展开(verbose 模式)
- 工具调用配对(tool_use → tool_result)
- 对话压缩边界显示

### VirtualMessageList（虚拟滚动）

`components/VirtualMessageList.tsx` (145K) 实现了虚拟滚动:

```typescript
type JumpHandle = {
  jumpToIndex: (i: number) => void      // 跳转到消息
  setSearchQuery: (q: string) => void   // 设置搜索
  nextMatch: () => void                  // 下一个匹配
  prevMatch: () => void                  // 上一个匹配
  warmSearchIndex: () => Promise<number> // 预热搜索索引
  disarmSearch: () => void               // 清除搜索位置
}
```

关键设计:
- **高度缓存** — WeakMap 缓存每条消息的渲染高度
- **窗口化** — 只渲染视口内 + 上下各一屏的消息
- **Spacer** — 未渲染区域用固定高度 Box 占位
- **搜索** — 全文搜索使用缓存的纯文本索引

### Markdown 渲染

`components/Markdown.tsx` (27K) 将 Markdown 渲染到终端:

- 使用 `marked` 解析 Markdown AST
- 代码块使用 `HighlightedCode` 做语法高亮
- 表格使用 `MarkdownTable` 做终端表格渲染
- 链接使用 OSC 8 终端超链接

---

## 8. Spinner 与进度显示

### Spinner 架构

`components/Spinner.tsx` (85K) 实现了复杂的加载状态显示:

```
┌─ SpinnerWithVerb ────────────────────────────────────────┐
│  ◐ Thinking... (3s)  │  claude-sonnet-4-20250514  [tokens] │
│  ├─ Task 1: Reading file ✓                                │
│  ├─ Task 2: Writing code ⟳                                │
│  └─ Agent: researcher - Running tests                      │
│                                                            │
│  [Latest response text preview...]                         │
└──────────────────────────────────────────────────────────┘
```

### Spinner 模式

```typescript
type SpinnerMode =
  | 'streaming'   // 流式输出中
  | 'thinking'    // 思考中(扩展思考)
  | 'tool'        // 工具执行中
  | 'waiting'     // 等待 API 响应
  | 'paused'      // 暂停(权限等待)
```

### 动画系统

使用 `useAnimationFrame(50)` 驱动 50ms 一帧的动画:
- **旋转字符** — `SPINNER_FRAMES` 回弹动画
- **Shimmer 效果** — 文本闪烁(Bridge 状态指示)
- **Glimmer 效果** — 颜色渐变
- **停滞检测** — 超时变红色警告

### Brief 模式

Kairos/Brief 模式下使用简化的 Spinner:

```typescript
function BriefSpinner({ mode, overrideMessage }) {
  // 只显示 "●" + 简短状态
  const isBriefOnly = useAppState(s => s.isBriefOnly)
  if (isBriefOnly) return <BriefIdleStatus mode={mode} />
}
```

---

## 9. 键绑定系统

### 架构层次

```
keybindings/
├── defaultBindings.ts        # 默认键绑定定义
├── loadUserBindings.ts       # 从 ~/.claude/keybindings.json 加载
├── schema.ts                 # 键绑定 JSON schema
├── parser.ts                 # 解析 "ctrl+shift+k" 为结构化数据
├── match.ts                  # 匹配输入与绑定
├── resolver.ts               # 解析引擎(含 chord 支持)
├── KeybindingContext.tsx      # React Context 提供者
├── KeybindingProviderSetup.tsx # 初始化和 chord 拦截器
├── useKeybinding.ts          # 消费 hook(单个绑定)
├── validate.ts               # 用户绑定验证
├── reservedShortcuts.ts      # 不可重绑定的快捷键
└── shortcutFormat.ts         # 显示格式化
```

### 绑定定义

每个绑定属于一个 Context(作用域),Context 决定优先级:

```typescript
const DEFAULT_BINDINGS: KeybindingBlock[] = [
  {
    context: 'Global',          // 全局
    bindings: {
      'ctrl+c': 'app:interrupt',
      'ctrl+d': 'app:exit',
      'ctrl+l': 'app:redraw',
      'ctrl+t': 'app:toggleTodos',
      'ctrl+o': 'app:toggleTranscript',
      'ctrl+r': 'history:search',
    }
  },
  {
    context: 'Chat',            // 聊天输入模式
    bindings: {
      'escape': 'chat:cancel',
      'shift+tab': 'chat:cycleMode',    // 切换权限模式
      'meta+p': 'chat:modelPicker',     // 模型选择
      'enter': 'chat:submit',
      'up': 'history:previous',
      'ctrl+x ctrl+e': 'chat:externalEditor',  // chord 序列!
      'ctrl+x ctrl+k': 'chat:killAgents',
    }
  },
  {
    context: 'Scroll',          // 滚动模式
    bindings: {
      'pageup': 'scroll:pageUp',
      'wheelup': 'scroll:lineUp',
      'ctrl+shift+c': 'selection:copy',
    }
  },
  // ... Confirmation, Settings, Transcript, etc.
]
```

### Chord 支持

支持 Emacs 风格的多键序列(chord):

```typescript
// 用户按 ctrl+x → 进入 chord pending 状态
// 显示 "ctrl+x ..." 提示
// 用户按 ctrl+e → 匹配 'ctrl+x ctrl+e' → 触发 'chat:externalEditor'
// 用户按其他键 → chord 取消

type ChordResolveResult =
  | { type: 'match'; action: string }         // 完成匹配
  | { type: 'chord_started'; pending: ... }   // chord 开始
  | { type: 'chord_cancelled' }               // chord 取消
  | { type: 'unbound' }                       // 显式解绑
  | { type: 'none' }                          // 无匹配
```

### useKeybinding Hook

组件使用 `useKeybinding` 注册动作处理器:

```typescript
// 单个绑定
useKeybinding('app:toggleTodos', () => {
  setShowTodos(prev => !prev)
}, { context: 'Global' })

// 多个绑定
useKeybindings({
  'chat:submit': () => handleSubmit(),
  'chat:cancel': () => handleCancel(),
}, { context: 'Chat' })
```

内部使用 `useInput` hook 接收键盘事件,通过 KeybindingContext 的 `resolve()` 方法查找匹配的 action,然后调用注册的 handler。

---

## 10. Vim 模式实现

### 状态机设计

`vim/` 目录实现了完整的 Vim 文本编辑模式,用一个干净的状态机:

```typescript
// 完整的 Vim 状态
type VimState =
  | { mode: 'INSERT'; insertedText: string }    // 插入模式
  | { mode: 'NORMAL'; command: CommandState }    // 普通模式

// 普通模式下的命令状态机
type CommandState =
  | { type: 'idle' }                             // 等待输入
  | { type: 'count'; digits: string }            // 输入计数 (3dw)
  | { type: 'operator'; op: Operator; count }    // 等待 motion (d_)
  | { type: 'operatorCount'; op, count, digits }  // operator + count (d3w)
  | { type: 'operatorFind'; op, count, find }     // operator + find (df_)
  | { type: 'operatorTextObj'; op, count, scope }  // operator + text obj (diw)
  | { type: 'find'; find: FindType; count }       // f/F/t/T 等待字符
  | { type: 'g'; count }                          // g 前缀命令
  | { type: 'replace'; count }                    // r 等待替换字符
  | { type: 'indent'; dir: '>' | '<'; count }     // >> / << 缩进
```

### 状态转换

```
State Diagram:
  idle ──┬─[d/c/y]──► operator
         ├─[1-9]────► count
         ├─[fFtT]───► find
         ├─[g]──────► g
         ├─[r]──────► replace
         └─[><]─────► indent

  operator ─┬─[motion]──► execute
            ├─[0-9]────► operatorCount
            ├─[ia]─────► operatorTextObj
            └─[fFtT]───► operatorFind
```

转换函数是纯函数,返回 `{ next?: CommandState; execute?: () => void }`:

```typescript
function transition(state, input, ctx): TransitionResult {
  switch (state.type) {
    case 'idle':     return fromIdle(input, ctx)
    case 'count':    return fromCount(state, input, ctx)
    case 'operator': return fromOperator(state, input, ctx)
    // ...
  }
}
```

### 持久状态（记忆）

```typescript
type PersistentState = {
  lastChange: RecordedChange | null  // dot-repeat (.)
  lastFind: { type, char } | null   // repeat find (;/,)
  register: string                   // yank 寄存器
  registerIsLinewise: boolean
}
```

### RecordedChange（dot-repeat）

```typescript
type RecordedChange =
  | { type: 'insert'; text: string }      // i + 输入文本
  | { type: 'operator'; op, motion, count } // dw, c$, y0
  | { type: 'operatorTextObj'; op, objType, scope, count } // diw, ca"
  | { type: 'replace'; char, count }      // r + 字符
  | { type: 'x'; count }                  // x 删除
  | { type: 'toggleCase'; count }         // ~ 大小写切换
  | { type: 'indent'; dir, count }        // >> / <<
  | { type: 'openLine'; direction }       // o / O
  | { type: 'join'; count }              // J 合并行
```

### 支持的操作

- **Movement** — h/l/j/k, w/b/e/W/B/E, 0/^/$, gg/G, gj/gk
- **Operators** — d(delete), c(change), y(yank), >/< (indent)
- **Find** — f/F/t/T + 字符, ;/, 重复
- **Text Objects** — iw/aw, i"/a", i(/a(, i{/a{, i[/a[, i</a<
- **Commands** — x, ~, r, J, p/P, D/C/Y, o/O, u(undo), .(repeat)

### 与 TextInput 集成

`components/VimTextInput.tsx` (15K) 将 Vim 状态机与输入框集成:

```typescript
function VimTextInput({ vim, value, onChange, ... }) {
  // Normal 模式: 拦截按键 → transition() → 执行操作
  // Insert 模式: 正常文本输入 + 记录 insertedText
  // Esc: 切换到 Normal 模式
}
```

---

## 11. 主题与输出样式系统

### 主题系统

`components/design-system/ThemeProvider.tsx` 提供主题上下文:

- 内置多种主题(light/dark/高对比度等)
- 通过 `useTheme()` hook 获取当前主题颜色
- 主题色包含: primary, secondary, success, error, warning, dimmed 等

### 输出样式

`outputStyles/loadOutputStylesDir.ts` 加载自定义输出样式:

```
加载源:
  ~/.claude/output-styles/*.md     → 用户级样式
  .claude/output-styles/*.md       → 项目级样式(优先)
  插件提供的样式

每个 .md 文件:
  ---
  name: concise
  description: Brief and to the point
  keep-coding-instructions: true
  ---
  [样式 prompt 内容]
```

输出样式实际上是注入到系统 prompt 中的指令,指导 AI 的输出格式和风格。

### OutputStylePicker

`components/OutputStylePicker.tsx` 提供交互式样式选择:
- 列出所有可用样式
- 实时预览
- 选择后保存到设置

---

## 12. 虚拟滚动与全屏布局

### FullscreenLayout

`components/FullscreenLayout.tsx` (82K) 是全屏模式的布局容器:

```
┌─ AlternateScreen ──────────────────────────────────┐
│  ┌─ FullscreenLayout ────────────────────────────┐ │
│  │  ┌─ ScrollBox ──────────────────────────────┐ │ │
│  │  │                                          │ │ │
│  │  │  [消息列表 / VirtualMessageList]          │ │ │
│  │  │                                          │ │ │
│  │  └──────────────────────────────────────────┘ │ │
│  │  ┌─ StatusLine ─────────────────────────────┐ │ │
│  │  │  model │ permission │ cwd │ tokens │ cost │ │ │
│  │  └──────────────────────────────────────────┘ │ │
│  │  ┌─ PromptInput ────────────────────────────┐ │ │
│  │  │  > [用户输入...]                          │ │ │
│  │  │  [自动完成建议]                           │ │ │
│  │  │  [footer pills: tasks/teams/bridge]       │ │ │
│  │  └──────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

### ScrollBox 组件

```typescript
// ink/components/ScrollBox.tsx
type ScrollBoxHandle = {
  scrollTo(y: number): void
  scrollBy(dy: number): void
  scrollToElement(el, offset?): void  // 延迟到渲染时读位置
  scrollToBottom(): void
  getScrollTop(): number
  getScrollHeight(): number
  getViewportHeight(): number
  isSticky(): boolean                  // 是否跟随底部
  subscribe(listener): () => void      // 滚动事件
  setClampBounds(min?, max?): void     // 虚拟滚动钳制
}
```

关键设计:
- `stickyScroll` — 新内容自动滚动到底部
- `pendingScrollDelta` — 累积滚动增量,每帧最多消费 SCROLL_MAX_PER_FRAME 行
- `scrollAnchor` — 延迟到渲染时读取元素位置(避免竞态)
- `scrollClampMin/Max` — 虚拟滚动的可见范围钳制

### useVirtualScroll Hook

`hooks/useVirtualScroll.ts` (34K) 实现虚拟化:

核心思路:
1. 所有消息有估算高度(缓存实际高度)
2. 根据 scrollTop 计算可见窗口内的消息索引范围
3. 只渲染窗口内 + buffer 区域的消息
4. 未渲染区域用 Spacer(固定高度 Box)占位
5. 滚动时动态更新可见范围

---

## 13. 可复用架构模式

### 模式 1: 自制 Store 替代 Redux/Zustand

仅用 35 行代码实现了足够用的状态管理:
- `getState()`/`setState()` — 同步读写
- `subscribe()` — 订阅变更
- `Object.is()` — 引用相等跳过
- `onChange` 回调 — 集中处理副作用
- `useSyncExternalStore` — 与 React 18/19 集成

**适用场景**: 单一全局状态,无需中间件/devtools 的项目。

### 模式 2: DeepImmutable + 可变例外

```typescript
type AppState = DeepImmutable<{
  // 大部分状态是深度不可变的
  settings: SettingsJson
  verbose: boolean
}> & {
  // 包含函数类型或 Map/Set 的字段排除
  tasks: { [taskId: string]: TaskState }
  agentNameRegistry: Map<string, AgentId>
}
```

**好处**: 编译时防止意外 mutation,但不强制 class 实例等不可序列化类型。

### 模式 3: 事件驱动的 DOM 事件模型

将 W3C 事件模型移植到终端:
- capture → target → bubble 三阶段
- `stopPropagation()` / `stopImmediatePropagation()`
- 事件优先级映射 React scheduler
- 焦点管理(focus stack + auto-focus)

**适用场景**: 需要复杂交互的终端 UI。

### 模式 4: Keybinding 分层解析

```
用户输入
  → parser.ts: "ctrl+x ctrl+e" → [{ctrl: true, key: 'x'}, {ctrl: true, key: 'e'}]
  → resolver.ts: 匹配绑定 → 'chat:externalEditor'
  → useKeybinding.ts: 查找注册的 handler → 执行
```

分离了: 按键解析 / 绑定配置 / 动作处理。支持 chord 序列、context 优先级、用户自定义覆盖。

### 模式 5: 权限组件工厂

每个工具有自己的权限 UI 组件,通过 `permissionComponentForTool()` 工厂函数选择:

```typescript
// 统一的 PermissionRequestProps 接口
type PermissionRequestProps = {
  tool: Tool
  input: AnyObject
  toolUseContext: ToolUseContext
  assistantMessage: AssistantMessage
  onAllow: (input?, updates?) => void
  onDeny: (message?) => void
  onAbort: () => void
}

// 各工具实现自己的展示逻辑
function BashPermissionRequest(props) {
  // 显示命令内容 + 安全警告 + 操作按钮
}
```

### 模式 6: feature() 编译时消除

```typescript
import { feature } from 'bun:bundle'

// 编译时消除——外部构建不包含这些代码
const VoiceProvider = feature('VOICE_MODE')
  ? require('../context/voice.js').VoiceProvider
  : ({ children }) => children

// 条件组件
if (feature('COORDINATOR_MODE')) {
  // ...
}
```

Bun bundler 的 `feature()` 在编译时求值为布尔常量,未启用的分支被 DCE 完全移除。

### 模式 7: React Compiler 输出

所有组件都经过 React Compiler 编译,生成优化的 memoization 代码:

```typescript
// 编译前(源代码)
function App({ children, initialState }) {
  return (
    <AppStateProvider initialState={initialState} onChangeAppState={onChangeAppState}>
      {children}
    </AppStateProvider>
  )
}

// 编译后(实际运行)
function App(t0) {
  const $ = _c(9)  // 缓存数组
  const { children, initialState } = t0
  let t1
  if ($[0] !== children || $[1] !== initialState) {
    t1 = <AppStateProvider ...>{children}</AppStateProvider>
    $[0] = children; $[1] = initialState; $[2] = t1
  } else {
    t1 = $[2]  // 命中缓存,跳过创建
  }
  return t1
}
```

### 模式 8: Vim 状态机

将编辑器状态建模为有限状态机:
- 每个状态是一个类型化的 union member
- 转换函数是纯函数: `(state, input, ctx) → { next?, execute? }`
- TypeScript 的 exhaustive switch 确保处理所有状态
- 持久状态(寄存器、lastChange)与瞬态状态(command)分离

---

## 附录: 关键文件大小排名

| 文件 | 大小 | 说明 |
|------|------|------|
| `screens/REPL.tsx` | 874K | 主交互界面 |
| `ink/ink.tsx` | 246K | Ink 渲染引擎核心 |
| `hooks/useTypeahead.tsx` | 207K | 自动完成 |
| `components/LogSelector.tsx` | 195K | 日志选择器 |
| `components/Stats.tsx` | 149K | 统计面板 |
| `components/VirtualMessageList.tsx` | 145K | 虚拟滚动消息列表 |
| `components/ScrollKeybindingHandler.tsx` | 145K | 滚动键绑定处理 |
| `components/Messages.tsx` | 144K | 消息渲染 |
| `components/MessageSelector.tsx` | 112K | 消息选择器 |
| `hooks/useReplBridge.tsx` | 112K | REPL Bridge hook |
| `hooks/useVoiceIntegration.tsx` | 97K | 语音集成 |
| `components/Spinner.tsx` | 85K | Spinner 动画 |
| `components/FullscreenLayout.tsx` | 82K | 全屏布局 |
| `components/ConsoleOAuthFlow.tsx` | 78K | OAuth 流程 |
| `components/Message.tsx` | 77K | 单条消息渲染 |
| `components/ContextVisualization.tsx` | 74K | 上下文可视化 |
| `screens/Doctor.tsx` | 71K | 环境诊断 |
