# M7 CLI 解读

本文用于精读 Smart-Workflow 的 M7 代码，重点讲 **CLI 的用法**与**实现逻辑**，覆盖：

- `cmd/swf/main.go`：命令行入口
- `internal/cli/root.go`：根命令与命令树装配
- `internal/cli/session.go`：IR/DSL 两层会话目录读写
- `internal/cli/output.go`：统一 JSON envelope
- `internal/cli/client.go`：调服务端的薄客户端
- `internal/cli/build_cmds.go`：增量构建命令（init/add-node/add-edge/bind/remove-*）
- `internal/cli/verify_cmds.go`：验证与发布命令（validate/preview/node-debug/run/upload）
- `internal/cli/helpers.go` / `poll.go`：会话加载、渲染、轮询
- `internal/cli/commands_test.go`：CLI 全链路测试

M7 的核心目标：

```text
把前 6 个里程碑的能力包装成一个面向 Agent 的命令行 swf：
用增量小步命令把工作流"搭"出来（改本地 IR），进程内零成本 validate/preview，
真跑类命令（node-debug/run/upload）才转发服务端。
让 Agent 仅靠 CLI 就能完成：建图 → bind → validate → node-debug → upload → run。
```

先给结论：

```text
M7 是这套系统的"卖点"（plan.md 标 ★）。
它不是给人敲的交互式 CLI，而是给 Agent 调的"结构化命令集"：
每条命令吃 flag、吐 JSON envelope，Agent 能直接解析结果和错误，逐步自我纠错。
两条主线：本地 IR 编辑（离线、零成本、可回滚）+ 服务端真跑（node-debug/run/upload）。
```

---

## 1. 先回答几个最容易混淆的问题

### 1.1 CLI 编辑的是 IR 还是 DSL？改在哪里？

编辑的是 **IR**，改在**本地会话目录**的 `ir.json`，完全不碰服务端和数据库。

M1 讲过 IR 是"编辑态"、DSL 是"执行态"。M7 的所有构建命令（add-node/bind/…）都只改本地 IR：

```text
swf add-node / add-edge / bind / remove-*
   |
   v
读 $SWF_SESSIONS_DIR/<sid>/ir.json -> 改 IR 结构 -> 原子写回 ir.json
```

只有到了 `preview`/`upload`/`node-debug` 才会把 IR 渲染成 DSL（M1 的 render）。这就是 `session.go` 开头那段注释说的分层：

```text
validate/preview 在 CLI 进程内跑（零成本离线），
node-debug/run 才碰服务端；"upload 前先修到 0 error" 得以成立。
```

这契合你项目里"读写分离，除 upload 外的 builder 子命令仅影响本地 session，编辑过程零代价可回滚"的规范——改错了直接删会话目录重来，没有任何服务端副作用。

### 1.2 为什么 CLI 输出的是 JSON，而不是给人看的表格？

因为 **swf 的主要使用者是 Agent，不是人**。

所有命令统一输出 `Envelope`（`output.go`），和服务端 `httpx.Envelope` 同形：

```text
成功: { "ok": true,  "data": {...} }
失败: { "ok": false, "error": { "code": "...", "message": "...", "hint": "..." } }
```

关键是 `error` 里比服务端多了个 `hint` 字段——面向 Agent 的**下一步建议**：

```go
type CLIErr struct {
    Code    string `json:"code"`
    Message string `json:"message"`
    Hint    string `json:"hint,omitempty"`  // 如"先 swf add-node"
    Details any    `json:"details,omitempty"`
}
```

举个实际的例子（`build_cmds.go` 的 add-edge）：

```go
return a.emitErr(newErr("NODE_NOT_FOUND",
    fmt.Sprintf("source node %q not found", source),
    "先 add-node 再连边"))   // <- hint 直接告诉 Agent 怎么修
```

于是 Agent 拿到的不是"报错了"，而是"错误码 NODE_NOT_FOUND + 修复建议先 add-node"——可以据此自动纠错。这是 M7 作为"Agent CLI"的灵魂：**错误也是结构化、可执行的**。

### 1.3 会话（session）是什么？存在哪？

会话是"一次工作流构建过程"的本地工作目录，默认在 `$HOME/tmp/swf/sessions/<sid>/`。

`session.go` 定义了目录结构：

```text
<sid>/
├── ir.json      编辑态 IR（所有 build 命令改这个）
├── meta.json    会话元信息（name/project_id/source/时间戳）
├── dsl.json     渲染出的 DSL（preview/upload 时落）
└── app_cache/   app schema 缓存（M8 用）
```

根目录可通过 `SWF_SESSIONS_DIR` 环境变量覆盖（测试就靠这个隔离）：

```go
func SessionsRoot() string {
    if v := os.Getenv("SWF_SESSIONS_DIR"); v != "" {
        return v
    }
    home, _ := os.UserHomeDir()
    return filepath.Join(home, "tmp", "swf", "sessions")
}
```

一个 `sid` 对应一个工作流的构建过程。`swf init` 建会话拿 `sid`，后续所有命令都靠 `--sid` 找到它。

### 1.4 为什么写文件要"原子写"？

防止半截文件。`session.go` 的 `writeJSONAtomic` 是先写 `.tmp` 再 `rename`：

```go
func writeJSONAtomic(path string, v any) error {
    b, _ := json.MarshalIndent(v, "", "  ")
    tmp := path + ".tmp"
    os.WriteFile(tmp, b, 0o644)   // 先写临时文件
    os.Rename(tmp, path)          // 再原子重命名
}
```

为什么重要？因为每条 build 命令都是"读 ir.json → 改 → 写回"。如果写到一半进程被杀（或 Agent 并发调用），直接覆盖原文件会留下损坏的半截 JSON，整个会话就废了。`rename` 在同一文件系统上是原子操作，要么完整替换、要么不变，永远不会读到半截文件。

这和你 memory 里记的"编辑过程零代价可回滚"是配套的——每一步都完整落盘，随时可中断、可回退。

### 1.5 哪些命令碰服务端，哪些不碰？

一条清晰的分界线：**改 IR 的不碰，真跑的才碰**。

```text
纯本地（进程内，离线，零成本）：
  init          建会话
  add-node      加节点
  add-edge      加边
  bind          绑定端口
  remove-node/edge/binding  删除
  validate      静态校验（进程内跑 M3 validator）
  preview       渲染 IR→DSL（进程内跑 M1 render）

转发服务端（需 swf-server 在跑）：
  node-debug    单节点调试（服务端复用引擎 + sidecar）
  run           整图运行（服务端异步）
  upload        上传草稿（服务端落库）
```

为什么这么分？`client.go` 注释点破了：

```text
真跑类命令走它，把 sidecar/DB 访问留在服务端，
CLI 自身保持无状态、可离线做 IR 编辑。
```

即 **CLI 保持薄**：打 sidecar、连数据库这种"重"活留在服务端，CLI 只管编辑 IR 和转发请求。好处是 CLI 二进制轻量、无状态、能在任何地方离线搭图，只有真要执行时才需要服务端。

### 1.6 `validate` 在哪里跑？和服务端的 validate 是一套吗？

`swf validate` 在 **CLI 进程内**跑，直接调 M3 的 `validator.Validate(s.IR)`，不发任何网络请求。

```go
// verify_cmds.go
res := validator.Validate(s.IR)   // 直接吃会话里的 IR，进程内
```

对比 M6 的服务端 validate（`POST /workflows/:id/validate`）：

```text
CLI validate：    吃本地 session 的 IR，进程内，零成本、秒级、离线
服务端 validate：  吃库里的 DSL，先 ToIR 再 Validate（M6 讲过）
```

两者复用**同一个** `validator` 包，只是"IR 从哪来"不同。CLI 的 IR 本来就在手边，直接校验；服务端的是 DSL，要先反编译。这就是为什么 M3 讲义强调"validator 吃 IR"——CLI 场景是它最直接的消费者。

好处：Agent 可以在 upload 之前，进程内反复 `validate` 修到 0 error，完全不占用服务端资源。这是 M7 "upload 前先修到 0 error" 闭环成立的技术前提。

---

## 2. M7 的整体架构

CLI 的分层：

```text
cmd/swf/main.go                入口：NewRootCmd(os.Stdout).Execute()
   |
   v
root.go（appCtx + cobra 命令树）  全局 flag（--server/--api-key）+ 挂载子命令
   |
   +-- build_cmds.go   本地 IR 编辑：init/add-node/add-edge/bind/remove-*
   +-- verify_cmds.go  验证发布：validate/preview/node-debug/run/upload
   +-- catalog/plan/clone/dataset 命令（M8/M10 扩展）
   |
   v
session.go（会话读写）  ir.json/meta.json/dsl.json/app_cache，原子写
output.go（envelope）   统一 {ok,data}/{ok,error} 输出
client.go（HTTP 客户端） 真跑类命令转发服务端，解析 envelope
```

两条数据流：

```text
【本地编辑流】
  命令 → mustLoad(sid) 读 ir.json → 改 IR → s.Save() 原子写回

【服务端转发流】
  命令 → renderSession 渲染 IR→DSL → client.doJSON(POST/PUT ...) → 解析 envelope
```

核心文件与职责：

```text
root.go       命令树装配 + appCtx（out/serverURL/apiKey）+ emitOK/emitErr
session.go    会话目录的 CRUD + 原子写
output.go     Envelope + CLIErr（带 hint）+ writeOK/writeErr
client.go     doJSON：发请求、解 envelope、服务端错误转 *CLIErr
build_cmds.go 6 个增量构建命令，只改本地 IR
verify_cmds.go 5 个验证发布命令，validate/preview 本地，其余转发
helpers.go    mustLoad / genSID / renderSession
poll.go       run --wait 的轮询逻辑
```

一句话概括：

```text
build_cmds 管"搭图"（改 IR），verify_cmds 管"验证和上线"（渲染 + 转发）。
session 管落盘，output 管输出格式，client 管和服务端说话。
```

---

## 3. 命令入口与装配：main.go + root.go

### 3.1 入口极简

`cmd/swf/main.go` 只有一件事——构造根命令、执行、据错误设退出码：

```go
func main() {
    root := cli.NewRootCmd(os.Stdout)
    if err := root.Execute(); err != nil {
        os.Exit(1)   // envelope 已由命令自己打印，这里只设非零退出码
    }
}
```

注意退出码的设计：结构化 envelope 已经由命令内部 `emitErr` 打印了，`main` 这里只据 error 设 `os.Exit(1)`，供脚本/CI 判断成败。**打印和退出码分离**——Agent 读 envelope，脚本读退出码。

### 3.2 appCtx：可注入的运行上下文

`root.go` 的 `appCtx` 是命令间共享的上下文：

```go
type appCtx struct {
    out       io.Writer  // 输出流（生产 os.Stdout，测试注入 buffer）
    serverURL string     // --server
    apiKey    string     // --api-key
}
```

用**依赖注入**而非包级全局，关键好处是**可测**：测试可以把 `out` 换成 `bytes.Buffer` 来捕获 envelope（第 8 节的 `runCLI` 就靠这个）。

全局 flag 有环境变量兜底：

```go
defaultServer := os.Getenv("SWF_SERVER_URL")
if defaultServer == "" {
    defaultServer = "http://127.0.0.1:8080"
}
root.PersistentFlags().StringVar(&app.serverURL, "server", defaultServer, ...)
root.PersistentFlags().StringVar(&app.apiKey, "api-key", os.Getenv("SWF_API_KEY"), ...)
```

`--server` 缺省读 `SWF_SERVER_URL`，`--api-key` 缺省读 `SWF_API_KEY`（服务端未开鉴权时留空）。

### 3.3 命令树

`NewRootCmd` 挂载了全部子命令，按功能分组：

```text
构建（build_cmds.go）：init/add-node/add-edge/bind/remove-node/remove-edge/remove-binding
验证发布（verify_cmds.go）：validate/preview/node-debug/run/upload
能力发现（M8）：search/app-schema/scope
复用（M8/M10）：clone-ref/plan-apply/plan-schema
数据集（M10）：dataset-create/dataset-list/dataset-get
```

### 3.4 emitOK / emitErr：统一出口

所有命令通过这两个方法输出：

```go
func (a *appCtx) emitOK(data any) error  { return writeOK(a.out, data) }
func (a *appCtx) emitErr(err error) error {
    _ = writeErr(a.out, err)
    return err   // 返回原始 error 让 Execute 据此设非零退出码
}
```

`emitErr` 有个细节：它既打印 envelope，又返回 error。打印是给 Agent 看的，返回 error 是给 `main` 设退出码用的。一举两得。

---

## 4. 会话管理：session.go（实现核心）

会话是 CLI 本地状态的载体，第 1.3/1.4 已讲结构和原子写，这里补生命周期。

### 4.1 三个生命周期方法

```text
NewSession(sid, meta)  建会话：MkdirAll + 空 IR 骨架 + 落盘
LoadSession(sid)       读会话：读 meta.json + ir.json，反序列化成 Session
Save()                 存会话：原子写 meta.json + ir.json
```

`NewSession` 会拒绝重复（防误覆盖）：

```go
if _, err := os.Stat(dir); err == nil {
    return nil, fmt.Errorf("session %q already exists", sid)
}
```

### 4.2 meta 与 IR.Meta 的同步

`Save` 时有个一致性维护——把会话 `Meta` 同步进 `IR.Meta`：

```go
func (s *Session) Save() error {
    s.IR.Meta.Name = s.Meta.Name
    s.IR.Meta.ProjectID = s.Meta.ProjectID
    s.IR.Meta.Source = s.Meta.Source
    ...
}
```

为什么？因为 M1 的 IR 自带 `Meta`（name/project_id/source），而 CLI 又单独存了一份 `meta.json`（多了时间戳）。两者要保持同步，upload 时才能把正确的 name/project 传给服务端。

### 4.3 mustLoad：命令的统一入口守卫

`helpers.go` 的 `mustLoad` 是几乎所有命令的第一行：

```go
func mustLoad(a *appCtx, sid string) (*Session, error) {
    if sid == "" {
        return nil, a.emitErr(newErr("BAD_REQUEST", "--sid is required", "先 swf init 拿到 sid"))
    }
    s, err := LoadSession(sid)
    if err != nil {
        return nil, a.emitErr(newErr("SESSION_NOT_FOUND", err.Error(), "检查 sid，或 swf init 新建"))
    }
    return s, nil
}
```

它把"sid 校验 + 加载 + 错误 envelope"三件事收敛成一行调用。命令里只需 `s, err := mustLoad(a, sid); if err != nil { return err }`——错误 envelope 已经打印好了。

---

## 5. 增量构建命令：build_cmds.go（用法 + 实现）

这是 M7 的"搭图"能力。每个命令都遵循同一个模式：**mustLoad → 校验参数 → 改 IR → Save → emitOK**。

### 5.1 init：建会话

```bash
swf init --name 情感评测 --project-id 6970 [--from-dsl file.json] [--sid s_xxx]
```

实现要点：

```text
1. --name 必填，缺 sid 则 genSID() 自动生成
2. NewSession 建目录 + 空 IR
3. --from-dsl：把现有 DSL 反渲染成 IR 导入（复用优先的兜底入口）
```

`--from-dsl` 用到了 M1 的 `ToIR`（DSL→IR 反编译）：

```go
ir, terr := dsl.ToIR(&d, dsl.Meta{Name: name, ProjectID: projectID, Source: fromDSL})
s.IR = ir
```

这是"复用优先"策略的一个入口——不从零搭，而是导入一个现成 DSL 再改。

### 5.2 add-node：加节点（含端口声明）

```bash
swf add-node --sid s_xxx --id app_1 --kind application \
  --app-id 12345 --title 情感分类器 \
  --inputs "text:string:required" --outputs "label:string,confidence:number"
```

实现要点：

```text
1. --id/--kind 必填，重复 ID 报 NODE_EXISTS
2. --inputs/--outputs 用 parsePorts 解析 "name:type[:required]" 逗号分隔
3. 组装 dsl.Node 追加进 IR.Nodes
```

端口声明格式 `name:type[:required]`（`parsePorts`）：

```text
"text:string:required"          -> Port{Name:text, Type:string, Required:true}
"label:string,confidence:number" -> 两个 Port
```

这就是 M3 讲的"端口声明"的录入入口——有了它，validator 才能做类型校验、连线校验。M1 的"改③补端口声明"在 CLI 层的落点就是这个 `--inputs/--outputs`。

### 5.3 bind：绑定输入端口

```bash
# ref 模式：引用上游输出
swf bind --sid s_xxx --node-id app_1 --port text --mode ref --source-node start_0 --source-port query

# literal 模式：写死常量
swf bind --sid s_xxx --node-id app_1 --port temperature --mode literal --value 0.7
```

实现要点（对齐 M1 的 Binding 语义）：

```text
1. mode 缺省推断：有 --source-node 判 ref，否则 literal（和 M1 resolvedMode 一致）
2. literal：--value 先尝试按 JSON 解析，失败则当原始字符串
3. 同端口重复绑定则覆盖（增量编辑友好）
```

literal 的 JSON 解析很贴心：

```go
if json.Unmarshal([]byte(value), &v) != nil {
    v = value   // 不是合法 JSON 就当字符串
}
```

所以 `--value 0.7` 解析成数字 0.7，`--value hello` 解析失败就当字符串 "hello"。Agent 不用纠结类型。

"重复绑定覆盖"是增量编辑的关键——Agent 改主意时直接重新 bind 同一个 port 即可，不用先 remove：

```go
for i := range node.Bindings {
    if node.Bindings[i].Port == port {
        node.Bindings[i] = b   // 覆盖
        replaced = true
        break
    }
}
```

### 5.4 add-edge：加控制流边

```bash
swf add-edge --sid s_xxx --source start_0 --target app_1
# condition 分支边：
swf add-edge --sid s_xxx --source cond_0 --target end_pass --source-port 0
```

实现要点：

```text
1. --source/--target 必填，且两端节点都必须已存在（否则 NODE_NOT_FOUND + hint"先 add-node"）
2. --source-port 用于 condition 分支序号（M1/M4 讲的 SourceHandle）
```

### 5.5 remove-* ：删除（带级联清理）

```bash
swf remove-node --sid s_xxx --id app_1        # 删节点 + 清理关联边 + 清理对它的 ref 绑定
swf remove-edge --sid s_xxx --source a --target b
swf remove-binding --sid s_xxx --node-id app_1 --port text
```

`remove-node` 最讲究——删一个节点要**级联清理三样东西**：

```text
1. 节点本身
2. 所有 source/target 是它的边
3. 所有 source_node 指向它的 ref 绑定
```

```go
// 清理指向被删节点的 ref 绑定
for _, b := range n.Bindings {
    if b.SourceNode == id { continue }  // 跳过（即删除）
    kept = append(kept, b)
}
// 清理关联边
for _, e := range s.IR.Edges {
    if e.Source == id || e.Target == id { continue }
    edges = append(edges, e)
}
```

为什么要级联？因为如果只删节点、留下悬空的边和绑定，下次 validate 就会报一堆 `bad_edge`/`ref_node_not_found`。级联清理保证删完后 IR 仍然自洽。这是"增量编辑友好"的体现。

---

## 6. 验证与发布命令：verify_cmds.go（用法 + 实现）

这是 M7 的"验证和上线"能力，分本地（validate/preview）和转发（node-debug/run/upload）两类。

### 6.1 validate：进程内静态校验

```bash
swf validate --sid s_xxx
```

实现（第 1.6 已讲）：直接 `validator.Validate(s.IR)`，输出 error/warn 计数和 issues 清单：

```go
res := validator.Validate(s.IR)
errs, warns := countSeverity(res.Issues)
return a.emitOK(map[string]any{
    "has_error": res.HasError(), "error_num": errs, "warn_num": warns, "issues": res.Issues,
})
```

Agent 读 `has_error` 判断能否继续，读 `issues` 逐条修复。

### 6.2 preview：渲染 IR→DSL

```bash
swf preview --sid s_xxx [--print-full]
```

实现要点：

```text
1. renderSession 跑 M1 的 render，把 IR 渲染成 DSL
2. 落 dsl.json
3. 默认打印节点摘要（id/type/alias），--print-full 打印完整 DSL
```

preview 是"渲染出来看看对不对"的离线预演，也顺带把 dsl.json 落盘供审计。渲染失败会提示"先 swf validate 定位问题"——因为 render 不做完整校验（M1 讲过），坏结构应该先被 validate 抓出来。

### 6.3 node-debug：单节点调试（转发服务端）

```bash
swf node-debug --sid s_xxx --node-id app_1 --inputs '{"text":"这家店不错"}' --cost-target-sec 5
```

实现要点（衔接 M6 的无状态 node-debug）：

```text
1. renderSession 渲染出 DSL
2. findRenderedNode 用 IR 可读 ID 定位渲染后的 DSL 节点
3. 转发服务端 POST /v1/node-debug（M6 的无状态端点）
```

这里有个巧妙的 ID 对齐问题。M1 讲过 render 会把可读 ID（`app_1`）变成执行态 ID（`application::xxx`）。用户用可读 ID 指定要调哪个节点，CLI 得找到渲染后的对应节点。`findRenderedNode` 利用"render 保持节点顺序"这一性质，按下标对齐：

```go
// Render 保持节点顺序，故按 IR.Nodes 的下标对齐 d.Nodes
for i, n := range ir.Nodes {
    if n.ID == readableID { idx = i; break }
}
return d.Nodes[idx], true
```

然后把这个 DSL 节点连同 inputs 发给 M6 的 `/v1/node-debug`——正好对上 M6 讲的"无状态端点吃渲染好的 DSL 节点，支撑 CLI upload 前调试"。**M6 那个端点就是为这里设计的。**

### 6.4 run：整图运行（转发服务端）

```bash
swf run --workflow-id wf_xxx --input '{"query":"hi"}' [--version -1] [--wait|--stream]
```

实现要点：

```text
1. 转发服务端 POST /v1/runs（M6 的异步 run）
2. 默认：只提交，返回 {run_id, status:pending}
3. --wait：pollRun 轮询到终态
4. --stream：订阅 SSE 实时看每步（不可用自动回退轮询）
```

三种模式对应不同的交互需求：

```text
不加 flag：拿了 run_id 就走（Agent 稍后自己轮询）
--wait：   CLI 帮你轮询到 succeeded/failed（poll.go，30s 超时，300ms 间隔）
--stream： 实时事件流（M9-b SSE）
```

`--version` 用了 cobra 的 `Changed` 来区分"省略"和"显式传值"——对应 M6 讲的 version 语义消歧：

```go
if c.Flags().Changed("version") {
    versionSet = true
}
if versionSet {
    body["version"] = version   // 只在用户显式传了才放进请求
}
```

不显式传就不放 `version` 字段，服务端按"最新发布版本"处理；这和 M6 的 `*int32` 指针语义严丝合缝。

### 6.5 upload：上传草稿（转发服务端）

```bash
# 默认：创建新副本
swf upload --sid s_xxx --description "情感评测流"

# 覆盖更新现有工作流（需 --confirm）
swf upload --sid s_xxx --update-id wf_xxx --confirm --version-lock 3
```

实现要点（对齐你 memory 里的 upload 规范）：

```text
1. renderSession 渲染 IR→DSL
2. 默认新副本：POST /v1/workflows（创建）
3. --update-id 覆盖更新：必须带 --confirm，否则报 CONFIRM_REQUIRED
4. 覆盖走 PUT /v1/workflows/:id，带 version_lock（M2 的乐观锁）
```

`--confirm` 门禁是安全设计——覆盖是破坏性操作，默认不允许：

```go
if updateID != "" {
    if !confirm {
        return a.emitErr(newErr("CONFIRM_REQUIRED",
            fmt.Sprintf("overwriting workflow %q needs --confirm", updateID),
            "确认覆盖请加 --confirm；否则去掉 --update-id 走新副本"))
    }
    ...
}
```

覆盖更新带的 `--version-lock` 正是 M2 讲的乐观锁——如果这期间别人改过草稿，version_lock 对不上，服务端返回 409 VERSION_CONFLICT。这把 M2 的并发保护一路贯通到了 CLI。

新副本成功后还会回填 `source`，便于后续迭代：

```go
s.Meta.Source = resp.WorkflowID   // 记住上传到了哪个 workflow
```

---

## 7. 服务端客户端：client.go

真跑类命令都通过 `Client.doJSON` 和服务端说话。它的核心价值是**把服务端错误也归一成 CLIErr**，让 Agent 拿到同构反馈。

```go
func (c *Client) doJSON(method, path string, body any, out any) error {
    // ... 发请求 ...
    resp, err := c.http.Do(req)
    if err != nil {
        return newErr("SERVER_UNREACHABLE", err.Error(),
            "is swf-server running? check --server / SWF_SERVER_URL")
    }
    // 解析服务端 envelope
    if !env.OK {
        ce := &CLIErr{Code: env.Error.Code, Message: env.Error.Message}
        // 保留服务端 details（如 validate 的 issues）
        if len(env.Error.Details) > 0 { ... ce.Details = d }
        return ce
    }
    // 成功：把 data 解到 out
}
```

三个设计点：

```text
1. 连不上服务端 -> SERVER_UNREACHABLE + hint（"是不是没启动 server"）
2. 服务端 ok=false -> 转成 *CLIErr，保留服务端的 code/message/details
3. details 透传 -> 服务端 validate 的 issues 能原样传回给 Agent
```

第 2 点是关键：无论错误来自 CLI 本地（如 NODE_NOT_FOUND）还是服务端（如 VALIDATION_FAILED），Agent 拿到的都是同一个 `{code, message, hint/details}` 结构。**Agent 不用区分错误来源，一套解析逻辑通吃。** 这是 M6 和 M7 共用 envelope 契约的最大红利。

---

## 8. M7 测试怎么读？

源码：`internal/cli/commands_test.go`

M7 测试最漂亮的地方是——**几乎全在进程内、离线跑通**，不需要服务端。

### 8.1 测试骨架 runCLI

```go
func runCLI(t *testing.T, args ...string) (Envelope, error) {
    var buf bytes.Buffer
    root := NewRootCmd(&buf)   // out 注入 buffer
    root.SetArgs(args)
    err := root.Execute()
    var env Envelope
    json.Unmarshal(buf.Bytes(), &env)   // 解析捕获的 envelope
    return env, err
}
```

靠两个注入点实现完全隔离：

```text
out 注入 bytes.Buffer     -> 捕获 envelope 断言
SWF_SESSIONS_DIR = t.TempDir() -> 会话写到临时目录，测试互不干扰
```

这正是 3.2 讲的"appCtx 依赖注入"的回报——`out` 可替换，测试才能捕获输出。

### 8.2 核心测试：全离线构建闭环

`TestCLI_FullOfflineBuildLoop` 验证 M7 验收的最小闭环：

```text
init → add-node ×3 → add-edge ×2 → bind ×3 → validate(0 error) → preview
```

全程不碰服务端——这证明了 M7 的核心主张：**Agent 能纯离线把图搭出来、校验通过，只在最后 upload/run 时才需要服务端**。

测试命令：

```bash
go test ./internal/cli/ -v
```

CLI 单测不依赖 MySQL / sidecar / server，纯进程内。转发类命令（run/upload）的测试用 `httptest` mock 服务端（`client_test.go`）。

---

## 9. 一次完整的 Agent 工作流（把 M1~M7 串起来）

用一个情感评测流走一遍，看 CLI 如何把前 6 个里程碑串成一条命令链：

```bash
# 1. 建会话（本地）
swf init --name 情感评测 --project-id 6970
# -> {ok:true, data:{sid:"s_ab12", ...}}

# 2. 增量搭图（本地改 IR，M1）
swf add-node --sid s_ab12 --id start_0 --kind start --outputs "query:string:required"
swf add-node --sid s_ab12 --id app_1 --kind application --app-id 12345 \
  --inputs "text:string:required" --outputs "label:string"
swf add-node --sid s_ab12 --id end_0 --kind end --inputs "out:string"

# 3. 连边 + 绑定（本地改 IR，M1）
swf add-edge --sid s_ab12 --source start_0 --target app_1
swf add-edge --sid s_ab12 --source app_1 --target end_0
swf bind --sid s_ab12 --node-id app_1 --port text --source-node start_0 --source-port query
swf bind --sid s_ab12 --node-id end_0 --port out --source-node app_1 --source-port label

# 4. 静态校验（本地，M3）—— 修到 0 error
swf validate --sid s_ab12
# -> {ok:true, data:{has_error:false, error_num:0, ...}}

# 5. 单节点调试（转发服务端，M4 引擎 + M5 sidecar + M6 端点）
swf node-debug --sid s_ab12 --node-id app_1 --inputs '{"text":"这家店不错"}'
# -> {ok:true, data:{status:"succeeded", assertions:[...]}}

# 6. 上传（转发服务端，M2 落库）
swf upload --sid s_ab12 --description "情感评测流"
# -> {ok:true, data:{workflow_id:"wf_xyz", created:true}}

# 7. 运行（转发服务端，M6 异步 + M4 调度）
swf run --workflow-id wf_xyz --input '{"query":"这家店不错"}' --wait
# -> {ok:true, data:{status:"succeeded", output:{...}, nodes:[...]}}
```

每一步都对应一个里程碑：

```text
搭图/绑定  -> M1（IR/DSL/render）
validate  -> M3（validator）
node-debug -> M4（DebugNode）+ M5（sidecar）+ M6（/node-debug 端点）
upload    -> M2（存储）+ M6（/workflows）
run       -> M6（异步 run）+ M4（引擎调度）
```

M7 把它们全部收敛成一条 Agent 可以逐步执行、逐步纠错的命令链。这就是"卖点"的含义。

---

## 10. 当前 M7 的设计边界和后续增强

当前已经完成：

```text
1. 增量构建：init/add-node/add-edge/bind/remove-node/remove-edge/remove-binding
2. 验证发布：validate/preview/node-debug/run/upload
3. session 目录：ir.json/meta.json/dsl.json/app_cache + 原子写
4. 统一 envelope + 带 hint 的结构化错误
5. 本地/服务端分层：编辑离线零成本，真跑才转发
6. upload 新副本默认 + 覆盖更新需 --confirm + version_lock 乐观锁
7. run 三模式：提交/--wait 轮询/--stream SSE
8. --from-dsl 导入 + clone-ref/plan-apply（复用优先，M8/M10）
9. 全离线构建闭环测试（不依赖服务端）
```

当前还可以增强：

```text
1. findRenderedNode 靠"render 保持顺序"按下标对齐 IR↔DSL 节点，
   较脆——若渲染逻辑改动顺序会错位，可改用稳定映射（如 render 返回 idMap）。
2. 无 session gc：会话目录只增不删，长期会堆积（可加 swf session prune）。
3. bind literal 的类型未在 CLI 侧校验（对应 M3 讲义盲区②，literal 类型漏检）。
4. 无 session lock：同一 sid 被并发命令编辑时，原子写只保证单次不损坏，
   但两条命令交错可能后写覆盖前写（Agent 串行调用时不成问题）。
5. run --wait 轮询固定 30s/300ms，长流会超时；可暴露为 flag。
```

其中 findRenderedNode 的顺序依赖最值得关注——它是 CLI↔render 之间一个隐式契约，render 一旦改动节点顺序就会静默错位。

---

## 11. 一句话总结 M7

```text
M7 是面向 Agent 的 swf CLI（★卖点）：所有构建命令（init/add-node/add-edge/bind/remove-*）
只改本地会话目录里的 ir.json（原子写、离线、零代价可回滚），validate/preview 在进程内直接跑
M3 validator 与 M1 render（零成本、秒级），只有 node-debug/run/upload 才转发服务端（把
sidecar/DB 留在服务端、CLI 保持薄）；每条命令吃 flag、吐统一 envelope（{ok,data}/{ok,error}
带面向 Agent 的 hint），服务端错误也被 client 归一成同构 CLIErr——于是 Agent 能只靠 CLI
完成"建图→bind→validate 修到 0 error→node-debug→upload→run"全闭环，逐步执行、逐步自纠错。
```
