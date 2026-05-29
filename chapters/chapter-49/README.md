# 第49章：Harness Engineering —— Agent 工程的新范式

## 学习目标

通过本章的学习，你将能够：
1. 理解 Harness Engineering 的核心理念和设计原则
2. 掌握 Agent 系统的工程化架构设计方法
3. 学会构建可观测、可测试、可扩展的 Agent 应用
4. 理解 Agent 工程中的质量保证和风险管理
5. 掌握 Prompt 工程、工具管理和流程编排的最佳实践
6. 能够设计和实施完整的 Agent 工程化方案

## 核心问题

在前面的章节中，我们学习了如何构建各种 Agent。但当我们试图将这些 Agent 从实验阶段推向生产阶段时，会遇到一系列工程化的问题：

- 如何保证 Agent 的行为是可预测的？
- 如何测试 Agent 的功能？
- 如何监控 Agent 的运行状态？
- 如何处理 Agent 的错误和异常？
- 如何让多个 Agent 协同工作？

这些问题不仅仅是技术问题，更是工程问题。传统的软件工程有成熟的开发流程、测试方法和运维实践，但 Agent 系统由于其非确定性的本质，需要一套全新的工程方法。

Harness Engineering（驾驭工程）正是为了解决这些问题而提出的新范式。它不是一种具体的框架或工具，而是一套系统化的方法论，帮助我们"驾驭" AI Agent 的不确定性和复杂性。

---

## 原理讲解

### 什么是 Harness Engineering

"Harness"这个词的本意是"驾驭"，就像驾驭一匹野马一样。AI Agent 有着强大的能力，但也有着不可预测性。Harness Engineering 就是关于如何驾驭这种能力和不确定性的工程实践。

传统的软件是确定性的——给定相同的输入，总是产生相同的输出。但 AI Agent 是非确定性的——即使是相同的输入，由于 LLM 的随机性，可能产生不同的输出。这种非确定性给工程实践带来了全新的挑战。

Harness Engineering 的核心原则包括：

1. **可观测性（Observability）**：Agent 的内部状态和决策过程应该是可见的
2. **可测试性（Testability）**：Agent 的功能应该可以通过自动化测试来验证
3. **可控制性（Controllability）**：Agent 的行为应该可以通过配置和策略来控制
4. **可恢复性（Resilience）**：Agent 应该能够优雅地处理错误和异常
5. **可扩展性（Scalability）**：Agent 系统应该能够水平扩展以应对负载

### 可观测性：理解 Agent 在做什么

可观测性是 Harness Engineering 的基础。对于 Agent 系统，可观测性包括三个支柱：

**日志（Logging）**：记录 Agent 的每一步操作、每一次决策、每一次工具调用。日志应该包含足够的上下文信息，以便在问题发生时进行排查。

**指标（Metrics）**：收集 Agent 的运行指标，如响应时间、成功率、工具调用频率、token 使用量等。这些指标帮助我们了解 Agent 的健康状态和性能表现。

**追踪（Tracing）**：追踪一次完整的请求在 Agent 系统中的流转过程。追踪可以帮助我们理解 Agent 的决策路径，识别性能瓶颈。

```
用户请求 → [追踪开始]
    ↓
  解析意图 → [日志：识别到任务类型 X]
    ↓
  选择工具 → [日志：选择工具 Y]
    ↓
  调用工具 → [指标：工具调用耗时 200ms]
    ↓
  生成回复 → [日志：生成回复长度 500 字符]
    ↓
[追踪结束] → [指标：总耗时 1.5s，token 使用 2000]
```

### 可测试性：确保 Agent 做对了

Agent 的测试比传统软件测试更复杂，因为输出是非确定性的。Harness Engineering 提出了几种测试策略：

**确定性测试**：对于 Agent 的确定性部分（如工具调用、数据处理），使用传统的单元测试和集成测试。

**回归测试**：维护一组测试用例，每次 Agent 改进后都运行这些测试，确保没有引入新的问题。

**质量评估**：使用自动化的质量评估指标（如相关性、完整性、可读性）来评估 Agent 的输出质量。

**人工评估**：定期进行人工评估，检查 Agent 的输出是否符合预期。

### 可控制性：管理 Agent 的行为

Agent 的可控制性体现在多个层面：

**Prompt 管理**：通过管理 Prompt 模板来控制 Agent 的行为。不同的 Prompt 可以让 Agent 表现出不同的风格和能力。

**工具策略**：控制 Agent 可以使用哪些工具，以及如何使用这些工具。比如限制 Agent 只能读取文件，不能写入文件。

**流程控制**：定义 Agent 的工作流程。比如在执行敏感操作前必须获得用户确认。

**资源限制**：设置 token 使用限制、时间限制、调用次数限制等，防止 Agent 消耗过多资源。

### 可恢复性：优雅地处理失败

Agent 系统可能会在多个层面失败：
- LLM API 调用失败
- 工具执行失败
- 网络超时
- 依赖服务不可用

Harness Engineering 要求 Agent 能够优雅地处理这些失败：

**降级策略**：当主要方案不可用时，自动切换到备用方案。比如当 Claude 不可用时，切换到 GPT-4。

**重试机制**：对于临时性的错误，自动重试。使用指数退避策略避免重试风暴。

**熔断机制**：当某个服务持续失败时，暂时停止调用它，避免级联故障。

**回滚机制**：当 Agent 执行了错误的操作时，能够撤销这些操作。

### 可扩展性：应对增长

随着用户数量和任务复杂度的增长，Agent 系统需要能够水平扩展：

**无状态设计**：Agent 的核心逻辑应该是无状态的，所有的状态都存储在外部存储中。

**异步处理**：对于耗时的任务，使用异步处理和队列机制。

**缓存策略**：对于重复的请求，使用缓存来减少 LLM 调用。

**负载均衡**：在多个 Agent 实例之间分配请求。

### 工具管理的最佳实践

工具是 Agent 与外界交互的桥梁。良好的工具管理对于 Agent 系统的可靠性至关重要：

**工具注册中心**：维护一个工具注册中心，记录所有可用工具的定义、版本和状态。

**工具版本控制**：工具的接口可能会变化，需要有版本控制机制。

**工具沙箱**：在隔离的环境中执行工具，防止工具的错误影响 Agent 的核心功能。

**工具监控**：监控工具的调用频率、成功率和性能。

### Prompt 工程：Agent 的灵魂

Prompt 是 Agent 行为的核心定义。Harness Engineering 将 Prompt 视为代码来管理：

**版本控制**：所有 Prompt 模板都应该纳入版本控制。

**A/B 测试**：对于关键的 Prompt，进行 A/B 测试来比较效果。

**模板化**：使用模板系统来管理 Prompt，支持变量替换和条件逻辑。

**评估框架**：建立 Prompt 的评估框架，量化不同 Prompt 的效果。

### 流程编排：Agent 的工作流

复杂的 Agent 任务通常需要多个步骤的协调。流程编排定义了这些步骤的执行顺序和交互方式：

**DAG（有向无环图）编排**：将任务分解为多个节点，定义节点之间的依赖关系。

**状态机编排**：定义 Agent 的状态和状态转换条件。

**事件驱动编排**：基于事件来触发 Agent 的行为。

**人机协作编排**：在关键节点引入人工审核和确认。

---

## 完整代码示例

### 示例 1：可观测性框架

```python
"""
Agent 可观测性框架
提供日志、指标和追踪功能
"""

import time
import json
import os
import uuid
from datetime import datetime
from typing import Dict, List, Optional, Any, Callable
from dataclasses import dataclass, field, asdict
from functools import wraps
from collections import defaultdict
import threading


@dataclass
class Span:
    """追踪 span"""
    span_id: str
    trace_id: str
    name: str
    start_time: float
    end_time: Optional[float] = None
    parent_span_id: Optional[str] = None
    attributes: Dict = field(default_factory=dict)
    events: List[Dict] = field(default_factory=list)
    status: str = "OK"
    
    def finish(self, status: str = "OK"):
        self.end_time = time.time()
        self.status = status
    
    @property
    def duration_ms(self) -> float:
        if self.end_time:
            return (self.end_time - self.start_time) * 1000
        return 0


@dataclass
class Metric:
    """指标"""
    name: str
    value: float
    timestamp: float
    tags: Dict = field(default_factory=dict)


class AgentTracer:
    """Agent 追踪器"""
    
    def __init__(self, service_name: str = "agent"):
        self.service_name = service_name
        self.traces: Dict[str, List[Span]] = {}
        self.metrics: List[Metric] = []
        self._lock = threading.Lock()
    
    def start_trace(self, name: str, attributes: Dict = None) -> str:
        """开始一个新的追踪"""
        trace_id = str(uuid.uuid4())[:16]
        span = Span(
            span_id=str(uuid.uuid4())[:8],
            trace_id=trace_id,
            name=name,
            start_time=time.time(),
            attributes=attributes or {},
        )
        
        with self._lock:
            self.traces[trace_id] = [span]
        
        return trace_id
    
    def start_span(self, trace_id: str, name: str, parent_span_id: str = None,
                   attributes: Dict = None) -> str:
        """在追踪中创建新的 span"""
        span_id = str(uuid.uuid4())[:8]
        span = Span(
            span_id=span_id,
            trace_id=trace_id,
            name=name,
            start_time=time.time(),
            parent_span_id=parent_span_id,
            attributes=attributes or {},
        )
        
        with self._lock:
            if trace_id in self.traces:
                self.traces[trace_id].append(span)
        
        return span_id
    
    def finish_span(self, trace_id: str, span_id: str, status: str = "OK"):
        """完成一个 span"""
        with self._lock:
            if trace_id in self.traces:
                for span in self.traces[trace_id]:
                    if span.span_id == span_id:
                        span.finish(status)
                        break
    
    def add_event(self, trace_id: str, span_id: str, name: str, attributes: Dict = None):
        """向 span 添加事件"""
        with self._lock:
            if trace_id in self.traces:
                for span in self.traces[trace_id]:
                    if span.span_id == span_id:
                        span.events.append({
                            "name": name,
                            "timestamp": time.time(),
                            "attributes": attributes or {},
                        })
                        break
    
    def record_metric(self, name: str, value: float, tags: Dict = None):
        """记录指标"""
        metric = Metric(
            name=name,
            value=value,
            timestamp=time.time(),
            tags=tags or {},
        )
        with self._lock:
            self.metrics.append(metric)
    
    def get_trace(self, trace_id: str) -> List[Span]:
        """获取追踪的所有 span"""
        return self.traces.get(trace_id, [])
    
    def get_metrics_summary(self, time_range_seconds: int = 3600) -> Dict:
        """获取指标摘要"""
        cutoff = time.time() - time_range_seconds
        
        with self._lock:
            recent_metrics = [m for m in self.metrics if m.timestamp > cutoff]
        
        # 按名称聚合
        aggregated = defaultdict(list)
        for metric in recent_metrics:
            aggregated[metric.name].append(metric.value)
        
        summary = {}
        for name, values in aggregated.items():
            summary[name] = {
                "count": len(values),
                "sum": sum(values),
                "mean": sum(values) / len(values) if values else 0,
                "min": min(values) if values else 0,
                "max": max(values) if values else 0,
            }
        
        return summary
    
    def export_trace(self, trace_id: str) -> Dict:
        """导出追踪数据"""
        spans = self.get_trace(trace_id)
        return {
            "trace_id": trace_id,
            "spans": [asdict(s) for s in spans],
        }


class AgentLogger:
    """Agent 日志记录器"""
    
    def __init__(self, log_file: str = "agent_logs.jsonl"):
        self.log_file = log_file
        self._lock = threading.Lock()
    
    def log(self, level: str, message: str, context: Dict = None):
        """记录日志"""
        log_entry = {
            "timestamp": datetime.now().isoformat(),
            "level": level,
            "message": message,
            "context": context or {},
        }
        
        with self._lock:
            with open(self.log_file, 'a', encoding='utf-8') as f:
                f.write(json.dumps(log_entry, ensure_ascii=False) + '\n')
    
    def info(self, message: str, context: Dict = None):
        self.log("INFO", message, context)
    
    def warning(self, message: str, context: Dict = None):
        self.log("WARNING", message, context)
    
    def error(self, message: str, context: Dict = None):
        self.log("ERROR", message, context)
    
    def debug(self, message: str, context: Dict = None):
        self.log("DEBUG", message, context)


class AgentMetrics:
    """Agent 指标收集器"""
    
    def __init__(self):
        self.counters = defaultdict(int)
        self.gauges = defaultdict(float)
        self.histograms = defaultdict(list)
        self._lock = threading.Lock()
    
    def increment(self, name: str, value: int = 1, tags: Dict = None):
        """增加计数器"""
        key = f"{name}:{json.dumps(tags or {}, sort_keys=True)}"
        with self._lock:
            self.counters[key] += value
    
    def set_gauge(self, name: str, value: float, tags: Dict = None):
        """设置仪表盘值"""
        key = f"{name}:{json.dumps(tags or {}, sort_keys=True)}"
        with self._lock:
            self.gauges[key] = value
    
    def observe(self, name: str, value: float, tags: Dict = None):
        """观察直方图值"""
        key = f"{name}:{json.dumps(tags or {}, sort_keys=True)}"
        with self._lock:
            self.histograms[key].append(value)
    
    def get_summary(self) -> Dict:
        """获取指标摘要"""
        with self._lock:
            return {
                "counters": dict(self.counters),
                "gauges": dict(self.gauges),
                "histograms": {
                    k: {
                        "count": len(v),
                        "mean": sum(v) / len(v) if v else 0,
                        "min": min(v) if v else 0,
                        "max": max(v) if v else 0,
                    }
                    for k, v in self.histograms.items()
                },
            }


class ObservableAgent:
    """可观测的 Agent"""
    
    def __init__(self):
        self.tracer = AgentTracer("my-agent")
        self.logger = AgentLogger("agent_logs.jsonl")
        self.metrics = AgentMetrics()
        self.client = None  # LLM client
    
    def trace_operation(self, operation_name: str):
        """追踪装饰器"""
        def decorator(func: Callable):
            @wraps(func)
            async def wrapper(*args, **kwargs):
                trace_id = kwargs.get('trace_id') or self.tracer.start_trace(operation_name)
                span_id = self.tracer.start_span(trace_id, operation_name)
                
                start_time = time.time()
                
                try:
                    result = await func(*args, **kwargs)
                    self.tracer.finish_span(trace_id, span_id, "OK")
                    self.metrics.increment("operation_success", tags={"operation": operation_name})
                    return result
                except Exception as e:
                    self.tracer.finish_span(trace_id, span_id, "ERROR")
                    self.tracer.add_event(trace_id, span_id, "error", {"error": str(e)})
                    self.metrics.increment("operation_error", tags={"operation": operation_name})
                    raise
                finally:
                    duration = (time.time() - start_time) * 1000
                    self.metrics.observe("operation_duration_ms", duration, tags={"operation": operation_name})
                    self.logger.info(
                        f"操作完成: {operation_name}",
                        {"duration_ms": duration, "trace_id": trace_id}
                    )
            
            return wrapper
        return decorator
    
    async def process_request(self, request: str) -> str:
        """处理用户请求"""
        trace_id = self.tracer.start_trace("process_request")
        
        self.logger.info("收到请求", {"request": request[:100]})
        self.metrics.increment("requests_total")
        
        try:
            # Step 1: 解析意图
            span_id = self.tracer.start_span(trace_id, "parse_intent")
            intent = self._parse_intent(request)
            self.tracer.finish_span(trace_id, span_id)
            self.metrics.increment("intent_parsed")
            
            # Step 2: 选择工具
            span_id = self.tracer.start_span(trace_id, "select_tool")
            tool = self._select_tool(intent)
            self.tracer.finish_span(trace_id, span_id)
            
            # Step 3: 执行工具
            span_id = self.tracer.start_span(trace_id, "execute_tool", attributes={"tool": tool})
            result = await self._execute_tool(tool, request)
            self.tracer.finish_span(trace_id, span_id)
            self.metrics.increment("tool_executed", tags={"tool": tool})
            
            # Step 4: 生成回复
            span_id = self.tracer.start_span(trace_id, "generate_response")
            response = self._generate_response(result)
            self.tracer.finish_span(trace_id, span_id)
            
            self.logger.info("请求处理完成", {"trace_id": trace_id})
            self.metrics.increment("requests_completed")
            
            return response
            
        except Exception as e:
            self.logger.error("请求处理失败", {"error": str(e), "trace_id": trace_id})
            self.metrics.increment("requests_failed")
            raise
    
    def _parse_intent(self, request: str) -> Dict:
        """解析意图"""
        return {"type": "query", "complexity": "medium"}
    
    def _select_tool(self, intent: Dict) -> str:
        """选择工具"""
        return "default_tool"
    
    async def _execute_tool(self, tool: str, request: str) -> str:
        """执行工具"""
        return f"工具 {tool} 的执行结果"
    
    def _generate_response(self, result: str) -> str:
        """生成回复"""
        return f"基于结果的回复: {result}"


def demo_observability():
    """演示可观测性"""
    print("=" * 60)
    print("可观测性框架演示")
    print("=" * 60)
    
    agent = ObservableAgent()
    
    # 模拟请求处理
    import asyncio
    
    async def run_demo():
        for i in range(3):
            try:
                result = await agent.process_request(f"测试请求 {i+1}")
                print(f"请求 {i+1} 完成: {result}")
            except Exception as e:
                print(f"请求 {i+1} 失败: {e}")
    
    asyncio.run(run_demo())
    
    # 显示指标
    print("\n指标摘要:")
    summary = agent.metrics.get_summary()
    print(json.dumps(summary, indent=2, ensure_ascii=False))
    
    # 显示追踪数据
    print("\n追踪数据:")
    traces = agent.tracer.traces
    for trace_id, spans in traces.items():
        print(f"\nTrace: {trace_id}")
        for span in spans:
            print(f"  {span.name}: {span.duration_ms:.2f}ms ({span.status})")


if __name__ == "__main__":
    demo_observability()
```

### 示例 2：Agent 测试框架

```python
"""
Agent 测试框架
提供 Agent 功能测试的工具
"""

import json
import os
import time
import asyncio
from typing import Dict, List, Optional, Callable, Any
from dataclasses import dataclass, field
import anthropic


@dataclass
class TestCase:
    """测试用例"""
    test_id: str
    description: str
    input_text: str
    expected_behavior: Dict  # 期望的行为
    tags: List[str] = field(default_factory=list)
    timeout_seconds: float = 30.0


@dataclass
class TestResult:
    """测试结果"""
    test_id: str
    passed: bool
    actual_output: str
    score: float  # 0-1
    details: Dict = field(default_factory=dict)
    duration_ms: float = 0
    error: Optional[str] = None


class AgentTestSuite:
    """Agent 测试套件"""
    
    def __init__(self, agent_func: Callable):
        self.agent_func = agent_func
        self.test_cases: List[TestCase] = []
        self.results: List[TestResult] = []
        self.client = anthropic.Anthropic(
            api_key=os.environ.get("ANTHROPIC_API_KEY")
        )
    
    def add_test_case(self, test_id: str, description: str, input_text: str,
                      expected_behavior: Dict, tags: List[str] = None):
        """添加测试用例"""
        test_case = TestCase(
            test_id=test_id,
            description=description,
            input_text=input_text,
            expected_behavior=expected_behavior,
            tags=tags or [],
        )
        self.test_cases.append(test_case)
    
    async def run_test(self, test_case: TestCase) -> TestResult:
        """运行单个测试"""
        start_time = time.time()
        
        try:
            # 运行 Agent
            actual_output = await self.agent_func(test_case.input_text)
            
            # 评估结果
            evaluation = await self._evaluate_output(
                test_case.input_text,
                actual_output,
                test_case.expected_behavior,
            )
            
            duration_ms = (time.time() - start_time) * 1000
            
            return TestResult(
                test_id=test_case.test_id,
                passed=evaluation["passed"],
                actual_output=actual_output,
                score=evaluation["score"],
                details=evaluation,
                duration_ms=duration_ms,
            )
            
        except Exception as e:
            duration_ms = (time.time() - start_time) * 1000
            return TestResult(
                test_id=test_case.test_id,
                passed=False,
                actual_output="",
                score=0.0,
                duration_ms=duration_ms,
                error=str(e),
            )
    
    async def _evaluate_output(self, input_text: str, output: str,
                               expected: Dict) -> Dict:
        """使用 LLM 评估输出质量"""
        
        evaluation_prompt = f"""请评估以下 AI Agent 的输出质量。

用户输入：{input_text}
Agent 输出：{output}

期望行为：
{json.dumps(expected, ensure_ascii=False, indent=2)}

请从以下维度评估（0-1 分）：
1. 相关性：输出是否回答了用户的问题
2. 准确性：输出的信息是否正确
3. 完整性：输出是否涵盖了所有必要的信息
4. 可读性：输出是否清晰易懂

请用 JSON 格式回答：
{{
    "relevance": 0-1,
    "accuracy": 0-1,
    "completeness": 0-1,
    "readability": 0-1,
    "overall_score": 0-1,
    "passed": true/false,
    "issues": ["问题列表"],
    "suggestions": ["改进建议"]
}}
"""
        
        try:
            response = self.client.messages.create(
                model="claude-sonnet-4-20250514",
                max_tokens=1024,
                messages=[{"role": "user", "content": evaluation_prompt}],
            )
            
            text = response.content[0].text
            start = text.find('{')
            end = text.rfind('}') + 1
            
            if start >= 0 and end > start:
                return json.loads(text[start:end])
        except Exception as e:
            print(f"评估失败: {e}")
        
        return {
            "relevance": 0.5,
            "accuracy": 0.5,
            "completeness": 0.5,
            "readability": 0.5,
            "overall_score": 0.5,
            "passed": False,
            "issues": ["评估失败"],
            "suggestions": [],
        }
    
    async def run_all_tests(self) -> List[TestResult]:
        """运行所有测试"""
        self.results = []
        
        for test_case in self.test_cases:
            print(f"\n运行测试: {test_case.test_id} - {test_case.description}")
            result = await self.run_test(test_case)
            self.results.append(result)
            
            status = "✓ 通过" if result.passed else "✗ 失败"
            print(f"  {status} (分数: {result.score:.2f}, 耗时: {result.duration_ms:.0f}ms)")
        
        return self.results
    
    def generate_report(self) -> Dict:
        """生成测试报告"""
        total = len(self.results)
        passed = sum(1 for r in self.results if r.passed)
        failed = total - passed
        
        avg_score = sum(r.score for r in self.results) / total if total > 0 else 0
        avg_duration = sum(r.duration_ms for r in self.results) / total if total > 0 else 0
        
        report = {
            "summary": {
                "total_tests": total,
                "passed": passed,
                "failed": failed,
                "pass_rate": passed / total if total > 0 else 0,
                "average_score": avg_score,
                "average_duration_ms": avg_duration,
            },
            "details": [
                {
                    "test_id": r.test_id,
                    "passed": r.passed,
                    "score": r.score,
                    "duration_ms": r.duration_ms,
                    "error": r.error,
                }
                for r in self.results
            ],
        }
        
        return report


def demo_agent_testing():
    """演示 Agent 测试"""
    print("=" * 60)
    print("Agent 测试框架演示")
    print("=" * 60)
    
    # 定义测试用的 Agent
    async def simple_agent(input_text: str) -> str:
        """一个简单的 Agent"""
        return f"这是一个回复：{input_text}"
    
    # 创建测试套件
    test_suite = AgentTestSuite(simple_agent)
    
    # 添加测试用例
    test_suite.add_test_case(
        test_id="test_001",
        description="基本问答测试",
        input_text="什么是机器学习？",
        expected_behavior={
            "should_contain": ["机器学习", "人工智能", "学习"],
            "should_be_length": "medium",
        },
        tags=["basic", "qa"],
    )
    
    test_suite.add_test_case(
        test_id="test_002",
        description="代码生成测试",
        input_text="写一个 Python 函数计算斐波那契数列",
        expected_behavior={
            "should_contain": ["def", "fibonacci", "return"],
            "should_have_code": True,
        },
        tags=["code", "generation"],
    )
    
    # 运行测试
    import asyncio
    results = asyncio.run(test_suite.run_all_tests())
    
    # 生成报告
    report = test_suite.generate_report()
    
    print("\n" + "=" * 60)
    print("测试报告")
    print("=" * 60)
    print(json.dumps(report, indent=2, ensure_ascii=False))


if __name__ == "__main__":
    demo_agent_testing()
```

### 示例 3：工具管理框架

```python
"""
工具管理框架
提供工具注册、版本控制和监控功能
"""

import json
import time
import os
from typing import Dict, List, Optional, Callable, Any
from dataclasses import dataclass, field
from datetime import datetime
from collections import defaultdict
import hashlib


@dataclass
class ToolDefinition:
    """工具定义"""
    name: str
    description: str
    version: str
    input_schema: Dict
    output_schema: Dict
    tags: List[str] = field(default_factory=list)
    deprecated: bool = False
    created_at: str = field(default_factory=lambda: datetime.now().isoformat())
    updated_at: str = field(default_factory=lambda: datetime.now().isoformat())


@dataclass
class ToolMetrics:
    """工具指标"""
    call_count: int = 0
    success_count: int = 0
    error_count: int = 0
    total_duration_ms: float = 0
    last_called_at: Optional[str] = None
    
    @property
    def success_rate(self) -> float:
        if self.call_count == 0:
            return 0
        return self.success_count / self.call_count
    
    @property
    def avg_duration_ms(self) -> float:
        if self.call_count == 0:
            return 0
        return self.total_duration_ms / self.call_count


class ToolRegistry:
    """工具注册中心"""
    
    def __init__(self, storage_path: str = "tool_registry"):
        self.storage_path = storage_path
        os.makedirs(storage_path, exist_ok=True)
        
        self.tools: Dict[str, ToolDefinition] = {}
        self.tool_functions: Dict[str, Callable] = {}
        self.tool_metrics: Dict[str, ToolMetrics] = {}
        
        self._load_tools()
    
    def _load_tools(self):
        """加载已注册的工具"""
        filepath = os.path.join(self.storage_path, "tools.json")
        if os.path.exists(filepath):
            with open(filepath, 'r', encoding='utf-8') as f:
                data = json.load(f)
                for name, tool_data in data.items():
                    self.tools[name] = ToolDefinition(**tool_data)
                    self.tool_metrics[name] = ToolMetrics()
    
    def _save_tools(self):
        """保存工具定义"""
        filepath = os.path.join(self.storage_path, "tools.json")
        data = {name: {
            "name": t.name,
            "description": t.description,
            "version": t.version,
            "input_schema": t.input_schema,
            "output_schema": t.output_schema,
            "tags": t.tags,
            "deprecated": t.deprecated,
            "created_at": t.created_at,
            "updated_at": t.updated_at,
        } for name, t in self.tools.items()}
        
        with open(filepath, 'w', encoding='utf-8') as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
    
    def register_tool(self, definition: ToolDefinition, func: Callable):
        """注册工具"""
        self.tools[definition.name] = definition
        self.tool_functions[definition.name] = func
        self.tool_metrics[definition.name] = ToolMetrics()
        
        self._save_tools()
        print(f"工具已注册: {definition.name} v{definition.version}")
    
    def unregister_tool(self, name: str):
        """注销工具"""
        if name in self.tools:
            self.tools[name].deprecated = True
            self._save_tools()
            print(f"工具已标记为废弃: {name}")
    
    def get_tool(self, name: str) -> Optional[ToolDefinition]:
        """获取工具定义"""
        return self.tools.get(name)
    
    def get_tool_function(self, name: str) -> Optional[Callable]:
        """获取工具函数"""
        return self.tool_functions.get(name)
    
    def list_tools(self, include_deprecated: bool = False) -> List[ToolDefinition]:
        """列出所有工具"""
        tools = list(self.tools.values())
        if not include_deprecated:
            tools = [t for t in tools if not t.deprecated]
        return tools
    
    def search_tools(self, query: str) -> List[ToolDefinition]:
        """搜索工具"""
        results = []
        query_lower = query.lower()
        
        for tool in self.tools.values():
            if tool.deprecated:
                continue
            
            if (query_lower in tool.name.lower() or
                query_lower in tool.description.lower() or
                any(query_lower in tag.lower() for tag in tool.tags)):
                results.append(tool)
        
        return results
    
    async def execute_tool(self, name: str, arguments: Dict) -> Any:
        """执行工具"""
        if name not in self.tool_functions:
            raise ValueError(f"工具未找到: {name}")
        
        if self.tools[name].deprecated:
            raise ValueError(f"工具已废弃: {name}")
        
        func = self.tool_functions[name]
        metrics = self.tool_metrics[name]
        
        start_time = time.time()
        
        try:
            result = await func(**arguments) if asyncio.iscoroutinefunction(func) else func(**arguments)
            
            metrics.call_count += 1
            metrics.success_count += 1
            metrics.total_duration_ms += (time.time() - start_time) * 1000
            metrics.last_called_at = datetime.now().isoformat()
            
            return result
            
        except Exception as e:
            metrics.call_count += 1
            metrics.error_count += 1
            metrics.total_duration_ms += (time.time() - start_time) * 1000
            
            raise
    
    def get_metrics_report(self) -> Dict:
        """获取工具指标报告"""
        report = {}
        
        for name, metrics in self.tool_metrics.items():
            if metrics.call_count > 0:
                report[name] = {
                    "call_count": metrics.call_count,
                    "success_rate": metrics.success_rate,
                    "avg_duration_ms": metrics.avg_duration_ms,
                    "last_called_at": metrics.last_called_at,
                }
        
        return report


def demo_tool_registry():
    """演示工具注册中心"""
    print("=" * 60)
    print("工具管理框架演示")
    print("=" * 60)
    
    registry = ToolRegistry("demo_tool_registry")
    
    # 定义工具
    def get_weather(city: str) -> str:
        """获取天气"""
        return f"{city}: 晴天, 25°C"
    
    def calculate(expression: str) -> float:
        """计算表达式"""
        return eval(expression)
    
    # 注册工具
    registry.register_tool(
        ToolDefinition(
            name="get_weather",
            description="获取城市天气信息",
            version="1.0.0",
            input_schema={
                "type": "object",
                "properties": {
                    "city": {"type": "string"}
                },
                "required": ["city"]
            },
            output_schema={"type": "string"},
            tags=["weather", "query"],
        ),
        get_weather,
    )
    
    registry.register_tool(
        ToolDefinition(
            name="calculate",
            description="执行数学计算",
            version="1.0.0",
            input_schema={
                "type": "object",
                "properties": {
                    "expression": {"type": "string"}
                },
                "required": ["expression"]
            },
            output_schema={"type": "number"},
            tags=["math", "calculation"],
        ),
        calculate,
    )
    
    # 使用工具
    print("\n使用工具:")
    result = registry.execute_tool("get_weather", {"city": "北京"})
    print(f"  get_weather: {result}")
    
    result = registry.execute_tool("calculate", {"expression": "2 + 3"})
    print(f"  calculate: {result}")
    
    # 搜索工具
    print("\n搜索 '天气' 相关工具:")
    tools = registry.search_tools("天气")
    for tool in tools:
        print(f"  - {tool.name}: {tool.description}")
    
    # 显示指标
    print("\n工具指标:")
    report = registry.get_metrics_report()
    print(json.dumps(report, indent=2, ensure_ascii=False))


if __name__ == "__main__":
    import asyncio
    demo_tool_registry()
```

### 示例 4：流程编排引擎

```python
"""
流程编排引擎
定义和执行 Agent 的工作流
"""

import json
import time
import asyncio
from typing import Dict, List, Optional, Callable, Any, Set
from dataclasses import dataclass, field
from enum import Enum
from collections import defaultdict


class StepStatus(Enum):
    """步骤状态"""
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"
    SKIPPED = "skipped"


@dataclass
class WorkflowStep:
    """工作流步骤"""
    step_id: str
    name: str
    handler: Callable
    dependencies: List[str] = field(default_factory=list)
    timeout_seconds: float = 60.0
    retry_count: int = 0
    retry_delay_seconds: float = 1.0
    condition: Optional[Callable] = None  # 条件函数
    
    # 运行时状态
    status: StepStatus = StepStatus.PENDING
    result: Any = None
    error: Optional[str] = None
    start_time: Optional[float] = None
    end_time: Optional[float] = None


@dataclass
class Workflow:
    """工作流"""
    workflow_id: str
    name: str
    description: str
    steps: Dict[str, WorkflowStep] = field(default_factory=dict)
    context: Dict = field(default_factory=dict)
    
    # 运行时状态
    status: str = "pending"
    start_time: Optional[float] = None
    end_time: Optional[float] = None


class WorkflowEngine:
    """工作流引擎"""
    
    def __init__(self):
        self.workflows: Dict[str, Workflow] = {}
        self.execution_history: List[Dict] = []
    
    def create_workflow(self, workflow_id: str, name: str, description: str) -> Workflow:
        """创建工作流"""
        workflow = Workflow(
            workflow_id=workflow_id,
            name=name,
            description=description,
        )
        self.workflows[workflow_id] = workflow
        return workflow
    
    def add_step(self, workflow_id: str, step: WorkflowStep):
        """添加步骤到工作流"""
        if workflow_id not in self.workflows:
            raise ValueError(f"工作流未找到: {workflow_id}")
        
        self.workflows[workflow_id].steps[step.step_id] = step
    
    async def execute_workflow(self, workflow_id: str, initial_context: Dict = None) -> Dict:
        """执行工作流"""
        if workflow_id not in self.workflows:
            raise ValueError(f"工作流未找到: {workflow_id}")
        
        workflow = self.workflows[workflow_id]
        workflow.context = initial_context or {}
        workflow.status = "running"
        workflow.start_time = time.time()
        
        print(f"\n开始执行工作流: {workflow.name}")
        print(f"描述: {workflow.description}")
        
        try:
            # 拓扑排序确定执行顺序
            execution_order = self._topological_sort(workflow)
            
            # 执行步骤
            for step_id in execution_order:
                step = workflow.steps[step_id]
                
                # 检查依赖是否都完成了
                dependencies_met = all(
                    workflow.steps[dep].status == StepStatus.COMPLETED
                    for dep in step.dependencies
                )
                
                if not dependencies_met:
                    step.status = StepStatus.SKIPPED
                    print(f"  跳过步骤: {step.name} (依赖未满足)")
                    continue
                
                # 检查条件
                if step.condition and not step.condition(workflow.context):
                    step.status = StepStatus.SKIPPED
                    print(f"  跳过步骤: {step.name} (条件不满足)")
                    continue
                
                # 执行步骤
                await self._execute_step(workflow, step)
                
                # 如果步骤失败，终止工作流
                if step.status == StepStatus.FAILED:
                    workflow.status = "failed"
                    print(f"\n工作流执行失败: {step.name}")
                    break
            else:
                # 所有步骤都成功完成
                workflow.status = "completed"
                print(f"\n工作流执行完成: {workflow.name}")
            
        except Exception as e:
            workflow.status = "failed"
            print(f"\n工作流执行异常: {e}")
        
        finally:
            workflow.end_time = time.time()
            duration = workflow.end_time - workflow.start_time
            print(f"执行时间: {duration:.2f} 秒")
        
        # 记录执行历史
        self.execution_history.append({
            "workflow_id": workflow_id,
            "name": workflow.name,
            "status": workflow.status,
            "duration_seconds": workflow.end_time - workflow.start_time,
            "timestamp": time.time(),
        })
        
        return {
            "status": workflow.status,
            "context": workflow.context,
            "steps": {
                sid: {
                    "name": s.name,
                    "status": s.status.value,
                    "result": s.result,
                }
                for sid, s in workflow.steps.items()
            },
        }
    
    async def _execute_step(self, workflow: Workflow, step: WorkflowStep):
        """执行单个步骤"""
        print(f"  执行步骤: {step.name}")
        
        step.status = StepStatus.RUNNING
        step.start_time = time.time()
        
        for attempt in range(step.retry_count + 1):
            try:
                # 执行步骤处理函数
                result = await asyncio.wait_for(
                    step.handler(workflow.context),
                    timeout=step.timeout_seconds,
                )
                
                step.result = result
                step.status = StepStatus.COMPLETED
                
                # 将结果添加到上下文
                workflow.context[f"{step.step_id}_result"] = result
                
                step.end_time = time.time()
                duration = step.end_time - step.start_time
                print(f"    完成 (耗时: {duration:.2f}s)")
                
                return
                
            except asyncio.TimeoutError:
                step.error = f"步骤执行超时 ({step.timeout_seconds}s)"
                print(f"    超时 (尝试 {attempt + 1}/{step.retry_count + 1})")
                
            except Exception as e:
                step.error = str(e)
                print(f"    错误: {e} (尝试 {attempt + 1}/{step.retry_count + 1})")
            
            # 重试前等待
            if attempt < step.retry_count:
                await asyncio.sleep(step.retry_delay_seconds)
        
        # 所有重试都失败了
        step.status = StepStatus.FAILED
        step.end_time = time.time()
    
    def _topological_sort(self, workflow: Workflow) -> List[str]:
        """拓扑排序"""
        # 计算入度
        in_degree = defaultdict(int)
        for step in workflow.steps.values():
            for dep in step.dependencies:
                in_degree[step.step_id] += 1
        
        # BFS 拓扑排序
        queue = [
            sid for sid, step in workflow.steps.items()
            if in_degree[sid] == 0
        ]
        result = []
        
        while queue:
            node = queue.pop(0)
            result.append(node)
            
            # 更新依赖此节点的步骤的入度
            for step in workflow.steps.values():
                if node in step.dependencies:
                    in_degree[step.step_id] -= 1
                    if in_degree[step.step_id] == 0:
                        queue.append(step.step_id)
        
        return result


def demo_workflow():
    """演示工作流引擎"""
    print("=" * 60)
    print("流程编排引擎演示")
    print("=" * 60)
    
    engine = WorkflowEngine()
    
    # 创建工作流
    workflow = engine.create_workflow(
        workflow_id="data_processing",
        name="数据处理工作流",
        description="读取、处理、分析数据的完整流程",
    )
    
    # 定义步骤
    async def read_data(context):
        """读取数据"""
        await asyncio.sleep(0.5)  # 模拟 IO
        return {"records": 100, "source": "database"}
    
    async def clean_data(context):
        """清洗数据"""
        await asyncio.sleep(0.3)
        data = context.get("read_data_result", {})
        return {"records": data.get("records", 0) - 10, "cleaned": True}
    
    async def analyze_data(context):
        """分析数据"""
        await asyncio.sleep(0.4)
        data = context.get("clean_data_result", {})
        return {"statistics": {"mean": 150, "median": 120, "std": 50}}
    
    async def generate_report(context):
        """生成报告"""
        await asyncio.sleep(0.2)
        stats = context.get("analyze_data_result", {})
        return {"report": f"分析完成，共处理 {stats.get('statistics', {}).get('mean', 0)} 条数据"}
    
    # 添加步骤
    engine.add_step("data_processing", WorkflowStep(
        step_id="read_data",
        name="读取数据",
        handler=read_data,
    ))
    
    engine.add_step("data_processing", WorkflowStep(
        step_id="clean_data",
        name="清洗数据",
        handler=clean_data,
        dependencies=["read_data"],
    ))
    
    engine.add_step("data_processing", WorkflowStep(
        step_id="analyze_data",
        name="分析数据",
        handler=analyze_data,
        dependencies=["clean_data"],
    ))
    
    engine.add_step("data_processing", WorkflowStep(
        step_id="generate_report",
        name="生成报告",
        handler=generate_report,
        dependencies=["analyze_data"],
    ))
    
    # 执行工作流
    result = asyncio.run(engine.execute_workflow("data_processing"))
    
    print("\n执行结果:")
    print(json.dumps(result, indent=2, ensure_ascii=False))


if __name__ == "__main__":
    demo_workflow()
```

---

## 案例分析

### 案例 1：生产级客服系统的 Harness Engineering 实践

**系统架构**：

```
用户请求 → 负载均衡 → Agent 集群
                        ↓
                   ┌────┼────┐
                   ↓    ↓    ↓
                  日志  指标  追踪
                   ↓    ↓    ↓
                 可观测性平台
                   ↓
               告警 & 报表
```

**关键实践**：

1. **Prompt 版本控制**：所有的 Prompt 模板都存储在 Git 中，每次修改都有代码审查
2. **A/B 测试**：对于关键的 Prompt 改进，通过 A/B 测试验证效果
3. **熔断机制**：当 LLM API 的错误率超过 10% 时，自动切换到备用模型
4. **限流机制**：每个用户的请求频率限制为每分钟 10 次
5. **全链路追踪**：每个请求都有唯一的 trace_id，可以追踪完整的处理流程

**效果指标**：
- 系统可用性：99.95%
- 平均响应时间：1.2 秒
- 用户满意度：4.2/5.0
- 错误率：0.3%

### 案例 2：多 Agent 协作系统的流程编排

**场景**：一个内容创作平台，需要多个 Agent 协作完成文章创作。

**流程设计**：

```
用户主题 → [策划 Agent] → 大纲
              ↓
         [研究 Agent] → 素材
              ↓
         [写作 Agent] → 初稿
              ↓
         [审核 Agent] → 审核意见
              ↓
         [修改 Agent] → 定稿
              ↓
         [发布 Agent] → 发布
```

**关键设计**：

1. **步骤依赖**：每个步骤都依赖前一个步骤的输出
2. **并行执行**：研究 Agent 可以并行搜索多个信息源
3. **条件分支**：如果审核不通过，跳转回写作 Agent 修改
4. **超时控制**：每个步骤都有超时限制，防止无限等待
5. **错误恢复**：如果某个 Agent 失败，可以使用备用 Agent

---

## 常见坑

### 坑 1：过度设计

不要一开始就构建过于复杂的工程化框架。从简单开始，根据实际需要逐步添加功能。

### 坑 2：忽略非确定性

Agent 的输出是非确定性的，测试时需要考虑这一点。不要期望完全相同的输入产生完全相同的输出。

### 坑 3：日志过多或过少

日志太少无法排查问题，日志太多影响性能。需要找到平衡点。

### 坑 4：没有设置资源限制

不设置 token 限制和时间限制，可能导致 Agent 消耗过多资源或陷入死循环。

### 坑 5：忽略安全性

Agent 可能执行敏感操作，需要有权限控制和操作审计。

---

## 练习题

### 练习 1：可观测性实现
为一个 Agent 实现完整的可观测性，包括：
- 结构化日志记录
- 关键指标收集
- 分布式追踪

### 练习 2：测试套件
为一个 Agent 构建测试套件，包括：
- 功能测试用例
- 质量评估指标
- 自动化测试执行

### 练习 3：工具管理
实现一个工具管理系统，包括：
- 工具注册和发现
- 工具版本控制
- 工具使用监控

### 练习 4：流程编排
设计一个工作流系统，支持：
- 步骤依赖定义
- 并行执行
- 错误处理和重试

### 练习 5：监控告警
实现 Agent 监控告警系统：
- 定义告警规则
- 实现告警通知
- 告警历史管理

### 练习 6：性能优化
优化 Agent 的性能：
- 实现缓存机制
- 优化 Prompt 以减少 token 使用
- 实现请求批处理

---

## 实战任务

### 任务 1：构建生产级 Agent 服务（中等难度）

**功能需求：**
1. 实现完整的可观测性
2. 实现自动化测试
3. 实现工具管理
4. 实现基本的监控告警

**技术要求：**
1. 使用结构化日志
2. 收集关键运行指标
3. 实现单元测试和集成测试
4. 支持健康检查

### 任务 2：构建多 Agent 协作平台（高难度）

**功能需求：**
1. 支持多 Agent 流程编排
2. 实现 Agent 间的通信
3. 支持并行和串行执行
4. 实现完整的监控和调试工具

**技术要求：**
1. 使用工作流引擎
2. 实现 Agent 发现和注册
3. 支持动态流程调整
4. 实现分布式追踪

---

## 本章小结

本章我们深入学习了 Harness Engineering，这是 Agent 工程化的新范式。我们从以下几个方面进行了探索：

**核心理念**：Harness Engineering 的核心是"驾驭" AI Agent 的不确定性和复杂性。通过系统化的工程实践，让 Agent 可观测、可测试、可控制、可恢复、可扩展。

**可观测性**：掌握了日志、指标、追踪三个可观测性支柱的实现方法。可观测性是 Harness Engineering 的基础。

**可测试性**：理解了 Agent 测试的特殊性，学会了使用自动化评估和回归测试来保证 Agent 质量。

**可控制性**：掌握了 Prompt 管理、工具策略、流程控制和资源限制等控制手段。

**可恢复性**：学会了降级策略、重试机制、熔断机制和回滚机制等容错技术。

**流程编排**：掌握了 DAG 编排、条件分支、并行执行等流程编排技术。

Harness Engineering 是将 Agent 从实验推向生产的关键。只有通过系统化的工程实践，才能让 Agent 在实际应用中稳定、可靠地运行。

在下一章中，我们将分析 OpenClaw 框架，看看它如何将 Harness Engineering 的理念融入框架设计。

---

## 延伸阅读

1. Observability Engineering (O'Reilly)
2. Site Reliability Engineering (Google)
3. Designing Data-Intensive Applications (O'Reilly)
4. AI Engineering (Chip Huyen)
5. Building LLM Apps (Anthropic Cookbook)
