# 第一章：序言——全书地图

> 在我们深入任何一行代码之前，先站在高处看一眼整片森林。这一章是全书的导航仪。读完这一章，你将知道 OpenHands 是什么、它由哪些大块组成、各个模块叫什么名字，以及一次典型交互在极简层面上是如何发生的。后续每一章都会拿起这张地图，放大其中一个区域。

---

## 1.1 它是什么？

OpenHands 是一个 **AI 驱动的软件工程代理平台**。

用一句话说：你把一个编程任务交给它，它会自己启动一个隔离的代码执行环境，反复调用大语言模型（LLM）来决定下一步做什么，在那个环境里执行代码、读写文件、搜索互联网，直到任务完成——就像一个真实的程序员在用电脑工作，只不过它是 AI。

**它解决的核心问题是：** LLM 本身只能生成文字，无法真正执行任何东西。OpenHands 搭建了"LLM 大脑 + 真实执行环境"的桥梁，让 AI 从"能说"变成"能做"。

### 与其他工具的对比定位

| 工具 | 定位 | OpenHands 的区别 |
|------|------|----------------|
| ChatGPT / Claude | 对话 AI，生成代码文本 | OpenHands 真正执行代码 |
| GitHub Copilot | IDE 内代码补全 | OpenHands 自主完成多步任务 |
| Claude Code / Codex CLI | 单轮命令行执行 | OpenHands 支持多轮、多工具、有状态 |
| Devin / Jules | 类似定位 | OpenHands 完全开源，SWE-Bench 评分 77.6% |

### 四种使用形态

OpenHands 本身不是单一产品，而是一个生态：

```
┌─────────────────────────────────────────────────────────┐
│                      OpenHands 生态                      │
├──────────────┬──────────────┬──────────────┬────────────┤
│  SDK         │  CLI         │  Local GUI   │  Cloud     │
│ Python 库    │ 命令行工具   │ 本地网页界面 │ 托管服务   │
│ 可编程调用   │ 类似 Claude  │ 类似 Devin   │ 企业级功能 │
│              │ Code 体验    │              │            │
└──────────────┴──────────────┴──────────────┴────────────┘
```

本书聚焦的是这个 GitHub 仓库中的 **Local GUI**（本地网页界面）及其底层实现，这里包含了所有核心机制。

---

## 1.2 架构全景图

OpenHands 的整体架构可以分为三个层次：

```
┌─────────────────────────────────────────────────────────────┐
│                        用户界面层                            │
│   React 前端 (frontend/)  │  CLI  │  外部 API 调用          │
└──────────────────────────────┬──────────────────────────────┘
                               │ HTTP/WebSocket
┌──────────────────────────────▼──────────────────────────────┐
│                        应用服务器层                          │
│                                                              │
│  V1 App Server (openhands/app_server/)                       │
│  ┌────────────┐ ┌──────────┐ ┌──────────┐ ┌─────────────┐  │
│  │ 对话管理   │ │ 沙箱管理 │ │ 用户管理 │ │ 事件路由    │  │
│  │/app_conv.. │ │/sandbox  │ │/user     │ │/event       │  │
│  └────────────┘ └──────────┘ └──────────┘ └─────────────┘  │
│                                                              │
│  V0 Legacy Server (openhands/server/)  [计划 2026-04 移除]  │
└──────────────────────────────┬──────────────────────────────┘
                               │ 内部调用
┌──────────────────────────────▼──────────────────────────────┐
│                         代理核心层                           │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │           EventStream（事件总线）                   │    │
│  │   所有组件通过它通信：发布事件、订阅事件            │    │
│  └──────┬────────────────┬───────────────┬─────────────┘    │
│         │                │               │                   │
│  ┌──────▼──────┐  ┌──────▼──────┐  ┌────▼────────────────┐ │
│  │AgentController│ │  Runtime    │  │      LLM            │ │
│  │（调度员）     │ │（沙箱）     │  │  （通过 LiteLLM）   │ │
│  │  驱动主循环  │ │ 执行 Action │  │  思考，返回 Action  │ │
│  └──────┬──────┘  └──────┬──────┘  └─────────────────────┘ │
│         │                │                                   │
│  ┌──────▼──────┐  ┌──────▼──────┐                           │
│  │  CodeAct    │  │  Condenser  │                           │
│  │  Agent      │  │（记忆管理）  │                           │
│  │  (主 Agent) │  └─────────────┘                           │
│  └─────────────┘                                            │
└─────────────────────────────────────────────────────────────┘
```

**理解这张图的关键：** EventStream（事件总线）是中心枢纽，而非某个组件直接调用另一个组件。AgentController 向 EventStream 写入 Action，Runtime 从 EventStream 读取 Action 并执行，再把 Observation（执行结果）写回 EventStream，AgentController 读到 Observation 后再让 Agent 下一步决策。这个设计让各组件彻底解耦。

---

## 1.3 核心概念词典

在进入细节之前，先把本书将反复出现的核心术语解释清楚。

### Action（动作）——项目特有术语

Agent 的"意图指令"。Agent 每次调用 LLM 后，决定要做什么，就会生成一个 Action。

Action 有许多种类，但核心的几个是：

| Action 类型 | 含义 | 例子 |
|-------------|------|------|
| `CmdRunAction` | 运行 bash 命令 | `ls -la /workspace` |
| `IPythonRunCellAction` | 运行 Python 代码 | `import pandas as pd; ...` |
| `FileReadAction` | 读取文件 | 读取 `main.py` |
| `FileEditAction` | 编辑文件 | 修改第 10-20 行 |
| `BrowseURLAction` | 打开网页 | 打开 `https://...` |
| `MessageAction` | 向用户发消息 | "我需要你提供更多信息" |
| `AgentFinishAction` | 宣布任务完成 | "任务已完成" |
| `AgentDelegateAction` | 委派给子 Agent | "让搜索 Agent 去找文档" |

**类比：** Action 就像是程序员写在便利贴上的"待办事项"，标注了要执行的操作类型和参数。

### Observation（观测）——项目特有术语

"环境反馈"。每个 Action 执行后，系统会生成一个对应的 Observation，告诉 Agent 发生了什么。

| Observation 类型 | 对应的 Action | 含义 |
|-----------------|---------------|------|
| `CmdOutputObservation` | `CmdRunAction` | 命令的输出（stdout/stderr） |
| `IPythonRunCellObservation` | `IPythonRunCellAction` | Python 执行结果 |
| `FileReadObservation` | `FileReadAction` | 文件内容 |
| `ErrorObservation` | 任意 | 执行出错了 |
| `BrowserObservation` | `BrowseURLAction` | 网页内容 |

**类比：** Observation 是环境对"待办事项"的"执行报告"——成功了还是失败了，输出了什么。

### EventStream（事件总线）——项目特有术语

系统中所有 Action 和 Observation 都流经的中央通道。它同时承担两个职责：

1. **实时通知**：当一个新事件被写入时，立即通知所有已订阅的组件
2. **持久化存储**：把所有事件持久化到磁盘，保证会话可以恢复

**类比：** 像一个既是实时广播频道又是历史存档的消息系统（有点像 Kafka，但轻量得多）。

### Agent（代理）——行业标准术语

能够感知环境、做出决策、采取行动的 AI 实体。在 OpenHands 中，Agent 的职责是：

- 接收当前 State（包含历史事件）
- 调用 LLM 决定下一步
- 返回一个 Action

OpenHands 内置多个 Agent，最核心的是 **CodeActAgent**（详见 ch04）。

### Runtime（运行时）——项目特有术语

负责真正执行 Action 的执行环境。它是 Agent 的"手"——Agent 决定做什么，Runtime 去做。

有三种实现：
- **DockerRuntime**：Agent 在 Docker 容器里执行代码（默认，最安全）
- **LocalRuntime**：直接在宿主机执行（仅限信任环境）
- **RemoteRuntime**：在云端远程沙箱执行

### AgentController（控制器）——项目特有术语

主循环协调者。它驱动整个"Agent 决策 → Action 执行 → Observation 返回 → 再决策"的循环，管理 Agent 的状态机，处理异常（比如 Agent 卡死、上下文超限等）。

注意：这是 V0 的核心组件，V1 中这个职责由 Software Agent SDK 承担。

### Condenser（压缩器）——项目特有术语

管理 Agent 的"记忆"。LLM 有上下文窗口大小限制，对话历史太长时必须压缩。Condenser 决定哪些历史事件可以被总结/遗忘，保持 Agent 的有效记忆在限制内。**详见第六章。**

### Microagent（微代理）——项目特有术语

动态注入的"专家知识卡片"。不是独立运行的 Agent，而是 Markdown 文件，包含特定领域的知识或操作规范。当用户消息包含特定触发词时，对应的 Microagent 内容会被注入到 Agent 的系统提示中。**详见第七章。**

---

## 1.4 代码库地图

```
fork-OpenHands/
├── openhands/                    # 核心 Python 包
│   ├── app_server/               # V1 应用服务器（FastAPI）⭐ 现行
│   │   ├── app_conversation/     # 对话管理 API
│   │   ├── sandbox/              # 沙箱管理 API
│   │   ├── user/                 # 用户管理 API
│   │   └── event/                # 事件 API
│   ├── agenthub/                 # Agent 实现集合
│   │   ├── codeact_agent/        # ⭐ 主力 Agent（CodeAct 范式）
│   │   ├── browsing_agent/       # 专注网页浏览的 Agent
│   │   └── readonly_agent/       # 只读模式 Agent
│   ├── controller/               # Agent 控制器（V0 遗留）
│   │   ├── agent.py              # Agent 抽象基类
│   │   ├── agent_controller.py   # 主循环协调者
│   │   └── state/                # State 数据结构
│   ├── events/                   # 事件系统 ⭐ 核心
│   │   ├── action/               # 所有 Action 类型定义
│   │   ├── observation/          # 所有 Observation 类型定义
│   │   ├── stream.py             # EventStream（事件总线）
│   │   └── event.py              # Event 基类
│   ├── llm/                      # LLM 封装层（V0 遗留）
│   │   ├── llm.py                # LiteLLM 封装
│   │   └── metrics.py            # Token 使用统计
│   ├── runtime/                  # 执行环境
│   │   ├── base.py               # Runtime 抽象基类（V0）
│   │   └── impl/                 # 各 Runtime 具体实现
│   │       ├── docker/           # Docker 沙箱
│   │       ├── local/            # 本地执行
│   │       └── remote/           # 远程执行
│   ├── memory/                   # 记忆管理
│   │   ├── condenser/            # Condenser 压缩器体系
│   │   └── conversation_memory.py
│   ├── microagent/               # 微代理系统
│   ├── server/                   # V0 遗留服务器
│   └── storage/                  # 持久化存储
│       └── data_models/          # 数据模型定义
├── frontend/                     # React 前端
│   └── src/
│       ├── api/                  # API 调用层
│       ├── hooks/                # TanStack Query hooks
│       └── routes/               # 页面路由
├── microagents/                  # 内置微代理知识库（Markdown 文件）
└── tests/                        # 测试目录
```

**哪些目录最重要？**

如果你只想理解核心机制，需要重点阅读：
1. `openhands/events/` — 理解整个系统的通信基础
2. `openhands/agenthub/codeact_agent/` — 理解 Agent 如何决策
3. `openhands/controller/` — 理解主循环（V0）
4. `openhands/runtime/` — 理解代码如何真正被执行

---

## 1.5 一次典型交互的极简全流程

在深入任何细节之前，先用最少的步骤理解一次交互的全貌。假设用户在网页界面输入：

> "帮我在 main.py 里加一个 hello world 函数"

下面是从输入到输出的极简流程：

```
用户输入消息
    │
    ▼
[前端] React 界面捕获输入
    │ WebSocket / HTTP
    ▼
[应用服务器] 接收请求，路由到对话处理
    │ 写入事件
    ▼
[EventStream] 记录 MessageAction（用户消息）
    │ 通知订阅者
    ▼
[AgentController] 收到事件，触发 agent.step(state)
    │
    ▼
[CodeActAgent] 构建对话历史 → 调用 LLM
    │ LLM 返回：执行 FileEditAction
    ▼
[EventStream] 记录 FileEditAction
    │ 通知 Runtime
    ▼
[Runtime] 真正修改 main.py 文件
    │ 操作完成
    ▼
[EventStream] 记录 FileEditObservation（成功）
    │ 通知 AgentController
    ▼
[AgentController] 再次触发 agent.step(state)
    │
    ▼
[CodeActAgent] 读取到修改成功 → 调用 LLM
    │ LLM 返回：AgentFinishAction（任务完成）
    ▼
[EventStream] 记录 AgentFinishAction
    │
    ▼
[前端] 显示"任务完成"消息
```

**关键洞察：**
- 整个流程中，没有任何一个组件直接调用另一个组件——所有通信都通过 EventStream
- Agent 不执行任何代码，它只返回"意图"（Action）；Runtime 才真正执行
- 每次 LLM 只做一步决策（"我的下一个动作是什么"），而不是一次性规划全部步骤

这个"Action → EventStream → Runtime → Observation → EventStream → Agent → 再 Action"的循环，就是 OpenHands 的核心工作机制。接下来的各章都是围绕这个循环展开，逐步放大每个环节的细节。

---

## 1.6 V0 与 V1：你需要了解的历史背景

阅读代码时你会经常看到这样的注释：

```python
# IMPORTANT: LEGACY V0 CODE - Deprecated since version 1.0.0,
# scheduled for removal April 1, 2026
```

这说明 OpenHands 正在进行一次重大架构迁移：

- **V0**：所有代码都在这个仓库内，包括 Agent 核心逻辑
- **V1**：Agent 核心逻辑被抽离到独立的 [Software Agent SDK](https://github.com/OpenHands/software-agent-sdk)，本仓库保留应用服务器（`openhands/app_server/`）

本书在讲解核心机制时，会以 V0 的代码为主要参考（因为它在本仓库中完整可读），同时标注 V1 的对应位置。这两个版本的核心设计思想是一致的，V1 只是把边界划分得更清晰。

---

下一章，我们将追踪一条真实消息的完整旅程，看清每一步数据的具体形态是如何变化的。

---

### 质检报告

**讲解节奏**
- [x] 先讲"OpenHands 是什么"（1.1），再讲"里面有什么"（1.2-1.4）

**周边知识**
- [x] 1.1 中有与其他工具的对比，帮助定位
- [x] 1.6 交代了 V0/V1 迁移背景，为全书阅读做铺垫

**讲透了吗**
- [x] 核心概念词典覆盖了全书将反复出现的术语
- [x] 极简全流程给出了完整的 Action→Observation 循环
- [x] 没有跳步：每个术语都有类比解释

**代码纪律**
- [x] 全章代码片段：1 处（V0 注释示例，不贴不知道 V0/V1 划分）
- [x] 该处确实是"不贴无法理解上下文"

**流程图准确性**
- [x] 极简全流程基于源码中 AgentController → agent.step() → EventStream → Runtime 的实际调用关系
- [x] 图下方有逐步文字解释

**过渡自然吗**
- [x] 章尾引出第二章（追踪完整消息旅程）

**准确吗**
- [x] SWE-Bench 分数来自 README
- [x] Action/Observation 类型均来自源码 `openhands/events/action/__init__.py` 和 `openhands/events/observation/__init__.py`

**读得下去吗**
- [x] 所有术语首次出现均有解释
- [x] 每张图都有文字说明
