---
name: byteval
description: "用于在用户需要对 Byteval 平台资源执行真实操作时触发；当用户提到 byteval，并希望“查询/列出/有哪些/查看详情/创建/更新/启动/暂停/恢复/调试”等项目、应用、算子、评测集、工作流、工作流任务、登录与配置相关操作时，即使用户只说“查一下 Byteval 相关资源/工作流”，也必须使用此技能获取真实结果并整理返回。默认通过 `byteval` cli工具执行。"
---

# Byteval 平台操作（Agent 执行型）

你是"执行器"，不是"说明书"。当用户提出"在 Byteval 上帮我做 X"时，直接操作 Byteval 平台完成对应任务，并把结果整理返回；默认通过 `byteval` 执行。


## 登录引导

### 1. 检查 CLI 是否可用

```bash
command -v byteval && byteval version && byteval update
```

若 CLI 可用，`byteval update` 会自动拉取并替换为最新版本；若不存在，执行远程安装脚本（自动探测平台，安装到 `~/.local/bin`）：

```bash
curl -fsSL https://lf0-fast-deliver-inner.bytedance.net/obj/byteval-common/byteval_cli/release/install.sh -o /tmp/byteval-install.sh && bash /tmp/byteval-install.sh && export PATH="$HOME/.local/bin:$PATH"
```

### 2. 检查登录态与环境配置

```bash
byteval auth login           # 优先执行：已登录则直接返回摘要；未登录则返回 login_url
```

最简流程（优先执行，不要预先跑一堆检查命令）：

1) 先执行 `byteval auth login`

- 若输出包含 `login_url`：
  - 把 `login_url` 展示给用户，提示用户在浏览器完成授权后回复“已完成授权”
  - **停止执行**（不要自动轮询/不要提前执行 `--complete`）
  - 用户确认后执行一次：

```bash
byteval auth login --complete
```

- 若返回 `pending`：提示用户继续完成授权，等用户确认后再重试一次（不要循环刷）
- 若输出直接包含 `email/organization/site/projectID/projectName`：说明已登录，继续下一步

2) 检查是否已选择默认项目

根据 `auth login` 输出里的 `projectID` 判断：

- 若 `projectID` 非空：说明 project 已就绪，直接进入后续具体任务
- 若 `projectID` 为空：需要引导用户选择 project

3) 选择并设置 project（只在 project 为空时执行）

```bash
byteval project list
```

- 优先策略：根据 `auth login` 输出里的 `organization` 关键词，在项目列表的 `NAME` 中找最接近的 2~3 个候选，给用户选择
- 兜底策略：若没有明显匹配，优先建议 `通用项目`、`体验项目`（若存在）；否则让用户给一个关键字再筛（例如“byteval”“demo”“评测”）
- 无权限兜底：如果用户没有任何有权限的项目，引导用户去平台(https://byteval.bytedance.net/)申请`体验租户(13)`的权限
- 用户确认后执行：

```bash
byteval config set project <project_id>   # 会额外输出 projectName
```

4) 当“已登录 + project 已设置”同时满足后，直接执行用户的后续具体任务

补充说明：

- 若用户需切换站点：`byteval config set site <boe|cn|sg>` 然后再执行 `byteval auth login`（默认站点是CN）


### 3. 识别资源与动作，查阅场景文档执行

- 资源：`auth` / `config` / `project` / `evaluation-set` / `evaluation-metric` / `evaluator` / `evaluation-report` / `application` / `workflow` / `workflow-task` / `algo`（算法评测：model / benchmark / dataset / task / task-group / leaderboard）
- 动作：查询（list/detail/get）/ 创建（create）/ 更新（update）/ 运行控制（start/pause/resume/retry/rerun-from-node/debug）
- 根据场景查阅下方索引中的参考文档，执行命令并返回结果

## 行为准则

- **先确认可执行与已登录**：第一步检查 `byteval` 是否已安装且可运行，并用 `byteval auth status` 确认登录态（必要时再检查 `config get site` / `config get project`）
- **优先执行**：用户意图明确时直接跑命令，不要只给命令示例
- **先查后改**：用户没给 ID 时，先 `list` 定位，再执行写操作
- **新增/更新先确认**：凡是对 Byteval 资源执行新增、更新、启动、暂停、恢复、调试等会改变平台状态的操作，都必须先和用户确认，再真正执行
- **写操作参数优先用文件**：需要请求体时优先 `--body-file`（避免 JSON 转义/换行问题）；stdin 用 `--body-file -`
- **用户无感 CLI**：默认只输出“平台视角”的结果与结论，不暴露 CLI 命令/flag；仅在用户明确要求复现或排障时，再附上执行细节
- **创建/发布资源后主动返回平台链接**：应用（算子）、评测集、工作流等资源创建或发布成功后，无需用户额外要求，主动按 `references/platform-link-rules.md` 的规则生成对应平台链接并附在结果中返回；用户明确要求其他资源链接时同理直接给出
- **登录需用户交互时要停住**：检测到未登录就立刻给出 `login_url` 并等待用户确认"已完成授权"，不要自动轮询/不要提前执行 `--complete`
- **算法空间（Project 33）任务语义先澄清**："任务"一词在不同上下文有不同含义，按下面规则处理：
  - **当前 `projectID == 33` 或用户明确提到"33 空间/项目"时**：直接把"任务"理解为 **Merlin Task / Task Group**（评测任务，绑定 model + benchmark + dataset），走 `byteval algo task ...` / `byteval algo task-group ...`，无需再和用户确认
  - **当用户提到"算法"、Merlin、Benchmark、Leaderboard 等关键词，但当前并不在 33 空间**：此时"任务"存在歧义，**先和用户确认"你说的任务是 Merlin 算法任务还是工作流任务"**，再决定走 `algo task` / `algo task-group` 还是 `workflow-task`，不要混用
  - **其他普通项目下**：`byteval workflow-task ...` 指工作流任务（workflow 的运行实例）
  - Merlin 任务相关数据集请走 `algo dataset ...`，不要和普通 `evaluation-set ...` 命令混用


## 平台概览

Byteval 是字节跳动大模型评测平台，核心资源模块如下：

| 模块 | 说明 |
|---|---|
| `auth` / `config` | 登录认证与环境配置（站点、默认项目） |
| `project` | 评测项目管理 |
| `evaluation-set` | 评测集（数据集）管理（普通项目，本地文件上传） |
| `evaluation-metric` | 评测标准、评测维度推荐 |
| `evaluator` | 基于评测标准生成场景级评估器 skill |
| `evaluation-report` | 根据本地 CSV 评测结果生成飞书文档报告或 HTML 报告 |
| `application` | 被评测的应用（算子）管理 |
| `workflow` | 评测工作流管理 |
| `workflow-task` | 工作流任务（运行实例）管理 |
| `algo` | 算法评测专用域（默认聚焦 Project 33）：模型、Benchmark、数据集、Merlin 任务/任务组、排行榜 |


## 场景索引

根据用户意图，读取对应参考文档获取命令与操作经验：

### 认证与配置

| 场景 | 参考 |
|---|---|
| 登录 / 查看登录状态 | 见本页“登录引导” |
| 配置站点与默认项目 | 见本页“登录引导”与 `byteval config -h` |

### 项目

| 场景 | 参考文档 |
|---|---|
| 查询项目列表、项目配置 | `references/project/query-project.md` |

### 评测集

| 场景 | 参考文档 |
|---|---|
| 搜索评测集、查看版本与详情 | `references/evaluation-set/query-evaluation-set.md` |
| 创建评测集（含 CSV 格式经验） | `references/evaluation-set/create-evaluation-set.md` |
| 更新评测集 | `references/evaluation-set/update-evaluation-set.md` |

### 评测标准与评估器

| 场景 | 参考文档 |
|---|---|
| 推荐评测标准 / 评测维度 | `references/evaluation-metric/evaluation-metric-recommend.md` |
| 推荐可复用评估器 / 指标列表 | `references/evaluator/evaluator-recommend.md` |
| 基于评测标准生成场景级评估器 skill | `references/evaluator/evaluator-generate.md` |

### 评测报告

| 场景 | 参考文档 |
|---|---|
| 根据本地 CSV 评测结果生成报告（SCORE/LABEL/GSB/MULTI） | `references/evaluation-report/intelligent-evaluation-report-generation.md` |

### 应用

| 场景 | 参考文档 |
|---|---|
| 查询应用列表与详情 | `references/application/query-app.md` |
| 创建/更新应用（含 Python/HTTP/RPC 模板、Python 调试/保存/发布流程） | `references/application/create-update-app.md` |
| 测试应用 | `references/application/test-app.md` |

### 算法评测（algo，主要服务 Project 33）

> ⚠️ 任务语义判定：
> - 当前 `projectID == 33` 或用户提到"33 空间/项目"时，"任务"直接按 Merlin Task / Task Group 处理（`algo task` / `algo task-group`），无需再和用户确认
> - 用户只提到"算法"、Merlin、Benchmark、Leaderboard 但不在 33 空间时，**先和用户确认是 Merlin 算法任务还是工作流任务**，不要混用

| 场景 | 命令入口（详见仓库 README `### algo` 章节） |
|---|---|
| 模型管理（list/create/update/delete） | `byteval algo model ...` |
| Benchmark 管理与版本 | `byteval algo benchmark ...` |
| 算法数据集（评测集）查询 / 创建 / 更新 | `byteval algo dataset list|detail|versions|create|update`（支持本地文件、HDFS、完整 JSON 与 selective flag 两种模式） |
| 创建 Merlin 算法任务（绑定 model+benchmark+dataset） | `byteval algo task create` |
| 查看任务报告 / 任务配置 | `byteval algo task report` / `byteval algo task config` |
| 任务组列表与详情（按创建人/状态等筛选） | `byteval algo task-group list|detail` |
| 排行榜：Benchmark / 模型 / 指标视角 | `byteval algo leaderboard benchmarks|models|metrics` |

常用经验：

- 在算法空间，**"任务"默认指 Merlin Task Group**，写操作前必须和用户确认是 Merlin 任务还是工作流任务
- `algo dataset create/update` 与 `evaluation-set create/update` 是两条独立链路：前者支持完整 JSON / HDFS / FileStoreInfo 等高级场景；后者只支持本地 csv/tsv/jsonl/xlsx 自动推断 Features 上传
- `algo dataset update --update-type=2`（默认）必须同时携带 `--version-id`，否则后端会报"测试集版本已被删除"
- `algo task-group list` 支持按创建人筛选（`--creator`），定位 Merlin 任务组时优先用此 flag

### 工作流与任务

| 场景 | 参考文档 |
|---|---|
| 查询工作流列表与详情 | `references/workflow/query-workflow.md` |
| 创建/更新工作流 | `references/workflow/create-update-workflow.md` |
| Agent 分步构建/编辑工作流 DSL（总览） | `references/workflow/builder/README.md` |
| 自然语言需求到 Builder 执行流程 | `references/workflow/builder/interaction-guide.md` |
| Builder 子命令与 validate 规则 | `references/workflow/builder/command-reference.md` |
| 启动工作流（创建任务并启动） | `references/workflow/start-workflow.md` |
| 查看工作流进度（任务状态、节点信息） | `references/workflow/check-workflow-progress.md` |
| 暂停/恢复工作流任务 | `references/workflow/pause-resume-workflow.md` |
| 重试任务 / 从节点重新执行工作流任务 | `references/workflow/retry-rerun-workflow.md` |
| 调试工作流 | `references/workflow/debug-workflow.md` |
| 分析工作流结果并出具报告 | `references/workflow/analyze-workflow-result.md` |

### 平台链接

| 场景 | 参考文档 |
|---|---|
| 用户希望获取任务页 / 报告页 / 评测集页 / 应用页等平台链接 | `references/platform-link-rules.md` |


## 输出约定

- 默认只给结果摘要（按平台概念组织：命中数量、关键字段、ID/状态等），避免让用户感知到底层是 CLI
- 仅当用户明确要求复现或排障时，再补充“执行细节”（例如实际执行的 `byteval ...` 命令/关键参数/关键返回信息）
- 查询类：返回"可用的 ID 列表/命中数量/关键字段"，并提示下一步可做什么
- 写/控制类：返回"创建/更新/启动"的目标 ID、状态，并说明后续如何追踪进度/查看详情
