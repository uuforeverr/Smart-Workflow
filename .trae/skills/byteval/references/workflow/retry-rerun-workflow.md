# 场景：重试任务 / 从节点重新执行工作流任务

## CLI 命令

```bash
# 从失败节点开始重试
byteval workflow-task retry --id <task_id>

# 处理批处理节点中的失败子任务
byteval workflow-task retry --id <task_id> --ignore-failed-child

# 清理指定节点及后继节点状态后，从该节点重新执行
byteval workflow-task rerun-from-node --task-id <task_id> --node-id <node_id>
```

## 执行步骤

1. 先确认目标任务 ID；如果不知道，可执行：
   ```bash
   byteval workflow-task list --workflow-id <workflow_id>
   ```
2. 若要从指定节点重跑，先查看任务详情或节点信息，确认 `node-id`：
   ```bash
   byteval workflow-task detail --id <task_id> --format json --asl-view compact
   byteval workflow-task node-info --task-id <task_id> --node-id <node_id> --format json
   ```
3. 若任务运行失败，且希望从失败节点继续执行：
   ```bash
   byteval workflow-task retry --id <task_id>
   ```
4. 若失败节点是批处理节点，且需要按失败子任务维度处理：
   ```bash
   byteval workflow-task retry --id <task_id> --ignore-failed-child
   ```
5. 若任务已完成、失败或运行中，且某个节点已是终态，想从该节点重新跑：
   ```bash
   byteval workflow-task rerun-from-node --task-id <task_id> --node-id <node_id>
   ```
6. 执行后用 `workflow-task detail --id <task_id>` 或 `workflow-task node-info --task-id <task_id> --node-id <node_id>` 确认状态变化。

## 适用建议

- `retry`：适合任务已失败，且希望从失败节点继续执行；如果失败节点是批处理节点，则会定位失败的子任务继续重试，而不是把已成功部分全部重跑。
- `retry --ignore-failed-child`：用于批处理节点失败时，按失败子任务维度进行重试，并结合后端的 ignore-failed-child 语义处理失败子任务。
- `rerun-from-node`：适合已有任务处于完成、失败或运行中时，指定一个已终态节点重新执行；后端会清理该节点及其后继节点的运行状态，再从该节点继续跑。

## 注意事项

- `task_id` 必填；`rerun-from-node` 额外要求 `node-id` 必填。
- `node-id` 建议先从 `workflow-task detail --format json --asl-view compact` 中确认，避免手填错误。
- `rerun-from-node` 不要求任务整体失败，只要求目标节点已经达到终态。
- 若用户执行后还想要平台任务页 / 报告页链接，按 `references/platform-link-rules.md` 的统一规则生成并返回。
