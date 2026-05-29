# 第 13 章：Memory 系统 —— Agent 的记忆机制

---

## 学习目标

完成本章学习后，你将能够：

1. 理解 Agent 记忆系统的本质需求和设计原则
2. 掌握短期记忆、长期记忆、工作记忆三种记忆类型的区别和实现
3. 能够从零实现一个带有记忆系统的 Agent
4. 理解记忆的存储、检索、遗忘机制
5. 掌握对话历史管理的工程实践
6. 了解记忆系统在实际产品中的应用模式

## 核心问题

1. 为什么 Agent 需要记忆系统？没有记忆会怎样？
2. 短期记忆和长期记忆的区别是什么？如何实现？
3. 记忆系统如何影响 Agent 的用户体验？

---

## 13.1 为什么 Agent 需要记忆

### 13.1.1 没有记忆的 Agent

一个没有记忆的 Agent，每次对话都是"初次见面"：

```
用户第 1 轮：我叫张三，是一名 Python 工程师
Agent：你好张三！
用户第 2 轮：帮我写一个排序算法
Agent：好的，请问你是做什么工作的？（因为它忘了）
```

这就是没有记忆的 Agent 的典型表现——它无法记住之前的对话内容，导致用户体验极差。

### 13.1.2 记忆的本质

记忆的本质是**信息的持久化和检索**。Agent 的记忆系统需要解决三个核心问题：

| 问题 | 描述 | 解决方案 |
|------|------|----------|
| 存储什么 | 哪些信息值得记住 | 信息筛选与提取 |
| 如何存储 | 用什么结构保存信息 | 数据结构设计 |
| 如何检索 | 怎么快速找到相关记忆 | 索引与搜索 |

### 13.1.3 记忆的分类

Agent 的记忆可以从多个维度分类：

**按时间维度：**
- **短期记忆（Short-term Memory）：** 当前对话的上下文，通常就是对话历史
- **长期记忆（Long-term Memory）** 跨对话的持久化信息
- **工作记忆（Working Memory）** 当前任务的中间状态和推理过程

**按内容维度：**
- **事实记忆：** 用户的姓名、偏好、历史行为
- **程序记忆：** 用户习惯的工作流程
- **语义记忆：** 通用知识和领域知识
- **情景记忆：** 具体的对话事件和经历

---

## 13.2 短期记忆：对话历史管理

### 13.2.1 什么是短期记忆

短期记忆就是 Agent 在当前会话中维护的对话历史。它是最基础的记忆形式，直接决定了 Agent 能"记住"多少当前对话的内容。

```python
# 最简单的短期记忆实现
class ShortTermMemory:
    """基于列表的短期记忆"""
    
    def __init__(self):
        self.messages: list[dict] = []
    
    def add(self, role: str, content: str):
        """添加一条消息"""
        self.messages.append({
            "role": role,
            "content": content,
            "timestamp": time.time()
        })
    
    def get_all(self) -> list[dict]:
        """获取所有消息"""
        return self.messages.copy()
    
    def get_recent(self, n: int) -> list[dict]:
        """获取最近 n 条消息"""
        return self.messages[-n:] if len(self.messages) >= n else self.messages
    
    def clear(self):
        """清空记忆"""
        self.messages.clear()
```

### 13.2.2 为什么需要管理短期记忆

LLM 的上下文窗口是有限的。当对话历史超过上下文窗口的限制，就必须进行管理：

```
上下文窗口限制：128K Token
System Prompt：~1K Token
每轮对话：~500 Token
最大对话轮数：~250 轮（理论值）

实际可用轮数远少于此，因为：
1. 工具调用返回的结果可能很长
2. 长文本会消耗更多 Token
3. 需要预留输出空间
```

### 13.2.3 滑动窗口策略

最常用的短期记忆管理策略是滑动窗口——保留最近的 N 条消息：

```python
class SlidingWindowMemory:
    """滑动窗口记忆管理"""
    
    def __init__(self, max_messages: int = 20, max_tokens: int = 4000):
        self.max_messages = max_messages
        self.max_tokens = max_tokens
        self.messages: list[dict] = []
    
    def add(self, role: str, content: str):
        """添加消息并自动管理窗口"""
        self.messages.append({
            "role": role,
            "content": content
        })
        
        # 策略1：按消息数量裁剪
        if len(self.messages) > self.max_messages:
            self.messages = self.messages[-self.max_messages:]
        
        # 策略2：按 Token 数量裁剪
        while self._estimate_tokens() > self.max_tokens and len(self.messages) > 1:
            self.messages.pop(0)  # 移除最早的消息
    
    def _estimate_tokens(self) -> int:
        """估算当前消息的 Token 数（粗略估计）"""
        total_chars = sum(len(msg["content"]) for msg in self.messages)
        return total_chars // 2  # 中文大约 2 字符 = 1 Token
    
    def get_messages(self) -> list[dict]:
        """获取当前窗口内的消息"""
        return self.messages.copy()
```

**适用场景：** 大多数对话型 Agent 的基础记忆管理。

**局限性：** 裁剪掉的消息会永久丢失，可能丢失重要信息。

### 13.2.4 摘要压缩策略

在裁剪之前，先对旧消息进行摘要压缩，保留关键信息：

```python
class SummarizingMemory:
    """摘要压缩记忆管理"""
    
    def __init__(self, llm_client, max_messages: int = 20):
        self.llm_client = llm_client
        self.max_messages = max_messages
        self.messages: list[dict] = []
        self.summary: str = ""  # 历史摘要
    
    def add(self, role: str, content: str):
        """添加消息"""
        self.messages.append({
            "role": role,
            "content": content
        })
        
        if len(self.messages) > self.max_messages:
            self._compress()
    
    def _compress(self):
        """压缩旧消息为摘要"""
        # 保留最近 5 条消息不动
        keep_recent = 5
        to_summarize = self.messages[:-keep_recent]
        self.messages = self.messages[-keep_recent:]
        
        # 生成摘要
        prompt = f"""请将以下对话内容总结为简洁的摘要，保留关键信息：
        
已有的摘要：{self.summary}

需要总结的对话：
{self._format_messages(to_summarize)}

请输出新的摘要："""
        
        self.summary = self.llm_client.generate(prompt)
    
    def get_messages(self) -> list[dict]:
        """获取消息（包含摘要）"""
        result = []
        
        # 如果有摘要，作为系统消息添加
        if self.summary:
            result.append({
                "role": "system",
                "content": f"之前的对话摘要：{self.summary}"
            })
        
        result.extend(self.messages)
        return result
    
    def _format_messages(self, messages: list[dict]) -> str:
        """格式化消息为文本"""
        return "\n".join(
            f"{msg['role']}: {msg['content']}" 
            for msg in messages
        )
```

**适用场景：** 长对话场景，需要保留历史信息但又不能无限扩展上下文。

**注意：** 摘要过程会调用 LLM，增加延迟和成本。

---

## 13.3 长期记忆：跨对话的信息持久化

### 13.3.1 什么是长期记忆

长期记忆是跨对话持久化存储的信息。即使用户关闭了对话窗口，下次回来时 Agent 仍然能记住之前的信息。

```
短期记忆：当前对话的消息列表 → 对话结束就丢失
长期记忆：持久化存储的用户信息 → 永久保存
```

### 13.3.2 长期记忆的存储方案

**方案1：基于文件的存储**

最简单的长期记忆实现：

```python
import json
import os
from pathlib import Path

class FileBasedMemory:
    """基于文件的长期记忆"""
    
    def __init__(self, storage_dir: str = "./memory"):
        self.storage_dir = Path(storage_dir)
        self.storage_dir.mkdir(exist_ok=True)
    
    def save_user_memory(self, user_id: str, key: str, value: str):
        """保存用户记忆"""
        file_path = self.storage_dir / f"{user_id}.json"
        
        # 读取现有记忆
        memory = self.load_user_memory(user_id)
        
        # 更新记忆
        memory[key] = {
            "value": value,
            "timestamp": time.time()
        }
        
        # 写入文件
        with open(file_path, "w", encoding="utf-8") as f:
            json.dump(memory, f, ensure_ascii=False, indent=2)
    
    def load_user_memory(self, user_id: str) -> dict:
        """加载用户记忆"""
        file_path = self.storage_dir / f"{user_id}.json"
        
        if file_path.exists():
            with open(file_path, "r", encoding="utf-8") as f:
                return json.load(f)
        
        return {}
    
    def search_memory(self, user_id: str, query: str) -> list[dict]:
        """搜索用户记忆"""
        memory = self.load_user_memory(user_id)
        results = []
        
        for key, item in memory.items():
            if query.lower() in key.lower() or query.lower() in item["value"].lower():
                results.append({
                    "key": key,
                    "value": item["value"],
                    "timestamp": item["timestamp"]
                })
        
        return results
```

**方案2：基于数据库的存储**

适合生产环境的长期记忆实现：

```python
import sqlite3
from datetime import datetime

class DatabaseMemory:
    """基于 SQLite 的长期记忆"""
    
    def __init__(self, db_path: str = "memory.db"):
        self.conn = sqlite3.connect(db_path)
        self._init_db()
    
    def _init_db(self):
        """初始化数据库"""
        self.conn.execute("""
            CREATE TABLE IF NOT EXISTS memories (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id TEXT NOT NULL,
                category TEXT NOT NULL,
                key TEXT NOT NULL,
                value TEXT NOT NULL,
                importance REAL DEFAULT 0.5,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                accessed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                access_count INTEGER DEFAULT 0
            )
        """)
        self.conn.commit()
    
    def save(self, user_id: str, category: str, key: str, value: str, importance: float = 0.5):
        """保存记忆"""
        # 检查是否已存在
        cursor = self.conn.execute(
            "SELECT id FROM memories WHERE user_id=? AND category=? AND key=?",
            (user_id, category, key)
        )
        existing = cursor.fetchone()
        
        if existing:
            # 更新
            self.conn.execute(
                "UPDATE memories SET value=?, importance=?, accessed_at=CURRENT_TIMESTAMP WHERE id=?",
                (value, importance, existing[0])
            )
        else:
            # 插入
            self.conn.execute(
                "INSERT INTO memories (user_id, category, key, value, importance) VALUES (?, ?, ?, ?, ?)",
                (user_id, category, key, value, importance)
            )
        
        self.conn.commit()
    
    def recall(self, user_id: str, category: str = None, limit: int = 10) -> list[dict]:
        """召回记忆"""
        if category:
            cursor = self.conn.execute(
                "SELECT key, value, importance, created_at FROM memories WHERE user_id=? AND category=? ORDER BY importance DESC, accessed_at DESC LIMIT ?",
                (user_id, category, limit)
            )
        else:
            cursor = self.conn.execute(
                "SELECT key, value, importance, created_at FROM memories WHERE user_id=? ORDER BY importance DESC, accessed_at DESC LIMIT ?",
                (user_id, limit)
            )
        
        results = []
        for row in cursor.fetchall():
            results.append({
                "key": row[0],
                "value": row[1],
                "importance": row[2],
                "created_at": row[3]
            })
        
        # 更新访问时间和次数
        for r in results:
            self.conn.execute(
                "UPDATE memories SET accessed_at=CURRENT_TIMESTAMP, access_count=access_count+1 WHERE user_id=? AND key=?",
                (user_id, r["key"])
            )
        self.conn.commit()
        
        return results
    
    def forget(self, user_id: str, key: str):
        """遗忘记忆"""
        self.conn.execute(
            "DELETE FROM memories WHERE user_id=? AND key=?",
            (user_id, key)
        )
        self.conn.commit()
```

### 13.3.3 记忆的分类存储

在实际应用中，记忆通常按类别组织：

```python
class CategorizedMemory:
    """分类记忆管理"""
    
    CATEGORIES = {
        "user_profile": "用户个人信息",
        "preferences": "用户偏好",
        "facts": "用户提到的事实",
        "conversations": "重要对话摘要",
        "tasks": "任务相关记忆"
    }
    
    def __init__(self, db: DatabaseMemory):
        self.db = db
    
    def remember_user_info(self, user_id: str, info_type: str, value: str):
        """记住用户信息"""
        importance_map = {
            "name": 0.9,
            "job": 0.8,
            "preference": 0.7,
            "fact": 0.5,
        }
        importance = importance_map.get(info_type, 0.5)
        self.db.save(user_id, "user_profile", info_type, value, importance)
    
    def get_user_context(self, user_id: str) -> str:
        """获取用户上下文（注入到 System Prompt）"""
        memories = self.db.recall(user_id, limit=20)
        
        if not memories:
            return ""
        
        context_parts = ["以下是你记住的关于这个用户的信息："]
        for mem in memories:
            context_parts.append(f"- {mem['key']}: {mem['value']}")
        
        return "\n".join(context_parts)
```

---

## 13.4 工作记忆：任务执行的临时状态

### 13.4.1 什么是工作记忆

工作记忆是 Agent 在执行任务时维护的临时状态。它类似于人类工作记忆中的"心理草稿纸"：

```
任务：帮我规划一次北京旅行
工作记忆：
  - 已确认：出发日期 5月1日
  - 已确认：预算 5000元
  - 待确认：住宿偏好
  - 中间结果：已找到3个候选酒店
  - 待执行：比较酒店价格
```

### 13.4.2 工作记忆的实现

```python
from dataclasses import dataclass, field
from typing import Any
from enum import Enum

class TaskStatus(Enum):
    PENDING = "pending"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"
    FAILED = "failed"

@dataclass
class WorkingMemory:
    """工作记忆"""
    
    task_id: str
    task_description: str
    status: TaskStatus = TaskStatus.PENDING
    
    # 已确认的信息
    confirmed_facts: dict = field(default_factory=dict)
    
    # 中间结果
    intermediate_results: list = field(default_factory=list)
    
    # 待处理事项
    pending_items: list = field(default_factory=list)
    
    # 错误记录
    errors: list = field(default_factory=list)
    
    def add_fact(self, key: str, value: Any):
        """添加已确认的事实"""
        self.confirmed_facts[key] = value
    
    def add_result(self, result: str):
        """添加中间结果"""
        self.intermediate_results.append({
            "content": result,
            "timestamp": time.time()
        })
    
    def add_pending(self, item: str):
        """添加待处理事项"""
        self.pending_items.append(item)
    
    def complete_pending(self, item: str, result: str):
        """完成待处理事项"""
        if item in self.pending_items:
            self.pending_items.remove(item)
            self.add_result(result)
    
    def to_prompt(self) -> str:
        """转换为 Prompt 格式"""
        lines = [f"## 当前任务：{self.task_description}"]
        lines.append(f"状态：{self.status.value}")
        
        if self.confirmed_facts:
            lines.append("\n### 已确认信息：")
            for k, v in self.confirmed_facts.items():
                lines.append(f"- {k}: {v}")
        
        if self.intermediate_results:
            lines.append("\n### 中间结果：")
            for r in self.intermediate_results[-5:]:  # 只显示最近5个
                lines.append(f"- {r['content']}")
        
        if self.pending_items:
            lines.append("\n### 待处理：")
            for item in self.pending_items:
                lines.append(f"- [ ] {item}")
        
        return "\n".join(lines)
```

---

## 13.5 记忆系统的完整实现

### 13.5.1 统一记忆架构

将三种记忆整合到一个统一的架构中：

```python
class AgentMemorySystem:
    """Agent 统一记忆系统"""
    
    def __init__(self, llm_client, user_id: str):
        self.user_id = user_id
        self.llm_client = llm_client
        
        # 短期记忆：对话历史
        self.short_term = SlidingWindowMemory(max_messages=20)
        
        # 长期记忆：持久化存储
        self.long_term = DatabaseMemory()
        
        # 工作记忆：当前任务状态
        self.working: WorkingMemory | None = None
    
    def add_message(self, role: str, content: str):
        """添加消息到短期记忆"""
        self.short_term.add(role, content)
        
        # 自动提取值得长期保存的信息
        if role == "user":
            self._extract_and_save(content)
    
    def _extract_and_save(self, content: str):
        """从用户消息中提取值得保存的信息"""
        prompt = f"""分析以下用户消息，提取值得长期记住的信息。
        
用户消息：{content}

如果有值得记住的信息，输出 JSON 格式：
{{"key": "信息类型", "value": "信息内容", "importance": 0.0-1.0}}

如果没有值得记住的信息，输出：{{"key": null}}

只输出 JSON，不要其他内容："""
        
        response = self.llm_client.generate(prompt)
        
        try:
            data = json.loads(response)
            if data.get("key"):
                self.long_term.save(
                    self.user_id, 
                    "auto_extracted", 
                    data["key"], 
                    data["value"],
                    data.get("importance", 0.5)
                )
        except json.JSONDecodeError:
            pass
    
    def get_context_for_llm(self) -> list[dict]:
        """获取发送给 LLM 的完整上下文"""
        messages = []
        
        # 1. 长期记忆作为系统消息
        user_context = self.long_term.recall(self.user_id)
        if user_context:
            context_text = "\n".join(
                f"- {m['key']}: {m['value']}" 
                for m in user_context
            )
            messages.append({
                "role": "system",
                "content": f"用户信息：\n{context_text}"
            })
        
        # 2. 工作记忆（如果有）
        if self.working:
            messages.append({
                "role": "system",
                "content": self.working.to_prompt()
            })
        
        # 3. 短期记忆（对话历史）
        messages.extend(self.short_term.get_messages())
        
        return messages
    
    def start_task(self, task_description: str):
        """开始新任务"""
        self.working = WorkingMemory(
            task_id=f"task_{int(time.time())}",
            task_description=task_description,
            status=TaskStatus.IN_PROGRESS
        )
    
    def end_task(self):
        """结束任务"""
        if self.working:
            # 将重要结果保存到长期记忆
            if self.working.intermediate_results:
                summary = "\n".join(
                    r["content"] for r in self.working.intermediate_results
                )
                self.long_term.save(
                    self.user_id,
                    "task_results",
                    f"task_{self.working.task_id}",
                    summary,
                    importance=0.6
                )
            self.working = None
```

### 13.5.2 记忆的自动提取

从对话中自动提取值得保存的信息：

```python
class MemoryExtractor:
    """记忆信息提取器"""
    
    EXTRACTION_PROMPT = """分析以下对话，提取用户的个人信息和偏好。

对话内容：
{conversation}

请提取以下类型的信息：
1. 个人信息（姓名、职业、年龄等）
2. 偏好（喜欢/不喜欢的东西）
3. 重要事实（用户提到的具体事件或信息）

以 JSON 数组格式输出，每条包含：
- type: 信息类型
- key: 信息键名
- value: 信息值
- importance: 重要性（0-1）

如果没有值得提取的信息，输出空数组 []"""

    def __init__(self, llm_client):
        self.llm_client = llm_client
    
    def extract(self, conversation: str) -> list[dict]:
        """从对话中提取记忆"""
        prompt = self.EXTRACTION_PROMPT.format(conversation=conversation)
        response = self.llm_client.generate(prompt)
        
        try:
            return json.loads(response)
        except json.JSONDecodeError:
            return []
```

---

## 13.6 记忆的检索策略

### 13.6.1 相关性检索

不是所有记忆都需要加载到上下文中，需要根据当前对话检索相关的记忆：

```python
class MemoryRetriever:
    """记忆检索器"""
    
    def __init__(self, memory_db: DatabaseMemory):
        self.db = memory_db
    
    def retrieve(self, user_id: str, current_message: str, top_k: int = 5) -> list[dict]:
        """检索相关记忆"""
        # 获取所有记忆
        all_memories = self.db.recall(user_id, limit=100)
        
        if not all_memories:
            return []
        
        # 简单的关键词匹配（生产环境应使用向量搜索）
        scored_memories = []
        for mem in all_memories:
            score = self._compute_relevance(current_message, mem["key"], mem["value"])
            scored_memories.append((score, mem))
        
        # 按分数排序
        scored_memories.sort(key=lambda x: x[0], reverse=True)
        
        return [mem for _, mem in scored_memories[:top_k]]
    
    def _compute_relevance(self, query: str, key: str, value: str) -> float:
        """计算相关性分数"""
        # 简单的 TF-like 评分
        query_words = set(query.lower().split())
        key_words = set(key.lower().split())
        value_words = set(value.lower().split())
        
        all_words = key_words | value_words
        overlap = query_words & all_words
        
        if not all_words:
            return 0.0
        
        return len(overlap) / len(query_words) if query_words else 0.0
```

### 13.6.2 基于向量的语义检索

使用 Embedding 进行语义相似度检索：

```python
import numpy as np

class SemanticMemoryRetriever:
    """基于语义相似度的记忆检索器"""
    
    def __init__(self, embedding_client):
        self.embedding_client = embedding_client
        self.memory_store: list[dict] = []
        self.embeddings: np.ndarray = None
    
    def index_memories(self, memories: list[dict]):
        """为记忆建立索引"""
        self.memory_store = memories
        
        # 生成所有记忆的 Embedding
        texts = [f"{m['key']}: {m['value']}" for m in memories]
        self.embeddings = np.array([
            self.embedding_client.embed(text) for text in texts
        ])
    
    def retrieve(self, query: str, top_k: int = 5) -> list[dict]:
        """语义检索"""
        if self.embeddings is None or len(self.memory_store) == 0:
            return []
        
        # 计算查询的 Embedding
        query_embedding = np.array(self.embedding_client.embed(query))
        
        # 计算余弦相似度
        similarities = np.dot(self.embeddings, query_embedding) / (
            np.linalg.norm(self.embeddings, axis=1) * np.linalg.norm(query_embedding)
        )
        
        # 获取 Top-K
        top_indices = np.argsort(similarities)[-top_k:][::-1]
        
        results = []
        for idx in top_indices:
            results.append({
                **self.memory_store[idx],
                "similarity": float(similarities[idx])
            })
        
        return results
```

---

## 13.7 记忆的遗忘机制

### 13.7.1 为什么需要遗忘

记忆不能无限增长，需要遗忘机制来管理记忆容量：

1. **空间限制：** 存储空间有限
2. **检索效率：** 记忆越多，检索越慢
3. **信息质量：** 过时的信息可能误导 Agent

### 13.7.2 遗忘策略

```python
class MemoryForgetManager:
    """记忆遗忘管理器"""
    
    def __init__(self, db: DatabaseMemory, max_memories: int = 1000):
        self.db = db
        self.max_memories = max_memories
    
    def should_forget(self, user_id: str) -> bool:
        """检查是否需要遗忘"""
        cursor = self.db.conn.execute(
            "SELECT COUNT(*) FROM memories WHERE user_id=?",
            (user_id,)
        )
        count = cursor.fetchone()[0]
        return count > self.max_memories
    
    def forget_low_importance(self, user_id: str, keep_count: int = 500):
        """遗忘低重要性的记忆"""
        self.db.conn.execute("""
            DELETE FROM memories 
            WHERE user_id=? AND id NOT IN (
                SELECT id FROM memories 
                WHERE user_id=? 
                ORDER BY importance DESC, accessed_at DESC 
                LIMIT ?
            )
        """, (user_id, user_id, keep_count))
        self.db.conn.commit()
    
    def forget_old_memories(self, user_id: str, days: int = 90):
        """遗忘过期的记忆"""
        self.db.conn.execute("""
            DELETE FROM memories 
            WHERE user_id=? 
            AND importance < 0.7 
            AND accessed_at < datetime('now', ? || ' days')
        """, (user_id, f"-{days}"))
        self.db.conn.commit()
```

---

## 13.8 常见坑

### 13.8.1 把所有对话历史都保存

**问题：** 不加筛选地保存所有对话内容，导致存储爆炸。

**解决：** 只保存有价值的信息，对对话内容进行筛选。

### 13.8.2 记忆检索不考虑时效性

**问题：** 检索时只考虑相关性，不考虑时间，可能返回过时信息。

**解决：** 在检索时引入时间衰减因子。

### 13.8.3 忽略记忆的隐私问题

**问题：** 保存了用户的敏感信息（密码、身份证号等）。

**解决：** 在保存前进行敏感信息过滤。

```python
import re

def filter_sensitive_info(text: str) -> str:
    """过滤敏感信息"""
    # 过滤邮箱
    text = re.sub(r'\b[\w.-]+@[\w.-]+\.\w+\b', '[EMAIL]', text)
    # 过滤手机号
    text = re.sub(r'1[3-9]\d{9}', '[PHONE]', text)
    # 过滤身份证号
    text = re.sub(r'\d{17}[\dXx]', '[ID_CARD]', text)
    return text
```

### 13.8.4 过度依赖 LLM 进行记忆提取

**问题：** 每次对话都调用 LLM 提取记忆，增加延迟和成本。

**解决：** 使用规则引擎进行初步筛选，只有高价值信息才调用 LLM。

---

## 13.9 练习题

### 练习1：实现基础记忆系统
实现一个支持短期记忆和长期记忆的 Agent，要求：
- 短期记忆支持滑动窗口管理
- 长期记忆支持按用户存储
- 能自动从对话中提取用户信息

### 练习2：记忆检索优化
基于练习1的记忆系统，添加记忆检索功能：
- 根据当前对话内容检索相关记忆
- 只将最相关的记忆注入到上下文中
- 评估不同检索策略的效果

### 练习3：记忆的遗忘机制
为记忆系统添加遗忘功能：
- 实现基于重要性和时间的遗忘策略
- 实现相似记忆的合并
- 监控记忆使用情况

---

## 13.10 实战任务

### 任务：构建一个有记忆的个人助手

**目标：** 构建一个能记住用户信息的个人助手 Agent。

**要求：**
1. 实现用户信息的自动提取和存储
2. 在对话开始时加载用户的相关记忆
3. 支持用户查看和管理自己的记忆
4. 实现记忆的定期清理和合并

**参考实现框架：**

```python
class PersonalAssistant:
    """有记忆的个人助手"""
    
    def __init__(self, llm_client, user_id: str):
        self.memory = AgentMemorySystem(llm_client, user_id)
        self.llm_client = llm_client
    
    def chat(self, user_message: str) -> str:
        # 1. 添加用户消息到记忆
        self.memory.add_message("user", user_message)
        
        # 2. 获取完整上下文
        context = self.memory.get_context_for_llm()
        
        # 3. 生成回复
        response = self.llm_client.chat(context)
        
        # 4. 保存助手回复
        self.memory.add_message("assistant", response)
        
        return response
    
    def show_memories(self) -> str:
        """显示当前记住的用户信息"""
        memories = self.memory.long_term.recall(self.memory.user_id)
        
        if not memories:
            return "我还没有记住任何关于你的信息。"
        
        result = "我记住了以下关于你的信息：\n"
        for mem in memories:
            result += f"- {mem['key']}: {mem['value']}\n"
        
        return result
    
    def forget_me(self, key: str = None):
        """忘记某些信息"""
        if key:
            self.memory.long_term.forget(self.memory.user_id, key)
        else:
            # 清除所有记忆
            self.memory.long_term.clear_user(self.memory.user_id)
```

---

## 13.11 本章小结

- **记忆是 Agent 的核心能力之一**，没有记忆的 Agent 每次对话都是"初次见面"
- **短期记忆**是当前对话的历史，通过滑动窗口或摘要压缩管理
- **长期记忆**是跨对话的持久化信息，通过文件或数据库存储
- **工作记忆**是任务执行的临时状态，帮助 Agent 跟踪任务进度
- **记忆检索**需要根据当前上下文选择相关记忆，避免信息过载
- **遗忘机制**是记忆系统的重要组成部分，防止信息过载
- **自动提取**可以从对话中提取有价值的信息，减少人工配置
- 记忆系统的**隐私保护**不可忽视，需要过滤敏感信息

