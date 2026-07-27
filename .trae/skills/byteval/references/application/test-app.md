# 场景：测试已发布的应用

对已发布的应用发送请求验证其运行结果。

## CLI 命令

```bash
# 第一步：查看入参模板
byteval application get --id <id> --view input-schema

# 第二步：填入实际值后测试
byteval application test --id <id> --body-param '<json>' --json
```

## 注意事项

- `--json` 等价于 `--format json`，输出完整结构化结果
- 低层 `--body-file` 中的 `AppItem` 若来自 `application get` 返回，需去掉 `StatisticInfo` 字段
