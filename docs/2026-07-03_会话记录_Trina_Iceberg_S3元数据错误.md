# 2026-07-03 会话记录：Trino Iceberg S3 元数据访问失败

## 用户问题

执行 SQL 查询 `iceberg.lake_dwd.dwd_trade_order_detail_wide` 时失败：

```
TrinoQueryError(type=INTERNAL_ERROR, name=GENERIC_INTERNAL_ERROR,
message="Failed to get status for file: s3://lecoo-iceberg/lake_dwd/dwd_trade_order_detail_wide/metadata/snap-252235777549068781-1-d78b3c4f-c3e1-4678-8d22-042f7b288b4d.avro")
```

触发位置：`/opt/lecoo_sql_ai/app/apps/chat/task/llm.py` → `execute_sql` → `exec_sql`

## 结论

- **不是 SQL 语法问题**，查询语句本身合理。
- **根因在数据平台层**：Trino 读取 Iceberg 表元数据时，无法在 S3 上访问 snapshot 元数据文件（`.avro`）。
- 常见原因：元数据与 S3 不一致、写入提交失败导致元数据被误删、S3 权限/生命周期策略、或表元数据损坏。

## 建议排查步骤

1. 确认 S3 文件是否存在：`aws s3 ls s3://lecoo-iceberg/lake_dwd/dwd_trade_order_detail_wide/metadata/`
2. 查看表历史：`SELECT * FROM iceberg.lake_dwd."dwd_trade_order_detail_wide$history"`
3. 对比 snapshot 与 manifest：`$snapshots`、`$manifests` 系统表
4. 检查近期 ETL/写入任务是否有失败或超时
5. 若确认元数据损坏，由数据平台执行 snapshot 回滚或 `CALL iceberg.system.rollback_to_snapshot`
6. 应用层可对 `Failed to get status for file` 做友好提示，引导联系数据平台

## 本次环境说明

- 当前 workspace 仓库无 `lecoo_sql_ai` 应用代码，无法直接修改 `llm.py`。
- 错误堆栈来自运行环境 `/opt/lecoo_sql_ai/`。
