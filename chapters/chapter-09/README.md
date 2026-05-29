# 第 9 章：Agent 的错误处理与容错

> **本章定位：** Agent 在执行过程中会遇到各种错误——LLM 调用失败、工具执行出错、推理错误、状态丢失。没有容错机制的 Agent 在生产环境中就像一辆没有刹车的汽车，迟早会出事。本章将带你深入理解 Agent 的错误类型，掌握错误恢复策略，以及如何设计 Agent 的自我纠错机制。

---

## 学习目标

1. **理解 Agent 常见错误类型** —— 能区分 LLM 错误、工具错误、推理错误、状态错误、安全错误
2. **掌握错误恢复策略** —— 能实现重试、降级、回退等策略
3. **理解 Agent 的自我纠错机制** —— 能让 Agent 从错误中学习和修正
4. **掌握错误日志和告警设计** —— 能设计完整的错误监控体系

## 核心问题

1. **Agent 会遇到哪些类型的错误？** 每种错误的原因和后果是什么？
2. **如何让 Agent 从错误中恢复？** 重试、降级、回退各适用于什么场景？
3. **如何设计 Agent 的错误处理策略？** 如何平衡错误处理的成本和效果？

---

## 9.1 Agent 会遇到哪些错误

### 9.1.1 一个真实的错误场景

想象你正在使用一个研究助手 Agent，让它帮你搜索关于"量子计算最新进展"的信息。

**场景 1：LLM API 调用失败**

```
用户：搜索量子计算最新进展
Agent：（调用 LLM）
LLM API：429 Too Many Requests
Agent：???
```

这是最常见的错误。OpenAI 的 API 有速率限制，当请求过多时会返回 429 错误。如果没有重试机制，Agent 直接崩溃。

**场景 2：工具执行失败**

```
Agent：（调用搜索工具）
搜索 API：网络超时
Agent：???
```

外部工具（如搜索 API、天气 API）可能因为网络问题、服务不可用等原因失败。如果 Agent 不处理这种情况，用户会看到一个错误信息。

**场景 3：推理错误**

```
用户：北京天气怎么样？
Agent：（思考）用户在问数学问题，我应该调用计算器
Agent：（行动）[calculator(expression="天气")]
Agent：计算错误
```

LLM 可能做出错误的推理，选择了不合适的工具。这种错误比前两种更难发现，因为 Agent 没有报错，只是给出了错误的结果。

**场景 4：幻觉导致的错误**

```
用户：Python 最新版本是什么？
Agent：Python 最新版本是 3.15。（这是幻觉，3.15 可能还不存在）
```

LLM 生成了看似合理但实际错误的信息。这种错误特别危险，因为用户可能相信这个错误信息。

### 9.1.2 错误分类框架

让我们系统地梳理 Agent 可能遇到的所有错误类型：

**LLM 层错误**

这类错误发生在 LLM API 调用过程中。最常见的包括限流（429 错误，通常是因为请求频率过高）、超时（API 响应太慢）、服务器错误（5xx 错误，通常是 OpenAI 或 Anthropic 的服务端问题）、认证失败（API Key 错误或过期）。这些错误通常是暂时性的，可以通过重试来解决。

**工具层错误**

这类错误发生在工具执行过程中。工具不存在（LLM 调用了未注册的工具）、参数错误（LLM 传入了错误的参数格式）、执行失败（工具内部逻辑出错）、超时（工具执行时间过长）。这些错误需要根据具体情况进行处理。

**推理层错误**

这类错误发生在 LLM 的推理过程中。推理错误（LLM 做出了错误的决策）、幻觉（LLM 编造了不存在的信息）、工具选择错误（LLM 选择了不合适的工具）。这类错误最难发现和处理，因为 Agent 没有报错，只是给出了错误的结果。

**状态层错误**

这类错误发生在状态管理过程中。状态丢失（对话历史丢失）、状态不一致（任务状态和实际状态不匹配）、序列化错误（状态无法正确保存或加载）。这类错误可能导致 Agent 丢失上下文，无法继续之前的任务。

**安全层错误**

这类错误发生在安全检查过程中。Prompt Injection（用户试图操纵 Agent 执行恶意操作）、权限越界（Agent 试图执行超出权限的操作）。这类错误需要特别注意，因为可能导致安全问题。

---

## 9.2 错误恢复策略

### 9.2.1 重试策略

重试是最简单的错误恢复策略。当遇到暂时性错误时，等待一段时间后重试。

**为什么需要重试？**

很多错误是暂时性的。比如 API 限流，通常是因为短时间内请求太多。等待几秒后重试，很可能就能成功。如果没有重试机制，Agent 在遇到第一个错误就崩溃了。

**指数退避重试**

指数退避（Exponential Backoff）是重试的标准策略。每次重试时，等待时间指数增长。

```python
import time
import random


def retry_with_backoff(fn, max_retries=3, base_delay=1, max_delay=60):
    """指数退避重试"""
    for attempt in range(max_retries):
        try:
            return fn()
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
```

**为什么需要随机抖动？**

如果没有随机抖动，多个客户端会在完全相同的时间重试，导致"惊群效应"（Thundering Herd），进一步加剧限流。随机抖动让重试时间分散，减少对服务器的压力。

**重试策略的适用场景：**

重试适用于暂时性错误，比如 API 限流、网络超时、服务器临时错误。但对于永久性错误（如 API Key 错误、工具不存在），重试没有意义，应该直接报错。

### 9.2.2 降级策略

当主要功能不可用时，提供一个简化版本作为替代。

**什么是降级？**

降级不是"失败"，而是"优雅地降低服务质量"。比如，当搜索 API 不可用时，Agent 可以基于自己的知识回答问题，而不是直接报错。

```python
class DegradationHandler:
    def __init__(self):
        self.fallback_responses = {
            "search": "搜索服务暂时不可用，请稍后再试。我可以基于已有知识回答您的问题。",
            "weather": "天气服务暂时不可用。请问您想了解哪个城市的天气？我可以基于历史数据提供参考。",
            "calculator": "计算器暂时不可用。请手动计算或稍后再试。",
        }

    def handle_error(self, tool_name: str, error: Exception) -> str:
        """处理工具错误，提供降级响应"""
        fallback = self.fallback_responses.get(
            tool_name,
            f"工具 {tool_name} 暂时不可用：{error}"
        )
        return fallback
```

**降级策略的设计原则：**

降级时要告诉用户当前状态，不要假装一切正常。降级版本要能完成核心功能，至少能提供部分价值。降级时要记录日志，方便后续分析和改进。

### 9.2.3 回退策略

当主要方案失败时，回退到备选方案。

**重试 vs 降级 vs 回退**

重试是等待后重新尝试相同的操作，适用于暂时性错误。降级是提供简化版本，适用于功能不可用但核心价值仍在的场景。回退是切换到完全不同的方案，适用于主要方案彻底失败的场景。

```python
class FallbackHandler:
    def __init__(self, primary_fn, fallback_fn):
        self.primary = primary_fn
        self.fallback = fallback_fn

    def execute(self, *args, **kwargs):
        try:
            return self.primary(*args, **kwargs)
        except Exception as e:
            print(f"主方案失败：{e}，切换到备选方案")
            return self.fallback(*args, **kwargs)
```

**回退策略的典型应用：**

当 GPT-4o 不可用时，回退到 GPT-4o-mini。当搜索 API 失败时，回退到本地知识库。当数据库不可用时，回退到缓存数据。

### 9.2.4 策略选择指南

```
错误是暂时性的吗？
├── 是 → 重试（指数退避）
└── 否 → 2

主要功能完全不可用吗？
├── 是 → 回退到备选方案
└── 否 → 3

能提供简化版本吗？
├── 是 → 降级
└── 否 → 报错并通知用户
```

---

## 9.3 自我纠错机制

### 9.3.1 什么是自我纠错

自我纠错是让 Agent 能发现并修正自己错误的能力。这是 Agent 和普通程序的重要区别——普通程序出错就崩溃了，而 Agent 能"意识到"自己可能错了，并尝试修正。

**一个自我纠错的例子：**

```
用户：计算 123 的阶乘

第 1 次尝试：
Agent：[calculator(expression="123!")]
结果：计算超时（数字太大）

自我纠错：
Agent：（思考）直接计算 123 的阶乘会导致超时，我应该使用近似方法
Agent：[calculator(expression="sum(log(i) for i in range(1, 124))")]
结果：219.48

Agent：123! 的自然对数约为 219.48，所以 123! ≈ e^219.48 ≈ 2.3×10^95
```

### 9.3.2 实现自我纠错

```python
class SelfCorrectingAgent:
    def __init__(self, llm_client, tools: dict, max_corrections: int = 3):
        self.llm = llm_client
        self.tools = tools
        self.max_corrections = max_corrections

    def run_with_correction(self, task: str) -> str:
        """带自我纠错的 Agent 运行"""
        # 第一次尝试
        result = self._attempt(task)

        for correction_round in range(self.max_corrections):
            # 自我审查
            critique = self._self_critique(task, result)

            if "没有错误" in critique or "正确" in critique:
                # 没有发现错误，返回结果
                return result

            # 自我修正
            result = self._self_correct(task, result, critique)

        return result

    def _attempt(self, task: str) -> str:
        """第一次尝试"""
        # 正常的 Agent 执行逻辑
        pass

    def _self_critique(self, task: str, result: str) -> str:
        """自我审查"""
        prompt = f"""请审查以下任务的执行结果，指出其中的错误或不足。

任务：{task}
结果：{result}

如果没有错误，请说"没有错误"。如果有错误，请描述问题并建议如何修正。"""

        response = self.llm.chat(
            messages=[{"role": "user", "content": prompt}]
        )
        return response

    def _self_correct(self, task: str, result: str, critique: str) -> str:
        """自我修正"""
        prompt = f"""请根据以下审查意见修正结果。

任务：{task}
原结果：{result}
审查意见：{critique}

请给出修正后的结果。"""

        response = self.llm.chat(
            messages=[{"role": "user", "content": prompt}]
        )
        return response
```

### 9.3.3 自我纠错的局限

自我纠错不是万能的。LLM 可能无法发现自己的错误（盲点），修正可能引入新的错误，每次纠错都需要额外的 LLM 调用（成本高）。因此，自我纠错应该作为最后的手段，而不是主要的错误处理策略。

---

## 9.4 错误日志和告警

### 9.4.1 为什么需要错误日志

错误日志是 Agent 可观测性的基础。没有日志，你就无法知道 Agent 遇到了什么错误、错误发生的频率、错误的原因。日志是改进 Agent 的基础。

### 9.4.2 结构化错误日志

```python
import json
import logging
from datetime import datetime
from dataclasses import dataclass, asdict
from typing import Any


@dataclass
class ErrorEntry:
    """错误日志条目"""
    timestamp: str
    error_type: str
    error_message: str
    context: dict
    stack_trace: str = ""


class ErrorLogger:
    def __init__(self, name: str):
        self.logger = logging.getLogger(name)
        self.errors = []

    def log_error(
        self,
        error_type: str,
        error: Exception,
        context: dict = None,
    ):
        """记录错误"""
        entry = ErrorEntry(
            timestamp=datetime.now().isoformat(),
            error_type=error_type,
            error_message=str(error),
            context=context or {},
        )
        self.errors.append(entry)
        self.logger.error(json.dumps(asdict(entry), ensure_ascii=False))

    def get_error_summary(self) -> dict:
        """获取错误摘要"""
        summary = {}
        for error in self.errors:
            error_type = error.error_type
            summary[error_type] = summary.get(error_type, 0) + 1
        return summary

    def get_recent_errors(self, n: int = 10) -> list[ErrorEntry]:
        """获取最近的错误"""
        return self.errors[-n:]
```

### 9.4.3 告警设计

```python
class AlertManager:
    def __init__(self, thresholds: dict):
        self.thresholds = thresholds
        self.error_counts = {}

    def check_and_alert(self, error_type: str):
        """检查是否需要告警"""
        self.error_counts[error_type] = self.error_counts.get(error_type, 0) + 1
        count = self.error_counts[error_type]

        threshold = self.thresholds.get(error_type, 10)
        if count >= threshold:
            self._send_alert(error_type, count)

    def _send_alert(self, error_type: str, count: int):
        """发送告警"""
        message = f"告警：{error_type} 错误已发生 {count} 次"
        print(f"[ALERT] {message}")
        # 实际项目中可以发送邮件、Slack 消息等
```

---

## 9.5 电路断路器模式

### 9.5.1 什么是电路断路器

电路断路器（Circuit Breaker）是一种经典的容错模式，最初用于电力系统。它的核心思想是：当某个服务连续失败时，暂时"断开"对该服务的调用，避免无意义的重试浪费资源。过一段时间后再尝试恢复。

这在 Agent 系统中非常有用。比如，如果搜索 API 连续失败了 5 次，很可能不是你的代码有问题，而是搜索服务本身挂了。此时继续重试不仅浪费时间，还会阻塞整个 Agent 流程。

### 9.5.2 实现电路断路器

```python
import time
import random
from enum import Enum


class CircuitState(Enum):
    """电路状态"""
    CLOSED = "closed"       # 正常状态，请求正常通过
    OPEN = "open"           # 断开状态，请求直接失败
    HALF_OPEN = "half_open" # 半开状态，允许少量请求通过以测试恢复


class CircuitBreaker:
    """电路断路器"""

    def __init__(
        self,
        failure_threshold: int = 5,
        recovery_timeout: float = 30,
        half_open_max_calls: int = 1,
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.half_open_max_calls = half_open_max_calls

        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.last_failure_time = 0
        self.half_open_calls = 0

    def call(self, fn, *args, **kwargs):
        """通过电路断路器调用函数"""
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
                self.half_open_calls = 0
            else:
                raise CircuitBreakerOpenError(
                    f"电路断开，将在 {self.recovery_timeout}s 后重试"
                )

        if self.state == CircuitState.HALF_OPEN:
            if self.half_open_calls >= self.half_open_max_calls:
                raise CircuitBreakerOpenError("电路半开，等待更多恢复信号")
            self.half_open_calls += 1

        try:
            result = fn(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _on_success(self):
        """调用成功"""
        self.failure_count = 0
        if self.state == CircuitState.HALF_OPEN:
            self.state = CircuitState.CLOSED

    def _on_failure(self):
        """调用失败"""
        self.failure_count += 1
        self.last_failure_time = time.time()

        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN


class CircuitBreakerOpenError(Exception):
    """电路断路器打开异常"""
    pass
```

### 9.5.3 在 Agent 中使用电路断路器

```python
class AgentWithCircuitBreaker:
    """带电路断路器的 Agent"""

    def __init__(self, llm_client, tools: dict):
        self.llm = llm_client
        self.tools = tools
        self.breakers = {
            name: CircuitBreaker(failure_threshold=3, recovery_timeout=30)
            for name in tools
        }

    def _execute_tool_with_breaker(self, tool_name: str, args: dict) -> str:
        """通过电路断路器执行工具"""
        breaker = self.breakers.get(tool_name)
        if not breaker:
            return f"工具 {tool_name} 未注册"

        try:
            result = breaker.call(self.tools[tool_name], **args)
            return str(result)
        except CircuitBreakerOpenError as e:
            return f"工具 {tool_name} 暂时不可用（{e}），请稍后重试"
        except Exception as e:
            return f"工具 {tool_name} 执行失败：{e}"
```

---

## 9.6 完整的错误处理框架

```python
class AgentErrorHandler:
    """Agent 错误处理框架"""

    def __init__(self, max_retries: int = 3, enable_self_correction: bool = True):
        self.max_retries = max_retries
        self.enable_self_correction = enable_self_correction
        self.error_logger = ErrorLogger("agent")
        self.alert_manager = AlertManager({
            "llm_error": 5,
            "tool_error": 10,
            "推理错误": 3,
        })
        # 每个工具的电路断路器
        self._circuit_breakers = {}

    def get_circuit_breaker(self, tool_name: str) -> CircuitBreaker:
        """获取或创建工具的电路断路器"""
        if tool_name not in self._circuit_breakers:
            self._circuit_breakers[tool_name] = CircuitBreaker(
                failure_threshold=3,
                recovery_timeout=30,
            )
        return self._circuit_breakers[tool_name]

    def handle_llm_error(self, error: Exception, attempt: int) -> bool:
        """处理 LLM 错误"""
        self.error_logger.log_error("llm_error", error)
        self.alert_manager.check_and_alert("llm_error")

        if attempt < self.max_retries:
            wait_time = min(2 ** attempt + random.uniform(0, 1), 60)
            time.sleep(wait_time)
            return True  # 可以重试
        return False  # 不再重试

    def handle_tool_error(self, tool_name: str, error: Exception) -> str:
        """处理工具错误"""
        self.error_logger.log_error("tool_error", error, {"tool": tool_name})
        self.alert_manager.check_and_alert("tool_error")

        # 降级处理
        return f"工具 {tool_name} 暂时不可用，请稍后再试"

    def handle_reasoning_error(self, task: str, result: str) -> str:
        """处理推理错误"""
        self.error_logger.log_error("推理错误", Exception("推理错误"), {"task": task})
        self.alert_manager.check_and_alert("推理错误")

        if self.enable_self_correction:
            return self._self_correct(task, result)
        return result

    def _self_correct(self, task: str, result: str) -> str:
        """自我纠错"""
        pass

    def get_health_report(self) -> dict:
        """获取错误处理健康报告"""
        report = {}
        for tool_name, breaker in self._circuit_breakers.items():
            report[tool_name] = {
                "state": breaker.state.value,
                "failure_count": breaker.failure_count,
            }
        return {
            "error_summary": self.error_logger.get_error_summary(),
            "circuit_breakers": report,
        }

---

## 9.6 常见坑

### 9.6.1 不处理错误

```
问题：Agent 遇到错误直接崩溃
症状：用户看到 Python 错误堆栈
解决：始终 try-except，提供友好的错误信息
```

### 9.6.2 无限重试

```
问题：没有最大重试次数限制
症状：Agent 陷入无限循环
解决：设置 max_retries，超限后报错
```

### 9.6.3 不记录错误

```
问题：无法分析和改进
症状：同样的错误反复出现
解决：记录所有错误，定期分析
```

---

## 9.7 练习题

### 概念理解题

1. Agent 会遇到哪些类型的错误？请用一个具体的场景串联所有错误类型。

2. 重试、降级、回退策略各适用于什么场景？请举例说明。

3. 自我纠错机制的原理是什么？它有哪些局限性？

4. 为什么需要结构化错误日志？它对 Agent 的改进有什么帮助？

### 动手实践题

1. **错误处理实现：** 为 Agent 实现完整的错误处理框架，覆盖 LLM 错误、工具错误、推理错误。

2. **重试实验：** 模拟 API 限流，测试指数退避重试的效果。

3. **自我纠错实验：** 让 Agent 故意犯错，观察自我纠错的效果。

### 思考题

1. 自我纠错的"成本"是什么？如何平衡纠错效果和成本？

2. 在生产环境中，如何设计 Agent 的错误告警策略？

---

## 9.8 实战任务

**任务：为 Agent 添加完整的错误处理机制**

**具体要求：**

1. 实现 LLM 错误处理（重试、降级）
2. 实现工具错误处理（降级、回退）
3. 实现推理错误处理（自我纠错）
4. 实现错误日志记录
5. 实现告警机制

**评估标准：**

| 维度 | 权重 | 评分标准 |
|------|------|---------|
| 错误覆盖 | 30% | 覆盖所有错误类型 |
| 恢复策略 | 25% | 策略合理，效果良好 |

---

## 9.8 错误处理的高级模式

### 9.8.1 错误分类与优先级

在复杂的 Agent 系统中，错误可能同时来自多个层面。建立清晰的错误分类体系有助于快速定位和处理问题。

```python
class ErrorPriority:
    """错误优先级定义"""

    CRITICAL = "critical"    # 系统崩溃、数据丢失
    HIGH = "high"            # 核心功能不可用
    MEDIUM = "medium"        # 非核心功能异常
    LOW = "low"              # 轻微问题、可忽略


class ErrorClassifier:
    """错误分类器"""

    ERROR_MAP = {
        "APIKeyError": ErrorPriority.CRITICAL,
        "RateLimitError": ErrorPriority.MEDIUM,
        "TimeoutError": ErrorPriority.MEDIUM,
        "ToolExecutionError": ErrorPriority.HIGH,
        "ParsingError": ErrorPriority.LOW,
    }

    @classmethod
    def classify(cls, error: Exception) -> str:
        """对错误进行分类"""
        error_type = type(error).__name__
        return cls.ERROR_MAP.get(error_type, ErrorPriority.MEDIUM)
```

### 9.8.2 错误传播链

当多个组件串联时，错误可能在组件之间传播。理解错误传播链有助于在正确的层面处理错误。

```python
class ErrorPropagationChain:
    """错误传播链管理"""

    def __init__(self):
        self.chain = []

    def add_handler(self, name: str, handler):
        """添加错误处理器"""
        self.chain.append({"name": name, "handler": handler})

    def handle(self, error: Exception, context: dict = None):
        """沿传播链处理错误"""
        for step in self.chain:
            try:
                result = step["handler"](error, context)
                if result is not None:
                    return result
            except Exception as e:
                error = e
                continue

        # 所有处理器都失败了
        return {"status": "unhandled", "error": str(error)}
```

### 9.8.3 错误恢复的测试

错误处理代码本身也需要测试。通过模拟各种错误场景，验证 Agent 的恢复能力。

```python
import pytest
from unittest.mock import patch, MagicMock


class TestErrorRecovery:
    """错误恢复测试"""

    def test_api_timeout_recovery(self):
        """测试 API 超时后的恢复"""
        agent = create_test_agent()

        # 模拟第一次调用超时，第二次成功
        with patch("openai.OpenAI") as mock_client:
            mock_client.side_effect = [TimeoutError(), MagicMock()]
            result = agent.run("测试任务")

            # 应该能从超时中恢复
            assert result is not None

    def test_tool_failure_recovery(self):
        """测试工具失败后的恢复"""
        agent = create_test_agent()
        agent.register_tool("fail_tool", lambda: 1/0)

        # 工具失败后应该能继续
        result = agent.run("请使用 fail_tool")
        assert "错误" in result or "无法" in result

    def test_rate_limit_retry(self):
        """测试限流后的重试"""
        call_count = 0

        def mock_api_call():
            nonlocal call_count
            call_count += 1
            if call_count < 3:
                raise RateLimitError("Too many requests")
            return {"content": "成功"}

        result = retry_with_backoff(mock_api_call, max_retries=5)
        assert result["content"] == "成功"
        assert call_count == 3
```

### 9.8.4 错误处理的度量

监控错误处理的效果是持续改进的基础。

```python
class ErrorMetrics:
    """错误处理度量"""

    def __init__(self):
        self.error_counts = {}
        self.recovery_counts = {}
        self.failure_counts = {}

    def record_error(self, error_type: str, recovered: bool):
        """记录错误和恢复情况"""
        self.error_counts[error_type] = self.error_counts.get(error_type, 0) + 1
        if recovered:
            self.recovery_counts[error_type] = self.recovery_counts.get(error_type, 0) + 1
        else:
            self.failure_counts[error_type] = self.failure_counts.get(error_type, 0) + 1

    def get_recovery_rate(self, error_type: str) -> float:
        """获取错误恢复率"""
        total = self.error_counts.get(error_type, 0)
        if total == 0:
            return 0.0
        recovered = self.recovery_counts.get(error_type, 0)
        return recovered / total

    def get_report(self) -> str:
        """生成错误处理报告"""
        lines = ["错误处理报告", "=" * 40]
        for error_type in self.error_counts:
            total = self.error_counts[error_type]
            recovered = self.recovery_counts.get(error_type, 0)
            rate = self.get_recovery_rate(error_type)
            lines.append(f"{error_type}: 总计 {total}, 恢复 {recovered}, 恢复率 {rate*100:.1f}%")
        return "\n".join(lines)
```
| 日志记录 | 20% | 结构化、完整 |
| 自我纠错 | 15% | 能发现并修正错误 |
| 代码质量 | 10% | 结构清晰，可维护 |

---

## 9.9 本章小结

### 核心知识点回顾

- **错误类型：** LLM 错误（API 问题）、工具错误（执行失败）、推理错误（决策错误）、状态错误（状态丢失）、安全错误（权限问题）

- **恢复策略：** 重试（暂时性错误）、降级（功能降级但可用）、回退（切换到备选方案）

- **自我纠错：** 让 Agent 能发现并修正自己的错误，但有局限性

- **错误日志：** 结构化记录所有错误，支持后续分析和改进

### 关键公式

```
Agent 可靠性 = 错误检测能力 × 错误恢复能力 × 错误学习能力
```

### 下一章预告

下一章我们将整合前 9 章的知识，**构建你的第一个完整 Agent**。你将看到如何将 LLM、Tool Calling、状态管理、错误处理整合到一个完整的 Agent 系统中。

---

*上一章：[第 8 章：Agent 的状态管理](../chapter-08/README.md)*
*下一章：[第 10 章：构建你的第一个完整 Agent](../chapter-10/README.md)*
