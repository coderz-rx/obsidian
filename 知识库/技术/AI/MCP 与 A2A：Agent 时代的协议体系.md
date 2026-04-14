# MCP 与 A2A：Agent 时代的工具协议与协作协议

> 本文合并自《A2A 与 Agent+MCP》和《AI MCP》，并补充了 MCP 实现原理、自定义 Server 开发及对外暴露的完整实践。

---

## 一、背景：为什么需要 MCP 和 A2A？

大模型本身只有"大脑"，没有"手脚"。为了让 Agent 能够查天气、读数据库、发邮件、操作文件系统，业界早期采用 **Function Calling** 的方式——每个应用自己定义一套 JSON Schema，自己对接 API。这导致了严重的碎片化：

- 同样的 "查天气" 能力，ChatGPT、Claude、Cursor 各写一套适配层
- 同样的 "读 GitHub Issue"，每个 Agent 框架都要重新集成
- 工具提供方（如 Slack、Notion）需要为每个平台写适配器

**MCP（Model Context Protocol）** 的本质就是解决这个碎片化问题：它为 "大模型如何调用外部工具" 定义了一个**统一、开放、去中心化**的标准协议。

而 **A2A（Agent-to-Agent Protocol）** 解决的是另一个维度的问题：当系统中有多个 Agent 时，它们如何**发现彼此、协商任务、交换中间结果**。

| 协议 | 解决的问题 | 类比 |
|------|-----------|------|
| **MCP** | Agent 与**工具/外部系统**的通信 | USB-C 接口标准 |
| **A2A** | Agent 与**Agent**之间的通信 | 企业间的 B2B 协作流程 |

---

## 二、MCP：Model Context Protocol

### 2.1 什么是 MCP

MCP 是由 **Anthropic 于 2024 年底开源提出**的开放协议，现已成为 AI 应用与外部工具对接的事实标准之一。Cursor、Claude Desktop、Claude Code、OpenClaw、Windsurf 等均已支持。

一句话概括：**MCP 是面向大模型的 "USB-C" 接口标准**。任何符合 MCP 协议的工具（MCP Server），可以被任何符合 MCP 协议的 Agent（MCP Client）即插即用。

### 2.2 MCP 的核心定位

- **Server = API 接口提供者**：暴露一组可被调用的工具（Tools），也可能暴露资源（Resources）和提示模板（Prompts）
- **Client = 业务编排的决策者**：通常是 Agent 框架或 IDE，负责根据用户意图选择调用哪个 Server 的哪个 Tool
- **Host = 宿主应用**：承载 Client 的实际应用程序（如 Claude Desktop、Cursor、OpenClaw CLI）

MCP 的定位不是取代 Function Calling，而是**将其规范化、标准化、生态化**。它让 "大模型做自己应该做的事（推理和决策）"，而把 "手脚能力" 交给专业的 Server 去实现。

### 2.3 MCP 的三层架构

```
┌─────────────────────────────────────────┐
│              Host（宿主应用）             │
│   Claude Desktop / Cursor / OpenClaw    │
├─────────────────────────────────────────┤
│           MCP Client（客户端）            │
│  负责：意图理解 → 工具选择 → 参数组装      │
├─────────────────────────────────────────┤
│  MCP Server A   MCP Server B   MCP ...  │
│  ├─ 查天气       ├─ 读数据库      ...    │
│  ├─ 发邮件       ├─ 操作文件             │
│  └─ 查 GitHub    └─ 调用 Slack          │
└─────────────────────────────────────────┘
```

**一个 Host 可以连接多个 Client，一个 Client 可以连接多个 Server。**

### 2.4 MCP 的实现原理

#### 2.4.1 基于 JSON-RPC 2.0

MCP 的通信协议完全基于 **JSON-RPC 2.0**。所有交互都是标准的请求-响应或通知消息：

```json
// Client 请求调用工具
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": { "city": "Beijing" }
  }
}

// Server 返回结果
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      { "type": "text", "text": "北京今天晴，25°C" }
    ]
  }
}
```

#### 2.4.2 核心通信原语

| 方法 | 方向 | 作用 |
|------|------|------|
| `initialize` | C → S | 握手，协商协议版本和能力 |
| `initialized` | C → S | 通知 Server 初始化完成 |
| `tools/list` | C → S | 获取 Server 暴露的所有工具列表 |
| `tools/call` | C → S | 调用指定工具 |
| `resources/list` | C → S | 获取资源列表 |
| `prompts/list` | C → S | 获取提示模板列表 |
| `notifications/progress` | S → C | Server 报告进度 |

#### 2.4.3 传输层（Transports）

MCP 协议本身与传输层解耦，官方支持三种传输方式：

| 传输方式 | 适用场景 | 特点 |
|----------|----------|------|
| **stdio** | 本地 Server | Server 作为子进程启动，通过 stdin/stdout 通信，最安全 |
| **SSE** | 远程 Server | 基于 HTTP，Client 通过 SSE 接收 Server 推送，适合 Web 环境 |
| **HTTP** | 远程/无状态 | 直接 REST-like 调用，最简单，但部分高级特性受限 |

**Claude Desktop 默认使用 stdio**，因为它启动的 MCP Server 通常是本地脚本；而 **Web 版 Claude 或远程部署则更可能用 SSE/HTTP**。

#### 2.4.4 能力协商（Capabilities）

初始化时，Client 和 Server 会交换各自的 `capabilities`：

```json
{
  "protocolVersion": "2024-11-05",
  "capabilities": {
    "tools": { "listChanged": true },
    "resources": {},
    "prompts": {}
  },
  "clientInfo": { "name": "claude-desktop", "version": "1.0" }
}
```

这意味着：如果一个 Server 只暴露了 Tools，但没有 Resources，Client 就不会去请求 `resources/list`。

#### 2.4.5 完整的工具调用生命周期

```
1. Host 启动，加载配置中注册的所有 MCP Servers
2. 对每个 Server：
   a. 建立传输连接（stdio / SSE / HTTP）
   b. 发送 initialize 握手
   c. 发送 tools/list 获取工具列表
   d. 将工具名称、描述、参数 Schema 注入 LLM 的系统提示
3. 用户输入到达后：
   a. LLM 根据系统提示中的工具信息，决定调用哪个 Tool
   b. Client 组装 JSON-RPC 请求，发送 tools/call
   c. Server 执行实际逻辑，返回结果
   d. Client 将结果回传给 LLM，LLM 生成最终回复
```

---

## 三、如何自定义一个 MCP Server

### 3.1 开发流程概览

自定义 MCP Server 的完整流程：

```
选择 SDK → 定义 Tools → 实现 Handlers → 选择 Transport → 注册到 Host
```

### 3.2 使用 Python SDK 开发（推荐入门）

Anthropic 官方提供了 Python SDK：`mcp`。

#### 步骤 1：安装依赖

```bash
pip install mcp
```

#### 步骤 2：编写 Server 代码

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent
import asyncio

# 创建 Server 实例
server = Server("my-weather-server")

@server.list_tools()
async def list_tools():
    """告诉 Client 我有哪些工具"""
    return [
        Tool(
            name="get_weather",
            description="获取指定城市的当前天气",
            inputSchema={
                "type": "object",
                "properties": {
                    "city": {"type": "string", "description": "城市名称，如 Beijing"}
                },
                "required": ["city"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    """处理工具调用"""
    if name == "get_weather":
        city = arguments.get("city", "Unknown")
        # 这里可以调用真实的天气 API
        result = f"{city} 今天晴朗，气温 25°C，空气质量优。"
        return [TextContent(type="text", text=result)]
    
    raise ValueError(f"Unknown tool: {name}")

async def main():
    async with stdio_server(server) as (read_stream, write_stream):
        await server.run(
            read_stream,
            write_stream,
            server.create_initialization_options()
        )

if __name__ == "__main__":
    asyncio.run(main())
```

#### 步骤 3：本地测试运行

```bash
python weather_server.py
```

如果一切正常，这个程序会等待 stdin 上的 JSON-RPC 消息（不要手动输入，它是在被 Host 启动时工作的）。

### 3.3 使用 TypeScript SDK 开发

官方 TypeScript SDK：`@modelcontextprotocol/sdk`。

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

const server = new Server(
  { name: "my-weather-server", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: "get_weather",
        description: "获取指定城市的当前天气",
        inputSchema: {
          type: "object",
          properties: {
            city: { type: "string", description: "城市名称" },
          },
          required: ["city"],
        },
      },
    ],
  };
});

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name === "get_weather") {
    const city = request.params.arguments?.city || "Unknown";
    return {
      content: [
        { type: "text", text: `${city} 今天晴朗，气温 25°C。` },
      ],
    };
  }
  throw new Error("Unknown tool");
});

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
}

main();
```

---

## 四、对外暴露 MCP Server 的三种方式

### 4.1 方式一：stdio（本地子进程）

**最常用、最安全**，适用于 Claude Desktop、Claude Code 等本地应用。

**配置示例（Claude Desktop 的 `claude_desktop_config.json`）**：

```json
{
  "mcpServers": {
    "weather": {
      "command": "python",
      "args": ["/absolute/path/to/weather_server.py"]
    }
  }
}
```

启动时，Host 会 `spawn` 这个 Python 进程，通过 stdin/stdout 与其通信。进程结束后自动关闭连接。

**优点**：
- 零网络暴露，最安全
- 开发调试最简单
- 本地权限天然隔离

**缺点**：
- 只能在单台机器上使用
- 无法被远程 Client 调用

### 4.2 方式二：SSE（Server-Sent Events）

**适合远程暴露**，让局域网或互联网上的 Client 连接你的 Server。

#### Python SSE Server 示例

```python
from mcp.server import Server
from mcp.server.sse import SseServerTransport
from mcp.types import Tool, TextContent
from starlette.applications import Starlette
from starlette.routing import Route
import uvicorn

server = Server("my-weather-server")
sse = SseServerTransport("/message")

@server.list_tools()
async def list_tools():
    return [Tool(name="get_weather", description="...", inputSchema={...})]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    return [TextContent(type="text", text="晴天 25°C")]

async def handle_sse(request):
    async with sse.connect_sse(
        request.scope, request.receive, request._send
    ) as (read_stream, write_stream):
        await server.run(
            read_stream,
            write_stream,
            server.create_initialization_options()
        )

app = Starlette(routes=[
    Route("/sse", endpoint=handle_sse),
    Route("/message", endpoint=sse.handle_post_message, methods=["POST"]),
])

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=3000)
```

**Client 连接方式**：

```json
{
  "mcpServers": {
    "weather": {
      "url": "http://localhost:3000/sse"
    }
  }
}
```

**工作原理**：
- Client 先向 `/sse` 建立 SSE 长连接，用于接收 Server 推送
- 后续所有 Client → Server 的请求通过 POST 到 `/message`
- Server → Client 的响应通过 SSE 通道推送回去

**优点**：
- 支持远程访问
- 基于标准 HTTP，易于穿透 NAT、部署到容器/云端
- 支持 Server 主动向 Client 推送进度和通知

**缺点**：
- 需要处理网络安全（防火墙、认证）
- 相比 stdio 多了一个网络层

### 4.3 方式三：HTTP（无状态 REST）

MCP 协议本身是基于 JSON-RPC 的，但如果你只暴露简单的工具调用能力，也可以直接写一个 HTTP 接口，外层包装一层 JSON-RPC 适配器。

更常见的做法是：使用支持 HTTP Transport 的 MCP SDK 或网关（如 `mcp-proxy`）。

```bash
# 用 mcp-proxy 把 stdio Server 暴露为 HTTP
npx mcp-proxy python weather_server.py --port 3001
```

这样外部 Client 就可以直接通过 `http://localhost:3001` 访问这个 Server。

---

## 五、MCP 的注册与发现：没有统一目录

### 5.1 MCP 是去中心化的

**MCP 协议本身没有强制性的统一注册中心。** 这与很多传统 SaaS 平台的 "应用商店" 模式有本质区别。

一个团队或个人开发了一个 MCP Server 后，可以直接通过 SSE/HTTP 暴露在自己的域名/IP 上。任何知道这个地址的 Client 都可以直接连接，**不需要向任何第三方注册**。

例如你自己的 Server 跑在：
```
https://your-api.com/mcp/sse
```
Claude Desktop、Cursor、OpenClaw 都可以直接配置这个 URL 来连接，中间没有任何网关或审批。

### 5.2 获取 MCP Server 的三种途径

| 途径 | 说明 | 例子 |
|------|------|------|
| **独立部署** | 绝大多数情况。团队或个人自建自管 | 公司内部的 HR 系统 MCP Server |
| **聚合市场** | 可选的 "黄页"，方便发现，但非强制 | Smithery、MCP.so、Cursor Marketplace、Glama |
| **封闭生态** | 特定平台自己的分发渠道 | OpenClaw 的 `clawhub`、Claude Code Plugins |

这些市场的作用类似于**应用商店**——开发者可以自愿提交，用户可以上去搜索，但完全不涉及协议层面的 "统一注册" 或 "路由转发"。

### 5.3 Client 如何获取 MCP 能力？目前是手动的

**Client 不会自动扫描互联网来找工具。** 你必须先把 Server 的信息手动配置到 Client 中。

以 Claude Desktop 为例，需要编辑 `claude_desktop_config.json`：

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"]
    },
    "my-custom-server": {
      "url": "https://your-api.com/mcp/sse"
    }
  }
}
```

配置完成后，Client 下次启动时才会：
1. 按照配置连接这些 Server
2. 调用 `tools/list` 获取工具列表
3. 把工具信息注入到大模型的系统提示中

**如果某个 Server 没有在配置里注册，Client 完全不知道它的存在。**

### 5.4 大模型能感知到 MCP 服务吗？

**完全不能主动感知。**

大模型对 MCP 工具的认知范围，严格受限于 **Client 在系统提示中注入的内容**。它看到的只是类似这样的一段描述：

```
You have access to the following tools:
- get_weather(city: string): 获取指定城市的天气
- search_github(query: string): 搜索 GitHub 仓库
```

如果某个 Server 没被连接，这段描述里就没有对应的工具，**大模型绝不会凭空知道它可以调用某个外部服务**，也不会主动上网搜索新的 MCP Server。

### 5.5 未来的可能方向

业界正在探索的改进方向包括：
- **一键安装**：聚合市场和 Host 深度集成，在 Client 内直接搜索安装
- **动态注册表**：Server 暴露元注册表，Client 可以查询可用 Server 列表
- **Agent 驱动发现**：由 Agent 自主决定是否连接新的 Server

但这些目前都还处于概念或早期实验阶段，**当前最现实的工作流仍然是 "手动发现 + 手动配置"**。

---

## 六、MCP Server 的安全与认证

### 6.1 stdio 模式的安全边界

由于 Server 作为子进程运行，它的权限完全取决于启动它的用户。建议：
- 对敏感操作（如写文件、执行系统命令）做白名单校验
- 在 Server 代码中显式限制可访问的目录和命令

### 6.2 远程 SSE/HTTP 模式的认证

目前 MCP 协议本身**没有内建认证机制**，但你可以在 HTTP 层实现：

- **API Key**：Client 在请求头中携带 `X-API-Key`
- **Bearer Token**：SSE 连接时通过 Query Parameter 或 Header 传递 JWT
- **mTLS**：在云环境中使用双向 TLS 认证
- **IP 白名单**：限制只有特定 Host 可以访问

**示例：API Key 校验中间件**

```python
from starlette.middleware.base import BaseHTTPMiddleware

class APIKeyMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        api_key = request.headers.get("X-API-Key")
        if api_key != "your-secret-key":
            return Response("Unauthorized", status_code=401)
        return await call_next(request)

app.add_middleware(APIKeyMiddleware)
```

---

## 七、A2A：Agent-to-Agent Protocol

### 7.1 什么是 A2A

A2A 是由 Google 主导的开放协议（2025 年提出），解决的是**多 Agent 协作**的问题。

如果说 MCP 是 "Agent 调用工具的标准接口"，那么 A2A 就是 "Agent 之间通信的标准接口"。

### 7.2 A2A 与 MCP 的本质区别

| 维度 | MCP | A2A |
|------|-----|-----|
| **通信双方** | Agent ↔ Tool/Service | Agent ↔ Agent |
| **关系** | 主从调用 | 对等协作 |
| **信息交换** | 单次工具调用，返回结果 | 多轮协商，传递任务、状态、中间产物 |
| **典型场景** | 查天气、读数据库、发邮件 | 调度 Agent 分解任务，审查 Agent 检查结果 |
| **协议层** | JSON-RPC 2.0 | HTTP + JSON（基于 Agent Card） |

### 7.3 A2A 解决的核心问题

1. **Agent 发现**：一个 Agent 如何找到另一个 Agent 的能力（Agent Card）
2. **任务委托**：如何将复杂任务拆解并分发给不同垂域 Agent
3. **状态同步**：多轮协作中，Agent 之间如何交换中间状态和最终产物
4. **安全边界**：不同厂商、不同权限的 Agent 如何在可信范围内协作

### 7.4 为什么分 Agent？

> "一个脑子装太多内容容易混乱，各司其职可能效果会更好。"

分 Agent 的本质不是为了 "做不同 function call 的封装"，而是为了 **Context 隔离** 和 **推理专注度**：

- **Context 隔离**：每个垂域 Agent 只加载自己领域的上下文，避免长对话中的上下文腐化
- **推理专注度**：单一 Agent 的 Prompt 和工具集更聚焦，决策质量更高
- **模块化扩展**：新增垂域能力只需要增加一个 Agent，调度 Agent 的架构保持不变

### 7.5 垂域 Agent 如何具备业务能力？

你提到一个关键问题：如果不做微调，垂域 Agent 怎么具备领域知识？

**答案是：知识不加载到权重中，而是外置到 Context 中。**

现代 Agent 架构普遍采用 **RAG + 领域文档 + 结构化 Prompt** 的组合：

```
用户请求 → 调度 Agent
              ↓
      路由到「财务审计 Agent」
              ↓
    加载财务规则文档（SKILL.md / CLAUDE.md）
    加载相关账表数据（通过 MCP 读取）
    加载历史审计案例（RAG 检索）
              ↓
    在浓缩后的 Context 中进行推理和决策
```

这与人类专家的工作方式类似：不需要把整本税法背在脑子里，而是在处理具体案件时查阅相关条款和先例。

---

## 八、MCP + A2A 的协同架构

在一个复杂的企业级 Agent 系统中，MCP 和 A2A 通常是**共存互补**的：

```
┌─────────────────────────────────────────────┐
│            Orchestrator Agent               │
│         （调度中枢，通过 A2A 协议通信）         │
└──────────────┬──────────────────────────────┘
               │ A2A
    ┌──────────┼──────────┐
    ▼          ▼          ▼
┌───────┐  ┌───────┐  ┌───────┐
│财务Agent│  │法务Agent│  │开发Agent│
└───┬───┘  └───┬───┘  └───┬───┘
    │ MCP      │ MCP      │ MCP
    ▼          ▼          ▼
┌───────┐  ┌───────┐  ┌───────┐
│账表DB  │  │法规库  │  │GitHub  │
│SAP接口 │  │合同系统│  │CI/CD   │
└───────┘  └───────┘  └───────┘
```

- **A2A** 负责 Agent 之间的任务分发和协作
- **MCP** 负责每个 Agent 与底层工具/数据的对接

---

## 九、总结

### MCP 的核心价值

> **让大模型做自己应该做的事（推理和决策），而 "手脚能力" 交给标准化的 Server 去实现。**

- **实现原理**：基于 JSON-RPC 2.0，通过 `initialize` → `tools/list` → `tools/call` 完成工具发现与调用
- **自定义方式**：使用 Python/TS SDK 定义 Tools 和 Handlers，选择 stdio/SSE/HTTP 传输层
- **对外暴露**：本地用 stdio，远程用 SSE，网关场景用 HTTP 或 mcp-proxy

### A2A 的核心价值

> **解决多 Agent 协作的「发现、委托、同步」问题。**

- 与 MCP 不是竞争关系，而是**上下层关系**
- MCP 是 Agent 的「手和脚」，A2A 是 Agent 之间的「语言和流程」

### 实践建议

1. **从 MCP 开始**：先让自己的 Agent 具备标准化的工具调用能力
2. **拆分垂域 Skill**：用 SKILL.md 把领域知识模块化，避免 CLAUDE.md 膨胀
3. **再引入 A2A**：当单一 Agent 的上下文和职责过于复杂时，才考虑多 Agent 协作架构
4. **知识外置化**：优先用文档、RAG、MCP 资源来承载领域知识，而非微调

---

*记录时间：2026-04-14*
