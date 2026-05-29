# 第 17 章：Agent 推理模式深入 —— 从 ReAct 到 Reflexion

---

## 学习目标

完成本章学习后，你将能够：

1. 深入理解 ReAct 模式的核心思想——推理与行动如何交替进行，以及为什么这种方式比单纯的推理或行动更有效
2. 掌握 Reflexion 自我反思机制的原理——Agent 如何从错误中学习，并在后续尝试中避免重复犯错
3. 了解 Plan-and-Execute 模式——先规划后执行的结构化推理方式
4. 能够从零实现基于 ReAct 和 Reflexion 的 Agent
5. 理解不同推理模式对 Agent 性能的影响，学会为不同场景选择最合适的推理模式
6. 掌握混合推理模式的设计——根据任务复杂度自动切换推理策略

## 核心问题

1. ReAct 模式为什么比纯推理或纯行动更有效？
2. Reflexion 如何让 Agent 从错误中学习？
3. 如何选择合适的推理模式？

---

## 17.1 推理模式概览

### 17.1.1 为什么推理模式重要

在前面的章节中，我们学习了 Agent 的记忆、规划等能力。但还有一个关键问题没有解决：Agent 在执行任务时，到底是怎么"思考"的？

最简单的 Agent 不做任何推理——给它一个任务，它直接生成一个回答，不管对不对。稍微好一点的 Agent 会做一点"思考"，但这种思考是隐式的、无结构的。

推理模式（Reasoning Pattern）就是给 Agent 的思考过程定义一个清晰的框架。有了框架，Agent 的思考就不再是"想到哪写到哪"，而是有章可循的、可重复的、可优化的。

打个比方：如果把 Agent 比作一个侦探，推理模式就是侦探的"破案方法论"。有的侦探喜欢先收集所有线索再推理（Plan-and-Execute），有的侦探喜欢边查线索边推理（ReAct），有的侦探会在破案失败后反思总结经验（Reflexion）。

### 17.1.2 三种主要推理模式的对比

| 模式 | 核心思想 | 优势 | 劣势 | 适用场景 |
|------|----------|------|------|----------|
| ReAct | 推理 + 行动交替 | 灵活、可解释、实时调整 | 可能陷入循环、Token 消耗大 | 需要外部信息的问答、工具调用 |
| Reflexion | 执行 + 反思 + 改进 | 能从错误中学习、持续改进 | 需要多次尝试、成本高 | 需要精确答案的任务、代码编写 |
| Plan-and-Execute | 先规划后执行 | 结构化、可控、高效 | 缺乏灵活性、不适应变化 | 结构明确的多步骤任务 |

---

## 17.2 ReAct 模式

### 17.2.1 ReAct 的核心思想

ReAct（Reasoning + Acting）是目前最流行的 Agent 推理模式。它的核心思想可以用一句话概括：**每一步都先想一想（Reasoning），然后做一做（Acting），再看看结果（Observation），再想一想……**

这个循环看起来很简单，但效果出奇地好。原因在于：

1. **推理让行动更有针对性**：Agent 不是盲目地执行工具调用，而是先思考"我需要什么信息"，再决定"应该调用什么工具"
2. **观察让推理更准确**：每一步行动的结果都会反馈给下一步的推理，让 Agent 能根据实际情况调整策略
3. **过程是可解释的**：每一步的推理过程都是可见的，方便调试和理解

```
ReAct 的典型循环：

Thought: 我需要找到用户问的这个问题的答案。让我先搜索一下。
Action: search("Python 异步编程最佳实践")
Observation: 搜索结果返回了 5 篇相关文章...

Thought: 搜索结果显示了一些好文章，但我需要更具体的信息。让我查一下官方文档。
Action: lookup("asyncio official documentation")
Observation: Python 官方文档中关于 asyncio 的说明...

Thought: 现在我有了足够的信息，可以回答用户的问题了。
Action: respond("Python 异步编程的最佳实践包括...")
```

### 17.2.2 ReAct Agent 的完整实现

```python
import re
import json
import time
from typing import Callable


class ReActAgent:
    """
    ReAct 推理模式的 Agent
    
    核心循环：Thought → Action → Observation → Thought → ...
    
    每一步：
    1. Thought：Agent 思考当前情况，决定下一步行动
    2. Action：Agent 调用工具执行操作
    3. Observation：Agent 观察工具返回的结果
    4. 重复直到得出最终答案
    """
    
    def __init__(self, llm_client, tools: dict[str, Callable],
                 max_iterations: int = 10, verbose: bool = True):
        """
        参数:
            llm_client: LLM 客户端，需要有 generate(prompt) 方法
            tools: 工具字典，格式为 {tool_name: tool_function}
            max_iterations: 最大循环次数（防止无限循环）
            verbose: 是否打印详细的执行过程
        """
        self.llm_client = llm_client
        self.tools = tools
        self.max_iterations = max_iterations
        self.verbose = verbose
        self.execution_trace: list[dict] = []  # 执行轨迹
    
    def run(self, query: str) -> dict:
        """
        执行 ReAct 循环
        
        参数:
            query: 用户的问题或任务
        
        返回:
            包含答案和执行轨迹的字典
        """
        tools_desc = self._format_tools_description()
        
        # 初始 Prompt
        system_prompt = f"""你是一个智能助手，使用 ReAct 模式来解决问题。

你可以使用以下工具：
{tools_desc}

请严格按照以下格式思考和行动：
Thought: <你的思考过程，分析当前情况，决定下一步行动>
Action: <工具名称>(<参数>)
Observation: <工具返回的结果（由系统自动填入，你不需要写这部分）>
... (可以重复多轮 Thought → Action → Observation)
Thought: <最终思考，分析所有收集到的信息>
Final Answer: <最终答案>

重要规则：
1. 每次只执行一个 Action
2. Action 的参数必须是工具实际支持的
3. 如果工具调用失败，分析失败原因并尝试其他方法
4. 不要编造工具不存在的功能

用户问题：{query}

开始："""

        current_prompt = system_prompt
        self.execution_trace = []
        
        for iteration in range(self.max_iterations):
            if self.verbose:
                print(f"\n--- 迭代 {iteration + 1} ---")
            
            # 调用 LLM 生成下一步
            response = self.llm_client.generate(current_prompt)
            
            # 检查是否有最终答案
            if "Final Answer:" in response:
                answer = self._extract_final_answer(response)
                self._log_trace("final", response, answer)
                
                if self.verbose:
                    print(f"\n最终答案: {answer}")
                
                return {
                    "answer": answer,
                    "trace": self.execution_trace,
                    "iterations": iteration + 1,
                    "success": True,
                }
            
            # 解析 Thought 和 Action
            thought = self._extract_thought(response)
            action = self._parse_action(response)
            
            if action:
                tool_name, tool_input = action
                self._log_trace("action", response, {"tool": tool_name, "input": tool_input})
                
                if self.verbose:
                    print(f"Thought: {thought}")
                    print(f"Action: {tool_name}({tool_input})")
                
                # 执行工具
                if tool_name in self.tools:
                    try:
                        observation = self.tools[tool_name](tool_input)
                    except Exception as e:
                        observation = f"工具执行出错: {str(e)}"
                else:
                    observation = f"错误：工具 '{tool_name}' 不存在。可用工具: {list(self.tools.keys())}"
                
                self._log_trace("observation", observation, observation)
                
                if self.verbose:
                    print(f"Observation: {observation}")
                
                # 更新 Prompt
                current_prompt += response + f"\nObservation: {observation}\n"
            else:
                # 没有解析到 Action，可能是格式问题
                self._log_trace("error", response, "未能解析出 Action")
                current_prompt += response + "\nThought: "
        
        # 达到最大迭代次数
        return {
            "answer": "达到最大迭代次数，未能得出最终答案",
            "trace": self.execution_trace,
            "iterations": self.max_iterations,
            "success": False,
        }
    
    def _format_tools_description(self) -> str:
        """格式化工具描述"""
        lines = []
        for name, tool in self.tools.items():
            doc = tool.__doc__ or "无描述"
            # 提取函数签名
            import inspect
            sig = inspect.signature(tool)
            params = ", ".join(
                f"{p.name}" + (f"={p.default}" if p.default != inspect.Parameter.empty else "")
                for p in sig.parameters.values()
            )
            lines.append(f"- {name}({params}): {doc.strip()}")
        return "\n".join(lines)
    
    def _extract_thought(self, response: str) -> str:
        """从响应中提取 Thought"""
        match = re.search(r'Thought:\s*(.+?)(?:\nAction:|\Z)', response, re.DOTALL)
        return match.group(1).strip() if match else ""
    
    def _parse_action(self, response: str) -> tuple[str, str] | None:
        """从响应中解析 Action"""
        # 匹配 Action: tool_name(arguments) 格式
        match = re.search(r'Action:\s*(\w+)\((.+?)\)', response, re.DOTALL)
        if match:
            tool_name = match.group(1).strip()
            tool_input = match.group(2).strip().strip('"\'')
            return tool_name, tool_input
        return None
    
    def _extract_final_answer(self, response: str) -> str:
        """提取最终答案"""
        if "Final Answer:" in response:
            return response.split("Final Answer:")[-1].strip()
        return response
    
    def _log_trace(self, step_type: str, raw: str, parsed):
        """记录执行轨迹"""
        self.execution_trace.append({
            "step_type": step_type,
            "raw_output": raw[:500] if isinstance(raw, str) else str(raw)[:500],
            "parsed": parsed,
            "timestamp": time.time(),
        })
```

### 17.2.3 ReAct 的运行示例

```python
def demo_react_agent():
    """演示 ReAct Agent 的运行"""
    
    # 定义工具
    def search(query: str) -> str:
        """搜索互联网获取信息"""
        # 模拟搜索结果
        results = {
            "python async": "Python 异步编程主要使用 asyncio 模块，支持 async/await 语法。",
            "langchain": "LangChain 是一个用于构建 LLM 应用的开源框架。",
        }
        for key, value in results.items():
            if key in query.lower():
                return value
        return f"搜索 '{query}' 的结果：未找到相关信息"
    
    def calculator(expression: str) -> str:
        """计算数学表达式"""
        try:
            # 安全的数学计算
            import ast
            tree = ast.parse(expression, mode='eval')
            return str(eval(compile(tree, '<calc>', 'eval')))
        except Exception as e:
            return f"计算错误: {e}"
    
    # 模拟 LLM
    class MockLLM:
        def __init__(self):
            self.responses = [
                'Thought: 用户问的是 LangChain 是什么，让我搜索一下。\nAction: search("langchain")',
                'Thought: 搜索结果显示 LangChain 是一个 LLM 应用框架。让我再补充一些细节。\nAction: search("langchain features")',
                'Thought: 现在我有足够的信息来回答了。\nFinal Answer: LangChain 是一个用于构建 LLM 应用的开源框架，提供了 Prompt 模板、链式调用、Agent 等核心组件。',
            ]
            self.call_idx = 0
        
        def generate(self, prompt):
            if self.call_idx < len(self.responses):
                resp = self.responses[self.call_idx]
                self.call_idx += 1
                return resp
            return "Final Answer: 我已经回答过了。"
    
    # 创建并运行 Agent
    llm = MockLLM()
    agent = ReActAgent(llm, tools={"search": search, "calculator": calculator})
    result = agent.run("LangChain 是什么？")
    
    print(f"\n最终答案: {result['answer']}")
    print(f"迭代次数: {result['iterations']}")
    print(f"执行轨迹长度: {len(result['trace'])} 步")


if __name__ == "__main__":
    demo_react_agent()
```

---

## 17.3 Reflexion 模式

### 17.3.1 Reflexion 的核心思想

如果说 ReAct 是"边做边想"，那 Reflexion 就是"做完再想"。Reflexion 的核心思想是：**当 Agent 的尝试失败后，它不是简单地重试，而是先反思"为什么失败了"，然后把反思结果作为经验，指导后续的尝试。**

这就像一个学生做考试题：第一次做错了，不是直接重做，而是先分析"我错在哪里"、"为什么错"、"下次应该怎么避免"，然后带着这些经验教训去做下一套题。

Reflexion 的循环是这样的：

```
第一次尝试：
  Plan → Execute → Evaluate（失败）→ Reflect（反思）→ 记录经验教训

第二次尝试：
  Plan（考虑之前的经验教训）→ Execute → Evaluate（可能成功）

关键：反思结果作为"长期经验"存储，影响后续所有决策
```

### 17.3.2 Reflexion Agent 的实现

```python
from dataclasses import dataclass, field


@dataclass
class Reflection:
    """
    反思记录
    
    每次失败后，Agent 会生成一条反思记录，
    包含失败原因分析和经验教训。
    """
    task: str
    attempt_number: int
    result: str
    reflection: str
    lessons_learned: list[str] = field(default_factory=list)
    timestamp: float = field(default_factory=time.time)
    
    def to_prompt_text(self) -> str:
        """转为可注入 Prompt 的文本"""
        lessons = "\n".join(f"  - {l}" for l in self.lessons_learned)
        return f"尝试 #{self.attempt_number}: {self.reflection}\n经验教训:\n{lessons}"


class ReflexionAgent:
    """
    Reflexion 推理模式的 Agent
    
    核心流程：
    1. 生成计划（考虑之前的经验教训）
    2. 执行计划
    3. 评估结果是否成功
    4. 如果失败，进行反思并记录经验教训
    5. 用新的经验教训指导下一次尝试
    """
    
    def __init__(self, llm_client, max_attempts: int = 3,
                 evaluation_threshold: float = 0.7):
        """
        参数:
            llm_client: LLM 客户端
            max_attempts: 最大尝试次数
            evaluation_threshold: 成功的最低评估分数 (0-1)
        """
        self.llm_client = llm_client
        self.max_attempts = max_attempts
        self.evaluation_threshold = evaluation_threshold
        self.reflections: list[Reflection] = []
        self.all_lessons: list[str] = []
    
    def run(self, task: str) -> dict:
        """
        执行 Reflexion 循环
        
        返回包含最终结果和所有反思记录的字典
        """
        for attempt in range(self.max_attempts):
            print(f"\n{'='*50}")
            print(f"尝试 #{attempt + 1}/{self.max_attempts}")
            print(f"{'='*50}")
            
            # 第一步：生成计划（考虑之前的经验教训）
            plan = self._generate_plan(task, attempt)
            print(f"计划:\n{plan[:200]}...")
            
            # 第二步：执行计划
            result = self._execute(task, plan)
            print(f"\n执行结果:\n{result[:200]}...")
            
            # 第三步：评估结果
            evaluation = self._evaluate(task, result)
            print(f"\n评估: {'成功' if evaluation['success'] else '失败'} "
                  f"(分数: {evaluation.get('score', 0):.2f})")
            
            if evaluation["success"]:
                return {
                    "task": task,
                    "final_result": result,
                    "attempts": attempt + 1,
                    "success": True,
                    "reflections": [r.__dict__ for r in self.reflections],
                    "total_lessons_learned": len(self.all_lessons),
                }
            
            # 第四步：反思
            if attempt < self.max_attempts - 1:
                reflection = self._reflect(task, result, evaluation, attempt)
                self.reflections.append(reflection)
                self.all_lessons.extend(reflection.lessons_learned)
                
                print(f"\n反思: {reflection.reflection}")
                print("经验教训:")
                for lesson in reflection.lessons_learned:
                    print(f"  - {lesson}")
        
        return {
            "task": task,
            "final_result": None,
            "attempts": self.max_attempts,
            "success": False,
            "reflections": [r.__dict__ for r in self.reflections],
            "total_lessons_learned": len(self.all_lessons),
        }
    
    def _generate_plan(self, task: str, attempt: int) -> str:
        """生成执行计划（考虑历史经验教训）"""
        lessons_text = ""
        if self.all_lessons:
            lessons_text = "\n".join(f"- {l}" for l in self.all_lessons[-10:])  # 最多看最近 10 条
            lessons_text = f"\n\n历史经验教训（请务必参考）：\n{lessons_text}"
        
        prompt = f"""请为以下任务制定执行计划。

任务：{task}
当前尝试次数：{attempt + 1}
{lessons_text}

请制定一个详细的执行计划，特别注意避免之前犯过的错误。

执行计划："""

        return self.llm_client.generate(prompt)
    
    def _execute(self, task: str, plan: str) -> str:
        """执行计划"""
        prompt = f"""请按照以下计划执行任务。

任务：{task}
计划：{plan}

请详细执行计划中的每个步骤，并输出完整的执行结果："""

        return self.llm_client.generate(prompt)
    
    def _evaluate(self, task: str, result: str) -> dict:
        """
        评估执行结果
        
        使用 LLM 来评估结果是否满足任务要求
        """
        prompt = f"""请评估以下执行结果是否成功完成了任务。

任务：{task}
执行结果：{result}

请从以下维度评估（每个 0-1 分）：
1. 完成度：是否完成了任务的所有要求
2. 正确性：结果是否正确
3. 质量：结果的质量如何

输出 JSON 格式：
{{
    "success": true/false,
    "score": 0.0-1.0,
    "completion": 0.0-1.0,
    "correctness": 0.0-1.0,
    "quality": 0.0-1.0,
    "feedback": "总体评价"
}}"""

        response = self.llm_client.generate(prompt)
        
        try:
            evaluation = json.loads(response)
            evaluation["success"] = evaluation.get("score", 0) >= self.evaluation_threshold
            return evaluation
        except json.JSONDecodeError:
            return {"success": False, "score": 0.5, "feedback": "评估解析失败"}
    
    def _reflect(self, task: str, result: str, evaluation: dict,
                 attempt: int) -> Reflection:
        """
        反思：分析失败原因，总结经验教训
        
        这是 Reflexion 的核心——从失败中学习
        """
        prompt = f"""请深入反思以下执行过程，分析问题并总结经验教训。

任务：{task}
执行结果：{result}
评估反馈：{json.dumps(evaluation, ensure_ascii=False)}
尝试次数：{attempt + 1}

请分析：
1. 为什么失败了？根本原因是什么？
2. 哪些地方做得好？哪些地方做得不好？
3. 如果重新来过，你会怎么做？请给出具体的改进建议。
4. 你学到了什么经验教训？（每条教训应该具体、可操作）

输出格式：
反思：<你的总体反思，2-3 句话>
经验教训：
- 教训1（具体、可操作）
- 教训2
- ..."""

        response = self.llm_client.generate(prompt)
        
        # 解析反思和经验教训
        reflection_text = response
        lessons = []
        
        if "经验教训：" in response:
            parts = response.split("经验教训：")
            reflection_text = parts[0].strip()
            lessons_text = parts[1]
            for line in lessons_text.split("\n"):
                line = line.strip()
                if line.startswith("-"):
                    lesson = line.lstrip("- ").strip()
                    if lesson:
                        lessons.append(lesson)
        
        return Reflection(
            task=task,
            attempt=attempt + 1,
            result=result[:200],
            reflection=reflection_text[:500],
            lessons_learned=lessons,
        )
```

### 17.3.3 运行示例

```python
def demo_reflexion():
    """演示 Reflexion Agent"""
    
    class MockLLM:
        def __init__(self):
            self.call_count = 0
        
        def generate(self, prompt):
            self.call_count += 1
            
            if "制定执行计划" in prompt:
                if "经验教训" in prompt:
                    return "计划：1. 使用更简单的方法 2. 先测试再提交"
                return "计划：1. 实现功能 2. 测试 3. 提交"
            elif "执行任务" in prompt:
                if self.call_count <= 4:
                    return "尝试实现了一个方案，但测试发现了 bug"
                return "根据经验教训，使用了更简单的方法，测试通过"
            elif "评估" in prompt:
                if self.call_count <= 5:
                    return json.dumps({"success": False, "score": 0.4, "completion": 0.5, "correctness": 0.3, "quality": 0.4, "feedback": "代码有 bug"})
                return json.dumps({"success": True, "score": 0.9, "completion": 0.95, "correctness": 0.9, "quality": 0.85, "feedback": "方案正确，测试通过"})
            elif "反思" in prompt:
                return "反思：方案过于复杂导致出错。\n经验教训：\n- 应该优先选择简单直接的方案\n- 写代码前先写测试用例"
            return "继续"
    
    llm = MockLLM()
    agent = ReflexionAgent(llm, max_attempts=3)
    result = agent.run("实现一个排序函数")
    
    print(f"\n{'='*50}")
    print(f"最终结果: {'成功' if result['success'] else '失败'}")
    print(f"尝试次数: {result['attempts']}")
    print(f"经验教训数: {result['total_lessons_learned']}")


if __name__ == "__main__":
    demo_reflexion()
```

---

## 17.4 Plan-and-Execute 模式

### 17.4.1 什么是 Plan-and-Execute

Plan-and-Execute 是一种"先想清楚再动手"的推理模式。它将整个过程分为两个明确的阶段：

1. **规划阶段（Plan）**：分析任务，制定详细的执行计划
2. **执行阶段（Execute）**：按计划逐步执行

这和 ReAct 的区别在于：ReAct 是"边想边做"，每一步都可能改变方向；而 Plan-and-Execute 是"先全部想好，再全部执行"。

### 17.4.2 Plan-and-Execute Agent 的实现

```python
class PlanAndExecuteAgent:
    """
    Plan-and-Execute 推理模式
    
    两个阶段：
    1. Plan：让 LLM 生成一个详细的、有结构的执行计划
    2. Execute：按计划逐步执行每一步
    """
    
    def __init__(self, llm_client, tools: dict = None):
        self.llm_client = llm_client
        self.tools = tools or {}
    
    def run(self, task: str) -> dict:
        """执行任务"""
        # 阶段 1：规划
        print("阶段 1: 制定计划...")
        plan = self._plan(task)
        
        print(f"计划包含 {len(plan.get('steps', []))} 个步骤:")
        for step in plan.get("steps", []):
            print(f"  {step.get('id', '?')}. {step.get('description', '未描述')}")
        
        # 阶段 2：执行
        print("\n阶段 2: 按计划执行...")
        results = []
        for step in plan.get("steps", []):
            print(f"\n  执行步骤 {step.get('id', '?')}: {step.get('description', '')[:50]}...")
            result = self._execute_step(step)
            results.append({"step": step, "result": result})
            print(f"    结果: {'成功' if result.get('success') else '失败'}")
        
        # 综合结果
        final_result = self._synthesize(task, plan, results)
        
        return {
            "task": task,
            "plan": plan,
            "results": results,
            "final_result": final_result,
            "success": all(r["result"].get("success", False) for r in results),
        }
    
    def _plan(self, task: str) -> dict:
        """生成执行计划"""
        tools_desc = "\n".join(
            f"- {name}: {func.__doc__ or '无描述'}"
            for name, func in self.tools.items()
        )
        
        prompt = f"""请为以下任务制定详细的执行计划。

任务：{task}

可用工具：
{tools_desc if tools_desc else '无可用工具'}

输出 JSON 格式的计划：
{{
    "goal": "最终目标",
    "steps": [
        {{
            "id": 1,
            "description": "步骤描述",
            "tool": "使用的工具（可选）",
            "input": "需要的输入",
            "expected_output": "预期输出"
        }}
    ]
}}

只输出 JSON："""

        response = self.llm_client.generate(prompt)
        
        try:
            return json.loads(response)
        except json.JSONDecodeError:
            return {"goal": task, "steps": []}
    
    def _execute_step(self, step: dict) -> dict:
        """执行单个步骤"""
        tool_name = step.get("tool")
        
        if tool_name and tool_name in self.tools:
            try:
                result = self.tools[tool_name](step.get("input", step.get("description", "")))
                return {"output": result, "success": True}
            except Exception as e:
                return {"output": str(e), "success": False}
        else:
            # 没有可用工具，用 LLM 直接执行
            prompt = f"请执行以下步骤：{step.get('description', '')}"
            result = self.llm_client.generate(prompt)
            return {"output": result, "success": True}
    
    def _synthesize(self, task: str, plan: dict, results: list) -> str:
        """综合所有结果"""
        results_text = "\n".join(
            f"步骤 {r['step'].get('id', '?')}: {r['result'].get('output', '')[:200]}"
            for r in results
        )
        
        prompt = f"""请综合以下执行结果，给出最终答案。

任务：{task}
执行结果：
{results_text}

最终答案："""

        return self.llm_client.generate(prompt)
```

---

## 17.5 推理模式的选择策略

### 17.5.1 如何选择推理模式

选择推理模式需要考虑三个核心因素：

**因素 1：任务是否需要外部信息**

如果任务需要调用外部工具（搜索引擎、数据库、API 等）来获取信息，ReAct 是最佳选择。因为 ReAct 天然支持"思考 → 调用工具 → 观察结果 → 继续思考"的循环。

**因素 2：任务的试错成本**

如果任务很容易失败（比如写代码、做数学题），且失败后需要从错误中学习，Reflexion 更合适。它的反思机制能帮助 Agent 避免重复犯错。

**因素 3：任务的结构化程度**

如果任务的步骤非常明确（比如"先做 A，再做 B，再做 C"），Plan-and-Execute 效率最高。因为它一次性规划好所有步骤，然后高效执行。

### 17.5.2 混合推理模式

在实际应用中，最好的方案往往是混合使用多种推理模式。一个常见的设计是：先用复杂度分析来判断任务类型，然后选择最合适的推理模式。

```python
class HybridReasoningAgent:
    """
    混合推理模式 Agent
    
    根据任务特征自动选择最合适的推理模式：
    - 简单任务 → 直接推理（不使用特殊模式）
    - 需要工具调用 → ReAct
    - 需要精确答案 → Reflexion
    - 结构明确的多步骤任务 → Plan-and-Execute
    """
    
    def __init__(self, llm_client, tools: dict = None):
        self.llm_client = llm_client
        self.tools = tools or {}
        
        # 初始化各种推理模式
        self.react_agent = ReActAgent(llm_client, self.tools, verbose=False)
        self.reflexion_agent = ReflexionAgent(llm_client, max_attempts=3)
        self.plan_execute_agent = PlanAndExecuteAgent(llm_client, self.tools)
    
    def run(self, task: str) -> dict:
        """根据任务类型选择推理模式"""
        # 分析任务特征
        task_type = self._analyze_task(task)
        print(f"任务类型: {task_type}")
        
        if task_type == "simple":
            print("使用模式: 直接推理")
            result = self.llm_client.generate(task)
            return {"answer": result, "mode": "direct", "success": True}
        
        elif task_type == "needs_tools":
            print("使用模式: ReAct")
            result = self.react_agent.run(task)
            return {**result, "mode": "react"}
        
        elif task_type == "needs_precision":
            print("使用模式: Reflexion")
            result = self.reflexion_agent.run(task)
            return {**result, "mode": "reflexion"}
        
        else:  # structured
            print("使用模式: Plan-and-Execute")
            result = self.plan_execute_agent.run(task)
            return {**result, "mode": "plan_and_execute"}
    
    def _analyze_task(self, task: str) -> str:
        """分析任务类型"""
        prompt = f"""分析以下任务的特征，选择最适合的推理模式。

任务：{task}

选项：
- simple: 简单直接的任务，不需要工具调用或复杂推理
- needs_tools: 需要调用外部工具（搜索、计算等）获取信息
- needs_precision: 需要高精度答案，可能需要多次尝试和修正
- structured: 步骤明确的多步骤任务

推理模式："""

        response = self.llm_client.generate(prompt)
        
        for mode in ["needs_tools", "needs_precision", "structured", "simple"]:
            if mode in response.lower():
                return mode
        
        return "simple"
```

---

## 17.6 常见坑

### 17.6.1 ReAct 陷入循环

**问题描述：** Agent 反复执行相同的 Action，陷入死循环。比如反复搜索同一个关键词，或者反复调用同一个工具但每次都得到相同的结果。

**原因：** LLM 没有意识到自己在重复，或者工具返回的结果没有变化但 LLM 不知道该怎么办。

**解决方案：**
1. 添加循环检测：如果连续两次的 Action 相同，强制改变策略
2. 设置最大迭代次数
3. 在 Prompt 中明确告诉 LLM："如果相同的操作已经执行过，请尝试其他方法"

```python
def detect_loop(trace: list[dict], window: int = 3) -> bool:
    """检测是否陷入循环"""
    if len(trace) < window:
        return False
    
    recent_actions = [
        t.get("parsed", {}).get("tool", "")
        for t in trace[-window:]
        if t.get("step_type") == "action"
    ]
    
    # 如果最近 N 次都是同一个工具
    return len(set(recent_actions)) == 1 and len(recent_actions) >= window
```

### 17.6.2 Reflexion 反思质量差

**问题描述：** 反思流于表面，没有真正分析问题原因。比如只是说"下次要更仔细"，但没有指出具体哪里出了问题、应该如何改进。

**解决方案：**
1. 提供结构化的反思模板，引导深度分析
2. 要求 Agent 给出具体的、可操作的改进建议
3. 把反思结果格式化为"经验教训列表"，方便后续参考

### 17.6.3 推理模式选择不当

**问题描述：** 简单任务用了复杂推理模式，浪费资源。或者复杂任务用了简单模式，效果不好。

**解决方案：** 根据任务复杂度动态选择推理模式。对于简单任务（如"今天天气怎么样"），直接回答即可；对于需要工具调用的任务，使用 ReAct；对于需要精确答案的任务，使用 Reflexion。

### 17.6.4 Token 消耗过大

**问题描述：** ReAct 和 Reflexion 都涉及多轮 LLM 调用，Token 消耗可能很大。

**解决方案：**
1. 设置最大迭代次数
2. 在 Prompt 中压缩历史记录，只保留最近几轮的 Thought-Action-Observation
3. 使用更便宜的模型来处理简单步骤

---

## 17.7 练习题

### 练习 1：实现 ReAct Agent

实现一个完整的 ReAct Agent，要求：
- 支持至少 3 种工具调用（搜索、计算、查天气等）
- 能处理工具调用失败的情况
- 有循环检测机制（防止重复调用同一个工具）
- 记录完整的执行轨迹（Thought → Action → Observation）
- 支持最大迭代次数限制

### 练习 2：实现 Reflexion Agent

实现一个 Reflexion Agent，要求：
- 能从失败中学习（生成结构化的反思和经验教训）
- 维护经验教训库，在后续尝试中参考
- 支持多次尝试（至少 3 次）
- 每次尝试后评估结果质量
- 有最大尝试次数限制

### 练习 3：实现 Plan-and-Execute Agent

实现一个 Plan-and-Execute Agent，要求：
- 能生成结构化的执行计划（JSON 格式）
- 按计划逐步执行
- 执行过程中记录结果
- 最终综合所有结果给出答案

### 练习 4：实现混合推理 Agent

实现一个能自动选择推理模式的 Agent，要求：
- 能分析任务特征
- 根据任务特征选择最合适的推理模式
- 支持 ReAct、Reflexion、Plan-and-Execute 三种模式
- 记录每次选择了什么模式以及为什么

### 练习 5：推理模式对比实验

对以下三种任务，分别用 ReAct、Reflexion、Plan-and-Execute 三种模式执行，对比结果：
1. "搜索并总结 Python 3.12 的新特性"（需要工具调用）
2. "实现一个二分查找算法"（需要精确答案）
3. "规划一次三天的旅行"（结构化的多步骤任务）

对比指标：执行时间、Token 消耗、结果质量。

---

## 17.8 实战任务

### 任务：构建智能推理 Agent

**目标：** 构建一个能根据任务自动选择推理模式的 Agent，支持 ReAct、Reflexion 和 Plan-and-Execute 三种模式。

**要求：**

1. 实现三种推理模式的完整实现
2. 能根据任务特征自动选择最合适的模式
3. 支持推理过程的可视化（打印每一步的 Thought → Action → Observation）
4. 记录执行轨迹，方便调试和分析
5. 有错误处理和超时机制

**参考实现：**

```python
class SmartReasoningAgent:
    """智能推理 Agent"""
    
    def __init__(self, llm_client, tools: dict = None):
        self.hybrid = HybridReasoningAgent(llm_client, tools)
        self.history: list[dict] = []
    
    def chat(self, task: str) -> str:
        """处理用户任务"""
        result = self.hybrid.run(task)
        self.history.append(result)
        
        if result.get("success"):
            return result.get("answer") or result.get("final_result", "执行完成")
        else:
            return f"任务未能成功完成（模式: {result.get('mode', 'unknown')}）"
    
    def get_stats(self) -> dict:
        """获取统计信息"""
        modes = [r.get("mode", "unknown") for r in self.history]
        return {
            "total_tasks": len(self.history),
            "success_rate": sum(1 for r in self.history if r.get("success")) / max(len(self.history), 1),
            "mode_distribution": {m: modes.count(m) for m in set(modes)},
        }
```

---

## 17.9 本章小结

- **ReAct（Reasoning + Acting）** 是最常用的 Agent 推理模式。它让 Agent 交替进行推理和行动：先思考需要什么信息（Thought），再调用工具获取信息（Action），观察结果（Observation），然后继续思考。这种方式的优势是灵活、可解释、实时调整。劣势是可能陷入循环，Token 消耗较大。

- **Reflexion** 通过自我反思让 Agent 从错误中学习。当执行失败后，Agent 不是简单地重试，而是先分析"为什么失败"、"学到了什么"，然后把这些经验教训应用到后续的尝试中。Reflexion 的核心价值在于"从失败中学习"，特别适合需要精确答案的任务。

- **Plan-and-Execute** 是"先规划后执行"的结构化推理方式。它先让 LLM 制定一个详细的执行计划，然后按计划逐步执行。优势是结构清晰、效率高；劣势是缺乏灵活性，不适应计划执行过程中的变化。

- **混合推理模式** 根据任务特征自动选择最合适的推理模式。需要工具调用时用 ReAct，需要精确答案时用 Reflexion，结构明确的多步骤任务用 Plan-and-Execute。在实际应用中，混合模式往往能取得最好的效果。

- **选择推理模式时要考虑**：任务是否需要外部信息（→ ReAct）、试错成本是否高（→ Reflexion）、步骤是否明确（→ Plan-and-Execute）、计算资源是否充足。

- **常见陷阱**包括 ReAct 陷入循环、Reflexion 反思质量差、推理模式选择不当、Token 消耗过大。解决这些陷阱的关键是添加循环检测、结构化反思模板、动态模式选择和资源控制。

