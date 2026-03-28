# OpenHands 实现原理全解 — 写作进度

## 章节规划

| # | 章节标题 | 文件名 | 核心覆盖 | 状态 |
|---|---------|--------|---------|------|
| 1 | 序言：全书地图 | ch01-preface.md | 项目定位、架构全景图、核心概念词典、代码库地图、极简全流程 | ✅ |
| 2 | 数据流全景：一条消息的完整旅程 | ch02-data-flow.md | 用户消息→EventStream→Agent→LLM→Action→Runtime→Observation 完整链路，每步数据形态 | ✅ |
| 3 | 核心循环：动作与观测的世界观 | ch03-action-observation.md | Action/Observation 类型体系、EventStream 机制、AgentController 状态机 | ⏳ |
| 4 | 大脑：LLM 层与函数调用 | ch04-llm-layer.md | LiteLLM 封装、function calling、重试机制、CodeAct 范式 | ⏳ |
| 5 | 执行环境：运行时沙箱深解 | ch05-runtime-sandbox.md | Docker/Local/Kubernetes Runtime、action_execution_server、插件系统 | ⏳ |
| 6 | 记忆管理：Condenser 与上下文窗口 | ch06-memory-condenser.md | ConversationMemory、Condenser 体系、截断策略 | ⏳ |
| 7 | 微代理：动态知识注入机制 | ch07-microagents.md | KnowledgeMicroagent、RepoMicroagent、触发机制、Prompt 构建 | ⏳ |
| 8 | 项目演进史：从 OpenDevin 到 OpenHands V1 | ch08-evolution.md | 初创期→V0 成熟→V1 迁移，架构演变与设计决策 | ⏳ |
| 9 | 端到端追踪：修复一个 Bug 的完整旅程 | ch09-end-to-end.md | 串联全书，追踪"修复 GitHub Issue"场景全流程 | ⏳ |

## 章节规划说明

认知路径：
```
全局画面（ch01）
  → 主干数据流（ch02）
  → 事件系统深解（ch03）—— 理解"消息总线"才能理解后续各模块如何协作
  → LLM 大脑（ch04）—— 理解决策来源
  → 执行沙箱（ch05）—— 理解动作如何被执行
  → 记忆管理（ch06）—— 理解长对话如何保持
  → 微代理（ch07）—— 理解知识如何注入
  → 演进史（ch08）—— 理解"为什么是现在这个样子"
  → 端到端追踪（ch09）—— 串联验收
```

## 项目关键信息

**项目本质**：AI 驱动的软件工程代理平台，让 LLM 能真正在沙箱环境中执行代码、操作文件、浏览网页。

**当前版本状态**：
- **V0（遗留）**：所有文件标注 `# IMPORTANT: LEGACY V0 CODE`，计划 2026-04-01 移除
- **V1（现行）**：`openhands/app_server/` 为新应用服务器，核心 Agent 逻辑迁移至外部 [Software Agent SDK](https://github.com/OpenHands/software-agent-sdk)

**核心范式**：CodeAct —— Agent 通过执行代码（bash/Python）来完成任务，而非仅生成文本。

## 状态说明
- ✅ 已完成
- 🔄 进行中
- ⏳ 待开始

## 术语约定

| 术语 | 类型 | 说明 |
|------|------|------|
| Action | 项目特有 | Agent 发出的"意图指令"，类比为"命令" |
| Observation | 项目特有 | 环境返回给 Agent 的"执行结果"，类比为"反馈" |
| EventStream | 项目特有 | 中央事件总线，所有组件通过它通信 |
| AgentController | 项目特有 | Agent 主循环协调器（V0），类比为"调度员" |
| Runtime | 项目特有 | 沙箱执行环境，类比为"操作系统接口层" |
| Condenser | 项目特有 | 上下文压缩器，类比为"记忆整理者" |
| Microagent | 项目特有 | 动态注入的小型提示片段，类比为"专家知识卡片" |
| CodeAct | 行业新范式 | 用代码表达一切 Agent 动作的设计思想 |
| LiteLLM | 行业标准 | 统一的 LLM API 代理库，类比为"适配器" |
| Sandbox | 行业标准 | 隔离的代码执行环境 |
| function calling | 行业标准 | LLM 调用结构化工具的机制 |

## 下次续写指引

### 从哪里继续
ch03 — 核心循环：动作与观测的世界观

### 交接备忘
- ch01 和 ch02 已完成并提交
- V0 代码大量标注 LEGACY，写作时需同时覆盖 V0（核心机制说明）和 V1（现行架构）
- CodeActAgent 是主要 Agent，step() 方法是核心入口
- AgentController 的 `_step()` → `agent.step(state)` → LLM → Action → EventStream → Runtime → Observation → EventStream → back to `_step()`
- EventStream 既是持久化存储又是实时通知总线

### 待验证项
- V1 App Server 与 Software Agent SDK 的具体交互协议（SDK 为外部 repo，本书专注此 repo）
- Kubernetes Runtime 的具体实现细节
