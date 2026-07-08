# 二级域名分发系统

一个基于 **Laravel 5.8** 开发的多用户、多域名、多平台二级域名分发系统。支持对接多家主流 DNS 解析服务商，提供 Web 端用户中心与后台管理，可用于学习、测试或内部二级域名自助分发场景。

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="License"></a>
  <img src="https://img.shields.io/badge/web_server-IIS-0078D4?logo=iis" alt="IIS">
  <img src="https://img.shields.io/badge/web_server-Apache-D22128?logo=apache" alt="Apache">
  <img src="https://img.shields.io/badge/web_server-Nginx-009639?logo=nginx" alt="Nginx">
  <img src="https://img.shields.io/badge/web_server-Caddy-22B638?logo=caddy" alt="Caddy">
  <img src="https://img.shields.io/badge/web_server-LiteSpeed-00AEEF" alt="LiteSpeed">
  <img src="https://img.shields.io/badge/support-all_web_servers-green" alt="All Web Servers">
  <img src="https://img.shields.io/badge/php-7.1--7.3-777BB4?logo=php" alt="PHP">
  <img src="https://img.shields.io/badge/mysql-5.6+-4479A1?logo=mysql" alt="MySQL">
</p>

---

## 目录

1. [项目特点](#项目特点)
2. [技术栈](#技术栈)
3. [环境要求](#环境要求)
4. [目录结构](#目录结构)
5. [快速安装](#快速安装)
6. [伪静态配置](#伪静态配置)
7. [使用说明](#使用说明)
8. [API 文档](#api-文档)
9. [支持的 DNS 解析平台](#支持的-dns-解析平台)
10. [系统配置项](#系统配置项)
11. [自动检测任务](#自动检测任务)
12. [更新日志](#更新日志)
13. [问题反馈](#问题反馈)

---

## 项目特点

- **多平台支持**：可对接 DNSPod、阿里云、Cloudflare、DNS.com、DNSLA、DNSDun 等主流解析平台。
- **多租户架构**：支持多用户、多用户组、多域名同时存在，不同用户组可分配不同域名权限。
- **积分机制**：域名可设置添加记录所需积分，管理员可手动增减用户积分。
- **邮箱认证**：支持注册邮箱激活认证，支持 SMTP 邮件发送（密码找回、激活邮件）。
- **内容检测**：内置 Cron 定时任务，可按关键词自动检测解析记录指向的网页内容并清理违规记录。
- **权限分离**：普通用户中心与管理员后台分离，管理员账号为 `gid = 99` 的特殊用户组。
- **响应式界面**：基于 Vue + Layui 构建，操作简单，交互统一。

---

## 技术栈

| 层级 | 技术 |
|------|------|
| 后端框架 | Laravel 5.8 |
| 前端框架 | Vue.js 2.x |
| UI 组件 | Layui |
| 数据库 | MySQL 5.6+ |
| HTTP 客户端 | GuzzleHttp |
| 验证码 | Gregwar/Captcha |
| 环境配置 | vlucas/phpdotenv |

---

## 环境要求

- PHP 7.1 - 7.3（推荐）
- PHP OpenSSL 扩展
- PHP PDO 扩展
- PHP Mbstring 扩展
- PHP Tokenizer 扩展
- PHP XML 扩展
- PHP Ctype 扩展
- PHP JSON 扩展
- PHP BCMath 扩展
- MySQL 5.6 或更高版本
- Web 服务器需支持 URL 重写（伪静态）

---

## 目录结构

```text
free-dns/
├── css/                    # 前端样式
├── images/                 # 图片资源
├── js/                     # 前端脚本
│   └── main.js             # 全局请求封装、Vue 扩展
├── src/                    # Laravel 应用源码
│   ├── app/
│   │   ├── Helper.php                 # 公共辅助函数
│   │   ├── Http/Controllers/          # 控制器
│   │   ├── Klsf/Dns/                  # DNS 解析平台适配器
│   │   ├── Models/                    # Eloquent 模型
│   │   └── ...
│   ├── config/             # Laravel 配置文件
│   ├── database/           # 迁移、填充、工厂
│   ├── install/            # 安装 SQL 脚本
│   ├── routes/             # 路由定义
│   ├── storage/            # 缓存、日志、会话存储
│   └── vendor/             # Composer 依赖
├── config/                 # 安装后生成的数据库配置文件
├── docs/                   # 项目文档
│   ├── API.md              # API 接口文档
│   └── Migration.md        # 数据库迁移与升级指南
├── index.php               # 统一入口文件
├── .htaccess               # Apache 重写规则
├── web.config              # IIS URL Rewrite 配置
└── README.md               # 项目说明
```

---

## 快速安装

1. 将源码上传至 Web 站点根目录，确保 `src/config`、`src/storage` 目录可写。
2. 创建 MySQL 数据库，并授予对应账号读写权限。
3. 访问 `http://你的域名/install`，填写数据库信息完成安装。
4. 安装完成后使用默认管理员登录：
   - 用户名：`admin`
   - 密码：`admin`
   - 登录地址：`http://你的域名/admin/login`
5. 进入后台 → 系统配置，完善网站名称、邮箱 SMTP、注册开关等。
6. 进入后台 → DNS 配置，添加并校验你的 DNS 解析平台账号。
7. 进入后台 → 域名管理，从解析平台拉取并添加可分发域名。

> **安全提示**：首次登录后请立即修改默认管理员密码，并更换监控密钥。

---

## 伪静态配置

### Apache

确保已启用 `mod_rewrite`，站点根目录已包含 `.htaccess` 文件，内容如下：

```apache
<IfModule mod_rewrite.c>
    <IfModule mod_negotiation.c>
        Options -MultiViews -Indexes
    </IfModule>

    RewriteEngine On

    # Handle Authorization Header
    RewriteCond %{HTTP:Authorization} .
    RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]

    # Redirect Trailing Slashes If Not A Folder...
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_URI} (.+)/$
    RewriteRule ^ %1 [L,R=301]

    # Send Requests To Front Controller...
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteRule ^ index.php [L]
</IfModule>
```

### Nginx

```nginx
location / {
    try_files $uri $uri/ /index.php?$query_string;
}
```

### IIS

确保已安装 [URL Rewrite Module 2.1](https://www.iis.net/downloads/microsoft/url-rewrite)，并将项目根目录的 `web.config` 保留在站点根目录。该配置会自动将所有非文件、非目录请求转发到 `index.php`。

---

## 使用说明

### 普通用户流程

1. 访问首页注册账号，若开启邮箱认证则需点击邮件链接激活。
2. 登录后进入用户中心 `/home`。
3. 在可用域名列表中选择域名，填写主机记录、记录类型、记录值和线路，提交添加解析。
4. 解析记录会同步写入对应的 DNS 解析平台，等待 DNS 生效即可使用。
5. 用户可在积分记录中查看积分变动明细。

### 管理员流程

1. 登录后台 `/admin`。
2. **系统配置**：设置网站信息、注册开关、SMTP 邮箱、首页 HTML、保留前缀、监控密钥等。
3. **DNS 配置**：添加并校验 DNS 解析平台账号，支持多个平台同时存在。
4. **域名管理**：从平台拉取域名，设置可用用户组、所需积分和说明。
5. **用户管理**：查看、编辑、删除用户，手动增减积分。
6. **解析记录管理**：查看全站解析记录，删除违规记录。
7. **用户组管理**：自定义用户分组，实现不同域名按组分配。

---

## API 文档

前后端交互接口已整理为独立文档：

👉 [docs/API.md](docs/API.md)

文档包含：
- 通用响应格式与 CSRF 说明
- 公共接口（登录、注册、验证码、找回密码、安装等）
- 用户中心接口（解析记录管理、域名列表、积分记录等）
- 管理员接口（用户/用户组/域名/解析记录/DNS 配置/系统配置）
- 数据模型字段说明
- DNS 平台配置字段速查

---

## 支持的 DNS 解析平台

| 平台 | 标识 | 所需配置 |
|------|------|----------|
| DNSPod | `Dnspod` | ID、Token |
| 阿里云 | `Aliyun` | AccessKeyId、AccessKeySecret |
| Cloudflare | `Cloudflare` | ApiKey、Email |
| DNS.com | `DnsCom` | ApiKey、ApiSecret |
| DNSLA | `DnsLa` | ApiId、ApiKey |
| DNSDun | `DnsDun` | UID、API_KEY |

平台适配器源码位于 `src/app/Klsf/Dns/`，新增平台需实现 `DnsInterface` 接口。

---

## 系统配置项

后台「系统配置」中可配置以下常用项：

| 配置项 | 说明 |
|--------|------|
| 网站名称 / 标题 / SEO | 首页与邮件模板中使用 |
| 用户注册开关 | 是否开放注册 |
| 邮箱认证开关 | 注册后是否需要邮件激活 |
| 注册赠送积分 | 新用户注册时赠送的积分 |
| SMTP 邮箱 | 用于发送激活/找回密码邮件 |
| 保留域名前缀 | 禁止用户注册的前缀，如 `www`、`m`、`qq` |
| 首页链接 | 每行 `标题|链接` 格式 |
| 首页 HTML | 首页顶部与用户中心欢迎内容 |
| 检测关键词 | Cron 任务中用于扫描网页内容的敏感词 |
| 监控密钥 | Cron 任务 URL `/cron/check/{key}` 的校验密钥 |

---

## 自动检测任务

系统内置关键词扫描任务，用于定期检查解析记录指向的网页内容。

- **访问地址**：`GET /cron/check/{key}`
- **密钥位置**：后台 → 系统配置 → 监控密钥
- **检测逻辑**：
  1. 每次选取 `checked_at` 超过 5 分钟且最久未检测的记录。
  2. 访问 `http://{name}.{domain}` 获取网页内容。
  3. 若命中配置的任意关键词，则同步删除本地记录与 DNS 平台记录。
  4. 单次任务执行时间不超过 25 秒。

建议通过服务器 Cron 每分钟访问一次该地址，例如：

```bash
* * * * * curl -s "https://你的域名/cron/check/你的32位密钥" > /dev/null 2>&1
```