# 第 20 章：Agent 的上下文管理 —— 窗口限制的工程对策

---

## 学习目标

完成本章学习后，你将能够：

1. 深入理解 LLM 上下文窗口限制的本质——它不仅是"能装多少字"的问题，更影响 Agent 的设计和用户体验
2. 掌握上下文压缩和摘要技术——如何在有限的空间里保留最重要的信息
3. 了解上下文管理的工程实践——不同场景下的最佳策略
4. 能够实现高效的上下文管理策略——滑动窗口、摘要压缩、重要性排序
5. 理解工具结果对上下文空间的冲击——为什么工具返回结果是上下文管理的最大挑战
6. 掌握统一上下文管理器的设计——将多种策略组合成一个完整的解决方案

## 核心问题

1. 上下文窗口限制如何影响 Agent 的设计？
2. 如何在有限的上下文中容纳更多信息？
3. 如何平衡信息完整性和处理效率？

---

## 20.1 上下文窗口的本质

### 20.1.1 什么是上下文窗口

上下文窗口（Context Window）是 LLM 一次能处理的最大 Token 数量。你可以把它想象成 LLM 的"工作台"——它一次只能在工作台上摊开这么多资料。超出的部分，它就"看不见"了。

这个工作台的空间是有限的，而且不是全部都能用来放对话历史。它需要分成几个区域：

```
┌─────────────────────────────────────────────┐
│              上下文窗口（例如 128K Token）      │
│                                              │
│  ┌──────────────┬──────────────┬──────────┐ │
│  │ System Prompt │  对话历史     │  输出空间  │ │
│  │   (~500)      │  (~10000)    │ (~4096)  │ │
│  └──────────────┴──────────────┴──────────┘ │
│                                              │
│  System Prompt: Agent 的身份、能力、规则       │
│  对话历史: 用户和 Agent 的对话记录              │
│  输出空间: Agent 回复需要预留的空间             │
│                                              │
│  ↑ 总空间有限，需要精心管理                    │
└─────────────────────────────────────────────┘
```

### 20.1.2 不同模型的上下文窗口

| 模型 | 上下文窗口 | 输出限制 | 特点 |
|------|-----------|---------|------|
| GPT-4o | 128K | 16K | 通用场景，平衡性能和成本 |
| Claude Sonnet 4.6 | 200K | 8K | 超长上下文，适合长文档 |
| GPT-4o-mini | 128K | 16K | 成本敏感场景 |
| DeepSeek V3 | 128K | 8K | 开源模型 |

### 20.1.3 上下文空间的"挤占"问题

在实际的 Agent 应用中，上下文空间的消耗速度远比你想象的快：

```python
def estimate_context_usage():
    """估算典型 Agent 对话的上下文消耗"""
    
    system_prompt = 500      # System Prompt
    user_message = 200       # 用户消息
    agent_response = 500     # Agent 回复
    tool_result = 3000       # 工具返回结果（往往很长！）
    tool_call_overhead = 200 # 工具调用的格式开销
    
    single_turn = user_message + agent_response + tool_result + tool_call_overhead
    print(f"单轮对话消耗（含工具调用）: ~{single_turn} tokens")
    
    max_turns_no_tools = (128000 - system_prompt) // (user_message + agent_response)
    max_turns_with_tools = (128000 - system_prompt) // single_turn
    
    print(f"不含工具调用: 最多 ~{max_turns_no_tools} 轮")
    print(f"含工具调用: 最多 ~{max_turns_with_tools} 轮")
    print(f"\n结论: 工具调用会大幅减少可用的对话轮数!")

estimate_context_usage()
```

你会发现：如果不调用工具，128K 的上下文窗口可以支撑几百轮对话。但一旦开始调用工具（特别是搜索、代码执行等返回大量内容的工具），可用轮数会急剧下降到几十轮甚至更少。

### 20.1.4 为什么上下文管理很重要

上下文管理的核心挑战是：**你需要在有限的空间里放进去最有用的信息。**

放少了，Agent 可能忘记之前对话的关键信息，或者没有足够的上下文来做出好的决策。放多了，要么超出窗口限制导致信息丢失，要么挤占了输出空间导致回复不完整。

好的上下文管理策略需要在"信息完整性"和"空间效率"之间找到平衡。

---

## 20.2 上下文压缩技术

### 20.2.1 滑动窗口

滑动窗口是最简单也最常用的策略：只保留最近 N 条消息，丢弃更早的消息。

```python
import time


class SlidingWindowManager:
    """
    滑动窗口上下文管理器
    
    策略：当消息总 Token 数超过限制时，移除最早的消息。
    
    优点：简单、高效、可预测
    缺点：可能丢失早期的重要信息
    """
    
    def __init__(self, max_tokens: int = 8000, reserve_output: int = 4096):
        """
        参数:
            max_tokens: 上下文窗口的总 Token 数
            reserve_output: 为输出预留的 Token 数
        """
        self.max_tokens = max_tokens
        self.reserve_output = reserve_output
        self.messages: list[dict] = []
    
    def add_message(self, role: str, content: str):
        """添加消息并自动管理窗口"""
        tokens = self._estimate_tokens(content)
        
        self.messages.append({
            "role": role,
            "content": content,
            "tokens": tokens,
            "timestamp": time.time(),
        })
        
        # 检查并裁剪
        self._trim()
    
    def get_messages(self) -> list[dict]:
        """获取当前窗口内的消息"""
        return [
            {"role": msg["role"], "content": msg["content"]}
            for msg in self.messages
        ]
    
    def _trim(self):
        """裁剪消息直到总 Token 数在限制内"""
        available = self.max_tokens - self.reserve_output
        current_tokens = sum(msg["tokens"] for msg in self.messages)
        
        while current_tokens > available and len(self.messages) > 1:
            removed = self.messages.pop(0)
            current_tokens -= removed["tokens"]
    
    def get_stats(self) -> dict:
        """获取统计信息"""
        total_tokens = sum(msg["tokens"] for msg in self.messages)
        available = self.max_tokens - self.reserve_output
        return {
            "message_count": len(self.messages),
            "total_tokens": total_tokens,
            "available_tokens": available,
            "usage_percent": round(total_tokens / available * 100, 1) if available > 0 else 0,
        }
    
    def _estimate_tokens(self, text: str) -> int:
        """粗略估算 Token 数"""
        chinese_chars = sum(1 for c in text if '一' <= c <= '鿿')
        other_chars = len(text) - chinese_chars
        return chinese_chars // 2 + other_chars // 4 + 1
```

### 20.2.2 摘要压缩

滑动窗口的缺点是"直接丢弃"——被移除的消息就永远丢失了。摘要压缩更聪明：在移除旧消息之前，先让 LLM 为这些消息生成一个摘要，这样关键信息就被"压缩"保留了下来。

```python
class SummarizingManager:
    """
    摘要压缩上下文管理器
    
    当消息积累到一定数量时，将旧消息压缩为摘要，
    同时保留最近几条消息的完整内容。
    
    效果：空间节省 50-80%，同时保留关键信息
    """
    
    def __init__(self, llm_client, max_tokens: int = 8000,
                 compress_threshold: float = 0.75,
                 keep_recent: int = 5):
        """
        参数:
            llm_client: LLM 客户端（用于生成摘要）
            max_tokens: 最大 Token 数
            compress_threshold: 触发压缩的阈值（占 max_tokens 的比例）
            keep_recent: 压缩时保留最近几条消息
        """
        self.llm_client = llm_client
        self.max_tokens = max_tokens
        self.compress_threshold = compress_threshold
        self.keep_recent = keep_recent
        self.messages: list[dict] = []
        self.summary: str = ""
        self.summary_tokens: int = 0
    
    def add_message(self, role: str, content: str):
        """添加消息"""
        self.messages.append({
            "role": role,
            "content": content,
            "tokens": self._estimate_tokens(content),
        })
        
        # 检查是否需要压缩
        total = self.summary_tokens + sum(m["tokens"] for m in self.messages)
        if total > self.max_tokens * self.compress_threshold:
            self._compress()
    
    def get_messages(self) -> list[dict]:
        """获取消息（包含摘要）"""
        result = []
        
        # 摘要作为 System 消息
        if self.summary:
            result.append({
                "role": "system",
                "content": f"之前的对话摘要：\n{self.summary}",
            })
        
        # 最近的消息
        result.extend([
            {"role": m["role"], "content": m["content"]}
            for m in self.messages
        ])
        
        return result
    
    def _compress(self):
        """压缩旧消息为摘要"""
        if len(self.messages) <= self.keep_recent:
            return
        
        # 分割：要压缩的 vs 要保留的
        to_compress = self.messages[:-self.keep_recent]
        self.messages = self.messages[-self.keep_recent:]
        
        # 生成摘要
        conversation_text = "\n".join(
            f"{m['role']}: {m['content']}"
            for m in to_compress
        )
        
        prompt = f"""请将以下对话内容总结为简洁的摘要，保留关键信息（用户需求、重要决策、关键结论）。

已有的历史摘要：
{self.summary if self.summary else "无"}

需要总结的新对话：
{conversation_text}

请输出更新后的完整摘要（保留历史摘要中的重要内容）："""
        
        self.summary = self.llm_client.generate(prompt)
        self.summary_tokens = self._estimate_tokens(self.summary)
        
        print(f"[压缩] 旧消息 {len(to_compress)} 条已压缩为摘要 "
              f"({self.summary_tokens} tokens)")
    
    def _estimate_tokens(self, text: str) -> int:
        """粗略估算 Token 数"""
        chinese_chars = sum(1 for c in text if '一' <= c <= '鿿')
        other_chars = len(text) - chinese_chars
        return chinese_chars // 2 + other_chars // 4 + 1
```

---

## 20.3 智能上下文管理

### 20.3.1 基于重要性的管理

不是所有消息都一样重要。用户的需求描述、关键决策通常比"好的"、"收到"这样的确认消息重要得多。基于重要性的管理策略会为每条消息打一个重要性分数，压缩时优先保留高重要性的消息。

```python
class ImportanceBasedManager:
    """
    基于重要性的上下文管理器
    
    为每条消息评估重要性，压缩时优先保留高重要性的消息。
    """
    
    def __init__(self, llm_client, max_tokens: int = 8000):
        self.llm_client = llm_client
        self.max_tokens = max_tokens
        self.messages: list[dict] = []
    
    def add_message(self, role: str, content: str,
                    importance: float = 0.5):
        """
        添加消息
        
        importance: 0-1 之间的重要性分数
        - 0.9-1.0: 关键信息（用户需求、最终决策）
        - 0.5-0.8: 一般对话
        - 0.1-0.4: 确认、问候等
        """
        self.messages.append({
            "role": role,
            "content": content,
            "tokens": self._estimate_tokens(content),
            "importance": importance,
            "timestamp": time.time(),
        })
        
        self._manage()
    
    def _manage(self):
        """管理上下文空间"""
        total_tokens = sum(m["tokens"] for m in self.messages)
        
        if total_tokens <= self.max_tokens * 0.8:
            return  # 空间充足，不需要压缩
        
        # 按重要性排序，保留最重要的消息
        target_tokens = int(self.max_tokens * 0.6)
        sorted_msgs = sorted(
            self.messages,
            key=lambda m: (m["importance"], m["timestamp"]),
            reverse=True,
        )
        
        kept = []
        kept_tokens = 0
        for msg in sorted_msgs:
            if kept_tokens + msg["tokens"] <= target_tokens:
                kept.append(msg)
                kept_tokens += msg["tokens"]
        
        # 按时间顺序重新排列
        self.messages = sorted(kept, key=lambda m: m["timestamp"])
    
    def get_messages(self) -> list[dict]:
        """获取消息"""
        return [
            {"role": m["role"], "content": m["content"]}
            for m in self.messages
        ]
    
    def _estimate_tokens(self, text: str) -> int:
        chinese_chars = sum(1 for c in text if '一' <= c <= '鿿')
        other_chars = len(text) - chinese_chars
        return chinese_chars // 2 + other_chars // 4 + 1
```

### 20.3.2 智能重要性评估

手动为每条消息打重要性分数不现实，我们可以用 LLM 来自动评估：

```python
class AutoImportanceScorer:
    """自动评估消息重要性"""
    
    def __init__(self, llm_client):
        self.llm_client = llm_client
    
    def score(self, message: str, conversation_context: str = "") -> float:
        """
        评估消息的重要性
        
        返回 0-1 的分数
        """
        prompt = f"""评估以下消息在对话中的重要性。

对话上下文：
{conversation_context[:500] if conversation_context else "无"}

消息：{message}

重要性评分标准：
- 0.9-1.0: 包含关键需求、决策、或核心信息
- 0.5-0.8: 一般性的讨论或信息交换
- 0.1-0.4: 确认、问候、无实质内容的回复

只输出 0-1 之间的数字："""
        
        response = self.llm_client.generate(prompt)
        
        try:
            import re
            numbers = re.findall(r'[\d.]+', response.strip())
            if numbers:
                score = float(numbers[0])
                return max(0.0, min(1.0, score))
        except (ValueError, IndexError):
            pass
        
        return 0.5  # 默认中等重要性
```

---

## 20.4 工具结果的上下文管理

### 20.4.1 为什么工具结果是最大的挑战

在所有消耗上下文空间的因素中，工具返回结果是最大的"罪魁祸首"。一次搜索可能返回几千字的结果，一次代码执行可能输出几百行日志。如果不加控制地把这些结果全部放入上下文，空间很快就会被耗尽。

### 20.4.2 工具结果压缩器

```python
class ToolResultCompressor:
    """
    工具结果压缩器
    
    根据工具类型选择不同的压缩策略：
    - 搜索结果：保留最相关的几条，其余摘要
    - 代码输出：保留头尾，省略中间
    - 通用文本：使用 LLM 生成摘要
    """
    
    def __init__(self, llm_client, max_result_tokens: int = 2000):
        self.llm_client = llm_client
        self.max_result_tokens = max_result_tokens
    
    def compress(self, tool_name: str, result: str,
                 context: str = "") -> str:
        """压缩工具结果"""
        result_tokens = self._estimate_tokens(result)
        
        if result_tokens <= self.max_result_tokens:
            return result  # 不需要压缩
        
        # 根据工具类型选择压缩策略
        if tool_name in ("search", "web_search", "google"):
            return self._compress_search_results(result, context)
        elif tool_name in ("run_code", "execute", "code_executor"):
            return self._compress_code_output(result)
        elif tool_name in ("read_file", "file_read"):
            return self._compress_file_content(result)
        else:
            return self._compress_generic(result)
    
    def _compress_search_results(self, results: str,
                                  query: str = "") -> str:
        """压缩搜索结果：只保留最相关的"""
        prompt = f"""请压缩以下搜索结果，只保留与查询最相关的信息。

查询：{query or "未知"}
原始搜索结果：
{results[:3000]}

要求：
1. 只保留前 3 个最相关的结果
2. 每个结果保留标题和关键摘要
3. 总长度控制在 {self.max_result_tokens} tokens 以内

压缩后的结果："""
        
        return self.llm_client.generate(prompt)
    
    def _compress_code_output(self, output: str) -> str:
        """压缩代码输出：保留头尾，省略中间"""
        lines = output.split("\n")
        
        if len(lines) <= 30:
            return output
        
        head_lines = lines[:15]
        tail_lines = lines[-10:]
        omitted = len(lines) - 25
        
        compressed = "\n".join(head_lines)
        compressed += f"\n\n... 省略 {omitted} 行 ...\n\n"
        compressed += "\n".join(tail_lines)
        
        return compressed
    
    def _compress_file_content(self, content: str) -> str:
        """压缩文件内容"""
        lines = content.split("\n")
        
        if len(lines) <= 50:
            return content
        
        # 保留前 30 行和后 20 行
        compressed = "\n".join(lines[:30])
        compressed += f"\n\n... 省略 {len(lines) - 50} 行 ...\n\n"
        compressed += "\n".join(lines[-20:])
        
        return compressed
    
    def _compress_generic(self, text: str) -> str:
        """通用压缩"""
        prompt = f"""请将以下文本压缩到约 {self.max_result_tokens} tokens，保留核心信息。

原始文本：
{text[:3000]}

压缩后的文本："""
        
        return self.llm_client.generate(prompt)
    
    def _estimate_tokens(self, text: str) -> int:
        chinese_chars = sum(1 for c in text if '一' <= c <= '鿿')
        other_chars = len(text) - chinese_chars
        return chinese_chars // 2 + other_chars // 4 + 1
```

---

## 20.5 统一上下文管理器

### 20.5.1 将所有策略组合在一起

实际应用中，我们通常需要将多种管理策略组合使用。统一上下文管理器提供了一个完整的解决方案。

```python
class UnifiedContextManager:
    """
    统一上下文管理器
    
    组合了多种上下文管理策略：
    1. 滑动窗口：控制消息数量
    2. 摘要压缩：保留关键信息
    3. 工具结果压缩：控制工具输出大小
    4. 重要性排序：优先保留重要消息
    """
    
    def __init__(self, llm_client, config: dict = None):
        config = config or {}
        
        self.llm_client = llm_client
        self.max_tokens = config.get("max_tokens", 128000)
        self.system_prompt_tokens = config.get("system_prompt_tokens", 1000)
        self.reserve_output = config.get("reserve_output", 4096)
        self.compression_threshold = config.get("compression_threshold", 0.8)
        self.keep_recent = config.get("keep_recent", 10)
        
        # 组件
        self.messages: list[dict] = []
        self.summary: str = ""
        self.summary_tokens: int = 0
        self.compressor = ToolResultCompressor(llm_client)
        
        # 统计
        self.stats = {
            "total_messages": 0,
            "compressions": 0,
            "tokens_saved": 0,
        }
    
    def add_message(self, role: str, content: str,
                    is_tool_result: bool = False,
                    tool_name: str = ""):
        """
        添加消息
        
        如果是工具结果，会先进行压缩
        """
        # 工具结果压缩
        if is_tool_result and tool_name:
            original_tokens = self._estimate_tokens(content)
            content = self.compressor.compress(tool_name, content)
            compressed_tokens = self._estimate_tokens(content)
            self.stats["tokens_saved"] += original_tokens - compressed_tokens
        
        tokens = self._estimate_tokens(content)
        
        self.messages.append({
            "role": role,
            "content": content,
            "tokens": tokens,
            "timestamp": time.time(),
        })
        self.stats["total_messages"] += 1
        
        # 检查是否需要压缩
        self._manage()
    
    def get_messages(self, system_prompt: str = "") -> list[dict]:
        """获取完整的上下文消息列表"""
        result = []
        
        # System Prompt
        if system_prompt:
            result.append({
                "role": "system",
                "content": system_prompt,
            })
        
        # 对话摘要
        if self.summary:
            result.append({
                "role": "system",
                "content": f"对话历史摘要：\n{self.summary}",
            })
        
        # 对话消息
        result.extend([
            {"role": m["role"], "content": m["content"]}
            for m in self.messages
        ])
        
        return result
    
    def _manage(self):
        """管理上下文空间"""
        available = self.max_tokens - self.system_prompt_tokens - self.reserve_output
        current = self.summary_tokens + sum(m["tokens"] for m in self.messages)
        
        if current <= available * self.compression_threshold:
            return
        
        # 执行压缩
        self._compress()
    
    def _compress(self):
        """压缩旧消息为摘要"""
        if len(self.messages) <= self.keep_recent:
            return
        
        to_compress = self.messages[:-self.keep_recent]
        self.messages = self.messages[-self.keep_recent:]
        
        conversation = "\n".join(
            f"{m['role']}: {m['content'][:200]}"
            for m in to_compress
        )
        
        prompt = f"""请总结以下对话的核心要点，保留关键信息。

已有摘要：{self.summary if self.summary else "无"}

对话内容：
{conversation}

请输出更新后的摘要："""
        
        self.summary = self.llm_client.generate(prompt)
        self.summary_tokens = self._estimate_tokens(self.summary)
        self.stats["compressions"] += 1
    
    def _estimate_tokens(self, text: str) -> int:
        chinese_chars = sum(1 for c in text if '一' <= c <= '鿿')
        other_chars = len(text) - chinese_chars
        return chinese_chars // 2 + other_chars // 4 + 1
    
    def get_usage_report(self) -> dict:
        """获取上下文使用报告"""
        available = self.max_tokens - self.system_prompt_tokens - self.reserve_output
        current = self.summary_tokens + sum(m["tokens"] for m in self.messages)
        
        return {
            "max_tokens": self.max_tokens,
            "available_tokens": available,
            "current_tokens": current,
            "usage_percent": round(current / available * 100, 1) if available > 0 else 0,
            "message_count": len(self.messages),
            "summary_tokens": self.summary_tokens,
            "compressions": self.stats["compressions"],
            "tokens_saved_by_compression": self.stats["tokens_saved"],
        }
```

---

## 20.6 常见坑

### 20.6.1 过度压缩

**问题描述：** 压缩过于激进，丢失了重要信息。比如把 10 条消息压缩成一句话，结果关键的用户需求和决策细节都没了。

**解决方案：** 设置合理的压缩阈值（比如 80% 时才开始压缩），保留最近几条消息的完整内容，压缩时明确要求"保留关键信息"。

### 20.6.2 压缩延迟

**问题描述：** 每次压缩都需要调用 LLM 生成摘要，这会增加几百毫秒到几秒的延迟，影响用户体验。

**解决方案：** 使用异步压缩——在后台线程中执行压缩，不阻塞主流程。或者使用本地的小模型（如 T5）来生成摘要，速度更快。

### 20.6.3 工具结果溢出

**问题描述：** 工具返回的结果过长，一次就消耗了大量上下文空间，导致没有空间放其他消息。

**解决方案：** 在工具结果返回时就进行压缩，而不是等到放入上下文时再压缩。设置工具结果的最大 Token 数限制。

### 20.6.4 System Prompt 过长

**问题描述：** System Prompt 占用了太多上下文空间（比如放了大量的工具描述、角色设定、规则说明），留给对话历史的空间不够。

**解决方案：** 精简 System Prompt，把不常用的信息（如工具的详细文档）移到外部存储，只在需要时按需加载。

---

## 20.7 不同场景的上下文管理策略

### 20.7.1 客服场景

客服 Agent 需要记住用户的问题和之前的对话，但客服对话通常有明确的生命周期（一次服务会话）。推荐策略：

- 使用滑动窗口 + 摘要压缩
- 保留最近 10-20 条消息的完整内容
- 旧消息压缩为摘要（保留用户的核心诉求和已提供的解决方案）

```python
# 客服场景的推荐配置
customer_service_config = {
    "max_tokens": 8000,
    "reserve_output": 2000,
    "compression_threshold": 0.7,
    "keep_recent": 10,
    "system_prompt_tokens": 800,
}
```

### 20.7.2 长期助手场景

个人助手需要跨对话记住用户信息，上下文管理需要结合长期记忆：

- 使用摘要压缩保留对话核心内容
- 关键信息提取到长期记忆（向量数据库）
- 每次新对话开始时，从长期记忆中检索相关上下文注入

```python
# 长期助手的上下文管理策略
class LongTermAssistantContext:
    def __init__(self, llm_client, vector_memory):
        self.context_manager = SummarizingManager(llm_client, max_tokens=10000)
        self.long_term_memory = vector_memory
    
    def get_context(self, current_message: str) -> list[dict]:
        # 1. 从长期记忆中检索相关记忆
        relevant_memories = self.long_term_memory.search(current_message, top_k=3)
        
        # 2. 构建上下文
        context = []
        
        # 注入相关记忆
        if relevant_memories:
            memory_text = "\n".join(m["content"] for m in relevant_memories)
            context.append({
                "role": "system",
                "content": f"关于这个用户的记忆：\n{memory_text}",
            })
        
        # 添加对话历史
        context.extend(self.context_manager.get_messages())
        
        return context
```

### 20.7.3 代码助手场景

代码助手的特点是工具调用频繁（搜索文档、执行代码、读写文件），工具结果消耗大量上下文。推荐策略：

- 对工具结果进行严格压缩
- 代码输出只保留头尾
- 搜索结果只保留最相关的条目
- 使用较高的压缩阈值（0.6）

### 20.7.4 策略选择指南

| 场景 | 推荐策略 | 最大 Token | 保留最近 | 压缩阈值 |
|------|----------|-----------|---------|---------|
| 客服对话 | 滑动窗口 + 摘要 | 8K | 10 条 | 0.7 |
| 长期助手 | 摘要 + 向量记忆 | 10K | 5 条 | 0.75 |
| 代码助手 | 工具结果压缩 | 16K | 15 条 | 0.6 |
| 文档问答 | 滑动窗口 | 32K | 20 条 | 0.8 |

---

## 20.8 练习题

### 练习 1：实现滑动窗口管理器

实现一个 SlidingWindowManager，要求：
- 支持 Token 数量估算
- 自动裁剪超出限制的消息
- 保持消息的时间顺序
- 提供使用统计（当前 Token 数、可用空间、使用率）

### 练习 2：实现摘要压缩管理器

实现一个 SummarizingManager，要求：
- 在超出阈值时自动压缩
- 保留最近 N 条消息
- 维护对话摘要
- 摘要包含之前所有压缩的内容（增量摘要）

### 练习 3：实现工具结果压缩

实现一个 ToolResultCompressor，要求：
- 识别不同类型的工具结果（搜索、代码输出、文件内容）
- 根据类型选择压缩策略
- 保持结果的关键信息
- 统计压缩前后的 Token 数变化

### 练习 4：实现基于重要性的管理

实现一个 ImportanceBasedManager，要求：
- 为每条消息评估重要性
- 压缩时优先保留高重要性的消息
- 支持手动和自动（LLM 评估）两种重要性评估方式

### 练习 5：实现统一上下文管理器

实现一个 UnifiedContextManager，组合以上所有策略，要求：
- 自动选择合适的压缩策略
- 提供完整的使用报告
- 支持配置化（最大 Token 数、压缩阈值等）
- 提供调试信息（每条消息的 Token 数、压缩历史等）

---

## 20.8 实战任务

### 任务：构建高效上下文管理系统

**目标：** 构建一个高效的上下文管理系统，确保 Agent 在有限的上下文窗口中始终保持最佳的对话质量。

**要求：**

1. 支持多种压缩策略（滑动窗口、摘要压缩、重要性排序）
2. 实现智能的工具结果压缩
3. 提供上下文使用统计和监控
4. 支持配置化管理
5. 提供可视化的使用报告

---

## 20.9 本章小结

- **上下文窗口限制是 LLM 的核心约束**，直接影响 Agent 的设计。每个 Token 都很珍贵，需要精心管理。上下文空间需要在 System Prompt、对话历史和输出空间之间分配。

- **滑动窗口**是最简单的上下文管理策略：只保留最近 N 条消息。它简单、高效、可预测，但缺点是直接丢弃旧消息，可能丢失重要信息。

- **摘要压缩**在滑动窗口的基础上更进一步：在移除旧消息之前，先让 LLM 生成摘要保留关键信息。这样既节省了空间，又保留了重要的上下文。摘要压缩的代价是需要额外调用 LLM，增加延迟和成本。

- **基于重要性的管理**通过为消息打重要性分数，优先保留高重要性的消息。这比简单的滑动窗口更智能，但需要一个可靠的重要性评估机制。

- **工具结果是上下文管理的最大挑战**。搜索结果、代码输出、文件内容等工具返回值往往很长，一次调用就可能消耗几千甚至上万 Token。必须在工具结果返回时就进行压缩。

- **统一上下文管理器**将多种策略组合在一起，提供完整的上下文管理方案。好的统一管理器应该支持配置化、提供使用统计、有合理的默认值。

- **上下文管理的核心原则**：保留最重要的信息，丢弃最不重要的信息，在信息完整性和空间效率之间找到最佳平衡点。

