# 第 32 章：LangSmith 与追踪实践

> **本章定位：** 上一章我们从零构建了 Agent 可观测性的基础框架。但在实际生产中，自己从头造轮子既耗时又难以维护。LangSmith 是 LangChain 团队推出的 LLM 应用可观测性平台，它提供了开箱即用的追踪、评估和调试能力。本章将带你从注册使用到深度集成，全面掌握 LangSmith 的核心功能。

---

## 学习目标

完成本章学习后，你将能够：

1. **理解 LangSmith 的核心概念** —— 能清晰区分 Run、Trace、Dataset、Experiment 等核心抽象
2. **将 LangSmith 集成到 Agent 项目中** —— 能用最少的代码改动让 Agent 的所有行为自动被追踪
3. **使用 LangSmith 调试 Agent 问题** —— 能通过 LangSmith 的 UI 定位 Agent 的推理错误和性能瓶颈
4. **构建评估数据集** —— 能创建和管理用于评估 Agent 质量的测试数据集
5. **运行自动化评估** —— 能用 LangSmith 的评估框架批量测试 Agent 的表现

## 核心问题

1. **LangSmith 和上一章自己搭建的追踪系统相比，核心优势是什么？** 为什么生产环境通常会选择专业平台而不是自建方案？
2. **如何用最小的代码改动把 LangSmith 接入已有的 Agent 项目？** 接入的过程会遇到哪些常见问题？
3. **LangSmith 的评估功能如何帮助你持续改进 Agent？** 它和传统的测试方法有什么本质区别？

---

## 32.1 LangSmith 概览

### 32.1.1 LangSmith 是什么

LangSmith 是由 LangChain 团队开发的一个全方位的 LLM 应用开发平台。你可以把它想象成 LLM 世界的"Datadog + GitHub"——它不仅提供运行时的追踪和监控能力，还提供离线的评估、测试和版本管理功能。

LangSmith 的核心价值主张是：让你能够"看见"LLM 应用的内部运作。在上一章中，我们自己实现了追踪器、指标收集器和监控面板。LangSmith 把这些功能（以及更多）打包成了一个云服务，你只需要几行代码就能获得比自建方案强大得多的可观测性能力。

### 32.1.2 核心概念

在开始使用 LangSmith 之前，你需要理解以下几个核心概念，它们构成了 LangSmith 的基础抽象模型。

**Project（项目）** 是 LangSmith 中最高层级的组织单元。一个 Project 对应一个 LLM 应用或 Agent。你在 LangSmith 上创建的所有追踪数据、评估结果、数据集都归属于某个 Project。建议按应用的粒度来创建 Project：一个客服 Agent 是一个 Project，一个代码生成 Agent 是另一个 Project。

**Run（运行）** 是 LangSmith 中最基本的追踪单元。每一次 LLM 调用、工具执行或链式调用都会生成一个 Run。Run 记录了输入、输出、耗时、Token 消耗、错误信息等所有关键数据。

**Trace（追踪）** 是一组相关 Run 的集合，它们共同完成一个用户请求。一个 Trace 从接收用户输入开始，到返回最终回答结束。在 Trace 内部，你可以看到 Agent 的每一步推理、每一次 LLM 调用、每一次工具执行，以及它们之间的父子关系。

**Dataset（数据集）** 是用于评估 LLM 应用质量的测试数据集合。每个数据集包含多条测试用例（Examples），每条用例包含输入和预期输出（或参考答案）。LangSmith 支持动态生成测试用例，也支持手动创建。

**Experiment（实验）** 是对一个 Dataset 的一次评估运行。你选择一个模型或 Agent 配置，对 Dataset 中的每条用例运行一次，LangSmith 会自动比较输出和参考答案，给出评分和对比结果。

### 32.1.3 LangSmith vs 自建方案

我们来客观对比一下 LangSmith 和上一章自建的追踪方案：

**开发速度：** LangSmith 只需要几行代码就能接入，自建方案需要从头实现追踪器、存储层、查询接口。如果你需要快速验证想法，LangSmith 明显更快。

**功能深度：** LangSmith 提供了可视化的 Trace 时间线、Token 消耗分析、错误模式识别、自动评估等高级功能。自建方案要达到同等水平需要大量额外开发。

**数据所有权：** 这是自建方案的优势。使用 LangSmith 意味着你的追踪数据存储在 LangChain 的服务器上。对于涉及敏感数据的企业场景，这可能是个合规问题。LangSmith 提供了自托管（Self-hosted）方案来解决这个问题，但需要额外的部署和运维成本。

**成本：** LangSmith 有免费额度，但高频使用需要付费。自建方案的基础设施成本可能更低，但需要计算人力成本。

**定制性：** 自建方案完全可控，可以按需定制。LangSmith 的定制性依赖于它的 API 和配置选项，虽然已经很丰富，但仍有边界。

对于大多数团队来说，建议的路径是：原型阶段用 LangSmith 快速迭代，等到应用成熟后再评估是否需要迁移到自托管或自建方案。

---

## 32.2 LangSmith 环境配置

### 32.2.1 注册和获取 API Key

第一步是在 LangSmith 官网注册账号并获取 API Key。你需要准备一个邮箱地址，注册过程大约需要 2 分钟。

注册完成后，你可以在 LangSmith 的设置页面找到 API Key。这个 Key 用于认证你的代码和 LangSmith 服务之间的通信。

### 32.2.2 环境变量配置

LangSmith 通过环境变量来配置。创建一个 `.env` 文件来管理这些配置：

```env
# LangSmith 配置
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=your_langsmith_api_key_here
LANGCHAIN_PROJECT=my-agent-project
LANGCHAIN_ENDPOINT=https://api.smith.langchain.com

# LLM API Key（根据你使用的模型）
OPENAI_API_KEY=your_openai_api_key_here
```

在 Python 代码中加载这些环境变量：

```python
import os
from dotenv import load_dotenv

load_dotenv()

# 验证配置
print(f"LangSmith 追踪: {os.environ.get('LANGCHAIN_TRACING_V2', '未启用')}")
print(f"LangSmith 项目: {os.environ.get('LANGCHAIN_PROJECT', '未设置')}")
```

### 32.2.3 安装依赖

```bash
# 安装 LangChain 和 LangSmith
pip install langchain langchain-openai langsmith

# 如果你使用 LangGraph 搭建 Agent
pip install langgraph
```

### 32.2.4 验证连接

在写任何代码之前，先验证 LangSmith 连接是否正常：

```python
from langsmith import Client

client = Client()
try:
    # 尝试列出项目
    projects = list(client.list_projects(limit=5))
    print(f"连接成功！当前有 {len(projects)} 个项目")
    for p in projects:
        print(f"  - {p.name}")
except Exception as e:
    print(f"连接失败: {e}")
    print("请检查 LANGCHAIN_API_KEY 是否正确")
```

---

## 32.3 LangSmith 追踪实战

### 32.3.1 最简追踪：自动追踪 LangChain 调用

LangSmith 最强大的特性之一是它的"零侵入"追踪。只要设置了环境变量，所有 LangChain 的调用都会自动被追踪。

```python
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, SystemMessage
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent

load_dotenv()

# ============================================================
# 1. 基础 LLM 调用追踪
# ============================================================

def basic_llm_trace():
    """
    最简单的追踪示例。
    只要设置了 LANGCHAIN_TRACING_V2=true，每次 LLM 调用都会自动被追踪。
    """
    llm = ChatOpenAI(
        model="gpt-4o",
        temperature=0.7,
    )
    
    # 这次调用会自动出现在 LangSmith 的追踪面板中
    response = llm.invoke([
        SystemMessage(content="你是一个有帮助的助手。"),
        HumanMessage(content="用一句话解释什么是可观测性。"),
    ])
    
    print(f"回答: {response.content}")
    print("\n-> 查看 LangSmith 面板，你应该能看到这次调用的完整追踪")


# ============================================================
# 2. 带工具调用的 Agent 追踪
# ============================================================

@tool
def search_knowledge_base(query: str) -> str:
    """搜索知识库，返回相关信息"""
    # 模拟知识库搜索
    knowledge = {
        "可观测性": "可观测性是指从系统外部输出推断内部状态的能力，包括日志、指标和追踪三大支柱。",
        "Agent": "Agent 是能够自主决策和执行任务的 AI 系统。",
        "LangSmith": "LangSmith 是 LangChain 团队开发的 LLM 应用可观测性平台。",
    }
    
    for key, value in knowledge.items():
        if key in query:
            return value
    return f"未找到关于 '{query}' 的相关信息"


@tool
def calculate(expression: str) -> str:
    """安全地计算数学表达式"""
    try:
        allowed = {"abs": abs, "round": round, "min": min, "max": max}
        result = eval(expression, {"__builtins__": {}}, allowed)
        return f"计算结果: {result}"
    except Exception as e:
        return f"计算错误: {e}"


def traced_agent():
    """
    一个使用 LangGraph 的完整 Agent，自动被 LangSmith 追踪。
    
    每一步都会被记录：
    - Agent 的推理过程
    - 工具调用的输入和输出
    - LLM 的 Token 消耗
    - 每一步的耗时
    """
    llm = ChatOpenAI(model="gpt-4o", temperature=0)
    tools = [search_knowledge_base, calculate]
    
    # 创建 ReAct Agent
    agent = create_react_agent(llm, tools)
    
    # 运行 Agent —— 所有行为自动被追踪
    result = agent.invoke({
        "messages": [
            {"role": "user", "content": "什么是可观测性？请用它计算一下 2+3 等于多少。"}
        ]
    })
    
    # 打印结果
    for message in result["messages"]:
        role = message.__class__.__name__
        content = message.content if hasattr(message, 'content') else str(message)
        print(f"[{role}] {content[:200]}")
    
    print("\n-> 在 LangSmith 面板中查看完整的 Agent 执行追踪")


if __name__ == "__main__":
    print("=" * 60)
    print("示例 1: 基础 LLM 调用追踪")
    print("=" * 60)
    basic_llm_trace()
    
    print("\n")
    print("=" * 60)
    print("示例 2: 带工具调用的 Agent 追踪")
    print("=" * 60)
    traced_agent()
```

### 32.3.2 自定义追踪：手动创建 Span

有时候你需要在自动追踪之外，手动添加额外的追踪信息。LangSmith 提供了装饰器和上下文管理器来实现这一点。

```python
import os
import time
from dotenv import load_dotenv
from langsmith import traceable, Client
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage

load_dotenv()

client = Client()

# ============================================================
# 使用 @traceable 装饰器追踪自定义函数
# ============================================================

@traceable(
    name="数据预处理",
    run_type="chain",
    tags=["preprocessing", "v2"],
    metadata={"version": "2.0", "stage": "input_processing"},
)
def preprocess_input(user_input: str) -> str:
    """预处理用户输入，添加追踪信息"""
    time.sleep(0.1)  # 模拟处理
    
    # LangSmith 会自动记录输入和输出
    cleaned = user_input.strip().lower()
    return cleaned


@traceable(
    name="上下文检索",
    run_type="retriever",
    tags=["retrieval", "rag"],
)
def retrieve_context(query: str, top_k: int = 3) -> list:
    """检索相关上下文，追踪检索过程"""
    # 模拟检索
    time.sleep(0.2)
    
    contexts = [
        f"上下文 {i}: 关于 '{query}' 的信息片段 {i}"
        for i in range(top_k)
    ]
    
    # 返回值会被 LangSmith 自动记录
    return contexts


@traceable(
    name="回答生成",
    run_type="llm",
    tags=["generation", "final"],
)
def generate_answer(query: str, contexts: list, llm) -> str:
    """基于检索到的上下文生成回答"""
    context_text = "\n".join(contexts)
    
    response = llm.invoke([
        {"role": "system", "content": f"基于以下上下文回答问题：\n{context_text}"},
        {"role": "user", "content": query},
    ])
    
    return response.content


# ============================================================
# 组合完整的 RAG 流程
# ============================================================

@traceable(
    name="RAG 完整流程",
    run_type="chain",
    tags=["rag", "pipeline", "production"],
    metadata={"pipeline_version": "1.0"},
)
def rag_pipeline(user_input: str) -> str:
    """
    完整的 RAG 流程，每一步都有追踪。
    
    在 LangSmith 面板中，你会看到一个层级化的追踪树：
    - RAG 完整流程 (chain)
      - 数据预处理 (chain)
      - 上下文检索 (retriever)
      - 回答生成 (llm)
    """
    # 步骤 1: 预处理
    cleaned_input = preprocess_input(user_input)
    
    # 步骤 2: 检索
    contexts = retrieve_context(cleaned_input, top_k=3)
    
    # 步骤 3: 生成
    llm = ChatOpenAI(model="gpt-4o", temperature=0.3)
    answer = generate_answer(cleaned_input, contexts, llm)
    
    return answer


# ============================================================
# 使用自定义元数据和标签
# ============================================================

@traceable(
    name="多轮对话处理",
    run_type="chain",
    tags=["conversation", "multi-turn"],
    # metadata 可以包含任意键值对
    metadata={
        "model": "gpt-4o",
        "max_turns": 10,
        "features": ["memory", "tools"],
    },
)
def handle_conversation(messages: list, llm) -> str:
    """处理多轮对话，追踪完整的对话历史"""
    response = llm.invoke(messages)
    return response.content


def run_custom_tracing_demo():
    """运行自定义追踪示例"""
    llm = ChatOpenAI(model="gpt-4o", temperature=0.7)
    
    # 示例 1: RAG 流程追踪
    print("=" * 60)
    print("RAG 流程追踪")
    print("=" * 60)
    answer = rag_pipeline("什么是 Agent 的可观观测性？")
    print(f"回答: {answer}\n")
    
    # 示例 2: 多轮对话追踪
    print("=" * 60)
    print("多轮对话追踪")
    print("=" * 60)
    messages = [
        {"role": "system", "content": "你是一个有帮助的助手。"},
        {"role": "user", "content": "你好，请自我介绍。"},
    ]
    response1 = handle_conversation(messages, llm)
    print(f"助手: {response1}")
    
    messages.append({"role": "assistant", "content": response1})
    messages.append({"role": "user", "content": "你能做什么？"})
    response2 = handle_conversation(messages, llm)
    print(f"助手: {response2}")
    
    print("\n-> 在 LangSmith 面板中查看追踪层级结构")


if __name__ == "__main__":
    run_custom_tracing_demo()
```

### 32.3.3 在 Agent 循环中追踪决策

对于复杂的 Agent，我们不仅想追踪 LLM 调用和工具执行，还想追踪 Agent 的决策逻辑——为什么它选择了这个工具？为什么它决定停止推理？

```python
import os
import time
import json
from dotenv import load_dotenv
from langsmith import traceable
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, SystemMessage, AIMessage, ToolMessage

load_dotenv()

class TracedAgent:
    """
    一个完全可观测的 Agent
    
    每个决策步骤都有独立的追踪，让你能在 LangSmith 中
    看到 Agent 的完整思考过程。
    """
    
    def __init__(self, model: str = "gpt-4o"):
        self.llm = ChatOpenAI(model=model, temperature=0)
        self.tools = {
            "search": self._search,
            "calculate": self._calculate,
            "summarize": self._summarize,
        }
    
    @traceable(name="搜索工具", run_type="tool", tags=["search"])
    def _search(self, query: str) -> str:
        time.sleep(0.1)
        return f"搜索结果: 关于 '{query}' 的相关信息..."
    
    @traceable(name="计算工具", run_type="tool", tags=["calculation"])
    def _calculate(self, expression: str) -> str:
        try:
            result = eval(expression, {"__builtins__": {}}, {})
            return f"计算结果: {expression} = {result}"
        except Exception as e:
            return f"计算错误: {e}"
    
    @traceable(name="摘要工具", run_type="tool", tags=["summarization"])
    def _summarize(self, text: str) -> str:
        time.sleep(0.1)
        return f"摘要: {text[:100]}..."
    
    @traceable(name="Agent 决策步骤", run_type="chain", tags=["decision"])
    def _decision_step(self, messages: list, step: int) -> dict:
        """
        Agent 的单步决策。
        追踪每个决策步骤的输入和输出。
        """
        response = self.llm.invoke(messages)
        
        # 分析 Agent 的决策
        decision = {
            "step": step,
            "response_type": "tool_call" if response.tool_calls else "final_answer",
            "content_preview": response.content[:200] if response.content else "N/A",
            "tool_calls": [
                {"name": tc["name"], "args": tc["args"]}
                for tc in (response.tool_calls or [])
            ],
        }
        
        return {"response": response, "decision": decision}
    
    @traceable(name="Agent 主循环", run_type="chain", tags=["agent", "main-loop"])
    def run(self, user_input: str, max_steps: int = 5) -> str:
        """
        Agent 的主循环。
        追踪整个推理过程，包括每一步的决策。
        """
        messages = [
            SystemMessage(content="你是一个智能助手。根据需要使用工具来帮助用户。"),
            HumanMessage(content=user_input),
        ]
        
        for step in range(max_steps):
            result = self._decision_step(messages, step)
            response = result["response"]
            decision = result["decision"]
            
            print(f"\n--- 步骤 {step + 1} ---")
            print(f"决策类型: {decision['response_type']}")
            if decision["tool_calls"]:
                for tc in decision["tool_calls"]:
                    print(f"工具调用: {tc['name']}({tc['args']})")
            
            # 如果没有工具调用，返回最终回答
            if not response.tool_calls:
                return response.content
            
            # 执行工具调用
            messages.append(AIMessage(content=response.content, tool_calls=response.tool_calls))
            
            for tool_call in response.tool_calls:
                tool_name = tool_call["name"]
                tool_args = tool_call["args"]
                
                if tool_name in self.tools:
                    tool_result = self.tools[tool_name](**tool_args)
                    messages.append(ToolMessage(
                        content=tool_result,
                        tool_call_id=tool_call["id"],
                    ))
                    print(f"工具结果: {tool_result[:100]}")
        
        return "达到最大步骤数，推理结束"


def main():
    agent = TracedAgent()
    
    print("=" * 60)
    print("Agent 决策追踪示例")
    print("=" * 60)
    
    answer = agent.run("帮我搜索一下 LangSmith 的功能，然后计算 15% 的折扣后价格是原价 200 元的多少")
    print(f"\n最终回答: {answer}")
    
    print("\n-> 在 LangSmith 面板中查看每个决策步骤")


if __name__ == "__main__":
    main()
```

---

## 32.4 LangSmith 评估体系

### 32.4.1 为什么需要评估

在上一章中，我们学会了追踪 Agent 的运行过程。追踪告诉你"Agent 做了什么"，但不告诉你"Agent 做得好不好"。评估（Evaluation）就是回答后一个问题的。

LangSmith 的评估体系允许你定义"什么是好的回答"，然后自动运行大量测试用例来验证你的 Agent 是否达标。这对于持续改进 Agent 质量至关重要。

### 32.4.2 创建评估数据集

```python
import os
from dotenv import load_dotenv
from langsmith import Client
from langsmith.evaluation import evaluate
from langsmith.schemas import Example, Run

load_dotenv()

client = Client()

# ============================================================
# 1. 创建评估数据集
# ============================================================

def create_evaluation_dataset():
    """创建一个用于评估 Agent 的数据集"""
    dataset_name = "agent-qa-evaluation-v1"
    
    # 检查数据集是否已存在
    existing = client.list_datasets(dataset_name=dataset_name)
    if existing:
        print(f"数据集 '{dataset_name}' 已存在，跳过创建")
        return dataset_name
    
    # 创建数据集
    dataset = client.create_dataset(
        dataset_name=dataset_name,
        description="用于评估 Agent 问答质量的测试数据集",
    )
    
    # 添加测试用例
    test_cases = [
        {
            "inputs": {"question": "什么是 LangSmith？"},
            "outputs": {"answer": "LangSmith 是 LangChain 团队开发的 LLM 应用开发平台，提供追踪、评估、调试等功能。"},
        },
        {
            "inputs": {"question": "Agent 和 Chatbot 有什么区别？"},
            "outputs": {"answer": "Agent 能够自主决策和使用工具，而 Chatbot 通常只能进行预定义的对话。"},
        },
        {
            "inputs": {"question": "如何减少 LLM 的幻觉？"},
            "outputs": {"answer": "可以通过 RAG 检索增强、温度调低、提示工程、后处理验证等方式减少幻觉。"},
        },
        {
            "inputs": {"question": "解释一下 ReAct 框架"},
            "outputs": {"answer": "ReAct 是一种 Agent 框架，结合了推理（Reasoning）和行动（Acting），让 LLM 交替进行思考和工具调用。"},
        },
        {
            "inputs": {"question": "什么是 Prompt Engineering？"},
            "outputs": {"answer": "Prompt Engineering 是设计和优化 LLM 输入提示的技术，目的是引导模型生成更好的输出。"},
        },
    ]
    
    for case in test_cases:
        client.create_example(
            dataset_id=dataset.id,
            inputs=case["inputs"],
            outputs=case["outputs"],
        )
    
    print(f"创建了包含 {len(test_cases)} 条用例的数据集 '{dataset_name}'")
    return dataset_name


# ============================================================
# 2. 定义评估函数
# ============================================================

def correctness_evaluator(run: Run, example: Example) -> dict:
    """
    评估 Agent 回答的正确性
    
    在实际场景中，你可能会用另一个 LLM 来评估回答质量。
    这里我们用简单的关键词匹配来演示。
    """
    agent_answer = run.outputs.get("answer", "").lower()
    reference_answer = example.outputs.get("answer", "").lower()
    
    # 提取关键词进行匹配
    ref_keywords = set(reference_answer.replace("，", " ").replace("。", " ").split())
    agent_keywords = set(agent_answer.replace("，", " ").replace("。", " ").split())
    
    overlap = ref_keywords & agent_keywords
    score = len(overlap) / max(len(ref_keywords), 1)
    
    return {
        "key": "correctness",
        "score": min(score, 1.0),
        "comment": f"关键词匹配率: {score:.2%} (匹配了 {len(overlap)}/{len(ref_keywords)} 个关键词)",
    }


def relevance_evaluator(run: Run, example: Example) -> dict:
    """评估 Agent 回答与问题的相关性"""
    question = example.inputs.get("question", "")
    answer = run.outputs.get("answer", "")
    
    # 简单的相关性评估：检查回答是否包含问题中的关键概念
    q_words = set(question.split())
    a_words = set(answer.split())
    
    relevant_words = q_words & a_words
    # 去掉常见停用词
    stop_words = {"什么", "是", "的", "了", "和", "有", "在", "与"}
    relevant_words -= stop_words
    
    score = 1.0 if len(relevant_words) > 0 else 0.0
    
    return {
        "key": "relevance",
        "score": score,
        "comment": f"回答包含 {len(relevant_words)} 个问题关键词",
    }


# ============================================================
# 3. 运行评估
# ============================================================

def run_evaluation(dataset_name: str):
    """对数据集运行评估"""
    
    def agent_predict(inputs: dict) -> dict:
        """Agent 预测函数 - 在评估中被自动调用"""
        from langchain_openai import ChatOpenAI
        
        llm = ChatOpenAI(model="gpt-4o", temperature=0)
        response = llm.invoke([
            {"role": "system", "content": "你是一个有帮助的助手，请简洁准确地回答问题。"},
            {"role": "user", "content": inputs["question"]},
        ])
        
        return {"answer": response.content}
    
    print(f"\n开始对数据集 '{dataset_name}' 运行评估...")
    
    results = evaluate(
        agent_predict,
        data=dataset_name,
        evaluators=[correctness_evaluator, relevance_evaluator],
        experiment_prefix="gpt4o-baseline",
        metadata={
            "model": "gpt-4o",
            "temperature": 0,
            "prompt_version": "v1.0",
        },
    )
    
    print(f"\n评估完成！")
    print(f"总用例数: {results['total_count']}")
    
    # 打印每个评估器的结果
    for eval_result in results.get("results", []):
        print(f"\n用例: {eval_result.get('example_id', 'N/A')}")
        for key, value in eval_result.get("feedback", {}).items():
            print(f"  {key}: {value}")
    
    return results


def main():
    # 创建数据集
    dataset_name = create_evaluation_dataset()
    
    # 运行评估
    results = run_evaluation(dataset_name)
    
    print("\n-> 在 LangSmith 面板中查看评估结果和对比分析")


if __name__ == "__main__":
    main()
```

---

## 32.5 LangSmith 在生产环境中的应用

### 32.5.1 基于追踪数据的告警

在生产环境中，你需要对 Agent 的异常行为设置告警。LangSmith 提供了基于反馈的告警机制。

```python
import os
from dotenv import load_dotenv
from langsmith import Client
from langsmith import traceable

load_dotenv()

client = Client()

class ProductionAgent:
    """
    生产环境 Agent，集成 LangSmith 的在线评估和告警
    """
    
    def __init__(self):
        self.alert_thresholds = {
            "max_latency_ms": 10000,       # 最大响应时间
            "max_tokens_per_request": 4000,  # 每次请求最大 Token
            "max_tool_retries": 3,           # 工具最大重试次数
        }
        self._alert_callbacks = []
    
    def on_alert(self, callback):
        """注册告警回调"""
        self._alert_callbacks.append(callback)
    
    def _send_alert(self, alert_type: str, message: str, data: dict = None):
        """发送告警"""
        alert = {
            "type": alert_type,
            "message": message,
            "data": data or {},
        }
        for callback in self._alert_callbacks:
            callback(alert)
        print(f"[ALERT] {alert_type}: {message}")
    
    @traceable(name="生产环境 Agent", run_type="chain", tags=["production"])
    def run(self, user_input: str) -> dict:
        """运行 Agent 并监控关键指标"""
        import time
        start_time = time.time()
        
        try:
            from langchain_openai import ChatOpenAI
            llm = ChatOpenAI(model="gpt-4o", temperature=0.3)
            
            response = llm.invoke([
                {"role": "system", "content": "你是一个有帮助的助手。"},
                {"role": "user", "content": user_input},
            ])
            
            latency_ms = (time.time() - start_time) * 1000
            token_count = len(response.content) // 2  # 粗略估计
            
            # 检查延迟
            if latency_ms > self.alert_thresholds["max_latency_ms"]:
                self._send_alert(
                    "HIGH_LATENCY",
                    f"响应延迟 {latency_ms:.0f}ms 超过阈值 {self.alert_thresholds['max_latency_ms']}ms",
                    {"latency_ms": latency_ms, "input": user_input[:100]},
                )
            
            # 检查 Token
            if token_count > self.alert_thresholds["max_tokens_per_request"]:
                self._send_alert(
                    "HIGH_TOKEN_USAGE",
                    f"Token 消耗 {token_count} 超过阈值 {self.alert_thresholds['max_tokens_per_request']}",
                    {"token_count": token_count, "input": user_input[:100]},
                )
            
            return {
                "answer": response.content,
                "latency_ms": latency_ms,
                "token_estimate": token_count,
            }
            
        except Exception as e:
            self._send_alert(
                "AGENT_ERROR",
                f"Agent 运行错误: {str(e)}",
                {"input": user_input[:100], "error": str(e)},
            )
            return {"answer": "抱歉，处理请求时出现错误。", "error": str(e)}


def production_demo():
    """生产环境示例"""
    agent = ProductionAgent()
    
    # 注册告警回调
    agent.on_alert(lambda alert: print(f"  [回调] 告警已记录: {alert['type']}"))
    
    # 测试正常请求
    result = agent.run("什么是 LangSmith？")
    print(f"回答: {result['answer'][:100]}...")
    print(f"延迟: {result.get('latency_ms', 0):.0f}ms")
    
    # 测试多个请求
    questions = [
        "解释 Agent 和 Chain 的区别",
        "如何优化 LLM 的性能？",
        "什么是 Function Calling？",
    ]
    
    for q in questions:
        result = agent.run(q)
        print(f"\nQ: {q}")
        print(f"A: {result['answer'][:100]}...")


if __name__ == "__main__":
    production_demo()
```

---

## 32.6 案例分析：用 LangSmith 优化 Agent

### 32.6.1 案例背景

假设你有一个客服 Agent，上线后用户满意度只有 72%。你需要用 LangSmith 来分析问题并改进。

### 32.6.2 分析过程

**第一步：查看追踪数据，找到问题模式。** 通过 LangSmith 的 Trace 视图，你发现 60% 的不满意回答都有一个共同特征：Agent 在第一轮推理时就直接给出了回答，没有调用知识库搜索工具。进一步分析发现，这是因为在 System Prompt 中没有明确指示 Agent "先搜索再回答"。

**第二步：修改 Prompt，创建新版本。** 在 System Prompt 中添加："收到用户问题后，先搜索知识库获取最新信息，然后基于搜索结果回答。只有当问题是通用常识时才可以直接回答。"

**第三步：创建评估数据集。** 从生产日志中抽取 100 条用户问题，人工标注期望的回答。这些用例构成了评估数据集。

**第四步：运行 A/B 评估。** 用 LangSmith 对比旧版 Prompt 和新版 Prompt 在同一数据集上的表现。结果发现：新版 Prompt 的回答准确率从 72% 提升到 89%，但平均延迟增加了 2 秒（因为多了一轮搜索）。

**第五步：权衡决策。** 准确率的提升（17 个百分点）远比延迟的增加（2 秒）更重要，所以决定上线新版 Prompt。

### 32.6.3 关键收获

这个案例展示了一个完整的数据驱动改进循环：追踪发现问题 -> 分析根因 -> 提出改进 -> 量化验证 -> 做出决策。LangSmith 在这个循环中的核心价值是让每一步都有数据支撑，而不是靠直觉和猜测。

---

## 32.7 常见坑与最佳实践

### 32.7.1 常见坑

**坑 1：忘记设置环境变量。** 最常见的错误是忘记设置 `LANGCHAIN_TRACING_V2=true`，导致追踪数据没有上报。如果你发现 LangSmith 面板上没有数据，首先检查环境变量。

**坑 2：追踪数据泄露敏感信息。** LangSmith 会记录 LLM 调用的完整输入和输出。如果你的 Agent 处理的是用户隐私数据（如医疗记录、财务信息），确保在 LangSmith 上设置好数据脱敏规则，或者使用自托管方案。

**坑 3：评估数据集过小。** 用 5 条测试用例来评估 Agent 质量是不够的。小样本的评估结果波动很大，可能给出误导性的结论。建议至少有 30 条以上的测试用例。

**坑 4：只看整体指标不看细节。** 平均正确率 85% 看起来不错，但如果特定类型的查询正确率只有 30%，用户感受会很差。在 LangSmith 中要按查询类型细分分析。

**坑 5：评估函数本身有偏差。** 如果你的评估函数实现得不好（比如只检查关键词匹配），评估结果就不可信。评估函数的质量直接决定了评估结论的可靠性。

### 32.7.2 最佳实践

**为每次部署创建独立的实验。** 每次修改 Agent 后（换模型、改 Prompt、加工具），在 LangSmith 上创建一个新的 Experiment，和之前的基线做对比。这样你可以清楚地知道每次修改带来的影响。

**利用 LangSmith 的反馈功能。** 在生产环境中，让用户对 Agent 的回答点赞或点踩。这些反馈数据会自动关联到对应的 Trace 上，帮助你理解哪些场景下 Agent 表现不好。

**定期清理旧数据。** LangSmith 的追踪数据会不断积累，定期清理不再需要的旧数据可以降低成本和查询延迟。

**结合 LangSmith 和自定义日志。** LangSmith 不是万能的。对于一些 LangSmith 不直接支持的自定义指标（如业务转化率），你仍然需要自定义日志来记录。两者结合使用效果最好。

---

## 32.8 练习题

**练习 1：搭建 LangSmith 追踪。** 注册 LangSmith 账号，配置环境变量，编写一个至少包含 3 个工具的 Agent，确保每次运行都在 LangSmith 上生成完整的追踪数据。截图验证追踪数据的正确性。

**练习 2：创建自定义评估器。** 编写一个评估器，评估 Agent 回答的"简洁性"——如果回答超过 200 字扣分，超过 500 字不得分。用这个评估器测试你的 Agent。

**练习 3：构建对比实验。** 创建一个包含 10 条测试用例的数据集，分别用 GPT-4o（temperature=0）和 GPT-4o（temperature=0.7）运行评估，对比两个版本在准确性和创造性上的差异。

**练习 4：实现生产告警。** 在 Agent 中实现以下告警规则：(1) 响应时间超过 5 秒；(2) 单次请求 Token 超过 3000；(3) 工具调用失败。确保告警信息包含足够的上下文用于排查。

**练习 5：分析追踪数据。** 运行你的 Agent 处理 20 个不同的查询，在 LangSmith 上查看追踪数据，找出哪些查询的处理步骤最多、Token 消耗最大，并分析原因。

**练习 6：设计端到端评估流程。** 设计一个完整的评估流程：从生产日志中自动提取测试用例 -> 人工标注期望输出 -> 运行自动评估 -> 生成评估报告。用伪代码或 Python 代码实现这个流程的框架。

---

## 32.9 本章小结

本章我们深入学习了 LangSmith 这个专业的 LLM 应用可观测性平台。从环境配置到追踪集成，从评估体系到生产告警，我们覆盖了 LangSmith 在 Agent 开发全生命周期中的核心应用。

LangSmith 的核心价值在于它提供了一个从"追踪发现问题"到"评估验证改进"的完整闭环。通过自动化的追踪，你可以随时观察 Agent 的行为；通过结构化的评估，你可以量化 Agent 的表现；通过版本化的实验，你可以安全地迭代和改进。

在实际使用中，建议从简单的自动追踪开始，逐步引入自定义追踪、评估数据集和生产告警。不要一开始就追求完美的可观观测性体系——先让它能工作，再让它工作得更好。

下一章我们将更深入地探讨 Agent 的评估体系，不仅仅局限于 LangSmith，而是系统性地讨论如何评估 Agent 的各个方面——正确性、安全性、效率和用户体验。

---

> **下一章预告：** 第 33 章将系统性地探讨 Agent 评估体系，学习如何从多个维度全面评估 Agent 的质量。
