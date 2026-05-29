# 第48章：自学习 Agent —— Hermes 与学习循环

## 学习目标

通过本章的学习，你将能够：
1. 理解自学习 Agent 的核心理念和设计原则
2. 掌握学习循环（Learning Loop）的实现方式
3. 学会构建具有经验积累和知识提取能力的 Agent
4. 理解反思、总结和模式识别在 Agent 学习中的作用
5. 掌握长期记忆与短期记忆的协同机制
6. 能够构建一个完整的自学习 Agent 系统

## 核心问题

在前面的章节中，我们构建的 Agent 都有一个共同的局限：它们不会从经验中学习。每次对话都是全新的开始，Agent 不记得之前犯过的错误，不会总结成功的策略，也不会根据用户反馈调整自己的行为。

这就好比一个每天都会失忆的人——他可能很聪明，但他永远无法变得更有经验。他每天都在重新学习如何与人交流，如何解决问题。

那么，有没有可能让 Agent 具有"学习"能力？让它能够从每次交互中提取经验，不断改进自己的行为？

自学习 Agent（Self-Learning Agent）正是为了解决这个问题而设计的。它的核心思想是：Agent 不仅要完成任务，还要从任务执行过程中提取经验，将这些经验存储在记忆中，并在未来的任务中使用这些经验来改进表现。

Hermes 是这个领域的一个重要研究项目，它提出了一种系统化的学习循环框架。

---

## 原理讲解

### 什么是自学习 Agent

自学习 Agent 是一种能够从经验中学习并改进自身行为的 Agent。与传统的 Agent 不同，自学习 Agent 不仅仅执行任务，还会：

1. **记录经验**：每次任务执行的过程和结果都被记录下来
2. **提取模式**：从历史经验中识别出成功的策略和失败的原因
3. **更新知识**：将提取的模式整合到 Agent 的知识库中
4. **应用学习**：在新的任务中使用学到的知识来改进表现

这种学习过程是持续的、渐进的。随着经验的积累，Agent 会变得越来越擅长处理特定类型的任务。

### 学习循环的核心机制

自学习 Agent 的核心是学习循环（Learning Loop）。一个完整的学习循环包括以下几个阶段：

**1. 任务执行（Task Execution）**
Agent 接收任务并尝试完成它。在这个过程中，Agent 会调用各种工具、进行推理决策、与用户交互。

**2. 经验记录（Experience Recording）**
在任务执行过程中，Agent 详细记录每一步的操作、决策和结果。这些记录形成了原始的经验数据。

**3. 反思评估（Reflection & Evaluation）**
任务完成后，Agent 对自己的表现进行反思：哪些地方做得好？哪些地方可以改进？遇到了什么困难？

**4. 模式提取（Pattern Extraction）**
从反思中提取可复用的模式：成功的策略是什么？失败的根本原因是什么？有没有通用的解决方案？

**5. 知识更新（Knowledge Update）**
将提取的模式整合到 Agent 的知识库中，更新长期记忆。

**6. 行为调整（Behavior Adjustment）**
根据学到的知识调整 Agent 的行为策略，在未来的任务中应用这些改进。

```
任务执行 → 经验记录 → 反思评估 → 模式提取
    ↑                                    ↓
    ← 行为调整 ← 知识更新 ←
```

### Hermes 架构

Hermes 是一个自学习 Agent 框架，它实现了一个完整的学习循环。Hermes 的核心组件包括：

**Experience Memory（经验记忆）**
存储 Agent 的历史经验。每条经验包括：任务描述、执行过程、最终结果、反思评价。

**Pattern Store（模式库）**
存储从经验中提取的可复用模式。每个模式包括：触发条件、执行策略、预期结果。

**Reflection Engine（反思引擎）**
负责对任务执行过程进行反思，识别成功和失败的原因。

**Pattern Extractor（模式提取器）**
负责从反思中提取可复用的模式，并将它们整合到模式库中。

**Strategy Adapter（策略适配器）**
负责根据当前任务和模式库，选择合适的执行策略。

### 经验的表示与存储

经验的表示方式对学习效果有重要影响。一个好的经验表示应该包含以下信息：

```python
experience = {
    "task": {
        "description": "用户请求...",
        "type": "data_analysis",
        "complexity": "medium",
    },
    "execution": {
        "steps": [...],
        "tools_used": [...],
        "decisions_made": [...],
    },
    "result": {
        "success": True,
        "quality_score": 0.85,
        "user_feedback": "很好，但可以更详细",
    },
    "reflection": {
        "strengths": ["使用了合适的工具", "分析逻辑清晰"],
        "weaknesses": ["遗漏了一个维度的分析"],
        "lessons_learned": ["数据分析应该从多个维度进行"],
    },
    "metadata": {
        "timestamp": "2024-01-15T10:30:00",
        "duration_seconds": 45,
        "token_usage": 2500,
    },
}
```

### 模式的类型与提取

从经验中可以提取多种类型的模式：

**成功模式**：在类似任务中反复成功的策略。
```
模式：数据分析任务
策略：先了解数据结构，再进行统计分析，最后可视化
触发条件：任务涉及数据处理和分析
预期效果：高质量的分析报告
```

**失败模式**：导致失败的常见原因。
```
模式：工具调用失败
原因：参数格式错误
解决方案：在调用前验证参数格式
```

**用户偏好模式**：用户的行为偏好。
```
模式：用户偏好
特点：喜欢详细的解释，不喜欢过多的代码
策略：在解释概念时，多用自然语言，少用代码
```

**效率模式**：提高效率的方法。
```
模式：并行调用
策略：对于独立的工具调用，使用并行执行
适用场景：需要查询多个独立数据源时
```

### 反思机制的实现

反思是自学习 Agent 最核心的能力之一。有效的反思需要回答以下问题：

1. **目标达成**：我是否完成了用户的请求？
2. **过程评估**：我的执行过程是否高效？
3. **错误分析**：如果失败了，原因是什么？
4. **改进方向**：下次如何做得更好？

反思可以通过两种方式实现：

**即时反思**：每次任务完成后立即进行反思。这种方式简单直接，但反思质量可能不高。

**延迟反思**：积累一定数量的经验后再进行批量反思。这种方式可以发现跨任务的模式，但实现更复杂。

### 知识的迁移与泛化

自学习 Agent 的一个重要目标是将学到的知识迁移到新的任务中。这需要：

1. **抽象化**：将具体的经验抽象为通用的策略
2. **类比推理**：识别新任务与历史任务的相似性
3. **组合创新**：将多个已知策略组合成新的解决方案

例如，Agent 在处理"数据分析"任务时学到的策略，可能也适用于"日志分析"任务，因为两者在本质上都是从数据中提取信息。

---

## 完整代码示例

### 示例 1：基础经验记录系统

```python
"""
基础经验记录系统
记录 Agent 的每次任务执行过程
"""

import json
import os
import hashlib
from datetime import datetime
from typing import Dict, List, Optional, Any
from dataclasses import dataclass, field, asdict


@dataclass
class ExecutionStep:
    """执行步骤"""
    step_number: int
    action: str
    tool_used: Optional[str] = None
    input_data: Optional[Dict] = None
    output_data: Optional[Dict] = None
    success: bool = True
    error_message: Optional[str] = None
    duration_ms: float = 0


@dataclass
class Reflection:
    """反思评价"""
    strengths: List[str] = field(default_factory=list)
    weaknesses: List[str] = field(default_factory=list)
    lessons_learned: List[str] = field(default_factory=list)
    improvement_suggestions: List[str] = field(default_factory=list)
    quality_score: float = 0.0  # 0-1


@dataclass
class Experience:
    """一条经验"""
    experience_id: str
    task_description: str
    task_type: str
    execution_steps: List[ExecutionStep]
    final_result: str
    success: bool
    reflection: Reflection
    timestamp: str = field(default_factory=lambda: datetime.now().isoformat())
    duration_seconds: float = 0
    token_usage: int = 0
    tags: List[str] = field(default_factory=list)


class ExperienceRecorder:
    """经验记录器"""
    
    def __init__(self, storage_path: str = "experiences"):
        self.storage_path = storage_path
        os.makedirs(storage_path, exist_ok=True)
        self.current_experience: Optional[Experience] = None
        self.current_steps: List[ExecutionStep] = []
        self.step_counter = 0
    
    def start_experience(self, task_description: str, task_type: str = "general"):
        """开始记录一次经验"""
        exp_id = hashlib.md5(
            f"{task_description}{datetime.now().isoformat()}".encode()
        ).hexdigest()[:12]
        
        self.current_experience = Experience(
            experience_id=exp_id,
            task_description=task_description,
            task_type=task_type,
            execution_steps=[],
            final_result="",
            success=False,
            reflection=Reflection(),
        )
        self.current_steps = []
        self.step_counter = 0
        
        print(f"\n[经验记录] 开始: {exp_id}")
        print(f"  任务: {task_description}")
    
    def record_step(self, action: str, tool_used: str = None,
                    input_data: Dict = None, output_data: Dict = None,
                    success: bool = True, error_message: str = None,
                    duration_ms: float = 0):
        """记录一个执行步骤"""
        self.step_counter += 1
        
        step = ExecutionStep(
            step_number=self.step_counter,
            action=action,
            tool_used=tool_used,
            input_data=input_data,
            output_data=output_data,
            success=success,
            error_message=error_message,
            duration_ms=duration_ms,
        )
        
        self.current_steps.append(step)
        
        status = "✓" if success else "✗"
        print(f"  [Step {self.step_counter}] {status} {action}")
    
    def finish_experience(self, final_result: str, success: bool,
                          duration_seconds: float = 0):
        """完成一次经验记录"""
        if not self.current_experience:
            return
        
        self.current_experience.execution_steps = self.current_steps
        self.current_experience.final_result = final_result
        self.current_experience.success = success
        self.current_experience.duration_seconds = duration_seconds
        
        # 自动进行基础反思
        self.current_experience.reflection = self._auto_reflect()
        
        # 保存经验
        self._save_experience(self.current_experience)
        
        print(f"\n[经验记录] 完成: {self.current_experience.experience_id}")
        print(f"  成功: {success}")
        print(f"  质量分数: {self.current_experience.reflection.quality_score:.2f}")
        
        return self.current_experience
    
    def _auto_reflect(self) -> Reflection:
        """自动反思"""
        steps = self.current_steps
        total_steps = len(steps)
        successful_steps = sum(1 for s in steps if s.success)
        failed_steps = total_steps - successful_steps
        
        strengths = []
        weaknesses = []
        lessons = []
        suggestions = []
        
        # 分析成功率
        if total_steps > 0:
            success_rate = successful_steps / total_steps
            if success_rate >= 0.8:
                strengths.append("任务执行成功率高")
            elif success_rate < 0.5:
                weaknesses.append("任务执行成功率低")
        
        # 分析工具使用
        tools_used = [s.tool_used for s in steps if s.tool_used]
        if tools_used:
            strengths.append(f"使用了 {len(set(tools_used))} 种工具")
        
        # 分析失败步骤
        failed_steps_list = [s for s in steps if not s.success]
        for step in failed_steps_list:
            if step.error_message:
                weaknesses.append(f"步骤 {step.step_number} 失败: {step.error_message}")
                lessons.append(f"需要避免: {step.action} 可能导致失败")
        
        # 分析效率
        total_duration = sum(s.duration_ms for s in steps)
        if total_duration > 0:
            avg_step_duration = total_duration / total_steps
            if avg_step_duration > 5000:  # 超过 5 秒
                suggestions.append("考虑优化耗时较长的步骤")
        
        # 计算质量分数
        quality_score = 0.5  # 基础分
        if self.current_experience and self.current_experience.success:
            quality_score += 0.3
        quality_score += min(0.2, success_rate * 0.2) if total_steps > 0 else 0
        
        return Reflection(
            strengths=strengths,
            weaknesses=weaknesses,
            lessons_learned=lessons,
            improvement_suggestions=suggestions,
            quality_score=min(1.0, quality_score),
        )
    
    def _save_experience(self, experience: Experience):
        """保存经验到文件"""
        filename = f"{experience.experience_id}.json"
        filepath = os.path.join(self.storage_path, filename)
        
        with open(filepath, 'w', encoding='utf-8') as f:
            json.dump(asdict(experience), f, ensure_ascii=False, indent=2)
    
    def load_experiences(self, limit: int = 100) -> List[Experience]:
        """加载最近的经验"""
        experiences = []
        
        files = sorted(
            [f for f in os.listdir(self.storage_path) if f.endswith('.json')],
            reverse=True,
        )[:limit]
        
        for filename in files:
            filepath = os.path.join(self.storage_path, filename)
            with open(filepath, 'r', encoding='utf-8') as f:
                data = json.load(f)
                
                # 重建对象
                steps = [ExecutionStep(**s) for s in data.get('execution_steps', [])]
                reflection_data = data.get('reflection', {})
                reflection = Reflection(
                    strengths=reflection_data.get('strengths', []),
                    weaknesses=reflection_data.get('weaknesses', []),
                    lessons_learned=reflection_data.get('lessons_learned', []),
                    improvement_suggestions=reflection_data.get('improvement_suggestions', []),
                    quality_score=reflection_data.get('quality_score', 0),
                )
                
                exp = Experience(
                    experience_id=data['experience_id'],
                    task_description=data['task_description'],
                    task_type=data.get('task_type', 'general'),
                    execution_steps=steps,
                    final_result=data.get('final_result', ''),
                    success=data.get('success', False),
                    reflection=reflection,
                    timestamp=data.get('timestamp', ''),
                    duration_seconds=data.get('duration_seconds', 0),
                    token_usage=data.get('token_usage', 0),
                    tags=data.get('tags', []),
                )
                experiences.append(exp)
        
        return experiences


def demo_recorder():
    """演示经验记录器"""
    print("=" * 60)
    print("经验记录器演示")
    print("=" * 60)
    
    recorder = ExperienceRecorder("demo_experiences")
    
    # 模拟一次任务执行
    recorder.start_experience("分析销售数据并生成报告", "data_analysis")
    
    recorder.record_step(
        "读取数据文件",
        tool_used="file_reader",
        input_data={"path": "sales.csv"},
        output_data={"rows": 1000, "columns": 5},
        duration_ms=150,
    )
    
    recorder.record_step(
        "数据清洗",
        tool_used="data_cleaner",
        input_data={"remove_nulls": True},
        output_data={"rows_after": 980},
        duration_ms=200,
    )
    
    recorder.record_step(
        "统计分析",
        tool_used="statistics",
        input_data={"metrics": ["mean", "median", "std"]},
        output_data={"mean": 1500, "median": 1200, "std": 500},
        duration_ms=100,
    )
    
    recorder.record_step(
        "生成图表",
        tool_used="matplotlib",
        input_data={"chart_type": "bar"},
        success=False,
        error_message="matplotlib 未安装",
        duration_ms=50,
    )
    
    recorder.record_step(
        "生成文本报告",
        tool_used="report_generator",
        input_data={"format": "markdown"},
        output_data={"report_length": 1500},
        duration_ms=300,
    )
    
    result = recorder.finish_experience(
        final_result="完成了数据分析，生成了文本报告。图表生成失败。",
        success=True,
        duration_seconds=8.0,
    )
    
    # 展示反思结果
    print(f"\n反思结果:")
    print(f"  优点: {result.reflection.strengths}")
    print(f"  不足: {result.reflection.weaknesses}")
    print(f"  经验教训: {result.reflection.lessons_learned}")
    print(f"  改进建议: {result.reflection.improvement_suggestions}")


if __name__ == "__main__":
    demo_recorder()
```

### 示例 2：模式提取器

```python
"""
模式提取器
从历史经验中提取可复用的模式
"""

import json
import os
from typing import Dict, List, Optional
from collections import Counter, defaultdict
from dataclasses import dataclass, field, asdict


@dataclass
class Pattern:
    """一个提取的模式"""
    pattern_id: str
    pattern_type: str  # success, failure, efficiency, preference
    description: str
    trigger_conditions: List[str]
    strategy: str
    expected_outcome: str
    confidence: float = 0.0  # 0-1
    occurrence_count: int = 0
    examples: List[str] = field(default_factory=list)


class PatternExtractor:
    """模式提取器"""
    
    def __init__(self, storage_path: str = "patterns"):
        self.storage_path = storage_path
        os.makedirs(storage_path, exist_ok=True)
        self.patterns: Dict[str, Pattern] = {}
        self._load_patterns()
    
    def _load_patterns(self):
        """加载已有的模式"""
        filepath = os.path.join(self.storage_path, "patterns.json")
        if os.path.exists(filepath):
            with open(filepath, 'r', encoding='utf-8') as f:
                data = json.load(f)
                for pid, pdata in data.items():
                    self.patterns[pid] = Pattern(**pdata)
    
    def _save_patterns(self):
        """保存模式"""
        filepath = os.path.join(self.storage_path, "patterns.json")
        data = {pid: asdict(p) for pid, p in self.patterns.items()}
        with open(filepath, 'w', encoding='utf-8') as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
    
    def extract_patterns(self, experiences: List[Dict]) -> List[Pattern]:
        """从经验列表中提取模式"""
        new_patterns = []
        
        # 提取成功模式
        success_patterns = self._extract_success_patterns(experiences)
        new_patterns.extend(success_patterns)
        
        # 提取失败模式
        failure_patterns = self._extract_failure_patterns(experiences)
        new_patterns.extend(failure_patterns)
        
        # 提取效率模式
        efficiency_patterns = self._extract_efficiency_patterns(experiences)
        new_patterns.extend(efficiency_patterns)
        
        # 保存所有新模式
        self._save_patterns()
        
        return new_patterns
    
    def _extract_success_patterns(self, experiences: List[Dict]) -> List[Pattern]:
        """提取成功模式"""
        patterns = []
        
        # 按任务类型分组
        by_type = defaultdict(list)
        for exp in experiences:
            if exp.get('success', False):
                by_type[exp.get('task_type', 'general')].append(exp)
        
        for task_type, exps in by_type.items():
            if len(exps) < 2:
                continue  # 需要至少 2 个经验才能形成模式
            
            # 找出共同的工具使用模式
            tool_sequences = []
            for exp in exps:
                tools = [s.get('tool_used') for s in exp.get('execution_steps', []) if s.get('tool_used')]
                tool_sequences.append(tools)
            
            # 找出最常见的工具组合
            common_tools = Counter()
            for seq in tool_sequences:
                for tool in seq:
                    common_tools[tool] += 1
            
            # 如果某些工具在所有成功经验中都使用了，提取为模式
            most_common = common_tools.most_common(3)
            if most_common:
                pattern = Pattern(
                    pattern_id=f"success_{task_type}_{len(self.patterns)}",
                    pattern_type="success",
                    description=f"成功的 {task_type} 任务通常使用以下工具: {', '.join(t[0] for t in most_common)}",
                    trigger_conditions=[f"任务类型为 {task_type}"],
                    strategy=f"优先使用: {', '.join(t[0] for t in most_common)}",
                    expected_outcome="任务成功完成",
                    confidence=len(exps) / (len(exps) + 5),  # 简单的贝叶斯估计
                    occurrence_count=len(exps),
                )
                patterns.append(pattern)
                self.patterns[pattern.pattern_id] = pattern
        
        return patterns
    
    def _extract_failure_patterns(self, experiences: List[Dict]) -> List[Pattern]:
        """提取失败模式"""
        patterns = []
        
        # 分析失败步骤
        failure_reasons = Counter()
        for exp in experiences:
            for step in exp.get('execution_steps', []):
                if not step.get('success', True) and step.get('error_message'):
                    failure_reasons[step['error_message']] += 1
        
        # 将常见的失败原因提取为模式
        for reason, count in failure_reasons.most_common(5):
            if count >= 2:
                pattern = Pattern(
                    pattern_id=f"failure_{len(self.patterns)}",
                    pattern_type="failure",
                    description=f"常见失败原因: {reason}",
                    trigger_conditions=[f"可能遇到: {reason}"],
                    strategy=f"预防措施: 检查相关前置条件",
                    expected_outcome="避免相同错误",
                    confidence=count / (count + 5),
                    occurrence_count=count,
                )
                patterns.append(pattern)
                self.patterns[pattern.pattern_id] = pattern
        
        return patterns
    
    def _extract_efficiency_patterns(self, experiences: List[Dict]) -> List[Pattern]:
        """提取效率模式"""
        patterns = []
        
        # 找出快速完成的任务
        fast_tasks = []
        slow_tasks = []
        
        for exp in experiences:
            duration = exp.get('duration_seconds', 0)
            steps = len(exp.get('execution_steps', []))
            
            if steps > 0 and duration > 0:
                avg_step_time = duration / steps
                if exp.get('success', False):
                    if avg_step_time < 2:  # 平均每步小于 2 秒
                        fast_tasks.append(exp)
                    elif avg_step_time > 5:  # 平均每步大于 5 秒
                        slow_tasks.append(exp)
        
        if fast_tasks and slow_tasks:
            # 比较快慢任务的差异
            pattern = Pattern(
                pattern_id=f"efficiency_{len(self.patterns)}",
                pattern_type="efficiency",
                description="高效任务通常有更少的步骤和更快的执行时间",
                trigger_conditions=["需要优化执行效率时"],
                strategy="减少不必要的步骤，使用更高效的工具",
                expected_outcome="减少执行时间",
                confidence=0.6,
                occurrence_count=len(fast_tasks),
            )
            patterns.append(pattern)
            self.patterns[pattern.pattern_id] = pattern
        
        return patterns
    
    def get_relevant_patterns(self, task_type: str, context: Dict = None) -> List[Pattern]:
        """获取与当前任务相关的模式"""
        relevant = []
        
        for pattern in self.patterns.values():
            # 检查触发条件
            is_relevant = False
            for condition in pattern.trigger_conditions:
                if task_type in condition:
                    is_relevant = True
                    break
            
            if is_relevant:
                relevant.append(pattern)
        
        # 按置信度排序
        relevant.sort(key=lambda p: p.confidence, reverse=True)
        
        return relevant
    
    def update_pattern_confidence(self, pattern_id: str, success: bool):
        """根据新的经验更新模式的置信度"""
        if pattern_id in self.patterns:
            pattern = self.patterns[pattern_id]
            pattern.occurrence_count += 1
            
            # 简单的置信度更新
            if success:
                pattern.confidence = min(1.0, pattern.confidence + 0.05)
            else:
                pattern.confidence = max(0.0, pattern.confidence - 0.1)
            
            self._save_patterns()


def demo_pattern_extraction():
    """演示模式提取"""
    print("=" * 60)
    print("模式提取器演示")
    print("=" * 60)
    
    extractor = PatternExtractor("demo_patterns")
    
    # 模拟历史经验
    experiences = [
        {
            "task_type": "data_analysis",
            "success": True,
            "execution_steps": [
                {"tool_used": "file_reader", "success": True},
                {"tool_used": "data_cleaner", "success": True},
                {"tool_used": "statistics", "success": True},
            ],
            "duration_seconds": 5.0,
        },
        {
            "task_type": "data_analysis",
            "success": True,
            "execution_steps": [
                {"tool_used": "file_reader", "success": True},
                {"tool_used": "pandas", "success": True},
                {"tool_used": "statistics", "success": True},
                {"tool_used": "matplotlib", "success": True},
            ],
            "duration_seconds": 8.0,
        },
        {
            "task_type": "data_analysis",
            "success": False,
            "execution_steps": [
                {"tool_used": "file_reader", "success": True},
                {"tool_used": "data_cleaner", "success": False, "error_message": "数据格式错误"},
                {"tool_used": "statistics", "success": False},
            ],
            "duration_seconds": 3.0,
        },
    ]
    
    # 提取模式
    new_patterns = extractor.extract_patterns(experiences)
    
    print(f"\n提取了 {len(new_patterns)} 个新模式:")
    for pattern in new_patterns:
        print(f"\n  [{pattern.pattern_type}] {pattern.description}")
        print(f"    触发条件: {pattern.trigger_conditions}")
        print(f"    策略: {pattern.strategy}")
        print(f"    置信度: {pattern.confidence:.2f}")
    
    # 获取相关模式
    print("\n\n查询 'data_analysis' 相关模式:")
    relevant = extractor.get_relevant_patterns("data_analysis")
    for pattern in relevant:
        print(f"  - {pattern.description} (置信度: {pattern.confidence:.2f})")


if __name__ == "__main__":
    demo_pattern_extraction()
```

### 示例 3：完整的自学习 Agent

```python
"""
完整的自学习 Agent
集成经验记录、模式提取、反思和策略适配
"""

import json
import os
from datetime import datetime
from typing import Dict, List, Optional, Any
from dataclasses import dataclass, field, asdict
import anthropic


@dataclass
class MemoryEntry:
    """记忆条目"""
    entry_type: str  # experience, pattern, preference, fact
    content: Dict
    importance: float = 0.5
    access_count: int = 0
    last_accessed: str = ""
    created_at: str = field(default_factory=lambda: datetime.now().isoformat())


class SelfLearningMemory:
    """自学习记忆系统"""
    
    def __init__(self, storage_path: str = "agent_memory"):
        self.storage_path = storage_path
        os.makedirs(storage_path, exist_ok=True)
        
        self.short_term: List[MemoryEntry] = []
        self.long_term: List[MemoryEntry] = []
        self.max_short_term = 20
        self.max_long_term = 500
        
        self._load_long_term()
    
    def _load_long_term(self):
        """加载长期记忆"""
        filepath = os.path.join(self.storage_path, "long_term_memory.json")
        if os.path.exists(filepath):
            with open(filepath, 'r', encoding='utf-8') as f:
                data = json.load(f)
                self.long_term = [MemoryEntry(**item) for item in data]
    
    def _save_long_term(self):
        """保存长期记忆"""
        filepath = os.path.join(self.storage_path, "long_term_memory.json")
        data = [asdict(entry) for entry in self.long_term]
        with open(filepath, 'w', encoding='utf-8') as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
    
    def add_experience(self, experience: Dict):
        """添加经验到短期记忆"""
        entry = MemoryEntry(
            entry_type="experience",
            content=experience,
            importance=experience.get('quality_score', 0.5),
        )
        self.short_term.append(entry)
        
        # 检查是否需要转入长期记忆
        self._consolidate_memory()
    
    def add_pattern(self, pattern: Dict):
        """添加模式到长期记忆"""
        entry = MemoryEntry(
            entry_type="pattern",
            content=pattern,
            importance=pattern.get('confidence', 0.5),
        )
        self.long_term.append(entry)
        self._save_long_term()
    
    def add_preference(self, key: str, value: Any):
        """添加用户偏好"""
        # 检查是否已存在
        for entry in self.long_term:
            if entry.entry_type == "preference" and entry.content.get('key') == key:
                entry.content['value'] = value
                entry.content['updated_at'] = datetime.now().isoformat()
                self._save_long_term()
                return
        
        entry = MemoryEntry(
            entry_type="preference",
            content={"key": key, "value": value},
            importance=0.7,
        )
        self.long_term.append(entry)
        self._save_long_term()
    
    def _consolidate_memory(self):
        """整理记忆：将重要的短期记忆转入长期记忆"""
        if len(self.short_term) > self.max_short_term:
            # 按重要性排序
            sorted_entries = sorted(
                self.short_term,
                key=lambda e: e.importance,
                reverse=True,
            )
            
            # 将重要的转入长期记忆
            for entry in sorted_entries[:5]:
                if entry.importance > 0.6:
                    self.long_term.append(entry)
            
            # 保留最近的在短期记忆
            self.short_term = sorted_entries[5:]
            
            # 长期记忆也有限制
            if len(self.long_term) > self.max_long_term:
                self.long_term = sorted(
                    self.long_term,
                    key=lambda e: e.importance,
                    reverse=True,
                )[:self.max_long_term]
            
            self._save_long_term()
    
    def get_relevant_memories(self, query: str, memory_type: str = None,
                              limit: int = 10) -> List[MemoryEntry]:
        """获取与查询相关的记忆"""
        relevant = []
        
        all_memories = self.short_term + self.long_term
        
        for entry in all_memories:
            if memory_type and entry.entry_type != memory_type:
                continue
            
            # 简单的相关性检查
            content_str = json.dumps(entry.content, ensure_ascii=False).lower()
            query_lower = query.lower()
            
            # 检查关键词匹配
            query_words = query_lower.split()
            matches = sum(1 for word in query_words if word in content_str)
            
            if matches > 0:
                entry.access_count += 1
                entry.last_accessed = datetime.now().isoformat()
                relevant.append((matches * entry.importance, entry))
        
        # 按相关性排序
        relevant.sort(key=lambda x: x[0], reverse=True)
        
        return [entry for _, entry in relevant[:limit]]


class SelfLearningAgent:
    """自学习 Agent"""
    
    def __init__(self):
        self.client = anthropic.Anthropic(
            api_key=os.environ.get("ANTHROPIC_API_KEY")
        )
        self.memory = SelfLearningMemory("agent_memory")
        self.experience_count = 0
        self.total_quality_score = 0.0
    
    def think_and_act(self, task: str) -> str:
        """思考并执行任务（带学习）"""
        print(f"\n{'='*60}")
        print(f"任务: {task}")
        print(f"{'='*60}")
        
        # 1. 从记忆中获取相关经验
        relevant_memories = self.memory.get_relevant_memories(task, limit=5)
        
        # 2. 构建上下文
        context = self._build_context(task, relevant_memories)
        
        # 3. 调用 LLM 执行任务
        print("\n[思考中...]")
        response = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            system=context,
            messages=[{"role": "user", "content": task}],
        )
        
        result = response.content[0].text
        print(f"\n[执行结果]\n{result}")
        
        # 4. 自我反思
        reflection = self._reflect(task, result, relevant_memories)
        print(f"\n[反思]\n{reflection}")
        
        # 5. 记录经验
        experience = {
            "task": task,
            "result": result,
            "reflection": reflection,
            "quality_score": reflection.get('quality_score', 0.5),
            "timestamp": datetime.now().isoformat(),
        }
        self.memory.add_experience(experience)
        self.experience_count += 1
        self.total_quality_score += reflection.get('quality_score', 0.5)
        
        # 6. 提取模式（每 5 次任务后）
        if self.experience_count % 5 == 0:
            self._extract_and_store_patterns()
        
        return result
    
    def _build_context(self, task: str, memories: List[MemoryEntry]) -> str:
        """构建上下文"""
        context = """你是一个自学习助手。你可以从历史经验中学习，不断改进自己的表现。

你的能力：
1. 分析任务并给出解决方案
2. 从历史经验中学习
3. 适应用户的偏好
4. 持续改进自己的表现
"""
        
        if memories:
            context += "\n\n相关的历史经验：\n"
            for mem in memories:
                if mem.entry_type == "experience":
                    context += f"- 任务: {mem.content.get('task', 'N/A')[:50]}...\n"
                    context += f"  结果: {mem.content.get('result', 'N/A')[:50]}...\n"
                elif mem.entry_type == "pattern":
                    context += f"- 模式: {mem.content.get('description', 'N/A')}\n"
                elif mem.entry_type == "preference":
                    context += f"- 用户偏好: {mem.content.get('key')}: {mem.content.get('value')}\n"
        
        return context
    
    def _reflect(self, task: str, result: str, memories: List[MemoryEntry]) -> Dict:
        """自我反思"""
        reflection_prompt = f"""请对以下任务执行进行反思：

任务：{task}
结果：{result[:200]}...

请评估：
1. 任务是否成功完成？（是/部分/否）
2. 结果的质量如何？（0-1 分）
3. 有哪些可以改进的地方？
4. 下次遇到类似任务应该怎么做？

请用 JSON 格式回答，包含以下字段：
- success: true/false/partial
- quality_score: 0-1 的浮点数
- strengths: 优点列表
- improvements: 改进点列表
- strategy_for_next_time: 下次的策略
"""
        
        response = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            messages=[{"role": "user", "content": reflection_prompt}],
        )
        
        try:
            # 尝试解析 JSON
            reflection_text = response.content[0].text
            # 提取 JSON 部分
            start = reflection_text.find('{')
            end = reflection_text.rfind('}') + 1
            if start >= 0 and end > start:
                return json.loads(reflection_text[start:end])
        except json.JSONDecodeError:
            pass
        
        return {
            "success": True,
            "quality_score": 0.5,
            "strengths": [],
            "improvements": [],
            "strategy_for_next_time": "",
        }
    
    def _extract_and_store_patterns(self):
        """提取并存储模式"""
        print("\n[提取模式...]")
        
        experiences = self.memory.get_relevant_memories("", memory_type="experience", limit=10)
        
        if len(experiences) < 3:
            return
        
        # 分析共同的成功策略
        successful_tasks = []
        for exp in experiences:
            if exp.content.get('reflection', {}).get('success'):
                successful_tasks.append(exp.content)
        
        if len(successful_tasks) >= 2:
            # 提取模式
            pattern = {
                "type": "success_strategy",
                "description": f"基于 {len(successful_tasks)} 次成功经验的模式",
                "tasks": [t.get('task', '')[:50] for t in successful_tasks],
                "strategies": [t.get('reflection', {}).get('strategy_for_next_time', '') for t in successful_tasks],
                "confidence": min(0.8, len(successful_tasks) * 0.15),
            }
            
            self.memory.add_pattern(pattern)
            print(f"  新模式: {pattern['description']}")
    
    def get_performance_report(self) -> Dict:
        """获取性能报告"""
        return {
            "total_tasks": self.experience_count,
            "average_quality": self.total_quality_score / max(1, self.experience_count),
            "memories": {
                "short_term": len(self.memory.short_term),
                "long_term": len(self.memory.long_term),
            },
        }


def demo_self_learning_agent():
    """演示自学习 Agent"""
    print("=" * 60)
    print("自学习 Agent 演示")
    print("=" * 60)
    
    agent = SelfLearningAgent()
    
    # 模拟一系列任务
    tasks = [
        "请帮我写一个 Python 函数，计算斐波那契数列的第 n 项",
        "请解释什么是机器学习中的过拟合，以及如何避免",
        "请帮我优化这个 SQL 查询，提高查询性能",
        "请帮我设计一个简单的 REST API，用于用户管理",
        "请帮我分析这段代码的性能问题",
    ]
    
    for task in tasks:
        agent.think_and_act(task)
    
    # 显示性能报告
    print("\n" + "=" * 60)
    print("性能报告")
    print("=" * 60)
    
    report = agent.get_performance_report()
    print(f"总任务数: {report['total_tasks']}")
    print(f"平均质量分数: {report['average_quality']:.2f}")
    print(f"短期记忆: {report['memories']['short_term']} 条")
    print(f"长期记忆: {report['memories']['long_term']} 条")


if __name__ == "__main__":
    demo_self_learning_agent()
```

### 示例 4：反思引擎

```python
"""
反思引擎
对任务执行过程进行深度反思
"""

import json
from typing import Dict, List
from datetime import datetime
import anthropic


class ReflectionEngine:
    """反思引擎"""
    
    def __init__(self):
        self.client = anthropic.Anthropic(
            api_key=os.environ.get("ANTHROPIC_API_KEY")
        )
    
    def reflect_on_task(self, task: str, steps: List[Dict],
                        result: str, success: bool) -> Dict:
        """对任务执行进行反思"""
        
        steps_description = "\n".join([
            f"步骤 {i+1}: {step.get('action', 'N/A')}"
            f" (工具: {step.get('tool_used', 'N/A')}, "
            f"成功: {step.get('success', True)})"
            for i, step in enumerate(steps)
        ])
        
        reflection_prompt = f"""请对以下任务执行进行深度反思。

任务：{task}

执行步骤：
{steps_description}

最终结果：{result[:300]}
任务成功：{success}

请从以下维度进行反思：

1. **目标达成度**：任务是否完全达成了用户的目标？
2. **过程效率**：执行过程是否高效？有没有冗余步骤？
3. **工具使用**：是否选择了合适的工具？工具使用是否正确？
4. **错误分析**：如果失败了，根本原因是什么？
5. **改进方向**：下次遇到类似任务，应该如何改进？

请用 JSON 格式回答：
{{
    "goal_achievement": {{
        "score": 0-1,
        "analysis": "分析"
    }},
    "process_efficiency": {{
        "score": 0-1,
        "redundant_steps": ["冗余步骤列表"],
        "optimization_suggestions": ["优化建议"]
    }},
    "tool_usage": {{
        "score": 0-1,
        "appropriate_tools": ["合适的工具"],
        "inappropriate_tools": ["不合适的工具"],
        "missing_tools": ["缺少的工具"]
    }},
    "error_analysis": {{
        "has_error": true/false,
        "root_cause": "根本原因",
        "prevention_measures": ["预防措施"]
    }},
    "improvement_directions": [
        {{
            "area": "改进领域",
            "suggestion": "具体建议",
            "priority": "high/medium/low"
        }}
    ],
    "overall_quality": 0-1,
    "lessons_learned": ["经验教训"]
}}
"""
        
        response = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=2048,
            messages=[{"role": "user", "content": reflection_prompt}],
        )
        
        try:
            text = response.content[0].text
            start = text.find('{')
            end = text.rfind('}') + 1
            if start >= 0 and end > start:
                return json.loads(text[start:end])
        except json.JSONDecodeError:
            pass
        
        return {"overall_quality": 0.5, "lessons_learned": []}
    
    def generate_improvement_plan(self, reflections: List[Dict]) -> Dict:
        """根据多次反思生成改进计划"""
        
        reflections_text = json.dumps(reflections[-5:], ensure_ascii=False, indent=2)
        
        prompt = f"""根据以下最近的反思记录，生成一个改进计划。

反思记录：
{reflections_text}

请生成一个具体的改进计划，包括：
1. 需要优先改进的领域
2. 具体的改进措施
3. 衡量改进效果的指标
4. 实施时间表

请用 JSON 格式回答。"""
        
        response = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=2048,
            messages=[{"role": "user", "content": prompt}],
        )
        
        try:
            text = response.content[0].text
            start = text.find('{')
            end = text.rfind('}') + 1
            if start >= 0 and end > start:
                return json.loads(text[start:end])
        except json.JSONDecodeError:
            pass
        
        return {"priority_areas": [], "measures": []}


def demo_reflection_engine():
    """演示反思引擎"""
    print("=" * 60)
    print("反思引擎演示")
    print("=" * 60)
    
    engine = ReflectionEngine()
    
    # 模拟任务执行
    task = "分析销售数据并生成报告"
    steps = [
        {"action": "读取数据文件", "tool_used": "pandas", "success": True},
        {"action": "数据清洗", "tool_used": "pandas", "success": True},
        {"action": "统计分析", "tool_used": "numpy", "success": True},
        {"action": "生成图表", "tool_used": "matplotlib", "success": False},
        {"action": "生成文本报告", "tool_used": "jinja2", "success": True},
    ]
    result = "完成数据分析，生成了文本报告。图表生成失败。"
    
    # 进行反思
    print("\n[反思任务执行...]")
    reflection = engine.reflect_on_task(task, steps, result, True)
    
    print(f"\n反思结果:")
    print(json.dumps(reflection, ensure_ascii=False, indent=2))
    
    # 生成改进计划
    print("\n[生成改进计划...]")
    plan = engine.generate_improvement_plan([reflection])
    
    print(f"\n改进计划:")
    print(json.dumps(plan, ensure_ascii=False, indent=2))


if __name__ == "__main__":
    demo_reflection_engine()
```

---

## 案例分析

### 案例 1：客服 Agent 的自学习

**场景**：一个客服 Agent 需要处理各种客户问题，并从每次交互中学习。

**学习循环设计**：

1. **任务类型识别**：Agent 首先识别客户问题的类型（咨询、投诉、技术支持等）
2. **解决方案执行**：根据问题类型选择解决方案
3. **客户反馈收集**：收集客户的满意度评价
4. **经验记录**：记录整个交互过程
5. **模式提取**：从成功和失败的案例中提取模式
6. **策略更新**：更新客服策略

**关键学习点**：

- **成功模式**："当客户抱怨产品质量时，先表达理解，然后提供具体解决方案"的成功率是 85%
- **失败模式**："直接给出技术方案而不表达理解"会导致客户不满
- **用户偏好**："客户更喜欢简洁的回答，避免过多的技术术语"

### 案例 2：代码审查 Agent 的进化

**场景**：一个代码审查 Agent 需要不断学习新的代码模式和最佳实践。

**学习机制**：

1. **规则库积累**：每次审查都会发现新的代码问题和最佳实践
2. **误报学习**：如果开发者忽略了某个建议（认为是误报），Agent 会降低该规则的权重
3. **模式识别**：识别项目特定的代码模式和规范
4. **适应性调整**：根据不同团队的偏好调整审查标准

**效果指标**：
- 误报率从初始的 30% 下降到 10%
- 漏报率从 20% 下降到 5%
- 开发者满意度从 60% 提升到 85%

---

## 常见坑

### 坑 1：过度拟合历史经验

Agent 可能过度依赖历史经验，无法处理新的情况。

**解决方案**：保持探索性，为新的策略留出空间。

```python
def select_strategy(self, task_type: str) -> str:
    """选择策略"""
    # 80% 使用已知策略，20% 尝试新策略
    if random.random() < 0.8:
        return self.get_best_known_strategy(task_type)
    else:
        return self.explore_new_strategy(task_type)
```

### 坑 2：反思质量不足

自动反思的质量可能不如人工反思。

**解决方案**：结合多种反思方式，包括规则检查和 LLM 反思。

### 坑 3：记忆膨胀

随着时间推移，记忆库可能变得过大，影响检索效率。

**解决方案**：实现记忆压缩和遗忘机制。

```python
def compress_memory(self):
    """压缩记忆"""
    # 合并相似的记忆
    # 删除不重要的记忆
    # 对记忆进行摘要
    pass
```

### 坑 4：学习偏差

如果训练数据有偏差，Agent 可能学到错误的模式。

**解决方案**：实现多样性检查和偏见检测。

### 坑 5：评估困难

如何客观评估 Agent 的学习效果是一个挑战。

**解决方案**：建立多维度的评估体系，包括定量指标和定性评价。

---

## 练习题

### 练习 1：经验记录系统
实现一个经验记录系统，能够：
- 记录任务执行的每个步骤
- 自动评估任务成功与否
- 保存执行历史供后续分析

### 练习 2：模式提取器
实现一个模式提取器，能够：
- 从历史经验中提取成功模式
- 识别常见的失败原因
- 生成可复用的策略建议

### 练习 3：反思引擎
实现一个反思引擎，能够：
- 分析任务执行过程
- 识别改进点
- 生成具体的改进建议

### 练习 4：记忆管理系统
实现一个完整的记忆管理系统，包括：
- 短期记忆和长期记忆
- 记忆的存储和检索
- 记忆的压缩和遗忘

### 练习 5：自学习 Agent
将上述组件组合成一个完整的自学习 Agent，实现：
- 经验记录
- 模式提取
- 反思评估
- 策略适配

### 练习 6：学习效果评估
设计一个评估系统，能够：
- 量化 Agent 的学习进度
- 对比不同学习策略的效果
- 生成学习报告

---

## 实战任务

### 任务 1：构建一个自学习的问答 Agent（中等难度）

**功能需求：**
1. 回答用户的问题
2. 记录问答历史
3. 从成功的回答中学习模式
4. 根据用户反馈调整回答风格

**技术要求：**
1. 实现经验记录系统
2. 实现基础的模式提取
3. 实现反思机制
4. 支持至少 5 轮的学习循环

### 任务 2：构建一个进化的代码助手（高难度）

**功能需求：**
1. 帮助用户编写代码
2. 记录代码生成过程
3. 学习用户的编码风格
4. 随着使用时间增长，代码质量不断提高

**技术要求：**
1. 实现完整的自学习循环
2. 支持代码质量评估
3. 实现模式库管理
4. 支持多轮学习和进化

---

## 本章小结

本章我们深入学习了自学习 Agent 的概念和实现。这是 Agent 领域的一个前沿方向，它让 Agent 具备了从经验中学习和改进的能力。

**核心概念**：理解了自学习 Agent 的核心理念——不仅要完成任务，还要从任务执行过程中提取经验，不断改进自身行为。

**学习循环**：掌握了学习循环的六个阶段：任务执行、经验记录、反思评估、模式提取、知识更新、行为调整。这个循环是自学习 Agent 的核心机制。

**Hermes 架构**：了解了 Hermes 框架的核心组件：经验记忆、模式库、反思引擎、模式提取器、策略适配器。

**经验表示**：学会了如何表示和存储经验，包括任务描述、执行过程、结果评价和反思内容。

**模式提取**：掌握了从历史经验中提取成功模式、失败模式和效率模式的方法。

**反思机制**：理解了反思在自学习中的关键作用，包括目标达成评估、过程效率分析、错误根因分析和改进方向规划。

自学习 Agent 代表了 Agent 技术的一个重要发展方向。虽然目前的实现还比较简单，但随着 LLM 能力的提升和学习算法的改进，自学习 Agent 将会变得越来越强大。

在下一章中，我们将学习 Harness Engineering，看看如何系统化地构建和管理 Agent 应用。

---

## 延伸阅读

1. Learning to Reason with LLMs (OpenAI Blog)
2. Reflexion: Language Agents with Verbal Reinforcement Learning (论文)
3. Self-Refine: Iterative Refinement with Self-Feedback (论文)
4. Hermes: A Self-Learning Agent Framework (研究项目)
5. Voyager: An Open-Ended Embodied Agent with Large Language Models (论文)
