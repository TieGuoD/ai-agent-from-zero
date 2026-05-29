# 第 8 章：Agent 的状态管理 —— 对话历史与上下文

> **本章定位：** Agent 需要记住之前的对话和行为。没有良好的状态管理，Agent 每次对话都是"失忆"状态，无法完成需要上下文的复杂任务。本章将带你像读故事一样理解 Agent 的状态管理，从"为什么 Agent 会失忆"开始，逐步深入到对话历史管理、上下文窗口优化、状态持久化等核心话题。

---

## 学习目标

1. **理解 Agent 的状态类型** —— 对话状态、任务状态、环境状态的区别和联系，以及为什么每种状态都重要
2. **掌握对话历史的存储和管理策略** —— 滑动窗口、摘要压缩、选择性保留，理解每种策略的原理和适用场景
3. **理解上下文窗口管理** —— Token 计数、智能裁剪、选择性注入，理解如何在有限空间内最大化 Agent 的能力
4. **掌握状态持久化方案** —— 如何让 Agent 跨会话保持状态

## 核心问题

1. **Agent 需要管理哪些状态？** 每种状态的作用是什么？如果缺少某种状态会怎样？
2. **对话历史过长怎么办？** 上下文窗口有限，如何在有限空间内保留最重要的信息？
3. **如何在多轮对话中保持上下文一致性？** Agent 如何"记住"之前的对话内容？

---

## 8.1 一个"失忆"Agent 的故事

### 8.1.1 Agent 的尴尬时刻

想象你正在和一个 Agent 对话。你告诉它："我叫张三，我是一个 Python 开发者。"Agent 回答："好的，张三，我会记住你的信息。"

然后你问："帮我推荐一些 Python 学习资源。"Agent 给了你一些推荐。

接着你问："根据我的背景，哪些资源最适合我？"

这时候，Agent 可能会回答："请问您的背景是什么？"

这就是"失忆"——Agent 忘记了你之前告诉它的信息。你明明说过你是 Python 开发者，但它"忘了"。

### 8.1.2 为什么会"失忆"

Agent "失忆"的原因是：LLM 的上下文窗口是有限的。每一轮对话都会消耗 Token，当对话历史超过上下文窗口的限制时，早期的对话就会被"挤出去"。

更根本的原因是：LLM 本身是无状态的。每次调用 LLM，它都只看到你这次发送的消息，不知道之前的对话历史。是我们在外面"包装"了一层状态管理，让 LLM 看起来有记忆。

```
实际上：每次调用 LLM 都是独立的，没有记忆
看起来：通过管理对话历史，让 LLM 看起来有记忆
```

### 8.1.3 状态管理的重要性

状态管理是 Agent 能持续工作的基础。没有良好的状态管理：

- Agent 会"失忆"，忘记之前的对话
- Agent 无法跟踪任务进度
- Agent 无法利用历史信息做出更好的决策
- 用户体验极差（每次都要重复告诉 Agent 信息）

---

## 8.2 Agent 的三种核心状态

### 8.2.1 对话状态：Agent 的"短期记忆"

对话状态记录了用户和 Agent 之间的对话历史。这是 Agent 最基本的状态。

```python
# 对话状态的结构
conversation_state = [
    {"role": "user", "content": "我叫张三"},
    {"role": "assistant", "content": "你好，张三！"},
    {"role": "user", "content": "我是 Python 开发者"},
    {"role": "assistant", "content": "了解，Python 是一门很棒的语言！"},
    {"role": "user", "content": "帮我推荐学习资源"},
    # ... 更多对话
]
```

对话状态的作用是让 Agent 记住之前的对话。当用户说"根据我的背景推荐"时，Agent 可以从对话状态中找到用户之前说过的话（"我是 Python 开发者"），从而给出个性化的推荐。

### 8.2.2 任务状态：Agent 的"工作日志"

任务状态记录了当前任务的执行状态。当 Agent 在执行一个复杂任务时，需要跟踪任务进度。

```python
# 任务状态的结构
task_state = {
    "current_task": "研究 AI Agent 的发展趋势",
    "start_time": "2024-01-15 10:30",
    "progress": 60,  # 完成百分比
    "subtasks": [
        {"name": "搜索资料", "status": "completed", "result": "找到 10 篇相关文章"},
        {"name": "阅读资料", "status": "completed", "result": "提取了 5 个关键趋势"},
        {"name": "整理大纲", "status": "in_progress", "result": ""},
        {"name": "撰写报告", "status": "pending", "result": ""},
    ],
    "intermediate_results": {
        "key_trends": ["趋势1", "趋势2", "趋势3"],
        "sources": ["来源1", "来源2"],
    }
}
```

任务状态的作用是让 Agent 能跟踪任务进度。当 Agent 在执行过程中"被打断"（比如用户问了另一个问题），它可以保存任务状态，之后回来继续执行。

### 8.2.3 环境状态：Agent 的"世界模型"

环境状态记录了外部环境的信息。这帮助 Agent 做出上下文感知的决策。

```python
# 环境状态的结构
environment_state = {
    "current_time": "2024-01-15 10:30",
    "user_location": "北京",
    "available_tools": ["search", "weather", "calculator"],
    "api_status": {
        "openai": "healthy",
        "weather_api": "healthy",
        "search_api": "degraded",
    },
    "user_preferences": {
        "language": "zh-CN",
        "response_length": "concise",
    }
}
```

环境状态的作用是让 Agent 能根据环境做出更好的决策。比如，如果天气 API 不可用，Agent 可以选择使用其他方式获取天气信息。

### 8.2.4 三种状态的关系

```
用户输入
    ↓
[环境状态] 提供上下文（时间、位置、可用工具）
    ↓
[对话状态] 提供历史信息（之前的对话内容）
    ↓
LLM 推理（基于上下文和历史）
    ↓
[任务状态] 更新进度（任务完成情况）
    ↓
执行行动
    ↓
更新所有状态
```

三种状态共同构成了 Agent 的"记忆系统"。对话状态让 Agent 记住"之前发生了什么"，任务状态让 Agent 知道"现在在做什么"，环境状态让 Agent 了解"外部世界是什么样的"。

---

## 8.3 对话历史管理：当记忆太长

### 8.3.1 问题的本质

LLM 的上下文窗口是有限的（比如 GPT-4o 是 128K Token）。随着对话进行，对话历史会不断增长。假设每轮对话消耗 500 Token，那么 256 轮对话后就会超出 128K 的限制。

更糟糕的是，对话历史只是上下文窗口的一部分。System Prompt、工具描述、RAG 结果等也需要占用空间。实际可用的空间可能只有 100K Token。

### 8.3.2 策略 1：滑动窗口——只记住最近的

最简单的策略：只保留最近的 N 条对话，丢弃早期的对话。

```python
class SlidingWindowMemory:
    """滑动窗口记忆：只保留最近的 N 条消息"""

    def __init__(self, max_messages: int = 20):
        self.max_messages = max_messages
        self.messages = []

    def add(self, role: str, content: str):
        """添加一条消息"""
        self.messages.append({"role": role, "content": content})

        # 如果超过限制，移除最早的消息
        if len(self.messages) > self.max_messages:
            self.messages = self.messages[-self.max_messages:]

    def get(self) -> list[dict]:
        """获取所有消息"""
        return self.messages.copy()

    def get_last_n(self, n: int) -> list[dict]:
        """获取最近的 N 条消息"""
        return self.messages[-n:]

    def clear(self):
        """清空所有消息"""
        self.messages.clear()
```

**优点：** 实现简单，性能好，内存占用可控。

**缺点：** 丢失早期重要信息。如果用户在第 1 轮说了"我是 Python 开发者"，到第 25 轮时这条信息就被丢掉了。

**适用场景：** 简单对话，不需要长期记忆。比如客服对话（通常不会太长）、一次性问答。

### 8.3.3 策略 2：摘要压缩——记住精华

当对话历史太长时，将早期的对话压缩为摘要，保留关键信息。

```python
class SummarizingMemory:
    """摘要压缩记忆：将旧消息压缩为摘要"""

    def __init__(self, llm_fn, max_messages: int = 20, summary_threshold: int = 15):
        """
        参数:
            llm_fn: 用于生成摘要的 LLM 函数
            max_messages: 最大消息数
            summary_threshold: 触发摘要的消息数阈值
        """
        self.llm_fn = llm_fn
        self.max_messages = max_messages
        self.summary_threshold = summary_threshold
        self.messages = []
        self.summary = ""

    def add(self, role: str, content: str):
        """添加一条消息"""
        self.messages.append({"role": role, "content": content})

        # 如果消息数超过阈值，触发摘要
        if len(self.messages) >= self.summary_threshold:
            self._compress()

    def _compress(self):
        """将旧消息压缩为摘要"""
        # 取出前一半的消息进行摘要
        old_messages = self.messages[:len(self.messages) // 2]

        # 使用 LLM 生成摘要
        messages_text = "\n".join([
            f"{msg['role']}: {msg['content']}"
            for msg in old_messages
        ])

        self.summary = self.llm_fn(
            f"请用 100 字以内摘要以下对话的关键信息：\n{messages_text}"
        )

        # 保留后一半的消息
        self.messages = self.messages[len(self.messages) // 2:]

    def get(self) -> list[dict]:
        """获取消息（包含摘要）"""
        result = []

        # 如果有摘要，添加到开头
        if self.summary:
            result.append({
                "role": "system",
                "content": f"之前的对话摘要：{self.summary}"
            })

        # 添加当前消息
        result.extend(self.messages)

        return result
```

**优点：** 保留关键信息，节省空间。即使对话很长，核心信息也不会丢失。

**缺点：** 摘要可能丢失细节，需要额外的 LLM 调用（增加成本和延迟）。

**适用场景：** 长对话，需要保留关键信息。比如研究助手（需要记住之前的研究发现）、个人助手（需要记住用户的偏好）。

### 8.3.4 策略 3：选择性保留——记住重要的

不是所有消息都一样重要。有些消息（比如用户的关键需求、重要的决策结果）应该保留，有些消息（比如闲聊、重复的信息）可以丢弃。

```python
class SelectiveMemory:
    """选择性记忆：根据重要性选择保留哪些消息"""

    def __init__(self, max_tokens: int = 50000):
        self.max_tokens = max_tokens
        self.messages = []

    def add(self, role: str, content: str, importance: float = 0.5):
        """
        添加消息

        参数:
            role: 消息角色
            content: 消息内容
            importance: 重要性评分（0-1），1 表示最重要
        """
        self.messages.append({
            "role": role,
            "content": content,
            "importance": importance,
        })

    def get_optimized(self) -> list[dict]:
        """返回优化后的消息列表"""
        if not self.messages:
            return []

        # 按重要性排序
        sorted_msgs = sorted(
            self.messages,
            key=lambda x: x["importance"],
            reverse=True,
        )

        # 保留到 token 限制内
        result = []
        total_tokens = 0
        for msg in sorted_msgs:
            # 粗略估计 Token 数（中文约 2 字符/Token，英文约 4 字符/Token）
            tokens = len(msg["content"]) // 2
            if total_tokens + tokens <= self.max_tokens:
                result.append({
                    "role": msg["role"],
                    "content": msg["content"],
                })
                total_tokens += tokens

        # 按原始顺序排序（保持对话的时序性）
        result.sort(key=lambda x: next(
            i for i, m in enumerate(self.messages)
            if m["content"] == x["content"]
        ))

        return result
```

**优点：** 智能保留重要信息，最大化利用上下文窗口。

**缺点：** 需要定义重要性评分规则，实现复杂。

**适用场景：** 复杂对话，需要保留关键上下文。比如代码审查（需要记住关键的设计决策）、项目管理（需要记住重要的里程碑）。

### 8.3.5 三种策略的对比

| 策略 | 原理 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|---------|
| 滑动窗口 | 只保留最近 N 条 | 简单、快速 | 丢失早期信息 | 短对话 |
| 摘要压缩 | 将旧消息压缩为摘要 | 保留关键信息 | 需要额外 LLM 调用 | 长对话 |
| 选择性保留 | 根据重要性选择 | 最大化信息保留 | 实现复杂 | 复杂对话 |

---

## 8.4 上下文窗口管理：在有限空间内最大化能力

### 8.4.1 Token 计数：知道用了多少空间

要管理上下文窗口，首先要能准确计算 Token 数。

```python
import tiktoken


class TokenCounter:
    """Token 计数器"""

    def __init__(self, model: str = "gpt-4o"):
        """
        参数:
            model: 模型名称，用于选择对应的分词器
        """
        self.enc = tiktoken.encoding_for_model(model)

    def count(self, text: str) -> int:
        """计算文本的 Token 数"""
        return len(self.enc.encode(text))

    def count_messages(self, messages: list[dict]) -> int:
        """计算消息列表的 Token 数"""
        total = 0
        for msg in messages:
            # 每条消息有固定的 overhead（约 4 个 Token）
            total += 4
            total += self.count(msg.get("content", ""))
        return total
```

### 8.4.2 智能裁剪：在窗口内塞入最多信息

当消息列表超出上下文窗口时，需要智能地裁剪，保留最重要的信息。

```python
class ContextManager:
    """上下文窗口管理器"""

    def __init__(self, model: str = "gpt-4o", max_tokens: int = 128000):
        self.counter = TokenCounter(model)
        self.max_tokens = max_tokens

    def fit_to_window(
        self,
        messages: list[dict],
        reserved_tokens: int = 4096,
    ) -> list[dict]:
        """
        将消息列表调整到上下文窗口内

        参数:
            messages: 消息列表
            reserved_tokens: 为输出预留的 Token 数

        返回:
            调整后的消息列表
        """
        available = self.max_tokens - reserved_tokens
        total = self.counter.count_messages(messages)

        if total <= available:
            return messages  # 没有超出限制

        # 策略：优先保留 system 消息和最近的消息
        result = []
        current_tokens = 0

        # 第一步：添加所有 system 消息（这些通常很重要）
        for msg in messages:
            if msg["role"] == "system":
                tokens = self.counter.count(msg["content"])
                if current_tokens + tokens <= available:
                    result.append(msg)
                    current_tokens += tokens

        # 第二步：从后往前添加非 system 消息（保留最近的）
        for msg in reversed(messages):
            if msg["role"] == "system":
                continue
            tokens = self.counter.count(msg["content"])
            if current_tokens + tokens <= available:
                # 插入到 system 消息之后
                insert_pos = len([m for m in result if m["role"] == "system"])
                result.insert(insert_pos, msg)
                current_tokens += tokens

        return result
```

### 8.4.3 选择性注入：只给 Agent 需要的信息

不是所有信息都需要注入到上下文中。根据当前任务选择性注入，可以最大化上下文窗口的利用效率。

```python
class SelectiveInjector:
    """选择性注入器：根据任务选择性注入信息"""

    def __init__(self, max_context_tokens: int = 8000):
        self.max_context_tokens = max_context_tokens

    def inject(
        self,
        system_prompt: str,
        conversation: list[dict],
        memory: list[str],
        tools: list[dict],
        task: str = "",
    ) -> list[dict]:
        """
        选择性注入信息

        参数:
            system_prompt: 系统提示词
            conversation: 对话历史
            memory: 记忆列表
            tools: 工具列表
            task: 当前任务（用于选择相关的记忆）

        返回:
            优化后的消息列表
        """
        messages = []

        # 1. System Prompt（必须）
        messages.append({"role": "system", "content": system_prompt})
        used_tokens = len(system_prompt) // 2

        # 2. 工具描述（如果还有空间）
        if tools and used_tokens < self.max_context_tokens * 0.3:
            tool_desc = self._format_tools(tools)
            messages.append({"role": "system", "content": tool_desc})
            used_tokens += len(tool_desc) // 2

        # 3. 相关记忆（选择最相关的几条）
        if memory and used_tokens < self.max_context_tokens * 0.5:
            relevant = self._select_relevant(memory, task, max_items=3)
            memory_desc = "\n".join([f"- {m}" for m in relevant])
            messages.append({
                "role": "system",
                "content": f"相关记忆：\n{memory_desc}"
            })
            used_tokens += len(memory_desc) // 2

        # 4. 对话历史（填充剩余空间）
        remaining = self.max_context_tokens - used_tokens
        context_manager = ContextManager(max_tokens=remaining * 2)  # Token 估算
        fitted_conversation = context_manager.fit_to_window(conversation)
        messages.extend(fitted_conversation)

        return messages

    def _format_tools(self, tools: list[dict]) -> str:
        """格式化工具描述"""
        lines = ["可用工具："]
        for tool in tools:
            lines.append(f"- {tool['name']}: {tool['description']}")
        return "\n".join(lines)

    def _select_relevant(
        self,
        memory: list[str],
        task: str,
        max_items: int = 3,
    ) -> list[str]:
        """选择与当前任务最相关的记忆"""
        if not task:
            # 没有指定任务，返回最近的记忆
            return memory[-max_items:]

        # 简单的关键词匹配（实际项目中可以用向量搜索）
        scored = []
        task_words = set(task.lower().split())
        for item in memory:
            item_words = set(item.lower().split())
            overlap = len(task_words & item_words)
            scored.append((overlap, item))

        # 按相关性排序
        scored.sort(key=lambda x: x[0], reverse=True)
        return [item for _, item in scored[:max_items]]
```

---

## 8.5 状态持久化：让 Agent 跨会话保持状态

### 8.5.1 为什么需要持久化

如果 Agent 的状态只保存在内存中，当程序重启时，所有状态都会丢失。用户之前告诉 Agent 的信息、任务进度、对话历史——全部消失。

持久化让 Agent 的状态保存到外部存储（文件、数据库），重启后可以恢复。

### 8.5.2 文件持久化

```python
import json
from pathlib import Path


class FilePersistence:
    """基于文件的状态持久化"""

    def __init__(self, storage_path: str = "./agent_state"):
        self.path = Path(storage_path)
        self.path.mkdir(parents=True, exist_ok=True)

    def save(self, agent_id: str, state: dict):
        """保存状态到文件"""
        file_path = self.path / f"{agent_id}.json"
        with open(file_path, "w", encoding="utf-8") as f:
            json.dump(state, f, ensure_ascii=False, indent=2)

    def load(self, agent_id: str) -> dict | None:
        """从文件加载状态"""
        file_path = self.path / f"{agent_id}.json"
        if file_path.exists():
            with open(file_path, "r", encoding="utf-8") as f:
                return json.load(f)
        return None

    def delete(self, agent_id: str):
        """删除状态文件"""
        file_path = self.path / f"{agent_id}.json"
        if file_path.exists():
            file_path.unlink()
```

**优点：** 实现简单，不需要额外的依赖。

**缺点：** 并发访问时可能有问题，不适合分布式系统。

**适用场景：** 单机部署的 Agent，用户量不大。

### 8.5.3 数据库持久化

```python
import sqlite3
import json


class DatabasePersistence:
    """基于数据库的状态持久化"""

    def __init__(self, db_path: str = "./agent_state.db"):
        self.conn = sqlite3.connect(db_path)
        self._init_db()

    def _init_db(self):
        """初始化数据库表"""
        self.conn.execute("""
            CREATE TABLE IF NOT EXISTS agent_state (
                agent_id TEXT PRIMARY KEY,
                state TEXT,
                updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        self.conn.commit()

    def save(self, agent_id: str, state: dict):
        """保存状态到数据库"""
        state_json = json.dumps(state, ensure_ascii=False)
        self.conn.execute(
            "INSERT OR REPLACE INTO agent_state (agent_id, state) VALUES (?, ?)",
            (agent_id, state_json),
        )
        self.conn.commit()

    def load(self, agent_id: str) -> dict | None:
        """从数据库加载状态"""
        cursor = self.conn.execute(
            "SELECT state FROM agent_state WHERE agent_id = ?",
            (agent_id,),
        )
        row = cursor.fetchone()
        if row:
            return json.loads(row[0])
        return None
```

**优点：** 支持并发访问，适合分布式系统。

**缺点：** 需要额外的依赖（数据库），实现稍复杂。

**适用场景：** 多用户部署的 Agent，生产环境。

---

## 8.6 完整的状态管理器

把上面的所有组件组合起来，我们得到一个完整的状态管理器：

```python
class AgentStateManager:
    """Agent 状态管理器：管理对话、任务和记忆"""

    def __init__(self, agent_id: str, persistence=None):
        self.agent_id = agent_id
        self.persistence = persistence or FilePersistence()

        # 对话状态
        self.conversation = []

        # 任务状态
        self.task_state = {
            "current_task": None,
            "progress": 0,
            "subtasks": [],
            "intermediate_results": {},
        }

        # 记忆
        self.memory = []

        # 加载已保存的状态
        self._load()

    def add_message(self, role: str, content: str):
        """添加对话消息"""
        self.conversation.append({"role": role, "content": content})
        self._save()

    def get_messages(self) -> list[dict]:
        """获取对话历史"""
        return self.conversation.copy()

    def update_task(self, **kwargs):
        """更新任务状态"""
        self.task_state.update(kwargs)
        self._save()

    def add_memory(self, content: str):
        """添加记忆"""
        self.memory.append(content)
        self._save()

    def search_memory(self, query: str, top_k: int = 3) -> list[str]:
        """搜索记忆"""
        # 简单实现：关键词匹配
        results = []
        for item in self.memory:
            if query.lower() in item.lower():
                results.append(item)
        return results[:top_k]

    def clear_conversation(self):
        """清空对话历史"""
        self.conversation = []
        self._save()

    def get_stats(self) -> dict:
        """获取状态统计"""
        return {
            "conversation_length": len(self.conversation),
            "memory_count": len(self.memory),
            "task_progress": self.task_state.get("progress", 0),
        }

    def _save(self):
        """保存状态"""
        state = {
            "conversation": self.conversation,
            "task_state": self.task_state,
            "memory": self.memory,
        }
        self.persistence.save(self.agent_id, state)

    def _load(self):
        """加载状态"""
        state = self.persistence.load(self.agent_id)
        if state:
            self.conversation = state.get("conversation", [])
            self.task_state = state.get("task_state", {})
            self.memory = state.get("memory", [])
```

---

## 8.7 常见坑

### 8.7.1 不清理旧状态

问题：内存溢出。对话历史无限增长，最终耗尽内存。

解决：设置 max_messages，定期清理。或者使用滑动窗口策略。

### 8.7.2 不持久化状态

问题：会话丢失。程序重启后，所有状态消失。

解决：实现状态持久化。根据场景选择文件或数据库。

### 8.7.3 Token 计数不准确

问题：超出上下文窗口。LLM 返回错误或截断输出。

解决：使用 tiktoken 准确计数。预留足够的输出空间。

### 8.7.4 摘要丢失关键信息

问题：压缩后的摘要丢失了重要细节。

解决：在摘要 Prompt 中强调"保留关键信息"。或者使用选择性保留策略。

---

## 8.8 练习题

### 概念理解题

1. **Agent 需要管理哪些类型的状态？** 每种状态的作用是什么？如果缺少某种状态会怎样？

2. **滑动窗口和摘要压缩各有什么优缺点？** 在什么场景下应该用哪种策略？

3. **如何设计状态持久化方案？** 文件持久化和数据库持久化各有什么优缺点？

4. **上下文窗口管理的核心挑战是什么？** 如何在有限空间内最大化 Agent 的能力？

5. **选择性注入的原理是什么？** 它如何帮助 Agent 在有限的上下文窗口内获得最多的信息？

### 动手实践题

1. **实现状态管理器：** 用本章的代码实现一个完整的 AgentStateManager。

2. **对话历史实验：** 让 Agent 进行 30 轮对话，观察对话历史的增长。分别测试滑动窗口和摘要压缩的效果。

3. **Token 计数实验：** 计算不同长度对话的 Token 数，分析上下文窗口的使用情况。

### 思考题

1. 如果 Agent 需要"忘记"某些信息（如用户要求删除对话记录），应该如何实现？

2. 在 Multi-Agent 系统中，多个 Agent 如何共享状态？有哪些挑战？

3. 如何设计一个状态管理系统，让它既能记住重要信息，又能控制内存使用？

---

## 8.9 实战任务

**任务：实现一个带状态管理的对话 Agent，支持长对话**

**具体要求：**

1. 实现对话历史管理（至少支持 2 种策略：滑动窗口和摘要压缩）
2. 实现任务状态跟踪
3. 实现记忆存储和检索
4. 实现状态持久化（重启后状态不丢失）
5. 测试长对话（至少 30 轮），观察状态管理的效果

**评估标准：**

| 维度 | 权重 | 评分标准 |
|------|------|---------|
| 对话管理 | 30% | 支持多种策略，效果良好 |
| 任务状态 | 20% | 能正确跟踪任务进度 |
| 记忆系统 | 20% | 能存储和检索记忆 |
| 持久化 | 15% | 重启后状态不丢失 |
| 代码质量 | 15% | 结构清晰，可维护 |

---

## 8.10 本章小结

### 核心知识点回顾

- **三种状态：** 对话状态（对话历史）、任务状态（任务进度）、环境状态（外部信息）。三者共同构成 Agent 的"记忆系统"。

- **对话管理策略：** 滑动窗口（简单但丢信息）、摘要压缩（保留关键但需 LLM）、选择性保留（智能但复杂）。根据场景选择合适的策略。

- **上下文窗口管理：** Token 计数（知道用了多少空间）、智能裁剪（在窗口内塞入最多信息）、选择性注入（只给 Agent 需要的信息）。

- **状态持久化：** 文件持久化（简单，适合单机）、数据库持久化（可靠，适合生产）。

### 关键公式

```
状态管理 = 对话状态 + 任务状态 + 环境状态
上下文管理 = Token 计数 + 智能裁剪 + 选择性注入
记忆保留 = 重要性 × 相关性 × 时效性
```

### 下一章预告

下一章我们将探讨 **Agent 的错误处理与容错** —— Agent 在执行过程中会遇到各种错误。没有容错机制的 Agent 在生产环境中就像一辆没有刹车的汽车，迟早会出事。你将学习如何设计健壮的错误处理策略。

---

*上一章：[第 7 章：Agent 的认知架构](../chapter-07/README.md)*
*下一章：[第 9 章：Agent 的错误处理与容错](../chapter-09/README.md)*
