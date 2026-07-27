# 场景：分析工作流结果并出具报告

## Table of Contents

- [CLI 命令](#cli-命令)
- [执行步骤](#执行步骤)
- [节点识别经验](#节点识别经验)
- [最终输出结果的查看方式](#最终输出结果的查看方式)
- [平台链接生成规则](#平台链接生成规则)
- [动态分析建议](#动态分析建议)
- [建议输出模板](#建议输出模板)
- [注意事项](#注意事项)

## CLI 命令

```bash
# 先看任务详情与主链路节点
byteval workflow-task detail --id <task_id> --format json --asl-view compact

# 查看子任务进度汇总
byteval workflow-task subtask-progress --id <task_id> --format json

# 查看某个节点的输入输出
byteval workflow-task node-info --task-id <task_id> --node-id <node_id> --format json

# 查看批处理 Workflow 节点展开出的子任务明细
byteval workflow-task list --parent-task-id <task_id> --json

# 大输出建议落文件
byteval workflow-task detail --id <task_id> --format json --asl-view compact -o /tmp/task-detail.json
byteval workflow-task node-info --task-id <task_id> --node-id <node_id> --format json -o /tmp/node-info.json
byteval workflow-task list --parent-task-id <task_id> --json -o /tmp/subtasks.json
```

## 执行步骤

1. 先获取任务整体状态、报告 ID、主链路节点：
   ```bash
   byteval workflow-task detail --id <task_id> --format json --asl-view compact
   ```
   重点关注：
   - `Status`：任务是否已完成
   - `ReportID`：是否已经绑定报告
   - `TaskNodeInfos`：主链路节点状态
   - `WorkflowASL.nodes`：节点类型、标题、节点 ID

2. 在 `WorkflowASL.nodes` 中识别“结果产出节点”。
   常见信号：
   - 标题包含“生成多维表格”“导出飞书表格”“导出报告”“上传文件”“写入结果”
   - `nodeType` 为 `Application`，且通常靠近 `End` 节点
   - `nodeType` 为 `Dataset`，表示结果已被写入某个数据节点/数据集

3. 对候选结果节点逐个查看 `node-info`：
   ```bash
   byteval workflow-task node-info --task-id <task_id> --node-id <node_id> --format json
   ```
   重点关注 `TaskNodeInfo.Output` 中是否包含：
   - `table_url`
   - `file_url` / `download_url` / `url`
   - `report_url`
   - `data.table_url`
   - 其他导出文件链接、对象存储链接、下游平台跳转链接

4. 若某个结果节点后面跟着 `Dataset` 节点，再继续查看该 `Dataset` 节点的 `node-info`。
   这类节点通常表示：
   - 前一个节点产出的表格/文件链接已成功入库
   - 最终结果已经沉淀到平台数据节点，可作为后续查询入口

5. 若工作流中存在批处理 `Workflow` 节点，先看整体成功率：
   ```bash
   byteval workflow-task subtask-progress --id <task_id> --format json
   ```
   常见判断：
   - `TotalCount == SuccessCount`：整体执行成功
   - 存在失败/未完成：需要继续看子任务明细，定位异常样本

6. 查看子任务明细，补齐动态分析线索：
   ```bash
   byteval workflow-task list --parent-task-id <task_id> --json
   ```
   重点关注：
   - 子任务 `Status`
   - 子任务 `RuntimeDuration`
   - 子任务 `ReportID`
   - `ExtraInfo` 中的外部链接

7. 解析子任务 `ExtraInfo`，提取外部分析入口。
   常见字段：
   - `ReportID`
   - `评测`：评测流水线链接
   - `推理`：推理任务链接
   - `ReqID`：排障时可用于日志/链路检索

8. 当多维表或飞书表格链接已生成时，优先把它作为“最终结果页”返回给用户；同时补充任务侧摘要：
   - 主任务状态
   - 子任务总数 / 成功数 / 失败数
   - 报告 ID
   - 关键结果链接（优先补平台任务页 / 报告页）
   - 是否已成功写入数据节点

## 节点识别经验

### 1) 导出飞书表格 / 多维表格

典型特征：

- 标题类似“生成多维表格”“导出飞书表格”“写报告表”
- 常为 `Application` 节点
- `Output` 中常出现：
  ```json
  {
    "data": {
      "table_url": "https://bytedance.larkoffice.com/base/..."
    }
  }
  ```

拿到 `table_url` 后，可将其视为最终报告入口。

### 2) 存储到数据节点 / 数据集

典型特征：

- `nodeType` 为 `Dataset`
- 标题类似“结果表”“总任务表”“报告数据集”
- `Input` 中常直接引用上一个节点的 `table_url` 或文件链接

这类节点本身未必返回可读摘要，但可以证明：

- 结果文件/表格已经被后续节点消费
- 工作流结果已沉淀到平台内的可追踪数据节点

### 3) 批处理 Workflow 子任务

典型特征：

- `nodeType` 为 `Workflow`
- `batch.enabled = true`
- `outputs.outputList` 中往往是子任务或子流程结果集合

遇到这类节点时，不要只看主任务状态，还要补查：

- `subtask-progress`
- `list --parent-task-id`

否则容易漏掉部分样本失败、长尾慢任务或局部异常。

## 最终输出结果的查看方式

### 1) 表格链接

若结果节点输出了 `table_url`，直接返回给用户，例如：

```text
https://bytedance.larkoffice.com/base/...
```

说明：

- 这通常是飞书多维表的结果页
- 可用于查看总表、样本明细、维度统计等
- 当前执行环境若没有用户浏览器登录态，可能只能拿到链接，无法直接展开表内数据

### 2) 文件链接

若输出中出现 `file_url`、`download_url`、对象存储地址等：

- 直接返回链接
- 说明文件类型（若能判断）
- 说明其来源节点

### 3) 数据节点写入状态

若结果被写入 `Dataset` 节点：

- 返回“已成功写入数据节点”
- 若节点标题能体现目标表/目标数据集名称，也一并返回

## 平台链接生成规则

当用户明确希望“给我平台链接”“给我任务页/报告页链接”时，优先读取统一规则文档：

- `references/platform-link-rules.md`

本场景只需要额外记住两点：

- 先通过 `workflow-task detail --id <task_id> --format json` 获取 `ProjectID`、`ID`、`ReportID`
- 若 `ReportID` 为空或为 `0`，说明当前任务没有报告 ID，也没有报告链接；此时只返回任务页链接

## 动态分析建议

当用户要求“帮我分析结果”时，建议按以下顺序组织：

1. 任务概况
   - 任务 ID / 名称
   - 状态
   - 开始时间 / 结束时间 / 运行时长
   - 报告 ID

2. 结果入口
   - 多维表链接
   - 文件链接
   - 数据节点名称

3. 执行质量
   - 子任务总数
   - 成功数 / 失败数
   - 是否全量成功

4. 结构化观察
   - 是否存在长尾子任务
   - 是否存在失败子任务
   - 是否有可继续下钻的评测/推理链接

5. 下一步建议
   - 看总表
   - 查失败样本
   - 查最长耗时子任务
   - 查看某个节点的输入输出

## 建议输出模板

可按下面结构给用户返回：

```md
- 任务 `<task_id>` 已完成，状态为 `<status>`
- 报告 ID：`<report_id>`
- 结果表：<table_url>
- 子任务进度：`<success_count>/<total_count>` 成功
- 结果已写入数据节点：`<dataset_node_title>`

结论：
- 整体是否成功
- 是否存在失败样本或长尾任务
- 推荐优先查看的链接或下一步动作
```

## 注意事项

- `workflow-task detail --format json --asl-view compact` 通常最适合结果分析；`full` 输出过大，仅在需要完整 ASL 时再用
- `TaskNodeInfo.Input` / `Output` 常是 JSON 字符串，分析时要按 JSON 语义解析，不要只做字符串包含判断
- 多维表/飞书表格链接常受登录态与权限控制影响；即使 CLI 能拿到 `table_url`，也不代表当前环境可直接读取表内数据
- 若只拿到 `ReportID` 而无直接报告命令，可继续从结果节点、子任务 `ExtraInfo`、评测流水线链接补齐结果入口
- 若用户只要平台任务页/报告页链接，优先直接按“平台链接生成规则”返回，不必等结果节点产出 `table_url`
- 当 `node-info` 输出过大时，优先使用 `-o` 落盘，再局部分析，避免终端输出被截断
