# 平台链接规则

## Table of Contents

- [适用范围](#适用范围)
- [通用规则](#通用规则)
- [CN 站点 Host](#cn-站点-host)
- [工作流](#工作流)
- [工作流任务与报告](#工作流任务与报告)
- [评测集](#评测集)
- [应用](#应用)
- [返回建议](#返回建议)
- [注意事项](#注意事项)

## 适用范围

当用户明确希望“给我平台链接”“给我任务页/报告页/评测集页/应用页链接”时，优先按本页规则生成可直接打开的 Byteval 平台 URL。

本页只汇总目前已经验证过的前端路由规则；若后续平台前端改版，应以最新验证结果为准。

## 通用规则

生成链接前，优先确认以下字段：

- `project_id`
- 资源 ID（如 `task_id` / `evaluation_set_id` / `app_id`）
- 若是工作流结果场景，再补查 `ReportID`

`project_id` 取值优先级：

1. 当前命令显式传入的 `--project-id`
2. 资源详情返回中的 `ProjectID`
3. 当前本地配置的默认项目 ID

## CN 站点 Host

当前已确认可用的 CN 站点 Host：

```text
https://byteval.bytedance.net
```

当前已确认可用的 BOE 站点 Host：

```text
https://lsep-boe.bytedance.net
```

## 工作流

### 工作流页

CN 站点：

```text
https://byteval.bytedance.net/project/<project_id>/manageWorkflow/edit?id=<workflow_id>
```

BOE 站点：

```text
https://lsep-boe.bytedance.net/project/<project_id>/manageWorkflow/edit?id=<workflow_id>
```

### 生成规则

1. 先通过 `workflow list`、`workflow detail` 或已有上下文拿到：
   - `workflow_id`
   - `project_id`
2. 根据当前 `site` 选择对应 Host，路径格式与 CN 相同
3. 根据模板生成 `workflow_url`

### 补充说明

- 当前验证过的是工作流编辑页路由；对用户而言可作为工作流详情入口使用

## 工作流任务与报告

### 任务页

CN 站点：

```text
https://byteval.bytedance.net/project/<project_id>/manageWorkflow/task?taskId=<task_id>
```

BOE 站点：

```text
https://lsep-boe.bytedance.net/project/<project_id>/manageWorkflow/task?taskId=<task_id>
```

### 报告页

CN 站点：

```text
https://byteval.bytedance.net/project/<project_id>/manageWorkflow/report?taskID=<task_id>
```

BOE 站点：

```text
https://lsep-boe.bytedance.net/project/<project_id>/manageWorkflow/report?taskID=<task_id>
```

### 生成规则

1. 先通过 `workflow-task detail --id <task_id> --format json` 获取：
   - `ProjectID`
   - `ID`
   - `ReportID`
   - `Status`
2. 总是可以先生成 `task_url`
3. 只有当 `ReportID` 非空且不为 `0` 时，才生成 `report_url`

### 关键约束

- 报告页当前使用的是 `taskID=<task_id>`，不是 `ReportID=<report_id>`
- 若 `ReportID` 为空或为 `0`，说明当前任务没有报告 ID，也没有报告链接；此时只返回任务页链接

## 评测集

### 评测集页

CN 站点：

```text
https://byteval.bytedance.net/project/<project_id>/manageEvaluationSet?evaluationSetID=<evaluation_set_id>
```

BOE 站点：

```text
https://lsep-boe.bytedance.net/project/<project_id>/manageEvaluationSet?evaluationSetID=<evaluation_set_id>
```

### 生成规则

1. 先通过 `evaluation-set list`、`evaluation-set detail` 或已有上下文拿到：
   - `evaluation_set_id`
   - `project_id`
2. 根据当前 `site` 选择对应 Host，路径格式与 CN 相同
3. 根据模板生成 `evaluation_set_url`

## 应用

### 应用页

CN 站点：

```text
https://byteval.bytedance.net/project/<project_id>/manageApplication/edit?from=AppListPage&id=<app_id>
```

BOE 站点：

```text
https://lsep-boe.bytedance.net/project/<project_id>/manageApplication/edit?from=AppListPage&id=<app_id>
```

### 生成规则

1. 先通过 `application list`、`application get` 或已有上下文拿到：
   - `app_id`
   - `project_id`
2. 根据模板生成 `application_url`

### 补充说明

- 若应用来自跨项目共享场景，当前项目页路由是否仍可直接打开，以实际平台行为为准

## 返回建议

给用户返回平台链接时，建议同时补充资源摘要，避免只扔 URL：

- 工作流任务：`task_id`、`report_id`、`status`
- 评测集：`evaluation_set_id`、名称、版本
- 应用：`app_id`、名称、接入类型或状态

## 注意事项

- BOE 与 CN 站点的前端路由路径格式**相同**，只需把 Host 从 `byteval.bytedance.net` 换成 `lsep-boe.bytedance.net`；不要猜测其他域名
- SG 站点链接暂未验证，切到 `sg` 时应重新核实前端路由
- 当用户明确要平台链接时，优先直接返回链接，不必只停留在 CLI 查询结果
