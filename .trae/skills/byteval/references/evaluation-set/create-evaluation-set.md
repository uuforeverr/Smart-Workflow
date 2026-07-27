# 场景：创建评测集

## CLI 命令

```bash
byteval evaluation-set create --name "<名称>" --desc "<描述>" --file <path/to/data.csv>
```

## 执行步骤

1. 准备好数据文件（支持 `.csv` / `.jsonl`）
2. 确认文件路径存在
3. 执行创建命令，成功后返回 `EvaluationSetID` 和 `EvaluationSetVersionID`

## CSV 格式要求与常见坑

- 必须有表头，每行一条样本
- 字段包含逗号/换行时必须用双引号包裹：`"你好, 世界",positive`
- 字段内容本身含双引号 `"` 时，必须转义为连续两个双引号 `""`
  - 例：想表达 `{"a":1}` → CSV 中写 `"{""a"":1}"`
- 典型报错：`csv 文件格式不合法：parse error ... extraneous or missing " in quoted-field`
  → 原因：引号未转义或不配对

## 注意事项

- 创建是异步流程，命令成功返回 ID 即视为提交成功，无需额外校验或预览
- `--desc` 建议填写，便于后续识别
