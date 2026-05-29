# 第 12 章：RAG 进阶 —— 检索质量优化

> **本章定位：** 基础 RAG 的检索质量往往不够好——检索结果可能不相关、不完整、或者排序不当。优化检索质量是 RAG 系统实用化的关键。本章将带你深入理解 Hybrid Search、Re-ranking、查询改写等高级技术，让 RAG 的检索质量更上一层楼。

---

## 学习目标

1. **掌握高级检索策略** —— Hybrid Search、Re-ranking、Metadata Filtering 的原理和实现
2. **理解查询改写和扩展技术** —— 如何让用户的查询更有效地匹配文档
3. **掌握 RAG 评估方法** —— 如何衡量 RAG 系统的效果
4. **了解 GraphRAG 等新范式** —— RAG 的未来发展方向

## 核心问题

1. **如何提高 RAG 的检索质量？** 检索结果不相关怎么办？
2. **Hybrid Search 为什么比纯向量搜索效果好？** 它结合了什么优势？
3. **如何评估 RAG 系统的效果？** 有哪些评估指标？

---

## 12.1 为什么基础 RAG 不够好

### 12.1.1 基础 RAG 的局限

上一章我们实现了一个简单的 RAG 系统。但在实际使用中，你可能会发现它有很多问题。

**问题 1：检索结果不相关**

用户搜索"Python 最新版本"，但检索结果可能是关于 Python 历史的文档，而不是关于最新版本的。

**问题 2：检索结果不完整**

用户搜索"Python 和 Java 的区别"，但检索结果只包含了 Python 的信息，没有 Java 的。

**问题 3：关键词匹配失败**

用户搜索"3.13 新特性"，但文档中写的是"Python 3.13 引入了..."，关键词不完全匹配。

**问题 4：排序不当**

最相关的结果排在后面，不相关的排在前面。

### 12.1.2 问题的根源

这些问题的根源在于：纯向量搜索虽然能捕捉语义，但在某些场景下不够精确。向量搜索擅长"语义匹配"，但在"精确匹配"方面不如关键词搜索。

比如，用户搜索"Python 3.13"，向量搜索可能返回关于 Python 的通用文档，而关键词搜索能精确匹配包含"3.13"的文档。

---

## 12.2 Hybrid Search（混合搜索）

### 12.2.1 核心思想

Hybrid Search 结合了向量搜索和关键词搜索的优势。向量搜索擅长语义匹配，关键词搜索擅长精确匹配。两者结合，效果最好。

```
用户搜索："Python 3.13 新特性"

纯向量搜索：返回关于 Python 的通用文档（语义匹配）
纯关键词搜索：返回包含"3.13"的文档（精确匹配）
混合搜索：两者结合，返回既语义相关又包含关键词的文档
```

### 12.2.2 实现

```python
class HybridSearch:
    """混合搜索：结合向量搜索和关键词搜索"""

    def __init__(self, vector_weight: float = 0.7, keyword_weight: float = 0.3):
        """
        参数:
            vector_weight: 向量搜索的权重
            keyword_weight: 关键词搜索的权重
        """
        self.vector_weight = vector_weight
        self.keyword_weight = keyword_weight

    def search(
        self,
        query: str,
        vector_results: list[str],
        keyword_results: list[str],
        top_k: int = 5,
    ) -> list[str]:
        """
        混合搜索

        参数:
            query: 查询
            vector_results: 向量搜索结果
            keyword_results: 关键词搜索结果
            top_k: 返回数量

        返回:
            混合后的结果
        """
        # 合并结果，计算分数
        combined = {}

        # 向量搜索结果
        for i, doc in enumerate(vector_results):
            score = self.vector_weight * (1 / (i + 1))  # 排名越靠前分数越高
            combined[doc] = combined.get(doc, 0) + score

        # 关键词搜索结果
        for i, doc in enumerate(keyword_results):
            score = self.keyword_weight * (1 / (i + 1))
            combined[doc] = combined.get(doc, 0) + score

        # 按分数排序
        sorted_results = sorted(combined.items(), key=lambda x: x[1], reverse=True)

        return [doc for doc, score in sorted_results[:top_k]]
```

### 12.2.3 Chroma 中的混合搜索

Chroma 本身不直接支持混合搜索，但我们可以通过组合实现：

```python
import chromadb
import re
from collections import Counter


class HybridRAG:
    """使用混合搜索的 RAG"""

    def __init__(self):
        self.chroma = chromadb.Client()
        self.collection = self.chroma.create_collection("documents")
        self.documents = []  # 保存原始文档

    def index(self, documents: list[str], ids: list[str]):
        """索引文档"""
        self.documents = documents
        self.collection.add(documents=documents, ids=ids)

    def hybrid_search(self, query: str, top_k: int = 3) -> list[str]:
        """混合搜索"""
        # 1. 向量搜索
        vector_results = self.collection.query(
            query_texts=[query],
            n_results=top_k * 2,
        )["documents"][0]

        # 2. 关键词搜索
        keyword_results = self._keyword_search(query, top_k * 2)

        # 3. 混合
        hybrid = HybridSearch()
        return hybrid.search(query, vector_results, keyword_results, top_k)

    def _keyword_search(self, query: str, top_k: int) -> list[str]:
        """简单的关键词搜索"""
        query_words = set(query.lower().split())
        scored = []
        for doc in self.documents:
            doc_words = set(doc.lower().split())
            overlap = len(query_words & doc_words)
            if overlap > 0:
                scored.append((overlap, doc))
        scored.sort(key=lambda x: x[0], reverse=True)
        return [doc for _, doc in scored[:top_k]]
```

---

## 12.3 Re-ranking（重排序）

### 12.3.1 什么是 Re-ranking

Re-ranking 是对初始检索结果进行二次排序。初始检索（如向量搜索）使用轻量级模型快速筛选，Re-ranking 使用更精确的模型评估文档与查询的真实相关性。

```
初始检索：从 10000 个文档中快速筛选出 20 个候选
Re-ranking：对 20 个候选进行精确排序，返回最相关的 5 个
```

### 12.3.2 实现

```python
from sentence_transformers import CrossEncoder


class Reranker:
    """重排序器"""

    def __init__(self, model_name: str = "cross-encoder/ms-marco-MiniLM-L-6-v2"):
        self.model = CrossEncoder(model_name)

    def rerank(self, query: str, documents: list[str], top_k: int = 3) -> list[str]:
        """
        重排序

        参数:
            query: 查询
            documents: 候选文档
            top_k: 返回数量

        返回:
            重排序后的文档
        """
        if not documents:
            return []

        # 构建查询-文档对
        pairs = [(query, doc) for doc in documents]

        # 计算相关性分数
        scores = self.model.predict(pairs)

        # 按分数排序
        scored_docs = list(zip(documents, scores))
        scored_docs.sort(key=lambda x: x[1], reverse=True)

        return [doc for doc, score in scored_docs[:top_k]]
```

### 12.3.3 为什么 Re-ranking 有效

初始检索（如向量搜索）使用的是轻量级模型，速度快但精度有限。Re-ranking 使用的是交叉编码器（Cross Encoder），它同时考虑查询和文档的交互，精度更高但速度慢。

两阶段的设计平衡了速度和精度：第一阶段快速筛选，第二阶段精确排序。

---

## 12.4 查询改写

### 12.4.1 查询扩展

用户的查询可能太简短或不够精确。查询扩展通过生成多个相关查询来提高召回率。

```python
def expand_query(query: str, llm_fn, num_expansions: int = 3) -> list[str]:
    """
    查询扩展

    参数:
        query: 原始查询
        llm_fn: LLM 函数
        num_expansions: 扩展数量

    返回:
        扩展后的查询列表
    """
    prompt = f"""请为以下查询生成 {num_expansions} 个相关的搜索查询。

原始查询：{query}

请生成 {num_expansions} 个不同的搜索查询，每个查询从不同角度搜索相关信息。"""

    response = llm_fn(prompt)

    # 解析扩展查询
    expanded = [query]  # 保留原始查询
    for line in response.split("\n"):
        line = line.strip()
        if line and line != query:
            expanded.append(line)

    return expanded[:num_expansions + 1]
```

### 12.4.2 HyDE（Hypothetical Document Embeddings）

HyDE 的思想是：先让 LLM 生成一个"假设性文档"（包含可能的答案），然后用这个文档的 Embedding 进行搜索。

```python
def hyde_search(query: str, llm_fn, embedding_fn, index) -> list[str]:
    """
    HyDE 搜索

    参数:
        query: 用户查询
        llm_fn: LLM 函数
        embedding_fn: Embedding 函数
        index: 向量索引

    返回:
        搜索结果
    """
    # 1. 生成假设性文档
    prompt = f"""请写一段可能包含以下问题答案的文档片段。

问题：{query}

文档片段："""

    hypothetical_doc = llm_fn(prompt)

    # 2. 用假设性文档的 Embedding 搜索
    query_embedding = embedding_fn(hypothetical_doc)

    # 3. 在索引中搜索
    results = index.search(query_embedding, top_k=5)

    return results
```

---

## 12.5 RAG 评估

### 12.5.1 评估维度

**检索质量：** 检索到的文档是否相关？使用 Recall@K（前 K 个结果中包含正确答案的比例）和 MRR（第一个正确结果的排名倒数）来衡量。

**生成质量：** 生成的回答是否准确？使用 Faithfulness（回答是否基于检索到的文档）和 Relevance（回答是否相关）来衡量。

### 12.5.2 实现评估

```python
import math


class RAGEvaluator:
    """RAG 评估器"""

    def __init__(self, rag_system):
        self.rag = rag_system

    def recall_at_k(
        self,
        test_cases: list[dict],
        k: int = 3,
    ) -> float:
        """Recall@K：前 K 个结果中包含正确答案的比例"""
        hits = 0
        for case in test_cases:
            results = self.rag.retrieve(case["query"], top_k=k)
            expected = case["expected_answer"]
            if any(expected.lower() in doc.lower() for doc in results):
                hits += 1
        return hits / len(test_cases) if test_cases else 0

    def mrr(
        self,
        test_cases: list[dict],
        k: int = 10,
    ) -> float:
        """MRR（Mean Reciprocal Rank）：第一个正确结果的排名倒数的平均值"""
        rr_sum = 0
        for case in test_cases:
            results = self.rag.retrieve(case["query"], top_k=k)
            expected = case["expected_answer"]
            rr = 0
            for i, doc in enumerate(results):
                if expected.lower() in doc.lower():
                    rr = 1 / (i + 1)
                    break
            rr_sum += rr
        return rr_sum / len(test_cases) if test_cases else 0

    def ndcg_at_k(
        self,
        test_cases: list[dict],
        k: int = 5,
    ) -> float:
        """NDCG@K（Normalized Discounted Cumulative Gain）"""
        total = 0
        for case in test_cases:
            results = self.rag.retrieve(case["query"], top_k=k)
            expected = case["expected_answer"]

            # 计算 DCG
            dcg = 0
            for i, doc in enumerate(results):
                relevance = 1 if expected.lower() in doc.lower() else 0
                dcg += relevance / math.log2(i + 2)

            # 计算理想 DCG（最理想情况）
            ideal_dcg = 1 / math.log2(2)  # 第一个位置就是相关的

            total += dcg / ideal_dcg if ideal_dcg > 0 else 0

        return total / len(test_cases) if test_cases else 0

    def faithfulness(
        self,
        test_cases: list[dict],
    ) -> float:
        """忠实度：回答是否基于检索到的文档（而非幻觉）"""
        faithful_count = 0
        for case in test_cases:
            results = self.rag.retrieve(case["query"], top_k=3)
            answer = self.rag.generate(case["query"], results)

            # 简单检查：回答中的关键实体是否出现在检索结果中
            context_text = " ".join(results).lower()
            answer_words = set(answer.lower().split())
            context_words = set(context_text.split())

            overlap = len(answer_words & context_words)
            total = len(answer_words)

            if total > 0 and overlap / total > 0.3:
                faithful_count += 1

        return faithful_count / len(test_cases) if test_cases else 0

    def full_evaluation(
        self,
        test_cases: list[dict],
    ) -> dict:
        """完整评估"""
        return {
            "recall_at_3": self.recall_at_k(test_cases, k=3),
            "recall_at_5": self.recall_at_k(test_cases, k=5),
            "mrr": self.mrr(test_cases),
            "ndcg_at_5": self.ndcg_at_k(test_cases, k=5),
            "faithfulness": self.faithfulness(test_cases),
            "test_count": len(test_cases),
        }
```

---

## 12.6 高级分块策略

### 12.6.1 语义分块

传统的分块策略按字符数或固定分隔符切分，但可能在句子中间切断。语义分块利用 Embedding 来检测语义边界。

```python
import numpy as np


def semantic_chunking(
    text: str,
    embedding_fn,
    threshold: float = 0.5,
    min_chunk_size: int = 100,
    max_chunk_size: int = 1000,
) -> list[str]:
    """
    语义分块：基于 Embedding 相似度检测语义边界

    参数:
        text: 原始文本
        embedding_fn: Embedding 函数
        threshold: 相似度阈值（低于此值则切分）
        min_chunk_size: 最小块大小
        max_chunk_size: 最大块大小

    返回:
        分块列表
    """
    # 先按句子切分
    sentences = text.replace("\n", " ").split("。")
    sentences = [s.strip() + "。" for s in sentences if s.strip()]

    if len(sentences) <= 1:
        return [text]

    # 获取每个句子的 Embedding
    embeddings = embedding_fn(sentences)

    # 计算相邻句子的相似度
    chunks = []
    current_chunk = [sentences[0]]

    for i in range(1, len(sentences)):
        sim = cosine_similarity(embeddings[i - 1], embeddings[i])
        current_text = " ".join(current_chunk)

        if sim < threshold or len(current_text) + len(sentences[i]) > max_chunk_size:
            # 语义断裂或超过最大长度，切分
            if len(current_text) >= min_chunk_size:
                chunks.append(current_text)
            current_chunk = [sentences[i]]
        else:
            current_chunk.append(sentences[i])

    if current_chunk:
        chunks.append(" ".join(current_chunk))

    return chunks if chunks else [text]
```

### 12.6.2 父子分块策略

这种策略将文档同时切分为大块和小块。检索时在小块中进行以获得精确匹配，但返回大块以提供足够的上下文。

```python
def parent_child_chunking(
    text: str,
    parent_size: int = 2000,
    child_size: int = 200,
    overlap: int = 50,
) -> list[dict]:
    """
    父子分块：小块用于检索，大块用于上下文

    返回:
        包含 parent 和 child 的分块列表
    """
    # 生成父块
    parents = []
    start = 0
    while start < len(text):
        end = start + parent_size
        parents.append({"text": text[start:end], "start": start, "end": end})
        start = end - overlap

    # 为每个父块生成子块
    all_chunks = []
    for parent_idx, parent in enumerate(parents):
        parent_text = parent["text"]
        child_start = 0
        while child_start < len(parent_text):
            child_end = child_start + child_size
            all_chunks.append({
                "child": parent_text[child_start:child_end],
                "parent": parent_text,
                "parent_index": parent_idx,
            })
            child_start = child_end - overlap

    return all_chunks
```

---

## 12.7 常见坑

---

## 12.6 常见坑

### 12.6.1 只用向量搜索

问题：漏掉精确匹配的文档。解决：使用 Hybrid Search 结合向量搜索和关键词搜索。

### 12.6.2 不进行重排序

问题：检索结果质量差，最相关的文档排在后面。解决：使用 Re-ranking 对结果进行二次排序。

### 12.6.3 不评估效果

问题：无法知道 RAG 是否工作正常，无法持续优化。解决：建立评估体系，定期评估检索质量和生成质量。

---

## 12.7 练习题

### 概念理解题

1. Hybrid Search 为什么比纯向量搜索效果好？它结合了什么优势？

2. Re-ranking 的原理是什么？为什么需要两阶段检索？

3. 查询改写有哪些方法？各适用于什么场景？

4. 如何评估 RAG 系统的效果？有哪些评估指标？

5. HyDE 的原理是什么？它适用于什么场景？

### 动手实践题

1. **Hybrid Search 实验：** 实现 Hybrid Search，对比纯向量搜索和混合搜索的效果。

2. **Re-ranking 实验：** 对同一个查询，对比有无 Re-ranking 的检索质量。

3. **评估实验：** 构建 10 个测试用例，评估 RAG 系统的效果。

### 思考题

1. RAG 的检索质量和生成质量之间有什么关系？如何平衡？

2. 如何处理 RAG 系统中的"没有相关信息"的情况？

---

## 12.8 实战任务

**任务：优化 RAG 系统的检索质量，对比不同策略的效果**

**具体要求：**

1. 实现 Hybrid Search
2. 实现 Re-ranking
3. 实现查询扩展
4. 构建评估数据集（至少 10 个测试用例）
5. 对比不同策略的效果

**评估标准：**

| 维度 | 权重 | 评分标准 |
|------|------|---------|
| Hybrid Search | 25% | 实现正确，效果提升明显 |
| Re-ranking | 25% | 实现正确，排序质量提升 |
| 查询改写 | 20% | 实现正确，召回率提升 |
| 评估体系 | 20% | 评估全面，结果可信 |
| 代码质量 | 10% | 结构清晰，可维护 |

---

## 12.9 RAG 系统的生产部署

### 12.9.1 生产环境的挑战

从原型到生产，RAG 系统面临很多新的挑战。开发环境用的 Chroma 内存数据库在生产中不够用，需要换成持久化的向量数据库。Embedding API 的调用需要考虑并发限制和错误处理。检索延迟需要满足用户对响应速度的期望。

### 12.9.2 生产架构设计

一个生产级的 RAG 系统通常包含以下组件：文档处理流水线（加载、清洗、分块、索引）、向量数据库（持久化存储和检索）、Embedding 服务（文本向量化）、检索服务（查询处理和结果返回）、生成服务（基于检索结果生成回答）、监控系统（追踪性能和质量指标）。

```python
class ProductionRAG:
    """生产级 RAG 系统"""

    def __init__(self, config):
        self.vector_store = self._init_vector_store(config)
        self.embedder = self._init_embedder(config)
        self.generator = self._init_generator(config)
        self.cache = self._init_cache(config)
        self.monitor = self._init_monitor(config)

    def query(self, question: str) -> dict:
        """生产级查询"""
        start_time = time.time()

        # 1. 缓存检查
        cached = self.cache.get(question)
        if cached:
            self.monitor.record("cache_hit", time.time() - start_time)
            return cached

        # 2. 检索
        results = self._retrieve(question)

        # 3. 生成
        answer = self._generate(question, results)

        # 4. 缓存结果
        response = {
            "answer": answer,
            "sources": [r["source"] for r in results],
            "latency_ms": (time.time() - start_time) * 1000,
        }
        self.cache.set(question, response, ttl=3600)

        # 5. 记录监控数据
        self.monitor.record("query", time.time() - start_time)

        return response
```

### 12.9.3 监控与告警

生产环境需要完善的监控来及时发现问题。

```python
class RAGMonitor:
    """RAG 系统监控"""

    def __init__(self):
        self.metrics = {
            "query_count": 0,
            "avg_latency_ms": 0,
            "cache_hit_rate": 0,
            "error_count": 0,
        }

    def record(self, metric_type: str, value: float):
        """记录监控指标"""
        if metric_type == "query":
            self.metrics["query_count"] += 1
            # 更新平均延迟
            n = self.metrics["query_count"]
            self.metrics["avg_latency_ms"] = (
                self.metrics["avg_latency_ms"] * (n - 1) + value * 1000
            ) / n
        elif metric_type == "cache_hit":
            self.metrics["cache_hit_rate"] += 1
        elif metric_type == "error":
            self.metrics["error_count"] += 1

    def get_dashboard(self) -> str:
        """获取监控面板"""
        return f"""
RAG 系统监控面板
{'='*40}
查询总数: {self.metrics['query_count']}
平均延迟: {self.metrics['avg_latency_ms']:.0f}ms
缓存命中率: {self.metrics['cache_hit_rate'] / max(self.metrics['query_count'], 1) * 100:.1f}%
错误数: {self.metrics['error_count']}
{'='*40}
"""
```

---

## 12.10 RAG 优化的实战经验

### 12.10.1 常见的优化陷阱

在优化 RAG 系统时，有几个常见的陷阱需要避免。

**过度优化分块**：把文档切得太细，每个块只有几十个字。虽然检索精度可能提高，但每个块缺少足够的上下文，LLM 难以理解。一般来说，200-500 个字符是比较合适的范围。

**忽略查询理解**：只关注检索和生成的优化，忽略了查询理解。用户的查询往往是口语化的、模糊的，直接用来检索效果不好。通过查询改写和扩展，可以显著提高检索质量。

**盲目追求 Recall**：Recall@K 高不代表系统好。如果为了提高 Recall 而返回太多结果，LLM 可能被无关信息干扰，回答质量反而下降。需要在 Recall 和 Precision 之间找到平衡。

### 12.10.2 优化的优先级

如果不知道从哪里开始优化，可以按照以下优先级进行。第一步，优化分块策略。好的分块是 RAG 质量的基础。第二步，实现 Hybrid Search。结合向量搜索和关键词搜索能显著提高检索质量。第三步，实现 Re-ranking。对检索结果进行二次排序能进一步提高精确性。第四步，优化查询理解。通过查询改写和扩展提高召回率。

### 12.10.3 A/B 测试验证优化效果

每次优化都应该通过 A/B 测试来验证效果。不要只看离线评估指标，还要看在线的实际效果。

```python
class RAGABTest:
    """RAG 优化的 A/B 测试"""

    def __init__(self, control_rag, treatment_rag):
        self.control = control_rag
        self.treatment = treatment_rag
        self.results = {"control": [], "treatment": []}

    def run_test(self, test_cases: list[dict], split: float = 0.5):
        """运行 A/B 测试"""
        import random
        for case in test_cases:
            if random.random() < split:
                answer = self.control.query(case["query"])
                self.results["control"].append({
                    "query": case["query"],
                    "answer": answer,
                    "expected": case.get("expected", ""),
                })
            else:
                answer = self.treatment.query(case["query"])
                self.results["treatment"].append({
                    "query": case["query"],
                    "answer": answer,
                    "expected": case.get("expected", ""),
                })

    def compare(self) -> dict:
        """对比两组结果"""
        def accuracy(results):
            correct = sum(
                1 for r in results
                if r["expected"].lower() in r["answer"].lower()
            )
            return correct / len(results) if results else 0

        return {
            "control_accuracy": accuracy(self.results["control"]),
            "treatment_accuracy": accuracy(self.results["treatment"]),
            "control_count": len(self.results["control"]),
            "treatment_count": len(self.results["treatment"]),
        }
```

---

## 12.11 本章小结

### 核心知识点回顾

- **Hybrid Search：** 结合向量搜索（语义匹配）和关键词搜索（精确匹配），提高检索质量。

- **Re-ranking：** 两阶段检索——第一阶段快速筛选，第二阶段精确排序。平衡速度和精度。

- **查询改写：** 通过查询扩展和 HyDE 提高检索召回率。

- **RAG 评估：** 从检索质量和生成质量两个维度评估，使用 Recall@K、MRR、Faithfulness 等指标。

### 关键公式

```
检索质量 = 向量搜索（语义） + 关键词搜索（精确） + 重排序（精度）
```

### 下一章预告

下一章我们将探讨 **Memory 系统** —— Agent 的记忆机制。你将学习如何让 Agent 拥有"长期记忆"，记住之前对话的关键信息。

---

*上一章：[第 11 章：RAG 基础](../chapter-11/README.md)*
*下一章：[第 13 章：Memory 系统](../chapter-13/README.md)*

RAG 进阶优化是一个持续的过程。通过 Hybrid Search、Re-ranking、查询改写等技术，你可以显著提高检索质量。记住，优化的目标不是追求单一指标的最大化，而是在检索质量、响应速度和成本之间找到最佳平衡。

在实际项目中，建议先建立评估体系，然后基于评估结果有针对性地进行优化。没有评估的优化是盲目的优化。同时，保持对 RAG 领域最新研究的关注，新的技术和方法不断涌现，持续学习是保持竞争力的关键。RAG 进阶优化是一个需要耐心和实践的过程，但每一次优化都能让你的系统更接近生产级的质量。希望本章的知识能帮助你构建出高质量的 RAG 系统。在下一章中，我们将探讨 Agent 的 Memory 系统，让你的 Agent 拥有长期记忆能力。RAG 技术是 Agent 开发中最重要的技术之一，值得你投入时间和精力深入学习。持续实践，持续优化，你会在 RAG 领域找到属于自己的技术优势。

---

*上一章：[第 11 章：RAG 基础](../chapter-11/README.md)*
*下一章：[第 13 章：Memory 系统](../chapter-13/README.md)*

> **进阶提示：** 混合搜索和重排序是生产级 RAG 的标配。如果你的 RAG 系统检索质量不理想，首先考虑这两种优化。
