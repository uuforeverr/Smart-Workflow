# 场景：推荐可复用评估器列表

## Table of Contents

- [适用范围](#适用范围)
- [能力边界](#能力边界)
- [执行入口](#执行入口)
- [参数说明](#参数说明)
- [输出字段](#输出字段)
- [使用建议](#使用建议)
- [返回模板](#返回模板)

## 适用范围

当用户已经描述评测诉求，希望从当前项目与共享空间中找可复用评估器时使用本页。

典型表达：

- “帮我找一下适合图片美学质量、图文匹配度的评估器”
- “这个评测指标有没有现成 evaluator 可以复用”
- “推荐几个和多图指令遵循相关的评估器”

## 能力边界

- 本能力只返回已有评估器推荐列表
- 不生成新的 evaluator skill；生成请读 `references/evaluator/evaluator-generate.md`
- 不推荐评测标准正文；标准推荐请读 `references/evaluation-metric/evaluation-metric-recommend.md`
- 推荐结果需要结合输入/输出字段确认是否能直接接入当前评测集或工作流

## 执行入口

优先使用项目内 CLI：

```bash
byteval evaluator recommend \
  --query "评测指标我们希望评估：图片美学质量、图文匹配度、多图指令遵循、视频强指令；" \
  --top-k 4
```

如果需要显式指定项目：

```bash
byteval evaluator recommend \
  --project-id 23 \
  --query-shared=true \
  --query "评测指标我们希望评估：图片美学质量、图文匹配度、多图指令遵循、视频强指令；" \
  --top-k 4
```

JSON 输出：

```bash
byteval evaluator recommend \
  --query "<evaluation_requirement>" \
  --top-k 4 \
  --format json
```

## 参数说明

| 参数 | 必填 | 默认值 | 说明 |
|---|---|---|---|
| `--query` | 是 | 无 | 用户的评测诉求或指标描述 |
| `--project-id` | 否 | 当前配置项目 | 当前项目 ID |
| `--query-shared` | 否 | `true` | 是否同时查询共享评估器 |
| `--top-k` | 否 | `4` | 返回推荐数量 |
| `--format` | 否 | `table` | `table` 或 `json` |

底层接口：

```http
POST https://byteval.bytedance.net/eagent/api/v1/evaluators/recommend
Content-Type: application/json
```

请求体：

```json
{
  "project_id": "23",
  "query_shared": true,
  "query": "评测指标我们希望评估：图片美学质量、图文匹配度、多图指令遵循、视频强指令；",
  "top_k": 4
}
```

## 输出字段

每个推荐项包含：

| 字段 | 说明 |
|---|---|
| `id` | 评估器标识 |
| `app_id` | ByteVal 应用 ID，可用于后续工作流/应用接入 |
| `name` | 评估器名称 |
| `category` | 评估器分类 |
| `score` | 推荐相关性分数 |
| `reason` | 命中原因 |
| `description` | 能力描述 |
| `project_id` | 评估器所属项目 |
| `detail` | 评估器详细说明，通常包含指标说明、输入字段、输出字段、参考链接 |

## 使用建议

推荐后需要重点核对：

- `app_id`：是否是后续要接入的真实应用 ID
- `description`：是否覆盖用户关心的评测目标
- `detail` 里的输入字段：是否能从当前评测集字段映射得到
- `detail` 里的输出字段：是否符合后续报告或工作流落表需要
- `project_id`：判断来自当前项目还是共享空间

当推荐结果不匹配时：

- 调整 `--query`，把业务对象、输入模态、关键指标写得更具体
- 增大 `--top-k`
- 必要时关闭 `--query-shared` 只查当前项目

## 返回模板

```md
推荐评估器：
- <name>（id=<id>, app_id=<app_id>, project_id=<project_id>）
  - 分类：<category>
  - 推荐原因：<reason>
  - 能力描述：<description>

接入前确认：
- 输入字段是否匹配当前评测集
- 输出字段是否满足后续工作流/报告使用
```
