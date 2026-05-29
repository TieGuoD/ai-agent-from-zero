# 第 57 章：项目 1 -- 研究助手 Agent

> **本章定位：** 这是实战项目的开篇之作。我们将从零开始构建一个真正能用的研究助手 Agent，它能够理解你的研究问题、自动搜索相关文献、提取关键信息、整理成结构化的研究报告。这个项目将把前面学到的所有 Agent 基础知识串联起来，让你体验从"知道"到"做到"的完整跨越。

---

## 学习目标

完成本章学习后，你将能够：

1. **设计一个完整的多步骤研究流程** -- 能够将"帮我研究某个话题"这样的模糊需求，拆解为搜索、筛选、阅读、分析、总结五个清晰的步骤
2. **实现基于工具调用的搜索系统** -- 能够让 Agent 自动调用搜索 API，获取互联网上的最新信息，而不是依赖模型的训练数据
3. **构建信息整合与去重机制** -- 能够让 Agent 从多个来源收集信息后，自动去重、交叉验证、合并相关信息
4. **实现结构化研究报告的生成** -- 能够让 Agent 将零散的信息片段整理成有逻辑、有层次的研究报告
5. **掌握 Agent 状态管理的核心技巧** -- 能够让 Agent 在多步骤执行过程中保持上下文连贯、不丢失关键信息
6. **完成一个可部署、可运行的完整项目** -- 能够将代码打包、配置环境、一键启动

## 核心问题

在开始构建之前，请先思考这三个问题。完成项目后，你应该能清晰地回答它们：

1. **研究助手 Agent 和搜索引擎有什么本质区别？** 为什么我们不能简单地用搜索引擎替代它？Agent 的"智能"体现在哪些地方？
2. **在多步骤的研究过程中，Agent 最容易在哪里出错？** 是搜索关键词生成不准？是信息筛选不当？还是报告组织混乱？我们如何逐一解决这些问题？
3. **如何平衡研究的深度和广度？** Agent 应该广泛搜索多个来源还是深入分析少数几篇核心文献？这个决策应该由谁来做？

---

## 5.1 项目背景与需求分析

### 5.1.1 为什么需要研究助手 Agent

想象这样一个场景：你是一名产品经理，老板让你在两天内提交一份关于"AI Agent 在金融行业的应用现状"的调研报告。传统做法是怎样的？你会打开 Google 搜索相关关键词，一篇一篇地阅读文章，手动记录关键信息，然后在 Word 里整理成报告。这个过程可能需要花费你一整天的时间，而且你还得忍受重复劳动带来的疲惫感。

现在，如果有一个 Agent 能够帮你完成这个过程中的大部分工作呢？你只需要告诉它"帮我调研 AI Agent 在金融行业的应用现状"，然后它就会自动搜索互联网、阅读相关文章、提取关键信息、分析行业趋势，最后生成一份结构化的研究报告。你只需要花半个小时审核和修改，就可以提交了。

这就是研究助手 Agent 的价值所在。它不是要替代人类的研究能力，而是要把人类从重复的信息收集和整理工作中解放出来，让人专注于更高层次的分析和决策。

### 5.1.2 功能需求定义

经过分析，我们确定研究助手 Agent 需要具备以下核心功能：

首先是**智能搜索功能**。Agent 需要能够根据用户的研究主题，自动生成多个搜索关键词，并调用搜索 API 获取互联网上的最新信息。这里的关键是，搜索不是一次性的，Agent 需要根据前几轮搜索的结果，动态调整搜索策略。

其次是**信息筛选功能**。搜索返回的结果中，既有高质量的行业报告和学术论文，也有低质量的营销软文和过时信息。Agent 需要能够根据信息源的权威性、内容的相关性、时效性等多个维度，自动筛选出有价值的信息。

然后是**深度阅读功能**。对于筛选出的高质量信息，Agent 需要能够提取其中的关键数据、观点、案例和结论。这不仅仅是简单的文本摘要，而是需要理解内容的逻辑结构。

最后是**报告生成功能**。Agent 需要将从多个来源收集到的信息，按照研究主题的逻辑结构组织起来，生成一份完整的研究报告。这份报告应该包含摘要、背景、现状分析、趋势预测、结论和建议等部分。

### 5.1.3 技术选型

我们选择以下技术栈来构建这个项目：

在模型层面，我们使用 OpenAI 的 GPT-4o 作为核心推理引擎。GPT-4o 在复杂推理和长文本处理方面表现出色，非常适合研究助手这种需要深度理解的任务。当然，你也可以替换成 Claude、Gemini 等其他模型，核心架构是通用的。

在工具层面，我们使用 DuckDuckGo Search API 作为搜索工具。它免费、无需 API Key、搜索质量不错，非常适合原型开发。如果你需要更高的搜索质量，可以升级到 Google Search API 或 SerpAPI。

在框架层面，我们使用纯 Python 加 OpenAI SDK 来实现，不依赖任何 Agent 框架。这让你能真正理解 Agent 的底层运行机制。当然，在学习完这个项目后，你可以尝试用 LangChain 或 CrewAI 重新实现一遍，对比两种方式的差异。

在输出层面，我们支持纯文本和 Markdown 两种格式。Markdown 格式可以直接在 Jupyter Notebook 中渲染，也可以导出为 PDF 或 HTML。

---

## 5.2 项目结构设计

一个清晰的项目结构是成功的一半。我们把研究助手 Agent 的代码组织成以下结构：

```
research-agent/
    main.py              # 入口文件，负责启动和协调
    agent.py             # Agent 核心逻辑
    tools.py             # 工具定义（搜索、阅读等）
    prompts.py           # 系统提示词和模板
    memory.py            # 记忆管理（研究上下文）
    report.py            # 报告生成器
    config.py            # 配置管理
    requirements.txt     # 依赖列表
    output/              # 报告输出目录
    README.md            # 项目说明文档
```

这个结构遵循了关注点分离的原则。每个文件负责一个明确的功能模块，修改其中一个不会影响其他模块。例如，你想把搜索工具从 DuckDuckGo 换成 Google Search，只需要修改 `tools.py`，其他文件完全不用动。

---

## 5.3 核心代码实现

### 5.3.1 配置管理

首先，我们实现配置管理模块。这个模块负责加载 API Key 和其他配置项：

```python
# config.py
import os
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class ResearchConfig:
    """研究助手 Agent 的配置类。

    所有敏感信息（如 API Key）都通过环境变量加载，
    绝不硬编码在代码中。这是安全开发的基本原则。
    """

    # OpenAI 配置
    openai_api_key: str = field(default_factory=lambda: os.environ.get("OPENAI_API_KEY", ""))
    openai_base_url: Optional[str] = field(default_factory=lambda: os.environ.get("OPENAI_BASE_URL", None))
    model_name: str = "gpt-4o"

    # 搜索配置
    search_provider: str = "duckduckgo"  # 可选: duckduckgo, serpapi, tavily
    max_search_results: int = 10
    max_search_rounds: int = 3

    # 阅读配置
    max_pages_to_read: int = 5
    max_content_length: int = 100000  # 单个页面最大字符数

    # 报告配置
    report_sections: list = field(default_factory=lambda: [
        "摘要", "背景与现状", "关键发现", "趋势分析", "结论与建议"
    ])
    target_report_length: int = 2000  # 目标报告字数

    # Agent 配置
    max_iterations: int = 20  # Agent 最大循环次数
    temperature: float = 0.3  # 研究任务偏保守，使用较低温度
    verbose: bool = True

    def validate(self) -> bool:
        """验证配置是否合法。"""
        if not self.openai_api_key:
            raise ValueError(
                "未找到 OpenAI API Key。请设置环境变量 OPENAI_API_KEY。\n"
                "Windows PowerShell: $env:OPENAI_API_KEY = 'your-key'\n"
                "Linux/Mac: export OPENAI_API_KEY='your-key'"
            )
        return True


def load_config(**overrides) -> ResearchConfig:
    """加载配置，支持通过参数覆盖默认值。"""
    config = ResearchConfig(**overrides)
    config.validate()
    return config
```

配置管理看起来简单，但它包含了几个重要的工程实践。第一，所有敏感信息通过环境变量加载，绝不硬编码在代码中。这不仅是安全要求，也是团队协作的基本规范。第二，使用 dataclass 而不是普通字典，这样 IDE 可以提供自动补全和类型检查。第三，提供 validate 方法在启动时验证配置，避免运行到一半才发现 API Key 没配置。

### 5.3.2 工具定义

接下来是整个项目最关键的模块之一 -- 工具定义。Agent 的能力边界由它的工具决定，所以我们需要精心设计每个工具。

```python
# tools.py
import json
import httpx
import re
from typing import Callable, Any
from dataclasses import dataclass, field

@dataclass
class Tool:
    """工具基类。每个工具有名称、描述、参数定义和执行函数。"""
    name: str
    description: str
    parameters: dict  # JSON Schema 格式的参数定义
    function: Callable

    def to_openai_format(self) -> dict:
        """转换为 OpenAI Function Calling 格式。"""
        return {
            "type": "function",
            "function": {
                "name": self.name,
                "description": self.description,
                "parameters": self.parameters
            }
        }


class SearchTool:
    """搜索工具：通过互联网搜索获取最新信息。"""

    def __init__(self, provider: str = "duckduckgo"):
        self.provider = provider
        self.client = httpx.Client(timeout=30.0)

    def search(self, query: str, max_results: int = 5) -> list[dict]:
        """执行搜索并返回结果列表。

        Args:
            query: 搜索关键词
            max_results: 最大返回结果数

        Returns:
            包含标题、链接、摘要的字典列表
        """
        if self.provider == "duckduckgo":
            return self._search_duckduckgo(query, max_results)
        elif self.provider == "serpapi":
            return self._search_serpapi(query, max_results)
        else:
            raise ValueError(f"不支持的搜索提供者: {self.provider}")

    def _search_duckduckgo(self, query: str, max_results: int) -> list[dict]:
        """使用 DuckDuckGo Instant Answer API 搜索。"""
        try:
            from duckduckgo_search import DDGS
            results = []
            with DDGS() as ddgs:
                for r in ddgs.text(query, max_results=max_results):
                    results.append({
                        "title": r.get("title", ""),
                        "url": r.get("href", ""),
                        "snippet": r.get("body", "")
                    })
            return results
        except ImportError:
            # 如果没安装 duckduckgo_search，使用简易 HTTP 请求
            return self._fallback_search(query, max_results)

    def _fallback_search(self, query: str, max_results: int) -> list[dict]:
        """备用搜索方案：直接请求 DuckDuckGo HTML 接口。"""
        url = "https://html.duckduckgo.com/html/"
        try:
            response = self.client.post(url, data={"q": query})
            # 简单的 HTML 解析
            results = []
            # 这里使用正则匹配，实际项目建议用 BeautifulSoup
            titles = re.findall(r'class="result__a"[^>]*>(.*?)</a>', response.text)
            snippets = re.findall(r'class="result__snippet">(.*?)</span>', response.text)
            for i in range(min(len(titles), max_results)):
                results.append({
                    "title": re.sub(r'<[^>]+>', '', titles[i]).strip(),
                    "url": "",
                    "snippet": re.sub(r'<[^>]+>', '', snippets[i]).strip() if i < len(snippets) else ""
                })
            return results
        except Exception as e:
            print(f"[搜索失败] {e}")
            return []

    def _search_serpapi(self, query: str, max_results: int) -> list[dict]:
        """使用 SerpAPI 搜索（需要 API Key）。"""
        import os
        api_key = os.environ.get("SERPAPI_KEY", "")
        if not api_key:
            raise ValueError("使用 SerpAPI 需要设置环境变量 SERPAPI_KEY")

        url = "https://serpapi.com/search"
        params = {
            "q": query,
            "api_key": api_key,
            "num": max_results
        }
        try:
            response = self.client.get(url, params=params)
            data = response.json()
            results = []
            for item in data.get("organic_results", [])[:max_results]:
                results.append({
                    "title": item.get("title", ""),
                    "url": item.get("link", ""),
                    "snippet": item.get("snippet", "")
                })
            return results
        except Exception as e:
            print(f"[SerpAPI 搜索失败] {e}")
            return []

    def get_tool(self) -> Tool:
        """返回 OpenAI Function Calling 格式的工具定义。"""
        return Tool(
            name="search_web",
            description=(
                "在互联网上搜索信息。当你需要获取最新资料、行业数据、"
                "技术趋势、市场信息等时，使用此工具。"
            ),
            parameters={
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "搜索关键词，建议使用英文以获得更好的搜索结果"
                    },
                    "max_results": {
                        "type": "integer",
                        "description": "最大返回结果数，默认5",
                        "default": 5
                    }
                },
                "required": ["query"]
            },
            function=self.search
        )


class WebpageReaderTool:
    """网页阅读工具：获取网页内容并提取关键信息。"""

    def __init__(self, max_content_length: int = 100000):
        self.max_content_length = max_content_length
        self.client = httpx.Client(
            timeout=30.0,
            headers={
                "User-Agent": "Mozilla/5.0 (compatible; ResearchBot/1.0)"
            }
        )

    def read(self, url: str) -> dict:
        """读取网页内容。

        Args:
            url: 网页 URL

        Returns:
            包含标题、内容、URL 的字典
        """
        try:
            response = self.client.get(url, follow_redirects=True)
            response.raise_for_status()
            content = response.text

            # 简单提取文本内容（去除 HTML 标签）
            # 移除 script 和 style 标签及其内容
            content = re.sub(r'<script[^>]*>.*?</script>', '', content, flags=re.DOTALL)
            content = re.sub(r'<style[^>]*>.*?</style>', '', content, flags=re.DOTALL)
            # 提取标题
            title_match = re.search(r'<title[^>]*>(.*?)</title>', content, re.DOTALL)
            title = title_match.group(1).strip() if title_match else url
            # 移除所有 HTML 标签
            text = re.sub(r'<[^>]+>', ' ', content)
            # 清理多余空白
            text = re.sub(r'\s+', ' ', text).strip()
            # 截断过长内容
            if len(text) > self.max_content_length:
                text = text[:self.max_content_length] + "...[内容已截断]"

            return {
                "title": title,
                "content": text,
                "url": url,
                "length": len(text)
            }
        except Exception as e:
            return {
                "title": f"无法访问: {url}",
                "content": f"读取失败: {str(e)}",
                "url": url,
                "length": 0
            }

    def get_tool(self) -> Tool:
        """返回 OpenAI Function Calling 格式的工具定义。"""
        return Tool(
            name="read_webpage",
            description=(
                "读取指定 URL 的网页内容。当你需要深入了解某篇文章、"
                "报告或页面的具体内容时，使用此工具。"
            ),
            parameters={
                "type": "object",
                "properties": {
                    "url": {
                        "type": "string",
                        "description": "要读取的网页 URL"
                    }
                },
                "required": ["url"]
            },
            function=self.read
        )
```

工具设计有几个值得注意的要点。第一，每个工具都封装为一个类，而不是零散的函数。这样可以方便地管理状态（如 HTTP 客户端）和配置。第二，`get_tool` 方法将工具转换为 OpenAI Function Calling 格式，这样 Agent 就能在对话中调用这些工具。第三，错误处理做得比较完善，网络请求失败不会导致整个 Agent 崩溃，而是返回一个包含错误信息的结果。

### 5.3.3 提示词设计

提示词是 Agent 的灵魂。一个好的提示词能够引导 Agent 按照正确的思路工作，而一个糟糕的提示词则会让 Agent 混乱无序。

```python
# prompts.py

SYSTEM_PROMPT = """你是一个专业的研究助手。你的任务是帮助用户深入研究他们感兴趣的话题，并生成一份高质量的研究报告。

## 你的工作流程

你需要按照以下步骤完成研究任务：

### 第一步：理解需求
仔细阅读用户的研究主题，明确以下几点：
- 研究的核心问题是什么？
- 需要覆盖哪些子话题？
- 有没有特殊的时间范围或行业限制？
- 用户期望的报告深度（概览 vs 深度分析）？

### 第二步：制定搜索策略
根据研究主题，制定一个系统的搜索计划。好的搜索策略应该：
- 从宽泛到具体，先搜索大方向，再深入细分领域
- 使用英文关键词搜索以获得更广泛的结果
- 同时搜索学术论文、行业报告、新闻动态等多种类型
- 考虑时间因素，优先获取最近1-2年的信息

### 第三步：执行搜索
使用 search_web 工具进行搜索，每次搜索后仔细阅读结果摘要，
判断哪些值得深入阅读。

### 第四步：深度阅读
使用 read_webpage 工具阅读高价值的网页，提取其中的关键信息，
包括数据、观点、案例和结论。

### 第五步：信息整理
将收集到的所有信息进行去重、分类和组织，形成清晰的知识框架。

### 第六步：生成报告
根据整理好的信息，生成结构清晰、论证充分的研究报告。

## 搜索技巧

1. **关键词要具体**：不要搜索"AI"，而是搜索"AI agent in financial services 2025"
2. **多角度搜索**：同一个话题用不同的关键词组合搜索
3. **关注权威来源**：McKinsey、Gartner、学术论文、行业头部公司的报告
4. **时效性优先**：AI 领域变化很快，优先获取最近6个月内的信息

## 报告格式要求

生成的研究报告应该包含以下部分：

1. **执行摘要**（200字以内）：报告的核心发现和建议
2. **研究背景**：为什么要做这个研究，研究的范围和方法
3. **现状分析**：基于收集到的信息，描述当前状况
4. **关键发现**：最重要的3-5个发现，每个附带数据支撑
5. **趋势预测**：基于现状分析，预测未来发展方向
6. **结论与建议**：总结性的结论和可操作的建议

## 重要原则

1. **事实优先**：所有结论必须有数据或案例支撑，不要凭空推测
2. **多源验证**：重要的事实至少要有两个独立来源的验证
3. **承认不确定性**：如果某个信息不够确定，明确标注
4. **保持客观**：不要在报告中表达个人偏好，保持中立客观的立场
"""

RESEARCH_START_PROMPT = """请帮我深入研究以下主题：

**研究主题：** {topic}

**额外要求：**
{requirements}

请按照你的研究流程，先制定搜索策略，然后逐步执行搜索和信息收集，
最后生成一份完整的研究报告。在过程中，请向我汇报关键发现。
"""

SEARCH_STRATEGY_PROMPT = """基于以下研究主题，请制定一个搜索策略：

**主题：** {topic}
**可用工具：** search_web（搜索互联网）、read_webpage（阅读网页内容）

请列出你计划执行的搜索关键词列表（建议5-8个），以及每个关键词的搜索目的。
格式要求：
1. 关键词1 - 搜索目的说明
2. 关键词2 - 搜索目的说明
...
"""

REPORT_GENERATION_PROMPT = """基于以下收集到的研究信息，请生成一份完整的研究报告。

**研究主题：** {topic}

**收集到的信息：**
{collected_info}

**要求：**
1. 按照标准研究报告格式组织
2. 所有结论必须有信息源支撑
3. 标注关键数据和引用来源
4. 语言专业、逻辑清晰
5. 总字数约 {target_length} 字
"""
```

提示词设计的关键在于"结构化"和"明确性"。注意系统提示词中明确列出了工作流程的六个步骤，这等于给 Agent 提供了一个标准操作程序（SOP）。Agent 不需要自己思考该怎么做，只需要按照这个流程执行就好了。这种设计大大减少了 Agent 出错的概率。

### 5.3.4 记忆管理

研究助手 Agent 的一个独特挑战是：研究过程可能涉及大量的信息，而 LLM 的上下文窗口是有限的。所以我们需要一个记忆管理模块来帮助 Agent 管理这些信息。

```python
# memory.py
import json
import time
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class SearchResult:
    """单条搜索结果的记忆。"""
    query: str              # 搜索关键词
    title: str              # 结果标题
    url: str                # 结果链接
    snippet: str            # 摘要
    timestamp: float = field(default_factory=time.time)
    is_read: bool = False   # 是否已深入阅读
    read_content: str = ""  # 阅读后的详细内容
    key_findings: list = field(default_factory=list)  # 提取的关键发现


@dataclass
class ResearchMemory:
    """研究助手的记忆系统。

    这个记忆系统管理三类信息：
    1. 研究计划：搜索策略和进度追踪
    2. 搜索历史：已执行的搜索和结果
    3. 知识库：整理后的关键发现和信息
    """

    # 研究主题
    topic: str = ""

    # 研究计划
    search_plan: list = field(default_factory=list)
    completed_searches: list = field(default_factory=list)

    # 搜索结果存储
    all_search_results: list[dict] = field(default_factory=list)
    read_pages: list[dict] = field(default_factory=list)

    # 知识库：经过整理的关键信息
    key_findings: list = str = field(default_factory=list)
    collected_evidence: list = field(default_factory=list)

    # 研究阶段追踪
    current_phase: str = "init"  # init, planning, searching, reading, synthesizing, reporting

    def add_search_result(self, query: str, results: list[dict]):
        """记录搜索结果。"""
        for result in results:
            self.all_search_results.append({
                "query": query,
                "title": result.get("title", ""),
                "url": result.get("url", ""),
                "snippet": result.get("snippet", ""),
                "timestamp": time.time()
            })
        self.completed_searches.append(query)

    def add_page_content(self, url: str, title: str, content: str, findings: str = ""):
        """记录已阅读的页面内容。"""
        self.read_pages.append({
            "url": url,
            "title": title,
            "content": content[:5000],  # 只存储前5000字符
            "findings": findings,
            "timestamp": time.time()
        })
        if findings:
            self.key_findings.append(findings)

    def get_search_summary(self) -> str:
        """生成搜索历史摘要。"""
        if not self.all_search_results:
            return "尚未执行任何搜索。"

        summary_lines = [f"已完成 {len(self.completed_searches)} 轮搜索，共获取 {len(self.all_search_results)} 条结果。\n"]
        summary_lines.append("搜索历史：")
        for i, query in enumerate(self.completed_searches, 1):
            result_count = sum(1 for r in self.all_search_results if r["query"] == query)
            summary_lines.append(f"  {i}. \"{query}\" -> {result_count} 条结果")
        return "\n".join(summary_lines)

    def get_findings_summary(self) -> str:
        """生成关键发现摘要。"""
        if not self.key_findings:
            return "尚未提取关键发现。"
        return "\n\n".join(self.key_findings)

    def get_context_for_llm(self) -> str:
        """生成适合发送给 LLM 的上下文摘要。

        这是记忆管理的核心方法。它将所有收集到的信息压缩成
        一个紧凑的摘要，控制在上下文窗口允许的范围内。
        """
        context_parts = []

        # 1. 研究主题和阶段
        context_parts.append(f"研究主题：{self.topic}")
        context_parts.append(f"当前阶段：{self.current_phase}")

        # 2. 搜索进度
        context_parts.append(f"\n{self.get_search_summary()}")

        # 3. 已阅读页面的发现
        if self.read_pages:
            context_parts.append("\n已阅读的信息来源：")
            for page in self.read_pages[-10:]:  # 只保留最近10条
                context_parts.append(f"\n【{page['title']}】")
                if page["findings"]:
                    context_parts.append(f"  发现：{page['findings']}")
                else:
                    context_parts.append(f"  内容摘要：{page['content'][:500]}...")

        # 4. 关键发现汇总
        if self.key_findings:
            context_parts.append(f"\n已收集的关键发现（共{len(self.key_findings)}条）：")
            for i, finding in enumerate(self.key_findings[-15:], 1):  # 只保留最近15条
                context_parts.append(f"  {i}. {finding[:300]}")

        return "\n".join(context_parts)

    def to_dict(self) -> dict:
        """序列化为字典，便于保存。"""
        return {
            "topic": self.topic,
            "search_plan": self.search_plan,
            "completed_searches": self.completed_searches,
            "all_search_results": self.all_search_results,
            "read_pages": self.read_pages,
            "key_findings": self.key_findings,
            "collected_evidence": self.collected_evidence,
            "current_phase": self.current_phase
        }

    @classmethod
    def from_dict(cls, data: dict) -> "ResearchMemory":
        """从字典恢复。"""
        memory = cls()
        memory.topic = data.get("topic", "")
        memory.search_plan = data.get("search_plan", [])
        memory.completed_searches = data.get("completed_searches", [])
        memory.all_search_results = data.get("all_search_results", [])
        memory.read_pages = data.get("read_pages", [])
        memory.key_findings = data.get("key_findings", [])
        memory.collected_evidence = data.get("collected_evidence", [])
        memory.current_phase = data.get("current_phase", "init")
        return memory
```

记忆管理系统的设计体现了"有限上下文窗口"这个 Agent 工程的核心约束。LLM 的上下文窗口虽然在不断扩大（GPT-4o 支持 128K tokens），但在一个复杂的研究任务中，收集到的信息量很容易超过这个限制。所以我们需要一个机制来决定"保留什么信息、丢弃什么信息"。

我们的策略是：保留搜索历史摘要（低成本高价值）、保留已阅读页面的关键发现（而非完整内容）、保留最近N条详细信息（利用时间局部性）。

### 5.3.5 Agent 核心逻辑

现在我们来实现 Agent 的核心逻辑。这是整个项目最复杂的部分，它把之前定义的所有组件串联起来。

```python
# agent.py
import json
import time
from typing import Optional
from openai import OpenAI
from config import ResearchConfig
from tools import SearchTool, WebpageReaderTool, Tool
from memory import ResearchMemory
from prompts import SYSTEM_PROMPT, RESEARCH_START_PROMPT, REPORT_GENERATION_PROMPT


class ResearchAgent:
    """研究助手 Agent 的核心类。

    这个 Agent 使用 ReAct（Reasoning + Acting）范式：
    1. 观察当前状态（记忆中的信息）
    2. 思考下一步该做什么（LLM 推理）
    3. 执行动作（调用工具）
    4. 将结果写入记忆
    5. 重复以上步骤直到完成
    """

    def __init__(self, config: ResearchConfig):
        self.config = config
        self.client = OpenAI(
            api_key=config.openai_api_key,
            base_url=config.openai_base_url
        )
        self.memory = ResearchMemory()
        self.tools: list[Tool] = []
        self.tool_map: dict[str, callable] = {}
        self.messages: list[dict] = []

        # 初始化工具
        self._init_tools()

        # 初始化对话
        self._init_conversation()

    def _init_tools(self):
        """初始化所有可用工具。"""
        search_tool = SearchTool(provider=self.config.search_provider)
        reader_tool = WebpageReaderTool(max_content_length=self.config.max_content_length)

        self.tools = [
            search_tool.get_tool(),
            reader_tool.get_tool(),
        ]

        self.tool_map = {
            "search_web": search_tool.search,
            "read_webpage": reader_tool.read,
        }

        if self.config.verbose:
            print(f"[工具初始化] 已加载 {len(self.tools)} 个工具: "
                  f"{', '.join(t.name for t in self.tools)}")

    def _init_conversation(self):
        """初始化对话历史。"""
        self.messages = [
            {"role": "system", "content": SYSTEM_PROMPT}
        ]

    def _call_llm(self, include_tools: bool = True) -> dict:
        """调用 LLM 并返回响应。

        这个方法处理了 OpenAI API 的所有细节，包括：
        - 工具定义
        - 错误处理
        - 响应解析
        """
        kwargs = {
            "model": self.config.model_name,
            "messages": self.messages,
            "temperature": self.config.temperature,
            "max_tokens": 4096,
        }

        if include_tools and self.tools:
            kwargs["tools"] = [t.to_openai_format() for t in self.tools]
            kwargs["tool_choice"] = "auto"

        try:
            response = self.client.chat.completions.create(**kwargs)
            message = response.choices[0].message

            result = {
                "content": message.content or "",
                "tool_calls": []
            }

            if message.tool_calls:
                for tc in message.tool_calls:
                    result["tool_calls"].append({
                        "id": tc.id,
                        "function_name": tc.function.name,
                        "arguments": json.loads(tc.function.arguments)
                    })

            return result

        except Exception as e:
            print(f"[LLM 调用失败] {e}")
            return {"content": f"抱歉，调用模型时出现错误: {e}", "tool_calls": []}

    def _execute_tool(self, tool_call: dict) -> str:
        """执行一个工具调用并返回结果。

        这个方法负责：
        1. 根据函数名找到对应的工具
        2. 调用工具执行操作
        3. 将结果记录到记忆中
        4. 返回格式化后的结果字符串
        """
        func_name = tool_call["function_name"]
        args = tool_call["arguments"]

        if self.config.verbose:
            print(f"\n[工具调用] {func_name}({json.dumps(args, ensure_ascii=False)[:200]})")

        # 执行工具
        if func_name in self.tool_map:
            try:
                raw_result = self.tool_map[func_name](**args)
            except Exception as e:
                raw_result = {"error": str(e)}
        else:
            raw_result = {"error": f"未知工具: {func_name}"}

        # 记录到记忆
        if func_name == "search_web":
            query = args.get("query", "")
            if isinstance(raw_result, list):
                self.memory.add_search_result(query, raw_result)
                result_str = json.dumps(raw_result, ensure_ascii=False, indent=2)
            else:
                result_str = str(raw_result)
        elif func_name == "read_webpage":
            url = args.get("url", "")
            if isinstance(raw_result, dict):
                self.memory.add_page_content(
                    url=url,
                    title=raw_result.get("title", ""),
                    content=raw_result.get("content", "")
                )
                # 只返回摘要给 LLM，避免上下文爆炸
                result_str = (
                    f"标题: {raw_result.get('title', '')}\n"
                    f"内容摘要: {raw_result.get('content', '')[:3000]}\n"
                    f"总长度: {raw_result.get('length', 0)} 字符"
                )
            else:
                result_str = str(raw_result)
        else:
            result_str = json.dumps(raw_result, ensure_ascii=False)

        if self.config.verbose:
            preview = result_str[:300] + "..." if len(result_str) > 300 else result_str
            print(f"[工具结果] {preview}")

        return result_str

    def run(self, topic: str, requirements: str = "无额外要求") -> str:
        """运行研究助手 Agent。

        这是 Agent 的主循环。它不断执行"思考->行动->观察"的循环，
        直到 Agent 认为研究已经完成并生成最终报告。

        Args:
            topic: 研究主题
            requirements: 额外的研究要求

        Returns:
            最终生成的研究报告
        """
        self.memory.topic = topic
        print(f"\n{'='*60}")
        print(f"  研究助手 Agent 启动")
        print(f"  研究主题：{topic}")
        print(f"{'='*60}\n")

        # 第一步：发送研究请求
        user_message = RESEARCH_START_PROMPT.format(
            topic=topic,
            requirements=requirements
        )
        self.messages.append({"role": "user", "content": user_message})

        # 主循环
        for iteration in range(self.config.max_iterations):
            print(f"\n--- 第 {iteration + 1} 轮迭代 ---")

            # 调用 LLM
            response = self._call_llm(include_tools=True)

            # 如果有文本内容，显示给用户
            if response["content"]:
                print(f"\n[Agent 思考] {response['content'][:500]}...")
                self.messages.append({"role": "assistant", "content": response["content"]})

            # 如果没有工具调用，说明 Agent 认为任务完成了
            if not response["tool_calls"]:
                print("\n[研究完成] Agent 生成了最终报告。")
                return response["content"]

            # 执行所有工具调用
            for tool_call in response["tool_calls"]:
                tool_result = self._execute_tool(tool_call)

                # 将工具结果添加到对话历史
                # 注意：OpenAI 要求 tool_call_id 和 role=tool
                self.messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call["id"],
                    "content": tool_result
                })

            # 每隔几轮，注入记忆摘要帮助 Agent 追踪进度
            if (iteration + 1) % 3 == 0:
                memory_context = self.memory.get_context_for_llm()
                self.messages.append({
                    "role": "system",
                    "content": f"[研究进度更新]\n{memory_context}\n\n请根据以上进度继续研究。"
                })

        # 如果达到最大迭代次数
        print("\n[警告] 达到最大迭代次数，强制结束。")
        final_response = self._call_llm(include_tools=False)
        return final_response["content"]
```

Agent 核心逻辑的几个设计要点值得深入讨论。第一，主循环的设计。Agent 的运行是一个"感知-思考-行动"的循环，每次迭代中，LLM 先思考当前应该做什么，然后决定是调用工具还是输出最终结果。当 LLM 不再调用工具时，我们就认为它已经完成了任务。

第二，记忆注入的时机。每隔三轮迭代，我们会把记忆摘要注入到对话历史中。这是因为在长时间的对话中，LLM 容易"忘记"之前收集到的信息。定期注入记忆可以帮助它保持全局视角。

第三，工具结果的处理。对于搜索工具，我们把完整结果都保存到记忆中，但只把摘要发给 LLM。对于网页阅读工具，我们也只发送前 3000 字符。这是控制上下文长度的重要手段。

### 5.3.6 报告生成器

最后一个模块是报告生成器。它的职责是把 Agent 在研究过程中收集到的所有信息，整理成一份格式规范、内容完整的研究报告。

```python
# report.py
import json
from datetime import datetime
from typing import Optional
from memory import ResearchMemory


class ReportGenerator:
    """研究报告生成器。

    这个类负责将研究记忆中的信息转化为格式化的研究报告。
    它支持 Markdown 和纯文本两种输出格式。
    """

    def __init__(self):
        self.timestamp = datetime.now().strftime("%Y-%m-%d %H:%M")

    def generate_from_memory(self, memory: ResearchMemory, format: str = "markdown") -> str:
        """根据研究记忆生成报告。

        Args:
            memory: 研究记忆对象
            format: 输出格式，支持 "markdown" 或 "text"

        Returns:
            格式化的研究报告字符串
        """
        if format == "markdown":
            return self._generate_markdown(memory)
        else:
            return self._generate_text(memory)

    def _generate_markdown(self, memory: ResearchMemory) -> str:
        """生成 Markdown 格式的报告。"""
        sections = []

        # 报告头部
        sections.append(f"# 研究报告：{memory.topic}")
        sections.append(f"\n> 生成时间：{self.timestamp}")
        sections.append(f"> 研究阶段：{memory.current_phase}")
        sections.append(f"> 信息来源数量：{len(memory.read_pages)}")
        sections.append(f"> 搜索轮次：{len(memory.completed_searches)}")

        # 研究方法说明
        sections.append("\n---\n")
        sections.append("## 研究方法")
        sections.append("\n本报告基于以下研究方法生成：")
        sections.append(f"- 进行了 {len(memory.completed_searches)} 轮互联网搜索")
        sections.append(f"- 深入阅读了 {len(memory.read_pages)} 个信息来源")
        sections.append(f"- 提取了 {len(memory.key_findings)} 条关键发现")

        # 搜索历史
        if memory.completed_searches:
            sections.append("\n### 搜索关键词")
            for query in memory.completed_searches:
                sections.append(f"- {query}")

        # 关键发现
        if memory.key_findings:
            sections.append("\n---\n")
            sections.append("## 关键发现")
            for i, finding in enumerate(memory.key_findings, 1):
                sections.append(f"\n### 发现 {i}")
                sections.append(f"\n{finding}")

        # 信息来源
        if memory.read_pages:
            sections.append("\n---\n")
            sections.append("## 参考来源")
            for i, page in enumerate(memory.read_pages, 1):
                sections.append(f"\n{i}. [{page['title']}]({page['url']})")

        # 研究元数据
        sections.append("\n---\n")
        sections.append("## 研究元数据")
        sections.append(f"\n- 总搜索结果数：{len(memory.all_search_results)}")
        sections.append(f"- 已阅读页面数：{len(memory.read_pages)}")
        sections.append(f"- 报告生成时间：{self.timestamp}")

        return "\n".join(sections)

    def _generate_text(self, memory: ResearchMemory) -> str:
        """生成纯文本格式的报告。"""
        sections = []

        sections.append(f"研究报告：{memory.topic}")
        sections.append(f"生成时间：{self.timestamp}")
        sections.append("=" * 60)

        if memory.key_findings:
            sections.append("\n关键发现：")
            for i, finding in enumerate(memory.key_findings, 1):
                sections.append(f"\n{i}. {finding}")

        if memory.read_pages:
            sections.append("\n\n参考来源：")
            for i, page in enumerate(memory.read_pages, 1):
                sections.append(f"{i}. {page['title']} - {page['url']}")

        return "\n".join(sections)

    def save_report(self, content: str, filename: str, output_dir: str = "output"):
        """保存报告到文件。"""
        import os
        os.makedirs(output_dir, exist_ok=True)
        filepath = os.path.join(output_dir, filename)
        with open(filepath, "w", encoding="utf-8") as f:
            f.write(content)
        print(f"[报告已保存] {filepath}")
        return filepath
```

### 5.3.7 入口文件

最后，我们把所有模块组装到入口文件中：

```python
# main.py
import sys
import os
from config import load_config
from agent import ResearchAgent
from report import ReportGenerator

def main():
    """研究助手 Agent 的主入口函数。"""
    print("=" * 60)
    print("  研究助手 Agent v1.0")
    print("  帮你高效完成研究任务")
    print("=" * 60)

    # 检查 API Key
    if not os.environ.get("OPENAI_API_KEY"):
        print("\n[错误] 请先设置 OPENAI_API_KEY 环境变量。")
        print("  Windows PowerShell: $env:OPENAI_API_KEY = 'your-key-here'")
        print("  Linux/Mac: export OPENAI_API_KEY='your-key-here'")
        return

    # 加载配置
    config = load_config(
        verbose=True,
        max_search_rounds=3,
        max_pages_to_read=5,
    )

    # 获取用户输入
    if len(sys.argv) > 1:
        topic = " ".join(sys.argv[1:])
    else:
        topic = input("\n请输入研究主题：").strip()
        if not topic:
            print("研究主题不能为空。")
            return

    # 额外要求
    requirements = input("请输入额外要求（可选，直接回车跳过）：").strip()
    if not requirements:
        requirements = "无额外要求"

    # 创建并运行 Agent
    agent = ResearchAgent(config)
    report = agent.run(topic, requirements)

    # 生成并保存报告
    generator = ReportGenerator()
    md_report = generator.generate_from_memory(agent.memory, format="markdown")
    text_report = generator.generate_from_memory(agent.memory, format="text")

    # 保存文件
    filename_base = topic.replace(" ", "_")[:50]
    generator.save_report(md_report, f"{filename_base}_report.md")
    generator.save_report(text_report, f"{filename_base}_report.txt")

    # 也保存记忆（便于后续分析和调试）
    import json
    os.makedirs("output", exist_ok=True)
    memory_file = os.path.join("output", f"{filename_base}_memory.json")
    with open(memory_file, "w", encoding="utf-8") as f:
        json.dump(agent.memory.to_dict(), f, ensure_ascii=False, indent=2)
    print(f"[记忆已保存] {memory_file}")

    # 显示最终报告
    print("\n" + "=" * 60)
    print("  最终研究报告")
    print("=" * 60)
    print(report)


if __name__ == "__main__":
    main()
```

---

## 5.4 运行指南

### 5.4.1 环境准备

在运行项目之前，你需要准备以下环境：

```bash
# 1. 创建虚拟环境
python -m venv research-agent-env

# 2. 激活虚拟环境
# Windows:
research-agent-env\Scripts\activate
# Linux/Mac:
source research-agent-env/bin/activate

# 3. 安装依赖
pip install openai httpx duckduckgo-search

# 4. 设置 API Key
# Windows PowerShell:
$env:OPENAI_API_KEY = "your-api-key-here"
# Linux/Mac:
export OPENAI_API_KEY="your-api-key-here"
```

### 5.4.2 启动运行

```bash
# 方式一：交互式运行
python main.py

# 方式二：直接指定主题
python main.py "AI Agent 在金融行业的应用现状"

# 方式三：使用中文主题
python main.py "大模型在医疗领域的最新进展"
```

### 5.4.3 输出说明

运行后，你会在 `output/` 目录下看到三个文件：

- `*_report.md`：Markdown 格式的研究报告，可以直接在 GitHub 或 Typora 中查看
- `*_report.txt`：纯文本格式的研究报告，适合进一步编辑
- `*_memory.json`：研究过程的完整记忆数据，可以用于调试和分析

---

## 5.5 案例分析

### 5.5.1 案例一：研究"AI Agent 的商业化路径"

我们用这个 Agent 来研究一个热门话题："AI Agent 的商业化路径"。

Agent 首先制定了搜索策略，计划搜索以下关键词：
1. "AI agent commercial applications 2025" - 了解已有的商业案例
2. "AI agent startup funding trends" - 分析投资趋势
3. "enterprise AI agent adoption" - 了解企业采用情况
4. "AI agent revenue model" - 分析商业模式

经过三轮搜索和多轮网页阅读，Agent 收集了来自 McKinsey、CB Insights、Y Combinator 等权威来源的信息。最终生成的报告包含：

- 执行摘要：AI Agent 正处于从概念验证到规模化部署的关键转折期
- 市场规模数据：引用了多家研究机构的预测数据
- 三大商业模式：SaaS 订阅、API 按量计费、企业定制
- 典型案例：Stripe、Jasper、Harvey 等公司的实践
- 风险与挑战：成本控制、可靠性、数据隐私
- 未来展望：2026-2028 年的发展预测

这个案例展示了 Agent 的一个典型工作流程：从宽泛到具体，从搜索到阅读，从信息到知识。

### 5.5.2 案例二：研究"Python 异步编程最佳实践"

对于技术类话题，Agent 同样表现出色。在研究"Python 异步编程最佳实践"时，Agent 搜索了 Python 官方文档、Real Python 教程、PyCon 演讲等技术资源，最终生成了一份包含 asyncio 核心概念、常见陷阱、性能优化技巧的实用指南。

这个案例说明，研究助手 Agent 不仅适用于商业和市场研究，也适用于技术学习和知识整理。

---

## 5.6 常见坑与解决方案

### 坑一：搜索结果质量参差不齐

**问题描述：** 互联网搜索返回的结果中，很多是低质量的营销内容或过时信息。

**解决方案：** 在搜索策略中增加"权威来源优先"的指令。例如，提示词中可以增加：
```
搜索时优先关注以下来源：
- 学术机构（.edu, .ac.uk）
- 行业研究机构（McKinsey, Gartner, CB Insights）
- 权威媒体（TechCrunch, Wired, MIT Technology Review）
- 官方文档和博客
```

### 坑二：上下文窗口溢出

**问题描述：** 当 Agent 阅读了大量网页后，上下文窗口可能会被塞满，导致 LLM 无法正常工作。

**解决方案：** 记忆管理系统已经处理了这个问题。关键策略包括：只保留摘要而非完整内容、定期压缩历史信息、控制每轮工具返回的文本量。如果你仍然遇到这个问题，可以进一步减小 `max_content_length` 参数。

### 坑三：Agent 陷入搜索循环

**问题描述：** Agent 反复执行相同的搜索，浪费时间和 API 调用额度。

**解决方案：** 在记忆系统中追踪已执行的搜索查询，并在提示词中告诉 Agent 避免重复搜索。如果检测到 Agent 正在循环，可以通过注入系统消息来干预：
```
系统提示：你已经搜索了以下关键词，请避免重复搜索，转向深入阅读已获取的结果。
```

### 坑四：报告质量不理想

**问题描述：** 生成的报告内容单薄、缺乏深度。

**解决方案：** 这通常是因为 Agent 没有足够深入地阅读信息源。解决方法是增加 `max_pages_to_read` 参数，并在提示词中强调"深度阅读"的重要性。另外，确保搜索策略覆盖了多种类型的信息源（新闻、论文、报告、博客等）。

### 坑五：API 调用成本过高

**问题描述：** 每次研究任务需要大量 LLM 调用，成本可能很高。

**解决方案：** 可以考虑以下几种策略来降低成本：使用 GPT-4o-mini 替代 GPT-4o（在不需要深度推理的步骤中）、减少最大迭代次数、缓存搜索结果（同一关键词的搜索结果在短时间内不会有大变化）、使用本地开源模型替代商业 API。

---

## 5.7 练习题

### 练习一：扩展搜索工具（难度：初级）

当前项目只支持 DuckDuckGo 搜索。请为项目添加一个新的搜索工具，支持使用 Tavily Search API。Tavily 专门为 AI 应用设计，返回的结果更适合 LLM 处理。

要求：
1. 在 `tools.py` 中创建 `TavilySearchTool` 类
2. 支持通过环境变量配置 API Key
3. 返回结果中包含"相关度评分"
4. 在 `config.py` 中添加对应的配置项

### 练习二：添加引用管理功能（难度：中级）

当前项目的信息来源管理比较简单。请为项目添加一个引用管理功能：

要求：
1. 创建 `citation.py` 模块，实现引用管理器
2. 每个引用包含：作者、标题、URL、访问日期、摘要
3. 在报告中自动生成参考文献列表
4. 支持 APA 和 MLA 两种引用格式
5. 能检测并合并重复的引用

### 练习三：实现增量研究（难度：中级）

当前项目每次运行都是从零开始。请实现增量研究功能：

要求：
1. 保存研究记忆到文件
2. 下次运行时可以加载之前的研究记忆，继续研究
3. 自动避免重复搜索已经搜过的内容
4. 合并新旧两轮研究的发现

### 练习四：添加可视化功能（难度：高级）

请为研究报告添加可视化元素：

要求：
1. 使用 matplotlib 或 plotly 生成数据图表
2. 生成研究主题的关键词云
3. 生成信息来源的权威性评分图
4. 将图表嵌入 Markdown 报告中

### 练习五：实现并行搜索（难度：高级）

当前项目的搜索是串行执行的。请实现并行搜索功能：

要求：
1. 使用 asyncio 或 concurrent.futures 实现并行搜索
2. 同时发起多个搜索请求，提高研究效率
3. 处理并发控制，避免触发 API 限流
4. 测量并行搜索相比串行搜索的速度提升

### 练习六：添加研究评审功能（难度：高级）

请为项目添加一个自动评审功能，由另一个 Agent 来评审研究报告的质量：

要求：
1. 创建 `reviewer.py` 模块
2. 评审维度包括：信息准确性、论证逻辑性、来源可靠性、报告完整性
3. 生成评审意见和改进建议
4. 支持迭代改进：根据评审意见自动修改报告

---

## 5.8 实战任务

现在是动手实践的时候了。请完成以下实战任务，将本章学到的知识转化为你的实际能力。

### 任务一：运行完整项目

按照 5.4 节的指南，在你的电脑上成功运行研究助手 Agent。选择一个你感兴趣的研究主题（建议选一个你熟悉的话题，这样你能判断 Agent 的输出质量），观察它的完整工作流程。记录下运行过程中遇到的任何问题，并尝试解决。

### 任务二：优化搜索策略

尝试修改 `prompts.py` 中的搜索策略提示词，让 Agent 在搜索时更加智能。例如：
- 增加"搜索不同时间范围"的指令
- 增加"使用不同语言搜索"的指令
- 增加"针对特定行业调整搜索关键词"的指令

比较修改前后的报告质量差异。

### 任务三：构建自己的工具

为研究助手 Agent 添加至少两个新工具。建议的工具包括：
- 学术论文搜索工具（使用 Semantic Scholar API）
- Twitter/X 搜索工具（搜索社交媒体上的讨论）
- 数据图表生成工具（将数据转化为可视化图表）

### 任务四：对比不同模型的效果

使用不同的 LLM 模型运行同一个研究主题，比较它们的输出质量。例如：
- GPT-4o vs GPT-4o-mini
- Claude 3.5 Sonnet vs GPT-4o
- 本地开源模型 vs 商业 API

记录并分析不同模型在以下方面的差异：搜索关键词生成质量、信息筛选准确度、报告组织逻辑性、整体研究深度。

---

## 5.9 本章小结

在本章中，我们从零开始构建了一个完整的研究助手 Agent。回顾整个过程，有几个关键点值得记住。

第一，**Agent 的能力由工具决定**。搜索工具和网页阅读工具是研究助手的核心能力边界。没有这些工具，LLM 只能依赖训练数据中的知识，无法获取互联网上的最新信息。选择和设计好的工具，是构建有效 Agent 的第一步。

第二，**记忆管理是复杂 Agent 的关键挑战**。研究助手需要处理大量的信息，而 LLM 的上下文窗口是有限的。通过设计合理的记忆系统，我们能够在有限的上下文窗口内管理无限的信息量。这个思路适用于所有需要处理大量数据的 Agent 场景。

第三，**提示词的结构化至关重要**。一个精心设计的系统提示词能够将复杂的任务分解为清晰的步骤，大大减少 Agent 出错的概率。好的提示词就像一份详尽的操作手册，让 Agent 按部就班地完成任务。

第四，**工程实践不可忽视**。配置管理、错误处理、日志记录、文件保存——这些看似琐碎的工程细节，实际上决定了项目能否真正投入使用。一个只有核心逻辑但缺乏工程支撑的项目，只能算是一个 Demo，而不是一个产品。

下一章，我们将构建另一个实战项目——PDF 分析 Agent。如果说研究助手 Agent 解决的是"获取信息"的问题，那么 PDF 分析 Agent 解决的就是"理解信息"的问题。两者的结合，将构成一个更强大的知识处理系统。
