# 场景：查看工作流进度

## Table of Contents

- [CLI 命令](#cli-命令)
- [执行步骤](#执行步骤)
- [平台链接生成规则](#平台链接生成规则)
- [注意事项](#注意事项)

## CLI 命令

```bash
# 列出任务（可按状态过滤）
byteval workflow-task list --workflow-id <workflow_id> [--status <running|...>]

# 查看任务详情
byteval workflow-task detail --id <task_id> [--format json] [--asl-view compact|skeleton|full]

- `skeleton`：默认模式，适合先看主链路结构、节点名称与是否批处理。
- `compact`：适合排查节点输入输出、数据集/工作流引用、批处理入参等更细节的信息。
- `full`：返回完整 `WorkflowASL`，输出很大，仅在精细排障时使用。

# 查看节点信息
byteval workflow-task node-info --task-id <task_id> --node-id <node_id>

# 查看子任务进度汇总
byteval workflow-task subtask-progress --id <task_id> [--format json]

- 当工作流任务中包含 `Workflow` 节点，且该节点以批处理方式展开子流程时，可用这个命令观察子流程的整体推进进度。
- 如需进一步查看子任务列表明细，可按父任务 ID 查询：
  ```bash
  byteval workflow-task list --parent-task-id=<task_id> --workflow-id=<workflow_id> --json
  ```
  其中 `--workflow-id` 是可选过滤条件，指的是子任务对应的 workflow ID，而不是父任务本身的 workflow ID；不传时可先按 `parent-task-id` 查询该父任务下的全部子任务明细。

# 当输出很大时，建议直接写入文件
byteval workflow-task node-info --task-id <task_id> --node-id <node_id> --format json -o /tmp/node-info.json
byteval workflow-task detail --id <task_id> --format json --asl-view full -o /tmp/task-detail.json
```

## 执行步骤

1. 列出目标工作流下的任务：
   ```bash
   byteval workflow-task list --workflow-id <workflow_id>
   ```
2. 找到目标任务，获取 `task_id` 和当前 `status`
3. 查看任务详情（含各节点运行状态）：
   ```bash
   byteval workflow-task detail --id <task_id>
   ```
4. 如需查看子任务整体进度汇总：
   ```bash
   byteval workflow-task subtask-progress --id <task_id>
   ```
5. 如需查看某个节点的具体输入/输出：
   ```bash
   byteval workflow-task node-info --task-id <task_id> --node-id <node_id>
   ```
6. 当节点输入/输出或 `full` 模式返回体很大时，优先使用 `--output-file` / `-o` 写入文件，避免终端刷屏：
   ```bash
   byteval workflow-task node-info --task-id <task_id> --node-id <node_id> --format json -o /tmp/node-info.json
   ```

## 平台链接生成规则

当用户在“查任务进度”的同时希望拿到平台页面链接时，优先读取统一规则文档：

- `references/platform-link-rules.md`

本场景只需要额外记住两点：

- 先执行 `workflow-task detail --id <task_id> --format json`，读取 `ProjectID`、`ID`、`ReportID`、`Status`
- 若 `ReportID` 为空或为 `0`，说明当前任务没有报告 ID，也没有报告链接；此时只返回任务链接

## 注意事项

- `--status` 可选值以实际 `-h` 输出为准，常见值：`running`、`finished`、`failed`、`paused`
- `node-id` 可从 `workflow-task detail` 返回的节点列表中获取
- `detail/node-info` 支持 `--output-file` / `-o`，大数据场景建议直接写文件；`-o -` 表示继续输出到 stdout
- 若用户明确要“任务链接 / 报告链接”，即使没有 `table_url` 或其他结果产物，也应按上述规则直接返回平台链接
