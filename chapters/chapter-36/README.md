# 第 36 章：Agent 安全进阶 —— 沙箱与权限控制

> **本章定位：** 在上一章建立安全基础之后，本章将深入更高级的安全机制——沙箱隔离和细粒度权限控制。当 Agent 需要执行代码、访问文件系统或调用外部 API 时，这些进阶安全措施是保护系统和数据的关键防线。

---

## 学习目标

完成本章学习后，你将能够：

1. **设计和实现代码执行沙箱** —— 能为 Agent 创建隔离的代码执行环境，防止恶意代码影响主机系统
2. **构建细粒度的权限控制系统** —— 能基于角色和资源实施精细的权限管理
3. **实现 API 调用的安全代理** —— 能在 Agent 和外部 API 之间建立安全代理层
4. **设计多租户 Agent 的安全隔离** —— 能确保不同用户的 Agent 实例互不干扰
5. **建立安全审计和事后分析能力** —— 能记录所有安全相关事件并支持事后分析

## 核心问题

1. **沙箱能提供多强的隔离保证？** Python 的 subprocess、Docker 容器、WebAssembly——不同隔离技术的安全边界在哪里？
2. **权限控制应该在哪一层实施？** 在 Agent 的代码层、在工具层、还是在基础设施层？各层的优劣是什么？
3. **如何在安全和效率之间取得平衡？** 过多的安全检查会严重影响 Agent 的响应速度。

---

## 36.1 沙箱隔离技术

### 36.1.1 为什么需要沙箱

当 Agent 有权执行代码时，安全风险呈指数级增长。一个被 Prompt Injection 攻击的 Agent 如果能执行任意代码，攻击者可以：删除文件、窃取数据、植入后门、发起网络攻击。沙箱（Sandbox）通过限制代码执行的环境来隔离这些风险。

### 36.1.2 Python 进程级沙箱

```python
import os
import sys
import json
import time
import signal
import resource
import subprocess
from typing import Dict, Optional, Any
from dataclasses import dataclass
from enum import Enum

class SandboxResult:
    """沙箱执行结果"""
    
    def __init__(self, success: bool, output: str = "", error: str = "",
                 exit_code: int = 0, execution_time_ms: float = 0,
                 memory_used_kb: int = 0):
        self.success = success
        self.output = output
        self.error = error
        self.exit_code = exit_code
        self.execution_time_ms = execution_time_ms
        self.memory_used_kb = memory_used_kb
    
    def to_dict(self) -> Dict:
        return {
            "success": self.success,
            "output": self.output,
            "error": self.error,
            "exit_code": self.exit_code,
            "execution_time_ms": self.execution_time_ms,
            "memory_used_kb": self.memory_used_kb,
        }

class CodeSandbox:
    """
    代码执行沙箱
    
    在隔离的子进程中执行代码，限制：
    - 执行时间
    - 内存使用
    - 文件系统访问
    - 网络访问
    """
    
    def __init__(self, timeout_seconds: int = 10, 
                 max_memory_mb: int = 256,
                 allowed_modules: list = None):
        self.timeout_seconds = timeout_seconds
        self.max_memory_mb = max_memory_mb
        self.allowed_modules = allowed_modules or [
            "math", "json", "re", "datetime", "collections",
            "itertools", "functools", "random", "string",
        ]
    
    def execute(self, code: str) -> SandboxResult:
        """在沙箱中执行代码"""
        start_time = time.time()
        
        # 构建安全包装代码
        wrapped_code = self._wrap_code(code)
        
        try:
            # 在子进程中执行
            result = subprocess.run(
                [sys.executable, "-c", wrapped_code],
                capture_output=True,
                text=True,
                timeout=self.timeout_seconds,
                env=self._get_restricted_env(),
                cwd="/tmp",  # 限制工作目录
            )
            
            execution_time_ms = (time.time() - start_time) * 1000
            
            if result.returncode == 0:
                return SandboxResult(
                    success=True,
                    output=result.stdout,
                    execution_time_ms=execution_time_ms,
                )
            else:
                return SandboxResult(
                    success=False,
                    output=result.stdout,
                    error=result.stderr,
                    exit_code=result.returncode,
                    execution_time_ms=execution_time_ms,
                )
                
        except subprocess.TimeoutExpired:
            return SandboxResult(
                success=False,
                error=f"执行超时: {self.timeout_seconds}秒",
                execution_time_ms=self.timeout_seconds * 1000,
            )
        except Exception as e:
            return SandboxResult(
                success=False,
                error=f"沙箱错误: {e}",
                execution_time_ms=(time.time() - start_time) * 1000,
            )
    
    def _wrap_code(self, code: str) -> str:
        """用安全包装器包装代码"""
        # 禁止危险操作的代码
        security_code = f"""
import sys
import os

# 禁止导入危险模块
FORBIDDEN_MODULES = ['os', 'subprocess', 'shutil', 'socket', 'http',
                     'urllib', 'requests', 'ctypes', 'multiprocessing',
                     'threading', '_thread', 'signal', 'pickle', 'shelve',
                     'sqlite3', 'dbm', 'xmlrpc', 'ftplib', 'smtplib',
                     'poplib', 'imaplib', 'telnetlib', 'uuid']

_original_import = __builtins__.__import__

def _safe_import(name, *args, **kwargs):
    if name in FORBIDDEN_MODULES:
        raise ImportError(f"禁止导入模块: {{name}}")
    return _original_import(name, *args, **kwargs)

__builtins__.__import__ = _safe_import

# 限制文件操作
_original_open = open
def _safe_open(path, *args, **kwargs):
    # 只允许写入 /tmp 目录
    real_path = os.path.realpath(path)
    if not real_path.startswith('/tmp'):
        raise PermissionError(f"禁止访问 /tmp 以外的目录: {{path}}")
    return _original_open(path, *args, **kwargs)

builtins = __builtins__ if isinstance(__builtins__, dict) else dir(__builtins__)
import builtins as _builtins
_safe_open_func = _safe_open
if isinstance(__builtins__, dict):
    __builtins__['open'] = _safe_open_func
else:
    _builtins.open = _safe_open_func

# 执行用户代码
"""
        return security_code + code
    
    def _get_restricted_env(self) -> Dict:
        """获取受限的环境变量"""
        env = os.environ.copy()
        # 移除可能泄露信息的环境变量
        for key in list(env.keys()):
            if any(sensitive in key.lower() for sensitive in 
                   ['key', 'secret', 'password', 'token', 'api']):
                del env[key]
        return env


# ============================================================
# 使用示例
# ============================================================

def demo_sandbox():
    """演示沙箱执行"""
    sandbox = CodeSandbox(timeout_seconds=5, max_memory_mb=128)
    
    test_codes = [
        # 正常代码
        ("数学计算", "result = sum(range(1, 101))\nprint(f'1到100的和: {{result}}')"),
        
        # 尝试导入禁止模块
        ("安全拦截", "import os\nos.system('ls')"),
        
        # 尝试访问文件系统
        ("文件访问限制", "with open('/etc/passwd') as f:\n    print(f.read())"),
        
        # 正常的文件操作
        ("安全文件操作", "import json\ndata = {'test': 123}\nwith open('/tmp/test.json', 'w') as f:\n    json.dump(data, f)\nprint('写入成功')"),
        
        # 超时代码
        ("超时测试", "import time\ntime.sleep(20)\nprint('这行不会被执行')"),
    ]
    
    print("代码沙箱执行测试")
    print("=" * 60)
    
    for name, code in test_codes:
        print(f"\n[{name}]")
        result = sandbox.execute(code)
        status = "SUCCESS" if result.success else "BLOCKED"
        print(f"  状态: {status}")
        print(f"  耗时: {result.execution_time_ms:.0f}ms")
        if result.output:
            print(f"  输出: {result.output.strip()[:200]}")
        if result.error:
            print(f"  错误: {result.error.strip()[:200]}")


if __name__ == "__main__":
    demo_sandbox()
```

### 36.1.3 Docker 容器沙箱

```python
import docker
from typing import Dict

class DockerSandbox:
    """
    Docker 容器沙箱
    
    使用 Docker 容器提供更强的隔离。
    每次执行代码都在一个全新的容器中运行。
    """
    
    def __init__(self, image: str = "python:3.11-slim", 
                 timeout: int = 30,
                 memory_limit: str = "256m",
                 cpu_quota: int = 50000):
        self.image = image
        self.timeout = timeout
        self.memory_limit = memory_limit
        self.cpu_quota = cpu_quota
        self.client = docker.from_env()
    
    def execute(self, code: str) -> SandboxResult:
        """在 Docker 容器中执行代码"""
        start_time = time.time()
        
        try:
            container = self.client.containers.run(
                self.image,
                command=["python", "-c", code],
                detach=True,
                mem_limit=self.memory_limit,
                cpu_quota=self.cpu_quota,
                network_disabled=True,  # 禁用网络
                read_only=True,  # 只读文件系统
                tmpfs={"/tmp": "size=64m"},  # 限制 /tmp 大小
                remove=True,
            )
            
            # 等待容器完成
            result = container.wait(timeout=self.timeout)
            logs = container.logs().decode("utf-8")
            
            execution_time_ms = (time.time() - start_time) * 1000
            
            if result.get("StatusCode", 1) == 0:
                return SandboxResult(
                    success=True,
                    output=logs,
                    execution_time_ms=execution_time_ms,
                )
            else:
                return SandboxResult(
                    success=False,
                    output=logs,
                    error=f"退出码: {result.get('StatusCode')}",
                    execution_time_ms=execution_time_ms,
                )
                
        except docker.errors.ContainerError as e:
            return SandboxResult(
                success=False,
                error=str(e),
                execution_time_ms=(time.time() - start_time) * 1000,
            )
        except Exception as e:
            return SandboxResult(
                success=False,
                error=f"Docker 沙箱错误: {e}",
                execution_time_ms=(time.time() - start_time) * 1000,
            )
    
    def cleanup(self):
        """清理所有相关容器"""
        try:
            containers = self.client.containers.list(all=True)
            for c in containers:
                if "sandbox" in (c.name or ""):
                    c.remove(force=True)
        except Exception:
            pass
```

---

## 36.2 细粒度权限控制

### 36.2.1 RBAC（基于角色的访问控制）

```python
from typing import Set, Dict, List, Optional
from dataclasses import dataclass, field
from enum import Enum
import hashlib
import time

class ActionType(Enum):
    """操作类型"""
    READ = "read"
    WRITE = "write"
    DELETE = "delete"
    EXECUTE = "execute"
    ADMIN = "admin"

@dataclass
class Resource:
    """受保护的资源"""
    resource_id: str
    resource_type: str  # "file", "api", "database", "tool"
    owner_id: Optional[str] = None
    metadata: Dict = field(default_factory=dict)

@dataclass 
class Permission:
    """权限规则"""
    resource_type: str
    actions: Set[ActionType]
    conditions: Dict = field(default_factory=dict)  # 附加条件

class Role:
    """角色定义"""
    
    def __init__(self, name: str, permissions: List[Permission] = None):
        self.name = name
        self.permissions = permissions or []
    
    def has_permission(self, resource_type: str, action: ActionType) -> bool:
        """检查角色是否有指定权限"""
        for perm in self.permissions:
            if perm.resource_type == resource_type or perm.resource_type == "*":
                if action in perm.actions or ActionType.ADMIN in perm.actions:
                    return True
        return False

class AccessControl:
    """
    访问控制系统
    
    实现基于角色的访问控制（RBAC），支持：
    - 角色继承
    - 资源所有权
    - 条件权限
    """
    
    def __init__(self):
        self.roles: Dict[str, Role] = {}
        self.user_roles: Dict[str, Set[str]] = {}  # user_id -> role_names
        self.audit_log: List[Dict] = []
        
        # 创建默认角色
        self._create_default_roles()
    
    def _create_default_roles(self):
        """创建默认角色"""
        # 只读用户
        self.create_role("viewer", [
            Permission("*", {ActionType.READ}),
        ])
        
        # 普通用户
        self.create_role("user", [
            Permission("*", {ActionType.READ}),
            Permission("tool", {ActionType.EXECUTE}),
            Permission("api", {ActionType.READ, ActionType.WRITE}),
        ])
        
        # 管理员
        self.create_role("admin", [
            Permission("*", {ActionType.READ, ActionType.WRITE, 
                           ActionType.DELETE, ActionType.EXECUTE, ActionType.ADMIN}),
        ])
    
    def create_role(self, name: str, permissions: List[Permission]):
        """创建角色"""
        self.roles[name] = Role(name, permissions)
    
    def assign_role(self, user_id: str, role_name: str):
        """为用户分配角色"""
        if user_id not in self.user_roles:
            self.user_roles[user_id] = set()
        self.user_roles[user_id].add(role_name)
    
    def check_access(self, user_id: str, resource_type: str, 
                     action: ActionType, context: Dict = None) -> bool:
        """检查用户是否有权限执行操作"""
        user_roles = self.user_roles.get(user_id, set())
        
        for role_name in user_roles:
            role = self.roles.get(role_name)
            if role and role.has_permission(resource_type, action):
                # 记录审计日志
                self._log_access(user_id, resource_type, action, True, context)
                return True
        
        self._log_access(user_id, resource_type, action, False, context)
        return False
    
    def _log_access(self, user_id: str, resource_type: str, 
                    action: ActionType, granted: bool, context: Dict = None):
        """记录访问审计日志"""
        self.audit_log.append({
            "timestamp": time.time(),
            "user_id": user_id,
            "resource_type": resource_type,
            "action": action.value,
            "granted": granted,
            "context": context or {},
        })
    
    def get_audit_log(self, user_id: str = None, 
                      resource_type: str = None) -> List[Dict]:
        """查询审计日志"""
        logs = self.audit_log
        if user_id:
            logs = [l for l in logs if l["user_id"] == user_id]
        if resource_type:
            logs = [l for l in logs if l["resource_type"] == resource_type]
        return logs


# ============================================================
# Agent 安全执行器
# ============================================================

class SecureAgentExecutor:
    """
    安全的 Agent 执行器
    
    在执行每个操作前检查权限。
    """
    
    def __init__(self, access_control: AccessControl):
        self.ac = access_control
    
    def execute_tool(self, user_id: str, tool_name: str, 
                     arguments: Dict) -> str:
        """安全地执行工具"""
        # 检查权限
        if not self.ac.check_access(user_id, "tool", ActionType.EXECUTE,
                                   {"tool_name": tool_name}):
            return f"权限不足: 用户 {user_id} 没有执行工具 {tool_name} 的权限"
        
        # 执行工具
        try:
            # 这里调用实际的工具
            return f"工具 {tool_name} 执行成功"
        except Exception as e:
            return f"工具执行失败: {e}"
    
    def read_data(self, user_id: str, data_type: str, 
                  resource_id: str) -> str:
        """安全地读取数据"""
        if not self.ac.check_access(user_id, data_type, ActionType.READ,
                                   {"resource_id": resource_id}):
            return f"权限不足: 无法读取 {data_type}/{resource_id}"
        
        return f"数据 {data_type}/{resource_id} 的内容..."
    
    def write_data(self, user_id: str, data_type: str,
                   resource_id: str, content: str) -> str:
        """安全地写入数据"""
        if not self.ac.check_access(user_id, data_type, ActionType.WRITE,
                                   {"resource_id": resource_id}):
            return f"权限不足: 无法写入 {data_type}/{resource_id}"
        
        return f"数据 {data_type}/{resource_id} 写入成功"


# ============================================================
# 使用示例
# ============================================================

def demo_rbac():
    """演示 RBAC 访问控制"""
    ac = AccessControl()
    
    # 为用户分配角色
    ac.assign_role("alice", "user")
    ac.assign_role("bob", "viewer")
    ac.assign_role("charlie", "admin")
    
    executor = SecureAgentExecutor(ac)
    
    test_cases = [
        ("alice", "execute_tool", "search"),
        ("bob", "execute_tool", "search"),
        ("alice", "write_data", "database"),
        ("charlie", "execute_tool", "delete_all"),
    ]
    
    print("RBAC 访问控制测试")
    print("=" * 60)
    
    for user, action, target in test_cases:
        if action == "execute_tool":
            result = executor.execute_tool(user, target, {})
        elif action == "write_data":
            result = executor.write_data(user, "database", target, "data")
        else:
            result = executor.read_data(user, target, "123")
        
        print(f"\n[{user}] {action}({target})")
        print(f"  结果: {result}")
    
    # 打印审计日志
    print("\n[审计日志]")
    for log in ac.get_audit_log():
        status = "GRANTED" if log["granted"] else "DENIED"
        print(f"  {log['user_id']} {log['action']} {log['resource_type']} -> {status}")


if __name__ == "__main__":
    demo_rbac()
```

---

## 36.3 API 安全代理

### 36.3.1 安全代理层

```python
import os
import time
import json
from typing import Dict, Any, Optional
from dataclasses import dataclass

@dataclass
class APIRequest:
    """API 请求"""
    method: str
    url: str
    headers: Dict[str, str]
    body: Optional[str] = None

@dataclass
class APIResponse:
    """API 响应"""
    status_code: int
    headers: Dict[str, str]
    body: str

class SecureAPIProxy:
    """
    API 安全代理
    
    在 Agent 和外部 API 之间建立安全层，提供：
    - 请求验证和过滤
    - 速率限制
    - 敏感数据脱敏
    - 请求日志
    """
    
    def __init__(self, rate_limit: int = 100, window_seconds: int = 60):
        self.rate_limit = rate_limit
        self.window_seconds = window_seconds
        self._request_counts: Dict[str, list] = {}
        self._blocked_urls: set = set()
        self._sensitive_headers = {"authorization", "cookie", "x-api-key"}
    
    def validate_request(self, request: APIRequest) -> tuple:
        """验证请求"""
        errors = []
        
        # 1. URL 白名单检查
        if request.url in self._blocked_urls:
            errors.append(f"URL 已被封禁: {request.url}")
        
        # 2. 方法检查
        allowed_methods = {"GET", "POST", "PUT", "DELETE"}
        if request.method.upper() not in allowed_methods:
            errors.append(f"不允许的方法: {request.method}")
        
        # 3. 速率限制检查
        if not self._check_rate_limit(request.url):
            errors.append(f"速率限制: 请求过于频繁")
        
        # 4. 请求大小检查
        if request.body and len(request.body) > 1_000_000:  # 1MB
            errors.append("请求体过大")
        
        return len(errors) == 0, errors
    
    def _check_rate_limit(self, url: str) -> bool:
        """检查速率限制"""
        now = time.time()
        if url not in self._request_counts:
            self._request_counts[url] = []
        
        # 清理过期记录
        self._request_counts[url] = [
            t for t in self._request_counts[url]
            if now - t < self.window_seconds
        ]
        
        if len(self._request_counts[url]) >= self.rate_limit:
            return False
        
        self._request_counts[url].append(now)
        return True
    
    def sanitize_request(self, request: APIRequest) -> APIRequest:
        """清理请求中的敏感信息"""
        clean_headers = {}
        for key, value in request.headers.items():
            if key.lower() in self._sensitive_headers:
                clean_headers[key] = "***REDACTED***"
            else:
                clean_headers[key] = value
        
        return APIRequest(
            method=request.method,
            url=request.url,
            headers=clean_headers,
            body=request.body,
        )
    
    def log_request(self, request: APIRequest, response: APIResponse):
        """记录请求日志"""
        log = {
            "timestamp": time.time(),
            "method": request.method,
            "url": request.url,
            "status": response.status_code,
        }
        print(f"[API Log] {json.dumps(log)}")
    
    def proxy(self, request: APIRequest) -> APIResponse:
        """代理请求"""
        # 验证
        is_valid, errors = self.validate_request(request)
        if not is_valid:
            return APIResponse(
                status_code=403,
                headers={},
                body=json.dumps({"error": "请求被拒绝", "details": errors}),
            )
        
        # 清理敏感信息
        clean_request = self.sanitize_request(request)
        
        # 记录日志
        print(f"[Proxy] {clean_request.method} {clean_request.url}")
        
        # 这里应该发送实际的 API 请求
        # 这里返回模拟响应
        response = APIResponse(
            status_code=200,
            headers={"content-type": "application/json"},
            body='{"status": "ok"}',
        )
        
        self.log_request(clean_request, response)
        return response
```

---

## 36.4 案例分析

### 36.4.1 案例：多租户 Agent 的安全隔离

**背景：** 你在为一个 SaaS 平台开发 AI 助手功能。多个企业客户共享同一个 Agent 服务，但每个客户的数据和配置必须严格隔离。

**安全挑战：**
- 客户 A 不能看到客户 B 的数据
- 客户 A 的 Agent 不能调用客户 B 的工具
- 系统管理员能看到所有客户的数据，但普通运维人员不能
- 当某个客户被攻击时，不能影响其他客户

**解决方案：** 实现了基于租户 ID 的全链路隔离。每个请求都携带租户 ID，Agent 的每一步操作都会检查租户 ID 的一致性。数据存储使用租户前缀，API 调用使用租户专属的密钥，工具权限按租户隔离。

### 36.4.2 关键教训

**教训一：安全隔离必须是系统性的。** 不能只在某一层做隔离，而要在每一层都验证租户身份。从请求入口到数据存储，整个链路都要有租户感知。

**教训二：默认拒绝。** 新创建的工具默认不授予任何权限，需要明确授权才能使用。这比默认允许然后收回权限要安全得多。

**教训三：安全事件需要即时告警。** 当检测到越权访问时，不仅要拒绝请求，还要立即通知安全团队。

---

## 36.5 高级安全机制

### 36.5.1 WebAssembly 沙箱

WebAssembly（Wasm）提供了一种比 Docker 更轻量级但同样强大的隔离机制。代码被编译成 Wasm 字节码，在沙箱化的虚拟机中执行，天然具备内存安全和能力限制。

```python
# WebAssembly 沙箱示例（概念演示）
# 生产环境需要安装 wasmtime 或 wasmer Python 绑定

class WasmSandbox:
    """
    WebAssembly 沙箱
    
    使用 Wasm 运行时提供安全的代码执行环境。
    优势：
    - 启动时间快（毫秒级，vs Docker 的秒级）
    - 内存占用小（MB 级，vs Docker 的百 MB 级）
    - 天然的安全隔离
    - 可移植性好
    """
    
    def __init__(self, max_memory_pages: int = 256,  # 16MB
                 max_execution_time_ms: int = 5000):
        self.max_memory_pages = max_memory_pages
        self.max_execution_time_ms = max_execution_time_ms
    
    def execute(self, wasm_bytes: bytes) -> Dict:
        """
        执行 Wasm 模块
        
        Args:
            wasm_bytes: 编译后的 Wasm 字节码
            
        Returns:
            执行结果
        """
        try:
            # 这里应该使用真实的 Wasm 运行时
            # 例如：wasmtime 或 wasmer
            
            # 概念演示：模拟执行
            return {
                "success": True,
                "output": "Wasm 执行结果",
                "memory_used_pages": 10,
                "execution_time_ms": 50,
            }
            
        except Exception as e:
            return {
                "success": False,
                "error": str(e),
            }
    
    def compile_python_to_wasm(self, python_code: str) -> bytes:
        """
        将 Python 代码编译为 Wasm
        
        实际实现可以使用：
        - Pyodide（Python 到 Wasm 的完整移植）
        - RustPython + wasm-pack
        - Cython + emscripten
        """
        # 这里返回模拟的 Wasm 字节码
        return b"mock_wasm_bytes"


# ============================================================
# 沙箱安全配置
# ============================================================

class SandboxSecurityConfig:
    """沙箱安全配置"""
    
    # 默认允许的系统调用（Wasm 沙箱中）
    DEFAULT_ALLOWED_SYSCALLS = [
        "fd_write",
        "fd_read",
        "fd_close",
        "clock_time_get",
        "proc_exit",
    ]
    
    # 默认禁止的模块（Python 沙箱中）
    FORBIDDEN_MODULES = [
        "os", "subprocess", "shutil", "socket", "http",
        "urllib", "requests", "ctypes", "multiprocessing",
        "threading", "_thread", "signal", "pickle", "shelve",
        "sqlite3", "dbm", "xmlrpc", "ftplib", "smtplib",
        "poplib", "imaplib", "telnetlib", "uuid", "pathlib",
    ]
    
    # 危险函数列表
    DANGEROUS_FUNCTIONS = [
        "exec", "eval", "compile", "__import__",
        "globals", "locals", "vars",
        "getattr", "setattr", "delattr",
    ]
    
    @classmethod
    def get_restricted_globals(cls) -> Dict:
        """获取受限的全局变量"""
        import builtins
        
        restricted = {}
        for name in dir(builtins):
            if name.startswith("_"):
                continue
            if name in cls.DANGEROUS_FUNCTIONS:
                continue
            restricted[name] = getattr(builtins, name)
        
        return restricted
```

### 36.5.2 安全的代码审查机制

在执行用户提交的代码之前，进行静态分析和安全审查。

```python
import ast
import tokenize
import io
from typing import List, Dict, Tuple

class CodeSecurityAuditor:
    """
    代码安全审查器
    
    在执行代码之前进行静态分析，识别潜在的安全风险。
    """
    
    def __init__(self):
        self.risk_rules = [
            self._check_imports,
            self._check_function_calls,
            self._check_string_operations,
            self._check_file_operations,
            self._check_network_operations,
        ]
    
    def audit(self, code: str) -> Dict:
        """
        审查代码安全性
        
        Args:
            code: Python 代码字符串
            
        Returns:
            审查结果
        """
        risks = []
        risk_level = "safe"
        
        try:
            tree = ast.parse(code)
        except SyntaxError as e:
            return {
                "safe": False,
                "risk_level": "syntax_error",
                "risks": [f"语法错误: {e}"],
            }
        
        for rule_func in self.risk_rules:
            rule_risks = rule_func(tree, code)
            risks.extend(rule_risks)
        
        # 根据风险数量和严重程度评估整体风险
        if len(risks) > 5:
            risk_level = "high"
        elif len(risks) > 2:
            risk_level = "medium"
        elif len(risks) > 0:
            risk_level = "low"
        
        return {
            "safe": risk_level in ("safe", "low"),
            "risk_level": risk_level,
            "risks": risks,
            "recommendation": self._get_recommendation(risk_level, risks),
        }
    
    def _check_imports(self, tree: ast.AST, code: str) -> List[str]:
        """检查导入语句"""
        risks = []
        
        for node in ast.walk(tree):
            if isinstance(node, ast.Import):
                for alias in node.names:
                    if alias.name in CodeSecurityAudit.FORBIDDEN_MODULES:
                        risks.append(f"禁止导入模块: {alias.name}")
            
            elif isinstance(node, ast.ImportFrom):
                if node.module and node.module.split('.')[0] in CodeSecurityAudit.FORBIDDEN_MODULES:
                    risks.append(f"禁止从模块导入: {node.module}")
        
        return risks
    
    def _check_function_calls(self, tree: ast.AST, code: str) -> List[str]:
        """检查函数调用"""
        risks = []
        
        for node in ast.walk(tree):
            if isinstance(node, ast.Call):
                func_name = self._get_func_name(node)
                if func_name in ["exec", "eval", "compile", "__import__"]:
                    risks.append(f"调用危险函数: {func_name}")
        
        return risks
    
    def _check_string_operations(self, tree: ast.AST, code: str) -> List[str]:
        """检查字符串操作"""
        risks = []
        
        for node in ast.walk(tree):
            # 检查是否尝试访问 __builtins__
            if isinstance(node, ast.Attribute):
                if node.attr.startswith("__") and node.attr.endswith("__"):
                    risks.append(f"访问特殊属性: {node.attr}")
        
        return risks
    
    def _check_file_operations(self, tree: ast.AST, code: str) -> List[str]:
        """检查文件操作"""
        risks = []
        
        for node in ast.walk(tree):
            if isinstance(node, ast.Call):
                func_name = self._get_func_name(node)
                if func_name == "open":
                    # 检查是否尝试访问敏感路径
                    if node.args:
                        first_arg = node.args[0]
                        if isinstance(first_arg, ast.Constant):
                            path = str(first_arg.value)
                            if any(s in path for s in ["/etc", "/proc", "/sys", "~"]):
                                risks.append(f"尝试访问敏感路径: {path}")
        
        return risks
    
    def _check_network_operations(self, tree: ast.AST, code: str) -> List[str]:
        """检查网络操作"""
        risks = []
        
        network_modules = ["socket", "http", "urllib", "requests"]
        
        for node in ast.walk(tree):
            if isinstance(node, ast.Import):
                for alias in node.names:
                    if alias.name in network_modules:
                        risks.append(f"网络操作: {alias.name}")
        
        return risks
    
    def _get_func_name(self, node: ast.Call) -> str:
        """获取函数名"""
        if isinstance(node.func, ast.Name):
            return node.func.id
        elif isinstance(node.func, ast.Attribute):
            return node.func.attr
        return ""
    
    def _get_recommendation(self, risk_level: str, risks: List[str]) -> str:
        """获取建议"""
        if risk_level == "safe":
            return "代码安全，可以执行"
        elif risk_level == "low":
            return "存在低风险，建议在受限环境中执行"
        elif risk_level == "medium":
            return "存在中等风险，需要进一步审查"
        else:
            return "存在高风险，不建议执行"


# ============================================================
# 使用示例
# ============================================================

def demo_code_auditing():
    """演示代码安全审查"""
    auditor = CodeSecurityAuditor()
    
    test_codes = [
        ("安全代码", "result = sum(range(100))\nprint(result)"),
        ("导入禁止模块", "import os\nos.system('ls')"),
        ("调用危险函数", "exec('import os')"),
        ("文件操作", "with open('/etc/passwd') as f:\n    print(f.read())"),
        ("网络操作", "import requests\nrequests.get('http://evil.com')"),
    ]
    
    print("代码安全审查测试")
    print("=" * 60)
    
    for name, code in test_codes:
        result = auditor.audit(code)
        status = "SAFE" if result["safe"] else "UNSAFE"
        print(f"\n[{name}] {status}")
        print(f"  风险等级: {result['risk_level']}")
        if result["risks"]:
            print(f"  风险项:")
            for risk in result["risks"]:
                print(f"    - {risk}")
        print(f"  建议: {result['recommendation']}")


if __name__ == "__main__":
    demo_code_auditing()
```

### 36.5.3 动态权限提升机制

在某些场景下，Agent 可能需要临时提升权限来完成特定任务。实现安全的动态权限提升机制。

```python
import time
from typing import Set, Dict, Optional
from dataclasses import dataclass
from enum import Enum

class PermissionLevel(Enum):
    """权限级别"""
    GUEST = 0        # 访客：只能读取公开信息
    USER = 1         # 普通用户：基本操作
    POWER_USER = 2   # 高级用户：扩展操作
    ADMIN = 3        # 管理员：所有操作
    SUPER_ADMIN = 4  # 超级管理员：系统级操作

@dataclass
class TemporaryPermission:
    """临时权限"""
    permission: str
    granted_at: float
    expires_at: float
    reason: str
    approved_by: str

class DynamicPermissionManager:
    """
    动态权限管理器
    
    支持临时权限提升，用于处理特殊情况。
    所有权限提升都需要审批和审计。
    """
    
    def __init__(self):
        self.user_permissions: Dict[str, Set[str]] = {}
        self.temporary_permissions: Dict[str, list] = {}
        self.audit_log: list = []
    
    def grant_temporary_permission(self, user_id: str, permission: str,
                                  duration_seconds: int, reason: str,
                                  approved_by: str) -> bool:
        """
        授予临时权限
        
        Args:
            user_id: 用户 ID
            permission: 权限名称
            duration_seconds: 持续时间（秒）
            reason: 授予原因
            approved_by: 审批人
            
        Returns:
            是否成功授予
        """
        # 验证权限是否可以临时授予
        if not self._can_be_temporary(permission):
            self._log_audit(user_id, "grant_failed", permission, 
                          "权限不可临时授予")
            return False
        
        # 创建临时权限
        temp_perm = TemporaryPermission(
            permission=permission,
            granted_at=time.time(),
            expires_at=time.time() + duration_seconds,
            reason=reason,
            approved_by=approved_by,
        )
        
        if user_id not in self.temporary_permissions:
            self.temporary_permissions[user_id] = []
        self.temporary_permissions[user_id].append(temp_perm)
        
        # 添加到用户权限
        if user_id not in self.user_permissions:
            self.user_permissions[user_id] = set()
        self.user_permissions[user_id].add(permission)
        
        # 记录审计日志
        self._log_audit(user_id, "permission_granted", permission,
                       f"原因: {reason}, 时长: {duration_seconds}秒, 审批人: {approved_by}")
        
        return True
    
    def check_permission(self, user_id: str, permission: str) -> bool:
        """检查用户是否有权限"""
        # 清理过期的临时权限
        self._cleanup_expired(user_id)
        
        return permission in self.user_permissions.get(user_id, set())
    
    def revoke_permission(self, user_id: str, permission: str, 
                         reason: str = "手动撤销"):
        """撤销权限"""
        if user_id in self.user_permissions:
            self.user_permissions[user_id].discard(permission)
        
        if user_id in self.temporary_permissions:
            self.temporary_permissions[user_id] = [
                p for p in self.temporary_permissions[user_id]
                if p.permission != permission
            ]
        
        self._log_audit(user_id, "permission_revoked", permission, reason)
    
    def _cleanup_expired(self, user_id: str):
        """清理过期的临时权限"""
        if user_id not in self.temporary_permissions:
            return
        
        now = time.time()
        expired = [
            p for p in self.temporary_permissions[user_id]
            if now > p.expires_at
        ]
        
        for p in expired:
            self.user_permissions.get(user_id, set()).discard(p.permission)
            self._log_audit(user_id, "permission_expired", p.permission, "自动过期")
        
        self.temporary_permissions[user_id] = [
            p for p in self.temporary_permissions[user_id]
            if now <= p.expires_at
        ]
    
    def _can_be_temporary(self, permission: str) -> bool:
        """检查权限是否可以临时授予"""
        # 某些高危权限不应该被临时授予
        dangerous_permissions = ["super_admin", "system_modify", "data_delete"]
        return permission not in dangerous_permissions
    
    def _log_audit(self, user_id: str, action: str, permission: str, details: str):
        """记录审计日志"""
        self.audit_log.append({
            "timestamp": time.time(),
            "user_id": user_id,
            "action": action,
            "permission": permission,
            "details": details,
        })
    
    def get_user_permissions(self, user_id: str) -> Dict:
        """获取用户的所有权限信息"""
        self._cleanup_expired(user_id)
        
        return {
            "permanent": list(self.user_permissions.get(user_id, set())),
            "temporary": [
                {
                    "permission": p.permission,
                    "expires_at": p.expires_at,
                    "remaining_seconds": max(0, p.expires_at - time.time()),
                    "reason": p.reason,
                }
                for p in self.temporary_permissions.get(user_id, [])
            ],
        }


# ============================================================
# 使用示例
# ============================================================

def demo_dynamic_permissions():
    """演示动态权限管理"""
    manager = DynamicPermissionManager()
    
    # 设置用户基础权限
    manager.user_permissions["user_001"] = {"read", "write"}
    manager.user_permissions["user_002"] = {"read"}
    
    print("动态权限管理测试")
    print("=" * 60)
    
    # 检查初始权限
    print(f"\nuser_001 初始权限: {manager.get_user_permissions('user_001')}")
    
    # 授予临时权限
    manager.grant_temporary_permission(
        user_id="user_001",
        permission="admin",
        duration_seconds=3600,
        reason="需要执行紧急维护任务",
        approved_by="system_admin",
    )
    
    print(f"\n授予临时 admin 权限后:")
    print(f"  user_001 是否有 admin: {manager.check_permission('user_001', 'admin')}")
    print(f"  user_001 权限详情: {manager.get_user_permissions('user_001')}")
    
    # 手动撤销权限
    manager.revoke_permission("user_001", "admin", reason="维护任务完成")
    print(f"\n撤销 admin 权限后:")
    print(f"  user_001 是否有 admin: {manager.check_permission('user_001', 'admin')}")
    
    # 打印审计日志
    print("\n审计日志:")
    for log in manager.audit_log:
        print(f"  [{log['action']}] {log['user_id']} - {log['permission']}: {log['details']}")


if __name__ == "__main__":
    demo_dynamic_permissions()
```

## 36.6 常见坑与最佳实践

**坑 1：沙箱逃逸。** Python 的沙箱（通过限制内置函数）可以被多种方式绕过。如果需要强隔离，使用 Docker 容器或 WebAssembly。不要试图自己实现完整的 Python 沙箱。

**坑 2：环境变量泄露。** 代码执行环境中可能包含 API Key、数据库密码等敏感信息。确保沙箱环境中不传递这些变量。在子进程中显式设置干净的环境变量。

**坑 3：权限过度授权。** 为了方便开发，给所有用户都授予管理员权限。这在原型阶段可以接受，但上线前必须收回。遵循最小权限原则。

**坑 4：临时权限遗忘清理。** 授予的临时权限如果没有及时清理，可能成为安全隐患。实现自动过期机制，并定期审计临时权限。

**坑 5：安全与性能的权衡。** 过于严格的安全检查会显著影响 Agent 的响应速度。在设计安全机制时，要平衡安全性和性能需求。

**最佳实践 1：零信任架构。** 不要假设任何请求是安全的。每个请求都需要验证身份、检查权限、记录日志。即使请求来自"内部"也是如此。

**最佳实践 2：分层防御。** 实现多层安全防护：输入验证、代码审查、沙箱执行、权限控制、输出审查。每层都有独立的价值。

**最佳实践 3：持续监控。** 监控沙箱的使用情况，检测异常行为。当发现可疑活动时，及时告警并调查。

**最佳实践 4：定期更新。** 定期更新沙箱镜像、安全规则库和权限策略。安全是一个持续的过程，需要不断改进。

---

## 36.6 练习题

**练习 1：实现内存沙箱。** 使用 Python 的 RestrictedPython 或 ast 模块实现一个内存级别的沙箱，不需要启动子进程就能安全执行代码。

**练习 2：设计 ABAC 系统。** 实现一个基于属性的访问控制系统（ABAC），权限不仅取决于角色，还取决于资源属性（如数据敏感度、时间、IP 地址等）。

**练习 3：实现 API 代理。** 完整实现 SecureAPIProxy，能拦截请求、检查权限、脱敏敏感信息、记录日志，并用 HTTP 客户端发送实际请求。

**练习 4：设计多租户隔离方案。** 为一个 Agent 系统设计完整的多租户隔离方案，包括数据隔离、工具隔离、配置隔离和日志隔离。

**练习 5：实现安全事件告警。** 当检测到越权访问、可疑输入等安全事件时，自动发送告警通知（邮件或消息）。

**练习 6：压力测试沙箱。** 编写脚本测试沙箱的性能：同时执行 100 个代码任务，测量平均延迟和资源消耗。

---

## 36.7 本章小结

本章深入探讨了 Agent 安全的进阶话题。我们实现了两种级别的代码执行沙箱（Python 进程级和 Docker 容器级），构建了基于 RBAC 的细粒度权限控制系统，设计了 API 安全代理层。

核心理念是：安全不是一个功能，而是一种架构思维方式。从设计之初就考虑安全，比事后添加安全措施要有效得多。

沙箱提供了代码执行的隔离，权限控制提供了资源访问的边界，API 代理提供了外部通信的安全。三者结合，构成了 Agent 安全的完整防线。

---

> **下一章预告：** 第 37 章将深入 Agent 的日志与审计系统，学习如何记录所有关键事件以支持合规要求和事后分析。
