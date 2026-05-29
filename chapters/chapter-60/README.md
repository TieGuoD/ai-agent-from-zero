# 第 60 章：项目 4 -- 代码 Agent

> **本章定位：** 代码 Agent 是当前 AI Agent 领域最活跃的应用方向之一。GitHub Copilot、Cursor、Claude Code 等产品已经证明了代码 Agent 的巨大价值。本章将从零构建一个代码 Agent，它能够理解代码库结构、定位 Bug、生成修复方案、编写测试用例，甚至重构代码。这个项目将让你深入理解 Agent 在软件开发场景中的工作原理。

---

## 学习目标

完成本章学习后，你将能够：

1. **构建代码理解系统** -- 能够让 Agent 解析代码库的结构、理解函数调用关系、追踪变量的流动路径
2. **实现自动化 Bug 定位与修复** -- 能够让 Agent 根据错误信息定位代码中的问题，并生成修复方案
3. **构建智能代码生成系统** -- 能够让 Agent 根据需求描述生成符合项目风格的代码，并确保代码能通过测试
4. **实现代码审查功能** -- 能够让 Agent 自动审查代码变更，发现潜在问题并给出改进建议
5. **掌握代码 Agent 的安全边界** -- 理解代码执行的危险性，设计安全的沙箱和权限控制机制
6. **完成一个可实际使用的代码助手** -- 支持代码解释、Bug 修复、测试生成、重构等核心功能

## 核心问题

1. **代码 Agent 和简单的代码补全有什么区别？** 为什么 Copilot 的代码补全已经很强大了，我们还需要代码 Agent？Agent 的额外能力体现在哪些地方？
2. **LLM 理解代码的能力边界在哪里？** 它能理解多复杂的代码逻辑？它最容易在什么地方出错？我们如何弥补这些不足？
3. **代码 Agent 的安全性如何保障？** 让 AI 修改和执行代码，风险有多大？如何设计安全的执行环境？

---

## 60.1 项目背景与需求分析

### 60.1.1 开发者的痛点

软件开发是一个高度依赖知识和经验的工作。一个资深工程师可能需要花 40% 的时间写新代码，30% 的时间阅读和理解已有代码，20% 的时间调试和修复 Bug，10% 的时间写测试和文档。代码 Agent 能够大幅缩短后三项的时间。

一个典型的场景：你在维护一个大型项目，某个模块突然出现了一个 Bug。你需要阅读大量的相关代码才能理解问题所在，然后小心翼翼地修改代码，确保不会引入新的问题。这个过程可能需要几个小时甚至几天。

代码 Agent 可以这样帮你：你把错误信息告诉它，它会在整个代码库中搜索相关的代码，定位最可能出问题的地方，分析 Bug 的根因，然后生成一个修复方案。你只需要审核这个方案，确认它不会引入新问题，然后应用这个修复。

### 60.1.2 功能需求定义

我们确定代码 Agent 需要具备以下核心功能：

首先是**代码库理解功能**。Agent 需要能够解析项目的文件结构、导入关系、类和函数的层次结构。它需要知道项目使用了什么技术栈、有什么设计模式、代码风格是什么样的。

其次是**代码分析功能**。Agent 需要能够阅读指定的代码文件，理解代码的逻辑，追踪变量的流动，分析函数的调用链。对于 Bug 场景，它需要能够根据错误信息反向追踪到问题的根源。

然后是**代码生成功能**。Agent 需要能够根据需求描述生成代码，根据测试失败信息修复代码，根据代码异味进行重构。生成的代码需要符合项目的风格和规范。

最后是**代码执行功能**。Agent 需要能够运行生成的测试用例，验证修复方案是否正确，运行代码格式化和静态分析工具。

### 60.1.3 安全考虑

代码 Agent 的一个特殊之处在于：它操作的是代码，而代码是具有"执行能力"的。一个错误的代码修改可能导致数据丢失，一个恶意的代码注入可能导致安全漏洞。因此，安全设计是代码 Agent 的重中之重。

我们的安全策略包括三个层面。第一，沙箱执行：所有代码执行都在受控的环境中进行，限制文件系统访问和网络访问。第二，版本控制：所有代码修改都通过 Git 操作，支持随时回滚。第三，人工审核：关键操作（如提交代码、运行生产脚本）需要用户确认。

---

## 60.2 项目结构设计

```
code-agent/
    main.py              # 入口文件
    agent.py             # Agent 核心逻辑
    code_analyzer.py     # 代码分析器
    code_executor.py     # 代码执行器（沙箱环境）
    git_manager.py       # Git 操作封装
    file_manager.py      # 文件系统操作
    prompts.py           # 提示词模板
    config.py            # 配置管理
    sandbox/             # 沙箱执行环境
    output/              # 输出目录
```

---

## 60.3 核心代码实现

### 60.3.1 代码分析器

```python
# code_analyzer.py
import os
import ast
import re
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class FunctionInfo:
    """函数信息。"""
    name: str
    file_path: str
    line_start: int
    line_end: int
    docstring: Optional[str] = None
    arguments: list[str] = field(default_factory=list)
    decorators: list[str] = field(default_factory=list)
    is_async: bool = False
    complexity: int = 1


@dataclass
class ClassInfo:
    """类信息。"""
    name: str
    file_path: str
    line_start: int
    line_end: int
    docstring: Optional[str] = None
    bases: list[str] = field(default_factory=list)
    methods: list[FunctionInfo] = field(default_factory=list)
    attributes: list[str] = field(default_factory=list)


@dataclass
class FileAnalysis:
    """文件分析结果。"""
    file_path: str
    language: str
    total_lines: int
    code_lines: int
    comment_lines: int
    blank_lines: int
    imports: list[str] = field(default_factory=list)
    functions: list[FunctionInfo] = field(default_factory=list)
    classes: list[ClassInfo] = field(default_factory=list)
    global_variables: list[str] = field(default_factory=list)


@dataclass
class ProjectAnalysis:
    """项目级分析结果。"""
    root_path: str
    total_files: int = 0
    total_lines: int = 0
    languages: dict = field(default_factory=dict)
    files: list[FileAnalysis] = field(default_factory=list)
    dependency_graph: dict = field(default_factory=dict)
    entry_points: list[str] = field(default_factory=list)


class CodeAnalyzer:
    """代码分析器。

    使用 Python AST（抽象语法树）来解析代码结构。
    AST 分析比正则表达式更准确，能够理解代码的语法含义。
    """

    # 忽略的目录
    IGNORED_DIRS = {
        ".git", "__pycache__", "node_modules", ".venv",
        "venv", ".env", "dist", "build", ".idea", ".vscode",
        "egg-info", ".eggs"
    }

    # 支持的语言
    LANGUAGE_MAP = {
        ".py": "python",
        ".js": "javascript",
        ".ts": "typescript",
        ".jsx": "javascript",
        ".tsx": "typescript",
    }

    def analyze_file(self, file_path: str) -> Optional[FileAnalysis]:
        """分析单个文件。"""
        _, ext = os.path.splitext(file_path)
        language = self.LANGUAGE_MAP.get(ext, "unknown")

        try:
            with open(file_path, "r", encoding="utf-8", errors="replace") as f:
                content = f.read()
        except Exception as e:
            print(f"[读取失败] {file_path}: {e}")
            return None

        lines = content.split("\n")
        total_lines = len(lines)
        blank_lines = sum(1 for line in lines if line.strip() == "")
        comment_lines = sum(1 for line in lines if line.strip().startswith("#"))

        analysis = FileAnalysis(
            file_path=file_path,
            language=language,
            total_lines=total_lines,
            code_lines=total_lines - blank_lines - comment_lines,
            comment_lines=comment_lines,
            blank_lines=blank_lines,
        )

        # Python 文件用 AST 分析
        if language == "python":
            self._analyze_python_file(content, analysis)

        return analysis

    def _analyze_python_file(self, content: str, analysis: FileAnalysis):
        """使用 AST 分析 Python 文件。"""
        try:
            tree = ast.parse(content)
        except SyntaxError as e:
            print(f"[语法错误] {analysis.file_path}: {e}")
            return

        for node in ast.iter_child_nodes(tree):
            if isinstance(node, ast.Import):
                for alias in node.names:
                    analysis.imports.append(alias.name)
            elif isinstance(node, ast.ImportFrom):
                module = node.module or ""
                for alias in node.names:
                    analysis.imports.append(f"{module}.{alias.name}")
            elif isinstance(node, ast.FunctionDef) or isinstance(node, ast.AsyncFunctionDef):
                func_info = FunctionInfo(
                    name=node.name,
                    file_path=analysis.file_path,
                    line_start=node.lineno,
                    line_end=getattr(node, "end_lineno", node.lineno) or node.lineno,
                    docstring=ast.get_docstring(node),
                    arguments=[arg.arg for arg in node.args.args],
                    decorators=[
                        ast.dump(d) for d in node.decorator_list
                    ],
                    is_async=isinstance(node, ast.AsyncFunctionDef),
                )
                analysis.functions.append(func_info)
            elif isinstance(node, ast.ClassDef):
                methods = []
                attributes = []
                for item in node.body:
                    if isinstance(item, (ast.FunctionDef, ast.AsyncFunctionDef)):
                        method = FunctionInfo(
                            name=item.name,
                            file_path=analysis.file_path,
                            line_start=item.lineno,
                            line_end=getattr(item, "end_lineno", item.lineno) or item.lineno,
                            docstring=ast.get_docstring(item),
                            arguments=[arg.arg for arg in item.args.args],
                        )
                        methods.append(method)
                    elif isinstance(item, ast.AnnAssign) and isinstance(item.target, ast.Name):
                        attributes.append(item.target.id)

                class_info = ClassInfo(
                    name=node.name,
                    file_path=analysis.file_path,
                    line_start=node.lineno,
                    line_end=getattr(node, "end_lineno", node.lineno) or node.lineno,
                    docstring=ast.get_docstring(node),
                    bases=[
                        ast.dump(base) for base in node.bases
                    ],
                    methods=methods,
                    attributes=attributes,
                )
                analysis.classes.append(class_info)

    def analyze_project(self, root_path: str,
                        include_patterns: list[str] = None) -> ProjectAnalysis:
        """分析整个项目。"""
        project = ProjectAnalysis(root_path=root_path)

        for dirpath, dirnames, filenames in os.walk(root_path):
            # 跳过忽略的目录
            dirnames[:] = [
                d for d in dirnames
                if d not in self.IGNORED_DIRS
            ]

            for filename in filenames:
                file_path = os.path.join(dirpath, filename)
                _, ext = os.path.splitext(filename)

                # 按扩展名过滤
                if ext not in self.LANGUAGE_MAP:
                    continue

                analysis = self.analyze_file(file_path)
                if analysis:
                    project.files.append(analysis)
                    project.total_files += 1
                    project.total_lines += analysis.total_lines

                    # 统计语言分布
                    lang = analysis.language
                    project.languages[lang] = project.languages.get(lang, 0) + 1

                    # 识别入口点
                    if filename in ("main.py", "app.py", "__main__.py",
                                    "manage.py", "cli.py"):
                        project.entry_points.append(file_path)

        print(f"[项目分析完成] {project.total_files} 个文件, "
              f"{project.total_lines} 行代码, "
              f"语言分布: {project.languages}")

        return project

    def find_function(self, project: ProjectAnalysis,
                      function_name: str) -> list[FunctionInfo]:
        """在项目中查找指定函数。"""
        results = []
        for file_analysis in project.files:
            for func in file_analysis.functions:
                if func.name == function_name:
                    results.append(func)
            for cls in file_analysis.classes:
                for method in cls.methods:
                    if method.name == function_name:
                        results.append(method)
        return results

    def get_file_content(self, file_path: str, start_line: int = None,
                         end_line: int = None) -> str:
        """获取文件内容，支持行范围。"""
        try:
            with open(file_path, "r", encoding="utf-8", errors="replace") as f:
                lines = f.readlines()

            if start_line is not None and end_line is not None:
                start = max(0, start_line - 1)
                end = min(len(lines), end_line)
                return "".join(lines[start:end])

            return "".join(lines)
        except Exception as e:
            return f"[读取失败] {e}"

    def get_context_for_llm(self, project: ProjectAnalysis,
                            max_files: int = 20) -> str:
        """为 LLM 生成项目上下文摘要。"""
        parts = []
        parts.append(f"项目根目录: {project.root_path}")
        parts.append(f"文件总数: {project.total_files}")
        parts.append(f"代码总行数: {project.total_lines}")
        parts.append(f"语言分布: {project.languages}")
        parts.append(f"入口点: {project.entry_points}")

        # 文件列表
        parts.append("\n文件列表:")
        for fa in project.files[:max_files]:
            rel_path = os.path.relpath(fa.file_path, project.root_path)
            parts.append(
                f"  {rel_path} ({fa.language}, {fa.code_lines}行, "
                f"{len(fa.functions)}函数, {len(fa.classes)}类)"
            )

        # 关键函数摘要
        parts.append("\n关键函数:")
        for fa in project.files[:max_files]:
            for func in fa.functions[:5]:
                args_str = ", ".join(func.arguments[:5])
                doc_preview = (func.docstring or "")[:100]
                parts.append(
                    f"  {func.name}({args_str}) - {doc_preview}"
                )

        return "\n".join(parts)
```

### 60.3.2 代码执行器

```python
# code_executor.py
import subprocess
import os
import tempfile
import time
from dataclasses import dataclass
from typing import Optional


@dataclass
class ExecutionResult:
    """代码执行结果。"""
    command: str
    stdout: str
    stderr: str
    return_code: int
    execution_time: float  # 秒
    is_success: bool


class CodeExecutor:
    """代码执行器。

    提供安全的代码执行环境。支持：
    - 运行 Python 脚本
    - 运行测试（pytest, unittest）
    - 运行代码检查（linting）
    - 运行格式化工具

    安全措施：
    - 执行超时控制
    - 限制工作目录
    - 捕获所有输出
    """

    def __init__(self, working_dir: str = ".",
                 timeout: int = 60,
                 python_executable: str = "python"):
        self.working_dir = os.path.abspath(working_dir)
        self.timeout = timeout
        self.python_executable = python_executable
        self.execution_history: list[ExecutionResult] = []

    def run_script(self, script_path: str,
                   args: list[str] = None) -> ExecutionResult:
        """运行 Python 脚本。"""
        cmd = [self.python_executable, script_path]
        if args:
            cmd.extend(args)
        return self._execute(cmd)

    def run_code_snippet(self, code: str) -> ExecutionResult:
        """运行代码片段（写入临时文件后执行）。"""
        with tempfile.NamedTemporaryFile(
            mode="w", suffix=".py",
            delete=False, encoding="utf-8"
        ) as f:
            f.write(code)
            temp_path = f.name

        try:
            result = self.run_script(temp_path)
            return result
        finally:
            os.unlink(temp_path)

    def run_tests(self, test_path: str = "tests/",
                  pattern: str = "test_*.py") -> ExecutionResult:
        """运行 pytest 测试。"""
        cmd = [self.python_executable, "-m", "pytest",
               test_path, f"-k={pattern}", "-v", "--tb=short"]
        return self._execute(cmd)

    def run_lint(self, file_path: str) -> ExecutionResult:
        """运行代码检查。"""
        cmd = [self.python_executable, "-m", "ruff", "check",
               file_path, "--output-format=text"]
        return self._execute(cmd)

    def run_format(self, file_path: str) -> ExecutionResult:
        """运行代码格式化。"""
        cmd = [self.python_executable, "-m", "ruff", "format", file_path]
        return self._execute(cmd)

    def run_type_check(self, file_path: str) -> ExecutionResult:
        """运行类型检查。"""
        cmd = [self.python_executable, "-m", "mypy", file_path, "--ignore-missing-imports"]
        return self._execute(cmd)

    def diff_check(self, file_path: str) -> str:
        """获取文件的 git diff。"""
        try:
            result = subprocess.run(
                ["git", "diff", file_path],
                cwd=self.working_dir,
                capture_output=True,
                text=True,
                timeout=10,
            )
            return result.stdout
        except Exception:
            return ""

    def _execute(self, cmd: list[str]) -> ExecutionResult:
        """执行命令并捕获输出。"""
        start_time = time.time()
        try:
            result = subprocess.run(
                cmd,
                cwd=self.working_dir,
                capture_output=True,
                text=True,
                timeout=self.timeout,
            )
            execution_time = time.time() - start_time

            exec_result = ExecutionResult(
                command=" ".join(cmd),
                stdout=result.stdout,
                stderr=result.stderr,
                return_code=result.returncode,
                execution_time=execution_time,
                is_success=(result.returncode == 0),
            )
        except subprocess.TimeoutExpired:
            execution_time = time.time() - start_time
            exec_result = ExecutionResult(
                command=" ".join(cmd),
                stdout="",
                stderr=f"执行超时（{self.timeout}秒）",
                return_code=-1,
                execution_time=execution_time,
                is_success=False,
            )
        except Exception as e:
            execution_time = time.time() - start_time
            exec_result = ExecutionResult(
                command=" ".join(cmd),
                stdout="",
                stderr=str(e),
                return_code=-1,
                execution_time=execution_time,
                is_success=False,
            )

        self.execution_history.append(exec_result)
        return exec_result
```

### 60.3.3 Git 管理器

```python
# git_manager.py
import subprocess
import os
from dataclasses import dataclass


@dataclass
class GitDiff:
    """Git diff 信息。"""
    file_path: str
    additions: int
    deletions: int
    diff_text: str


class GitManager:
    """Git 操作封装。

    为代码 Agent 提供安全的 Git 操作接口。
    所有破坏性操作（如 reset, force push）都需要明确确认。
    """

    def __init__(self, repo_path: str = "."):
        self.repo_path = os.path.abspath(repo_path)

    def is_git_repo(self) -> bool:
        """检查当前目录是否是 Git 仓库。"""
        try:
            result = self._run(["git", "rev-parse", "--git-dir"])
            return result.returncode == 0
        except Exception:
            return False

    def get_status(self) -> str:
        """获取仓库状态。"""
        result = self._run(["git", "status", "--short"])
        return result.stdout

    def get_diff(self, staged: bool = False) -> str:
        """获取当前 diff。"""
        cmd = ["git", "diff"]
        if staged:
            cmd.append("--staged")
        result = self._run(cmd)
        return result.stdout

    def stage_file(self, file_path: str) -> bool:
        """暂存文件。"""
        result = self._run(["git", "add", file_path])
        return result.returncode == 0

    def commit(self, message: str) -> bool:
        """创建提交。"""
        result = self._run(["git", "commit", "-m", message])
        return result.returncode == 0

    def create_branch(self, branch_name: str) -> bool:
        """创建新分支。"""
        result = self._run(["git", "checkout", "-b", branch_name])
        return result.returncode == 0

    def get_log(self, limit: int = 10) -> str:
        """获取提交日志。"""
        result = self._run([
            "git", "log", f"--oneline", f"-{limit}"
        ])
        return result.stdout

    def stash_changes(self) -> bool:
        """暂存当前修改。"""
        result = self._run(["git", "stash"])
        return result.returncode == 0

    def discard_changes(self, file_path: str) -> bool:
        """丢弃文件的修改（危险操作，需要确认）。"""
        result = self._run(["git", "checkout", "--", file_path])
        return result.returncode == 0

    def _run(self, cmd: list[str]) -> subprocess.CompletedProcess:
        """执行 git 命令。"""
        try:
            return subprocess.run(
                cmd,
                cwd=self.repo_path,
                capture_output=True,
                text=True,
                timeout=30,
            )
        except Exception as e:
            return subprocess.CompletedProcess(
                cmd=cmd, returncode=-1,
                stdout="", stderr=str(e)
            )
```

### 60.3.4 代码 Agent 核心

```python
# agent.py
import json
import os
from openai import OpenAI
from code_analyzer import CodeAnalyzer, ProjectAnalysis
from code_executor import CodeExecutor
from git_manager import GitManager
from prompts import (
    CODE_EXPLAIN_PROMPT, BUG_FIX_PROMPT, CODE_GENERATE_PROMPT,
    CODE_REVIEW_PROMPT, TEST_GENERATE_PROMPT, REFACTOR_PROMPT,
    SYSTEM_PROMPT
)


class CodeAgent:
    """代码 Agent 核心类。

    提供以下核心能力：
    1. 代码库理解和分析
    2. Bug 定位与修复
    3. 代码生成
    4. 代码审查
    5. 测试生成
    6. 代码重构
    """

    def __init__(self, project_path: str = ".",
                 api_key: str = None,
                 model: str = "gpt-4o",
                 base_url: str = None):
        self.project_path = os.path.abspath(project_path)
        self.client = OpenAI(
            api_key=api_key or os.environ.get("OPENAI_API_KEY", ""),
            base_url=base_url or os.environ.get("OPENAI_BASE_URL", None),
        )
        self.model = model
        self.analyzer = CodeAnalyzer()
        self.executor = CodeExecutor(working_dir=self.project_path)
        self.git = GitManager(self.project_path)
        self.project_analysis: ProjectAnalysis = None
        self.messages: list[dict] = []

    def init_project(self):
        """分析项目并初始化 Agent 上下文。"""
        print(f"[初始化] 分析项目: {self.project_path}")
        self.project_analysis = self.analyzer.analyze_project(self.project_path)
        project_context = self.analyzer.get_context_for_llm(self.project_analysis)

        self.messages = [
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "assistant", "content": f"已加载项目信息:\n{project_context}"}
        ]

        print(f"[初始化完成] 项目已加载，开始提供服务。")

    def explain_code(self, file_path: str = None,
                     function_name: str = None,
                     line_range: tuple = None) -> str:
        """解释代码。"""
        if function_name and self.project_analysis:
            functions = self.analyzer.find_function(self.project_analysis, function_name)
            if functions:
                func = functions[0]
                code = self.analyzer.get_file_content(
                    func.file_path, func.line_start, func.line_end
                )
                file_path = func.file_path
            else:
                return f"未找到函数: {function_name}"
        elif file_path:
            code = self.analyzer.get_file_content(file_path)
        else:
            return "请指定文件路径或函数名。"

        prompt = CODE_EXPLAIN_PROMPT.format(code=code, file_path=file_path)
        return self._chat(prompt)

    def fix_bug(self, error_message: str,
                file_path: str = None,
                relevant_code: str = None) -> str:
        """定位并修复 Bug。"""
        # 自动收集上下文
        context = ""
        if file_path:
            code = self.analyzer.get_file_content(file_path)
            context = f"\n出错文件内容：\n```python\n{code}\n```"
        elif self.project_analysis:
            context = f"\n项目结构：\n{self.analyzer.get_context_for_llm(self.project_analysis, max_files=10)}"

        prompt = BUG_FIX_PROMPT.format(
            error_message=error_message,
            context=context,
            relevant_code=relevant_code or "未提供"
        )
        return self._chat(prompt)

    def generate_code(self, description: str,
                      target_file: str = None,
                      existing_code: str = None) -> str:
        """根据描述生成代码。"""
        context = ""
        if target_file and os.path.exists(target_file):
            existing = self.analyzer.get_file_content(target_file)
            context = f"\n现有文件内容：\n```python\n{existing}\n```"
        elif self.project_analysis:
            context = f"\n项目上下文：\n{self.analyzer.get_context_for_llm(self.project_analysis, max_files=10)}"

        prompt = CODE_GENERATE_PROMPT.format(
            description=description,
            context=context,
            existing_code=existing_code or "无"
        )
        return self._chat(prompt)

    def review_code(self, file_path: str = None,
                    diff_text: str = None) -> str:
        """审查代码。"""
        if diff_text is None and file_path:
            diff_text = self.executor.diff_check(file_path)
            if not diff_text:
                code = self.analyzer.get_file_content(file_path)
                diff_text = f"文件完整内容（非 diff 格式）：\n{code}"

        prompt = CODE_REVIEW_PROMPT.format(
            diff=diff_text or "无变更",
            file_path=file_path or "未指定"
        )
        return self._chat(prompt)

    def generate_tests(self, file_path: str) -> str:
        """为指定文件生成测试用例。"""
        code = self.analyzer.get_file_content(file_path)
        prompt = TEST_GENERATE_PROMPT.format(
            file_path=file_path,
            code=code,
            project_context=self.analyzer.get_context_for_llm(
                self.project_analysis, max_files=5
            ) if self.project_analysis else ""
        )
        return self._chat(prompt)

    def refactor(self, file_path: str, instructions: str) -> str:
        """重构代码。"""
        code = self.analyzer.get_file_content(file_path)
        prompt = REFACTOR_PROMPT.format(
            file_path=file_path,
            code=code,
            instructions=instructions
        )
        return self._chat(prompt)

    def apply_fix(self, file_path: str, new_content: str,
                  commit_message: str = None) -> bool:
        """将修复应用到文件并可选提交。"""
        try:
            # 先备份
            backup_path = file_path + ".bak"

            # 通过 Git 暂存当前版本
            if self.git.is_git_repo():
                self.git.stage_file(file_path)

            # 写入新内容
            with open(file_path, "w", encoding="utf-8") as f:
                f.write(new_content)
            print(f"[文件已更新] {file_path}")

            # 运行测试
            print("[运行测试验证...]")
            test_result = self.executor.run_tests()
            if test_result.is_success:
                print("[测试通过]")
            else:
                print(f"[测试失败]\n{test_result.stderr[:500]}")
                return False

            # 提交
            if commit_message and self.git.is_git_repo():
                self.git.stage_file(file_path)
                self.git.commit(commit_message)
                print(f"[已提交] {commit_message}")

            return True

        except Exception as e:
            print(f"[应用失败] {e}")
            return False

    def _chat(self, user_message: str) -> str:
        """与 LLM 对话。"""
        self.messages.append({"role": "user", "content": user_message})

        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=self.messages,
                temperature=0.2,
                max_tokens=4096,
            )
            assistant_message = response.choices[0].message.content
            self.messages.append({"role": "assistant", "content": assistant_message})
            return assistant_message
        except Exception as e:
            return f"[LLM 调用失败] {e}"
```

### 60.3.5 提示词模板

```python
# prompts.py

SYSTEM_PROMPT = """你是一个专业的代码助手 Agent。你能够帮助开发者理解代码、定位 Bug、
生成代码、审查代码和重构代码。

## 工作原则

1. **理解上下文**：在给出建议之前，先理解项目的整体架构和代码风格
2. **最小改动**：修复 Bug 时，只做必要的最小改动，不要过度重构
3. **保持一致性**：生成的代码要符合项目的现有风格（命名、缩进、注释等）
4. **安全优先**：不要建议可能引入安全漏洞的代码
5. **提供解释**：每次修改都要解释为什么这样改，帮助开发者理解

## 代码风格指南

- 遵循 PEP 8（Python）
- 函数和变量使用 snake_case
- 类使用 CamelCase
- 重要函数必须有 docstring
- 避免过度注释，代码本身应该可读
"""

CODE_EXPLAIN_PROMPT = """请详细解释以下代码的功能和逻辑。

文件: {file_path}

```python
{code}
```

请从以下方面解释：
1. 这段代码的整体功能是什么？
2. 主要的函数/类各有什么作用？
3. 代码的执行流程是怎样的？
4. 有没有需要注意的边界条件或潜在问题？
"""

BUG_FIX_PROMPT = """我遇到了一个 Bug，请帮我定位问题并提供修复方案。

错误信息：
```
{error_message}
```
{context}

相关代码：
```python
{relevant_code}
```

请：
1. 分析错误的根本原因
2. 找到最可能出问题的代码位置
3. 提供具体的修复代码
4. 解释修复的原理
5. 建议如何防止类似的 Bug
"""

CODE_GENERATE_PROMPT = """请根据以下需求生成代码。

需求描述：{description}
{context}

现有代码（如果需要补充）：
```python
{existing_code}
```

请：
1. 生成符合项目风格的代码
2. 添加必要的注释
3. 考虑错误处理
4. 确保代码可以正确运行
"""

CODE_REVIEW_PROMPT = """请审查以下代码变更。

文件: {file_path}

变更内容：
```diff
{diff}
```

请从以下方面审查：
1. 逻辑正确性：代码逻辑是否有错误？
2. 安全性：是否存在安全漏洞？
3. 性能：有没有性能问题？
4. 可读性：代码是否易于理解和维护？
5. 测试覆盖：是否需要添加测试用例？
6. 边界条件：是否处理了所有边界情况？

对于每个发现的问题，请给出具体的改进建议。
"""

TEST_GENERATE_PROMPT = """请为以下文件生成测试用例。

文件: {file_path}

代码：
```python
{code}
```

项目上下文：
{project_context}

请生成全面的测试用例，包括：
1. 正常路径测试
2. 边界条件测试
3. 异常情况测试
4. 使用 pytest 框架
5. 包含有意义的测试名称和注释
"""

REFACTOR_PROMPT = """请根据以下指示重构代码。

文件: {file_path}

重构指示：{instructions}

代码：
```python
{code}
```

请：
1. 保持功能不变
2. 改善代码结构和可读性
3. 遵循最佳实践
4. 解释每个改动的理由
"""
```

### 60.3.6 入口文件

```python
# main.py
import os
import sys
from agent import CodeAgent


def main():
    print("=" * 60)
    print("  代码 Agent v1.0")
    print("  你的 AI 编程助手")
    print("=" * 60)

    # 检查 API Key
    if not os.environ.get("OPENAI_API_KEY"):
        print("[错误] 请设置 OPENAI_API_KEY 环境变量。")
        return

    # 确定项目路径
    project_path = sys.argv[1] if len(sys.argv) > 1 else "."
    project_path = os.path.abspath(project_path)

    # 创建 Agent
    agent = CodeAgent(
        project_path=project_path,
        model=os.environ.get("CODE_AGENT_MODEL", "gpt-4o"),
    )

    # 初始化项目
    agent.init_project()

    # 交互模式
    print("\n命令：")
    print("  /explain <file>     - 解释代码")
    print("  /bug <error_msg>    - 修复 Bug")
    print("  /generate <desc>    - 生成代码")
    print("  /review <file>      - 审查代码")
    print("  /test <file>        - 生成测试")
    print("  /refactor <file>    - 重构代码")
    print("  /status             - 查看项目状态")
    print("  /quit               - 退出")
    print("-" * 40)

    while True:
        try:
            user_input = input("\ncode-agent> ").strip()
        except (EOFError, KeyboardInterrupt):
            break

        if not user_input:
            continue

        if user_input == "/quit":
            break

        elif user_input.startswith("/explain"):
            parts = user_input.split(maxsplit=1)
            file_path = parts[1] if len(parts) > 1 else None
            if file_path:
                result = agent.explain_code(file_path=file_path)
                print(f"\n{result}")
            else:
                print("用法: /explain <文件路径>")

        elif user_input.startswith("/bug"):
            error = user_input.split(maxsplit=1)
            if len(error) > 1:
                result = agent.fix_bug(error_message=error[1])
                print(f"\n{result}")
            else:
                print("用法: /bug <错误信息>")

        elif user_input.startswith("/generate"):
            desc = user_input.split(maxsplit=1)
            if len(desc) > 1:
                result = agent.generate_code(description=desc[1])
                print(f"\n{result}")
            else:
                print("用法: /generate <功能描述>")

        elif user_input.startswith("/review"):
            parts = user_input.split(maxsplit=1)
            file_path = parts[1] if len(parts) > 1 else None
            if file_path:
                result = agent.review_code(file_path=file_path)
                print(f"\n{result}")
            else:
                print("用法: /review <文件路径>")

        elif user_input.startswith("/test"):
            parts = user_input.split(maxsplit=1)
            file_path = parts[1] if len(parts) > 1 else None
            if file_path:
                result = agent.generate_tests(file_path=file_path)
                print(f"\n{result}")
            else:
                print("用法: /test <文件路径>")

        elif user_input.startswith("/refactor"):
            parts = user_input.split(maxsplit=1)
            if len(parts) > 1:
                file_path = parts[1].split()[0]
                instructions = parts[1][len(file_path):].strip()
                if not instructions:
                    instructions = "改善代码的可读性和结构"
                result = agent.refactor(file_path=file_path, instructions=instructions)
                print(f"\n{result}")
            else:
                print("用法: /refactor <文件路径> [重构指示]")

        elif user_input == "/status":
            if agent.project_analysis:
                pa = agent.project_analysis
                print(f"\n项目: {pa.root_path}")
                print(f"文件数: {pa.total_files}")
                print(f"代码行数: {pa.total_lines}")
                print(f"语言: {pa.languages}")
                print(f"Git 状态:")
                if agent.git.is_git_repo():
                    print(agent.git.get_status())
                else:
                    print("  不是 Git 仓库")
            else:
                print("项目未初始化。")

        else:
            # 自由对话
            result = agent._chat(user_input)
            print(f"\n{result}")

    print("\n再见！")


if __name__ == "__main__":
    main()
```

---

## 60.4 案例分析

### 60.4.1 案例一：定位并修复一个真实的 Bug

假设项目中有一个处理用户注册的函数，运行时报错 "TypeError: expected string, got int"。

```
code-agent> /bug TypeError: expected string, got int

[分析过程]
错误发生在 register_user 函数中，第 42 行的 validate_email 调用。
原因：从表单获取的 age 字段被传给了 email 验证函数。

[修复方案]
将第 42 行的 validate_email(user_data["age"]) 
改为 validate_email(user_data["email"])

[修复代码]
- line 42: email = validate_email(user_data["age"])
+ line 42: email = validate_email(user_data["email"])
```

### 60.4.2 案例二：为现有代码生成测试

```
code-agent> /test utils/parser.py

[分析 parser.py 中的函数]
- parse_config(file_path) -> dict
- validate_config(config) -> bool
- merge_configs(base, override) -> dict

[生成测试]
- test_parse_config_valid_json()
- test_parse_config_missing_file()
- test_parse_config_invalid_json()
- test_validate_config_complete()
- test_validate_config_missing_required()
- test_merge_configs_basic()
- test_merge_configs_deep_nested()
```

---

## 60.5 常见坑与解决方案

### 坑一：LLM 生成的代码风格与项目不一致

**问题描述：** Agent 生成的代码使用了与项目不同的命名规范或代码风格。

**解决方案：** 在项目根目录放置 `.editorconfig` 或 `pyproject.toml` 文件，并在提示词中包含项目的代码风格指南。更高级的做法是让 Agent 先读取项目中的几个示例文件来学习风格。

### 坑二：修复 Bug 时引入新的问题

**问题描述：** Agent 的修复虽然解决了原来的错误，但影响了其他功能。

**解决方案：** 每次修复后都运行完整的测试套件。如果项目没有足够的测试，Agent 应该先补充测试，确保现有功能被覆盖，然后再进行修复。

### 坑三：Agent 无法理解复杂的业务逻辑

**问题描述：** 对于涉及复杂业务规则的代码，Agent 的理解不够准确。

**解决方案：** 在提示词中提供业务上下文信息。例如，告诉 Agent "这是一个电商系统，订单金额不能为负数"。也可以让 Agent 先阅读相关的文档和注释。

### 坑四：代码执行的安全风险

**问题描述：** 让 Agent 执行代码可能带来安全风险。

**解决方案：** 严格的沙箱机制：使用 Docker 容器限制执行环境；代码执行前进行静态扫描；敏感操作（文件删除、网络请求）需要用户确认。

---

## 60.6 练习题

### 练习一：添加代码搜索功能（难度：初级）

请为代码 Agent 添加代码搜索功能。支持按函数名、类名、正则表达式等条件在项目中搜索代码。

### 练习二：实现依赖分析（难度：中级）

请实现模块依赖分析功能。生成项目的依赖图，分析循环依赖，识别核心模块。

### 练习三：构建代码文档生成器（难度：中级）

请实现自动文档生成功能。根据代码的 AST 分析和 LLM 理解，自动生成 API 文档和 README。

### 练习四：实现多文件重构（难度：高级）

当前 Agent 只支持单文件操作。请实现跨文件的重构功能，如提取公共函数、重新组织模块结构等。

### 练习五：添加性能分析功能（难度：高级）

请为代码 Agent 添加性能分析能力。使用 cProfile 分析代码性能，找出瓶颈，给出优化建议。

---

## 60.7 实战任务

### 任务一：用代码 Agent 分析自己的项目

选择你正在维护的一个项目，用代码 Agent 分析它的代码结构。生成一份代码质量报告，包括函数复杂度分析、代码覆盖率估算、潜在问题列表。

### 任务二：用代码 Agent 修复一个真实 Bug

找一个你项目中遇到的真实 Bug（或者从 GitHub Issues 中找一个），使用代码 Agent 来辅助定位和修复。记录 Agent 的表现和你的修改。

### 任务三：对比不同模型的效果

用 GPT-4o、Claude 3.5 Sonnet 和 GPT-4o-mini 分别执行同一个代码任务（如生成测试用例），对比它们的代码质量和理解能力。

---

## 60.8 本章小结

代码 Agent 是 AI Agent 在软件开发领域最有前景的应用之一。通过本章的项目实践，我们深入理解了几个关键点。

第一，**代码理解是代码 Agent 的基础能力**。通过 AST 分析和 LLM 理解的结合，Agent 能够准确把握代码的结构和逻辑。AST 提供了精确的语法信息，LLM 提供了语义理解能力，两者互补。

第二，**安全机制是代码 Agent 的生命线**。代码具有执行能力，任何错误的修改都可能导致严重后果。通过 Git 版本控制、测试验证、沙箱执行等多层安全机制，我们可以将风险控制在可接受的范围内。

第三，**代码 Agent 最适合辅助而非替代**。目前的 LLM 在复杂逻辑推理、大型项目理解等方面仍有局限。代码 Agent 的最佳使用方式是作为开发者的"第二双眼睛"，帮助发现容易忽略的问题，加速重复性的编码工作。

下一章，我们将构建一个"个人数字员工"项目，把前面学到的各种 Agent 能力整合到一个统一的系统中。
