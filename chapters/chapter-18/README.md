# 第 18 章：Tool Use 进阶 —— 动态工具与工具学习

---

## 学习目标

完成本章学习后，你将能够：

1. 理解为什么需要动态工具加载——静态工具列表的局限性以及动态加载带来的灵活性
2. 掌握工具发现和推荐机制——Agent 如何根据任务自动找到最合适的工具
3. 了解工具组合和工作流编排——如何将多个工具串联成复杂的工作流
4. 能够实现工具的自动注册、管理和热重载
5. 理解工具使用的安全性和权限控制——如何防止 Agent 滥用工具
6. 掌握工具使用的效果评估方法——如何知道工具用得好不好

## 核心问题

1. 如何让 Agent 动态发现和使用新工具？
2. 如何安全地管理和执行工具？
3. 如何评估工具使用的效果？

---

## 18.1 动态工具加载

### 18.1.1 为什么需要动态工具

在最简单的 Agent 系统中，工具是在代码里静态定义的——你在写代码时就把所有工具列好了，Agent 只能使用这些预定义的工具。这种方式在工具数量少、需求固定时没问题，但在以下场景中就不够用了：

**场景 1：工具数量很多**

假设你有一个 Agent 需要处理各种任务——写代码、查天气、搜索信息、操作数据库、发送邮件……如果把所有工具都塞进 Prompt，会消耗大量 Token，而且 LLM 在众多工具中选择时也容易犯错。

**场景 2：需要添加新工具**

用户说："我有一个内部 API，你能不能也支持调用？" 如果用静态方式，你需要修改代码、重新部署。而动态加载可以让你在运行时添加新工具，不需要重启。

**场景 3：不同任务需要不同工具**

写代码的任务不需要查天气的工具，查天气的任务不需要操作数据库的工具。动态加载可以根据当前任务只加载需要的工具，减少干扰。

### 18.1.2 动态工具加载器的实现

```python
import importlib
import inspect
from pathlib import Path
from typing import Callable


class DynamicToolLoader:
    """
    动态工具加载器
    
    从指定目录自动发现和加载工具。
    工具文件遵循约定：文件名以 tool_ 开头的函数会被自动加载。
    
    使用方式：
    1. 在 tools/ 目录下创建 Python 文件
    2. 文件中定义以 tool_ 开头的函数
    3. 加载器会自动发现并注册这些工具
    """
    
    def __init__(self, tools_dir: str = "./tools"):
        self.tools_dir = Path(tools_dir)
        self.loaded_tools: dict[str, Callable] = {}
        self.tool_metadata: dict[str, dict] = {}
    
    def load_tools(self) -> dict[str, Callable]:
        """
        从目录加载所有工具
        
        扫描 tools/ 目录下的所有 .py 文件，
        找到以 tool_ 开头的函数并注册为工具。
        """
        if not self.tools_dir.exists():
            print(f"工具目录不存在: {self.tools_dir}")
            return self.loaded_tools
        
        for tool_file in self.tools_dir.glob("*.py"):
            if tool_file.name.startswith("_"):
                continue  # 跳过 __init__.py 等私有文件
            
            try:
                self._load_file(tool_file)
            except Exception as e:
                print(f"加载 {tool_file.name} 失败: {e}")
        
        print(f"共加载 {len(self.loaded_tools)} 个工具")
        return self.loaded_tools
    
    def _load_file(self, file_path: Path):
        """加载单个工具文件"""
        module_name = file_path.stem
        spec = importlib.util.spec_from_file_location(module_name, file_path)
        module = importlib.util.module_from_spec(spec)
        spec.loader.exec_module(module)
        
        # 找到所有以 tool_ 开头的函数
        for name, obj in inspect.getmembers(module, inspect.isfunction):
            if name.startswith("tool_"):
                tool_name = name[5:]  # 去掉 tool_ 前缀
                self.loaded_tools[tool_name] = obj
                self.tool_metadata[tool_name] = {
                    "name": tool_name,
                    "description": (obj.__doc__ or "无描述").strip(),
                    "parameters": self._extract_parameters(obj),
                    "source_file": str(file_path.name),
                }
        
        print(f"  已加载: {file_path.name}")
    
    def _extract_parameters(self, func: Callable) -> dict:
        """提取函数的参数信息"""
        sig = inspect.signature(func)
        params = {}
        
        for name, param in sig.parameters.items():
            param_info = {
                "type": str(param.annotation) if param.annotation != inspect.Parameter.empty else "str",
                "required": param.default == inspect.Parameter.empty,
            }
            if param.default != inspect.Parameter.empty:
                param_info["default"] = param.default
            params[name] = param_info
        
        return params
    
    def get_tool_descriptions(self) -> str:
        """获取所有工具的格式化描述（用于注入 Prompt）"""
        lines = []
        for name, meta in self.tool_metadata.items():
            params_desc = ", ".join(
                f"{k}" + ("" if v.get("required") else f"={v.get('default', '')}")
                for k, v in meta["parameters"].items()
            )
            lines.append(f"- {name}({params_desc}): {meta['description']}")
        return "\n".join(lines)
    
    def reload_tools(self):
        """重新加载所有工具（热重载）"""
        self.loaded_tools.clear()
        self.tool_metadata.clear()
        self.load_tools()


# 工具文件示例：tools/search_tools.py
SEARCH_TOOLS_EXAMPLE = '''
def tool_web_search(query: str) -> str:
    """搜索互联网获取信息"""
    import requests
    # 实际实现中会调用搜索 API
    return f"搜索结果: 关于 {query} 的信息..."

def tool_news_search(keyword: str, days: int = 7) -> str:
    """搜索最近的新闻"""
    return f"最近 {days} 天关于 {keyword} 的新闻..."
'''

# 工具文件示例：tools/code_tools.py
CODE_TOOLS_EXAMPLE = '''
def tool_run_python(code: str) -> str:
    """执行 Python 代码并返回结果"""
    try:
        import io
        import contextlib
        output = io.StringIO()
        with contextlib.redirect_stdout(output):
            exec(code)
        return output.getvalue() or "代码执行成功（无输出）"
    except Exception as e:
        return f"执行错误: {e}"

def tool_explain_code(code: str) -> str:
    """解释代码的功能"""
    return f"这段代码的功能是: ..."
'''
```

### 18.1.3 工具注册表

工具注册表是管理工具的"中央控制台"，它记录了所有可用工具的信息、使用统计和健康状态。

```python
import json
import time
from pathlib import Path
from typing import Callable


class ToolRegistry:
    """
    工具注册表
    
    功能：
    1. 工具的注册和注销
    2. 工具信息的持久化存储
    3. 使用统计（调用次数、成功率）
    4. 工具健康状态监控
    """
    
    def __init__(self, storage_path: str = "tool_registry.json"):
        self.storage_path = Path(storage_path)
        self.tools: dict[str, dict] = {}
        self._load()
    
    def _load(self):
        """从文件加载注册表"""
        if self.storage_path.exists():
            with open(self.storage_path, "r", encoding="utf-8") as f:
                self.tools = json.load(f)
    
    def _save(self):
        """保存注册表到文件"""
        with open(self.storage_path, "w", encoding="utf-8") as f:
            json.dump(self.tools, f, ensure_ascii=False, indent=2)
    
    def register(self, tool_name: str, tool_func: Callable,
                 metadata: dict = None):
        """注册一个新工具"""
        sig = inspect.signature(tool_func)
        
        self.tools[tool_name] = {
            "name": tool_name,
            "description": (tool_func.__doc__ or "无描述").strip(),
            "parameters": {
                name: {
                    "type": str(param.annotation) if param.annotation != inspect.Parameter.empty else "str",
                    "required": param.default == inspect.Parameter.empty,
                }
                for name, param in sig.parameters.items()
            },
            "metadata": metadata or {},
            "registered_at": time.strftime("%Y-%m-%d %H:%M:%S"),
            "stats": {
                "total_calls": 0,
                "success_count": 0,
                "failure_count": 0,
                "total_duration": 0.0,
                "avg_duration": 0.0,
                "success_rate": 1.0,
            },
        }
        self._save()
        print(f"已注册工具: {tool_name}")
    
    def unregister(self, tool_name: str):
        """注销工具"""
        if tool_name in self.tools:
            del self.tools[tool_name]
            self._save()
            print(f"已注销工具: {tool_name}")
    
    def get_tool(self, tool_name: str) -> dict | None:
        """获取工具信息"""
        return self.tools.get(tool_name)
    
    def list_tools(self) -> list[dict]:
        """列出所有工具"""
        return list(self.tools.values())
    
    def update_stats(self, tool_name: str, success: bool, duration: float):
        """更新使用统计"""
        if tool_name not in self.tools:
            return
        
        stats = self.tools[tool_name]["stats"]
        stats["total_calls"] += 1
        
        if success:
            stats["success_count"] += 1
        else:
            stats["failure_count"] += 1
        
        stats["total_duration"] += duration
        stats["avg_duration"] = stats["total_duration"] / stats["total_calls"]
        stats["success_rate"] = stats["success_count"] / stats["total_calls"]
        
        self._save()
    
    def get_unhealthy_tools(self, min_calls: int = 10,
                            min_success_rate: float = 0.5) -> list[dict]:
        """找出"不健康"的工具（成功率过低）"""
        unhealthy = []
        for name, info in self.tools.items():
            stats = info["stats"]
            if (stats["total_calls"] >= min_calls and
                stats["success_rate"] < min_success_rate):
                unhealthy.append({
                    "name": name,
                    "success_rate": stats["success_rate"],
                    "total_calls": stats["total_calls"],
                })
        return unhealthy
```

---

## 18.2 工具发现与推荐

### 18.2.1 基于任务的工具推荐

当工具数量很多时，Agent 不应该看到所有工具，而应该只看到与当前任务相关的几个工具。工具推荐器就是干这个事的。

```python
class ToolRecommender:
    """
    工具推荐器
    
    根据当前任务，从工具库中推荐最合适的工具。
    评估维度：
    - 工具描述与任务的语义相关性
    - 工具的历史使用成功率
    - 工具的调用复杂度
    """
    
    def __init__(self, llm_client, registry: ToolRegistry):
        self.llm_client = llm_client
        self.registry = registry
    
    def recommend(self, task: str, top_k: int = 3) -> list[dict]:
        """
        根据任务推荐工具
        
        流程：
        1. 获取所有可用工具
        2. 用 LLM 评估每个工具与任务的相关性
        3. 综合相关性和历史成功率排序
        4. 返回 top_k 个推荐工具
        """
        all_tools = self.registry.list_tools()
        
        if not all_tools:
            return []
        
        # 批量评估相关性（一次 LLM 调用评估所有工具）
        tool_scores = self._batch_evaluate(task, all_tools)
        
        # 综合排序：相关性 * 0.7 + 成功率 * 0.3
        scored_tools = []
        for tool_info, relevance_score in zip(all_tools, tool_scores):
            tool_name = tool_info["name"]
            success_rate = tool_info["stats"]["success_rate"]
            combined_score = relevance_score * 0.7 + success_rate * 0.3
            scored_tools.append({
                "tool": tool_info,
                "relevance_score": relevance_score,
                "success_rate": success_rate,
                "combined_score": combined_score,
            })
        
        scored_tools.sort(key=lambda x: x["combined_score"], reverse=True)
        return scored_tools[:top_k]
    
    def _batch_evaluate(self, task: str, tools: list[dict]) -> list[float]:
        """批量评估工具与任务的相关性"""
        tools_desc = "\n".join(
            f"{i+1}. {t['name']}: {t['description']}"
            for i, t in enumerate(tools)
        )
        
        prompt = f"""请评估以下每个工具对完成这个任务的帮助程度。

任务：{task}

可用工具：
{tools_desc}

请为每个工具打分（0-1 分），输出格式：
工具1的分数
工具2的分数
...（每行一个分数）"""

        response = self.llm_client.generate(prompt)
        
        # 解析分数
        scores = []
        for line in response.strip().split("\n"):
            line = line.strip()
            try:
                import re
                numbers = re.findall(r'[\d.]+', line)
                if numbers:
                    scores.append(min(1.0, max(0.0, float(numbers[0]))))
                else:
                    scores.append(0.5)
            except (ValueError, IndexError):
                scores.append(0.5)
        
        # 补齐或截断到和工具数量一致
        while len(scores) < len(tools):
            scores.append(0.5)
        return scores[:len(tools)]
```

---

## 18.3 工具组合与工作流

### 18.3.1 工具链

有时候，一个任务需要多个工具协作完成。工具链将多个工具串联起来，前一个工具的输出作为后一个工具的输入。

```python
class ToolChain:
    """
    工具链：将多个工具按顺序串联
    
    类似于 Unix 的管道（pipe），数据从第一个工具流向最后一个工具。
    
    使用示例：
    chain = ToolChain(tools)
    chain.add_step("search", input_mapping={"query": "user_query"}, output_key="search_result")
    chain.add_step("summarize", input_mapping={"text": "search_result"}, output_key="summary")
    result = chain.execute({"user_query": "Python 异步编程"})
    """
    
    def __init__(self, tools: dict[str, Callable]):
        self.tools = tools
        self.steps: list[dict] = []
    
    def add_step(self, tool_name: str, input_mapping: dict = None,
                 output_key: str = None) -> 'ToolChain':
        """
        添加一个步骤
        
        参数:
            tool_name: 工具名称
            input_mapping: 输入参数映射，格式为 {"参数名": "上下文中的key"}
            output_key: 输出保存到上下文的 key
        """
        self.steps.append({
            "tool": tool_name,
            "input_mapping": input_mapping or {},
            "output_key": output_key or f"step_{len(self.steps)}_result",
        })
        return self  # 支持链式调用
    
    def execute(self, initial_input: dict) -> dict:
        """
        执行工具链
        
        返回包含所有中间结果的上下文字典
        """
        context = initial_input.copy()
        
        for i, step in enumerate(self.steps):
            tool_name = step["tool"]
            
            if tool_name not in self.tools:
                raise ValueError(f"工具 '{tool_name}' 不存在")
            
            # 解析输入：从上下文中取值
            tool_input = {}
            for param_name, context_key in step["input_mapping"].items():
                if context_key in context:
                    tool_input[param_name] = context[context_key]
                else:
                    tool_input[param_name] = context_key  # 直接使用值
            
            # 如果没有指定输入映射，把整个上下文作为输入
            if not tool_input:
                tool_input = context.copy()
            
            # 执行工具
            try:
                result = self.tools[tool_name](**tool_input)
            except Exception as e:
                result = f"工具 {tool_name} 执行出错: {e}"
            
            context[step["output_key"]] = result
        
        return context


# 使用示例
def demo_tool_chain():
    """演示工具链的使用"""
    
    # 定义工具
    def search(query: str) -> str:
        """搜索"""
        return f"搜索 '{query}' 的结果：Python 是一种流行的编程语言..."
    
    def extract_keywords(text: str) -> str:
        """提取关键词"""
        return "Python, 编程语言, 流行"
    
    def generate_summary(keywords: str) -> str:
        """生成摘要"""
        return f"关于 {keywords} 的摘要：Python 是最流行的编程语言之一。"
    
    tools = {
        "search": search,
        "extract_keywords": extract_keywords,
        "generate_summary": generate_summary,
    }
    
    # 构建工具链
    chain = ToolChain(tools)
    chain.add_step("search", {"query": "user_query"}, "search_result")
    chain.add_step("extract_keywords", {"text": "search_result"}, "keywords")
    chain.add_step("generate_summary", {"keywords": "keywords"}, "summary")
    
    # 执行
    result = chain.execute({"user_query": "Python 编程"})
    print(f"最终结果: {result['summary']}")


if __name__ == "__main__":
    demo_tool_chain()
```

### 18.3.2 工作流引擎

工具链是线性的（A → B → C），但实际工作流可能需要分支（if-else）、并行（同时执行 A 和 B）和循环（重复执行直到满足条件）。工作流引擎支持这些更复杂的编排模式。

```python
from enum import Enum
import concurrent.futures


class StepType(Enum):
    TOOL = "tool"           # 工具调用
    CONDITION = "condition" # 条件分支
    PARALLEL = "parallel"   # 并行执行
    LOOP = "loop"           # 循环


class WorkflowEngine:
    """
    工作流引擎
    
    支持四种步骤类型：
    - tool: 调用单个工具
    - condition: 根据条件选择不同的分支
    - parallel: 同时执行多个工具
    - loop: 重复执行直到满足条件
    """
    
    def __init__(self, tools: dict[str, Callable]):
        self.tools = tools
    
    def execute(self, workflow: dict, context: dict = None) -> dict:
        """执行工作流"""
        context = context or {}
        
        for step in workflow.get("steps", []):
            step_type = StepType(step.get("type", "tool"))
            
            if step_type == StepType.TOOL:
                context = self._execute_tool_step(step, context)
            elif step_type == StepType.CONDITION:
                context = self._execute_condition_step(step, context)
            elif step_type == StepType.PARALLEL:
                context = self._execute_parallel_step(step, context)
        
        return context
    
    def _execute_tool_step(self, step: dict, context: dict) -> dict:
        """执行工具步骤"""
        tool_name = step["tool"]
        inputs = self._resolve_inputs(step.get("inputs", {}), context)
        
        if tool_name in self.tools:
            result = self.tools[tool_name](**inputs)
            context[step.get("output_key", "result")] = result
        else:
            context[step.get("output_key", "result")] = f"工具 {tool_name} 不存在"
        
        return context
    
    def _execute_condition_step(self, step: dict, context: dict) -> dict:
        """执行条件分支"""
        condition_expr = step["condition"]
        
        try:
            condition_result = eval(condition_expr, {"context": context, "ctx": context})
        except Exception:
            condition_result = False
        
        if condition_result:
            branch = step.get("then", {})
        else:
            branch = step.get("else", {})
        
        if branch:
            return self.execute(branch, context)
        return context
    
    def _execute_parallel_step(self, step: dict, context: dict) -> dict:
        """并行执行多个工具"""
        with concurrent.futures.ThreadPoolExecutor() as executor:
            futures = {}
            for sub_step in step.get("steps", []):
                future = executor.submit(self._execute_tool_step, sub_step, context.copy())
                futures[future] = sub_step.get("output_key", "result")
            
            for future in concurrent.futures.as_completed(futures):
                output_key = futures[future]
                try:
                    result = future.result()
                    context[output_key] = result
                except Exception as e:
                    context[output_key] = {"error": str(e)}
        
        return context
    
    def _resolve_inputs(self, inputs: dict, context: dict) -> dict:
        """解析输入参数：$variable 从上下文中取值"""
        resolved = {}
        for key, value in inputs.items():
            if isinstance(value, str) and value.startswith("$"):
                var_name = value[1:]
                resolved[key] = context.get(var_name, "")
            else:
                resolved[key] = value
        return resolved
```

---

## 18.4 工具安全性

### 18.4.1 权限控制

Agent 调用工具时需要有权限控制——不是所有工具都能随便调用的。比如"删除数据库"这样的工具应该只有管理员才能使用。

```python
from enum import Enum


class Permission(Enum):
    READ = "read"        # 读取权限
    WRITE = "write"      # 写入权限
    EXECUTE = "execute"  # 执行权限
    NETWORK = "network"  # 网络权限
    DANGEROUS = "dangerous"  # 危险操作权限


class ToolSecurityManager:
    """
    工具安全管理器
    
    通过权限控制确保 Agent 不会滥用工具。
    
    工作流程：
    1. 注册每个工具需要的权限
    2. 设置每个用户的权限
    3. 在工具调用前检查权限
    """
    
    def __init__(self):
        self.tool_permissions: dict[str, set[Permission]] = {}
        self.user_permissions: dict[str, set[Permission]] = {}
        self.audit_log: list[dict] = []
    
    def register_tool_permissions(self, tool_name: str,
                                   permissions: list[Permission]):
        """注册工具所需的权限"""
        self.tool_permissions[tool_name] = set(permissions)
    
    def set_user_permissions(self, user_id: str,
                              permissions: list[Permission]):
        """设置用户权限"""
        self.user_permissions[user_id] = set(permissions)
    
    def check_permission(self, user_id: str, tool_name: str) -> tuple[bool, str]:
        """检查用户是否有权限调用工具"""
        required = self.tool_permissions.get(tool_name, set())
        granted = self.user_permissions.get(user_id, set())
        
        missing = required - granted
        
        if not missing:
            return True, "权限检查通过"
        else:
            missing_names = [p.value for p in missing]
            return False, f"缺少权限: {', '.join(missing_names)}"
    
    def log_attempt(self, user_id: str, tool_name: str,
                    allowed: bool, reason: str):
        """记录工具调用尝试（审计日志）"""
        self.audit_log.append({
            "user_id": user_id,
            "tool_name": tool_name,
            "allowed": allowed,
            "reason": reason,
            "timestamp": time.strftime("%Y-%m-%d %H:%M:%S"),
        })


class ToolInputValidator:
    """
    工具输入验证器
    
    在工具执行前验证输入参数的安全性，
    防止注入攻击、危险命令等。
    """
    
    def __init__(self):
        self.blocked_patterns = [
            (r"rm\s+-rf", "危险的文件删除命令"),
            (r"DROP\s+TABLE", "危险的数据库操作"),
            (r"<script>", "潜在的 XSS 攻击"),
            (r"eval\(", "潜在的代码注入"),
            (r"__import__", "潜在的代码注入"),
            (r"subprocess", "潜在的系统命令注入"),
        ]
    
    def validate(self, tool_name: str, inputs: dict) -> tuple[bool, str]:
        """验证输入参数"""
        import re
        
        for key, value in inputs.items():
            if not isinstance(value, str):
                continue
            
            for pattern, reason in self.blocked_patterns:
                if re.search(pattern, value, re.IGNORECASE):
                    return False, f"参数 '{key}' 包含危险内容: {reason}"
        
        return True, "验证通过"
```

---

## 18.5 工具使用评估

### 18.5.1 使用统计与效果评估

知道工具用得好不好，是优化 Agent 性能的基础。通过收集和分析工具使用数据，我们可以发现哪些工具经常失败、哪些工具效率低、哪些工具需要优化。

```python
class ToolUsageEvaluator:
    """
    工具使用评估器
    
    收集和分析工具使用数据，输出评估报告。
    """
    
    def __init__(self):
        self.records: list[dict] = []
    
    def record(self, tool_name: str, task: str, success: bool,
               duration: float, output: str = "", error: str = ""):
        """记录一次工具使用"""
        self.records.append({
            "tool_name": tool_name,
            "task": task,
            "success": success,
            "duration": duration,
            "output_length": len(output),
            "error": error,
            "timestamp": time.time(),
        })
    
    def get_tool_report(self, tool_name: str = None) -> dict:
        """获取工具使用报告"""
        if tool_name:
            records = [r for r in self.records if r["tool_name"] == tool_name]
        else:
            records = self.records
        
        if not records:
            return {"message": "暂无数据"}
        
        total = len(records)
        success_count = sum(1 for r in records if r["success"])
        avg_duration = sum(r["duration"] for r in records) / total
        
        # 按工具分组
        by_tool = {}
        for r in records:
            name = r["tool_name"]
            if name not in by_tool:
                by_tool[name] = {"total": 0, "success": 0, "total_duration": 0}
            by_tool[name]["total"] += 1
            if r["success"]:
                by_tool[name]["success"] += 1
            by_tool[name]["total_duration"] += r["duration"]
        
        tool_stats = {}
        for name, stats in by_tool.items():
            tool_stats[name] = {
                "total_calls": stats["total"],
                "success_rate": stats["success"] / stats["total"],
                "avg_duration": stats["total_duration"] / stats["total"],
            }
        
        return {
            "total_records": total,
            "overall_success_rate": success_count / total,
            "overall_avg_duration": avg_duration,
            "by_tool": tool_stats,
            "recent_errors": [r for r in records if not r["success"]][-5:],
        }
    
    def print_report(self):
        """打印评估报告"""
        report = self.get_tool_report()
        
        print("\n" + "="*50)
        print("工具使用评估报告")
        print("="*50)
        print(f"总调用次数: {report['total_records']}")
        print(f"总体成功率: {report['overall_success_rate']:.1%}")
        print(f"平均耗时: {report['overall_avg_duration']:.2f}秒")
        
        print("\n各工具统计:")
        for name, stats in report.get("by_tool", {}).items():
            print(f"  {name}: 调用 {stats['total_calls']} 次, "
                  f"成功率 {stats['success_rate']:.1%}, "
                  f"平均耗时 {stats['avg_duration']:.2f}秒")
        
        if report.get("recent_errors"):
            print("\n最近的错误:")
            for err in report["recent_errors"][-3:]:
                print(f"  [{err['tool_name']}] {err['error'][:80]}")
```

---

## 18.6 常见坑

### 18.6.1 工具描述不准确

**问题描述：** 工具的文档字符串（docstring）与实际功能不符，导致 LLM 根据描述选择了错误的工具，或者传入了错误的参数。

**解决方案：** 定期审查和更新工具描述。在描述中包含具体的输入输出示例，让 LLM 更准确地理解工具的用途。

### 18.6.2 工具调用参数错误

**问题描述：** LLM 生成的参数格式不正确。比如工具期望接收一个整数，但 LLM 传了一个字符串。

**解决方案：** 在工具内部添加参数类型校验和自动转换。对于明显的类型错误，自动修正而不是直接报错。

### 18.6.3 工具执行超时

**问题描述：** 工具执行时间过长，阻塞了整个 Agent 的运行。比如调用一个外部 API，对方服务器响应很慢。

**解决方案：** 为每个工具设置超时时间。使用异步执行或线程池来避免阻塞。

```python
import concurrent.futures

def execute_with_timeout(tool_func, args, timeout=30):
    """带超时的工具执行"""
    with concurrent.futures.ThreadPoolExecutor() as executor:
        future = executor.submit(tool_func, **args)
        try:
            return future.result(timeout=timeout)
        except concurrent.futures.TimeoutError:
            return f"工具执行超时（{timeout}秒）"
```

### 18.6.4 工具之间产生冲突

**问题描述：** 两个工具同时操作同一个资源（比如同时写入同一个文件），导致数据不一致。

**解决方案：** 使用锁机制确保互斥访问，或者设计工具时避免共享可变状态。

---

## 18.7 练习题

### 练习 1：实现动态工具加载器

要求：
- 从指定目录自动加载工具
- 提取工具的名称、描述、参数信息
- 支持工具的热重载（不重启程序添加新工具）
- 生成工具描述文本（可直接注入 Prompt）

### 练习 2：实现工具推荐系统

要求：
- 根据任务描述推荐最合适的工具（Top-3）
- 考虑工具的历史使用成功率
- 使用 LLM 评估工具与任务的相关性

### 练习 3：实现工具链

要求：
- 支持将多个工具串联
- 前一个工具的输出可以作为后一个工具的输入
- 支持链式调用 API
- 错误处理（某个工具失败时的处理）

### 练习 4：实现工作流引擎

要求：
- 支持工具调用步骤
- 支持条件分支（if-else）
- 支持并行执行
- 支持循环（重复执行直到满足条件）

### 练习 5：实现工具安全控制

要求：
- 权限管理（注册工具权限、设置用户权限、检查权限）
- 输入验证（检测危险命令、注入攻击）
- 审计日志（记录所有工具调用尝试）

---

## 18.8 实战任务

### 任务：构建可扩展的工具系统

**目标：** 构建一个可扩展、安全、可管理的工具系统，支持动态加载、智能推荐、工作流编排和安全控制。

**要求：**

1. 支持工具的动态加载和卸载
2. 实现工具推荐（根据任务自动选择工具）
3. 支持工具链和工作流编排
4. 实现完整的安全控制（权限、输入验证、审计）
5. 提供使用统计和效果评估

---

## 18.9 本章小结

- **动态工具加载**让 Agent 能灵活使用新工具，无需修改代码。通过文件系统扫描或 API 注册，Agent 可以在运行时发现和加载新工具。

- **工具推荐**根据任务自动选择最合适的工具。当工具数量很多时，把所有工具都塞进 Prompt 既浪费 Token 又容易选错，智能推荐能显著提升选择准确率。

- **工具链和工作流**将多个工具串联或并行，实现复杂功能。工具链适合线性流程，工作流引擎支持条件分支、并行和循环等复杂模式。

- **安全控制**确保工具使用不会造成危害。权限管理控制谁能用什么工具，输入验证防止注入攻击，审计日志记录所有操作。

- **使用评估**帮助优化工具选择和成功率。通过统计调用次数、成功率、响应时间等指标，可以发现需要优化或替换的工具。

