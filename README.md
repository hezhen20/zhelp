# zhelp

zhelp 是一个面向电商场景的智能客服示例系统。它把对话、业务工具、知识库问答、人工确认流程和运营评估页面组合在一个 FastAPI 服务中，适合本地开发、功能演示和 Agent/RAG 实验。

## 主要功能

- **流式客服对话**：通过 `/api/chat` 和 `/api/agent` 提供普通对话与 Agent 对话，支持会话上下文、意图路由、指代消解和上下文摘要。
- **业务查询与处置**：查询订单、商品、物流和售后信息，创建人工工单；退款流程支持等待用户确认后继续执行。
- **知识库问答**：从 `data/kb/*.md` 构建知识库，使用向量检索与 BM25 混合检索，再经过重排；证据不足时可以拒答。
- **知识运营闭环**：知识录入、切块、向量化、检索自测、用户反馈、对话挖掘和待审核知识队列。
- **评估与观测**：RAG 策略评估、工具审计、Langfuse 可观测、按意图统计成本和置信度校准。
- **可选主题分类器**：独立的语料构建、微调、评测、ONNX 导出和批量归类流程，不参与实时客服主链路。

## 技术栈

- Python 3.12+、[uv](https://docs.astral.sh/uv/)
- FastAPI + Uvicorn
- LangChain + LangGraph
- SQLAlchemy asyncio + asyncmy + MySQL 8
- Milvus Standalone（依赖 etcd 和 MinIO）
- MCP Python SDK / FastMCP
- OpenAI 兼容的聊天和嵌入接口；重排使用独立的 HTTP `rerank` 接口
- LangGraph SQLite checkpoint
- Langfuse（可选，用于观测和成本统计）
- 主题分类器的训练/导出依赖单独的 `ml` 依赖组（PyTorch、Transformers、ONNX Runtime 等）

聊天、嵌入和重排可以分别配置不同的供应商，配置项见 `.env.example` 和 `app/config.py`。

## 快速启动

### 前置条件

- Docker Desktop 或其他可用的 Docker Compose 环境
- `uv`、`make` 和 Bash
- 可访问的聊天、嵌入、重排模型服务及对应 API Key

`scripts/dev.sh` 和 Makefile 使用 Bash 命令。Windows 建议在 WSL 或 Git Bash 中执行；原生 PowerShell 不保证兼容。

### 安装和配置

```bash
uv sync
cp .env.example .env
```

编辑 `.env`，至少填写以下配置。`.env.example` 中的 `sk-xxx` 只是占位符，不能直接使用：

```dotenv
CHAT_BASE_URL=https://api.example.com/v1
CHAT_MODEL=<供应商认可的真实模型名>
CHAT_API_KEY=<聊天模型密钥>
EMBED_API_KEY=<嵌入模型密钥>
RERANK_API_KEY=<重排模型密钥>
```

`EMBED_BASE_URL` 和 `RERANK_BASE_URL` 有默认值；如果使用其他供应商，请显式填写。模型名称必须是对应上游真实支持的名称。

### 启动依赖和准备数据

```bash
# 启动 MySQL、etcd、MinIO 和 Milvus
docker compose up -d mysql etcd minio milvus-standalone
# 等待 MySQL 和 Milvus 就绪，可用 docker compose ps 查看状态

# 灌入 FAQ 种子数据；首次创建全新 MySQL volume 时，sql/ 也会自动初始化表结构
make seed

# 构建并向量化知识库。知识库问答需要这两步
make kb-build
make kb-vectorize
```

然后启动 MCP Server 和 FastAPI 应用：

```bash
make dev
```

`make dev` 会再次确认依赖容器已启动，等待 Milvus 就绪，启动物流和售后 MCP Server（`8101`、`8102`），最后启动应用（`8000`）。应用启动后访问 <http://localhost:8000>；也可以打开 <http://localhost:8000/docs> 查看 API 文档。

停止本地服务：

```bash
make dev-down
docker compose stop mysql etcd minio milvus-standalone
```

如果只想启动应用而不先构建知识库，可以跳过 `make kb-build` 和 `make kb-vectorize`，但知识类问题不会有完整数据可检索。

## 页面和接口

| 地址 | 用途 |
| --- | --- |
| `/` | 客服聊天 |
| `/admin` | 管理入口总览 |
| `/kb` | 知识录入、切块和检索自测 |
| `/rag-eval` | RAG 评估 |
| `/review` | 待审核知识和反馈飞轮 |
| `/observability` | 观测、评估趋势和成本 |
| `/topics` | 主题分类结果分布 |
| `/acceptance` | 分类器验收总览 |
| `/api/chat` | 流式聊天接口 |
| `/api/agent` | 流式 Agent 接口 |
| `/docs` | FastAPI OpenAPI 文档 |

分类器验收还有 `/acceptance/eval`、`/acceptance/data` 和 `/acceptance/errors` 详情页。

## 常用命令

```bash
make help             # 查看所有带说明的目标
make test             # 运行单元测试
make seed-conv        # 灌入历史会话，供知识挖掘流程使用
make kb-preview       # 预览知识库切块
make kb-repatch       # 只重建有变化的知识块
make kb-reset         # 清空知识库并删除 Milvus collection
```

Langfuse 和主题分类器是可选组件：

```bash
make langfuse-up      # 启动 Langfuse（http://localhost:3000）
uv sync --group ml    # 安装分类器训练/ONNX 依赖
make ch10-corpus
make ch10-train
make ch10-eval
make ch10-export
make classifier-up    # 启动分类器推理服务（http://localhost:8110）
```

## 注意事项

1. **业务数据是演示数据**：订单、商品、物流和售后工具目前使用稳定生成的 mock 数据，不连接真实电商系统，也不代表真实订单状态。
2. **模型服务需要联网和有效凭据**：聊天、嵌入、重排可使用不同供应商；错误的模型名、地址或 API Key 会导致启动后的调用失败。
3. **启动应用不会自动构建知识库**：必须执行 `make kb-build && make kb-vectorize`，并确保嵌入服务可访问。
4. **MySQL 初始化只对新数据卷生效**：`sql/` 下的脚本只会在全新的 MySQL volume 首次启动时按文件名执行。已有 volume 不会自动执行后来新增的 DDL，需要手动应用迁移脚本。
5. **`make kb-reset` 会清空知识库**：它会删除 MySQL 中的知识块和 Milvus collection，执行前确认不需要保留现有数据。
6. **端口可能冲突**：应用使用 `8000`，MCP 使用 `8101`/`8102`，Milvus 使用 `19530`，Langfuse 使用 `3000`，分类器使用 `8110`。
7. **Langfuse 和分类器不是主服务必需项**：不启动它们不会影响基本聊天、工具调用和知识库问答；分类器训练属于独立离线流程。

## 目录概览

| 目录 | 内容 |
| --- | --- |
| `app/api/` | HTTP 接口和页面后端 |
| `app/graph/` | LangGraph 状态、节点、路由和工作流组装 |
| `app/core/` | 模型客户端、检索、重排、上下文和观测能力 |
| `app/kb/` | 知识切块、入库、向量化和知识挖掘 |
| `app/tools/` | 内置工具、MCP 客户端和工具执行引擎 |
| `app/db/` | SQLAlchemy 模型和数据访问 |
| `app/static/` | Web 页面 |
| `mcp_servers/` | 物流和售后 MCP Server |
| `scripts/` | 知识库、评估、飞轮和分类器离线脚本 |
| `sql/` | MySQL 初始化和迁移脚本 |

