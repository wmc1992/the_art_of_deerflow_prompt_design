# 第 4 章：角色声明与身份锚定（`<role>` / `<soul>`）

## 本章导读

本章分析 P-01 系统 Prompt 的开篇部分：`<role>` 块、`{soul}` 注入机制，以及自定义 Agent 的 `<self_update>` 段落（P-04）。这三个组件共同回答了一个基础问题：**在 Prompt 设计中，如何建立 Agent 的身份，使其既稳定又可扩展？**

---

## 4.1 `<role>` 块：最短的身份声明

P-01 系统 Prompt 的正式内容从 `<role>` 标签开始：

```
<role>
You are {agent_name}, an open-source super agent.
</role>
```

> **中文附注**：
>
> ```
> <role>
> 你是 {agent_name}，一个开源超级 agent。
> </role>
> ```
>
> （`{agent_name}` 在构建时替换为实际名称，默认为 `DeerFlow 2.0`）

渲染后（默认配置 `agent_name="DeerFlow 2.0"`）：

```
<role>
You are DeerFlow 2.0, an open-source super agent.
</role>
```

> **中文附注**：
>
> ```
> <role>
> 你是 DeerFlow 2.0，一个开源超级 agent。
> </role>
> ```

这是一句 14 个 token 的声明。整个 Prompt 在此之后展开数百行规则和指令，但身份声明本身只有这一句。

### 为什么这么短？

这个选择背后有两层考量：

**考量一：过度定义身份会带来限制**

如果在 `<role>` 中写：
```
You are DeerFlow 2.0, an intelligent AI assistant specialized in research, 
code analysis, and document processing. You excel at...
```

这类详细的身份描述会把 Agent 锁定在特定的能力范围。但 DeerFlow 是个通用 Agent——用户可以用它做研究、写代码、分析文档、管理文件……任何具体的能力枚举都不完整，甚至会让模型在执行"不在列表上"的任务时产生犹豫。

**考量二：能力声明已经由后续段落承担**

`<role>` 不需要解释 Agent 能做什么，因为 `<skill_system>` 列出了技能，`<working_directory>` 定义了文件系统访问能力，`<subagent_system>`（P-02）定义了编排能力。这些段落已经完整描述了能力范围。

`<role>` 的唯一职责是建立**名字**和**基本角色定位**（"super agent"），而不是列举能力。

**"open-source super agent"的语义选择**

"super agent"是一个有意义的术语选择：它暗示了这个 Agent 不只是一个对话助手，它是一个能够编排其他 Agent、执行复杂多步任务的协调者。这个定位为后续的子 Agent 系统（P-02）和技能系统（P-06）奠定了语义基础。

"open-source"则建立了透明性期望：模型会以更开放、更乐于解释的方式对待用户的探究性问题。

---

## 4.2 `{soul}` 注入：可选的个性层

`<role>` 之后是 `{soul}` 占位符：

```python
# 在 SYSTEM_PROMPT_TEMPLATE 中
{soul}
```

在 `apply_prompt_template()` 中，这个占位符由以下函数填充：

```python
def get_agent_soul(agent_name: str | None) -> str:
    soul = load_agent_soul(agent_name)
    if soul:
        return f"<soul>\n{soul}\n</soul>\n" if soul else ""
    return ""
```

`load_agent_soul()` 从该 Agent 的 `SOUL.md` 文件加载内容。对于默认的 DeerFlow 2.0 Agent，这个文件不存在，`{soul}` 被替换为空字符串，`<soul>` 标签也不出现。

对于用户创建的自定义 Agent（通过 `/bootstrap` 命令创建），SOUL.md 可以包含个性描述。例如：

```markdown
I am Alex, a code review specialist with a focus on security and performance.
I prefer direct, technical communication and always explain the reasoning behind my suggestions.
I have strong opinions about code quality but I respect team conventions.
```

> **中文附注**：
>
> 我是 Alex，一名专注于安全与性能的代码审查专家。
> 我偏好直接、技术性的沟通，并始终解释我建议背后的理由。
> 我对代码质量有强烈的主见，但我尊重团队规范。

填充后，系统 Prompt 中会出现：

```
<soul>
I am Alex, a code review specialist with a focus on security and performance.
I prefer direct, technical communication and always explain the reasoning behind my suggestions.
I have strong opinions about code quality but I respect team conventions.
</soul>
```

> **中文附注**：
>
> ```
> <soul>
> 我是 Alex，一名专注于安全与性能的代码审查专家。
> 我偏好直接、技术性的沟通，并始终解释我建议背后的理由。
> 我对代码质量有强烈的主见，但我尊重团队规范。
> </soul>
> ```

### `<soul>` 的设计哲学

有几个设计细节值得注意：

**位置在 `<role>` 之后**：`<role>` 定义基础身份（名字 + 角色类型），`<soul>` 在此基础上叠加个性特质。这确保了个性不会覆盖基础身份。

**不约束格式**：SOUL.md 的内容没有格式要求，用户可以写一段描述性文字、一份人格特质列表，或者使用任何风格。这给了用户完整的个性定义自由。

**`<soul>` 标签的语义暗示**：标签名 `soul` 比 `personality` 或 `character` 更强调本质性而非表层特质。这引导模型将 SOUL.md 的内容理解为 Agent 的核心自我，而非可以随意忽略的行为提示。

**与 `<role>` 的层次关系**：如果发生冲突（例如 SOUL.md 说"I prefer minimalist answers"而后续段落要求详细引用），`<soul>` 的个性会被后续的规则段落约束——规则优先于个性。这个隐式的优先级来自 XML 标签的位置：后出现的、更具体的规则往往在注意力分配上有优势。

---

## 4.3 P-04：`<self_update>` 段落

P-04 是专门服务于自定义 Agent 的可选段落，只有当 `agent_name` 非空时才注入。完整原文：

```
<self_update>
You are running as the custom agent **{agent_name}** with a persisted SOUL.md and config.yaml.

When the user asks you to update your own description, personality, behaviour, skill set, tool groups, or default model,
you MUST persist the change with the `update_agent` tool. Do NOT use `bash`, `write_file`, or any sandbox tool to edit
SOUL.md or config.yaml — those write into a temporary sandbox/tool workspace and the changes will be lost on the next turn.

Rules:
- Always pass the FULL replacement text for `soul` (no patch semantics). Start from your current SOUL above and apply the user's edits.
- Only pass the fields that should change. Omit the others to preserve them.
- Never pass literal strings like `"null"`, `"none"`, or `"undefined"` for unchanged fields.
- Pass `skills=[]` to disable all skills, or omit `skills` to keep the existing whitelist.
- After `update_agent` returns successfully, tell the user the change is persisted and will take effect on the next turn.
</self_update>
```

> **中文附注**：
>
> ```
> <self_update>
> 你正在以自定义 agent **{agent_name}** 运行，具有持久化的 SOUL.md 和 config.yaml。
>
> 当用户要求你更新自身的描述、个性、行为、技能集、工具组或默认模型时，
> 你**必须**使用 `update_agent` 工具持久化变更。**不得**使用 `bash`、`write_file` 或任何 sandbox 工具编辑
> SOUL.md 或 config.yaml——这些写入的是临时 sandbox/工具工作区，更改在下一轮会丢失。
>
> 规则：
> - 始终传递 `soul` 的**完整替换文本**（非增量更新）。从当前 SOUL 出发，应用用户的编辑内容。
> - 只传递需要变更的字段，其余字段省略以保留原值。
> - 对未变更的字段，不得传递 `"null"`、`"none"` 或 `"undefined"` 等字面字符串。
> - 传递 `skills=[]` 以禁用所有技能；省略 `skills` 则保留现有白名单。
> - `update_agent` 成功返回后，告知用户变更已持久化，将在下一轮生效。
> </self_update>
> ```

### 逐行解析

**第一句**（重复身份确认）：
```
You are running as the custom agent **{agent_name}** with a persisted SOUL.md and config.yaml.
```

> **中文附注**：你正在以自定义 agent **{agent_name}** 运行，具有持久化的 SOUL.md 和 config.yaml。

这句话在系统 Prompt 中再次出现了 Agent 的名字，并明确了"你有一个可以持久化的 SOUL.md 和 config.yaml"。这是一个**上下文设定**，让后续的规则有了具体的落脚点。

**核心约束**：
```
you MUST persist the change with the `update_agent` tool. Do NOT use `bash`, `write_file`, or any sandbox tool to edit
SOUL.md or config.yaml — those write into a temporary sandbox/tool workspace and the changes will be lost on the next turn.
```

> **中文附注**：你**必须**使用 `update_agent` 工具持久化变更。**不得**使用 `bash`、`write_file` 或任何 sandbox 工具编辑 SOUL.md 或 config.yaml——这些写入的是临时 sandbox/工具工作区，**下一轮变更即会丢失**。

这是后果驱动约束（原则四）的典型例子。不只说"用 `update_agent`"，而是说明了为什么：使用 `bash` 或 `write_file` 写入的是临时沙箱工作空间，**下一轮就会消失**。"will be lost on the next turn"这个后果是具体的、可感知的，让模型理解为什么这个约束是必要的。

**"no patch semantics" 规则**：
```
Always pass the FULL replacement text for `soul` (no patch semantics). Start from your current SOUL above and apply the user's edits.
```

> **中文附注**：始终传递 `soul` 的**完整替换文本**（非增量更新）。从当前 SOUL 出发，应用用户的编辑内容。

"no patch semantics"是一个编程术语，但在 Prompt 中使用很有效——它精确地传达了"不是增量更新，而是全量替换"这个语义，比说"写出完整的新版 SOUL.md"更严格。

**"Only pass the fields that should change"**：这和"全量替换 soul"看起来矛盾，但实际上两者针对不同维度：
- `soul` 字段：需要全量传入（不能只传差异部分）
- `update_agent` 的其他参数（如 `model`、`skills`）：不需要更新的就不传，保持原值

这是一个通过精确的规则语言消除歧义的好例子。

### P-04 的激活条件

P-04 只在 `agent_name` 非空时出现，而 `agent_name` 非空意味着这是用户创建的自定义 Agent（不是默认的 DeerFlow 2.0）。默认 Agent 没有可以被 `update_agent` 修改的持久化配置，所以这段说明对它毫无意义——注入它只会浪费 token 并可能引起混淆。

这是条件注入（原则六，Token 预算意识）的一个实例：**只有当信息与当前上下文相关时才注入**。

---

## 4.4 三层身份结构

将这三个组件放在一起，可以看到 DeerFlow 的 Agent 身份构建是一个三层结构：

```
Layer 1: 基础身份（<role>）
  - 谁：名字
  - 什么：super agent
  - 来源：agent_name 参数（构建时确定）
  - 特点：极简，不可删除

Layer 2: 个性特质（<soul>）
  - 风格：交流方式、偏好、特点
  - 来源：SOUL.md（可选）
  - 特点：用户定义，可覆写

Layer 3: 自我维护能力（<self_update>）
  - 功能：如何持久化对自身的修改
  - 来源：固定模板，以 agent_name 填充
  - 触发：agent_name 非空时注入
```

这个三层结构有一个重要特性：**每一层只负责一件事，层与层之间的职责不重叠**。这使得系统具备了良好的可扩展性——添加新的自定义 Agent 只需要提供一个 SOUL.md 文件，不需要修改任何框架代码。

---

## 4.5 举一反三：身份层次设计的迁移

如果你在构建自己的 Agent 系统，这个三层模式可以直接迁移：

**建议 1**：将基础角色声明控制在 1-2 句，只包含名字和核心定位。避免在 `<role>` 里列举能力，能力描述交给后续专门的段落。

**建议 2**：提供一个类似 `SOUL.md` 的机制让终端用户定制 Agent 个性，但要设置明确的优先级：框架规则 > 个性描述。个性只影响沟通风格，不应该覆盖安全规则。

**建议 3**：使用 XML 标签包裹身份段落（如 `<role>`、`<soul>`），而非使用 Markdown 标题。这让模型更清楚这些内容的语义边界。

**建议 4**：如果你的系统有"Agent 可以修改自身配置"的需求，单独提供一个 `<self_update>` 类似的段落，并明确说明哪些操作会丢失持久化（后果驱动约束）。

---

## 本章小结

- `<role>` 只有一句话，用最小成本建立 Agent 名字和角色定位，把能力描述交给后续专门段落
- `{soul}` 机制通过 SOUL.md 让自定义 Agent 拥有可持久化的个性层，但不覆盖框架规则
- P-04（`<self_update>`）仅在自定义 Agent 上激活，用后果驱动的方式（"will be lost on the next turn"）防止用户使用错误的工具修改配置
- 三层身份结构（基础身份 + 个性 + 自维护）职责分离，支持可扩展的自定义 Agent 设计
