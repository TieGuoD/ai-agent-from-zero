# 第46章：Anthropic Claude 工具使用 —— Claude 的 Agent 能力

## 学习目标

通过本章的学习，你将能够：
1. 理解 Claude 工具使用的架构设计和核心理念
2. 掌握使用 Anthropic Python SDK 定义和调用工具
3. 学会构建基于 Claude 的 Agent 系统
4. 理解 Claude 与 OpenAI 工具调用的异同点
5. 掌握流式工具调用的实现方式
6. 能够实现复杂的多工具协作 Agent

## 核心问题

在上一章中，我们学习了 OpenAI 的 Assistants API，它提供了一种托管的 Agent 方案。但并非所有场景都适合使用托管服务。有时候我们需要更多的控制权——自己管理状态、自己处理工具调用、自己决定 Agent 的行为循环。

Anthropic 的 Claude 提供了另一种思路：它不提供托管的 Agent 服务，而是提供了一套强大的工具使用（Tool Use）能力。你可以用这套能力来构建任何你想要的 Agent 架构。这就像是给你一套完整的厨房工具，而不是直接给你一顿饭——你有更大的自由度，但也需要自己动手。

那么，Claude 的工具使用能力到底有多强？它能帮我们构建什么样的 Agent？和 OpenAI 相比有什么不同？

---

## 原理讲解

### Claude 工具使用的基本原理

Claude 的工具使用建立在一个简单而强大的概念之上：模型可以决定何时需要调用外部工具，以及如何调用这些工具。

当你在请求中定义了可用工具后，Claude 在处理用户请求时可能会做出这样的判断："这个问题我需要调用某个工具才能回答。" 于是它不会直接生成回复，而是返回一个特殊的响应，告诉你的代码："请帮我调用这个工具，参数是这些。"

你的代码执行完工具后，把结果再发给 Claude，Claude 就可以基于工具返回的结果生成最终回复了。

整个过程可以用一个流程图来表示：

```
用户请求 + 工具定义 → Claude 分析
                          ↓
                   需要工具？→ 返回 tool_use 内容块
                          ↓              ↓
                   不需要 →          你的代码执行工具
                   直接回复              ↓
                                 返回 tool_result → Claude 生成最终回复
```

这种设计有几个优点。首先，它把工具调用的决定权交给了模型本身——模型知道什么时候需要查天气、什么时候需要搜索数据库、什么时候需要执行代码。其次，它把工具执行的控制权交给了你的代码——你可以完全控制工具如何执行、是否执行、以及如何处理错误。

### Claude 工具使用的独特之处

虽然 Claude 和 OpenAI 都支持工具调用，但它们在设计理念上有一些重要的区别。

**多轮工具调用**：Claude 支持在一次响应中返回多个工具调用请求。这意味着如果 Claude 需要同时查询天气和航班信息，它可以在一次响应中同时请求这两个调用，而不是先查一个再查另一个。这对于提升效率非常有帮助。

**工具使用预算**：Claude 引入了一个叫做"预算"的概念。你可以设置一个工具使用的最大轮次（比如 10 轮），防止 Agent 陷入无限的工具调用循环。这是一个非常实用的特性，因为在实际应用中，工具调用循环的失控是一个常见的问题。

**扩展思考与工具使用的结合**：Claude 支持在使用扩展思考（Extended Thinking）的同时使用工具。这意味着 Claude 可以先进行深度推理，然后再决定调用哪些工具。这对于需要复杂推理的任务特别有用。

### 工具定义的结构

Claude 的工具定义相对简洁。每个工具需要包含：
- 工具名称（name）
- 工具描述（description）
- 输入参数的 JSON Schema（input_schema）

```python
{
    "name": "get_weather",
    "description": "获取指定城市的当前天气信息",
    "input_schema": {
        "type": "object",
        "properties": {
            "city": {
                "type": "string",
                "description": "城市名称"
            }
        },
        "required": ["city"]
    }
}
```

这个定义会告诉 Claude：有一个叫 `get_weather` 的工具，它接受一个 `city` 参数，用来获取天气信息。Claude 会根据这个定义来决定何时以及如何使用这个工具。

### 消息格式与工具调用

Claude 的消息格式和 OpenAI 有所不同。在 Claude 的消息中，工具调用是作为 content block 出现的：

```python
# Claude 的响应可能包含多个内容块
response = [
    {"type": "text", "text": "让我查询一下天气..."},
    {
        "type": "tool_use",
        "id": "toolu_01...",
        "name": "get_weather",
        "input": {"city": "北京"}
    }
]
```

当你的代码执行完工具后，需要以特定的格式返回结果：

```python
{
    "role": "user",
    "content": [
        {
            "type": "tool_result",
            "tool_use_id": "toolu_01...",
            "content": "北京：晴，25°C"
        }
    ]
}
```

### 流式工具调用

Claude 支持在流式模式下使用工具。当使用流式输出时，工具调用的信息会通过增量事件发送。这意味着你可以实时地看到 Claude 正在调用哪些工具，而不需要等到整个响应完成。

流式工具调用的事件序列大致如下：
1. `message_start`：消息开始
2. `content_block_start`：内容块开始（可能是文本或工具调用）
3. `content_block_delta`：内容块的增量数据
4. `content_block_stop`：内容块结束
5. `message_delta`：消息级别的增量信息
6. `message_stop`：消息结束

### 错误处理与重试

在工具调用过程中，错误处理是一个关键问题。Claude 提供了几种处理工具错误的方式：

1. **在 tool_result 中返回错误信息**：你可以把错误信息作为工具结果返回给 Claude，Claude 会根据错误信息决定下一步行动
2. **使用 is_error 标记**：tool_result 中可以设置 `is_error: true`，告诉 Claude 这个工具调用失败了
3. **重试机制**：你可以实现自己的重试逻辑，在返回错误之前先重试几次

### 安全考虑

在使用 Claude 的工具时，有几个安全方面需要注意：

1. **参数验证**：永远不要直接把 Claude 生成的参数传递给你的函数。你需要验证参数的类型和范围
2. **权限控制**：不是所有工具都应该无条件执行。某些敏感操作需要用户确认
3. **输入过滤**：对工具的输入进行过滤，防止注入攻击
4. **输出检查**：对工具的输出进行检查，确保不会泄露敏感信息

---

## 完整代码示例

### 示例 1：基础工具调用

```python
"""
Anthropic Claude 基础工具调用示例

环境要求：
    pip install anthropic

环境变量：
    ANTHROPIC_API_KEY: Anthropic API 密钥
"""

import os
import json
import anthropic


def basic_tool_calling():
    """基础工具调用示例：查询天气"""
    print("=" * 60)
    print("Claude 基础工具调用示例")
    print("=" * 60)
    
    client = anthropic.Anthropic(
        api_key=os.environ.get("ANTHROPIC_API_KEY")
    )
    
    # 定义工具
    tools = [
        {
            "name": "get_weather",
            "description": "获取指定城市的当前天气信息。当用户询问天气相关问题时使用此工具。",
            "input_schema": {
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "城市名称，例如：北京、上海、广州",
                    }
                },
                "required": ["city"],
            },
        }
    ]
    
    # 工具的实际实现
    def get_weather(city: str) -> str:
        """模拟天气查询"""
        weather_data = {
            "北京": "晴天，温度 25°C，湿度 45%，北风 3 级",
            "上海": "多云，温度 28°C，湿度 70%，东风 2 级",
            "广州": "阴天，温度 32°C，湿度 85%，南风 1 级",
        }
        return weather_data.get(city, f"无法获取 {city} 的天气信息")
    
    # 发送消息
    messages = [
        {"role": "user", "content": "北京今天天气怎么样？"}
    ]
    
    print("\n用户: 北京今天天气怎么样？\n")
    
    # 第一次调用：Claude 可能决定使用工具
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        tools=tools,
        messages=messages,
    )
    
    print(f"响应停止原因: {response.stop_reason}")
    print(f"使用 tokens: {response.usage.input_tokens} (输入) + {response.usage.output_tokens} (输出)")
    
    # 检查是否需要工具调用
    if response.stop_reason == "tool_use":
        print("\nClaude 需要调用工具...")
        
        # 提取工具调用信息
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                print(f"  工具: {block.name}")
                print(f"  参数: {json.dumps(block.input, ensure_ascii=False)}")
                
                # 执行工具
                if block.name == "get_weather":
                    result = get_weather(**block.input)
                    print(f"  结果: {result}")
                    
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result,
                    })
        
        # 将 Claude 的响应和工具结果添加到消息历史
        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": tool_results})
        
        # 第二次调用：Claude 基于工具结果生成最终回复
        final_response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            tools=tools,
            messages=messages,
        )
        
        # 输出最终回复
        for block in final_response.content:
            if block.type == "text":
                print(f"\n助手回复:\n{block.text}")
    else:
        # 不需要工具调用，直接输出回复
        for block in response.content:
            if block.type == "text":
                print(f"\n助手回复:\n{block.text}")


if __name__ == "__main__":
    basic_tool_calling()
```

### 示例 2：多工具协作 Agent

```python
"""
Claude 多工具协作 Agent 示例
一个能够查询天气、搜索信息、做计算的多功能助手
"""

import os
import json
import math
from datetime import datetime
import anthropic


def calculator(expression: str) -> str:
    """安全的数学计算器"""
    allowed_names = {
        "abs": abs, "round": round,
        "sin": math.sin, "cos": math.cos, "tan": math.tan,
        "sqrt": math.sqrt, "log": math.log, "log10": math.log10,
        "pi": math.pi, "e": math.e,
        "pow": pow, "min": min, "max": max,
    }
    
    try:
        # 使用受限的 eval
        result = eval(expression, {"__builtins__": {}}, allowed_names)
        return f"计算结果: {result}"
    except Exception as e:
        return f"计算错误: {str(e)}"


def get_current_time(timezone: str = "Asia/Shanghai") -> str:
    """获取当前时间"""
    now = datetime.now()
    return f"当前时间: {now.strftime('%Y-%m-%d %H:%M:%S')} (时区: {timezone})"


def search_knowledge(query: str) -> str:
    """模拟知识库搜索"""
    knowledge_base = {
        "Python": "Python 是一种高级编程语言，由 Guido van Rossum 于 1991 年发布。它以简洁易读的语法著称，广泛应用于 Web 开发、数据科学、人工智能等领域。",
        "机器学习": "机器学习是人工智能的一个分支，它使计算机系统能够从数据中学习和改进，而无需明确编程。常见的机器学习方法包括监督学习、无监督学习和强化学习。",
        "深度学习": "深度学习是机器学习的一个子领域，使用多层神经网络来学习数据的表示。它在图像识别、自然语言处理、语音识别等领域取得了突破性进展。",
        "Transformer": "Transformer 是一种基于自注意力机制的神经网络架构，由 Google 在 2017 年提出。它是当前大多数大型语言模型（如 GPT、Claude）的基础架构。",
    }
    
    # 简单的关键词匹配搜索
    results = []
    for key, value in knowledge_base.items():
        if key.lower() in query.lower() or query.lower() in key.lower():
            results.append(value)
    
    if results:
        return "\n\n".join(results)
    return f"未找到与 '{query}' 相关的信息"


def get_stock_price(symbol: str) -> str:
    """模拟股票价格查询"""
    stock_data = {
        "AAPL": {"name": "苹果公司", "price": 185.50, "change": 2.3},
        "GOOGL": {"name": "谷歌", "price": 140.25, "change": -1.5},
        "MSFT": {"name": "微软", "price": 378.80, "change": 1.8},
    }
    
    if symbol.upper() in stock_data:
        data = stock_data[symbol.upper()]
        change_emoji = "↑" if data["change"] > 0 else "↓"
        return (
            f"{data['name']} ({symbol.upper()}): "
            f"${data['price']:.2f} "
            f"{change_emoji} {abs(data['change']):.1f}%"
        )
    return f"未找到股票 {symbol} 的信息"


class MultiToolAgent:
    """多工具协作 Agent"""
    
    def __init__(self):
        self.client = anthropic.Anthropic(
            api_key=os.environ.get("ANTHROPIC_API_KEY")
        )
        
        # 定义工具
        self.tools = [
            {
                "name": "calculator",
                "description": "执行数学计算。支持基本运算和常用数学函数（sin, cos, sqrt, log 等）。",
                "input_schema": {
                    "type": "object",
                    "properties": {
                        "expression": {
                            "type": "string",
                            "description": "数学表达式，例如 '2 + 3 * 4' 或 'sqrt(16)'",
                        }
                    },
                    "required": ["expression"],
                },
            },
            {
                "name": "get_current_time",
                "description": "获取当前日期和时间",
                "input_schema": {
                    "type": "object",
                    "properties": {
                        "timezone": {
                            "type": "string",
                            "description": "时区，默认为 Asia/Shanghai",
                        }
                    },
                },
            },
            {
                "name": "search_knowledge",
                "description": "搜索知识库中的信息。当用户询问概念、定义或事实性问题时使用。",
                "input_schema": {
                    "type": "object",
                    "properties": {
                        "query": {
                            "type": "string",
                            "description": "搜索关键词",
                        }
                    },
                    "required": ["query"],
                },
            },
            {
                "name": "get_stock_price",
                "description": "获取股票的当前价格和涨跌幅。支持的股票代码：AAPL, GOOGL, MSFT。",
                "input_schema": {
                    "type": "object",
                    "properties": {
                        "symbol": {
                            "type": "string",
                            "description": "股票代码，例如 AAPL",
                        }
                    },
                    "required": ["symbol"],
                },
            },
        ]
        
        # 工具映射
        self.tool_functions = {
            "calculator": calculator,
            "get_current_time": get_current_time,
            "search_knowledge": search_knowledge,
            "get_stock_price": get_stock_price,
        }
    
    def execute_tool(self, tool_name: str, tool_input: dict) -> str:
        """安全执行工具调用"""
        if tool_name not in self.tool_functions:
            return f"错误: 未知工具 '{tool_name}'"
        
        try:
            return self.tool_functions[tool_name](**tool_input)
        except Exception as e:
            return f"工具执行错误: {str(e)}"
    
    def chat(self, user_message: str, max_tool_rounds: int = 10) -> str:
        """
        与 Agent 对话
        
        参数:
            user_message: 用户消息
            max_tool_rounds: 最大工具调用轮次（防止无限循环）
        """
        messages = [{"role": "user", "content": user_message}]
        
        for round_num in range(max_tool_rounds):
            # 调用 Claude
            response = self.client.messages.create(
                model="claude-sonnet-4-20250514",
                max_tokens=4096,
                tools=self.tools,
                messages=messages,
            )
            
            # 检查是否需要工具调用
            if response.stop_reason != "tool_use":
                # 不需要工具调用，提取最终回复
                final_text = ""
                for block in response.content:
                    if block.type == "text":
                        final_text += block.text
                return final_text
            
            # 处理工具调用
            print(f"  [第 {round_num + 1} 轮工具调用]")
            
            # 收集工具结果
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    print(f"    调用: {block.name}({json.dumps(block.input, ensure_ascii=False)})")
                    
                    result = self.execute_tool(block.name, block.input)
                    print(f"    结果: {result[:100]}...")
                    
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result,
                    })
            
            # 更新消息历史
            messages.append({"role": "assistant", "content": response.content})
            messages.append({"role": "user", "content": tool_results})
        
        return "达到最大工具调用轮次，对话终止"


def multi_tool_example():
    """多工具协作示例"""
    print("=" * 60)
    print("Claude 多工具协作 Agent")
    print("=" * 60)
    
    agent = MultiToolAgent()
    
    # 测试不同类型的查询
    queries = [
        "现在几点了？",
        "请帮我计算 sqrt(144) + log10(100) 的值",
        "什么是 Transformer 架构？",
        "苹果公司的股价怎么样？",
        "帮我算一下，如果我投资 10000 元，年利率 5%，5 年后本息合计多少？"
        "（复利公式：本金 * (1 + 利率)^年数）",
    ]
    
    for query in queries:
        print(f"\n{'=' * 40}")
        print(f"用户: {query}")
        print(f"{'=' * 40}")
        
        response = agent.chat(query)
        print(f"\n助手: {response}\n")


if __name__ == "__main__":
    multi_tool_example()
```

### 示例 3：流式工具调用

```python
"""
Claude 流式工具调用示例
实时展示工具调用的过程
"""

import os
import json
import anthropic


def streaming_tool_calling():
    """流式工具调用示例"""
    print("=" * 60)
    print("Claude 流式工具调用")
    print("=" * 60)
    
    client = anthropic.Anthropic(
        api_key=os.environ.get("ANTHROPIC_API_KEY")
    )
    
    # 定义工具
    tools = [
        {
            "name": "web_search",
            "description": "搜索互联网获取最新信息",
            "input_schema": {
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "搜索查询词",
                    }
                },
                "required": ["query"],
            },
        },
        {
            "name": "calculate",
            "description": "执行数学计算",
            "input_schema": {
                "type": "object",
                "properties": {
                    "expression": {
                        "type": "string",
                        "description": "数学表达式",
                    }
                },
                "required": ["expression"],
            },
        },
    ]
    
    messages = [
        {"role": "user", "content": "请搜索一下最新的 AI 发展趋势，然后计算一下如果一个 AI 模型的参数量是 70B，用 8 位量化后大小是多少 GB？"}
    ]
    
    # 使用流式输出
    print("\n用户: 请搜索一下最新的 AI 发展趋势，然后计算一下...")
    print("\nClaude 处理中...\n")
    
    current_tool_name = None
    
    with client.messages.stream(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        tools=tools,
        messages=messages,
    ) as stream:
        for event in stream:
            if event.type == "content_block_start":
                if event.content_block.type == "tool_use":
                    current_tool_name = event.content_block.name
                    print(f"\n[工具调用] {current_tool_name}")
                    print(f"  参数: ", end="")
            elif event.type == "content_block_delta":
                if event.delta.type == "input_json_delta":
                    print(event.delta.partial_json, end="")
                elif event.delta.type == "text_delta":
                    print(event.delta.text, end="", flush=True)
            elif event.type == "content_block_stop":
                if current_tool_name:
                    print()  # 换行
                    current_tool_name = None
    
    print("\n")


if __name__ == "__main__":
    streaming_tool_calling()
```

### 示例 4：具有记忆的 Agent

```python
"""
具有记忆能力的 Claude Agent
支持长期记忆和上下文管理
"""

import os
import json
from datetime import datetime
from typing import List, Dict, Optional
import anthropic


class MemoryManager:
    """记忆管理器"""
    
    def __init__(self, max_short_term: int = 20, max_long_term: int = 100):
        self.short_term: List[Dict] = []  # 短期记忆（最近的对话）
        self.long_term: List[Dict] = []   # 长期记忆（重要信息）
        self.max_short_term = max_short_term
        self.max_long_term = max_long_term
    
    def add_short_term(self, role: str, content: str):
        """添加短期记忆"""
        self.short_term.append({
            "role": role,
            "content": content,
            "timestamp": datetime.now().isoformat(),
        })
        
        # 超过限制时，将旧记忆移入长期记忆
        if len(self.short_term) > self.max_short_term:
            old_memory = self.short_term.pop(0)
            self.long_term.append(old_memory)
            
            # 长期记忆也有限制
            if len(self.long_term) > self.max_long_term:
                self.long_term.pop(0)
    
    def get_context(self, include_long_term: bool = True) -> List[Dict]:
        """获取对话上下文"""
        context = []
        
        if include_long_term and self.long_term:
            # 将长期记忆压缩为摘要
            summary = "之前的对话摘要：\n"
            for memory in self.long_term[-5:]:  # 只取最近 5 条
                summary += f"- {memory['role']}: {memory['content'][:100]}...\n"
            context.append({"role": "user", "content": summary})
        
        # 添加短期记忆
        for memory in self.short_term:
            context.append({
                "role": memory["role"],
                "content": memory["content"],
            })
        
        return context
    
    def save_important_fact(self, fact: str):
        """保存重要事实到长期记忆"""
        self.long_term.append({
            "role": "system",
            "content": f"[重要事实] {fact}",
            "timestamp": datetime.now().isoformat(),
            "type": "fact",
        })


class MemoryAgent:
    """具有记忆能力的 Agent"""
    
    def __init__(self):
        self.client = anthropic.Anthropic(
            api_key=os.environ.get("ANTHROPIC_API_KEY")
        )
        self.memory = MemoryManager()
        self.user_name: Optional[str] = None
        
        # 定义工具
        self.tools = [
            {
                "name": "remember_fact",
                "description": "记住一个重要的事实或用户偏好。当用户提供个人信息或表达偏好时使用。",
                "input_schema": {
                    "type": "object",
                    "properties": {
                        "fact": {
                            "type": "string",
                            "description": "要记住的事实",
                        }
                    },
                    "required": ["fact"],
                },
            },
            {
                "name": "recall_facts",
                "description": "回忆之前记住的事实。当需要参考之前的对话内容时使用。",
                "input_schema": {
                    "type": "object",
                    "properties": {},
                },
            },
        ]
    
    def chat(self, user_message: str) -> str:
        """与 Agent 对话"""
        # 添加用户消息到记忆
        self.memory.add_short_term("user", user_message)
        
        # 获取上下文
        messages = self.memory.get_context()
        
        # 调用 Claude
        response = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            tools=self.tools,
            system="你是一个友好的助手。你可以记住用户告诉你的信息，并在之后的对话中使用这些信息。如果用户告诉你重要的事实，请使用 remember_fact 工具保存。",
            messages=messages,
        )
        
        # 处理工具调用
        while response.stop_reason == "tool_use":
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    if block.name == "remember_fact":
                        fact = block.input.get("fact", "")
                        self.memory.save_important_fact(fact)
                        result = f"已记住: {fact}"
                    elif block.name == "recall_facts":
                        facts = [
                            m["content"] for m in self.memory.long_term
                            if m.get("type") == "fact"
                        ]
                        result = "已记住的事实：\n" + "\n".join(facts) if facts else "还没有记住任何事实"
                    else:
                        result = f"未知工具: {block.name}"
                    
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result,
                    })
            
            messages.append({"role": "assistant", "content": response.content})
            messages.append({"role": "user", "content": tool_results})
            
            response = self.client.messages.create(
                model="claude-sonnet-4-20250514",
                max_tokens=4096,
                tools=self.tools,
                system="你是一个友好的助手。你可以记住用户告诉你的信息，并在之后的对话中使用这些信息。",
                messages=messages,
            )
        
        # 提取最终回复
        final_text = ""
        for block in response.content:
            if block.type == "text":
                final_text += block.text
        
        # 添加助手回复到记忆
        self.memory.add_short_term("assistant", final_text)
        
        return final_text


def memory_agent_example():
    """记忆 Agent 示例"""
    print("=" * 60)
    print("具有记忆能力的 Claude Agent")
    print("=" * 60)
    
    agent = MemoryAgent()
    
    # 模拟多轮对话
    conversations = [
        "你好，我叫张明，我是一名软件工程师。",
        "我最近在学习机器学习，特别是自然语言处理方向。",
        "你记得我叫什么名字吗？",
        "我在哪家公司工作？",
        "帮我记住，我对花生过敏。",
    ]
    
    for message in conversations:
        print(f"\n用户: {message}")
        response = agent.chat(message)
        print(f"助手: {response}")
    
    # 查看记忆状态
    print(f"\n{'=' * 40}")
    print(f"短期记忆条数: {len(agent.memory.short_term)}")
    print(f"长期记忆条数: {len(agent.memory.long_term)}")
    
    print("\n已记住的事实:")
    for memory in agent.memory.long_term:
        if memory.get("type") == "fact":
            print(f"  - {memory['content']}")


if __name__ == "__main__":
    memory_agent_example()
```

### 示例 5：代码执行 Agent

```python
"""
Claude 代码执行 Agent 示例
安全地执行用户请求的代码
"""

import os
import json
import subprocess
import tempfile
import anthropic


def execute_python_code(code: str, timeout: int = 30) -> str:
    """安全执行 Python 代码"""
    try:
        # 创建临时文件
        with tempfile.NamedTemporaryFile(
            mode='w', suffix='.py', delete=False, encoding='utf-8'
        ) as f:
            f.write(code)
            temp_file = f.name
        
        # 执行代码
        result = subprocess.run(
            ['python', temp_file],
            capture_output=True,
            text=True,
            timeout=timeout,
        )
        
        # 清理临时文件
        os.unlink(temp_file)
        
        output = ""
        if result.stdout:
            output += f"输出:\n{result.stdout}"
        if result.stderr:
            output += f"\n错误:\n{result.stderr}"
        if result.returncode != 0:
            output += f"\n返回码: {result.returncode}"
        
        return output or "代码执行完成（无输出）"
        
    except subprocess.TimeoutExpired:
        return "错误: 代码执行超时"
    except Exception as e:
        return f"错误: {str(e)}"


class CodeAgent:
    """代码执行 Agent"""
    
    def __init__(self):
        self.client = anthropic.Anthropic(
            api_key=os.environ.get("ANTHROPIC_API_KEY")
        )
        
        self.tools = [
            {
                "name": "execute_code",
                "description": "执行 Python 代码并返回结果。可以用于计算、数据分析、文件处理等。",
                "input_schema": {
                    "type": "object",
                    "properties": {
                        "code": {
                            "type": "string",
                            "description": "要执行的 Python 代码",
                        }
                    },
                    "required": ["code"],
                },
            }
        ]
    
    def run(self, task: str, max_rounds: int = 5) -> str:
        """执行任务"""
        messages = [{"role": "user", "content": task}]
        
        for round_num in range(max_rounds):
            response = self.client.messages.create(
                model="claude-sonnet-4-20250514",
                max_tokens=4096,
                tools=self.tools,
                system=(
                    "你是一个代码执行助手。你可以编写并执行 Python 代码来完成用户的任务。"
                    "请确保代码是安全的，不要执行任何可能危害系统的操作。"
                    "每次执行代码后，分析结果并决定是否需要继续执行。"
                ),
                messages=messages,
            )
            
            if response.stop_reason != "tool_use":
                final_text = ""
                for block in response.content:
                    if block.type == "text":
                        final_text += block.text
                return final_text
            
            # 处理代码执行
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    if block.name == "execute_code":
                        code = block.input.get("code", "")
                        print(f"\n[执行代码]\n{code}\n")
                        
                        result = execute_python_code(code)
                        print(f"[结果]\n{result}\n")
                        
                        tool_results.append({
                            "type": "tool_result",
                            "tool_use_id": block.id,
                            "content": result,
                        })
            
            messages.append({"role": "assistant", "content": response.content})
            messages.append({"role": "user", "content": tool_results})
        
        return "达到最大执行轮次"


def code_agent_example():
    """代码执行 Agent 示例"""
    print("=" * 60)
    print("Claude 代码执行 Agent")
    print("=" * 60)
    
    agent = CodeAgent()
    
    # 测试任务
    task = """
    请帮我分析以下数据集：
    [23, 45, 12, 67, 89, 34, 56, 78, 90, 11, 44, 66]
    
    请计算：
    1. 平均值
    2. 中位数
    3. 标准差
    4. 最大值和最小值
    5. 绘制一个简单的柱状图（使用 ASCII 字符）
    """
    
    print(f"\n任务: {task}")
    print("\n" + "=" * 40)
    
    result = agent.run(task)
    print(f"\n最终结果:\n{result}")


if __name__ == "__main__":
    code_agent_example()
```

---

## 案例分析

### 案例 1：构建一个研究助手

让我们分析如何构建一个研究助手 Agent，它能够搜索信息、整理笔记、生成报告。

**系统架构：**

```
用户请求 → Claude 分析
              ↓
         需要搜索？→ 搜索工具 → 结果
              ↓
         需要计算？→ 计算工具 → 结果
              ↓
         需要保存？→ 存储工具 → 确认
              ↓
         生成报告 → 返回用户
```

**关键设计决策：**

1. **工具粒度**：研究助手的工具应该有足够的粒度。比如"搜索"工具应该支持不同的搜索引擎，"存储"工具应该支持不同的存储格式。

2. **错误恢复**：搜索可能失败，计算可能出错。Agent 需要能够优雅地处理这些错误。

3. **结果验证**：从外部获取的信息可能不准确。Agent 应该能够交叉验证多个来源的信息。

### 案例 2：构建一个调试助手

```python
"""
调试助手示例
帮助开发者调试代码问题
"""

import os
import json
import traceback
import anthropic


class DebugAssistant:
    """代码调试助手"""
    
    def __init__(self):
        self.client = anthropic.Anthropic(
            api_key=os.environ.get("ANTHROPIC_API_KEY")
        )
        
        self.tools = [
            {
                "name": "analyze_error",
                "description": "分析错误信息，找出可能的原因",
                "input_schema": {
                    "type": "object",
                    "properties": {
                        "error_type": {
                            "type": "string",
                            "description": "错误类型",
                        },
                        "error_message": {
                            "type": "string",
                            "description": "错误信息",
                        },
                        "code_context": {
                            "type": "string",
                            "description": "相关代码片段",
                        },
                    },
                    "required": ["error_type", "error_message"],
                },
            },
            {
                "name": "suggest_fix",
                "description": "根据错误分析建议修复方案",
                "input_schema": {
                    "type": "object",
                    "properties": {
                        "issue": {
                            "type": "string",
                            "description": "问题描述",
                        },
                        "suggestions": {
                            "type": "array",
                            "items": {"type": "string"},
                            "description": "可能的修复方案",
                        },
                    },
                    "required": ["issue", "suggestions"],
                },
            },
        ]
    
    def debug(self, error_info: str, code: str = "") -> str:
        """调试代码问题"""
        prompt = f"""
请帮我调试以下代码问题：

错误信息：
{error_info}

{"相关代码：" + code if code else ""}
"""
        
        messages = [{"role": "user", "content": prompt}]
        
        response = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            tools=self.tools,
            system="你是一个专业的代码调试助手。请分析错误原因并提供修复建议。",
            messages=messages,
        )
        
        final_text = ""
        for block in response.content:
            if block.type == "text":
                final_text += block.text
        
        return final_text


def debug_example():
    """调试助手示例"""
    assistant = DebugAssistant()
    
    error_info = """
Traceback (most recent call last):
  File "app.py", line 15, in process_data
    result = data['users'][index]['name']
KeyError: 'users'
"""
    
    code = """
def process_data(data, index):
    result = data['users'][index]['name']
    return result.lower()
"""
    
    print("=" * 60)
    print("调试助手分析结果：")
    print("=" * 60)
    
    result = assistant.debug(error_info, code)
    print(result)


if __name__ == "__main__":
    debug_example()
```

---

## 常见坑

### 坑 1：工具调用循环

Claude 可能会反复调用同一个工具，形成无限循环。解决方案是设置最大调用轮次。

```python
MAX_TOOL_ROUNDS = 10

for i in range(MAX_TOOL_ROUNDS):
    response = client.messages.create(...)
    if response.stop_reason != "tool_use":
        break
else:
    # 超过最大轮次，强制结束
    print("警告: 工具调用超过最大轮次")
```

### 坑 2：工具参数验证

永远不要信任模型生成的参数。必须在执行前验证参数的有效性。

```python
def safe_execute_tool(tool_name, tool_input):
    # 验证参数
    if not isinstance(tool_input, dict):
        return "错误: 参数必须是字典类型"
    
    # 根据工具类型进行具体验证
    if tool_name == "calculator":
        expression = tool_input.get("expression", "")
        # 检查是否包含危险操作
        dangerous = ["import", "exec", "eval", "__"]
        for d in dangerous:
            if d in expression:
                return "错误: 不允许执行危险操作"
    
    # 执行工具
    return execute_tool(tool_name, tool_input)
```

### 坑 3：上下文窗口管理

随着对话的进行，消息历史会越来越长。需要定期清理或压缩历史消息。

```python
def manage_context(messages, max_tokens=100000):
    """管理上下文长度"""
    total_tokens = sum(len(str(m["content"])) for m in messages)
    
    while total_tokens > max_tokens and len(messages) > 2:
        # 删除最旧的消息（保留第一条）
        removed = messages.pop(1)
        total_tokens -= len(str(removed["content"]))
    
    return messages
```

### 坑 4：错误信息泄露

工具执行的错误信息可能包含敏感数据。不要直接把错误信息返回给模型。

```python
def safe_tool_result(result, error):
    if error:
        # 只返回通用错误信息
        safe_errors = {
            "FileNotFoundError": "文件不存在",
            "PermissionError": "没有权限执行此操作",
            "ValueError": "输入参数无效",
        }
        error_type = type(error).__name__
        return safe_errors.get(error_type, "执行过程中发生错误")
    return result
```

### 坑 5：并发请求

Claude API 有并发限制。需要实现请求队列和限速机制。

```python
import time
from collections import deque

class RateLimiter:
    def __init__(self, max_requests_per_minute=60):
        self.max_requests = max_requests_per_minute
        self.timestamps = deque()
    
    def wait_if_needed(self):
        now = time.time()
        
        # 清理一分钟前的时间戳
        while self.timestamps and self.timestamps[0] < now - 60:
            self.timestamps.popleft()
        
        # 如果达到限制，等待
        if len(self.timestamps) >= self.max_requests:
            wait_time = 60 - (now - self.timestamps[0])
            if wait_time > 0:
                time.sleep(wait_time)
        
        self.timestamps.append(time.time())
```

---

## 练习题

### 练习 1：基础工具调用
创建一个包含以下工具的 Claude Agent：
- 计算器：执行数学运算
- 单位转换：长度、重量、温度之间的转换
- 时区转换：不同时间之间的转换

要求：Agent 能够根据用户问题自动选择合适的工具。

### 练习 2：多工具协作
设计一个旅行助手，它能够：
- 查询目的地天气
- 搜索航班信息
- 推荐餐厅
- 计算旅行预算

要求：一次查询可能需要多个工具协作完成。

### 练习 3：流式输出
实现一个流式工具调用的 Agent，能够：
- 实时显示 Claude 的思考过程
- 实时显示工具调用信息
- 实时显示最终回复

要求：使用 Anthropic SDK 的 stream 方法。

### 练习 4：记忆系统
为 Agent 实现完整的记忆系统：
- 短期记忆：最近 10 轮对话
- 长期记忆：用户偏好和重要事实
- 工作记忆：当前任务的上下文

要求：记忆能够在多次对话中保持。

### 练习 5：错误恢复
为 Agent 实现健壮的错误恢复机制：
- 网络错误重试
- 工具执行失败处理
- 超时处理
- 优雅降级

要求：所有错误都被妥善处理，不会导致 Agent 崩溃。

### 练习 6：安全框架
设计一个工具执行的安全框架：
- 参数验证
- 权限控制
- 操作审计
- 沙箱执行

要求：防止恶意代码执行和数据泄露。

---

## 实战任务

### 任务 1：构建一个完整的客服 Agent（中等难度）

**功能需求：**
1. 回答产品相关问题（使用知识库搜索）
2. 处理订单查询（使用数据库查询）
3. 计算退款金额（使用计算器）
4. 记录客户反馈（使用存储工具）

**技术要求：**
1. 使用 Claude 的工具调用能力
2. 实现完整的错误处理
3. 添加日志记录
4. 支持多轮对话

### 任务 2：构建一个数据分析平台（高难度）

**功能需求：**
1. 支持多种数据格式（CSV、JSON、Excel）
2. 自动识别数据类型
3. 生成数据可视化
4. 支持自然语言查询

**技术要求：**
1. 使用 Claude 编写和执行代码
2. 实现数据安全检查
3. 添加执行超时控制
4. 支持大型数据集

---

## 本章小结

本章我们深入学习了 Anthropic Claude 的工具使用能力。这是构建 Agent 系统的另一个重要基石。我们从以下几个方面进行了探索：

**工具定义**：学会了如何定义 Claude 可用的工具，包括工具名称、描述和参数 schema。工具定义的质量直接影响 Claude 使用工具的效果。

**工具调用循环**：理解了 Claude 工具调用的完整流程：分析需求 → 决定工具 → 返回调用请求 → 执行工具 → 返回结果 → 生成回复。这个循环是所有 Agent 系统的核心。

**多工具协作**：掌握了如何让 Claude 在一次响应中调用多个工具，以及如何让多个工具协作完成复杂任务。

**流式输出**：学会了如何使用流式输出实时展示工具调用过程，提升用户体验。

**记忆管理**：实现了具有记忆能力的 Agent，能够在多次对话中保持上下文和用户偏好。

**与 OpenAI 的对比**：Claude 的工具使用和 OpenAI 的函数调用在概念上类似，但在实现上有重要区别。Claude 支持在一次响应中返回多个工具调用，并且提供了工具使用预算等高级特性。

Claude 的工具使用能力为我们提供了构建 Agent 的底层积木。与 OpenAI 的 Assistants API 的托管模式不同，Claude 的工具使用更像是一套 SDK——它给你工具，但如何组装这些工具来构建 Agent，完全由你决定。这种灵活性意味着你可以构建任何你想要的 Agent 架构，但也意味着你需要自己处理更多的基础设施。

在下一章中，我们将学习 MCP 协议，看看它如何为 Agent 提供标准化的工具接口。

---

## 延伸阅读

1. Anthropic Tool Use 官方文档：https://docs.anthropic.com/en/docs/build-with-claude/tool-use
2. Claude API 快速入门：https://docs.anthropic.com/en/docs/quickstart
3. Claude 模型比较：https://docs.anthropic.com/en/docs/about-claude/models
4. Anthropic Python SDK：https://github.com/anthropics/anthropic-sdk-python
5. Claude 最佳实践：https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview
