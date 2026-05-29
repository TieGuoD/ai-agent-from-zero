# 第54章：Agent 与浏览器 —— Web Agent

## 学习目标

通过本章的学习，你将能够：
1. 理解 Web Agent 的工作原理和应用场景
2. 掌握浏览器自动化技术（Playwright、Selenium）
3. 学会构建能够理解和操作网页的 Agent
4. 理解网页内容提取和交互的实现方法
5. 掌握 Web Agent 的安全和伦理考虑
6. 能够构建完整的 Web Agent 应用

## 核心问题

互联网是信息的海洋，但大部分信息需要通过浏览器访问。传统的爬虫只能获取静态内容，无法处理需要交互的页面（如登录、搜索、点击等）。

Web Agent 是一种能够像人类一样使用浏览器的 Agent。它可以：
1. 打开网页并理解页面内容
2. 填写表单、点击按钮
3. 导航到不同的页面
4. 提取需要的信息
5. 完成复杂的在线任务

这就像是给 Agent 配了一双"手"和一双"眼睛"，让它能够像人类一样浏览网页。

---

## 原理讲解

### Web Agent 的核心组件

一个完整的 Web Agent 系统包括以下几个核心组件：

**1. 浏览器控制器（Browser Controller）**
负责与浏览器交互，包括打开页面、点击元素、输入文本等操作。

**2. 页面理解器（Page Interpreter）**
理解页面的结构和内容，识别可交互的元素。

**3. 任务规划器（Task Planner）**
将高层任务分解为具体的浏览器操作序列。

**4. 状态管理器（State Manager）**
跟踪 Agent 的当前状态和历史操作。

### 浏览器自动化技术

目前主流的浏览器自动化技术包括：

**Playwright**：微软开发的现代浏览器自动化框架，支持 Chromium、Firefox、WebKit。

**Selenium**：经典的浏览器自动化框架，支持多种浏览器。

**Puppeteer**：Google 开发的 Node.js 库，主要用于 Chrome。

### 页面理解方法

Agent 需要理解页面内容才能做出正确的操作：

**DOM 分析**：分析页面的 HTML 结构，识别元素和属性。

**视觉理解**：使用多模态模型理解页面的视觉内容。

**无障碍树**：利用浏览器的无障碍树获取页面的语义结构。

### 任务分解策略

复杂的网页任务需要分解为简单的操作序列：

**顺序执行**：操作按顺序依次执行。

**条件分支**：根据页面状态选择不同的操作路径。

**循环操作**：重复执行某些操作直到满足条件。

### 错误处理

Web Agent 面临多种可能的错误：

**页面加载失败**：网络问题或服务器错误。

**元素找不到**：页面结构变化或选择器错误。

**操作超时**：页面响应太慢。

**验证码**：需要人工干预的情况。

---

## 完整代码示例

### 示例 1：基础 Web Agent

```python
"""
基础 Web Agent
使用 Playwright 实现浏览器自动化
"""

import asyncio
import json
import os
from typing import Dict, List, Optional
from dataclasses import dataclass, field
from playwright.async_api import async_playwright, Page, Browser


@dataclass
class BrowserAction:
    """浏览器操作"""
    action_type: str
    target: str  # CSS 选择器或 URL
    value: Optional[str] = None
    description: str = ""


@dataclass
class PageState:
    """页面状态"""
    url: str
    title: str
    content: str
    elements: List[Dict]
    screenshot_path: Optional[str] = None


class WebAgent:
    """Web Agent"""
    
    def __init__(self):
        self.browser: Optional[Browser] = None
        self.page: Optional[Page] = None
        self.history: List[BrowserAction] = []
        self.state: Optional[PageState] = None
    
    async def start(self, headless: bool = True):
        """启动浏览器"""
        self.playwright = await async_playwright().start()
        self.browser = await self.playwright.chromium.launch(headless=headless)
        self.page = await self.browser.new_page()
        print("浏览器已启动")
    
    async def stop(self):
        """停止浏览器"""
        if self.browser:
            await self.browser.close()
        if self.playwright:
            await self.playwright.stop()
        print("浏览器已关闭")
    
    async def navigate(self, url: str) -> PageState:
        """导航到 URL"""
        print(f"\n导航到: {url}")
        
        await self.page.goto(url, wait_until="networkidle")
        await asyncio.sleep(1)  # 等待页面加载
        
        # 获取页面状态
        self.state = await self._get_page_state()
        
        self.history.append(BrowserAction(
            action_type="navigate",
            target=url,
            description=f"导航到 {url}",
        ))
        
        print(f"  页面标题: {self.state.title}")
        print(f"  页面内容长度: {len(self.state.content)} 字符")
        
        return self.state
    
    async def click(self, selector: str) -> bool:
        """点击元素"""
        print(f"\n点击元素: {selector}")
        
        try:
            await self.page.click(selector, timeout=5000)
            await asyncio.sleep(0.5)
            
            self.state = await self._get_page_state()
            
            self.history.append(BrowserAction(
                action_type="click",
                target=selector,
                description=f"点击 {selector}",
            ))
            
            print("  点击成功")
            return True
            
        except Exception as e:
            print(f"  点击失败: {e}")
            return False
    
    async def fill(self, selector: str, value: str) -> bool:
        """填写表单"""
        print(f"\n填写表单: {selector} = {value}")
        
        try:
            await self.page.fill(selector, value, timeout=5000)
            
            self.history.append(BrowserAction(
                action_type="fill",
                target=selector,
                value=value,
                description=f"填写 {selector}",
            ))
            
            print("  填写成功")
            return True
            
        except Exception as e:
            print(f"  填写失败: {e}")
            return False
    
    async def extract_text(self, selector: str = None) -> str:
        """提取文本"""
        try:
            if selector:
                text = await self.page.inner_text(selector)
            else:
                text = await self.page.inner_text("body")
            
            return text.strip()
            
        except Exception as e:
            print(f"  提取失败: {e}")
            return ""
    
    async def screenshot(self, path: str = "screenshot.png") -> str:
        """截图"""
        await self.page.screenshot(path=path, full_page=True)
        print(f"  截图保存到: {path}")
        return path
    
    async def wait_for_element(self, selector: str, timeout: int = 10000) -> bool:
        """等待元素出现"""
        try:
            await self.page.wait_for_selector(selector, timeout=timeout)
            return True
        except:
            return False
    
    async def _get_page_state(self) -> PageState:
        """获取页面状态"""
        url = self.page.url
        title = await self.page.title()
        content = await self.page.content()
        text = await self.page.inner_text("body")
        
        # 提取可交互元素
        elements = await self.page.evaluate("""
            () => {
                const elements = [];
                const interactiveTags = ['a', 'button', 'input', 'select', 'textarea'];
                
                document.querySelectorAll('*').forEach(el => {
                    if (interactiveTags.includes(el.tagName.toLowerCase()) || 
                        el.onclick || el.getAttribute('role') === 'button') {
                        elements.push({
                            tag: el.tagName.toLowerCase(),
                            text: el.innerText?.substring(0, 50) || '',
                            href: el.href || '',
                            type: el.type || '',
                            id: el.id || '',
                            className: el.className || '',
                        });
                    }
                });
                
                return elements.slice(0, 50);  // 限制数量
            }
        """)
        
        return PageState(
            url=url,
            title=title,
            content=text[:5000],  # 限制长度
            elements=elements,
        )
    
    async def execute_task(self, task: str) -> Dict:
        """执行网页任务"""
        print(f"\n{'='*60}")
        print(f"执行任务: {task}")
        print(f"{'='*60}")
        
        # 这里应该集成 LLM 来规划任务
        # 简化实现：直接返回当前状态
        
        return {
            "task": task,
            "status": "completed",
            "state": {
                "url": self.state.url if self.state else "",
                "title": self.state.title if self.state else "",
            },
        }


async def demo_web_agent():
    """演示 Web Agent"""
    print("=" * 60)
    print("Web Agent 演示")
    print("=" * 60)
    
    agent = WebAgent()
    
    try:
        # 启动浏览器
        await agent.start(headless=True)
        
        # 导航到示例页面
        state = await agent.navigate("https://example.com")
        
        print(f"\n页面内容预览:")
        print(state.content[:500])
        
        print(f"\n可交互元素:")
        for elem in state.elements[:10]:
            print(f"  {elem['tag']}: {elem.get('text', '')[:30]}")
        
        # 截图
        await agent.screenshot("example_page.png")
        
    finally:
        await agent.stop()


if __name__ == "__main__":
    asyncio.run(demo_web_agent())
```

### 示例 2：智能网页信息提取器

```python
"""
智能网页信息提取器
理解页面内容并提取结构化信息
"""

import asyncio
import json
import re
from typing import Dict, List, Optional
from dataclasses import dataclass, field
import anthropic
from playwright.async_api import async_playwright


@dataclass
class ExtractedInfo:
    """提取的信息"""
    category: str
    data: Dict
    confidence: float
    source_url: str


class SmartWebExtractor:
    """智能网页提取器"""
    
    def __init__(self):
        self.client = anthropic.Anthropic(
            api_key=os.environ.get("ANTHROPIC_API_KEY")
        )
    
    async def extract(self, url: str, extraction_goal: str) -> ExtractedInfo:
        """从网页提取信息"""
        print(f"\n提取目标: {extraction_goal}")
        print(f"来源: {url}")
        
        # 获取页面内容
        content = await self._fetch_page(url)
        
        # 使用 LLM 理解内容
        extracted = self._analyze_content(content, extraction_goal)
        
        return ExtractedInfo(
            category=extraction_goal,
            data=extracted.get("data", {}),
            confidence=extracted.get("confidence", 0),
            source_url=url,
        )
    
    async def _fetch_page(self, url: str) -> str:
        """获取页面内容"""
        async with async_playwright() as p:
            browser = await p.chromium.launch(headless=True)
            page = await browser.new_page()
            
            try:
                await page.goto(url, wait_until="networkidle", timeout=30000)
                content = await page.inner_text("body")
                await browser.close()
                return content[:10000]  # 限制长度
            except Exception as e:
                print(f"获取页面失败: {e}")
                await browser.close()
                return ""
    
    def _analyze_content(self, content: str, goal: str) -> Dict:
        """分析页面内容"""
        prompt = f"""从以下网页内容中提取相关信息。

提取目标：{goal}

网页内容：
{content[:5000]}

请用 JSON 格式回答：
{{
    "data": {{
        // 根据提取目标动态生成字段
    }},
    "confidence": 0-1,
    "notes": "提取说明"
}}"""
        
        response = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=2048,
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
        
        return {"data": {}, "confidence": 0}


def demo_extractor():
    """演示提取器"""
    print("=" * 60)
    print("智能网页提取器演示")
    print("=" * 60)
    
    extractor = SmartWebExtractor()
    
    result = asyncio.run(extractor.extract(
        url="https://example.com",
        extraction_goal="页面的主要内容和目的"
    ))
    
    print(f"\n提取结果:")
    print(f"  类别: {result.category}")
    print(f"  置信度: {result.confidence}")
    print(f"  数据: {json.dumps(result.data, ensure_ascii=False, indent=2)}")


if __name__ == "__main__":
    demo_extractor()
```

---

### 示例 3：LLM 驱动的智能 Web Agent

前面两个示例分别展示了浏览器控制和网页内容提取。但如果要让 Agent 真正"像人一样使用浏览器"，就需要将 LLM 的推理能力与浏览器操作结合起来。下面这个示例展示了一个完整的 LLM 驱动的 Web Agent，它能够理解自然语言指令，自动规划操作步骤，并逐步执行。

```python
"""
LLM 驱动的智能 Web Agent
结合 LLM 推理能力与浏览器自动化
"""

import asyncio
import json
import os
from typing import Dict, List, Optional
from dataclasses import dataclass, field
from playwright.async_api import async_playwright, Page
import anthropic


@dataclass
class AgentStep:
    """Agent 执行的一步操作"""
    step_id: int
    action: str
    target: str
    value: Optional[str] = None
    result: str = ""
    success: bool = True


class LLMWebAgent:
    """LLM 驱动的 Web Agent。

    工作流程：
    1. 用户给出自然语言指令
    2. LLM 分析当前页面状态，规划下一步操作
    3. 浏览器控制器执行操作
    4. 获取新的页面状态，反馈给 LLM
    5. 重复 2-4 直到任务完成
    """

    def __init__(self, api_key: Optional[str] = None):
        self.client = anthropic.Anthropic(
            api_key=api_key or os.environ.get("ANTHROPIC_API_KEY")
        )
        self.page: Optional[Page] = None
        self.steps: List[AgentStep] = []
        self.step_counter = 0

    async def start(self, headless: bool = True):
        """启动浏览器。"""
        self.playwright = await async_playwright().start()
        self.browser = await self.playwright.chromium.launch(
            headless=headless,
            args=[
                "--disable-blink-features=AutomationControlled",
                "--no-sandbox",
            ]
        )
        context = await self.browser.new_context(
            viewport={"width": 1280, "height": 720},
            user_agent=(
                "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
                "AppleWebKit/537.36 (KHTML, like Gecko) "
                "Chrome/120.0.0.0 Safari/537.36"
            ),
        )
        self.page = await context.new_page()
        print("[Agent] 浏览器已启动")

    async def stop(self):
        """关闭浏览器。"""
        if self.browser:
            await self.browser.close()
        if self.playwright:
            await self.playwright.stop()
        print("[Agent] 浏览器已关闭")

    async def _get_page_summary(self) -> str:
        """获取页面摘要，用于发送给 LLM。"""
        title = await self.page.title()
        url = self.page.url
        text = await self.page.inner_text("body")
        # 限制文本长度，避免 Token 过多
        text = text[:3000]

        # 获取可交互元素
        elements = await self.page.evaluate("""
            () => {
                const results = [];
                const tags = ['a', 'button', 'input', 'select', 'textarea'];
                document.querySelectorAll('*').forEach((el, idx) => {
                    if (tags.includes(el.tagName.toLowerCase()) ||
                        el.getAttribute('role') === 'button') {
                        const rect = el.getBoundingClientRect();
                        if (rect.width > 0 && rect.height > 0) {
                            results.push({
                                index: idx,
                                tag: el.tagName.toLowerCase(),
                                text: (el.innerText || '').substring(0, 60),
                                type: el.type || '',
                                placeholder: el.placeholder || '',
                                aria_label: el.getAttribute('aria-label') || '',
                            });
                        }
                    }
                });
                return results.slice(0, 30);
            }
        """)

        elements_str = json.dumps(elements, ensure_ascii=False, indent=2)

        return f"""当前页面状态：
URL: {url}
标题: {title}
页面文本（前2000字）：
{text[:2000]}

可交互元素：
{elements_str}"""

    def _build_system_prompt(self, task: str) -> str:
        """构建系统提示词。"""
        return f"""你是一个 Web Agent，能够像人类一样使用浏览器完成任务。

你的任务：{task}

你只能使用以下操作：
1. navigate(url): 导航到指定 URL
2. click(selector): 点击页面元素。selector 可以是 CSS 选择器，或者是 "#index-N" 格式引用可交互元素列表中的元素
3. fill(selector, value): 在输入框中填写内容
4. scroll_down(): 向下滚动页面
5. scroll_up(): 向上滚动页面
6. wait(seconds): 等待指定秒数
7. done(answer): 任务完成，返回最终结果

每次只能执行一个操作。请根据当前页面状态，决定下一步操作。

请严格按以下 JSON 格式回复：
{{
    "thought": "你的思考过程，分析当前页面状态和下一步计划",
    "action": "操作名称",
    "params": {{操作参数}}
}}

示例：
{{"thought": "我看到了搜索框，需要输入查询内容", "action": "fill", "params": {{"selector": "input[type='search']", "value": "Python 教程"}}}}
{{"thought": "我已经获得了搜索结果，任务完成", "action": "done", "params": {{"answer": "找到了相关结果"}}}}"""

    async def execute_task(self, task: str, max_steps: int = 15) -> Dict:
        """执行自然语言任务。

        Args:
            task: 自然语言任务描述
            max_steps: 最大步骤数

        Returns:
            任务执行结果
        """
        print(f"\n{'='*60}")
        print(f"[Agent] 开始执行任务: {task}")
        print(f"{'='*60}")

        messages = []

        for step_num in range(max_steps):
            # 获取当前页面状态
            page_summary = await self._get_page_summary()

            # 构建消息
            if step_num == 0:
                user_msg = f"请开始执行任务。\n\n{page_summary}"
            else:
                user_msg = f"操作执行结果：{self.steps[-1].result}\n\n{page_summary}"

            messages.append({"role": "user", "content": user_msg})

            # 调用 LLM 决定下一步
            response = self.client.messages.create(
                model="claude-sonnet-4-20250514",
                max_tokens=1024,
                system=self._build_system_prompt(task),
                messages=messages,
            )

            llm_text = response.content[0].text

            # 解析 LLM 的决策
            action_plan = self._parse_action(llm_text)

            # 添加助手消息到历史
            messages.append({"role": "assistant", "content": llm_text})

            # 如果是 done 操作，结束
            if action_plan["action"] == "done":
                answer = action_plan.get("params", {}).get("answer", "任务已完成")
                print(f"\n[Agent] 任务完成: {answer}")
                return {
                    "task": task,
                    "status": "completed",
                    "answer": answer,
                    "steps": [s.__dict__ for s in self.steps],
                    "total_steps": len(self.steps),
                }

            # 执行操作
            step_result = await self._execute_action(action_plan)

            self.step_counter += 1
            step = AgentStep(
                step_id=self.step_counter,
                action=action_plan["action"],
                target=json.dumps(action_plan.get("params", {}), ensure_ascii=False),
                result=step_result,
            )
            self.steps.append(step)

            print(f"  [步骤 {step_num+1}] {step.action} -> {step_result[:100]}")

        return {
            "task": task,
            "status": "max_steps_reached",
            "steps": [s.__dict__ for s in self.steps],
            "total_steps": len(self.steps),
        }

    def _parse_action(self, llm_text: str) -> Dict:
        """解析 LLM 返回的操作指令。"""
        try:
            # 尝试提取 JSON
            start = llm_text.find('{')
            end = llm_text.rfind('}') + 1
            if start >= 0 and end > start:
                return json.loads(llm_text[start:end])
        except json.JSONDecodeError:
            pass

        # 回退到默认
        return {"action": "done", "params": {"answer": "无法解析操作指令"}}

    async def _execute_action(self, action_plan: Dict) -> str:
        """执行一个操作。"""
        action = action_plan.get("action", "")
        params = action_plan.get("params", {})

        try:
            if action == "navigate":
                url = params.get("url", "")
                await self.page.goto(url, wait_until="networkidle", timeout=15000)
                await asyncio.sleep(1)
                return f"已导航到 {url}"

            elif action == "click":
                selector = params.get("selector", "")
                # 支持 #index-N 格式的元素引用
                if selector.startswith("#index-"):
                    idx = int(selector.split("-")[1])
                    element = await self.page.evaluate(f"""
                        () => {{
                            const tags = ['a', 'button', 'input', 'select', 'textarea'];
                            const elements = [];
                            document.querySelectorAll('*').forEach(el => {{
                                if (tags.includes(el.tagName.toLowerCase())) {{
                                    const rect = el.getBoundingClientRect();
                                    if (rect.width > 0 && rect.height > 0) {{
                                        elements.push(el);
                                    }}
                                }}
                            }});
                            return elements[{idx}] ? true : false;
                        }}
                    """)
                    if element:
                        # 使用 JavaScript 点击
                        await self.page.evaluate(f"""
                            () => {{
                                const tags = ['a', 'button', 'input', 'select', 'textarea'];
                                const elements = [];
                                document.querySelectorAll('*').forEach(el => {{
                                    if (tags.includes(el.tagName.toLowerCase())) {{
                                        const rect = el.getBoundingClientRect();
                                        if (rect.width > 0 && rect.height > 0) {{
                                            elements.push(el);
                                        }}
                                    }}
                                }});
                                elements[{idx}].click();
                            }}
                        """)
                        await asyncio.sleep(1)
                        return f"已点击元素 #{idx}"
                    else:
                        return f"元素 #{idx} 不存在"
                else:
                    await self.page.click(selector, timeout=5000)
                    await asyncio.sleep(1)
                    return f"已点击 {selector}"

            elif action == "fill":
                selector = params.get("selector", "")
                value = params.get("value", "")
                await self.page.fill(selector, value, timeout=5000)
                return f"已在 {selector} 中填写 '{value}'"

            elif action == "scroll_down":
                await self.page.evaluate("window.scrollBy(0, 500)")
                await asyncio.sleep(0.5)
                return "已向下滚动"

            elif action == "scroll_up":
                await self.page.evaluate("window.scrollBy(0, -500)")
                await asyncio.sleep(0.5)
                return "已向上滚动"

            elif action == "wait":
                seconds = params.get("seconds", 1)
                await asyncio.sleep(seconds)
                return f"等待了 {seconds} 秒"

            elif action == "done":
                return params.get("answer", "任务完成")

            else:
                return f"未知操作: {action}"

        except Exception as e:
            return f"操作失败: {str(e)}"


async def demo_llm_web_agent():
    """演示 LLM 驱动的 Web Agent。"""
    agent = LLMWebAgent()

    try:
        await agent.start(headless=True)

        # 示例任务：在维基百科上搜索信息
        result = await agent.execute_task(
            "访问维基百科，搜索 'Artificial Intelligence'，并告诉我页面的前三个段落讲了什么",
            max_steps=10,
        )

        print(f"\n{'='*60}")
        print("执行结果:")
        print(f"  状态: {result['status']}")
        print(f"  回答: {result.get('answer', 'N/A')}")
        print(f"  总步骤数: {result['total_steps']}")

    finally:
        await agent.stop()


if __name__ == "__main__":
    asyncio.run(demo_llm_web_agent())
```

这个示例展示了 Web Agent 的核心工作模式：LLM 负责理解和决策，浏览器控制器负责执行。每执行一步操作后，Agent 都会重新观察页面状态，然后由 LLM 决定下一步。这种"观察-思考-行动"的循环，是几乎所有 Web Agent 的基础架构。

---

## 案例分析

### 案例 1：自动化测试 Agent

**场景**：企业需要自动化测试 Web 应用。

一个真实的需求是这样的：某电商平台每次上线新版本前，测试团队需要手动执行 200 多个测试用例，涵盖用户注册、登录、搜索、下单、支付等完整流程。每次回归测试需要 3 天时间，严重影响了发布节奏。

**Agent 功能**：
1. 理解测试用例描述（如"以普通用户身份登录，搜索'手机'，将第一个结果加入购物车"）
2. 自动生成 Playwright 测试脚本
3. 执行测试并截图，记录每一步的页面状态
4. 分析测试结果，对比预期结果和实际结果
5. 生成测试报告，包括通过率、失败原因、截图证据

**实现要点**：这个 Agent 的核心是将自然语言测试用例转化为可执行的测试步骤。LLM 负责理解测试用例的语义，浏览器控制器负责执行操作。关键的技术挑战是如何处理动态页面内容——页面上的商品列表每次加载可能不同，Agent 需要能够识别和操作任意一个商品条目。

```python
# 自动化测试 Agent 的核心逻辑示例
class TestAgent:
    """自动化测试 Agent 的简化版"""

    def __init__(self, llm_client, browser_agent: LLMWebAgent):
        self.llm = llm_client
        self.browser = browser_agent
        self.test_results = []

    async def run_test_case(self, test_case: str) -> Dict:
        """执行一个测试用例。"""
        # 1. LLM 解析测试用例，生成操作步骤
        steps = self._plan_steps(test_case)

        # 2. 逐步执行
        for i, step in enumerate(steps):
            await self.browser._execute_action(step)
            # 截图记录
            screenshot_path = f"test_step_{i+1}.png"
            await self.browser.page.screenshot(path=screenshot_path)

        # 3. 验证结果
        verification = self._verify_result(test_case)

        return {
            "test_case": test_case,
            "steps_executed": len(steps),
            "passed": verification["passed"],
            "screenshots": [f"test_step_{i+1}.png" for i in range(len(steps))],
        }

    def _plan_steps(self, test_case: str) -> List[Dict]:
        """让 LLM 将测试用例分解为操作步骤。"""
        prompt = f"""将以下测试用例分解为浏览器操作步骤：

测试用例：{test_case}

可用操作：navigate(url), click(selector), fill(selector, value), scroll_down()

请返回 JSON 格式的操作步骤列表。"""

        response = self.llm.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            messages=[{"role": "user", "content": prompt}],
        )
        # 解析并返回步骤列表
        return self._parse_steps(response.content[0].text)

    def _verify_result(self, test_case: str) -> Dict:
        """验证测试结果。"""
        # 简化实现：检查页面是否包含预期内容
        return {"passed": True, "reason": "所有检查通过"}
```

### 案例 2：价格监控 Agent

**场景**：监控电商网站的价格变化。

一个常见的需求是：某采购经理需要跟踪 50 种原材料在不同供应商网站上的价格变化，以便在价格低点时集中采购。手动检查每个网站每天需要花费 2 小时。

**Agent 功能**：
1. 定期访问目标页面（使用 Playwright 打开浏览器，模拟真实用户访问）
2. 提取价格信息（通过 CSS 选择器或 LLM 理解页面结构来定位价格元素）
3. 检测价格变化（与历史价格对比，计算涨跌幅）
4. 发送通知（价格变动超过阈值时，通过邮件或即时通讯工具通知）
5. 生成价格趋势报告（按时间维度展示价格走势）

**实现要点**：价格监控的关键挑战是两个——一是反爬虫机制，很多电商网站会检测自动化访问；二是页面结构经常变化，价格元素的选择器可能失效。解决方案是使用真实的浏览器指纹、随机化访问间隔、同时维护多套选择器策略。

```python
# 价格监控 Agent 的核心逻辑示例
import json
from pathlib import Path
from datetime import datetime
from typing import List, Dict


class PriceMonitorAgent:
    """价格监控 Agent"""

    def __init__(self, storage_file: str = "price_history.json"):
        self.storage_file = Path(storage_file)
        self.history = self._load_history()

    def _load_history(self) -> Dict:
        """加载历史价格数据。"""
        if self.storage_file.exists():
            return json.loads(self.storage_file.read_text(encoding="utf-8"))
        return {}

    def _save_history(self):
        """保存历史价格数据。"""
        self.storage_file.write_text(
            json.dumps(self.history, ensure_ascii=False, indent=2),
            encoding="utf-8",
        )

    def record_price(self, product_id: str, price: float, source: str):
        """记录一次价格。"""
        if product_id not in self.history:
            self.history[product_id] = []

        self.history[product_id].append({
            "price": price,
            "source": source,
            "timestamp": datetime.now().isoformat(),
        })
        self._save_history()

    def check_price_change(self, product_id: str,
                           threshold: float = 0.05) -> Dict:
        """检查价格变化。

        Args:
            product_id: 商品 ID
            threshold: 变化阈值（如 0.05 表示 5%）

        Returns:
            价格变化信息
        """
        if product_id not in self.history:
            return {"changed": False, "reason": "没有历史数据"}

        records = self.history[product_id]
        if len(records) < 2:
            return {"changed": False, "reason": "历史数据不足"}

        latest = records[-1]["price"]
        previous = records[-2]["price"]
        change_pct = (latest - previous) / previous if previous else 0

        result = {
            "changed": abs(change_pct) >= threshold,
            "product_id": product_id,
            "previous_price": previous,
            "current_price": latest,
            "change_pct": f"{change_pct*100:.2f}%",
            "direction": "上涨" if change_pct > 0 else "下跌",
        }

        if result["changed"]:
            result["message"] = (
                f"价格{result['direction']}！{product_id} "
                f"从 {previous} 变为 {latest}（{result['change_pct']}）"
            )

        return result

    def get_price_trend(self, product_id: str) -> List[Dict]:
        """获取价格趋势。"""
        return self.history.get(product_id, [])
```

### 案例 3：内容聚合 Agent

**场景**：自动聚合多个新闻网站的内容。

**Agent 功能**：同时监控多个新闻网站，自动提取文章标题、摘要、发布时间和链接，按照主题分类整理，生成每日新闻简报。这个场景非常适合 Web Agent，因为不同网站的页面结构各不相同，传统的爬虫需要为每个网站单独编写解析规则，而 Web Agent 可以利用 LLM 的理解能力来适应不同的页面布局。

---

## Web Agent 的高级话题

### 反检测与浏览器指纹

网站的反爬虫系统会通过多种方式检测自动化访问：检查 `navigator.webdriver` 属性是否为 true、检测浏览器的 WebGL 渲染指纹、分析鼠标移动轨迹是否自然、检查页面加载时间是否异常。以下是一些基本的反检测策略。

```python
# 反检测策略
async def create_stealth_browser():
    """创建一个难以被检测的浏览器实例。"""
    playwright = await async_playwright().start()

    browser = await playwright.chromium.launch(
        headless=False,  # 非 headless 模式更难被检测
        args=[
            "--disable-blink-features=AutomationControlled",
            "--no-sandbox",
            "--disable-dev-shm-usage",
        ],
    )

    context = await browser.new_context(
        viewport={"width": 1920, "height": 1080},
        user_agent=(
            "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
            "AppleWebKit/537.36 (KHTML, like Gecko) "
            "Chrome/120.0.0.0 Safari/537.36"
        ),
        locale="zh-CN",
        timezone_id="Asia/Shanghai",
    )

    # 注入反检测脚本
    await context.add_init_script("""
        // 隐藏 webdriver 标记
        Object.defineProperty(navigator, 'webdriver', {
            get: () => undefined,
        });

        // 修改 navigator.plugins
        Object.defineProperty(navigator, 'plugins', {
            get: () => [1, 2, 3, 4, 5],
        });

        // 修改 navigator.languages
        Object.defineProperty(navigator, 'languages', {
            get: () => ['zh-CN', 'zh', 'en-US', 'en'],
        });
    """)

    return playwright, browser, context
```

### 页面内容的智能提取

不同的网页有不同的结构，硬编码的 CSS 选择器很容易失效。更好的做法是使用多种策略相结合：首先尝试精确的 CSS 选择器，如果失败则使用 LLM 来理解页面结构并动态生成选择器。

```python
async def smart_extract(page: Page, target_info: str, llm_client) -> str:
    """智能提取页面信息。

    结合 CSS 选择器和 LLM 理解来提取信息。
    """
    # 策略 1：尝试常见的 CSS 选择器
    common_selectors = {
        "价格": [".price", "[data-price]", ".product-price", "#price"],
        "标题": ["h1", ".title", ".article-title"],
        "正文": ["article", ".content", ".post-body", "main"],
    }

    for category, selectors in common_selectors.items():
        if category in target_info:
            for selector in selectors:
                try:
                    element = await page.query_selector(selector)
                    if element:
                        text = await element.inner_text()
                        if text.strip():
                            return text.strip()
                except Exception:
                    continue

    # 策略 2：使用 LLM 分析页面内容
    full_text = await page.inner_text("body")
    prompt = f"""从以下网页内容中提取：{target_info}

网页内容：
{full_text[:3000]}

请直接返回提取到的信息，不要添加额外解释。"""

    response = llm_client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        messages=[{"role": "user", "content": prompt}],
    )

    return response.content[0].text.strip()
```

---

## 常见坑

---

## 常见坑

### 坑 1：反爬虫机制

网站可能检测和阻止自动化访问。

**解决方案**：使用真实的浏览器指纹，添加随机延迟。

### 坑 2：页面动态加载

内容可能通过 JavaScript 动态加载。

**解决方案**：等待页面完全加载，使用 headless 浏览器执行 JS。

### 坑 3：验证码

需要处理验证码才能继续。

**解决方案**：使用验证码识别服务，或请求人工协助。

### 坑 4：页面结构变化

网站更新可能导致选择器失效。

**解决方案**：使用多种选择器策略，定期更新。

### 坑 5：性能问题

浏览器操作可能比较慢。

**解决方案**：并行执行多个任务，使用缓存。

---

## 练习题

### 练习 1：基础浏览器操作（难度：初级）
实现基础的浏览器操作：
- 导航到页面
- 点击链接
- 填写表单

提示：使用 Playwright 的 `page.goto()`、`page.click()`、`page.fill()` 方法。注意处理页面加载等待和元素查找超时。

### 练习 2：信息提取（难度：初级）
实现网页信息提取器：
- 提取文本内容
- 提取链接
- 提取图片

提示：使用 `page.inner_text()` 提取文本，使用 `page.evaluate()` 执行 JavaScript 来提取链接和图片 URL。

### 练习 3：表单自动化（难度：中级）
实现表单自动填写：
- 识别表单字段（输入框、下拉框、复选框）
- 自动填写
- 提交表单

提示：不同类型表单元素的操作方式不同——输入框用 `fill()`，下拉框用 `select_option()`，复选框用 `check()`。可以先用 JavaScript 扫描页面获取所有表单元素的类型和位置。

### 练习 4：页面监控（难度：中级）
实现页面变化监控：
- 定期检查页面（使用 asyncio 定时器）
- 检测内容变化（计算两次抓取内容的相似度）
- 发送通知（通过邮件或 webhook 发送变化提醒）

提示：可以用简单的字符串对比来检测变化，也可以用 difflib 库计算内容差异的详细信息。

### 练习 5：多页面导航（难度：高级）
实现多页面导航：
- 处理弹窗（`page.on("dialog")` 事件监听）
- 处理新标签页（监听 `page.on("popup")` 事件）
- 处理 iframe（使用 `page.frame()` 进入 iframe 上下文）

提示：弹窗处理需要在弹窗出现之前就注册事件监听器。新标签页处理需要在 popup 事件中获取新的 page 对象。

### 练习 6：完整的 Web Agent（难度：高级）
构建一个完整的 Web Agent，能够执行复杂的网页任务。

要求：结合示例 3 的 LLMWebAgent，实现一个能完成以下任务的 Agent：
1. 访问任意网页
2. 理解页面内容
3. 根据用户的自然语言指令执行操作
4. 支持多步骤任务
5. 有完善的错误处理机制

---

## 实战任务

### 任务 1：构建价格监控系统（中等难度）

**功能需求：**
1. 监控多个电商网站
2. 提取价格信息
3. 检测价格变化
4. 发送通知

**技术要求：**
1. 使用 Playwright
2. 实现定时任务
3. 支持多种网站
4. 生成价格报告

### 任务 2：构建自动化测试平台（高难度）

**功能需求：**
1. 理解测试用例
2. 生成测试脚本
3. 执行测试
4. 生成报告

**技术要求：**
1. 使用 Agent 规划测试
2. 支持多种测试场景
3. 实现截图对比
4. 集成 CI/CD

---

## 本章小结

本章我们学习了 Agent 与浏览器的结合，即 Web Agent。我们从以下几个方面进行了探索：

**核心组件**：理解了浏览器控制器、页面理解器、任务规划器和状态管理器四个核心组件。

**浏览器自动化**：掌握了 Playwright 等浏览器自动化技术的使用方法。

**页面理解**：学会了 DOM 分析、视觉理解和无障碍树等页面理解方法。

**任务分解**：理解了如何将复杂任务分解为简单的浏览器操作。

Web Agent 正在改变我们与互联网交互的方式。它可以让重复性的网页操作自动化，帮助用户完成复杂的在线任务。

在下一章中，我们将学习 Agent 与文件系统的结合，看看如何构建文件操作 Agent。

---

## 延伸阅读

1. WebAgent: A Survey (研究论文)
2. WebArena: A Realistic Web Environment (基准测试)
3. Playwright 官方文档
4. Browser Automation Best Practices
5. Web Scraping Ethics (伦理指南)
