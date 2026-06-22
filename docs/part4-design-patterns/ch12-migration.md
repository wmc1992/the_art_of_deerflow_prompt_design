# 第 12 章：从 DeerFlow 到你的系统——实战迁移指南

## 本章导读

本书前十一章系统分析了 DeerFlow 的 25 个 Prompt 及其设计原则。本章是将这些分析转化为实践行动的桥梁：面对你自己的 Agent 系统，如何判断应该应用哪些模式？如何评估当前 Prompt 的健康程度？如何在不重写所有内容的情况下逐步改善？

---

## 12.1 从分析到行动的三个误区

在开始迁移指南之前，先点明三个常见误区：

**误区一：完整复制 DeerFlow 的结构**

DeerFlow 的 Prompt 体系是为其特定需求设计的：超级 Agent、多子 Agent 并发、22 个技能、长会话记忆。如果你的系统是一个简单的单轮问答 Agent，复制整个 P-01 的 10,000 token 结构，不会让你的 Agent 更好，只会让它更慢、更贵。

迁移不是复制，而是**有选择地借鉴针对你问题的解法**。

**误区二：一次性全部应用七个模式**

七个模式同时引入，你无法判断哪个改变产生了哪个效果。更糟糕的是，某些模式的引入可能相互干扰（如同时引入渐进式加载和层级密度调整，token 成本的变化无法归因）。

**正确做法**：识别你的系统最突出的问题，优先应用对应的模式，一次一个，观察效果。

**误区三：把这些模式当成万能药**

Prompt 工程没有万能解法。DeerFlow 的某些设计选择是有代价的——P-01 的 10,000 token 系统 Prompt 增加了首次请求的处理成本；P-02 的子 Agent 系统增加了协调开销。这些代价在 DeerFlow 的使用场景中是合理的，但在你的系统中不一定。

---

## 12.2 系统诊断：找到你的 Prompt 的主要问题

在应用任何模式之前，先诊断你的系统当前面临的主要问题。下面是一个快速诊断框架：

### 诊断维度一：行为一致性

**症状**：同样的任务，模型有时执行正确，有时不遵循规则。

**可能原因和对应模式**：

| 具体表现 | 可能原因 | 对应模式 |
|---------|---------|---------|
| 某条规则时遵守时违反 | 规则描述不够清晰，触发条件模糊 | 命名场景约束（模式三）|
| 某个工具过度使用 | 缺少排斥边界定义 | 双向边界定义（模式五）|
| 重要约束被跳过 | 约束用一行描述，与其他规则等权 | 层级指令密度（模式七）+ 后果驱动约束（模式四）|
| 不同轮次行为不同 | 动态内容混入系统 Prompt，导致前缀不稳定 | 静动分离（模式一）|

### 诊断维度二：Token 成本

**症状**：每次请求的 token 消耗异常高，或随着功能增加线性增长。

| 具体表现 | 可能原因 | 对应模式 |
|---------|---------|---------|
| 系统 Prompt 超过 5000 token | 大量内容不加区分地全量注入 | 渐进式加载（模式六）|
| 添加新功能导致 token 暴增 | 没有按需加载机制 | 渐进式加载（模式六）|
| 前缀缓存命中率低 | 系统 Prompt 每次略有变化 | 静动分离（模式一）|

**检测方法**：对比连续两次相同类型的请求，检查系统 Prompt 内容是否完全相同（bit-level identical）。若不同，定位变化的部分，考虑将其移至动态注入。

### 诊断维度三：可维护性

**症状**：修改一条规则时，不确定会影响哪些行为；Prompt 文件越来越长，越来越难以推理。

| 具体表现 | 可能原因 | 对应模式 |
|---------|---------|---------|
| 不同功能的规则混在一起 | 缺乏结构化组织 | XML 标签锚定（模式二）|
| 重要规则与次要规则看起来一样 | 所有规则用相同的格式 | 层级指令密度（模式七）|
| 新加规则效果难以评估 | 结构不清晰，干扰项太多 | XML 标签锚定（模式二）+ 层级指令密度（模式七）|

---

## 12.3 增量迁移路径：五个阶段

假设你有一个已有的 Agent 系统，以下是建议的增量迁移顺序，每个阶段风险可控、效果可观察。

### 阶段 1：结构化（成本低，收益立竿见影）

**对应模式**：XML 标签锚定（模式二）

**操作**：将现有系统 Prompt 按语义功能分块，加上 XML 标签。

```xml
<!-- 之前 -->
You are an assistant. Be helpful and concise.
When the user asks about X, do Y.
When editing files, prefer minimal changes.
...

<!-- 之后 -->
<role>
You are an assistant. Be helpful and concise.
</role>

<editing_guidelines>
When editing files, prefer minimal changes.
</editing_guidelines>

<behavior_rules>
When the user asks about X, do Y.
</behavior_rules>
```

这个改动不会改变任何规则的内容，但会让后续的优化（调整密度、增加规则）变得有据可查。

**预期效果**：行为变化不大（内容没变），但代码可维护性显著提升，为后续优化打下基础。

### 阶段 2：静动分离（成本适中，收益显著）

**对应模式**：静动分离（模式一）

**操作**：识别系统 Prompt 中哪些内容每次请求都相同，哪些每次不同。

常见的动态内容：
- 当前时间（`The current time is {now}`）
- 用户状态（用户名、账号信息、权限级别）
- 会话特定上下文（这次上传的文件、这次选择的模式）
- 外部工具列表（如果工具集会动态变化）

将这些内容从 SystemMessage 中移出，改为在每轮请求时以 `HumanMessage` 追加：

```python
# 之前（在 system message 中，每次内容略有变化）
system_msg = f"You are assistant. Current user: {user_name}. Time: {now}."

# 之后（system message 完全静态）
system_msg = "You are assistant."
# 动态内容通过 human message 追加
context_msg = HumanMessage(content=f"<context>User: {user_name}. Time: {now}.</context>")
messages = [SystemMessage(system_msg)] + history + [context_msg, new_user_msg]
```

**预期效果**：前缀缓存命中率提升，相同任务的首次 token 成本降低 20-40%（取决于系统 Prompt 的原有大小）。

### 阶段 3：约束优化（成本低，解决特定问题）

**对应模式**：后果驱动约束（模式四）+ 双向边界定义（模式五）

**操作**：找出最常被违反的 3 条规则，对每条规则执行以下检查：

1. **有没有说清楚为什么这条规则存在？**
   - 没有 → 加上后果说明
   - 例：`"Do not call X twice"` → `"Do not call X twice — a second call will invalidate the first result and you will need to start over"`

2. **有没有同时定义"不应该做"的边界？**
   - 没有 → 加上排斥条件
   - 例：`"Use tool Y for file operations"` → 加上 `"Do NOT use tool Y for: simple reads under 10 lines, operations on non-existent files"`

**预期效果**：目标规则的遵守率提升，副作用较小（只修改了特定规则的描述方式）。

### 阶段 4：决策明确化（成本适中，解决一致性问题）

**对应模式**：命名场景约束（模式三）

**操作**：找出系统中"什么情况下做 X"最模糊的决策点。常见候选：
- 何时主动提问？
- 何时直接执行 vs 先确认？
- 何时使用工具 A vs 工具 B？

对这些决策点，逐一定义命名场景：

```
When to ask for clarification:
1. missing_required_param: The task cannot proceed without a specific value
   (Example: "Create a report" — for what time period?)
2. ambiguous_target: The instruction could apply to multiple objects
   (Example: "Delete the file" — which file?)
3. irreversible_action: The action cannot be undone and was not explicitly confirmed
   (Example: "Clean up the database" involves deleting records)

Do NOT ask when:
- The information can be inferred from context
- A reasonable default exists and the user hasn't indicated preference
```

> **中文附注**：
>
> 何时要求澄清：
> 1. `missing_required_param`（缺少必要参数）：任务无法在没有特定值的情况下继续
>    （示例："创建报告"——覆盖哪个时间段？）
> 2. `ambiguous_target`（目标模糊）：指令可能适用于多个对象
>    （示例："删除文件"——哪个文件？）
> 3. `irreversible_action`（不可逆操作）：操作无法撤销且未得到明确确认
>    （示例："清理数据库"涉及删除记录）
>
> 以下情况**不要**询问：
> - 信息可以从上下文中推断
> - 存在合理的默认值且用户未表明偏好

**预期效果**：模型的澄清行为从"随机"变为"可预测"。测试可以针对每个场景类型验证正确行为。

### 阶段 5：内容按需加载（成本高，解决规模问题）

**对应模式**：渐进式加载（模式六）

这是代码改动最大的阶段，适合系统 Prompt 已经超过 5000 token、或内容集合在快速增长的系统。

**操作步骤**：

1. 识别系统 Prompt 中的大型可选内容（技能说明、工具文档、配置示例）

2. 为每个大型内容创建独立文件，在系统 Prompt 中只保留摘要：
   ```
   Available skills:
   - **sql**: Run SQL queries and analyze data. (Use: /sql)
   - **viz**: Create data visualizations. (Use: /viz)
   - **report**: Generate formatted reports. (Use: /report)
   (Use /skill-name to activate a skill and load its full guide)
   ```

   > **中文附注**：
   >
   > 可用技能：
   > - **sql**：执行 SQL 查询并分析数据。（使用：/sql）
   > - **viz**：创建数据可视化图表。（使用：/viz）
   > - **report**：生成格式化报告。（使用：/report）
   > （输入 /技能名称 以激活技能并加载完整指南）

3. 实现 `load_skill(name)` 工具，供模型在被激活时调用：
   ```python
   async def load_skill(skill_name: str) -> str:
       path = SKILLS_DIR / f"{skill_name}.md"
       if path.exists():
           return path.read_text()
       return f"Skill '{skill_name}' not found."
   ```

4. 添加加载缓存：在会话级别缓存已加载的技能，避免同一会话内重复加载

**预期效果**：系统 Prompt 大幅缩短，每次请求固定 token 开销减少，前缀缓存命中范围扩大。

---

## 12.4 不同规模系统的迁移侧重

并非所有系统都需要应用全部七个模式。以下是按系统规模的建议：

### 轻量级系统（单 Agent，<2000 token 系统 Prompt）

优先：阶段 1（XML 结构化）+ 阶段 3（约束优化）

不必要：阶段 5（渐进式加载）——你的内容量不构成 token 问题

重点关注：模型的行为一致性（命名场景约束、后果驱动约束）

### 中型系统（单 Agent，2000-8000 token 系统 Prompt，3-10 个工具）

优先：阶段 1-3（结构化 + 静动分离 + 约束优化）

考虑：阶段 4（如果模型的某些决策点经常混乱）

不必要：阶段 5（除非你有 5+ 个大型可选内容）

### 大型系统（多 Agent，>8000 token 系统 Prompt，10+ 个工具/技能）

所有阶段都值得考虑，但顺序建议：
- 先完成阶段 2（静动分离）——影响最大，是后续优化的基础
- 再做阶段 5（渐进式加载）——大型系统 token 成本压力最大
- 最后做阶段 3-4（约束优化）——在结构稳定后才做行为细化

---

## 12.5 一份 Prompt 设计评审清单

在完成任何 Prompt 的编写或修改后，用以下清单进行自查：

### 结构清单

- [ ] 系统 Prompt 中的不同功能区域是否用 XML 标签明确分隔？
- [ ] 每个 XML 标签的名称是否直接描述了该区域的功能？
- [ ] 系统 Prompt 是否在不同请求间完全静态（bit-level identical）？
- [ ] 动态内容是否通过 HumanMessage 追加而非嵌入 SystemMessage？

### 规则表达清单

- [ ] 最重要的 3 条规则是否有足够的篇幅说明（触发条件、示例、后果）？
- [ ] 是否有至少一条规则采用了后果描述（"如果不这样，X 将发生"）？
- [ ] 对每个工具/功能，是否同时定义了"何时用"和"何时不用"？
- [ ] 是否存在任何依赖"模型自行推断"的决策点？如果有，是否已命名场景？

### Token 效率清单

- [ ] 系统 Prompt 中是否有内容在大多数请求中都用不到（应考虑渐进加载）？
- [ ] 相同的内容是否被重复描述了（通过不同措辞）？
- [ ] 对于简单规则（只需一句话），是否使用了过多的篇幅说明？

### 可测试性清单

- [ ] 是否可以为每个命名场景写出至少一个正例和反例？
- [ ] 是否可以通过修改一个 XML 块来验证该块的规则变化效果（不影响其他块）？
- [ ] 是否有至少 3 组回归测试用例，验证主要行为路径？

---

## 12.6 从 DeerFlow 学到的最重要一课

回顾本书分析的 25 个 Prompt，有一个贯穿始终的设计哲学值得单独强调：

**约束的设计服务于可预测性，而不是全面控制。**

DeerFlow 的系统 Prompt 没有试图穷举所有规则——它定义了关键决策点的边界（澄清、子 Agent、TodoList），允许模型在边界内自主判断。这种设计使系统既有明确的行为约束，又有足够的灵活性处理未预见的情形。

相反，试图用 Prompt 控制模型的每一步行为，往往会导致两个问题：
1. **规则冲突**：规则越多，边缘情况下规则相互矛盾的概率越高，模型行为反而变得不可预测
2. **新情形失控**：过度约束的模型在遇到未被规则覆盖的新情形时，往往不知所措

**最好的 Prompt 设计，是为模型提供明确的决策框架，而不是决策的全部答案。**

DeerFlow 的 `<clarification_system>` 教给模型的不是"问哪些问题"，而是"如何判断哪类情况需要澄清"——这种框架是可泛化的，而不只是一个问题列表。P-02 的 DECOMPOSE→DELEGATE→SYNTHESIZE 不是一个任务分配手册，而是一个决策过程的模板。

这个教训对你自己的 Prompt 设计同样适用：**给框架，不给答案。**

---

## 12.7 写在最后：Prompt 工程的本质

阅读本书，你会发现 DeerFlow 的 Prompt 设计遵循了工程领域的一个普遍原则：**明确性优于聪明性**。

- `silently discarded` 比 `not recommended` 更清晰
- `3+ distinct steps` 比 `complex tasks` 更明确
- `missing_info / ambiguous_requirement / approach_choice` 比 `unclear situations` 更精确

在 Prompt 工程中，你永远在与模型的"创造性解读"作斗争——模型会尽力完成任务，但它的解读方式可能不是你期望的。明确性是对抗这种解读漂移最有效的武器。

DeerFlow 花了数十个版本迭代才达到当前的 Prompt 设计质量，`issue #3189` 这样的实战经验被直接写进了代码注释，`correction` 类别的专门设计来自对记忆系统实际失效案例的反思。

**Prompt 工程不是一次性的设计工作，而是持续的测量-诊断-改进循环。** DeerFlow 给我们展示的，不只是一套当前的 Prompt，而是一种严肃对待每一行文本的工程态度。

---

## 本章小结

- 迁移不是复制，而是有选择地借鉴对应你问题的解法
- 建议的五阶段增量迁移路径：结构化 → 静动分离 → 约束优化 → 决策明确化 → 渐进加载
- 不同规模系统的侧重不同：轻量系统关注一致性，大型系统优先解决 token 成本
- 最重要的设计哲学：给框架不给答案，明确性优于聪明性
- Prompt 工程是持续的测量-诊断-改进循环，而不是一次性设计工作

---

## 全书结语

《DeerFlow Prompt 设计艺术》至此完结。我们沿着从宏观到微观、从原则到实例的路径，系统走过了：

- 25 个 Prompt 的完整文本与来源（附录 A-B）
- 22 个技能文件的完整内容（附录 C-D）
- 整体架构与 18 个中间件管道（第 1-3 章）
- 主系统 Prompt 的十个 XML 块逐一解析（第 4-7 章）
- 子 Agent、记忆、安全防线三大子系统（第 8-10 章）
- 可迁移的七大设计模式与实战迁移指南（第 11-12 章）

DeerFlow 的 Prompt 体系并非完美——它在某些地方仍有权衡和妥协的痕迹。但正是这些权衡，让它成为一个学习对象：不是用来崇拜，而是用来理解工程约束如何塑造设计决策。

希望本书帮助你建立了对工业级 Agent Prompt 设计的直觉，并在你自己的系统中找到那几个值得认真对待的决策点。
