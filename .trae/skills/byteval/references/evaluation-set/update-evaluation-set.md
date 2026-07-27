# 场景：更新评测集

## CLI 命令

```bash
byteval evaluation-set update --evaluation-set-id <id> --file <path/to/data.csv>
```

## 执行步骤

1. 先用 `evaluation-set list` 或 `versions` 确认目标评测集 ID
2. 准备好新版本数据文件
3. 执行更新命令，成功后返回新的 `EvaluationSetVersionID`

## 注意事项

- 更新会创建一个新版本，历史版本保留
- 文件格式要求与创建评测集相同，参见 `references/evaluation-set/create-evaluation-set.md`
