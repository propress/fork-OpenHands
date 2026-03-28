# 第七章：微代理——动态知识注入机制

> 前六章我们理解了系统的"骨骼"（EventStream）、"肌肉"（Runtime）、"大脑"（LLM）和"记忆"（Condenser）。这一章讲的是系统的"知识库"——Microagent（微代理）系统，它让 Agent 在面对不同任务时，能自动获取相关领域的专家知识和最佳实践。

---

## 7.1 问题：通用 Agent 的专业局限

一个通用 Agent 的系统提示是固定的：工具怎么用、基本工作原则……但对于专门的任务，这还不够。

比如：
- 处理 Python 代码时，需要了解 PEP 8 规范、常用测试框架
- 修改某个特定仓库时，需要了解这个仓库的编码规范和架构约定
- 处理 GitHub Issue 时，需要了解提 PR 的流程

把所有这些专业知识都塞进主系统提示，会让提示无限膨胀，而且大多数时候那些知识都用不上（处理 Python 代码时不需要 Ruby 的知识）。

**Microagent 系统的解法：按需加载，触发时注入。**

---

## 7.2 什么是 Microagent

Microagent（微代理）不是一个独立运行的 Agent，而是一个 **Markdown 文件**。它包含：
- **Frontmatter（YAML 头部）**：元数据，包含触发条件
- **正文**：要注入给 Agent 的知识内容

当特定条件满足时（比如用户消息包含触发词），这个 Markdown 文件的内容会被注入到 Agent 的系统提示中，让 Agent 临时获得这块领域的专家知识。

一个典型的 Microagent 文件长这样：

```
---
name: github
type: knowledge
triggers:
- github
- pull request
- PR
---
# GitHub 操作指南

## 提交 Pull Request
1. 先 fork 仓库
2. 创建功能分支
3. 提交代码后，使用 gh pr create 命令...

## 常见操作
- 查看 PR 状态：gh pr status
...
```

---

## 7.3 三种 Microagent 类型

**KnowledgeMicroagent（知识型）**

由触发词（triggers）激活。当用户消息中包含指定关键词时，这个 Microagent 的内容被注入。

```
triggers: [github, pull request, PR]
→ 用户输入包含 "github" 或 "PR"
→ 内容注入系统提示
```

适合：语言最佳实践、框架使用指南、工具操作规范。

**RepoMicroagent（仓库型）**

**始终激活**，不需要触发词。通常存放在被操作仓库的 `.openhands/microagents/repo.md` 文件中，包含这个仓库特有的规范和约定。

```
.openhands/microagents/repo.md → 始终注入，优先级高
.cursorrules → 自动识别为仓库型微代理（兼容 Cursor 编辑器配置）
agents.md / agent.md → 自动识别
```

适合：仓库架构说明、团队编码规范、项目特有工作流程。

**TaskMicroagent（任务型）**

需要用户输入额外参数才能激活。不仅注入知识，还可以配置特定的工具（通过 `mcp_tools`）。这是最新的扩展，让 Microagent 从"只注入文本"演进到"也能带上工具"。

---

## 7.4 Microagent 的加载与触发流程

```
[系统启动时]
    │
    └── load_microagents_from_dir()
        ├── 扫描公共知识库目录（microagents/）
        ├── 扫描仓库本地目录（.openhands/microagents/）
        └── 解析每个 Markdown 文件
            ├── 读取 YAML frontmatter
            ├── 根据 type 字段实例化对应的类
            └── 建立触发词索引

[每次 agent.step() 时]
    │
    └── PromptManager.get_system_message()
        │
        ├── 所有 RepoMicroagent → 始终包含
        │
        └── 对每个 KnowledgeMicroagent，检查 match_trigger(最新用户消息)
            └── 用户消息包含触发词？
                ├── 是 → 把这个 Microagent 的内容加入系统提示
                └── 否 → 跳过
```

触发匹配逻辑非常简单：`trigger.lower() in message.lower()`，也就是不区分大小写的字符串包含匹配。没有语义理解，但简单、快速、可预期。

---

## 7.5 Microagent 如何进入系统提示

PromptManager 使用 Jinja2 模板系统来构建最终的系统提示，Microagent 内容通过专门的模板插槽注入：

```
系统提示的组成（简化）：
┌────────────────────────────────────────────────┐
│  基础提示（system_prompt.j2）                    │
│  - Agent 角色定义                               │
│  - 工具使用说明                                 │
│  - 基本工作原则                                 │
├────────────────────────────────────────────────┤
│  微代理知识注入（microagent_info.j2 模板）        │
│  ┌──────────────────────────────────────────┐  │
│  │  [RepoMicroagent: 仓库规范]              │  │
│  │  "本仓库使用 Poetry 管理依赖..."          │  │
│  ├──────────────────────────────────────────┤  │
│  │  [KnowledgeMicroagent: github]           │  │
│  │  "GitHub 操作指南..."                    │  │
│  └──────────────────────────────────────────┘  │
├────────────────────────────────────────────────┤
│  运行时信息（additional_info.j2）                │
│  - 当前日期                                     │
│  - 工作目录                                     │
│  - 可用的主机地址                               │
└────────────────────────────────────────────────┘
```

每次 `agent.step()` 都会重新构建系统提示。这意味着：
- 如果对话中途出现了 GitHub 相关词汇，下一轮的系统提示就会多了 GitHub 知识
- 如果切换了仓库（Recall 系统加载了新的仓库信息），新仓库的 RepoMicroagent 也会注入

这种**动态性**是 Microagent 系统的核心价值之一。

---

## 7.6 仓库型 Microagent 的高级能力

RepoMicroagent 不只是注入文本，还有一些高级功能：

**兼容第三方格式：**
`.cursorrules`（Cursor AI 编辑器的配置文件）、`agents.md` 等文件会被自动识别为 RepoMicroagent。这让用有 Cursor 配置的项目可以直接被 OpenHands 利用，无需修改。

**通过 Recall 动态加载：**
当 Agent 被分配到一个仓库任务时（比如通过 GitHub Issue），系统会通过 `RecallAction` 主动检索并加载这个仓库的 Microagent。相关内容会出现在 `RecallObservation` 中，被永久保护（第六章中提到的）。

---

## 7.7 为什么不用 RAG？

熟悉 AI 系统的读者可能会问：为什么用简单的关键词匹配，而不用向量嵌入 + 语义检索（RAG）？

**RAG（Retrieval-Augmented Generation）** 是一种将外部知识库向量化，然后根据查询语义相似度检索相关片段的技术。它理论上更精确——可以找到"语义相关"但词语不同的内容。

OpenHands 选择简单关键词匹配的原因：

1. **知识库规模小**：软件工程领域的 Microagent 数量有限（几十个），全部触发词列表不大，简单匹配足够
2. **确定性和可调试性**：关键词匹配完全可预测，容易理解和排查；语义匹配可能产生意外的"不该触发的知识被注入"
3. **零延迟**：不需要调用嵌入模型或向量数据库，不引入新的依赖
4. **显式控制**：Microagent 作者可以精确控制哪些词触发，而不是依赖模型判断

这是一个"够用原则"的体现：在问题规模不需要复杂方案时，选择最简单、最可靠的实现。

---

## 7.8 Microagent 系统的扩展性

Microagent 系统有两个扩展维度：

**纵向：添加新知识**
任何人都可以在 `.openhands/microagents/` 目录下创建 `.md` 文件，立即为 OpenHands 添加针对这个仓库的专门知识。这是无代码扩展——只需要写 Markdown。

**横向：带工具的 Microagent（TaskMicroagent）**
通过在 frontmatter 中指定 `mcp_tools`，Microagent 可以引入新的 MCP 工具。这让专家知识和专门工具可以作为一个包一起分发，未来可能演化成"技能包"。

---

到这里，我们已经把 OpenHands 的核心机制——事件系统、代理循环、LLM 决策、执行沙箱、记忆管理、知识注入——全部深度解析了一遍。下一章，我们转换视角，通过阅读项目历史，理解这个系统是如何从最初的原型一步步演进成现在这个样子的。

---

### 质检报告

**讲解节奏**
- [x] 先讲问题（通用 Agent 的专业局限），再讲方案（Microagent），再讲各类型，再讲工作流程

**周边知识**
- [x] 7.1 充分解释了"为什么需要 Microagent"
- [x] 7.7 解释了为什么不用 RAG（背景知识 + 设计决策）

**讲透了吗**
- [x] 三种 Microagent 类型及各自场景
- [x] 触发流程完整说明
- [x] 系统提示的组成结构
- [x] 动态注入的意义
- [x] 扩展性两个维度

**代码纪律**
- [x] 全章代码片段：1 处（Microagent 文件格式示例，不贴读者不知道 frontmatter 是什么格式）

**流程图准确性**
- [x] 三种 Microagent 类型已在 types.py 中验证（KNOWLEDGE, REPO_KNOWLEDGE, TASK）
- [x] match_trigger 逻辑已在 microagent.py 中验证（trigger.lower() in message.lower()）
- [x] PromptManager 使用 Jinja2 模板已在 utils/prompt.py 中验证
- [x] load_microagents_from_dir 已在 microagent.py 中确认
- [x] .cursorrules 和 agents.md 的兼容性已在 PATH_TO_THIRD_PARTY_MICROAGENT_NAME 中验证

**过渡自然吗**
- [x] 章头：前六章覆盖了系统骨架，这章讲知识库
- [x] 章尾引出第八章（项目演进史）
- [x] 章内各节之间有自然衔接

**准确吗**
- [x] Microagent 是 Markdown 文件（frontmatter + 正文）已在代码中验证
- [x] 触发匹配使用 frontmatter 中的 triggers 字段已验证
- [x] RAG 的描述准确（向量嵌入 + 语义检索）
