# 后端实施方案

## 1. 文档信息

| 项目 | 内容 |
| --- | --- |
| 文档名称 | 后端实施方案 |
| 适用范围 | 使用 `tsu-cli` 工具模板生成的 MFE 微前端项目 |
| 依据文档 | [prd.md](prd.md) / [technical-execution-plan.md](technical-execution-plan.md) |
| 当前前提 | 主应用统一登录、邮箱登录/账号登录/微信登录、自建 BI、广告接入腾讯/百度、Python、PostgreSQL、GitHub CI/CD、Docker 容器部署、test/product 双环境 |
| 文档版本 | v0.1 |
| 日期 | 2026-06-22 |
| 目标读者 | 后端、架构、安全、测试 |

---

## 2. 后端目标

- 建立最小可用 Auth Service。
- 提供统一登录、退出、刷新、用户信息、权限接口。
- 提供埋点接收和事件流转能力。
- 为广告配置、广告回传和归因提供接口。
- 为前端提供稳定 BFF/Gateway 支撑。
- 后端语言统一使用 Python。
- 主数据库统一使用 PostgreSQL。

---

## 3. 后端模块划分

| 模块 | 职责 |
| --- | --- |
| Auth Service | 账号体系、邮箱/账号/微信登录、token、session、绑定关系 |
| BFF/Gateway | 前端统一接口、用户态、权限、刷新、退出 |
| Event Collector | 埋点接收、校验、清洗、转发 |
| Config Service | 页面配置、登录配置、广告开关、灰度 |
| Ads Service | 广告位配置、回传、收益记录 |

---

## 4. Auth Service 实施

### 4.1 账号体系

建议使用内部 `userId` 作为主键，邮箱、账号、微信作为身份绑定项。

#### 技术栈

- 后端语言：Python 3.12。
- 后端框架：FastAPI。
- 依赖管理：uv。
- 数据校验：Pydantic v2。
- ORM：默认 SQLAlchemy 2.x。
- 后端测试：pytest + httpx。
- 主数据库：PostgreSQL。
- 数据库迁移：Alembic。
- 基础数据初始化：seed 脚本。

#### 第一阶段 A 已确认后端口径

第一阶段 A 只搭建 `services/backend-api` 的 FastAPI 工程骨架和接口契约。

本阶段做：

- FastAPI app 启动骨架。
- 配置加载。
- health / ready 接口。
- Auth / Users / Permissions / Events / Common 模块占位。
- Auth API 契约占位。
- PostgreSQL / Redis 配置入口。
- Alembic 骨架。
- pytest + httpx 基础测试。

本阶段不做：

- 不实现真实邮箱、账号、微信登录闭环。
- 不创建真实用户、identity、RBAC、事件等业务表。
- 不创建 `ods_events_raw` 表。
- 不 seed 默认管理员。
- 不实现完整 Event Collector、BI Query Service、Ads Service。

身份类型：

- `email`
- `username`
- `wechat`

### 4.2 核心接口

| 接口 | 方法 | 说明 |
| --- | --- | --- |
| `/api/auth/login/email` | POST | 邮箱登录 |
| `/api/auth/login/account` | POST | 账号登录 |
| `/api/auth/login/wechat/url` | GET | 微信授权地址或二维码 |
| `/api/auth/callback` | POST | 登录回调 |
| `/api/auth/me` | GET | 当前用户 |
| `/api/auth/permissions` | GET | 当前权限 |
| `/api/auth/refresh` | POST | 刷新登录态 |
| `/api/auth/logout` | POST | 退出登录 |

### 4.3 登录态策略

- refresh token 尽量不暴露给前端。
- access token 建议通过 BFF + HttpOnly Cookie 处理。
- 登录后写入 session，并同步用户信息。
- 支持多标签页状态一致性。

### 4.4 登录流程

#### 邮箱登录

1. 校验邮箱格式。
2. 发送验证码或邮箱绑定校验。
3. 校验验证码。
4. 找到或创建用户。
5. 返回登录成功状态。

#### 账号登录

1. 校验账号和密码。
2. 校验密码哈希。
3. 检查账号是否锁定/禁用。
4. 返回登录成功状态。

#### 微信登录

1. 返回授权地址或二维码。
2. 用户扫码授权。
3. 回调换取 openid/unionid。
4. 找到或创建用户。
5. 建立绑定关系。

### 4.5 Seed 初始化脚本

seed 脚本用于初始化系统基础数据，包括：

- 默认管理员账号。
- 默认角色。
- 默认权限点。
- 默认登录方式配置。
- 默认菜单配置。
- test 环境测试用户。

seed 脚本必须满足：

- 可重复执行。
- 不产生重复数据。
- 不覆盖生产环境已有业务数据。
- product 环境只写入稳定基础数据。

第一阶段 A 已确认先跳过默认管理员 seed；本阶段只预留 seed 后续入口和幂等原则，不写入真实默认管理员、测试用户或业务基础数据。

---

## 5. 权限服务实施

### 5.1 权限模型

建议采用：

```txt
RBAC + 资源权限扩展
```

### 5.2 权限编码示例

```txt
user.profile.read
report.export
adslot.manage
billing.order.create
```

### 5.3 权限校验原则

- 前端权限只做展示。
- 后端接口必须再校验。
- 资源范围需校验组织、部门、个人或指定资源。

---

## 6. 埋点接收实施

### 6.1 Event Collector 接口

| 接口 | 方法 | 说明 |
| --- | --- | --- |
| `/api/events/collect` | POST | 单条或批量事件上报 |
| `/api/events/schema` | GET | 事件字典（可选） |

### 6.2 接收要求

- 支持批量上报。
- 支持异步入库。
- 支持失败重试和幂等。
- 校验公共字段和 schema。
- 原始事件保留可追溯。

### 6.3 核心事件

- `page_view`
- `login_start`
- `login_success`
- `login_fail`
- `logout`
- `permission_denied`
- `ad_request`
- `ad_impression`
- `ad_click`

---

## 7. 广告后端实施

### 7.1 广告配置接口

| 接口 | 方法 | 说明 |
| --- | --- | --- |
| `/api/ads/config` | GET | 获取广告位配置 |
| `/api/ads/report` | POST | 广告事件回传 |

### 7.2 广告位模型

建议字段：

- `adSlotId`
- `appId`
- `pageName`
- `position`
- `provider`
- `size`
- `enabled`
- `refreshInterval`
- `targeting`

### 7.3 广告事件

- 请求。
- 填充。
- 曝光。
- 点击。
- 关闭。
- 收益回传。

---

## 8. 安全要求

- 所有鉴权接口限流。
- 登录失败需要审计。
- 回调地址白名单校验。
- 防 CSRF / XSS / 重放攻击。
- 敏感字段脱敏输出。
- 微信 AppSecret 只存服务端。

---

## 9. 目录建议

第一阶段 A 后端目录为 `services/backend-api`：

```txt
services/backend-api/
  app/
    auth/
    permissions/
    events/
    users/
    common/
    db/
  alembic/
  tests/
  pyproject.toml
  Dockerfile
```

第一阶段 A 不创建独立 Auth Service、BFF/Gateway、Event Collector、BI Query Service、Ads Service 目录；这些能力先在 `backend-api` 内部模块化预留。

---

## 10. 验收标准

### 10.1 阶段 1A 验收标准

1. `services/backend-api` FastAPI 应用可启动。
2. Python 3.12 + uv + FastAPI + Pydantic v2 基础工程可用。
3. pytest + httpx 基础测试可运行。
4. health / ready 接口可用于 Docker 和 CI 检查。
5. Auth / Users / Permissions / Events / Common 模块占位清晰。
6. Auth API 契约存在，但不要求真实登录可用。
7. PostgreSQL / Redis 配置入口存在。
8. Alembic 骨架存在，但不创建真实业务表。
9. 不创建 `ods_events_raw` 表。
10. 不 seed 默认管理员。
11. 后端默认保留真实鉴权职责，前端权限不得作为唯一安全边界。

### 10.2 完整 MVP 验收标准

1. 邮箱、账号、微信登录可用。
2. `/me`、`/permissions`、`/refresh`、`/logout` 可用。
3. 权限接口能正确拒绝越权请求。
4. 埋点接口可批量接收事件。
5. 广告配置和回传接口可用。
6. 安全策略和日志审计可落地。
