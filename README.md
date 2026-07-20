<p align="center">
  <img src="banner.png" alt="智能搜索助手 — QA-SYS" width="100%" />
</p>

<p align="center">
  <a href="https://github.com/231dff/qa-sys-"><img src="https://img.shields.io/badge/GitHub-231dff%2Fqa--sys---181717?logo=github" alt="GitHub Repo"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-green.svg" alt="MIT License"></a>
  <img src="https://img.shields.io/badge/Python-3.12+-blue?logo=python" alt="Python 3.12+">
  <img src="https://img.shields.io/badge/Next.js-16-000000?logo=next.js" alt="Next.js 16">
  <img src="https://img.shields.io/badge/React-19-61DAFB?logo=react" alt="React 19">
  <img src="https://img.shields.io/badge/Tests-131%20passed-success?logo=pytest" alt="131 tests">
  <img src="https://img.shields.io/badge/Docker-ready-2496ED?logo=docker" alt="Docker">
  <img src="https://img.shields.io/badge/Prometheus-metrics-E6522C?logo=prometheus" alt="Prometheus">
  <img src="https://img.shields.io/badge/Grafana-dashboards-F46800?logo=grafana" alt="Grafana">
</p>

<br>

# 🔍 智能搜索助手 — 实时联网搜索 Agent 系统

基于 **LangGraph** + **Tavily API** + **FastAPI** + **Next.js 16** 构建的实时联网搜索 Agent。支持 OpenAI 兼容的 LLM（包括阿里云百炼 DashScope / 通义千问）。

## ✨ 核心特性

- **实时搜索**: Tavily API 联网搜索，突破 LLM 知识时效限制
- **智能改写**: LLM 驱动的查询改写，将口语化提问转为精准搜索词
- **并行搜索**: 子查询并行搜索 + Tavily 原始分数过滤，省去冗余 LLM 打分调用
- **SSE 流式输出**: LLM 答案逐 Token 推送到前端，实时打字效果
- **多层容错**: 指数退避重试 + LLM 降级回答，异常期可用性 **95%+**
- **多轮对话**: 基于会话 ID 的上下文缓存，跨轮意图连贯
- **限流保护**: 搜索 API 配额管理与滑动窗口限流
- **连接池复用**: 全局共享 httpx 客户端，消除 TCP/TLS 握手开销
- **可观测性栈**: Prometheus 指标采集 + Grafana 可视化仪表盘 + 告警规则
- **双前端**: Next.js 16 生产前端 + Streamlit 开发调试面板

## 🏗️ 架构

```
┌──────────────────┐     SSE Stream       ┌──────────────┐       HTTP         ┌────────────────┐
│ Next.js 16 前端   │ ◄─────────────────── │  FastAPI 后端  │ ──────────────────► │ LLM API        │
│ (port 3001)       │   event: progress    │  (port 8000)   │   streaming POST   │ (OpenAI 兼容)   │
│                   │   event: token       │               │                    │                │
│ React 19 + TS     │   event: sources     │  LangGraph     │ ──────────────────► │ Tavily Search   │
│ Tailwind CSS 4    │   event: done        │  Agent Pipeline│   Search API       │ API             │
│ App Router        │   event: error       │               │                    │                │
├──────────────────┤                       ├───────────────┤                    └────────────────┘
│ Streamlit 面板    │                       │ 全局 httpx     │
│ (port 8501)       │                       │ 连接池复用     │
│ 开发调试用         │                       │               │
└──────────────────┘                       ├───────────────┤
                                           │ /api/metrics   │ ◄── Prometheus 抓取 (每 15s)
                                           │ Prometheus 指标 │
┌──────────────────┐                       └───────────────┘
│ Grafana 仪表盘    │ ◄─── PromQL ──── ┌──────────────────┐
│ (port 3002)       │                  │ Prometheus        │
│ 可用性 SLA 监控   │                  │ (port 9090)       │
└──────────────────┘                  │ 抓取 + 告警        │
                                      └──────────────────┘
```

### Agent 工作流

```
┌──────────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  用户输入     │ ──→ │ 查询改写  │ ──→ │ 并行搜索  │ ──→ │ 答案生成  │
└──────────────┘     └──────────┘     └──────────┘     └──────────┘
                           │                │                │
                     意图识别 +          子查询并行          错误恢复
                     子查询拆分           跳过LLM打分        (降级回答)
```

## 📁 项目结构

```
qa/
├── agent/                          # Agent 核心模块
│   ├── __init__.py                 # 模块入口
│   ├── graph.py                    # LangGraph 状态图 (核心编排)
│   ├── models.py                   # Pydantic 数据模型 + AgentState TypedDict
│   └── streaming.py                # SSE 流式编排器 (生产者-消费者模式)
├── tools/                          # 工具注册中心
│   ├── __init__.py                 # 工具模块
│   └── registry.py                 # 4 个工具: 搜索/改写/打分/降级 + 组合流水线
├── memory/                         # 会话缓存
│   ├── __init__.py                 # 内存模块
│   └── session_store.py            # 会话缓存 (TTL 过期)
├── utils/                          # 工具函数
│   ├── __init__.py                 # 工具函数模块
│   ├── helpers.py                  # URL 去重, Token 计数, 限流, 上下文裁剪
│   ├── http_client.py              # 全局共享 HTTP 客户端 (连接池复用)
│   └── metrics.py                  # Prometheus 指标 + 滑动窗口可用性统计 (95% SLA)
├── monitoring/                     # 可观测性栈
│   ├── prometheus/
│   │   ├── prometheus.yml          # Prometheus 配置 (15s 抓取间隔)
│   │   └── alerts.yml              # 告警规则 (高错误率/高延迟/服务宕机)
│   └── grafana/
│       ├── dashboards/
│       │   ├── availability.json   # 可用性监控 Dashboard (4 区域 11 面板)
│       │   └── dashboard.yml       # Dashboard 自动加载配置
│       └── datasources/
│           └── prometheus.yml      # Prometheus 数据源
├── tests/                          # 测试 (131 条)
│   ├── __init__.py                 # 测试包
│   ├── test_core.py                # 单元测试 (去重/Token/限流/会话/模型)
│   ├── test_registry.py            # 工具注册中心测试 (搜索/改写/打分/降级/流水线)
│   ├── test_graph.py               # 状态图节点 + SearchAgent 类测试
│   ├── test_config.py              # 配置加载 + 单例测试
│   └── test_e2e.py                 # 端到端流水线 + 路由 + 会话测试
├── evals/                          # 行为评估
│   ├── __init__.py                 # 评估包
│   ├── golden_dataset.json         # 黄金行为数据集 (33 用例, 92 断言)
│   ├── evaluate.py                 # 自动化评估框架 (7 类别, HTML/JSON 报告)
│   ├── benchmark.py                # 延迟 & Token 基准测试 (对比优化前后)
│   ├── report.html                 # HTML 评估报告 (自动生成)
│   └── benchmark.html              # HTML 基准测试报告 (自动生成)
├── frontend/                       # Next.js 16 前端
│   ├── app/
│   │   ├── layout.tsx              # 根布局 (Geist 字体)
│   │   ├── page.tsx                # 应用入口页
│   │   ├── globals.css             # 全局样式 (Tailwind CSS 4)
│   │   └── api/chat/stream/
│   │       └── route.ts            # API 代理 (Next.js → FastAPI rewrite)
│   ├── src/
│   │   ├── components/
│   │   │   ├── ChatArea.tsx         # 聊天区域 (消息列表 + 输入框)
│   │   │   ├── ChatInput.tsx        # 消息输入框
│   │   │   ├── MessageBubble.tsx    # 消息气泡 (Markdown 渲染)
│   │   │   ├── Sidebar.tsx          # 侧边栏 (会话管理)
│   │   │   ├── SourcesPanel.tsx     # 来源面板
│   │   │   ├── MetaBar.tsx          # 元信息栏 (延迟/Token/置信度)
│   │   │   └── StreamingToken.tsx   # 流式 Token 动画
│   │   ├── hooks/
│   │   │   └── useSSE.ts            # SSE 流式连接 Hook
│   │   ├── lib/
│   │   │   ├── types.ts             # TypeScript 类型定义
│   │   │   └── api.ts               # API 调用封装
│   │   └── providers/
│   │       └── ChatProvider.tsx      # 聊天状态管理 (React Context)
│   ├── next.config.ts               # Next.js 配置 (rewrites 代理 + standalone 输出)
│   ├── package.json                 # 依赖: Next.js 16.2, React 19.2, Tailwind CSS 4
│   └── tsconfig.json                # TypeScript 配置
├── server.py                       # FastAPI 后端入口 (SSE + REST API + 生命周期管理)
├── app.py                           # Streamlit 开发调试面板
├── config.py                        # 全局配置 (pydantic-settings, 单例)
├── requirements.txt                 # Python 依赖
├── Dockerfile.backend               # 后端 Docker 镜像 (python:3.12-slim)
├── Dockerfile.frontend              # 前端 Docker 镜像 (多阶段 node:22-alpine)
├── docker-compose.yml               # 一键部署编排 (bridge 网络 + 健康检查)
├── .dockerignore                    # Docker 忽略规则
├── .env                             # 环境变量 (LLM_API_KEY, TAVILY_API_KEY 等, git-ignored)
├── fix-and-start.ps1                # Windows 一键修复 & 启动脚本
└── README.md
```

## 🚀 快速开始

### 方式一: Docker Compose (推荐)

确保已安装 [Docker Desktop](https://www.docker.com/products/docker-desktop/) 或 Docker Engine 20.10+。

```bash
# 1. 配置 API 密钥
# 编辑 .env，填入真实的 API Key。

# ── LLM 配置 (OpenAI 兼容协议) ──
# 阿里云百炼 DashScope (通义千问):
#   LLM_API_KEY=sk-xxxxxxxxxxxxxxxxxx
#   LLM_API_BASE=https://dashscope.aliyuncs.com/compatible-mode/v1
#   LLM_MODEL=qwen3.7-plus
#
# OpenAI / 其他兼容 API:
#   LLM_API_KEY=sk-your-openai-key
#   LLM_API_BASE=https://api.openai.com/v1
#   LLM_MODEL=gpt-4o-mini
#
# ── Tavily 搜索 API ──
# TAVILY_API_KEY=tvly-your-real-key

# 2. 构建并启动所有服务
docker compose up -d --build

# 3. 查看运行状态，确认所有服务 healthy
docker compose ps
```

启动后可访问以下地址：

| 服务 | 地址 | 说明 |
|------|------|------|
| 前端界面 | http://localhost:3001 | Next.js 16 生产模式 |
| FastAPI 文档 | http://localhost:8000/docs | Swagger UI |
| 健康检查 | http://localhost:8000/api/health | `{"status":"ok"}` |
| Prometheus 指标 | http://localhost:8000/api/metrics | 文本格式 (Prometheus 抓取) |
| Prometheus UI | http://localhost:9090 | 指标查询 + 告警状态 |
| Grafana 仪表盘 | http://localhost:3002 | 登录 admin/admin → 可用性监控面板 |

**容器端口映射**：

| 服务 | 容器内 | 宿主机 | 说明 |
|------|--------|--------|------|
| `search-backend` | 8000 | 8000 | FastAPI + uvicorn，SSE 流式聊天 |
| `search-frontend` | 3000 | 3001 | Next.js 16 生产模式 (standalone) |
| `search-prometheus` | 9090 | 9090 | 指标存储与查询 |
| `search-grafana` | 3000 | 3002 | 可视化仪表盘 (admin/admin) |

**常用操作**：

```bash
# 查看日志
docker compose logs -f backend          # 后端实时日志
docker compose logs -f frontend         # 前端实时日志
docker compose logs --tail=50 backend   # 后端最近 50 行

# 重启单个服务
docker compose restart backend

# 停止所有服务
docker compose down

# 完全重建（代码修改后）
docker compose down
docker compose build --no-cache
docker compose up -d

# 进入容器调试
docker exec -it search-backend bash
docker exec -it search-frontend sh
```

**镜像说明**：

| 文件 | 基础镜像 | 用途 |
|------|---------|------|
| `Dockerfile.backend` | `python:3.12-slim` | 安装 pip 依赖 → 启动 uvicorn |
| `Dockerfile.frontend` | 多阶段 `node:22-alpine` | `npm build` → 生产 runner 启动 `next start` |
| `docker-compose.yml` | — | 4 服务编排 + bridge 网络 + 健康检查 |

> **Windows 用户**: 如遇到 Docker Desktop gRPC 问题，可直接运行：
> ```powershell
> .\fix-and-start.ps1
> ```
> 该脚本自动检测 Docker 状态、清理旧资源、禁用 BuildKit 并启动服务。

### 方式二: 本地开发

#### 1. 安装依赖

```bash
pip install -r requirements.txt
```

#### 2. 配置 API 密钥

在 `.env` 文件中填入 API Key。本项目使用 **OpenAI 兼容协议**，支持：

| LLM 提供商 | LLM_API_BASE |
|------------|-------------|
| 阿里云百炼 DashScope (通义千问) | `https://dashscope.aliyuncs.com/compatible-mode/v1` |
| OpenAI | `https://api.openai.com/v1` |
| 其他兼容代理 (如 OneAPI, LiteLLM) | 自定 |

```bash
# 必填
LLM_API_KEY=sk-xxx
TAVILY_API_KEY=tvly-xxx

# 可选 (以下为默认值)
LLM_API_BASE=https://api.openai.com/v1
LLM_MODEL=gpt-4o-mini
LLM_TEMPERATURE=0.3
TAVILY_API_BASE=https://api.tavily.com
```

#### 3. 启动后端

```bash
# FastAPI 开发服务器 (port 8000)
uvicorn server:app --reload --port 8000

# 或直接运行
python server.py
```

#### 4. 启动前端 (二选一)

**Next.js 生产前端**:

```bash
cd frontend
npm install
npm run dev
# → http://localhost:3000
```

**Streamlit 开发面板**:

```bash
streamlit run app.py
# → http://localhost:8501
```

#### 5. 运行测试

```bash
# 全部测试 (131 条)
pytest tests/ -v

# 按模块运行
pytest tests/test_core.py -v      # 单元测试 (33 条)
pytest tests/test_registry.py -v  # 工具注册中心 (31 条)
pytest tests/test_graph.py -v     # 状态图节点 (31 条)
pytest tests/test_config.py -v    # 配置加载 (18 条)
pytest tests/test_e2e.py -v       # 端到端 (18 条)

# 行为评估 (33 用例, 92 断言) + 基准测试
python -m evals.evaluate                     # 运行全部评估
python -m evals.evaluate -r html             # 生成 HTML 报告
python -m evals.evaluate -c happy_path       # 按类别运行
python -m evals.evaluate --case happy-001    # 单条用例
python -m evals.benchmark                    # 延迟 & Token 对比摘要
python -m evals.benchmark --html             # 生成 HTML 基准报告
python -m evals.benchmark --json             # 输出 JSON 格式
```

## 📡 API 端点

| 端点 | 方法 | 描述 |
|------|------|------|
| `/api/chat/stream` | POST | SSE 流式聊天 (query, session_id, model, search_depth, top_k) |
| `/api/session/{id}` | GET | 获取会话历史 |
| `/api/session/{id}` | DELETE | 清除会话 |
| `/api/config/defaults` | GET | 获取默认配置 (model, search_depth, top_k, llm_api_base) |
| `/api/health` | GET | 健康检查 (status, active_sessions) |
| `/api/metrics` | GET | Prometheus 文本格式指标 |
| `/api/metrics?format=json` | GET | JSON 格式完整统计 (含可用性 + 指标快照) |
| `/api/metrics?format=availability` | GET | JSON 格式可用性统计 (仅 SLA) |

### SSE 事件类型

```
event: progress  → {"node":"rewrite|search|generate|fallback|score","message":"..."}
event: token     → {"text":"逐token文本"}
event: sources   → {"sources":[{url,title,snippet}]}
event: done      → {confidence,latency_ms,tokens_used,is_fallback}
event: error     → {message,code}
```

## 📊 可观测性 (Prometheus + Grafana)

项目内置完整的可观测性栈，提供指标采集、可视化仪表盘和告警通知。

### 指标概览

`utils/metrics.py` 通过独立 Prometheus Registry 采集以下指标：

| 指标名 | 类型 | 标签 | 说明 |
|--------|------|------|------|
| `http_requests_total` | Counter | method, path, status_code | HTTP 请求计数 |
| `http_request_duration_seconds` | Histogram | method, path | 请求延迟分布 |
| `sse_streams_total` | Counter | status | SSE 流连接 (started/completed/disconnected) |
| `search_requests_total` | Counter | provider, status | 搜索 API 调用 (success/error/timeout) |
| `tokens_consumed_total` | Counter | type | Token 消耗 (input/output) |
| `active_sessions` | Gauge | — | 活跃会话数 |
| `search_duration_seconds` | Histogram | provider | 搜索调用延迟 |
| `llm_call_duration_seconds` | Histogram | operation | LLM 调用延迟 (rewrite/score/generate/fallback) |
| `errors_total` | Counter | type | 错误事件 (search_failure/llm_error/timeout/fallback_triggered) |
| `http_availability_ratio` | Gauge | window | 滑动窗口可用性比率 (1m/5m/15m/1h) |
| `http_sla_violation` | Gauge | window | SLA 违规标记 (1=低于 95%) |

### 滑动窗口可用性统计

`AvailabilityTracker` 按秒级粒度追踪每次请求的成功/失败状态，支持多窗口查询：

```python
from utils.metrics import get_availability_tracker

tracker = get_availability_tracker()
print(tracker.availability_ratio(300))  # 5 分钟窗口
print(tracker.meets_sla(0.95))           # True/False
print(tracker.stats)                     # {"1m": {...}, "5m": {...}, "15m": {...}, "1h": {...}}
```

### Grafana 仪表盘

Dashboard 包含 4 个区域、11 个面板：

| 区域 | 面板 | 说明 |
|------|------|------|
| **可用性概览** | 可用性 Gauge / SLA 违规 / 请求速率 | 实时 95% SLA 状态 |
| **延迟 & 性能** | HTTP P50/P95/P99 / 搜索 P50/P95 | 端到端延迟趋势 |
| **请求 & 错误** | 状态码堆叠 / 错误速率 | 异常定位 |
| **Agent 工作负载** | 活跃会话 / Token 速率 / SSE 连接 / 搜索速率 | 吞吐量监控 |

### 告警规则

`monitoring/prometheus/alerts.yml` 定义了 5 条告警：

| 告警 | 严重级别 | 触发条件 |
|------|---------|----------|
| HighErrorRate | critical | 5xx 错误率 > 5%，持续 5 分钟 |
| HighLatency | warning | P95 延迟 > 10s，持续 5 分钟 |
| ServiceDown | critical | 后端无响应，持续 1 分钟 |
| HighSearchFailureRate | warning | 搜索 API 失败率 > 10%，持续 5 分钟 |
| HighTokenConsumption | info | Token 消耗 > 100k/s，持续 10 分钟 |

## 🔧 工具注册

项目遵循**职责单一原则**注册 4 个工具：

| 工具 | 类别 | 职责 |
|------|------|------|
| `tavily_search` | 搜索 | Tavily API 实时联网搜索，使用 `get_http_client` 连接池复用 |
| `query_rewriter` | 改写 | LLM 口语化→精准查询词，支持子查询拆分 + 意图识别 |
| `relevance_scorer` | 打分 | LLM 0-1 相关性评分 (默认关闭，使用 Tavily 原始分数) |
| `fallback_answer` | 降级 | 搜索不可用时 LLM 知识回答，带 ⚠️ 标记 |

另有组合工具 `search_and_filter_pipeline()` 串联搜索→去重→(可选)打分→裁剪完整流水线。

## 🧪 测试分层

项目采用**三层金字塔**测试策略，131 条测试 + 33 条评估用例（92 断言）全部 mock 外部 API，无需联网即可运行。

### 第一层: 单元测试 (`test_core.py` + `test_config.py`) — 51 条

| 测试类 | 覆盖内容 | 条数 |
|--------|----------|------|
| `TestURLDeduplicator` | URL 规范化、去重、滑动窗口、跟踪参数 | 7 |
| `TestTokenCounting` | 中英文 Token 计数 | 4 |
| `TestContentFingerprint` | SHA-256 内容指纹 | 4 |
| `TestRateLimiter` | 滑动窗口限流 | 3 |
| `TestContextTrimming` | Top-K + Token 裁剪 | 4 |
| `TestFormatSearchContext` | 搜索结果格式化 | 2 |
| `TestSessionStore` | 会话 TTL 存储 | 5 |
| `TestModels` | Pydantic 数据模型 (SearchResult/Answer/Query) | 3 |
| `TestAPIConfig` | 环境变量加载、SecretStr 安全、超限校验 | 7 |
| `TestAgentConfig` | 默认值、可变性、实例隔离 | 3 |
| `TestSingletonFunctions` | 单例缓存、类型区分 | 3 |
| `TestConfigCompleteness` | 字段完整性、非空校验 | 3 |
| `TestProjectRoot` | 根目录存在性 | 2 |

### 第二层: 集成测试 (`test_registry.py` + `test_graph.py`) — 62 条

| 测试类 | 覆盖内容 | 条数 |
|--------|----------|------|
| `TestTavilySearch` | 搜索成功/超时/异常/自定义参数/空结果/域名过滤 | 6 |
| `TestRewriteQuery` | 正常改写/历史上下文/LLM失败回退/JSON异常/默认值填充 | 6 |
| `TestScoreRelevance` | 打分成功/空输入/LLM失败回退/零分默认/内容截断 | 5 |
| `TestFallbackAnswer` | 降级回答/历史传递/自身失败/System Prompt/历史截断 | 5 |
| `TestSearchAndFilterPipeline` | 完整流水线/去重/API错误/空搜索/全重复/打分失败回退 | 6 |
| `TestToolRegistryMetadata` | 工具注册表完整性/类别匹配/唯一性 | 3 |
| `TestNodeRewriteQuery` | 状态更新/空查询/历史传递 | 3 |
| `TestNodeSearch` | 无查询/子查询拆分/状态键完整性 | 3 |
| `TestNodeGenerateAnswer` | 降级路径/片段生成/来源去重/历史上下文/二次降级 | 5 |
| `TestNodeErrorRecovery` | 标志设置/空消息/缺键默认 | 3 |
| `TestRoutingEdgeCases` | 缺键默认/空字符串/falsy值行为 | 5 |
| `TestSearchAgentClass` | 初始化/verbose/空历史/清除会话/去重器/latency | 6 |
| `TestAgentStateTypedDict` | 必需键完整/_fragments 私有键 | 2 |
| `TestAnswerSystemPrompt` | 非空/关键指令/足够长度 | 3 |

### 第三层: 端到端测试 (`test_e2e.py`) — 18 条

| 测试类 | 覆盖路径 | 条数 |
|--------|----------|------|
| `TestAgentPipelineE2E` | ①正常流程 ②搜索失败→降级 ③答案生成失败→二次降级 ④多轮对话 ⑤历史裁剪 ⑥改写失败→回退 | 6 |
| `TestRoutingLogic` | 搜索后路由 + 生成后路由 | 4 |
| `TestNodeBehaviors` | 错误恢复 + 空查询 + 降级生成 | 3 |
| `TestSessionManagement` | 自动 session + 清除会话 | 2 |
| `TestResultValidation` | 来源去重 + 延迟记录 + 流式输出 | 3 |

### 评估类别 (7 类, 33 用例, 92 断言)

| 类别 | 用例 | 覆盖场景 |
|------|------|----------|
| `happy_path` | 6 | 正常流水线 (事实查询/对比/指南/新闻/多语言) |
| `degradation` | 5 | 降级路径 (搜索失败/答案失败/改写失败/打分失败/超时) |
| `routing` | 5 | 路由逻辑 (搜索后/生成后/缺键边界) |
| `tool_behavior` | 6 | 工具行为 (搜索/SearchResponse/改写/打分/降级/流水线) |
| `multi_turn` | 4 | 多轮对话 (历史累积/上限裁剪/清除会话) |
| `state_integrity` | 2 | 状态完整性 (AgentState 键/GeneratedAnswer 字段) |
| `edge_case` | 5 | 边缘情况 (空查询/单字符/全角/超长/中文) |

## 🛡️ 容错机制

| 故障类型 | 恢复策略 |
|----------|----------|
| 搜索 API 超时 | 异步指数退避重试 3 次 → LLM 降级回答 |
| 查询改写失败 (JSON 解析/网络异常) | 回退到原始用户输入 |
| 相关性打分失败 | 使用 Tavily 原始分数 (已默认跳过 LLM 打分) |
| LLM API 故障 | 友好错误提示 + 二次降级 |
| 配额耗尽 | 滑动窗口限流，提前拦截 |

## 📈 效果指标

- 正常期可用性: **≈100%**
- 异常期可用性: **95%+** (Grafana 实时监控)
- LLM 调用减少: 5次 → 2次 (省 60%)
- Prompt Token 节省: **95.7%** (13,397 → 572 tokens/轮)
- 首 Token 延迟: 子查询并行 + 无打分等待，预计降低 **50%+**
- 连接池复用: 消除每次 HTTP 调用的 TCP/TLS 握手开销

## 📊 业务场景

- **实时新闻**: "今天有什么重大新闻？"
- **股价查询**: "苹果公司现在的股价是多少？"
- **热点事件**: "最近发生的 AI 领域大事件有哪些？"
- **对比分析**: "GPT-4 和 Claude 哪个更强？"
- **教程指南**: "最新的 Python 3.13 有哪些新特性？"

## 📄 License

MIT
