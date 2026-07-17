# 客服工单自动处理 Agent

一个面向电商售后场景的客服工单自动处理作品集项目。项目用 FastAPI、PostgreSQL、Qdrant RAG、规则状态机 Agent、React 工作台和离线评测系统，串起“用户问题识别、订单查询、政策检索、处理建议、工单流转、质量评估”的完整闭环。

当前版本主流程使用确定性规则状态机，并提供可选的 LLM-assisted 智能增强模式。智能增强接入阿里云 DashScope，只辅助非标准表达理解和客服回复润色，不让大模型直接执行退款、支付、库存或工单状态修改。没有 API Key 时默认 `rules` 模式仍可完整运行。

## 核心亮点

- 真实后端链路：FastAPI + SQLAlchemy 2.0 + PostgreSQL，不使用内存假数据。
- 业务工具 API：订单、用户、物流、退款资格、退款金额建议、工单创建、状态更新、人工升级。
- 政策知识库 RAG：本地政策文档切分、embedding、Qdrant 检索，支持 hashing backend 便于本地测试。
- 可控 Agent 工作流：规则状态机串联意图识别、信息检查、政策检索、工具选择、业务动作、回复生成。
- 安全边界清晰：退款只给审核建议，不真实退款，不修改支付状态，不修改库存。
- 业务化前端：客服处理台、工单中心、政策知识库、质检中心、后台管理，默认面向客服用户而不是开发调试台。
- 可选智能增强：客服可在处理台打开“智能增强”，偏好保存在浏览器本地；无 Key 或调用失败时自动回退规则处理。
- 离线评测闭环：50 条售后评测集，按 intent、工具调用、政策命中、人工升级和延迟统计质量。
- 人工反馈闭环：客服可对 AI 建议标记采纳、修改后采纳或不采纳，质检中心汇总反馈原因和样本。
- 后台政策管理：管理员可创建草稿、编辑、发布和停用政策，发布/停用后重建政策检索索引。
- 模型效果对比：后台可对比 rules 与 llm_assisted 的评测结果，并一键生成增强模式评测任务。
- 工单中心增强：支持状态/类型/优先级筛选、真实后端分页、详情联动和人工处理状态修改。

## 当前评测结果

最近一次评测来自 `data/eval/eval_report.json`：

| 指标 | 结果 |
| --- | ---: |
| 总用例 | 50 |
| 通过 | 50 |
| 失败 | 0 |
| 问题类型识别 | 100% |
| 处理流程匹配 | 100% |
| 政策命中 | 100% |
| 人工升级判断 | 100% |
| 平均响应耗时 | 43.21 ms |
| P95 响应耗时 | 56.70 ms |

最近验证状态：

- 后端全量测试：102 passed。
- 前端 `npm run build`：通过。
- 前端轻量回归：`api.request`、`admin-eval-trigger-source`、`admin-layout-css`、`knowledge-page-source`、`tickets-filter-source` 全部通过。
- rules 评测内联验证：50 total，50 passed，0 failed。
- 在受限沙箱中运行 `scripts.run_eval --mode rules` 写入 `data/eval/eval_report.json` 可能遇到文件权限限制；评测逻辑本身可通过。

## 系统架构

```text
React 客服工作台
  |
  | HTTP
  v
FastAPI 后端
  |
  |-- 业务工具服务 -> PostgreSQL
  |       |-- users / products / orders / order_items / shipments / tickets
  |       |-- agent_feedback / admin_users / admin_sessions / policy_documents
  |
  |-- 政策检索服务 -> Qdrant
  |       |-- data/knowledge/*.md
  |       |-- published admin policies
  |
  |-- 规则状态机 Agent
  |       |-- intent_classification
  |       |-- optional LLM-assisted intent/reply help
  |       |-- information_check
  |       |-- policy_retrieval
  |       |-- tool_selection
  |       |-- business_action
  |       |-- response_generation
  |       |-- trace_recording
  |
  |-- 工单中心
  |       |-- filters / pagination / detail / status update
  |
  |-- 后台管理
  |       |-- admin auth / policy drafts / publish / disable
  |       |-- rules vs llm_assisted comparison
  |       |-- background llm_assisted evaluation job
  |
  |-- 人工反馈
  |       |-- accepted / edited / rejected
  |       |-- feedback summary
  |
  |-- 评测系统
          |-- data/eval/customer_service_eval_cases.jsonl
          |-- data/eval/eval_report.json
          |-- data/eval/eval_report_llm_assisted.json
          |-- data/eval/history/*.json
```

更多说明见 [系统架构文档](docs/architecture.md) 和 [演示指南](docs/demo-guide.md)。

## 功能页面

- 客服处理台：输入用户问题，展示问题类型、处理方式、判断置信度、关联工单、处理结论、客服回复和参考政策。
- 工单中心：查看售后工单队列、状态、优先级、关联订单和处理结果；支持筛选、后端分页、详情联动和人工状态修改。
- 政策知识库：浏览售后政策，按用户问题检索相关政策片段，支持查看政策全文。
- 质检中心：查看自动处理质量、按场景质量、历史趋势、失败案例回放和人工反馈汇总。
- 后台管理：管理员登录后维护政策草稿、发布/停用政策、查看 rules 与 llm_assisted 模型效果对比，并触发增强模式评测。

## 项目截图

### 客服处理台
<img src="docs/assets/01.png" alt="客服处理台" width="900">


## 目录结构

```text
kefuAgent/
  backend/
    app/
      api/              FastAPI 路由
      db/               SQLAlchemy session 和依赖
      models/           用户、商品、订单、物流、工单模型
      schemas/          Pydantic 请求/响应结构
      services/         工具服务、RAG、Agent、评测、反馈、后台管理逻辑
    scripts/            初始化、seed、知识库导入、评测、后台账号脚本
    tests/              后端测试
    pyproject.toml
  frontend/
    src/
      App.tsx           React 工作台页面
      api.ts            前端 API 调用
      presentation.ts   业务展示文案映射
      styles.css        企业后台样式
      types.ts          前端类型
  data/
    knowledge/          售后政策文档
    eval/               评测集、最新报告、历史报告
  docs/
    architecture.md
    demo-guide.md
  docker-compose.yml
  README.md
```


### 环境要求

- Python 3.11+
- Node.js 18+
- Docker Desktop
- Git


### 1. 启动 PostgreSQL 和 Qdrant

```powershell
docker compose up -d postgres qdrant
```

### 2. 安装后端依赖

```powershell
cd kefuAgent\backend
py -3.11 -m venv .venv
.\.venv\Scripts\python.exe -m pip install --upgrade pip
.\.venv\Scripts\python.exe -m pip install -e ".[dev]"
```

如果本机没有 `py -3.11` 启动器，可以改用：

```powershell
python -m venv .venv
```

### 3. 初始化数据库和政策知识库

```powershell
cd kefuAgent\backend

$env:DATABASE_URL = 'postgresql+psycopg://postgres:yourPassword@localhost:5432/postgres'
$env:QDRANT_URL = 'http://localhost:6333'
$env:KEFU_EMBEDDING_BACKEND = 'hashing'
$env:AGENT_MODE = 'rules'

.\.venv\Scripts\python.exe -m scripts.init_db
.\.venv\Scripts\python.exe -m scripts.seed_db
.\.venv\Scripts\python.exe -m scripts.ingest_knowledge
```

### 4. 启动后端

继续在 `backend` 目录运行：

```powershell
cd kefuAgent\backend

$env:DATABASE_URL = 'postgresql+psycopg://postgres:yourPassword@localhost:5432/postgres'
$env:QDRANT_URL = 'http://localhost:6333'
$env:KEFU_EMBEDDING_BACKEND = 'hashing'
$env:AGENT_MODE = 'rules'

# 可选：阿里云 DashScope 智能增强。默认 rules，无 API Key 也能完整运行。
# $env:AGENT_MODE = 'llm_assisted'
# $env:DASHSCOPE_API_KEY = '<your-api-key>'
# $env:DASHSCOPE_MODEL = 'qwen-plus'

.\.venv\Scripts\python.exe -m uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```


### 5. 启动前端

另开一个 PowerShell 终端：

```powershell
cd kefuAgent\frontend
npm install
npm run dev
```

浏览器访问：`http://localhost:5173`

### 6. 可选：创建后台管理员

如果需要进入“后台管理”页面，另开终端运行：

```powershell
cd kefuAgent\backend
$env:DATABASE_URL = 'postgresql+psycopg://postgres:yourPassword@localhost:5432/postgres'
$env:ADMIN_USERNAME = 'admin'
$env:ADMIN_PASSWORD = 'change-me-before-real-use'
.\.venv\Scripts\python.exe -m scripts.seed_admin
```

## 验证命令

前端构建：

```powershell
cd kefuAgent\frontend
npm run build
```

后端测试：

```powershell
cd kefuAgent\backend

$env:DATABASE_URL = 'postgresql+psycopg://postgres:123456@localhost:5432/postgres'
$env:QDRANT_URL = 'http://localhost:6333'
$env:KEFU_EMBEDDING_BACKEND = 'hashing'

.\.venv\Scripts\python.exe -m pytest tests -q -p no:cacheprovider
```

运行评测：

```powershell
cd kefuAgent\backend

$env:DATABASE_URL = 'postgresql+psycopg://postgres:123456@localhost:5432/postgres'
$env:QDRANT_URL = 'http://localhost:6333'
$env:KEFU_EMBEDDING_BACKEND = 'hashing'

.\.venv\Scripts\python.exe -m scripts.run_eval
```

指定模式评测：

```powershell
.\.venv\Scripts\python.exe -m scripts.run_eval --mode rules
.\.venv\Scripts\python.exe -m scripts.run_eval --mode llm_assisted
```

前端轻量回归：

```powershell
cd kefuAgent\frontend
node tests\api.request.test.mjs
node tests\admin-eval-trigger-source.test.mjs
node tests\admin-layout-css.test.mjs
node tests\knowledge-page-source.test.mjs
node tests\tickets-filter-source.test.mjs
```

## 关键接口

- `GET /health`
- `POST /api/agent/process`
- `POST /api/agent/feedback`
- `POST /api/tools/policies/search`
- `GET /api/tickets`
- `GET /api/tickets/{ticket_number}`
- `PATCH /api/tickets/{ticket_number}/status`
- `GET /api/knowledge/documents`
- `GET /api/eval/latest`
- `GET /api/eval/history`
- `GET /api/eval/feedback-summary`
- `POST /api/admin/login`
- `POST /api/admin/logout`
- `GET /api/admin/policies`
- `POST /api/admin/policies`
- `PATCH /api/admin/policies/{policy_id}`
- `POST /api/admin/policies/{policy_id}/publish`
- `POST /api/admin/policies/{policy_id}/disable`
- `GET /api/admin/eval/compare`
- `POST /api/admin/eval/llm-assisted/run`
- `GET /api/admin/eval/llm-assisted/status`

## 安全边界

- 退款相关接口只给审核建议，不真实退款。
- 不修改支付状态。
- 不修改库存。
- 不修改退款状态。
- 不自动修改政策知识库，除非后台管理员手动发布或停用政策。
- LLM 不能决定工具调用结果、退款、库存、支付、权限或政策发布。
- DashScope API Key 只读取后端环境变量 `DASHSCOPE_API_KEY`，不进入前端、不入库、不写文件、不写日志。
- 投诉、账号等高风险场景默认转人工。
- 评测脚本默认使用外层事务 rollback，避免评测过程污染演示数据。

## LLM-assisted 智能增强

项目支持可选 `LLM-assisted` 模式：

- 默认仍是 `rules`，保证无 API Key 可运行。
- 请求体 `mode` 优先于环境变量 `AGENT_MODE`。
- 客服处理台提供“智能增强”开关，开关状态保存在浏览器本地。
- 有阿里云 DashScope API Key 且开启增强时，LLM 只辅助意图理解和回复润色。
- 无 `DASHSCOPE_API_KEY` 或 LLM 调用失败时，`llm_assisted` 会自动回退 rules。
- 所有订单、物流、退款资格和政策事实仍来自 PostgreSQL、工具 API 和 Qdrant。
- 退款、支付、库存、工单状态等高风险判断仍由确定性规则和工具校验。
- 评测脚本支持 `--mode rules|llm_assisted`，默认仍跑 rules 基线。
- `llm_assisted` 评测默认写入 `data/eval/eval_report_llm_assisted.json`，避免覆盖 rules 基线报告。
- 后台管理页支持一键生成增强模式评测；任务状态为进程内内存状态，服务重启后会回到 idle，但已生成的报告文件仍保留在 `data/eval`。

## 最近重要能力

- 退款审核会创建 `refund` 工单，并返回关联工单号；仍只是审核建议，不会真实退款、不会修改支付状态或库存。
- 工单中心筛选后，右侧详情会跟随当前列表；如果当前页无结果，会清空详情。
- 工单列表使用后端真实分页，`GET /api/tickets` 支持 `page`、`page_size`、`total`、`total_pages`，并保留旧 `limit` 参数兼容。
- 政策知识库卡片显示摘要，查看全文使用后端 `content` 字段。
- 后台发布或停用政策时会更新政策检索索引；如果向量服务不可用，会返回安全的中文业务错误。
- 后台模型效果对比在缺少增强模式报告时仍返回 200，并在页面展示空状态。
