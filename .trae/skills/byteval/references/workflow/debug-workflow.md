# 场景：调试工作流

## CLI 命令

```bash
byteval workflow-task debug --workflow-id <workflow_id> --body-file <debug_task.json>
```

## 执行步骤

1. 确认工作流 ID
2. 准备调试请求体 `debug_task.json`（结构与创建任务类似，以 `-h` 输出为准）
3. 执行调试命令：
   ```bash
   byteval workflow-task debug --workflow-id <workflow_id> --body-file debug_task.json
   ```
4. 观察调试输出，定位问题节点

## 注意事项

- 调试运行不会产生正式任务记录，适合验证工作流逻辑
- 请求体字段与 `workflow-task create` 基本一致，参见 `references/workflow/start-workflow.md`
- 不确定参数时先执行 `byteval workflow-task debug -h`
