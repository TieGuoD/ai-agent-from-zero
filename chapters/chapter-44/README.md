# 第 44 章：Workflow vs Agent —— 选型决策框架

> **本章定位：** 不是所有问题都需要 Agent 来解决。有些场景用确定性的工作流更合适，有些场景需要 Agent 的灵活性。本章将建立一个清晰的选型决策框架，帮你为每个具体场景选择最合适的技术方案。

---

## 学习目标

完成本章学习后，你将能够：

1. **清晰区分 Workflow 和 Agent 的本质差异** —— 能从确定性、灵活性、可控性等多个维度对比两种范式
2. **使用决策矩阵选择技术方案** —— 能根据任务特征用系统化的方法选择 Workflow 或 Agent
3. **设计 Workflow-Agent 混合架构** —— 能在同一个系统中组合使用两种范式
4. **评估迁移成本和收益** —— 能判断何时应该从 Workflow 迁移到 Agent，或反过来
5. **设计渐进式的 Agent 化策略** —— 能将现有 Workflow 系统逐步升级为 Agent 系统

## 核心问题

1. **什么时候用 Workflow，什么时候用 Agent？** 这不是一个简单的二选一，而是一个需要综合考虑多个因素的决策。
2. **Workflow 和 Agent 能否共存？** 一个系统中是否可以同时使用两种范式？
3. **从 Workflow 迁移到 Agent 的风险和收益是什么？** 这种迁移是否值得？

---

## 44.1 Workflow 和 Agent 的本质区别

### 44.1.1 什么是 Workflow

Workflow（工作流）是一系列预定义步骤的有序执行。每一步做什么、什么时候执行、输入输出是什么，都是在设计时就确定好的。它就像一条流水线——产品从一端进入，经过固定的加工步骤，从另一端出来。

想象一个电商的订单处理流程：用户下单 -> 库存检查 -> 支付处理 -> 发货通知 -> 订单完成。每个步骤都是确定的，顺序是固定的，每一步的输入输出都有明确的定义。这就是一个典型的 Workflow。

Workflow 的核心特征是确定性。给定相同的输入，Workflow 永远产生相同的输出（或者说相同顺序的中间步骤）。这让 Workflow 非常容易测试、调试和优化。

### 44.1.2 什么是 Agent

Agent（智能体）是一个能够根据环境动态决策的系统。它不是按照预定义的步骤执行，而是根据当前的情况自主决定下一步做什么。它就像一个有经验的员工——面对不同的问题，会灵活地选择不同的解决策略。

想象一个智能客服：用户可能问产品信息、可能抱怨物流问题、可能要求退款、可能只是闲聊。Agent 需要理解用户的意图，决定是否需要搜索知识库、是否需要查询订单、是否需要转接人工——这些决策都是动态的，无法在设计时完全预定义。

Agent 的核心特征是非确定性（或说灵活性）。给定相同的输入，Agent 可能采取不同的行动路径。这让 Agent 能够处理意料之外的情况，但也让它更难预测和控制。

### 44.1.3 核心差异对比

| 维度 | Workflow | Agent |
|------|----------|-------|
| 决策方式 | 预定义 | 动态推理 |
| 执行路径 | 固定 | 可变 |
| 确定性 | 高 | 低 |
| 可预测性 | 高 | 中 |
| 灵活性 | 低 | 高 |
| 可测试性 | 容易 | 较难 |
| 成本可控性 | 高 | 中 |
| 适用场景 | 结构化任务 | 开放式任务 |

---

## 44.2 选型决策框架

### 44.2.1 决策矩阵

```python
from enum import Enum
from typing import Dict, List, Tuple

class TaskType(Enum):
    """任务类型"""
    STRUCTURED = "structured"           # 结构化任务
    SEMI_STRUCTURED = "semi_structured" # 半结构化任务
    UNSTRUCTURED = "unstructured"       # 非结构化任务

class FlexibilityNeed(Enum):
    """灵活性需求"""
    LOW = "low"       # 低：步骤完全可预定义
    MEDIUM = "medium" # 中：部分步骤需要动态决策
    HIGH = "high"     # 高：几乎无法预定义所有步骤

class PredictabilityNeed(Enum):
    """可预测性需求"""
    HIGH = "high"   # 高：必须确保每次执行路径一致
    MEDIUM = "medium" # 中：允许一定变化
    LOW = "low"     # 低：结果正确即可

class CostConstraint(Enum):
    """成本约束"""
    TIGHT = "tight"   # 严格：预算有限
    MEDIUM = "medium" # 中等
    LOOSE = "loose"   # 宽松

@dataclass
class TaskProfile:
    """任务画像"""
    name: str
    task_type: TaskType
    flexibility_need: FlexibilityNeed
    predictability_need: PredictabilityNeed
    cost_constraint: CostConstraint
    description: str = ""
    
    # 额外因素
    has_clear_steps: bool = True          # 步骤是否清晰
    input_variability: str = "low"        # 输入变异性: low/medium/high
    output_variability: str = "low"       # 输出变异性
    error_tolerance: str = "low"          # 错误容忍度: low/medium/high
    latency_requirement: str = "medium"   # 延迟要求: low/medium/high

class SelectionFramework:
    """
    选型决策框架
    
    根据任务画像推荐使用 Workflow、Agent 还是混合方案。
    """
    
    def __init__(self):
        # 权重配置
        self.weights = {
            "flexibility": 0.25,
            "predictability": 0.20,
            "cost": 0.15,
            "structure": 0.15,
            "error_tolerance": 0.15,
            "latency": 0.10,
        }
    
    def analyze(self, profile: TaskProfile) -> Dict:
        """
        分析任务并推荐方案
        
        返回推荐结果和理由。
        """
        scores = {
            "workflow": 0.0,
            "agent": 0.0,
            "hybrid": 0.0,
        }
        reasons = []
        
        # 因素 1: 灵活性需求
        if profile.flexibility_need == FlexibilityNeed.HIGH:
            scores["agent"] += self.weights["flexibility"]
            reasons.append("高灵活性需求 -> 倾向 Agent")
        elif profile.flexibility_need == FlexibilityNeed.LOW:
            scores["workflow"] += self.weights["flexibility"]
            reasons.append("低灵活性需求 -> 倾向 Workflow")
        else:
            scores["hybrid"] += self.weights["flexibility"]
            reasons.append("中等灵活性需求 -> 倾向混合方案")
        
        # 因素 2: 可预测性需求
        if profile.predictability_need == PredictabilityNeed.HIGH:
            scores["workflow"] += self.weights["predictability"]
            reasons.append("高可预测性需求 -> 倾向 Workflow")
        elif profile.predictability_need == PredictabilityNeed.LOW:
            scores["agent"] += self.weights["predictability"]
            reasons.append("低可预测性需求 -> 倾向 Agent")
        else:
            scores["hybrid"] += self.weights["predictability"]
        
        # 因素 3: 成本约束
        if profile.cost_constraint == CostConstraint.TIGHT:
            scores["workflow"] += self.weights["cost"]
            reasons.append("成本约束严格 -> 倾向 Workflow")
        elif profile.cost_constraint == CostConstraint.LOOSE:
            scores["agent"] += self.weights["cost"]
        else:
            scores["hybrid"] += self.weights["cost"]
        
        # 因素 4: 任务结构化程度
        if profile.task_type == TaskType.STRUCTURED:
            scores["workflow"] += self.weights["structure"]
            reasons.append("结构化任务 -> 倾向 Workflow")
        elif profile.task_type == TaskType.UNSTRUCTURED:
            scores["agent"] += self.weights["structure"]
            reasons.append("非结构化任务 -> 倾向 Agent")
        else:
            scores["hybrid"] += self.weights["structure"]
        
        # 因素 5: 步骤清晰度
        if profile.has_clear_steps:
            scores["workflow"] += 0.05
            reasons.append("步骤清晰 -> Workflow 更合适")
        else:
            scores["agent"] += 0.05
            reasons.append("步骤不清晰 -> Agent 更合适")
        
        # 因素 6: 输入变异性
        variability_map = {"low": "workflow", "medium": "hybrid", "high": "agent"}
        recommended = variability_map.get(profile.input_variability, "hybrid")
        scores[recommended] += 0.05
        
        # 决定最终推荐
        best_option = max(scores, key=scores.get)
        confidence = scores[best_option] / sum(scores.values())
        
        return {
            "recommendation": best_option,
            "confidence": confidence,
            "scores": scores,
            "reasons": reasons,
            "profile": profile.name,
        }
    
    def print_analysis(self, result: Dict):
        """打印分析结果"""
        print("\n" + "=" * 60)
        print(f"  选型分析: {result['profile']}")
        print("=" * 60)
        
        print(f"\n推荐方案: {result['recommendation'].upper()}")
        print(f"置信度: {result['confidence']:.1%}")
        
        print(f"\n得分:")
        for option, score in result["scores"].items():
            bar = "█" * int(score * 40) + "░" * (40 - int(score * 40))
            marker = " <-- 推荐" if option == result["recommendation"] else ""
            print(f"  {option:10s} {bar} {score:.3f}{marker}")
        
        print(f"\n决策理由:")
        for reason in result["reasons"]:
            print(f"  - {reason}")
        
        print("=" * 60)


# ============================================================
# 预定义的任务画像
# ============================================================

TASK_PROFILES = {
    "email_processing": TaskProfile(
        name="邮件处理流水线",
        task_type=TaskType.STRUCTURED,
        flexibility_need=FlexibilityNeed.LOW,
        predictability_need=PredictabilityNeed.HIGH,
        cost_constraint=CostConstraint.TIGHT,
        has_clear_steps=True,
        input_variability="medium",
        description="接收邮件 -> 分类 -> 提取信息 -> 存入数据库 -> 发送通知",
    ),
    
    "customer_service": TaskProfile(
        name="智能客服",
        task_type=TaskType.UNSTRUCTURED,
        flexibility_need=FlexibilityNeed.HIGH,
        predictability_need=PredictabilityNeed.LOW,
        cost_constraint=CostConstraint.MEDIUM,
        has_clear_steps=False,
        input_variability="high",
        description="理解用户意图 -> 搜索知识库 -> 生成回答 -> 质量检查",
    ),
    
    "data_pipeline": TaskProfile(
        name="数据处理管道",
        task_type=TaskType.STRUCTURED,
        flexibility_need=FlexibilityNeed.LOW,
        predictability_need=PredictabilityNeed.HIGH,
        cost_constraint=CostConstraint.TIGHT,
        has_clear_steps=True,
        input_variability="low",
        description="数据清洗 -> 转换 -> 验证 -> 加载 -> 报告",
    ),
    
    "code_review": TaskProfile(
        name="代码审查助手",
        task_type=TaskType.SEMI_STRUCTURED,
        flexibility_need=FlexibilityNeed.MEDIUM,
        predictability_need=PredictabilityNeed.MEDIUM,
        cost_constraint=CostConstraint.MEDIUM,
        has_clear_steps=False,
        input_variability="high",
        description="阅读代码 -> 识别问题 -> 给出建议 -> 生成报告",
    ),
    
    "report_generation": TaskProfile(
        name="自动化报告生成",
        task_type=TaskType.STRUCTURED,
        flexibility_need=FlexibilityNeed.LOW,
        predictability_need=PredictabilityNeed.HIGH,
        cost_constraint=CostConstraint.MEDIUM,
        has_clear_steps=True,
        input_variability="low",
        description="收集数据 -> 分析 -> 生成图表 -> 撰写文字 -> 格式化输出",
    ),
}


# ============================================================
# 使用示例
# ============================================================

def demo_selection_framework():
    """演示选型框架"""
    framework = SelectionFramework()
    
    for task_id, profile in TASK_PROFILES.items():
        result = framework.analyze(profile)
        framework.print_analysis(result)


if __name__ == "__main__":
    demo_selection_framework()
```

---

## 44.3 Workflow-Agent 混合架构

### 44.3.1 混合架构设计

```python
from typing import Dict, List, Callable, Any
from enum import Enum

class StepType(Enum):
    """步骤类型"""
    WORKFLOW = "workflow"  # 确定性步骤
    AGENT = "agent"        # 智能决策步骤

class HybridStep:
    """混合步骤"""
    
    def __init__(self, name: str, step_type: StepType, 
                 func: Callable = None, agent_func: Callable = None):
        self.name = name
        self.step_type = step_type
        self.func = func
        self.agent_func = agent_func
    
    def execute(self, context: Dict) -> Dict:
        """执行步骤"""
        if self.step_type == StepType.WORKFLOW:
            return self.func(context) if self.func else context
        else:
            return self.agent_func(context) if self.agent_func else context

class HybridPipeline:
    """
    混合流水线
    
    在同一个流程中组合使用 Workflow 和 Agent 步骤。
    
    使用场景示例：
    1. 数据收集（Workflow）-> 内容理解（Agent）-> 格式化输出（Workflow）
    2. 请求验证（Workflow）-> 意图理解（Agent）-> 路由分发（Workflow）
    """
    
    def __init__(self, name: str):
        self.name = name
        self.steps: List[HybridStep] = []
    
    def add_workflow_step(self, name: str, func: Callable):
        """添加确定性步骤"""
        self.steps.append(HybridStep(name, StepType.WORKFLOW, func=func))
    
    def add_agent_step(self, name: str, agent_func: Callable):
        """添加智能决策步骤"""
        self.steps.append(HybridStep(name, StepType.AGENT, agent_func=agent_func))
    
    def execute(self, initial_context: Dict) -> Dict:
        """执行混合流水线"""
        context = initial_context
        execution_log = []
        
        for step in self.steps:
            start_time = time.time()
            
            try:
                context = step.execute(context)
                duration_ms = (time.time() - start_time) * 1000
                
                execution_log.append({
                    "step": step.name,
                    "type": step.step_type.value,
                    "status": "success",
                    "duration_ms": duration_ms,
                })
            except Exception as e:
                execution_log.append({
                    "step": step.name,
                    "type": step.step_type.value,
                    "status": "error",
                    "error": str(e),
                })
                context["error"] = str(e)
                break
        
        context["execution_log"] = execution_log
        return context


# ============================================================
# 实际应用示例
# ============================================================

def demo_hybrid_pipeline():
    """演示混合流水线"""
    
    # 定义步骤函数
    def validate_input(context: Dict) -> Dict:
        """步骤1: 输入验证（Workflow）"""
        user_input = context.get("user_input", "")
        if len(user_input) < 1:
            raise ValueError("输入不能为空")
        if len(user_input) > 4000:
            raise ValueError("输入过长")
        context["validated"] = True
        return context
    
    def classify_intent(context: Dict) -> Dict:
        """步骤2: 意图分类（Agent）"""
        user_input = context.get("user_input", "")
        # 模拟 LLM 分类
        if "价格" in user_input or "多少钱" in user_input:
            context["intent"] = "price_inquiry"
        elif "投诉" in user_input or "不满" in user_input:
            context["intent"] = "complaint"
        else:
            context["intent"] = "general"
        return context
    
    def route_by_intent(context: Dict) -> Dict:
        """步骤3: 路由（Workflow）"""
        intent = context.get("intent", "general")
        routing_map = {
            "price_inquiry": "price_handler",
            "complaint": "complaint_handler",
            "general": "general_handler",
        }
        context["handler"] = routing_map.get(intent, "general_handler")
        return context
    
    def generate_response(context: Dict) -> Dict:
        """步骤4: 生成回答（Agent）"""
        intent = context.get("intent", "general")
        user_input = context.get("user_input", "")
        
        responses = {
            "price_inquiry": f"关于'{user_input[:20]}'的价格信息...",
            "complaint": f"非常抱歉给您带来不便，关于'{user_input[:20]}'...",
            "general": f"感谢您的咨询，关于'{user_input[:20]}'...",
        }
        context["response"] = responses.get(intent, "感谢您的咨询。")
        return context
    
    def format_output(context: Dict) -> Dict:
        """步骤5: 格式化输出（Workflow）"""
        response = context.get("response", "")
        context["formatted_response"] = f"[{context.get('intent', 'unknown')}] {response}"
        return context
    
    # 构建混合流水线
    pipeline = HybridPipeline("客服处理流水线")
    pipeline.add_workflow_step("输入验证", validate_input)
    pipeline.add_agent_step("意图分类", classify_intent)
    pipeline.add_workflow_step("路由分发", route_by_intent)
    pipeline.add_agent_step("回答生成", generate_response)
    pipeline.add_workflow_step("格式化输出", format_output)
    
    # 测试
    test_cases = [
        "这个产品多少钱？",
        "我对服务很不满！",
        "你好，我想了解一下你们的服务",
    ]
    
    print("混合流水线测试")
    print("=" * 60)
    
    for user_input in test_cases:
        print(f"\n用户: {user_input}")
        result = pipeline.execute({"user_input": user_input})
        print(f"意图: {result.get('intent')}")
        print(f"处理器: {result.get('handler')}")
        print(f"回答: {result.get('formatted_response')}")
        
        # 打印执行日志
        print("执行路径:")
        for log in result.get("execution_log", []):
            type_label = "WF" if log["type"] == "workflow" else "AI"
            print(f"  [{type_label}] {log['step']}: {log['status']} ({log['duration_ms']:.0f}ms)")


if __name__ == "__main__":
    demo_hybrid_pipeline()
```

---

## 44.4 迁移策略

### 44.4.1 渐进式迁移框架

```python
class MigrationStrategy:
    """
    渐进式迁移策略
    
    从纯 Workflow 到混合方案，再到纯 Agent 的渐进式迁移路径。
    """
    
    def __init__(self):
        self.phases = [
            {
                "name": "阶段一：识别机会",
                "description": "分析现有 Workflow，找出适合 Agent 化的环节",
                "duration": "1-2 周",
                "actions": [
                    "梳理所有 Workflow 步骤",
                    "标注哪些步骤需要动态决策",
                    "评估 Agent 化的预期收益和风险",
                ],
            },
            {
                "name": "阶段二：试点 Agent",
                "description": "在一个非关键 Workflow 中引入 Agent 步骤",
                "duration": "2-4 周",
                "actions": [
                    "选择一个低风险的 Workflow",
                    "将其中最需要灵活性的步骤替换为 Agent",
                    "建立评估体系，对比前后效果",
                ],
            },
            {
                "name": "阶段三：混合架构",
                "description": "在更多 Workflow 中引入 Agent，建立混合架构",
                "duration": "1-2 月",
                "actions": [
                    "为 Agent 步骤建立监控和告警",
                    "实现 Workflow-Agent 的无缝集成",
                    "建立 Agent 的评估和迭代流程",
                ],
            },
            {
                "name": "阶段四：全面评估",
                "description": "评估是否需要进一步 Agent 化",
                "duration": "持续",
                "actions": [
                    "收集生产数据，分析 Agent 的表现",
                    "评估成本和收益",
                    "决定哪些环节保留 Workflow，哪些升级为 Agent",
                ],
            },
        ]
    
    def analyze_migration_readiness(self, workflow_steps: List[Dict]) -> Dict:
        """分析迁移就绪度"""
        total_steps = len(workflow_steps)
        agent_candidates = []
        
        for step in workflow_steps:
            score = 0
            reasons = []
            
            if step.get("input_variability") == "high":
                score += 2
                reasons.append("输入变异性高")
            if step.get("needs_judgment"):
                score += 3
                reasons.append("需要判断能力")
            if step.get("error_handling_complex"):
                score += 2
                reasons.append("错误处理复杂")
            if step.get("frequency") == "low":
                score += 1
                reasons.append("执行频率低")
            
            agent_candidates.append({
                "step": step["name"],
                "score": score,
                "reasons": reasons,
                "recommended": score >= 3,
            })
        
        return {
            "total_steps": total_steps,
            "agent_candidates": agent_candidates,
            "readiness_score": sum(1 for c in agent_candidates if c["recommended"]) / total_steps,
        }
    
    def print_strategy(self):
        """打印迁移策略"""
        print("渐进式迁移策略")
        print("=" * 60)
        
        for phase in self.phases:
            print(f"\n{phase['name']} ({phase['duration']})")
            print(f"  {phase['description']}")
            print(f"  行动项:")
            for action in phase["actions"]:
                print(f"    - {action}")


# ============================================================
# 使用示例
# ============================================================

def demo_migration():
    """演示迁移策略"""
    strategy = MigrationStrategy()
    strategy.print_strategy()
    
    # 分析迁移就绪度
    workflow_steps = [
        {"name": "输入验证", "input_variability": "low", "needs_judgment": False},
        {"name": "数据清洗", "input_variability": "medium", "needs_judgment": False},
        {"name": "内容理解", "input_variability": "high", "needs_judgment": True},
        {"name": "决策判断", "input_variability": "high", "needs_judgment": True,
         "error_handling_complex": True},
        {"name": "结果格式化", "input_variability": "low", "needs_judgment": False},
    ]
    
    readiness = strategy.analyze_migration_readiness(workflow_steps)
    
    print(f"\n\n迁移就绪度分析")
    print("=" * 60)
    print(f"总步骤数: {readiness['total_steps']}")
    print(f"就绪度评分: {readiness['readiness_score']:.1%}")
    print(f"\nAgent 候选步骤:")
    for candidate in readiness["agent_candidates"]:
        status = "推荐 Agent 化" if candidate["recommended"] else "保留 Workflow"
        print(f"  {candidate['step']}: {status} (评分: {candidate['score']})")
        for reason in candidate["reasons"]:
            print(f"    - {reason}")


if __name__ == "__main__":
    demo_migration()
```

---

## 44.5 案例分析

### 44.5.1 案例：电商客服从 Workflow 到 Agent

**背景：** 某电商的客服系统是一个纯 Workflow：用户提交工单 -> 自动分类 -> 路由到对应部门 -> 人工处理。分类准确率只有 70%，经常把投诉误分类为咨询。

**迁移决策：** 使用选型框架分析后，"工单分类"步骤的得分是：Workflow 0.35, Agent 0.45, 混合 0.50。推荐使用混合方案：保留 Workflow 的路由和分发逻辑，但将分类步骤替换为 Agent。

**迁移过程：** 第一阶段用 LLM 做分类试点，准确率提升到 88%。第二阶段添加了工具调用能力（查询订单、搜索知识库），让 Agent 能直接回答简单问题，减少人工介入。第三阶段建立了评估和监控体系。

**结果：** 人工处理量减少 45%，分类准确率从 70% 提升到 92%，平均响应时间从 5 分钟降到 30 秒。但运营成本增加了 20%（主要是 LLM API 费用）。总体 ROI 为正。

**关键教训：** 迁移不是"全有或全无"。找到最需要灵活性的环节，用 Agent 替换它，其余保留 Workflow。这样既能获得 Agent 的灵活性，又能保持 Workflow 的可控性。

---

## 44.6 高级选型技术

### 44.6.1 实时选型决策器

在运行时根据任务特征动态选择 Workflow 或 Agent。

```python
import time
from typing import Dict, List, Any, Callable, Optional
from dataclasses import dataclass, field
from enum import Enum

class ExecutionMode(Enum):
    """执行模式"""
    WORKFLOW = "workflow"
    AGENT = "agent"
    HYBRID = "hybrid"

@dataclass
class TaskFeatures:
    """任务特征"""
    input_length: int
    has_structured_output: bool
    requires_tool_use: bool
    complexity_score: float  # 0-1
    urgency_level: str  # low/medium/high
    error_tolerance: str  # low/medium/high

@dataclass
class ExecutionDecision:
    """执行决策"""
    mode: ExecutionMode
    confidence: float
    reasons: List[str]
    estimated_cost: float
    estimated_latency_ms: float

class RuntimeModeSelector:
    """
    运行时模式选择器
    
    在运行时根据任务特征动态选择执行模式。
    """
    
    def __init__(self):
        # 模式特征矩阵
        self.mode_profiles = {
            ExecutionMode.WORKFLOW: {
                "max_complexity": 0.4,
                "min_structured_output": True,
                "min_error_tolerance": "low",
                "cost_factor": 0.3,
                "latency_factor": 0.5,
            },
            ExecutionMode.AGENT: {
                "min_complexity": 0.6,
                "min_structured_output": False,
                "min_error_tolerance": "medium",
                "cost_factor": 1.0,
                "latency_factor": 1.0,
            },
            ExecutionMode.HYBRID: {
                "min_complexity": 0.3,
                "max_complexity": 0.7,
                "cost_factor": 0.6,
                "latency_factor": 0.7,
            },
        }
        
        # 历史决策记录
        self.decision_history: List[Dict] = []
    
    def select_mode(self, features: TaskFeatures) -> ExecutionDecision:
        """
        根据任务特征选择执行模式
        
        Args:
            features: 任务特征
            
        Returns:
            执行决策
        """
        scores = {}
        reasons = {}
        
        # 评估 Workflow 适合度
        wf_score = 1.0
        wf_reasons = []
        
        if features.complexity_score < 0.4:
            wf_score *= 1.2
            wf_reasons.append("低复杂度任务适合 Workflow")
        
        if features.has_structured_output:
            wf_score *= 1.1
            wf_reasons.append("需要结构化输出")
        
        if features.error_tolerance == "low":
            wf_score *= 1.15
            wf_reasons.append("低错误容忍度")
        
        if features.requires_tool_use:
            wf_score *= 0.8
            wf_reasons.append("需要工具调用，降低 Workflow 分数")
        
        scores[ExecutionMode.WORKFLOW] = wf_score
        reasons[ExecutionMode.WORKFLOW] = wf_reasons
        
        # 评估 Agent 适合度
        agent_score = 1.0
        agent_reasons = []
        
        if features.complexity_score > 0.6:
            agent_score *= 1.2
            agent_reasons.append("高复杂度任务需要 Agent")
        
        if not features.has_structured_output:
            agent_score *= 1.1
            agent_reasons.append("开放式输出适合 Agent")
        
        if features.error_tolerance in ["medium", "high"]:
            agent_score *= 1.1
            agent_reasons.append("较高的错误容忍度")
        
        if features.requires_tool_use:
            agent_score *= 1.15
            agent_reasons.append("需要工具调用")
        
        scores[ExecutionMode.AGENT] = agent_score
        reasons[ExecutionMode.AGENT] = agent_reasons
        
        # 评估混合模式适合度
        hybrid_score = 0.8
        hybrid_reasons = ["混合模式提供灵活性和可控性的平衡"]
        
        if 0.3 <= features.complexity_score <= 0.7:
            hybrid_score *= 1.15
            hybrid_reasons.append("中等复杂度适合混合模式")
        
        scores[ExecutionMode.HYBRID] = hybrid_score
        reasons[ExecutionMode.HYBRID] = hybrid_reasons
        
        # 选择最高分的模式
        best_mode = max(scores, key=scores.get)
        total_score = sum(scores.values())
        confidence = scores[best_mode] / total_score
        
        # 估算成本和延迟
        profile = self.mode_profiles[best_mode]
        estimated_cost = self._estimate_cost(features, profile["cost_factor"])
        estimated_latency = self._estimate_latency(features, profile["latency_factor"])
        
        decision = ExecutionDecision(
            mode=best_mode,
            confidence=confidence,
            reasons=reasons[best_mode],
            estimated_cost=estimated_cost,
            estimated_latency_ms=estimated_latency,
        )
        
        # 记录决策
        self.decision_history.append({
            "timestamp": time.time(),
            "features": {
                "complexity": features.complexity_score,
                "input_length": features.input_length,
                "tool_use": features.requires_tool_use,
            },
            "decision": best_mode.value,
            "confidence": confidence,
        })
        
        return decision
    
    def _estimate_cost(self, features: TaskFeatures, cost_factor: float) -> float:
        """估算成本"""
        base_cost = 0.001  # 基础成本
        complexity_cost = features.complexity_score * 0.01
        length_cost = features.input_length / 10000 * 0.005
        
        return (base_cost + complexity_cost + length_cost) * cost_factor
    
    def _estimate_latency(self, features: TaskFeatures, latency_factor: float) -> float:
        """估算延迟"""
        base_latency = 500  # 基础延迟 ms
        complexity_latency = features.complexity_score * 1000
        
        return (base_latency + complexity_latency) * latency_factor
    
    def get_statistics(self) -> Dict:
        """获取决策统计"""
        if not self.decision_history:
            return {"total_decisions": 0}
        
        mode_counts = {}
        confidences = []
        
        for record in self.decision_history:
            mode = record["decision"]
            mode_counts[mode] = mode_counts.get(mode, 0) + 1
            confidences.append(record["confidence"])
        
        return {
            "total_decisions": len(self.decision_history),
            "mode_distribution": mode_counts,
            "avg_confidence": sum(confidences) / len(confidences),
        }


# ============================================================
# 使用示例
# ============================================================

def demo_runtime_mode_selection():
    """演示运行时模式选择"""
    selector = RuntimeModeSelector()
    
    print("运行时模式选择演示")
    print("=" * 60)
    
    # 测试不同的任务特征
    test_tasks = [
        TaskFeatures(
            input_length=50,
            has_structured_output=True,
            requires_tool_use=False,
            complexity_score=0.2,
            urgency_level="high",
            error_tolerance="low",
        ),
        TaskFeatures(
            input_length=500,
            has_structured_output=False,
            requires_tool_use=True,
            complexity_score=0.8,
            urgency_level="medium",
            error_tolerance="medium",
        ),
        TaskFeatures(
            input_length=200,
            has_structured_output=False,
            requires_tool_use=True,
            complexity_score=0.5,
            urgency_level="medium",
            error_tolerance="medium",
        ),
    ]
    
    task_descriptions = [
        "简单的数据格式化任务",
        "复杂的多步骤推理任务",
        "中等复杂度的问答任务",
    ]
    
    for task, desc in zip(test_tasks, task_descriptions):
        print(f"\n任务: {desc}")
        print(f"  复杂度: {task.complexity_score}, 工具调用: {task.requires_tool_use}")
        
        decision = selector.select_mode(task)
        
        print(f"  推荐模式: {decision.mode.value}")
        print(f"  置信度: {decision.confidence:.2%}")
        print(f"  预估成本: ${decision.estimated_cost:.6f}")
        print(f"  预估延迟: {decision.estimated_latency_ms:.0f}ms")
        print(f"  理由: {', '.join(decision.reasons)}")
    
    # 打印统计
    print("\n决策统计:")
    stats = selector.get_statistics()
    print(f"  总决策数: {stats['total_decisions']}")
    print(f"  模式分布: {stats['mode_distribution']}")
    print(f"  平均置信度: {stats['avg_confidence']:.2%}")


if __name__ == "__main__":
    demo_runtime_mode_selection()
```

### 44.6.2 选型决策评估框架

系统性地评估和比较不同选型方案的效果。

```python
from typing import Dict, List, Any
from dataclasses import dataclass, field

@dataclass
class EvaluationCriteria:
    """评估标准"""
    name: str
    weight: float  # 权重 0-1
    description: str

@dataclass
class EvaluationResult:
    """评估结果"""
    criteria: str
    score: float  # 0-10
    weight: float
    weighted_score: float
    evidence: str

class SelectionEvaluator:
    """
    选型评估器
    
    系统性地评估不同选型方案的效果。
    """
    
    def __init__(self):
        self.criteria = [
            EvaluationCriteria("accuracy", 0.25, "任务完成准确率"),
            EvaluationCriteria("latency", 0.20, "响应延迟"),
            EvaluationCriteria("cost", 0.20, "运营成本"),
            EvaluationCriteria("maintainability", 0.15, "可维护性"),
            EvaluationCriteria("scalability", 0.10, "可扩展性"),
            EvaluationCriteria("reliability", 0.10, "可靠性"),
        ]
        
        self.evaluation_history: List[Dict] = []
    
    def evaluate(self, solution_name: str, 
                metrics: Dict[str, float]) -> Dict:
        """
        评估一个方案
        
        Args:
            solution_name: 方案名称
            metrics: 各项指标得分 (0-10)
            
        Returns:
            评估结果
        """
        results = []
        total_weighted_score = 0
        
        for criterion in self.criteria:
            score = metrics.get(criterion.name, 5.0)  # 默认 5 分
            weighted_score = score * criterion.weight
            total_weighted_score += weighted_score
            
            results.append(EvaluationResult(
                criteria=criterion.name,
                score=score,
                weight=criterion.weight,
                weighted_score=weighted_score,
                evidence=f"得分: {score}/10",
            ))
        
        evaluation = {
            "solution": solution_name,
            "total_score": total_weighted_score,
            "max_possible": sum(c.weight * 10 for c in self.criteria),
            "score_percentage": total_weighted_score / sum(c.weight * 10 for c in self.criteria) * 100,
            "details": results,
        }
        
        self.evaluation_history.append(evaluation)
        return evaluation
    
    def compare_solutions(self, evaluations: List[Dict]) -> Dict:
        """
        比较多个方案
        
        Args:
            evaluations: 评估结果列表
            
        Returns:
            比较结果
        """
        if not evaluations:
            return {"error": "没有可比较的方案"}
        
        # 按总分排序
        sorted_evals = sorted(evaluations, key=lambda x: x["total_score"], reverse=True)
        
        best = sorted_evals[0]
        
        # 计算各项指标的最佳方案
        criteria_best = {}
        for criterion in self.criteria:
            best_score = -1
            best_solution = None
            
            for evaluation in evaluations:
                for detail in evaluation["details"]:
                    if detail.criteria == criterion.name and detail.score > best_score:
                        best_score = detail.score
                        best_solution = evaluation["solution"]
            
            criteria_best[criterion.name] = {
                "best_solution": best_solution,
                "score": best_score,
            }
        
        return {
            "ranking": [
                {
                    "rank": i + 1,
                    "solution": e["solution"],
                    "total_score": e["total_score"],
                    "score_percentage": e["score_percentage"],
                }
                for i, e in enumerate(sorted_evals)
            ],
            "winner": best["solution"],
            "winner_score": best["score_percentage"],
            "criteria_best": criteria_best,
        }
    
    def generate_recommendation(self, comparison: Dict) -> str:
        """生成推荐建议"""
        winner = comparison["winner"]
        score = comparison["winner_score"]
        
        if score >= 80:
            confidence = "强烈推荐"
        elif score >= 60:
            confidence = "推荐"
        elif score >= 40:
            confidence = "可以考虑"
        else:
            confidence = "不推荐"
        
        recommendation = f"推荐使用 {winner} 方案（得分: {score:.1f}%，{confidence}）"
        
        # 添加改进意见
        ranking = comparison["ranking"]
        if len(ranking) > 1:
            second = ranking[1]
            gap = score - second["score_percentage"]
            if gap < 10:
                recommendation += f"。与第二名 {second['solution']} 差距较小（{gap:.1f}%），需综合考虑其他因素。"
        
        return recommendation


# ============================================================
# 使用示例
# ============================================================

def demo_evaluation_framework():
    """演示评估框架"""
    evaluator = SelectionEvaluator()
    
    print("选型评估框架演示")
    print("=" * 60)
    
    # 评估不同方案
    workflow_eval = evaluator.evaluate(
        "Workflow 方案",
        {
            "accuracy": 9.0,
            "latency": 8.0,
            "cost": 9.0,
            "maintainability": 8.5,
            "scalability": 7.0,
            "reliability": 9.0,
        }
    )
    
    agent_eval = evaluator.evaluate(
        "Agent 方案",
        {
            "accuracy": 7.5,
            "latency": 6.0,
            "cost": 5.0,
            "maintainability": 6.0,
            "scalability": 8.0,
            "reliability": 7.0,
        }
    )
    
    hybrid_eval = evaluator.evaluate(
        "混合方案",
        {
            "accuracy": 8.5,
            "latency": 7.0,
            "cost": 7.0,
            "maintainability": 7.0,
            "scalability": 7.5,
            "reliability": 8.0,
        }
    )
    
    # 比较方案
    comparison = evaluator.compare_solutions([workflow_eval, agent_eval, hybrid_eval])
    
    print("\n方案排名:")
    for item in comparison["ranking"]:
        print(f"  #{item['rank']} {item['solution']}: {item['score_percentage']:.1f}%")
    
    print(f"\n最佳方案: {comparison['winner']}")
    
    print("\n各项指标最佳:")
    for criteria, info in comparison["criteria_best"].items():
        print(f"  {criteria}: {info['best_solution']} ({info['score']:.1f})")
    
    # 生成推荐
    recommendation = evaluator.generate_recommendation(comparison)
    print(f"\n推荐建议: {recommendation}")
    
    # 打印详细评估
    print("\n详细评估 (Workflow 方案):")
    for detail in workflow_eval["details"]:
        print(f"  {detail.criteria}: {detail.score}/10 (权重: {detail.weight})")


if __name__ == "__main__":
    demo_evaluation_framework()
```

### 44.6.3 渐进式迁移路径规划

```python
from typing import Dict, List
from dataclasses import dataclass, field
from enum import Enum

class MigrationPhase(Enum):
    """迁移阶段"""
    ASSESSMENT = "assessment"         # 评估
    PILOT = "pilot"                   # 试点
    INCREMENTAL = "incremental"       # 增量迁移
    OPTIMIZATION = "optimization"     # 优化
    COMPLETION = "completion"         # 完成

@dataclass
class MigrationStep:
    """迁移步骤"""
    phase: MigrationPhase
    name: str
    description: str
    duration: str
    risks: List[str]
    success_criteria: List[str]
    dependencies: List[str] = field(default_factory=list)

class MigrationPlanner:
    """
    迁移规划器
    
    规划从 Workflow 到 Agent 的渐进式迁移路径。
    """
    
    def __init__(self):
        self.steps: List[MigrationStep] = []
        self._create_migration_plan()
    
    def _create_migration_plan(self):
        """创建迁移计划"""
        self.steps = [
            MigrationStep(
                phase=MigrationPhase.ASSESSMENT,
                name="现状评估",
                description="全面评估现有 Workflow 系统，识别适合 Agent 化的环节",
                duration="1-2 周",
                risks=["评估不全面", "遗漏关键环节"],
                success_criteria=[
                    "完成所有 Workflow 步骤的分析",
                    "识别出至少 3 个 Agent 候选环节",
                    "建立评估基线",
                ],
            ),
            MigrationStep(
                phase=MigrationPhase.ASSESSMENT,
                name="技术选型",
                description="选择 LLM 提供商、框架和技术栈",
                duration="1 周",
                risks=["技术选型不当", "成本预估不准确"],
                success_criteria=[
                    "完成技术 POC",
                    "确认成本模型",
                    "团队完成技术培训",
                ],
            ),
            MigrationStep(
                phase=MigrationPhase.PILOT,
                name="试点开发",
                description="在一个低风险环节开发 Agent 原型",
                duration="2-4 周",
                risks=["原型质量不达标", "团队经验不足"],
                success_criteria=[
                    "Agent 原型完成开发",
                    "准确率达到基线的 90%",
                    "延迟在可接受范围内",
                ],
                dependencies=["现状评估", "技术选型"],
            ),
            MigrationStep(
                phase=MigrationPhase.PILOT,
                name="试点验证",
                description="在生产环境小流量验证 Agent 效果",
                duration="2 周",
                risks=["用户反馈不佳", "出现意外问题"],
                success_criteria=[
                    "用户满意度不低于基线",
                    "错误率低于 5%",
                    "成本在预算内",
                ],
                dependencies=["试点开发"],
            ),
            MigrationStep(
                phase=MigrationPhase.INCREMENTAL,
                name="增量迁移",
                description="逐步将更多环节迁移到 Agent",
                duration="1-3 月",
                risks=["迁移过程影响稳定性", "团队负担过重"],
                success_criteria=[
                    "完成 50% 以上的环节迁移",
                    "整体指标不低于基线",
                    "建立完整的监控体系",
                ],
                dependencies=["试点验证"],
            ),
            MigrationStep(
                phase=MigrationPhase.OPTIMIZATION,
                name="性能优化",
                description="优化 Agent 性能和成本",
                duration="2-4 周",
                risks=["优化过度影响质量"],
                success_criteria=[
                    "延迟降低 20%",
                    "成本降低 30%",
                    "质量保持稳定",
                ],
                dependencies=["增量迁移"],
            ),
            MigrationStep(
                phase=MigrationPhase.COMPLETION,
                name="全面切换",
                description="完成所有迁移，关闭旧 Workflow",
                duration="1-2 周",
                risks=["回滚困难"],
                success_criteria=[
                    "所有环节完成迁移",
                    "旧系统成功下线",
                    "文档和培训完成",
                ],
                dependencies=["性能优化"],
            ),
        ]
    
    def get_plan(self) -> Dict:
        """获取完整迁移计划"""
        phases = {}
        for step in self.steps:
            phase = step.phase.value
            if phase not in phases:
                phases[phase] = []
            phases[phase].append({
                "name": step.name,
                "description": step.description,
                "duration": step.duration,
                "risks": step.risks,
                "success_criteria": step.success_criteria,
                "dependencies": step.dependencies,
            })
        
        return {
            "total_phases": len(phases),
            "total_steps": len(self.steps),
            "phases": phases,
        }
    
    def get_critical_path(self) -> List[str]:
        """获取关键路径"""
        # 简化版：返回有依赖关系的步骤
        critical = []
        for step in self.steps:
            if step.dependencies:
                critical.append(step.name)
        return critical
    
    def estimate_timeline(self) -> Dict:
        """估算时间线"""
        total_weeks = 0
        phase_durations = {}
        
        for step in self.steps:
            # 解析持续时间
            duration = step.duration
            if "月" in duration:
                weeks = int(duration.split("-")[-1].replace("月", "")) * 4
            elif "周" in duration:
                weeks = int(duration.split("-")[-1].replace("周", ""))
            else:
                weeks = 1
            
            phase = step.phase.value
            phase_durations[phase] = phase_durations.get(phase, 0) + weeks
            total_weeks += weeks
        
        return {
            "total_weeks": total_weeks,
            "total_months": total_weeks / 4,
            "phase_durations": phase_durations,
        }


# ============================================================
# 使用示例
# ============================================================

def demo_migration_planning():
    """演示迁移规划"""
    planner = MigrationPlanner()
    
    print("渐进式迁移规划")
    print("=" * 60)
    
    # 获取计划
    plan = planner.get_plan()
    
    print(f"\n总阶段数: {plan['total_phases']}")
    print(f"总步骤数: {plan['total_steps']}")
    
    for phase_name, steps in plan["phases"].items():
        print(f"\n阶段: {phase_name}")
        for step in steps:
            print(f"  - {step['name']}: {step['description']}")
            print(f"    持续时间: {step['duration']}")
            if step['dependencies']:
                print(f"    依赖: {', '.join(step['dependencies'])}")
    
    # 时间线估算
    timeline = planner.estimate_timeline()
    print(f"\n时间线估算:")
    print(f"  总周数: {timeline['total_weeks']}")
    print(f"  总月数: {timeline['total_months']:.1f}")
    print(f"  各阶段时间: {timeline['phase_durations']}")
    
    # 关键路径
    critical_path = planner.get_critical_path()
    print(f"\n关键路径: {' -> '.join(critical_path)}")


if __name__ == "__main__":
    demo_migration_planning()
```

## 44.7 常见坑与最佳实践

**坑 1：过度 Agent 化。** 把所有步骤都变成 Agent，结果系统变得不可预测、难以调试、成本飙升。只在真正需要灵活性的地方使用 Agent。

**坑 2：忽略 Workflow 的优势。** Workflow 的确定性、可测试性、低成本是巨大的优势。不要因为"Agent 很酷"就放弃这些优势。

**坑 3：没有建立评估体系就开始迁移。** 在迁移之前就建立好评估标准（准确率、延迟、成本），否则你无法判断迁移是否成功。

**坑 4：一次性大爆炸式迁移。** 试图一次性将所有 Workflow 迁移到 Agent，风险极高。采用渐进式迁移，逐步验证和调整。

**坑 5：忽略团队能力。** Agent 开发需要新的技能（Prompt 工程、LLM 调试等）。确保团队有足够的能力和培训。

**最佳实践 1：从最小的改动开始。** 先在一个小的、低风险的环节引入 Agent，验证效果后再扩展。渐进式迁移比大爆炸式迁移安全得多。

**最佳实践 2：保持可回滚能力。** 在迁移过程中，始终保持回滚到 Workflow 的能力。当 Agent 出现问题时，能快速切换回 Workflow。

**最佳实践 3：数据驱动决策。** 每次迁移决策都应该有数据支撑。通过 A/B 测试对比 Agent 和 Workflow 的效果。

**最佳实践 4：持续优化。** 迁移完成后，持续监控 Agent 的表现，寻找优化机会。技术在发展，最佳实践也在演变。

---

## 44.7 练习题

**练习 1：设计决策矩阵。** 为以下场景设计选型分析：(1) 自动化测试执行；(2) 智能日志分析；(3) 数据报表生成；(4) 代码生成。

**练习 2：构建混合流水线。** 设计一个"智能数据处理"混合流水线：数据验证（Workflow）-> 异常检测（Agent）-> 数据修复（Agent）-> 格式化输出（Workflow）。

**练习 3：评估迁移 ROI。** 假设你有一个每天处理 5000 个工单的 Workflow 系统。将分类步骤替换为 Agent 后，准确率从 70% 提升到 90%，但每次分类成本从 $0.001 增加到 $0.01。计算迁移的投资回报率。

**练习 4：设计降级方案。** 当 Agent 步骤失败时（如 LLM API 超时），设计自动降级到 Workflow 的方案。

**练习 5：实现 Workflow-Agent 切换器。** 设计一个开关，允许在运行时动态切换某个步骤是用 Workflow 还是 Agent 实现，便于 A/B 测试。

**练习 6：分析你的项目。** 选择你正在做的一个项目，用本章的选型框架分析每个步骤，判断哪些适合 Workflow，哪些适合 Agent，哪些适合混合方案。

---

## 44.8 本章小结

本章建立了一个系统性的 Workflow vs Agent 选型决策框架。核心观点是：Workflow 和 Agent 不是对立的，而是互补的。Workflow 提供确定性、可控性和低成本；Agent 提供灵活性、智能和适应性。

在实际项目中，大多数系统最终都会是混合架构——用 Workflow 处理确定性的步骤，用 Agent 处理需要智能决策的步骤。关键是找到正确的边界：在哪里从 Workflow 切换到 Agent，以及如何让两者无缝协作。

记住：技术选型不是目的，解决问题才是。选择最能解决问题的方案，而不是最"先进"的方案。

---

> **恭喜你完成了第 31-44 章的学习！** 这些章节覆盖了 Agent 工程化的关键话题：从可观测性到安全、从测试到部署、从成本优化到选型决策。掌握了这些知识，你就具备了将 Agent 从原型推向生产环境的完整能力。
