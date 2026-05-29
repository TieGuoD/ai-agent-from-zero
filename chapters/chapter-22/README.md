# 第 22 章：LangGraph —— 图驱动的 Agent 编排

---

## 学习目标

完成本章学习后，你将能够：

1. 理解 LangGraph 的设计哲学——为什么 LangChain 的链式调用不够用，以及图结构如何解决复杂编排问题
2. 掌握状态图（State Graph）的核心概念——状态、节点、边如何协同工作
3. 能够使用 LangGraph 构建复杂的 Agent 工作流，包括条件分支、循环和并行
4. 了解 LangGraph 与 LangChain 的区别，学会在不同场景下选择合适的工具
5. 掌握 LangGraph 的人机协作（Human-in-the-Loop）机制
6. 理解 LangGraph 在生产环境中的部署和监控实践

## 核心问题

1. 为什么需要 LangGraph？LangChain 有什么局限？
2. 状态图（State Graph）解决了什么问题？
3. LangGraph 如何实现复杂的 Agent 编排？

---

## 22.1 为什么需要 LangGraph

### 22.1.1 LangChain 的局限

上一章我们学习了 LangChain 的链式调用（LCEL）。LCEL 用管道语法（`|`）将组件串联，简洁直观。但当你需要构建更复杂的工作流时，LCEL 的线性结构就会成为瓶颈。

想象你在设计一个内容审核系统：

```
用户提交内容
  → AI 自动审核
    → 如果通过 → 直接发布
    → 如果不确定 → 交给人工审核
      → 人工审核通过 → 发布
      → 人工审核拒绝 → 通知用户修改
    → 如果明显违规 → 直接拒绝并通知用户
```

这个流程有分支、有条件判断、有人工介入——用 LCEL 的线性管道很难优雅地表达。你可以用 if-else 来模拟，但代码会变得混乱且难以维护。

LangGraph 就是为了解决这个问题而生的。它用**图结构**来描述工作流，节点是执行单元，边定义了流转关系。条件分支、循环、并行——这些在图结构中都是自然的表达。

### 22.1.2 LangGraph 解决的问题

| 问题 | LangChain LCEL | LangGraph |
|------|----------------|-----------|
| 流程控制 | 线性管道 | 任意图结构 |
| 状态管理 | 需手动管理 | 内置状态管理 |
| 循环 | 不支持 | 原生支持 |
| 条件分支 | 需要变通 | add_conditional_edges |
| 持久化 | 需额外实现 | 内置检查点 |
| 人机协作 | 需额外实现 | interrupt 机制 |
| 可视化 | 有限 | 完整的图可视化 |

---

## 22.2 核心概念

### 22.2.1 状态图（State Graph）的基本结构

LangGraph 的核心是状态图。你需要定义三样东西：

1. **状态（State）**：一个 TypedDict，定义了工作流中传递的数据结构
2. **节点（Node）**：每个节点是一个函数，接收状态、处理数据、返回更新后的状态
3. **边（Edge）**：定义节点之间的连接关系，可以是固定的也可以是条件的

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator


# 1. 定义状态
class AgentState(TypedDict):
    """Agent 的状态"""
    messages: Annotated[list, operator.add]  # 消息列表（自动追加）
    next_step: str                            # 下一步行动
    result: str                               # 最终结果
    iteration: int                            # 当前迭代次数


# 2. 定义节点函数
def analyze_node(state: AgentState) -> dict:
    """分析节点：分析用户输入，决定下一步"""
    last_message = state["messages"][-1] if state["messages"] else ""
    
    # 简单的分析逻辑
    if "搜索" in str(last_message):
        next_step = "search"
    elif "计算" in str(last_message):
        next_step = "calculate"
    else:
        next_step = "respond"
    
    return {
        "next_step": next_step,
        "messages": [f"分析完成，下一步: {next_step}"],
    }


def search_node(state: AgentState) -> dict:
    """搜索节点：执行搜索"""
    return {
        "messages": ["搜索结果: 找到了相关信息"],
        "result": "搜索完成",
    }


def respond_node(state: AgentState) -> dict:
    """回复节点：生成回复"""
    return {
        "messages": ["这是最终回复"],
        "result": "回复已生成",
    }


# 3. 定义路由函数
def route_after_analyze(state: AgentState) -> str:
    """根据分析结果选择下一个节点"""
    return state.get("next_step", "respond")


# 4. 构建图
graph = StateGraph(AgentState)

# 添加节点
graph.add_node("analyze", analyze_node)
graph.add_node("search", search_node)
graph.add_node("respond", respond_node)

# 设置入口
graph.set_entry_point("analyze")

# 添加边
graph.add_conditional_edges(
    "analyze",
    route_after_analyze,
    {
        "search": "search",
        "respond": "respond",
    }
)
graph.add_edge("search", "respond")
graph.add_edge("respond", END)

# 编译图
app = graph.compile()

# 5. 执行
result = app.invoke({
    "messages": ["帮我搜索 Python 教程"],
    "next_step": "",
    "result": "",
    "iteration": 0,
})

print("最终状态:")
print(f"  消息: {result['messages']}")
print(f"  结果: {result['result']}")
```

### 22.2.2 状态管理的细节

LangGraph 的状态管理有一个关键特性：`Annotated[list, operator.add]`。这意味着当一个节点返回 `{"messages": ["新消息"]}` 时，这条消息会被**追加**到现有的消息列表中，而不是替换。

```python
from typing import TypedDict, Annotated
import operator


class DetailedState(TypedDict):
    # 使用 operator.add 注解的字段会自动追加
    messages: Annotated[list, operator.add]
    
    # 普通字段会被替换
    current_step: str
    
    # 可以使用不同的聚合策略
    scores: Annotated[list[float], operator.add]


# 节点返回值的处理规则：
# - messages: 追加（不会丢失之前的消息）
# - current_step: 替换（新值覆盖旧值）
# - scores: 追加（新分数加到列表中）
```

---

## 22.3 构建 Agent 工作流

### 22.3.1 ReAct Agent 的 LangGraph 实现

让我们用 LangGraph 来实现经典的 ReAct 模式——这是 LangGraph 最常见的用法之一。

```python
from langgraph.graph import StateGraph, END
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, AIMessage, ToolMessage
from typing import TypedDict, Annotated, Literal
import operator
import os


# 定义状态
class ReActState(TypedDict):
    messages: Annotated[list, operator.add]
    tool_calls: list
    iteration: int


# 初始化模型
model = ChatOpenAI(
    model="gpt-4o",
    api_key=os.environ.get("OPENAI_API_KEY"),
)

# 定义工具
def search_tool(query: str) -> str:
    """搜索工具"""
    return f"搜索 '{query}' 的结果: Python 是一种流行的编程语言..."

def calculator_tool(expression: str) -> str:
    """计算器工具"""
    try:
        return str(eval(expression))
    except Exception as e:
        return f"计算错误: {e}"

tools = {"search": search_tool, "calculator": calculator_tool}


# 定义节点
def agent_node(state: ReActState) -> dict:
    """Agent 节点：调用 LLM 决定下一步"""
    # 绑定工具
    model_with_tools = model.bind_tools(list(tools.values()))
    
    response = model_with_tools.invoke(state["messages"])
    
    return {
        "messages": [response],
        "tool_calls": response.tool_calls if hasattr(response, "tool_calls") and response.tool_calls else [],
        "iteration": state.get("iteration", 0) + 1,
    }


def tool_node(state: ReActState) -> dict:
    """工具节点：执行工具调用"""
    results = []
    
    for tool_call in state.get("tool_calls", []):
        tool_name = tool_call.get("name", "")
        tool_args = tool_call.get("args", {})
        
        # 执行工具
        if tool_name in tools:
            try:
                result = tools[tool_name](**tool_args)
            except Exception as e:
                result = f"工具执行错误: {e}"
        else:
            result = f"未知工具: {tool_name}"
        
        results.append(ToolMessage(
            content=str(result),
            tool_call_id=tool_call.get("id", ""),
        ))
    
    return {
        "messages": results,
        "tool_calls": [],
    }


# 定义路由
def should_continue(state: ReActState) -> Literal["tools", "__end__"]:
    """决定是否继续调用工具"""
    # 如果有工具调用，继续执行工具
    if state.get("tool_calls"):
        return "tools"
    
    # 如果超过最大迭代次数，强制结束
    if state.get("iteration", 0) >= 10:
        return "__end__"
    
    return "__end__"


# 构建图
graph = StateGraph(ReActState)

# 添加节点
graph.add_node("agent", agent_node)
graph.add_node("tools", tool_node)

# 设置入口
graph.set_entry_point("agent")

# 添加条件边
graph.add_conditional_edges(
    "agent",
    should_continue,
    {
        "tools": "tools",
        "__end__": "__end__",
    }
)

# 工具执行后回到 Agent
graph.add_edge("tools", "agent")

# 编译
app = graph.compile()

# 执行
if __name__ == "__main__":
    result = app.invoke({
        "messages": [HumanMessage(content="搜索 Python 异步编程的信息")],
        "tool_calls": [],
        "iteration": 0,
    })
    
    # 打印最终回复
    for msg in result["messages"]:
        if hasattr(msg, "type") and msg.type == "ai" and not hasattr(msg, "tool_calls"):
            print(f"Agent 回复: {msg.content}")
```

### 22.3.2 带记忆的 Agent

LangGraph 内置了检查点（Checkpoint）机制，可以自动保存和恢复 Agent 的状态，实现跨对话的记忆。

```python
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.memory import MemorySaver
from langchain_core.messages import HumanMessage, AIMessage
from typing import TypedDict, Annotated
import operator


class ChatState(TypedDict):
    messages: Annotated[list, operator.add]


def chat_node(state: ChatState) -> dict:
    """简单的聊天节点"""
    last_msg = state["messages"][-1] if state["messages"] else ""
    return {
        "messages": [AIMessage(content=f"收到: {last_msg}")],
    }


# 构建图
graph = StateGraph(ChatState)
graph.add_node("chat", chat_node)
graph.set_entry_point("chat")
graph.add_edge("chat", END)

# 添加检查点（实现记忆）
memory = MemorySaver()
app = graph.compile(checkpointer=memory)

# 使用 thread_id 隔离不同对话
config = {"configurable": {"thread_id": "user_001"}}

# 第一轮对话
result = app.invoke(
    {"messages": [HumanMessage(content="我叫张三")]},
    config=config,
)
print(f"回复: {result['messages'][-1].content}")

# 第二轮对话（自动加载历史）
result = app.invoke(
    {"messages": [HumanMessage(content="我叫什么名字？")]},
    config=config,
)
print(f"回复: {result['messages'][-1].content}")

# 查看状态
state = app.get_state(config)
print(f"\n对话历史 ({len(state.values['messages'])} 条消息):")
for msg in state.values["messages"]:
    role = "用户" if hasattr(msg, "type") and msg.type == "human" else "AI"
    print(f"  {role}: {msg.content}")
```

---

## 22.4 高级特性

### 22.4.1 人机协作（Human-in-the-Loop）

LangGraph 支持在工作流的任意节点处暂停执行，等待人类审核或输入。这在需要人工把关的场景中非常有用（比如内容审核、重要决策）。

```python
from langgraph.graph import StateGraph, END
from langgraph.types import interrupt
from langgraph.checkpoint.memory import MemorySaver
from typing import TypedDict, Annotated, Literal
import operator


class ReviewState(TypedDict):
    content: str
    review_result: str
    status: str


def draft_node(state: ReviewState) -> dict:
    """起草内容"""
    return {
        "content": f"AI 起草的内容: 关于 {state.get('content', '某主题')} 的文章",
        "status": "drafted",
    }


def review_node(state: ReviewState) -> dict:
    """人工审核节点"""
    # interrupt 会暂停执行，等待人类输入
    human_feedback = interrupt({
        "instruction": "请审核以下内容，输入 approve 拒绝或 reject 拒绝",
        "content": state["content"],
    })
    
    return {
        "review_result": human_feedback,
        "status": "reviewed",
    }


def publish_node(state: ReviewState) -> dict:
    """发布内容"""
    return {
        "status": "published",
    }


def revise_node(state: ReviewState) -> dict:
    """修订内容"""
    return {
        "content": f"[修订版] {state['content']}",
        "status": "revised",
    }


def route_after_review(state: ReviewState) -> Literal["publish", "revise"]:
    """审核后的路由"""
    if state.get("review_result", "").lower() == "approve":
        return "publish"
    return "revise"


# 构建图
graph = StateGraph(ReviewState)
graph.add_node("draft", draft_node)
graph.add_node("review", review_node)
graph.add_node("publish", publish_node)
graph.add_node("revise", revise_node)

graph.set_entry_point("draft")
graph.add_edge("draft", "review")
graph.add_conditional_edges("review", route_after_review)
graph.add_edge("publish", END)
graph.add_edge("revise", "draft")  # 修订后重新起草

# 编译
memory = MemorySaver()
app = graph.compile(
    checkpointer=memory,
    interrupt_before=["review"],  # 在审核前暂停
)

# 使用
config = {"configurable": {"thread_id": "review_001"}}

# 开始执行（会暂停在审核节点）
result = app.invoke(
    {"content": "AI Agent 的未来发展趋势", "review_result": "", "status": "started"},
    config,
)
print(f"状态: {result.get('status')}")
print("等待人工审核...")

# 人工审核后继续
result = app.invoke("approve", config)
print(f"审核后状态: {result.get('status')}")
```

### 22.4.2 流式输出

LangGraph 支持流式输出，你可以看到每个节点的执行过程。

```python
# 流式输出
for event in app.stream(input_data, config):
    for node_name, node_output in event.items():
        print(f"[{node_name}]")
        if isinstance(node_output, dict):
            for key, value in node_output.items():
                print(f"  {key}: {str(value)[:100]}")
        print()
```

### 22.4.3 图的可视化

LangGraph 支持将图导出为图片，方便理解和调试。

```python
# 获取图的 Mermaid 表示
mermaid_code = app.get_graph().draw_mermaid()
print(mermaid_code)

# 在 Jupyter 中显示图片
from IPython.display import Image, display
display(Image(app.get_graph().draw_mermaid_png()))

# 保存为 PNG 文件
with open("graph.png", "wb") as f:
    f.write(app.get_graph().draw_mermaid_png())
```

---

## 22.5 LangGraph vs LangChain

### 22.5.1 什么时候用 LangChain

- **简单链式调用**：Prompt → Model → Parser，线性流程就够了
- **Prompt 模板管理**：需要管理和复用 Prompt 模板
- **基础 RAG**：文档加载 → 向量化 → 检索 → 问答
- **快速原型**：快速搭建一个简单的 LLM 应用

### 22.5.2 什么时候用 LangGraph

- **复杂工作流**：有条件分支、循环、并行的流程
- **状态管理**：需要在多个步骤之间传递和维护复杂状态
- **人机协作**：需要人类在关键步骤介入审核
- **持久化**：需要保存和恢复执行状态
- **多 Agent 协作**：多个 Agent 之间需要协调和通信

### 22.5.3 LangChain + LangGraph 组合使用

LangGraph 并不替代 LangChain，而是补充 LangChain 的不足。最佳实践是：

- 用 **LangChain** 来管理模型、Prompt、工具
- 用 **LangGraph** 来编排复杂的工作流

```python
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.memory import MemorySaver
from typing import TypedDict, Annotated
import operator
import os


# LangChain 组件
model = ChatOpenAI(model="gpt-4o", api_key=os.environ.get("OPENAI_API_KEY"))

@tool
def get_weather(city: str) -> str:
    """获取指定城市的天气信息"""
    return f"{city}今天晴天，气温 25°C"

# LangGraph 编排
class WeatherState(TypedDict):
    messages: Annotated[list, operator.add]
    city: str
    weather: str


def extract_city(state: WeatherState) -> dict:
    """提取城市名称"""
    last_msg = state["messages"][-1] if state["messages"] else ""
    # 简单提取（实际中可以用 LLM）
    city = "北京"  # 默认值
    for c in ["北京", "上海", "广州", "深圳"]:
        if c in str(last_msg):
            city = c
            break
    return {"city": city}


def fetch_weather(state: WeatherState) -> dict:
    """获取天气"""
    weather = get_weather.invoke(state["city"])
    return {"weather": weather, "messages": [f"天气信息: {weather}"]}


def respond(state: WeatherState) -> dict:
    """生成回复"""
    return {
        "messages": [f"{state['city']}的天气: {state['weather']}"],
    }


# 构建图
graph = StateGraph(WeatherState)
graph.add_node("extract", extract_city)
graph.add_node("fetch", fetch_weather)
graph.add_node("respond", respond)

graph.set_entry_point("extract")
graph.add_edge("extract", "fetch")
graph.add_edge("fetch", "respond")
graph.add_edge("respond", END)

memory = MemorySaver()
app = graph.compile(checkpointer=memory)

# 执行
config = {"configurable": {"thread_id": "weather_001"}}
result = app.invoke({
    "messages": ["北京今天天气怎么样？"],
    "city": "",
    "weather": "",
}, config)

print(f"回复: {result['messages'][-1]}")
```

---

## 22.6 常见坑

### 22.6.1 状态设计不当

**问题描述：** 状态字段过多导致数据传递困难，或者字段过少导致信息丢失。

**解决方案：** 精心设计状态结构，只包含必要的字段。使用 `Annotated[list, operator.add]` 来处理需要追加的字段。

### 22.6.2 死循环

**问题描述：** 条件边配置不当，导致节点之间无限循环（A → B → A → B → ...）。

**解决方案：** 添加迭代计数器和最大迭代限制。在条件边的路由函数中检查迭代次数。

```python
def should_continue(state: MyState) -> str:
    if state.get("iteration", 0) >= 10:
        return END  # 强制结束
    if state.get("needs_more_work"):
        return "process"
    return END
```

### 22.6.3 并发状态冲突

**问题描述：** 多个请求使用同一个 thread_id，导致状态混乱。

**解决方案：** 确保每个用户/会话使用唯一的 thread_id。

### 22.6.4 节点函数的返回值

**问题描述：** 节点函数返回了状态中未定义的字段，或者返回了错误类型的数据。

**解决方案：** 节点函数返回的字典 key 必须与 TypedDict 中定义的字段一致。追加字段（如 messages）必须返回列表，替换字段返回对应类型。

---

## 22.7 练习题

### 练习 1：构建基础状态图

构建一个简单的状态图，要求：
- 至少 3 个节点
- 支持条件分支
- 有明确的终止条件
- 使用状态在节点间传递数据

### 练习 2：用 LangGraph 实现 ReAct Agent

实现一个 ReAct Agent，要求：
- 支持至少 2 个工具
- 有循环检测（防止无限工具调用）
- 支持流式输出
- 使用 LangGraph 的条件边

### 练习 3：实现人机协作工作流

实现一个需要人工审核的工作流，要求：
- 在关键步骤中断
- 支持人类输入
- 能根据审核结果继续或终止
- 使用检查点保存状态

### 练习 4：实现多步骤数据处理

用 LangGraph 构建一个多步骤数据处理管道：
- 数据清洗节点
- 数据分析节点
- 报告生成节点
- 支持并行处理多个数据源

### 练习 5：构建对话系统

用 LangGraph 构建一个带记忆的对话系统：
- 使用 MemorySaver 实现跨对话记忆
- 支持多用户隔离（不同 thread_id）
- 有人工介入机制

---

## 22.8 实战任务

### 任务：构建复杂工作流系统

**目标：** 使用 LangGraph 构建一个完整的、可投入生产的工作流系统。

**要求：**

1. 支持多步骤处理流程
2. 有条件分支和循环
3. 状态持久化（使用 SQLite 检查点）
4. 有人工审核环节
5. 提供流程可视化

**参考实现：**

```python
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.types import interrupt
from typing import TypedDict, Annotated, Literal
import operator


class WorkflowState(TypedDict):
    messages: Annotated[list, operator.add]
    current_step: str
    data: dict
    approvals: list
    iteration: int


def intake_node(state: WorkflowState) -> dict:
    """接收任务"""
    return {
        "current_step": "process",
        "data": {**state.get("data", {}), "intake": "completed"},
    }


def process_node(state: WorkflowState) -> dict:
    """处理任务"""
    return {
        "data": {**state.get("data", {}), "processed": True},
        "current_step": "review",
    }


def review_node(state: WorkflowState) -> dict:
    """人工审核"""
    feedback = interrupt("请审核处理结果: approve 或 reject")
    return {
        "approvals": [feedback],
        "current_step": "finalize" if "approve" in str(feedback).lower() else "revise",
    }


def finalize_node(state: WorkflowState) -> dict:
    """完成"""
    return {
        "data": {**state.get("data", {}), "final": True},
        "current_step": "done",
    }


def revise_node(state: WorkflowState) -> dict:
    """修订"""
    return {
        "iteration": state.get("iteration", 0) + 1,
        "current_step": "process",
    }


def route_after_review(state: WorkflowState) -> Literal["finalize", "revise"]:
    return "finalize" if state["current_step"] == "finalize" else "revise"


# 构建图
graph = StateGraph(WorkflowState)
graph.add_node("intake", intake_node)
graph.add_node("process", process_node)
graph.add_node("review", review_node)
graph.add_node("finalize", finalize_node)
graph.add_node("revise", revise_node)

graph.set_entry_point("intake")
graph.add_edge("intake", "process")
graph.add_edge("process", "review")
graph.add_conditional_edges("review", route_after_review)
graph.add_edge("finalize", END)
graph.add_edge("revise", "process")

# 编译
with SqliteSaver.from_conn_string("workflow.db") as memory:
    app = graph.compile(
        checkpointer=memory,
        interrupt_before=["review"],
    )
    
    config = {"configurable": {"thread_id": "workflow_001"}}
    
    # 执行（会在 review 前暂停）
    result = app.invoke({
        "messages": ["开始任务"],
        "current_step": "started",
        "data": {},
        "approvals": [],
        "iteration": 0,
    }, config)
    
    print(f"暂停在: {result['current_step']}")
    
    # 人工审核后继续
    result = app.invoke("approve", config)
    print(f"最终状态: {result['current_step']}")
```

---

## 22.9 本章小结

- **LangGraph 是 LangChain 的图编排扩展**，专门为复杂工作流设计。当 LangChain 的线性管道不足以表达你的流程时，LangGraph 的图结构可以优雅地处理分支、循环、并行等复杂模式。

- **状态图（State Graph）** 是 LangGraph 的核心概念。你需要定义状态（TypedDict）、节点（处理函数）和边（连接关系）。状态在节点之间自动传递，节点可以读取和更新状态。

- **节点**是执行单元。每个节点是一个函数，接收当前状态，返回需要更新的字段。使用 `Annotated[list, operator.add]` 可以实现字段的自动追加。

- **边**定义了节点之间的流转关系。普通边是固定连接，条件边（`add_conditional_edges`）可以根据状态动态选择下一个节点。

- **条件边**支持分支和循环，是实现复杂逻辑的关键。路由函数根据状态返回下一个节点的名称。

- **检查点（Checkpoint）** 支持状态的持久化和恢复。MemorySaver 适合开发测试，SqliteSaver 适合生产环境。

- **人机协作**通过 `interrupt` 机制实现。在关键节点处暂停执行，等待人类输入后继续。

- LangGraph 适合**复杂的多步骤工作流**，特别是需要状态管理、条件分支和人工介入的场景。对于简单的链式调用，直接使用 LangChain 即可。

