# 第51章：Agentic RAG —— Agent 驱动的检索增强

## 学习目标

通过本章的学习，你将能够：
1. 理解 Agentic RAG 的核心理念和工作原理
2. 掌握 Agent 驱动的检索策略和优化方法
3. 学会构建支持多步推理的 RAG 系统
4. 理解查询规划和结果验证的实现方式
5. 掌握 Agentic RAG 与传统 RAG 的区别和优势
6. 能够构建完整的 Agentic RAG 应用

## 核心问题

在前面的章节中，我们学习了基础的 RAG（检索增强生成）技术。传统 RAG 的工作流程是固定的：用户提问 → 检索相关文档 → 将文档和问题一起发送给 LLM → 生成回答。

但这种固定流程有一个问题：它无法处理复杂的查询。比如用户问"比较 A 公司和 B 公司去年的营收增长率"，传统 RAG 可能只能检索到其中一家公司的信息，或者检索到的信息不够全面。

Agentic RAG 正是为了解决这个问题而设计的。它将 Agent 的自主决策能力引入 RAG 系统，让系统能够：

1. 分析查询的复杂性
2. 制定多步检索策略
3. 动态调整检索方向
4. 验证和整合检索结果

这就像是从"自动售货机"升级到了"智能导购员"——后者能够理解你的需求，主动寻找最合适的信息，甚至在找不到时提供替代方案。

---

## 原理讲解

### 从 RAG 到 Agentic RAG

传统 RAG 的核心假设是：一次检索就能找到所有需要的信息。但现实中，很多问题需要多步检索和推理才能回答。

举个例子：用户问"特斯拉 CEO 的母校在哪里？"

传统 RAG 的做法：
1. 检索"特斯拉 CEO"
2. 找到答案"埃隆·马斯克"
3. 继续检索"埃隆·马斯克的母校"
4. 找到答案"宾夕法尼亚大学"

这个过程需要两次检索，而且第二次检索依赖于第一次的结果。传统 RAG 无法自动处理这种多步检索。

Agentic RAG 的做法：
1. 分析查询，识别出需要两步检索
2. 第一步：检索特斯拉 CEO → 得到"埃隆·马斯克"
3. 第二步：根据第一步的结果，检索"埃隆·马斯克的母校" → 得到"宾夕法尼亚大学"
4. 整合结果，生成最终回答

### Agentic RAG 的核心组件

一个完整的 Agentic RAG 系统通常包含以下组件：

**1. 查询分析器（Query Analyzer）**
分析用户查询的复杂性，识别需要多少步检索，每步需要什么信息。

**2. 检索规划器（Retrieval Planner）**
制定检索策略，包括检索顺序、并行/串行执行、备用方案等。

**3. 检索执行器（Retrieval Executor）**
执行实际的检索操作，支持多种检索方式（关键词搜索、语义搜索、知识图谱查询等）。

**4. 结果验证器（Result Validator）**
验证检索结果的质量和完整性，判断是否需要补充检索。

**5. 答案生成器（Answer Generator）**
基于检索结果生成最终的回答，包括引用来源。

### 查询规划策略

查询规划是 Agentic RAG 的核心。常见的规划策略包括：

**分解策略（Decomposition）**
将复杂查询分解为多个简单子查询。

```
原始查询："比较特斯拉和比亚迪的电池技术"
分解为：
1. "特斯拉的电池技术是什么"
2. "比亚迪的电池技术是什么"
3. "比较特斯拉和比亚迪的电池技术"
```

**链式策略（Chaining）**
将检索过程组织成链式结构，每一步的输出作为下一步的输入。

```
步骤1：检索"A" → 结果A
步骤2：基于结果A，检索"B" → 结果B
步骤3：基于结果A和B，生成回答
```

**树形策略（Tree）**
探索多个可能的检索路径，选择最佳路径。

```
             问题
            /    \
         路径1  路径2
         /        \
      结果1      结果2
         \        /
          整合结果
```

**迭代策略（Iterative）**
反复检索和精炼，直到获得足够的信息。

```
循环：
1. 检索 → 评估是否足够
2. 如果不够 → 调整查询 → 继续检索
3. 如果足够 → 生成回答
```

### 检索质量评估

Agentic RAG 需要评估检索结果的质量，以决定是否需要继续检索。评估维度包括：

**相关性（Relevance）**：检索结果是否与查询相关？
**完整性（Completeness）**：是否收集到了所有需要的信息？
**一致性（Consistency）**：不同来源的信息是否一致？
**时效性（Freshness）**：信息是否是最新的？

### 结果整合与冲突解决

当从多个来源检索到信息时，需要整合这些信息并解决可能的冲突：

**信息融合**：将多个来源的信息合并成一个连贯的回答。
**冲突检测**：识别不同来源之间的矛盾信息。
**可信度评估**：评估每个来源的可信度，选择最可信的信息。
**不确定性标注**：对于无法确定的信息，标注不确定性。

### 引用与溯源

Agentic RAG 的一个重要特性是支持引用和溯源。系统应该能够：
- 标注每条信息的来源
- 提供原始文档的链接
- 显示检索的路径和过程

---

## 完整代码示例

### 示例 1：基础 Agentic RAG 系统

```python
"""
基础 Agentic RAG 系统
实现查询分析、多步检索和结果整合
"""

import json
import os
from typing import Dict, List, Optional, Tuple
from dataclasses import dataclass, field
import anthropic


@dataclass
class Document:
    """文档"""
    doc_id: str
    content: str
    metadata: Dict = field(default_factory=dict)
    score: float = 0.0


@dataclass
class RetrievalStep:
    """检索步骤"""
    step_id: int
    query: str
    results: List[Document]
    analysis: str = ""


class SimpleVectorStore:
    """简单的向量存储（使用关键词匹配模拟）"""
    
    def __init__(self):
        self.documents: List[Document] = []
    
    def add_document(self, doc: Document):
        self.documents.append(doc)
    
    def search(self, query: str, top_k: int = 5) -> List[Document]:
        """搜索文档"""
        query_words = set(query.lower().split())
        
        for doc in self.documents:
            doc_words = set(doc.content.lower().split())
            overlap = len(query_words & doc_words)
            doc.score = overlap / len(query_words) if query_words else 0
        
        results = [doc for doc in self.documents if doc.score > 0]
        results.sort(key=lambda x: x.score, reverse=True)
        
        return results[:top_k]


class AgenticRAG:
    """Agentic RAG 系统"""
    
    def __init__(self):
        self.client = anthropic.Anthropic(
            api_key=os.environ.get("ANTHROPIC_API_KEY")
        )
        self.vector_store = SimpleVectorStore()
        self.retrieval_history: List[RetrievalStep] = []
    
    def add_documents(self, documents: List[Dict]):
        """添加文档"""
        for i, doc in enumerate(documents):
            self.vector_store.add_document(
                Document(
                    doc_id=f"doc_{i}",
                    content=doc.get("content", ""),
                    metadata=doc.get("metadata", {}),
                )
            )
        print(f"已添加 {len(documents)} 个文档")
    
    def answer(self, query: str) -> Dict:
        """回答查询"""
        print(f"\n{'='*60}")
        print(f"查询: {query}")
        print(f"{'='*60}")
        
        self.retrieval_history = []
        
        # Step 1: 分析查询
        print("\n[Step 1] 分析查询...")
        analysis = self._analyze_query(query)
        print(f"  分析结果: {analysis}")
        
        # Step 2: 制定检索计划
        print("\n[Step 2] 制定检索计划...")
        plan = self._create_retrieval_plan(query, analysis)
        print(f"  检索计划: {json.dumps(plan, ensure_ascii=False, indent=2)}")
        
        # Step 3: 执行检索
        print("\n[Step 3] 执行检索...")
        all_results = self._execute_retrieval_plan(plan)
        
        # Step 4: 验证结果
        print("\n[Step 4] 验证结果...")
        validation = self._validate_results(query, all_results)
        print(f"  验证结果: {validation['assessment']}")
        
        # 如果需要补充检索
        if validation.get("need_more"):
            print("\n  执行补充检索...")
            additional_results = self._supplement_retrieval(
                query, validation.get("missing_info", [])
            )
            all_results.extend(additional_results)
        
        # Step 5: 生成回答
        print("\n[Step 5] 生成回答...")
        answer = self._generate_answer(query, all_results)
        
        return {
            "query": query,
            "answer": answer,
            "retrieval_steps": len(self.retrieval_history),
            "documents_used": len(all_results),
        }
    
    def _analyze_query(self, query: str) -> Dict:
        """分析查询"""
        prompt = f"""请分析以下查询的复杂性和检索需求：

查询：{query}

请用 JSON 格式回答：
{{
    "complexity": "simple/medium/complex",
    "requires_multi_step": true/false,
    "key_entities": ["实体列表"],
    "information_needed": ["需要的信息"],
    "estimated_steps": 1-5
}}
"""
        
        response = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            messages=[{"role": "user", "content": prompt}],
        )
        
        try:
            text = response.content[0].text
            start = text.find('{')
            end = text.rfind('}') + 1
            if start >= 0 and end > start:
                return json.loads(text[start:end])
        except:
            pass
        
        return {
            "complexity": "simple",
            "requires_multi_step": False,
            "key_entities": [],
            "information_needed": [query],
            "estimated_steps": 1,
        }
    
    def _create_retrieval_plan(self, query: str, analysis: Dict) -> List[Dict]:
        """创建检索计划"""
        if not analysis.get("requires_multi_step", False):
            return [{"step": 1, "query": query, "depends_on": []}]
        
        # 使用 LLM 生成检索计划
        prompt = f"""基于以下查询分析，生成一个多步检索计划。

查询：{query}
分析：{json.dumps(analysis, ensure_ascii=False)}

请生成一个检索计划，每个步骤包含：
- step: 步骤编号
- query: 检索查询
- depends_on: 依赖的步骤列表

用 JSON 数组格式回答。"""
        
        response = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            messages=[{"role": "user", "content": prompt}],
        )
        
        try:
            text = response.content[0].text
            start = text.find('[')
            end = text.rfind(']') + 1
            if start >= 0 and end > start:
                return json.loads(text[start:end])
        except:
            pass
        
        return [{"step": 1, "query": query, "depends_on": []}]
    
    def _execute_retrieval_plan(self, plan: List[Dict]) -> List[Document]:
        """执行检索计划"""
        all_results = []
        step_results = {}
        
        for step in plan:
            step_num = step.get("step", 1)
            query = step.get("query", "")
            depends_on = step.get("depends_on", [])
            
            # 替换依赖步骤的结果
            for dep in depends_on:
                if dep in step_results:
                    query = query.replace(f"{{step_{dep}}}", step_results[dep])
            
            print(f"  Step {step_num}: 检索 '{query}'")
            
            results = self.vector_store.search(query, top_k=3)
            
            retrieval_step = RetrievalStep(
                step_id=step_num,
                query=query,
                results=results,
                analysis=f"找到 {len(results)} 个相关文档",
            )
            self.retrieval_history.append(retrieval_step)
            
            if results:
                step_results[step_num] = results[0].content[:50]
                all_results.extend(results)
                print(f"    找到 {len(results)} 个相关文档")
            else:
                print(f"    未找到相关文档")
        
        return all_results
    
    def _validate_results(self, query: str, results: List[Document]) -> Dict:
        """验证检索结果"""
        if not results:
            return {
                "assessment": "没有找到相关结果",
                "need_more": True,
                "missing_info": ["所有相关信息"],
            }
        
        prompt = f"""评估以下检索结果是否足够回答查询。

查询：{query}
检索到的文档数量：{len(results)}
文档摘要：{[r.content[:100] for r in results[:3]]}

请评估：
1. 结果是否足够回答查询？
2. 缺少什么信息？（如果有）
3. 是否需要补充检索？

用 JSON 格式回答。"""
        
        response = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=512,
            messages=[{"role": "user", "content": prompt}],
        )
        
        try:
            text = response.content[0].text
            start = text.find('{')
            end = text.rfind('}') + 1
            if start >= 0 and end > start:
                return json.loads(text[start:end])
        except:
            pass
        
        return {"assessment": "结果足够", "need_more": False}
    
    def _supplement_retrieval(self, query: str, missing_info: List[str]) -> List[Document]:
        """补充检索"""
        additional_results = []
        
        for info in missing_info:
            results = self.vector_store.search(info, top_k=2)
            additional_results.extend(results)
        
        return additional_results
    
    def _generate_answer(self, query: str, results: List[Document]) -> str:
        """生成回答"""
        context = "\n\n".join([
            f"[文档 {i+1}] {doc.content}"
            for i, doc in enumerate(results[:5])
        ])
        
        prompt = f"""基于以下检索到的文档回答用户的问题。

用户问题：{query}

检索到的文档：
{context}

请生成一个准确、完整的回答。如果文档中没有足够的信息，请说明。"""
        
        response = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=2048,
            messages=[{"role": "user", "content": prompt}],
        )
        
        return response.content[0].text


def demo_agentic_rag():
    """演示 Agentic RAG"""
    print("=" * 60)
    print("Agentic RAG 演示")
    print("=" * 60)
    
    rag = AgenticRAG()
    
    # 添加示例文档
    documents = [
        {"content": "特斯拉是一家美国电动汽车公司，由埃隆·马斯克于2003年创立。", "metadata": {"source": "公司介绍"}},
        {"content": "比亚迪是中国最大的电动汽车制造商，由王传福于1995年创立。", "metadata": {"source": "公司介绍"}},
        {"content": "埃隆·马斯克毕业于宾夕法尼亚大学，获得经济学和物理学双学位。", "metadata": {"source": "人物介绍"}},
        {"content": "特斯拉的电池技术主要采用圆柱形锂电池，与松下合作生产。", "metadata": {"source": "技术文档"}},
        {"content": "比亚迪的刀片电池采用磷酸铁锂技术，具有更高的安全性。", "metadata": {"source": "技术文档"}},
        {"content": "2023年，特斯拉全球销量约180万辆，比亚迪销量约300万辆。", "metadata": {"source": "销售数据"}},
    ]
    
    rag.add_documents(documents)
    
    # 测试查询
    queries = [
        "特斯拉的CEO是谁？他的母校在哪里？",
        "比较特斯拉和比亚迪的电池技术",
    ]
    
    for query in queries:
        result = rag.answer(query)
        print(f"\n最终回答:\n{result['answer']}")
        print(f"检索步骤: {result['retrieval_steps']}, 使用文档: {result['documents_used']}")


if __name__ == "__main__":
    demo_agentic_rag()
```

### 示例 2：带反思的 Agentic RAG

```python
"""
带反思机制的 Agentic RAG
自动评估检索质量并优化策略
"""

import json
from typing import Dict, List
from dataclasses import dataclass, field
import anthropic


@dataclass
class RetrievalReflection:
    """检索反思"""
    query: str
    strategy_used: str
    results_count: int
    quality_score: float
    issues_found: List[str] = field(default_factory=list)
    improvement_suggestions: List[str] = field(default_factory=list)


class ReflectiveAgenticRAG:
    """带反思的 Agentic RAG"""
    
    def __init__(self):
        self.client = anthropic.Anthropic(
            api_key=os.environ.get("ANTHROPIC_API_KEY")
        )
        self.reflection_history: List[RetrievalReflection] = []
        self.strategy_performance: Dict[str, List[float]] = {}
    
    def answer_with_reflection(self, query: str, context: str = "") -> Dict:
        """带反思的回答"""
        print(f"\n查询: {query}")
        
        # Step 1: 选择策略
        strategy = self._select_strategy(query)
        print(f"选择策略: {strategy}")
        
        # Step 2: 执行检索
        results = self._execute_strategy(query, strategy, context)
        print(f"检索结果: {len(results)} 条")
        
        # Step 3: 反思
        reflection = self._reflect(query, strategy, results)
        self.reflection_history.append(reflection)
        
        # 更新策略性能
        if strategy not in self.strategy_performance:
            self.strategy_performance[strategy] = []
        self.strategy_performance[strategy].append(reflection.quality_score)
        
        print(f"质量评分: {reflection.quality_score:.2f}")
        if reflection.issues_found:
            print(f"发现问题: {reflection.issues_found}")
        
        # Step 4: 如果质量不够，尝试其他策略
        if reflection.quality_score < 0.6:
            print("质量不足，尝试其他策略...")
            alternative_strategy = self._get_alternative_strategy(strategy)
            alternative_results = self._execute_strategy(
                query, alternative_strategy, context
            )
            if len(alternative_results) > len(results):
                results = alternative_results
                print(f"使用替代策略，找到 {len(results)} 条结果")
        
        # Step 5: 生成回答
        answer = self._generate_answer(query, results)
        
        return {
            "query": query,
            "answer": answer,
            "strategy": strategy,
            "quality_score": reflection.quality_score,
        }
    
    def _select_strategy(self, query: str) -> str:
        """选择检索策略"""
        # 基于历史性能选择策略
        if self.strategy_performance:
            avg_scores = {
                strategy: sum(scores) / len(scores)
                for strategy, scores in self.strategy_performance.items()
            }
            best_strategy = max(avg_scores, key=avg_scores.get)
            if avg_scores[best_strategy] > 0.7:
                return best_strategy
        
        # 默认策略
        if "?" in query:
            return "question_answering"
        elif "比较" in query or "对比" in query:
            return "comparison"
        elif "如何" in query or "怎么" in query:
            return "how_to"
        else:
            return "direct"
    
    def _execute_strategy(self, query: str, strategy: str, context: str) -> List[Dict]:
        """执行检索策略"""
        # 简化实现
        results = [
            {"content": f"基于 {strategy} 策略检索到的结果: {query}", "score": 0.8}
        ]
        return results
    
    def _reflect(self, query: str, strategy: str, results: List[Dict]) -> RetrievalReflection:
        """反思检索结果"""
        quality_score = min(1.0, len(results) * 0.3)
        issues = []
        
        if len(results) == 0:
            issues.append("没有检索到结果")
            quality_score = 0.0
        elif len(results) < 3:
            issues.append("检索结果较少")
        
        return RetrievalReflection(
            query=query,
            strategy_used=strategy,
            results_count=len(results),
            quality_score=quality_score,
            issues_found=issues,
        )
    
    def _get_alternative_strategy(self, current: str) -> str:
        """获取替代策略"""
        strategies = ["direct", "question_answering", "comparison", "how_to"]
        alternatives = [s for s in strategies if s != current]
        return alternatives[0] if alternatives else "direct"
    
    def _generate_answer(self, query: str, results: List[Dict]) -> str:
        """生成回答"""
        context = "\n".join([r.get("content", "") for r in results])
        return f"基于检索结果，回答：{query}\n\n{context}"
    
    def get_strategy_report(self) -> Dict:
        """获取策略报告"""
        report = {}
        for strategy, scores in self.strategy_performance.items():
            report[strategy] = {
                "usage_count": len(scores),
                "avg_score": sum(scores) / len(scores),
            }
        return report


def demo_reflective_rag():
    """演示带反思的 RAG"""
    print("=" * 60)
    print("带反思的 Agentic RAG")
    print("=" * 60)
    
    rag = ReflectiveAgenticRAG()
    
    queries = [
        "什么是机器学习？",
        "比较深度学习和传统机器学习",
        "如何训练一个神经网络？",
    ]
    
    for query in queries:
        rag.answer_with_reflection(query)
    
    print("\n策略报告:")
    report = rag.get_strategy_report()
    print(json.dumps(report, indent=2, ensure_ascii=False))


if __name__ == "__main__":
    demo_reflective_rag()
```

---

## 案例分析

### 案例 1：学术文献检索系统

**场景**：研究人员需要从大量学术文献中找到相关研究，并进行综合分析。

**Agentic RAG 实现**：

1. **查询理解**：识别研究领域、关键词、时间范围
2. **多维检索**：同时搜索标题、摘要、全文、引用关系
3. **结果评估**：评估文献的相关性和权威性
4. **综合分析**：整合多篇文献的信息，生成综述
5. **引用管理**：自动生成参考文献列表

**关键优化**：
- 使用学科分类树进行精确检索
- 考虑文献的引用次数和影响因子
- 识别文献之间的关联和矛盾

### 案例 2：企业知识问答系统

**场景**：企业员工需要从内部文档中快速找到答案。

**Agentic RAG 实现**：

1. **意图识别**：判断是查定义、查流程、还是查数据
2. **权限检查**：验证用户有权访问相关文档
3. **多源检索**：同时搜索 Wiki、邮件、会议记录、代码库
4. **答案验证**：检查答案的准确性和时效性
5. **反馈学习**：根据用户反馈优化检索策略

**隐私保护**：
- 文档级别的访问控制
- 敏感信息脱敏
- 审计日志

---

## 常见坑

### 坑 1：过度检索

检索太多不相关的内容，反而影响回答质量。

**解决方案**：设置检索结果数量上限，使用相关性阈值过滤。

### 坑 2：循环检索

Agent 陷入检索-评估-再检索的循环。

**解决方案**：设置最大检索轮次，使用退化策略。

### 坑 3：信息冲突

不同来源的信息相互矛盾。

**解决方案**：实现冲突检测和可信度评估。

### 坑 4：延迟过高

多步检索导致响应时间过长。

**解决方案**：使用并行检索，设置超时限制。

### 坑 5：成本控制

大量 LLM 调用导致成本过高。

**解决方案**：使用缓存，优化检索策略。

---

## 练习题

### 练习 1：基础 Agentic RAG
实现一个基础的 Agentic RAG 系统，支持：
- 查询分析
- 多步检索
- 结果整合

### 练习 2：反思机制
为 Agentic RAG 添加反思功能：
- 评估检索质量
- 优化检索策略
- 学习最佳实践

### 练习 3：多源检索
实现支持多个数据源的检索：
- 文档数据库
- 知识图谱
- API 接口

### 练习 4：答案验证
实现答案验证机制：
- 事实核查
- 来源验证
- 一致性检查

### 练习 5：性能优化
优化 Agentic RAG 的性能：
- 实现缓存
- 并行检索
- 结果预过滤

### 练习 6：完整应用
构建一个完整的 Agentic RAG 应用：
- 用户界面
- 检索可视化
- 反馈收集

---

## 实战任务

### 任务 1：构建学术文献检索系统（中等难度）

**功能需求：**
1. 支持多关键词检索
2. 支持时间范围筛选
3. 支持引用关系查询
4. 生成文献综述

**技术要求：**
1. 使用 Agentic RAG 架构
2. 实现查询规划
3. 支持结果排序
4. 生成引用列表

### 任务 2：构建企业知识问答系统（高难度）

**功能需求：**
1. 支持多种文档格式
2. 实现权限控制
3. 支持多轮对话
4. 生成答案引用

**技术要求：**
1. 使用向量数据库
2. 实现文档索引
3. 支持增量更新
4. 实现审计日志

---

## 本章小结

本章我们深入学习了 Agentic RAG，这是 RAG 技术的重要演进方向。我们从以下几个方面进行了探索：

**核心理念**：Agentic RAG 将 Agent 的自主决策能力引入 RAG 系统，让系统能够处理复杂的多步检索任务。

**核心组件**：理解了查询分析器、检索规划器、检索执行器、结果验证器和答案生成器五个核心组件。

**查询规划**：掌握了分解策略、链式策略、树形策略和迭代策略四种查询规划方法。

**结果验证**：学会了评估检索结果的相关性、完整性、一致性和时效性。

**结果整合**：理解了信息融合、冲突检测、可信度评估和不确定性标注等结果整合技术。

**引用溯源**：掌握了信息来源标注和检索路径追踪的实现方式。

Agentic RAG 代表了 RAG 技术的未来发展方向。通过引入 Agent 的能力，RAG 系统从被动的检索工具变成了主动的知识探索者。

在下一章中，我们将学习 Agent 与数据库的结合，看看如何让 Agent 直接查询和分析数据。

---

## 延伸阅读

1. Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks (原始论文)
2. Self-RAG: Learning to Retrieve, Generate, and Critique (论文)
3. CRAG: Corrective Retrieval Augmented Generation (论文)
4. Adaptive RAG (研究方向)
5. GraphRAG: Graph-based Retrieval Augmented Generation (研究方向)
