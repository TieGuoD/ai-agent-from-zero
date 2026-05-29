# AI Agent 系统工程：从基础到生产

> 一本面向工程师的 AI Agent 系统级教程，从 LLM 原理到生产级 Agent 系统的完整知识体系。

**本书由 Claude Code（Anthropic 官方 CLI 工具）书写。**

## 书籍定位

本书不是浅层科普，而是一套完整的系统级教程。读完后你将：

- 理解 LLM、Prompt、Tool Calling、RAG、Memory、Planning、Workflow、Multi-Agent 的底层逻辑
- 理解 Agent = Model + Tools + Memory + State + Context + Runtime + Guardrails + Evaluation
- 能判断什么时候应该用 Agent，什么时候应该用普通 Workflow
- 理解 OpenClaw、Hermes 等前沿框架的架构思想
- 理解 Harness Engineering 为什么成为 2026 年 Agent 工程的新关键词
- 能设计安全、可观测、可评估、可部署的生产级 Agent 系统

## 仓库结构

```
ai-agent-from-zero/
└── chapters/          # 每章的学习笔记和总结（共 70 章）
```

## 全书目录

### 第一部分：基础认知（第 1-10 章）
1. LLM 的本质 —— 从语言模型到智能体的第一步
2. Prompt Engineering —— Agent 的语言接口
3. LLM API 工程 —— 从调用到生产
4. Tool Calling —— Agent 的手脚
5. Agent 的本质 —— LLM + 循环 + 工具
6. Agent 的分类与适用场景
7. Agent 的认知架构 —— ReAct 与推理模式
8. Agent 的状态管理 —— 对话历史与上下文
9. Agent 的错误处理与容错
10. 构建你的第一个完整 Agent

### 第二部分：核心能力（第 11-22 章）
11. RAG 基础 —— 检索增强生成的原理
12. RAG 进阶 —— 检索质量优化
13. Memory 系统 —— Agent 的记忆机制
14. Memory 进阶 —— 向量记忆与语义搜索
15. Planning —— Agent 的规划能力
16. Planning 进阶 —— 层次化规划与动态调整
17. Agent 推理模式深入 —— 从 ReAct 到 Reflexion
18. Tool Use 进阶 —— 动态工具与工具学习
19. Output Parsing —— 结构化输出与解析
20. Agent 的上下文管理 —— 窗口限制的工程对策
21. LangChain 基础 —— 框架的架构思想
22. LangGraph —— 图驱动的 Agent 编排

### 第三部分：Multi-Agent 系统（第 23-30 章）
23. Multi-Agent 基础 —— 为什么需要多个 Agent
24. Multi-Agent 通信 —— 消息传递与共享状态
25. Multi-Agent 协调 —— 策略与模式
26. 角色化 Multi-Agent —— 专业分工
27. Agent 辩论与自我纠正
28. AutoGen 框架 —— 对话驱动的 Multi-Agent
29. CrewAI 框架 —— 协作型 Multi-Agent
30. Multi-Agent 系统设计 —— 架构与权衡

### 第四部分：工程化（第 31-44 章）
31. Agent 可观测性 —— 理解 Agent 在做什么
32. LangSmith 与追踪实践
33. Agent 评估体系 —— 如何知道 Agent 做得好不好
34. Agent 单元测试 —— 测试 Agent 的代码部分
35. Agent 安全基础 —— 风险与防护
36. Agent 安全进阶 —— 沙箱与权限控制
37. Agent 日志与审计 —— 合规与追溯
38. Agent 成本优化 —— Token 经济学
39. Agent 调试 —— 当 Agent 不按预期工作
40. Agent 部署 —— 从开发到生产
41. Agent 监控与告警 —— 生产环境的守护者
42. Agent 版本管理与 A/B 测试
43. Agent 的可扩展性设计 —— 从单机到集群
44. Workflow vs Agent —— 选型决策框架

### 第五部分：前沿框架与思想（第 45-56 章）
45. OpenAI Assistants API —— 官方 Agent 实现
46. Anthropic Claude 工具使用 —— Claude 的 Agent 能力
47. MCP 协议 —— Agent 的标准化接口
48. 自学习 Agent —— Hermes 与学习循环
49. Harness Engineering —— Agent 工程的新范式
50. OpenClaw —— 个人数字员工框架分析
51. Agentic RAG —— Agent 驱动的检索增强
52. Agent 与数据库 —— Text-to-SQL 与数据分析
53. Agent 与代码 —— 代码生成、审查与执行
54. Agent 与浏览器 —— Web Agent
55. Agent 与文件系统 —— 文件操作 Agent
56. Agent 框架对比与选型指南

### 第六部分：实战项目（第 57-66 章）
57. 项目 1 —— 研究助手 Agent
58. 项目 2 —— PDF 分析 Agent
59. 项目 3 —— 邮件 Agent
60. 项目 4 —— 代码 Agent
61. 项目 5 —— 个人数字员工
62. 项目 6 —— 可学习 Agent（Hermes 风格）
63. 项目实战总结 —— 6 个项目的架构对比
64. 项目优化 —— 从原型到生产
65. 项目展示 —— 如何呈现你的 Agent 项目
66. 开源贡献 —— 为 Agent 生态做贡献

### 第七部分：生产与商业化（第 67-70 章）
67. Agent 产品设计 —— 从技术到产品
68. Agent 商业化 —— 模式与案例
69. Agent 的伦理与治理
70. Agent 的未来 —— 2026-2028 展望

## 核心公式

```
Agent = Model + Tools + Memory + State + Context + Runtime + Guardrails + Evaluation
```

## 许可证

MIT License
