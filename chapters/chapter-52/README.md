# 第52章：Agent 与数据库 —— Text-to-SQL 与数据分析

## 学习目标

通过本章的学习，你将能够：
1. 理解 Text-to-SQL 技术的原理和实现
2. 掌握 Agent 与数据库交互的安全方法
3. 学会构建支持自然语言查询的数据库 Agent
4. 理解 SQL 生成、执行和结果解释的完整流程
5. 掌握数据库 Agent 的安全防护措施
6. 能够构建完整的数据分析 Agent 系统

## 核心问题

数据库是企业最重要的数据存储系统，但大多数用户无法直接编写 SQL 查询。他们需要的是用自然语言描述需求，然后系统自动执行查询并返回结果。

Text-to-SQL 技术正是为了解决这个问题。但传统的 Text-to-SQL 只是简单地将自然语言转换为 SQL，缺乏对查询结果的理解和解释能力。

Agent 与数据库的结合可以让这个过程更加智能。Agent 不仅可以生成和执行 SQL，还可以：
1. 理解用户的真实意图
2. 自动优化查询性能
3. 解释查询结果
4. 处理异常情况
5. 提供数据分析建议

这就像是给数据库配了一个智能翻译官和数据分析师。

---

## 原理讲解

### Text-to-SQL 的核心挑战

将自然语言转换为 SQL 是一个复杂的问题，主要挑战包括：

**1. 语义理解**
用户说"显示所有高薪员工"，"高薪"的定义是什么？是超过平均工资？还是超过某个固定值？这需要理解业务语义。

**2. 模型映射**
用户说的"员工"对应数据库中的哪个表？"薪资"对应哪个字段？这需要理解数据库的模式。

**3. 查询构造**
如何将用户意图转换为正确的 SQL 语法？这需要掌握 SQL 语法和数据库特性。

**4. 异常处理**
查询可能返回空结果、执行出错、或者性能太差。如何优雅地处理这些情况？

### Agent 的解决方案

Agent 通过以下方式解决这些挑战：

**多步推理**：Agent 可以分解复杂查询，逐步理解用户意图。

**工具调用**：Agent 可以调用数据库工具执行查询，并根据结果调整策略。

**错误恢复**：当查询失败时，Agent 可以分析错误原因并修复。

**上下文理解**：Agent 可以利用对话历史来理解模糊的查询。

### 数据库交互架构

一个完整的数据库 Agent 架构包括：

```
用户请求 → 意图理解 → 模式分析 → SQL 生成
                                    ↓
                              执行查询 ← 安全检查
                                    ↓
                              结果处理 → 解释生成
                                    ↓
                              返回用户
```

**模式分析**：分析数据库的表结构、字段类型、索引等信息。

**SQL 生成**：根据用户意图和数据库模式生成 SQL。

**安全检查**：验证 SQL 的安全性，防止 SQL 注入等攻击。

**结果处理**：处理查询结果，包括格式化、聚合、排序等。

**解释生成**：用自然语言解释查询结果。

### 安全考虑

数据库 Agent 面临的安全挑战包括：

**SQL 注入**：恶意用户可能通过精心构造的输入来执行危险 SQL。

**权限控制**：不同用户应该有不同的数据访问权限。

**数据泄露**：Agent 可能在回答中无意泄露敏感数据。

**资源保护**：防止执行消耗过多资源的查询。

### 查询优化

Agent 可以帮助优化查询性能：

**索引建议**：分析查询模式，建议创建合适的索引。

**查询重写**：将低效的查询重写为高效的形式。

**缓存策略**：对于重复查询，建议使用缓存。

**分页处理**：对于大量结果，自动添加分页。

---

## 完整代码示例

### 示例 1：基础 Text-to-SQL Agent

```python
"""
基础 Text-to-SQL Agent
将自然语言转换为 SQL 并执行
"""

import json
import os
import sqlite3
from typing import Dict, List, Optional, Tuple
from dataclasses import dataclass, field
import anthropic


@dataclass
class TableSchema:
    """表结构"""
    table_name: str
    columns: List[Dict]
    description: str = ""
    sample_data: List[Dict] = field(default_factory=list)


class DatabaseManager:
    """数据库管理器"""
    
    def __init__(self, db_path: str = ":memory:"):
        self.conn = sqlite3.connect(db_path)
        self.conn.row_factory = sqlite3.Row
        self.tables: Dict[str, TableSchema] = {}
    
    def create_sample_database(self):
        """创建示例数据库"""
        cursor = self.conn.cursor()
        
        # 创建员工表
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS employees (
                id INTEGER PRIMARY KEY,
                name TEXT NOT NULL,
                department TEXT,
                position TEXT,
                salary REAL,
                hire_date TEXT,
                email TEXT
            )
        """)
        
        # 创建部门表
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS departments (
                id INTEGER PRIMARY KEY,
                name TEXT NOT NULL,
                manager TEXT,
                budget REAL
            )
        """)
        
        # 创建项目表
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS projects (
                id INTEGER PRIMARY KEY,
                name TEXT NOT NULL,
                department TEXT,
                start_date TEXT,
                end_date TEXT,
                status TEXT,
                budget REAL
            )
        """)
        
        # 插入示例数据
        employees = [
            (1, "张三", "技术部", "高级工程师", 25000, "2020-01-15", "zhangsan@company.com"),
            (2, "李四", "技术部", "工程师", 18000, "2021-03-20", "lisi@company.com"),
            (3, "王五", "市场部", "市场经理", 22000, "2019-06-10", "wangwu@company.com"),
            (4, "赵六", "财务部", "会计", 15000, "2022-01-05", "zhaoliu@company.com"),
            (5, "孙七", "技术部", "实习生", 8000, "2023-09-01", "sunqi@company.com"),
            (6, "周八", "市场部", "市场专员", 12000, "2022-06-15", "zhouba@company.com"),
            (7, "吴九", "人事部", "HR经理", 20000, "2018-11-20", "wujiu@company.com"),
            (8, "郑十", "技术部", "技术总监", 35000, "2015-03-01", "zhengshi@company.com"),
        ]
        cursor.executemany("INSERT OR IGNORE INTO employees VALUES (?,?,?,?,?,?,?)", employees)
        
        departments = [
            (1, "技术部", "郑十", 500000),
            (2, "市场部", "王五", 300000),
            (3, "财务部", "赵六", 200000),
            (4, "人事部", "吴九", 150000),
        ]
        cursor.executemany("INSERT OR IGNORE INTO departments VALUES (?,?,?,?)", departments)
        
        projects = [
            (1, "项目A", "技术部", "2023-01-01", "2023-06-30", "completed", 100000),
            (2, "项目B", "技术部", "2023-03-01", "2023-12-31", "in_progress", 200000),
            (3, "项目C", "市场部", "2023-06-01", "2023-09-30", "completed", 80000),
            (4, "项目D", "技术部", "2024-01-01", "2024-06-30", "planned", 150000),
        ]
        cursor.executemany("INSERT OR IGNORE INTO projects VALUES (?,?,?,?,?,?,?)", projects)
        
        self.conn.commit()
        
        # 记录表结构
        self._analyze_schema()
    
    def _analyze_schema(self):
        """分析数据库模式"""
        cursor = self.conn.cursor()
        
        # 获取所有表
        cursor.execute("SELECT name FROM sqlite_master WHERE type='table'")
        tables = cursor.fetchall()
        
        for table_name in tables:
            table_name = table_name[0]
            
            # 获取列信息
            cursor.execute(f"PRAGMA table_info({table_name})")
            columns = [
                {"name": col[1], "type": col[2], "nullable": not col[3]}
                for col in cursor.fetchall()
            ]
            
            # 获取样本数据
            cursor.execute(f"SELECT * FROM {table_name} LIMIT 3")
            sample_data = [dict(row) for row in cursor.fetchall()]
            
            self.tables[table_name] = TableSchema(
                table_name=table_name,
                columns=columns,
                sample_data=sample_data,
            )
    
    def execute_query(self, sql: str) -> Tuple[List[Dict], str]:
        """执行 SQL 查询"""
        try:
            cursor = self.conn.cursor()
            cursor.execute(sql)
            
            if cursor.description:
                columns = [desc[0] for desc in cursor.description]
                results = [dict(zip(columns, row)) for row in cursor.fetchall()]
                return results, "success"
            else:
                return [], "no_results"
                
        except Exception as e:
            return [], f"error: {str(e)}"
    
    def get_schema_description(self) -> str:
        """获取模式描述"""
        desc = "数据库表结构：\n\n"
        for table_name, schema in self.tables.items():
            desc += f"表: {table_name}\n"
            desc += "列: "
            desc += ", ".join([f"{col['name']} ({col['type']})" for col in schema.columns])
            desc += "\n"
            if schema.sample_data:
                desc += f"示例数据: {json.dumps(schema.sample_data[:2], ensure_ascii=False)}\n"
            desc += "\n"
        return desc


class TextToSQLAgent:
    """Text-to-SQL Agent"""
    
    def __init__(self):
        self.client = anthropic.Anthropic(
            api_key=os.environ.get("ANTHROPIC_API_KEY")
        )
        self.db = DatabaseManager()
        self.db.create_sample_database()
        self.conversation_history: List[Dict] = []
    
    def query(self, natural_language: str) -> Dict:
        """处理自然语言查询"""
        print(f"\n{'='*60}")
        print(f"用户查询: {natural_language}")
        print(f"{'='*60}")
        
        # Step 1: 理解用户意图
        print("\n[Step 1] 理解用户意图...")
        intent = self._understand_intent(natural_language)
        print(f"  意图: {intent}")
        
        # Step 2: 生成 SQL
        print("\n[Step 2] 生成 SQL...")
        sql = self._generate_sql(natural_language, intent)
        print(f"  SQL: {sql}")
        
        # Step 3: 安全检查
        print("\n[Step 3] 安全检查...")
        is_safe = self._check_safety(sql)
        if not is_safe:
            return {"error": "SQL 不安全，已拒绝执行"}
        print("  安全检查通过")
        
        # Step 4: 执行查询
        print("\n[Step 4] 执行查询...")
        results, status = self.db.execute_query(sql)
        print(f"  状态: {status}, 结果数: {len(results)}")
        
        # Step 5: 生成解释
        print("\n[Step 5] 生成解释...")
        explanation = self._generate_explanation(
            natural_language, sql, results, status
        )
        
        # 记录对话历史
        self.conversation_history.append({
            "query": natural_language,
            "sql": sql,
            "results": results[:5],  # 只保存前5条
            "explanation": explanation,
        })
        
        return {
            "query": natural_language,
            "sql": sql,
            "results": results,
            "explanation": explanation,
            "status": status,
        }
    
    def _understand_intent(self, query: str) -> Dict:
        """理解用户意图"""
        prompt = f"""分析以下用户查询的意图。

用户查询：{query}

数据库模式：
{self.db.get_schema_description()}

请用 JSON 格式回答：
{{
    "intent": "查询/统计/分析",
    "tables": ["涉及的表"],
    "conditions": ["筛选条件"],
    "aggregations": ["聚合操作"],
    "sort_by": "排序字段",
    "limit": "限制数量"
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
        
        return {"intent": "查询", "tables": [], "conditions": []}
    
    def _generate_sql(self, query: str, intent: Dict) -> str:
        """生成 SQL"""
        prompt = f"""将以下自然语言查询转换为 SQL。

用户查询：{query}
意图分析：{json.dumps(intent, ensure_ascii=False)}

数据库模式：
{self.db.get_schema_description()}

请只返回 SQL 语句，不要包含其他内容。"""
        
        response = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            messages=[{"role": "user", "content": prompt}],
        )
        
        sql = response.content[0].text.strip()
        
        # 清理 SQL
        if sql.startswith("```sql"):
            sql = sql[6:]
        if sql.endswith("```"):
            sql = sql[:-3]
        sql = sql.strip()
        
        return sql
    
    def _check_safety(self, sql: str) -> bool:
        """安全检查"""
        # 检查危险操作
        dangerous_keywords = ["DROP", "DELETE", "UPDATE", "INSERT", "ALTER", "TRUNCATE"]
        sql_upper = sql.upper()
        
        for keyword in dangerous_keywords:
            if keyword in sql_upper:
                print(f"  警告: 检测到危险操作 {keyword}")
                return False
        
        # 检查是否是 SELECT 查询
        if not sql_upper.strip().startswith("SELECT"):
            print("  警告: 只允许 SELECT 查询")
            return False
        
        return True
    
    def _generate_explanation(self, query: str, sql: str, 
                              results: List[Dict], status: str) -> str:
        """生成解释"""
        if status != "success":
            return f"查询执行失败: {status}"
        
        if not results:
            return "查询没有返回结果"
        
        prompt = f"""用自然语言解释以下查询结果。

原始查询：{query}
执行的 SQL：{sql}
查询结果（前5条）：{json.dumps(results[:5], ensure_ascii=False, indent=2)}
总结果数：{len(results)}

请提供：
1. 结果的简要总结
2. 关键发现
3. 如果有意义，提供数据洞察"""
        
        response = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            messages=[{"role": "user", "content": prompt}],
        )
        
        return response.content[0].text


def demo_text_to_sql():
    """演示 Text-to-SQL"""
    print("=" * 60)
    print("Text-to-SQL Agent 演示")
    print("=" * 60)
    
    agent = TextToSQLAgent()
    
    # 测试查询
    queries = [
        "显示所有员工的信息",
        "技术部有哪些员工？",
        "工资最高的3名员工是谁？",
        "每个部门的平均工资是多少？",
        "2020年之后入职的员工有哪些？",
    ]
    
    for query in queries:
        result = agent.query(query)
        print(f"\n结果摘要: {result['explanation'][:200]}...")
        print("-" * 40)


if __name__ == "__main__":
    demo_text_to_sql()
```

### 示例 2：数据分析 Agent

```python
"""
数据分析 Agent
自动分析数据并提供洞察
"""

import json
import os
from typing import Dict, List, Optional
from dataclasses import dataclass, field
import anthropic


@dataclass
class AnalysisResult:
    """分析结果"""
    analysis_type: str
    summary: str
    findings: List[str]
    recommendations: List[str]
    visualization_suggestions: List[str]


class DataAnalysisAgent:
    """数据分析 Agent"""
    
    def __init__(self):
        self.client = anthropic.Anthropic(
            api_key=os.environ.get("ANTHROPIC_API_KEY")
        )
    
    def analyze(self, data: List[Dict], question: str = "") -> AnalysisResult:
        """分析数据"""
        print(f"\n数据分析开始...")
        print(f"数据量: {len(data)} 条")
        
        # Step 1: 数据概览
        print("\n[Step 1] 数据概览...")
        overview = self._get_data_overview(data)
        print(f"  字段: {overview['columns']}")
        print(f"  数据类型: {overview['types']}")
        
        # Step 2: 统计分析
        print("\n[Step 2] 统计分析...")
        statistics = self._calculate_statistics(data, overview)
        
        # Step 3: 异常检测
        print("\n[Step 3] 异常检测...")
        anomalies = self._detect_anomalies(data, statistics)
        
        # Step 4: 模式识别
        print("\n[Step 4] 模式识别...")
        patterns = self._identify_patterns(data, overview)
        
        # Step 5: 生成洞察
        print("\n[Step 5] 生成洞察...")
        insights = self._generate_insights(
            overview, statistics, anomalies, patterns, question
        )
        
        return AnalysisResult(
            analysis_type="comprehensive",
            summary=insights.get("summary", ""),
            findings=insights.get("findings", []),
            recommendations=insights.get("recommendations", []),
            visualization_suggestions=insights.get("visualizations", []),
        )
    
    def _get_data_overview(self, data: List[Dict]) -> Dict:
        """获取数据概览"""
        if not data:
            return {"columns": [], "types": {}, "row_count": 0}
        
        columns = list(data[0].keys())
        types = {}
        
        for col in columns:
            values = [row.get(col) for row in data if row.get(col) is not None]
            if values:
                if isinstance(values[0], (int, float)):
                    types[col] = "numeric"
                elif isinstance(values[0], bool):
                    types[col] = "boolean"
                else:
                    types[col] = "text"
            else:
                types[col] = "unknown"
        
        return {
            "columns": columns,
            "types": types,
            "row_count": len(data),
        }
    
    def _calculate_statistics(self, data: List[Dict], overview: Dict) -> Dict:
        """计算统计信息"""
        statistics = {}
        
        for col, col_type in overview["types"].items():
            values = [row.get(col) for row in data if row.get(col) is not None]
            
            if col_type == "numeric" and values:
                statistics[col] = {
                    "mean": sum(values) / len(values),
                    "min": min(values),
                    "max": max(values),
                    "count": len(values),
                }
            elif col_type == "text" and values:
                from collections import Counter
                value_counts = Counter(values)
                statistics[col] = {
                    "unique_count": len(value_counts),
                    "top_values": value_counts.most_common(5),
                }
        
        return statistics
    
    def _detect_anomalies(self, data: List[Dict], statistics: Dict) -> List[Dict]:
        """检测异常"""
        anomalies = []
        
        for col, stats in statistics.items():
            if "mean" in stats and "std" in stats:
                # 简单的异常检测：超过 2 个标准差
                mean = stats["mean"]
                std = stats.get("std", 0)
                
                if std > 0:
                    for row in data:
                        value = row.get(col)
                        if value is not None and abs(value - mean) > 2 * std:
                            anomalies.append({
                                "column": col,
                                "value": value,
                                "type": "outlier",
                            })
        
        return anomalies
    
    def _identify_patterns(self, data: List[Dict], overview: Dict) -> List[Dict]:
        """识别模式"""
        patterns = []
        
        # 识别趋势
        numeric_cols = [col for col, t in overview["types"].items() if t == "numeric"]
        
        for col in numeric_cols:
            values = [row.get(col) for row in data if row.get(col) is not None]
            if len(values) >= 3:
                # 简单的趋势检测
                if all(values[i] <= values[i+1] for i in range(len(values)-1)):
                    patterns.append({
                        "type": "trend",
                        "column": col,
                        "direction": "increasing",
                    })
                elif all(values[i] >= values[i+1] for i in range(len(values)-1)):
                    patterns.append({
                        "type": "trend",
                        "column": col,
                        "direction": "decreasing",
                    })
        
        return patterns
    
    def _generate_insights(self, overview: Dict, statistics: Dict,
                          anomalies: List, patterns: List, question: str) -> Dict:
        """生成洞察"""
        prompt = f"""基于以下数据分析结果，生成洞察和建议。

数据概览：
- 字段: {overview['columns']}
- 数据类型: {overview['types']}
- 行数: {overview['row_count']}

统计信息：
{json.dumps(statistics, ensure_ascii=False, indent=2)}

异常值：
{json.dumps(anomalies[:10], ensure_ascii=False, indent=2)}

识别的模式：
{json.dumps(patterns, ensure_ascii=False, indent=2)}

用户问题：{question if question else "无特定问题"}

请生成：
1. 数据摘要
2. 关键发现
3. 建议
4. 可视化建议

用 JSON 格式回答。"""
        
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
        
        return {
            "summary": "数据分析完成",
            "findings": [],
            "recommendations": [],
            "visualizations": [],
        }


def demo数据分析():
    """演示数据分析"""
    print("=" * 60)
    print("数据分析 Agent 演示")
    print("=" * 60)
    
    agent = DataAnalysisAgent()
    
    # 示例数据
    sales_data = [
        {"month": "1月", "sales": 10000, "cost": 6000, "profit": 4000},
        {"month": "2月", "sales": 12000, "cost": 7000, "profit": 5000},
        {"month": "3月", "sales": 15000, "cost": 8000, "profit": 7000},
        {"month": "4月", "sales": 13000, "cost": 7500, "profit": 5500},
        {"month": "5月", "sales": 18000, "cost": 9000, "profit": 9000},
        {"month": "6月", "sales": 20000, "cost": 10000, "profit": 10000},
    ]
    
    result = agent.analyze(sales_data, "销售趋势如何？")
    
    print(f"\n分析摘要:\n{result.summary}")
    print(f"\n关键发现:")
    for finding in result.findings:
        print(f"  - {finding}")
    print(f"\n建议:")
    for rec in result.recommendations:
        print(f"  - {rec}")


if __name__ == "__main__":
    demo数据分析()
```

---

## 案例分析

### 案例 1：商业智能助手

**场景**：企业需要一个能够回答各种业务问题的智能助手。

**架构设计**：

```
用户问题 → 意图理解 → 数据源选择
              ↓
         SQL 生成 → 安全检查 → 执行查询
              ↓
         结果分析 → 可视化 → 报告生成
              ↓
         自然语言解释
```

**关键功能**：
1. **多数据源支持**：支持 SQL 数据库、NoSQL 数据库、API 等
2. **智能缓存**：对于重复查询使用缓存
3. **权限控制**：不同角色看到不同的数据
4. **审计日志**：记录所有查询操作

### 案例 2：数据库运维助手

**场景**：DBA 需要一个能够帮助监控和优化数据库的助手。

**关键功能**：
1. **性能监控**：查询数据库性能指标
2. **慢查询分析**：识别和分析慢查询
3. **索引建议**：根据查询模式建议索引
4. **容量预测**：预测存储和性能需求

---

## 常见坑

### 坑 1：SQL 注入风险

用户输入可能包含恶意 SQL。

**解决方案**：使用参数化查询，严格验证输入。

### 坑 2：性能问题

复杂查询可能消耗大量资源。

**解决方案**：设置查询超时，限制结果数量。

### 坑 3：模式变化

数据库模式可能随时间变化。

**解决方案**：定期更新模式信息，使用动态模式分析。

### 坑 4：权限管理

不同用户有不同的数据访问权限。

**解决方案**：实现行级和列级权限控制。

### 坑 5：结果解释不准确

LLM 可能错误解释查询结果。

**解决方案**：验证解释的准确性，提供原始数据参考。

---

## 练习题

### 练习 1：基础 Text-to-SQL
实现一个基础的 Text-to-SQL 系统，支持：
- 简单的 SELECT 查询
- WHERE 条件过滤
- ORDER BY 排序

### 练习 2：聚合查询
扩展 Text-to-SQL 系统，支持：
- GROUP BY 分组
- 聚合函数（SUM, AVG, COUNT）
- HAVING 过滤

### 练习 3：安全检查
实现 SQL 安全检查机制：
- 检测危险操作
- 防止 SQL 注入
- 权限验证

### 练习 4：结果可视化
为查询结果添加可视化：
- 表格展示
- 图表生成
- 导出功能

### 练习 5：多表查询
支持复杂的多表查询：
- JOIN 操作
- 子查询
- UNION 操作

### 练习 6：性能优化
优化查询性能：
- 查询缓存
- 索引建议
- 查询重写

---

## 实战任务

### 任务 1：构建企业数据助手（中等难度）

**功能需求：**
1. 支持自然语言查询
2. 支持多数据源
3. 实现权限控制
4. 生成可视化报告

**技术要求：**
1. 使用 Text-to-SQL 技术
2. 实现安全检查
3. 支持查询缓存
4. 生成审计日志

### 任务 2：构建数据库运维助手（高难度）

**功能需求：**
1. 性能监控
2. 慢查询分析
3. 索引优化建议
4. 容量预测

**技术要求：**
1. 连接多种数据库
2. 实现实时监控
3. 支持告警通知
4. 生成运维报告

---

## 本章小结

本章我们深入学习了 Agent 与数据库的结合，特别是 Text-to-SQL 技术。我们从以下几个方面进行了探索：

**核心挑战**：理解了 Text-to-SQL 面临的语义理解、模型映射、查询构造和异常处理等挑战。

**Agent 解决方案**：掌握了 Agent 如何通过多步推理、工具调用、错误恢复和上下文理解来解决这些挑战。

**架构设计**：理解了从意图理解到结果解释的完整流程。

**安全考虑**：学会了 SQL 注入防护、权限控制和数据保护等安全措施。

**查询优化**：掌握了索引建议、查询重写和缓存策略等优化方法。

Agent 与数据库的结合让用户可以用自然语言与数据交互，大大降低了数据分析的门槛。随着 Text-to-SQL 技术的进步，这种交互方式将变得越来越智能和可靠。

在下一章中，我们将学习 Agent 与代码的结合，看看如何让 Agent 生成、审查和执行代码。

---

## 延伸阅读

1. Text-to-SQL 综述论文
2. Spider: A Large-Scale Human-Labeled Text-to-SQL Dataset
3. SQLCoder: A State-of-the-Art LLM for Text-to-SQL
4. Database Agent 研究方向
5. Natural Language Interfaces to Databases (研究领域)
