# 第 6 章：技能系统与延迟工具（`<skill_system>` / `<available-deferred-tools>`）

## 本章导读

本章分析 P-01 中两个涉及"能力扩展"的段落：`<skill_system>`（P-06）和 `<available-deferred-tools>`（P-07）。这两个机制共同回答了同一个问题：**当 Agent 拥有数十个外部工具和工作流时，如何在不爆炸上下文体积的前提下让模型知道这些能力的存在？**

本章同时分析 P-03（技能自演化段落）和 P-25（`tool_search` 工具描述），它们是这两个机制的关键组成部分。

---

## 6.1 P-06：`<skill_system>` 段落原文

`<skill_system>` 由 `_get_cached_skills_prompt_section()` 函数生成。以下是完整的模板内容（`{skill_evolution_section}` 和 `{skills_list}` 为占位符）：

```
<skill_system>
You have access to skills that provide optimized workflows for specific tasks. Each skill contains best practices, frameworks, and references to additional resources.

**Progressive Loading Pattern:**
1. When a user query matches a skill's use case, immediately call `read_file` on the skill's main file using the path attribute provided in the skill tag below
2. Read and understand the skill's workflow and instructions
3. The skill file contains references to external resources under the same folder
4. Load referenced resources only when needed during execution
5. Follow the skill's instructions precisely

**Explicit Slash Skill Activation:**
- If the user starts a request with `/<skill-name>`, that skill was explicitly requested for the current turn.
- Follow the activated skill before choosing a general workflow.
- The runtime injects the activated skill content for explicit slash activations; do not call `read_file` for that SKILL.md again unless the injected skill references supporting resources you need.

**Skills are located at:** {container_base_path}
{skill_evolution_section}
{skills_list}

</skill_system>
```

`{skills_list}` 渲染后的实际形态（以 22 个内置技能中的 deep-research 为例）：

```
<available_skills>
    <skill>
        <name>deep-research</name>
        <description>Use this skill instead of WebSearch for ANY question that benefits from comprehensive, multi-source research... [built-in]</description>
        <location>/mnt/skills/public/deep-research/SKILL.md</location>
    </skill>
    ... （22 个技能，每个约 4 行）
</available_skills>
```

---

## 6.2 Progressive Loading Pattern 解析

`<skill_system>` 的核心是五步"渐进式加载模式"：

```
**Progressive Loading Pattern:**
1. When a user query matches a skill's use case, immediately call `read_file` on the skill's main file using the path attribute provided in the skill tag below
2. Read and understand the skill's workflow and instructions
3. The skill file contains references to external resources under the same folder
4. Load referenced resources only when needed during execution
5. Follow the skill's instructions precisely
```

这五步定义了一个**延迟加载工作流**：

- 步骤 1：技能匹配 → 加载主文件（触发点）
- 步骤 2：理解工作流
- 步骤 3：识别引用的外部资源
- 步骤 4：按需加载外部资源（而不是预加载全部）
- 步骤 5：严格执行技能指令

步骤 4 是这个模式的第二层懒加载：即使加载了 SKILL.md，文件中引用的其他资源（如模板文件、示例文档等）也不会立即加载，只在执行过程中遇到需要时才按需读取。这是懒加载原则在两个层次上的应用。

### 为什么用 `read_file` 而非工具调用？

步骤 1 中，加载 SKILL.md 的方式是 `read_file` 工具调用，而不是一个专门的"技能加载"工具。这个选择有两个好处：

**一致性**：`read_file` 是沙箱的通用文件读取工具，不需要为"读取技能"创建特殊的工具绑定。技能文件就是文件，通用工具读取通用文件。

**透明性**：用户可以在对话历史中看到模型调用了 `read_file("/mnt/skills/public/deep-research/SKILL.md")`，而不是一个神秘的"load_skill"调用。这符合"open-source"的透明性立场。

---

## 6.3 Slash Skill Activation 设计

```
**Explicit Slash Skill Activation:**
- If the user starts a request with `/<skill-name>`, that skill was explicitly requested for the current turn.
- Follow the activated skill before choosing a general workflow.
- The runtime injects the activated skill content for explicit slash activations; do not call `read_file` for that SKILL.md again unless the injected skill references supporting resources you need.
```

这段文字处理了一个重要的边界情况：当用户显式使用 `/skill-name` 语法激活技能时，系统（`SkillActivationMiddleware`）会将 SKILL.md 的完整内容直接注入到当前对话的上下文中。

此时模型需要知道：
1. 这个技能是被显式请求的，优先级高于任何自动匹配
2. **不要再次调用 `read_file`** 读取已经被注入的 SKILL.md——内容已经在上下文中，重复读取会浪费 token 并引入重复内容

第三条规则 `"do not call read_file for that SKILL.md again"` 是一个微妙的重复防护：模型看到 `<skill_system>` 中的步骤 1 "immediately call `read_file`"，如果不加说明，它可能不知道在斜杠激活的情况下这一步应该跳过。

---

## 6.4 `<available_skills>` 的 XML 结构

每个技能的元数据以如下结构呈现：

```xml
<skill>
    <name>deep-research</name>
    <description>Use this skill instead of WebSearch for ANY question that benefits from comprehensive, multi-source research... [built-in]</description>
    <location>/mnt/skills/public/deep-research/SKILL.md</location>
</skill>
```

三个字段的设计逻辑：

**`<name>`**：斜杠激活时使用的标识符（`/deep-research`），也是分类识别的关键词

**`<description>`**：这是最关键的字段。它是决策信息，不是完整文档。模型依赖这段描述来判断"这个任务是否应该激活这个技能"。注意描述使用了 `"Use this skill instead of WebSearch for ANY question..."` 这种包含比较信息的写法——不只说"能做什么"，还说"相比其他选择，什么时候用这个"。

`[built-in]` 标签区分了内置技能和用户自定义技能（自定义技能显示 `[custom, editable]`），让模型了解技能的可变性。

**`<location>`**：完整的容器内虚拟路径，直接传给 `read_file` 工具无需任何转换。路径即操作的参数。

---

## 6.5 P-03：技能自演化段落原文

当 `skill_evolution_enabled=True` 时，`{skill_evolution_section}` 被替换为以下内容：

```
## Skill Self-Evolution
After completing a task, consider creating or updating a skill when:
- The task required 5+ tool calls to resolve
- You overcame non-obvious errors or pitfalls
- The user corrected your approach and the corrected version worked
- You discovered a non-trivial, recurring workflow
If you used a skill and encountered issues not covered by it, patch it immediately.
Prefer patch over edit. Before creating a new skill, confirm with the user first.
Skip simple one-off tasks.
```

### P-03 的设计哲学

这是 DeerFlow 的一个高级特性：Agent 可以创建和更新自己的技能文件（对于 `[custom, editable]` 技能）。P-03 定义了这个能力的触发条件。

**触发条件的设计**：

```
- The task required 5+ tool calls to resolve
- You overcame non-obvious errors or pitfalls
- The user corrected your approach and the corrected version worked
- You discovered a non-trivial, recurring workflow
```

这四个条件形成了一个**价值过滤器**：只有当任务展现出复用价值时（5+工具调用意味着非平凡工作流；用户纠正意味着模型原始方案有缺陷值得记录；"recurring workflow"意味着会重复出现）才创建技能。

`"Skip simple one-off tasks"` 是明确的反例，防止模型在每次完成任务后都创建一个新技能（导致技能库爆炸）。

**"Prefer patch over edit"**：patch（追加/局部修改）比 edit（重写）更安全，能保留现有内容的同时添加新发现。这条规则减少了因重写导致的信息丢失风险。

**"Before creating a new skill, confirm with the user first"**：技能是持久化的，创建新技能会影响未来所有会话。这是一个涉及持久状态的操作，需要用户审批——对应第 2 章介绍的 `risk_confirmation` 场景。

---

## 6.6 P-07：`<available-deferred-tools>` 段落原文

当 MCP 工具被配置为延迟加载时，`get_deferred_tools_prompt_section()` 生成以下内容：

```
<available-deferred-tools>
{deferred_tool_names}
</available-deferred-tools>
```

渲染示例（假设有 3 个延迟工具）：

```
<available-deferred-tools>
slack_send_message
notion_create_page
github_create_pr
</available-deferred-tools>
```

注意：这里只有工具名称，没有描述、没有参数 schema、没有使用指南。这是极简信息设计：**让模型知道工具存在，但不暴露完整 schema**。

### 延迟工具的生命周期

延迟工具机制有四个状态：

```
状态 1：工具名称在 <available-deferred-tools> 中
  → 模型知道工具存在，但无法调用（没有 schema）

状态 2：模型调用 tool_search("slack send")
  → P-25（tool_search 描述）定义了如何搜索

状态 3：tool_search 返回工具的完整 schema（OpenAI function calling 格式）
  → schema 进入消息历史

状态 4：DeferredToolFilterMiddleware 在下一轮将工具"提升"为可绑定
  → 模型现在可以直接调用 slack_send_message
```

这个生命周期确保了：工具信息在 **需要时** 才占用上下文，在 **不需要时** 只是一个名字（约 5 token）。

---

## 6.7 P-25：`tool_search` 工具描述（完整原文）

```
Fetches full schema definitions for deferred tools so they can be called.

Deferred tools appear by name in <available-deferred-tools> in the system
prompt. Until fetched, only the name is known. This tool matches a query
against the deferred tools and returns the matched tools complete schemas;
once returned, a tool becomes callable.

Query forms:
  - "select:Read,Edit" -- fetch these exact tools by name
  - "notebook jupyter" -- keyword search, up to max_results best matches
  - "+slack send" -- require "slack" in the name, rank by remaining terms
```

### P-25 的结构分析

**第一段**（解释工具的作用）：描述了"为什么需要这个工具"——因为延迟工具只有名字，需要通过这个工具获取完整 schema 才能调用。

**"Until fetched, only the name is known"**：这句话直接回答了模型可能会问的问题：为什么我在 `<available-deferred-tools>` 中只看到名字，没有参数信息？

**"once returned, a tool becomes callable"**：明确了调用 `tool_search` 的效果——工具从"知道存在但不能用"变为"可以直接调用"。这是后果描述，帮助模型理解调用这个工具的价值。

**三种查询格式**：

```
- "select:Read,Edit" -- fetch these exact tools by name
- "notebook jupyter" -- keyword search, up to max_results best matches
- "+slack send" -- require "slack" in the name, rank by remaining terms
```

这三种格式覆盖了不同的使用场景：
- `select:` 适合已知精确工具名的情况（最高效）
- 关键词搜索适合不确定名称、需要探索的情况
- `+required_term` 适合知道工具名中必须包含某个词、但需要进一步过滤的情况

注意这里的格式描述直接是可使用的示例（`"select:Read,Edit"` 不是元语言，就是实际的查询字符串），降低了使用门槛。

---

## 6.8 技能系统 vs 延迟工具：两种懒加载的对比

技能系统（P-06/P-07 的技能部分）和延迟工具系统（P-07/P-25）都实现了懒加载，但机制不同：

| 维度 | 技能懒加载 | 延迟工具懒加载 |
|------|-----------|-------------|
| **上下文中的存在形式** | `<skill>` 标签（名称 + 描述 + 路径）| 纯名称列表 |
| **加载触发方式** | 模型调用 `read_file` | 模型调用 `tool_search` |
| **加载后的形态** | ToolMessage 中的 SKILL.md 全文 | ToolMessage 中的 OpenAI function schema |
| **加载后是否可以"使用"** | 是（按 SKILL.md 指令执行）| 需要下一轮才能绑定 |
| **加载的 token 成本** | ~1500-3000 token/技能 | ~200-500 token/工具 |

技能加载是**内容加载**（把工作流指令放入上下文），工具加载是**接口加载**（把调用规范放入上下文）。技能告诉模型怎么做事，工具 schema 告诉模型怎么发起调用。

这两层懒加载合在一起，解决了 DeerFlow 面临的"能力越多，上下文越大"的规模问题：

- 22 个技能 × ~2000 token = **44,000 token（如果全量加载）**
- 22 个技能仅元数据 = **~600 token（懒加载）**
- 10 个 MCP 工具全量 schema = **~3000 token（如果全量绑定）**
- 10 个 MCP 工具仅名称 = **~50 token（延迟绑定）**

**总节省：约 47,000 token 的固定开销**，对应每次请求的成本几乎降到零。

---

## 6.9 `@lru_cache` 与技能的并发安全设计

第三章提到了 `@lru_cache(maxsize=32)` 的设计。完整来看，这里实际上有两套缓存机制协同工作：

**缓存 1**：`_get_cached_skills_prompt_section()`（LRU cache，maxsize=32）
- 缓存键：技能签名 tuple + 可用技能 tuple + 容器路径 + 自演化段落
- 适用场景：相同配置下的重复 Agent 构建（如多线程请求复用同一配置）
- 失效条件：任何技能参数变化

**缓存 2**：`_enabled_skills_cache`（全局变量 + threading.Lock）
- 用于缓存从磁盘加载的技能列表
- 通过后台线程异步刷新（`_refresh_enabled_skills_cache_worker`）
- 失效条件：技能配置变更（通过 `_invalidate_enabled_skills_cache()` 触发）

这两层缓存设计保证了：
1. **非阻塞请求路径**：技能列表从缓存读取，不阻塞事件循环
2. **高缓存命中率**：`lru_cache` 在系统 Prompt 构建层面缓存，避免重复字符串生成
3. **最终一致性**：技能配置变更时，后台刷新机制保证最终收敛到新状态

这是一个为高并发生产环境设计的缓存策略，不是简单的"加个缓存"。

---

## 6.10 举一反三：能力扩展系统的迁移

**建议 1**：在系统 Prompt 中只放元数据（名称 + 描述 + 路径），完整内容按需加载。阈值参考：单个内容 > 500 token 时值得懒加载。

**建议 2**：技能描述使用"Use this skill when..."（对比结构）而非只描述功能。"何时用这个而不是用那个"比"这个能做什么"更有助于触发正确的技能选择。

**建议 3**：对于工具 schema，考虑实现类似 `tool_search` 的发现机制，而不是将所有工具的 schema 绑定到模型。特别是当 MCP 工具数量超过 10 个时，完整绑定会显著增加上下文体积。

**建议 4**：在系统 Prompt 中明确说明"何时触发加载"的行为规范（如 P-06 的 Progressive Loading Pattern），而不是假设模型会"自然地"在需要时读取文件。

---

## 本章小结

- P-06（`<skill_system>`）通过"渐进式加载模式"实现技能的懒加载：系统 Prompt 中只放元数据（~25 token/技能），完整 SKILL.md 在匹配时按需加载
- P-03（技能自演化段落）通过四个价值过滤条件控制 Agent 何时应该创建/更新自定义技能，防止技能库膨胀
- P-07（延迟工具段落）对 MCP 工具实现极简存在声明：只有工具名称，完整 schema 通过 `tool_search` 按需获取
- P-25（`tool_search` 描述）通过三种查询格式覆盖不同的工具发现场景，支持精确选择、关键词搜索和约束式搜索
- 两套懒加载机制合计节省约 47,000 token 的固定上下文开销
