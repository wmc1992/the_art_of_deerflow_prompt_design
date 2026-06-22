# 第 1 章：DeerFlow 系统概览与 Prompt 地图

## 本章导读

在深入分析每一个 Prompt 之前，我们需要先建立一张"地图"——理解 DeerFlow 系统的整体架构，知道这些 Prompt 分别存在于系统的哪个位置，以及它们之间的关系是什么。

本章不涉及设计分析，只做事实梳理：系统是什么、Prompt 在哪里、它们如何被加载和使用。

---

## 1.1 DeerFlow 是什么

DeerFlow（Deep Exploration and Efficient Research Flow）是字节跳动开源的"超级 Agent 框架"。它的核心定位是：

> 一个以 LangGraph 为运行时、能够协调多个专门化子 Agent 并行工作的生产级 Agent 系统。

v2.0.0 是对 v1.x 架构的彻底重写，主要特征：

- **运行时**：LangGraph（StateGraph + checkpointer）
- **入口**：FastAPI 网关（port 8001），通过 Nginx 反向代理暴露统一端口（2026）
- **沙箱**：支持本地沙箱和 Docker 隔离沙箱（AioSandboxProvider）
- **工具**：内置工具（文件操作、网络搜索、澄清等）+ MCP 外接工具
- **技能**：22 个内置技能（位于 `skills/public/`），支持用户自定义技能
- **记忆**：LLM 驱动的事实提取与用户画像（SQLite 持久化）

### 关键架构边界

DeerFlow 有一个被 CI 测试强制执行的边界规则：

```
deerflow.* 包永远不导入 app.*
```

这个边界保证了 Agent Harness 层（`deerflow.*`）与 FastAPI 网关层（`app.*`）的解耦，Harness 可以被嵌入使用（不需要 HTTP），也可以通过 LangGraph SDK 远程访问。

本书分析的所有 Prompt 都位于 `deerflow.*` 层。

---

## 1.2 主 Agent 的执行路径

理解 Prompt 的位置，首先要理解主 Agent 的执行路径。

```
用户请求
    ↓
FastAPI 网关（app.*）
    ↓
LangGraph 运行时（StateGraph）
    ↓
┌─────────────────────────────────────────────────────┐
│            Lead Agent（deerflow.agents）              │
│                                                     │
│  [Middleware Chain]  ←── 18 个中间件按序执行          │
│       ↓                                             │
│  wrap_model_call()  ←── 包裹每次 LLM 调用            │
│       ↓                                             │
│  LLM Provider（Claude / GPT / Gemini）               │
│       ↓                                             │
│  工具执行（Tool Calling）                            │
│       ↓（循环）                                      │
│  最终响应                                            │
└─────────────────────────────────────────────────────┘
    ↓
（可选）子 Agent（通过 task 工具触发）
    ↓
StreamBridge → 前端 SSE 流
```

**Prompt 在哪里进入这个路径？**

- **系统 Prompt（P-01 系列）**：在 Agent 构建时（`make_lead_agent()`）装配完成，作为静态内容传入 LangGraph
- **动态上下文（含记忆、日期）**：由 `DynamicContextMiddleware` 在每次请求时以 `HumanMessage` 形式注入
- **Loop 检测消息（P-13 至 P-16）**：由 `LoopDetectionMiddleware` 在 `wrap_model_call()` 或 `after_model()` 时注入
- **工具 Prompt（P-21 至 P-25）**：工具绑定到模型时，docstring 被 LangChain 解析为工具 schema 的 `description` 字段

---

## 1.3 中间件链：Prompt 的运行时控制层

DeerFlow 的 18 个中间件构成了一条有序的流水线，每个中间件都有机会在 LLM 调用前后修改消息或状态。理解这条链对理解 Prompt 的注入时机至关重要。

```
ThreadData
    ↓ Uploads
    ↓ Sandbox
    ↓ DanglingToolCall
    ↓ LLMErrorHandling
    ↓ Guardrail
    ↓ SandboxAudit
    ↓ ToolErrorHandling
    ↓ SkillActivation        ← 斜杠技能激活时注入 SKILL.md 全文
    ↓ Summarization
    ↓ TodoList               ← is_plan_mode=True 时添加 P-08 系统 Prompt 段落
    ↓ TokenUsage
    ↓ Title                  ← 第一轮响应后触发 P-12 标题生成
    ↓ Memory                 ← 将记忆内容以 HumanMessage 注入
    ↓ ViewImage
    ↓ DeferredToolFilter     ← 管理延迟工具可见性（P-07）
    ↓ SubagentLimit
    ↓ LoopDetection          ← 注入 P-13 至 P-16
    ↓ Safety
    ↓ Clarification          ← 拦截 ask_clarification 工具调用（P-21）
```

这条链的顺序是固定的，不可动态调整。每个中间件的位置都有其逻辑原因：例如，`Clarification` 排在最后，确保它在其他所有处理（包括安全检查）之后才拦截澄清工具调用，保证澄清流程的完整性。

---

## 1.4 Prompt 全景地图

本书分析的 25 个 Python 源码 Prompt 可以按用途分为五类：

### 类别一：主系统 Prompt（P-01 至 P-07）

这是构成主 Agent "身份和能力"的核心 Prompt 集合。P-01 是主模板，P-02 至 P-07 是注入到 P-01 各占位符的子段落。

```
P-01: SYSTEM_PROMPT_TEMPLATE
├── {soul}          ← SOUL.md 内容（可选）
├── {subagent_thinking} + {subagent_reminder} ← P-02 相关（动态）
├── {self_update_section}  ← P-04（可选）
├── {skills_section}       ← P-06（动态，带缓存）
├── {deferred_tools_section} ← P-07（动态）
├── {subagent_section}     ← P-02（动态，可选）
└── {acp_section}          ← P-05（可选）

P-03: 技能自演化段落（注入到 P-06 的 {skill_evolution_section}）
```

### 类别二：Plan Mode 支撑（P-08 至 P-09）

仅在 `is_plan_mode=True` 时激活，为 Agent 提供任务列表管理能力。

```
P-08: TodoList 系统 Prompt（以 SystemMessage 形式追加）
P-09: write_todos 工具的 description（工具 schema）
```

### 类别三：记忆子系统（P-10 至 P-11）

独立于主 Agent 运行的记忆处理 Prompt，在每次对话结束后由专门的 LLM 调用执行。

```
P-10: MEMORY_UPDATE_PROMPT（分析对话，更新用户画像）
P-11: FACT_EXTRACTION_PROMPT（从单条消息提取事实）
```

### 类别四：安全与监控（P-12 至 P-16）

四个 Loop 检测消息构成分级干预机制，P-12 是会话标题生成 Prompt。

```
P-12: 标题生成（在第一轮响应后异步触发）
P-13: Loop 检测警告（相同工具调用重复 ≥ warn_threshold）
P-14: 工具频率警告（单个工具调用次数超限）
P-15: Loop 强制停止（总调用次数超 hard_stop_threshold）
P-16: 工具频率强制停止
```

### 类别五：子 Agent 与工具（P-17 至 P-25）

子 Agent 的系统 Prompt 和各工具的描述性文本，形成了工具使用行为的"说明书"。

```
子 Agent:
  P-17: general-purpose 子 Agent 系统 Prompt
  P-18: general-purpose 子 Agent 描述
  P-19: bash 子 Agent 系统 Prompt
  P-20: bash 子 Agent 描述

工具描述（docstring）:
  P-21: ask_clarification（澄清工具）
  P-22: task（子 Agent 启动工具）
  P-23: present_files（文件展示工具）
  P-24: view_image（图像查看工具）
  P-25: tool_search（延迟工具搜索）
```

---

## 1.5 22 个技能文件的组织方式

技能文件（S-01 至 S-22）存放在 `skills/public/` 目录下，每个技能是一个独立的目录，必须包含 `SKILL.md` 文件。

系统 Prompt 中只包含技能的**元数据**（名称、描述、路径），完整的 `SKILL.md` 内容在运行时由 Agent 按需通过 `read_file` 工具加载。这是一个**懒加载设计**：

```
系统 Prompt 中（P-06 渲染结果）：
<skill>
    <name>deep-research</name>
    <description>Use this skill instead of WebSearch for ANY question...</description>
    <location>/mnt/skills/public/deep-research/SKILL.md</location>
</skill>

↑ 只有 ~100 个 token

实际 SKILL.md（按需加载）：
# Deep Research Skill
## Overview
...（约 200 行完整内容）

↑ 约 1500-3000 个 token
```

这个设计的代价-收益非常清晰：**节省了每次对话中 22 个技能的全量 token 开销（约 30,000-50,000 token）**，换来的是在需要时多一次 `read_file` 工具调用（约 300-500 token）。

---

## 1.6 Token 预算概览

理解 Prompt 设计，离不开 token 成本的意识。以下是 DeerFlow 典型配置下的系统 Prompt token 分布估算：

| 组成部分 | 估算 token | 说明 |
|---------|----------|------|
| P-01 核心框架（无可选段落）| ~1,200 | 静态内容 |
| `<clarification_system>` | ~450 | P-01 的最大段落 |
| `<citations>` | ~320 | P-01 的次大段落 |
| `<critical_reminders>` | ~200 | |
| P-06 技能列表（22 个，仅元数据）| ~600 | 每个技能约 25 token |
| P-02 子 Agent 段落（n=3）| ~1,000 | 仅在 subagent_enabled 时 |
| P-04 自更新段落 | ~200 | 仅自定义 Agent |
| P-07 延迟工具列表（10 个）| ~80 | 仅在有 MCP 工具时 |
| P-08 TodoList 段落 | ~350 | 仅在 plan_mode 时 |
| **最小配置总计** | **~2,770** | 仅核心 + 技能列表 |
| **完整配置总计** | **~4,500–7,000** | 全部可选段落启用 |

动态注入内容（每次请求额外开销）：

| 组成部分 | 估算 token | 触发条件 |
|---------|----------|---------|
| 记忆注入（DynamicContextMiddleware）| 最多 2,000 | 有记忆时 |
| 当前日期注入 | ~20 | 每次请求 |
| 斜杠技能激活内容 | 500-3,000 | 用户显式触发 |

**关键洞察**：如果将所有 22 个 SKILL.md 的完整内容都放入系统 Prompt，会增加大约 30,000-50,000 token 的固定开销（相当于整本 Prompt 体积增加 10-20 倍）。懒加载设计将这部分成本几乎降到零。

---

## 1.7 Prompt 的生命周期

为了直观理解这些 Prompt 何时被激活，以下展示一次完整的对话请求生命周期：

```
① 用户发送消息："研究一下 LangGraph 的最新动态"
           ↓
② Middleware 链执行（前向）
   - Memory 中间件：读取用户记忆，注入 HumanMessage
   - DeferredToolFilter：过滤延迟工具列表
   - LoopDetection：检查 Loop 状态，重置计数器
           ↓
③ LLM 调用（第一次）
   - 系统 Prompt：P-01 全文（已在 Agent 构建时装配好）
   - 消息历史：[HumanMessage(记忆), HumanMessage(用户请求)]
           ↓
④ LLM 输出：判断需要激活 deep-research 技能
   → 工具调用：read_file("/mnt/skills/public/deep-research/SKILL.md")
           ↓
⑤ 工具执行：读取 SKILL.md 全文（S-08 的内容）
           ↓
⑥ LLM 调用（第二次）
   - 消息历史：[..., ToolResult(SKILL.md 内容)]
   - LLM 按照技能指令执行研究工作流
           ↓
⑦ 研究完成，LLM 生成最终响应
           ↓
⑧ Middleware 链执行（后向）
   - Title 中间件：触发 P-12 标题生成（异步）
   - Memory 中间件：触发 P-10 记忆更新（异步）
           ↓
⑨ 响应通过 StreamBridge 流向前端
```

这个生命周期图揭示了两个重要事实：

**事实一**：系统 Prompt（P-01 系列）在请求到达之前就已经装配完成，它不参与每次请求的动态调整。改变系统 Prompt 需要重新构建 Agent。

**事实二**：记忆（P-10/P-11 的产物）和标题生成（P-12）发生在响应之后，是异步操作，不阻塞主响应流程。这两个子系统有独立的 LLM 调用预算。

---

## 本章小结

- DeerFlow 的 Prompt 分为五类：主系统 Prompt（P-01-07）、Plan Mode 支撑（P-08-09）、记忆子系统（P-10-11）、安全监控（P-12-16）、子 Agent 与工具（P-17-25）
- 技能文件（S-01-22）通过懒加载设计节省了每次对话约 30,000-50,000 token 的开销
- 系统 Prompt 在 Agent 构建时静态装配，动态内容通过 Middleware 链注入为 HumanMessage
- 18 个中间件构成有序流水线，各自负责不同类型的 Prompt 注入或消息拦截

下一章，我们将从这张地图中提炼出 DeerFlow 在 Prompt 设计上最核心的原则。
