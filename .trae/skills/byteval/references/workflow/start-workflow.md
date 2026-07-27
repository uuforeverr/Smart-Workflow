# 场景：启动工作流（创建并启动任务）

## CLI 命令

```bash
# 创建任务
byteval workflow-task create --workflow-id <workflow_id> --body-file <task.json>

# 启动任务
byteval workflow-task start --id <task_id>
```

## 执行步骤

1. 确认工作流 ID（不知道时先执行 `byteval workflow list`）
2. 查看工作流 Start 节点参数定义（必填/默认值/类型/说明）：
   ```bash
   byteval workflow inputs --id <workflow_id> --format table
   # 或原始 JSON：byteval workflow inputs --id <workflow_id>
   ```
   说明：该命令输出的参数建议写入 `GlobalVariable`（业务入参为 `user`，平台通用变量为 `common`）。
3. 准备任务创建请求体 `task.json`（参见下方 Schema）
4. 创建任务：
   ```bash
   byteval workflow-task create --workflow-id <workflow_id> --body-file task.json
   ```
5. 从返回结果中获取 `task_id`
6. 启动任务：
   ```bash
   byteval workflow-task start --id <task_id>
   ```
7. 若用户随后希望“给我任务链接 / 报告链接”，优先读取统一规则文档：
   - `references/platform-link-rules.md`
   说明：
   - `project_id` 优先取任务详情里的 `ProjectID`
   - 先从 `workflow-task detail --id <task_id> --format json` 读取 `ReportID`
   - 只有当 `ReportID` 非空且不为 `0` 时，才返回报告链接；否则只返回任务链接

## 请求体 Schema（task.json）

字段名**大小写敏感**，必须使用大写驼峰。

说明：Start 节点参数（`workflow inputs` 输出的 `NAME` 列）通常写进 `GlobalVariable`（而不是 `Input` 字段）。

```json
{
  "WorkflowID": 104,
  "Name": "任务名称",
  "Description": "任务描述",
  "ExpectedStartTime": 1777367614,
  "GlobalVariable": {
    "input": "helloworld",

    "evaluationSet": 15272,
    "evaluationSetVersion": 1,
    "ReportTemplateID": 1213,

    "x": [1],
    "y": 2
  }
}
```

| 字段 | 是否必填 | 说明 |
|---|---|---|
| `WorkflowID` | 必填 | 工作流 ID，整数 |
| `Name` | 必填 | 任务名称 |
| `Description` | 必填 | 不可省略 |
| `ExpectedStartTime` | 可选 | 预期启动时间（延迟启动场景才需要填写），秒级时间戳 |
| `GlobalVariable` | 必填 | Start 节点参数与业务变量；支持直接传 object，CLI 会内部序列化为 JSON 字符串（空时写 `{}`） |
| `ProjectID` | 可省略 | CLI 会自动注入默认项目 ID |
| `Managers` | 可省略 | CLI 会自动注入当前登录用户；若需显式传，格式为 `[{"Username":"xxx"}]` |
