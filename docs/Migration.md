# 数据库迁移与升级指南

本文档说明如何在新环境首次初始化数据库，以及如何从旧版本升级数据表结构。

---

## 目录

1. [首次安装](#首次安装)
2. [自动升级](#自动升级)
3. [手动升级](#手动升级)
4. [版本升级脚本说明](#版本升级脚本说明)
5. [数据迁移到其他服务器](#数据迁移到其他服务器)
6. [常见问题](#常见问题)

---

## 首次安装

### 方式一：通过安装向导（推荐）

1. 确保 `src/config` 和 `src/storage` 目录可写。
2. 创建空白 MySQL 数据库。
3. 访问 `http://你的域名/install`。
4. 填写数据库主机、端口、数据库名、用户名、密码、表前缀。
5. 点击安装，系统会自动执行 `src/install/install.sql` 并写入 `config/mysql.php` 与 `config/version.php`。

### 方式二：手动导入 SQL

如果安装向导无法使用，可手动导入：

```bash
mysql -u 用户名 -p 数据库名 < src/install/install.sql
```

导入后：

1. 在 `config/` 目录下创建 `mysql.php`，内容参考：

```php
<?php
return [
    'host'     => '127.0.0.1',
    'port'     => '3306',
    'database' => 'kldns',
    'username' => 'root',
    'password' => 'your_password',
    'prefix'   => 'kldns_',
];
```

2. 在 `config/` 目录下创建 `version.php`，内容：

```php
<?php
return "3.1.0";
```

3. 修改 `src/install/install.sql` 中的默认邮箱与 SMTP 密码（生产环境必须修改）。

---

## 自动升级

kldns 内置了数据库结构自动升级逻辑，源码位于 `src/app/Http/Controllers/InstallController.php`。

升级检测逻辑：

1. 读取 `config/version.php` 中的当前版本。
2. 与代码中硬编码的版本号（`3.1.0`）比较。
3. 如果当前版本低于目标版本，自动扫描 `src/install/update.x.x.x.sql` 文件。
4. 按版本号从小到大依次执行 SQL 脚本。
5. 每个脚本执行成功后，更新 `config/version.php`。

### 触发自动升级

访问以下地址即可触发升级（需能正常连接数据库）：

```text
http://你的域名/install/update
```

> 注意：升级过程中任何一条 SQL 执行失败都会直接报错并终止，需检查报错信息并手动修复。

---

## 手动升级

如果你希望完全掌控升级过程，或在自动升级失败时排查问题，可以手动执行升级脚本。

### 升级前必读

- **务必备份数据库**，避免升级失败导致数据丢失。
- 确认当前版本号（查看 `config/version.php`）。
- 只执行高于当前版本的升级脚本，按版本号从小到大依次执行。

### 手动执行示例

假设当前版本为 `3.0.1`，需要升级到 `3.1.0`：

```bash
# 1. 备份数据库
mysqldump -u 用户名 -p 数据库名 > kldns_backup_$(date +%Y%m%d).sql

# 2. 执行升级脚本
mysql -u 用户名 -p 数据库名 < src/install/update.3.1.0.sql

# 3. 更新版本号文件
echo '<?php' > config/version.php
echo 'return "3.1.0";' >> config/version.php
```

---

## 版本升级脚本说明

### v3.0.1 → v3.1.0

文件：`src/install/update.3.1.0.sql`

变更内容：

```sql
-- 1. 在解析记录表中增加 checked_at 字段，用于记录最近检测时间
ALTER TABLE `kldns_domain_records`
ADD COLUMN `checked_at` int(10) UNSIGNED NOT NULL DEFAULT 0 AFTER `updated_at`;

-- 2. 为 checked_at 添加索引，提高定时检测任务查询效率
ALTER TABLE `kldns_domain_records`
ADD INDEX (`checked_at`);

-- 3. 在域名表中增加 desc 字段，用于填写域名说明
ALTER TABLE `kldns_domains`
ADD COLUMN `desc` text NULL AFTER `point`;
```

> 如果你的表前缀不是 `kldns_`，请将 SQL 中的 `kldns_` 替换为实际前缀。

---

## 数据迁移到其他服务器

### 场景一：迁移到新的 Web 服务器

1. 备份原服务器数据库：

```bash
mysqldump -u 用户名 -p 数据库名 > kldns_full.sql
```

2. 备份以下文件/目录：

```text
config/mysql.php
config/version.php
src/storage/
```

3. 在新服务器导入数据库：

```bash
mysql -u 用户名 -p 数据库名 < kldns_full.sql
```

4. 上传源码，恢复 `config/mysql.php`、`config/version.php` 和 `src/storage/`。
5. 根据新服务器环境配置伪静态（Apache / Nginx / IIS）。
6. 检查 `config/mysql.php` 中的数据库连接信息是否需要修改。

### 场景二：从其他二级域名分发系统迁移

如果你从其他系统迁移到 kldns，需要至少准备以下数据：

| 数据 | 说明 |
|------|------|
| 用户数据 | 导入到 `kldns_users` 表，`password` 必须为 Bcrypt 加密 |
| 域名数据 | 导入到 `kldns_domains`，需正确填写 `dns` 平台标识和 `domain_id` |
| 解析记录 | 导入到 `kldns_domain_records`，`record_id` 必须与 DNS 平台侧一致 |
| DNS 配置 | 导入到 `kldns_dns_configs`，`config` 字段为 JSON 格式 |
| 系统配置 | 导入到 `kldns_configs` |

迁移步骤建议：

1. 在目标环境全新安装 kldns。
2. 使用 SQL 工具将旧系统数据按字段映射导入对应表。
3. 对每条域名，确认 `dns` 字段值与 `src/app/Klsf/Dns/` 下平台类名一致。
4. 对每个 DNS 配置，校验 JSON 中字段名与目标平台 `configInfo()` 返回一致。
5. 登录后台，重新保存一次系统配置以刷新缓存。
6. 对关键域名和解析记录进行抽样测试，确保平台 API 可正常读写。

---

## 常见问题

### Q1：自动升级时报“更新数据表错误”怎么办？

可能原因：

- 数据库账号权限不足，无法执行 `ALTER TABLE`。
- 目标字段或索引已存在，说明之前手动执行过部分升级脚本。
- 表前缀不匹配，导致找不到表。

解决方法：

1. 检查 `config/mysql.php` 中的 `prefix` 是否正确。
2. 手动连接数据库，查看报错涉及的表结构。
3. 如果字段已存在，可以跳过对应 SQL，手动更新 `config/version.php`。

### Q2：如何确认当前数据库版本？

查看 `config/version.php` 文件内容：

```bash
cat config/version.php
```

### Q3：升级后需要清理缓存吗？

- 如果开启了 OPcache，建议访问一次 `/cache` 清理 OPcache。
- Laravel 配置缓存：若之前执行过 `php artisan config:cache`，需执行 `php artisan config:clear`。

### Q4：能否从 v2.x 直接升级到 v3.1.0？

v3.x 相对于 v2.x 框架由 ThinkPHP 改为 Laravel，表结构差异较大，**不支持直接 SQL 升级**。建议：

1. 在新环境安装 v3.1.0。
2. 手动导出用户、域名、解析记录等核心数据。
3. 按字段映射导入新系统。

---

## 相关文件位置

| 文件 | 说明 |
|------|------|
| `src/install/install.sql` | 首次安装脚本 |
| `src/install/update.3.1.0.sql` | v3.0.1 → v3.1.0 升级脚本 |
| `src/app/Http/Controllers/InstallController.php` | 安装与自动升级逻辑 |
| `config/mysql.php` | 数据库连接配置 |
| `config/version.php` | 当前数据库版本号 |
