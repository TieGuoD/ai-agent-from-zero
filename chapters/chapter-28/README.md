# 第 28 章 AutoGen 框架 —— 对话驱动的 Multi-Agent

> "对话是人类协作的最基本形式。AutoGen 框架的核心理念正是如此——让多个 Agent 通过自然语言对话来协作完成任务，就像人类团队成员之间的讨论一样自然和高效。"

---

## 学习目标

通过本章的学习，你将能够：

1. 理解 AutoGen 框架的核心架构和设计理念
2. 掌握 AutoGen 中 Agent 的类型和特点（AssistantAgent、UserProxyAgent 等）
3. 学会使用 AutoGen 构建对话驱动的 Multi-Agent 系统
4. 理解 AutoGen 的代码执行机制和安全沙箱
5. 使用 Python 实现基于 AutoGen 的实际应用
6. 掌握 AutoGen 的自定义扩展方法
7. 了解 AutoGen 的最佳实践和常见问题

---

## 核心问题

- AutoGen 框架的核心设计理念是什么？它与其他 Multi-Agent 框架有何不同？
- AutoGen 中的对话驱动模式是如何工作的？
- 如何在 AutoGen 中安全地执行代码？
- AutoGen 适合什么样的应用场景？有什么局限性？
- 如何自定义 AutoGen 的行为以满足特定需求？

---

## 28.1 AutoGen 框架概述

### 28.1.1 什么是 AutoGen

AutoGen 是微软研究院（Microsoft Research）开发的一个开源 Multi-Agent 对话框架。它的核心理念是：通过多个 Agent 之间的对话来解决复杂任务。在 AutoGen 的世界里，Agent 之间的交互被抽象为"对话"——就像人类团队成员之间通过讨论来协作一样。

AutoGen 的名字来自 "Automated Agent Generation"（自动化 Agent 生成），但它真正的核心不是自动生成 Agent，而是提供一个灵活的框架，让开发者能够轻松地定义 Agent、配置它们之间的对话关系，然后让它们自主地协作完成任务。

### 28.1.2 AutoGen 的核心理念

AutoGen 的设计理念可以用三个关键词来概括：**对话驱动**、**人类参与**、**代码执行**。

**对话驱动**意味着 Agent 之间的所有协作都是通过对话来实现的。一个 Agent 的输出就是另一个 Agent 的输入，对话的流转就是任务的流转。这种设计非常自然——因为它模仿了人类团队的工作方式。

**人类参与**意味着 AutoGen 支持人在回路（Human-in-the-Loop）的模式。在 Agent 对话的任何环节，人类都可以介入、提供建议或做出决策。这在高风险场景下尤为重要。

**代码执行**意味着 AutoGen 内置了代码执行能力。Agent 可以编写代码，然后在一个安全的沙箱环境中执行它。这使得 AutoGen 不仅能"说"，还能"做"——它能够真正地执行数据分析、生成图表、调用 API 等操作。

### 28.1.3 AutoGen 的架构

AutoGen 的架构围绕着三个核心概念构建：Agent、对话和可执行代码。

**Agent** 是 AutoGen 中的基本计算单元。每个 Agent 都有自己的角色定义（系统提示词）、LLM 配置和行为规则。Agent 可以是纯 LLM 驱动的（只做推理和生成），也可以是可执行代码的（能够运行代码来完成实际任务）。

**对话**是 Agent 之间的交互方式。AutoGen 使用消息传递的方式来管理对话——每个 Agent 的输出都会作为下一个 Agent 的输入。对话可以是两两之间的，也可以是群聊（Group Chat）模式下的多方对话。

**可执行代码**是 AutoGen 的独特能力。通过 Docker 沙箱或本地执行环境，Agent 生成的代码可以在安全的环境中运行，并将执行结果反馈给 Agent。

---

## 28.2 安装与配置

### 28.2.1 环境准备

```bash
# 安装 AutoGen
pip install pyautogen

# 如果需要代码执行功能，还需要安装 Docker
# Docker 用于创建安全的代码执行沙箱
pip install docker
```

### 28.2.2 基础配置

```python
"""
第 28 章：AutoGen 框架基础
配置和入门示例
"""
import os

# ============================================================
# AutoGen 配置
# ============================================================

# 方法一：通过配置字典
config_list = [
    {
        "model": "gpt-4o",
        "api_key": os.environ.get("OPENAI_API_KEY"),
    }
]

# 方法二：通过配置文件
# 将以下内容保存为 OAI_CONFIG_LIST 文件：
# [
#     {
#         "model": "gpt-4o",
#         "api_key": "your-api-key-here"
#     }
# ]

# LLM 配置参数
llm_config = {
    "config_list": config_list,
    "temperature": 0.7,
    "timeout": 120,
}

print("AutoGen 配置完成!")
print(f"使用模型: {config_list[0]['model']}")
```

---

## 28.3 AutoGen 核心概念详解

### 28.3.1 AssistantAgent

AssistantAgent 是 AutoGen 中最基础的 Agent 类型。它是一个由 LLM 驱动的对话 Agent，能够理解任务、生成回复，并与其他 Agent 进行对话。

AssistantAgent 的特点是：它不执行代码，只进行推理和对话。它的输出是自然语言文本，可以被其他 Agent 理解和处理。

### 28.3.2 UserProxyAgent

UserProxyAgent 是 AutoGen 中另一个核心的 Agent 类型。它的设计目的是代理人类用户参与对话。UserProxyAgent 有两个关键功能：

1. **接收人类输入**：在需要时请求人类用户提供信息或做出决策。
2. **执行代码**：当对话中出现代码时，UserProxyAgent 会尝试执行它，并将执行结果返回给对话。

UserProxyAgent 的 code_execution_config 参数控制代码执行的行为，比如是否使用 Docker 沙箱、工作目录是什么等。

### 28.3.3 对话流程

AutoGen 的对话流程遵循以下模式：

1. 一个 Agent 发送消息给另一个 Agent
2. 接收消息的 Agent 处理消息并发送回复
3. 如果回复中包含代码且配置了代码执行，代码会被执行
4. 执行结果作为新的消息发送回对话
5. 重复直到任务完成或人类介入

这种对话驱动的模式使得 AutoGen 的行为非常可预测和可调试——你可以通过查看对话日志来了解系统的每一步决策。

```python
"""
第 28 章：AutoGen 核心概念
AssistantAgent 和 UserProxyAgent 的基础使用
"""
import os

# 注意：以下代码需要安装 pyautogen
# pip install pyautogen

# 由于 AutoGen 需要实际的 LLM API 调用，
# 以下代码展示的是代码结构和配置方式
# 实际运行需要有效的 API key


def demo_basic_autogen():
    """
    演示 AutoGen 的基础使用。
    
    这个示例展示了如何：
    1. 配置 LLM
    2. 创建 AssistantAgent 和 UserProxyAgent
    3. 启动两个 Agent 之间的对话
    """
    
    try:
        from autogen import AssistantAgent, UserProxyAgent
        
        # LLM 配置
        config_list = [
            {
                "model": "gpt-4o",
                "api_key": os.environ.get("OPENAI_API_KEY"),
            }
        ]
        
        llm_config = {
            "config_list": config_list,
            "temperature": 0.7,
        }
        
        # 创建 AssistantAgent
        # AssistantAgent 是由 LLM 驱动的对话 Agent
        assistant = AssistantAgent(
            name="assistant",
            system_message="""你是一个有帮助的 AI 助手。
            当你完成任务后，请回复 TERMINATE。""",
            llm_config=llm_config,
        )
        
        # 创建 UserProxyAgent
        # UserProxyAgent 代理人类用户参与对话，
        # 并且可以执行代码
        user_proxy = UserProxyAgent(
            name="user_proxy",
            human_input_mode="NEVER",  # 不请求人类输入
            max_consecutive_auto_reply=10,
            is_termination_msg=lambda x: x.get("content", "").rstrip().endswith("TERMINATE"),
            code_execution_config={
                "work_dir": "coding",
                "use_docker": False,  # 本地执行（生产环境建议使用 Docker）
            },
        )
        
        # 启动对话
        user_proxy.initiate_chat(
            assistant,
            message="""请用 Python 编写一个函数，计算斐波那契数列的前 N 个数。"""
        )
        
    except ImportError:
        print("请先安装 AutoGen: pip install pyautogen")
    except Exception as e:
        print(f"运行出错: {e}")
        print("请确保设置了 OPENAI_API_KEY 环境变量")


# ============================================================
# 不使用 AutoGen 的简化实现
# ============================================================

from openai import OpenAI
import json
import time
from typing import Optional
from dataclasses import dataclass, field


def call_llm(
    messages: list[dict],
    model: str = "gpt-4o",
    temperature: float = 0.7,
    max_tokens: int = 2000
) -> str:
    """统一的 LLM 调用封装"""
    client = OpenAI()
    for attempt in range(3):
        try:
            response = client.chat.completions.create(
                model=model,
                messages=messages,
                temperature=temperature,
                max_tokens=max_tokens
            )
            return response.choices[0].message.content.strip()
        except Exception as e:
            print(f"  [重试 {attempt + 1}/3] LLM 调用失败: {e}")
            if attempt == 2:
                raise
            time.sleep(2 ** attempt)


class SimpleAssistantAgent:
    """
    简化的 AssistantAgent 实现。
    
    这个类模拟了 AutoGen AssistantAgent 的核心行为：
    - 维护系统提示词和对话历史
    - 通过 LLM 生成回复
    - 支持终止条件
    """
    
    def __init__(
        self,
        name: str,
        system_message: str = "",
        model: str = "gpt-4o",
        temperature: float = 0.7
    ):
        self.name = name
        self.system_message = system_message
        self.model = model
        self.temperature = temperature
        self.chat_history: list[dict] = []
        self.silent: bool = False  # 是否静默模式（不打印日志）
    
    def receive(self, message: str, sender: str) -> str:
        """
        接收消息并生成回复。
        
        这是 Agent 的核心交互方法。
        """
        # 添加用户消息到历史
        self.chat_history.append({
            "role": "user",
            "content": f"[来自 {sender}]: {message}"
        })
        
        # 构建消息列表
        messages = []
        if self.system_message:
            messages.append({"role": "system", "content": self.system_message})
        messages.extend(self.chat_history[-10:])
        
        # 调用 LLM 生成回复
        response = call_llm(
            messages,
            model=self.model,
            temperature=self.temperature
        )
        
        # 添加助手回复到历史
        self.chat_history.append({
            "role": "assistant",
            "content": response
        })
        
        if not self.silent:
            print(f"\n[{self.name}]: {response[:200]}...")
        
        return response


class SimpleUserProxyAgent:
    """
    简化的 UserProxyAgent 实现。
    
    模拟了 AutoGen UserProxyAgent 的核心行为：
    - 发起对话
    - 可选的人类输入
    - 代码执行（简化版）
    """
    
    def __init__(
        self,
        name: str,
        human_input_mode: str = "NEVER",
        max_consecutive_auto_reply: int = 10
    ):
        self.name = name
        self.human_input_mode = human_input_mode
        self.max_consecutive_auto_reply = max_consecutive_auto_reply
        self.chat_history: list[dict] = []
    
    def get_human_input(self, prompt: str = "") -> str:
        """请求人类输入"""
        if self.human_input_mode == "NEVER":
            return ""
        elif self.human_input_mode == "TERMINATE":
            return "exit"
        else:
            return input(f"[人类输入] {prompt}: ")
    
    def initiate_chat(self, assistant: SimpleAssistantAgent, message: str):
        """
        发起与 Assistant 的对话。
        
        这是 AutoGen 的核心交互模式——
        UserProxy 发起对话，Assistant 回复，
        两者之间进行多轮自动对话。
        """
        print(f"\n{'='*60}")
        print(f"[{self.name}] 发起对话")
        print(f"消息: {message[:100]}...")
        print(f"{'='*60}")
        
        # 发送初始消息
        current_message = message
        turn = 0
        
        while turn < self.max_consecutive_auto_reply:
            turn += 1
            
            # Assistant 回复
            response = assistant.receive(current_message, self.name)
            
            # 检查终止条件
            if "TERMINATE" in response:
                print(f"\n[对话结束] Assistant 请求终止")
                break
            
            # 检查是否需要人类输入
            if self.human_input_mode == "ALWAYS":
                human_input = self.get_human_input()
                if human_input.lower() in ["exit", "quit", "stop"]:
                    print(f"\n[对话结束] 人类请求终止")
                    break
                current_message = human_input
            else:
                # 自动回复
                current_message = response
        
        print(f"\n[对话结束] 共 {turn} 轮")


def demo_simple_autogen():
    """演示简化的 AutoGen 模式"""
    
    print("\n" + "="*60)
    print("演示：简化版 AutoGen 对话模式")
    print("="*60)
    
    # 创建 Assistant Agent
    assistant = SimpleAssistantAgent(
        name="编程助手",
        system_message="""你是一个专业的 Python 编程助手。
        当用户给你编程任务时，你编写代码来完成它。
        当任务完成后，回复 TERMINATE。""",
        model="gpt-4o",
        temperature=0.7
    )
    
    # 创建 User Proxy Agent
    user_proxy = SimpleUserProxyAgent(
        name="用户",
        human_input_mode="NEVER",
        max_consecutive_auto_reply=5
    )
    
    # 发起对话
    user_proxy.initiate_chat(
        assistant,
        message="请用 Python 编写一个函数，判断一个数是否是回文数。给出函数代码和测试用例。"
    )


if __name__ == "__main__":
    demo_simple_autogen()
```

---

## 28.4 AutoGen 的群聊模式（Group Chat）

### 28.4.1 群聊是什么

在基础模式中，AutoGen 的对话是两两之间的——一个 UserProxy 对应一个 Assistant。但在实际应用中，我们经常需要多个 Agent 之间的协作。AutoGen 的群聊模式（GroupChat）就是为此设计的。

群聊模式允许多个 Agent 在同一个对话中交互。有一个 GroupChatManager 负责管理群聊的流程——决定谁发言、什么时候发言、对话何时结束。

### 28.4.2 群聊的使用场景

群聊模式特别适合以下场景：

**多角色协作**：多个专业角色（如程序员、测试员、架构师）协作完成一个开发任务。

**辩论和审查**：多个 Agent 从不同角度讨论一个问题，通过辩论来提升结论的质量。

**流水线处理**：数据在多个 Agent 之间流转，每个 Agent 负责处理的一个环节。

```python
"""
第 28 章：AutoGen 群聊模式
多个 Agent 在同一个对话中协作
"""
import os
import json
import time
from typing import Optional
from dataclasses import dataclass, field
from openai import OpenAI


# ============================================================
# LLM 调用封装
# ============================================================

def call_llm(
    messages: list[dict],
    model: str = "gpt-4o",
    temperature: float = 0.7,
    max_tokens: int = 2000
) -> str:
    """统一的 LLM 调用封装"""
    client = OpenAI()
    for attempt in range(3):
        try:
            response = client.chat.completions.create(
                model=model,
                messages=messages,
                temperature=temperature,
                max_tokens=max_tokens
            )
            return response.choices[0].message.content.strip()
        except Exception as e:
            print(f"  [重试 {attempt + 1}/3] LLM 调用失败: {e}")
            if attempt == 2:
                raise
            time.sleep(2 ** attempt)


# ============================================================
# 群聊管理器
# ============================================================

@dataclass
class ChatAgent:
    """群聊中的 Agent"""
    name: str
    system_message: str
    model: str = "gpt-4o"
    temperature: float = 0.7
    chat_history: list[dict] = field(default_factory=list)
    
    def respond(self, context: str) -> str:
        """生成回复"""
        messages = [
            {"role": "system", "content": self.system_message},
            {"role": "user", "content": context}
        ]
        
        # 加入最近的对话历史
        messages.extend(self.chat_history[-6:])
        
        response = call_llm(
            messages,
            model=self.model,
            temperature=self.temperature
        )
        
        self.chat_history.append({"role": "assistant", "content": response})
        return response


class GroupChatManager:
    """
    群聊管理器：协调多个 Agent 的群聊对话。
    
    工作方式：
    1. 接收用户的初始消息
    2. 根据策略选择一个 Agent 发言
    3. 将 Agent 的发言广播给其他 Agent
    4. 重复直到满足终止条件
    
    选择策略：
    - round_robin: 轮流发言
    - llm_select: 由一个 LLM 来决定谁应该发言
    - random: 随机选择
    """
    
    def __init__(
        self,
        agents: list[ChatAgent],
        selection_method: str = "round_robin",
        max_turns: int = 10,
        model: str = "gpt-4o"
    ):
        self.agents = {a.name: a for a in agents}
        self.agent_names = [a.name for a in agents]
        self.selection_method = selection_method
        self.max_turns = max_turns
        self.model = model
        self.chat_log: list[dict] = []
        self.current_index = 0  # 轮流发言用的索引
    
    def run_chat(self, topic: str, verbose: bool = True) -> str:
        """
        运行群聊。
        
        Args:
            topic: 群聊话题
            verbose: 是否打印详细过程
        
        Returns:
            最后的发言内容
        """
        if verbose:
            print(f"\n{'='*60}")
            print(f"群聊开始: {topic}")
            print(f"参与者: {', '.join(self.agent_names)}")
            print(f"选择策略: {self.selection_method}")
            print(f"{'='*60}")
        
        current_message = topic
        last_response = ""
        
        for turn in range(self.max_turns):
            # 选择发言者
            speaker_name = self._select_speaker(current_message)
            
            if verbose:
                print(f"\n--- 第 {turn + 1} 轮 ---")
                print(f"发言者: {speaker_name}")
            
            # 获取发言者的回复
            agent = self.agents[speaker_name]
            
            # 构建上下文
            context = f"群聊话题: {topic}\n\n最新消息: {current_message}"
            response = agent.respond(context)
            
            # 记录日志
            self.chat_log.append({
                "turn": turn + 1,
                "speaker": speaker_name,
                "content": response
            })
            
            if verbose:
                print(f"[{speaker_name}]: {response[:200]}...")
            
            # 检查终止条件
            if "TERMINATE" in response or "结束" in response:
                if verbose:
                    print(f"\n[群聊结束] {speaker_name} 请求终止")
                last_response = response
                break
            
            current_message = f"[{speaker_name}]: {response}"
            last_response = response
        
        if verbose:
            print(f"\n{'='*60}")
            print(f"群聊结束，共 {len(self.chat_log)} 轮")
            print(f"{'='*60}")
        
        return last_response
    
    def _select_speaker(self, context: str) -> str:
        """选择下一个发言者"""
        if self.selection_method == "round_robin":
            return self._round_robin_select()
        elif self.selection_method == "llm_select":
            return self._llm_select(context)
        else:
            return self._round_robin_select()
    
    def _round_robin_select(self) -> str:
        """轮流选择"""
        name = self.agent_names[self.current_index % len(self.agent_names)]
        self.current_index += 1
        return name
    
    def _llm_select(self, context: str) -> str:
        """使用 LLM 选择下一个发言者"""
        agent_descriptions = "\n".join([
            f"- {a.name}: {a.system_message[:100]}"
            for a in self.agents.values()
        ])
        
        prompt = f"""根据当前的群聊上下文，选择下一个最适合发言的 Agent。

当前上下文: {context[:300]}

可用的 Agent:
{agent_descriptions}

请只输出应该发言的 Agent 名称，不要其他文字。"""
        
        messages = [{"role": "user", "content": prompt}]
        selected = call_llm(messages, model=self.model, temperature=0.3)
        
        # 确保返回的名称在有效范围内
        for name in self.agent_names:
            if name in selected:
                return name
        
        return self._round_robin_select()


# ============================================================
# 演示：群聊模式
# ============================================================

def demo_group_chat():
    """演示 AutoGen 群聊模式"""
    
    print("\n" + "="*60)
    print("演示：群聊模式")
    print("="*60)
    
    # 定义多个 Agent
    agents = [
        ChatAgent(
            name="产品经理",
            system_message="""你是一位产品经理。在群聊中，你从用户需求和产品价值的角度发言。
            关注功能的实用性和用户体验。如果讨论已经充分，请说 TERMINATE。""",
            temperature=0.7
        ),
        ChatAgent(
            name="技术架构师",
            system_message="""你是一位技术架构师。在群聊中，你从技术可行性和系统设计的角度发言。
            关注技术方案的合理性和可扩展性。如果讨论已经充分，请说 TERMINATE。""",
            temperature=0.6
        ),
        ChatAgent(
            name="用户体验设计师",
            system_message="""你是一位用户体验设计师。在群聊中，你从用户界面和交互体验的角度发言。
            关注设计的美观性和易用性。如果讨论已经充分，请说 TERMINATE。""",
            temperature=0.7
        ),
    ]
    
    # 创建群聊管理器
    manager = GroupChatManager(
        agents=agents,
        selection_method="round_robin",
        max_turns=6
    )
    
    # 运行群聊
    result = manager.run_chat(
        topic="我们应该如何设计一个新的任务管理应用？请讨论核心功能和设计理念。",
        verbose=True
    )
    
    print(f"\n--- 最终结果 ---")
    print(result[:300] + "...")


if __name__ == "__main__":
    demo_group_chat()
```

---

## 28.5 AutoGen 中的代码执行

### 28.5.1 代码执行的重要性

AutoGen 最独特的能力之一是内置的代码执行功能。不同于大多数 Multi-Agent 框架只做"对话"，AutoGen 中的 Agent 可以真正地编写和执行代码。

这使得 AutoGen 非常适合以下场景：

**数据分析**：Agent 编写 Python 代码来处理和分析数据，生成统计报告和可视化图表。

**科学计算**：Agent 编写数学计算代码来解决复杂的科学问题。

**自动化测试**：Agent 编写测试代码来验证其他 Agent 的产出。

**系统原型**：Agent 编写代码来快速构建系统原型。

### 28.5.2 安全的代码执行

代码执行涉及安全风险。AutoGen 通过以下方式来保障安全：

**Docker 沙箱**：代码在一个隔离的 Docker 容器中执行，即使代码有问题也不会影响宿主系统。

**超时控制**：设置执行超时，防止恶意代码或死循环。

**文件系统隔离**：限制代码可以访问的文件和目录。

```python
"""
第 28 章：AutoGen 代码执行
安全地执行 Agent 生成的代码
"""
import os
import json
import time
import tempfile
import subprocess
from typing import Optional
from dataclasses import dataclass


# ============================================================
# 代码执行沙箱
# ============================================================

@dataclass
class CodeExecutionResult:
    """代码执行结果"""
    success: bool
    output: str
    error: str
    execution_time: float
    code: str


class CodeExecutionSandbox:
    """
    代码执行沙箱：在安全的环境中执行 Python 代码。
    
    支持三种执行模式：
    1. 本地执行：直接在本地 Python 环境中执行（不安全，仅用于测试）
    2. Docker 执行：在 Docker 容器中执行（推荐的生产环境方案）
    3. 子进程执行：使用子进程执行，有一定的隔离性
    """
    
    def __init__(self, work_dir: str = None, timeout: int = 30):
        self.work_dir = work_dir or tempfile.mkdtemp()
        self.timeout = timeout
        self.execution_history: list[CodeExecutionResult] = []
    
    def execute(self, code: str, mode: str = "subprocess") -> CodeExecutionResult:
        """
        执行代码。
        
        Args:
            code: 要执行的 Python 代码
            mode: 执行模式
        
        Returns:
            执行结果
        """
        print(f"\n  [沙箱] 执行代码:")
        print(f"  {'-'*40}")
        for line in code.split('\n')[:10]:
            print(f"  | {line}")
        if len(code.split('\n')) > 10:
            print(f"  | ... ({len(code.split(chr(10)))} 行)")
        print(f"  {'-'*40}")
        
        start_time = time.time()
        
        if mode == "subprocess":
            result = self._execute_subprocess(code)
        elif mode == "local":
            result = self._execute_local(code)
        else:
            result = self._execute_subprocess(code)
        
        result.execution_time = time.time() - start_time
        result.code = code
        
        self.execution_history.append(result)
        
        if result.success:
            print(f"  [沙箱] 执行成功 ({result.execution_time:.2f}s)")
            if result.output:
                print(f"  输出: {result.output[:200]}")
        else:
            print(f"  [沙箱] 执行失败 ({result.execution_time:.2f}s)")
            print(f"  错误: {result.error[:200]}")
        
        return result
    
    def _execute_subprocess(self, code: str) -> CodeExecutionResult:
        """使用子进程执行代码"""
        try:
            # 将代码写入临时文件
            code_file = os.path.join(self.work_dir, "exec_code.py")
            with open(code_file, "w", encoding="utf-8") as f:
                f.write(code)
            
            # 使用子进程执行
            proc = subprocess.run(
                ["python", code_file],
                capture_output=True,
                text=True,
                timeout=self.timeout,
                cwd=self.work_dir
            )
            
            return CodeExecutionResult(
                success=proc.returncode == 0,
                output=proc.stdout,
                error=proc.stderr,
                execution_time=0,
                code=""
            )
            
        except subprocess.TimeoutExpired:
            return CodeExecutionResult(
                success=False,
                output="",
                error=f"执行超时（{self.timeout}秒）",
                execution_time=0,
                code=""
            )
        except Exception as e:
            return CodeExecutionResult(
                success=False,
                output="",
                error=str(e),
                execution_time=0,
                code=""
            )
    
    def _execute_local(self, code: str) -> CodeExecutionResult:
        """在本地执行代码（不安全，仅用于测试）"""
        try:
            # 捕获标准输出
            import io
            import sys
            
            old_stdout = sys.stdout
            sys.stdout = captured_output = io.StringIO()
            
            exec_globals = {"__name__": "__main__"}
            exec(code, exec_globals)
            
            sys.stdout = old_stdout
            output = captured_output.getvalue()
            
            return CodeExecutionResult(
                success=True,
                output=output,
                error="",
                execution_time=0,
                code=""
            )
            
        except Exception as e:
            import sys
            sys.stdout = sys.__stdout__
            return CodeExecutionResult(
                success=False,
                output="",
                error=str(e),
                execution_time=0,
                code=""
            )


# ============================================================
# 代码生成与执行的 Agent
# ============================================================

class CodeAgent:
    """
    代码 Agent：能够编写和执行代码的智能体。
    
    结合了 LLM 的推理能力和实际的代码执行能力。
    工作流程：
    1. 理解编程任务
    2. 生成 Python 代码
    3. 执行代码
    4. 根据执行结果修正代码
    """
    
    def __init__(
        self,
        name: str = "代码Agent",
        model: str = "gpt-4o",
        max_retries: int = 2
    ):
        self.name = name
        self.model = model
        self.max_retries = max_retries
        self.sandbox = CodeExecutionSandbox()
    
    def solve(self, task: str, verbose: bool = True) -> str:
        """
        执行编程任务。
        
        Args:
            task: 编程任务描述
            verbose: 是否打印详细过程
        
        Returns:
            任务执行结果
        """
        if verbose:
            print(f"\n{'='*60}")
            print(f"[{self.name}] 开始执行任务: {task[:80]}...")
            print(f"{'='*60}")
        
        # 生成初始代码
        code = self._generate_code(task)
        
        for attempt in range(self.max_retries + 1):
            if verbose:
                print(f"\n--- 尝试 {attempt + 1}/{self.max_retries + 1} ---")
            
            # 执行代码
            result = self.sandbox.execute(code)
            
            if result.success:
                if verbose:
                    print(f"\n  ✅ 任务完成!")
                return f"代码执行成功:\n{result.output}"
            
            # 代码执行失败，尝试修正
            if attempt < self.max_retries:
                if verbose:
                    print(f"\n  ⚠️ 代码执行失败，正在修正...")
                code = self._fix_code(task, code, result.error)
        
        return f"经过 {self.max_retries + 1} 次尝试后仍失败。最后的错误: {result.error}"
    
    def _generate_code(self, task: str) -> str:
        """生成 Python 代码"""
        prompt = f"""请根据以下任务编写 Python 代码：

任务: {task}

要求：
1. 代码应该是完整的、可直接运行的
2. 包含必要的 import 语句
3. 包含适当的错误处理
4. 使用 print() 输出结果
5. 只输出 Python 代码，不要其他文字

```python"""
        
        messages = [{"role": "user", "content": prompt}]
        response = call_llm(messages, model=self.model, temperature=0.3)
        
        # 提取代码
        if "```python" in response:
            code = response.split("```python")[1].split("```")[0]
        elif "```" in response:
            code = response.split("```")[1].split("```")[0]
        else:
            code = response
        
        return code.strip()
    
    def _fix_code(self, task: str, original_code: str, error: str) -> str:
        """修正有问题的代码"""
        prompt = f"""以下代码执行时出现了错误，请修正。

原始任务: {task}

原始代码:
```python
{original_code}
```

错误信息:
{error}

请输出修正后的完整代码。只输出 Python 代码，不要其他文字。"""
        
        messages = [{"role": "user", "content": prompt}]
        response = call_llm(messages, model=self.model, temperature=0.3)
        
        # 提取代码
        if "```python" in response:
            code = response.split("```python")[1].split("```")[0]
        elif "```" in response:
            code = response.split("```")[1].split("```")[0]
        else:
            code = response
        
        return code.strip()


# ============================================================
# 演示：代码生成与执行
# ============================================================

def demo_code_execution():
    """演示代码生成与执行"""
    
    print("\n" + "="*60)
    print("演示：代码生成与执行")
    print("="*60)
    
    code_agent = CodeAgent()
    
    # 执行一个编程任务
    result = code_agent.solve(
        "编写一个 Python 函数，找出给定列表中的所有质数，并测试该函数"
    )
    
    print(f"\n--- 最终结果 ---")
    print(result)


if __name__ == "__main__":
    demo_code_execution()
```

---

## 28.6 AutoGen 的实际应用案例

### 28.6.1 案例：自动化数据分析流水线

AutoGen 非常适合构建自动化的数据分析流水线。在这样的系统中：

1. **数据获取 Agent**：编写代码从数据库或 API 获取数据
2. **数据清洗 Agent**：编写代码清洗和预处理数据
3. **分析 Agent**：编写代码进行统计分析和数据挖掘
4. **可视化 Agent**：编写代码生成数据可视化图表
5. **报告 Agent**：综合分析结果撰写分析报告

每个 Agent 都能够真正地编写和执行代码，而不仅仅是进行对话。这使得整个流水线能够真正地"动手"完成数据分析任务。

### 28.6.2 案例：代码审查与修复

AutoGen 可以构建自动化的代码审查系统：

1. **审查 Agent**：阅读代码，发现潜在问题
2. **修复 Agent**：根据审查结果编写修复代码
3. **测试 Agent**：编写测试代码来验证修复是否正确
4. **验证 Agent**：运行测试并确认所有问题都已解决

这种循环审查-修复-测试的模式能够自动提升代码质量。

---

## 28.7 常见坑与解决方案

### 坑 1：API 费用高昂

**问题描述**：AutoGen 的对话驱动模式导致 LLM 调用次数很多，每次对话都是一次 API 调用，费用可能超出预期。

**解决方案**：设置 max_consecutive_auto_reply 限制自动回复轮数。使用更经济的模型（如 GPT-3.5-turbo）处理简单的对话轮次。监控 API 用量。

### 坑 2：对话陷入循环

**问题描述**：Agent 之间的对话可能陷入无意义的循环，双方反复说同样的话。

**解决方案**：设置 max_turns 限制总对话轮数。在系统提示词中明确要求 Agent 在完成任务后说 TERMINATE。

### 坑 3：代码执行安全风险

**问题描述**：Agent 生成的代码可能包含危险操作（如删除文件、访问敏感数据等）。

**解决方案**：始终使用 Docker 沙箱执行代码。设置严格的文件系统访问限制。监控代码执行过程中的系统资源使用。

### 坑 4：Agent 角色混淆

**问题描述**：在群聊模式中，Agent 可能会偏离自己的角色，做出不符合角色定位的发言。

**解决方案**：在系统提示词中强调角色约束。使用更强的模型来维持角色一致性。

### 坑 5：调试困难

**问题描述**：当对话链较长时，追踪问题的根源变得困难。

**解决方案**：实现详细的对话日志记录。使用 AutoGen 的内置日志功能。在关键节点添加状态检查。

---

## 28.8 练习题

### 练习 1：基本对话模式

**难度：★☆☆☆☆**

使用 AutoGen 的 AssistantAgent 和 UserProxyAgent 创建一个简单的问答系统。测试不同的终止条件配置。

### 练习 2：群聊协作

**难度：★★☆☆☆**

使用群聊模式，让 3 个 Agent 协作完成一个翻译任务：一个 Agent 负责直译，一个负责润色，一个负责校对。

### 练习 3：代码执行

**难度：★★★☆☆**

使用 AutoGen 的代码执行功能，让 Agent 自动分析一个 CSV 数据文件。Agent 应该能够生成数据读取、清洗、分析和可视化的完整代码。

### 练习 4：人在回路

**难度：★★★☆☆**

实现一个支持人类实时介入的 AutoGen 系统。在关键决策点，Agent 会暂停并等待人类的输入。

### 练习 5：错误恢复

**难度：★★★★☆**

为 AutoGen 系统添加错误恢复机制。当代码执行失败时，系统能够自动诊断问题、修正代码并重试。

---

## 28.9 实战任务

### 任务：构建 AutoGen 自动化测试系统

**目标**：使用 AutoGen 构建一个自动化的 Python 代码测试系统。

**系统设计**：
1. **代码编写 Agent**：根据需求编写 Python 函数
2. **测试编写 Agent**：为编写的函数编写单元测试
3. **执行 Agent**：运行代码和测试
4. **修复 Agent**：根据测试失败信息修正代码

**功能要求**：
1. 支持多种类型的编程任务
2. 自动化的代码-测试-修复循环
3. 完整的执行日志
4. 代码完整可运行

---

## 28.10 本章小结

本章深入学习了 AutoGen 框架——微软开源的对话驱动 Multi-Agent 开发框架。AutoGen 的核心理念是通过 Agent 之间的自然语言对话来协作完成任务。

我们学习了 AutoGen 的三大核心概念：对话驱动的交互模式、人类参与的灵活机制、以及安全的代码执行能力。通过 AssistantAgent 和 UserProxyAgent 的组合，我们可以构建从简单到复杂的 Multi-Agent 系统。

群聊模式（GroupChat）让多个 Agent 能够在同一个对话中协作，支持轮流发言和智能选择。代码执行能力使得 AutoGen 不仅能"说"，还能"做"——真正地编写和运行代码来完成实际任务。

AutoGen 的优势在于它的灵活性和安全性——既支持纯对话模式，也支持代码执行模式；既有自动化的对话流转，也支持人类随时介入。

> "AutoGen 的设计哲学是：让对话成为协作的语言，让代码成为行动的手段。两者结合，Agent 就能够像人类团队一样高效地工作。"

---

**下一章预告**：在第 29 章中，我们将学习另一个流行的 Multi-Agent 框架——CrewAI。CrewAI 以"角色"和"任务"为核心，提供了更加结构化的 Multi-Agent 协作方式。敬请期待。
