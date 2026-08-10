# AI 框架设计与选型

> 来源：agent 课程 · 第 8 课《AI框架设计与选型》（模块二：开发框架）
> 整理时间：2026-08-10

---

## 一、AI Agent 的四大核心问题

设计一个 Agent 框架，本质上要解决四个核心问题：**大脑（LLM 接口）、双手（工具）、记忆（Context）、中枢（编排）**。

### 1. LLM 统一接口（大脑）

- 大脑的适配层：**LLM 统一接口 + Prompt 管理**
- 通过统一接口屏蔽各家模型（GPT、Claude、Qwen……）的 API 差异，让上层逻辑与具体模型解耦，模型可插拔

![[images/agent-llm-interface.png]]

### 2. 工具注册与调度（双手）

- 工具以注册表形式挂载到 Agent，框架负责把工具描述（名称、参数 schema、用途）注入提示词
- 注意：**调用工具本身不消耗 token，"推理要使用哪些工具"才消耗 token**——工具描述越精简、数量越克制，推理成本越低

![[images/agent-tool-scheduling.png]]

### 3. Context 管理机制（记忆）

- **短期记忆**：
  - 对话历史截断
  - 滑动窗口策略
  - 避免 token 爆炸
- **长期记忆**：
  - VectorStoreIndex 向量索引
  - 文档分块与 Embedding
  - 相似度检索

![[images/agent-context-memory.png]]

### 4. 控制流编排（中枢）

- **为什么需要控制流编排**：复杂任务不能靠 LLM 一口气说完，必须拆解步骤、控制节奏
- **三种经典编排模式**：
  - **Chain（链式）**：输入确定、输出确定，像工厂流水线（Pipeline）
  - **Loop（循环）**：即 ReAct 模式（思考—行动—观察—再思考），像一个不断试错的实验员，直到任务完成
  - **DAG（有向无环图）**：像多人接力赛，任务节点间有明确的前后依赖关系

![[images/agent-orchestration.png]]

---

## 二、主流 Agent 框架

### 1. LangChain —— 全能 LLM 应用框架

- 全能、生态和组件丰富、适合各种复杂场景
- 向量数据库支持矩阵：
  - 本地/轻量：`Chroma` / `FAISS`
  - 生产环境（已有关系型数据库）：`PostgreSQL + PGVector`
  - 生产环境（海量/亿级数据）：`Milvus` / `Qdrant` / `Pinecone`

### 2. LlamaIndex —— 数据驱动的 RAG 专家

- **定位**：为 LLM 装上私有数据的最强接口。如果不涉及复杂的多 Agent 协作、只想做基于文档的问答，它是首选
- 不同于 LangChain 关注**流程**，LlamaIndex 关注**数据结构**——它认为 LLM 应用的核心瓶颈在于如何让 LLM 索引私有数据
- LlamaIndex 本身不是向量数据库，而是"高级索引与检索引擎"，**与 LangChain 可以完美配合**：
  - 最经典的组合范式：**用 LlamaIndex 做前端的数据清洗、解析、复杂索引与高级检索，然后将 LlamaIndex 包装为一个 `Retriever`（检索器），喂给 LangChain 的 Agent 或 LCEL 链做决策与编排**
  - LlamaIndex 原生提供了对 LangChain 的适配工具（在 `llama-index-core` 中）
- **MinerU**：专业处理 PDF 文档的解析工具（公式、表格、版面还原效果好）

### 3. Qwen-Agent —— 轻量级的全能选手

- 有可视化界面（Gradio），适合做快速原型展示（原型开发）
- 内置 RAG 流程（Agentic RAG）、Code Interpreter（数据分析）
- 特点：快、简洁

![[images/qwen-agent-scenarios.png]]

### 4. 多框架选型对比

- **选 LangChain**：开发通用 AI 应用，需要灵活控制流程，或者需要切换多种模型
- **选 LlamaIndex**：主要做 RAG（企业知识库），手里有一堆 PDF/Word/Excel 要处理
- **选 Qwen-Agent**：主要用 Qwen 模型，需要做数据分析（Code Interpreter）或处理超长文档（1M Context）
- **选 AutoGen**：任务太复杂，一个人（Agent）干不完，需要团队（多角色）"吵架"/协作才能出结果

![[images/agent-framework-comparison.png]]
