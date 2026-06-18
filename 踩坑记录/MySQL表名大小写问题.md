# MySQL 表名大小写敏感导致 GORM 自动建表问题

## 问题现象

本地启动项目后发现：

```text
GORM AutoMigrate 自动创建了一批新表

Article
User
Category
...
```

而数据库原本已经存在：

```text
article
user
category
...
```

同时查询出现：

```text
record not found
```

日志示例：

```sql
SELECT `id`,`slug`,`title`
FROM `Article`
WHERE isPublished = true
  AND isFeatured = true
ORDER BY updatedAt DESC
LIMIT 1;
```

返回：

```text
rows:0
record not found
```

---

## 根本原因

Go Model 中显式指定了表名：

```go
func (Article) TableName() string {
    return "Article"
}
```

GORM 会认为表名是：

```text
Article
```

而数据库实际表名是：

```text
article
```

在 MySQL 大小写敏感环境下：

```text
Article ≠ article
```

因此：

1. 查询不到原表
2. AutoMigrate 认为表不存在
3. 自动创建新表 `Article`

最终导致：

```text
article   （旧数据）
Article   （空表）
```

同时存在。

---

## 为什么线上正常，本地异常？

检查：

```sql
SHOW VARIABLES LIKE 'lower_case_table_names';
```

线上结果：

```text
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| lower_case_table_names | 1     |
+------------------------+-------+
```

说明：

```text
MySQL 大小写不敏感
```

以下访问完全等价：

```sql
SELECT * FROM article;
SELECT * FROM Article;
SELECT * FROM ARTICLE;
```

因此线上即使代码写：

```go
func (Article) TableName() string {
    return "Article"
}
```

也能正常访问：

```text
article
```

表。

---

## MySQL lower_case_table_names 说明

### 0

```text
大小写敏感
```

常见于 Linux。

例如：

```text
Article
article
```

是两张不同表。

---

### 1

```text
大小写不敏感
```

常见于 Windows 或特殊配置环境。

例如：

```text
Article
article
ARTICLE
```

全部视为同一张表。

---

## 查看当前配置

```sql
SHOW VARIABLES LIKE 'lower_case_table_names';
```

---

## 排查步骤

### 1. 查看真实表名

```sql
SHOW TABLES;
```

确认实际是：

```text
article
```

还是：

```text
Article
```

---

### 2. 查看 MySQL 是否大小写敏感

```sql
SHOW VARIABLES LIKE 'lower_case_table_names';
```

---

### 3. 查看 GORM 实际查询的表

开启 SQL 日志：

```go
db = db.Debug()
```

观察日志：

```sql
FROM `Article`
```

还是：

```sql
FROM `article`
```

---

## 解决方案

### 方案1（推荐）

统一使用数据库真实表名：

```go
func (Article) TableName() string {
    return "article"
}
```

```go
func (User) TableName() string {
    return "user"
}
```

保证：

```text
代码表名 = 数据库表名
```

不依赖 MySQL 配置。

---

### 方案2

统一数据库迁移为 PascalCase：

```text
Article
User
Category
```

并保持：

```go
func (Article) TableName() string {
    return "Article"
}
```

但需要迁移历史数据。

---

### 方案3（不推荐）

修改 MySQL 配置：

```ini
[mysqld]
lower_case_table_names=1
```

依赖数据库配置解决。

缺点：

```text
不同环境行为可能不同
迁移到 Linux 容易再次踩坑
```

---

## 本次问题结论

线上数据库：

```text
lower_case_table_names = 1
```

因此：

```go
TableName() string { return "Article" }
```

能够访问：

```text
article
```

表。

本地数据库因为表名大小写与代码不一致，导致：

```text
Article
```

被认为是新表。

最终触发：

```text
GORM AutoMigrate 自动建表
查询数据异常
record not found
```

---

## 后续规范

1. 所有数据库表名统一使用小写
2. GORM TableName 与数据库表名保持完全一致
3. 生产环境禁止执行 AutoMigrate
4. 新环境部署后第一时间检查：

```sql
SHOW VARIABLES LIKE 'lower_case_table_names';
```

5. 遇到自动建表问题优先检查：

```sql
SHOW TABLES;
SHOW VARIABLES LIKE 'lower_case_table_names';
```

再排查 GORM 配置。