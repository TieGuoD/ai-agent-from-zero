# 第 11 章：RAG 基础 —— 检索增强生成的原理

> **本章定位：** RAG（Retrieval-Augmented Generation，检索增强生成）是解决 LLM 知识截止和幻觉问题的核心技术。所有知识密集型 Agent 都依赖 RAG。本章将带你从零理解 RAG 的完整流程，从"为什么需要 RAG"开始，逐步深入到 Embedding、向量数据库、Chunking 等核心技术。

---

## 学习目标

1. **理解 RAG 的完整流程** —— 能描述从文档索引到检索到生成的完整链路
2. **理解 Embedding 的原理和作用** —— 知道文本如何变成向量，向量如何支持语义搜索
3. **掌握向量数据库的基本操作** —— 能使用 Chroma 进行文档索引和检索
4. **理解 Chunking 策略对检索质量的影响** —— 能选择合适的分块策略

## 核心问题

1. **RAG 如何解决 LLM 的知识截止问题？** 它和直接让 LLM 回答有什么区别？
2. **为什么文本需要转换为向量？** 向量搜索和关键词搜索有什么区别？
3. **如何选择合适的 Chunking 策略？** 不同的分块方式对检索质量有什么影响？

---

## 11.1 为什么需要 RAG

### 11.1.1 LLM 的三个核心问题

让我们从一个具体的故事开始。

想象你是一个律师，正在处理一个案件。你需要查找最新的法律法规来支持你的论点。你问 LLM："2024 年新修订的《公司法》有哪些变化？"

LLM 可能会回答："2024 年新修订的《公司法》主要变化包括..."——但这些信息可能是编造的。因为 LLM 的训练数据有截止日期，它可能根本不知道 2024 年的法律变化。

这就是 LLM 的三个核心问题：

**问题 1：知识截止**

LLM 的训练数据有截止日期。比如 GPT-4o 的训练数据截止到 2024 年 4 月，之后发生的事情它不知道。对于需要最新信息的场景（新闻、法规、产品更新），LLM 无法提供准确答案。

**问题 2：幻觉**

LLM 可能编造不存在的信息。当它不确定答案时，它会根据统计模式"猜测"一个看起来合理的答案，但这个答案可能是错误的。

**问题 3：无私有知识**

LLM 不知道你的私有数据。你公司的内部文档、你的个人笔记、你的数据库——LLM 都不知道。它只能基于公开的训练数据回答问题。

### 11.1.2 RAG 的解决方案

RAG 的核心思想是：**在生成回答前，先从外部知识库检索相关信息，用检索结果增强 LLM 的输入。**

```
传统 LLM：问题 → LLM → 猜测答案（可能有幻觉）

RAG：问题 → 检索知识库 → 获取相关文档 → 将文档注入 Prompt → LLM → 基于真实数据回答
```

**RAG 的优势：**

1. **解决知识截止：** 知识库可以随时更新，检索到的信息是最新的
2. **减少幻觉：** 回答基于检索到的真实文档，不是 LLM 猜测的
3. **支持私有知识：** 可以将私有文档加入知识库，让 LLM 能回答相关问题

### 11.1.3 RAG 的一个具体例子

```
用户："Python 3.13 有什么新特性？"

没有 RAG：
  LLM：（可能编造）Python 3.13 引入了 XXX 语法...

有 RAG：
  1. 检索知识库：找到一篇关于 Python 3.13 的文章
  2. 文章内容："Python 3.13 引入了更好的错误消息、性能优化、新语法..."
  3. LLM 基于文章回答："根据官方文档，Python 3.13 主要新特性包括：更好的错误消息、性能优化约 5%、新的类型注解语法..."
```

---

## 11.2 RAG 完整流程

### 11.2.1 两个阶段

RAG 的流程分为两个阶段：离线的索引阶段和在线的检索生成阶段。

**索引阶段（离线）**

```
文档集合
    ↓
[文档加载] → 读取各种格式的文档（PDF、Word、网页等）
    ↓
[文本分块] → 将长文档切成合适大小的块（Chunk）
    ↓
[向量化] → 使用 Embedding 模型将文本转换为向量
    ↓
[存储] → 将向量存入向量数据库
```

**检索生成阶段（在线）**

```
用户问题
    ↓
[问题向量化] → 将问题转换为向量
    ↓
[向量搜索] → 在向量数据库中搜索最相似的文档块
    ↓
[结果排序] → 按相关性排序，选择最相关的 K 个结果
    ↓
[上下文构建] → 将检索结果注入 Prompt
    ↓
[LLM 生成] → 基于检索结果生成回答
```

### 11.2.2 每一步的作用

让我用一个比喻来解释每一步的作用。想象你是一个图书馆管理员，有人来问你一个问题。

**文档加载** 就像把所有书都搬到图书馆。你需要把各种格式的资料（PDF、Word、网页）都收集起来。

**文本分块** 就像给每本书的每个章节做索引。你不能把整本书都塞给提问者，需要找到最相关的章节。

**向量化** 就像给每个章节做"语义标签"。传统的标签是关键词（"Python"、"编程"），而向量标签能捕捉语义（"Python 是一种编程语言"和"Python 用于软件开发"语义相近）。

**向量搜索** 就像根据语义标签查找最相关的章节。提问者问"Python 编程"，你能找到"Python 是一种编程语言"和"Python 用于软件开发"这两个章节，即使它们没有完全相同的关键词。

---

## 11.3 Embedding（嵌入）

### 11.3.1 什么是 Embedding

Embedding 是将文本转换为高维向量的过程。每个文本被映射到一个数百或数千维的向量空间中。

```
"Python 是一种编程语言" → [0.023, -0.156, 0.892, ..., 0.341]  # 1536 维向量
```

向量空间的关键特性是：**语义相近的文本，向量距离也相近。**

```
"Python 是一种编程语言" 和 "Python 用于软件开发" → 向量距离近
"Python 是一种编程语言" 和 "今天天气很好" → 向量距离远
```

### 11.3.2 为什么需要 Embedding

传统的关键词搜索有一个根本问题：它无法理解语义。

```
用户搜索："Python 编程"

关键词搜索：只找包含"Python"和"编程"这两个词的文档
  ✓ "Python 是一种编程语言"  （包含两个关键词）
  ✗ "Python 用于软件开发"    （没有"编程"这个词，但语义相关）
  ✗ "Guido van Rossum 创建了 Python"  （没有"编程"这个词，但语义相关）

向量搜索：找语义相似的文档
  ✓ "Python 是一种编程语言"  （语义完全匹配）
  ✓ "Python 用于软件开发"    （语义相近）
  ✓ "Guido van Rossum 创建了 Python"  （语义相关）
```

这就是 Embedding 的价值——它让搜索从"关键词匹配"升级到"语义匹配"。

### 11.3.3 如何生成 Embedding

```python
from openai import OpenAI
import os

client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

# 单个文本的 Embedding
response = client.embeddings.create(
    model="text-embedding-3-small",
    input="Python 是一种编程语言",
)

vector = response.data[0].embedding
print(f"向量维度：{len(vector)}")  # 1536

# 批量 Embedding
texts = [
    "Python 是一种编程语言",
    "Python 用于软件开发",
    "今天天气很好",
]

response = client.embeddings.create(
    model="text-embedding-3-small",
    input=texts,
)

vectors = [d.embedding for d in response.data]
print(f"生成了 {len(vectors)} 个向量")
```

### 11.3.4 语义相似度计算

```python
import numpy as np


def cosine_similarity(v1, v2):
    """计算余弦相似度"""
    return np.dot(v1, v2) / (np.linalg.norm(v1) * np.linalg.norm(v2))


# 获取三个句子的向量
texts = [
    "Python 是一种编程语言",
    "Python 用于软件开发",
    "今天天气很好",
]

vectors = []
for text in texts:
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=text,
    )
    vectors.append(response.data[0].embedding)

# 计算相似度
print(f"句1 vs 句2：{cosine_similarity(vectors[0], vectors[1]):.3f}")  # 高（~0.85）
print(f"句1 vs 句3：{cosine_similarity(vectors[0], vectors[2]):.3f}")  # 低（~0.20）
```

---

## 11.4 Chunking（分块）

### 11.4.1 为什么需要分块

LLM 的上下文窗口有限（比如 128K Token），不能把整个文档塞进去。一个 100 页的 PDF 可能有 50,000+ Token，远超上下文窗口。更糟糕的是，即使能塞进去，LLM 也很难从海量信息中找到最相关的部分。

分块的目的是将长文档切成合适大小的块，每个块都包含完整的语义信息。

### 11.4.2 分块策略

**策略 1：固定大小分块**

最简单的分块方式：按字符数或 Token 数分块。

```python
def chunk_by_size(text: str, chunk_size: int = 500, overlap: int = 50) -> list[str]:
    """固定大小分块，带重叠"""
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunk = text[start:end]
        chunks.append(chunk)
        start = end - overlap  # 重叠部分，避免切断语义
    return chunks
```

**优点：** 实现简单，块大小一致。
**缺点：** 可能在句子中间切断，破坏语义完整性。

**策略 2：按句子分块**

按句子边界分块，保持语义完整性。

```python
import re


def chunk_by_sentences(text: str) -> list[str]:
    """按句子分块"""
    sentences = re.split(r'[。！？\n]', text)
    return [s.strip() for s in sentences if s.strip()]
```

**优点：** 保持语义完整。
**缺点：** 块大小不一致，可能太小或太大。

**策略 3：按段落分块**

按段落边界分块。

```python
def chunk_by_paragraphs(text: str) -> list[str]:
    """按段落分块"""
    paragraphs = text.split('\n\n')
    return [p.strip() for p in paragraphs if p.strip()]
```

**优点：** 语义完整性最好。
**缺点：** 块大小差异可能很大。

### 11.4.3 分块大小的影响

分块大小对检索质量有直接影响。块太小，缺少足够上下文，检索到的信息可能不完整。块太大，包含太多无关信息，LLM 难以找到关键内容。一般来说，200-1000 个字符是比较合适的范围。

---

## 11.5 向量数据库

### 11.5.1 什么是向量数据库

向量数据库是专门存储和检索向量数据的数据库。它支持高效的相似度搜索——给定一个查询向量，快速找到最相似的 K 个向量。

### 11.5.2 Chroma（入门推荐）

Chroma 是最简单的向量数据库，适合入门和小规模应用。

```python
import chromadb

# 创建客户端（内存模式）
client = chromadb.Client()

# 创建集合
collection = client.create_collection("documents")

# 添加文档
collection.add(
    documents=[
        "Python 是一种编程语言",
        "Java 也是一种编程语言",
        "今天天气很好",
    ],
    ids=["doc1", "doc2", "doc3"],
    metadatas=[
        {"source": "wiki", "topic": "programming"},
        {"source": "wiki", "topic": "programming"},
        {"source": "weather", "topic": "weather"},
    ],
)

# 查询
results = collection.query(
    query_texts=["什么语言用于编程"],
    n_results=2,
)

print(results)
# {'documents': [['Python 是一种编程语言', 'Java 也是一种编程语言']], ...}
```

### 11.5.3 生产级选择

| 数据库 | 特点 | 适用场景 | 部署方式 |
|--------|------|---------|---------|
| Chroma | 简单、本地 | 开发、小规模 | 本地/嵌入 |
| Pinecone | 托管、易用 | 中等规模 | 托管服务 |
| Weaviate | 功能丰富 | 大规模、复杂查询 | 自部署/托管 |
| Milvus | 高性能 | 大规模、高性能 | 自部署 |
| Qdrant | Rust 实现 | 高性能 | 自部署/托管 |

---

## 11.6 完整 RAG 实现

### 11.6.1 代码实现

```python
from openai import OpenAI
import chromadb
import os


class SimpleRAG:
    """简单的 RAG 系统"""

    def __init__(self, api_key: str = None):
        self.client = OpenAI(api_key=api_key or os.getenv("OPENAI_API_KEY"))
        self.chroma = chromadb.Client()
        self.collection = self.chroma.create_collection("documents")

    def index(self, documents: list[str], ids: list[str], metadatas: list[dict] = None):
        """
        索引文档

        参数:
            documents: 文档列表
            ids: 文档 ID 列表
            metadatas: 元数据列表（可选）
        """
        # 生成 Embeddings
        response = self.client.embeddings.create(
            model="text-embedding-3-small",
            input=documents,
        )
        embeddings = [d.embedding for d in response.data]

        # 存入向量数据库
        self.collection.add(
            documents=documents,
            embeddings=embeddings,
            ids=ids,
            metadatas=metadatas or [{}] * len(documents),
        )

        print(f"已索引 {len(documents)} 个文档")

    def retrieve(self, query: str, top_k: int = 3) -> list[str]:
        """
        检索相关文档

        参数:
            query: 查询文本
            top_k: 返回最相关的 K 个结果

        返回:
            相关文档列表
        """
        results = self.collection.query(
            query_texts=[query],
            n_results=top_k,
        )
        return results["documents"][0]

    def generate(self, query: str, context: list[str]) -> str:
        """
        基于检索结果生成回答

        参数:
            query: 用户问题
            context: 检索到的上下文

        返回:
            生成的回答
        """
        context_text = "\n".join([f"- {doc}" for doc in context])

        response = self.client.chat.completions.create(
            model="gpt-4o",
            messages=[
                {
                    "role": "system",
                    "content": f"基于以下信息回答问题。如果信息不足以回答，请说'根据现有信息无法回答'。\n\n参考信息：\n{context_text}",
                },
                {"role": "user", "content": query},
            ],
            temperature=0.3,
        )
        return response.choices[0].message.content

    def query(self, question: str, top_k: int = 3) -> str:
        """
        完整 RAG 流程

        参数:
            question: 用户问题
            top_k: 检索数量

        返回:
            回答
        """
        # 1. 检索
        context = self.retrieve(question, top_k)

        # 2. 生成
        return self.generate(question, context)
```

### 11.6.2 使用示例

```python
# 创建 RAG 系统
rag = SimpleRAG()

# 索引文档
documents = [
    "Python 3.13 于 2024 年 10 月发布，主要新特性包括性能提升、错误消息改进等。",
    "Python 3.12 于 2023 年 10 月发布，引入了新的类型注解语法。",
    "Python 是一种高级编程语言，由 Guido van Rossum 于 1991 年创建。",
    "Java 是一种面向对象的编程语言，由 Sun Microsystems 于 1995 年发布。",
]

rag.index(
    documents=documents,
    ids=["doc1", "doc2", "doc3", "doc4"],
)

# 查询
answer = rag.query("Python 最新版本是什么？")
print(answer)
# 基于检索结果，Python 最新版本是 3.13，于 2024 年 10 月发布...
```

---

## 11.7 文档加载的深入理解

### 11.7.1 不同格式的文档加载

实际应用中，文档来源多种多样。一个成熟的 RAG 系统需要支持多种文档格式。

```python
from pathlib import Path


class DocumentLoader:
    """文档加载器：支持多种格式"""

    def load(self, path: str) -> list[dict]:
        """根据文件扩展名选择加载方式"""
        file_path = Path(path)

        if file_path.suffix == ".txt":
            return self._load_text(path)
        elif file_path.suffix == ".md":
            return self._load_markdown(path)
        elif file_path.suffix == ".pdf":
            return self._load_pdf(path)
        else:
            return self._load_text(path)

    def _load_text(self, path: str) -> list[dict]:
        """加载纯文本文件"""
        with open(path, "r", encoding="utf-8") as f:
            content = f.read()
        return [{"content": content, "source": path, "type": "text"}]

    def _load_markdown(self, path: str) -> list[dict]:
        """加载 Markdown 文件，按标题分段"""
        with open(path, "r", encoding="utf-8") as f:
            content = f.read()

        sections = []
        current_section = []
        for line in content.split("\n"):
            if line.startswith("# ") and current_section:
                sections.append("\n".join(current_section))
                current_section = [line]
            else:
                current_section.append(line)

        if current_section:
            sections.append("\n".join(current_section))

        return [
            {"content": s, "source": path, "type": "markdown"}
            for s in sections if s.strip()
        ]

    def _load_pdf(self, path: str) -> list[dict]:
        """加载 PDF 文件（简化版）"""
        try:
            import PyPDF2
            pages = []
            with open(path, "rb") as f:
                reader = PyPDF2.PdfReader(f)
                for i, page in enumerate(reader.pages):
                    pages.append({
                        "content": page.extract_text(),
                        "source": path,
                        "page": i + 1,
                        "type": "pdf",
                    })
            return pages
        except ImportError:
            return [{"content": "PDF 支持未安装，请安装 PyPDF2", "source": path}]
```

### 11.7.2 文档元数据的价值

元数据在 RAG 系统中非常重要，它可以帮助过滤和排序检索结果。

```python
# 常用的元数据字段
metadata_examples = {
    "source": "文档来源（URL、文件路径）",
    "title": "文档标题",
    "author": "作者",
    "created_at": "创建时间",
    "updated_at": "更新时间",
    "category": "分类标签",
    "language": "语言",
    "chunk_index": "块在文档中的位置",
    "doc_length": "原始文档长度",
}

# 使用元数据过滤检索结果
results = collection.query(
    query_texts=["Python 编程"],
    n_results=5,
    where={"category": "programming"},  # 只检索编程类文档
)
```

---

## 11.8 常见坑

---

## 11.7 常见坑

### 11.7.1 Chunk 太大或太小

块太小，缺少足够上下文，检索到的信息不完整。块太大，包含太多无关信息，LLM 难以找到关键内容。一般来说，200-1000 个字符是比较合适的范围。

### 11.7.2 Embedding 模型选择不当

不同的 Embedding 模型对不同语言的支持不同。中文场景需要使用中文优化的模型（如 BGE-M3），而不是英文模型。

### 11.7.3 不考虑重叠

固定大小分块如果不考虑重叠，可能在句子中间切断，破坏语义完整性。添加 10-20% 的重叠可以缓解这个问题。

---

## 11.8 练习题

### 概念理解题

1. RAG 的完整流程是什么？每个步骤的作用是什么？

2. 为什么文本需要转换为向量？向量搜索和关键词搜索有什么区别？

3. 不同的 Chunking 策略各有什么优缺点？在什么场景下应该用哪种？

4. 向量数据库和传统数据库有什么区别？

5. RAG 的检索质量受哪些因素影响？

### 动手实践题

1. **Embedding 实验：** 使用 OpenAI API 生成几个句子的 Embedding，计算它们之间的相似度。

2. **RAG 实现：** 用 Chroma 实现一个简单的 RAG 系统，索引 10 个文档，测试 5 个查询。

3. **Chunking 实验：** 对同一个文档，使用不同的 Chunking 策略，对比检索效果。

### 思考题

1. RAG 有什么局限性？在什么场景下 RAG 不适用？

2. 如何评估 RAG 系统的效果？有哪些评估指标？

---

## 11.9 实战任务

**任务：用 Chroma 实现一个简单的 RAG 系统**

**具体要求：**

1. 实现文档索引功能
2. 实现语义检索功能
3. 实现基于检索结果的回答生成
4. 索引至少 10 个文档
5. 测试 5 个查询，分析检索质量

**评估标准：**

| 维度 | 权重 | 评分标准 |
|------|------|---------|
| 索引功能 | 20% | 能正确索引文档 |
| 检索质量 | 30% | 检索结果相关性高 |
| 生成质量 | 25% | 回答准确、基于检索结果 |
| 代码质量 | 15% | 结构清晰，可维护 |
| 测试完整性 | 10% | 测试用例全面 |

---

## 11.10 RAG 系统的性能优化

### 11.10.1 索引阶段优化

RAG 系统的索引阶段是一次性的离线操作，但索引质量直接影响检索效果。以下是几个关键的优化点。

**批量 Embedding**：逐个文档生成 Embedding 效率很低。大多数 Embedding API 支持批量处理，一次请求可以处理多个文档。这不仅提高了效率，还能降低 API 调用成本。

**增量索引**：当文档库更新时，不需要重新索引所有文档。只需要为新增和修改的文档生成 Embedding，然后添加到向量数据库中。

**元数据过滤**：在索引时为每个文档块添加元数据（如来源、时间、分类），检索时可以利用元数据进行过滤，提高检索的精确性。

```python
class OptimizedIndexer:
    """优化的文档索引器"""

    def __init__(self, embedding_model, vector_store):
        self.model = embedding_model
        self.store = vector_store

    def batch_index(self, documents: list[dict], batch_size: int = 100):
        """批量索引文档"""
        for i in range(0, len(documents), batch_size):
            batch = documents[i:i + batch_size]
            texts = [doc["content"] for doc in batch]

            # 批量生成 Embedding
            embeddings = self.model.embed(texts)

            # 批量存储
            self.store.add(
                texts=texts,
                embeddings=embeddings,
                metadatas=[doc.get("metadata", {}) for doc in batch],
                ids=[doc["id"] for doc in batch],
            )

    def incremental_update(self, updated_docs: list[dict]):
        """增量更新索引"""
        # 删除旧的
        ids = [doc["id"] for doc in updated_docs]
        self.store.delete(ids=ids)

        # 添加新的
        self.batch_index(updated_docs)
```

### 11.10.2 检索阶段优化

检索阶段的优化目标是在保证召回率的前提下，提高检索速度和精确性。

**查询缓存**：对于相同的查询，直接返回缓存结果，避免重复的向量搜索。这对于高频查询特别有效。

**异步检索**：在 Agent 等待 LLM 响应的同时，可以异步预检索相关信息，减少整体延迟。

**分层检索**：先用轻量级模型快速筛选候选文档，再用精确模型对候选文档进行排序。这平衡了速度和精度。

```python
class OptimizedRetriever:
    """优化的检索器"""

    def __init__(self, vector_store, cache):
        self.store = vector_store
        self.cache = cache

    def retrieve(self, query: str, top_k: int = 3) -> list[str]:
        """优化的检索流程"""
        # 1. 检查缓存
        cache_key = f"query:{query}:k:{top_k}"
        cached = self.cache.get(cache_key)
        if cached:
            return cached

        # 2. 向量搜索
        results = self.store.query(query_texts=[query], n_results=top_k * 2)

        # 3. 去重和截断
        unique_results = list(dict.fromkeys(results["documents"][0]))
        final_results = unique_results[:top_k]

        # 4. 缓存结果
        self.cache.set(cache_key, final_results, ttl=3600)

        return final_results
```

### 11.10.3 生成阶段优化

生成阶段的优化目标是提高回答质量和减少 Token 消耗。

**Prompt 精简**：只将最相关的检索结果注入 Prompt，避免注入过多无关信息。过多的信息反而会干扰 LLM 的判断。

**引用标注**：在检索结果中标注来源，让 LLM 在生成回答时能引用具体来源，提高回答的可信度。

**分段生成**：对于复杂问题，可以分段生成回答。先生成回答的结构，再逐段填充内容。

---

## 11.11 本章小结

### 核心知识点回顾

- **RAG 的核心思想：** 在生成回答前，先从外部知识库检索相关信息，用检索结果增强 LLM 的输入。这解决了 LLM 的知识截止、幻觉和无私有知识三个问题。

- **RAG 的完整流程：** 索引阶段（文档加载→分块→向量化→存储）+ 检索生成阶段（问题向量化→向量搜索→上下文构建→LLM 生成）。

- **Embedding：** 将文本转换为向量，支持语义搜索。语义相近的文本，向量距离也相近。

- **Chunking：** 将长文档切成合适大小的块。块大小影响检索质量，200-1000 字符是合适的范围。

- **向量数据库：** 专门存储和检索向量数据。Chroma 适合入门，Pinecone/Weaviate 适合生产。

### 实践建议

学习 RAG 最好的方式是动手实践。建议你先用 Chroma 搭建一个简单的 RAG 系统，索引一些你熟悉的文档（比如你自己写的技术笔记），然后尝试各种查询，观察检索结果的质量。在这个基础上，逐步尝试优化分块策略、更换 Embedding 模型、实现 Hybrid Search。每一次优化都能让你对 RAG 有更深的理解。

### 关键公式

```
RAG(question) = retrieve(question, index) → relevant_chunks
              + augment(question, relevant_chunks) → enhanced_prompt
              + generate(enhanced_prompt) → answer
```

### 下一章预告

下一章我们将深入探讨 **RAG 进阶** —— 检索质量优化。你将学习 Hybrid Search、Re-ranking、查询改写等高级技术，让 RAG 的检索质量更上一层楼。

---

*上一章：[第 10 章：构建你的第一个完整 Agent](../chapter-10/README.md)*
*下一章：[第 12 章：RAG 进阶](../chapter-12/README.md)*

RAG 技术是 Agent 获取外部知识的关键能力。掌握了 RAG，你就掌握了让 Agent 从"猜测答案"变成"基于真实数据回答"的核心技术。在下一章中，我们将深入探讨 RAG 的进阶优化技术，让你的 RAG 系统达到生产级的质量。

记住，RAG 的核心思想很简单：先检索，再生成。但要让这个简单的思想在实际应用中发挥作用，需要在分块策略、Embedding 模型选择、向量数据库配置、Prompt 设计等多个环节进行精心的设计和优化。RAG 是一个值得深入学习和实践的技术方向，它在 Agent 开发中扮演着越来越重要的角色。随着 LLM 上下文窗口的不断扩大，RAG 的实现方式也在不断演进，但其核心价值——让 LLM 基于真实数据回答问题——不会改变。掌握了 RAG，你就掌握了让 Agent 从"猜测"变成"基于事实回答"的关键能力。在实践中不断优化你的 RAG 系统，你会对这项技术有越来越深的理解。RAG 技术的学习之旅才刚刚开始，保持好奇心和探索精神，你会发现更多有趣的可能性。祝你在 RAG 的学习和实践中取得丰硕的成果，构建出真正有价值的 RAG 应用。

---

*上一章：[第 10 章：构建你的第一个完整 Agent](../chapter-10/README.md)*
*下一章：[第 12 章：RAG 进阶](../chapter-12/README.md)*

> **练习提示：** 如果你在实现过程中遇到困难，可以参考本章的代码示例。关键是要理解每一步的原理，而不只是复制代码。
