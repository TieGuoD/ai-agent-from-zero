# 第 41 章：Agent 监控与告警 —— 生产环境的守护者

> **本章定位：** Agent 上线后，你需要一双"眼睛"来实时观察它的运行状态，以及一个"警报器"在出问题时第一时间通知你。本章将构建一个完整的生产级 Agent 监控与告警系统。

---

## 学习目标

完成本章学习后，你将能够：

1. **设计 Agent 的关键监控指标** —— 能定义和追踪延迟、吞吐量、错误率、Token 消耗等核心指标
2. **实现 Prometheus 指标暴露** —— 能用 prometheus_client 为 Agent 添加指标端点
3. **配置 Grafana 监控仪表盘** —— 能搭建可视化面板实时展示 Agent 状态
4. **实现多级告警策略** —— 能设计从警告到紧急的多级告警规则
5. **构建告警通知系统** —— 能通过邮件、Webhook 等渠道发送告警通知

## 核心问题

1. **Agent 需要监控哪些指标？** 和传统 Web 应用相比，Agent 有哪些特有的指标需要关注？
2. **告警阈值如何设置？** 太敏感会频繁误报，太迟钝会错过真正的问题。
3. **监控系统的性能开销如何控制？** 监控本身不能成为性能瓶颈。

---

## 41.1 Agent 监控指标体系

### 41.1.1 四大指标类别

**请求指标：** QPS（每秒请求数）、延迟分布（P50/P95/P99）、错误率、成功率。这些是所有 Web 服务都需要的基础指标。

**LLM 指标：** Token 消耗速率、模型切换频率、LLM 调用延迟、LLM 错误率。这些是 Agent 特有的指标，反映了 LLM 层面的健康状况。

**业务指标：** 用户满意度（基于反馈）、任务完成率、工具调用成功率、平均对话轮数。这些指标反映了 Agent 的业务价值。

**资源指标：** CPU 使用率、内存使用率、网络 I/O、磁盘使用率。这些指标反映了基础设施的健康状况。

---

## 41.2 Prometheus 指标集成

### 41.2.1 指标收集器

```python
import os
import time
from typing import Dict, Optional
from prometheus_client import (
    Counter, Histogram, Gauge, Summary,
    generate_latest, CONTENT_TYPE_LATEST,
    start_http_server
)

class AgentMetrics:
    """
    Agent Prometheus 指标收集器
    
    暴露标准的 Prometheus 指标，用于监控 Agent 的运行状态。
    """
    
    def __init__(self, prefix: str = "agent"):
        self.prefix = prefix
        
        # 请求指标
        self.request_count = Counter(
            f"{prefix}_requests_total",
            "总请求数",
            ["status", "endpoint"],
        )
        
        self.request_duration = Histogram(
            f"{prefix}_request_duration_seconds",
            "请求处理时间",
            ["endpoint"],
            buckets=[0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0, 30.0],
        )
        
        # LLM 指标
        self.llm_calls = Counter(
            f"{prefix}_llm_calls_total",
            "LLM 调用次数",
            ["model", "status"],
        )
        
        self.llm_tokens = Counter(
            f"{prefix}_llm_tokens_total",
            "Token 消耗总数",
            ["model", "type"],  # type: input/output
        )
        
        self.llm_duration = Histogram(
            f"{prefix}_llm_duration_seconds",
            "LLM 调用延迟",
            ["model"],
            buckets=[0.5, 1.0, 2.0, 5.0, 10.0, 30.0],
        )
        
        # 工具指标
        self.tool_calls = Counter(
            f"{prefix}_tool_calls_total",
            "工具调用次数",
            ["tool_name", "status"],
        )
        
        self.tool_duration = Histogram(
            f"{prefix}_tool_duration_seconds",
            "工具调用延迟",
            ["tool_name"],
        )
        
        # Agent 状态指标
        self.active_requests = Gauge(
            f"{prefix}_active_requests",
            "当前活跃请求数",
        )
        
        self.conversation_turns = Histogram(
            f"{prefix}_conversation_turns",
            "对话轮数分布",
            buckets=[1, 2, 3, 4, 5, 6, 7, 8, 9, 10],
        )
    
    def record_request(self, status: str, endpoint: str, duration: float):
        """记录请求"""
        self.request_count.labels(status=status, endpoint=endpoint).inc()
        self.request_duration.labels(endpoint=endpoint).observe(duration)
    
    def record_llm_call(self, model: str, status: str, duration: float,
                        input_tokens: int = 0, output_tokens: int = 0):
        """记录 LLM 调用"""
        self.llm_calls.labels(model=model, status=status).inc()
        self.llm_duration.labels(model=model).observe(duration)
        if input_tokens > 0:
            self.llm_tokens.labels(model=model, type="input").inc(input_tokens)
        if output_tokens > 0:
            self.llm_tokens.labels(model=model, type="output").inc(output_tokens)
    
    def record_tool_call(self, tool_name: str, status: str, duration: float):
        """记录工具调用"""
        self.tool_calls.labels(tool_name=tool_name, status=status).inc()
        self.tool_duration.labels(tool_name=tool_name).observe(duration)
    
    def set_active_requests(self, count: int):
        """设置活跃请求数"""
        self.active_requests.set(count)
    
    def record_conversation_turns(self, turns: int):
        """记录对话轮数"""
        self.conversation_turns.observe(turns)


# ============================================================
# 使用示例
# ============================================================

def demo_metrics():
    """演示指标收集"""
    metrics = AgentMetrics()
    
    # 模拟请求
    import random
    for _ in range(100):
        status = "success" if random.random() > 0.1 else "error"
        duration = random.uniform(0.1, 5.0)
        metrics.record_request(status, "/chat", duration)
    
    # 模拟 LLM 调用
    for _ in range(50):
        model = random.choice(["gpt-4o", "gpt-4o-mini"])
        status = "success" if random.random() > 0.05 else "error"
        duration = random.uniform(0.5, 3.0)
        input_tokens = random.randint(100, 1000)
        output_tokens = random.randint(50, 500)
        metrics.record_llm_call(model, status, duration, input_tokens, output_tokens)
    
    # 模拟工具调用
    for _ in range(30):
        tool = random.choice(["search", "calculate", "database"])
        status = "success" if random.random() > 0.1 else "error"
        duration = random.uniform(0.05, 2.0)
        metrics.record_tool_call(tool, status, duration)
    
    print("指标已生成，访问 http://localhost:9090/metrics 查看 Prometheus 指标")
    
    # 启动 Prometheus 指标端点
    start_http_server(9090)
    print("Prometheus 指标端点已启动: http://localhost:9090/metrics")
    
    # 保持运行
    try:
        time.sleep(3600)
    except KeyboardInterrupt:
        pass


if __name__ == "__main__":
    demo_metrics()
```

---

## 41.3 告警系统

### 41.3.1 多级告警规则

```python
import os
import json
import time
from typing import Dict, List, Callable, Optional
from dataclasses import dataclass, field
from enum import Enum
from collections import deque

class AlertSeverity(Enum):
    """告警级别"""
    INFO = "info"
    WARNING = "warning"
    CRITICAL = "critical"
    EMERGENCY = "emergency"

@dataclass
class AlertRule:
    """告警规则"""
    name: str
    description: str
    severity: AlertSeverity
    metric_name: str
    threshold: float
    comparison: str  # "gt", "lt", "eq"
    duration_seconds: int = 60  # 持续时间
    cooldown_seconds: int = 300  # 冷却时间
    enabled: bool = True

@dataclass
class Alert:
    """告警实例"""
    rule_name: str
    severity: AlertSeverity
    message: str
    metric_value: float
    threshold: float
    timestamp: float = field(default_factory=time.time)
    resolved: bool = False

class AlertManager:
    """
    Agent 告警管理器
    
    监控指标并在异常时发送告警通知。
    """
    
    def __init__(self):
        self.rules: List[AlertRule] = []
        self.active_alerts: Dict[str, Alert] = {}
        self.alert_history: List[Alert] = []
        self.notification_callbacks: List[Callable] = []
        self._cooldown_tracker: Dict[str, float] = {}
        
        # 创建默认规则
        self._create_default_rules()
    
    def _create_default_rules(self):
        """创建默认告警规则"""
        self.rules = [
            AlertRule(
                name="high_error_rate",
                description="错误率过高",
                severity=AlertSeverity.CRITICAL,
                metric_name="error_rate",
                threshold=0.1,  # 10%
                comparison="gt",
                duration_seconds=60,
            ),
            AlertRule(
                name="high_latency_p95",
                description="P95 延迟过高",
                severity=AlertSeverity.WARNING,
                metric_name="p95_latency",
                threshold=5.0,  # 5 秒
                comparison="gt",
                duration_seconds=120,
            ),
            AlertRule(
                name="high_token_usage",
                description="Token 消耗异常",
                severity=AlertSeverity.WARNING,
                metric_name="tokens_per_minute",
                threshold=100000,
                comparison="gt",
                duration_seconds=300,
            ),
            AlertRule(
                name="low_qps",
                description="请求量异常低",
                severity=AlertSeverity.INFO,
                metric_name="qps",
                threshold=0.1,
                comparison="lt",
                duration_seconds=300,
            ),
            AlertRule(
                name="service_down",
                description="服务不可用",
                severity=AlertSeverity.EMERGENCY,
                metric_name="health_status",
                threshold=0,
                comparison="eq",
                duration_seconds=30,
                cooldown_seconds=60,
            ),
        ]
    
    def register_notification(self, callback: Callable):
        """注册告警通知回调"""
        self.notification_callbacks.append(callback)
    
    def evaluate(self, metrics: Dict[str, float]) -> List[Alert]:
        """
        评估所有告警规则
        
        返回新触发的告警列表。
        """
        new_alerts = []
        
        for rule in self.rules:
            if not rule.enabled:
                continue
            
            metric_value = metrics.get(rule.metric_name)
            if metric_value is None:
                continue
            
            # 检查是否满足触发条件
            triggered = False
            if rule.comparison == "gt" and metric_value > rule.threshold:
                triggered = True
            elif rule.comparison == "lt" and metric_value < rule.threshold:
                triggered = True
            elif rule.comparison == "eq" and metric_value == rule.threshold:
                triggered = True
            
            if triggered:
                # 检查冷却时间
                last_alert_time = self._cooldown_tracker.get(rule.name, 0)
                if time.time() - last_alert_time < rule.cooldown_seconds:
                    continue
                
                # 创建告警
                alert = Alert(
                    rule_name=rule.name,
                    severity=rule.severity,
                    message=f"{rule.description}: {rule.metric_name}={metric_value} (阈值={rule.threshold})",
                    metric_value=metric_value,
                    threshold=rule.threshold,
                )
                
                new_alerts.append(alert)
                self.active_alerts[rule.name] = alert
                self.alert_history.append(alert)
                self._cooldown_tracker[rule.name] = time.time()
                
                # 发送通知
                self._send_notification(alert)
            
            else:
                # 如果之前有活跃告警，现在恢复了
                if rule.name in self.active_alerts:
                    resolved_alert = self.active_alerts.pop(rule.name)
                    resolved_alert.resolved = True
                    self._send_resolution(resolved_alert)
        
        return new_alerts
    
    def _send_notification(self, alert: Alert):
        """发送告警通知"""
        for callback in self.notification_callbacks:
            try:
                callback(alert)
            except Exception as e:
                print(f"告警通知发送失败: {e}")
    
    def _send_resolution(self, alert: Alert):
        """发送告警恢复通知"""
        print(f"[RESOLVED] {alert.rule_name}: {alert.message}")
    
    def get_active_alerts(self) -> List[Alert]:
        """获取所有活跃告警"""
        return list(self.active_alerts.values())
    
    def get_alert_stats(self) -> Dict:
        """获取告警统计"""
        stats = {
            "total_rules": len(self.rules),
            "active_alerts": len(self.active_alerts),
            "total_history": len(self.alert_history),
            "by_severity": {},
        }
        
        for alert in self.alert_history:
            severity = alert.severity.value
            stats["by_severity"][severity] = stats["by_severity"].get(severity, 0) + 1
        
        return stats


# ============================================================
# 通知渠道实现
# ============================================================

class WebhookNotifier:
    """Webhook 通知器"""
    
    def __init__(self, webhook_url: str):
        self.webhook_url = webhook_url
    
    def __call__(self, alert: Alert):
        """发送 Webhook 通知"""
        payload = {
            "text": f"[{alert.severity.value.upper()}] {alert.message}",
            "alert": {
                "rule": alert.rule_name,
                "severity": alert.severity.value,
                "message": alert.message,
                "metric_value": alert.metric_value,
                "threshold": alert.threshold,
                "timestamp": alert.timestamp,
            }
        }
        print(f"[Webhook] {json.dumps(payload, ensure_ascii=False)}")
        # 实际实现中应该用 requests.post 发送


class ConsoleNotifier:
    """控制台通知器"""
    
    def __call__(self, alert: Alert):
        """打印告警到控制台"""
        colors = {
            AlertSeverity.INFO: "\033[34m",      # 蓝色
            AlertSeverity.WARNING: "\033[33m",   # 黄色
            AlertSeverity.CRITICAL: "\033[31m",   # 红色
            AlertSeverity.EMERGENCY: "\033[35m",  # 紫色
        }
        reset = "\033[0m"
        color = colors.get(alert.severity, "")
        
        print(f"\n{color}[ALERT] {alert.severity.value.upper()}: {alert.message}{reset}")


# ============================================================
# 使用示例
# ============================================================

def demo_alerting():
    """演示告警系统"""
    # 创建告警管理器
    manager = AlertManager()
    manager.register_notification(ConsoleNotifier())
    manager.register_notification(WebhookNotifier("https://hooks.example.com/alert"))
    
    print("告警系统测试")
    print("=" * 60)
    
    # 模拟正常指标
    normal_metrics = {
        "error_rate": 0.02,
        "p95_latency": 2.0,
        "tokens_per_minute": 50000,
        "qps": 10,
        "health_status": 1,
    }
    
    alerts = manager.evaluate(normal_metrics)
    print(f"\n正常状态下: {len(alerts)} 个告警")
    
    # 模拟异常指标
    abnormal_metrics = {
        "error_rate": 0.15,  # 超过 10% 阈值
        "p95_latency": 8.0,  # 超过 5 秒阈值
        "tokens_per_minute": 120000,  # 超过 100k 阈值
        "qps": 10,
        "health_status": 1,
    }
    
    alerts = manager.evaluate(abnormal_metrics)
    print(f"\n异常状态下: {len(alerts)} 个告警")
    for alert in alerts:
        print(f"  [{alert.severity.value}] {alert.message}")
    
    # 统计信息
    stats = manager.get_alert_stats()
    print(f"\n告警统计: {json.dumps(stats, ensure_ascii=False)}")


if __name__ == "__main__":
    demo_alerting()
```

---

## 41.4 Grafana 仪表盘配置

### 41.4.1 仪表盘 JSON 模板

```json
{
  "dashboard": {
    "title": "Agent 监控仪表盘",
    "panels": [
      {
        "title": "请求速率 (QPS)",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(agent_requests_total[5m])",
            "legendFormat": "{{endpoint}} - {{status}}"
          }
        ]
      },
      {
        "title": "P95 延迟",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(agent_request_duration_seconds_bucket[5m]))",
            "legendFormat": "P95"
          }
        ]
      },
      {
        "title": "错误率",
        "type": "singlestat",
        "targets": [
          {
            "expr": "rate(agent_requests_total{status='error'}[5m]) / rate(agent_requests_total[5m]) * 100",
            "legendFormat": "{{.Name}}"
          }
        ],
        "thresholds": [
          {"value": 5, "color": "yellow"},
          {"value": 10, "color": "red"}
        ]
      },
      {
        "title": "Token 消耗趋势",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(agent_llm_tokens_total[5m])",
            "legendFormat": "{{model}} - {{type}}"
          }
        ]
      }
    ]
  }
}
```

---

## 41.5 案例分析

### 41.5.1 案例：凌晨 3 点的告警

**事件：** 凌晨 3 点，监控系统检测到 Agent 的错误率突然从 2% 飙升到 25%，触发了 CRITICAL 级别告警。运维人员收到通知后立即排查。

**排查过程：** 通过 Grafana 仪表盘发现，错误主要来自 LLM API 调用超时。进一步检查发现，OpenAI 的 API 在凌晨有维护窗口，部分请求会超时。

**解决方案：** 实现了 LLM API 的故障转移机制：当主 API（OpenAI）超时时，自动切换到备用 API（Anthropic）。这使得 Agent 在 LLM 提供商故障时仍然能正常工作。

**关键教训：** 监控系统的价值不仅在于发现问题，更在于缩短从发现问题到解决问题的时间（MTTR）。

---

## 41.6 高级监控功能

### 41.6.1 分布式追踪集成

在微服务架构中，需要追踪请求在多个服务之间的流转。

```python
import time
import uuid
import json
from typing import Dict, List, Optional, Any
from dataclasses import dataclass, field
from datetime import datetime
import threading

@dataclass
class TraceSpan:
    """追踪跨度"""
    trace_id: str
    span_id: str
    parent_span_id: Optional[str]
    service_name: str
    operation_name: str
    start_time: float
    end_time: Optional[float] = None
    duration_ms: Optional[float] = None
    status: str = "in_progress"
    tags: Dict[str, Any] = field(default_factory=dict)
    logs: List[Dict] = field(default_factory=list)

class DistributedTracer:
    """
    分布式追踪器
    
    提供完整的分布式追踪能力：
    - 生成和传播追踪上下文
    - 记录跨度信息
    - 支持多服务关联
    """
    
    _local = threading.local()
    
    def __init__(self, service_name: str):
        self.service_name = service_name
    
    @classmethod
    def generate_trace_id(cls) -> str:
        """生成追踪 ID"""
        return str(uuid.uuid4())[:16]
    
    @classmethod
    def generate_span_id(cls) -> str:
        """生成跨度 ID"""
        return str(uuid.uuid4())[:8]
    
    def start_span(self, operation_name: str, 
                  parent_span_id: Optional[str] = None) -> TraceSpan:
        """
        开始新的追踪跨度
        
        Args:
            operation_name: 操作名称
            parent_span_id: 父跨度 ID
            
        Returns:
            TraceSpan 对象
        """
        # 获取或创建追踪上下文
        if not hasattr(self._local, 'trace_id'):
            self._local.trace_id = self.generate_trace_id()
        
        span = TraceSpan(
            trace_id=self._local.trace_id,
            span_id=self.generate_span_id(),
            parent_span_id=parent_span_id,
            service_name=self.service_name,
            operation_name=operation_name,
            start_time=time.time(),
        )
        
        return span
    
    def finish_span(self, span: TraceSpan, status: str = "success",
                   error: Optional[str] = None):
        """
        完成追踪跨度
        
        Args:
            span: TraceSpan 对象
            status: 状态 (success/error)
            error: 错误信息
        """
        span.end_time = time.time()
        span.duration_ms = (span.end_time - span.start_time) * 1000
        span.status = status
        
        if error:
            span.tags["error"] = True
            span.tags["error.message"] = error
            self.add_log(span, "error", error)
        
        # 这里应该发送到追踪收集器
        self._send_to_collector(span)
    
    def add_tag(self, span: TraceSpan, key: str, value: Any):
        """添加标签"""
        span.tags[key] = value
    
    def add_log(self, span: TraceSpan, level: str, message: str):
        """添加日志"""
        span.logs.append({
            "timestamp": time.time(),
            "level": level,
            "message": message,
        })
    
    def inject_headers(self, headers: Dict) -> Dict:
        """
        将追踪上下文注入到 HTTP 头
        
        用于在微服务之间传播追踪上下文。
        """
        headers["X-Trace-ID"] = getattr(self._local, 'trace_id', '')
        return headers
    
    def extract_headers(self, headers: Dict):
        """
        从 HTTP 头提取追踪上下文
        """
        trace_id = headers.get("X-Trace-ID", "")
        if trace_id:
            self._local.trace_id = trace_id
    
    def _send_to_collector(self, span: TraceSpan):
        """发送到追踪收集器"""
        # 实际实现应该异步发送到 Jaeger/Zipkin 等
        trace_data = {
            "trace_id": span.trace_id,
            "span_id": span.span_id,
            "parent_span_id": span.parent_span_id,
            "service": span.service_name,
            "operation": span.operation_name,
            "duration_ms": span.duration_ms,
            "status": span.status,
            "tags": span.tags,
            "timestamp": datetime.fromtimestamp(span.start_time).isoformat(),
        }
        
        # 打印追踪信息（生产环境应该发送到收集器）
        print(f"[TRACE] {span.service_name}/{span.operation_name}: "
              f"{span.duration_ms:.2f}ms ({span.status})")


# ============================================================
# 使用示例
# ============================================================

def demo_distributed_tracing():
    """演示分布式追踪"""
    # 创建追踪器
    tracer = DistributedTracer("agent-service")
    
    print("分布式追踪演示")
    print("=" * 60)
    
    # 开始主追踪
    main_span = tracer.start_span("process_request")
    tracer.add_tag(main_span, "user_id", "user_123")
    
    # 模拟 LLM 调用
    llm_span = tracer.start_span("llm_call", parent_span_id=main_span.span_id)
    tracer.add_tag(llm_span, "model", "gpt-4o")
    tracer.add_tag(llm_span, "input_tokens", 150)
    
    time.sleep(0.1)  # 模拟延迟
    
    tracer.finish_span(llm_span, status="success")
    tracer.add_tag(llm_span, "output_tokens", 200)
    
    # 模拟工具调用
    tool_span = tracer.start_span("tool_call", parent_span_id=main_span.span_id)
    tracer.add_tag(tool_span, "tool_name", "search")
    
    time.sleep(0.05)  # 模拟延迟
    
    tracer.finish_span(tool_span, status="success")
    
    # 完成主追踪
    tracer.finish_span(main_span, status="success")
    
    # 演示上下文传播
    print("\n模拟服务间调用:")
    headers = {}
    tracer.inject_headers(headers)
    print(f"  传递的头部: {headers}")
    
    # 模拟另一个服务接收
    remote_tracer = DistributedTracer("tool-service")
    remote_tracer.extract_headers(headers)
    remote_span = remote_tracer.start_span("execute_tool")
    time.sleep(0.02)
    remote_tracer.finish_span(remote_span)


if __name__ == "__main__":
    demo_distributed_tracing()
```

### 41.6.2 自定义业务指标

```python
from typing import Dict, List, Any
from dataclasses import dataclass
import time

@dataclass
class BusinessMetric:
    """业务指标"""
    name: str
    value: float
    timestamp: float
    tags: Dict[str, str] = None

class BusinessMetricsCollector:
    """
    业务指标收集器
    
    收集和分析 Agent 的业务指标。
    """
    
    def __init__(self):
        self.metrics: List[BusinessMetric] = []
        self._counters: Dict[str, int] = {}
        self._gauges: Dict[str, float] = {}
        self._histograms: Dict[str, List[float]] = {}
    
    def increment_counter(self, name: str, value: int = 1, 
                         tags: Dict[str, str] = None):
        """增加计数器"""
        key = self._make_key(name, tags)
        self._counters[key] = self._counters.get(key, 0) + value
        
        self.metrics.append(BusinessMetric(
            name=name,
            value=self._counters[key],
            timestamp=time.time(),
            tags=tags,
        ))
    
    def set_gauge(self, name: str, value: float, 
                  tags: Dict[str, str] = None):
        """设置仪表盘值"""
        key = self._make_key(name, tags)
        self._gauges[key] = value
        
        self.metrics.append(BusinessMetric(
            name=name,
            value=value,
            timestamp=time.time(),
            tags=tags,
        ))
    
    def observe_histogram(self, name: str, value: float,
                         tags: Dict[str, str] = None):
        """观察直方图值"""
        key = self._make_key(name, tags)
        if key not in self._histograms:
            self._histograms[key] = []
        self._histograms[key].append(value)
        
        self.metrics.append(BusinessMetric(
            name=name,
            value=value,
            timestamp=time.time(),
            tags=tags,
        ))
    
    def get_counter(self, name: str, tags: Dict[str, str] = None) -> int:
        """获取计数器值"""
        key = self._make_key(name, tags)
        return self._counters.get(key, 0)
    
    def get_gauge(self, name: str, tags: Dict[str, str] = None) -> float:
        """获取仪表盘值"""
        key = self._make_key(name, tags)
        return self._gauges.get(key, 0.0)
    
    def get_histogram_stats(self, name: str, 
                           tags: Dict[str, str] = None) -> Dict:
        """获取直方图统计"""
        key = self._make_key(name, tags)
        values = self._histograms.get(key, [])
        
        if not values:
            return {"count": 0}
        
        values_sorted = sorted(values)
        n = len(values_sorted)
        
        return {
            "count": n,
            "min": values_sorted[0],
            "max": values_sorted[-1],
            "mean": sum(values) / n,
            "p50": values_sorted[int(n * 0.5)],
            "p95": values_sorted[int(n * 0.95)],
            "p99": values_sorted[int(n * 0.99)],
        }
    
    def _make_key(self, name: str, tags: Dict[str, str] = None) -> str:
        """生成唯一键"""
        if tags:
            tag_str = ",".join(f"{k}={v}" for k, v in sorted(tags.items()))
            return f"{name}{{{tag_str}}}"
        return name
    
    def record_user_satisfaction(self, rating: int, 
                                session_id: str = None):
        """记录用户满意度"""
        self.observe_histogram(
            "user_satisfaction_rating",
            rating,
            tags={"session": session_id} if session_id else None,
        )
        
        if rating >= 4:
            self.increment_counter("user_satisfied", tags={"level": "high"})
        elif rating >= 3:
            self.increment_counter("user_satisfied", tags={"level": "medium"})
        else:
            self.increment_counter("user_satisfied", tags={"level": "low"})
    
    def record_task_completion(self, task_type: str, success: bool,
                              duration_ms: float):
        """记录任务完成情况"""
        self.increment_counter(
            "task_completion",
            tags={"type": task_type, "success": str(success).lower()},
        )
        
        self.observe_histogram(
            "task_duration_ms",
            duration_ms,
            tags={"type": task_type},
        )
    
    def record_cost(self, model: str, input_tokens: int, output_tokens: int,
                   cost_usd: float):
        """记录成本"""
        self.increment_counter(
            "llm_cost_usd",
            tags={"model": model},
        )
        
        self.observe_histogram(
            "tokens_per_request",
            input_tokens + output_tokens,
            tags={"model": model, "type": "total"},
        )


# ============================================================
# 使用示例
# ============================================================

def demo_business_metrics():
    """演示业务指标"""
    collector = BusinessMetricsCollector()
    
    print("业务指标收集演示")
    print("=" * 60)
    
    import random
    
    # 模拟收集指标
    for i in range(100):
        # 用户满意度
        rating = random.choice([1, 2, 3, 4, 5])
        collector.record_user_satisfaction(rating, f"session_{i % 10}")
        
        # 任务完成
        task_type = random.choice(["qa", "search", "generation"])
        success = random.random() > 0.1
        duration = random.uniform(100, 3000)
        collector.record_task_completion(task_type, success, duration)
        
        # 成本
        model = random.choice(["gpt-4o", "gpt-4o-mini"])
        input_tokens = random.randint(100, 1000)
        output_tokens = random.randint(50, 500)
        cost = (input_tokens * 0.0025 + output_tokens * 0.01) / 1000
        collector.record_cost(model, input_tokens, output_tokens, cost)
    
    # 打印统计
    print("\n满意度统计:")
    satisfaction_stats = collector.get_histogram_stats("user_satisfaction_rating")
    print(f"  样本数: {satisfaction_stats['count']}")
    print(f"  平均分: {satisfaction_stats['mean']:.2f}")
    print(f"  P50: {satisfaction_stats['p50']}")
    print(f"  P95: {satisfaction_stats['p95']}")
    
    print("\n任务完成统计:")
    qa_total = collector.get_counter("task_completion", {"type": "qa", "success": "true"})
    qa_failed = collector.get_counter("task_completion", {"type": "qa", "success": "false"})
    print(f"  QA 任务: 成功 {qa_total}, 失败 {qa_failed}")
    
    print("\n成本统计:")
    cost_stats = collector.get_histogram_stats("tokens_per_request", {"model": "gpt-4o", "type": "total"})
    print(f"  每请求 Token 数 - 平均: {cost_stats.get('mean', 0):.0f}, P95: {cost_stats.get('p95', 0)}")


if __name__ == "__main__":
    demo_business_metrics()
```

### 41.6.3 SLA 计算与报告

```python
from typing import Dict, List
from datetime import datetime, timedelta

class SLACalculator:
    """
    SLA 计算器
    
    计算和报告 Agent 的服务级别协议达成情况。
    """
    
    def __init__(self, target_availability: float = 99.9,
                 target_p95_latency_ms: float = 3000,
                 target_error_rate: float = 0.01):
        self.target_availability = target_availability
        self.target_p95_latency_ms = target_p95_latency_ms
        self.target_error_rate = target_error_rate
        
        self.incidents: List[Dict] = []
    
    def record_incident(self, start_time: datetime, end_time: datetime,
                       impact: str = "partial"):
        """记录故障事件"""
        self.incidents.append({
            "start": start_time,
            "end": end_time,
            "duration_minutes": (end_time - start_time).total_seconds() / 60,
            "impact": impact,
        })
    
    def calculate_availability(self, period_start: datetime, 
                              period_end: datetime) -> Dict:
        """
        计算可用性
        
        Args:
            period_start: 统计周期开始
            period_end: 统计周期结束
            
        Returns:
            可用性统计
        """
        total_minutes = (period_end - period_start).total_seconds() / 60
        
        # 计算故障时间
        downtime_minutes = 0
        for incident in self.incidents:
            # 只统计在统计周期内的故障
            incident_start = max(incident["start"], period_start)
            incident_end = min(incident["end"], period_end)
            
            if incident_start < incident_end:
                if incident["impact"] == "full":
                    downtime_minutes += (incident_end - incident_start).total_seconds() / 60
                elif incident["impact"] == "partial":
                    downtime_minutes += (incident_end - incident_start).total_seconds() / 120  # 部分故障按50%计算
        
        availability = (1 - downtime_minutes / total_minutes) * 100 if total_minutes > 0 else 100
        
        return {
            "period": f"{period_start.date()} to {period_end.date()}",
            "total_minutes": total_minutes,
            "downtime_minutes": downtime_minutes,
            "availability_percent": availability,
            "target": self.target_availability,
            "met_target": availability >= self.target_availability,
        }
    
    def calculate_performance_sla(self, metrics: Dict) -> Dict:
        """
        计算性能 SLA
        
        Args:
            metrics: 性能指标
            
        Returns:
            性能 SLA 报告
        """
        p95_latency = metrics.get("p95_latency_ms", 0)
        error_rate = metrics.get("error_rate", 0)
        
        latency_met = p95_latency <= self.target_p95_latency_ms
        error_rate_met = error_rate <= self.target_error_rate
        
        return {
            "latency": {
                "current": p95_latency,
                "target": self.target_p95_latency_ms,
                "met": latency_met,
            },
            "error_rate": {
                "current": error_rate,
                "target": self.target_error_rate,
                "met": error_rate_met,
            },
            "overall_met": latency_met and error_rate_met,
        }
    
    def generate_sla_report(self, period_start: datetime, 
                           period_end: datetime,
                           performance_metrics: Dict) -> Dict:
        """
        生成完整的 SLA 报告
        """
        availability = self.calculate_availability(period_start, period_end)
        performance = self.calculate_performance_sla(performance_metrics)
        
        return {
            "report_period": {
                "start": period_start.isoformat(),
                "end": period_end.isoformat(),
            },
            "availability": availability,
            "performance": performance,
            "overall_status": "met" if (availability["met_target"] and 
                                       performance["overall_met"]) else "violated",
            "incidents_count": len(self.incidents),
        }


# ============================================================
# 使用示例
# ============================================================

def demo_sla_calculation():
    """演示 SLA 计算"""
    calculator = SLACalculator(
        target_availability=99.9,
        target_p95_latency_ms=3000,
        target_error_rate=0.01,
    )
    
    # 模拟故障事件
    from datetime import datetime
    
    calculator.record_incident(
        start_time=datetime(2024, 1, 15, 3, 0),
        end_time=datetime(2024, 1, 15, 3, 15),
        impact="full",
    )
    
    calculator.record_incident(
        start_time=datetime(2024, 1, 20, 14, 30),
        end_time=datetime(2024, 1, 20, 14, 45),
        impact="partial",
    )
    
    # 计算 SLA
    period_start = datetime(2024, 1, 1)
    period_end = datetime(2024, 1, 31)
    
    performance_metrics = {
        "p95_latency_ms": 2500,
        "error_rate": 0.005,
    }
    
    report = calculator.generate_sla_report(period_start, period_end, performance_metrics)
    
    print("SLA 报告")
    print("=" * 60)
    
    print(f"\n统计周期: {report['report_period']['start']} 到 {report['report_period']['end']}")
    print(f"总体状态: {report['overall_status'].upper()}")
    
    print(f"\n可用性:")
    print(f"  当前: {report['availability']['availability_percent']:.3f}%")
    print(f"  目标: {report['availability']['target']}%")
    print(f"  达标: {'是' if report['availability']['met_target'] else '否'}")
    print(f"  故障时间: {report['availability']['downtime_minutes']:.1f} 分钟")
    
    print(f"\n性能:")
    print(f"  P95 延迟: {report['performance']['latency']['current']:.0f}ms (目标: {report['performance']['latency']['target']:.0f}ms)")
    print(f"  错误率: {report['performance']['error_rate']['current']:.2%} (目标: {report['performance']['error_rate']['target']:.2%})")


if __name__ == "__main__":
    demo_sla_calculation()
```

## 41.7 常见坑与最佳实践

**坑 1：告警风暴。** 一个问题可能同时触发多个告警，导致通知系统被打爆。实现告警聚合和升级机制——相关告警合并为一条通知。

**坑 2：监控系统自身故障。** 监控系统挂了你就成了"瞎子"。监控系统本身也需要高可用部署和健康检查。

**坑 3：指标太多导致信息过载。** 不是所有指标都需要实时关注。按重要性分级：核心指标实时监控，次要指标每日审查，详细指标按需查询。

**坑 4：忽略延迟分布。** 只看平均延迟可能掩盖长尾问题。使用 P95、P99 等百分位指标来衡量用户体验。

**坑 5：监控数据没有利用。** 收集了大量监控数据但没有分析和利用。定期审查监控数据，发现趋势和改进机会。

**最佳实践 1：建立 On-Call 轮值制度。** 确保每个时间段都有人负责响应告警。告警通知应该包含足够的上下文信息，让值班人员能快速判断问题的严重性。

**最佳实践 2：监控驱动开发。** 在开发新功能时，同步设计监控指标。确保新功能上线后能被有效监控。

**最佳实践 3：建立 SLA 监控。** 将 SLA 指标纳入监控体系，实时追踪是否达标。

**最佳实践 4：定期回顾监控有效性。** 定期检查告警规则是否仍然有效，是否有新的问题需要监控。

---

## 41.7 练习题

**练习 1：完善指标收集。** 为 Agent 添加以下指标：对话满意度（基于用户反馈）、模型切换次数、缓存命中率。

**练习 2：实现告警升级。** 当一个 CRITICAL 告警在 15 分钟内未被处理时，自动升级为 EMERGENCY 并通知更高级别的人员。

**练习 3：设计告警仪表盘。** 用 Grafana 创建一个 Agent 监控仪表盘，包含至少 6 个面板。

**练习 4：实现告警历史查询。** 编写一个 API 端点，支持按时间范围、严重级别等条件查询历史告警。

**练习 5：实现 SLA 计算。** 根据监控数据计算 Agent 的 SLA（服务级别协议）达标情况：可用性是否达到 99.9%？P95 延迟是否在 3 秒以内？

**练习 6：设计混沌工程测试。** 编写脚本模拟 LLM API 超时、网络中断等故障场景，验证监控和告警系统是否能正确触发。

---

## 41.8 本章小结

本章构建了完整的 Agent 生产监控与告警体系。从 Prometheus 指标收集到 Grafana 可视化，从多级告警规则到通知渠道，我们覆盖了生产环境守护的关键环节。

核心理念是：监控不是被动地等待问题发生，而是主动地发现问题的趋势。通过持续监控关键指标，你可以在问题影响用户之前就发现和处理它们。

---

> **下一章预告：** 第 42 章将深入 Agent 的版本管理与 A/B 测试，学习如何安全地迭代和改进 Agent。
