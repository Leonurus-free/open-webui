# Open WebUI OAuth/SSO 单点登录配置指南

本文档详细说明了 Open WebUI 中所有 OAuth/SSO 相关的环境变量配置参数。

## 目录

- [通用配置](#通用配置)
- [OIDC/通用 OAuth 配置](#oidc通用-oauth-配置)
- [Google OAuth 配置](#google-oauth-配置)
- [Microsoft OAuth 配置](#microsoft-oauth-配置)
- [GitHub OAuth 配置](#github-oauth-配置)
- [飞书 OAuth 配置](#飞书-oauth-配置)
- [角色管理配置](#角色管理配置)
- [组管理配置](#组管理配置)
- [高级配置](#高级配置)
- [配置示例](#配置示例)

---

## 通用配置

### ENABLE_OAUTH_PERSISTENT_CONFIG
- **类型**: Boolean
- **默认值**: `false`
- **说明**: 是否启用 OAuth 配置的数据库持久化。设置为 `false` 时，强制从 `.env` 文件读取配置。
- **示例**:
  ```bash
  ENABLE_OAUTH_PERSISTENT_CONFIG=false
  ```

### ENABLE_OAUTH_SIGNUP
- **类型**: Boolean
- **默认值**: `false`
- **说明**: 是否允许通过 OAuth 登录的用户自动注册新账户。设置为 `true` 时，首次通过 OAuth 登录的用户会自动创建账户。
- **示例**:
  ```bash
  ENABLE_OAUTH_SIGNUP=true
  ```

### OAUTH_MERGE_ACCOUNTS_BY_EMAIL
- **类型**: Boolean
- **默认值**: `false`
- **说明**: 是否根据邮箱地址合并账户。当设置为 `true` 时，如果 OAuth 用户的邮箱与现有用户匹配，将关联到现有账户。
- **示例**:
  ```bash
  OAUTH_MERGE_ACCOUNTS_BY_EMAIL=true
  ```

### OAUTH_ALLOWED_DOMAINS
- **类型**: String (逗号分隔)
- **默认值**: `*`
- **说明**: 允许登录的邮箱域名白名单。使用 `*` 表示允许所有域名。可以指定多个域名，用逗号分隔。
- **示例**:
  ```bash
  OAUTH_ALLOWED_DOMAINS=*
  # 或限制特定域名
  OAUTH_ALLOWED_DOMAINS=example.com,company.com
  ```

### WEBUI_URL
- **类型**: String (URL)
- **默认值**: 无
- **说明**: Open WebUI 的前端访问地址。用于 OAuth 回调和重定向。
- **示例**:
  ```bash
  WEBUI_URL=http://192.168.100.123:15701
  ```

### ENABLE_OAUTH_EMAIL_FALLBACK
- **类型**: Boolean
- **默认值**: `false`
- **说明**: 当 OAuth 提供商未返回邮箱时，是否使用 `{provider}@{sub}.local` 格式生成邮箱。
- **示例**:
  ```bash
  ENABLE_OAUTH_EMAIL_FALLBACK=false
  ```

### ENABLE_OAUTH_ID_TOKEN_COOKIE
- **类型**: Boolean
- **默认值**: `true`
- **说明**: 是否在 Cookie 中存储 OAuth ID Token（用于兼容旧版前端）。
- **示例**:
  ```bash
  ENABLE_OAUTH_ID_TOKEN_COOKIE=true
  ```

---

## OIDC/通用 OAuth 配置

适用于 Casdoor、Keycloak、Authentik、Auth0 等支持 OpenID Connect 的身份提供商。

### OAUTH_CLIENT_ID
- **类型**: String
- **默认值**: 无
- **说明**: OAuth 客户端 ID，从身份提供商获取。
- **必需**: ✅ 是
- **示例**:
  ```bash
  OAUTH_CLIENT_ID=34c0ccfa111882b676cf
  ```

### OAUTH_CLIENT_SECRET
- **类型**: String
- **默认值**: 无
- **说明**: OAuth 客户端密钥，从身份提供商获取。
- **必需**: ✅ 是（除非使用 PKCE）
- **示例**:
  ```bash
  OAUTH_CLIENT_SECRET=de3b07d7365fb356630835e189aaeeebd76ae2b7
  ```

### OPENID_PROVIDER_URL
- **类型**: String (URL)
- **默认值**: 无
- **说明**: OpenID Connect 发现端点 URL，通常是 `/.well-known/openid-configuration` 路径。
- **必需**: ✅ 是
- **示例**:
  ```bash
  # Casdoor
  OPENID_PROVIDER_URL=http://192.168.100.100:18104/.well-known/openid-configuration

  # Keycloak
  OPENID_PROVIDER_URL=https://keycloak.example.com/realms/myrealm/.well-known/openid-configuration

  # Authentik
  OPENID_PROVIDER_URL=https://authentik.example.com/application/o/myapp/.well-known/openid-configuration
  ```

### OPENID_REDIRECT_URI
- **类型**: String (URL)
- **默认值**: 无
- **说明**: OAuth 回调 URI，必须在身份提供商中配置。通常是 `{BACKEND_URL}/oauth/oidc/callback`。
- **必需**: ✅ 是
- **示例**:
  ```bash
  OPENID_REDIRECT_URI=http://192.168.100.123:15700/oauth/oidc/callback
  ```

### OAUTH_SCOPES
- **类型**: String (空格分隔)
- **默认值**: `openid email profile`
- **说明**: 请求的 OAuth 权限范围。
- **示例**:
  ```bash
  OAUTH_SCOPES=openid email profile
  ```

### OAUTH_PROVIDER_NAME
- **类型**: String
- **默认值**: `SSO`
- **说明**: 在登录页面显示的提供商名称。
- **示例**:
  ```bash
  OAUTH_PROVIDER_NAME=沧澜SSO
  ```

### OAUTH_EMAIL_CLAIM
- **类型**: String
- **默认值**: `email`
- **说明**: 从 OAuth 用户信息中提取邮箱的字段名。
- **示例**:
  ```bash
  OAUTH_EMAIL_CLAIM=email
  ```

### OAUTH_USERNAME_CLAIM
- **类型**: String
- **默认值**: `name`
- **说明**: 从 OAuth 用户信息中提取用户名的字段名。
- **示例**:
  ```bash
  OAUTH_USERNAME_CLAIM=name
  # 或使用其他字段
  OAUTH_USERNAME_CLAIM=preferred_username
  ```

### OAUTH_PICTURE_CLAIM
- **类型**: String
- **默认值**: `picture`
- **说明**: 从 OAuth 用户信息中提取头像 URL 的字段名。
- **示例**:
  ```bash
  OAUTH_PICTURE_CLAIM=picture
  ```

### OAUTH_SUB_CLAIM
- **类型**: String
- **默认值**: `sub`
- **说明**: 从 OAuth 用户信息中提取唯一标识符的字段名。通常不需要修改。
- **示例**:
  ```bash
  OAUTH_SUB_CLAIM=sub
  ```

### OAUTH_TIMEOUT
- **类型**: Integer (秒)
- **默认值**: 无
- **说明**: OAuth 请求的超时时间。
- **示例**:
  ```bash
  OAUTH_TIMEOUT=30
  ```

### OAUTH_TOKEN_ENDPOINT_AUTH_METHOD
- **类型**: String
- **默认值**: 自动检测
- **说明**: Token 端点的认证方法。可选值：`client_secret_basic`、`client_secret_post`、`none`。
- **示例**:
  ```bash
  OAUTH_TOKEN_ENDPOINT_AUTH_METHOD=client_secret_post
  ```

### OAUTH_CODE_CHALLENGE_METHOD
- **类型**: String
- **默认值**: 无
- **说明**: PKCE 代码挑战方法。设置为 `S256` 启用 PKCE（无需 client_secret）。
- **示例**:
  ```bash
  OAUTH_CODE_CHALLENGE_METHOD=S256
  ```

### OAUTH_UPDATE_PICTURE_ON_LOGIN
- **类型**: Boolean
- **默认值**: `false`
- **说明**: 是否在每次登录时更新用户头像。
- **示例**:
  ```bash
  OAUTH_UPDATE_PICTURE_ON_LOGIN=true
  ```

### OAUTH_AUDIENCE
- **类型**: String
- **默认值**: 无
- **说明**: OAuth 请求的 audience 参数（某些提供商需要）。
- **示例**:
  ```bash
  OAUTH_AUDIENCE=https://api.example.com
  ```

### OAUTH_ACCESS_TOKEN_REQUEST_INCLUDE_CLIENT_ID
- **类型**: Boolean
- **默认值**: `false`
- **说明**: 在 access token 请求中是否包含 client_id（某些提供商需要）。
- **示例**:
  ```bash
  OAUTH_ACCESS_TOKEN_REQUEST_INCLUDE_CLIENT_ID=true
  ```

---

## Google OAuth 配置

### GOOGLE_CLIENT_ID
- **类型**: String
- **默认值**: 无
- **说明**: Google OAuth 客户端 ID。
- **示例**:
  ```bash
  GOOGLE_CLIENT_ID=123456789-abcdefg.apps.googleusercontent.com
  ```

### GOOGLE_CLIENT_SECRET
- **类型**: String
- **默认值**: 无
- **说明**: Google OAuth 客户端密钥。
- **示例**:
  ```bash
  GOOGLE_CLIENT_SECRET=GOCSPX-xxxxxxxxxxxxx
  ```

### GOOGLE_OAUTH_SCOPE
- **类型**: String (空格分隔)
- **默认值**: `openid email profile`
- **说明**: Google OAuth 权限范围。
- **示例**:
  ```bash
  GOOGLE_OAUTH_SCOPE=openid email profile
  ```

### GOOGLE_REDIRECT_URI
- **类型**: String (URL)
- **默认值**: 无
- **说明**: Google OAuth 回调 URI。
- **示例**:
  ```bash
  GOOGLE_REDIRECT_URI=http://localhost:8080/oauth/google/callback
  ```

---

## Microsoft OAuth 配置

### MICROSOFT_CLIENT_ID
- **类型**: String
- **默认值**: 无
- **说明**: Microsoft/Azure AD 应用程序（客户端）ID。
- **示例**:
  ```bash
  MICROSOFT_CLIENT_ID=12345678-1234-1234-1234-123456789012
  ```

### MICROSOFT_CLIENT_SECRET
- **类型**: String
- **默认值**: 无
- **说明**: Microsoft/Azure AD 客户端密钥。
- **示例**:
  ```bash
  MICROSOFT_CLIENT_SECRET=xxxxxxxxxxxxxxxxxxxxx
  ```

### MICROSOFT_CLIENT_TENANT_ID
- **类型**: String
- **默认值**: 无
- **说明**: Azure AD 租户 ID。使用 `common` 支持多租户。
- **示例**:
  ```bash
  MICROSOFT_CLIENT_TENANT_ID=common
  # 或特定租户
  MICROSOFT_CLIENT_TENANT_ID=12345678-1234-1234-1234-123456789012
  ```

### MICROSOFT_CLIENT_LOGIN_BASE_URL
- **类型**: String (URL)
- **默认值**: `https://login.microsoftonline.com`
- **说明**: Microsoft 登录基础 URL。
- **示例**:
  ```bash
  MICROSOFT_CLIENT_LOGIN_BASE_URL=https://login.microsoftonline.com
  ```

### MICROSOFT_CLIENT_PICTURE_URL
- **类型**: String (URL)
- **默认值**: `https://graph.microsoft.com/v1.0/me/photo/$value`
- **说明**: Microsoft Graph API 头像 URL。
- **示例**:
  ```bash
  MICROSOFT_CLIENT_PICTURE_URL=https://graph.microsoft.com/v1.0/me/photo/$value
  ```

### MICROSOFT_OAUTH_SCOPE
- **类型**: String (空格分隔)
- **默认值**: `openid email profile`
- **说明**: Microsoft OAuth 权限范围。
- **示例**:
  ```bash
  MICROSOFT_OAUTH_SCOPE=openid email profile
  ```

### MICROSOFT_REDIRECT_URI
- **类型**: String (URL)
- **默认值**: 无
- **说明**: Microsoft OAuth 回调 URI。
- **示例**:
  ```bash
  MICROSOFT_REDIRECT_URI=http://localhost:8080/oauth/microsoft/callback
  ```

---

## GitHub OAuth 配置

### GITHUB_CLIENT_ID
- **类型**: String
- **默认值**: 无
- **说明**: GitHub OAuth App 客户端 ID。
- **示例**:
  ```bash
  GITHUB_CLIENT_ID=Iv1.1234567890abcdef
  ```

### GITHUB_CLIENT_SECRET
- **类型**: String
- **默认值**: 无
- **说明**: GitHub OAuth App 客户端密钥。
- **示例**:
  ```bash
  GITHUB_CLIENT_SECRET=1234567890abcdef1234567890abcdef12345678
  ```

### GITHUB_CLIENT_SCOPE
- **类型**: String (空格分隔)
- **默认值**: `user:email`
- **说明**: GitHub OAuth 权限范围。
- **示例**:
  ```bash
  GITHUB_CLIENT_SCOPE=user:email
  ```

### GITHUB_CLIENT_REDIRECT_URI
- **类型**: String (URL)
- **默认值**: 无
- **说明**: GitHub OAuth 回调 URI。
- **示例**:
  ```bash
  GITHUB_CLIENT_REDIRECT_URI=http://localhost:8080/oauth/github/callback
  ```

---

## 飞书 OAuth 配置

### FEISHU_CLIENT_ID
- **类型**: String
- **默认值**: 无
- **说明**: 飞书应用 App ID。
- **示例**:
  ```bash
  FEISHU_CLIENT_ID=cli_a1234567890abcde
  ```

### FEISHU_CLIENT_SECRET
- **类型**: String
- **默认值**: 无
- **说明**: 飞书应用 App Secret。
- **示例**:
  ```bash
  FEISHU_CLIENT_SECRET=1234567890abcdef1234567890abcdef
  ```

### FEISHU_OAUTH_SCOPE
- **类型**: String (空格分隔)
- **默认值**: `contact:user.base:readonly`
- **说明**: 飞书 OAuth 权限范围。
- **示例**:
  ```bash
  FEISHU_OAUTH_SCOPE=contact:user.base:readonly
  ```

### FEISHU_REDIRECT_URI
- **类型**: String (URL)
- **默认值**: 无
- **说明**: 飞书 OAuth 回调 URI。
- **示例**:
  ```bash
  FEISHU_REDIRECT_URI=http://localhost:8080/oauth/feishu/callback
  ```

---

## 角色管理配置

### ENABLE_OAUTH_ROLE_MANAGEMENT
- **类型**: Boolean
- **默认值**: `false`
- **说明**: 是否启用基于 OAuth 的角色管理。启用后，用户角色将根据 OAuth 提供商返回的角色信息自动分配。
- **示例**:
  ```bash
  ENABLE_OAUTH_ROLE_MANAGEMENT=true
  ```

### OAUTH_ROLES_CLAIM
- **类型**: String
- **默认值**: `roles`
- **说明**: 从 OAuth 用户信息中提取角色列表的字段名。支持嵌套字段（使用 `.` 分隔）。
- **示例**:
  ```bash
  OAUTH_ROLES_CLAIM=roles
  # 或嵌套字段
  OAUTH_ROLES_CLAIM=resource_access.open-webui.roles
  ```

### OAUTH_ROLES_SEPARATOR
- **类型**: String
- **默认值**: `,`
- **说明**: 当角色以字符串形式返回时，用于分隔多个角色的分隔符。
- **示例**:
  ```bash
  OAUTH_ROLES_SEPARATOR=,
  ```

### OAUTH_ALLOWED_ROLES
- **类型**: String (分隔符分隔)
- **默认值**: `user,admin`
- **说明**: 允许登录的角色列表。用户必须拥有列表中的至少一个角色才能登录，并被分配为 `user` 角色。
- **示例**:
  ```bash
  OAUTH_ALLOWED_ROLES=user,member,employee
  ```

### OAUTH_ADMIN_ROLES
- **类型**: String (分隔符分隔)
- **默认值**: `admin`
- **说明**: 管理员角色列表。拥有这些角色的用户将被分配为 `admin` 角色。
- **示例**:
  ```bash
  OAUTH_ADMIN_ROLES=admin,superadmin,administrator
  ```

---

## 组管理配置

### ENABLE_OAUTH_GROUP_MANAGEMENT
- **类型**: Boolean
- **默认值**: `false`
- **说明**: 是否启用基于 OAuth 的组管理。启用后，用户的组成员关系将根据 OAuth 提供商返回的组信息自动同步。
- **示例**:
  ```bash
  ENABLE_OAUTH_GROUP_MANAGEMENT=true
  ```

### OAUTH_GROUPS_CLAIM
- **类型**: String
- **默认值**: `groups`
- **说明**: 从 OAuth 用户信息中提取组列表的字段名。支持嵌套字段（使用 `.` 分隔）。
- **示例**:
  ```bash
  OAUTH_GROUPS_CLAIM=groups
  # 或嵌套字段
  OAUTH_GROUPS_CLAIM=user.groups
  ```

### OAUTH_GROUPS_SEPARATOR
- **类型**: String
- **默认值**: `;`
- **说明**: 当组以字符串形式返回时，用于分隔多个组的分隔符。
- **示例**:
  ```bash
  OAUTH_GROUPS_SEPARATOR=;
  ```

### ENABLE_OAUTH_GROUP_CREATION
- **类型**: Boolean
- **默认值**: `false`
- **说明**: 是否自动创建 OAuth 中存在但 Open WebUI 中不存在的组。
- **示例**:
  ```bash
  ENABLE_OAUTH_GROUP_CREATION=true
  ```

### OAUTH_BLOCKED_GROUPS
- **类型**: String (JSON 数组)
- **默认值**: `[]`
- **说明**: 阻止同步的组列表。支持精确匹配、通配符（`*`、`?`）和正则表达式。
- **示例**:
  ```bash
  # 精确匹配
  OAUTH_BLOCKED_GROUPS=["admin-group","system-group"]

  # 通配符
  OAUTH_BLOCKED_GROUPS=["system-*","temp-*"]

  # 正则表达式
  OAUTH_BLOCKED_GROUPS=["^admin-.*",".*-internal$"]
  ```

---

## 高级配置

### RESET_CONFIG_ON_START
- **类型**: Boolean
- **默认值**: `false`
- **说明**: 是否在启动时重置配置到环境变量值。
- **示例**:
  ```bash
  RESET_CONFIG_ON_START=false
  ```

### WEBUI_AUTH_SIGNOUT_REDIRECT_URL
- **类型**: String (URL)
- **默认值**: 无
- **说明**: 用户登出后的重定向 URL。
- **示例**:
  ```bash
  WEBUI_AUTH_SIGNOUT_REDIRECT_URL=http://192.168.100.123:15701/auth
  ```

### OAUTH_CLIENT_INFO_ENCRYPTION_KEY
- **类型**: String
- **默认值**: `WEBUI_SECRET_KEY` 的值
- **说明**: 用于加密 OAuth 客户端信息的密钥。
- **示例**:
  ```bash
  OAUTH_CLIENT_INFO_ENCRYPTION_KEY=your-encryption-key-here
  ```

### OAUTH_SESSION_TOKEN_ENCRYPTION_KEY
- **类型**: String
- **默认值**: `WEBUI_SECRET_KEY` 的值
- **说明**: 用于加密 OAuth 会话令牌的密钥。
- **示例**:
  ```bash
  OAUTH_SESSION_TOKEN_ENCRYPTION_KEY=your-encryption-key-here
  ```

---

## 配置示例

### 示例 1: Casdoor SSO 配置

```bash
# 基础配置
ENABLE_OAUTH_PERSISTENT_CONFIG=false
ENABLE_OAUTH_SIGNUP=true
OAUTH_MERGE_ACCOUNTS_BY_EMAIL=true
OAUTH_ALLOWED_DOMAINS=*
WEBUI_URL=http://192.168.100.123:15701

# Casdoor OIDC 配置
OAUTH_CLIENT_ID=34c0ccfa111882b676cf
OAUTH_CLIENT_SECRET=de3b07d7365fb356630835e189aaeeebd76ae2b7
OPENID_PROVIDER_URL=http://192.168.100.100:18104/.well-known/openid-configuration
OPENID_REDIRECT_URI=http://192.168.100.123:15700/oauth/oidc/callback
OAUTH_SCOPES=openid email profile
OAUTH_EMAIL_CLAIM=email
OAUTH_USERNAME_CLAIM=name
OAUTH_PROVIDER_NAME=沧澜SSO

# 登出配置
WEBUI_AUTH_SIGNOUT_REDIRECT_URL=http://192.168.100.123:15701/auth
```

### 示例 2: Keycloak 配置（带角色和组管理）

```bash
# 基础配置
ENABLE_OAUTH_SIGNUP=true
OAUTH_MERGE_ACCOUNTS_BY_EMAIL=true
OAUTH_ALLOWED_DOMAINS=example.com
WEBUI_URL=https://openwebui.example.com

# Keycloak OIDC 配置
OAUTH_CLIENT_ID=open-webui
OAUTH_CLIENT_SECRET=your-client-secret
OPENID_PROVIDER_URL=https://keycloak.example.com/realms/myrealm/.well-known/openid-configuration
OPENID_REDIRECT_URI=https://openwebui.example.com/oauth/oidc/callback
OAUTH_SCOPES=openid email profile
OAUTH_PROVIDER_NAME=Keycloak

# 角色管理
ENABLE_OAUTH_ROLE_MANAGEMENT=true
OAUTH_ROLES_CLAIM=resource_access.open-webui.roles
OAUTH_ALLOWED_ROLES=user,member
OAUTH_ADMIN_ROLES=admin

# 组管理
ENABLE_OAUTH_GROUP_MANAGEMENT=true
OAUTH_GROUPS_CLAIM=groups
ENABLE_OAUTH_GROUP_CREATION=true
OAUTH_BLOCKED_GROUPS=["system-*","temp-*"]
```

### 示例 3: Google OAuth 配置

```bash
# 基础配置
ENABLE_OAUTH_SIGNUP=true
OAUTH_ALLOWED_DOMAINS=example.com,company.com
WEBUI_URL=https://openwebui.example.com

# Google OAuth 配置
GOOGLE_CLIENT_ID=123456789-abcdefg.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=GOCSPX-xxxxxxxxxxxxx
GOOGLE_OAUTH_SCOPE=openid email profile
GOOGLE_REDIRECT_URI=https://openwebui.example.com/oauth/google/callback
```

### 示例 4: 多提供商配置

```bash
# 基础配置
ENABLE_OAUTH_SIGNUP=true
OAUTH_MERGE_ACCOUNTS_BY_EMAIL=true
OAUTH_ALLOWED_DOMAINS=*
WEBUI_URL=https://openwebui.example.com

# Google
GOOGLE_CLIENT_ID=123456789-abcdefg.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=GOCSPX-xxxxxxxxxxxxx
GOOGLE_REDIRECT_URI=https://openwebui.example.com/oauth/google/callback

# GitHub
GITHUB_CLIENT_ID=Iv1.1234567890abcdef
GITHUB_CLIENT_SECRET=1234567890abcdef1234567890abcdef12345678
GITHUB_CLIENT_REDIRECT_URI=https://openwebui.example.com/oauth/github/callback

# Microsoft
MICROSOFT_CLIENT_ID=12345678-1234-1234-1234-123456789012
MICROSOFT_CLIENT_SECRET=xxxxxxxxxxxxxxxxxxxxx
MICROSOFT_CLIENT_TENANT_ID=common
MICROSOFT_REDIRECT_URI=https://openwebui.example.com/oauth/microsoft/callback
```

---

## 故障排查

### 常见问题

1. **OAuth 登录失败，显示 "Invalid credentials"**
   - 检查 `OAUTH_CLIENT_ID` 和 `OAUTH_CLIENT_SECRET` 是否正确
   - 确认 `OPENID_PROVIDER_URL` 可以访问
   - 验证身份提供商中的回调 URL 配置

2. **回调后显示 "OAuth callback failed: Missing state parameter"**
   - 可能是会话过期，尝试重新登录
   - 检查 Cookie 设置和浏览器隐私设置

3. **用户无法注册**
   - 确认 `ENABLE_OAUTH_SIGNUP=true`
   - 检查 `OAUTH_ALLOWED_DOMAINS` 是否包含用户的邮箱域名

4. **角色/组同步不工作**
   - 确认 `ENABLE_OAUTH_ROLE_MANAGEMENT` 或 `ENABLE_OAUTH_GROUP_MANAGEMENT` 已启用
   - 检查 `OAUTH_ROLES_CLAIM` 或 `OAUTH_GROUPS_CLAIM` 字段名是否正确
   - 查看身份提供商返回的用户信息中是否包含相应字段

5. **端口配置问题**
   - 前后端分离部署时，`OPENID_REDIRECT_URI` 应使用后端端口
   - `WEBUI_URL` 应使用前端访问地址
   - 确保 `CORS_ALLOW_ORIGIN` 包含所有需要的端口

### 调试方法

1. **启用详细日志**
   ```bash
   GLOBAL_LOG_LEVEL=DEBUG
   ```

2. **测试 OpenID 配置端点**
   ```bash
   curl http://your-provider/.well-known/openid-configuration
   ```

3. **检查回调 URL**
   - 确保身份提供商中配置的回调 URL 与 `OPENID_REDIRECT_URI` 完全一致
   - 注意协议（http/https）、域名、端口和路径都必须匹配

---

## 安全建议

1. **使用 HTTPS**: 生产环境中务必使用 HTTPS 协议
2. **保护密钥**: 不要将 `OAUTH_CLIENT_SECRET` 等敏感信息提交到版本控制系统
3. **限制域名**: 使用 `OAUTH_ALLOWED_DOMAINS` 限制允许登录的邮箱域名
4. **定期更新**: 定期更新客户端密钥和加密密钥
5. **最小权限**: 只请求必需的 OAuth scopes

---

## 参考资源

- [Open WebUI 官方文档](https://docs.openwebui.com/)
- [OpenID Connect 规范](https://openid.net/connect/)
- [OAuth 2.0 规范](https://oauth.net/2/)
- [Casdoor 文档](https://casdoor.org/docs/overview)
- [Keycloak 文档](https://www.keycloak.org/documentation)

---

**文档版本**: 1.0
**最后更新**: 2026-02-04
