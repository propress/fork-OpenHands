# 第六章：记忆管理——Condenser 与上下文窗口

> 任何 LLM 都有一个根本限制：上下文窗口（context window）。即使是拥有 200K token 上下文的 Claude，在处理复杂的长任务时也可能耗尽。这一章我们看 OpenHands 如何通过 Condenser（压缩器）系统来管理 Agent 的"工作记忆"，让 Agent 能够处理远超上下文限制的长任务。

---

## 6.1 问题：上下文窗口是有限资源

想象 Agent 正在处理一个大型代码库，已经执行了数十个命令、读取了多个文件、进行了多轮对话。此时，事件历史中已经积累了大量内容：

```
系统提示（2K tokens）
+ 第一条用户消息（0.1K）
+ 读取文件操作及内容（5K）
+ bash 命令及输出（8K）
+ 更多文件读取（10K）
+ 更多命令（6K）
...
= 总计 50K+ tokens
```

如果模型的上下文限制是 128K tokens，这还好，但随着任务继续，迟早会超限。

更微妙的问题是：**就算没超限，太长的上下文也会让 LLM 的注意力分散**，早期的内容逐渐被"淡忘"（LLM 在长上下文中往往对中间部分的关注度下降）。

---

## 6.2 Condenser 的核心设计

Condenser 的职责是：**接收完整的事件历史（View），决定哪些保留、哪些压缩/遗忘，返回一个更短的视图**。

### 两个关键数据结构

**View**：事件的有序列表，是 Condenser 的输入和输出单位。

**Condensation**：Condenser 返回的"压缩指令"，告诉系统要遗忘哪些事件（通过 `CondensationAction`）。

Condenser 有两种工作模式，通过 `condensed_history(state)` 方法返回不同类型的结果：

```
Condenser.condensed_history(state)
    │
    ├── 返回 View(events=..., forgotten_event_ids=...)
    │       当前不需要压缩，Agent 使用这个视图继续执行
    │
    └── 返回 Condensation(action=CondensationAction)
            需要压缩，Agent 应该直接返回这个 action
            → AgentController 把它写入 EventStream
            → 下次 step() 时，历史中包含这个 Condensation 事件
            → Condenser 看到这个 Condensation 事件，生成压缩后的 View
```

这是一个**异步的两阶段协议**：
1. Condenser 说"我想压缩，请先记录这个意图"（写 CondensationAction 到事件流）
2. 下一轮，Condenser 用这个意图事件生成实际的压缩视图

为什么要两阶段？因为压缩行为本身也需要被记录到 EventStream 中，这样会话可以被完整重放——重放时同样会经历这个压缩步骤。

---

## 6.3 Condenser 的完整家族

OpenHands 提供了多种 Condenser 实现，可以单独使用也可以组合使用（通过 Pipeline）：

```
Condenser（抽象基类）
│
├── NoOpCondenser               什么都不做，保留全部历史
│   适用：短任务，或测试场景
│
├── RecentEventsCondenser       只保留最近 N 个事件
│   适用：极简场景，不需要长期记忆
│
├── ObservationMaskingCondenser 截断过长的 Observation 内容
│   适用：文件读取或命令输出很长时，截断中间部分
│   不删除事件，只压缩内容长度
│
├── BrowserOutputCondenser      专门处理浏览器输出
│   适用：网页内容通常很长，特殊处理以提取关键信息
│
├── ConversationWindowCondenser 保留最近半数事件（滑动窗口）
│   逻辑：
│   - 永远保留：系统消息、第一条用户消息、Recall 结果
│   - 保留最近约 50% 的事件
│   - 保证 Action-Observation 配对不被拆开
│   默认配置时使用此方案
│
├── AmortizedForgettingCondenser 摊销式渐进遗忘
│   逻辑：
│   - 保留 keep_first 个事件（开头的重要上下文）
│   - 在内容达到阈值时，一次性遗忘一批中间的事件
│   - 效果：内存使用平滑增长，不会突然爆炸
│
├── LLMSummarizingCondenser     使用 LLM 生成摘要再遗忘
│   逻辑：
│   - 达到 max_size 时，让 LLM 生成被遗忘事件的摘要
│   - 把摘要存为 AgentCondensationObservation
│   - 遗忘原始事件，保留摘要
│   特点：损失细节，保留语义摘要
│
├── LLMAttentionCondenser       基于 LLM 注意力的智能选择
│   逻辑：让 LLM 决定哪些事件是"重要的"，有选择性地保留
│
├── StructuredSummaryCondenser  结构化摘要（保留任务状态等结构信息）
│
└── PipelineCondenser           组合多个 Condenser 顺序应用
    例：先 ObservationMasking，再 ConversationWindow
```

### 实践中的典型配置

默认配置（`ConversationWindowCondenserConfig`）：
- 保留系统消息、第一条用户消息
- 保留最近约一半的历史事件
- 触发条件：历史超出阈值

高质量长任务推荐（`LLMSummarizingCondenser`）：
- 使用 LLM 生成被压缩部分的摘要
- 消耗额外 token，但保留了重要的语义信息

---

## 6.4 ConversationWindowCondenser 详解

这是默认方案，值得深入理解其具体逻辑。

### 保护性事件（永远不遗忘）

```
1. SystemMessageAction（系统提示）
   包含 Agent 的全部工具说明和行为规范
   遗忘后 Agent 不知道自己能做什么
   
2. 第一条 MessageAction（source=USER）
   原始任务说明
   遗忘后 Agent 不知道要完成什么任务
   
3. RecallObservation（如果有）
   包含从长期记忆检索到的背景知识
   遗忘后 Agent 缺少重要背景
```

### 滑动窗口逻辑

```
完整历史（100 个事件）：
[0: System][1: UserMsg][2: Recall][3-99: 各种操作]

压缩后（保留约 50 个事件）：
[0: System][1: UserMsg][2: Recall][50-99: 最近的操作]
                                   ^^^^^^^^^^^^^^^^^
                                   最近约半数保留
```

关键约束：**Action-Observation 配对不能拆开**。如果第 48 个事件是 `CmdRunAction`，第 49 个是对应的 `CmdOutputObservation`，不能只保留 49 而丢掉 48——否则 LLM 看到一个没有来源的命令输出，会困惑。因此压缩时会向前多保留一个事件，确保每个 Observation 都有对应的 Action。

---

## 6.5 LLMSummarizingCondenser：用 LLM 来压缩 LLM 的记忆

当任务需要很长时间且语义延续性很重要时，`LLMSummarizingCondenser` 是更好的选择。

### 摘要生成的过程

```
历史达到 max_size（比如 100 个事件）时：

1. 确定要遗忘的事件
   保留前 keep_first 个事件（开头上下文）
   保留最后约一半事件（近期操作）
   要遗忘的：中间那部分

2. 调用 LLM 生成摘要
   输入：要被遗忘的事件列表 + 上一次的摘要（如果有）
   输出：结构化摘要，包含：
   - USER_CONTEXT：用户的核心需求
   - TASK_TRACKING：正在进行的任务和状态
   - COMPLETED_ACTIONS：已完成的重要操作
   - ENVIRONMENT_STATE：环境的当前状态（文件、安装的包等）

3. 把摘要存为 AgentCondensationObservation
   这个事件会被插入到历史的"开头"之后

4. 遗忘原始中间事件
   发出 CondensationAction
```

### 摘要的持续更新

每次新的压缩发生时，旧的 `AgentCondensationObservation` 也会被包含在要摘要的范围内。新摘要会整合旧摘要的内容，形成一个**累积的语义压缩**——越压越精炼，但核心信息被保留。

---

## 6.6 触发条件：什么时候开始压缩

Condenser 并不是时刻在压缩，而是在特定条件下触发：

**主动触发（超出限制前）：** `ConversationWindowCondenser` 等在历史超过一定长度时主动触发。

**被动触发（已经超出）：** 当 LLM API 返回 `ContextWindowExceededError` 时，AgentController 会立即触发一个 `CondensationRequestAction`，强制进行一次压缩。

```
Agent.step() 调用 LLM
    │
    │  LLM 返回 ContextWindowExceededError（上下文太长）
    ▼
AgentController 捕获错误
    │
    │  如果 enable_history_truncation=True
    ▼
EventStream.add_event(CondensationRequestAction)
    │
    ▼
AgentController 让 agent.step() 再次执行
    │
    ▼
Condenser 看到 CondensationRequestAction，立即压缩
    │
    ▼
返回更短的 View，重新尝试 LLM 调用
```

这个机制是"最后防线"——即使 Condenser 没有主动压缩，当真的超限时系统也能自动恢复。

---

## 6.7 长期记忆：Recall 机制

除了上下文窗口管理，OpenHands 还有一个独立的"长期记忆检索"机制，通过 `RecallAction` 实现。

```
Agent 在开始任务时可以发出 RecallAction
    │
    │  recall_type: KNOWLEDGE（全局知识库）
    │  或 recall_type: WORKSPACE（工作区文件）
    │
    ▼
Memory 模块接收 RecallAction
    │
    ├── 检索微代理知识（Microagents，见第七章）
    ├── 扫描工作区文件目录结构
    └── 检索之前对话的相关内容
    │
    ▼
返回 RecallObservation（检索结果）
    │
    ▼
这个 Observation 被永久保护（ConversationWindowCondenser 永不遗忘它）
```

这让 Agent 在任务开始时建立一个"工作记忆基础"，即使后续历史不断压缩，这些重要的背景知识也始终保留在上下文中。

---

## 6.8 全局视角：记忆的三个层次

综合来看，OpenHands 对 Agent 记忆的管理分为三个层次：

```
短期工作记忆（上下文窗口内）
├── 最近的操作历史
├── 当前任务的进展
└── 由 Condenser 管理大小

中期压缩记忆（AgentCondensationObservation）
├── 已被 LLM 摘要的历史操作
├── 保留语义但丢失细节
└── 由 LLMSummarizingCondenser 生成

长期背景记忆（永久保护的事件）
├── 系统提示（工具和规范）
├── 原始用户任务
└── RecallObservation（背景知识）
```

通过这三个层次的协同，Agent 即使在处理需要数百轮操作的复杂任务时，也能保持对任务目标的清醒认知，同时不会因为上下文溢出而崩溃。

---

下一章，我们来看微代理（Microagent）系统——这是 OpenHands 向 Agent 注入"专家知识"的机制，让 Agent 在处理不同类型的任务时，能自动获取相应领域的最佳实践和操作规范。

---

### 质检报告

**讲解节奏**
- [x] 先讲问题（上下文窗口限制），再讲解决方案（Condenser 设计），再讲各种实现

**周边知识**
- [x] 6.1 充分解释了上下文窗口是什么、为什么是问题
- [x] 6.2 解释了两阶段压缩协议的原因（需要事件可回放）
- [x] 6.7 解释了 Recall 机制作为长期记忆的补充

**讲透了吗**
- [x] 所有 Condenser 类型全覆盖并说明适用场景
- [x] ConversationWindowCondenser 的保护逻辑详细解析
- [x] LLMSummarizingCondenser 的摘要生成过程
- [x] Action-Observation 配对保护的细节
- [x] 触发条件（主动 + 被动）都覆盖
- [x] 三层记忆模型

**代码纪律**
- [x] 全章无代码片段（全用文字和流程图）

**流程图准确性**
- [x] Condenser 家族已在 impl/ 目录验证
- [x] ConversationWindowCondenser 的逻辑（保护 SystemMessage、第一条 UserMsg、Recall）已在源码验证
- [x] LLMSummarizingCondenser 的摘要结构（USER_CONTEXT, TASK_TRACKING 等）已在源码中的 prompt 验证
- [x] ContextWindowExceededError → CondensationRequestAction 流程已在 agent_controller.py 验证
- [x] RecallAction 永久保护逻辑已在 ConversationWindowCondenser 源码中验证

**过渡自然吗**
- [x] 章头：从 Runtime 执行转向"执行了太多怎么办"
- [x] 章尾引出第七章（微代理）
- [x] 章内小节之间有自然衔接

**准确吗**
- [x] 所有 Condenser 类型名称已在目录结构验证
- [x] LLM 注意力在长上下文中对中间部分下降（公认的 LLM 行为）
