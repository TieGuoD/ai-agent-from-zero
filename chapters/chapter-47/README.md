# 第47章：MCP 协议 —— Agent 的标准化接口

## 学习目标

通过本章的学习，你将能够：
1. 理解 MCP（Model Context Protocol）协议的设计理念和架构
2. 掌握 MCP 中 Server、Client、Transport 三个核心概念
3. 学会使用 Python 实现 MCP Server 和 Client
4. 理解 MCP 的工具、资源和提示模板三大原语
5. 掌握基于 stdio 和 SSE 两种传输方式的实现
6. 能够构建一个完整的 MCP 应用

## 核心问题

在前面的章节中，我们学习了如何为 Agent 构建各种工具。但这里有一个问题：每个 Agent 框架都有自己定义工具的方式，每种 LLM API 也有自己的工具格式。这意味着你为 OpenAI 写的工具代码，不能直接用在 Claude 上；你为 LangChain 写的工具，不能直接用在 AutoGen 上。

这就好比每个国家都有自己独特的电源插头标准——你去日本买的电器，带回中国可能就用不了，需要一个转换器。这种碎片化让工具开发者和 Agent 开发者都很痛苦。

MCP（Model Context Protocol，模型上下文协议）正是为了解决这个问题而诞生的。它是一个开放的标准化协议，旨在统一 Agent 与外部工具、数据源之间的通信方式。就像 USB 接口统一了各种设备的连接方式一样，MCP 希望统一 Agent 与外界交互的方式。

那么，MCP 到底是什么？它解决了哪些问题？我们如何使用它？

---

## 原理讲解

### MCP 的诞生背景

在 MCP 出现之前，Agent 生态系统面临一个严重的碎片化问题。

假设你开发了一个很棒的数据库查询工具。为了让 Agent 能够使用这个工具，你需要为每个 Agent 框架分别适配：LangChain 有它自己的 Tool 接口，OpenAI 有 Function Calling 格式，Claude 有自己的 Tool Use 格式，AutoGen 又有自己的方式。这意味着你的工具需要被"翻译"成多种格式，才能被不同的 Agent 使用。

反过来也是一样。如果你开发了一个 Agent 框架，你需要为每个可能的工具分别适配。如果有 N 种工具和 M 种框架，你可能需要实现 N*M 个适配器。

Anthropic 在 2024 年底发布了 MCP 协议，提出了一个优雅的解决方案：在工具和 Agent 之间引入一个标准层。工具开发者只需要按照 MCP 标准实现一次，就可以被所有支持 MCP 的 Agent 使用。Agent 开发者也只需要实现一次 MCP 客户端，就可以接入所有 MCP 兼容的工具。

```
传统方式（N*M 适配器）:
Agent 框架 A ←→ 工具 1
Agent 框架 A ←→ 工具 2
Agent 框架 B ←→ 工具 1
Agent 框架 B ←→ 工具 2

MCP 方式（N+M 适配器）:
Agent 框架 A ←→ MCP 协议 ←→ 工具 1
Agent 框架 B ←→ MCP 协议 ←→ 工具 2
```

### MCP 的核心架构

MCP 采用了一种客户端-服务器架构。在这种架构中：

**MCP Host（宿主）**：这是一个应用程序，比如 Claude Desktop、IDE 插件或者你自己的 Agent 应用。它需要访问外部工具和数据。

**MCP Client（客户端）**：这是 Host 内部的组件，负责与 MCP Server 通信。一个 Host 可以包含多个 Client，每个 Client 与一个 Server 保持一对一的连接。

**MCP Server（服务器）**：这是一个轻量级的服务，暴露特定的工具、资源和提示模板。Server 只负责提供能力，不关心谁在使用这些能力。

用一个比喻来理解：MCP Server 就像是一个个应用商店里的 App，MCP Client 就像是你的手机上的应用商店客户端，MCP Host 就像是你的手机本身。

### 三大原语：工具、资源和提示模板

MCP 定义了三种"原语"（Primitives），它们是 Server 可以提供的三种不同类型的能力：

**工具（Tools）**：这是最常用的原语。工具是 Server 可以执行的操作，类似于函数调用。比如查询数据库、发送邮件、执行计算等。工具的调用是由模型决定的——模型分析用户请求后，决定是否需要调用某个工具。

**资源（Resources）**：资源是 Server 可以提供的数据。与工具不同，资源是只读的，它们是数据而不是操作。比如一个文件系统 Server 可以提供文件内容作为资源，一个数据库 Server 可以提供表结构信息作为资源。资源通常由应用程序控制，而不是由模型决定。

**提示模板（Prompts）**：提示模板是 Server 可以提供的预定义提示。它们可以被 Host 用来生成特定格式的请求。比如一个代码审查 Server 可以提供"审查代码"的提示模板，帮助 Host 生成合适的审查请求。

这三种原语各有用途，共同构成了 MCP 的能力体系。

### 传输协议：Server 如何与 Client 通信

MCP 定义了两种标准的传输协议：

**stdio（标准输入/输出）**：这是最简单的传输方式。MCP Server 作为一个子进程运行，Client 通过标准输入和标准输出与它通信。这种方式适合本地工具，比如文件系统操作、本地数据库查询等。

**HTTP + SSE（Server-Sent Events）**：这种方式适合远程 Server。Client 通过 HTTP POST 发送请求，Server 通过 SSE 推送响应。这种方式适合需要网络访问的工具。

选择哪种传输方式取决于你的使用场景。如果工具只需要访问本地资源，stdio 更简单高效；如果工具需要远程访问或者需要被多个 Client 共享，HTTP + SSE 更合适。

### MCP 的消息格式

MCP 使用 JSON-RPC 2.0 作为消息格式。这是一种轻量级的远程过程调用协议。主要的消息类型包括：

**请求（Request）**：Client 发送给 Server 的调用请求。
```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
        "name": "get_weather",
        "arguments": {"city": "北京"}
    }
}
```

**响应（Response）**：Server 返回给 Client 的结果。
```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [{"type": "text", "text": "北京：晴天，25°C"}]
    }
}
```

**通知（Notification）**：没有 id 的消息，不需要回复。用于状态更新等场景。

### MCP 与现有工具标准的对比

你可能会问：已经有了 OpenAI 的 Function Calling 和 Anthropic 的 Tool Use，为什么还需要 MCP？

关键区别在于：

1. **Function Calling/Tool Use 是 API 层面的标准**，它们定义了模型如何请求工具调用。而 MCP 是应用层面的标准，它定义了工具如何被发现和调用。

2. **Function Calling/Tool Use 是模型特定的**，OpenAI 和 Claude 的格式略有不同。而 MCP 是模型无关的，同一个 MCP Server 可以同时服务于 OpenAI 和 Claude。

3. **MCP 提供了工具发现机制**，Client 可以在运行时查询 Server 提供了哪些工具。而 Function Calling/Tool Use 需要你在代码中预先定义所有工具。

### MCP 的生态系统

MCP 的生态系统正在快速发展。一些主要的参与者包括：

**官方 Server**：Anthropic 提供了一些官方的 MCP Server 实现，比如文件系统、数据库、Web 搜索等。

**社区 Server**：开发者社区正在为各种服务创建 MCP Server，比如 GitHub、Slack、Google Drive 等。

**SDK 和框架**：有多个语言的 MCP SDK，包括 Python、TypeScript、Java 等。LangChain、LlamaIndex 等框架也在集成 MCP 支持。

**Host 应用**：Claude Desktop 是第一个原生支持 MCP 的应用。Cursor、Continue 等 IDE 也在集成 MCP 支持。

---

## 完整代码示例

### 示例 1：基础 MCP Server

```python
"""
基础 MCP Server 示例
实现一个简单的天气查询服务

环境要求：
    pip install mcp

运行方式：
    python weather_mcp_server.py
"""

import asyncio
import json
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import (
    Tool,
    TextContent,
    CallToolResult,
)


# 创建 Server 实例
server = Server("weather-service")


# 模拟天气数据
WEATHER_DATA = {
    "北京": {
        "temp": 25,
        "condition": "晴天",
        "humidity": 45,
        "wind": "北风 3 级",
        "aqi": 75,
        "forecast": [
            {"day": "明天", "temp": "22-28°C", "condition": "多云"},
            {"day": "后天", "temp": "20-26°C", "condition": "小雨"},
        ],
    },
    "上海": {
        "temp": 28,
        "condition": "多云",
        "humidity": 70,
        "wind": "东风 2 级",
        "aqi": 60,
        "forecast": [
            {"day": "明天", "temp": "26-30°C", "condition": "晴"},
            {"day": "后天", "temp": "24-29°C", "condition": "多云"},
        ],
    },
    "广州": {
        "temp": 32,
        "condition": "雷阵雨",
        "humidity": 85,
        "wind": "南风 1 级",
        "aqi": 45,
        "forecast": [
            {"day": "明天", "temp": "28-33°C", "condition": "阵雨"},
            {"day": "后天", "temp": "27-32°C", "condition": "多云"},
        ],
    },
}


@server.list_tools()
async def list_tools():
    """列出所有可用工具"""
    return [
        Tool(
            name="get_current_weather",
            description="获取指定城市的当前天气信息",
            inputSchema={
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "城市名称",
                    }
                },
                "required": ["city"],
            },
        ),
        Tool(
            name="get_weather_forecast",
            description="获取指定城市未来几天的天气预报",
            inputSchema={
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "城市名称",
                    },
                    "days": {
                        "type": "integer",
                        "description": "预报天数（1-3天）",
                    },
                },
                "required": ["city"],
            },
        ),
        Tool(
            name="compare_weather",
            description="比较两个城市的天气情况",
            inputSchema={
                "type": "object",
                "properties": {
                    "city1": {
                        "type": "string",
                        "description": "第一个城市",
                    },
                    "city2": {
                        "type": "string",
                        "description": "第二个城市",
                    },
                },
                "required": ["city1", "city2"],
            },
        ),
    ]


@server.call_tool()
async def call_tool(name: str, arguments: dict):
    """处理工具调用"""
    if name == "get_current_weather":
        city = arguments.get("city", "")
        if city in WEATHER_DATA:
            data = WEATHER_DATA[city]
            result = (
                f"📍 {city} 当前天气：\n"
                f"🌡️ 温度：{data['temp']}°C\n"
                f"☁️ 天气：{data['condition']}\n"
                f"💧 湿度：{data['humidity']}%\n"
                f"🌬️ 风力：{data['wind']}\n"
                f"🏭 空气质量指数：{data['aqi']}"
            )
        else:
            result = f"未找到 {city} 的天气信息"
        
        return CallToolResult(
            content=[TextContent(type="text", text=result)]
        )
    
    elif name == "get_weather_forecast":
        city = arguments.get("city", "")
        days = arguments.get("days", 2)
        
        if city in WEATHER_DATA:
            forecast = WEATHER_DATA[city]["forecast"][:days]
            result = f"📅 {city} 未来 {days} 天天气预报：\n\n"
            for f in forecast:
                result += f"{f['day']}：{f['condition']}，{f['temp']}\n"
        else:
            result = f"未找到 {city} 的天气预报"
        
        return CallToolResult(
            content=[TextContent(type="text", text=result)]
        )
    
    elif name == "compare_weather":
        city1 = arguments.get("city1", "")
        city2 = arguments.get("city2", "")
        
        if city1 in WEATHER_DATA and city2 in WEATHER_DATA:
            d1 = WEATHER_DATA[city1]
            d2 = WEATHER_DATA[city2]
            
            result = (
                f"🌤️ 天气对比：{city1} vs {city2}\n\n"
                f"{'指标':<8} {'城市':<6} {'城市':<6}\n"
                f"{'温度':<8} {d1['temp']:<6}°C {d2['temp']:<6}°C\n"
                f"{'天气':<8} {d1['condition']:<8} {d2['condition']:<8}\n"
                f"{'湿度':<8} {d1['humidity']:<6}% {d2['humidity']:<6}%\n"
            )
            
            # 建议
            if d1['temp'] > d2['temp']:
                result += f"\n💡 {city1} 比 {city2} 热 {d1['temp'] - d2['temp']}°C"
            else:
                result += f"\n💡 {city2} 比 {city1} 热 {d2['temp'] - d1['temp']}°C"
        else:
            missing = []
            if city1 not in WEATHER_DATA:
                missing.append(city1)
            if city2 not in WEATHER_DATA:
                missing.append(city2)
            result = f"未找到以下城市的天气信息：{', '.join(missing)}"
        
        return CallToolResult(
            content=[TextContent(type="text", text=result)]
        )
    
    return CallToolResult(
        content=[TextContent(type="text", text=f"未知工具: {name}")]
    )


async def main():
    """启动 MCP Server"""
    # 使用 stdio 传输
    async with stdio_server() as (read_stream, write_stream):
        await server.run(
            read_stream,
            write_stream,
            server.create_initialization_options(),
        )


if __name__ == "__main__":
    asyncio.run(main())
```

### 示例 2：MCP Client 连接和使用

```python
"""
MCP Client 示例
连接 MCP Server 并使用其工具

环境要求：
    pip install mcp anthropic
"""

import asyncio
import json
import os
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
import anthropic


class MCPClient:
    """MCP 客户端"""
    
    def __init__(self):
        self.llm_client = anthropic.Anthropic(
            api_key=os.environ.get("ANTHROPIC_API_KEY")
        )
        self.session = None
        self.tools = []
    
    async def connect(self, server_script_path: str):
        """连接到 MCP Server"""
        # 配置 stdio 传输
        server_params = StdioServerParameters(
            command="python",
            args=[server_script_path],
        )
        
        # 创建客户端会话
        self.transport = stdio_client(server_params)
        self.read, self.write = await self.transport.__aenter__()
        self.session = ClientSession(self.read, self.write)
        await self.session.__aenter__()
        
        # 初始化连接
        await self.session.initialize()
        
        # 获取可用工具列表
        await self._load_tools()
        
        print("成功连接到 MCP Server")
        print(f"可用工具: {[t['name'] for t in self.tools]}")
    
    async def _load_tools(self):
        """加载 Server 提供的工具"""
        tools_response = await self.session.list_tools()
        
        self.tools = []
        for tool in tools_response.tools:
            self.tools.append({
                "name": tool.name,
                "description": tool.description,
                "input_schema": tool.inputSchema,
            })
    
    def _format_tools_for_llm(self):
        """将 MCP 工具转换为 Claude 格式"""
        claude_tools = []
        for tool in self.tools:
            claude_tools.append({
                "name": tool["name"],
                "description": tool["description"],
                "input_schema": tool["input_schema"],
            })
        return claude_tools
    
    async def call_tool(self, tool_name: str, arguments: dict) -> str:
        """调用 MCP Server 的工具"""
        result = await self.session.call_tool(tool_name, arguments)
        
        # 提取文本内容
        text_parts = []
        for content in result.content:
            if content.type == "text":
                text_parts.append(content.text)
        
        return "\n".join(text_parts)
    
    async def chat(self, user_message: str) -> str:
        """与集成了 MCP 工具的 Claude 对话"""
        messages = [{"role": "user", "content": user_message}]
        claude_tools = self._format_tools_for_llm()
        
        while True:
            # 调用 Claude
            response = self.llm_client.messages.create(
                model="claude-sonnet-4-20250514",
                max_tokens=4096,
                tools=claude_tools,
                messages=messages,
            )
            
            # 检查是否需要工具调用
            if response.stop_reason != "tool_use":
                final_text = ""
                for block in response.content:
                    if block.type == "text":
                        final_text += block.text
                return final_text
            
            # 处理工具调用
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    print(f"  调用 MCP 工具: {block.name}({json.dumps(block.input, ensure_ascii=False)})")
                    
                    result = await self.call_tool(block.name, block.input)
                    print(f"  工具结果: {result[:100]}...")
                    
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result,
                    })
            
            messages.append({"role": "assistant", "content": response.content})
            messages.append({"role": "user", "content": tool_results})
    
    async def close(self):
        """关闭连接"""
        if self.session:
            await self.session.__aexit__(None, None, None)
        if self.transport:
            await self.transport.__aexit__(None, None, None)


async def main():
    """主函数"""
    print("=" * 60)
    print("MCP Client 示例")
    print("=" * 60)
    
    client = MCPClient()
    
    try:
        # 连接到天气服务 MCP Server
        await client.connect("weather_mcp_server.py")
        
        # 进行对话
        queries = [
            "北京现在天气怎么样？",
            "上海和广州哪个更热？",
            "明天北京天气会怎样？",
        ]
        
        for query in queries:
            print(f"\n用户: {query}")
            response = await client.chat(query)
            print(f"助手: {response}")
    
    finally:
        await client.close()


if __name__ == "__main__":
    asyncio.run(main())
```

### 示例 3：带资源的 MCP Server

```python
"""
带资源（Resources）的 MCP Server 示例
提供文件内容作为资源

环境要求：
    pip install mcp
"""

import asyncio
import os
from pathlib import Path
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import (
    Tool,
    Resource,
    TextContent,
    CallToolResult,
    TextResourceContents,
    BlobResourceContents,
)


server = Server("file-service")


# 模拟文件系统
VIRTUAL_FS = {
    "/docs/readme.txt": "这是一个示例文件。\n项目名称：MCP 示例\n版本：1.0",
    "/docs/config.json": '{"debug": true, "log_level": "info", "port": 8080}',
    "/data/sales.csv": "月份,销售额,成本\n1月,10000,6000\n2月,12000,7000\n3月,15000,8000",
}


@server.list_resources()
async def list_resources():
    """列出所有可用资源"""
    resources = []
    for path, content in VIRTUAL_FS.items():
        resources.append(
            Resource(
                uri=f"file://{path}",
                name=os.path.basename(path),
                description=f"虚拟文件: {path}",
                mimeType="text/plain" if path.endswith(".txt") else "application/json",
            )
        )
    return resources


@server.read_resource()
async def read_resource(uri: str) -> str:
    """读取资源内容"""
    if uri.startswith("file://"):
        path = uri[7:]  # 移除 "file://" 前缀
        if path in VIRTUAL_FS:
            return VIRTUAL_FS[path]
    return f"未找到资源: {uri}"


@server.list_tools()
async def list_tools():
    """列出所有工具"""
    return [
        Tool(
            name="list_files",
            description="列出虚拟文件系统中的所有文件",
            inputSchema={"type": "object", "properties": {}},
        ),
        Tool(
            name="read_file",
            description="读取指定路径的文件内容",
            inputSchema={
                "type": "object",
                "properties": {
                    "path": {
                        "type": "string",
                        "description": "文件路径",
                    }
                },
                "required": ["path"],
            },
        ),
        Tool(
            name="write_file",
            description="向虚拟文件系统写入内容",
            inputSchema={
                "type": "object",
                "properties": {
                    "path": {
                        "type": "string",
                        "description": "文件路径",
                    },
                    "content": {
                        "type": "string",
                        "description": "文件内容",
                    },
                },
                "required": ["path", "content"],
            },
        ),
        Tool(
            name="analyze_data",
            description="分析 CSV 格式的数据文件",
            inputSchema={
                "type": "object",
                "properties": {
                    "path": {
                        "type": "string",
                        "description": "CSV 文件路径",
                    }
                },
                "required": ["path"],
            },
        ),
    ]


@server.call_tool()
async def call_tool(name: str, arguments: dict):
    """处理工具调用"""
    if name == "list_files":
        files = list(VIRTUAL_FS.keys())
        result = "📁 文件列表：\n" + "\n".join(f"  {f}" for f in files)
        return CallToolResult(content=[TextContent(type="text", text=result)])
    
    elif name == "read_file":
        path = arguments.get("path", "")
        if path in VIRTUAL_FS:
            return CallToolResult(
                content=[TextContent(type="text", text=VIRTUAL_FS[path])]
            )
        return CallToolResult(
            content=[TextContent(type="text", text=f"文件不存在: {path}")]
        )
    
    elif name == "write_file":
        path = arguments.get("path", "")
        content = arguments.get("content", "")
        VIRTUAL_FS[path] = content
        return CallToolResult(
            content=[TextContent(type="text", text=f"文件已写入: {path}")]
        )
    
    elif name == "analyze_data":
        path = arguments.get("path", "")
        if path not in VIRTUAL_FS:
            return CallToolResult(
                content=[TextContent(type="text", text=f"文件不存在: {path}")]
            )
        
        data = VIRTUAL_FS[path]
        lines = data.strip().split("\n")
        
        if len(lines) < 2:
            return CallToolResult(
                content=[TextContent(type="text", text="数据不足，无法分析")]
            )
        
        headers = lines[0].split(",")
        rows = [line.split(",") for line in lines[1:]]
        
        # 简单统计
        result = f"📊 数据分析结果：\n\n"
        result += f"列名: {', '.join(headers)}\n"
        result += f"行数: {len(rows)}\n\n"
        
        for i, header in enumerate(headers):
            try:
                values = [float(row[i]) for row in rows if i < len(row)]
                if values:
                    result += f"{header}:\n"
                    result += f"  最小值: {min(values)}\n"
                    result += f"  最大值: {max(values)}\n"
                    result += f"  平均值: {sum(values)/len(values):.2f}\n"
                    result += f"  总和: {sum(values)}\n\n"
            except ValueError:
                result += f"{header}: (非数值列)\n\n"
        
        return CallToolResult(content=[TextContent(type="text", text=result)])
    
    return CallToolResult(
        content=[TextContent(type="text", text=f"未知工具: {name}")]
    )


async def main():
    """启动 Server"""
    async with stdio_server() as (read_stream, write_stream):
        await server.run(
            read_stream,
            write_stream,
            server.create_initialization_options(),
        )


if __name__ == "__main__":
    asyncio.run(main())
```

### 示例 4：带提示模板的 MCP Server

```python
"""
带提示模板（Prompts）的 MCP Server 示例

环境要求：
    pip install mcp
"""

import asyncio
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import (
    Tool,
    Prompt,
    PromptArgument,
    TextContent,
    CallToolResult,
    GetPromptResult,
)


server = Server("prompt-service")


@server.list_prompts()
async def list_prompts():
    """列出所有可用的提示模板"""
    return [
        Prompt(
            name="code_review",
            description="代码审查提示模板",
            arguments=[
                PromptArgument(
                    name="language",
                    description="编程语言",
                    required=True,
                ),
                PromptArgument(
                    name="focus",
                    description="审查重点（安全性/性能/可读性）",
                    required=False,
                ),
            ],
        ),
        Prompt(
            name="data_analysis",
            description="数据分析报告模板",
            arguments=[
                PromptArgument(
                    name="data_type",
                    description="数据类型（sales/user/log）",
                    required=True,
                ),
                PromptArgument(
                    name="time_range",
                    description="时间范围（如：最近一周、2024年Q1）",
                    required=False,
                ),
            ],
        ),
        Prompt(
            name="api_documentation",
            description="API 文档生成模板",
            arguments=[
                PromptArgument(
                    name="endpoint",
                    description="API 端点路径",
                    required=True,
                ),
                PromptArgument(
                    name="method",
                    description="HTTP 方法（GET/POST/PUT/DELETE）",
                    required=True,
                ),
            ],
        ),
    ]


@server.get_prompt()
async def get_prompt(name: str, arguments: dict):
    """获取提示模板"""
    if name == "code_review":
        language = arguments.get("language", "Python")
        focus = arguments.get("focus", "全面审查")
        
        prompt_text = f"""请对以下 {language} 代码进行全面审查。

审查重点：{focus}

请从以下方面进行审查：
1. 代码质量和可读性
2. 潜在的 bug 和错误
3. 性能问题
4. 安全漏洞
5. 最佳实践和设计模式

请提供具体的改进建议和代码示例。"""
        
        return GetPromptResult(
            description=f"{language} 代码审查",
            messages=[
                {"role": "user", "content": prompt_text}
            ],
        )
    
    elif name == "data_analysis":
        data_type = arguments.get("data_type", "sales")
        time_range = arguments.get("time_range", "最近一个月")
        
        type_descriptions = {
            "sales": "销售数据",
            "user": "用户行为数据",
            "log": "系统日志数据",
        }
        
        desc = type_descriptions.get(data_type, "通用数据")
        
        prompt_text = f"""请分析以下{desc}（时间范围：{time_range}）。

分析要求：
1. 数据概览：数据规模、字段说明、数据质量
2. 趋势分析：关键指标的变化趋势
3. 异常检测：识别异常数据点
4. 相关性分析：找出变量之间的关联
5. 业务洞察：基于数据的业务建议

请使用图表和数据来支持你的分析。"""
        
        return GetPromptResult(
            description=f"{desc}分析报告",
            messages=[
                {"role": "user", "content": prompt_text}
            ],
        )
    
    elif name == "api_documentation":
        endpoint = arguments.get("endpoint", "/api/resource")
        method = arguments.get("method", "GET").upper()
        
        prompt_text = f"""请为以下 API 端点生成详细的文档：

端点：{method} {endpoint}

文档要求：
1. 端点描述
2. 请求参数说明
3. 响应格式说明
4. 错误码说明
5. 使用示例（curl / Python）
6. 注意事项

请使用 Markdown 格式输出。"""
        
        return GetPromptResult(
            description=f"{method} {endpoint} API 文档",
            messages=[
                {"role": "user", "content": prompt_text}
            ],
        )
    
    return GetPromptResult(
        description="未知提示模板",
        messages=[{"role": "user", "content": f"未知的提示模板: {name}"}],
    )


@server.list_tools()
async def list_tools():
    """列出工具（此 Server 也提供一些工具）"""
    return [
        Tool(
            name="format_code",
            description="格式化代码",
            inputSchema={
                "type": "object",
                "properties": {
                    "code": {"type": "string", "description": "要格式化的代码"},
                    "language": {"type": "string", "description": "编程语言"},
                },
                "required": ["code", "language"],
            },
        ),
    ]


@server.call_tool()
async def call_tool(name: str, arguments: dict):
    """处理工具调用"""
    if name == "format_code":
        code = arguments.get("code", "")
        language = arguments.get("language", "python")
        
        # 简单的格式化处理
        formatted = code.strip()
        
        return CallToolResult(
            content=[TextContent(type="text", text=f"格式化后的 {language} 代码：\n\n{formatted}")]
        )
    
    return CallToolResult(
        content=[TextContent(type="text", text=f"未知工具: {name}")]
    )


async def main():
    """启动 Server"""
    async with stdio_server() as (read_stream, write_stream):
        await server.run(
            read_stream,
            write_stream,
            server.create_initialization_options(),
        )


if __name__ == "__main__":
    asyncio.run(main())
```

### 示例 5：SSE 传输的 MCP Server

```python
"""
使用 SSE（Server-Sent Events）传输的 MCP Server 示例
适合远程访问场景

环境要求：
    pip install mcp fastapi uvicorn
"""

import asyncio
import json
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from mcp.server import Server
from mcp.server.sse import SseServerTransport
from mcp.types import (
    Tool,
    TextContent,
    CallToolResult,
)
import uvicorn


app = FastAPI(title="MCP SSE Server")
server = Server("sse-weather-service")
sse_server = SseServerTransport("/messages/")


# 天气数据
WEATHER_DATA = {
    "北京": {"temp": 25, "condition": "晴天", "humidity": 45},
    "上海": {"temp": 28, "condition": "多云", "humidity": 70},
    "广州": {"temp": 32, "condition": "雷阵雨", "humidity": 85},
}


@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="get_weather",
            description="获取城市天气",
            inputSchema={
                "type": "object",
                "properties": {
                    "city": {"type": "string", "description": "城市名称"}
                },
                "required": ["city"],
            },
        ),
    ]


@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "get_weather":
        city = arguments.get("city", "")
        if city in WEATHER_DATA:
            data = WEATHER_DATA[city]
            result = f"{city}: {data['condition']}, {data['temp']}°C, 湿度 {data['humidity']}%"
        else:
            result = f"未找到 {city} 的天气信息"
        
        return CallToolResult(content=[TextContent(type="text", text=result)])
    
    return CallToolResult(content=[TextContent(type="text", text=f"未知工具: {name}")])


@app.get("/sse")
async def sse_endpoint(request: Request):
    """SSE 端点"""
    async def event_generator():
        async with sse_server.connect_sse(
            request.scope, request.receive, request._send
        ) as streams:
            await server.run(
                streams[0],
                streams[1],
                server.create_initialization_options(),
            )
    
    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
    )


@app.post("/messages/")
async def messages_endpoint(request: Request):
    """消息端点"""
    await sse_server.handle_post_message(request.scope, request.receive, request._send)
    return {"status": "ok"}


if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## 案例分析

### 案例 1：构建一个完整的 MCP 工具生态系统

让我们分析如何构建一个为开发者服务的 MCP 工具生态系统：

**工具层（MCP Server）：**

1. **代码工具 Server**：提供代码格式化、语法检查、测试运行等功能
2. **文档工具 Server**：提供文档搜索、API 查询、代码解释等功能
3. **部署工具 Server**：提供服务器管理、日志查看、监控告警等功能
4. **协作工具 Server**：提供代码审查、问题跟踪、团队沟通等功能

**宿主层（MCP Host）：**

1. **IDE 集成**：在 IDE 中嵌入 MCP Client，让开发者在编写代码时可以随时调用工具
2. **CLI 工具**：命令行界面，支持快速调用各种 MCP 工具
3. **Web 界面**：基于 Web 的管理界面，可视化管理各种 MCP Server

**关键设计决策：**

1. **工具粒度**：每个 MCP Server 应该专注于一个领域，提供 3-10 个相关工具
2. **错误处理**：工具执行失败时，应该返回有用的错误信息，而不是简单的"失败"
3. **安全性**：敏感操作（如删除文件、修改配置）需要额外的权限控制
4. **性能**：对于耗时操作，应该实现异步执行和进度通知

### 案例 2：MCP 在企业环境中的应用

**场景**：一家大型企业需要为员工提供各种内部工具的 AI 访问能力。

**架构设计：**

```
员工 → Claude Desktop / IDE 插件
            ↓
       MCP Client
            ↓
    ┌───────┼───────┐
    ↓       ↓       ↓
 代码审查  文档查询  部署管理
  Server   Server   Server
    ↓       ↓       ↓
  GitLab   Confluence  K8s API
```

**安全考虑：**

1. **认证**：每个 MCP Server 需要验证请求者的身份
2. **授权**：不同角色的员工可以使用不同的工具
3. **审计**：所有工具调用都需要记录日志
4. **隔离**：MCP Server 运行在隔离的环境中，防止数据泄露

---

## 常见坑

### 坑 1：异步处理不当

MCP Server 的所有方法都是异步的，但很多初学者忘记使用 async/await。

```python
# 错误写法
@server.call_tool()
def call_tool(name, arguments):
    result = some_sync_function()
    return result

# 正确写法
@server.call_tool()
async def call_tool(name: str, arguments: dict):
    result = await some_async_function()
    return result
```

### 坑 2：工具定义不清晰

工具的描述和参数 schema 定义不清晰，导致模型不知道何时使用、如何使用。

```python
# 不好的描述
Tool(
    name="process",
    description="处理数据",
    inputSchema={...}
)

# 好的描述
Tool(
    name="analyze_sales_data",
    description="分析销售数据，计算关键指标（销售额、利润、增长率），生成分析报告",
    inputSchema={
        "type": "object",
        "properties": {
            "data": {"type": "string", "description": "CSV 格式的销售数据"},
            "metrics": {
                "type": "array",
                "items": {"type": "string"},
                "description": "要计算的指标列表"
            },
        },
        "required": ["data"],
    }
)
```

### 坑 3：资源 URI 设计不当

资源 URI 应该是唯一的、可预测的，便于客户端发现和缓存。

```python
# 不好的 URI 设计
Resource(uri="file1", name="第一个文件", ...)
Resource(uri="file2", name="第二个文件", ...)

# 好的 URI 设计
Resource(uri="file:///docs/readme.txt", name="readme.txt", ...)
Resource(uri="file:///docs/config.json", name="config.json", ...)
```

### 坑 4：错误处理不完善

MCP Server 应该返回有意义的错误信息，而不是抛出异常。

```python
# 不好的错误处理
@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name not in valid_tools:
        raise ValueError(f"未知工具: {name}")  # 不好！
    return result

# 好的错误处理
@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name not in valid_tools:
        return CallToolResult(
            content=[TextContent(type="text", text=f"未知工具: {name}")],
            isError=True,
        )
    return result
```

### 坑 5：忽略传输层的选择

stdio 适合本地工具，HTTP + SSE 适合远程工具。选择不当会导致性能问题或部署困难。

---

## 练习题

### 练习 1：基础 MCP Server
实现一个简单的数学计算 MCP Server，提供以下工具：
- add：加法
- subtract：减法
- multiply：乘法
- divide：除法
- calculate：计算数学表达式

### 练习 2：带资源的 MCP Server
扩展练习 1 的数学计算 Server，添加以下资源：
- 常用数学常数（pi, e 等）
- 常用数学公式
- 计算历史记录

### 练习 3：MCP Client
编写一个 MCP Client，能够：
- 连接到任意 MCP Server
- 发现并列出可用工具
- 使用自然语言调用工具
- 处理工具调用错误

### 练习 4：提示模板 Server
实现一个代码助手 MCP Server，提供以下提示模板：
- code_review：代码审查
- refactor：代码重构建议
- documentation：文档生成
- testing：测试用例生成

### 练习 5：SSE Server
将练习 1 的数学计算 Server 改为使用 SSE 传输，使其可以通过 HTTP 访问。

### 练习 6：完整的 MCP 生态
构建一个包含以下组件的完整 MCP 生态：
- 3 个不同领域的 MCP Server
- 1 个支持多个 Server 的 MCP Client
- 与 Claude 集成的完整对话流程

---

## 实战任务

### 任务 1：构建一个开发者工具包 MCP Server（中等难度）

**功能需求：**
1. 代码格式化工具（支持 Python、JavaScript）
2. 代码统计工具（行数、复杂度分析）
3. 依赖检查工具
4. 文档生成工具

**技术要求：**
1. 使用 MCP 协议实现
2. 支持 stdio 传输
3. 提供清晰的工具定义
4. 实现完善的错误处理

### 任务 2：构建一个数据分析平台 MCP Server（高难度）

**功能需求：**
1. 支持多种数据格式（CSV、JSON、Excel）
2. 自动数据分析和可视化
3. 自然语言查询数据
4. 生成分析报告

**技术要求：**
1. 使用 MCP 协议实现
2. 支持 SSE 传输
3. 实现资源和提示模板
4. 添加用户认证和权限控制

---

## 本章小结

本章我们深入学习了 MCP（Model Context Protocol）协议，这是 Agent 生态系统标准化的重要一步。我们从以下几个方面进行了探索：

**设计理念**：MCP 通过在 Agent 和工具之间引入标准协议层，解决了工具碎片化问题。开发者只需要按照 MCP 标准实现一次工具，就可以被所有支持 MCP 的 Agent 使用。

**核心架构**：理解了 MCP Host、Client、Server 三种角色的关系。Host 是应用程序，Client 负责与 Server 通信，Server 提供具体的能力。

**三大原语**：掌握了工具、资源和提示模板三种原语的使用方式。工具是可执行的操作，资源是只读的数据，提示模板是预定义的请求格式。

**传输协议**：了解了 stdio 和 HTTP + SSE 两种传输方式的区别和适用场景。stdio 适合本地工具，HTTP + SSE 适合远程工具。

**实际应用**：通过完整的代码示例和案例分析，看到了 MCP 在实际应用中的价值。

MCP 代表了 Agent 工具标准化的一个重要方向。虽然它还在快速发展中，但它已经展现出了巨大的潜力。随着越来越多的工具和服务采用 MCP 标准，Agent 生态系统的碎片化问题将得到显著改善。

在下一章中，我们将学习自学习 Agent，看看 Agent 如何从经验中学习和改进。

---

## 延伸阅读

1. MCP 官方文档：https://modelcontextprotocol.io/
2. MCP 规范：https://spec.modelcontextprotocol.io/
3. MCP Python SDK：https://github.com/modelcontextprotocol/python-sdk
4. Anthropic MCP 博客：https://www.anthropic.com/news/model-context-protocol
5. MCP Server 示例集合：https://github.com/modelcontextprotocol/servers
