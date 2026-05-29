# 第53章：Agent 与代码 —— 代码生成、审查与执行

## 学习目标

通过本章的学习，你将能够：
1. 理解 Agent 代码生成的原理和最佳实践
2. 掌握代码审查 Agent 的实现方法
3. 学会构建安全的代码执行环境
4. 理解代码测试和调试的自动化
5. 掌握代码重构和优化的 Agent 实现
6. 能够构建完整的代码助手系统

## 核心问题

代码是程序员与计算机沟通的语言，但编写高质量的代码需要大量的知识和经验。对于很多开发者来说，代码生成、审查和调试是最耗时的工作。

Agent 与代码的结合可以让这个过程更加高效。一个代码 Agent 可以：
1. 根据需求自动生成代码
2. 审查代码质量并提供改进建议
3. 在安全环境中执行代码并测试
4. 自动调试和修复代码问题
5. 重构和优化现有代码

这就像是给每个开发者配了一个 AI 结对编程伙伴。

---

## 原理讲解

### 代码生成的核心挑战

代码生成面临几个核心挑战：

**1. 需求理解**
用户说"写一个排序函数"，具体是什么排序？稳定性要求？性能要求？这需要深入理解需求。

**2. 代码正确性**
生成的代码必须是可执行的、正确的。这需要理解编程语言的语法和语义。

**3. 代码质量**
代码不仅要能工作，还要可读、可维护、高效。这需要遵循编码规范和最佳实践。

**4. 上下文理解**
代码不是孤立的，它需要与现有代码库集成。这需要理解项目的架构和风格。

### 代码 Agent 的能力

一个完整的代码 Agent 应该具备以下能力：

**代码生成**：根据自然语言描述生成代码。
**代码理解**：理解现有代码的功能和结构。
**代码审查**：检查代码质量，发现潜在问题。
**代码测试**：编写和运行测试用例。
**代码调试**：定位和修复代码错误。
**代码重构**：改进代码结构和性能。

### 代码执行的安全考虑

在 Agent 中执行代码是一个敏感操作，需要特别注意安全：

**沙箱环境**：在隔离的环境中执行代码，防止对系统造成影响。

**资源限制**：限制 CPU、内存和执行时间，防止资源耗尽。

**权限控制**：限制代码可以访问的系统资源。

**代码审查**：在执行前审查代码，排除明显的危险操作。

### 代码质量评估

评估生成代码的质量需要考虑多个维度：

**正确性**：代码是否正确实现了需求。
**可读性**：代码是否易于理解和维护。
**效率**：代码的性能是否合理。
**健壮性**：代码是否能处理各种边界情况。
**安全性**：代码是否存在安全漏洞。

---

## 完整代码示例

### 示例 1：代码生成 Agent

```python
"""
代码生成 Agent
根据自然语言描述生成代码
"""

import json
import os
import ast
import traceback
from typing import Dict, List, Optional, Tuple
from dataclasses import dataclass, field
import anthropic


@dataclass
class CodeResult:
    """代码生成结果"""
    code: str
    language: str
    explanation: str
    test_cases: List[Dict]
    dependencies: List[str]


class CodeGeneratorAgent:
    """代码生成 Agent"""
    
    def __init__(self):
        self.client = anthropic.Anthropic(
            api_key=os.environ.get("ANTHROPIC_API_KEY")
        )
        self.conversation_history: List[Dict] = []
    
    def generate(self, description: str, language: str = "python",
                context: Dict = None) -> CodeResult:
        """生成代码"""
        print(f"\n{'='*60}")
        print(f"生成代码: {description}")
        print(f"语言: {language}")
        print(f"{'='*60}")
        
        # Step 1: 分析需求
        print("\n[Step 1] 分析需求...")
        requirements = self._analyze_requirements(description, context)
        print(f"  功能: {requirements.get('functionality', '')}")
        print(f"  参数: {requirements.get('parameters', [])}")
        print(f"  返回值: {requirements.get('return_type', '')}")
        
        # Step 2: 生成代码
        print("\n[Step 2] 生成代码...")
        code = self._generate_code(description, requirements, language)
        print(f"  代码长度: {len(code)} 字符")
        
        # Step 3: 验证代码
        print("\n[Step 3] 验证代码...")
        is_valid = self._validate_code(code, language)
        print(f"  验证结果: {'通过' if is_valid else '失败'}")
        
        # Step 4: 生成测试用例
        print("\n[Step 4] 生成测试用例...")
        test_cases = self._generate_test_cases(code, requirements, language)
        print(f"  生成 {len(test_cases)} 个测试用例")
        
        # Step 5: 生成说明
        print("\n[Step 5] 生成说明...")
        explanation = self._generate_explanation(code, requirements, language)
        
        # 提取依赖
        dependencies = self._extract_dependencies(code, language)
        
        return CodeResult(
            code=code,
            language=language,
            explanation=explanation,
            test_cases=test_cases,
            dependencies=dependencies,
        )
    
    def _analyze_requirements(self, description: str, context: Dict = None) -> Dict:
        """分析需求"""
        prompt = f"""分析以下代码需求。

描述：{description}
上下文：{json.dumps(context or {}, ensure_ascii=False)}

请用 JSON 格式回答：
{{
    "functionality": "功能描述",
    "parameters": ["参数列表"],
    "return_type": "返回类型",
    "edge_cases": ["边界情况"],
    "complexity": "simple/medium/complex"
}}"""
        
        response = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
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
        
        return {"functionality": description, "parameters": [], "return_type": "any"}
    
    def _generate_code(self, description: str, requirements: Dict, 
                      language: str) -> str:
        """生成代码"""
        prompt = f"""用 {language} 编写以下功能的代码。

功能描述：{description}
需求：{json.dumps(requirements, ensure_ascii=False)}

要求：
1. 代码要简洁、可读
2. 包含必要的注释
3. 处理边界情况
4. 遵循语言的最佳实践

只返回代码，不要包含其他内容。"""
        
        response = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=2048,
            messages=[{"role": "user", "content": prompt}],
        )
        
        code = response.content[0].text.strip()
        
        # 清理代码块标记
        if code.startswith("```"):
            lines = code.split("\n")
            code = "\n".join(lines[1:-1] if lines[-1].strip() == "```" else lines[1:])
        
        return code
    
    def _validate_code(self, code: str, language: str) -> bool:
        """验证代码"""
        if language.lower() == "python":
            try:
                ast.parse(code)
                return True
            except SyntaxError as e:
                print(f"  语法错误: {e}")
                return False
        return True
    
    def _generate_test_cases(self, code: str, requirements: Dict,
                            language: str) -> List[Dict]:
        """生成测试用例"""
        prompt = f"""为以下代码生成测试用例。

代码：
```{language}
{code}
```

需求：{json.dumps(requirements, ensure_ascii=False)}

请生成至少 3 个测试用例，包括正常情况和边界情况。

用 JSON 数组格式回答：
[
    {{
        "name": "测试名称",
        "input": "输入",
        "expected": "预期输出",
        "description": "测试说明"
    }}
]"""
        
        response = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            messages=[{"role": "user", "content": prompt}],
        )
        
        try:
            text = response.content[0].text
            start = text.find('[')
            end = text.rfind(']') + 1
            if start >= 0 and end > start:
                return json.loads(text[start:end])
        except:
            pass
        
        return [{"name": "基础测试", "input": "", "expected": "", "description": ""}]
    
    def _generate_explanation(self, code: str, requirements: Dict,
                            language: str) -> str:
        """生成代码说明"""
        prompt = f"""为以下代码生成说明文档。

代码：
```{language}
{code}
```

请提供：
1. 功能概述
2. 主要函数/类的说明
3. 使用示例
4. 注意事项"""
        
        response = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            messages=[{"role": "user", "content": prompt}],
        )
        
        return response.content[0].text
    
    def _extract_dependencies(self, code: str, language: str) -> List[str]:
        """提取依赖"""
        dependencies = []
        
        if language.lower() == "python":
            try:
                tree = ast.parse(code)
                for node in ast.walk(tree):
                    if isinstance(node, ast.Import):
                        for alias in node.names:
                            dependencies.append(alias.name)
                    elif isinstance(node, ast.ImportFrom):
                        dependencies.append(node.module or "")
            except:
                pass
        
        return list(set(dependencies))


def demo_code_generation():
    """演示代码生成"""
    print("=" * 60)
    print("代码生成 Agent 演示")
    print("=" * 60)
    
    agent = CodeGeneratorAgent()
    
    # 测试生成
    result = agent.generate(
        description="写一个函数，计算斐波那契数列的第 n 项",
        language="python",
    )
    
    print(f"\n生成的代码:\n{result.code}")
    print(f"\n说明:\n{result.explanation[:300]}...")
    print(f"\n依赖: {result.dependencies}")
    print(f"\n测试用例: {len(result.test_cases)} 个")


if __name__ == "__main__":
    demo_code_generation()
```

### 示例 2：代码审查 Agent

```python
"""
代码审查 Agent
自动审查代码质量并提供改进建议
"""

import json
import os
import ast
from typing import Dict, List, Optional
from dataclasses import dataclass, field
import anthropic


@dataclass
class ReviewIssue:
    """审查问题"""
    severity: str  # critical, major, minor, info
    category: str
    line: Optional[int]
    message: str
    suggestion: str


@dataclass
class CodeReview:
    """代码审查结果"""
    overall_score: float  # 0-100
    issues: List[ReviewIssue]
    summary: str
    recommendations: List[str]


class CodeReviewAgent:
    """代码审查 Agent"""
    
    def __init__(self):
        self.client = anthropic.Anthropic(
            api_key=os.environ.get("ANTHROPIC_API_KEY")
        )
    
    def review(self, code: str, language: str = "python",
              context: Dict = None) -> CodeReview:
        """审查代码"""
        print(f"\n{'='*60}")
        print(f"代码审查")
        print(f"{'='*60}")
        
        # Step 1: 静态分析
        print("\n[Step 1] 静态分析...")
        static_issues = self._static_analysis(code, language)
        print(f"  发现 {len(static_issues)} 个问题")
        
        # Step 2: LLM 审查
        print("\n[Step 2] 深度审查...")
        llm_review = self._llm_review(code, language, context)
        
        # Step 3: 合并结果
        all_issues = static_issues + llm_review.get("issues", [])
        
        # Step 4: 计算评分
        score = self._calculate_score(all_issues)
        print(f"  评分: {score}/100")
        
        return CodeReview(
            overall_score=score,
            issues=all_issues,
            summary=llm_review.get("summary", ""),
            recommendations=llm_review.get("recommendations", []),
        )
    
    def _static_analysis(self, code: str, language: str) -> List[ReviewIssue]:
        """静态分析"""
        issues = []
        
        if language.lower() == "python":
            # 检查语法
            try:
                ast.parse(code)
            except SyntaxError as e:
                issues.append(ReviewIssue(
                    severity="critical",
                    category="syntax",
                    line=e.lineno,
                    message=f"语法错误: {e.msg}",
                    suggestion="修复语法错误",
                ))
            
            # 检查代码风格
            lines = code.split("\n")
            for i, line in enumerate(lines, 1):
                # 行长度检查
                if len(line) > 120:
                    issues.append(ReviewIssue(
                        severity="minor",
                        category="style",
                        line=i,
                        message="行长度超过 120 字符",
                        suggestion="将长行拆分为多行",
                    ))
                
                # 缩进检查
                if line and not line.startswith("#"):
                    stripped = line.lstrip()
                    if stripped and len(line) - len(stripped) % 4 != 0:
                        issues.append(ReviewIssue(
                            severity="minor",
                            category="style",
                            line=i,
                            message="缩进不是 4 的倍数",
                            suggestion="使用 4 空格缩进",
                        ))
        
        return issues
    
    def _llm_review(self, code: str, language: str, 
                    context: Dict = None) -> Dict:
        """使用 LLM 进行深度审查"""
        prompt = f"""审查以下 {language} 代码。

代码：
```{language}
{code}
```

上下文：{json.dumps(context or {}, ensure_ascii=False)}

请从以下方面审查：
1. 正确性：代码是否正确实现了预期功能
2. 安全性：是否存在安全漏洞
3. 性能：是否有性能问题
4. 可读性：代码是否易于理解
5. 可维护性：代码是否易于维护和扩展

用 JSON 格式回答：
{{
    "issues": [
        {{
            "severity": "critical/major/minor/info",
            "category": "类别",
            "line": 行号或null,
            "message": "问题描述",
            "suggestion": "改进建议"
        }}
    ],
    "summary": "总体评价",
    "recommendations": ["改进建议列表"]
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
                result = json.loads(text[start:end])
                
                # 转换为 ReviewIssue 对象
                issues = []
                for issue_data in result.get("issues", []):
                    issues.append(ReviewIssue(
                        severity=issue_data.get("severity", "info"),
                        category=issue_data.get("category", ""),
                        line=issue_data.get("line"),
                        message=issue_data.get("message", ""),
                        suggestion=issue_data.get("suggestion", ""),
                    ))
                
                result["issues"] = issues
                return result
        except:
            pass
        
        return {"issues": [], "summary": "审查完成", "recommendations": []}
    
    def _calculate_score(self, issues: List[ReviewIssue]) -> float:
        """计算评分"""
        score = 100.0
        
        for issue in issues:
            if issue.severity == "critical":
                score -= 20
            elif issue.severity == "major":
                score -= 10
            elif issue.severity == "minor":
                score -= 3
            elif issue.severity == "info":
                score -= 1
        
        return max(0, score)


def demo_code_review():
    """演示代码审查"""
    print("=" * 60)
    print("代码审查 Agent 演示")
    print("=" * 60)
    
    agent = CodeReviewAgent()
    
    # 示例代码
    code = """
def calculate_average(numbers):
    total = 0
    for n in numbers:
        total += n
    return total / len(numbers)

def find_max(numbers):
    max_val = numbers[0]
    for n in numbers:
        if n > max_val:
            max_val = n
    return max_val

# 没有处理空列表的情况
result1 = calculate_average([])
result2 = find_max([])
"""
    
    review = agent.review(code)
    
    print(f"\n评分: {review.overall_score}/100")
    print(f"\n总结:\n{review.summary}")
    print(f"\n发现的问题:")
    for issue in review.issues:
        print(f"  [{issue.severity}] {issue.message}")
        if issue.suggestion:
            print(f"    建议: {issue.suggestion}")
    print(f"\n改进建议:")
    for rec in review.recommendations:
        print(f"  - {rec}")


if __name__ == "__main__":
    demo_code_review()
```

### 示例 3：代码执行沙箱

```python
"""
安全的代码执行沙箱
在隔离环境中执行代码
"""

import os
import sys
import json
import tempfile
import subprocess
import resource
from typing import Dict, List, Optional, Tuple
from dataclasses import dataclass
from contextlib import contextmanager
import signal


@dataclass
class ExecutionResult:
    """执行结果"""
    success: bool
    output: str
    error: str
    exit_code: int
    execution_time_ms: float
    memory_used_kb: float


class CodeSandbox:
    """代码执行沙箱"""
    
    def __init__(self, timeout_seconds: int = 10, 
                 max_memory_mb: int = 100):
        self.timeout_seconds = timeout_seconds
        self.max_memory_mb = max_memory_mb
    
    def execute_python(self, code: str, 
                      input_data: str = "") -> ExecutionResult:
        """执行 Python 代码"""
        # 创建临时文件
        with tempfile.NamedTemporaryFile(
            mode='w', suffix='.py', delete=False, encoding='utf-8'
        ) as f:
            f.write(code)
            temp_file = f.name
        
        try:
            # 使用子进程执行，设置资源限制
            start_time = os.times()
            
            result = subprocess.run(
                [sys.executable, temp_file],
                capture_output=True,
                text=True,
                timeout=self.timeout_seconds,
                input=input_data,
                env=self._get_sandbox_env(),
            )
            
            end_time = os.times()
            
            # 计算执行时间
            cpu_time = end_time[2] - start_time[2]  # 用户时间
            execution_time_ms = cpu_time * 1000
            
            return ExecutionResult(
                success=result.returncode == 0,
                output=result.stdout,
                error=result.stderr,
                exit_code=result.returncode,
                execution_time_ms=execution_time_ms,
                memory_used_kb=0,  # 简化实现
            )
            
        except subprocess.TimeoutExpired:
            return ExecutionResult(
                success=False,
                output="",
                error=f"执行超时 ({self.timeout_seconds}秒)",
                exit_code=-1,
                execution_time_ms=self.timeout_seconds * 1000,
                memory_used_kb=0,
            )
        except Exception as e:
            return ExecutionResult(
                success=False,
                output="",
                error=str(e),
                exit_code=-1,
                execution_time_ms=0,
                memory_used_kb=0,
            )
        finally:
            # 清理临时文件
            try:
                os.unlink(temp_file)
            except:
                pass
    
    def _get_sandbox_env(self) -> Dict:
        """获取沙箱环境变量"""
        env = os.environ.copy()
        
        # 限制环境变量
        allowed_vars = ["PATH", "PYTHONPATH", "HOME"]
        return {k: v for k, v in env.items() if k in allowed_vars}
    
    def execute_with_test(self, code: str, test_code: str) -> Dict:
        """执行代码并运行测试"""
        # 执行代码
        code_result = self.execute_python(code)
        
        if not code_result.success:
            return {
                "code_success": False,
                "code_error": code_result.error,
                "tests_passed": False,
            }
        
        # 组合代码和测试
        combined_code = f"{code}\n\n{test_code}"
        
        # 执行测试
        test_result = self.execute_python(combined_code)
        
        return {
            "code_success": True,
            "code_output": code_result.output,
            "tests_passed": test_result.success,
            "test_output": test_result.output,
            "test_error": test_result.error if not test_result.success else "",
        }


def demo_sandbox():
    """演示代码沙箱"""
    print("=" * 60)
    print("代码执行沙箱演示")
    print("=" * 60)
    
    sandbox = CodeSandbox(timeout_seconds=5)
    
    # 测试 1: 正常代码
    code1 = """
def add(a, b):
    return a + b

print(add(1, 2))
"""
    print("\n测试 1: 正常代码")
    result = sandbox.execute_python(code1)
    print(f"  成功: {result.success}")
    print(f"  输出: {result.output.strip()}")
    
    # 测试 2: 有错误的代码
    code2 = """
def divide(a, b):
    return a / b

print(divide(1, 0))
"""
    print("\n测试 2: 有错误的代码")
    result = sandbox.execute_python(code2)
    print(f"  成功: {result.success}")
    print(f"  错误: {result.error[:100] if result.error else 'None'}")
    
    # 测试 3: 执行代码和测试
    code3 = """
def factorial(n):
    if n <= 1:
        return 1
    return n * factorial(n - 1)
"""
    test3 = """
assert factorial(0) == 1
assert factorial(1) == 1
assert factorial(5) == 120
print("所有测试通过!")
"""
    print("\n测试 3: 代码 + 测试")
    result = sandbox.execute_with_test(code3, test3)
    print(f"  代码成功: {result['code_success']}")
    print(f"  测试通过: {result['tests_passed']}")
    print(f"  输出: {result.get('test_output', '').strip()}")


if __name__ == "__main__":
    demo_sandbox()
```

---

## 案例分析

### 案例 1：在线代码学习平台

**场景**：构建一个帮助初学者学习编程的平台。

**Agent 功能**：
1. 根据学习目标生成示例代码
2. 解释代码的每个部分
3. 提供练习题和参考答案
4. 审查学生提交的代码
5. 提供个性化的学习建议

**架构设计**：

```
学习目标 → 代码生成 → 代码解释 → 练习题
    ↓                         ↓
学生代码 → 代码审查 → 反馈生成 → 学习建议
```

### 案例 2：企业代码质量保障

**场景**：企业需要确保代码库的质量和一致性。

**Agent 功能**：
1. 自动审查 Pull Request
2. 检查编码规范
3. 发现潜在 Bug
4. 建议重构方案
5. 生成代码文档

---

## 常见坑

### 坑 1：代码安全性

执行用户生成的代码有安全风险。

**解决方案**：使用沙箱环境，限制系统资源访问。

### 坑 2：代码正确性

生成的代码可能有逻辑错误。

**解决方案**：结合测试用例验证，提供代码审查。

### 坑 3：上下文理解

代码生成可能忽略项目上下文。

**解决方案**：分析现有代码库，保持风格一致。

### 坑 4：性能问题

生成的代码可能效率低下。

**解决方案**：进行性能分析，提供优化建议。

### 坑 5：依赖管理

代码可能依赖不存在的库。

**解决方案**：检查依赖可用性，提供替代方案。

---

## 练习题

### 练习 1：代码生成
实现一个代码生成器，支持：
- 函数生成
- 类生成
- 模块生成

### 练习 2：代码审查
实现一个代码审查器，检查：
- 代码风格
- 潜在错误
- 性能问题

### 练习 3：测试生成
实现一个测试生成器，能够：
- 分析代码结构
- 生成单元测试
- 生成边界测试

### 练习 4：代码重构
实现一个重构工具，支持：
- 提取函数
- 重命名变量
- 简化逻辑

### 练习 5：调试助手
实现一个调试助手，能够：
- 分析错误信息
- 定位问题代码
- 提供修复建议

### 练习 6：完整代码助手
构建一个完整的代码助手系统，集成以上所有功能。

---

## 实战任务

### 任务 1：构建在线代码学习助手（中等难度）

**功能需求：**
1. 生成示例代码
2. 解释代码功能
3. 审查学生代码
4. 提供练习题

**技术要求：**
1. 使用代码生成 Agent
2. 实现代码执行沙箱
3. 支持多种编程语言
4. 生成学习报告

### 任务 2：构建代码质量保障系统（高难度）

**功能需求：**
1. 自动代码审查
2. 编码规范检查
3. 安全漏洞扫描
4. 性能分析

**技术要求：**
1. 集成静态分析工具
2. 实现规则引擎
3. 支持自定义规则
4. 生成质量报告

---

## 本章小结

本章我们深入学习了 Agent 与代码的结合，包括代码生成、审查和执行。我们从以下几个方面进行了探索：

**代码生成**：理解了从需求分析到代码生成的完整流程，包括需求理解、代码生成、验证和测试。

**代码审查**：掌握了静态分析和 LLM 审查两种代码审查方法，能够发现代码中的问题并提供改进建议。

**代码执行**：学会了构建安全的代码执行沙箱，在隔离环境中运行代码。

**安全考虑**：了解了代码执行的安全风险和防护措施。

Agent 与代码的结合正在改变软件开发的方式。虽然目前的代码生成还不能完全替代人类开发者，但它已经能够显著提高开发效率，特别是在重复性任务和初学者学习场景中。

在下一章中，我们将学习 Agent 与浏览器的结合，看看如何构建 Web Agent。

---

## 延伸阅读

1. Code Generation with LLMs (研究综述)
2. AlphaCode: Competitive Programming with AI
3. Codex: Evaluating Large Language Models Trained on Code
4. Program Synthesis (研究领域)
5. Automated Code Review (最佳实践)
