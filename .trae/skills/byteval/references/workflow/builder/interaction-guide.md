# Workflow Builder Agent 交互指引

本文说明 Agent 如何把用户的自然语言工作流需求，转化为可复用、可验证、可上传、可迭代的 ByteVal workflow。具体 builder 命令见 `command-reference.md`。

## Table of Contents

- [总原则](#总原则)
- [推荐流程](#推荐流程)
- [意图归纳](#意图归纳)
- [Workflow 复用策略](#workflow-复用策略)
- [ASCII Brief](#ascii-brief)
- [应用选择](#应用选择)
- [自动绑定策略](#自动绑定策略)
- [胶水节点](#胶水节点)
- [上传与迭代](#上传与迭代)
- [与命令参考的关系](#与命令参考的关系)

## 总原则

- 用户确认业务拓扑，Agent 处理技术绑定。
- 先搜索可复用 workflow，再搜索 application，最后补胶水节点。
- 相似 workflow 优先 clone 到 IR 后二次修改，从零搭建只是兜底方案。
- 大部分端口绑定不暴露给用户，只有低置信度或多候选时才确认。
- `validate` 必须达到 0 error；warning 需要检查并在必要时说明。
- 上传可作为可视化确认手段，默认创建草稿/副本；覆盖更新已有 workflow 必须明确确认。
- 节点类型相关命令统一走各自命令组：`node-batch ...` 是任意节点（Application / Workflow / Code …）通用的 batch 开关与查看；`workflow-batch apply-schema` 仅 Workflow 节点专用（需要拉子流 DSL 反推 outputList）；`condition ...` 管 Condition 节点的分支与表达式。旧的 `workflow-node-schema` 名以及 `workflow-batch disable/inspect` 已是过渡别名，新代码请走 `node-batch`。

## 推荐流程

1. 意图归纳：把用户目标总结为能力链路、输入来源、输出目标和运行方式。
2. 搜索 workflow：先找完整相似 workflow、可复用子 workflow 或可参考拓扑。
3. 决定构建策略：完全复用、复制微调、引用子流、参考拓扑或从零搭建。
4. 搜索缺失应用：仅对相似 workflow 缺失或需要替换的能力槽位搜索 application。
5. 生成 ASCII Brief：用图、复用策略、风险提示和待确认项让用户确认业务设计。
6. 导入或创建 session：相似 workflow 用 `clone-ref` / `--from-dsl` 转 IR；从零搭建用 `init`。
7. 拉真实 schema：使用 `application get --json` / `app-schema` 缓存应用输入输出。
8. 自动绑定：Agent 基于 schema、端口语义和上游连通性完成 `scope` / `bind`。
9. 动态补胶水节点：schema 不兼容时，先搜索已有转换应用，再建议创建 Python 应用。
10. validate 与修复：自动修复结构和绑定错误，直到 0 error。
11. 上传可视化草稿：默认创建新 workflow / 副本，让用户到平台 UI 看图确认。
12. 二次 update：根据用户反馈，基于同一 session 或重新 clone 草稿 workflow 继续修改。

## 意图归纳

用户通常不会提供节点 ID、端口名、ASL 或完整参数绑定。Agent 应先把需求归纳为少量业务槽位：

- 项目与目标：`project_id`、workflow 名称、是否创建父子工作流。
- 输入来源：手动输入、评测集、测试集、已有 workflow 输出或固定 literal。
- 能力链路：例如 `测试集读取 -> 文本提取 -> 文生图 -> 图文裁判 -> 表格导出`。
- 输出目标：End 字段、飞书表格、报告、保存结果或其他平台资源。
- 运行方式：是否 batch、batchSize、是否单条样本子 workflow。
- 权限信息：Managers、描述、是否允许搜索/复用 workflow、是否允许创建胶水应用。

首次澄清要尽量一次性收集关键信息，不要逐字段追问。可使用如下话术：

```text
我理解你想搭一个“测试集文本 -> 文生图 -> 图文裁判 -> 表格导出”的工作流。
为了减少来回确认，请一次性补充或确认：

1. project_id 和 workflow 名称是什么？
2. 输入来自测试集、评测集，还是手动文本？
3. 是否允许我先搜索可复用 workflow，再搜索缺失应用？
4. 是否允许必要时创建 Python 胶水应用做字段转换？
5. 最终输出到 End、飞书表格，还是两者都要？
6. 是否需要 batch 运行，batchSize 默认 1 可以吗？
7. Managers / 描述是否有指定值？
```

## Workflow 复用策略

搜索 workflow 后，按改造成本选择策略：

- 完全复用：已有 workflow 能直接满足目标，只需返回或引用。
- 复制微调：拓扑高度相似，但应用、输入、导出字段或常量需要调整。
- 作为子流引用：已有 workflow 能完成某个能力段，例如单条样本处理。
- 仅参考拓扑：结构相似但 schema、项目或权限不适配，只参考节点顺序。
- 从零搭建：没有相似 workflow，或改造成本高于新建。

当选择复制微调时，不要重新从 0 搭建。应拉取已有 workflow 的 DSL/ASL，导入 session 转成 IR，然后用 builder 对 IR 做差异修改。

```text
workflow detail / 本地 DSL
  -> clone-ref / --from-dsl
  -> ir.json
  -> 替换应用 / 增删节点 / 改边 / 改绑定 / 改 End 输出
  -> preview
  -> validate
  -> upload 新副本或 --update-id 更新
```

## ASCII Brief

Workflow Brief 面向用户确认业务设计，不面向机器执行。避免展示 YAML、完整 ASL 或全部端口绑定。

Brief 应包含：

- 图：父工作流 / 子工作流 / 关键数据流。
- 复用策略：完全复用、复制微调、引用子流、从零搭建。
- 计划改动：保留、替换、新增、删除、修改的节点或输出。
- 风险提示：schema 不兼容、候选应用不确定、导出字段不完整等。
- 待确认项：只问会影响业务拓扑或平台写入的决策。
- 自动处理项：schema 拉取、字段绑定、batch、ASL、validate 修复等。

### 从零搭建示例

```text
计划创建 2 个工作流

[父工作流] 批量调度与导出
测试集 3439/v1
  -> 查询测试集明细
  -> 文本提取
  -> 批量调用 [子工作流] batchSize=1
  -> 飞书表格导出
  -> End(success=ok)

[子工作流] 单条文生图与裁判
输入文本
  -> 文生图
  -> 图文裁判
  -> End(输入文本, 输出图片, 图文匹配度, 图像清晰度, 图像美观度, 图像创意分)

需要你确认：
1. 是否按“父工作流负责批量，子工作流负责单条处理”设计？
2. 推荐应用是否正确？
3. 导出表格字段是否就是子工作流 End 的 6 个字段？

我会自动处理：
- 真实 application schema 拉取
- 字段绑定
- batch input / outputList
- ASL 生成
- validate 修复到 0 error
```

### 二次修改示例

```text
改造计划：基于已有 workflow 二次编辑

来源 workflow：
文生图评测导出工作流  id=1588

当前流程：
测试集 -> 文本解析 -> 文生图A -> 裁判A -> 导出表格

目标流程：
测试集 -> 文本解析 -> 文生图B -> 裁判B -> 导出表格

计划改动：
[保留] 测试集读取
[保留] 文本解析
[替换] 文生图A -> 文生图B
[替换] 裁判A -> 裁判B
[修改] End 输出增加：图像创意分
[修改] 导出字段增加：输入文本、输出图片、四个评分

风险提示：
1. 替换应用后字段 schema 可能不同，需要重新绑定。
2. 导出表格字段需要和子工作流 outputList 对齐。
3. 如果字段不兼容，可能新增 Python 胶水节点。

需要确认：
1. 是否基于 workflow 1588 二次修改？
2. 是否保留测试集读取和导出逻辑？
3. 是否允许上传草稿后再在平台 UI 继续确认？
```

### Condition 分支示例

```text
在"图文裁判"后加一个 Condition 节点，根据匹配度走不同分支：

图文裁判 -> Condition(图文匹配度 >= 0.8)
  ├─ true  -> 合格结果合并 -> 飞书表格导出
  └─ false -> 不合格结果合并 -> 失败样本表导出

说明：
- Condition 节点用 `condition` 命令组管理分支与表达式。
- 表达式示例：`inputs.image_text_judge.图文匹配度 >= 0.8`。
- 两个分支最终都汇入 End，End 字段保持一致以便统一导出。

需要确认：
1. 阈值是否取 0.8？
2. 不合格样本是否单独导出到失败表？
```

## 应用选择

搜索应用前先确定能力槽位，再为缺失槽位搜索 application：

- `dataset_detail_query`：查询测试集/评测集明细。
- `text_extract`：从明细中提取输入文本。
- `text_to_image`：文生图。
- `image_text_judge`：图文裁判或多维打分。
- `table_export`：飞书表格导出。
- `merge_result`：把原始输入、模型输出和评分合成表格行。

应用候选展示 1 个推荐和少量备选即可。选择依据应包含名称相似度、项目范围、真实 schema 匹配度、上下游兼容性和历史 workflow 复用情况。

## 自动绑定策略

端口绑定是技术细节，默认不让用户确认。Agent 应通过以下信息自动决定：

- `app-schema` / `application get --json` 的真实输入输出。
- `scope` 返回的类型兼容候选。
- 端口名称和同义词，如 `prompt`、`输入文本`、`text`、`query`。
- 应用能力元信息，如 `input_roles`、`output_roles`、`preferred_upstream`。
- 图连通性，绑定来源必须是目标节点上游。
- batch 语义，父 workflow 调用子 workflow 时必须处理 batch input 和 `outputList`。

只有在以下情况展示绑定确认：

- 多个候选同名或语义相近。
- 类型兼容但业务含义不确定。
- 需要从复杂对象里选择字段。
- 系统模板、固定 literal 或敏感参数会影响输出。
- 自动绑定会改变用户可见结果。

### Condition 分支编排

如果业务需要"按条件走不同分支"，优先使用 Condition 节点而不是堆多份 Application 节点：

- 用 `add-node --kind condition` 建节点；用 `condition add-branch` 追加分支。
- 用 `condition set-expression --comparator ... --left-node ... --left-port ... --right ...` 描述每个分支判定。
- Condition 的输出连线必须显式带 `--source-port`：`add-edge --source cond --target X --source-port 0` 表示走第 0 条分支；`--source-port 0`/`1` 等数字与 `condition list-branches` 给出的索引一一对应。
- 每条分支至少要有一条出边；Validate 会用 `condition_branch_unrouted` 提醒未连出的分支。
- 如果业务要求"匹配某个条件后才走 Application"，就让 Application 的入边来自 Condition 的对应分支端口；否则容易出现"分支为真但下游永远跑不到"的情况。

### Workflow 子流批量编排

调用子 workflow 时，结合 `node-batch`（通用）与 `workflow-batch apply-schema`（Workflow 专属）：

- 通用开关：`node-batch enable` 给任意节点（包括 Workflow / Application / Code 等）开启 batch；`--batch-source-node/--batch-source-port` 指向真正给批输入的父级输出，`--input-list-name` 默认与 source-port 同名，可显式覆盖。
- Workflow 子流场景再额外执行 `workflow-batch apply-schema`，由它反推 batch inputs / outputList schema（这是 Workflow 节点独有的，依赖子流 DSL 的 Start/End）。
- 关闭/查看：统一用 `node-batch disable` / `node-batch inspect`；旧的 `workflow-batch disable/inspect` 仍可用，但已标记 Deprecated，输出会提示迁移。
- inspect 仅展示当前 batch 配置（包括通用 `nodeBatchIssues` 与 Workflow 额外的 outputList / 子流 schema 校验），不修改任何状态，可在出错时优先调用确认现状。

## 胶水节点

胶水节点用于解决应用之间的 schema 不兼容，不需要在最初需求澄清时全部确定。Agent 可以在拉 schema、绑定或 validate 过程中动态建议。

常见胶水节点：

- 字段提取：从对象数组中提取文本、ID、图片 URL。
- 字段重命名：把 `image_url` 转成 `输出图片`。
- 数组包装：把单个对象包装成数组。
- 数组展开：把列表作为 batch 输入。
- 结果合并：把输入、输出和评分合成导出行。
- 类型转换：字符串、数字、布尔、JSON 对象互转。
- 默认值补齐：补 `ProjectID`、标题、管理员等固定参数。
- 过滤清洗：过滤空文本、非法图片 URL 或异常评分。

处理顺序：

1. 先搜索已有转换应用。
2. 找不到合适应用时，建议创建 Python 应用。
3. 用户确认转换意图，不要求用户确认代码细节。
4. 创建后拉取真实 schema，再插入 workflow 并重新 validate。

运行中提示示例：

```text
发现需要一个转换步骤：

上游：查询测试集明细列表.EvaluationSubSetDetails
下游：子工作流批量输入.输入文本列表

问题：下游需要字符串数组，但上游是测试集详情对象数组。

推荐方案：
A. 搜索已有“解析测试集文本内容”应用
B. 创建 Python 胶水应用：从每条详情中提取“输入文本”
C. 手动指定应用 ID

是否按 A -> 不存在则 B 执行？
```

## 上传与迭代

Workflow 图结构很难只在对话框里确认。生成并 validate 通过后，可以上传一个可视化草稿，让用户到平台 UI 看图后反馈。

默认策略：

- 基于已有 workflow 改造时，默认创建新副本，不覆盖原 workflow。
- 只有用户明确要求更新某个 workflow 时，才使用 `upload --update-id`。
- 草稿名称建议加 `Agent Draft` 或日期后缀，description 记录来源 workflow。
- 上传前说明项目、名称、来源、动作和 validate 结果。
- 上传后保留 session，后续根据用户反馈继续 update。
- **`upload` 必须传 `--description`**，平台不接受空值；名称不能与同项目已有 workflow 重复（加后缀避免）。

上传前轻量确认示例：

```text
我已经生成并校验通过，准备上传一个可视化草稿：

名称：文生图评测工作流 - Agent Draft
项目：13
来源：基于 workflow 1588 二次修改
动作：创建新 workflow，不覆盖原 workflow
validate：0 error

是否上传到平台供你可视化确认？
```

覆盖更新必须强确认：

```text
这是更新已有 workflow，会覆盖线上配置：
workflow_id=1588

请确认是否执行 update。
```

## 评测集工作流标准模式

### 场景描述

当用户需要"读取评测集 → 解析文本 → 批量调用分类/评测算法"时，推荐以下父子工作流结构：

```text
[父工作流] 批量调度
Start(evaluationSet, evaluationSetVersion)
  -> GetEvaluationSubSetDetails(appID=6970)   # 读取评测集明细
  -> EvalParser(Python glue)                  # 解析明细，提取 input_list
  -> [子工作流] batchSize=1                   # 批量处理每条样本
  -> End

[子工作流] 单条处理
Start(input_text)
  -> Classifier / Evaluator
  -> End(label, confidence, ...)
```

### 1. 读取评测集节点（appID=6970）

`GetEvaluationSubSetDetails` 固定 appID=6970，不要通过 `search` 查找（搜索可能返回空）。

```bash
# 直接用 appID 拉 schema，不要先 search；add-node 前必须先 app-schema 缓存，否则报 missing_app_schema
byteval workflow builder app-schema --sid "$SID" --app-id 6970 --project-id <project_id>
byteval workflow builder add-node --sid "$SID" --kind application --app-id 6970 --title "读取评测集"
```

**必填输入绑定**（三个字段缺一不可）：

| 输入端口 | 绑定来源 | 说明 |
|---|---|---|
| `EvaluationSetID` | Start.evaluationSet | 评测集 ID |
| `EvaluationSetVersion` | Start.evaluationSetVersion | 版本号 |
| `ProjectID` | literal（`--literal-type 2`） | 项目 ID，传数字即可，内部自动存为字符串 |

绑定 `ProjectID` 示例：

```bash
byteval workflow builder bind --sid "$SID" --node-id <app_node_id> \
  --port ProjectID --mode literal --value 12 --literal-type 2
```

**输出**：`EvaluationSubSetDetails`（`[]string`，每条是 JSON 序列化字符串）

### 2. 解析评测集明细（Python 胶水节点）

`EvaluationSubSetDetails` 中每条是 JSON 字符串，结构固定为：

```json
{
  "Content": { ...评测集原始字段... },
  "EvaluationSetID": 21610,
  "EvaluationSetVersion": 1,
  "EvaluationSetVersionID": 34969,
  "IndexID": 0,
  "_id": "..."
}
```

**关键：`Content` 内的字段名由评测集上传的原始数据决定，不能硬编码。** 写解析器前必须先查 `EvaluationSetVersion.Features` 确认字段名：

```bash
byteval evaluation-set detail --evaluation-set-id <id> --version <ver> --format json
```

重点关注返回中的 `EvaluationSetVersion.Features`，它列出了 `Content` 的所有字段名与类型。

#### 完整示例

**第一步：查看评测集字段**

以评测集 `SentimentEvalSet_20`（ID=21610，version=1）为例，`Features` 为：

```json
[
  {"Name": "answer",   "Type": "string"},
  {"Name": "id",       "Type": "integer"},
  {"Name": "question", "Type": "string"}
]
```

**第二步：读取评测集节点输出**

`GetEvaluationSubSetDetails`（appID=6970）返回的 `EvaluationSubSetDetails` 是字符串数组，每条反序列化后如下：

```json
{
  "Content": {
    "id": 1,
    "question": "什么是 Byteval？",
    "answer": "Byteval 是字节跳动的大模型评测平台。"
  },
  "EvaluationSetID": 21610,
  "EvaluationSetVersion": 1,
  "EvaluationSetVersionID": 34969,
  "IndexID": 0,
  "_id": "6a02c741689731d7ec9f7dff"
}
```

**第三步：按字段名写解析器**

本例要把 `question` 提取出来作为分类器的输入文本：

```python
import json
def sink(data: dict) -> dict:
    """
    data 格式: {"details": ["<json_str>", ...]}
    每条 json_str 结构: {"Content": {...}, "EvaluationSetID": ..., "IndexID": ...}
    Content 内的字段名取决于评测集，需提前通过 Features 确认
    """
    details = data.get("details", [])
    inputs = []
    for item_str in details:
        try:
            item = json.loads(item_str)
            content = item.get("Content", {})
            # 字段名根据 EvaluationSetVersion.Features 确认，不要硬编码 "input"
            inputs.append(content.get("question", ""))
        except:
            inputs.append("")  # 单条失败补空，保持与输入等长，避免下游 batch 索引错位
    return {"input_list": inputs}
```

**第四步：验证输出**

用真实评测集数据调试，预期输出：

```json
{
  "input_list": [
    "什么是 Byteval？",
    "Byteval 支持哪些文件格式？",
    "如何上传评测集？"
  ]
}
```

若 `input_list` 中出现空字符串，说明字段名写错，需重新检查 `Features`。

**注意**：
- `except` 分支用 `append("")` 而非 `continue`，保证 `input_list` 长度与 `details` 始终一致，下游 batch 节点按索引对应不会错位
- 创建或更新解析器应用后，务必用真实评测集数据调试验证输出不为空字符串

**输入 schema**：`{"details": []}` **输出 schema**：`{"input_list": []}`

将解析器的 `details` 端口绑定到 `GetEvaluationSubSetDetails.EvaluationSubSetDetails`。

### 3. 子工作流批处理配置

> 通用形态：任意节点（Application / Workflow / Code …）都可以用 `node-batch enable` 开启 batch；Workflow 子流再叠加 `workflow-batch apply-schema` 反推 outputList。

```bash
# 第一步（仅 Workflow 子流场景）：生成子工作流节点 schema（子工作流必须已上传，有真实 ID）
byteval workflow builder workflow-batch apply-schema --sid "$PARENT_SID" \
  --workflow-id <child_wf_id> \
  --batch-source-node <parser_node_id> \
  --batch-source-port input_list

# 第二步：通用 batch 开关 —— 任意节点都可以
byteval workflow builder node-batch enable --sid "$PARENT_SID" \
  --node-id <batch_node_id> \
  --batch-source-node <parser_node_id> \
  --batch-source-port input_list \
  --input-list-name item \
  --batch-size 1

# 第三步：把节点入参绑定到自身的迭代变量端口（item），用 bind 而非手撸 set-raw
byteval workflow builder bind --sid "$PARENT_SID" \
  --node-id <batch_node_id> \
  --port input_text --mode ref \
  --source-node <batch_node_id> --source-port item

# 第四步：查看现状（含通用 nodeBatchIssues + Workflow 额外校验）
byteval workflow builder node-batch inspect --sid "$PARENT_SID" --node-id <batch_node_id>
```

**关键规则**：
- `node-batch enable` 写出的 `batch.inputLists` 自带迭代变量定义，`input` 绑定到上游的数组端口；不用再手撸 set-raw 框架。
- 节点入参（`inputParameters`）仍要用 `bind --mode ref --source-node <self> --source-port <listName>` 引用自己的迭代变量端口。
- `inputParameters` 的 `name` 必须与目标节点入参（如子工作流 Start 节点的输入端口名）一致。
- 仅当需要调整 `concurrencyMode` / `batchTimeoutSec` 等高级 batch 子字段时才回退到 `edit-raw --set inputs.batch.concurrencyMode=...`。

### 4. Start 节点端口管理

`init` 默认使用 `--start-template minimal`，会自动创建标准评测集 7 个输出端口（`evaluationSet`、`evaluationSetVersion`、`managers`、`notify`、`notificationGroups`、`tag`、`ReportTemplateID`），**同时会多出一个空名端口，需手动删除**：

```bash
byteval workflow builder remove-port --sid "$SID" --node-id start_0 --side output --name ""
```

删除后 Start 节点有恰好 7 个标准端口，无需再用 `set-raw` 重置。

若子工作流只需部分端口，用 `remove-port` 逐个删除多余端口即可：

```bash
# 子工作流 Start 只保留 input_text，其余删掉
byteval workflow builder remove-port --sid "$CHILD_SID" --node-id start_0 --side output --name evaluationSet
# ...依此类推
```

**`set-raw` 中类型字段注意**：ASL 要求端口类型为整数（`1`=String、`2`=Number、`7`=Array 等），不接受字符串名称（`"string"` 会导致解析错误）。优先用 `add-port`/`remove-port` 操作端口，只在必要时使用 `set-raw`。

### 5. 工作流任务启动

CLI `workflow-task create` 在某些环境下返回 "missing require field content" 错误，无法确定正确的 body 结构。**建议直接在平台 UI 启动任务**，选择已上传的父工作流，填入评测集 ID 和版本后运行。

### 完整构建顺序

1. 创建并上传子工作流（先上传以获得真实 Workflow ID）
2. 用 `init` 创建父工作流 session（默认 `--start-template minimal --bare-end true`）
3. **删除 Start 节点的空名端口**：`remove-port --node-id start_0 --side output --name ""`
4. 添加 `GetEvaluationSubSetDetails`（appID=6970）节点：先 `app-schema`，再 `add-node`，然后 `bind` 三个必填入参
5. 创建/更新 Python 解析器应用，添加解析节点，绑定 `EvaluationSubSetDetails`
6. 用 `node-batch enable`（任意节点通用）开启批处理；Workflow 子流再补一次 `workflow-batch apply-schema` 反推 outputList
7. 用 `bind` 把节点入参引用到自身迭代变量端口；`node-batch inspect` 确认 `batch.inputLists` 和 `inputParameters` 结构
8. `validate` 到 0 error；`upload --description "..."` 上传（description 必填，名称不能与已有 workflow 重复）
9. 平台 UI 启动任务验证

## 与命令参考的关系

- 本文负责交互流程、复用策略和用户确认边界。
- `command-reference.md` 负责具体命令、参数、session 文件和 validate 覆盖范围。
- 执行阶段必须以 builder session 里的 `ir.json`、`dsl.json` 和 validate 输出为准。
