# MFE 微前端项目技术执行方案

## 1. 文档信息

| 项目 | 内容 |
| --- | --- |
| 文档名称 | MFE 微前端项目技术执行方案 |
| 适用范围 | 使用 `tsu-cli` 工具模板生成的 MFE 微前端项目 |
| 依据文档 | [prd.md](prd.md) |
| 当前登录形态 | 邮箱登录、账号登录、微信登录 |
| 当前技术前提 | 主应用统一登录、qiankun、无现成账号体系、自建 BI、广告接入腾讯/百度 |
| 文档版本 | v0.1 |
| 日期 | 2026-06-22 |
| 目标读者 | 前端、后端、架构、测试、数据、安全、产品 |

---

## 2. 目标与原则

### 2.1 执行目标

将 PRD 中的产品目标拆解为可落地的技术工作，形成一条从主应用壳、登录体系、子应用接入、埋点采集、BI 入仓到广告适配的完整执行链路。

### 2.2 执行原则

1. **登录统一在主应用**：任何认证动作都由主应用承接，子应用只发起登录请求。
2. **子应用低耦合**：子应用只消费登录态、权限和公共 SDK，不重复实现认证能力。
3. **先基础、后扩展**：优先落地主应用壳、账号体系、埋点、BI 基础链路，再做广告和增强能力。
4. **前后端双校验**：前端只控制展示，后端负责真实鉴权。
5. **数据先行**：行为数据、登录数据、广告数据从第一天就走统一事件规范。
6. **可演进**：当前方案要能平滑扩展 SSO、多端账号、更多广告平台和更复杂 BI 模型。

---

## 3. 总体技术架构

### 3.1 模块划分

| 模块 | 职责 |
| --- | --- |
| 主应用 Shell | 微前端容器、全局导航、登录入口、布局、公共状态、权限拦截 |
| Auth SDK | 登录态读取、登录请求封装、退出、刷新、权限判断、状态广播 |
| 子应用 | 业务页面实现、按规范声明登录策略、消费登录态、触发登录 |
| Tracking SDK | 事件采集、批量上报、失败重试、公共字段注入 |
| Ads Adapter | 腾讯/百度广告适配、广告位管理、曝光/点击采集、开关控制 |
| Config Center | 登录策略、广告开关、页面配置、灰度控制 |
| API Gateway/BFF | 接口聚合、会话管理、用户信息、权限校验、统一鉴权 |
| Auth Service | 账号体系、邮箱/账号/微信登录、token、session、绑定关系 |
| Event Collector | 行为数据接收、清洗、入库、转发 |
| Data Warehouse/BI | 指标建模、主题分析、看板、分群、归因 |

### 3.2 推荐架构图

```txt
Browser
  ├─ 主应用 Shell
  │    ├─ Auth SDK
  │    ├─ Tracking SDK
  │    ├─ Ads Adapter
  │    └─ qiankun 子应用容器
  ├─ 子应用 A / B / C
  └─ 登录页 / 403 / 404 / 500

Backend
  ├─ API Gateway / BFF
  ├─ Auth Service
  ├─ Event Collector
  └─ Data Warehouse / BI
```

---

## 4. 技术栈建议

### 4.1 前端

- 框架：React + TypeScript。
- 包管理器：pnpm。
- monorepo：采用 pnpm workspace + Turborepo。
- 微前端：`qiankun`。
- 状态管理：轻量全局状态 + 事件广播。
- 路由：主应用统一路由控制，子应用独立路由子树。
- 样式：CSS Modules / SCSS / UnoCSS / Tailwind 任选其一，统一规范即可。
- 公共包：`auth-sdk`、`tracking-sdk`、`config-sdk`、`shared-ui`、`shared-types`；`ads-sdk` 本阶段只做后续预留，不强制创建完整包。
- 前端测试：Vitest。
- E2E 测试：Playwright。

### 4.2 后端

- 语言：Python 3.12。
- Web 框架：FastAPI。
- 依赖管理：uv。
- 数据校验：Pydantic v2。
- ORM：默认 SQLAlchemy 2.x。
- 测试：pytest + httpx。
- 第一阶段 A 服务形态：`backend-api` 单 FastAPI 服务，内部按 Auth / Users / Permissions / Events / Common 模块化。
- 后续演进形态：再按需要拆分 Auth Service、BFF/Gateway、Event Collector、BI Query Service、Ads Service。
- BI 存储：先保留链路和接口边界，再做原始事件层、主题层和看板层。

### 4.3 基础设施

- 域名与 HTTPS 证书。
- Cookie 域和 SameSite 策略。
- 日志采集与审计。
- 监控与告警。
- 对象存储或数据库用于事件与原始数据。
- CI/CD：使用 GitHub Actions。
- 环境：区分 test 和 product 两套环境。
- 部署方式：前端、后端、Event Collector、BI 查询服务均采用 Docker 容器部署。

---

## 5. 分阶段执行方案

### 阶段 1：主应用壳与账号体系

阶段 1 拆分为“阶段 1A：工程骨架与接口契约”和“阶段 1B：真实登录 MVP”。截至 2026-06-22，已确认先执行阶段 1A。

#### 阶段 1A：工程骨架与接口契约

##### 目标

建立可启动、可检查、可扩展的基础工程骨架，先把主应用壳、FastAPI 后端骨架、Alembic 骨架、Docker Compose test 和 GitHub Actions 基础检查搭起来。

##### 工作项

1. 搭建 React + TypeScript 主应用 Shell。
2. 配置 qiankun 子应用容器占位。
3. 落地主应用左侧菜单、顶部导航和内容区骨架。
4. 实现统一登录入口和登录页壳。
5. 预留邮箱登录、账号登录、微信登录三种方式的页面占位。
6. 建立 `services/backend-api` FastAPI 骨架。
7. 建立 health / ready、Auth / Users / Permissions / Events 模块占位。
8. 建立 Auth Bridge、Tracking SDK 和公共类型契约。
9. 建立 PostgreSQL / Redis 配置入口和 Alembic 骨架。
10. 建立 `docker-compose.test.yml`，包含 `main-web`、`backend-api`、`postgres`、`redis`。
11. 建立 GitHub Actions CI、Docker build 和 test 部署占位。

##### 交付结果

- 主应用 Shell 可以启动并展示基础布局。
- 登录页壳、用户中心入口或页面壳、403/404/500 兜底页可用。
- 子应用可以通过契约读取或触发主应用登录能力，但没有真实登录闭环。
- FastAPI 后端骨架、health / ready 和接口契约可用。
- Alembic 骨架可用，但不创建真实业务表。
- Docker Compose test 和 GitHub Actions 基础检查可用。

##### 明确不做

- 不实现真实邮箱、账号、微信登录闭环。
- 不创建真实用户、identity、RBAC、事件等业务表。
- 不创建 `ods_events_raw` 表。
- 不 seed 默认管理员。
- 不实现完整 BI 看板、广告接入或 product 自动部署。

#### 阶段 1B：真实登录 MVP

##### 目标

在阶段 1A 骨架基础上，实现用户可以在主应用完成真实登录，并让子应用读取登录态。

##### 工作项

1. 实现邮箱登录、账号登录、微信登录。
2. 实现登录态读取、登录后回跳、退出登录。
3. 实现用户、identity、RBAC 基础表和 seed。
4. 支持主应用统一广播登录态给子应用。

##### 交付结果

- 用户可以在主应用完成登录。
- 子应用可以读取登录态。
- 登录页、用户中心、403/404/500 兜底页可用。

---

### 阶段 2：子应用接入与权限控制

#### 目标

把子应用接入到统一登录和统一权限链路中，保证强登录页面和权限页面表现一致。

#### 工作项

1. 制定子应用接入规范。
2. 在子应用中注入 `authBridge`。
3. 实现路由级强登录拦截。
4. 实现操作级登录请求。
5. 实现权限判断与 403 页面。
6. 建立子应用元信息 `appMeta` 和路由 `meta` 规范。
7. 统一主应用和子应用的页面状态样式。

#### 交付结果

- 子应用无需独立处理登录。
- 路由级强登录和权限控制生效。
- 用户在多个子应用间切换不重复登录。

---

### 阶段 3：统一埋点与 BI 基础链路

#### 目标

把用户行为、登录行为和页面行为统一采集，形成自建 BI 的基础数据链路。

#### 工作项

1. 设计埋点事件规范。
2. 落地 `Tracking SDK`。
3. 统一上报 `page_view`、`login_start`、`login_success`、`login_fail`、`logout`、`permission_denied`、`ad_request`、`ad_impression`、`ad_click` 等事件。
4. 建立匿名访客 `visitorId`、会话 `sessionId`、登录用户 `userId` 关联。
5. 建立原始事件表。
6. 建立基础指标看板。
7. 建立登录转化、活跃、留存、子应用访问排行的基础统计。

#### 交付结果

- 事件可采、可查、可追溯。
- 能够分析登录转化、页面流量、子应用表现和广告基础数据。

---

### 阶段 4：广告适配与收益采集

#### 目标

在不影响主流程的前提下接入腾讯/百度广告，并把广告事件纳入统一数据链路。

#### 工作项

1. 落地 Ads Adapter。
2. 接入腾讯广告和百度广告配置。
3. 建立广告位管理模型。
4. 统一处理广告请求、填充、曝光、点击事件。
5. 在低风险页面做试点。
6. 建立广告收益和转化归因基础数据。

#### 交付结果

- 广告接入有统一层，不散落在各子应用。
- 广告数据和用户行为数据可以统一分析。

---

## 6. 登录技术执行方法

### 6.1 主应用统一登录

#### 6.1.1 接入方式

登录入口只保留在主应用：

- 顶部全局登录入口。
- 登录页。
- 登录弹窗。
- 强登录拦截页。

子应用需要登录时，只能通过：

```ts
auth.requireLogin({ redirectUri, reason, sourceAppId })
```

#### 6.1.2 登录形态

当前只落地三种：

- 邮箱登录。
- 账号登录。
- 微信登录。

#### 6.1.3 建议接口

| 接口 | 方法 | 说明 |
| --- | --- | --- |
| `/api/auth/login/email` | POST | 邮箱登录 |
| `/api/auth/login/account` | POST | 账号登录 |
| `/api/auth/login/wechat/url` | GET | 获取微信授权地址或二维码 |
| `/api/auth/callback` | POST | 登录回调处理 |
| `/api/auth/me` | GET | 获取当前用户 |
| `/api/auth/permissions` | GET | 获取权限 |
| `/api/auth/refresh` | POST | 刷新登录态 |
| `/api/auth/logout` | POST | 退出登录 |

#### 6.1.4 登录态建议

- token 建议走 BFF + HttpOnly Cookie。
- 前端只保存轻量状态，不直接保存 refresh token。
- 登录成功后广播 `auth:state_change`。
- 多标签页使用 `BroadcastChannel` 同步状态。

#### 6.1.5 账户绑定关系

建议使用内部 `userId` 作为主键，邮箱、账号、微信作为身份绑定项：

- `identityType = email`
- `identityType = username`
- `identityType = wechat`

---

### 6.2 子应用接入

#### 6.2.1 标准接入对象

每个子应用至少需要：

- `appId`
- `appName`
- `auth.required`
- `auth.fallback`
- `tracking.enabled`
- `ads.enabled`

#### 6.2.2 接入步骤

1. 子应用通过 qiankun 挂载。
2. 主应用注入 `authBridge` 和 `tracking`。
3. 子应用初始化时读取登录态。
4. 访问强登录页面时调用 `requireLogin()`。
5. 访问无权限页面时展示 403。
6. 业务操作产生事件时调用 `track()`。

#### 6.2.3 禁止事项

- 子应用不得直接实现登录页。
- 子应用不得自行刷新 token。
- 子应用不得直接访问 SSO callback。
- 子应用不得直接保存 refresh token。

---

### 6.3 权限执行方法

#### 6.3.1 权限模型

建议采用：

```txt
RBAC + 资源权限扩展
```

#### 6.3.2 执行方式

- 前端：控制菜单、按钮、页面入口。
- 后端：控制接口访问、资源范围、数据范围。
- 统一用权限编码，如：

```txt
user.profile.read
report.export
adslot.manage
```

#### 6.3.3 资源范围建议

- 全局。
- 组织。
- 部门。
- 个人。
- 指定资源列表。

---

### 6.4 埋点执行方法

#### 6.4.1 SDK 能力

`Tracking SDK` 需要提供：

- `init()`
- `identify()`
- `alias()`
- `track()`
- `page()`
- `flush()`

#### 6.4.2 公共事件字段

所有事件建议包含：

- `eventId`
- `eventName`
- `eventTime`
- `appId`
- `pageName`
- `routePath`
- `userId`
- `visitorId`
- `sessionId`
- `referrer`
- `utmSource`
- `utmMedium`
- `utmCampaign`

#### 6.4.3 事件优先级

P0 必须先落地：

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

### 6.5 BI 执行方法

#### 6.5.1 数据流

```txt
前端事件 -> Event Collector -> 原始事件表 -> 清洗表 -> 主题表 -> 指标看板
```

#### 6.5.2 推荐分层

| 层级 | 说明 |
| --- | --- |
| ODS | 原始事件 |
| DWD | 清洗明细 |
| DWS | 主题聚合 |
| ADS | 看板指标 |

#### 6.5.3 先做哪些指标

- PV / UV。
- 登录成功率。
- 登录失败原因。
- 子应用访问排行。
- 用户活跃数。
- 广告曝光 / 点击。

---

### 6.6 广告执行方法

#### 6.6.1 广告接入方式

- 主应用统一加载广告配置。
- 子应用只声明广告位。
- 广告 SDK 加载失败不影响主流程。
- 广告位必须支持兜底状态。

#### 6.6.2 广告数据采集

- 请求。
- 填充。
- 曝光。
- 点击。
- 关闭。
- 收益回传。

---

## 7. 目录与工程建议

### 7.1 第一阶段 A monorepo 目录

第一阶段 A 已确认采用 pnpm workspace + Turborepo，建议目录为：

```txt
apps/
  main-shell/

packages/
  auth-sdk/
  tracking-sdk/
  shared-ui/
  shared-types/
  config-sdk/

services/
  backend-api/
```

`ads-sdk` 可在广告适配阶段再创建，第一阶段 A 不强制创建完整广告包。

### 7.2 主应用目录

主应用目录为 `apps/main-shell`，建议保留：

```txt
apps/main-shell/
  src/
    shell/
    auth/
    layout/
    routes/
    config/
    shared/
  tests/
```

### 7.3 子应用目录

后续每个子应用建议保留：

```txt
src/
  pages/
  components/
  routes/
  meta.ts
  app.tsx
```

### 7.4 后端目录

第一阶段 A 后端目录为 `services/backend-api`，内部按模块化预留：

```txt
services/backend-api/
  app/
    auth/
    users/
    permissions/
    events/
    common/
    db/
  alembic/
  tests/
```

---

## 8. 里程碑与排期建议

### 第 1 周到第 2 周

- 完成主应用壳。
- 完成 qiankun 子应用接入骨架。
- 完成统一登录页壳。
- 完成最小 Auth Service。

### 第 3 周到第 4 周

- 完成邮箱登录、账号登录、微信登录。
- 完成登录态共享和退出。
- 完成强登录拦截和 403 页。
- 完成用户中心。

### 第 5 周到第 6 周

- 完成 Tracking SDK。
- 完成基础埋点上报。
- 完成原始事件入库。
- 完成基础看板。

### 第 7 周到第 8 周

- 完成 Ads Adapter。
- 接入腾讯/百度广告。
- 建立广告事件归因。
- 在低风险页面试点。

---

## 9. 风险与应对

| 风险 | 影响 | 应对 |
| --- | --- | --- |
| 主应用登录故障 | 全站不可登录 | 增加兜底页、监控和快速回滚 |
| 子应用重复实现登录 | 维护成本高 | 强制通过 Auth SDK 接入 |
| 埋点口径不一致 | BI 不可用 | 建立事件字典和 SDK 约束 |
| 广告影响体验 | 转化下降 | 广告隔离、懒加载、低风险页面试点 |
| 权限判断不一致 | 越权风险 | 前后端双校验 |
| 数据量增长过快 | 成本上升 | 原始事件保留策略、分层存储、采样 |

---

## 10. 验收标准

### 10.1 阶段 1A 验收标准

1. 主应用 Shell 可以启动，顶部导航、左侧菜单、内容区可见。
2. 登录入口和登录页壳只存在于主应用。
3. 邮箱、账号、微信三种登录方式有 UI 占位，但不要求真实登录可用。
4. 403 / 404 / 500 页面可兜底。
5. Auth Bridge 契约存在，且不暴露 refresh token。
6. Tracking SDK 契约存在，采集失败不影响主流程。
7. FastAPI `backend-api` 骨架可启动。
8. health / ready 接口可用于 Docker 和 CI 检查。
9. Alembic 骨架存在，但不要求创建真实业务表。
10. `docker-compose.test.yml` 可拉起 `main-web`、`backend-api`、`postgres`、`redis`。
11. GitHub Actions 能执行基础 CI、Docker build，并预留 test 部署。
12. product 发布不因 push 自动触发。

### 10.2 完整 MVP 验收标准

1. 用户只能在主应用完成登录。
2. 子应用之间登录态共享且不重复登录。
3. 路由级、操作级强登录有效。
4. 邮箱、账号、微信三种登录方式可用。
5. 登录前后行为可通过 visitorId / userId 关联。
6. 埋点事件可稳定进入采集链路。
7. BI 基础看板可用。
8. 广告位可配置、可采集、可回传。
9. 403 / 404 / 500 页面可兜底。
10. 主应用与子应用解耦，不互相污染登录逻辑。

---

## 11. 推荐交付顺序

建议按以下顺序推进：

1. 主应用壳与 qiankun 接入。
2. 统一登录与 Auth SDK。
3. 子应用接入规范与权限控制。
4. 埋点 SDK 与 Event Collector。
5. BI 基础看板。
6. 广告适配层与广告位试点。

---

## 12. 结论

这套技术执行方案的核心是：

- 主应用统一承接登录和全局控制。
- 子应用只做业务，不做独立认证。
- 埋点、BI、广告全部统一进平台化基础能力。
- 通过分阶段交付，先把登录与数据链路跑通，再扩展商业化能力。

如果按这个顺序执行，可以在不牺牲一致性和扩展性的情况下，尽快把 MFE 平台跑起来。
