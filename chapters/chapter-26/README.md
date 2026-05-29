# 第 26 章 角色化 Multi-Agent —— 专业分工

> "术业有专攻。当每个 Agent 都成为各自领域的专家，整个系统的智能就会超越任何单个 Agent 的能力上限。这就是角色化 Multi-Agent 的魔力。"

---

## 学习目标

通过本章的学习，你将能够：

1. 理解角色化 Multi-Agent 系统的核心理念和设计原则
2. 掌握 Agent 角色定义的方法论，包括角色描述、能力边界和行为规范
3. 学会设计不同类型的角色（创造者、批判者、执行者、协调者等）
4. 理解角色之间的协作关系和信息流动
5. 使用 Python 实现完整的角色化 Multi-Agent 系统
6. 掌握角色设计中的常见陷阱和优化方法
7. 了解角色化系统在实际应用中的最佳实践

---

## 核心问题

- 如何定义一个有效的 Agent 角色？角色描述应该包含哪些要素？
- 不同类型的角色如何相互配合？
- 角色的"专业度"和"灵活性"之间如何平衡？
- 如何避免角色定义导致的思维僵化？
- 角色化 Multi-Agent 系统的最佳实践是什么？

---

## 26.1 角色的力量

### 26.1.1 从通用到专业

在前几章中，我们搭建的 Multi-Agent 系统中的 Agent 通常只有简单的角色区分——研究员、分析师、作家。这些区分更多是基于"谁做什么"的任务分工，而不是真正的"角色化"设计。

角色化 Multi-Agent 的核心理念是：每个 Agent 不仅仅是一个执行特定任务的工具，而是一个有独特视角、专业能力和行为风格的"角色"。就像一部好的电影，每个角色都有自己的性格、动机和行为方式，他们的互动才构成了一个生动的故事。

让我们用一个类比来说明。假设你要拍一部电影。如果所有演员都不区分角色，都用同一种方式表演，观众一定会觉得无聊。但如果每个演员都深入理解自己的角色——反派有自己的动机和苦衷，英雄有自己的弱点和成长，配角有自己的故事线——整部电影就会变得引人入胜。

Multi-Agent 系统也是如此。当每个 Agent 都有了独特的角色设定——不仅知道自己要做什么，还知道为什么这样做、应该怎么做、不应该怎么做——系统整体的表现就会有质的飞跃。

### 26.1.2 角色 vs 任务

任务分工和角色化设计的区别在哪里？

任务分工回答的是"做什么"——这个 Agent 负责搜集数据，那个 Agent 负责写报告。每个 Agent 像一个流水线上的工人，只需要完成自己的工序。

角色化设计回答的是"是什么"和"怎么做"——这个 Agent 是一个严谨的数据科学家，它会质疑数据的来源和质量，会用统计方法验证假设；那个 Agent 是一个富有创意的文案策划，它会从用户的角度思考，用故事来包装数据。

区别在于，角色化设计不仅定义了 Agent 的功能，还定义了它的思维方式、价值判断和行为模式。这使得 Agent 的输出不仅在内容上有所不同，在风格、深度和视角上也会呈现出真正的多样性。

### 26.1.3 角色设计的核心要素

一个完整的 Agent 角色通常包含以下要素：

**身份定义**：Agent 是谁？它有什么背景和经历？这影响它的知识结构和思维方式。

**专业能力**：Agent 擅长什么？它能使用什么工具？这决定了它能处理什么类型的任务。

**行为规范**：Agent 应该如何行事？它应该遵循什么原则和标准？这确保了 Agent 的行为可预测且一致。

**思维方式**：Agent 如何分析问题？它倾向于从什么角度思考？这影响了它的分析和判断。

**交互风格**：Agent 如何与其他人/Agent 沟通？它的沟通风格是正式的还是轻松的？这影响了团队协作的氛围。

**限制边界**：Agent 不应该做什么？在什么情况下应该拒绝或转交任务？这防止了 Agent 越界或做出不当行为。

---

## 26.2 经典角色类型

### 26.2.1 创造者（Creator）

创造者角色负责生成新的内容、想法或方案。它的核心能力是创造力和想象力，能够从无到有地产出有价值的东西。

在代码世界中，创造者可以是撰写代码的开发者、设计架构的架构师、撰写文档的技术作家。在分析场景中，创造者可以是提出假设的研究员、构思营销方案的策划人员。

### 26.2.2 批判者（Critic）

批判者角色负责审查和评估其他 Agent 的产出。它的核心能力是批判性思维和细致的观察力，能够发现潜在的问题、错误和改进空间。

批判者的存在是 Multi-Agent 系统质量保证的关键。没有批判者，创造者的产出可能包含错误或遗漏而无人发现。但批判者也需要注意分寸——过度的批评会打击创造者的积极性，而过于宽松的审查则失去了批判的意义。

### 26.2.3 协调者（Coordinator）

协调者角色负责团队的组织和协调。它的核心能力是全局视野和组织能力，能够理解任务的整体结构，合理分配工作，确保团队高效协作。

协调者不需要在任何特定领域都是专家，但它需要理解每个成员的能力和限制，能够在合适的时机做出正确的调度决策。

### 26.2.4 专家（Expert）

专家角色在特定领域拥有深厚的知识和技能。它的核心能力是专业深度——在自己的领域内，它的判断和产出应该是整个团队中最好的。

一个 Multi-Agent 系统通常有多个不同领域的专家，它们各自在自己的领域发挥专长。

### 26.2.5 研究员（Researcher）

研究员角色负责搜集和整理信息。它的核心能力是信息检索和筛选，能够从大量信息中找到最有价值的内容，并以结构化的方式呈现。

### 26.2.6 执行者（Executor）

执行者角色负责将计划转化为实际的行动。它的核心能力是执行力和细节把控，能够准确无误地完成具体的任务。

---

## 26.3 完整实现：角色化 Multi-Agent 写作系统

让我们用一个完整的代码示例来展示角色化 Multi-Agent 的设计和实现。这个系统模拟了一个"AI 编辑部"，其中每个 Agent 都有鲜明的角色设定。

```python
"""
第 26 章：角色化 Multi-Agent 系统
AI 编辑部：每个 Agent 都有独特的角色和专业能力
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
# 角色定义
# ============================================================

@dataclass
class RoleProfile:
    """
    角色画像：定义一个 Agent 的完整角色设定。
    
    这是角色化 Multi-Agent 的核心数据结构。
    一个好的角色画像应该能够让 LLM "入戏"——
    真正以这个角色的视角来思考和行动。
    """
    name: str  # 角色名称
    title: str  # 职位/头衔
    identity: str  # 身份描述（背景、经历）
    expertise: list[str]  # 专业领域
    personality: str  # 性格特征
    thinking_style: str  # 思维方式
    communication_style: str  # 沟通风格
    principles: list[str]  # 行为准则
    boundaries: list[str]  # 限制边界
    tools_allowed: list[str] = field(default_factory=list)  # 允许使用的工具
    model: str = "gpt-4o"  # 使用的 LLM 模型
    
    def to_system_prompt(self) -> str:
        """
        将角色画像转换为系统提示词。
        
        这是角色化系统的关键方法——它将结构化的角色定义
        转化为 LLM 能够理解的自然语言提示。
        """
        principles_text = "\n".join(f"  - {p}" for p in self.principles)
        boundaries_text = "\n".join(f"  - {b}" for b in self.boundaries)
        expertise_text = "、".join(self.expertise)
        
        return f"""你是一位{self.title}，名叫{self.name}。

## 你的身份
{self.identity}

## 你的专业领域
{expertise_text}

## 你的性格特点
{self.personality}

## 你的思维方式
{self.thinking_style}

## 你的沟通风格
{self.communication_style}

## 你的行为准则
{principles_text}

## 你的限制边界
{boundaries_text}

请始终保持你的角色设定。你的回答应该体现你的专业知识、性格特点和思维方式。
不要偏离你的角色，也不要做出超出你角色能力范围的承诺。"""


# ============================================================
# 角色化 Agent
# ============================================================

class RoleAgent:
    """
    角色化 Agent：具有完整角色设定的智能体。
    
    与普通的 Agent 不同，RoleAgent 的行为完全由其角色设定驱动。
    它不仅知道自己要做什么，还知道为什么、怎么做、不应该做什么。
    """
    
    def __init__(self, role: RoleProfile):
        self.role = role
        self.conversation_history: list[dict] = []
        self.work_output: dict[str, str] = {}  # 工作产出记录
    
    @property
    def name(self) -> str:
        return self.role.name
    
    @property
    def system_prompt(self) -> str:
        return self.role.to_system_prompt()
    
    def think(self, task: str, context: str = "") -> str:
        """
        以角色的身份进行思考和回答。
        
        Args:
            task: 任务描述
            context: 额外的上下文信息
        
        Returns:
            角色的响应
        """
        messages = [{"role": "system", "content": self.system_prompt}]
        
        # 加入对话历史
        messages.extend(self.conversation_history[-10:])
        
        # 构建任务消息
        task_msg = task
        if context:
            task_msg = f"""任务: {task}

背景信息:
{context}"""
        
        messages.append({"role": "user", "content": task_msg})
        
        # 根据角色选择不同的温度
        temperature = self._get_temperature()
        
        result = call_llm(
            messages,
            model=self.role.model,
            temperature=temperature,
            max_tokens=2000
        )
        
        # 更新对话历史
        self.conversation_history.append({"role": "user", "content": task_msg})
        self.conversation_history.append({"role": "assistant", "content": result})
        
        return result
    
    def review(self, content: str, reviewer_perspective: str = "") -> str:
        """
        以角色的视角审查内容。
        
        Args:
            content: 待审查的内容
            reviewer_perspective: 额外的审查角度
        
        Returns:
            审查意见
        """
        prompt = f"""请以你的专业视角审查以下内容：

---
{content}
---

{f"请特别关注: {reviewer_perspective}" if reviewer_perspective else ""}

请给出详细的审查意见，包括：
1. 你认为做得好的地方
2. 你发现的问题和不足
3. 你建议的改进方向

请保持你的专业水准和角色风格。"""
        
        return self.think(prompt)
    
    def collaborate(self, other_agent_name: str, message: str) -> str:
        """
        以角色的身份与其他 Agent 协作。
        
        Args:
            other_agent_name: 协作对象的名称
            message: 要传达的信息
        
        Returns:
            协作响应
        """
        prompt = f"""你需要与「{other_agent_name}」进行协作。

信息: {message}

请以你的角色身份回应，体现你的专业能力和协作精神。"""
        
        return self.think(prompt)
    
    def _get_temperature(self) -> float:
        """根据角色类型选择合适的温度"""
        if "批判" in self.role.title or "审查" in self.role.title:
            return 0.3  # 批判者使用低温度，确保稳定性
        elif "创造" in self.role.title or "设计" in self.role.title:
            return 0.8  # 创造者使用高温度，增加多样性
        else:
            return 0.6  # 其他角色使用中等温度
    
    def get_work_history(self) -> dict:
        """获取工作历史"""
        return {
            "name": self.name,
            "role": self.role.title,
            "tasks_completed": len(self.work_output),
            "outputs": self.work_output
        }


# ============================================================
# 角色化团队
# ============================================================

class RoleBasedTeam:
    """
    角色化团队：管理多个角色化 Agent 的协作。
    
    团队负责：
    1. 管理团队成员的角色和状态
    2. 协调成员之间的协作
    3. 整合各成员的产出
    4. 控制整体工作流程
    """
    
    def __init__(self, team_name: str):
        self.team_name = team_name
        self.members: dict[str, RoleAgent] = {}
        self.workflow_log: list[dict] = []
    
    def add_member(self, agent: RoleAgent):
        """添加团队成员"""
        self.members[agent.name] = agent
        print(f"  [{self.team_name}] 添加成员: {agent.name} ({agent.role.title})")
    
    def get_member(self, name: str) -> Optional[RoleAgent]:
        """获取团队成员"""
        return self.members.get(name)
    
    def list_members(self):
        """列出所有成员"""
        print(f"\n  [{self.team_name}] 团队成员:")
        for name, agent in self.members.items():
            expertise = "、".join(agent.role.expertise)
            print(f"    - {name} ({agent.role.title}): {expertise}")
    
    def coordinate_task(
        self,
        task: str,
        workflow: list[dict],
        verbose: bool = True
    ) -> str:
        """
        协调团队完成任务。
        
        Args:
            task: 总任务描述
            workflow: 工作流程定义，每个步骤包含 agent（执行者）和 action（动作）
            verbose: 是否打印详细日志
        
        Returns:
            最终产出
        """
        if verbose:
            print(f"\n{'='*60}")
            print(f"团队 [{self.team_name}] 开始协作")
            print(f"任务: {task}")
            print(f"{'='*60}")
        
        context = f"总任务: {task}"
        results = {}
        
        for i, step in enumerate(workflow):
            agent_name = step["agent"]
            action = step["action"]
            depends_on = step.get("depends_on", [])
            
            agent = self.members.get(agent_name)
            if not agent:
                print(f"  [警告] 找不到成员 '{agent_name}'，跳过此步骤")
                continue
            
            # 构建上下文
            step_context = context
            for dep in depends_on:
                if dep in results:
                    step_context += f"\n\n来自 {dep} 的输入:\n{results[dep]}"
            
            if verbose:
                print(f"\n--- 步骤 {i+1}: {agent_name} - {action} ---")
            
            # 执行
            result = agent.think(f"{action}\n\n任务: {task}", step_context)
            results[agent_name] = result
            agent.work_output[action] = result
            
            # 记录日志
            self.workflow_log.append({
                "step": i + 1,
                "agent": agent_name,
                "action": action,
                "result_length": len(result)
            })
            
            if verbose:
                preview = result[:150]
                print(f"  产出: {preview}...")
        
        # 整合最终结果
        final_result = self._integrate_results(task, results)
        
        if verbose:
            print(f"\n{'='*60}")
            print(f"团队 [{self.team_name}] 协作完成")
            print(f"{'='*60}")
        
        return final_result
    
    def _integrate_results(self, task: str, results: dict) -> str:
        """整合各成员的产出"""
        # 如果只有一个成员的结果，直接返回
        if len(results) == 1:
            return list(results.values())[0]
        
        # 否则，尝试整合
        results_text = "\n\n".join([
            f"## {name} 的产出\n{output}"
            for name, output in results.items()
        ])
        
        prompt = f"""请将以下多个团队成员的产出整合成一个完整、连贯的结果。

任务: {task}

各成员产出:
{results_text}

请整合以上内容，确保最终结果：
1. 完整覆盖任务要求
2. 内容连贯、无矛盾
3. 质量达到专业标准"""
        
        messages = [{"role": "user", "content": prompt}]
        return call_llm(messages, temperature=0.5)
    
    def internal_review(self, content: str, reviewer_name: str) -> str:
        """
        团队内部审查。
        
        指定一个成员对内容进行审查。
        """
        reviewer = self.members.get(reviewer_name)
        if not reviewer:
            return f"找不到审查者 '{reviewer_name}'"
        
        return reviewer.review(content)
    
    def team_discussion(self, topic: str, max_rounds: int = 2) -> dict:
        """
        团队内部讨论。
        
        所有成员围绕一个话题发表各自的观点。
        """
        print(f"\n--- 团队讨论: {topic} ---")
        
        all_opinions = {}
        
        for round_num in range(max_rounds):
            print(f"\n  第 {round_num + 1} 轮:")
            
            for name, agent in self.members.items():
                if round_num == 0:
                    # 第一轮：发表初始观点
                    prompt = f"请就以下话题发表你的专业观点：\n\n{topic}"
                else:
                    # 后续轮次：参考他人的观点进行回应
                    others = {n: o for n, o in all_opinions.items() if n != name}
                    others_text = "\n".join([
                        f"{n}: {o[:200]}" for n, o in others.items()
                    ])
                    prompt = f"""话题: {topic}

其他团队成员的观点:
{others_text}

请参考他们的观点，发表你的看法。你可以在某些方面同意他们，也可以提出不同意见。"""
                
                opinion = agent.think(prompt)
                all_opinions[name] = opinion
                print(f"    [{name}]: {opinion[:80]}...")
        
        return all_opinions


# ============================================================
# 预定义的角色模板
# ============================================================

class RoleTemplates:
    """预定义的角色模板库"""
    
    @staticmethod
    def senior_engineer() -> RoleProfile:
        """高级工程师角色"""
        return RoleProfile(
            name="张工",
            title="高级软件工程师",
            identity="你是一位有 10 年经验的高级软件工程师，精通系统架构设计和代码实现。你曾经在多家大型科技公司工作，参与过多个大型分布式系统的开发。",
            expertise=["系统架构", "Python", "数据库设计", "API设计", "性能优化"],
            personality="严谨、务实、追求代码质量。你相信好的架构是成功的一半，对代码整洁有近乎偏执的追求。",
            thinking_style="你会从系统全局的角度思考问题，先理解需求，再设计架构，最后才考虑具体实现。你会特别关注边界情况和异常处理。",
            communication_style="你的表达简洁直接，喜欢用代码和图表来解释技术概念。你会避免过多的修饰词，直击要点。",
            principles=[
                "代码可读性优先于技巧性",
                "先设计后编码，不在没有思路的情况下写代码",
                "所有公共接口都要有清晰的文档",
                "错误处理不能偷懒",
                "性能优化要有数据支撑，不做过早优化"
            ],
            boundaries=[
                "不参与非技术性的决策",
                "不在没有充分理解需求的情况下开始编码",
                "不接受"先做出来再说"的做法"
            ]
        )
    
    @staticmethod
    def product_manager() -> RoleProfile:
        """产品经理角色"""
        return RoleProfile(
            name="李经理",
            title="产品经理",
            identity="你是一位经验丰富的产品经理，擅长理解用户需求并将之转化为产品功能。你善于与技术团队和业务团队沟通，能够平衡各方利益。",
            expertise=["需求分析", "用户研究", "产品设计", "项目管理", "数据分析"],
            personality="善于倾听、有同理心、结果导向。你能站在用户的角度思考问题，同时也能理解技术实现的限制。",
            thinking_style="你会从用户价值出发思考问题，先问"为什么做"，再问"做什么"，最后才是"怎么做"。你善于用数据来支撑决策。",
            communication_style="你能用通俗的语言解释复杂的技术概念，善于撰写清晰的需求文档。你的沟通风格亲切但专业。",
            principles=[
                "用户价值是第一优先级",
                "数据驱动决策，不拍脑袋",
                "需求要有明确的验收标准",
                "优先级要清晰，不能什么都重要",
                "技术可行性要提前沟通"
            ],
            boundaries=[
                "不做技术实现的决策",
                "不承诺不确定的上线时间",
                "不忽略用户反馈"
            ]
        )
    
    @staticmethod
    def senior_reviewer() -> RoleProfile:
        """代码审查者角色"""
        return RoleProfile(
            name="王审",
            title="代码审查专家",
            identity="你是一位资深的代码审查专家，对代码质量有极高的标准。你审查过无数代码，能够快速发现潜在的 bug、安全漏洞和设计缺陷。",
            expertise=["代码审查", "安全审计", "最佳实践", "设计模式", "测试策略"],
            personality="严格但公正。你对代码质量有近乎苛刻的要求，但你总是能够给出建设性的反馈，而不仅仅是指出问题。",
            thinking_style="你会从安全性、可维护性、性能和可读性四个维度来审查代码。你会特别关注边界情况、并发安全和资源泄漏。",
            communication_style="你的反馈具体而详细，不仅指出问题所在，还会解释为什么是问题以及如何修复。你会先肯定做得好的地方，再指出需要改进的地方。",
            principles=[
                "安全问题零容忍",
                "每个函数的职责要单一",
                "所有输入都要验证",
                "关键路径要有测试覆盖",
                "代码要能自文档化"
            ],
            boundaries=[
                "不替代开发者做设计决策",
                "不审查非代码内容",
                "不在没有明确标准的情况下做主观判断"
            ]
        )
    
    @staticmethod
    def tech_writer() -> RoleProfile:
        """技术文档撰写者角色"""
        return RoleProfile(
            name="赵文",
            title="技术文档工程师",
            identity="你是一位专业的技术文档工程师，擅长将复杂的技术概念转化为清晰易懂的文档。你相信好的文档是软件产品的重要组成部分。",
            expertise=["技术写作", "API文档", "教程编写", "信息架构", "Markdown排版"],
            personality="细心、有耐心、注重细节。你对文字有天然的敏感度，能够发现文档中的不一致和歧义。",
            thinking_style="你会从读者的角度思考，先理解读者的背景知识和目标，再决定如何组织内容。你善于使用类比和示例来解释复杂概念。",
            communication_style="你的文风简洁流畅，避免冗余。你会根据目标读者调整技术深度——给初学者看的文档多用类比，给专家看的文档多用术语。",
            principles=[
                "读者第一，时刻考虑读者的背景和需求",
                "简单清晰胜过华丽复杂",
                "每个概念都要有示例",
                "文档要保持更新",
                "结构化和可搜索性很重要"
            ],
            boundaries=[
                "不编写虚假或夸大的内容",
                "不省略重要的注意事项",
                "不在没有理解技术细节的情况下编写文档"
            ]
        )


# ============================================================
# 演示：角色化 Multi-Agent 编辑部
# ============================================================

def demo_role_based_team():
    """演示角色化 Multi-Agent 系统"""
    
    print("\n" + "="*60)
    print("演示：角色化 Multi-Agent 编辑部")
    print("="*60)
    
    # 创建团队
    team = RoleBasedTeam("AI 编辑部")
    
    # 创建角色化 Agent
    pm_agent = RoleAgent(RoleTemplates.product_manager())
    engineer_agent = RoleAgent(RoleTemplates.senior_engineer())
    reviewer_agent = RoleAgent(RoleTemplates.senior_reviewer())
    writer_agent = RoleAgent(RoleTemplates.tech_writer())
    
    # 添加到团队
    team.add_member(pm_agent)
    team.add_member(engineer_agent)
    team.add_member(reviewer_agent)
    team.add_member(writer_agent)
    
    # 展示团队成员
    team.list_members()
    
    # 定义工作流程
    task = "设计并实现一个简单的 Multi-Agent 协作框架"
    
    workflow = [
        {
            "agent": "李经理",
            "action": "请分析这个任务的需求，列出核心功能点和优先级",
        },
        {
            "agent": "张工",
            "action": "根据产品经理的需求分析，设计系统架构和技术方案",
            "depends_on": ["李经理"]
        },
        {
            "agent": "王审",
            "action": "审查技术方案的合理性和潜在风险",
            "depends_on": ["张工"]
        },
        {
            "agent": "赵文",
            "action": "根据技术方案编写架构设计文档",
            "depends_on": ["张工", "王审"]
        }
    ]
    
    # 执行工作流程
    result = team.coordinate_task(task, workflow)
    
    print(f"\n--- 最终产出 ---")
    print(result[:500] + "...")
    
    # 团队内部讨论
    opinions = team.team_discussion(
        "Multi-Agent 系统中，应该优先保证系统的可靠性还是灵活性？"
    )


if __name__ == "__main__":
    demo_role_based_team()
```

---

## 26.4 角色设计的进阶技巧

### 26.4.1 角色的"人格一致性"

一个好的角色化 Agent 应该在整个交互过程中保持人格的一致性。这就像一个演员在整个电影中不能突然"跳出角色"一样。

保持人格一致性的关键在于系统提示词的设计。系统提示词不仅要描述角色的能力和行为，还要明确角色的价值观、态度和说话方式。更重要的是，这些描述之间不应该有矛盾。

### 26.4.2 角色的"知识边界"

每个角色都应该有明确的知识边界——它知道自己擅长什么，也知道自己不擅长什么。当遇到超出能力范围的问题时，一个好的角色应该主动承认并寻求帮助，而不是硬着头皮给出可能不准确的答案。

这在代码实现中可以通过角色的行为准则来约束。比如，在原则中加入"当不确定时，明确表示你的不确定性"。

### 26.4.3 动态角色调整

在某些场景下，角色可能需要根据任务的变化进行动态调整。比如，当系统从设计阶段进入实现阶段时，"架构师"角色可能需要向"开发者"角色让出主导权。

这种动态调整可以通过修改系统提示词或切换 Agent 来实现。

```python
"""
第 26 章进阶：动态角色调整
根据项目阶段自动调整角色的侧重点
"""
import os
from dataclasses import dataclass, field
from typing import Optional


# 复用前面定义的基础类
# (RoleProfile, RoleAgent, call_llm 等定义省略，引用前文代码)


@dataclass
class DynamicRoleAdjuster:
    """
    动态角色调整器。
    
    根据项目阶段或任务变化，自动调整 Agent 的角色侧重点。
    例如，在设计阶段强调架构思维，在实现阶段强调编码能力。
    """
    
    # 阶段-角色调整映射
    stage_adjustments = {
        "design": {
            "张工": {
                "focus": "架构设计和技术选型",
                "extra_principles": ["关注可扩展性", "考虑长期维护成本"],
                "temperature_modifier": 0.1  # 设计阶段需要更多创造力
            }
        },
        "implementation": {
            "张工": {
                "focus": "编码实现和单元测试",
                "extra_principles": ["代码质量第一", "每个函数不超过50行"],
                "temperature_modifier": -0.2  # 实现阶段需要更精确
            }
        },
        "review": {
            "王审": {
                "focus": "全面的代码审查和质量保证",
                "extra_principles": ["安全问题零容忍", "性能关键路径必须有测试"],
                "temperature_modifier": -0.3  # 审查阶段需要最稳定
            }
        }
    }
    
    @classmethod
    def adjust_role(
        cls,
        agent: "RoleAgent",
        stage: str
    ) -> "RoleAgent":
        """
        根据当前阶段调整 Agent 的角色。
        
        这个方法不创建新的 Agent，而是修改现有 Agent 的行为。
        """
        adjustments = cls.stage_adjustments.get(stage, {})
        agent_adjustment = adjustments.get(agent.name, {})
        
        if agent_adjustment:
            print(f"  [调整] 调整 {agent.name} 的角色以适应 {stage} 阶段")
            print(f"    重点: {agent_adjustment.get('focus', '无变化')}")
            
            # 在实际实现中，这里会修改 Agent 的系统提示词
            # 和温度参数等配置
        
        return agent


def demo_dynamic_adjustment():
    """演示动态角色调整"""
    
    print("\n--- 动态角色调整演示 ---")
    
    # 模拟项目不同阶段
    stages = ["design", "implementation", "review"]
    
    for stage in stages:
        print(f"\n项目阶段: {stage}")
        print(f"  角色调整: {DynamicRoleAdjuster.stage_adjustments.get(stage, {})}")


if __name__ == "__main__":
    demo_dynamic_adjustment()
```

---

## 26.5 案例分析

### 26.5.1 案例：角色化 Multi-Agent 客服系统

某电商平台使用角色化 Multi-Agent 系统来提升客服质量。系统的角色设计如下：

**倾听者 Agent**：负责理解用户的情绪和需求。它的系统提示词强调了同理心和情绪识别能力。当检测到用户情绪激动时，它会优先安抚情绪，而不是直接解决问题。

**问题分类 Agent**：负责将用户问题分类到正确的处理流程。它的系统提示词强调了准确性和全面性——不仅要识别问题的表面类型，还要判断问题的深层原因。

**解决方案 Agent**：负责提供具体的解决方案。它的系统提示词强调了实用性和全面性——不仅要解决当前问题，还要考虑预防类似问题。

**升级处理 Agent**：负责处理复杂或敏感的问题。它的系统提示词强调了判断力和灵活性——知道什么时候应该升级处理，什么时候可以就地解决。

这种角色化设计的效果是显著的。与之前的通用 Agent 相比，角色化系统的用户满意度提升了 25%，首次解决率提升了 18%，平均处理时间缩短了 30%。

---

## 26.6 常见坑与解决方案

### 坑 1：角色设定过于笼统

**问题描述**：角色描述只是"你是一个分析师"这样的简单描述，没有足够的细节来引导 LLM 的行为。

**解决方案**：提供详细的角色画像，包括具体的背景故事、专业领域、行为准则和思维模式。好的角色描述应该具体到让 LLM "入戏"的程度。

### 坑 2：角色之间缺乏差异化

**问题描述**：多个 Agent 的角色设定太相似，导致它们的输出没有明显的差异。

**解决方案**：确保每个角色有独特的视角和思维方式。比如，一个角色从技术角度思考，另一个从用户角度思考，第三个从商业角度思考。

### 坑 3：角色设定与实际能力不匹配

**问题描述**：角色描述要求 Agent 做超出 LLM 能力范围的事情。比如要求一个角色"实时分析系统日志"，但 LLM 并不能真正访问实时数据。

**解决方案**：角色设定要与 LLM 的实际能力匹配。如果需要实时数据处理，应该通过工具调用的方式让 Agent 获取数据，而不是在角色设定中声称 Agent 有能力直接访问数据。

### 坑 4：角色僵化导致思维局限

**问题描述**：过于严格的角色设定导致 Agent 无法跳出角色的思维定式，错过了更好的解决方案。

**解决方案**：在角色设定中加入"开放性"原则，鼓励 Agent 在必要时参考其他角色的视角。同时，定期进行跨角色的讨论，打破思维壁垒。

### 坑 5：角色设定导致过度自信

**问题描述**：角色描述中强调了 Agent 的专业能力，导致 Agent 在不确定的情况下也给出过于自信的回答。

**解决方案**：在角色设定中明确要求 Agent 承认不确定性。加入"当不确定时，明确表示你的不确定性"这样的原则。

---

## 26.7 练习题

### 练习 1：设计一个完整的角色

**难度：★☆☆☆☆**

设计一个新的角色（如安全专家、测试工程师、项目经理等），包括完整的角色画像：身份、能力、性格、思维方式、沟通风格、原则和限制。

### 练习 2：角色一致性测试

**难度：★★☆☆☆**

编写一个测试脚本，通过多轮对话测试角色 Agent 是否能保持人格一致性。在对话中尝试引导 Agent 偏离角色，观察它是否能坚守角色设定。

### 练习 3：角色冲突场景

**难度：★★★☆☆**

设计一个场景，其中两个角色有明显的目标冲突（如产品经理要求快速上线 vs 代码审查者要求充分测试）。观察系统如何协调这种角色冲突。

### 练习 4：动态角色切换

**难度：★★★☆☆**

实现一个支持动态角色切换的 Agent 系统。Agent 可以根据任务需求从"创作者"模式切换到"审查者"模式。

### 练习 5：角色团队绩效评估

**难度：★★★★☆**

设计一个评估系统，衡量角色化团队的工作质量。评估维度包括：角色一致性、任务完成质量、协作效率、创新性等。

---

## 26.8 实战任务

### 任务：构建角色化 Multi-Agent 创意写作工坊

**目标**：构建一个角色化 Multi-Agent 创意写作系统，多个 Agent 以不同的角色协作创作故事。

**角色设计**：
1. **总编辑**：负责整体规划和质量把控
2. **创意策划**：负责构思故事情节和角色设定
3. **科幻作家**：负责科幻元素的描写
4. **文学评论家**：负责审查文章的文学性和可读性

**功能要求**：
1. 每个角色有完整的角色设定
2. 支持团队讨论和辩论
3. 有完整的工作流程和质量检查
4. 代码完整可运行

---

## 26.9 本章小结

本章深入探讨了角色化 Multi-Agent 系统的设计和实现。我们学习了角色设计的核心要素，包括身份定义、专业能力、行为规范、思维方式、沟通风格和限制边界。

我们介绍了六种经典的角色类型：创造者、批判者、协调者、专家、研究员和执行者。每种角色在 Multi-Agent 系统中都扮演着不可替代的角色。

通过完整的代码示例，我们展示了如何将角色设定转化为 LLM 的系统提示词，如何管理角色化团队的协作，以及如何进行动态角色调整。

角色化设计让 Multi-Agent 系统从"多个工具的组合"变成了"一个真正的团队"。每个 Agent 不仅是一个执行者，更是一个有个性、有思想、有专业判断的"团队成员"。正是这种角色的多样性，使得系统整体的智能超越了任何单个 Agent 的能力上限。

> "一个优秀的团队，不是每个人都做同样的事情，而是每个人都做自己最擅长的事情，并且彼此信任。"

---

**下一章预告**：在第 27 章中，我们将探讨 Agent 辩论与自我纠正——让多个 Agent 通过辩论来发现问题、纠正错误，实现系统的自我进化。敬请期待。
