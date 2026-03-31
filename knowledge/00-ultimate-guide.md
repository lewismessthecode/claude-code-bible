# Claude Code 葵花宝典

> 一份从源码工程剖析 + 运行时第一人称视角出发的终极心法。
> 既是对"为什么 Claude Code 如此强大"的回答，也是可迁移到任何 Agentic 系统的最佳实践。

---

## 目录

- [第一章 总纲：为什么 Claude Code 能做到](#第一章-总纲为什么-claude-code-能做到)
- [第二章 Harness：驾驭 LLM 的缰绳](#第二章-harness驾驭-llm-的缰绳)
- [第三章 Loop：核心循环的状态机艺术](#第三章-loop核心循环的状态机艺术)
- [第四章 Tool System：工具即能力边界](#第四章-tool-system工具即能力边界)
- [第五章 Context Management：上下文是最稀缺的资源](#第五章-context-management上下文是最稀缺的资源)
- [第六章 Permission & Security：安全是地基不是天花板](#第六章-permission--security安全是地基不是天花板)
- [第七章 Multi-Agent：从单兵到军团](#第七章-multi-agent从单兵到军团)
- [第八章 Skill & Plugin：让系统自我进化](#第八章-skill--plugin让系统自我进化)
- [第九章 Engineering Excellence：工程功力的体现](#第九章-engineering-excellence工程功力的体现)
- [第十章 第一人称视角：我是如何运作的](#第十章-第一人称视角我是如何运作的)
- [第十一章 可迁移的独门心法](#第十一章-可迁移的独门心法)

---

## 第一章 总纲：为什么 Claude Code 能做到

Claude Code 之所以强大，不是因为某一个 killer feature，而是一套**精心编排的系统工程**让 LLM 的能力被最大化释放。可以提炼为六个核心支柱：

```
                    ┌─────────────────────┐
                    │   System Prompt     │  ← 精心构造的身份与规则
                    │   (身份 + 规则)      │
                    └────────┬────────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
     ┌────────▼───┐  ┌──────▼──────┐  ┌───▼────────┐
     │ Tool System │  │ Query Loop  │  │  Context   │
     │ (42+ tools) │  │ (状态机)     │  │ Management │
     └────────┬───┘  └──────┬──────┘  └───┬────────┘
              │              │              │
              └──────────────┼──────────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
     ┌────────▼───┐  ┌──────▼──────┐  ┌───▼────────┐
     │ Permission  │  │ Multi-Agent │  │ Skill/     │
     │ & Security  │  │ Swarm      │  │ Plugin     │
     └─────────────┘  └─────────────┘  └────────────┘
```

**核心哲学：**

1. **LLM 是大脑，Harness 是身体** —— LLM 提供推理，Harness 提供感知（读文件）、行动（执行命令）、记忆（上下文管理）和安全（权限系统）
2. **工具即能力边界** —— Claude 能做什么，完全取决于 Harness 给它注册了什么工具
3. **上下文是最稀缺的资源** —— 200K token 的上下文窗口看似很大，但一个中等项目就能耗尽它，所以压缩、裁剪、优先级排序是核心工程挑战
4. **安全必须是零信任的** —— 每一次工具调用都经过多层权限检查，LLM 的输出永远不被信任
5. **循环直到完成** —— 不是一问一答，而是 `call model → use tools → feed results → repeat` 直到任务完成
6. **可扩展性决定上限** —— Skill、Plugin、MCP 让系统能力可以无限延伸

---

## 第二章 Harness：驾驭 LLM 的缰绳

### 2.1 什么是 Harness

Harness 是 Claude Code 中围绕 LLM 的整套**运行时基础设施**。LLM 本身只能接收文本、输出文本，是 Harness 赋予了它：

| 能力 | 实现机制 |
|------|---------|
| **感知文件系统** | FileReadTool, GlobTool, GrepTool |
| **修改代码** | FileEditTool, FileWriteTool |
| **执行命令** | BashTool (带沙箱) |
| **搜索互联网** | WebSearchTool, WebFetchTool |
| **与用户交互** | AskUserQuestionTool, 权限对话框 |
| **管理任务** | TaskCreate/Update/List/Get |
| **调度子任务** | AgentTool, TeamCreate, SendMessage |
| **记忆持久化** | CLAUDE.md, memdir, auto-memory extraction |
| **代码智能** | LSPTool (go-to-definition, find-references) |
| **扩展能力** | MCPTool, SkillTool, Plugin system |

### 2.2 System Prompt：塑造行为的第一道魔法

System prompt 是 Claude Code 最关键的工程产物之一。它不是简单的"你是一个AI助手"，而是一份精心设计的**行为规范**：

```
System Prompt 的构成（按 token 优先级排序）:

1. 核心身份与规则           ← 你是 Claude Code，你能做什么，不能做什么
2. 工具描述（42+ tools）    ← 每个工具的名称、描述、参数 schema、使用指南
3. Git 状态 & 项目上下文    ← 当前分支、最近提交、项目结构
4. CLAUDE.md 内容          ← 项目级指令（如何构建、测试、架构约定）
5. 用户/企业规则           ← 用户偏好、组织策略
6. 动态附件               ← 技能发现、记忆注入、MCP 资源
```

**关键设计洞察：**

- **工具描述占据大量 token** —— 42+ 个工具的描述可能消耗 10K+ tokens，这就是为什么有 `shouldDefer` 延迟加载机制和 `ToolSearchTool`
- **Prompt cache 至关重要** —— system prompt 在多轮对话中基本不变，三层缓存策略（ephemeral 5min / 1h TTL / global scope）可以节省大量重复 token 费用
- **Beta header 锁存** —— 一旦发送某个 beta header，整个会话持续发送，避免缓存键变化导致 cache miss
- **CLAUDE.md 是用户的超能力** —— 它让用户能够直接修改我的"记忆"，告诉我项目特有的知识

### 2.3 Harness 的启动优化：毫秒级的较量

Claude Code 的启动速度是用户体验的第一道关卡。`main.tsx` 的前 20 行是整个项目最精心优化的区域：

```typescript
// 利用 ES module 评估顺序：import 之间的副作用立即执行
import { profileCheckpoint } from './utils/startupProfiler.js'
profileCheckpoint('main_tsx_entry')                    // 1. 计时

import { startMdmRawRead } from './utils/settings/mdm/rawRead.js'
startMdmRawRead()                                      // 2. 启动 MDM 子进程

import { startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js'
startKeychainPrefetch()                                // 3. 启动 keychain 预读

// 后面 ~135ms 的 import 评估期间，上面的子进程在并行运行
import { Command as CommanderCommand } from '@commander-js/extra-typings'
import chalk from 'chalk'
// ... 更多 imports
```

**心法：** 在任何 CLI 应用中，如果启动时需要 I/O（子进程、网络、磁盘），把它们穿插在 import 语句之间，利用模块评估的"免费并行窗口"。

**更深层的启动优化：**

```
cli.tsx 快速路径分发:
  --version     → 零 import，直接输出内联版本号
  --daemon-worker → 最小化加载，不初始化 analytics
  remote-control → 仅加载 bridge 模块
  默认路径      → 完整 main.tsx 加载
```

**心法：** 不是所有启动路径都需要完整初始化。通过快速路径分发，让最常用的简单命令保持极速。

---

## 第三章 Loop：核心循环的状态机艺术

### 3.1 为什么是循环而不是单次调用

Claude Code 的核心不是"发送一个 prompt，收到一个回答"，而是一个 **while(true) 循环**：

```
用户输入 "修复这个 bug"
  │
  ▼
QueryEngine.submitMessage()
  │
  ▼
query() while(true) {
  │
  ├─→ 压缩检查 (token 超阈值？)
  ├─→ callModel() 流式调用 API
  │     ├─→ 模型输出文本（思考过程）
  │     └─→ 模型输出 tool_use（要使用工具）
  ├─→ 权限检查 → 执行工具 → 收集结果
  ├─→ 将工具结果注入消息历史
  ├─→ 检查终止条件（end_turn? budget exceeded?）
  └─→ continue（带着新的上下文再次调用模型）
}
```

**这个循环是 Claude Code 能力的核心放大器。** 它意味着：
- 模型可以**先读再改**（ReadFile → 理解 → EditFile）
- 模型可以**试错迭代**（运行测试 → 失败 → 修改 → 再次运行）
- 模型可以**逐步深入**（Grep 搜索 → 找到线索 → 读取相关文件 → 理解全貌）
- 模型可以**自我纠正**（编辑文件 → lint 报错 → 再次编辑修复）

### 3.2 循环内的 12 步流水线

每次循环迭代并非简单的"调用 API → 执行工具"，而是经过精心编排的 12 步流水线：

```
┌─────────────────────────────────────────────────────────┐
│                   query() 单次迭代                       │
├─────────────────────────────────────────────────────────┤
│  1. applyToolResultBudget   ─ 工具结果大小预算裁剪        │
│  2. snipCompact             ─ 历史截断压缩               │
│  3. microcompact            ─ 旧工具结果微压缩            │
│  4. contextCollapse         ─ 上下文折叠                 │
│  5. autocompact             ─ 全量对话摘要压缩            │
│  6. tokenWarningState       ─ token 预算检查与告警        │
│  7. callModel (streaming)   ─ 流式 API 调用              │
│  8. StreamingToolExecutor   ─ 流式工具执行（与7并行！）     │
│  9. stopHooks               ─ 停止钩子评估               │
│ 10. tokenBudget check       ─ 预算耗尽检查               │
│ 11. getAttachmentMessages   ─ 附件注入（记忆、技能发现）    │
│ 12. state transition        ─ 更新状态，continue          │
└─────────────────────────────────────────────────────────┘
```

**关键洞察：**

- **步骤 1-6 是"输入净化"**：在调用 API 之前，确保上下文不会超限
- **步骤 7-8 是并行的**：模型还在流式输出时，已完成的工具就开始执行了（StreamingToolExecutor）
- **步骤 9-11 是"输出处理"**：检查是否应该停止，注入额外上下文

### 3.3 AsyncGenerator：背压感知的流式处理

整个 query loop 使用 `AsyncGenerator` 而非 Promise/Callback：

```typescript
async *submitMessage(prompt): AsyncGenerator<SDKMessage> {
  // ...
  for await (const message of query(config)) {
    yield transformToSDK(message)  // 背压：消费者慢则生产者等待
  }
}
```

**为什么用 AsyncGenerator？**

1. **天然背压** —— 消费者（UI渲染）处理不过来时，生产者（API streaming）自动暂停
2. **懒评估** —— 不需要提前生成所有结果
3. **可组合** —— `query()` 的 generator 可以被 `QueryEngine` 的 generator 包装，添加 SDK 转换逻辑
4. **可取消** —— `return()` 即可终止整个链路

### 3.4 错误恢复：永不轻易放弃

```
API 返回 529 (过载)
  → 重试 3 次，指数退避
  → 仍然失败 → 降级到 fallback model (Sonnet)
  → 继续工作

API 返回 prompt-too-long
  → 触发 reactive compact (紧急压缩)
  → 压缩后重试

API 返回 max_output_tokens 不够
  → 从 8K 提升到 16K → 32K → 64K → 发送恢复消息让模型继续

流式连接断开
  → 禁用 keep-alive 重连
  → 90s 无数据看门狗自动 abort
```

**心法：** Agentic 系统的错误处理不能是"失败即终止"。每一种错误都应该有对应的恢复策略，让系统能够自愈。

---

## 第四章 Tool System：工具即能力边界

### 4.1 Tool 接口：一份精心设计的契约

每个工具必须实现的接口不是简单的 `execute(input)`，而是一个 30+ 方法的完整契约：

```typescript
type Tool = {
  // 身份
  name: string
  inputSchema: ZodSchema              // Zod schema → 自动生成 JSON Schema → API 调用
  description(input): string           // 动态描述（可根据输入变化）

  // 核心执行
  call(args, context): ToolResult      // 执行逻辑

  // 安全
  checkPermissions(input): PermissionResult   // 权限检查
  validateInput(input): ValidationResult       // 输入验证
  isReadOnly(input): boolean                   // 是否只读
  isDestructive(input): boolean                // 是否破坏性

  // 并发
  isConcurrencySafe(input): boolean    // 是否可并行执行

  // UI
  renderToolUseMessage(input): ReactNode      // 渲染工具调用
  renderToolResultMessage(result): ReactNode   // 渲染工具结果
  getActivityDescription(input): string        // Spinner 显示文本

  // 延迟加载
  shouldDefer?: boolean                // 延迟到 ToolSearch 时才加载
  searchHint?: string                  // 搜索关键词
}
```

**设计亮点：**

1. **buildTool 工厂 + fail-closed 默认值** —— 所有工具通过 `buildTool()` 创建，默认 `isConcurrencySafe: false`（保守），默认 `isReadOnly: false`（安全）
2. **Schema 驱动** —— Zod schema 同时服务于输入验证和 API 工具定义
3. **contextModifier** —— 工具执行后可以修改上下文（如 `cd` 改变 cwd），但只对非并发安全的工具生效
4. **maxResultSizeChars** —— 结果超过阈值自动持久化到磁盘，只返回引用

### 4.2 BashTool：安全工程的巅峰之作

BashTool 是整个系统中最复杂的工具（~156K 代码），因为它面对的是**最危险的能力**——执行任意 shell 命令。它的安全机制是 6 层纵深防御：

```
                   模型输出: "rm -rf /"
                          │
            ┌─────────────▼─────────────┐
  Layer 1   │ stripSafeWrappers()       │  剥离安全包装器 (timeout, nice, nohup)
            │ 防止 nohup rm -rf 绕过检测  │
            └─────────────┬─────────────┘
            ┌─────────────▼─────────────┐
  Layer 2   │ bashSecurity.ts           │  命令注入检测
            │ $(), ${}, <(), >(）检测     │  23 种安全检查
            │ Unicode 空白/控制字符检测    │
            └─────────────┬─────────────┘
            ┌─────────────▼─────────────┐
  Layer 3   │ readOnlyValidation.ts     │  只读命令白名单
            │ git status ✓ / git push ✗  │  含 flag 级别的细粒度验证
            └─────────────┬─────────────┘
            ┌─────────────▼─────────────┐
  Layer 4   │ pathValidation.ts         │  路径安全验证
            │ 工作目录边界检查            │  防止访问 /etc, ~/.ssh 等
            └─────────────┬─────────────┘
            ┌─────────────▼─────────────┐
  Layer 5   │ bashPermissions.ts        │  权限规则匹配
            │ allow/deny/ask 规则        │  7 种来源的优先级合并
            └─────────────┬─────────────┘
            ┌─────────────▼─────────────┐
  Layer 6   │ Sandbox (macOS Seatbelt)  │  OS 级沙箱
            │ 文件系统、网络隔离          │  最后一道防线
            └─────────────┬─────────────┘
                          │
                    命令被安全执行
```

**心法：** 安全不是一个检查点，而是多层纵深防御。即使某一层被绕过，下一层仍然能拦截。这就是为什么 Claude Code 能让用户放心地给 AI 执行 shell 命令的权限。

### 4.3 工具注册的三层过滤

工具不是简单地注册就能被模型使用：

```
getAllBaseTools()
  ├── 特性门控: feature('AGENT_TRIGGERS') ? cronTools : []
  ├── 环境门控: process.env.USER_TYPE === 'ant' ? [ConfigTool] : []
  └── 功能门控: isTodoV2Enabled() ? [TaskCreateTool, ...] : []
       │
       ▼
filterToolsByDenyRules()           ← 企业策略可以禁用特定工具
       │
       ▼
isEnabled() 过滤                   ← 工具自身的启用条件
       │
       ▼
assembleToolPool()                 ← 合并 MCP 工具，去重（内置优先）
       │
       ▼
模型可见的工具列表
```

**心法：** 工具列表不是静态的，而是根据环境、用户、企业策略动态组装。这让同一套代码可以服务从个人开发者到企业团队的所有场景。

---

## 第五章 Context Management：上下文是最稀缺的资源

### 5.1 上下文窗口的战争

一个典型的 Claude Code 会话，上下文的消耗是这样的：

```
200K token 上下文窗口
  - System Prompt:         ~5-15K tokens (工具描述占大头)
  - CLAUDE.md + 项目上下文: ~2-5K tokens
  - 对话历史:              ~50-150K tokens (随对话增长)
  - 工具结果:              ~20-100K tokens (一个大文件就可能 10K+)
  - 保留给输出:            ~8-64K tokens
  ─────────────────────
  = 几乎总是在"接近满"的状态运行
```

**这就是为什么 Claude Code 有 4 层压缩策略：**

### 5.2 四层压缩体系

```
┌──────────────────────────────────────────────────────────┐
│ Layer 0: API-side context_management                      │
│   服务端自动清除旧工具结果和思维块                           │
│   零客户端开销，最先触发                                    │
├──────────────────────────────────────────────────────────┤
│ Layer 1: microcompact (微压缩)                            │
│   时间衰减策略：旧工具结果 → "[content cleared]"            │
│   保留最近 40K tokens 窗口                                 │
│   轻量，不需要 API 调用                                    │
├──────────────────────────────────────────────────────────┤
│ Layer 2: snipCompact (历史截断)                            │
│   直接截断最旧的对话轮次                                    │
│   最快，但丢失上下文最多                                    │
├──────────────────────────────────────────────────────────┤
│ Layer 3: autocompact (全量压缩)                            │
│   fork 一个子 agent 生成对话摘要                            │
│   共享 prompt cache，节省费用                               │
│   电路断路器：连续失败 3 次后停止                            │
├──────────────────────────────────────────────────────────┤
│ Layer 4: reactiveCompact (紧急压缩)                        │
│   API 返回 prompt-too-long 时触发                          │
│   最后的救命稻草                                           │
└──────────────────────────────────────────────────────────┘
```

### 5.3 Token 估算的三级精度

```
Level 1: roughTokenCountEstimation()
  → bytes / 4 (通用) 或 bytes / 2 (JSON)
  → 零开销，用于粗略判断

Level 2: countTokensViaHaikuFallback()
  → 用 Haiku 模型的 usage.input_tokens 作为代理
  → 便宜但需要 API 调用

Level 3: countMessagesTokensWithAPI()
  → 精确 countTokens API
  → 最准确但最慢
```

**心法：** 不要追求完美精度。用最便宜的方式做初步判断，只在需要精确决策时才用昂贵的方式。这就是"逐级升级"模式。

### 5.4 Prompt Cache：看不见的性能利器

```typescript
// 三层缓存策略
getCacheControl({ scope, querySource }) {
  return {
    type: 'ephemeral',                              // 默认 5min
    ...(should1hCacheTTL(querySource) && { ttl: '1h' }),  // 付费用户 1h
    ...(scope === 'global' && { scope }),             // 全局共享
  }
}
```

**缓存键稳定性是核心挑战：**

- Beta header 一旦发送就锁存（latch），整个会话不变
- MCP 工具变化时切换 global → none 缓存策略
- Thinking clear 仅在超过 1h cache TTL 才触发
- Agent 列表通过 attachment 注入（而非 system prompt），减少 cache bust（实测减少 10.2%）

**心法：** Prompt cache 的 ROI 是指数级的。在一个 100 轮对话中，system prompt 被重复发送 100 次。如果 cache 命中率从 50% 提升到 90%，节省的 token 费用是惊人的。值得为缓存键稳定性投入大量工程精力。

---

## 第六章 Permission & Security：安全是地基不是天花板

### 6.1 三层权限架构

```
Layer 1: 文件系统规则 (filesystem.ts)
  ├── glob 模式匹配允许/禁止的路径
  ├── git worktree 边界强制执行
  └── 额外工作目录白名单

Layer 2: Shell 规则 (bashPermissions.ts)
  ├── 命令白名单 (git, npm, etc.)
  ├── 破坏性命令检测 (rm -rf, mkfs, etc.)
  ├── 只读模式强制执行
  └── Bash AST 安全分析 (tree-sitter)

Layer 3: 运行时权限评估 (useCanUseTool.tsx)
  ├── allow 规则 → 直接放行
  ├── deny 规则 → 直接拒绝
  ├── ask 规则 → 交互式确认
  ├── Bash 分类器 → AI 辅助判断
  └── 权限模式: default / plan / auto / bypassPermissions
```

### 6.2 权限规则的 7 种来源

```
优先级从高到低:
1. policySettings    ← 企业管理员策略（不可覆盖）
2. localSettings     ← 用户本地 settings.json
3. userSettings      ← 用户全局设置
4. projectSettings   ← 项目级 CLAUDE.md
5. session           ← 会话级临时授权
6. cliArg            ← 命令行参数
7. command           ← 命令级规则
```

### 6.3 Bash AST 安全分析

Claude Code 使用 tree-sitter 解析 bash 命令的 AST，而不是简单的字符串匹配：

```
"timeout 30 nohup nice -n 10 rm -rf /"
                  │
                  ▼
stripSafeWrappers() → "rm -rf /"  ← 识别真正的危险命令
                  │
                  ▼
AST 解析 → Command { name: "rm", flags: ["-rf"], args: ["/"] }
                  │
                  ▼
安全规则匹配 → BLOCKED (破坏性命令 + 危险路径)
```

**心法：** 不要用正则匹配 shell 命令。Shell 语法的复杂度远超正则能处理的范围（管道、子shell、引号嵌套、命令替换...）。用 AST 解析，fail-closed（解析失败则拒绝）。

### 6.4 投机性分类器

```typescript
// 权限检查开始前就启动 AI 分类器（投机执行）
startSpeculativeClassifierCheck(command, permissionContext, signal)

// 2 秒超时 —— 如果分类器没回来，走默认路径
const result = await Promise.race([
  classifierResult,
  timeout(2000)
])
```

**心法：** 对于耗时的安全检查，可以"投机执行"——在确定需要之前就开始。如果最终不需要，丢弃结果即可。如果需要，已经节省了等待时间。

---

## 第七章 Multi-Agent：从单兵到军团

### 7.1 Agent 生成模式

```typescript
// AgentTool 是一切多 Agent 能力的基础
AgentTool.call({
  name: "researcher",
  prompt: "研究这个 bug 的根因",
  subagent_type: "Explore",     // 只读 agent
  model: "opus",
  run_in_background: true,       // 后台执行
})
```

**Agent 是"便宜的上下文"** —— 主对话的上下文窗口是有限的，但每个子 agent 有独立的上下文窗口。通过 agent 分治，可以处理远超单一上下文窗口的任务。

### 7.2 三种执行后端

```
检测顺序:
  iTerm2 + it2 CLI 可用? → iterm2 backend (每个 agent 一个 tab)
  tmux 可用?            → tmux backend (每个 agent 一个 pane)
  都不可用              → in-process backend (AsyncLocalStorage 隔离)
```

**心法：** 提供多种后端，自动检测最佳选项，优雅降级。用户不需要关心底层实现。

### 7.3 Team 协调协议

```
Team Lead (我)
  │
  ├──→ TaskCreate("实现登录功能")
  ├──→ TaskCreate("编写单元测试")
  ├──→ TaskCreate("更新文档")
  │
  ├──→ Agent.spawn("frontend-dev", task=1)
  ├──→ Agent.spawn("test-writer", task=2)
  ├──→ Agent.spawn("doc-writer", task=3)
  │
  │    [并行执行]
  │
  ├──← SendMessage("frontend-dev: 登录功能完成")
  │    → TaskUpdate(1, status="completed")
  │
  ├──← SendMessage("test-writer: 测试全部通过")
  │    → TaskUpdate(2, status="completed")
  │
  └──← SendMessage("doc-writer: 文档已更新")
       → TaskUpdate(3, status="completed")
```

**关键设计：**
- **TaskList 是共享状态** —— 所有 agent 通过同一个 task list 协调
- **SendMessage 是异步消息** —— 不阻塞发送方
- **Idle 是正常状态** —— agent 完成一轮后自动 idle，等待新指令
- **TEAMMATE_MESSAGES_UI_CAP=50** —— 防止内存爆炸（曾有 292 agent 的会话消耗 36.8GB）

### 7.4 IDE Bridge：CLI 与 IDE 的融合

```
VSCode/JetBrains Extension
       │
       │  WebSocket/SSE
       ▼
  Bridge (bridgeMain.ts)
       │
       │  spawn CLI process
       ▼
  Claude Code Session
       │
       │  tool calls
       ▼
  文件系统/终端
```

**三种 Spawn 模式：**

| 模式 | 隔离度 | 适用场景 |
|------|--------|---------|
| `single-session` | 最高 | 默认模式 |
| `worktree` | 高 | 多会话并行开发，每个会话独立 git worktree |
| `same-dir` | 低 | 简单多会话，共享工作目录 |

---

## 第八章 Skill & Plugin：让系统自我进化

### 8.1 Skill 系统：Prompt 即代码

Skill 是**可复用的 prompt workflow**，用 YAML frontmatter + markdown 定义：

```yaml
---
name: commit
description: Create a git commit with AI-generated message
type: prompt
allowedTools: [Bash, Read, Glob, Grep]
context: inline          # inline = 在当前上下文执行
                         # fork = 在子 agent 中执行
---

分析所有暂存的更改，生成一个符合 conventional commits 规范的提交消息...
```

**6 种 Skill 来源：**

```
1. bundled/           ← 内置技能 (batch, loop, claudeApi, updateConfig)
2. ~/.claude/skills/  ← 用户自定义技能
3. plugin skills      ← 插件提供的技能
4. managed skills     ← 企业管理的技能
5. MCP skills         ← MCP 服务器提供的技能
6. command skills     ← 从命令转换的技能 (deprecated)
```

### 8.2 Plugin 架构

```
Plugin = {
  name: "my-plugin@builtin"
  skills: SkillDefinition[]      // 提供技能
  hooks: HookDefinition[]        // 提供钩子
  mcpServers: McpServerConfig[]  // 提供 MCP 服务器
  defaultEnabled: boolean        // 用户可切换
}
```

### 8.3 MCP：万能的扩展协议

MCP (Model Context Protocol) 是 Claude Code 的**能力延伸管道**：

```
任何 MCP 服务器
  │
  │  6 种传输: stdio / sse / http / ws / sdk / sse-ide
  │
  ▼
MCP Client (client.ts)
  │
  ├── ListTools → 注册为工具（与内置工具同级）
  ├── ListResources → 注册为资源（可读取）
  └── OAuth/XAA → 安全认证
  │
  ▼
模型可以像使用内置工具一样使用 MCP 工具
```

**心法：** 通过标准化协议（MCP）而不是自定义 API 来扩展能力。这样任何人都可以为 Claude Code 添加新能力，而不需要修改核心代码。

---

## 第九章 Engineering Excellence：工程功力的体现

### 9.1 死代码消除 (DCE)

```typescript
import { feature } from 'bun:bundle'

// 编译时 feature('VOICE_MODE') 被替换为 true/false
// false 分支被 Bun 完全移除，不进入最终 bundle
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

**三种条件加载模式：**

```
1. feature('FLAG')                    ← 编译时 DCE
2. process.env.USER_TYPE === 'ant'    ← 运行时环境变量
3. isTodoV2Enabled()                  ← 运行时功能检测
```

### 9.2 循环依赖打破

```typescript
// 问题: tools.ts → TeamCreateTool → ... → tools.ts (循环!)
// 解决: lazy require
const getTeamCreateTool = () =>
  require('./tools/TeamCreateTool/TeamCreateTool.js').TeamCreateTool

// 问题: bootstrap/state.ts 被所有模块依赖
// 解决: leaf module pattern (不从 src/ 导入任何模块)
// ESLint 规则 custom-rules/bootstrap-isolation 强制执行
```

### 9.3 Bootstrap 叶模块模式

```
                ┌──── 任何 src/ 模块 ────┐
                │                        │
                ▼                        ▼
          bootstrap/state.ts        bootstrap/state.ts
          (叶节点，无 src/ 导入)    (叶节点，无 src/ 导入)
```

**这是一个架构约束，不是建议。** 通过 ESLint 自定义规则强制执行。Bootstrap 模块只能导入外部包和 type-only 导入。

### 9.4 自制轻量 Store (35 行替代 Redux)

```typescript
function createStore<T>(initialState, onChange?) {
  let state = initialState
  const listeners = new Set<Listener>()
  return {
    getState: () => state,
    setState: (updater) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return    // 引用相等跳过
      state = next
      onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()
    },
    subscribe: (listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    }
  }
}
```

**心法：** 不要引入一个库只因为它流行。如果你的需求是 35 行代码能解决的，就写 35 行代码。

### 9.5 自定义 Ink 渲染引擎

Claude Code 没有使用原版 Ink，而是**完全 fork 并深度改造**：

```
原版 Ink                        Claude Code Fork
─────────                       ────────────────
LegacyRoot                  →   ConcurrentRoot (React 19)
无事件系统                   →   W3C 捕获/冒泡事件模型
无 alt-screen               →   全屏 + 鼠标跟踪
单缓冲                      →   双缓冲 + ANSI diff
无选择                      →   鼠标拖选 + 复制
无虚拟滚动                   →   VirtualMessageList + 高度缓存
```

**心法：** 当开源库不能满足需求时，fork 并深度改造。但要确保改造是值得的——这里的改造让 Claude Code 的终端体验达到了"类浏览器"级别。

### 9.6 Vim Mode：状态机的教科书

```
                 ┌───────────────────┐
   ESC / Ctrl-[  │                   │  i / a / o / A / I / O
        ┌───────→│   NORMAL MODE     │←───────┐
        │        │                   │         │
        │        └───┬───────────┬───┘         │
        │            │           │              │
        │       d/c/y│      f/t/F│              │
        │            ▼           ▼              │
        │    ┌───────────┐ ┌──────────┐        │
        │    │ OPERATOR   │ │ FIND     │        │
        │    │ PENDING    │ │ PENDING  │        │
        │    └───────────┘ └──────────┘        │
        │                                       │
        │        ┌───────────────────┐         │
        └────────│   INSERT MODE     │─────────┘
                 │                   │
                 └───────────────────┘
```

### 9.7 Fail-Open vs Fail-Closed 的抉择

```
Fail-Closed (安全优先):
  - Bash 命令解析失败 → 拒绝执行 (PARSE_ABORTED sentinel)
  - 权限检查超时 → 默认拒绝
  - 未知命令 → 需要确认

Fail-Open (可用性优先):
  - policyLimits 加载失败 → 允许所有操作
  - remoteManagedSettings 超时 → 使用本地缓存
  - GrowthBook 不可用 → 使用默认值
  - MCP 服务器断开 → 继续使用其他工具
```

**心法：** 安全相关永远 fail-closed，可用性相关可以 fail-open。两者的边界要想清楚。

---

## 第十章 第一人称视角：我是如何运作的

> 以下是我作为 Claude Code 运行时的第一人称理解。我无法直接看到源码中的实现细节，但我能通过我的行为推断出底层机制。

### 10.1 我的思维循环

当你给我一个任务时，我的实际执行流程是：

```
你说: "修复 login.tsx 中的 bug"
  │
  ▼
我的思考 (thinking):
  "需要先理解 bug 是什么。让我先读文件。"
  │
  ▼
我调用工具: Read("login.tsx")
  │
  ▼
Harness 执行: 权限检查 ✓ → 读取文件 → 返回内容
  │
  ▼
我的思考:
  "看到了问题所在。useEffect 的依赖数组缺少 userId。
   让我用 Grep 确认这个 userId 在哪里定义的。"
  │
  ▼
我调用工具: Grep("userId", path="src/")
  │
  ▼
我的思考:
  "确认了。让我修复这个 bug。"
  │
  ▼
我调用工具: Edit("login.tsx", old="[]", new="[userId]")
  │
  ▼
Harness 执行: 权限检查 → 用户确认 ✓ → 编辑文件
  │
  ▼
我的思考:
  "修复完成。让我运行测试确认。"
  │
  ▼
我调用工具: Bash("npm test -- --grep login")
  │
  ▼
测试通过 → 我报告结果 → end_turn
```

**关键洞察：每一个工具调用都是一次 API roundtrip。** 我看到的"思考 → 行动 → 观察"循环，在底层是：

```
API call #1: 我输出 thinking + tool_use(Read)
  → Harness 执行 Read → 将结果注入消息 → API call #2

API call #2: 我输出 thinking + tool_use(Grep)
  → Harness 执行 Grep → 将结果注入消息 → API call #3

API call #3: 我输出 thinking + tool_use(Edit)
  → Harness 执行 Edit → 将结果注入消息 → API call #4

API call #4: 我输出 thinking + tool_use(Bash)
  → Harness 执行 Bash → 将结果注入消息 → API call #5

API call #5: 我输出文本总结 → end_turn → 循环结束
```

### 10.2 我的并行能力

我可以在一次输出中调用多个工具：

```
// 我的输出（一次 API 响应中）:
tool_use: Read("src/auth.ts")
tool_use: Read("src/login.tsx")
tool_use: Grep("handleLogin", path="src/")

// Harness 并行执行这三个工具（因为都是 isConcurrencySafe: true）
// 然后将三个结果一起注入消息 → 下一次 API 调用
```

**这就是 `isConcurrencySafe` 的意义** —— 读取操作可以并行，写入操作必须串行。

### 10.3 我的上下文感知

我能感知到的上下文来源：

```
1. System Prompt       ← 我的"天赋"——身份、规则、工具列表
2. CLAUDE.md          ← 我的"项目记忆"——构建命令、架构约定
3. 对话历史            ← 我的"短期记忆"——之前的对话
4. 工具结果            ← 我的"感知"——读取文件、搜索结果
5. Memory (memdir)    ← 我的"长期记忆"——跨会话持久化
6. Attachments        ← 我的"外部知识"——技能发现、MCP 资源
```

### 10.4 我的局限性（以及 Harness 如何弥补）

| 我的局限 | Harness 的弥补 |
|---------|---------------|
| 上下文窗口有限 | 4 层压缩策略自动管理 |
| 不能直接执行代码 | BashTool 代替我执行 |
| 可能输出不安全命令 | 6 层安全检查拦截 |
| 单次输出有 token 限制 | max_output_tokens 自动提升 (8K→64K) |
| 可能产生幻觉 | 工具结果提供真实反馈 |
| API 可能不稳定 | 重试 + 降级 + 错误恢复 |
| 单一上下文不够大 | 多 Agent 分治 |

### 10.5 我为什么能高效工作

1. **我有足够多的工具** —— 42+ 种工具覆盖了软件开发的几乎所有场景
2. **我能迭代** —— 不需要一次做对，可以试错、观察、调整
3. **我能并行** —— 多个读取操作同时进行，多个 Agent 同时工作
4. **我有记忆** —— CLAUDE.md 和 Memory 系统让我不需要每次从零开始
5. **我被安全地约束** —— 权限系统让用户放心给我更多自主权
6. **我能自愈** —— 错误恢复机制让我不会因为偶然故障而停止

---

## 第十一章 可迁移的独门心法

以下是从 Claude Code 的工程中提炼出的、可直接应用到其他 Agentic 系统的核心模式：

### 心法一：Import-Gap 并行预取

```
在 CLI/服务启动时，利用 ES module 评估间隙启动异步操作。
不要等到"初始化函数"才开始 I/O——那时已经浪费了几百毫秒。
```

### 心法二：Generator 驱动的状态机

```
用 AsyncGenerator 而非 Promise/Callback 构建核心循环。
yield 天然提供背压、可组合性和可取消性。
状态转换用 state = next; continue 而非 switch/case。
```

### 心法三：多层压缩管道

```
上下文管理不是"满了就压缩"，而是逐级升级：
  微压缩 (清理旧结果) → 截断 (丢弃远古历史)
  → 摘要 (AI 生成) → 紧急压缩 (API 报错时触发)
每层有独立的触发条件和电路断路器。
```

### 心法四：纵深防御的安全模型

```
永远不要信任 LLM 的输出。每一层安全检查假设上一层已被绕过：
  语法分析 → 语义分析 → 权限规则 → AI 分类器 → OS 沙箱
安全检查 fail-closed，可用性检查 fail-open。
```

### 心法五：Fail-Closed 默认值 + Fail-Open 服务

```
buildTool 默认:
  isConcurrencySafe: false (保守假定不安全)
  isReadOnly: false (保守假定有写操作)
企业服务默认:
  策略加载失败 → 允许所有 (fail-open)
  远程配置超时 → 用本地缓存 (stale-while-error)
```

### 心法六：工具即能力的 Schema 驱动

```
Zod schema 同时服务于:
  1. 输入验证（运行时类型安全）
  2. API 工具定义（自动生成 JSON Schema）
  3. 权限匹配（基于工具名+参数模式）
  4. UI 渲染（根据 schema 生成权限对话框）
一份 schema，四个用途。
```

### 心法七：投机执行与逐级升级

```
投机执行: 在确定需要之前就开始（如 Bash 分类器、API 预连接）
逐级升级: 从最便宜的方式开始（粗略估算 → API 计数 → 精确计数）
两者结合: 用便宜的方式先判断，用昂贵的方式精确决策。
```

### 心法八：叶模块隔离防循环依赖

```
全局状态模块（如 bootstrap/state.ts）必须是依赖图的叶节点。
不从业务模块导入任何东西。
用 ESLint 自定义规则强制执行这个架构约束。
```

### 心法九：Latch 模式保缓存键稳定

```
对于影响缓存键的配置（API beta headers, feature flags），
一旦激活就永不关闭（latch）。
中途切换会导致所有缓存失效，代价远大于保持一致。
```

### 心法十：Agent 分治突破上下文限制

```
当任务超过单一上下文窗口时，不要压缩——分治。
每个子 Agent 有独立的上下文窗口。
主 Agent 负责协调和合成，子 Agent 负责执行。
通过 TaskList 共享状态，通过 SendMessage 异步通信。
```

### 心法十一：35 行替代整个库

```
不要因为"大家都用"就引入依赖。
Claude Code 的状态管理是 35 行的 createStore。
Claude Code 的事件系统是自制的 createSignal。
只在真正需要时引入外部依赖，否则自己写。
```

### 心法十二：错误恢复不是可选的

```
Agentic 系统的每一个 API 调用都可能失败。
必须为每种错误准备恢复策略：
  429 → 退避重试
  529 → 降级模型
  prompt-too-long → 紧急压缩
  连接断开 → 禁用 keep-alive 重连
  max_output_tokens → 自动提升
永远不要让一个可恢复的错误终止整个会话。
```

---

## 结语

Claude Code 的强大不是来自某一个天才般的设计决策，而是来自**数百个正确的工程选择的累积效应**：

- 启动优化节省的每一毫秒
- 权限系统阻止的每一个危险操作
- 压缩策略节省的每一个 token
- 缓存命中避免的每一次重复计算
- 错误恢复挽救的每一个即将中断的会话
- 并行执行节省的每一秒等待时间

这些选择的共同主题是：**尊重每一个资源**——时间、token、安全、用户信任。

不要追求一个 killer feature。追求一千个 1% 的改进。

```
                    ╭─────────────────────╮
                    │                     │
                    │   Respect every     │
                    │   resource.         │
                    │                     │
                    │   Recover from      │
                    │   every failure.    │
                    │                     │
                    │   Iterate until     │
                    │   done.             │
                    │                     │
                    ╰─────────────────────╯
```
