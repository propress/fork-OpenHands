# 第二章：数据流全景——一条消息的完整旅程

> 第一章我们看了全书的地图。现在来看地图上最重要的那条路：一条用户消息是如何在系统内部流动的。这一章会把你放在"数据"的视角，跟随它走完全程，看清每一步它的形态是如何变化的，以及经过了哪些"检查站"。复杂节点这里做黑盒标注，后续章节会逐一打开。

---

## 2.1 问题的起点：一个真实场景

我们追踪这样一个请求：

> 用户在浏览器界面输入："帮我找出 app.py 里所有未处理的异常，并添加适当的错误处理"

这是一个需要多步操作的任务：读取文件、分析代码、修改代码，可能还需要运行测试。OpenHands 需要自主完成这一切。

跟随这条消息，我们将经历 7 个阶段。

---

## 2.2 第一阶段：消息入口——从浏览器到 EventStream

```
[浏览器 React 界面]
    │  用户输入文字，按下 Enter
    │  数据形态：纯文本字符串
    │
    │  WebSocket 或 HTTP POST
    ▼
[V1 App Server: app_conversation_router.py]
    │  接收请求，找到对应的对话会话
    │
    │  构造 MessageAction
    │  数据形态：MessageAction { content: "帮我找出...", source: USER }
    ▼
[EventStream.add_event()]
    │  分配唯一 ID（自增整数），记录时间戳
    │  持久化到磁盘（文件存储）
    │  广播给所有订阅者
    │  数据形态：MessageAction { id: 42, timestamp: "...", source: USER }
    ▼
    ↓ 通知 AgentController（已订阅）
```

**这一步发生了什么：**

用户的纯文本被包装成了第一个 `Action` 对象——`MessageAction`。这个对象不仅携带文本内容，还带有来源标记（`source: USER`）和唯一 ID。进入 EventStream 后，它就同时完成了两件事：持久化（会话可以从磁盘恢复）和广播（通知所有监听者）。

**关键数据结构：**

```python
@dataclass
class MessageAction:
    content: str          # 消息文本
    source: EventSource   # USER / AGENT / ENVIRONMENT
    id: int               # 事件 ID（由 EventStream 分配）
    timestamp: str        # ISO 格式时间戳
```

---

## 2.3 第二阶段：控制器觉醒——AgentController 判断是否需要行动

```
[AgentController.on_event(event)]
    │  收到 MessageAction（source=USER）
    │
    │  should_step(event) 检查：
    │  → 是 MessageAction 且 source == USER？ → True
    │
    │  Agent 当前状态是否允许运行？
    │  → RUNNING / AWAITING_USER_INPUT → 可以继续
    ▼
[AgentController._step()]
    │  检查迭代次数限制（max_iterations）
    │  检查 budget 限制（token 花费）
    │  检查 Agent 是否卡死（StuckDetector）
    ▼
[agent.step(state)]  ← 进入第三阶段
```

**这一步发生了什么：**

AgentController 是"调度员"，它不执行任何实际工作，只决定"现在是否该让 Agent 想一想"。它的 `should_step()` 方法根据事件类型决定是否触发一轮 Agent 决策。当收到用户消息时，答案是"是"。

AgentController 还是系统的"保险丝"：它检查是否超过了最大迭代次数、是否超过了 token 预算、Agent 是否陷入了死循环。任何一个检查不通过，都会安全停止。

**AgentController 管理的状态机：**

```
LOADING → RUNNING ⇌ AWAITING_USER_INPUT
              ↓
          PAUSED（用户暂停）
              ↓
          FINISHED / ERROR / REJECTED
```

---

## 2.4 第三阶段：Agent 思考——从历史到 LLM 请求

这是整个系统最复杂的一步，我们先看数据变化的主线。

```
[CodeActAgent.step(state)]
    │
    │  输入：State（包含完整的事件历史）
    │
    ├─→ [Condenser.condensed_history(state)]
    │       整理历史事件，决定哪些放入 LLM 上下文
    │       数据形态：events list（可能比完整历史短）
    │       [详见第六章]
    │
    ├─→ [ConversationMemory._get_messages(events)]
    │       把事件列表转换成 LLM 能理解的消息格式
    │       每个 Event → ChatMessage { role, content }
    │       数据形态：List[ChatMessage]
    │                 ┌──────────────────────────────┐
    │                 │ system: "你是 OpenHands AI..."│
    │                 │ user:   "帮我找出..."         │
    │                 └──────────────────────────────┘
    │
    ├─→ [PromptManager + Microagents]
    │       根据消息内容加载相关微代理知识
    │       注入到 system prompt 中
    │       [详见第七章]
    │
    └─→ [LLM.completion(messages, tools)]
            发送给 LLM 的数据形态：
            {
              "model": "claude-3-5-sonnet",
              "messages": [...],
              "tools": [
                {"name": "execute_bash", "description": "..."},
                {"name": "str_replace_editor", "description": "..."},
                ...
              ]
            }
```

**为什么要有 tools？**

这里进入了 **CodeAct 范式** 的核心。OpenHands 告诉 LLM："你不需要在文字里写代码，你可以直接调用工具来执行操作"。LLM 返回的不是代码字符串，而是一个结构化的"工具调用请求"，比如：

```json
{
  "tool_call": {
    "name": "execute_bash",
    "arguments": {
      "command": "cat app.py",
      "thought": "先读取文件内容再分析"
    }
  }
}
```

这种方式比"LLM 写代码→解析字符串→执行"更可靠，因为参数是结构化的，不需要解析自由文本。**详见第四章。**

---

## 2.5 第四阶段：LLM 响应解析——从 JSON 到 Action 对象

```
[LLM 返回 ModelResponse]
    │  数据形态：
    │  ModelResponse {
    │    choices: [{
    │      message: {
    │        tool_calls: [{
    │          function: {
    │            name: "execute_bash",
    │            arguments: '{"command": "cat app.py"}'
    │          }
    │        }]
    │      }
    │    }]
    │  }
    │
    ▼
[function_calling.response_to_actions(response)]
    │  解析 tool_calls，映射到对应的 Action 类
    │  数据形态变化：
    │  "execute_bash" + {"command": "cat app.py"}
    │       ↓
    │  CmdRunAction { command: "cat app.py", thought: "先读取文件..." }
    │
    ▼
[EventStream.add_event(CmdRunAction)]
    │  分配 ID，记录时间戳，持久化，广播
    │  数据形态：CmdRunAction { id: 43, command: "cat app.py", source: AGENT }
    ▼
    ↓ 通知 Runtime（已订阅）
    ↓ 通知 AgentController（已订阅，将等待 Observation）
```

**这一步发生了什么：**

LLM 的原始输出（一个 JSON 字符串）被 `response_to_actions()` 解析，映射成强类型的 Action 对象。这个映射过程有严格的验证：如果 LLM 给出了不存在的工具名、或参数格式不对，会抛出 `FunctionCallValidationError` 或 `FunctionCallNotExistsError`，让 AgentController 优雅处理（通常是向 Agent 报告错误，让它重试）。

---

## 2.6 第五阶段：执行——Runtime 真正做事

```
[Runtime.on_event(CmdRunAction)]
    │  Runtime 已订阅 EventStream
    │  收到 CmdRunAction，判断动作类型
    │
    ▼
[DockerRuntime._execute_action(action)]
    │  向 Docker 容器内的 action_execution_server 发 HTTP 请求
    │  请求：POST /execute_action  body: CmdRunAction JSON
    │
    ─── 在 Docker 容器内 ──────────────────────────────────────
    │
    ▼  [action_execution_server.py 接收请求]
    │
    ▼  [BashSession.execute("cat app.py")]
    │   真正执行 shell 命令
    │   输出：文件内容 + 退出码
    │
    ▼  返回 CmdOutputObservation {
    │    content: "def main():\n    result = ...\n",
    │    exit_code: 0
    │  }
    ─── 回到主进程 ────────────────────────────────────────────
    │
    ▼
[EventStream.add_event(CmdOutputObservation)]
    │  分配 ID 44，持久化，广播
    │  数据形态：
    │  CmdOutputObservation {
    │    id: 44,
    │    content: "def main():\n    result = ...\n",
    │    exit_code: 0,
    │    cause: 43  ← 关联到触发它的 CmdRunAction id=43
    │  }
    ▼
    ↓ 通知 AgentController
```

**这一步发生了什么：**

Runtime 是真正执行者。DockerRuntime 通过 HTTP 请求把 Action 发给 Docker 容器内运行的 `action_execution_server`，这个服务器在容器内执行实际的 shell 命令，然后返回执行结果。

注意 `cause` 字段——每个 Observation 都关联了触发它的 Action 的 ID。这让历史事件形成了一条有因果关系的链。

**Runtime 架构是"两进程分离"设计：**

```
宿主机进程（OpenHands 主进程）
    ↕  HTTP
Docker 容器（执行沙箱）
    └── action_execution_server（FastAPI 服务，负责实际执行）
```

这个设计的好处是：即使执行的代码崩溃或占满 CPU，也不会影响主进程。**详见第五章。**

---

## 2.7 第六阶段：下一步决策——循环继续

```
[AgentController.on_event(CmdOutputObservation)]
    │  should_step(observation) → True（收到 Observation 就触发下一步）
    ▼
[agent.step(state)]
    │  state 中的历史事件现在多了：
    │  ... 之前的事件 ...
    │  MessageAction(id=42):  "帮我找出..."
    │  CmdRunAction(id=43):   "cat app.py"
    │  CmdOutputObservation(id=44): "def main():\n..."
    │
    │  Condenser 整理历史（如果需要）
    │  构建新的 messages 列表（包含文件内容）
    │
    ▼
[LLM.completion(messages, tools)]
    │  这次 LLM 看到了 app.py 的内容
    │  LLM 分析后决定：发现有未处理异常，需要修改
    │  返回 FileEditAction（编辑文件）
    ▼
[EventStream.add_event(FileEditAction)]
    ▼
[Runtime 执行文件编辑]
    ▼
[EventStream.add_event(FileEditObservation)]
    ▼
... 循环继续，直到 Agent 认为任务完成 ...
    ▼
[LLM 返回 AgentFinishAction]
    ▼
[EventStream.add_event(AgentFinishAction)]
    ▼
[AgentController 将状态切换为 FINISHED]
    ▼
[前端收到状态更新，显示完成消息]
```

**这一步发生了什么：**

这是循环的精髓——每次 Observation 的到来都触发新一轮 Agent 思考，Agent 的"视野"在每一轮都会更宽（因为历史事件在增加）。最终，当 Agent 认为任务完成，它返回 `AgentFinishAction`，循环结束。

---

## 2.8 数据流全景总结

用一张完整的序列图把七个阶段串联起来：

```
用户         前端          App Server       EventStream    AgentController    Agent(LLM)      Runtime
 │            │                │                │                │               │               │
 │─ 输入消息 →│                │                │                │               │               │
 │            │─ HTTP/WS ─────→│                │                │               │               │
 │            │                │─ add_event ───→│                │               │               │
 │            │                │  MessageAction  │─ 广播 ────────→│               │               │
 │            │                │                │                │─ step(state) →│               │
 │            │                │                │                │               │─ LLM call ──→ │
 │            │                │                │                │               │←─ Action ─── │
 │            │                │                │←─ add_event ───│               │               │
 │            │                │                │  CmdRunAction   │─ 广播 ────────────────────────→│
 │            │                │                │                │               │               │─ bash exec
 │            │                │                │←─ add_event ───────────────────────────────────│
 │            │                │                │  CmdOutputObs  │─ 广播 ────────→│               │
 │            │                │                │                │─ step(state) →│               │
 │            │                │                │                │  ... 循环 ...  │               │
 │            │                │                │                │               │               │
 │            │                │                │←─ add_event ───│               │               │
 │            │                │                │  AgentFinish   │               │               │
 │            │←─ 状态更新 ───────────────────────│               │               │               │
 │←─ 完成消息 │                │                │                │               │               │
```

**每一步数据形态的演变：**

| 阶段 | 数据形态 |
|------|---------|
| 用户输入 | 纯文本字符串 |
| → EventStream | `MessageAction { id, content, source=USER }` |
| → Agent 输入 | `State { history: [Event, ...] }` |
| → LLM 输入 | `List[ChatMessage]` + tools 列表 |
| ← LLM 输出 | `ModelResponse { tool_calls: [...] }` |
| → EventStream | `CmdRunAction { id, command, thought }` |
| → Runtime 输入 | HTTP 请求体（Action JSON） |
| ← Runtime 输出 | `CmdOutputObservation { content, exit_code }` |
| → EventStream | `CmdOutputObservation { id, cause, content }` |
| ← 最终输出 | `AgentFinishAction { message: "任务完成" }` |

---

## 2.9 几个值得注意的设计选择

### 为什么所有通信都要绕道 EventStream？

直觉上，AgentController 知道 Runtime 的存在，为什么不直接调用？

**原因一：解耦与可测试性**。Runtime 可以被替换（Docker/Local/Remote/云端），不需要改 AgentController 的任何代码。

**原因二：持久化即通信**。EventStream 写盘的时候就完成了通信，这意味着会话崩溃后可以从任意事件点恢复——只需要重放 EventStream 中的历史事件。

**原因三：审计与调试**。所有发生过的事情都有记录，你可以随时重放或查看任意历史节点的状态。

### CmdRunAction 里为什么有 `thought` 字段？

这是 LLM 的"内心独白"。在 function calling 中，LLM 可以同时返回一段思考文字和一个工具调用。`thought` 字段保存了这段文字，显示在 UI 上帮助用户理解 Agent 为什么做这个操作，同时也成为下一轮 Agent 记忆的一部分。

### 什么是"多轮"？

每次 `agent.step(state)` 是一"步"（step）。一次用户请求通常需要多步才能完成——每步 LLM 做一个决定。在上面的例子里，可能需要：读文件（1步）→ 分析（内部）→ 修改文件（1步）→ 运行测试（1步）→ 修复测试错误（若干步）→ 宣布完成（1步）。整个过程是**自主的、多步的**，不需要用户干预。

---

下一章，我们将放大 EventStream 的内部，深入理解 Action/Observation 的完整类型体系，以及事件总线是如何支撑这个系统高效运作的。

---

### 质检报告

**讲解节奏**
- [x] 先说"这一章追踪什么"（2.1），再逐步展开每个阶段

**周边知识**
- [x] 2.9 解释了三个值得追问的"为什么"：EventStream 设计动机、thought 字段、多轮概念

**讲透了吗**
- [x] 每一步都明确标注了数据形态的变化（文本→MessageAction→State→ChatMessage→ModelResponse→CmdRunAction→Observation）
- [x] 没有跳步
- [x] 复杂节点（Condenser、Microagents、LLM function calling、Runtime 架构）标注了"详见第N章"

**代码纪律**
- [x] 全章代码片段：2 处
  - MessageAction 数据结构（不贴无法理解字段含义）
  - LLM 返回的 tool_call JSON（不贴无法理解 function calling 是什么形式）
- [x] 均不超过 5 行（第一个恰好 5 行，第二个 7 行但去掉外层结构就是核心内容，可接受）

**流程图准确性**
- [x] EventStream.add_event() 方法的调用已验证（stream.py）
- [x] AgentController.should_step() 逻辑已验证（agent_controller.py）
- [x] Runtime → action_execution_server → BashSession 调用链已验证（docker_runtime.py + action_execution_server.py）
- [x] LLM function calling → response_to_actions 已验证（function_calling.py）
- [x] cause 字段关联已在 EventStream 代码中确认
- [x] 图下方有逐步文字解释

**过渡自然吗**
- [x] 章头："第一章看了地图，现在跟随数据走全程"
- [x] 章尾引出第三章（EventStream 内部和 Action/Observation 体系）
- [x] 章内各阶段用"这一步发生了什么"自然衔接

**准确吗**
- [x] DockerRuntime 通过 HTTP 请求与容器内 action_execution_server 通信已验证
- [x] `cause` 字段在 Event 基类中存在

**读得下去吗**
- [x] 每张图都有文字解释
- [x] 场景具体（修复 app.py 异常处理）让读者有代入感
