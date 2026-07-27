# 场景：创建/更新应用

用户想把一个算法（Python 脚本、HTTP 服务、RPC 接口）接入 Byteval 评测体系，或对现有应用更新元数据/运行配置时。

## Table of Contents

- [场景路径](#场景路径)
- [常用命令](#常用命令)
- [元数据更新](#元数据更新)
- [Python 应用](#python-应用)
- [HTTP 应用](#http-应用)
- [RPC 应用](#rpc-应用)

## 场景路径

### Python / HTTP 应用

| 场景 | 路径 |
|------|------|
| 场景1：创建新应用 | `create python\|http` → `test python\|http` → `update python\|http [--status 1]` |
| 场景2：涉及逻辑改动的更新 | `get --view update-body` → `test python\|http` → `update python\|http [--status 1]` |

### 元数据更新

推荐路径：

```bash
# 第一步：先查看当前应用信息
byteval application get --project-id <project_id> --id <app_id> --view summary

# 第二步：优先用 selective update 只改显式字段
byteval application update --project-id <project_id> --id <app_id> \
  --name "新名称" \
  --managers '["a@bytedance.com","b@bytedance.com"]' \
  --concurrency 10 --qps 2.5 --qpm 100 --timeout 30 \
  --share-project-ids '[12,34]'

# 第三步：如果 CLI flags 无法覆盖目标字段，再导出 update-body 到文件
byteval application get --project-id <project_id> --id <app_id> --view update-body > update.json

# 第四步：手动修改 update.json 后回传
byteval application update --project-id <project_id> --body-file update.json
```

说明：

- selective update 只覆盖显式传入字段，未传字段保持原值
- 复杂结构字段若 CLI 暂未覆盖，再回退到 `update-body` 文件流，避免默认就让 Agent 直接改整包 JSON

### RPC 应用

| 场景 | 路径 |
|------|------|
| 场景1：创建新应用 | `create rpc` → `rpc list-methods --psm <psm>` → `update rpc --psm <psm> --method <method>` → `test rpc --body-param <template>` |
| 场景2：更新绑定的方法 | `update rpc --psm <new_psm> --method <new_method>` → `test rpc --body-param <template>` |

> `update --status` 默认为 1（直接发布），保存草稿传 `--status 0`。RPC Input/Output schema 全程自动拉取，无需手动填写。

## 常用命令

```bash
# 创建
byteval application create python --project-id <project_id> --name <name> --description <desc>
byteval application create http   --project-id <project_id> --name <name> --description <desc>
byteval application create rpc    --project-id <project_id> --name <name> --description <desc>

# 查看视图
byteval application get --project-id <project_id> --id <app_id> --view summary
byteval application get --project-id <project_id> --id <app_id> --view raw

# Python 调试
byteval application test python --project-id <project_id> --id <app_id> \
  --code-file <sink.py> --input-example '{"a":1}' --output-example '{"sum":1}' --body-param '{"a":1}'

# Python 更新并发布（默认 status=1）
byteval application update python --project-id <project_id> --id <app_id> \
  --code-file <sink.py> --input-example '{"a":1}' --output-example '{"sum":1}'

# Python 更新并保存草稿
byteval application update python --project-id <project_id> --id <app_id> \
  --code-file <sink.py> --input-example '{"a":1}' --output-example '{"sum":1}' --status 0

# HTTP 更新并发布
byteval application update http --project-id <project_id> --id <app_id> \
  --url <url> --method POST --body-example '{"key":"val"}' --output-example '{"result":"ok"}'

# HTTP 测试
byteval application test http --project-id <project_id> --id <app_id> \
  --query-param '{"p1":"v1"}' --body-param '{"key":"val"}' --format json

# RPC 发现
byteval application rpc list-versions --project-id <project_id> --psm <psm>
byteval application rpc list-methods  --project-id <project_id> --psm <psm> [--version <version>]

# RPC 更新并发布
byteval application update rpc --project-id <project_id> --id <app_id> \
  --psm <psm> --method <method>

# RPC 测试（body-param 模板由 update rpc 输出提供）
byteval application test rpc --project-id <project_id> --id <app_id> \
  --body-param '<json>' --format json

# 只改元数据（selective update）
byteval application update --project-id <project_id> --id <app_id> \
  --name "新名称" \
  --managers '["a@bytedance.com","b@bytedance.com"]' \
  --concurrency 10 --qps 2.5 --qpm 100 --timeout 30 \
  --share-project-ids '[12,34]'

# 低层兜底：直接传 JSON
byteval application create --project-id <project_id> --body-file <app.json>
byteval application test   --project-id <project_id> --body-file <test.json>
byteval application update --project-id <project_id> --body-file <update.json>
```


## 元数据更新

`application update --id ...` 支持 updateSelective 语义：仅覆盖显式传入的字段，未传字段保持原值。当前已支持：

- `--name`
- `--managers`：JSON 字符串数组，语义为整组替换，例如 `["a@bytedance.com","b@bytedance.com"]`
- `--concurrency`
- `--qps`
- `--qpm`
- `--timeout`
- `--share-project-ids`：JSON 数字数组，语义为整组替换，例如 `[12,34]`

`--managers '[]'` 和 `--share-project-ids '[]'` 会清空对应字段。复杂结构（如 `Input/Output`、`HttpInfo`、`RpcInfo`、标签等）仍建议通过 `get --view update-body` + `update --body-file` 兜底。

### 常用枚举

- `ModalTypes`：`文本` / `图像` / `视频` / `音频` / `数据文件`
- `BusinessTags`：`安全` / `通用` / `剪C` / `AI4SE` / `内容创作` / `智能问答` / `智能客服` / `内容安全与风控` / `营销` / `Agent` / `数字人`
- `AbilityTags`：`数据生成` / `评测` / `打标` / `打分` / `物料生产` / `算法结果生成` / `通用能力`
- `CCApplicationType`：当前已验证用 `0`
- `Interpreter`：Python 当前已验证用 `1`

## Python 应用

### Python 模板

函数名固定为 `sink`，入参与返回值都必须是 `dict`，`FuncContent` 不能留空。

```python
# 当前方法名只能为sink，入参和出参都必须为dict，请不要更改。并注意：不要进行打印
# 适用于简单的数据处理，解析输入的字典数据，构造需要的字典等。
# 如需引入第三方库、进行复杂的逻辑处理、调用其他服务接口等建议使用faas实现，然后新建RPC/HTTP应用接入该faas服务即可。
# 有任何疑问请联系平台管理员。
def sink(data: dict) -> dict:
    # TODO: 根据实际业务逻辑处理 data
    result = {}
    return result
```

### 调试示例

```bash
# 第一步：新建应用
byteval application create python \
  --project-id <project_id> \
  --name "文本情感分类" \
  --description "输入文本，输出情感标签与置信度"

# 第二步：调试代码逻辑（--body-param 为实际运行时入参）
byteval application test python \
  --project-id <project_id> \
  --id <app_id> \
  --code-file sink.py \
  --input-example '{"text":"hello"}' \
  --output-example '{"label":"positive","score":0.9}' \
  --body-param '{"text":"hello"}' \
  --format json

# 第三步：调试通过后，更新代码/schema 并发布
byteval application update python \
  --project-id <project_id> \
  --id <app_id> \
  --code-file sink.py \
  --input-example '{"text":"hello"}' \
  --output-example '{"label":"positive","score":0.9}'

# 第四步：发布成功后，按 platform-link-rules.md 规则生成并输出算子链接
# 应用链接格式（BOE 站点换 lsep-boe.bytedance.net）：
# https://byteval.bytedance.net/project/<project_id>/manageApplication/edit?from=AppListPage&id=<app_id>
```

## HTTP 应用

### 参数 location

| 参数位置 | update flag | test flag | Location 值 |
|---------|-------------|-----------|------------|
| Query string | `--query-example` | `--query-param` | 1 |
| Request body | `--body-example` | `--body-param` | 2 |
| Header（可选，无默认值） | `--header-example` | `--header-param` | 3 |
| Header（必填，有默认值） | `--preset-header Key=Value` | — | 3，IsRequired=true |

`Content-Type: application/json` 自动注入为必填 header，无需手动指定；传 `--preset-header Content-Type=text/plain` 可覆盖。

### 调试示例

```bash
# 第一步：新建应用（可一步注入 URL 和 Method）
byteval application create http \
  --project-id <project_id> \
  --name "情感分析 HTTP" \
  --url "https://9l07rubs.fn.bytedance.net/mos" \
  --method GET

# 第二步：测试接口是否可用
byteval application test http \
  --project-id <project_id> \
  --id <app_id> \
  --query-param '{"p1":"val1"}' \
  --body-param '{"b1":"val1","b2":"val2"}' \
  --format json

# 第三步：更新参数 schema 并发布
byteval application update http \
  --project-id <project_id> \
  --id <app_id> \
  --url "https://9l07rubs.fn.bytedance.net/mos" \
  --method GET \
  --query-example '{"p1":"","p2":""}' \
  --body-example '{"b1":"","b2":""}' \
  --header-example '{"h1":""}' \
  --output-example '{"result":"ok"}'

# 如需覆盖 Content-Type 或新增其他必填 header（可重复）
byteval application update http \
  --project-id <project_id> \
  --id <app_id> \
  --preset-header "Authorization=Bearer <token>" \
  --preset-header "Content-Type=application/x-www-form-urlencoded" \
  --output-example '{"result":"ok"}'
```

## RPC 应用

RPC 应用的 Input/Output schema 由平台从 PSM 注册表自动拉取，无需手动指定参数列表。`update rpc` 命令会自动完成：版本解析 → EndpointID 解析 → schema 拉取 → 写入应用，并在 text 输出末尾给出 `test rpc` 可直接使用的 body-param 模板。

### 发现命令

```bash
# 查询 PSM 的可用版本列表
byteval application rpc list-versions \
  --project-id <project_id> \
  --psm ies.efficiency.byteval_server

# 查询某版本下的可用方法列表（不传 --version 默认取最新版本）
byteval application rpc list-methods \
  --project-id <project_id> \
  --psm ies.efficiency.byteval_server \
  --version <version>
```

### 调试示例

```bash
# 第一步：新建 RPC 应用
byteval application create rpc \
  --project-id <project_id> \
  --name "GetItem 接口"

# 第二步（可选）：查询可用方法，确认方法名
byteval application rpc list-methods \
  --project-id <project_id> \
  --psm ies.efficiency.byteval_server

# 第三步：更新 schema 并发布（自动拉取最新版本 + 指定方法的 Input/Output）
# 输出末尾会附带 test rpc 的 body-param 模板
byteval application update rpc \
  --project-id <project_id> \
  --id <app_id> \
  --psm ies.efficiency.byteval_server \
  --method GetItem

# 第四步：用模板测试（从 update rpc 输出复制 body-param，填入实际值）
byteval application test rpc \
  --project-id <project_id> \
  --id <app_id> \
  --body-param '{"name":"example","page":0,"pageSize":10}' \
  --format json

# 如需指定特定版本（默认取最新）
byteval application update rpc \
  --project-id <project_id> \
  --id <app_id> \
  --psm ies.efficiency.byteval_server \
  --method GetItem \
  --version <version>

# 保存草稿（不发布）
byteval application update rpc \
  --project-id <project_id> \
  --id <app_id> \
  --psm ies.efficiency.byteval_server \
  --method GetItem \
  --status 0

# 如果忘了 schema，可重新查看 input 模板
byteval application get --project-id <project_id> --id <app_id> --view input-schema
```

### update rpc 参数说明

| Flag | 必填 | 说明 |
|------|------|------|
| `--id` | ✓ | 应用 ID |
| `--psm` | ✓ | 服务 PSM，如 `ies.efficiency.byteval_server` |
| `--method` | ✓ | RPC 方法名，如 `GetItem` |
| `--version` | | 服务版本，不传则自动取最新版本 |
| `--idc` | | IDC（可选） |
| `--cluster` | | 集群（可选） |
| `--status` | | 发布状态：1=发布（默认），0=草稿 |
| `--project-id` | | 不传则用本地配置的 ProjectID |
