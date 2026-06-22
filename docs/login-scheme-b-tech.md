# 登录方案 B 技术实现说明：独立 SSO 登录中心，主应用代理承接

## 1. 文档信息

| 项目 | 内容 |
| --- | --- |
| 文档名称 | 登录方案 B 技术实现说明 |
| 方案名称 | 独立 SSO 登录中心，主应用代理承接 |
| 适用项目 | 使用 `tsu-cli` 工具模板生成的 MFE 微前端项目 |
| 日期 | 2026-06-21 |
| 目标读者 | 前端、后端、架构、测试、安全、数据、产品 |

---

## 2. 方案定位

### 2.1 核心原则

登录方案 B 的核心是：

> 账号认证能力由独立 SSO 登录中心提供，但在 MFE 前端产品体验上，用户仍然只通过主应用发起登录，子应用不直接登录。

也就是说：

- SSO 登录中心负责真实认证、账号体系、会话、token 签发。
- 主应用负责承接登录入口、登录跳转、登录回调、登录态初始化、登录态分发。
- 子应用只消费主应用提供的登录态，不实现独立登录页，不直接接入 SSO 登录页。

### 2.2 适用场景

该方案适合以下场景：

1. 未来不止一个 MFE 应用，需要多个站点、多个端共享账号体系。
2. 后续可能接入移动端、管理后台、开放平台、合作方系统。
3. 需要支持 OAuth2、OIDC、企业 SSO、第三方登录等标准认证能力。
4. 登录、风控、账号安全需要独立演进，不希望强绑定在主应用中。
5. 当前产品体验仍要求：登录都在主应用触发，子应用不能自行登录。

---

## 3. 总体架构

### 3.1 架构角色

| 模块 | 职责 |
| --- | --- |
| 主应用 Shell | 提供登录入口、触发 SSO 跳转、处理登录回调、维护前端登录态、向子应用分发用户状态 |
| 子应用 | 声明登录策略、消费登录态、触发主应用登录、展示业务页面 |
| SSO 登录中心 | 统一账号认证、登录页、注册、第三方登录、token 签发、session 管理 |
| Auth BFF/API Gateway | 前端认证代理层，处理 token 换取、刷新、用户信息、权限聚合 |
| Auth SDK | 主应用侧认证封装与子应用桥接能力 |
| Config Center | 子应用登录策略、路由权限策略、灰度配置 |
| Tracking SDK | 登录、登出、权限拦截、用户行为事件采集 |
| Event Collector | 接收行为数据，用于 BI 和审计分析 |

### 3.2 推荐架构图

```txt
┌─────────────────────────────────────────────────────────────┐
│                        Browser                              │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                   主应用 Shell                         │  │
│  │                                                       │  │
│  │  Login Button / Login Modal Trigger                   │  │
│  │  Auth SDK                                             │  │
│  │  Route Guard                                          │  │
│  │  User Context                                         │  │
│  │  Event Bus                                            │  │
│  │                                                       │  │
│  │   ┌──────────────┐   ┌──────────────┐                 │  │
│  │   │ 子应用 A      │   │ 子应用 B      │                 │  │
│  │   │ consume auth │   │ requireLogin │                 │  │
│  │   └──────────────┘   └──────────────┘                 │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ redirect / callback
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     SSO 登录中心                             │
│  登录页 / 注册 / 第三方登录 / MFA / 风控 / 授权码签发          │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ token exchange / user info
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Auth BFF / API Gateway                    │
│  token 换取 / token 刷新 / 用户信息 / 权限 / session 管理      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 Event Collector / BI / Audit                 │
│  login_start / login_success / login_fail / logout / pv      │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. 登录流程设计

### 4.1 未登录访问公开页面

```txt
用户访问 URL
  ↓
主应用加载
  ↓
Auth SDK 初始化
  ↓
检查本地 session/token
  ↓
无登录态
  ↓
生成 visitorId / sessionId
  ↓
允许访问公开子应用页面
  ↓
Tracking SDK 上报 page_view
```

说明：

- 公开页面不强制登录。
- 即使未登录，也需要生成匿名访客标识。
- 匿名用户行为后续可在登录成功后通过 alias 规则关联到 userId。

### 4.2 未登录访问强登录页面

```txt
用户访问强登录页面
  ↓
主应用加载路由配置
  ↓
判断 route.meta.authRequired = true
  ↓
Auth SDK 判断未登录
  ↓
记录当前目标地址 redirect_uri
  ↓
主应用触发 requireLogin
  ↓
跳转到 SSO 登录中心
  ↓
用户在 SSO 完成登录
  ↓
SSO 回跳主应用 callback 地址，携带 code/state
  ↓
主应用调用 Auth BFF 换取 token/session
  ↓
主应用拉取用户信息和权限
  ↓
广播 auth_change: logged_in
  ↓
回跳 redirect_uri
  ↓
子应用读取最新登录态并渲染页面
```

### 4.3 子应用触发操作级登录

适用于收藏、评论、保存、下载、提交表单等场景。

```txt
用户在子应用点击需要登录的操作
  ↓
子应用调用 authBridge.requireLogin({ reason, from })
  ↓
主应用接管登录流程
  ↓
主应用跳转 SSO 或展示登录承接页
  ↓
用户完成登录
  ↓
主应用同步登录态
  ↓
Promise resolve(user)
  ↓
子应用继续执行原业务操作
```

子应用示例：

```ts
async function handleFavorite() {
  const user = await authBridge.requireLogin({
    reason: 'favorite',
    from: location.href,
  })

  await favoriteApi.create({ userId: user.userId, itemId })
  tracking.track('favorite_success', { itemId })
}
```

### 4.4 登录回调流程

```txt
SSO 登录中心回跳：
/main/auth/callback?code=xxx&state=yyy

主应用处理：
1. 校验 state，防止 CSRF。
2. 使用 code 请求 Auth BFF。
3. Auth BFF 与 SSO 完成 token exchange。
4. Auth BFF 设置安全 cookie 或返回短期 access token。
5. 主应用调用 /api/auth/me 获取用户信息。
6. 主应用调用 /api/auth/permissions 获取权限。
7. 主应用写入内存态，必要时写入安全存储。
8. 主应用广播登录成功事件。
9. 主应用回跳 redirect_uri。
```

### 4.5 退出登录流程

```txt
用户点击主应用退出登录
  ↓
主应用调用 /api/auth/logout
  ↓
Auth BFF 清理本系统 session/token
  ↓
可选：跳转或调用 SSO logout 清理 SSO 会话
  ↓
主应用清理本地用户态
  ↓
广播 auth_change: logged_out
  ↓
所有子应用进入匿名态
  ↓
如果当前页面强登录，则跳转未登录拦截页或首页
```

---

## 5. OAuth2/OIDC 推荐实现

### 5.1 推荐授权模式

Web 主应用推荐使用：

```txt
Authorization Code Flow + PKCE
```

原因：

- 比 implicit flow 更安全。
- 适合浏览器前端与 BFF 组合模式。
- 可以降低授权码被截获后的风险。
- 便于后续兼容 OIDC。

### 5.2 关键参数

| 参数 | 说明 |
| --- | --- |
| client_id | 主应用在 SSO 注册的客户端 ID |
| redirect_uri | 主应用登录回调地址 |
| response_type | 固定为 `code` |
| scope | 请求权限范围，例如 `openid profile email` |
| state | 防 CSRF，同时可关联登录上下文 |
| code_challenge | PKCE 挑战码 |
| code_challenge_method | 推荐 `S256` |
| nonce | OIDC 场景用于防重放 |

### 5.3 登录跳转 URL 示例

```txt
https://sso.example.com/oauth/authorize
  ?client_id=mfe-main-app
  &redirect_uri=https%3A%2F%2Fapp.example.com%2Fauth%2Fcallback
  &response_type=code
  &scope=openid%20profile%20email
  &state=STATE_VALUE
  &code_challenge=CODE_CHALLENGE
  &code_challenge_method=S256
```

### 5.4 回调地址建议

主应用统一维护登录回调地址：

```txt
/auth/callback
```

不建议每个子应用拥有自己的 SSO 回调地址。

原因：

- 避免子应用直接接入认证。
- 降低 SSO 客户端配置复杂度。
- 登录回跳、token 换取、错误处理更统一。
- 便于记录统一登录事件。

---

## 6. Token 与 Session 管理

### 6.1 推荐存储策略

优先推荐：

```txt
BFF + HttpOnly Secure SameSite Cookie
```

即：

- 浏览器不直接长期持有 refresh token。
- refresh token 尽量只存在服务端或 HttpOnly Cookie 中。
- 前端只持有短生命周期 access token，甚至仅通过 cookie 鉴权。
- 用户信息和权限在主应用内存中维护。

### 6.2 Token 类型

| Token | 用途 | 建议生命周期 | 存储建议 |
| --- | --- | --- | --- |
| authorization code | 登录回调换 token | 一次性，极短 | 不落地 |
| access token | API 访问 | 5-30 分钟 | 内存或 HttpOnly Cookie |
| refresh token | 刷新 access token | 7-30 天，按业务定 | 服务端/session 或 HttpOnly Cookie |
| id token | OIDC 用户身份声明 | 短期 | 后端校验，前端谨慎使用 |

### 6.3 刷新策略

```txt
主应用启动
  ↓
调用 /api/auth/me
  ↓
如果 401
  ↓
调用 /api/auth/refresh
  ↓
刷新成功：重新拉取用户信息
刷新失败：进入匿名态
```

建议：

- access token 即将过期前主动刷新。
- 多标签页只允许一个标签执行刷新，避免并发刷新风暴。
- 刷新失败后清理本地状态并广播退出。
- 子应用不得自行刷新 token。

---

## 7. 主应用 Auth SDK 设计

### 7.1 主应用 SDK 接口

```ts
export interface MainAppAuthSDK {
  init(): Promise<AuthState>
  login(options?: LoginOptions): Promise<void>
  handleCallback(params: CallbackParams): Promise<LoginResult>
  logout(options?: LogoutOptions): Promise<void>
  refreshToken(): Promise<void>
  getUser(): User | null
  getToken(): string | null
  getAuthState(): AuthState
  isAuthenticated(): boolean
  hasPermission(permission: string): boolean
  requireLogin(options?: RequireLoginOptions): Promise<User>
  onAuthChange(callback: (state: AuthState) => void): () => void
}
```

### 7.2 子应用桥接接口

```ts
export interface SubAppAuthBridge {
  getUser(): User | null
  getAuthState(): AuthState
  isAuthenticated(): boolean
  hasPermission(permission: string): boolean
  requireLogin(options?: RequireLoginOptions): Promise<User>
  onAuthChange(callback: (state: AuthState) => void): () => void
}
```

### 7.3 AuthState 类型建议

```ts
export interface AuthState {
  status: 'initializing' | 'anonymous' | 'authenticated' | 'expired' | 'error'
  user: User | null
  permissions: string[]
  roles: string[]
  tokenExpiresAt?: number
  loginSource?: string
  lastUpdatedAt: number
}
```

### 7.4 User 类型建议

```ts
export interface User {
  userId: string
  nickname?: string
  avatar?: string
  email?: string
  phone?: string
  roles: string[]
  permissions: string[]
  createdAt?: string
  lastLoginAt?: string
  userLevel?: string
}
```

---

## 8. 主应用与子应用通信

### 8.1 推荐通信方式

根据微前端框架能力选择一种或组合使用：

| 方式 | 说明 | 适用场景 |
| --- | --- | --- |
| props 注入 | 主应用加载子应用时传入 `authBridge` | 子应用挂载初始化 |
| 事件总线 | `auth_change`、`login_required` 等事件 | 登录态变更广播 |
| 全局上下文 | `window.__MFE_AUTH__` | 简化接入，需注意类型和隔离 |
| BroadcastChannel | 多标签页同步登录态 | 多窗口状态同步 |
| CustomEvent | 简单浏览器事件通信 | 框架无关通信 |

### 8.2 子应用接收 props 示例

```ts
export interface SubAppProps {
  appId: string
  authBridge: SubAppAuthBridge
  tracking: TrackingSDK
}

export function mount(props: SubAppProps) {
  const { authBridge } = props

  authBridge.onAuthChange((state) => {
    // 更新子应用用户状态
  })
}
```

### 8.3 事件名称建议

| 事件名 | 方向 | 说明 |
| --- | --- | --- |
| `auth:init` | 主应用内部 | Auth SDK 初始化 |
| `auth:login_required` | 子应用 → 主应用 | 子应用请求登录 |
| `auth:login_start` | 主应用 | 开始登录 |
| `auth:login_success` | 主应用 → 子应用 | 登录成功 |
| `auth:login_fail` | 主应用 → 子应用 | 登录失败 |
| `auth:logout` | 主应用 → 子应用 | 退出登录 |
| `auth:state_change` | 主应用 → 子应用 | 登录态变化 |
| `auth:permission_denied` | 子应用/主应用 | 权限不足 |

### 8.4 子应用禁止事项

子应用禁止：

1. 直接展示登录表单。
2. 直接跳转 SSO 授权地址。
3. 直接处理 SSO callback。
4. 直接调用 `/api/auth/login`、`/api/auth/refresh`。
5. 直接保存 refresh token。
6. 自行判断 token 是否过期并刷新。
7. 将用户敏感信息写入不受控存储。

---

## 9. 路由守卫与强登录

### 9.1 应用级配置

```ts
export const appMeta = {
  appId: 'user-center',
  appName: '用户中心',
  auth: {
    required: true,
    fallback: 'main-app-login',
  },
}
```

### 9.2 路由级配置

```ts
export const routes = [
  {
    path: '/profile',
    component: ProfilePage,
    meta: {
      authRequired: true,
      permission: 'user.profile.read',
      eventPageName: 'profile_page',
    },
  },
  {
    path: '/landing',
    component: LandingPage,
    meta: {
      authRequired: false,
      eventPageName: 'landing_page',
    },
  },
]
```

### 9.3 主应用路由守卫伪代码

```ts
async function beforeRouteEnter(to: Route) {
  const authState = auth.getAuthState()
  const routeMeta = resolveRouteMeta(to)

  if (!routeMeta.authRequired) {
    return true
  }

  if (!authState.user) {
    await auth.requireLogin({
      redirectUri: to.fullPath,
      sourceAppId: routeMeta.appId,
      reason: 'route_guard',
    })
    return false
  }

  if (routeMeta.permission && !auth.hasPermission(routeMeta.permission)) {
    tracking.track('permission_denied', {
      appId: routeMeta.appId,
      routePath: to.path,
      permission: routeMeta.permission,
    })
    return '/403'
  }

  return true
}
```

---

## 10. 后端接口设计

### 10.1 主应用调用接口

| 接口 | 方法 | 说明 |
| --- | --- | --- |
| `/api/auth/sso/url` | GET | 获取 SSO 授权跳转 URL |
| `/api/auth/callback` | POST | 使用 code/state 换取本系统 session/token |
| `/api/auth/me` | GET | 获取当前用户信息 |
| `/api/auth/permissions` | GET | 获取当前用户权限 |
| `/api/auth/refresh` | POST | 刷新登录态 |
| `/api/auth/logout` | POST | 退出当前系统 |
| `/api/auth/sso/logout-url` | GET | 获取 SSO 退出地址，可选 |

### 10.2 `/api/auth/sso/url` 请求示例

```http
GET /api/auth/sso/url?redirect_uri=https%3A%2F%2Fapp.example.com%2Fuser%2Fprofile&source_app_id=user-center
```

响应示例：

```json
{
  "authorizeUrl": "https://sso.example.com/oauth/authorize?...",
  "state": "st_abc123"
}
```

### 10.3 `/api/auth/callback` 请求示例

```json
{
  "code": "auth_code_xxx",
  "state": "st_abc123",
  "redirectUri": "https://app.example.com/auth/callback"
}
```

响应示例：

```json
{
  "user": {
    "userId": "u_10001",
    "nickname": "Tom",
    "roles": ["user"],
    "permissions": ["user.profile.read"]
  },
  "expiresAt": 1782016800000
}
```

说明：

- 如果使用 HttpOnly Cookie，响应体可以不返回 access token。
- token 推荐由服务端通过 `Set-Cookie` 写入。

### 10.4 Cookie 建议

```http
Set-Cookie: sid=xxx; Path=/; HttpOnly; Secure; SameSite=Lax; Max-Age=604800
```

如果主应用和 SSO 跨站点，需要结合实际域名策略评估 `SameSite=None; Secure`。

---

## 11. 数据埋点设计

### 11.1 登录相关事件

| 事件名 | 触发方 | 触发时机 | 关键字段 |
| --- | --- | --- | --- |
| `login_required` | 子应用/主应用 | 访问强登录路由或操作 | appId、routePath、reason |
| `login_start` | 主应用 | 开始跳转 SSO | sourceAppId、redirectUri、loginScene |
| `sso_redirect` | 主应用 | 跳转 SSO 前 | provider、stateId |
| `sso_callback` | 主应用 | 收到 SSO 回调 | success、errorCode |
| `login_success` | 主应用 | token 换取成功且用户信息拉取成功 | userId、isNewUser、sourceAppId |
| `login_fail` | 主应用 | 登录失败 | reason、errorCode、sourceAppId |
| `logout` | 主应用 | 用户退出 | sourceAppId |
| `token_refresh_success` | 主应用 | token 刷新成功 | expiresAt |
| `token_refresh_fail` | 主应用 | token 刷新失败 | reason |
| `permission_denied` | 主应用/子应用 | 权限不足 | permission、routePath、appId |

### 11.2 登录前后用户关联

登录成功后需要执行：

```ts
tracking.alias(visitorId, user.userId)
tracking.identify(user.userId, {
  roles: user.roles,
  userLevel: user.userLevel,
})
```

### 11.3 公共字段

登录相关事件必须包含：

- `eventId`
- `eventName`
- `eventTime`
- `appId`
- `sourceAppId`
- `routePath`
- `redirectUri`
- `visitorId`
- `sessionId`
- `userId`
- `loginScene`
- `utmSource`
- `utmMedium`
- `utmCampaign`

---

## 12. 安全设计

### 12.1 必须实现

1. 使用 Authorization Code Flow + PKCE。
2. 校验 `state`，防止 CSRF。
3. OIDC 场景校验 `nonce`。
4. token exchange 只允许后端或 BFF 执行。
5. refresh token 不暴露给子应用。
6. 子应用不直接访问 SSO callback。
7. 登录回跳地址必须做白名单校验，防止开放重定向。
8. 用户权限必须由后端接口再次校验，前端仅用于体验控制。
9. 登录、刷新、短信验证码等接口需要限流。
10. 关键登录失败原因需要打点和安全审计。

### 12.2 redirect_uri 白名单

允许：

```txt
https://app.example.com/auth/callback
https://staging-app.example.com/auth/callback
http://localhost:3000/auth/callback
```

禁止：

```txt
https://unknown.example.com/auth/callback
https://evil.com/callback
javascript:alert(1)
```

### 12.3 敏感数据处理

- 手机号、邮箱前端展示必须脱敏。
- access token 不写入 localStorage。
- refresh token 不暴露给 JS。
- 用户权限字段只包含前端必要权限。
- 埋点中不要上传明文手机号、邮箱、身份证、详细地址等敏感信息。

---

## 13. 异常与兜底

### 13.1 SSO 不可用

| 场景 | 处理 |
| --- | --- |
| 公开页面 | 允许匿名访问，展示非阻塞提示或不提示 |
| 强登录页面 | 展示主应用统一错误页，提示稍后重试 |
| 操作级登录 | 保留用户操作上下文，提示登录服务暂不可用 |

### 13.2 callback 失败

可能原因：

- code 过期。
- state 不匹配。
- token exchange 失败。
- 用户被禁用。
- 网络异常。

处理方式：

1. 清理本次登录上下文。
2. 上报 `login_fail`。
3. 展示友好错误。
4. 提供重新登录入口。
5. 不直接进入子应用强登录页面。

### 13.3 token 刷新失败

处理方式：

1. 主应用清理登录态。
2. 广播 `auth:state_change`。
3. 子应用切换为匿名态。
4. 当前页面如果要求强登录，则跳转主应用登录承接页。

---

## 14. 多标签页同步

### 14.1 推荐方案

使用 `BroadcastChannel` 同步登录态变化。

```ts
const channel = new BroadcastChannel('mfe-auth')

channel.postMessage({
  type: 'auth:state_change',
  payload: auth.getAuthState(),
})

channel.onmessage = (event) => {
  if (event.data.type === 'auth:state_change') {
    auth.applyRemoteState(event.data.payload)
  }
}
```

### 14.2 同步事件

- 登录成功。
- 退出登录。
- token 刷新成功。
- token 刷新失败。
- 用户权限变更。

---

## 15. 开发与本地调试

### 15.1 本地开发模式

本地开发建议支持两种模式：

| 模式 | 说明 |
| --- | --- |
| mock auth | 不连接 SSO，使用本地 mock 用户 |
| real sso | 连接测试 SSO 环境，走真实登录流程 |

### 15.2 Mock 用户示例

```ts
export const mockUser: User = {
  userId: 'mock_user_001',
  nickname: 'Mock User',
  roles: ['user'],
  permissions: ['user.profile.read'],
}
```

### 15.3 环境变量建议

```txt
VITE_AUTH_MODE=mock | sso
VITE_SSO_BASE_URL=https://sso.example.com
VITE_AUTH_CALLBACK_URL=http://localhost:3000/auth/callback
VITE_AUTH_CLIENT_ID=mfe-main-app
```

---

## 16. 测试用例建议

### 16.1 登录流程

1. 匿名用户访问公开页面，不触发登录。
2. 匿名用户访问强登录页面，跳转 SSO。
3. SSO 登录成功后回跳原页面。
4. SSO 登录失败后展示错误页。
5. 登录成功后刷新页面仍保持登录态。
6. 多标签页登录后其他标签同步登录态。
7. 多标签页退出后其他标签同步退出。

### 16.2 子应用行为

1. 子应用不包含独立登录页面。
2. 子应用触发 `requireLogin` 后由主应用接管。
3. 子应用读取用户信息正确。
4. 子应用权限不足时展示无权限页。
5. 子应用不能直接刷新 token。

### 16.3 安全测试

1. state 不匹配时拒绝登录。
2. 非白名单 redirect_uri 被拒绝。
3. code 重放失败。
4. token 过期后刷新成功。
5. refresh 失败后进入匿名态。
6. 未登录直接调用强权限接口返回 401。
7. 无权限用户调用接口返回 403。

### 16.4 埋点测试

1. `login_start` 上报成功。
2. `sso_callback` 上报成功。
3. `login_success` 包含 userId、visitorId、sessionId。
4. 登录失败包含 errorCode 和 sourceAppId。
5. 子应用触发登录时包含 reason 和 routePath。
6. 登录成功后执行 visitorId 到 userId 的 alias。

---

## 17. 里程碑拆分

### 阶段 1：主应用认证骨架

- 主应用 Auth SDK 初始化。
- `/auth/callback` 页面。
- `/api/auth/me` 接入。
- 登录态全局上下文。
- 子应用 authBridge 注入。

### 阶段 2：SSO 登录闭环

- `/api/auth/sso/url`。
- SSO authorize 跳转。
- callback code/state 处理。
- token exchange。
- 登录成功回跳。
- 登录失败兜底。

### 阶段 3：强登录与权限

- 应用级强登录。
- 路由级强登录。
- 操作级登录拦截。
- 权限判断。
- 403 页面。

### 阶段 4：安全与稳定性

- PKCE。
- state/nonce 校验。
- redirect_uri 白名单。
- token 刷新。
- 多标签页同步。
- 登录异常监控。

### 阶段 5：数据与 BI

- 登录相关事件埋点。
- visitorId 与 userId 关联。
- 登录转化漏斗。
- 子应用登录来源分析。
- 登录失败原因分析。

---

## 18. 推荐结论

登录方案 B 适合作为中长期账号体系演进方案。它将真实认证能力沉淀到独立 SSO 登录中心，同时保持当前 MFE 产品原则：

1. 用户只从主应用触发登录。
2. 子应用不做独立登录。
3. 主应用统一处理 SSO 跳转、回调、登录态和退出。
4. 子应用只消费登录态和触发登录请求。
5. 所有登录事件集中采集，便于 BI、风控和审计。

建议技术实现优先采用：

```txt
主应用 Auth SDK + SSO Authorization Code Flow with PKCE + Auth BFF + HttpOnly Cookie + 子应用 Auth Bridge
```

该组合能兼顾安全性、扩展性、用户体验和后续数据建设需求。
