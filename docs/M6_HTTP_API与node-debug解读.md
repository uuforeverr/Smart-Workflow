# M6 HTTP API 与 node-debug 解读

本文用于精读 Smart-Workflow 的 M6 代码，重点讲 **HTTP API** 与 **node-debug** 两部分，覆盖：

- `internal/httpx/response.go`：统一响应契约 Envelope
- `internal/api/router.go`：gin 路由装配、Deps、handlers 聚合
- `internal/api/helpers.go`：错误映射 `failFromErr` + 分页保护
- `internal/api/runs.go` + `internal/service/run.go`：run 异步模型 + version 语义 + 校验闸门
- `internal/api/dispatcher.go`：后台运行调度器 RunDispatcher
- `internal/api/nodedebug.go` + `internal/api/workflows.go` + `internal/service/nodedebug.go`：两种 node-debug
- `internal/api/nodedebug_test.go` / `e2e_integration_test.go`：M6 测试

M6 的核心目标：

```text
把前 5 个里程碑攒出来的能力，用一层 HTTP API 暴露出去：
资源 CRUD（projects/workflows/applications/datasets）+ 校验 + 发布 +
运行（提交即返回 runID，异步执行，轮询查状态）+ 单节点调试。
让 curl / CLI / Agent 能只靠 HTTP 走完 create → validate → run → node-debug 全链路。
```

先给结论：

```text
M6 是"门面层"，本身几乎不含业务逻辑——业务都在前面的 service/engine 里。
M6 干三件事：把 HTTP 请求翻译成 service 调用、把 service 错误翻译成 HTTP 状态码、
把"同步的 HTTP 请求"和"异步的工作流执行"用 dispatcher 解耦。
两个重点：run 的异步模型（提交/轮询），node-debug 的两种形态（有状态/无状态）。
```

---

## 1. 先回答几个最容易混淆的问题

### 1.1 为什么 `POST /v1/runs` 不等运行完就返回？

因为工作流可能跑很久（HTTP 节点、code 节点、batch……），同步等会导致请求超时、连接占用。M6 用的是**提交即返回**模型：

```text
POST /v1/runs
  1. 校验 + 落一条 workflow_run(status=pending)
  2. 交给后台 dispatcher 去调 engine.Run
  3. 立即返回 {run_id, status: "pending"}    <- 不等执行完

GET /v1/runs/{id}
  轮询查状态，直到 succeeded / failed
```

看 `createRun`（`runs.go`）：

```go
runID, err := h.runs.CreateRun(...)      // 落 pending
if !h.runner.Submit(runID) { ... 503 }   // 交后台执行
httpx.OK(c, gin.H{"run_id": runID, "status": "pending"})  // 立即返回
```

这正好接上 M4 的设计——M4 讲过 `engine.Run(ctx, runID)` 是纯函数入口，只吃 runID。M6 就是那个"先落 pending、再把 runID 交出去"的调用方。plan.md 写得很直白：

```text
run（提交即返回 runID + GET /runs/{id} 轮询）
  —— M6 阶段可先在 handler 内直接调 engine.Run 跑完再落库，
     M9-a 再换成入队；因引擎已是 Run(ctx,runID) 纯函数，切换不改核心
```

### 1.2 后台执行用的是"裸 goroutine"吗？请求结束了，执行会被取消吗？

不是裸 goroutine，是 `RunDispatcher`；执行**不会**随 HTTP 请求取消。

这是 M6 反思后的一个重要加固（硬伤1 / TD-8）。如果直接 `go engine.Run(c.Request.Context(), runID)`，会有三个致命问题：

```text
1. c.Request.Context() 在 HTTP 响应返回后就被取消 —— 工作流刚提交就被掐断
2. 裸 goroutine panic 会打崩整个 server 进程（gin.Recovery 护不到自起的 goroutine）
3. 无并发上限 —— 批量提交会 OOM / 打爆 sidecar
```

`RunDispatcher` 用三招解决（`dispatcher.go`）：

```text
1. 独立 baseCtx：不随单个 HTTP 请求取消（用 context.WithCancel(context.Background())）
2. panic 兜底：每条运行 goroutine 内 recover，崩溃只让该 run 落 failed
3. semaphore 限并发：满载时 Submit 返回 false，handler 把 run 改判 failed + 回 503
```

所以 `engine.Run` 拿到的是 dispatcher 的 `baseCtx`，不是请求 ctx：

```go
if err := d.engine.Run(d.baseCtx, runID); err != nil { ... }
```

请求早就返回了，工作流还在后台用独立 context 跑着。

### 1.3 node-debug 为什么有两个端点？它们有什么区别？

因为有两种使用场景：**已保存的工作流**调试，和**还没保存**的本地图调试。

```text
POST /v1/workflows/:id/node-debug   有状态：按 workflowID + nodeID 从库里定位节点
POST /v1/node-debug                 无状态：直接吃一个渲染好的 DSL 节点，不碰库
```

区别在"节点从哪来"：

```text
有状态（workflows.go: nodeDebugWorkflow）：
  wf = 从库读工作流草稿
  node = 在 wf.Draft 里按 nodeID 找
  —— 适合"已 upload 的工作流，调其中某个节点"

无状态（nodedebug.go: debugNode）：
  node = 请求体直接带过来的 dsl.DSLNode
  —— 适合 CLI"本地 validate/preview → node-debug → upload"闭环
     还没 upload，就想先调试单个节点
```

两者最终都汇到同一个引擎能力——M4 讲的 `engine.DebugNode`（关重试 + 3 断言）。无状态端点的设计注释点破了动机（`nodedebug.go`）：

```text
支撑 CLI "本地 validate/preview → node-debug → upload" 闭环。
```

即：**upload 之前就能验证单节点**，不必先把半成品存进库。这契合你项目里"读写分离、编辑过程零代价可回滚"的规范。

### 1.4 node-debug 和正式 run 的执行有什么不同？

三个关键区别，M4 讲 `DebugNode` 时提过，这里从 API 视角再对齐一次：

```text
                正式 run                    node-debug
执行范围        整张图                       单个节点
输入来源        变量池上游解析（ResolveInputs） 调用方直接喂 inputs
重试            按 RetryConfig               强制关闭（拿即时真实结果）
落库            node_run / workflow_run      不落库
额外产出        无                           3 条断言（status/output_not_empty/cost）
```

为什么 debug 要直接喂 inputs、不做上游解析？因为调试的是"这一个节点行不行"，不想被上游是否跑通干扰。这就是"隔离"调试——把节点从图里单独拎出来测。

### 1.5 API 的错误码是怎么定的？为什么有的是 404 有的是 422？

靠一个集中的映射函数 `failFromErr`，把 service 层的**语义错误**翻译成 **HTTP 状态码 + 业务错误码**。

看 `helpers.go`：

```go
func failFromErr(c *gin.Context, err error) {
    var verr *service.ValidationError
    if errors.As(err, &verr) {         // 校验失败 -> 422 + issues
        httpx.Fail(c, 422, "VALIDATION_FAILED", ..., gin.H{"issues": verr.Issues})
        return
    }
    switch {
    case errors.Is(err, service.ErrNotFound):     // -> 404
    case errors.Is(err, service.ErrVersionLock):  // -> 409
    case errors.Is(err, service.ErrInvalidVersion): // -> 400
    case errors.Is(err, service.ErrInvalidJSON):  // -> 400
    default:                                        // -> 500
    }
}
```

映射表：

```text
ErrNotFound / ErrNodeNotFound  -> 404 NOT_FOUND
ErrVersionLock                 -> 409 VERSION_CONFLICT （M2 的乐观锁冲突）
ErrInvalidVersion              -> 400 INVALID_VERSION
ErrInvalidJSON                 -> 400 INVALID_JSON
ValidationError                -> 422 VALIDATION_FAILED（带 issues）
其它                            -> 500 INTERNAL
```

为什么校验失败是 422 而不是 400？因为 400（Bad Request）语义是"请求本身格式坏"，而 422（Unprocessable Entity）语义是"请求格式没问题，但内容语义不合法"——工作流图能解析，只是图有 error 级问题。这个区分让 Agent 能精确判断"是我请求发错了，还是我的图画错了"。

### 1.6 `version` 字段省略、传 -1、传 0 分别是什么意思？

这是 M6 反思②（TD-9）专门消歧的一个点。`createRunReq.Version` 用的是 **`*int32` 指针**，就是为了区分"省略"和"显式传 0"：

```go
type createRunReq struct {
    Version *int32 `json:"version"` // 省略=最新发布 / -1=草稿 / N>0=指定版本
    ...
}
```

规则（`run.go` 的 `resolveVersion`）：

```text
nil（JSON 里没这个字段）  -> 取 workflow.published_ver（未发布则报错）
-1                        -> 草稿调试（对齐 engine.DraftVersion）
N > 0                     -> 指定已发布版本
0 或 < -1                 -> ErrInvalidVersion（400，不再静默当"最新发布"）
```

为什么非要用指针？因为 Go 里 `int32` 的零值就是 0，如果用值类型，"用户没传"和"用户显式传了 0"无法区分——都会是 0。用 `*int32`，省略时是 `nil`，显式传 0 时是指向 0 的指针，两者能分开。这呼应了 M1 讲 `resolvedMode` 时同样的"如何表达缺省"心智。

---

## 2. M6 的整体架构

一次 API 请求的分层：

```text
HTTP 请求（curl / CLI / Agent）
   |
   v
gin router（router.go）                     路由 + 中间件（日志/鉴权/限流）
   |
   v
handler（runs.go / workflows.go / ...）      解析请求体、校验必填、调 service
   |
   v
service（service/*.go）                      业务逻辑（M2~M5 已建）
   |
   v
engine / storage                             执行 / 落库
   |
   v
httpx.Envelope（response.go）                统一响应体，回给客户端
```

run 的异步分叉：

```text
createRun handler
   |  CreateRun 落 pending
   |  runner.Submit(runID)
   v
RunDispatcher（dispatcher.go）              独立 ctx + panic 兜底 + semaphore
   |  go engine.Run(baseCtx, runID)
   v
[后台] M4 引擎执行 -> 落 node_run/workflow_run 终态

（HTTP 请求此时早已返回 {run_id, status:pending}）
```

核心文件与职责：

```text
httpx/response.go   统一 Envelope：{ok, data} / {ok, error:{code,message,details}}
api/router.go       路由表 + Deps 依赖注入 + handlers 聚合各 service
api/helpers.go      failFromErr 错误映射 + pageParams 分页保护
api/runs.go         run handler：提交即返回 + 满载 503
api/dispatcher.go   后台运行：独立 ctx / panic recover / 限并发 / 优雅退出
api/workflows.go    workflow CRUD + publish + validate + node-debug（有状态）
api/nodedebug.go    node-debug（无状态，吃 DSL 节点）
service/run.go      CreateRun：version 语义 + TD-10 校验闸门 + RunView
service/nodedebug.go NodeDebugService：定位节点 -> engine.DebugNode
```

一句话概括分层：

```text
router 管"哪条路径走哪个 handler"。
handler 管"HTTP 进出 + 参数校验"，几乎不写业务。
service 管业务，engine 管执行，httpx 管响应格式。
```

---

## 3. 统一响应契约 Envelope

源码：`internal/httpx/response.go`

整个平台（Go API、CLI、Python sidecar）用同一套响应体，这是 M6 API 的"外观标准"。

```go
type Envelope struct {
    OK    bool   `json:"ok"`
    Data  any    `json:"data,omitempty"`
    Error *Error `json:"error,omitempty"`
}
type Error struct {
    Code    string `json:"code"`     // 机器判断
    Message string `json:"message"`  // 人读
    Details any    `json:"details,omitempty"`  // hint / issues / options
}
```

两个出口函数：

```go
func OK(c *gin.Context, data any)                       // {ok:true, data}
func Fail(c *gin.Context, httpStatus int, code, message string, details any)  // {ok:false, error}
```

设计意图和 M3 的 `validator.Issue` 一脉相承：

```text
Code    给 Agent/程序判断"是哪类错"，可分类自动修复
Message 给人看
Details 放结构化补充信息（如校验失败的 issues 清单）
```

M5 讲过 sidecar 也用这套 envelope——所以从 Python 子进程到 Go API，错误码风格是统一的。一个请求失败，客户端拿到的永远是 `{ok:false, error:{code, message, details}}`，不用猜格式。

---

## 4. 路由装配：router.go

源码：`internal/api/router.go`

### 4.1 Deps：依赖注入入口

```go
type Deps struct {
    Cfg    *config.Config
    Logger *zap.Logger
    Store  *mysql.Store
    Engine *engine.Engine
    Runner RunSubmitter        // 后台运行提交器
    EventSource eventbus.Source // SSE 事件源（M9-b）
}
```

`main` 组装好这些依赖，交给 `NewRouter`。这种显式依赖注入让测试很好写——可以只传 `Cfg + Logger` 复用探针路由，也可以传 mock。

### 4.2 RunSubmitter：为 M9 预留的接口

```go
type RunSubmitter interface {
    Submit(runID string) bool
    Shutdown(ctx context.Context) error
}
```

这个接口是 M6 → M9 平滑演进的关键。`createRun` handler 只依赖这个接口，不关心背后是进程内 dispatcher 还是 asynq 队列：

```text
M6：RunDispatcher（进程内 goroutine + semaphore）实现 RunSubmitter
M9：asynq 入队器实现 RunSubmitter
   —— createRun handler 一行不用改，只换注入的实现
```

这正是 M4 那句"避免 M9 加队列时重构调度入口"在 API 层的兑现。

### 4.3 路由表

M6 挂载的资源路由（`register`）：

```text
/v1/projects       POST/GET/GET:id/PUT:id/DELETE:id
/v1/applications   POST/GET/GET:id/PUT:id/DELETE:id
/v1/datasets       POST/GET/GET:id/PUT:id/DELETE:id
/v1/workflows      POST/GET/GET:id/PUT:id/DELETE:id
                   + POST:id/publish
                   + POST:id/validate
                   + POST:id/node-debug        <- 有状态 node-debug
/v1/runs           POST/GET/GET:id
                   + GET:id/events             <- SSE（M9-b）
/v1/node-debug     POST                        <- 无状态 node-debug
```

两个健康探针挂在 `/v1` 之外（不受鉴权/限流影响）：

```text
GET /healthz          存活探针
GET /healthz/sidecar  验证 Go 能否调通 Python sidecar 的 /healthz（M5 联通性）
```

`/healthz/sidecar` 很实用——它把 M5 的"Go↔sidecar 联通性"做成了一个可 curl 的探针，部署时一眼看出 sidecar 通不通。

### 4.4 一个务实的装配细节

```go
if d.Store != nil {
    runner := d.Runner
    if runner == nil {
        runner = NewRunDispatcher(d.Engine, d.Logger, 0)  // 默认进程内 dispatcher
    }
    h := &handlers{...}
    h.register(v1)
}
```

只在 `store != nil` 时挂资源路由。这让单测能只传 `Cfg + Logger` 起一个"只有探针"的 router，不需要真库。`Runner` 为 nil 时兜底自建 dispatcher——调用方（main）通常自己建并持有引用，以便优雅退出时 `Shutdown`。

---

## 5. run 异步模型（重点）

这是 M6 两大重点之一，跨 `runs.go`（handler）、`run.go`（service）、`dispatcher.go`（后台）三个文件。

### 5.1 createRun handler：提交即返回

`runs.go` 的 `createRun` 流程：

```text
1. 解析请求体，校验 workflow_id 必填
2. CreateRun 落 pending 记录（含 version 归一 + 校验闸门）
3. runner.Submit(runID)：
   - 成功 -> 返回 {run_id, status:pending}
   - 满载/关停 -> FailRunWithError 改判 failed + 回 503 RUN_REJECTED
```

满载处理很讲究——不是直接丢弃，而是把已落的 pending **改判 failed**，避免"僵尸 pending"：

```go
if !h.runner.Submit(runID) {
    h.engine.FailRunWithError(ctx, runID, "run rejected: dispatcher at capacity or shutting down")
    httpx.Fail(c, http.StatusServiceUnavailable, "RUN_REJECTED", "server is at capacity, please retry later", gin.H{"run_id": runID})
    return
}
```

这用到了 M4 讲的 `FailRunWithError`（墓碑方法）——保证任何异常路径下 run 都有终态，不会永久卡在 pending。

### 5.2 CreateRun service：version 语义 + 校验闸门

`run.go` 的 `CreateRun` 做两件"运行前"的关键事：

**第一件：version 归一**（`resolveVersion`，已在 1.6 讲）

**第二件：TD-10 校验闸门**——运行前先静态校验，坏图直接拒绝：

```go
if verr := s.validateForRun(ctx, workflowID, resolved); verr != nil {
    return "", verr   // 坏图不落 pending、不进 builder/scheduler
}
```

`validateForRun` 加载目标版本的 DSL，跑 M3 的 validator：

```go
res := validateDSL(d, dsl.Meta{})   // M3: DSL -> ToIR -> Validate
if res.HasError() {
    return &ValidationError{Issues: res.Issues}  // 带上全部 issues
}
```

这个闸门的价值在于：**把 M3 的校验能力真正接进执行主路径**。M4 讲义第 10 节曾把"无 run 前 validation gate"列为待补项——M6 的 TD-10 就是补上它。效果：

```text
没有闸门：坏图 -> 落 pending -> 进 builder -> 报错 -> 落 failed
          用户只拿到一句裸错误字符串，不知道错在哪
有闸门：  坏图 -> 直接 422 VALIDATION_FAILED + issues 清单
          Agent 拿到"哪个节点哪个字段错"，可自动修复
```

`ValidationError` 是自定义错误类型，携带结构化 issues：

```go
type ValidationError struct {
    Issues []validator.Issue
}
```

`failFromErr` 用 `errors.As` 识别它，映射成 422 + `details.issues`（见 1.5）。

### 5.3 RunDispatcher：后台执行的三重加固

源码：`internal/api/dispatcher.go`

已在 1.2 讲过设计动机，这里看实现要点。

**Submit：占坑 + 背压**

```go
func (d *RunDispatcher) Submit(runID string) bool {
    d.mu.Lock()
    if d.closed { d.mu.Unlock(); return false }  // 已关停，拒绝
    select {
    case d.sem <- struct{}{}:   // 占一个并发坑
        d.wg.Add(1)
        d.mu.Unlock()
    default:
        d.mu.Unlock()
        return false            // semaphore 满，背压拒绝
    }
    go func() { ... }()
    return true
}
```

关键：占坑在**持锁期间**尝试，避免与 Shutdown 竞争后仍加 wg。semaphore 用带缓冲 channel 实现，满了走 `default` 分支直接拒绝——这是"背压"（拒绝而非无限堆积）。

**goroutine：panic 兜底**

```go
go func() {
    defer d.wg.Done()
    defer func() { <-d.sem }()   // 释放并发坑
    defer func() {
        if r := recover(); r != nil {
            // panic 后用独立 ctx 落墓碑
            d.engine.FailRunWithError(context.Background(), runID, fmt.Sprintf("run panicked: %v", r))
        }
    }()
    d.engine.Run(d.baseCtx, runID)
}()
```

三个 defer 的顺序很重要（LIFO）：先 recover 兜 panic，再释放 semaphore 坑，最后 wg.Done。panic 后用 `context.Background()` 而非可能已污染的 baseCtx 落墓碑。

**Shutdown：优雅退出**

```go
func (d *RunDispatcher) Shutdown(ctx context.Context) error {
    d.closed = true
    d.cancel()          // 取消 baseCtx，通知在途 run 收尾
    // wg.Wait() 等所有在途 goroutine 排空，或 ctx 到期
}
```

停机时：停收新 run → 取消在途 run 的 ctx → 等它们排空（或超时）。这保证 server 关闭时不会硬切在途工作流。

### 5.4 GetRun：轮询视图

`run.go` 的 `GetRun` 返回 `RunView`——包含整次运行状态 + 所有节点快照：

```go
type RunView struct {
    RunID, WorkflowID, Status, TriggerType string
    Version int32
    Input, Output json.RawMessage
    Error string
    Nodes []NodeRunView   // 每个节点的 input/output/status/cost/attempt
}
```

它读的正是 M4 落库的 `workflow_run` + `node_run`。客户端轮询 `GET /runs/{id}`，从 `status` 字段判断跑完没有，从 `Nodes` 看每个节点的执行快照——这就是 M2 建 `node_run` 表、M4 写快照的最终消费出口。

---

## 6. node-debug（重点）

M6 第二大重点。两种形态，共用同一个引擎能力。

### 6.1 引擎能力回顾（M4 debug.go）

M4 讲过 `engine.DebugNode`：

```text
1. 直接喂 inputs，不做变量池上游解析（隔离）
2. 强制关重试（拿即时真实结果）
3. 用临时空 varpool，RunID="debug"，不落库
4. 附带 3 条断言：status_success / output_not_empty / cost_under_sec
```

M6 的两个端点都是给它套不同的"节点从哪来"的皮。

### 6.2 有状态 node-debug（workflows.go + service/nodedebug.go）

路由：`POST /v1/workflows/:id/node-debug`

handler（`workflows.go`）：

```go
func (h *handlers) nodeDebugWorkflow(c *gin.Context) {
    var req nodeDebugReq  // {node_id, inputs, cost_target_sec}
    ...
    result, err := h.nodeDebug.DebugNode(ctx, c.Param("id"), req.NodeID, req.Inputs, req.CostTargetSec)
    ...
}
```

service（`service/nodedebug.go`）：

```go
func (s *NodeDebugService) DebugNode(ctx, workflowID, nodeID, inputs, costTargetSec) (*engine.DebugResult, error) {
    wf, err := s.wf.Get(ctx, workflowID)          // 从库读工作流
    node, ok := findNode(wf.Draft, nodeID)        // 在草稿里按 ID 找节点
    if !ok { return nil, ErrNodeNotFound }        // 找不到 -> 404
    return s.engine.DebugNode(ctx, node, inputs, costTargetSec), nil
}
```

链路：`workflowID → 读草稿 → 定位节点 → engine.DebugNode`。适合"已保存工作流，调其中一个节点"。找不到节点返回 `ErrNodeNotFound`，`failFromErr` 映射成 404。

### 6.3 无状态 node-debug（nodedebug.go）

路由：`POST /v1/node-debug`（注意：不在 workflows 组下，不需要 workflowID）

```go
type statelessNodeDebugReq struct {
    Node          dsl.DSLNode    `json:"node"`     // 直接带一个渲染好的 DSL 节点
    Inputs        map[string]any `json:"inputs"`
    CostTargetSec float64        `json:"cost_target_sec"`
}

func (h *handlers) debugNode(c *gin.Context) {
    var req statelessNodeDebugReq
    ...
    if req.Node.ID == "" { failBadRequest(c, "node.id is required"); return }
    if req.Node.Data.NodeMeta.NodeType == "" { failBadRequest(...); return }
    result := h.engine.DebugNode(c.Request.Context(), req.Node, req.Inputs, req.CostTargetSec)
    httpx.OK(c, result)
}
```

链路：`请求体带 DSL 节点 → 直接 engine.DebugNode`。全程不碰库。

这个端点是 CLI 闭环的关键（`nodedebug.go` 注释）：

```text
CLI 在本地把 IR 渲染成 DSL，取出要调的那个 DSL 节点直接提交，
不需要先 upload 成 workflow 再按 id 调试——支撑"upload 前先验证"闭环。
sidecar 访问仍集中在服务端，CLI 保持薄。
```

两句话点破了它的设计哲学：

```text
1. upload 前就能调 —— 编辑态零代价验证（呼应你项目的"读写分离"规范）
2. CLI 保持薄 —— 打 sidecar 这种脏活留在服务端，CLI 只管发请求
```

### 6.4 两种 node-debug 对比

```text
                有状态                          无状态
路由            /workflows/:id/node-debug       /node-debug
节点来源        库里工作流草稿 + nodeID          请求体直接带 DSL 节点
依赖存储        是（先 Get 工作流）              否
典型调用方      调已保存工作流的某节点            CLI upload 前的本地调试
找不到节点      404 ErrNodeNotFound             400（node.id 必填校验）
共用            都汇到 engine.DebugNode（关重试 + 3 断言）
```

---

## 7. 分页保护 pageParams

一个小但重要的防御细节（`helpers.go`）：

```go
func pageParams(c *gin.Context) (limit, offset int32) {
    limit, offset = 20, 0           // 默认 limit=20
    if v := c.Query("limit"); v != "" {
        if n, err := strconv.Atoi(v); err == nil && n > 0 {
            limit = int32(n)
        }
    }
    if limit > 200 { limit = 200 }  // 硬上限 200，防止一次拉爆
    ...
}
```

所有列表接口（listProjects/listWorkflows/listRuns…）都走它：

```text
默认 limit=20，offset=0
limit 上限 200（防止 ?limit=999999 一次性拖垮 DB / 内存）
非法值静默回退默认（不报错，保证列表接口鲁棒）
```

这和 M5 的"超时双向夹取"是同一种防御思路：**不信任外部传入的分页参数**。

---

## 8. M6 测试怎么读？

M6 测试分"纯逻辑单测"和"E2E 集成"两层。

### 8.1 node-debug 单测（mock sidecar，纯内存）

源码：`internal/api/nodedebug_test.go`

```text
TestNodeDebug_Stateless     无状态端点：吃 code DSL 节点 + mock sidecar，
                             验证 status=succeeded、output.doubled=42、3 条断言
TestNodeDebug_MissingFields  缺 node.id / nodeType -> 400
TestFailFromErr_ValidationError  ValidationError -> 422 + details.issues
```

`TestNodeDebug_Stateless` 很典型——`engine.New(nil, sidecar.URL)` 的 store 传 nil，因为 DebugNode 只用 registry 不触库，所以无状态调试能在无数据库环境测。这印证了 6.3 的"全程不碰库"。

### 8.2 错误映射单测

`TestFailFromErr_ValidationError` 守卫 TD-10 闸门的错误映射：`*service.ValidationError` → 422 + `details.issues`。这是把 1.5 的映射规则固化成回归测试。

测试命令：

```bash
go test ./internal/api/ -run 'TestNodeDebug|TestFailFromErr' -v
```

### 8.3 E2E 集成测试（真 MySQL + mock sidecar）

源码：`internal/api/e2e_integration_test.go`

```go
//go:build integration
func TestM6_E2E(...)
```

它通过 HTTP 走完整链路（对应 plan.md 的 M6 验收）：

```text
create → validate → publish → run → 轮询 succeeded → node-debug
```

断言：

```text
1. 每步响应都是统一 envelope
2. node_run 落库（run 跑完后 GET /runs/{id} 能看到节点快照）
3. 404 / 400 错误码映射正确
```

前置（注释里写明，和你 memory 里的本地环境一致）：

```bash
SWF_TEST_DSN='swf:swfpass@tcp(127.0.0.1:3308)/smart_workflow?parseTime=true' \
go test -tags integration ./internal/api/ -run TestM6_E2E -v
```

这个 E2E 是 M6 的验收核心——它证明"curl 能只靠 HTTP 完成全链路"这个目标真的达成了。

---

## 9. 当前 M6 的设计边界和后续增强

当前已经完成：

```text
1. 四类资源 CRUD（projects/workflows/applications/datasets）全路由
2. 统一 Envelope + failFromErr 错误码映射（404/409/400/422/500）
3. run 异步模型：提交即返回 runID + GET 轮询 + node_run 快照
4. RunDispatcher：独立 ctx + panic 兜底 + semaphore 限并发 + 优雅退出（TD-8）
5. version 语义消歧：*int32 区分省略/-1/N/非法（TD-9）
6. TD-10 校验闸门：run 前静态校验，坏图 422 + issues
7. node-debug 两形态：有状态（workflowID+nodeID）/ 无状态（DSL 节点）
8. RunSubmitter 接口：为 M9 换 asynq 预留，handler 不改
9. 分页保护 + sidecar 联通探针
10. E2E 全链路验收（真 MySQL + mock sidecar）
```

当前还可以增强（部分记入技术债）：

```text
1. TD-11 handler 强耦合：handler 直接 new service，未做接口抽象，
   单测只能靠真 store 或整体 E2E，难对 handler 单独 mock。
2. TD-12 列表/大对象/断言契约债：
   - 列表无游标分页（offset 深翻页性能差）
   - node_run output 大对象直出，无裁剪/对象存储引用（M2/M5 同类债）
   - node-debug 断言只有固定 3 条，未支持自定义断言表达式
3. run 提交无幂等键：网络重试可能重复创建 run（可加 idempotency-key）。
4. 轮询模型的实时性：GET 轮询有延迟，M9-b 的 SSE（/runs/:id/events）
   已埋点但需 EventSource 才生效。
5. 鉴权/限流为 M10 加固项，默认 no-op（空 APIKeys 放行、rps<=0 不限流）。
```

其中最值得关注的是 **TD-11**（handler 可测性）和 **SSE 实时性**——前者影响测试成本，后者影响"轮询"这种略笨重的交互体验。

---

## 10. 一句话总结 M6

```text
M6 是把前 5 个里程碑用 HTTP 暴露出去的门面层：统一 Envelope（{ok,data}/{ok,error}）+
failFromErr 把 service 语义错误映射成 404/409/400/422/500；run 走"提交即返回 runID +
轮询"的异步模型——CreateRun 先 version 归一、再 TD-10 校验闸门（坏图 422+issues）、落
pending，交给 RunDispatcher 用独立 ctx + panic 兜底 + semaphore 限并发在后台调
engine.Run（请求早已返回），满载改判 failed 回 503；node-debug 分有状态（按 workflowID+
nodeID 从库定位）与无状态（直接吃渲染好的 DSL 节点、支撑 CLI upload 前调试）两形态，都汇到
engine.DebugNode（关重试 + 3 断言）；RunSubmitter 接口让 M9 换 asynq 时 handler 零改动。
```
