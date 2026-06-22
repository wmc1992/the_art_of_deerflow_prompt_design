# 第 9 章：记忆系统——P-10 的知识提炼机制

## 本章导读

本章分析 DeerFlow 记忆子系统的两个 Prompt：P-10（`MEMORY_UPDATE_PROMPT`，负责分析对话并更新完整用户画像）和 P-11（`FACT_EXTRACTION_PROMPT`，负责从单条消息提取离散事实）。这两个 Prompt 运行在主 Agent 之外的独立 LLM 调用中，是整个系统中设计最精细的分析型 Prompt。

---

## 9.1 记忆系统的架构位置

在分析 Prompt 文本之前，理解 P-10/P-11 的运行位置很重要。

```
对话结束
    ↓
MemoryMiddleware（队列）
    ↓
debounce（默认 30 秒）
    ↓
后台线程（独立于主 Agent）
    ↓
P-10 或 P-11 的 LLM 调用
    ↓
更新 memory.json（原子写入）
    ↓
下次对话时注入 <memory> 标签
```

这个异步管道有几个重要特性：
- **不阻塞主流程**：记忆更新在对话结束后才触发，不影响响应延迟
- **独立的 LLM 调用**：可以使用比主 Agent 更小、更快的模型（通过 `config.yaml` 的 `memory.model_name` 指定）
- **debounce 防抖**：30 秒内的多次对话会被合并为一次更新，避免频繁写入

---

## 9.2 P-10：`MEMORY_UPDATE_PROMPT` 完整原文

```
You are a memory management system. Your task is to analyze a conversation and update the user's memory profile.

Current Memory State:
<current_memory>
{current_memory}
</current_memory>

New Conversation to Process:
<conversation>
{conversation}
</conversation>

Instructions:
1. Analyze the conversation for important information about the user
2. Extract relevant facts, preferences, and context with specific details (numbers, names, technologies)
3. Update the memory sections as needed following the detailed length guidelines below

Before extracting facts, perform a structured reflection on the conversation:
1. Error/Retry Detection: Did the agent encounter errors, require retries, or produce incorrect results?
   If yes, record the root cause and correct approach as a high-confidence fact with category "correction".
2. User Correction Detection: Did the user correct the agent's direction, understanding, or output?
   If yes, record the correct interpretation or approach as a high-confidence fact with category "correction".
   Include what went wrong in "sourceError" only when category is "correction" and the mistake is explicit in the conversation.
3. Project Constraint Discovery: Were any project-specific constraints discovered during the conversation?
   If yes, record them as facts with the most appropriate category and confidence.

{correction_hint}

Memory Section Guidelines:

**User Context** (Current state - concise summaries):
- workContext: Professional role, company, key projects, main technologies (2-3 sentences)
  Example: Core contributor, project names with metrics (16k+ stars), technical stack
- personalContext: Languages, communication preferences, key interests (1-2 sentences)
  Example: Bilingual capabilities, specific interest areas, expertise domains
- topOfMind: Multiple ongoing focus areas and priorities (3-5 sentences, detailed paragraph)
  Example: Primary project work, parallel technical investigations, ongoing learning/tracking
  Include: Active implementation work, troubleshooting issues, market/research interests
  Note: This captures SEVERAL concurrent focus areas, not just one task

**History** (Temporal context - rich paragraphs):
- recentMonths: Detailed summary of recent activities (4-6 sentences or 1-2 paragraphs)
  Timeline: Last 1-3 months of interactions
  Include: Technologies explored, projects worked on, problems solved, interests demonstrated
- earlierContext: Important historical patterns (3-5 sentences or 1 paragraph)
  Timeline: 3-12 months ago
  Include: Past projects, learning journeys, established patterns
- longTermBackground: Persistent background and foundational context (2-4 sentences)
  Timeline: Overall/foundational information
  Include: Core expertise, longstanding interests, fundamental working style

**Facts Extraction**:
- Extract specific, quantifiable details (e.g., "16k+ GitHub stars", "200+ datasets")
- Include proper nouns (company names, project names, technology names)
- Preserve technical terminology and version numbers
- Categories:
  * preference: Tools, styles, approaches user prefers/dislikes
  * knowledge: Specific expertise, technologies mastered, domain knowledge
  * context: Background facts (job title, projects, locations, languages)
  * behavior: Working patterns, communication habits, problem-solving approaches
  * goal: Stated objectives, learning targets, project ambitions
  * correction: Explicit agent mistakes or user corrections, including the correct approach
- Confidence levels:
  * 0.9-1.0: Explicitly stated facts ("I work on X", "My role is Y")
  * 0.7-0.8: Strongly implied from actions/discussions
  * 0.5-0.6: Inferred patterns (use sparingly, only for clear patterns)

**What Goes Where**:
- workContext: Current job, active projects, primary tech stack
- personalContext: Languages, personality, interests outside direct work tasks
- topOfMind: Multiple ongoing priorities and focus areas user cares about recently (gets updated most frequently)
  Should capture 3-5 concurrent themes: main work, side explorations, learning/tracking interests
- recentMonths: Detailed account of recent technical explorations and work
- earlierContext: Patterns from slightly older interactions still relevant
- longTermBackground: Unchanging foundational facts about the user

**Multilingual Content**:
- Preserve original language for proper nouns and company names
- Keep technical terms in their original form (DeepSeek, LangGraph, etc.)
- Note language capabilities in personalContext

Output Format (JSON):
{
  "user": {
    "workContext": { "summary": "...", "shouldUpdate": true/false },
    "personalContext": { "summary": "...", "shouldUpdate": true/false },
    "topOfMind": { "summary": "...", "shouldUpdate": true/false }
  },
  "history": {
    "recentMonths": { "summary": "...", "shouldUpdate": true/false },
    "earlierContext": { "summary": "...", "shouldUpdate": true/false },
    "longTermBackground": { "summary": "...", "shouldUpdate": true/false }
  },
  "newFacts": [
    { "content": "...", "category": "preference|knowledge|context|behavior|goal|correction", "confidence": 0.0-1.0 }
  ],
  "factsToRemove": ["fact_id_1", "fact_id_2"]
}

Important Rules:
- Only set shouldUpdate=true if there's meaningful new information
- Follow length guidelines: workContext/personalContext are concise (1-3 sentences), topOfMind and history sections are detailed (paragraphs)
- Include specific metrics, version numbers, and proper nouns in facts
- Only add facts that are clearly stated (0.9+) or strongly implied (0.7+)
- Use category "correction" for explicit agent mistakes or user corrections; assign confidence >= 0.95 when the correction is explicit
- Include "sourceError" only for explicit correction facts when the prior mistake or wrong approach is clearly stated; omit it otherwise
- Remove facts that are contradicted by new information
- When updating topOfMind, integrate new focus areas while removing completed/abandoned ones
  Keep 3-5 concurrent focus themes that are still active and relevant
- For history sections, integrate new information chronologically into appropriate time period
- Preserve technical accuracy - keep exact names of technologies, companies, projects
- Focus on information useful for future interactions and personalization
- IMPORTANT: Do NOT record file upload events in memory. Uploaded files are
  session-specific and ephemeral — they will not be accessible in future sessions.
  Recording upload events causes confusion in subsequent conversations.

Return ONLY valid JSON, no explanation or markdown.
```

> **中文附注**：
>
> 你是一个记忆管理系统。你的任务是分析对话并更新用户的记忆档案。
>
> **占位符**：`{current_memory}` 为当前记忆状态，`{conversation}` 为待处理的新对话，`{correction_hint}` 为可选的纠正提示。
>
> **指令**：（1）分析对话，提取关于用户的重要信息；（2）提取含具体细节（数字、名称、技术名称）的相关事实、偏好和背景；（3）按以下长度指导更新记忆区域。
>
> **在提取事实之前，先进行结构化反思**：
> 1. **错误/重试检测**：agent 是否遇到错误、需要重试或产生不正确的结果？若是，将根本原因和正确方法记录为高置信度事实，类别为 `"correction"`。
> 2. **用户纠正检测**：用户是否纠正了 agent 的方向、理解或输出？若是，将正确的解释或方法记录为高置信度事实，类别为 `"correction"`。只有当类别为 `"correction"` 且对话中明确提到错误时，才在 `"sourceError"` 中包含出错信息。
> 3. **项目约束发现**：对话中是否发现了项目特定的约束？若是，用最合适的类别和置信度记录这些事实。
>
> **用户上下文**（当前状态——简洁摘要）：
> - `workContext`：职业角色、公司、关键项目、主要技术（2-3 句）
> - `personalContext`：语言、沟通偏好、主要兴趣（1-2 句）
> - `topOfMind`：多个持续进行的关注领域和优先事项（3-5 句，详细段落）；注：捕捉**多个**并发关注领域，而非只记录单个任务
>
> **历史上下文**（时间轴视图——详细段落）：
> - `recentMonths`：近 1-3 个月详细活动摘要（4-6 句或 1-2 段）
> - `earlierContext`：3-12 个月前的重要历史模式（3-5 句或 1 段）
> - `longTermBackground`：持久的基础背景（2-4 句）
>
> **事实提取**：提取具体可量化细节、专有名词（公司/项目/技术名）、版本号。
> - 类别：`preference`（偏好）/ `knowledge`（知识）/ `context`（背景）/ `behavior`（行为）/ `goal`（目标）/ `correction`（纠正：明确的 agent 错误或用户纠正，含正确方法）
> - 置信度：0.9-1.0 明确陈述 / 0.7-0.8 强烈暗示 / 0.5-0.6 推断模式（谨慎使用）
>
> **重要规则**：只有存在有意义的新信息时才设置 `shouldUpdate=true`；`correction` 类置信度 ≥ 0.95；只有纠正事实且先前错误在对话中明确陈述时才填写 `sourceError`；删除被新信息矛盾的旧事实；保留技术、公司、项目的确切名称；多语言专有名词保持原始形式。
>
> **重要**：**不得**在记忆中记录文件上传事件。上传的文件是会话特有且临时的——在未来的会话中将无法访问。记录上传事件会在后续对话中造成混乱。
>
> 只返回有效的 JSON，不加解释和 Markdown。

---

## 9.3 P-10 结构解析：六个记忆区域

P-10 定义了两层记忆结构，共六个区域：

### 第一层：用户上下文（User Context）——横截面视图

```
- workContext: Professional role, company, key projects, main technologies (2-3 sentences)
- personalContext: Languages, communication preferences, key interests (1-2 sentences)
- topOfMind: Multiple ongoing focus areas and priorities (3-5 sentences, detailed paragraph)
```

> **中文附注**：
>
> - `workContext`：职业角色、公司、关键项目、主要技术（2-3 句）
> - `personalContext`：语言、沟通偏好、主要兴趣（1-2 句）
> - `topOfMind`：多个持续进行的关注领域和优先事项（3-5 句，详细段落）

这三个区域是**当前状态的快照**：用户现在是谁（`workContext`）、用户的个人特征（`personalContext`）、用户正在关注什么（`topOfMind`）。

`topOfMind` 的设计特别值得注意：

```
Note: This captures SEVERAL concurrent focus areas, not just one task
```

> **中文附注**：注：此项捕捉**多个**并发关注领域，而非只记录单个任务。

这个设计反映了真实用户行为的多线程性：用户同时在推进多个项目、追踪多个技术方向、学习多件事情。如果 `topOfMind` 只记录最近一次对话的话题，记忆会迅速退化为对最后一次交互的重复。通过强调"SEVERAL concurrent focus areas"，P-10 引导模型保持记忆的多维度性。

### 第二层：历史上下文（History）——时间轴视图

```
- recentMonths: Last 1-3 months (4-6 sentences or 1-2 paragraphs)
- earlierContext: 3-12 months ago (3-5 sentences or 1 paragraph)
- longTermBackground: Overall/foundational information (2-4 sentences)
```

> **中文附注**：
>
> - `recentMonths`：近 1-3 个月（4-6 句或 1-2 段）
> - `earlierContext`：3-12 个月前（3-5 句或 1 段）
> - `longTermBackground`：总体/基础信息（2-4 句）

三个时间段的划分创建了一个**记忆衰减模型**：
- 最近 1-3 个月：详细，包含具体的技术探索和工作内容
- 3-12 个月前：概括，只保留"重要的历史模式"
- 长期背景：极简，只有"核心专长、长期兴趣、根本工作风格"

这个衰减设计避免了记忆的无限膨胀：随着时间推移，信息自然从"详细"区域迁移到"概括"区域，最终沉淀为"长期背景"或被遗忘（通过 `factsToRemove` 删除）。

---

## 9.4 结构化反思：先思考，后提取

P-10 中一个被容易忽略的设计是"结构化反思"步骤：

```
Before extracting facts, perform a structured reflection on the conversation:
1. Error/Retry Detection: Did the agent encounter errors, require retries, or produce incorrect results?
2. User Correction Detection: Did the user correct the agent's direction, understanding, or output?
3. Project Constraint Discovery: Were any project-specific constraints discovered?
```

> **中文附注**：
>
> 在提取事实之前，先对对话进行结构化反思：
> 1. 错误/重试检测：agent 是否遇到错误、需要重试或产生不正确的结果？
> 2. 用户纠正检测：用户是否纠正了 agent 的方向、理解或输出？
> 3. 项目约束发现：对话中是否发现了项目特定的约束？

这个步骤强制模型**在提取事实之前先做一轮专项扫描**，而不是直接跳入"找出所有重要信息"。

为什么需要先扫描错误和纠正？因为这类信息对未来交互最有价值，但也最容易被忽略：

- 对话中发生的错误往往以负面事件的形式出现（"出错了，换个方法"），而不是以正面陈述的形式
- 用户纠正通常是隐式的（"不，我的意思是..."），模型需要主动识别这些信号
- 项目约束往往在"顺带提及"的语境中出现（"对了，我们不用 X 框架"）

通过命名这三类特殊情况，P-10 确保了这些高价值信息不会被通用的"提取重要信息"指令遗漏。

---

## 9.5 `correction` 类别：专为纠错设计的记忆类型

P-10 中六个事实分类里，`correction` 是设计最特殊的一个：

```
- correction: Explicit agent mistakes or user corrections, including the correct approach
```

> **中文附注**：`correction`（纠正）：明确的 agent 错误或用户纠正，包括正确方法。

**高置信度规则**：
```
- Use category "correction" for explicit agent mistakes or user corrections; assign confidence >= 0.95 when the correction is explicit
```

> **中文附注**：对明确的 agent 错误或用户纠正使用 `"correction"` 类别；纠正明确时，置信度 ≥ 0.95。

普通事实的置信度可以低至 0.5（推断的模式），但 `correction` 类别的置信度被强制设为 ≥ 0.95。这反映了一个重要的认知：**明确的纠正比任何推断都更确定**，应该以最高置信度记录，以确保它在未来注入时不会被低置信度事实挤出。

**`sourceError` 字段的条件性**：
```
- Include "sourceError" only for explicit correction facts when the prior mistake or wrong approach is clearly stated; omit it otherwise
```

> **中文附注**：只有当类别为纠正事实且先前的错误或错误方法在对话中明确陈述时，才包含 `"sourceError"`；否则省略。

`sourceError` 记录的是"之前的错误是什么"，这样在未来类似场景发生时，模型可以知道不只是"要做 Y"，还是"因为 X 是错的，要做 Y"。

---

## 9.6 事实置信度分级

```
- Confidence levels:
  * 0.9-1.0: Explicitly stated facts ("I work on X", "My role is Y")
  * 0.7-0.8: Strongly implied from actions/discussions
  * 0.5-0.6: Inferred patterns (use sparingly, only for clear patterns)
```

> **中文附注**：
>
> - 置信度：
>   * 0.9-1.0：明确陈述的事实（"我在 X 工作"，"我的职位是 Y"）
>   * 0.7-0.8：从行动/讨论中强烈暗示的
>   * 0.5-0.6：推断的模式（谨慎使用，仅用于明显的模式）

三个置信度区间对应三种信息来源：

| 区间 | 信息来源 | 使用限制 |
|------|---------|---------|
| 0.9-1.0 | 用户直接陈述 | 无限制，收集所有 |
| 0.7-0.8 | 行为强烈暗示 | 正常使用 |
| 0.5-0.6 | 模式推断 | 谨慎使用（`use sparingly`）|

在 `format_memory_for_injection()` 中，事实按置信度降序排列后注入：

```python
ranked_facts = sorted(
    facts_data,
    key=lambda fact: _coerce_confidence(fact.get("confidence"), default=0.0),
    reverse=True,
)
```

当记忆接近 2000 token 的预算上限时，低置信度的事实会被自然截断。这意味着 `correction` 类别（≥ 0.95）的事实总是优先出现在注入内容中，设计上保证了纠错信息不会因为 token 预算而被丢弃。

---

## 9.7 JSON 输出格式的设计

P-10 要求输出纯 JSON，不带任何解释或 Markdown 标记：

```
Return ONLY valid JSON, no explanation or markdown.
```

> **中文附注**：只返回有效的 JSON，不加解释和 Markdown。

JSON 格式的 `shouldUpdate` 字段设计很有意思：

```json
"workContext": { "summary": "...", "shouldUpdate": true/false }
```

`shouldUpdate=false` 表示"这个区域没有新的有意义的信息，不需要更新"。这允许 LLM 在没有新信息的情况下直接跳过某个区域的更新，而不必重新生成相同的内容。

同样，`factsToRemove` 字段允许显式删除已被新信息矛盾的旧事实：

```json
"factsToRemove": ["fact_id_1", "fact_id_2"]
```

这是一个**主动遗忘机制**：记忆不只是不断增长，还可以通过明确标记来修正过时的信息。

---

## 9.8 "不记录文件上传"规则

P-10 末尾有一条值得特别关注的规则：

```
IMPORTANT: Do NOT record file upload events in memory. Uploaded files are
session-specific and ephemeral — they will not be accessible in future sessions.
Recording upload events causes confusion in subsequent conversations.
```

> **中文附注**：**重要**：**不得**在记忆中记录文件上传事件。上传的文件是会话特有且临时的——在未来的会话中将无法访问。记录上传事件会在后续对话中造成混乱。

这条规则解决了一个实际问题：如果记忆记录了"用户上传了 report.pdf"，下次会话时系统会以为这个文件仍然可访问，但实际上每次会话的上传文件是独立的临时文件。这种跨会话的路径引用会导致模型在未来尝试访问不存在的文件，造成混乱。

这是一条从实际故障中提炼的规则，不是抽象的设计原则。

代码层面也有对应的预处理：

```python
if role == "human":
    content = re.sub(r"<uploaded_files>[\s\S]*?</uploaded_files>\n*", "", str(content)).strip()
    if not content:
        continue
```

在将对话格式化给 P-10 之前，代码会先剥离消息中的 `<uploaded_files>` 标签，从源头防止文件路径信息进入 LLM 的分析输入。Prompt 规则和代码预处理共同构成了双重防护。

---

## 9.9 P-11：`FACT_EXTRACTION_PROMPT` 完整原文

```
Extract factual information about the user from this message.

Message:
{message}

Extract facts in this JSON format:
{
  "facts": [
    { "content": "...", "category": "preference|knowledge|context|behavior|goal|correction", "confidence": 0.0-1.0 }
  ]
}

Categories:
- preference: User preferences (likes/dislikes, styles, tools)
- knowledge: User's expertise or knowledge areas
- context: Background context (location, job, projects)
- behavior: Behavioral patterns
- goal: User's goals or objectives
- correction: Explicit corrections or mistakes to avoid repeating

Rules:
- Only extract clear, specific facts
- Confidence should reflect certainty (explicit statement = 0.9+, implied = 0.6-0.8)
- Skip vague or temporary information

Return ONLY valid JSON.
```

> **中文附注**：
>
> 从此消息中提取关于用户的事实性信息。
>
> 消息：`{message}`
>
> 以此 JSON 格式提取事实：`{ "facts": [{ "content": "...", "category": "...", "confidence": 0.0-1.0 }] }`
>
> 类别：
> - `preference`：用户偏好（喜好/厌恶、风格、工具）
> - `knowledge`：用户的专业知识或知识领域
> - `context`：背景信息（地点、工作、项目）
> - `behavior`：行为模式
> - `goal`：用户的目标或目的
> - `correction`：应避免重复的明确纠正或错误
>
> 规则：只提取清晰、具体的事实；置信度应反映确定性（明确陈述 = 0.9+，暗示 = 0.6-0.8）；跳过模糊或临时信息。
>
> 只返回有效的 JSON。

### P-11 与 P-10 的分工

P-11 是 P-10 的轻量版本，两者处理不同的触发场景：

| 维度 | P-10（MEMORY_UPDATE）| P-11（FACT_EXTRACTION）|
|------|---------------------|----------------------|
| **输入** | 完整对话历史 | 单条消息 |
| **输出** | 完整记忆更新（六区域 + 事实）| 只有新事实列表 |
| **Token 成本** | 高（分析整段对话）| 低（分析单条消息）|
| **触发时机** | 对话结束后 debounce 执行 | 需要从单条消息快速提取时 |
| **结构化反思** | 有（错误检测、纠正检测、约束发现）| 无（直接提取）|

P-11 的简洁性不是疏忽，而是职责边界的精确划分：当只需要从一条消息中提取事实时，不需要六区域的完整分析框架。

---

## 9.10 记忆注入的格式：从 memory.json 到 `<memory>` 标签

P-10/P-11 生成的数据最终以什么形式注入到主 Agent 的对话中？

`format_memory_for_injection()` 将 memory.json 的结构化数据转换为一段注入文本，格式如下（最多 2000 token）：

```
User Context:
- Work: [workContext.summary]
- Personal: [personalContext.summary]
- Current Focus: [topOfMind.summary]

History:
- Recent: [recentMonths.summary]
- Earlier: [earlierContext.summary]
- Background: [longTermBackground.summary]

Facts:
- [preference | 0.95] User prefers TypeScript over JavaScript for new projects
- [correction | 0.98] Agent should not use class components; user migrated to hooks (avoid: using React.Component)
- [knowledge | 0.90] User has 5+ years experience with LangChain
...
```

> **中文附注**（记忆注入示例格式）：
>
> 用户上下文：
> - 工作：[workContext.summary]
> - 个人：[personalContext.summary]
> - 当前关注：[topOfMind.summary]
>
> 历史：
> - 近期：[recentMonths.summary]
> - 早期：[earlierContext.summary]
> - 背景：[longTermBackground.summary]
>
> 事实（格式：`[类别 | 置信度] 内容`，按置信度降序排列，高置信度事实优先出现）

这段文本被包裹在 `<memory>` 标签中，通过 `DynamicContextMiddleware` 注入到每次请求的 `<system-reminder>` 块中。

事实格式 `[category | confidence] content` 提供了足够的元信息让主 Agent 在使用记忆时做出判断：置信度低的推断事实（如 `0.62`）会比显式陈述事实（`0.95`）获得更少的信任权重。

---

## 本章小结

- P-10 是 DeerFlow 中设计最精细的分析型 Prompt，通过"结构化反思前置"确保高价值的错误和纠正信息不被遗漏
- 六区域记忆结构（3 个横截面视图 + 3 个时间轴视图）和三层置信度分级共同构成了一个有衰减机制的长期记忆模型
- `correction` 类别的 ≥ 0.95 置信度规则与置信度排序注入机制配合，保证了纠错信息在 token 预算有限时优先保留
- "不记录文件上传"规则由 Prompt 约束和代码预处理双重保障，是从实际故障中提炼的防御性设计
- P-11 是 P-10 的轻量版本，职责明确划分为"从单条消息提取事实"，避免了对完整分析框架的不必要使用
