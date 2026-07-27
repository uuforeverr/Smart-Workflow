# 场景：创建/更新工作流

## CLI 命令

```bash
# 创建工作流
byteval workflow create --body-file <workflow.json>

# 更新工作流（以 -h 确认是否需要 --id）
byteval workflow update --body-file <workflow.json> [--id <id>]
```

## 执行步骤

1. 准备工作流定义 JSON 文件
2. 不确定字段结构时，先执行 `byteval workflow create -h` 确认参数
3. 执行创建/更新命令，记录返回的工作流 ID

## Agent Builder

当用户要“从自然语言组装/修改工作流”而不是已经提供完整 JSON body 时，优先使用内置 Builder：

```bash
byteval workflow builder init --name "<name>" --project-id <project_id>
byteval workflow builder add-node --sid <sid> --kind application --app-id <app_id>
byteval workflow builder scope --sid <sid> --node-id <node_id> --port <input_port>
byteval workflow builder bind --sid <sid> --node-id <node_id> --port <input_port> --mode ref --source-node <source_node> --source-port <source_port>
byteval workflow builder validate --sid <sid>
byteval workflow builder preview --sid <sid>
byteval workflow builder upload --sid <sid>
```

总览见 `references/workflow/builder/README.md`；自然语言需求澄清、workflow 复用和上传草稿策略见 `references/workflow/builder/interaction-guide.md`；具体命令和 validate 规则见 `references/workflow/builder/command-reference.md`。

## 注意事项

- 优先使用 `--body-file` 传参，避免 JSON 转义/换行问题
- 更新时是否需要 `--id` 以实际 `-h` 输出为准
- Builder 已内置到 `byteval` Go 二进制，无需再执行旧版 `python3 -m scripts.cli`
