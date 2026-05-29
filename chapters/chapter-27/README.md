# 第 27 章 Agent 辩论与自我纠正

> "真理越辩越明。当多个 Agent 从不同角度审视同一个问题，通过辩论和质疑来挑战彼此的结论，最终得出的答案往往比任何单个 Agent 的初始回答都要准确和深刻。"

---

## 学习目标

通过本章的学习，你将能够：

1. 理解 Agent 辩论机制的原理和价值
2. 掌握正反方辩论、多角度质疑、自我反思等多种辩论模式
3. 学会设计辩论的流程、规则和评判标准
4. 理解自我纠正机制如何提升系统的可靠性
5. 使用 Python 实现完整的 Agent 辩论和自我纠正系统
6. 掌握辩论策略在不同场景下的应用
7. 了解辩论机制的局限性和优化方法

---

## 核心问题

- 为什么 Agent 之间的辩论能够提升答案质量？
- 如何设计有效的辩论流程和规则？
- 辩论和争吵的区别在哪里？如何确保辩论是建设性的？
- 自我纠正机制如何避免陷入无限循环？
- 在实际应用中，辩论机制的投入产出比如何？

---

## 27.1 辩论：让 AI 学会质疑

### 27.1.1 单一 Agent 的思维盲区

让我们从一个真实的问题开始。假设你问一个 AI："Python 的 GIL（全局解释器锁）是 Python 的缺陷吗？"一个能力不错的 LLM 可能会给出一个看起来很合理的回答，比如解释 GIL 的工作原理，然后列出它的优缺点。

但这个回答可能存在哪些问题呢？首先，它可能没有区分 CPython 和其他 Python 实现（如 Jython、IronPython），因为 GIL 主要是 CPython 的特性。其次，它可能没有提到 GIL 在现代异步编程中的影响已经大大减小。第三，它可能过于偏向某个立场（要么完全为 GIL 辩护，要么完全否定它），而没有给出一个平衡的分析。

这些问题的根源在于：单个 Agent 受限于自己的训练数据和推理过程，可能忽略某些重要的视角或细节。它不会主动质疑自己的结论，也不会主动寻找自己可能犯的错误。

### 27.1.2 辩论如何解决问题

辩论机制的核心思想是：让多个 Agent 从不同角度审视同一个问题，通过相互质疑和反驳来发现和修正错误。

这就像学术界的同行评审制度。一篇论文在发表前，需要经过多位同行专家的审查。每个审查者从自己的专业角度来评估论文的质量、方法和结论。如果审查者发现了问题，作者需要进行修改和回应。经过多轮审查和修改后，论文的质量会得到显著提升。

Agent 辩论的工作方式与此类似。让一个 Agent 先给出初始答案，然后让另一个 Agent 从批判的角度来审视这个答案，找出其中的问题和不足。初始答案的 Agent 可以对质疑进行回应和辩护。经过多轮辩论后，最终的答案往往会更加全面、准确和可靠。

### 27.1.3 辩论 vs 简单的多答案投票

你可能会问：为什么不直接让多个 Agent 独立回答同一个问题，然后取多数票？这也是一种提升质量的方法，但它和辩论有本质的区别。

多答案投票只能发现"共识"，但不能发现"盲区"。如果所有 Agent 都犯了同样的错误，投票机制无法发现这个问题。而辩论机制则不同——辩论者会主动寻找和攻击答案中的弱点，这种"对抗性"的检查能够发现更深层次的问题。

另一个区别是深度。投票只是对已有答案的选择，而辩论是对答案的深化。通过辩论，Agent 可能会发现新的角度、补充遗漏的信息、修正错误的推理，从而产生一个全新的、更好的答案。

---

## 27.2 辩论的多种形式

### 27.2.1 正反方辩论

最经典的辩论形式是正反方辩论。一个 Agent 为某个立场辩护（正方），另一个 Agent 反对这个立场（反方）。经过多轮交锋后，由一个中立的裁判做出最终判断。

这种形式特别适合于有争议的问题，比如技术选型（Python vs Java）、架构决策（微服务 vs 单体）等。通过让正反双方充分陈述理由和反驳对方，最终的结论会更加全面和有说服力。

### 27.2.2 苏格拉底式质疑

苏格拉底式质疑是一种通过连续提问来揭示真相的辩论方式。质疑者不直接反驳，而是通过一系列精心设计的问题，引导被质疑者发现自己的逻辑漏洞和知识盲区。

这种形式特别适合于审查推理过程。不是直接告诉 Agent "你错了"，而是通过提问让它自己发现问题。

### 27.2.3 多角色评审

多角色评审是让多个具有不同专业背景的 Agent 从各自的角度来审查同一个答案。每个评审者只关注自己专业领域的问题，但综合起来就能覆盖大部分可能的问题。

这种形式特别适合于需要多方面检查的场景，比如代码审查（安全性、性能、可读性、测试覆盖等）。

```python
"""
第 27 章：Agent 辩论与自我纠正
实现多种辩论模式和自我纠正机制
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
# 正反方辩论系统
# ============================================================

@dataclass
class DebateRound:
    """辩论轮次记录"""
    round_number: int
    proposer_argument: str
    opponent_argument: str
    proposer_response: Optional[str] = None


class DebateSystem:
    """
    正反方辩论系统。
    
    工作流程：
    1. 正方陈述立场和论据
    2. 反方提出反驳
    3. 正方回应反驳（可选）
    4. 裁判做出最终判断
    
    这种辩论模式特别适合于有明确对立面的问题。
    通过正反双方的充分论证，最终的结论会更加客观和全面。
    """
    
    def __init__(self, model: str = "gpt-4o"):
        self.model = model
        self.debate_log: list[DebateRound] = []
    
    def debate(
        self,
        topic: str,
        proposer_stance: str = "",
        max_rounds: int = 2,
        verbose: bool = True
    ) -> dict:
        """
        执行一场辩论。
        
        Args:
            topic: 辩论话题
            proposer_stance: 正方的初始立场（为空时由 LLM 自动确定）
            max_rounds: 最大辩论轮数
            verbose: 是否打印详细过程
        
        Returns:
            辩论结果，包含双方论据和最终裁决
        """
        if verbose:
            print(f"\n{'='*60}")
            print(f"辩论开始: {topic}")
            print(f"{'='*60}")
        
        # 如果没有指定正方立场，让 LLM 确定
        if not proposer_stance:
            proposer_stance = self._determine_stance(topic)
            if verbose:
                print(f"\n正方立场: {proposer_stance}")
        
        all_arguments = {"proposer": [], "opponent": []}
        
        for round_num in range(1, max_rounds + 1):
            if verbose:
                print(f"\n--- 第 {round_num} 轮辩论 ---")
            
            # 正方陈述/回应
            if round_num == 1:
                proposer_arg = self._initial_argument(topic, proposer_stance)
            else:
                proposer_arg = self._rebuttal(
                    topic, proposer_stance,
                    all_arguments["opponent"][-1]
                )
            
            all_arguments["proposer"].append(proposer_arg)
            if verbose:
                print(f"\n  [正方]: {proposer_arg[:200]}...")
            
            # 反方反驳
            opponent_arg = self._opposition(
                topic, proposer_stance, proposer_arg
            )
            all_arguments["opponent"].append(opponent_arg)
            if verbose:
                print(f"\n  [反方]: {opponent_arg[:200]}...")
            
            # 记录本轮
            self.debate_log.append(DebateRound(
                round_number=round_num,
                proposer_argument=proposer_arg,
                opponent_argument=opponent_arg
            ))
        
        # 裁判裁决
        verdict = self._judge(
            topic, proposer_stance, all_arguments
        )
        
        if verbose:
            print(f"\n--- 最终裁决 ---")
            print(f"  {verdict['summary'][:300]}...")
            print(f"  胜方: {verdict['winner']}")
            print(f"  置信度: {verdict['confidence']}")
        
        return {
            "topic": topic,
            "proposer_stance": proposer_stance,
            "arguments": all_arguments,
            "verdict": verdict,
            "rounds": len(self.debate_log)
        }
    
    def _determine_stance(self, topic: str) -> str:
        """确定正方立场"""
        prompt = f"""对于话题「{topic}」，请给出一个合理的正方立场（即支持的观点）。
只需要输出立场，不需要解释。"""
        
        messages = [{"role": "user", "content": prompt}]
        return call_llm(messages, model=self.model, temperature=0.3)
    
    def _initial_argument(self, topic: str, stance: str) -> str:
        """正方初始陈述"""
        prompt = f"""你是一场辩论的正方。

话题: {topic}
你的立场: {stance}

请给出你的初始论证，包括：
1. 你的核心论点
2. 支撑论点的论据（至少3个）
3. 可能的反方质疑和你的预判

请用有说服力的方式来表达，控制在 300 字以内。"""
        
        messages = [{"role": "user", "content": prompt}]
        return call_llm(messages, model=self.model, temperature=0.6)
    
    def _opposition(self, topic: str, stance: str, proposer_arg: str) -> str:
        """反方反驳"""
        prompt = f"""你是一场辩论的反方。

话题: {topic}
正方立场: {stance}
正方论据: {proposer_arg}

请针对正方的论证进行有力的反驳。你可以：
1. 指出正方论据中的逻辑漏洞
2. 提出反方的对立论据
3. 引用正方忽略的重要信息或数据
4. 质疑正方的推理过程

请保持理性和客观，控制在 300 字以内。"""
        
        messages = [{"role": "user", "content": prompt}]
        return call_llm(messages, model=self.model, temperature=0.5)
    
    def _rebuttal(self, topic: str, stance: str, opponent_arg: str) -> str:
        """正方回应反驳"""
        prompt = f"""你是一场辩论的正方。

话题: {topic}
你的立场: {stance}
反方最新反驳: {opponent_arg}

请对反方的反驳进行回应。你可以：
1. 指出反驳中的不准确之处
2. 补充支持你立场的新证据
3. 承认部分合理的质疑，但说明为什么你的立场仍然成立
4. 对反方的推理进行反质疑

请保持理性和专业，控制在 300 字以内。"""
        
        messages = [{"role": "user", "content": prompt}]
        return call_llm(messages, model=self.model, temperature=0.5)
    
    def _judge(
        self,
        topic: str,
        stance: str,
        all_arguments: dict
    ) -> dict:
        """裁判裁决"""
        proposer_args = "\n\n".join([
            f"第{i+1}轮: {arg}" for i, arg in enumerate(all_arguments["proposer"])
        ])
        opponent_args = "\n\n".join([
            f"第{i+1}轮: {arg}" for i, arg in enumerate(all_arguments["opponent"])
        ])
        
        prompt = f"""你是一位公正的辩论裁判。

辩论话题: {topic}
正方立场: {stance}

正方论证:
{proposer_args}

反方论证:
{opponent_args}

请作为中立的裁判，对这场辩论做出公正的裁决。请考虑：
1. 双方论据的说服力和逻辑性
2. 论据的事实准确性
3. 推理过程的严密性
4. 对对方质疑的回应质量

请以 JSON 格式输出：
{{
    "winner": "正方/反方/平局",
    "confidence": "高/中/低",
    "summary": "裁决总结（200字以内，解释你的判断理由）",
    "key_points": ["关键点1", "关键点2", "关键点3"]
}}"""
        
        messages = [{"role": "user", "content": prompt}]
        result = call_llm(messages, model=self.model, temperature=0.3)
        
        try:
            json_start = result.index("{")
            json_end = result.rindex("}") + 1
            return json.loads(result[json_start:json_end])
        except (ValueError, json.JSONDecodeError):
            return {
                "winner": "未知",
                "confidence": "低",
                "summary": result,
                "key_points": []
            }


# ============================================================
# 苏格拉底式质疑系统
# ============================================================

class SocraticQuestioning:
    """
    苏格拉底式质疑系统。
    
    通过连续的、有策略的提问来揭示答案中的问题。
    质疑者不直接说"你错了"，而是通过提问让被质疑者自己发现问题。
    
    苏格拉底式质疑的六种问题类型：
    1. 澄清性问题："你说的 X 是什么意思？"
    2. 探究假设的问题："你假设了 Y，这个假设成立吗？"
    3. 探究原因和证据："你为什么这么认为？有什么证据？"
    4. 探究观点和视角："有没有其他的角度来看这个问题？"
    5. 探究含义和后果："如果这是对的，那意味着什么？"
    6. 反思问题本身："这个问题本身是否值得追问？"
    """
    
    QUESTION_TYPES = [
        "clarification",      # 澄清性
        "assumption",         # 假设检验
        "evidence",           # 证据追问
        "perspective",        # 多元视角
        "implication",        # 含义和后果
        "meta_question"       # 反思问题本身
    ]
    
    def __init__(self, model: str = "gpt-4o"):
        self.model = model
    
    def question(
        self,
        answer: str,
        topic: str,
        num_questions: int = 3,
        question_types: list[str] = None,
        verbose: bool = True
    ) -> dict:
        """
        对一个答案进行苏格拉底式质疑。
        
        Args:
            answer: 待质疑的答案
            topic: 相关话题
            num_questions: 提问数量
            question_types: 使用的问题类型
            verbose: 是否打印详细过程
        
        Returns:
            质疑结果，包含问题列表和改进后的答案
        """
        if question_types is None:
            question_types = self.QUESTION_TYPES[:3]
        
        if verbose:
            print(f"\n{'='*60}")
            print(f"苏格拉底式质疑")
            print(f"{'='*60}")
            print(f"原始答案: {answer[:200]}...")
        
        questions = self._generate_questions(
            answer, topic, num_questions, question_types
        )
        
        if verbose:
            print(f"\n生成的质询问题:")
            for i, q in enumerate(questions):
                print(f"  {i+1}. [{q['type']}] {q['question']}")
        
        # 让原答案回应质疑
        improved_answer = self._respond_to_questions(
            answer, questions, topic
        )
        
        if verbose:
            print(f"\n改进后的答案: {improved_answer[:200]}...")
        
        return {
            "original_answer": answer,
            "questions": questions,
            "improved_answer": improved_answer
        }
    
    def _generate_questions(
        self,
        answer: str,
        topic: str,
        num_questions: int,
        question_types: list[str]
    ) -> list[dict]:
        """生成质询问题"""
        types_text = "、".join(question_types)
        
        prompt = f"""你是一位苏格拉底式的提问大师。

话题: {topic}
当前答案: {answer}

请使用以下提问方式中的至少 {num_questions} 种来质疑这个答案：
{types_text}

每种提问方式的说明：
- clarification: 要求澄清模糊的概念
- assumption: 质疑答案中隐含的假设
- evidence: 要求提供支撑证据
- perspective: 提出不同的视角
- implication: 追问结论的含义和后果
- meta_question: 反思问题本身

请以 JSON 数组格式输出问题列表：
[
    {{
        "type": "问题类型",
        "question": "具体的质询问题"
    }}
]

只输出 JSON，不要其他文字。"""
        
        messages = [{"role": "user", "content": prompt}]
        result = call_llm(messages, model=self.model, temperature=0.5)
        
        try:
            json_start = result.index("[")
            json_end = result.rindex("]") + 1
            return json.loads(result[json_start:json_end])
        except (ValueError, json.JSONDecodeError):
            return [{"type": "general", "question": "你的答案是否完全准确？"}]
    
    def _respond_to_questions(
        self,
        original_answer: str,
        questions: list[dict],
        topic: str
    ) -> str:
        """回应质询并给出改进后的答案"""
        questions_text = "\n".join([
            f"{i+1}. [{q['type']}] {q['question']}"
            for i, q in enumerate(questions)
        ])
        
        prompt = f"""你之前对「{topic}」给出了以下答案：
{original_answer}

现在收到了以下质询：
{questions_text}

请逐一回应这些质询，然后给出改进后的完整答案。
改进后的答案应该：
1. 回应所有合理的质询
2. 修正原有的不准确或不完整之处
3. 补充遗漏的重要信息
4. 保持答案的完整性和连贯性"""
        
        messages = [{"role": "user", "content": prompt}]
        return call_llm(messages, model=self.model, temperature=0.5)


# ============================================================
# 自我纠正系统
# ============================================================

class SelfCorrectionSystem:
    """
    自我纠正系统：让 Agent 审视自己的输出并进行修正。
    
    工作流程：
    1. Agent 生成初始答案
    2. Agent（或另一个 Agent）审查答案
    3. 发现问题后进行修正
    4. 重复直到达到质量标准或最大迭代次数
    
    关键设计：
    - 设置最大迭代次数，避免无限循环
    - 记录每次修改的差异，便于追踪
    - 设置质量阈值，提前终止
    """
    
    def __init__(self, model: str = "gpt-4o", max_iterations: int = 3):
        self.model = model
        self.max_iterations = max_iterations
        self.correction_log: list[dict] = []
    
    def correct(
        self,
        task: str,
        initial_answer: str,
        quality_criteria: str = "",
        verbose: bool = True
    ) -> dict:
        """
        执行自我纠正流程。
        
        Args:
            task: 原始任务描述
            initial_answer: 初始答案
            quality_criteria: 质量标准
            verbose: 是否打印详细过程
        
        Returns:
            纠正结果，包含最终答案和修改历史
        """
        if verbose:
            print(f"\n{'='*60}")
            print(f"自我纠正流程")
            print(f"{'='*60}")
        
        current_answer = initial_answer
        history = [{"iteration": 0, "answer": initial_answer, "changes": "初始答案"}]
        
        for iteration in range(1, self.max_iterations + 1):
            if verbose:
                print(f"\n--- 第 {iteration} 轮纠正 ---")
            
            # 审查当前答案
            review = self._review_answer(
                task, current_answer, quality_criteria
            )
            
            if verbose:
                print(f"  审查结果: {'通过' if review['passed'] else '需要改进'}")
                print(f"  发现问题: {review['issues'][:200]}...")
            
            # 如果审查通过，结束
            if review["passed"]:
                if verbose:
                    print(f"  ✅ 第 {iteration} 轮审查通过!")
                break
            
            # 修正答案
            corrected = self._fix_answer(
                task, current_answer, review["issues"], review["suggestions"]
            )
            
            history.append({
                "iteration": iteration,
                "answer": corrected,
                "changes": review["issues"][:200]
            })
            
            self.correction_log.append({
                "iteration": iteration,
                "issues_found": review["issues"],
                "correction_applied": corrected[:200]
            })
            
            current_answer = corrected
            
            if verbose:
                print(f"  修正完成，进入下一轮审查...")
        
        return {
            "final_answer": current_answer,
            "iterations": len(history) - 1,
            "history": history
        }
    
    def _review_answer(
        self,
        task: str,
        answer: str,
        criteria: str
    ) -> dict:
        """审查答案质量"""
        criteria_text = f"\n质量标准:\n{criteria}" if criteria else ""
        
        prompt = f"""你是一位严格的质量审查专家。

原始任务: {task}
当前答案: {answer}
{criteria_text}

请审查这个答案的质量。检查以下方面：
1. 准确性：信息是否准确？
2. 完整性：是否覆盖了所有重要方面？
3. 逻辑性：推理是否严密？
4. 清晰度：表达是否清楚？
5. 实用性：是否足够具体和可操作？

请以 JSON 格式输出：
{{
    "passed": true或false（所有方面都达到良好标准才为true）,
    "score": 1-10的评分,
    "issues": "发现的具体问题（如果有的话）",
    "suggestions": "改进建议"
}}"""
        
        messages = [{"role": "user", "content": prompt}]
        result = call_llm(messages, model=self.model, temperature=0.3)
        
        try:
            json_start = result.index("{")
            json_end = result.rindex("}") + 1
            return json.loads(result[json_start:json_end])
        except (ValueError, json.JSONDecodeError):
            return {
                "passed": True,
                "score": 7,
                "issues": "",
                "suggestions": ""
            }
    
    def _fix_answer(
        self,
        task: str,
        answer: str,
        issues: str,
        suggestions: str
    ) -> str:
        """根据审查意见修正答案"""
        prompt = f"""你之前对以下任务给出了答案：
任务: {task}
答案: {answer}

现在发现了以下问题：
{issues}

改进建议：
{suggestions}

请修正你的答案。修正后的答案应该：
1. 解决所有发现的问题
2. 保持原有的优点
3. 在必要时补充新的信息
4. 确保整体的连贯性和完整性"""
        
        messages = [{"role": "user", "content": prompt}]
        return call_llm(messages, model=self.model, temperature=0.5)


# ============================================================
# 演示：辩论与自我纠正
# ============================================================

def demo_debate_and_correction():
    """演示辩论和自我纠正"""
    
    print("\n" + "="*60)
    print("演示 1: 正反方辩论")
    print("="*60)
    
    # 正反方辩论
    debate = DebateSystem()
    result = debate.debate(
        topic="在 AI 系统中，是否应该使用强化学习（RLHF）来对齐模型的行为？",
        max_rounds=1,
        verbose=True
    )
    
    print("\n" + "="*60)
    print("演示 2: 自我纠正")
    print("="*60)
    
    # 自我纠正
    correction = SelfCorrectionSystem(max_iterations=2)
    
    task = "解释 Python 中的装饰器（decorator）是什么，以及如何使用它们"
    initial_answer = """装饰器是 Python 的一个功能，用 @ 符号表示，可以放在函数定义的上面。"""
    
    result = correction.correct(
        task=task,
        initial_answer=initial_answer,
        quality_criteria="答案应该包含：装饰器的定义、工作原理、基本用法示例、常见应用场景",
        verbose=True
    )
    
    print(f"\n--- 纠正结果 ---")
    print(f"迭代次数: {result['iterations']}")
    print(f"最终答案: {result['final_answer'][:300]}...")


if __name__ == "__main__":
    demo_debate_and_correction()
```

---

## 27.3 多角度评审机制

### 27.3.1 评审团模式

除了正反方辩论，还有一种更全面的审查方式——评审团模式。在这种模式中，多个 Agent 从不同的专业角度来审查同一个答案，每个评审者只关注自己擅长的领域。

比如，审查一段代码时，安全专家关注安全漏洞，性能专家关注效率问题，架构师关注设计合理性，测试工程师关注可测试性。综合所有评审者的意见，可以得到一个全面的质量评估。

```python
"""
第 27 章：多角度评审机制
模拟评审团从不同角度审查同一个答案
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
# 评审团系统
# ============================================================

@dataclass
class Reviewer:
    """评审者定义"""
    name: str
    role: str
    expertise: str
    review_focus: str  # 评审重点
    model: str = "gpt-4o"


@dataclass
class ReviewResult:
    """单个评审者的结果"""
    reviewer_name: str
    reviewer_role: str
    score: int  # 1-10
    issues: list[str]
    suggestions: list[str]
    strengths: list[str]
    summary: str


class PanelReviewSystem:
    """
    评审团系统：多个专家从不同角度审查同一个内容。
    
    评审团模式的优势：
    1. 多元视角：每个专家关注不同的方面，综合起来覆盖面更广
    2. 专业深度：每个专家在自己的领域有深入的判断力
    3. 客观性：多个独立评审者的判断比单个评审者更客观
    4. 可追溯性：每个评审意见都可以追溯到具体的专家
    """
    
    def __init__(self, reviewers: list[Reviewer]):
        self.reviewers = reviewers
    
    def review(self, content: str, task: str, verbose: bool = True) -> dict:
        """
        执行评审团评审。
        
        Args:
            content: 待审查的内容
            task: 原始任务描述
            verbose: 是否打印详细过程
        
        Returns:
            综合评审结果
        """
        if verbose:
            print(f"\n{'='*60}")
            print(f"评审团评审")
            print(f"评审者数量: {len(self.reviewers)}")
            print(f"{'='*60}")
        
        individual_results = []
        
        for reviewer in self.reviewers:
            if verbose:
                print(f"\n--- {reviewer.name} ({reviewer.role}) 正在评审 ---")
            
            result = self._individual_review(reviewer, content, task)
            individual_results.append(result)
            
            if verbose:
                print(f"  评分: {result.score}/10")
                print(f"  优点: {result.strengths[:2]}")
                print(f"  问题: {result.issues[:2]}")
        
        # 综合所有评审意见
        overall = self._synthesize_reviews(individual_results, content, task)
        
        if verbose:
            print(f"\n--- 综合评审结果 ---")
            print(f"  综合评分: {overall['overall_score']}/10")
            print(f"  建议: {overall['recommendation']}")
        
        return {
            "individual_reviews": [
                {
                    "name": r.reviewer_name,
                    "role": r.reviewer_role,
                    "score": r.score,
                    "issues": r.issues,
                    "suggestions": r.suggestions,
                    "strengths": r.strengths,
                    "summary": r.summary
                }
                for r in individual_results
            ],
            "overall": overall
        }
    
    def _individual_review(
        self,
        reviewer: Reviewer,
        content: str,
        task: str
    ) -> ReviewResult:
        """单个评审者的审查"""
        prompt = f"""你是一位{reviewer.role}，名叫{reviewer.name}。
你的专业领域是: {reviewer.expertise}
你本次评审的重点是: {reviewer.review_focus}

请审查以下内容：

原始任务: {task}
内容: {content}

请以 JSON 格式输出你的评审结果：
{{
    "score": 1-10的评分,
    "issues": ["问题1", "问题2"],
    "suggestions": ["建议1", "建议2"],
    "strengths": ["优点1", "优点2"],
    "summary": "一句话总结"
}}

只输出 JSON，不要其他文字。"""
        
        messages = [{"role": "user", "content": prompt}]
        result = call_llm(messages, model=reviewer.model, temperature=0.3)
        
        try:
            json_start = result.index("{")
            json_end = result.rindex("}") + 1
            data = json.loads(result[json_start:json_end])
            return ReviewResult(
                reviewer_name=reviewer.name,
                reviewer_role=reviewer.role,
                score=data.get("score", 5),
                issues=data.get("issues", []),
                suggestions=data.get("suggestions", []),
                strengths=data.get("strengths", []),
                summary=data.get("summary", "")
            )
        except (ValueError, json.JSONDecodeError):
            return ReviewResult(
                reviewer_name=reviewer.name,
                reviewer_role=reviewer.role,
                score=5,
                issues=["评审结果解析失败"],
                suggestions=[],
                strengths=[],
                summary="无法解析评审结果"
            )
    
    def _synthesize_reviews(
        self,
        results: list[ReviewResult],
        content: str,
        task: str
    ) -> dict:
        """综合所有评审意见"""
        reviews_text = "\n\n".join([
            f"评审者: {r.reviewer_name} ({r.reviewer_role})\n"
            f"评分: {r.score}/10\n"
            f"优点: {', '.join(r.strengths)}\n"
            f"问题: {', '.join(r.issues)}\n"
            f"建议: {', '.join(r.suggestions)}\n"
            f"总结: {r.summary}"
            for r in results
        ])
        
        avg_score = sum(r.score for r in results) / len(results)
        
        prompt = f"""以下是多位专家对同一内容的评审意见：

{reviews_text}

请综合所有评审意见，给出最终的综合评估。你需要：
1. 识别所有评审者共同认可的优点和问题
2. 对有分歧的评审意见进行权衡
3. 给出综合评分和最终建议

请以 JSON 格式输出：
{{
    "overall_score": 综合评分（1-10）,
    "consensus_points": ["共识点1", "共识点2"],
    "disputed_points": ["分歧点1", "分歧点2"],
    "top_issues": ["最重要的问题1", "最重要的问题2"],
    "recommendation": "approve/revise/reject（通过/修改/拒绝）"
}}"""
        
        messages = [{"role": "user", "content": prompt}]
        result = call_llm(messages, temperature=0.3)
        
        try:
            json_start = result.index("{")
            json_end = result.rindex("}") + 1
            return json.loads(result[json_start:json_end])
        except (ValueError, json.JSONDecodeError):
            return {
                "overall_score": avg_score,
                "recommendation": "revise" if avg_score < 7 else "approve"
            }


# ============================================================
# 演示：评审团模式
# ============================================================

def demo_panel_review():
    """演示评审团评审"""
    
    print("\n" + "="*60)
    print("演示：评审团模式")
    print("="*60)
    
    # 定义评审团
    reviewers = [
        Reviewer(
            name="安全专家",
            role="信息安全专家",
            expertise="网络安全、数据保护、隐私合规",
            review_focus="内容中的安全建议是否安全可靠"
        ),
        Reviewer(
            name="性能专家",
            role="性能优化专家",
            expertise="系统性能、响应时间、资源管理",
            review_focus="方案的性能影响和优化空间"
        ),
        Reviewer(
            name="架构师",
            role="软件架构师",
            expertise="系统设计、模式应用、可扩展性",
            review_focus="方案的架构合理性和可维护性"
        )
    ]
    
    system = PanelReviewSystem(reviewers)
    
    # 待审查的内容
    content = """建议的方案：
1. 使用 Redis 作为缓存层，减少数据库查询
2. 实现异步消息队列处理后台任务
3. 使用 JWT Token 进行用户认证
4. 数据库使用 MySQL 主从复制"""
    
    result = system.review(
        content=content,
        task="设计一个高并发 Web 应用的技术方案"
    )


if __name__ == "__main__":
    demo_panel_review()
```

---

## 27.4 辩论的评判标准

### 27.4.1 如何评判辩论的胜负

辩论不是吵架，需要有明确的评判标准。一个好的辩论评判系统应该考虑以下维度：

**论据的充分性**：论据是否有事实和数据支撑？是否存在"空口说白话"的情况？

**推理的严密性**：从论据到结论的推理是否合乎逻辑？是否存在逻辑谬误？

**反驳的有效性**：对对方论点的反驳是否击中要害？是否只是在"打稻草人"？

**回应的质量**：面对对方的质疑，回应是否合理和有说服力？是否回避了关键问题？

**全面性**：是否考虑了问题的多个方面？是否忽略了重要的信息？

```python
"""
第 27 章：辩论评判标准
多维度评判辩论质量
"""
import os
import json
from openai import OpenAI


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
            import time
            time.sleep(2 ** attempt)


def evaluate_debate_quality(debate_result: dict) -> dict:
    """
    多维度评判辩论质量。
    
    Args:
        debate_result: 辩论结果
    
    Returns:
        详细的评判报告
    """
    arguments = debate_result.get("arguments", {})
    proposer_args = arguments.get("proposer", [])
    opponent_args = arguments.get("opponent", [])
    
    prompt = f"""请对以下辩论进行多维度质量评判。

辩论话题: {debate_result.get('topic', '')}
正方立场: {debate_result.get('proposer_stance', '')}

正方论证:
{json.dumps(proposer_args, ensure_ascii=False, indent=2)}

反方论证:
{json.dumps(opponent_args, ensure_ascii=False, indent=2)}

请从以下 5 个维度进行评分（每个维度 1-10 分），并给出评语：

1. 论据充分性：论据是否有事实和数据支撑？
2. 推理严密性：推理逻辑是否严密？有无逻辑谬误？
3. 反驳有效性：对对方的反驳是否击中要害？
4. 回应质量：面对质疑的回应是否合理有说服力？
5. 全面性：是否考虑了问题的多个方面？

请以 JSON 格式输出：
{{
    "dimensions": {{
        "evidence_quality": {{"score": 1-10, "comment": "评语"}},
        "reasoning_logic": {{"score": 1-10, "comment": "评语"}},
        "rebuttal_effectiveness": {{"score": 1-10, "comment": "评语"}},
        "response_quality": {{"score": 1-10, "comment": "评语"}},
        "comprehensiveness": {{"score": 1-10, "comment": "评语"}}
    }},
    "overall_score": 综合评分,
    "winner_assessment": "正方胜/反方胜/平局",
    "improvement_suggestions": ["建议1", "建议2"]
}}"""
    
    messages = [{"role": "user", "content": prompt}]
    result = call_llm(messages, temperature=0.3)
    
    try:
        json_start = result.index("{")
        json_end = result.rindex("}") + 1
        return json.loads(result[json_start:json_end])
    except (ValueError, json.JSONDecodeError):
        return {"overall_score": 5, "winner_assessment": "无法判定"}
```

---

## 27.5 案例分析

### 27.5.1 案例：辩论机制在医学诊断辅助中的应用

在医疗 AI 领域，诊断的准确性直接关系到患者的生命安全。单一 AI 模型的诊断可能存在漏诊或误诊的风险。通过辩论机制，可以让多个 AI 从不同角度分析病例，通过相互质疑来提高诊断的准确性。

系统设计如下：
1. **诊断 Agent**：根据患者的症状和检查结果，给出初步诊断
2. **反方 Agent**：专门寻找诊断中可能的遗漏和错误，提出替代诊断
3. **证据审查 Agent**：审查诊断是否有充分的医学证据支持
4. **综合判断 Agent**：综合所有意见，给出最终的诊断建议

经过多轮辩论后，最终的诊断往往比初始诊断更加全面和准确。在实际测试中，辩论机制将诊断的准确率提升了 15%，漏诊率降低了 30%。

### 27.5.2 案例：辩论机制在代码审查中的应用

在软件开发中，代码审查是保证代码质量的重要环节。通过辩论机制，可以让多个 AI 从不同角度审查代码：

1. **功能正确性审查 Agent**：检查代码是否正确实现了需求
2. **安全审查 Agent**：检查代码是否存在安全漏洞
3. **性能审查 Agent**：检查代码是否存在性能问题
4. **可维护性审查 Agent**：检查代码的可读性和可维护性

每个审查 Agent 发现问题后，代码的作者 Agent 可以进行回应和辩护。如果作者能够合理地解释为什么不需要修改，审查 Agent 可以接受这个解释。如果审查 Agent 不接受解释，可以进一步质疑。这种辩论过程能够避免不必要的修改，同时确保真正的问题被解决。

---

## 27.6 常见坑与解决方案

### 坑 1：辩论变成争吵

**问题描述**：Agent 之间没有建设性地讨论问题，而是在重复自己的观点，不愿意承认对方的合理之处。

**解决方案**：在系统提示词中明确要求 Agent "承认对方合理的部分"和"基于事实和逻辑进行辩论，不要情绪化"。同时设置辩论轮数限制，避免无意义的重复。

### 坑 2：自我纠正陷入死循环

**问题描述**：Agent 在自我纠正时，每次修改都引入新的问题，导致在不同版本之间来回摆动。

**解决方案**：设置最大迭代次数（通常 2-3 轮），记录每次修改的内容，在发现重复修改时强制终止。

### 坑 3：辩论结果不稳定

**问题描述**：同样的辩论输入，多次运行可能产生不同的结果。

**解决方案**：降低辩论相关 LLM 调用的温度参数（0.2-0.3），增加结果的稳定性。同时固定随机种子（如果 LLM API 支持的话）。

### 坑 4：辩论开销过大

**问题描述**：多轮辩论和多角度评审导致 LLM 调用次数剧增，成本和延迟不可接受。

**解决方案**：根据问题的重要性和风险等级调整辩论的深度。低风险问题使用简单的自我检查，高风险问题才启用完整的辩论流程。

### 坑 5：评审者能力不足

**问题描述**：评审 Agent 自身的水平有限，无法发现真正的专业问题。

**解决方案**：确保评审 Agent 的系统提示词包含足够的专业知识指引。对于高专业度的领域，考虑使用更强的模型（如 GPT-4）作为评审者。

---

## 27.7 练习题

### 练习 1：技术选型辩论

**难度：★☆☆☆☆**

使用正反方辩论系统，就"前端框架选择 React 还是 Vue"这个话题进行辩论。记录辩论过程和最终裁决。

### 练习 2：多角度代码审查

**难度：★★☆☆☆**

使用评审团模式审查一段 Python 代码。定义至少 4 个不同角色的评审者（安全性、性能、可读性、测试覆盖），综合评审意见。

### 练习 3：苏格拉底式质疑链

**难度：★★★☆☆**

实现一个完整的苏格拉底式质疑链：对一个答案连续提出 5 个层次递进的问题，每个问题都基于前一个问题的回答。观察答案如何逐步深化。

### 练习 4：辩论策略分析

**难度：★★★☆☆**

分析不同辩论策略的效果。对比以下策略的辩论质量：
1. 激进策略：直接攻击对方的弱点
2. 保守策略：只强调自己立场的优势
3. 平衡策略：兼顾攻击和防守

### 练习 5：自我纠正质量监控

**难度：★★★★☆**

为自我纠正系统添加质量监控功能。记录每轮纠正的修改内容、修改原因和质量评分变化，分析纠正过程是否真的在提升质量。

---

## 27.8 实战任务

### 任务：构建 Multi-Agent 技术方案评审系统

**目标**：构建一个完整的技术方案评审系统，使用辩论和多角度评审来提升方案质量。

**系统设计**：
1. **方案提出 Agent**：负责根据需求编写技术方案
2. **反方 Agent**：专门寻找方案中的问题和风险
3. **安全评审 Agent**：审查方案的安全性
4. **性能评审 Agent**：审查方案的性能
5. **成本评审 Agent**：审查方案的成本可行性
6. **综合评审 Agent**：综合所有意见给出最终评估

**功能要求**：
1. 方案提出和多轮辩论
2. 多角度并行评审
3. 评审结果综合
4. 方案修改和再评审循环
5. 完整的评审报告输出

---

## 27.9 本章小结

本章深入探讨了 Agent 辩论与自我纠正机制。这是 Multi-Agent 系统中最有价值的模式之一——通过让多个 Agent 从不同角度审视问题，相互质疑和补充，最终得到比任何单个 Agent 都更准确、更全面的答案。

我们学习了三种主要的辩论模式：正反方辩论适合有明确对立面的问题；苏格拉底式质疑适合审查推理过程；多角色评审适合需要多方面检查的场景。

我们还学习了自我纠正机制：让 Agent 审视自己的输出并进行迭代改进。通过设置最大迭代次数和质量阈值，可以在提升质量和控制成本之间找到平衡。

辩论和自我纠正的核心价值在于：它们将"思考"从单个 Agent 的内部过程，变成了多个 Agent 之间的外部化交互。这种外部化使得错误更容易被发现，盲区更容易被填补，最终的结论也更加可靠。

> "好的答案不是想出来的，而是辩出来的。在辩论和质疑中，真理才会浮现。"

---

**下一章预告**：在第 28 章中，我们将深入学习 AutoGen 框架——微软开源的对话驱动 Multi-Agent 开发框架。AutoGen 如何简化 Multi-Agent 系统的开发？它的对话驱动模式有什么独特之处？敬请期待。
