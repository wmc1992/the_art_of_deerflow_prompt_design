# 第 2 章：Prompt 工程核心原则

## 本章导读

本章从 DeerFlow 的全部 Prompt 中归纳出 7 条核心设计原则。这些原则不是凭空总结的抽象教条，而是有具体 Prompt 作为证据的工程决策。每条原则配有：**原则定义 → 在 DeerFlow 中的体现 → 设计动机 → 反面教材（如果有）**。

第 11 章的"七大设计模式"是本章原则的细化版本，包含更多迁移指导。如果你时间紧张，可以先读本章建立框架，再按需参考第 11 章。

---

## 原则一：前缀缓存优先（Prefix Cache First）

**定义**：系统 Prompt 保持内容完全静态，不嵌入任何随请求变化的内容（如当前时间、用户记忆、会话状态）。动态内容通过独立的消息对象注入。

### 在 DeerFlow 中的体现

在 `DynamicContextMiddleware` 的实现中，动态内容的注入方式是：

```python
# 不修改系统 Prompt
# 而是向消息链追加一条 HumanMessage
dynamic_content = f"<system-reminder>\n{date_info}\n{memory_info}\n</system-reminder>"
messages = [HumanMessage(content=dynamic_content)] + existing_messages
```

系统 Prompt 模板（P-01）中**没有任何**涉及运行时状态的占位符。即使是 `{soul}` 这样的占位符，也是在 Agent **构建时**（`make_lead_agent()`）填充的，而不是在每次请求时动态替换。

对比一下这个常见的反模式：
```python
# ❌ 反模式：每次请求都修改系统 Prompt
system_prompt = f"""
You are a helpful assistant.
Current date: {datetime.now().strftime('%Y-%m-%d')}
User memory: {load_user_memory(user_id)}
"""
```

DeerFlow 的做法：
```python
# ✅ DeerFlow 方式：系统 Prompt 静态，动态内容走 HumanMessage
# 系统 Prompt 在 Agent 构建时固定
system_prompt = STATIC_SYSTEM_PROMPT  # 不变

# 每次请求时动态注入，不修改系统 Prompt
injected = HumanMessage(f"<system-reminder>{date}\n{memory}</system-reminder>")
```

### 设计动机

大多数 LLM 提供商（OpenAI、Anthropic、Google）都实现了前缀缓存（Prefix KV Cache）机制：如果相同的 token 序列出现在多次请求的开头，这部分的 KV 缓存可以被复用，显著降低首 token 延迟（TTFT）和计算成本。

系统 Prompt 通常是每次请求 token 序列的前几百到几千个 token。如果系统 Prompt 保持静态：
- 100 个并发用户 × 4,000 token 的系统 Prompt = 缓存命中 100 次 = 计算量约等于 1 次
- 如果系统 Prompt 动态变化（每个用户不同） = 100 次完整计算

这个差异在高并发场景下是数量级的成本差异。

### 额外的工程价值

前缀缓存优先原则除了降低成本，还带来了另一个好处：**可测试性**。静态系统 Prompt 可以在构建时被完整地序列化、记录、版本控制和对比。你知道系统 Prompt 的每个版本对应什么内容，可以精确追踪模型行为的变化来源。

如果系统 Prompt 包含动态内容，每次的实际 Prompt 都可能不同，行为的不一致性很难定位原因。

---

## 原则二：XML 标签锚定结构（XML Tag Anchoring）

**定义**：使用 XML 标签（`<role>`、`<thinking_style>` 等）将系统 Prompt 切割为有语义边界的功能单元，而非使用 Markdown 标题或纯文本段落。

### 在 DeerFlow 中的体现

P-01 的整体结构：

```
<role>...</role>

{soul}

{self_update_section}

<thinking_style>...</thinking_style>

<clarification_system>...</clarification_system>

{skills_section}

{deferred_tools_section}

{subagent_section}

<working_directory>...</working_directory>

<response_style>...</response_style>

<citations>...</citations>

<critical_reminders>...</critical_reminders>
```

这 8 个 XML 标签（加上 `{soul}` 的 `<soul>` 标签）构成了系统 Prompt 的骨架。每个标签都有明确的、互不重叠的职责范围：

| 标签 | 职责范围 |
|------|---------|
| `<role>` | 基础身份声明 |
| `<soul>` | 个性特质（可选，自定义 Agent 专用）|
| `<thinking_style>` | 内部推理行为规范 |
| `<clarification_system>` | 何时/如何向用户请求澄清 |
| `<skill_system>` | 技能发现与加载规则 |
| `<working_directory>` | 文件系统访问规范 |
| `<response_style>` | 输出格式与语气 |
| `<citations>` | 信息来源引用规范 |
| `<critical_reminders>` | 高优先级规则汇总 |

### 设计动机

现代 LLM（尤其是 Claude）被训练为能够识别 XML 标签的语义边界，并对标签内容给予与标签语义匹配的注意力。将 `<clarification_system>` 包裹在标签中，相当于告诉模型：**这个区块的内容是关于澄清行为的，当触发澄清相关的决策时，这里的规则优先生效**。

相比之下，使用 Markdown 标题：
```markdown
## Clarification Rules
...
## Working Directory
...
```

在 LLM 的注意力分配上，Markdown 标题和正文之间的语义距离远不如 XML 标签清晰。XML 标签在模型的表征空间中构成了更强的"注意力锚点"。

另一个重要好处是**条件注入的干净性**。当 `subagent_enabled=False` 时，`{subagent_section}` 被替换为空字符串，整个 `<subagent_system>...</subagent_system>` 块（见 P-02）就完整消失了。这比在 Prompt 中写"如果没有子 Agent 能力则忽略以下规则"要干净得多——模型不需要处理一段永远为假的条件分支。

---

## 原则三：命名场景模式（Named Scenario Pattern）

**定义**：将模型需要识别的情境抽象为有名字的类别（而非描述性规则），并为每个类别提供明确的定义、示例和对应操作。

### 在 DeerFlow 中的体现

P-01 的 `<clarification_system>` 段落没有写"当遇到不清楚的情况时向用户提问"，而是定义了 5 种具名场景：

```
1. Missing Information (missing_info)
2. Ambiguous Requirements (ambiguous_requirement)
3. Approach Choices (approach_choice)
4. Risky Operations (risk_confirmation)
5. Suggestions (suggestion)
```

> **中文附注**：
>
> 1. 缺失信息（`missing_info`）
> 2. 模糊需求（`ambiguous_requirement`）
> 3. 方案选择（`approach_choice`）
> 4. 风险操作（`risk_confirmation`）
> 5. 建议（`suggestion`）

每个场景都有：
- 正式名称（英文标识符，出现在代码参数中）
- 自然语言描述
- 2 个具体示例（"Example: User says X but doesn't specify Y"）
- 明确的行动指令（"REQUIRED ACTION: Call ask_clarification to..."）

同样的模式出现在记忆系统（P-10）中，事实被分为 6 个类别：

```
- preference（偏好）
- knowledge（知识）
- context（背景）
- behavior（行为）
- goal（目标）
- correction（纠正）
```

### 设计动机

这个模式解决了一个微妙但重要的问题：**模型的泛化能力是把双刃剑**。

当你写"遇到不清楚的情况问用户"时，模型会用它理解的"不清楚"来泛化这条规则。但"不清楚"可以有无数种解释：信息缺失是"不清楚"，技术方案有多种选择也是"不清楚"，一个危险操作的确认需求也算"不清楚"。

如果这三种情况对应不同的处理优先级（危险操作 > 信息缺失 > 方案选择），模糊的规则就无法传递这种优先级差异。

命名场景模式通过两步解决这个问题：

1. **用分类替代泛化**：给模型一个确定的分类框架，而非让它自由泛化
2. **用示例消除歧义**：每个类别的具体示例消除了类别定义本身的多义性

`missing_info` 和 `ambiguous_requirement` 之间的区别是微妙的（前者是信息根本不存在，后者是信息存在但解读不唯一），但通过例子就能区分。这种"定义 + 例子"的组合，比任何程度的形式化描述都更有效。

---

## 原则四：后果驱动约束（Consequence-Based Constraints）

**定义**：在描述禁止行为时，不只说"不要做X"，而是解释"如果做X，会发生Y后果"——用后果的严重性来传递约束的重要性。

### 在 DeerFlow 中的体现

**P-01 子 Agent 段落（P-02）中的并发限制**：

```
⛔ HARD CONCURRENCY LIMIT: MAXIMUM {n} `task` CALLS PER RESPONSE. THIS IS NOT OPTIONAL.
- Each response, you may include at most {n} `task` tool calls. Any excess calls are
  silently discarded by the system — you will lose that work.
```

> **中文附注**：
>
> ⛔ **硬性并发限制：每次回复最多 {n} 个 `task` 调用。这不是可选项。**
> - 每次回复，你最多可以包含 {n} 个 task 工具调用。超出的调用会被系统**静默丢弃**——你会失去那些工作。

不是说"最多调用 {n} 次"，而是加了"`silently discarded` — you will **lose that work**"。这个后果描述（工作会被悄悄丢弃）比单纯的数字限制有力得多。

**P-01 `<clarification_system>` 中的执行中断说明**：

```
After calling ask_clarification, execution will be interrupted automatically
```

> **中文附注**：
>
> 调用 ask_clarification 后，执行将被自动中断。

不是"调用后等待用户回应"，而是"执行将自动中断"——这传递了一个关键信息：这不是礼貌性的暂停，而是系统级的强制中断。

**P-22 `task` 工具描述**：

```
When NOT to use this tool:
- Simple, single-step operations (use tools directly)
- Tasks requiring user interaction or clarification
```

> **中文附注**：
>
> 不应使用此工具的情况：
> - 简单的单步操作（直接使用工具）
> - 需要用户交互或澄清的任务

通过明确"不应用在哪里"（而非只说"应用在哪里"）来防止工具被滥用。工具的误用往往来自边界不清，而不是使用者不了解工具能做什么。

### 设计动机

基于权威的约束（"你必须..."、"禁止..."）依赖模型对权威的服从，这种服从在上下文压力下可能失效。基于后果的约束（"如果你这样做，结果是..."）则触发模型的因果推理能力——模型会将后果纳入自己的决策计算，而不是单纯靠"服从规则"来约束行为。

一个实用的测试：把你的约束规则念给不了解系统的人听，如果他们会问"为什么不行？"，那这条规则就需要加上后果描述。

---

## 原则五：双向边界定义（Bidirectional Boundary Definition）

**定义**：对于工具和技能，同时明确定义"何时使用（When to Use）"和"何时不使用（When NOT to Use）"，而非只描述能力范围。

### 在 DeerFlow 中的体现

P-22（`task` 工具描述）：
```
When to use this tool:
- Complex tasks requiring multiple steps or tools
- Tasks that produce verbose output
...

When NOT to use this tool:
- Simple, single-step operations (use tools directly)
- Tasks requiring user interaction or clarification
```

> **中文附注**：
>
> 何时使用此工具：
> - 需要多个步骤或工具的复杂任务
> - 会产生大量输出的任务
> ……
>
> 何时不使用此工具：
> - 简单的单步操作（直接使用工具）
> - 需要用户交互或澄清的任务

P-21（`ask_clarification` 工具描述）：
```
Use this tool when you encounter situations where you cannot proceed
without user input:
- Missing information
- Ambiguous requirements
...
Best practices:
- Ask ONE clarification at a time for clarity
- Do not make assumptions when clarification is needed
```

> **中文附注**：
>
> 当遇到无法在没有用户输入的情况下继续的情形时，使用此工具：
> - 信息缺失
> - 需求模糊
> ……
>
> 最佳实践：
> - 一次只提出**一个**澄清问题，保持清晰
> - 需要澄清时，不得擅自做假设

P-18（通用子 Agent 描述）：
```
Use this subagent when:
- The task requires both exploration and modification
...
Do NOT use for simple, single-step operations.
```

> **中文附注**：
>
> 在以下情况使用此子 agent：
> - 任务同时需要探索和修改
> ……
>
> 不要用于简单的单步操作。

整个 P-02 的子 Agent 编排段落也是以"何时用/何时不用"作为骨架。

同样的模式出现在所有 22 个 SKILL.md 文件中，几乎每一个都包含"When to Use This Skill"和隐式的不适用场景说明。

### 设计动机

工具滥用的根本原因是**使用边界模糊**。当模型只知道"工具 X 可以做 Y"，它会在每次看到 Y 相关的任务时都倾向于使用 X，即使在某些情况下直接完成任务效率更高。

"When NOT to Use"的核心价值是**排除法**：在模型激活工具决策时，它会先经过一个过滤步骤——"这个情况是否在禁用范围内？"这个过滤步骤可以阻断很多"惯性使用"的情况（例如：每次文本任务都启动子 Agent，即使任务只需要一次 LLM 调用）。

---

## 原则六：Token 预算意识（Token Budget Awareness）

**定义**：Prompt 设计需要考虑 token 成本，通过懒加载、条件注入和内容分级来控制初始上下文规模。

### 在 DeerFlow 中的体现

**懒加载技能**：P-06 中只在系统 Prompt 里放技能元数据（每个约 25 token），完整 SKILL.md 按需加载（约 1500-3000 token）。22 个技能可节省约 30,000-50,000 token 的固定开销。

**条件注入**：P-02（子 Agent 段落，约 1000 token）、P-04（自更新段落，约 200 token）、P-07（延迟工具列表）、P-08（TodoList，约 350 token）均为条件注入，只在对应功能启用时才出现在系统 Prompt 中。

**记忆 Token 预算硬限制**：
```python
# DynamicContextMiddleware 中的硬性限制
MAX_MEMORY_TOKENS = 2000  # 记忆注入不超过 2000 token
```

**延迟工具的惰性绑定**：
- 系统 Prompt 中只有工具名称列表（P-07，每个工具约 5 token）
- 工具的完整 schema（每个约 200-500 token）在模型调用 `tool_search` 后才绑定到模型
- 对于有 20+ MCP 工具的配置，这可以节省 4,000-10,000 token

### 设计动机

Token 成本在实际系统中有三个维度：
1. **货币成本**（按 token 计费）
2. **延迟成本**（更长的上下文 = 更高的 TTFT）
3. **质量成本**（超长上下文中模型对早期内容的注意力会衰减）

工业系统中往往第三个维度最被低估。一个塞满了大量信息的系统 Prompt，并不会让模型更"聪明"——相反，它可能导致模型对真正重要的规则注意力不足，因为需要关注的东西太多了。

懒加载设计的哲学是：**只在需要时加载，让上下文窗口始终聚焦在当前任务的相关内容上**。

---

## 原则七：层次化指令密度（Hierarchical Instruction Density）

**定义**：最重要的规则放在最显眼的位置（独立标签、粗体、特殊格式），次要规则以较低密度呈现，避免所有内容"等权重"地堆砌。

### 在 DeerFlow 中的体现

**`<critical_reminders>` 段落**：P-01 的最后一个标签是对最高优先级规则的汇总：

```
<critical_reminders>
- **Clarification First**: ALWAYS clarify...
- **Skill First**: Always load the relevant skill...
- Progressive Loading...
- Output Files...
</critical_reminders>
```

> **中文附注**：
>
> `<critical_reminders>`（关键提醒）
> - **澄清优先**：始终澄清……
> - **技能优先**：始终加载相关技能……
> - 渐进加载……
> - 输出文件……
> `</critical_reminders>`

这些规则并不是全新的内容——它们在前面的段落中都有详细展开。`<critical_reminders>` 的作用是**重复强调**：通过在 Prompt 末尾再次列出最重要的规则，确保它们在模型生成响应前是"新鲜"的（距离生成点最近的内容通常有更高的注意力权重）。

**❌/✅ 对比格式**：`<clarification_system>` 中的 STRICT ENFORCEMENT 段落：
```
- ❌ DO NOT start working and then ask for clarification mid-execution
- ❌ DO NOT skip clarification for "efficiency"
...
- ✅ Analyze the request in thinking → Identify unclear aspects → Ask BEFORE any action
```

> **中文附注**：
>
> - ❌ 不得先开始工作再在执行中途要求澄清
> - ❌ 不得以"效率"为由跳过澄清
> ……
> - ✅ 在思考中分析请求 → 识别不清晰之处 → 在任何行动前询问

这种视觉对比格式降低了错误行为和正确行为之间的混淆概率。

**分层的 ⛔/‼️/✅ 信号**：P-02 中用 ⛔ 标记硬性限制，与普通规则在视觉上区分开来。

### 设计动机

把所有规则写成平铺的 bullet list，会导致模型在分配注意力时无法优先处理关键规则。分层设计通过视觉差异（粗体、特殊符号、独立标签）告诉模型"这里特别重要"，触发模型对关键规则更高的激活强度。

`<critical_reminders>` 的末尾放置策略，利用了 LLM 的"近因效应"——距离生成点越近的内容，对最终输出的影响越直接。把最重要的规则放在 Prompt 末尾是一个经过实践验证的技巧。

---

## 本章小结：七条原则对照表

| 编号 | 原则名称 | 核心思想 | 主要体现 Prompt |
|------|---------|---------|---------------|
| 1 | 前缀缓存优先 | 系统 Prompt 完全静态，动态内容走 HumanMessage | P-01, DynamicContextMiddleware |
| 2 | XML 标签锚定结构 | XML 标签创造语义边界和注意力锚点 | P-01（8 个 XML 标签）|
| 3 | 命名场景模式 | 用具名分类替代模糊的泛化描述 | P-01 澄清系统、P-10 记忆分类 |
| 4 | 后果驱动约束 | 描述违反规则的具体后果，而非只说禁止 | P-02 并发限制、P-21 中断说明 |
| 5 | 双向边界定义 | 同时明确何时用和何时不用 | P-21、P-22、P-17、P-18 |
| 6 | Token 预算意识 | 懒加载+条件注入控制上下文规模 | P-06 技能懒加载、P-07 延迟工具 |
| 7 | 层次化指令密度 | 重要规则用视觉差异突出，末尾重复关键规则 | P-01 `<critical_reminders>` |

这七条原则在后续章节中都会被反复引用，作为具体 Prompt 分析的理论框架。第 11 章会将它们展开为更具操作性的设计模式，附带可直接使用的模板。
