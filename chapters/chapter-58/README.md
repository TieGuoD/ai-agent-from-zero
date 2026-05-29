# 第 58 章：项目 2 -- PDF 分析 Agent

> **本章定位：** PDF 是企业中最常见的文档格式，但也是最难自动化处理的格式之一。本章将构建一个 PDF 分析 Agent，它能够读取 PDF 文档、理解其中的内容结构、提取关键信息、回答用户的提问。这个项目将让你深入理解 Agent 在文档处理场景中的应用。

---

## 学习目标

完成本章学习后，你将能够：

1. **实现 PDF 文档的智能解析** -- 能够处理文本型和扫描型 PDF，提取其中的文本、表格、图片等不同类型的内容
2. **构建基于 RAG 的文档问答系统** -- 能够让 Agent 基于 PDF 的内容回答用户的问题，并给出准确的引用来源
3. **实现多文档对比分析** -- 能够同时分析多份 PDF，找出它们之间的异同点和关联关系
4. **设计结构化信息提取管道** -- 能够从 PDF 中提取合同条款、财务数据、技术规格等结构化信息
5. **掌握 PDF 处理中的常见陷阱** -- 能够处理编码问题、扫描件、加密 PDF、复杂排版等边界情况
6. **完成一个完整、可运行的 PDF 分析系统** -- 包括命令行界面和 Web 界面两种使用方式

## 核心问题

1. **PDF 解析为什么这么难？** 和纯文本或 HTML 相比，PDF 的内部结构有什么特殊之处？这给自动化处理带来了哪些挑战？
2. **RAG（检索增强生成）在文档分析中起什么作用？** 为什么不能直接把整个 PDF 塞给 LLM？分块策略如何影响回答质量？
3. **如何确保 Agent 从 PDF 中提取的信息是准确的？** LLM 可能会"幻觉"出不存在的信息，我们如何让 Agent 只基于文档内容回答问题？

---

## 58.1 项目背景与需求分析

### 58.1.1 为什么需要 PDF 分析 Agent

在企业环境中，大量关键信息以 PDF 格式存储。法律合同、财务报告、技术手册、学术论文、政府文件——这些都是 PDF。一个典型的法务团队可能需要从数百页的合同中找出特定的条款，一个投资分析师可能需要从数十份财报中提取关键财务指标，一个研究人员可能需要从大量论文中筛选最相关的文献。

传统做法是人工阅读这些 PDF，然后手动提取信息。这个过程不仅耗时，而且容易出错——人类在阅读长文档时很容易遗漏关键信息，特别是在疲劳的时候。

PDF 分析 Agent 的价值在于，它能够以远超人类的速度和一致性来处理 PDF 文档。它可以几分钟内读完一份 100 页的报告，提取所有关键数据，并以结构化的格式呈现给用户。当然，它不能替代人类的判断力，但它可以把人类从繁琐的信息提取工作中解放出来。

### 58.1.2 核心功能定义

基于以上分析，我们确定 PDF 分析 Agent 需要具备以下核心功能：

首先是**PDF 解析功能**。Agent 需要能够读取 PDF 文件，理解其中的文本内容、表格数据、图片信息等。不同的 PDF 结构差异很大——有些是纯文本排版，有些包含复杂的表格和图表，有些甚至是扫描的图片。Agent 需要能够处理各种类型的 PDF。

其次是**智能问答功能**。用户可以针对 PDF 的内容提出问题，Agent 基于文档内容给出准确的回答。关键在于，Agent 的回答必须基于文档内容，不能凭空捏造信息。每个回答都应该标注信息来源。

然后是**结构化提取功能**。除了回答自由形式的问题外，Agent 还需要能够按照预定义的模板提取结构化信息。例如，从合同中提取甲乙双方、合同金额、有效期等字段。

最后是**多文档分析功能**。Agent 需要能够同时分析多份 PDF，进行横向对比和综合分析。例如，对比两份合同的条款差异，或综合多份财报的数据。

---

## 58.2 技术选型与架构设计

### 58.2.1 PDF 解析库选择

PDF 解析是这个项目的基础，选择合适的解析库至关重要。Python 生态中有几个主流的 PDF 解析库，各有优劣。

**PyPDF2** 是最老牌的 PDF 库，它能够提取文本和元数据，但对复杂排版和表格的支持较弱。它的优点是纯 Python 实现，不需要安装系统依赖。

**pdfplumber** 是基于 pdfminer 的高级封装，它在文本提取和表格识别方面表现更好。它能够自动检测表格边界并提取为结构化数据。

**PyMuPDF（fitz）** 是一个高性能的 PDF 库，基于 MuPDF 引擎。它能够处理文本、图片、注释等多种元素，解析速度非常快。

**unstructured.io** 是一个专门为 AI 应用设计的文档解析库，它能够自动识别文档的结构，将 PDF 分解为标题、段落、表格、图片等元素。

我们选择 **pdfplumber** 作为主要的解析库，因为它在文本和表格提取方面平衡了易用性和准确性。同时使用 **PyMuPDF** 作为辅助库，用于处理图片提取等 pdfplumber 不擅长的任务。

### 58.2.2 整体架构

```
pdf-agent/
    main.py              # 入口文件
    pdf_parser.py        # PDF 解析器
    chunker.py           # 文档分块器
    vector_store.py      # 向量存储
    qa_agent.py          # 问答 Agent
    extractor.py         # 结构化信息提取器
    prompts.py           # 提示词模板
    config.py            # 配置管理
    web_app.py           # Web 界面（可选）
    requirements.txt     # 依赖列表
    docs/                # 存放待分析的 PDF 文件
    output/              # 输出目录
```

---

## 58.3 核心代码实现

### 58.3.1 PDF 解析器

```python
# pdf_parser.py
import pdfplumber
import fitz  # PyMuPDF
import os
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class PDFContent:
    """PDF 文档的结构化内容表示。"""
    filename: str
    page_count: int
    metadata: dict = field(default_factory=dict)
    pages: list = field(default_factory=list)
    tables: list = field(default_factory=list)
    images: list = field(default_factory=list)
    full_text: str = ""
    total_chars: int = 0


@dataclass
class PageContent:
    """单页内容。"""
    page_number: int
    text: str
    tables: list = field(default_factory=list)
    width: float = 0
    height: float = 0


class PDFParser:
    """PDF 文档解析器。

    这个类使用 pdfplumber 解析 PDF 的文本和表格，
    使用 PyMuPDF 解析图片和元数据。
    两种库的组合能够覆盖大多数 PDF 类型。
    """

    def __init__(self):
        self.supported_formats = [".pdf"]

    def parse(self, filepath: str) -> PDFContent:
        """解析 PDF 文件并返回结构化内容。

        Args:
            filepath: PDF 文件路径

        Returns:
            PDFContent 对象，包含文档的所有结构化内容

        Raises:
            FileNotFoundError: 文件不存在
            ValueError: 文件不是有效的 PDF
        """
        if not os.path.exists(filepath):
            raise FileNotFoundError(f"文件不存在: {filepath}")

        if not filepath.lower().endswith(".pdf"):
            raise ValueError(f"不支持的文件格式: {filepath}")

        filename = os.path.basename(filepath)
        content = PDFContent(filename=filename, page_count=0)

        # 使用 pdfplumber 解析文本和表格
        try:
            self._parse_with_pdfplumber(filepath, content)
        except Exception as e:
            print(f"[pdfplumber 解析警告] {e}，尝试备用方案...")
            self._parse_with_pymupdf_only(filepath, content)

        # 使用 PyMuPDF 补充元数据和图片
        try:
            self._parse_metadata_with_pymupdf(filepath, content)
        except Exception as e:
            print(f"[PyMuPDF 元数据解析警告] {e}")

        content.total_chars = len(content.full_text)
        print(f"[解析完成] {filename}: {content.page_count} 页, "
              f"{content.total_chars} 字符, {len(content.tables)} 个表格")

        return content

    def _parse_with_pdfplumber(self, filepath: str, content: PDFContent):
        """使用 pdfplumber 解析文本和表格。"""
        with pdfplumber.open(filepath) as pdf:
            content.page_count = len(pdf.pages)
            content.metadata = pdf.metadata or {}

            all_text_parts = []

            for i, page in enumerate(pdf.pages):
                page_content = PageContent(
                    page_number=i + 1,
                    width=page.width,
                    height=page.height
                )

                # 提取文本
                text = page.extract_text() or ""
                page_content.text = text
                all_text_parts.append(f"\n--- 第 {i + 1} 页 ---\n{text}")

                # 提取表格
                tables = page.extract_tables()
                for table_data in tables:
                    if table_data and len(table_data) > 0:
                        # 将表格转换为 Markdown 格式
                        table_md = self._table_to_markdown(table_data)
                        page_content.tables.append({
                            "page": i + 1,
                            "data": table_data,
                            "markdown": table_md
                        })
                        content.tables.append({
                            "page": i + 1,
                            "markdown": table_md
                        })

                content.pages.append(page_content)

            content.full_text = "\n".join(all_text_parts)

    def _parse_metadata_with_pymupdf(self, filepath: str, content: PDFContent):
        """使用 PyMuPDF 补充元数据。"""
        doc = fitz.open(filepath)
        meta = doc.metadata
        if meta:
            content.metadata.update({
                "title": meta.get("title", ""),
                "author": meta.get("author", ""),
                "subject": meta.get("subject", ""),
                "creator": meta.get("creator", ""),
                "producer": meta.get("producer", ""),
                "creation_date": meta.get("creationDate", ""),
                "modification_date": meta.get("modDate", ""),
            })
        doc.close()

    def _parse_with_pymupdf_only(self, filepath: str, content: PDFContent):
        """使用 PyMuPDF 作为备用解析方案。"""
        doc = fitz.open(filepath)
        content.page_count = len(doc)

        all_text_parts = []
        for i in range(len(doc)):
            page = doc[i]
            text = page.get_text("text")
            page_content = PageContent(
                page_number=i + 1,
                text=text,
                width=page.rect.width,
                height=page.rect.height
            )
            all_text_parts.append(f"\n--- 第 {i + 1} 页 ---\n{text}")
            content.pages.append(page_content)

        content.full_text = "\n".join(all_text_parts)
        doc.close()

    def _table_to_markdown(self, table_data: list) -> str:
        """将表格数据转换为 Markdown 格式。"""
        if not table_data or len(table_data) < 1:
            return ""

        # 清理单元格内容
        cleaned = []
        for row in table_data:
            cleaned_row = []
            for cell in row:
                if cell is None:
                    cleaned_row.append("")
                else:
                    # 清理文本中的换行符和多余空格
                    cell_text = str(cell).replace("\n", " ").strip()
                    cleaned_row.append(cell_text)
            cleaned.append(cleaned_row)

        # 生成 Markdown 表格
        if not cleaned:
            return ""

        # 表头
        header = "| " + " | ".join(cleaned[0]) + " |"
        separator = "| " + " | ".join(["---"] * len(cleaned[0])) + " |"

        lines = [header, separator]
        for row in cleaned[1:]:
            # 确保每行列数一致
            while len(row) < len(cleaned[0]):
                row.append("")
            lines.append("| " + " | ".join(row[:len(cleaned[0])]) + " |")

        return "\n".join(lines)

    def extract_text_for_analysis(self, content: PDFContent, max_length: int = 100000) -> str:
        """提取适合发送给 LLM 分析的文本。

        这个方法做了额外的清理工作，确保文本适合 LLM 处理。
        """
        text = content.full_text

        # 清理多余空白
        import re
        text = re.sub(r'\n{3,}', '\n\n', text)
        text = re.sub(r' {2,}', ' ', text)

        # 截断过长文本
        if len(text) > max_length:
            text = text[:max_length] + "\n\n[文本已截断，以上为前部分]"

        return text
```

### 58.3.2 文档分块器

为什么需要分块？因为 RAG（检索增强生成）系统的核心思路是：将长文档切分成小块，每块建立向量索引，查询时只检索最相关的几个块发送给 LLM。这样既避免了上下文窗口溢出，又能确保 LLM 接收到的信息是高度相关的。

```python
# chunker.py
import re
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class Chunk:
    """文档分块。"""
    chunk_id: int
    text: str
    source_page: int
    chunk_type: str  # "text", "table", "header"
    metadata: dict = field(default_factory=dict)
    start_pos: int = 0
    end_pos: int = 0


class DocumentChunker:
    """文档分块器。

    分块策略是 RAG 系统中最重要的设计决策之一。
    分块太大，检索精度低；分块太小，丢失上下文。
    我们使用"重叠窗口"策略来平衡这两者。
    """

    def __init__(self, chunk_size: int = 1000, chunk_overlap: int = 200):
        """
        Args:
            chunk_size: 每个块的目标大小（字符数）
            chunk_overlap: 相邻块之间的重叠大小
        """
        self.chunk_size = chunk_size
        self.chunk_overlap = chunk_overlap

    def chunk_text(self, text: str, source_page: int = 0) -> list[Chunk]:
        """将文本切分为重叠的块。

        策略：优先在段落边界处分割，如果段落太长则在句子边界分割。
        """
        chunks = []
        # 先按段落分割
        paragraphs = re.split(r'\n\s*\n', text)

        current_text = ""
        chunk_id = 0

        for para in paragraphs:
            para = para.strip()
            if not para:
                continue

            # 如果当前块加上新段落不超过限制，合并
            if len(current_text) + len(para) + 2 <= self.chunk_size:
                current_text = current_text + "\n\n" + para if current_text else para
            else:
                # 保存当前块
                if current_text:
                    chunks.append(Chunk(
                        chunk_id=chunk_id,
                        text=current_text,
                        source_page=source_page,
                        chunk_type="text",
                        start_pos=len(chunks) * (self.chunk_size - self.chunk_overlap),
                    ))
                    chunk_id += 1

                # 如果单个段落超过块大小，按句子分割
                if len(para) > self.chunk_size:
                    sub_chunks = self._split_long_paragraph(para, source_page, chunk_id)
                    chunks.extend(sub_chunks)
                    chunk_id += len(sub_chunks)
                    current_text = ""
                else:
                    # 保留重叠部分
                    overlap_text = current_text[-self.chunk_overlap:] if current_text else ""
                    current_text = overlap_text + "\n\n" + para if overlap_text else para

        # 不要忘记最后一块
        if current_text:
            chunks.append(Chunk(
                chunk_id=chunk_id,
                text=current_text,
                source_page=source_page,
                chunk_type="text",
            ))

        return chunks

    def _split_long_paragraph(self, text: str, source_page: int, start_id: int) -> list[Chunk]:
        """将过长的段落在句子边界处分割。"""
        chunks = []
        # 按句子分割（中英文标点）
        sentences = re.split(r'(?<=[。！？.!?])\s*', text)

        current_text = ""
        chunk_id = start_id

        for sentence in sentences:
            sentence = sentence.strip()
            if not sentence:
                continue

            if len(current_text) + len(sentence) + 1 <= self.chunk_size:
                current_text = current_text + " " + sentence if current_text else sentence
            else:
                if current_text:
                    chunks.append(Chunk(
                        chunk_id=chunk_id,
                        text=current_text,
                        source_page=source_page,
                        chunk_type="text",
                    ))
                    chunk_id += 1
                current_text = sentence

        if current_text:
            chunks.append(Chunk(
                chunk_id=chunk_id,
                text=current_text,
                source_page=source_page,
                chunk_type="text",
            ))

        return chunks

    def chunk_pdf_content(self, pdf_content) -> list[Chunk]:
        """解析完整的 PDFContent 对象。"""
        all_chunks = []
        chunk_id = 0

        for page in pdf_content.pages:
            # 分块页面文本
            if page.text:
                text_chunks = self.chunk_text(page.text, page.page_number)
                for tc in text_chunks:
                    tc.chunk_id = chunk_id
                    all_chunks.append(tc)
                    chunk_id += 1

            # 表格作为独立块
            for table in page.tables:
                table_chunk = Chunk(
                    chunk_id=chunk_id,
                    text=table["markdown"],
                    source_page=page.page_number,
                    chunk_type="table",
                    metadata={"has_table": True}
                )
                all_chunks.append(table_chunk)
                chunk_id += 1

        print(f"[分块完成] 共 {len(all_chunks)} 个块，"
              f"平均大小 {sum(len(c.text) for c in all_chunks) // max(len(all_chunks), 1)} 字符")
        return all_chunks
```

### 58.3.3 向量存储

```python
# vector_store.py
import numpy as np
from openai import OpenAI
from dataclasses import dataclass
from typing import Optional


@dataclass
class SearchResult:
    """检索结果。"""
    chunk_id: int
    text: str
    score: float
    source_page: int
    metadata: dict


class VectorStore:
    """简易向量存储。

    这是一个纯内存的向量存储实现，不依赖外部数据库。
    适合中小规模的文档处理（几千个块以内）。
    如果需要处理更大规模的数据，建议使用 ChromaDB 或 Pinecone。
    """

    def __init__(self, api_key: str, model: str = "text-embedding-3-small"):
        self.client = OpenAI(api_key=api_key)
        self.embedding_model = model
        self.chunks: list[dict] = []
        self.embeddings: list[list[float]] = []

    def add_chunks(self, chunks: list):
        """将文档块添加到向量存储中。

        分批处理 embedding 请求，避免单次请求过大。
        """
        batch_size = 100
        for i in range(0, len(chunks), batch_size):
            batch = chunks[i:i + batch_size]
            texts = [c.text for c in batch]

            # 调用 OpenAI Embedding API
            try:
                response = self.client.embeddings.create(
                    model=self.embedding_model,
                    input=texts
                )
                batch_embeddings = [item.embedding for item in response.data]

                for chunk, embedding in zip(batch, batch_embeddings):
                    self.chunks.append({
                        "chunk_id": chunk.chunk_id,
                        "text": chunk.text,
                        "source_page": chunk.source_page,
                        "chunk_type": chunk.chunk_type,
                        "metadata": getattr(chunk, "metadata", {}),
                    })
                    self.embeddings.append(embedding)

            except Exception as e:
                print(f"[Embedding 批次 {i // batch_size + 1} 失败] {e}")

        print(f"[向量索引] 已索引 {len(self.chunks)} 个块")

    def search(self, query: str, top_k: int = 5) -> list[SearchResult]:
        """根据查询检索最相关的文档块。

        使用余弦相似度进行向量检索。
        """
        if not self.embeddings:
            return []

        # 将查询转换为向量
        try:
            response = self.client.embeddings.create(
                model=self.embedding_model,
                input=[query]
            )
            query_embedding = response.data[0].embedding
        except Exception as e:
            print(f"[Embedding 查询失败] {e}")
            return []

        # 计算余弦相似度
        query_vec = np.array(query_embedding)
        similarities = []
        for i, emb in enumerate(self.embeddings):
            emb_vec = np.array(emb)
            cosine_sim = np.dot(query_vec, emb_vec) / (
                np.linalg.norm(query_vec) * np.linalg.norm(emb_vec)
            )
            similarities.append((i, cosine_sim))

        # 排序并返回 top_k 结果
        similarities.sort(key=lambda x: x[1], reverse=True)

        results = []
        for idx, score in similarities[:top_k]:
            chunk = self.chunks[idx]
            results.append(SearchResult(
                chunk_id=chunk["chunk_id"],
                text=chunk["text"],
                score=float(score),
                source_page=chunk["source_page"],
                metadata=chunk["metadata"],
            ))

        return results
```

### 58.3.4 PDF 问答 Agent

```python
# qa_agent.py
import json
from openai import OpenAI
from pdf_parser import PDFParser, PDFContent
from chunker import DocumentChunker, Chunk
from vector_store import VectorStore, SearchResult
from prompts import QA_SYSTEM_PROMPT, EXTRACT_SYSTEM_PROMPT


class PDFAgent:
    """PDF 分析 Agent。

    这个 Agent 能够：
    1. 加载和解析 PDF 文档
    2. 建立文档的向量索引
    3. 基于文档内容回答用户问题
    4. 提取结构化信息
    """

    def __init__(self, api_key: str, model: str = "gpt-4o",
                 base_url: str = None, chunk_size: int = 1000):
        self.client = OpenAI(api_key=api_key, base_url=base_url)
        self.model = model
        self.parser = PDFParser()
        self.chunker = DocumentChunker(chunk_size=chunk_size)
        self.vector_store = VectorStore(api_key=api_key)
        self.loaded_documents: dict[str, PDFContent] = {}
        self.total_chunks = 0

    def load_pdf(self, filepath: str) -> dict:
        """加载 PDF 文件并建立索引。

        Returns:
            包含加载状态信息的字典
        """
        # 解析 PDF
        pdf_content = self.parser.parse(filepath)
        self.loaded_documents[pdf_content.filename] = pdf_content

        # 分块
        chunks = self.chunker.chunk_pdf_content(pdf_content)

        # 建立向量索引
        self.vector_store.add_chunks(chunks)
        self.total_chunks += len(chunks)

        return {
            "filename": pdf_content.filename,
            "pages": pdf_content.page_count,
            "chunks": len(chunks),
            "total_chars": pdf_content.total_chars,
            "tables": len(pdf_content.tables),
            "total_indexed": self.total_chunks
        }

    def load_multiple_pdfs(self, filepaths: list[str]) -> list[dict]:
        """加载多个 PDF 文件。"""
        results = []
        for fp in filepaths:
            try:
                result = self.load_pdf(fp)
                results.append({"status": "success", **result})
            except Exception as e:
                results.append({"status": "error", "file": fp, "error": str(e)})
        return results

    def ask(self, question: str, top_k: int = 5, return_sources: bool = True) -> dict:
        """基于已加载的 PDF 内容回答用户问题。

        这是 RAG 的核心流程：
        1. 将问题转换为向量
        2. 检索最相关的文档块
        3. 将相关块和问题一起发送给 LLM
        4. LLM 基于提供的上下文生成回答
        """
        # 检索相关文档块
        search_results = self.vector_store.search(question, top_k=top_k)

        if not search_results:
            return {
                "answer": "抱歉，未在已加载的文档中找到相关信息。请确认已加载相关 PDF 文件。",
                "sources": [],
                "confidence": "low"
            }

        # 构建上下文
        context_parts = []
        for i, result in enumerate(search_results):
            context_parts.append(
                f"[来源 {i+1}] (第 {result.source_page} 页, 相关度: {result.score:.2f})\n"
                f"{result.text}"
            )
        context = "\n\n---\n\n".join(context_parts)

        # 调用 LLM
        messages = [
            {"role": "system", "content": QA_SYSTEM_PROMPT},
            {"role": "user", "content": f"以下是文档中的相关内容：\n\n{context}\n\n"
                                        f"用户的问题是：{question}"}
        ]

        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=messages,
                temperature=0.1,  # 事实性任务用低温度
                max_tokens=2000,
            )
            answer = response.choices[0].message.content
        except Exception as e:
            answer = f"回答生成失败: {e}"

        result = {
            "answer": answer,
            "confidence": self._estimate_confidence(search_results),
        }

        if return_sources:
            result["sources"] = [
                {
                    "page": r.source_page,
                    "score": round(r.score, 3),
                    "excerpt": r.text[:200] + "..." if len(r.text) > 200 else r.text
                }
                for r in search_results
            ]

        return result

    def extract_structured_info(self, template: dict, pages: list[int] = None) -> dict:
        """从 PDF 中提取结构化信息。

        Args:
            template: 提取模板，格式为 {"字段名": "字段描述", ...}
            pages: 指定提取的页码范围（None 表示全部）

        Returns:
            提取的结构化信息字典
        """
        # 获取相关文本
        if pages:
            all_text = ""
            for doc in self.loaded_documents.values():
                for page in doc.pages:
                    if page.page_number in pages:
                        all_text += page.text + "\n\n"
        else:
            all_text = "\n\n".join(
                doc.full_text for doc in self.loaded_documents.values()
            )

        # 构建提取提示
        template_str = json.dumps(template, ensure_ascii=False, indent=2)

        messages = [
            {"role": "system", "content": EXTRACT_SYSTEM_PROMPT},
            {"role": "user", "content": f"请从以下文档内容中提取结构化信息。\n\n"
                                        f"提取模板：\n{template_str}\n\n"
                                        f"文档内容：\n{all_text[:50000]}"}
        ]

        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=messages,
                temperature=0.0,  # 提取任务用最低温度
                max_tokens=2000,
                response_format={"type": "json_object"},
            )
            result = json.loads(response.choices[0].message.content)
        except Exception as e:
            result = {"error": str(e)}

        return result

    def summarize(self, max_length: int = 1000) -> str:
        """生成已加载 PDF 的摘要。"""
        all_text = "\n\n".join(
            doc.full_text[:10000] for doc in self.loaded_documents.values()
        )

        messages = [
            {"role": "system", "content": "你是一个专业的文档摘要专家。请对以下文档内容生成简洁的摘要。"},
            {"role": "user", "content": f"请为以下文档生成摘要（约 {max_length} 字）：\n\n{all_text}"}
        ]

        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=messages,
                temperature=0.2,
                max_tokens=1500,
            )
            return response.choices[0].message.content
        except Exception as e:
            return f"摘要生成失败: {e}"

    def compare_documents(self, doc1_name: str, doc2_name: str) -> str:
        """对比分析两份文档。"""
        doc1 = self.loaded_documents.get(doc1_name)
        doc2 = self.loaded_documents.get(doc2_name)

        if not doc1 or not doc2:
            return "请确保两份文档都已加载。"

        text1 = doc1.full_text[:15000]
        text2 = doc2.full_text[:15000]

        messages = [
            {"role": "system", "content": (
                "你是一个专业的文档分析师。请对比两份文档的内容，"
                "从以下维度进行分析：\n"
                "1. 主题和目的的异同\n"
                "2. 关键内容的差异\n"
                "3. 数据和事实的对比\n"
                "4. 结论和观点的异同\n"
                "请用中文回答，结构清晰。"
            )},
            {"role": "user", "content": (
                f"文档 1: {doc1_name}\n内容:\n{text1}\n\n"
                f"文档 2: {doc2_name}\n内容:\n{text2}"
            )}
        ]

        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=messages,
                temperature=0.2,
                max_tokens=3000,
            )
            return response.choices[0].message.content
        except Exception as e:
            return f"对比分析失败: {e}"

    def _estimate_confidence(self, results: list[SearchResult]) -> str:
        """根据检索结果估计回答的置信度。"""
        if not results:
            return "low"
        max_score = max(r.score for r in results)
        if max_score > 0.85:
            return "high"
        elif max_score > 0.7:
            return "medium"
        else:
            return "low"
```

### 58.3.5 提示词设计

```python
# prompts.py

QA_SYSTEM_PROMPT = """你是一个专业的文档分析助手。你的任务是基于提供的文档内容，准确回答用户的问题。

## 回答原则

1. **严格基于文档**：你的回答必须完全基于提供的文档内容，不要添加文档中没有的信息。
2. **标注来源**：在回答中引用具体信息时，标注它来自哪个来源（如"根据第3页的内容..."）。
3. **承认未知**：如果文档中没有包含回答问题所需的信息，明确告诉用户"文档中未包含此信息"。
4. **保持准确**：对于数字、日期、名称等关键信息，确保与文档完全一致。
5. **逻辑清晰**：如果问题涉及多个来源的信息，需要综合分析并说明信息来源。

## 回答格式

1. 先给出直接的回答
2. 然后提供支持该回答的文档证据
3. 如果适用，说明回答的置信度
4. 如果信息不完整，说明需要补充什么信息

## 注意事项

- 不要猜测或推断文档中没有明确说明的信息
- 对于矛盾的信息，指出两个来源并说明差异
- 如果问题超出文档范围，建议用户查找其他来源
"""

EXTRACT_SYSTEM_PROMPT = """你是一个专业的信息提取专家。你的任务是从文档内容中精确提取指定的结构化信息。

## 提取原则

1. **精确提取**：只提取文档中明确出现的信息，不要推测或补全
2. **保持原样**：数字、日期、名称等信息保持原文中的格式
3. **缺失标注**：如果某个字段在文档中找不到对应信息，标记为 null
4. **JSON 输出**：严格按照要求的 JSON 格式输出
5. **去噪处理**：忽略文档中的页眉、页脚、页码等无关内容

## 处理特殊情况

- 表格中的数据：保持数值格式
- 多值字段：使用数组格式
- 日期格式：尽量统一为 YYYY-MM-DD 格式
- 金额格式：保留原始货币单位和数值
"""

SUMMARY_SYSTEM_PROMPT = """你是一个专业的文档摘要专家。请按照以下结构生成摘要：

1. **核心主题**：文档主要讲什么
2. **关键要点**：3-5个最重要的信息点
3. **重要数据**：文档中提到的关键数字和统计
4. **结论**：文档的主要结论或建议

摘要应该简洁、准确、结构清晰。
"""
```

### 58.3.6 命令行界面

```python
# main.py
import sys
import os
import json
from config import load_config
from qa_agent import PDFAgent


def print_banner():
    print("=" * 60)
    print("  PDF 分析 Agent v1.0")
    print("  让 AI 帮你理解 PDF 文档")
    print("=" * 60)


def interactive_mode(agent: PDFAgent):
    """交互式问答模式。"""
    print("\n进入交互式模式。输入问题开始分析。")
    print("命令：")
    print("  /load <文件路径>  - 加载 PDF 文件")
    print("  /list            - 列出已加载的文件")
    print("  /extract         - 提取结构化信息")
    print("  /compare         - 对比两份文档")
    print("  /summarize       - 生成文档摘要")
    print("  /quit            - 退出程序")
    print("-" * 40)

    while True:
        try:
            user_input = input("\n> ").strip()
        except (EOFError, KeyboardInterrupt):
            print("\n再见！")
            break

        if not user_input:
            continue

        if user_input == "/quit":
            print("再见！")
            break

        elif user_input.startswith("/load "):
            filepath = user_input[6:].strip().strip('"').strip("'")
            if not os.path.exists(filepath):
                print(f"文件不存在: {filepath}")
                continue
            try:
                result = agent.load_pdf(filepath)
                print(f"\n加载成功！")
                print(f"  文件名: {result['filename']}")
                print(f"  页数: {result['pages']}")
                print(f"  字符数: {result['total_chars']}")
                print(f"  表格数: {result['tables']}")
                print(f"  分块数: {result['chunks']}")
            except Exception as e:
                print(f"加载失败: {e}")

        elif user_input == "/list":
            if not agent.loaded_documents:
                print("尚未加载任何文件。")
            else:
                print(f"\n已加载 {len(agent.loaded_documents)} 个文件：")
                for name, doc in agent.loaded_documents.items():
                    print(f"  - {name} ({doc.page_count} 页, {doc.total_chars} 字符)")

        elif user_input == "/summarize":
            print("\n正在生成摘要...")
            summary = agent.summarize()
            print(f"\n{summary}")

        elif user_input.startswith("/compare"):
            docs = list(agent.loaded_documents.keys())
            if len(docs) < 2:
                print("需要加载至少两份文档才能进行对比。")
                continue
            print(f"\n可用文档：")
            for i, name in enumerate(docs):
                print(f"  {i+1}. {name}")
            try:
                choice = input("选择对比的文档编号（格式：1 2）：").strip()
                idx1, idx2 = map(int, choice.split())
                result = agent.compare_documents(docs[idx1-1], docs[idx2-1])
                print(f"\n{result}")
            except Exception as e:
                print(f"对比失败: {e}")

        elif user_input == "/extract":
            print("\n请以 JSON 格式输入提取模板，例如：")
            print('  {"甲方": "合同甲方名称", "金额": "合同总金额", "日期": "签署日期"}')
            try:
                template_str = input("\n提取模板: ").strip()
                template = json.loads(template_str)
                print("\n正在提取...")
                result = agent.extract_structured_info(template)
                print(f"\n提取结果：")
                print(json.dumps(result, ensure_ascii=False, indent=2))
            except json.JSONDecodeError:
                print("JSON 格式不正确，请重试。")
            except Exception as e:
                print(f"提取失败: {e}")

        else:
            # 默认作为问题处理
            print("\n正在分析...")
            result = agent.ask(user_input)
            print(f"\n回答：")
            print(result["answer"])
            if result.get("sources"):
                print(f"\n参考来源（置信度: {result['confidence']}）：")
                for src in result["sources"]:
                    print(f"  - 第 {src['page']} 页 (相关度: {src['score']})")


def main():
    print_banner()

    # 检查 API Key
    api_key = os.environ.get("OPENAI_API_KEY")
    if not api_key:
        print("\n[错误] 请先设置 OPENAI_API_KEY 环境变量。")
        return

    # 创建 Agent
    agent = PDFAgent(
        api_key=api_key,
        model=os.environ.get("PDF_AGENT_MODEL", "gpt-4o"),
        base_url=os.environ.get("OPENAI_BASE_URL", None),
    )

    # 命令行参数模式
    if len(sys.argv) > 1:
        filepath = sys.argv[1]
        if os.path.exists(filepath):
            result = agent.load_pdf(filepath)
            print(f"\n已加载 {result['filename']}")

            if len(sys.argv) > 2:
                question = " ".join(sys.argv[2:])
                answer = agent.ask(question)
                print(f"\n回答：\n{answer['answer']}")
            else:
                interactive_mode(agent)
        else:
            print(f"文件不存在: {filepath}")
    else:
        # 交互式模式
        interactive_mode(agent)


if __name__ == "__main__":
    main()
```

---

## 58.4 运行指南

### 58.4.1 安装依赖

```bash
# 创建虚拟环境
python -m venv pdf-agent-env
source pdf-agent-env/bin/activate  # Linux/Mac
# pdf-agent-env\Scripts\activate   # Windows

# 安装依赖
pip install pdfplumber PyMuPDF openai numpy httpx

# 设置 API Key
export OPENAI_API_KEY="your-api-key-here"  # Linux/Mac
# $env:OPENAI_API_KEY = "your-api-key-here"  # Windows PowerShell
```

### 58.4.2 使用方式

```bash
# 方式一：直接在命令行提问
python main.py document.pdf "这份报告的核心结论是什么？"

# 方式二：交互式模式
python main.py document.pdf

# 方式三：先加载再提问
python main.py
# 然后在交互界面中使用 /load 命令加载文件
```

---

## 58.5 案例分析

### 58.5.1 案例一：分析年度财报

假设你收到了一份上市公司年报 PDF（200 页）。使用我们的 PDF 分析 Agent，你可以这样操作：

```
> /load annual_report_2025.pdf
加载成功！文件名: annual_report_2025.pdf, 页数: 200, 字符数: 85000

> 请提取公司的关键财务指标
[Agent 回答]
根据年报内容，2025 年度关键财务数据如下：
- 总收入：125.3 亿元（同比增长 15.2%）
- 净利润：18.7 亿元（同比增长 22.1%）
- 研发投入：12.4 亿元（占总收入的 9.9%）
...

> /extract
提取模板: {"收入": "公司总收入", "利润": "净利润", "增长": "收入增长率", "员工数": "公司总人数"}

提取结果：
{
    "收入": "125.3亿元",
    "利润": "18.7亿元",
    "增长": "15.2%",
    "员工数": "3,456人"
}
```

### 58.5.2 案例二：合同条款对比

法务团队需要对比两份供应商合同的条款差异：

```
> /load contract_v1.pdf
> /load contract_v2.pdf

> /compare
选择对比的文档编号（格式：1 2）：1 2

[对比分析结果]
两份合同的主要差异如下：
1. 付款条件：V1 为 30 天账期，V2 改为 45 天账期
2. 违约责任：V2 新增了不可抗力免责条款
3. 保密期限：从 2 年延长至 3 年
...
```

---

## 58.6 常见坑与解决方案

### 坑一：PDF 编码问题导致乱码

**问题描述：** 某些 PDF 文件使用了特殊的字体编码，提取出来的文本是乱码。

**解决方案：** 首先尝试使用 PyMuPDF 替代 pdfplumber 进行解析。如果仍然乱码，可能是因为 PDF 是扫描件（图片），需要用 OCR 技术（如 pytesseract）先将图片转换为文本。

### 坑二：表格提取不准确

**问题描述：** 复杂的合并单元格表格，提取出来格式混乱。

**解决方案：** 增加表格后处理逻辑，自动检测合并单元格并填充。对于特别复杂的表格，可以考虑使用视觉模型（如 GPT-4o 的视觉能力）直接分析表格的截图。

### 坑三：检索结果不相关

**问题描述：** 用户提问后，检索到的文档块与问题不太相关。

**解决方案：** 尝试调整 chunk_size 和 chunk_overlap 参数。更小的块能提高检索精度，但可能丢失上下文。更好的做法是使用混合检索策略：同时使用向量检索和关键词检索。

### 坑四：回答中包含"幻觉"

**问题描述：** Agent 的回答中包含了文档里不存在的信息。

**解决方案：** 降低 temperature（建议 0.0-0.1），并在提示词中反复强调"只使用提供的文档内容回答"。如果仍然出现幻觉，考虑使用更强的模型或增加检索的 top_k 值。

### 坑五：大文件处理内存溢出

**问题描述：** 处理几百页的大型 PDF 时，内存使用量暴涨。

**解决方案：** 使用流式处理，一次只解析和索引一部分页面。或者在分块时减小 chunk_size，让每次发送给 embedding 模型的文本量更小。

---

## 58.7 练习题

### 练习一：添加 OCR 支持（难度：初级）

当前项目只能处理文本型 PDF，无法处理扫描件。请添加 OCR 支持：

要求：
1. 集成 pytesseract 或 PaddleOCR
2. 检测 PDF 是否为扫描件（文本内容过少则判定为扫描件）
3. 自动对扫描页进行 OCR 处理
4. 将 OCR 结果合并到正常解析流程中

### 练习二：实现 PDF 批量处理（难度：中级）

请为项目添加批量处理功能：

要求：
1. 支持指定文件夹，自动处理其中的所有 PDF
2. 每个 PDF 独立索引，支持跨文件检索
3. 生成批量处理报告
4. 支持断点续传（处理中断后可恢复）

### 练习三：添加 PDF 标注功能（难度：中级）

请实现一个功能，让 Agent 能够自动在 PDF 中标注关键段落：

要求：
1. 使用 PyMuPDF 的标注功能
2. 根据用户的问题，自动高亮相关段落
3. 在标注旁添加 AI 生成的注释
4. 保存标注后的 PDF 文件

### 练习四：实现多模态分析（难度：高级）

当前项目只处理文本和表格，不处理图片。请添加图片分析功能：

要求：
1. 使用 PyMuPDF 提取 PDF 中的图片
2. 使用 GPT-4o 的视觉能力分析图片内容
3. 将图片分析结果与文本分析结果整合
4. 在回答中能够引用图片中的信息

### 练习五：构建 Web 界面（难度：高级）

请为 PDF 分析 Agent 构建一个简单的 Web 界面：

要求：
1. 使用 Streamlit 或 Gradio 构建
2. 支持文件上传和拖拽
3. 实时显示解析进度
4. 对话式的问答界面
5. 展示检索到的来源和相关度

---

## 58.8 实战任务

### 任务一：分析真实 PDF

找一份你工作或学习中实际会用到的 PDF（合同、报告、论文等），使用本章构建的 Agent 进行分析。评估 Agent 的表现，记录它做得好的地方和不足之处。

### 任务二：优化分块策略

尝试不同的分块策略（不同的 chunk_size、chunk_overlap，以及按段落分块 vs 按固定长度分块），对比它们对问答质量的影响。用 10 个问题作为测试集，评估每种策略的准确率。

### 任务三：构建领域特定模板

为你的工作领域创建 5 个结构化提取模板。例如，如果你是财务分析师，可以创建"收入分析模板"、"风险评估模板"等。测试这些模板在真实 PDF 上的表现。

---

## 58.9 本章小结

PDF 分析 Agent 展示了 Agent 在文档处理领域的强大能力。通过将 PDF 解析、向量检索和 LLM 推理结合起来，我们构建了一个能够"理解" PDF 内容的智能系统。

关键收获有三点。第一，**PDF 解析是一个被低估的挑战**。不同的 PDF 结构差异巨大，没有一种万能的解析方案。在实际项目中，你可能需要组合多种解析库，并针对特定类型的 PDF 进行优化。

第二，**分块策略直接影响检索质量**。分块太大，检索精度低；分块太小，丢失上下文。重叠窗口策略是一个好的起点，但在生产环境中，你可能需要根据具体的文档类型和使用场景调整参数。

第三，**RAG 是处理长文档的标准模式**。虽然 LLM 的上下文窗口在不断扩大，但在处理多份大型文档时，RAG 仍然是最实用的方案。它不仅解决了上下文窗口的限制，还提供了信息来源的可追溯性，这对于企业应用来说至关重要。

下一章，我们将构建一个邮件 Agent，将 Agent 的能力延伸到日常办公自动化领域。
