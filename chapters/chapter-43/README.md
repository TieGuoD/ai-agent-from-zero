# 第 43 章：Agent 的可扩展性设计 —— 从单机到集群

> **本章定位：** 当你的 Agent 从每天处理几百个请求增长到几万个，单机架构就会成为瓶颈。本章将系统性地讲解如何设计可扩展的 Agent 架构，让它能应对不断增长的负载。

---

## 学习目标

完成本章学习后，你将能够：

1. **识别 Agent 系统的扩展瓶颈** —— 能分析 LLM 调用、工具执行、数据存储等各层的扩展限制
2. **设计水平扩展架构** —— 能通过多实例部署、负载均衡实现水平扩展
3. **实现请求队列和异步处理** —— 能用消息队列解耦 Agent 的各组件
4. **设计有状态会话管理** —— 能在多实例环境下管理用户的对话状态
5. **实现 LLM 调用的连接池和限流** —— 能高效管理与 LLM API 的连接

## 核心问题

1. **Agent 的扩展瓶颈在哪里？** 和传统 Web 应用不同，Agent 的瓶颈可能在 LLM API 的速率限制上，而不在服务器资源上。
2. **有状态的对话如何在多实例间共享？** Agent 需要记住之前的对话历史，但多实例部署时每个请求可能被路由到不同的实例。
3. **何时应该从单机升级到集群？** 过早扩展浪费资源，过晚扩展影响用户体验。

---

## 43.1 扩展性分析

### 43.1.1 Agent 的扩展瓶颈

**LLM API 速率限制** —— 这是最常见的瓶颈。OpenAI 的速率限制是每分钟几千次请求。当你的 Agent 流量超过这个限制时，再多的服务器实例也没用。

**工具执行延迟** —— 如果 Agent 调用的外部工具（如数据库查询、API 调用）很慢，它会占用并发资源，降低整体吞吐量。

**会话状态存储** —— 有状态的对话需要存储在某个地方。当多实例部署时，需要一个共享的状态存储（如 Redis）。

**Embedding 计算** —— 如果 Agent 使用 RAG，Embedding 模型的推理速度可能成为瓶颈。

### 43.1.2 扩展策略选择

根据瓶颈类型，选择不同的扩展策略：

**LLM API 限制** -> 模型路由、请求队列、缓存、多提供商冗余
**工具执行延迟** -> 异步执行、连接池、工具缓存
**会话状态** -> Redis/数据库存储、无状态设计
**Embedding 瓶颈** -> GPU 实例扩展、缓存、批处理

---

## 43.2 水平扩展架构

### 43.2.1 可扩展的 Agent 架构

```python
import os
import json
import time
import asyncio
import hashlib
from typing import Dict, List, Optional, Any, Callable
from dataclasses import dataclass, field
from collections import defaultdict
import threading

# ============================================================
# 1. 分布式会话存储
# ============================================================

class SessionStore:
    """
    会话存储 - 支持多实例共享
    
    生产环境应该用 Redis，这里用内存模拟。
    """
    
    def __init__(self):
        self._store: Dict[str, Dict] = {}
        self._lock = threading.Lock()
        self._ttl: Dict[str, float] = {}
        self.default_ttl = 3600  # 1小时
    
    def get(self, session_id: str) -> Optional[Dict]:
        """获取会话数据"""
        with self._lock:
            if session_id in self._store:
                # 检查 TTL
                if time.time() < self._ttl.get(session_id, float('inf')):
                    return self._store[session_id]
                else:
                    # 过期
                    del self._store[session_id]
                    del self._ttl[session_id]
        return None
    
    def set(self, session_id: str, data: Dict, ttl: int = None):
        """设置会话数据"""
        with self._lock:
            self._store[session_id] = data
            self._ttl[session_id] = time.time() + (ttl or self.default_ttl)
    
    def delete(self, session_id: str):
        """删除会话"""
        with self._lock:
            self._store.pop(session_id, None)
            self._ttl.pop(session_id, None)
    
    def cleanup_expired(self):
        """清理过期会话"""
        now = time.time()
        with self._lock:
            expired = [k for k, v in self._ttl.items() if now >= v]
            for k in expired:
                del self._store[k]
                del self._ttl[k]


# ============================================================
# 2. LLM 连接池
# ============================================================

class LLMConnectionPool:
    """
    LLM API 连接池
    
    管理与 LLM API 的连接，支持：
    - 连接复用
    - 速率限制
    - 自动重试
    """
    
    def __init__(self, max_concurrent: int = 10, 
                 rate_limit_per_minute: int = 300):
        self.max_concurrent = max_concurrent
        self.rate_limit = rate_limit_per_minute
        self._semaphore = asyncio.Semaphore(max_concurrent)
        self._request_times: List[float] = []
        self._lock = threading.Lock()
    
    async def acquire(self):
        """获取连接"""
        await self._semaphore.acquire()
        
        # 速率限制
        with self._lock:
            now = time.time()
            # 清理一分钟前的记录
            self._request_times = [t for t in self._request_times if now - t < 60]
            
            if len(self._request_times) >= self.rate_limit:
                # 需要等待
                wait_time = 60 - (now - self._request_times[0])
                if wait_time > 0:
                    await asyncio.sleep(wait_time)
            
            self._request_times.append(time.time())
    
    def release(self):
        """释放连接"""
        self._semaphore.release()
    
    async def call(self, func: Callable, *args, **kwargs) -> Any:
        """通过连接池调用 LLM"""
        await self.acquire()
        try:
            return await func(*args, **kwargs)
        finally:
            self.release()


# ============================================================
# 3. 请求队列
# ============================================================

class RequestQueue:
    """
    请求队列
    
    异步处理 Agent 请求，支持：
    - 优先级队列
    - 请求去重
    - 超时处理
    """
    
    def __init__(self, max_size: int = 1000):
        self.max_size = max_size
        self._queue: asyncio.Queue = None
        self._processing = False
        self._handlers: List[Callable] = []
    
    async def initialize(self):
        """初始化异步队列"""
        self._queue = asyncio.Queue(maxsize=self.max_size)
    
    def register_handler(self, handler: Callable):
        """注册处理函数"""
        self._handlers.append(handler)
    
    async def enqueue(self, request: Dict, priority: int = 0):
        """入队"""
        if self._queue.full():
            raise Exception("队列已满")
        
        await self._queue.put({
            "request": request,
            "priority": priority,
            "timestamp": time.time(),
        })
    
    async def process(self):
        """处理队列中的请求"""
        self._processing = True
        
        while self._processing:
            try:
                item = await asyncio.wait_for(
                    self._queue.get(), timeout=1.0
                )
                
                # 调用所有处理函数
                for handler in self._handlers:
                    try:
                        await handler(item["request"])
                    except Exception as e:
                        print(f"处理错误: {e}")
                
                self._queue.task_done()
                
            except asyncio.TimeoutError:
                continue
            except Exception as e:
                print(f"队列处理错误: {e}")


# ============================================================
# 4. 负载均衡器
# ============================================================

class LoadBalancer:
    """
    负载均衡器
    
    将请求分配到不同的 Agent 实例。
    """
    
    def __init__(self):
        self.instances: Dict[str, Dict] = {}
        self._round_robin_index = 0
    
    def register_instance(self, instance_id: str, address: str):
        """注册实例"""
        self.instances[instance_id] = {
            "address": address,
            "active_requests": 0,
            "total_requests": 0,
            "last_health_check": time.time(),
            "healthy": True,
        }
    
    def get_instance(self, strategy: str = "round_robin") -> Optional[str]:
        """获取一个可用实例"""
        healthy_instances = [
            (k, v) for k, v in self.instances.items()
            if v["healthy"]
        ]
        
        if not healthy_instances:
            return None
        
        if strategy == "round_robin":
            idx = self._round_robin_index % len(healthy_instances)
            self._round_robin_index += 1
            return healthy_instances[idx][0]
        
        elif strategy == "least_connections":
            # 选择连接数最少的实例
            return min(healthy_instances, key=lambda x: x[1]["active_requests"])[0]
        
        elif strategy == "random":
            import random
            return random.choice(healthy_instances)[0]
        
        return healthy_instances[0][0]


# ============================================================
# 5. 缓存层
# ============================================================

class DistributedCache:
    """
    分布式缓存
    
    生产环境应该用 Redis 集群，这里用内存模拟。
    """
    
    def __init__(self):
        self._cache: Dict[str, Dict] = {}
        self._lock = threading.Lock()
    
    def get(self, key: str) -> Optional[Any]:
        """获取缓存"""
        with self._lock:
            entry = self._cache.get(key)
            if entry and time.time() < entry["expires"]:
                return entry["value"]
            elif entry:
                del self._cache[key]
        return None
    
    def set(self, key: str, value: Any, ttl: int = 300):
        """设置缓存"""
        with self._lock:
            self._cache[key] = {
                "value": value,
                "expires": time.time() + ttl,
            }
    
    def get_or_set(self, key: str, factory: Callable, ttl: int = 300) -> Any:
        """获取缓存，不存在则创建"""
        value = self.get(key)
        if value is not None:
            return value
        
        value = factory()
        self.set(key, value, ttl)
        return value


# ============================================================
# 6. 可扩展的 Agent
# ============================================================

class ScalableAgent:
    """
    可扩展的 Agent
    
    支持多实例部署、会话共享、请求队列和缓存。
    """
    
    def __init__(self, instance_id: str = "default"):
        self.instance_id = instance_id
        self.session_store = SessionStore()
        self.llm_pool = LLMConnectionPool(max_concurrent=10)
        self.cache = DistributedCache()
        self.load_balancer = LoadBalancer()
        self.request_count = 0
    
    async def process_request(self, session_id: str, 
                              user_input: str) -> Dict:
        """处理请求"""
        self.request_count += 1
        start_time = time.time()
        
        # 1. 获取或创建会话
        session = self.session_store.get(session_id)
        if not session:
            session = {
                "session_id": session_id,
                "messages": [],
                "created_at": time.time(),
            }
        
        # 2. 检查缓存
        cache_key = hashlib.md5(
            f"{session_id}:{user_input}".encode()
        ).hexdigest()
        
        cached_response = self.cache.get(cache_key)
        if cached_response:
            return {
                "response": cached_response,
                "source": "cache",
                "instance": self.instance_id,
            }
        
        # 3. 通过连接池调用 LLM
        messages = session["messages"] + [
            {"role": "user", "content": user_input}
        ]
        
        response = await self.llm_pool.call(
            self._mock_llm_call, messages
        )
        
        # 4. 更新会话
        session["messages"] = messages + [
            {"role": "assistant", "content": response}
        ]
        self.session_store.set(session_id, session)
        
        # 5. 缓存结果
        self.cache.set(cache_key, response, ttl=300)
        
        latency_ms = (time.time() - start_time) * 1000
        
        return {
            "response": response,
            "source": "llm",
            "instance": self.instance_id,
            "latency_ms": latency_ms,
        }
    
    async def _mock_llm_call(self, messages: list) -> str:
        """模拟 LLM 调用"""
        await asyncio.sleep(0.1)  # 模拟网络延迟
        last_msg = messages[-1]["content"] if messages else ""
        return f"Agent ({self.instance_id}) 回答: {last_msg[:50]}..."
    
    def get_stats(self) -> Dict:
        """获取实例统计"""
        return {
            "instance_id": self.instance_id,
            "request_count": self.request_count,
            "active_sessions": len(self.session_store._store),
        }


# ============================================================
# 使用示例
# ============================================================

async def demo_scalable_agent():
    """演示可扩展 Agent"""
    # 创建多个 Agent 实例
    agents = [
        ScalableAgent(f"instance_{i}")
        for i in range(3)
    ]
    
    # 创建负载均衡器
    lb = LoadBalancer()
    for agent in agents:
        lb.register_instance(agent.instance_id, f"localhost:800{i}")
    
    print("可扩展 Agent 测试")
    print("=" * 60)
    
    # 模拟并发请求
    async def handle_request(request_id: int):
        instance_id = lb.get_instance("round_robin")
        agent = next(a for a in agents if a.instance_id == instance_id)
        result = await agent.process_request(
            f"session_{request_id % 5}",  # 5 个会话
            f"问题 {request_id}"
        )
        return result
    
    # 并发执行 20 个请求
    tasks = [handle_request(i) for i in range(20)]
    results = await asyncio.gather(*tasks)
    
    # 打印结果统计
    source_counts = defaultdict(int)
    instance_counts = defaultdict(int)
    for r in results:
        source_counts[r["source"]] += 1
        instance_counts[r["instance"]] += 1
    
    print(f"\n结果来源分布: {dict(source_counts)}")
    print(f"实例处理分布: {dict(instance_counts)}")
    
    # 打印各实例统计
    for agent in agents:
        stats = agent.get_stats()
        print(f"\n{stats['instance_id']}: {stats['request_count']} 请求, {stats['active_sessions']} 活跃会话")


if __name__ == "__main__":
    asyncio.run(demo_scalable_agent())
```

---

## 43.3 案例分析

### 43.3.1 案例：从单机到集群的演进

**阶段一（日请求 < 1000）：** 单机部署，所有组件（Agent、数据库、缓存）运行在同一台服务器上。简单、低成本、易维护。

**阶段二（日请求 1000-10000）：** 分离组件。Agent API 和数据库分别部署在不同的服务器上。添加 Redis 作为会话缓存。使用 Nginx 做负载均衡，部署 2-3 个 Agent 实例。

**阶段三（日请求 > 10000）：** 完整的微服务架构。Agent API、工具服务、向量数据库、缓存层各自独立部署和扩展。使用 Kubernetes 管理容器编排。实现自动水平扩展（HPA）。

**关键教训：** 不要过早优化架构复杂度。在每个阶段选择恰好满足当前需求的架构，等到真正遇到瓶颈时再升级。

---

## 43.4 高级扩展技术

### 43.4.1 自动扩展策略

实现基于指标的自动扩展，根据负载动态调整实例数量。

```python
import time
import asyncio
from typing import Dict, List, Optional, Callable
from dataclasses import dataclass, field
from enum import Enum

class ScalingDirection(Enum):
    """扩展方向"""
    UP = "up"      # 扩容
    DOWN = "down"  # 缩容
    NONE = "none"  # 不变

@dataclass
class ScalingConfig:
    """扩展配置"""
    min_instances: int = 1
    max_instances: int = 10
    target_cpu_percent: float = 70.0
    target_memory_percent: float = 80.0
    target_request_per_instance: int = 100
    scale_up_threshold: float = 0.8  # 超过目标的 80% 时扩容
    scale_down_threshold: float = 0.5  # 低于目标的 50% 时缩容
    cooldown_seconds: int = 300  # 冷却时间 5 分钟

class AutoScaler:
    """
    自动扩展器
    
    根据监控指标自动调整实例数量。
    """
    
    def __init__(self, config: ScalingConfig = None):
        self.config = config or ScalingConfig()
        self.instances: Dict[str, Dict] = {}
        self.scaling_history: List[Dict] = []
        self._last_scaling_time: float = 0
    
    def register_instance(self, instance_id: str, metadata: Dict = None):
        """注册实例"""
        self.instances[instance_id] = {
            "id": instance_id,
            "created_at": time.time(),
            "status": "running",
            "metadata": metadata or {},
        }
    
    def remove_instance(self, instance_id: str):
        """移除实例"""
        if instance_id in self.instances:
            self.instances[instance_id]["status"] = "terminated"
            del self.instances[instance_id]
    
    def evaluate_scaling(self, metrics: Dict) -> Dict:
        """
        评估是否需要扩展
        
        Args:
            metrics: 当前指标
                - cpu_percent: CPU 使用率
                - memory_percent: 内存使用率
                - requests_per_second: 每秒请求数
                - active_connections: 活跃连接数
                
        Returns:
            扩展决策
        """
        current_instances = len([i for i in self.instances.values() 
                               if i["status"] == "running"])
        
        # 检查冷却时间
        if time.time() - self._last_scaling_time < self.config.cooldown_seconds:
            return {
                "direction": ScalingDirection.NONE,
                "reason": "冷却时间内",
                "current_instances": current_instances,
            }
        
        # 计算负载指标
        cpu_percent = metrics.get("cpu_percent", 0)
        memory_percent = metrics.get("memory_percent", 0)
        rps = metrics.get("requests_per_second", 0)
        
        # 计算每个实例的负载
        rps_per_instance = rps / max(current_instances, 1)
        
        # 判断扩展方向
        direction = ScalingDirection.NONE
        reason = "负载正常"
        
        # 检查是否需要扩容
        if (cpu_percent > self.config.target_cpu_percent * self.config.scale_up_threshold or
            rps_per_instance > self.config.target_request_per_instance * self.config.scale_up_threshold):
            direction = ScalingDirection.UP
            reason = f"负载过高 (CPU: {cpu_percent:.1f}%, RPS/实例: {rps_per_instance:.0f})"
        
        # 检查是否需要缩容
        elif (cpu_percent < self.config.target_cpu_percent * self.config.scale_down_threshold and
              rps_per_instance < self.config.target_request_per_instance * self.config.scale_down_threshold and
              current_instances > self.config.min_instances):
            direction = ScalingDirection.DOWN
            reason = f"负载较低 (CPU: {cpu_percent:.1f}%, RPS/实例: {rps_per_instance:.0f})"
        
        # 计算目标实例数
        target_instances = current_instances
        if direction == ScalingDirection.UP:
            # 根据负载计算需要的实例数
            target_by_cpu = int(cpu_percent / self.config.target_cpu_percent * current_instances) + 1
            target_by_rps = int(rps / self.config.target_request_per_instance) + 1
            target_instances = min(max(target_by_cpu, target_by_rps), self.config.max_instances)
        
        elif direction == ScalingDirection.DOWN:
            target_instances = max(current_instances - 1, self.config.min_instances)
        
        result = {
            "direction": direction,
            "reason": reason,
            "current_instances": current_instances,
            "target_instances": target_instances,
            "scaling_needed": target_instances != current_instances,
        }
        
        # 记录扩展决策
        self.scaling_history.append({
            "timestamp": time.time(),
            "metrics": metrics,
            "decision": result,
        })
        
        return result
    
    def execute_scaling(self, decision: Dict, 
                       create_instance: Callable = None,
                       terminate_instance: Callable = None) -> bool:
        """
        执行扩展决策
        
        Args:
            decision: evaluate_scaling 的返回值
            create_instance: 创建实例的回调函数
            terminate_instance: 终止实例的回调函数
            
        Returns:
            是否成功执行
        """
        if not decision.get("scaling_needed"):
            return True
        
        current = decision["current_instances"]
        target = decision["target_instances"]
        direction = decision["direction"]
        
        try:
            if direction == ScalingDirection.UP:
                # 扩容
                instances_to_add = target - current
                for i in range(instances_to_add):
                    instance_id = f"instance_{int(time.time())}_{i}"
                    if create_instance:
                        create_instance(instance_id)
                    self.register_instance(instance_id)
                    print(f"  创建实例: {instance_id}")
            
            elif direction == ScalingDirection.DOWN:
                # 缩容
                instances_to_remove = current - target
                running_instances = [i for i in self.instances.values() 
                                   if i["status"] == "running"]
                
                # 选择最旧的实例进行终止
                running_instances.sort(key=lambda x: x["created_at"])
                
                for i in range(min(instances_to_remove, len(running_instances))):
                    instance = running_instances[i]
                    if terminate_instance:
                        terminate_instance(instance["id"])
                    self.remove_instance(instance["id"])
                    print(f"  终止实例: {instance['id']}")
            
            self._last_scaling_time = time.time()
            return True
            
        except Exception as e:
            print(f"扩展执行失败: {e}")
            return False
    
    def get_scaling_stats(self) -> Dict:
        """获取扩展统计"""
        running = len([i for i in self.instances.values() if i["status"] == "running"])
        terminated = len([i for i in self.instances.values() if i["status"] == "terminated"])
        
        scale_ups = len([h for h in self.scaling_history 
                        if h["decision"]["direction"] == ScalingDirection.UP])
        scale_downs = len([h for h in self.scaling_history 
                          if h["decision"]["direction"] == ScalingDirection.DOWN])
        
        return {
            "current_instances": running,
            "terminated_instances": terminated,
            "total_scaling_events": len(self.scaling_history),
            "scale_ups": scale_ups,
            "scale_downs": scale_downs,
        }


# ============================================================
# 使用示例
# ============================================================

def demo_auto_scaling():
    """演示自动扩展"""
    config = ScalingConfig(
        min_instances=2,
        max_instances=8,
        target_cpu_percent=70,
        target_request_per_instance=100,
    )
    
    scaler = AutoScaler(config)
    
    # 初始实例
    for i in range(2):
        scaler.register_instance(f"instance_{i}")
    
    print("自动扩展测试")
    print("=" * 60)
    
    # 模拟负载变化
    scenarios = [
        {"cpu_percent": 30, "requests_per_second": 50, "description": "低负载"},
        {"cpu_percent": 85, "requests_per_second": 250, "description": "高负载"},
        {"cpu_percent": 40, "requests_per_second": 80, "description": "负载降低"},
        {"cpu_percent": 90, "requests_per_second": 400, "description": "非常高负载"},
    ]
    
    for scenario in scenarios:
        print(f"\n场景: {scenario['description']}")
        print(f"  CPU: {scenario['cpu_percent']}%, RPS: {scenario['requests_per_second']}")
        
        decision = scaler.evaluate_scaling(scenario)
        print(f"  决策: {decision['direction'].value}")
        print(f"  原因: {decision['reason']}")
        print(f"  当前实例: {decision['current_instances']}, 目标: {decision['target_instances']}")
        
        if decision["scaling_needed"]:
            # 模拟实例创建/终止
            scaler.execute_scaling(decision)
        
        stats = scaler.get_scaling_stats()
        print(f"  实例数: {stats['current_instances']}")
        
        time.sleep(0.1)  # 避免冷却时间冲突
    
    print("\n扩展统计:")
    stats = scaler.get_scaling_stats()
    print(f"  总扩展次数: {stats['total_scaling_events']}")
    print(f"  扩容次数: {stats['scale_ups']}")
    print(f"  缩容次数: {stats['scale_downs']}")


if __name__ == "__main__":
    demo_auto_scaling()
```

### 43.4.2 会话亲和性

确保同一用户的请求被路由到同一实例，减少状态同步开销。

```python
import hashlib
from typing import Dict, List, Optional
from dataclasses import dataclass

@dataclass
class SessionAffinityConfig:
    """会话亲和性配置"""
    method: str = "consistent_hash"  # consistent_hash / sticky / round_robin
    hash_salt: str = ""  # 哈希盐值

class SessionAffinityRouter:
    """
    会话亲和性路由器
    
    将同一用户的请求路由到同一实例，减少会话状态同步。
    """
    
    def __init__(self, config: SessionAffinityConfig = None):
        self.config = config or SessionAffinityConfig()
        self.instances: Dict[str, Dict] = {}
        self._hash_ring: List[str] = []  # 一致性哈希环
    
    def register_instance(self, instance_id: str, address: str,
                         weight: int = 1):
        """注册实例"""
        self.instances[instance_id] = {
            "id": instance_id,
            "address": address,
            "weight": weight,
        }
        
        # 更新哈希环
        self._rebuild_hash_ring()
    
    def remove_instance(self, instance_id: str):
        """移除实例"""
        if instance_id in self.instances:
            del self.instances[instance_id]
            self._rebuild_hash_ring()
    
    def _rebuild_hash_ring(self):
        """重建一致性哈希环"""
        self._hash_ring = []
        for instance_id, info in self.instances.items():
            # 每个实例根据权重创建多个虚拟节点
            for i in range(info["weight"] * 10):
                key = f"{instance_id}:{i}"
                hash_val = self._hash(key)
                self._hash_ring.append((hash_val, instance_id))
        
        self._hash_ring.sort(key=lambda x: x[0])
    
    def _hash(self, key: str) -> int:
        """计算哈希值"""
        content = f"{self.config.hash_salt}:{key}".encode()
        return int(hashlib.md5(content).hexdigest(), 16)
    
    def route(self, user_id: str) -> Optional[str]:
        """
        根据用户 ID 路由到实例
        
        Args:
            user_id: 用户 ID
            
        Returns:
            实例 ID
        """
        if not self._hash_ring:
            return None
        
        if self.config.method == "consistent_hash":
            return self._route_consistent_hash(user_id)
        elif self.config.method == "sticky":
            return self._route_sticky(user_id)
        else:
            return self._route_round_robin(user_id)
    
    def _route_consistent_hash(self, user_id: str) -> str:
        """一致性哈希路由"""
        user_hash = self._hash(user_id)
        
        # 在哈希环上找到第一个大于等于用户哈希的节点
        for hash_val, instance_id in self._hash_ring:
            if hash_val >= user_hash:
                return instance_id
        
        # 如果没找到，回到环的起点
        return self._hash_ring[0][1]
    
    def _route_sticky(self, user_id: str) -> str:
        """粘性会话路由"""
        # 使用简单的模运算
        instance_ids = list(self.instances.keys())
        if not instance_ids:
            return None
        
        index = self._hash(user_id) % len(instance_ids)
        return instance_ids[index]
    
    _round_robin_index = 0
    
    def _route_round_robin(self, user_id: str) -> str:
        """轮询路由（忽略 user_id）"""
        instance_ids = list(self.instances.keys())
        if not instance_ids:
            return None
        
        index = self._round_robin_index % len(instance_ids)
        self._round_robin_index += 1
        return instance_ids[index]
    
    def get_routing_distribution(self, sample_users: List[str]) -> Dict:
        """分析路由分布"""
        distribution = {}
        
        for user_id in sample_users:
            instance_id = self.route(user_id)
            if instance_id:
                distribution[instance_id] = distribution.get(instance_id, 0) + 1
        
        total = len(sample_users)
        return {
            "distribution": distribution,
            "percentages": {k: v / total * 100 for k, v in distribution.items()},
            "balance_score": self._calculate_balance(distribution),
        }
    
    def _calculate_balance(self, distribution: Dict) -> float:
        """计算负载均衡分数"""
        if not distribution:
            return 0.0
        
        values = list(distribution.values())
        avg = sum(values) / len(values)
        
        if avg == 0:
            return 1.0
        
        variance = sum((v - avg) ** 2 for v in values) / len(values)
        std_dev = variance ** 0.5
        
        # 标准差越小，均衡性越好
        return max(0, 1 - std_dev / avg)


# ============================================================
# 使用示例
# ============================================================

def demo_session_affinity():
    """演示会话亲和性"""
    # 创建路由器
    config = SessionAffinityConfig(method="consistent_hash")
    router = SessionAffinityRouter(config)
    
    # 注册实例
    router.register_instance("instance_1", "10.0.0.1", weight=2)
    router.register_instance("instance_2", "10.0.0.2", weight=1)
    router.register_instance("instance_3", "10.0.0.3", weight=1)
    
    print("会话亲和性路由测试")
    print("=" * 60)
    
    # 测试路由一致性
    test_users = [f"user_{i}" for i in range(100)]
    
    print("\n路由一致性测试:")
    for user_id in test_users[:5]:
        routes = [router.route(user_id) for _ in range(3)]
        consistent = len(set(routes)) == 1
        print(f"  {user_id}: {routes[0]} (一致: {consistent})")
    
    # 分析路由分布
    print("\n路由分布分析:")
    distribution = router.get_routing_distribution(test_users)
    
    for instance_id, count in distribution["distribution"].items():
        percentage = distribution["percentages"][instance_id]
        print(f"  {instance_id}: {count} 用户 ({percentage:.1f}%)")
    
    print(f"\n负载均衡分数: {distribution['balance_score']:.2f}")
    
    # 测试实例移除后的影响
    print("\n移除 instance_2 后的路由变化:")
    router.remove_instance("instance_2")
    
    changed_count = 0
    for user_id in test_users:
        # 这里简化演示，实际应该保存之前的路由结果
    
    print(f"  路由重新分配完成")


if __name__ == "__main__":
    demo_session_affinity()
```

### 43.4.3 限流与降级

实现智能的限流和降级策略，保护系统在高负载下仍然可用。

```python
import time
import asyncio
from typing import Dict, List, Optional, Callable, Any
from dataclasses import dataclass
from enum import Enum
from collections import deque
import threading

class RateLimitStrategy(Enum):
    """限流策略"""
    FIXED_WINDOW = "fixed_window"
    SLIDING_WINDOW = "sliding_window"
    TOKEN_BUCKET = "token_bucket"
    LEAKY_BUCKET = "leaky_bucket"

@dataclass
class RateLimitConfig:
    """限流配置"""
    strategy: RateLimitStrategy = RateLimitStrategy.SLIDING_WINDOW
    requests_per_minute: int = 100
    burst_size: int = 20  # 突发请求大小
    window_seconds: int = 60

class AdaptiveRateLimiter:
    """
    自适应限流器
    
    根据系统负载动态调整限流阈值。
    """
    
    def __init__(self, config: RateLimitConfig = None):
        self.config = config or RateLimitConfig()
        self._request_times: deque = deque()
        self._lock = threading.Lock()
        self._current_limit = self.config.requests_per_minute
        self._load_factor = 1.0  # 负载因子
    
    def allow_request(self) -> Dict:
        """
        检查是否允许请求
        
        Returns:
            {
                "allowed": bool,
                "remaining": int,
                "retry_after": float,  # 秒
            }
        """
        with self._lock:
            now = time.time()
            window_start = now - self.config.window_seconds
            
            # 清理过期记录
            while self._request_times and self._request_times[0] < window_start:
                self._request_times.popleft()
            
            current_count = len(self._request_times)
            effective_limit = int(self._current_limit * self._load_factor)
            
            if current_count >= effective_limit:
                # 计算需要等待的时间
                oldest = self._request_times[0]
                retry_after = oldest + self.config.window_seconds - now
                
                return {
                    "allowed": False,
                    "remaining": 0,
                    "retry_after": max(0, retry_after),
                    "limit": effective_limit,
                    "current": current_count,
                }
            
            # 允许请求
            self._request_times.append(now)
            
            return {
                "allowed": True,
                "remaining": effective_limit - current_count - 1,
                "retry_after": 0,
                "limit": effective_limit,
                "current": current_count + 1,
            }
    
    def update_load_factor(self, cpu_percent: float, 
                          error_rate: float):
        """
        根据系统负载更新负载因子
        
        当系统负载高时，自动降低允许的请求量。
        """
        # CPU 负载影响
        if cpu_percent > 80:
            cpu_factor = 0.5
        elif cpu_percent > 60:
            cpu_factor = 0.75
        else:
            cpu_factor = 1.0
        
        # 错误率影响
        if error_rate > 0.1:
            error_factor = 0.5
        elif error_rate > 0.05:
            error_factor = 0.75
        else:
            error_factor = 1.0
        
        self._load_factor = cpu_factor * error_factor
        
        print(f"  负载因子更新: CPU={cpu_percent:.1f}%, 错误率={error_rate:.2%}, "
              f"因子={self._load_factor:.2f}")
    
    def get_stats(self) -> Dict:
        """获取限流统计"""
        with self._lock:
            now = time.time()
            window_start = now - self.config.window_seconds
            
            while self._request_times and self._request_times[0] < window_start:
                self._request_times.popleft()
            
            return {
                "current_count": len(self._request_times),
                "limit": int(self._current_limit * self._load_factor),
                "load_factor": self._load_factor,
                "window_seconds": self.config.window_seconds,
            }


class DegradationManager:
    """
    降级管理器
    
    在系统负载过高时，自动降级非核心功能。
    """
    
    def __init__(self):
        self.degradation_levels = {
            0: "正常",       # 全功能
            1: "轻度降级",   # 关闭非核心功能
            2: "中度降级",   # 使用缓存替代实时计算
            3: "重度降级",   # 只保留核心功能
            4: "紧急模式",   # 最小功能集
        }
        
        self.current_level = 0
        self.degraded_features: Dict[int, List[str]] = {
            1: ["analytics", "recommendations"],
            2: ["real_time_search", "personalization"],
            3: ["advanced_tools", "multi_turn"],
            4: ["all_except_basic_qa"],
        }
    
    def evaluate_degradation_level(self, metrics: Dict) -> int:
        """
        评估应该处于哪个降级级别
        
        Args:
            metrics: 系统指标
            
        Returns:
            降级级别 (0-4)
        """
        cpu = metrics.get("cpu_percent", 0)
        memory = metrics.get("memory_percent", 0)
        error_rate = metrics.get("error_rate", 0)
        latency_p99 = metrics.get("latency_p99_ms", 0)
        
        # 根据指标判断降级级别
        if cpu > 95 or memory > 95 or error_rate > 0.2:
            return 4  # 紧急模式
        elif cpu > 85 or memory > 85 or error_rate > 0.1:
            return 3  # 重度降级
        elif cpu > 70 or memory > 75 or error_rate > 0.05:
            return 2  # 中度降级
        elif cpu > 50 or latency_p99 > 5000:
            return 1  # 轻度降级
        else:
            return 0  # 正常
    
    def update_degradation_level(self, metrics: Dict) -> Dict:
        """更新降级级别"""
        new_level = self.evaluate_degradation_level(metrics)
        old_level = self.current_level
        
        if new_level != self.current_level:
            self.current_level = new_level
            print(f"降级级别变更: {old_level} -> {new_level} ({self.degradation_levels[new_level]})")
        
        return {
            "level": self.current_level,
            "name": self.degradation_levels[self.current_level],
            "degraded_features": self.degraded_features.get(self.current_level, []),
        }
    
    def is_feature_available(self, feature: str) -> bool:
        """检查功能是否可用"""
        degraded = self.degraded_features.get(self.current_level, [])
        
        for level in range(self.current_level + 1):
            if feature in self.degraded_features.get(level, []):
                return False
        
        return True
    
    def get_degradation_status(self) -> Dict:
        """获取降级状态"""
        return {
            "level": self.current_level,
            "name": self.degradation_levels[self.current_level],
            "available_features": [
                f for f in ["basic_qa", "search", "tools", "analytics", "personalization"]
                if self.is_feature_available(f)
            ],
            "unavailable_features": [
                f for f in ["basic_qa", "search", "tools", "analytics", "personalization"]
                if not self.is_feature_available(f)
            ],
        }


# ============================================================
# 使用示例
# ============================================================

def demo_rate_limiting_and_degradation():
    """演示限流和降级"""
    # 创建限流器
    limiter = AdaptiveRateLimiter(RateLimitConfig(
        requests_per_minute=100,
    ))
    
    # 创建降级管理器
    degradation = DegradationManager()
    
    print("限流和降级测试")
    print("=" * 60)
    
    # 模拟正常负载
    print("\n正常负载:")
    for _ in range(5):
        result = limiter.allow_request()
        print(f"  请求: {'允许' if result['allowed'] else '拒绝'} "
              f"(剩余: {result['remaining']})")
    
    # 模拟高负载
    print("\n更新负载指标 (高 CPU):")
    degradation.update_degradation_level({"cpu_percent": 85, "error_rate": 0.02})
    limiter.update_load_factor(cpu_percent=85, error_rate=0.02)
    
    print("\n高负载下的限流:")
    stats = limiter.get_stats()
    print(f"  当前限制: {stats['limit']} 请求/分钟")
    print(f"  负载因子: {stats['load_factor']:.2f}")
    
    # 检查功能可用性
    print("\n功能可用性:")
    status = degradation.get_degradation_status()
    print(f"  降级级别: {status['level']} ({status['name']})")
    print(f"  可用功能: {status['available_features']}")
    print(f"  不可用功能: {status['unavailable_features']}")


if __name__ == "__main__":
    demo_rate_limiting_and_degradation()
```

## 43.5 常见坑与最佳实践

**坑 1：会话状态丢失。** 多实例部署时，如果会话存在单个实例的内存中，用户下次请求被路由到不同实例时就会丢失对话历史。使用 Redis 等外部存储来共享会话状态。

**坑 2：缓存一致性问题。** 多实例各自的本地缓存可能不同步。使用分布式缓存（如 Redis）或在更新时广播失效消息。

**坑 3：LLM API 限流。** 所有实例共享同一个 API Key，速率限制是全局的。使用请求队列来控制并发，避免触发限流。

**坑 4：扩展过于激进。** 频繁的扩容缩容会导致系统不稳定。设置合理的冷却时间和扩展阈值。

**坑 5：降级策略不清晰。** 没有明确定义在什么情况下应该降级哪些功能。提前制定降级策略，确保在紧急情况下能快速响应。

**最佳实践 1：设计无状态的 Agent。** 尽量将状态外置到 Redis 或数据库，让 Agent 实例本身是无状态的。这样可以随意增减实例，扩展和回滚都更简单。

**最佳实践 2：使用一致性哈希。** 一致性哈希可以在实例变化时最小化路由调整，减少会话迁移的开销。

**最佳实践 3：实现优雅降级。** 在系统负载过高时，优先保证核心功能可用，逐步降级非核心功能。

**最佳实践 4：定期进行压力测试。** 了解系统的极限在哪里，才能制定合理的扩展策略和降级阈值。

---

## 43.5 练习题

**练习 1：实现 Redis 会话存储。** 将 SessionStore 替换为基于 Redis 的实现，支持 TTL 自动过期。

**练习 2：实现自动扩展策略。** 编写一个自动扩展控制器：当 CPU 使用率 > 70% 时增加实例，< 30% 时减少实例。

**练习 3：设计 LLM 限流策略。** 实现一个令牌桶算法来控制 LLM API 的调用速率。

**练习 4：实现健康检查。** 为 Agent 实例添加 /health 端点，返回实例的健康状态和负载信息。

**练习 5：设计故障转移方案。** 当一个 Agent 实例不可用时，自动将请求路由到其他健康实例。

**练习 6：实现分布式追踪。** 在多实例环境下，确保同一请求的所有步骤能通过 request_id 关联起来。

---

## 43.6 本章小结

本章系统性地探讨了 Agent 的可扩展性设计。从扩展瓶颈分析到水平扩展架构，从会话管理到连接池，我们建立了一个完整的可扩展 Agent 架构知识体系。

核心理念是：扩展性是设计出来的，不是事后加上去的。在设计 Agent 架构时就考虑未来的扩展需求，但不要过度设计——选择恰好满足当前和近期需求的架构。

---

> **下一章预告：** 第 44 章将深入 Workflow vs Agent 的选型决策，帮你判断什么场景该用确定性工作流，什么场景该用灵活的 Agent。
