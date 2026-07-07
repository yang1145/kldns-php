# 快乐二级域名分发系统 API 文档

> 项目版本：`kldns v3.1.0`
> 后端框架：Laravel 5.8
> 文档日期：2026-07-08

本文档面向前后端开发者，梳理系统中所有可供前端或外部调用的接口。接口均位于 Web 路由中，返回 JSON 数据。

---

## 目录

1. [通用约定](#1-通用约定)
   - [1.1 基础路径](#11-基础路径)
   - [1.2 请求方式与 Content-Type](#12-请求方式与-content-type)
   - [1.3 统一响应格式](#13-统一响应格式)
   - [1.4 CSRF 防护](#14-csrf-防护)
   - [1.5 认证与会话](#15-认证与会话)
   - [1.6 错误码说明](#16-错误码说明)
2. [公共接口](#2-公共接口)
3. [用户中心接口](#3-用户中心接口)
4. [管理员接口](#4-管理员接口)
5. [数据模型与字段](#5-数据模型与字段)
6. [附录：DNS 解析平台配置字段](#6-附录dns-解析平台配置字段)
7. [附录：源码位置速查](#7-附录源码位置速查)

---

## 1. 通用约定

### 1.1 基础路径

所有接口都以项目部署域名为基础路径，入口文件为项目根目录的 `index.php`。

示例：

```text
https://example.com/login
https://example.com/home
https://example.com/admin/user
```

### 1.2 请求方式与 Content-Type

- 读接口：`GET` 或 `POST`。
- 写接口：`POST`。
- 表单提交时使用 `application/x-www-form-urlencoded`。
- 文件/图片接口按常规浏览器行为处理（如验证码直接返回图片）。

### 1.3 统一响应格式

绝大多数 JSON 接口返回如下结构：

```json
{
  "status": 0,
  "message": "操作成功",
  "data": {}
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `status` | int | `0` 表示成功，非 `0` 表示失败或异常 |
| `message` | string | 提示信息 |
| `data` | mixed | 业务数据，成功时视接口而定；失败时可能为空 |

部分接口会扩展额外字段，例如登录成功返回 `go` 字段表示跳转地址。

### 1.4 CSRF 防护

系统启用 Laravel Web 中间件，所有 `POST / PUT / DELETE` 等写入请求必须携带 CSRF Token。

- 前端从页面 `<meta name="csrf-token">` 标签读取。
- 请求头名称：`X-CSRF-TOKEN`。
- 未携带或已过期将返回 `419` HTTP 状态码，前端应提示刷新页面。

参考实现：`js/main.js:21-32`。

### 1.5 认证与会话

- 用户登录态：基于 Laravel Session + `web` Guard。
- 管理员登录态：基于 Laravel Session + `admin` Guard。
- 登录成功后服务端会写入 `login_user_sid_web` 或 `login_user_sid_admin` Session 键。
- 退出登录会重置用户 `sid` 字段，使已有会话失效。

管理员判定条件：`gid == 99`。

### 1.6 错误码说明

| 状态码/场景 | 含义 |
|-------------|------|
| `status = 0` | 操作成功 |
| `status = -1` | 业务错误（参数校验失败、权限不足、操作不存在等） |
| `status = 1` | 部分接口中的失败状态（如登录/安装校验） |
| HTTP 419 | CSRF Token 过期或缺失 |
| HTTP 500 | 服务端异常或链接失效（如邮件激活链接过期） |

---

## 2. 公共接口

无需登录即可访问。

### 2.1 获取验证码

- **URL**：`GET /captcha`
- **说明**：直接返回 JPEG 图片，验证码内容写入 Session `captcha_code`。
- **源码**：`src/app/Http/Controllers/Index/IndexController.php:246-268`

### 2.2 用户登录

- **URL**：`POST /login`
- **源码**：`src/app/Http/Controllers/Auth/LoginController.php:60-72`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `username` | string | 是 | 用户名 |
| `password` | string | 是 | 密码 |

**成功响应**：

```json
{
  "status": 0,
  "message": "登录成功！",
  "go": "/home"
}
```

**失败响应**：

```json
{
  "status": -1,
  "message": "账号或者密码不正确"
}
```

### 2.3 管理员登录

- **URL**：`POST /admin/login`
- **源码**：`src/app/Http/Controllers/Auth/LoginController.php:74-89`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `username` | string | 是 | 管理员用户名 |
| `password` | string | 是 | 密码 |
| `code` | string | 是 | 验证码（需先请求 `/captcha`） |

**成功响应**：

```json
{
  "status": 0,
  "message": "登录成功！",
  "go": "/admin"
}
```

### 2.4 用户注册

- **URL**：`POST /reg`
- **源码**：`src/app/Http/Controllers/Index/IndexController.php:187-225`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `username` | string | 是 | 用户名，至少 4 位 |
| `password` | string | 是 | 密码，至少 5 位 |
| `email` | string | 是 | 邮箱地址 |
| `code` | string | 是 | 验证码 |

**成功响应**：

```json
{
  "status": 0,
  "message": "恭喜您，注册成功，马上登录！"
}
```

若后台开启邮箱认证，返回：

```json
{
  "status": 0,
  "message": "恭喜您，注册成功。已发送一封激活邮件到你的邮箱：xxx@example.com，请注意查收！"
}
```

### 2.5 找回密码

#### 2.5.1 发送重置邮件

- **URL**：`POST /password`
- **源码**：`src/app/Http/Controllers/Index/IndexController.php:144-170`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `action` | string | 是 | 固定值 `sendPasswordEmail` |
| `username` | string | 是 | 用户名或邮箱 |
| `code` | string | 是 | 验证码 |

#### 2.5.2 设置新密码

- **URL**：`POST /password`
- **源码**：`src/app/Http/Controllers/Index/IndexController.php:118-142`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `action` | string | 是 | 固定值 `setPassword` |
| `code` | string | 是 | 邮件链接中的 `code` 参数 |
| `password` | string | 是 | 新密码 |
| `re_password` | string | 是 | 确认新密码 |

### 2.6 退出登录

- **URL**：`POST /logout` 或 `POST /admin/logout`
- **说明**：清除当前 Guard 会话，并重置用户 `sid`。
- **源码**：`src/app/Http/Controllers/Auth/LoginController.php:91-107`

### 2.7 邮箱激活验证

- **URL**：`GET /verify?code={code}`
- **说明**：通过邮件链接激活账号，成功后跳转 `/home`。
- **源码**：`src/app/Http/Controllers/Index/IndexController.php:172-185`

### 2.8 检查二级域名是否可用

- **URL**：`POST /check`
- **源码**：`src/app/Http/Controllers/Index/IndexController.php:227-244`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 二级域名前缀 |
| `did` | int | 是 | 域名 ID |

**成功响应**：

```json
{
  "status": 0,
  "message": "test.example.com 可用！"
}
```

### 2.9 获取版本号

- **URL**：`GET /version`
- **响应**：

```json
{
  "status": 0,
  "version": "3.1.0"
}
```

### 2.10 安装接口

- **URL**：`POST /install`
- **说明**：首次安装时配置 MySQL 数据库。
- **源码**：`src/app/Http/Controllers/InstallController.php:75-139`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `action` | string | 是 | 固定值 `mysql` |
| `host` | string | 是 | 数据库主机 |
| `port` | string | 是 | 数据库端口 |
| `database` | string | 是 | 数据库名 |
| `username` | string | 是 | 数据库用户名 |
| `password` | string | 是 | 数据库密码 |
| `prefix` | string | 是 | 表前缀 |

### 2.11 自动检测任务（Cron）

- **URL**：`GET /cron/check/{key}`
- **说明**：定时任务入口，按配置关键词检测解析记录指向的网页内容，命中则删除记录。
- **参数**：`key` 为 32 位监控密钥，存储于 `kldns_configs.cronKey`。
- **源码**：`src/app/Http/Controllers/Index/IndexController.php:28-93`

---

## 3. 用户中心接口

需要用户登录，前缀 `/home`，中间件：`auth`, `auth.session:web`。

所有接口统一入口：

```text
POST /home
```

通过 `action` 参数区分操作。

### 3.1 发送认证邮件

- **action**：`verify`
- **源码**：`src/app/Http/Controllers/Home/HomeController.php:54-69`

**请求参数**：无

**响应**：

```json
{
  "status": 0,
  "message": "已将认证邮件发送到xxx@example.com，请注意查收！"
}
```

### 3.2 修改密码

- **action**：`profile`
- **源码**：`src/app/Http/Controllers/Home/HomeController.php:71-93`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `old_password` | string | 是 | 旧密码 |
| `new_password` | string | 是 | 新密码，至少 5 位 |

### 3.3 查询我的解析记录

- **action**：`recordList`
- **源码**：`src/app/Http/Controllers/Home/HomeController.php:177-184`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `did` | int | 否 | 按域名 ID 筛选 |
| `type` | string | 否 | 按记录类型筛选，如 `A`、`CNAME` |
| `name` | string | 否 | 按主机记录筛选 |
| `value` | string | 否 | 按记录值筛选 |
| `pageSize` | int | 否 | 每页条数，默认 10，范围 10-200 |

**响应 `data`**：Laravel 分页对象，包含 `data`、`current_page`、`last_page`、`total` 等字段。

### 3.4 查询可用域名列表

- **action**：`domainList`
- **源码**：`src/app/Http/Controllers/Home/HomeController.php:186-205`

**请求参数**：无

**响应 `data`**：

```json
[
  {
    "did": 100,
    "domain": "example.com",
    "point": 0,
    "desc": "",
    "line": [
      { "Id": "0", "Name": "默认" }
    ]
  }
]
```

### 3.5 新增/修改解析记录

- **action**：`recordStore`
- **说明**：需用户状态为已认证（`status = 2`）。`id` 为空时新增，否则修改。
- **源码**：`src/app/Http/Controllers/Home/HomeController.php:101-175`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | int | 否 | 记录 ID，修改时必填 |
| `did` | int | 是 | 域名 ID |
| `name` | string | 是 | 主机记录前缀 |
| `type` | string | 是 | 记录类型 |
| `value` | string | 是 | 记录值 |
| `line_id` | string | 是 | 线路 ID |

### 3.6 删除解析记录

- **action**：`recordDelete`
- **源码**：`src/app/Http/Controllers/Home/HomeController.php:207-222`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | int | 是 | 记录 ID |

### 3.7 查询积分记录

- **action**：`pointRecord`
- **源码**：`src/app/Http/Controllers/Home/HomeController.php:95-99`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `act` | string | 否 | `increase` 只查增加，`reduce` 只查扣除，其他按 `action` 字段匹配 |
| `pageSize` | int | 否 | 每页条数 |

---

## 4. 管理员接口

需要管理员登录，前缀 `/admin`，中间件：`auth:admin`, `auth.session:admin`。

统一入口：`POST /admin`，各子模块分别为 `POST /admin/user`、`POST /admin/domain` 等。

### 4.1 管理员修改密码

- **URL**：`POST /admin`
- **action**：`profile`
- **源码**：`src/app/Http/Controllers/Admin/AdminController.php:32-54`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `old_password` | string | 是 | 旧密码 |
| `new_password` | string | 是 | 新密码 |

### 4.2 用户管理（`/admin/user`）

#### 4.2.1 查询用户列表

- **action**：`select`
- **源码**：`src/app/Http/Controllers/Admin/UserController.php:95-99`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `gid` | int | 否 | 按用户组筛选 |
| `username` | string | 否 | 按用户名筛选 |
| `uid` | int | 否 | 按用户 ID 筛选 |
| `email` | string | 否 | 按邮箱筛选 |
| `pageSize` | int | 否 | 每页条数 |

#### 4.2.2 修改用户信息

- **action**：`update`
- **源码**：`src/app/Http/Controllers/Admin/UserController.php:65-93`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `uid` | int | 是 | 用户 ID |
| `gid` | int | 是 | 用户组 ID |
| `status` | int | 是 | 状态：0 禁用、1 待认证、2 已认证 |
| `email` | string | 是 | 邮箱 |
| `password` | string | 否 | 新密码，不填则不修改 |

#### 4.2.3 删除用户

- **action**：`delete`
- **源码**：`src/app/Http/Controllers/Admin/UserController.php:101-113`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | int | 是 | 用户 ID |

#### 4.2.4 积分操作

- **action**：`point`
- **源码**：`src/app/Http/Controllers/Admin/UserController.php:46-63`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `uid` | int | 是 | 用户 ID |
| `point` | int | 是 | 积分数，必须大于 0 |
| `remark` | string | 否 | 备注 |
| `act` | int | 否 | 传 `1` 表示扣除，否则表示增加 |

#### 4.2.5 查询积分记录

- **action**：`pointRecord`
- **源码**：`src/app/Http/Controllers/Admin/UserController.php:40-44`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `uid` | int | 否 | 按用户 ID 筛选 |
| `act` | string | 否 | 同 3.7 |
| `pageSize` | int | 否 | 每页条数 |

### 4.3 用户组管理（`/admin/user/group`）

#### 4.3.1 查询用户组列表

- **action**：`select`
- **源码**：`src/app/Http/Controllers/Admin/UserGroupController.php:55-59`

#### 4.3.2 新增/修改用户组

- **action**：`store`
- **源码**：`src/app/Http/Controllers/Admin/UserGroupController.php:33-53`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `gid` | int | 否 | 用户组 ID，为空则新增 |
| `name` | string | 是 | 用户组名称 |

#### 4.3.3 删除用户组

- **action**：`delete`
- **说明**：`gid < 101` 的系统默认组不可删除。删除后，该组用户会被重置为默认组（gid=100）。
- **源码**：`src/app/Http/Controllers/Admin/UserGroupController.php:61-76`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | int | 是 | 用户组 ID |

### 4.4 域名管理（`/admin/domain`）

#### 4.4.1 查询域名列表

- **action**：`select`
- **源码**：`src/app/Http/Controllers/Admin/DomainController.php:146-150`

#### 4.4.2 从解析平台拉取域名列表

- **action**：`domainList`
- **说明**：用于添加域名前的选择，只返回系统中未添加的域名。
- **源码**：`src/app/Http/Controllers/Admin/DomainController.php:39-68`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `dns` | string | 是 | 解析平台标识，如 `Dnspod`、`Aliyun` |

#### 4.4.3 添加域名

- **action**：`add`
- **源码**：`src/app/Http/Controllers/Admin/DomainController.php:98-144`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `dns` | string | 是 | 解析平台标识 |
| `domain` | string | 是 | 格式：`domain_id,domain`，例如 `123456,example.com` |
| `desc` | string | 否 | 域名说明 |
| `groups` | array | 是 | 可使用的用户组 ID 数组，传 `["0"]` 表示所有组 |
| `point` | int | 否 | 添加记录所需积分，默认 0 |

#### 4.4.4 修改域名

- **action**：`update`
- **源码**：`src/app/Http/Controllers/Admin/DomainController.php:70-96`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `did` | int | 是 | 域名 ID |
| `groups` | array | 是 | 用户组 ID 数组 |
| `point` | int | 否 | 所需积分 |
| `desc` | string | 否 | 说明 |

#### 4.4.5 删除域名

- **action**：`delete`
- **说明**：删除域名会级联删除其下所有解析记录。
- **源码**：`src/app/Http/Controllers/Admin/DomainController.php:152-165`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | int | 是 | 域名 ID |

### 4.5 解析记录管理（`/admin/domain/record`）

#### 4.5.1 查询解析记录

- **action**：`select`
- **源码**：`src/app/Http/Controllers/Admin/DomainRecordController.php:31-35`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `did` | int | 否 | 按域名 ID 筛选 |
| `type` | string | 否 | 按类型筛选 |
| `name` | string | 否 | 按主机记录筛选 |
| `value` | string | 否 | 按记录值筛选 |
| `uid` | int | 否 | 按用户 ID 筛选 |
| `pageSize` | int | 否 | 每页条数 |

#### 4.5.2 删除解析记录

- **action**：`delete`
- **源码**：`src/app/Http/Controllers/Admin/DomainRecordController.php:37-52`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | int | 是 | 记录 ID |

### 4.6 DNS 解析平台配置（`/admin/config/dns`）

#### 4.6.1 获取所有支持的解析平台

- **action**：`all`
- **说明**：返回各平台需要的配置字段。
- **源码**：`src/app/Http/Controllers/Admin/DnsConfigController.php:35-44`

**响应 `data`**：

```json
{
  "Dnspod": [
    { "name": "ID", "placeholder": "请输入ID", "tips": "..." },
    { "name": "Token", "placeholder": "请输入Token", "tips": "" }
  ],
  "Aliyun": [
    { "name": "AccessKeyId", "placeholder": "...", "tips": "..." },
    { "name": "AccessKeySecret", "placeholder": "...", "tips": "" }
  ]
}
```

#### 4.6.2 保存解析平台配置

- **action**：`store`
- **说明**：保存前会调用平台接口校验配置是否可用。
- **源码**：`src/app/Http/Controllers/Admin/DnsConfigController.php:46-74`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `dns` | string | 是 | 平台标识 |
| `config` | object | 是 | 平台配置对象，字段见第 6 节 |

#### 4.6.3 查询已保存的配置列表

- **action**：`select`
- **源码**：`src/app/Http/Controllers/Admin/DnsConfigController.php:76-80`

#### 4.6.4 删除配置

- **action**：`delete`
- **源码**：`src/app/Http/Controllers/Admin/DnsConfigController.php:82-94`

**请求参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `dns` | string | 是 | 平台标识 |

### 4.7 系统配置（`/admin/config`）

#### 4.7.1 保存系统配置

- **action**：`config`
- **说明**：除 `action` 外，其余 POST 字段都会写入 `kldns_configs` 表。数组类型字段会按 `array_{key}` 存储为 JSON。
- **源码**：`src/app/Http/Controllers/Admin/ConfigController.php:58-79`

**常用配置项**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `web[name]` | string | 网站名称 |
| `web[title]` | string | 网站标题 |
| `web[keywords]` | string | SEO 关键词 |
| `web[description]` | string | SEO 描述 |
| `user[reg]` | int | 是否开放注册：0 关闭、1 开启 |
| `user[email]` | int | 是否开启邮箱认证 |
| `user[point]` | int | 注册赠送积分 |
| `mail[host]` | string | SMTP 服务器 |
| `mail[port]` | string | SMTP 端口 |
| `mail[encryption]` | string | 加密方式，如 ssl |
| `mail[username]` | string | 邮箱账号 |
| `mail[password]` | string | 邮箱密码/授权码 |
| `mail[test]` | string | 测试收件邮箱 |
| `keywords` | string | 自动检测关键词，每行一个 |
| `reserve_domain_name` | string | 保留域名前缀，逗号分隔 |
| `index_urls` | string | 首页链接，每行 `标题|链接` |
| `html_header` | string | 首页顶部 HTML |
| `html_home` | string | 用户中心首页 HTML |

#### 4.7.2 获取监控任务信息

- **action**：`getKeywordsInfo`
- **说明**：返回 Cron 监控密钥和最近检测时间。
- **源码**：`src/app/Http/Controllers/Admin/ConfigController.php:41-56`

**响应 `data`**：

```json
{
  "key": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "checked_at": "2026-07-08 06:00:00"
}
```

#### 4.7.3 更换监控密钥

- **action**：`changeKey`
- **源码**：`src/app/Http/Controllers/Admin/ConfigController.php:34-39`

**响应**：

```json
{
  "status": 0,
  "message": "更换成功",
  "key": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
}
```

---

## 5. 数据模型与字段

### 5.1 用户表 `kldns_users`

| 字段 | 类型 | 说明 |
|------|------|------|
| `uid` | int | 主键 |
| `gid` | int | 用户组 ID，`99` 为管理组，`100` 为默认组 |
| `status` | tinyint | 0 禁用、1 待认证、2 已认证 |
| `username` | string | 用户名 |
| `password` | string | 加密密码 |
| `remember_token` | string | Laravel 记住我 Token |
| `sid` | string | 会话安全 ID |
| `email` | string | 邮箱 |
| `point` | int | 积分余额 |
| `created_at` | int | 注册时间（Unix 时间戳） |
| `updated_at` | int | 更新时间（Unix 时间戳） |

### 5.2 用户组表 `kldns_user_groups`

| 字段 | 类型 | 说明 |
|------|------|------|
| `gid` | int | 主键，`gid >= 100` 为普通组 |
| `name` | string | 组名 |

### 5.3 域名表 `kldns_domains`

| 字段 | 类型 | 说明 |
|------|------|------|
| `did` | int | 主键 |
| `dns` | string | 解析平台标识 |
| `domain_id` | string | 平台侧域名 ID |
| `domain` | string | 域名，如 `example.com` |
| `groups` | string | 可用用户组，逗号分隔，`0` 表示所有组 |
| `point` | int | 添加记录所需积分 |
| `desc` | text | 说明 |

### 5.4 解析记录表 `kldns_domain_records`

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | int | 主键 |
| `uid` | int | 用户 ID |
| `did` | int | 域名 ID |
| `record_id` | string | 平台侧记录 ID |
| `name` | string | 主机记录 |
| `type` | string | 记录类型 |
| `value` | string | 记录值 |
| `line_id` | string | 线路 ID |
| `line` | string | 线路名称 |
| `checked_at` | int | 最近检测时间 |

### 5.5 积分记录表 `kldns_user_point_records`

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | int | 主键 |
| `uid` | int | 用户 ID |
| `action` | string | 动作描述，如 `增加`、`扣除`、`消费` |
| `point` | int | 变动积分，正数增加、负数扣除 |
| `rest` | int | 变动后余额 |
| `remark` | string | 备注 |

### 5.6 系统配置表 `kldns_configs`

| 字段 | 类型 | 说明 |
|------|------|------|
| `k` | string | 配置键 |
| `v` | text | 配置值 |

### 5.7 DNS 配置表 `kldns_dns_configs`

| 字段 | 类型 | 说明 |
|------|------|------|
| `dns` | string | 平台标识，主键 |
| `config` | string | JSON 格式配置 |

---

## 6. 附录：DNS 解析平台配置字段

系统内置支持的解析平台及对应配置字段：

| 平台 | 标识 | 配置字段 |
|------|------|----------|
| DNSPod | `Dnspod` | `ID`、`Token` |
| 阿里云 | `Aliyun` | `AccessKeyId`、`AccessKeySecret` |
| Cloudflare | `Cloudflare` | `ApiKey`、`Email` |
| DNS.com | `DnsCom` | `ApiKey`、`ApiSecret` |
| DNSLA | `DnsLa` | `ApiId`、`ApiKey` |
| DNSDun | `DnsDun` | `UID`、`API_KEY` |

平台类位于 `src/app/Klsf/Dns/` 目录，新增平台需实现 `DnsInterface` 接口。

---

## 7. 附录：源码位置速查

| 模块 | 文件 |
|------|------|
| Web 路由 | `src/routes/web.php` |
| API 路由 | `src/routes/api.php` |
| 首页/公共控制器 | `src/app/Http/Controllers/Index/IndexController.php` |
| 登录认证 | `src/app/Http/Controllers/Auth/LoginController.php` |
| 用户中心 | `src/app/Http/Controllers/Home/HomeController.php` |
| 管理员首页 | `src/app/Http/Controllers/Admin/AdminController.php` |
| 用户管理 | `src/app/Http/Controllers/Admin/UserController.php` |
| 用户组管理 | `src/app/Http/Controllers/Admin/UserGroupController.php` |
| 域名管理 | `src/app/Http/Controllers/Admin/DomainController.php` |
| 解析记录管理 | `src/app/Http/Controllers/Admin/DomainRecordController.php` |
| DNS 配置管理 | `src/app/Http/Controllers/Admin/DnsConfigController.php` |
| 系统配置管理 | `src/app/Http/Controllers/Admin/ConfigController.php` |
| 安装控制器 | `src/app/Http/Controllers/InstallController.php` |
| 前端请求封装 | `js/main.js` |
| 公共辅助函数 | `src/app/Helper.php` |
| 数据库安装脚本 | `src/install/install.sql` |
