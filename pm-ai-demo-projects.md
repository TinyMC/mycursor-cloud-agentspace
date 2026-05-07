# 产品经理 AI 模型练手 Demo（三个项目）

> **适用人设**：技术型产品经理 | Cursor Pro+ | Dify | LiteLLM  
> **目标**：结构性掌握 Constraints / Skills / RAG / Agent / Workflow / MCP / DB 等核心概念  
> **工具链**：Cursor Pro+（主 IDE）+ Dify（工作流编排）+ SQLite/PostgreSQL（数据库）+ Python（胶水语言）

---

## 项目总览

| # | 项目名称 | 核心练点 | 技术栈 | 难度 | 预估耗时 |
|:---:|:---|:---|:---|:---:|:---:|
| 1 | **PRD 智能审查助手** | Constraints + Skills + Prompt 工程 | Cursor Rules/Skills + Dify Workflow + SQLite | ⭐⭐ | 2-3天 |
| 2 | **竞品情报 RAG 知识库** | RAG + 向量检索 + 知识库管理 | Dify Knowledge + Embedding + PostgreSQL + pgvector | ⭐⭐⭐ | 3-5天 |
| 3 | **需求排期 Multi-Agent 系统** | Agent 编排 + MCP + 工具调用 + 数据库 | Dify Agent + MCP Server + SQLite + LiteLLM 路由 | ⭐⭐⭐⭐ | 5-7天 |

---

## Demo 1：PRD 智能审查助手

> **一句话**：把你的 PRD 审查经验变成 AI 可执行的约束规则，丢进去一篇 PRD，自动输出审查报告。

### 1.1 你会学到什么

| 概念 | 对应练点 |
|:---|:---|
| **Constraints（约束）** | 用 System Prompt + Rules 约束模型输出格式与评审标准 |
| **Skills（技能）** | 编写 Cursor SKILL.md，让 Agent 按 SOP 执行审查流程 |
| **Prompt 工程** | 多步骤 Chain-of-Thought，结构化评分 |
| **Dify Workflow** | 将审查流程编排为可 API 调用的工作流 |
| **数据库** | 审查结果持久化到 SQLite |

### 1.2 架构图

```
┌──────────────┐     ┌──────────────────┐     ┌───────────────┐
│  用户输入 PRD  │────▶│  Dify Workflow    │────▶│  审查报告输出  │
│  (Markdown)   │     │                  │     │  (结构化JSON)  │
└──────────────┘     │  ① 格式检查节点   │     └───────┬───────┘
                     │  ② 完整性评估节点  │             │
                     │  ③ 逻辑一致性节点  │             ▼
                     │  ④ 评分汇总节点   │     ┌───────────────┐
                     └──────────────────┘     │  SQLite 存储   │
                                              │  审查历史记录   │
                                              └───────────────┘
```

### 1.3 实施步骤

#### Step 1：定义审查约束（Constraints）

在 Cursor 项目根目录创建 `.cursor/rules/prd-review.mdc`：

```markdown
---
description: PRD审查规则 - 对PRD文档进行结构化评审
globs: ["*.prd.md", "docs/prd/**"]
---

## PRD 审查约束

你是一个资深产品经理审查助手。审查 PRD 时必须遵循以下约束：

### 输出格式约束
- 必须输出 JSON 格式，包含 score（0-100）、issues[]、suggestions[]
- 每个 issue 必须标注 severity: critical / major / minor
- 每个 issue 必须引用原文位置

### 评审维度约束（5个维度，各20分）
1. **需求清晰度**：用户故事是否完整、验收标准是否可测试
2. **技术可行性**：是否考虑技术约束、性能要求是否量化
3. **数据闭环**：是否定义核心指标、埋点方案是否完整
4. **边界定义**：异常流程是否覆盖、MVP 范围是否明确
5. **协作信息**：排期/依赖/风险是否列出

### 行为约束
- 不编造原文中不存在的问题
- 给出具体修改建议而非泛泛而谈
- 对评分为 critical 的问题必须给出修改示例
```

#### Step 2：编写 Cursor Skill

创建 `.cursor/skills/prd-review-skill/SKILL.md`：

```markdown
# PRD 审查技能

## 触发条件
当用户要求审查 PRD 文档时使用此技能。

## 执行步骤

### 1. 读取 PRD 文档
- 使用 Read 工具读取目标 PRD 文件
- 确认文件格式为 Markdown

### 2. 调用 Dify 审查工作流
- 使用 Shell 工具执行:
  ```bash
  python scripts/call_dify_review.py --input <prd_file_path>
  ```
- 该脚本调用 Dify Workflow API，传入 PRD 全文

### 3. 解析审查结果
- 读取返回的 JSON 审查报告
- 如果总分 < 60，标记为"需大幅修改"
- 如果总分 60-80，标记为"需优化"
- 如果总分 > 80，标记为"基本合格"

### 4. 写入数据库
- 执行 `python scripts/save_review.py` 保存审查记录

### 5. 输出报告
- 以 Markdown 表格形式展示各维度得分
- 列出所有 critical 和 major 问题
- 给出 Top 3 改进建议
```

#### Step 3：搭建 Dify Workflow

在 Dify 中创建工作流，节点设计：

```
[开始] → [变量: prd_content]
   ↓
[LLM节点1: 格式与完整性检查]
   Prompt: "检查以下PRD的格式规范性和内容完整性...输出JSON..."
   ↓
[LLM节点2: 逻辑一致性分析]
   Prompt: "基于上一步结果，深入分析需求的逻辑一致性..."
   ↓
[LLM节点3: 评分汇总]
   Prompt: "综合以上分析，按5个维度各打分，输出最终审查报告..."
   ↓
[结束: 返回 JSON 报告]
```

发布后获取 API Key，记录 Workflow endpoint。

#### Step 4：Python 调用脚本

创建 `scripts/call_dify_review.py`：

```python
"""调用 Dify PRD 审查工作流"""
import requests
import json
import sys
import sqlite3
from datetime import datetime

DIFY_API_BASE = "https://api.dify.ai/v1"  # 或你的私有部署地址
DIFY_API_KEY = "app-your-workflow-api-key"
DB_PATH = "data/prd_reviews.db"

def call_dify_workflow(prd_content: str) -> dict:
    """调用 Dify Workflow API"""
    resp = requests.post(
        f"{DIFY_API_BASE}/workflows/run",
        headers={
            "Authorization": f"Bearer {DIFY_API_KEY}",
            "Content-Type": "application/json",
        },
        json={
            "inputs": {"prd_content": prd_content},
            "response_mode": "blocking",  # 短文档用blocking，长文档用async
            "user": "hanhui-pm",
        },
    )
    resp.raise_for_status()
    return resp.json()

def save_to_db(filename: str, result: dict):
    """审查结果持久化到 SQLite"""
    conn = sqlite3.connect(DB_PATH)
    conn.execute("""
        CREATE TABLE IF NOT EXISTS reviews (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            filename TEXT,
            score INTEGER,
            result_json TEXT,
            reviewed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    conn.execute(
        "INSERT INTO reviews (filename, score, result_json) VALUES (?, ?, ?)",
        (filename, result.get("score", 0), json.dumps(result, ensure_ascii=False)),
    )
    conn.commit()
    conn.close()

if __name__ == "__main__":
    prd_file = sys.argv[1]
    with open(prd_file, "r") as f:
        content = f.read()

    print(f"📄 正在审查: {prd_file}")
    result = call_dify_workflow(content)

    output = result.get("data", {}).get("outputs", {})
    save_to_db(prd_file, output)

    print(f"✅ 审查完成，得分: {output.get('score', 'N/A')}")
    print(json.dumps(output, indent=2, ensure_ascii=False))
```

#### Step 5：配置 MCP 连接 SQLite（可选进阶）

在 `.cursor/mcp.json` 中添加：

```json
{
  "mcpServers": {
    "prd-review-db": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-sqlite", "data/prd_reviews.db"]
    }
  }
}
```

配置后，在 Cursor 中可以直接用自然语言查询审查历史：
- "查看最近10次审查中得分最低的PRD"
- "哪些PRD的数据闭环维度低于60分"

### 1.4 验收标准

- [ ] 输入一篇 PRD，能输出结构化评审报告（JSON）
- [ ] Dify Workflow 可通过 API 独立调用
- [ ] 审查结果写入 SQLite 并可查询历史
- [ ] Cursor 中通过 Skill 一键触发完整流程

---

## Demo 2：竞品情报 RAG 知识库

> **一句话**：把竞品分析报告、行业文章、产品文档灌入 RAG 知识库，用自然语言查询竞品情报并生成分析报告。

### 2.1 你会学到什么

| 概念 | 对应练点 |
|:---|:---|
| **RAG（检索增强生成）** | 文档切片、Embedding、向量检索、重排序的完整管线 |
| **知识库管理** | Dify Knowledge 的文档上传、索引策略、召回测试 |
| **混合检索** | 向量检索 + 关键词检索 + RRF 融合 |
| **Dify Workflow** | RAG 检索 → 摘要生成 → 报告输出的工作流 |
| **数据库** | PostgreSQL + pgvector 存储向量（Dify 内置或独立部署） |

### 2.2 架构图

```
┌─────────────────────────────────────────────────┐
│                  数据摄入层                       │
│  竞品报告(PDF) + 行业文章(URL) + 产品文档(MD)     │
└───────────────────────┬─────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────┐
│              Dify Knowledge 知识库                │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │文档切片   │  │Embedding │  │ PostgreSQL   │   │
│  │(512 token)│  │(bge-large│  │ + pgvector   │   │
│  │          │  │ -zh)     │  │ 向量存储      │   │
│  └──────────┘  └──────────┘  └──────────────┘   │
└───────────────────────┬─────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────┐
│              Dify Workflow（API化）               │
│                                                  │
│  [用户提问] → [查询改写] → [混合检索] → [重排序]  │
│       → [上下文构建] → [LLM生成] → [结构化输出]   │
└───────────────────────┬─────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────┐
│              输出层                               │
│  ① 直接回答（含引用来源）                         │
│  ② 竞品对比表格                                  │
│  ③ SWOT 分析报告                                 │
└─────────────────────────────────────────────────┘
```

### 2.3 实施步骤

#### Step 1：准备知识库数据

创建 `data/knowledge/` 目录，按竞品分类存放：

```
data/knowledge/
├── competitor_A/
│   ├── product_analysis_2026Q1.md
│   ├── pricing_page_snapshot.md
│   └── user_reviews_summary.md
├── competitor_B/
│   └── ...
├── industry/
│   ├── market_report_2026.pdf
│   └── trend_analysis.md
└── our_product/
    ├── feature_list.md
    └── roadmap_2026.md
```

**格式建议**：每篇文档顶部加元数据标签：

```markdown
---
source: competitor_A
type: product_analysis
date: 2026-05
confidence: high
---

# Competitor A 产品分析
...
```

#### Step 2：在 Dify 中创建知识库

1. **创建 Knowledge**：Dify → Knowledge → Create Knowledge
2. **上传文档**：支持 PDF / MD / TXT / DOCX
3. **索引设置**（关键参数）：

| 参数 | 推荐值 | 说明 |
|:---|:---|:---|
| Chunk Size | 512 tokens | 竞品文档信息密度高，小切片更精准 |
| Chunk Overlap | 50 tokens | 避免上下文断裂 |
| Embedding Model | `bge-large-zh-v1.5` | 中文效果好且免费 |
| 检索模式 | 混合检索（Hybrid） | 向量 + 关键词双路召回 |
| Top-K | 5 | 返回前5个最相关片段 |
| Score 阈值 | 0.6 | 低于此分数的结果丢弃 |

4. **召回测试**：上传后立即用"Competitor A 的定价策略是什么"测试召回质量

#### Step 3：搭建 Dify RAG 工作流

```
[开始: 用户问题 + report_type]
   ↓
[LLM节点: 查询改写]
   Prompt: "将用户的口语化问题改写为适合检索的关键词查询。
            原始问题: {{query}}
            输出3个检索query，JSON数组格式。"
   ↓
[知识检索节点] ← 关联你的 Knowledge
   检索模式: Hybrid
   Top-K: 5
   ↓
[条件分支: report_type]
   ├── "quick_answer" → [LLM: 直接回答，引用来源]
   ├── "comparison"   → [LLM: 生成竞品对比表格]
   └── "swot"         → [LLM: 生成SWOT分析]
   ↓
[结束: 返回结果 + 引用来源列表]
```

#### Step 4：Python 客户端 + 数据库记录

```python
"""竞品情报 RAG 查询客户端"""
import requests
import json
import psycopg2  # 或用 sqlite3 做轻量版
from datetime import datetime

DIFY_API_BASE = "https://api.dify.ai/v1"
DIFY_API_KEY = "app-your-rag-workflow-key"

def query_competitor_intel(question: str, report_type: str = "quick_answer") -> dict:
    """查询竞品情报"""
    resp = requests.post(
        f"{DIFY_API_BASE}/workflows/run",
        headers={
            "Authorization": f"Bearer {DIFY_API_KEY}",
            "Content-Type": "application/json",
        },
        json={
            "inputs": {
                "query": question,
                "report_type": report_type,  # quick_answer / comparison / swot
            },
            "response_mode": "blocking",
            "user": "hanhui-pm",
        },
    )
    return resp.json()

def query_with_logging(question: str, report_type: str = "quick_answer"):
    """查询并记录到数据库（追踪哪些问题常被问、检索质量如何）"""
    result = query_competitor_intel(question, report_type)
    output = result.get("data", {}).get("outputs", {})

    # 写入 PostgreSQL（或 SQLite）
    conn = psycopg2.connect("postgresql://localhost:5432/competitor_intel")
    conn.autocommit = True
    cur = conn.cursor()
    cur.execute("""
        CREATE TABLE IF NOT EXISTS query_logs (
            id SERIAL PRIMARY KEY,
            question TEXT,
            report_type VARCHAR(20),
            answer TEXT,
            sources TEXT,
            created_at TIMESTAMP DEFAULT NOW()
        )
    """)
    cur.execute(
        "INSERT INTO query_logs (question, report_type, answer, sources) VALUES (%s,%s,%s,%s)",
        (question, report_type, output.get("answer", ""), json.dumps(output.get("sources", []))),
    )
    conn.close()
    return output

# 使用示例
if __name__ == "__main__":
    # 快速问答
    print(query_with_logging("Competitor A 最近有什么新功能上线？"))

    # 对比表格
    print(query_with_logging("对比我们和 Competitor A 在协作功能上的差异", "comparison"))

    # SWOT 分析
    print(query_with_logging("帮我做一个 Competitor B 的 SWOT 分析", "swot"))
```

#### Step 5：Cursor 中集成查询（MCP + Rules）

在 `.cursor/rules/competitor-intel.mdc` 中定义：

```markdown
---
description: 竞品情报查询 - 当需要查询竞品信息时自动调用RAG知识库
globs: ["docs/competitor/**", "docs/prd/**"]
---

当用户询问竞品相关问题时：
1. 调用 `python scripts/query_competitor.py "<问题>"` 获取知识库检索结果
2. 基于检索结果回答，必须标注信息来源
3. 如果检索结果置信度低（无高相关结果），明确告知用户"知识库中暂无此信息"
4. 不得编造竞品数据
```

### 2.4 验收标准

- [ ] 至少灌入 10 篇竞品相关文档到 Dify Knowledge
- [ ] 混合检索召回准确率 ≥ 80%（手动测试 20 个问题）
- [ ] Workflow 支持 3 种输出模式（快速问答/对比表格/SWOT）
- [ ] 查询日志写入数据库，可统计高频问题
- [ ] 在 Cursor 中通过自然语言触发竞品查询

---

## Demo 3：需求排期 Multi-Agent 系统

> **一句话**：多个 AI Agent 协作，自动拆解需求、评估工时、检测冲突、生成排期表——模拟真实的产品-研发协作场景。

### 3.1 你会学到什么

| 概念 | 对应练点 |
|:---|:---|
| **Agent 编排** | 多 Agent 角色分工、消息传递、结果汇总 |
| **MCP Server** | 自定义 MCP 工具，让 Agent 读写数据库 |
| **工具调用（Function Calling）** | Agent 调用外部工具（日历API、JIRA API模拟） |
| **LiteLLM 路由** | 不同 Agent 使用不同模型（成本优化） |
| **Dify Agent** | Dify 的 Agent 模式 + 工具绑定 |
| **数据库** | SQLite 存储需求池、排期表、冲突日志 |

### 3.2 架构图

```
                    ┌──────────────────┐
                    │   用户输入需求    │
                    │ "开发用户画像功能  │
                    │  Q3上线，优先级P1" │
                    └────────┬─────────┘
                             ▼
                    ┌──────────────────┐
              ┌─────│  协调Agent(主控)  │─────┐
              │     │  Claude Sonnet   │     │
              │     └──────────────────┘     │
              ▼              ▼               ▼
   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
   │ 需求拆解Agent │ │ 工时评估Agent │ │ 冲突检测Agent │
   │  Claude Haiku │ │  Claude Haiku │ │ Claude Sonnet│
   │              │ │              │ │              │
   │ 拆user story │ │ 评估每个story│ │ 查排期表     │
   │ 定义验收标准  │ │ 的人天工时   │ │ 检测资源冲突  │
   └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
          │                │                │
          ▼                ▼                ▼
   ┌─────────────────────────────────────────────┐
   │              SQLite 数据库                    │
   │  requirements | schedules | conflicts        │
   └─────────────────────────────────────────────┘
                             ▼
                    ┌──────────────────┐
                    │  汇总Agent       │
                    │  生成排期甘特图    │
                    │  + 风险报告       │
                    └──────────────────┘
```

### 3.3 实施步骤

#### Step 1：初始化数据库

创建 `scripts/init_db.py`：

```python
"""初始化需求排期数据库"""
import sqlite3

DB_PATH = "data/sprint_planner.db"

def init():
    conn = sqlite3.connect(DB_PATH)
    conn.executescript("""
        CREATE TABLE IF NOT EXISTS requirements (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL,
            description TEXT,
            priority TEXT CHECK(priority IN ('P0','P1','P2','P3')),
            status TEXT DEFAULT 'new',
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        );

        CREATE TABLE IF NOT EXISTS user_stories (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            req_id INTEGER REFERENCES requirements(id),
            story TEXT NOT NULL,
            acceptance_criteria TEXT,
            estimated_days REAL,
            assignee TEXT,
            sprint TEXT
        );

        CREATE TABLE IF NOT EXISTS schedules (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            story_id INTEGER REFERENCES user_stories(id),
            start_date DATE,
            end_date DATE,
            assignee TEXT,
            status TEXT DEFAULT 'planned'
        );

        CREATE TABLE IF NOT EXISTS conflicts (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            story_id_a INTEGER,
            story_id_b INTEGER,
            conflict_type TEXT,
            description TEXT,
            detected_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        );

        -- 预置一些测试数据：已排期的需求
        INSERT OR IGNORE INTO requirements (id, title, priority, status)
        VALUES
            (1, '用户登录优化', 'P1', 'in_progress'),
            (2, '支付流程重构', 'P0', 'in_progress');

        INSERT OR IGNORE INTO user_stories (id, req_id, story, estimated_days, assignee, sprint)
        VALUES
            (1, 1, '支持手机号一键登录', 3, '张三', '2026-S1'),
            (2, 1, '增加微信登录', 5, '张三', '2026-S1'),
            (3, 2, '接入新支付渠道', 8, '李四', '2026-S1');

        INSERT OR IGNORE INTO schedules (story_id, start_date, end_date, assignee)
        VALUES
            (1, '2026-05-12', '2026-05-14', '张三'),
            (2, '2026-05-15', '2026-05-21', '张三'),
            (3, '2026-05-12', '2026-05-21', '李四');
    """)
    conn.commit()
    conn.close()
    print("✅ 数据库初始化完成")

if __name__ == "__main__":
    init()
```

#### Step 2：搭建自定义 MCP Server（让 Agent 操作数据库）

创建 `mcp_server/sprint_tools.py`：

```python
"""自定义 MCP Server：需求排期工具集"""
from mcp.server.fastmcp import FastMCP
import sqlite3
import json

DB_PATH = "data/sprint_planner.db"
mcp = FastMCP("Sprint Planner Tools")

@mcp.tool()
def list_requirements(status: str = "all") -> str:
    """查看需求池。status可选: all/new/in_progress/done"""
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    if status == "all":
        rows = conn.execute("SELECT * FROM requirements ORDER BY priority").fetchall()
    else:
        rows = conn.execute("SELECT * FROM requirements WHERE status=?", (status,)).fetchall()
    conn.close()
    return json.dumps([dict(r) for r in rows], ensure_ascii=False, indent=2)

@mcp.tool()
def add_requirement(title: str, description: str, priority: str) -> str:
    """添加新需求到需求池"""
    conn = sqlite3.connect(DB_PATH)
    cur = conn.execute(
        "INSERT INTO requirements (title, description, priority) VALUES (?,?,?)",
        (title, description, priority),
    )
    conn.commit()
    req_id = cur.lastrowid
    conn.close()
    return f"✅ 需求已创建，ID={req_id}"

@mcp.tool()
def add_user_stories(req_id: int, stories_json: str) -> str:
    """为某个需求批量添加用户故事。stories_json格式: [{"story":"...", "acceptance_criteria":"...", "estimated_days": 3}]"""
    stories = json.loads(stories_json)
    conn = sqlite3.connect(DB_PATH)
    ids = []
    for s in stories:
        cur = conn.execute(
            "INSERT INTO user_stories (req_id, story, acceptance_criteria, estimated_days) VALUES (?,?,?,?)",
            (req_id, s["story"], s.get("acceptance_criteria", ""), s.get("estimated_days", 0)),
        )
        ids.append(cur.lastrowid)
    conn.commit()
    conn.close()
    return f"✅ 已添加 {len(ids)} 个用户故事，IDs={ids}"

@mcp.tool()
def check_schedule_conflicts(assignee: str, start_date: str, end_date: str) -> str:
    """检查某开发者在指定时间段是否有排期冲突"""
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    conflicts = conn.execute("""
        SELECT s.*, us.story FROM schedules s
        JOIN user_stories us ON s.story_id = us.id
        WHERE s.assignee = ?
        AND s.start_date <= ? AND s.end_date >= ?
    """, (assignee, end_date, start_date)).fetchall()
    conn.close()
    if not conflicts:
        return f"✅ {assignee} 在 {start_date} ~ {end_date} 期间无冲突"
    return f"⚠️ 发现 {len(conflicts)} 个冲突:\n" + json.dumps(
        [dict(c) for c in conflicts], ensure_ascii=False, indent=2
    )

@mcp.tool()
def create_schedule(story_id: int, start_date: str, end_date: str, assignee: str) -> str:
    """为用户故事创建排期"""
    conn = sqlite3.connect(DB_PATH)
    conn.execute(
        "INSERT INTO schedules (story_id, start_date, end_date, assignee) VALUES (?,?,?,?)",
        (story_id, start_date, end_date, assignee),
    )
    conn.commit()
    conn.close()
    return f"✅ 排期已创建: Story#{story_id} → {assignee} ({start_date} ~ {end_date})"

@mcp.tool()
def get_sprint_overview(sprint: str = "2026-S1") -> str:
    """获取某个Sprint的全局排期视图"""
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    rows = conn.execute("""
        SELECT us.story, us.assignee, us.estimated_days, s.start_date, s.end_date, s.status
        FROM user_stories us
        LEFT JOIN schedules s ON us.id = s.story_id
        WHERE us.sprint = ?
        ORDER BY s.start_date
    """, (sprint,)).fetchall()
    conn.close()
    return json.dumps([dict(r) for r in rows], ensure_ascii=False, indent=2)

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

在 `.cursor/mcp.json` 中注册：

```json
{
  "mcpServers": {
    "sprint-planner": {
      "command": "python",
      "args": ["mcp_server/sprint_tools.py"]
    }
  }
}
```

#### Step 3：在 Dify 中搭建 Multi-Agent Workflow

```
[开始: 需求描述 + 优先级 + 目标上线时间]
   ↓
[Agent节点1: 需求拆解]
   角色: 资深产品经理
   工具: 无（纯LLM推理）
   Prompt: "将以下需求拆解为用户故事，每个story包含:
            - 故事描述（As a...I want...So that...）
            - 验收标准（Given/When/Then）
            - 预估复杂度（S/M/L/XL）"
   输出: stories_json
   ↓
[Agent节点2: 工时评估]
   角色: 技术Lead
   工具: Dify 内置 HTTP 工具（调用你的 MCP Server API）
   Prompt: "基于拆解的用户故事，评估每个story的人天工时。
            规则: S=1-2天, M=3-5天, L=5-8天, XL=8-13天
            考虑技术复杂度和依赖关系。"
   输出: estimated_stories
   ↓
[Agent节点3: 冲突检测]
   角色: 项目管理专家
   工具: HTTP请求（查询现有排期数据库）
   Prompt: "检查团队成员排期冲突，资源是否充足。
            当前团队: 张三(前端), 李四(后端), 王五(全栈)
            输出冲突列表和建议。"
   输出: conflicts + suggestions
   ↓
[汇总节点: 生成排期方案]
   Prompt: "综合以上信息，生成:
            1. 排期表（表格格式）
            2. 关键路径标注
            3. 风险清单
            4. 里程碑节点"
   ↓
[结束: 返回完整排期方案]
```

#### Step 4：通过 LiteLLM 路由不同模型（成本优化）

在你本地已跑的 LiteLLM（`127.0.0.1:4000`）中配置模型路由：

```yaml
# litellm_config.yaml
model_list:
  - model_name: "planner-heavy"    # 协调Agent + 冲突检测（需要强推理）
    litellm_params:
      model: "anthropic/claude-sonnet-4-20250514"
      api_key: "sk-xxx"

  - model_name: "planner-light"    # 需求拆解 + 工时评估（较简单任务）
    litellm_params:
      model: "anthropic/claude-haiku-3"
      api_key: "sk-xxx"

router_settings:
  routing_strategy: "simple-shuffle"  # 同名模型有多key时做负载均衡
```

Python 调用端统一走 LiteLLM：

```python
import openai

client = openai.OpenAI(base_url="http://127.0.0.1:4000/v1", api_key="sk-litellm")

# 简单任务用轻量模型
stories = client.chat.completions.create(
    model="planner-light",
    messages=[{"role": "user", "content": "拆解需求: 用户画像功能..."}],
)

# 复杂推理用重量模型
conflicts = client.chat.completions.create(
    model="planner-heavy",
    messages=[{"role": "user", "content": "检测以下排期冲突..."}],
)
```

#### Step 5：Cursor 中的完整使用流程

在 Cursor Composer 中直接说：

```
帮我排期一个新需求：
- 需求：用户画像功能，支持标签管理和行为分析
- 优先级：P1
- 目标上线：2026年Q3
- 团队：张三(前端)、李四(后端)
```

Agent 会自动：
1. 通过 MCP 调用 `add_requirement` 写入需求池
2. 调用 Dify Workflow 拆解 → 评估 → 检测冲突
3. 通过 MCP 调用 `create_schedule` 写入排期
4. 输出完整排期方案 + 风险报告

### 3.4 验收标准

- [ ] MCP Server 注册成功，Cursor 可直接调用 6 个工具
- [ ] Dify Multi-Agent Workflow 可独立通过 API 运行
- [ ] 输入一个需求，自动输出完整排期方案（含拆解/工时/冲突/甘特图）
- [ ] LiteLLM 路由正常，不同 Agent 使用不同模型
- [ ] 所有数据持久化到 SQLite，可查询历史

---

## 项目间递进关系

```
Demo 1 (基础)              Demo 2 (进阶)              Demo 3 (综合)
───────────              ───────────              ───────────
Constraints              RAG 知识库               Multi-Agent
Skills                   向量检索                 MCP Server
Prompt 工程              混合检索策略              工具调用
Dify Workflow            Dify Knowledge           Dify Agent
SQLite 基础              PostgreSQL + pgvector    LiteLLM 路由
                                                  完整数据模型
```

每个 Demo 的技能是累加的——做完 Demo 1 掌握的 Dify Workflow + SQLite 基础，在 Demo 2/3 中会复用和深化。

---

## 技术栈速查表

| 组件 | 用途 | 安装/配置 |
|:---|:---|:---|
| **Cursor Pro+** | 主 IDE + Agent + MCP | 已有 |
| **Dify** | Workflow + Knowledge + Agent | `docker compose up -d`（自部署）或 dify.ai（云端） |
| **Python 3.11+** | 脚本 + MCP Server | 系统自带或 `brew install python` |
| **SQLite** | 轻量数据库 | Python 内置，零配置 |
| **PostgreSQL + pgvector** | 向量数据库（Demo 2） | `brew install postgresql` + pgvector 扩展 |
| **LiteLLM** | 模型路由网关 | 已部署在 `127.0.0.1:4000` |
| **MCP Python SDK** | 自定义 MCP Server | `pip install mcp` |
| **dify-python-sdk** | Dify API 客户端 | `pip install dify-python-sdk` |

---

## 给 Hanhui 的建议

1. **从 Demo 1 开始**，它最轻量，2-3 天可跑通，建立信心
2. **Demo 2 是 RAG 的"必修课"**，理解 Embedding/检索/重排序后，你能评审任何 RAG 方案
3. **Demo 3 是综合大作业**，做完后你对 Agent 编排的理解会超过 90% 的产品经理
4. **全程用 Cursor Agent 写代码**——你不需要自己写，但你需要**审查**它写的代码，这本身就是技术型 PM 的核心能力
5. **每个 Demo 完成后写一篇复盘**，这些就是你的作品集

---

*祝练手愉快，下午看完如有疑问随时 ping 我。*
