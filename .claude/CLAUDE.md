# CLAUDE.md

## 项目概览

这是一个基于 `tsu-cli` 模板规划的 MFE 微前端项目，微前端框架采用 `qiankun`。当前仓库主要是产品、技术、资源与部署方案文档，尚未包含完整前后端工程代码。

核心目标：

- 主应用统一承接登录、导航、权限拦截和全局状态。
- 子应用只消费主应用登录态，不独立实现登录。
- 登录方式暂定为邮箱登录、账号登录、微信登录。
- 后端语言使用 Python。
- 主数据库使用 PostgreSQL。
- 数据库迁移工具使用 Alembic。
- BI 采用自建方案。
- 广告后续接入腾讯和百度，不做个性化广告。
- CI/CD 使用 GitHub Actions。
- 部署区分 test 和 product 环境。
- 前端、后端、Event Collector、BI 查询服务采用 Docker 容器部署。

## 关键文档

优先参考以下文档：

- [prd.md](../docs/prd.md)：产品需求与总体方案。
- [technical-execution-plan.md](../docs/technical-execution-plan.md)：总体技术执行方案。
- [frontend-implementation.md](../docs/frontend-implementation.md)：前端实施方案。
- [backend-implementation.md](../docs/backend-implementation.md)：后端实施方案。
- [data-bi-implementation.md](../docs/data-bi-implementation.md)：自建 BI 和数据实施方案。
- [github-cicd-docker-deployment.md](../docs/github-cicd-docker-deployment.md)：GitHub CI/CD 与 Docker 部署方案。
- [resource-preparation-list.md](../docs/resource-preparation-list.md)：资源申请与配置清单。
- [login-method-preparation.md](../docs/login-method-preparation.md)：邮箱、账号、微信登录准备清单。
- [rbac-resource-permission.md](../docs/rbac-resource-permission.md)：RBAC + 资源权限扩展说明。
- [mfeui-design-reference.md](../docs/mfeui-design-reference.md)：MFE UI 设计参考。
- [multi-agent-auto-development.md](../docs/multi-agent-auto-development.md)：多 Agent 自动开发方案。

## 架构约束

### 主应用

主应用负责：

- qiankun 子应用容器。
- 顶部导航和左侧子应用菜单。
- 登录入口、登录页、登录弹窗。
- 登录态初始化、刷新、退出。
- 权限拦截、403/404/500 兜底页。
- Auth SDK、Tracking SDK、Ads Adapter 初始化。

### 子应用

子应用负责：

- 业务页面展示。
- 声明 `appMeta` 和路由 `meta`。
- 消费主应用注入的 `authBridge`。
- 调用 `auth.requireLogin()` 触发主应用登录。
- 调用 `tracking.track()` 上报业务事件。

子应用禁止：

- 独立实现登录页。
- 直接调用登录接口。
- 自行刷新 token。
- 保存 refresh token。
- 直接处理微信或 SSO callback。

## 登录与权限

- 登录全部在主应用完成。
- 登录方式：邮箱登录、账号登录、微信登录。
- 内部用户主键使用 `userId`，邮箱、账号、微信作为 identity 绑定。
- 权限模型采用 RBAC + 资源权限扩展。
- 前端权限只控制展示，后端必须做真实鉴权。

## 后端与数据库

- 后端语言：Python。
- 后端框架：FastAPI。
- 主数据库：PostgreSQL。
- 缓存：Redis，用于 session、验证码、限流和缓存。
- 数据库迁移：Alembic。
- 数据库结构初始化与结构变更统一使用 Alembic。
- 默认角色、默认权限、默认管理员、系统基础配置使用 seed 脚本初始化。
- seed 脚本必须幂等，允许重复执行但不能产生重复数据。
- 生产发布前必须审查 migration 和 seed 脚本，禁止直接手工改生产库。

## 数据与 BI

- BI 自建。
- 事件链路：前端事件 -> Event Collector -> ODS -> DWD -> DWS -> ADS。
- 必须保留 `visitorId`、`sessionId`、`userId` 关联能力。
- P0 事件：`page_view`、`login_start`、`login_success`、`login_fail`、`logout`、`permission_denied`、`ad_request`、`ad_impression`、`ad_click`。

## 部署约束

- 代码托管和 CI/CD：GitHub + GitHub Actions。
- 环境：test 和 product 必须隔离。
- 部署方式：Docker 容器部署。
- product 发布建议使用 GitHub Release、tag 或审批流程触发。
- test 环境可由 test 分支 push 自动部署。
- PostgreSQL test/product 独立。
- Redis test/product 独立。

## UI 方向

UI 风格参考 [mfeui-ui-design.html](../mfeui-ui-design.html)：

- 简约企业后台风。
- 白底、浅灰背景、蓝色主色。
- 顶部横向导航。
- 左侧菜单承载子应用入口。
- 登录页、用户中心、状态页保持统一风格。

## 工作约定

- 修改方案文档时，优先保持各文档口径一致。
- 新增重要约束时，同步检查 `docs/technical-execution-plan.md`、对应实施文档和资源清单。
- 若涉及部署、环境、数据库、后端语言、登录方式变更，必须同步更新本文件。
- 当前项目以方案设计为主；如开始生成代码，需先确认技术栈、目录结构和目标框架。
- 多 Agent 自动开发优先使用 `/agent-plan` 进行任务拆分，或使用 `/auto-dev` 进行“先规划、再确认、后执行、再审查”的编排。
- 多 Agent 产出必须经 `tsuz-reviewer` 汇总审查，检查是否符合本文件和关键方案文档约束。
- 当前仓库无完整工程代码时，任何自动开发流程不得擅自创建前后端工程；必须先确认前端框架、目录结构、包管理器、后端目录、测试框架和部署结构。
