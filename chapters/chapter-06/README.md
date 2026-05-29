# 第 6 章：Agent 的分类与适用场景

> **本章定位：** 不是所有问题都需要 Agent。理解 Agent 的分类和适用场景，才能在实际项目中做出正确的技术选型。错误的技术选型会导致项目失败——用 Agent 处理简单任务是浪费资源，用 Workflow 处理复杂任务则无法满足需求。本章将带你深入分析 Agent 的分类体系，掌握 Agent vs Workflow 的决策框架，以及如何设计混合架构。

---

## 学习目标

完成本章学习后，你将能够：

1. **掌握 Agent 的主要分类** —— 能区分 Reactive、Deliberative、Hybrid 三种架构，理解每种架构的原理、优势和局限
2. **理解 Agent 适用和不适用的场景** —— 能判断什么时候应该用 Agent，什么时候不应该用
3. **掌握 Agent vs Workflow 的决策框架** —— 能从五个维度评估任务，做出正确的技术选型
4. **了解行业中的 Agent 应用案例** —— 理解 Agent 在客服、数据分析、软件开发、金融等领域的应用
5. **能设计混合架构** —— 理解如何结合 Agent 和 Workflow 的优势

## 核心问题

1. **什么时候应该用 Agent？什么时候应该用 Workflow？** 这是架构师的核心判断能力。错误的选型会导致项目失败。
2. **不同类型的 Agent 适用于什么场景？** Reactive Agent 和 Deliberative Agent 各自的优势和局限是什么？
3. **如何评估一个任务是否适合用 Agent 解决？** 有没有一个系统化的决策框架？

---

## 6.1 为什么选型如此重要

### 6.1.1 一个真实的失败案例

某公司想构建一个"智能客服系统"。他们的第一反应是："现在 Agent 很火，我们用 Agent 吧！"

于是他们用 Agent 构建了整个客服系统：

```
用户："你们的退货政策是什么？"
Agent：（思考）用户在问退货政策，我需要搜索相关信息...
Agent：（行动）[search_web(query="退货政策")]
Agent：（观察）找到了一些信息...
Agent：（思考）我来整理一下...
Agent：（回答）你们的退货政策是...
```

**问题：**

1. **成本高：** 每个简单问题都需要多次 LLM 调用，成本是普通系统的 10 倍
2. **延迟大：** 用户等待 3-5 秒才能得到回答，而普通系统只需要 0.5 秒
3. **不稳定：** Agent 有时会"想太多"，给出不必要的复杂回答

**正确的做法：**

```
标准问题（80%）→ Workflow 自动回复（快速、便宜、稳定）
复杂问题（20%）→ Agent 处理（灵活、智能）
```

### 6.1.2 选型的核心原则

**原则 1：简单问题用简单方案**

如果问题可以用固定的步骤解决，就不要用 Agent。

**原则 2：复杂问题用合适方案**

如果问题需要动态决策、推理、多步规划，才考虑 Agent。

**原则 3：混合架构往往是最优解**

大多数实际系统都是 Agent + Workflow 的混合架构。

---

## 6.2 Agent 的分类

### 6.2.1 按架构分类

**Reactive Agent（反应式 Agent）**

Reactive Agent 只响应当前输入，没有记忆和规划能力。它是最简单的 Agent 类型。

```
输入 → LLM → 输出
（没有循环，没有状态管理）
```

**工作原理：**

```
用户输入 → LLM 分析 → 选择行动 → 执行 → 输出
```

特点：
- **简单：** 架构简单，易于实现
- **快速：** 只需要一次 LLM 调用
- **无状态：** 没有记忆，每次都是独立的
- **无规划：** 不能做多步规划

**适用场景：**
- 简单问答
- 文本分类
- 情感分析
- 简单的工具调用

**不适用场景：**
- 需要多轮对话
- 需要记忆
- 需要规划
- 复杂任务

**示例代码：**

```python
class ReactiveAgent:
    def __init__(self, llm_client):
        self.llm = llm_client

    def run(self, user_input: str) -> str:
        """简单的输入-输出，没有循环"""
        response = self.llm.chat(
            messages=[{"role": "user", "content": user_input}]
        )
        return response
```

**Deliberative Agent（审议式 Agent）**

Deliberative Agent 有规划和推理能力，能处理复杂任务。它是 Agent 的"完整形态"。

```
输入 → [循环] → 规划 → 执行 → 反馈 → ... → 输出
（有循环，有状态管理，有规划）
```

**工作原理：**

```
用户输入 → [循环开始]
    → 观察当前状态
    → LLM 推理（Thought）
    → 决定行动（Action）
    → 执行行动
    → 获取反馈（Observation）
    → 更新状态
    → 检查是否完成
[循环结束] → 输出最终结果
```

特点：
- **有状态：** 能记住之前的对话和行动
- **有规划：** 能将复杂任务分解为子任务
- **有推理：** 能进行逻辑推理和决策
- **能自我修正：** 能从错误中学习

**适用场景：**
- 复杂任务（研究、分析、写作）
- 多轮对话
- 需要规划的任务
- 需要工具调用的任务

**不适用场景：**
- 简单问答（杀鸡用牛刀）
- 对延迟敏感的场景（多次 LLM 调用导致延迟）
- 对成本敏感的场景（多次 LLM 调用导致成本高）

**示例代码：**

```python
class DeliberativeAgent:
    def __init__(self, llm_client, tools: dict, max_iterations: int = 10):
        self.llm = llm_client
        self.tools = tools
        self.max_iterations = max_iterations
        self.state = {"messages": [], "task_state": {}}

    def run(self, user_input: str) -> str:
        """有循环、有状态、有规划"""
        self.state["messages"].append({"role": "user", "content": user_input})

        for iteration in range(self.max_iterations):
            # 观察
            observation = self._observe()

            # 推理
            thought = self._think(observation)

            # 行动
            action = self._decide_action(thought)

            if action["type"] == "answer":
                return action["content"]

            # 执行
            result = self._execute(action)

            # 反馈
            self._update_state(result)

        return "任务未完成"
```

**Hybrid Agent（混合式 Agent）**

Hybrid Agent 结合了 Reactive 和 Deliberative 的特点，根据任务复杂度选择不同的处理路径。

```
简单任务 → Reactive 路径（快速响应）
复杂任务 → Deliberative 路径（规划执行）
```

**工作原理：**

```
用户输入 → 任务分类
    ├── 简单任务 → Reactive Agent（快速响应）
    └── 复杂任务 → Deliberative Agent（规划执行）
```

特点：
- **平衡性能和能力：** 简单任务快速响应，复杂任务智能处理
- **架构复杂：** 需要任务分类器
- **成本优化：** 简单任务用便宜的模型，复杂任务用贵的模型

**适用场景：**
- 智能客服（80% 标准问题 + 20% 复杂问题）
- 个人助手（简单查询 + 复杂任务）
- 企业应用（混合场景）

**示例代码：**

```python
class HybridAgent:
    def __init__(self, reactive_agent, deliberative_agent, classifier):
        self.reactive = reactive_agent
        self.deliberative = deliberative_agent
        self.classifier = classifier

    def run(self, user_input: str) -> str:
        """根据任务复杂度选择处理路径"""
        # 分类任务
        task_type = self.classifier.classify(user_input)

        if task_type == "simple":
            return self.reactive.run(user_input)
        else:
            return self.deliberative.run(user_input)
```

### 6.2.2 按应用分类

| 类型 | 描述 | 典型应用 | 核心能力 |
|------|------|---------|---------|
| 助手型 | 回答问题、提供建议 | 客服助手、研究助手 | 理解、推理、生成 |
| 执行型 | 执行具体任务 | 邮件Agent、文件Agent | 工具调用、任务执行 |
| 决策型 | 做出决策并执行 | 投资Agent、调度Agent | 分析、决策、执行 |
| 学习型 | 从经验中学习 | 推荐系统、自优化Agent | 学习、适应、优化 |

---

## 6.3 Agent vs Workflow：决策框架

### 6.3.1 Workflow 的本质

Workflow 是预定义的任务执行流程。它的核心特征是：**步骤固定、可预测。**

```
Workflow 示例：每日销售报表生成

1. 从数据库查询昨日销售数据（固定步骤）
2. 计算关键指标（固定公式）
3. 生成图表（固定格式）
4. 发送邮件（固定收件人）
```

**Workflow 的优势：**
- **可靠：** 步骤固定，不会出错
- **快速：** 不需要 LLM 调用，延迟低
- **便宜：** 不消耗 Token，成本低
- **可预测：** 输出格式固定

**Workflow 的局限：**
- **不灵活：** 无法处理未预见的情况
- **不智能：** 无法进行推理和决策
- **不适应：** 无法根据情况变化调整

### 6.3.2 Agent 的本质

Agent 是基于 LLM 的智能系统。它的核心特征是：**动态决策、可适应。**

```
Agent 示例：用户投诉处理

1. 理解用户投诉内容（动态分析）
2. 判断投诉类型（动态决策）
3. 查询相关政策（工具调用）
4. 制定解决方案（动态推理）
5. 与用户沟通（多轮对话）
6. 执行解决方案（动态执行）
```

**Agent 的优势：**
- **灵活：** 能处理未预见的情况
- **智能：** 能进行推理和决策
- **适应：** 能根据情况变化调整
- **自然语言：** 能理解自然语言输入

**Agent 的局限：**
- **不稳定：** LLM 的非确定性导致输出不稳定
- **昂贵：** 多次 LLM 调用导致成本高
- **慢：** 多次 LLM 调用导致延迟高
- **不可预测：** 可能做出意外的决策

### 6.3.3 决策流程图

```
                    任务是否需要动态决策？
                           │
              ┌────────────┴────────────┐
              │                          │
              否                          是
              │                          │
      使用 Workflow              任务是否需要工具调用？
                                        │
                           ┌────────────┴────────────┐
                           │                          │
                           否                          是
                           │                          │
                   使用 LLM 直接回答          任务步骤是否固定？
                                                    │
                                       ┌────────────┴────────────┐
                                       │                          │
                                       是                          否
                                       │                          │
                               使用 Workflow + LLM          使用 Agent
```

### 6.3.4 五维度评估框架

| 维度 | 评估问题 | Workflow 倾向 | Agent 倾向 |
|------|---------|--------------|-----------|
| 复杂度 | 任务步骤是否固定？ | 步骤固定 | 步骤不固定 |
| 决策性 | 是否需要动态决策？ | 不需要 | 需要 |
| 可靠性 | 对可靠性要求多高？ | 高可靠性 | 可以容忍不确定性 |
| 成本 | 预算是否充足？ | 预算有限 | 预算充足 |
| 时效性 | 对响应速度要求多高？ | 要求快速 | 可以等待 |

### 6.3.5 具体场景分析

**场景 1：每日销售报表生成**

```
分析：
- 步骤固定：查询数据 → 计算指标 → 生成图表 → 发送邮件
- 不需要动态决策：每一步都是确定的
- 高可靠性：报表必须准确
- 预算有限：每天都要执行
- 要求快速：需要在早上 9 点前完成

结论：使用 Workflow
原因：步骤固定、不需要决策、高可靠性要求
```

**场景 2：用户投诉处理**

```
分析：
- 步骤不固定：每个投诉都不同
- 需要动态决策：需要理解投诉内容、判断类型、制定方案
- 可以容忍不确定性：处理结果可以调整
- 预算充足：投诉处理是高价值任务
- 可以等待：用户可以等待几分钟

结论：使用 Agent
原因：需要理解自然语言、动态决策、多轮对话
```

**场景 3：数据分析报告**

```
分析：
- 混合场景：
  - 数据查询：固定步骤（Workflow）
  - 数据分析：需要推理（Agent）
  - 报告生成：固定格式（Workflow）

结论：使用混合架构
原因：部分步骤固定，部分需要推理
```

**场景 4：代码审查**

```
分析：
- 步骤不固定：每个代码片段都不同
- 需要动态决策：需要理解代码逻辑、判断问题
- 可以容忍不确定性：审查意见可以讨论
- 预算充足：代码审查是高价值任务
- 可以等待：审查可以花时间

结论：使用 Agent
原因：需要理解代码、动态推理、给出专业意见
```

**场景 5：自动化测试**

```
分析：
- 步骤固定：运行测试 → 收集结果 → 生成报告
- 不需要动态决策：测试步骤是预定义的
- 高可靠性：测试结果必须准确
- 预算有限：每次提交都要执行
- 要求快速：需要快速反馈

结论：使用 Workflow
原因：步骤固定、不需要决策、高可靠性要求
```

### 6.3.6 决策矩阵

```
                    步骤固定          步骤不固定
需要推理      Workflow+LLM        Agent
不需要推理    Workflow            Workflow+简单规则
```

---

## 6.4 混合架构设计

### 6.4.1 为什么需要混合架构

大多数实际系统都不是纯 Agent 或纯 Workflow，而是两者的混合。原因：

1. **成本优化：** 简单任务用 Workflow（便宜），复杂任务用 Agent（贵但智能）
2. **性能优化：** 简单任务用 Workflow（快），复杂任务用 Agent（慢但准确）
3. **可靠性优化：** 关键步骤用 Workflow（可靠），辅助步骤用 Agent（灵活）

### 6.4.2 混合架构模式

**模式 1：路由模式**

根据任务类型路由到不同的处理路径。

```python
class RoutingHybrid:
    def __init__(self):
        self.workflow = WorkflowEngine()
        self.agent = AgentEngine()
        self.classifier = TaskClassifier()

    def run(self, task):
        task_type = self.classifier.classify(task)
        if task_type == "standard":
            return self.workflow.run(task)
        else:
            return self.agent.run(task)
```

**模式 2：增强模式**

Workflow 处理主流程，Agent 处理异常和边缘情况。

```python
class AugmentationHybrid:
    def __init__(self):
        self.workflow = WorkflowEngine()
        self.agent = AgentEngine()

    def run(self, task):
        try:
            # 先尝试 Workflow
            return self.workflow.run(task)
        except Exception as e:
            # Workflow 失败，用 Agent 处理
            return self.agent.run(task, context=f"Workflow 失败：{e}")
```

**模式 3：协作模式**

Workflow 和 Agent 协作完成任务。

```python
class CollaborationHybrid:
    def __init__(self):
        self.workflow = WorkflowEngine()
        self.agent = AgentEngine()

    def run(self, task):
        # Workflow 处理数据准备
        prepared_data = self.workflow.prepare(task)

        # Agent 处理分析和决策
        analysis = self.agent.analyze(prepared_data)

        # Workflow 处理输出生成
        output = self.workflow.generate(analysis)

        return output
```

---

## 6.5 行业应用案例

### 6.5.1 客服领域

**Workflow 部分：**
- 标准问题自动回复（FAQ）
- 工单创建和分配
- 满意度调查
- 订单状态查询

**Agent 部分：**
- 复杂问题推理
- 多轮对话
- 情绪识别和安抚
- 个性化服务

**架构：**

```
用户输入 → 意图识别
    ├── 标准问题 → Workflow 自动回复
    ├── 订单查询 → Workflow 查询系统
    └── 复杂问题 → Agent 处理
```

### 6.5.2 数据分析

**Workflow 部分：**
- 定时报表生成
- 数据清洗
- 指标计算
- 数据可视化

**Agent 部分：**
- 探索性分析
- 异常检测
- 洞察发现
- 自然语言查询

**架构：**

```
用户查询 → 查询理解
    ├── 标准报表 → Workflow 生成
    ├── 简单查询 → Workflow 执行 SQL
    └── 复杂分析 → Agent 探索性分析
```

### 6.5.3 软件开发

**Workflow 部分：**
- CI/CD 流水线
- 自动化测试
- 代码部署
- 环境配置

**Agent 部分：**
- 代码审查
- Bug 修复建议
- 架构建议
- 文档生成

**架构：**

```
代码提交 → Workflow CI/CD
    ├── 测试失败 → Agent 分析失败原因
    ├── 代码问题 → Agent 审查代码
    └── 部署完成 → Agent 生成文档
```

### 6.5.4 金融领域

**Workflow 部分：**
- 交易执行
- 风险计算
- 报表生成
- 合规检查

**Agent 部分：**
- 投资分析
- 风险评估
- 市场预测
- 客户服务

**架构：**

```
交易请求 → Workflow 风险检查
    ├── 低风险 → Workflow 自动执行
    ├── 中风险 → Agent 分析后执行
    └── 高风险 → Agent 分析 + 人工确认
```

---

## 6.6 选型决策的量化方法

### 6.6.1 成本效益分析

在实际项目中，我们可以通过量化分析来辅助选型决策。下面的框架帮助你从成本角度评估 Agent 和 Workflow 的优劣。

```python
@dataclass
class TaskProfile:
    """任务画像"""
    name: str
    daily_volume: int          # 每日处理量
    avg_complexity: float      # 复杂度（1-10）
    needs_reasoning: bool      # 是否需要推理
    reliability_requirement: float  # 可靠性要求（0-1）
    latency_budget_ms: float   # 延迟预算（毫秒）
    budget_per_task: float     # 每任务预算（美元）


@dataclass
class SolutionCost:
    """方案成本"""
    setup_cost: float          # 搭建成本
    per_task_cost: float       # 单次成本
    monthly_maintenance: float  # 月维护成本
    avg_latency_ms: float      # 平均延迟
    reliability: float          # 可靠性评分


def evaluate_solution(task: TaskProfile, solution: SolutionCost) -> dict:
    """评估方案是否适合任务"""
    monthly_cost = (
        solution.setup_cost / 12
        + solution.per_task_cost * task.daily_volume * 30
        + solution.monthly_maintenance
    )

    within_budget = solution.per_task_cost <= task.budget_per_task
    within_latency = solution.avg_latency_ms <= task.latency_budget_ms
    meets_reliability = solution.reliability >= task.reliability_requirement

    score = 0
    if within_budget:
        score += 30
    if within_latency:
        score += 25
    if meets_reliability:
        score += 25
    if solution.per_task_cost < task.budget_per_task * 0.5:
        score += 20  # 成本远低于预算加分

    return {
        "task": task.name,
        "monthly_cost": f"${monthly_cost:.2f}",
        "within_budget": within_budget,
        "within_latency": within_latency,
        "meets_reliability": meets_reliability,
        "suitability_score": score,
        "recommendation": "推荐" if score >= 70 else "需要权衡" if score >= 50 else "不推荐",
    }
```

### 6.6.2 复杂度评分卡

对于无法量化的维度，可以使用评分卡来辅助决策。

```python
class ComplexityScorecard:
    """任务复杂度评分卡"""

    DIMENSIONS = {
        "input_variability": "输入变化程度（1-固定，10-完全随机）",
        "decision_points": "需要动态决策的环节数量",
        "external_dependencies": "外部系统依赖数量",
        "error_scenarios": "可能的错误场景数量",
        "domain_knowledge": "所需的领域知识深度",
        "user_expectation": "用户对智能程度的期望",
    }

    @staticmethod
    def score_task(dimension_scores: dict) -> dict:
        """计算任务总分"""
        total = sum(dimension_scores.values())
        max_score = len(dimension_scores) * 10
        ratio = total / max_score

        if ratio <= 0.3:
            recommendation = "使用 Workflow"
            reason = "任务简单，固定流程即可"
        elif ratio <= 0.6:
            recommendation = "使用 Workflow + 简单规则"
            reason = "任务有一定复杂度，但不需要完整推理"
        elif ratio <= 0.8:
            recommendation = "使用 Agent 或混合架构"
            reason = "任务较复杂，需要动态决策"
        else:
            recommendation = "使用 Agent"
            reason = "任务高度复杂，需要完整推理能力"

        return {
            "scores": dimension_scores,
            "total": total,
            "max": max_score,
            "ratio": round(ratio, 2),
            "recommendation": recommendation,
            "reason": reason,
        }
```

---

## 6.7 常见坑与最佳实践

### 6.7.1 常见坑

**坑 1：过度使用 Agent**

```
问题：简单任务用 Agent 是浪费资源
示例：用 Agent 来回答 "1+1=?" 
症状：成本高、延迟大、用户体验差
解决：评估任务复杂度，简单任务用简单方案
```

**坑 2：Agent 不适合高可靠性场景**

```
问题：Agent 的非确定性不适合关键业务
示例：用 Agent 控制核电站
症状：输出不稳定、可能做出错误决策
解决：关键业务用 Workflow + 规则引擎
```

**坑 3：忽略成本**

```
问题：Agent 的成本远高于 Workflow
示例：每天处理 10 万个请求，每个都用 Agent
症状：成本失控
解决：评估 ROI，只有高价值任务用 Agent
```

**坑 4：混合架构设计不当**

```
问题：Agent 和 Workflow 之间切换不顺畅
示例：Workflow 失败后 Agent 无法获取上下文
症状：用户体验差、系统不稳定
解决：设计清晰的接口和上下文传递机制
```

**坑 5：没有监控和反馈循环**

```
问题：上线后不知道 Agent 表现如何
症状：无法发现问题，无法持续优化
解决：建立完善的监控体系，收集用户反馈
```

### 6.7.2 最佳实践

1. **从 Workflow 开始：** 如果能用 Workflow 解决，就不要用 Agent
2. **渐进式引入 Agent：** 先在边缘场景测试 Agent，验证效果后再推广
3. **监控成本和性能：** 建立监控体系，及时发现和解决问题
4. **设计清晰的接口：** Agent 和 Workflow 之间的接口要清晰、稳定
5. **持续优化：** 根据实际使用情况，不断调整 Agent 和 Workflow 的边界
6. **A/B 测试验证效果：** 在引入 Agent 时，用 A/B 测试对比 Agent 和非 Agent 方案的效果差异

---

## 6.7 练习题

### 概念理解题

1. **Reactive Agent 和 Deliberative Agent 的区别是什么？** 请从架构、能力、适用场景三个维度对比。

2. **什么时候应该用 Workflow 而不是 Agent？** 请举 3 个具体场景，并说明理由。

3. **混合架构的优势是什么？** 请设计一个混合架构，说明各部分的职责。

4. **如何评估一个任务是否适合用 Agent 解决？** 请使用五维度评估框架分析。

5. **Agent 的"非确定性"是缺点还是优点？** 在什么场景下它是优点？在什么场景下它是缺点？

### 动手实践题

1. **场景分析：** 分析以下 10 个场景，判断应该用 Agent 还是 Workflow：
   - 每日邮件摘要
   - 代码审查
   - 数据库备份
   - 客户投诉处理
   - 自动化测试
   - 研究报告生成
   - 系统监控告警
   - 文件格式转换
   - 用户行为分析
   - 智能推荐

2. **架构设计：** 为"智能客服系统"设计架构，说明哪些部分用 Agent，哪些部分用 Workflow。

3. **成本分析：** 对比 Agent 和 Workflow 处理同一个任务的成本，分析 ROI。

### 思考题

1. 随着 LLM 能力的提升，Workflow 会被 Agent 完全取代吗？为什么？

2. 在什么场景下，Agent 的"非确定性"反而是一个优势？

3. 如何设计一个系统，让它能自动判断应该用 Agent 还是 Workflow？

---

## 6.8 实战任务

**任务：分析 10 个场景，判断用 Agent 还是 Workflow**

**具体要求：**

1. 分析 10 个不同场景
2. 对每个场景给出选型建议
3. 说明选型理由（使用五维度评估框架）
4. 如果是混合架构，说明各部分的职责
5. 估算每种方案的成本和性能

**评估标准：**

| 维度 | 权重 | 评分标准 |
|------|------|---------|
| 分析深度 | 30% | 不只是给结论，还有详细的分析过程 |
| 选型准确性 | 30% | 选型建议合理，有说服力 |
| 理由充分性 | 25% | 理由基于五维度框架，逻辑清晰 |
| 考虑全面性 | 15% | 考虑了成本、可靠性、性能等因素 |

---

## 6.9 本章小结

### 核心知识点回顾

- **Agent 分类：** Reactive（反应式，简单快速）、Deliberative（审议式，智能灵活）、Hybrid（混合式，平衡性能和能力）

- **适用场景：** Agent 适合需要动态决策的复杂任务；Workflow 适合步骤固定的标准化任务

- **决策框架：** 从复杂度、决策性、可靠性、成本、时效性五个维度评估

- **混合架构：** 路由模式（按任务类型路由）、增强模式（Workflow 主 + Agent 辅）、协作模式（两者协作）

- **行业应用：** 客服、数据分析、软件开发、金融等领域都有 Agent + Workflow 的混合应用

### 关键公式

```
选型决策 = f(任务复杂度, 决策需求, 可靠性要求, 成本预算, 时效要求)
混合架构价值 = Workflow 的可靠性 + Agent 的智能性
```

### 下一章预告

下一章我们将深入探讨 **Agent 的认知架构** —— ReAct 与推理模式。你将理解 Agent 如何"思考"和"决策"，以及不同的推理模式如何影响 Agent 的表现。

---

*上一章：[第 5 章：Agent 的本质](../chapter-05/README.md)*
*下一章：[第 7 章：Agent 的认知架构](../chapter-07/README.md)*
