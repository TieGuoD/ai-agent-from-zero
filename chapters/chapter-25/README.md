# 第 25 章 Multi-Agent 协调 —— 策略与模式

> "一群乌合之众和一支精锐之师的区别，不在于个体的能力，而在于协调的质量。Multi-Agent 系统的威力，很大程度上取决于协调策略的设计。"

---

## 学习目标

通过本章的学习，你将能够：

1. 理解 Multi-Agent 协调的核心挑战和设计原则
2. 掌握任务分解、分配和调度的基本策略
3. 学会设计投票、辩论、共识等群体决策机制
4. 理解层级协调、市场机制、合同网等协调模式
5. 使用 Python 实现多种协调策略的 Multi-Agent 系统
6. 掌握冲突检测与解决的方法
7. 了解大规模 Multi-Agent 系统的协调挑战

---

## 核心问题

- 多个 Agent 如何有效地分工，避免重复和遗漏？
- 当 Agent 之间的目标发生冲突时，如何解决？
- 如何设计公平且高效的决策机制？
- 不同的协调模式分别适用于什么场景？
- 如何在保证协调质量的同时控制通信开销？

---

## 25.1 协调：Multi-Agent 系统的灵魂

### 25.1.1 单打独斗 vs 团队协作

在前两章中，我们了解了 Multi-Agent 系统的基础和通信机制。但仅仅有多个 Agent 和通信能力还不够——如果 Agent 之间没有良好的协调，它们可能各自为政，甚至相互干扰，结果比单个 Agent 还要差。

想象一下这个场景：你要装修一套房子。你请了五个工人——一个水电工、一个泥瓦工、一个木工、一个油漆工和一个安装工。如果这五个人各干各的，水电工把线走好了，泥瓦工可能把砖砌在了水管上面，木工做完柜子发现尺寸和水电预留的位置不匹配，油漆工在泥瓦工还没干透的墙面上刷漆……这就是缺乏协调的灾难。

反之，如果有一个有经验的工长来协调——先做水电改造，再做泥瓦工程，然后木工进场，接着油漆施工，最后安装——整个装修过程就会顺畅得多。

Multi-Agent 系统面临的挑战与此完全一致。多个 Agent 同时工作时，如果没有协调，就可能出现以下问题：

**任务重复**：两个 Agent 同时做同一件事，浪费计算资源。

**任务遗漏**：所有 Agent 都以为某个子任务由其他 Agent 负责，结果谁都没做。

**资源冲突**：两个 Agent 同时修改同一个文件或数据，导致数据不一致。

**顺序错误**：应该先完成 A 再做 B，但 B 在 A 完成之前就开始了，导致结果错误。

协调策略就是要解决这些问题，让多个 Agent 像一支训练有素的团队一样高效协作。

### 25.1.2 协调的核心要素

一个完整的协调机制通常包含以下核心要素：

**任务分解**：把一个大任务拆分成若干个子任务。这一步的质量直接影响后续所有环节。

**任务分配**：决定哪个 Agent 负责哪个子任务。好的分配应该考虑 Agent 的能力、当前负载和任务的依赖关系。

**执行调度**：决定子任务的执行顺序和时间安排。哪些可以并行执行，哪些必须串行执行。

**冲突解决**：当 Agent 之间出现目标冲突、资源竞争或意见分歧时，如何达成一致。

**质量保证**：确保各个 Agent 的产出能够正确地组合在一起，满足整体质量要求。

---

## 25.2 任务分解策略

### 25.2.1 自上而下的分解

最直观的任务分解方式是自上而下：先将大任务分解为中等粒度的子任务，再将每个中等粒度的子任务分解为更细粒度的子子任务，直到每个子任务都足够小、足够明确，可以由单个 Agent 独立完成。

```python
"""
第 25 章：任务分解策略
实现自上而下的层级任务分解
"""
import os
import json
import time
import uuid
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
# 任务定义
# ============================================================

@dataclass
class Task:
    """
    任务数据结构。
    
    每个任务都有唯一标识、描述、状态和依赖关系。
    任务可以包含子任务，形成树状结构。
    """
    task_id: str = field(default_factory=lambda: str(uuid.uuid4())[:8])
    description: str = ""
    assigned_to: str = ""
    status: str = "pending"  # pending, in_progress, completed, failed
    result: str = ""
    dependencies: list[str] = field(default_factory=list)  # 依赖的任务 ID
    subtasks: list["Task"] = field(default_factory=list)
    parent_id: str = ""
    priority: int = 0  # 优先级，数值越大优先级越高
    
    def is_ready(self, completed_tasks: set[str] = None) -> bool:
        """检查任务是否准备好执行（所有依赖已完成）"""
        if completed_tasks is None:
            completed_tasks = set()
        return all(dep in completed_tasks for dep in self.dependencies)
    
    def to_dict(self) -> dict:
        """转为字典格式"""
        return {
            "task_id": self.task_id,
            "description": self.description,
            "assigned_to": self.assigned_to,
            "status": self.status,
            "dependencies": self.dependencies,
            "priority": self.priority,
            "subtask_count": len(self.subtasks)
        }


# ============================================================
# 任务分解器
# ============================================================

class TaskDecomposer:
    """
    任务分解器：使用 LLM 将复杂任务分解为可执行的子任务。
    
    分解策略：
    1. 分析任务的结构，识别可以独立执行的部分
    2. 确定子任务之间的依赖关系
    3. 为每个子任务分配适当的粒度
    4. 确保分解的完整性和无遗漏
    """
    
    def __init__(self, model: str = "gpt-4o"):
        self.model = model
    
    def decompose(self, task_description: str, max_depth: int = 2) -> list[Task]:
        """
        将任务分解为子任务列表。
        
        Args:
            task_description: 任务描述
            max_depth: 最大分解深度
        
        Returns:
            扁平化的子任务列表
        """
        print(f"\n{'='*50}")
        print(f"任务分解: {task_description[:80]}...")
        print(f"{'='*50}")
        
        # 第一步：生成高层分解
        high_level = self._decompose_level(task_description, level=1)
        
        all_tasks = []
        
        for i, subtask_info in enumerate(high_level):
            task = Task(
                description=subtask_info["description"],
                dependencies=subtask_info.get("dependencies", []),
                priority=subtask_info.get("priority", len(high_level) - i)
            )
            
            # 如果还有分解空间，继续分解
            if max_depth > 1 and subtask_info.get("needs_further_decomposition", False):
                sub_subtasks = self._decompose_level(
                    subtask_info["description"],
                    level=2
                )
                for j, sub_info in enumerate(sub_subtasks):
                    subtask = Task(
                        description=sub_info["description"],
                        dependencies=sub_info.get("dependencies", []),
                        parent_id=task.task_id,
                        priority=sub_info.get("priority", len(sub_subtasks) - j)
                    )
                    task.subtasks.append(subtask)
                    all_tasks.append(subtask)
            
            all_tasks.append(task)
        
        # 打印分解结果
        print(f"\n分解结果: 共 {len(all_tasks)} 个子任务")
        for task in all_tasks:
            deps = f" (依赖: {task.dependencies})" if task.dependencies else ""
            print(f"  [{task.task_id}] {task.description[:60]}...{deps}")
        
        return all_tasks
    
    def _decompose_level(self, description: str, level: int) -> list[dict]:
        """在一个层级上进行分解"""
        
        prompt = f"""请将以下任务分解为 2-5 个子任务。每个子任务应该：
1. 职责单一、边界清晰
2. 粒度适中（不太大也不太小）
3. 有明确的完成标准

任务描述: {description}

请以 JSON 数组格式输出，每个元素包含：
- description: 子任务描述
- dependencies: 依赖的其他子任务的索引号（列表），没有依赖则为空列表
- priority: 优先级（1-5，5最高）
- needs_further_decomposition: 是否需要进一步分解（布尔值）

只输出 JSON，不要其他文字。"""
        
        messages = [{"role": "user", "content": prompt}]
        result = call_llm(messages, model=self.model, temperature=0.3)
        
        try:
            json_start = result.index("[")
            json_end = result.rindex("]") + 1
            subtasks = json.loads(result[json_start:json_end])
            return subtasks
        except (ValueError, json.JSONDecodeError) as e:
            print(f"  解析失败，使用默认分解: {e}")
            return [{"description": description, "dependencies": [], "priority": 3}]


# ============================================================
# 任务调度器
# ============================================================

class TaskScheduler:
    """
    任务调度器：决定子任务的执行顺序。
    
    使用拓扑排序来确定执行顺序，确保依赖关系被正确处理。
    支持并行执行：没有依赖关系的任务可以同时执行。
    """
    
    def __init__(self):
        self.tasks: dict[str, Task] = {}
        self.completed_tasks: set[str] = set()
    
    def add_tasks(self, tasks: list[Task]):
        """添加任务到调度器"""
        for task in tasks:
            self.tasks[task.task_id] = task
    
    def get_executable_tasks(self) -> list[Task]:
        """获取当前可以执行的任务（所有依赖已完成）"""
        executable = []
        for task in self.tasks.values():
            if task.status == "pending" and task.is_ready(self.completed_tasks):
                executable.append(task)
        
        # 按优先级排序
        executable.sort(key=lambda t: t.priority, reverse=True)
        return executable
    
    def mark_completed(self, task_id: str, result: str = ""):
        """标记任务完成"""
        if task_id in self.tasks:
            self.tasks[task_id].status = "completed"
            self.tasks[task_id].result = result
            self.completed_tasks.add(task_id)
            print(f"  [调度器] 任务 {task_id} 已完成")
    
    def get_execution_plan(self) -> list[list[Task]]:
        """
        生成执行计划。
        
        返回一个二维列表，每个元素是一组可以并行执行的任务。
        例如：[[task1, task2], [task3], [task4, task5]]
        表示 task1 和 task2 可以并行，然后执行 task3，最后 task4 和 task5 并行。
        """
        remaining = {tid: task for tid, task in self.tasks.items() if task.status == "pending"}
        completed = set()
        plan = []
        
        while remaining:
            # 找出可以执行的任务
            batch = []
            for tid, task in remaining.items():
                if task.is_ready(completed):
                    batch.append(task)
            
            if not batch:
                print("  [调度器] 警告: 存在循环依赖或无法完成的任务!")
                break
            
            plan.append(batch)
            for task in batch:
                completed.add(task.task_id)
                del remaining[task.task_id]
        
        return plan
    
    def print_plan(self, plan: list[list[Task]]):
        """打印执行计划"""
        print(f"\n--- 执行计划 ({len(plan)} 个阶段) ---")
        for i, batch in enumerate(plan):
            task_descs = [f"{t.task_id}({t.description[:30]})" for t in batch]
            parallel = "并行" if len(batch) > 1 else "串行"
            print(f"  阶段 {i+1} [{parallel}]: {', '.join(task_descs)}")


# ============================================================
# 演示：任务分解与调度
# ============================================================

def demo_task_decomposition():
    """演示任务分解和调度"""
    
    print("\n" + "="*60)
    print("演示：任务分解与调度")
    print("="*60)
    
    # 创建任务分解器
    decomposer = TaskDecomposer()
    
    # 分解一个复杂任务
    tasks = decomposer.decompose(
        "开发一个 Multi-Agent 协作系统，包括 Agent 通信、任务协调和结果整合",
        max_depth=1
    )
    
    # 创建调度器
    scheduler = TaskScheduler()
    scheduler.add_tasks(tasks)
    
    # 生成执行计划
    plan = scheduler.get_execution_plan()
    scheduler.print_plan(plan)
    
    # 模拟执行
    print("\n--- 模拟执行 ---")
    for batch in plan:
        print(f"\n  执行批次: {[t.task_id for t in batch]}")
        for task in batch:
            print(f"    执行任务 {task.task_id}: {task.description[:50]}...")
            scheduler.mark_completed(task.task_id, f"{task.description[:20]} 的结果")


if __name__ == "__main__":
    demo_task_decomposition()
```

### 25.2.2 基于能力的分解

除了自上而下的分解，还有一种基于能力的分解方式：先了解每个 Agent 擅长什么，然后根据 Agent 的能力来划分任务。这种方式更适合 Agent 能力差异较大的场景。

---

## 25.3 任务分配策略

任务分解完成后，下一步是将子任务分配给最合适的 Agent。分配策略的好坏直接影响系统的整体效率和产出质量。

### 25.3.1 最简单的分配：轮流制

最简单的分配方式是轮流分配——Agent 1 做第一个任务，Agent 2 做第二个，以此类推。这种方式公平但不高效，因为它没有考虑 Agent 的能力和任务的特点。

### 25.3.2 能力匹配分配

更合理的方式是根据 Agent 的能力和任务的要求进行匹配。比如，一个擅长数据分析的 Agent 应该被分配数据分析类的任务，一个擅长写作的 Agent 应该被分配写作类的任务。

```python
"""
第 25 章：任务分配策略
实现基于能力匹配和负载均衡的分配
"""
import os
import json
import time
from typing import Any
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
# Agent 能力模型
# ============================================================

@dataclass
class AgentProfile:
    """
    Agent 能力画像。
    
    描述一个 Agent 的能力、专长和当前状态。
    用于任务分配时的匹配计算。
    """
    name: str
    skills: list[str] = field(default_factory=list)  # 技能列表
    specialties: list[str] = field(default_factory=list)  # 专长领域
    current_load: int = 0  # 当前负载（进行中的任务数）
    max_load: int = 3  # 最大负载
    completed_tasks: int = 0  # 已完成任务数
    avg_quality: float = 1.0  # 平均质量评分（0-1）
    cost_per_task: float = 1.0  # 每个任务的成本（可以是 token 消耗等）
    
    @property
    def available(self) -> bool:
        """检查 Agent 是否可用"""
        return self.current_load < self.max_load
    
    @property
    def efficiency_score(self) -> float:
        """综合效率评分"""
        return self.avg_quality * (1 - self.current_load / self.max_load)
    
    def to_description(self) -> str:
        """生成能力描述文本"""
        return f"""Agent: {self.name}
技能: {', '.join(self.skills)}
专长: {', '.join(self.specialties)}
当前负载: {self.current_load}/{self.max_load}
平均质量: {self.avg_quality:.1f}
效率评分: {self.efficiency_score:.2f}"""


# ============================================================
# 任务分配器
# ============================================================

class TaskAllocator:
    """
    任务分配器：将任务分配给最合适的 Agent。
    
    支持三种分配策略：
    1. 能力匹配：根据 Agent 的技能和任务要求匹配
    2. 负载均衡：优先分配给负载低的 Agent
    3. 质量优先：优先分配给质量评分高的 Agent
    
    实际使用时通常组合多种策略。
    """
    
    def __init__(self, agents: list[AgentProfile], strategy: str = "balanced"):
        """
        Args:
            agents: 可用的 Agent 列表
            strategy: 分配策略 ("capability", "load_balanced", "quality_first", "balanced")
        """
        self.agents = {a.name: a for a in agents}
        self.strategy = strategy
    
    def allocate(self, task_desc: str, task_type: str = "general") -> AgentProfile:
        """
        为一个任务分配 Agent。
        
        Args:
            task_desc: 任务描述
            task_type: 任务类型
        
        Returns:
            被分配的 Agent
        """
        available_agents = [a for a in self.agents.values() if a.available]
        
        if not available_agents:
            print(f"  [分配器] 警告: 没有可用的 Agent!")
            # 返回负载最低的 Agent（即使超载）
            return min(self.agents.values(), key=lambda a: a.current_load)
        
        if self.strategy == "capability":
            return self._allocate_by_capability(available_agents, task_desc, task_type)
        elif self.strategy == "load_balanced":
            return self._allocate_by_load(available_agents)
        elif self.strategy == "quality_first":
            return self._allocate_by_quality(available_agents)
        else:  # balanced
            return self._allocate_balanced(available_agents, task_desc, task_type)
    
    def _allocate_by_capability(
        self, agents: list[AgentProfile], task_desc: str, task_type: str
    ) -> AgentProfile:
        """按能力匹配分配"""
        best_agent = None
        best_score = -1
        
        for agent in agents:
            # 计算能力匹配度
            score = 0
            for skill in agent.skills:
                if skill.lower() in task_desc.lower() or skill.lower() in task_type.lower():
                    score += 2
            for specialty in agent.specialties:
                if specialty.lower() in task_desc.lower():
                    score += 3
            
            if score > best_score:
                best_score = score
                best_agent = agent
        
        return best_agent or agents[0]
    
    def _allocate_by_load(self, agents: list[AgentProfile]) -> AgentProfile:
        """按负载均衡分配"""
        return min(agents, key=lambda a: a.current_load)
    
    def _allocate_by_quality(self, agents: list[AgentProfile]) -> AgentProfile:
        """按质量优先分配"""
        return max(agents, key=lambda a: a.avg_quality)
    
    def _allocate_balanced(
        self, agents: list[AgentProfile], task_desc: str, task_type: str
    ) -> AgentProfile:
        """综合平衡分配"""
        best_agent = None
        best_score = -1
        
        for agent in agents:
            # 综合评分 = 能力匹配 * 0.4 + 空闲度 * 0.3 + 质量 * 0.3
            capability_score = 0
            for skill in agent.skills:
                if skill.lower() in task_desc.lower():
                    capability_score += 1
            for specialty in agent.specialties:
                if specialty.lower() in task_desc.lower():
                    capability_score += 1.5
            
            idle_score = 1 - (agent.current_load / agent.max_load)
            quality_score = agent.avg_quality
            
            total_score = capability_score * 0.4 + idle_score * 0.3 + quality_score * 0.3
            
            if total_score > best_score:
                best_score = total_score
                best_agent = agent
        
        return best_agent or agents[0]
    
    def batch_allocate(self, tasks: list[dict]) -> dict[str, str]:
        """
        批量分配任务。
        
        Args:
            tasks: 任务列表，每个任务包含 description 和 type
        
        Returns:
            {task_description: agent_name} 的映射
        """
        allocation = {}
        
        for task in tasks:
            agent = self.allocate(task["description"], task.get("type", "general"))
            agent.current_load += 1
            allocation[task["description"]] = agent.name
            print(f"  [分配器] '{task['description'][:40]}...' → {agent.name}")
        
        return allocation


# ============================================================
# 演示：任务分配
# ============================================================

def demo_task_allocation():
    """演示任务分配策略"""
    
    print("\n" + "="*60)
    print("演示：基于能力匹配的任务分配")
    print("="*60)
    
    # 定义 Agent 能力
    agents = [
        AgentProfile(
            name="Alice",
            skills=["python", "数据分析", "机器学习"],
            specialties=["数据处理", "统计分析", "可视化"],
            max_load=3
        ),
        AgentProfile(
            name="Bob",
            skills=["写作", "编辑", "校对"],
            specialties=["技术文档", "博客文章", "报告撰写"],
            max_load=2
        ),
        AgentProfile(
            name="Charlie",
            skills=["前端开发", "UI设计", "CSS"],
            specialties=["用户界面", "交互设计", "响应式布局"],
            max_load=3
        ),
    ]
    
    # 打印 Agent 能力
    print("\n--- Agent 能力列表 ---")
    for agent in agents:
        print(f"\n{agent.to_description()}")
    
    # 定义任务
    tasks = [
        {"description": "分析用户行为数据并生成统计报告", "type": "数据分析"},
        {"description": "撰写 Multi-Agent 系统的技术文档", "type": "写作"},
        {"description": "设计 Agent 交互界面的原型", "type": "UI设计"},
        {"description": "处理和清洗训练数据集", "type": "数据处理"},
        {"description": "编写系统使用指南", "type": "写作"},
    ]
    
    # 分配任务
    allocator = TaskAllocator(agents, strategy="balanced")
    
    print("\n--- 任务分配结果 ---")
    allocation = allocator.batch_allocate(tasks)
    
    # 展示分配后的负载
    print("\n--- 分配后负载 ---")
    for agent in agents:
        print(f"  {agent.name}: {agent.current_load}/{agent.max_load} 任务")


if __name__ == "__main__":
    demo_task_allocation()
```

### 25.3.3 合同网协议（Contract Net Protocol）

合同网协议是一种经典的分布式任务分配机制，灵感来源于市场经济中的招标-投标-中标过程。在这种模式中：

1. **发布者**（相当于甲方）发布任务公告
2. **潜在执行者**（相当于乙方）根据自己的能力评估任务，提交"投标"
3. **发布者**根据投标方案选择最合适的执行者，发出"中标通知"
4. **中标者**开始执行任务

这种模式的优势在于它是分布式的——不需要一个中央权威来决定谁做什么，而是通过市场机制让 Agent 自主竞争。

```python
"""
第 25 章：合同网协议实现
模拟招标-投标-中标的任务分配过程
"""
import os
import json
import time
import random
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
# 合同网协议
# ============================================================

@dataclass
class TaskAnnouncement:
    """任务公告"""
    task_id: str
    description: str
    publisher: str
    deadline: float = 0.0  # 截止时间
    max_cost: float = 10.0  # 最大预算
    requirements: list[str] = field(default_factory=list)


@dataclass
class Bid:
    """投标方案"""
    task_id: str
    bidder: str
    estimated_cost: float  # 预估成本
    estimated_time: float  # 预估时间
    confidence: float  # 完成信心（0-1）
    proposal: str  # 方案描述
    
    @property
    def score(self) -> float:
        """综合评分（越高越好）"""
        return self.confidence / (self.estimated_cost + 0.1) * 10


@dataclass
class Contract:
    """合同（中标结果）"""
    task_id: str
    task_description: str
    contractor: str  # 承包者
    agreed_cost: float
    agreed_time: float
    status: str = "active"  # active, completed, failed


class ContractNetManager:
    """
    合同网管理者：协调招标-投标-中标过程。
    
    工作流程：
    1. 发布任务公告
    2. 收集投标方案
    3. 评标并选择中标者
    4. 签订合同
    5. 监督执行
    """
    
    def __init__(self):
        self.announcements: dict[str, TaskAnnouncement] = {}
        self.bids: dict[str, list[Bid]] = {}
        self.contracts: dict[str, Contract] = {}
    
    def announce_task(self, announcement: TaskAnnouncement):
        """发布任务公告"""
        self.announcements[announcement.task_id] = announcement
        self.bids[announcement.task_id] = []
        print(f"  [合同网] 发布任务公告: {announcement.task_id}")
        print(f"    描述: {announcement.description[:60]}...")
        print(f"    预算: {announcement.max_cost}")
    
    def submit_bid(self, bid: Bid):
        """提交投标"""
        if bid.task_id in self.bids:
            self.bids[bid.task_id].append(bid)
            print(f"  [合同网] 收到投标: {bid.bidder} for {bid.task_id}")
            print(f"    成本: {bid.estimated_cost}, 信心: {bid.confidence:.1%}")
    
    def evaluate_and_award(self, task_id: str) -> Optional[Contract]:
        """评标并签订合同"""
        if task_id not in self.bids or not self.bids[task_id]:
            print(f"  [合同网] 任务 {task_id} 没有收到投标")
            return None
        
        bids = self.bids[task_id]
        announcement = self.announcements[task_id]
        
        # 按综合评分排序
        valid_bids = [b for b in bids if b.estimated_cost <= announcement.max_cost]
        if not valid_bids:
            print(f"  [合同网] 所有投标都超出预算")
            return None
        
        valid_bids.sort(key=lambda b: b.score, reverse=True)
        winner = valid_bids[0]
        
        # 签订合同
        contract = Contract(
            task_id=task_id,
            task_description=announcement.description,
            contractor=winner.bidder,
            agreed_cost=winner.estimated_cost,
            agreed_time=winner.estimated_time
        )
        self.contracts[task_id] = contract
        
        print(f"\n  [合同网] 评标结果:")
        for b in valid_bids:
            print(f"    {b.bidder}: 评分 {b.score:.2f}")
        print(f"  [合同网] 中标者: {winner.bidder}")
        
        return contract


class BiddingAgent:
    """投标 Agent"""
    
    def __init__(self, name: str, skills: list[str], model: str = "gpt-4o"):
        self.name = name
        self.skills = skills
        self.model = model
    
    def evaluate_task(self, announcement: TaskAnnouncement) -> Optional[Bid]:
        """
        评估任务并决定是否投标。
        
        Agent 会根据自己的能力评估任务的难度和所需资源，
        然后决定是否投标以及投标方案。
        """
        # 基于技能匹配的简单评估
        skill_match = sum(1 for skill in self.skills 
                        if skill.lower() in announcement.description.lower())
        
        if skill_match == 0:
            print(f"  [{self.name}] 技能不匹配，不投标")
            return None
        
        # 模拟评估过程
        confidence = min(0.9, 0.5 + skill_match * 0.15)
        estimated_cost = max(1.0, announcement.max_cost * (1 - confidence) * 0.8)
        estimated_time = random.uniform(1, 5)
        
        bid = Bid(
            task_id=announcement.task_id,
            bidder=self.name,
            estimated_cost=estimated_cost,
            estimated_time=estimated_time,
            confidence=confidence,
            proposal=f"使用 {', '.join(self.skills[:2])} 技能完成任务"
        )
        
        return bid


# ============================================================
# 演示：合同网协议
# ============================================================

def demo_contract_net():
    """演示合同网协议"""
    
    print("\n" + "="*60)
    print("演示：合同网协议")
    print("="*60)
    
    # 创建合同网管理者
    manager = ContractNetManager()
    
    # 创建投标 Agent
    agents = [
        BiddingAgent("数据分析Agent", ["数据分析", "Python", "统计"]),
        BiddingAgent("写作Agent", ["写作", "编辑", "文档"]),
        BiddingAgent("全栈Agent", ["Python", "写作", "数据分析"]),
    ]
    
    # 发布任务
    announcement = TaskAnnouncement(
        task_id="T001",
        description="分析销售数据并撰写分析报告",
        publisher="项目经理",
        max_cost=8.0,
        requirements=["数据分析", "报告撰写"]
    )
    manager.announce_task(announcement)
    
    # 各 Agent 评估并投标
    print("\n--- 投标阶段 ---")
    for agent in agents:
        bid = agent.evaluate_task(announcement)
        if bid:
            manager.submit_bid(bid)
    
    # 评标和签订合同
    print("\n--- 评标阶段 ---")
    contract = manager.evaluate_and_award("T001")
    
    if contract:
        print(f"\n--- 合同详情 ---")
        print(f"  任务: {contract.task_description[:50]}...")
        print(f"  承包者: {contract.contractor}")
        print(f"  成本: {contract.agreed_cost}")
        print(f"  预计时间: {contract.agreed_time:.1f} 小时")


if __name__ == "__main__":
    demo_contract_net()
```

---

## 25.4 群体决策机制

### 25.4.1 投票机制

当多个 Agent 对同一个问题有不同的看法时，投票是一种简单而有效的决策方式。每个 Agent 投出自己的一票，得票最多的选项获胜。

但投票也有多种变体。简单多数投票（得票最多者获胜）是最常见的，但也可能产生"少数派被忽略"的问题。加权投票（根据 Agent 的专业度给予不同权重）能部分解决这个问题。排序投票（每个 Agent 对选项进行排序，然后计算综合得分）能更精确地反映群体的偏好。

```python
"""
第 25 章：群体决策机制
实现投票、辩论和共识达成
"""
import os
import json
import time
from typing import Any
from dataclasses import dataclass, field
from collections import Counter
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
# 投票系统
# ============================================================

class VotingSystem:
    """
    Multi-Agent 投票决策系统。
    
    支持三种投票方式：
    1. 简单多数投票：每个 Agent 一票，得票最多者获胜
    2. 加权投票：根据 Agent 的专业度给予不同权重
    3. 排序投票：每个 Agent 对选项排序，计算综合得分
    """
    
    def __init__(self):
        self.votes: list[dict] = []
    
    def simple_vote(self, votes: list[str]) -> dict:
        """
        简单多数投票。
        
        Args:
            votes: 每个 Agent 的投票（选项名称）
        
        Returns:
            包含获胜者和投票统计的字典
        """
        counter = Counter(votes)
        winner = counter.most_common(1)[0]
        
        result = {
            "method": "简单多数",
            "winner": winner[0],
            "winning_votes": winner[1],
            "total_votes": len(votes),
            "distribution": dict(counter)
        }
        
        print(f"\n  [投票] 方法: 简单多数")
        print(f"  [投票] 分布: {dict(counter)}")
        print(f"  [投票] 获胜: {result['winner']} ({result['winning_votes']}/{result['total_votes']} 票)")
        
        return result
    
    def weighted_vote(
        self, votes: list[tuple[str, float]]
    ) -> dict:
        """
        加权投票。
        
        Args:
            votes: (选项名称, 权重) 的列表
        
        Returns:
            包含获胜者和加权统计的字典
        """
        weighted_scores: dict[str, float] = {}
        
        for option, weight in votes:
            weighted_scores[option] = weighted_scores.get(option, 0) + weight
        
        winner = max(weighted_scores.items(), key=lambda x: x[1])
        
        result = {
            "method": "加权投票",
            "winner": winner[0],
            "winning_score": winner[1],
            "distribution": weighted_scores
        }
        
        print(f"\n  [投票] 方法: 加权投票")
        print(f"  [投票] 加权得分: {weighted_scores}")
        print(f"  [投票] 获胜: {result['winner']} (得分: {result['winning_score']:.2f})")
        
        return result
    
    def ranked_vote(self, rankings: list[list[str]]) -> dict:
        """
        排序投票（Borda 计数法）。
        
        每个 Agent 对所有选项排序，排名越靠前得分越高。
        最后累加所有 Agent 的得分，得分最高者获胜。
        
        Args:
            rankings: 每个 Agent 的排序列表（从最偏好到最不偏好）
        
        Returns:
            包含获胜者和得分统计的字典
        """
        # 收集所有选项
        all_options = set()
        for ranking in rankings:
            all_options.update(ranking)
        
        # Borda 计分
        scores = {opt: 0 for opt in all_options}
        for ranking in rankings:
            n = len(ranking)
            for i, option in enumerate(ranking):
                scores[option] += (n - 1 - i)  # 排名越前得分越高
        
        winner = max(scores.items(), key=lambda x: x[1])
        
        result = {
            "method": "排序投票 (Borda)",
            "winner": winner[0],
            "winning_score": winner[1],
            "distribution": scores
        }
        
        print(f"\n  [投票] 方法: 排序投票 (Borda)")
        print(f"  [投票] 得分: {scores}")
        print(f"  [投票] 获胜: {result['winner']} (得分: {result['winning_score']})")
        
        return result


# ============================================================
# 共识达成系统
# ============================================================

class ConsensusSystem:
    """
    共识达成系统：通过多轮讨论让 Agent 达成一致意见。
    
    工作流程：
    1. 每个 Agent 表达自己的观点
    2. Agent 之间相互讨论和质疑
    3. Agent 根据他人的意见调整自己的立场
    4. 重复直到达成共识或达到最大轮数
    """
    
    def __init__(self, model: str = "gpt-4o"):
        self.model = model
    
    def reach_consensus(
        self,
        topic: str,
        agent_opinions: dict[str, str],
        max_rounds: int = 3
    ) -> dict:
        """
        通过多轮讨论达成共识。
        
        Args:
            topic: 讨论话题
            agent_opinions: {Agent名称: 初始观点}
            max_rounds: 最大讨论轮数
        
        Returns:
            最终共识和讨论过程
        """
        print(f"\n{'='*50}")
        print(f"共识达成: {topic}")
        print(f"参与 Agent: {list(agent_opinions.keys())}")
        print(f"{'='*50}")
        
        opinions = dict(agent_opinions)
        discussion_log = []
        
        for round_num in range(1, max_rounds + 1):
            print(f"\n--- 第 {round_num} 轮讨论 ---")
            
            new_opinions = {}
            
            for agent_name, current_opinion in opinions.items():
                # 获取其他 Agent 的观点
                others_opinions = {
                    name: op for name, op in opinions.items() if name != agent_name
                }
                
                prompt = f"""你是一个 AI Agent，名叫「{agent_name}」。

讨论话题: {topic}

你的当前观点: {current_opinion}

其他 Agent 的观点:
{json.dumps(others_opinions, ensure_ascii=False, indent=2)}

请参考其他 Agent 的观点，如果你认为他们的观点有道理，可以调整自己的立场。
如果你仍然坚持自己的观点，请说明理由。

请输出你经过思考后的最新观点（100-200字）："""
                
                messages = [{"role": "user", "content": prompt}]
                new_opinion = call_llm(messages, model=self.model, temperature=0.5)
                
                new_opinions[agent_name] = new_opinion
                print(f"  [{agent_name}] 更新观点: {new_opinion[:80]}...")
                
                discussion_log.append({
                    "round": round_num,
                    "agent": agent_name,
                    "opinion": new_opinion
                })
            
            opinions = new_opinions
        
        # 最终总结
        all_opinions_text = "\n".join([
            f"{name}: {op}" for name, op in opinions.items()
        ])
        
        summary_prompt = f"""以下是多个 AI Agent 对「{topic}」的最终讨论结果：

{all_opinions_text}

请综合所有 Agent 的观点，给出一个最终的共识总结（200-300字）。"""
        
        messages = [{"role": "user", "content": summary_prompt}]
        consensus = call_llm(messages, model=self.model)
        
        print(f"\n--- 最终共识 ---")
        print(f"  {consensus[:200]}...")
        
        return {
            "topic": topic,
            "final_consensus": consensus,
            "final_opinions": opinions,
            "discussion_log": discussion_log
        }


# ============================================================
# 演示：群体决策
# ============================================================

def demo_group_decision():
    """演示群体决策机制"""
    
    print("\n" + "="*60)
    print("演示：群体决策机制")
    print("="*60)
    
    # --- 投票演示 ---
    print("\n=== 投票决策 ===")
    
    voting = VotingSystem()
    
    # 简单多数投票
    votes = ["方案A", "方案B", "方案A", "方案C", "方案A", "方案B", "方案A"]
    voting.simple_vote(votes)
    
    # 加权投票
    weighted_votes = [
        ("方案A", 0.9),  # 专家1推荐A，权重0.9
        ("方案B", 0.6),  # 专家2推荐B，权重0.6
        ("方案A", 0.7),  # 专家3推荐A，权重0.7
        ("方案C", 0.8),  # 专家4推荐C，权重0.8
    ]
    voting.weighted_vote(weighted_votes)
    
    # 排序投票
    rankings = [
        ["方案A", "方案B", "方案C"],
        ["方案B", "方案A", "方案C"],
        ["方案A", "方案C", "方案B"],
        ["方案C", "方案A", "方案B"],
    ]
    voting.ranked_vote(rankings)


if __name__ == "__main__":
    demo_group_decision()
```

---

## 25.5 冲突检测与解决

### 25.5.1 冲突的类型

在 Multi-Agent 系统中，冲突主要有以下几种类型：

**目标冲突**：不同 Agent 的目标相互矛盾。比如，成本优化 Agent 希望使用更便宜的模型，而质量保证 Agent 希望使用更强大的模型。

**资源冲突**：多个 Agent 争抢同一个有限资源。比如，两个 Agent 同时需要写入同一个文件。

**知识冲突**：Agent 之间对同一个事实有不同的认知。比如，Agent A 认为某个 API 已经废弃，而 Agent B 认为它还在使用。

**时间冲突**：多个 Agent 需要在同一时间段内使用同一个资源，但资源只能支持一个使用者。

### 25.5.2 冲突解决策略

```python
"""
第 25 章：冲突解决机制
检测和解决 Agent 之间的冲突
"""
import os
import json
import time
from typing import Any, Optional
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
# 冲突检测与解决
# ============================================================

@dataclass
class Conflict:
    """冲突描述"""
    conflict_id: str
    conflict_type: str  # goal, resource, knowledge, time
    agents_involved: list[str]
    description: str
    severity: str = "medium"  # low, medium, high


class ConflictResolver:
    """
    冲突解决器。
    
    检测 Agent 之间的冲突，并尝试通过协商来解决。
    如果协商失败，则由仲裁者做出最终裁决。
    """
    
    def __init__(self, model: str = "gpt-4o"):
        self.model = model
        self.conflicts: list[Conflict] = []
        self.resolutions: list[dict] = []
    
    def detect_conflicts(
        self, agent_outputs: dict[str, str], topic: str
    ) -> list[Conflict]:
        """
        检测 Agent 输出之间的冲突。
        
        使用 LLM 来分析多个 Agent 的输出，识别潜在的矛盾和冲突。
        """
        outputs_text = "\n".join([
            f"【{name}】: {output[:300]}" 
            for name, output in agent_outputs.items()
        ])
        
        prompt = f"""请分析以下多个 Agent 对「{topic}」的输出，检测是否存在冲突：

{outputs_text}

请以 JSON 数组格式输出发现的冲突（如果没有冲突则输出空数组）：
[
    {{
        "conflict_type": "goal/resource/knowledge/time",
        "agents_involved": ["Agent A", "Agent B"],
        "description": "冲突描述",
        "severity": "low/medium/high"
    }}
]

只输出 JSON，不要其他文字。"""
        
        messages = [{"role": "user", "content": prompt}]
        result = call_llm(messages, model=self.model, temperature=0.3)
        
        try:
            json_start = result.index("[")
            json_end = result.rindex("]") + 1
            conflicts_data = json.loads(result[json_start:json_end])
            
            conflicts = []
            for i, cd in enumerate(conflicts_data):
                conflict = Conflict(
                    conflict_id=f"C{i+1:03d}",
                    conflict_type=cd.get("conflict_type", "unknown"),
                    agents_involved=cd.get("agents_involved", []),
                    description=cd.get("description", ""),
                    severity=cd.get("severity", "medium")
                )
                conflicts.append(conflict)
            
            self.conflicts.extend(conflicts)
            return conflicts
            
        except (ValueError, json.JSONDecodeError):
            print("  [冲突检测] 解析失败，假设无冲突")
            return []
    
    def resolve_by_negotiation(
        self, conflict: Conflict, agent_positions: dict[str, str]
    ) -> dict:
        """
        通过协商解决冲突。
        
        让涉及冲突的 Agent 各自阐述立场，然后尝试找到折中方案。
        """
        print(f"\n  [解决] 尝试通过协商解决冲突 {conflict.conflict_id}")
        
        positions_text = "\n".join([
            f"{name}: {pos}" for name, pos in agent_positions.items()
        ])
        
        prompt = f"""以下是两个 AI Agent 在「{conflict.description}」上的分歧：

{positions_text}

请作为调解人，帮助这两个 Agent 达成一个双方都能接受的折中方案。

要求：
1. 尊重双方的合理诉求
2. 找到共同点和互补之处
3. 提出一个具体的、可执行的方案
4. 解释为什么这个方案对双方都有利

请输出调解结果（200-300字）："""
        
        messages = [{"role": "user", "content": prompt}]
        resolution = call_llm(messages, model=self.model)
        
        result = {
            "conflict_id": conflict.conflict_id,
            "method": "协商",
            "resolution": resolution,
            "status": "resolved"
        }
        
        self.resolutions.append(result)
        print(f"  [解决] 协商结果: {resolution[:100]}...")
        
        return result
    
    def resolve_by_arbitration(
        self, conflict: Conflict, agent_positions: dict[str, str]
    ) -> dict:
        """
        通过仲裁解决冲突。
        
        当协商失败时，由仲裁者根据客观标准做出最终裁决。
        """
        print(f"\n  [仲裁] 对冲突 {conflict.conflict_id} 进行仲裁")
        
        positions_text = "\n".join([
            f"{name}: {pos}" for name, pos in agent_positions.items()
        ])
        
        prompt = f"""你是一位公正的仲裁者。以下两个 AI Agent 在「{conflict.description}」上存在无法调和的分歧：

{positions_text}

请根据以下原则做出裁决：
1. 以事实和逻辑为依据
2. 考虑整体利益最大化
3. 确保裁决的公平性
4. 给出详细的裁决理由

请输出你的裁决结果（200-300字）："""
        
        messages = [{"role": "user", "content": prompt}]
        arbitration = call_llm(messages, model=self.model, temperature=0.3)
        
        result = {
            "conflict_id": conflict.conflict_id,
            "method": "仲裁",
            "resolution": arbitration,
            "status": "resolved"
        }
        
        self.resolutions.append(result)
        print(f"  [仲裁] 裁决结果: {arbitration[:100]}...")
        
        return result
    
    def auto_resolve(
        self, conflict: Conflict, agent_positions: dict[str, str]
    ) -> dict:
        """
        自动选择解决策略。
        
        优先尝试协商，如果协商失败（模拟判断）则使用仲裁。
        对于低优先级冲突，使用简单的优先级规则。
        """
        if conflict.severity == "low":
            # 低优先级冲突：直接使用优先级规则
            return {
                "conflict_id": conflict.conflict_id,
                "method": "优先级规则",
                "resolution": f"低优先级冲突，采用第一个 Agent 的方案: {list(agent_positions.values())[0][:100]}",
                "status": "resolved"
            }
        else:
            # 中高优先级：先尝试协商
            return self.resolve_by_negotiation(conflict, agent_positions)


# ============================================================
# 演示：冲突检测与解决
# ============================================================

def demo_conflict_resolution():
    """演示冲突检测与解决"""
    
    print("\n" + "="*60)
    print("演示：冲突检测与解决")
    print("="*60)
    
    resolver = ConflictResolver()
    
    # 模拟 Agent 输出存在冲突
    agent_outputs = {
        "成本优化Agent": "建议使用 GPT-3.5-turbo 模型，因为它的性价比最高，可以在保证基本质量的前提下将成本降低 60%。",
        "质量保证Agent": "建议使用 GPT-4 模型，因为在关键业务场景下，输出质量直接影响用户体验和业务指标，不建议为成本牺牲质量。",
        "安全审计Agent": "建议在使用任何外部模型前，先进行安全审查，确保数据不会泄露到不安全的环境。"
    }
    
    # 检测冲突
    print("\n--- 检测冲突 ---")
    conflicts = resolver.detect_conflicts(agent_outputs, "选择 LLM 模型")
    
    for conflict in conflicts:
        print(f"\n  冲突 {conflict.conflict_id}:")
        print(f"    类型: {conflict.conflict_type}")
        print(f"    涉及: {conflict.agents_involved}")
        print(f"    描述: {conflict.description}")
        print(f"    严重度: {conflict.severity}")
    
    # 解决冲突
    if conflicts:
        positions = {
            "成本优化Agent": agent_outputs["成本优化Agent"],
            "质量保证Agent": agent_outputs["质量保证Agent"]
        }
        resolver.resolve_by_negotiation(conflicts[0], positions)


if __name__ == "__main__":
    demo_conflict_resolution()
```

---

## 25.6 案例分析：协调策略在实际项目中的应用

### 25.6.1 案例：Multi-Agent 数据分析平台

某数据分析平台使用 Multi-Agent 系统来自动处理数据查询请求。系统的协调策略如下：

**任务分解**：将用户的自然语言查询分解为数据获取、数据清洗、分析计算、结果可视化四个子任务。

**任务分配**：使用能力匹配策略。数据获取 Agent 负责 SQL 生成和执行，数据清洗 Agent 负责缺失值处理和异常值检测，分析计算 Agent 负责统计分析，可视化 Agent 负责图表生成。

**调度策略**：数据获取必须先于数据清洗（依赖关系），数据清洗和分析计算可以部分并行（在数据量足够大时），可视化必须在分析计算之后。

**冲突解决**：当多个用户同时查询时，使用优先级调度——VIP 用户的查询优先处理。当数据源出现不一致时，使用权威数据源优先的规则。

---

## 25.7 常见坑与解决方案

### 坑 1：过度协调导致系统僵化

**问题**：过度依赖中央协调者，导致系统变得僵化，无法灵活应对突发情况。

**解决方案**：采用混合协调策略——常规任务使用预定义的协调规则，突发情况使用基于 LLM 的动态协调。

### 坑 2：任务分解粒度不当

**问题**：分解得太细导致通信开销大，分解得太粗导致单个 Agent 负担重。

**解决方案**：根据 Agent 的处理能力和任务的复杂度动态调整分解粒度。

### 坑 3：死锁和活锁

**问题**：Agent 之间出现循环等待（死锁）或无意义的重复操作（活锁）。

**解决方案**：设置超时机制和最大循环次数，使用心跳检测发现死锁。

### 坑 4：协调者成为瓶颈

**问题**：所有决策都集中在一个协调者身上，当 Agent 数量增多时协调者处理不过来。

**解决方案**：引入层级协调——每个子团队有自己的协调者，主协调者只处理跨团队的协调。

### 坑 5：决策质量不稳定

**问题**：基于 LLM 的决策质量受 prompt 和上下文影响较大，有时做出不理想的决策。

**解决方案**：引入多轮验证机制——重要决策需要多个 Agent 独立评估，结果一致才执行。

---

## 25.8 练习题

### 练习 1：实现任务依赖图可视化

**难度：★☆☆☆☆**

实现一个任务依赖图的文本可视化功能。给定一组带有依赖关系的任务，以文本方式绘制出任务之间的依赖关系图。

### 练习 2：实现拍卖式任务分配

**难度：★★☆☆☆**

实现一种拍卖式任务分配机制：Agent 对任务进行竞价，出价最低（成本最低）的 Agent 获得任务。

### 练习 3：实现基于议价的协商

**难度：★★★☆☆**

实现一个 Agent 间的议价协商系统：买卖双方 Agent 就价格进行多轮议价，最终达成交易。

### 练习 4：实现投票系统

**难度：★★★☆☆**

实现一个支持多种投票方式的系统：简单多数、加权投票、排序投票、淘汰投票。对比不同方式的结果差异。

### 练习 5：实现冲突自动解决

**难度：★★★★☆**

构建一个自动冲突检测和解决系统：监控多个 Agent 的输出，自动检测矛盾和冲突，并选择合适的策略解决。

---

## 25.9 实战任务

### 任务：构建 Multi-Agent 项目管理系统

**目标**：构建一个模拟项目管理的 Multi-Agent 系统，实现任务的自动分解、分配、执行和监控。

**功能要求**：
1. 任务分解：将项目目标自动分解为可执行的子任务
2. 任务分配：基于 Agent 能力进行智能分配
3. 进度跟踪：实时监控各 Agent 的任务进度
4. 冲突处理：自动检测和解决 Agent 之间的冲突

---

## 25.10 本章小结

本章深入探讨了 Multi-Agent 协调的核心策略和模式。我们学习了任务分解、任务分配、群体决策和冲突解决四大核心领域。

任务分解是协调的基础，好的分解决定了后续所有环节的效率。任务分配需要考虑 Agent 的能力、负载和质量，合同网协议提供了一种优雅的分布式分配方案。群体决策机制（投票、辩论、共识）让多个 Agent 能够在有分歧时达成一致。冲突检测和解决确保系统在面对矛盾时能够正常运转。

> "协调不是控制，而是让每个 Agent 在正确的时间做正确的事，并确保它们的行动相互配合而非相互干扰。"

---

**下一章预告**：在第 26 章中，我们将探讨角色化 Multi-Agent 系统——如何为每个 Agent 赋予独特的角色和专业能力，实现真正的专业分工。敬请期待。
