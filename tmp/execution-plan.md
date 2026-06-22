# 执行计划：《DeerFlow Prompt 设计艺术》

> 版本：v1.0 | 基于 deer-flow v2.0.0-rc1（tag 2026-06-19）

---

## 一、完整 Prompt 资产清单

本书需要收录的所有原始 prompt，按来源文件分类。

### 1.1 核心 Python 代码中的 Prompt

| 编号 | 名称 | 来源文件 | 类型 | 估算行数 |
|---|---|---|---|---|
| P-01 | 主 System Prompt 模板 | `agents/lead_agent/prompt.py::SYSTEM_PROMPT_TEMPLATE` | 静态模板字符串 | ~200 行 |
| P-02 | SubAgent 编排段（动态） | `agents/lead_agent/prompt.py::_build_subagent_section()` | 动态生成段 | ~130 行模板 |
| P-03 | Skill 自演化段 | `agents/lead_agent/prompt.py::_build_skill_evolution_section()` | 可选段 | ~10 行 |
| P-04 | Agent 自更新段 | `agents/lead_agent/prompt.py::_build_self_update_section()` | 可选段 | ~25 行 |
| P-05 | ACP Agent 段 | `agents/lead_agent/prompt.py::_build_acp_section()` | 可选段 | ~10 行 |
| P-06 | Skills 元数据段 | `agents/lead_agent/prompt.py::_get_cached_skills_prompt_section()` | 动态生成段 | ~20 行模板 |
| P-07 | 延迟工具段 | `tools/builtins/tool_search.py::get_deferred_tools_prompt_section()` | 动态生成段 | ~8 行 |
| P-08 | TodoList 系统提示 | `agents/lead_agent/agent.py::_create_todo_list_middleware()::system_prompt` | 静态字符串 | ~60 行 |
| P-09 | TodoList 工具描述 | `agents/lead_agent/agent.py::_create_todo_list_middleware()::tool_description` | 静态字符串 | ~70 行 |
| P-10 | 记忆更新提示 | `agents/memory/prompt.py::MEMORY_UPDATE_PROMPT` | 静态字符串 | ~138 行 |
| P-11 | 事实提取提示 | `agents/memory/prompt.py::FACT_EXTRACTION_PROMPT` | 静态字符串 | ~30 行 |
| P-12 | Title 生成提示 | `config/title_config.py::TitleConfig.prompt_template` | 默认值字符串 | 1 行 |
| P-13 | 循环检测警告消息 | `agents/middlewares/loop_detection_middleware.py::_WARNING_MSG` | 静态字符串 | 1 行 |
| P-14 | 工具频率警告消息 | `agents/middlewares/loop_detection_middleware.py::_TOOL_FREQ_WARNING_MSG` | 带格式化的字符串 | 1 行 |
| P-15 | 循环强制停止消息 | `agents/middlewares/loop_detection_middleware.py::_HARD_STOP_MSG` | 静态字符串 | 1 行 |
| P-16 | 工具频率强制停止消息 | `agents/middlewares/loop_detection_middleware.py::_TOOL_FREQ_HARD_STOP_MSG` | 带格式化的字符串 | 1 行 |
| P-17 | General-Purpose SubAgent 系统提示 | `subagents/builtins/general_purpose.py::GENERAL_PURPOSE_CONFIG.system_prompt` | 静态字符串 | ~35 行 |
| P-18 | General-Purpose SubAgent 描述 | `subagents/builtins/general_purpose.py::GENERAL_PURPOSE_CONFIG.description` | 静态字符串 | ~12 行 |
| P-19 | Bash SubAgent 系统提示 | `subagents/builtins/bash_agent.py::BASH_AGENT_CONFIG.system_prompt` | 静态字符串 | ~30 行 |
| P-20 | Bash SubAgent 描述 | `subagents/builtins/bash_agent.py::BASH_AGENT_CONFIG.description` | 静态字符串 | ~10 行 |
| P-21 | ask_clarification 工具描述 | `tools/builtins/clarification_tool.py` 函数 docstring | 工具 Schema | ~35 行 |
| P-22 | task 工具描述 | `tools/builtins/task_tool.py` 函数 docstring | 工具 Schema | ~30 行 |
| P-23 | present_files 工具描述 | `tools/builtins/present_file_tool.py` 函数 docstring | 工具 Schema | ~15 行 |
| P-24 | view_image 工具描述 | `tools/builtins/view_image_tool.py` 函数 docstring | 工具 Schema | ~15 行 |
| P-25 | tool_search 工具描述 | `tools/builtins/tool_search.py` make_tool_search 内 docstring | 工具 Schema | ~15 行 |

### 1.2 SKILL.md 文件（22 个）

按行数从大到小排列：

| 编号 | Skill 名称 | 行数 | 代表的设计模式 |
|---|---|---|---|
| S-01 | consulting-analysis | 631 | 两阶段 SOP + 分析框架模板 |
| S-02 | skill-creator | 485 | 元技能（用于创建 skill 的 skill） |
| S-03 | ppt-generation | 463 | 多风格模板 + 样式指南 |
| S-04 | code-documentation | 415 | 代码理解 + 结构化输出 |
| S-05 | newsletter-generation | 343 | 内容创作 + 分节生产 |
| S-06 | academic-paper-review | 289 | 学术评审标准化 |
| S-07 | data-analysis | 248 | 数据分析工作流 |
| S-08 | systematic-literature-review | 235 | 文献综述 SOP |
| S-09 | claude-to-deerflow | 217 | 迁移指南型 skill |
| S-10 | image-generation | 208 | 多步骤生成 + 工具协调 |
| S-11 | podcast-generation | 203 | 多角色内容生产 |
| S-12 | deep-research | 198 | 分阶段研究方法论（重点分析） |
| S-13 | github-deep-research | 166 | 垂直领域研究变体 |
| S-14 | video-generation | 151 | 媒体生成型 |
| S-15 | find-skills | 138 | 元查询（查找适合的 skill） |
| S-16 | vercel-deploy-claimable | 112 | 部署操作型 |
| S-17 | bootstrap | 94 | 对话式引导 + 多文件引用（重点分析） |
| S-18 | frontend-design | 92 | UI 生成型 |
| S-19 | music-generation | 76 | 音乐创作型 |
| S-20 | chart-visualization | 72 | 可视化生成型（重点分析） |
| S-21 | surprise-me | 53 | 开放式创意型 |
| S-22 | web-design-guidelines | 39 | 设计规范参考型（最轻量，重点分析） |

**正文深度分析的 5 个代表性 Skill**（覆盖不同设计模式）：
- S-12 `deep-research`：多阶段研究 SOP
- S-17 `bootstrap`：对话式引导 + 多文件引用结构
- S-20 `chart-visualization`：轻量工具协调型
- S-01 `consulting-analysis`：最复杂的两阶段分析框架
- S-22 `web-design-guidelines`：最轻量的参考型（设计对比用）

其余 17 个 skill 全部收录于附录，正文不展开分析。

---

## 二、书籍文件结构

所有文件存放于 `docs/book/` 目录下，适配 MkDocs 部署。

```
docs/book/
├── index.md                        # 书籍主页（封面 + 简介）
├── preface.md                      # 前言
├── how-to-read.md                  # 如何阅读本书
│
├── part1-foundation/               # 第一部分：基础理论
│   ├── index.md
│   ├── ch01-why-engineering.md     # 第 1 章：为什么 Prompt 是工程问题
│   ├── ch02-philosophy.md          # 第 2 章：DeerFlow 的 Prompt 设计哲学全景
│   └── ch03-five-principles.md     # 第 3 章：Prompt 有效性的五个底层原理
│
├── part2-core-system-prompt/       # 第二部分：核心系统 Prompt 深度解析
│   ├── index.md
│   ├── ch04-architecture.md        # 第 4 章：主 Agent System Prompt 整体架构
│   ├── ch05-clarification.md       # 第 5 章：行为约束：Clarification 系统
│   ├── ch06-subagent.md            # 第 6 章：并发编排：SubAgent 控制
│   └── ch07-static-dynamic.md     # 第 7 章：信息架构：静态与动态分离
│
├── part3-subsystem-prompts/        # 第三部分：子系统 Prompt 解析
│   ├── index.md
│   ├── ch08-memory.md              # 第 8 章：长期记忆的提取与注入
│   ├── ch09-skill-system.md        # 第 9 章：Skill 系统：可插拔专家知识
│   └── ch10-auxiliary.md          # 第 10 章：辅助 Prompt（Title、TodoList、Loop Detection、Tools）
│
├── part4-design-patterns/          # 第四部分：设计模式总结
│   ├── index.md
│   ├── ch11-pattern-catalog.md     # 第 11 章：DeerFlow Prompt 设计模式目录
│   └── ch12-prompt-vs-code.md      # 第 12 章：Prompt 与代码的职责分工
│
└── appendix/                       # 附录
    ├── appA-prompts-en.md          # 附录 A：所有原始 Prompt（英文版）
    ├── appB-prompts-zh.md          # 附录 B：所有原始 Prompt（中文翻译版）
    ├── appC-skills-en.md           # 附录 C：全部 SKILL.md 文件（英文原版）
    └── appD-skills-zh.md           # 附录 D：全部 SKILL.md 文件（中文翻译版）
```

---

## 三、各章内容规划

### 前言（preface.md）
- DeerFlow 项目简介（一段）
- 本书写作动机
- 本书的定位：面向算法专业人员的工程参考书
- 致谢

### 如何阅读本书（how-to-read.md）
- 章节导航图（不同背景读者的推荐阅读路径）
- 体例说明：每章 prompt 的展示格式（英文原版 → 中文注释）
- 约定：动态 prompt 的展示规则（模板 + 渲染示例）
- 如何查阅附录

---

### 第一部分：基础理论

**第 1 章：为什么 Prompt 是工程问题**
- Prompt 的本质：在推理时修改模型的条件概率分布
- 从"魔法咒语"到工程规范：行业认知的演变
- 可靠性是核心：一次有效 vs 每次有效的区别
- DeerFlow 的工程立场：prompt 是系统设计，不是调参

**第 2 章：DeerFlow 的 Prompt 设计哲学全景**
- 整体架构一图（系统 prompt 分层图）
- 五个设计原则预览（第 3 章展开）
- Prompt 与中间件的分工边界
- 可扩展性设计：如何让一套 prompt 支撑数十种定制 agent

**第 3 章：Prompt 有效性的五个底层原理**
- 原理一：场景化 > 规则化
- 原理二：后果描述 > 权威要求
- 原理三：代码 > 自然语言描述程序性步骤
- 原理四：静态/动态分离（缓存友好）
- 原理五：末尾强化（对抗 lost-in-the-middle）

---

### 第二部分：核心系统 Prompt 深度解析

**第 4 章：主 Agent System Prompt 整体架构**
- 完整展示 `SYSTEM_PROMPT_TEMPLATE`（P-01，英文原版 + 中文注释）
- 逐段解析每个 XML 标签存在的理由
- 各段的 token 预算估算
- 动态拼接机制：哪些段条件性出现、触发条件是什么

**第 5 章：行为约束——Clarification 系统**
- 完整展示 `<clarification_system>` 段（从 P-01 中抽取，英文原版 + 中文注释）
- 为什么 5 种具名场景比 1 条规则更有效
- ✅/❌ 对称结构的心理学原理
- 鲁棒性设计：非互斥分类的防御性意义
- 与 `ClarificationMiddleware` 代码层的配合

**第 6 章：并发编排——SubAgent 控制**
- 完整展示 `_build_subagent_section()` 渲染结果（P-02，英文原版 + 中文注释）
- 硬约束的三重强调写法解析
- 伪代码示例的使用时机
- "HARD LIMIT"语气的工程依据（对应中间件截断逻辑）

**第 7 章：信息架构——静态与动态分离**
- 完整展示动态注入机制（`DynamicContextMiddleware` 注入格式，英文原版 + 中文注释）
- Prefix Cache 的工作原理（简明解释）
- 设计决策树：什么内容进 system prompt，什么进 reminder
- Token 成本的量化估算

---

### 第三部分：子系统 Prompt 解析

**第 8 章：长期记忆的提取与注入**
- 完整展示 `MEMORY_UPDATE_PROMPT`（P-10，英文原版 + 中文注释）
- 完整展示 `FACT_EXTRACTION_PROMPT`（P-11，英文原版 + 中文注释）
- JSON Schema 输出格式的设计逻辑
- 置信度分层系统（0.9+ / 0.7-0.8 / 0.5-0.6）的工程含义
- `correction` 类别：捕捉模型犯错信号的设计
- Memory 注入的 token 预算控制（2000 token 上限的来由）
- 注入格式展示（`format_memory_for_injection` 输出样例）

**第 9 章：Skill 系统——可插拔的专家知识**
- Skill 的本质：可按需加载的专家 SOP
- System prompt 中的元数据格式（P-06，英文原版 + 中文注释）
- `<skill_system>` 段完整展示及解析（从 P-01 抽取）
- 深度分析 5 个代表性 Skill（各展示完整英文原版 + 中文注释）：
  - S-22 `web-design-guidelines`：最轻量的参考型（对照基准）
  - S-20 `chart-visualization`：轻量工具协调型
  - S-12 `deep-research`：分阶段研究 SOP（代表性最强）
  - S-17 `bootstrap`：对话式引导 + 多文件引用结构
  - S-01 `consulting-analysis`：最复杂的两阶段分析框架
- 其余 17 个 skill 的设计模式简评（表格对比，引用附录）
- Skill frontmatter `description` 字段的写法：触发条件的语言学分析

**第 10 章：辅助 Prompt**
- TodoList/Plan Mode 系统提示（P-08，英文原版 + 中文注释）
- TodoList 工具描述（P-09，英文原版 + 中文注释）
- Title 生成提示（P-12，英文原版 + 中文注释）
- 循环检测消息（P-13 至 P-16，英文原版 + 中文注释）
- General-Purpose SubAgent 提示（P-17/P-18，英文原版 + 中文注释）
- Bash SubAgent 提示（P-19/P-20，英文原版 + 中文注释）
- 工具 Schema 作为 Prompt（P-21 至 P-25，英文原版 + 中文注释）
- 延迟工具发现段（P-07，英文原版 + 中文注释）

---

### 第四部分：设计模式总结

**第 11 章：DeerFlow Prompt 设计模式目录**

每个模式的格式：名称 → 定义 → 适用场景 → 代码中的实例引用

收录以下模式（待写作时按实际情况扩充）：
1. 具名场景模式（Named Scenario Pattern）
2. 后果约束模式（Consequence Constraint Pattern）
3. 伪代码示范模式（Pseudocode Example Pattern）
4. 静态/动态分离模式（Static-Dynamic Split Pattern）
5. 末尾强化模式（Tail Reinforcement Pattern）
6. 非互斥分类模式（Non-exclusive Taxonomy Pattern）
7. 元数据懒加载模式（Metadata Lazy-load Pattern）
8. 置信度分层模式（Confidence Tiering Pattern）
9. XML 锚点模式（XML Anchor Pattern）

**第 12 章：Prompt 与代码的职责分工**
- Prompt 的边界：能保证什么，不能保证什么
- 中间件作为"兜底层"：从 DeerFlow 18 个中间件中提炼分工原则
- 决策框架：给定一个行为约束，该放进 prompt 还是代码

---

### 附录规划

**附录 A（appA-prompts-en.md）**：所有 P-01 至 P-25 的英文原版，按编号排列，每条注明来源文件和行号

**附录 B（appB-prompts-zh.md）**：附录 A 的逐条中文翻译

**附录 C（appC-skills-en.md）**：22 个 SKILL.md 文件的完整英文原版，按 S-01 至 S-22 排列

**附录 D（appD-skills-zh.md）**：附录 C 的逐条中文翻译

---

## 四、动态 Prompt 的展示约定

以下 prompt 在运行时根据配置动态生成，展示方式如下：

| Prompt | 展示方式 |
|---|---|
| P-02 SubAgent 段 | 展示原始模板（含 `{n}` 等占位符）+ 默认配置渲染示例（n=3） |
| P-06 Skills 元数据段 | 展示生成函数逻辑 + 典型渲染示例（3 个 skill） |
| P-07 延迟工具段 | 展示生成函数 + 渲染示例 |
| P-03/P-04/P-05 可选段 | 展示原始模板 + 说明触发条件 |

---

## 五、写作顺序

按以下顺序逐章完成，每章完成后立即存为对应 md 文件：

```
阶段 1（基础框架）：
  附录 A → 附录 B → 附录 C → 附录 D
  （先把所有原始内容整理好，作为后续章节写作的参考基础）

阶段 2（正文）：
  前言 → 如何阅读
  → 第 1 章 → 第 2 章 → 第 3 章
  → 第 4 章 → 第 5 章 → 第 6 章 → 第 7 章
  → 第 8 章 → 第 9 章 → 第 10 章
  → 第 11 章 → 第 12 章
```

原因：先写附录可以确保所有原始 prompt 已经过校验，正文写作时可以直接引用，不需要再次核对。

---

## 六、质量保证

### 英文 Prompt 准确性校验（写完后执行）

对每条 P-01 至 P-25 执行以下校验：

1. 将附录 A 中的文本与源代码文件逐字对比
2. 对于动态生成段，校验模板变量名与代码一致
3. 记录每条 prompt 的来源文件路径和起止行号，作为附录 A 的元数据

校验工具：直接用 `grep` 在源文件中搜索附录中的关键句子，确认存在且无修改。

### 翻译一致性
- 技术术语保留英文（如 middleware、prefix cache、token、LangGraph）
- 首次出现的术语括号注英文原文
- 命令式语气的 prompt 指令，翻译时保持祈使句式

---

## 七、预估工作量

| 部分 | 文件数 | 预估字数（中文） |
|---|---|---|
| 附录 A+B（Prompt 英文+翻译） | 2 | ~15,000 |
| 附录 C+D（Skills 英文+翻译） | 2 | ~60,000 |
| 前言 + 如何读 | 2 | ~3,000 |
| 第一部分（3 章） | 3 | ~15,000 |
| 第二部分（4 章） | 4 | ~30,000 |
| 第三部分（3 章） | 3 | ~35,000 |
| 第四部分（2 章） | 2 | ~15,000 |
| **合计** | **18** | **~173,000** |

---

## 八、尚待确认的事项

以下问题在写作过程中如有歧义，按此处约定处理：

1. **工具 docstring 是否算"Prompt"**：是，工具描述直接进入模型的 context，功能上等同于 prompt，收录于附录 A 并分析
2. **循环检测的 4 条消息**（P-13 至 P-16）：虽然是单行字符串，仍完整收录，分析其"在 HumanMessage 里注入"这一特殊机制
3. **SubAgent 的 description 字段**（P-18/P-20）：这是 LangGraph tool schema 的一部分，模型通过它决定何时调用什么 subagent，功能与 prompt 等价，收录并分析
4. **Bootstrap Skill 引用的子文件**（`references/conversation-guide.md`、`templates/SOUL.template.md`）：作为 S-17 的附属文件，在第 9 章分析时展示，附录 C 中也完整收录
