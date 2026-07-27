# 场景：根据 CSV 评测结果生成评测报告

## Table of Contents

- [适用范围](#适用范围)
- [报告类型](#报告类型)
- [执行入口](#执行入口)
- [飞书文档报告](#飞书文档报告)
- [HTML 报告](#html-报告)
- [CSV 表头规范](#csv-表头规范)
- [结果与错误处理](#结果与错误处理)
- [返回模板](#返回模板)

## 适用范围

用于根据输入的评测结果集（本地 CSV 文件），结合指定的评测类型及配置参数生成报告，适用于各类 AI 任务、文本处理任务的评测结果汇总分析。用户可根据需要选择二选一产出报告：

- 飞书文档版：调用报告生成服务，最终输出包含报告具体内容的飞书文档链接。
- HTML 可视化版：调用 HTML 报告生成 API，CLI 将返回的 HTML 内容写成本地单文件。

支持的报告类型：

| 类型 | 适用数据 |
|---|---|
| `SCORE` | 打分型评测结果 |
| `LABEL` | 打标型评测结果 |
| `GSB` | 多模型 GSB 对比结果 |
| `MULTI` | 多维度通过/失败/枚举类评估结果 |

## 报告类型
### MULTI（综合报告）
- 定义：对模型输出进行综合评估，存在一个或多个维度，每个维度有评估结果和评估原因，评估原因非必选。
- 数据字段：
    - 评估结果字段：字段类型可以是评分、打标标签。
    - 评估原因字段：字段类型为字符串。

### SCORE（评分报告）
- 定义：对模型输出进行评分，存在一个或多个维度，每个维度有评分和评分原因，评分原因非必选。
- 数据字段：
    - 评分字段：字段类型为浮点数或整数
    - 评分原因字段：字段类型为字符串。

### LABEL（打标报告）
- 定义：对模型输出效果进行打标，结果常见为 GoodCase / BadCase / NormalCase、优/中/差 等标签。
- 数据字段：
    - 打标字段：字段类型为字符串，记录模型输出的标签，例如 GoodCase / BadCase / NormalCase、优/中/差 等。
    - 打标原因字段：字段类型为字符串，记录对模型输出标签的解释或原因。

### GSB（GSB报告）
- 定义：对两个模型结果（分为基准模型和评测模型）进行对比，同时有归因字段记录较差模型的归因。GSB也可能存在一个或多个维度。
- 数据字段：
    - 评测结果字段：字段类型为字符串，记录对比结果，评测结果通常是某个模型的名称，表明该模型更优，或者使用其他方式标记哪个模型更优。
    - 指标归因字段：字段类型为字符串，记录对应模型的评估归因。
    - 模型结果字段：字段类型为字符串，记录对应模型的输出结果。

## 能力边界

- 本能力只做“根据已有 CSV 结果生成报告”
- 不负责推荐评测标准；标准推荐请读 `references/evaluation-metric/evaluation-metric-recommend.md`
- 不负责生成评估器；评估器生成请读 `references/evaluator/evaluator-generate.md`
- 飞书报告依赖后端报告生成接口；HTML 报告依赖 HTML 报告生成 API，并在本地落盘 API 返回的 HTML

## 执行入口

优先使用项目内 CLI：

```bash
byteval evaluation-report generate \
  --output-kind feishu \
  --user-email <user_email> \
  --report-title <report_title> \
  --report-type <SCORE|LABEL|GSB|MULTI> \
  --csv-data-path <csv_path>
```

通用参数：

| 参数 | 必填 | 说明 |
|---|---|---|
| `--output-kind` | 否 | `feishu` 或 `html`，默认 `feishu` |
| `--user-email` | 否 | 报告生成服务使用的用户邮箱，默认 `byteval_cli@bytedance.com` |
| `--report-title` | 是 | 报告标题 |
| `--report-type` | 是 | `SCORE` / `LABEL` / `GSB` / `MULTI` |
| `--csv-data-path` | 是 | 本地 CSV 评测结果路径 |
| `--report-rule` | 否 | 报告生成规则或额外要求；HTML 模式下用于承载 Agent 生成的报告诉求 |
| `--html-output` | 否 | HTML 报告本地输出路径，仅 `--output-kind html` 使用 |
| `--format` | 否 | `text` 或 `json` |

## 飞书文档报告

飞书报告会把 CSV 内容发送给报告生成服务，并从 SSE 返回流中读取最终报告链接。

```bash
byteval evaluation-report generate \
  --output-kind feishu \
  --user-email alice@bytedance.com \
  --report-title "小说摘要评测报告" \
  --report-type SCORE \
  --report-rule "按维度总结问题分布，突出低分样例" \
  --csv-data-path /tmp/score.csv
```

注意：

- `--user-email` 未传时默认使用 `byteval_cli@bytedance.com`
- CLI 会先校验 CSV 表头是否匹配 `--report-type`
- 若返回网络异常或服务端失败，需要重试或检查鉴权/服务状态

## HTML 报告

HTML 报告通过远端 API 生成。CLI 会读取 CSV 的完整评测结果，将其按 CSV 字符串作为 `dataset` 发送给服务，并把服务返回的 HTML 内容写成本地文件。

```bash
byteval evaluation-report generate \
  --output-kind html \
  --user-email alice@bytedance.com \
  --report-title "多维评估报告" \
  --report-type MULTI \
  --report-rule "请生成一份包含整体结论、关键指标分析、典型样例和优化建议的 HTML 评测报告" \
  --csv-data-path /tmp/multi.csv \
  --html-output /tmp/multi-report.html
```

说明：

- HTML 生成接口：`https://byteqa.bytedance.net/eval_agent/html_report_generate`
- 请求体由 CLI 组装为 `{ "prompt": "<prompt>", "dataset": "<完整 CSV 评测结果>" }`
- 不要使用 `--prompt` flag。HTML 模式下，Agent 应把报告诉求写入已有的 `--report-rule`；CLI 会据此生成发送给 API 的 `prompt`
- `--user-email` 未传时默认使用 `byteval_cli@bytedance.com`
- `--html-output` 未传时，默认按报告标题生成当前目录下的 `.html` 文件
- HTML 模式不依赖本地表头模式识别；CLI 会把 CSV 明细完整发送给 API
- 不要在 Agent 侧手写 HTML，也不要沿用旧版本地 HTML 样式/图表/指标计算说明；HTML 的展示能力以服务返回为准

## CSV 表头规范

飞书模式会根据表头推断数据类型，并要求推断结果与 `--report-type` 一致。HTML 模式不依赖这些表头模式，CLI 会把 CSV 明细完整发送给 HTML 生成 API。

### SCORE

必需列：

- 至少一个 `{{维度名称}}评分`

可选列：

- `{{维度名称}}评估原因`
- 或公共列 `评估原因`

示例：

```csv
问题,回答,真实性评分,真实性评估原因
q1,a1,4,基本准确
```

### LABEL

必需列：

- 至少一个 `{{维度名称}}打标结果`

可选列：

- `{{维度名称}}打标原因`
- 或公共列 `打标原因`

示例：

```csv
问题,回答,安全打标结果,安全打标原因
q1,a1,通过,无风险
```

### GSB

必需列：

- 模型输出列，例如 `模型A输出`、`模型B输出`
- 至少一个 `{{维度名称}}GSB结果`
- 至少一个 `{{维度名称}}指标归因-{{模型名称}}`

`GSB结果` 的值通常是胜出的模型名，或 `无法区分`。

示例：

```csv
问题,模型A输出,模型B输出,有用性GSB结果,有用性指标归因-模型A
q1,a,b,模型A,信息更完整
```

### MULTI

必需列：

- 至少一个 `{{维度名称}}评估结果`

可选列：

- `{{维度名称}}评估原因`

示例：

```csv
问题,回答,真实性评估结果,真实性评估原因
q1,a1,通过,事实一致
```

## 结果与错误处理

成功时文本输出：

```text
report_url=<feishu_report_url>
```

或：

```text
html_report_file=<html_path>
```

常见错误：

| 错误 | 处理 |
|---|---|
| CSV 表头与报告类型不匹配 | 仅飞书模式会做严格校验；按上方规范修改表头，或改正 `--report-type` |
| 报告服务网络失败 | 重试，或检查网络、鉴权和服务状态 |
| 报告服务返回空内容 | 检查 CSV 是否为空、报告类型是否正确 |

## 返回模板

飞书报告：

```md
已生成评测报告：
- 报告链接：<report_url>
- 报告类型：<report_type>
- 数据文件：<csv_path>
```

HTML 报告：

```md
已生成 HTML 评测报告：
- 文件路径：<html_report_file>
- 报告类型：<report_type>
- 数据文件：<csv_path>
```
