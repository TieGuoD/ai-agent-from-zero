# 第 15 章：Planning —— Agent 的规划能力

---

## 学习目标

完成本章学习后，你将能够：

1. 深入理解为什么 Agent 需要规划能力，以及没有规划能力时 Agent 会遇到什么问题
2. 掌握 Chain-of-Thought（思维链）规划策略的原理和实现，理解它如何帮助 Agent "想清楚再动手"
3. 掌握 Tree-of-Thought（思维树）规划策略，理解它如何让 Agent 探索多条路径并选择最优方案
4. 掌握 Goal-of-Action（目标导向）规划策略，理解它如何从终点反向推导起点
5. 能够实现 Plan-Execute 循环，让规划和执行交替进行、互相反馈
6. 了解不同规划策略的适用场景，学会为不同任务选择合适的规划方式
7. 理解规划过程中的常见陷阱和应对策略

## 核心问题

1. 为什么 Agent 需要规划能力？没有规划会怎样？
2. 不同的规划策略（CoT、ToT、GoA）有什么区别？各自的优缺点是什么？
3. 如何在规划和执行之间取得平衡——既不过度规划，也不走一步看一步？

---

## 15.1 规划能力的本质

### 15.1.1 没有规划的 Agent 会怎样

让我们先来看一个没有规划能力的 Agent 面对复杂任务时会发生什么。

想象你让 Agent "帮我搭建一个个人博客网站"。没有规划的 Agent 会怎么做呢？它大概率会直接开始写代码——先创建一个 index.html，然后写着写着发现需要一个后端，于是开始写后端代码，写着写着发现需要数据库，于是开始设计数据库，设计到一半发现前端的样式还没写……

这种"走一步看一步"的方式，在简单任务上也许能凑合，但面对复杂任务时一定会出问题：

1. **方向迷失**：写着写着忘记了最初的目标是什么
2. **重复劳动**：做了一些后来发现没必要的工作
3. **遗漏关键步骤**：忘了某些必要的环节（比如测试、部署）
4. **资源浪费**：在不重要的细节上花太多时间

```python
# 没有规划的 Agent：走一步看一步
def naive_agent(task: str, llm_client) -> str:
    """
    最简单的 Agent：直接把任务丢给 LLM
    适合简单任务，但复杂任务容易出问题
    """
    prompt = f"请完成以下任务：{task}"
    return llm_client.generate(prompt)

# 有规划的 Agent：先想清楚再动手
def planned_agent(task: str, llm_client) -> str:
    """
    先制定计划，再按计划执行
    适合复杂任务
    """
    # 第一步：制定计划
    plan = create_plan(task, llm_client)
    
    # 第二步：按计划逐步执行
    results = []
    for step in plan["steps"]:
        result = execute_step(step, results, llm_client)
        results.append(result)
    
    # 第三步：综合结果
    return synthesize(results, llm_client)
```

### 15.1.2 规划的本质是什么

规划的本质可以用一句话概括：**将复杂任务分解为可执行的步骤序列，并为每个步骤分配资源和时间。**

一个完整的规划过程包含五个核心环节：

1. **目标分析**：理解任务的最终目标是什么，成功的标准是什么
2. **子目标分解**：把大目标拆成若干个小目标，每个小目标都是可独立完成的
3. **步骤排序**：确定步骤之间的先后顺序和依赖关系
4. **资源分配**：确定每个步骤需要什么工具、什么信息、多少时间
5. **执行监控**：在执行过程中检查进度，必要时调整计划

好的规划应该具备四个特征：

- **完整性**：覆盖任务的所有方面，没有遗漏
- **可行性**：每个步骤都是可以实际执行的
- **可调整**：能根据执行反馈修改计划（计划赶不上变化是常态）
- **效率**：避免不必要的步骤和重复工作

### 15.1.3 规划策略的全景图

在 Agent 领域，常见的规划策略可以按复杂度分为三个层次：

1. **Chain-of-Thought（思维链）**：让 LLM 按步骤展示推理过程，是最基础也最常用的规划策略
2. **Tree-of-Thought（思维树）**：让 LLM 探索多条可能的路径，评估每条路径的价值，选择最优方案
3. **Goal-of-Action（目标导向）**：从最终目标出发，反向推导需要的步骤

这三种策略各有优劣，适用于不同的场景。接下来我们逐一深入讲解。

---

## 15.2 Chain-of-Thought (CoT) 思维链

### 15.2.1 什么是 CoT

Chain-of-Thought（思维链）是一种让 LLM "展示推理过程"的技术。它的核心思想很简单：与其让 LLM 直接给出答案，不如让它一步一步地想，把中间的推理过程写出来。

为什么这样做有效？因为 LLM 在生成每个 token 时，都会受到前面所有 token 的影响。如果它先把推理过程写出来了，那后续生成的答案就会受到这些推理过程的"引导"，从而更准确、更合理。

这就好比你做数学题时在草稿纸上写解题步骤——虽然最终只需要写答案，但在草稿纸上一步步推导能大大降低出错的概率。

### 15.2.2 CoT 规划器的实现

```python
import json
import time


class CoTPlanner:
    """
    基于 Chain-of-Thought 的规划器
    
    工作流程：
    1. 接收任务描述
    2. 让 LLM 分析任务、制定计划
    3. 按步骤逐步执行
    4. 每步执行后检查结果，必要时调整后续计划
    """
    
    def __init__(self, llm_client):
        self.llm_client = llm_client
    
    def plan(self, task: str) -> dict:
        """
        为任务生成执行计划
        
        让 LLM 先分析任务的目标、约束和关键步骤，
        输出结构化的 JSON 格式计划。
        """
        prompt = f"""你是一个任务规划专家。请为以下任务制定详细的执行计划。

任务：{task}

请仔细思考以下问题：
1. 这个任务的最终目标是什么？
2. 需要哪些前置条件？
3. 可以分成几个子任务？
4. 每个子任务之间有什么依赖关系？
5. 有哪些潜在的风险？

请按以下 JSON 格式输出计划：
{{
    "goal": "任务的最终目标",
    "subtasks": [
        {{
            "id": 1,
            "description": "子任务描述",
            "dependencies": [],
            "estimated_time": "预计耗时",
            "tools_needed": ["需要的工具"],
            "success_criteria": "成功的标准"
        }}
    ],
    "risks": [
        {{
            "risk": "风险描述",
            "mitigation": "应对措施"
        }}
    ],
    "success_criteria": ["整体任务的成功标准"]
}}

只输出 JSON，不要其他内容："""

        response = self.llm_client.generate(prompt)
        
        try:
            # 尝试解析 JSON
            plan = json.loads(response)
            return plan
        except json.JSONDecodeError:
            # 如果解析失败，尝试从文本中提取 JSON
            return self._extract_json_from_text(response)
    
    def execute_plan(self, plan: dict) -> dict:
        """
        执行计划
        
        按照计划的顺序逐步执行每个子任务，
        并记录每步的结果。
        """
        execution_log = []
        context = {"plan": plan, "results": []}
        
        for step in plan.get("subtasks", []):
            print(f"\n--- 执行步骤 {step['id']}: {step['description']} ---")
            
            # 检查依赖是否满足
            dependencies = step.get("dependencies", [])
            if dependencies:
                unmet = [d for d in dependencies if not self._is_step_done(d, execution_log)]
                if unmet:
                    print(f"  依赖未满足: {unmet}，跳过此步骤")
                    continue
            
            # 执行步骤
            start_time = time.time()
            result = self._execute_step(step, context)
            duration = time.time() - start_time
            
            # 记录结果
            log_entry = {
                "step_id": step["id"],
                "description": step["description"],
                "result": result,
                "duration": round(duration, 2),
                "success": result.get("success", False),
            }
            execution_log.append(log_entry)
            context["results"].append(result)
            
            status = "成功" if result.get("success") else "失败"
            print(f"  结果: {status} ({duration:.1f}秒)")
        
        return {
            "plan": plan,
            "execution_log": execution_log,
            "success": all(r.get("success", False) for r in execution_log),
        }
    
    def replan(self, original_task: str, execution_log: list[dict]) -> dict:
        """
        根据执行结果重新规划
        
        当某些步骤失败时，需要根据已有结果调整计划
        """
        results_summary = "\n".join(
            f"步骤 {r['step_id']}: {r['description']} -> {'成功' if r['success'] else '失败'}"
            for r in execution_log
        )
        
        prompt = f"""原始任务：{original_task}

已执行的步骤和结果：
{results_summary}

根据以上结果，请重新制定剩余步骤的计划。已经成功的步骤不需要重复。
输出新的 JSON 格式计划（只包含未完成的步骤）："""

        response = self.llm_client.generate(prompt)
        
        try:
            return json.loads(response)
        except json.JSONDecodeError:
            return {"subtasks": []}
    
    def _execute_step(self, step: dict, context: dict) -> dict:
        """执行单个步骤"""
        previous_results = json.dumps(context.get("results", []), ensure_ascii=False)
        
        prompt = f"""请执行以下步骤：

步骤描述：{step['description']}
成功的标准：{step.get('success_criteria', '完成即可')}
之前步骤的结果：{previous_results}

请输出执行结果（JSON 格式）：
{{
    "output": "执行结果的具体内容",
    "success": true/false,
    "notes": "备注信息"
}}"""

        response = self.llm_client.generate(prompt)
        
        try:
            return json.loads(response)
        except json.JSONDecodeError:
            return {"output": response, "success": True, "notes": "返回格式非JSON"}
    
    def _is_step_done(self, step_id: int, log: list[dict]) -> bool:
        """检查某个步骤是否已完成"""
        return any(r["step_id"] == step_id and r["success"] for r in log)
    
    def _extract_json_from_text(self, text: str) -> dict:
        """从文本中提取 JSON"""
        import re
        match = re.search(r'\{.*\}', text, re.DOTALL)
        if match:
            try:
                return json.loads(match.group())
            except json.JSONDecodeError:
                pass
        return {"subtasks": [], "error": "计划解析失败"}
```

### 15.2.3 CoT 的优势与局限

CoT 最大的优势是简单直观。它不需要复杂的算法，只需要在 Prompt 中引导 LLM 按步骤思考就能显著提升规划质量。这使得它成为大多数 Agent 系统的默认选择。

但 CoT 也有明显的局限。首先，它是一条"单链"——LLM 只探索一条推理路径，如果这条路径走偏了，后面的步骤也会跟着错。其次，它没有"回溯"能力——一旦某个步骤执行失败，它只能简单地重新规划，而不是回到之前的某个分叉点尝试另一条路。

这些局限催生了 Tree-of-Thought 策略。

---

## 15.3 Tree-of-Thought (ToT) 思维树

### 15.3.1 什么是 ToT

如果说 CoT 是一条"思维链"，那 ToT 就是一棵"思维树"。它让 LLM 在每一步都生成多个可能的"下一步"，评估每个选项的价值，然后选择最好的继续深入。

这就好比下棋时，高手不会只想一种走法，而是会考虑多种可能的走法，评估每种走法的好坏，然后选择最优的那一步。

```
                    任务
                 /   |   \
              方案A  方案B  方案C
              / \    |    / \
           A1  A2   B1  C1  C2
           |       / \       |
          A11    B11 B12    C21
```

ToT 的关键在于"探索"和"评估"——在每个节点生成多个选项，然后用某种标准（通常是 LLM 自身的评估能力）来打分，只保留最好的几个选项继续探索。

### 15.3.2 ToT 规划器的实现

```python
from dataclasses import dataclass, field
import time


@dataclass
class ThoughtNode:
    """
    思维树中的一个节点
    
    每个节点代表一个"想法"或"方案的一部分"，
    包含内容、评估分数和子节点。
    """
    content: str
    score: float = 0.0
    depth: int = 0
    children: list = field(default_factory=list)
    is_leaf: bool = False
    
    def to_dict(self) -> dict:
        """转为字典，便于序列化"""
        return {
            "content": self.content,
            "score": self.score,
            "depth": self.depth,
            "is_leaf": self.is_leaf,
            "children": [c.to_dict() for c in self.children],
        }


class TreeOfThoughtsPlanner:
    """
    基于思维树的规划器
    
    核心思路：
    1. 在每一步生成多个可能的"下一步"
    2. 评估每个选项的价值
    3. 只保留最好的几个选项继续探索
    4. 到达最大深度后，选择得分最高的路径
    """
    
    def __init__(self, llm_client, max_depth: int = 3,
                 max_branches: int = 3, eval_threshold: float = 0.3):
        """
        参数:
            llm_client: LLM 客户端
            max_depth: 思维树的最大深度（层数）
            max_branches: 每个节点最多生成几个子节点
            eval_threshold: 评估分数的最低阈值，低于此分数的节点不会被探索
        """
        self.llm_client = llm_client
        self.max_depth = max_depth
        self.max_branches = max_branches
        self.eval_threshold = eval_threshold
        self.node_count = 0  # 用于统计探索了多少个节点
    
    def solve(self, task: str) -> dict:
        """
        使用思维树解决任务
        
        返回最佳路径和对应的方案
        """
        # 创建根节点
        root = ThoughtNode(content=task, depth=0)
        self.node_count = 0
        
        # 递归探索
        start_time = time.time()
        self._explore(root, depth=0)
        explore_time = time.time() - start_time
        
        # 找到最佳路径
        best_path = self._find_best_path(root)
        
        return {
            "task": task,
            "best_solution": best_path[-1].content if best_path else None,
            "best_score": best_path[-1].score if best_path else 0,
            "path": [{"content": n.content, "score": n.score} for n in best_path],
            "total_nodes_explored": self.node_count,
            "explore_time_seconds": round(explore_time, 2),
        }
    
    def _explore(self, node: ThoughtNode, depth: int):
        """
        递归探索思维树
        
        在每个节点上：
        1. 生成多个可能的下一步
        2. 评估每个选项
        3. 对高分选项递归探索
        """
        # 达到最大深度，停止探索
        if depth >= self.max_depth:
            node.is_leaf = True
            return
        
        # 生成子节点
        candidates = self._generate_candidates(node.content, depth)
        
        for candidate_content in candidates[:self.max_branches]:
            self.node_count += 1
            
            # 评估这个候选方案
            score = self._evaluate(candidate_content, node.content)
            
            if score < self.eval_threshold:
                continue  # 分数太低，不继续探索
            
            # 创建子节点
            child = ThoughtNode(
                content=candidate_content,
                score=score,
                depth=depth + 1,
            )
            node.children.append(child)
            
            # 递归探索
            self._explore(child, depth + 1)
    
    def _generate_candidates(self, current_thought: str, depth: int) -> list[str]:
        """生成当前思路的多个可能延续"""
        prompt = f"""你正在思考一个问题。

当前进展：{current_thought}
思考深度：{depth + 1}/{self.max_depth}

请生成 {self.max_branches} 个可能的下一步思考方向。每个方向一行，简洁明了。
不要编号，每行一个想法："""

        response = self.llm_client.generate(prompt)
        
        # 清理输出：去掉空行和编号
        lines = [line.strip() for line in response.split("\n") if line.strip()]
        # 去掉行首的编号（如 "1. " "2. "）
        cleaned = []
        for line in lines:
            # 去掉 "1. " "2. " 这样的前缀
            import re
            cleaned_line = re.sub(r'^\d+[\.\)]\s*', '', line)
            if cleaned_line:
                cleaned.append(cleaned_line)
        
        return cleaned
    
    def _evaluate(self, thought: str, parent_thought: str) -> float:
        """评估一个思维方向的价值"""
        prompt = f"""请评估以下思考方向的价值。

原始问题：{parent_thought}
新的思考：{thought}

请给出 0-1 之间的分数（0 表示完全没价值，1 表示非常有价值）。
只输出一个数字："""

        response = self.llm_client.generate(prompt)
        
        try:
            # 提取数字
            import re
            numbers = re.findall(r'[\d.]+', response.strip())
            if numbers:
                score = float(numbers[0])
                return max(0.0, min(1.0, score))  # 确保在 0-1 范围内
        except (ValueError, IndexError):
            pass
        
        return 0.5  # 默认分数
    
    def _find_best_path(self, root: ThoughtNode) -> list[ThoughtNode]:
        """
        找到从根到叶的最高分路径
        
        使用贪心策略：每一步选择分数最高的子节点
        """
        if not root.children:
            return [root]
        
        best_child = max(root.children, key=lambda n: n.score)
        return [root] + self._find_best_path(best_child)
    
    def visualize_tree(self, node: ThoughtNode, prefix: str = "", is_last: bool = True):
        """可视化思维树（控制台输出）"""
        connector = "└── " if is_last else "├── "
        score_str = f"[{node.score:.2f}]" if node.score > 0 else ""
        print(f"{prefix}{connector}{node.content[:60]}... {score_str}")
        
        new_prefix = prefix + ("    " if is_last else "│   ")
        for i, child in enumerate(node.children):
            self.visualize_tree(child, new_prefix, i == len(node.children) - 1)
```

### 15.3.3 ToT 的优势与代价

ToT 相比 CoT 的优势很明显：它不会"吊死在一棵树上"，而是探索多条路径，大大降低了选错方向的风险。

但代价也很明显：**计算成本成倍增长**。如果每步生成 3 个分支，深度为 3，那就要评估 3 + 9 + 27 = 39 个节点，每个节点都要调用 LLM 两次（一次生成、一次评估）。这意味着 ToT 的计算成本大约是 CoT 的 10 倍以上。

所以 ToT 适合那些"选错方向代价很大"的任务（比如重要的决策、创造性写作、复杂问题求解），而不适合那些"即使选错也容易修正"的简单任务。

---

## 15.4 Goal-of-Action (GoA) 目标导向规划

### 15.4.1 什么是 GoA

CoT 和 ToT 都是"正向规划"——从起点出发，一步步往前走。GoA 则是"反向规划"——从最终目标出发，反向推导出需要完成的步骤。

这就像你计划一次旅行：不是从"今天早上起床"开始想，而是从"我想在国庆节去日本看红叶"这个目标出发，反向推导出"需要办护照"、"需要买机票"、"需要订酒店"、"需要准备日语常用语"等前置条件。

GoA 的优势在于：它能确保所有步骤都是"为了达成目标"而存在的，不会做无用功。

### 15.4.2 GoA 规划器的实现

```python
class GoalOrientedPlanner:
    """
    目标导向规划器
    
    从最终目标出发，反向推导：
    1. 要达到这个目标，需要满足什么前置条件？
    2. 要满足这些前置条件，又需要什么更基础的条件？
    3. 一直推导到最基础的、可以直接执行的步骤
    4. 然后反转顺序，得到正向的执行计划
    """
    
    def __init__(self, llm_client, max_prerequisite_depth: int = 3):
        """
        参数:
            llm_client: LLM 客户端
            max_prerequisite_depth: 最大前置条件推导深度
        """
        self.llm_client = llm_client
        self.max_depth = max_prerequisite_depth
    
    def plan_from_goal(self, goal: str) -> dict:
        """
        从目标反向推导计划
        
        第一步：分析目标，识别所有前置条件
        第二步：递归推导每个前置条件的子前置条件
        第三步：将推导结果反转为正向执行顺序
        """
        # 反向推导
        prerequisite_tree = self._backward_chain(goal, depth=0)
        
        # 转换为正向执行顺序
        execution_order = self._flatten_to_order(prerequisite_tree)
        
        # 识别关键路径（最长的依赖链）
        critical_path = self._find_critical_path(prerequisite_tree)
        
        return {
            "goal": goal,
            "prerequisite_tree": prerequisite_tree,
            "execution_order": execution_order,
            "critical_path": critical_path,
            "total_steps": len(execution_order),
        }
    
    def _backward_chain(self, goal: str, depth: int) -> dict:
        """
        反向链：递归推导前置条件
        
        对于每个目标，询问 LLM：
        "要达到这个目标，需要先完成什么？"
        """
        if depth >= self.max_depth:
            return {
                "condition": goal,
                "is_atomic": True,
                "sub_conditions": [],
            }
        
        prompt = f"""要达到以下目标，需要先满足什么前置条件？

目标：{goal}
当前推导深度：{depth + 1}/{self.max_depth}

请列出所有必要的前置条件（2-4 个），每行一个。
如果这个目标已经可以直接执行（不需要更多前置条件），请回复"可以直接执行"："""

        response = self.llm_client.generate(prompt)
        
        # 检查是否可以直接执行
        if "可以直接执行" in response or "directly" in response.lower():
            return {
                "condition": goal,
                "is_atomic": True,
                "sub_conditions": [],
            }
        
        # 解析前置条件
        prerequisites = []
        for line in response.split("\n"):
            line = line.strip()
            if line and not line.startswith("如果"):
                # 去掉编号
                import re
                cleaned = re.sub(r'^\d+[\.\)]\s*', '', line)
                if cleaned:
                    prerequisites.append(cleaned)
        
        # 递归推导每个前置条件
        sub_conditions = []
        for prereq in prerequisites[:4]:  # 限制最多 4 个
            sub_tree = self._backward_chain(prereq, depth + 1)
            sub_conditions.append(sub_tree)
        
        return {
            "condition": goal,
            "is_atomic": False,
            "sub_conditions": sub_conditions,
        }
    
    def _flatten_to_order(self, tree: dict) -> list[str]:
        """
        将前置条件树转换为正向执行顺序
        
        使用拓扑排序：没有子依赖的条件先执行
        """
        order = []
        self._topo_sort(tree, order, set())
        return order
    
    def _topo_sort(self, node: dict, order: list, visited: set):
        """拓扑排序辅助方法"""
        condition = node["condition"]
        if condition in visited:
            return
        visited.add(condition)
        
        # 先处理子条件（依赖项）
        for sub in node.get("sub_conditions", []):
            self._topo_sort(sub, order, visited)
        
        # 再添加当前条件
        order.append(condition)
    
    def _find_critical_path(self, tree: dict) -> list[str]:
        """
        找到关键路径：依赖链最长的那条路径
        
        关键路径决定了整个计划的最短完成时间
        """
        if tree["is_atomic"] or not tree.get("sub_conditions"):
            return [tree["condition"]]
        
        # 找到最长的子路径
        longest_sub_path = []
        for sub in tree["sub_conditions"]:
            sub_path = self._find_critical_path(sub)
            if len(sub_path) > len(longest_sub_path):
                longest_sub_path = sub_path
        
        return longest_sub_path + [tree["condition"]]
```

### 15.4.3 GoA 与 CoT 的对比

GoA 和 CoT 的核心区别在于思考方向：

- **CoT 是"因为……所以……"**：因为有这些资源和条件，所以我能做这些事情
- **GoA 是"如果要……就得……"**：如果要达到这个目标，就得先做这些事情

在实际应用中，GoA 特别适合以下场景：
- 目标非常明确，但实现路径不明确的任务
- 需要确保不遗漏关键步骤的任务
- 资源有限，需要精确规划每一步的任务

---

## 15.5 Plan-Execute 循环

### 15.5.1 为什么需要 Plan-Execute 循环

前面介绍的三种规划策略都有一个共同的问题：它们都假设规划和执行是两个独立的阶段——先规划好，再执行。但在现实中，计划往往赶不上变化。

一个更好的方式是让规划和执行交替进行：先做一个简单的计划，执行几步，根据执行结果调整计划，再执行，如此循环。这就是 Plan-Execute 循环的核心思想。

### 15.5.2 Plan-Execute 循环的实现

```python
import json
import time
from enum import Enum


class StepStatus(Enum):
    PENDING = "pending"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"
    FAILED = "failed"
    SKIPPED = "skipped"


class PlanExecuteLoop:
    """
    规划-执行循环
    
    核心流程：
    1. 制定初始计划
    2. 选择下一个要执行的步骤
    3. 执行该步骤
    4. 根据执行结果评估是否需要调整计划
    5. 如果还有未完成的步骤，回到第 2 步
    6. 所有步骤完成后，综合结果
    """
    
    def __init__(self, llm_client, max_iterations: int = 15,
                 max_replans: int = 3):
        """
        参数:
            llm_client: LLM 客户端
            max_iterations: 最大执行轮数（防止无限循环）
            max_replans: 最大重新规划次数
        """
        self.llm_client = llm_client
        self.max_iterations = max_iterations
        self.max_replans = max_replans
        self.replan_count = 0
    
    def run(self, task: str) -> dict:
        """
        执行任务
        
        返回完整的执行日志和最终结果
        """
        # 第一步：制定初始计划
        print(f"\n{'='*60}")
        print(f"任务: {task}")
        print(f"{'='*60}")
        
        plan = self._create_plan(task)
        execution_log = []
        
        print(f"\n初始计划 ({len(plan.get('steps', []))} 个步骤):")
        for step in plan.get("steps", []):
            print(f"  - {step.get('description', '未描述')}")
        
        # 执行循环
        for iteration in range(self.max_iterations):
            if not plan.get("steps"):
                break
            
            # 选择下一个要执行的步骤
            current_step = self._select_next_step(plan, execution_log)
            if current_step is None:
                break
            
            # 标记为进行中
            current_step["status"] = StepStatus.IN_PROGRESS.value
            
            # 执行步骤
            print(f"\n[迭代 {iteration + 1}] 执行: {current_step['description'][:50]}...")
            start_time = time.time()
            result = self._execute_step(current_step, execution_log)
            duration = time.time() - start_time
            
            # 记录执行结果
            step_result = {
                "iteration": iteration + 1,
                "step": current_step["description"],
                "result": result,
                "duration": round(duration, 2),
            }
            execution_log.append(step_result)
            
            # 评估结果
            if result.get("success", False):
                current_step["status"] = StepStatus.COMPLETED.value
                print(f"  完成! ({duration:.1f}秒)")
                
                # 移除已完成的步骤
                plan["steps"] = [
                    s for s in plan.get("steps", [])
                    if s.get("status") != StepStatus.COMPLETED.value
                ]
            else:
                current_step["status"] = StepStatus.FAILED.value
                print(f"  失败: {result.get('error', '未知错误')}")
                
                # 判断是否需要重新规划
                if self.replan_count < self.max_replans:
                    plan = self._replan(task, execution_log, plan)
                    self.replan_count += 1
        
        # 综合最终结果
        final_result = self._synthesize(task, execution_log)
        
        return {
            "task": task,
            "total_iterations": len(execution_log),
            "replan_count": self.replan_count,
            "execution_log": execution_log,
            "final_result": final_result,
            "success": all(
                r.get("result", {}).get("success", False)
                for r in execution_log
            ),
        }
    
    def _create_plan(self, task: str) -> dict:
        """创建初始计划"""
        prompt = f"""请为以下任务制定执行计划。

任务：{task}

输出 JSON 格式：
{{
    "goal": "最终目标",
    "steps": [
        {{
            "id": 1,
            "description": "步骤描述",
            "dependencies": [],
            "tools_needed": ["需要的工具"]
        }}
    ]
}}

只输出 JSON："""

        response = self.llm_client.generate(prompt)
        
        try:
            plan = json.loads(response)
            # 为每个步骤添加状态
            for step in plan.get("steps", []):
                step["status"] = StepStatus.PENDING.value
            return plan
        except json.JSONDecodeError:
            return {"goal": task, "steps": []}
    
    def _select_next_step(self, plan: dict, log: list) -> dict | None:
        """选择下一个要执行的步骤"""
        completed_steps = {r["step"] for r in log if r.get("result", {}).get("success")}
        
        for step in plan.get("steps", []):
            if step.get("status") != StepStatus.PENDING.value:
                continue
            
            # 检查依赖是否满足
            deps = step.get("dependencies", [])
            # 简化：假设依赖通过步骤 ID 匹配
            # 实际实现中需要更精确的依赖检查
            
            return step
        
        return None
    
    def _execute_step(self, step: dict, log: list) -> dict:
        """执行单个步骤"""
        context = "\n".join(
            f"步骤 {r['step'][:30]}: {'成功' if r.get('result', {}).get('success') else '失败'}"
            for r in log[-5:]  # 只看最近 5 条记录
        )
        
        prompt = f"""请执行以下步骤：

步骤：{step['description']}
可用工具：{json.dumps(step.get('tools_needed', []), ensure_ascii=False)}
之前的执行记录：
{context if context else '无'}

请输出结果（JSON 格式）：
{{
    "output": "执行结果",
    "success": true,
    "notes": "备注"
}}"""

        response = self.llm_client.generate(prompt)
        
        try:
            return json.loads(response)
        except json.JSONDecodeError:
            return {"output": response, "success": True, "notes": "返回非JSON格式"}
    
    def _replan(self, task: str, log: list, current_plan: dict) -> dict:
        """根据执行结果重新规划"""
        results_text = "\n".join(
            f"- {r['step'][:40]}: {'成功' if r.get('result', {}).get('success') else '失败 - ' + str(r.get('result', {}).get('error', ''))}"
            for r in log
        )
        
        prompt = f"""任务执行过程中遇到了问题，需要重新规划。

原始任务：{task}

已完成的步骤和结果：
{results_text}

请根据已完成的工作，重新制定剩余步骤的计划。
输出新的 JSON 格式计划（只包含未完成的步骤）："""

        response = self.llm_client.generate(prompt)
        
        try:
            plan = json.loads(response)
            for step in plan.get("steps", []):
                step["status"] = StepStatus.PENDING.value
            return plan
        except json.JSONDecodeError:
            return current_plan
    
    def _synthesize(self, task: str, log: list) -> str:
        """综合所有执行结果，给出最终答案"""
        results_text = "\n\n".join(
            f"步骤: {r['step']}\n结果: {json.dumps(r.get('result', {}), ensure_ascii=False)}"
            for r in log
        )
        
        prompt = f"""请综合以下执行结果，给出最终答案。

任务：{task}

执行结果：
{results_text}

最终答案："""

        return self.llm_client.generate(prompt)
```

### 15.5.3 运行演示

```python
def demo_plan_execute():
    """演示 Plan-Execute 循环"""
    
    class MockLLMClient:
        """模拟 LLM 客户端，用于测试"""
        def __init__(self):
            self.call_count = 0
        
        def generate(self, prompt: str) -> str:
            self.call_count += 1
            
            if "制定执行计划" in prompt or "制定计划" in prompt:
                return json.dumps({
                    "goal": "编写一个简单的计算器程序",
                    "steps": [
                        {"id": 1, "description": "设计计算器的功能需求", "dependencies": [], "tools_needed": []},
                        {"id": 2, "description": "实现加减乘除四个基本运算", "dependencies": [1], "tools_needed": ["code_editor"]},
                        {"id": 3, "description": "实现用户输入和输出界面", "dependencies": [2], "tools_needed": ["code_editor"]},
                        {"id": 4, "description": "测试计算器的正确性", "dependencies": [3], "tools_needed": ["terminal"]},
                    ]
                })
            elif "请执行以下步骤" in prompt:
                return json.dumps({
                    "output": f"步骤执行完成（第 {self.call_count} 次调用）",
                    "success": True,
                    "notes": ""
                })
            elif "综合以下执行结果" in prompt:
                return "计算器程序已完成开发和测试，所有功能正常运行。"
            else:
                return '{"steps": []}'
    
    # 运行演示
    llm = MockLLMClient()
    planner = PlanExecuteLoop(llm, max_iterations=10)
    result = planner.run("编写一个简单的计算器程序")
    
    print(f"\n{'='*60}")
    print(f"执行完成!")
    print(f"总迭代次数: {result['total_iterations']}")
    print(f"重新规划次数: {result['replan_count']}")
    print(f"最终结果: {result['final_result'][:100]}...")


if __name__ == "__main__":
    demo_plan_execute()
```

---

## 15.6 规划策略的选择指南

### 15.6.1 不同场景的推荐策略

选择哪种规划策略，取决于任务的特征：

**使用 CoT 的场景：**
- 任务步骤明确，不太需要探索多种方案
- 计算资源有限（CoT 成本最低）
- 需要快速给出结果
- 任务相对简单，比如"写一封邮件"、"查询天气"

**使用 ToT 的场景：**
- 任务有多种可能的解决方案，需要比较选择
- 选错方向的代价很大（比如重要决策）
- 有充足的时间和计算资源
- 需要创造性方案（比如写作、设计）

**使用 GoA 的场景：**
- 目标非常明确，但不知道怎么达到
- 需要确保不遗漏关键步骤
- 有很多前置条件需要梳理
- 比如"部署一个生产环境"、"通过一个认证考试"

**使用 Plan-Execute 循环的场景：**
- 任务复杂度高，不可能一次规划到位
- 执行过程中可能遇到意外情况
- 需要根据中间结果调整策略
- 比如"开发一个完整的应用"、"做一次科学研究"

### 15.6.2 混合使用

在实际应用中，这些策略往往不是互斥的，而是可以组合使用。例如：

- 用 GoA 来确定整体方向和关键步骤
- 在每个关键步骤内部用 CoT 来规划具体执行
- 如果某个步骤特别重要或特别困难，用 ToT 来探索多种方案
- 整体上用 Plan-Execute 循环来串联所有步骤

---

## 15.7 常见坑

### 15.7.1 过度规划

**问题描述：** 花大量时间制定一个"完美"的计划，但执行时发现很多假设不成立，计划不可行。这就像旅游前花一周时间规划每一天的行程，到了目的地发现天气不好、景点关闭、交通不便，计划全部作废。

**解决方案：** 采用增量规划。先制定一个粗略的计划（3-5 个大步骤），执行几步后再细化后续步骤。计划不需要完美，只要方向正确就行。

```python
def incremental_plan(task: str, llm_client) -> dict:
    """增量规划：先粗后细"""
    
    # 第一步：制定粗略计划（只确定大方向）
    coarse_plan = llm_client.generate(
        f"请为以下任务制定 3-5 个大步骤的粗略计划：{task}"
    )
    
    # 第二步：执行第一个大步骤后，再细化后续步骤
    # ...（此处省略执行细节）
    
    return {"coarse_plan": coarse_plan}
```

### 15.7.2 规划与执行脱节

**问题描述：** 计划写得很详细，但执行时 LLM "忘了"按照计划来做，自己又开始"自由发挥"。这在 Prompt 工程中很常见——LLM 有时候会"无视"你给它的指令。

**解决方案：** 在执行每个步骤时，把计划作为上下文传给 LLM，并明确要求它"按照以下计划执行"。

### 15.7.3 忽略错误处理

**问题描述：** 计划只考虑了"一切顺利"的情况，没有考虑"如果失败了怎么办"。一旦某个步骤失败，整个计划就崩了。

**解决方案：** 在计划中为每个步骤标注失败时的回退策略。在 Plan-Execute 循环中，失败后自动触发重新规划。

```python
# 好的计划应该包含失败处理
step_with_fallback = {
    "id": 1,
    "description": "调用天气 API 获取天气信息",
    "success_criteria": "返回 JSON 格式的天气数据",
    "fallback": {
        "if_fails": "使用缓存的天气数据",
        "last_resort": "返回'无法获取天气信息'的提示"
    }
}
```

### 15.7.4 忽略资源约束

**问题描述：** 规划时不考虑时间、Token、API 调用次数等资源限制，导致计划虽然理论上可行，但实际上会超出预算。

**解决方案：** 在规划阶段就明确资源约束，并在计划中估算每步的资源消耗。

---

## 15.8 练习题

### 练习 1：CoT 规划器的实现与测试

实现一个基于 CoT 的任务规划器，要求：
- 接收自然语言任务描述
- 输出结构化的 JSON 执行计划（包含步骤、依赖、时间估算）
- 能执行计划并记录结果
- 支持计划的动态调整

测试任务：
1. "帮我规划一次三天的北京旅行"
2. "设计并实现一个简单的待办事项 API"
3. "帮我准备一场 30 分钟的技术分享"

对比三种任务的计划质量，分析 CoT 的优缺点。

### 练习 2：ToT 探索器的实现与可视化

实现一个 ToT 规划器，要求：
- 能探索多条执行路径
- 能评估每条路径的价值
- 选择最优路径执行
- 支持思维树的可视化输出（控制台打印树形结构）

测试：用 ToT 解决"如何用 Python 实现一个 LRU 缓存"这个问题，观察它探索了哪些方案，最终选择了哪个。

### 练习 3：GoA 反向规划器

实现一个 GoA 规划器，要求：
- 从目标反向推导前置条件
- 支持递归推导（至少 3 层深度）
- 将反向结果转换为正向执行顺序
- 识别关键路径

测试：用 GoA 规划"在自己的服务器上部署一个 Docker 化的 Web 应用"。

### 练习 4：Plan-Execute 循环的实现

实现一个完整的 Plan-Execute 循环，要求：
- 支持动态重规划
- 能处理执行中的错误
- 有最大迭代次数限制
- 记录完整的执行日志

### 练习 5：规划策略的对比实验

对同一个任务，分别用 CoT、ToT、GoA 三种策略规划，对比以下指标：
- 计划的完整度（是否遗漏关键步骤）
- 计划的可行性（步骤是否可执行）
- 计算成本（LLM 调用次数）
- 最终执行效果

分析哪种策略在什么场景下更优。

---

## 15.9 实战任务

### 任务：构建智能任务规划系统

**目标：** 构建一个能自动规划和执行复杂任务的 Agent，支持多种规划策略。

**要求：**

1. 实现 CoT、ToT、GoA 三种规划策略
2. 根据任务复杂度自动选择最合适的策略
3. 支持 Plan-Execute 循环
4. 有错误处理和重新规划机制
5. 记录完整的执行日志

**参考实现框架：**

```python
class SmartPlanner:
    """智能规划系统：根据任务特征选择最佳策略"""
    
    def __init__(self, llm_client):
        self.llm_client = llm_client
        self.cot = CoTPlanner(llm_client)
        self.tot = TreeOfThoughtsPlanner(llm_client)
        self.goa = GoalOrientedPlanner(llm_client)
        self.loop = PlanExecuteLoop(llm_client)
    
    def plan_and_execute(self, task: str) -> dict:
        """根据任务特征选择策略并执行"""
        
        # 1. 分析任务复杂度
        complexity = self._analyze_complexity(task)
        
        # 2. 选择策略
        if complexity == "simple":
            print("选择策略: CoT（简单任务，直接规划）")
            plan = self.cot.plan(task)
            return self.cot.execute_plan(plan)
        elif complexity == "medium":
            print("选择策略: GoA + Plan-Execute（中等复杂度）")
            return self.loop.run(task)
        else:
            print("选择策略: ToT（复杂任务，多路径探索）")
            result = self.tot.solve(task)
            # 用最佳方案执行
            plan = self.cot.plan(result["best_solution"])
            return self.cot.execute_plan(plan)
    
    def _analyze_complexity(self, task: str) -> str:
        """分析任务复杂度"""
        prompt = f"""分析以下任务的复杂度，选择一个级别：
- simple: 单步可完成的简单任务
- medium: 需要多个步骤，但路径明确
- complex: 需要多步骤，且有多种可能的解决方案

任务：{task}

复杂度（simple/medium/complex）："""

        response = self.llm_client.generate(prompt)
        
        if "simple" in response.lower():
            return "simple"
        elif "complex" in response.lower():
            return "complex"
        else:
            return "medium"
```

---

## 15.10 本章小结

- **规划是 Agent 处理复杂任务的关键能力**。没有规划能力的 Agent 只能"走一步看一步"，面对复杂任务容易迷失方向、遗漏步骤、浪费资源。

- **Chain-of-Thought（CoT）** 是最基础的规划策略，让 LLM 按步骤展示推理过程。它的优势是简单、成本低；局限是只探索一条路径，没有回溯能力。适合大多数日常任务。

- **Tree-of-Thought（ToT）** 让 Agent 探索多条可能的路径，评估每条路径的价值，选择最优方案。它的优势是能找到更好的方案；局限是计算成本高（约 10 倍于 CoT）。适合重要决策和创造性任务。

- **Goal-of-Action（GoA）** 从最终目标出发，反向推导需要的步骤。它的优势是确保不遗漏关键步骤、不做无用功；适合目标明确但路径不明确的任务。

- **Plan-Execute 循环** 让规划和执行交替进行，每次执行后根据结果调整计划。它的优势是能适应变化、处理意外情况；适合复杂度高、执行过程中可能遇到意外的任务。

- 选择规划策略时需要考虑**任务复杂度、计算资源、时间约束、选错方向的代价**等因素。在实际应用中，这些策略往往可以组合使用。

- **规划中的常见陷阱**包括过度规划、规划与执行脱节、忽略错误处理、忽略资源约束。避免这些陷阱的关键是保持计划的灵活性和可调整性。

