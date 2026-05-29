# 第 16 章：Planning 进阶 —— 层次化规划与动态调整

---

## 学习目标

完成本章学习后，你将能够：

1. 理解为什么需要层次化规划，以及它如何降低复杂任务的管理难度
2. 掌握任务分解的多种策略——功能分解、数据流分解、依赖分析，知道何时用哪种
3. 能够实现支持动态调整的规划系统，让计划在执行过程中自适应变化
4. 理解规划中的资源管理和约束处理，避免"计划很美好，执行行不通"的尴尬
5. 掌握多 Agent 协作规划的模式，理解多个 Agent 如何协同制定一个统一的计划
6. 了解如何评估和优化规划系统的性能

## 核心问题

1. 层次化规划解决了什么问题？为什么不把所有步骤放在同一层？
2. 如何在规划中处理资源限制和约束？
3. 多 Agent 协作规划时如何保持一致性？

---

## 16.1 层次化规划

### 16.1.1 为什么需要层次化规划

上一章我们学了 CoT、ToT、GoA 等规划策略，它们在处理中等复杂度的任务时表现不错。但当你面对一个真正复杂的任务时——比如"开发一个完整的电商平台"——你会发现这些策略都有些力不从心。

问题出在哪里？在于"扁平化"的规划方式。当你把一个复杂任务拆成 20 个步骤，平铺在一个列表里时，你很难一眼看出这些步骤之间的层次关系，也很难管理它们之间的依赖关系。就像你读一本 500 页的书，如果没有目录和章节划分，你根本不知道从哪里开始、哪些内容是相关的。

层次化规划的核心思想就是：**把任务分成多层来管理，每一层只关注自己层级的复杂度。** 就像一个公司的组织架构——CEO 管理 VP，VP 管理 Director，Director 管理 Engineer。每一层只需要关注自己直接下属的工作，不需要操心更底层的细节。

```
Level 0: 开发一个电商网站（CEO 的视角）
├── Level 1: 后端开发（VP 的视角）
│   ├── Level 2: 数据库设计（Director 的视角）
│   │   ├── Level 3: 用户表设计（Engineer 的任务）
│   │   ├── Level 3: 商品表设计
│   │   └── Level 3: 订单表设计
│   ├── Level 2: API 开发
│   │   ├── Level 3: 用户 API
│   │   ├── Level 3: 商品 API
│   │   └── Level 3: 订单 API
│   └── Level 2: 业务逻辑
├── Level 1: 前端开发
│   ├── Level 2: 页面组件
│   └── Level 2: 状态管理
└── Level 1: 测试部署
    ├── Level 2: 单元测试
    └── Level 2: 部署上线
```

层次化规划有四个核心优势：

1. **降低复杂度**：每一层只需要处理自己层级的问题，不会被底层细节淹没
2. **支持并行**：同一层级的子任务往往可以并行执行，提高效率
3. **便于调整**：修改一个子任务不会影响其他分支，改动范围可控
4. **进度追踪**：可以按层级追踪进度，一眼看出哪些模块完成了、哪些还在进行中

### 16.1.2 层次化任务的数据结构

```python
import json
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional


class TaskStatus(Enum):
    """任务状态枚举"""
    PENDING = "pending"           # 待执行
    IN_PROGRESS = "in_progress"   # 执行中
    COMPLETED = "completed"       # 已完成
    FAILED = "failed"             # 失败
    BLOCKED = "blocked"           # 被阻塞（等待依赖完成）
    SKIPPED = "skipped"           # 被跳过


@dataclass
class HierarchicalTask:
    """
    层次化任务
    
    每个任务可以包含子任务，形成树状结构。
    每个层级只关注自己这一层的任务，不需要操心更底层的细节。
    """
    id: str
    description: str
    level: int = 0
    status: TaskStatus = TaskStatus.PENDING
    parent_id: Optional[str] = None
    subtasks: list = field(default_factory=list)
    dependencies: list = field(default_factory=list)  # 依赖的其他任务 ID
    result: Optional[str] = None
    assigned_to: Optional[str] = None  # 分配给谁执行
    estimated_time: Optional[str] = None
    actual_time: Optional[float] = None
    
    def add_subtask(self, subtask: 'HierarchicalTask'):
        """添加子任务"""
        subtask.parent_id = self.id
        subtask.level = self.level + 1
        self.subtasks.append(subtask)
    
    def get_all_subtasks(self) -> list['HierarchicalTask']:
        """递归获取所有子任务（深度优先）"""
        result = []
        for subtask in self.subtasks:
            result.append(subtask)
            result.extend(subtask.get_all_subtasks())
        return result
    
    def is_complete(self) -> bool:
        """检查任务是否完成"""
        if not self.subtasks:
            # 叶子节点：检查自身状态
            return self.status == TaskStatus.COMPLETED
        # 非叶子节点：所有子任务都完成才算完成
        return all(sub.is_complete() for sub in self.subtasks)
    
    def get_progress(self) -> dict:
        """获取进度报告"""
        all_tasks = [self] + self.get_all_subtasks()
        total = len(all_tasks)
        completed = sum(1 for t in all_tasks if t.status == TaskStatus.COMPLETED)
        in_progress = sum(1 for t in all_tasks if t.status == TaskStatus.IN_PROGRESS)
        failed = sum(1 for t in all_tasks if t.status == TaskStatus.FAILED)
        
        return {
            "task_id": self.id,
            "description": self.description,
            "total": total,
            "completed": completed,
            "in_progress": in_progress,
            "failed": failed,
            "progress_percent": round(completed / total * 100, 1) if total > 0 else 0,
            "is_complete": self.is_complete(),
        }
    
    def to_dict(self) -> dict:
        """转为字典（便于序列化和调试）"""
        return {
            "id": self.id,
            "description": self.description,
            "level": self.level,
            "status": self.status.value,
            "parent_id": self.parent_id,
            "subtasks": [s.to_dict() for s in self.subtasks],
            "dependencies": self.dependencies,
            "result": self.result,
            "assigned_to": self.assigned_to,
        }
    
    def visualize(self, prefix: str = "", is_last: bool = True):
        """在控制台中可视化任务树"""
        status_icons = {
            TaskStatus.PENDING: "[ ]",
            TaskStatus.IN_PROGRESS: "[>]",
            TaskStatus.COMPLETED: "[v]",
            TaskStatus.FAILED: "[x]",
            TaskStatus.BLOCKED: "[!]",
            TaskStatus.SKIPPED: "[-]",
        }
        
        connector = "+-- " if self.subtasks else "`-- "
        icon = status_icons.get(self.status, "[?]")
        print(f"{prefix}{connector}{icon} {self.description}")
        
        new_prefix = prefix + ("|   " if self.subtasks else "    ")
        for i, subtask in enumerate(self.subtasks):
            subtask.visualize(new_prefix, i == len(self.subtasks) - 1)
```

### 16.1.3 层次化规划器的实现

```python
class HierarchicalPlanner:
    """
    层次化规划器
    
    工作流程：
    1. 接收顶层任务描述
    2. 将任务分解为 2-4 个子任务（第一层分解）
    3. 对每个子任务递归分解，直到达到最大深度或任务足够简单
    4. 返回完整的任务树
    """
    
    def __init__(self, llm_client, max_depth: int = 3,
                 min_subtasks: int = 2, max_subtasks: int = 4):
        """
        参数:
            llm_client: LLM 客户端
            max_depth: 最大分解深度
            min_subtasks: 每个任务最少分解为几个子任务
            max_subtasks: 每个任务最多分解为几个子任务
        """
        self.llm_client = llm_client
        self.max_depth = max_depth
        self.min_subtasks = min_subtasks
        self.max_subtasks = max_subtasks
        self.task_counter = 0  # 全局任务计数器
    
    def plan(self, task_description: str) -> HierarchicalTask:
        """生成层次化计划"""
        root = HierarchicalTask(
            id="root",
            description=task_description,
        )
        self.task_counter = 0
        
        # 递归分解
        self._decompose(root, depth=0)
        
        return root
    
    def _decompose(self, task: HierarchicalTask, depth: int):
        """
        递归分解任务
        
        在每一层，使用 LLM 将当前任务分解为若干子任务。
        如果达到最大深度或任务已经足够简单，则停止分解。
        """
        if depth >= self.max_depth:
            return
        
        # 判断是否需要继续分解
        if not self._should_decompose(task.description, depth):
            return
        
        # 使用 LLM 分解任务
        subtask_descriptions = self._llm_decompose(task.description)
        
        for desc in subtask_descriptions:
            self.task_counter += 1
            subtask = HierarchicalTask(
                id=f"{task.id}_sub_{self.task_counter}",
                description=desc,
            )
            task.add_subtask(subtask)
            
            # 递归分解子任务
            self._decompose(subtask, depth + 1)
    
    def _should_decompose(self, description: str, depth: int) -> bool:
        """判断一个任务是否需要继续分解"""
        # 如果已达到最大深度，不再分解
        if depth >= self.max_depth:
            return False
        
        # 让 LLM 判断任务是否足够简单
        prompt = f"""判断以下任务是否足够简单，可以直接执行（不需要进一步分解）。

任务：{description}

这个任务是否足够简单？（是/否）：
- 如果任务可以由一个人在一天内完成，回答"是"
- 如果任务需要多人协作或超过一天，回答"否""""

        response = self.llm_client.generate(prompt)
        return "否" in response
    
    def _llm_decompose(self, task_description: str) -> list[str]:
        """使用 LLM 将任务分解为子任务"""
        prompt = f"""请将以下任务分解为 {self.min_subtasks}-{self.max_subtasks} 个子任务。

任务：{task_description}

要求：
1. 子任务应该是可独立执行的
2. 子任务之间有清晰的边界
3. 所有子任务完成后，原任务也应该完成
4. 子任务的描述应该具体、明确

输出格式（每行一个子任务，不要编号）："""

        response = self.llm_client.generate(prompt)
        
        # 解析输出
        lines = [line.strip() for line in response.split("\n") if line.strip()]
        import re
        cleaned = [re.sub(r'^\d+[\.\)]\s*', '', line) for line in lines]
        cleaned = [line for line in cleaned if line and len(line) > 3]
        
        return cleaned[:self.max_subtasks]
    
    def print_plan(self, root: HierarchicalTask):
        """打印计划的树形结构"""
        print(f"\n{'='*60}")
        print(f"任务计划: {root.description}")
        print(f"{'='*60}")
        root.visualize()
        
        progress = root.get_progress()
        print(f"\n总计 {progress['total']} 个任务，"
              f"完成 {progress['completed']} 个，"
              f"进行中 {progress['in_progress']} 个，"
              f"进度 {progress['progress_percent']}%")


# 使用示例
def demo_hierarchical_planning():
    """演示层次化规划"""
    
    class MockLLM:
        def __init__(self):
            self.call_count = 0
        
        def generate(self, prompt):
            self.call_count += 1
            
            if "分解为" in prompt and "子任务" in prompt:
                if "电商网站" in prompt:
                    return "后端开发\n前端开发\n测试部署"
                elif "后端" in prompt:
                    return "数据库设计\nAPI 开发\n业务逻辑实现"
                elif "前端" in prompt:
                    return "页面组件开发\n状态管理\n响应式适配"
                else:
                    return "步骤 A\n步骤 B"
            elif "是否足够简单" in prompt:
                if self.call_count > 6:
                    return "是"
                return "否"
            return "否"
    
    llm = MockLLM()
    planner = HierarchicalPlanner(llm, max_depth=3)
    
    root = planner.plan("开发一个电商网站")
    planner.print_plan(root)


if __name__ == "__main__":
    demo_hierarchical_planning()
```

---

## 16.2 任务分解策略

### 16.2.1 功能分解

功能分解是最常见的任务分解方式——按系统的功能模块来拆分任务。比如一个电商系统可以分解为用户管理、商品管理、订单管理、支付管理等模块。

```python
class FunctionalDecomposer:
    """
    功能分解器
    
    将系统或项目按功能模块进行分解。
    适合有明确功能边界的项目。
    """
    
    def __init__(self, llm_client):
        self.llm_client = llm_client
    
    def decompose(self, system_description: str) -> dict:
        """
        将系统按功能模块分解
        
        返回包含模块列表、依赖关系和关键路径的分解结果
        """
        prompt = f"""请将以下系统按功能模块进行分解。

系统描述：{system_description}

请分析：
1. 系统包含哪些主要功能模块？
2. 模块之间的依赖关系是什么？
3. 哪些模块是关键路径（必须先完成）？

输出 JSON 格式：
{{
    "modules": [
        {{
            "name": "模块名称",
            "description": "模块描述",
            "responsibilities": ["该模块负责的功能"],
            "dependencies": ["依赖的其他模块"],
            "priority": "high/medium/low"
        }}
    ],
    "critical_path": ["关键路径上的模块名称"],
    "parallel_groups": [
        ["可以并行开发的模块组"]
    ]
}}

只输出 JSON："""

        response = self.llm_client.generate(prompt)
        
        try:
            return json.loads(response)
        except json.JSONDecodeError:
            return {"modules": [], "critical_path": [], "parallel_groups": []}
```

### 16.2.2 依赖分析

在任务分解之后，我们需要分析任务之间的依赖关系。依赖分析的目的有两个：一是确定执行顺序（有依赖关系的任务必须按顺序执行），二是识别可以并行执行的任务组。

```python
class DependencyAnalyzer:
    """
    依赖分析器
    
    分析任务之间的依赖关系，输出：
    - 依赖图（哪些任务依赖哪些任务）
    - 并行组（哪些任务可以同时执行）
    - 关键路径（决定整体完成时间的任务链）
    """
    
    def __init__(self, llm_client):
        self.llm_client = llm_client
    
    def analyze(self, tasks: list[dict]) -> dict:
        """
        分析任务依赖关系
        
        参数:
            tasks: 任务列表，每个任务包含 id 和 description
        """
        task_list = "\n".join(
            f"- [{t['id']}] {t['description']}"
            for t in tasks
        )
        
        prompt = f"""分析以下任务之间的依赖关系。

任务列表：
{task_list}

请分析：
1. 哪些任务必须在其他任务之前完成？
2. 哪些任务可以并行执行？
3. 哪些任务在关键路径上？

输出 JSON 格式：
{{
    "dependencies": [
        {{"from": "任务A的id", "to": "任务B的id", "reason": "依赖原因"}}
    ],
    "parallel_groups": [
        ["可以并行执行的任务id组"]
    ],
    "critical_path": ["关键路径上的任务id"],
    "execution_order": ["推荐的执行顺序"]
}}

只输出 JSON："""

        response = self.llm_client.generate(prompt)
        
        try:
            return json.loads(response)
        except json.JSONDecodeError:
            return {
                "dependencies": [],
                "parallel_groups": [],
                "critical_path": [],
                "execution_order": [t["id"] for t in tasks],
            }
    
    def topological_sort(self, tasks: list[dict],
                         dependencies: list[dict]) -> list[str]:
        """
        拓扑排序：根据依赖关系确定执行顺序
        
        如果任务 A 依赖任务 B，那 B 必须排在 A 前面
        """
        # 构建邻接表和入度表
        task_ids = {t["id"] for t in tasks}
        in_degree = {t["id"]: 0 for t in tasks}
        adj = {t["id"]: [] for t in tasks}
        
        for dep in dependencies:
            from_id = dep["from"]
            to_id = dep["to"]
            if from_id in task_ids and to_id in task_ids:
                adj[from_id].append(to_id)
                in_degree[to_id] += 1
        
        # BFS 拓扑排序
        queue = [tid for tid, deg in in_degree.items() if deg == 0]
        order = []
        
        while queue:
            current = queue.pop(0)
            order.append(current)
            
            for neighbor in adj[current]:
                in_degree[neighbor] -= 1
                if in_degree[neighbor] == 0:
                    queue.append(neighbor)
        
        return order
```

---

## 16.3 动态调整机制

### 16.3.1 为什么计划需要动态调整

在实际执行中，计划几乎不可能完全按照预期进行。可能遇到的情况包括：

1. **任务失败**：某个步骤执行失败，需要寻找替代方案
2. **需求变化**：用户中途改变了需求
3. **资源变化**：某些资源变得不可用（比如 API 服务宕机）
4. **新信息**：执行过程中发现了新的信息，需要调整计划

一个好的规划系统必须能处理这些变化，而不是固守原来的计划。

### 16.3.2 自适应规划器的实现

```python
import time


class AdaptivePlanner:
    """
    自适应规划器
    
    核心能力：
    1. 监控执行过程，检测异常情况
    2. 分析反馈信息，判断调整的级别
    3. 执行相应级别的调整（微调/大改/重做）
    """
    
    def __init__(self, llm_client):
        self.llm_client = llm_client
        self.execution_history: list[dict] = []
        self.adjustment_count = 0
    
    def analyze_feedback(self, feedback: dict) -> str:
        """
        分析反馈信息，确定需要什么级别的调整
        
        返回调整级别：
        - "none": 不需要调整
        - "minor": 小调整（修改部分步骤）
        - "major": 大调整（重新规划大部分步骤）
        - "restart": 完全重做
        """
        error_type = feedback.get("error_type", "")
        impact = feedback.get("impact", "low")
        
        if error_type == "resource_unavailable":
            return "minor"  # 资源问题，找替代方案即可
        elif error_type == "requirement_change":
            return "major"  # 需求变化，需要重新规划
        elif error_type == "fundamental_flaw":
            return "restart"  # 根本性错误，需要重做
        elif impact == "high":
            return "major"
        elif impact == "medium":
            return "minor"
        else:
            return "none"
    
    def adjust_plan(self, current_plan: dict, feedback: dict) -> dict:
        """
        根据反馈调整计划
        
        根据调整级别采取不同的策略：
        - minor: 保留大部分步骤，只修改受影响的部分
        - major: 保留已完成的部分，重新规划未完成的部分
        - restart: 完全重新规划
        """
        adjustment_type = self.analyze_feedback(feedback)
        self.adjustment_count += 1
        
        print(f"\n[调整] 类型: {adjustment_type}, 原因: {feedback.get('reason', '未知')}")
        
        if adjustment_type == "none":
            return current_plan
        elif adjustment_type == "minor":
            return self._minor_adjust(current_plan, feedback)
        elif adjustment_type == "major":
            return self._major_replan(current_plan, feedback)
        else:
            return self._full_restart(current_plan, feedback)
    
    def _minor_adjust(self, plan: dict, feedback: dict) -> dict:
        """小调整：修改部分步骤"""
        completed_steps = [s for s in plan.get("steps", [])
                          if s.get("status") == "completed"]
        remaining_steps = [s for s in plan.get("steps", [])
                          if s.get("status") != "completed"]
        
        prompt = f"""当前计划的部分步骤需要调整。

已完成的步骤：
{json.dumps(completed_steps, ensure_ascii=False, indent=2)}

待完成的步骤：
{json.dumps(remaining_steps, ensure_ascii=False, indent=2)}

需要调整的原因：
{json.dumps(feedback, ensure_ascii=False, indent=2)}

请保留已完成的步骤，只对待完成的步骤进行调整。
输出调整后的待完成步骤列表（JSON 数组）："""

        response = self.llm_client.generate(prompt)
        
        try:
            new_remaining = json.loads(response)
        except json.JSONDecodeError:
            new_remaining = remaining_steps
        
        return {
            "steps": completed_steps + new_remaining,
            "adjustment_type": "minor",
            "adjustment_reason": feedback.get("reason", ""),
        }
    
    def _major_replan(self, plan: dict, feedback: dict) -> dict:
        """大调整：重新规划未完成的部分"""
        completed_steps = [s for s in plan.get("steps", [])
                          if s.get("status") == "completed"]
        
        prompt = f"""计划需要进行较大调整。

原始任务目标：{plan.get('goal', '未知')}
已完成的步骤：
{json.dumps(completed_steps, ensure_ascii=False, indent=2)}

调整原因：
{json.dumps(feedback, ensure_ascii=False, indent=2)}

请根据已完成的工作和新的情况，重新制定剩余部分的计划。
输出新的步骤列表（JSON 数组）："""

        response = self.llm_client.generate(prompt)
        
        try:
            new_steps = json.loads(response)
        except json.JSONDecodeError:
            new_steps = []
        
        return {
            "goal": plan.get("goal", ""),
            "steps": completed_steps + new_steps,
            "adjustment_type": "major",
            "adjustment_reason": feedback.get("reason", ""),
        }
    
    def _full_restart(self, plan: dict, feedback: dict) -> dict:
        """完全重做"""
        prompt = f"""之前的计划因为根本性问题需要完全重做。

原始任务：{plan.get('goal', '未知')}
失败原因：
{json.dumps(feedback, ensure_ascii=False, indent=2)}

请重新制定完整的计划。
输出新的计划（JSON 格式，包含 goal 和 steps）："""

        response = self.llm_client.generate(prompt)
        
        try:
            new_plan = json.loads(response)
        except json.JSONDecodeError:
            new_plan = {"goal": plan.get("goal", ""), "steps": []}
        
        new_plan["adjustment_type"] = "restart"
        new_plan["adjustment_reason"] = feedback.get("reason", "")
        return new_plan
```

### 16.3.3 完整的动态调整示例

```python
def demo_adaptive_planning():
    """演示自适应规划"""
    
    class MockLLM:
        def __init__(self):
            self.call_count = 0
        
        def generate(self, prompt):
            self.call_count += 1
            
            if "调整后的待完成步骤" in prompt or "新的步骤列表" in prompt:
                return json.dumps([
                    {"id": "adjusted_1", "description": "调整后的步骤1", "status": "pending"},
                    {"id": "adjusted_2", "description": "调整后的步骤2", "status": "pending"},
                ])
            elif "重新制定完整的计划" in prompt:
                return json.dumps({
                    "goal": "重新规划的任务",
                    "steps": [
                        {"id": "restart_1", "description": "重做步骤1", "status": "pending"},
                    ]
                })
            return "[]"
    
    llm = MockLLM()
    planner = AdaptivePlanner(llm)
    
    # 模拟执行计划
    current_plan = {
        "goal": "开发一个 Web 应用",
        "steps": [
            {"id": 1, "description": "设计数据库", "status": "completed"},
            {"id": 2, "description": "开发后端 API", "status": "in_progress"},
            {"id": 3, "description": "开发前端页面", "status": "pending"},
        ]
    }
    
    # 模拟反馈：API 服务不可用
    feedback = {
        "error_type": "resource_unavailable",
        "reason": "第三方 API 服务宕机，无法调用",
        "impact": "medium",
    }
    
    print("原始计划:")
    for s in current_plan["steps"]:
        print(f"  [{s['status']}] {s['description']}")
    
    new_plan = planner.adjust_plan(current_plan, feedback)
    
    print("\n调整后的计划:")
    for s in new_plan.get("steps", []):
        print(f"  [{s.get('status', 'pending')}] {s['description']}")


if __name__ == "__main__":
    demo_adaptive_planning()
```

---

## 16.4 资源管理与约束处理

### 16.4.1 为什么需要考虑资源约束

一个好的计划不仅要考虑"做什么"，还要考虑"有多少资源来做"。在 Agent 系统中，常见的资源约束包括：

- **时间约束**：整个任务必须在某个截止时间前完成
- **Token 预算**：LLM 调用有 Token 消耗限制
- **API 调用次数**：某些 API 有调用频率限制
- **并行度限制**：同时只能执行 N 个任务

如果不考虑这些约束，计划很可能在执行时"卡住"。

### 16.4.2 带资源约束的规划器

```python
@dataclass
class ResourceConstraint:
    """资源约束"""
    max_time_seconds: float = 300.0       # 最大执行时间
    max_llm_calls: int = 20               # 最大 LLM 调用次数
    max_tokens: int = 100000              # 最大 Token 消耗
    max_parallel_tasks: int = 3           # 最大并行任务数
    
    def to_prompt(self) -> str:
        """转换为 Prompt 中的约束描述"""
        return f"""资源约束：
- 最大执行时间: {self.max_time_seconds} 秒
- 最大 LLM 调用次数: {self.max_llm_calls} 次
- 最大 Token 消耗: {self.max_tokens}
- 最大并行任务数: {self.max_parallel_tasks}"""


@dataclass
class ResourceTracker:
    """资源使用追踪器"""
    time_used: float = 0.0
    llm_calls: int = 0
    tokens_used: int = 0
    parallel_tasks: int = 0
    
    def check_constraint(self, constraint: ResourceConstraint) -> tuple[bool, str]:
        """检查是否超出约束"""
        if self.time_used >= constraint.max_time_seconds:
            return False, "超出时间限制"
        if self.llm_calls >= constraint.max_llm_calls:
            return False, "超出 LLM 调用次数限制"
        if self.tokens_used >= constraint.max_tokens:
            return False, "超出 Token 消耗限制"
        return True, "资源充足"
    
    def estimate_step_cost(self, step: dict) -> dict:
        """估算一个步骤的资源消耗"""
        # 基于步骤描述的长度和复杂度来估算
        desc_length = len(step.get("description", ""))
        
        estimated_time = desc_length * 0.1  # 粗略估算
        estimated_llm_calls = 2  # 通常每个步骤至少 2 次 LLM 调用
        estimated_tokens = desc_length * 10  # 粗略估算
        
        return {
            "time": estimated_time,
            "llm_calls": estimated_llm_calls,
            "tokens": estimated_tokens,
        }


class ResourceAwarePlanner:
    """
    资源感知规划器
    
    在规划时考虑资源约束：
    1. 估算每步的资源消耗
    2. 检查总消耗是否超出约束
    3. 必要时合并或简化步骤以节省资源
    """
    
    def __init__(self, llm_client, constraint: ResourceConstraint = None):
        self.llm_client = llm_client
        self.constraint = constraint or ResourceConstraint()
        self.tracker = ResourceTracker()
    
    def plan(self, task: str) -> dict:
        """生成资源感知的计划"""
        constraint_text = self.constraint.to_prompt()
        
        prompt = f"""请为以下任务制定执行计划。

任务：{task}

{constraint_text}

请在资源约束内制定计划。估算每个步骤的资源消耗，确保总消耗不超出约束。

输出 JSON 格式：
{{
    "goal": "任务目标",
    "steps": [
        {{
            "id": 1,
            "description": "步骤描述",
            "estimated_time": 30,
            "estimated_llm_calls": 2,
            "estimated_tokens": 5000
        }}
    ],
    "total_estimated_time": 120,
    "total_estimated_llm_calls": 8,
    "total_estimated_tokens": 20000
}}

只输出 JSON："""

        response = self.llm_client.generate(prompt)
        
        try:
            plan = json.loads(response)
            # 验证是否超出约束
            if plan.get("total_estimated_time", 0) > self.constraint.max_time_seconds:
                plan["warning"] = "计划可能超出时间限制"
            return plan
        except json.JSONDecodeError:
            return {"goal": task, "steps": [], "error": "计划生成失败"}
```

---

## 16.5 多 Agent 协作规划

### 16.5.1 为什么需要多 Agent 协作规划

在某些场景下，单个 Agent 的规划能力有限。比如一个大型软件项目，可能需要前端专家、后端专家、数据库专家分别从各自的角度来规划，然后把三个专家的方案整合成一个统一的计划。

多 Agent 协作规划的基本流程是：

1. **独立规划**：每个 Agent 根据自己的专业领域独立制定计划
2. **计划合并**：将多个计划合并为一个统一的计划
3. **冲突解决**：解决不同计划之间的矛盾和冲突
4. **最终协调**：确保合并后的计划是可行的、一致的

### 16.5.2 协作规划器的实现

```python
class CollaborativePlanner:
    """
    多 Agent 协作规划器
    
    模拟多个专家协同制定计划的过程：
    1. 每个专家（Agent）独立制定自己领域的计划
    2. 一个协调者（Coordinating Agent）合并所有计划
    3. 协调者解决冲突并输出最终计划
    """
    
    def __init__(self, agents: list[dict]):
        """
        参数:
            agents: Agent 列表，每个 Agent 是一个字典：
                    {"id": "agent_1", "role": "后端专家", "expertise": "..."}
        """
        self.agents = agents
        self.llm_client = None  # 会在外部设置
    
    def collaborative_plan(self, task: str) -> dict:
        """
        协作生成计划
        
        三阶段流程：独立规划 → 合并 → 冲突解决
        """
        # 阶段 1：每个 Agent 独立规划
        print("阶段 1: 各专家独立规划...")
        individual_plans = []
        for agent in self.agents:
            plan = self._agent_plan(agent, task)
            individual_plans.append({
                "agent_id": agent["id"],
                "agent_role": agent["role"],
                "plan": plan,
            })
            print(f"  {agent['role']} 完成规划，包含 {len(plan.get('steps', []))} 个步骤")
        
        # 阶段 2：合并计划
        print("\n阶段 2: 合并计划...")
        merged_plan = self._merge_plans(individual_plans, task)
        print(f"  合并后共 {len(merged_plan.get('steps', []))} 个步骤")
        
        # 阶段 3：解决冲突
        print("\n阶段 3: 解决冲突...")
        final_plan = self._resolve_conflicts(merged_plan, individual_plans)
        
        return final_plan
    
    def _agent_plan(self, agent: dict, task: str) -> dict:
        """让单个 Agent 制定计划"""
        prompt = f"""你是{agent['role']}，专业领域是{agent.get('expertise', agent['role'])}。

请为以下任务制定你专业领域内的执行计划。

任务：{task}

请从你的专业角度出发，制定详细的步骤。

输出 JSON 格式：
{{
    "steps": [
        {{
            "id": 1,
            "description": "步骤描述",
            "dependencies": [],
            "estimated_time": "预计耗时"
        }}
    ]
}}

只输出 JSON："""

        response = self.llm_client.generate(prompt)
        
        try:
            return json.loads(response)
        except json.JSONDecodeError:
            return {"steps": []}
    
    def _merge_plans(self, plans: list[dict], task: str) -> dict:
        """合并多个计划"""
        plans_text = json.dumps(plans, ensure_ascii=False, indent=2)
        
        prompt = f"""请合并以下多个专家的计划为一个统一的计划。

任务：{task}

各专家的计划：
{plans_text}

合并要求：
1. 保留所有专家的贡献
2. 去除重复的步骤
3. 添加必要的协调步骤（如接口对接、联调测试）
4. 保持逻辑一致性

输出合并后的 JSON 格式计划："""

        response = self.llm_client.generate(prompt)
        
        try:
            return json.loads(response)
        except json.JSONDecodeError:
            return {"steps": [], "error": "合并失败"}
    
    def _resolve_conflicts(self, merged_plan: dict,
                           individual_plans: list[dict]) -> dict:
        """解决计划中的冲突"""
        prompt = f"""检查以下合并后的计划，解决其中的冲突和不一致。

合并后的计划：
{json.dumps(merged_plan, ensure_ascii=False, indent=2)}

各专家的原始计划：
{json.dumps(individual_plans, ensure_ascii=False, indent=2)}

请检查并解决：
1. 步骤之间的依赖冲突
2. 时间安排的冲突
3. 资源分配的冲突

输出修复后的 JSON 格式计划："""

        response = self.llm_client.generate(prompt)
        
        try:
            return json.loads(response)
        except json.JSONDecodeError:
            return merged_plan
```

---

## 16.6 规划系统的评估

### 16.6.1 评估指标

如何衡量一个规划系统的好坏？我们通常关注以下几个维度：

1. **成功率**：计划执行成功的比例
2. **效率**：完成任务所需的平均时间
3. **计划质量**：计划的完整性、可行性、可调整性
4. **适应性**：面对变化时调整计划的能力

```python
class PlannerEvaluator:
    """
    规划器评估器
    
    通过在一组测试任务上运行规划器，收集各项指标，
    生成评估报告。
    """
    
    def __init__(self):
        self.results: list[dict] = []
    
    def evaluate(self, planner, tasks: list[dict],
                 executor=None) -> dict:
        """
        评估规划器
        
        参数:
            planner: 要评估的规划器
            tasks: 测试任务列表
            executor: 执行器（可选，不提供则只评估计划生成）
        """
        self.results = []
        
        for task in tasks:
            result = self._evaluate_single(planner, task, executor)
            self.results.append(result)
        
        # 汇总指标
        metrics = {
            "total_tasks": len(tasks),
            "success_rate": self._calc_success_rate(),
            "avg_planning_time": self._calc_avg_time("planning_time"),
            "avg_execution_time": self._calc_avg_time("execution_time"),
            "avg_plan_steps": self._calc_avg_steps(),
            "plan_quality_scores": self._calc_quality_scores(),
        }
        
        return metrics
    
    def _evaluate_single(self, planner, task: dict,
                         executor=None) -> dict:
        """评估单个任务"""
        import time
        
        # 计时规划阶段
        start = time.time()
        plan = planner.plan(task["description"]) if hasattr(planner, 'plan') else {}
        planning_time = time.time() - start
        
        # 计时执行阶段
        execution_time = 0
        success = True
        if executor:
            start = time.time()
            try:
                result = executor.execute(plan)
                success = result.get("success", False)
            except Exception as e:
                success = False
            execution_time = time.time() - start
        
        return {
            "task": task["description"],
            "success": success,
            "planning_time": planning_time,
            "execution_time": execution_time,
            "plan_steps": len(plan.get("steps", [])) if isinstance(plan, dict) else 0,
        }
    
    def _calc_success_rate(self) -> float:
        if not self.results:
            return 0.0
        return sum(1 for r in self.results if r["success"]) / len(self.results)
    
    def _calc_avg_time(self, key: str) -> float:
        if not self.results:
            return 0.0
        return sum(r[key] for r in self.results) / len(self.results)
    
    def _calc_avg_steps(self) -> float:
        if not self.results:
            return 0.0
        return sum(r["plan_steps"] for r in self.results) / len(self.results)
    
    def _calc_quality_scores(self) -> dict:
        """计算计划质量分数"""
        return {
            "avg_planning_time": self._calc_avg_time("planning_time"),
            "planning_time_std": self._calc_std("planning_time"),
        }
    
    def _calc_std(self, key: str) -> float:
        """计算标准差"""
        values = [r[key] for r in self.results]
        if not values:
            return 0.0
        mean = sum(values) / len(values)
        variance = sum((v - mean) ** 2 for v in values) / len(values)
        return variance ** 0.5
    
    def print_report(self):
        """打印评估报告"""
        metrics = self.evaluate(None, []) if not self.results else {
            "total_tasks": len(self.results),
            "success_rate": self._calc_success_rate(),
            "avg_planning_time": self._calc_avg_time("planning_time"),
            "avg_execution_time": self._calc_avg_time("execution_time"),
            "avg_plan_steps": self._calc_avg_steps(),
        }
        
        print("\n" + "="*50)
        print("规划器评估报告")
        print("="*50)
        print(f"测试任务数: {metrics['total_tasks']}")
        print(f"成功率: {metrics['success_rate']:.1%}")
        print(f"平均规划时间: {metrics['avg_planning_time']:.2f}秒")
        print(f"平均执行时间: {metrics['avg_execution_time']:.2f}秒")
        print(f"平均步骤数: {metrics['avg_plan_steps']:.1f}")
```

---

## 16.7 常见坑

### 16.7.1 过度分解

**问题描述：** 将任务分解得过于细碎，导致管理成本超过收益。比如把"写一个函数"分解为"定义函数名"、"写参数列表"、"写函数体"、"写返回值"四个子任务，这种分解粒度太细了，管理它们反而比直接写函数更费劲。

**解决方案：** 设置合理的分解粒度。一个经验法则是：分解到"一个人可以在一到两天内完成"的粒度即可。太粗的分解会导致单步复杂度高，太细的分解会导致管理成本高。

### 16.7.2 忽略并行性

**问题描述：** 所有步骤都串行执行，明明可以同时做的任务也排队等着，效率低下。

**解决方案：** 在规划时分析任务之间的依赖关系，识别可以并行执行的任务组，然后同时调度执行。在多 Agent 场景下，这一点尤为重要。

```python
def find_parallel_groups(tasks: list[dict], dependencies: list[dict]) -> list[list[str]]:
    """找到可以并行执行的任务组"""
    # 构建依赖图
    dep_map = {}
    for dep in dependencies:
        dep_map.setdefault(dep["to"], []).append(dep["from"])
    
    # 找到没有未完成依赖的任务（可以立即执行）
    completed = set()
    groups = []
    
    while len(completed) < len(tasks):
        group = []
        for task in tasks:
            if task["id"] in completed:
                continue
            # 检查所有依赖是否已完成
            deps = dep_map.get(task["id"], [])
            if all(d in completed for d in deps):
                group.append(task["id"])
        
        if not group:
            break  # 有循环依赖
        
        groups.append(group)
        completed.update(group)
    
    return groups
```

### 16.7.3 约束处理不当

**问题描述：** 规划时忽略资源约束（时间、Token、API 调用次数等），执行时才发现不可行。

**解决方案：** 在规划阶段就明确所有约束条件，并在计划中估算每步的资源消耗。如果总消耗超出约束，自动简化或合并步骤。

### 16.7.4 缺乏回退机制

**问题描述：** 计划只考虑了成功路径，没有考虑失败时怎么办。

**解决方案：** 在计划中为每个步骤标注回退策略。实现 Plan-Execute 循环，在失败时自动调整。

---

## 16.8 练习题

### 练习 1：实现一个支持 3 层分解的层次化规划器

要求：
- 接收顶层任务描述
- 递归分解为 3 层子任务
- 支持任务树的可视化输出
- 支持进度追踪

测试任务：
1. "开发一个博客系统"
2. "组织一次公司年会"
3. "编写一本技术教程"

### 练习 2：实现一个动态调整系统

要求：
- 监控任务执行状态
- 检测失败和异常
- 根据异常类型选择调整策略
- 记录调整历史

测试场景：
1. 模拟某个步骤失败
2. 模拟需求变更
3. 模拟资源不足

### 练习 3：实现依赖分析和拓扑排序

要求：
- 分析一组任务的依赖关系
- 识别可以并行执行的任务组
- 通过拓扑排序确定执行顺序
- 检测循环依赖

### 练习 4：实现多 Agent 协作规划

要求：
- 至少 3 个不同角色的 Agent
- 独立规划 → 合并 → 冲突解决的三阶段流程
- 合并时去除重复步骤
- 输出最终的统一计划

### 练习 5：实现规划器评估系统

要求：
- 准备至少 5 个测试任务
- 评估指标包括：成功率、规划时间、计划质量
- 对比不同规划策略的表现
- 生成评估报告

---

## 16.9 实战任务

### 任务：构建自适应项目规划系统

**目标：** 构建一个能自动规划软件开发项目的系统，支持需求分析、任务分解、动态调整和进度追踪。

**要求：**

1. 支持自然语言需求输入和分析
2. 自动生成层次化项目计划（至少 3 层）
3. 支持计划的动态调整（小调整、大调整、重做）
4. 提供项目进度追踪和可视化
5. 支持资源约束检查

**参考实现框架：**

```python
class ProjectPlanner:
    """自适应项目规划系统"""
    
    def __init__(self, llm_client):
        self.llm_client = llm_client
        self.hierarchical_planner = HierarchicalPlanner(llm_client)
        self.adaptive_planner = AdaptivePlanner(llm_client)
        self.current_plan = None
    
    def analyze_requirements(self, requirements: str) -> dict:
        """分析需求"""
        prompt = f"""分析以下项目需求，提取关键信息。

需求描述：{requirements}

请分析：
1. 项目的核心功能
2. 技术栈要求
3. 时间约束
4. 关键风险

输出 JSON 格式分析报告："""

        response = self.llm_client.generate(prompt)
        try:
            return json.loads(response)
        except json.JSONDecodeError:
            return {"core_features": [], "tech_stack": [], "risks": []}
    
    def create_project_plan(self, requirements: str) -> dict:
        """创建项目计划"""
        # 分析需求
        analysis = self.analyze_requirements(requirements)
        
        # 层次化分解
        root_task = self.hierarchical_planner.plan(requirements)
        
        self.current_plan = {
            "requirements": requirements,
            "analysis": analysis,
            "task_tree": root_task,
        }
        
        return self.current_plan
    
    def handle_issue(self, issue: dict) -> dict:
        """处理执行中的问题"""
        return self.adaptive_planner.adjust_plan(
            self._plan_to_dict(self.current_plan),
            issue,
        )
    
    def get_progress(self) -> dict:
        """获取项目进度"""
        if self.current_plan and self.current_plan.get("task_tree"):
            return self.current_plan["task_tree"].get_progress()
        return {"progress_percent": 0}
    
    def _plan_to_dict(self, plan: dict) -> dict:
        """将计划转换为字典格式"""
        result = {"goal": plan.get("requirements", ""), "steps": []}
        if plan.get("task_tree"):
            all_tasks = plan["task_tree"].get_all_subtasks()
            result["steps"] = [
                {"id": t.id, "description": t.description, "status": t.status.value}
                for t in all_tasks
            ]
        return result
```

---

## 16.10 本章小结

- **层次化规划**通过多层分解降低复杂度。每一层只关注自己层级的问题，就像公司的组织架构一样——CEO 不需要操心每个工程师的具体代码。层次化规划的核心优势是降低复杂度、支持并行、便于调整和进度追踪。

- **任务分解策略**包括功能分解（按模块拆分）、数据流分解（按数据处理流程拆分）、依赖分析（识别任务间的前后关系）。选择哪种策略取决于任务的性质。在分解时要注意粒度——太粗管理不了，太细管理成本高。

- **动态调整机制**是规划系统应对不确定性的关键。通过监控执行过程、分析反馈信息、按级别调整（微调/大改/重做），让计划能在执行过程中自适应变化。记住，计划不是一成不变的，它需要随着实际情况不断调整。

- **资源管理和约束处理**确保计划的可行性。在规划阶段就要明确时间、Token、API 调用次数等约束，并估算每步的资源消耗。如果总消耗超出约束，需要简化或合并步骤。

- **多 Agent 协作规划**通过"独立规划 → 合并 → 冲突解决"的三阶段流程，让多个专家协同制定计划。它适合大型复杂项目，能利用各领域的专业知识制定更全面的计划。

- **规划系统评估**通过成功率、效率、计划质量等指标来衡量规划系统的好坏，是持续改进的基础。

