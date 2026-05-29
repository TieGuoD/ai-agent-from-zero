# 第 14 章：Memory 进阶 —— 向量记忆与语义搜索

---

## 学习目标

完成本章学习后，你将能够：

1. 理解为什么关键词搜索在 Agent 记忆场景中力不从心，以及语义搜索如何弥补这一不足
2. 深入理解 Embedding 的原理——文本是如何变成一串数字的，这些数字又如何表达语义
3. 从零实现一个基于向量的语义记忆检索系统，包含存储、检索、排序的完整流程
4. 掌握 ChromaDB 等向量数据库在 Agent 记忆中的实战用法
5. 理解混合检索策略的设计思路——为什么语义搜索加关键词搜索往往比单独使用任何一种都更好
6. 了解记忆分块、向量归一化等工程最佳实践，避免常见的"坑"

## 核心问题

1. 为什么关键词搜索不够用？语义搜索解决了什么问题？
2. 如何将记忆转换为向量并进行相似度检索？
3. 向量数据库在 Agent 记忆中扮演什么角色？
4. 混合检索为什么比单一检索更可靠？

---

## 14.1 从关键词搜索到语义搜索

### 14.1.1 关键词搜索的局限性

让我们先从一个真实的场景说起。假设你有一个 Agent，它记住了用户之前说过的一句话："我上次去北京玩的时候特别喜欢故宫"。现在用户问 Agent："你还记得我之前提到的旅行经历吗？" 如果你用关键词搜索来查找这条记忆，问题就来了——用户的提问里没有"北京"，没有"故宫"，甚至没有"旅行"这个词的精确匹配。关键词搜索很可能返回零结果，或者返回一些碰巧包含"旅行"但跟用户经历毫无关系的记忆。

这就是关键词搜索的根本局限：它只看字面是否匹配，不理解语义。再举一个更直观的例子：

```
用户搜索："如何让程序运行更快"

关键词搜索可能返回：
  - "Python 程序性能优化"（包含"程序"这个词，相关）
  - "代码运行速度提升"（语义高度相关，但关键词不匹配）
  - "性能调优指南"（语义非常相关，但没有一个关键词重合）
  - "减少程序延迟"（语义相关，但关键词不匹配）
  - "Java 程序设计入门"（包含"程序"这个词，但完全不相关）
```

你会发现，关键词搜索既会漏掉真正相关的结果（因为它只认字面），又会返回不相关的结果（因为它只看字面）。在 Agent 的记忆检索场景中，这个问题尤其严重——用户不会用完全相同的措辞来描述同一件事。

### 14.1.2 语义搜索的思路

语义搜索的核心想法其实很朴素：如果我们能把文本的"含义"表示成一种可以计算的形式，那就能比较两段文本在含义上的相似度，而不只是看字面是否重合。

这就引出了 Embedding 的概念。

### 14.1.3 Embedding 是什么

Embedding（嵌入）是将一段文本映射到一个高维向量空间的过程。你可以把它想象成给每段文本画一个"语义指纹"——这个指纹不是文字，而是一串数字（比如 1536 个浮点数），但它的精妙之处在于：语义相似的文本，它们的"指纹"在向量空间中的距离就更近。

用一个生活中的比喻来理解：假如你要给每本书编一个分类号，你可能会用"文学/科幻/赛博朋克"这样的层级分类。Embedding 做的事情类似，只不过它不是用人类定义的分类标签，而是用数学方法自动学习出一种高维的"语义坐标"。在这个坐标系中，"我喜欢吃苹果"和"苹果很好吃"的坐标很接近，而"今天天气不错"的坐标则远离它们。

让我们通过代码来感受一下这个过程：

```python
import numpy as np

# 模拟 Embedding 的效果（真实场景中使用 Embedding 模型）
# 这里我们用简化的例子来展示核心概念

def simple_demo():
    """演示 Embedding 的核心思想"""
    
    # 假设我们有一个简化的 3 维 Embedding 空间
    # 实际中通常是 768 维或 1536 维
    
    embeddings = {
        "我喜欢吃苹果": np.array([0.8, 0.6, 0.1]),
        "苹果很好吃":   np.array([0.78, 0.58, 0.12]),   # 和上面很接近
        "我爱吃水果":   np.array([0.75, 0.55, 0.15]),   # 也接近
        "今天天气真好":  np.array([0.1, 0.2, 0.9]),     # 差很远
        "程序运行太慢了": np.array([0.05, 0.05, 0.85]),  # 和天气有点接近
    }
    
    def cosine_similarity(a, b):
        """计算余弦相似度"""
        return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
    
    # 查询："我想吃水果"
    query = np.array([0.76, 0.56, 0.14])
    
    print("查询：我想吃水果\n")
    print("各记忆的相似度：")
    for text, emb in embeddings.items():
        sim = cosine_similarity(query, emb)
        print(f"  {text}: {sim:.4f}")
    
    # 结果会显示语义相似的文本分数更高

simple_demo()
```

这段代码展示了一个关键事实：在 Embedding 空间中，"我喜欢吃苹果"、"苹果很好吃"、"我爱吃水果"这三个短语的距离很近，而"今天天气真好"和"程序运行太慢了"则远离它们。这就是语义搜索的基础——通过计算向量之间的距离来衡量语义的相似度。

### 14.1.4 余弦相似度的直觉理解

你可能会问：为什么用余弦相似度，而不是直接计算两个向量之间的欧几里得距离（就是直线距离）？

原因是这样的：在高维空间中，向量的长度（模）往往受到很多因素的影响，比如文本的长度、用词的频率等。但我们在意的是方向——也就是文本"指向"语义空间中的哪个区域。余弦相似度恰好衡量的是两个向量之间的夹角，完全忽略长度，只看方向。

打个比方：如果向量是一支箭，那余弦相似度看的是"两支箭指向是否相同"，而不是"两支箭是否一样长"。对于语义搜索来说，这正是我们想要的。

```python
import numpy as np

def cosine_similarity(vec1, vec2):
    """
    计算两个向量的余弦相似度
    
    余弦相似度的取值范围是 [-1, 1]
    - 1 表示完全相同的方向
    - 0 表示垂直（不相关）
    - -1 表示完全相反的方向
    """
    vec1 = np.array(vec1, dtype=float)
    vec2 = np.array(vec2, dtype=float)
    
    dot_product = np.dot(vec1, vec2)
    norm1 = np.linalg.norm(vec1)
    norm2 = np.linalg.norm(vec2)
    
    if norm1 == 0 or norm2 == 0:
        return 0.0
    
    return float(dot_product / (norm1 * norm2))


# 示例：验证余弦相似度对长度不敏感
vec_a = np.array([1.0, 2.0, 3.0])
vec_b = np.array([10.0, 20.0, 30.0])  # vec_a 的 10 倍
vec_c = np.array([3.0, 2.0, 1.0])     # 和 vec_a 方向不同

print(f"vec_a vs vec_b (方向相同，长度不同): {cosine_similarity(vec_a, vec_b):.4f}")  # 1.0
print(f"vec_a vs vec_c (方向不同): {cosine_similarity(vec_a, vec_c):.4f}")            # 较低
```

---

## 14.2 向量记忆的实现

### 14.2.1 记忆条目的数据结构

在实现向量记忆之前，我们需要先设计记忆条目的数据结构。每条记忆不仅包含文本内容和向量表示，还需要一些元数据来辅助检索和管理。

```python
import time
import uuid
from dataclasses import dataclass, field

@dataclass
class MemoryItem:
    """
    向量记忆条目
    
    每条记忆都包含：
    - content: 原始文本内容
    - embedding: 内容的向量表示
    - metadata: 元数据（如来源、时间、类别等）
    - importance: 重要性评分（0-1），影响检索排序
    - access_count: 被检索到的次数，用于热度统计
    - timestamp: 创建时间
    """
    id: str = field(default_factory=lambda: str(uuid.uuid4())[:8])
    content: str = ""
    embedding: list[float] = field(default_factory=list)
    metadata: dict = field(default_factory=dict)
    importance: float = 0.5
    access_count: int = 0
    created_at: float = field(default_factory=time.time)
    last_accessed: float = field(default_factory=time.time)
    
    def to_dict(self) -> dict:
        """转为字典，便于序列化存储"""
        return {
            "id": self.id,
            "content": self.content,
            "metadata": self.metadata,
            "importance": self.importance,
            "access_count": self.access_count,
            "created_at": self.created_at,
            "last_accessed": self.last_accessed,
        }
```

### 14.2.2 向量记忆存储的核心实现

现在让我们来实现核心的向量记忆存储类。这是整个语义搜索系统的"心脏"，它负责将记忆存储为向量，并在检索时计算相似度。

```python
import numpy as np
import time


class VectorMemory:
    """
    基于向量的记忆存储
    
    工作流程：
    1. 存储时：文本 → Embedding 模型 → 向量 → 保存到内存
    2. 检索时：查询文本 → Embedding 模型 → 向量 → 计算相似度 → 返回最相关的结果
    """
    
    def __init__(self, embedding_client):
        """
        初始化向量记忆存储
        
        参数:
            embedding_client: Embedding 客户端，需要有 embed(text) 方法，
                             返回 list[float] 类型的向量
        """
        self.embedding_client = embedding_client
        self.memories: list[MemoryItem] = []
        self.index: dict[str, int] = {}  # id -> 在 memories 列表中的索引
    
    def add(self, content: str, metadata: dict = None, importance: float = 0.5) -> str:
        """
        添加一条记忆
        
        参数:
            content: 记忆的文本内容
            metadata: 元数据，如 {"source": "chat", "topic": "travel"}
            importance: 重要性评分 (0-1)
        
        返回:
            memory_id: 新记忆的唯一标识
        """
        # 第一步：调用 Embedding 模型将文本转为向量
        embedding = self.embedding_client.embed(content)
        
        # 第二步：创建记忆条目
        item = MemoryItem(
            content=content,
            embedding=embedding,
            metadata=metadata or {},
            importance=importance,
        )
        
        # 第三步：保存到内存中
        self.index[item.id] = len(self.memories)
        self.memories.append(item)
        
        return item.id
    
    def add_batch(self, contents: list[str], metadatas: list[dict] = None,
                  importances: list[float] = None) -> list[str]:
        """
        批量添加记忆
        
        批量操作比逐条添加更高效，因为可以减少 Embedding API 的调用次数
        """
        metadatas = metadatas or [{}] * len(contents)
        importances = importances or [0.5] * len(contents)
        
        # 批量生成 Embedding（一次 API 调用）
        embeddings = self.embedding_client.embed_batch(contents)
        
        ids = []
        for content, embedding, metadata, importance in zip(
            contents, embeddings, metadatas, importances
        ):
            item = MemoryItem(
                content=content,
                embedding=embedding,
                metadata=metadata or {},
                importance=importance,
            )
            self.index[item.id] = len(self.memories)
            self.memories.append(item)
            ids.append(item.id)
        
        return ids
    
    def search(self, query: str, top_k: int = 5,
               threshold: float = 0.5, metadata_filter: dict = None) -> list[dict]:
        """
        语义搜索：找到与查询最相关的记忆
        
        参数:
            query: 查询文本
            top_k: 返回最多几条结果
            threshold: 最低相似度阈值，低于这个分数的结果会被过滤掉
            metadata_filter: 元数据过滤条件，如 {"topic": "travel"}
        
        返回:
            包含 id、content、similarity、metadata 等字段的结果列表
        """
        if not self.memories:
            return []
        
        # 生成查询的向量
        query_embedding = self.embedding_client.embed(query)
        
        # 计算每条记忆与查询的相似度
        scored_memories = []
        for item in self.memories:
            # 应用元数据过滤
            if metadata_filter:
                match = all(
                    item.metadata.get(k) == v
                    for k, v in metadata_filter.items()
                )
                if not match:
                    continue
            
            # 计算余弦相似度
            sim = self._cosine_similarity(query_embedding, item.embedding)
            scored_memories.append((sim, item))
        
        # 按相似度排序
        scored_memories.sort(key=lambda x: x[0], reverse=True)
        
        # 过滤低分结果，取 top_k
        results = []
        for sim, item in scored_memories[:top_k]:
            if sim < threshold:
                break
            
            # 更新访问计数和时间
            item.access_count += 1
            item.last_accessed = time.time()
            
            results.append({
                "id": item.id,
                "content": item.content,
                "similarity": round(sim, 4),
                "metadata": item.metadata,
                "importance": item.importance,
            })
        
        return results
    
    def delete(self, memory_id: str) -> bool:
        """删除指定记忆"""
        if memory_id not in self.index:
            return False
        
        idx = self.index[memory_id]
        self.memories.pop(idx)
        
        # 重建索引（因为删除后索引会变化）
        self.index = {item.id: i for i, item in enumerate(self.memories)}
        return True
    
    def get_stats(self) -> dict:
        """获取存储统计信息"""
        return {
            "total_memories": len(self.memories),
            "avg_importance": (
                sum(m.importance for m in self.memories) / len(self.memories)
                if self.memories else 0
            ),
            "total_accesses": sum(m.access_count for m in self.memories),
        }
    
    def _cosine_similarity(self, vec1: list[float], vec2: list[float]) -> float:
        """计算余弦相似度"""
        v1 = np.array(vec1, dtype=float)
        v2 = np.array(vec2, dtype=float)
        
        norm1 = np.linalg.norm(v1)
        norm2 = np.linalg.norm(v2)
        
        if norm1 == 0 or norm2 == 0:
            return 0.0
        
        return float(np.dot(v1, v2) / (norm1 * norm2))
```

### 14.2.3 Embedding 客户端的实现

要让上面的 VectorMemory 工作起来，我们需要一个 Embedding 客户端。这里以 OpenAI 的 Embedding API 为例，展示如何封装一个可用的客户端。

```python
import os
import numpy as np


class OpenAIEmbeddingClient:
    """
    OpenAI Embedding 客户端
    
    使用 text-embedding-3-small 模型，返回 1536 维的向量。
    这个模型性价比高，适合大多数场景。
    """
    
    def __init__(self, model: str = "text-embedding-3-small"):
        import openai
        self.client = openai.OpenAI(api_key=os.environ.get("OPENAI_API_KEY"))
        self.model = model
    
    def embed(self, text: str) -> list[float]:
        """将单条文本转为向量"""
        response = self.client.embeddings.create(
            model=self.model,
            input=text,
        )
        return response.data[0].embedding
    
    def embed_batch(self, texts: list[str]) -> list[list[float]]:
        """批量将文本转为向量（效率更高，推荐使用）"""
        response = self.client.embeddings.create(
            model=self.model,
            input=texts,
        )
        return [item.embedding for item in response.data]


class LocalEmbeddingClient:
    """
    本地 Embedding 客户端（不需要 API key，适合开发和测试）
    
    使用 sentence-transformers 库在本地运行 Embedding 模型。
    """
    
    def __init__(self, model_name: str = "all-MiniLM-L6-v2"):
        from sentence_transformers import SentenceTransformer
        self.model = SentenceTransformer(model_name)
    
    def embed(self, text: str) -> list[float]:
        """将单条文本转为向量"""
        embedding = self.model.encode(text)
        return embedding.tolist()
    
    def embed_batch(self, texts: list[str]) -> list[list[float]]:
        """批量将文本转为向量"""
        embeddings = self.model.encode(texts)
        return embeddings.tolist()
```

### 14.2.4 完整的演示

把上面的组件组合起来，让我们看一个完整的例子：

```python
def demo_vector_memory():
    """演示向量记忆系统的完整使用流程"""
    
    # 1. 创建 Embedding 客户端（这里用本地模型演示）
    client = LocalEmbeddingClient()
    
    # 2. 创建向量记忆存储
    memory = VectorMemory(client)
    
    # 3. 添加一些记忆（模拟 Agent 在对话中记住的信息）
    memories_to_add = [
        ("用户叫张三，是一名 Python 工程师", {"type": "user_profile"}, 0.9),
        ("用户喜欢用 VSCode 写代码", {"type": "preference"}, 0.7),
        ("用户最近在学习 LangChain", {"type": "activity"}, 0.8),
        ("用户家在北京朝阳区", {"type": "location"}, 0.6),
        ("用户养了一只叫豆豆的猫", {"type": "hobby"}, 0.5),
        ("用户上周末去了故宫游玩", {"type": "travel"}, 0.6),
        ("用户喜欢吃川菜，特别是水煮鱼", {"type": "food_preference"}, 0.7),
        ("用户计划明年去日本旅行", {"type": "travel_plan"}, 0.8),
    ]
    
    for content, meta, imp in memories_to_add:
        memory.add(content, metadata=meta, importance=imp)
    
    print(f"已存储 {len(memory.memories)} 条记忆\n")
    
    # 4. 进行语义搜索
    queries = [
        "用户的工作是什么？",
        "用户有什么爱好？",
        "用户去过哪里旅游？",
        "用户喜欢吃什么？",
    ]
    
    for query in queries:
        print(f"查询：{query}")
        results = memory.search(query, top_k=3)
        for i, r in enumerate(results, 1):
            print(f"  {i}. [{r['similarity']:.4f}] {r['content']}")
        print()
    
    # 5. 带过滤条件的搜索
    print("查询：用户有什么旅行相关的记忆？")
    results = memory.search("旅行", top_k=3, metadata_filter={"type": "travel"})
    for i, r in enumerate(results, 1):
        print(f"  {i}. [{r['similarity']:.4f}] {r['content']}")
    
    # 6. 查看统计
    print(f"\n统计信息：{memory.get_stats()}")


# 运行演示
if __name__ == "__main__":
    demo_vector_memory()
```

---

## 14.3 向量数据库在记忆中的应用

### 14.3.1 为什么需要向量数据库

上面实现的 VectorMemory 是把所有向量都存在内存里。在记忆数量较少的时候（比如几百条、几千条），这种方式完全没问题。但当记忆数量增长到一定程度时，就会遇到两个瓶颈：

第一个瓶颈是内存。假设每条记忆的向量是 1536 维的 float32 数组，那一条向量就占 6KB 的内存。10000 条记忆就是 60MB，100 万条就是 600MB。这在很多部署场景下是不可接受的。

第二个瓶颈是搜索速度。当你有 100 万条记忆时，每次检索都要遍历所有向量计算相似度，这在 Python 里可能需要几百毫秒甚至几秒。在交互式应用中，这种延迟是用户无法忍受的。

向量数据库就是为了解决这两个问题而生的。它使用高效的索引结构（如 HNSW、IVF 等）来加速搜索，同时支持数据的持久化存储。常见的向量数据库包括 ChromaDB、FAISS、Pinecone、Weaviate 等。

### 14.3.2 ChromaDB 集成

ChromaDB 是一个轻量级的向量数据库，非常适合在 Agent 项目中使用。它支持持久化存储，可以在本地运行，不需要额外部署数据库服务。

```python
import chromadb


class ChromaMemoryStore:
    """
    基于 ChromaDB 的向量记忆存储
    
    ChromaDB 会自动处理：
    - 向量的存储和索引
    - 余弦相似度的高效计算
    - 数据的持久化
    """
    
    def __init__(self, collection_name: str = "agent_memory",
                 persist_dir: str = "./chroma_db"):
        """
        初始化 ChromaDB 记忆存储
        
        参数:
            collection_name: 集合名称，相当于数据库中的表
            persist_dir: 持久化目录，数据会保存在这个目录下
        """
        self.client = chromadb.PersistentClient(path=persist_dir)
        
        self.collection = self.client.get_or_create_collection(
            name=collection_name,
            metadata={"hnsw:space": "cosine"}  # 使用余弦相似度
        )
    
    def add_memory(self, memory_id: str, content: str, metadata: dict = None):
        """
        添加一条记忆
        
        ChromaDB 会自动为 content 生成 Embedding 向量
        （它内置了 Embedding 模型，不需要我们手动调用）
        """
        self.collection.upsert(
            ids=[memory_id],
            documents=[content],
            metadatas=[metadata] if metadata else None,
        )
    
    def add_batch(self, ids: list[str], contents: list[str],
                  metadatas: list[dict] = None):
        """批量添加记忆"""
        self.collection.upsert(
            ids=ids,
            documents=contents,
            metadatas=metadatas,
        )
    
    def search(self, query: str, n_results: int = 5,
               where: dict = None) -> list[dict]:
        """
        语义搜索
        
        参数:
            query: 查询文本
            n_results: 返回结果数量
            where: 元数据过滤条件
        
        返回:
            匹配的记忆列表
        """
        kwargs = {
            "query_texts": [query],
            "n_results": n_results,
        }
        if where:
            kwargs["where"] = where
        
        results = self.collection.query(**kwargs)
        
        # 整理结果格式
        memories = []
        if results["ids"] and results["ids"][0]:
            for i in range(len(results["ids"][0])):
                memory = {
                    "id": results["ids"][0][i],
                    "content": results["documents"][0][i],
                }
                if results["distances"] and results["distances"][0]:
                    # ChromaDB 返回的是距离（越小越相似），转换为相似度
                    memory["similarity"] = 1 - results["distances"][0][i]
                if results["metadatas"] and results["metadatas"][0]:
                    memory["metadata"] = results["metadatas"][0][i]
                memories.append(memory)
        
        return memories
    
    def delete_memory(self, memory_id: str):
        """删除记忆"""
        self.collection.delete(ids=[memory_id])
    
    def count(self) -> int:
        """获取记忆总数"""
        return self.collection.count()


# 使用示例
def demo_chromadb_memory():
    """演示 ChromaDB 记忆存储"""
    
    # 创建存储
    store = ChromaMemoryStore(
        collection_name="demo_memory",
        persist_dir="./demo_chroma"
    )
    
    # 添加记忆
    memories = [
        ("user_001", "用户叫张三，是一名 Python 工程师", {"category": "profile"}),
        ("user_002", "用户喜欢用 VSCode 写代码", {"category": "preference"}),
        ("user_003", "用户最近在学习 LangChain", {"category": "activity"}),
        ("user_004", "用户上周末去了故宫游玩", {"category": "travel"}),
    ]
    
    for mid, content, meta in memories:
        store.add_memory(mid, content, meta)
    
    print(f"存储了 {store.count()} 条记忆")
    
    # 搜索
    results = store.search("用户的爱好是什么", n_results=2)
    for r in results:
        print(f"  [{r['similarity']:.4f}] {r['content']}")


if __name__ == "__main__":
    demo_chromadb_memory()
```

### 14.3.3 FAISS 集成

FAISS（Facebook AI Similarity Search）是 Meta 开源的向量检索库，专注于高效的向量搜索。它的优势在于极快的搜索速度，特别是在大规模数据集上。

```python
import faiss
import numpy as np
import pickle
from pathlib import Path


class FAISSMemoryStore:
    """
    基于 FAISS 的向量记忆存储
    
    FAISS 的优势：
    - 搜索速度极快（百万级数据毫秒级响应）
    - 支持多种索引策略（Flat、IVF、HNSW 等）
    - 内存效率高
    
    FAISS 的限制：
    - 只存储向量，不存储原始文本（需要我们自己维护映射）
    - 不支持元数据过滤（需要在应用层实现）
    """
    
    def __init__(self, dimension: int = 1536):
        self.dimension = dimension
        # 使用 Flat 索引（精确搜索，适合小规模数据）
        # 大规模数据可以使用 IndexIVFFlat 或 IndexHNSWFlat
        self.index = faiss.IndexFlatIP(dimension)  # IP = Inner Product（内积）
        self.memories: list[dict] = []
    
    def add(self, content: str, embedding: list[float],
            metadata: dict = None):
        """添加记忆"""
        vec = np.array([embedding], dtype=np.float32)
        
        # FAISS 要求向量已经归一化（使用内积等价于余弦相似度）
        faiss.normalize_L2(vec)
        
        self.index.add(vec)
        self.memories.append({
            "content": content,
            "metadata": metadata or {},
        })
    
    def search(self, query_embedding: list[float], top_k: int = 5) -> list[dict]:
        """搜索最相似的记忆"""
        if self.index.ntotal == 0:
            return []
        
        query_vec = np.array([query_embedding], dtype=np.float32)
        faiss.normalize_L2(query_vec)
        
        # 搜索
        distances, indices = self.index.search(query_vec, min(top_k, self.index.ntotal))
        
        results = []
        for dist, idx in zip(distances[0], indices[0]):
            if idx < 0:
                continue
            results.append({
                "content": self.memories[idx]["content"],
                "similarity": float(dist),
                "metadata": self.memories[idx]["metadata"],
            })
        
        return results
    
    def save(self, path: str):
        """保存索引到文件"""
        faiss.write_index(self.index, f"{path}.faiss")
        with open(f"{path}.meta.pkl", "wb") as f:
            pickle.dump(self.memories, f)
    
    def load(self, path: str):
        """从文件加载索引"""
        self.index = faiss.read_index(f"{path}.faiss")
        with open(f"{path}.meta.pkl", "rb") as f:
            self.memories = pickle.load(f)
```

---

## 14.4 混合记忆检索策略

### 14.4.1 为什么需要混合搜索

语义搜索虽然强大，但它并不是万能的。在某些场景下，关键词搜索反而更可靠。举个例子：

假设用户问："Python 的 list comprehension 语法是什么？" 如果你只有语义搜索，它可能会返回一些"语义相关"但不精确的结果，比如"Python 数据结构概述"。但如果你有关键词搜索，它能精确匹配到包含"list comprehension"这个具体术语的记忆。

反过来，如果用户问："怎么让代码跑得更快"，关键词搜索可能找不到任何包含"性能优化"、"代码效率"这些词的记忆，但语义搜索可以。

所以，最稳妥的策略是把两种搜索结合起来：语义搜索捕捉含义上的相关性，关键词搜索捕捉精确的术语匹配，然后综合两者的得分来排序。

### 14.4.2 混合检索器的实现

```python
class HybridMemoryRetriever:
    """
    混合记忆检索器
    
    结合语义搜索和关键词搜索的优势：
    - 语义搜索：理解含义，找到语义相关的结果
    - 关键词搜索：精确匹配，找到包含具体术语的结果
    - 综合排序：加权融合两种搜索的得分
    """
    
    def __init__(self, vector_memory: VectorMemory, keyword_index: dict = None):
        """
        参数:
            vector_memory: 向量记忆存储
            keyword_index: 关键词索引，格式为 {doc_id: set_of_words}
        """
        self.vector_memory = vector_memory
        self.keyword_index = keyword_index or {}
        
        # 权重配置：语义搜索通常更可靠，所以权重更高
        self.vector_weight = 0.7
        self.keyword_weight = 0.3
    
    def retrieve(self, query: str, top_k: int = 5) -> list[dict]:
        """
        执行混合检索
        
        流程：
        1. 分别用两种方式搜索
        2. 归一化各自的得分
        3. 加权融合得分
        4. 按融合后的得分排序
        """
        # 1. 语义搜索（多取一些，后面要融合）
        vector_results = self.vector_memory.search(query, top_k=top_k * 2, threshold=0.3)
        
        # 2. 关键词搜索
        keyword_results = self._keyword_search(query, top_k=top_k * 2)
        
        # 3. 融合得分
        combined = {}
        
        # 处理语义搜索结果
        for i, result in enumerate(vector_results):
            doc_id = result["id"]
            # 使用排名归一化得分：第 i 名得分为 (top_k - i) / top_k
            rank_score = (len(vector_results) - i) / len(vector_results)
            combined[doc_id] = {
                **result,
                "vector_score": result["similarity"],
                "keyword_score": 0.0,
                "combined_score": rank_score * self.vector_weight,
            }
        
        # 处理关键词搜索结果
        for i, result in enumerate(keyword_results):
            doc_id = result["id"]
            rank_score = (len(keyword_results) - i) / len(keyword_results)
            
            if doc_id in combined:
                combined[doc_id]["keyword_score"] = result.get("score", 0)
                combined[doc_id]["combined_score"] += rank_score * self.keyword_weight
            else:
                combined[doc_id] = {
                    "id": doc_id,
                    "content": result.get("content", ""),
                    "vector_score": 0.0,
                    "keyword_score": result.get("score", 0),
                    "combined_score": rank_score * self.keyword_weight,
                    "metadata": {},
                }
        
        # 4. 按融合得分排序
        sorted_results = sorted(
            combined.values(),
            key=lambda x: x["combined_score"],
            reverse=True,
        )
        
        return sorted_results[:top_k]
    
    def _keyword_search(self, query: str, top_k: int) -> list[dict]:
        """关键词搜索"""
        query_words = set(query.lower().split())
        results = []
        
        for doc_id, words in self.keyword_index.items():
            if isinstance(words, str):
                words = set(words.lower().split())
            overlap = query_words & words
            if overlap:
                results.append({
                    "id": doc_id,
                    "score": len(overlap) / len(query_words) if query_words else 0,
                })
        
        results.sort(key=lambda x: x["score"], reverse=True)
        return results[:top_k]
    
    def update_keyword_index(self, memory_id: str, content: str):
        """更新关键词索引"""
        words = set(content.lower().split())
        self.keyword_index[memory_id] = words
```

### 14.4.3 混合搜索的效果对比

为了展示混合搜索的优势，让我们做一个简单的对比实验：

```python
def compare_search_methods():
    """对比语义搜索、关键词搜索和混合搜索的效果"""
    
    client = LocalEmbeddingClient()
    memory = VectorMemory(client)
    
    # 添加记忆
    test_memories = [
        ("Python 的列表推导式语法是 [x for x in iterable if condition]", {"topic": "python"}),
        ("性能优化可以通过减少循环次数和使用内置函数来实现", {"topic": "performance"}),
        ("Git 的分支管理是团队协作的重要工具", {"topic": "git"}),
        ("机器学习中的过拟合可以通过正则化来缓解", {"topic": "ml"}),
        ("Docker 容器化技术简化了应用的部署流程", {"topic": "devops"}),
    ]
    
    for content, meta in test_memories:
        memory.add(content, metadata=meta, importance=0.5)
    
    # 查询
    query = "Python 怎么写得更快"
    
    print(f"查询：{query}\n")
    
    # 语义搜索
    print("【语义搜索结果】")
    results = memory.search(query, top_k=3)
    for r in results:
        print(f"  [{r['similarity']:.4f}] {r['content'][:50]}...")
    
    # 混合搜索（这里简化展示，实际需要关键词索引）
    print("\n【说明】混合搜索会在语义搜索的基础上，额外考虑关键词精确匹配，")
    print("在实际应用中通常能返回更准确的结果。")


if __name__ == "__main__":
    compare_search_methods()
```

---

## 14.5 记忆的向量化最佳实践

### 14.5.1 记忆分块策略

在实际应用中，对话历史往往很长，直接把整段对话作为一个记忆条目向量化，效果并不好。原因是一段长对话可能涉及多个主题，Embedding 模型会把所有主题"混"成一个向量，导致任何一个主题的查询都无法精确匹配。

解决办法是把长对话切成较小的"块"（chunk），每块只包含一个主题相关的几条消息。这样每块的语义更集中，检索效果也更好。

```python
class MemoryChunker:
    """
    记忆分块器
    
    将长对话或长文本切分为大小合适的块，
    每块保留语义完整性，同时支持块之间的重叠以避免信息丢失。
    """
    
    def __init__(self, max_chunk_chars: int = 500, overlap_chars: int = 50):
        """
        参数:
            max_chunk_chars: 每个块的最大字符数
            overlap_chars: 相邻块之间的重叠字符数（防止关键信息被切断）
        """
        self.max_chunk_chars = max_chunk_chars
        self.overlap_chars = overlap_chars
    
    def chunk_conversation(self, messages: list[dict]) -> list[dict]:
        """
        将对话消息列表切分为块
        
        策略：
        1. 按消息逐条累加，直到达到最大长度
        2. 在切分点保留一些重叠消息，确保上下文连续
        """
        chunks = []
        current_chunk = []
        current_size = 0
        
        for msg in messages:
            msg_size = len(msg.get("content", ""))
            
            # 如果加上这条消息会超出限制，先保存当前块
            if current_size + msg_size > self.max_chunk_chars and current_chunk:
                chunks.append(self._create_chunk(current_chunk))
                
                # 保留最后几条消息作为重叠（从后往前取）
                overlap_messages = []
                overlap_size = 0
                for m in reversed(current_chunk):
                    m_size = len(m.get("content", ""))
                    if overlap_size + m_size > self.overlap_chars:
                        break
                    overlap_messages.insert(0, m)
                    overlap_size += m_size
                
                current_chunk = overlap_messages
                current_size = overlap_size
            
            current_chunk.append(msg)
            current_size += msg_size
        
        # 别忘了最后一个块
        if current_chunk:
            chunks.append(self._create_chunk(current_chunk))
        
        return chunks
    
    def _create_chunk(self, messages: list[dict]) -> dict:
        """创建一个记忆块"""
        content = "\n".join(
            f"{m.get('role', 'unknown')}: {m.get('content', '')}"
            for m in messages
        )
        return {
            "content": content,
            "message_count": len(messages),
            "timestamp": messages[-1].get("timestamp", 0),
        }
    
    def chunk_text(self, text: str, separators: list[str] = None) -> list[str]:
        """
        将长文本切分为块
        
        优先在分隔符处切分，保持语义完整性
        """
        if separators is None:
            separators = ["\n\n", "\n", "。", ".", " "]
        
        chunks = []
        remaining = text
        
        while remaining:
            if len(remaining) <= self.max_chunk_chars:
                chunks.append(remaining.strip())
                break
            
            # 尝试在分隔符处切分
            best_split = -1
            for sep in separators:
                # 从 max_chunk_chars 附近往前找最近的分隔符
                idx = remaining.rfind(sep, 0, self.max_chunk_chars)
                if idx > best_split:
                    best_split = idx + len(sep)
            
            if best_split <= 0:
                # 没找到合适的分隔符，强制在 max_chunk_chars 处切分
                best_split = self.max_chunk_chars
            
            chunk = remaining[:best_split].strip()
            if chunk:
                chunks.append(chunk)
            
            # 加上重叠
            remaining = remaining[max(0, best_split - self.overlap_chars):]
        
        return chunks
```

### 14.5.2 向量归一化的重要性

在使用余弦相似度时，有一个常见的陷阱：如果你使用的 Embedding 模型返回的向量没有归一化，而你的向量数据库使用的是内积（Inner Product）而不是余弦相似度来搜索，那搜索结果就是错误的。

归一化很简单——把每个向量除以它的模长，让它变成"单位向量"（长度为 1）。归一化后，内积就等价于余弦相似度了。

```python
def normalize_vectors(vectors: list[list[float]]) -> list[list[float]]:
    """将向量列表归一化为单位向量"""
    normalized = []
    for vec in vectors:
        v = np.array(vec, dtype=float)
        norm = np.linalg.norm(v)
        if norm > 0:
            normalized.append((v / norm).tolist())
        else:
            normalized.append(vec)
    return normalized


# 验证归一化的效果
v1 = np.array([3.0, 4.0])       # 长度 = 5
v2 = np.array([6.0, 8.0])       # 长度 = 10，方向和 v1 相同

# 归一化前的内积
print(f"归一化前的内积: {np.dot(v1, v2)}")  # 50

# 归一化后
v1_norm = v1 / np.linalg.norm(v1)  # [0.6, 0.8]
v2_norm = v2 / np.linalg.norm(v2)  # [0.6, 0.8]
print(f"归一化后的内积（=余弦相似度）: {np.dot(v1_norm, v2_norm)}")  # 1.0
```

### 14.5.3 处理 Embedding 模型的限制

大多数 Embedding 模型对输入文本有长度限制（比如 OpenAI 的模型限制 8191 tokens）。如果你的记忆文本超长，就需要截断或分段处理。

```python
class SafeEmbeddingClient:
    """
    安全的 Embedding 客户端
    
    自动处理：
    - 文本截断（超过模型限制时）
    - 错误重试（API 调用失败时）
    - 缓存（相同文本不重复调用）
    """
    
    def __init__(self, base_client, max_tokens: int = 8000):
        self.base_client = base_client
        self.max_tokens = max_tokens
        self.cache = {}
    
    def embed(self, text: str) -> list[float]:
        """安全地生成 Embedding"""
        # 检查缓存
        if text in self.cache:
            return self.cache[text]
        
        # 截断超长文本
        truncated = self._truncate(text, self.max_tokens)
        
        # 生成 Embedding
        embedding = self.base_client.embed(truncated)
        
        # 缓存结果
        self.cache[text] = embedding
        
        return embedding
    
    def embed_batch(self, texts: list[str]) -> list[list[float]]:
        """批量生成 Embedding"""
        results = []
        to_embed = []
        to_embed_indices = []
        
        for i, text in enumerate(texts):
            if text in self.cache:
                results.append(self.cache[text])
            else:
                results.append(None)
                to_embed.append(self._truncate(text, self.max_tokens))
                to_embed_indices.append(i)
        
        if to_embed:
            new_embeddings = self.base_client.embed_batch(to_embed)
            for idx, embedding in zip(to_embed_indices, new_embeddings):
                results[idx] = embedding
                self.cache[texts[idx]] = embedding
        
        return results
    
    def _truncate(self, text: str, max_tokens: int) -> str:
        """截断文本（粗略估计：1 token 约等于 2 个中文字符或 4 个英文字符）"""
        max_chars = max_tokens * 3  # 保守估计
        if len(text) <= max_chars:
            return text
        return text[:max_chars]
```

---

## 14.6 常见坑

### 14.6.1 Embedding 模型选择不当

**问题描述：** 不同的 Embedding 模型擅长不同的领域。通用模型（如 text-embedding-3-small）在日常对话场景下表现不错，但在医学、法律、金融等专业领域的效果可能很差，因为它在训练时没有见过足够的专业术语。

**解决方案：** 根据应用场景选择合适的 Embedding 模型。对于专业领域，可以考虑使用领域特化的模型，或者对通用模型进行微调。

**排查方法：** 如果你发现语义搜索的结果很不理想，首先要检查的就是 Embedding 模型是否适合你的场景。可以用一批你领域内的查询和期望结果来评估模型的效果。

### 14.6.2 向量维度不匹配

**问题描述：** 不同的 Embedding 模型返回的向量维度不同。比如 text-embedding-3-small 返回 1536 维，而 all-MiniLM-L6-v2 返回 384 维。如果你在存储时用了 A 模型，搜索时用了 B 模型，维度不匹配会导致搜索失败。

**解决方案：** 统一使用同一个 Embedding 模型来生成所有向量。在项目启动时就确定好模型，不要在中途更换。

### 14.6.3 忽略向量归一化

**问题描述：** 如果你使用余弦相似度检索，但向量没有归一化，搜索结果可能不正确。特别是使用 FAISS 的 IndexFlatIP（内积索引）时，必须确保向量已归一化，否则内积不等于余弦相似度。

**解决方案：** 在存储向量之前进行归一化，或者在 Embedding 客户端中封装归一化逻辑。

```python
# 正确做法：归一化后使用内积
vec = np.array(embedding, dtype=float)
vec_norm = vec / np.linalg.norm(vec)
# 然后将 vec_norm 存入向量数据库
```

### 14.6.4 过度依赖单一检索策略

**问题描述：** 只使用语义搜索时，可能会漏掉一些精确匹配的关键词信息。只使用关键词搜索时，又会错过语义相关但措辞不同的结果。

**解决方案：** 使用混合搜索策略，结合语义搜索和关键词搜索。在大多数场景下，混合搜索的效果都优于单一策略。

### 14.6.5 忽略记忆的时效性

**问题描述：** 记忆检索只考虑语义相似度，不考虑时间因素。用户上周说的偏好可能已经变了，但检索系统仍然返回旧的记忆。

**解决方案：** 在检索时引入时间衰减因子。较新的记忆得分更高，较旧的记忆得分被折扣。

```python
def time_decay_score(similarity: float, created_at: float,
                     decay_rate: float = 0.01) -> float:
    """加入时间衰减的综合得分"""
    age_days = (time.time() - created_at) / (24 * 3600)
    decay_factor = 1.0 / (1.0 + decay_rate * age_days)
    return similarity * decay_factor
```

---

## 14.7 练习题

### 练习 1：实现一个完整的向量记忆系统

从零实现一个支持以下功能的向量记忆系统：
- 添加记忆（单条和批量）
- 语义搜索（支持相似度阈值和 top_k）
- 元数据过滤
- 记忆删除
- 统计信息

要求使用本地 Embedding 模型（如 sentence-transformers），不依赖外部 API。

提示：参考本章的 VectorMemory 类，需要实现 `_cosine_similarity` 方法。

### 练习 2：对比纯语义搜索和混合搜索的效果

准备 20 条记忆和 10 个查询，分别用纯语义搜索和混合搜索进行检索，对比两者的准确率。

要求：
- 记忆和查询覆盖至少 3 个不同主题
- 评估指标：Top-1 命中率、Top-3 命中率
- 分析两种策略各自擅长和不擅长的场景

提示：可以构造一些需要精确术语匹配的查询（混合搜索更好）和需要语义理解的查询（语义搜索更好）。

### 练习 3：实现对话分块和向量化

实现一个完整的管道：
1. 接收一段长对话（多条消息）
2. 使用 MemoryChunker 进行分块
3. 为每个块生成 Embedding
4. 存储到向量数据库
5. 支持对块级别的检索

要求：
- 分块时考虑语义完整性
- 相邻块有适当的重叠
- 检索结果能定位到具体的对话片段

### 练习 4：实现带时间衰减的记忆检索

在 VectorMemory 的基础上添加时间衰减功能：
- 新记忆的权重更高，旧记忆的权重逐渐降低
- 提供衰减率配置参数
- 对比有无时间衰减的检索结果差异

### 练习 5：实现记忆的自动合并

当多条记忆的内容高度相似时（余弦相似度 > 0.9），自动将它们合并为一条：
- 合并后的内容保留各条记忆的关键信息
- 合并后的记忆的重要性取各条中最高的
- 定期（或在每次添加新记忆后）检查并合并

提示：可以调用 LLM 来生成合并后的内容。

---

## 14.8 实战任务

### 任务：构建语义记忆检索系统

**目标：** 构建一个支持语义检索的 Agent 记忆系统，能够跨对话记住用户信息，并在需要时精准召回。

**要求：**

1. 使用向量数据库（ChromaDB 或 FAISS）存储记忆
2. 实现基于语义相似度的检索，支持元数据过滤
3. 支持对话历史的自动分块和向量化
4. 实现混合搜索策略（语义 + 关键词）
5. 支持记忆的更新和删除
6. 提供记忆管理的 API（添加、查询、删除、统计）

**参考架构：**

```python
class SemanticMemorySystem:
    """语义记忆检索系统"""
    
    def __init__(self, llm_client, user_id: str):
        self.user_id = user_id
        
        # 向量记忆存储（使用 ChromaDB）
        self.vector_store = ChromaMemoryStore(
            collection_name=f"user_{user_id}_memory"
        )
        
        # 分块器
        self.chunker = MemoryChunker(max_chunk_chars=500)
    
    def store_conversation(self, messages: list[dict]):
        """存储对话"""
        chunks = self.chunker.chunk_conversation(messages)
        
        ids = [f"{self.user_id}_chunk_{i}" for i in range(len(chunks))]
        contents = [c["content"] for c in chunks]
        metadatas = [{"message_count": c["message_count"]} for c in chunks]
        
        self.vector_store.add_batch(ids, contents, metadatas)
    
    def recall(self, query: str, top_k: int = 5) -> list[str]:
        """召回相关记忆"""
        results = self.vector_store.search(query, n_results=top_k)
        return [r["content"] for r in results]
    
    def get_context_for_chat(self, current_message: str) -> str:
        """获取对话上下文（注入到 System Prompt）"""
        memories = self.recall(current_message, top_k=3)
        
        if not memories:
            return ""
        
        context_parts = ["以下是关于这个用户的一些记忆："]
        for mem in memories:
            context_parts.append(f"- {mem}")
        
        return "\n".join(context_parts)
```

**扩展要求（选做）：**
- 支持多用户隔离
- 实现记忆的定期清理和合并
- 添加记忆使用效果的监控和统计

---

## 14.9 本章小结

- **语义搜索**基于 Embedding 向量的相似度来衡量文本之间的语义关系，比关键词搜索更灵活、更智能。关键词搜索只看字面是否匹配，而语义搜索能理解"意思相近但措辞不同"的文本。

- **Embedding** 是将文本映射到高维向量空间的过程。在这个空间中，语义相似的文本向量距离更近。这是语义搜索的数学基础。

- **余弦相似度**是衡量两个向量相似度的常用指标，它只看方向不看长度，非常适合语义比较。使用时要注意向量归一化的问题。

- **向量数据库**（如 ChromaDB、FAISS）提供了高效的向量存储和检索能力。ChromaDB 适合中小规模、需要持久化的场景；FAISS 适合大规模、对速度要求极高的场景。

- **混合搜索**结合了语义搜索和关键词搜索的优势，在大多数场景下效果都优于单一策略。语义搜索擅长理解含义，关键词搜索擅长精确匹配，两者互补。

- **记忆分块**是提升检索质量的重要技巧。将长文本切成大小合适的块，每块语义更集中，检索更精准。切分时要注意保持语义完整性，相邻块之间保留适当的重叠。

- **向量归一化**在使用余弦相似度或内积检索时是必要的步骤，忽略归一化会导致搜索结果不正确。

- 在实际工程中，还需要考虑 **Embedding 模型选择、向量维度一致性、记忆时效性** 等问题。

