# 第 11 章：七大可复用设计模式

## 本章导读

前十章对 DeerFlow 的 25 个核心 Prompt 进行了逐一分析。本章从这些分析中提炼出七个可迁移的设计模式——每个模式在 DeerFlow 代码中有明确的实例，有其背后的工程逻辑，也有超出 DeerFlow 场景的普遍适用性。

这七个模式并非凭空发明，而是从具体代码中反向提炼。每个模式都附有：
- **来源 Prompt**：DeerFlow 中的具体实例
- **模式本质**：抽象后的可迁移表述
- **适用信号**：什么情况下应该使用这个模式
- **反模式**：这个模式的常见误用

---

## 11.1 模式一：静动分离（Static-Dynamic Split）

### DeerFlow 实例

**来源**：P-01、P-02 至 P-07、`DynamicContextMiddleware`

P-01 是一个永远不变的静态模板，通过 `SystemMessage` 在每次请求中原样发送。P-02 至 P-07 等动态段落在每次请求时根据当前状态构建，以 `HumanMessage` 形式注入到对话末尾（`DynamicContextMiddleware`）。

### 模式本质

将 Prompt 内容分为两类：
- **静态内容**：身份定义、行为规范、永久约束——这些内容在不同请求之间不变
- **动态内容**：当前上下文、实时状态、本次请求特有信息——这些内容每次请求都不同

静态内容放入 `SystemMessage` 并固定在对话头部；动态内容通过追加 `HumanMessage` 在每轮注入。

### 工程价值：前缀缓存

这个分离不只是逻辑上的，它直接映射到 LLM 推理层的成本优化：

```
会话 1: [SystemMessage（静态）] [Human][AI][Human][AI][Human]
会话 2: [SystemMessage（静态）] [Human][AI][Human][AI][Human]
         ↑─────────────────────────
         这段完全相同，可以被 KV Cache 复用
```

前缀缓存要求被缓存的前缀在 token 级别完全相同。一旦静态部分发生任何变化（包括空格、换行），缓存立即失效。

DeerFlow 为此甚至设置了专门的 LRU 缓存（`_skill_section_cache`），确保技能元数据段落的文本在相同技能列表下完全一致，保护这一段的缓存命中率。

### 适用信号

- 你的 Agent 运行在多会话、高并发场景（前缀缓存的收益最大化）
- 系统 Prompt 中有频繁变化的部分（动态注入比修改系统 Prompt 更安全）
- 你需要在不同会话中维持同一个 Agent 身份，但上下文每次不同

### 反模式

**把所有内容都放进 SystemMessage**：将当前时间、用户状态、会话上下文等动态信息直接硬编码进系统 Prompt，导致系统 Prompt 每次请求都不同，前缀缓存永远不会命中。

**把所有动态内容都追加成独立 HumanMessage**：每个动态模块都追加一条 `HumanMessage`，导致消息链过长，模型需要在多条消息之间查找上下文。更好的做法是将同一时机的动态内容合并为一条 `HumanMessage`（用 `<system-reminder>` 或 XML 标签区分不同模块）。

---

## 11.2 模式二：XML 标签锚定（XML Tag Anchoring）

### DeerFlow 实例

**来源**：P-01 的 `<role>`、`<soul>`、`<thinking_style>`、`<clarification_system>`、`<skill_system>`、`<subagent_system>`、`<working_directory>`、`<response_style>`、`<citations>`、`<critical_reminders>`

P-01 将整个系统 Prompt 分为十个语义命名的 XML 块，每个块有明确的开关标签。

### 模式本质

使用自定义 XML 标签（不是 Markdown 标题）将 Prompt 的不同语义区域显式划定，每个标签名即该区域的功能描述。

```xml
<clarification_system>
...澄清系统的完整规范...
</clarification_system>
```

### 工程价值：注意力锚点

XML 标签对模型的作用类似于代码中的函数边界：它告诉模型"这里开始一个新的语义区域"，并在解析时提供清晰的边界。

与 Markdown `##` 标题的关键区别：
- Markdown 标题是视觉层级，XML 标签是语义边界
- Markdown 没有关闭标签（`##` 的范围到下一个同级 `##`，通过缩进推断），XML 有明确的 `</tag>` 收口
- XML 标签可以嵌套，Markdown 标题的嵌套语义模糊

P-01 的 10,000+ token 系统 Prompt 之所以仍然可以被模型准确执行，部分原因正是 XML 标签提供的结构化锚点：模型在处理某个请求时，能够"找到"对应的 XML 块（如 `<clarification_system>`），而不是在一大块非结构化文本中搜索相关规则。

### 适用信号

- 系统 Prompt 超过 2000 token，包含多个功能区域
- 不同的规则适用于不同的触发场景（如澄清规则只在需要澄清时相关）
- 需要在代码中动态替换 Prompt 的某个区域（XML 标签提供了可编程的插值位置）

### 反模式

**过度拆分**：每条规则都放在独立的 XML 标签中，导致几十个标签，反而破坏了结构的可读性。DeerFlow 的 10 个标签是经过筛选的语义单元，不是每条规则都值得一个标签。

**标签名语义模糊**：使用 `<section1>`、`<part_a>` 等无意义名称，失去了 XML 标签作为注意力锚点的价值。标签名应该直接描述该区域的功能（`clarification_system`、`working_directory`）。

---

## 11.3 模式三：命名场景约束（Named Scenario Constraints）

### DeerFlow 实例

**来源**：P-01 的 `<clarification_system>`（五类澄清场景），P-08/P-09（TodoList 的"When to Use / When NOT to Use"），P-18/P-20（子 Agent 的使用场景描述），P-22（task 工具的触发条件）

澄清系统定义了五类命名场景：
```
1. missing_info — 信息缺失型
2. ambiguous_requirement — 需求歧义型
3. approach_choice — 方案选择型
4. risk_confirmation — 风险确认型
5. suggestion — 主动建议型
```

每个场景有名称、触发条件和对应行为，而不是一条笼统的"如果不确定就问"规则。

### 模式本质

将"什么情况下做什么"的决策分解为**有命名的离散场景**，每个场景具有：
1. 一个唯一的标识符/名称
2. 明确的触发条件（"什么情况下"）
3. 对应的行为规范（"应该怎么做"）

### 工程价值：消歧义与可测试性

笼统规则（"如果不确定就问"）留有太多解释空间：什么算"不确定"？什么算"足够确定"？模型会按照自己的判断，导致行为不一致。

命名场景将这些判断边界显式化：模型在看到 `missing_info` 的触发条件描述时，只需要判断"当前情况是否符合这个具体描述"，而不是做开放性的"我确定吗"判断。

这也带来了可测试性：你可以针对每个命名场景构造测试用例（"这个输入应该触发 missing_info 澄清"），而针对笼统规则则无法构造有意义的测试。

### 适用信号

- 你有一条行为规则，但在实践中模型的执行结果忽紧忽松
- 规则涉及"何时做 X"的判断，且"何时"有多种不同情形
- 你需要对模型行为进行系统性测试

### 反模式

**场景过于细碎**：如果场景数量超过 7 个，模型维持每个场景的精确定义会变得困难。DeerFlow 选择了 5 个场景——不多不少，覆盖主要类型。

**场景之间边界模糊**：两个场景的触发条件有大量重叠，模型无法判断应该使用哪个。每个场景应该有独特的触发特征，避免与其他场景混淆。

---

## 11.4 模式四：后果驱动约束（Consequence-Driven Constraints）

### DeerFlow 实例

**来源**：P-02（`silently discarded`），P-04（`will be lost on the next turn`），P-08（`do NOT batch completions`），Loop 检测 P-13（`you will lose that work`）

几处关键对比：

| 规则型约束 | 后果型约束 |
|-----------|---------|
| `Maximum {n} task calls per response.` | `Any excess calls are silently discarded by the system — you will lose that work.` |
| `Mark todos as completed immediately.` | `do NOT batch completions` + 隐含的"用户会看到虚假进度" |
| `Don't modify the system prompt.` | `The modifications will be lost on the next turn.` |

### 模式本质

将约束从"规则"改为"因果链"：不只说**必须怎样**，还说**如果不这样将发生什么**。

```
约束型：You MUST NOT do X.
后果型：If you do X, Y will happen (and Y is bad for you/the user).
```

### 工程价值：减少"规则执行者"心态

"规则执行者"心态会导致模型寻找规则的边界（"这个特殊情况算不算 X"），而后果驱动约束直接激活目标导向的推理（"我不想 Y 发生，所以我不做 X"）。

`silently discarded` 比 `not allowed` 更有威慑力的原因：
- `not allowed` → 模型推断：会报错，我会有反馈
- `silently discarded` → 模型推断：不会报错，我不会知道发生了什么，损失是永久的

静默失败比显式错误更难被发现和修复，这种不可观测性本身就是一种更强的约束。

### 适用信号

- 约束违反时不会产生显式错误（系统会静默执行但结果不符合预期）
- 约束的背后有清晰的技术原因（可以用一句话解释）
- 已有的规则型约束被模型经常违反

### 反模式

**描述过于恐怖**：为了增强约束效果，将后果描述为极端灾难（"这将导致系统完全崩溃"），但实际上后果并没有那么严重。不准确的后果描述会破坏模型的信任，并可能导致过度谨慎（什么都不敢做）。

**没有出路的约束**：只说"不能做 X"，但没有提供在无法满足约束时的替代方案。P-13 的 `"If you cannot complete the task, summarize what you accomplished so far"` 就是一个出路设计。

---

## 11.5 模式五：双向边界定义（Bidirectional Boundary Definition）

### DeerFlow 实例

**来源**：P-08/P-09（`When to Use` / `When NOT to Use`），P-18（`Use when` / `Do NOT use for`），P-20（`Use when` / `Do NOT use for`），P-22（`When to use` / `When NOT to use`）

P-08 的 TodoList：
```
When to Use:
- Complex multi-step tasks (3+ distinct steps)
- Non-trivial tasks needing careful planning
- User explicitly requests a todo list
...

When NOT to Use:
- Single, straightforward tasks
- Trivial tasks (< 3 steps)
- Purely conversational or informational requests
...
```

### 模式本质

对任何功能/工具/行为，同时定义**适用边界**（应该使用的情形）和**排斥边界**（不应该使用的情形），而不是只定义其中一侧。

### 工程价值：防止过用与欠用

一个工具的"何时不用"往往比"何时用"更难判断，因为正例更显著（"这明显是 X 的用途"），而反例需要主动识别（"等等，这其实不适合用 X"）。

显式的 `When NOT to Use` 直接提供了反例集：
- P-08：`< 3 steps` 是 TodoList 不适合的具体阈值
- P-20：`simple single commands` 是 Bash 子 Agent 不适合的具体场景
- P-22：`tasks requiring user interaction` 是 `task` 工具的禁区

没有排斥边界的定义会导致"如果有这个工具，就用"的过度使用，增加不必要的开销（TodoList 追踪单步任务）或产生错误（子 Agent 执行需要用户交互的任务）。

### 适用信号

- 工具或功能有明显的过用风险（如 TodoList 被用于所有任务，包括简单任务）
- 工具的适用边界与不适用边界在直觉上容易混淆
- 对功能描述中只说"这个工具用于 X"，但实践中发现模型误用频繁

### 反模式

**排斥边界过于宽泛**：`Do NOT use for anything that could be handled differently`——这样的排斥边界没有操作意义。有效的排斥边界应该有具体的类别（`< 3 steps`、`single commands`）或具体的场景类型（`conversational requests`）。

**正向边界过于宽泛**：`Use for complex tasks`——什么是"复杂"？P-08 给出了 `3+ distinct steps` 的可操作定义，这才是有意义的边界。

---

## 11.6 模式六：渐进式加载（Progressive Loading）

### DeerFlow 实例

**来源**：P-06/P-07（技能元数据段落 + 延迟工具系统），P-03（技能自演化段落），`tool_search` 工具（P-25）

DeerFlow 的技能系统在系统 Prompt 中只注入技能的**元数据**（名称、描述、slash 触发词），而不是 22 个 SKILL.md 文件的完整内容（合计约 47,000 token）：

```
Available skills:
- **code** (/code): Write, review, debug code in any language. [...]
- **deep-research** (/deep-research): Conduct comprehensive multi-step research. [...]
```

只有当用户明确使用 slash 命令（`/code`）时，才通过 `read_file` 工具加载对应技能的完整内容。

类似地，MCP 工具的完整 schema 在系统 Prompt 中只显示工具名称（`<available-deferred-tools>`），schema 需要通过 `tool_search` 工具查询后才注入。

### 模式本质

将 Prompt 中的大型内容按"摘要先行，详情按需"的策略分层加载：
1. 系统 Prompt 中只包含足够触发决策的摘要信息
2. 详细内容通过工具调用（`read_file`、`tool_search`）按需加载
3. 加载后的详细内容以 `HumanMessage` 或 `ToolMessage` 形式注入，不修改系统 Prompt

### 工程价值：大规模内容的 Token 管理

如果把所有内容都放进系统 Prompt：
- 22 个技能文件全文：~47,000 token（每次请求固定开销）
- 100 个 MCP 工具的完整 schema：可能 20,000-50,000 token

渐进式加载将这些开销转化为按需支出：大多数对话只使用 1-3 个技能，只有这 1-3 个技能的 token 成本被实际消耗。

渐进式加载还有一个附带效果：系统 Prompt 保持较短，前缀缓存的命中范围更大（缓存越短，单次失效的影响越小）。

### 适用信号

- 系统 Prompt 中有大量可选内容（技能、工具、配置），但每次对话只用到一小部分
- 内容集合随时间动态增长（新增技能、新增工具），不希望每次内容变化都导致系统 Prompt 版本更新
- 对话 context 窗口有限，需要为对话内容保留足够空间

### 反模式

**加载时机过于激进**：所有技能在会话开始时一次性全部加载，而不是按需加载，失去了渐进式加载的节省效果。

**加载后不缓存**：同一个技能在一次会话中被多次加载（每次 `/code` 调用都重新读文件），不必要地重复了 `read_file` 工具调用。DeerFlow 在会话级别缓存已加载的技能内容，避免了这个问题。

---

## 11.7 模式七：层级指令密度（Hierarchical Instruction Density）

### DeerFlow 实例

**来源**：P-01 整体结构，P-10（`MEMORY_UPDATE_PROMPT`），P-02（`<subagent_system>`），P-08（TodoList 系统 Prompt）

P-01 的 `<role>` 块只有 3 行（~14 token），而 `<clarification_system>` 有超过 60 行（~500 token）。这不是偶然的——`<role>` 定义了一个稳定的身份（不需要详细说明），`<clarification_system>` 定义了一个复杂的决策树（需要详细规范）。

P-10 中，六个记忆区域的长度指导：
```
workContext (2-3 sentences)
personalContext (1-2 sentences)
topOfMind (3-5 sentences, detailed paragraph)
recentMonths (4-6 sentences or 1-2 paragraphs)
```

每个区域的指定长度不同，反映了每个区域对记忆质量的重要程度。

### 模式本质

在 Prompt 中，根据每个规则/模块对模型行为影响的重要程度，分配不同密度的说明：
- **高影响规则**：详细说明（触发条件、执行步骤、示例、反例）
- **低影响规则**：简短声明（一句话说清楚）
- **背景信息**：最简（标签名即说明，无需额外解释）

### 工程价值：注意力分配优化

模型处理 Prompt 时，注意力资源有限。如果每条规则都用相同的篇幅说明，模型无法区分哪些规则更重要——所有规则看起来都一样。

层级密度通过篇幅本身传递了权重信息：花了 500 token 来解释的规则，比只用 10 token 说清楚的规则，在模型的注意力中自然占有更大权重。

这也解释了 DeerFlow `<role>` 极简的原因：模型身份（"you are an intelligent assistant"）一旦建立就不需要反复强调，真正需要详细说明的是容易被误判的决策（何时澄清？何时用子 Agent？）。

P-02 中最长的部分是并发限制的说明（约 200 token），而不是编排哲学的总结（约 30 token）——并发限制是最容易被违反的约束，因此分配了最高的指令密度。

### 适用信号

- 你的 Prompt 中所有规则都用相同的格式和篇幅，但某些规则被模型频繁违反
- Prompt 过长，导致某些重要规则"淹没"在非重要说明中
- 添加了更多规则说明，但模型行为反而变得更不可预测（注意力分散）

### 反模式

**均匀密度**：每条规则都写成一个 5 行的说明块，无论重要程度。这会让模型认为所有规则等权，高优先级规则的违反率不会低于低优先级规则。

**核心约束用一行说清**："`Maximum {n} concurrent subagents.`"——没有后果说明、没有计数工作流、没有示例。把重要约束压缩到极短，反而会导致被忽视。

---

## 11.8 七大模式的相互关系

这七个模式不是独立的——在 DeerFlow 的设计中，它们形成了一个互相支撑的体系：

```
静动分离 ────────────────────────────────────────────────────────────
（基础架构层）                                                       ↓
                                                          前缀缓存复用

XML 标签锚定 + 层级指令密度 ────────────────────────────────────────
（结构组织层）                                                       ↓
                                                          模型注意力管理

命名场景约束 + 后果驱动约束 + 双向边界定义 ──────────────────────────
（规则表达层）                                                       ↓
                                                          模型行为可预测性

渐进式加载 ─────────────────────────────────────────────────────────
（资源管理层）                                                       ↓
                                                          Token 成本控制
```

具体地：
- 静动分离（模式一）是前提，没有它，其他模式（特别是渐进式加载）无法实现
- XML 标签锚定（模式二）使层级指令密度（模式七）变得可见：不同 XML 块的篇幅差异直接体现了密度差异
- 命名场景约束（模式三）为双向边界定义（模式五）提供了一致的表达框架
- 后果驱动约束（模式四）使双向边界定义（模式五）中的"不应该使用"边界更有说服力

---

## 11.9 一张速查表

| 模式 | 核心问题 | DeerFlow 最典型实例 | 一句话判断 |
|------|---------|-------------------|----------|
| 静动分离 | 哪些内容每次相同，哪些每次变化？ | P-01 vs DynamicContextMiddleware | 变化的内容不进 SystemMessage |
| XML 标签锚定 | 长 Prompt 如何让模型"找到"相关规则？ | P-01 的 10 个语义块 | 标签名就是功能描述 |
| 命名场景约束 | 如何消除"什么情况下应该执行规则 X"的歧义？ | `<clarification_system>` 5 类场景 | 为每种情形取一个名字 |
| 后果驱动约束 | 规则经常被忽视，怎么加强？ | `silently discarded` | 说清楚"不做会损失什么" |
| 双向边界定义 | 工具被过度使用，怎么纠正？ | P-08/P-09 的 When/When NOT | 同时写"何时用"和"何时不用" |
| 渐进式加载 | 内容太多放不下，怎么处理？ | P-06 + P-07 + `read_file` | 摘要先行，详情按需加载 |
| 层级指令密度 | 规则有轻重，Prompt 应该如何体现？ | `<role>` 3 行 vs `<clarification_system>` 60 行 | 重要规则多写，次要规则少写 |

---

## 本章小结

本章从 DeerFlow 25 个 Prompt 的分析中提炼了七个可迁移的设计模式。这些模式不是凭空的设计理念，而是工程约束（前缀缓存、token 预算、注意力有限）驱动下的实践解法。

下一章将讨论如何将这些模式应用到你自己的 Agent 系统设计中。
