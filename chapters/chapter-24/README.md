# 第 24 章 Multi-Agent 通信 —— 消息传递与共享状态

> "沟通是协作的桥梁。在 Multi-Agent 系统中，Agent 之间的通信方式直接决定了整个系统的效率和可靠性。就像人类社会一样，信息如何流动，往往比信息本身更重要。"

---

## 学习目标

通过本章的学习，你将能够：

1. 深入理解 Multi-Agent 系统中的通信模型和机制
2. 掌握消息传递、共享黑板、事件驱动等不同通信模式的原理与实现
3. 学会设计结构化的 Agent 间消息格式
4. 理解同步通信与异步通信的区别和适用场景
5. 使用 Python 实现多种通信模式的 Multi-Agent 系统
6. 掌握通信过程中的错误处理和容错机制
7. 了解通信优化的策略和技术

---

## 核心问题

- Agent 之间如何高效、可靠地传递信息？
- 不同的通信模式分别适用于什么场景？
- 如何设计一个既灵活又可控的通信协议？
- 在大规模 Multi-Agent 系统中，通信的扩展性如何保证？
- 如何处理通信过程中的延迟、丢失和错误？

---

## 24.1 通信：Multi-Agent 系统的血液

### 24.1.1 为什么通信如此重要

如果说 Agent 是 Multi-Agent 系统的"器官"，那么通信就是连接这些器官的"血管"。没有高效的血液循环，再强壮的器官也无法发挥作用。同样，没有高效的通信机制，再智能的 Agent 也无法形成有效的协作。

在上一章中，我们搭建了一个简单的 Multi-Agent 研究团队。在那个系统中，Agent 之间的通信是通过协调器来中转的——研究员把结果交给协调器，协调器再转交给分析师。这种方式虽然简单，但它只是通信的最基础形式。当我们面对更复杂的场景时——比如多个 Agent 需要同时协作、需要实时交换信息、需要处理通信故障——我们需要更丰富的通信机制。

### 24.1.2 通信的本质

从本质上说，Multi-Agent 之间的通信就是在 Agent 之间传递信息。这个看似简单的定义背后，蕴含着丰富的技术内涵。

传递什么信息？可以是任务指令（"请分析这份数据"）、数据结果（"分析结果如下..."）、状态更新（"我已完成任务"）、请求协助（"我遇到了困难，需要帮助"）、反馈意见（"你的结果有以下问题..."）等等。

通过什么方式传递？可以是直接的函数调用、消息队列、共享内存、文件系统，甚至可以通过网络协议。

以什么格式传递？可以是纯文本、JSON、Protocol Buffers、自定义的结构化格式等等。

这些看似基础的问题，实际上对系统的性能、可靠性和可维护性有着深远的影响。接下来，我们将逐一深入探讨。

---

## 24.2 四大通信模式

### 24.2.1 消息传递模式（Message Passing）

消息传递是最直接、最直观的通信方式。就像人与人之间发消息一样，Agent A 构造一条消息，直接发送给 Agent B。Agent B 收到消息后进行处理，然后可能回复一条消息给 Agent A。

这种模式的核心特点是：信息的流动是显式的、可控的。每个 Agent 都清楚地知道自己的消息发给了谁、收到了谁的消息。这种透明性使得系统的调试和追踪变得相对容易。

让我们先看看最简单的消息传递实现：

```python
"""
第 24 章：消息传递模式
最基础的 Multi-Agent 通信方式
"""
import os
import json
import time
from dataclasses import dataclass, field, asdict
from typing import Optional, Callable
from openai import OpenAI


# ============================================================
# 消息定义
# ============================================================

@dataclass
class AgentMessage:
    """
    Agent 间传递的消息格式。
    
    使用 dataclass 来定义消息结构，确保所有消息都有统一的格式。
    这是通信协议的基础——所有 Agent 都必须遵守这个格式。
    
    Attributes:
        sender: 发送者标识
        receiver: 接收者标识
        msg_type: 消息类型（task / result / feedback / error / status）
        content: 消息内容（可以是文本、JSON 字符串等）
        timestamp: 消息发送时间
        conversation_id: 会话标识，用于追踪多轮对话
        metadata: 额外的元数据
    """
    sender: str
    receiver: str
    msg_type: str  # task, result, feedback, error, status
    content: str
    timestamp: float = field(default_factory=time.time)
    conversation_id: str = ""
    metadata: dict = field(default_factory=dict)
    
    def to_json(self) -> str:
        """序列化为 JSON 字符串"""
        return json.dumps(asdict(self), ensure_ascii=False, indent=2)
    
    @classmethod
    def from_json(cls, json_str: str) -> "AgentMessage":
        """从 JSON 字符串反序列化"""
        data = json.loads(json_str)
        return cls(**data)
    
    def reply(self, content: str, msg_type: str = "result") -> "AgentMessage":
        """
        创建一条回复消息。
        
        这是一个便捷方法，自动设置发送者、接收者和会话 ID，
        确保回复消息与原始消息的关联性。
        """
        return AgentMessage(
            sender=self.receiver,
            receiver=self.sender,
            msg_type=msg_type,
            content=content,
            conversation_id=self.conversation_id,
            metadata={"reply_to": self.timestamp}
        )


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
# 消息路由器
# ============================================================

class MessageRouter:
    """
    消息路由器：负责在 Agent 之间传递消息。
    
    这是消息传递模式的核心组件。它维护了一个路由表，
    记录了每个 Agent 的标识和对应的消息处理函数。
    
    当路由器收到一条消息时，它会根据消息的 receiver 字段
    查找对应的 Agent，并将消息转发给它。
    
    设计思想：
    - Agent 不需要知道其他 Agent 的具体实现，只需要知道标识
    - 路由器是唯一的通信中枢，所有的消息都经过它
    - 路由器可以添加日志、验证、限流等横切关注点
    """
    
    def __init__(self):
        # 路由表：agent_name -> handler_function
        self._routes: dict[str, Callable] = {}
        # 消息历史：用于调试和审计
        self._message_log: list[AgentMessage] = []
        
    def register(self, agent_name: str, handler: Callable):
        """
        注册一个 Agent 的消息处理器。
        
        Args:
            agent_name: Agent 的唯一标识
            handler: 消息处理函数，接收 AgentMessage 参数
        """
        self._routes[agent_name] = handler
        print(f"  [路由器] 已注册 Agent: {agent_name}")
    
    def route(self, message: AgentMessage) -> Optional[AgentMessage]:
        """
        路由一条消息到目标 Agent。
        
        Args:
            message: 待路由的消息
        
        Returns:
            目标 Agent 的回复消息，如果没有回复则返回 None
        """
        # 记录消息日志
        self._message_log.append(message)
        
        receiver = message.receiver
        
        if receiver not in self._routes:
            print(f"  [路由器] 警告：找不到 Agent '{receiver}'，消息被丢弃。")
            return None
        
        print(f"  [路由器] {message.sender} → {receiver} [{message.msg_type}]")
        
        try:
            handler = self._routes[receiver]
            response = handler(message)
            if response:
                self._message_log.append(response)
            return response
        except Exception as e:
            error_msg = f"Agent '{receiver}' 处理消息时出错: {e}"
            print(f"  [路由器] {error_msg}")
            return message.reply(f"处理失败: {e}", msg_type="error")
    
    def broadcast(self, message: AgentMessage, exclude: list[str] = None) -> dict:
        """
        广播消息给所有已注册的 Agent（排除指定的 Agent）。
        
        Args:
            message: 待广播的消息
            exclude: 不接收广播的 Agent 列表
        
        Returns:
            所有 Agent 的回复
        """
        exclude = exclude or []
        responses = {}
        
        for agent_name, handler in self._routes.items():
            if agent_name in exclude or agent_name == message.sender:
                continue
            
            broadcast_msg = AgentMessage(
                sender=message.sender,
                receiver=agent_name,
                msg_type=message.msg_type,
                content=message.content,
                conversation_id=message.conversation_id,
                metadata=message.metadata
            )
            
            response = self.route(broadcast_msg)
            if response:
                responses[agent_name] = response
        
        return responses
    
    def get_log(self) -> list[AgentMessage]:
        """获取消息日志"""
        return self._message_log.copy()


# ============================================================
# 基于消息传递的 Agent 基类
# ============================================================

class MessagePassingAgent:
    """
    基于消息传递的 Agent 基类。
    
    每个 Agent 都有一个消息处理器，当收到消息时，
    Agent 会根据消息类型进行相应的处理。
    """
    
    def __init__(self, name: str, system_prompt: str, router: MessageRouter,
                 model: str = "gpt-4o"):
        self.name = name
        self.system_prompt = system_prompt
        self.model = model
        self.router = router
        self.conversation_history: dict[str, list] = {}  # conversation_id -> messages
        
        # 注册到路由器
        router.register(name, self.handle_message)
    
    def handle_message(self, message: AgentMessage) -> Optional[AgentMessage]:
        """
        消息处理入口。根据消息类型分发到不同的处理方法。
        
        这是消息传递模式中的关键设计——每个 Agent 都有一个统一的消息入口，
        在入口处根据消息类型进行分发。这种设计使得 Agent 的行为可预测且易于调试。
        """
        print(f"  [{self.name}] 收到来自 {message.sender} 的 {message.msg_type} 消息")
        
        # 获取或创建会话历史
        conv_id = message.conversation_id or f"default_{message.sender}"
        if conv_id not in self.conversation_history:
            self.conversation_history[conv_id] = []
        
        if message.msg_type == "task":
            return self._handle_task(message, conv_id)
        elif message.msg_type == "feedback":
            return self._handle_feedback(message, conv_id)
        elif message.msg_type == "query":
            return self._handle_query(message, conv_id)
        else:
            return message.reply(
                f"收到未知消息类型: {message.msg_type}",
                msg_type="error"
            )
    
    def _handle_task(self, message: AgentMessage, conv_id: str) -> AgentMessage:
        """处理任务消息：调用 LLM 完成任务"""
        self.conversation_history[conv_id].append({
            "role": "user",
            "content": f"来自 {message.sender} 的任务: {message.content}"
        })
        
        messages = [{"role": "system", "content": self.system_prompt}]
        messages.extend(self.conversation_history[conv_id][-10:])
        
        result = call_llm(messages, model=self.model)
        
        self.conversation_history[conv_id].append({
            "role": "assistant",
            "content": result
        })
        
        return message.reply(result, msg_type="result")
    
    def _handle_feedback(self, message: AgentMessage, conv_id: str) -> AgentMessage:
        """处理反馈消息：根据反馈调整后续行为"""
        self.conversation_history[conv_id].append({
            "role": "user",
            "content": f"来自 {message.sender} 的反馈: {message.content}"
        })
        
        messages = [{"role": "system", "content": self.system_prompt}]
        messages.extend(self.conversation_history[conv_id][-10:])
        
        prompt = f"以下是你之前的输出收到的反馈，请根据反馈进行修改：\n反馈内容: {message.content}\n请重新输出改进后的结果。"
        messages.append({"role": "user", "content": prompt})
        
        result = call_llm(messages, model=self.model)
        
        self.conversation_history[conv_id].append({
            "role": "assistant",
            "content": result
        })
        
        return message.reply(result, msg_type="result")
    
    def _handle_query(self, message: AgentMessage, conv_id: str) -> AgentMessage:
        """处理查询消息：回答其他 Agent 的询问"""
        messages = [
            {"role": "system", "content": self.system_prompt},
            {"role": "user", "content": f"来自 {message.sender} 的问题: {message.content}"}
        ]
        
        result = call_llm(messages, model=self.model)
        
        return message.reply(result, msg_type="result")


# ============================================================
# 演示：使用消息传递的 Multi-Agent 系统
# ============================================================

def demo_message_passing():
    """演示消息传递通信模式"""
    
    print("\n" + "="*60)
    print("演示：消息传递通信模式")
    print("="*60)
    
    # 创建消息路由器
    router = MessageRouter()
    
    # 创建 Agent
    researcher = MessagePassingAgent(
        name="研究员",
        system_prompt="你是一位技术研究员，擅长搜集和整理资料。请用简洁专业的语言回答。",
        router=router
    )
    
    writer = MessagePassingAgent(
        name="作家",
        system_prompt="你是一位技术作家，擅长将复杂的技术概念用通俗的语言解释清楚。",
        router=router
    )
    
    reviewer = MessagePassingAgent(
        name="审查者",
        system_prompt="你是一位严格的内容审查专家，擅长发现文章中的问题并提出改进建议。",
        router=router
    )
    
    # 模拟消息传递流程
    print("\n--- 第一轮：分配研究任务 ---")
    msg = AgentMessage(
        sender="协调者",
        receiver="研究员",
        msg_type="task",
        content="请研究 Multi-Agent 通信的主要模式，包括消息传递、共享黑板、事件驱动等方式。",
        conversation_id="task_001"
    )
    response = router.route(msg)
    if response:
        print(f"\n  研究员回复: {response.content[:200]}...")
    
    # 研究员的结果传递给作家
    print("\n--- 第二轮：将研究成果传递给作家 ---")
    if response:
        writer_msg = AgentMessage(
            sender="协调者",
            receiver="作家",
            msg_type="task",
            content=f"请根据以下研究资料，撰写一段关于 Multi-Agent 通信模式的说明文字：\n\n{response.content}",
            conversation_id="task_001"
        )
        writer_response = router.route(writer_msg)
        if writer_response:
            print(f"\n  作家回复: {writer_response.content[:200]}...")
    
    # 作家的结果传递给审查者
    print("\n--- 第三轮：将文章传递给审查者 ---")
    if writer_response:
        review_msg = AgentMessage(
            sender="协调者",
            receiver="审查者",
            msg_type="task",
            content=f"请审查以下文章段落的质量：\n\n{writer_response.content}",
            conversation_id="task_001"
        )
        review_response = router.route(review_msg)
        if review_response:
            print(f"\n  审查者回复: {review_response.content[:200]}...")
    
    # 展示消息日志
    print("\n--- 消息日志 ---")
    log = router.get_log()
    print(f"共传递了 {len(log)} 条消息")
    for msg in log:
        print(f"  {msg.sender} → {msg.receiver} [{msg.msg_type}]")


if __name__ == "__main__":
    demo_message_passing()
```

### 24.2.2 共享黑板模式（Blackboard Pattern）

消息传递模式虽然直观，但在某些场景下显得笨拙。想象一下，在一个多人协作的编辑场景中，五个 Agent 同时在编辑同一份文档。如果使用消息传递，每做一个修改都要通知其他四个 Agent，通信量会爆炸式增长。

共享黑板模式提供了一种更优雅的解决方案。它的灵感来源于现实世界中的黑板——教授在黑板上写字，所有学生都能看到；学生也可以走上讲台，在黑板上添加自己的内容。没有人需要特意去"告诉"别人黑板上写了什么，因为大家看到的是同一块黑板。

在 Multi-Agent 系统中，"黑板"就是一个共享的数据存储空间。Agent 可以从黑板上读取信息（观察其他 Agent 的工作成果），也可以向黑板上写入信息（分享自己的工作成果）。所有 Agent 都能看到黑板上的所有信息，但不需要知道是谁写的。

```python
"""
第 24 章：共享黑板模式
多个 Agent 通过共享数据空间协作
"""
import os
import json
import time
import threading
from typing import Any, Optional, Callable
from dataclasses import dataclass, field, asdict
from openai import OpenAI


# ============================================================
# 黑板系统
# ============================================================

@dataclass
class BlackboardEntry:
    """
    黑板上的一条记录。
    
    每条记录都有作者、键名、值和时间戳。
    其他 Agent 可以通过键名来查找需要的信息。
    """
    author: str
    key: str
    value: Any
    timestamp: float = field(default_factory=time.time)
    version: int = 1
    tags: list[str] = field(default_factory=list)


class Blackboard:
    """
    共享黑板：Multi-Agent 系统的共享数据空间。
    
    这个类实现了线程安全的共享存储，支持以下操作：
    - write: 写入或更新一条记录
    - read: 读取一条记录
    - read_all: 读取某个作者的所有记录
    - search: 按关键字搜索
    - list_keys: 列出所有键名
    
    黑板还支持事件通知：当有新记录写入时，可以通知订阅者。
    """
    
    def __init__(self):
        self._data: dict[str, BlackboardEntry] = {}
        self._lock = threading.RLock()  # 可重入锁，支持嵌套锁定
        self._subscribers: list[Callable] = []  # 订阅者列表
        self._history: list[dict] = []  # 操作历史
    
    def write(self, author: str, key: str, value: Any, tags: list[str] = None):
        """
        向黑板写入或更新一条记录。
        
        如果键名已存在，则更新值并增加版本号。
        
        Args:
            author: 写入者标识
            key: 记录的键名
            value: 记录的值
            tags: 可选的标签列表，用于分类和搜索
        """
        with self._lock:
            if key in self._data:
                # 更新已有记录
                existing = self._data[key]
                existing.value = value
                existing.version += 1
                existing.timestamp = time.time()
                if tags:
                    existing.tags = tags
                action = "更新"
            else:
                # 创建新记录
                self._data[key] = BlackboardEntry(
                    author=author,
                    key=key,
                    value=value,
                    tags=tags or []
                )
                action = "创建"
            
            # 记录操作历史
            self._history.append({
                "action": action,
                "author": author,
                "key": key,
                "timestamp": time.time()
            })
            
            print(f"  [黑板] {author} {action}了记录 '{key}'")
            
            # 通知订阅者
            self._notify(author, key, value)
    
    def read(self, key: str) -> Optional[BlackboardEntry]:
        """
        从黑板读取一条记录。
        
        Args:
            key: 记录的键名
        
        Returns:
            对应的记录，如果不存在则返回 None
        """
        with self._lock:
            return self._data.get(key)
    
    def read_all(self, author: str = None, tag: str = None) -> list[BlackboardEntry]:
        """
        读取黑板上的所有记录（可按作者或标签过滤）。
        
        Args:
            author: 可选的作者过滤
            tag: 可选的标签过滤
        
        Returns:
            符合条件的所有记录
        """
        with self._lock:
            entries = list(self._data.values())
            if author:
                entries = [e for e in entries if e.author == author]
            if tag:
                entries = [e for e in entries if tag in e.tags]
            return entries
    
    def search(self, keyword: str) -> list[BlackboardEntry]:
        """
        在黑板上搜索包含关键字的记录。
        
        搜索范围包括记录的键名和值（如果是字符串类型）。
        """
        with self._lock:
            results = []
            for entry in self._data.values():
                if keyword in entry.key:
                    results.append(entry)
                elif isinstance(entry.value, str) and keyword in entry.value:
                    results.append(entry)
            return results
    
    def list_keys(self) -> list[str]:
        """列出黑板上所有的键名"""
        with self._lock:
            return list(self._data.keys())
    
    def subscribe(self, callback: Callable):
        """
        订阅黑板的更新事件。
        
        当有新的记录写入时，所有订阅者的回调函数都会被调用。
        这实现了观察者模式，使得 Agent 可以被动地接收更新通知。
        """
        self._subscribers.append(callback)
    
    def _notify(self, author: str, key: str, value: Any):
        """通知所有订阅者"""
        for callback in self._subscribers:
            try:
                callback(author, key, value)
            except Exception as e:
                print(f"  [黑板] 通知订阅者失败: {e}")
    
    def get_history(self) -> list[dict]:
        """获取操作历史"""
        with self._lock:
            return self._history.copy()
    
    def get_snapshot(self) -> str:
        """获取黑板当前状态的文本快照（用于调试）"""
        with self._lock:
            lines = ["===== 黑板快照 ====="]
            for key, entry in self._data.items():
                value_preview = str(entry.value)[:100]
                if len(str(entry.value)) > 100:
                    value_preview += "..."
                lines.append(f"  [{entry.author}] {key} (v{entry.version}): {value_preview}")
            lines.append(f"===== 共 {len(self._data)} 条记录 =====")
            return "\n".join(lines)


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
# 基于黑板的 Agent
# ============================================================

class BlackboardAgent:
    """
    基于黑板的 Agent。
    
    这种 Agent 不通过消息传递来通信，而是通过读写黑板来协作。
    每个 Agent 关注黑板上的特定键名，当这些键名有更新时，
    Agent 就会被触发去执行相应的任务。
    """
    
    def __init__(
        self,
        name: str,
        system_prompt: str,
        blackboard: Blackboard,
        read_keys: list[str] = None,
        write_keys: list[str] = None,
        model: str = "gpt-4o"
    ):
        """
        Args:
            name: Agent 名称
            system_prompt: 系统提示词
            blackboard: 共享的黑板实例
            read_keys: 本 Agent 关注的黑板键名（当这些键有更新时会触发处理）
            write_keys: 本 Agent 可以写入的黑板键名
            model: 使用的 LLM 模型
        """
        self.name = name
        self.system_prompt = system_prompt
        self.blackboard = blackboard
        self.read_keys = read_keys or []
        self.write_keys = write_keys or []
        self.model = model
        
        # 订阅黑板更新
        self.blackboard.subscribe(self._on_blackboard_update)
        
        # 已处理的更新，避免重复处理
        self._processed_keys: set[str] = set()
    
    def _on_blackboard_update(self, author: str, key: str, value: Any):
        """
        黑板更新回调。当黑板上有新数据写入时被调用。
        
        只有当更新的键名在本 Agent 的 read_keys 列表中，
        且更新者不是本 Agent 自己时，才会触发处理。
        """
        if key in self.read_keys and author != self.name:
            # 检查是否需要处理（简单去重）
            version_key = f"{key}_{value}" if not isinstance(value, str) else key
            if version_key not in self._processed_keys:
                self._processed_keys.add(version_key)
                print(f"\n  [{self.name}] 检测到黑板更新: {key} (来自 {author})")
                self.process(key, value, author)
    
    def process(self, key: str, value: Any, author: str):
        """
        处理黑板上的更新。子类应该重写这个方法来实现具体逻辑。
        
        Args:
            key: 更新的键名
            value: 更新的值
            author: 更新的作者
        """
        # 默认实现：读取所有相关数据，生成新的输出
        context = self._gather_context()
        
        prompt = f"""你是「{self.name}」。你刚刚在黑板上检测到来自「{author}」的更新：
键名: {key}
内容: {value}

当前黑板上的所有信息：
{context}

请根据以上信息，完成你的任务。如果有需要补充或改进的地方，请写入黑板。"""
        
        messages = [
            {"role": "system", "content": self.system_prompt},
            {"role": "user", "content": prompt}
        ]
        
        result = call_llm(messages, model=self.model)
        
        # 将结果写入黑板
        if self.write_keys:
            output_key = self.write_keys[0]  # 使用第一个写入键
            self.blackboard.write(self.name, output_key, result)
    
    def _gather_context(self) -> str:
        """从黑板上收集上下文信息"""
        lines = []
        for entry in self.blackboard.read_all():
            value_preview = str(entry.value)[:500]
            lines.append(f"[{entry.author}] {entry.key}: {value_preview}")
        return "\n".join(lines) if lines else "(黑板为空)"


# ============================================================
# 演示：共享黑板协作
# ============================================================

def demo_blackboard():
    """演示共享黑板通信模式"""
    
    print("\n" + "="*60)
    print("演示：共享黑板通信模式")
    print("="*60)
    
    # 创建黑板
    blackboard = Blackboard()
    
    # 创建 Agent
    researcher = BlackboardAgent(
        name="研究员",
        system_prompt="你是一位技术研究员，擅长搜集和整理资料。请基于已有的资料进行补充和整理。",
        blackboard=blackboard,
        read_keys=["topic"],
        write_keys=["research_data"]
    )
    
    analyst = BlackboardAgent(
        name="分析师",
        system_prompt="你是一位技术分析师，擅长从资料中提取关键观点和洞察。",
        blackboard=blackboard,
        read_keys=["research_data"],
        write_keys=["analysis"]
    )
    
    writer = BlackboardAgent(
        name="作家",
        system_prompt="你是一位技术作家，擅长将分析结果转化为流畅的文章。",
        blackboard=blackboard,
        read_keys=["analysis"],
        write_keys=["article"]
    )
    
    # 开始协作流程
    print("\n--- 第一步：用户在黑板上发布研究主题 ---")
    blackboard.write("用户", "topic", "大语言模型中的 Agent 技术及其应用")
    
    # 展示黑板状态
    print(f"\n{blackboard.get_snapshot()}")
    
    # 注意：在实际运行中，由于 LLM 调用需要 API key，
    # 这里我们手动推进流程来演示黑板的工作原理
    
    print("\n--- 手动模拟后续步骤 ---")
    
    # 模拟研究员的输出
    research_data = """## Agent 技术概述
Agent 是一种能够自主感知环境、做出决策并执行行动的人工智能系统。在大语言模型时代，Agent 技术获得了全新的生命力。

### 核心技术
1. ReAct 框架：结合推理和行动
2. 工具使用：Agent 通过调用外部工具来扩展能力
3. 记忆系统：短期记忆和长期记忆的管理
4. 规划能力：将复杂任务分解为可执行的步骤

### 应用场景
- 自动化编程助手
- 智能客服系统
- 数据分析自动化
- 研究报告生成"""
    
    blackboard.write("研究员", "research_data", research_data)
    
    # 模拟分析师的输出
    analysis = """## 关键洞察
1. Agent 技术的核心在于"规划-执行-反思"的循环
2. 工具使用能力是 Agent 区别于普通 LLM 的关键特征
3. 记忆系统的质量直接影响 Agent 的长期表现

## 趋势判断
- Multi-Agent 协作将成为主流
- Agent 的自主性会进一步提升
- 安全性和可控性将受到更多关注"""
    
    blackboard.write("分析师", "analysis", analysis)
    
    # 展示最终黑板状态
    print(f"\n{blackboard.get_snapshot()}")
    
    # 展示操作历史
    print("\n--- 黑板操作历史 ---")
    for record in blackboard.get_history():
        print(f"  {record['action']}: {record['author']} -> {record['key']}")


if __name__ == "__main__":
    demo_blackboard()
```

### 24.2.3 事件驱动模式（Event-Driven）

在消息传递模式中，Agent 需要明确地知道消息发给谁。在黑板模式中，Agent 需要轮询黑板来检查更新。这两种方式在某些场景下都不够优雅。

事件驱动模式提供了一种更加松耦合的通信方式。在这种模式中，Agent 不直接互相通信，也不直接读取共享数据，而是通过发布和订阅事件来间接通信。

想象一个现代化的微服务架构：当用户下单成功时，订单服务不是直接通知库存服务、物流服务和通知服务，而是发布一个"订单创建成功"的事件。库存服务订阅了这个事件，自动扣减库存；物流服务订阅了这个事件，自动安排发货；通知服务订阅了这个事件，自动发送确认邮件。每个服务只关心自己感兴趣的事件，不需要知道是谁发布了这个事件，也不需要知道还有谁也在订阅这个事件。

```python
"""
第 24 章：事件驱动模式
通过发布-订阅机制实现松耦合的 Agent 通信
"""
import os
import json
import time
import uuid
from typing import Any, Callable, Optional
from dataclasses import dataclass, field
from openai import OpenAI


# ============================================================
# 事件系统
# ============================================================

@dataclass
class Event:
    """
    事件对象。
    
    事件是事件驱动系统中的基本通信单元。
    每个事件都有类型、发布者、数据和时间戳。
    
    事件类型决定了哪些订阅者会响应这个事件。
    订阅者通过事件类型来过滤自己感兴趣的事件。
    """
    event_type: str      # 事件类型，如 "task_completed"、"error_occurred"
    publisher: str       # 发布者标识
    data: dict           # 事件数据
    event_id: str = field(default_factory=lambda: str(uuid.uuid4())[:8])
    timestamp: float = field(default_factory=time.time)
    
    def __repr__(self):
        return f"Event({self.event_type}, by={self.publisher}, id={self.event_id})"


class EventBus:
    """
    事件总线：事件驱动系统的核心。
    
    事件总线负责：
    1. 接收事件发布
    2. 将事件分发给所有匹配的订阅者
    3. 维护事件历史，支持回放
    
    设计原则：
    - 发布者不需要知道订阅者的存在
    - 订阅者不需要知道发布者的存在
    - 同一个事件可以被多个订阅者处理
    - 事件的处理是异步的（在我们的简化实现中是同步的）
    """
    
    def __init__(self):
        # 订阅表：event_type -> list of callback functions
        self._subscribers: dict[str, list[Callable]] = {}
        # 通配符订阅：监听所有事件
        self._wildcard_subscribers: list[Callable] = []
        # 事件历史
        self._event_history: list[Event] = []
        # 统计信息
        self._stats: dict[str, int] = {}
    
    def subscribe(self, event_type: str, callback: Callable):
        """
        订阅特定类型的事件。
        
        Args:
            event_type: 要订阅的事件类型
            callback: 事件处理回调函数
        """
        if event_type not in self._subscribers:
            self._subscribers[event_type] = []
        self._subscribers[event_type].append(callback)
    
    def subscribe_all(self, callback: Callable):
        """
        订阅所有类型的事件（通配符订阅）。
        
        这个功能通常用于日志记录或监控系统。
        """
        self._wildcard_subscribers.append(callback)
    
    def publish(self, event: Event):
        """
        发布一个事件。
        
        事件总线会将事件分发给所有订阅了该事件类型的订阅者，
        以及所有通配符订阅者。
        
        Args:
            event: 要发布的事件
        """
        self._event_history.append(event)
        self._stats[event.event_type] = self._stats.get(event.event_type, 0) + 1
        
        print(f"  [事件总线] 发布事件: {event}")
        
        # 分发给特定类型的订阅者
        if event.event_type in self._subscribers:
            for callback in self._subscribers[event.event_type]:
                try:
                    callback(event)
                except Exception as e:
                    print(f"  [事件总线] 处理事件时出错: {e}")
        
        # 分发给通配符订阅者
        for callback in self._wildcard_subscribers:
            try:
                callback(event)
            except Exception as e:
                print(f"  [事件总线] 通配符处理事件时出错: {e}")
    
    def get_history(self, event_type: str = None) -> list[Event]:
        """获取事件历史（可按类型过滤）"""
        if event_type:
            return [e for e in self._event_history if e.event_type == event_type]
        return self._event_history.copy()
    
    def get_stats(self) -> dict:
        """获取事件统计信息"""
        return self._stats.copy()


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
# 事件驱动 Agent
# ============================================================

class EventDrivenAgent:
    """
    事件驱动的 Agent。
    
    这种 Agent 通过订阅事件来触发自己的行为，
    通过发布事件来通知其他 Agent 自己的工作成果。
    
    与消息传递模式的区别：
    - 消息传递：Agent A 直接发消息给 Agent B
    - 事件驱动：Agent A 发布事件，所有订阅该事件的 Agent 都会收到通知
    
    事件驱动模式的优势在于松耦合——Agent 之间不需要知道彼此的存在。
    """
    
    def __init__(
        self,
        name: str,
        system_prompt: str,
        event_bus: EventBus,
        subscribe_events: list[str] = None,
        produce_events: list[str] = None,
        model: str = "gpt-4o"
    ):
        self.name = name
        self.system_prompt = system_prompt
        self.event_bus = event_bus
        self.subscribe_events = subscribe_events or []
        self.produce_events = produce_events or []
        self.model = model
        self.state: dict[str, Any] = {}  # Agent 的内部状态
        
        # 订阅事件
        for event_type in self.subscribe_events:
            event_bus.subscribe(event_type, self._handle_event)
    
    def _handle_event(self, event: Event):
        """
        事件处理入口。
        
        当订阅的事件被触发时，这个方法会被调用。
        它会调用 Agent 的 LLM 能力来处理事件，并可能发布新的事件。
        """
        print(f"\n  [{self.name}] 收到事件: {event.event_type}")
        
        # 构建上下文
        context = self._build_context(event)
        
        prompt = f"""你是「{self.name}」，一个事件驱动的 AI Agent。

当前收到的事件:
- 类型: {event.event_type}
- 发布者: {event.publisher}
- 数据: {json.dumps(event.data, ensure_ascii=False, indent=2)}

上下文信息:
{context}

请根据以上信息完成你的任务。你的回复将作为产出数据发布到事件总线上。"""
        
        messages = [
            {"role": "system", "content": self.system_prompt},
            {"role": "user", "content": prompt}
        ]
        
        result = call_llm(messages, model=self.model)
        
        # 更新内部状态
        self.state[event.event_type] = result
        
        # 发布结果事件
        for event_type in self.produce_events:
            new_event = Event(
                event_type=event_type,
                publisher=self.name,
                data={
                    "input_event": event.event_type,
                    "result": result,
                    "source": event.publisher
                }
            )
            self.event_bus.publish(new_event)
    
    def _build_context(self, event: Event) -> str:
        """构建事件处理的上下文信息"""
        lines = []
        for key, value in self.state.items():
            preview = str(value)[:300]
            lines.append(f"[内部状态] {key}: {preview}")
        
        # 查看事件历史中的相关事件
        history = self.event_bus.get_history()
        for past_event in history[-5:]:  # 最近 5 个事件
            if past_event.event_type in self.subscribe_events:
                lines.append(f"[历史事件] {past_event.event_type} from {past_event.publisher}")
        
        return "\n".join(lines) if lines else "(无上下文)"


# ============================================================
# 演示：事件驱动的 Multi-Agent 系统
# ============================================================

def demo_event_driven():
    """演示事件驱动通信模式"""
    
    print("\n" + "="*60)
    print("演示：事件驱动通信模式")
    print("="*60)
    
    # 创建事件总线
    event_bus = EventBus()
    
    # 创建事件驱动的 Agent
    researcher = EventDrivenAgent(
        name="研究员",
        system_prompt="你是一位技术研究员。根据研究主题搜集相关资料。",
        event_bus=event_bus,
        subscribe_events=["new_topic"],
        produce_events=["research_completed"]
    )
    
    analyst = EventDrivenAgent(
        name="分析师",
        system_prompt="你是一位技术分析师。根据研究资料进行深入分析。",
        event_bus=event_bus,
        subscribe_events=["research_completed"],
        produce_events=["analysis_completed"]
    )
    
    writer = EventDrivenAgent(
        name="作家",
        system_prompt="你是一位技术作家。根据分析结果撰写文章。",
        event_bus=event_bus,
        subscribe_events=["analysis_completed"],
        produce_events=["article_completed"]
    )
    
    # 日志 Agent：监听所有事件
    logger_state = {"events": []}
    
    def log_event(event: Event):
        logger_state["events"].append({
            "type": event.event_type,
            "publisher": event.publisher,
            "time": event.timestamp
        })
    
    event_bus.subscribe_all(log_event)
    
    # 开始协作
    print("\n--- 发布初始事件 ---")
    initial_event = Event(
        event_type="new_topic",
        publisher="用户",
        data={"topic": "Multi-Agent 通信模式详解"}
    )
    event_bus.publish(initial_event)
    
    # 展示事件统计
    print(f"\n--- 事件统计 ---")
    stats = event_bus.get_stats()
    for event_type, count in stats.items():
        print(f"  {event_type}: {count} 次")
    
    # 展示日志 Agent 记录的事件
    print(f"\n--- 日志 Agent 记录的事件 ---")
    for record in logger_state["events"]:
        print(f"  {record['type']} by {record['publisher']}")


if __name__ == "__main__":
    demo_event_driven()
```

### 24.2.4 管道模式（Pipeline）

管道模式是消息传递的一种特殊形式，特别适合于数据处理流水线。在这种模式中，Agent 按照固定的顺序排列，数据从管道的一端流入，经过每个 Agent 的处理后，从另一端流出。

管道模式的特点是数据流向是单向的，每个 Agent 只需要关注自己的输入和输出，不需要了解整个管道的结构。就像工厂里的流水线——每个工位只需要完成自己的工序，不需要了解前面或后面的工位在做什么。

```python
"""
第 24 章：管道模式
数据在 Agent 流水线中单向流动
"""
import os
import json
import time
from typing import Any, Callable, Optional
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
# 管道系统
# ============================================================

@dataclass
class PipelineData:
    """
    在管道中流动的数据包。
    
    每个数据包包含原始数据、处理结果和元信息。
    当数据包经过每个 Agent 时，Agent 会将结果附加到 data 字段中。
    """
    original_input: str
    stages: list[dict] = field(default_factory=list)
    current_stage: int = 0
    metadata: dict = field(default_factory=dict)
    is_error: bool = False
    
    def add_stage(self, agent_name: str, result: str):
        """添加一个处理阶段的结果"""
        self.stages.append({
            "agent": agent_name,
            "result": result,
            "timestamp": time.time()
        })
        self.current_stage += 1
    
    def get_latest_result(self) -> str:
        """获取最新的处理结果"""
        if self.stages:
            return self.stages[-1]["result"]
        return self.original_input
    
    def get_summary(self) -> str:
        """获取处理过程的摘要"""
        lines = [f"原始输入: {self.original_input[:100]}..."]
        for i, stage in enumerate(self.stages):
            preview = stage["result"][:100]
            lines.append(f"  阶段 {i+1} ({stage['agent']}): {preview}...")
        return "\n".join(lines)


class PipelineStage:
    """
    管道中的一个处理阶段。
    
    每个阶段对应一个 Agent，负责对数据进行特定的处理。
    """
    
    def __init__(self, name: str, system_prompt: str, model: str = "gpt-4o"):
        self.name = name
        self.system_prompt = system_prompt
        self.model = model
    
    def process(self, data: PipelineData) -> PipelineData:
        """
        处理管道数据。
        
        读取前一个阶段的结果，进行处理，将结果附加到数据包中。
        """
        input_text = data.get_latest_result()
        
        messages = [
            {"role": "system", "content": self.system_prompt},
            {"role": "user", "content": f"请处理以下内容：\n\n{input_text}"}
        ]
        
        result = call_llm(messages, model=self.model)
        data.add_stage(self.name, result)
        
        return data


class Pipeline:
    """
    管道：将多个处理阶段串联起来。
    
    数据从第一个阶段流入，依次经过每个阶段的处理，
    最终从最后一个阶段流出。
    
    管道支持：
    - 链式处理：数据按顺序经过每个阶段
    - 中间检查：在每个阶段后检查数据质量
    - 错误处理：某个阶段失败时的降级策略
    """
    
    def __init__(self, name: str):
        self.name = name
        self.stages: list[PipelineStage] = []
    
    def add_stage(self, stage: PipelineStage) -> "Pipeline":
        """
        添加一个处理阶段。
        
        支持链式调用：pipeline.add_stage(s1).add_stage(s2)
        """
        self.stages.append(stage)
        return self
    
    def run(self, input_text: str, verbose: bool = True) -> PipelineData:
        """
        执行管道。
        
        Args:
            input_text: 输入文本
            verbose: 是否打印详细日志
        
        Returns:
            包含所有处理结果的数据包
        """
        if verbose:
            print(f"\n{'='*50}")
            print(f"管道 [{self.name}] 开始执行")
            print(f"输入: {input_text[:100]}...")
            print(f"{'='*50}")
        
        data = PipelineData(original_input=input_text)
        
        for i, stage in enumerate(self.stages):
            if verbose:
                print(f"\n--- 阶段 {i+1}/{len(self.stages)}: {stage.name} ---")
            
            try:
                data = stage.process(data)
                if verbose:
                    result_preview = data.get_latest_result()[:150]
                    print(f"  输出: {result_preview}...")
            except Exception as e:
                if verbose:
                    print(f"  阶段 {stage.name} 失败: {e}")
                data.is_error = True
                data.add_stage(stage.name, f"ERROR: {e}")
                break
        
        if verbose:
            print(f"\n{'='*50}")
            print(f"管道 [{self.name}] 执行完成")
            print(f"经过 {len(data.stages)} 个阶段")
            print(f"{'='*50}")
        
        return data


# ============================================================
# 演示：数据处理管道
# ============================================================

def demo_pipeline():
    """演示管道通信模式"""
    
    print("\n" + "="*60)
    print("演示：管道通信模式")
    print("="*60)
    
    # 创建处理管道
    pipeline = Pipeline("文本处理管道")
    
    # 添加处理阶段
    pipeline.add_stage(PipelineStage(
        name="语言检测与清洗",
        system_prompt="你是文本预处理专家。请检测输入文本的语言，并进行基本的清洗（去除多余空格、标准化标点等）。输出清洗后的文本和检测到的语言。"
    ))
    
    pipeline.add_stage(PipelineStage(
        name="关键信息提取",
        system_prompt="你是信息提取专家。请从文本中提取关键信息，包括：主题、关键词、重要观点。以结构化的方式输出。"
    ))
    
    pipeline.add_stage(PipelineStage(
        name="摘要生成",
        system_prompt="你是摘要生成专家。请根据提取的关键信息，生成一段简洁的摘要（100-200字）。"
    ))
    
    # 执行管道
    input_text = """大语言模型（LLM）是近年来人工智能领域最重要的突破之一。这些模型通过在海量文本数据上进行训练，学会了理解和生成自然语言。从 GPT 系列到 Claude 系列，大语言模型的能力不断提升，在问答、翻译、编程、创意写作等多个领域展现出了惊人的表现。然而，大语言模型也面临着幻觉、偏见、安全性等挑战，需要持续的研究和改进。"""
    
    result = pipeline.run(input_text)
    
    # 展示结果
    print(f"\n--- 处理结果摘要 ---")
    print(result.get_summary())


if __name__ == "__main__":
    demo_pipeline()
```

---

## 24.3 消息格式的设计艺术

### 24.3.1 为什么消息格式如此重要

在 Multi-Agent 系统中，消息格式就是 Agent 之间的"通用语言"。一个设计良好的消息格式能让 Agent 之间高效、准确地沟通；而一个设计糟糕的消息格式则会导致误解、错误和系统故障。

这就像人类社会中的合同和法律文书——格式越规范、越明确，出现歧义和纠纷的概率就越低。

### 24.3.2 设计结构化消息

```python
"""
第 24 章：结构化消息设计
定义 Agent 之间通信的消息协议
"""
import os
import json
import time
import uuid
from enum import Enum
from typing import Any, Optional
from dataclasses import dataclass, field, asdict


class MessageType(str, Enum):
    """消息类型枚举"""
    TASK = "task"              # 任务分配
    RESULT = "result"          # 任务结果
    FEEDBACK = "feedback"      # 反馈意见
    QUERY = "query"            # 信息查询
    STATUS = "status"          # 状态更新
    ERROR = "error"            # 错误通知
    HEARTBEAT = "heartbeat"    # 心跳（存活检测）
    BROADCAST = "broadcast"    # 广播消息


class Priority(str, Enum):
    """消息优先级"""
    LOW = "low"
    NORMAL = "normal"
    HIGH = "high"
    URGENT = "urgent"


@dataclass
class StructuredMessage:
    """
    结构化消息：Agent 间通信的标准格式。
    
    这个消息格式包含了通信所需的所有元信息：
    - 身份信息：谁发的、发给谁
    - 路由信息：消息类型、优先级
    - 内容信息：实际的数据和指令
    - 追踪信息：消息 ID、会话 ID、时间戳
    - 上下文信息：前序消息引用、额外元数据
    
    设计原则：
    1. 自描述性：收到消息的 Agent 能从消息本身理解其含义
    2. 可追溯性：每条消息都有唯一 ID，可以追踪其流转过程
    3. 可扩展性：通过 metadata 字段支持自定义扩展
    """
    # 身份信息
    sender: str
    receiver: str  # 可以是具体的 Agent 名，也可以是 "broadcast"
    
    # 路由信息
    message_type: MessageType
    priority: Priority = Priority.NORMAL
    
    # 内容信息
    subject: str = ""       # 消息主题/标题
    content: str = ""       # 消息正文
    payload: dict = field(default_factory=dict)  # 结构化数据载荷
    
    # 追踪信息
    message_id: str = field(default_factory=lambda: str(uuid.uuid4())[:12])
    conversation_id: str = ""
    reply_to: str = ""      # 引用的消息 ID（用于回复）
    timestamp: float = field(default_factory=time.time)
    
    # 上下文信息
    metadata: dict = field(default_factory=dict)
    ttl: int = 300          # 消息的生存时间（秒）
    
    def to_json(self) -> str:
        """序列化为 JSON"""
        data = asdict(self)
        data["message_type"] = self.message_type.value
        data["priority"] = self.priority.value
        return json.dumps(data, ensure_ascii=False, indent=2)
    
    @classmethod
    def from_json(cls, json_str: str) -> "StructuredMessage":
        """从 JSON 反序列化"""
        data = json.loads(json_str)
        data["message_type"] = MessageType(data["message_type"])
        data["priority"] = Priority(data["priority"])
        return cls(**data)
    
    def is_expired(self) -> bool:
        """检查消息是否已过期"""
        return time.time() - self.timestamp > self.ttl
    
    def create_reply(
        self,
        content: str,
        payload: dict = None,
        message_type: MessageType = MessageType.RESULT
    ) -> "StructuredMessage":
        """创建一条回复消息"""
        return StructuredMessage(
            sender=self.receiver,
            receiver=self.sender,
            message_type=message_type,
            subject=f"Re: {self.subject}",
            content=content,
            payload=payload or {},
            conversation_id=self.conversation_id,
            reply_to=self.message_id,
            priority=self.priority
        )


# ============================================================
# 消息验证器
# ============================================================

class MessageValidator:
    """
    消息验证器：确保消息的格式和内容符合规范。
    
    在 Multi-Agent 系统中，消息验证是保证系统可靠性的重要环节。
    不合法的消息可能会导致 Agent 的行为异常，甚至导致系统崩溃。
    """
    
    # 规则定义：每种消息类型必须包含的字段
    REQUIRED_FIELDS = {
        MessageType.TASK: ["subject", "content"],
        MessageType.RESULT: ["content"],
        MessageType.FEEDBACK: ["content"],
        MessageType.QUERY: ["subject"],
        MessageType.STATUS: ["content"],
        MessageType.ERROR: ["content"],
    }
    
    # 内容长度限制
    MAX_CONTENT_LENGTH = 50000
    MAX_SUBJECT_LENGTH = 200
    
    @classmethod
    def validate(cls, message: StructuredMessage) -> tuple[bool, str]:
        """
        验证消息是否合法。
        
        Returns:
            (是否合法, 错误描述)
        """
        # 检查过期
        if message.is_expired():
            return False, "消息已过期"
        
        # 检查发送者和接收者
        if not message.sender or not message.receiver:
            return False, "发送者或接收者为空"
        
        # 检查必填字段
        required = cls.REQUIRED_FIELDS.get(message.message_type, [])
        for field_name in required:
            value = getattr(message, field_name, None)
            if not value:
                return False, f"缺少必填字段: {field_name}"
        
        # 检查内容长度
        if len(message.content) > cls.MAX_CONTENT_LENGTH:
            return False, f"内容超过最大长度限制 ({cls.MAX_CONTENT_LENGTH})"
        
        if len(message.subject) > cls.MAX_SUBJECT_LENGTH:
            return False, f"主题超过最大长度限制 ({cls.MAX_SUBJECT_LENGTH})"
        
        return True, "验证通过"


# ============================================================
# 演示
# ============================================================

def demo_structured_messages():
    """演示结构化消息的使用"""
    
    print("\n" + "="*60)
    print("演示：结构化消息设计")
    print("="*60)
    
    # 创建一条任务消息
    task_msg = StructuredMessage(
        sender="协调者",
        receiver="研究员",
        message_type=MessageType.TASK,
        priority=Priority.HIGH,
        subject="研究 Multi-Agent 通信协议",
        content="请研究和整理 Multi-Agent 系统中常用的通信协议和消息格式。",
        conversation_id="session_001",
        metadata={"deadline": "2024-12-31", "max_tokens": 3000}
    )
    
    print(f"\n--- 任务消息 ---")
    print(task_msg.to_json())
    
    # 验证消息
    is_valid, error = MessageValidator.validate(task_msg)
    print(f"\n验证结果: {'通过' if is_valid else error}")
    
    # 创建回复消息
    reply_msg = task_msg.create_reply(
        content="研究完成。主要的通信协议包括...（省略详细内容）",
        payload={
            "sources_count": 5,
            "key_findings": ["JSON-RPC", "REST API", "消息队列"]
        }
    )
    
    print(f"\n--- 回复消息 ---")
    print(reply_msg.to_json())
    
    # 验证回复消息
    is_valid, error = MessageValidator.validate(reply_msg)
    print(f"\n验证结果: {'通过' if is_valid else error}")
    
    # 创建一条非法消息（缺少必填字段）
    bad_msg = StructuredMessage(
        sender="测试",
        receiver="目标",
        message_type=MessageType.TASK,
        subject=""  # 空主题，应该不合法
    )
    
    is_valid, error = MessageValidator.validate(bad_msg)
    print(f"\n--- 非法消息测试 ---")
    print(f"验证结果: {'通过' if is_valid else error}")


if __name__ == "__main__":
    demo_structured_messages()
```

---

## 24.4 同步通信与异步通信

### 24.4.1 同步通信：简单直接但可能阻塞

同步通信就像打电话——你打给对方，必须等对方接电话、说完话、挂了电话，你才能继续做其他事情。如果对方很忙或者电话占线，你就只能等着。

在 Multi-Agent 系统中，同步通信意味着 Agent A 发送消息给 Agent B 后，必须等待 Agent B 的回复才能继续执行。这种模式的优点是简单直观，缺点是如果 Agent B 处理时间很长，Agent A 就会被阻塞。

### 24.4.2 异步通信：灵活高效但更复杂

异步通信就像发短信——你发了消息给对方，不需要等对方回复就可以继续做自己的事情。对方会在方便的时候回复你。

在 Multi-Agent 系统中，异步通信意味着 Agent A 发送消息给 Agent B 后，可以继续执行其他任务，不需要等待 Agent B 的回复。Agent B 处理完消息后，会通过回调、事件等方式将结果返回给 Agent A。

异步通信更复杂，但在很多场景下更实用。特别是在 Agent 处理时间不确定、或者多个 Agent 需要并行工作的情况下。

```python
"""
第 24 章：同步 vs 异步通信
对比两种通信模式的实现和适用场景
"""
import os
import json
import time
import asyncio
import threading
from queue import Queue
from typing import Any, Optional, Callable
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
# 同步消息传递
# ============================================================

class SyncMessageSystem:
    """
    同步消息传递系统。
    
    特点：
    - 发送消息后，发送者会阻塞等待回复
    - 通信是请求-响应模式
    - 实现简单，但效率较低
    """
    
    def __init__(self):
        self.agents: dict[str, Callable] = {}
    
    def register(self, name: str, handler: Callable):
        """注册 Agent"""
        self.agents[name] = handler
    
    def send_and_wait(
        self,
        sender: str,
        receiver: str,
        content: str,
        timeout: float = 30.0
    ) -> Optional[str]:
        """
        发送消息并等待回复（同步阻塞）。
        
        Args:
            sender: 发送者
            receiver: 接收者
            content: 消息内容
            timeout: 超时时间（秒）
        
        Returns:
            接收者的回复，超时返回 None
        """
        if receiver not in self.agents:
            print(f"  [同步] Agent '{receiver}' 不存在")
            return None
        
        print(f"  [同步] {sender} → {receiver}: 发送消息...")
        
        start_time = time.time()
        
        try:
            handler = self.agents[receiver]
            result = handler(content, sender)
            
            elapsed = time.time() - start_time
            print(f"  [同步] {sender} ← {receiver}: 收到回复 ({elapsed:.1f}s)")
            return result
            
        except Exception as e:
            print(f"  [同步] 通信失败: {e}")
            return None


# ============================================================
# 异步消息传递
# ============================================================

class AsyncMessageSystem:
    """
    异步消息传递系统。
    
    特点：
    - 发送消息后，发送者可以继续做其他事情
    - 回复通过回调函数异步返回
    - 支持并行处理多个消息
    - 实现更复杂，但效率更高
    """
    
    def __init__(self):
        self.agents: dict[str, Callable] = {}
        self.message_queue: Queue = Queue()
        self.response_callbacks: dict[str, Callable] = {}
        self._running = False
        self._worker_thread: Optional[threading.Thread] = None
    
    def register(self, name: str, handler: Callable):
        """注册 Agent"""
        self.agents[name] = handler
    
    def start(self):
        """启动异步消息处理"""
        self._running = True
        self._worker_thread = threading.Thread(target=self._process_messages, daemon=True)
        self._worker_thread.start()
        print("  [异步] 消息处理系统已启动")
    
    def stop(self):
        """停止异步消息处理"""
        self._running = False
        if self._worker_thread:
            self._worker_thread.join(timeout=5)
        print("  [异步] 消息处理系统已停止")
    
    def send_async(
        self,
        sender: str,
        receiver: str,
        content: str,
        callback: Callable = None
    ):
        """
        异步发送消息（非阻塞）。
        
        Args:
            sender: 发送者
            receiver: 接收者
            content: 消息内容
            callback: 收到回复后的回调函数
        """
        msg_id = f"{sender}_{receiver}_{time.time()}"
        
        if callback:
            self.response_callbacks[msg_id] = callback
        
        self.message_queue.put({
            "id": msg_id,
            "sender": sender,
            "receiver": receiver,
            "content": content,
            "timestamp": time.time()
        })
        
        print(f"  [异步] {sender} → {receiver}: 消息已入队 (ID: {msg_id[:20]})")
    
    def _process_messages(self):
        """后台消息处理循环"""
        while self._running:
            try:
                msg = self.message_queue.get(timeout=1)
            except:
                continue
            
            receiver = msg["receiver"]
            if receiver not in self.agents:
                print(f"  [异步] Agent '{receiver}' 不存在，消息丢弃")
                continue
            
            try:
                print(f"  [异步] 处理消息: {msg['sender']} → {receiver}")
                handler = self.agents[receiver]
                result = handler(msg["content"], msg["sender"])
                
                # 触发回调
                msg_id = msg["id"]
                if msg_id in self.response_callbacks:
                    callback = self.response_callbacks[msg_id]
                    callback(msg["sender"], result)
                    del self.response_callbacks[msg_id]
                    
            except Exception as e:
                print(f"  [异步] 处理消息失败: {e}")


# ============================================================
# 演示：同步 vs 异步
# ============================================================

def demo_sync_vs_async():
    """对比同步和异步通信模式"""
    
    print("\n" + "="*60)
    print("演示：同步 vs 异步通信")
    print("="*60)
    
    def simple_handler(content: str, sender: str) -> str:
        """简单的消息处理函数"""
        time.sleep(0.5)  # 模拟处理时间
        return f"已处理来自 {sender} 的消息: {content[:50]}..."
    
    # --- 同步演示 ---
    print("\n--- 同步通信演示 ---")
    sync_system = SyncMessageSystem()
    sync_system.register("agent_a", simple_handler)
    sync_system.register("agent_b", simple_handler)
    
    start = time.time()
    for i in range(3):
        result = sync_system.send_and_wait("协调者", "agent_a", f"任务 {i+1}")
    elapsed_sync = time.time() - start
    print(f"  同步通信总耗时: {elapsed_sync:.1f}s")
    
    # --- 异步演示 ---
    print("\n--- 异步通信演示 ---")
    async_system = AsyncMessageSystem()
    async_system.register("agent_a", simple_handler)
    async_system.register("agent_b", simple_handler)
    
    results = []
    start = time.time()
    for i in range(3):
        def callback(sender, result, idx=i):
            results.append((idx, result))
        async_system.send_async("协调者", "agent_a", f"任务 {i+1}", callback=callback)
    elapsed_async = time.time() - start
    print(f"  异步消息发送耗时: {elapsed_async:.1f}s")
    print(f"  (注: 异步模式下消息在后台处理，发送不阻塞)")


if __name__ == "__main__":
    demo_sync_vs_async()
```

---

## 24.5 案例分析：通信模式的选择

### 24.5.1 案例一：代码审查系统的通信设计

**场景**：构建一个 Multi-Agent 代码审查系统，多个 Agent 从不同角度审查代码。

**通信方案**：
1. **代码提交** → 使用事件驱动模式发布"代码提交"事件
2. **审查分配** → 路由 Agent 通过消息传递将代码片段分配给各审查 Agent
3. **结果汇总** → 审查结果写入共享黑板
4. **最终报告** → 报告 Agent 从黑板读取所有审查结果，生成最终报告

**为什么混合使用**：不同阶段的通信需求不同。代码提交是不确定时间的，适合事件驱动；审查分配需要明确指定目标，适合消息传递；结果汇总需要多个来源的数据，适合黑板模式。

### 24.5.2 案例二：实时对话系统的通信设计

**场景**：构建一个多人对话 AI 系统，多个 Agent 同时与用户交互。

**通信方案**：使用异步消息传递。每个 Agent 运行在独立的线程中，通过消息队列接收用户输入和向用户输出。异步模式确保一个 Agent 的慢速响应不会阻塞其他 Agent。

---

## 24.6 常见坑与解决方案

### 坑 1：消息丢失

**问题描述**：在异步通信中，如果消息队列满了或者消费者处理失败，消息可能会丢失。

**解决方案**：实现消息确认机制（ACK）。接收方处理完消息后发送确认，发送方在收到确认前保持消息副本。使用持久化队列防止系统崩溃导致的消息丢失。

### 坑 2：消息顺序混乱

**问题描述**：异步通信中，消息的发送顺序和接收顺序可能不一致。

**解决方案**：在消息中添加序列号，接收方根据序列号重新排序。或者在需要严格顺序的场景下使用同步通信。

### 坑 3：通信死锁

**问题描述**：Agent A 等待 Agent B 的回复，同时 Agent B 也在等待 Agent A 的回复，双方永远等不到对方的回复。

**解决方案**：设置超时机制，当等待时间超过阈值时自动放弃并报错。避免循环依赖的通信模式。

### 坑 4：消息格式不一致

**问题描述**：不同 Agent 对消息格式的理解不一致，导致解析错误。

**解决方案**：使用统一的消息格式定义（如本章的 StructuredMessage），并添加消息验证器在接收端检查消息格式。

### 坑 5：通信开销过大

**问题描述**：Agent 之间传递的数据量过大，导致通信成本高、延迟大。

**解决方案**：对传输的数据进行压缩和摘要。只传递必要的信息，避免冗余。使用缓存减少重复传输。

---

## 24.7 练习题

### 练习 1：实现带确认的消息传递

**难度：★☆☆☆☆**

在消息传递模式的基础上，添加消息确认机制。每条消息都有一个唯一的 ID，接收方处理完消息后返回一个确认消息。发送方如果没有在超时时间内收到确认，则重发消息。

**要求**：实现至少 3 次重试逻辑。

### 练习 2：实现订阅过滤的黑板系统

**难度：★★☆☆☆**

扩展黑板系统，支持更精细的订阅过滤。Agent 不仅可以按键名订阅，还可以按标签、作者、值的内容进行过滤。

**示例**：Agent A 只关心来自"研究员"且带有"重要"标签的记录。

### 练习 3：实现支持优先级的消息队列

**难度：★★☆☆☆**

实现一个支持优先级的消息队列。高优先级的消息优先被处理。当队列满时，低优先级的消息会被丢弃。

**要求**：实现 4 个优先级：LOW、NORMAL、HIGH、URGENT。

### 练习 4：实现双向管道

**难度：★★★☆☆**

设计一个支持双向数据流的管道。数据不仅可以从头流向尾，还可以从尾流向头。比如，最后一个 Agent 的审查意见可以反馈给前面的 Agent 进行修改。

### 练习 5：实现通信性能监控

**难度：★★★☆☆**

为 Multi-Agent 通信系统添加性能监控功能。记录每条消息的发送时间、接收时间、处理时间，计算平均延迟、吞吐量等指标。支持生成性能报告。

### 练习 6：设计跨语言通信协议

**难度：★★★★☆**

设计一个能够支持不同编程语言实现的 Agent 之间通信的协议。使用 JSON-RPC 或类似的标准化协议，确保不同语言实现的 Agent 可以互相通信。

---

## 24.8 实战任务

### 任务：构建 Multi-Agent 协作文档编辑系统

**目标**：构建一个支持多个 Agent 协作编辑文档的系统，综合运用本章学到的各种通信模式。

**系统设计**：

1. **事件总线**：处理文档更新通知、用户编辑事件等
2. **共享黑板**：存储文档的当前版本、编辑历史、评论等
3. **消息传递**：Agent 之间的点对点协调（如请求协助、反馈意见）
4. **管道**：文档处理流水线（语法检查 → 格式化 → 最终输出）

**Agent 设计**：

1. **作者 Agent**：负责撰写文档内容
2. **编辑 Agent**：负责改善文档的表达和结构
3. **校对 Agent**：负责检查语法和拼写错误
4. **格式化 Agent**：负责将文档转换为目标格式（Markdown、HTML 等）

**要求**：
1. 综合使用至少 3 种通信模式
2. 代码完整可运行
3. 支持通过环境变量配置 API key
4. 包含详细的通信日志

---

## 24.9 本章小结

本章深入探讨了 Multi-Agent 系统中的通信机制，这是理解 Multi-Agent 协作的关键基础。

我们学习了四种核心通信模式。消息传递是最直观的方式，Agent 直接互相发送消息；共享黑板通过共享数据空间实现协作，适合松耦合的场景；事件驱动通过发布-订阅机制实现最松散的耦合；管道模式适合线性的数据处理流水线。

我们还学习了结构化消息的设计方法——好的消息格式是可靠通信的基础。以及同步通信和异步通信的区别：同步简单但可能阻塞，异步灵活但更复杂。

在实际应用中，通常不会只使用单一的通信模式。一个设计良好的 Multi-Agent 系统往往会根据不同的场景和需求，混合使用多种通信模式，取长补短。

通信是 Multi-Agent 系统的基石。有了可靠的通信机制，我们才能在接下来的章节中探讨更高级的协作策略和模式。

> "通信的质量决定了协作的上限。再聪明的 Agent，如果不能有效地与其他 Agent 沟通，也无法发挥出应有的能力。"

---

**下一章预告**：在第 25 章中，我们将探讨 Multi-Agent 协调的策略与模式——多个 Agent 如何高效地分工协作？谁来决定谁做什么？当 Agent 之间出现冲突时如何解决？敬请期待。
