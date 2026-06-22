---
description: 多 Agent 自动开发入口：先规划，再按确认后的范围调度 Agent 实施，最后 Reviewer 汇总。
argument-hint: <任务描述>
---

# /auto-dev

你是 tsuz-client-web 项目的自动开发多 Agent 编排器。根据用户输入的任务：

```txt
$ARGUMENTS
```

按“先规划、再确认、后执行、再审查”的方式推进。

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

## 默认执行流程

### Phase 1：多 Agent 规划

先按 `/agent-plan` 的逻辑完成任务拆解：

1. 调用 `tsuz-product-planner` 分析需求范围和验收标准。
2. 调用 `tsuz-architect` 判断架构影响、文件归属和执行顺序。
3. 按影响范围调用领域 Agent：
   - `tsuz-frontend`
   - `tsuz-backend`
   - `tsuz-data-bi`
   - `tsuz-devops`
   - `tsuz-qa`
   - `tsuz-security`
4. 调用 `tsuz-reviewer` 做计划审查。

### Phase 2：执行前确认

如果任务涉及创建或修改代码，并且当前仓库缺少对应工程骨架或技术栈未明确，必须先向用户确认，不能擅自创建代码。

需要确认的问题包括但不限于：

- 前端框架：React / Vue / 其他。
- 是否使用 monorepo。
- 包管理器：pnpm / npm / yarn。
- 前端目录结构。
- 后端目录结构。
- 测试框架。
- Docker Compose 结构。
- 数据库连接配置方式。

如果用户已经明确这些信息，可以继续执行。

### Phase 3：实施

根据确认后的任务包执行：

- 每次只处理一个清晰边界内的小任务。
- 避免多个 Agent 修改同一文件。
- 公共接口、类型、目录边界先由 `tsuz-architect` 确认。
- migration 和 seed 统一交给 `tsuz-backend` 管理。
- DevOps 变更不得绕过 product 发布审批约束。

### Phase 4：审查与验证

实施后必须：

1. 调用 `tsuz-reviewer` 审查结果。
2. 如涉及安全敏感内容，调用 `tsuz-security`。
3. 如涉及功能行为，调用 `tsuz-qa` 输出或补充测试计划。
4. 运行可用的检查命令；如果仓库尚无工程代码或命令不可用，明确说明跳过原因。

## Workflow 工具说明

默认不要调用 Claude Code Workflow 工具。

只有当用户明确说“使用 workflow / 多 Agent workflow / 工作流模式 / fan out agents / orchestrate this with workflow”时，才使用 Workflow 工具。否则使用项目级 Agent 配置和 Claude Code 的 Agent/subagent 编排。

## 输出格式

请根据阶段输出：

1. 任务理解
2. 参与 Agent 与分工
3. 执行计划
4. 是否需要用户确认
5. 实施结果（如已执行）
6. 验证结果
7. Reviewer 结论
8. 风险与后续建议

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
- product 发布必须人工审批或受控触发。
