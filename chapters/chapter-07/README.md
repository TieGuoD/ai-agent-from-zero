# 第 7 章：Agent 的认知架构 —— ReAct 与推理模式

> **本章定位：** ReAct 是最经典的 Agent 推理模式，理解 ReAct 才能理解 Agent 如何"思考"和"决策"。本章将带你像读故事一样理解 ReAct 的原理，从一个简单的例子出发，逐步深入到 ReAct 的实现细节，并介绍 Plan-and-Execute、Reflexion 等其他推理模式。

---

## 学习目标

1. **深入理解 ReAct（Reasoning + Acting）模式** —— 能用自己的话解释 Thought → Action → Observation 循环的每一步在做什么
2. **理解 ReAct 和普通 Tool Calling 的区别** —— 为什么说 ReAct 是 Tool Calling 的"升级版"
3. **掌握 ReAct 的实现细节** —— 能从零实现一个 ReAct Agent
4. **了解其他推理模式** —— Plan-and-Execute、Reflexion、LATS 的原理和适用场景
5. **能根据场景选择合适的推理模式** —— 不同模式各适用于什么场景？

## 核心问题

1. **ReAct 模式为什么能让 Agent 做出更好的决策？** 推理步骤到底起了什么作用？
2. **ReAct 和普通 Tool Calling 的区别是什么？** 为什么说 ReAct 是 Tool Calling 的"升级版"？
3. **什么时候应该用 ReAct？什么时候应该用其他模式？** 不同模式的适用场景是什么？

---

## 7.1 从一个故事开始

### 7.1.1 一个没有"思考"的 Agent

想象你有一个助手，你问他："北京天气怎么样？"他的工作方式是这样的：

你问他，他直接拿起电话打给气象台，问了天气，然后告诉你结果。

这个助手很勤快，但他有一个问题——他不会"想"。每次你问他问题，他都是直接行动，不会先分析一下这个问题需不需要行动、应该用什么方式行动。

如果有一天你问他："1+1等于几？"他可能还是会拿起电话打给气象台，问"1+1等于几"。气象台的人会觉得他很奇怪，但他自己不知道——因为他没有"思考"这个步骤。

这就是没有 ReAct 的 Agent。它能调用工具，但它不会"想"为什么要调用这个工具、应该调用哪个工具、调用之后该怎么处理结果。

### 7.1.2 一个会"思考"的 Agent

现在，我们给这个助手加上"思考"的能力。同样是你问他"北京天气怎么样？"他的工作方式变成了：

他在心里想："用户在问北京的天气。我需要获取准确的天气信息，不能瞎猜。我应该调用天气查询工具。"然后他拿起电话打给气象台，获取了天气信息。获取之后，他又想："我已经拿到了天气数据，现在可以回答用户了。用户可能还想知道适不适合出门，我可以补充一下。"然后他告诉你："北京今天晴，温度 25°C，适合外出。"

这个助手多了一个"思考"的步骤。这个步骤看起来很简单，但它带来了巨大的变化：他能理解用户真正想要什么，能选择合适的工具，能在获取结果后给出更好的回答。

这就是 ReAct 的核心思想：**让 Agent 在行动前先思考。**

---

## 7.2 ReAct 的三个步骤

### 7.2.1 Thought（思考）

Thought 是 Agent 的"内心独白"。在这一步，Agent 分析当前情况，决定下一步行动。

让我举一个具体的例子。假设用户问："帮我查一下 Python 最新版本，然后告诉我它比 3.10 新了多少。"

Agent 的第一轮 Thought 可能是这样的：

```
Thought: 用户想知道两件事：Python 最新版本是什么，以及它和 3.10 的差值。
我需要先搜索 Python 的最新版本，然后计算差值。
我应该先调用搜索工具。
```

这个 Thought 做了三件事：理解了用户的意图（两件事），规划了行动步骤（先搜索再计算），选择了工具（搜索工具）。

如果没有 Thought，Agent 可能直接调用计算器，试图计算"Python 最新版本 - 3.10"，这显然会失败。

### 7.2.2 Action（行动）

Action 是 Agent 实际执行的操作。在 ReAct 中，Action 通常是调用一个工具，但也可能是直接给出答案。

```python
# 调用工具的 Action
Action: [search_web(query="Python latest version 2024")]

# 直接回答的 Action
Answer: Python 最新版本是 3.13.1
```

注意 Action 的格式。在我们之前的章节中，我们学过 Tool Calling。ReAct 中的 Action 本质上就是 Tool Calling，但它有一个重要的区别——Action 是在 Thought 之后执行的，而不是直接从用户输入中推断出来的。

### 7.2.3 Observation（观察）

Observation 是 Action 的结果。Agent 执行了 Action 之后，会获取一个结果，这个结果就是 Observation。

```
Observation: Python 3.13.1，于 2024 年 10 月发布，主要新特性包括性能提升、错误消息改进等。
```

Observation 会被反馈给 Agent，作为下一轮 Thought 的输入。Agent 会基于 Observation 继续思考和行动。

### 7.2.4 一个完整的 ReAct 过程

让我们把这三个步骤串起来，看看一个完整的 ReAct 过程是什么样的：

```
用户：帮我查一下 Python 最新版本，然后告诉我它比 3.10 新了多少。

第 1 轮：
Thought: 用户想知道 Python 最新版本和 3.10 的差值。我需要先搜索最新版本。
Action: [search_web(query="Python latest version")]
Observation: Python 3.13.1，2024年10月发布

第 2 轮：
Thought: 我已经知道最新版本是 3.13.1。现在需要计算 3.13.1 - 3.10。
Action: [calculator(expression="3.13.1 - 3.10")]
Observation: 0.031

第 3 轮：
Thought: 我已经获取了所有信息。Python 最新版本是 3.13.1，比 3.10 新了 0.031 个版本号。
但这不是一个有意义的比较方式。我应该告诉用户 3.13.1 相比 3.10 有哪些新特性。
Answer: Python 最新版本是 3.13.1（2024年10月发布）。相比 3.10，主要改进包括：性能提升约 5%、更好的错误消息、新的类型注解语法等。
```

注意第 3 轮的 Thought。Agent 不是简单地回答"0.031"，而是思考了一下——版本号的差值没有太大意义，用户真正想知道的是新版本有什么改进。这就是 Thought 的价值：它让 Agent 能做出更好的决策。

---

## 7.3 ReAct vs 普通 Tool Calling

### 7.3.1 普通 Tool Calling 的工作方式

在普通 Tool Calling 中，LLM 直接决定调用哪个工具：

```
用户：北京天气怎么样？
LLM：（直接决定）应该调用天气工具
Action: [get_weather(city="北京")]
```

这种方式简单直接，但有一个问题——LLM 是"直觉性"地选择工具，没有经过深思熟虑。在简单场景下这没问题，但在复杂场景下可能会出错。

### 7.3.2 ReAct 的工作方式

在 ReAct 中，LLM 先思考再行动：

```
用户：北京天气怎么样？
Thought: 用户在问天气。我需要获取准确的天气数据，不能瞎猜。天气工具可以提供实时天气信息。
Action: [get_weather(city="北京")]
```

多了 Thought 这一步，LLM 的决策就从"直觉"变成了"推理"。这让它在复杂场景下更可靠。

### 7.3.3 什么时候差异最明显

差异最明显的场景是需要多步推理的任务。比如用户问："帮我比较北京和上海的天气，看哪个更适合周末出游。"

**普通 Tool Calling：**

```
LLM：（可能直接调用一次天气工具，只获取一个城市的天气）
Action: [get_weather(city="北京")]
结果：北京晴，25°C
LLM：北京今天晴，温度 25°C。（没有比较，没有建议）
```

**ReAct：**

```
Thought: 用户想比较北京和上海的天气，看哪个更适合出游。我需要获取两个城市的天气数据。
Action: [get_weather(city="北京")]
Observation: 北京晴，25°C，湿度 40%

Thought: 我已经获取了北京的天气。现在需要获取上海的天气。
Action: [get_weather(city="上海")]
Observation: 上海多云，22°C，湿度 70%

Thought: 我已经获取了两个城市的天气。北京晴朗但稍热，上海凉爽但多云。周末出游的话，晴天更适合户外活动，所以北京更好。
Answer: 北京今天晴，25°C，湿度 40%；上海多云，22°C，湿度 70%。周末出游的话，建议选择北京，因为晴天更适合户外活动。
```

这就是 ReAct 的威力——它能进行多步推理，综合多个信息源，给出更好的回答。

---

## 7.4 ReAct 的实现

### 7.4.1 完整代码实现

让我从零开始实现一个 ReAct Agent。我会逐行解释每部分代码的作用。

```python
import re
import json
from openai import OpenAI


class ReActAgent:
    """一个完整的 ReAct Agent"""

    def __init__(self, api_key: str, tools: dict):
        """
        初始化 ReAct Agent

        参数:
            api_key: OpenAI API Key
            tools: 工具字典，格式为 {"工具名": 工具函数}
        """
        self.client = OpenAI(api_key=api_key)
        self.tools = tools

    def run(self, question: str, max_steps: int = 5) -> str:
        """
        运行 ReAct Agent

        参数:
            question: 用户问题
            max_steps: 最大步数（防止无限循环）

        返回:
            最终答案
        """
        # 构建初始 Prompt
        prompt = self._build_initial_prompt(question)

        for step in range(max_steps):
            # 调用 LLM
            response = self.client.chat.completions.create(
                model="gpt-4o",
                messages=[{"role": "user", "content": prompt}],
                temperature=0.3,  # 低温度，更确定性的推理
                max_tokens=500,
            )

            output = response.choices[0].message.content

            # 解析输出
            thought = self._extract_tag(output, "Thought")
            action = self._extract_tag(output, "Action")
            answer = self._extract_tag(output, "Answer")

            # 如果有最终答案，返回
            if answer:
                return answer

            # 如果有行动，执行
            if action:
                # 解析工具名和参数
                tool_name, args = self._parse_action(action)

                # 执行工具
                observation = self._execute_tool(tool_name, args)

                # 更新 Prompt，加入这一轮的结果
                prompt += f"\n\nThought: {thought}"
                prompt += f"\nAction: {action}"
                prompt += f"\nObservation: {observation}"

        # 达到最大步数
        return "达到最大步数，任务未完成"

    def _build_initial_prompt(self, question: str) -> str:
        """构建初始 Prompt"""
        # 获取工具描述
        tool_descriptions = []
        for name, fn in self.tools.items():
            doc = fn.__doc__ or "无描述"
            tool_descriptions.append(f"- {name}: {doc}")
        tools_text = "\n".join(tool_descriptions)

        return f"""你是一个 ReAct Agent。请按照以下格式思考和行动：

Thought: [你的推理过程，分析当前情况，决定下一步]
Action: [tool_name(param1=value1, param2=value2)]
Observation: [工具返回的结果，由系统自动填充]
... (可以重复多次 Thought → Action → Observation)
Thought: [最终推理，综合所有信息]
Answer: [最终答案]

可用工具：
{tools_text}

重要提示：
- 每次只执行一个 Action
- Observation 由系统自动填充，你不需要自己写
- 当你有了足够的信息可以回答用户时，输出 Answer

问题：{question}"""

    def _extract_tag(self, text: str, tag: str) -> str | None:
        """从文本中提取指定标签的内容"""
        match = re.search(rf"{tag}: (.+?)(?:\n|$)", text)
        return match.group(1).strip() if match else None

    def _parse_action(self, action: str) -> tuple[str, dict]:
        """解析 Action，提取工具名和参数"""
        match = re.match(r"(\w+)\((.+)\)", action)
        if not match:
            return action, {}

        tool_name = match.group(1)
        args_str = match.group(2)

        # 解析参数
        args = {}
        if args_str:
            for param in args_str.split(","):
                if "=" in param:
                    key, value = param.split("=", 1)
                    args[key.strip()] = value.strip().strip("'\"")

        return tool_name, args

    def _execute_tool(self, tool_name: str, args: dict) -> str:
        """执行工具并返回结果"""
        tool_fn = self.tools.get(tool_name)
        if not tool_fn:
            return f"错误：工具 '{tool_name}' 不存在。可用工具：{list(self.tools.keys())}"

        try:
            result = tool_fn(**args)
            return str(result)
        except Exception as e:
            return f"工具执行失败：{e}"
```

### 7.4.2 代码逐行解析

让我详细解释代码中的关键部分。

**初始 Prompt 的设计**

初始 Prompt 是 ReAct Agent 的"灵魂"。它告诉 LLM 应该按照什么格式思考和行动。Prompt 中的关键要素包括格式说明（Thought、Action、Observation 的格式）、工具列表（让 LLM 知道有哪些工具可用）、和重要提示（每次只执行一个 Action，Observation 由系统填充等）。

**输出解析**

LLM 的输出是自由文本，我们需要用正则表达式提取 Thought、Action、Answer。这是一个简化的实现。实际项目中，你可能需要更复杂的解析逻辑来处理各种边界情况。

**工具执行**

工具执行后，结果会被添加到 Prompt 中，作为下一轮 Thought 的输入。这就是 ReAct 的"循环"——每一轮的结果都会影响下一轮的决策。

### 7.4.3 运行示例

```python
# 定义工具
def search_web(query: str) -> str:
    """搜索互联网获取信息"""
    # 实际项目中调用搜索 API
    return f"搜索结果：关于 '{query}' 的最新信息..."

def calculator(expression: str) -> str:
    """执行数学计算"""
    try:
        return str(eval(expression))
    except Exception as e:
        return f"计算错误：{e}"

# 创建 Agent
agent = ReActAgent(
    api_key="your-api-key",
    tools={"search_web": search_web, "calculator": calculator},
)

# 运行
result = agent.run("Python 最新版本是什么？它的版本号乘以 2 等于多少？")
print(result)
```

**输出过程：**

```
Thought: 用户想知道两件事：Python 最新版本，以及版本号乘以 2 的结果。我需要先搜索最新版本。
Action: [search_web(query="Python latest version")]
Observation: Python 3.13.1，2024年10月发布

Thought: 我知道最新版本是 3.13.1。现在需要计算 3.13.1 * 2。
Action: [calculator(expression="3.13.1 * 2")]
Observation: 6.262

Thought: 我已经获取了所有信息。Python 最新版本是 3.13.1，乘以 2 等于 6.262。
Answer: Python 最新版本是 3.13.1（2024年10月发布）。3.13.1 × 2 = 6.262。
```

---

## 7.5 其他推理模式

### 7.5.1 Plan-and-Execute（先规划后执行）

Plan-and-Execute 的核心思想是：先制定完整的计划，然后逐步执行。这和 ReAct 不同——ReAct 是"边想边做"，而 Plan-and-Execute 是"先想好再做"。

**一个 Plan-and-Execute 的例子：**

```
用户：帮我写一篇关于 AI Agent 的技术博客

Plan:
1. 搜索 AI Agent 的最新进展（5分钟）
2. 搜索 AI Agent 的应用案例（5分钟）
3. 整理文章大纲（3分钟）
4. 撰写文章初稿（15分钟）
5. 审查和修改（5分钟）

Execute:
Step 1: [search_web("AI Agent latest progress 2024")]
结果: ...

Step 2: [search_web("AI Agent application cases")]
结果: ...

Step 3: [generate_outline(research_results)]
结果: 大纲...

Step 4: [write_article(outline, research_results)]
结果: 初稿...

Step 5: [review_and_edit(draft)]
结果: 最终文章...
```

**Plan-and-Execute 的实现：**

```python
class PlanAndExecuteAgent:
    def __init__(self, llm_client, tools: dict):
        self.llm = llm_client
        self.tools = tools

    def run(self, task: str) -> str:
        # 第一阶段：制定计划
        plan = self._create_plan(task)

        # 第二阶段：逐步执行
        results = []
        for i, step in enumerate(plan):
            print(f"执行步骤 {i+1}/{len(plan)}: {step}")
            result = self._execute_step(step, results)
            results.append({"step": step, "result": result})

        # 第三阶段：综合结果
        return self._synthesize(task, results)

    def _create_plan(self, task: str) -> list[str]:
        """让 LLM 制定计划"""
        prompt = f"""请为以下任务制定详细的执行计划。

任务：{task}

请以编号列表的形式输出计划，每个步骤应该具体、可执行。
例如：
1. 搜索相关信息
2. 分析搜索结果
3. 生成报告"""

        response = self.llm.chat(
            messages=[{"role": "user", "content": prompt}]
        )

        # 解析计划
        steps = []
        for line in response.split("\n"):
            line = line.strip()
            if line and line[0].isdigit():
                # 去掉编号
                step = re.sub(r"^\d+[\.\)]\s*", "", line)
                steps.append(step)

        return steps

    def _execute_step(self, step: str, previous_results: list) -> str:
        """执行单个步骤"""
        # 构建上下文
        context = "之前步骤的结果：\n"
        for r in previous_results:
            context += f"- {r['step']}: {r['result'][:200]}\n"

        prompt = f"""请执行以下步骤：

步骤：{step}
{context}

请调用合适的工具来完成这个步骤。"""

        response = self.llm.chat(
            messages=[{"role": "user", "content": prompt}]
        )

        # 这里可以解析工具调用并执行
        return response

    def _synthesize(self, task: str, results: list) -> str:
        """综合所有结果"""
        results_text = "\n".join([
            f"步骤 {i+1}: {r['step']}\n结果: {r['result']}"
            for i, r in enumerate(results)
        ])

        prompt = f"""请基于以下步骤的结果，完成最终任务。

任务：{task}

各步骤结果：
{results_text}

请生成最终输出。"""

        response = self.llm.chat(
            messages=[{"role": "user", "content": prompt}]
        )

        return response
```

**Plan-and-Execute 什么时候比 ReAct 好？**

当任务可以预先规划时，Plan-and-Execute 比 ReAct 更好。比如"写一篇博客"这样的任务，步骤是相对固定的：搜索→整理→写作→审查。Plan-and-Execute 可以一次性规划好所有步骤，然后逐步执行，效率更高。

而 ReAct 更适合步骤不确定的任务。比如"帮我解决这个问题"，你不知道需要多少步、每步做什么，需要边做边想。

### 7.5.2 Reflexion（自我反思）

Reflexion 的核心思想是：在行动后进行反思，从错误中学习。

**一个 Reflexion 的例子：**

```
用户：计算 123 的阶乘

第 1 次尝试：
Action: [calculator(expression="123!")]
Observation: 计算超时（数字太大）

Reflection: 直接计算 123 的阶乘会导致超时。我应该使用近似方法，比如用对数来计算。

第 2 次尝试：
Action: [calculator(expression="sum(log(i) for i in range(1, 124))")]
Observation: 219.48

Reflection: ln(123!) ≈ 219.48，所以 123! ≈ e^219.48 ≈ 2.3×10^95

最终答案: 123! ≈ 2.3×10^95
```

Reflexion 的关键在于 Reflection 这一步——Agent 不是简单地重试，而是分析了失败的原因（数字太大），并提出了改进方案（使用对数近似）。

**Reflexion 的实现：**

```python
class ReflexionAgent:
    def __init__(self, llm_client, tools: dict, max_reflections: int = 3):
        self.llm = llm_client
        self.tools = tools
        self.max_reflections = max_reflections

    def run(self, task: str) -> str:
        history = []  # 记录历史尝试和反思

        for round_num in range(self.max_reflections):
            # 执行任务
            result = self._attempt(task, history)

            # 检查结果是否满意
            if self._is_satisfactory(result):
                return result

            # 反思
            reflection = self._reflect(task, result, history)
            history.append({
                "attempt": result,
                "reflection": reflection,
            })

        return "无法完成任务"

    def _attempt(self, task: str, history: list) -> str:
        """尝试执行任务"""
        # 构建包含历史反思的 Prompt
        prompt = f"任务：{task}\n"

        if history:
            prompt += "之前的尝试和反思：\n"
            for h in history:
                prompt += f"尝试：{h['attempt']}\n"
                prompt += f"反思：{h['reflection']}\n\n"

        prompt += "请尝试完成任务。"

        response = self.llm.chat(
            messages=[{"role": "user", "content": prompt}]
        )
        return response

    def _reflect(self, task: str, result: str, history: list) -> str:
        """反思"""
        prompt = f"""请反思以下任务的执行结果。

任务：{task}
执行结果：{result}
之前的反思：{[h['reflection'] for h in history]}

请分析：
1. 这次执行有什么问题？
2. 问题的根本原因是什么？
3. 下次应该如何改进？"""

        response = self.llm.chat(
            messages=[{"role": "user", "content": prompt}]
        )
        return response

    def _is_satisfactory(self, result: str) -> str | None:
        """检查结果是否满意"""
        # 简单实现：检查是否包含错误信息
        if "错误" in result or "失败" in result or "超时" in result:
            return False
        return True
```

### 7.5.3 LATS（Language Agent Tree Search）

LATS 将蒙特卡洛树搜索（MCTS）与 LLM 结合。它不是线性地尝试一种方案，而是同时探索多种可能的行动路径，然后选择最好的。

```
           Root (任务)
          /    |    \
      Action1 Action2 Action3
      /  \    |     /  \
    R1a  R1b  R2   R3a  R3b
    |         |         |
   最终结果1 最终结果2 最终结果3
   
选择：评估所有最终结果，选择最好的
```

LATS 适用于有明确评估标准的复杂优化问题，但计算成本很高。

---

## 7.6 推理模式对比

| 模式 | 核心思想 | 优点 | 缺点 | 适用场景 | 成本 |
|------|---------|------|------|---------|------|
| ReAct | 边想边做 | 可解释、可调试 | 每步都要调用LLM | 需要推理的任务 | 中 |
| Plan-and-Execute | 先想好再做 | 全局规划、效率高 | 计划可能过时 | 步骤可预先确定的任务 | 中 |
| Reflexion | 从错误中学习 | 能自我修正 | 需要额外LLM调用 | 容错要求高的任务 | 高 |
| LATS | 探索多种路径 | 找到最优解 | 计算成本极高 | 复杂优化问题 | 很高 |

### 选择指南

```
1. 任务需要推理吗？
   ├── 否 → 直接使用 Tool Calling（不需要推理模式）
   └── 是 → 2

2. 任务步骤可以预先确定吗？
   ├── 是 → Plan-and-Execute（效率更高）
   └── 否 → 3

3. 任务需要从错误中学习吗？
   ├── 是 → Reflexion（能自我修正）
   └── 否 → ReAct（通用推理模式）
```

---

## 7.7 常见坑

### 7.7.1 ReAct 步骤过多

问题：每步都调用 LLM，成本高、延迟大。用户可能等不及。

解决：设置合理的 max_steps（建议 5-10 步），监控步骤数。如果超过阈值，强制结束并返回当前最佳结果。

### 7.7.2 推理错误传播

问题：一步推理错误会影响后续所有步骤。如果第一轮的 Thought 就错了，后面的步骤都会基于错误的前提。

解决：加入 Reflexion 机制，定期检查和修正。或者在每一步都进行结果验证。

### 7.7.3 Plan-and-Execute 计划过时

问题：执行过程中情况变化，计划需要更新。比如搜索结果和预期不同，需要调整后续步骤。

解决：在执行过程中定期重新评估计划。如果发现计划不再适用，重新制定计划。

### 7.7.4 Prompt 格式不稳定

问题：LLM 有时不按照 Thought/Action/Answer 的格式输出，导致解析失败。

解决：在 Prompt 中加入 Few-shot 示例来稳定格式；使用更健壮的解析逻辑（支持多种格式变体）；当解析失败时，将 LLM 输出作为最终回答返回。

---

## 7.8 推理模式的工程实践

### 7.8.1 如何选择推理模式

在实际项目中，选择合适的推理模式是一个需要权衡的决策。以下是几个关键的考量维度。

**任务的可预测性**：如果任务的步骤是固定的（比如"搜索-分析-报告"这种模式），Plan-and-Execute 是更好的选择，因为你可以一次性规划好所有步骤。如果任务的步骤取决于中间结果（比如"搜索一下，根据搜索结果决定下一步"），ReAct 更合适。

**对延迟的容忍度**：ReAct 每一步都要调用 LLM，延迟较高。如果用户需要快速响应，Plan-and-Execute 因为可以批量执行可能更快。对于时间敏感的场景，也可以考虑将 ReAct 的多步推理浓缩到单次 LLM 调用中。

**对准确性的要求**：如果任务需要很高的准确性（比如代码生成、法律文书），Reflexion 的自我纠错能力很有价值。如果可以容忍一定误差，标准的 ReAct 就够了。

### 7.8.2 混合推理模式

在实际项目中，我们经常需要组合多种推理模式。比如，一个复杂的研究任务可能先用 Plan-and-Execute 制定总体计划，然后在执行每个步骤时用 ReAct 进行详细的推理，最后用 Reflexion 对整体结果进行检查和修正。

```python
class HybridReasoningAgent:
    """混合推理模式 Agent"""

    def __init__(self, llm_client, tools: dict):
        self.llm = llm_client
        self.tools = tools

    def run(self, task: str) -> str:
        # 阶段 1：Plan-and-Execute 制定计划
        plan = self._create_plan(task)
        print(f"制定计划：{len(plan)} 个步骤")

        # 阶段 2：用 ReAct 执行每个步骤
        results = []
        for i, step in enumerate(plan):
            print(f"\n执行步骤 {i+1}: {step}")
            step_result = self._execute_with_react(step)
            results.append({"step": step, "result": step_result})

        # 阶段 3：综合结果
        answer = self._synthesize(task, results)

        # 阶段 4：Reflexion 检查
        reflection = self._reflect(task, answer)
        if "需要改进" in reflection:
            answer = self._improve(task, answer, reflection)

        return answer
```

### 7.8.3 推理过程的可视化

为了调试和改进 Agent 的推理过程，将推理过程可视化是非常有帮助的。

```python
def visualize_react_trace(trace: list[dict]):
    """可视化 ReAct 执行轨迹"""
    print("=" * 60)
    print("ReAct 执行轨迹")
    print("=" * 60)

    for i, step in enumerate(trace):
        step_num = i + 1
        print(f"\n--- 第 {step_num} 轮 ---")

        if "thought" in step:
            print(f"  [思考] {step['thought']}")

        if "action" in step:
            print(f"  [行动] 调用 {step['action']['tool']}")
            print(f"         参数: {step['action']['args']}")

        if "observation" in step:
            result = step['observation'][:150]
            print(f"  [观察] {result}...")

    print(f"\n{'=' * 60}")
```

---

## 7.8 练习题

### 概念理解题

1. **ReAct 和普通 Tool Calling 的区别是什么？** 请用一个具体例子说明，为什么 ReAct 能做出更好的决策。

2. **ReAct 的三个步骤（Thought、Action、Observation）分别起什么作用？** 如果去掉其中一个步骤，会有什么问题？

3. **Plan-and-Execute 和 ReAct 的区别是什么？** 各适用于什么场景？

4. **Reflexion 如何帮助 Agent 从错误中学习？** 它的 Reflection 步骤做了什么？

5. **LATS 的原理是什么？** 它为什么计算成本很高？

### 动手实践题

1. **实现 ReAct Agent：** 用本章的代码实现一个 ReAct Agent，测试 5 个不同的问题。

2. **对比实验：** 对同一个问题，分别用 ReAct 和普通 Tool Calling 解决，对比输出质量。

3. **Reflexion 实验：** 故意让 Agent 犯错（比如调用一个会失败的工具），观察 Reflexion 如何修正。

### 思考题

1. ReAct 的"思考"步骤是否真的在"思考"？它和人类的思考有什么区别？

2. 如果 ReAct 的 Thought 步骤输出了错误的推理，Agent 会怎么处理？如何防止这种情况？

3. 多个推理模式可以结合使用吗？比如先 Plan-and-Execute 制定计划，然后用 ReAct 执行每一步？

---

## 7.9 实战任务

**任务：实现一个 ReAct Agent，完成多步推理任务**

**具体要求：**

1. 实现完整的 ReAct 循环（Thought → Action → Observation）
2. 支持至少 3 个工具（搜索、计算、摘要）
3. 支持多步推理（至少 3 步）
4. 记录每一步的 Thought、Action、Observation（打印出来）
5. 测试 5 个不同的问题，分析 Agent 的推理过程

**评估标准：**

| 维度 | 权重 | 评分标准 |
|------|------|---------|
| ReAct 实现 | 30% | Thought→Action→Observation 循环正确 |
| 工具调用 | 25% | 能正确选择和调用工具 |
| 推理质量 | 20% | Thought 合理、有逻辑、能指导行动 |
| 错误处理 | 15% | 能处理工具调用失败 |
| 代码质量 | 10% | 结构清晰，可维护 |

---

## 7.10 本章小结

### 核心知识点回顾

- **ReAct 的核心思想：** 让 Agent 在行动前先思考。Thought → Action → Observation 的循环让 Agent 的决策从"直觉"变成"推理"。

- **ReAct 的优势：** 可解释性（能看到推理过程）、可调试性（能定位问题）、准确性（推理后行动更准确）。

- **Plan-and-Execute：** 先制定完整计划，然后逐步执行。适合步骤可以预先确定的复杂任务。

- **Reflexion：** 在行动后进行反思，从错误中学习。适合容错要求高的任务。

- **LATS：** 将树搜索与 LLM 结合，探索多种路径。适合复杂优化问题，但成本很高。

### 关键公式

```
ReAct = Reasoning（推理） + Acting（行动） + Observation（观察）
ReAct 循环：while not done: thought → action → observation
```

### 下一章预告

下一章我们将探讨 **Agent 的状态管理** —— 对话历史与上下文。你将学习如何管理 Agent 的对话历史，如何在长对话中保持上下文一致性，以及如何优化上下文窗口的使用。

---

*上一章：[第 6 章：Agent 的分类与适用场景](../chapter-06/README.md)*
*下一章：[第 8 章：Agent 的状态管理](../chapter-08/README.md)*
