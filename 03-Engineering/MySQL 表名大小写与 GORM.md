---
title: MySQL 表名大小写与 GORM
tags: [mysql, gorm, go]
created: 2026-07-22
updated: 2026-07-22
aliases: [lower_case_table_names, GORM AutoMigrate 表名]
summary: 表名大小写与 GORM TableName 不一致导致 AutoMigrate 建新表与 record not found
type: pitfall
---

# 问题

本地启动后 GORM AutoMigrate 新建了 `Article` / `User` / `Category` 等表，而库中已有小写的 `article` / `user` / `category`。查询出现 `record not found`。

日志示例：

```sql
SELECT `id`,`slug`,`title`
FROM `Article`
WHERE isPublished = true
  AND isFeatured = true
ORDER BY updatedAt DESC
LIMIT 1;
```

返回 `rows:0` / `record not found`。

# 原因

Model 显式指定了 PascalCase 表名：

```go
func (Article) TableName() string {
    return "Article"
}
```

在大小写敏感环境下 `Article ≠ article`，于是：

1. 查不到原表
2. AutoMigrate 认为表不存在
3. 自动创建空表 `Article`

最终 `article`（旧数据）与 `Article`（空表）并存。

线上若 `lower_case_table_names = 1`，大小写不敏感，同一问题可能不明显。

| 值 | 含义 | 常见环境 |
|----|------|----------|
| 0 | 大小写敏感 | Linux |
| 1 | 大小写不敏感 | Windows 或特殊配置 |

# 解决方案

## 方案1（推荐）

`TableName` 与数据库真实表名一致（建议统一小写）：

```go
func (Article) TableName() string {
    return "article"
}
```

不依赖 MySQL 大小写配置。

## 方案2

数据库与代码统一为 PascalCase，并迁移历史数据。

## 方案3（不推荐）

```ini
[mysqld]
lower_case_table_names=1
```

依赖库配置，换环境易再踩坑。

## 后续规范

1. 表名统一小写
2. GORM `TableName` 与库表名完全一致
3. 生产禁止 AutoMigrate
4. 新环境先查：`SHOW VARIABLES LIKE 'lower_case_table_names';`

# 验证

```sql
SHOW TABLES;
SHOW VARIABLES LIKE 'lower_case_table_names';
```

```go
db = db.Debug()
```

确认 SQL 中的表名与真实表一致，查询不再 `record not found`，且不再误建新表。

# 相关链接

- [[Docker MySQL 远程连接]]
- [[MySQL]]
- [[GORM]]
