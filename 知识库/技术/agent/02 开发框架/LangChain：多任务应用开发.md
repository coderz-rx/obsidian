# LangChain：多任务应用开发

> 来源：agent 课程 · 第 7 课《LangChain：多任务应用开发》（模块二：开发框架）
> 整理时间：2026-08-10

---

## 一、课前答疑：Skill、MCP 与 Function Call

### 1. Skill 和 MCP 的区别

大模型调用 **Skill（技能）** 与调用 **MCP 接口**，本质上都是基于大模型的 Function Calling / Tool Use 能力，但在**抽象层级、实现粒度、交互范式**上有明显差异：

- **MCP 接口**：偏底层、偏协议化（类比 TCP/IP 或 REST API），解决"大模型如何**连接**外部数据库、本地文件、SaaS 系统"的通信标准问题——可以理解为 AI 大模型体系下的 API 接口协议。
- **Skill**：偏高层、偏业务逻辑与工程规程（类比封装好的 SDK、工作流或 SOP），解决"大模型如何**按照特定步骤/规范**高效完成一项复杂任务"的问题。

在现代 Agent 框架（Semantic Kernel、Claude Code Skills、LangChain Agent Tools、OpenSpec 等）中，一个 Skill 通常不是单一函数，而是一个**包含特定领域上下文、控制逻辑与执行步骤的"能力包"**。标准结构通常包含：

1. **Skill Prompt / Instructions（元指令/SOP）**：告诉大模型这个技能是干什么的、什么情况下触发、分几步执行、避坑指南是什么
2. **Tools / Actions（底层工具链）**：Skill 依赖的实际物理工具（文件读写、Bash 命令、git 操作等），底层往往通过 MCP 或系统 API 提供
3. **Sub-agents / Workflows（子流程/子 Agent，可选）**：复杂 Skill（如 TDD 单元测试 Skill）可能派生出专门的子 Agent 独立跑流水线

### 2. 三者关系总结

| 层级 | 概念 | 定位 |
|------|------|------|
| 上层 | **Skill** | 业务编排：把"如何调用 MCP/API"编进流程；Skills 以列表形式加载到系统提示词中，让大模型发现并使用 |
| 中层 | **MCP** | 底层能力的具体实现（类似 Dubbo 接口 API），提供标准化的底层能力 |
| 底层 | **Function Call** | 大模型的底层能力：把自然语言转为结构化 JSON 指令；Agent 拿到 JSON 指令后即可调用 MCP 或 API（这部分是系统工程实现） |

---

## 二、LangChain 基本概念

LangChain 提供了一套工具、组件和接口（框架定位类似 Java 生态的 Spring），简化了 LLM 应用的创建过程。核心组件：

| 组件 | 作用 |
|------|------|
| **Models** | 模型接入层，如 GPT-4o |
| **Prompts** | 提示词管理、优化与序列化 |
| **Memory** | 保存与模型交互时的上下文记忆 |
| **Indexes** | 索引：结构化文档以便与模型交互。构建知识库时需要各类文档的加载、转换、长文本切割、文本向量计算、向量索引存储查询等 |
| **Chains** | 链：一系列对各种组件的调用编排 |
| **Agents** | 代理：决定模型采取哪些行动，执行并观察流程，直到完成为止 |

- API 风格全面转向 **LCEL**（LangChain Expression Language，链式表达式），鼓励用 `|` 运算符把组件拼成 Runnable
- 最基础的 LangChain 用法：将 Prompt 模板和 LLM 组合成可执行链

---

## 三、Tools

- **SerpAPI**：用于联网搜索的 Tool
- 可以使用 `@tool` 装饰器自定义 Tool——这也是早期实现 Function Call 的基本方式

---

## 四、Memory（短期记忆方案）

| 记忆方式 | 机制 |
|----------|------|
| **BufferMemory** | 将之前的对话完全存储下来，原样传给 LLM |
| **BufferWindowMemory** | 只存储最近 K 组对话，传给 LLM |
| **ConversationSummaryMemory** | 对对话进行摘要，将压缩后的历史对话传递给 LLM |
| **VectorStore-backed Memory** | 将所有对话向量化存入 VectorDB（向量数据库）；每次对话根据用户输入，检索向量库中最相似的 K 组历史对话 |

---

## 五、Case 讲解：ReAct 范式

- **ReAct = Reasoning + Acting**：将推理和动作相结合，克服 LLM"胡言乱语"（幻觉）的问题，同时提高结果的可解释性和可信赖度
- 人们在执行多步骤任务时，步骤之间、动作之间一般都会有推理过程——ReAct 正是模拟这一点（与 CoT 思维链"刨根问底"式推理一脉相承）

**核心认知：**

- **Agent** 的核心是把 LLM 当作**推理引擎**，让它能使用外部工具以及自己的长期记忆，从而完成灵活的决策步骤，进行复杂任务
- **Chain** 是由**人**定义的一套流程步骤让 LLM 执行——可以看成把 LLM 当成了一个强大的多任务工具
- **典型 Agent 逻辑（如 ReAct）**：
  1. 由 LLM 选择工具
  2. 执行工具后，将输出结果返回给 LLM
  3. 不断重复上述过程，直到达到停止条件（通常是 LLM 自己认为找到答案了）

---

## 六、使用 LCEL 构建任务链

工具链组合设计：在 Agent 系统中，工具链是实现复杂任务的关键组件。通过将多个工具组合，Agent 可以逐步处理复杂问题。

LCEL 是 LangChain 推出的链式表达式语言，支持用 `|` 操作符将各类单元（Prompt、LLM、Parser 等）组合。每个 `|` 左侧的输出会自动作为右侧的输入，实现数据流式传递。

**优势：**

- 代码简洁，逻辑清晰，易于多步任务编排
- 支持多分支、条件、并行等复杂链路
- 易于插拔、复用和调试每个子任务

**典型用法：**

```python
# 串联：A 的输出传给 B，B 的输出传给 C
chain = prompt | llm | output_parser

# 分支/并行：A 和 B 并行执行
branch = {"x": chain_a, "y": chain_b}

# 流式：.stream() 方法可边生成边消费
for chunk in chain.stream(input):
    print(chunk)
```

---

## 七、AI Agent 框架速览

课程提及的对比对象：LangChain、LangGraph、Coze、Dify、Qwen-Agent 等（详细对比见第 8 课《AI框架设计与选型》笔记）。

- **LangChain**：全能型 LLM 应用框架，生态与组件最丰富，适合复杂场景
- **LangGraph**：LangChain 出品，基于图（Graph）的 Agent 编排，适合带状态、带循环的复杂控制流
- **Coze / Dify**：低代码平台，可视化拖拽搭建，适合快速验证与业务同学使用
- **Qwen-Agent**：通义官方轻量框架，内置 RAG 与 Code Interpreter，适合原型开发
