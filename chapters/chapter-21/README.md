# 第 21 章：LangChain 基础 —— 框架的架构思想

---

## 学习目标

完成本章学习后，你将能够：

1. 理解为什么需要使用框架——直接调用 API 有哪些痛点，框架如何解决这些问题
2. 掌握 LangChain 的六大核心模块——Models、Prompts、Chains、Memory、Agents、Retrievers
3. 能够使用 LCEL（LangChain Expression Language）构建灵活的处理链
4. 了解 LangChain 的链式调用机制——组件如何通过管道语法串联
5. 理解 LangChain 的可扩展性设计——如何自定义组件
6. 掌握使用 LangChain 构建 RAG 应用的基本流程

## 核心问题

1. 为什么要使用框架？直接调用 API 有什么问题？
2. LangChain 的核心架构是什么？
3. LangChain 如何简化 Agent 开发？

---

## 21.1 为什么需要框架

### 21.1.1 直接调用 API 的痛点

在前面的章节中，我们都是直接调用 OpenAI 或其他 LLM 的 API。这种方式在学习阶段没问题，但在构建实际应用时会遇到很多重复性的工作：

```python
import openai
import json

# 直接调用 API 时，你需要自己处理：
# 1. Prompt 模板管理
system_prompt = "你是一个{role}助手"
user_prompt = "请回答关于{topic}的问题"

# 2. 消息格式化
messages = [
    {"role": "system", "content": system_prompt.format(role="翻译")},
    {"role": "user", "content": user_prompt.format(topic="机器学习")},
]

# 3. API 调用和错误处理
try:
    client = openai.OpenAI(api_key=os.environ.get("OPENAI_API_KEY"))
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=messages,
        temperature=0.7,
    )
    answer = response.choices[0].message.content
except openai.APIError as e:
    print(f"API 错误: {e}")
    # 需要自己实现重试逻辑...
except openai.RateLimitError:
    # 需要自己实现限流逻辑...
    pass

# 4. 输出解析
try:
    result = json.loads(answer)
except json.JSONDecodeError:
    # 需要自己实现容错解析...
    pass

# 5. 对话历史管理
# 需要自己维护 messages 列表...

# 6. 如果要做 RAG...
# 需要自己实现文档加载、分割、向量化、检索、问答链...
```

当你的应用变复杂时（多步骤处理、工具调用、对话历史、RAG、流式输出……），自己管理这些底层细节会让代码变得非常臃肿和脆弱。

### 21.1.2 框架的价值

LangChain 提供了一层抽象，让你可以专注于业务逻辑，而不是底层细节。它把常见的操作封装成了可组合的模块，你只需要"搭积木"就能构建复杂的 LLM 应用。

框架的核心价值：
1. **标准化**：统一的接口和模式，代码更一致
2. **可组合**：模块化的组件，可以自由组合
3. **可扩展**：易于添加新的 LLM 提供商、工具、数据源
4. **社区**：丰富的集成、示例和最佳实践
5. **生产就绪**：内置了错误处理、重试、日志等生产环境需要的功能

---

## 21.2 LangChain 的核心概念

### 21.2.1 六大核心模块

LangChain 的架构由六个核心模块组成：

```
LangChain 架构
├── Models（模型）    → LLM 和 Chat Model 的统一抽象
├── Prompts（提示）   → Prompt 模板管理和复用
├── Chains（链）      → 组件的组合和编排
├── Memory（记忆）    → 对话历史管理
├── Agents（代理）    → 自主决策和工具使用
└── Retrievers（检索器）→ RAG 相关组件
```

### 21.2.2 安装和配置

```bash
# 安装 LangChain 核心库
pip install langchain

# 安装 OpenAI 集成
pip install langchain-openai

# 安装社区集成（可选）
pip install langchain-community
```

```python
import os

# 设置 API Key（使用环境变量，不要硬编码）
os.environ["OPENAI_API_KEY"] = "your-api-key-here"
```

---

## 21.3 Models 模块

### 21.3.1 LLM 和 Chat Model 的区别

LangChain 区分了两种模型接口：

- **LLM**：接收字符串，返回字符串。适合简单的文本生成任务。
- **Chat Model**：接收消息列表，返回消息对象。适合对话和需要角色区分的场景。

在实际应用中，Chat Model 是更常用的，因为大多数现代 LLM 都是对话模型。

```python
from langchain_openai import ChatOpenAI, OpenAI
from langchain_core.messages import HumanMessage, SystemMessage

# 创建 Chat Model（推荐）
chat_model = ChatOpenAI(
    model="gpt-4o",
    temperature=0.7,
    max_tokens=2000,
)

# 使用消息格式调用
messages = [
    SystemMessage(content="你是一个有帮助的助手"),
    HumanMessage(content="什么是 LangChain？"),
]

response = chat_model.invoke(messages)
print(response.content)

# 流式输出
print("\n流式输出: ", end="")
for chunk in chat_model.stream(messages):
    print(chunk.content, end="", flush=True)
print()
```

### 21.3.2 不同 LLM 提供商的支持

LangChain 支持几乎所有主流的 LLM 提供商：

```python
# OpenAI
from langchain_openai import ChatOpenAI
model_openai = ChatOpenAI(model="gpt-4o")

# Anthropic (Claude)
from langchain_anthropic import ChatAnthropic
model_claude = ChatAnthropic(model="claude-sonnet-4-20250514")

# 本地模型 (Ollama)
from langchain_ollama import ChatOllama
model_local = ChatOllama(model="llama3")

# 所有模型都使用相同的接口
response = model_openai.invoke([HumanMessage(content="你好")])
response = model_claude.invoke([HumanMessage(content="你好")])
response = model_local.invoke([HumanMessage(content="你好")])
```

---

## 21.4 Prompts 模块

### 21.4.1 Prompt 模板

Prompt 模板让你可以定义可复用的提示格式，通过变量替换来适应不同的输入。

```python
from langchain_core.prompts import ChatPromptTemplate, PromptTemplate

# 简单模板
simple_template = PromptTemplate(
    input_variables=["topic"],
    template="请用一段话详细解释 {topic}",
)

# 使用模板
prompt = simple_template.invoke({"topic": "机器学习"})
print(prompt)  # "请用一段话详细解释 机器学习"

# Chat 模板（推荐用于对话场景）
chat_template = ChatPromptTemplate.from_messages([
    ("system", "你是一个{role}专家，请用专业但易懂的语言回答问题"),
    ("human", "{question}"),
])

# 使用 Chat 模板
messages = chat_template.invoke({
    "role": "数据科学",
    "question": "什么是过拟合？如何避免？",
})

print(messages)  # 包含 SystemMessage 和 HumanMessage
```

### 21.4.2 Few-shot 模板

Few-shot 模板通过提供示例来引导 LLM 的输出格式和风格。

```python
from langchain_core.prompts import FewShotChatMessagePromptTemplate

# 定义示例
examples = [
    {"input": "这个产品太棒了", "output": '{"sentiment": "positive", "score": 0.95}'},
    {"input": "质量很差非常失望", "output": '{"sentiment": "negative", "score": 0.1}'},
    {"input": "一般般没什么特别的", "output": '{"sentiment": "neutral", "score": 0.5}'},
]

# 示例格式
example_prompt = ChatPromptTemplate.from_messages([
    ("human", "{input}"),
    ("ai", "{output}"),
])

# Few-shot 模板
few_shot_prompt = FewShotChatMessagePromptTemplate(
    example_prompt=example_prompt,
    examples=examples,
)

# 完整提示
final_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个情感分析器，请分析用户评论的情感，输出 JSON 格式。"),
    few_shot_prompt,
    ("human", "{input}"),
])

# 使用
result = final_prompt.invoke({"input": "服务态度很好但价格太贵了"})
print(result)
```

---

## 21.5 Chains 模块

### 21.5.1 LCEL（LangChain Expression Language）

LCEL 是 LangChain 的核心编排语言，使用管道操作符（`|`）将组件串联起来。

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

# 创建组件
model = ChatOpenAI(model="gpt-4o")
prompt = ChatPromptTemplate.from_template(
    "请用一句话解释：{topic}"
)
parser = StrOutputParser()

# 使用 LCEL 编排（管道语法）
chain = prompt | model | parser

# 执行
result = chain.invoke({"topic": "量子计算"})
print(result)

# 这等价于：
# 1. prompt 将输入格式化为消息
# 2. model 调用 LLM 生成回复
# 3. parser 提取回复中的文本
```

### 21.5.2 链的组合与并行

```python
from langchain_core.runnables import RunnableParallel, RunnableLambda

# 定义多个链
analysis_prompt = ChatPromptTemplate.from_template("分析以下文本的情感：{text}")
analysis_chain = analysis_prompt | model | parser

summary_prompt = ChatPromptTemplate.from_template("总结以下内容的核心要点：{text}")
summary_chain = summary_prompt | model | parser

# 并行执行（同时进行情感分析和摘要）
parallel_chain = RunnableParallel(
    analysis=analysis_chain,
    summary=summary_chain,
)

# 执行
result = parallel_chain.invoke({"text": "LangChain 是一个强大的 LLM 应用框架"})
print(f"情感分析: {result['analysis']}")
print(f"内容摘要: {result['summary']}")

# 顺序组合
full_chain = parallel_chain | RunnableLambda(
    lambda x: f"分析结果: {x['analysis']}\n\n摘要: {x['summary']}"
)

result = full_chain.invoke({"text": "Python 是最流行的编程语言之一"})
print(result)
```

---

## 21.6 Memory 模块

### 21.6.1 内置记忆类型

```python
from langchain_core.chat_history import InMemoryChatMessageHistory
from langchain_core.messages import HumanMessage, AIMessage

# 创建内存聊天历史
history = InMemoryChatMessageHistory()

# 添加消息
history.add_user_message("你好，我叫张三")
history.add_ai_message("你好张三！有什么可以帮助你的？")

# 获取消息
for msg in history.messages:
    print(f"{msg.type}: {msg.content}")
```

### 21.6.2 与 Chain 集成

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_openai import ChatOpenAI

# 带记忆的对话
model = ChatOpenAI(model="gpt-4o")

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个有帮助的助手，请记住用户的个人信息。"),
    MessagesPlaceholder(variable_name="history"),
    ("human", "{input}"),
])

chain = prompt | model

# 对话循环
history = []

# 第一轮
response = chain.invoke({"history": history, "input": "我叫张三，是一名 Python 工程师"})
print(f"Agent: {response.content}")

# 更新历史
history.append(HumanMessage(content="我叫张三，是一名 Python 工程师"))
history.append(AIMessage(content=response.content))

# 第二轮（Agent 应该记得你叫张三）
response = chain.invoke({"history": history, "input": "我叫什么名字？做什么工作的？"})
print(f"Agent: {response.content}")
```

---

## 21.7 Agents 模块

### 21.7.1 使用 LangChain 构建 Agent

```python
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

# 定义工具
@tool
def search(query: str) -> str:
    """搜索互联网获取信息。传入搜索关键词。"""
    return f"搜索结果：关于 {query}，最新的信息显示..."

@tool
def calculator(expression: str) -> str:
    """计算数学表达式。传入数学表达式字符串。"""
    try:
        # 安全的数学计算
        import ast
        tree = ast.parse(expression, mode='eval')
        return str(eval(compile(tree, '<calc>', 'eval')))
    except Exception as e:
        return f"计算错误: {e}"

# 创建 Agent
model = ChatOpenAI(model="gpt-4o")
tools = [search, calculator]

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个有帮助的助手，可以使用工具来完成任务。请先思考需要什么工具，然后调用。"),
    ("human", "{input}"),
    MessagesPlaceholder(variable_name="agent_scratchpad"),
])

agent = create_tool_calling_agent(model, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True, max_iterations=5)

# 执行
result = executor.invoke({"input": "搜索 LangChain 的最新版本信息，然后计算 2 的 10 次方"})
print(f"\n最终结果: {result['output']}")
```

---

## 21.8 Retrievers 模块

### 21.8.1 RAG 链

```python
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough

# 1. 准备文档
docs = [
    "LangChain 是一个用于构建 LLM 应用的开源框架",
    "它支持多种 LLM 提供商，包括 OpenAI、Anthropic、Google 等",
    "LangChain 的核心组件包括 Models、Prompts、Chains、Memory、Agents",
    "LCEL 是 LangChain 的表达式语言，使用管道语法编排组件",
]

# 2. 创建向量存储
embeddings = OpenAIEmbeddings()
vectorstore = FAISS.from_texts(docs, embeddings)
retriever = vectorstore.as_retriever(search_kwargs={"k": 2})

# 3. 创建 RAG 链
model = ChatOpenAI(model="gpt-4o")

template = """基于以下上下文回答问题。如果上下文中没有相关信息，请说明你不知道。

上下文：
{context}

问题：{question}

回答："""

prompt = ChatPromptTemplate.from_template(template)

def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | model
    | StrOutputParser()
)

# 4. 使用
result = rag_chain.invoke("LangChain 是什么？")
print(result)
```

---

## 21.9 LangChain 的流式处理

### 21.9.1 流式输出的基础用法

流式输出让用户不必等待完整的回复，而是逐字逐句地看到 Agent 的回答，大大提升了用户体验。

```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage

model = ChatOpenAI(model="gpt-4o", streaming=True)

messages = [HumanMessage(content="用三句话介绍 Python")]

# 方式 1：使用 stream 方法
print("方式 1 - stream:")
for chunk in model.stream(messages):
    print(chunk.content, end="", flush=True)
print()

# 方式 2：使用 astream（异步）
import asyncio

async def async_stream():
    print("方式 2 - astream (异步):")
    async for chunk in model.astream(messages):
        print(chunk.content, end="", flush=True)
    print()

asyncio.run(async_stream())
```

### 21.9.2 在链中使用流式输出

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_template("请写一首关于{topic}的诗")
model = ChatOpenAI(model="gpt-4o", streaming=True)
parser = StrOutputParser()

chain = prompt | model | parser

# 链也支持流式输出
print("流式输出:")
for chunk in chain.stream({"topic": "人工智能"}):
    print(chunk, end="", flush=True)
print()
```

---

## 21.10 错误处理与重试

### 21.10.1 LangChain 的内置重试

```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage

# 使用 with_retry 装饰器
model = ChatOpenAI(model="gpt-4o").with_retry(
    stop_after_attempt=3,          # 最多重试 3 次
    wait_exponential_jitter=True,   # 指数退避 + 随机抖动
    retry_on_exception_types=(     # 只在这些异常时重试
        ConnectionError,
        TimeoutError,
    ),
)

response = model.invoke([HumanMessage(content="你好")])
print(response.content)
```

### 21.10.2 自定义错误处理

```python
from langchain_core.runnables import RunnableLambda

def safe_model_call(input_data: dict) -> dict:
    """带错误处理的模型调用"""
    try:
        model = ChatOpenAI(model="gpt-4o")
        response = model.invoke(input_data["messages"])
        return {"result": response.content, "error": None}
    except Exception as e:
        return {"result": None, "error": str(e)}

safe_chain = RunnableLambda(safe_model_call)
result = safe_chain.invoke({"messages": [HumanMessage(content="你好")]})

if result["error"]:
    print(f"调用失败: {result['error']}")
else:
    print(f"结果: {result['result']}")
```

---

## 21.11 LangChain 的调试技巧

### 21.11.1 verbose 模式

```python
# 在 AgentExecutor 中启用 verbose
executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True,  # 打印每一步的详细信息
    max_iterations=5,
)

# verbose 模式会输出：
# > Entering new AgentExecutor chain...
# Invoking: `search` with `Python tutorial`
# > 搜索结果...
# > Finished chain.
```

### 21.11.2 使用 LangSmith 进行可视化调试

LangSmith 是 LangChain 官方的监控和调试平台。它能追踪链的每一次调用，显示输入输出、耗时、Token 消耗等信息。

```python
# 设置 LangSmith 环境变量
import os
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "your-langsmith-api-key"

# 之后所有的 LangChain 调用都会自动追踪
model = ChatOpenAI(model="gpt-4o")
response = model.invoke([HumanMessage(content="你好")])
# 在 LangSmith 控制台中可以看到这次调用的详细信息
```

---

## 21.12 自定义组件

### 21.12.1 自定义输出解析器

```python
from langchain_core.output_parsers import BaseOutputParser
import json


class JSONOutputParser(BaseOutputParser):
    """自定义 JSON 输出解析器"""
    
    def parse(self, text: str) -> dict:
        """解析 LLM 输出为 JSON"""
        try:
            return json.loads(text)
        except json.JSONDecodeError:
            # 尝试从文本中提取 JSON
            import re
            match = re.search(r'\{.*\}', text, re.DOTALL)
            if match:
                try:
                    return json.loads(match.group())
                except json.JSONDecodeError:
                    pass
            return {"error": "JSON 解析失败", "raw": text}
    
    @property
    def _type(self) -> str:
        return "json_output_parser"


# 使用自定义解析器
parser = JSONOutputParser()
result = parser.parse('{"answer": 42, "confidence": 0.95}')
print(result)  # {'answer': 42, 'confidence': 0.95}
```

### 21.12.2 自定义 Runnable

```python
from langchain_core.runnables import Runnable


class TextCleaner(Runnable):
    """自定义文本清理器"""
    
    def invoke(self, input_text: str, config=None) -> str:
        """清理文本"""
        # 去除多余空白
        text = " ".join(input_text.split())
        # 去除特殊字符
        text = text.strip()
        return text
    
    async def ainvoke(self, input_text: str, config=None) -> str:
        """异步版本"""
        return self.invoke(input_text, config)


# 在链中使用自定义 Runnable
cleaner = TextCleaner()
chain = cleaner | prompt | model | parser
```

---

## 21.13 常见坑

### 21.9.1 过度依赖框架

**问题描述：** 使用框架但不理解底层原理，遇到问题无法调试。比如 LCEL 管道报错时，不知道是哪个组件出了问题。

**解决方案：** 先理解 LLM API 的基础，再学习框架。在使用框架的同时，了解它背后的实现机制。

### 21.9.2 版本兼容性

**问题描述：** LangChain 更新非常频繁，API 可能在一个大版本内就发生变化。今天能跑的代码，下个月升级后可能就报错了。

**解决方案：** 锁定版本（在 requirements.txt 中指定版本号），关注官方文档的迁移指南。对于生产环境，不要轻易升级。

### 21.9.3 性能开销

**问题描述：** 框架的抽象层增加了额外开销。LangChain 的一次调用可能在底层做了很多额外的工作（日志、回调、验证等），比直接调用 API 慢。

**解决方案：** 对于性能敏感的场景（比如高并发服务），考虑直接使用 API。LangChain 适合快速原型开发和中低并发场景。

### 21.9.4 调试困难

**问题描述：** 链式调用出错时，很难定位问题出在哪个环节。

**解决方案：** 使用 `verbose=True` 参数查看每一步的输入输出。使用 LangSmith（LangChain 的官方监控平台）进行可视化调试。

---

## 21.10 练习题

### 练习 1：LangChain 基础

使用 LangChain 实现一个简单的聊天机器人，要求：
- 使用 Chat Model
- 支持对话历史（Memory）
- 使用 Prompt 模板

### 练习 2：Chain 组合

使用 LCEL 实现一个多步骤处理链，要求：
- 包含至少 3 个处理步骤
- 支持并行执行
- 有错误处理

### 练习 3：Agent 实现

使用 LangChain 实现一个工具调用 Agent，要求：
- 支持至少 2 个工具
- 有清晰的推理过程
- 能处理工具调用失败

### 练习 4：RAG 应用

使用 LangChain 构建一个简单的 RAG 应用，要求：
- 加载和分割文档
- 向量化和存储
- 检索和问答

### 练习 5：Chain 调试

给定一个有 bug 的 LCEL 链，找出问题所在并修复。练习使用 verbose 模式和调试技巧。

---

## 21.11 实战任务

### 任务：构建 RAG 应用

**目标：** 使用 LangChain 构建一个完整的 RAG（检索增强生成）应用，能够基于文档库回答问题。

**要求：**

1. 文档加载和分割（支持 txt、md 等格式）
2. 向量化和存储（使用 FAISS 或 ChromaDB）
3. 检索和问答链
4. 支持对话历史
5. 提供简单的 Web 界面（可选）

---

## 21.12 本章小结

- **LangChain 是最流行的 LLM 应用开发框架**。它通过提供标准化的接口和可组合的模块，大大简化了 LLM 应用的开发。直接调用 API 虽然灵活，但在构建复杂应用时会面临大量重复性工作和维护成本。

- **六大核心模块**：Models（模型抽象）、Prompts（提示模板）、Chains（组件编排）、Memory（对话记忆）、Agents（自主代理）、Retrievers（检索器）。每个模块负责一个方面的功能，可以自由组合。

- **LCEL（LangChain Expression Language）** 使用管道操作符（`|`）将组件串联，代码简洁直观。支持顺序执行、并行执行和自定义函数，是 LangChain 的核心编排机制。

- **Prompt 模板**简化了提示管理，支持变量替换、Few-shot 示例、消息格式化。好的 Prompt 模板能显著提升 LLM 的输出质量。

- **Agents** 让 LLM 能自主使用工具，通过 create_tool_calling_agent 可以快速构建工具调用代理。

- **Retrievers** 是 RAG 的基础，LangChain 提供了完整的文档加载、分割、向量化、检索、问答链的工具链。

- 使用框架时要**理解底层原理**，避免过度依赖。框架是工具，不是银弹。对于简单任务，直接调用 API 可能更高效。学习 LangChain 的最佳方式是先掌握 LLM API 的基础，再通过框架来提升开发效率。

- LangChain 的**生态非常丰富**，除了核心库之外还有 langchain-openai、langchain-anthropic、langchain-community 等集成包，覆盖了几乎所有主流 LLM 提供商和工具。选择集成包时要注意版本兼容性。

- **生产环境建议**：使用 LangSmith 进行监控和调试，使用 with_retry 实现重试机制，使用 streaming=True 提升用户体验，使用环境变量管理 API Key。

## 21.14 LangChain vs 直接调用 API：何时选择哪个

理解框架的价值和局限，才能做出正确的技术选型。

**选择直接调用 API 的场景：** 简单的聊天应用，只需要一次 LLM 调用就能完成任务；性能敏感的高并发服务，每一毫秒延迟都很重要；需要完全控制底层行为的场景，框架的抽象层可能隐藏了重要的细节；学习和理解 LLM 工作原理的阶段。

**选择 LangChain 的场景：** 需要多个 LLM 调用串联的复杂应用；需要集成多种工具和数据源的 Agent 系统；需要快速原型验证的项目；需要对话历史管理、RAG、Agent 等高级功能；团队协作开发，需要统一的代码规范和接口。

**折中方案：** 使用 LangChain 的底层组件（如 Prompt 模板、输出解析器），但不用它的链式编排。这样既能享受组件化的好处，又不会有太多框架开销。

