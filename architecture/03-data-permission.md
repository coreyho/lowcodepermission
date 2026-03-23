# 数据权限设计

## 1. 数据权限维度

### 1.1 维度类型定义

```yaml
# 数据权限配置 DSL（应用级别，每个应用独立一份）
app_id: "app_crm"                    # 应用唯一标识（租户键）

data_permissions:
  dimensions:
    - code: "dim_org"
      name: "组织维度"
      type: "organization"
      entity_field: "org_id"
      source: "sys_organization"
      
    - code: "dim_dept"
      name: "部门维度"
      type: "department"
      entity_field: "dept_id"
      source: "sys_department"
      
    - code: "dim_owner"
      name: "Owner维度"
      type: "owner"
      entity_field: "created_by"
      
    - code: "dim_region"
      name: "区域维度"
      type: "custom"
      entity_field: "region_code"
      source: "enum_region"
```

### 1.2 维度说明

| 维度 | 类型 | 说明 | 典型场景 |
|------|------|------|----------|
| `organization` | 系统维度 | 多组织架构 | SaaS平台多租户 |
| `department` | 系统维度 | 部门层级 | 企业内部部门隔离 |
| `owner` | 系统维度 | 数据Owner | 仅查看自己的数据 |
| `custom` | 自定义 | 业务维度 | 区域/产品线/客户等级 |

## 2. 数据规则模板

### 2.1 预定义规则

```yaml
rule_templates:
  - code: "rule_self_only"
    name: "仅本人数据"
    expression: "created_by == ${currentUserId}"
    
  - code: "rule_dept_only"
    name: "本部门数据"
    expression: "dept_id == ${currentDeptId}"
    
  - code: "rule_dept_and_sub"
    name: "本部门及子部门"
    expression: "dept_id IN ${currentDeptTree}"
    
  - code: "rule_org_only"
    name: "本组织数据"
    expression: "org_id == ${currentOrgId}"
    
  - code: "rule_custom"
    name: "自定义规则"
    expression: ""  # 运营管理员填写
```

### 2.2 上下文变量

| 变量 | 说明 | 示例值 |
|------|------|--------|
| `${currentUserId}` | 当前用户ID | "user_123" |
| `${currentDeptId}` | 当前部门ID | "dept_456" |
| `${currentDeptTree}` | 部门树（含子部门） | ["dept_1", "dept_2"] |
| `${currentOrgId}` | 当前组织ID | "org_789" |
| `${currentTime}` | 当前时间 | 1700000000000 |

## 3. 数据规则类型

### 3.1 模板规则

继承预定义模板，参数化配置：

```yaml
data_rules:
  - id: "rule_001"
    name: "本部门订单数据"
    entity: "order"
    type: "template"
    template: "rule_dept_and_sub"    # 引用模板
    operations: ["READ"]             # 操作类型：READ（只读）、WRITE（读写）
                                     # 不指定则默认 ["READ", "WRITE"]
```

**操作类型说明：**

| 类型 | 说明 | 对应 SQL 操作 |
|------|------|---------------|
| `READ` | 只读权限 | SELECT |
| `WRITE` | 读写权限 | INSERT, UPDATE, DELETE |

### 3.2 字段规则

基于字段条件的规则：

```yaml
data_rules:
  - id: "rule_002"
    name: "大客户订单"
    entity: "order"
    type: "field_based"
    conditions:
      - field: "customer_type"
        operator: "EQ"
        value: "VIP"
      - field: "amount"
        operator: "GT"
        value: 100000
    logic: "AND"                      # AND | OR
    operations: ["READ"]              # 仅允许查看，不允许修改
```

**支持的操作符：**

| 操作符 | 说明 | 示例 |
|--------|------|------|
| `EQ` | 等于 | `status EQ 'ACTIVE'` |
| `NE` | 不等于 | `status NE 'DELETED'` |
| `GT` | 大于 | `amount GT 1000` |
| `GTE` | 大于等于 | `age GTE 18` |
| `LT` | 小于 | `amount LT 10000` |
| `LTE` | 小于等于 | `age LTE 60` |
| `IN` | 包含 | `status IN ('A', 'B')` |
| `NOT_IN` | 不包含 | `status NOT_IN ('X')` |
| `LIKE` | 模糊匹配 | `name LIKE '%keyword%'` |
| `BETWEEN` | 范围 | `date BETWEEN '2024-01-01' AND '2024-12-31'` |

### 3.3 自定义表达式规则

使用表达式引擎编写复杂规则：

```yaml
data_rules:
  - id: "rule_003"
    name: "华北区销售数据"
    entity: "order"
    type: "custom"
    expression: "region_code IN ('BJ', 'TJ', 'HEB') AND created_by IN (${userIds})"
    parameters:
      userIds: ["user_001", "user_002", "user_003"]
    operations: ["READ", "WRITE"]     # 可读可写
```

## 4. 角色数据权限配置

### 4.1 数据权限策略

```yaml
roles:
  - code: "role_manager"
    name: "部门经理"
    permissions:
      data:
        strategy: "all"               # 策略类型
```

**策略类型：**

| 策略 | 说明 | 适用场景 |
|------|------|----------|
| `all` | 无限制，可访问全部数据 | 系统管理员 |
| `rule_based` | 基于规则的数据范围 | 普通角色 |
| `custom` | 自定义规则 | 特殊业务场景 |
| `none` | 无数据权限 | 待分配角色 |

### 4.2 规则绑定

```yaml
roles:
  - code: "role_sales_rep"
    name: "销售代表"
    permissions:
      data:
        strategy: "rule_based"
        rules:
          - entity: "order"           # 实体编码
            rule: "rule_self_only"    # 规则编码
            operations: ["READ", "WRITE"]  # 操作类型

          - entity: "customer"
            rule: "rule_dept_only"
            operations: ["READ"]       # 仅查看客户，不允许修改

  - code: "role_finance_auditor"
    name: "财务审核"
    permissions:
      data:
        strategy: "rule_based"
        rules:
          - entity: "order"
            rule: "rule_pending_audit"    # 待审核订单
            operations: ["READ", "WRITE"]  # 可读可改（审批通过/拒绝）
```

**按操作类型配置说明：**

| 场景 | 配置方式 | 示例 |
|------|----------|------|
| 仅查看 | `operations: ["READ"]` | 销售代表查看订单，但不能修改 |
| 可查看可修改 | `operations: ["READ", "WRITE"]` 或不指定 | 销售经理管理订单 |
| 仅修改指定范围 | 多个规则组合 | 财务只能修改待审状态的订单 |

## 5. 数据权限生效机制

### 5.1 SQL 拦截器架构

```
┌─────────────────────────────────────────────────────────────┐
│                     数据权限拦截器                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   原始SQL: SELECT * FROM orders WHERE status = 'ACTIVE'      │
│                                                              │
│        ↓                                                     │
│   ┌─────────────────────────────────────────────────────┐   │
│   │ 1. 解析SQL (JSqlParser)                              │   │
│   │ 2. 获取实体映射 (orders → order)                     │   │
│   │ 3. 识别操作类型（SELECT/INSERT/UPDATE/DELETE → R/W）  │   │
│   │ 4. 查询用户数据规则（按操作类型过滤）                 │   │
│   │ 5. 构建权限条件表达式                                 │   │
│   │ 6. 注入WHERE子句                                     │   │
│   └─────────────────────────────────────────────────────┘   │
│        ↓                                                     │
│   处理后SQL: SELECT * FROM orders                            │
│              WHERE status = 'ACTIVE'                         │
│              AND (created_by = 'user_123')                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**SQL 操作类型映射：**

| SQL 类型 | 操作类型 | 说明 |
|----------|----------|------|
| SELECT | READ | 查询操作 |
| INSERT | WRITE | 新增操作 |
| UPDATE | WRITE | 更新操作 |
| DELETE | WRITE | 删除操作 |

### 5.1.1 角色数据策略查询说明

`Role.getDataStrategy(entityCode)` 返回该角色对指定实体的数据策略。实现时需注意：

- 若角色配置了 `strategy: "all"`（全局无限制），则对任意实体均返回 `ALL`
- 若角色配置了 `strategy: "none"`，则对任意实体返回 `NONE`
- 若角色配置了 `strategy: "rule_based"`，则按实体查找绑定规则；若该实体无绑定规则，降级为 `NONE`

### 5.2 规则合并逻辑

当用户有多个角色的数据规则时，采用 **OR** 合并：

```sql
-- 角色1规则: created_by = 'user_1'
-- 角色2规则: dept_id = 'dept_1'

-- 合并后
WHERE (created_by = 'user_1') OR (dept_id = 'dept_1')
```

特殊情况：如果任一角色策略为 `all`，则跳过数据权限过滤。

### 5.3 数据权限执行流程

```java
public DataPermissionResult calculateDataPermission(
        String appId,
        String userId,
        String entityCode,
        OperationType operationType,    // READ / WRITE
        PermissionContext context) {

    // 1. 获取用户所有角色（按 app_id 隔离）
    List<Role> roles = roleService.getUserRoles(appId, userId);

    // 2. 检查无限制策略（必须在过滤实体规则之前）
    boolean hasAllAccess = roles.stream()
        .anyMatch(r -> r.getDataStrategy(entityCode, operationType) == DataStrategy.ALL);
    if (hasAllAccess) {
        return DataPermissionResult.noRestriction();
    }

    // 3. 收集当前实体的数据规则（按操作类型过滤）
    List<DataRule> rules = roles.stream()
        .flatMap(r -> r.getDataRules().stream())
        .filter(r -> r.getEntity().equals(entityCode))
        .filter(r -> r.hasOperation(operationType))  // 匹配操作类型
        .collect(Collectors.toList());

    // 4. 无规则时拒绝访问
    if (rules.isEmpty()) {
        return DataPermissionResult.noAccess();
    }

    // 5. 合并规则表达式（多角色取 OR 并集）
    String combinedExpression = mergeRules(rules, context);

    return DataPermissionResult.withFilter(combinedExpression);
}
```

**操作类型匹配逻辑：**

```java
// DataRule.hasOperation 实现
public boolean hasOperation(OperationType type) {
    // 未指定 operations 则默认拥有 READ + WRITE
    if (this.operations == null || this.operations.isEmpty()) {
        return true;
    }
    return this.operations.contains(type);
}

// WRITE 权限隐含 READ 权限
// 示例：若用户有 WRITE 规则，则 READ 操作也允许
// 实际实现时，配置 WRITE 自动包含 READ，或查询时同时匹配 WRITE 规则
```

## 6. 运行时数据权限管理

### 6.1 管理界面功能

```
┌──────────────────────────────────────────────────────────────┐
│                    数据权限管理                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐  ┌───────────────────────────────────────┐  │
│  │ 数据维度    │  │                                       │  │
│  │ ├── 组织    │  │         数据规则配置                   │  │
│  │ ├── 部门    │  │                                       │  │
│  │ ├── Owner   │  │  • 规则名称  • 规则类型               │  │
│  │ └── 自定义  │  │  • 适用实体  • 条件配置               │  │
│  │             │  │  • 表达式编辑  • 参数配置             │  │
│  └─────────────┘  └───────────────────────────────────────┘  │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                角色数据权限矩阵（按操作类型）            │  │
│  │                                                       │  │
│  │  角色 \ 实体    │      订单      │      客户      │  报表  │
│  │  ──────────────┼────────────────┼────────────────┼───────┤
│  │               │  R  │   W      │  R  │   W      │  R/W  │
│  │  ─────────────┼─────┼──────────┼─────┼──────────┼───────┤
│  │  销售经理     │ 本部门+子 │ 本部门+子  │ 部门  │  本人    │  全部  │
│  │  销售代表     │  本人   │  本人      │ 本人  │  无      │  无    │
│  │  财务审核     │  待审   │  待审      │  VIP  │  无      │  全部  │
│  │  运营分析     │  全部   │  无        │ 全部  │  无      │  全部  │
│  └───────────────────────────────────────────────────────┘  │
│  说明：R=READ(只读)  W=WRITE(读写)                          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 6.2 规则测试功能

```yaml
# 规则测试配置
test_case:
  rule_id: "rule_001"
  test_data:
    - entity: "order"
      record:
        id: "1"
        created_by: "user_123"
        dept_id: "dept_456"
      expected: true                    # 期望是否有权限
      
    - entity: "order"
      record:
        id: "2"
        created_by: "user_999"
        dept_id: "dept_789"
      expected: false
```
