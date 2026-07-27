# 场景：暂停/恢复工作流任务

## CLI 命令

```bash
# 暂停任务
byteval workflow-task pause --id <task_id>

# 恢复任务
byteval workflow-task resume --id <task_id>
```

## 执行步骤

1. 先确认任务 ID（不知道时执行 `byteval workflow-task list --workflow-id <workflow_id>`）
2. 暂停：`byteval workflow-task pause --id <task_id>`
3. 恢复：`byteval workflow-task resume --id <task_id>`
4. 执行后用 `workflow-task detail --id <task_id>` 确认状态变更

## 注意事项

- 只有处于 `running` 状态的任务才能暂停
- 只有处于 `paused` 状态的任务才能恢复
