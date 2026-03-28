# 第五章：执行环境——运行时沙箱深解

> Agent 的"大脑"（LLM）决定了做什么，但真正去做的是 Runtime——执行环境。这一章我们深入 Runtime 层，理解为什么需要沙箱隔离、Docker 容器是如何启动和通信的、Action 是如何被真正执行的，以及插件系统是如何扩展沙箱能力的。

---

## 5.1 为什么需要沙箱

一个能自主执行代码的 AI，如果没有隔离，会非常危险：

- Agent 可能错误地删除系统文件
- Agent 可能安装恶意软件包
- Agent 可能修改主机的配置文件
- 一个崩溃的进程可能拖垮整个系统

沙箱（Sandbox）的作用是：**把 Agent 的活动空间限制在一个隔离的容器内**，即使 Agent 做了破坏性操作，也只影响这个容器，宿主机和其他会话完全不受影响。

OpenHands 的沙箱实现，是一个**两进程分离的架构**：

```
宿主机（安全区域）
├── OpenHands 主进程
│   ├── AgentController（决策调度）
│   ├── EventStream（事件总线）
│   └── DockerRuntime（沙箱管理者）
│          │
│          │  HTTP 通信
│          │  端口范围 30000-39999
│          ▼
├── Docker 容器（执行区域，隔离）
│   └── action_execution_server（FastAPI 服务）
│          ├── BashSession（管理 bash 会话）
│          ├── IPython 内核（管理 Python 会话）
│          ├── 浏览器（Playwright）
│          └── MCP 代理（外部工具）
```

这个设计的核心价值是：**代码崩溃只影响容器，不影响 OpenHands 主进程**。容器可以随时被销毁重建，而主进程和对话历史完整保留。

---

## 5.2 Runtime 的层次结构

OpenHands 的 Runtime 有清晰的抽象层次：

```
Runtime（抽象基类）
    └── ActionExecutionClient（共享 HTTP 客户端逻辑）
         ├── DockerRuntime    — 本地 Docker 容器（默认）
         └── RemoteRuntime   — 云端远程沙箱

Runtime（抽象基类）
    └── LocalRuntime         — 直接在宿主机执行（仅信任环境）
```

还有：
```
KubernetesRuntime            — Kubernetes Pod（企业级扩展）
CLIRuntime                   — 命令行模式简化版
```

`ActionExecutionClient` 是 Docker 和 Remote 两种 Runtime 共享的基类，它封装了与 `action_execution_server` 通信的 HTTP 客户端逻辑。本章以 `DockerRuntime` 为主要分析对象。

---

## 5.3 Docker Runtime 的生命周期

### 阶段一：启动容器

当一个新的对话开始时，DockerRuntime 负责启动一个新的 Docker 容器：

```
DockerRuntime.__init__()
    │
    ├── 分配端口（30000-39999 范围内找一个空闲端口）
    │   使用 PortLock 防止并发时的端口冲突
    │
    ├── build_runtime_image()
    │   检查是否已有对应版本的运行时镜像
    │   如果没有，从基础镜像构建（加入 action_execution_server 等）
    │
    ├── 启动 Docker 容器
    │   指定镜像、端口映射、挂载工作目录
    │   环境变量（包括用户凭证）注入
    │
    ├── 等待 action_execution_server 健康检查通过
    │   GET /alive → 200 OK
    │   使用 tenacity 重试，最多等待 60 秒
    │
    └── 初始化插件（Jupyter、AgentSkills、VSCode）
```

端口范围的意义在于：同一台机器上可以同时运行多个 OpenHands 会话，每个会话都有自己的容器和端口，互不干扰。

### 阶段二：执行 Action

```
DockerRuntime.on_event(action)
    │
    ├── 检查 action 是否 runnable
    │   ├── runnable=False → 不需要执行，直接跳过
    │   └── runnable=True → 继续
    │
    ├── 确认模式检查
    │   └── 如果 confirmation_mode=True，等待用户确认
    │
    └── ActionExecutionClient._execute_action(action)
            │
            ├── 序列化 action → JSON
            ├── POST http://localhost:{port}/execute_action
            └── 解析响应 → Observation 对象
```

序列化和反序列化让 Action 可以跨进程传输。容器内的 `action_execution_server` 接收 JSON，执行操作，然后把结果序列化为 Observation JSON 返回。

### 阶段三：清理

```
DockerRuntime.close()
    │
    ├── 发送 POST /stop 给容器内服务
    ├── docker stop {container_name}
    ├── docker rm {container_name}
    └── 释放端口锁
```

容器名格式是 `openhands-runtime-{sid}`，每个会话对应一个唯一容器。应用退出时，通过 `atexit` 钩子确保所有容器被清理。

---

## 5.4 action_execution_server：容器内的执行大管家

`action_execution_server.py` 是运行在 Docker 容器内的 FastAPI 服务，它是真正执行代码的地方。

### 它维护的持久状态

这个服务的关键特性是：**它维护了持久的会话状态**，而不是每次都重新启动。

```
action_execution_server
├── BashSession               单个持久 bash 会话（跨请求共享）
│   ├── libtmux session       基于 tmux 的终端复用
│   ├── 工作目录               在 bash 命令之间保持
│   └── 环境变量               export 的变量在命令之间保持
│
├── BrowserEnv               Playwright 浏览器实例（长期运行）
│   ├── 当前页面 URL
│   └── 页面状态（DOM、截图）
│
└── MCPProxyManager          MCP 工具代理（详见下方）
```

**BashSession 为什么要用 tmux？**

直接执行子进程（`subprocess.run`）有一个问题：每次都是一个全新的 shell，无法保持状态。Agent 执行 `cd /app`，下次执行 `ls`，如果是两个独立的 shell，`ls` 仍然在原来的目录。

tmux 提供了一个持久的终端会话，所有命令在同一个 shell 中运行，工作目录、环境变量、已激活的 Python 虚拟环境都可以跨命令保持。这对软件开发任务至关重要。

### 软超时机制

普通的命令执行有一个棘手问题：某些命令可能运行很长时间（比如 `npm install` 或运行测试套件）。如果等待完成，可能阻塞很久；如果直接超时，又会中断正常操作。

OpenHands 的解决方案是**软超时**：

```
命令开始执行
    │
    │  10 秒后（软超时）
    ▼
如果命令还在运行：
    └── 返回 exit_code=-1（特殊标记）
        content: 当前已有的输出
        ──────────────────────────
        Agent 看到 exit_code=-1，可以：
        1. 继续等待（发送空命令，继续读取输出）
        2. 发送 Ctrl+C 终止进程
        3. 认为这是长期后台任务，继续其他操作
```

这个机制让 Agent 可以优雅地处理长时间运行的命令，而不是傻等或粗暴超时。

---

## 5.5 插件系统：扩展沙箱能力

沙箱的基本能力是 bash 执行。但 Agent 还需要 Python 的交互式执行（有完整的计算环境）、VS Code 集成等能力。这些额外能力通过**插件系统**提供。

### 三种内置插件

**1. JupyterPlugin（Python 交互式执行）**

```
容器内：
├── Jupyter 内核进程（常驻）
└── IPython 会话

使用场景：
Agent 选择 IPythonRunCellAction 而非 CmdRunAction，
当需要：
- 使用 pandas/numpy 等数据科学库
- 在同一 Python 进程内累积状态（变量、导入）
- 处理结构化输出（DataFrame, plots）
```

Jupyter 内核是一个独立进程，通过 ZeroMQ 通信，比 bash 中运行 `python` 更高效，因为不需要每次重新启动解释器和导入库。

**2. AgentSkillsPlugin（扩展工具函数）**

这个插件向 Python 环境注入了一组实用函数（`agent_skills`），比如 `open_file()`、`search_file()` 等。这些函数在 IPython 会话中可以直接使用，给 Agent 提供更高层级的文件操作 API。

**3. VSCodePlugin（开发环境集成）**

启动一个轻量级的 Code Server（VS Code 的浏览器版），让用户可以在同一个容器内打开一个完整的 IDE，直接查看和编辑 Agent 在修改的文件。

### 插件的加载流程

```
Runtime 初始化
    │
    ├── 读取 Agent 的 sandbox_plugins 列表
    │   CodeActAgent.sandbox_plugins = [AgentSkillsRequirement, JupyterRequirement]
    │
    ├── 为每个 PluginRequirement，在容器内执行初始化脚本
    │   AgentSkills: pip install agent_skills, setup Python path
    │   Jupyter: start jupyter kernel in background
    │
    └── 确认所有插件启动成功
```

`sandbox_plugins` 是 Agent 类的类变量，每种 Agent 声明自己需要哪些插件，Runtime 在启动时自动满足这些需求。

---

## 5.6 文件系统：Agent 与宿主机共享工作目录

用户的工作目录（workspace）通过 Docker volume 挂载进容器：

```
宿主机：/path/to/workspace/
    │
    │  Docker volume mount
    ▼
容器内：/workspace/（Agent 的工作目录）
```

这意味着：
- Agent 在容器内写的文件，立即出现在宿主机的 workspace 目录
- 用户在宿主机放的文件，Agent 在容器内可以直接读取
- 即使容器被销毁重建，workspace 目录中的文件仍然保留

Git 相关操作也有特殊处理：Runtime 内置了 `GitHandler`，帮助 Agent 管理 Git 仓库的状态（尤其是在 GitHub/GitLab 集成场景下）。

---

## 5.7 MCP：连接外部世界的协议

**MCP（Model Context Protocol）** 是 Anthropic 2024 年底提出的开放协议，用于标准化 LLM 调用外部工具的方式。可以把它理解为"给 AI 的插件系统"——外部服务实现 MCP 协议，AI 就可以调用它们提供的工具，而不需要为每个外部服务写专门的集成代码。

OpenHands 在容器内运行一个 `MCPProxyManager`，它：

```
MCPProxyManager
├── 连接到配置的 MCP 服务器
│   ├── Stdio 类型：在本地启动子进程（如 filesystem server）
│   └── SSE 类型：连接到远程 HTTP 服务
│
├── 发现并注册每个 MCP 服务器提供的工具
│   每个 MCP 工具都会被加入 Agent 的工具列表
│
└── 代理 Agent 对 MCP 工具的调用
    MCPAction → MCPProxyManager → 对应 MCP 服务器 → MCPObservation
```

这让 OpenHands 可以无缝接入任何实现了 MCP 协议的外部工具，无论是数据库、代码分析工具，还是企业内部系统。

---

## 5.8 LocalRuntime：信任环境下的简化版

如果你完全信任 Agent（比如在受控的测试环境中），可以使用 `LocalRuntime`——它直接在宿主机上运行代码，不启动 Docker 容器。

```
LocalRuntime vs DockerRuntime

DockerRuntime：
  + 完全隔离
  + 容器内崩溃不影响主机
  - 需要 Docker，冷启动慢（10-30秒）
  - 不支持所有浏览器功能

LocalRuntime：
  + 零冷启动时间
  + 可以访问宿主机的所有工具和配置
  - 无隔离，Agent 操作直接影响宿主机
  - 仅适合受信任的开发/测试场景
```

LocalRuntime 也实现了 `action_execution_server` 的所有接口，只是这些操作直接发生在宿主机进程内，而不是通过 HTTP 发给容器。

---

## 5.9 Runtime 选择：系统的可扩展点

Runtime 的可插拔设计是 OpenHands 最重要的扩展点之一。通过配置 `RUNTIME` 环境变量，可以选择不同的执行后端：

```
RUNTIME=docker   → DockerRuntime（默认）
RUNTIME=local    → LocalRuntime
RUNTIME=remote   → RemoteRuntime（连接到云端沙箱服务）
RUNTIME=k8s      → KubernetesRuntime
```

每种 Runtime 都需要实现相同的接口——本质上是"接收 Action，返回 Observation"。这意味着同一个 Agent，可以在笔记本电脑的 Docker 容器里运行，也可以无缝切换到企业 Kubernetes 集群，代码不需要任何改变。

---

下一章，我们来看 Agent 的"记忆管理"问题——当对话越来越长，超出 LLM 的上下文窗口时，系统是如何选择性地遗忘、如何压缩历史，同时保证 Agent 不失去重要的上下文。

---

### 质检报告

**讲解节奏**
- [x] 先讲"为什么需要沙箱"（5.1），再讲"沙箱怎么组织"（5.2），再讲"Docker 沙箱怎么工作"（5.3-5.4），再讲"能力扩展"（5.5-5.7），最后讲"其他选项"（5.8-5.9）

**周边知识**
- [x] 5.1 充分解释了沙箱的动机
- [x] 5.4 解释了为什么用 tmux（持久会话）而非 subprocess
- [x] 5.7 解释了 MCP 协议是什么

**讲透了吗**
- [x] Docker Runtime 完整生命周期（启动→执行→清理）
- [x] action_execution_server 持久状态机制
- [x] 软超时机制（exit_code=-1）完整解释
- [x] 三种插件各自的作用和加载流程
- [x] 文件系统挂载机制
- [x] MCP 集成工作方式

**代码纪律**
- [x] 全章无代码片段（全部用流程图和文字描述）

**流程图准确性**
- [x] 两进程分离架构已在 docker_runtime.py 端口范围和 HTTP 通信中验证
- [x] 插件类型已在 runtime/plugins/__init__.py 中验证（Jupyter, AgentSkills, VSCode）
- [x] sandbox_plugins 类变量已在 codeact_agent.py 中验证
- [x] 端口范围 30000-39999 已在 docker_runtime.py 中验证
- [x] BashSession 使用 libtmux 已在 bash.py 中验证
- [x] MCP Proxy Manager 已在 action_execution_server.py imports 中验证

**过渡自然吗**
- [x] 章头：从"大脑决策"到"执行"
- [x] 章尾引出第六章（记忆管理）
- [x] 5.3 三个阶段之间自然衔接

**准确吗**
- [x] Docker volume 挂载机制（标准 Docker 知识）
- [x] MCP 协议 Anthropic 2024 年底提出（公开信息）
- [x] LocalRuntime 不支持浏览器功能（来自 AgentConfig 注释）
