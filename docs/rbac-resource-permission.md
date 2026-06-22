# RBAC + 资源权限扩展说明

## 1. 文档信息

| 项目 | 内容 |
| --- | --- |
| 文档名称 | RBAC + 资源权限扩展说明 |
| 适用范围 | 使用 `tsu-cli` 工具模板生成的 MFE 微前端项目 |
| 文档版本 | v0.1 |
| 日期 | 2026-06-21 |
| 目标读者 | 产品、前端、后端、测试、安全、数据 |

---

## 2. 这是什么

### 2.1 一句话解释

`RBAC + 资源权限扩展` 是一种权限控制方案：

- **RBAC** 负责回答“你是什么角色”。
- **资源权限** 负责回答“你能对哪个资源做什么操作”。

它不是只有“是不是管理员”这种粗粒度控制，而是可以进一步控制到：

- 某个用户能不能看某个页面。
- 某个用户能不能导出某份报表。
- 某个用户能不能编辑某个项目。
- 某个用户能不能操作自己团队下的资源，但不能操作别人的资源。

### 2.2 为什么要加“资源权限扩展”

纯 RBAC 只适合“角色很少、边界很清楚”的场景，比如：

- 管理员
- 普通用户
- 运营
- 审核员

但在实际业务里，很多权限不是“某类人能不能做”，而是“某类人对**哪一个对象**能不能做”：

- 只能编辑自己创建的内容。
- 只能查看自己部门的数据。
- 只能管理自己负责的广告位。
- 只能导出自己权限范围内的报表。
- 只能审批自己所在组织线下的单据。

这时候就需要 RBAC 结合资源维度来扩展。

---

## 3. 核心概念

### 3.1 RBAC 是什么

RBAC = Role Based Access Control，基于角色的访问控制。

它的基本关系是：

```txt
用户 User -> 角色 Role -> 权限 Permission
```

例如：

- 用户 A 拥有“运营”角色。
- “运营”角色拥有“查看报表”“编辑活动”“查看用户列表”等权限。

RBAC 的优点是简单、清晰、容易管理。

### 3.2 资源权限是什么

资源权限关注的是权限对象本身。

例如：

- 报表 `report_001`
- 项目 `project_abc`
- 广告位 `adslot_88`
- 团队 `team_12`
- 用户组 `group_3`
- 订单 `order_998`

资源权限通常会描述：

- 资源属于谁。
- 资源属于哪个组织。
- 用户能对资源执行什么动作。
- 用户的权限范围是全局、组织级、部门级、个人级，还是仅某个资源实例。

### 3.3 动作 Action 是什么

动作表示对资源能做什么，例如：

- `read`：查看
- `create`：创建
- `update`：编辑
- `delete`：删除
- `export`：导出
- `approve`：审批
- `publish`：发布
- `manage`：管理

最终的权限判断，常常是：

```txt
某角色的某个权限 + 某个资源 + 某个动作 + 某个范围
```

---

## 4. 为什么不是只用 RBAC

### 4.1 纯 RBAC 的问题

假设只有以下角色：

- 管理员
- 运营
- 普通用户

那么如果一个运营只能看“自己负责的活动”，纯 RBAC 很难表达。

因为“运营”这个角色本身太宽泛，无法区分：

- 运营 A 只能管活动组 1
- 运营 B 只能管活动组 2
- 运营 C 只能看不能改

如果强行用角色拆分，会变成：

- 运营-活动组1
- 运营-活动组2
- 运营-活动组3
- 运营-只读
- 运营-编辑

角色数量会爆炸，维护很困难。

### 4.2 资源维度能解决什么

资源维度把“谁能做什么”拆成两层：

1. 角色定义能力边界。
2. 资源定义作用范围。

例如：

- 角色：运营
- 权限：`report.read`
- 范围：部门内
- 资源：某个报表、某个项目、某个广告位

这样就能做到：

- 同一个角色，不同人权限范围不同。
- 同一个操作，不同资源有不同结果。
- 角色数量不会随着业务复杂度无限增长。

---

## 5. 推荐权限模型

建议采用下面的组合：

```txt
RBAC + Resource Scope + Action + Optional Condition
```

可以理解成四层：

1. **Role**：这个人是什么角色。
2. **Permission**：这个角色拥有什么能力。
3. **Resource Scope**：这个能力能作用于哪些资源范围。
4. **Condition**：在什么条件下允许，比如资源归属、部门、状态。

---

## 6. 数据模型说明

### 6.1 基础实体

#### User

用户。

#### Role

角色。

#### Permission

权限点。

#### Resource

资源实例。

#### ResourceType

资源类型。

#### ResourceScope

权限范围。

#### Policy

授权策略。

---

### 6.2 建议字段设计

#### 6.2.1 用户 User

| 字段 | 说明 |
| --- | --- |
| userId | 用户唯一 ID |
| name | 用户名称 |
| roles | 拥有的角色列表 |
| orgId | 所属组织 |
| departmentId | 所属部门 |
| status | 用户状态 |

#### 6.2.2 角色 Role

| 字段 | 说明 |
| --- | --- |
| roleId | 角色 ID |
| roleCode | 角色编码 |
| roleName | 角色名称 |
| description | 角色说明 |
| enabled | 是否启用 |

#### 6.2.3 权限 Permission

| 字段 | 说明 |
| --- | --- |
| permissionId | 权限 ID |
| permissionCode | 权限编码 |
| permissionName | 权限名称 |
| resourceType | 资源类型 |
| action | 动作 |
| description | 权限说明 |

示例：

```txt
report.read
report.export
adslot.update
user.manage
project.approve
```

#### 6.2.4 资源 Resource

| 字段 | 说明 |
| --- | --- |
| resourceId | 资源 ID |
| resourceType | 资源类型 |
| resourceName | 资源名称 |
| ownerUserId | 所属用户 |
| ownerOrgId | 所属组织 |
| ownerDepartmentId | 所属部门 |
| status | 资源状态 |
| extra | 扩展属性 |

#### 6.2.5 权限策略 Policy

| 字段 | 说明 |
| --- | --- |
| policyId | 策略 ID |
| roleCode | 关联角色 |
| permissionCode | 权限编码 |
| scopeType | 范围类型 |
| scopeValue | 范围值 |
| condition | 条件表达式 |
| effect | allow / deny |

---

## 7. 资源权限常见范围

### 7.1 全局范围

用户可以操作所有该类资源。

例如：

- 平台超级管理员可以查看所有报表。
- 超级运营可以管理所有广告位。

### 7.2 组织范围

用户只能操作自己组织内的资源。

例如：

- 某大区运营只能看本大区数据。
- 某事业部管理员只能管理本事业部成员。

### 7.3 部门范围

用户只能操作本部门资源。

例如：

- 只能审批本部门的申请单。
- 只能导出本部门的用户数据。

### 7.4 个人范围

用户只能操作自己创建或拥有的资源。

例如：

- 只能编辑自己创建的活动草稿。
- 只能查看自己提交的表单记录。

### 7.5 指定资源范围

用户只能操作特定资源列表。

例如：

- 只允许访问指定项目 ID 列表。
- 只允许管理指定广告位。

---

## 8. 典型权限表达方式

### 8.1 基础表达

```txt
permission = resourceType + '.' + action
```

示例：

- `report.read`
- `report.export`
- `project.update`
- `adslot.publish`

### 8.2 带范围表达

```txt
permission = resourceType + '.' + action + '@' + scope
```

示例：

- `report.read@org`
- `report.export@dept`
- `project.update@self`
- `adslot.manage@all`

### 8.3 带条件表达

```txt
permission = resourceType + '.' + action + ' where ' + condition
```

示例：

- `project.update where ownerUserId = currentUserId`
- `report.read where orgId in currentUserOrgIds`
- `adslot.publish where status = draft`

---

## 9. 权限校验流程

### 9.1 前端校验

前端用于：

- 控制菜单显示。
- 控制按钮是否展示。
- 控制页面是否进入。
- 提供更友好的提示。

前端不能作为唯一安全边界。

### 9.2 后端校验

后端用于：

- 真实接口鉴权。
- 防止越权访问。
- 校验资源归属。
- 校验动作合法性。
- 校验范围和条件。

### 9.3 典型校验步骤

```txt
用户请求接口
  ↓
后端读取当前用户身份
  ↓
查询用户角色
  ↓
查询角色对应权限
  ↓
匹配请求的资源类型与动作
  ↓
检查资源范围（全局 / 组织 / 部门 / 个人 / 指定资源）
  ↓
检查附加条件
  ↓
允许或拒绝
```

---

## 10. 示例场景

### 10.1 运营看报表

需求：

- 运营可以看报表，但只能看自己负责业务线的数据。

实现：

- 角色：运营。
- 权限：`report.read`。
- 范围：`businessLineId in currentUserBusinessLines`。
- 后端在查询报表时附加业务线过滤条件。

### 10.2 编辑活动

需求：

- 活动专员可以编辑自己创建的活动。
- 主管可以编辑本部门所有活动。

实现：

- 活动专员：`campaign.update@self`。
- 主管：`campaign.update@dept`。

### 10.3 导出用户数据

需求：

- 有些人可以查看用户数据，但不能导出。
- 少数管理员可以导出。

实现：

- `user.read` 允许查看。
- `user.export` 单独授权。
- 导出接口必须二次校验。
- 导出的字段还要检查脱敏规则。

### 10.4 广告位管理

需求：

- 某广告运营只能管理指定几个广告位。

实现：

- 角色：广告运营。
- 权限：`adslot.update`。
- 范围：指定 `adSlotId` 列表。

---

## 11. 前端如何使用

### 11.1 菜单控制

```ts
if (auth.hasPermission('report.read')) {
  showMenu('报表中心')
}
```

### 11.2 按钮控制

```ts
const canExport = auth.hasPermission('report.export')
```

### 11.3 路由守卫

```ts
if (!auth.hasPermission('project.update')) {
  return '/403'
}
```

### 11.4 资源级判断

```ts
function canEditProject(project: Project, user: UserContext) {
  if (user.roles.includes('admin')) return true
  if (project.ownerUserId === user.userId) return true
  if (project.ownerOrgId === user.orgId && user.permissions.includes('project.update@org')) return true
  return false
}
```

---

## 12. 后端如何使用

### 12.1 接口鉴权

后端通常不要只判断“这个人是不是某角色”，而要判断：

1. 是否拥有该动作权限。
2. 资源是否属于其范围。
3. 资源状态是否允许操作。
4. 是否满足额外安全条件。

### 12.2 示例伪代码

```ts
function authorize(user, action, resource) {
  const permissions = getPermissionsByRoles(user.roles)

  const matched = permissions.some((permission) => {
    return permission.action === action && permission.resourceType === resource.type
  })

  if (!matched) return false

  if (user.isAdmin) return true

  if (permission.scope === 'self' && resource.ownerUserId === user.userId) return true
  if (permission.scope === 'dept' && resource.ownerDepartmentId === user.departmentId) return true
  if (permission.scope === 'org' && resource.ownerOrgId === user.orgId) return true

  return false
}
```

---

## 13. 与 ACL、ABAC 的区别

### 13.1 ACL

ACL = Access Control List，访问控制列表。

特点：

- 直接给某个用户配置某个资源的权限。
- 粒度非常细。
- 配置容易变多。
- 不适合大规模管理。

### 13.2 RBAC

特点：

- 按角色授权。
- 管理简单。
- 适合角色稳定的业务。

### 13.3 ABAC

ABAC = Attribute Based Access Control，基于属性的访问控制。

特点：

- 依据用户、资源、环境属性做判断。
- 表达能力最强。
- 规则复杂时维护成本高。

### 13.4 本方案的定位

我们推荐的不是纯 ACL，也不是纯 ABAC，而是：

```txt
RBAC + 资源范围 + 少量条件规则
```

这样能兼顾：

- 可维护性。
- 可扩展性。
- 可解释性。
- 研发实现成本。

---

## 14. 适合什么项目

这套方案特别适合：

- MFE 微前端项目。
- 多子应用平台。
- 中后台管理系统。
- 需要组织、部门、角色、资源共同管理的系统。
- 既有“菜单权限”，又有“数据权限”的系统。

---

## 15. 设计原则

### 15.1 先粗后细

先做角色权限，再做资源范围，再做复杂条件。

不要一开始就把权限做得过于复杂。

### 15.2 角色不要过多

角色应该是“职责分类”，不是“每个资源一个角色”。

### 15.3 资源范围要标准化

尽量统一成：

- 全局
- 组织
- 部门
- 个人
- 指定资源

### 15.4 前后端双校验

- 前端负责体验。
- 后端负责安全。

### 15.5 权限点要可审计

每个权限点都应该能回答：

- 谁拥有它？
- 为什么拥有？
- 能操作什么资源？
- 在什么条件下允许？

---

## 16. 常见误区

### 16.1 把角色当用户标签

角色是权限边界，不是用户画像。

### 16.2 把权限写死在前端

前端权限只能做展示控制，不能代替后端鉴权。

### 16.3 用太多角色代替资源范围

这会造成角色爆炸，后期很难维护。

### 16.4 忽略资源归属

没有资源归属，数据权限和业务权限都很难落地。

### 16.5 导出权限和查看权限混在一起

导出通常比查看更敏感，必须拆成独立权限点。

---

## 17. 推荐落地方式

### 17.1 MVP

先实现：

- 用户角色。
- 权限点。
- 菜单/按钮控制。
- 资源归属判断。
- 基础范围：全局 / 组织 / 个人。

### 17.2 增强版

再实现：

- 部门范围。
- 指定资源列表。
- 条件表达式。
- 审计日志。
- 权限变更记录。

### 17.3 高级版

最后实现：

- ABAC 条件规则。
- 动态策略引擎。
- 复杂审批流联动。
- 实时权限缓存刷新。

---

## 18. 结论

`RBAC + 资源权限扩展` 的本质是：

- 用 RBAC 解决“谁能做什么”。
- 用资源权限解决“对哪个对象能做”。
- 用范围和条件解决“做到什么程度”。

它比纯 RBAC 更适合中后台、MFE 平台和数据型系统，也比纯 ABAC 更容易落地和维护。

如果你后续要做登录、权限、BI、广告一体化平台，这套模型是比较稳妥的基础。
