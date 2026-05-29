# 第 40 章：Agent 部署 —— 从开发到生产

> **本章定位：** 把 Agent 从本地笔记本搬到生产服务器，这中间有无数的坑。本章将覆盖容器化、API 服务化、环境管理、部署策略等关键话题，帮你建立一个可靠的 Agent 部署流水线。

---

## 学习目标

完成本章学习后，你将能够：

1. **容器化 Agent 应用** —— 能用 Docker 将 Agent 打包为可移植的容器镜像
2. **构建 Agent API 服务** —— 能用 FastAPI 搭建一个生产级的 Agent HTTP API
3. **管理环境配置和密钥** —— 能安全地管理不同环境（开发、测试、生产）的配置
4. **实施零停机部署** —— 能使用蓝绿部署或滚动更新策略更新 Agent
5. **处理 Agent 的异步和流式输出** —— 能实现 SSE 流式响应，提升用户体验

## 核心问题

1. **Agent 部署和传统 Web 应用部署有什么不同？** Agent 依赖外部 LLM API、有长运行的推理过程、输出是概率性的——这些特点如何影响部署策略？
2. **如何确保部署过程中 Agent 的服务不中断？** Agent 的请求可能持续数秒甚至数分钟，如何在更新时优雅处理这些长连接？
3. **如何管理 Agent 的多个版本？** 新版本上线后发现问题如何快速回滚？

---

## 40.1 容器化 Agent

### 40.1.1 Dockerfile 编写

```dockerfile
# Dockerfile - Agent 生产部署

# 阶段 1: 构建
FROM python:3.11-slim as builder

WORKDIR /app

# 安装构建依赖
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# 安装 Python 依赖
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# 阶段 2: 运行
FROM python:3.11-slim

WORKDIR /app

# 从构建阶段复制依赖
COPY --from=builder /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH

# 复制应用代码
COPY ./src ./src
COPY ./config ./config

# 创建非 root 用户
RUN groupadd -r agent && useradd -r -g agent agent
USER agent

# 环境变量
ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

# 健康检查
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"

EXPOSE 8000

CMD ["python", "-m", "uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 40.1.2 依赖管理

```txt
# requirements.txt
# LLM SDK
openai>=1.0.0
anthropic>=0.18.0

# Web 框架
fastapi>=0.104.0
uvicorn[standard]>=0.24.0
pydantic>=2.0.0

# 数据处理
numpy>=1.24.0

# 可观测性
langsmith>=0.0.0
prometheus-client>=0.19.0

# 工具
httpx>=0.25.0
tenacity>=8.2.0

# 开发工具（仅开发环境）
# pytest>=7.4.0
# black>=23.0.0
```

### 40.1.3 环境配置管理

```python
import os
from typing import Optional
from pydantic_settings import BaseSettings
from pydantic import Field

class AgentSettings(BaseSettings):
    """
    Agent 配置管理
    
    支持从环境变量、.env 文件和默认值加载配置。
    不同环境（开发、测试、生产）使用不同的配置。
    """
    
    # 应用配置
    app_name: str = "Agent Service"
    app_version: str = "1.0.0"
    debug: bool = False
    environment: str = "development"  # development, testing, production
    
    # 服务器配置
    host: str = "0.0.0.0"
    port: int = 8000
    workers: int = 1
    
    # LLM 配置
    openai_api_key: str = Field(..., env="OPENAI_API_KEY")
    default_model: str = "gpt-4o"
    max_tokens: int = 4096
    temperature: float = 0.7
    
    # Agent 配置
    max_iterations: int = 10
    request_timeout: int = 60
    
    # 安全配置
    rate_limit_per_minute: int = 60
    max_input_length: int = 4000
    
    # 日志配置
    log_level: str = "INFO"
    log_dir: str = "./logs"
    
    # 缓存配置
    cache_enabled: bool = True
    cache_ttl_seconds: int = 3600
    
    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"
        case_sensitive = False

# 全局设置实例
settings = AgentSettings()
```

---

## 40.2 Agent API 服务

### 40.2.1 FastAPI 服务实现

```python
import os
import json
import time
import asyncio
from typing import AsyncGenerator
from datetime import datetime

from fastapi import FastAPI, HTTPException, Request, BackgroundTasks
from fastapi.responses import StreamingResponse
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel, Field

# ============================================================
# 数据模型
# ============================================================

class ChatRequest(BaseModel):
    """聊天请求"""
    message: str = Field(..., min_length=1, max_length=4000)
    session_id: str = Field(default="default")
    stream: bool = Field(default=False)
    temperature: float = Field(default=0.7, ge=0, le=2)
    max_tokens: int = Field(default=1000, ge=1, le=4096)

class ChatResponse(BaseModel):
    """聊天响应"""
    response: str
    session_id: str
    request_id: str
    model: str
    tokens_used: int
    latency_ms: float
    timestamp: str

class HealthResponse(BaseModel):
    """健康检查响应"""
    status: str
    version: str
    uptime_seconds: float
    environment: str

# ============================================================
# FastAPI 应用
# ============================================================

app = FastAPI(
    title="Agent API Service",
    description="生产级 Agent HTTP API",
    version="1.0.0",
)

# CORS 中间件
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # 生产环境应该限制
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 全局变量
start_time = time.time()

# ============================================================
# 核心 Agent 逻辑
# ============================================================

class AgentService:
    """Agent 服务核心"""
    
    def __init__(self):
        self.request_count = 0
        self.total_tokens = 0
    
    async def process(self, request: ChatRequest) -> ChatResponse:
        """处理聊天请求"""
        self.request_count += 1
        request_id = f"req_{int(time.time() * 1000)}"
        
        start = time.time()
        
        # 模拟 Agent 处理
        await asyncio.sleep(0.1)  # 模拟延迟
        
        response_text = f"收到您的消息: {request.message[:50]}。这是 Agent 的回答。"
        tokens_used = len(response_text) // 2 + len(request.message) // 2
        self.total_tokens += tokens_used
        
        latency_ms = (time.time() - start) * 1000
        
        return ChatResponse(
            response=response_text,
            session_id=request.session_id,
            request_id=request_id,
            model="gpt-4o",
            tokens_used=tokens_used,
            latency_ms=latency_ms,
            timestamp=datetime.utcnow().isoformat(),
        )
    
    async def process_stream(self, request: ChatRequest) -> AsyncGenerator[str, None]:
        """流式处理聊天请求"""
        response_text = f"这是流式响应示例。您的消息是: {request.message[:50]}。"
        
        # 模拟流式输出
        words = response_text.split()
        for word in words:
            yield f"data: {json.dumps({'content': word + ' ', 'done': False})}\n\n"
            await asyncio.sleep(0.05)
        
        yield f"data: {json.dumps({'content': '', 'done': True})}\n\n"

agent_service = AgentService()

# ============================================================
# API 端点
# ============================================================

@app.get("/health", response_model=HealthResponse)
async def health_check():
    """健康检查"""
    return HealthResponse(
        status="healthy",
        version="1.0.0",
        uptime_seconds=time.time() - start_time,
        environment=os.environ.get("ENVIRONMENT", "development"),
    )

@app.post("/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):
    """同步聊天端点"""
    try:
        if request.stream:
            # 流式响应
            return StreamingResponse(
                agent_service.process_stream(request),
                media_type="text/event-stream",
            )
        
        # 同步响应
        response = await agent_service.process(request)
        return response
        
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/stats")
async def get_stats():
    """获取服务统计"""
    return {
        "request_count": agent_service.request_count,
        "total_tokens": agent_service.total_tokens,
        "uptime_seconds": time.time() - start_time,
    }


# ============================================================
# 启动命令
# ============================================================
# uvicorn src.main:app --host 0.0.0.0 --port 8000 --workers 4
```

### 40.2.2 流式响应实现

```python
import asyncio
from typing import AsyncGenerator

async def stream_agent_response(user_input: str) -> AsyncGenerator[str, None]:
    """
    流式 Agent 响应
    
    将 Agent 的输出以 SSE (Server-Sent Events) 格式流式返回。
    """
    # 模拟 Agent 的流式处理
    steps = [
        "正在理解您的问题...\n",
        "正在搜索相关信息...\n",
        "正在生成回答...\n\n",
        "基于搜索结果，",
        "这是一个关于 ",
        user_input[:20],
        " 的回答。",
        " 希望对您有帮助。",
    ]
    
    for step in steps:
        yield f"data: {json.dumps({'content': step})}\n\n"
        await asyncio.sleep(0.1)
    
    # 发送完成信号
    yield f"data: {json.dumps({'done': True})}\n\n"


# FastAPI 流式端点
@app.post("/chat/stream")
async def chat_stream(request: ChatRequest):
    """流式聊天端点"""
    return StreamingResponse(
        stream_agent_response(request.message),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",
        },
    )
```

---

## 40.3 部署策略

### 40.3.1 docker-compose 配置

```yaml
# docker-compose.yml
version: '3.8'

services:
  agent-api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - ENVIRONMENT=production
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - LOG_LEVEL=INFO
    volumes:
      - ./logs:/app/logs
    restart: unless-stopped
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"]
      interval: 30s
      timeout: 10s
      retries: 3
```

### 40.3.2 部署检查清单

在部署 Agent 到生产环境之前，检查以下项目：

1. **环境变量** —— 所有 API Key 和敏感配置通过环境变量注入，而不是硬编码在代码中
2. **健康检查** —— /health 端点正常工作，能准确反映服务状态
3. **日志** —— 日志级别正确配置，敏感信息已脱敏
4. **监控** —— Prometheus 指标端点正常，关键指标（请求数、延迟、错误率）可查询
5. **安全** —— 输入验证、输出审查、速率限制都已启用
6. **回滚方案** —— 已准备好快速回滚的脚本和流程

---

## 40.4 案例分析

### 40.4.1 案例：零停机部署

**背景：** Agent 服务每天处理 10 万次请求，不能在更新时中断服务。

**方案：** 使用 Docker Compose 的 rolling update 策略。更新时先启动新版本容器，等待健康检查通过后，再停止旧版本容器。在切换过程中，负载均衡器会将请求路由到健康的容器。

**关键配置：**
```yaml
deploy:
  update_config:
    parallelism: 1
    delay: 10s
    failure_action: rollback
    order: start-first
```

**结果：** 更新过程中服务零中断，用户无感知。

---

## 40.5 高级部署技术

### 40.5.1 健康检查与优雅关闭

生产环境中的 Agent 服务需要完善的健康检查和优雅关闭机制。

```python
import os
import signal
import asyncio
from typing import Dict, Any
from datetime import datetime
import time

class HealthChecker:
    """
    健康检查器
    
    提供多维度的健康状态检查。
    """
    
    def __init__(self):
        self.start_time = time.time()
        self._is_shutting_down = False
        self._active_requests = 0
        self._checks = {
            "liveness": self._check_liveness,
            "readiness": self._check_readiness,
            "dependencies": self._check_dependencies,
        }
    
    def check(self, check_type: str = "full") -> Dict[str, Any]:
        """
        执行健康检查
        
        Args:
            check_type: 检查类型 (liveness/readiness/full)
            
        Returns:
            健康状态
        """
        if check_type == "full":
            results = {}
            for name, check_func in self._checks.items():
                results[name] = check_func()
            
            overall_status = "healthy"
            if any(r["status"] != "healthy" for r in results.values()):
                overall_status = "unhealthy"
            
            return {
                "status": overall_status,
                "timestamp": datetime.now().isoformat(),
                "uptime_seconds": time.time() - self.start_time,
                "checks": results,
            }
        elif check_type in self._checks:
            result = self._checks[check_type]()
            result["timestamp"] = datetime.now().isoformat()
            return result
        
        return {"status": "unknown", "error": f"Unknown check type: {check_type}"}
    
    def _check_liveness(self) -> Dict:
        """存活检查 - 服务进程是否正常运行"""
        return {
            "status": "healthy" if not self._is_shutting_down else "shutting_down",
            "message": "Service is running" if not self._is_shutting_down else "Service is shutting down",
        }
    
    def _check_readiness(self) -> Dict:
        """就绪检查 - 服务是否准备好接受请求"""
        if self._is_shutting_down:
            return {"status": "unhealthy", "message": "Service is shutting down"}
        
        # 检查是否有过多的活跃请求
        if self._active_requests > 100:
            return {"status": "degraded", "message": f"High load: {self._active_requests} active requests"}
        
        return {"status": "healthy", "message": "Ready to accept requests"}
    
    def _check_dependencies(self) -> Dict:
        """依赖检查 - 外部依赖是否可用"""
        # 检查 LLM API（模拟）
        llm_healthy = True  # 实际应该测试 API 连接
        
        # 检查数据库（模拟）
        db_healthy = True
        
        if not llm_healthy:
            return {"status": "unhealthy", "message": "LLM API unavailable"}
        
        if not db_healthy:
            return {"status": "unhealthy", "message": "Database unavailable"}
        
        return {"status": "healthy", "message": "All dependencies available"}
    
    def start_shutdown(self):
        """开始关闭流程"""
        self._is_shutting_down = True
    
    def increment_requests(self):
        """增加活跃请求数"""
        self._active_requests += 1
    
    def decrement_requests(self):
        """减少活跃请求数"""
        self._active_requests = max(0, self._active_requests - 1)


class GracefulShutdownHandler:
    """
    优雅关闭处理器
    
    确保服务在关闭时正确处理所有进行中的请求。
    """
    
    def __init__(self, health_checker: HealthChecker):
        self.health_checker = health_checker
        self._shutdown_timeout = 30  # 秒
        self._shutdown_event = asyncio.Event()
    
    def setup_signal_handlers(self):
        """设置信号处理器"""
        signal.signal(signal.SIGTERM, self._handle_sigterm)
        signal.signal(signal.SIGINT, self._handle_sigint)
    
    def _handle_sigterm(self, signum, frame):
        """处理 SIGTERM 信号"""
        print("\n收到 SIGTERM 信号，开始优雅关闭...")
        self.start_shutdown()
    
    def _handle_sigint(self, signum, frame):
        """处理 SIGINT 信号"""
        print("\n收到 SIGINT 信号，开始优雅关闭...")
        self.start_shutdown()
    
    def start_shutdown(self):
        """开始关闭流程"""
        self.health_checker.start_shutdown()
        self._shutdown_event.set()
    
    async def wait_for_shutdown(self):
        """等待关闭完成"""
        await self._shutdown_event.wait()
        
        # 等待活跃请求完成
        start_time = time.time()
        while self.health_checker._active_requests > 0:
            if time.time() - start_time > self._shutdown_timeout:
                print(f"关闭超时，强制关闭（还有 {self.health_checker._active_requests} 个活跃请求）")
                break
            await asyncio.sleep(0.1)
        
        print("优雅关闭完成")


# ============================================================
# FastAPI 集成示例
# ============================================================

# 在实际的 FastAPI 应用中使用：
"""
from fastapi import FastAPI, Request
from contextlib import asynccontextmanager

health_checker = HealthChecker()
shutdown_handler = GracefulShutdownHandler(health_checker)

@asynccontextmanager
async def lifespan(app: FastAPI):
    # 启动时
    shutdown_handler.setup_signal_handlers()
    yield
    # 关闭时
    shutdown_handler.start_shutdown()
    await shutdown_handler.wait_for_shutdown()

app = FastAPI(lifespan=lifespan)

@app.get("/health")
async def health_check():
    return health_checker.check("full")

@app.get("/health/live")
async def liveness():
    return health_checker.check("liveness")

@app.get("/health/ready")
async def readiness():
    return health_checker.check("readiness")

@app.middleware("http")
async def track_requests(request: Request, call_next):
    health_checker.increment_requests()
    try:
        response = await call_next(request)
        return response
    finally:
        health_checker.decrement_requests()
"""


def demo_health_check():
    """演示健康检查"""
    checker = HealthChecker()
    
    print("健康检查测试")
    print("=" * 60)
    
    # 完整检查
    result = checker.check("full")
    print(f"\n完整检查: {result['status']}")
    print(f"  运行时间: {result['uptime_seconds']:.1f}秒")
    for check_name, check_result in result["checks"].items():
        print(f"  {check_name}: {check_result['status']}")
    
    # 模拟高负载
    for _ in range(101):
        checker.increment_requests()
    
    result = checker.check("readiness")
    print(f"\n高负载下就绪检查: {result['status']}")
    print(f"  消息: {result['message']}")
    
    # 模拟关闭
    checker.start_shutdown()
    result = checker.check("liveness")
    print(f"\n关闭中存活检查: {result['status']}")


if __name__ == "__main__":
    demo_health_check()
```

### 40.5.2 配置热更新

允许在不重启服务的情况下更新某些配置。

```python
import json
import time
import threading
from typing import Dict, Any, Callable, Optional
from pathlib import Path
from dataclasses import dataclass, field
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

@dataclass
class HotReloadConfig:
    """支持热更新的配置"""
    # 不支持热更新的配置（需要重启）
    _static_keys = {"host", "port", "workers", "log_file"}
    
    # 动态配置
    temperature: float = 0.7
    max_tokens: int = 4096
    rate_limit_per_minute: int = 60
    timeout_seconds: int = 30
    enable_cache: bool = True
    
    # 配置版本
    version: int = 0
    last_updated: str = ""

class ConfigHotReloader:
    """
    配置热更新器
    
    监控配置文件变化，自动重新加载。
    """
    
    def __init__(self, config_path: str, reload_interval: int = 5):
        self.config_path = Path(config_path)
        self.reload_interval = reload_interval
        self.config = HotReloadConfig()
        self._callbacks: list = []
        self._observer: Optional[Observer] = None
        self._running = False
    
    def load_config(self) -> Dict:
        """加载配置"""
        try:
            if self.config_path.exists():
                with open(self.config_path, "r", encoding="utf-8") as f:
                    data = json.load(f)
                
                # 更新配置
                for key, value in data.items():
                    if hasattr(self.config, key) and key not in HotReloadConfig._static_keys:
                        setattr(self.config, key, value)
                
                self.config.version += 1
                self.config.last_updated = time.strftime("%Y-%m-%d %H:%M:%S")
                
                return data
        except Exception as e:
            print(f"加载配置失败: {e}")
        
        return {}
    
    def register_callback(self, callback: Callable):
        """注册配置更新回调"""
        self._callbacks.append(callback)
    
    def start_watching(self):
        """开始监控配置文件"""
        class ConfigFileHandler(FileSystemEventHandler):
            def __init__(self, reloader):
                self.reloader = reloader
            
            def on_modified(self, event):
                if event.src_path == str(self.reloader.config_path):
                    print(f"配置文件变化: {event.src_path}")
                    self.reloader.reload()
            
            def on_created(self, event):
                if event.src_path == str(self.reloader.config_path):
                    print(f"配置文件创建: {event.src_path}")
                    self.reloader.reload()
        
        self._running = True
        self._observer = Observer()
        handler = ConfigFileHandler(self)
        self._observer.schedule(handler, path=str(self.config_path.parent), recursive=False)
        self._observer.start()
        
        print(f"开始监控配置文件: {self.config_path}")
    
    def stop_watching(self):
        """停止监控"""
        self._running = False
        if self._observer:
            self._observer.stop()
            self._observer.join()
    
    def reload(self):
        """重新加载配置"""
        old_version = self.config.version
        self.load_config()
        
        if self.config.version != old_version:
            print(f"配置已更新: v{old_version} -> v{self.config.version}")
            
            # 触发回调
            for callback in self._callbacks:
                try:
                    callback(self.config)
                except Exception as e:
                    print(f"配置更新回调失败: {e}")
    
    def get_config(self) -> HotReloadConfig:
        """获取当前配置"""
        return self.config


# ============================================================
# 使用示例
# ============================================================

def demo_hot_reload():
    """演示配置热更新"""
    # 创建示例配置文件
    config_dir = Path("./demo_config")
    config_dir.mkdir(exist_ok=True)
    config_file = config_dir / "agent_config.json"
    
    # 初始配置
    initial_config = {
        "temperature": 0.7,
        "max_tokens": 4096,
        "rate_limit_per_minute": 60,
        "timeout_seconds": 30,
        "enable_cache": True,
    }
    
    with open(config_file, "w", encoding="utf-8") as f:
        json.dump(initial_config, f, indent=2)
    
    # 创建热更新器
    reloader = ConfigHotReloader(str(config_file))
    
    # 注册回调
    def on_config_change(config):
        print(f"  回调触发: temperature={config.temperature}, max_tokens={config.max_tokens}")
    
    reloader.register_callback(on_config_change)
    
    # 初始加载
    reloader.load_config()
    config = reloader.get_config()
    
    print("配置热更新测试")
    print("=" * 60)
    print(f"\n初始配置:")
    print(f"  temperature: {config.temperature}")
    print(f"  max_tokens: {config.max_tokens}")
    print(f"  版本: {config.version}")
    
    # 模拟更新配置
    time.sleep(1)
    updated_config = {
        "temperature": 0.5,  # 修改了
        "max_tokens": 4096,
        "rate_limit_per_minute": 100,  # 修改了
        "timeout_seconds": 30,
        "enable_cache": True,
    }
    
    with open(config_file, "w", encoding="utf-8") as f:
        json.dump(updated_config, f, indent=2)
    
    # 手动重新加载（生产环境应该使用文件监控）
    reloader.reload()
    config = reloader.get_config()
    
    print(f"\n更新后配置:")
    print(f"  temperature: {config.temperature}")
    print(f"  max_tokens: {config.max_tokens}")
    print(f"  rate_limit_per_minute: {config.rate_limit_per_minute}")
    print(f"  版本: {config.version}")
    print(f"  最后更新: {config.last_updated}")


if __name__ == "__main__":
    demo_hot_reload()
```

### 40.5.3 CI/CD 流水线配置

```yaml
# .github/workflows/agent-deploy.yml
name: Agent CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/agent-service

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-asyncio
      
      - name: Run tests
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: pytest tests/ -v --tb=short

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run security scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          severity: 'CRITICAL,HIGH'

  build:
    needs: [test, security-scan]
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha
            type=raw,value=latest
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    
    steps:
      - name: Deploy to production
        run: |
          echo "Deploying to production..."
          # 这里添加实际的部署命令
          # 例如：kubectl apply -f k8s/
          # 或：docker-compose -f docker-compose.prod.yml up -d
      
      - name: Health check
        run: |
          sleep 30
          curl -f https://your-agent-service.com/health || exit 1
          echo "Deployment successful!"
```

## 40.6 常见坑与最佳实践

**坑 1：镜像太大。** 使用多阶段构建和 slim 基础镜像可以显著减小镜像大小。一个典型的 Agent 镜像应该在 200-500MB 之间。

**坑 2：API Key 泄露到镜像中。** 永远不要在 Dockerfile 中 COPY .env 文件或硬编码 API Key。使用 Docker Secrets 或环境变量注入。

**坑 3：没有处理 LLM API 的限流。** LLM 提供商有速率限制。实现指数退避重试和请求队列，避免在高峰期被打爆。

**坑 4：忽略启动时间。** Agent 服务可能需要加载模型或预热缓存，导致启动时间较长。在健康检查中配置足够的启动时间。

**坑 5：日志输出到 stdout/stderr。** 容器化环境中，应该将日志输出到标准输出，由容器运行时统一收集。

**最佳实践 1：使用 CI/CD 自动化部署。** 每次代码合并到主分支时自动构建、测试和部署。减少人为操作带来的错误。

**最佳实践 2：实施蓝绿部署或金丝雀发布。** 不要一次性替换所有实例，而是逐步发布，降低风险。

**最佳实践 3：配置回滚策略。** 确保能快速回滚到上一个稳定版本。自动化回滚脚本可以大大缩短故障恢复时间。

**最佳实践 4：监控部署过程。** 在部署过程中监控关键指标（错误率、延迟），当指标异常时自动暂停部署。

---

## 40.6 练习题

**练习 1：完善 Dockerfile。** 为你的 Agent 项目编写一个优化的 Dockerfile，使用多阶段构建，最终镜像大小控制在 300MB 以内。

**练习 2：实现优雅关闭。** 为 Agent API 添加优雅关闭功能：收到 SIGTERM 信号后，等待正在处理的请求完成，然后关闭服务。

**练习 3：实现请求队列。** 当 LLM API 触发限流时，将请求放入队列稍后重试，而不是直接返回错误。

**练习 4：设计回滚方案。** 编写一个回滚脚本，能在 30 秒内将 Agent 回退到上一个稳定版本。

**练习 5：实现配置热更新。** 允许在不重启服务的情况下更新某些配置（如 temperature、max_tokens）。

**练习 6：设计多区域部署。** 为 Agent 设计一个多区域部署方案：在两个不同的数据中心部署 Agent，通过 DNS 负载均衡分配流量。

---

## 40.7 本章小结

本章覆盖了 Agent 从开发到生产部署的关键环节。从 Docker 容器化到 FastAPI 服务化，从环境配置管理到流式响应，从部署策略到回滚方案，我们建立了一个完整的 Agent 部署知识体系。

核心理念是：部署不是一次性的事情，而是一个持续的过程。建立自动化的 CI/CD 流水线，让每次代码变更都能安全、快速地到达生产环境。

---

> **下一章预告：** 第 41 章将深入 Agent 的监控与告警系统，学习如何在生产环境中实时守护 Agent 的健康运行。
