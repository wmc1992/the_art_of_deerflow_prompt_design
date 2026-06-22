# DeerFlow Prompt 设计艺术

基于 [DeerFlow](https://github.com/bytedance/deer-flow) v2.0.0-rc1 的 Prompt 工程深度解析，面向算法工程师的实践指南。

---

## 关于本书

DeerFlow 是字节跳动开源的超级 Agent 框架。本书系统分析其全部 Prompt 设计：

- **25 个 Python 源码 Prompt**（P-01 至 P-25）：主系统 Prompt、子 Agent 编排、记忆系统、Loop 检测、工具 Schema 等
- **22 个内置技能文件**（S-01 至 S-22）：从最轻量的参考型到最复杂的两阶段分析框架

每个 Prompt 均给出英文原文（与源码逐字一致）+ 中文译注 + 设计解读。

**读完本书，你将能够：**

- 理解工业级 Agent 系统中 Prompt 的完整设计体系
- 掌握前缀缓存优化、静动分离、XML 锚点等核心设计模式
- 具备评估和设计复杂多 Agent 系统 Prompt 的能力

---

## 阅读方式

**在线阅读**（推荐）：访问 [GitHub Pages](http://mingchao.wang/the_art_of_deerflow_prompt_design/)

**本地运行**：

```bash
pip install mkdocs-material
mkdocs serve
```

访问 `http://127.0.0.1:8000`

---

## 书籍结构

```
前言
├── 序言：为什么要读一个框架的 Prompt
└── 如何使用本书

第一部分：基础认知
├── 第 1 章：DeerFlow 系统概览与 Prompt 地图
├── 第 2 章：Prompt 工程核心原则
└── 第 3 章：静态与动态 Prompt 的设计哲学

第二部分：主系统 Prompt 深度解析
├── 第 4 章：角色声明与身份锚定
├── 第 5 章：思维风格与澄清系统
├── 第 6 章：技能系统与延迟工具
└── 第 7 章：约束体系

第三部分：子系统 Prompt 解析
├── 第 8 章：子 Agent 编排——并行设计哲学
├── 第 9 章：记忆系统——知识提炼机制
└── 第 10 章：安全防线——Loop 检测、TodoList 与标题生成

第四部分：设计模式与综合提升
├── 第 11 章：七大可复用设计模式
└── 第 12 章：从 DeerFlow 到你的系统——实战迁移指南

附录
├── 附录 A：全部 Prompt 英文原文（P-01 至 P-25）
├── 附录 B：全部 Prompt 中文译注（P-01 至 P-25）
├── 附录 C：全部技能文件英文原文（S-01 至 S-22）
└── 附录 D：全部技能文件中文译注（S-01 至 S-22）
```

---

## 核心设计原则速览

本书提炼的七大可复用设计模式：

| 模式 | 核心思想 |
|------|---------|
| 具名场景模式 | 用有名字的分类替代抽象规则，降低模型模式匹配成本 |
| 后果约束模式 | 描述违反会发生什么，激活因果推理而非盲目服从 |
| 静态/动态分离模式 | 静态内容进 System Prompt 可缓存，动态内容走 `<system-reminder>` |
| 元数据懒加载模式 | Skills 只存 name + description + path，完整 SOP 按需加载 |
| XML 锚点模式 | 用 XML 标签切割语义单元，帮助模型定位相关规则段落 |
| 末尾强化模式 | `<critical_reminders>` 在末尾重复高优先级规则，对抗 lost-in-the-middle |
| 伪代码示范模式 | 执行序列用伪代码，行为规范用自然语言 |

---

## 说明

本书分析基于 DeerFlow **v2.0.0-rc1**（GitHub tag：2026-06-19）。  
所有源码路径相对于仓库的 `backend/packages/harness/` 目录。

> 本书内容**完全由 Claude Code + Claude Sonnet 4.6 自动生成**，未经人工逐字校验。英文 Prompt 原文已与源文件逐字核对，分析文字与中文翻译为模型输出，可能存在误读或过度推断。遇到存疑之处，以 DeerFlow 源代码为准。
