# Claude Code Bible

> 对 Claude Code 源码的深度架构剖析 —— 一份 Agentic 系统工程的圣经。
> 从 512K+ 行源码中提炼出的架构模式、工程技巧和可迁移的最佳实践。

---

## 背景

2026 年 3 月 31 日，Claude Code 的源码通过 npm 包中的 `.map` 文件意外公开。[@Fried_rice](https://x.com/Fried_rice/status/2038894956459290963) 首先发现了这一事件。

本仓库对这份公开的源码快照（~1,900 个 TypeScript 文件，512,000+ 行代码）进行了**系统性的架构分析与知识沉淀**，旨在理解世界级 Agentic CLI 系统的工程实践。

**技术栈**：TypeScript / Bun / React + Ink / Commander.js / Zod v4 / MCP SDK

---

## 知识文档

| 文档 | 大小 | 核心内容 |
|------|------|---------|
| [**葵花宝典**](knowledge/00-ultimate-guide.md) | 42K | 终极心法 —— 架构总纲、12 条可迁移模式、第一人称运行时视角 |
| [核心引擎](knowledge/01-core-engine.md) | 40K | 启动优化、QueryEngine 状态机、12 步流水线、流式处理、DCE |
| [工具系统](knowledge/02-tool-system.md) | 32K | 42+ 工具架构、BashTool 6 层安全纵深、权限模型、Tool 接口契约 |
| [服务层](knowledge/03-services-layer.md) | 32K | API 多供应商工厂、MCP 6 种传输协议、3 层上下文压缩、重试引擎 |
| [UI 与状态](knowledge/04-ui-and-state.md) | 39K | 自定义 Ink 渲染引擎、35 行 Zustand-like Store、Vim FSM、虚拟滚动 |
| [多 Agent 协调](knowledge/05-multi-agent-coordination.md) | 35K | IDE Bridge 协议、Agent Swarm 3 种后端、7 种任务类型、技能/插件系统 |
| [基础设施](knowledge/06-utils-infrastructure.md) | 43K | Bootstrap 叶模块隔离、5 层配置合并、Bash AST 安全分析、记忆系统 |

**总计 263K 的深度知识沉淀。**

---

## 为什么 Claude Code 如此强大？

葵花宝典（[00-ultimate-guide.md](knowledge/00-ultimate-guide.md)）给出了完整回答，核心在于六个支柱：

```
              System Prompt (精心构造的身份与规则)
                        │
          ┌─────────────┼─────────────┐
          │             │             │
    Tool System    Query Loop    Context Mgmt
    (42+ 工具)      (状态机)     (4层压缩)
          │             │             │
          └─────────────┼─────────────┘
                        │
          ┌─────────────┼─────────────┐
          │             │             │
    Permission     Multi-Agent   Skill/Plugin
    & Security      Swarm        扩展系统
```

**12 条可迁移心法**（详见葵花宝典第十一章）：

1. Import-Gap 并行预取
2. Generator 驱动的状态机
3. 多层压缩管道
4. 纵深防御的安全模型
5. Fail-Closed 默认值 + Fail-Open 服务
6. Schema 驱动的工具系统
7. 投机执行与逐级升级
8. 叶模块隔离防循环依赖
9. Latch 模式保缓存键稳定
10. Agent 分治突破上下文限制
11. 35 行替代整个库
12. 错误恢复不是可选的

---

## 研究方法

使用 6 个 Claude Opus agent 并行深度学习源码，每个 agent 负责一个核心模块：

| Agent | 负责模块 | 方法 |
|-------|---------|------|
| core-engine-agent | 核心引擎 | 阅读 main.tsx, QueryEngine.ts, query.ts, Tool.ts 等 |
| tool-system-agent | 工具系统 | 阅读 42+ 工具实现，权限系统 hooks |
| services-agent | 服务层 | 阅读 API 客户端、MCP、压缩、认证等 |
| ui-state-agent | UI 与状态 | 阅读 React/Ink 组件、状态管理、Vim 模式 |
| multi-agent-agent | 多 Agent 协调 | 阅读 Bridge、协调器、任务、技能/插件 |
| utils-infra-agent | 基础设施 | 阅读 330+ utils、bootstrap、配置、安全 |

最终由 team lead 融合所有模块知识 + 运行时第一人称视角，撰写葵花宝典。

---

## 声明

- 本仓库是**教育和安全研究性质的知识沉淀**，不包含 Claude Code 的源代码
- 所有文档基于公开暴露的源码快照进行架构分析
- Claude Code 原始代码的所有权归 **Anthropic** 所有
- 本仓库与 Anthropic 无任何关联
