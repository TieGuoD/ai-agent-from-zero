# 第 37 章：Agent 日志与审计 —— 合规与追溯

> **本章定位：** 在企业环境中，Agent 不仅要"能工作"，还要"可审计"。日志与审计系统是满足合规要求、支持事后分析、追踪问题根因的关键基础设施。本章将从合规需求出发，构建一个完整的 Agent 日志与审计体系。

---

## 学习目标

完成本章学习后，你将能够：

1. **理解 Agent 审计的合规要求** —— 能说明 GDPR、SOC 2 等法规对 Agent 日志的具体要求
2. **设计完整的审计日志架构** —— 能为 Agent 系统设计覆盖全链路的审计日志方案
3. **实现不可篡改的审计追踪** —— 能用哈希链等技术确保审计日志的完整性
4. **实现日志的生命周期管理** —— 能设计日志的保留策略、归档和安全删除方案
5. **构建审计查询和分析能力** —— 能高效查询和分析历史审计数据

## 核心问题

1. **Agent 审计日志和传统应用日志有什么不同？** 传统应用记录的是确定性的操作，Agent 的日志还需要记录概率性的推理过程——这如何影响日志设计？
2. **审计日志的完整性如何保证？** 如果攻击者能篡改审计日志，审计就失去了意义。
3. **日志保留多久合适？** 太短不满足合规，太长增加成本和风险——如何找到平衡点？

---

## 37.1 合规需求与审计目标

### 37.1.1 为什么需要审计

想象这样一个场景：你的客服 Agent 在一次对话中给了用户错误的退款金额，导致公司损失了 5000 元。管理层要求你解释：Agent 为什么给出了这个错误答案？是谁触发的？Agent 的推理过程是什么？当时使用了哪些工具？

如果你没有完整的审计日志，这些问题都无法回答。审计日志不仅是合规的要求，更是建立用户信任和改进系统的基础。

### 37.1.2 常见合规框架

**GDPR（通用数据保护条例）。** 如果你的 Agent 处理欧盟用户的数据，GDPR 要求你记录数据处理的完整过程，包括谁访问了什么数据、用于什么目的。用户有权要求查看关于他们的所有处理记录。

**SOC 2（服务组织控制 2）。** SOC 2 审计关注五个信任原则：安全性、可用性、处理完整性、机密性和隐私。对于 Agent 系统，重点是安全性和处理完整性——确保 Agent 的行为是受控的、可追溯的。

**行业特定法规。** 金融行业有 PCI DSS（支付卡行业数据安全标准），医疗行业有 HIPAA（健康保险可携性和责任法案）。这些法规对日志有更具体的要求。

### 37.1.3 Agent 审计的特殊挑战

Agent 的审计比传统应用更复杂，因为：

**推理过程是非确定性的。** 传统应用的操作日志是确定性的——同样的输入产生同样的操作。Agent 的推理过程可能因 temperature、上下文等因素而不同。

**决策链路更长。** 一个 Agent 请求可能涉及多轮 LLM 推理、多次工具调用、多个中间状态。每一步都需要记录。

**数据流更复杂。** Agent 可能从用户输入、知识库、外部 API 等多个来源获取数据，然后将处理结果返回给用户或写入存储。数据的流向需要完整追踪。

**隐私风险更高。** Agent 处理的是自然语言交互，其中可能包含大量个人隐私信息。日志中如何处理这些隐私数据是一个重要挑战。

---

## 37.2 审计日志架构设计

### 37.2.1 日志层次模型

```python
import os
import json
import time
import hashlib
import uuid
from datetime import datetime, timedelta
from typing import Dict, List, Any, Optional
from dataclasses import dataclass, field, asdict
from enum import Enum
from pathlib import Path
from collections import deque

class AuditLevel(Enum):
    """审计级别"""
    TRACE = "trace"      # 最详细，每一步操作
    DEBUG = "debug"      # 调试信息
    INFO = "info"        # 正常操作
    WARNING = "warning"  # 异常但可恢复
    ERROR = "error"      # 错误
    CRITICAL = "critical"  # 严重安全事件

class AuditCategory(Enum):
    """审计类别"""
    AUTH = "authentication"       # 认证相关
    ACCESS = "access_control"     # 访问控制
    LLM_CALL = "llm_interaction"  # LLM 调用
    TOOL_CALL = "tool_execution"  # 工具调用
    DATA_OP = "data_operation"    # 数据操作
    SECURITY = "security_event"   # 安全事件
    SYSTEM = "system_event"       # 系统事件
    USER_ACTION = "user_action"   # 用户操作

@dataclass
class AuditEntry:
    """审计日志条目"""
    entry_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    timestamp: str = field(default_factory=lambda: datetime.utcnow().isoformat() + "Z")
    level: AuditLevel = AuditLevel.INFO
    category: AuditCategory = AuditCategory.SYSTEM
    event_type: str = ""
    user_id: str = ""
    session_id: str = ""
    request_id: str = ""
    source_ip: str = ""
    
    # 操作详情
    action: str = ""
    resource: str = ""
    outcome: str = "success"  # success, failure, denied
    
    # 上下文
    details: Dict[str, Any] = field(default_factory=dict)
    
    # 数据流追踪
    data_sources: List[str] = field(default_factory=list)
    data_destinations: List[str] = field(default_factory=list)
    
    # 完整性校验
    previous_hash: str = ""
    entry_hash: str = ""
    
    def compute_hash(self) -> str:
        """计算条目的哈希值"""
        content = json.dumps({
            "entry_id": self.entry_id,
            "timestamp": self.timestamp,
            "level": self.level.value,
            "category": self.category.value,
            "event_type": self.event_type,
            "user_id": self.user_id,
            "action": self.action,
            "resource": self.resource,
            "outcome": self.outcome,
            "previous_hash": self.previous_hash,
        }, sort_keys=True)
        return hashlib.sha256(content.encode()).hexdigest()


# ============================================================
# 审计日志管理器
# ============================================================

class AuditLogger:
    """
    Agent 审计日志管理器
    
    提供完整的审计日志功能：
    - 结构化日志记录
    - 哈希链保证完整性
    - 多种输出目标
    - 日志查询和分析
    """
    
    def __init__(self, log_dir: str = "./audit_logs",
                 retention_days: int = 90):
        self.log_dir = Path(log_dir)
        self.log_dir.mkdir(parents=True, exist_ok=True)
        self.retention_days = retention_days
        self._previous_hash = "GENESIS"
        self._buffer: List[AuditEntry] = []
        self._buffer_size = 100  # 缓冲区满后批量写入
    
    def log(self, level: AuditLevel, category: AuditCategory,
            event_type: str, action: str, resource: str,
            user_id: str = "", session_id: str = "",
            request_id: str = "", source_ip: str = "",
            outcome: str = "success", details: Dict = None,
            data_sources: List[str] = None,
            data_destinations: List[str] = None):
        """记录审计日志"""
        entry = AuditEntry(
            level=level,
            category=category,
            event_type=event_type,
            user_id=user_id,
            session_id=session_id,
            request_id=request_id,
            source_ip=source_ip,
            action=action,
            resource=resource,
            outcome=outcome,
            details=details or {},
            data_sources=data_sources or [],
            data_destinations=data_destinations or [],
            previous_hash=self._previous_hash,
        )
        
        # 计算哈希
        entry.entry_hash = entry.compute_hash()
        self._previous_hash = entry.entry_hash
        
        # 添加到缓冲区
        self._buffer.append(entry)
        
        if len(self._buffer) >= self._buffer_size:
            self._flush_buffer()
        
        return entry
    
    def _flush_buffer(self):
        """将缓冲区写入文件"""
        if not self._buffer:
            return
        
        today = datetime.utcnow().strftime("%Y-%m-%d")
        log_file = self.log_dir / f"audit_{today}.jsonl"
        
        with open(log_file, "a", encoding="utf-8") as f:
            for entry in self._buffer:
                f.write(json.dumps(asdict(entry), ensure_ascii=False, default=str) + "\n")
        
        self._buffer.clear()
    
    def flush(self):
        """强制刷新缓冲区"""
        self._flush_buffer()
    
    def query(self, start_time: str = None, end_time: str = None,
              user_id: str = None, category: AuditCategory = None,
              level: AuditLevel = None, event_type: str = None,
              limit: int = 100) -> List[Dict]:
        """查询审计日志"""
        results = []
        
        # 扫描所有日志文件
        for log_file in sorted(self.log_dir.glob("audit_*.jsonl"), reverse=True):
            with open(log_file, "r", encoding="utf-8") as f:
                for line in f:
                    entry = json.loads(line.strip())
                    
                    # 应用过滤条件
                    if start_time and entry["timestamp"] < start_time:
                        continue
                    if end_time and entry["timestamp"] > end_time:
                        continue
                    if user_id and entry.get("user_id") != user_id:
                        continue
                    if category and entry.get("category") != category.value:
                        continue
                    if level and entry.get("level") != level.value:
                        continue
                    if event_type and entry.get("event_type") != event_type:
                        continue
                    
                    results.append(entry)
                    
                    if len(results) >= limit:
                        return results
        
        return results
    
    def verify_integrity(self) -> Dict:
        """验证审计日志的完整性"""
        total_entries = 0
        broken_chains = 0
        
        previous_hash = "GENESIS"
        
        for log_file in sorted(self.log_dir.glob("audit_*.jsonl")):
            with open(log_file, "r", encoding="utf-8") as f:
                for line in f:
                    total_entries += 1
                    entry = json.loads(line.strip())
                    
                    # 验证哈希链
                    if entry.get("previous_hash") != previous_hash:
                        broken_chains += 1
                    
                    previous_hash = entry.get("entry_hash", "")
        
        return {
            "total_entries": total_entries,
            "broken_chains": broken_chains,
            "integrity_ok": broken_chains == 0,
        }
    
    def get_statistics(self) -> Dict:
        """获取审计日志统计"""
        stats = {
            "total_entries": 0,
            "by_level": {},
            "by_category": {},
            "by_outcome": {},
            "unique_users": set(),
        }
        
        for log_file in self.log_dir.glob("audit_*.jsonl"):
            with open(log_file, "r", encoding="utf-8") as f:
                for line in f:
                    entry = json.loads(line.strip())
                    stats["total_entries"] += 1
                    
                    level = entry.get("level", "unknown")
                    stats["by_level"][level] = stats["by_level"].get(level, 0) + 1
                    
                    category = entry.get("category", "unknown")
                    stats["by_category"][category] = stats["by_category"].get(category, 0) + 1
                    
                    outcome = entry.get("outcome", "unknown")
                    stats["by_outcome"][outcome] = stats["by_outcome"].get(outcome, 0) + 1
                    
                    if entry.get("user_id"):
                        stats["unique_users"].add(entry["user_id"])
        
        stats["unique_users"] = len(stats["unique_users"])
        return stats
    
    def cleanup_old_logs(self):
        """清理过期的日志"""
        cutoff = datetime.utcnow() - timedelta(days=self.retention_days)
        
        for log_file in self.log_dir.glob("audit_*.jsonl"):
            # 从文件名提取日期
            try:
                date_str = log_file.stem.replace("audit_", "")
                file_date = datetime.strptime(date_str, "%Y-%m-%d")
                if file_date < cutoff:
                    # 归档后删除
                    archive_file = self.log_dir / "archive" / log_file.name
                    archive_file.parent.mkdir(exist_ok=True)
                    log_file.rename(archive_file)
                    print(f"已归档: {log_file.name}")
            except (ValueError, OSError):
                pass
```

### 37.2.2 Agent 集成审计

```python
import os
import time
import json
from typing import Dict, Any, List

class AuditableAgent:
    """
    带完整审计功能的 Agent
    
    每个操作都会生成审计日志，支持：
    - 用户会话追踪
    - LLM 调用审计
    - 工具调用审计
    - 数据流向追踪
    - 安全事件记录
    """
    
    def __init__(self, audit_logger: AuditLogger):
        self.audit = audit_logger
        self.session_users: Dict[str, str] = {}  # session_id -> user_id
    
    def authenticate_user(self, user_id: str, session_id: str, 
                          method: str = "token") -> bool:
        """用户认证"""
        # 记录认证事件
        self.audit.log(
            level=AuditLevel.INFO,
            category=AuditCategory.AUTH,
            event_type="login_attempt",
            action="authenticate",
            resource="session",
            user_id=user_id,
            session_id=session_id,
            details={"method": method},
            outcome="success",
        )
        
        self.session_users[session_id] = user_id
        return True
    
    def process_request(self, user_input: str, session_id: str,
                        source_ip: str = "") -> Dict:
        """处理用户请求（带审计）"""
        user_id = self.session_users.get(session_id, "anonymous")
        request_id = str(uuid.uuid4())[:8]
        
        # 记录请求开始
        self.audit.log(
            level=AuditLevel.INFO,
            category=AuditCategory.USER_ACTION,
            event_type="request_start",
            action="process_input",
            resource="agent",
            user_id=user_id,
            session_id=session_id,
            request_id=request_id,
            source_ip=source_ip,
            details={"input_length": len(user_input)},
        )
        
        start_time = time.time()
        
        try:
            # 模拟 LLM 调用
            llm_response = self._call_llm(user_input, request_id)
            
            # 记录 LLM 调用
            self.audit.log(
                level=AuditLevel.INFO,
                category=AuditCategory.LLM_CALL,
                event_type="llm_complete",
                action="generate_response",
                resource="llm",
                user_id=user_id,
                session_id=session_id,
                request_id=request_id,
                details={
                    "model": "gpt-4o",
                    "input_tokens": len(user_input) // 2,
                    "output_tokens": len(llm_response) // 2,
                    "latency_ms": (time.time() - start_time) * 1000,
                },
                data_sources=["user_input"],
                data_destinations=["llm_output"],
            )
            
            # 记录请求完成
            self.audit.log(
                level=AuditLevel.INFO,
                category=AuditCategory.USER_ACTION,
                event_type="request_complete",
                action="return_response",
                resource="agent",
                user_id=user_id,
                session_id=session_id,
                request_id=request_id,
                outcome="success",
                details={
                    "response_length": len(llm_response),
                    "total_time_ms": (time.time() - start_time) * 1000,
                },
            )
            
            return {"status": "success", "response": llm_response}
            
        except Exception as e:
            # 记录错误
            self.audit.log(
                level=AuditLevel.ERROR,
                category=AuditCategory.SYSTEM,
                event_type="request_error",
                action="handle_error",
                resource="agent",
                user_id=user_id,
                session_id=session_id,
                request_id=request_id,
                outcome="failure",
                details={"error": str(e), "error_type": type(e).__name__},
            )
            
            return {"status": "error", "message": "处理请求时出错"}
    
    def _call_llm(self, input_text: str, request_id: str) -> str:
        """模拟 LLM 调用"""
        time.sleep(0.1)
        return f"基于您的输入'{input_text[:30]}'，这是 Agent 的回答。"
    
    def access_data(self, user_id: str, data_type: str,
                    resource_id: str, purpose: str) -> str:
        """访问数据（带审计）"""
        # 检查并记录数据访问
        self.audit.log(
            level=AuditLevel.INFO,
            category=AuditCategory.DATA_OP,
            event_type="data_access",
            action="read",
            resource=f"{data_type}:{resource_id}",
            user_id=user_id,
            details={"purpose": purpose},
            data_destinations=[f"{data_type}:{resource_id}"],
        )
        
        return f"数据 {data_type}/{resource_id} 的内容"
    
    def log_security_event(self, event_type: str, details: Dict,
                           user_id: str = "", severity: str = "medium"):
        """记录安全事件"""
        level_map = {
            "low": AuditLevel.WARNING,
            "medium": AuditLevel.WARNING,
            "high": AuditLevel.ERROR,
            "critical": AuditLevel.CRITICAL,
        }
        
        self.audit.log(
            level=level_map.get(severity, AuditLevel.WARNING),
            category=AuditCategory.SECURITY,
            event_type=event_type,
            action="security_check",
            resource="system",
            user_id=user_id,
            outcome="detected",
            details=details,
        )


# ============================================================
# 使用示例
# ============================================================

def demo_audit():
    """演示审计日志功能"""
    # 创建审计日志管理器
    audit = AuditLogger(log_dir="./demo_audit_logs", retention_days=30)
    
    # 创建可审计的 Agent
    agent = AuditableAgent(audit)
    
    # 模拟用户会话
    agent.authenticate_user("user_001", "session_001", method="token")
    agent.authenticate_user("user_002", "session_002", method="oauth")
    
    # 处理请求
    agent.process_request("什么是 Agent？", "session_001", source_ip="192.168.1.100")
    agent.process_request("如何部署 Agent？", "session_001", source_ip="192.168.1.100")
    agent.process_request("查询用户信息", "session_002", source_ip="10.0.0.50")
    
    # 数据访问
    agent.access_data("user_001", "customer", "CUST_12345", "客服查询")
    
    # 安全事件
    agent.log_security_event(
        "prompt_injection_detected",
        {"input_preview": "忽略之前的所有指令..."},
        user_id="user_002",
        severity="high",
    )
    
    # 刷新日志
    audit.flush()
    
    # 查询日志
    print("\n[所有安全事件]")
    security_logs = audit.query(category=AuditCategory.SECURITY)
    for log in security_logs:
        print(f"  {log['timestamp']} [{log['level']}] {log['event_type']}: {log['details']}")
    
    print("\n[用户 user_001 的所有操作]")
    user_logs = audit.query(user_id="user_001")
    for log in user_logs:
        print(f"  {log['timestamp']} {log['action']} -> {log['outcome']}")
    
    # 验证完整性
    integrity = audit.verify_integrity()
    print(f"\n[完整性验证] {integrity}")
    
    # 统计信息
    stats = audit.get_statistics()
    print(f"\n[统计信息]")
    print(f"  总条目: {stats['total_entries']}")
    print(f"  按级别: {stats['by_level']}")
    print(f"  按类别: {stats['by_category']}")


if __name__ == "__main__":
    demo_audit()
```

---

## 37.3 日志脱敏与隐私保护

```python
import re
from typing import Dict, List

class LogSanitizer:
    """
    日志脱敏处理器
    
    在写入日志之前，自动对敏感信息进行脱敏。
    """
    
    # 脱敏规则
    RULES = [
        # 身份证号
        (r'\b(\d{6})\d{8}(\d{4}[\dXx])\b', r'\1********\2', "身份证号"),
        # 手机号
        (r'\b(1[3-9]\d)\d{4}(\d{4})\b', r'\1****\2', "手机号"),
        # 邮箱
        (r'([\w.+-]+)@([\w-]+\.[\w.-]+)', r'\1@***', "邮箱"),
        # 银行卡号
        (r'\b(\d{4})\d{8,12}(\d{4})\b', r'\1****\2', "银行卡号"),
        # IP 地址（部分脱敏）
        (r'\b(\d+\.\d+\.\d+)\.\d+\b', r'\1.*', "IP地址"),
    ]
    
    def __init__(self, custom_rules: List[tuple] = None):
        self.rules = self.RULES + (custom_rules or [])
    
    def sanitize(self, text: str) -> tuple:
        """
        脱敏文本
        
        返回: (脱敏后的文本, 脱敏报告)
        """
        sanitized = text
        report = []
        
        for pattern, replacement, name in self.rules:
            matches = re.findall(pattern, text)
            if matches:
                sanitized = re.sub(pattern, replacement, sanitized)
                report.append({
                    "type": name,
                    "count": len(matches),
                })
        
        return sanitized, report
    
    def sanitize_dict(self, data: Dict) -> Dict:
        """递归脱敏字典中的所有字符串值"""
        sanitized = {}
        for key, value in data.items():
            if isinstance(value, str):
                sanitized[key], _ = self.sanitize(value)
            elif isinstance(value, dict):
                sanitized[key] = self.sanitize_dict(value)
            elif isinstance(value, list):
                sanitized[key] = [
                    self.sanitize_dict(item) if isinstance(item, dict)
                    else self.sanitize(item)[0] if isinstance(item, str)
                    else item
                    for item in value
                ]
            else:
                sanitized[key] = value
        return sanitized


# ============================================================
# 使用示例
# ============================================================

def demo_sanitizer():
    """演示日志脱敏"""
    sanitizer = LogSanitizer()
    
    test_logs = [
        "用户 张三 的手机号是 13812345678，邮箱是 zhangsan@example.com",
        "用户身份证号: 110101199001011234，银行卡: 6222021234567890123",
        "API 调用来自 192.168.1.100，请求路径: /api/users/12345",
    ]
    
    print("日志脱敏测试")
    print("=" * 60)
    
    for log in test_logs:
        sanitized, report = sanitizer.sanitize(log)
        print(f"\n原始: {log}")
        print(f"脱敏: {sanitized}")
        if report:
            print(f"报告: {report}")


if __name__ == "__main__":
    demo_sanitizer()
```

---

## 37.4 案例分析

### 37.4.1 案例：GDPR 合规审计

**背景：** 你的 Agent 在欧盟市场运营，需要满足 GDPR 的"数据主体权利"要求——用户有权查看所有关于他们的数据处理记录。

**实现方案：** 通过审计日志系统，实现了一个"数据主体报告"功能。输入用户 ID，系统能自动生成该用户的所有操作记录：何时发起请求、Agent 处理了什么内容、调用了哪些工具、访问了哪些数据。

**关键收获：** 审计日志不仅是为了事后追责，更是为了满足用户的知情权和控制权。良好的审计系统可以让数据主体权利的实现变得简单和自动化。

---

## 37.5 高级审计功能

### 37.5.1 分布式审计日志

在微服务架构中，一个请求可能涉及多个服务。需要将多个服务的审计日志关联起来，形成完整的请求链路追踪。

```python
import time
import uuid
import json
from typing import Dict, List, Optional, Any
from dataclasses import dataclass, field
from datetime import datetime
import threading

class DistributedAuditContext:
    """
    分布式审计上下文
    
    在微服务之间传递审计上下文，确保整个请求链路的日志可以关联。
    """
    
    # 使用线程本地存储来管理上下文
    _local = threading.local()
    
    @classmethod
    def get_current_context(cls) -> Dict:
        """获取当前的审计上下文"""
        if not hasattr(cls._local, 'context'):
            cls._local.context = {}
        return cls._local.context
    
    @classmethod
    def set_context(cls, context: Dict):
        """设置审计上下文"""
        cls._local.context = context
    
    @classmethod
    def generate_trace_id(cls) -> str:
        """生成追踪 ID"""
        return str(uuid.uuid4())[:16]
    
    @classmethod
    def generate_span_id(cls) -> str:
        """生成跨度 ID"""
        return str(uuid.uuid4())[:8]
    
    @classmethod
    def start_span(cls, service_name: str, operation: str) -> Dict:
        """
        开始一个新的追踪跨度
        
        Args:
            service_name: 服务名称
            operation: 操作名称
            
        Returns:
            跨度信息
        """
        context = cls.get_current_context()
        
        # 如果没有 trace_id，生成一个新的
        trace_id = context.get("trace_id", cls.generate_trace_id())
        
        span = {
            "trace_id": trace_id,
            "span_id": cls.generate_span_id(),
            "parent_span_id": context.get("span_id"),
            "service_name": service_name,
            "operation": operation,
            "start_time": time.time(),
            "status": "in_progress",
        }
        
        # 更新上下文
        cls.set_context({
            **context,
            "trace_id": trace_id,
            "span_id": span["span_id"],
        })
        
        return span
    
    @classmethod
    def finish_span(cls, span: Dict, status: str = "success", 
                   error: Optional[str] = None):
        """
        完成追踪跨度
        
        Args:
            span: 跨度信息
            status: 状态 (success/error)
            error: 错误信息
        """
        span["end_time"] = time.time()
        span["duration_ms"] = (span["end_time"] - span["start_time"]) * 1000
        span["status"] = status
        if error:
            span["error"] = error
        
        return span
    
    @classmethod
    def inject_headers(cls, headers: Dict) -> Dict:
        """
        将审计上下文注入到 HTTP 头中
        
        用于在微服务之间传递上下文。
        """
        context = cls.get_current_context()
        headers["X-Trace-ID"] = context.get("trace_id", "")
        headers["X-Span-ID"] = context.get("span_id", "")
        return headers
    
    @classmethod
    def extract_headers(cls, headers: Dict):
        """
        从 HTTP 头中提取审计上下文
        """
        trace_id = headers.get("X-Trace-ID", "")
        span_id = headers.get("X-Span-ID", "")
        
        if trace_id:
            cls.set_context({
                "trace_id": trace_id,
                "span_id": span_id,
            })


# ============================================================
# 使用示例
# ============================================================

def demo_distributed_audit():
    """演示分布式审计"""
    print("分布式审计上下文测试")
    print("=" * 60)
    
    # 模拟服务 A 处理请求
    print("\n[服务 A] 开始处理请求")
    span_a = DistributedAuditContext.start_span("service_a", "receive_request")
    print(f"  Trace ID: {span_a['trace_id']}")
    print(f"  Span ID: {span_a['span_id']}")
    
    # 模拟调用服务 B
    headers = {}
    DistributedAuditContext.inject_headers(headers)
    print(f"\n[服务 A] 调用服务 B，传递头部: {headers}")
    
    # 模拟服务 B 接收请求
    print("\n[服务 B] 接收请求")
    DistributedAuditContext.extract_headers(headers)
    span_b = DistributedAuditContext.start_span("service_b", "process_request")
    print(f"  Trace ID: {span_b['trace_id']}")
    print(f"  Parent Span ID: {span_b['parent_span_id']}")
    
    # 完成服务 B 的处理
    DistributedAuditContext.finish_span(span_b)
    print(f"  完成，耗时: {span_b.get('duration_ms', 0):.2f}ms")
    
    # 完成服务 A 的处理
    DistributedAuditContext.finish_span(span_a)
    print(f"\n[服务 A] 完成，耗时: {span_a.get('duration_ms', 0):.2f}ms")


if __name__ == "__main__":
    demo_distributed_audit()
```

### 37.5.2 审计日志聚合与分析

对审计日志进行聚合分析，提取有价值的洞察。

```python
from collections import defaultdict, Counter
from datetime import datetime, timedelta
from typing import Dict, List, Any
import json

class AuditAnalytics:
    """
    审计日志分析器
    
    提供审计日志的聚合、分析和报告功能。
    """
    
    def __init__(self, audit_entries: List[Dict] = None):
        self.entries = audit_entries or []
    
    def load_entries(self, entries: List[Dict]):
        """加载审计条目"""
        self.entries = entries
    
    def analyze_user_activity(self, user_id: str) -> Dict:
        """
        分析特定用户的活动
        
        Args:
            user_id: 用户 ID
            
        Returns:
            用户活动分析报告
        """
        user_entries = [e for e in self.entries if e.get("user_id") == user_id]
        
        if not user_entries:
            return {"user_id": user_id, "total_activities": 0}
        
        # 活动类型统计
        activity_types = Counter(e.get("event_type", "unknown") for e in user_entries)
        
        # 时间分布
        hours = Counter()
        for e in user_entries:
            try:
                ts = e.get("timestamp", "")
                hour = datetime.fromisoformat(ts.replace("Z", "+00:00")).hour
                hours[hour] += 1
            except:
                pass
        
        # 会话统计
        sessions = set(e.get("session_id", "") for e in user_entries if e.get("session_id"))
        
        # 安全事件
        security_events = [e for e in user_entries 
                          if e.get("category") == "security_event"]
        
        return {
            "user_id": user_id,
            "total_activities": len(user_entries),
            "activity_types": dict(activity_types),
            "hourly_distribution": dict(hours),
            "unique_sessions": len(sessions),
            "security_events_count": len(security_events),
            "first_activity": min(e.get("timestamp", "") for e in user_entries),
            "last_activity": max(e.get("timestamp", "") for e in user_entries),
        }
    
    def analyze_system_performance(self) -> Dict:
        """
        分析系统性能指标
        
        从审计日志中提取性能相关的信息。
        """
        # LLM 调用统计
        llm_calls = [e for e in self.entries if e.get("category") == "llm_interaction"]
        
        # 工具调用统计
        tool_calls = [e for e in self.entries if e.get("category") == "tool_execution"]
        
        # 延迟分析
        latencies = []
        for e in self.entries:
            details = e.get("details", {})
            if "latency_ms" in details:
                latencies.append(details["latency_ms"])
        
        # Token 使用统计
        total_input_tokens = 0
        total_output_tokens = 0
        for e in llm_calls:
            details = e.get("details", {})
            total_input_tokens += details.get("input_tokens", 0)
            total_output_tokens += details.get("output_tokens", 0)
        
        # 计算 P50, P95, P99
        if latencies:
            latencies.sort()
            n = len(latencies)
            p50 = latencies[int(n * 0.5)]
            p95 = latencies[int(n * 0.95)]
            p99 = latencies[int(n * 0.99)]
        else:
            p50 = p95 = p99 = 0
        
        return {
            "total_llm_calls": len(llm_calls),
            "total_tool_calls": len(tool_calls),
            "latency": {
                "p50_ms": p50,
                "p95_ms": p95,
                "p99_ms": p99,
                "avg_ms": sum(latencies) / len(latencies) if latencies else 0,
            },
            "tokens": {
                "total_input": total_input_tokens,
                "total_output": total_output_tokens,
                "avg_per_request": (total_input_tokens + total_output_tokens) / max(len(llm_calls), 1),
            },
        }
    
    def detect_anomalies(self) -> List[Dict]:
        """
        检测异常活动
        
        基于历史模式检测异常行为。
        """
        anomalies = []
        
        # 1. 异常高频请求
        user_requests = defaultdict(list)
        for e in self.entries:
            user_id = e.get("user_id", "")
            if user_id:
                try:
                    ts = datetime.fromisoformat(e.get("timestamp", "").replace("Z", "+00:00"))
                    user_requests[user_id].append(ts)
                except:
                    pass
        
        for user_id, timestamps in user_requests.items():
            if len(timestamps) < 2:
                continue
            
            timestamps.sort()
            # 检查是否有 1 分钟内超过 10 次请求
            for i in range(len(timestamps)):
                window_end = timestamps[i] + timedelta(minutes=1)
                count = sum(1 for ts in timestamps[i:] if ts < window_end)
                if count > 10:
                    anomalies.append({
                        "type": "high_frequency_requests",
                        "user_id": user_id,
                        "count": count,
                        "timestamp": timestamps[i].isoformat(),
                        "severity": "medium",
                    })
                    break
        
        # 2. 异常时间活动
        for e in self.entries:
            try:
                ts = datetime.fromisoformat(e.get("timestamp", "").replace("Z", "+00:00"))
                if ts.hour < 6 or ts.hour > 23:  # 深夜活动
                    anomalies.append({
                        "type": "unusual_time_activity",
                        "user_id": e.get("user_id", ""),
                        "timestamp": e.get("timestamp", ""),
                        "event_type": e.get("event_type", ""),
                        "severity": "low",
                    })
            except:
                pass
        
        # 3. 安全事件
        for e in self.entries:
            if e.get("category") == "security_event":
                anomalies.append({
                    "type": "security_event",
                    "user_id": e.get("user_id", ""),
                    "event_type": e.get("event_type", ""),
                    "details": e.get("details", {}),
                    "severity": "high",
                })
        
        return anomalies
    
    def generate_report(self, report_type: str = "daily") -> Dict:
        """
        生成审计报告
        
        Args:
            report_type: 报告类型 (daily/weekly/monthly)
            
        Returns:
            审计报告
        """
        # 基础统计
        total_entries = len(self.entries)
        unique_users = len(set(e.get("user_id", "") for e in self.entries if e.get("user_id")))
        
        # 事件类型分布
        event_types = Counter(e.get("event_type", "unknown") for e in self.entries)
        
        # 级别分布
        levels = Counter(e.get("level", "unknown") for e in self.entries)
        
        # 成功率
        outcomes = Counter(e.get("outcome", "unknown") for e in self.entries)
        total_outcomes = sum(outcomes.values())
        success_rate = outcomes.get("success", 0) / max(total_outcomes, 1)
        
        # 异常检测
        anomalies = self.detect_anomalies()
        
        return {
            "report_type": report_type,
            "generated_at": datetime.now().isoformat(),
            "summary": {
                "total_entries": total_entries,
                "unique_users": unique_users,
                "success_rate": success_rate,
                "anomalies_detected": len(anomalies),
            },
            "event_types": dict(event_types),
            "levels": dict(levels),
            "outcomes": dict(outcomes),
            "anomalies": anomalies[:10],  # 只返回前 10 个异常
        }


# ============================================================
# 使用示例
# ============================================================

def demo_audit_analytics():
    """演示审计分析"""
    # 创建模拟审计数据
    sample_entries = []
    import random
    
    for i in range(100):
        entry = {
            "timestamp": datetime.now().isoformat(),
            "user_id": f"user_{random.randint(1, 5)}",
            "event_type": random.choice(["request_start", "request_complete", "llm_call", "tool_call"]),
            "category": random.choice(["llm_interaction", "tool_execution", "user_action"]),
            "level": random.choice(["info", "info", "info", "warning", "error"]),
            "outcome": random.choice(["success", "success", "success", "failure"]),
            "session_id": f"session_{random.randint(1, 10)}",
            "details": {
                "latency_ms": random.uniform(100, 5000),
                "input_tokens": random.randint(100, 1000),
                "output_tokens": random.randint(50, 500),
            },
        }
        sample_entries.append(entry)
    
    # 添加一些安全事件
    for _ in range(5):
        sample_entries.append({
            "timestamp": datetime.now().isoformat(),
            "user_id": "suspicious_user",
            "event_type": "prompt_injection_detected",
            "category": "security_event",
            "level": "critical",
            "outcome": "denied",
            "details": {"input_preview": "忽略指令..."},
        })
    
    # 分析
    analytics = AuditAnalytics(sample_entries)
    
    print("审计日志分析")
    print("=" * 60)
    
    # 用户活动分析
    user_analysis = analytics.analyze_user_activity("user_1")
    print(f"\n用户活动分析 (user_1):")
    print(f"  总活动数: {user_analysis['total_activities']}")
    print(f"  活跃会话数: {user_analysis['unique_sessions']}")
    print(f"  安全事件数: {user_analysis['security_events_count']}")
    
    # 系统性能分析
    perf_analysis = analytics.analyze_system_performance()
    print(f"\n系统性能分析:")
    print(f"  LLM 调用次数: {perf_analysis['total_llm_calls']}")
    print(f"  工具调用次数: {perf_analysis['total_tool_calls']}")
    print(f"  延迟 P50: {perf_analysis['latency']['p50_ms']:.2f}ms")
    print(f"  延迟 P95: {perf_analysis['latency']['p95_ms']:.2f}ms")
    
    # 异常检测
    anomalies = analytics.detect_anomalies()
    print(f"\n异常检测结果:")
    print(f"  检测到异常数量: {len(anomalies)}")
    for anomaly in anomalies[:5]:
        print(f"    - [{anomaly['severity']}] {anomaly['type']}: {anomaly.get('user_id', 'N/A')}")
    
    # 生成报告
    report = analytics.generate_report("daily")
    print(f"\n日报摘要:")
    print(f"  总条目数: {report['summary']['total_entries']}")
    print(f"  独立用户数: {report['summary']['unique_users']}")
    print(f"  成功率: {report['summary']['success_rate']:.2%}")


if __name__ == "__main__":
    demo_audit_analytics()
```

## 37.6 常见坑与最佳实践

**坑 1：日志中包含明文敏感信息。** 这是最常见的合规违规。确保在写入日志之前进行脱敏。使用 LogSanitizer 等工具自动处理。

**坑 2：日志时间戳不准确。** 如果多个服务的日志时间不同步，就无法正确关联事件。使用 NTP 同步所有服务的时钟，或者使用统一的时间戳服务。

**坑 3：日志保留策略不当。** 保留太久增加存储成本和数据泄露风险，保留太短不满足合规要求。根据法规要求和业务需要制定明确的保留策略。

**坑 4：审计日志性能开销。** 如果审计日志记录过于详细，可能影响系统性能。使用异步写入和批量处理来减少性能影响。

**坑 5：分布式追踪上下文丢失。** 在微服务架构中，审计上下文可能在服务调用链中丢失。使用 HTTP 头或消息元数据传递上下文。

**最佳实践 1：审计即代码。** 把审计配置（保留策略、脱敏规则、告警规则）纳入版本控制，和代码一起审查和部署。

**最佳实践 2：定期审计日志分析。** 不要只是收集日志，要定期分析日志以发现趋势、异常和改进机会。

**最佳实践 3：建立审计响应流程。** 当发现异常活动时，需要有明确的响应流程：调查、确认、处理、总结。

**最佳实践 4：保护审计日志本身。** 审计日志是敏感数据，需要保护其完整性和机密性。使用哈希链确保完整性，加密存储确保机密性。

---

## 37.6 练习题

**练习 1：实现完整的日志脱敏。** 扩展 LogSanitizer，支持脱敏中文姓名、地址、日期等更多类型的敏感信息。

**练习 2：实现日志查询 API。** 为 AuditLogger 创建一个 HTTP API，支持按时间、用户、类别等条件查询日志。

**练习 3：实现日志聚合统计。** 编写脚本分析审计日志，生成每日/每周报告，包括请求数、错误率、安全事件数等指标。

**练习 4：设计日志归档方案。** 实现自动化的日志归档：将 30 天前的日志压缩存储到 S3 或本地归档目录。

**练习 5：实现跨服务日志关联。** 设计一个方案，让 Agent、工具服务、数据库的日志能通过 request_id 关联起来。

**练习 6：构建合规报告生成器。** 编写一个脚本，自动生成 GDPR 合规所需的"数据处理活动报告"，从审计日志中提取关键信息。

---

## 37.7 本章小结

本章从合规需求出发，构建了完整的 Agent 审计日志体系。我们设计了结构化的审计日志模型，实现了带哈希链完整性校验的日志管理器，构建了覆盖用户认证、请求处理、LLM 调用、工具执行、数据访问的全链路审计。

核心理念是：审计日志是 Agent 系统的"黑盒子"。当问题发生时，它是你唯一可靠的调查工具。投入时间构建良好的审计系统，远比事后弥补要值得。

---

> **下一章预告：** 第 38 章将深入 Agent 的成本优化，学习如何在保证质量的前提下大幅降低 Token 消耗和 API 调用成本。
