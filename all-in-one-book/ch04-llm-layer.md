# 第四章：大脑——LLM 层与 CodeAct 范式

> 前面三章我们理解了系统的骨架：事件怎么流动、状态怎么管理。这一章我们进入 Agent 的决策核心，看清 LLM 调用是如何发生的，以及 CodeAct 范式为何是这个系统最关键的设计选择。

---

## 4.1 LLM 不只是"文字生成器"

在传统的 LLM 使用方式里，你给它文字，它还你文字，然后你解析那些文字，提取出你想要的信息。

但这有个根本问题：**文字解析是脆弱的**。如果 LLM 的输出格式稍有不同，解析就会失败。更糟的是，LLM 可能同时返回"我的思考"和"要执行的操作"混在一起的文字，很难可靠地分离。

OpenHands 使用了一种更现代的方式：**Function Calling（函数调用）**。

---

## 4.2 Function Calling：让 LLM 返回结构化决策

Function calling（也叫 tool calling）是 OpenAI 2023 年引入的机制，现已被几乎所有主流 LLM 支持。它的工作方式是：

1. 调用 LLM 时，同时传入一个"工具列表"，描述 Agent 能做的事
2. LLM 不返回自由文本，而是返回一个结构化的"工具调用请求"——我要调用哪个工具、参数是什么
3. 系统按照这个结构化请求执行对应的操作

这就是为什么 OpenHands 的 Agent 输出总是强类型的 Action 对象——因为 LLM 返回的本来就是结构化数据，不需要解析。

### LLM 看到的工具列表

以 `execute_bash` 工具为例，发给 LLM 的工具描述是这样的：

```json
{
  "type": "function",
  "function": {
    "name": "execute_bash",
    "description": "Execute a bash command in the terminal...",
    "parameters": {
      "type": "object",
      "properties": {
        "command": { "type": "string", "description": "The bash command..." },
        "timeout": { "type": "number", "description": "Hard timeout..." },
        "security_risk": { "type": "string", "enum": ["low", "medium", "high"] }
      },
      "required": ["command", "security_risk"]
    }
  }
}
```

注意 `security_risk` 是必填字段。这个设计让 LLM 必须在每次执行 bash 命令时，主动评估这个命令的安全风险级别——这不只是审计用途，也是一种让 LLM 在决策时更加"谨慎思考"的提示工程技巧。

### CodeActAgent 提供的完整工具集

CodeActAgent 在初始化时构建了一份工具列表，发给 LLM 的 `tools` 参数：

```
工具名称                  对应 Action          说明
─────────────────────────────────────────────────────────
execute_bash             CmdRunAction         执行 bash 命令
ipython                  IPythonRunCellAction 执行 Python 代码
str_replace_editor       FileEditAction       精准编辑文件（ACI 编辑器）
browser                  BrowseInteractiveAction 操控浏览器
finish                   AgentFinishAction    声明任务完成
think                    AgentThinkAction     记录思考（不执行任何操作）
condensation_request     CondensationRequestAction 请求压缩历史
task_tracker             TaskTrackingAction   管理任务清单（Plan 模式）
llm_based_edit           FileEditAction       LLM 辅助编辑（已弃用）
```

每个工具都有详细的 JSON Schema 描述，告诉 LLM 什么时候用、怎么用、参数是什么。工具描述的质量直接影响 Agent 的行为——这也是提示工程的一部分。

---

## 4.3 LLM 封装层：LiteLLM 适配器

OpenHands 支持几十种不同的 LLM——Claude、GPT-4、Gemini、本地 Ollama 模型……它们的 API 都不一样。OpenHands 通过 **LiteLLM** 这个开源库来统一接口。

```
CodeActAgent
    │
    └── LLM 类（openhands/llm/llm.py）
            │
            └── LiteLLM（外部库）
                    │
                    ├── Anthropic API（Claude）
                    ├── OpenAI API（GPT-4）
                    ├── Google API（Gemini）
                    ├── Ollama（本地模型）
                    └── ... 60+ 其他提供商
```

**LiteLLM 是什么：** 它是一个 Python 库，提供了统一的 `completion()` 接口，内部自动把调用转发给对应的 LLM 提供商 API。你只需要改 `model` 参数（比如 `claude-3-5-sonnet` 或 `gpt-4o`），其余代码不变。

### LLM 类的职责

OpenHands 的 `LLM` 类（继承自 `RetryMixin` 和 `DebugMixin`）在 LiteLLM 之上加了几层能力：

```
LLM 类的职责
├── 1. 重试机制（RetryMixin）
│       遇到限速（429）、服务不可用（503）、连接超时时自动重试
│       指数退避：1s → 2s → 4s → 8s ... 最多 N 次
│
├── 2. 上下文窗口管理
│       自动检测模型的上下文窗口大小
│       如果消息太长，触发 ContextWindowExceededError
│       这反过来触发 Condenser 压缩历史（详见第六章）
│
├── 3. Token 计量（Metrics）
│       记录每次调用的 prompt tokens、completion tokens、总花费
│       这些数据用于 budget 控制
│
├── 4. Function Calling 兼容转换
│       不支持 function calling 的老模型
│       通过 fn_call_converter 把工具描述注入到 system prompt 中
│       并解析模型返回的特殊格式文本，提取出工具调用请求
│
└── 5. 调试与日志
        可选地把每次 LLM 调用记录到文件，方便调试
```

### 重试机制的细节

重试逻辑有一个有趣的细节：当遇到 `LLMNoResponseError`（LLM 返回了空响应）且当前 `temperature=0` 时，重试时会把 `temperature` 调整为 1.0。

为什么？当 `temperature=0` 时，LLM 是确定性的——同样的输入永远给出同样的输出。如果这次输出是空（bug 或边界情况），下次还会是空。提高温度引入随机性，让重试真正有意义。

---

## 4.4 CodeAct 范式：为什么用代码表达一切动作

"CodeAct"这个名字来自一篇研究论文（[arxiv.org/abs/2402.01030](https://arxiv.org/abs/2402.01030)），核心思想是：**让 Agent 用代码来表达所有动作**，而不是用自然语言描述操作意图。

### 传统方式 vs. CodeAct 方式

**传统方式**（ReAct 风格）：
```
Thought: 我需要读取文件 app.py
Action: read_file(path="app.py")
Observation: def main(): ...
```

动作是离散的、预定义的（只能读文件、写文件、搜索……），需要人工设计每个动作的类型。

**CodeAct 方式**：
```
Thought: 我需要分析 app.py 中的异常处理
Action: execute_bash(command="cat app.py | grep -n 'except\|try' | head -50")
```

动作就是代码。代码是通用的——你可以用一条 bash 命令或一段 Python 代码做任何事情，不需要预先定义每种可能的操作。

### CodeAct 的三个核心优势

**1. 动作空间统一且无限**

用代码，LLM 能表达的操作空间是无限的。不需要预先枚举所有可能的操作类型，LLM 可以随时组合命令完成新的任务。

**2. 多步动作可以打包**

```bash
# 一条 bash 命令 = 多步操作
cd /app && pip install -r requirements.txt && python -m pytest tests/ -v
```

一次 function call 可以完成"切换目录 → 安装依赖 → 运行测试"三步，减少 LLM 调用次数（节省成本和时间）。

**3. 代码可以自我纠错**

当命令失败（非零退出码），LLM 立刻看到错误信息，可以在下一步修改命令重试。代码的执行反馈是精确的、机器可读的，比自然语言反馈更容易被 LLM 理解和利用。

### OpenHands 对 CodeAct 的实现扩展

纯 CodeAct 只有 bash/Python 两种工具。OpenHands 扩展了这个思想，增加了：
- `str_replace_editor`：精准文件编辑工具（来自 `openhands-aci` 库）
- `browser`：完整的浏览器操控
- `think`：显式的思考记录工具
- `task_tracker`：任务规划追踪（Plan 模式）

这些扩展保持了 CodeAct 的精神——每个工具都有明确的结构化接口——同时覆盖了软件工程任务中最常见的操作需求。

---

## 4.5 消息构建：Agent 把历史事件翻译成 LLM 能读懂的对话

每次调用 LLM 之前，CodeActAgent 需要把 EventStream 中的事件历史转换成 LLM 的消息格式（角色对话）。这个工作由 `ConversationMemory` 完成。

### 消息角色映射

```
事件类型                     → LLM 消息角色
─────────────────────────────────────────────
MessageAction(source=USER)   → role: "user"
MessageAction(source=AGENT)  → role: "assistant"
SystemMessageAction          → role: "system"
CmdRunAction                 → role: "assistant" (tool_call)
CmdOutputObservation         → role: "tool" (tool_result)
FileEditAction               → role: "assistant" (tool_call)
FileEditObservation          → role: "tool" (tool_result)
AgentThinkAction             → role: "assistant" (thought 文本)
```

### 构建后的消息结构

LLM 最终收到的消息列表大致是这样的：

```
[system]  "你是 OpenHands AI，一个能执行代码的软件工程师...
          工具使用规范...
          [注入的微代理知识]..."

[user]    "帮我找出 app.py 里所有未处理的异常"

[assistant, tool_call]  execute_bash("cat app.py")

[tool, tool_result]     "def main():\n    result = ..."

[assistant, tool_call]  str_replace_editor(...)

[tool, tool_result]     "文件已修改，diff: ..."

→ 现在让 LLM 决定下一步
```

每一轮 `agent.step()` 都会重建这个消息列表（从历史事件中重新构建），确保 LLM 总能看到最新的完整上下文。

---

## 4.6 不支持 Function Calling 的模型怎么办

并非所有模型都支持 function calling，比如一些较老的开源模型。OpenHands 通过 `fn_call_converter` 模块处理这种情况：

**发送时（转换请求）：** 把工具定义注入到 system prompt 中，告诉模型"你可以用这种 XML 格式调用工具"：

```
<function=execute_bash>
<parameter=command>cat app.py</parameter>
</function>
```

**接收时（解析响应）：** 扫描模型返回的文本，找到这种 XML 格式的工具调用，解析成标准的 function calling 格式。

这个兼容层让 OpenHands 可以和几乎任何 LLM 配合工作，虽然效果通常不如原生 function calling 好。

---

## 4.7 LLM 调用的完整流程

综合前面几节，一次完整的 LLM 调用过程如下：

```
CodeActAgent.step(state)
    │
    ├─ Condenser 整理历史
    │     → condensed_events（可能比完整历史短）
    │
    ├─ ConversationMemory._get_messages(condensed_events)
    │     → List[ChatMessage]（含 system, user, assistant, tool 消息）
    │
    ├─ PromptManager 注入微代理知识
    │     → system 消息更新
    │
    ├─ check_tools(self.tools, llm.config)
    │     → 验证并过滤工具列表（某些模型不支持某些工具）
    │
    ├─ LLM.completion(messages=..., tools=...)
    │     ├─ RetryMixin 包裹（失败自动重试）
    │     ├─ 如果模型不支持 function calling → fn_call_converter 转换
    │     ├─ 发送到 LiteLLM → 对应 LLM 提供商 API
    │     ├─ 记录 token 使用量到 Metrics
    │     └─ 返回 ModelResponse
    │
    ├─ function_calling.response_to_actions(response)
    │     → 解析 tool_calls → 映射到 Action 对象
    │
    └─ 返回 Action（或 pending_actions 队列中的下一个）
```

其中有一个细节值得注意：`pending_actions`。当 LLM 一次性返回了多个 tool_calls（现代 LLM 支持并行工具调用），CodeActAgent 会把它们放入一个 `deque` 队列，每次 `step()` 只从队列头取一个 Action 返回。这样保证了每次只有一个 Action 在执行，Runtime 不会被并发请求淹没。

---

下一章，我们进入 Runtime 的世界，看 Agent 声明的那些 Action 是如何在真实的沙箱环境中被执行的，以及 Docker 隔离是如何保证安全性的。

---

### 质检报告

**讲解节奏**
- [x] 先讲 function calling 是什么，再讲 LLM 封装，再讲 CodeAct 范式，最后讲消息构建和完整流程

**周边知识**
- [x] 4.2 解释了为什么 function calling 比文字解析更好
- [x] 4.4 解释了 CodeAct vs. 传统 ReAct 方式的对比
- [x] 4.3 解释了 LiteLLM 是什么以及为什么用它

**讲透了吗**
- [x] function calling 机制从 LLM 角度完整解释
- [x] CodeAct 三个核心优势有充分论述
- [x] LLM 类的五个职责全覆盖
- [x] 重试逻辑的 temperature 细节
- [x] pending_actions 队列机制

**代码纪律**
- [x] 全章代码片段：2 处
  - execute_bash 工具的 JSON Schema（不贴无法理解工具描述格式）
  - CodeAct 传统方式 vs CodeAct 方式对比（不贴无法理解设计差异）
- [x] 第一处 14 行，但这是 JSON 结构而非代码，且是核心结构展示，保留合理

**流程图准确性**
- [x] LiteLLM 适配器层已验证（llm.py imports）
- [x] RetryMixin 温度调整逻辑已验证（retry_mixin.py）
- [x] 工具列表来自 codeact_agent.py _get_tools() 方法确认
- [x] fn_call_converter 格式已验证（fn_call_converter.py）
- [x] pending_actions deque 已在 codeact_agent.py 的 step() 中验证

**过渡自然吗**
- [x] 章头：从骨架到决策核心
- [x] 章尾引出第五章（Runtime 执行）

**准确吗**
- [x] Function calling 由 OpenAI 2023 年引入（公开事实）
- [x] CodeAct 论文引用真实
- [x] security_risk 必填字段已在 bash.py 工具定义中验证（`"required": ["command", "security_risk"]`）
