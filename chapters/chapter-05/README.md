# 第 5 章：Agent 的本质 —— LLM + 循环 + 工具

> **本章定位：** 这是全书的核心概念章。Agent 不是一个框架，而是一种架构模式。理解 Agent 的本质才能在后续章节中做出正确的设计决策。本章将带你深入理解 Agent 的核心公式、Agent Loop 的工作机制，以及如何平衡 Agent 的自主性和人类控制。

---

## 学习目标

1. **理解 Agent = LLM + Loop + Tools 的核心公式** —— 能解释每个组件的作用和它们之间的关系
2. **理解 Agent 和 Chatbot 的本质区别** —— 能从架构层面分析两者的差异
3. **理解 Agent Loop 的工作机制** —— 能描述观察→思考→行动→反馈的完整循环
4. **理解 Agent 的自主性层次** —— 能根据场景选择合适的自主性级别
5. **能从零实现一个最简 Agent** —— 用 <100 行代码实现一个可运行的 Agent

## 核心问题

1. **Agent 和 Chatbot 的本质区别是什么？** 为什么说 Agent 是 Chatbot 的"进化版"？它们在架构上有什么根本不同？
2. **Agent Loop 为什么是 Agent 的核心？** 没有循环的"Agent"能叫 Agent 吗？循环如何让 Agent 能持续工作？
3. **如何平衡 Agent 的自主性和人类控制？** 自主性太高有什么风险？太低有什么问题？

---

## 5.1 Agent 的定义

### 5.1.1 一句话定义

**Agent = LLM + Loop + Tools**

这是一个看似简单但内涵丰富的公式：

- **LLM（大脑）：** 负责理解用户意图、分析当前情况、做出决策
- **Loop（心跳）：** 让 Agent 能持续工作，而不是一次性的输入-输出
- **Tools（手脚）：** 让 Agent 能执行行动，获取外部数据

### 5.1.2 Agent 的完整公式

展开来看，Agent 的完整公式是：

```
Agent = Model + Tools + Memory + State + Context + Runtime + Guardrails + Evaluation
```

| 组件 | 作用 | 类比 | 本章重点 |
|------|------|------|---------|
| Model | 理解和推理 | 大脑 | 第 1 章 |
| Tools | 执行行动 | 手脚 | 第 4 章 |
| Memory | 记忆 | 记忆力 | 第 13 章 |
| State | 当前状态 | 注意力 | 第 8 章 |
| Context | 上下文信息 | 视野 | 第 20 章 |
| Runtime | 运行环境 | 身体 | 第 49 章 |
| Guardrails | 安全护栏 | 免疫系统 | 第 35 章 |
| Evaluation | 评估反馈 | 学习能力 | 第 33 章 |

### 5.1.3 为什么这个公式重要

这个公式不是学术定义，而是工程实践的总结。当你设计一个 Agent 时，你需要回答：

1. 用什么模型？（Model）
2. 用什么工具？（Tools）
3. 如何管理记忆？（Memory）
4. 如何管理状态？（State）
5. 如何管理上下文？（Context）
6. 如何部署和运行？（Runtime）
7. 如何保证安全？（Guardrails）
8. 如何评估效果？（Evaluation）

每个组件都需要仔细设计，任何一个组件出问题都会影响整个 Agent 的表现。

---

## 5.2 Agent Loop：Agent 的心跳

### 5.2.1 为什么需要循环

没有循环的 LLM 只能做一次性的输入-输出：

```
用户：今天天气怎么样？
LLM：今天天气不错。（完成）
```

有了循环，LLM 可以持续工作：

```
用户：帮我规划今天的行程
LLM：（循环 1）我需要先查天气 → 调用天气工具
LLM：（循环 2）天气晴朗，可以安排户外活动 → 调用日历工具
LLM：（循环 3）查看你的日历，下午有空 → 生成行程
LLM：（循环 4）行程规划完成，返回给用户
```

### 5.2.2 Agent Loop 的核心循环

Agent 的核心是一个循环：

```python
def agent_loop(user_input: str) -> str:
    """Agent 的核心循环"""
    state = initialize_state(user_input)

    while not task_complete(state):
        # 1. 观察：获取当前状态
        observation = observe(state)

        # 2. 思考：LLM 分析情况，决定下一步
        thought = think(observation, state)

        # 3. 行动：执行工具调用或返回答案
        action = decide_action(thought)

        if action.type == "answer":
            return action.content

        # 4. 反馈：获取行动结果
        result = execute_action(action)

        # 5. 更新状态
        state = update_state(state, result)

    return "任务未完成"
```

### 5.2.3 循环的关键特征

**特征 1：自主性**

Agent 自己决定下一步做什么，不需要人类每一步都指导。

```
人类：帮我写一篇关于 AI 的文章
Agent：（自主决定）
  1. 搜索相关资料
  2. 阅读资料
  3. 大纲规划
  4. 撰写文章
  5. 审查修改
```

**特征 2：迭代性**

通过多次循环逐步完成任务，而不是一步到位。

```
第 1 轮：搜索 "AI 最新进展"
第 2 轮：阅读搜索结果
第 3 轮：搜索 "AI 应用案例"
第 4 轮：综合信息，生成文章
```

**特征 3：反馈驱动**

每次行动的结果影响下一步决策。

```
第 1 轮行动：搜索 "Python 最新版本"
反馈：搜索失败（网络错误）
第 2 轮决策：重试搜索，或者换一个搜索词
```

**特征 4：有界性**

需要有终止条件，防止无限循环。

```python
MAX_ITERATIONS = 10  # 最大循环次数
TIMEOUT = 60  # 超时时间

for i in range(MAX_ITERATIONS):
    # ... 循环逻辑
    if time.time() - start_time > TIMEOUT:
        break
```

---

## 5.3 Agent vs Chatbot：本质区别

### 5.3.1 架构对比

```
Chatbot 架构：
  用户输入 → LLM → 输出
  （一次性的输入-输出，没有循环）

Agent 架构：
  用户输入 → [循环] → LLM → 工具调用 → 反馈 → LLM → ... → 输出
  （多次迭代，有状态管理）
```

### 5.3.2 能力对比

| 特征 | Chatbot | Agent |
|------|---------|-------|
| 核心能力 | 生成文本 | 执行行动 |
| 工作模式 | 一问一答 | 自主循环 |
| 外部交互 | 无 | 有（工具调用） |
| 状态管理 | 简单对话历史 | 复杂状态（任务状态、环境状态） |
| 适用场景 | 问答、闲聊 | 任务执行、决策、规划 |
| 错误处理 | 重试 | 自我修正 |
| 成本 | 低 | 高（多次 LLM 调用） |

### 5.3.3 一个具体的对比

**场景：用户想知道北京的天气**

```
Chatbot：
  用户：北京天气怎么样？
  LLM：北京今天可能天气不错。（可能是幻觉）
  （结束）

Agent：
  用户：北京天气怎么样？
  Agent：（思考）用户想知道北京天气，我需要调用天气工具
  Agent：（行动）[get_weather(city="北京")]
  工具返回：{"temp": 25, "weather": "晴", "humidity": 40}
  Agent：（思考）已经获取了天气信息，可以回答用户了
  Agent：（回答）北京今天晴，温度 25°C，湿度 40%。
  （结束）
```

### 5.3.4 什么时候用 Chatbot，什么时候用 Agent

```
用 Chatbot：
  - 简单问答
  - 文本生成
  - 翻译
  - 不需要外部数据

用 Agent：
  - 需要执行行动
  - 需要获取外部数据
  - 任务需要多步完成
  - 需要规划和决策
```

---

## 5.4 最简 Agent 实现

### 5.4.1 100 行代码的 Agent

```python
import os
import re
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv()
client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))


# 工具定义
def calculator(expression: str) -> str:
    """数学计算工具"""
    try:
        return str(eval(expression))
    except Exception as e:
        return f"计算错误：{e}"


def search(query: str) -> str:
    """搜索工具（简化版）"""
    return f"搜索结果：关于 '{query}' 的信息..."  # 实际应调用搜索 API


# 工具注册
TOOLS = {
    "calculator": calculator,
    "search": search,
}

SYSTEM_PROMPT = """你是一个助手。你可以使用以下工具：
- calculator(expression): 数学计算
- search(query): 搜索信息

当需要使用工具时，输出格式：[tool_name(args)]

直接回答时，直接输出答案。"""


def agent_loop(user_input: str, max_iterations: int = 5) -> str:
    """最简 Agent Loop"""
    messages = [
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": user_input},
    ]

    for iteration in range(max_iterations):
        # 调用 LLM
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            temperature=0.3,
        )
        assistant_message = response.choices[0].message.content

        # 检查是否需要调用工具
        match = re.search(r"\[(\w+)\((.*?)\)\]", assistant_message)
        if not match:
            # 不需要工具，返回答案
            return assistant_message

        # 执行工具
        tool_name = match.group(1)
        tool_args_str = match.group(2)
        tool_fn = TOOLS.get(tool_name)

        if tool_fn:
            result = tool_fn(tool_args_str.strip("'\""))
        else:
            result = f"工具 {tool_name} 不存在"

        # 更新消息
        messages.append({"role": "assistant", "content": assistant_message})
        messages.append({"role": "user", "content": f"工具返回：{result}"})

    return "达到最大迭代次数，任务未完成"


# 测试
if __name__ == "__main__":
    print(agent_loop("计算 123 * 456 + 789"))
    # 输出：62967

    print(agent_loop("Python 是谁发明的？"))
    # 输出：Python 是由 Guido van Rossum 在 1991 年发明的。
```

### 5.4.2 代码解析

让我们逐行分析这个 Agent 的工作原理：

1. **工具定义：** 定义了 `calculator` 和 `search` 两个工具
2. **System Prompt：** 告诉 LLM 它有哪些工具，以及如何调用
3. **Agent Loop：** 循环执行，直到任务完成或达到最大迭代次数
4. **工具调用解析：** 使用正则表达式解析 LLM 输出的工具调用
5. **结果反馈：** 将工具返回的结果添加到消息历史，让 LLM 能看到

---

## 5.5 Agent 的自主性层次

### 5.5.1 自主性光谱

Agent 的自主性不是非黑即白的，而是一个光谱：

```
完全被动 ←————————————————————→ 完全自主
   |                                    |
  工具          协助型Agent        自主型Agent
(被动执行)    (建议+执行)      (自主决策+执行)
```

| 层次 | 描述 | 示例 | 适用场景 |
|------|------|------|---------|
| Level 0 | 纯聊天 | 没有工具调用 | 闲聊、问答 |
| Level 1 | 工具调用 | 用户明确指定用什么工具 | 简单自动化 |
| Level 2 | 建议+执行 | Agent 建议工具，用户确认后执行 | 客服、邮件处理 |
| Level 3 | 自主决策 | Agent 自己决定用什么工具 | 研究助手、数据分析 |
| Level 4 | 完全自主 | Agent 自己设定目标并执行 | 个人数字员工 |

### 5.5.2 为什么自主性层次很重要

不同的场景需要不同的自主性层次：

**低自主性场景（Level 1-2）：**

```
场景：邮件处理 Agent
风险：可能发送错误的邮件
策略：Agent 建议回复内容，用户确认后发送
```

**高自主性场景（Level 3-4）：**

```
场景：研究助手 Agent
风险：搜索和分析是安全的操作
策略：Agent 自主搜索、分析、生成报告
```

### 5.5.3 人类控制机制

```python
class HumanInTheLoop:
    """人类确认机制"""

    def __init__(self, auto_approve_tools: list[str] = None):
        self.auto_approve = set(auto_approve_tools or [])

    def confirm(self, tool_name: str, args: dict, description: str) -> bool:
        """确认是否执行工具"""
        if tool_name in self.auto_approve:
            return True

        print(f"\n工具调用请求：")
        print(f"  工具：{tool_name}")
        print(f"  描述：{description}")
        print(f"  参数：{args}")

        response = input("是否允许执行？(y/n): ")
        return response.lower() == "y"
```

---

## 5.6 案例分析：Agent 的设计决策

### 场景：构建一个邮件管理 Agent

**设计决策 1：自主性层次**

选择 Level 2（建议+执行），原因：
- 读取邮件：自动执行（安全）
- 搜索邮件：自动执行（安全）
- 草拟回复：自动执行（安全）
- **发送邮件：需要人类确认**（不可逆操作）

**设计决策 2：工具选择**

```python
tools = [
    {"name": "read_email", "description": "读取邮件内容"},
    {"name": "search_emails", "description": "搜索邮件"},
    {"name": "draft_reply", "description": "草拟邮件回复"},
    {"name": "send_email", "description": "发送邮件（需要确认）"},
]
```

**设计决策 3：安全策略**

```python
security_rules = """
1. 不自动删除邮件
2. 不自动发送给非白名单地址
3. 所有操作记录审计日志
4. 发送邮件前必须获得用户确认
"""
```

---

## 5.7 Agent Loop 的实现细节

### 5.7.1 状态机视角

从状态机的角度来看，Agent Loop 就是一个有限状态机。理解这一点有助于我们设计更健壮的 Agent。

```python
from enum import Enum


class AgentState(Enum):
    """Agent 状态枚举"""
    IDLE = "idle"                    # 空闲，等待用户输入
    THINKING = "thinking"            # LLM 正在推理
    TOOL_CALLING = "tool_calling"    # 正在调用工具
    PROCESSING = "processing"        # 处理工具返回结果
    GENERATING = "generating"        # 正在生成最终回答
    ERROR = "error"                  # 出错状态
    DONE = "done"                    # 任务完成


class AgentStateMachine:
    """Agent 状态机"""

    def __init__(self):
        self.state = AgentState.IDLE
        self.transitions = self._build_transitions()

    def _build_transitions(self) -> dict:
        """定义合法的状态转换"""
        return {
            AgentState.IDLE: [AgentState.THINKING, AgentState.DONE],
            AgentState.THINKING: [AgentState.TOOL_CALLING, AgentState.GENERATING, AgentState.ERROR],
            AgentState.TOOL_CALLING: [AgentState.PROCESSING, AgentState.ERROR],
            AgentState.PROCESSING: [AgentState.THINKING, AgentState.GENERATING],
            AgentState.GENERATING: [AgentState.DONE, AgentState.ERROR],
            AgentState.ERROR: [AgentState.IDLE, AgentState.THINKING],
            AgentState.DONE: [AgentState.IDLE],
        }

    def transition(self, new_state: AgentState) -> bool:
        """执行状态转换"""
        allowed = self.transitions.get(self.state, [])
        if new_state in allowed:
            self.state = new_state
            return True
        raise ValueError(
            f"非法状态转换：{self.state.value} -> {new_state.value}。"
            f"允许的目标状态：{[s.value for s in allowed]}"
        )
```

### 5.7.2 Agent Loop 的可观测性

一个健壮的 Agent Loop 需要完整的日志记录，这在调试和生产监控中至关重要。

```python
import json
from datetime import datetime
from dataclasses import dataclass, field, asdict


@dataclass
class AgentStep:
    """Agent 步骤记录"""
    step_number: int
    timestamp: str
    state: str
    action: str = ""
    tool_name: str = ""
    tool_args: dict = field(default_factory=dict)
    tool_result: str = ""
    thought: str = ""
    duration_ms: float = 0
    tokens_used: int = 0

    def to_dict(self) -> dict:
        return asdict(self)


class AgentTracer:
    """Agent 执行追踪器"""

    def __init__(self):
        self.steps: list[AgentStep] = []
        self.start_time = datetime.now()

    def record_step(self, step_number: int, state: str, **kwargs):
        """记录一步执行"""
        step = AgentStep(
            step_number=step_number,
            timestamp=datetime.now().isoformat(),
            state=state,
            **kwargs,
        )
        self.steps.append(step)

    def get_trace(self) -> str:
        """获取可读的执行轨迹"""
        lines = ["=== Agent 执行轨迹 ==="]
        for step in self.steps:
            lines.append(f"\n步骤 {step.step_number} [{step.state}] {step.timestamp}")
            if step.thought:
                lines.append(f"  思考: {step.thought}")
            if step.action:
                lines.append(f"  行动: {step.action}")
            if step.tool_name:
                lines.append(f"  工具: {step.tool_name}({step.tool_args})")
            if step.tool_result:
                lines.append(f"  结果: {step.tool_result[:100]}...")
            if step.duration_ms:
                lines.append(f"  耗时: {step.duration_ms:.0f}ms")
        return "\n".join(lines)

    def get_summary(self) -> dict:
        """获取执行摘要"""
        total_time = (datetime.now() - self.start_time).total_seconds()
        tool_calls = [s for s in self.steps if s.tool_name]
        return {
            "total_steps": len(self.steps),
            "tool_calls": len(tool_calls),
            "total_time_seconds": total_time,
            "tools_used": list(set(s.tool_name for s in tool_calls)),
        }
```

### 5.7.3 优雅的 Agent Loop 终止策略

仅仅设置 max_iterations 是不够的，我们需要更智能的终止策略。

```python
class TerminationStrategy:
    """终止策略组合"""

    def __init__(
        self,
        max_iterations: int = 10,
        max_tokens: int = 50000,
        max_tool_calls: int = 15,
        timeout_seconds: float = 120,
    ):
        self.max_iterations = max_iterations
        self.max_tokens = max_tokens
        self.max_tool_calls = max_tool_calls
        self.timeout_seconds = timeout_seconds

        self.iteration_count = 0
        self.total_tokens = 0
        self.tool_call_count = 0
        self.start_time = time.time()

    def should_terminate(self) -> tuple[bool, str]:
        """检查是否应该终止"""
        reasons = []

        if self.iteration_count >= self.max_iterations:
            reasons.append(f"超过最大迭代次数 ({self.max_iterations})")

        if self.total_tokens >= self.max_tokens:
            reasons.append(f"超过最大 Token 限制 ({self.max_tokens})")

        if self.tool_call_count >= self.max_tool_calls:
            reasons.append(f"超过最大工具调用次数 ({self.max_tool_calls})")

        elapsed = time.time() - self.start_time
        if elapsed >= self.timeout_seconds:
            reasons.append(f"超过超时时间 ({self.timeout_seconds}s)")

        if reasons:
            return True, "终止原因: " + "; ".join(reasons)

        return False, ""

    def record_iteration(self, tokens: int = 0, tool_called: bool = False):
        """记录一次迭代"""
        self.iteration_count += 1
        self.total_tokens += tokens
        if tool_called:
            self.tool_call_count += 1
```

---

## 5.8 常见坑与最佳实践

### 5.8.1 常见坑

1. **没有终止条件：** Agent 可能陷入无限循环，消耗大量 Token 和时间。

2. **自主性过高：** Agent 可能执行危险操作（如删除文件、发送邮件），造成不可逆的损失。

3. **自主性过低：** 需要人类频繁确认，失去自动化的价值。

4. **不记录行为：** 无法审计 Agent 的决策过程，出了问题无法追溯。

5. **消息历史无限增长：** 长对话中，消息列表越来越长，消耗大量 Token。需要有消息压缩或截断策略。

6. **工具返回值未处理：** 工具返回了大量数据，全部塞进上下文窗口，导致后续推理质量下降。应该对工具返回值进行摘要或截断。

### 5.8.2 最佳实践

1. **始终设置 max_iterations** —— 防止无限循环
2. **根据操作风险设置自主性层次** —— 高风险操作需要人类确认
3. **关键操作需要人类确认** —— 发送邮件、删除文件等
4. **记录所有 Agent 行为** —— 支持审计和调试
5. **消息历史要有上限** —— 超过上限时压缩旧消息为摘要
6. **工具返回值要截断或摘要** —— 防止上下文窗口爆炸
7. **每个 Agent 循环都要有超时机制** —— 防止工具调用卡住

---

## 5.8 练习题

### 概念理解题

1. Agent 和 Chatbot 的本质区别是什么？请从架构层面分析。

2. Agent Loop 的四个步骤是什么？每个步骤的作用是什么？

3. 如何平衡 Agent 的自主性和人类控制？请举一个具体的场景说明。

4. 为什么需要设置 max_iterations？如果不设置会有什么问题？

5. 设计一个文件管理 Agent 的安全策略。哪些操作需要人类确认？哪些可以自动执行？

### 动手实践题

1. **实现最简 Agent：** 用 <100 行代码实现一个 Agent，支持至少 2 个工具。

2. **自主性实验：** 对同一个任务，分别用 Level 2 和 Level 3 的自主性实现，对比效果。

3. **终止条件实验：** 故意不设置 max_iterations，观察 Agent 的行为。

### 思考题

1. Agent 的自主性是否有"最优"级别？还是完全取决于场景？

2. 如果 Agent 执行了一个错误的操作，应该如何回滚？Agent 需要"撤销"能力吗？

3. 多个 Agent 协作时，如何分配自主性？每个 Agent 都应该有相同的自主性级别吗？

---

## 5.9 实战任务

**任务：从零实现一个最简 Agent（<100 行代码）**

**具体要求：**

1. 实现 Agent Loop
2. 支持至少 2 个工具
3. 支持多轮对话
4. 设置终止条件（max_iterations）
5. 记录 Agent 的决策过程（打印每一步的 Thought、Action、Observation）

**评估标准：**

| 维度 | 权重 | 评分标准 |
|------|------|---------|
| 代码正确性 | 30% | Agent 能正确运行 |
| 工具调用 | 25% | 能正确选择和调用工具 |
| 循环控制 | 20% | 有终止条件，不会无限循环 |
| 决策记录 | 15% | 能看到 Agent 的思考过程 |
| 代码质量 | 10% | 结构清晰，可读性好 |

---

## 5.10 Agent 开发的常见误区

### 5.10.1 误区一：Agent 能解决所有问题

很多人在学习 Agent 后，会想要用 Agent 来解决所有问题。但实际上，很多问题用简单的程序逻辑就能解决，不需要引入 Agent。比如，数据格式转换、定时任务、简单的 CRUD 操作——这些都不需要 Agent。

### 5.10.2 误区二：Agent 越自主越好

自主性是一把双刃剑。过高的自主性意味着更多的不确定性，可能带来安全风险和不可预期的行为。在大多数实际场景中，适度的自主性加上人类监督是更好的选择。

### 5.10.3 误区三：只要 Prompt 写得好，Agent 就能工作

Prompt Engineering 确实很重要，但它不是万能的。一个健壮的 Agent 需要完整的工程支持：错误处理、状态管理、安全机制、监控日志。仅仅依靠 Prompt 无法构建生产级的 Agent 系统。

### 5.10.4 误区四：Agent 的成本只是 API 调用费用

API 调用费用只是 Agent 成本的一部分。开发成本、测试成本、运维成本、监控成本——这些隐性成本往往被低估。在设计 Agent 系统时，需要全面考虑成本结构。

---

## 5.11 Agent 的监控与可观测性

### 5.11.1 为什么需要监控

Agent 在生产环境中运行时，你需要知道它在做什么、做得怎么样、有没有出问题。没有监控的 Agent 就像没有仪表盘的汽车——你不知道它跑多快、油还剩多少、有没有故障。

### 5.11.2 监控指标

Agent 系统需要监控以下几个关键指标。

**性能指标**：每次 Agent 调用的延迟、LLM 的响应时间、工具执行的时间。这些指标帮助你发现性能瓶颈。

**质量指标**：任务完成率、用户满意度、工具调用成功率。这些指标帮助你评估 Agent 的表现。

**成本指标**：每次调用的 Token 消耗、API 调用费用。这些指标帮助你控制成本。

**安全指标**：异常输入的数量、安全拦截的次数、权限越界的尝试。这些指标帮助你发现安全威胁。

```python
class AgentMonitor:
    """Agent 监控系统"""

    def __init__(self):
        self.metrics = {
            "total_calls": 0,
            "success_count": 0,
            "error_count": 0,
            "avg_latency_ms": 0,
            "total_tokens": 0,
        }

    def record_call(self, latency_ms: float, tokens: int, success: bool):
        """记录一次 Agent 调用"""
        self.metrics["total_calls"] += 1
        if success:
            self.metrics["success_count"] += 1
        else:
            self.metrics["error_count"] += 1

        n = self.metrics["total_calls"]
        self.metrics["avg_latency_ms"] = (
            self.metrics["avg_latency_ms"] * (n - 1) + latency_ms
        ) / n
        self.metrics["total_tokens"] += tokens

    def get_success_rate(self) -> float:
        """获取成功率"""
        total = self.metrics["total_calls"]
        if total == 0:
            return 0.0
        return self.metrics["success_count"] / total
```

---

## 5.12 本章小结

### 核心知识点回顾

- **Agent 的核心公式：** Agent = LLM + Loop + Tools。LLM 是大脑，Loop 是心跳，Tools 是手脚。

- **Agent Loop：** 观察→思考→行动→反馈的循环。这是 Agent 能持续工作的基础。

- **Agent vs Chatbot：** Agent 有循环、有工具调用、有状态管理；Chatbot 是一次性的输入-输出。

- **自主性层次：** 从 Level 0（纯聊天）到 Level 4（完全自主），不同场景需要不同的自主性。

- **人类控制：** 关键操作需要人类确认，防止 Agent 执行危险操作。

### 关键公式

```
Agent = LLM（理解） + Loop（持续工作） + Tools（执行行动）
Agent 的价值 = 自主性 × 可控性 × 可靠性
```

### 下一章预告

下一章我们将探讨 **Agent 的分类与适用场景** —— 不是所有问题都需要 Agent。你将学习如何判断什么时候应该用 Agent，什么时候应该用 Workflow，以及如何设计混合架构。

---

*上一章：[第 4 章：Tool Calling](../chapter-04/README.md)*
*下一章：[第 6 章：Agent 的分类与适用场景](../chapter-06/README.md)*
