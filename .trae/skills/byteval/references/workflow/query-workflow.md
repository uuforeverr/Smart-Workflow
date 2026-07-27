# 场景：查询工作流

## CLI 命令

```bash
# 列出工作流
byteval workflow list [--id <id>] [--name <name>] [--manager <email>] [--created-by <email>] [--is-custom true] [--custom-tag <tag>]

# 查看工作流详情
byteval workflow detail --id <id> [--format json] [--asl-view compact|skeleton|full]

- `skeleton`：默认模式，输出最精简，适合先快速查看主链路结构、节点名称、连接关系，以及节点是否开启批处理。
- `compact`：在结构图基础上保留变量关系与调试信息，适合排查节点 `inputs/outputs`、变量引用路径（如 `$node.output`）、条件分支、数据集/工作流引用，以及批处理入参等问题。
- `full`：返回完整解析后的 `ASL`，信息最全但体积可能很大，只建议在精细排障或需要核对原始节点定义时使用。

# 查看工作流单个节点的完整定义
byteval workflow node-info --id <id> --node-id <node_id> [--format json]

```

## 执行步骤

1. `byteval workflow list` 获取工作流列表
2. 按需增加筛选条件：
   - `--id`：按工作流 ID 精确筛选
   - `--name`：按名称筛选（精确/模糊以实际后端行为为准）
   - `--manager`：按管理员邮箱筛选（可重复传参，例如 `--manager alice@bytedance.com --manager bob@bytedance.com`）
   - `--created-by`：按创建人邮箱筛选（可重复传参）
   - `--is-custom true`：过滤自定义工作流
   - `--custom-tag`：按标签过滤
3. 记录目标工作流 ID
4. 按需执行 `workflow detail --id <id>` 查看工作流配置；JSON 输出会对 `ASL` 字段本身做压缩处理
5. `--asl-view` 使用建议：
   - `skeleton`：默认模式，适合先看主链路结构、节点名称与是否批处理
   - `compact`：适合排查输入输出、节点引用与批处理细节
   - `full`：返回完整 `ASL`，输出较大，慎用
6. 若 `compact` 中有参数被折叠，可用 `workflow node-info --id <id> --node-id <node_id>` 查看节点完整定义

## 注意事项

- `workflow list` 表格输出包含：ID、名称、描述、标签、管理员、创建人、创建时间、更新人、更新时间
- 如需查看创建任务所需的输入参数定义，优先使用 `workflow inputs --id <id>`
