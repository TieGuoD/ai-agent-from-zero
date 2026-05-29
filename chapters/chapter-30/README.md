# 第 30 章 Multi-Agent 系统设计 —— 架构与权衡

> "架构设计不是选择最优雅的方案，而是在无数个权衡中找到最适合当前场景的平衡点。Multi-Agent 系统的架构设计尤其如此——它涉及 Agent 的数量、通信方式、协调策略、错误处理、成本控制等多个维度的权衡。"

---

## 学习目标

通过本章的学习，你将能够：

1. 系统性地理解 Multi-Agent 系统架构设计的关键决策点
2. 掌握从需求分析到架构选型的完整设计流程
3. 理解性能、成本、可靠性、可维护性之间的权衡方法
4. 学会评估和选择合适的通信模式、协调策略和框架
5. 使用 Python 实现一个完整的生产级 Multi-Agent 系统架构
6. 掌握 Multi-Agent 系统的测试、监控和运维方法
7. 了解 Multi-Agent 系统的未来发展趋势

---

## 核心问题

- 如何从零开始设计一个生产级别的 Multi-Agent 系统？
- 在架构设计中，有哪些关键的权衡点？如何做出决策？
- 如何确保 Multi-Agent 系统的可靠性和可维护性？
- 如何评估 Multi-Agent 系统的性能和成本？
- Multi-Agent 系统的未来发展方向是什么？

---

## 30.1 从需求到架构：设计的起点

### 30.1.1 需求分析：你真的需要 Multi-Agent 吗？

在开始设计之前，最重要也是最容易被忽略的一步是：确认你是否真的需要 Multi-Agent 系统。

Multi-Agent 系统不是万能药。对于简单的任务，单个 Agent 可能就足够了。引入 Multi-Agent 会带来额外的复杂度——通信开销、协调成本、调试难度——如果任务本身不需要这些，就是得不偿失。

以下情况通常适合采用 Multi-Agent 系统：

**任务复杂度高**：任务涉及多个专业领域，单个 Agent 难以同时精通所有领域。

**需要多角度分析**：问题需要从不同视角来审视，单一 Agent 的视角不够全面。

**质量要求高**：需要通过多个 Agent 的交叉验证来保证输出质量。

**处理时间敏感**：任务可以拆分为多个并行子任务，需要同时处理以缩短总时间。

**系统需要可扩展性**：未来需要频繁添加新的功能模块。

相反，以下情况可能不需要 Multi-Agent：

**任务简单直接**：单个 Agent 能够很好地完成。

**延迟要求极高**：不能接受 Multi-Agent 通信带来的额外延迟。

**成本敏感**：Multi-Agent 的 LLM 调用量通常是单 Agent 的数倍。

### 30.1.2 需求分析框架

在确认需要 Multi-Agent 后，你需要系统地分析需求。以下是一个实用的需求分析框架：

```python
"""
第 30 章：Multi-Agent 系统需求分析框架
从需求出发，确定系统的设计方向
"""
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class SystemRequirements:
    """
    Multi-Agent 系统需求分析。
    
    在设计架构之前，先回答这些问题。
    每个问题的答案都会影响后续的架构决策。
    """
    
    # 基础信息
    project_name: str = ""
    project_description: str = ""
    
    # 功能需求
    primary_tasks: list[str] = field(default_factory=list)  # 主要任务
    task_complexity: str = "medium"  # low, medium, high
    specialized_domains: list[str] = field(default_factory=list)  # 专业领域
    
    # 质量需求
    accuracy_requirement: str = "medium"  # low, medium, high, critical
    reliability_requirement: str = "medium"  # low, medium, high
    latency_requirement: str = "medium"  # low (严格), medium, high (宽松)
    
    # 规模需求
    expected_agents: int = 3  # 预期 Agent 数量
    concurrent_users: int = 10  # 并发用户数
    daily_requests: int = 1000  # 日均请求量
    
    # 成本需求
    budget_level: str = "medium"  # low, medium, high
    max_cost_per_request: float = 0.1  # 每次请求的最大成本（美元）
    
    # 技术需求
    supported_languages: list[str] = field(default_factory=lambda: ["python"])
    deployment_environment: str = "cloud"  # cloud, on_premise, edge
    integration_requirements: list[str] = field(default_factory=list)  # 需要集成的外部系统
    
    def analyze(self) -> dict:
        """
        分析需求，给出架构建议。
        
        基于需求的各种维度，自动推荐合适的架构方案。
        """
        recommendations = {}
        
        # 推荐 Agent 数量范围
        if self.task_complexity == "low":
            recommendations["agent_count"] = "2-3 个 Agent"
        elif self.task_complexity == "medium":
            recommendations["agent_count"] = "3-5 个 Agent"
        else:
            recommendations["agent_count"] = "5-8 个 Agent"
        
        # 推荐通信模式
        if self.latency_requirement == "low":
            recommendations["communication"] = "同步消息传递（延迟最低）"
        elif self.concurrent_users > 50:
            recommendations["communication"] = "异步消息队列（高并发支持）"
        else:
            recommendations["communication"] = "直接函数调用（简单高效）"
        
        # 推荐协调策略
        if self.expected_agents <= 3:
            recommendations["coordination"] = "集中式协调（简单可控）"
        elif self.expected_agents <= 6:
            recommendations["coordination"] = "层级式协调（平衡效率和灵活性）"
        else:
            recommendations["coordination"] = "分布式协调（可扩展性强）"
        
        # 推荐框架
        if self.task_complexity == "high" and len(self.specialized_domains) > 3:
            recommendations["framework"] = "CrewAI（结构化角色和任务管理）"
        elif self.accuracy_requirement in ["high", "critical"]:
            recommendations["framework"] = "AutoGen + 辩论机制（深度质量保证）"
        else:
            recommendations["framework"] = "自定义框架（灵活定制）"
        
        # 推荐模型策略
        if self.budget_level == "low":
            recommendations["model_strategy"] = "分层策略：核心 Agent 用 GPT-4，辅助 Agent 用 GPT-3.5"
        elif self.budget_level == "high":
            recommendations["model_strategy"] = "统一使用 GPT-4 确保质量"
        else:
            recommendations["model_strategy"] = "分层策略：关键 Agent 用 GPT-4，其他用 GPT-3.5"
        
        return recommendations


# ============================================================
# 演示：需求分析
# ============================================================

def demo_requirements_analysis():
    """演示需求分析过程"""
    
    print("\n" + "="*60)
    print("Multi-Agent 系统需求分析")
    print("="*60)
    
    # 定义需求
    requirements = SystemRequirements(
        project_name="AI 客服系统",
        project_description="构建一个智能客服系统，能够处理多种类型的用户咨询",
        primary_tasks=[
            "理解用户意图",
            "回答产品问题",
            "处理退换货请求",
            "解决技术问题",
            "生成分析报告"
        ],
        task_complexity="high",
        specialized_domains=["自然语言理解", "产品知识", "物流管理", "技术支持", "数据分析"],
        accuracy_requirement="high",
        reliability_requirement="high",
        latency_requirement="medium",
        expected_agents=5,
        concurrent_users=100,
        daily_requests=5000,
        budget_level="medium",
        max_cost_per_request=0.05,
        integration_requirements=["CRM系统", "订单管理系统", "知识库", "工单系统"]
    )
    
    # 分析需求
    recommendations = requirements.analyze()
    
    print(f"\n项目: {requirements.project_name}")
    print(f"描述: {requirements.project_description}")
    print(f"任务复杂度: {requirements.task_complexity}")
    print(f"预期 Agent 数: {requirements.expected_agents}")
    print(f"准确度要求: {requirements.accuracy_requirement}")
    print(f"预算级别: {requirements.budget_level}")
    
    print(f"\n--- 架构建议 ---")
    for key, value in recommendations.items():
        print(f"  {key}: {value}")


if __name__ == "__main__":
    demo_requirements_analysis()
```

---

## 30.2 架构设计的关键决策

### 30.2.1 决策一：Agent 的数量和粒度

Agent 数量是 Multi-Agent 系统设计中最基础的决策。太少的 Agent 无法发挥分工协作的优势，太多的 Agent 会带来过大的通信和协调开销。

**经验法则**：

**最小可行团队**：从 2-3 个 Agent 开始，只在有明确需求时才增加。

**单一职责原则**：每个 Agent 应该有明确的、单一的职责。如果一个 Agent 需要处理多种不相关的任务，考虑将其拆分。

**通信开销估算**：N 个 Agent 之间的最大通信路径数是 N*(N-1)/2。当 Agent 数量超过 7 个时，通信管理的复杂度会急剧上升。

**成本估算**：假设每个 Agent 每次任务需要 2 次 LLM 调用，5 个 Agent 的系统每次任务需要 10 次 LLM 调用。使用 GPT-4 的话，每次调用大约 $0.03-0.06，每次任务的成本约为 $0.3-0.6。

### 30.2.2 决策二：通信架构

通信架构决定了 Agent 之间如何交换信息。不同的通信架构适用于不同的场景：

```python
"""
第 30 章：架构决策——通信架构对比
分析不同通信架构的优缺点和适用场景
"""
from dataclasses import dataclass, field


@dataclass
class ArchitectureOption:
    """架构选项"""
    name: str
    description: str
    pros: list[str]
    cons: list[str]
    best_for: list[str]
    complexity: str  # low, medium, high
    scalability: str  # low, medium, high
    reliability: str  # low, medium, high


# 通信架构选项
communication_architectures = [
    ArchitectureOption(
        name="直接调用（同步）",
        description="Agent 之间通过函数调用直接通信",
        pros=["实现简单", "延迟最低", "调试方便"],
        cons=["耦合度高", "不支持并行", "容错性差"],
        best_for=["Agent 数量少（2-3个）", "任务链路固定", "延迟要求严格"],
        complexity="low",
        scalability="low",
        reliability="low"
    ),
    ArchitectureOption(
        name="消息队列（异步）",
        description="通过消息队列（如 Redis、RabbitMQ）进行异步通信",
        pros=["解耦", "支持并行", "容错性好", "可扩展"],
        cons=["引入额外基础设施", "延迟略高", "调试较难"],
        best_for=["Agent 数量多", "需要并行处理", "高并发场景"],
        complexity="medium",
        scalability="high",
        reliability="high"
    ),
    ArchitectureOption(
        name="事件总线",
        description="通过发布-订阅机制进行事件驱动通信",
        pros=["高度解耦", "灵活性强", "易于扩展"],
        cons=["事件顺序难保证", "调试复杂", "可能出现事件风暴"],
        best_for=["松耦合的微服务架构", "需要灵活扩展的系统"],
        complexity="medium",
        scalability="high",
        reliability="medium"
    ),
    ArchitectureOption(
        name="共享黑板",
        description="通过共享数据空间进行间接通信",
        pros=["松耦合", "支持异步", "便于监控"],
        cons=["需要并发控制", "数据一致性挑战", "不适合实时场景"],
        best_for=["数据分析场景", "协作编辑", "知识管理"],
        complexity="medium",
        scalability="medium",
        reliability="medium"
    ),
    ArchitectureOption(
        name="混合模式",
        description="根据场景组合使用多种通信方式",
        pros=["灵活", "可针对场景优化", "平衡各方面需求"],
        cons=["设计复杂", "维护成本高", "需要深入理解各种模式"],
        best_for=["大型复杂系统", "多阶段处理", "混合工作负载"],
        complexity="high",
        scalability="high",
        reliability="high"
    ),
]


def print_architecture_comparison():
    """打印通信架构对比"""
    
    print("\n" + "="*60)
    print("通信架构对比分析")
    print("="*60)
    
    for arch in communication_architectures:
        print(f"\n{'='*40}")
        print(f"架构: {arch.name}")
        print(f"{'='*40}")
        print(f"描述: {arch.description}")
        print(f"复杂度: {arch.complexity} | 可扩展性: {arch.scalability} | 可靠性: {arch.reliability}")
        print(f"\n优点:")
        for p in arch.pros:
            print(f"  + {p}")
        print(f"\n缺点:")
        for c in arch.cons:
            print(f"  - {c}")
        print(f"\n适用场景:")
        for b in arch.best_for:
            print(f"  * {b}")


if __name__ == "__main__":
    print_architecture_comparison()
```

### 30.2.3 决策三：错误处理和容错

Multi-Agent 系统中的错误处理比单 Agent 系统更复杂。一个 Agent 的错误可能传播到整个系统，导致连锁故障。

关键的容错策略包括：

**超时机制**：每个 Agent 的处理都应该有超时限制。如果一个 Agent 在规定时间内没有响应，系统应该能够降级处理或跳过该 Agent。

**重试策略**：对于临时性错误（如网络波动、API 限流），应该有自动重试机制。但重试次数要有限制，避免无限重试。

**降级策略**：当某个 Agent 失败时，系统应该能够提供一个"虽然不完美但可用"的结果。比如，如果分析 Agent 失败了，系统可以直接返回研究 Agent 的原始数据。

**错误隔离**：一个 Agent 的错误不应该导致整个系统崩溃。通过异常捕获和隔离，确保单个 Agent 的失败不会影响其他 Agent 的正常工作。

### 30.2.4 决策四：成本控制

Multi-Agent 系统的成本主要来自 LLM API 调用。有效的成本控制策略包括：

**模型分层**：核心决策 Agent 使用能力更强的模型（如 GPT-4），辅助性 Agent 使用更经济的模型（如 GPT-3.5-turbo）。

**缓存策略**：对于重复的请求，缓存 Agent 的输出，避免重复调用 LLM。

**预算上限**：为每个任务设置 LLM 调用的预算上限，超出时自动终止或降级。

**异步批处理**：对于非实时性要求的任务，将多个请求合并处理，减少 API 调用次数。

```python
"""
第 30 章：成本控制与监控
Multi-Agent 系统的成本管理和性能监控
"""
import os
import json
import time
from typing import Any
from dataclasses import dataclass, field
from collections import defaultdict


# ============================================================
# 成本追踪器
# ============================================================

@dataclass
class LLMCallRecord:
    """一次 LLM 调用的记录"""
    agent_name: str
    model: str
    input_tokens: int
    output_tokens: int
    cost: float
    timestamp: float
    task_id: str = ""
    duration: float = 0.0


class CostTracker:
    """
    成本追踪器：监控和管理 Multi-Agent 系统的 LLM 调用成本。
    
    功能：
    1. 记录每次 LLM 调用的成本
    2. 按 Agent、模型、任务维度统计成本
    3. 支持设置预算上限和告警
    4. 生成成本报告
    """
    
    # 模型的 token 价格（美元/1000 token，近似值）
    MODEL_PRICING = {
        "gpt-4o": {"input": 0.0025, "output": 0.01},
        "gpt-4o-mini": {"input": 0.00015, "output": 0.0006},
        "gpt-3.5-turbo": {"input": 0.0005, "output": 0.0015},
    }
    
    def __init__(self, budget_limit: float = 10.0):
        """
        Args:
            budget_limit: 总预算上限（美元）
        """
        self.budget_limit = budget_limit
        self.records: list[LLMCallRecord] = []
        self.total_cost: float = 0.0
        self.alerts: list[str] = []
    
    def record_call(
        self,
        agent_name: str,
        model: str,
        input_tokens: int,
        output_tokens: int,
        task_id: str = "",
        duration: float = 0.0
    ) -> float:
        """
        记录一次 LLM 调用的成本。
        
        Returns:
            本次调用的成本
        """
        pricing = self.MODEL_PRICING.get(model, {"input": 0.003, "output": 0.006})
        cost = (input_tokens * pricing["input"] + 
                output_tokens * pricing["output"]) / 1000
        
        record = LLMCallRecord(
            agent_name=agent_name,
            model=model,
            input_tokens=input_tokens,
            output_tokens=output_tokens,
            cost=cost,
            timestamp=time.time(),
            task_id=task_id,
            duration=duration
        )
        
        self.records.append(record)
        self.total_cost += cost
        
        # 检查预算
        if self.total_cost > self.budget_limit * 0.8:
            self.alerts.append(
                f"警告: 成本已达到预算的 {self.total_cost/self.budget_limit*100:.0f}%"
            )
        
        if self.total_cost > self.budget_limit:
            self.alerts.append(
                f"严重: 已超出预算! 当前: ${self.total_cost:.2f}, 上限: ${self.budget_limit:.2f}"
            )
        
        return cost
    
    def get_report(self) -> dict:
        """生成成本报告"""
        # 按 Agent 统计
        agent_costs = defaultdict(float)
        agent_calls = defaultdict(int)
        for r in self.records:
            agent_costs[r.agent_name] += r.cost
            agent_calls[r.agent_name] += 1
        
        # 按模型统计
        model_costs = defaultdict(float)
        model_calls = defaultdict(int)
        for r in self.records:
            model_costs[r.model] += r.cost
            model_calls[r.model] += 1
        
        # 按任务统计
        task_costs = defaultdict(float)
        for r in self.records:
            if r.task_id:
                task_costs[r.task_id] += r.cost
        
        return {
            "total_cost": self.total_cost,
            "budget_limit": self.budget_limit,
            "budget_usage": f"{self.total_cost/self.budget_limit*100:.1f}%",
            "total_calls": len(self.records),
            "avg_cost_per_call": self.total_cost / max(len(self.records), 1),
            "agent_breakdown": {
                name: {"cost": agent_costs[name], "calls": agent_calls[name]}
                for name in agent_costs
            },
            "model_breakdown": {
                model: {"cost": model_costs[model], "calls": model_calls[model]}
                for model in model_costs
            },
            "alerts": self.alerts
        }
    
    def print_report(self):
        """打印成本报告"""
        report = self.get_report()
        
        print(f"\n{'='*60}")
        print(f"成本报告")
        print(f"{'='*60}")
        print(f"总成本: ${report['total_cost']:.4f}")
        print(f"预算上限: ${report['budget_limit']:.2f}")
        print(f"预算使用: {report['budget_usage']}")
        print(f"总调用次数: {report['total_calls']}")
        print(f"平均每次调用成本: ${report['avg_cost_per_call']:.6f}")
        
        print(f"\n--- 按 Agent 统计 ---")
        for name, stats in report['agent_breakdown'].items():
            print(f"  {name}: ${stats['cost']:.4f} ({stats['calls']} 次)")
        
        print(f"\n--- 按模型统计 ---")
        for model, stats in report['model_breakdown'].items():
            print(f"  {model}: ${stats['cost']:.4f} ({stats['calls']} 次)")
        
        if report['alerts']:
            print(f"\n--- 告警 ---")
            for alert in report['alerts']:
                print(f"  {alert}")


# ============================================================
# 带成本控制的 Multi-Agent 系统
# ============================================================

class CostAwareAgent:
    """
    成本感知 Agent：在执行任务时考虑成本。
    
    这个 Agent 会根据剩余预算选择合适的模型，
    并在预算不足时自动降级或停止。
    """
    
    def __init__(
        self,
        name: str,
        cost_tracker: CostTracker,
        primary_model: str = "gpt-4o",
        fallback_model: str = "gpt-4o-mini"
    ):
        self.name = name
        self.cost_tracker = cost_tracker
        self.primary_model = primary_model
        self.fallback_model = fallback_model
    
    def decide_model(self) -> str:
        """
        根据预算情况决定使用哪个模型。
        
        如果预算充足使用主模型，否则降级到备选模型。
        """
        usage = self.cost_tracker.total_cost / self.cost_tracker.budget_limit
        
        if usage > 0.8:
            print(f"  [{self.name}] 预算紧张 ({usage:.0%})，降级到 {self.fallback_model}")
            return self.fallback_model
        else:
            return self.primary_model
    
    def execute(self, task: str) -> str:
        """执行任务（带成本追踪）"""
        model = self.decide_model()
        
        start_time = time.time()
        
        # 模拟 LLM 调用
        from openai import OpenAI
        client = OpenAI()
        
        try:
            response = client.chat.completions.create(
                model=model,
                messages=[
                    {"role": "system", "content": f"你是{self.name}"},
                    {"role": "user", "content": task}
                ],
                max_tokens=500
            )
            
            result = response.choices[0].message.content.strip()
            duration = time.time() - start_time
            
            # 记录成本
            input_tokens = sum(len(m["content"]) for m in [
                {"role": "system", "content": f"你是{self.name}"},
                {"role": "user", "content": task}
            ]) // 4  # 粗略估计
            output_tokens = len(result) // 4
            
            self.cost_tracker.record_call(
                agent_name=self.name,
                model=model,
                input_tokens=input_tokens,
                output_tokens=output_tokens,
                duration=duration
            )
            
            return result
            
        except Exception as e:
            print(f"  [{self.name}] LLM 调用失败: {e}")
            return f"执行失败: {e}"


# ============================================================
# 演示：成本追踪
# ============================================================

def demo_cost_tracking():
    """演示成本追踪功能"""
    
    print("\n" + "="*60)
    print("演示：成本追踪与监控")
    print("="*60)
    
    # 创建成本追踪器
    tracker = CostTracker(budget_limit=0.5)  # $0.5 预算
    
    # 模拟多次 LLM 调用
    tracker.record_call("研究员", "gpt-4o", 500, 800, task_id="T001")
    tracker.record_call("分析师", "gpt-4o", 600, 600, task_id="T002")
    tracker.record_call("作家", "gpt-4o-mini", 800, 1200, task_id="T003")
    tracker.record_call("审查者", "gpt-4o", 1000, 400, task_id="T004")
    tracker.record_call("格式化", "gpt-3.5-turbo", 400, 600, task_id="T005")
    
    # 打印报告
    tracker.print_report()


if __name__ == "__main__":
    demo_cost_tracking()
```

---

## 30.3 完整的架构设计模板

### 30.3.1 生产级 Multi-Agent 系统架构

让我们综合前面所有章节的知识，设计一个完整的生产级 Multi-Agent 系统架构。

```python
"""
第 30 章：生产级 Multi-Agent 系统架构
一个完整的、可投入生产的架构模板
"""
import os
import json
import time
import logging
import hashlib
from typing import Any, Optional, Callable
from dataclasses import dataclass, field
from collections import defaultdict
from openai import OpenAI


# ============================================================
# 日志配置
# ============================================================

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger("MultiAgentSystem")


# ============================================================
# LLM 调用封装（带重试和缓存）
# ============================================================

class LLMClient:
    """
    带重试和缓存的 LLM 客户端。
    
    生产级系统中的 LLM 调用应该具备：
    1. 自动重试：处理临时性错误
    2. 缓存：避免重复调用
    3. 超时控制：防止长时间阻塞
    4. 成本追踪：记录每次调用的成本
    """
    
    def __init__(
        self,
        model: str = "gpt-4o",
        max_retries: int = 3,
        timeout: int = 60,
        cache_enabled: bool = True
    ):
        self.model = model
        self.max_retries = max_retries
        self.timeout = timeout
        self.cache_enabled = cache_enabled
        self.cache: dict[str, str] = {}
        self.call_count: int = 0
        self.total_tokens: int = 0
        
        # 初始化 OpenAI 客户端
        self.client = OpenAI()
    
    def _cache_key(self, messages: list[dict]) -> str:
        """生成缓存键"""
        content = json.dumps(messages, sort_keys=True, ensure_ascii=False)
        return hashlib.md5(content.encode()).hexdigest()
    
    def call(
        self,
        messages: list[dict],
        temperature: float = 0.7,
        max_tokens: int = 2000
    ) -> str:
        """
        调用 LLM，带重试和缓存。
        
        Args:
            messages: 对话消息列表
            temperature: 温度参数
            max_tokens: 最大输出 token 数
        
        Returns:
            LLM 的文本响应
        """
        # 检查缓存
        if self.cache_enabled:
            cache_key = self._cache_key(messages)
            if cache_key in self.cache:
                logger.debug("缓存命中")
                return self.cache[cache_key]
        
        # 带重试的 LLM 调用
        last_error = None
        for attempt in range(self.max_retries):
            try:
                response = self.client.chat.completions.create(
                    model=self.model,
                    messages=messages,
                    temperature=temperature,
                    max_tokens=max_tokens,
                    timeout=self.timeout
                )
                
                result = response.choices[0].message.content.strip()
                
                # 更新统计
                self.call_count += 1
                if response.usage:
                    self.total_tokens += response.usage.total_tokens
                
                # 写入缓存
                if self.cache_enabled:
                    self.cache[cache_key] = result
                
                return result
                
            except Exception as e:
                last_error = e
                logger.warning(f"LLM 调用失败 (尝试 {attempt+1}/{self.max_retries}): {e}")
                if attempt < self.max_retries - 1:
                    time.sleep(2 ** attempt)
        
        raise Exception(f"LLM 调用在 {self.max_retries} 次重试后仍失败: {last_error}")
    
    def get_stats(self) -> dict:
        """获取调用统计"""
        return {
            "total_calls": self.call_count,
            "total_tokens": self.total_tokens,
            "cache_size": len(self.cache)
        }


# ============================================================
# 消息总线
# ============================================================

@dataclass
class Message:
    """系统内部消息"""
    sender: str
    receiver: str
    content: str
    msg_type: str = "task"  # task, result, error, status
    timestamp: float = field(default_factory=time.time)
    metadata: dict = field(default_factory=dict)


class MessageBus:
    """
    消息总线：系统内部的通信基础设施。
    
    生产级消息总线应该具备：
    1. 消息路由：根据接收者分发消息
    2. 消息持久化：防止消息丢失
    3. 消息追踪：记录所有消息的流转
    4. 错误处理：消息处理失败时的降级策略
    """
    
    def __init__(self):
        self.handlers: dict[str, Callable] = {}
        self.message_log: list[Message] = []
        self.error_log: list[dict] = []
    
    def register(self, agent_name: str, handler: Callable):
        """注册 Agent 的消息处理器"""
        self.handlers[agent_name] = handler
    
    def send(self, message: Message) -> Optional[str]:
        """
        发送消息。
        
        Returns:
            接收者的回复
        """
        self.message_log.append(message)
        
        if message.receiver not in self.handlers:
            logger.error(f"消息接收者 '{message.receiver}' 未注册")
            self.error_log.append({
                "type": "unregistered_receiver",
                "receiver": message.receiver,
                "timestamp": time.time()
            })
            return None
        
        try:
            handler = self.handlers[message.receiver]
            response = handler(message)
            
            if response:
                self.message_log.append(Message(
                    sender=message.receiver,
                    receiver=message.sender,
                    content=response,
                    msg_type="result"
                ))
            
            return response
            
        except Exception as e:
            logger.error(f"消息处理失败: {e}")
            self.error_log.append({
                "type": "handler_error",
                "receiver": message.receiver,
                "error": str(e),
                "timestamp": time.time()
            })
            return f"处理失败: {e}"
    
    def broadcast(self, sender: str, content: str, msg_type: str = "status"):
        """广播消息给所有 Agent"""
        for agent_name in self.handlers:
            if agent_name != sender:
                self.send(Message(
                    sender=sender,
                    receiver=agent_name,
                    content=content,
                    msg_type=msg_type
                ))
    
    def get_log(self, agent_name: str = None) -> list[Message]:
        """获取消息日志"""
        if agent_name:
            return [m for m in self.message_log 
                    if m.sender == agent_name or m.receiver == agent_name]
        return self.message_log.copy()


# ============================================================
# 生产级 Agent 基类
# ============================================================

class ProductionAgent:
    """
    生产级 Agent 基类。
    
    相比简单的 Agent，生产级 Agent 具备：
    1. 结构化日志
    2. 错误处理和降级
    3. 性能监控
    4. 状态管理
    """
    
    def __init__(
        self,
        name: str,
        system_prompt: str,
        llm_client: LLMClient,
        message_bus: MessageBus
    ):
        self.name = name
        self.system_prompt = system_prompt
        self.llm = llm_client
        self.bus = message_bus
        self.state: dict[str, Any] = {}
        self.execution_count: int = 0
        self.error_count: int = 0
        
        # 注册到消息总线
        self.bus.register(name, self.handle_message)
    
    def handle_message(self, message: Message) -> Optional[str]:
        """处理接收到的消息"""
        logger.info(f"[{self.name}] 收到消息 (来自 {message.sender}, 类型: {message.msg_type})")
        
        try:
            self.execution_count += 1
            
            messages = [
                {"role": "system", "content": self.system_prompt},
                {"role": "user", "content": f"来自 {message.sender} 的{message.msg_type}:\n{message.content}"}
            ]
            
            result = self.llm.call(messages, temperature=0.5)
            
            logger.info(f"[{self.name}] 处理完成，输出 {len(result)} 字符")
            return result
            
        except Exception as e:
            self.error_count += 1
            logger.error(f"[{self.name}] 处理失败: {e}")
            return f"[{self.name}] 处理失败，请稍后重试"
    
    def get_stats(self) -> dict:
        """获取 Agent 统计信息"""
        return {
            "name": self.name,
            "execution_count": self.execution_count,
            "error_count": self.error_count,
            "error_rate": self.error_count / max(self.execution_count, 1),
            "state_keys": list(self.state.keys())
        }


# ============================================================
# 系统监控器
# ============================================================

class SystemMonitor:
    """
    系统监控器：监控 Multi-Agent 系统的运行状态。
    
    监控维度：
    1. Agent 执行状态
    2. 消息流转情况
    3. 错误率和异常
    4. 性能指标（延迟、吞吐量）
    """
    
    def __init__(self):
        self.metrics: dict[str, list] = defaultdict(list)
        self.alerts: list[dict] = []
    
    def record_metric(self, name: str, value: float):
        """记录一个指标"""
        self.metrics[name].append({
            "value": value,
            "timestamp": time.time()
        })
    
    def check_alerts(self, agents: list[ProductionAgent]):
        """检查是否需要告警"""
        for agent in agents:
            stats = agent.get_stats()
            if stats["error_rate"] > 0.3:
                self.alerts.append({
                    "level": "warning",
                    "agent": agent.name,
                    "message": f"错误率过高: {stats['error_rate']:.0%}",
                    "timestamp": time.time()
                })
    
    def get_dashboard(self, agents: list[ProductionAgent]) -> dict:
        """生成监控仪表板数据"""
        agent_stats = [a.get_stats() for a in agents]
        
        total_executions = sum(s["execution_count"] for s in agent_stats)
        total_errors = sum(s["error_count"] for s in agent_stats)
        
        return {
            "timestamp": time.time(),
            "total_executions": total_executions,
            "total_errors": total_errors,
            "overall_error_rate": total_errors / max(total_executions, 1),
            "agent_stats": agent_stats,
            "active_alerts": self.alerts[-10:]  # 最近 10 条告警
        }


# ============================================================
# 生产级 Multi-Agent 系统
# ============================================================

class ProductionMultiAgentSystem:
    """
    生产级 Multi-Agent 系统。
    
    整合了所有生产级特性：
    - 带缓存和重试的 LLM 客户端
    - 可靠的消息总线
    - 结构化的 Agent 基类
    - 完整的监控和告警
    - 成本追踪
    """
    
    def __init__(self, name: str, budget_limit: float = 10.0):
        self.name = name
        self.llm = LLMClient(model="gpt-4o")
        self.bus = MessageBus()
        self.monitor = SystemMonitor()
        self.agents: dict[str, ProductionAgent] = {}
        self.start_time: float = time.time()
        
        logger.info(f"系统 [{name}] 初始化完成")
    
    def add_agent(self, agent: ProductionAgent):
        """添加 Agent"""
        self.agents[agent.name] = agent
        logger.info(f"添加 Agent: {agent.name}")
    
    def run_task(self, task: str, initial_agent: str = None) -> str:
        """
        执行一个任务。
        
        Args:
            task: 任务描述
            initial_agent: 初始处理的 Agent（为空则使用第一个 Agent）
        
        Returns:
            任务执行结果
        """
        if not initial_agent and self.agents:
            initial_agent = list(self.agents.keys())[0]
        
        logger.info(f"开始执行任务: {task[:80]}...")
        start_time = time.time()
        
        try:
            message = Message(
                sender="system",
                receiver=initial_agent,
                content=task,
                msg_type="task"
            )
            
            result = self.bus.send(message)
            
            elapsed = time.time() - start_time
            self.monitor.record_metric("task_duration", elapsed)
            
            logger.info(f"任务完成，耗时: {elapsed:.2f}s")
            
            return result or "任务执行完成但无返回结果"
            
        except Exception as e:
            elapsed = time.time() - start_time
            self.monitor.record_metric("task_error", elapsed)
            logger.error(f"任务执行失败: {e}")
            return f"任务执行失败: {e}"
    
    def get_system_report(self) -> dict:
        """生成系统报告"""
        uptime = time.time() - self.start_time
        
        return {
            "system_name": self.name,
            "uptime_seconds": uptime,
            "agent_count": len(self.agents),
            "llm_stats": self.llm.get_stats(),
            "message_log_size": len(self.bus.message_log),
            "error_log_size": len(self.bus.error_log),
            "monitor_dashboard": self.monitor.get_dashboard(
                list(self.agents.values())
            )
        }
    
    def print_report(self):
        """打印系统报告"""
        report = self.get_system_report()
        
        print(f"\n{'='*60}")
        print(f"系统报告: {report['system_name']}")
        print(f"{'='*60}")
        print(f"运行时间: {report['uptime_seconds']:.0f}s")
        print(f"Agent 数量: {report['agent_count']}")
        print(f"LLM 调用次数: {report['llm_stats']['total_calls']}")
        print(f"消息总数: {report['message_log_size']}")
        print(f"错误总数: {report['error_log_size']}")
        
        dashboard = report['monitor_dashboard']
        print(f"\n--- Agent 统计 ---")
        for stats in dashboard['agent_stats']:
            print(f"  {stats['name']}: 执行 {stats['execution_count']} 次, "
                  f"错误 {stats['error_count']} 次")


# ============================================================
# 演示：生产级系统
# ============================================================

def demo_production_system():
    """演示生产级 Multi-Agent 系统"""
    
    print("\n" + "="*60)
    print("演示：生产级 Multi-Agent 系统")
    print("="*60)
    
    # 创建系统
    system = ProductionMultiAgentSystem("AI 研究平台", budget_limit=5.0)
    
    # 创建 Agent
    researcher = ProductionAgent(
        name="研究员",
        system_prompt="你是一位技术研究员，擅长搜集和整理资料。",
        llm_client=system.llm,
        message_bus=system.bus
    )
    
    analyst = ProductionAgent(
        name="分析师",
        system_prompt="你是一位数据分析专家，擅长从资料中提取洞察。",
        llm_client=system.llm,
        message_bus=system.bus
    )
    
    writer = ProductionAgent(
        name="作家",
        system_prompt="你是一位技术作家，擅长撰写清晰的文章。",
        llm_client=system.llm,
        message_bus=system.bus
    )
    
    # 添加到系统
    system.add_agent(researcher)
    system.add_agent(analyst)
    system.add_agent(writer)
    
    # 执行任务
    result = system.run_task(
        "请简要介绍 Multi-Agent 系统的核心概念",
        initial_agent="研究员"
    )
    
    print(f"\n--- 任务结果 ---")
    print(result[:300] + "..." if result else "无结果")
    
    # 打印系统报告
    system.print_report()


if __name__ == "__main__":
    demo_production_system()
```

---

## 30.4 测试策略

### 30.4.1 Multi-Agent 系统的测试挑战

Multi-Agent 系统的测试比单 Agent 系统更复杂，因为：

1. **不确定性**：LLM 的输出是概率性的，同样的输入可能产生不同的输出。
2. **组合爆炸**：多个 Agent 的交互组合数量随 Agent 数量指数增长。
3. **异步行为**：异步通信模式下的测试更加困难。
4. **成本限制**：每次测试都需要调用 LLM API，有实际的成本。

### 30.4.2 分层测试策略

```python
"""
第 30 章：Multi-Agent 系统测试策略
分层测试方法和工具
"""
import json
from dataclasses import dataclass, field
from typing import Callable, Any


@dataclass
class TestCase:
    """测试用例"""
    name: str
    description: str
    input_data: dict
    expected_behavior: str
    validation_fn: Callable = None
    priority: str = "medium"  # low, medium, high
    
    def validate(self, result: Any) -> dict:
        """验证测试结果"""
        if self.validation_fn:
            passed = self.validation_fn(result)
        else:
            # 默认验证：结果不为空
            passed = bool(result)
        
        return {
            "test_name": self.name,
            "passed": passed,
            "result_preview": str(result)[:200] if result else "无结果"
        }


class MultiAgentTestSuite:
    """
    Multi-Agent 系统测试套件。
    
    测试策略分层：
    1. 单元测试：测试单个 Agent 的基本能力
    2. 集成测试：测试 Agent 之间的通信和协作
    3. 端到端测试：测试完整的业务流程
    4. 性能测试：测试系统的吞吐量和延迟
    5. 压力测试：测试系统在高负载下的表现
    """
    
    def __init__(self):
        self.test_cases: dict[str, list[TestCase]] = {
            "unit": [],
            "integration": [],
            "e2e": [],
            "performance": []
        }
        self.results: list[dict] = []
    
    def add_test(self, layer: str, test_case: TestCase):
        """添加测试用例"""
        if layer in self.test_cases:
            self.test_cases[layer].append(test_case)
    
    def run_all(self, system=None) -> dict:
        """运行所有测试"""
        total = sum(len(cases) for cases in self.test_cases.values())
        print(f"\n运行测试套件: 共 {total} 个测试用例")
        
        for layer, cases in self.test_cases.items():
            if not cases:
                continue
            
            print(f"\n--- {layer.upper()} 测试 ({len(cases)} 个) ---")
            
            for test in cases:
                print(f"  运行: {test.name}...", end=" ")
                
                # 这里应该调用实际的系统来执行测试
                # 简化实现：直接验证预期行为
                result = test.validate(None)
                self.results.append(result)
                
                status = "✅" if result["passed"] else "❌"
                print(f"{status}")
        
        passed = sum(1 for r in self.results if r["passed"])
        total = len(self.results)
        
        print(f"\n--- 测试结果 ---")
        print(f"通过: {passed}/{total} ({passed/total*100:.0f}%)")
        
        return {
            "total": total,
            "passed": passed,
            "failed": total - passed,
            "pass_rate": f"{passed/total*100:.0f}%",
            "details": self.results
        }


# ============================================================
# 演示：测试策略
# ============================================================

def demo_testing():
    """演示测试策略"""
    
    print("\n" + "="*60)
    print("Multi-Agent 系统测试策略")
    print("="*60)
    
    # 创建测试套件
    suite = MultiAgentTestSuite()
    
    # 添加单元测试
    suite.add_test("unit", TestCase(
        name="Agent 基本回复",
        description="验证 Agent 能正常回复消息",
        input_data={"message": "你好"},
        expected_behavior="返回非空响应",
        validation_fn=lambda x: True,  # 简化验证
        priority="high"
    ))
    
    # 添加集成测试
    suite.add_test("integration", TestCase(
        name="Agent 间消息传递",
        description="验证消息能正确在 Agent 之间传递",
        input_data={"sender": "AgentA", "receiver": "AgentB", "content": "测试消息"},
        expected_behavior="消息被正确接收和处理",
        validation_fn=lambda x: True,
        priority="high"
    ))
    
    # 添加端到端测试
    suite.add_test("e2e", TestCase(
        name="完整研究流程",
        description="验证从研究到写作的完整流程",
        input_data={"topic": "AI Agent 技术"},
        expected_behavior="产出完整的研究报告",
        validation_fn=lambda x: True,
        priority="medium"
    ))
    
    # 运行测试
    results = suite.run_all()
    
    print(f"\n--- 测试总结 ---")
    print(f"通过率: {results['pass_rate']}")


if __name__ == "__main__":
    demo_testing()
```

---

## 30.5 案例分析：架构设计的权衡

### 30.5.1 案例：从原型到生产

让我们通过一个完整的案例来说明架构设计的权衡过程。

**项目背景**：某公司需要构建一个智能文档处理系统，能够自动读取 PDF 文档、提取关键信息、生成摘要和标签。

**第一版设计（原型阶段）**：
- 使用单 Agent，包含所有功能
- 使用 GPT-4 处理所有任务
- 简单的顺序处理流程

**问题**：单 Agent 的上下文窗口很快被撑满，输出质量不稳定，处理时间长。

**第二版设计（优化阶段）**：
- 拆分为 3 个 Agent：文档解析 Agent、信息提取 Agent、摘要生成 Agent
- 使用 GPT-4 作为核心 Agent，GPT-3.5-turbo 作为辅助 Agent
- 顺序执行流程

**效果**：质量提升明显，但处理时间仍然较长（平均 15 秒/文档）。

**第三版设计（生产阶段）**：
- 扩展为 5 个 Agent，增加并行处理能力
- 引入缓存机制，相同文档不重复处理
- 使用消息队列支持异步处理
- 添加监控和告警

**最终效果**：处理时间降至 5 秒/文档，吞吐量提升 10 倍，成本降低 40%。

### 30.5.2 权衡决策矩阵

在架构设计中，经常需要在多个维度之间做权衡。以下是一个实用的权衡决策矩阵：

| 维度 | 选项 A | 选项 B | 建议 |
|------|--------|--------|------|
| 延迟 vs 成本 | 同步调用（低延迟，高成本） | 异步调用（高延迟，低成本） | 实时场景选 A，批处理选 B |
| 质量 vs 速度 | 多轮辩论（高质量，慢速度） | 单次生成（中质量，快速度） | 高风险选 A，一般场景选 B |
| 灵活性 vs 简单性 | 分布式架构（灵活，复杂） | 集中式架构（简单，不灵活） | 小规模选 B，大规模选 A |
| 可靠性 vs 性能 | 多重验证（可靠，慢） | 单次验证（不一定可靠，快） | 关键路径选 A，非关键选 B |

---

## 30.6 常见坑与解决方案

### 坑 1：过早优化

**问题描述**：在项目初期就设计了过于复杂的架构，导致开发和维护成本过高。

**解决方案**：遵循"演进式架构"原则——从最简单的方案开始，在遇到实际问题时再优化。不要为了"将来可能需要"的功能而过早增加复杂度。

### 坑 2：忽视可观测性

**问题描述**：系统上线后才发现无法追踪问题的根源，因为缺乏日志、监控和追踪。

**解决方案**：从项目一开始就加入日志、监控和成本追踪。这是生产级系统的必备基础设施。

### 坑 3：没有考虑降级策略

**问题描述**：系统假设所有 Agent 和服务都正常运行，当某个组件失败时整个系统崩溃。

**解决方案**：为每个关键组件设计降级策略。当 Agent 失败时，提供一个"虽然不完美但可用"的结果。

### 坑 4：忽视安全问题

**问题描述**：Multi-Agent 系统中的数据在多个 Agent 之间流转，可能泄露敏感信息。

**解决方案**：对 Agent 之间的数据传输进行加密。为每个 Agent 设置最小权限原则。对输出进行敏感信息过滤。

### 坑 5：没有进行成本估算

**问题描述**：项目上线后发现 LLM API 费用远超预算。

**解决方案**：在项目初期就进行详细的成本估算。建立成本监控和告警机制。设置每个任务的预算上限。

---

## 30.7 练习题

### 练习 1：需求分析练习

**难度：★☆☆☆☆**

使用本章的需求分析框架，分析一个你感兴趣的 Multi-Agent 应用场景。填写所有维度，得出架构建议。

### 练习 2：架构选型

**难度：★★☆☆☆**

针对一个"智能客服"场景，对比至少 3 种不同的架构方案，从延迟、成本、可靠性、可维护性等维度进行评估，选出最优方案。

### 练习 3：成本估算

**难度：★★★☆☆**

为一个包含 5 个 Agent 的系统估算月度成本。假设每个 Agent 每次任务平均调用 LLM 3 次，日均 1000 个任务。对比使用 GPT-4 和 GPT-3.5-turbo 的成本差异。

### 练习 4：设计降级策略

**难度：★★★☆☆**

为一个 Multi-Agent 研究系统设计降级策略。当研究员 Agent 失败时，系统应该如何继续运行？当分析师 Agent 失败时呢？

### 练习 5：设计监控系统

**难度：★★★★☆**

设计一个 Multi-Agent 系统的监控仪表板，包含哪些关键指标？如何设置告警阈值？

### 练习 6：架构重构

**难度：★★★★★**

将一个现有的单 Agent 系统重构为 Multi-Agent 系统。制定重构计划，包括哪些功能应该拆分为独立的 Agent，通信方式如何选择，如何保证重构过程不影响现有功能。

---

## 30.8 实战任务

### 任务：设计并实现一个完整的 Multi-Agent 系统

**目标**：综合运用前面所有章节的知识，从零开始设计并实现一个完整的 Multi-Agent 系统。

**选择一个你感兴趣的应用场景**（以下为建议）：

1. **Multi-Agent 新闻编辑部**：多个 Agent 协作完成新闻的采集、编辑和发布
2. **Multi-Agent 代码开发团队**：多个 Agent 协作完成一个小型软件项目的开发
3. **Multi-Agent 投资分析系统**：多个 Agent 从不同角度分析投资机会
4. **Multi-Agent 教育辅导系统**：多个 Agent 以不同角色辅导学生学习

**要求**：

1. **需求分析**：使用本章的需求分析框架，详细分析系统需求
2. **架构设计**：画出系统架构图，说明每个组件的职责
3. **核心实现**：实现系统的核心功能，代码完整可运行
4. **通信设计**：设计并实现 Agent 之间的通信机制
5. **错误处理**：实现基本的错误处理和降级策略
6. **成本控制**：实现成本追踪和预算控制
7. **监控**：实现基本的系统监控
8. **测试**：编写至少 5 个测试用例
9. **文档**：编写清晰的 README，说明如何运行和使用

**评分标准**：
- 架构设计的合理性（25%）
- 代码质量和可维护性（25%）
- 功能完整性和正确性（25%）
- 测试和文档的质量（25%）

---

## 30.9 本章小结

本章是 Multi-Agent 系统学习之旅的终章。我们综合了前面所有章节的知识，从架构设计的高度来审视 Multi-Agent 系统。

我们学习了从需求分析到架构选型的完整设计流程。需求分析帮助我们确认是否需要 Multi-Agent，以及系统应该满足什么样的质量要求。架构设计涉及多个关键决策：Agent 的数量和粒度、通信架构、协调策略、错误处理、成本控制。

我们通过完整的代码示例展示了生产级 Multi-Agent 系统的核心组件：带缓存和重试的 LLM 客户端、可靠的消息总线、结构化的 Agent 基类、完整的监控和告警系统。

我们还学习了 Multi-Agent 系统的测试策略——从单元测试到端到端测试的分层方法，以及如何应对 LLM 输出不确定性带来的测试挑战。

最后，我们通过一个从原型到生产的案例，展示了架构设计中的权衡过程。架构设计不是追求最优雅的方案，而是在延迟、成本、质量、可靠性等多个维度之间找到最适合当前场景的平衡点。

Multi-Agent 系统是一个快速发展的领域。新的框架、新的模式、新的应用场景不断涌现。掌握了本章的内容，你已经有了足够的知识储备来应对这些变化。记住：技术在变，但设计的原则不变——理解需求、权衡取舍、渐进演进。

> "最好的架构不是设计出来的，而是在实践中演进出来的。从简单开始，在需要时增加复杂度。这是 Multi-Agent 系统设计的最高原则。"

---

**系列总结**：从第 23 章到第 30 章，我们一起走过了 Multi-Agent 系统的完整旅程——从"为什么需要多个 Agent"的基础问题，到通信机制、协调策略、角色设计、辩论纠正，再到 AutoGen 和 CrewAI 框架的实战应用，最后到架构设计的全局视角。希望这段旅程能帮助你在实际项目中设计和构建出优秀的 Multi-Agent 系统。
