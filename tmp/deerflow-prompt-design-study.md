# DeerFlow Prompt 设计艺术学习方案

> 基于 v2.0.0-rc1（2026-06-19 tag）源码分析
> 学习目标：不只是"读一遍 prompt"，而是理解**为什么有效、怎么写才能让模型更准确执行任务**

---

## 学习地图总览

本方案围绕 5 个核心维度展开，每个维度对应 DeerFlow 里的具体文件和设计决策：

| 维度 | 核心问题 | 主要文件 |
|---|---|---|
| 行为工程 | 怎么让模型可靠地选 A 而不是 B | `lead_agent/prompt.py` |
| 信息架构 | 什么内容放哪里，为什么 | `prompt.py` + `dynamic_context_middleware.py` |
| 注意力管理 | 如何对抗 lost-in-the-middle | `SYSTEM_PROMPT_TEMPLATE` 结构 |
| 模块化与懒加载 | prompt 如何随系统扩展而不膨胀 | `skills/public/*/SKILL.md` |
| Token 预算 | 每部分的成本，以及预算决策背后的工程权衡 | 多处 |

---

## 维度一：行为工程

**核心问题**：同一条"规则"，不同的写法会让模型的服从率差异巨大。为什么？

### 1.1 规则 < 决策树 < 场景示范

以"需要澄清时要先澄清"为例，三种写法效果递增：

```
# 弱：规则陈述（模型需要自己判断"什么是 unclear"）
Always clarify when something is unclear.

# 中：带触发条件的决策树（降低判断成本）
If requirements are missing/ambiguous/risky, call ask_clarification FIRST.

# 强：DeerFlow 的写法（prompt.py 第 388-447 行）
5 种具名场景(missing_info / ambiguous_requirement / approach_choice /
risk_confirmation / suggestion) + 每种场景的触发例子 + 反例 + 伪代码
```

**为什么"具名场景"比规则更有效**：模型判断"当前情况是否符合规则"需要消耗推理能力，且容易漏判。具名分类相当于给了模型一个"模式匹配索引"，降低识别成本、减少漏触发。

**设计要点**：场景名本身要语义自洽（`missing_info` 比 `type1` 有效得多），且分类之间最好不互斥——即使模型对边界判断有偏差，"总有一个分类能匹配"会触发正确行为，这是**鲁棒性设计**。

### 1.2 后果描述 vs 权威要求

```
# 弱：权威要求（模型盲目服从，推理链断裂）
NEVER launch more than 3 task calls per response.

# 强：DeerFlow 的写法（_build_subagent_section，~第 247 行）
⛔ HARD CONCURRENCY LIMIT: MAXIMUM 3 task calls per response. THIS IS NOT OPTIONAL.
- Any excess calls are **silently discarded** by the system — you will lose that work.
```

**原理**：模型在训练中学到了因果关系。后果描述激活的是它对因果链的理解（"发出第 4 个调用 → 工作丢失"），比单纯的禁令更能建立持久的行为约束。

注意 DeerFlow 用的是三重强调：`⛔` 符号 + 全大写 + "THIS IS NOT OPTIONAL"。但这不是为了"凶"，而是为了让这条规则在 context 里的视觉权重足够高，对抗 lost-in-the-middle。

### 1.3 代码 > 自然语言描述程序性步骤

```python
# subagent_system 里的示例（prompt.py ~第 326 行）
# User asks: "Why is Tencent's stock price declining?"
# Thinking: 3 sub-tasks → fits in 1 batch

task(description="Tencent financial data", prompt="...", subagent_type="general-purpose")
task(description="Tencent news & regulation", prompt="...", subagent_type="general-purpose")
task(description="Industry & market trends", prompt="...", subagent_type="general-purpose")
```

**原理**：对于工具调用序列这类"程序性"行为，伪代码比自然语言更精确，也更符合模型的训练分布（模型训练了大量代码，代码的每一行都有明确的语义）。

**实践结论**：描述**行为规范**用自然语言；描述**执行序列**用代码或伪代码。

### 1.4 正反对比的力量

DeerFlow 在 `<clarification_system>` 和 `<subagent_system>` 里大量使用 ✅/❌ 对比，并且反例往往比正例更重要：

```
❌ DO NOT start working and then ask for clarification mid-execution — clarify FIRST
❌ DO NOT skip clarification for "efficiency" — accuracy matters more than speed
❌ DO NOT make assumptions when information is missing — ALWAYS ask
```

**原理**：模型在"应该做什么"上往往有多个合理选项，反例精确地排除了那些看起来合理但错误的选项，缩小了模型的决策空间。

---

## 维度二：信息架构——静态与动态的分离

这是 DeerFlow 里**最有工程价值**的 prompt 设计决策，也是最不容易被看出来的。

### 2.1 核心设计决策

```
System Prompt（完全静态）
  ├── 规则/人格/工具说明
  ├── Skills 元数据（仅 name + description + path）
  └── 所有用户、所有 thread 共享同一份
      → prefix cache 命中 → 节省大量 token 费用

HumanMessage 里的 <system-reminder>（每轮动态注入）
  ├── 当前日期
  ├── 用户记忆（长期 facts）
  └── 上传文件列表
      → 因用户/时间不同而变化，不污染可缓存的前缀
```

**来源**：`prompt.py` 第 799-803 行注释（最关键的注释之一）：
```python
# Memory and current date are injected per-turn via DynamicContextMiddleware
# as a <system-reminder> in the first HumanMessage, keeping this prompt
# identical across users and sessions for maximum prefix-cache reuse.
```

### 2.2 设计原则提炼

**不是"重要的就放 system prompt"，而是"内容类型决定放在哪"**：

| 内容类型 | 适合位置 | 原因 |
|---|---|---|
| 规则、人格、工具说明 | System Prompt | 静态，所有用户共享，可缓存 |
| 当前日期、记忆、上传文件 | HumanMessage `<system-reminder>` | 每轮/每用户不同，不能缓存 |
| 技能详细 SOP | 按需 `read_file` 加载 | 仅当用到时才进入活跃 context |
| 工具 schema（MCP） | 延迟绑定 | `DeferredToolFilterMiddleware` 控制何时暴露给模型 |

### 2.3 实践启示

当你设计一个系统的 prompt 时，第一步应该问：**"这段内容是所有用户/所有时间都一样的，还是会因人因时而变？"** 变化的部分从 system prompt 里移出去。

---

## 维度三：注意力管理

### 3.1 XML 标签作为导航锚点

DeerFlow 的 XML 标签系统（`<role>`, `<thinking_style>`, `<clarification_system>`, `<skill_system>`, `<subagent_system>`, `<working_directory>`, `<response_style>`, `<citations>`, `<critical_reminders>`）不只是视觉分区，本质上是在帮模型建立一个**可导航的"内存布局"**。

当模型需要回忆"澄清规则是什么"时，`<clarification_system>` 标签是一个强锚点，模型可以"定位"到这个区域而不需要扫描全文。

### 3.2 每个 XML 段存在的理由

| 标签 | 存在理由 | 如果缺失会怎样 |
|---|---|---|
| `<role>` | 锚定身份，防止人格漂移 | 多轮对话后模型可能忘记自己是谁 |
| `<soul>` | 个性化注入点（用户自定义） | 每个定制 agent 都需要维护一份完整 system prompt |
| `<self_update>` | 教 agent 如何安全地更新自己 | 模型会用错误的工具（bash/write_file）修改配置 |
| `<thinking_style>` | 规范推理纪律，先想后做 | 模型容易直接行动，不检查是否需要澄清 |
| `<clarification_system>` | 中断错误执行的"门卫" | 模型会对不清楚的需求猜测并执行 |
| `<skill_system>` | 技能发现和加载协议 | 模型不知道如何/何时使用 skills |
| `<subagent_system>` | 并发编排规则 + 硬限制说明 | 模型会发出超限的 task 调用，工作丢失 |
| `<working_directory>` | 文件系统契约 | 模型使用错误路径，文件操作失败 |
| `<response_style>` | 输出格式基准 | 模型倾向于过度格式化（大量 bullet points） |
| `<citations>` | 研究类任务的来源规范 | 模型生成内容不引用来源，无法溯源 |
| `<critical_reminders>` | 高优先级规则的末尾强化 | 长 prompt 中段的规则容易被"遗忘" |

### 3.3 `<critical_reminders>` 放在最后不是偶然

这是有意识地对抗 lost-in-the-middle 问题：模型对 context 开头和结尾的注意力权重高于中间部分。最重要的规则在两端各出现一次（一次在各自的专用段，一次在末尾的 reminders 里）。

---

## 维度四：模块化与懒加载

### 4.1 Skills 系统：prompt 的"按需扩展"机制

System prompt 里只放元数据：
```xml
<skill>
  <name>deep-research</name>
  <description>Use this skill instead of WebSearch for ANY question requiring
  web research. Trigger on queries like "what is X"...</description>
  <location>/mnt/skills/public/deep-research/SKILL.md</location>
</skill>
```

完整内容（几百到几千 token 的详细 SOP）在需要时由模型主动 `read_file` 加载。

**解决的两个问题**：
1. 20 个 skill 全量放入 system prompt 会消耗 2000-5000 token，且让 context 充斥大量低相关内容
2. 按需加载让相关内容在"需要它的那一轮"进入 context 活跃窗口，大幅减少 lost-in-the-middle

### 4.2 Skill 文件本身的 prompt 设计

以 `skills/public/deep-research/SKILL.md` 为例，它不是一段"说明文"，而是一套**可执行的 SOP**：

- Phase 1（广度探索）：具体告诉模型先做什么搜索、识别哪些维度
- Phase 2（深度挖掘）：每个维度如何深入
- 明确的"触发条件"（frontmatter 的 description）：模型看到这段 description 就知道什么时候该加载

这个设计的本质：**skill = 可插拔的专家知识 + 专家工作流，而不是简单的"指令集"**。

### 4.3 SOUL.md：人格层的模块化

```
完整 Agent = SYSTEM_PROMPT_TEMPLATE（通用规则）
           + <soul>（用户自定义的 SOUL.md 内容）
           + 运行时动态注入（记忆、日期）
```

这个分层让一套系统可以运行任意数量的"定制 agent"，而不需要每个 agent 维护一份完整 system prompt。SOUL.md 只需要写"这个 agent 是谁、擅长什么、有什么偏好"，通用规则由框架负责。

---

## 维度五：Token 预算

### 5.1 各模块的 token 量级估算

基于 `prompt.py` 源码分析（v2.0.0-rc1）：

| 模块 | 是否可选 | 估算 token 量 | 预算设计说明 |
|---|---|---|---|
| Base system prompt（role/thinking/clarification/citations/reminders） | 必选 | ~1800-2500 | clarification 段约 300 token，citations 段约 200 token |
| Skills 元数据（~20 个 skill） | 可配置 | ~500-800 | 只放 name+desc+path，不放全文 |
| Subagent section | 仅启用时 | ~1000-1500 | 大量示例代码，这是整个 prompt 里最"重"的可选模块 |
| Memory 注入（上限硬编码） | 可配置 | ≤2000 | `format_memory_for_injection()` 中 `max_tokens=2000` 硬上限 |
| Dynamic date/file reminder | 必选 | ~50-100 | 每轮注入，极轻量 |
| **典型全功能配置总计** | | **~4500-7000** | |

### 5.2 预算决策背后的工程权衡

**Memory 上限 2000 token**（`memory/prompt.py`）：
- 记忆越多越有用，但 context 成本不能失控
- 2000 token 约等于 1500 个英文单词，足以放入用户的核心 facts
- 超出预算时按置信度降序截断，保留最重要的内容

**Subagent section 做成可关闭的开关**：
- 它是整个 system prompt 里 token 消耗最大的可选模块（1000-1500 token）
- 不需要并发任务的场景（如简单问答）不应该为此付出 context 成本

**Skills 只存元数据**（每个约 30-50 token）：
- 如果全量展开 20 个 skill，可能消耗 5000-15000 token
- 通过懒加载，平均每次只有 1-2 个 skill 的完整内容进入 context

**Facts 按置信度排序截断**（`format_memory_for_injection()`）：
- 高置信度 facts（0.9+，明确陈述的）优先保留
- 低置信度的推断性内容在预算紧张时最先被截断

---

## 关键视角六：Prompt 与中间件的职责分工

这是很多人忽视的一个角度：**DeerFlow 里部分行为不是靠 prompt 保证的，而是靠代码强制执行的。**

| 行为 | 靠 prompt 还是代码 | 具体实现 |
|---|---|---|
| "最多 3 个并发 task" | 两者都有 | Prompt 说明规则 + `SubagentLimitMiddleware` 截断超出的调用 |
| "ask_clarification 后中断执行" | 代码 | `ClarificationMiddleware` 把工具调用翻译成 `Command(goto=END)` |
| "工具调用错误不崩溃" | 代码 | `ToolErrorHandlingMiddleware` 把异常转换为 ToolMessage |
| "循环检测" | 代码 | `LoopDetectionMiddleware` 检测重复调用模式 |
| "安全终止" | 代码 | `SafetyFinishReasonMiddleware` 清理 tool_calls |

**设计哲学**：**Prompt 驯化意图（让模型理解规则、倾向于遵守），代码保证边界（无论模型是否理解，硬约束不可违反）。**

这意味着：
- 对于"重要但模型偶尔会犯错"的规则，只靠 prompt 不够，要加中间件保底
- Prompt 的作用是减少需要中间件介入的频次，而不是替代中间件

---

## 学习路径建议

### 第一阶段（1 周）：读懂整体架构

**目标**：搞清楚 prompt 的"层次结构"，理解静态/动态分离的设计动机。

1. 精读 `backend/packages/harness/deerflow/agents/lead_agent/prompt.py` 全文，重点关注：
   - `SYSTEM_PROMPT_TEMPLATE` 的每个 XML 段，问自己"如果缺了这段会怎样"
   - `apply_prompt_template()` 函数，理解哪些内容是动态拼接的、哪些是静态的
   - 第 799-803 行注释（static/dynamic split 的设计说明）

2. 读 `agents/middlewares/dynamic_context_middleware.py`，理解 `<system-reminder>` 注入机制

### 第二阶段（1 周）：深入行为工程

**目标**：提炼出"让模型可靠执行"的写法规律。

1. 把 `<clarification_system>` 段（prompt.py 第 385-447 行）拆解分析：
   - 5 种场景分类的命名逻辑
   - 每种场景的"触发例子"是怎么选的
   - 反例（❌）和正例（✅）的对称结构

2. 把 `_build_subagent_section()` 函数（prompt.py 第 214-362 行）拆解分析：
   - 硬约束的三重强调写法
   - 伪代码示例的作用
   - "Counter-Example - Direct Execution"反例的位置选择

3. **实践练习**：选一个你自己的 prompt，用 DeerFlow 的技法重写，对比前后

### 第三阶段（1 周）：理解 Skill 的 SOP 设计

**目标**：理解"可插拔的专家知识"是怎么写的。

1. 精读以下 skill（按复杂度递增）：
   - `skills/public/data-analysis/SKILL.md`（相对简单）
   - `skills/public/deep-research/SKILL.md`（多阶段 SOP）
   - `skills/public/bootstrap/SKILL.md`（多文件引用结构）

2. 注意每个 skill 的 frontmatter `description` 字段——这是触发条件，写法决定了模型什么时候主动加载

### 第四阶段（1 周）：Memory 提示词设计

**目标**：理解"结构化提取型 prompt"的设计范式。

1. 精读 `agents/memory/prompt.py` 中的 `MEMORY_UPDATE_PROMPT`：
   - JSON Schema 输出格式的设计（`shouldUpdate` 字段的作用）
   - 置信度分层（0.9+ / 0.7-0.8 / 0.5-0.6）及其工程含义
   - `correction` 类别的特殊设计（捕捉"模型犯错→用户纠正"信号）
   - "IMPORTANT: Do NOT record file upload events"——这类反模式提醒为什么有效

### 第五阶段（持续）：横向对比与总结

把以下三段 prompt 放在一起横向对比，总结"强制行为型 prompt"的共同写法模式：
- `<subagent_system>`（并发限制）
- `<clarification_system>`（澄清优先）
- `_create_todo_list_middleware()` 里的 TodoList prompt（任务状态管理）

---

## 核心设计原则总结

1. **场景化 > 规则化**：用"具名场景 + 触发例子"替代抽象规则，降低模型的模式匹配成本

2. **后果描述 > 权威要求**：告诉模型"违反会发生什么"，激活因果推理而非盲目服从

3. **缓存友好分层**：静态内容进 system prompt，动态内容进 `<system-reminder>`，按需加载的进文件

4. **代码 > 自然语言描述程序性步骤**：执行序列用伪代码，行为规范用自然语言

5. **模块化 + 懒加载**：skills 只存元数据，详细 SOP 按需读取，避免 context 膨胀

6. **prompt 驯化意图，代码保证边界**：关键的硬约束要有中间件兜底，不能只靠 prompt

7. **末尾强化高优先级规则**：`<critical_reminders>` 在末尾重复最重要的规则，对抗 lost-in-the-middle
