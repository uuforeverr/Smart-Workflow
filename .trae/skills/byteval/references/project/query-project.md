# 场景：查询项目

## CLI 命令

```bash
# 列出项目
byteval project list

# 按 ID 筛选项目
byteval project list --project-id <id>

# 查看项目配置
byteval project config --key <key> [--key <key2> ...]

# 查看项目字段
byteval project field --material-type <text|...>
```

## 执行步骤

1. `byteval project list` 获取可用项目列表
2. 记录目标项目的 ID，按需用 `byteval config set project <id>` 设为默认
3. 如需查看项目的特定配置项，用 `project config --key <key>`

## 注意事项
- `--key` 可重复传多个，批量查询多个配置项
