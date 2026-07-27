# 场景：查询应用

## Table of Contents

- [CLI 命令](#cli-命令)
- [执行步骤](#执行步骤)
- [平台链接生成规则](#平台链接生成规则)
- [注意事项](#注意事项)

## CLI 命令

```bash
# 按语义范围列出应用
byteval application list --project-id <project_id>   # 默认等价于 --scope tenant
byteval application list --scope owned --project-id <project_id>
byteval application list --scope tenant --project-id <project_id>
byteval application list --scope shared --project-id <project_id>
byteval application list --scope available [--user-email <email>]

# 通用过滤
byteval application list [--scope <owned|tenant|shared|available>] [--name <name>] [--access-type <http|...>] [--status <int>]

# 获取应用详情
byteval application get --id <id>
```

## 执行步骤

1. 先判断查询范围：
   - 未显式传 `--scope`：默认按 `tenant` 查询“本租户下所有应用”
   - 查“本租户内我自己的应用”：`--scope owned --project-id <project_id>`
   - 查“本租户下所有应用”：`--scope tenant --project-id <project_id>`
   - 查“共享到该租户的应用”：`--scope shared --project-id <project_id>`
   - 查“当前用户可用应用（本空间 + 共享）”：`--scope available`
2. 再按需追加 `--name`、`--access-type`、`--status` 等过滤条件
2. 记录目标应用 ID
3. 按需用 `application get --id <id>` 查看完整详情

## 平台链接生成规则

当用户明确希望“给我应用链接”“给我平台应用页”时，优先读取统一规则文档：

- `references/platform-link-rules.md`

本场景只需要额外记住两点：

- 先通过 `application list`、`application get` 或已有上下文拿到 `app_id`
- `project_id` 优先取当前命令显式传入的 `--project-id`，否则回退到本地默认项目 ID；若应用来自跨项目共享场景，页面是否可直接打开以平台实际行为为准

## 注意事项

- `--access-type` 可过滤接入类型，如 `http`
- `--scope owned` / `--scope available` 默认优先使用当前登录用户邮箱；取不到时再显式传 `--user-email`
- `--scope tenant` / `--scope shared` 默认补 `status=1`，如需覆盖可显式传 `--status`
- 若用户明确要平台链接，应优先直接返回应用页链接，不必只停留在 CLI 查询结果
