# 工作流 Builder（Go 版）

`workflow builder` 已内置到主 `byteval` 二进制中，用于让 Agent 以小步命令构建、编辑、校验和上传 Byteval 工作流 DSL。旧版 `skills/workflow-builder` 的 Python 入口已迁移为 Go 实现，运行时代码位于 `internal/services/workflowbuilder` 并通过 `cmd/workflow.go` 挂载，`skills` 目录只保留操作说明。

## 入口

```bash
byteval workflow builder <subcommand> [flags]
```

所有子命令输出统一 JSON envelope：

```json
{ "ok": true, "data": { } }
{ "ok": false, "error": { "code": "...", "message": "...", "details": { } } }
```

## Session

除 `search` / `app-schema` 外，命令都围绕 session 工作：

```text
${BYTEVAL_WB_SESSIONS_DIR:-$HOME/tmp/byteval_wb/sessions}/<sid>/
├── ir.json
├── meta.json
├── dsl.json
└── app_cache/<app_id>.json
```

## 常用流程

```bash
SID=$(byteval workflow builder init --name "My eval" --project-id 6970 \
  | jq -r '.data.sid')

byteval workflow builder search --kind application --name "qwen" --project-id 6970
byteval workflow builder app-schema --sid "$SID" --app-id 12345 --project-id 6970
byteval workflow builder add-node --sid "$SID" --kind application --app-id 12345 --title "Qwen"
byteval workflow builder add-edge --sid "$SID" --source start_0 --target <node_id>
byteval workflow builder scope --sid "$SID" --node-id <node_id> --port prompt
byteval workflow builder bind --sid "$SID" --node-id <node_id> --port prompt \
  --mode ref --source-node start_0 --source-port agent
byteval workflow builder validate --sid "$SID"
byteval workflow builder preview --sid "$SID" --print-full
byteval workflow builder upload --sid "$SID"
```

## 子命令

| 子命令 | 用途 |
|---|---|
| `init` | 创建空 session，或通过 `--from-dsl` 导入现有 DSL |
| `search` | 搜索 application / workflow / dataset |
| `app-schema` | 获取并缓存 Application 输入输出 schema |
| `add-node` | 添加 application / dataset / workflow / start / end 节点 |
| `clone-ref` | 从已有工作流详情或本地 JSON 克隆 DSL 到 session |
| `add-edge` / `remove-edge` / `remove-node` | 编辑图连接和节点 |
| `set-position` / `auto-layout` | 设置或自动计算节点位置 |
| `scope` | 按上游连通性和 ValueType 过滤可绑定输出端口 |
| `bind` | 设置 input literal / ref / schema-ref / clear |
| `add-port` / `remove-port` / `rename-port` / `set-output-literal` | 编辑端口 |
| `set-raw` / `clear-raw` / `export-raw` / `edit-raw` | 编辑复杂节点的原始 `data.inputs` / `data.outputs` 片段 |
| `workflow-node-schema` | 根据子工作流 DSL 生成 Workflow 节点 batch inputs 和 outputList |
| `preview` | 渲染 IR 到 DSL，并写入 session `dsl.json` |
| `validate` | 校验结构错误和警告 |
| `upload` | 渲染并调用 `byteval workflow create` 创建工作流 |
| `plan apply` | 按 JSON plan 批量创建 session 草稿 |

## Agent 规则

- 构建工作流时优先使用 `byteval workflow builder ...`，不要再调用旧 Python 脚本。
- 渲染不做自动端口匹配；先 `scope` 收敛候选，再由 Agent 选择语义正确的源端口并 `bind`。
- `upload` 是唯一会写平台的 builder 子命令，执行前需要用户确认。
- `ASL` 上传时由 builder 自动序列化为字符串，不要手工把 DSL 对象嵌入请求体。
- 若需要完全自定义复杂节点结构，优先用 `set-raw` / `edit-raw`，避免手拼完整 workflow body。

## 构建说明

Go 版 builder 挂载在主 CLI 的 `workflow builder` 子命令下，随仓库根目录的 `go build .`、`make -C build build-all` 和 `make -C build release` 一起编译发布。`package-skill` 只负责打包 skill 操作说明，不需要单独编译 builder 二进制。
