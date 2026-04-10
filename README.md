# AI 学习文档与代码 📚🤖

本项目是个人 AI 学习过程中的知识沉淀，涵盖提示词工程、LangChain 框架学习等内容，包含详细的文档说明和代码示例。

---

## 📁 项目结构

```
ai-learn-doc/
├── 01_提示词/                          # 提示词工程学习
│   ├── 00_输出格式.md                   # AI 输出格式规范
│   ├── 01_技巧.md                       # 提示词核心技巧与参数调优
│   └── 02_10 个提升学习效率的 GPT 指令模板.md
│
├── 02_langchain学习路线架构图/          # LangChain TS 学习
│   ├── 00_路线.md                       # 分阶段学习路线
│   ├── 01.plantuml                      # 架构图
│   ├── 02_LangChain_TS_组件关系图.plantuml
│   ├── 03_LangChain_TS_学习时序图.plantuml
│   ├── 04_LangChain_TS_RAG流程图.plantuml
│   ├── 05_LangChain_TS_Agent架构图.plantuml
│   ├── 06_LangChain_TS_LangGraph状态机.plantuml
│   └── *.svg                            # 架构图导出文件
│
└── README.md                            # 本文件
```

---

## 🎯 内容概览

### 一、提示词工程 (Prompt Engineering)

学习如何与 AI 高效沟通，掌握提示词设计的核心技巧：

| 主题         | 内容                           |
| ------------ | ------------------------------ |
| **角色扮演** | 通过角色设定引导 AI 输出风格   |
| **核心技巧** | 指令、场景、输入、输出四要素   |
| **参数调优** | Temperature、Top-P、Top-K 详解 |
| **实用模板** | 10 个提升学习效率的 GPT 指令   |

**关键知识点：**

- 🌡️ **Temperature 梯度**：0.0(冻结) → 0.3(保守) → 0.7(平衡) → 1.0(活跃) → 1.3+(奔放)
- 🎛️ **Top-P 配置**：低(0.1-0.5)精确输出，中(0.7-0.9)平衡，高(0.95-1.0)创意
- 🔧 **问题诊断**：输出重复、生僻词、逻辑跳跃等问题的修复方案

---

### 二、LangChain TypeScript 学习

系统学习 LangChain.js 框架，从基础到进阶的完整路线：

#### 📚 分阶段学习路线

| 阶段       | 周期   | 目标     | 核心内容                                      |
| ---------- | ------ | -------- | --------------------------------------------- |
| **阶段 1** | 1-2 周 | 基础入门 | Model I/O、Prompts、Output Parsers、LCEL 基础 |
| **阶段 2** | 2-3 周 | 核心进阶 | Memory、RAG、Agents                           |
| **阶段 3** | 2-3 周 | 高级应用 | LangGraph、生产部署、性能优化                 |

#### 🏗️ 架构图一览

| 图表             | 说明                              |
| ---------------- | --------------------------------- |
| 组件关系图       | LangChain TS 核心组件间的依赖关系 |
| 学习时序图       | 各阶段学习内容的调用顺序          |
| RAG 流程图       | 检索增强生成的完整流程            |
| Agent 架构图     | 智能体的设计与执行流程            |
| LangGraph 状态机 | 状态化工作流的流转逻辑            |

#### 🔧 核心包体系

```
@langchain/core          # 核心抽象（Prompts、Messages、Runnables）
langchain                # 高层封装（Chains、Agents）
@langchain/openai        # OpenAI 集成
@langchain/langgraph     # 状态化 Agent 工作流
@langchain/community     # 社区集成（向量库、文档加载器等）
```

## 📝 更新日志

| 日期    | 更新内容                           |
| ------- | ---------------------------------- |
| 2026-04 | 项目初始化，添加提示词工程文档     |
| 2026-04 | 添加 LangChain TS 学习路线与架构图 |


还在更新中。。。。
---

## 📌 相关资源

- [LangChain 官方文档](https://js.langchain.com/)
- [OpenAI API 文档](https://platform.openai.com/docs)
- [LangGraph 文档](https://langchain-ai.github.io/langgraph/)

---

> 💡 **提示**: 本项目为个人学习笔记，内容持续更新中。如有错误或建议，欢迎指正交流！
