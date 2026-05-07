# Demo 1：PRD 智能审查助手 — 手搓复盘报告

> **执行人**：Hanhui  
> **执行日期**：2026年5月5日-7日  
> **测试 PRD**：PRD-资金-电汇流水单-水单影像改造-v20260505.md  
> **技术栈**：Cursor Pro+ / Dify Workflow（SSE） / Python / SQLite

---

## 一、最终项目结构

```
project/
├── .cursor/
│   ├── rules/
│   │   └── prd-review.mdc          ← Constraints（约束规则）
│   └── skills/
│       └── prd-review-skill/
│           └── SKILL.md             ← Skills（执行SOP）
├── scripts/
│   ├── call_dify_review.py          ← 主脚本：调 Dify API + 写 SQLite
│   └── __pycache__/
├── data/
│   └── prd_reviews.db               ← SQLite 审查记录
├── docs/
│   └── prd/
│       └── PRD-资金-电汇流水单-水单影像改造-v20260505.md  ← 测试用真实PRD
└── outputs.json                      ← 审查结果全文
```

---

## 二、完整执行链路（从你敲命令到拿到报告）

下面是 Demo 1 从头到尾的**实际执行顺序**，编号对应下方详解：

```
你（用户）
  │
  │ ① 在 Cursor Composer 中说："帮我审查这篇PRD"
  │    或直接在终端执行 python3 命令
  ▼
┌─────────────────────────────────────────────────────────┐
│  ② Cursor Agent 加载上下文                                │
│     ├── 自动匹配 .cursor/rules/prd-review.mdc           │
│     │   （因为文件路径命中 globs: docs/prd/**）            │
│     │   → 注入 System Prompt（Constraints 约束生效）      │
│     │                                                    │
│     └── 自动/手动加载 .cursor/skills/SKILL.md            │
│         → Agent 获得执行步骤 SOP                          │
└──────────────────────────┬──────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────┐
│  ③ Agent 按 SKILL.md 的 Step 1 执行                      │
│     → 读取目标 PRD 文件全文                               │
└──────────────────────────┬──────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────┐
│  ④ Agent 按 SKILL.md 的 Step 2 执行                      │
│     → 调用 Shell 工具:                                    │
│       python3 scripts/call_dify_review.py \              │
│         docs/prd/PRD-资金-电汇流水单-...v20260505.md      │
└──────────────────────────┬──────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────┐
│  ⑤ call_dify_review.py 执行                              │
│     │                                                    │
│     ├── 读取 PRD 文件内容                                 │
│     ├── POST https://api.dify.ai/v1/workflows/run        │
│     │   Headers: Authorization: Bearer app-qevzx8d...    │
│     │   Body: {                                          │
│     │     "inputs": {"prd_content": "<PRD全文>"},         │
│     │     "response_mode": "streaming",  ← SSE模式       │
│     │     "user": "hanhui-pm"                            │
│     │   }                                                │
│     │                                                    │
│     │         ┌──── Dify Cloud ────────────────┐         │
│     │         │                                │         │
│     │         │  [LLM节点1: 格式与完整性检查]    │         │
│     │         │         ↓                      │         │
│     │         │  [LLM节点2: 逻辑一致性分析]     │         │
│     │         │         ↓                      │         │
│     │         │  [LLM节点3: 评分汇总]           │         │
│     │         │                                │         │
│     │         └──── SSE 流式返回 ──────────────┘         │
│     │                                                    │
│     ├── 解析 SSE 事件流，拼接完整响应                      │
│     ├── 写入 SQLite: data/prd_reviews.db                  │
│     └── 输出结果到终端 + outputs.json                     │
└──────────────────────────┬──────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────┐
│  ⑥ Agent 按 SKILL.md 的 Step 3-5 执行                    │
│     → 解析审查结果 JSON                                   │
│     → 判断分数等级                                        │
│     → 以 Markdown 表格输出给你                            │
└─────────────────────────────────────────────────────────┘
```

---

## 三、Constraints（约束）调用原理详解

### 3.1 什么是 Constraints

Constraints = **给模型划的框**。你在 `.cursor/rules/prd-review.mdc` 里写的每一条规则，本质上是**被注入到 LLM 的 System Prompt 里**，让模型"不得不"按你的规矩来。

### 3.2 它是怎么被调用的

```
                  你的 .mdc 文件
                       │
                       ▼
┌───────────────────────────────────────────────┐
│  prd-review.mdc                               │
│                                               │
│  ---                                          │
│  description: PRD审查规则                      │
│  globs: ["*.prd.md", "docs/prd/**"]  ← 触发器 │
│  ---                                          │
│                                               │
│  ## PRD 审查约束                               │
│  ...（你写的所有约束内容）                       │
│                                               │
└───────────────────────────────────────────────┘
```

**触发机制（关键）**：

| 阶段 | 发生了什么 | 谁在做 |
|:---|:---|:---|
| **1. 文件匹配** | 当你在 Cursor 中打开或引用 `docs/prd/PRD-资金-电汇流水单-...md` 时，Cursor 检查所有 `.mdc` 文件的 `globs` 字段 | Cursor IDE 引擎 |
| **2. 命中规则** | `docs/prd/**` 匹配成功 → `prd-review.mdc` 被激活 | Cursor Rules 系统 |
| **3. 注入 Prompt** | `.mdc` 中的全部文本被拼接到发给 LLM 的 System Prompt 中 | Cursor Agent |
| **4. 约束生效** | LLM 在生成回复时，被这些约束"框住"——必须按5个维度打分、必须输出JSON、不能编造问题 | LLM（Claude等） |

**核心理解**：`.mdc` 文件 ≠ 代码，它是**声明式的 Prompt 片段**。Cursor 的 Rules 系统根据 `globs` 自动决定何时注入——你不需要手动调用它。

### 3.3 你的 Constraints 实际效果

从终端输出看，模型确实遵循了约束：

```
✅ 按5个维度评分（完整度与细节 4.5/5 分...）
✅ 输出了结构化评审报告
✅ 标注了 critical/major/minor 问题
✅ 给出了具体的 Action Items（不是泛泛而谈）
✅ 引用了原文位置（如 "Draft 状态的对外展示"）
```

### 3.4 Constraints 的两层生效位置

实际上你的约束在**两个地方**同时生效：

```
层级1：Cursor Agent 侧（本地）
  └── .cursor/rules/prd-review.mdc
      → 约束 Cursor Agent 的行为（如何调用脚本、如何解读结果）

层级2：Dify Workflow 侧（云端）
  └── Dify 各 LLM 节点的 System Prompt
      → 约束 Dify 中模型的审查行为（输出格式、评分标准）
```

**这是很多人忽略的**——你的审查标准需要在两侧都定义，否则 Cursor 端知道怎么评分但 Dify 端不知道，或反之。在你的实现中，核心审查逻辑放在了 Dify Workflow 的 Prompt 里，`.mdc` 主要约束的是 Cursor Agent 的调用行为和结果解读。

---

## 四、Skills（技能）调用原理详解

### 4.1 什么是 Skills

Skills = **给 Agent 的操作手册（SOP）**。如果说 Constraints 是"你应该遵守什么规矩"，那 Skills 是"你应该按什么步骤做事"。

### 4.2 它是怎么被调用的

```
┌────────────────────────────────────────────────────────┐
│  .cursor/skills/prd-review-skill/SKILL.md              │
│                                                        │
│  # PRD 审查技能                                         │
│                                                        │
│  ## 触发条件                                            │
│  当用户要求审查 PRD 文档时使用此技能。                     │
│                                                        │
│  ## 执行步骤                                            │
│  ### 1. 读取 PRD 文档                                   │
│  ### 2. 调用 Dify 审查工作流                             │
│  ### 3. 解析审查结果                                     │
│  ### 4. 写入数据库                                      │
│  ### 5. 输出报告                                        │
└────────────────────────────────────────────────────────┘
```

**触发机制**：

| 类型 | 触发方式 | 说明 |
|:---|:---|:---|
| **手动触发** | 在 Composer 中 `@SKILL.md` 或提到"审查PRD" | Agent 读取 SKILL.md 后按步骤执行 |
| **自动触发** | Cursor 的动态上下文发现机制匹配关键词 | Cursor 2.4+ 支持根据任务语义自动加载 Skill |
| **规则关联** | `.mdc` 中引用 Skill 路径 | Rules 触发后可进一步加载 Skill |

### 4.3 Skills vs Constraints 的本质区别

这是理解 Cursor AI 架构的**关键认知**：

```
┌──────────────────────────────────────────────────┐
│                                                  │
│   Constraints（.mdc Rules）                      │
│   ─────────────────────                          │
│   性质：声明式                                    │
│   作用：告诉模型 "什么能做、什么不能做"             │
│   注入方式：自动（globs匹配 → System Prompt）     │
│   类比：交通法规 🚦                               │
│                                                  │
│   Skills（SKILL.md）                             │
│   ────────────────                               │
│   性质：过程式                                    │
│   作用：告诉模型 "按什么顺序、怎么一步步做"         │
│   注入方式：手动或半自动（@引用 / 语义匹配）       │
│   类比：导航路线 🗺️                              │
│                                                  │
│   两者协作：                                      │
│   Constraints 画边界 → Skills 定路径              │
│   Agent 在边界内、沿路径执行                       │
│                                                  │
└──────────────────────────────────────────────────┘
```

### 4.4 你的 Skills 实际执行轨迹

从截图中的终端输出，可以还原 Agent 的实际执行轨迹：

```
SKILL Step 1 → 读取 PRD 文件
  ✅ 读取了 docs/prd/PRD-资金-电汇流水单-水单影像改造-v20260505.md

SKILL Step 2 → 调用 Dify 审查工作流
  ✅ python3 scripts/call_dify_review.py docs/prd/PRD-资金-...md
  ✅ Dify API 返回了审查结果（SSE 流式）

SKILL Step 3 → 解析审查结果
  ⚠️ "未能从正文解析总分，请查看 outputs.json 全文"
  （Dify返回的JSON格式可能与预期的score字段不完全匹配）

SKILL Step 4 → 写入数据库
  ✅ 结果写入 data/prd_reviews.db

SKILL Step 5 → 输出报告
  ✅ 终端输出了完整审查报告（5维度评分 + Action Items）
```

---

## 五、Dify Workflow 的执行细节

### 5.1 你的实际技术选型

从代码中可以看到，你在实战中做了关键决策：

| 设计方案中 | 你的实际选择 | 为什么 |
|:---|:---|:---|
| `response_mode: "blocking"` | `response_mode: "streaming"` (SSE) | blocking 模式在长 PRD 审查时会触发网关 504 超时 |
| 固定超时 | `CONNECT_TIMEOUT_S=60` + `STREAM_READ_TIMEOUT_S=None` | SSE 模式下读取无上限，由服务端结束或轮询兜底 |
| 一次性返回 | SSE 事件流逐步拼接 | 避免长时间等待无反馈 |

**这是一个重要的工程经验**：Dify Workflow 如果包含多个 LLM 节点（3个串联），总耗时可能超过 30-60 秒，blocking 模式极易超时。SSE 是更稳健的选择。

### 5.2 Dify Workflow 内部执行流

```
                 POST /v1/workflows/run
                 response_mode: streaming
                         │
                         ▼
            ┌─── Dify Cloud Workflow ───┐
            │                           │
  t=0s      │  [开始节点]               │ → event: workflow_started
            │     ↓                     │
            │  [变量: prd_content]      │
            │     ↓                     │
  t≈5-10s   │  [LLM节点1]              │ → event: node_started
            │  格式与完整性检查          │ → event: node_finished
            │  "检查PRD格式规范性..."    │    (含节点输出)
            │     ↓                     │
  t≈15-25s  │  [LLM节点2]              │ → event: node_started
            │  逻辑一致性分析            │ → event: node_finished
            │  "深入分析需求逻辑..."     │    (含节点输出)
            │     ↓                     │
  t≈30-45s  │  [LLM节点3]              │ → event: node_started
            │  评分汇总                  │ → event: node_finished
            │  "按5维度打分..."          │    (含最终评分)
            │     ↓                     │
  t≈45-60s  │  [结束节点]               │ → event: workflow_finished
            │                           │    (含完整outputs)
            └───────────────────────────┘
                         │
                         ▼
              Python SSE 客户端逐事件解析
              拼接最终 outputs JSON
```

### 5.3 实际审查输出分析

从终端输出可以看到模型产出了高质量的审查报告：

**评分结果**（部分可见）：

| 维度 | 得分 | 说明 |
|:---|:---:|:---|
| 完整度与细节把控 | 4.5/5 | 文档结构规范，In Scope/Out of Scope 界定清晰 |
| （其余维度） | — | 完整结果在 outputs.json 中 |

**发现的关键问题**（从终端可见部分）：
- TBD（待定项）非常精准，说明作者需要明确哪些技术细节
- OCR 失败的降级路径未明确定义
- Excel 导入异常策略需完善
- R1 风险（当 Excel 行数据提取失败时的业务规则）需明确

**Action Items 输出**：
1. PRD 中补充/明确 TBD 待定项
2. 明确 OCR 失败降级路径（处理中/需人工操作的视觉反馈）
3. 完善 Excel 导入异常策略

---

## 六、SQLite 数据持久化

### 6.1 数据模型

```sql
CREATE TABLE reviews (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    filename TEXT,              -- 审查的PRD文件路径
    score INTEGER,              -- 总分（0-100）
    result_json TEXT,           -- 完整审查结果JSON
    reviewed_at TIMESTAMP       -- 审查时间
);
```

### 6.2 数据流

```
Dify API 返回 → Python 解析 → INSERT INTO reviews → prd_reviews.db
                                                          │
                                                          ▼
                                                  可被 MCP Server
                                                  或直接 SQL 查询
```

### 6.3 实际遇到的问题

从终端输出 `"未能从正文解析总分"` 可以推断：

- Dify Workflow 的最终输出格式可能不是严格的 `{"score": 85, ...}` JSON
- 而是更接近自然语言的评审报告（包含评分但未以JSON字段形式输出）
- Python 脚本用正则/JSON 解析时未命中 `score` 字段

**这是典型的 Prompt 工程问题**——需要在 Dify 的最终 LLM 节点中更强地约束输出格式，例如：
```
你必须严格按以下JSON格式输出，不要包含任何其他文本：
{"score": <总分>, "dimensions": [...], "issues": [...], "suggestions": [...]}
```

---

## 七、关键原理总结

### 7.1 Constraints + Skills + Dify 的三层架构

```
┌─────────────────────────────────────────────────────────────┐
│                     用户层（你）                              │
│  "帮我审查这篇PRD"                                          │
└─────────────────────────┬───────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  第一层：Cursor Rules（Constraints）                         │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ .cursor/rules/prd-review.mdc                          │  │
│  │                                                       │  │
│  │ 作用：约束 Agent 行为边界                               │  │
│  │ 注入方式：globs 匹配 → 自动注入 System Prompt          │  │
│  │ 生效时机：打开/引用 docs/prd/** 下文件时                │  │
│  │                                                       │  │
│  │ 约束了什么：                                           │  │
│  │  • 必须输出JSON格式（score + issues + suggestions）    │  │
│  │  • 必须按5个维度各20分评审                             │  │
│  │  • 不能编造问题                                        │  │
│  │  • critical问题必须给修改示例                           │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────┬───────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  第二层：Cursor Skills（SOP执行流程）                        │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ .cursor/skills/prd-review-skill/SKILL.md              │  │
│  │                                                       │  │
│  │ 作用：定义 Agent 的执行步骤                             │  │
│  │ 注入方式：手动@引用 或 语义匹配自动加载                  │  │
│  │ 生效时机：用户意图命中 "审查PRD" 时                     │  │
│  │                                                       │  │
│  │ 定义了什么：                                           │  │
│  │  Step1 → 读取PRD文件                                  │  │
│  │  Step2 → 调用 python scripts/call_dify_review.py      │  │
│  │  Step3 → 解析结果、判断分数等级                        │  │
│  │  Step4 → 写入SQLite                                   │  │
│  │  Step5 → 输出Markdown表格报告                          │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────┬───────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  第三层：Dify Workflow（实际审查引擎）                        │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ Dify Cloud / 私有部署                                  │  │
│  │ API: POST /v1/workflows/run                           │  │
│  │                                                       │  │
│  │ 作用：执行多步骤审查逻辑                                │  │
│  │ 调用方式：Python requests（SSE streaming）             │  │
│  │                                                       │  │
│  │ 执行了什么：                                           │  │
│  │  节点1 → 格式与完整性检查                              │  │
│  │  节点2 → 逻辑一致性分析                                │  │
│  │  节点3 → 评分汇总                                     │  │
│  │  返回 → SSE事件流 → Python拼接 → JSON输出             │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 一句话理解

> **Constraints 画圈（不许出界）→ Skills 画路（按步执行）→ Dify 干活（实际推理）**

三者的关系就像：
- **Constraints** = 公司制度（所有员工必须遵守）
- **Skills** = 岗位SOP（特定角色的操作手册）
- **Dify Workflow** = 外包团队（接到任务，按流程交付成果）

---

## 八、踩坑与经验

### 8.1 实际踩坑清单

| # | 坑 | 现象 | 解法 |
|:---:|:---|:---|:---|
| 1 | **Blocking 模式 504** | Dify Workflow 3个LLM节点串联，总耗时超网关超时 | 改用 `response_mode: streaming`（SSE） |
| 2 | **总分解析失败** | "未能从正文解析总分" | Dify 最终节点的 Prompt 需强约束 JSON 输出格式 |
| 3 | **SSE 超时控制** | 流式读取不知道何时结束 | `STREAM_READ_TIMEOUT_S=None` + `POLL_INTERVAL_S=2.0` 轮询兜底 |
| 4 | **globs 匹配范围** | `.mdc` 的 globs 写窄了可能不触发 | 实际文件路径必须严格匹配 glob 模式 |

### 8.2 你获得的关键认知

1. **Constraints 是被动触发的**——不是你调用它，是 Cursor 根据 globs 自动注入
2. **Skills 是主动/半主动加载的**——需要你@引用或语义匹配
3. **Dify Workflow 是同步阻塞的黑盒**——从 Cursor Agent 视角看，它就是一个 HTTP 调用
4. **SSE > Blocking**——凡是多节点 Workflow，必用 streaming 模式
5. **约束要两端对齐**——Cursor Rules 和 Dify Prompt 中的约束需要一致

---

## 九、与设计方案的对比

| 设计方案写的 | 实际做的 | 差异原因 |
|:---|:---|:---|
| `response_mode: blocking` | `response_mode: streaming` | 实测 blocking 超时，工程经验教训 |
| 直接解析 `score` 字段 | 正则+降级解析 | Dify 输出不一定是严格 JSON |
| MCP 连 SQLite 查询 | 暂未配置 MCP | 核心链路优先，MCP 是锦上添花 |
| 5个维度各20分 | 实际输出约为 X/5 格式 | Dify Prompt 中的评分制可调 |

---

## 十、下一步优化方向

### 短期（可立即做）
- [ ] 修复总分解析：在 Dify 最终节点强制输出 `{"score": N, ...}` JSON
- [ ] 配置 MCP Server 连 SQLite，实现自然语言查历史审查记录
- [ ] 添加更多测试 PRD，验证约束的泛化性

### 中期（Demo 2 衔接）
- [ ] 将审查标准文档化并灌入 RAG 知识库（与 Demo 2 打通）
- [ ] 审查结果可追溯引用来源（RAG检索）

### 长期（Demo 3 衔接）
- [ ] 审查结果自动同步到需求排期系统
- [ ] 多人协作审查（不同 Agent 审查不同维度）

---

## 十一、核心收获总结

```
┌─────────────────────────────────────────────────┐
│                                                 │
│   你通过 Demo 1 掌握了：                         │
│                                                 │
│   ✅ Constraints 声明式约束原理                  │
│      → 知道 .mdc 如何被自动注入                  │
│      → 知道 globs 匹配机制                       │
│                                                 │
│   ✅ Skills 过程式 SOP 原理                      │
│      → 知道 SKILL.md 如何指导 Agent 行动         │
│      → 理解 Constraints 画圈 + Skills 画路       │
│                                                 │
│   ✅ Dify Workflow API 集成                      │
│      → 学会了 SSE 流式调用（而非 blocking）       │
│      → 踩过超时的坑，理解了为什么                 │
│                                                 │
│   ✅ 端到端数据持久化                             │
│      → PRD → Dify → JSON → SQLite 全链路         │
│                                                 │
│   ✅ Prompt 工程实战经验                          │
│      → 发现"输出格式约束"需要在两端对齐           │
│      → 理解了结构化输出的难点                     │
│                                                 │
└─────────────────────────────────────────────────┘
```

---

*Demo 1 完成度：90%（核心链路跑通，MCP 查询 + 总分解析待优化）*  
*复盘时间：2026年5月7日*
