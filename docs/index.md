# DeerFlow Prompt 设计艺术

> **基于 DeerFlow v2.0.0-rc1（2026-06-19）的深度解析**  
> 面向算法工程师的 Prompt 工程实践

---

!!! warning "关于本书的生成方式"

    本书**完全由 [Claude Code](https://claude.ai/code) + Claude Sonnet 4.6 自动写成**，未经人工逐字校验。

    具体而言：所有章节内容均由 AI 在阅读 DeerFlow 源码后自动生成；英文 Prompt 原文已与源文件逐字核对，但分析文字、中文翻译及设计解读均为模型输出，可能存在误读、遗漏或过度推断。

    阅读建议：将本书视为"有经验的读者对源码的第一遍注解"，而非权威文档。遇到存疑之处，请以 DeerFlow 源代码为准。

---

## 关于本书

本书以字节跳动开源的超级 Agent 框架 [DeerFlow](https://github.com/bytedance/deer-flow) 为研究对象，系统分析其全部 Prompt 设计——25 个核心 Python 源码 Prompt（P-01 至 P-25）与 22 个内置技能文件（S-01 至 S-22）——提炼出可迁移的 Prompt 工程原则。

**读完本书，你将能够：**

- 理解工业级 Agent 系统中 Prompt 的完整设计体系
- 掌握前缀缓存优化、静动分离、XML 锚点等核心设计模式
- 具备评估和设计复杂多 Agent 系统 Prompt 的能力
- 直接获取所有原始 Prompt 英文原文及中文译注（附录 A-D）

---

## 目录

### 前言

- [序言：为什么要读一个框架的 Prompt](preface.md)
- [如何使用本书](how-to-read.md)

---

### 第一部分：基础认知

- [第 1 章：DeerFlow 系统概览与 Prompt 地图](part1-foundation/ch01-overview.md)
- [第 2 章：Prompt 工程核心原则](part1-foundation/ch02-core-principles.md)
- [第 3 章：静态与动态 Prompt 的设计哲学](part1-foundation/ch03-static-dynamic.md)

---

### 第二部分：主系统 Prompt 深度解析

- [第 4 章：角色声明与身份锚定（`<role>` / `<soul>`）](part2-core-system-prompt/ch04-role-soul.md)
- [第 5 章：思维风格与澄清系统（`<thinking_style>` / `<clarification_system>`）](part2-core-system-prompt/ch05-thinking-clarification.md)
- [第 6 章：技能系统与延迟工具（`<skill_system>` / `<available-deferred-tools>`）](part2-core-system-prompt/ch06-skills-deferred.md)
- [第 7 章：约束体系：工作目录、回复风格、引用与关键提醒](part2-core-system-prompt/ch07-constraints.md)

---

### 第三部分：子系统 Prompt 解析

- [第 8 章：子 Agent 编排——P-02 的并行设计哲学](part3-subsystem-prompts/ch08-subagent.md)
- [第 9 章：记忆系统——P-10 的知识提炼机制](part3-subsystem-prompts/ch09-memory.md)
- [第 10 章：安全防线——Loop 检测、TodoList 与标题生成](part3-subsystem-prompts/ch10-safety-tools.md)

---

### 第四部分：设计模式与综合提升

- [第 11 章：七大可复用设计模式](part4-design-patterns/ch11-patterns.md)
- [第 12 章：从 DeerFlow 到你的系统——实战迁移指南](part4-design-patterns/ch12-migration.md)

---

### 附录

- [附录 A：全部 Prompt 英文原文（P-01 至 P-25）](appendix/appA-prompts-en.md)
- [附录 B：全部 Prompt 中文译注（P-01 至 P-25）](appendix/appB-prompts-zh.md)
- [附录 C：全部技能文件英文原文（S-01 至 S-22）](appendix/appC-skills-en.md)
- [附录 D：全部技能文件中文译注（S-01 至 S-22）](appendix/appD-skills-zh.md)

---

## Prompt 索引

| 编号 | 名称 | 来源文件 | 类型 |
|------|------|---------|------|
| P-01 | 主 Agent 系统 Prompt 模板 | `lead_agent/prompt.py` | 静态模板 |
| P-02 | 子 Agent 编排段落 | `lead_agent/prompt.py` | 动态生成 |
| P-03 | 技能自演化段落 | `lead_agent/prompt.py` | 可选段落 |
| P-04 | Agent 自更新段落 | `lead_agent/prompt.py` | 可选段落 |
| P-05 | ACP Agent 段落 | `lead_agent/prompt.py` | 可选段落 |
| P-06 | 技能元数据段落 | `lead_agent/prompt.py` | 动态生成（带缓存）|
| P-07 | 延迟工具段落 | `tool_search.py` | 动态生成 |
| P-08 | TodoList 系统 Prompt | `lead_agent/agent.py` | 静态字符串 |
| P-09 | TodoList 工具描述 | `lead_agent/agent.py` | 静态字符串 |
| P-10 | 记忆更新 Prompt | `memory/prompt.py` | 静态模板 |
| P-11 | 事实提取 Prompt | `memory/prompt.py` | 静态模板 |
| P-12 | 标题生成 Prompt | `config/title_config.py` | 静态模板（可配置）|
| P-13 | Loop 检测警告消息 | `loop_detection_middleware.py` | 静态字符串 |
| P-14 | 工具频率警告消息 | `loop_detection_middleware.py` | 格式字符串 |
| P-15 | Loop 强制停止消息 | `loop_detection_middleware.py` | 静态字符串 |
| P-16 | 工具频率强制停止消息 | `loop_detection_middleware.py` | 格式字符串 |
| P-17 | 通用子 Agent 系统 Prompt | `general_purpose.py` | 静态字符串 |
| P-18 | 通用子 Agent 描述 | `general_purpose.py` | 静态字符串 |
| P-19 | Bash 子 Agent 系统 Prompt | `bash_agent.py` | 静态字符串 |
| P-20 | Bash 子 Agent 描述 | `bash_agent.py` | 静态字符串 |
| P-21 | `ask_clarification` 工具描述 | `clarification_tool.py` | 工具 docstring |
| P-22 | `task` 工具描述 | `task_tool.py` | 工具 docstring |
| P-23 | `present_files` 工具描述 | `present_file_tool.py` | 工具 docstring |
| P-24 | `view_image` 工具描述 | `view_image_tool.py` | 工具 docstring |
| P-25 | `tool_search` 工具描述 | `tool_search.py` | 工具 docstring |

---

*DeerFlow 开源项目地址：https://github.com/bytedance/deer-flow*  
*本书分析版本：v2.0.0-rc1（tag 打出时间：2026-06-19）*
