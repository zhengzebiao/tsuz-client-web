# 多 Agent 自动开发方案

## 1. 文档信息

| 项目 | 内容 |
| --- | --- |
| 文档名称 | 多 Agent 自动开发方案 |
| 适用范围 | 使用 Claude Code / Agent SDK 协助开发 MFE 微前端项目 |
| 依据文档 | [technical-execution-plan.md](technical-execution-plan.md) / [frontend-implementation.md](frontend-implementation.md) / [backend-implementation.md](backend-implementation.md) |
| 当前技术前提 | qiankun、FastAPI、PostgreSQL、Alembic、Docker、GitHub Actions、test/product 双环境 |
| 文档版本 | v0.1 |
| 日期 | 2026-06-22 |
| 目标读者 | 项目负责人、前端、后端、测试、运维、数据、安全 |

---

## 2. 目标

多 Agent 自动开发的目标不是让 Agent 直接无监督改完所有代码，而是通过**角色分工 + 任务拆分 + 自动检查 + 人工审批**提升研发效率。

核心目标：

1. 将复杂项目拆成可并行的小任务。
2. 让不同 Agent 专注不同领域。
3. 减少上下文混乱和重复沟通。
4. 通过代码评审、测试和 CI/CD 控制风险。
5. 形成可复用的自动开发流程。

---

## 3. 基本原则

### 3.1 Agent 不直接替代负责人

Agent 负责任务执行、草拟代码、补测试、写文档；最终合并、上线和关键决策仍由负责人审批。

### 3.2 每个 Agent 只做清晰边界内的事

例如：

- 前端 Agent 不改数据库 schema。
- 后端 Agent 不改 UI 设计。
- 数据 Agent 不改登录业务逻辑。
- DevOps Agent 不改产品需求。

### 3.3 小步提交

每个任务尽量控制在：

- 1 个模块。
- 1 个功能。
- 1 组相关文件。
- 可单独测试和回滚。

### 3.4 先计划，后编码

复杂任务必须先产出计划，再执行代码。

### 3.5 所有自动开发结果必须过检查

至少包括：

- 类型检查。
- 单元测试。
- lint。
- 构建。
- 基础安全检查。
- 人工 code review。

---

## 4. Agent 角色划分

### 4.1 Product Agent

职责：

- 解析 PRD。
- 拆解用户故事。
- 生成验收标准。
- 维护需求与范围边界。

产出：

- user story。
- acceptance criteria。
- 需求变更说明。

---

### 4.2 Architect Agent

职责：

- 设计整体架构。
- 拆分模块边界。
- 输出接口和数据流。
- 识别技术风险。

产出：

- 架构图。
- 模块拆分。
- 技术决策记录。

---

### 4.3 Frontend Agent

职责：

- 主应用 Shell。
- qiankun 子应用接入。
- 登录页。
- 用户中心。
- Auth SDK 前端桥接。
- Tracking SDK 前端调用。
- UI 组件和页面状态。

产出：

- 前端代码。
- 页面组件。
- 单元测试。
- Story 或 Demo。

---

### 4.4 Backend Agent

职责：

- FastAPI 服务。
- Auth Service。
- BFF/Gateway。
- 用户、权限、身份绑定。
- API 设计与实现。
- Alembic migration。
- seed 脚本。

产出：

- FastAPI 接口代码。
- 数据模型。
- Alembic migration。
- seed 脚本。
- API 测试。

---

### 4.5 Data/BI Agent

职责：

- 埋点事件规范。
- Event Collector。
- ODS / DWD / DWS / ADS 数据建模。
- BI 指标计算。
- 看板接口。

产出：

- 事件 schema。
- 数据表设计。
- ETL / 计算任务。
- 指标口径文档。

---

### 4.6 DevOps Agent

职责：

- Dockerfile。
- Docker Compose。
- GitHub Actions。
- test/product 环境配置。
- 部署脚本。
- 回滚策略。

产出：

- CI/CD workflow。
- Docker 配置。
- 环境变量模板。
- 部署说明。

---

### 4.7 QA Agent

职责：

- 测试用例。
- E2E 测试。
- 登录流程测试。
- 权限测试。
- 回归测试。

产出：

- 测试计划。
- 自动化测试。
- 缺陷报告。

---

### 4.8 Security Agent

职责：

- 登录安全检查。
- 权限越权检查。
- token / cookie 安全。
- API 风险检查。
- 依赖漏洞和密钥泄露检查。

产出：

- 安全审查报告。
- 风险修复建议。

---

### 4.9 Reviewer Agent

职责：

- 审查 diff。
- 找 bug。
- 检查是否符合 CLAUDE.md 和 docs 约束。
- 检查是否缺测试、缺迁移、缺文档。

产出：

- code review 报告。
- 风险清单。
- 修改建议。

---

## 5. 推荐工作流

### 5.1 单任务工作流

```txt
需求输入
  ↓
Product Agent 拆需求
  ↓
Architect Agent 设计方案
  ↓
Frontend / Backend / Data / DevOps Agent 并行实现
  ↓
QA Agent 补测试
  ↓
Reviewer Agent 审查
  ↓
CI/CD 检查
  ↓
人工确认合并
```

### 5.2 多任务并行工作流

适用于项目初期多个模块同时开发。

```txt
主任务拆分
  ├─ 前端任务 -> Frontend Agent
  ├─ 后端任务 -> Backend Agent
  ├─ 数据任务 -> Data Agent
  ├─ 部署任务 -> DevOps Agent
  └─ 测试任务 -> QA Agent

每个 Agent 独立分支/工作区执行
  ↓
统一 Reviewer Agent 汇总审查
  ↓
CI/CD 统一检查
  ↓
人工合并
```

---

## 6. 任务拆分建议

### 6.1 前端任务

- 主应用壳。
- 登录页。
- 用户中心。
- 403 / 404 / 500 页面。
- qiankun 子应用注册。
- Auth SDK。
- Tracking SDK。
- Ads Adapter 前端接入。

### 6.2 后端任务

- FastAPI 项目骨架。
- 用户模型。
- 身份绑定模型。
- 邮箱登录。
- 账号登录。
- 微信登录。
- RBAC 权限。
- Alembic migration。
- seed 脚本。

### 6.3 数据任务

- 事件表。
- Event Collector。
- 登录事件处理。
- 行为事件处理。
- 广告事件处理。
- 基础指标计算。

### 6.4 DevOps 任务

- Dockerfile。
- docker-compose.test.yml。
- docker-compose.product.yml。
- GitHub Actions test 部署。
- GitHub Actions product 发布。
- 环境变量模板。

---

## 7. Agent 输入模板

### 7.1 通用任务模板

```txt
任务名称：
目标：
相关文档：
涉及模块：
不可修改范围：
验收标准：
需要补充测试：
需要同步文档：
```

### 7.2 前端 Agent 模板

```txt
请基于 docs/frontend-implementation.md 和 .claude/CLAUDE.md 实现：

任务：
页面/组件：
交互要求：
接口依赖：
状态处理：
验收标准：
```

### 7.3 后端 Agent 模板

```txt
请基于 docs/backend-implementation.md 和 .claude/CLAUDE.md 实现：

任务：
服务模块：
接口：
数据表：
是否需要 Alembic migration：
是否需要 seed 脚本：
验收标准：
```

### 7.4 DevOps Agent 模板

```txt
请基于 docs/github-cicd-docker-deployment.md 实现：

任务：
环境：test/product
服务：
Docker 镜像：
GitHub Actions 触发方式：
Secrets：
验收标准：
```

---

## 8. 分支与工作区策略

### 8.1 推荐方式

每个 Agent 使用独立分支或独立 worktree。

示例：

```txt
feature/frontend-shell
feature/backend-auth
feature/data-events
feature/devops-cicd
```

### 8.2 合并策略

- 每个 Agent 输出一个 PR。
- Reviewer Agent 先审查。
- CI 通过后人工合并。
- 不建议多个 Agent 直接同时改同一文件。

---

## 9. 冲突控制

### 9.1 文件归属

| 文件类型 | 主要 Agent |
| --- | --- |
| 前端页面 | Frontend Agent |
| API 服务 | Backend Agent |
| 数据模型 | Backend Agent / Data Agent |
| migration | Backend Agent |
| seed | Backend Agent |
| CI/CD | DevOps Agent |
| 测试用例 | QA Agent |
| 文档 | 对应模块 Agent + Reviewer Agent |

### 9.2 冲突避免

- Agent 任务必须明确文件范围。
- 同一文件同一时间只交给一个 Agent。
- 公共类型和接口先由 Architect Agent 定义。
- 迁移文件命名必须避免冲突。

---

## 10. 自动检查链路

每个 PR 至少执行：

- lint。
- type check。
- unit test。
- build。
- Docker build。
- migration check。
- seed 幂等性检查。
- API contract check。

---

## 11. 审核清单

Reviewer Agent 和人工 review 需要检查：

1. 是否符合 [docs/prd.md](prd.md)。
2. 是否符合 [.claude/CLAUDE.md](../.claude/CLAUDE.md)。
3. 子应用是否没有独立实现登录。
4. 后端是否使用 FastAPI。
5. 数据库是否使用 PostgreSQL。
6. migration 是否使用 Alembic。
7. seed 脚本是否幂等。
8. 是否区分 test/product。
9. 是否使用 Docker 部署。
10. 是否有测试和验收说明。

---

## 12. 推荐落地顺序

### 第 1 批 Agent

- Architect Agent：确认目录和模块边界。
- DevOps Agent：搭建 Docker 和 GitHub Actions 骨架。
- Backend Agent：搭建 FastAPI + PostgreSQL + Alembic 骨架。
- Frontend Agent：搭建主应用 Shell + qiankun 骨架。

### 第 2 批 Agent

- Backend Agent：实现邮箱/账号/微信登录。
- Frontend Agent：实现登录页和用户中心。
- Data Agent：实现 Tracking SDK 事件规范和 Event Collector。
- QA Agent：补登录和权限测试。

### 第 3 批 Agent

- Data Agent：实现基础 BI。
- DevOps Agent：完善 product 发布和回滚。
- Security Agent：做登录、权限、密钥、依赖安全审查。
- Reviewer Agent：统一 review。

---

## 13. Claude Code 可执行配置

本项目的多 Agent 自动开发第一版采用 **Claude Code 项目级 Agent 配置 + Slash Command 编排**，不依赖 Anthropic Managed Agents API，也不默认创建远端 Agent / Environment / Session。

### 13.1 Agent 配置文件

项目级 Agent 配置放在 `.claude/agents/`：

| Agent | 配置文件 | 主要职责 |
| --- | --- | --- |
| Product Agent | `.claude/agents/tsuz-product-planner.md` | 需求解析、user story、验收标准、范围控制 |
| Architect Agent | `.claude/agents/tsuz-architect.md` | 架构判断、模块边界、任务拆分、文件归属 |
| Frontend Agent | `.claude/agents/tsuz-frontend.md` | 主应用 Shell、qiankun、登录 UI、Auth SDK、Tracking SDK |
| Backend Agent | `.claude/agents/tsuz-backend.md` | FastAPI、Auth、用户、权限、PostgreSQL、Alembic、seed |
| Data/BI Agent | `.claude/agents/tsuz-data-bi.md` | Tracking、Event Collector、数仓分层、BI 指标 |
| DevOps Agent | `.claude/agents/tsuz-devops.md` | Docker、GitHub Actions、test/product 环境、发布回滚 |
| QA Agent | `.claude/agents/tsuz-qa.md` | 测试计划、E2E、登录权限回归、验收验证 |
| Security Agent | `.claude/agents/tsuz-security.md` | 登录安全、token/cookie、越权、secrets、依赖风险 |
| Reviewer Agent | `.claude/agents/tsuz-reviewer.md` | 汇总审查、风险清单、测试和文档同步检查 |

### 13.2 Slash Command 入口

项目级 Slash Command 放在 `.claude/commands/`。

#### `/agent-plan`

用途：只规划，不直接修改代码。

示例：

```txt
/agent-plan 搭建主应用 Shell + qiankun 子应用容器的第一阶段任务
```

默认流程：

1. 调用 `tsuz-product-planner` 分析需求范围和验收标准。
2. 调用 `tsuz-architect` 判断架构影响、文件归属和执行顺序。
3. 根据影响范围调用 Frontend / Backend / Data / DevOps / QA / Security Agent。
4. 调用 `tsuz-reviewer` 汇总审查。
5. 输出任务包、验收标准、测试计划、文档同步计划和待确认问题。

#### `/auto-dev`

用途：自动开发编排，遵循“先规划、再确认、后执行、再审查”。

示例：

```txt
/auto-dev 设计 FastAPI Auth Service 的用户、身份绑定和登录接口骨架
```

默认流程：

1. 先执行 `/agent-plan` 同等的任务拆分。
2. 判断是否涉及代码生成。
3. 若工程目录、技术栈或测试框架未确认，先要求用户确认。
4. 确认后再按任务边界调度相关 Agent 实施。
5. 实施后由 `tsuz-reviewer` 汇总审查，必要时再调用 `tsuz-qa` 和 `tsuz-security`。

### 13.3 代码生成前置确认

当前仓库仍以方案设计为主，尚未包含完整前后端工程代码。因此任何自动开发流程在创建代码前，必须确认：

- 前端框架：React / Vue / 其他。
- 是否使用 monorepo。
- 包管理器：pnpm / npm / yarn。
- 前端目录结构。
- 后端目录结构。
- 测试框架。
- Docker Compose 结构。
- 数据库连接配置方式。

未确认前，Agent 只能输出任务拆分、方案、文件范围和待确认问题，不应擅自生成工程代码。

### 13.4 Workflow 使用说明

`.claude/commands/auto-dev.md` 默认使用 Claude Code 的项目级 Agent/subagent 编排，不默认调用 Workflow 工具。

只有当用户明确要求“使用 workflow / 多 Agent workflow / 工作流模式 / fan out agents / orchestrate this with workflow”时，才进入 Workflow 工具路线。

### 13.5 Reviewer 汇总要求

所有多 Agent 产出必须由 `tsuz-reviewer` 做最终汇总，至少检查：

1. 是否符合 `docs/prd.md`。
2. 是否符合 `.claude/CLAUDE.md`。
3. 子应用是否没有独立实现登录。
4. 后端是否使用 FastAPI。
5. 数据库是否使用 PostgreSQL。
6. migration 是否使用 Alembic。
7. seed 脚本是否幂等。
8. 是否区分 test/product。
9. 是否使用 Docker 部署。
10. 是否有测试和验收说明。

---

## 14. 风险与应对

| 风险 | 说明 | 应对 |
| --- | --- | --- |
| Agent 改错范围 | 改了不该改的模块 | 明确任务文件范围 |
| 多 Agent 冲突 | 同时改同一文件 | 独立分支/工作区 + 文件归属 |
| 代码风格不一致 | 各 Agent 风格不同 | lint + 统一模板 + Reviewer Agent |
| 需求理解不一致 | Agent 对 PRD 理解不同 | 任务输入必须引用 docs 和 CLAUDE.md |
| migration 冲突 | 多个 Agent 同时建 migration | 统一由 Backend Agent 管理 |
| seed 不幂等 | 重复执行产生脏数据 | seed 幂等性检查 |
| 自动部署风险 | Agent 改动直接上 product | product 发布必须审批 |

---

## 15. 结论

本项目适合采用“多 Agent 辅助开发 + 人工审核 + CI/CD 兜底”的方式推进。

推荐最小可行流程是：

```txt
任务拆分 -> 独立 Agent 实现 -> Reviewer Agent 审查 -> CI/CD 检查 -> 人工合并
```

不要让 Agent 直接无审核发布 product 环境。所有重要变更必须经过 review、测试和审批。
