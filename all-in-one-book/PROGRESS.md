# OpenHands 实现原理全解 — 写作进度

## 章节规划

| # | 章节标题 | 文件名 | 核心覆盖 | 状态 |
|---|---------|--------|---------|------|
| 0 | 序章：全书地图 | ch00-overview.md | 项目定位 / 架构全景图 / 核心概念词典 / 代码库地图 / 一次典型交互极简全流程 | ✅ |
| 1 | 一次对话的完整旅程：数据流全景 | ch01-data-flow.md | 用户输入→LLM推理→沙箱执行→结果返回的完整数据流，每步拆解数据形态变化 | ⏳ |
| 2 | 事件系统：OpenHands 的神经网络 | ch02-event-system.md | Event / Action / Observation / EventStream / 订阅分发 / 序列化持久化 | ⏳ |
| 3 | 控制器：决策中枢 | ch03-controller.md | AgentController 主循环 / 状态机 / 步进逻辑 / 委托机制 / 卡死检测 | ⏳ |
| 4 | CodeActAgent：主力智能体 | ch04-codeact-agent.md | step() 流程 / 消息构建 / 工具注册 / 响应解析 / 多动作队列 | ⏳ |
| 5 | LLM 集成：与大模型对话 | ch05-llm.md | LLM 类 / litellm 适配 / 函数调用转换 / 重试机制 / token 追踪 / 路由 | ⏳ |
| 6 | 运行时：沙箱执行环境 | ch06-runtime.md | Runtime 抽象 / Docker 实现 / ActionExecutionServer / 容器生命周期 / 文件操作 | ⏳ |
| 7 | 记忆与历史管理 | ch07-memory.md | Condenser 体系 / 历史压缩策略 / ConversationMemory / 上下文窗口管理 | ⏳ |
| 8 | 服务端与前端：用户界面层 | ch08-server-frontend.md | FastAPI+SocketIO 服务器 / V0 与 V1 架构 / 前端 React 应用 / WebSocket 实时通信 | ⏳ |
| 9 | 安全体系 | ch09-security.md | ActionSecurityRisk / 确认模式 / SecurityAnalyzer / 秘钥管理 | ⏳ |
| 10 | 项目演进史 | ch10-evolution.md | 从 2024.3 初始化到 2026.3 的架构演进：基础搭建→核心框架→产品化→企业化 | ⏳ |
| 11 | 端到端追踪：完整场景串联 | ch11-e2e-trace.md | "帮我修一个 bug"全流程追踪，串联全书所有模块 | ⏳ |

## 章节规划说明

**认知路径设计：**

```
序章（全局画面，建立心理模型）
  → 第1章（跟着一条数据走一遍，建立直觉）
    → 第2章（事件系统——一切通信的基础设施）
      → 第3章（控制器——驱动循环的引擎）
        → 第4章（CodeActAgent——做决策的大脑）
          → 第5章（LLM集成——大脑的思考能力）
            → 第6章（运行时——执行命令的双手）
              → 第7章（记忆管理——不遗忘的能力）
                → 第8章（服务端+前端——用户看到的部分）
                  → 第9章（安全体系——保护机制）
                    → 第10章（演进史——为什么长成这样）
                      → 第11章（端到端验收——串联一切）
```

**为什么这样排？**

- 序章+第1章让读者先有全局画面和直觉，后面的深入不会迷路
- 第2章事件系统是所有模块通信的基础，必须先懂
- 第3-6章沿着"控制器调 Agent → Agent 调 LLM → 控制器派发到 Runtime"的主干依次展开
- 第7章记忆管理是 Agent 的支撑系统，放在 Agent 和 LLM 之后
- 第8章服务端/前端是外围，读者此时已理解核心，可以理解 API 设计决策
- 第9章安全体系贯穿多层，需要前面的背景
- 第10章演进史，读者已经理解当前架构，能体会"为什么"
- 第11章端到端是总验收

## 状态说明
- ✅ 已完成
- 🔄 进行中
- ⏳ 待开始

## 术语约定

### 行业标准术语
| 术语 | 说明 |
|------|------|
| LLM (Large Language Model) | 大语言模型 |
| Agent | 智能体，能自主决策和行动的 AI 程序 |
| Sandbox | 沙箱，隔离的代码执行环境 |
| Runtime | 运行时，代码实际执行的环境 |
| Event Sourcing | 事件溯源，用事件序列记录系统状态变化 |
| WebSocket | 全双工通信协议 |
| Function Calling | 函数调用，LLM 结构化输出调用工具的能力 |
| Token | LLM 处理文本的基本单位 |

### 项目特有术语
| 术语 | 类比 | 区别 |
|------|------|------|
| EventStream | 类似消息队列 | OpenHands 的核心事件总线，所有模块通过它通信，同时负责持久化 |
| Action | 类似"命令" | Agent 发出的操作指令，如执行命令、编辑文件 |
| Observation | 类似"响应" | 环境返回给 Agent 的执行结果 |
| AgentController | 类似"调度器" | 驱动 Agent 循环的控制器，管理状态机和步进逻辑 |
| CodeActAgent | 项目主力 Agent | 能执行代码、编辑文件、浏览网页的全能 Agent |
| Condenser | 类似"摘要器" | 压缩对话历史以适应 LLM 上下文窗口 |
| ActionExecutionServer | 类似"远程执行器" | 运行在 Docker 容器内的 HTTP 服务，接收并执行 Action |
| FileStore | 类似"文件数据库" | 基于文件系统的事件持久化存储 |

## 下次续写指引

### 从哪里继续
从第1章 ch01-data-flow.md 开始写作。序章已完成。

### 交接备忘
- 项目版本 v1.5.0，处于 V0→V1 迁移期（V0 将于 2026.4 废弃）
- V0 架构：AgentController + EventStream + Runtime（当前主要运行路径）
- V1 架构：AppServer + SDK + AppConversationService（新推荐路径）
- 总 commit 约 6370 个，时间跨度 2024.3 - 2026.3
- 主要 Agent 是 CodeActAgent（版本 2.2），其他 Agent 为辅助角色
- 核心数据流：User → EventStream → AgentController → Agent.step() → LLM → Actions → EventStream → Runtime → Observations → EventStream → 循环

### 待验证项
- [ ] V1 AppConversationService 的完整调用路径（与 V0 对比）
- [ ] Condenser 的具体实现策略（LLM-based vs. 截断 vs. 摘要）
- [ ] MCP (Model Context Protocol) 集成的具体机制
- [ ] SecurityAnalyzer 的完整实现
- [ ] 前端 WebSocket 事件的完整监听列表
