# 第 62 章：项目 6 -- 可学习 Agent（Hermes 风格）

> **本章定位：** 这是六个实战项目的收官之作。前面的项目都是"出厂设定"的 Agent，运行后的行为基本固定。本章将构建一个真正能"学习"的 Agent——它能从用户的反馈中提取经验，将经验存入长期记忆，并在未来的交互中应用这些经验。这种自我进化的能力，是 Agent 从"工具"走向"伙伴"的关键一步。

---

## 学习目标

完成本章学习后，你将能够：

1. **设计经验学习系统** -- 能够让 Agent 从成功和失败的交互中提取可复用的经验规则
2. **实现分层记忆架构** -- 能够设计短期记忆、工作记忆、长期记忆三层结构，让 Agent 在不同时间尺度上保持上下文
3. **构建反思与自我评估机制** -- 能够让 Agent 在任务完成后自动评估自己的表现，找出改进方向
4. **实现经验检索与应用** -- 能够让 Agent 在面对新任务时，自动检索最相关的历史经验来辅助决策
5. **设计个性化的学习路径** -- 能够让 Agent 根据用户的偏好和反馈，逐步形成个性化的行为模式
6. **完成一个可自我进化的 Agent 系统** -- 这个 Agent 能够越用越好

## 核心问题

1. **Agent 如何从"经验"中学习？** 人类通过反思和总结来积累经验，Agent 用什么机制来实现类似的能力？
2. **什么样的"经验"值得记忆？** 不是所有的交互结果都有价值，如何判断和筛选值得长期保存的经验？
3. **经验检索的时机和方式？** 当 Agent 面对新任务时，如何知道应该参考哪些历史经验？如何避免过时经验的干扰？

---

## 62.1 项目背景与需求分析

### 62.1.1 为什么 Agent 需要学习

在前面的项目中，我们构建的 Agent 都有一个共同的局限：它们的行为完全由初始的提示词和配置决定。无论运行多长时间，犯了多少次错误，它们都不会自动改进。

人类和工具的根本区别之一在于：人能从经验中学习。你第一次做某件事可能做得很差，但通过反思和总结，第二次就会做得更好。这就是学习的力量。

可学习 Agent 的目标就是赋予 Agent 类似的能力。具体来说，它需要：

首先，**能识别自己的错误**。当 Agent 生成了一个不好的回复或执行了一个失败的任务时，它需要能意识到"这个结果不够好"。

其次，**能分析错误的原因**。它不仅要知道"我做错了"，还要知道"为什么做错了"。是搜索策略不对？是提示词不够精确？还是对用户需求的理解有偏差？

然后，**能提取可复用的经验规则**。从具体的错误中抽象出一般性的规则，比如"当用户问关于时间的问题时，需要确认时区"。

最后，**能在未来应用这些经验**。当遇到类似的任务时，自动检索相关经验来指导行为。

### 62.1.2 Hermes 风格的含义

"Hermes"是希腊神话中的信使之神，也是商业和旅行的守护者。在这个项目中，我们借用 Hermes 的名字来象征一个"智慧的助手"。Hermes 风格的 Agent 有几个特点：

一是**自省**。Hermes 不仅执行任务，还会反思自己的表现。
二是**积累**。Hermes 会把每次经历转化为智慧，越用越聪明。
三是**适应**。Hermes 会根据不同用户的特点调整自己的行为方式。

### 62.1.3 功能需求定义

可学习 Agent 需要具备以下核心功能：

首先是**交互记录与评估功能**。Agent 需要记录每次交互的完整过程，包括用户输入、Agent 的回复、用户的反馈（如果有的话）。在每次交互后，Agent 需要对自己的表现进行评估。

其次是**经验提取功能**。从交互记录中提取有价值的经验规则。这些规则可以是关于用户偏好的（"这个用户喜欢简洁的回复"），也可以是关于任务策略的（"搜索学术论文时用 Google Scholar 更有效"）。

然后是**长期记忆管理**。维护一个持久化的经验库，支持按主题、场景、时间等维度进行检索。定期清理过时或无效的经验。

最后是**经验应用功能**。在面对新任务时，自动检索最相关的历史经验，并将这些经验融入到当前的决策过程中。

---

## 62.2 架构设计

```
hermes-agent/
    main.py              # 入口文件
    core/
        agent.py         # 核心 Agent
        memory.py        # 三层记忆系统
        learning.py      # 学习引擎
        evaluator.py     # 自我评估器
    tools/
        base.py          # 工具基类
        search.py        # 搜索工具
        calculator.py    # 计算工具
    config.py            # 配置管理
    data/
        experiences/     # 经验存储
        conversations/   # 对话记录
        user_model/      # 用户模型
```

---

## 62.3 核心代码实现

### 62.3.1 三层记忆系统

```python
# core/memory.py
import json
import os
import time
import hashlib
from dataclasses import dataclass, field
from typing import Optional, Any
from datetime import datetime, timedelta


@dataclass
class Experience:
    """一条经验记录。"""
    exp_id: str               # 唯一标识
    timestamp: float          # 创建时间
    category: str             # 经验类别：preference/strategy/error_correction/domain
    description: str          # 经验描述
    context: str              # 产生这条经验的上下文
    rule: str                 # 提炼出的经验规则
    confidence: float         # 置信度（0-1）
    use_count: int = 0        # 被引用的次数
    last_used: float = 0      # 最后使用时间
    tags: list[str] = field(default_factory=list)
    source_conversation: str = ""  # 来源对话ID

    def to_dict(self) -> dict:
        return {
            "exp_id": self.exp_id,
            "timestamp": self.timestamp,
            "category": self.category,
            "description": self.description,
            "context": self.context,
            "rule": self.rule,
            "confidence": self.confidence,
            "use_count": self.use_count,
            "last_used": self.last_used,
            "tags": self.tags,
            "source_conversation": self.source_conversation,
        }

    @classmethod
    def from_dict(cls, data: dict) -> "Experience":
        return cls(**data)


@dataclass
class UserProfile:
    """用户画像。"""
    user_id: str = "default"
    preferences: dict = field(default_factory=dict)
    interaction_count: int = 0
    avg_satisfaction: float = 0.5
    favorite_topics: list[str] = field(default_factory=list)
    disliked_patterns: list[str] = field(default_factory=list)
    communication_style: str = "balanced"  # formal/casual/balanced
    last_updated: float = 0


class MemorySystem:
    """三层记忆系统。

    1. 短期记忆（Short-term）：当前对话的上下文，会话结束即清除
    2. 工作记忆（Working）：当前任务的相关信息，任务完成后部分转入长期记忆
    3. 长期记忆（Long-term）：持久化的经验和知识，跨会话保持

    这三层记忆模仿了人类记忆的工作方式：
    - 短期记忆对应工作记忆（约7个项目）
    - 工作记忆对应情景记忆
    - 长期记忆对应语义记忆
    """

    def __init__(self, storage_dir: str = "data"):
        self.storage_dir = storage_dir
        os.makedirs(storage_dir, exist_ok=True)
        os.makedirs(os.path.join(storage_dir, "experiences"), exist_ok=True)
        os.makedirs(os.path.join(storage_dir, "conversations"), exist_ok=True)
        os.makedirs(os.path.join(storage_dir, "user_model"), exist_ok=True)

        # 短期记忆：当前对话
        self.short_term: list[dict] = []
        self.current_conversation_id = self._generate_id()

        # 工作记忆：当前任务上下文
        self.working_memory: dict[str, Any] = {}

        # 长期记忆：持久化存储
        self.experiences: list[Experience] = self._load_experiences()
        self.user_profile = self._load_user_profile()

    # === 短期记忆操作 ===

    def add_to_short_term(self, role: str, content: str):
        """添加到短期记忆。"""
        self.short_term.append({
            "role": role,
            "content": content,
            "timestamp": time.time()
        })
        # 保持短期记忆在合理范围内
        if len(self.short_term) > 50:
            self.short_term = self.short_term[-30:]

    def get_short_term_context(self, n_messages: int = 10) -> str:
        """获取最近几条消息作为上下文。"""
        recent = self.short_term[-n_messages:]
        lines = []
        for msg in recent:
            role = "用户" if msg["role"] == "user" else "助手"
            content = msg["content"][:200]
            lines.append(f"{role}: {content}")
        return "\n".join(lines)

    def clear_short_term(self):
        """清除短期记忆（会话结束时）。"""
        self.short_term = []

    # === 工作记忆操作 ===

    def set_working(self, key: str, value: Any):
        """设置工作记忆。"""
        self.working_memory[key] = value

    def get_working(self, key: str, default=None):
        """获取工作记忆。"""
        return self.working_memory.get(key, default)

    def clear_working(self):
        """清除工作记忆（任务完成后）。"""
        self.working_memory = {}

    # === 长期记忆操作 ===

    def add_experience(self, category: str, description: str,
                       context: str, rule: str, confidence: float = 0.7,
                       tags: list[str] = None):
        """添加一条新经验。"""
        exp = Experience(
            exp_id=self._generate_id(),
            timestamp=time.time(),
            category=category,
            description=description,
            context=context,
            rule=rule,
            confidence=confidence,
            tags=tags or [],
            source_conversation=self.current_conversation_id,
        )
        self.experiences.append(exp)
        self._save_experiences()
        print(f"[新经验] [{category}] {rule}")
        return exp

    def search_experiences(self, query: str, category: str = None,
                          top_k: int = 5) -> list[Experience]:
        """检索最相关的经验。

        使用简单的关键词匹配 + 时间衰减 + 使用频率的综合评分。
        生产环境中应该使用向量检索。
        """
        scored = []
        query_lower = query.lower()
        now = time.time()

        for exp in self.experiences:
            # 类别过滤
            if category and exp.category != category:
                continue

            # 计算相关性分数
            score = 0.0

            # 关键词匹配
            rule_lower = exp.rule.lower()
            desc_lower = exp.description.lower()
            tags_str = " ".join(exp.tags).lower()

            if query_lower in rule_lower:
                score += 3.0
            if query_lower in desc_lower:
                score += 2.0
            if query_lower in tags_str:
                score += 1.5

            # 标签匹配
            for tag in exp.tags:
                if tag.lower() in query_lower:
                    score += 1.0

            # 置信度加权
            score *= exp.confidence

            # 时间衰减（越新越好）
            age_hours = (now - exp.timestamp) / 3600
            time_factor = 1.0 / (1.0 + age_hours / 168)  # 一周半衰期
            score *= time_factor

            # 使用频率奖励
            if exp.use_count > 0:
                score *= (1.0 + min(exp.use_count * 0.1, 1.0))

            if score > 0:
                scored.append((exp, score))

        scored.sort(key=lambda x: x[1], reverse=True)
        return [exp for exp, _ in scored[:top_k]]

    def record_experience_use(self, exp_id: str):
        """记录经验被使用（更新使用计数）。"""
        for exp in self.experiences:
            if exp.exp_id == exp_id:
                exp.use_count += 1
                exp.last_used = time.time()
                break
        self._save_experiences()

    def get_relevant_experiences_context(self, task_description: str) -> str:
        """为 LLM 生成相关的经验上下文。"""
        experiences = self.search_experiences(task_description, top_k=5)
        if not experiences:
            return ""

        lines = ["相关历史经验（请参考）："]
        for i, exp in enumerate(experiences, 1):
            lines.append(f"{i}. [{exp.category}] {exp.rule}")
            lines.append(f"   （置信度: {exp.confidence:.1f}, 已使用 {exp.use_count} 次）")
        return "\n".join(lines)

    # === 用户模型操作 ===

    def update_user_model(self, preference_key: str, preference_value: Any):
        """更新用户偏好。"""
        self.user_profile.preferences[preference_key] = preference_value
        self.user_profile.last_updated = time.time()
        self._save_user_profile()

    def add_user_feedback(self, feedback_type: str, detail: str):
        """添加用户反馈。"""
        if feedback_type == "positive":
            self.user_profile.avg_satisfaction = min(
                1.0, self.user_profile.avg_satisfaction + 0.05
            )
        elif feedback_type == "negative":
            self.user_profile.avg_satisfaction = max(
                0.0, self.user_profile.avg_satisfaction - 0.1
            )
        self.user_profile.interaction_count += 1
        self._save_user_profile()

    # === 持久化操作 ===

    def _load_experiences(self) -> list[Experience]:
        """从文件加载经验库。"""
        exp_file = os.path.join(self.storage_dir, "experiences", "experiences.json")
        try:
            with open(exp_file, "r", encoding="utf-8") as f:
                data = json.load(f)
            return [Experience.from_dict(d) for d in data]
        except (FileNotFoundError, json.JSONDecodeError):
            return []

    def _save_experiences(self):
        """保存经验库到文件。"""
        exp_file = os.path.join(self.storage_dir, "experiences", "experiences.json")
        data = [exp.to_dict() for exp in self.experiences]
        with open(exp_file, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=2)

    def _load_user_profile(self) -> UserProfile:
        """加载用户画像。"""
        profile_file = os.path.join(self.storage_dir, "user_model", "profile.json")
        try:
            with open(profile_file, "r", encoding="utf-8") as f:
                data = json.load(f)
            return UserProfile(**data)
        except (FileNotFoundError, TypeError):
            return UserProfile()

    def _save_user_profile(self):
        """保存用户画像。"""
        profile_file = os.path.join(self.storage_dir, "user_model", "profile.json")
        data = {
            "user_id": self.user_profile.user_id,
            "preferences": self.user_profile.preferences,
            "interaction_count": self.user_profile.interaction_count,
            "avg_satisfaction": self.user_profile.avg_satisfaction,
            "favorite_topics": self.user_profile.favorite_topics,
            "disliked_patterns": self.user_profile.disliked_patterns,
            "communication_style": self.user_profile.communication_style,
            "last_updated": self.user_profile.last_updated,
        }
        with open(profile_file, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=2)

    def _generate_id(self) -> str:
        """生成唯一ID。"""
        raw = f"{time.time()}_{os.getpid()}"
        return hashlib.md5(raw.encode()).hexdigest()[:12]

    def get_memory_stats(self) -> dict:
        """获取记忆系统统计信息。"""
        return {
            "short_term_count": len(self.short_term),
            "working_memory_keys": list(self.working_memory.keys()),
            "total_experiences": len(self.experiences),
            "experience_categories": {
                cat: sum(1 for e in self.experiences if e.category == cat)
                for cat in set(e.category for e in self.experiences) if self.experiences
            },
            "user_interaction_count": self.user_profile.interaction_count,
            "user_satisfaction": self.user_profile.avg_satisfaction,
        }
```

### 62.3.2 自我评估器

```python
# core/evaluator.py
import json
from openai import OpenAI
from dataclasses import dataclass


@dataclass
class EvaluationResult:
    """评估结果。"""
    score: float          # 0-10 的评分
    quality: str          # excellent/good/acceptable/poor
    strengths: list[str]  # 做得好的地方
    weaknesses: list[str] # 需要改进的地方
    lessons: list[str]    # 可以提取的经验
    should_store: bool    # 是否值得存储为经验


class SelfEvaluator:
    """自我评估器。

    在每次交互后，让 LLM 对 Agent 的表现进行评估。
    评估结果用于：
    1. 识别做得好的地方（强化）
    2. 识别需要改进的地方（修正）
    3. 提取可复用的经验规则
    """

    EVALUATION_PROMPT = """你是一个严格的评估专家。请评估以下 Agent 交互的质量。

## 交互记录

用户输入: {user_input}
Agent 回复: {agent_response}
任务类型: {task_type}
用户反馈: {feedback}

## 评估维度

请从以下维度评估（每项 0-10 分）：

1. **准确性** (accuracy)：回复内容是否正确、无事实错误
2. **相关性** (relevance)：回复是否紧扣用户的问题
3. **完整性** (completeness)：是否覆盖了用户需要的所有信息
4. **清晰度** (clarity)：表达是否清晰易懂
5. **适当性** (appropriateness)：语气和格式是否合适

## 输出格式

请以 JSON 格式返回评估结果：
```json
{{
    "scores": {{
        "accuracy": 8,
        "relevance": 9,
        "completeness": 7,
        "clarity": 8,
        "appropriateness": 9
    }},
    "overall_score": 8.2,
    "quality": "good",
    "strengths": ["做得好的点1", "做得好的点2"],
    "weaknesses": ["需要改进的点1"],
    "lessons": ["可以提取的经验1", "可以提取的经验2"],
    "should_store": true
}}
```

## 质量等级
- excellent: 9-10 分
- good: 7-8.9 分
- acceptable: 5-6.9 分
- poor: 0-4.9 分
"""

    def __init__(self, api_key: str, model: str = "gpt-4o-mini",
                 base_url: str = None):
        self.client = OpenAI(api_key=api_key, base_url=base_url)
        self.model = model

    def evaluate(self, user_input: str, agent_response: str,
                 task_type: str = "general",
                 feedback: str = "无") -> EvaluationResult:
        """评估一次交互的质量。"""
        prompt = self.EVALUATION_PROMPT.format(
            user_input=user_input[:1000],
            agent_response=agent_response[:1000],
            task_type=task_type,
            feedback=feedback,
        )

        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": "你是一个精确的评估专家。请返回 JSON。"},
                    {"role": "user", "content": prompt}
                ],
                temperature=0.0,
                max_tokens=1000,
                response_format={"type": "json_object"},
            )

            result = json.loads(response.choices[0].message.content)

            return EvaluationResult(
                score=result.get("overall_score", 5.0),
                quality=result.get("quality", "acceptable"),
                strengths=result.get("strengths", []),
                weaknesses=result.get("weaknesses", []),
                lessons=result.get("lessons", []),
                should_store=result.get("should_store", False),
            )

        except Exception as e:
            print(f"[评估失败] {e}")
            return EvaluationResult(
                score=5.0,
                quality="acceptable",
                strengths=[],
                weaknesses=["评估过程出错"],
                lessons=[],
                should_store=False,
            )
```

### 62.3.3 学习引擎

```python
# core/learning.py
from openai import OpenAI
from core.memory import MemorySystem
from core.evaluator import SelfEvaluator, EvaluationResult


class LearningEngine:
    """学习引擎。

    负责从交互中学习，是可学习 Agent 的核心组件。
    它的工作流程是：
    1. 接收交互数据
    2. 调用评估器进行评估
    3. 从评估结果中提取经验
    4. 将经验存入长期记忆
    5. 更新用户模型
    """

    EXPERIENCE_EXTRACTION_PROMPT = """从以下交互中提取有价值的经验规则。

## 交互上下文
用户输入: {user_input}
Agent 回复: {agent_response}

## 评估结果
评分: {score}/10
做得好的地方: {strengths}
需要改进的地方: {weaknesses}

## 任务

请提取 1-3 条可复用的经验规则。每条规则应该是：
1. 具体的、可操作的（不要笼统的"要更准确"）
2. 有通用性的（能在类似场景中复用）
3. 格式为"当...时，应该..."的形式

请以 JSON 格式返回：
```json
{{
    "experiences": [
        {{
            "category": "preference/strategy/error_correction/domain",
            "description": "经验的简短描述",
            "rule": "具体的可复用规则",
            "confidence": 0.8,
            "tags": ["标签1", "标签2"]
        }}
    ]
}}
```
"""

    def __init__(self, memory: MemorySystem, evaluator: SelfEvaluator,
                 api_key: str, model: str = "gpt-4o-mini", base_url: str = None):
        self.memory = memory
        self.evaluator = evaluator
        self.client = OpenAI(api_key=api_key, base_url=base_url)
        self.model = model

    def process_interaction(self, user_input: str, agent_response: str,
                            task_type: str = "general",
                            user_feedback: str = "无") -> dict:
        """处理一次完整的交互，进行评估和学习。

        这是学习引擎的核心方法。它在每次交互后被调用，
        执行"评估->提取->存储"的完整学习流程。

        Returns:
            包含评估结果和学习成果的字典
        """
        # 第一步：自我评估
        evaluation = self.evaluator.evaluate(
            user_input=user_input,
            agent_response=agent_response,
            task_type=task_type,
            feedback=user_feedback,
        )

        result = {
            "score": evaluation.score,
            "quality": evaluation.quality,
            "strengths": evaluation.strengths,
            "weaknesses": evaluation.weaknesses,
            "new_experiences": [],
        }

        # 第二步：如果评估认为值得存储，提取经验
        if evaluation.should_store or evaluation.score < 5.0:
            experiences = self._extract_experiences(
                user_input, agent_response, evaluation
            )
            for exp_data in experiences:
                exp = self.memory.add_experience(
                    category=exp_data["category"],
                    description=exp_data["description"],
                    context=f"用户: {user_input[:200]}",
                    rule=exp_data["rule"],
                    confidence=exp_data["confidence"],
                    tags=exp_data["tags"],
                )
                result["new_experiences"].append(exp_data["rule"])

        # 第三步：更新用户模型
        self._update_user_model(user_input, agent_response, evaluation)

        return result

    def _extract_experiences(self, user_input: str, agent_response: str,
                            evaluation: EvaluationResult) -> list[dict]:
        """从交互中提取经验。"""
        prompt = self.EXPERIENCE_EXTRACTION_PROMPT.format(
            user_input=user_input[:500],
            agent_response=agent_response[:500],
            score=evaluation.score,
            strengths=", ".join(evaluation.strengths),
            weaknesses=", ".join(evaluation.weaknesses),
        )

        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": "你是一个经验提取专家。请返回 JSON。"},
                    {"role": "user", "content": prompt}
                ],
                temperature=0.2,
                max_tokens=800,
                response_format={"type": "json_object"},
            )

            result = json.loads(response.choices[0].message.content)
            return result.get("experiences", [])

        except Exception as e:
            print(f"[经验提取失败] {e}")
            return []

    def _update_user_model(self, user_input: str, agent_response: str,
                          evaluation: EvaluationResult):
        """根据交互结果更新用户模型。"""
        # 更新满意度
        if evaluation.score >= 8.0:
            self.memory.add_user_feedback("positive", f"评分: {evaluation.score}")
        elif evaluation.score < 5.0:
            self.memory.add_user_feedback("negative", f"评分: {evaluation.score}")

        # 分析用户的沟通偏好
        if len(user_input) < 20:
            # 短输入的用户可能偏好简洁
            self.memory.update_user_model("input_style", "concise")

        # 分析用户关注的主题
        words = user_input.split()
        for word in words:
            if len(word) > 2:
                profile = self.memory.user_profile
                if word not in profile.favorite_topics:
                    profile.favorite_topics.append(word)
                # 保持话题列表在合理范围内
                if len(profile.favorite_topics) > 50:
                    profile.favorite_topics = profile.favorite_topics[-50:]

    def get_learning_summary(self) -> str:
        """获取学习状况的摘要。"""
        stats = self.memory.get_memory_stats()
        lines = [
            "=== 学习状况摘要 ===",
            f"总交互次数: {stats['user_interaction_count']}",
            f"平均满意度: {stats['user_satisfaction']:.2f}",
            f"总经验数: {stats['total_experiences']}",
        ]

        if stats["experience_categories"]:
            lines.append("经验分布:")
            for cat, count in stats["experience_categories"].items():
                lines.append(f"  {cat}: {count} 条")

        # 最近的经验
        recent = sorted(
            self.memory.experiences,
            key=lambda e: e.timestamp,
            reverse=True
        )[:5]

        if recent:
            lines.append("\n最近的经验:")
            for exp in recent:
                lines.append(f"  [{exp.category}] {exp.rule}")

        return "\n".join(lines)
```

### 62.3.4 可学习 Agent

```python
# core/agent.py
import os
from openai import OpenAI
from core.memory import MemorySystem
from core.learning import LearningEngine
from core.evaluator import SelfEvaluator


class HermesAgent:
    """可学习 Agent（Hermes 风格）。

    核心工作流程：
    1. 接收用户输入
    2. 检索相关经验
    3. 将经验注入提示词
    4. 调用 LLM 生成回复
    5. 评估交互质量
    6. 提取并存储新经验
    """

    SYSTEM_PROMPT = """你是一个可学习的 AI 助手，名叫 "Hermes"。

## 你的特点
1. 你会从每次交互中学习，不断改进
2. 你会记住用户的偏好和习惯
3. 你会参考历史经验来做更好的决策
4. 你承认自己的不足，并努力改进

## 已知的用户偏好
{user_preferences}

## 相关历史经验
{relevant_experiences}

## 工作原则
1. 参考历史经验，但不要被过时的经验束缚
2. 根据用户偏好调整你的回复风格
3. 如果你不确定，坦诚地说出来
4. 每次回复都尽量做到最好
"""

    def __init__(self, api_key: str = None,
                 model: str = "gpt-4o",
                 base_url: str = None,
                 storage_dir: str = "data"):
        self.api_key = api_key or os.environ.get("OPENAI_API_KEY", "")
        self.model = model
        self.client = OpenAI(api_key=self.api_key, base_url=base_url)

        # 初始化记忆系统
        self.memory = MemorySystem(storage_dir=storage_dir)

        # 初始化评估器和学习引擎
        self.evaluator = SelfEvaluator(
            api_key=self.api_key,
            model="gpt-4o-mini",
            base_url=base_url,
        )
        self.learning = LearningEngine(
            memory=self.memory,
            evaluator=self.evaluator,
            api_key=self.api_key,
            model="gpt-4o-mini",
            base_url=base_url,
        )

    def chat(self, user_input: str) -> str:
        """与 Agent 对话。

        这是核心交互方法，完整实现了学习循环。
        """
        # 1. 记录用户输入到短期记忆
        self.memory.add_to_short_term("user", user_input)

        # 2. 检索相关经验
        relevant_experiences = self.memory.get_relevant_experiences_context(user_input)

        # 3. 获取用户偏好
        user_prefs = self._format_user_preferences()

        # 4. 构建系统提示词
        system_prompt = self.SYSTEM_PROMPT.format(
            user_preferences=user_prefs,
            relevant_experiences=relevant_experiences or "暂无相关经验",
        )

        # 5. 构建对话消息
        messages = [
            {"role": "system", "content": system_prompt},
        ]

        # 添加最近的对话历史
        recent_context = self.memory.get_short_term_context(n_messages=8)
        if recent_context:
            messages.append({
                "role": "user",
                "content": f"[对话上下文]\n{recent_context}\n\n[当前问题]"
            })

        messages.append({"role": "user", "content": user_input})

        # 6. 调用 LLM
        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=messages,
                temperature=0.3,
                max_tokens=2000,
            )
            agent_response = response.choices[0].message.content
        except Exception as e:
            agent_response = f"抱歉，处理请求时出错: {e}"

        # 7. 记录 Agent 回复到短期记忆
        self.memory.add_to_short_term("assistant", agent_response)

        # 8. 学习（评估 + 经验提取）
        learning_result = self.learning.process_interaction(
            user_input=user_input,
            agent_response=agent_response,
        )

        # 如果产生了新经验，显示给用户
        if learning_result.get("new_experiences"):
            agent_response += f"\n\n[学习笔记] 从这次交互中，我学到了：\n"
            for exp in learning_result["new_experiences"]:
                agent_response += f"- {exp}\n"

        return agent_response

    def give_feedback(self, feedback: str):
        """给 Agent 反馈。

        用户可以主动给 Agent 反馈，帮助它学习。
        """
        self.memory.add_to_short_term("user", f"[反馈] {feedback}")

        # 分析反馈并提取经验
        learning_result = self.learning.process_interaction(
            user_input=f"用户反馈: {feedback}",
            agent_response="已收到反馈",
            user_feedback=feedback,
        )

        stats = self.memory.get_memory_stats()
        return (f"收到反馈！当前满意度: {stats['user_satisfaction']:.2f}, "
                f"总经验数: {stats['total_experiences']}")

    def show_memory_stats(self) -> str:
        """显示记忆系统状态。"""
        return self.learning.get_learning_summary()

    def show_experiences(self, category: str = None) -> str:
        """显示所有经验。"""
        experiences = self.memory.experiences
        if category:
            experiences = [e for e in experiences if e.category == category]

        if not experiences:
            return "暂无经验记录。"

        lines = [f"共 {len(experiences)} 条经验：\n"]
        for i, exp in enumerate(experiences, 1):
            lines.append(f"{i}. [{exp.category}] {exp.rule}")
            lines.append(f"   置信度: {exp.confidence:.1f} | 使用次数: {exp.use_count}")
            lines.append(f"   描述: {exp.description}")
            lines.append("")

        return "\n".join(lines)

    def _format_user_preferences(self) -> str:
        """格式化用户偏好为字符串。"""
        prefs = self.memory.user_profile.preferences
        if not prefs:
            return "暂无用户偏好记录"

        lines = []
        for key, value in prefs.items():
            lines.append(f"- {key}: {value}")

        profile = self.memory.user_profile
        if profile.favorite_topics:
            lines.append(f"- 常见话题: {', '.join(profile.favorite_topics[:10])}")

        return "\n".join(lines)
```

### 62.3.5 入口文件

```python
# main.py
import os
from core.agent import HermesAgent


def main():
    print("=" * 60)
    print("  Hermes 可学习 Agent v1.0")
    print("  我会从每次交互中学习和成长")
    print("=" * 60)

    api_key = os.environ.get("OPENAI_API_KEY", "")
    if not api_key:
        print("[错误] 请设置 OPENAI_API_KEY 环境变量。")
        return

    # 创建 Agent
    agent = HermesAgent(
        api_key=api_key,
        model=os.environ.get("HERMES_MODEL", "gpt-4o"),
        storage_dir="data",
    )

    print("\n你好！我是 Hermes，你的可学习助手。")
    print("我会从每次交互中学习，越来越了解你。")
    print("\n命令：")
    print("  /feedback <内容>  - 给我反馈")
    print("  /stats            - 查看学习状况")
    print("  /experiences      - 查看所有经验")
    print("  /quit             - 退出")
    print("-" * 40)

    while True:
        try:
            user_input = input("\n你> ").strip()
        except (EOFError, KeyboardInterrupt):
            break

        if not user_input:
            continue

        if user_input == "/quit":
            break

        elif user_input == "/stats":
            print(agent.show_memory_stats())

        elif user_input == "/experiences":
            print(agent.show_experiences())

        elif user_input.startswith("/feedback "):
            feedback = user_input[10:].strip()
            result = agent.give_feedback(feedback)
            print(result)

        else:
            response = agent.chat(user_input)
            print(f"\nHermes> {response}")

    print("\n再见！我会记住我们这次对话的。下次见！")


if __name__ == "__main__":
    main()
```

---

## 62.4 案例分析

### 62.4.1 案例：学习用户偏好

第一次交互：
```
你> 帮我解释一下什么是 Transformer
Hermes> [长篇解释，包含很多技术细节...]
```

用户反馈：
```
你> /feedback 你的回复太长了，我想要更简洁的解释
Hermes> 收到反馈！我会记住你喜欢简洁的回复。
```

学习系统提取了经验：
```
[新经验] [preference] 当用户要求解释概念时，应该提供简洁版本，
除非用户明确要求详细解释。用户偏好简洁的回复风格。
```

第二次交互：
```
你> 帮我解释一下 RAG
Hermes> RAG（检索增强生成）是一种让 LLM 获取外部知识的技术。
它先从数据库中检索相关信息，然后把检索结果和问题一起发给 LLM 生成回答。
这样 LLM 就能回答基于最新数据的问题，而不仅依赖训练数据。
[这次回复明显更简洁了]
```

### 62.4.2 案例：从错误中学习

Agent 在回答一个数学计算时出错：
```
你> 1234 * 5678 等于多少？
Hermes> 1234 * 5678 = 7006652 [实际答案是 7006652，这次碰巧对了]
```

但如果计算错误：
```
你> /feedback 你算错了，正确答案应该是 7006652
```

系统会提取经验：
```
[新经验] [error_correction] 对于精确的数学计算，应该使用计算器工具
而不是让 LLM 直接计算。LLM 的算术能力不可靠。
```

---

## 62.5 常见坑与解决方案

### 坑一：经验质量参差不齐

**问题描述：** LLM 提取的经验有些很有价值，有些则过于笼统。

**解决方案：** 对提取的经验进行二次过滤。使用另一个 LLM 调用来评估每条经验的质量，只保留高质量的经验。也可以设置经验的最低置信度阈值。

### 坑二：经验过多导致检索困难

**问题描述：** 随着使用时间增长，经验库越来越大，检索效率和准确性下降。

**解决方案：** 定期进行经验清理和合并。相似的经验合并为一条，长期未使用的经验降低权重或删除。使用向量检索替代关键词匹配。

### 坑三：过时经验的干扰

**问题描述：** 用户的偏好可能随时间变化，但旧的经验仍然被检索到。

**解决方案：** 实现经验的时效性衰减。越老的经验权重越低。同时，当用户给出与旧经验矛盾的反馈时，自动降低旧经验的权重。

---

## 62.6 练习题

### 练习一：实现经验衰减机制（难度：初级）
实现经验的自动衰减：长时间未使用的经验降低权重，超过一定时间且从未使用的经验自动归档。

### 练习二：添加经验合并功能（难度：中级）
实现相似经验的自动合并。当新经验与已有经验高度相似时，合并为一条并更新置信度。

### 练习三：实现多用户记忆隔离（难度：中级）
让系统支持多个用户，每个用户有独立的记忆和经验库。

### 练习四：实现可视化的学习曲线（难度：高级）
生成用户满意度随时间变化的图表，展示 Agent 的学习进度。

### 练习五：实现经验的自动验证（难度：高级）
当一条经验被多次使用且反馈良好时，自动提高其置信度；反之则降低。

---

## 62.7 实战任务

### 任务一：至少使用 20 轮对话

认真使用 Hermes Agent 进行至少 20 轮对话，期间给出真实的反馈。观察 Agent 是否真正"学到"了你的偏好。

### 任务二：故意测试错误修正

故意指出 Agent 的错误，看它能否从中提取有用的经验，并在未来避免类似的错误。

### 任务三：分析经验质量

查看 Agent 积累的所有经验，评估哪些是有价值的、哪些是冗余的。手动清理和优化经验库。

---

## 62.8 本章小结

可学习 Agent 项目展示了 Agent 从"静态工具"到"动态伙伴"的进化路径。这个项目有几个核心启示。

第一，**三层记忆架构模仿了人类的记忆机制**。短期记忆处理即时上下文，工作记忆维持任务状态，长期记忆积累持久经验。这种分层设计让 Agent 能够在不同时间尺度上保持连贯性。

第二，**自我评估是学习的前提**。没有评估就无法区分好与坏，也就无法从交互中提取有价值的经验。通过让 LLM 对自己的表现进行评估，Agent 获得了一面"镜子"。

第三，**经验的筛选和管理是长期挑战**。不是所有的交互结果都值得记忆，不是所有的经验都永远有效。如何判断经验的价值、如何处理过时的经验、如何避免经验的冲突，这些都是需要持续优化的问题。

至此，我们完成了六个实战项目的构建。从研究助手到可学习 Agent，每个项目都解决了一个特定场景的问题，同时展示了 Agent 开发的不同方面。下一章，我们将对这六个项目进行系统的对比和总结。
