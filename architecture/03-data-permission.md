# 数据权限设计

## 1. 数据权限维度

### 1.1 维度类型定义

```yaml
# 数据权限配置 DSL（应用配置期生成，应用运行期分配）
# ┌─────────────────────────────────────────────────────────────────┐
# │  生成阶段：应用配置期（Config Space）                            │
# │  - 使用方：应用开发者、实体设计师                                 │
# │  - 操作：定义数据维度、设计数据规则模板、标记实体字段             │
# │  - 输出：本 DSL 文件，描述该应用有哪些数据规则可用                │
# │                                                                 │
# │  分配阶段：应用运行期（Runtime）                                 │
# │  - 使用方：运营管理员、权限管理员                                 │
# │  - 操作：基于本 DSL 中的数据规则，为角色配置数据权限范围          │
# │  - 存储：角色-数据规则绑定关系存入权限服务数据库                   │
# └─────────────────────────────────────────────────────────────────┘

app_id: "app_crm"                    # 应用唯一标识（租户键）

data_permissions:
  dimensions:
    - code: "dim_org"
      name: "组织维度"
      type: "organization"
      entity_field: "org_id"
      source_type: "table"
      source_config:
        table: "sys_organization"

    - code: "dim_dept"
      name: "部门维度"
      type: "department"
      entity_field: "dept_id"
      source_type: "table"
      source_config:
        table: "sys_department"

    - code: "dim_owner"
      name: "Owner维度"
      type: "owner"
      entity_field: "created_by"

    - code: "dim_region"
      name: "区域维度"
      type: "custom"
      entity_field: "region_code"
      source_type: "enum"
      source_config:
        values:
          - code: "BJ"
            name: "北京"
          - code: "SH"
            name: "上海"
          - code: "GZ"
            name: "广州"
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
    param_schema:
      params: []  # 无参数，直接使用上下文变量

  - code: "rule_dept_only"
    name: "本部门数据"
    expression: "dept_id == ${currentDeptId}"
    param_schema:
      params: []

  - code: "rule_dept_and_sub"
    name: "本部门及子部门"
    expression: "dept_id IN ${currentDeptTree}"
    param_schema:
      params: []

  - code: "rule_org_only"
    name: "本组织数据"
    expression: "org_id == ${currentOrgId}"
    param_schema:
      params: []

  - code: "rule_custom"
    name: "自定义规则"
    expression: ""  # 运营管理员填写
    param_schema:
      params:
        - name: "dimension"
          type: "dimension"
          required: true
          label: "数据维度"
        - name: "operator"
          type: "enum"
          required: true
          options: ["EQ", "IN", "GT", "LT"]
          default: "EQ"
        - name: "value"
          type: "context_variable"
          required: true
          options: ["${currentUserId}", "${currentDeptId}", "${currentOrgId}"]
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

### 4.2 规则绑定（应用运行期配置）

```yaml
# 角色数据权限配置（应用运行期 Runtime）
# - 使用方：运营管理员、权限管理员
# - 数据来源：基于配置期生成的 data_rules 中的 id 进行分配
# - 操作：为每个角色配置各实体的数据权限范围（绑定具体规则）

roles:
  - code: "role_sales_rep"
    name: "销售代表"
    permissions:
      data:
        strategy: "rule_based"
        rules:
          # 从上方 data_rules 中选取规则 id，指定操作类型
          - entity: "order"
            rule: "rule_self_only"         # 引用数据规则 id
            operations: ["READ", "WRITE"]   # 对自己的订单可读可写

          - entity: "customer"
            rule: "rule_dept_only"
            operations: ["READ"]            # 仅查看本部门客户，不允许修改

  - code: "role_sales_manager"
    name: "销售经理"
    permissions:
      data:
        strategy: "rule_based"
        rules:
          - entity: "order"
            rule: "rule_dept_and_sub"      # 管理部门及子部门订单
            operations: ["READ", "WRITE"]

          - entity: "customer"
            rule: "rule_dept_only"
            operations: ["READ", "WRITE"]   # 可维护本部门客户

  - code: "role_finance_auditor"
    name: "财务审核"
    permissions:
      data:
        strategy: "rule_based"
        rules:
          - entity: "order"
            rule: "rule_pending_audit"     # 待审核订单（需在 data_rules 中定义）
            operations: ["READ", "WRITE"]   # 可读可改（审批通过/拒绝）
```

**配置流程说明：**
1. **配置期**：开发者定义 `data_rules` 中的规则（如 rule_self_only、rule_dept_only 等）
2. **运行期**：管理员创建角色时，为每个实体选择适用的数据规则
3. **操作类型**：按 READ/WRITE 区分只读或读写权限

**按操作类型配置示例：**

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

## 7. 列权限（字段级权限）

列权限控制用户对实体字段的访问能力，与行权限（数据范围）共同构成完整的数据权限体系。

### 7.1 权限维度

| 维度 | 说明 | 控制粒度 |
|------|------|----------|
| **可见性 (Visible)** | 能否看到字段值 | 字段级别 |
| **可编辑性 (Editable)** | 能否修改字段值 | 字段级别 |
| **数据脱敏 (Masking)** | 字段值是否脱敏显示 | 字段级别 |

### 7.2 与功能权限的关系

```
┌─────────────────────────────────────────────────────────────┐
│                     权限体系层级                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  功能权限（控制功能入口）                                    │
│  ├── 页面权限：能否进入订单详情页                            │
│  └── 按钮权限：能否点击编辑按钮                              │
│                                                             │
│  数据权限（控制数据内容）                                    │
│  ├── 行权限：能看到哪些订单记录（本部门/本人等）              │
│  └── 列权限：能看到订单的哪些字段（金额/成本价等）            │
│      ├── visible：字段是否可见                              │
│      ├── editable：字段是否可编辑                           │
│      └── masking：字段是否脱敏                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**配置位置：**
- 功能权限（02-functional-permission.md）：配置字段是否对角色可见（通过权限码控制）
- 数据权限（本文档）：配置字段的可编辑性和脱敏规则

### 7.3 列权限配置模型

```yaml
# 列权限配置（应用运行期）
roles:
  - code: "role_sales_rep"
    name: "销售代表"
    column_permissions:          # 列权限（字段级权限）
      - entity: "order"
        fields:
          - field_code: "field_amount"
            visible: true        # 可见（功能权限已授权）
            editable: false      # 不可编辑（只读）
            masking:
              enabled: true      # 脱敏显示
              type: "partial"

          - field_code: "field_cost_price"
            visible: false       # 不可见（功能权限未授权）

  - code: "role_finance"
    name: "财务"
    column_permissions:
      - entity: "order"
        fields:
          - field_code: "field_amount"
            visible: true
            editable: true       # 财务可编辑金额
            masking:
              enabled: false     # 财务看不脱敏

          - field_code: "field_cost_price"
            visible: true        # 财务可见成本价
            editable: false
```

### 7.4 多角色权限合并

```java
/**
 * 列权限合并规则
 */
public FieldPermission mergeColumnPermissions(List<FieldPermission> permissions) {
    // 1. 可见性：功能权限层面已控制，数据权限只处理可见字段
    boolean visible = permissions.stream()
        .anyMatch(p -> p.isVisible());

    // 2. 可编辑性：所有角色都可编辑才可编辑
    boolean editable = visible && permissions.stream()
        .allMatch(p -> p.isEditable());

    // 3. 脱敏：任一角色需要脱敏则脱敏（取最严格）
    MaskingConfig masking = permissions.stream()
        .filter(p -> p.getMasking() != null && p.getMasking().isEnabled())
        .map(FieldPermission::getMasking)
        .max(Comparator.comparing(MaskingConfig::getLevel))
        .orElse(MaskingConfig.none());

    return FieldPermission.builder()
        .visible(visible)
        .editable(editable)
        .masking(masking)
        .build();
}
```

### 7.5 列权限生效机制

列权限在 API 响应阶段生效：

```java
@RestControllerAdvice
public class ColumnPermissionResponseAdvice implements ResponseBodyAdvice<Object> {

    @Autowired
    private ColumnPermissionService columnPermissionService;

    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType,
                                  MediaType selectedContentType,
                                  Class<? extends HttpMessageConverter<?>> selectedConverterType,
                                  ServerHttpRequest request, ServerHttpResponse response) {

        if (!(body instanceof Result)) {
            return body;
        }

        Result<?> result = (Result<?>) body;
        if (result.getData() == null) {
            return body;
        }

        // 获取实体注解
        EntityResponse entityAnnotation = returnType.getMethodAnnotation(EntityResponse.class);
        if (entityAnnotation == null) {
            return body;
        }

        String entityCode = entityAnnotation.value();
        String appId = TenantContext.getCurrentTenant();
        String userId = SecurityContext.getCurrentUserId();

        // 获取列权限
        Map<String, FieldPermission> columnPermissions = columnPermissionService
            .getFieldPermissions(appId, userId, entityCode);

        // 过滤和脱敏处理
        Object filteredData = applyColumnPermissions(result.getData(), columnPermissions);
        result.setData(filteredData);

        return body;
    }
}
```

### 7.6 完整数据权限示意图

```
┌─────────────────────────────────────────────────────────────────┐
│                         订单表 (order)                          │
├──────────┬──────────┬──────────┬──────────┬─────────────────────┤
│   id     │ customer │  amount  │ cost_    │    created_by       │
│          │          │          │ price    │                     │
├──────────┼──────────┼──────────┼──────────┼─────────────────────┤
│  1001    │  张三    │  ¥5000   │  ¥3000   │    user_001         │  ← 行权限控制
│  1002    │  李四    │  ¥8000   │  ¥4500   │    user_002         │     销售代表只能
│  1003    │  王五    │  ¥12000  │  ¥7000   │    user_003         │     看到自己的数据
├──────────┴──────────┴──────────┴──────────┴─────────────────────┤
│                                                                 │
│  列权限控制：                                                    │
│  • 销售代表能看到：id, customer, amount(脱敏), created_by        │
│    看不到：cost_price                                           │
│                                                                 │
│  • 财务能看到：id, customer, amount(不脱敏), cost_price          │
│    可编辑：amount                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**详细列权限设计参考：** `architecture/07-field-permission.md`
```
