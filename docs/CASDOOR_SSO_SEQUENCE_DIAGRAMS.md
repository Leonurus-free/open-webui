# Casdoor 单点登录时序图

本文档详细展示了 Open WebUI 与 Casdoor 集成的各种 OAuth/OIDC 流程时序图。

## 目录

- [1. 首次登录流程（OAuth 授权码流程）](#1-首次登录流程oauth-授权码流程)
- [2. 已有账户登录流程](#2-已有账户登录流程)
- [3. Token 刷新流程](#3-token-刷新流程)
- [4. 登出流程](#4-登出流程)
- [5. 角色和组同步流程](#5-角色和组同步流程)
- [6. 错误处理流程](#6-错误处理流程)

---

## 1. 首次登录流程（OAuth 授权码流程）

这是用户第一次通过 Casdoor 登录 Open WebUI 的完整流程，使用 OAuth 2.0 授权码模式。

```mermaid
sequenceDiagram
    participant User as 用户浏览器
    participant Frontend as Open WebUI 前端<br/>(15701)
    participant Backend as Open WebUI 后端<br/>(15700)
    participant Casdoor as Casdoor 服务<br/>(18104)
    participant DB as 数据库

    Note over User,DB: 用户首次登录流程

    User->>Frontend: 1. 访问 Open WebUI<br/>http://192.168.100.123:15701
    Frontend->>User: 2. 显示登录页面<br/>包含 "使用沧澜SSO登录" 按钮

    User->>Frontend: 3. 点击 SSO 登录按钮
    Frontend->>Backend: 4. 请求 OAuth 授权<br/>GET /oauth/oidc/authorize

    Backend->>Casdoor: 5. 获取 OIDC 配置<br/>GET /.well-known/openid-configuration
    Casdoor->>Backend: 6. 返回 OIDC 元数据<br/>(authorization_endpoint, token_endpoint, etc.)

    Backend->>Backend: 7. 生成 state 和 code_verifier<br/>构造授权 URL
    Backend->>User: 8. 重定向到 Casdoor 授权页面<br/>302 Redirect

    User->>Casdoor: 9. 访问 Casdoor 授权页面<br/>带参数: client_id, redirect_uri, scope, state
    Casdoor->>User: 10. 显示 Casdoor 登录页面

    User->>Casdoor: 11. 输入用户名和密码
    Casdoor->>Casdoor: 12. 验证用户凭证

    alt 认证成功
        Casdoor->>User: 13. 重定向到回调 URL<br/>带授权码 (code) 和 state
        User->>Backend: 14. 访问回调 URL<br/>GET /oauth/oidc/callback?code=xxx&state=xxx

        Backend->>Backend: 15. 验证 state 参数

        Backend->>Casdoor: 16. 交换授权码获取 Token<br/>POST /token<br/>参数: code, client_id, client_secret
        Casdoor->>Backend: 17. 返回 Token<br/>{access_token, id_token, refresh_token}

        Backend->>Casdoor: 18. 获取用户信息<br/>GET /userinfo<br/>Bearer access_token
        Casdoor->>Backend: 19. 返回用户信息<br/>{sub, email, name, picture}

        Backend->>DB: 20. 查询用户是否存在<br/>根据 provider + sub
        DB->>Backend: 21. 用户不存在

        Backend->>Backend: 22. 检查 ENABLE_OAUTH_SIGNUP=true

        Backend->>DB: 23. 创建新用户<br/>email, name, profile_image_url, role
        DB->>Backend: 24. 返回新用户信息

        Backend->>Backend: 25. 生成 JWT Token<br/>包含 user.id

        Backend->>DB: 26. 存储 OAuth Session<br/>user_id, provider, token, expires_at
        DB->>Backend: 27. 返回 session_id

        Backend->>User: 28. 重定向到前端<br/>302 http://192.168.100.123:15701/auth<br/>Set-Cookie: token=jwt_token<br/>Set-Cookie: oauth_session_id=xxx

        User->>Frontend: 29. 访问前端页面<br/>携带 Cookie
        Frontend->>Frontend: 30. 读取 JWT Token<br/>解析用户信息
        Frontend->>User: 31. 显示主界面<br/>用户已登录

    else 认证失败
        Casdoor->>User: 显示错误信息
    end
```

### 关键步骤说明

| 步骤 | 说明 | 涉及配置 |
|------|------|----------|
| 5-6 | 获取 OIDC 配置 | `OPENID_PROVIDER_URL` |
| 9 | 授权请求参数 | `OAUTH_CLIENT_ID`, `OPENID_REDIRECT_URI`, `OAUTH_SCOPES` |
| 16 | Token 交换 | `OAUTH_CLIENT_ID`, `OAUTH_CLIENT_SECRET` |
| 18-19 | 获取用户信息 | `OAUTH_EMAIL_CLAIM`, `OAUTH_USERNAME_CLAIM` |
| 22-24 | 自动注册用户 | `ENABLE_OAUTH_SIGNUP` |
| 28 | 重定向到前端 | `WEBUI_URL` |

---

## 2. 已有账户登录流程

用户已经在 Open WebUI 中有账户，通过 Casdoor 再次登录的流程。

```mermaid
sequenceDiagram
    participant User as 用户浏览器
    participant Frontend as Open WebUI 前端<br/>(15701)
    participant Backend as Open WebUI 后端<br/>(15700)
    participant Casdoor as Casdoor 服务<br/>(18104)
    participant DB as 数据库

    Note over User,DB: 已有账户登录流程

    User->>Frontend: 1. 访问登录页面
    Frontend->>User: 2. 显示 SSO 登录按钮

    User->>Frontend: 3. 点击 SSO 登录
    Frontend->>Backend: 4. GET /oauth/oidc/authorize

    Backend->>User: 5. 重定向到 Casdoor<br/>(步骤同首次登录)

    User->>Casdoor: 6. 访问 Casdoor

    alt Casdoor Session 有效
        Note over Casdoor: 用户在 Casdoor 中已登录
        Casdoor->>User: 7. 直接返回授权码<br/>无需再次输入密码
    else Casdoor Session 过期
        Casdoor->>User: 7. 显示登录页面
        User->>Casdoor: 8. 输入凭证
        Casdoor->>User: 9. 返回授权码
    end

    User->>Backend: 10. 回调 /oauth/oidc/callback?code=xxx

    Backend->>Casdoor: 11. 交换 Token
    Casdoor->>Backend: 12. 返回 Token

    Backend->>Casdoor: 13. 获取用户信息
    Casdoor->>Backend: 14. 返回用户信息

    Backend->>DB: 15. 查询用户<br/>根据 provider + sub
    DB->>Backend: 16. 返回已存在的用户

    alt 启用角色管理
        Backend->>Backend: 17. 检查用户角色<br/>ENABLE_OAUTH_ROLE_MANAGEMENT=true
        Backend->>DB: 18. 更新用户角色<br/>如果角色发生变化
    end

    alt 启用头像更新
        Backend->>Backend: 19. 检查头像更新<br/>OAUTH_UPDATE_PICTURE_ON_LOGIN=true
        Backend->>Casdoor: 20. 下载新头像
        Casdoor->>Backend: 21. 返回头像数据
        Backend->>DB: 22. 更新用户头像
    end

    Backend->>Backend: 23. 生成新的 JWT Token

    Backend->>DB: 24. 清理旧的 OAuth Session
    Backend->>DB: 25. 创建新的 OAuth Session
    DB->>Backend: 26. 返回 session_id

    Backend->>User: 27. 重定向到前端<br/>Set-Cookie: token, oauth_session_id

    User->>Frontend: 28. 访问前端
    Frontend->>User: 29. 显示主界面
```

### 关键差异

- **步骤 16**：用户已存在，不需要创建新账户
- **步骤 17-18**：如果启用角色管理，会更新用户角色
- **步骤 19-22**：如果启用头像更新，会同步最新头像
- **步骤 24-25**：清理旧 Session，创建新 Session

---

## 3. Token 刷新流程

当 Access Token 即将过期时，使用 Refresh Token 获取新的 Access Token。

```mermaid
sequenceDiagram
    participant Frontend as Open WebUI 前端
    participant Backend as Open WebUI 后端
    participant Casdoor as Casdoor 服务
    participant DB as 数据库

    Note over Frontend,DB: Token 刷新流程

    Frontend->>Backend: 1. API 请求<br/>携带 JWT Token
    Backend->>Backend: 2. 验证 JWT Token<br/>提取 user_id

    Backend->>DB: 3. 查询 OAuth Session<br/>根据 user_id 和 provider
    DB->>Backend: 4. 返回 Session 信息<br/>{token, expires_at}

    Backend->>Backend: 5. 检查 Token 是否即将过期<br/>expires_at - now < 5分钟

    alt Token 即将过期
        Backend->>Casdoor: 6. 获取 OIDC 配置<br/>获取 token_endpoint
        Casdoor->>Backend: 7. 返回 token_endpoint

        Backend->>Casdoor: 8. 刷新 Token<br/>POST /token<br/>grant_type=refresh_token<br/>refresh_token=xxx<br/>client_id, client_secret

        alt 刷新成功
            Casdoor->>Backend: 9. 返回新的 Token<br/>{access_token, refresh_token, expires_in}

            Backend->>Backend: 10. 计算新的 expires_at<br/>now + expires_in

            Backend->>DB: 11. 更新 OAuth Session<br/>新的 token 和 expires_at
            DB->>Backend: 12. 更新成功

            Backend->>Frontend: 13. 返回 API 响应<br/>使用新的 access_token

        else 刷新失败
            Casdoor->>Backend: 9. 返回错误<br/>invalid_grant / invalid_token

            Backend->>DB: 10. 删除无效的 Session
            DB->>Backend: 11. 删除成功

            Backend->>Frontend: 12. 返回 401 Unauthorized<br/>需要重新登录

            Frontend->>Frontend: 13. 清除本地 Token
            Frontend->>Frontend: 14. 重定向到登录页面
        end

    else Token 仍然有效
        Backend->>Frontend: 6. 直接返回 API 响应<br/>使用现有 access_token
    end
```

### 刷新策略

- **触发时机**：Token 过期前 5 分钟自动刷新
- **刷新方式**：使用 Refresh Token 向 Casdoor 请求新的 Access Token
- **失败处理**：如果刷新失败，删除 Session 并要求用户重新登录

---

## 4. 登出流程

用户主动登出 Open WebUI 的流程。

```mermaid
sequenceDiagram
    participant User as 用户浏览器
    participant Frontend as Open WebUI 前端<br/>(15701)
    participant Backend as Open WebUI 后端<br/>(15700)
    participant DB as 数据库
    participant Casdoor as Casdoor 服务<br/>(可选)

    Note over User,Casdoor: 用户登出流程

    User->>Frontend: 1. 点击登出按钮
    Frontend->>Backend: 2. POST /api/v1/auths/signout<br/>携带 JWT Token 和 oauth_session_id

    Backend->>Backend: 3. 验证 JWT Token<br/>提取 user_id

    Backend->>DB: 4. 查询 OAuth Session<br/>根据 session_id 和 user_id
    DB->>Backend: 5. 返回 Session 信息

    Backend->>DB: 6. 删除 OAuth Session
    DB->>Backend: 7. 删除成功

    Backend->>Frontend: 8. 返回登出成功<br/>Clear-Cookie: token<br/>Clear-Cookie: oauth_session_id<br/>Clear-Cookie: oauth_id_token

    Frontend->>Frontend: 9. 清除本地存储<br/>localStorage, sessionStorage

    alt 配置了登出重定向
        Frontend->>User: 10. 重定向到登录页面<br/>http://192.168.100.123:15701/auth<br/>(WEBUI_AUTH_SIGNOUT_REDIRECT_URL)
    else 未配置重定向
        Frontend->>User: 10. 显示登录页面
    end

    Note over User,Casdoor: 可选：Casdoor 端登出

    opt 需要 Casdoor 端也登出
        User->>Casdoor: 11. 访问 Casdoor 登出端点<br/>GET /logout
        Casdoor->>Casdoor: 12. 清除 Casdoor Session
        Casdoor->>User: 13. 重定向到登录页面
    end
```

### 登出说明

1. **Open WebUI 端登出**：
   - 删除数据库中的 OAuth Session
   - 清除浏览器 Cookie（token, oauth_session_id）
   - 清除前端本地存储

2. **Casdoor 端登出**（可选）：
   - 如果需要同时登出 Casdoor，用户需要手动访问 Casdoor 的登出端点
   - 或者在前端实现自动跳转到 Casdoor 登出页面

3. **重定向配置**：
   - 使用 `WEBUI_AUTH_SIGNOUT_REDIRECT_URL` 配置登出后的跳转地址
   - 通常设置为前端的登录页面

---

## 5. 角色和组同步流程

当启用角色管理和组管理时，登录过程中会同步用户的角色和组信息。

```mermaid
sequenceDiagram
    participant Backend as Open WebUI 后端
    participant Casdoor as Casdoor 服务
    participant DB as 数据库

    Note over Backend,DB: 角色和组同步流程（登录时）

    Backend->>Casdoor: 1. 获取用户信息<br/>包含 roles 和 groups
    Casdoor->>Backend: 2. 返回用户信息<br/>{sub, email, name, roles: [...], groups: [...]}

    rect rgb(200, 220, 250)
        Note over Backend,DB: 角色同步

        alt ENABLE_OAUTH_ROLE_MANAGEMENT=true
            Backend->>Backend: 3. 提取角色信息<br/>根据 OAUTH_ROLES_CLAIM

            Backend->>Backend: 4. 检查用户角色<br/>roles: ["user", "developer"]

            alt 用户是第一个用户
                Backend->>Backend: 5. 分配 admin 角色
            else 用户角色在 OAUTH_ADMIN_ROLES 中
                Backend->>Backend: 5. 分配 admin 角色<br/>例如: roles 包含 "admin"
            else 用户角色在 OAUTH_ALLOWED_ROLES 中
                Backend->>Backend: 5. 分配 user 角色<br/>例如: roles 包含 "user"
            else 用户角色不匹配
                Backend->>Backend: 5. 使用默认角色<br/>DEFAULT_USER_ROLE
            end

            Backend->>DB: 6. 更新用户角色
            DB->>Backend: 7. 更新成功
        end
    end

    rect rgb(220, 250, 220)
        Note over Backend,DB: 组同步

        alt ENABLE_OAUTH_GROUP_MANAGEMENT=true
            Backend->>Backend: 8. 提取组信息<br/>根据 OAUTH_GROUPS_CLAIM

            Backend->>Backend: 9. 过滤阻止的组<br/>OAUTH_BLOCKED_GROUPS: ["system-*"]

            Backend->>DB: 10. 查询用户当前的组
            DB->>Backend: 11. 返回当前组列表<br/>["group-a", "group-b"]

            Backend->>DB: 12. 查询所有可用的组
            DB->>Backend: 13. 返回所有组列表

            alt ENABLE_OAUTH_GROUP_CREATION=true
                loop 对于每个 OAuth 组
                    alt 组不存在
                        Backend->>DB: 14. 创建新组<br/>name, description, permissions
                        DB->>Backend: 15. 返回新组信息
                    end
                end
            end

            loop 移除用户不再属于的组
                Backend->>DB: 16. 从组中移除用户<br/>如果组不在 OAuth groups 中
                DB->>Backend: 17. 移除成功
            end

            loop 添加用户到新组
                Backend->>DB: 18. 将用户添加到组<br/>如果组在 OAuth groups 中
                DB->>Backend: 19. 添加成功
            end
        end
    end

    Backend->>Backend: 20. 完成角色和组同步
```

### 角色管理配置

```bash
# 启用角色管理
ENABLE_OAUTH_ROLE_MANAGEMENT=true

# Casdoor 中的角色字段名
OAUTH_ROLES_CLAIM=roles

# 允许登录的角色（分配为 user）
OAUTH_ALLOWED_ROLES=user,member,employee

# 管理员角色（分配为 admin）
OAUTH_ADMIN_ROLES=admin,superadmin
```

### 组管理配置

```bash
# 启用组管理
ENABLE_OAUTH_GROUP_MANAGEMENT=true

# Casdoor 中的组字段名
OAUTH_GROUPS_CLAIM=groups

# 自动创建不存在的组
ENABLE_OAUTH_GROUP_CREATION=true

# 阻止同步的组（支持通配符和正则）
OAUTH_BLOCKED_GROUPS=["system-*","temp-*","^admin-.*"]
```

---

## 6. 错误处理流程

展示各种错误场景的处理流程。

```mermaid
sequenceDiagram
    participant User as 用户浏览器
    participant Frontend as Open WebUI 前端
    participant Backend as Open WebUI 后端
    participant Casdoor as Casdoor 服务

    Note over User,Casdoor: 错误处理流程

    rect rgb(255, 220, 220)
        Note over User,Casdoor: 场景 1: 回调 URL 不匹配

        User->>Backend: 1. 点击 SSO 登录
        Backend->>User: 2. 重定向到 Casdoor<br/>redirect_uri=http://wrong-url/callback

        User->>Casdoor: 3. 访问 Casdoor
        Casdoor->>Casdoor: 4. 验证 redirect_uri
        Casdoor->>User: 5. 返回错误<br/>error=redirect_uri_mismatch

        User->>User: 6. 显示错误页面<br/>"回调地址不匹配"
    end

    rect rgb(255, 240, 220)
        Note over User,Casdoor: 场景 2: Client Secret 错误

        User->>Backend: 1. OAuth 回调<br/>code=xxx
        Backend->>Casdoor: 2. 交换 Token<br/>client_secret=wrong_secret

        Casdoor->>Casdoor: 3. 验证 client_secret
        Casdoor->>Backend: 4. 返回错误<br/>401 Unauthorized<br/>error=invalid_client

        Backend->>Backend: 5. 构造错误消息<br/>"OAuth callback failed: invalid_client"

        Backend->>User: 6. 重定向到前端<br/>http://...15701/auth?error=OAuth+callback+failed

        User->>Frontend: 7. 访问前端
        Frontend->>User: 8. 显示错误提示<br/>"OAuth 回调失败: invalid_client"
    end

    rect rgb(255, 255, 220)
        Note over User,Casdoor: 场景 3: 用户邮箱域名不允许

        User->>Backend: 1. OAuth 回调成功
        Backend->>Casdoor: 2. 获取用户信息
        Casdoor->>Backend: 3. 返回用户信息<br/>email=user@blocked-domain.com

        Backend->>Backend: 4. 检查邮箱域名<br/>OAUTH_ALLOWED_DOMAINS=company.com
        Backend->>Backend: 5. 域名不在白名单中

        Backend->>User: 6. 重定向到前端<br/>error=Invalid+credentials

        User->>Frontend: 7. 访问前端
        Frontend->>User: 8. 显示错误<br/>"无效的凭证"
    end

    rect rgb(240, 255, 240)
        Note over User,Casdoor: 场景 4: 用户未启用自动注册

        User->>Backend: 1. OAuth 回调成功<br/>首次登录
        Backend->>Casdoor: 2. 获取用户信息
        Casdoor->>Backend: 3. 返回用户信息

        Backend->>Backend: 4. 查询用户不存在
        Backend->>Backend: 5. 检查 ENABLE_OAUTH_SIGNUP=false

        Backend->>User: 6. 返回 403 Forbidden<br/>error=Access+prohibited

        User->>Frontend: 7. 访问前端
        Frontend->>User: 8. 显示错误<br/>"访问被禁止"
    end

    rect rgb(240, 240, 255)
        Note over User,Casdoor: 场景 5: State 参数丢失或不匹配

        User->>Backend: 1. OAuth 回调<br/>code=xxx (缺少 state)

        Backend->>Backend: 2. 验证 state 参数<br/>Session 中没有对应的 state

        Backend->>Backend: 3. 抛出异常<br/>KeyError: 'state'

        Backend->>Backend: 4. 构造错误消息<br/>"Missing state parameter (session may have expired)"

        Backend->>User: 5. 重定向到前端<br/>error=OAuth+callback+failed

        User->>Frontend: 6. 访问前端
        Frontend->>User: 7. 显示错误<br/>"OAuth 回调失败: 缺少 state 参数（会话可能已过期）"
    end
```

### 常见错误码

| 错误码 | 说明 | 解决方法 |
|--------|------|----------|
| `redirect_uri_mismatch` | 回调 URL 不匹配 | 检查 Casdoor 和 `OPENID_REDIRECT_URI` 配置 |
| `invalid_client` | Client ID 或 Secret 错误 | 检查 `OAUTH_CLIENT_ID` 和 `OAUTH_CLIENT_SECRET` |
| `invalid_grant` | 授权码无效或过期 | 重新发起登录流程 |
| `access_denied` | 用户拒绝授权 | 用户需要同意授权 |
| `Missing state parameter` | State 参数丢失 | 会话过期，重新登录 |
| `Invalid credentials` | 邮箱域名不允许或其他认证失败 | 检查 `OAUTH_ALLOWED_DOMAINS` 配置 |
| `Access prohibited` | 未启用自动注册 | 设置 `ENABLE_OAUTH_SIGNUP=true` |

---

## 附录：完整的数据流

### OAuth Token 结构

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "id_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "scope": "openid email profile"
}
```

### Casdoor 用户信息结构

```json
{
  "sub": "admin",
  "iss": "http://192.168.100.100:18104",
  "aud": "34c0ccfa111882b676cf",
  "name": "管理员",
  "preferred_username": "admin",
  "email": "admin@example.com",
  "email_verified": true,
  "picture": "http://192.168.100.100:18104/files/avatar.png",
  "roles": ["admin", "user"],
  "groups": ["administrators", "developers"]
}
```

### Open WebUI OAuth Session 结构

```json
{
  "id": "session-uuid",
  "user_id": "user-uuid",
  "provider": "oidc",
  "token": {
    "access_token": "...",
    "refresh_token": "...",
    "id_token": "...",
    "expires_in": 3600,
    "issued_at": 1704067200,
    "expires_at": 1704070800
  },
  "created_at": 1704067200,
  "updated_at": 1704067200
}
```

---

## 参考资源

- [OAuth 2.0 授权码流程](https://oauth.net/2/grant-types/authorization-code/)
- [OpenID Connect 核心规范](https://openid.net/specs/openid-connect-core-1_0.html)
- [Casdoor OAuth 文档](https://casdoor.org/docs/how-to-connect/oauth)
- [Open WebUI OAuth 配置](OAUTH_SSO_CONFIGURATION.md)
- [Casdoor SSO 配置指南](CASDOOR_SSO_CONFIGURATION.md)

---

**文档版本**: 1.0
**最后更新**: 2026-02-05
**适用版本**: Open WebUI (支持 OIDC), Casdoor v1.x+
