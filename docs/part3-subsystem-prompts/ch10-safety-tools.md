# 第 10 章：安全防线——Loop 检测、TodoList 与标题生成

## 本章导读

本章分析三组辅助 Prompt：P-13 至 P-16（Loop 检测系统的四条干预消息）、P-08/P-09（TodoList 中间件的系统 Prompt 和工具描述）、P-12（会话标题自动生成 Prompt）。这三组 Prompt 都是系统监控和辅助功能的一部分，而非主 Agent 的核心行为规范。

---

## 10.1 Loop 检测系统概述

在第 3 章，我们分析了 Loop 检测消息为什么必须以 `HumanMessage` 而非 `SystemMessage` 形式注入（避免破坏工具调用配对约束）。本章深入分析这四条消息的文本内容本身。

Loop 检测中间件实现了两层检测机制：

**第一层：哈希匹配检测**
- 对每次 AIMessage 中的工具调用集合计算哈希
- 相同哈希出现 ≥ `warn_threshold`（默认 3）次 → 注入 P-13 警告
- 相同哈希出现 ≥ `hard_limit`（默认 5）次 → 注入 P-15 强制停止

**第二层：工具频率检测**
- 统计每种工具的调用总次数（不看参数是否相同）
- 单工具调用 ≥ `tool_freq_warn`（默认 30）次 → 注入 P-14 频率警告
- 单工具调用 ≥ `tool_freq_hard_limit`（默认 50）次 → 注入 P-16 频率强制停止

两层机制覆盖了不同的循环模式：
- **哈希检测**：捕获完全相同的重复（如一直用相同参数调用同一个工具）
- **频率检测**：捕获"看似不同但实质重复"的循环（如用 `read_file` 读取 40 个不同文件却没有给出最终答案）

---

## 10.2 P-13：Loop 检测警告消息（完整原文）

```
[LOOP DETECTED] You are repeating the same tool calls. Stop calling tools and produce your final answer now. If you cannot complete the task, summarize what you accomplished so far.
```

> **中文附注**：[检测到循环] 你正在重复相同的工具调用。立即停止调用工具并给出最终答案。如果你无法完成任务，请总结你到目前为止完成的内容。

这是一条 38 个 token 的干预消息，用三句话完成了三件事：

**第一句**（诊断）：
```
[LOOP DETECTED] You are repeating the same tool calls.
```

> **中文附注**：[检测到循环] 你正在重复相同的工具调用。

方括号标记 `[LOOP DETECTED]` 是机器可读的信号前缀，也是系统日志中可以用于追踪的关键词。`"You are repeating"`是直接的陈述句，没有委婉或模糊。

**第二句**（指令）：
```
Stop calling tools and produce your final answer now.
```

> **中文附注**：立即停止调用工具并给出最终答案。

`Stop` 是命令式，`now` 强调立即性。这不是建议，而是一个紧急命令。

**第三句**（出路）：
```
If you cannot complete the task, summarize what you accomplished so far.
```

> **中文附注**：如果你无法完成任务，请总结你到目前为止完成的内容。

这句话非常重要——它为模型提供了一个"降级路径"。当模型陷入循环时，往往是因为它无法达到预期目标（找不到某个文件、搜索结果不满意等），而强制它"立即给出最终答案"如果没有出路会导致模型继续尝试工具调用。

`"summarize what you accomplished so far"` 允许模型以部分完成的状态退出，给用户一个可用的（即使不完整的）结果，而不是陷入更深的循环或崩溃退出。

---

## 10.3 P-14：工具频率警告消息（完整原文）

```
[LOOP DETECTED] You have called {tool_name} {count} times without producing a final answer. Stop calling tools and produce your final answer now. If you cannot complete the task, summarize what you accomplished so far.
```

> **中文附注**：[检测到循环] 你已调用 `{tool_name}` `{count}` 次而未给出最终答案。立即停止调用工具并给出最终答案。如果你无法完成任务，请总结你到目前为止完成的内容。

P-14 是 P-13 的频率版本，核心差异在第一句：

```
You have called {tool_name} {count} times without producing a final answer.
```

> **中文附注**：你已调用 `{tool_name}` `{count}` 次而未给出最终答案。

两个格式化变量：
- `{tool_name}`：具体的工具名（如 `read_file`、`web_search`）
- `{count}`：调用次数（如 `30`）

这两个变量让警告从通用（"你在循环"）变为具体（"你已经调用 read_file 30 次了"）。具体数字有额外的说服力：模型在看到"30 次"这个数字时，更容易触发自我评估——"确实，我读了 30 个文件还没给出答案，这是有问题的"。

---

## 10.4 P-15：Loop 强制停止消息（完整原文）

```
[FORCED STOP] Repeated tool calls exceeded the safety limit. Producing final answer with results collected so far.
```

> **中文附注**：[强制停止] 重复工具调用超出安全限制。正在使用目前收集到的结果生成最终答案。

P-15 与 P-13 的差异不只是标签从 `[LOOP DETECTED]` 变为 `[FORCED STOP]`：

```
Producing final answer with results collected so far.
```

> **中文附注**：正在使用目前收集到的结果生成最终答案。

P-13 是命令（`Stop calling tools and produce your final answer`），P-15 是陈述（`Producing final answer`）。这个语态变化很微妙：

- P-13：警告，模型仍然有选择权（虽然被强烈建议停止）
- P-15：强制，模型的工具调用已经被系统清空（`tool_calls = []`），它**没有**选择权

P-15 的陈述句语气对应了代码层面的事实：在注入 P-15 的同时，`LoopDetectionMiddleware` 会执行以下操作：

```python
content = self._append_text(last_msg.content, warning or _HARD_STOP_MSG)
stripped_msg = last_msg.model_copy(update=self._build_hard_stop_update(last_msg, content))
```

`_build_hard_stop_update()` 清空了 `tool_calls`，并将 `finish_reason` 从 `"tool_calls"` 改为 `"stop"`。P-15 文本被追加到已生成的 AIMessage 末尾，此时工具调用队列已经为空，模型唯一能做的就是生成文本输出。

这是 Prompt 文本与系统行为的完美协同：Prompt 的语态（陈述句）匹配了系统已经为它做出的决定（强制停止）。

---

## 10.5 P-16：工具频率强制停止消息（完整原文）

```
[FORCED STOP] Tool {tool_name} called {count} times — exceeded the per-tool safety limit. Producing final answer with results collected so far.
```

> **中文附注**：[强制停止] 工具 `{tool_name}` 被调用了 `{count}` 次——超过了每工具安全限制。正在使用目前收集到的结果生成最终答案。

P-16 与 P-15 的关系，等同于 P-14 与 P-13 的关系：通用的强制停止 → 带具体工具名和次数的频率强制停止。

`"— exceeded the per-tool safety limit"` 中的破折号是一个语义桥梁，解释了为什么触发强制停止，而不只是宣告停止。这让模型理解这是一个系统层面的保护机制，而不是随机的中断。

---

## 10.6 四条消息的比较设计

把四条消息放在一起看：

| | 触发机制 | 响应级别 | 包含具体工具/次数 |
|--|---------|---------|----------------|
| P-13 | 相同调用集哈希重复 ≥ warn | 警告（HumanMessage 注入）| 否 |
| P-14 | 单工具频率 ≥ freq_warn | 警告（HumanMessage 注入）| 是 |
| P-15 | 相同调用集哈希重复 ≥ hard | 强制停止（清空工具调用）| 否 |
| P-16 | 单工具频率 ≥ freq_hard | 强制停止（清空工具调用）| 是 |

四条消息形成了一个**2×2 矩阵**：（警告 vs 强制）×（通用 vs 频率）。每个格子的文本都与其对应的系统行为匹配：
- 警告消息使用命令式（`Stop calling tools`），因为模型仍然有机会自主改变
- 强制停止消息使用陈述式（`Producing final answer`），因为系统已经代替模型做了决定

---

## 10.7 P-08：TodoList 系统 Prompt 原文

```
<todo_list_system>
You have access to the `write_todos` tool to help you manage and track complex multi-step objectives.

**CRITICAL RULES:**
- Mark todos as completed IMMEDIATELY after finishing each step - do NOT batch completions
- Keep EXACTLY ONE task as `in_progress` at any time (unless tasks can run in parallel)
- Update the todo list in REAL-TIME as you work - this gives users visibility into your progress
- DO NOT use this tool for simple tasks (< 3 steps) - just complete them directly

**When to Use:**
This tool is designed for complex objectives that require systematic tracking:
- Complex multi-step tasks requiring 3+ distinct steps
- Non-trivial tasks needing careful planning and execution
- User explicitly requests a todo list
- User provides multiple tasks (numbered or comma-separated list)
- The plan may need revisions based on intermediate results

**When NOT to Use:**
- Single, straightforward tasks
- Trivial tasks (< 3 steps)
- Purely conversational or informational requests
- Simple tool calls where the approach is obvious

**Best Practices:**
- Break down complex tasks into smaller, actionable steps
- Use clear, descriptive task names
- Remove tasks that become irrelevant
- Add new tasks discovered during implementation
- Don't be afraid to revise the todo list as you learn more

**Task Management:**
Writing todos takes time and tokens - use it when helpful for managing complex problems, not for simple requests.
</todo_list_system>
```

> **中文附注**：
>
> `<todo_list_system>`
> 你有访问 `write_todos` 工具的权限，帮助你管理和追踪复杂的多步目标。
>
> **关键规则**：
> - 完成每个步骤后立即将 todo 标记为已完成——不得批量完成
> - 任何时候保持**恰好一个**任务为 `in_progress`（除非任务可以并行运行）
> - 工作时实时更新 todo 列表——让用户了解你的进度
> - 不得对简单任务（< 3 步）使用此工具——直接完成它们
>
> **何时使用**：专为需要系统性追踪的复杂目标设计——需要 3+ 个不同步骤的多步任务、需要仔细规划的非平凡任务、用户明确要求 todo 列表、用户提供多个任务、计划可能需要根据中间结果修订。
>
> **何时不使用**：单一直接的任务、平凡任务（< 3 步）、纯粹的对话或信息性请求、方法明显的简单工具调用。
>
> **最佳实践**：将复杂任务拆分为更小的可操作步骤；使用清晰、描述性的任务名称；移除不再相关的任务；添加实现过程中发现的新任务；随着了解加深，不要害怕修订 todo 列表。
>
> **任务管理**：编写 todo 需要时间和 token——在管理复杂问题时使用，而非简单请求。
> `</todo_list_system>`

### P-08 的注入方式

P-08 不是 P-01 的一部分。它由 `TodoMiddleware` 以独立的 `SystemMessage` 形式追加到消息链中，只有当 `is_plan_mode=True` 时才激活：

```python
# TodoMiddleware 的 before_model 逻辑
messages = [SystemMessage(content=self.system_prompt)] + existing_messages
```

这意味着在 Plan Mode 下，对话的系统 Prompt 实际上由两个 `SystemMessage` 组成：
1. P-01（主系统 Prompt）
2. P-08（TodoList 系统 Prompt，追加）

### P-08 的"实时更新"设计

```
- Update the todo list in REAL-TIME as you work - this gives users visibility into your progress
```

> **中文附注**：工作时实时更新 todo 列表——这让用户了解你的进度。

`gives users visibility into your progress` 解释了为什么要实时更新，而不只是说"必须实时更新"。这是后果驱动约束的另一个例子：告诉模型实时更新的**用户价值**（让用户看到进度），而不只是说"规则是这样"。

```
- Mark todos as completed IMMEDIATELY after finishing each step - do NOT batch completions
```

> **中文附注**：完成每个步骤后立即将 todo 标记为已完成——不得批量完成。

`do NOT batch completions` 预防了一个具体的反模式：模型完成了 5 个步骤之后，统一把这 5 个都标为完成。这种批量更新破坏了进度可视化的价值——用户在 5 步完成之前看到的都是一片"waiting"状态，最后突然全部变成"done"。

---

## 10.8 P-09：`write_todos` 工具描述原文

```
Use this tool to create and manage a structured task list for complex work sessions.

**IMPORTANT: Only use this tool for complex tasks (3+ steps). For simple requests, just do the work directly.**

## When to Use

Use this tool in these scenarios:
1. **Complex multi-step tasks**: When a task requires 3 or more distinct steps or actions
2. **Non-trivial tasks**: Tasks requiring careful planning or multiple operations
3. **User explicitly requests todo list**: When the user directly asks you to track tasks
4. **Multiple tasks**: When users provide a list of things to be done
5. **Dynamic planning**: When the plan may need updates based on intermediate results

## When NOT to Use

Skip this tool when:
1. The task is straightforward and takes less than 3 steps
2. The task is trivial and tracking provides no benefit
3. The task is purely conversational or informational
4. It's clear what needs to be done and you can just do it

## How to Use

1. **Starting a task**: Mark it as `in_progress` BEFORE beginning work
2. **Completing a task**: Mark it as `completed` IMMEDIATELY after finishing
3. **Updating the list**: Add new tasks, remove irrelevant ones, or update descriptions as needed
4. **Multiple updates**: You can make several updates at once (e.g., complete one task and start the next)

## Task States

- `pending`: Task not yet started
- `in_progress`: Currently working on (can have multiple if tasks run in parallel)
- `completed`: Task finished successfully

## Task Completion Requirements

**CRITICAL: Only mark a task as completed when you have FULLY accomplished it.**

Never mark a task as completed if:
- There are unresolved issues or errors
- Work is partial or incomplete
- You encountered blockers preventing completion
- You couldn't find necessary resources or dependencies
- Quality standards haven't been met

If blocked, keep the task as `in_progress` and create a new task describing what needs to be resolved.

## Best Practices

- Create specific, actionable items
- Break complex tasks into smaller, manageable steps
- Use clear, descriptive task names
- Update task status in real-time as you work
- Mark tasks complete IMMEDIATELY after finishing (don't batch completions)
- Remove tasks that are no longer relevant
- **IMPORTANT**: When you write the todo list, mark your first task(s) as `in_progress` immediately
- **IMPORTANT**: Unless all tasks are completed, always have at least one task `in_progress` to show progress

Being proactive with task management demonstrates thoroughness and ensures all requirements are completed successfully.

**Remember**: If you only need a few tool calls to complete a task and it's clear what to do, it's better to just do the task directly and NOT use this tool at all.
```

> **中文附注**：
>
> 使用此工具为复杂工作会话创建和管理结构化任务列表。
>
> **重要**：只对复杂任务（3+ 步）使用此工具。对于简单请求，直接完成工作。
>
> **何时使用**：（1）复杂多步任务（需要 3 个或以上不同步骤）；（2）非平凡任务（需要仔细规划或多项操作）；（3）用户明确要求 todo 列表；（4）用户提供多个任务清单；（5）动态规划（计划可能需要根据中间结果更新）。
>
> **何时不使用**：任务简单且不足 3 步；任务平凡且追踪无益；任务纯属对话或信息性；做什么一目了然、直接做即可。
>
> **使用方式**：开始任务前标为 `in_progress`；完成后立即标为 `completed`；根据需要添加/删除/更新任务；可一次进行多项更新。
>
> **任务状态**：`pending`（未开始）/ `in_progress`（进行中，并行任务可以有多个）/ `completed`（已成功完成）。
>
> **任务完成要求**：**关键**：只有在**完全**完成任务时才将其标记为已完成。以下情况不得标为完成：存在未解决问题或错误；工作部分完成或不完整；遇到阻碍完成的障碍；无法找到必要资源或依赖项；质量标准未达到。如受阻，将任务保持为 `in_progress` 并创建新任务描述需要解决的问题。
>
> **最佳实践**：创建具体可操作的任务；拆分复杂任务；使用清晰描述性的任务名称；实时更新状态；完成后立即标记（不批量）；移除不再相关的任务；写 todo 后立即将第一个任务标为 `in_progress`；除非全部完成，始终保持至少一个 `in_progress`。
>
> **记住**：如果只需几次工具调用就能完成任务且做什么一目了然，直接做更好，不必使用此工具。

### P-09 的"任务完成标准"设计

P-09 中最细致的部分是任务完成标准：

```
**CRITICAL: Only mark a task as completed when you have FULLY accomplished it.**

Never mark a task as completed if:
- There are unresolved issues or errors
- Work is partial or incomplete
- You encountered blockers preventing completion
- You couldn't find necessary resources or dependencies
- Quality standards haven't been met
```

> **中文附注**：
>
> **关键**：只有在**完全**完成任务时才将其标记为已完成。
>
> 以下情况不得将任务标记为已完成：
> - 存在未解决的问题或错误
> - 工作部分完成或不完整
> - 遇到阻碍完成的障碍
> - 无法找到必要的资源或依赖项
> - 质量标准未达到

这五个"不应标为完成"的条件覆盖了模型可能错误标为"完成"的主要场景。

特别值得注意的是最后一条：

```
- Quality standards haven't been met
```

这是一个主观标准，比其他四条都更难以客观评估。把它明确列出，是在提醒模型不要仅仅因为"我执行了这个步骤"就标为完成——还需要评估执行结果的质量。

`If blocked, keep the task as in_progress and create a new task describing what needs to be resolved.`

> **中文附注**：如受阻，将任务保持为 `in_progress` 并创建新任务描述需要解决的问题。

这条规则提供了被阻塞时的标准处理方式：不是放弃任务，而是创建一个专门描述阻塞原因的新任务。这保持了进度可视化的完整性，同时也给用户提供了调试信息。

---

## 10.9 P-12：标题生成 Prompt（完整原文）

```
Generate a concise title (max {max_words} words) for this conversation.
User: {user_msg}
Assistant: {assistant_msg}

Return ONLY the title, no quotes, no explanation.
```

> **中文附注**：
>
> 为这次对话生成一个简洁的标题（最多 `{max_words}` 个词）。
> 用户：`{user_msg}`
> 助手：`{assistant_msg}`
>
> 只返回标题，不加引号，不加解释。

默认参数：`max_words=6`，`max_chars=60`。

这是本书 25 个 Prompt 中最短的一个，完整内容不超过 30 个 token（不含占位符填充内容）。

### P-12 的极简设计

P-12 的极简是合理的：生成对话标题是一个**边界清晰的单一任务**，不需要复杂的规范。过度描述不会改善结果，只会增加 token 成本。

`"Return ONLY the title, no quotes, no explanation"` 预防了 LLM 的常见输出习惯——加上引号、加上"这个对话的标题是..."这样的前缀。这三个"no"明确了输出格式：纯文本标题，没有任何装饰。

### 标题生成的触发机制

P-12 由 `TitleMiddleware` 在**第一次完整交换之后**异步触发：

```python
# TitleMiddleware 的 after_model 逻辑（简化）
if self._should_generate_title(state):
    asyncio.create_task(self._generate_title(state))
```

`create_task`（而非 `await`）意味着标题生成完全不阻塞主响应流程。用户会先看到 Assistant 的回复，标题稍后异步更新到会话元数据。

P-12 是在**独立 LLM 调用**中运行的，使用的模型可以与主 Agent 不同（通过 `title.model_name` 配置指定较小的模型），因为标题生成不需要强大的推理能力。

### `{user_msg}` 和 `{assistant_msg}` 的截断策略

`TitleMiddleware` 在填充 P-12 之前会对消息内容进行截断，避免将超长的对话内容喂给标题生成模型。具体地，它提取第一条用户消息和第一条 AI 响应的前若干字符（约 500-1000 字符），而不是完整内容。

标题生成只需要理解对话的**主题**，不需要完整的技术细节。这是 Token 预算意识（原则六）在子任务层面的体现。

---

## 10.10 三组 Prompt 的共同模式

回顾本章的三组 Prompt，可以发现它们共享了一个模式：**适用范围的精确划定**。

每一组 Prompt 都花了相当篇幅定义"什么时候**不**用这个功能"：

- Loop 检测：不阻止普通的工具使用，只在超过阈值时介入
- TodoList：明确说"不要用在简单任务上（< 3 步）"，多次重复这个限制
- 标题生成：不需要定义"不用"场景，但通过极简设计隐含了"只做这一件事"

这种"限制优先"的设计倾向，使辅助功能不会过度干预主 Agent 的正常工作，同时在真正需要时精准介入。

---

## 10.11 P-08/P-09 与 P-02 的对比：两种编排模式

P-08/P-09（TodoList，Plan Mode）和 P-02（子 Agent 编排）都是为了管理复杂的多步骤任务，但它们代表了两种不同的执行模型：

| 维度 | P-08/P-09（TodoList）| P-02（子 Agent）|
|------|---------------------|----------------|
| **执行者** | 主 Agent 自己逐步执行 | 多个子 Agent 并行执行 |
| **适用任务** | 有强顺序依赖的任务 | 可以独立并行的子任务 |
| **可见性** | 实时更新的任务列表 | 并行运行，结果最后汇总 |
| **失败处理** | 单步失败可以标记并继续 | 子 Agent 失败返回错误信息 |
| **交互性** | 主 Agent 可以随时与用户交互 | 子 Agent 不能发起澄清 |

两种模式可以共存：启用 Plan Mode 的同时也可以启用 subagent_enabled。此时主 Agent 可以用 TodoList 追踪整体任务进度，同时对其中可以并行的子任务启动子 Agent。

---

## 本章小结

- P-13 至 P-16 构成了分级干预的 Loop 检测系统：两种检测机制（哈希匹配 + 工具频率）× 两种干预级别（警告 + 强制）= 四条消息
- 警告消息使用命令式语气（模型有选择），强制消息使用陈述式语气（系统已代为决定），两者的语态与系统行为精确匹配
- P-13 的"降级路径"设计（`summarize what you accomplished so far`）是危机状态下的优雅退出机制，保证用户总能得到可用的（即使不完整的）结果
- P-08 的"实时更新"规则通过解释用户价值（而非只说规则）来驱动行为；P-09 的任务完成标准覆盖了模型可能误判完成状态的五种情况
- P-12 是本书最短的 Prompt（约 30 token），极简设计对应了标题生成的单一任务边界；异步独立执行不占主流程预算
