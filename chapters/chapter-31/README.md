# 第 31 章：Agent 可观测性 —— 理解 Agent 在做什么

> **本章定位：** 可观测性是 Agent 从"能用"到"好用"的关键跨越。一个看不见内部运作的 Agent 就像一个黑盒子，出了问题你完全不知道该往哪里找原因。本章将带你从零理解可观测性的三大支柱，并通过完整代码示例，帮你为 Agent 构建一套可落地的可观测性体系。

---

## 学习目标

完成本章学习后，你将能够：

1. **理解可观测性的三大支柱** —— 能清晰区分日志（Logs）、指标（Metrics）和追踪（Traces），知道它们各自解决什么问题
2. **为 Agent 设计可观测性架构** —— 能根据 Agent 的复杂度选择合适的可观测性方案，从简单日志到分布式追踪
3. **实现 Agent 的实时监控面板** —— 能用代码搭建一个监控 Agent 运行状态的仪表盘
4. **建立 Agent 的性能基线** —— 能定义和跟踪响应时间、Token 消耗、错误率等关键指标
5. **诊断 Agent 的运行时问题** —— 能通过可观测性数据定位 Agent 的性能瓶颈和逻辑错误

## 核心问题

在开始学习之前，请先思考这三个问题。学完本章后，你应该能清晰地回答它们：

1. **为什么 Agent 比传统应用更需要可观测性？** 传统应用的输入输出是确定性的，Agent 的行为是概率性的——这个区别为什么让可观观测性变得特别重要？
2. **当你看到一个 Agent 给出了错误的回答，你如何知道问题出在哪里？** 是 LLM 理解错了？是工具调用失败了？还是上下文被截断了？没有可观测性，这些问题都无法回答。
3. **可观测性会不会影响 Agent 的性能？** 如何在"看得清楚"和"跑得快"之间取得平衡？

---

## 3.1 为什么 Agent 需要可观观测性

### 3.1.1 从一个真实故事说起

想象你开发了一个客服 Agent，部署在公司官网上。上线第一周，一切正常，客户反馈也不错。到了第二周，突然有用户投诉说 Agent 给出了完全错误的产品价格信息。你赶紧去查，但面对以下困境：

你不知道这个用户问了什么——因为没有记录对话内容。你不知道 Agent 为什么选择了那个回答——因为没有记录推理过程。你不知道 Agent 调用了哪些工具——因为没有记录工具调用链。你甚至不知道这个错误是偶发的还是系统性的——因为你没有统计数据。

这就是没有可观测性的 Agent 的典型困境。你造了一个看起来能工作的黑盒子，但一旦它出了问题，你就完全抓瞎了。

### 3.1.2 传统应用 vs Agent 的可观测性差异

传统 Web 应用的行为是确定性的：同样的输入，同样的处理逻辑，同样的输出。所以你可以写单元测试来覆盖所有分支，用断言来验证每个步骤。但 Agent 不一样——同样的输入，LLM 可能给出不同的输出；同样的推理路径，不同的 temperature 设置可能导致完全不同的决策。

这意味着传统的测试和监控方法在 Agent 面前显得力不从心。你需要一种全新的方式来理解 Agent 在做什么，以及它为什么这样做。可观测性就是这个全新的方式。

可观测性（Observability）不同于监控（Monitoring）。监控是预先定义好要看什么指标，然后持续收集和告警。可观测性则是让你能够从外部输出推断系统内部状态的能力——即使你没有预先想到要观察某个方面。对于 Agent 这种行为不确定的系统来说，可观测性比传统监控更加重要。

### 3.1.3 可观测性的三大支柱

可观测性建立在三大支柱之上，就像一个三角形的三条边，缺一不可。

**日志（Logs）** 是最基础的。每一条日志就是一个事件的文本记录，带有时间戳和级别。在 Agent 场景中，日志可以记录用户输入、LLM 响应、工具调用结果、决策过程等。日志的优势是灵活和详细，你可以随时决定记录什么。劣势是海量日志难以快速检索和分析。

**指标（Metrics）** 是可量化的数值数据。比如 Agent 的平均响应时间、每分钟处理的请求数、Token 消耗率、错误率等。指标的优势是可以做统计分析、趋势判断和阈值告警。劣势是它丢失了上下文——你知道错误率升高了，但不知道为什么升高。

**追踪（Traces）** 是将一个请求在系统中的完整生命周期串联起来。对于 Agent 来说，一个追踪可以展示从接收用户输入到返回最终回答的每一步：推理、工具调用、再次推理、输出。追踪的优势是能够完整还原一个请求的执行路径，非常适合调试复杂的行为问题。劣势是存储和查询成本较高。

三者的关系是互补的：日志提供细节，指标提供趋势，追踪提供上下文。一个好的可观测性体系需要三者配合使用。

---

## 3.2 Agent 可观测性架构设计

### 3.2.1 需要观测的关键节点

在设计可观测性架构之前，我们需要先搞清楚：Agent 运行过程中，哪些节点是最值得观测的？

**请求入口** —— 当用户发送一条消息给 Agent 时，记录请求 ID、用户 ID、时间戳、输入内容（或其摘要）。这是追踪的起点。

**LLM 调用** —— 每次调用 LLM 都应该记录：调用了哪个模型、输入的 Token 数、输出的 Token 数、响应时间、temperature 设置、是否触发了工具调用。这是 Agent 最核心的环节。

**工具调用** —— Agent 决定调用工具时，记录：调用了哪个工具、传入了什么参数、工具返回了什么结果、耗时多久、是否出错。

**推理决策** —— Agent 的每一步决策都应该有记录：为什么选择调用这个工具而不是那个？为什么决定直接回答而不是继续推理？

**最终输出** —— Agent 给用户的最终回答，以及生成这个回答的完整推理链路。

**异常事件** —— 超时、Token 超限、工具调用失败、LLM 返回异常等所有非预期情况。

### 3.2.2 从简单到复杂的三层架构

不是每个项目都需要一开始就搭建复杂的可观测性系统。根据项目的规模和阶段，可以选择不同层次的架构。

**第一层：文件日志** —— 最简单的方案，适合个人项目和原型开发阶段。把所有日志写入文件，用 Python 的 logging 模块就够了。优点是零成本，缺点是无法实时查看、无法做统计分析。

**第二层：结构化日志 + 本地存储** —— 适合小型团队和内部工具。用 JSON 格式记录结构化日志，存入本地的 SQLite 或者简单的日志聚合服务。优点是可查询、可分析，缺点是扩展性有限。

**第三层：分布式追踪 + 专业工具** —— 适合生产环境和多 Agent 协作场景。使用 OpenTelemetry 标准，配合 LangSmith、Jaeger 或 Zipkin 等专业工具。优点是功能全面，缺点是学习成本和运维成本较高。

### 3.2.3 与 LLM 可观测性的融合

Agent 的可观观测性不是从零开始的——LLM 的可观测性是它的基础。各大 LLM 提供商和工具链都在推进可观测性标准化。LangSmith、Langfuse、Phoenix（Arize）等工具提供了专门针对 LLM 应用的追踪和分析能力。

关键是理解：Agent 的可观测性 = LLM 的可观测性 + 工具调用的可观测性 + 业务逻辑的可观测性。你不能只看 LLM 的输出，还要看 LLM 是基于什么信息做出的决策，以及这些决策在业务层面意味着什么。

---

## 3.3 实现 Agent 可观测性：完整代码示例

### 3.3.1 基础层：结构化日志系统

让我们从最基础的部分开始，构建一个专门为 Agent 设计的日志系统。这个系统会自动记录 Agent 运行过程中的所有关键事件，并且使用结构化的 JSON 格式，方便后续的查询和分析。

```python
import json
import time
import uuid
import logging
from datetime import datetime
from typing import Any, Optional, Dict, List
from dataclasses import dataclass, field, asdict
from enum import Enum
from pathlib import Path
from functools import wraps

# ============================================================
# 1. 定义 Agent 事件类型
# ============================================================

class EventType(Enum):
    """Agent 运行过程中可能发生的所有事件类型"""
    REQUEST_START = "request_start"           # 用户请求开始
    REQUEST_END = "request_end"               # 用户请求结束
    LLM_CALL_START = "llm_call_start"        # LLM 调用开始
    LLM_CALL_END = "llm_call_end"            # LLM 调用结束
    TOOL_CALL_START = "tool_call_start"      # 工具调用开始
    TOOL_CALL_END = "tool_call_end"          # 工具调用结束
    REASONING_STEP = "reasoning_step"        # 推理步骤
    ERROR = "error"                           # 错误事件
    GUARDRAIL_TRIGGERED = "guardrail"         # 安全护栏触发
    CONTEXT_TRUNCATED = "context_truncated"   # 上下文被截断

class EventLevel(Enum):
    """事件级别"""
    DEBUG = "debug"
    INFO = "info"
    WARNING = "warning"
    ERROR = "error"

# ============================================================
# 2. 事件数据结构
# ============================================================

@dataclass
class AgentEvent:
    """一个 Agent 事件的完整数据结构"""
    event_id: str = field(default_factory=lambda: str(uuid.uuid4())[:8])
    event_type: EventType = EventType.REQUEST_START
    level: EventLevel = EventLevel.INFO
    timestamp: str = field(default_factory=lambda: datetime.now().isoformat())
    request_id: str = ""
    span_id: str = ""                         # 用于关联同一次调用链
    duration_ms: float = 0.0
    data: Dict[str, Any] = field(default_factory=dict)
    
    def to_json(self) -> str:
        """转换为 JSON 字符串"""
        d = asdict(self)
        d["event_type"] = self.event_type.value
        d["level"] = self.level.value
        return json.dumps(d, ensure_ascii=False, default=str)

# ============================================================
# 3. Agent 追踪器（核心）
# ============================================================

class AgentTracer:
    """
    Agent 追踪器 - 记录 Agent 运行过程中的所有事件
    
    使用方式：
        tracer = AgentTracer(log_dir="./agent_logs")
        with tracer.trace_request("user_123") as ctx:
            # 你的 Agent 逻辑
            pass
    """
    
    def __init__(self, log_dir: str = "./agent_logs", 
                 console_output: bool = True,
                 json_file: bool = True):
        self.log_dir = Path(log_dir)
        self.log_dir.mkdir(parents=True, exist_ok=True)
        self.console_output = console_output
        self.json_file = json_file
        
        # 设置日志记录器
        self.logger = logging.getLogger("agent_tracer")
        self.logger.setLevel(logging.DEBUG)
        
        # 控制台输出
        if console_output:
            console_handler = logging.StreamHandler()
            console_handler.setLevel(logging.INFO)
            console_handler.setFormatter(logging.Formatter(
                "%(asctime)s [%(levelname)s] %(message)s",
                datefmt="%H:%M:%S"
            ))
            self.logger.addHandler(console_handler)
        
        # JSON 文件输出
        if json_file:
            today = datetime.now().strftime("%Y-%m-%d")
            file_handler = logging.FileHandler(
                self.log_dir / f"agent_{today}.jsonl",
                encoding="utf-8"
            )
            file_handler.setLevel(logging.DEBUG)
            self.logger.addHandler(file_handler)
        
        # 追踪状态
        self._active_spans: Dict[str, Dict] = {}
        self._events: List[AgentEvent] = []
        self._stats = {
            "total_requests": 0,
            "total_llm_calls": 0,
            "total_tool_calls": 0,
            "total_errors": 0,
            "total_tokens_used": 0,
            "llm_call_durations": [],
            "tool_call_durations": [],
        }
    
    def _emit(self, event: AgentEvent):
        """发出一个事件"""
        self._events.append(event)
        self.logger.log(
            getattr(logging, event.level.value.upper(), logging.INFO),
            event.to_json()
        )
    
    def _update_stats(self, event: AgentEvent):
        """更新统计数据"""
        if event.event_type == EventType.REQUEST_START:
            self._stats["total_requests"] += 1
        elif event.event_type == EventType.LLM_CALL_END:
            self._stats["total_llm_calls"] += 1
            if event.duration_ms > 0:
                self._stats["llm_call_durations"].append(event.duration_ms)
            tokens = event.data.get("total_tokens", 0)
            self._stats["total_tokens_used"] += tokens
        elif event.event_type == EventType.TOOL_CALL_END:
            self._stats["total_tool_calls"] += 1
            if event.duration_ms > 0:
                self._stats["tool_call_durations"].append(event.tool_call_durations if hasattr(event, 'tool_call_durations') else 0)
        elif event.event_type == EventType.ERROR:
            self._stats["total_errors"] += 1
    
    def trace_request(self, user_id: str = "anonymous"):
        """上下文管理器：追踪一个完整的用户请求"""
        return RequestSpan(self, user_id)
    
    def emit_event(self, event: AgentEvent):
        """直接发出一个事件"""
        self._emit(event)
        self._update_stats(event)
    
    def get_stats(self) -> Dict:
        """获取统计数据摘要"""
        stats = dict(self._stats)
        durations = stats.pop("llm_call_durations", [])
        tool_durations = stats.pop("tool_call_durations", [])
        
        if durations:
            stats["avg_llm_duration_ms"] = sum(durations) / len(durations)
            stats["p95_llm_duration_ms"] = sorted(durations)[int(len(durations) * 0.95)] if len(durations) > 1 else durations[0]
        
        if tool_durations:
            stats["avg_tool_duration_ms"] = sum(tool_durations) / len(tool_durations)
        
        return stats
    
    def export_events(self, filepath: str):
        """导出所有事件到 JSON 文件"""
        events_data = [asdict(e) for e in self._events]
        for e in events_data:
            e["event_type"] = e["event_type"].value if isinstance(e["event_type"], EventType) else e["event_type"]
            e["level"] = e["level"].value if isinstance(e["level"], EventLevel) else e["level"]
        
        with open(filepath, "w", encoding="utf-8") as f:
            json.dump(events_data, f, ensure_ascii=False, indent=2, default=str)


class RequestSpan:
    """
    请求级别的追踪 Span
    
    管理一个请求的完整生命周期，自动记录开始和结束事件。
    """
    
    def __init__(self, tracer: AgentTracer, user_id: str):
        self.tracer = tracer
        self.request_id = str(uuid.uuid4())[:12]
        self.user_id = user_id
        self.span_id = str(uuid.uuid4())[:8]
        self.start_time: float = 0
        self.step_count: int = 0
    
    def __enter__(self):
        self.start_time = time.time()
        self.tracer.emit_event(AgentEvent(
            event_type=EventType.REQUEST_START,
            request_id=self.request_id,
            span_id=self.span_id,
            data={"user_id": self.user_id}
        ))
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        duration_ms = (time.time() - self.start_time) * 1000
        self.tracer.emit_event(AgentEvent(
            event_type=EventType.REQUEST_END if not exc_type else EventType.ERROR,
            request_id=self.request_id,
            span_id=self.span_id,
            duration_ms=duration_ms,
            data={
                "user_id": self.user_id,
                "steps": self.step_count,
                "error": str(exc_val) if exc_val else None,
            }
        ))
        return False  # 不抑制异常
    
    def llm_call(self, model: str, input_tokens: int, output_tokens: int, 
                 temperature: float = 0.7, **kwargs):
        """记录一次 LLM 调用"""
        self.step_count += 1
        return LLMCallSpan(self, model, input_tokens, output_tokens, temperature, **kwargs)
    
    def tool_call(self, tool_name: str, arguments: dict):
        """记录一次工具调用"""
        self.step_count += 1
        return ToolCallSpan(self, tool_name, arguments)
    
    def reasoning(self, step: str, details: dict = None):
        """记录一个推理步骤"""
        self.tracer.emit_event(AgentEvent(
            event_type=EventType.REASONING_STEP,
            request_id=self.request_id,
            span_id=self.span_id,
            data={
                "step_number": self.step_count,
                "reasoning": step,
                "details": details or {},
            }
        ))


class LLMCallSpan:
    """LLM 调用的追踪 Span"""
    
    def __init__(self, parent: RequestSpan, model: str, 
                 input_tokens: int, output_tokens: int, temperature: float, **kwargs):
        self.parent = parent
        self.model = model
        self.input_tokens = input_tokens
        self.output_tokens = output_tokens
        self.temperature = temperature
        self.extra = kwargs
        self.start_time = time.time()
        
        self.parent.tracer.emit_event(AgentEvent(
            event_type=EventType.LLM_CALL_START,
            request_id=self.parent.request_id,
            span_id=self.parent.span_id,
            data={
                "model": model,
                "input_tokens": input_tokens,
                "temperature": temperature,
                **kwargs,
            }
        ))
    
    def __enter__(self):
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        duration_ms = (time.time() - self.start_time) * 1000
        self.parent.tracer.emit_event(AgentEvent(
            event_type=EventType.LLM_CALL_END if not exc_type else EventType.ERROR,
            request_id=self.parent.request_id,
            span_id=self.parent.span_id,
            duration_ms=duration_ms,
            data={
                "model": self.model,
                "input_tokens": self.input_tokens,
                "output_tokens": self.output_tokens,
                "total_tokens": self.input_tokens + self.output_tokens,
                "temperature": self.temperature,
                "error": str(exc_val) if exc_val else None,
                **self.extra,
            }
        ))
        return False


class ToolCallSpan:
    """工具调用的追踪 Span"""
    
    def __init__(self, parent: RequestSpan, tool_name: str, arguments: dict):
        self.parent = parent
        self.tool_name = tool_name
        self.arguments = arguments
        self.start_time = time.time()
        
        self.parent.tracer.emit_event(AgentEvent(
            event_type=EventType.TOOL_CALL_START,
            request_id=self.parent.request_id,
            span_id=self.parent.span_id,
            data={
                "tool_name": tool_name,
                "arguments": arguments,
            }
        ))
    
    def __enter__(self):
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        duration_ms = (time.time() - self.start_time) * 1000
        self.parent.tracer.emit_event(AgentEvent(
            event_type=EventType.TOOL_CALL_END if not exc_type else EventType.ERROR,
            request_id=self.parent.request_id,
            span_id=self.parent.span_id,
            duration_ms=duration_ms,
            data={
                "tool_name": self.tool_name,
                "error": str(exc_val) if exc_val else None,
            }
        ))
        return False
```

### 3.3.2 将追踪器集成到 Agent 中

有了追踪器之后，我们需要把它集成到 Agent 的核心循环中。下面是一个完整的、可运行的 Agent 示例，展示了如何在每个关键节点使用追踪器。

```python
import os
import time
import json
import random
from typing import List, Dict, Any

# 假设你已经安装了 openai
# pip install openai
from openai import OpenAI

# ============================================================
# 完整的可追踪 Agent
# ============================================================

class ObservableAgent:
    """
    一个具备完整可观测性的 Agent
    
    这个 Agent 演示了如何将追踪器集成到 Agent 的每个关键环节。
    它支持工具调用、多步推理，并且每一步都有详细的追踪记录。
    """
    
    def __init__(self, model: str = "gpt-4o"):
        self.model = model
        self.client = OpenAI(api_key=os.environ.get("OPENAI_API_KEY"))
        self.tracer = AgentTracer(log_dir="./agent_logs", console_output=True)
        self.tools = self._register_tools()
        self.system_prompt = """你是一个智能助手。你可以使用以下工具来帮助用户：
        
1. search_web(query) - 搜索网页信息
2. calculator(expression) - 计算数学表达式
3. get_weather(city) - 获取天气信息

当你需要使用工具时，请使用 JSON 格式指定工具名称和参数。
当你已经有了足够的信息来回答用户时，请直接给出回答。"""
    
    def _register_tools(self) -> Dict[str, Any]:
        """注册可用工具"""
        return {
            "search_web": self._search_web,
            "calculator": self._calculator,
            "get_weather": self._get_weather,
        }
    
    def _search_web(self, query: str) -> str:
        """模拟搜索网页"""
        time.sleep(0.1)  # 模拟网络延迟
        return f"搜索结果：关于 '{query}' 的最新信息..."
    
    def _calculator(self, expression: str) -> str:
        """安全地计算数学表达式"""
        try:
            # 注意：生产环境应该用更安全的表达式解析器
            allowed_names = {"abs": abs, "round": round, "min": min, "max": max}
            result = eval(expression, {"__builtins__": {}}, allowed_names)
            return str(result)
        except Exception as e:
            return f"计算错误: {e}"
    
    def _get_weather(self, city: str) -> str:
        """模拟获取天气"""
        time.sleep(0.05)
        temp = random.randint(15, 35)
        return f"{city} 天气：晴，温度 {temp}°C"
    
    def run(self, user_input: str, user_id: str = "anonymous") -> str:
        """
        运行 Agent，处理用户输入
        
        这是 Agent 的主入口。每次调用都会创建一个完整的追踪上下文。
        """
        with self.tracer.trace_request(user_id) as ctx:
            # 记录用户输入
            ctx.reasoning("收到用户输入", {"input": user_input[:200]})
            
            messages = [
                {"role": "system", "content": self.system_prompt},
                {"role": "user", "content": user_input}
            ]
            
            max_iterations = 5
            iteration = 0
            
            while iteration < max_iterations:
                iteration += 1
                ctx.reasoning(f"推理轮次 {iteration}", {
                    "messages_count": len(messages),
                    "iteration": iteration,
                })
                
                # 调用 LLM
                response_text = self._call_llm(ctx, messages)
                
                # 检查是否需要工具调用
                tool_call = self._parse_tool_call(response_text)
                
                if tool_call:
                    # 执行工具调用
                    tool_result = self._execute_tool(ctx, tool_call)
                    
                    # 将工具结果添加到消息历史
                    messages.append({"role": "assistant", "content": response_text})
                    messages.append({
                        "role": "user", 
                        "content": f"工具 {tool_call['name']} 返回结果: {tool_result}\n\n请根据这个结果继续回答用户的问题。"
                    })
                else:
                    # 不需要工具调用，直接返回回答
                    ctx.reasoning("生成最终回答", {"response_length": len(response_text)})
                    return response_text
            
            # 达到最大迭代次数
            return "抱歉，我无法在有限的步骤内完成这个任务。请尝试简化你的问题。"
    
    def _call_llm(self, ctx: 'RequestSpan', messages: List[Dict]) -> str:
        """调用 LLM 并记录追踪信息"""
        with ctx.llm_call(
            model=self.model,
            input_tokens=sum(len(m["content"]) // 2 for m in messages),  # 粗略估计
            output_tokens=0,  # 调用后更新
            temperature=0.7,
            messages_count=len(messages),
        ) as llm_span:
            try:
                response = self.client.chat.completions.create(
                    model=self.model,
                    messages=messages,
                    temperature=0.7,
                    max_tokens=1000,
                )
                
                response_text = response.choices[0].message.content
                
                # 更新 token 统计
                if response.usage:
                    llm_span.extra["input_tokens"] = response.usage.prompt_tokens
                    llm_span.extra["output_tokens"] = response.usage.completion_tokens
                    llm_span.extra["total_tokens"] = response.usage.total_tokens
                
                return response_text
                
            except Exception as e:
                ctx.tracer.emit_event(AgentEvent(
                    event_type=EventType.ERROR,
                    request_id=ctx.request_id,
                    span_id=ctx.span_id,
                    level=EventLevel.ERROR,
                    data={"error": str(e), "stage": "llm_call"}
                ))
                raise
    
    def _parse_tool_call(self, text: str) -> Dict:
        """从 LLM 响应中解析工具调用"""
        # 简单的解析逻辑，生产环境应该更健壮
        if "search_web(" in text:
            start = text.index("search_web(") + len("search_web(")
            end = text.index(")", start)
            query = text[start:end].strip().strip("'\"")
            return {"name": "search_web", "arguments": {"query": query}}
        elif "calculator(" in text:
            start = text.index("calculator(") + len("calculator(")
            end = text.index(")", start)
            expr = text[start:end].strip().strip("'\"")
            return {"name": "calculator", "arguments": {"expression": expr}}
        elif "get_weather(" in text:
            start = text.index("get_weather(") + len("get_weather(")
            end = text.index(")", start)
            city = text[start:end].strip().strip("'\"")
            return {"name": "get_weather", "arguments": {"city": city}}
        return None
    
    def _execute_tool(self, ctx: 'RequestSpan', tool_call: Dict) -> str:
        """执行工具调用并记录追踪信息"""
        tool_name = tool_call["name"]
        arguments = tool_call["arguments"]
        
        ctx.reasoning(f"决定调用工具: {tool_name}", {"arguments": arguments})
        
        with ctx.tool_call(tool_name, arguments):
            tool_func = self.tools.get(tool_name)
            if not tool_func:
                return f"错误：未知工具 {tool_name}"
            try:
                result = tool_func(**arguments)
                return result
            except Exception as e:
                return f"工具调用失败: {e}"
    
    def get_stats(self) -> Dict:
        """获取 Agent 的运行统计"""
        return self.tracer.get_stats()


# ============================================================
# 使用示例
# ============================================================

def main():
    """演示如何使用可观测性 Agent"""
    # 创建 Agent
    agent = ObservableAgent(model="gpt-4o")
    
    # 模拟多轮对话
    test_queries = [
        ("帮我查一下北京的天气", "user_001"),
        ("计算 123 * 456 + 789", "user_002"),
        ("搜索最新的 AI 新闻，然后总结一下", "user_001"),
    ]
    
    for query, user_id in test_queries:
        print(f"\n{'='*60}")
        print(f"用户 [{user_id}]: {query}")
        print(f"{'='*60}")
        
        response = agent.run(query, user_id)
        print(f"\nAgent: {response}")
    
    # 打印统计信息
    print(f"\n{'='*60}")
    print("运行统计:")
    stats = agent.get_stats()
    for key, value in stats.items():
        print(f"  {key}: {value}")
    
    # 导出追踪数据
    agent.tracer.export_events("./agent_logs/trace_export.json")
    print("\n追踪数据已导出到 ./agent_logs/trace_export.json")


if __name__ == "__main__":
    main()
```

### 3.3.3 实时监控面板

有了结构化的日志数据，我们可以搭建一个简单的监控面板来实时查看 Agent 的运行状态。这里我们用一个基于终端的简易面板，不需要任何 Web 框架。

```python
import os
import json
import time
import threading
from datetime import datetime, timedelta
from collections import defaultdict, deque
from typing import Dict, List

class AgentMonitor:
    """
    Agent 实时监控面板
    
    在终端中实时展示 Agent 的运行状态，包括：
    - 请求吞吐量
    - 响应时间分布
    - Token 消耗趋势
    - 错误率
    - 活跃请求
    """
    
    def __init__(self, tracer: AgentTracer, window_seconds: int = 60):
        self.tracer = tracer
        self.window_seconds = window_seconds
        
        # 滑动窗口数据
        self._request_times: deque = deque()
        self._llm_durations: deque = deque()
        self._tool_durations: deque = deque()
        self._errors: deque = deque()
        self._token_counts: deque = deque()
        
        # 当前活跃的请求
        self._active_requests: Dict[str, Dict] = {}
        
        # 历史快照
        self._snapshots: List[Dict] = []
    
    def _clean_old_data(self):
        """清理过期的滑动窗口数据"""
        cutoff = time.time() - self.window_seconds
        while self._request_times and self._request_times[0] < cutoff:
            self._request_times.popleft()
        while self._llm_durations and self._llm_durations[0][0] < cutoff:
            self._llm_durations.popleft()
        while self._tool_durations and self._tool_durations[0][0] < cutoff:
            self._tool_durations.popleft()
        while self._errors and self._errors[0] < cutoff:
            self._errors.popleft()
        while self._token_counts and self._token_counts[0][0] < cutoff:
            self._token_counts.popleft()
    
    def on_event(self, event: AgentEvent):
        """处理一个追踪事件，更新监控数据"""
        now = time.time()
        
        if event.event_type == EventType.REQUEST_START:
            self._request_times.append(now)
            self._active_requests[event.request_id] = {
                "user_id": event.data.get("user_id", "unknown"),
                "start_time": datetime.now().strftime("%H:%M:%S"),
                "steps": 0,
            }
        
        elif event.event_type == EventType.REQUEST_END:
            self._active_requests.pop(event.request_id, None)
        
        elif event.event_type == EventType.LLM_CALL_END:
            self._llm_durations.append((now, event.duration_ms))
            tokens = event.data.get("total_tokens", 0)
            if tokens > 0:
                self._token_counts.append((now, tokens))
        
        elif event.event_type == EventType.TOOL_CALL_END:
            self._tool_durations.append((now, event.duration_ms))
        
        elif event.event_type == EventType.ERROR:
            self._errors.append(now)
    
    def take_snapshot(self) -> Dict:
        """捕获当前时刻的监控快照"""
        self._clean_old_data()
        
        snapshot = {
            "timestamp": datetime.now().isoformat(),
            "requests_in_window": len(self._request_times),
            "active_requests": len(self._active_requests),
            "errors_in_window": len(self._errors),
            "error_rate": len(self._errors) / max(len(self._request_times), 1),
        }
        
        # 计算 LLM 延迟统计
        if self._llm_durations:
            durations = [d[1] for d in self._llm_durations]
            snapshot["llm_avg_duration_ms"] = sum(durations) / len(durations)
            sorted_d = sorted(durations)
            snapshot["llm_p50_duration_ms"] = sorted_d[len(sorted_d) // 2]
            snapshot["llm_p95_duration_ms"] = sorted_d[int(len(sorted_d) * 0.95)]
        
        # 计算 Token 消耗
        if self._token_counts:
            total_tokens = sum(t[1] for t in self._token_counts)
            snapshot["total_tokens_in_window"] = total_tokens
            snapshot["avg_tokens_per_request"] = total_tokens / max(len(self._token_counts), 1)
        
        self._snapshots.append(snapshot)
        return snapshot
    
    def print_dashboard(self):
        """在终端打印监控面板"""
        snapshot = self.take_snapshot()
        
        # 清屏
        os.system("cls" if os.name == "nt" else "clear")
        
        print("=" * 70)
        print("  Agent 可观测性监控面板")
        print(f"  更新时间: {snapshot['timestamp']}")
        print(f"  滑动窗口: {self.window_seconds} 秒")
        print("=" * 70)
        
        # 请求统计
        print("\n[请求统计]")
        print(f"  窗口内请求数: {snapshot.get('requests_in_window', 0)}")
        print(f"  当前活跃请求: {snapshot.get('active_requests', 0)}")
        print(f"  窗口内错误数: {snapshot.get('errors_in_window', 0)}")
        print(f"  错误率: {snapshot.get('error_rate', 0):.2%}")
        
        # 延迟统计
        if "llm_avg_duration_ms" in snapshot:
            print("\n[LLM 延迟 (ms)]")
            print(f"  平均: {snapshot['llm_avg_duration_ms']:.0f}")
            print(f"  P50:  {snapshot.get('llm_p50_duration_ms', 0):.0f}")
            print(f"  P95:  {snapshot.get('llm_p95_duration_ms', 0):.0f}")
        
        # Token 统计
        if "total_tokens_in_window" in snapshot:
            print("\n[Token 消耗]")
            print(f"  窗口内总 Token: {snapshot['total_tokens_in_window']}")
            print(f"  平均每次请求: {snapshot.get('avg_tokens_per_request', 0):.0f}")
        
        # 活跃请求
        if self._active_requests:
            print("\n[活跃请求]")
            for req_id, info in self._active_requests.items():
                print(f"  {req_id}: user={info['user_id']} since={info['start_time']} steps={info['steps']}")
        
        # 简易吞吐量图
        print("\n[请求吞吐量趋势]")
        self._print_mini_chart()
        
        print("\n" + "=" * 70)
        print("  按 Ctrl+C 停止监控")
        print("=" * 70)
    
    def _print_mini_chart(self, width: int = 50, height: int = 8):
        """打印一个简易的 ASCII 柱状图"""
        if not self._snapshots:
            print("  暂无数据")
            return
        
        # 取最近的快照
        recent = self._snapshots[-width:]
        values = [s.get("requests_in_window", 0) for s in recent]
        
        if not values or max(values) == 0:
            print("  暂无请求数据")
            return
        
        max_val = max(values)
        for row in range(height, 0, -1):
            threshold = max_val * row / height
            line = "  "
            for v in values:
                if v >= threshold:
                    line += "█ "
                else:
                    line += "  "
            line += f" |{threshold:.0f}" if row == height or row == 1 else ""
            print(line)
        print("  " + "─" * (width * 2))
    
    def start_continuous_monitoring(self, interval: float = 2.0):
        """启动持续监控（每 interval 秒刷新一次）"""
        try:
            while True:
                self.print_dashboard()
                time.sleep(interval)
        except KeyboardInterrupt:
            print("\n监控已停止")


# ============================================================
# 将监控面板接入追踪器
# ============================================================

class ObservableAgentWithDashboard(ObservableAgent):
    """带监控面板的 Agent"""
    
    def __init__(self, model: str = "gpt-4o"):
        super().__init__(model)
        self.monitor = AgentMonitor(self.tracer)
        
        # 注册事件监听器
        original_emit = self.tracer._emit
    
        def monitored_emit(event):
            original_emit(event)
            self.monitor.on_event(event)
        
        self.tracer._emit = monitored_emit
    
    def start_monitor_dashboard(self):
        """在后台线程启动监控面板"""
        monitor_thread = threading.Thread(
            target=self.monitor.start_continuous_monitoring,
            daemon=True,
        )
        monitor_thread.start()
        return monitor_thread
```

---

## 3.4 进阶：自定义指标收集器

### 3.4.1 指标收集器的设计

除了事件追踪之外，我们还需要一个专门的指标收集器来记录可量化的性能数据。指标收集器和事件追踪器的区别在于：追踪器记录的是每一个具体事件，而指标收集器关注的是汇总统计。

```python
import time
import threading
import statistics
from collections import defaultdict
from typing import Callable, Optional

class MetricsCollector:
    """
    Agent 指标收集器
    
    收集和聚合 Agent 运行过程中的关键指标。
    支持计数器、直方图和速率计算。
    """
    
    def __init__(self):
        self._counters: Dict[str, int] = defaultdict(int)
        self._gauges: Dict[str, float] = {}
        self._histograms: Dict[str, List[float]] = defaultdict(list)
        self._rate_windows: Dict[str, deque] = defaultdict(lambda: deque(maxlen=1000))
        self._lock = threading.Lock()
    
    def increment(self, name: str, value: int = 1):
        """增加计数器"""
        with self._lock:
            self._counters[name] += value
    
    def set_gauge(self, name: str, value: float):
        """设置当前值（如活跃连接数）"""
        with self._lock:
            self._gauges[name] = value
    
    def observe(self, name: str, value: float):
        """记录一个观测值到直方图"""
        with self._lock:
            self._histograms[name].append(value)
            # 保持直方图大小合理
            if len(self._histograms[name]) > 10000:
                self._histograms[name] = self._histograms[name][-5000:]
    
    def record_rate(self, name: str):
        """记录一个速率事件"""
        with self._lock:
            self._rate_windows[name].append(time.time())
    
    def get_counter(self, name: str) -> int:
        return self._counters.get(name, 0)
    
    def get_gauge(self, name: str) -> float:
        return self._gauges.get(name, 0.0)
    
    def get_histogram_stats(self, name: str) -> Dict:
        """获取直方图的统计信息"""
        values = self._histograms.get(name, [])
        if not values:
            return {"count": 0}
        
        sorted_values = sorted(values)
        return {
            "count": len(values),
            "min": min(values),
            "max": max(values),
            "mean": statistics.mean(values),
            "median": statistics.median(values),
            "p95": sorted_values[int(len(sorted_values) * 0.95)],
            "p99": sorted_values[int(len(sorted_values) * 0.99)],
        }
    
    def get_rate(self, name: str, window_seconds: float = 60.0) -> float:
        """获取指定时间窗口内的事件速率"""
        now = time.time()
        cutoff = now - window_seconds
        timestamps = self._rate_windows.get(name, deque())
        count = sum(1 for t in timestamps if t >= cutoff)
        return count / window_seconds if window_seconds > 0 else 0
    
    def get_all_metrics(self) -> Dict:
        """获取所有指标的快照"""
        with self._lock:
            result = {
                "counters": dict(self._counters),
                "gauges": dict(self._gauges),
                "histograms": {},
                "rates": {},
            }
            for name in self._histograms:
                result["histograms"][name] = self.get_histogram_stats(name)
            for name in self._rate_windows:
                result["rates"][name] = {
                    "per_second": self.get_rate(name, 1.0),
                    "per_minute": self.get_rate(name, 60.0),
                }
            return result


class MetricsMiddleware:
    """
    指标中间件 - 自动为 Agent 的各个组件收集指标
    
    使用方式：
        metrics = MetricsCollector()
        middleware = MetricsMiddleware(metrics)
        
        # 装饰需要收集指标的函数
        @middleware.track_llm_call
        def call_llm(...):
            ...
    """
    
    def __init__(self, collector: MetricsCollector):
        self.collector = collector
    
    def track_llm_call(self, func: Callable):
        """追踪 LLM 调用的装饰器"""
        @wraps(func)
        def wrapper(*args, **kwargs):
            self.collector.increment("llm.calls.total")
            self.collector.record_rate("llm.calls.rate")
            
            start = time.time()
            try:
                result = func(*args, **kwargs)
                duration = (time.time() - start) * 1000
                self.collector.observe("llm.latency_ms", duration)
                self.collector.increment("llm.calls.success")
                return result
            except Exception as e:
                self.collector.increment("llm.calls.error")
                self.collector.increment(f"llm.errors.{type(e).__name__}")
                raise
        return wrapper
    
    def track_tool_call(self, func: Callable):
        """追踪工具调用的装饰器"""
        @wraps(func)
        def wrapper(*args, **kwargs):
            tool_name = kwargs.get("tool_name", args[0] if args else "unknown")
            self.collector.increment("tool.calls.total")
            self.collector.increment(f"tool.calls.{tool_name}")
            self.collector.record_rate("tool.calls.rate")
            
            start = time.time()
            try:
                result = func(*args, **kwargs)
                duration = (time.time() - start) * 1000
                self.collector.observe("tool.latency_ms", duration)
                self.collector.observe(f"tool.latency_ms.{tool_name}", duration)
                self.collector.increment("tool.calls.success")
                return result
            except Exception as e:
                self.collector.increment("tool.calls.error")
                self.collector.increment(f"tool.errors.{tool_name}")
                raise
        return wrapper
    
    def track_request(self, func: Callable):
        """追踪用户请求的装饰器"""
        @wraps(func)
        def wrapper(*args, **kwargs):
            self.collector.increment("requests.total")
            self.collector.record_rate("requests.rate")
            self.collector.set_gauge("requests.active", 
                                     self.collector.get_gauge("requests.active") + 1)
            
            start = time.time()
            try:
                result = func(*args, **kwargs)
                duration = (time.time() - start) * 1000
                self.collector.observe("request.latency_ms", duration)
                self.collector.increment("requests.success")
                return result
            except Exception as e:
                self.collector.increment("requests.error")
                raise
            finally:
                self.collector.set_gauge("requests.active",
                                         max(0, self.collector.get_gauge("requests.active") - 1))
        return wrapper
```

---

## 3.5 案例分析：通过可观测性定位 Agent 问题

### 3.5.1 案例：Agent 偶尔给出重复回答

假设你的 Agent 有一个奇怪的 bug：在处理某些查询时，它会连续给出两遍相同的回答。直接看代码找不到问题，但通过可观测性数据，我们可以定位原因。

**通过追踪数据发现线索：**

当我们查看出问题的请求的追踪数据时，发现了一个模式：某些请求的 LLM 调用次数异常多（5-6 次），而且每次调用的上下文中包含了重复的消息。进一步分析发现，这是因为工具调用失败后，Agent 会重试，但重试时没有正确清理之前的消息历史，导致上下文中积累了重复的信息。

**根因分析：**

LLM 看到上下文中已经有了自己的回答，于是它在生成新回答时会倾向于"继续"之前的回答，导致输出重复。这不是 LLM 的问题，而是 Agent 的消息管理逻辑有缺陷。

**修复方案：**

在工具调用重试时，清理掉上次失败的助手消息和对应的工具结果，只保留干净的消息历史。修复后，通过可观测性数据验证：平均 LLM 调用次数从 4.2 降到了 2.1，重复回答的问题也消失了。

### 3.5.2 案例：Token 消耗突然翻倍

某天你发现 Agent 的 Token 消耗突然翻倍了。通过指标监控面板，你发现是 `context_length` 指标在快速增长。进一步分析追踪数据，发现是因为新增了一个工具，这个工具返回的结果特别长，而且没有被截断。每轮对话后，这些冗长的工具结果都会累积在上下文中，导致后续每次 LLM 调用的输入 Token 数都大幅增加。

**解决方案：**

为工具结果添加长度限制和摘要机制。当工具结果超过 500 个字符时，自动截断并添加"[结果已截断]"的标记。这使得 Token 消耗回到了正常水平。

---

## 3.6 常见坑与最佳实践

### 3.6.1 常见坑

**坑 1：日志太多导致性能下降。** 记录所有细节听起来很安全，但当你在每个 LLM 调用中都记录完整的 prompt 和 response 时，日志量会爆炸。解决方案是分级别记录：默认只记录摘要信息，需要调试时再开启详细日志。

**坑 2：追踪数据中的 PII（个人身份信息）。** 用户的输入可能包含手机号、身份证号、邮箱等敏感信息。如果你把这些信息完整记录到日志中，可能违反数据保护法规。解决方案是在记录前对 PII 进行脱敏处理。

**坑 3：追踪 ID 丢失。** 在异步调用或多线程环境中，追踪 ID 可能无法正确传递，导致同一请求的不同步骤无法关联。解决方案是使用 contextvars 或线程局部变量来传递追踪上下文。

**坑 4：指标计算不准确。** 直方图的 P95 延迟如果只基于少量样本计算，可能非常不稳定。解决方案是确保有足够的样本量后再展示百分位统计。

**坑 5：监控面板遮盖了真正的错误。** 有时候你会过于关注监控面板上的数字波动，而忽略了真正需要关注的错误模式。解决方案是设置合理的告警阈值，让系统在真正需要关注时才打扰你。

### 3.6.2 最佳实践

**从简单开始。** 不要一开始就搭建复杂的可观测性系统。先用文件日志，当你发现手动翻日志太痛苦时，再升级到结构化日志和查询工具。

**追踪 ID 贯穿始终。** 从用户请求进入的那一刻起，就生成一个唯一的追踪 ID，让它贯穿 Agent 运行的所有步骤。这样你就能随时从一个追踪 ID 找到一个请求的完整生命周期。

**异步写入。** 日志写入不应该阻塞 Agent 的主流程。使用异步日志或消息队列来处理日志写入，确保可观测性不影响 Agent 的响应速度。

**定期审查。** 每周花 15 分钟审查一下 Agent 的指标趋势，关注以下问题：错误率是否在上升？平均响应时间是否在增加？Token 消耗是否在异常增长？提前发现趋势比事后救火要好得多。

---

## 3.7 练习题

**练习 1：扩展事件类型。** 在 AgentTracer 中添加以下新的事件类型：`context_overflow`（上下文溢出）、`rate_limit_hit`（触发限流）、`cache_hit`（缓存命中）。为每个事件类型添加对应的记录方法和数据字段。

**练习 2：实现告警规则。** 创建一个 AlertRule 类，支持定义告警条件（如"错误率超过 5%"），当条件满足时触发通知（可以是打印消息或发送邮件）。将 AlertRule 集成到 AgentMonitor 中。

**练习 3：实现追踪数据的查询功能。** 为 AgentTracer 添加一个 search_events 方法，支持按时间范围、事件类型、请求 ID 等条件查询历史事件。提示：可以使用列表推导式和条件过滤。

**练习 4：实现一个更真实的工具模拟器。** 替换本章中的模拟工具（search_web、calculator、get_weather），实现带有随机延迟和偶发错误的工具模拟器。这样可以更好地测试可观测性系统在异常情况下的表现。

**练习 5：设计多 Agent 协作的可观观测性方案。** 假设你有三个 Agent 协作处理一个请求（规划 Agent -> 执行 Agent -> 评审 Agent），如何设计追踪系统让三个 Agent 的行为关联在同一个追踪 ID 下？画出数据流图并写出伪代码。

**练习 6：实现 Token 预算监控。** 为 Agent 添加 Token 预算功能：每个请求最多使用 10000 个 Token。通过可观观测性系统实时监控 Token 消耗，当接近预算时发出警告，超出时强制停止。

---

## 3.8 实战任务

**任务：为一个完整的 RAG Agent 搭建可观测性体系**

本实战任务要求你为一个 RAG（检索增强生成）Agent 搭建完整的可观观测性体系。这个 Agent 的工作流程是：接收用户查询 -> 检索相关文档 -> 将文档和查询一起发送给 LLM -> 生成回答。

你需要完成以下步骤：

1. **定义追踪事件模型**：为 RAG Agent 的每个环节定义追踪事件，包括文档检索的开始/结束、检索到的文档数量和相关性分数、LLM 调用的输入输出、最终回答的质量评分等。

2. **实现追踪中间件**：创建一个可插拔的追踪中间件，可以用装饰器或上下文管理器的方式接入 RAG Agent 的每个组件，无需修改 Agent 的核心逻辑。

3. **实现性能基线**：记录 RAG Agent 在正常运行时的各项指标（检索延迟、LLM 延迟、总延迟、Token 消耗等），建立性能基线。

4. **实现实时监控面板**：创建一个终端监控面板，实时展示 RAG Agent 的运行状态。

5. **模拟异常场景**：模拟以下异常并验证可观测性系统能正确捕获和记录：检索服务超时、LLM 返回空响应、Token 超限。

6. **编写分析报告**：根据模拟运行产生的追踪数据，分析 Agent 的性能瓶颈，给出优化建议。

---

## 3.9 本章小结

本章我们深入探讨了 Agent 可观测性这个关键话题。核心观点是：对于行为不确定的 Agent 系统来说，可观观测性不是可有可无的"锦上添花"，而是确保系统可靠运行的"基础设施"。

我们首先理解了可观测性的三大支柱——日志、指标和追踪，以及它们各自的优势和局限。然后设计了从简单到复杂的三层可观测性架构，让你可以根据项目的实际需要选择合适的方案。

在代码实现部分，我们构建了一个完整的 Agent 追踪系统，包括事件模型、追踪器、请求 Span、LLM 调用 Span 和工具调用 Span。通过一个具体的 Agent 示例，我们展示了如何将追踪器无缝集成到 Agent 的运行流程中。

我们还实现了实时监控面板和指标收集器，让你能够实时观察 Agent 的运行状态。通过两个真实案例分析，我们展示了可观测性如何帮助定位看似"不可能"的 bug。

记住本章的三个核心理念：从简单开始，追踪 ID 贯穿始终，异步写入不影响性能。这些理念将帮助你在实际项目中构建实用而非过度设计的可观观测性体系。

在下一章中，我们将深入学习 LangSmith 这个专业的 LLM 可观测性平台，看看如何利用它来更高效地追踪和分析 Agent 的行为。

---

> **下一章预告：** 第 32 章将深入 LangSmith 这个专业的 LLM 可观测性平台，学习如何利用它的追踪、评估和调试功能来全面提升 Agent 的可观观测性水平。
