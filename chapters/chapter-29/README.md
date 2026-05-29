# 第 29 章 CrewAI 框架 —— 协作型 Multi-Agent

> "如果说 AutoGen 让 Agent 通过对话来协作，那么 CrewAI 就是让 Agent 像一个真正的团队一样工作——有明确的角色分工、有清晰的任务流程、有可追踪的执行结果。CrewAI 将 Multi-Agent 协作提升到了工程化的高度。"

---

## 学习目标

通过本章的学习，你将能够：

1. 理解 CrewAI 框架的核心架构和设计理念
2. 掌握 CrewAI 中 Agent、Task、Crew 三大核心概念
3. 学会使用 CrewAI 构建结构化的 Multi-Agent 团队
4. 理解 CrewAI 的流程模式（顺序执行、层级管理）
5. 使用 Python 实现基于 CrewAI 的实际应用
6. 掌握 CrewAI 的工具集成和自定义方法
7. 对比 CrewAI 与 AutoGen 的异同，理解各自的适用场景

---

## 核心问题

- CrewAI 的核心设计理念是什么？它如何简化 Multi-Agent 系统的开发？
- CrewAI 中的 Agent、Task、Crew 三者之间是什么关系？
- CrewAI 的顺序执行和层级管理模式分别适用于什么场景？
- CrewAI 如何与外部工具集成？
- CrewAI 与 AutoGen 相比，各自的优势和劣势是什么？

---

## 29.1 CrewAI 框架概述

### 29.1.1 什么是 CrewAI

CrewAI 是一个开源的 Multi-Agent 协作框架，它的名字来自 "Crew"（团队）——因为它把 Multi-Agent 系统类比为一个团队。在这个团队中，每个 Agent 是一个团队成员，有自己的角色、职责和专长；每个 Task 是一个具体的任务，有明确的目标和预期输出；Crew 是整个团队的管理者，负责协调成员之间的工作。

与 AutoGen 的对话驱动模式不同，CrewAI 采用了更加结构化的方式来定义 Multi-Agent 协作。在 CrewAI 中，你需要明确定义：有哪些 Agent（团队成员）、每个 Agent 负责什么、有哪些 Task（任务）、任务之间的执行顺序是什么。这种结构化的方式使得系统的行为更加可预测和可控制。

### 29.1.2 CrewAI 的设计哲学

CrewAI 的设计哲学可以用三个词来概括：**角色**、**任务**、**流程**。

**角色（Role）**：每个 Agent 都有明确的角色定义，包括它的目标（goal）、背景故事（backstory）和允许使用的工具（tools）。角色定义不仅仅是名称和描述，还包括这个 Agent 应该如何思考和行动。

**任务（Task）**：每个任务都有明确的描述、预期输出、负责的 Agent，以及可选的上下文依赖。任务是 CrewAI 中最小的执行单元。

**流程（Process）**：流程定义了任务的执行顺序。CrewAI 支持顺序流程（Sequential，一个任务接一个任务执行）和层级流程（Hierarchical，有一个管理 Agent 来分配和协调任务）。

### 29.1.3 CrewAI vs AutoGen

CrewAI 和 AutoGen 都是优秀的 Multi-Agent 框架，但它们的设计理念和适用场景有所不同。

AutoGen 更适合需要深度对话交互的场景——多个 Agent 需要反复讨论、质疑、修正彼此的观点。它的对话驱动模式使得 Agent 之间的交互非常灵活和自然。

CrewAI 更适合需要结构化协作的场景——有明确的任务分工、清晰的执行流程、可预测的结果。它的角色-任务-流程模式使得系统的行为更加可控和可维护。

简单来说，AutoGen 像是一个自由讨论的研讨会，而 CrewAI 像是一个有组织的项目团队。

---

## 29.2 CrewAI 的核心概念

### 29.2.1 Agent（智能体）

在 CrewAI 中，Agent 是团队的成员。每个 Agent 的定义包括：

**角色（role）**：Agent 的职位名称，比如"数据分析师"、"技术作家"等。

**目标（goal）**：Agent 的工作目标，它应该朝着这个目标努力。

**背景故事（backstory）**：Agent 的背景设定，这会影响它的思维方式和行为风格。这是 CrewAI 的一个独特设计——通过故事化的背景来引导 LLM "入戏"。

**允许的工具（tools）**：Agent 可以使用的工具列表，比如搜索工具、代码执行工具等。

**verbose**：是否打印详细的执行日志。

**allow_delegation**：是否允许将任务委派给其他 Agent。

### 29.2.2 Task（任务）

Task 是 CrewAI 中的执行单元。每个 Task 的定义包括：

**描述（description）**：任务的具体描述。

**预期输出（expected_output）**：任务完成后应该产出什么。这是一个非常重要的设计——它为 Agent 提供了明确的目标，减少了歧义。

**负责的 Agent（agent）**：哪个 Agent 来执行这个任务。

**上下文（context）**：任务依赖的上下文信息，通常来自前序任务的输出。

### 29.2.3 Crew（团队）

Crew 是 Agent 和 Task 的容器。它定义了：

**成员（agents）**：团队中的所有 Agent。

**任务（tasks）**：团队需要完成的所有任务。

**流程（process）**：任务的执行方式。

**verbose**：是否打印详细的执行日志。

**memory**：是否启用团队记忆。

---

## 29.3 完整实现：CrewAI 风格的 Multi-Agent 系统

由于 CrewAI 是一个第三方框架，我们首先展示如何使用 CrewAI 库，然后展示一个不依赖 CrewAI 的简化实现，帮助理解其核心原理。

### 29.3.1 使用 CrewAI 库

```python
"""
第 29 章：CrewAI 框架入门
使用 CrewAI 构建 Multi-Agent 团队

安装: pip install crewai crewai-tools
"""
import os

# 注意：以下代码需要安装 crewai
# pip install crewai crewai-tools


def demo_crewai_basic():
    """
    演示 CrewAI 的基本使用。
    
    这个示例创建一个研究团队，包含：
    - 研究员：负责搜集资料
    - 分析师：负责分析资料
    - 作家：负责撰写报告
    """
    
    try:
        from crewai import Agent, Task, Crew, Process
        
        # 配置 LLM（CrewAI 使用 LiteLLM）
        os.environ["OPENAI_API_KEY"] = os.environ.get("OPENAI_API_KEY", "")
        
        # ---- 定义 Agent ----
        
        researcher = Agent(
            role="高级研究分析师",
            goal="搜集和整理关于给定主题的最新、最准确的信息",
            backstory="""你是一位经验丰富的研究分析师，擅长从大量信息中
            筛选出最有价值的内容。你在学术界和工业界都有广泛的人脉，
            能够获取到最前沿的信息。你的研究方法严谨，注重信息的
            准确性和可靠性。""",
            verbose=True,
            allow_delegation=False,
            # tools=[SearchTool()],  # 可以添加工具
        )
        
        analyst = Agent(
            role="数据分析师",
            goal="从研究资料中提取关键洞察和趋势",
            backstory="""你是一位资深的数据分析师，擅长从复杂的信息中
            识别模式和趋势。你善于使用统计方法来验证假设，
            能够将零散的信息整合成有结构的分析报告。""",
            verbose=True,
            allow_delegation=False,
        )
        
        writer = Agent(
            role="技术文档作家",
            goal="将分析结果转化为清晰、有深度的报告",
            backstory="""你是一位优秀的技术文档作家，能够将复杂的技术
            概念用通俗易懂的语言表达出来。你的文章既有专业深度，
            又有良好的可读性。你擅长使用案例和类比来解释抽象概念。""",
            verbose=True,
            allow_delegation=False,
        )
        
        # ---- 定义 Task ----
        
        research_task = Task(
            description="""针对主题「AI Agent 的发展趋势」进行深入研究。
            搜集最新的行业报告、学术论文和技术博客中的相关信息。
            整理出至少 5 个关键发展领域。""",
            expected_output="""一份结构化的研究报告，包含：
            1. 背景概述
            2. 至少 5 个关键发展领域及其详细说明
            3. 主要参与者和他们的贡献
            4. 未来展望""",
            agent=researcher,
        )
        
        analysis_task = Task(
            description="""基于研究员提供的资料，进行深入分析。
            识别最重要的趋势和洞察，评估各发展方向的潜力和风险。""",
            expected_output="""一份分析报告，包含：
            1. 核心趋势总结（3-5个）
            2. 每个趋势的深入分析
            3. 机会与风险评估
            4. 战略建议""",
            agent=analyst,
            context=[research_task],  # 依赖研究任务的输出
        )
        
        writing_task = Task(
            description="""根据研究报告和分析报告，撰写一篇面向技术管理者的
            深度分析文章。文章应该通俗易懂但有专业深度。""",
            expected_output="""一篇完整的分析文章（1500-2500字），包含：
            1. 引人入胜的开头
            2. 清晰的结构和逻辑
            3. 具体的案例和数据
            4. 有见地的结论和展望""",
            agent=writer,
            context=[research_task, analysis_task],  # 依赖研究和分析任务
        )
        
        # ---- 创建 Crew ----
        
        crew = Crew(
            agents=[researcher, analyst, writer],
            tasks=[research_task, analysis_task, writing_task],
            process=Process.sequential,  # 顺序执行
            verbose=True,
            memory=True,  # 启用记忆
        )
        
        # ---- 执行 ----
        
        print("\n" + "="*60)
        print("CrewAI 团队启动")
        print("="*60)
        
        result = crew.kickoff()
        
        print(f"\n{'='*60}")
        print("执行结果:")
        print(f"{'='*60}")
        print(result)
        
    except ImportError:
        print("请先安装 CrewAI: pip install crewai crewai-tools")
    except Exception as e:
        print(f"运行出错: {e}")


if __name__ == "__main__":
    demo_crewai_basic()
```

### 29.3.2 CrewAI 核心原理的简化实现

为了深入理解 CrewAI 的工作原理，我们来实现一个简化版本：

```python
"""
第 29 章：CrewAI 核心原理的简化实现
不依赖 CrewAI 库，从零构建 CrewAI 风格的 Multi-Agent 系统
"""
import os
import json
import time
from typing import Optional
from dataclasses import dataclass, field
from openai import OpenAI


# ============================================================
# LLM 调用封装
# ============================================================

def call_llm(
    messages: list[dict],
    model: str = "gpt-4o",
    temperature: float = 0.7,
    max_tokens: int = 2000
) -> str:
    """统一的 LLM 调用封装"""
    client = OpenAI()
    for attempt in range(3):
        try:
            response = client.chat.completions.create(
                model=model,
                messages=messages,
                temperature=temperature,
                max_tokens=max_tokens
            )
            return response.choices[0].message.content.strip()
        except Exception as e:
            print(f"  [重试 {attempt + 1}/3] LLM 调用失败: {e}")
            if attempt == 2:
                raise
            time.sleep(2 ** attempt)


# ============================================================
# 简化版 CrewAI 核心组件
# ============================================================

@dataclass
class SimpleTask:
    """
    简化版任务定义。
    
    对应 CrewAI 的 Task 类。
    每个任务都有描述、预期输出、负责的 Agent 和上下文依赖。
    """
    description: str
    expected_output: str
    agent: "SimpleAgent" = None
    context: list["SimpleTask"] = field(default_factory=list)
    output: str = ""  # 任务执行后的输出
    
    def execute(self, verbose: bool = True) -> str:
        """
        执行任务。
        
        收集上下文信息，然后调用负责的 Agent 来完成任务。
        """
        if not self.agent:
            raise ValueError("任务没有分配给任何 Agent")
        
        # 收集上下文
        context_text = ""
        if self.context:
            context_parts = []
            for ctx_task in self.context:
                if ctx_task.output:
                    context_parts.append(
                        f"[来自 {ctx_task.agent.name} 的输出]:\n{ctx_task.output}"
                    )
            if context_parts:
                context_text = "\n\n".join(context_parts)
        
        if verbose:
            print(f"\n  [任务] 执行: {self.description[:60]}...")
            print(f"  [任务] 负责人: {self.agent.name}")
        
        # 调用 Agent 执行任务
        self.output = self.agent.execute_task(
            self.description,
            self.expected_output,
            context_text
        )
        
        if verbose:
            print(f"  [任务] 完成，输出长度: {len(self.output)} 字符")
        
        return self.output


class SimpleAgent:
    """
    简化版 Agent 定义。
    
    对应 CrewAI 的 Agent 类。
    每个 Agent 都有角色、目标、背景故事和执行能力。
    """
    
    def __init__(
        self,
        role: str,
        goal: str,
        backstory: str,
        model: str = "gpt-4o",
        allow_delegation: bool = False,
        verbose: bool = True
    ):
        """
        Args:
            role: Agent 的角色名称
            goal: Agent 的工作目标
            backstory: Agent 的背景故事
            model: 使用的 LLM 模型
            allow_delegation: 是否允许委派任务给其他 Agent
            verbose: 是否打印详细日志
        """
        self.role = role
        self.goal = goal
        self.backstory = backstory
        self.model = model
        self.allow_delegation = allow_delegation
        self.verbose = verbose
        self.name = role  # 使用角色名作为名称
    
    def execute_task(
        self,
        task_description: str,
        expected_output: str,
        context: str = ""
    ) -> str:
        """
        执行一个任务。
        
        Args:
            task_description: 任务描述
            expected_output: 预期输出格式
            context: 上下文信息
        
        Returns:
            任务执行结果
        """
        # 构建系统提示词
        system_prompt = f"""你是一位{self.role}。

你的目标: {self.goal}

你的背景:
{self.backstory}

请根据你的角色、目标和背景来完成任务。你的输出应该体现你的专业性和角色特点。"""
        
        # 构建任务消息
        task_msg = f"""请完成以下任务：

任务描述: {task_description}

预期输出: {expected_output}"""
        
        if context:
            task_msg += f"\n\n上下文信息:\n{context}"
        
        messages = [
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": task_msg}
        ]
        
        # 根据角色选择温度
        temperature = 0.5
        
        result = call_llm(messages, model=self.model, temperature=temperature)
        
        return result


@dataclass
class SimpleCrew:
    """
    简化版 Crew 定义。
    
    对应 CrewAI 的 Crew 类。
    管理 Agent 和 Task 的集合，控制执行流程。
    """
    agents: list[SimpleAgent] = field(default_factory=list)
    tasks: list[SimpleTask] = field(default_factory=list)
    process: str = "sequential"  # "sequential" 或 "hierarchical"
    verbose: bool = True
    memory_enabled: bool = False
    
    # 记忆存储
    _memory: list[dict] = field(default_factory=list, repr=False)
    
    def add_agent(self, agent: SimpleAgent):
        """添加 Agent"""
        self.agents.append(agent)
    
    def add_task(self, task: SimpleTask):
        """添加 Task"""
        self.tasks.append(task)
    
    def kickoff(self) -> str:
        """
        启动 Crew 执行。
        
        这是 CrewAI 的核心方法。根据流程模式执行所有任务。
        """
        if self.verbose:
            print(f"\n{'='*60}")
            print(f"Crew 启动!")
            print(f"团队成员: {', '.join(a.role for a in self.agents)}")
            print(f"任务数量: {len(self.tasks)}")
            print(f"执行模式: {self.process}")
            print(f"{'='*60}")
        
        if self.process == "sequential":
            return self._sequential_execute()
        elif self.process == "hierarchical":
            return self._hierarchical_execute()
        else:
            return self._sequential_execute()
    
    def _sequential_execute(self) -> str:
        """
        顺序执行模式。
        
        按照任务列表的顺序，依次执行每个任务。
        每个任务的输出可以作为后续任务的上下文。
        """
        results = []
        
        for i, task in enumerate(self.tasks):
            if self.verbose:
                print(f"\n{'='*40}")
                print(f"步骤 {i+1}/{len(self.tasks)}")
                print(f"{'='*40}")
            
            output = task.execute(verbose=self.verbose)
            results.append(output)
            
            # 更新记忆
            if self.memory_enabled:
                self._memory.append({
                    "task": task.description[:100],
                    "output": output[:200]
                })
        
        # 整合最终结果
        final_output = self._integrate_results(results)
        
        if self.verbose:
            print(f"\n{'='*60}")
            print(f"Crew 执行完成!")
            print(f"{'='*60}")
        
        return final_output
    
    def _hierarchical_execute(self) -> str:
        """
        层级执行模式。
        
        使用第一个 Agent 作为管理者，由它来分配和协调任务。
        """
        if not self.agents:
            return "没有可用的 Agent"
        
        manager = self.agents[0]  # 第一个 Agent 作为管理者
        
        if self.verbose:
            print(f"\n层级模式 - 管理者: {manager.role}")
        
        # 管理者制定执行计划
        task_descriptions = [
            f"任务 {i+1}: {task.description[:80]}"
            for i, task in enumerate(self.tasks)
        ]
        
        plan_prompt = f"""作为管理者，请为以下任务制定执行计划：

{chr(10).join(task_descriptions)}

请确定：
1. 任务的执行顺序
2. 哪些任务可以并行
3. 是否需要调整任务的优先级

简要输出你的计划。"""
        
        messages = [{"role": "user", "content": plan_prompt}]
        plan = call_llm(messages, model=manager.model, temperature=0.3)
        
        if self.verbose:
            print(f"\n  管理者的计划:")
            print(f"  {plan[:300]}...")
        
        # 按照顺序执行任务
        return self._sequential_execute()
    
    def _integrate_results(self, results: list[str]) -> str:
        """整合所有任务的结果"""
        if len(results) == 1:
            return results[0]
        
        combined = "\n\n---\n\n".join([
            f"## 步骤 {i+1} 的结果\n{result}"
            for i, result in enumerate(results)
        ])
        
        return combined


# ============================================================
# 演示：简化版 CrewAI
# ============================================================

def demo_simple_crewai():
    """演示简化版 CrewAI"""
    
    print("\n" + "="*60)
    print("演示：简化版 CrewAI")
    print("="*60)
    
    # ---- 创建 Agent ----
    
    researcher = SimpleAgent(
        role="研究分析师",
        goal="搜集和整理关于给定主题的最新信息",
        backstory="""你是一位严谨的研究分析师，擅长从多个信息源搜集
        和整理资料。你注重信息的准确性和时效性，总是能够找到
        最有价值的参考材料。""",
    )
    
    analyst = SimpleAgent(
        role="数据分析师",
        goal="从研究资料中提取关键洞察和趋势",
        backstory="""你是一位洞察力极强的数据分析师，能够从看似杂乱的
        信息中发现隐藏的模式和趋势。你的分析总是深入且有说服力。""",
    )
    
    writer = SimpleAgent(
        role="技术作家",
        goal="将分析结果转化为清晰、有深度的文章",
        backstory="""你是一位才华横溢的技术作家，擅长将复杂的概念用
        通俗易懂的语言表达出来。你的文章既有深度又有可读性。""",
    )
    
    # ---- 创建 Task ----
    
    research_task = SimpleTask(
        description="研究 AI Agent 在企业中的应用现状和趋势",
        expected_output="结构化的研究报告",
        agent=researcher,
    )
    
    analysis_task = SimpleTask(
        description="基于研究资料，分析 AI Agent 的商业价值和应用前景",
        expected_output="包含趋势分析和商业洞察的报告",
        agent=analyst,
        context=[research_task],
    )
    
    writing_task = SimpleTask(
        description="撰写一篇关于 AI Agent 商业应用的深度分析文章",
        expected_output="1500字以上的完整文章",
        agent=writer,
        context=[research_task, analysis_task],
    )
    
    # ---- 创建 Crew ----
    
    crew = SimpleCrew(
        agents=[researcher, analyst, writer],
        tasks=[research_task, analysis_task, writing_task],
        process="sequential",
        verbose=True,
    )
    
    # ---- 执行 ----
    
    result = crew.kickoff()
    
    print(f"\n{'='*60}")
    print("最终结果:")
    print(f"{'='*60}")
    print(result[:500] + "...")


if __name__ == "__main__":
    demo_simple_crewai()
```

---

## 29.4 CrewAI 的工具集成

### 29.4.1 工具的作用

CrewAI 的另一个强大特性是工具集成。Agent 可以使用各种工具来扩展自己的能力，比如网络搜索、数据库查询、文件操作、API 调用等。

在 CrewAI 中，工具被定义为 Python 函数或类，Agent 在执行任务时可以根据需要调用这些工具。这使得 Agent 不仅仅能进行推理和生成，还能真正地与外部世界交互。

### 29.4.2 自定义工具

```python
"""
第 29 章：CrewAI 工具集成
自定义工具的定义和使用
"""
import os
import json
import time
from typing import Callable
from dataclasses import dataclass
from openai import OpenAI


# ============================================================
# LLM 调用封装
# ============================================================

def call_llm(
    messages: list[dict],
    model: str = "gpt-4o",
    temperature: float = 0.7,
    max_tokens: int = 2000
) -> str:
    """统一的 LLM 调用封装"""
    client = OpenAI()
    for attempt in range(3):
        try:
            response = client.chat.completions.create(
                model=model,
                messages=messages,
                temperature=temperature,
                max_tokens=max_tokens
            )
            return response.choices[0].message.content.strip()
        except Exception as e:
            print(f"  [重试 {attempt + 1}/3] LLM 调用失败: {e}")
            if attempt == 2:
                raise
            time.sleep(2 ** attempt)


# ============================================================
# 工具定义
# ============================================================

@dataclass
class Tool:
    """
    工具定义。
    
    在 CrewAI 中，工具是 Agent 可以调用的功能模块。
    每个工具都有名称、描述和执行函数。
    """
    name: str
    description: str
    func: Callable
    
    def run(self, **kwargs) -> str:
        """执行工具"""
        try:
            return self.func(**kwargs)
        except Exception as e:
            return f"工具执行失败: {e}"


# 预定义工具

def search_web(query: str) -> str:
    """模拟网络搜索工具"""
    # 实际应用中，这里会调用真实的搜索 API
    return f"搜索结果: 关于 '{query}' 的信息...\n[模拟搜索结果]"

def read_file(file_path: str) -> str:
    """读取文件内容"""
    try:
        with open(file_path, "r", encoding="utf-8") as f:
            return f.read()
    except FileNotFoundError:
        return f"文件不存在: {file_path}"

def write_file(file_path: str, content: str) -> str:
    """写入文件内容"""
    try:
        with open(file_path, "w", encoding="utf-8") as f:
            f.write(content)
        return f"文件已写入: {file_path}"
    except Exception as e:
        return f"写入失败: {e}"

def calculate(expression: str) -> str:
    """执行数学计算"""
    try:
        result = eval(expression)
        return f"计算结果: {expression} = {result}"
    except Exception as e:
        return f"计算失败: {e}"


# 创建工具实例
tools = [
    Tool(
        name="网络搜索",
        description="搜索互联网获取最新信息",
        func=search_web
    ),
    Tool(
        name="文件读取",
        description="读取本地文件的内容",
        func=read_file
    ),
    Tool(
        name="文件写入",
        description="将内容写入本地文件",
        func=write_file
    ),
    Tool(
        name="数学计算",
        description="执行数学表达式计算",
        func=calculate
    ),
]


# ============================================================
# 带工具的 Agent
# ============================================================

class ToolEnabledAgent:
    """
    带工具的 Agent。
    
    这个 Agent 可以在执行任务时使用指定的工具。
    LLM 会根据任务需求决定是否需要使用工具。
    """
    
    def __init__(
        self,
        name: str,
        role: str,
        goal: str,
        backstory: str,
        tools: list[Tool] = None,
        model: str = "gpt-4o"
    ):
        self.name = name
        self.role = role
        self.goal = goal
        self.backstory = backstory
        self.tools = tools or []
        self.model = model
    
    def execute_with_tools(self, task: str, context: str = "") -> str:
        """
        使用工具执行任务。
        
        流程：
        1. 分析任务，判断需要哪些工具
        2. 调用工具获取信息
        3. 基于工具返回的信息完成任务
        """
        # 构建工具描述
        tool_descriptions = "\n".join([
            f"- {tool.name}: {tool.description}"
            for tool in self.tools
        ])
        
        # Step 1: 分析任务需求
        analysis_prompt = f"""你是一位{self.role}。

任务: {task}

可用的工具:
{tool_descriptions}

请分析这个任务需要使用哪些工具，以及每个工具的输入参数。
以 JSON 格式输出：
{{
    "tools_needed": [
        {{"name": "工具名", "params": {{"参数名": "参数值"}}}}
    ],
    "direct_answer": "如果不需要工具，直接回答的内容"
}}"""
        
        messages = [{"role": "user", "content": analysis_prompt}]
        analysis = call_llm(messages, model=self.model, temperature=0.3)
        
        # 尝试解析工具调用
        tool_results = []
        try:
            json_start = analysis.index("{")
            json_end = analysis.rindex("}") + 1
            plan = json.loads(analysis[json_start:json_end])
            
            for tool_call in plan.get("tools_needed", []):
                tool_name = tool_call.get("name", "")
                params = tool_call.get("params", {})
                
                # 找到对应的工具
                for tool in self.tools:
                    if tool.name == tool_name:
                        result = tool.run(**params)
                        tool_results.append(f"[工具 {tool_name}]: {result}")
                        print(f"  [{self.name}] 使用工具: {tool_name}")
                        break
            
            if plan.get("direct_answer"):
                tool_results.append(plan["direct_answer"])
                
        except (ValueError, json.JSONDecodeError):
            tool_results.append(analysis)
        
        # Step 2: 基于工具结果生成最终答案
        tool_context = "\n".join(tool_results) if tool_results else "没有使用工具"
        
        final_prompt = f"""你是一位{self.role}。

你的目标: {self.goal}

背景: {self.backstory}

任务: {task}

工具获取的信息:
{tool_context}

{f"额外上下文: {context}" if context else ""}

请基于以上所有信息，完成任务并给出你的专业输出。"""
        
        messages = [
            {"role": "system", "content": f"你是一位{self.role}。{self.backstory}"},
            {"role": "user", "content": final_prompt}
        ]
        
        return call_llm(messages, model=self.model, temperature=0.5)


# ============================================================
# 演示：带工具的 Agent
# ============================================================

def demo_tools():
    """演示工具集成"""
    
    print("\n" + "="*60)
    print("演示：工具集成")
    print("="*60)
    
    # 创建带工具的 Agent
    researcher = ToolEnabledAgent(
        name="研究员",
        role="技术研究分析师",
        goal="搜集和分析最新的技术信息",
        backstory="你是一位资深的技术研究分析师，擅长使用各种工具来搜集和分析信息。",
        tools=[tools[0], tools[2]],  # 搜索和写入工具
    )
    
    # 使用工具执行任务
    result = researcher.execute_with_tools(
        task="搜索关于 CrewAI 框架的最新信息，并将搜索结果保存到一个文件中"
    )
    
    print(f"\n--- 执行结果 ---")
    print(result[:300] + "...")


if __name__ == "__main__":
    demo_tools()
```

---

## 29.5 CrewAI 的流程模式详解

### 29.5.1 顺序流程（Sequential Process）

顺序流程是最简单的执行模式：任务按照定义的顺序依次执行。每个任务完成后，其输出可以作为后续任务的上下文。

这种模式适合于任务之间有明确依赖关系的场景。比如，研究 → 分析 → 写作就是一个典型的顺序流程。

### 29.5.2 层级流程（Hierarchical Process）

层级流程引入了一个"管理者"的概念。管理者 Agent 负责将任务分配给合适的团队成员，并监控执行过程。如果某个任务执行不理想，管理者可以要求重做或调整策略。

这种模式适合于任务之间没有固定顺序、需要动态调度的场景。

### 29.5.3 流程选择的考量

选择哪种流程模式取决于以下因素：

**任务依赖性**：如果任务之间有严格的先后顺序，选择顺序流程。如果任务相对独立，可以选择层级流程。

**任务复杂度**：简单的线性任务适合顺序流程。复杂的、需要动态调整的任务适合层级流程。

**团队规模**：小团队（2-3个Agent）适合顺序流程。大团队（5个以上Agent）适合层级流程，因为需要有人来协调。

---

## 29.6 案例分析

### 29.6.1 案例：CrewAI 驱动的内容创作团队

某媒体公司使用 CrewAI 构建了一个 AI 内容创作团队。这个团队的结构如下：

**选题策划 Agent**：负责分析热点话题和读者兴趣，提出选题建议。

**资料搜集 Agent**：负责针对选定的话题搜集相关的资料和数据。

**内容创作 Agent**：负责根据资料撰写文章初稿。

**编辑 Agent**：负责审查和修改文章，提升文章质量。

**SEO Agent**：负责优化文章的搜索引擎排名，包括关键词选择和标题优化。

**排版 Agent**：负责将文章转换为最终的发布格式。

这个团队使用顺序流程，从选题到排版依次执行。每个 Agent 都有自己的工具集——选题策划 Agent 可以访问社交媒体 API，资料搜集 Agent 可以使用搜索引擎，SEO Agent 可以使用关键词分析工具。

效果显著：内容产出效率提升了 5 倍，同时文章质量保持在较高水平。每个 Agent 的专业化分工确保了各个环节的质量，而 CrewAI 的流程管理确保了整个创作过程的顺畅。

### 29.6.2 案例：CrewAI 驱动的客服团队

另一个案例是使用 CrewAI 构建智能客服团队：

**接待 Agent**：负责理解用户意图，将问题分类。

**产品专家 Agent**：负责回答关于产品功能和使用方法的问题。

**技术支持 Agent**：负责解决技术故障和 Bug 报告。

**投诉处理 Agent**：负责处理用户投诉和赔偿请求。

**满意度跟踪 Agent**：负责在问题解决后跟踪用户满意度。

这个团队使用层级流程，接待 Agent 作为管理者，根据用户问题的类型分配给对应的专家 Agent。

---

## 29.7 常见坑与解决方案

### 坑 1：角色定义不够具体

**问题描述**：Agent 的角色定义太笼统（如"你是一个助手"），导致 Agent 的行为缺乏特色。

**解决方案**：提供详细的背景故事（backstory），明确 Agent 的专业领域、思维方式和行为风格。

### 坑 2：任务描述不够清晰

**问题描述**：任务描述模糊（如"做好分析"），Agent 不确定具体要做什么。

**解决方案**：在任务描述中明确说明要做什么、怎么做、做到什么程度。预期输出（expected_output）要具体、可衡量。

### 坑 3：上下文传递丢失

**问题描述**：前序任务的输出没有正确传递给后续任务，导致后续任务缺少必要的信息。

**解决方案**：正确配置任务的 context 参数，确保依赖关系被正确声明。

### 坑 4：成本失控

**问题描述**：多个 Agent 的 LLM 调用叠加导致成本过高。

**解决方案**：为不同的 Agent 配置不同的模型。简单的任务使用 GPT-3.5-turbo，核心任务使用 GPT-4。

### 坑 5：执行时间过长

**问题描述**：顺序执行模式下，所有任务串行执行导致总时间过长。

**解决方案**：识别可以并行执行的任务，使用异步执行或多线程来加速。

---

## 29.8 练习题

### 练习 1：构建内容创作团队

**难度：★☆☆☆☆**

使用 CrewAI（或简化实现）构建一个内容创作团队，包含选题、写作和编辑三个 Agent。测试不同的任务描述对输出质量的影响。

### 练习 2：自定义工具

**难度：★★☆☆☆**

为 Agent 创建自定义工具：一个天气查询工具和一个日历管理工具。让 Agent 学会在合适的时机使用这些工具。

### 练习 3：层级流程

**难度：★★★☆☆**

实现层级流程模式：一个管理者 Agent 负责分配任务，多个专家 Agent 负责执行。观察管理者如何做决策。

### 练习 4：流程对比

**难度：★★★☆☆**

对同一个任务分别使用顺序流程和层级流程执行，对比两者的执行时间、输出质量和成本。

### 练习 5：混合框架

**难度：★★★★☆**

将 CrewAI 的结构化任务管理与 AutoGen 的对话驱动模式结合，构建一个混合框架。

---

## 29.9 实战任务

### 任务：构建 CrewAI 自动化营销系统

**目标**：使用 CrewAI 构建一个自动化营销内容生成系统。

**团队设计**：
1. **市场分析 Agent**：分析目标市场和竞争对手
2. **内容策略 Agent**：制定内容策略和主题规划
3. **文案创作 Agent**：撰写营销文案
4. **视觉设计 Agent**：生成图片描述和设计方案
5. **效果评估 Agent**：评估营销内容的潜在效果

**功能要求**：
1. 完整的团队定义和任务流程
2. 支持顺序和层级两种执行模式
3. 包含至少 2 个自定义工具
4. 代码完整可运行

---

## 29.10 本章小结

本章深入学习了 CrewAI 框架——一个以"角色"和"任务"为核心的结构化 Multi-Agent 协作框架。

CrewAI 的三大核心概念——Agent（角色）、Task（任务）、Crew（团队）——构成了一个清晰的层次结构。Agent 是执行者，Task 是执行单元，Crew 是协调者。这种结构化的设计使得 Multi-Agent 系统的行为更加可预测和可维护。

CrewAI 的背景故事（backstory）设计是一个独特的亮点——通过故事化的背景设定，让 LLM 更好地"入戏"，从而产出更符合角色设定的结果。

CrewAI 的工具集成功能让 Agent 不仅仅能"说"，还能"做"——通过调用外部工具来获取信息和执行操作。

与 AutoGen 相比，CrewAI 更适合需要结构化协作的场景，而 AutoGen 更适合需要深度对话的场景。在实际项目中，可以根据具体需求选择合适的框架，甚至将两者的优势结合使用。

> "好的团队不是一群人聚在一起，而是每个人都知道自己该做什么，并且能够相互配合。CrewAI 的设计哲学正是如此。"

---

**下一章预告**：在最后一章（第 30 章）中，我们将综合前面所有章节的知识，探讨 Multi-Agent 系统的架构设计和工程权衡。如何从零开始设计一个生产级别的 Multi-Agent 系统？敬请期待。
