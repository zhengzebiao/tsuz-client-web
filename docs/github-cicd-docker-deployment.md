# GitHub CI/CD 与 Docker 部署方案

## 1. 文档信息

| 项目 | 内容 |
| --- | --- |
| 文档名称 | GitHub CI/CD 与 Docker 部署方案 |
| 适用范围 | 使用 `tsu-cli` 工具模板生成的 MFE 微前端项目 |
| 依据文档 | [technical-execution-plan.md](technical-execution-plan.md) / [resource-preparation-list.md](resource-preparation-list.md) |
| 当前前提 | GitHub CI/CD、Docker 容器部署、test/product 双环境、后端 Python、数据库 PostgreSQL |
| 文档版本 | v0.1 |
| 日期 | 2026-06-22 |
| 目标读者 | 前端、后端、运维、测试、安全 |

---

## 2. 部署目标

项目部署采用：

```txt
GitHub 仓库 + GitHub Actions CI/CD + Docker 镜像 + test/product 双环境部署
```

目标是：

1. 代码提交后自动构建和检查。
2. 合并到测试分支后自动部署 test 环境。
3. 发布正式版本后部署 product 环境。
4. 前端、后端、埋点服务、BI 查询服务全部容器化。
5. test 和 product 使用独立配置、独立密钥、独立数据库或 schema。

### 2.1 第一阶段 A 已确认 DevOps 口径

第一阶段 A 只做工程骨架相关的 CI、Docker build 和 test 部署占位。

已确认：

- 包管理器：pnpm。
- monorepo 工具：Turborepo。
- 后端依赖管理：uv。
- Docker Compose test 文件：`docker-compose.test.yml`。
- test 环境 PostgreSQL / Redis 由 Compose 内容器提供。
- Docker 镜像仓库：GHCR。
- GitHub Actions：CI + Docker build + test 部署占位。
- product 发布：本阶段不实现自动部署，只保留 GitHub Release、tag 或 manual approval 原则。

第一阶段 A Compose 最小服务：

```txt
services:
  main-web
  backend-api
  postgres
  redis
```

第一阶段 A 不做：

- 不做 product 自动部署。
- 不接入 product secrets。
- 不执行 product migration / seed。
- 不创建完整 Event Collector、BI Query Service、Ads Service 独立容器。

---

## 3. 环境划分

### 3.1 test 环境

用于开发联调、测试验收、产品预览。

建议域名：

```txt
test.example.com
api-test.example.com
bi-test.example.com
```

特点：

- 可以频繁部署。
- 允许 mock 或测试数据。
- 接入测试邮件服务和微信测试回调。
- 广告可使用测试广告位或关闭。

### 3.2 product 环境

用于正式用户访问。

建议域名：

```txt
app.example.com
api.example.com
bi.example.com
```

特点：

- 只允许正式发布流程部署。
- 使用正式数据库和正式配置。
- 使用正式微信开放平台配置。
- 使用正式广告配置。
- 需要更严格的权限、审计和回滚策略。

---

## 4. 分支与发布策略

### 4.1 推荐分支

| 分支 | 用途 | 部署环境 |
| --- | --- | --- |
| `main` | 正式稳定分支 | product |
| `test` | 测试环境分支 | test |
| `feature/*` | 功能开发 | 不自动部署 |
| `hotfix/*` | 紧急修复 | 按需部署 |

### 4.2 推荐流程

```txt
feature/* -> PR -> test -> 自动部署 test
                   ↓
                验收通过
                   ↓
                 PR -> main -> 部署 product
```

### 4.3 生产发布建议

product 环境建议不要每次 push main 都直接无条件发布，推荐：

- 使用 GitHub Release。
- 使用 tag 触发。
- 或使用 GitHub Actions manual approval。

---

## 5. Docker 化范围

### 5.1 需要容器化的服务

| 服务 | 镜像 | 说明 |
| --- | --- | --- |
| 主应用 Shell | `mfe-main-web` | 主应用、登录页、全局布局 |
| 子应用 | `mfe-user-center` 等 | 各子应用独立构建部署 |
| Auth Service | `mfe-auth-service` | Python 后端登录服务 |
| BFF/Gateway | `mfe-bff-gateway` | 前端统一接口层 |
| Event Collector | `mfe-event-collector` | 埋点接收服务 |
| BI Query Service | `mfe-bi-service` | 指标查询和看板接口 |
| Ads Service | `mfe-ads-service` | 广告配置和回传 |

### 5.2 Docker 镜像命名建议

```txt
ghcr.io/{org}/{service}:{env}-{gitSha}
ghcr.io/{org}/{service}:{version}
```

示例：

```txt
ghcr.io/company/mfe-main-web:test-a1b2c3d
ghcr.io/company/mfe-auth-service:prod-v1.0.0
```

---

## 6. GitHub Actions 流程

### 6.1 CI 流程

每次 PR 或 push 执行：

1. 安装依赖。
2. 代码格式检查。
3. 类型检查。
4. 单元测试。
5. 构建前端。
6. 构建 Docker 镜像。
7. 安全扫描，可选。

### 6.2 test 环境 CD

触发条件：

```txt
push 到 test 分支
```

执行：

1. 构建镜像。
2. 推送镜像到镜像仓库。
3. 拉取 test 环境配置。
4. 部署 Docker 容器。
5. 执行健康检查。
6. 部署成功通知。

### 6.3 product 环境 CD

触发条件建议：

```txt
发布 GitHub Release 或打 tag
```

执行：

1. 构建正式镜像。
2. 推送镜像到镜像仓库。
3. 拉取 product 环境配置。
4. 备份当前版本信息。
5. 滚动更新或蓝绿部署。
6. 健康检查。
7. 发布结果通知。

---

## 7. 环境变量与密钥管理

### 7.1 环境变量分类

| 类型 | 示例 | 存放位置 |
| --- | --- | --- |
| 前端公开变量 | API 地址、环境名 | 构建变量 |
| 后端服务变量 | 数据库地址、Redis 地址 | GitHub Secrets / 服务器配置 |
| 密钥 | JWT secret、微信 AppSecret | GitHub Secrets / 密钥管理服务 |
| 广告配置 | 腾讯/百度广告参数 | 配置中心或后端配置表 |

### 7.2 test/product 隔离

必须区分：

- 数据库连接。
- Redis 连接。
- Cookie 域。
- 微信回调地址。
- 邮件服务配置。
- 广告位配置。
- BI 存储位置。

### 7.3 GitHub Secrets 建议

```txt
TEST_SERVER_HOST
TEST_SERVER_USER
TEST_SERVER_SSH_KEY
TEST_DATABASE_URL
TEST_REDIS_URL
TEST_WECHAT_APP_ID
TEST_WECHAT_APP_SECRET

PROD_SERVER_HOST
PROD_SERVER_USER
PROD_SERVER_SSH_KEY
PROD_DATABASE_URL
PROD_REDIS_URL
PROD_WECHAT_APP_ID
PROD_WECHAT_APP_SECRET

GHCR_TOKEN
```

---

## 8. Docker Compose 建议

### 8.1 test 环境

第一阶段 A 使用 `docker-compose.test.yml` 的最小服务：

```txt
services:
  main-web
  backend-api
  postgres
  redis
```

完整 MVP 或后续阶段可扩展为：

```txt
services:
  main-web
  auth-service
  bff-gateway
  event-collector
  bi-service
  ads-service
  postgres
  redis
```

### 8.2 product 环境

建议使用 `docker-compose.product.yml`：

```txt
services:
  main-web
  auth-service
  bff-gateway
  event-collector
  bi-service
  ads-service
  redis
```

说明：

- product 环境 PostgreSQL 建议使用独立数据库实例，不建议直接跑在同一 compose 内，除非资源受限。
- test 环境可以使用 compose 快速拉起完整链路。

---

## 9. 前端部署说明

### 9.1 主应用

主应用 Docker 镜像负责：

- 打包主应用静态资源。
- 提供 Nginx 静态服务。
- 代理子应用资源或读取子应用注册配置。

### 9.2 子应用

子应用可以：

- 独立镜像部署。
- 或统一打包到静态资源服务器。

推荐：

- MVP 阶段可以统一部署。
- 后续子应用多了，再拆成独立镜像和独立发布流程。

---

## 10. 后端部署说明

### 10.1 Python 服务

后端服务统一使用 Python：

- FastAPI 服务推荐使用 `uvicorn` 或 `gunicorn + uvicorn worker`。

### 10.2 需要容器化的 Python 服务

- Auth Service。
- BFF/Gateway。
- Event Collector。
- BI Query Service。
- Ads Service。

### 10.3 健康检查

第一阶段 A 的 `backend-api` 建议提供：

```txt
GET /api/health/live
GET /api/health/ready
```

后续拆分多个服务时，每个服务都应提供等价 health / ready 接口；如部署平台需要，也可以额外保留 `/health`、`/ready` 别名。

---

## 11. 数据库与迁移

### 11.1 PostgreSQL

test 和 product 使用独立 PostgreSQL 数据库。

建议：

- test：可以使用独立实例或 Docker PostgreSQL。
- product：建议使用云数据库或独立高可用实例。

### 11.2 数据库迁移

数据库迁移工具统一使用 **Alembic**。

部署前需要执行迁移：

```txt
构建镜像 -> 执行 Alembic 迁移 -> 启动服务 -> 健康检查
```

推荐迁移命令：

```bash
alembic upgrade head
```

product 环境迁移需要谨慎：

- 发布前先备份 PostgreSQL。
- migration 文件必须进入代码评审。
- 禁止直接在生产数据库手工改表。
- 避免破坏性变更，例如直接删除字段、删除表、修改字段类型。
- 必要时使用“新增字段 -> 双写/兼容 -> 数据迁移 -> 下线旧字段”的渐进式变更。

### 11.3 Seed 初始化脚本

数据库结构初始化与结构变更统一使用 Alembic；默认角色、默认权限、默认管理员、系统基础配置使用 seed 脚本初始化。

seed 脚本用于向空库或新环境写入基础数据，例如：

- 默认管理员账号。
- 默认角色。
- 默认权限点。
- 默认登录方式配置。
- 默认菜单配置。
- test 环境测试用户。
- test 环境测试广告位。

seed 脚本必须具备幂等性：允许重复执行，但不能产生重复数据。

推荐流程：

```txt
构建镜像 -> 执行 Alembic 迁移 -> 执行 seed 脚本 -> 启动服务 -> 健康检查
```

product 环境执行 seed 需要谨慎：

- 只允许写入稳定的系统基础数据。
- 不要通过 seed 写入临时运营数据。
- 默认管理员账号必须要求首次登录后修改密码。
- seed 脚本必须进入代码评审。

---

## 12. 回滚策略

### 12.1 镜像回滚

保留最近 N 个稳定镜像版本。

回滚方式：

```txt
将服务镜像 tag 切回上一个稳定版本
重新部署容器
执行健康检查
```

### 12.2 数据库回滚

数据库回滚比镜像回滚复杂，建议：

- product 发布前备份。
- 避免不可逆 migration。
- 对字段删除、表删除做延迟处理。

---

## 13. 权限与安全

### 13.1 GitHub 权限

建议：

- main 分支开启保护。
- product 发布需要审批。
- GitHub Secrets 不允许普通成员查看。
- CI/CD token 最小权限。

### 13.2 服务器权限

建议：

- 部署账号独立。
- SSH key 专用。
- 禁止使用 root 直接部署。
- 生产环境只开放必要端口。

---

## 14. 通知与可观测性

部署完成后建议通知：

- GitHub Actions 状态。
- 飞书/钉钉通知。
- 邮件通知，可选。

监控建议：

- 容器存活。
- CPU/内存。
- 接口错误率。
- 登录成功率。
- 埋点接收成功率。
- BI 任务成功率。

---

## 15. 最小执行清单

MVP 阶段至少准备：

1. GitHub 仓库。
2. GitHub Actions 权限。
3. Dockerfile。
4. Docker 镜像仓库。
5. test 环境服务器。
6. product 环境服务器。
7. PostgreSQL test/product。
8. Redis test/product。
9. 环境变量和 GitHub Secrets。
10. 部署脚本。
11. 健康检查接口。
12. 回滚策略。

---

## 16. 验收标准

### 16.1 阶段 1A 验收标准

1. GitHub Actions 能执行基础 CI。
2. GitHub Actions 能执行 Docker build。
3. test 部署 workflow 有占位，但不要求真实远程部署可用。
4. `docker-compose.test.yml` 可拉起 `main-web`、`backend-api`、`postgres`、`redis`。
5. test PostgreSQL / Redis 由 Compose 内容器提供。
6. Docker 镜像仓库按 GHCR 预留。
7. test 和 product 配置命名隔离。
8. product 发布不因 push 自动触发，本阶段不接入 product secrets。

### 16.2 完整部署验收标准

1. push 到 test 分支后能自动部署 test 环境。
2. product 发布通过 release/tag 或审批流程触发。
3. 前端、后端、采集、BI 服务均以 Docker 容器运行。
4. test 和 product 配置完全隔离。
5. 发布失败能明确报错。
6. 发布成功有通知。
7. 支持回滚到上一个稳定版本。
