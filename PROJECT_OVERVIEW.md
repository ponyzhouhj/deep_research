# DeepResearch 多智能体工程梳理

## 1. 项目定位

这是一个面向“深度研究 / 企业知识问答 / 多来源证据分析”的多智能体应用工程。整体目标是接收用户问题后，自动判断是走快速问答路径，还是走完整的研究链路；在研究链路中完成问题拆解、网络检索、本地知识库检索、证据裁判、分析反思和最终 Markdown 研报生成。

工程由 Python 后端、LangGraph 多智能体编排、DashScope/Qwen 模型调用、Milvus 本地知识库检索、记忆系统、FastAPI 服务接口和 Vue 前端组成。

## 2. 目录结构

```text
.
├── main.py                         # CLI 启动入口，加载 .env 并运行 mult_agents.main
├── app/
│   ├── app_main.py                 # FastAPI 应用入口
│   ├── backend/                    # Web API 层
│   │   ├── config/settings.py      # FastAPI 配置
│   │   ├── router/                 # health、research 路由
│   │   ├── schemas/                # Pydantic 请求/响应模型
│   │   └── service/workflow_service.py
│   └── mult_agents/                # 多智能体核心
│       ├── config.py               # AppConfig，加载 config.json 和环境变量
│       ├── graph.py                # LangGraph 工作流编排
│       ├── nodes.py                # 当前工作流实际使用的节点实现
│       ├── main.py                 # CLI 运行器、Agent 构建、记忆与 checkpointer 初始化
│       ├── prompts.py              # 各 Agent system prompt
│       ├── state.py                # ResearchState 状态定义
│       ├── tools.py                # Web 搜索、RAG 查询、工具函数
│       ├── rag/                    # Milvus + DashScope Embedding 知识库
│       └── memory/                 # 短期/长期记忆系统
├── front/agent_front/              # Vue 3 + Vite 前端
├── config.json                     # 默认运行配置
├── .env.example                    # 环境变量模板
├── pyproject.toml                  # Python 项目元信息
└── requirements.txt                # Python 依赖锁定清单
```

## 3. 技术栈

后端：

- Python 3.10+
- FastAPI、Uvicorn、Starlette
- Pydantic / pydantic-settings
- LangGraph、LangChain、LangChain Core、LangChain Community
- `langchain_community.chat_models.ChatTongyi` 调用通义千问/Qwen
- DashScope Embeddings
- Milvus / pymilvus / langchain-milvus
- PostgreSQL / psycopg
- Redis / langgraph-checkpoint-redis
- Bocha Web Search API

前端：

- Vue 3
- Vite
- TypeScript
- 原生 `fetch` + SSE 流式事件消费

数据与持久化：

- LangGraph checkpointer：优先 PostgreSQL，随后 Redis，最后内存降级
- 短期记忆：PostgreSQL、Redis 或内存
- 长期记忆：PostgreSQL、SQLite fallback、Milvus 语义检索增强
- 本地知识库：Milvus collection

## 4. 启动入口与调用方式

### CLI 入口

根目录 `main.py` 会把 `app` 加入 `sys.path`，加载根目录 `.env`，然后调用 `mult_agents.main.main()`。

`app/mult_agents/main.py` 负责：

- 解析 CLI 参数，例如 `--once-query`、`--user-id`、`--thread-id`、`--enable-memory` 等。
- 从 `config.json` / `.env` 构建 `AppConfig`。
- 初始化 `MemoryManager`。
- 初始化 RAG 系统。
- 构建多个 Agent。
- 构建 LangGraph checkpointer。
- 编译并执行工作流。

典型一次性运行方式：

```bash
python main.py --once-query "请调研企业知识库 Agent 平台市场"
```

### Web API 入口

`app/app_main.py` 创建 FastAPI 应用，注册：

- `GET /health`
- `POST /api/v1/research/run`
- `POST /api/v1/research/stream`

`/run` 返回一次性 JSON 响应，`/stream` 返回 `text/event-stream`，前端主要使用 `/stream` 来显示阶段进度。

后端服务启动方式：

```bash
cd app
uvicorn app_main:app --host 0.0.0.0 --port 8000
```

前端开发服务：

```bash
cd front/agent_front
npm run dev
```

Vite 配置中会把 `/api` 和 `/health` 代理到 `http://127.0.0.1:8000`。

## 5. 多智能体架构

当前实际编排在 `app/mult_agents/graph.py` 中，节点实现来自 `app/mult_agents/nodes.py`。

工作流节点如下：

```text
START
  -> intent
      -> direct_answer -> END
      -> plan
          -> web_search
          -> local_rag
          -> deep_dive
          -> analyze
              -> reflect -> web_search/local_rag ...
              -> write -> END
```

### Agent 角色

`app/mult_agents/main.py` 的 `AgentBundle` 定义了以下 Agent：

- `intent_router`：意图分流，判断 direct / multiagent。
- `direct_responder`：简单问题快速回答。
- `planner`：任务拆解，生成子问题、大纲、检索计划、预算。
- `scout_web`：整理网络检索证据。
- `scout_local`：整理本地 RAG 证据。
- `evidence_judge`：证据评分、去重、冲突审计。
- `analyst`：形成结论、判断证据是否充分。
- `writer`：生成最终 Markdown 研报。

这些 Agent 都通过 `langchain.agents.create_agent` 创建，模型底座是 `ChatTongyi`，prompt 统一维护在 `prompts.py`。

### 工作流设计要点

- 意图分流：`intent_node` 会先做启发式识别，再结合 LLM JSON 输出决定路径。
- 并行检索：`plan` 后同时进入 `web_search` 和 `local_rag` 两个分支。
- 证据池：Web 和 Local 结果会被转成结构化 evidence，再由 `deep_dive` 合并为 `evidence_pool`。
- 反思补搜：`analyze` 如果设置 `needs_more_research=true`，且未超过 `max_iterations`，进入 `reflect` 生成补充检索计划。
- 引用约束：`write_node` 会校验最终报告中的 source_id，只允许使用 `source_index` 中存在的引用编号，并自动追加参考资料与执行附录。

## 6. 状态模型

`ResearchState` 是 LangGraph 共享状态，包含：

- 用户与租户：`query`、`user_id`、`tenant_id`
- 记忆上下文：`memory_context`
- 规划：`plan`、`outline`、`sub_questions`、`research_questions`、`search_plan`、`budget`
- 检索：`web_evidence`、`local_evidence`、`web_search_trace`、`local_rag_trace`
- 证据与审计：`evidence_pool`、`audit_flags`、`source_index`
- 分析：`analysis`、`findings`、`claim_map`、`needs_more_research`、`missing_gaps`
- 输出：`draft`、`final`
- 迭代控制：`iteration`、`max_iterations`

这个状态设计偏“证据驱动研报生成”，不是简单聊天消息堆叠。`nodes.py` 中也刻意避免把历史 `messages` 全量传给每个节点，以控制 token 累积。

## 7. 检索与证据设计

### Web 搜索

`tools.py` 中的 `bocha_web_search_records()` 调用 Bocha Web Search API：

- 从环境变量 `BOCHA_API_KEY` 读取密钥。
- 请求 `https://api.bocha.cn/v1/web-search`。
- 返回结构化记录：`title`、`url`、`snippet`、`domain`、`published_at`。

`web_search_node` 会：

- 根据规划生成查询词。
- 每个查询默认取 4 条结果。
- 分配形如 `WEB1_1-1` 的 source_id。
- 去重、过滤空结果。
- 让 WebScout 将原始结果转成结构化 evidence。
- 记录 raw/kept/rejected trace。

### 本地 RAG

`rag/core.py` 实现了 `RAGSystem`：

- 使用 `DashScopeEmbeddings`，默认模型 `text-embedding-v1`。
- 使用 `RecursiveCharacterTextSplitter`，默认 chunk size 500，overlap 50。
- 连接 Milvus。
- `ingest_text()` / `ingest_paths()` 支持文档入库。
- `search_records()` 返回本地证据片段。

`local_rag_node` 类似 Web 搜索，会把 Milvus 命中的片段整理为 `LOC...` source_id，并进入统一证据池。

## 8. 记忆系统设计

记忆系统集中在 `app/mult_agents/memory/`。

核心类型：

- `MemoryType.SHORT_TERM`：短期对话上下文。
- `MemoryType.SEMANTIC`：事实、偏好、用户画像。
- `MemoryType.EPISODIC`：历史任务、任务结果。
- `MemoryType.PROCEDURAL`：预留的过程/行为模式记忆。

`MemoryManager` 的能力：

- 根据 `tenant_id / user_id / thread_id` 组织多租户、多用户、多会话记忆。
- 从短期历史中提取最近消息和对话摘要。
- 从长期语义/任务记忆中按查询召回相关内容。
- 构造注入 prompt 的个性化上下文。
- 在一轮对话结束后持久化用户问题与回答。
- 识别“记住、我叫、我的偏好、remember”等显式记忆触发词，提取长期偏好或事实。
- 支持 Redis / PostgreSQL / SQLite / Milvus 多后端组合，并在连接失败时降级。

`config.json` 默认启用：

- `enable_memory=true`
- `short_term_backend=postgres`
- `long_term_backend=postgres`
- `checkpointer_backend=postgres`
- `enable_milvus=true`

如果 `POSTGRES_DSN`、`REDIS_URL`、`MILVUS_HOST` 未配置，对应能力会初始化失败并降级。

## 9. FastAPI 服务层

`WorkflowService` 是 Web API 和多智能体运行时之间的封装层。

主要设计：

- 懒初始化：第一次请求时加载配置、构建记忆、构建 Agent、编译工作流。
- 单例缓存：`backend/service/__init__.py` 使用 `lru_cache(maxsize=1)` 缓存服务实例。
- 同步执行包装：实际 LangGraph 调用是同步的，API 层通过 `asyncio.to_thread()` 放到线程执行。
- SSE 流式事件：`stream_events()` 开线程执行 workflow stream，并通过 `asyncio.Queue` 把事件送回异步响应。

SSE 事件类型：

- `status`：任务接收与初始化。
- `phase`：某个节点开始/完成阶段提示。
- `route`：最终走 direct 还是 multiagent。
- `final`：最终结果。
- `error`：异常信息。

## 10. 前端设计

前端位于 `front/agent_front`，是 Vue 3 + Vite 单页应用。

主要功能：

- 类聊天窗口输入研究问题。
- 侧边栏配置 `userId`、`threadId`、`tenantId`。
- 内置起手问题。
- 调用 `/api/v1/research/stream`。
- 解析 SSE 事件，展示阶段进度。
- 收到 `final` 后把 Markdown 内容渲染为 HTML。
- 自带一个轻量 Markdown 渲染函数，支持标题、列表、代码块、粗体、斜体、行内代码和 HTTP 链接。

注意：当前前端源码中大量中文显示为乱码，推测是历史编码转换问题，不影响架构判断，但会影响实际 UI 文案质量。

## 11. 配置说明

配置来源优先级：

1. 环境变量
2. `config.json`
3. 代码默认值

关键配置：

| 配置项 | 说明 |
| --- | --- |
| `DASHSCOPE_API_KEY` / `api_key` | Qwen 和 Embedding 调用密钥 |
| `MODEL` / `model` | 对话模型，默认配置中为 `qwen-turbo` 或 `.env.example` 中的 `qwen-plus` |
| `BOCHA_API_KEY` | Web 搜索密钥，仅在环境变量中读取 |
| `MAX_ITERATIONS` | 反思补搜最大轮数 |
| `ENABLE_MEMORY` | 是否启用记忆系统 |
| `SHORT_TERM_BACKEND` | 短期记忆后端：postgres / redis / memory |
| `LONG_TERM_BACKEND` | 长期记忆后端：postgres / sqlite / disabled |
| `CHECKPOINTER_BACKEND` | LangGraph checkpoint 后端：postgres / redis / memory / auto |
| `POSTGRES_DSN` | PostgreSQL 连接串 |
| `REDIS_URL` | Redis 连接串 |
| `MILVUS_HOST` / `MILVUS_PORT` | Milvus 连接配置 |
| `MILVUS_COLLECTION` | Milvus collection 名称 |

## 12. 当前代码中的重要观察

1. 多处中文注释和前端文案存在乱码。
   - 代码逻辑仍可读，但 prompt、日志、UI 文案的可维护性受影响。
   - 如果后续要生产化，建议统一修复文件编码，确认源文件均为 UTF-8。

2. `app/mult_agents/main.py` 和 `app/mult_agents/nodes.py` 存在部分重复节点实现。
   - 当前 `graph.py` 实际引用的是 `nodes.py`。
   - `main.py` 中早期版本的 `plan_node/web_search_node/...` 仍残留，但不是当前 LangGraph 主链路。

3. `app/mult_agents/rag/ingest.py` 存在明显导入路径问题。
   - 文件中引用了 `mult_agents.src.mult_agents...`、`mult_agents_memory...` 等不符合当前目录结构的模块。
   - 同时 `INPUT_PATH` 是开发者本机绝对路径。
   - 该脚本大概率需要修复后才能作为通用入库工具使用。

4. 配置默认值倾向生产级外部依赖，但本地 `config.json` 中连接串为空。
   - PostgreSQL、Redis、Milvus、Bocha 如果未配置，部分能力会降级或返回空结果。
   - 这意味着“能启动”和“能完成高质量检索”依赖不同配置条件。

5. 前端只实现了应用内轻量 Markdown 渲染。
   - 不依赖 Markdown 库，简单可靠。
   - 但复杂表格、引用、脚注、嵌套列表等 Markdown 能力不足。

6. 工具模块里有大量 stub 工具。
   - 如 Python 执行、SQL、AMAP、文件操作、财经/新闻等，大多是模拟返回。
   - 当前核心链路主要依赖 Bocha Web Search 和 RAG。

## 13. 功能点总结

已实现或基本具备：

- 用户问题接收与 Web API 封装。
- 意图分流：简单问题快速回答，复杂问题进入研究链。
- 多 Agent 分工：规划、检索、证据裁判、分析、写作。
- Web 搜索接入 Bocha。
- 本地知识库接入 Milvus。
- 引用编号与最终报告来源约束。
- 反思补搜迭代。
- 多租户/多用户/多会话记忆。
- SSE 阶段进度推送。
- Vue 聊天式前端。

待完善或存在风险：

- 修复乱码文案和注释。
- 修复 RAG 入库脚本导入路径与参数化输入。
- 补充测试覆盖，当前只看到 Bocha API 测试文件。
- 明确部署依赖：PostgreSQL schema、Redis Stack/普通 Redis 差异、Milvus collection 管理。
- 把 stub 工具和真实工具边界写清楚，避免误以为已具备真实执行能力。
- 增强前端 Markdown 渲染和错误处理。

## 14. 一句话架构总结

该项目本质上是一个基于 LangGraph 的证据驱动型 DeepResearch 多智能体系统：FastAPI/Vue 提供交互层，LangGraph 编排 Qwen 多角色 Agent，Bocha 与 Milvus 提供外部/本地证据来源，PostgreSQL/Redis/Milvus/SQLite 组成记忆与 checkpoint 持久化体系，最终输出带引用约束的 Markdown 研究报告。
