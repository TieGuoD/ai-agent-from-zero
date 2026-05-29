# 第 1 章：LLM 的本质 —— 从语言模型到智能体的第一步

> **本章定位：** 全书的起点。不理解 LLM 的本质，后续所有关于 Agent 的讨论都是空中楼阁。本章将从零开始，带你理解大语言模型到底是什么、它能做什么、它不能做什么，以及这些特性如何深刻地影响了 Agent 的设计。

---

## 学习目标

完成本章学习后，你将能够：

1. **解释 LLM 的工作原理** —— 能用自己的话描述从输入文本到输出文本的完整流程，包括 Tokenization、Embedding、Transformer 处理、文本生成四个阶段
2. **理解上下文窗口的限制** —— 能量化分析上下文窗口对 Agent 设计的影响，知道为什么上下文管理是 Agent 工程的核心挑战
3. **识别和理解幻觉问题** —— 能区分幻觉和错误，理解幻觉的成因，知道如何在 Agent 中缓解幻觉
4. **掌握模型参数的工程含义** —— 能根据场景选择合适的 Temperature、Top-p、Max Tokens 设置
5. **区分主流 LLM 模型** —— 能根据任务需求选择合适的模型（GPT-4o、Claude 4.x、开源模型）
6. **搭建完整的开发环境** —— 能配置 Python 环境、安装依赖、管理 API Key、成功调用 LLM API

## 核心问题

在开始学习之前，请先思考这三个问题。学完本章后，你应该能清晰地回答它们：

1. **LLM 到底是什么？** 它为什么能"理解"人类语言？这种"理解"和人类的理解有什么区别？
2. **上下文窗口为什么是 Agent 设计的核心约束？** 它如何限制了 Agent 的能力？工程师如何在有限的窗口内最大化 Agent 的表现？
3. **幻觉问题为什么对 Agent 特别危险？** 它和普通的"错误"有什么区别？为什么 Agent 场景下的幻觉后果更严重？

---

## 1.1 什么是大语言模型

### 1.1.1 一句话定义

大语言模型（Large Language Model, LLM）是一个基于 Transformer 架构、在海量文本数据上训练的神经网络模型，它的核心能力是：**给定一段文本的前文，预测下一个 Token 出现的概率分布。**

这个定义看似简单，但蕴含了深刻的意义。让我们逐层拆解。

### 1.1.2 从统计语言模型到神经语言模型

语言模型的历史可以追溯到信息论之父香农（Claude Shannon）。1948 年，香农提出了用概率来量化语言的思想：

```
一个句子的概率 = P(w_1) × P(w_2|w_1) × P(w_3|w_1,w_2) × ... × P(w_n|w_1,...,w_{n-1})
```

这意味着，如果我们知道每个词在给定前文条件下出现的概率，我们就能计算任意句子的概率。

**早期方法：N-gram 模型**

N-gram 模型假设一个词的概率只取决于它前面的 N-1 个词：

```
P("散步" | "今天天气", "很适合") ≈ P("散步" | "适合")  # Bigram (N=2)
```

这种方法简单有效，但有两个致命缺陷：
- **稀疏性：** 很多词组合在训练数据中从未出现过，概率为 0
- **长距离依赖：** N 很大时，需要的训练数据指数增长

**神经语言模型的突破**

2013 年，Word2Vec 证明了可以用神经网络将词映射到连续的向量空间中，语义相近的词在向量空间中距离也相近。这为后续的突破奠定了基础。

### 1.1.3 Transformer：LLM 的核心架构

2017 年，Google 发表了划时代论文《Attention Is All You Need》，提出了 Transformer 架构。这是现代 LLM 的基石。

**Transformer 的核心思想：自注意力机制（Self-Attention）**

传统的序列模型（如 RNN、LSTM）按顺序处理文本，无法并行化。Transformer 通过自注意力机制，让每个词都能直接"关注"序列中的所有其他词。

```
Attention(Q, K, V) = softmax(QK^T / √d_k) × V

其中：
Q (Query): 查询向量，表示"我想找什么"
K (Key): 键向量，表示"我是什么"
V (Value): 值向量，表示"我能提供什么信息"
d_k: 向量维度，用于缩放
```

**直觉理解：**

想象你在读一篇关于"苹果"的文章。当你看到"苹果发布了新 iPhone"时，你的大脑会自动将"苹果"理解为"Apple 公司"而不是"水果"。自注意力机制做的事情类似——它让模型在处理每个词时，能够参考整个上下文来理解其含义。

**Transformer 的优势：**

1. **并行化：** 所有位置可以同时计算，训练速度快
2. **长距离依赖：** 任意两个位置之间的关系可以直接建模
3. **可扩展：** 通过增加层数和参数量，模型能力持续提升

### 1.1.4 LLM 的训练过程

LLM 的训练通常分为两个阶段：

**阶段 1：预训练（Pre-training）**

在海量文本数据（通常是互联网文本）上，训练模型预测下一个 Token。

```
训练目标：给定 "今天天气"，预测下一个 Token 是 "很好" 的概率
训练数据：数万亿 Token 的文本
训练成本：数百万美元的计算资源
```

这个阶段让模型学会了：
- 语法和语义知识
- 世界知识（虽然有截止日期）
- 推理模式（虽然不是显式训练的）

**阶段 2：对齐（Alignment）**

通过人类反馈强化学习（RLHF）或类似技术，让模型的行为符合人类期望。

```
预训练模型：能生成流畅的文本，但可能生成有害内容
对齐后模型：生成有用、无害、诚实的回复
```

### 1.1.5 涌现能力：量变到质变

当模型参数量超过一定阈值（通常在 100 亿以上），会突然展现出训练目标中没有明确要求的能力。这些能力被称为"涌现能力"（Emergent Abilities）。

**主要涌现能力：**

| 能力 | 描述 | 为什么重要 |
|------|------|-----------|
| 少样本学习 | 给几个示例就能完成新任务 | Agent 不需要为每个任务微调模型 |
| 链式推理 | 能展示推理过程 | Agent 的 ReAct 模式依赖此能力 |
| 代码生成 | 能理解和生成程序代码 | 代码 Agent 的基础 |
| 指令遵循 | 能理解并执行复杂指令 | Agent 能接收复杂的任务描述 |
| 多语言能力 | 能理解和生成多种语言 | 多语言 Agent 的基础 |

**涌现能力对 Agent 的意义：**

涌现能力意味着 LLM 不仅仅是一个"预测下一个词"的模型，它具备了理解、推理、规划的能力。这正是 Agent 能够工作的基础——如果 LLM 只能做简单的文本预测，它就无法理解用户的意图、选择合适的工具、制定执行计划。

---

## 1.2 上下文窗口：Agent 设计的核心约束

### 1.2.1 什么是上下文窗口

上下文窗口（Context Window）是 LLM 一次能处理的最大 Token 数量。它就像 LLM 的"工作记忆"——模型只能"看到"上下文窗口内的内容，窗口外的信息对模型来说不存在。

```
┌─────────────────────────────────────────────────────────┐
│                    上下文窗口（例如 128K Token）            │
│                                                          │
│  ┌──────────────┬───────────────┬──────────────────────┐ │
│  │ System Prompt │  对话历史      │  输出（生成空间）      │ │
│  │   (~500)      │  (~50,000)    │  (剩余空间)           │ │
│  └──────────────┴───────────────┴──────────────────────┘ │
│                                                          │
│  ↑ 这些内容共享同一个有限的空间                             │
│  ↑ 每个组件都在"争夺"这个空间                              │
└─────────────────────────────────────────────────────────┘
```

### 1.2.2 Token 到底是什么

Token 是 LLM 处理文本的最小单位。一个 Token 可能是一个字、一个词、一个子词，甚至一个标点符号。

```python
import tiktoken

# 使用 GPT-4o 的分词器
enc = tiktoken.encoding_for_model("gpt-4o")

# 英文分词示例
tokens = enc.encode("Hello, how are you?")
print(f"原文: 'Hello, how are you?'")
print(f"Token IDs: {tokens}")  # [9906, 11, 2643, 527, 499, 30]
print(f"Token 数量: {len(tokens)}")  # 6

# 解码回文本
for token_id in tokens:
    print(f"  {token_id} -> '{enc.decode([token_id])}'")

# 中文分词示例
tokens_zh = enc.encode("大语言模型是人工智能的重要突破")
print(f"\n原文: '大语言模型是人工智能的重要突破'")
print(f"Token 数量: {len(tokens_zh)}")  # 通常是中文字符数的 1-2 倍
```

**关键洞察：中英文 Token 消耗差异**

一个中文字符通常需要 1-2 个 Token，而一个英文单词通常只需要 1 个 Token。这意味着：

| 内容 | 大约 Token 数 | 成本影响 |
|------|-------------|---------|
| 1000 个中文字 | ~1500 Token | 高 |
| 1000 个英文单词 | ~1300 Token | 中 |
| 一段 500 字的中文摘要 | ~750 Token | 中 |

对于 Agent 来说，这意味着：
1. 中文 Agent 的 API 成本可能比英文 Agent 高 20-50%
2. 上下文窗口的"有效容量"在中文场景下更小
3. 设计 Agent 时需要更精细地管理上下文空间

### 1.2.3 不同模型的上下文窗口

| 模型 | 上下文窗口 | 输出限制 | 输入价格（每百万Token） | 输出价格（每百万Token） |
|------|-----------|---------|----------------------|----------------------|
| GPT-4o | 128K | 16K | $2.50 | $10.00 |
| GPT-4o-mini | 128K | 16K | $0.15 | $0.60 |
| Claude Sonnet 4.6 | 200K | 8K | $3.00 | $15.00 |
| Claude Haiku 4.5 | 200K | 8K | $0.80 | $4.00 |

> **注意：** 以上价格仅供参考，请查阅各提供商的最新官方定价。价格可能随时间变化。

**上下文窗口对 Agent 设计的量化影响**

让我们做一个具体的计算。假设我们使用 GPT-4o（128K 窗口）构建一个 Agent：

```
System Prompt（角色定义、工具描述、行为规范）：~2000 Token
对话历史（20 轮对话，每轮 ~300 Token）：~6000 Token
工具调用结果（5 个工具，每个 ~200 Token）：~1000 Token
用户当前输入：~200 Token
─────────────────────────────────────────
已使用：~9,400 Token
可用输出空间：128,000 - 9,400 = ~118,600 Token
```

看起来空间很充裕？但别忘了：

1. **长对话场景：** 如果用户和 Agent 进行 100 轮对话，对话历史可能达到 30,000+ Token
2. **RAG 场景：** 如果 Agent 需要处理长文档，文档内容可能占用 50,000+ Token
3. **Multi-Agent 场景：** 多个 Agent 的消息传递会占用大量空间

**设计启示：** 上下文窗口管理是 Agent 工程的核心挑战之一。优秀的 Agent 必须精心管理上下文空间，不能浪费在无关信息上。

### 1.2.4 上下文窗口的管理策略

策略 1：**滑动窗口** —— 只保留最近的 N 条消息

```python
class SlidingWindowMemory:
    def __init__(self, max_messages: int = 20):
        self.max_messages = max_messages
        self.messages = []

    def add(self, role: str, content: str):
        self.messages.append({"role": role, "content": content})
        if len(self.messages) > self.max_messages:
            self.messages = self.messages[-self.max_messages:]
```

策略 2：**摘要压缩** —— 将旧消息压缩为摘要

```python
class SummarizingMemory:
    def __init__(self, llm_fn, max_messages: int = 20):
        self.llm_fn = llm_fn
        self.max_messages = max_messages
        self.messages = []
        self.summary = ""

    def add(self, role: str, content: str):
        self.messages.append({"role": role, "content": content})
        if len(self.messages) > self.max_messages:
            self._compress()

    def _compress(self):
        old_messages = self.messages[:10]
        self.summary = self.llm_fn(
            f"请用 100 字以内摘要以下对话内容：\n{old_messages}"
        )
        self.messages = self.messages[10:]
```

策略 3：**选择性保留** —— 根据重要性保留消息

```python
class SelectiveMemory:
    def __init__(self, max_tokens: int = 50000):
        self.max_tokens = max_tokens
        self.messages = []

    def add(self, role: str, content: str, importance: float = 0.5):
        self.messages.append({
            "role": role,
            "content": content,
            "importance": importance,
        })

    def get_optimized(self) -> list[dict]:
        """返回优化后的消息列表"""
        # 按重要性排序，保留最重要的消息
        sorted_msgs = sorted(
            self.messages,
            key=lambda x: x["importance"],
            reverse=True,
        )
        # 保留到 token 限制内
        result = []
        total_tokens = 0
        for msg in sorted_msgs:
            tokens = len(msg["content"]) // 2  # 粗略估计
            if total_tokens + tokens <= self.max_tokens:
                result.append(msg)
                total_tokens += tokens
        return result
```

---

## 1.3 幻觉：Agent 可靠性的最大威胁

### 1.3.1 什么是幻觉

幻觉（Hallucination）是指 LLM 生成看似合理、流畅，但实际上错误或虚构的内容。这些内容在语法上是正确的，在语义上是"说得通的"，但在事实上是错误的。

**幻觉 vs 错误**

| 特征 | 幻觉 | 错误 |
|------|------|------|
| 表面现象 | 看起来很合理 | 可能看起来不合理 |
| 发现难度 | 难以发现 | 容易发现 |
| 原因 | 模型"编造"了不存在的信息 | 模型推理或计算出错 |
| 示例 | "Python 3.15 引入了 XXX 特性"（如果 3.15 不存在） | "123 × 456 = 55088"（计算错误） |

**为什么 LLM 会产生幻觉**

LLM 的本质是"概率预测下一个 Token"。当模型不确定下一个词应该是什么时，它会根据统计模式"猜测"一个看起来合理的词。这个过程没有"事实核查"机制——模型不知道自己说的是对还是错。

```
用户："Python 3.15 的新特性是什么？"

模型内部的推理（简化）：
- "Python 3.x" 的新特性通常包括：类型注解改进、性能优化、语法糖...
- "3.15" 应该是一个较新的版本...
- 我来"编造"一些看起来合理的特性...

输出："Python 3.15 引入了 XXX 语法、YYY 性能优化、ZZZ 新模块..."

问题：Python 3.15 可能还不存在，这些特性是模型编造的
```

### 1.3.2 幻觉的成因

1. **训练数据偏差：** 模型学到的模式不完全准确。如果训练数据中某些错误信息出现频率高，模型可能会"学到"这些错误。

2. **概率采样的随机性：** 生成时 Temperature > 0 会引入随机性，可能选择概率较低但不正确的 Token。

3. **知识截止：** 模型的知识有截止日期（例如训练数据截止到 2024 年 4 月）。对于截止日期之后的信息，模型可能会"编造"。

4. **过度泛化：** 模型将已知模式错误地应用到新场景。例如，模型知道"Python 3.12 有新特性"，就可能推断"Python 3.15 也有新特性"，即使 3.15 还不存在。

5. **缺乏事实核查机制：** LLM 没有内置的"事实数据库"来验证生成的内容。

### 1.3.3 幻觉对 Agent 的影响

这是理解 Agent 设计的关键：**Agent 有执行能力，幻觉的影响比普通聊天机器人严重得多。**

```
普通聊天机器人的幻觉：
  用户看到错误信息 → 可以质疑、可以搜索验证
  后果：用户被误导

Agent 的幻觉：
  Agent 基于错误信息执行操作 → 操作已经执行
  后果：不可逆的错误操作
```

**具体场景分析：**

| 场景 | 聊天机器人幻觉后果 | Agent 幻觉后果 |
|------|-------------------|---------------|
| 天气查询 | 用户看到错误温度 | Agent 可能基于错误温度做决策（如取消户外活动） |
| 代码生成 | 用户看到有 bug 的代码 | Agent 可能执行有 bug 的代码，导致系统故障 |
| 邮件处理 | 用户看到错误摘要 | Agent 可能发送错误的邮件回复 |
| 数据分析 | 用户看到错误的分析结果 | Agent 可能基于错误分析做出错误的商业决策 |

### 1.3.4 缓解幻觉的策略

**策略 1：RAG（检索增强生成）**

在生成回答前，先从外部知识库检索相关信息，用检索结果增强 LLM 的输入。

```
用户问题 → 检索知识库 → 获取相关文档 → 将文档注入 Prompt → LLM 生成回答

优势：回答基于真实数据，减少幻觉
局限：检索质量影响回答质量
```

**策略 2：Tool Calling（工具调用）**

让 LLM 调用工具获取真实数据，而不是自己"猜测"。

```
用户："Python 最新版本是什么？"

无 Tool Calling：
  LLM 可能输出 "Python 3.13"（可能是幻觉）

有 Tool Calling：
  LLM 决定调用 search 工具
  工具返回 "Python 3.13.1（2024年10月发布）"
  LLM 基于真实数据回答
```

**策略 3：人类确认（Human-in-the-Loop）**

关键操作前请求人类确认。

```
Agent："我将发送以下邮件给客户..."
用户确认后，Agent 才执行发送操作
```

**策略 4：输出验证**

对 LLM 输出进行格式和逻辑检查。

```python
def validate_output(output: str, expected_format: str) -> bool:
    """验证 LLM 输出"""
    if expected_format == "json":
        try:
            json.loads(output)
            return True
        except json.JSONDecodeError:
            return False
    return True
```

**策略 5：多模型投票**

多个模型生成结果，取共识。

```python
def multi_model_vote(question: str, models: list) -> str:
    """多模型投票"""
    answers = []
    for model in models:
        response = model.generate(question)
        answers.append(response)
    # 选择出现频率最高的答案
    from collections import Counter
    return Counter(answers).most_common(1)[0][0]
```

---

## 1.4 模型参数：工程师的调优旋钮

### 1.4.1 Temperature：控制随机性

Temperature 控制 LLM 生成时的随机性。值越低，生成越确定性；值越高，生成越随机。

**数学原理：**

```
softmax(z_i / T)

T = 1: 保持原始概率分布
T < 1: 分布变得更"尖锐"（更确定性）
T > 1: 分布变得更"平坦"（更随机）
T → 0: 几乎总是选择概率最高的 Token
T → ∞: 均匀分布，完全随机
```

**直觉理解：**

想象你在选择餐厅：
- Temperature = 0：你总是去评分最高的那家
- Temperature = 0.5：你倾向于去评分高的，但偶尔会尝试新餐厅
- Temperature = 1.0：你随机选择，但高评分的更有可能被选中
- Temperature = 1.5：你完全随机选择，所有餐厅机会均等

**Agent 中的 Temperature 设置建议：**

| 场景 | Temperature | 原因 |
|------|------------|------|
| 工具调用决策 | 0 | 需要确定性，不能"随机"选择工具 |
| 事实性回答 | 0-0.3 | 需要准确性 |
| 代码生成 | 0-0.2 | 代码需要精确 |
| 文本摘要 | 0.3-0.5 | 需要准确性但允许一定灵活性 |
| 创意写作 | 0.7-1.0 | 需要多样性和创造性 |
| 头脑风暴 | 0.8-1.2 | 需要大量创意 |

### 1.4.2 Top-p：核采样

Top-p（Nucleus Sampling）控制采样的范围。只从累积概率达到 p 的 Token 中采样。

```python
# Top-p = 0.1：只从概率最高的 Token 中采样（约前 10%）
# Top-p = 0.5：从累积概率达到 50% 的 Token 中采样
# Top-p = 0.9：从累积概率达到 90% 的 Token 中采样
# Top-p = 1.0：不截断，从所有 Token 中采样
```

**Temperature vs Top-p 的区别：**

- Temperature：调整整个概率分布的"尖锐程度"（全局调整）
- Top-p：截断概率分布的"尾部"（局部截断）

**最佳实践：** 通常只调整其中一个，不要同时调整。同时调整可能导致不可预测的行为。

### 1.4.3 Max Tokens：输出长度限制

Max Tokens 限制输出的最大 Token 数。这是一个硬限制，输出不会超过这个值。

```python
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "写一篇长文"}],
    max_tokens=1000,  # 最多输出 1000 个 Token
)
```

**注意事项：**
- Max Tokens 不影响输入，只影响输出
- 如果输出达到 Max Tokens 限制，输出会被截断
- 合理设置 Max Tokens 可以控制成本和延迟

### 1.4.4 其他重要参数

| 参数 | 描述 | 建议值 |
|------|------|--------|
| frequency_penalty | 降低重复 Token 的概率 | 0-0.5 |
| presence_penalty | 鼓励谈论新话题 | 0-0.5 |
| stop | 停止生成的标记 | 根据场景设置 |
| logprobs | 返回 Token 的概率 | 调试时使用 |

---

## 1.5 主流模型对比与选型

### 1.5.1 OpenAI 系列

**GPT-4o**

- **定位：** OpenAI 的旗舰模型，平衡性能和成本
- **上下文窗口：** 128K Token
- **特点：** 多模态（文本+图像+音频）、速度快、性价比高
- **适用场景：** 通用 Agent 开发的首选
- **局限：** 中文能力略弱于专门优化的中文模型

**GPT-4o-mini**

- **定位：** 轻量级模型，成本极低
- **上下文窗口：** 128K Token
- **特点：** 速度快、成本低（输入 $0.15/百万Token）
- **适用场景：** 简单任务、高并发场景、成本敏感场景
- **局限：** 复杂推理能力不如 GPT-4o

**o1 / o3 系列**

- **定位：** 推理增强模型
- **特点：** 内置推理链，适合复杂推理任务
- **适用场景：** 数学、逻辑推理、复杂编程
- **局限：** 成本高、延迟高

### 1.5.2 Anthropic Claude 系列

**Claude Sonnet 4.6**

- **定位：** Anthropic 的平衡模型
- **上下文窗口：** 200K Token
- **特点：** 代码能力强、长上下文处理好、安全性高
- **适用场景：** 代码 Agent、长文档处理
- **局限：** 中文能力可能不如中文优化模型

**Claude Haiku 4.5**

- **定位：** 轻量级模型
- **特点：** 速度快、成本低
- **适用场景：** 简单任务、快速响应
- **局限：** 复杂任务能力有限

### 1.5.3 开源模型

**Llama 3.1（Meta）**

- **参数量：** 8B / 70B / 405B
- **特点：** 完全开源、社区活跃、可本地部署
- **适用场景：** 数据隐私要求高、需要定制化
- **局限：** 需要自行部署和维护

**Qwen 2.5（阿里通义）**

- **参数量：** 0.5B 到 72B
- **特点：** 中文能力强、多模态支持
- **适用场景：** 中文 Agent 开发
- **局限：** 国际社区相对较小

**DeepSeek V3**

- **参数量：** 671B MoE（混合专家）
- **特点：** 性价比极高、推理能力强
- **适用场景：** 复杂推理任务
- **局限：** MoE 架构的部署复杂度较高

### 1.5.4 选型决策框架

```
1. 数据隐私要求高吗？
   ├── 是 → 开源模型（本地部署）
   └── 否 → 2

2. 任务复杂度如何？
   ├── 简单（问答、摘要） → GPT-4o-mini 或 Claude Haiku
   ├── 中等（工具调用、RAG） → GPT-4o 或 Claude Sonnet
   └── 复杂（推理、编程） → GPT-4o 或 o1/o3

3. 中文能力要求高吗？
   ├── 是 → Qwen 2.5 或 DeepSeek
   └── 否 → GPT-4o 或 Claude Sonnet
```

---

## 1.6 案例分析：为什么 LLM 不能直接用于生产

### 场景描述

某电商公司想用 LLM 构建客服系统，直接让 LLM 回答用户关于产品、订单、物流的问题。

### 初始方案

```python
def customer_service(user_question: str) -> str:
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "你是一个客服助手，回答用户关于产品、订单、物流的问题。"},
            {"role": "user", "content": user_question},
        ],
    )
    return response.choices[0].message.content
```

### 遇到的问题

**问题 1：幻觉导致错误信息**

```
用户："你们的 XX 手机支持 5G 吗？"
LLM："是的，XX 手机支持 5G 网络。"
实际情况：该手机不支持 5G
后果：用户基于错误信息做出购买决策
```

**问题 2：知识截止导致过时信息**

```
用户："你们最新款手机是什么？"
LLM："你们最新款是 XX 手机（2023年发布）。"
实际情况：2024 年已经发布了新款
后果：用户认为公司信息过时
```

**问题 3：不一致性**

```
用户第一次问："退货政策是什么？"
LLM："7天无理由退货。"

用户第二次问（换个说法）："我不想要了可以退吗？"
LLM："需要在3天内申请退货。"

两次回答不一致，用户困惑
```

**问题 4：安全风险**

```
用户："帮我查一下订单 12345 的收货地址"
LLM：（可能泄露其他用户的地址信息）
```

### 改进方案：Agent 架构

```
用户问题
    ↓
[输入安全检查] → [Prompt Injection 检测]
    ↓
[意图识别] → 产品问题 / 订单问题 / 物流问题 / 投诉
    ↓
[工具调用]
    ├── 产品问题 → 调用产品数据库 API → 获取准确信息
    ├── 订单问题 → 调用订单系统 API → 获取订单状态
    ├── 物流问题 → 调用物流 API → 获取物流信息
    └── 投诉 → 转接人工客服
    ↓
[回答生成] → 基于真实数据生成回答
    ↓
[输出过滤] → 脱敏敏感信息
    ↓
返回给用户
```

**关键改进：**
1. 用 Tool Calling 解决幻觉问题（查询真实数据）
2. 用 RAG 解决知识截止问题（连接最新知识库）
3. 用 Guardrails 解决安全问题（输入验证、输出过滤）
4. 用 Human-in-the-Loop 解决高风险问题（投诉转人工）

---

## 1.7 代码实战

### 1.7.1 环境搭建

```bash
# 1. 创建项目目录
mkdir agent-tutorial && cd agent-tutorial

# 2. 创建虚拟环境
python -m venv venv

# 3. 激活虚拟环境
# Linux/macOS:
source venv/bin/activate
# Windows:
venv\Scripts\activate

# 4. 安装依赖
pip install openai anthropic python-dotenv tiktoken

# 5. 创建环境变量文件
echo 'OPENAI_API_KEY=your_key_here' > .env
echo 'ANTHROPIC_API_KEY=your_key_here' >> .env
```

### 1.7.2 第一个 LLM 调用

```python
"""
第 1 章示例 1：第一个 LLM API 调用

运行前请确保：
1. 已安装依赖：pip install openai python-dotenv
2. 已配置 .env 文件中的 OPENAI_API_KEY
"""
import os
from dotenv import load_dotenv
from openai import OpenAI

# 加载环境变量
load_dotenv()

# 初始化客户端
client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

# 基本调用
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "你是一个有帮助的助手。"},
        {"role": "user", "content": "用一句话解释什么是 AI Agent。"},
    ],
    temperature=0.7,
    max_tokens=100,
)

# 输出结果
print(f"回答：{response.choices[0].message.content}")
print(f"模型：{response.model}")
print(f"Token 使用：输入={response.usage.prompt_tokens}, 输出={response.usage.completion_tokens}, 总计={response.usage.total_tokens}")
```

### 1.7.3 参数调优实验

```python
"""
实验：Temperature 对输出多样性的影响
"""
import os
from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()
client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

prompt = "写一个关于人工智能的一句话描述"

print("Temperature 对输出的影响：")
print("=" * 60)

for temp in [0, 0.5, 1.0, 1.5]:
    print(f"\nTemperature = {temp}：")
    for i in range(3):  # 每个 temperature 运行 3 次
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": prompt}],
            temperature=temp,
            max_tokens=50,
        )
        print(f"  [{i+1}] {response.choices[0].message.content}")
```

### 1.7.4 Token 计数实验

```python
"""
实验：不同语言的 Token 消耗差异
"""
import tiktoken

enc = tiktoken.encoding_for_model("gpt-4o")

test_cases = [
    ("英文短句", "Hello, how are you today?"),
    ("中文短句", "你好，今天怎么样？"),
    ("英文长文", "Artificial intelligence is transforming the way we work and live. " * 10),
    ("中文长文", "人工智能正在改变我们的工作和生活方式。" * 10),
]

print("Token 消耗对比：")
print("=" * 60)

for name, text in test_cases:
    tokens = enc.encode(text)
    print(f"{name}：{len(text)} 字符 -> {len(tokens)} Token (比例: {len(tokens)/len(text):.2f})")
```

---

## 1.8 常见误区与最佳实践

### 1.8.1 常见误区

**误区 1：LLM 理解语义**

LLM 不是真正"理解"语义，而是学习了 Token 序列的统计模式。当模型说"我理解你的问题"时，它实际上是在根据统计模式生成一个"看起来像理解"的回复。

但这个区别在大多数工程场景下并不重要——重要的是模型的输出是否符合我们的需求。

**误区 2：更大的模型一定更好**

更大的模型通常更强，但也更慢、更贵。对于简单任务（如分类、摘要），小模型可能完全够用，甚至更好（因为更快、更便宜）。

**误区 3：Temperature 越低越好**

Temperature = 0 适合需要确定性的场景（如工具调用），但创意性任务需要一定的随机性。如果 Temperature = 0 用于创意写作，输出会非常单调。

**误区 4：上下文窗口越大越好**

更大的窗口意味着更高的成本和更慢的响应。对于大多数任务，精心管理的 8K 窗口可能比随意使用的 128K 窗口效果更好。

### 1.8.2 最佳实践

1. **始终使用环境变量管理 API Key** —— 永远不要硬编码
2. **根据任务选择合适的模型** —— 不要一律用最强的
3. **控制输出长度** —— 设置合理的 Max Tokens
4. **记录 Token 使用量** —— 监控成本
5. **为 Agent 设计 Tool Calling** —— 让 LLM 调用工具获取真实数据，而不是自己"猜测"
6. **测试不同 Temperature** —— 找到当前任务的最佳设置
7. **理解幻觉风险** —— 在关键场景中加入验证机制

---

## 1.9 练习题

### 概念理解题

1. **Token 是什么？** 为什么中文比英文消耗更多 Token？这对 Agent 的成本有什么影响？请用具体数字举例说明。

2. **上下文窗口包括哪些内容？** 如果一个模型的上下文窗口是 128K Token，System Prompt 占 1000 Token，工具描述占 3000 Token，对话历史占 50000 Token，那么留给输出的空间是多少？如果用户发送一个 5000 Token 的文档让 Agent 分析，会发生什么？

3. **幻觉和错误有什么区别？** 为什么 Agent 中的幻觉比普通聊天机器人更危险？请举一个具体场景说明。

4. **Temperature 和 Top-p 分别控制什么？** 在什么场景下应该设置 Temperature=0？在什么场景下应该设置 Temperature=1.0？

5. **涌现能力是什么？** 为什么小模型没有涌现能力？涌现能力对 Agent 的意义是什么？

### 动手实践题

1. **Token 计数实验：** 使用 tiktoken 库，计算以下文本的 Token 数量：
   - "Hello, World!"
   - "你好，世界！"
   - 一段 500 字的中文文章
   - 一段 500 词的英文文章
   
   观察中英文 Token 消耗的差异，并分析对 Agent 成本的影响。

2. **参数调优实验：** 对同一个 Prompt "写一个关于 AI 的一句话描述"，分别设置 Temperature 为 0、0.5、1.0、1.5，每个设置运行 5 次，观察输出的多样性。制作一个表格记录每次的输出。

3. **多模型对比：** 用同一个 Prompt "解释 Transformer 的自注意力机制" 分别调用 GPT-4o 和 Claude，对比两个模型的输出质量、长度和风格。

### 思考题

1. 如果你正在设计一个代码生成 Agent，上下文窗口需要容纳代码库的摘要、用户的请求、生成的代码。你会如何管理上下文窗口？请设计一个策略。

2. 幻觉问题在什么场景下是可以接受的？在什么场景下是绝对不能接受的？请举例说明。

3. 为什么说"LLM 不是真正理解语义"？这个区别在 Agent 开发中重要吗？为什么？

---

## 1.10 实战任务

**任务：搭建 Agent 开发环境并完成第一次 API 调用**

**具体要求：**

1. 安装 Python 3.10+ 和必要的依赖（openai、anthropic、python-dotenv、tiktoken）
2. 配置 OpenAI API Key（通过环境变量，不要硬编码）
3. 编写一个 Python 脚本，完成以下功能：
   - 调用 GPT-4o 模型
   - 发送一个包含 System Prompt 和 User Message 的请求
   - System Prompt 定义 Agent 的角色（例如"你是一个代码审查助手"）
   - 设置 Temperature = 0.3
   - 设置 Max Tokens = 200
   - 打印模型的回复
   - 打印 Token 使用情况（输入、输出、总计）
4. 编写第二个脚本，实现 Token 计数实验：
   - 对比中英文的 Token 消耗
   - 计算不同长度文本的 Token 数
5. 将代码和运行结果一起提交

**评估标准：**

| 维度 | 权重 | 评分标准 |
|------|------|---------|
| 代码正确性 | 30% | 代码能正确运行，输出符合预期 |
| API Key 管理 | 20% | 使用环境变量，不硬编码 |
| 代码质量 | 20% | 结构清晰，有适当注释 |
| 实验完整性 | 15% | 完成了所有实验要求 |
| 结果分析 | 15% | 能解释实验结果的含义 |

---

## 1.11 本章小结

### 核心知识点回顾

- **LLM 的本质：** 基于 Transformer 架构的概率语言模型，通过预测下一个 Token 来生成文本。它不是真正"理解"语义，而是学习了 Token 序列的统计模式。

- **四个核心阶段：** Tokenization（分词）→ Embedding（向量化）→ Transformer（处理）→ Generation（生成）

- **上下文窗口：** LLM 一次能处理的最大 Token 数，是 Agent 设计的核心约束。所有组件（System Prompt、对话历史、工具描述、RAG 结果）都在争夺有限的窗口空间。

- **幻觉问题：** LLM 生成错误内容的现象。在 Agent 中特别危险，因为 Agent 有执行能力，幻觉可能导致不可逆的错误操作。

- **模型参数：** Temperature 控制随机性，Top-p 控制采样范围，Max Tokens 控制输出长度。根据场景选择合适的参数。

- **涌现能力：** 大模型突然展现出的能力（推理、代码生成等），是 Agent 能工作的基础。

### 关键公式

```
Agent 的基础 = LLM（概率预测）+ 上下文窗口（有限空间）+ 工具调用（外部能力）
```

### 下一章预告

下一章我们将深入探讨 **Prompt Engineering** —— Agent 与 LLM 的核心通信接口。你将学习如何设计高质量的 Prompt，让 LLM 按照你的预期工作。Prompt 的质量直接决定 Agent 的行为质量，这是 Agent 工程师的核心技能之一。

---

## 配图清单

本章需要以下配图（建议使用 Draw.io 或 Mermaid 绘制）：

1. **Transformer 架构简化图**（优先级：高）
   - 展示 Encoder-Decoder 结构
   - 标注 Self-Attention 位置
   - 展示输入→输出的数据流

2. **Tokenization 过程图**（优先级：高）
   - 展示文本如何被切分成 Token
   - 对比中英文的分词结果
   - 标注 Token ID

3. **上下文窗口示意图**（优先级：中）
   - 展示上下文窗口的组成
   - 标注各部分的 Token 占用
   - 展示空间竞争关系

4. **幻觉成因分析图**（优先级：中）
   - 展示幻觉的四个成因
   - 展示幻觉 vs 错误的区别

5. **模型选型决策树**（优先级：中）
   - 展示选型决策流程
   - 标注不同场景的推荐模型

---

*本章代码示例见 [code/ch01-llm-basics/](../../code/ch01-llm-basics/)*
*下一章：[第 2 章：Prompt Engineering](../chapter-02/README.md)*
