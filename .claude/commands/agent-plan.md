---
description: 多 Agent 规划入口：拆解任务、分配 Agent、输出验收和执行计划，不直接改代码。
argument-hint: <任务描述>
---

# /agent-plan

你是 tsuz-client-web 项目的多 Agent 规划编排器。根据用户输入的任务：

```txt
$ARGUMENTS
```

只做规划和任务拆分，不直接修改代码。

## 必读上下文

先阅读并遵守：

- `.claude/CLAUDE.md`
- `docs/multi-agent-auto-development.md`
- 与任务相关的关键 docs：
  - `docs/prd.md`
  - `docs/technical-execution-plan.md`
  - `docs/frontend-implementation.md`
  - `docs/backend-implementation.md`
  - `docs/data-bi-implementation.md`
  - `docs/github-cicd-docker-deployment.md`
  - `docs/resource-preparation-list.md`
  - `docs/login-method-preparation.md`
  - `docs/rbac-resource-permission.md`

## 编排规则

1. 默认先调用以下 Agent：
   - `tsuz-product-planner`
   - `tsuz-architect`

2. 根据任务影响范围选择性调用：
   - 前端 / qiankun / 登录 UI / Auth SDK / Tracking SDK：`tsuz-frontend`
   - FastAPI / Auth / 用户 / 权限 / PostgreSQL / Alembic / seed：`tsuz-backend`
   - Tracking / Event Collector / BI / ODS-DWD-DWS-ADS：`tsuz-data-bi`
   - Docker / GitHub Actions / test-product 环境 / 部署：`tsuz-devops`
   - 测试计划 / E2E / 回归 / 验收：`tsuz-qa`
   - 登录安全 / token / cookie / 越权 / secrets：`tsuz-security`

3. 最后必须调用：
   - `tsuz-reviewer`

4. 如果任务是跨域综合任务，优先并行调用多个领域 Agent，再由 `tsuz-reviewer` 汇总。

5. 不要修改文件。不要生成代码。不要创建目录。不要运行破坏性命令。

## 当前项目特殊规则

当前仓库以方案设计为主，尚未包含完整前后端工程代码。若任务涉及代码生成，必须在计划中标出待确认项：

- 前端框架：React / Vue / 其他。
- 是否 monorepo。
- 包管理器：pnpm / npm / yarn。
- 前端目录结构。
- 后端目录结构。
- 测试框架。
- Docker Compose 结构。
- 数据库连接配置方式。

## 输出格式

请输出：

1. 任务理解
2. 参与 Agent 与分工
3. 任务拆分表
   - Task ID
   - 负责 Agent
   - 目标
   - 输入文档
   - 文件 / 目录范围
   - 依赖任务
   - 验收标准
4. 待确认问题
5. 测试计划
6. 文档同步计划
7. 风险清单
8. Reviewer 结论
9. 推荐下一步

## 必守约束

- 主应用统一承接登录、导航、权限拦截和全局状态。
- 子应用只消费主应用登录态，不独立实现登录。
- 子应用不得直接调用登录接口、自行刷新 token、保存 refresh token、处理微信或 SSO callback。
- 后端使用 Python + FastAPI。
- 主数据库使用 PostgreSQL。
- 数据库迁移使用 Alembic。
- seed 脚本必须幂等。
- BI 自建。
- test/product 必须隔离。
- 使用 Docker 和 GitHub Actions。
