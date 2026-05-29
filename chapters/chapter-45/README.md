# 第45章：OpenAI Assistants API —— 官方 Agent 实现

## 学习目标

通过本章的学习，你将能够：
1. 理解 OpenAI Assistants API 的设计理念和架构
2. 掌握 Assistant、Thread、Run 三个核心概念之间的关系
3. 学会使用 Assistants API 创建具有工具调用能力的 Agent
4. 理解内置工具（代码解释器、文件搜索、函数调用）的使用方式
5. 掌握流式输出和轮询两种运行模式
6. 能够构建一个完整的、具有持久化对话能力的 Agent 应用

## 核心问题

在前面的章节中，我们学习了如何从零开始构建 Agent，包括 ReAct 模式、工具调用、记忆管理等。但这些实现都有一个共同的问题：我们需要自己处理大量的基础设施代码。状态管理、工具调用循环、对话持久化、文件处理……这些"脏活累活"消耗了我们大量的精力。

那么，有没有一种方式，让我们只需要关注"Agent 要做什么"，而不需要关心"Agent 怎么做到"呢？

OpenAI 的 Assistants API 正是为回答这个问题而生的。它将 Agent 的核心循环封装成了一个托管服务，让我们可以用几行代码就能创建一个功能完整的 Agent。这就像是从自己造汽车，变成了买一辆车——你只需要告诉它去哪里，它就会带你过去。

---

## 原理讲解

### 从 Chat Completions 到 Assistants：一次思维的跃迁

要理解 Assistants API，我们首先需要回顾一下 Chat Completions API 的局限性。

在 Chat Completions 的世界里，每次 API 调用都是一次性的：你发送消息，模型返回回复，然后对话结束。如果你想实现一个多轮对话，你需要自己把之前的所有消息都重新发送一遍。这就好比每次和朋友聊天，你都得把之前说过的话全部复述一遍，才能继续对话——这显然很不方便。

更重要的是，Chat Completions API 本身不提供任何持久化的状态。你的对话历史存在你自己的服务器上，如果服务器重启了，对话就丢失了。模型也不记得它之前说了什么，除非你主动告诉它。

Assistants API 彻底改变了这种模式。它引入了三个核心概念：

**Thread（线程）**：这就是对话本身。你可以把 Thread 想象成一个聊天窗口，它会自动保存所有的对话历史。你不需要在每次请求时重新发送整个对话——只需要发送新的消息，API 会自动处理上下文。

**Assistant（助手）**：这就是你的 Agent 的"人格"。它定义了 Agent 的行为方式，包括它的系统提示、它可以使用的工具、以及它使用的模型。你可以创建不同用途的 Assistant，比如一个专门写代码的助手、一个专门分析数据的助手。

**Run（运行）**：这是一次执行过程。当你向 Thread 中添加一条新消息时，你需要触发一个 Run，让 Assistant 处理这条消息。Run 会自动处理工具调用的循环——如果 Assistant 需要调用工具，API 会自动执行，然后把结果反馈给 Assistant，直到 Assistant 给出最终回复。

这三个概念之间的关系可以用一个生活中的比喻来理解。想象你去银行办理业务：

- **Thread** 就是你在柜台前的整个服务过程，从你坐下到离开
- **Assistant** 就是银行柜员，他具备各种技能（操作电脑、查询系统、打印文件等）
- **Run** 就是柜员处理你每一个具体请求的过程

当你告诉柜员"我要转账"（添加消息），柜员就会开始工作（Run 开始）：他可能需要查询你的余额（调用工具），确认收款人信息（调用工具），然后执行转账（调用工具），最后告诉你"转账成功"（最终回复）。

### Assistants API 的核心架构

让我们从技术角度来理解 Assistants API 的架构。

```
用户消息 → Thread → 触发 Run → Assistant 处理
                                  ↓
                            需要工具？ → 执行工具 → 结果返回
                                  ↓
                            不需要 → 生成最终回复
                                  ↓
                            Run 结束 → 回复出现在 Thread 中
```

整个过程是异步的。你创建一个 Run 后，它会立即返回一个 Run 对象，状态为 "queued"。你需要不断查询 Run 的状态，直到它变成 "completed"。这就像你在网上提交了一个快递订单，你需要不断刷新页面查看快递状态。

不过，Assistants API 也支持流式输出（Streaming）。在流式模式下，你可以实时地看到 Assistant 的思考过程和回复内容，就像看着快递员一步步接近你的家。

### 内置工具：三大法宝

Assistants API 提供了三个内置工具，它们是 Agent 能力的核心：

**代码解释器（Code Interpreter）**：这让 Assistant 能够编写和执行 Python 代码。它在一个安全的沙箱环境中运行，可以处理数学计算、数据分析、文件处理等任务。比如你问"帮我分析这个 CSV 文件"，Assistant 会自动编写 Python 代码来读取和分析文件。

**文件搜索（File Search）**：这让 Assistant 能够搜索上传的文件。它使用向量搜索技术，能够找到文件中最相关的内容。比如你上传了一本手册，然后问"产品保修期是多久"，Assistant 会在手册中找到答案。

**函数调用（Function Calling）**：这和 Chat Completions 中的函数调用类似，但更加自动化。你只需要定义函数的签名，API 会自动处理调用和结果返回。

这三种工具可以组合使用，让 Assistant 成为一个真正强大的助手。

### 持久化与状态管理

Assistants API 最大的优势之一就是状态管理。所有的 Thread 和 Message 都会被自动保存在 OpenAI 的服务器上。这意味着：

1. 你不需要管理对话历史——API 会自动保存和检索
2. 即使你的服务器重启了，对话也不会丢失
3. 你可以随时查看任何 Thread 的历史消息
4. 同一个 Thread 可以被多次使用，就像一个持续的聊天记录

这种设计大大简化了 Agent 的开发。你不再需要自己实现复杂的对话管理系统，只需要把 Thread ID 保存好，就可以随时恢复对话。

### 并发与限制

在使用 Assistants API 时，有几个重要的限制需要注意：

1. **Thread 中的消息数量限制**：每个 Thread 最多可以有约 32,768 条消息
2. **同时运行的 Run 数量**：每个 Thread 同时只能有一个 Run 在执行
3. **Assistant 的并发限制**：不同 tier 的账户有不同的并发限制
4. **工具使用限制**：每个 Run 中工具调用的次数有上限

了解这些限制对于设计健壮的应用至关重要。

---

## 完整代码示例

### 示例 1：基础 Assistant 创建与对话

```python
"""
OpenAI Assistants API 基础示例
创建一个简单的对话助手

环境要求：
    pip install openai
    
环境变量：
    OPENAI_API_KEY: OpenAI API 密钥
"""

import os
import time
from openai import OpenAI

# 从环境变量获取 API 密钥
client = OpenAI(api_key=os.environ.get("OPENAI_API_KEY"))


def create_assistant(name: str, instructions: str, model: str = "gpt-4o") -> dict:
    """
    创建一个新的 Assistant
    
    参数:
        name: 助手的名称
        instructions: 助手的系统提示（定义行为）
        model: 使用的模型
    
    返回:
        Assistant 对象的字典
    """
    assistant = client.beta.assistants.create(
        name=name,
        instructions=instructions,
        model=model,
    )
    print(f"Assistant 创建成功: {assistant.id}")
    return assistant


def create_thread() -> dict:
    """创建一个新的 Thread（对话线程）"""
    thread = client.beta.threads.create()
    print(f"Thread 创建成功: {thread.id}")
    return thread


def add_message_to_thread(thread_id: str, role: str, content: str) -> dict:
    """
    向 Thread 中添加一条消息
    
    参数:
        thread_id: Thread 的 ID
        role: 消息角色，可以是 "user" 或 "assistant"
        content: 消息内容
    """
    message = client.beta.threads.messages.create(
        thread_id=thread_id,
        role=role,
        content=content,
    )
    return message


def run_assistant(thread_id: str, assistant_id: str) -> dict:
    """
    触发一次 Assistant 运行
    
    这个函数会等待 Assistant 完成处理，然后返回最终的 Run 对象
    """
    run = client.beta.threads.runs.create(
        thread_id=thread_id,
        assistant_id=assistant_id,
    )
    
    # 轮询等待 Run 完成
    while run.status != "completed":
        time.sleep(1)  # 每秒查询一次
        run = client.beta.threads.runs.retrieve(
            thread_id=thread_id,
            run_id=run.id,
        )
        print(f"  Run 状态: {run.status}")
    
    return run


def get_latest_assistant_message(thread_id: str) -> str:
    """获取 Assistant 的最新回复"""
    messages = client.beta.threads.messages.list(thread_id=thread_id)
    
    # 获取最新的 assistant 消息
    for message in messages.data:
        if message.role == "assistant":
            # 提取文本内容
            for content_block in message.content:
                if content_block.type == "text":
                    return content_block.text.value
    
    return "没有找到 Assistant 的回复"


def basic_conversation():
    """完整的对话流程示例"""
    print("=" * 60)
    print("基础 Assistant 对话示例")
    print("=" * 60)
    
    # 1. 创建 Assistant
    assistant = create_assistant(
        name="学习助手",
        instructions=(
            "你是一个耐心的学习助手。你擅长用简单的语言解释复杂的概念。"
            "如果用户的问题不清楚，你会先询问以获取更多信息。"
            "你总是鼓励学生，并提供具体的例子来帮助理解。"
        ),
        model="gpt-4o"
    )
    
    # 2. 创建 Thread
    thread = create_thread()
    
    # 3. 添加用户消息并运行
    print("\n--- 用户提问：什么是机器学习？ ---\n")
    add_message_to_thread(
        thread_id=thread.id,
        role="user",
        content="什么是机器学习？能给我举一个简单的例子吗？"
    )
    
    run_assistant(thread_id=thread.id, assistant_id=assistant.id)
    
    response = get_latest_assistant_message(thread_id=thread.id)
    print(f"Assistant 回复:\n{response}\n")
    
    # 4. 继续对话（多轮对话）
    print("\n--- 用户追问：能再深入解释一下吗？ ---\n")
    add_message_to_thread(
        thread_id=thread.id,
        role="user",
        content="你能再深入解释一下机器学习和深度学习的区别吗？"
    )
    
    run_assistant(thread_id=thread.id, assistant_id=assistant.id)
    
    response = get_latest_assistant_message(thread_id=thread.id)
    print(f"Assistant 回复:\n{response}\n")
    
    # 清理资源
    client.beta.assistants.delete(assistant_id=assistant.id)
    print("\nAssistant 已删除")


if __name__ == "__main__":
    basic_conversation()
```

### 示例 2：使用代码解释器工具

```python
"""
使用代码解释器工具的 Assistant 示例
Assistant 可以编写和执行 Python 代码来解决问题
"""

import os
import time
import json
from openai import OpenAI

client = OpenAI(api_key=os.environ.get("OPENAI_API_KEY"))


def run_with_code_interpreter():
    """使用代码解释器的完整示例"""
    print("=" * 60)
    print("代码解释器示例：数据分析助手")
    print("=" * 60)
    
    # 创建一个具有代码解释器能力的 Assistant
    assistant = client.beta.assistants.create(
        name="数据分析助手",
        instructions=(
            "你是一个专业的数据分析助手。你可以使用 Python 代码来处理数据、"
            "生成图表、进行统计分析。当用户提供数据时，你应该：\n"
            "1. 理解数据的结构\n"
            "2. 编写 Python 代码进行分析\n"
            "3. 将代码执行结果清晰地展示给用户\n"
            "4. 提供有价值的洞察和建议"
        ),
        model="gpt-4o",
        tools=[{"type": "code_interpreter"}],
    )
    
    # 创建 Thread
    thread = client.beta.threads.create()
    
    # 发送一个数据分析任务
    analysis_request = """
    请帮我分析以下销售数据：
    
    月份,销售额,成本,利润
    1月,10000,6000,4000
    2月,12000,7000,5000
    3月,15000,8000,7000
    4月,13000,7500,5500
    5月,18000,9000,9000
    6月,20000,10000,10000
    
    请计算：
    1. 半年的总销售额、总成本和总利润
    2. 平均月利润率
    3. 哪个月的表现最好，为什么
    4. 趋势分析：下半年应该如何调整策略
    """
    
    print("\n用户请求数据分析...")
    client.beta.threads.messages.create(
        thread_id=thread.id,
        role="user",
        content=analysis_request,
    )
    
    # 运行 Assistant
    print("Assistant 正在分析数据...")
    run = client.beta.threads.runs.create(
        thread_id=thread.id,
        assistant_id=assistant.id,
    )
    
    # 监控运行过程
    while run.status != "completed":
        time.sleep(2)
        run = client.beta.threads.runs.retrieve(
            thread_id=thread.id,
            run_id=run.id,
        )
        print(f"  状态: {run.status}")
        
        # 检查是否有工具调用
        if run.status == "requires_action":
            print("  Assistant 需要执行工具调用...")
    
    # 获取所有消息（包括代码执行结果）
    print("\n" + "=" * 60)
    print("分析结果：")
    print("=" * 60)
    
    messages = client.beta.threads.messages.list(thread_id=thread.id)
    
    for message in messages.data:
        if message.role == "assistant":
            for content_block in message.content:
                if content_block.type == "text":
                    print(f"\nAssistant:\n{content_block.text.value}")
                elif content_block.type == "tool_result":
                    print(f"\n工具执行结果:\n{content_block.tool_result}")
    
    # 清理
    client.beta.assistants.delete(assistant_id=assistant.id)


if __name__ == "__main__":
    run_with_code_interpreter()
```

### 示例 3：使用函数调用工具

```python
"""
使用自定义函数调用的 Assistant 示例
定义外部函数，让 Assistant 能够调用自定义的 API
"""

import os
import time
import json
from openai import OpenAI

client = OpenAI(api_key=os.environ.get("OPENAI_API_KEY"))


# 定义可供 Assistant 调用的函数

def get_weather(city: str, unit: str = "celsius") -> dict:
    """获取指定城市的天气信息"""
    # 模拟天气 API 返回
    weather_data = {
        "北京": {"temp": 25, "condition": "晴", "humidity": 45},
        "上海": {"temp": 28, "condition": "多云", "humidity": 70},
        "广州": {"temp": 32, "condition": "阴", "humidity": 85},
        "深圳": {"temp": 30, "condition": "雷阵雨", "humidity": 80},
    }
    
    data = weather_data.get(city, {"temp": 20, "condition": "未知", "humidity": 50})
    
    if unit == "fahrenheit":
        data["temp"] = data["temp"] * 9 / 5 + 32
    
    return {
        "city": city,
        "temperature": data["temp"],
        "condition": data["condition"],
        "humidity": data["humidity"],
        "unit": unit,
    }


def search_flights(origin: str, destination: str, date: str) -> dict:
    """搜索航班信息"""
    # 模拟航班搜索结果
    return {
        "origin": origin,
        "destination": destination,
        "date": date,
        "flights": [
            {"airline": "中国国航", "flight_no": "CA1234", "departure": "08:00", "arrival": "11:00", "price": 1200},
            {"airline": "东方航空", "flight_no": "MU5678", "departure": "10:30", "arrival": "13:30", "price": 980},
            {"airline": "南方航空", "flight_no": "CZ9012", "departure": "14:00", "arrival": "17:00", "price": 1100},
        ],
    }


def book_restaurant(restaurant_name: str, date: str, time: str, party_size: int) -> dict:
    """预订餐厅"""
    return {
        "restaurant": restaurant_name,
        "date": date,
        "time": time,
        "party_size": party_size,
        "status": "confirmed",
        "confirmation_number": "RES2024001",
    }


# 定义工具列表
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "获取指定城市的当前天气信息，包括温度、天气状况和湿度",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "城市名称，例如：北京、上海、广州",
                    },
                    "unit": {
                        "type": "string",
                        "enum": ["celsius", "fahrenheit"],
                        "description": "温度单位，默认摄氏度",
                    },
                },
                "required": ["city"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "search_flights",
            "description": "搜索指定日期从一个城市到另一个城市的航班信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "origin": {
                        "type": "string",
                        "description": "出发城市",
                    },
                    "destination": {
                        "type": "string",
                        "description": "到达城市",
                    },
                    "date": {
                        "type": "string",
                        "description": "出发日期，格式：YYYY-MM-DD",
                    },
                },
                "required": ["origin", "destination", "date"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "book_restaurant",
            "description": "为用户预订餐厅",
            "parameters": {
                "type": "object",
                "properties": {
                    "restaurant_name": {
                        "type": "string",
                        "description": "餐厅名称",
                    },
                    "date": {
                        "type": "string",
                        "description": "预订日期，格式：YYYY-MM-DD",
                    },
                    "time": {
                        "type": "string",
                        "description": "预订时间，格式：HH:MM",
                    },
                    "party_size": {
                        "type": "integer",
                        "description": "用餐人数",
                    },
                },
                "required": ["restaurant_name", "date", "time", "party_size"],
            },
        },
    },
]


# 映射函数名到实际函数
available_functions = {
    "get_weather": get_weather,
    "search_flights": search_flights,
    "book_restaurant": book_restaurant,
}


def process_tool_calls(run, thread_id):
    """处理 Assistant 的工具调用"""
    tool_outputs = []
    
    for tool_call in run.required_action.submit_tool_outputs.tool_calls:
        function_name = tool_call.function.name
        function_args = json.loads(tool_call.function.arguments)
        
        print(f"\n  调用函数: {function_name}")
        print(f"  参数: {json.dumps(function_args, ensure_ascii=False)}")
        
        # 执行函数
        if function_name in available_functions:
            result = available_functions[function_name](**function_args)
            print(f"  结果: {json.dumps(result, ensure_ascii=False)}")
            
            tool_outputs.append({
                "tool_call_id": tool_call.id,
                "output": json.dumps(result, ensure_ascii=False),
            })
        else:
            tool_outputs.append({
                "tool_call_id": tool_call.id,
                "output": json.dumps({"error": f"未知函数: {function_name}"}),
            })
    
    # 提交工具输出
    return client.beta.threads.runs.submit_tool_outputs(
        thread_id=thread_id,
        run_id=run.id,
        tool_outputs=tool_outputs,
    )


def travel_assistant():
    """旅行规划助手示例"""
    print("=" * 60)
    print("旅行规划助手")
    print("=" * 60)
    
    # 创建 Assistant
    assistant = client.beta.assistants.create(
        name="旅行规划助手",
        instructions=(
            "你是一个贴心的旅行规划助手。你可以帮用户查询天气、搜索航班、"
            "预订餐厅。在做推荐时，你要考虑用户的预算和偏好。"
            "你需要先获取足够的信息，然后给出完整的旅行建议。"
        ),
        model="gpt-4o",
        tools=tools,
    )
    
    # 创建 Thread
    thread = client.beta.threads.create()
    
    # 用户请求
    user_request = "我计划下周五从北京去上海出差，你能帮我看看天气和航班吗？"
    
    print(f"\n用户: {user_request}\n")
    
    client.beta.threads.messages.create(
        thread_id=thread.id,
        role="user",
        content=user_request,
    )
    
    # 运行
    run = client.beta.threads.runs.create(
        thread_id=thread.id,
        assistant_id=assistant.id,
    )
    
    # 处理运行循环
    while run.status != "completed":
        time.sleep(1)
        run = client.beta.threads.runs.retrieve(
            thread_id=thread.id,
            run_id=run.id,
        )
        print(f"  Run 状态: {run.status}")
        
        # 如果需要工具调用
        if run.status == "requires_action":
            print("\n  Assistant 需要调用工具...")
            run = process_tool_calls(run, thread.id)
    
    # 获取最终回复
    messages = client.beta.threads.messages.list(thread_id=thread.id)
    for message in messages.data:
        if message.role == "assistant":
            for block in message.content:
                if block.type == "text":
                    print(f"\n助手回复:\n{block.text.value}")
    
    # 清理
    client.beta.assistants.delete(assistant_id=assistant.id)


if __name__ == "__main__":
    travel_assistant()
```

### 示例 4：文件搜索助手

```python
"""
使用文件搜索工具的 Assistant 示例
上传文件并让 Assistant 在文件中搜索信息
"""

import os
import time
from openai import OpenAI

client = OpenAI(api_key=os.environ.get("OPENAI_API_KEY"))


def create_sample_file():
    """创建一个示例文件用于上传"""
    content = """
# 公司产品手册 v2.0

## 产品一：Smart Widget

### 产品描述
Smart Widget 是一款智能硬件设备，可以连接到用户的智能家居系统。
它支持 WiFi 和蓝牙两种连接方式，兼容所有主流的智能家居平台。

### 技术参数
- 尺寸：10cm x 10cm x 5cm
- 重量：200g
- 电源：USB-C，5V/2A
- 工作温度：-10°C 到 50°C
- 连接方式：WiFi 2.4GHz / Bluetooth 5.0

### 价格
- 标准版：¥299
- 专业版：¥499（含高级传感器）
- 企业版：¥899（含API访问权限）

### 保修政策
- 标准保修：1年
- 专业版保修：2年
- 企业版保修：3年
- 延保服务：可额外购买1年延保，价格为产品售价的10%

## 产品二：CloudSync

### 产品描述
CloudSync 是一款云端数据同步服务，支持实时同步文件、数据库和配置信息。
提供 99.99% 的可用性保证。

### 定价方案
- 免费版：5GB 存储，每月 1GB 流量
- 基础版：¥49/月，50GB 存储，每月 10GB 流量
- 专业版：¥199/月，500GB 存储，每月 100GB 流量
- 企业版：¥999/月，无限存储，无限流量

### 技术支持
- 免费版：仅邮件支持，48小时内回复
- 基础版：在线客服，24小时内回复
- 专业版：专属客服，4小时内回复
- 企业版：7x24 电话支持，1小时内响应

### 数据安全
- 所有数据传输使用 TLS 1.3 加密
- 数据存储使用 AES-256 加密
- 符合 GDPR 和中国网络安全法要求
- 支持数据导出和删除请求

## 常见问题

Q: 产品的退款政策是什么？
A: 购买后30天内可申请全额退款，需保持产品完好。

Q: 企业版的 SLA 是什么？
A: 企业版保证 99.99% 的可用性，如未达标，按比例退还当月费用。

Q: 如何联系技术支持？
A: 可通过 support@company.com 或拨打 400-XXX-XXXX。
"""
    
    file_path = "product_manual.txt"
    with open(file_path, "w", encoding="utf-8") as f:
        f.write(content)
    
    return file_path


def file_search_assistant():
    """文件搜索助手示例"""
    print("=" * 60)
    print("文件搜索助手")
    print("=" * 60)
    
    # 1. 创建示例文件
    file_path = create_sample_file()
    print(f"创建示例文件: {file_path}")
    
    # 2. 上传文件
    print("\n上传文件到 OpenAI...")
    with open(file_path, "rb") as f:
        uploaded_file = client.files.create(
            file=f,
            purpose="assistants",
        )
    print(f"文件上传成功: {uploaded_file.id}")
    
    # 3. 创建 Vector Store 并关联文件
    print("\n创建 Vector Store...")
    vector_store = client.beta.vector_stores.create(
        name="产品手册库",
        file_ids=[uploaded_file.id],
    )
    print(f"Vector Store 创建成功: {vector_store.id}")
    
    # 4. 创建具有文件搜索能力的 Assistant
    assistant = client.beta.assistants.create(
        name="产品知识助手",
        instructions=(
            "你是一个产品知识助手。你的任务是根据上传的产品文档回答用户的问题。"
            "在回答时，你应该引用具体的文档内容，并给出准确的答案。"
            "如果文档中没有相关信息，请诚实地说不知道。"
        ),
        model="gpt-4o",
        tools=[{"type": "file_search"}],
        tool_resources={
            "file_search": {
                "vector_store_ids": [vector_store.id],
            }
        },
    )
    print(f"Assistant 创建成功: {assistant.id}")
    
    # 5. 创建 Thread 并对话
    thread = client.beta.threads.create()
    
    # 提问
    questions = [
        "Smart Widget 专业版的价格是多少？保修期是多久？",
        "CloudSync 企业版支持什么级别的技术支持？",
        "如果我想退货，退款政策是什么？",
    ]
    
    for question in questions:
        print(f"\n{'-' * 40}")
        print(f"用户: {question}\n")
        
        client.beta.threads.messages.create(
            thread_id=thread.id,
            role="user",
            content=question,
        )
        
        run = client.beta.threads.runs.create(
            thread_id=thread.id,
            assistant_id=assistant.id,
        )
        
        while run.status != "completed":
            time.sleep(1)
            run = client.beta.threads.runs.retrieve(
                thread_id=thread.id,
                run_id=run.id,
            )
        
        # 获取回复
        messages = client.beta.threads.messages.list(thread_id=thread.id)
        for msg in messages.data:
            if msg.role == "assistant":
                for block in msg.content:
                    if block.type == "text":
                        print(f"助手: {block.text.value}")
    
    # 6. 清理资源
    print("\n清理资源...")
    client.beta.assistants.delete(assistant_id=assistant.id)
    client.beta.vector_stores.delete(vector_store_id=vector_store.id)
    client.files.delete(file_id=uploaded_file.id)
    os.remove(file_path)
    print("清理完成")


if __name__ == "__main__":
    file_search_assistant()
```

### 示例 5：流式输出的 Assistant

```python
"""
Assistant 的流式输出示例
实时获取 Assistant 的思考和回复过程
"""

import os
import json
from openai import OpenAI

client = OpenAI(api_key=os.environ.get("OPENAI_API_KEY"))


def streaming_assistant():
    """使用流式输出的 Assistant"""
    print("=" * 60)
    print("流式输出 Assistant 示例")
    print("=" * 60)
    
    # 创建 Assistant
    assistant = client.beta.assistants.create(
        name="写作助手",
        instructions=(
            "你是一个创意写作助手。你擅长写短篇故事、诗歌和文案。"
            "你的文字优美、生动，能够抓住读者的注意力。"
            "在创作时，你会先构思大纲，然后逐步展开。"
        ),
        model="gpt-4o",
    )
    
    # 创建 Thread
    thread = client.beta.threads.create()
    
    # 添加用户消息
    client.beta.threads.messages.create(
        thread_id=thread.id,
        role="user",
        content="请写一个关于程序员的小故事，大约200字，要幽默风趣。",
    )
    
    # 使用流式输出
    print("\nAssistant 正在创作...\n")
    
    with client.beta.threads.runs.stream(
        thread_id=thread.id,
        assistant_id=assistant.id,
    ) as stream:
        for event in stream:
            # 处理文本增量
            if event.event == "thread.run.created":
                print("[Run 创建]")
            elif event.event == "thread.run.queued":
                print("[排队中]")
            elif event.event == "thread.run.in_progress":
                print("[处理中]")
            elif event.event == "thread.run.delta":
                # 获取文本增量
                delta = event.data.delta
                if hasattr(delta, "content"):
                    for block in delta.content:
                        if hasattr(block, "text"):
                            print(block.text.value, end="", flush=True)
            elif event.event == "thread.run.completed":
                print("\n\n[完成]")
    
    # 清理
    client.beta.assistants.delete(assistant_id=assistant.id)


if __name__ == "__main__":
    streaming_assistant()
```

---

## 案例分析

### 案例 1：构建一个客服 Agent

让我们分析一个实际的场景：为一家电商公司构建客服 Agent。

**需求分析：**
- 回答产品相关问题（需要文件搜索）
- 处理订单查询（需要函数调用）
- 计算退换货金额（需要代码解释器）
- 保持对话上下文（需要 Thread）

**架构设计：**

```python
"""
电商客服 Agent 完整示例
"""

import os
import time
import json
from openai import OpenAI

client = OpenAI(api_key=os.environ.get("OPENAI_API_KEY"))


# 数据库模拟
orders_db = {
    "ORD001": {
        "order_id": "ORD001",
        "customer": "张三",
        "items": [
            {"name": "无线耳机", "price": 299, "quantity": 1},
            {"name": "手机壳", "price": 49, "quantity": 2},
        ],
        "total": 397,
        "status": "已发货",
        "shipping_date": "2024-01-15",
    },
    "ORD002": {
        "order_id": "ORD002",
        "customer": "李四",
        "items": [
            {"name": "智能手表", "price": 1299, "quantity": 1},
        ],
        "total": 1299,
        "status": "待发货",
        "shipping_date": None,
    },
}


def lookup_order(order_id: str) -> dict:
    """查询订单信息"""
    return orders_db.get(order_id, {"error": "订单不存在"})


def calculate_refund(order_id: str, item_name: str, reason: str) -> dict:
    """计算退款金额"""
    order = orders_db.get(order_id)
    if not order:
        return {"error": "订单不存在"}
    
    for item in order["items"]:
        if item["name"] == item_name:
            refund_amount = item["price"] * item["quantity"]
            # 根据退货原因决定是否扣除运费
            if reason in ["质量问题", "商品损坏"]:
                return {
                    "refund_amount": refund_amount,
                    "shipping_fee_refund": 10,
                    "total_refund": refund_amount + 10,
                    "reason": reason,
                }
            else:
                return {
                    "refund_amount": refund_amount,
                    "shipping_fee_refund": 0,
                    "total_refund": refund_amount,
                    "reason": reason,
                }
    return {"error": "订单中未找到该商品"}


# 工具定义
tools = [
    {
        "type": "function",
        "function": {
            "name": "lookup_order",
            "description": "根据订单ID查询订单的详细信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "order_id": {
                        "type": "string",
                        "description": "订单ID，格式：ORD001",
                    },
                },
                "required": ["order_id"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "calculate_refund",
            "description": "计算退货退款金额",
            "parameters": {
                "type": "object",
                "properties": {
                    "order_id": {
                        "type": "string",
                        "description": "订单ID",
                    },
                    "item_name": {
                        "type": "string",
                        "description": "要退货的商品名称",
                    },
                    "reason": {
                        "type": "string",
                        "description": "退货原因",
                    },
                },
                "required": ["order_id", "item_name", "reason"],
            },
        },
    },
]

available_functions = {
    "lookup_order": lookup_order,
    "calculate_refund": calculate_refund,
}


def customer_service_agent():
    """客服 Agent 主流程"""
    print("=" * 60)
    print("智能客服系统 v1.0")
    print("=" * 60)
    
    # 创建客服 Assistant
    assistant = client.beta.assistants.create(
        name="智能客服小助手",
        instructions=(
            "你是一家电子产品商店的智能客服。你能够：\n"
            "1. 回答产品相关的问题\n"
            "2. 查询订单状态\n"
            "3. 处理退换货请求\n\n"
            "你应该礼貌、耐心地帮助客户解决问题。"
            "在处理退换货时，你需要先查询订单信息，然后计算退款金额。"
        ),
        model="gpt-4o",
        tools=tools,
    )
    
    # 为每个客户创建独立的 Thread
    thread = client.beta.threads.create()
    
    # 模拟客户对话
    customer_messages = [
        "你好，我想查询一下我的订单 ORD001 的状态",
        "已经发货了是吗？大概什么时候能到？",
        "对了，我想退掉那个手机壳，因为买错了型号",
    ]
    
    for message in customer_messages:
        print(f"\n客户: {message}\n")
        
        client.beta.threads.messages.create(
            thread_id=thread.id,
            role="user",
            content=message,
        )
        
        run = client.beta.threads.runs.create(
            thread_id=thread.id,
            assistant_id=assistant.id,
        )
        
        # 处理运行循环
        while run.status != "completed":
            time.sleep(1)
            run = client.beta.threads.runs.retrieve(
                thread_id=thread.id,
                run_id=run.id,
            )
            
            # 处理工具调用
            if run.status == "requires_action":
                tool_outputs = []
                for tool_call in run.required_action.submit_tool_outputs.tool_calls:
                    func_name = tool_call.function.name
                    func_args = json.loads(tool_call.function.arguments)
                    
                    result = available_functions[func_name](**func_args)
                    
                    tool_outputs.append({
                        "tool_call_id": tool_call.id,
                        "output": json.dumps(result, ensure_ascii=False),
                    })
                
                run = client.beta.threads.runs.submit_tool_outputs(
                    thread_id=thread.id,
                    run_id=run.id,
                    tool_outputs=tool_outputs,
                )
        
        # 获取回复
        messages = client.beta.threads.messages.list(thread_id=thread.id)
        for msg in messages.data:
            if msg.role == "assistant":
                for block in msg.content:
                    if block.type == "text":
                        print(f"客服: {block.text.value}")
    
    # 清理
    client.beta.assistants.delete(assistant_id=assistant.id)


if __name__ == "__main__":
    customer_service_agent()
```

这个案例展示了 Assistants API 在实际业务中的应用。关键点包括：
- 使用函数调用连接后端系统
- 使用 Thread 维护客户会话
- 使用 Assistant 的 instructions 定义服务行为

### 案例 2：多 Agent 协作系统

```python
"""
多 Agent 协作示例
一个负责规划，一个负责执行
"""

import os
import time
import json
from openai import OpenAI

client = OpenAI(api_key=os.environ.get("OPENAI_API_KEY"))


def multi_agent_collaboration():
    """多 Agent 协作示例"""
    print("=" * 60)
    print("多 Agent 协作：项目规划与执行")
    print("=" * 60)
    
    # 创建规划师 Agent
    planner = client.beta.assistants.create(
        name="项目经理",
        instructions=(
            "你是一个经验丰富的项目经理。你的任务是：\n"
            "1. 分析用户的需求\n"
            "2. 制定详细的项目计划，包括任务分解、时间安排和资源分配\n"
            "3. 将计划以结构化的格式输出\n"
            "4. 考虑潜在的风险和应对措施"
        ),
        model="gpt-4o",
    )
    
    # 创建执行者 Agent
    executor = client.beta.assistants.create(
        name="技术负责人",
        instructions=(
            "你是一个技术负责人。你的任务是：\n"
            "1. 接收项目经理提供的项目计划\n"
            "2. 将计划转化为具体的技术方案\n"
            "3. 编写关键代码片段或伪代码\n"
            "4. 评估技术风险和工期\n"
            "请用专业的技术语言输出你的方案。"
        ),
        model="gpt-4o",
    )
    
    # 模拟协作流程
    project_request = "我需要开发一个简单的待办事项Web应用，支持增删改查功能"
    
    # Step 1: 项目经理制定计划
    print("\n[Step 1] 项目经理制定计划...")
    planner_thread = client.beta.threads.create()
    
    client.beta.threads.messages.create(
        thread_id=planner_thread.id,
        role="user",
        content=f"请为以下项目制定详细计划：\n{project_request}",
    )
    
    run = client.beta.threads.runs.create(
        thread_id=planner_thread.id,
        assistant_id=planner.id,
    )
    
    while run.status != "completed":
        time.sleep(1)
        run = client.beta.threads.runs.retrieve(
            thread_id=planner_thread.id,
            run_id=run.id,
        )
    
    # 获取计划
    planner_messages = client.beta.threads.messages.list(
        thread_id=planner_thread.id
    )
    plan = ""
    for msg in planner_messages.data:
        if msg.role == "assistant":
            for block in msg.content:
                if block.type == "text":
                    plan = block.text.value
                    print(f"\n项目经理的计划:\n{plan}\n")
    
    # Step 2: 技术负责人制定技术方案
    print("\n[Step 2] 技术负责人制定技术方案...")
    executor_thread = client.beta.threads.create()
    
    client.beta.threads.messages.create(
        thread_id=executor_thread.id,
        role="user",
        content=f"请根据以下项目计划制定技术方案：\n\n{plan}",
    )
    
    run = client.beta.threads.runs.create(
        thread_id=executor_thread.id,
        assistant_id=executor.id,
    )
    
    while run.status != "completed":
        time.sleep(1)
        run = client.beta.threads.runs.retrieve(
            thread_id=executor_thread.id,
            run_id=run.id,
        )
    
    # 获取技术方案
    executor_messages = client.beta.threads.messages.list(
        thread_id=executor_thread.id
    )
    for msg in executor_messages.data:
        if msg.role == "assistant":
            for block in msg.content:
                if block.type == "text":
                    print(f"\n技术负责人的方案:\n{block.text.value}")
    
    # 清理
    client.beta.assistants.delete(assistant_id=planner.id)
    client.beta.assistants.delete(assistant_id=executor.id)
    print("\n协作完成，资源已清理")


if __name__ == "__main__":
    multi_agent_collaboration()
```

---

## 常见坑

### 坑 1：Run 轮询的效率问题

很多人在使用轮询时会设置很短的间隔（比如 100ms），这会导致大量的 API 请求，不仅浪费配额，还可能被限速。

**正确做法**：使用指数退避策略，初始间隔 1 秒，逐步增加到 5-10 秒。

```python
import time

def wait_for_run(thread_id: str, run_id: str, max_wait: int = 300):
    """使用指数退避等待 Run 完成"""
    wait_time = 1
    total_wait = 0
    
    while total_wait < max_wait:
        run = client.beta.threads.runs.retrieve(
            thread_id=thread_id,
            run_id=run_id,
        )
        
        if run.status in ["completed", "failed", "cancelled"]:
            return run
        
        if run.status == "requires_action":
            return run  # 需要工具调用，立即返回
        
        time.sleep(wait_time)
        total_wait += wait_time
        wait_time = min(wait_time * 1.5, 10)  # 最大间隔 10 秒
    
    raise TimeoutError(f"Run 在 {max_wait} 秒内未完成")
```

### 坑 2：Thread 消息数量限制

随着对话的进行，Thread 中的消息会越来越多。当消息数量接近限制时，需要实现消息清理策略。

```python
def manage_thread_messages(thread_id: str, max_messages: int = 100):
    """管理 Thread 中的消息数量"""
    messages = client.beta.threads.messages.list(thread_id=thread_id)
    
    if len(messages.data) > max_messages:
        # 保留最近的消息，删除旧的消息
        messages_to_keep = messages.data[:max_messages]
        messages_to_delete = messages.data[max_messages:]
        
        # 注意：OpenAI API 目前不支持批量删除消息
        # 需要逐个删除
        for msg in messages_to_delete:
            try:
                client.beta.threads.messages.delete(
                    thread_id=thread_id,
                    message_id=msg.id,
                )
            except Exception as e:
                print(f"删除消息失败: {e}")
```

### 坑 3：工具调用的错误处理

工具调用可能会失败，需要妥善处理各种错误情况。

```python
def safe_tool_execution(tool_call, available_functions):
    """安全执行工具调用"""
    try:
        function_name = tool_call.function.name
        function_args = json.loads(tool_call.function.arguments)
        
        if function_name not in available_functions:
            return {
                "tool_call_id": tool_call.id,
                "output": json.dumps({
                    "error": f"未知函数: {function_name}"
                }),
            }
        
        # 执行函数
        result = available_functions[function_name](**function_args)
        
        return {
            "tool_call_id": tool_call.id,
            "output": json.dumps(result, ensure_ascii=False),
        }
        
    except json.JSONDecodeError as e:
        return {
            "tool_call_id": tool_call.id,
            "output": json.dumps({
                "error": f"参数解析失败: {str(e)}"
            }),
        }
    except TypeError as e:
        return {
            "tool_call_id": tool_call.id,
            "output": json.dumps({
                "error": f"参数类型错误: {str(e)}"
            }),
        }
    except Exception as e:
        return {
            "tool_call_id": tool_call.id,
            "output": json.dumps({
                "error": f"执行失败: {str(e)}"
            }),
        }
```

### 坑 4：Assistant ID 的管理

创建的 Assistant 不会自动删除，如果不管理好，会占用资源。

```python
def cleanup_old_assistants(max_age_hours: int = 24):
    """清理过期的 Assistant"""
    import datetime
    
    assistants = client.beta.assistants.list()
    cutoff_time = datetime.datetime.now() - datetime.timedelta(hours=max_age_hours)
    
    for assistant in assistants.data:
        created_at = datetime.datetime.fromtimestamp(assistant.created_at)
        if created_at < cutoff_time:
            try:
                client.beta.assistants.delete(assistant_id=assistant.id)
                print(f"已删除过期 Assistant: {assistant.name} ({assistant.id})")
            except Exception as e:
                print(f"删除 Assistant 失败: {e}")
```

### 坑 5：忽略 Run 的 status 检查

Run 可能因为多种原因失败（模型拒绝、超时、配额不足等），必须检查 status 而不是假设它总是成功。

```python
def robust_run_execution(thread_id: str, assistant_id: str):
    """健壮的 Run 执行"""
    run = client.beta.threads.runs.create(
        thread_id=thread_id,
        assistant_id=assistant_id,
    )
    
    while True:
        time.sleep(1)
        run = client.beta.threads.runs.retrieve(
            thread_id=thread.id,
            run_id=run.id,
        )
        
        if run.status == "completed":
            return run
        elif run.status == "failed":
            error_msg = run.last_error.message if run.last_error else "未知错误"
            raise RuntimeError(f"Run 失败: {error_msg}")
        elif run.status == "cancelled":
            raise RuntimeError("Run 被取消")
        elif run.status == "expired":
            raise RuntimeError("Run 已过期")
        elif run.status == "requires_action":
            return run  # 需要工具调用
        # queued, in_progress 继续等待
```

---

## 练习题

### 练习 1：基础对话实现
创建一个学习助手 Assistant，它能够：
- 回答 Python 编程相关的问题
- 在回答后主动询问用户是否理解
- 支持至少 5 轮的连续对话

提示：注意管理 Thread 的消息上下文。

### 练习 2：文件处理助手
创建一个文件处理助手，它能够：
- 接受用户上传的文本文件
- 分析文件的统计信息（行数、单词数、字符数）
- 搜索文件中的特定内容
- 生成文件的摘要

提示：使用代码解释器工具处理文件分析，使用文件搜索工具进行内容搜索。

### 练习 3：旅行规划器
创建一个旅行规划助手，它能够：
- 查询目的地天气
- 搜索航班信息
- 推荐餐厅
- 生成完整的旅行计划文档

提示：定义至少 3 个自定义函数作为工具。

### 练习 4：多助手协作
设计一个文档审核系统：
- 助手 A 负责检查语法和拼写错误
- 助手 B 负责检查逻辑和内容一致性
- 助手 C 负责生成最终的修改建议

提示：一个助手的输出作为另一个助手的输入。

### 练习 5：流式输出应用
创建一个故事创作助手，使用流式输出实时展示创作过程：
- 支持多种故事类型（科幻、悬疑、爱情）
- 根据用户的关键词创作故事
- 实时显示创作进度
- 支持用户中途修改方向

提示：使用 `client.beta.threads.runs.stream()` 方法。

### 练习 6：错误恢复机制
为你的 Assistant 应用实现完整的错误恢复机制：
- 网络错误时自动重试
- Run 失败时的降级处理
- 工具调用失败时的备选方案
- 配额超限时的等待策略

---

## 实战任务

### 任务 1：构建一个完整的客服系统（中等难度）

为一家小型在线商店构建客服系统：

**功能需求：**
1. 自动回答常见问题（FAQ）
2. 查询订单状态
3. 处理退换货请求
4. 记录客户反馈

**技术要求：**
1. 使用 Assistants API 的函数调用工具
2. 为每个客户创建独立的 Thread
3. 实现消息持久化（存储 Thread ID）
4. 添加错误处理和日志记录

**扩展挑战：**
- 实现客户满意度评价
- 添加转接人工客服的功能
- 实现多语言支持

### 任务 2：构建一个数据分析平台（高难度）

使用 Assistants API 构建一个简单的数据分析平台：

**功能需求：**
1. 支持上传 CSV/JSON 数据文件
2. 自动生成数据摘要和统计报告
3. 根据用户描述生成数据可视化
4. 支持自然语言查询数据

**技术要求：**
1. 使用代码解释器处理数据
2. 使用文件搜索实现数据查询
3. 实现数据导出功能
4. 添加用户认证和权限管理

---

## 本章小结

本章我们深入学习了 OpenAI Assistants API，这是目前最成熟的官方 Agent 实现方案之一。我们从以下几个方面进行了探索：

**核心概念**：理解了 Thread、Assistant、Run 三个核心概念及其关系。Thread 是对话的容器，Assistant 是 Agent 的人格和能力定义，Run 是一次执行过程。

**内置工具**：掌握了代码解释器、文件搜索和函数调用三种工具的使用方式。这些工具让 Assistant 能够执行代码、检索文档和调用外部 API。

**对话管理**：学会了如何使用 Thread 实现持久化的多轮对话，以及如何管理对话历史。

**实际应用**：通过客服系统和多 Agent 协作等案例，看到了 Assistants API 在实际业务中的应用。

**最佳实践**：了解了常见的坑和解决方案，包括轮询效率、消息管理、错误处理等。

**Assistants API 的优势**在于它的简单性和托管特性。你不需要自己管理状态、处理工具调用循环、或者担心对话持久化。API 帮你处理了所有这些基础设施问题，让你可以专注于业务逻辑。

**Assistants API 的局限**在于它只能使用 OpenAI 的模型，而且所有的数据都存储在 OpenAI 的服务器上，这在某些场景下可能是一个问题。此外，它的定价相对较高，特别是在需要频繁使用代码解释器或文件搜索时。

在下一章中，我们将学习 Anthropic Claude 的工具使用方式，看看另一种 Agent 实现范式。

---

## 延伸阅读

1. OpenAI Assistants API 官方文档：https://platform.openai.com/docs/assistants/overview
2. Assistants API 入门指南：https://platform.openai.com/docs/assistants/quickstart
3. 代码解释器工具详解：https://platform.openai.com/docs/assistants/tools/code-interpreter
4. 文件搜索工具详解：https://platform.openai.com/docs/assistants/tools/file-search
5. 函数调用工具详解：https://platform.openai.com/docs/assistants/tools/function-calling
