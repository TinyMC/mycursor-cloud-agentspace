# Demo 2 准备指南：Dify 知识库 + RAG 原理 + MCP 集成

> **目标读者**：技术型产品经理，需要理解原理后动手搭建  
> **目标**：读完本文后，你能直接开始 Demo 2（竞品情报 RAG 知识库）的实操  
> **工具链**：Dify（知识库+Workflow）+ Cursor Pro+（MCP）+ Python

---

## 目录

- [第一部分：RAG 原理深度解析](#第一部分rag-原理深度解析)
- [第二部分：Dify 知识库搭建操作手册](#第二部分dify-知识库搭建操作手册)
- [第三部分：元数据标签体系设计](#第三部分元数据标签体系设计)
- [第四部分：MCP 集成方案（读取链接文件）](#第四部分mcp-集成方案读取链接文件)
- [第五部分：Demo 2 完整实施清单](#第五部分demo-2-完整实施清单)

---

# 第一部分：RAG 原理深度解析

## 1.1 RAG 是什么——给 PM 的一句话解释

> **RAG = 开卷考试**  
> 普通 LLM 是"闭卷考试"——靠训练时记住的知识回答。  
> RAG 是"开卷考试"——先去知识库里翻书找相关段落，再基于找到的内容回答。

**解决的核心问题**：
- LLM 知识有截止日期（不知道你昨天上传的竞品报告）
- LLM 会幻觉（编造不存在的竞品数据）
- 企业私有数据 LLM 从未见过（你的内部分析文档）

## 1.2 RAG 完整数据流（逐步拆解）

```
                            RAG 全链路
                            =========

    ┌───────────────── 离线阶段（建库）─────────────────┐
    │                                                   │
    │  ① 原始文档                                       │
    │  ┌──────┐ ┌──────┐ ┌──────┐                      │
    │  │PDF   │ │MD    │ │DOCX  │                      │
    │  │竞品报告│ │分析文档│ │会议纪要│                     │
    │  └──┬───┘ └──┬───┘ └──┬───┘                      │
    │     └────────┼────────┘                           │
    │              ▼                                    │
    │  ② 文档解析（Document Parsing）                    │
    │     提取纯文本，处理表格/图片/页眉页脚               │
    │              ▼                                    │
    │  ③ 文本分段（Chunking）                            │
    │     按规则切成小块（如每块 500 token）               │
    │     ┌──────┐┌──────┐┌──────┐┌──────┐             │
    │     │Chunk1││Chunk2││Chunk3││Chunk4│              │
    │     └──┬───┘└──┬───┘└──┬───┘└──┬───┘             │
    │        └───────┼───────┼───────┘                  │
    │                ▼                                  │
    │  ④ 向量化（Embedding）                             │
    │     每个 Chunk → 一个高维向量（如 1024 维浮点数组）   │
    │     [0.023, -0.156, 0.891, ...]                   │
    │                ▼                                  │
    │  ⑤ 存入向量数据库                                  │
    │     Chunk 文本 + 向量 + 元数据 → PostgreSQL/pgvector│
    │                                                   │
    └───────────────────────────────────────────────────┘

    ┌───────────────── 在线阶段（查询）─────────────────┐
    │                                                   │
    │  ⑥ 用户提问                                       │
    │     "Competitor A 的定价策略是什么？"               │
    │              ▼                                    │
    │  ⑦ 查询改写（Query Rewriting）                     │
    │     LLM 将口语化问题改写为检索友好的关键词           │
    │     → ["Competitor A pricing", "定价模式", "收费标准"]│
    │              ▼                                    │
    │  ⑧ 检索（Retrieval）                              │
    │     ┌─────────────┐  ┌─────────────┐             │
    │     │ 向量检索     │  │ 关键词检索   │             │
    │     │ (语义相似度) │  │ (BM25精确匹配)│            │
    │     └──────┬──────┘  └──────┬──────┘             │
    │            └────────┬───────┘                     │
    │                     ▼                             │
    │  ⑨ 融合 + 重排序（RRF + Rerank）                   │
    │     合并两路结果 → Cross-Encoder 精排 → Top-K      │
    │              ▼                                    │
    │  ⑩ 上下文构建（Context Assembly）                   │
    │     System Prompt + 检索结果 + 用户问题 → 拼成完整 Prompt│
    │              ▼                                    │
    │  ⑪ LLM 生成回答                                   │
    │     基于检索到的真实数据生成，附带引用来源            │
    │                                                   │
    └───────────────────────────────────────────────────┘
```

## 1.3 每个环节的 PM 决策点

这是 RAG 管线中**你作为 PM 必须做的决策**，不是工程师的活：

| 环节 | PM 需要决策什么 | 决策错了会怎样 |
|:---|:---|:---|
| **③ 分段** | Chunk 多大？重叠多少？ | 太大→检索不精准，太小→缺上下文 |
| **④ 向量化** | 用哪个 Embedding 模型？ | 模型不对→中文语义理解差，召回率低 |
| **⑦ 查询改写** | 要不要改写？改写策略？ | 不改写→用户口语化问题检索效果差 |
| **⑧ 检索** | 向量/关键词/混合？Top-K 设多少？ | 单向量→精确词漏召回；纯关键词→同义词漏召回 |
| **⑨ 重排序** | 要不要 Rerank？用哪个模型？ | 不重排→噪声结果混入上下文，影响生成质量 |
| **⑩ 上下文** | 塞多少结果给 LLM？ | 太多→token 超限 + 噪声；太少→信息不够 |

## 1.4 Embedding（向量化）原理——PM 必知

### 什么是 Embedding

把一段文字变成一组数字（向量），使得**语义相近的文字，数字也相近**。

```
"苹果手机很贵"  → [0.82, -0.13, 0.45, ...]  ─┐
                                               ├── 距离近（语义相关）
"iPhone价格高"   → [0.79, -0.11, 0.48, ...]  ─┘

"今天天气真好"  → [-0.34, 0.67, -0.21, ...] ← 距离远（语义无关）
```

### 中文 Embedding 模型选型（2026年）

| 模型 | 来源 | 中文效果 | 成本 | 推荐场景 |
|:---|:---|:---:|:---|:---|
| `bge-large-zh-v1.5` | 智源BAAI，开源 | ★★★★★ | 免费（自部署） | **Demo 2 首选**：中文效果最佳，Dify 原生支持 |
| `bge-m3` | 智源BAAI，开源 | ★★★★ | 免费 | 多语言混合文档 |
| `text-embedding-3-large` | OpenAI | ★★★★ | $0.13/MTok | 不差钱+需要英文混合 |
| `gte-large-zh` | 阿里，开源 | ★★★★ | 免费 | 阿里云生态用户 |
| 千帆 `ernie-embedding` | 百度 | ★★★★ | ¥0.35/MTok | 百度云生态 |

**Demo 2 推荐**：`bge-large-zh-v1.5`——免费、中文效果最佳、Dify 原生支持。

## 1.5 检索策略对比——混合检索为什么是最优解

```
场景1：用户搜"苹果的收入"
┌────────────────────────────────────────────────────┐
│ 向量检索结果：                                       │
│  ✅ "Apple 2024年营收达3800亿美元"（语义匹配"苹果"）  │
│  ✅ "iPhone 销售额同比增长5%"（语义关联"收入"）       │
│  ❌ "水果市场苹果价格上涨"（语义歧义！）              │
│                                                     │
│ 关键词检索结果：                                      │
│  ✅ 精确匹配含"苹果"和"收入"的段落                    │
│  ❌ 漏掉"Apple revenue"（不识别同义词）               │
│                                                     │
│ 混合检索 = 两者合并 → RRF 融合排序 → 最佳结果         │
└────────────────────────────────────────────────────┘
```

| 检索方式 | 擅长 | 不擅长 | 适合场景 |
|:---|:---|:---|:---|
| **纯向量** | 同义词、跨语言、语义推理 | 精确人名/型号/编号 | 开放式问题 |
| **纯关键词** | 精确匹配、专有名词 | 语义改写、同义表达 | 精确查找 |
| **混合检索** | 两者兼具 | 配置复杂度稍高 | **生产环境首选** |

### RRF（倒数排名融合）算法

```
两路检索各自返回 Top-10：
  向量检索：  Doc_A(第1) Doc_B(第3) Doc_C(第5) ...
  关键词检索：Doc_B(第1) Doc_D(第2) Doc_A(第4) ...

RRF Score = Σ  1/(k + rank_i)     其中 k=60（常数）

Doc_A: 1/(60+1) + 1/(60+4) = 0.0164 + 0.0156 = 0.032  ← 综合第1
Doc_B: 1/(60+3) + 1/(60+1) = 0.0159 + 0.0164 = 0.032  ← 综合第2
Doc_D: 0       + 1/(60+2) = 0.0161                      ← 综合第3
```

两路检索中都排名靠前的文档，融合后得分最高。

## 1.6 重排序（Rerank）——从"召回"到"精排"

```
     初始检索返回 20 个 Chunk（召回阶段，求全）
                    │
                    ▼
     ┌─────────────────────────────┐
     │  Rerank 模型（如 bge-reranker）│
     │                             │
     │  输入: Query + 每个 Chunk    │
     │  输出: 相关性分数 0~1        │
     │                             │
     │  精排规则：                   │
     │  • 不只看关键词重叠           │
     │  • 深度理解语义关联           │
     │  • 考虑上下文连贯性           │
     └──────────────┬──────────────┘
                    ▼
     精排后 Top-5（精排阶段，求准）→ 送入 LLM
```

**PM 理解要点**：检索阶段是"海选"（宁可多选不漏选），重排序是"决赛"（精确筛选最相关的）。

## 1.7 2026年 RAG 架构新趋势（了解即可）

| 趋势 | 说明 | 成熟度 |
|:---|:---|:---:|
| **父子分段** | 子Chunk精确匹配，返回父Chunk提供上下文 | ✅ Dify已支持 |
| **GraphRAG** | 知识图谱增强，捕捉实体关系 | 🔶 实验阶段 |
| **Agentic RAG** | Agent 自主决定是否检索、检索什么、迭代几次 | 🔶 部分落地 |
| **多模态RAG** | 同时检索文本+图片+表格 | 🔶 Dify支持中 |

---

# 第二部分：Dify 知识库搭建操作手册

## 2.1 操作全流程（截图级详细）

### Step 1：创建知识库

```
操作路径：Dify 控制台 → 左侧「知识库」→ 右上角「创建知识库」

┌────────────────────────────────────────────┐
│  创建知识库                                  │
│                                             │
│  知识库名称：竞品情报库                       │
│  描述：存储竞品分析报告、行业文章、产品文档     │
│                                             │
│  数据来源（选一种）：                         │
│  ● 上传本地文件  ← Demo 2 选这个             │
│  ○ 从 Notion 导入                           │
│  ○ 从网页导入（Firecrawl/Jina）             │
│  ○ 创建空知识库                              │
│                                             │
│  [下一步]                                   │
└────────────────────────────────────────────┘
```

**注意**：选择"上传本地文件"后，后续仍可继续添加本地文件。但如果选了 Notion/网页，就不能再加本地文件——这是个不可逆的选择。

### Step 2：上传文档

```
支持格式：PDF / MD / TXT / DOCX / CSV / HTML
单文件限制：15MB
批量上传：✅ 支持

Demo 2 建议的文件组织：
├── competitor_A_产品分析_2026Q1.md
├── competitor_A_定价策略.md
├── competitor_B_SWOT分析.md
├── industry_市场报告_2026.pdf
├── our_product_功能清单.md
└── our_product_路线图_2026H2.md
```

### Step 3：配置分段设置（关键步骤）

```
┌────────────────────────────────────────────────────┐
│  分段设置                                           │
│                                                     │
│  分段模式：                                         │
│  ● 通用模式  ← Demo 2 先用这个（简单可控）           │
│  ○ 父子分段  ← 高阶进阶用                           │
│  ○ Q&A 模式  ← 适合 FAQ 文档                        │
│                                                     │
│  ─── 通用模式参数 ───                                │
│                                                     │
│  分段标识符：  \n\n  ← 双换行（按段落切分）           │
│  最大分段长度：500 tokens  ← ⭐ 推荐值               │
│  分段重叠：    50 tokens   ← 分段长度的 10%          │
│                                                     │
│  ─── 文本清洗 ───                                   │
│                                                     │
│  ☑ 替换连续空格/换行符/制表符                        │
│  ☑ 删除 URL 和邮箱地址      ← 看情况，竞品文档URL可能有用│
│                                                     │
└────────────────────────────────────────────────────┘
```

**分段参数决策指南**：

| 文档类型 | 推荐 Chunk Size | 推荐 Overlap | 理由 |
|:---|:---:|:---:|:---|
| 竞品分析报告（信息密集） | 500 tokens | 50 tokens | 信息密集，小块精准 |
| 行业报告（段落完整） | 800 tokens | 80 tokens | 段落完整性更重要 |
| FAQ / QA 文档 | 用 Q&A 模式 | — | 自动拆成问答对 |
| 会议纪要（结构松散） | 300 tokens | 50 tokens | 碎片化信息需小块 |

### Step 4：选择索引方式

```
┌────────────────────────────────────────────────────┐
│  索引方式                                           │
│                                                     │
│  ● 高质量模式  ← 必选！Demo 2 需要语义检索           │
│  ○ 经济模式    ← 仅关键词，效果差，不推荐             │
│                                                     │
│  Embedding 模型：                                    │
│  ▼ bge-large-zh-v1.5  ← ⭐ 中文首选                │
│                                                     │
│  ⚠️ 选定后不可更换索引模式（只能删库重建）             │
│  ⚠️ 可以更换 Embedding 模型（会触发全量重新向量化）    │
│                                                     │
└────────────────────────────────────────────────────┘
```

### Step 5：配置检索设置

```
┌────────────────────────────────────────────────────┐
│  检索设置                                           │
│                                                     │
│  检索模式：                                         │
│  ○ 向量检索                                        │
│  ○ 全文检索                                        │
│  ● 混合检索  ← ⭐ Demo 2 必选                      │
│                                                     │
│  ─── 混合检索参数 ───                                │
│                                                     │
│  向量检索权重：  0.7  ← 语义为主                     │
│  全文检索权重：  0.3  ← 关键词辅助                    │
│                                                     │
│  Rerank 模型：                                      │
│  ☑ 启用   ← ⭐ 强烈建议开启                         │
│  ▼ bge-reranker-v2-m3  ← 中文效果好                │
│                                                     │
│  Top-K：  5   ← 返回前5个最相关分段                  │
│  Score 阈值：  0.5  ← 低于此分数丢弃                 │
│                                                     │
│  ⚠️ 如果总是"未找到相关内容"，把阈值降到 0.3 试试     │
│                                                     │
└────────────────────────────────────────────────────┘
```

### Step 6：等待处理 + 召回测试

```
处理完成后，立即做召回测试（非常重要！）：

操作路径：知识库 → 选择文档 → 右上角「召回测试」

测试问题示例：
┌─────────────────────────────────────────────┐
│  Q: "Competitor A 的定价策略是什么？"          │
│                                              │
│  期望召回：competitor_A_定价策略.md 的相关段落  │
│                                              │
│  实际结果：                                   │
│  ┌──────────────────────────────────────┐    │
│  │ #1 [Score: 0.87] competitor_A_定价... │    │
│  │ #2 [Score: 0.72] competitor_A_产品... │    │
│  │ #3 [Score: 0.65] industry_市场报告... │    │
│  └──────────────────────────────────────┘    │
│                                              │
│  ✅ Score > 0.7 的结果与预期一致 → 配置OK      │
│  ❌ 无相关结果或 Score 都很低 → 需调参         │
└─────────────────────────────────────────────┘
```

**召回测试 Checklist（至少测 10 个问题）**：

| # | 测试问题 | 期望命中文档 | 实际结果 | 通过？ |
|:---:|:---|:---|:---|:---:|
| 1 | Competitor A 定价策略 | competitor_A_定价策略.md | | |
| 2 | 行业市场规模多大 | industry_市场报告_2026.pdf | | |
| 3 | 我们产品的路线图 | our_product_路线图_2026H2.md | | |
| 4 | Competitor B 的优势和劣势 | competitor_B_SWOT分析.md | | |
| 5 | 定价多少钱（口语化） | competitor_A_定价策略.md | | |
| ... | ... | ... | | |

## 2.2 父子分段（高阶模式）

当你发现通用模式的召回质量不够好时，可以升级到父子分段：

```
问题：通用模式下，Chunk 包含一句"该功能定价 $99/月"
     → 被召回了，但缺少前后文（为什么定价 $99？对比竞品如何？）

解决：父子分段
     子Chunk："该功能定价 $99/月"  ← 精确匹配
     父Chunk：整个定价章节（2000字） ← 完整上下文送给 LLM
```

**配置方式**：

```
分段模式：● 父子分段

父分段设置：
  模式：段落模式（推荐）
  分段标识符：\n\n
  最大长度：2000 tokens

子分段设置：
  分段标识符：。（按句子切分）
  最大长度：200 tokens
```

**适用场景**：
- ✅ 长篇竞品分析报告（章节结构清晰）
- ✅ 技术文档（需要完整上下文）
- ❌ FAQ 文档（用 Q&A 模式更好）
- ❌ 短文档（本身就一两百字，无需分层）

## 2.3 Q&A 分段模式

Dify 可以自动将文档拆成问答对：

```
原始文档内容：
"Competitor A 成立于2015年，总部位于深圳。公司主要做跨境供应链服务，
 2024年营收达到50亿元。创始人张三曾在华为工作10年。"

Q&A 模式自动生成：
Q: Competitor A 是什么时候成立的？
A: Competitor A 成立于2015年，总部位于深圳。

Q: Competitor A 的营收是多少？
A: 2024年营收达到50亿元。

Q: Competitor A 的创始人背景是什么？
A: 创始人张三曾在华为工作10年。
```

**适用场景**：客户 FAQ、产品问答、常见问题汇总。

---

# 第三部分：元数据标签体系设计

## 3.1 什么是元数据

> **元数据 = 贴在文档上的标签**  
> 就像图书馆里每本书的分类标签（作者、出版年、类别）一样，  
> 元数据让你在检索时可以**先筛选再搜索**，大幅提升精准度。

```
没有元数据的检索：
  Q: "Competitor A 2026年的定价"
  → 在所有文档中搜索 → 可能召回 2024 年旧数据 ❌

有元数据的检索：
  Q: "Competitor A 2026年的定价"
  → 先筛选 source=competitor_A AND year=2026
  → 在筛选后的小范围内搜索 → 精准命中 ✅
```

## 3.2 Dify 元数据功能（v1.1.0+）

### 内置元数据（自动生成）

| 字段 | 说明 | 示例 |
|:---|:---|:---|
| `document_name` | 文件名 | competitor_A_定价策略.md |
| `uploader` | 上传者 | hanhui |
| `upload_date` | 上传时间 | 2026-05-07 |
| `last_update_date` | 最后更新时间 | 2026-05-07 |
| `source` | 来源（Notion/网页/本地） | local_file |

### 自定义元数据（需手动添加）

```
操作路径：知识库 → 选择文档 → 右侧「元数据」面板 → 「添加字段」

Demo 2 推荐的自定义元数据字段：
```

| 字段名 | 类型 | 用途 | 示例值 |
|:---|:---|:---|:---|
| `competitor` | String | 所属竞品公司 | "行云集团" / "纵腾" / "our_product" |
| `doc_type` | String | 文档类型 | "product_analysis" / "pricing" / "swot" / "market_report" |
| `market` | String | 涉及市场 | "russia" / "usa" / "brazil" / "global" |
| `confidence` | String | 数据可信度 | "high" / "medium" / "low" |
| `report_date` | Time | 报告日期 | 2026-05-01 |
| `data_year` | Number | 数据年份 | 2026 |

### 在 Workflow 中使用元数据过滤

```
Dify Workflow → 知识检索节点 → 启用「元数据过滤」

过滤规则示例：

规则1：只搜 Competitor A 的文档
  competitor = "行云集团"

规则2：只搜 2025-2026 年数据
  data_year >= 2025

规则3：组合过滤
  competitor = "纵腾" AND doc_type = "pricing" AND data_year = 2026

效果：从100篇文档 → 筛选到3篇 → 在3篇中做向量检索
     精准度从 70% 提升到 95%+
```

## 3.3 文档元数据模板（直接复用）

在你的竞品文档顶部加入 YAML Front Matter，上传前统一格式：

```markdown
---
competitor: "行云集团"
doc_type: "product_analysis"
market: "global"
confidence: "high"
report_date: "2026-05-01"
data_year: 2026
author: "hanhui"
---

# 行云集团产品分析（2026Q1）

## 1. 公司基本面
...
```

**注意**：Dify 目前不会自动解析 YAML Front Matter 为元数据字段——你需要在上传后手动在 Dify UI 中添加这些元数据。但在文档中保留这些标签有两个好处：
1. 你自己管理文档时一目了然
2. Chunk 中会包含这些标签文本，向量检索时能提供额外语义信号

## 3.4 元数据标签最佳实践

| 原则 | 说明 |
|:---|:---|
| **命名统一** | 用英文小写+下划线（`doc_type` 不是 `文档类型`），避免中英混用 |
| **值域收敛** | `competitor` 字段的值事先定好候选列表，不要一会儿写"行云"一会儿写"行云集团" |
| **必填 vs 选填** | `competitor` 和 `doc_type` 必填；`confidence` 可选填 |
| **时间字段** | 用 `data_year`（Number）而非 `report_date`（Time）做过滤更方便——数字比较比日期比较简单 |
| **批量管理** | 上传完所有文档后，在 Dify 的文档列表页统一打标签，比一个个加快 |

---

# 第四部分：MCP 集成方案（读取链接文件）

## 4.1 为什么要在 Demo 2 引入 MCP

```
没有 MCP 的工作流：
  你手动下载竞品网页 → 粘贴到 MD 文件 → 上传到 Dify → 查询
  问题：数据不新鲜、手动操作多、无法自动化

有 MCP 的工作流：
  Cursor Agent 自动读取竞品网页/文件链接 → 转为文本 → 推送到 Dify → 查询
  优势：数据实时、全自动、可定期刷新
```

## 4.2 三个 MCP Server 方案

### 方案A：读取本地文件（Filesystem MCP）

**用途**：让 Cursor Agent 直接读取本地竞品文档文件

`.cursor/mcp.json`：
```json
{
  "mcpServers": {
    "competitor-docs": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/path/to/your/project/data/knowledge"
      ]
    }
  }
}
```

**Cursor 中使用**：
```
你：帮我读取 data/knowledge/competitor_A/ 下的所有文件，总结 Competitor A 的核心能力

Agent 自动：
  1. 通过 MCP 调用 list_directory 列出文件
  2. 逐个 read_file 读取内容
  3. 汇总分析后回复你
```

### 方案B：抓取网页链接（Fetch MCP）

**用途**：让 Cursor Agent 直接读取竞品官网、新闻链接等在线内容

`.cursor/mcp.json`：
```json
{
  "mcpServers": {
    "web-fetch": {
      "command": "npx",
      "args": ["-y", "@tokenizin/mcp-npx-fetch"]
    }
  }
}
```

**提供的工具**：

| 工具名 | 功能 | 适用场景 |
|:---|:---|:---|
| `fetch_markdown` | 网页 → Markdown | 竞品官网、博客文章 |
| `fetch_html` | 网页 → 原始 HTML | 需要保留结构 |
| `fetch_json` | API → JSON | 竞品公开 API |
| `fetch_txt` | 网页 → 纯文本 | 通用文本提取 |

**Cursor 中使用**：
```
你：帮我抓取 https://www.competitor-a.com/pricing 的内容，提取定价信息

Agent 自动：
  1. 调用 fetch_markdown("https://www.competitor-a.com/pricing")
  2. 获取 Markdown 格式的网页内容
  3. 提取定价信息并总结
```

**进阶用法——批量抓取 + 自动入库**：

```python
# scripts/fetch_and_index.py
# 批量抓取竞品链接，转为MD文件，准备上传到 Dify

import subprocess
import json

COMPETITOR_URLS = {
    "competitor_A": [
        "https://www.competitor-a.com/pricing",
        "https://www.competitor-a.com/about",
        "https://www.competitor-a.com/blog/2026-strategy",
    ],
    "competitor_B": [
        "https://www.competitor-b.com/products",
    ],
}

for competitor, urls in COMPETITOR_URLS.items():
    for url in urls:
        slug = url.split("/")[-1] or "index"
        output_path = f"data/knowledge/{competitor}/{slug}.md"

        # 你可以在这里调用 MCP fetch_markdown 或直接用 requests
        # 这个脚本在 Cursor 外独立运行时用 requests：
        import requests
        resp = requests.get(url, timeout=30)

        with open(output_path, "w") as f:
            # 写入元数据头
            f.write(f"---\n")
            f.write(f"competitor: \"{competitor}\"\n")
            f.write(f"source_url: \"{url}\"\n")
            f.write(f"doc_type: \"web_scrape\"\n")
            f.write(f"data_year: 2026\n")
            f.write(f"---\n\n")
            f.write(resp.text)

        print(f"✅ Saved: {output_path}")
```

### 方案C：直接查询 Dify 知识库（Dify Knowledge MCP）

**用途**：让 Cursor Agent 直接查询 Dify 中已建好的知识库，不用手动打开 Dify UI

`.cursor/mcp.json`：
```json
{
  "mcpServers": {
    "dify-knowledge": {
      "command": "npx",
      "args": ["-y", "@funtuantw/dify-external-knowledge-mcp"],
      "env": {
        "DIFY_KNOWLEDGE_ENDPOINT": "https://api.dify.ai/v1",
        "DIFY_API_KEY": "your-dify-dataset-api-key",
        "DIFY_KNOWLEDGE_ID": "your-knowledge-base-id"
      }
    }
  }
}
```

**Cursor 中使用**：
```
你：从竞品知识库中查一下行云集团的融资历程

Agent 自动：
  1. 调用 query_dify_knowledge("行云集团 融资历程")
  2. Dify 返回相关 Chunk（带 Score 和来源）
  3. Agent 基于检索结果回答你
```

## 4.3 三个方案的组合策略

```
┌──────────────────────────────────────────────────────────┐
│                   Demo 2 MCP 集成全景                     │
│                                                          │
│  方案A: Filesystem MCP        方案B: Fetch MCP            │
│  读取本地文档文件              抓取网页链接                 │
│         │                          │                      │
│         └─────────┬────────────────┘                      │
│                   ▼                                       │
│          data/knowledge/                                  │
│          本地文档仓库                                      │
│                   │                                       │
│                   ▼                                       │
│          上传到 Dify Knowledge                            │
│          （手动或通过 Dify API）                           │
│                   │                                       │
│                   ▼                                       │
│  方案C: Dify Knowledge MCP                               │
│  在 Cursor 中直接查询 Dify 知识库                         │
│                                                          │
│  完整链路：                                               │
│  网页/文件 → MCP读取 → 本地存储 → Dify入库 → MCP查询     │
└──────────────────────────────────────────────────────────┘
```

**推荐执行顺序**：
1. 先配置方案A（Filesystem），确保 Agent 能读本地文件
2. 再配置方案B（Fetch），实现网页抓取
3. 最后配置方案C（Dify Knowledge），打通 Cursor ↔ Dify 闭环

---

# 第五部分：Demo 2 完整实施清单

## 5.1 前置准备 Checklist

| # | 事项 | 状态 | 说明 |
|:---:|:---|:---:|:---|
| 1 | Dify 账号就绪 | ☐ | dify.ai 云端 或 本地 docker 部署 |
| 2 | Dify 中配置 Embedding 模型 | ☐ | 推荐 `bge-large-zh-v1.5` |
| 3 | Dify 中配置 Rerank 模型 | ☐ | 推荐 `bge-reranker-v2-m3` |
| 4 | Dify 中配置 LLM | ☐ | 用于 Workflow 中的 LLM 节点 |
| 5 | 准备 10+ 篇竞品文档 | ☐ | MD/PDF 格式，按竞品分文件夹 |
| 6 | 每篇文档加元数据头 | ☐ | YAML Front Matter 格式 |
| 7 | Cursor MCP 配置就绪 | ☐ | `.cursor/mcp.json` 三个 Server |
| 8 | Python 环境 | ☐ | `pip install requests dify-python-sdk` |

## 5.2 执行步骤（按天拆分）

### Day 1：知识库搭建

```
上午：
  1. 整理 10+ 篇竞品文档（可直接用领航数贸竞品报告拆分）
  2. 为每篇文档添加元数据头（competitor / doc_type / data_year）
  3. 在 Dify 创建知识库「竞品情报库」
  4. 上传文档，配置分段参数：
     - 模式：通用
     - Chunk Size：500
     - Overlap：50
     - 索引：高质量（bge-large-zh-v1.5）

下午：
  5. 配置检索设置：
     - 混合检索（向量 0.7 + 全文 0.3）
     - 开启 Rerank
     - Top-K：5，Score 阈值：0.5
  6. 召回测试：准备 20 个测试问题，逐一验证
  7. 调参：根据测试结果调整 Chunk Size / Top-K / 阈值
```

### Day 2：Workflow 搭建

```
上午：
  8. 在 Dify 创建 Workflow「竞品情报查询」
  9. 搭建节点：
     [开始] → [查询改写LLM] → [知识检索] → [条件分支] → [生成LLM] → [结束]
  10. 配置条件分支：quick_answer / comparison / swot

下午：
  11. 为知识检索节点配置元数据过滤
  12. 发布 Workflow，获取 API Key
  13. 编写 Python 调用脚本 scripts/query_competitor.py
  14. 测试 API 调用：3 种 report_type 各测 3 个问题
```

### Day 3：MCP 集成 + 数据库

```
上午：
  15. 配置 .cursor/mcp.json（方案A + B + C）
  16. 在 Cursor 中测试：
      - Filesystem MCP 读本地文件
      - Fetch MCP 抓网页
      - Dify Knowledge MCP 查知识库

下午：
  17. 编写查询日志数据库（SQLite 或 PostgreSQL）
  18. 在 Cursor 中编写 Rules 规则（.cursor/rules/competitor-intel.mdc）
  19. 端到端验收测试：在 Cursor Composer 中用自然语言查竞品信息
  20. 复盘 + 输出 Demo 2 复盘报告
```

## 5.3 验收标准

| # | 验收项 | 达标标准 |
|:---:|:---|:---|
| 1 | 知识库文档数 | ≥ 10 篇 |
| 2 | 召回测试准确率 | ≥ 80%（20题中16题命中正确文档） |
| 3 | Workflow API | 可独立通过 API 调用，3种输出模式都正常 |
| 4 | 元数据过滤 | 按 competitor/doc_type/data_year 过滤有效 |
| 5 | MCP Filesystem | Cursor Agent 能读取本地文档 |
| 6 | MCP Fetch | Cursor Agent 能抓取网页内容 |
| 7 | MCP Dify Knowledge | Cursor Agent 能直接查询 Dify 知识库 |
| 8 | 查询日志 | 写入数据库，可统计高频问题 |
| 9 | Cursor Rules | 在 Composer 中用自然语言触发竞品查询 |

## 5.4 从领航数贸报告快速生成知识库文档

你已有的 `领航数贸竞品深度分析报告.md` 可以直接拆分为多篇知识库文档：

```
原始文件：领航数贸竞品深度分析报告.md（587行）
                    │
                    ▼ 拆分为
┌───────────────────────────────────┐
│ competitor_行云集团_基本面.md       │ ← Step 2 第1家的内容
│ competitor_行云集团_SWOT.md        │
│ competitor_星商_基本面.md          │
│ competitor_星商_SWOT.md           │
│ competitor_纵腾集团_基本面.md      │
│ competitor_纵腾集团_SWOT.md       │
│ competitor_递四方_基本面.md        │
│ competitor_飞书深诺_基本面.md      │
│ competitor_乐歌股份_基本面.md      │
│ industry_竞争格局分析.md          │ ← Step 3 的内容
│ strategy_战略建议.md              │
└───────────────────────────────────┘

每篇加元数据头后上传到 Dify Knowledge
```

---

## 附录：关键概念速查卡

| 概念 | 一句话解释 | Demo 2 中在哪用到 |
|:---|:---|:---|
| **RAG** | 先搜后答的AI模式 | 整个 Demo 2 的核心架构 |
| **Chunk** | 文档切成的小块 | Dify 知识库分段设置 |
| **Embedding** | 文字变数字（向量） | Dify 索引时自动执行 |
| **向量检索** | 用语义相似度搜索 | 检索设置 - 混合检索 |
| **关键词检索** | 精确匹配关键词 | 检索设置 - 混合检索 |
| **混合检索** | 向量+关键词双路合并 | Demo 2 的检索策略 |
| **RRF** | 融合两路排名的算法 | 混合检索内部自动执行 |
| **Rerank** | 二次精排检索结果 | 检索设置 - Rerank 模型 |
| **父子分段** | 小块匹配大块回传 | 进阶优化（Day 3 可选） |
| **元数据** | 文档的标签（公司/类型/年份） | 过滤检索范围 |
| **Top-K** | 返回前K个结果 | 检索设置参数 |
| **Score 阈值** | 低于此分数的结果丢弃 | 检索设置参数 |
| **MCP** | 模型连接外部工具的协议 | 三个 MCP Server |
| **Dify Workflow** | 可视化编排AI工作流 | Demo 2 的查询管线 |
| **Dify Knowledge** | Dify 内置知识库 | 存储竞品文档 |

---

*准备完成后，直接按 Day 1-3 的步骤执行即可。遇到问题随时 ping 我。*
