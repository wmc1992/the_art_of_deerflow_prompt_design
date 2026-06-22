# 第 3 章：静态与动态 Prompt 的设计哲学

## 本章导读

第 2 章的原则一（前缀缓存优先）点出了 DeerFlow 最重要的架构决策之一：系统 Prompt 静态，动态内容走 HumanMessage。本章深入展开这个决策，分析它在代码层面如何实现，以及这种静/动分离如何贯穿整个 Prompt 体系的设计。

---

## 3.1 什么叫"静态 Prompt"

在 DeerFlow 的语境中，"静态 Prompt"指的是：**从 Agent 构建（`make_lead_agent()` 调用）到 Agent 销毁之间，内容不发生任何变化的 Prompt 段落**。

系统 Prompt（P-01 系列）就是静态的。以下是 `apply_prompt_template()` 函数的简化逻辑：

```python
def apply_prompt_template(
    agent_name: str,
    soul: str = "",
    skills_section: str = "",       # P-06，构建时计算
    subagent_section: str = "",      # P-02，构建时生成
    deferred_tools_section: str = "", # P-07，构建时生成
    # ... 其他可选段落
) -> str:
    return SYSTEM_PROMPT_TEMPLATE.format(
        agent_name=agent_name,
        soul=soul,
        skills_section=skills_section,
        subagent_section=subagent_section,
        deferred_tools_section=deferred_tools_section,
        # ...
    )
```

这个函数在 `make_lead_agent()` 中被调用一次，结果字符串被固定下来，后续不再修改。

**关键理解**：P-02 虽然是"动态生成"的（代码中有参数决定它的内容），但它是在 **Agent 构建时** 动态生成的，生成后同样是静态的。"动态生成"描述的是生成过程的灵活性，而非运行时的可变性。

---

## 3.2 什么叫"动态 Prompt"

"动态 Prompt"指的是：**每次请求（甚至每次 LLM 调用）时内容都可能不同的 Prompt 段落**。

DeerFlow 中的动态 Prompt 分为两种：

### 种类 A：请求级动态（Per-Request Dynamic）

每次用户发出请求时生成，在整个请求处理过程中保持不变。

**代表**：`DynamicContextMiddleware` 注入的内容

```python
# 每次请求时执行
class DynamicContextMiddleware:
    async def before_model(self, state):
        parts = []
        
        # 当前日期
        parts.append(f"Today's date is {datetime.now().strftime('%Y-%m-%d')}.")
        
        # 用户记忆（从 SQLite 读取）
        memory = await self.memory_store.get_memory(user_id)
        if memory:
            parts.append(memory.to_prompt_string())
        
        # 以 HumanMessage 形式注入，不修改系统 Prompt
        reminder = "\n".join(parts)
        return [HumanMessage(content=f"<system-reminder>\n{reminder}\n</system-reminder>")]
```

**注入效果**（在消息链中的位置）：

```
SystemMessage: [静态系统 Prompt，P-01 全文]
HumanMessage: [<system-reminder>
               Today's date is 2026-06-22.
               # userEmail
               The user's email address is ...
               # Memory
               User is a software engineer focused on AI...
               </system-reminder>]
HumanMessage: [用户实际消息]
```

### 种类 B：调用级动态（Per-Call Dynamic）

在每次 LLM 调用时可能变化，由中间件在 `before_model()` 钩子中实时生成。

**代表 1**：`LoopDetectionMiddleware` 的警告消息（P-13 至 P-16）

```python
# 在 wrap_model_call 中检查并注入
def wrap_model_call(self, messages):
    if self._detect_loop(messages):
        # 在当前调用的消息链末尾追加 HumanMessage
        return messages + [HumanMessage(content=_WARNING_MSG)]
    return messages
```

这类 Prompt 在绝大多数调用中不会出现，只在检测到异常时才注入。

**代表 2**：`SkillActivation` 中间件注入的斜杠技能内容

```python
# 当用户使用 /deep-research 时
if skill_name := extract_slash_command(user_message):
    skill_content = read_skill_file(skill_name)
    # 将技能全文注入到当前调用的上下文
    messages = inject_skill_content(messages, skill_content)
```

---

## 3.3 为什么不把动态内容放进系统 Prompt

这个问题值得深入探讨，因为在实践中有很多团队会选择"在系统 Prompt 里塞更多信息"，认为这样模型会更有"上下文意识"。DeerFlow 的设计选择了相反的方向，背后有几层考量：

### 考量一：前缀缓存命中率（已在第 2 章分析）

每次系统 Prompt 变化 = 缓存失效 = 全量重新计算。在高并发场景下，保持系统 Prompt 静态是最直接的成本控制手段。

### 考量二：`<system-reminder>` 标签的语义设计

注意动态内容使用了 `<system-reminder>` 标签，而非 `<system>` 或其他标签。这个标签的选择不是随意的：

- `<system>` 标签暗示核心行为规范，模型会给予最高优先级和最严格的遵守
- `<system-reminder>` 标签暗示"辅助提醒性信息"，模型会把它理解为对话上下文的补充，而非行为规范

这种语义差异很重要：日期信息和记忆信息是"提醒模型知道什么"，而不是"命令模型做什么"。错误地使用高权限标签会导致模型赋予辅助信息过高的决策优先级。

### 考量三：关注点分离（Separation of Concerns）

```
系统 Prompt（静态）
  ↕ 关注：角色 / 规则 / 能力范围
  ↕ 性质：通用、长期有效

动态注入（HumanMessage）
  ↕ 关注：当前会话的具体上下文
  ↕ 性质：特定于本次请求
```

把这两种关注点混合在一个地方（系统 Prompt），会导致：
- 测试难度增加：每次测试需要模拟运行时状态
- 版本管理复杂：系统 Prompt 的"真实内容"依赖于运行时
- 调试困难：行为异常时不清楚是"规则问题"还是"上下文问题"

分离后，你可以独立测试系统 Prompt 的规则逻辑（使用固定的测试记忆），也可以独立验证动态内容的注入是否正确（不需要运行完整的 Agent）。

---

## 3.4 动态 Prompt 的三种注入方式

DeerFlow 使用了三种不同的动态内容注入方式，每种方式有不同的语义和位置意义：

### 注入方式 1：HumanMessage（前置）

**使用场景**：DynamicContextMiddleware、SkillActivation（显式激活）

**消息链位置**：
```
SystemMessage: [静态系统 Prompt]
HumanMessage: [<system-reminder>动态上下文</system-reminder>]  ← 前置
HumanMessage: [用户实际消息]
AIMessage: [Assistant 响应]
```

**特点**：动态内容出现在用户消息之前，相当于"先告诉模型背景，再看用户说什么"。

### 注入方式 2：HumanMessage（追加到末尾）

**使用场景**：LoopDetectionMiddleware（P-13 至 P-16）

**消息链位置**：
```
SystemMessage: [静态系统 Prompt]
HumanMessage: [用户消息]
AIMessage: [工具调用]
ToolMessage: [工具结果]
AIMessage: [工具调用（重复）]
ToolMessage: [工具结果]
HumanMessage: [LOOP DETECTED] 警告消息  ← 追加到末尾，紧邻下次 LLM 调用
```

**特点**：放在消息链末尾，利用近因效应，确保警告在最近的 LLM 调用中最为显眼。

**特别设计细节**：为什么循环检测消息要用 HumanMessage 而不是 SystemMessage？

如果你把警告注入为一条额外的 SystemMessage，LangGraph 的消息链会变成：
```
[SystemMessage][HumanMessage][AIMessage(tool_calls)][ToolMessage][SystemMessage(警告)]
```
这会破坏 AIMessage + ToolMessage 的配对关系，导致某些 LLM provider 的 API 报错（因为它们要求 ToolMessage 之后必须是 HumanMessage 或 AIMessage，不能是 SystemMessage）。

用 HumanMessage 注入警告则不破坏消息配对约束：
```
[SystemMessage][HumanMessage][AIMessage(tool_calls)][ToolMessage][HumanMessage(警告)]
```
这是一个"干预不破坏协议"的典型设计。

### 注入方式 3：AIMessage 内容修改（后置截断）

**使用场景**：LoopDetectionMiddleware 的强制停止（P-15、P-16）

**操作**：不是添加新消息，而是修改已生成的 AIMessage：
1. 将强制停止消息追加到 AIMessage.content 末尾
2. 清空 AIMessage.tool_calls（这样模型不会执行被截断的工具调用）

```python
# 伪代码
def _force_stop(self, ai_message):
    ai_message.content += _HARD_STOP_MSG
    ai_message.tool_calls = []  # 清除工具调用，阻止执行
    return ai_message
```

这种方式的语义是：**模型已经决定了下一步行动（tool_calls），但系统强制取消了这个决定**。用 AIMessage 修改而非新消息注入，保持了消息序列的语义完整性（这仍然是"模型的输出"，只是被修改过了）。

---

## 3.5 动态生成的静态段落：P-02 的特殊地位

P-02（子 Agent 编排段落）有一个值得专门讨论的特殊性：它是**动态生成的，但在运行时是静态的**。

回顾 P-02 的模板结构（详见附录 A），它包含多个 `{n}` 占位符（最大并发 Agent 数）和 `{available_subagents}` 占位符（可用子 Agent 列表）。这些占位符在 `_build_subagent_section()` 函数中，在 **Agent 构建时** 被实际值替换：

```python
def _build_subagent_section(
    max_concurrent: int,          # 从 config 读取，例如 3
    available_subagents: list,    # 注册的子 Agent 列表
    direct_tool_examples: str,    # 直接工具示例
    direct_execution_example: str,# 直接执行代码示例
) -> str:
    template = SUBAGENT_SECTION_TEMPLATE
    return template.format(
        n=max_concurrent,
        available_subagents=format_subagents(available_subagents),
        direct_tool_examples=direct_tool_examples,
        direct_execution_example=direct_execution_example,
    )
```

生成后的字符串是：

```
<subagent_system>
**🚀 SUBAGENT MODE ACTIVE - DECOMPOSE, DELEGATE, SYNTHESIZE**
...
**⛔ HARD CONCURRENCY LIMIT: MAXIMUM 3 `task` CALLS PER RESPONSE. THIS IS NOT OPTIONAL.**
...
**Available Subagents:**
- **general-purpose**: A capable agent for complex, multi-step tasks...
- **bash**: Command execution specialist...
</subagent_system>
```

> **中文附注**：
>
> `<subagent_system>`
> **🚀 子 AGENT 模式已激活——分解、委派、综合**
> ……
> **⛔ 硬性并发限制：每次回复最多 3 个 `task` 调用。这不是可选项。**
> ……
> **可用子 Agent：**
> - **general-purpose**：适用于需要复杂多步任务的能力 agent……
> - **bash**：命令执行专家……
> `</subagent_system>`

这个段落在 Agent 的整个生命周期中内容固定。改变并发限制（`n`）或增减子 Agent 类型，需要重新构建 Agent。

**为什么这样设计而不是运行时动态注入？**

P-02 的内容约有 1000 token，如果在每次请求时动态生成并注入，会破坏前缀缓存（原则一）。但它的内容本身不需要每次改变——子 Agent 的可用列表和并发限制在配置时就确定了，不是会话状态的一部分。

这是"在构建时做尽可能多的工作，在运行时做尽可能少的工作"这一工程原则的体现。

---

## 3.6 P-06 的 LRU 缓存：静动边界的精细化控制

技能元数据段落（P-06）使用了 Python 的 `@lru_cache(maxsize=1)` 装饰器：

```python
@lru_cache(maxsize=1)
def _get_cached_skills_prompt_section(
    skills_tuple: tuple,          # 技能列表的不可变表示
    container_base_path: str,
    skill_evolution_enabled: bool,
) -> str:
    # 生成技能列表 XML
    ...
```

这个缓存的意义：

1. **技能列表通常是稳定的**：用户不会每秒更改技能配置，因此同样参数下的调用结果可以缓存复用
2. **技能列表偶尔会变**：用户启用/禁用技能时缓存会失效（因为 `skills_tuple` 参数变化），重新生成
3. **参数使用 tuple**：Python 的 `lru_cache` 要求参数可哈希，因此技能列表转为不可变的 tuple

这是一个精细化的缓存控制：**在"完全静态"（Agent 构建时固定）和"每次请求动态"之间找到了第三个位置**——"参数不变时缓存，参数变化时重算"。

---

## 3.7 静/动分离的设计权衡

用一张表来总结 DeerFlow 静/动分离决策的代价与收益：

| 维度 | 静态系统 Prompt | 动态 HumanMessage 注入 |
|------|---------------|----------------------|
| **前缀缓存** | ✅ 完全命中 | ❌ 不参与（每次不同）|
| **测试难度** | ✅ 固定内容，易于测试 | 需要模拟注入内容 |
| **版本控制** | ✅ 内容可追踪 | 运行时内容难以追溯 |
| **实时性** | ❌ 无法反映实时状态 | ✅ 每次请求都是最新 |
| **个性化** | ❌ 所有用户同一 Prompt | ✅ 可以注入用户特定内容 |
| **Token 成本** | 固定（无额外开销）| 每次请求额外消耗 |

DeerFlow 的选择是：**把需要版本控制、需要缓存的"能力规范"放在静态系统 Prompt；把需要实时更新的"上下文信息"放在动态 HumanMessage**。

这个边界在实践中是清晰的：
- 日期、用户记忆、会话状态 → HumanMessage（动态）
- 角色定义、行为规则、工具使用规范、技能能力 → 系统 Prompt（静态）

---

## 3.8 对技能文件的启示：SKILL.md 的静/动设计

虽然 SKILL.md 的内容是静态文件，但技能系统本身体现了一种更大层次的静/动分离：

```
系统 Prompt 中（静态，永远在上下文）：
  技能元数据（名称 + 描述 + 路径）= ~25 token/技能

对话中（动态，按需加载）：
  技能完整内容（工作流 + 指令 + 示例）= ~1500-3000 token/技能
```

这意味着 SKILL.md 实际上也是一种"动态注入"——只是触发时机是"技能匹配"，注入方式是工具调用结果（`read_file` 返回的 ToolMessage）。

如果你在设计自己的技能系统，这个分层模式提供了一个很好的参考：**描述性元数据放在系统 Prompt，执行性指令按需加载**。

---

## 本章小结

- **静态系统 Prompt**：Agent 构建时装配完成，整个生命周期不变；保证前缀缓存命中，支持版本控制和测试
- **动态 HumanMessage**：每次请求/每次调用时实时生成；包含日期、记忆、Loop 检测警告等上下文敏感内容
- **三种注入方式**：前置 HumanMessage（上下文背景）、末尾追加 HumanMessage（干预信号）、AIMessage 修改（强制截断）
- **P-02 的特殊地位**：动态生成但运行时静态，体现了"构建时做尽可能多的工作"原则
- **P-06 的 LRU 缓存**：在完全静态和每次动态之间的第三条路——参数不变时缓存，参数变化时重算

第二部分（第 4-7 章）将带着这三章建立的框架，深入分析 P-01 的每个段落。
