# 工作流 Builder 命令参考（Go 版）

`workflow builder` 已内置到主 `byteval` 二进制中，用于让 Agent 以小步命令构建、编辑、校验和上传 Byteval 工作流 DSL。旧版 `skills/workflow-builder` 的 Python 入口已迁移为 Go 实现，运行时代码位于 `internal/services/workflowbuilder` 并通过 `cmd/workflow.go` 挂载，`skills` 目录只保留操作说明。

自然语言需求澄清、workflow 复用、ASCII Brief、胶水节点和上传草稿策略见 `interaction-guide.md`。

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
byteval workflow builder upload --sid "$SID" --description "My eval workflow" \
  --manager alice@bytedance.com
```

## 子命令

| 子命令 | 用途 |
|---|---|
| `init` | 创建空 session，或通过 `--from-dsl` 导入现有 DSL |
| `search` | 搜索 application / workflow / dataset |
| `app-schema` | 获取并缓存 Application 输入输出 schema |
| `add-node` | 添加 application / dataset / workflow / start / end / condition 节点 |
| `clone-ref` | 从已有工作流详情或本地 JSON 克隆 DSL 到 session |
| `add-edge` / `remove-edge` / `remove-node` | 编辑图连接和节点，`add-edge --source-port/--target-port` 用于 Condition 等多端口节点 |
| `set-position` / `auto-layout` | 设置或自动计算节点位置 |
| `scope` | 按上游连通性和 ValueType 过滤可绑定输出端口 |
| `bind` | 设置 input literal / ref / schema-ref / clear |
| `add-port` / `remove-port` / `rename-port` / `set-output-literal` | 编辑端口 |
| `set-raw` / `clear-raw` / `export-raw` / `edit-raw` | 编辑复杂节点的原始 `data.inputs` / `data.outputs` 片段 |
| `node-batch enable` / `disable` / `inspect` | 在任意节点上启用/关闭/查看 ASL `data.inputs.batch` 批处理；对 Application、Workflow、Code 等都通用 |
| `workflow-batch apply-schema` | Workflow 节点专属：根据子工作流 DSL 生成 batch inputs 和 outputList（替代旧 `workflow-node-schema`） |
| `workflow-batch disable` / `inspect` | （Deprecated 别名）等价于 `node-batch disable/inspect`，仅作用于 Workflow 节点 |
| `condition add-branch` / `remove-branch` / `list-branches` | 编辑 Condition 节点的分支列表 |
| `condition set-expression` / `remove-expression` | 编辑某条分支内的表达式（支持 literal / ref 两种右值） |
| `preview` | 渲染 IR 到 DSL，并写入 session `dsl.json` |
| `validate` | 校验结构错误和警告 |
| `upload` | 渲染并调用 `byteval workflow create` 创建工作流，或用 `--update-id` 更新现有工作流；平台要求 `Description` 与 `Managers` 非空，session 缺少 manager 时默认使用当前登录用户 email |
| `plan apply` | 按 JSON plan 批量创建 session 草稿，节点可声明 `bindings`，边可声明 `source_port`/`target_port` |
| `plan schema` | 打印各 profile 接受的 plan.json 节点规格（含 `node_bindings` 说明），便于 Agent 自检字段拼写 |

## Plan Schema

`plan apply` 接受的 plan.json 由 `plan schema` 暴露真实约束，节点解析与端口绑定均经由各 profile 路由：

- `workflow`：基本元数据，可选 `name` / `description` / `project_id` / `start_template` / `bare_end`。
- `nodes[]`：每个节点至少给出 `id` 和 `kind`（`start` / `end` / `application` / `dataset` / `workflow` / `condition` / `comment` / `fallback`）；
  - `application` 必须带 `app_id`，可选 `inputs` / `outputs` / `retry_limit` / `retry_interval`；
  - `dataset` 必须带 `dataset_id`；
  - `workflow` 必须带 `workflow_id`；
  - `condition` 可携带 `branches`，与 `condition add-branch` 写出的数据结构一致；
  - 通用字段：`title` / `icon` / `position` / `x` / `y`。
- `nodes[].bindings[]`：单节点的端口绑定，避免后续再调 `bind`。每条形如 `{ "port": "prompt", "mode": "literal", "value": "hi" }` 或 `{ "port": "agent", "mode": "ref", "source_node": "start_0", "source_port": "agent" }`，`mode` 缺省时若有 `source_node` 推断为 `ref`，否则按 `literal`，`mode=clear` 用于显式清空。
- `edges[]`：可使用对象 `{ "source", "target", "source_port", "target_port" }`，其中 `source_port` 用于 Condition 多路输出（取分支序号字符串）。

> 提示：执行 `byteval workflow builder plan schema` 可获得机器可读版本，节点字段名拼错时直接以该输出为准。

## Validate 覆盖范围

`validate` 会输出 `error` / `warning` 两类问题，Agent 必须修复所有 error，再按 warning 检查连线语义：

- 图结构：Start/End 数量、重复节点、重复边、自环、环路、坏边、Start 入边、End 出边、不可达节点、到不了 End 的分支、传递冗余边。
- 节点关键字段：Application `appID`、Workflow `workflowID`、Dataset `datasetID`、缺少 app schema。
- 输入绑定：必填未绑定、必填仍是 placeholder、可选未绑定、raw 必填未绑定、raw 输入名不在缓存 schema 中。
- 数据引用：引用缺失节点/端口、Ref 类型不匹配、Ref 不是目标节点上游、End 收集非上游、相似端口名建议、多上游同名端口歧义。
- Literal / schema-ref：Literal 类型可疑、系统模板放在非字符串端口、schema-ref 子字段缺失或类型未知。
- Workflow batch：batch input 缺失、batchSize 非法、batch 输入非数组（这三项已下沉为 `nodeBatchIssues` 通用校验，对任意节点生效）；缺少 outputList、outputList schema 缺失、子工作流 schema 可能过期（仅 Workflow 节点）。
- Condition：节点缺少 branches、分支无任何 conditions、表达式缺 comparator、分支缺少 outgoing 边（`condition_branches_missing` / `condition_branch_empty` / `condition_comparator_missing` / `condition_branch_unrouted`）。
- LLM 语义辅助：未使用输出、控制边缺少对应数据依赖、分支未汇入 End。

## 节点级 batch 配置

ASL 的 `data.inputs.batch` 在任意节点上都可以打开（Application / Workflow / Code …），它会让节点对一个数组进行迭代调用。builder 暴露三层入口：

### 1. 通用：`node-batch enable / disable / inspect`

适用于绝大多数批处理场景（包括把普通 Application 节点 batch 化）：

```bash
# 在 app_1 上启用 batch，把 parser_1.input_list 作为迭代源
byteval workflow builder node-batch enable \
  --sid "$SID" --node-id app_1 \
  --batch-source-node parser_1 --batch-source-port input_list \
  --batch-size 1

# 查看当前 batch 配置 + 批校验结果
byteval workflow builder node-batch inspect --sid "$SID" --node-id app_1

# 关闭 batch（仅清 _batch / data.inputs.batch，不动 inputParameters）
byteval workflow builder node-batch disable --sid "$SID" --node-id app_1
```

`enable` 写出的最小 batch 框架：

```json
{
  "batchEnable": true,
  "batchSize": 1,
  "inputLists": [
    {"name": "input_list", "type": "ref",
     "input": {"type": 7, "value": {"type": "Ref", "content": {"nodeID": "parser_1", "name": "input_list"}}}}
  ]
}
```

之后通常还要用 `bind` 把节点真正的输入端口指向迭代变量（即 `inputLists[].name`，默认与 `--batch-source-port` 同名）。例如：

```bash
byteval workflow builder bind --sid "$SID" --node-id app_1 \
  --port input_text --mode ref \
  --source-node app_1 --source-port input_list
```

### 2. Workflow 节点专属：`workflow-batch apply-schema`

子工作流节点除了 batch，还要按子工作流 DSL 同步生成 inputParameters / outputList schema，这一步只对 Workflow 节点有意义，独立保留：

```bash
byteval workflow builder workflow-batch apply-schema \
  --sid "$PARENT_SID" --node-id <child_workflow_node_id> \
  --from-dsl ./child.json \
  --batch-source-node <upstream_node_id> --batch-source-port <array_output_port>
```

`workflow-batch disable` / `workflow-batch inspect` 仍然存在但已 Deprecated，等价于在 Workflow 节点上执行 `node-batch disable/inspect`。

### 3. 兜底：`set-raw` / `edit-raw`

当 `node-batch enable` 还不够（例如平台新增了 `concurrencyMode`、`batchTimeoutSec` 等还没建模的子字段，或者要塞复杂的多 inputList 结构），仍可用 raw 系命令：

```bash
# 整段覆盖 data.inputs（含 batch + inputParameters）
byteval workflow builder set-raw --sid "$SID" --node-id app_1 --field inputs --from-file inputs.json

# 仅微调 batch 子字段
byteval workflow builder edit-raw --sid "$SID" --node-id app_1 --field inputs \
  --set inputs/batch/concurrencyMode='"parallel"'

# 把当前 raw 写出到文件做 diff
byteval workflow builder export-raw --sid "$SID" --node-id app_1 --field inputs --to-file /tmp/inputs.json
```

优先级建议：**`node-batch enable` ➜ 用 `bind` 把端口接到迭代变量 ➜ 必要时再用 `edit-raw`/`set-raw` 兜底**。

## Agent 规则

- 构建工作流时优先使用 `byteval workflow builder ...`，不要再调用旧 Python 脚本。
- 渲染不做自动端口匹配；先 `scope` 收敛候选，再由 Agent 选择语义正确的源端口并 `bind`。
- `upload` 是唯一会写平台的 builder 子命令；创建可视化草稿/副本前需要轻量确认，使用 `--update-id` 覆盖已有 workflow 前必须明确确认。
- `ASL` 上传时由 builder 自动序列化为字符串，不要手工把 DSL 对象嵌入请求体。
- 父工作流引用子工作流时，必须先上传/更新子工作流，拿到平台真实 Workflow ID 后再写入父工作流的 Workflow 节点；更新父工作流时禁止 Workflow 节点引用父工作流自身，否则平台卡片“编辑”会跳转错误。
- 若需要完全自定义复杂节点结构，优先用 `set-raw` / `edit-raw`，避免手拼完整 workflow body。
- 连线表示执行/控制顺序；若 `A -> C` 已经可通过 `A -> B -> C` 传递到达，不需要再额外添加 `A -> C`。`validate` 会对这类传递冗余边给出 `redundant_edge` error。

## 构建说明

Go 版 builder 挂载在主 CLI 的 `workflow builder` 子命令下，随仓库根目录的 `go build .`、`make -C build build-all` 和 `make -C build release` 一起编译发布。`package-skill` 只负责打包 skill 操作说明，不需要单独编译 builder 二进制。
