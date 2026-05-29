# 第 4 章：Tool Calling —— Agent 的手脚

> **本章定位：** Tool Calling 是 Agent 从"只会说话"到"能做事"的关键转折。没有 Tool Calling，LLM 只是一个聊天机器人；有了 Tool Calling，LLM 变成了一个能执行行动的 Agent。本章将带你深入理解 Tool Calling 的机制，从工具定义到工具编排，构建一个完整的 Tool Calling 系统。

---

## 学习目标

1. 理解 Function Calling 的底层机制 —— LLM 如何决定使用哪个工具？它如何生成工具参数？
2. 掌握工具定义（JSON Schema）和工具注册 —— 如何设计一个好的工具描述？如何管理多个工具？
3. 理解工具调用的完整流程 —— 从用户输入到工具执行到结果回传的完整链路
4. 掌握多工具编排和工具链设计 —— 如何让多个工具协作完成复杂任务？
5. 能实现一个完整的 Tool Calling 系统

## 核心问题

1. **LLM 如何决定使用哪个工具？** 它是"随机"选择的吗？工具描述如何影响选择？
2. **工具调用结果如何反馈给 LLM？** 工具返回的数据如何被 LLM 理解和利用？
3. **如何处理工具调用失败？** 当工具不存在、参数错误、执行失败时，Agent 应该如何应对？

---

## 4.1 从聊天到执行：Tool Calling 的本质

### 4.1.1 一个思想实验

想象你有一个非常聪明的助手，但他只能"说话"，不能"做事"：

```
你：帮我查一下北京的天气
助手：北京今天可能天气不错，温度大约 25°C。（这是他猜的，可能是错的）
```

现在，如果你给他一部手机，让他能查天气：

```
你：帮我查一下北京的天气
助手：好的，我来查一下。（他打开手机，查看天气应用）
助手：北京今天晴，温度 25°C，湿度 40%。（这是真实数据）
```

**Tool Calling 就是那部手机。** 它让 LLM 从"猜测答案"变成"获取真实数据"。

### 4.1.2 Tool Calling 的核心价值

| 价值 | 描述 | 示例 |
|------|------|------|
| 获取实时数据 | LLM 的知识有截止日期 | 搜索最新新闻、查询实时股价 |
| 执行精确操作 | LLM 不能做数学计算 | 调用计算器、执行 SQL 查询 |
| 访问外部系统 | LLM 不能读写文件 | 读取文件、发送邮件、调用 API |
| 保证准确性 | 工具返回的结果是确定的 | 查询数据库、调用官方 API |

### 4.1.3 没有 Tool Calling vs 有 Tool Calling

```
没有 Tool Calling：
  用户："Python 最新版本是什么？"
  LLM："Python 最新版本是 3.13。"（可能是幻觉）

有 Tool Calling：
  用户："Python 最新版本是什么？"
  LLM：我来搜索一下。[search_web(query="Python latest version")]
  工具返回："Python 3.13.1，发布于 2024年10月"
  LLM："Python 最新版本是 3.13.1，于 2024 年 10 月发布。"
```

---

## 4.2 工具定义：JSON Schema

### 4.2.1 工具描述的结构

每个工具需要描述三部分：名称、描述、参数。这三部分共同告诉 LLM "这个工具是什么、什么时候用、怎么用"。

```python
weather_tool = {
    "type": "function",
    "function": {
        "name": "get_weather",                          # 工具名称（唯一标识）
        "description": "获取指定城市的当前天气信息。当用户询问天气时使用此工具。",  # 工具描述
        "parameters": {                                  # 参数定义（JSON Schema）
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "城市名称，如'北京'、'上海'",
                },
                "unit": {
                    "type": "string",
                    "enum": ["celsius", "fahrenheit"],
                    "description": "温度单位，默认摄氏度",
                },
            },
            "required": ["city"],  # 必需参数
        },
    },
}
```

### 4.2.2 工具描述的重要性

工具描述是 LLM 决定是否使用工具的唯一依据。描述质量直接影响工具调用的准确性。

**好的工具描述 vs 差的工具描述：**

```python
# 好的工具描述
{
    "name": "search_web",
    "description": "在互联网上搜索信息。当需要获取实时信息、最新新闻、或不确定的事实时使用此工具。不要用于计算或数学问题。",
}

# 差的工具描述
{
    "name": "search_web",
    "description": "搜索",  # 太简短，LLM 不知道什么时候用
}
```

**为什么描述质量这么重要？**

LLM 是通过阅读工具描述来决定是否使用工具的。如果描述不清楚，LLM 可能：
1. 在不需要工具时调用工具（浪费资源）
2. 在需要工具时不调用工具（得到错误答案）
3. 选择错误的工具（得到无关结果）

### 4.2.3 参数设计原则

**原则 1：参数名要有描述性**

```python
# 差的参数名
{"name": "q", "description": "查询"}

# 好的参数名
{"name": "query", "description": "搜索查询关键词"}
```

**原则 2：提供枚举值**

```python
# 差的：没有枚举
{"name": "unit", "type": "string", "description": "单位"}

# 好的：有枚举
{"name": "unit", "type": "string", "enum": ["celsius", "fahrenheit"], "description": "温度单位"}
```

**原则 3：明确默认值**

```python
# 在描述中说明默认值
{"name": "unit", "type": "string", "description": "温度单位，默认摄氏度"}
```

**原则 4：只暴露必要参数**

不要暴露实现细节。用户不需要知道 API 的内部参数。

```python
# 差的：暴露了内部参数
{"name": "api_key", "type": "string", "description": "API 密钥"}

# 好的：只暴露用户需要的参数
{"name": "city", "type": "string", "description": "城市名称"}
```

---

## 4.3 工具注册与管理

### 4.3.1 工具注册表

```python
from typing import Any, Callable
from dataclasses import dataclass, field


@dataclass
class Tool:
    """工具定义"""
    name: str
    description: str
    function: Callable[..., Any]
    parameters: dict
    required_params: list[str] = field(default_factory=list)


class ToolRegistry:
    """工具注册表"""

    def __init__(self):
        self._tools: dict[str, Tool] = {}

    def register(
        self,
        name: str,
        description: str,
        function: Callable[..., Any],
        parameters: dict,
        required_params: list[str] | None = None,
    ):
        """注册一个工具"""
        self._tools[name] = Tool(
            name=name,
            description=description,
            function=function,
            parameters=parameters,
            required_params=required_params or [],
        )

    def get_tool(self, name: str) -> Tool | None:
        """获取工具"""
        return self._tools.get(name)

    def list_tools(self) -> list[str]:
        """列出所有工具名称"""
        return list(self._tools.keys())

    def get_openai_tools(self) -> list[dict]:
        """生成 OpenAI 格式的工具定义"""
        tools = []
        for tool in self._tools.values():
            tools.append({
                "type": "function",
                "function": {
                    "name": tool.name,
                    "description": tool.description,
                    "parameters": tool.parameters,
                },
            })
        return tools

    def execute(self, name: str, **kwargs) -> Any:
        """执行工具"""
        tool = self._tools.get(name)
        if not tool:
            raise ValueError(f"工具不存在：{name}")

        # 检查必需参数
        for param in tool.required_params:
            if param not in kwargs:
                raise ValueError(f"缺少必需参数：{param}")

        return tool.function(**kwargs)
```

### 4.3.2 工具装饰器

使用装饰器可以简化工具注册：

```python
def tool(name: str, description: str):
    """工具装饰器"""
    def decorator(func):
        func._tool_name = name
        func._tool_description = description
        return func
    return decorator


@tool(name="calculator", description="执行数学计算")
def calculator(expression: str) -> str:
    """计算数学表达式"""
    try:
        result = eval(expression)  # 注意：生产环境不要用 eval
        return str(result)
    except Exception as e:
        return f"计算错误：{e}"


# 自动注册
registry = ToolRegistry()
registry.register(
    name=calculator._tool_name,
    description=calculator._tool_description,
    function=calculator,
    parameters={"type": "object", "properties": {"expression": {"type": "string"}}},
)
```

---

## 4.4 工具调用流程

### 4.4.1 完整流程图

```
1. 用户输入："北京天气怎么样？"
         ↓
2. 构建 Prompt（包含工具描述）→ 发送给 LLM
         ↓
3. LLM 分析：用户在问天气，应该调用天气工具
         ↓
4. LLM 输出：{"tool": "get_weather", "args": {"city": "北京"}}
         ↓
5. Agent 执行：调用天气 API → 获取天气数据
         ↓
6. 工具返回：{"temp": 25, "weather": "晴"}
         ↓
7. 将工具结果反馈给 LLM
         ↓
8. LLM 生成最终回答："北京今天晴，温度 25°C"
         ↓
9. 返回给用户
```

### 4.4.2 代码实现

```python
import json
import re
from openai import OpenAI


class ToolCallingAgent:
    def __init__(self, api_key: str, tools: ToolRegistry):
        self.client = OpenAI(api_key=api_key)
        self.tools = tools
        self.system_prompt = """你是一个有帮助的助手。你可以使用以下工具来完成任务。

工具列表：
{tool_descriptions}

当需要使用工具时，请输出以下格式：
[tool_name(param1=value1, param2=value2)]

当不需要工具时，直接回答用户问题。"""

    def run(self, user_input: str) -> str:
        """运行 Agent"""
        # 构建 Prompt
        tool_descriptions = self._get_tool_descriptions()
        system = self.system_prompt.format(tool_descriptions=tool_descriptions)

        messages = [
            {"role": "system", "content": system},
            {"role": "user", "content": user_input},
        ]

        # 调用 LLM
        response = self.client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            temperature=0.3,
        )

        assistant_message = response.choices[0].message.content

        # 检查是否需要调用工具
        tool_call = self._parse_tool_call(assistant_message)

        if tool_call is None:
            # 不需要调用工具，直接返回
            return assistant_message

        # 执行工具
        tool_name, tool_args = tool_call
        try:
            tool_result = self.tools.execute(tool_name, **tool_args)
        except Exception as e:
            tool_result = f"工具执行失败：{e}"

        # 将工具结果反馈给 LLM
        messages.append({"role": "assistant", "content": assistant_message})
        messages.append({
            "role": "user",
            "content": f"工具 {tool_name} 的返回结果：{tool_result}",
        })

        # 再次调用 LLM 生成最终回答
        final_response = self.client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            temperature=0.3,
        )

        return final_response.choices[0].message.content

    def _parse_tool_call(self, message: str) -> tuple[str, dict] | None:
        """解析工具调用"""
        pattern = r"\[(\w+)\((.*?)\)\]"
        match = re.search(pattern, message)
        if not match:
            return None

        tool_name = match.group(1)
        args_str = match.group(2)

        args = {}
        if args_str:
            for param in args_str.split(","):
                if "=" in param:
                    key, value = param.split("=", 1)
                    args[key.strip()] = value.strip().strip("'\"")

        return tool_name, args

    def _get_tool_descriptions(self) -> str:
        """获取工具描述"""
        descriptions = []
        for tool in self.tools._tools.values():
            desc = f"- {tool.name}: {tool.description}"
            descriptions.append(desc)
        return "\n".join(descriptions)
```

---

## 4.5 多工具编排

### 4.5.1 并行工具调用

某些 LLM（如 GPT-4o）支持在一次响应中调用多个工具：

```python
# LLM 输出多个工具调用
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "北京和上海的天气分别怎么样？"}],
    tools=tools,
)

# 可能返回两个工具调用
tool_calls = response.choices[0].message.tool_calls
# [
#   {"function": {"name": "get_weather", "arguments": '{"city": "北京"}'}},
#   {"function": {"name": "get_weather", "arguments": '{"city": "上海"}'}},
# ]

# 并行执行
results = {}
for tc in tool_calls:
    args = json.loads(tc.function.arguments)
    result = tools.execute(tc.function.name, **args)
    results[tc.function.name] = result
```

### 4.5.2 工具链

某些任务需要多个工具按顺序执行：

```
用户："帮我搜索 Python 最新版本，然后计算从 3.10 到最新版本的版本差"

工具链：
1. search_web("Python latest version") → "Python 3.13"
2. calculator("3.13 - 3.10") → "0.03"
```

```python
class ToolChain:
    def __init__(self, tools: ToolRegistry):
        self.tools = tools

    def execute_chain(self, steps: list[dict]) -> list[Any]:
        """执行工具链"""
        results = []
        context = {}

        for step in steps:
            tool_name = step["tool"]
            # 将之前的结果注入到参数中
            args = self._resolve_args(step["args"], context)

            result = self.tools.execute(tool_name, **args)
            results.append(result)
            context[step.get("output_key", tool_name)] = result

        return results

    def _resolve_args(self, args: dict, context: dict) -> dict:
        """解析参数中的变量引用"""
        resolved = {}
        for key, value in args.items():
            if isinstance(value, str) and value.startswith("$"):
                var_name = value[1:]
                resolved[key] = context.get(var_name, value)
            else:
                resolved[key] = value
        return resolved
```

---

## 4.6 工具调用的错误处理

### 4.6.1 常见错误类型

| 错误类型 | 描述 | 示例 |
|---------|------|------|
| 工具不存在 | LLM 调用了未注册的工具 | 工具名拼写错误 |
| 参数错误 | LLM 传入了错误的参数 | 缺少必需参数 |
| 执行失败 | 工具执行过程中出错 | API 调用失败 |
| 超时 | 工具执行时间过长 | 网络请求超时 |

### 4.6.2 错误恢复策略

```python
def execute_tool_with_recovery(
    registry: ToolRegistry,
    tool_name: str,
    args: dict,
) -> str:
    """带错误恢复的工具执行"""

    # 检查工具是否存在
    tool = registry.get_tool(tool_name)
    if not tool:
        available = ", ".join(registry.list_tools())
        return f"错误：工具 '{tool_name}' 不存在。可用工具：{available}"

    # 检查必需参数
    for param in tool.required_params:
        if param not in args:
            return f"错误：缺少必需参数 '{param}'。参数说明：{tool.parameters}"

    # 执行工具
    try:
        result = registry.execute(tool_name, **args)
        return str(result)
    except TimeoutError:
        return f"错误：工具 '{tool_name}' 执行超时，请稍后重试"
    except Exception as e:
        return f"错误：工具 '{tool_name}' 执行失败：{e}"
```

---

## 4.7 案例分析：构建一个搜索助手

### 需求

构建一个能搜索互联网、获取天气、进行计算的助手。

### 工具定义

```python
import os
import requests


def search_web(query: str) -> str:
    """搜索互联网"""
    api_key = os.getenv("TAVILY_API_KEY")
    response = requests.post(
        "https://api.tavily.com/search",
        json={"api_key": api_key, "query": query, "max_results": 3},
    )
    results = response.json().get("results", [])
    return "\n".join([f"- {r['title']}: {r['content'][:200]}" for r in results[:3]])


def get_weather(city: str) -> str:
    """获取天气"""
    api_key = os.getenv("OPENWEATHER_API_KEY")
    response = requests.get(
        "https://api.openweathermap.org/data/2.5/weather",
        params={"q": city, "appid": api_key, "units": "metric"},
    )
    data = response.json()
    return f"{city}：{data['weather'][0]['description']}，温度 {data['main']['temp']}°C"


def calculator(expression: str) -> str:
    """数学计算"""
    try:
        result = eval(expression)
        return str(result)
    except Exception as e:
        return f"计算错误：{e}"


# 注册工具
registry = ToolRegistry()
registry.register("search_web", "搜索互联网获取信息", search_web,
                  {"type": "object", "properties": {"query": {"type": "string"}}})
registry.register("get_weather", "获取城市天气", get_weather,
                  {"type": "object", "properties": {"city": {"type": "string"}}})
registry.register("calculator", "数学计算", calculator,
                  {"type": "object", "properties": {"expression": {"type": "string"}}})
```

---

## 4.8 常见坑与最佳实践

### 4.8.1 常见坑

1. **工具描述不清：** LLM 不知道什么时候该用工具。解决：提供详细、具体的描述。

2. **参数设计复杂：** LLM 容易传错参数。解决：参数尽量简单，使用枚举。

3. **不处理工具失败：** 工具执行失败时 Agent 崩溃。解决：始终处理异常。

4. **工具数量过多：** 超过 10 个工具时 LLM 容易混淆。解决：控制工具数量，使用工具分组。

### 4.8.2 最佳实践

1. **工具描述要清晰、具体** —— 说清楚工具做什么、什么时候用、什么时候不用
2. **参数尽量简单，使用枚举** —— 减少 LLM 犯错的机会
3. **始终处理工具调用失败** —— 提供有意义的错误信息
4. **控制工具数量在 5-10 个** —— 太多工具会让 LLM 困惑
5. **使用工具装饰器简化注册** —— 减少样板代码

---

## 4.9 高级话题：工具的动态发现与组合

### 4.9.1 动态工具注册

在复杂的 Agent 系统中，工具可能不是预先定义好的，而是根据任务动态加载的。比如一个通用助手 Agent，它可能在处理法律任务时加载法律工具，处理财务任务时加载财务工具。

```python
import importlib


class DynamicToolLoader:
    """动态工具加载器"""

    def __init__(self):
        self._tool_modules = {}
        self._loaded_tools = {}

    def register_module(self, category: str, module_path: str):
        """注册工具模块"""
        self._tool_modules[category] = module_path

    def load_tools(self, category: str) -> dict:
        """按类别加载工具"""
        if category in self._loaded_tools:
            return self._loaded_tools[category]

        module_path = self._tool_modules.get(category)
        if not module_path:
            raise ValueError(f"未注册的工具类别：{category}")

        module = importlib.import_module(module_path)
        tools = {}
        for attr_name in dir(module):
            attr = getattr(module, attr_name)
            if callable(attr) and hasattr(attr, "_tool_name"):
                tools[attr._tool_name] = {
                    "function": attr,
                    "description": attr._tool_description,
                }

        self._loaded_tools[category] = tools
        return tools

    def get_tools_for_task(self, task_description: str, llm_client) -> list:
        """根据任务描述动态选择需要的工具类别"""
        prompt = f"""根据以下任务描述，判断需要哪些类别的工具。
可用类别：{list(self._tool_modules.keys())}

任务：{task_description}

请返回需要的工具类别列表（逗号分隔）。"""

        response = llm_client.chat(
            messages=[{"role": "user", "content": prompt}]
        )

        categories = [c.strip() for c in response.split(",")]
        all_tools = {}
        for cat in categories:
            if cat in self._tool_modules:
                all_tools.update(self.load_tools(cat))

        return all_tools
```

### 4.9.2 工具组合模式

有时候，单一工具无法完成一个任务，需要将多个工具组合在一起形成"工具套件"。

```python
class ToolSuite:
    """工具套件：将多个工具组合为一个高级工具"""

    def __init__(self, name: str, description: str, tools: list):
        self.name = name
        self.description = description
        self.tools = {t.name: t for t in tools}

    def execute(self, task: str, llm_client) -> str:
        """使用 LLM 自动编排工具来完成任务"""
        tool_descriptions = "\n".join(
            f"- {t.name}: {t.description}" for t in self.tools.values()
        )
        prompt = f"""你有一个工具套件可以用来完成任务。

可用工具：
{tool_descriptions}

任务：{task}

请列出完成这个任务需要的工具调用步骤（按顺序）。格式：
1. tool_name(param=value)
2. tool_name(param=value)
..."""

        response = llm_client.chat(
            messages=[{"role": "user", "content": prompt}]
        )

        results = []
        for line in response.split("\n"):
            line = line.strip()
            if not line or not line[0].isdigit():
                continue
            # 解析并执行每个工具调用
            call = line.split(".", 1)[-1].strip()
            results.append(f"执行: {call}")

        return "\n".join(results)
```

---

## 4.10 练习题

### 概念理解题

1. Tool Calling 的核心流程是什么？请用流程图描述从用户输入到最终回答的完整过程。

2. 工具描述为什么这么重要？一个好的工具描述应该包含哪些要素？

3. 如何处理工具调用失败？请列出至少 3 种错误恢复策略。

4. 多工具编排有哪些模式？并行和串行各适用于什么场景？

5. 工具数量过多会有什么问题？如何解决？

### 动手实践题

1. **设计工具：** 为以下场景设计 3 个工具（名称、描述、参数）：
   - 文件操作 Agent
   - 数据分析 Agent
   - 邮件处理 Agent

2. **实现工具注册表：** 使用本章的代码，实现一个完整的 ToolRegistry。

3. **测试工具调用：** 用 5 个不同的用户输入测试工具调用的准确性。

4. **实现错误恢复：** 编写一个工具执行函数，支持超时处理、重试和降级，分别用装饰器和上下文管理器两种方式实现。

5. **设计工具组合：** 为"数据分析助手"设计一个工具组合，包含数据获取、清洗、分析、可视化四个环节的工具。

### 思考题

1. 如果 LLM 选择了一个不合适的工具，Agent 应该如何处理？
2. 如何设计一个工具，让它能处理多种不同类型的任务？
3. Tool Calling 和 RAG 各适用于什么场景？它们可以结合使用吗？

---

## 4.10 实战任务

**任务：实现一个带 5 个工具的 Tool Calling 系统**

**具体要求：**

1. 实现工具注册表（ToolRegistry）
2. 注册 5 个工具（搜索、天气、计算、翻译、摘要）
3. 实现工具调用解析
4. 实现错误处理
5. 测试完整流程（至少 10 个测试用例）

**评估标准：**

| 维度 | 权重 | 评分标准 |
|------|------|---------|
| 工具设计 | 25% | 描述清晰、参数合理 |
| 注册表实现 | 20% | 功能完整、接口清晰 |
| 调用解析 | 20% | 正确解析各种格式 |
| 错误处理 | 20% | 覆盖各种错误场景 |
| 测试覆盖 | 15% | 测试用例全面 |

---

## 4.11 本章小结

### 核心知识点回顾

- **Tool Calling 的本质：** 让 LLM 从"猜测答案"变成"获取真实数据"。它是 Agent 从聊天机器人到智能体的关键转折。

- **工具描述的三要素：** 名称（唯一标识）、描述（告诉 LLM 什么时候用）、参数（告诉 LLM 怎么用）。描述质量直接影响调用准确性。

- **工具调用流程：** 用户输入 → LLM 决策 → 工具执行 → 结果回传 → LLM 生成回答。

- **多工具编排：** 并行调用提高效率，工具链支持复杂任务。

- **错误处理：** 始终处理工具不存在、参数错误、执行失败等场景。

### 关键公式

```
Tool Calling = 工具定义 × 工具描述质量 × LLM 决策能力 × 错误处理
```

### 下一章预告

下一章我们将深入探讨 **Agent 的本质** —— LLM + Loop + Tools。你将理解 Agent 为什么需要循环，循环如何让 Agent 能持续工作，以及如何平衡 Agent 的自主性和人类控制。

---

*上一章：[第 3 章：LLM API 工程](../chapter-03/README.md)*
*下一章：[第 5 章：Agent 的本质](../chapter-05/README.md)*
