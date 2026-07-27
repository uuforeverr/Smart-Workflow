# 场景：查询评测集

## Table of Contents

- [CLI 命令](#cli-命令)
- [执行步骤](#执行步骤)
- [平台链接生成规则](#平台链接生成规则)
- [注意事项](#注意事项)

## CLI 命令

```bash
# 搜索评测集列表
byteval evaluation-set list --keyword <关键词>

# 查看评测集版本列表
byteval evaluation-set versions --evaluation-set-id <id>

# 查看评测集详情（预览数据）
byteval evaluation-set detail --evaluation-set-id <id> --format md
```

## 执行步骤

1. `byteval evaluation-set list --keyword <关键词>` 搜索目标评测集
2. 从结果中获取 `evaluation-set-id`
3. 按需执行 `versions` 查看历史版本，或 `detail` 预览数据内容

## 平台链接生成规则

当用户明确希望“给我评测集链接”“给我平台页链接”时，优先读取统一规则文档：

- `references/platform-link-rules.md`

本场景只需要额外记住两点：

- 先通过 `evaluation-set list`、`evaluation-set detail` 或已有上下文拿到 `evaluation_set_id`
- `project_id` 优先取当前命令显式传入的 `--project-id`，否则回退到本地默认项目 ID

## 注意事项

- `--keyword` 支持模糊搜索
- `detail` 命令使用 `--format` 指定输出格式（例如 `--format md`）
- 若用户明确要平台链接，应优先直接返回评测集页链接，不必只停留在 CLI 查询结果
