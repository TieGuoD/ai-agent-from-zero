# 第 3 章：LLM API 工程 —— 从调用到生产

> **本章定位：** Agent 的底层是 LLM API 调用。不掌握 API 的工程细节（重试、限流、成本控制、错误处理），就无法构建可靠的 Agent。本章将带你从简单的 API 调用到生产级的工程实践，构建一个健壮的 LLM API 客户端。

---

## 学习目标

1. 掌握 OpenAI API 和 Claude API 的核心用法和差异
2. 理解 Streaming、Function Calling、JSON Mode 等高级特性
3. 掌握 API 调用的工程最佳实践（重试、超时、限流、成本追踪）
4. 理解多模型切换的架构设计
5. 能构建一个生产级的 LLM API 客户端

## 核心问题

1. API 调用失败了怎么办？重试策略如何设计？
2. 如何控制 LLM API 的调用成本？
3. 如何在不修改业务代码的情况下切换不同的 LLM 提供商？

---

## 3.1 OpenAI API 详解

### 3.1.1 核心概念

OpenAI API 的核心是 Chat Completions 接口。让我们逐个拆解每个参数的含义。

```python
from openai import OpenAI
import os

client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

response = client.chat.completions.create(
    model="gpt-4o",           # 使用的模型
    messages=[                 # 消息列表
        {"role": "system", "content": "你是一个助手"},  # 系统消息
        {"role": "user", "content": "你好"},             # 用户消息
    ],
    temperature=0.7,           # 随机性控制
    max_tokens=1000,           # 最大输出长度
    top_p=1.0,                 # 核采样参数
    frequency_penalty=0,       # 频率惩罚
    presence_penalty=0,        # 存在惩罚
    stop=None,                 # 停止标记
    stream=False,              # 是否流式输出
)
```

**消息角色详解：**

| 角色 | 说明 | 用途 | 在 Agent 中的作用 |
|------|------|------|------------------|
| system | 系统消息 | 定义 Agent 行为 | System Prompt |
| user | 用户消息 | 用户输入 | 用户的请求 |
| assistant | 助手消息 | LLM 的回复 | Agent 的回答 |
| tool | 工具消息 | 工具调用结果 | 工具返回的数据 |

**响应结构详解：**

```python
# 响应对象的结构
response = client.chat.completions.create(...)

# 基本信息
print(response.id)           # "chatcmpl-abc123"
print(response.model)        # "gpt-4o"
print(response.object)       # "chat.completion"

# 生成的内容
content = response.choices[0].message.content  # LLM 的回复文本

# Token 使用情况
usage = response.usage
print(f"输入 Token：{usage.prompt_tokens}")      # 输入消耗的 Token
print(f"输出 Token：{usage.completion_tokens}")   # 输出消耗的 Token
print(f"总计 Token：{usage.total_tokens}")        # 总消耗

# 完成原因
finish_reason = response.choices[0].finish_reason
# "stop": 正常结束
# "length": 达到 max_tokens 限制
# "tool_calls": 需要调用工具
```

### 3.1.2 Function Calling

Function Calling 是 OpenAI 的工具调用机制。它让 LLM 能够"调用"外部函数，获取真实数据。

**为什么需要 Function Calling？**

```
没有 Function Calling：
  用户：北京天气怎么样？
  LLM：北京今天可能天气不错。（可能是幻觉）

有 Function Calling：
  用户：北京天气怎么样？
  LLM：我来查询一下。[get_weather(city="北京")]
  工具返回：晴，25°C
  LLM：北京今天晴，温度 25°C。（基于真实数据）
```

**完整的 Function Calling 流程：**

```python
# 1. 定义工具
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "获取指定城市的当前天气信息。当用户询问天气时使用此工具。",
            "parameters": {
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
                "required": ["city"],
            },
        },
    }
]

# 2. 调用 LLM
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "北京天气怎么样？"}],
    tools=tools,
    tool_choice="auto",  # 让 LLM 自动决定是否调用工具
)

# 3. 检查是否需要调用工具
message = response.choices[0].message
if message.tool_calls:
    # LLM 决定调用工具
    tool_call = message.tool_calls[0]
    print(f"工具名称：{tool_call.function.name}")        # "get_weather"
    print(f"工具参数：{tool_call.function.arguments}")    # '{"city": "北京"}'

    # 4. 执行工具
    import json
    args = json.loads(tool_call.function.arguments)
    weather_result = get_weather(**args)  # 调用实际的天气函数

    # 5. 将工具结果反馈给 LLM
    messages = [
        {"role": "user", "content": "北京天气怎么样？"},
        message,  # LLM 的工具调用请求
        {
            "role": "tool",
            "tool_call_id": tool_call.id,
            "content": json.dumps(weather_result),  # 工具返回的结果
        },
    ]

    # 6. 再次调用 LLM 生成最终回答
    final_response = client.chat.completions.create(
        model="gpt-4o",
        messages=messages,
    )
    print(final_response.choices[0].message.content)
    # "北京今天晴，温度 25°C，适合外出。"
```

**工具描述的重要性：**

工具描述是 LLM 决定是否使用工具的唯一依据。描述质量直接影响调用准确性。

```python
# 好的工具描述
{
    "name": "search_web",
    "description": "在互联网上搜索信息。当需要获取实时信息、最新新闻或不确定的事实时使用此工具。不要用于计算或数学问题。",
}

# 差的工具描述
{
    "name": "search_web",
    "description": "搜索",  # 太简短，LLM 不知道什么时候用
}
```

### 3.1.3 JSON Mode

强制 LLM 输出有效的 JSON：

```python
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "你是一个数据分析师，输出 JSON 格式的数据"},
        {"role": "user", "content": "分析以下销售数据：Q1: 100万, Q2: 120万, Q3: 150万, Q4: 180万"},
    ],
    response_format={"type": "json_object"},  # 强制 JSON 输出
)

# 输出保证是有效的 JSON
print(response.choices[0].message.content)
# {"analysis": {"trend": "持续增长", "q1_to_q4_growth": "80%", ...}}
```

### 3.1.4 流式输出

流式输出让用户能立即看到 LLM 的生成过程：

```python
# 流式调用
stream = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "写一个关于 AI 的故事"}],
    stream=True,
)

# 逐字输出
full_response = ""
for chunk in stream:
    if chunk.choices[0].delta.content:
        content = chunk.choices[0].delta.content
        print(content, end="", flush=True)
        full_response += content
```

**流式输出在 Agent 中的价值：**

1. **用户体验：** 用户立即看到生成过程，不用等待
2. **实时反馈：** 可以在生成过程中进行干预
3. **工具调用检测：** 可以在流式输出中检测工具调用

---

## 3.2 Claude API 详解

### 3.2.1 核心概念

Claude API 使用 Messages 接口，与 OpenAI 有显著差异。

```python
import anthropic
import os

client = anthropic.Anthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))

response = client.messages.create(
    model="claude-sonnet-4-6-20250514",
    max_tokens=1024,
    system="你是一个助手",  # System Prompt 作为单独参数
    messages=[
        {"role": "user", "content": "你好"},
    ],
)

print(response.content[0].text)
```

**与 OpenAI 的主要区别：**

| 特性 | OpenAI | Claude |
|------|--------|--------|
| System Prompt | 在 messages 中 | 单独的 system 参数 |
| 工具定义 | tools 参数 | tools 参数（格式不同） |
| 响应格式 | response.choices[0].message.content | response.content[0].text |
| 流式输出 | stream=True | stream=True |
| 最大输出 | max_tokens | max_tokens |

### 3.2.2 Claude 工具使用

```python
tools = [
    {
        "name": "get_weather",
        "description": "获取指定城市的天气",
        "input_schema": {  # 注意：Claude 使用 input_schema
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "城市名称"},
            },
            "required": ["city"],
        },
    }
]

response = client.messages.create(
    model="claude-sonnet-4-6-20250514",
    max_tokens=1024,
    system="你是一个天气助手",
    messages=[{"role": "user", "content": "北京天气怎么样？"}],
    tools=tools,
)

# 检查工具调用
for content in response.content:
    if content.type == "tool_use":
        print(f"工具：{content.name}")
        print(f"参数：{content.input}")
```

### 3.2.3 Extended Thinking

Claude 的 Extended Thinking 功能让模型在回答前进行深入思考：

```python
response = client.messages.create(
    model="claude-sonnet-4-6-20250514",
    max_tokens=16000,
    thinking={
        "type": "enabled",
        "budget_tokens": 10000,  # 给思考过程分配的 Token 预算
    },
    messages=[{"role": "user", "content": "解决这个复杂数学问题..."}],
)

# 响应包含思考过程和最终答案
for content in response.content:
    if content.type == "thinking":
        print(f"思考过程：{content.thinking}")
    elif content.type == "text":
        print(f"最终答案：{content.text}")
```

---

## 3.3 统一 API 封装

### 3.3.1 为什么需要统一封装

Agent 代码不应该绑定到特定的 LLM 提供商。通过统一的封装层，可以：
1. 在不修改业务代码的情况下切换提供商
2. 统一错误处理和重试逻辑
3. 统一成本追踪

### 3.3.2 设计接口

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import Generator


@dataclass
class LLMResponse:
    """统一的 LLM 响应格式"""
    content: str
    model: str
    usage: dict  # {"prompt_tokens": int, "completion_tokens": int, "total_tokens": int}
    tool_calls: list[dict] | None = None
    finish_reason: str = ""


class LLMProvider(ABC):
    """LLM 提供商基类"""

    @abstractmethod
    def chat(
        self,
        messages: list[dict],
        model: str | None = None,
        temperature: float = 0.7,
        max_tokens: int = 4096,
        tools: list[dict] | None = None,
    ) -> LLMResponse:
        """同步聊天"""
        pass

    @abstractmethod
    def chat_stream(
        self,
        messages: list[dict],
        model: str | None = None,
        temperature: float = 0.7,
        max_tokens: int = 4096,
    ) -> Generator[str, None, None]:
        """流式聊天"""
        pass
```

### 3.3.3 OpenAI 实现

```python
from openai import OpenAI


class OpenAIProvider(LLMProvider):
    def __init__(self, api_key: str, default_model: str = "gpt-4o"):
        self.client = OpenAI(api_key=api_key)
        self.default_model = default_model

    def chat(
        self,
        messages: list[dict],
        model: str | None = None,
        temperature: float = 0.7,
        max_tokens: int = 4096,
        tools: list[dict] | None = None,
    ) -> LLMResponse:
        kwargs = {
            "model": model or self.default_model,
            "messages": messages,
            "temperature": temperature,
            "max_tokens": max_tokens,
        }
        if tools:
            kwargs["tools"] = tools

        response = self.client.chat.completions.create(**kwargs)

        # 解析工具调用
        tool_calls = None
        if response.choices[0].message.tool_calls:
            tool_calls = [
                {
                    "name": tc.function.name,
                    "arguments": tc.function.arguments,
                    "id": tc.id,
                }
                for tc in response.choices[0].message.tool_calls
            ]

        return LLMResponse(
            content=response.choices[0].message.content or "",
            model=response.model,
            usage={
                "prompt_tokens": response.usage.prompt_tokens,
                "completion_tokens": response.usage.completion_tokens,
                "total_tokens": response.usage.total_tokens,
            },
            tool_calls=tool_calls,
            finish_reason=response.choices[0].finish_reason or "",
        )

    def chat_stream(
        self,
        messages: list[dict],
        model: str | None = None,
        temperature: float = 0.7,
        max_tokens: int = 4096,
    ) -> Generator[str, None, None]:
        stream = self.client.chat.completions.create(
            model=model or self.default_model,
            messages=messages,
            temperature=temperature,
            max_tokens=max_tokens,
            stream=True,
        )
        for chunk in stream:
            if chunk.choices[0].delta.content:
                yield chunk.choices[0].delta.content
```

### 3.3.4 Claude 实现

```python
import anthropic


class ClaudeProvider(LLMProvider):
    def __init__(self, api_key: str, default_model: str = "claude-sonnet-4-6-20250514"):
        self.client = anthropic.Anthropic(api_key=api_key)
        self.default_model = default_model

    def chat(
        self,
        messages: list[dict],
        model: str | None = None,
        temperature: float = 0.7,
        max_tokens: int = 4096,
        tools: list[dict] | None = None,
    ) -> LLMResponse:
        # 分离 system 消息
        system_msg = ""
        user_messages = []
        for msg in messages:
            if msg["role"] == "system":
                system_msg = msg["content"]
            else:
                user_messages.append(msg)

        kwargs = {
            "model": model or self.default_model,
            "max_tokens": max_tokens,
            "messages": user_messages,
        }
        if system_msg:
            kwargs["system"] = system_msg
        if tools:
            kwargs["tools"] = tools

        response = self.client.messages.create(**kwargs)

        # 解析响应
        content = ""
        tool_calls = []
        for block in response.content:
            if block.type == "text":
                content += block.text
            elif block.type == "tool_use":
                tool_calls.append({
                    "name": block.name,
                    "arguments": block.input,
                    "id": block.id,
                })

        return LLMResponse(
            content=content,
            model=response.model,
            usage={
                "prompt_tokens": response.usage.input_tokens,
                "completion_tokens": response.usage.output_tokens,
                "total_tokens": response.usage.input_tokens + response.usage.output_tokens,
            },
            tool_calls=tool_calls if tool_calls else None,
            finish_reason=response.stop_reason or "",
        )

    def chat_stream(
        self,
        messages: list[dict],
        model: str | None = None,
        temperature: float = 0.7,
        max_tokens: int = 4096,
    ) -> Generator[str, None, None]:
        system_msg = ""
        user_messages = []
        for msg in messages:
            if msg["role"] == "system":
                system_msg = msg["content"]
            else:
                user_messages.append(msg)

        kwargs = {
            "model": model or self.default_model,
            "max_tokens": max_tokens,
            "messages": user_messages,
        }
        if system_msg:
            kwargs["system"] = system_msg

        with self.client.messages.stream(**kwargs) as stream:
            for text in stream.text_stream:
                yield text
```

---

## 3.4 错误处理与重试

### 3.4.1 常见错误类型

| 错误码 | 含义 | 是否重试 | 处理策略 |
|--------|------|---------|---------|
| 400 | 请求错误 | 否 | 修复请求参数 |
| 401 | 认证失败 | 否 | 检查 API Key |
| 403 | 权限不足 | 否 | 检查权限 |
| 408 | 请求超时 | 是 | 重试 |
| 429 | 限流 | 是 | 指数退避重试 |
| 500 | 服务器错误 | 是 | 重试 |
| 502 | 网关错误 | 是 | 重试 |
| 503 | 服务不可用 | 是 | 等待后重试 |

### 3.4.2 指数退避重试

指数退避（Exponential Backoff）是一种处理暂时性错误的策略：每次重试时，等待时间指数增长。

```python
import time
import random
from functools import wraps


def retry_with_backoff(max_retries=3, base_delay=1, max_delay=60):
    """指数退避重试装饰器"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_retries - 1:
                        raise

                    # 计算延迟：base_delay * 2^attempt + 随机抖动
                    delay = min(
                        base_delay * (2 ** attempt) + random.uniform(0, 1),
                        max_delay
                    )
                    print(f"重试 {attempt + 1}/{max_retries}，等待 {delay:.1f} 秒...")
                    time.sleep(delay)
            return None
        return wrapper
    return decorator
```

**为什么需要随机抖动？**

如果没有随机抖动，多个客户端会在完全相同的时间重试，导致"惊群效应"（Thundering Herd），进一步加剧限流。

### 3.4.3 完整的错误处理实现

```python
import logging
import time
import random
from openai import OpenAI, APIError, RateLimitError, APITimeoutError, APIConnectionError

logger = logging.getLogger(__name__)


class LLMClient:
    def __init__(self, api_key: str, max_retries: int = 3, timeout: float = 30.0):
        self.client = OpenAI(api_key=api_key, timeout=timeout)
        self.max_retries = max_retries

    def chat(self, messages: list[dict], model: str = "gpt-4o", **kwargs):
        last_exception = None

        for attempt in range(self.max_retries):
            try:
                response = self.client.chat.completions.create(
                    model=model,
                    messages=messages,
                    **kwargs,
                )
                return response

            except RateLimitError as e:
                # 限流：指数退避
                wait_time = min(2 ** attempt + random.uniform(0, 1), 60)
                logger.warning(f"限流，等待 {wait_time:.1f} 秒后重试 (尝试 {attempt + 1}/{self.max_retries})")
                time.sleep(wait_time)
                last_exception = e

            except APITimeoutError as e:
                # 超时：重试
                logger.warning(f"请求超时，重试 {attempt + 1}/{self.max_retries}")
                last_exception = e

            except APIConnectionError as e:
                # 连接错误：重试
                logger.warning(f"连接错误，重试 {attempt + 1}/{self.max_retries}")
                time.sleep(2 ** attempt)
                last_exception = e

            except APIError as e:
                # API 错误：5xx 重试，4xx 不重试
                if e.status_code and e.status_code >= 500:
                    logger.warning(f"服务器错误 {e.status_code}，重试")
                    time.sleep(2 ** attempt)
                    last_exception = e
                else:
                    logger.error(f"API 错误：{e.status_code} - {e.message}")
                    raise

            except Exception as e:
                # 未知错误：不重试
                logger.error(f"未知错误：{e}")
                raise

        # 所有重试都失败
        raise Exception(f"重试 {self.max_retries} 次后仍然失败") from last_exception
```

---

## 3.5 成本控制

### 3.5.1 Token 计费模型

LLM API 按 Token 计费。理解计费模型是成本控制的基础。

```python
# 价格参考（请查阅最新官方定价）
PRICING = {
    "gpt-4o": {"input": 2.50, "output": 10.00},      # 每百万 Token
    "gpt-4o-mini": {"input": 0.15, "output": 0.60},
    "claude-sonnet-4-6": {"input": 3.00, "output": 15.00},
    "claude-haiku-4-5": {"input": 0.80, "output": 4.00},
}


def calculate_cost(model: str, input_tokens: int, output_tokens: int) -> float:
    """计算 API 调用成本（美元）"""
    if model not in PRICING:
        return 0.0

    pricing = PRICING[model]
    cost = (
        input_tokens * pricing["input"] / 1_000_000
        + output_tokens * pricing["output"] / 1_000_000
    )
    return cost


# 示例
cost = calculate_cost("gpt-4o", input_tokens=1000, output_tokens=500)
print(f"本次调用成本：${cost:.6f}")  # $0.0075
```

### 3.5.2 成本追踪器

```python
from dataclasses import dataclass, field
from datetime import datetime


@dataclass
class CostRecord:
    timestamp: str
    model: str
    input_tokens: int
    output_tokens: int
    cost: float


class CostTracker:
    def __init__(self, budget: float = 100.0):
        self.budget = budget
        self.records: list[CostRecord] = []
        self.total_cost = 0.0

    def record(self, model: str, input_tokens: int, output_tokens: int):
        """记录一次 API 调用的成本"""
        cost = calculate_cost(model, input_tokens, output_tokens)
        self.total_cost += cost

        record = CostRecord(
            timestamp=datetime.now().isoformat(),
            model=model,
            input_tokens=input_tokens,
            output_tokens=output_tokens,
            cost=cost,
        )
        self.records.append(record)

        # 检查预算
        if self.total_cost > self.budget * 0.8:
            logger.warning(f"成本已达到预算的 {self.total_cost/self.budget*100:.1f}%")

        return cost

    def get_summary(self) -> dict:
        """获取成本摘要"""
        return {
            "total_cost": f"${self.total_cost:.4f}",
            "budget": f"${self.budget:.2f}",
            "budget_used": f"{self.total_cost/self.budget*100:.1f}%",
            "total_calls": len(self.records),
            "by_model": self._by_model(),
        }

    def _by_model(self) -> dict:
        """按模型统计成本"""
        by_model = {}
        for record in self.records:
            if record.model not in by_model:
                by_model[record.model] = {"cost": 0, "calls": 0}
            by_model[record.model]["cost"] += record.cost
            by_model[record.model]["calls"] += 1
        return by_model
```

### 3.5.3 成本优化策略

**策略 1：模型路由**

根据任务复杂度选择合适的模型。

```python
class CostAwareRouter:
    def __init__(self):
        self.routes = {
            "simple": ("gpt-4o-mini", 0.15),    # 简单任务用便宜模型
            "complex": ("gpt-4o", 2.50),         # 复杂任务用强模型
            "code": ("claude-sonnet-4-6", 3.00), # 代码任务用 Claude
        }

    def route(self, task_type: str) -> str:
        model, cost = self.routes.get(task_type, ("gpt-4o", 2.50))
        return model
```

**策略 2：缓存**

对相同或相似的请求缓存结果。

```python
import hashlib
import json


class ResponseCache:
    def __init__(self, max_size: int = 1000):
        self.cache = {}
        self.max_size = max_size

    def _make_key(self, messages: list[dict], model: str) -> str:
        content = json.dumps(messages, sort_keys=True) + model
        return hashlib.md5(content.encode()).hexdigest()

    def get(self, messages: list[dict], model: str) -> str | None:
        key = self._make_key(messages, model)
        return self.cache.get(key)

    def set(self, messages: list[dict], model: str, response: str):
        if len(self.cache) >= self.max_size:
            # 简单的 LRU：删除最早的条目
            oldest_key = next(iter(self.cache))
            del self.cache[oldest_key]

        key = self._make_key(messages, model)
        self.cache[key] = response
```

**策略 3：输出长度控制**

设置合理的 max_tokens，避免生成过长的输出。

```python
# 根据任务类型设置不同的 max_tokens
MAX_TOKENS_CONFIG = {
    "classification": 50,     # 分类任务：简短输出
    "summary": 500,           # 摘要任务：中等长度
    "generation": 2000,       # 生成任务：较长输出
    "code": 4000,             # 代码生成：可能很长
}
```

---

## 3.6 流式输出

### 3.6.1 流式输出的价值

流式输出（Streaming）让用户能立即看到 LLM 的生成过程，而不是等待完整响应。

```
同步调用：用户等待 3-5 秒 → 看到完整回答
流式调用：用户立即看到 → 逐字显示 → 体验更好
```

### 3.6.2 流式输出在 Agent 中的应用

```python
class StreamingAgent:
    def __init__(self, llm_client):
        self.llm_client = llm_client

    def chat_stream(self, user_input: str):
        """支持流式输出的 Agent"""
        messages = [{"role": "user", "content": user_input}]

        # 流式调用
        full_response = ""
        for chunk in self.llm_client.chat_stream(messages):
            full_response += chunk
            yield chunk  # 实时返回给用户

        # 检查是否需要工具调用
        # ...（工具调用逻辑）
```

---

## 3.7 多模型路由

### 3.7.1 为什么需要多模型路由

不同任务对 LLM 的要求不同。使用多模型路由可以：
1. 降低成本（简单任务用便宜模型）
2. 提高质量（复杂任务用强模型）
3. 提高可用性（一个提供商不可用时切换到另一个）

### 3.7.2 路由策略实现

```python
class ModelRouter:
    def __init__(self, providers: dict[str, LLMProvider]):
        self.providers = providers
        self.default_provider = list(providers.keys())[0]

    def route(self, task_type: str) -> tuple[str, LLMProvider]:
        """根据任务类型路由到合适的提供商"""
        routing_table = {
            "simple": "openai_mini",
            "complex": "openai",
            "code": "claude",
            "chinese": "qwen",
        }

        provider_name = routing_table.get(task_type, self.default_provider)
        return provider_name, self.providers[provider_name]

    def classify_task(self, user_input: str) -> str:
        """简单任务分类"""
        # 实际项目中可以用 LLM 或规则来分类
        input_lower = user_input.lower()

        if any(kw in input_lower for kw in ["代码", "编程", "code", "python"]):
            return "code"
        elif len(user_input) < 20:
            return "simple"
        else:
            return "complex"
```

---

## 3.8 案例分析：构建一个健壮的 LLM 客户端

### 需求

构建一个生产级的 LLM 客户端，支持：
1. 多模型切换
2. 重试和错误处理
3. 成本追踪
4. 流式输出
5. 响应缓存

### 完整实现

```python
import os
import time
import random
import logging
from typing import Generator
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv()
logger = logging.getLogger(__name__)


class ProductionLLMClient:
    """生产级 LLM 客户端"""

    def __init__(
        self,
        api_key: str | None = None,
        default_model: str = "gpt-4o",
        max_retries: int = 3,
        timeout: float = 30.0,
        budget: float = 100.0,
    ):
        self.api_key = api_key or os.getenv("OPENAI_API_KEY")
        self.client = OpenAI(api_key=self.api_key, timeout=timeout)
        self.default_model = default_model
        self.max_retries = max_retries
        self.cost_tracker = CostTracker(budget=budget)
        self.cache = ResponseCache()

    def chat(
        self,
        messages: list[dict],
        model: str | None = None,
        temperature: float = 0.7,
        max_tokens: int = 4096,
        use_cache: bool = True,
    ) -> str:
        """同步聊天"""
        model = model or self.default_model

        # 检查缓存
        if use_cache:
            cached = self.cache.get(messages, model)
            if cached:
                logger.info("缓存命中")
                return cached

        # 调用 LLM（带重试）
        response = self._call_with_retry(messages, model, temperature, max_tokens)

        # 记录成本
        self.cost_tracker.record(
            model=model,
            input_tokens=response.usage.prompt_tokens,
            output_tokens=response.usage.completion_tokens,
        )

        content = response.choices[0].message.content

        # 缓存结果
        if use_cache:
            self.cache.set(messages, model, content)

        return content

    def chat_stream(
        self,
        messages: list[dict],
        model: str | None = None,
        temperature: float = 0.7,
        max_tokens: int = 4096,
    ) -> Generator[str, None, None]:
        """流式聊天"""
        model = model or self.default_model

        stream = self.client.chat.completions.create(
            model=model,
            messages=messages,
            temperature=temperature,
            max_tokens=max_tokens,
            stream=True,
        )

        for chunk in stream:
            if chunk.choices[0].delta.content:
                yield chunk.choices[0].delta.content

    def _call_with_retry(self, messages, model, temperature, max_tokens):
        """带重试的 LLM 调用"""
        for attempt in range(self.max_retries):
            try:
                return self.client.chat.completions.create(
                    model=model,
                    messages=messages,
                    temperature=temperature,
                    max_tokens=max_tokens,
                )
            except Exception as e:
                if attempt == self.max_retries - 1:
                    raise
                wait_time = min(2 ** attempt + random.uniform(0, 1), 60)
                logger.warning(f"重试 {attempt + 1}，等待 {wait_time:.1f}秒")
                time.sleep(wait_time)

    def get_cost_summary(self) -> dict:
        """获取成本摘要"""
        return self.cost_tracker.get_summary()
```

---

## 3.9 常见坑与最佳实践

### 3.9.1 常见坑

1. **不处理限流：** 429 错误是生产环境最常见的问题。没有重试机制的 Agent 在高并发下会频繁失败。

2. **不追踪成本：** 没有成本追踪，月底可能收到意外账单。一个不小心的循环调用可能花费数百美元。

3. **硬编码 API Key：** 安全风险。如果代码泄露，API Key 也会泄露。

4. **不处理超时：** 长时间没有响应会导致用户流失。设置合理的超时时间很重要。

5. **不使用流式输出：** 用户体验差。在 Agent 中，流式输出几乎是必须的。

### 3.9.2 最佳实践

1. **使用环境变量管理 API Key** —— 永远不要硬编码
2. **实现指数退避重试** —— 处理暂时性错误
3. **追踪每次调用的成本** —— 监控预算
4. **设置超时时间** —— 防止长时间等待
5. **使用流式输出** —— 提升用户体验
6. **实现多模型路由** —— 优化成本和质量
7. **使用缓存** —— 减少重复调用

---

## 3.10 练习题

### 概念理解题

1. OpenAI 和 Claude API 的主要区别是什么？在 Agent 开发中，这些区别会带来什么影响？

2. 指数退避重试的原理是什么？为什么需要随机抖动？

3. 如何设计一个成本控制策略？请列出至少 3 种方法。

4. 流式输出在 Agent 中有什么价值？除了用户体验，还有什么好处？

5. 多模型路由的优势是什么？如何设计路由策略？

### 动手实践题

1. **构建 API 客户端：** 使用本章的代码，构建一个支持 OpenAI 和 Claude 的统一 API 客户端。

2. **重试实验：** 模拟 API 调用失败的情况，测试重试机制是否正常工作。

3. **成本追踪：** 运行 10 次 API 调用，记录每次的成本，生成成本报告。

### 思考题

1. 如果你的 Agent 需要处理每秒 100 个请求，你会如何设计 LLM API 的调用策略？

2. 如何在成本和质量之间找到平衡？你会如何决定什么时候用便宜模型，什么时候用贵模型？

---

## 3.11 实战任务

**任务：构建一个带重试、限流、成本追踪的 LLM API 客户端**

**具体要求：**

1. 实现统一的 LLM API 封装（支持 OpenAI 和 Claude）
2. 实现指数退避重试
3. 实现成本追踪
4. 实现流式输出
5. 实现响应缓存
6. 编写单元测试
7. 生成成本报告

**评估标准：**

| 维度 | 权重 | 评分标准 |
|------|------|---------|
| 代码正确性 | 25% | 功能正常，错误处理完善 |
| 重试机制 | 20% | 指数退避正确实现 |
| 成本追踪 | 20% | 准确记录每次调用成本 |
| 流式输出 | 15% | 流式输出正常工作 |
| 测试覆盖 | 10% | 核心功能有测试 |
| 代码质量 | 10% | 结构清晰，可维护 |

---

## 3.12 本章小结

### 核心知识点回顾

- **OpenAI 和 Claude API：** 两者有显著差异（System Prompt 格式、工具定义格式、响应结构），需要统一封装。

- **Function Calling：** 让 LLM 能调用外部工具获取真实数据。工具描述的质量直接影响调用准确性。

- **错误处理：** 指数退避重试是处理暂时性错误的标准策略。需要区分可重试错误和不可重试错误。

- **成本控制：** 模型路由、缓存、输出长度控制是主要优化手段。成本追踪是基础。

- **流式输出：** 提升用户体验，在 Agent 中几乎是必须的。

### 关键公式

```
生产级 LLM 客户端 = 统一接口 + 重试机制 + 成本追踪 + 流式输出 + 缓存
```

### 下一章预告

下一章我们将深入探讨 **Tool Calling** —— Agent 从"只会说话"到"能做事"的关键转折。你将学习如何设计工具、注册工具、调用工具，以及如何处理工具调用的各种边界情况。

---

*上一章：[第 2 章：Prompt Engineering](../chapter-02/README.md)*
*下一章：[第 4 章：Tool Calling](../chapter-04/README.md)*
