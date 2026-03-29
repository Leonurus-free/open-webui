# Open WebUI 集成 Casdoor 单点登录配置指南

本文档详细说明如何将 Open WebUI 与 Casdoor 进行单点登录（SSO）集成。

## 目录

- [概述](#概述)
- [前置要求](#前置要求)
- [Casdoor 配置步骤](#casdoor-配置步骤)
- [Open WebUI 配置步骤](#open-webui-配置步骤)
- [配置验证](#配置验证)
- [高级配置](#高级配置)
- [故障排查](#故障排查)
- [安全建议](#安全建议)

---

## 概述

Casdoor 是一个开源的身份和访问管理（IAM）平台，支持 OAuth 2.0 和 OpenID Connect (OIDC) 协议。通过集成 Casdoor，Open WebUI 可以实现：

- 统一的用户身份认证
- 单点登录（SSO）体验
- 集中的用户管理
- 基于角色的访问控制（可选）
- 用户组管理（可选）

### 架构说明

```
用户浏览器 <---> Open WebUI 前端 <---> Open WebUI 后端 <---> Casdoor
                (15701端口)           (15700端口)         (18104端口)
```

**重要端口说明**：
- **前端端口 (15701)**：用户访问的 Web 界面地址，用于 `WEBUI_URL` 和登出重定向
- **后端端口 (15700)**：API 服务地址，用于 OAuth 回调 `OPENID_REDIRECT_URI`
- **Casdoor 端口 (18104)**：身份认证服务地址，用于 `OPENID_PROVIDER_URL`

---

## 前置要求

### Casdoor 要求

- Casdoor 服务已部署并可访问
- 具有 Casdoor 管理员权限
- Casdoor 版本：v1.x 或更高

### Open WebUI 要求

- Open WebUI 版本：支持 OIDC 的版本
- 可以访问 `.env` 配置文件
- 了解前后端部署架构（特别是端口配置）

### 网络要求

- Open WebUI 后端可以访问 Casdoor 服务
- 用户浏览器可以访问 Casdoor 服务
- 确保防火墙允许相关端口通信

---

## Casdoor 配置步骤

### 步骤 1: 登录 Casdoor 管理后台

1. 访问 Casdoor 管理界面（例如：`http://192.168.100.100:18104`）
2. 使用管理员账号登录

### 步骤 2: 创建应用（Application）

1. 在左侧菜单中选择 **Applications**
2. 点击 **Add** 按钮创建新应用
3. 填写应用基本信息：

   | 字段 | 值 | 说明 |
   |------|-----|------|
   | Name | `open-webui` | 应用名称，可自定义 |
   | Display name | `Open WebUI` | 显示名称 |
   | Organization | 选择组织 | 通常选择 `built-in` |
   | Logo | 上传 Logo（可选） | 登录页面显示的图标 |

### 步骤 3: 配置 OAuth 设置

在应用配置页面的 **OAuth** 部分：

1. **Client ID**：系统自动生成，记录此值（例如：`34c0ccfa111882b676cf`）
2. **Client Secret**：系统自动生成，记录此值（例如：`de3b07d7365fb356630835e189aaeeebd76ae2b7`）
3. **Redirect URLs**：添加 Open WebUI 的回调地址
   ```
   http://192.168.100.123:15700/oauth/oidc/callback
   ```
   **注意**：这里使用的是**后端端口**（15700），不是前端端口（15701）

4. **Grant types**：勾选以下选项
   - ✅ Authorization code
   - ✅ Refresh token

5. **Token format**：选择 `JWT`

### 步骤 4: 配置用户信息字段

在 **User info** 部分，确保以下字段已启用：

- ✅ `openid`
- ✅ `email`
- ✅ `profile`
- ✅ `name`

### 步骤 5: 配置 OIDC 发现端点

Casdoor 的 OIDC 发现端点格式为：
```
http://<casdoor-host>:<port>/.well-known/openid-configuration
```

例如：
```
http://192.168.100.100:18104/.well-known/openid-configuration
```

可以通过浏览器访问此 URL 验证配置是否正确，应该返回 JSON 格式的 OIDC 元数据。

### 步骤 6: 保存配置

点击 **Save** 按钮保存应用配置。

---

## Open WebUI 配置步骤

### 步骤 1: 编辑 .env 文件

在 Open WebUI 项目根目录找到 `.env` 文件，添加或修改以下配置：

```bash
# ============================================
# 认证方式配置
# ============================================
# 生产模式：仅支持 OAuth/SSO 登录（推荐）
ENABLE_PASSWORD_AUTH=false
ENABLE_LOGIN_FORM=false
ENABLE_SIGNUP=false

# 开发模式：同时支持密码登录和 OAuth 登录（调试用）
# ENABLE_PASSWORD_AUTH=true
# ENABLE_LOGIN_FORM=true
# ENABLE_SIGNUP=true

# ============================================
# SSO - Casdoor 配置
# ============================================
# OAuth 客户端 ID（从 Casdoor 应用配置中获取）
OAUTH_CLIENT_ID=34c0ccfa111882b676cf

# OAuth 客户端密钥（从 Casdoor 应用配置中获取）
OAUTH_CLIENT_SECRET=de3b07d7365fb356630835e189aaeeebd76ae2b7

# OpenID Connect 发现端点 URL
OPENID_PROVIDER_URL=http://192.168.100.100:18104/.well-known/openid-configuration

# OAuth 回调 URI（必须与 Casdoor 中配置的一致，使用后端端口）
OPENID_REDIRECT_URI=http://192.168.100.123:15700/oauth/oidc/callback

# OAuth 权限范围
OAUTH_SCOPES=openid email profile

# 登录页面显示的提供商名称
OAUTH_PROVIDER_NAME=沧澜SSO

# ============================================
# OAuth 注册和账户管理
# ============================================
# 允许通过 OAuth 自动注册新用户
ENABLE_OAUTH_SIGNUP=true

# 根据邮箱地址合并账户
OAUTH_MERGE_ACCOUNTS_BY_EMAIL=true

# 允许登录的邮箱域名（* 表示允许所有域名）
OAUTH_ALLOWED_DOMAINS=*

# Open WebUI 前端访问地址（使用前端端口）
WEBUI_URL=http://192.168.100.123:15701

# OAuth 登出后重定向 URL（使用前端端口）
WEBUI_AUTH_SIGNOUT_REDIRECT_URL=http://192.168.100.123:15701/auth

# ============================================
# 配置管理
# ============================================
# 禁用数据库持久化，强制使用 .env 文件
ENABLE_OAUTH_PERSISTENT_CONFIG=false

# 启动时不重置配置
RESET_CONFIG_ON_START=false
```

### 步骤 2: 配置说明

#### 必需配置项

| 配置项 | 说明 | 示例值 |
|--------|------|--------|
| `OAUTH_CLIENT_ID` | Casdoor 应用的客户端 ID | `34c0ccfa111882b676cf` |
| `OAUTH_CLIENT_SECRET` | Casdoor 应用的客户端密钥 | `de3b07d7365fb356630835e189aaeeebd76ae2b7` |
| `OPENID_PROVIDER_URL` | OIDC 发现端点 URL | `http://192.168.100.100:18104/.well-known/openid-configuration` |
| `OPENID_REDIRECT_URI` | OAuth 回调地址（后端端口） | `http://192.168.100.123:15700/oauth/oidc/callback` |
| `WEBUI_URL` | 前端访问地址（前端端口） | `http://192.168.100.123:15701` |

#### 可选配置项

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `OAUTH_SCOPES` | `openid email profile` | OAuth 权限范围 |
| `OAUTH_PROVIDER_NAME` | `SSO` | 登录按钮显示的名称 |
| `ENABLE_OAUTH_SIGNUP` | `false` | 是否允许自动注册 |
| `OAUTH_MERGE_ACCOUNTS_BY_EMAIL` | `false` | 是否根据邮箱合并账户 |
| `OAUTH_ALLOWED_DOMAINS` | `*` | 允许的邮箱域名 |
| `WEBUI_AUTH_SIGNOUT_REDIRECT_URL` | 无 | 登出后重定向地址 |

### 步骤 3: 重启 Open WebUI

配置完成后，重启 Open WebUI 服务使配置生效：

```bash
# 如果使用 Docker
docker-compose restart

# 如果使用 Python 直接运行
# 停止当前进程，然后重新启动
python -m open_webui.main
```

---

## 配置验证

### 验证步骤

1. **访问登录页面**
   - 打开浏览器访问：`http://192.168.100.123:15701`
   - 应该看到登录页面显示 "使用 沧澜SSO 登录" 按钮

2. **测试 OIDC 发现端点**
   ```bash
   curl http://192.168.100.100:18104/.well-known/openid-configuration
   ```
   应该返回包含以下字段的 JSON：
   ```json
   {
     "issuer": "http://192.168.100.100:18104",
     "authorization_endpoint": "...",
     "token_endpoint": "...",
     "userinfo_endpoint": "...",
     "jwks_uri": "...",
     ...
   }
   ```

3. **测试登录流程**
   - 点击 "使用 沧澜SSO 登录" 按钮
   - 应该跳转到 Casdoor 登录页面
   - 输入 Casdoor 用户凭证
   - 登录成功后应该跳转回 Open WebUI 并自动登录

4. **检查用户创建**
   - 首次登录的用户应该自动在 Open WebUI 中创建账户
   - 用户信息（邮箱、姓名）应该从 Casdoor 同步

### 验证清单

- [ ] OIDC 发现端点可访问
- [ ] 登录页面显示 SSO 登录按钮
- [ ] 点击按钮后跳转到 Casdoor
- [ ] Casdoor 登录成功
- [ ] 回调到 Open WebUI 成功
- [ ] 用户自动创建/登录
- [ ] 用户信息正确同步

---

## 高级配置

### 1. 限制登录域名

如果只允许特定域名的邮箱登录：

```bash
# 只允许 example.com 和 company.com 域名
OAUTH_ALLOWED_DOMAINS=example.com,company.com
```

### 2. 基于角色的访问控制

如果 Casdoor 中配置了用户角色，可以启用角色管理：

```bash
# 启用角色管理
ENABLE_OAUTH_ROLE_MANAGEMENT=true

# 角色字段名（Casdoor 默认为 roles）
OAUTH_ROLES_CLAIM=roles

# 允许登录的角色
OAUTH_ALLOWED_ROLES=user,member,employee

# 管理员角色
OAUTH_ADMIN_ROLES=admin,superadmin
```

**Casdoor 配置**：
1. 在 Casdoor 中创建角色（Roles）
2. 将角色分配给用户
3. 在应用配置中启用 `roles` scope

### 3. 用户组管理

如果需要同步 Casdoor 的用户组：

```bash
# 启用组管理
ENABLE_OAUTH_GROUP_MANAGEMENT=true

# 组字段名
OAUTH_GROUPS_CLAIM=groups

# 自动创建不存在的组
ENABLE_OAUTH_GROUP_CREATION=true

# 阻止同步的组（使用通配符）
OAUTH_BLOCKED_GROUPS=["system-*","temp-*"]
```

### 4. 自动更新头像

每次登录时更新用户头像：

```bash
OAUTH_UPDATE_PICTURE_ON_LOGIN=true
OAUTH_PICTURE_CLAIM=picture
```

### 5. 自定义用户信息字段

如果 Casdoor 使用了自定义字段：

```bash
# 自定义邮箱字段
OAUTH_EMAIL_CLAIM=email

# 自定义用户名字段
OAUTH_USERNAME_CLAIM=name
# 或使用其他字段
# OAUTH_USERNAME_CLAIM=preferred_username

# 自定义唯一标识符字段
OAUTH_SUB_CLAIM=sub
```

### 6. CORS 配置

如果前后端分离部署，需要配置 CORS：

```bash
# 允许的源地址（包含前端和后端端口）
CORS_ALLOW_ORIGIN=http://192.168.100.123:15701;http://192.168.100.123:15700
```

---

## 故障排查

### 常见问题

#### 1. 点击登录按钮后无反应

**可能原因**：
- `OPENID_PROVIDER_URL` 配置错误
- Casdoor 服务不可访问

**解决方法**：
```bash
# 测试 OIDC 端点
curl http://192.168.100.100:18104/.well-known/openid-configuration

# 检查网络连接
ping 192.168.100.100
```

#### 2. 回调失败：redirect_uri_mismatch

**可能原因**：
- Casdoor 中配置的回调 URL 与 `OPENID_REDIRECT_URI` 不一致
- 端口配置错误（前后端端口混淆）

**解决方法**：
1. 检查 Casdoor 应用配置中的 Redirect URLs
2. 确保使用**后端端口**（15700）而不是前端端口（15701）
3. 确保协议（http/https）、域名、端口、路径完全一致

#### 3. 回调成功但显示 "Invalid credentials"

**可能原因**：
- `OAUTH_CLIENT_ID` 或 `OAUTH_CLIENT_SECRET` 配置错误
- Casdoor 应用未启用正确的 Grant types

**解决方法**：
1. 重新检查 Client ID 和 Client Secret
2. 确认 Casdoor 应用启用了 "Authorization code" 和 "Refresh token"
3. 检查 Open WebUI 后端日志

#### 4. 登录成功但显示 JSON 而不是跳转

**可能原因**：
- `WEBUI_URL` 配置错误或未配置
- `WEBUI_AUTH_SIGNOUT_REDIRECT_URL` 配置错误

**解决方法**：
```bash
# 确保配置了前端 URL（使用前端端口）
WEBUI_URL=http://192.168.100.123:15701
WEBUI_AUTH_SIGNOUT_REDIRECT_URL=http://192.168.100.123:15701/auth
```

#### 5. 用户无法自动注册

**可能原因**：
- `ENABLE_OAUTH_SIGNUP` 未启用
- 邮箱域名不在 `OAUTH_ALLOWED_DOMAINS` 列表中

**解决方法**：
```bash
ENABLE_OAUTH_SIGNUP=true
OAUTH_ALLOWED_DOMAINS=*
```

#### 6. 端口配置混淆

**问题说明**：
这是最常见的配置错误。Open WebUI 前后端分离部署时有两个端口：

| 端口 | 用途 | 配置项 |
|------|------|--------|
| 15700 | 后端 API | `OPENID_REDIRECT_URI` |
| 15701 | 前端界面 | `WEBUI_URL`, `WEBUI_AUTH_SIGNOUT_REDIRECT_URL` |

**正确配置示例**：
```bash
# 后端端口 - 用于 OAuth 回调
OPENID_REDIRECT_URI=http://192.168.100.123:15700/oauth/oidc/callback

# 前端端口 - 用于用户访问
WEBUI_URL=http://192.168.100.123:15701
WEBUI_AUTH_SIGNOUT_REDIRECT_URL=http://192.168.100.123:15701/auth
```

### 调试方法

#### 1. 启用详细日志

```bash
# 在 .env 文件中添加
GLOBAL_LOG_LEVEL=DEBUG
```

#### 2. 查看后端日志

```bash
# Docker 部署
docker-compose logs -f open-webui

# Python 直接运行
# 查看控制台输出
```

#### 3. 检查浏览器控制台

1. 打开浏览器开发者工具（F12）
2. 切换到 Console 标签
3. 查看是否有 JavaScript 错误
4. 切换到 Network 标签查看网络请求

#### 4. 测试 OAuth 流程

使用 curl 手动测试 OAuth 流程：

```bash
# 1. 获取 OIDC 配置
curl http://192.168.100.100:18104/.well-known/openid-configuration

# 2. 测试授权端点（在浏览器中访问）
# 从上一步的响应中获取 authorization_endpoint
# 构造 URL 并在浏览器中访问

# 3. 检查回调端点是否可访问
curl http://192.168.100.123:15700/oauth/oidc/callback
```

### 日志分析

#### 成功登录的日志示例

```
INFO: OAuth callback successful for provider oidc
INFO: User authenticated: user@example.com
INFO: Stored OAuth session server-side for user xxx, provider oidc
```

#### 失败登录的日志示例

```
WARNING: OAuth callback failed: Missing state parameter
ERROR: OAuth callback error for user_id=xxx provider=oidc: Invalid credentials
ERROR: Token exchange failed: invalid_client
```

---

## 安全建议

### 1. 使用 HTTPS

生产环境中务必使用 HTTPS 协议：

```bash
# 生产环境配置示例
OPENID_PROVIDER_URL=https://casdoor.example.com/.well-known/openid-configuration
OPENID_REDIRECT_URI=https://openwebui.example.com/oauth/oidc/callback
WEBUI_URL=https://openwebui.example.com
```

### 2. 保护敏感信息

- 不要将 `.env` 文件提交到版本控制系统
- 使用 `.gitignore` 排除 `.env` 文件
- 定期更新 `OAUTH_CLIENT_SECRET`

```bash
# .gitignore
.env
.env.local
.env.*.local
```

### 3. 限制访问域名

不要使用 `*` 允许所有域名，而是明确指定：

```bash
# 推荐：限制特定域名
OAUTH_ALLOWED_DOMAINS=company.com,example.com

# 不推荐：允许所有域名（仅用于测试）
OAUTH_ALLOWED_DOMAINS=*
```

### 4. 配置防火墙

确保只有必要的端口对外开放：

```bash
# 只开放前端端口给用户
# 后端端口和 Casdoor 端口应该在内网访问
```

### 5. 定期审计

- 定期检查 OAuth 会话
- 监控异常登录行为
- 定期更新 Casdoor 和 Open WebUI 版本

### 6. 备份配置

定期备份 `.env` 文件和 Casdoor 配置：

```bash
# 备份 .env 文件（注意安全存储）
cp .env .env.backup.$(date +%Y%m%d)
```

---

## 完整配置示例

### 开发环境配置

```bash
# 认证配置 - 开发模式（同时支持密码和 SSO）
ENABLE_PASSWORD_AUTH=true
ENABLE_LOGIN_FORM=true
ENABLE_SIGNUP=true

# Casdoor SSO 配置
OAUTH_CLIENT_ID=34c0ccfa111882b676cf
OAUTH_CLIENT_SECRET=de3b07d7365fb356630835e189aaeeebd76ae2b7
OPENID_PROVIDER_URL=http://192.168.100.100:18104/.well-known/openid-configuration
OPENID_REDIRECT_URI=http://192.168.100.123:15700/oauth/oidc/callback
OAUTH_SCOPES=openid email profile
OAUTH_PROVIDER_NAME=Casdoor

# OAuth 注册配置
ENABLE_OAUTH_SIGNUP=true
OAUTH_MERGE_ACCOUNTS_BY_EMAIL=true
OAUTH_ALLOWED_DOMAINS=*
WEBUI_URL=http://192.168.100.123:15701
WEBUI_AUTH_SIGNOUT_REDIRECT_URL=http://192.168.100.123:15701/auth

# 配置管理
ENABLE_OAUTH_PERSISTENT_CONFIG=false
RESET_CONFIG_ON_START=false

# CORS 配置
CORS_ALLOW_ORIGIN=http://192.168.100.123:15701;http://192.168.100.123:15700

# 调试日志
GLOBAL_LOG_LEVEL=DEBUG
```

### 生产环境配置

```bash
# 认证配置 - 生产模式（仅支持 SSO）
ENABLE_PASSWORD_AUTH=false
ENABLE_LOGIN_FORM=false
ENABLE_SIGNUP=false

# Casdoor SSO 配置
OAUTH_CLIENT_ID=your-production-client-id
OAUTH_CLIENT_SECRET=your-production-client-secret
OPENID_PROVIDER_URL=https://casdoor.company.com/.well-known/openid-configuration
OPENID_REDIRECT_URI=https://openwebui.company.com/oauth/oidc/callback
OAUTH_SCOPES=openid email profile
OAUTH_PROVIDER_NAME=公司SSO

# OAuth 注册配置
ENABLE_OAUTH_SIGNUP=true
OAUTH_MERGE_ACCOUNTS_BY_EMAIL=true
OAUTH_ALLOWED_DOMAINS=company.com
WEBUI_URL=https://openwebui.company.com
WEBUI_AUTH_SIGNOUT_REDIRECT_URL=https://openwebui.company.com/auth

# 角色管理
ENABLE_OAUTH_ROLE_MANAGEMENT=true
OAUTH_ROLES_CLAIM=roles
OAUTH_ALLOWED_ROLES=user,member
OAUTH_ADMIN_ROLES=admin

# 配置管理
ENABLE_OAUTH_PERSISTENT_CONFIG=false
RESET_CONFIG_ON_START=false

# 生产日志级别
GLOBAL_LOG_LEVEL=INFO
```

---

## 参考资源

### 官方文档

- [Casdoor 官方文档](https://casdoor.org/docs/overview)
- [Casdoor OAuth 配置](https://casdoor.org/docs/application/config)
- [Open WebUI OAuth 配置](docs/OAUTH_SSO_CONFIGURATION.md)
- [OpenID Connect 规范](https://openid.net/connect/)

### 相关链接

- [Casdoor GitHub](https://github.com/casdoor/casdoor)
- [Open WebUI GitHub](https://github.com/open-webui/open-webui)
- [OAuth 2.0 规范](https://oauth.net/2/)

### 社区支持

- Casdoor 社区：https://casdoor.org/community
- Open WebUI 讨论区：https://github.com/open-webui/open-webui/discussions

---

## 附录

### A. 端口配置速查表

| 服务 | 端口 | 用途 | 配置项 |
|------|------|------|--------|
| Open WebUI 前端 | 15701 | 用户访问界面 | `WEBUI_URL` |
| Open WebUI 后端 | 15700 | API 服务 | `OPENID_REDIRECT_URI` |
| Casdoor | 18104 | 身份认证服务 | `OPENID_PROVIDER_URL` |

### B. 环境变量速查表

| 变量名 | 必需 | 默认值 | 说明 |
|--------|------|--------|------|
| `OAUTH_CLIENT_ID` | ✅ | 无 | Casdoor 客户端 ID |
| `OAUTH_CLIENT_SECRET` | ✅ | 无 | Casdoor 客户端密钥 |
| `OPENID_PROVIDER_URL` | ✅ | 无 | OIDC 发现端点 |
| `OPENID_REDIRECT_URI` | ✅ | 无 | OAuth 回调地址 |
| `WEBUI_URL` | ✅ | 无 | 前端访问地址 |
| `OAUTH_SCOPES` | ❌ | `openid email profile` | OAuth 权限范围 |
| `OAUTH_PROVIDER_NAME` | ❌ | `SSO` | 登录按钮显示名称 |
| `ENABLE_OAUTH_SIGNUP` | ❌ | `false` | 允许自动注册 |
| `OAUTH_MERGE_ACCOUNTS_BY_EMAIL` | ❌ | `false` | 根据邮箱合并账户 |
| `OAUTH_ALLOWED_DOMAINS` | ❌ | `*` | 允许的邮箱域名 |

### C. Casdoor 用户信息字段映射

| Casdoor 字段 | Open WebUI 字段 | 配置项 |
|--------------|-----------------|--------|
| `sub` | 用户唯一标识 | `OAUTH_SUB_CLAIM` |
| `email` | 邮箱 | `OAUTH_EMAIL_CLAIM` |
| `name` | 用户名 | `OAUTH_USERNAME_CLAIM` |
| `picture` | 头像 | `OAUTH_PICTURE_CLAIM` |
| `roles` | 角色 | `OAUTH_ROLES_CLAIM` |
| `groups` | 用户组 | `OAUTH_GROUPS_CLAIM` |

---

**文档版本**: 1.0
**最后更新**: 2026-02-05
**适用版本**: Open WebUI (支持 OIDC 的版本), Casdoor v1.x+
