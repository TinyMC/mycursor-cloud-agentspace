# AI 产品经理 6 个月学习路径图

> 从零到一，系统构建 AI 产品经理的核心能力。

---

## 目录

- [导读](#导读)
- [第 1 月：AI/ML 基础认知](#第-1-月aiml-基础认知)
- [第 2 月：数据思维与产品分析](#第-2-月数据思维与产品分析)
- [第 3 月：NLP 与大模型应用](#第-3-月nlp-与大模型应用)
- [第 4 月：AI 产品设计与用户体验](#第-4-月ai-产品设计与用户体验)
- [第 5 月：AI 产品战略与商业化](#第-5-月ai-产品战略与商业化)
- [第 6 月：综合实战与作品集](#第-6-月综合实战与作品集)
- [补充板块](#补充板块)
  - [推荐工具栈](#推荐工具栈)
  - [精选书单](#精选书单)
  - [精选课程](#精选课程)
  - [模型使用建议](#模型使用建议)
  - [AI/ML 核心术语表](#aiml-核心术语表)

---

## 导读

### 适用人群

| 人群 | 说明 |
|---|---|
| 转型 AI PM | 有互联网产品经验，希望进入 AI 赛道 |
| 传统 PM 补充 AI 技能 | 当前工作中已涉及 AI 功能，需要体系化知识 |
| 应届生 / 求职者 | 目标岗位为 AI 产品经理，需要从头建立知识体系 |

### 使用方式

- **节奏**：每月一个阶段，每周投入 **8–10 小时**
- **方式**：理论学习 + 动手实践 + 输出交付物
- **建议**：每周末花 30 分钟回顾本周笔记，每月末完成交付物并自我复盘

### 学习产出总览

| 月份 | 核心交付物 |
|---|---|
| 第 1 月 | AI/ML 概念笔记 + 模型分类思维导图 |
| 第 2 月 | 数据分析报告（基于公开数据集） |
| 第 3 月 | LLM 应用 Demo + Prompt 工程手册 |
| 第 4 月 | AI 产品交互设计文档 |
| 第 5 月 | AI 产品商业计划书 |
| 第 6 月 | 完整 PRD + 作品集 + 模拟面试记录 |

---

## 第 1 月：AI/ML 基础认知

> **目标**：建立对机器学习和深度学习的技术直觉，能与工程师进行有效沟通。

### 周计划

| 周次 | 主题 | 学习内容 |
|---|---|---|
| 第 1 周 | 机器学习概览 | 什么是 ML、监督/无监督/强化学习、常见算法（线性回归、决策树、SVM）、过拟合与欠拟合 |
| 第 2 周 | 深度学习直觉 | 神经网络基本结构、前向传播与反向传播（直觉理解）、CNN / RNN 的应用场景 |
| 第 3 周 | 模型训练与评估 | 训练集/验证集/测试集、损失函数、准确率/精确率/召回率/F1、混淆矩阵 |
| 第 4 周 | 行业全景 | AI 在各行业的落地案例（金融、医疗、电商、教育）、当前 AI 技术边界与局限性 |

### 推荐资源

- **课程**：吴恩达《Machine Learning Specialization》(Coursera)
- **书籍**：《百面机器学习》——适合快速建立概念框架
- **工具**：Google Colab（免费 GPU 环境，跑通第一个模型）
- **文章**：Google 的 "Machine Learning Crash Course"

### 本月交付物

- [ ] AI/ML 核心概念笔记（Markdown 或 Notion）
- [ ] 常见模型分类思维导图
- [ ] 在 Colab 中跑通一个分类模型（如鸢尾花分类）

---

## 第 2 月：数据思维与产品分析

> **目标**：掌握数据驱动决策的方法论，能够独立设计实验和解读数据。

### 周计划

| 周次 | 主题 | 学习内容 |
|---|---|---|
| 第 1 周 | 数据基础 | 数据类型、数据清洗、缺失值处理、数据管道（ETL）概念 |
| 第 2 周 | 产品指标体系 | 北极星指标、DAU/MAU、留存率、LTV/CAC、漏斗分析 |
| 第 3 周 | A/B 测试 | 假设检验、样本量计算、显著性水平、常见陷阱（辛普森悖论、新奇效应） |
| 第 4 周 | 特征工程与数据产品 | 特征工程概念、数据标注流程、数据质量对模型的影响 |

### 推荐资源

- **课程**：《Lean Analytics》配套在线课程
- **书籍**：《精益数据分析》(Lean Analytics)、《数据化运营》
- **工具**：SQL 基础（Mode Analytics 教程）、Python Pandas、Mixpanel / Amplitude（产品分析平台）
- **实践数据集**：Kaggle Datasets、Google Dataset Search

### 本月交付物

- [ ] 针对一个公开数据集的完整分析报告（含可视化）
- [ ] 为一款假想 AI 产品设计指标体系
- [ ] 一份 A/B 测试方案设计文档

---

## 第 3 月：NLP 与大模型应用

> **目标**：深入理解大语言模型的能力与局限，掌握 Prompt Engineering，了解主流 LLM 产品模式。

### 周计划

| 周次 | 主题 | 学习内容 |
|---|---|---|
| 第 1 周 | NLP 基础 | 分词、词向量（Word2Vec / Embedding）、文本分类、情感分析 |
| 第 2 周 | Transformer 与 LLM | Attention 机制直觉、GPT / BERT 架构对比、预训练与微调、Scaling Law |
| 第 3 周 | Prompt Engineering | Prompt 设计原则、System Prompt / Few-shot / Chain-of-Thought / ReAct、Prompt 迭代与评测 |
| 第 4 周 | LLM 产品模式 | RAG（检索增强生成）、Agent / Tool Use、Chatbot / Copilot / 自主 Agent 架构对比、多模态应用 |

### 推荐资源

- **课程**：DeepLearning.AI《ChatGPT Prompt Engineering for Developers》、Andrej Karpathy 的 YouTube 系列
- **书籍**：《大规模语言模型：从理论到实践》
- **工具**：OpenAI Playground / API、LangChain、HuggingFace Transformers
- **文章**：Lilian Weng 的博客（lilianweng.github.io）

### 本月交付物

- [ ] 个人 Prompt 工程手册（记录有效 Prompt 模式与失败案例）
- [ ] 一个基于 LLM API 的小型应用 Demo（如文档问答 Bot）
- [ ] RAG vs Fine-tuning 的对比分析文档

---

## 第 4 月：AI 产品设计与用户体验

> **目标**：掌握 AI 产品的独特设计挑战，能够设计出兼顾用户体验与 AI 能力的产品方案。

### 周计划

| 周次 | 主题 | 学习内容 |
|---|---|---|
| 第 1 周 | 人机交互模式 | AI 产品的交互范式（对话式、推荐式、辅助式、自动化）、输入/输出设计 |
| 第 2 周 | 信任与透明度 | 可解释性（XAI）、置信度展示、用户预期管理、AI 错误的沟通策略 |
| 第 3 周 | 容错与不确定性设计 | Graceful Degradation、Fallback 策略、人机协作（Human-in-the-Loop）、反馈闭环设计 |
| 第 4 周 | AI 伦理与偏见 | 数据偏见来源、公平性指标、AI 伦理框架（EU AI Act 等）、负责任的 AI 产品设计 |

### 推荐资源

- **书籍**：《Designing Human-Centric AI Experiences》、Google PAIR Guidebook
- **课程**：Google PAIR（People + AI Research）在线资源
- **案例**：分析 ChatGPT / Copilot / Midjourney / Notion AI 的交互设计
- **工具**：Figma（原型设计）、Miro（用户旅程地图）

### 本月交付物

- [ ] 一份 AI 产品交互设计文档（选择一个具体场景，如 AI 客服或 AI 写作助手）
- [ ] 竞品 AI 产品的 UX 分析报告
- [ ] AI 伦理检查清单（Checklist）

---

## 第 5 月：AI 产品战略与商业化

> **目标**：具备 AI 产品的商业思维，能够进行市场分析、制定 GTM 策略并评估技术方案。

### 周计划

| 周次 | 主题 | 学习内容 |
|---|---|---|
| 第 1 周 | 市场分析 | AI 市场规模评估（TAM/SAM/SOM）、AI 行业图谱、竞品分析框架 |
| 第 2 周 | 商业模式与定价 | AI 产品的定价模型（按量 / 订阅 / Freemium / 按效果付费）、成本结构（推理成本、标注成本） |
| 第 3 周 | Build vs Buy | 自研 vs 调用第三方 API vs 开源模型部署、供应商评估框架、技术债务管理 |
| 第 4 周 | 合规与 GTM | 数据隐私法规（GDPR / 《个人信息保护法》）、AI 监管趋势、GTM 策略设计 |

### 推荐资源

- **书籍**：《AI 战略》(Competing in the Age of AI)、《启示录》(Inspired)
- **报告**：a16z "State of AI" 年度报告、McKinsey AI 报告、CB Insights AI 100
- **课程**：Reforge "AI for Product Managers" 系列
- **工具**：Crunchbase（市场研究）、SimilarWeb（竞品流量分析）

### 本月交付物

- [ ] 一份 AI 产品商业计划书（含市场分析、定价策略、GTM 方案）
- [ ] Build vs Buy 决策矩阵文档
- [ ] 一份 AI 供应商评估报告

---

## 第 6 月：综合实战与作品集

> **目标**：将前 5 个月所学整合为完整的产品方案和作品集，为求职或内部晋升做好准备。

### 周计划

| 周次 | 主题 | 学习内容 |
|---|---|---|
| 第 1 周 | 端到端 PRD | 撰写完整的 AI 产品 PRD：问题定义、用户故事、功能规格、技术方案、指标定义、里程碑 |
| 第 2 周 | 模拟产品发布 | 制定上线计划（灰度策略、监控指标、回滚方案）、准备 Launch Review 文档 |
| 第 3 周 | 作品集打磨 | 整理 6 个月的交付物、撰写项目案例（STAR 格式）、搭建个人作品集页面 |
| 第 4 周 | 面试准备 | AI PM 高频面试题（产品设计题、估算题、技术理解题）、模拟面试练习 |

### 推荐资源

- **面试准备**：Lewis C. Lin《Decode and Conquer》、Exponent PM 面试平台
- **PRD 模板**：参考 Coda / Notion 的 AI 产品 PRD 模板
- **作品集工具**：Notion（个人主页）、GitHub Pages、Read.cv
- **社区**：Product Hunt（关注 AI 新品）、即刻 / X（AI PM 圈子）

### 本月交付物

- [ ] 完整的 AI 产品 PRD（覆盖从需求到上线的全流程）
- [ ] 产品上线模拟文档
- [ ] 个人作品集（至少包含 3 个项目案例）
- [ ] 模拟面试记录与改进要点

---

## 补充板块

### 推荐工具栈

| 类别 | 工具 | 用途 |
|---|---|---|
| 编程环境 | Google Colab / Jupyter Notebook | 运行 Python 代码、体验模型训练 |
| LLM 平台 | OpenAI API / Claude API / HuggingFace | 调用和体验大语言模型 |
| 开发框架 | LangChain / LlamaIndex | 构建 RAG 和 Agent 应用 |
| 数据分析 | SQL + Pandas + Matplotlib | 数据查询、处理与可视化 |
| 产品分析 | Mixpanel / Amplitude / Google Analytics | 用户行为分析与实验 |
| 设计工具 | Figma / Miro | 原型设计与用户旅程地图 |
| 笔记与文档 | Notion / Obsidian | 知识管理与交付物输出 |
| 项目管理 | Linear / Jira | 模拟产品迭代管理 |

### 精选书单

| 书名 | 关联阶段 | 说明 |
|---|---|---|
| 《百面机器学习》 | 第 1 月 | ML 概念速览，适合非技术背景 |
| 《精益数据分析》 | 第 2 月 | 数据驱动产品决策的经典 |
| 《大规模语言模型：从理论到实践》 | 第 3 月 | 中文 LLM 技术科普 |
| 《Designing Human-Centric AI Experiences》 | 第 4 月 | AI 产品设计方法论 |
| 《AI 战略》(Competing in the Age of AI) | 第 5 月 | AI 商业化视角 |
| 《启示录》(Inspired) | 第 5-6 月 | 产品管理经典，适合贯穿全程 |
| 《Decode and Conquer》 | 第 6 月 | PM 面试准备 |

### 精选课程

| 课程 | 平台 | 关联阶段 |
|---|---|---|
| Machine Learning Specialization | Coursera (Andrew Ng) | 第 1 月 |
| ChatGPT Prompt Engineering for Developers | DeepLearning.AI | 第 3 月 |
| Google PAIR Guidebook | Google | 第 4 月 |
| Reforge "AI for Product Managers" | Reforge | 第 5 月 |
| Exponent PM Interview Course | Exponent | 第 6 月 |

### 模型使用建议

本节面向产品经理，提供主流 AI 模型的选型与使用指南。

#### 一、主流模型对比

| 模型 | 厂商 | 核心优势 | 适用场景 | 成本量级 |
|---|---|---|---|---|
| GPT-4o | OpenAI | 综合能力强、多模态 | 通用对话、内容生成、代码辅助 | 中高 |
| Claude 3.5 Sonnet | Anthropic | 长文本理解、安全性高 | 文档分析、企业级对话 | 中 |
| Gemini 1.5 Pro | Google | 超长上下文、多模态 | 长文档处理、视频理解 | 中 |
| DeepSeek-V3 | DeepSeek | 高性价比、中文能力强 | 中文场景、成本敏感型应用 | 低 |
| Llama 3 / Qwen 2.5 | Meta / 阿里 | 开源可私有化部署 | 数据合规要求高、需要定制化 | 部署成本自担 |

> **PM 要点**：没有"最好的模型"，只有最适合当前产品场景和成本约束的模型。选型时优先考虑 **任务匹配度、延迟要求、成本预算、数据合规** 四个维度。

#### 二、不同产品场景的模型选择策略

| 产品场景 | 推荐策略 | 说明 |
|---|---|---|
| 智能客服 / 对话 | GPT-4o-mini / DeepSeek 做日常对话，复杂问题路由到 GPT-4o | 分层调用，平衡成本与体验 |
| 内容生成 | Claude / GPT-4o | 长文质量高，需关注品牌调性一致性 |
| 代码辅助 | GPT-4o / Claude | 代码补全与解释能力突出 |
| 文档 / 知识问答 | RAG + Gemini 1.5 Pro / Claude | 利用长上下文优势减少分块损失 |
| 多模态理解 | GPT-4o / Gemini | 支持图片、音频输入 |
| 端侧 / 低延迟 | 量化开源小模型（Llama 3 8B 等） | 延迟 < 100ms 场景 |

#### 三、API 调用基础（PM 必知）

| 概念 | 说明 |
|---|---|
| Token | 模型处理文本的基本单位，约 1 个中文字 ≈ 1.5–2 Token |
| 计费方式 | 通常按输入 Token + 输出 Token 分别计费 |
| Rate Limit | API 的调用频率限制，超出会被限流（HTTP 429） |
| 流式输出 (Streaming) | 逐 Token 返回结果，提升用户感知速度 |
| Temperature | 控制输出随机性：0 = 确定性，1 = 更有创意 |
| Context Window | 模型一次能处理的最大 Token 数（如 128K） |

#### 四、Prompt 设计最佳实践

| 技巧 | 说明 | 示例场景 |
|---|---|---|
| System Prompt | 在对话开头定义角色和规则 | 客服 Bot 设定回复风格 |
| Few-shot | 提供 2–3 个示例引导输出格式 | 结构化数据提取 |
| Chain-of-Thought (CoT) | 要求模型分步推理 | 复杂问题拆解、数学计算 |
| ReAct | 推理 + 行动交替执行 | Agent 调用外部工具 |
| 输出格式约束 | 指定 JSON / Markdown 等输出格式 | API 返回结构化数据 |
| 分步校验 | 让模型先生成再自我检查 | 减少幻觉 |

#### 五、模型评测方法

| 步骤 | 关键动作 |
|---|---|
| 1. 构建评测集 | 收集 100–500 条真实用户 Query，标注预期答案 |
| 2. 定义指标 | 准确率、相关性评分（1–5 分）、延迟（P50/P99）、单次成本 |
| 3. 自动评测 | 用另一个 LLM（如 GPT-4o）作为评委自动打分 |
| 4. 人工评审 | 抽样 10–20% 进行人工复核，校准自动评分 |
| 5. 对比决策 | 制作 Scorecard，综合质量 / 延迟 / 成本做出选型决策 |

#### 六、安全与合规注意事项

| 维度 | 要点 |
|---|---|
| 内容审核 | 对模型输出做二次安全过滤（敏感词、有害内容、违规信息） |
| 数据隐私 | 确认 API 调用数据是否被用于训练；敏感数据考虑私有化部署 |
| 幻觉管理 | 结合 RAG 提供事实依据；在 UI 中标注信息来源和置信度 |
| 合规要求 | 了解《生成式人工智能服务管理暂行办法》等国内法规；出海产品需关注 EU AI Act |
| 日志与审计 | 记录所有 LLM 交互日志，便于问题排查和合规审计 |

### AI/ML 核心术语表

| 术语 | 英文 | 解释 |
|---|---|---|
| 机器学习 | Machine Learning (ML) | 让计算机从数据中学习规律，无需显式编程 |
| 深度学习 | Deep Learning (DL) | 基于多层神经网络的机器学习方法 |
| 监督学习 | Supervised Learning | 使用带标签的数据进行训练 |
| 无监督学习 | Unsupervised Learning | 从无标签数据中发现模式 |
| 强化学习 | Reinforcement Learning (RL) | 通过奖惩信号学习最优策略 |
| 自然语言处理 | Natural Language Processing (NLP) | 让计算机理解和生成人类语言 |
| 大语言模型 | Large Language Model (LLM) | 基于 Transformer 的大规模预训练语言模型 |
| 词向量 / 嵌入 | Embedding | 将文本映射为高维向量表示 |
| Transformer | Transformer | 基于自注意力机制的神经网络架构 |
| 微调 | Fine-tuning | 在预训练模型基础上用特定数据继续训练 |
| 检索增强生成 | Retrieval-Augmented Generation (RAG) | 先检索相关知识，再交给 LLM 生成回答 |
| 提示工程 | Prompt Engineering | 设计输入提示以引导模型产生期望输出 |
| 幻觉 | Hallucination | 模型生成看似合理但实际错误的内容 |
| Token | Token | 模型处理文本的最小单位 |
| 上下文窗口 | Context Window | 模型一次能处理的最大输入长度 |
| 推理 | Inference | 使用训练好的模型进行预测 |
| 过拟合 | Overfitting | 模型在训练数据上表现好但泛化能力差 |
| 损失函数 | Loss Function | 衡量模型预测与真实值之间差距的函数 |
| 特征工程 | Feature Engineering | 从原始数据中提取有用特征的过程 |
| A/B 测试 | A/B Testing | 通过对照实验比较两种方案的效果 |

---

> **祝你学习顺利，成为出色的 AI 产品经理！**
