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
```

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
            
          - entity: "customer"
            rule: "rule_dept_only"
```

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
│   │ 3. 查询用户数据规则                                   │   │
│   │ 4. 构建权限条件表达式                                 │   │
│   │ 5. 注入WHERE子句                                     │   │
│   └─────────────────────────────────────────────────────┘   │
│        ↓                                                     │
│   处理后SQL: SELECT * FROM orders                            │
│              WHERE status = 'ACTIVE'                         │
│              AND (created_by = 'user_123')                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

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
        PermissionContext context) {

    // 1. 获取用户所有角色（按 app_id 隔离）
    List<Role> roles = roleService.getUserRoles(appId, userId);

    // 2. 检查无限制策略（必须在过滤实体规则之前）
    // strategy=all 的角色不绑定具体实体规则，过滤后列表中不会出现，需提前检查角色级别策略
    boolean hasAllAccess = roles.stream()
        .anyMatch(r -> r.getDataStrategy(entityCode) == DataStrategy.ALL);
    if (hasAllAccess) {
        return DataPermissionResult.noRestriction();
    }

    // 3. 收集当前实体的数据规则（仅 rule_based / custom 策略）
    List<DataRule> rules = roles.stream()
        .flatMap(r -> r.getDataRules().stream())
        .filter(r -> r.getEntity().equals(entityCode))
        .collect(Collectors.toList());

    // 4. 无规则时拒绝访问（对应 strategy=none 的角色）
    if (rules.isEmpty()) {
        return DataPermissionResult.noAccess();
    }

    // 5. 合并规则表达式（多角色取 OR 并集）
    String combinedExpression = mergeRules(rules, context);

    return DataPermissionResult.withFilter(combinedExpression);
}
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
│  │                   角色数据权限矩阵                      │  │
│  │                                                       │  │
│  │  角色 \ 实体    │  订单  │  客户  │  报表  │           │  │
│  │  ──────────────┼───────┼───────┼───────┤           │  │
│  │  销售经理      │  部门+子 │  部门  │  全部  │           │  │
│  │  销售代表      │  本人   │  本人  │  无    │           │  │
│  │  财务审核      │  待审   │  VIP   │  全部  │           │  │
│  └───────────────────────────────────────────────────────┘  │
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
