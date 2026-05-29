# 第 34 章：Agent 单元测试 —— 测试 Agent 的代码部分

> **本章定位：** 单元测试是软件质量的基石，但 Agent 的概率性输出让传统测试方法遇到了挑战。本章将系统性地解决这个问题——如何为 Agent 的确定性代码部分编写单元测试，如何处理 LLM 输出的不确定性，以及如何构建可维护的 Agent 测试套件。

---

## 学习目标

完成本章学习后，你将能够：

1. **区分 Agent 中可测试和难测试的部分** —— 能识别 Agent 代码中哪些组件适合传统单元测试，哪些需要特殊处理
2. **Mock LLM 调用进行确定性测试** —— 能使用 mock 技术隔离 LLM 调用，对 Agent 的逻辑部分进行确定性测试
3. **测试 Agent 的工具组件** —— 能为 Agent 的每个工具编写独立的单元测试
4. **处理 LLM 输出的不确定性** —— 能用模糊匹配、断言模式等技术测试概率性输出
5. **组织和运行 Agent 测试套件** —— 能用 pytest 搭建一个可维护的 Agent 测试项目

## 核心问题

1. **Agent 的哪部分适合单元测试，哪部分不适合？** 传统软件的所有逻辑都应该能被单元测试覆盖，但 Agent 的核心——LLM 推理——天然是不确定的，这如何处理？
2. **如何 mock LLM 调用而不失去测试的真实性？** Mock 太简单可能遗漏真实场景中的问题，Mock 太复杂又失去了测试的确定性。
3. **测试覆盖率对 Agent 有意义吗？** 传统的代码覆盖率指标对 Agent 还适用吗？应该关注什么样的覆盖率？

---

## 34.1 Agent 的可测试性分析

### 34.1.1 Agent 的层次结构

要理解 Agent 的测试策略，首先要理解 Agent 的层次结构。一个典型的 Agent 可以分为以下几层：

**确定性逻辑层：** 包括工具函数（搜索、计算、API 调用）、输入解析器、输出格式化器、路由逻辑等。这些组件的行为是确定性的——给定相同的输入，永远返回相同的输出。它们完全适合传统单元测试。

**概率性推理层：** 包括 LLM 调用、决策逻辑、计划生成等。这些组件的核心依赖 LLM 的输出，而 LLM 的输出天然是概率性的。它们需要特殊的测试策略。

**集成层：** 包括 LLM 与工具的协调、多步推理的流程控制、上下文管理等。这些组件的测试需要同时考虑确定性和概率性部分。

### 34.1.2 测试策略

对于确定性逻辑层，使用标准的单元测试即可。对于概率性推理层，使用 mock + 模糊断言的策略。对于集成层，使用端到端测试，结合 mock 和真实 LLM 调用。

核心原则是：**尽量把逻辑推到确定性层。** 如果你能把 LLM 的决策逻辑转换成确定性的规则（比如用正则解析 LLM 输出来决定调用哪个工具），那就这样做——它不仅更好测试，而且更可靠、更可控。

---

## 34.2 Mock LLM 调用进行测试

### 34.2.1 构建可测试的 Agent

在写测试之前，我们首先需要构建一个易于测试的 Agent。关键设计原则是依赖注入——让 Agent 的 LLM 客户端可以被替换，这样测试时可以注入 mock 对象。

```python
import os
import json
import time
import re
from typing import List, Dict, Any, Optional, Callable, Protocol
from dataclasses import dataclass

# ============================================================
# 1. 定义协议接口（便于 Mock）
# ============================================================

class LLMClient(Protocol):
    """LLM 客户端协议"""
    def chat(self, messages: List[Dict], temperature: float = 0.7) -> Dict:
        """发送聊天请求"""
        ...

class Tool(Protocol):
    """工具协议"""
    name: str
    description: str
    
    def execute(self, **kwargs) -> str:
        """执行工具"""
        ...


# ============================================================
# 2. 可测试的 Agent 实现
# ============================================================

@dataclass
class AgentConfig:
    """Agent 配置"""
    model: str = "gpt-4o"
    temperature: float = 0.7
    max_iterations: int = 5
    system_prompt: str = "你是一个智能助手。"

class TestableAgent:
    """
    可测试的 Agent
    
    关键设计：
    1. LLM 客户端通过构造函数注入，测试时可以替换为 mock
    2. 工具通过注册方式添加，可以独立测试
    3. 决策逻辑和 LLM 调用分离
    """
    
    def __init__(self, llm_client: LLMClient, config: AgentConfig = None):
        self.llm = llm_client
        self.config = config or AgentConfig()
        self.tools: Dict[str, Callable] = {}
        self._iteration_count = 0
    
    def register_tool(self, name: str, func: Callable, description: str = ""):
        """注册工具"""
        self.tools[name] = {
            "func": func,
            "description": description,
        }
    
    def _build_system_prompt(self) -> str:
        """构建系统提示"""
        tool_descriptions = "\n".join([
            f"- {name}: {info['description']}"
            for name, info in self.tools.items()
        ])
        
        return f"""{self.config.system_prompt}

你可以使用以下工具：
{tool_descriptions}

如果需要使用工具，请用以下格式：
ACTION: <工具名>
ARGS: <JSON格式的参数>

如果不需要工具，直接回答用户。"""
    
    def _parse_llm_output(self, text: str) -> Dict:
        """
        解析 LLM 输出，决定是工具调用还是直接回答
        
        这个函数是确定性的，可以用标准单元测试。
        """
        # 尝试匹配工具调用模式
        action_match = re.search(r"ACTION:\s*(\w+)", text)
        args_match = re.search(r"ARGS:\s*(\{.*?\})", text, re.DOTALL)
        
        if action_match and args_match:
            tool_name = action_match.group(1)
            try:
                args = json.loads(args_match.group(1))
            except json.JSONDecodeError:
                args = {}
            
            return {
                "type": "tool_call",
                "tool_name": tool_name,
                "arguments": args,
            }
        
        return {
            "type": "final_answer",
            "content": text,
        }
    
    def _execute_tool(self, tool_name: str, arguments: Dict) -> str:
        """执行工具调用"""
        if tool_name not in self.tools:
            return f"错误: 未知工具 '{tool_name}'"
        
        try:
            return self.tools[tool_name]["func"](**arguments)
        except Exception as e:
            return f"工具执行错误: {e}"
    
    def run(self, user_input: str) -> str:
        """运行 Agent"""
        messages = [
            {"role": "system", "content": self._build_system_prompt()},
            {"role": "user", "content": user_input},
        ]
        
        self._iteration_count = 0
        
        while self._iteration_count < self.config.max_iterations:
            self._iteration_count += 1
            
            response = self.llm.chat(messages, self.config.temperature)
            response_text = response.get("content", "")
            
            parsed = self._parse_llm_output(response_text)
            
            if parsed["type"] == "final_answer":
                return parsed["content"]
            
            # 工具调用
            tool_result = self._execute_tool(parsed["tool_name"], parsed["arguments"])
            
            messages.append({"role": "assistant", "content": response_text})
            messages.append({"role": "user", "content": f"工具结果: {tool_result}\n\n请根据结果回答用户。"})
        
        return "抱歉，我无法完成这个任务。"


# ============================================================
# 3. Mock LLM 客户端
# ============================================================

class MockLLMClient:
    """
    Mock LLM 客户端
    
    支持多种响应模式：
    - 固定响应
    - 调用链响应（第1次返回X，第2次返回Y）
    - 条件响应（根据输入返回不同输出）
    """
    
    def __init__(self):
        self._call_count = 0
        self._call_history: List[Dict] = []
        self._fixed_response: Optional[str] = None
        self._response_chain: List[str] = []
        self._response_func: Optional[Callable] = None
    
    def set_fixed_response(self, response: str):
        """设置固定响应"""
        self._fixed_response = response
        self._response_chain = []
        self._response_func = None
    
    def set_response_chain(self, responses: List[str]):
        """设置调用链响应"""
        self._response_chain = responses
        self._fixed_response = None
        self._response_func = None
    
    def set_response_func(self, func: Callable):
        """设置自定义响应函数"""
        self._response_func = func
        self._fixed_response = None
        self._response_chain = []
    
    def chat(self, messages: List[Dict], temperature: float = 0.7) -> Dict:
        """模拟 LLM 聊天"""
        self._call_count += 1
        self._call_history.append({
            "messages": messages,
            "temperature": temperature,
            "call_number": self._call_count,
        })
        
        if self._response_func:
            content = self._response_func(messages)
        elif self._response_chain:
            idx = min(self._call_count - 1, len(self._response_chain) - 1)
            content = self._response_chain[idx]
        elif self._fixed_response:
            content = self._fixed_response
        else:
            content = "默认 Mock 响应"
        
        return {"content": content}
    
    @property
    def call_count(self) -> int:
        return self._call_count
    
    @property
    def last_messages(self) -> List[Dict]:
        """获取最后一次调用的消息"""
        if self._call_history:
            return self._call_history[-1]["messages"]
        return []
    
    def reset(self):
        """重置状态"""
        self._call_count = 0
        self._call_history = []
```

### 34.2.2 确定性组件的单元测试

```python
import pytest
from typing import Dict, List

# ============================================================
# 测试 Agent 的确定性组件
# ============================================================

class TestAgentParsing:
    """测试 Agent 的输出解析逻辑"""
    
    def setup_method(self):
        """每个测试方法运行前的设置"""
        self.agent = TestableAgent(
            llm_client=MockLLMClient(),
            config=AgentConfig(),
        )
    
    def test_parse_tool_call_basic(self):
        """测试基本的工具调用解析"""
        llm_output = """我需要搜索相关信息。
ACTION: search
ARGS: {"query": "Python 测试"}"""
        
        result = self.agent._parse_llm_output(llm_output)
        
        assert result["type"] == "tool_call"
        assert result["tool_name"] == "search"
        assert result["arguments"] == {"query": "Python 测试"}
    
    def test_parse_tool_call_no_args(self):
        """测试无参数的工具调用"""
        llm_output = """ACTION: get_time
ARGS: {}"""
        
        result = self.agent._parse_llm_output(llm_output)
        
        assert result["type"] == "tool_call"
        assert result["tool_name"] == "get_time"
        assert result["arguments"] == {}
    
    def test_parse_final_answer(self):
        """测试直接回答的解析"""
        llm_output = "Python 是一种广泛使用的编程语言。"
        
        result = self.agent._parse_llm_output(llm_output)
        
        assert result["type"] == "final_answer"
        assert result["content"] == "Python 是一种广泛使用的编程语言。"
    
    def test_parse_invalid_json_args(self):
        """测试无效 JSON 参数的处理"""
        llm_output = """ACTION: search
ARGS: {invalid json}"""
        
        result = self.agent._parse_llm_output(llm_output)
        
        assert result["type"] == "tool_call"
        assert result["arguments"] == {}  # 应该优雅地处理
    
    def test_parse_multiple_tool_calls(self):
        """测试只提取第一个工具调用"""
        llm_output = """ACTION: search
ARGS: {"query": "test"}
ACTION: calculate
ARGS: {"expression": "1+1"}"""
        
        result = self.agent._parse_llm_output(llm_output)
        
        assert result["type"] == "tool_call"
        assert result["tool_name"] == "search"  # 只取第一个


class TestAgentTools:
    """测试 Agent 的工具组件"""
    
    def setup_method(self):
        self.mock_llm = MockLLMClient()
        self.agent = TestableAgent(llm_client=self.mock_llm)
        
        # 注册测试工具
        self.agent.register_tool(
            "search",
            lambda query="": f"搜索结果: {query}",
            "搜索信息"
        )
        self.agent.register_tool(
            "calculate",
            lambda expression="": str(eval(expression)),
            "计算数学表达式"
        )
        self.agent.register_tool(
            "echo",
            lambda text="": text,
            "回显文本"
        )
    
    def test_search_tool(self):
        """测试搜索工具"""
        result = self.agent._execute_tool("search", {"query": "Python"})
        assert "Python" in result
        assert "搜索结果" in result
    
    def test_calculate_tool(self):
        """测试计算工具"""
        result = self.agent._execute_tool("calculate", {"expression": "2 + 3"})
        assert result == "5"
    
    def test_calculate_tool_complex(self):
        """测试复杂计算"""
        result = self.agent._execute_tool("calculate", {"expression": "(10 + 5) * 2"})
        assert result == "30"
    
    def test_unknown_tool(self):
        """测试未知工具"""
        result = self.agent._execute_tool("unknown_tool", {})
        assert "错误" in result
        assert "未知工具" in result
    
    def test_tool_execution_error(self):
        """测试工具执行错误"""
        def failing_tool():
            raise ValueError("工具内部错误")
        
        self.agent.register_tool("fail", failing_tool)
        result = self.agent._execute_tool("fail", {})
        assert "错误" in result
    
    def test_tool_registration(self):
        """测试工具注册"""
        assert "search" in self.agent.tools
        assert "calculate" in self.agent.tools
        assert self.agent.tools["search"]["description"] == "搜索信息"


class TestAgentSystemPrompt:
    """测试系统提示的构建"""
    
    def setup_method(self):
        self.agent = TestableAgent(
            llm_client=MockLLMClient(),
            config=AgentConfig(system_prompt="你是一个测试助手。"),
        )
        self.agent.register_tool("search", lambda query="": "", "搜索")
        self.agent.register_tool("calculate", lambda expression="": "", "计算")
    
    def test_prompt_contains_tools(self):
        """测试提示包含工具描述"""
        prompt = self.agent._build_system_prompt()
        assert "search" in prompt
        assert "calculate" in prompt
        assert "搜索" in prompt
        assert "计算" in prompt
    
    def test_prompt_contains_custom_system_message(self):
        """测试提示包含自定义系统消息"""
        prompt = self.agent._build_system_prompt()
        assert "你是一个测试助手。" in prompt
    
    def test_prompt_contains_action_format(self):
        """测试提示包含动作格式说明"""
        prompt = self.agent._build_system_prompt()
        assert "ACTION:" in prompt
        assert "ARGS:" in prompt


class TestAgentWithMockLLM:
    """使用 Mock LLM 测试 Agent 的完整流程"""
    
    def setup_method(self):
        self.mock_llm = MockLLMClient()
        self.agent = TestableAgent(
            llm_client=self.mock_llm,
            config=AgentConfig(max_iterations=3),
        )
        self.agent.register_tool(
            "search",
            lambda query="": f"搜索 '{query}' 的结果: 相关信息...",
            "搜索信息"
        )
    
    def test_direct_answer(self):
        """测试 LLM 直接回答（不调用工具）"""
        self.mock_llm.set_fixed_response("Python 是一种编程语言。")
        
        result = self.agent.run("什么是 Python？")
        
        assert result == "Python 是一种编程语言。"
        assert self.mock_llm.call_count == 1
    
    def test_tool_call_then_answer(self):
        """测试工具调用后回答"""
        # 第一次返回工具调用，第二次返回最终回答
        self.mock_llm.set_response_chain([
            'ACTION: search\nARGS: {"query": "Python 版本"}',
            "Python 当前最新版本是 3.12。",
        ])
        
        result = self.agent.run("Python 最新版本是什么？")
        
        assert "3.12" in result
        assert self.mock_llm.call_count == 2
    
    def test_max_iterations_limit(self):
        """测试最大迭代次数限制"""
        # 始终返回工具调用，但工具结果不会让 LLM 停止
        self.mock_llm.set_fixed_response(
            'ACTION: search\nARGS: {"query": "test"}'
        )
        
        result = self.agent.run("测试问题")
        
        assert "抱歉" in result  # 应该返回超限提示
        assert self.mock_llm.call_count == 3  # 应该正好调用了 max_iterations 次
    
    def test_unknown_tool_handling(self):
        """测试 Agent 处理未知工具"""
        self.mock_llm.set_response_chain([
            'ACTION: unknown_tool\nARGS: {}',
            "我无法使用那个工具，让我直接回答。",
        ])
        
        result = self.agent.run("测试")
        
        # Agent 应该能处理未知工具的错误并继续
        assert self.mock_llm.call_count == 2


# ============================================================
# 运行测试
# ============================================================

def run_all_tests():
    """运行所有测试"""
    print("运行 Agent 单元测试...")
    print("=" * 60)
    
    # 手动运行测试（不依赖 pytest）
    test_classes = [
        TestAgentParsing(),
        TestAgentTools(),
        TestAgentSystemPrompt(),
        TestAgentWithMockLLM(),
    ]
    
    total = 0
    passed = 0
    failed = 0
    errors = []
    
    for test_class in test_classes:
        class_name = test_class.__class__.__name__
        print(f"\n[{class_name}]")
        
        # 获取所有测试方法
        test_methods = [m for m in dir(test_class) if m.startswith("test_")]
        
        for method_name in test_methods:
            total += 1
            method = getattr(test_class, method_name)
            
            try:
                # 运行 setup
                if hasattr(test_class, "setup_method"):
                    test_class.setup_method()
                
                # 运行测试
                method()
                passed += 1
                print(f"  PASS: {method_name}")
            except AssertionError as e:
                failed += 1
                errors.append((class_name, method_name, str(e)))
                print(f"  FAIL: {method_name} - {e}")
            except Exception as e:
                failed += 1
                errors.append((class_name, method_name, str(e)))
                print(f"  ERROR: {method_name} - {e}")
    
    print(f"\n{'='*60}")
    print(f"总计: {total}, 通过: {passed}, 失败: {failed}")
    
    if errors:
        print(f"\n失败详情:")
        for cls, method, err in errors:
            print(f"  {cls}.{method}: {err}")
    
    return failed == 0


if __name__ == "__main__":
    success = run_all_tests()
    exit(0 if success else 1)
```

---

## 34.3 处理 LLM 输出不确定性

### 34.3.1 模糊断言策略

当测试涉及 LLM 输出时，精确匹配通常不可行。以下是一些实用的模糊断言策略。

```python
import re
from typing import List, Set

class FuzzyAssertions:
    """
    模糊断言工具集
    
    提供多种方式来测试 LLM 的概率性输出。
    """
    
    @staticmethod
    def contains_any(text: str, keywords: List[str], min_match: int = 1) -> bool:
        """检查文本是否包含至少 N 个关键词"""
        matched = sum(1 for kw in keywords if kw.lower() in text.lower())
        return matched >= min_match
    
    @staticmethod
    def contains_all(text: str, keywords: List[str]) -> bool:
        """检查文本是否包含所有关键词"""
        return all(kw.lower() in text.lower() for kw in keywords)
    
    @staticmethod
    def matches_pattern(text: str, pattern: str) -> bool:
        """检查文本是否匹配正则模式"""
        return bool(re.search(pattern, text, re.IGNORECASE))
    
    @staticmethod
    def length_between(text: str, min_len: int, max_len: int) -> bool:
        """检查文本长度是否在范围内"""
        return min_len <= len(text) <= max_len
    
    @staticmethod
    def has_structure(text: str) -> bool:
        """检查文本是否有结构（分段、列表等）"""
        return "\n" in text or re.search(r"[\d]+\.", text) or "-" in text
    
    @staticmethod
    def sentiment_score(text: str) -> float:
        """简单的正向情感评分（0-1）"""
        positive_words = {"好", "棒", "优秀", "出色", "感谢", "帮助", "支持", "成功"}
        negative_words = {"差", "坏", "失败", "错误", "问题", "抱歉", "无法", "不能"}
        
        pos_count = sum(1 for w in positive_words if w in text)
        neg_count = sum(1 for w in negative_words if w in text)
        
        total = pos_count + neg_count
        if total == 0:
            return 0.5
        return pos_count / total
    
    @staticmethod
    def answer_quality_check(answer: str, question: str, min_score: float = 0.5) -> Dict:
        """
        综合回答质量检查
        
        返回各项检查的结果和总分。
        """
        checks = {}
        scores = {}
        
        # 1. 回答不是空的
        checks["not_empty"] = len(answer.strip()) > 0
        scores["not_empty"] = 1.0 if checks["not_empty"] else 0.0
        
        # 2. 长度合理
        checks["reasonable_length"] = FuzzyAssertions.length_between(answer, 10, 2000)
        scores["reasonable_length"] = 1.0 if checks["reasonable_length"] else 0.5
        
        # 3. 包含问题中的关键词（至少 1 个）
        q_words = [w for w in question.split() if len(w) > 1]
        checks["contains_question_keywords"] = FuzzyAssertions.contains_any(answer, q_words, min_match=1)
        scores["contains_question_keywords"] = 1.0 if checks["contains_question_keywords"] else 0.3
        
        # 4. 没有明显的错误标记
        error_markers = ["错误", "失败", "异常", "无法处理"]
        checks["no_error_markers"] = not FuzzyAssertions.contains_any(answer, error_markers)
        scores["no_error_markers"] = 1.0 if checks["no_error_markers"] else 0.0
        
        # 5. 有结构（可选加分）
        checks["has_structure"] = FuzzyAssertions.has_structure(answer)
        scores["has_structure"] = 1.0 if checks["has_structure"] else 0.8
        
        total_score = sum(scores.values()) / len(scores)
        
        return {
            "checks": checks,
            "scores": scores,
            "total_score": total_score,
            "passed": total_score >= min_score,
        }


# ============================================================
# 使用模糊断言的测试示例
# ============================================================

def test_llm_output_fuzzy():
    """使用模糊断言测试 LLM 输出"""
    fuzzy = FuzzyAssertions()
    
    # 示例：测试 Agent 对"什么是 Python"的回答
    mock_agent_output = "Python 是一种高级编程语言，由 Guido van Rossum 于 1991 年创建。它以简洁易读的语法著称。"
    question = "什么是 Python？"
    
    # 检查回答质量
    quality = fuzzy.answer_quality_check(mock_agent_output, question)
    
    print(f"回答质量检查:")
    print(f"  总分: {quality['total_score']:.2f}")
    print(f"  是否通过: {quality['passed']}")
    for check, passed in quality['checks'].items():
        print(f"  {check}: {'PASS' if passed else 'FAIL'}")
    
    # 使用模糊断言
    assert fuzzy.contains_any(mock_agent_output, ["编程", "语言", "Python"])
    assert fuzzy.length_between(mock_agent_output, 10, 500)
    assert fuzzy.sentiment_score(mock_agent_output) >= 0.3
    
    print("\n所有模糊断言通过！")


if __name__ == "__main__":
    test_llm_output_fuzzy()
```

---

## 34.4 Agent 测试最佳实践

### 34.4.1 测试金字塔

Agent 的测试应该遵循类似传统软件的测试金字塔，但有一些调整：

**底层（大量）：确定性组件的单元测试。** 工具函数、解析器、格式化器——这些应该有 100% 的测试覆盖率。

**中层（适量）：集成测试（带 Mock LLM）。** 测试 Agent 的完整流程，但用 Mock LLM 确保结果确定。这类测试应该覆盖所有主要的使用路径。

**顶层（少量）：端到端测试（用真实 LLM）。** 用真实的 LLM API 运行完整测试。这类测试不稳定，但能发现 Mock 测试遗漏的问题。

### 34.4.2 测试组织

```python
# tests/conftest.py - 共享的测试夹具

import pytest

@pytest.fixture
def mock_llm():
    """共享的 Mock LLM"""
    return MockLLMClient()

@pytest.fixture
def agent_with_tools(mock_llm):
    """带工具的 Agent"""
    agent = TestableAgent(llm_client=mock_llm)
    agent.register_tool("search", lambda query="": f"搜索: {query}", "搜索")
    agent.register_tool("calculate", lambda expression="": str(eval(expression)), "计算")
    return agent

@pytest.fixture
def fuzzy():
    """模糊断言工具"""
    return FuzzyAssertions()
```

### 34.4.3 持续集成中的测试

在 CI/CD 管道中，建议分两阶段运行测试：

第一阶段（每次提交都运行）：只运行确定性组件的单元测试和带 Mock 的集成测试。这些测试快速且稳定。

第二阶段（每天或每次发布前运行）：运行端到端测试。这些测试较慢且不稳定，但能发现真实场景中的问题。

---

## 34.5 常见坑与最佳实践

**坑 1：过度 Mock 导致测试失去意义。** 如果你 mock 了太多东西，测试就变成了"验证 mock 是否正确配置"而不是"验证代码是否正确工作"。只 mock 必要的外部依赖（LLM API、网络请求），不要 mock 内部逻辑。

**坑 2：测试中硬编码 LLM 输出。** 如果你的测试假设 LLM 会输出特定的字符串，一旦模型更新，测试就会失败。使用模糊断言而不是精确匹配。

**坑 3：忽略边界情况。** 空输入、超长输入、特殊字符、并发调用——这些边界情况在真实场景中一定会遇到，但很容易在测试中被忽略。

**坑 4：测试运行太慢。** 如果你的测试套件需要 10 分钟才能运行完，开发者就不会频繁运行它。把快速的单元测试和慢速的集成测试分开，确保开发者能在 30 秒内得到反馈。

**最佳实践：为每个 bug 写回归测试。** 当你发现并修复了一个 Agent 的 bug 时，把它变成一个测试用例。这样同样的 bug 就不会再次出现。随着 bug 修复的积累，你的测试套件也会越来越完善。

---

## 34.6 练习题

**练习 1：为一个计算器工具写完整的单元测试。** 这个工具支持加减乘除、幂运算和取模。写至少 10 个测试用例，覆盖正常计算、除零错误、无效表达式等场景。

**练习 2：设计一个 Mock LLM 的条件响应。** 实现一个 MockLLMClient，它能根据用户输入中的关键词返回不同的响应。比如输入包含"搜索"就返回工具调用，否则返回直接回答。

**练习 3：测试 Agent 的错误恢复能力。** 设计测试用例验证当工具调用失败时，Agent 能否优雅地恢复并给出有意义的回答。

**练习 4：实现参数化测试。** 为 Agent 的输出解析器创建参数化测试，用 20 组不同的输入输出对来验证解析器的健壮性。

**练习 5：编写性能测试。** 为 Agent 的工具执行编写性能测试，确保每个工具在指定的时间限制内返回结果。

**练习 6：设计 Agent 的契约测试。** 为 Agent 定义一个"输入-输出契约"（比如输入必须是非空字符串，输出必须在 1000 字以内），然后写测试来验证这个契约。

**练习 7：编写性能测试。** 为 Agent 的工具执行编写性能测试，确保每个工具在指定的时间限制内返回结果。使用 pytest-benchmark 来测量和比较不同实现的性能。

---

## 34.7 本章小结

本章系统性地探讨了 Agent 的单元测试方法。核心思路是：将 Agent 分为确定性层和概率性层，确定性层用传统单元测试，概率性层用 Mock + 模糊断言。

通过依赖注入让 Agent 的 LLM 客户端可替换，我们实现了在不调用真实 LLM 的情况下测试 Agent 的完整推理流程。通过 MockLLMClient 的多种响应模式，我们能够模拟各种 Agent 行为场景。

模糊断言策略解决了 LLM 输出不确定性带来的测试挑战。通过关键词匹配、长度检查、模式匹配等技术，我们可以在不要求精确匹配的前提下验证 LLM 输出的质量。

记住测试金字塔：大量的确定性单元测试作为基础，适量的 Mock 集成测试覆盖主要路径，少量的端到端测试验证真实行为。

---

> **下一章预告：** 第 35 章将进入 Agent 安全领域，学习 Agent 面临的主要安全风险以及基础的防护措施。
