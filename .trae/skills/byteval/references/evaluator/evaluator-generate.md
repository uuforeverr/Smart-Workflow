# 场景：创建场景级评估器

## Table of Contents

- [适用范围](#适用范围)
- [能力来源](#能力来源)
- [前置检查](#前置检查)
- [执行步骤](#执行步骤)
- [CLI 执行协议](#cli-执行协议)
- [输出结构](#输出结构)
- [生成物验收](#生成物验收)
- [统一 runner 约定](#统一-runner-约定)
- [后续接入建议](#后续接入建议)
- [错误处理](#错误处理)
- [注意事项](#注意事项)

## 适用范围

当用户已经有评测标准，并希望把一个业务场景生成可运行的 evaluator skill bundle 时，使用本页。

典型触发：

- “根据这套标准创建评估器”
- “生成 evaluator skill”
- “把这些评测维度合成一个评估器”
- “一个场景多个维度，帮我生成统一 runner”

若用户还没有评测标准，先读取 `references/evaluation-metric/evaluation-metric-recommend.md` 推荐标准。

## 能力来源

本流程融合自现有 `evaluator-creator` skill。核心约定：

- 一个业务场景只生成一个 skill 目录
- 多个评估维度共用一个 `scripts/runner.py`
- 每个维度有独立 judge prompt 和 sample
- `SKILL.md` 与 `skill.yaml` 必须体现“单 runner、多维并发”的场景级结构
- 模型配置不写入样本数据，统一通过 CLI 参数或环境变量传入

当前能力已内置到 `byteval evaluator generate`。执行时优先使用 CLI 命令，不再依赖外部 `evaluator-creator` skill 目录。CLI 内部已复用原 skill 的核心约定：请求 prompt 生成接口、必要时对内联 JSON 标准做本地 fallback、物化场景级 skill 目录和统一 runner。

## 前置检查

执行前确认这些参数：

| 参数 | 必填 | 说明 |
|---|---|---|
| `--scene-name` | 是 | 业务场景名称，用于 skill 目录名、skill 标题和输出摘要 |
| `--eval-standard` | 二选一 | 评测标准 JSON 字符串 |
| `--eval-standard-url` | 二选一 | 评测标准文档链接 |
| `--data-url` | 否 | 评测数据链接，默认空字符串 |
| `--max-score` | 否 | 评分上限，默认 `5` |
| `--user-requirement` | 否 | 用户补充要求，默认空字符串 |
| `--output-dir` | 否 | 输出目录，默认 `/tmp/gen_eval_atom_output` |
| `--timeout` | 否 | 生成超时秒数，建议 300 秒以上 |

关键校验：

- `--scene-name` 缺失时先补问
- `--eval-standard` 与 `--eval-standard-url` 必须且只能提供一个
- 两者都缺失时，先转到标准推荐或向用户要标准
- 两者同时提供时，应让用户选择一种来源

## 执行步骤

1. 确认场景名和评测标准来源。
2. 若只有自然语言需求，先用 `references/evaluation-metric/evaluation-metric-recommend.md` 生成标准。
3. 确认当前 `byteval` 已包含 `evaluator generate` 子命令。
4. 执行生成命令。
5. 从 JSON 输出中记录：
   - `skill_dir`
   - `generated_evaluator_count`
   - `manifest`
   - `summary`
6. 检查生成目录结构与关键文件。
7. 返回给用户：生成目录、维度数量、摘要文件、后续如何运行。

## CLI 执行协议

使用内联评测标准生成：

```bash
byteval evaluator generate \
  --scene-name "<scene-name>" \
  --eval-standard '<eval-standard-json>' \
  --data-url "<data-url>" \
  --max-score 5 \
  --user-requirement "<user-requirement>" \
  --output-dir /tmp/gen_eval_atom_output \
  --timeout 300 \
  --json
```

使用评测标准文档链接生成：

```bash
byteval evaluator generate \
  --scene-name "<scene-name>" \
  --eval-standard-url "<standard-url>" \
  --data-url "<data-url>" \
  --max-score 5 \
  --output-dir /tmp/gen_eval_atom_output \
  --timeout 300 \
  --json
```

说明：

- 默认接口：`https://bv9euxsg.fn.bytedance.net/api/gen_eval_atom`
- `--endpoint` 可显式覆盖完整接口地址
- 全局 `--json` 或命令级 `--format json` 会返回结构化生成结果
- 若使用 `--eval-standard` 且远端接口不可用，CLI 会按标准 JSON 在本地生成基础 prompt 和完整目录；若使用 `--eval-standard-url`，接口不可用时不能 fallback

## 输出结构

成功后输出目录应符合场景级 skill 结构：

```text
<output_dir>/
├── generated_skill_manifest.json
├── generated_skill_payload.json
├── gen_eval_atom_summary.md
└── <scene-name-slug>/
    ├── SKILL.md
    ├── skill.yaml
    ├── scripts/
    │   └── runner.py
    ├── assets/
    │   ├── <评估器名>_judge_prompt.md
    │   └── ...
    └── fixtures/
        ├── sample.json
        ├── <评估器名>_sample.json
        └── ...
```

关键文件含义：

- `generated_skill_manifest.json`：生成摘要，便于 Agent 快速读取路径和维度
- `generated_skill_payload.json`：完整生成结果
- `gen_eval_atom_summary.md`：人类可读摘要
- `<scene-name-slug>/SKILL.md`：生成出的场景级 skill 说明
- `<scene-name-slug>/skill.yaml`：场景级结构化配置
- `<scene-name-slug>/scripts/runner.py`：唯一执行入口
- `<scene-name-slug>/assets/*_judge_prompt.md`：各维度 prompt
- `<scene-name-slug>/fixtures/*_sample.json`：各维度样例
- `<scene-name-slug>/fixtures/sample.json`：场景级样例索引

## 生成物验收

生成后至少检查：

- 只生成一个场景级 skill 目录，而不是每个维度一个独立 skill
- `scripts/runner.py` 存在，且所有维度都指向它
- `SKILL.md` 使用中文，说明目标、目录结构、运行方式、输入输出、维度关系
- `skill.yaml` 包含场景名、场景 ID、维度列表、prompt/sample 路径、runtime contract
- `skill.yaml` 允许的 provider 只有 `ark` 和 `modelhub`
- 样例数据不包含 `provider`、`model`、`api_key`
- 每个维度都有自己的 prompt 文件和 sample 文件

建议快速读取 manifest：

```bash
cat /tmp/gen_eval_atom_output/generated_skill_manifest.json
```

## 统一 runner 约定

生成出的 `scripts/runner.py` 应支持：

```bash
python3 scripts/runner.py --input '<json>' --provider ark --model <model> --api-key <api-key>
python3 scripts/runner.py --input /path/to/batch.json --provider ark --model <model> --api-key <api-key>
python3 scripts/runner.py --input /path/to/batch.csv --provider ark --model <model> --api-key <api-key>
python3 scripts/runner.py --input /path/to/batch.csv --output /tmp/custom_eval.json --provider ark --model <model> --api-key <api-key>
```

输入支持：

- 单条 JSON object
- 指向 `.json` 或 `.csv` 文件的路径
- 批量 JSON array
- 包含 `rows` 字段的 JSON object
- CSV 行记录

每条输入记录建议包含：

```json
{
  "inputs": {
    "业务字段": "业务值"
  },
  "urls": []
}
```

输出约定：

- 单条输入返回：
  - `success`
  - `dimension_results`
  - `success_count`
  - `failure_count`
- 批量输入返回：
  - `results`
  - `success_count`
  - `failure_count`

`dimension_results` 每项至少包含：

- `dimension`
- `skill_id`
- `prompt_file`
- `input_fields`
- `result`

未显式传 `--output` 时，默认写到：

```text
/tmp/eval_result/<评测场景>_评测结果.json
```

## 后续接入建议

生成出的 evaluator skill 是本地评估器 skill bundle，不等同于 Byteval 平台上的 application 或 workflow。

如果用户要继续接入 Byteval 平台：

1. 先确认要接入的形态：
   - 作为本地 skill 运行评估
   - 封装成 Python / HTTP / RPC application
   - 放进 workflow 执行
2. 若要创建平台 application，读取 `references/application/create-update-app.md`
3. 若要组装 workflow，读取 `references/workflow/create-update-workflow.md` 和 `references/workflow/builder/README.md`

## 错误处理

- 缺 `scene-name`：先补问
- 缺标准来源：先推荐标准或向用户要标准
- 同时传 `eval-standard` 和 `eval-standard-url`：让用户选一种
- 生成命令失败：原样转述错误
- 没有实际执行 CLI 时，不要报告任何生成路径

## 注意事项

- 不要把模型 API key 写入样例、manifest 或 skill 文档
- 不要复制旧脚本中的硬编码凭据；运行时凭据只通过生成物 runner 的 CLI 参数或环境变量提供
- `eval-standard` 建议传合法 JSON，避免自然语言标准在 shell 中被错误转义
- 大段 JSON 标准优先写入临时文件后再安全地作为参数传入，避免 shell 转义错误
- 生成过程可能依赖网络和内部服务；只有内联 `--eval-standard` 支持本地 fallback，其他失败不要用手写文件冒充生成结果
