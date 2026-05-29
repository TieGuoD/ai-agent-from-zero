# 第 61 章：项目 5 -- 个人数字员工

> **本章定位：** 这是前五个项目的集大成之作。我们将构建一个统一的"个人数字员工"平台，它整合了研究助手、PDF分析、邮件处理、代码助手等多种能力，并通过一个统一的对话界面让用户与之交互。这个项目将让你体验如何把多个独立的 Agent 能力融合成一个协调工作的智能系统。

---

## 学习目标

完成本章学习后，你将能够：

1. **设计多技能 Agent 的统一架构** -- 能够将多个独立的 Agent 能力（搜索、文档分析、邮件处理、代码操作等）整合到一个协调工作的系统中
2. **实现智能任务路由机制** -- 能够让系统自动识别用户的意图，并将任务分配给最合适的技能模块
3. **构建共享上下文管理系统** -- 能够让不同的技能模块共享用户偏好、历史交互、项目状态等上下文信息
4. **设计可扩展的插件架构** -- 能够让新技能以插件的形式添加到系统中，无需修改核心代码
5. **实现持续学习和偏好记忆** -- 能够让系统记住用户的偏好、习惯和反馈，在交互中不断改进
6. **完成一个可日常使用的个人助手** -- 通过自然语言交互，完成各种日常工作任务

## 核心问题

1. **为什么要把多个 Agent 能力整合到一个系统中？** 相比多个独立的 Agent，统一平台的优势和挑战是什么？
2. **任务路由如何设计才足够智能？** 用户的请求可能模糊或包含多个意图，系统如何正确理解并分配？
3. **如何让"数字员工"越来越了解你？** 记忆和学习机制如何设计，才能在保护隐私的前提下提供个性化服务？

---

## 61.1 项目背景与需求分析

### 61.1.1 从单一能力到综合能力

回顾前面的项目，我们分别构建了研究助手（第57章）、PDF分析Agent（第58章）、邮件Agent（第59章）和代码Agent（第60章）。每个 Agent 在各自的领域都表现出色，但它们是独立运行的。

在实际工作中，这些能力往往是交织在一起的。你可能需要先研究一个话题，然后阅读相关的PDF文档，接着给同事发一封邮件总结发现，最后根据研究结果修改代码。如果你需要在四个不同的工具之间切换，效率并不会比手工操作提高多少。

个人数字员工的目标就是打破这些壁垒。它就像一个真正的工作伙伴，你说一句"帮我调研竞品的技术方案，总结成报告，然后邮件发给团队"，它就会自动完成整个工作流。

### 61.1.2 核心功能定义

个人数字员工需要具备以下核心能力：

首先是**自然语言理解**。系统需要能够理解用户的自然语言指令，识别其中的意图和关键信息。这包括单步指令（"帮我搜索XXX"）和复合指令（"搜索XXX，整理成报告，然后邮件发送"）。

其次是**任务路由**。当用户发出指令时，系统需要判断这个指令属于哪个技能领域，并调用相应的处理模块。对于复合指令，系统还需要自动拆解为多个子任务并按顺序执行。

然后是**上下文管理**。系统需要维护一个统一的上下文，让不同的技能模块能够共享信息。例如，研究助手找到的信息可以被邮件Agent用来生成回复草稿。

最后是**个性化学习**。系统需要记住用户的偏好和习惯，在后续交互中越来越贴合用户的需求。

---

## 61.2 架构设计

### 61.2.1 插件架构

```
personal-worker/
    main.py              # 入口文件
    core/
        agent.py         # 核心 Agent
        router.py        # 任务路由器
        context.py       # 上下文管理器
        memory.py        # 持久化记忆
        planner.py       # 任务规划器
    skills/
        base.py          # 技能基类
        research.py      # 研究技能
        document.py      # 文档分析技能
        email_skill.py   # 邮件技能
        code_skill.py    # 代码技能
        web.py           # 网络搜索技能
        calendar.py      # 日历技能
    config.py            # 配置管理
    requirements.txt
```

---

## 61.3 核心代码实现

### 61.3.1 技能基类

```python
# skills/base.py
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import Any, Optional


@dataclass
class SkillResult:
    """技能执行结果。"""
    success: bool
    data: Any = None
    message: str = ""
    metadata: dict = field(default_factory=dict)


class BaseSkill(ABC):
    """所有技能的基类。

    每个技能必须实现以下方法：
    - name: 技能名称
    - description: 技能描述（用于路由决策）
    - execute: 执行技能
    - get_capabilities: 返回技能能力列表
    """

    @property
    @abstractmethod
    def name(self) -> str:
        """技能名称。"""
        pass

    @property
    @abstractmethod
    def description(self) -> str:
        """技能描述，用于帮助路由器理解这个技能能做什么。"""
        pass

    @abstractmethod
    def execute(self, action: str, params: dict,
                context: dict = None) -> SkillResult:
        """执行技能。

        Args:
            action: 具体的操作（如 "search", "summarize"）
            params: 操作参数
            context: 共享上下文

        Returns:
            SkillResult 对象
        """
        pass

    @abstractmethod
    def get_capabilities(self) -> list[str]:
        """返回技能支持的能力列表。

        例如：["搜索网络", "阅读网页", "下载文件"]
        """
        pass

    def can_handle(self, intent: str) -> bool:
        """判断这个技能是否能处理给定的意图。

        默认实现：检查意图是否包含技能名称或能力关键词。
        子类可以覆盖这个方法提供更精确的判断。
        """
        capabilities = self.get_capabilities()
        intent_lower = intent.lower()

        if self.name.lower() in intent_lower:
            return True

        for cap in capabilities:
            if cap.lower() in intent_lower:
                return True

        return False
```

### 61.3.2 任务路由器

```python
# core/router.py
import json
from openai import OpenAI
from skills.base import BaseSkill


class TaskRouter:
    """任务路由器。

    负责理解用户意图，并将任务分配给最合适的技能。
    使用 LLM 进行意图识别，比传统的关键词匹配更准确。
    """

    ROUTING_PROMPT = """你是一个任务路由器。根据用户的请求，决定应该调用哪些技能。

## 可用技能

{skills_description}

## 用户请求

"{user_request}"

## 要求

请分析用户请求，返回 JSON 格式的路由决策：

```json
{{
    "tasks": [
        {{
            "skill": "技能名称",
            "action": "具体操作",
            "params": {{"参数名": "参数值"}},
            "priority": "high/medium/low",
            "depends_on": []
        }}
    ],
    "overall_intent": "用户意图的一句话概括",
    "requires_clarification": false,
    "clarification_question": ""
}}
```

## 路由规则

1. 如果用户的请求不明确，设置 requires_clarification=true
2. 如果请求包含多个步骤，拆分为多个任务，设置正确的依赖关系
3. 选择最合适的技能来处理每个任务
4. 如果没有任何技能能处理，返回空的 tasks 列表
"""

    def __init__(self, skills: list[BaseSkill], api_key: str,
                 model: str = "gpt-4o-mini", base_url: str = None):
        self.skills = {s.name: s for s in skills}
        self.client = OpenAI(api_key=api_key, base_url=base_url)
        self.model = model

    def route(self, user_request: str) -> dict:
        """分析用户请求并返回路由决策。"""
        skills_desc = self._format_skills_description()
        prompt = self.ROUTING_PROMPT.format(
            skills_description=skills_desc,
            user_request=user_request
        )

        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": "你是一个精确的任务路由器。请返回 JSON 格式。"},
                    {"role": "user", "content": prompt}
                ],
                temperature=0.0,
                max_tokens=1000,
                response_format={"type": "json_object"},
            )

            result = json.loads(response.choices[0].message.content)
            return result

        except Exception as e:
            print(f"[路由失败] {e}")
            return {
                "tasks": [],
                "overall_intent": "无法识别",
                "requires_clarification": True,
                "clarification_question": "抱歉，我没有理解你的意思，请再说一次。",
            }

    def _format_skills_description(self) -> str:
        """格式化所有技能的描述。"""
        lines = []
        for name, skill in self.skills.items():
            capabilities = ", ".join(skill.get_capabilities())
            lines.append(f"- **{name}**: {skill.description}")
            lines.append(f"  能力: {capabilities}")
        return "\n".join(lines)
```

### 61.3.3 上下文管理器

```python
# core/context.py
import json
import os
import time
from dataclasses import dataclass, field
from typing import Any, Optional


@dataclass
class UserPreferences:
    """用户偏好。"""
    language: str = "zh"           # 首选语言
    response_style: str = "detailed"  # detailed/concise/formal
    timezone: str = "Asia/Shanghai"
    email_address: str = ""
    default_model: str = "gpt-4o"
    custom_settings: dict = field(default_factory=dict)


@dataclass
class ConversationTurn:
    """单轮对话记录。"""
    timestamp: float
    user_input: str
    agent_response: str
    tasks_executed: list[dict] = field(default_factory=list)
    feedback: Optional[str] = None  # 用户反馈


class ContextManager:
    """上下文管理器。

    管理三层上下文：
    1. 会话上下文：当前对话的临时状态（易失）
    2. 用户上下文：用户偏好和设置（持久化）
    3. 共享上下文：技能间传递的信息（会话级）
    """

    def __init__(self, storage_dir: str = "data"):
        self.storage_dir = storage_dir
        os.makedirs(storage_dir, exist_ok=True)

        # 会话上下文
        self.session_id = str(int(time.time()))
        self.conversation_history: list[ConversationTurn] = []
        self.shared_data: dict = {}

        # 用户上下文（持久化）
        self.user_prefs = self._load_user_prefs()

    def set_shared_data(self, key: str, value: Any):
        """设置共享数据（技能间传递信息用）。"""
        self.shared_data[key] = value

    def get_shared_data(self, key: str, default=None):
        """获取共享数据。"""
        return self.shared_data.get(key, default)

    def add_conversation_turn(self, user_input: str, agent_response: str,
                              tasks: list[dict] = None):
        """记录一轮对话。"""
        turn = ConversationTurn(
            timestamp=time.time(),
            user_input=user_input,
            agent_response=agent_response,
            tasks_executed=tasks or [],
        )
        self.conversation_history.append(turn)
        self._save_conversation()

    def get_recent_context(self, n_turns: int = 5) -> str:
        """获取最近几轮对话的上下文摘要。"""
        recent = self.conversation_history[-n_turns:]
        if not recent:
            return "这是对话的开始。"

        lines = ["最近的对话记录："]
        for turn in recent:
            lines.append(f"用户: {turn.user_input[:100]}")
            lines.append(f"助手: {turn.agent_response[:100]}")
        return "\n".join(lines)

    def get_full_context_for_llm(self) -> str:
        """为 LLM 生成完整的上下文信息。"""
        parts = []

        # 用户偏好
        prefs = self.user_prefs
        parts.append(f"用户偏好：语言={prefs.language}, 风格={prefs.response_style}")

        # 最近对话
        parts.append(f"\n{self.get_recent_context(5)}")

        # 共享数据摘要
        if self.shared_data:
            parts.append("\n当前共享数据：")
            for key, value in self.shared_data.items():
                val_str = str(value)[:200]
                parts.append(f"  {key}: {val_str}")

        return "\n".join(parts)

    def record_feedback(self, turn_index: int, feedback: str):
        """记录用户反馈（用于学习）。"""
        if 0 <= turn_index < len(self.conversation_history):
            self.conversation_history[turn_index].feedback = feedback

    def _load_user_prefs(self) -> UserPreferences:
        """加载用户偏好。"""
        prefs_file = os.path.join(self.storage_dir, "user_prefs.json")
        try:
            with open(prefs_file, "r", encoding="utf-8") as f:
                data = json.load(f)
            return UserPreferences(**data)
        except (FileNotFoundError, TypeError):
            return UserPreferences()

    def save_user_prefs(self):
        """保存用户偏好。"""
        prefs_file = os.path.join(self.storage_dir, "user_prefs.json")
        data = {
            "language": self.user_prefs.language,
            "response_style": self.user_prefs.response_style,
            "timezone": self.user_prefs.timezone,
            "email_address": self.user_prefs.email_address,
            "default_model": self.user_prefs.default_model,
            "custom_settings": self.user_prefs.custom_settings,
        }
        with open(prefs_file, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=2)

    def _save_conversation(self):
        """保存对话记录。"""
        conv_file = os.path.join(
            self.storage_dir, f"conversation_{self.session_id}.json"
        )
        data = []
        for turn in self.conversation_history:
            data.append({
                "timestamp": turn.timestamp,
                "user_input": turn.user_input,
                "agent_response": turn.agent_response[:500],
                "tasks_executed": turn.tasks_executed,
                "feedback": turn.feedback,
            })
        with open(conv_file, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
```

### 61.3.4 研究技能（简化版）

```python
# skills/research.py
from openai import OpenAI
from skills.base import BaseSkill, SkillResult
import json


class ResearchSkill(BaseSkill):
    """研究技能：搜索互联网、阅读网页、整理信息。"""

    @property
    def name(self) -> str:
        return "research"

    @property
    def description(self) -> str:
        return "在互联网上搜索信息、阅读网页内容、整理研究资料"

    def __init__(self, api_key: str, model: str = "gpt-4o", base_url: str = None):
        self.client = OpenAI(api_key=api_key, base_url=base_url)
        self.model = model

    def get_capabilities(self) -> list[str]:
        return ["搜索", "研究", "调研", "查找", "搜索互联网", "阅读网页", "总结信息"]

    def execute(self, action: str, params: dict, context: dict = None) -> SkillResult:
        if action == "search":
            return self._search(params.get("query", ""), params.get("max_results", 5))
        elif action == "summarize_topic":
            return self._summarize_topic(params.get("topic", ""))
        else:
            return SkillResult(success=False, message=f"未知操作: {action}")

    def _search(self, query: str, max_results: int) -> SkillResult:
        """执行搜索。"""
        try:
            from duckduckgo_search import DDGS
            results = []
            with DDGS() as ddgs:
                for r in ddgs.text(query, max_results=max_results):
                    results.append({
                        "title": r.get("title", ""),
                        "url": r.get("href", ""),
                        "snippet": r.get("body", "")
                    })
            return SkillResult(
                success=True,
                data=results,
                message=f"搜索 '{query}' 找到 {len(results)} 条结果"
            )
        except ImportError:
            return SkillResult(
                success=False,
                message="请安装 duckduckgo_search: pip install duckduckgo-search"
            )
        except Exception as e:
            return SkillResult(success=False, message=f"搜索失败: {e}")

    def _summarize_topic(self, topic: str) -> SkillResult:
        """让 LLM 总结一个话题。"""
        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": "你是一个研究助手。请简洁地总结以下话题。"},
                    {"role": "user", "content": f"请总结关于 {topic} 的最新发展，约300字。"}
                ],
                temperature=0.3,
                max_tokens=1000,
            )
            summary = response.choices[0].message.content
            return SkillResult(success=True, data=summary, message="总结完成")
        except Exception as e:
            return SkillResult(success=False, message=f"总结失败: {e}")
```

### 61.3.5 邮件技能（简化版）

```python
# skills/email_skill.py
from skills.base import BaseSkill, SkillResult


class EmailSkill(BaseSkill):
    """邮件技能：读取、分类、回复邮件。"""

    @property
    def name(self) -> str:
        return "email"

    @property
    def description(self) -> str:
        return "处理邮件：读取、发送、分类、回复、生成摘要"

    def __init__(self, connector=None, reply_generator=None):
        self.connector = connector
        self.reply_generator = reply_generator

    def get_capabilities(self) -> list[str]:
        return ["邮件", "收件箱", "发送邮件", "回复邮件", "邮件摘要", "邮件分类", "email"]

    def execute(self, action: str, params: dict, context: dict = None) -> SkillResult:
        if action == "read_summary":
            return self._read_summary(params.get("limit", 10))
        elif action == "generate_reply":
            return self._generate_reply(params.get("uid", ""), params.get("style", "professional"))
        elif action == "send":
            return self._send_email(params)
        else:
            return SkillResult(success=False, message=f"未知操作: {action}")

    def _read_summary(self, limit: int) -> SkillResult:
        """读取邮件摘要。"""
        if not self.connector:
            return SkillResult(success=False, message="邮件连接器未初始化")

        try:
            uids = self.connector.search_emails(limit=limit)
            emails = self.connector.fetch_multiple(uids)
            summaries = []
            for em in emails:
                summaries.append({
                    "uid": em.uid,
                    "from": em.sender,
                    "subject": em.subject,
                    "preview": em.get_preview(100),
                    "date": str(em.date) if em.date else "",
                })
            return SkillResult(
                success=True,
                data=summaries,
                message=f"获取了 {len(summaries)} 封邮件摘要"
            )
        except Exception as e:
            return SkillResult(success=False, message=f"读取邮件失败: {e}")

    def _generate_reply(self, uid: str, style: str) -> SkillResult:
        """生成邮件回复。"""
        if not self.reply_generator:
            return SkillResult(success=False, message="回复生成器未初始化")

        try:
            email_msg = self.connector.fetch_email(uid) if self.connector else None
            if not email_msg:
                return SkillResult(success=False, message=f"未找到邮件: {uid}")

            reply = self.reply_generator.generate_reply(
                sender=email_msg.sender,
                subject=email_msg.subject,
                body=email_msg.body_text,
                style=style,
            )
            return SkillResult(
                success=True,
                data={"reply": reply, "to": email_msg.sender, "subject": email_msg.subject},
                message="回复草稿已生成"
            )
        except Exception as e:
            return SkillResult(success=False, message=f"生成回复失败: {e}")

    def _send_email(self, params: dict) -> SkillResult:
        """发送邮件。"""
        if not self.connector:
            return SkillResult(success=False, message="邮件连接器未初始化")

        try:
            success = self.connector.send_email(
                to=params.get("to", ""),
                subject=params.get("subject", ""),
                body=params.get("body", ""),
            )
            if success:
                return SkillResult(success=True, message="邮件已发送")
            else:
                return SkillResult(success=False, message="邮件发送失败")
        except Exception as e:
            return SkillResult(success=False, message=f"发送失败: {e}")
```

### 61.3.6 代码技能（简化版）

```python
# skills/code_skill.py
import os
from skills.base import BaseSkill, SkillResult


class CodeSkill(BaseSkill):
    """代码技能：代码分析、生成、审查。"""

    @property
    def name(self) -> str:
        return "code"

    @property
    def description(self) -> str:
        return "代码操作：分析代码结构、解释代码、生成代码、代码审查"

    def get_capabilities(self) -> list[str]:
        return ["代码", "编程", "函数", "类", "bug", "修复", "测试", "code", "python"]

    def __init__(self, api_key: str = None, project_path: str = "."):
        self.project_path = os.path.abspath(project_path)
        self.api_key = api_key

    def execute(self, action: str, params: dict, context: dict = None) -> SkillResult:
        if action == "analyze":
            return self._analyze_project()
        elif action == "explain":
            return self._explain_file(params.get("file_path", ""))
        elif action == "list_files":
            return self._list_files()
        else:
            return SkillResult(success=False, message=f"未知操作: {action}")

    def _analyze_project(self) -> SkillResult:
        """分析项目结构。"""
        try:
            files = []
            for root, dirs, filenames in os.walk(self.project_path):
                dirs[:] = [d for d in dirs if d not in {
                    ".git", "__pycache__", "node_modules", ".venv"
                }]
                for fn in filenames:
                    if fn.endswith((".py", ".js", ".ts")):
                        rel_path = os.path.relpath(
                            os.path.join(root, fn), self.project_path
                        )
                        files.append(rel_path)

            return SkillResult(
                success=True,
                data={"files": files, "count": len(files)},
                message=f"项目包含 {len(files)} 个代码文件"
            )
        except Exception as e:
            return SkillResult(success=False, message=f"分析失败: {e}")

    def _explain_file(self, file_path: str) -> SkillResult:
        """解释代码文件。"""
        full_path = os.path.join(self.project_path, file_path)
        if not os.path.exists(full_path):
            return SkillResult(success=False, message=f"文件不存在: {file_path}")

        try:
            with open(full_path, "r", encoding="utf-8", errors="replace") as f:
                content = f.read()
            return SkillResult(
                success=True,
                data={"content": content[:5000], "path": file_path},
                message=f"已读取 {file_path}"
            )
        except Exception as e:
            return SkillResult(success=False, message=f"读取失败: {e}")

    def _list_files(self) -> SkillResult:
        """列出项目文件。"""
        return self._analyze_project()
```

### 61.3.7 核心 Agent

```python
# core/agent.py
import json
from openai import OpenAI
from core.router import TaskRouter
from core.context import ContextManager
from skills.base import BaseSkill, SkillResult


class PersonalWorker:
    """个人数字员工的核心 Agent。

    工作流程：
    1. 接收用户输入
    2. 路由器分析意图
    3. 执行对应的技能
    4. 整合结果生成回复
    5. 记录上下文和反馈
    """

    def __init__(self, skills: list[BaseSkill], api_key: str,
                 model: str = "gpt-4o", base_url: str = None):
        self.client = OpenAI(api_key=api_key, base_url=base_url)
        self.model = model
        self.skills = {s.name: s for s in skills}
        self.context = ContextManager()
        self.router = TaskRouter(skills, api_key=api_key, model="gpt-4o-mini", base_url=base_url)

        self.SYSTEM_PROMPT = """你是一个个人数字员工，名叫"小助"。你能够帮助用户完成各种工作任务。

## 你的能力
你拥有以下技能模块，可以根据用户的需求调用：
{skills_list}

## 工作原则
1. 理解用户的意图，调用合适的技能来完成任务
2. 将复杂的任务拆解为清晰的步骤
3. 给出明确的执行结果和下一步建议
4. 如果信息不足，主动向用户确认
5. 保持友好、专业的沟通风格

## 回复格式
- 对于简单查询，直接给出结果
- 对于复杂任务，先说明执行步骤，再逐一展示结果
- 如果任务执行失败，说明原因并提供替代方案
"""

    def chat(self, user_input: str) -> str:
        """处理用户输入并返回回复。

        这是数字员工的核心交互方法。
        """
        # 获取上下文
        context_str = self.context.get_full_context_for_llm()

        # 路由决策
        routing = self.router.route(user_input)

        # 检查是否需要澄清
        if routing.get("requires_clarification", False):
            clarification = routing.get("clarification_question", "请再说详细一些。")
            self.context.add_conversation_turn(user_input, clarification)
            return clarification

        # 执行任务
        tasks = routing.get("tasks", [])
        execution_results = []

        for task in tasks:
            skill_name = task.get("skill", "")
            action = task.get("action", "")
            params = task.get("params", {})

            if skill_name in self.skills:
                skill = self.skills[skill_name]
                result = skill.execute(action, params, context=self.context.shared_data)
                execution_results.append({
                    "skill": skill_name,
                    "action": action,
                    "success": result.success,
                    "message": result.message,
                    "data": str(result.data)[:500] if result.data else None,
                })

                # 将结果放入共享上下文
                if result.success and result.data:
                    self.context.set_shared_data(
                        f"{skill_name}_{action}", result.data
                    )
            else:
                execution_results.append({
                    "skill": skill_name,
                    "success": False,
                    "message": f"技能 '{skill_name}' 不存在",
                })

        # 生成综合回复
        response = self._generate_response(
            user_input, routing, execution_results, context_str
        )

        # 记录对话
        self.context.add_conversation_turn(
            user_input, response,
            tasks=[{"skill": t.get("skill"), "action": t.get("action")} for t in tasks]
        )

        return response

    def _generate_response(self, user_input: str, routing: dict,
                           results: list[dict], context: str) -> str:
        """基于执行结果生成自然语言回复。"""
        intent = routing.get("overall_intent", "处理用户请求")
        results_str = json.dumps(results, ensure_ascii=False, indent=2)

        messages = [
            {"role": "system", "content": self.SYSTEM_PROMPT.format(
                skills_list="\n".join(f"- {name}" for name in self.skills.keys())
            )},
            {"role": "user", "content": f"""用户请求: {user_input}

识别到的意图: {intent}

执行结果:
{results_str}

上下文信息:
{context}

请根据以上信息，给用户一个友好、清晰的回复。如果任务执行成功，总结结果。如果失败，说明原因并建议替代方案。"""}
        ]

        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=messages,
                temperature=0.3,
                max_tokens=1500,
            )
            return response.choices[0].message.content
        except Exception as e:
            return f"处理请求时出错: {e}"

    def get_capabilities(self) -> str:
        """获取所有技能的能力描述。"""
        lines = []
        for name, skill in self.skills.items():
            caps = ", ".join(skill.get_capabilities())
            lines.append(f"- {name}: {caps}")
        return "\n".join(lines)
```

### 61.3.8 入口文件

```python
# main.py
import os
from core.agent import PersonalWorker
from skills.research import ResearchSkill
from skills.email_skill import EmailSkill
from skills.code_skill import CodeSkill


def main():
    print("=" * 60)
    print("  个人数字员工 v1.0")
    print("  你的 AI 工作伙伴")
    print("=" * 60)

    api_key = os.environ.get("OPENAI_API_KEY", "")
    if not api_key:
        print("[错误] 请设置 OPENAI_API_KEY 环境变量。")
        return

    # 初始化技能
    skills = []

    # 研究技能
    research_skill = ResearchSkill(
        api_key=api_key,
        model=os.environ.get("RESEARCH_MODEL", "gpt-4o"),
    )
    skills.append(research_skill)
    print("[+] 研究技能已加载")

    # 代码技能
    code_skill = CodeSkill(api_key=api_key, project_path=".")
    skills.append(code_skill)
    print("[+] 代码技能已加载")

    # 邮件技能（需要配置邮箱）
    if os.environ.get("EMAIL_ADDRESS"):
        from mail_connector import MailConnector, MailConfig
        from reply_generator import ReplyGenerator

        mail_config = MailConfig.from_env("gmail")
        connector = MailConnector(mail_config)
        if connector.connect():
            reply_gen = ReplyGenerator(api_key=api_key)
            email_skill = EmailSkill(connector=connector, reply_generator=reply_gen)
            skills.append(email_skill)
            print("[+] 邮件技能已加载")

    # 创建数字员工
    worker = PersonalWorker(
        skills=skills,
        api_key=api_key,
        model=os.environ.get("WORKER_MODEL", "gpt-4o"),
    )

    print(f"\n技能列表：\n{worker.get_capabilities()}")
    print("\n你好！我是你的数字员工"小助"，有什么可以帮你的？")
    print("输入 /quit 退出\n")

    while True:
        try:
            user_input = input("你> ").strip()
        except (EOFError, KeyboardInterrupt):
            break

        if not user_input:
            continue
        if user_input == "/quit":
            break

        response = worker.chat(user_input)
        print(f"\n小助> {response}\n")

    print("\n再见！")


if __name__ == "__main__":
    main()
```

---

## 61.4 案例分析

### 61.4.1 案例：复合任务执行

用户输入："帮我搜索 AI Agent 最新趋势，然后总结成要点列表"

Agent 的处理过程：
1. 路由器识别为复合任务，包含"搜索"和"总结"两个步骤
2. 先调用研究技能搜索相关信息
3. 将搜索结果放入共享上下文
4. 调用 LLM 对搜索结果进行总结
5. 生成格式化的要点列表

### 61.4.2 案例：意图模糊时的澄清

用户输入："帮我处理那个文件"

路由器发现请求不明确，返回澄清问题："请问你想对文件做什么操作？是分析、修改、还是发送？"

---

## 61.5 常见坑与解决方案

### 坑一：路由决策不准确

**问题描述：** 路由器错误地将任务分配给了不合适的技能。

**解决方案：** 优化技能描述，让它更精确地表达能力范围。同时在路由器的提示词中增加更多示例。

### 坑二：复合任务执行中断

**问题描述：** 多步任务中，某一步失败导致后续步骤无法执行。

**解决方案：** 在任务规划器中增加容错逻辑。某一步失败时，记录失败原因并尝试替代方案，而不是直接中断。

### 坑三：上下文信息过多导致 LLM 混乱

**问题描述：** 上下文中包含了太多历史信息，干扰了 LLM 的决策。

**解决方案：** 对上下文进行智能压缩。只保留与当前任务最相关的历史信息。

---

## 61.6 练习题

### 练习一：添加日历技能（难度：初级）
实现一个日历技能，支持查看今天的日程、添加新事件、查找空闲时间。

### 练习二：添加文件管理技能（难度：中级）
实现文件管理技能，支持文件搜索、重命名、整理（按类型分类）等操作。

### 练习三：实现工作流引擎（难度：高级）
支持用户定义自定义工作流，如"每天早上9点检查邮件，整理成摘要，然后推送给我"。

### 练习四：实现语音交互（难度：高级）
添加语音输入和语音输出功能，让用户可以通过语音与数字员工交互。

### 练习五：添加多用户支持（难度：高级）
让系统支持多个用户，每个用户有独立的偏好和记忆。

---

## 61.7 实战任务

### 任务一：配置并测试个人数字员工

在你的电脑上配置个人数字员工，至少启用研究技能和代码技能。测试以下场景：
1. 让它帮你研究一个技术话题
2. 让它分析你当前项目的代码结构
3. 测试复合任务（"搜索XXX，然后分析它在代码中的实现"）

### 任务二：添加新技能

为个人数字员工添加至少一个新技能。建议的方向：
- 天气查询技能
- 翻译技能
- 数据分析技能（读取 CSV/Excel 并生成分析报告）

### 任务三：优化路由准确率

准备 20 个测试用例，覆盖各种可能的用户输入。测试路由器的准确率，找出分类错误的案例，优化提示词。

---

## 61.8 本章小结

个人数字员工项目展示了如何将多个独立的 Agent 能力整合到一个统一的系统中。这个项目有几个重要的架构启示。

第一，**插件架构是多技能 Agent 的最佳实践**。通过定义统一的技能接口（BaseSkill），我们可以轻松地添加或移除技能，而不需要修改核心代码。这种松耦合的设计使得系统具有良好的可扩展性。

第二，**LLM 驱动的路由比规则路由更智能**。传统的意图识别依赖预定义的规则，无法处理模糊或多义的用户输入。使用 LLM 进行路由决策，能够理解自然语言的细微差别，处理更加灵活。

第三，**共享上下文是多技能协作的桥梁**。不同的技能模块需要共享信息才能完成复杂的任务。通过 ContextManager，我们提供了一个统一的上下文管理机制，让技能之间可以无缝协作。

下一章，我们将构建最后一个项目——一个可学习的 Agent，它能够从与用户的交互中不断学习和改进。
