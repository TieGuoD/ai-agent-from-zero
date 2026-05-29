# 第 42 章：Agent 版本管理与 A/B 测试

> **本章定位：** Agent 不是"部署一次就完事"的产品。Prompt 在迭代、模型在更新、工具在增加。如何安全地管理这些变化，如何用数据验证每次改进是否有效，是本章要解决的核心问题。

---

## 学习目标

完成本章学习后，你将能够：

1. **建立 Agent 的版本管理体系** —— 能为 Prompt、配置、模型选择等关键组件实施版本控制
2. **设计和运行 A/B 测试** —— 能用统计学方法验证 Agent 改进的实际效果
3. **实现灰度发布** —— 能逐步将新版本 Agent 推送给部分用户
4. **构建实验追踪系统** —— 能记录和对比不同版本 Agent 的表现
5. **实施快速回滚策略** —— 当新版本出现问题时能快速恢复

## 核心问题

1. **Agent 的"版本"包含什么？** 传统软件的版本是代码，Agent 的版本还包含 Prompt、模型配置、工具集——如何管理这些非代码组件？
2. **A/B 测试需要多少样本量？** 测试太少结论不可靠，测试太久影响用户体验。
3. **如何判断新版本是否"显著更好"？** LLM 输出的概率性让统计显著性检验变得复杂。

---

## 42.1 Agent 版本管理

### 42.1.1 版本管理的范围

Agent 的版本管理不仅仅是代码版本控制。以下组件都需要版本化：

**System Prompt** —— 这是 Agent 行为的核心定义。修改一个词可能导致 Agent 的行为发生显著变化。每次 Prompt 修改都应该有版本记录。

**模型配置** —— 模型名称、temperature、max_tokens 等参数的组合决定了 LLM 的行为特征。

**工具集** —— Agent 可用的工具列表和工具的实现。新增工具、修改工具行为、移除工具都会影响 Agent 的能力。

**评估数据集** —— 用于评估 Agent 质量的测试用例。评估数据集的版本应该和 Agent 版本对应。

### 42.1.2 版本管理器实现

```python
import os
import json
import hashlib
import time
from typing import Dict, List, Optional, Any
from dataclasses import dataclass, field, asdict
from datetime import datetime
from pathlib import Path

@dataclass
class AgentVersion:
    """Agent 版本定义"""
    version_id: str
    name: str
    description: str
    system_prompt: str
    model: str
    temperature: float
    max_tokens: int
    tools: List[str]
    metadata: Dict[str, Any] = field(default_factory=dict)
    created_at: str = field(default_factory=lambda: datetime.now().isoformat())
    
    def compute_hash(self) -> str:
        """计算版本指纹"""
        content = json.dumps({
            "system_prompt": self.system_prompt,
            "model": self.model,
            "temperature": self.temperature,
            "max_tokens": self.max_tokens,
            "tools": sorted(self.tools),
        }, sort_keys=True)
        return hashlib.sha256(content.encode()).hexdigest()[:12]

class VersionManager:
    """
    Agent 版本管理器
    
    管理 Agent 的所有版本，支持：
    - 版本创建和存储
    - 版本对比
    - 版本回滚
    - 版本标签
    """
    
    def __init__(self, storage_dir: str = "./agent_versions"):
        self.storage_dir = Path(storage_dir)
        self.storage_dir.mkdir(parents=True, exist_ok=True)
        self.versions: Dict[str, AgentVersion] = {}
        self.current_version: Optional[str] = None
        self.version_history: List[str] = []
        
        # 加载已有版本
        self._load_versions()
    
    def _load_versions(self):
        """从磁盘加载版本"""
        index_file = self.storage_dir / "index.json"
        if index_file.exists():
            with open(index_file, "r", encoding="utf-8") as f:
                data = json.load(f)
                self.current_version = data.get("current_version")
                self.version_history = data.get("history", [])
                for v_id, v_data in data.get("versions", {}).items():
                    self.versions[v_id] = AgentVersion(**v_data)
    
    def _save_versions(self):
        """保存版本到磁盘"""
        index_file = self.storage_dir / "index.json"
        data = {
            "current_version": self.current_version,
            "history": self.version_history,
            "versions": {v_id: asdict(v) for v_id, v in self.versions.items()},
        }
        with open(index_file, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
    
    def create_version(self, name: str, description: str,
                       system_prompt: str, model: str,
                       temperature: float = 0.7,
                       max_tokens: int = 4096,
                       tools: List[str] = None) -> AgentVersion:
        """创建新版本"""
        version_id = f"v{len(self.versions) + 1}_{int(time.time())}"
        
        version = AgentVersion(
            version_id=version_id,
            name=name,
            description=description,
            system_prompt=system_prompt,
            model=model,
            temperature=temperature,
            max_tokens=max_tokens,
            tools=tools or [],
            metadata={"fingerprint": ""},
        )
        version.metadata["fingerprint"] = version.compute_hash()
        
        self.versions[version_id] = version
        self.current_version = version_id
        self.version_history.append(version_id)
        
        self._save_versions()
        return version
    
    def get_current(self) -> Optional[AgentVersion]:
        """获取当前版本"""
        return self.versions.get(self.current_version)
    
    def rollback(self, version_id: str) -> Optional[AgentVersion]:
        """回滚到指定版本"""
        if version_id not in self.versions:
            print(f"版本 {version_id} 不存在")
            return None
        
        self.current_version = version_id
        self._save_versions()
        print(f"已回滚到版本: {version_id}")
        return self.versions[version_id]
    
    def compare(self, v1_id: str, v2_id: str) -> Dict:
        """对比两个版本"""
        v1 = self.versions.get(v1_id)
        v2 = self.versions.get(v2_id)
        
        if not v1 or not v2:
            return {"error": "版本不存在"}
        
        differences = {}
        
        if v1.system_prompt != v2.system_prompt:
            differences["system_prompt"] = {
                "old_preview": v1.system_prompt[:100],
                "new_preview": v2.system_prompt[:100],
            }
        if v1.model != v2.model:
            differences["model"] = {"old": v1.model, "new": v2.model}
        if v1.temperature != v2.temperature:
            differences["temperature"] = {"old": v1.temperature, "new": v2.temperature}
        
        old_tools = set(v1.tools)
        new_tools = set(v2.tools)
        if old_tools != new_tools:
            differences["tools"] = {
                "added": list(new_tools - old_tools),
                "removed": list(old_tools - new_tools),
            }
        
        return {
            "version_1": v1_id,
            "version_2": v2_id,
            "differences": differences,
            "same_fingerprint": v1.compute_hash() == v2.compute_hash(),
        }
    
    def list_versions(self) -> List[Dict]:
        """列出所有版本"""
        return [
            {
                "id": v.version_id,
                "name": v.name,
                "created_at": v.created_at,
                "fingerprint": v.compute_hash(),
                "is_current": v.version_id == self.current_version,
            }
            for v in self.versions.values()
        ]


# ============================================================
# 使用示例
# ============================================================

def demo_version_management():
    """演示版本管理"""
    vm = VersionManager("./demo_versions")
    
    # 创建版本
    v1 = vm.create_version(
        name="初始版本",
        description="基础客服 Agent",
        system_prompt="你是一个客服助手。请友好地回答用户的问题。",
        model="gpt-4o",
        temperature=0.7,
        tools=["search", "faq_lookup"],
    )
    
    v2 = vm.create_version(
        name="优化版本",
        description="添加了工具调用能力",
        system_prompt="你是一个智能客服助手。先搜索知识库，再回答用户问题。",
        model="gpt-4o",
        temperature=0.5,
        tools=["search", "faq_lookup", "order_query"],
    )
    
    # 列出版本
    print("版本列表:")
    for v in vm.list_versions():
        current = " [当前]" if v["is_current"] else ""
        print(f"  {v['id']}: {v['name']}{current}")
    
    # 对比版本
    diff = vm.compare(v1.version_id, v2.version_id)
    print(f"\n版本差异:")
    for key, value in diff["differences"].items():
        print(f"  {key}: {value}")
    
    # 回滚
    vm.rollback(v1.version_id)
    print(f"\n当前版本: {vm.current_version}")


if __name__ == "__main__":
    demo_version_management()
```

---

## 42.2 A/B 测试框架

### 42.2.1 统计显著性检验

```python
import math
from typing import List, Dict, Tuple
from dataclasses import dataclass, field

@dataclass
class ABTestResult:
    """A/B 测试结果"""
    variant_a: Dict
    variant_b: Dict
    metric_name: str
    p_value: float
    is_significant: bool
    confidence_level: float
    effect_size: float
    recommendation: str

class ABTestFramework:
    """
    Agent A/B 测试框架
    
    支持对 Agent 的不同版本进行统计学显著性检验。
    """
    
    def __init__(self, confidence_level: float = 0.95):
        self.confidence_level = confidence_level
        self.experiments: List[Dict] = []
    
    def compute_metrics(self, scores: List[float]) -> Dict:
        """计算一组分数的统计指标"""
        n = len(scores)
        if n == 0:
            return {"mean": 0, "std": 0, "n": 0}
        
        mean = sum(scores) / n
        variance = sum((x - mean) ** 2 for x in scores) / max(n - 1, 1)
        std = math.sqrt(variance)
        
        # 计算置信区间
        se = std / math.sqrt(n)
        z = 1.96 if self.confidence_level == 0.95 else 2.576
        ci_lower = mean - z * se
        ci_upper = mean + z * se
        
        return {
            "mean": mean,
            "std": std,
            "n": n,
            "se": se,
            "ci_lower": ci_lower,
            "ci_upper": ci_upper,
        }
    
    def welch_t_test(self, scores_a: List[float], 
                     scores_b: List[float]) -> Tuple[float, float]:
        """
        Welch t 检验
        
        比较两组样本的均值是否有显著差异。
        """
        stats_a = self.compute_metrics(scores_a)
        stats_b = self.compute_metrics(scores_b)
        
        if stats_a["n"] < 2 or stats_b["n"] < 2:
            return 0.0, 1.0
        
        # t 统计量
        se_diff = math.sqrt(stats_a["se"]**2 + stats_b["se"]**2)
        if se_diff == 0:
            return 0.0, 1.0
        
        t_stat = (stats_b["mean"] - stats_a["mean"]) / se_diff
        
        # 自由度（Welch-Satterthwaite 公式）
        num = (stats_a["se"]**2 + stats_b["se"]**2) ** 2
        den = (stats_a["se"]**4 / (stats_a["n"] - 1) + 
               stats_b["se"]**4 / (stats_b["n"] - 1))
        df = num / den if den > 0 else 1
        
        # 简化的 p 值计算（使用正态近似）
        # 生产环境应该用 scipy.stats
        p_value = 2 * (1 - self._normal_cdf(abs(t_stat)))
        
        return t_stat, p_value
    
    def _normal_cdf(self, x: float) -> float:
        """正态分布 CDF 的近似计算"""
        return 0.5 * (1 + math.erf(x / math.sqrt(2)))
    
    def compute_effect_size(self, scores_a: List[float],
                           scores_b: List[float]) -> float:
        """计算 Cohen's d 效应量"""
        stats_a = self.compute_metrics(scores_a)
        stats_b = self.compute_metrics(scores_b)
        
        pooled_std = math.sqrt(
            ((stats_a["n"] - 1) * stats_a["std"]**2 + 
             (stats_b["n"] - 1) * stats_b["std"]**2) /
            (stats_a["n"] + stats_b["n"] - 2)
        )
        
        if pooled_std == 0:
            return 0.0
        
        return (stats_b["mean"] - stats_a["mean"]) / pooled_std
    
    def run_test(self, variant_a_name: str, scores_a: List[float],
                 variant_b_name: str, scores_b: List[float],
                 metric_name: str = "score") -> ABTestResult:
        """运行 A/B 测试"""
        stats_a = self.compute_metrics(scores_a)
        stats_b = self.compute_metrics(scores_b)
        
        t_stat, p_value = self.welch_t_test(scores_a, scores_b)
        effect_size = self.compute_effect_size(scores_a, scores_b)
        
        is_significant = p_value < (1 - self.confidence_level)
        
        # 生成建议
        if not is_significant:
            recommendation = "差异不显著，建议继续测试或增加样本量"
        elif effect_size > 0.2:
            recommendation = f"变体 B 显著优于变体 A (效应量={effect_size:.3f})，建议采用变体 B"
        elif effect_size < -0.2:
            recommendation = f"变体 A 显著优于变体 B (效应量={effect_size:.3f})，建议保留变体 A"
        else:
            recommendation = "差异虽显著但效应量小，需结合其他因素决定"
        
        result = ABTestResult(
            variant_a={"name": variant_a_name, **stats_a},
            variant_b={"name": variant_b_name, **stats_b},
            metric_name=metric_name,
            p_value=p_value,
            is_significant=is_significant,
            confidence_level=self.confidence_level,
            effect_size=effect_size,
            recommendation=recommendation,
        )
        
        self.experiments.append({
            "result": result,
            "timestamp": time.time(),
        })
        
        return result
    
    def print_result(self, result: ABTestResult):
        """打印测试结果"""
        print("\n" + "=" * 60)
        print("  A/B 测试结果")
        print("=" * 60)
        
        print(f"\n指标: {result.metric_name}")
        print(f"置信水平: {result.confidence_level:.0%}")
        
        print(f"\n变体 A ({result.variant_a['name']}):")
        print(f"  均值: {result.variant_a['mean']:.4f}")
        print(f"  标准差: {result.variant_a['std']:.4f}")
        print(f"  样本量: {result.variant_a['n']}")
        print(f"  95% CI: [{result.variant_a['ci_lower']:.4f}, {result.variant_a['ci_upper']:.4f}]")
        
        print(f"\n变体 B ({result.variant_b['name']}):")
        print(f"  均值: {result.variant_b['mean']:.4f}")
        print(f"  标准差: {result.variant_b['std']:.4f}")
        print(f"  样本量: {result.variant_b['n']}")
        print(f"  95% CI: [{result.variant_b['ci_lower']:.4f}, {result.variant_b['ci_upper']:.4f}]")
        
        print(f"\n统计检验:")
        print(f"  p 值: {result.p_value:.6f}")
        print(f"  显著: {'是' if result.is_significant else '否'}")
        print(f"  效应量: {result.effect_size:.4f}")
        
        print(f"\n建议: {result.recommendation}")
        print("=" * 60)


# ============================================================
# 使用示例
# ============================================================

def demo_ab_test():
    """演示 A/B 测试"""
    import random
    
    framework = ABTestFramework(confidence_level=0.95)
    
    # 模拟两个版本的 Agent 评分
    random.seed(42)
    
    # 版本 A: 基线版本
    scores_a = [random.gauss(0.72, 0.15) for _ in range(50)]
    scores_a = [max(0, min(1, s)) for s in scores_a]
    
    # 版本 B: 优化版本
    scores_b = [random.gauss(0.78, 0.15) for _ in range(50)]
    scores_b = [max(0, min(1, s)) for s in scores_b]
    
    # 运行测试
    result = framework.run_test(
        variant_a_name="基线版本 (v1)",
        scores_a=scores_a,
        variant_b_name="优化版本 (v2)",
        scores_b=scores_b,
        metric_name="回答质量评分",
    )
    
    framework.print_result(result)


if __name__ == "__main__":
    demo_ab_test()
```

---

## 42.3 灰度发布策略

### 42.3.1 灰度发布器

```python
import random
import time
from typing import Dict, List, Callable
from dataclasses import dataclass, field

@dataclass
class TrafficSplit:
    """流量分配"""
    version_id: str
    percentage: float  # 0-100

class CanaryDeployer:
    """
    灰度发布器
    
    支持渐进式的 Agent 版本发布：
    1. 5% 流量 -> 观察
    2. 20% 流量 -> 观察
    3. 50% 流量 -> 观察
    4. 100% 流量 -> 全量发布
    """
    
    def __init__(self):
        self.stages = [5, 20, 50, 100]  # 渐进百分比
        self.current_stage: int = 0
        self.deployments: Dict[str, Dict] = {}
        self.metrics: Dict[str, List[float]] = {}
    
    def start_deployment(self, version_id: str, 
                         version_config: Dict) -> Dict:
        """开始灰度部署"""
        self.deployments[version_id] = {
            "config": version_config,
            "stage": 0,
            "percentage": self.stages[0],
            "start_time": time.time(),
            "status": "in_progress",
            "metrics_history": [],
        }
        
        print(f"开始灰度部署 {version_id}")
        print(f"  第一阶段: {self.stages[0]}% 流量")
        
        return self.deployments[version_id]
    
    def evaluate_stage(self, version_id: str, 
                       stage_metrics: Dict) -> bool:
        """
        评估当前阶段是否通过
        
        返回 True 表示可以进入下一阶段。
        """
        deployment = self.deployments.get(version_id)
        if not deployment:
            return False
        
        deployment["metrics_history"].append(stage_metrics)
        
        # 检查关键指标
        error_rate = stage_metrics.get("error_rate", 0)
        avg_latency = stage_metrics.get("avg_latency", 0)
        user_satisfaction = stage_metrics.get("user_satisfaction", 0)
        
        # 通过条件
        passed = (
            error_rate < 0.05 and  # 错误率 < 5%
            avg_latency < 5.0 and  # 平均延迟 < 5 秒
            user_satisfaction > 0.7  # 满意度 > 70%
        )
        
        if passed:
            # 进入下一阶段
            deployment["stage"] += 1
            if deployment["stage"] < len(self.stages):
                deployment["percentage"] = self.stages[deployment["stage"]]
                print(f"  阶段通过！进入下一阶段: {deployment['percentage']}% 流量")
            else:
                deployment["status"] = "completed"
                print(f"  灰度部署完成！{version_id} 已全量上线")
        else:
            deployment["status"] = "failed"
            print(f"  阶段未通过。错误率={error_rate:.2%}, 延迟={avg_latency:.1f}s, 满意度={user_satisfaction:.2%}")
            print(f"  部署已终止，建议回滚")
        
        return passed
    
    def get_traffic_allocation(self, user_id: str) -> str:
        """根据用户 ID 决定路由到哪个版本"""
        # 使用一致性哈希确保同一用户总是路由到同一版本
        hash_value = hash(user_id) % 100
        
        active_deployments = [
            (v_id, d) for v_id, d in self.deployments.items()
            if d["status"] == "in_progress"
        ]
        
        if not active_deployments:
            return "current"
        
        # 简单的流量分配
        version_id, deployment = active_deployments[0]
        if hash_value < deployment["percentage"]:
            return version_id
        return "current"
    
    def rollback(self, version_id: str):
        """回滚部署"""
        if version_id in self.deployments:
            self.deployments[version_id]["status"] = "rolled_back"
            print(f"已回滚: {version_id}")


# ============================================================
# 使用示例
# ============================================================

def demo_canary_deploy():
    """演示灰度部署"""
    deployer = CanaryDeployer()
    
    # 开始部署
    deployment = deployer.start_deployment(
        "v2_optimized",
        {"model": "gpt-4o", "temperature": 0.5}
    )
    
    # 模拟每个阶段的指标
    import random
    random.seed(42)
    
    stages_data = [
        {"error_rate": 0.02, "avg_latency": 1.5, "user_satisfaction": 0.82},
        {"error_rate": 0.03, "avg_latency": 1.8, "user_satisfaction": 0.85},
        {"error_rate": 0.04, "avg_latency": 2.0, "user_satisfaction": 0.83},
        {"error_rate": 0.02, "avg_latency": 1.6, "user_satisfaction": 0.86},
    ]
    
    for i, metrics in enumerate(stages_data):
        print(f"\n--- 阶段 {i+1} 评估 ---")
        passed = deployer.evaluate_stage("v2_optimized", metrics)
        if not passed:
            break
    
    # 查看最终状态
    final = deployer.deployments["v2_optimized"]
    print(f"\n最终状态: {final['status']}")


if __name__ == "__main__":
    demo_canary_deploy()
```

---

## 42.4 案例分析

### 42.4.1 案例：Prompt 迭代的 A/B 测试

**背景：** Agent 的回答质量评分稳定在 72%。团队提出了一个新的 System Prompt，期望提升到 80%。

**A/B 测试设计：** 50% 用户看到旧 Prompt 的 Agent，50% 看到新 Prompt 的 Agent。运行 7 天，收集了每组各 5000 个样本的回答质量评分。

**结果：** 旧版均值 0.723，新版均值 0.741。p 值 = 0.03，在 95% 置信水平下差异显著。但效应量只有 0.08（小效应），意味着新 Prompt 只带来了约 2 个百分点的提升。

**决策：** 虽然差异显著，但效应量太小。团队决定继续优化 Prompt，而不是仓促上线。两周后的新版本带来了 8 个百分点的提升（效应量 0.25），这才值得全量发布。

**关键收获：** 统计显著性和实际显著性是两回事。大样本量下，很小的差异也能达到统计显著，但这种微小的改进可能不值得承担部署风险。

---

## 42.5 高级版本管理功能

### 42.5.1 版本标签与元数据

为版本添加标签和丰富的元数据，便于管理和追踪。

```python
from typing import Dict, List, Any, Optional
from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum

class VersionStatus(Enum):
    """版本状态"""
    DRAFT = "draft"           # 草稿
    TESTING = "testing"       # 测试中
    CANARY = "canary"         # 金丝雀发布
    PRODUCTION = "production" # 生产环境
    ARCHIVED = "archived"     # 已归档
    DEPRECATED = "deprecated" # 已废弃

@dataclass
class VersionTag:
    """版本标签"""
    name: str
    color: str  # 标签颜色（用于 UI 显示）
    description: str = ""

@dataclass
class VersionMetadata:
    """版本元数据"""
    author: str
    created_at: str
    updated_at: str
    description: str
    changelog: List[str] = field(default_factory=list)
    tags: List[str] = field(default_factory=list)
    status: VersionStatus = VersionStatus.DRAFT
    performance_metrics: Dict[str, float] = field(default_factory=dict)
    related_tickets: List[str] = field(default_factory=list)
    approved_by: Optional[str] = None
    approved_at: Optional[str] = None

class EnhancedVersionManager:
    """
    增强版版本管理器
    
    提供更丰富的版本管理功能：
    - 版本标签
    - 审批流程
    - 性能追踪
    - 依赖管理
    """
    
    def __init__(self, storage_dir: str = "./enhanced_versions"):
        self.storage_dir = Path(storage_dir)
        self.storage_dir.mkdir(parents=True, exist_ok=True)
        
        self.versions: Dict[str, AgentVersion] = {}
        self.metadata: Dict[str, VersionMetadata] = {}
        self.tags: Dict[str, VersionTag] = {}
        
        self._create_default_tags()
    
    def _create_default_tags(self):
        """创建默认标签"""
        self.tags = {
            "stable": VersionTag("stable", "#4CAF50", "稳定版本"),
            "experimental": VersionTag("experimental", "#FF9800", "实验性功能"),
            "hotfix": VersionTag("hotfix", "#F44336", "紧急修复"),
            "breaking": VersionTag("breaking", "#9C27B0", "破坏性变更"),
        }
    
    def create_version(self, name: str, description: str,
                      system_prompt: str, model: str,
                      author: str, tags: List[str] = None,
                      changelog: List[str] = None,
                      **kwargs) -> AgentVersion:
        """创建新版本（带元数据）"""
        version_id = f"v{len(self.versions) + 1}_{int(time.time())}"
        
        version = AgentVersion(
            version_id=version_id,
            name=name,
            description=description,
            system_prompt=system_prompt,
            model=model,
            temperature=kwargs.get("temperature", 0.7),
            max_tokens=kwargs.get("max_tokens", 4096),
            tools=kwargs.get("tools", []),
        )
        
        self.versions[version_id] = version
        
        # 创建元数据
        now = datetime.now().isoformat()
        self.metadata[version_id] = VersionMetadata(
            author=author,
            created_at=now,
            updated_at=now,
            description=description,
            changelog=changelog or [],
            tags=tags or [],
            status=VersionStatus.DRAFT,
        )
        
        return version
    
    def update_status(self, version_id: str, new_status: VersionStatus,
                     approved_by: str = None):
        """更新版本状态"""
        if version_id not in self.metadata:
            return False
        
        metadata = self.metadata[version_id]
        old_status = metadata.status
        metadata.status = new_status
        metadata.updated_at = datetime.now().isoformat()
        
        if approved_by:
            metadata.approved_by = approved_by
            metadata.approved_at = datetime.now().isoformat()
        
        print(f"版本 {version_id} 状态变更: {old_status.value} -> {new_status.value}")
        return True
    
    def add_changelog(self, version_id: str, entry: str):
        """添加变更日志"""
        if version_id in self.metadata:
            self.metadata[version_id].changelog.append(entry)
            self.metadata[version_id].updated_at = datetime.now().isoformat()
    
    def record_performance(self, version_id: str, 
                          metrics: Dict[str, float]):
        """记录性能指标"""
        if version_id in self.metadata:
            self.metadata[version_id].performance_metrics.update(metrics)
            self.metadata[version_id].updated_at = datetime.now().isoformat()
    
    def get_version_report(self, version_id: str) -> Dict:
        """获取版本报告"""
        version = self.versions.get(version_id)
        metadata = self.metadata.get(version_id)
        
        if not version or not metadata:
            return {"error": "版本不存在"}
        
        return {
            "version_id": version_id,
            "name": version.name,
            "description": metadata.description,
            "status": metadata.status.value,
            "author": metadata.author,
            "created_at": metadata.created_at,
            "fingerprint": version.compute_hash(),
            "tags": metadata.tags,
            "changelog": metadata.changelog,
            "performance": metadata.performance_metrics,
            "approval": {
                "by": metadata.approved_by,
                "at": metadata.approved_at,
            },
        }
    
    def compare_versions(self, v1_id: str, v2_id: str) -> Dict:
        """对比两个版本"""
        v1 = self.versions.get(v1_id)
        v2 = self.versions.get(v2_id)
        m1 = self.metadata.get(v1_id)
        m2 = self.metadata.get(v2_id)
        
        if not all([v1, v2, m1, m2]):
            return {"error": "版本不存在"}
        
        diff = {
            "version_1": v1_id,
            "version_2": v2_id,
            "configuration_diff": {},
            "metadata_diff": {},
        }
        
        # 配置差异
        if v1.system_prompt != v2.system_prompt:
            diff["configuration_diff"]["system_prompt"] = {
                "changed": True,
                "length_diff": len(v2.system_prompt) - len(v1.system_prompt),
            }
        
        if v1.model != v2.model:
            diff["configuration_diff"]["model"] = {
                "old": v1.model,
                "new": v2.model,
            }
        
        # 元数据差异
        if m1.status != m2.status:
            diff["metadata_diff"]["status"] = {
                "old": m1.status.value,
                "new": m2.status.value,
            }
        
        # 性能对比
        if m1.performance_metrics and m2.performance_metrics:
            diff["performance_comparison"] = {}
            for metric in set(m1.performance_metrics.keys()) & set(m2.performance_metrics.keys()):
                diff["performance_comparison"][metric] = {
                    "v1": m1.performance_metrics[metric],
                    "v2": m2.performance_metrics[metric],
                    "change": m2.performance_metrics[metric] - m1.performance_metrics[metric],
                }
        
        return diff
    
    def list_versions_by_status(self, status: VersionStatus) -> List[Dict]:
        """按状态列出版本"""
        result = []
        
        for version_id, metadata in self.metadata.items():
            if metadata.status == status:
                version = self.versions.get(version_id)
                result.append({
                    "id": version_id,
                    "name": version.name if version else "Unknown",
                    "author": metadata.author,
                    "created_at": metadata.created_at,
                })
        
        return result


# ============================================================
# 使用示例
# ============================================================

def demo_enhanced_versioning():
    """演示增强版版本管理"""
    manager = EnhancedVersionManager("./demo_enhanced_versions")
    
    print("增强版版本管理")
    print("=" * 60)
    
    # 创建版本
    v1 = manager.create_version(
        name="初始版本",
        description="基础客服 Agent",
        system_prompt="你是一个客服助手。",
        model="gpt-4o",
        author="alice",
        tags=["stable"],
        changelog=["初始版本发布"],
    )
    
    v2 = manager.create_version(
        name="优化版本",
        description="优化了 Prompt 和添加了工具",
        system_prompt="你是一个智能客服助手。先搜索知识库，再回答。",
        model="gpt-4o",
        author="bob",
        tags=["experimental"],
        changelog=["优化 Prompt", "添加搜索工具"],
        tools=["search", "faq_lookup"],
    )
    
    # 更新状态
    manager.update_status(v1.version_id, VersionStatus.PRODUCTION, 
                         approved_by="admin")
    manager.update_status(v2.version_id, VersionStatus.TESTING,
                         approved_by="alice")
    
    # 记录性能
    manager.record_performance(v1.version_id, {
        "accuracy": 0.75,
        "avg_latency_ms": 1500,
    })
    manager.record_performance(v2.version_id, {
        "accuracy": 0.82,
        "avg_latency_ms": 1200,
    })
    
    # 获取版本报告
    print("\n版本报告 (v1):")
    report = manager.get_version_report(v1.version_id)
    print(f"  名称: {report['name']}")
    print(f"  状态: {report['status']}")
    print(f"  作者: {report['author']}")
    print(f"  标签: {report['tags']}")
    print(f"  性能: {report['performance']}")
    
    # 对比版本
    print("\n版本对比:")
    diff = manager.compare_versions(v1.version_id, v2.version_id)
    print(f"  配置差异: {diff.get('configuration_diff', {})}")
    print(f"  性能对比: {diff.get('performance_comparison', {})}")
    
    # 按状态列出
    print("\n生产环境版本:")
    prod_versions = manager.list_versions_by_status(VersionStatus.PRODUCTION)
    for v in prod_versions:
        print(f"  {v['id']}: {v['name']} (by {v['author']})")


if __name__ == "__main__":
    demo_enhanced_versioning()
```

### 42.5.2 版本依赖管理

管理 Agent 版本之间的依赖关系，确保兼容性。

```python
from typing import Dict, List, Set
from dataclasses import dataclass, field
from enum import Enum

class DependencyType(Enum):
    """依赖类型"""
    REQUIRES = "requires"       # 需要依赖
    CONFLICTS_WITH = "conflicts_with"  # 冲突
    REPLACES = "replaces"       # 替代

@dataclass
class VersionDependency:
    """版本依赖"""
    source_version: str
    target_version: str
    dependency_type: DependencyType
    description: str = ""

class DependencyManager:
    """
    版本依赖管理器
    
    管理 Agent 版本之间的依赖关系。
    """
    
    def __init__(self):
        self.dependencies: List[VersionDependency] = []
    
    def add_dependency(self, source_version: str, target_version: str,
                      dependency_type: DependencyType, 
                      description: str = ""):
        """添加依赖关系"""
        dep = VersionDependency(
            source_version=source_version,
            target_version=target_version,
            dependency_type=dependency_type,
            description=description,
        )
        self.dependencies.append(dep)
    
    def check_compatibility(self, version1: str, version2: str) -> Dict:
        """检查两个版本是否兼容"""
        conflicts = []
        
        for dep in self.dependencies:
            if dep.dependency_type == DependencyType.CONFLICTS_WITH:
                if (dep.source_version in [version1, version2] and
                    dep.target_version in [version1, version2]):
                    conflicts.append(dep)
        
        return {
            "compatible": len(conflicts) == 0,
            "conflicts": [
                {
                    "between": [dep.source_version, dep.target_version],
                    "reason": dep.description,
                }
                for dep in conflicts
            ],
        }
    
    def get_dependencies(self, version_id: str) -> Dict:
        """获取版本的所有依赖"""
        required_by = []
        conflicts_with = []
        replaces = []
        
        for dep in self.dependencies:
            if dep.source_version == version_id:
                if dep.dependency_type == DependencyType.REQUIRES:
                    required_by.append(dep.target_version)
                elif dep.dependency_type == DependencyType.CONFLICTS_WITH:
                    conflicts_with.append(dep.target_version)
                elif dep.dependency_type == DependencyType.REPLACES:
                    replaces.append(dep.target_version)
        
        return {
            "requires": required_by,
            "conflicts_with": conflicts_with,
            "replaces": replaces,
        }
    
    def validate_version(self, version_id: str, 
                        available_versions: List[str]) -> Dict:
        """验证版本是否可以部署"""
        deps = self.get_dependencies(version_id)
        issues = []
        
        # 检查所有依赖是否满足
        for required in deps["requires"]:
            if required not in available_versions:
                issues.append(f"缺少依赖: {required}")
        
        # 检查冲突
        for conflict in deps["conflicts_with"]:
            if conflict in available_versions:
                issues.append(f"存在冲突: {conflict}")
        
        return {
            "version": version_id,
            "deployable": len(issues) == 0,
            "issues": issues,
        }


# ============================================================
# 使用示例
# ============================================================

def demo_dependency_management():
    """演示依赖管理"""
    manager = DependencyManager()
    
    # 添加依赖关系
    manager.add_dependency(
        "v2_beta", "v1_stable",
        DependencyType.REQUIRES,
        "v2 需要 v1 的数据格式"
    )
    
    manager.add_dependency(
        "v2_beta", "v1_legacy",
        DependencyType.CONFLICTS_WITH,
        "v2 不兼容 v1 的旧 API"
    )
    
    manager.add_dependency(
        "v2_stable", "v2_beta",
        DependencyType.REPLACES,
        "v2 正式版替代 beta 版"
    )
    
    print("依赖管理测试")
    print("=" * 60)
    
    # 获取依赖
    print("\nv2_beta 的依赖:")
    deps = manager.get_dependencies("v2_beta")
    print(f"  需要: {deps['requires']}")
    print(f"  冲突: {deps['conflicts_with']}")
    
    # 检查兼容性
    print("\n兼容性检查 (v2_beta, v1_stable):")
    compat = manager.check_compatibility("v2_beta", "v1_stable")
    print(f"  兼容: {compat['compatible']}")
    
    print("\n兼容性检查 (v2_beta, v1_legacy):")
    compat = manager.check_compatibility("v2_beta", "v1_legacy")
    print(f"  兼容: {compat['compatible']}")
    if compat['conflicts']:
        for conflict in compat['conflicts']:
            print(f"  冲突原因: {conflict['reason']}")
    
    # 验证版本
    print("\n部署验证 (v2_beta):")
    available = ["v1_stable", "v2_beta", "v2_stable"]
    validation = manager.validate_version("v2_beta", available)
    print(f"  可部署: {validation['deployable']}")
    if validation['issues']:
        for issue in validation['issues']:
            print(f"  问题: {issue}")


if __name__ == "__main__":
    demo_dependency_management()
```

## 42.6 常见坑与最佳实践

**坑 1：测试时间太短。** 只跑了几小时就下结论。Agent 的使用模式可能在一天内有显著变化（工作时间 vs 非工作时间），至少需要覆盖一个完整的工作周。

**坑 2：样本分配不均匀。** 如果 A/B 测试的两组用户特征差异很大（比如一组主要是新用户），测试结果就不可靠。使用随机分配确保两组的用户特征相似。

**坑 3：只看平均分。** 平均分可能掩盖了严重的长尾问题。新版本的平均分可能更高，但对某些特定类型的问题可能表现更差。要按查询类型细分分析。

**坑 4：版本元数据不完整。** 版本信息不完整会导致后续难以追踪变更原因。确保记录完整的变更日志和作者信息。

**坑 5：忽略回滚测试。** 在发布新版本之前，先测试回滚流程是否顺畅。确保能在出现问题时快速回滚。

**最佳实践 1：每次只测试一个变量。** 如果你同时修改了 Prompt 和模型，就无法判断是哪个改动带来了效果变化。一次 A/B 测试只对比一个因素的差异。

**最佳实践 2：建立版本审批流程。** 重要版本需要经过代码审查和审批才能发布。这可以防止低质量的变更进入生产环境。

**最佳实践 3：使用语义化版本号。** 采用 MAJOR.MINOR.PATCH 的版本号格式，让用户清楚每个版本变更的性质。

**最佳实践 4：保持版本历史完整。** 不要删除旧版本，而是标记为已废弃。这样可以随时回滚到任何历史版本。

---

## 42.6 练习题

**练习 1：实现版本对比工具。** 编写一个工具，能并排对比两个 Agent 版本的 Prompt 差异，高亮显示修改的部分。

**练习 2：设计多臂赌博机策略。** 实现一个 Thompson Sampling 策略，自动将更多流量分配给表现更好的 Agent 版本。

**练习 3：实现样本量计算器。** 编写一个工具，根据期望的效应量和置信水平，计算 A/B 测试所需的最小样本量。

**练习 4：设计特性标记系统。** 实现一个 Feature Flag 系统，支持按用户 ID、地域、时间等条件控制新功能的发布。

**练习 5：实现自动回滚。** 当灰度部署中的错误率超过阈值时，自动触发回滚操作。

**练习 6：构建实验报告生成器。** 自动从 A/B 测试数据生成实验报告，包括统计检验结果、效应量、置信区间和建议。

---

## 42.7 本章小结

本章系统性地探讨了 Agent 的版本管理与 A/B 测试。我们构建了完整的版本管理器来追踪 Prompt、模型配置和工具集的变化；实现了基于统计学的 A/B 测试框架来验证改进效果；设计了灰度发布策略来安全地推出新版本。

核心理念是：数据驱动的迭代。每一次改进都应该有数据支撑，每一次发布都应该可回滚。不要靠直觉判断"新版本更好"，用 A/B 测试的数据来证明。

---

> **下一章预告：** 第 43 章将深入 Agent 的可扩展性设计，学习如何让 Agent 从单机部署扩展到高并发的集群架构。
