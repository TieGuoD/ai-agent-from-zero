# 第 64 章：项目优化 -- 从原型到生产

> **本章定位：** 前六章构建的项目都是原型级别的代码。它们展示了核心功能，但距离真正可用的产品还有很大的差距。本章将系统性地讨论如何把这些原型优化为生产级系统，涵盖错误处理、性能优化、安全加固、测试策略、部署方案等关键话题。这是从"能跑"到"能用"的关键一步。

---

## 学习目标

完成本章学习后，你将能够：

1. **识别原型代码中的生产风险** -- 能够系统性地审查原型代码，找出所有可能导致生产事故的问题
2. **实现全面的错误处理和恢复机制** -- 能够处理网络故障、API 限流、模型错误等各种异常情况
3. **设计生产级的日志和监控系统** -- 能够实现结构化日志、关键指标监控、告警机制
4. **优化 Agent 的性能和成本** -- 能够通过缓存、批处理、模型选择等手段降低延迟和成本
5. **实现安全加固** -- 能够处理敏感数据、控制权限、防御注入攻击
6. **设计可扩展的部署方案** -- 能够选择合适的部署架构（容器化、Serverless、微服务等）

## 核心问题

1. **从原型到生产，最关键的差距在哪里？** 是错误处理？是性能？是安全？还是可运维性？
2. **Agent 系统的 SLA 应该如何定义？** 什么算"正常运行"？什么算"服务降级"？什么算"服务中断"？
3. **如何在成本和质量之间找到平衡？** 更强的模型意味着更好的质量但更高的成本，如何优化这个权衡？

---

## 64.1 错误处理与恢复

### 64.1.1 原型代码中的常见错误处理问题

回顾前面六个项目的代码，错误处理是最薄弱的环节。大部分地方使用了简单的 `try-except`，捕获所有异常后打印错误信息。这在开发阶段可以接受，但在生产环境中远远不够。

生产级的错误处理需要考虑以下问题：异常的分类（哪些可以重试、哪些需要告警、哪些需要降级）、错误信息的脱敏（不能把 API Key 暴露给用户）、失败后的恢复策略（自动重试还是人工介入）、错误的上报和追踪（方便后续排查）。

### 64.1.2 分层错误处理框架

```python
# error_handling.py
import time
import logging
import traceback
from enum import Enum
from dataclasses import dataclass
from typing import Optional, Callable, Any
from functools import wraps


class ErrorSeverity(Enum):
    """错误严重等级。"""
    LOW = "low"           # 可以忽略的小问题
    MEDIUM = "medium"     # 需要关注但不影响核心功能
    HIGH = "high"         # 影响核心功能，需要及时处理
    CRITICAL = "critical" # 系统级故障，需要立即处理


class ErrorCategory(Enum):
    """错误分类。"""
    NETWORK = "network"           # 网络相关错误
    API_LIMIT = "api_limit"       # API 限流
    AUTH = "auth"                 # 认证错误
    INVALID_INPUT = "invalid_input"  # 输入错误
    MODEL_ERROR = "model_error"   # 模型相关错误
    TOOL_ERROR = "tool_error"     # 工具执行错误
    INTERNAL = "internal"         # 内部错误


@dataclass
class AgentError:
    """Agent 错误的标准化表示。"""
    category: ErrorCategory
    severity: ErrorSeverity
    message: str
    original_exception: Optional[Exception] = None
    context: dict = None
    retry_after: float = 0  # 建议的重试等待时间（秒）
    should_retry: bool = False

    def to_user_message(self) -> str:
        """生成用户友好的错误消息（不包含敏感信息）。"""
        messages = {
            ErrorCategory.NETWORK: "网络连接出现问题，请稍后重试。",
            ErrorCategory.API_LIMIT: "请求过于频繁，请稍后重试。",
            ErrorCategory.AUTH: "认证失败，请检查 API Key 配置。",
            ErrorCategory.INVALID_INPUT: "输入内容有误，请检查后重试。",
            ErrorCategory.MODEL_ERROR: "AI 模型暂时不可用，请稍后重试。",
            ErrorCategory.TOOL_ERROR: "工具执行出错，请稍后重试。",
            ErrorCategory.INTERNAL: "系统内部错误，请稍后重试。",
        }
        return messages.get(self.category, "发生未知错误，请稍后重试。")


class ErrorHandler:
    """生产级错误处理器。

    功能：
    1. 异常分类和标准化
    2. 自动重试策略
    3. 错误日志记录
    4. 用户友好消息生成
    5. 错误统计和告警
    """

    # 可重试的错误类别
    RETRYABLE_CATEGORIES = {ErrorCategory.NETWORK, ErrorCategory.API_LIMIT, ErrorCategory.MODEL_ERROR}

    def __init__(self, logger: logging.Logger = None):
        self.logger = logger or logging.getLogger("agent")
        self.error_counts: dict[str, int] = {}  # 错误计数
        self.total_errors = 0

    def classify_exception(self, exception: Exception) -> AgentError:
        """将原始异常分类为 AgentError。"""
        error_str = str(exception).lower()

        # 网络错误
        if any(kw in error_str for kw in ["connection", "timeout", "network", "dns"]):
            return AgentError(
                category=ErrorCategory.NETWORK,
                severity=ErrorSeverity.MEDIUM,
                message=str(exception),
                original_exception=exception,
                should_retry=True,
                retry_after=5.0,
            )

        # API 限流
        if any(kw in error_str for kw in ["rate limit", "429", "too many requests"]):
            return AgentError(
                category=ErrorCategory.API_LIMIT,
                severity=ErrorSeverity.MEDIUM,
                message=str(exception),
                original_exception=exception,
                should_retry=True,
                retry_after=30.0,
            )

        # 认证错误
        if any(kw in error_str for kw in ["auth", "401", "403", "api key", "unauthorized"]):
            return AgentError(
                category=ErrorCategory.AUTH,
                severity=ErrorSeverity.HIGH,
                message=str(exception),
                original_exception=exception,
                should_retry=False,
            )

        # 模型错误
        if any(kw in error_str for kw in ["model", "overloaded", "500", "502", "503"]):
            return AgentError(
                category=ErrorCategory.MODEL_ERROR,
                severity=ErrorSeverity.MEDIUM,
                message=str(exception),
                original_exception=exception,
                should_retry=True,
                retry_after=10.0,
            )

        # 默认为内部错误
        return AgentError(
            category=ErrorCategory.INTERNAL,
            severity=ErrorSeverity.HIGH,
            message=str(exception),
            original_exception=exception,
            should_retry=False,
        )

    def handle(self, exception: Exception, context: dict = None) -> AgentError:
        """处理异常，返回标准化的 AgentError。"""
        agent_error = self.classify_exception(exception)

        # 更新错误计数
        category_key = agent_error.category.value
        self.error_counts[category_key] = self.error_counts.get(category_key, 0) + 1
        self.total_errors += 1

        # 记录日志
        log_data = {
            "category": agent_error.category.value,
            "severity": agent_error.severity.value,
            "message": agent_error.message,
            "context": context or {},
        }
        if agent_error.severity in (ErrorSeverity.HIGH, ErrorSeverity.CRITICAL):
            self.logger.error(f"Agent Error: {log_data}")
        else:
            self.logger.warning(f"Agent Warning: {log_data}")

        return agent_error


def retry_with_backoff(max_retries: int = 3, base_delay: float = 1.0,
                       max_delay: float = 60.0,
                       retryable_exceptions: tuple = (Exception,)):
    """带指数退避的重试装饰器。

    使用方法：
    @retry_with_backoff(max_retries=3)
    def call_api():
        ...
    """
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        def wrapper(*args, **kwargs) -> Any:
            last_exception = None
            for attempt in range(max_retries + 1):
                try:
                    return func(*args, **kwargs)
                except retryable_exceptions as e:
                    last_exception = e
                    if attempt < max_retries:
                        delay = min(base_delay * (2 ** attempt), max_delay)
                        logging.getLogger("agent").warning(
                            f"重试 {func.__name__} ({attempt + 1}/{max_retries})，"
                            f"等待 {delay:.1f}s: {e}"
                        )
                        time.sleep(delay)
            raise last_exception
        return wrapper
    return decorator
```

### 64.1.3 优雅降级策略

```python
# fallback.py
from typing import Any, Callable, Optional


class FallbackChain:
    """降级策略链。

    当主方案失败时，按顺序尝试备选方案。
    例如：GPT-4o 失败 -> GPT-4o-mini -> 本地模型 -> 返回缓存结果
    """

    def __init__(self):
        self.steps: list[tuple[str, Callable, dict]] = []

    def add_step(self, name: str, func: Callable, kwargs: dict = None):
        """添加一个降级步骤。"""
        self.steps.append((name, func, kwargs or {}))
        return self

    def execute(self, *args, **kwargs) -> Any:
        """执行降级链。"""
        errors = []
        for name, func, step_kwargs in self.steps:
            try:
                result = func(*args, **{**kwargs, **step_kwargs})
                if result is not None:
                    return result
            except Exception as e:
                errors.append(f"{name}: {e}")
                continue

        raise RuntimeError(f"所有降级步骤都失败了:\n" + "\n".join(errors))


# 使用示例
def create_llm_fallback(api_key: str) -> FallbackChain:
    """创建 LLM 调用的降级链。"""
    from openai import OpenAI

    chain = FallbackChain()

    # 主方案：GPT-4o
    def call_gpt4o(prompt, **kwargs):
        client = OpenAI(api_key=api_key)
        resp = client.chat.completions.create(
            model="gpt-4o", messages=[{"role": "user", "content": prompt}],
            **kwargs
        )
        return resp.choices[0].message.content

    # 备选1：GPT-4o-mini
    def call_mini(prompt, **kwargs):
        client = OpenAI(api_key=api_key)
        resp = client.chat.completions.create(
            model="gpt-4o-mini", messages=[{"role": "user", "content": prompt}],
            **kwargs
        )
        return resp.choices[0].message.content

    # 备选2：返回缓存的默认回复
    def fallback_cached(prompt, **kwargs):
        return "抱歉，AI 服务暂时不可用，请稍后重试。"

    chain.add_step("GPT-4o", call_gpt4o)
    chain.add_step("GPT-4o-mini", call_mini)
    chain.add_step("缓存回复", fallback_cached)

    return chain
```

---

## 64.2 日志与监控

### 64.2.1 结构化日志

```python
# logging_config.py
import logging
import json
import time
import sys
from datetime import datetime


class JSONFormatter(logging.Formatter):
    """JSON 格式的日志格式化器。

    结构化日志比纯文本日志更容易被日志分析工具处理。
    """

    def format(self, record):
        log_data = {
            "timestamp": datetime.utcnow().isoformat(),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            "module": record.module,
            "function": record.funcName,
            "line": record.lineno,
        }

        # 附加额外数据
        if hasattr(record, "extra_data"):
            log_data["data"] = record.extra_data

        if record.exc_info:
            log_data["exception"] = self.formatException(record.exc_info)

        return json.dumps(log_data, ensure_ascii=False)


def setup_logging(log_file: str = "agent.log",
                  level: str = "INFO",
                  json_format: bool = True) -> logging.Logger:
    """配置生产级日志系统。"""
    logger = logging.getLogger("agent")
    logger.setLevel(getattr(logging, level.upper()))

    # 控制台输出
    console_handler = logging.StreamHandler(sys.stdout)
    if json_format:
        console_handler.setFormatter(JSONFormatter())
    else:
        console_handler.setFormatter(logging.Formatter(
            "%(asctime)s [%(levelname)s] %(name)s: %(message)s"
        ))
    logger.addHandler(console_handler)

    # 文件输出
    if log_file:
        file_handler = logging.FileHandler(log_file, encoding="utf-8")
        file_handler.setFormatter(JSONFormatter())
        logger.addHandler(file_handler)

    return logger
```

### 64.2.2 性能监控

```python
# metrics.py
import time
import threading
from dataclasses import dataclass, field
from collections import defaultdict


@dataclass
class MetricPoint:
    """单个指标数据点。"""
    name: str
    value: float
    timestamp: float
    tags: dict = field(default_factory=dict)


class MetricsCollector:
    """指标收集器。

    收集 Agent 运行时的关键指标：
    - 响应时间
    - API 调用次数和成本
    - 错误率
    - 工具使用频率
    - Token 使用量
    """

    def __init__(self):
        self.metrics: list[MetricPoint] = []
        self.counters: dict[str, int] = defaultdict(int)
        self.timers: dict[str, list[float]] = defaultdict(list)
        self._lock = threading.Lock()

    def record_metric(self, name: str, value: float, tags: dict = None):
        """记录一个指标。"""
        with self._lock:
            self.metrics.append(MetricPoint(
                name=name,
                value=value,
                timestamp=time.time(),
                tags=tags or {},
            ))

    def increment_counter(self, name: str, value: int = 1):
        """递增计数器。"""
        with self._lock:
            self.counters[name] += value

    def record_timer(self, name: str, duration: float):
        """记录耗时。"""
        with self._lock:
            self.timers[name].append(duration)

    def get_summary(self) -> dict:
        """生成指标摘要。"""
        summary = {
            "counters": dict(self.counters),
            "timers": {},
        }
        for name, durations in self.timers.items():
            if durations:
                summary["timers"][name] = {
                    "count": len(durations),
                    "avg_ms": sum(durations) / len(durations) * 1000,
                    "min_ms": min(durations) * 1000,
                    "max_ms": max(durations) * 1000,
                    "p95_ms": sorted(durations)[int(len(durations) * 0.95)] * 1000
                    if len(durations) > 1 else durations[0] * 1000,
                }
        return summary


class TimerContext:
    """计时上下文管理器。"""
    def __init__(self, collector: MetricsCollector, metric_name: str):
        self.collector = collector
        self.metric_name = metric_name
        self.start_time = 0

    def __enter__(self):
        self.start_time = time.time()
        return self

    def __exit__(self, *args):
        duration = time.time() - self.start_time
        self.collector.record_timer(self.metric_name, duration)


# 使用示例
# metrics = MetricsCollector()
# with TimerContext(metrics, "llm_call"):
#     response = call_llm(prompt)
# metrics.increment_counter("api_calls")
# print(metrics.get_summary())
```

---

## 64.3 性能优化

### 64.3.1 响应缓存

```python
# cache.py
import json
import hashlib
import os
import time
from typing import Optional, Any
from functools import wraps


class ResponseCache:
    """LLM 响应缓存。

    对相同（或相似）的请求缓存 LLM 的响应，
    避免重复调用 API。这对降低成本和提高响应速度都很有帮助。
    """

    def __init__(self, cache_dir: str = ".cache",
                 ttl: int = 3600 * 24,  # 默认 24 小时过期
                 max_entries: int = 1000):
        self.cache_dir = cache_dir
        self.ttl = ttl
        self.max_entries = max_entries
        os.makedirs(cache_dir, exist_ok=True)

    def _make_key(self, model: str, messages: list[dict],
                  temperature: float) -> str:
        """生成缓存键。"""
        # 对于确定性请求（temperature=0），使用完整的消息内容
        # 对于非确定性请求，使用消息的摘要
        content = json.dumps({
            "model": model,
            "messages": [
                {"role": m["role"], "content": m["content"][:500]}
                for m in messages
            ],
            "temperature": temperature,
        }, sort_keys=True)
        return hashlib.sha256(content.encode()).hexdigest()

    def get(self, model: str, messages: list[dict],
            temperature: float) -> Optional[str]:
        """获取缓存的响应。"""
        key = self._make_key(model, messages, temperature)
        cache_file = os.path.join(self.cache_dir, f"{key}.json")

        if not os.path.exists(cache_file):
            return None

        try:
            with open(cache_file, "r", encoding="utf-8") as f:
                data = json.load(f)

            # 检查是否过期
            if time.time() - data.get("timestamp", 0) > self.ttl:
                os.unlink(cache_file)
                return None

            return data.get("response")
        except Exception:
            return None

    def set(self, model: str, messages: list[dict],
            temperature: float, response: str):
        """缓存响应。"""
        key = self._make_key(model, messages, temperature)
        cache_file = os.path.join(self.cache_dir, f"{key}.json")

        try:
            with open(cache_file, "w", encoding="utf-8") as f:
                json.dump({
                    "response": response,
                    "timestamp": time.time(),
                    "model": model,
                }, f, ensure_ascii=False)
        except Exception:
            pass

        # 检查缓存数量限制
        self._cleanup_if_needed()

    def _cleanup_if_needed(self):
        """清理过期或超出限制的缓存。"""
        files = []
        for fn in os.listdir(self.cache_dir):
            if fn.endswith(".json"):
                fp = os.path.join(self.cache_dir, fn)
                files.append((fp, os.path.getmtime(fp)))

        if len(files) > self.max_entries:
            # 按修改时间排序，删除最旧的
            files.sort(key=lambda x: x[1])
            for fp, _ in files[:len(files) - self.max_entries]:
                try:
                    os.unlink(fp)
                except Exception:
                    pass


def cached(cache: ResponseCache):
    """缓存装饰器。"""
    def decorator(func):
        @wraps(func)
        def wrapper(self, messages, **kwargs):
            model = kwargs.get("model", getattr(self, "model", "gpt-4o"))
            temperature = kwargs.get("temperature", 0.3)

            # 尝试从缓存获取
            if temperature == 0:  # 只缓存确定性请求
                cached_response = cache.get(model, messages, temperature)
                if cached_response:
                    return cached_response

            # 调用原始函数
            response = func(self, messages, **kwargs)

            # 缓存结果
            if temperature == 0 and response:
                cache.set(model, messages, temperature, response)

            return response
        return wrapper
    return decorator
```

### 64.3.2 批量处理

```python
# batch.py
import asyncio
from typing import Callable
from concurrent.futures import ThreadPoolExecutor


class BatchProcessor:
    """批量处理器。

    将多个独立的操作合并处理，减少 API 调用次数。
    例如：批量 Embedding、批量分类。
    """

    def __init__(self, max_workers: int = 5):
        self.max_workers = max_workers
        self.executor = ThreadPoolExecutor(max_workers=max_workers)

    async def process_batch(self, items: list,
                           process_fn: Callable,
                           batch_size: int = 10) -> list:
        """批量处理项目。"""
        results = []
        for i in range(0, len(items), batch_size):
            batch = items[i:i + batch_size]
            # 使用线程池并行处理
            loop = asyncio.get_event_loop()
            batch_results = await asyncio.gather(*[
                loop.run_in_executor(self.executor, process_fn, item)
                for item in batch
            ])
            results.extend(batch_results)
        return results
```

---

## 64.4 安全加固

### 64.4.1 输入验证和清洗

```python
# security.py
import re
from typing import Optional


class InputValidator:
    """输入验证器。

    防止注入攻击和不安全的输入。
    """

    # 危险模式
    DANGEROUS_PATTERNS = [
        r"(?i)(system\s*prompt|ignore\s*previous|forget\s*everything)",
        r"(?i)(jailbreak|DAN|do\s*anything\s*now)",
        r"<script[^>]*>",
        r"(?i)(drop\s*table|delete\s*from|insert\s*into)",
    ]

    MAX_INPUT_LENGTH = 10000

    @classmethod
    def validate(cls, user_input: str) -> tuple[bool, Optional[str]]:
        """验证用户输入。

        Returns:
            (is_valid, error_message)
        """
        # 长度检查
        if len(user_input) > cls.MAX_INPUT_LENGTH:
            return False, f"输入过长（最大 {cls.MAX_INPUT_LENGTH} 字符）"

        # 危险模式检测
        for pattern in cls.DANGEROUS_PATTERNS:
            if re.search(pattern, user_input):
                return False, "输入包含不允许的内容"

        return True, None

    @classmethod
    def sanitize(cls, text: str) -> str:
        """清洗文本，去除潜在的有害内容。"""
        # 移除控制字符
        text = re.sub(r'[\x00-\x08\x0b\x0c\x0e-\x1f\x7f]', '', text)
        # 限制长度
        if len(text) > cls.MAX_INPUT_LENGTH:
            text = text[:cls.MAX_INPUT_LENGTH]
        return text


class SecretManager:
    """密钥管理器。

    确保敏感信息（API Key、密码等）不会泄露。
    """

    SENSITIVE_KEYS = {"api_key", "password", "secret", "token", "access_key"}

    @classmethod
    def mask(cls, text: str) -> str:
        """对文本中的敏感信息进行脱敏。"""
        for key in cls.SENSITIVE_KEYS:
            # 匹配 key=value 或 key: value 格式
            pattern = rf'({key}[=:]\s*)["\']?([A-Za-z0-9\-_]{{8}})[A-Za-z0-9\-_]*["\']?'
            text = re.sub(pattern, rf'\g<1>\2****', text, flags=re.IGNORECASE)
        return text

    @classmethod
    def validate_api_key(cls, key: str) -> bool:
        """验证 API Key 格式。"""
        if not key:
            return False
        # OpenAI API Key 通常以 "sk-" 开头
        if key.startswith("sk-") and len(key) > 20:
            return True
        # 其他格式也接受，但标记为未知
        return len(key) > 10
```

---

## 64.5 测试策略

### 64.5.1 Agent 测试框架

```python
# testing.py
import json
from typing import Callable, Any
from dataclasses import dataclass, field


@dataclass
class TestCase:
    """Agent 测试用例。"""
    name: str
    input_text: str
    expected_behavior: str  # "contains", "not_contains", "exact", "intent"
    expected_value: str
    tags: list[str] = field(default_factory=list)


@dataclass
class TestResult:
    """测试结果。"""
    test_case: TestCase
    actual_output: str
    passed: bool
    details: str = ""


class AgentTestSuite:
    """Agent 测试套件。

    自动化测试 Agent 的行为。
    """

    def __init__(self, agent_chat_fn: Callable):
        """
        Args:
            agent_chat_fn: Agent 的 chat 方法
        """
        self.chat_fn = agent_chat_fn
        self.test_cases: list[TestCase] = []

    def add_test(self, name: str, input_text: str,
                 expected_behavior: str, expected_value: str,
                 tags: list[str] = None):
        """添加测试用例。"""
        self.test_cases.append(TestCase(
            name=name,
            input_text=input_text,
            expected_behavior=expected_behavior,
            expected_value=expected_value,
            tags=tags or [],
        ))

    def run_all(self) -> list[TestResult]:
        """运行所有测试。"""
        results = []
        for tc in self.test_cases:
            result = self._run_single(tc)
            results.append(result)
            status = "PASS" if result.passed else "FAIL"
            print(f"  [{status}] {tc.name}")
        return results

    def _run_single(self, tc: TestCase) -> TestResult:
        """运行单个测试。"""
        try:
            output = self.chat_fn(tc.input_text)
        except Exception as e:
            return TestResult(
                test_case=tc,
                actual_output=str(e),
                passed=False,
                details=f"执行异常: {e}"
            )

        # 根据预期行为判断
        if tc.expected_behavior == "contains":
            passed = tc.expected_value.lower() in output.lower()
        elif tc.expected_behavior == "not_contains":
            passed = tc.expected_value.lower() not in output.lower()
        elif tc.expected_behavior == "intent":
            # 检查 Agent 是否理解了意图
            passed = self._check_intent(output, tc.expected_value)
        else:
            passed = output.strip() == tc.expected_value.strip()

        return TestResult(
            test_case=tc,
            actual_output=output,
            passed=passed,
            details=f"期望: {tc.expected_behavior}={tc.expected_value}"
        )

    def _check_intent(self, output: str, expected_intent: str) -> bool:
        """检查输出是否匹配预期意图。"""
        intent_keywords = {
            "search": ["搜索", "查找", "查询", "search"],
            "explain": ["解释", "说明", "介绍", "explain"],
            "create": ["创建", "生成", "创建", "create"],
        }
        keywords = intent_keywords.get(expected_intent, [expected_intent])
        return any(kw.lower() in output.lower() for kw in keywords)

    def generate_report(self, results: list[TestResult]) -> str:
        """生成测试报告。"""
        total = len(results)
        passed = sum(1 for r in results if r.passed)
        failed = total - passed

        lines = [
            "=" * 40,
            "Agent 测试报告",
            "=" * 40,
            f"总计: {total} | 通过: {passed} | 失败: {failed}",
            f"通过率: {passed / total * 100:.1f}%" if total > 0 else "无测试",
            "",
        ]

        if failed > 0:
            lines.append("失败的测试:")
            for r in results:
                if not r.passed:
                    lines.append(f"  - {r.test_case.name}: {r.details}")
                    lines.append(f"    输出: {r.actual_output[:200]}")

        return "\n".join(lines)
```

---

## 64.6 部署方案

### 64.6.1 Docker 容器化

```python
# Dockerfile (示意内容，以字符串形式展示)
DOCKERFILE_CONTENT = """
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# 安装依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 复制代码
COPY . .

# 创建非 root 用户
RUN useradd -m agentuser
USER agentuser

# 暴露端口（如果使用 Web 界面）
EXPOSE 8000

# 启动命令
CMD ["python", "main.py"]
"""
```

### 64.6.2 部署检查清单

在部署到生产环境之前，请确保完成以下检查：

环境与配置：API Key 通过环境变量注入（不硬编码），敏感配置使用 Secret Manager，配置了合适的日志级别，设置了合理的超时时间。

错误处理：所有外部 API 调用都有错误处理，实现了重试和降级策略，错误消息不包含敏感信息，有完善的异常上报机制。

性能：实现了响应缓存，API 调用有速率限制，文本处理有长度限制，考虑了并发处理。

安全：输入验证和清洗已就绪，敏感信息脱敏输出，API Key 不会被日志记录，有权限控制机制。

监控：关键指标可监控，有错误告警机制，有健康检查端点，有性能指标记录。

测试：有基本的单元测试，有集成测试用例，有错误场景的测试，压力测试通过。

运维：有 Docker 镜像，有日志持久化方案，有优雅退出处理，有重启恢复机制。

---

## 64.7 练习题

### 练习一：实现完整的错误处理（难度：初级）
选择前面六个项目中的任意一个，为其添加完整的错误处理。包括：异常分类、重试机制、降级策略、用户友好错误消息。

### 练习二：添加监控仪表盘（难度：中级）
使用 Prometheus 和 Grafana（或简单的 Flask + Chart.js），构建一个 Agent 运行状态的监控仪表盘。

### 练习三：实现自动测试（难度：中级）
为研究助手 Agent 创建一个包含 20 个测试用例的自动化测试套件，覆盖搜索、分析、报告生成等核心功能。

### 练习四：优化 API 成本（难度：高级）
分析你的 Agent 的 API 调用模式，识别成本优化机会。实现缓存、批处理、模型降级等策略，将成本降低 50% 以上。

### 练习五：设计完整的 CI/CD 流水线（难度：高级）
为 Agent 项目设计一个完整的 CI/CD 流水线，包括代码检查、自动测试、构建镜像、部署到云平台。

---

## 64.8 实战任务

### 任务一：生产化你的项目

选择一个你最感兴趣的项目，按照本章的建议将其优化到生产级别。重点关注错误处理和日志系统。

### 任务二：压力测试

对优化后的项目进行压力测试。模拟 100 个并发用户的请求，观察系统的响应时间和错误率。

### 任务三：部署到云平台

将项目部署到一个云平台（如 AWS Lambda、Vercel、Railway）。体验从本地开发到云端部署的完整流程。

---

## 64.9 本章小结

从原型到生产是 Agent 开发中最容易被忽视的环节。很多开发者花费大量时间优化核心算法，却忽略了错误处理、日志、监控、安全等"基础设施"。然而正是这些基础设施，决定了 Agent 能否真正投入使用。

关键要点回顾：错误处理要分层、要分类、要有降级策略；日志要用结构化格式、要脱敏、要有级别控制；性能优化要用缓存、要批处理、要选对模型；安全要验证输入、管理密钥、防御注入；测试要自动化、要覆盖边界情况；部署要容器化、要有监控、要有回滚方案。

下一章，我们将讨论如何将你的 Agent 项目展示给他人——好的展示是获得认可和反馈的关键。
