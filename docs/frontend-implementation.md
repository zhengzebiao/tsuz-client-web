# 前端实施方案

## 1. 文档信息

| 项目 | 内容 |
| --- | --- |
| 文档名称 | 前端实施方案 |
| 适用范围 | 使用 `tsu-cli` 工具模板生成的 MFE 微前端项目 |
| 依据文档 | [prd.md](prd.md) / [technical-execution-plan.md](technical-execution-plan.md) |
| 当前前提 | 主应用统一登录、qiankun、邮箱登录/账号登录/微信登录、自建 BI、GitHub CI/CD、Docker 容器部署、test/product 双环境 |
| 文档版本 | v0.1 |
| 日期 | 2026-06-22 |
| 目标读者 | 前端、UI、架构、测试 |

---

## 2. 前端目标

- 搭建主应用壳和 qiankun 子应用容器。
- 实现主应用统一登录入口与登录态管理。
- 实现子应用接入规范、权限展示和登录请求桥接。
- 接入埋点 SDK、广告适配层和基础异常兜底页面。
- 形成可复用的公共包和统一 UI 体系。

### 2.1 第一阶段 A 已确认前端口径

第一阶段 A 只做“工程骨架与接口契约”，前端不实现真实登录闭环。

已确认：

- 前端框架：React + TypeScript。
- 包管理器：pnpm。
- monorepo：pnpm workspace + Turborepo。
- 主应用目录：`apps/main-shell`。
- 前端测试框架：Vitest。
- E2E 测试框架：Playwright。

第一阶段 A 前端只交付：

- 主应用 Shell。
- qiankun 子应用容器占位。
- 顶部导航、左侧菜单、内容区。
- 登录入口和登录页壳。
- 邮箱登录、账号登录、微信登录三种方式的 UI 占位。
- 403 / 404 / 500 页面。
- Auth Bridge 契约占位。
- Tracking SDK 契约占位。

第一阶段 A 不交付：

- 不实现真实邮箱、账号、微信登录提交。
- 不实现真实 token 刷新。
- 不保存 refresh token。
- 不实现真实用户中心业务功能，只保留入口或页面壳。
- 不接入真实腾讯/百度广告 SDK。

---

## 3. 前端架构

### 3.1 主应用职责

- 顶部导航。
- 左侧子应用菜单。
- 登录入口。
- 全局布局。
- 权限拦截。
- 统一消息、状态和异常兜底。

### 3.2 子应用职责

- 业务页面渲染。
- 消费登录态。
- 调用 `requireLogin()`。
- 上报行为事件。
- 声明广告位。

### 3.3 公共前端包

```txt
packages/
  auth-sdk/
  tracking-sdk/
  ads-sdk/
  shared-ui/
  shared-types/
  config-sdk/
```

---

## 4. 主应用前端执行

### 4.1 主应用壳

主应用需要先完成：

1. 顶部导航。
2. 左侧子应用菜单。
3. 内容区容器。
4. 登录按钮。
5. 用户菜单。
6. 全局通知区。
7. 异常页兜底。

### 4.2 登录页

登录页只在主应用存在，支持三种登录方式：

- 邮箱登录。
- 账号登录。
- 微信登录。

建议页面结构：

- 左侧品牌说明区。
- 右侧登录卡片区。
- Tab 切换登录方式。
- 登录完成后回跳。

### 4.3 登录态管理

前端建议采用：

- `auth-sdk` 统一封装。
- 内存态保存当前用户。
- `BroadcastChannel` 同步多标签页状态。
- 登录态变化时通知子应用。

### 4.4 路由与权限

- 主应用路由控制子应用入口。
- 子应用路由中使用 `meta.authRequired` 和 `meta.permission`。
- 未登录跳登录。
- 无权限进 403 页面。

---

## 5. 子应用前端执行

### 5.1 接入规范

每个子应用需提供：

```ts
export const appMeta = {
  appId: 'user-center',
  appName: '用户中心',
  auth: { required: true, fallback: 'redirect' },
  tracking: { enabled: true },
  ads: { enabled: false },
}
```

### 5.2 子应用初始化流程

1. 接收主应用 props。
2. 读取当前登录态。
3. 初始化页面状态。
4. 注册 `authChange` 监听。
5. 按需调用 `requireLogin()`。

### 5.3 禁止事项

- 不得独立实现登录页。
- 不得直接调用登录接口。
- 不得直接保存 refresh token。
- 不得直接访问 SSO callback。

---

## 6. 前端 SDK 实施

### 6.1 Auth SDK

能力：

- `init()`
- `login()`
- `logout()`
- `requireLogin()`
- `getUser()`
- `getAuthState()`
- `hasPermission()`
- `onAuthChange()`

### 6.2 Tracking SDK

能力：

- `init()`
- `identify()`
- `alias()`
- `track()`
- `page()`
- `flush()`

### 6.3 Ads SDK

能力：

- 读取广告配置。
- 渲染广告位。
- 曝光/点击采集。
- 加载失败兜底。

---

## 7. 页面实现优先级

### 阶段 1A

- 主应用壳。
- qiankun 容器占位。
- 登录入口和登录页壳。
- 邮箱、账号、微信三种登录方式的 UI 占位。
- 用户中心入口或页面壳。
- 403 / 404 / 500 页面。
- 基础菜单切换。
- Auth Bridge 契约。
- Tracking SDK 契约。

### P0

- 主应用壳。
- 登录页。
- 用户中心。
- 403 / 404 / 500 页面。
- 基础菜单切换。
- 登录态初始化、登录后回跳、退出登录。

### P1

- 子应用接入模板。
- 路由级强登录。
- 操作级登录拦截。
- 基础广告位占位。

### P2

- 登录动画。
- 页面骨架屏。
- 更丰富的布局模板。

---

## 8. 目录建议

第一阶段 A 采用 monorepo，主应用目录为 `apps/main-shell`：

```txt
apps/main-shell/
  src/
    shell/
    auth/
    layout/
    routes/
    pages/
    components/
    shared/
  tests/
```

公共包目录建议：

```txt
packages/
  auth-sdk/
  tracking-sdk/
  shared-ui/
  shared-types/
  config-sdk/
```

`ads-sdk` 可后续按广告接入阶段再创建，第一阶段 A 不强制创建完整广告包。

---

## 9. 验收标准

### 9.1 阶段 1A 验收标准

1. `apps/main-shell` 主应用能启动。
2. 顶部导航、左侧菜单、内容区正常显示。
3. qiankun 子应用容器有占位和加载失败兜底。
4. 登录入口和登录页壳只存在于主应用。
5. 邮箱、账号、微信三种登录方式有 UI 占位，但不要求真实提交可用。
6. 403 / 404 / 500 正常显示。
7. Auth Bridge 契约存在，子应用只能通过 `requireLogin()` 触发主应用登录。
8. Tracking SDK 契约存在，调用失败不影响主流程。
9. 不保存 refresh token，不处理微信 callback。
10. Vitest 基础测试可运行，Playwright E2E 目录或配置可预留。

### 9.2 完整 MVP 验收标准

1. 主应用能统一登录。
2. 子应用能读取登录态。
3. 登录完成后能回跳。
4. 403 / 404 / 500 正常显示。
5. 埋点 SDK 和广告位占位可接入。
6. 主子应用之间不互相耦合登录逻辑。
