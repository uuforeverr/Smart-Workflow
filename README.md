# Smart-Workflow

面向 AI Agent 的可编程工作流引擎。它将工作流的**本地增量编辑、静态校验、单节点调试、版本发布和异步执行**组织成一条可自动化闭环。

项目以 Go 实现 API、CLI、调度器和存储层，以 Python sidecar 隔离执行用户代码。`swf` CLI 的输入是明确的参数，输出是统一 JSON envelope，便于 Agent 解析错误并逐步修正工作流。

## 核心能力

- **IR / DSL 两层模型**：本地 IR 适合增量编辑，发布或执行前渲染为 DSL；支持 DSL 反向恢复为 IR。
- **Agent 友好的 CLI**：支持命令式建图、声明式 `plan-apply`、复用已有工作流、离线校验、预览、调试、上传与运行。
- **运行前静态校验**：检查图结构、端口、绑定、引用、类型、环和可达性，返回结构化 issue。
- **依赖驱动调度**：按 `edges ∪ bindings` 构建依赖，并发执行就绪节点，记录节点输入、输出、耗时、状态和重试次数。
- **多类节点**：`start`、`end`、`condition`、`http`、`code`、`application`、`dataset`，以及节点级 `batch`。
- **Python 代码执行**：sidecar 在一次性子进程中执行用户代码，提供超时、日志/输出限流和错误脱敏。
- **版本化存储**：草稿可更新，发布版本不可变；更新草稿使用乐观锁。
- **异步运行与事件流**：Redis + asynq 执行长工作流，通过 Redis Stream 提供 SSE 事件；也保留进程内执行后端。
- **能力复用**：搜索 application/workflow、读取应用 schema、计算合法绑定候选、克隆已发布工作流。

## 系统架构

```text
                       MySQL
                         ▲
                         │ metadata / DSL / run snapshots
┌─────────┐  HTTP  ┌─────┴──────┐     enqueue      ┌────────────┐
│ swf CLI ├───────►│ swf-server ├─────────────────►│ swf-worker │
└────┬────┘        │  gin API   │     Redis/asynq  │ engine.Run │
     │             └─────┬──────┘◄─────────────────└─────┬──────┘
     │ local IR           │       Redis Stream / SSE      │
     │ validate/render    │                               │ code
     ▼                    ▼                               ▼
session directory   HTTP clients                    Python sidecar
ir.json / dsl.json  and API users                  one-shot runner
```

默认配置连接 Redis，因此 `swf-server` 负责入队，`swf-worker` 负责执行。完整 Docker Compose 会同时启动 MySQL、Redis、sidecar、server、worker，并通过一次性 migrate 服务自动建表。

## 快速开始

### 环境要求

- Docker 与 Docker Compose v2：运行完整服务栈。
- Go `1.26.4`：在宿主机编译或运行 `swf` CLI，与 [`go.mod`](go.mod) 保持一致。
- `jq`、`curl`：运行仓库内的情感评测示例。
- 本地开发 sidecar 时需要 Python `3.9+`；容器镜像使用 Python 3.12。

### 1. 启动完整服务栈

```bash
git clone https://github.com/uu999/Smart-Workflow.git
cd Smart-Workflow

make up-all
docker compose -f deployments/docker-compose.yaml ps
```

服务端口：

| 服务 | 宿主机地址 | 用途 |
|---|---|---|
| server | `http://127.0.0.1:8080` | HTTP API |
| sidecar | `http://127.0.0.1:8090` | Python code 节点 |
| MySQL | `127.0.0.1:3308` | 元数据与运行快照 |
| Redis | `127.0.0.1:6381` | 缓存、队列与事件流 |

检查服务：

```bash
curl -s http://127.0.0.1:8080/healthz | jq
curl -s http://127.0.0.1:8080/healthz/sidecar | jq
```

查看日志或停止服务：

```bash
make logs-all
make down-all
```

### 2. 安装 CLI

```bash
make build-swf
export PATH="$PWD/bin:$PATH"

swf --help
```

CLI 默认访问 `http://127.0.0.1:8080`。可通过 `--server` 或环境变量覆盖：

```bash
export SWF_SERVER_URL=http://127.0.0.1:8080
export SWF_API_KEY=your-key     # 仅在服务端启用 API Key 时需要
```

### 3. 运行端到端示例

[`examples/sentiment`](examples/sentiment) 展示：

```text
dataset → Python application（batch 逐条分类）→ code 聚合准确率 → end
```

```bash
export SWF_SERVER_URL=http://127.0.0.1:8080
export SWF=swf

bash examples/sentiment/01_seed.sh
bash examples/sentiment/02_run.sh
```

脚本会依次创建 project、dataset 和 application，通过 `plan-apply` 建图，离线校验并上传草稿，最后以 `run --stream` 执行。预期终态为 `succeeded`，输出包含 `accuracy`、`total` 和逐条 `details`。

## CLI 使用

### 标准工作流

```text
能力搜索/复用
      ↓
init 或 plan-apply
      ↓
add-node / add-edge / bind
      ↓
validate → preview
      ↓
node-debug
      ↓
upload
      ↓
run --stream
```

构建类命令只修改本地 session；`validate` 和 `preview` 也在 CLI 进程内完成。能力查询、数据集管理、`node-debug`、`upload` 和 `run` 等命令才访问服务端。

本地 session 默认位于 `$HOME/tmp/swf/sessions/<sid>/`，可用 `SWF_SESSIONS_DIR` 覆盖。目录中主要包含：

```text
ir.json       # 编辑态 IR
meta.json     # 名称、项目和来源信息
dsl.json      # preview/upload 产生的执行态 DSL
app_cache/    # application schema 缓存
```

### 声明式建图

复杂工作流推荐使用一份 plan 一次性声明节点、边、绑定、参数和 batch：

```bash
swf plan-schema
swf plan-apply --file examples/sentiment/plan.json
```

`plan-apply` 是**构建期**批量建图；`batch` 是**运行期**让单个节点遍历数组输入，两者语义不同。

### 增量建图

下面构建一个最小的 `start → end` 工作流：

```bash
swf init --name hello-flow --project-id <project-id> --sid hello

swf add-node --sid hello --id start_0 --kind start \
  --outputs 'message:string:required'
swf add-node --sid hello --id end_0 --kind end \
  --inputs 'message:string:required'

swf add-edge --sid hello --source start_0 --target end_0
swf bind --sid hello --node-id end_0 --port message \
  --mode ref --source-node start_0 --source-port message

swf validate --sid hello
swf preview --sid hello --print-full
swf upload --sid hello
```

所有命令输出统一 envelope：

```json
{
  "ok": false,
  "error": {
    "code": "NODE_NOT_FOUND",
    "message": "source node \"app_1\" not found",
    "hint": "先 add-node 再连边"
  }
}
```

### 常用命令

| 命令 | 作用 |
|---|---|
| `swf search --project-id ID --kind application\|workflow` | 搜索可复用能力 |
| `swf app-schema --sid SID --app-id ID` | 获取并缓存 application 端口 schema |
| `swf scope --sid SID --node-id ID --port PORT` | 计算某输入端口的合法引用候选 |
| `swf clone-ref --workflow-id ID` | 将已发布工作流克隆为本地 IR session |
| `swf plan-schema` | 输出 plan JSON 规格 |
| `swf plan-apply --file plan.json` | 声明式创建完整本地图 |
| `swf add-node` / `add-edge` / `bind` | 增量编辑本地 IR |
| `swf remove-node` / `remove-edge` / `remove-binding` | 删除并清理本地图元素 |
| `swf validate --sid SID` | 离线静态校验 |
| `swf preview --sid SID` | 离线渲染并保存 DSL |
| `swf node-debug --sid SID --node-id ID` | 上传前无状态单节点调试 |
| `swf upload --sid SID` | 新建服务端工作流草稿 |
| `swf upload --sid SID --update-id ID --confirm --version-lock N` | 乐观锁覆盖现有草稿 |
| `swf run --workflow-id ID --version -1 --stream` | 执行草稿并订阅实时事件 |
| `swf dataset-create` / `dataset-list` / `dataset-get` | 管理评测集 |

运行版本语义：

- 省略 `--version`：运行最新发布版本。
- `--version -1`：运行当前草稿。
- `--version N`：运行指定发布版本，`N > 0`。

## HTTP API

API 前缀为 `/v1`，统一返回：

```json
{"ok": true, "data": {}}
```

或：

```json
{"ok": false, "error": {"code": "NOT_FOUND", "message": "...", "details": {}}}
```

主要端点：

| 资源 | 端点 |
|---|---|
| 健康检查 | `GET /healthz`、`GET /healthz/sidecar` |
| Projects | `POST/GET /v1/projects`、`GET/PUT/DELETE /v1/projects/:id` |
| Applications | `POST/GET /v1/applications`、`GET/PUT/DELETE /v1/applications/:id` |
| Datasets | `POST/GET /v1/datasets`、`GET/PUT/DELETE /v1/datasets/:id` |
| Workflows | `POST/GET /v1/workflows`、`GET/PUT/DELETE /v1/workflows/:id` |
| 发布与校验 | `POST /v1/workflows/:id/publish`、`POST /v1/workflows/:id/validate` |
| Node debug | `POST /v1/workflows/:id/node-debug`、`POST /v1/node-debug` |
| Runs | `POST/GET /v1/runs`、`GET /v1/runs/:id` |
| 运行事件 | `GET /v1/runs/:id/events`（SSE） |

创建项目示例：

```bash
curl -sS -X POST http://127.0.0.1:8080/v1/projects \
  -H 'Content-Type: application/json' \
  -d '{"name":"demo","description":"README example"}' | jq
```

运行工作流采用异步协议：`POST /v1/runs` 先返回 `run_id` 和 `pending`，随后通过 `GET /v1/runs/:id` 轮询，或订阅 SSE：

```bash
curl -sS -X POST http://127.0.0.1:8080/v1/runs \
  -H 'Content-Type: application/json' \
  -d '{"workflow_id":"<workflow-id>","version":-1,"input":{"message":"hello"}}' | jq
```

## 本地开发

### 启动各组件

本地运行 Go/Python 进程时，可只用 Compose 启动 MySQL 与 Redis：

```bash
make infra-up
make migrate-up                 # 需要本机安装 goose

make sidecar-install            # 首次创建 Python venv
make run-sidecar                # 终端 1，127.0.0.1:8090
make run-server                 # 终端 2，:8080
make run-worker                 # 终端 3，默认 Redis 配置下必须启动
```

`make migrate-up` 使用：

```text
swf:swfpass@tcp(127.0.0.1:3308)/smart_workflow?parseTime=true
```

Goose 可按 [官方安装方式](https://github.com/pressly/goose)安装；完整 Compose 模式不需要宿主机安装 goose。

### 配置

配置优先级为：**环境变量 > 配置文件 > 默认值**。server 和 worker 均支持 `-config <path>`；环境变量使用 `SWF_` 前缀，嵌套项以下划线连接。

| 环境变量 | 默认值 | 说明 |
|---|---|---|
| `SWF_ENV` | `dev` | 运行环境 |
| `SWF_SERVER_ADDR` | `:8080` | API 监听地址 |
| `SWF_MYSQL_DSN` | 本地 `127.0.0.1:3308` DSN | MySQL 连接串 |
| `SWF_REDIS_ADDR` | `127.0.0.1:6381` | Redis 地址；配置后 server 使用 asynq |
| `SWF_REDIS_PASSWORD` | 空 | Redis 密码 |
| `SWF_REDIS_DB` | `0` | Redis DB |
| `SWF_SIDECAR_BASEURL` | `http://127.0.0.1:8090` | Python sidecar 地址 |
| `SWF_LOG_LEVEL` | `info` | Zap 日志级别 |
| `SWF_AUTH_API_KEYS` | 空 | 允许的 API Key；空表示关闭鉴权 |
| `SWF_AUTH_RATE_RPS` | `0` | 全局每秒请求数；`<=0` 表示不限流 |
| `SWF_AUTH_RATE_BURST` | `0` | 令牌桶突发容量 |

开启应用内鉴权后，客户端需发送 `X-API-Key`。生产环境仍应由统一网关承担身份认证与授权，应用内 API Key 只作为无网关环境的基础保护。

### 构建与测试

```bash
make build
make build-swf
make test
```

集成测试需要 MySQL 已启动并完成迁移：

```bash
SWF_TEST_DSN='swf:swfpass@tcp(127.0.0.1:3308)/smart_workflow?parseTime=true' \
  go test -tags integration ./...
```

Python runner 契约与脱敏测试不依赖 FastAPI：

```bash
python3 sidecar/contract/test_contract.py
python3 sidecar/test_runner_sanitize.py
```

## 项目结构

```text
cmd/
  server/             gin API 服务
  worker/             asynq 工作流消费者
  swf/                CLI 入口
internal/
  api/                HTTP handlers、中间件、SSE
  async/              asynq 提交器与 worker
  cli/                Cobra 命令、本地 session、HTTP client
  dsl/                IR/DSL、渲染、反渲染、依赖分析
  engine/             builder、scheduler、变量池、节点执行器
  service/            项目、工作流、应用、数据集、运行服务
  storage/mysql/      sqlc 查询与存储实现
  validator/          静态校验器
sidecar/              FastAPI 门面与一次性 Python runner
migrations/           Goose 数据库迁移
deployments/          Dockerfiles 与 Docker Compose
examples/sentiment/   可执行的端到端评测示例
docs/                 按里程碑组织的实现解读
```

## 设计文档

- [M1：IR / DSL 设计与代码解读](docs/M1_IR_DSL_解读.md)
- [M2：存储层与版本发布解读](docs/M2_存储层与版本发布解读.md)
- [M3：变量池与校验器解读](docs/M3_变量池与校验器解读.md)
- [M4：执行引擎与调度器解读](docs/M4_执行引擎与调度器解读.md)
- [M5：Python sidecar 与 code 节点解读](docs/M5_Python_sidecar与code节点解读.md)
- [M6：HTTP API 与 node-debug 解读](docs/M6_HTTP_API与node-debug解读.md)
- [M7：CLI 解读](docs/M7_CLI解读.md)
- [GraphJSON / IR / BPMN 生成方案](docs/M5_GraphJSON-IR-BPMN生成方案.md)
- [Agent CLI 设计](Smart-Workflow_Agent-CLI设计.md)
- [实施计划与里程碑](plan.md)

## 安全边界与限制

- Python runner 提供的是**进程隔离、超时和输出限制**，不是真正的系统级沙箱。不要直接执行不受信任的外部代码；生产环境应增加容器级隔离、cgroup、seccomp 或 gVisor。
- code 节点输出上限为 256 KiB，日志上限为 64 KiB；大对象存储尚未实现。
- 默认发布版使用 Redis/asynq。server 与 worker 必须连接同一 Redis，否则运行会停留在 `pending`。
- 当前仓库未提供 `LICENSE` 文件，使用或分发前请先确认授权方式。
