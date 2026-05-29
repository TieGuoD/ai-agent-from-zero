# 第 19 章：Output Parsing —— 结构化输出与解析

---

## 学习目标

完成本章学习后，你将能够：

1. 理解结构化输出在 Agent 系统中的重要性——为什么 LLM 的自由文本输出不够用
2. 掌握 JSON、XML、YAML 等格式的解析方法，包括从 LLM 输出中提取结构化数据
3. 能够设计健壮的输出解析器——处理 LLM 输出中的各种格式错误和异常
4. 掌握 Pydantic 在结构化输出中的应用——用数据模型定义输出格式并自动验证
5. 了解 LLM 输出的常见问题（格式错误、截断、类型不匹配）和处理策略
6. 掌握 Prompt 设计技巧——如何让 LLM 输出符合预期格式的内容

## 核心问题

1. 为什么 Agent 需要结构化输出？
2. 如何处理 LLM 输出的不确定性和错误？
3. 如何设计可扩展的解析器？

---

## 19.1 为什么需要结构化输出

### 19.1.1 LLM 输出的问题

LLM 的原始输出是自由文本——它想怎么写就怎么写。这在聊天场景下没问题，但在 Agent 系统中就会遇到大麻烦。

假设你让 LLM 分析一段用户评论的情感。它可能返回：

```
这段评论的情感是积极的，用户对产品非常满意。我给的情感分数是 85%，关键词有"满意"、"喜欢"、"推荐"。
```

这段回复看起来不错，但如果你需要程序自动处理这个结果呢？你需要从这段自由文本中提取出情感分数、情感类型、关键词列表……每一个都需要文本匹配或正则表达式，非常脆弱。

而且 LLM 的输出还经常出现这些问题：

- **格式不一致**：有时用"85%"，有时用"0.85"，有时用"积极"
- **多余内容**：除了结构化数据外还夹杂解释文字
- **格式错误**：JSON 缺少逗号、括号不匹配
- **截断**：输出到一半被 Token 限制截断了

### 19.1.2 结构化输出的优势

如果 LLM 能直接输出结构化数据，一切就简单了：

```json
{
    "sentiment": "positive",
    "confidence": 0.85,
    "keywords": ["满意", "喜欢", "推荐"]
}
```

程序可以直接解析这个 JSON，获取每个字段的值，不需要任何文本匹配。而且 Pydantic 等工具可以自动验证数据类型和格式，确保输出是合法的。

### 19.1.3 结构化输出的 Prompt 设计

要让 LLM 输出结构化数据，关键在于 Prompt 的设计。你需要明确告诉 LLM：输出什么格式、包含哪些字段、每个字段的类型是什么。

```python
def create_structured_prompt(task: str, schema: dict,
                              examples: list[dict] = None) -> str:
    """
    创建结构化输出的 Prompt
    
    好的结构化 Prompt 应该包含：
    1. 明确的任务描述
    2. 输出格式的 JSON Schema
    3. 至少一个示例
    4. 明确的格式要求
    """
    schema_text = json.dumps(schema, ensure_ascii=False, indent=2)
    
    prompt = f"""请完成以下任务，并严格按照指定的 JSON 格式输出结果。

任务：{task}

输出格式（必须严格遵循）：
{schema_text}
"""
    
    if examples:
        prompt += "\n示例：\n"
        for i, example in enumerate(examples):
            prompt += f"输入: {example.get('input', '')}\n"
            prompt += f"输出: {json.dumps(example.get('output', {}), ensure_ascii=False)}\n\n"
    
    prompt += """重要要求：
1. 只输出 JSON，不要添加任何解释文字
2. 确保 JSON 格式正确（括号匹配、逗号正确）
3. 字段类型必须符合 Schema 要求
4. 不要添加 Schema 中未定义的字段

输出："""
    
    return prompt
```

---

## 19.2 JSON 解析

### 19.2.1 从 LLM 输出中提取 JSON

LLM 的输出往往不是"纯净"的 JSON——它可能在 JSON 前后加了解释文字，或者被 markdown 的代码块包裹。我们需要一个健壮的解析器来处理这些情况。

```python
import json
import re


class RobustJSONParser:
    """
    健壮的 JSON 解析器
    
    处理 LLM 输出中的各种情况：
    1. 纯 JSON（直接解析）
    2. 被 ```json ``` 包裹的 JSON
    3. JSON 前后有解释文字
    4. JSON 格式有小错误（多余逗号等）
    """
    
    def parse(self, text: str) -> dict | tuple[None, str]:
        """
        从文本中提取 JSON
        
        返回: (解析结果, 错误信息)
        """
        if not text or not text.strip():
            return None, "输入为空"
        
        text = text.strip()
        
        # 策略 1：直接解析
        result = self._try_parse(text)
        if result is not None:
            return result, None
        
        # 策略 2：从 markdown 代码块中提取
        result = self._extract_from_code_block(text)
        if result is not None:
            return result, None
        
        # 策略 3：找到 { } 或 [ ] 包围的内容
        result = self._extract_by_braces(text)
        if result is not None:
            return result, None
        
        # 策略 4：修复常见错误后重试
        result = self._fix_and_parse(text)
        if result is not None:
            return result, None
        
        return None, f"无法从文本中提取 JSON: {text[:200]}..."
    
    def _try_parse(self, text: str):
        """尝试直接解析"""
        try:
            return json.loads(text)
        except json.JSONDecodeError:
            return None
    
    def _extract_from_code_block(self, text: str):
        """从 ```json ... ``` 代码块中提取"""
        patterns = [
            r'```json\s*(.*?)\s*```',
            r'```\s*(.*?)\s*```',
        ]
        for pattern in patterns:
            match = re.search(pattern, text, re.DOTALL)
            if match:
                result = self._try_parse(match.group(1).strip())
                if result is not None:
                    return result
        return None
    
    def _extract_by_braces(self, text: str):
        """通过大括号或中括号提取"""
        # 尝试找 { ... }
        start = text.find('{')
        if start >= 0:
            # 从最后一个 } 往前找
            end = text.rfind('}')
            if end > start:
                result = self._try_parse(text[start:end + 1])
                if result is not None:
                    return result
        
        # 尝试找 [ ... ]
        start = text.find('[')
        if start >= 0:
            end = text.rfind(']')
            if end > start:
                result = self._try_parse(text[start:end + 1])
                if result is not None:
                    return result
        
        return None
    
    def _fix_and_parse(self, text: str):
        """修复常见 JSON 错误后重试"""
        # 提取可能的 JSON 部分
        json_candidates = re.findall(r'[\{\[].*?[\}\]]', text, re.DOTALL)
        
        for candidate in json_candidates:
            # 修复常见错误
            fixed = candidate
            
            # 修复 trailing comma（尾部多余逗号）
            fixed = re.sub(r',\s*([}\]])', r'\1', fixed)
            
            # 修复缺少引号的键名（简单情况）
            fixed = re.sub(r'(\{|,)\s*(\w+)\s*:', r'\1"\2":', fixed)
            
            result = self._try_parse(fixed)
            if result is not None:
                return result
        
        return None
    
    def parse_with_retry(self, text: str, llm_client=None,
                          max_retries: int = 2) -> dict:
        """
        带重试的解析
        
        如果解析失败，让 LLM 修复格式后重试
        """
        result, error = self.parse(text)
        if result is not None:
            return result
        
        if llm_client is None:
            return {"error": error}
        
        # 让 LLM 修复
        for attempt in range(max_retries):
            prompt = f"""请修复以下文本中的 JSON 格式错误。

原始文本：
{text}

要求：
1. 只输出修复后的 JSON
2. 不要添加任何解释
3. 确保格式正确

修复后的 JSON："""
            
            text = llm_client.generate(prompt)
            result, _ = self.parse(text)
            if result is not None:
                return result
        
        return {"error": "JSON 解析失败（已重试）", "raw": text[:200]}
```

### 19.2.2 JSON 修复的常见技巧

```python
class JSONFixer:
    """JSON 常见错误修复器"""
    
    @staticmethod
    def fix_trailing_comma(text: str) -> str:
        """修复尾部多余逗号: {"a": 1,} → {"a": 1}"""
        return re.sub(r',\s*([}\]])', r'\1', text)
    
    @staticmethod
    def fix_unquoted_keys(text: str) -> str:
        """修复未加引号的键名: {a: 1} → {"a": 1}"""
        return re.sub(r'(?<=[{,])\s*(\w+)\s*:', r'"\1":', text)
    
    @staticmethod
    def fix_single_to_double_quotes(text: str) -> str:
        """修复单引号为双引号（简单情况）"""
        # 注意：这个方法不处理字符串值中的引号
        return text.replace("'", '"')
    
    @staticmethod
    def fix_comments(text: str) -> str:
        """移除 JSON 中的注释"""
        # 移除单行注释
        text = re.sub(r'//.*?$', '', text, flags=re.MULTILINE)
        # 移除多行注释
        text = re.sub(r'/\*.*?\*/', '', text, flags=re.DOTALL)
        return text
    
    @classmethod
    def fix_all(cls, text: str) -> str:
        """尝试修复所有常见错误"""
        text = cls.fix_comments(text)
        text = cls.fix_single_to_double_quotes(text)
        text = cls.fix_trailing_comma(text)
        text = cls.fix_unquoted_keys(text)
        return text
```

---

## 19.3 Pydantic 结构化输出

### 19.3.1 用 Pydantic 定义输出 Schema

Pydantic 是 Python 中最流行的数据验证库。用 Pydantic 定义输出模型，可以让解析和验证变得非常简单。

```python
from pydantic import BaseModel, Field, field_validator
from typing import Optional


class SentimentResult(BaseModel):
    """情感分析结果"""
    sentiment: str = Field(description="情感类型: positive/negative/neutral")
    confidence: float = Field(ge=0, le=1, description="置信度 0-1")
    keywords: list[str] = Field(default_factory=list, description="关键词列表")
    reasoning: Optional[str] = Field(None, description="推理过程")
    
    @field_validator('sentiment')
    @classmethod
    def validate_sentiment(cls, v):
        if v not in ['positive', 'negative', 'neutral']:
            raise ValueError(f'情感类型必须是 positive/negative/neutral，收到: {v}')
        return v


class TaskPlan(BaseModel):
    """任务计划"""
    goal: str = Field(description="任务目标")
    steps: list[dict] = Field(description="执行步骤列表")
    estimated_time: Optional[str] = Field(None, description="预计耗时")
    risks: list[str] = Field(default_factory=list, description="潜在风险")


class AnalysisResult(BaseModel):
    """分析结果"""
    summary: str = Field(description="分析摘要")
    key_points: list[str] = Field(description="关键发现")
    score: float = Field(ge=0, le=10, description="评分 0-10")
    recommendations: list[str] = Field(default_factory=list, description="建议")
```

### 19.3.2 Pydantic 解析器

```python
from pydantic import ValidationError


class PydanticOutputParser:
    """
    基于 Pydantic 的输出解析器
    
    工作流程：
    1. 将 Pydantic 模型的 Schema 注入 Prompt
    2. 让 LLM 按 Schema 生成 JSON
    3. 用 Pydantic 验证和解析 JSON
    4. 如果验证失败，自动重试
    """
    
    def __init__(self, model_class: type[BaseModel], llm_client):
        """
        参数:
            model_class: Pydantic 模型类
            llm_client: LLM 客户端
        """
        self.model_class = model_class
        self.llm_client = llm_client
        self.schema = model_class.model_json_schema()
    
    def parse(self, text: str) -> dict:
        """
        从文本中解析出结构化数据
        
        返回: 验证通过的数据字典，或包含 error 的字典
        """
        # 构造带 Schema 的 Prompt
        schema_text = json.dumps(self.schema, ensure_ascii=False, indent=2)
        
        prompt = f"""请根据以下文本生成 JSON 格式的输出。

文本：{text}

要求的 JSON Schema：
{schema_text}

只输出 JSON，不要其他内容："""
        
        response = self.llm_client.generate(prompt)
        
        # 解析 JSON
        try:
            data = json.loads(response)
        except json.JSONDecodeError as e:
            return {"error": f"JSON 解析失败: {e}", "raw": response[:200]}
        
        # Pydantic 验证
        try:
            validated = self.model_class(**data)
            return validated.model_dump()
        except ValidationError as e:
            return {"error": f"数据验证失败: {e}", "raw": data}
    
    def parse_with_retry(self, text: str, max_retries: int = 3) -> dict:
        """带重试的解析"""
        last_error = None
        
        for attempt in range(max_retries):
            result = self.parse(text)
            
            if "error" not in result:
                return result
            
            last_error = result.get("error", "未知错误")
            
            # 让 LLM 修复
            if attempt < max_retries - 1:
                prompt = f"""之前的输出验证失败，请修复。

原始文本：{text}
错误信息：{last_error}
期望的 Schema：{json.dumps(self.schema, ensure_ascii=False)}

请重新生成正确的 JSON 输出："""
                text = self.llm_client.generate(prompt)
        
        return {"error": f"解析失败（已重试 {max_retries} 次）: {last_error}"}
```

### 19.3.3 直接使用 OpenAI 的 Structured Outputs

OpenAI 提供了 Structured Outputs 功能，可以在 API 层面强制 LLM 输出符合 Schema 的 JSON，大大减少了解析和验证的需要。

```python
import os
import openai


class OpenAIStructuredOutput:
    """
    使用 OpenAI Structured Outputs
    
    优势：API 层面保证输出格式，不需要后处理
    限制：只支持 OpenAI 的模型
    """
    
    def __init__(self, model: str = "gpt-4o"):
        self.client = openai.OpenAI(api_key=os.environ.get("OPENAI_API_KEY"))
        self.model = model
    
    def generate_structured(self, prompt: str,
                             response_model: type[BaseModel]) -> dict:
        """
        生成结构化输出
        
        使用 Pydantic 模型作为 JSON Schema，
        OpenAI API 会保证输出符合这个 Schema。
        """
        schema = response_model.model_json_schema()
        
        response = self.client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": "你是一个数据分析助手，请始终以 JSON 格式回复。"},
                {"role": "user", "content": prompt},
            ],
            response_format={
                "type": "json_schema",
                "json_schema": {
                    "name": response_model.__name__,
                    "strict": True,
                    "schema": schema,
                },
            },
            temperature=0.7,
        )
        
        content = response.choices[0].message.content
        data = json.loads(content)
        
        # Pydantic 验证
        validated = response_model(**data)
        return validated.model_dump()
```

---

## 19.4 XML 解析

### 19.4.1 什么时候用 XML

虽然 JSON 是最常用的结构化输出格式，但在某些场景下 XML 更合适：

- 输出包含大量嵌套结构（XML 的层级表示更自然）
- 需要同时包含文本内容和结构化数据
- 与已有的 XML 系统集成

```python
import xml.etree.ElementTree as ET
import re


class XMLOutputParser:
    """
    XML 输出解析器
    
    从 LLM 输出中提取 XML 并解析为字典
    """
    
    def parse(self, text: str, root_tag: str = "response") -> dict:
        """解析 XML"""
        try:
            # 尝试提取 XML 块
            xml_match = re.search(
                f'<{root_tag}>.*?</{root_tag}>',
                text, re.DOTALL
            )
            xml_str = xml_match.group() if xml_match else text
            
            root = ET.fromstring(xml_str)
            return self._element_to_dict(root)
        except ET.ParseError as e:
            return {"error": f"XML 解析失败: {e}"}
    
    def _element_to_dict(self, element: ET.Element) -> dict:
        """将 XML 元素转换为字典"""
        result = {}
        
        # 添加属性
        if element.attrib:
            result["@attributes"] = dict(element.attrib)
        
        # 添加文本内容
        if element.text and element.text.strip():
            if len(element) == 0:  # 没有子元素
                return element.text.strip()
            result["text"] = element.text.strip()
        
        # 添加子元素
        children = list(element)
        if children:
            for child in children:
                child_dict = self._element_to_dict(child)
                child_tag = child.tag
                
                if child_tag in result:
                    if not isinstance(result[child_tag], list):
                        result[child_tag] = [result[child_tag]]
                    result[child_tag].append(child_dict)
                else:
                    result[child_tag] = child_dict
        
        return result


# 使用示例
def demo_xml_parsing():
    """演示 XML 解析"""
    
    llm_output = """分析结果如下：

<response>
    <sentiment>positive</sentiment>
    <confidence>0.92</confidence>
    <keywords>
        <keyword>满意</keyword>
        <keyword>推荐</keyword>
        <keyword>优质</keyword>
    </keywords>
    <summary>用户对产品非常满意，给出了积极评价。</summary>
</response>

以上是分析结果。"""
    
    parser = XMLOutputParser()
    result = parser.parse(llm_output)
    print(json.dumps(result, ensure_ascii=False, indent=2))


if __name__ == "__main__":
    demo_xml_parsing()
```

---

## 19.5 常见输出问题处理

### 19.5.1 输出截断

```python
class TruncationHandler:
    """处理 LLM 输出被截断的情况"""
    
    @staticmethod
    def detect_truncation(text: str) -> bool:
        """检测输出是否被截断"""
        # 检查括号是否匹配
        if text.count('{') != text.count('}'):
            return True
        if text.count('[') != text.count(']'):
            return True
        # 检查是否在引号中间截断
        if text.count('"') % 2 != 0:
            return True
        return False
    
    @staticmethod
    def fix_truncated_json(text: str) -> str:
        """尝试修复截断的 JSON"""
        # 补全缺失的括号
        open_braces = text.count('{') - text.count('}')
        open_brackets = text.count('[') - text.count(']')
        unclosed_quotes = text.count('"') % 2
        
        fixed = text
        
        # 关闭未闭合的字符串
        if unclosed_quotes:
            fixed += '"'
        
        # 关闭未闭合的数组
        if open_brackets > 0:
            # 先关闭当前可能在写的字符串和对象
            if open_braces > 0:
                fixed += '}'
                open_braces -= 1
            fixed += ']' * open_brackets
        
        # 关闭未闭合的对象
        if open_braces > 0:
            fixed += '}' * open_braces
        
        return fixed
```

### 19.5.2 类型不匹配

```python
class TypeCoercion:
    """类型自动转换"""
    
    @staticmethod
    def coerce(value, expected_type: str):
        """将值转换为期望的类型"""
        try:
            if expected_type == "integer":
                return int(float(str(value).strip('%').strip()))
            elif expected_type == "number" or expected_type == "float":
                return float(str(value).strip('%').strip())
            elif expected_type == "boolean":
                if isinstance(value, str):
                    return value.lower() in ('true', 'yes', '1', '是')
                return bool(value)
            elif expected_type == "string":
                return str(value)
            elif expected_type == "array":
                if isinstance(value, list):
                    return value
                return [value]
        except (ValueError, TypeError):
            return value  # 转换失败，返回原始值
```

### 19.5.3 多格式自动识别

```python
class AutoParser:
    """自动识别并解析多种格式"""
    
    def __init__(self):
        self.json_parser = RobustJSONParser()
        self.xml_parser = XMLOutputParser()
    
    def parse(self, text: str) -> dict:
        """自动检测格式并解析"""
        text = text.strip()
        
        # 检测 JSON
        if text.startswith('{') or text.startswith('['):
            result, error = self.json_parser.parse(text)
            if result is not None:
                return result
        
        # 检测 XML
        if text.startswith('<'):
            return self.xml_parser.parse(text)
        
        # 检测 markdown 代码块中的 JSON
        json_match = re.search(r'```(?:json)?\s*(.*?)\s*```', text, re.DOTALL)
        if json_match:
            result, _ = self.json_parser.parse(json_match.group(1))
            if result is not None:
                return result
        
        return {"error": "无法识别输出格式", "raw": text[:200]}
```

---

## 19.6 常见坑

### 19.6.1 JSON 格式错误

**问题描述：** LLM 输出的 JSON 包含语法错误：多余逗号（`{"a": 1,}`）、缺少逗号（`{"a": 1"b": 2}`）、使用了单引号而非双引号。

**解决方案：** 使用 RobustJSONParser 进行多策略解析。在 Prompt 中明确要求"只输出 JSON，不要添加任何其他内容"，并提供示例。

### 19.6.2 类型不匹配

**问题描述：** 期望数字但收到字符串（"85%" 而不是 0.85），期望数组但收到对象。

**解决方案：** 使用 Pydantic 进行类型验证和自动转换。在 Schema 中添加 field_validator 来处理边界情况。

### 19.6.3 输出不完整

**问题描述：** 由于 Token 限制，JSON 输出被截断，导致格式不完整。

**解决方案：** 检测截断并尝试修复（补全括号）。如果修复失败，减少输出要求（比如减少字段数量）后重试。设置合理的 max_tokens 参数。

### 19.6.4 LLM 不遵守格式要求

**问题描述：** 即使在 Prompt 中明确要求输出 JSON，LLM 有时仍然会加上解释文字或输出其他格式。

**解决方案：** 在 Prompt 中加入更强烈的格式约束，如"只输出 JSON，不要有任何其他文字"。使用 few-shot 示例展示期望的输出格式。如果条件允许，使用 OpenAI 的 Structured Outputs 功能。

---

## 19.7 练习题

### 练习 1：实现健壮的 JSON 解析器

实现一个 RobustJSONParser，要求：
- 处理常见的格式错误（多余逗号、未闭合括号、单引号等）
- 从 markdown 代码块中提取 JSON
- 从混合文本中提取 JSON
- 支持重试机制（让 LLM 修复后重试）

### 练习 2：Pydantic 集成

使用 Pydantic 定义一个复杂的数据结构（如"电影评论分析结果"），实现：
- 数据验证（字段类型、范围约束）
- 类型自动转换（字符串转数字等）
- 自定义验证器
- 错误信息友好的报告

### 练习 3：多格式解析器

实现一个能自动识别和解析多种格式（JSON、XML、YAML）的解析器，要求：
- 自动检测输入格式
- 使用对应的解析器解析
- 解析失败时返回清晰的错误信息

### 练习 4：结构化输出 Prompt 设计

设计一个 Prompt，让 LLM 输出包含以下字段的结构化数据：
- 标题（字符串）
- 摘要（字符串，不超过 100 字）
- 关键词（字符串数组，3-5 个）
- 情感（positive/negative/neutral）
- 置信度（0-1 的浮点数）

测试不同 LLM 的输出质量，分析哪个模型最可靠。

### 练习 5：输出解析的端到端测试

构建一个完整的管道：
1. 定义 Pydantic Schema
2. 生成结构化 Prompt
3. 调用 LLM 获取输出
4. 解析和验证输出
5. 处理失败情况（重试、降级）

对至少 10 个不同的输入进行测试，统计解析成功率。

---

## 19.8 实战任务

### 任务：构建生产级输出解析系统

**目标：** 构建一个生产级的输出解析系统，支持多种格式、健壮的错误处理和完整的监控。

**要求：**

1. 支持 JSON、XML 两种主要格式
2. 实现健壮的错误处理和重试机制
3. 使用 Pydantic 进行数据验证
4. 提供解析成功率、平均耗时等监控指标
5. 支持自定义 Schema 注册

---

## 19.9 本章小结

- **结构化输出**是 Agent 与外部系统交互的基础。LLM 的自由文本输出虽然灵活，但不方便程序处理。通过要求 LLM 输出 JSON/XML 等结构化格式，可以让 Agent 的输出变得可预测、可验证、可自动化处理。

- **JSON 是最常用的结构化输出格式**。从 LLM 输出中提取 JSON 需要健壮的解析器，因为 LLM 经常在 JSON 前后添加解释文字、使用 markdown 代码块、或者产生格式错误。RobustJSONParser 通过多策略解析（直接解析 → 代码块提取 → 括号提取 → 错误修复）来应对这些情况。

- **Pydantic 提供了强大的数据验证和类型转换能力**。定义 Pydantic 模型作为输出 Schema，可以自动验证 LLM 输出的类型、范围、格式是否正确。Pydantic 还支持自定义验证器，可以处理复杂的业务规则。

- **XML 适合复杂层级数据**，但解析比 JSON 复杂。在大多数场景下推荐使用 JSON，只有在需要与 XML 系统集成或输出结构特别复杂时才考虑 XML。

- **Prompt 设计对输出质量有重要影响**。好的结构化 Prompt 应该包含：明确的任务描述、JSON Schema、至少一个示例、格式约束。使用 few-shot 示例可以显著提升输出质量。

- **常见问题的处理策略**：格式错误用多策略解析，截断用括号补全，类型不匹配用 Pydantic 自动转换，LLM 不遵守格式用更严格的 Prompt 或 Structured Outputs API。

