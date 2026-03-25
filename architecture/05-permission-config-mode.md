# 权限配置模式设计

## 1. 模式概述

支持应用级别的权限配置模式切换，满足不同业务场景的管理需求。

### 1.1 模式定义

| 模式 | 名称 | 说明 | 适用场景 |
|------|------|------|----------|
| **MODE_ROLE_DATA** | 角色绑定数据权限 | 数据权限范围在角色上配置，用户绑定角色即继承数据范围 | 组织架构固定、角色权限相对标准化的企业 |
| **MODE_USER_DATA** | 用户绑定角色时配置数据权限 | 角色只定义功能权限，数据范围在用户-角色绑定关系上配置 | 组织架构灵活、同一角色不同用户数据范围不同的场景 |

### 1.2 核心区别

```
模式1：角色绑定数据权限 (MODE_ROLE_DATA)
┌─────────────┐       ┌─────────────┐       ┌─────────────────────────┐
│    用户      │ ────→ │    角色      │ ────→ │ 功能权限 + 数据权限范围  │
│   张三       │       │  销售经理    │       │ 查看订单、本部门数据     │
└─────────────┘       └─────────────┘       └─────────────────────────┘
       │
       └──────────────────────────────────────────────────────────────┐
                             ↓                                       │
                    数据范围 = 角色配置的数据范围                       │
                    张三能看到：本部门的订单数据                        │

模式2：用户绑定角色时配置数据权限 (MODE_USER_DATA)
┌─────────────┐       ┌─────────────┐       ┌─────────────────┐
│    用户      │ ────→ │    角色      │ ────→ │    功能权限      │
│   张三       │       │  销售经理    │       │ 查看订单        │
└─────────────┘       └─────────────┘       └─────────────────┘
       │                                                      ↑
       └───────────────────┬──────────────────────────────────┘
                           │
                    用户-角色绑定关系
                    ┌────────────────────────────────────────┐
                    │  数据权限范围：张三-华东区               │
                    │  数据权限范围：李四-华北区               │
                    └────────────────────────────────────────┘

                    张三能看到：华东区的订单数据
                    李四能看到：华北区的订单数据
                    （同一角色，不同数据范围）
```

## 2. 应用配置

### 2.1 应用元数据

```yaml
# 应用配置 DSL
app:
  id: "app_crm"
  name: "CRM客户管理系统"

  # 权限配置模式
  permission_config:
    mode: "MODE_ROLE_DATA"    # MODE_ROLE_DATA | MODE_USER_DATA
    # 模式一旦选择后不建议切换，切换需要重新配置所有权限数据
```

### 2.2 数据模型

```sql
-- 应用表
CREATE TABLE sys_app (
    id              VARCHAR(32) PRIMARY KEY,
    app_code        VARCHAR(64) NOT NULL COMMENT '应用编码',
    app_name        VARCHAR(128) NOT NULL COMMENT '应用名称',
    permission_mode VARCHAR(32) NOT NULL DEFAULT 'MODE_ROLE_DATA' COMMENT '权限配置模式',
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- 模式枚举
-- MODE_ROLE_DATA: 角色绑定数据权限（默认）
-- MODE_USER_DATA: 用户绑定角色时配置数据权限
```

## 3. 模式1：角色绑定数据权限

### 3.1 数据模型

```sql
-- 角色表（与现有设计一致）
CREATE TABLE sys_role (
    id          VARCHAR(32) PRIMARY KEY,
    app_id      VARCHAR(32) NOT NULL COMMENT '应用ID',
    role_code   VARCHAR(64) NOT NULL COMMENT '角色编码',
    role_name   VARCHAR(128) NOT NULL COMMENT '角色名称',
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_app_role (app_id, role_code)
);

-- 角色功能权限关联表
CREATE TABLE sys_role_functional_permission (
    id              VARCHAR(32) PRIMARY KEY,
    role_id         VARCHAR(32) NOT NULL COMMENT '角色ID',
    permission_code VARCHAR(128) NOT NULL COMMENT '权限码',
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 角色数据权限关联表
CREATE TABLE sys_role_data_permission (
    id              VARCHAR(32) PRIMARY KEY,
    role_id         VARCHAR(32) NOT NULL COMMENT '角色ID',
    entity_code     VARCHAR(64) NOT NULL COMMENT '实体编码',
    rule_id         VARCHAR(32) NOT NULL COMMENT '数据规则ID',
    operations      VARCHAR(32) COMMENT '操作类型：READ,WRITE',
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 用户角色关联表（简单关联）
CREATE TABLE sys_user_role (
    id          VARCHAR(32) PRIMARY KEY,
    app_id      VARCHAR(32) NOT NULL COMMENT '应用ID',
    user_id     VARCHAR(32) NOT NULL COMMENT '用户ID',
    role_id     VARCHAR(32) NOT NULL COMMENT '角色ID',
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_user_role (app_id, user_id, role_id)
);
```

### 3.2 配置流程

```yaml
# 步骤1：创建数据规则（应用配置期）
data_rules:
  - id: "rule_dept_and_sub"
    name: "本部门及子部门"
    expression: "dept_id IN ${currentDeptTree}"

# 步骤2：创建角色并绑定权限（应用运行期）
roles:
  - code: "role_sales_manager"
    name: "销售经理"
    permissions:
      functional:           # 功能权限在角色上配置
        - "page_order_list"
        - "btn_order_view"
      data:                 # 数据权限在角色上配置
        - entity: "order"
          rule: "rule_dept_and_sub"
          operations: ["READ", "WRITE"]

# 步骤3：用户绑定角色
users:
  - user_id: "user_zhangsan"
    roles:
      - "role_sales_manager"    # 直接绑定角色，继承角色的所有权限
```

## 4. 模式2：用户绑定角色时配置数据权限

### 4.1 数据模型

```sql
-- 角色表（与模式1相同）
CREATE TABLE sys_role (
    id          VARCHAR(32) PRIMARY KEY,
    app_id      VARCHAR(32) NOT NULL COMMENT '应用ID',
    role_code   VARCHAR(64) NOT NULL COMMENT '角色编码',
    role_name   VARCHAR(128) NOT NULL COMMENT '角色名称',
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_app_role (app_id, role_code)
);

-- 角色功能权限关联表（与模式1相同）
CREATE TABLE sys_role_functional_permission (
    id              VARCHAR(32) PRIMARY KEY,
    role_id         VARCHAR(32) NOT NULL COMMENT '角色ID',
    permission_code VARCHAR(128) NOT NULL COMMENT '权限码',
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 用户角色数据权限关联表（模式2核心差异：数据权限在用户-角色关系上）
CREATE TABLE sys_user_role_data_permission (
    id              VARCHAR(32) PRIMARY KEY,
    user_role_id    VARCHAR(32) NOT NULL COMMENT '用户角色关联ID',
    entity_code     VARCHAR(64) NOT NULL COMMENT '实体编码',
    rule_id         VARCHAR(32) NOT NULL COMMENT '数据规则ID',
    operations      VARCHAR(32) COMMENT '操作类型：READ,WRITE',
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_user_role_entity (user_role_id, entity_code)
);

-- 用户角色关联表（模式2需要独立ID用于关联数据权限）
CREATE TABLE sys_user_role (
    id          VARCHAR(32) PRIMARY KEY,
    app_id      VARCHAR(32) NOT NULL COMMENT '应用ID',
    user_id     VARCHAR(32) NOT NULL COMMENT '用户ID',
    role_id     VARCHAR(32) NOT NULL COMMENT '角色ID',
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_user_role (app_id, user_id, role_id)
);
```

### 4.2 配置流程

```yaml
# 步骤1：创建数据规则（应用配置期，与模式1相同）
data_rules:
  - id: "rule_region_east"
    name: "华东区数据"
    expression: "region_code = 'EAST'"

  - id: "rule_region_north"
    name: "华北区数据"
    expression: "region_code = 'NORTH'"

# 步骤2：创建角色并绑定功能权限（应用运行期）
roles:
  - code: "role_sales_rep"
    name: "销售代表"
    permissions:
      functional:           # 角色只配置功能权限
        - "page_order_list"
        - "btn_order_view"
      # 注意：角色不配置数据权限

# 步骤3：用户绑定角色，同时配置数据权限
users:
  - user_id: "user_zhangsan"
    role_bindings:
      - role: "role_sales_rep"
        data_permissions:   # 数据权限在用户-角色绑定关系上配置
          - entity: "order"
            rule: "rule_region_east"      # 张三负责华东区
            operations: ["READ", "WRITE"]
          - entity: "customer"
            rule: "rule_region_east"
            operations: ["READ"]

  - user_id: "user_lisi"
    role_bindings:
      - role: "role_sales_rep"
        data_permissions:
          - entity: "order"
            rule: "rule_region_north"     # 李四负责华北区
            operations: ["READ", "WRITE"]
```

### 4.3 模式2的权限计算

```java
/**
 * 模式2：用户绑定角色时配置数据权限的权限计算
 */
@Component
public class ModeUserDataPermissionCalculator {

    @Autowired
    private UserRoleRepository userRoleRepository;

    @Autowired
    private UserRoleDataPermissionRepository userRoleDataPermissionRepository;

    /**
     * 计算用户的数据权限
     */
    public DataPermissionResult calculateDataPermission(
            String appId,
            String userId,
            String entityCode,
            OperationType operationType) {

        // 1. 获取用户的所有角色绑定关系
        List<UserRole> userRoles = userRoleRepository
            .findByAppIdAndUserId(appId, userId);

        // 2. 从用户-角色绑定关系中获取数据权限
        List<DataRule> rules = new ArrayList<>();

        for (UserRole userRole : userRoles) {
            // 查询该用户在该角色下的数据权限配置
            List<UserRoleDataPermission> dataPerms = userRoleDataPermissionRepository
                .findByUserRoleIdAndEntityCode(userRole.getId(), entityCode);

            for (UserRoleDataPermission dataPerm : dataPerms) {
                // 检查操作类型匹配
                if (dataPerm.hasOperation(operationType)) {
                    DataRule rule = dataRuleRepository.findById(dataPerm.getRuleId());
                    rules.add(rule);
                }
            }
        }

        // 3. 合并规则（OR关系）
        if (rules.isEmpty()) {
            return DataPermissionResult.noAccess();
        }

        return DataPermissionResult.withRules(rules);
    }
}
```

## 5. 统一权限引擎适配

### 5.1 权限计算路由

```java
/**
 * 权限计算路由，根据应用的权限模式选择不同的计算器
 */
@Component
public class PermissionCalculatorRouter {

    @Autowired
    private ModeRoleDataCalculator modeRoleDataCalculator;

    @Autowired
    private ModeUserDataCalculator modeUserDataCalculator;

    @Autowired
    private AppRepository appRepository;

    /**
     * 计算功能权限
     */
    public PermissionResult calculateFunctionalPermission(
            String appId, String userId, String permissionCode) {
        // 功能权限两种模式计算方式相同
        return modeRoleDataCalculator.calculateFunctionalPermission(appId, userId, permissionCode);
    }

    /**
     * 计算数据权限
     */
    public DataPermissionResult calculateDataPermission(
            String appId, String userId, String entityCode, OperationType operationType) {

        // 获取应用的权限配置模式
        App app = appRepository.findById(appId);
        PermissionMode mode = app.getPermissionMode();

        // 根据模式路由到不同的计算器
        return switch (mode) {
            case MODE_ROLE_DATA ->
                modeRoleDataCalculator.calculateDataPermission(appId, userId, entityCode, operationType);
            case MODE_USER_DATA ->
                modeUserDataCalculator.calculateDataPermission(appId, userId, entityCode, operationType);
        };
    }
}
```

### 5.2 模式1计算器

```java
@Component
public class ModeRoleDataCalculator {

    @Autowired
    private RoleService roleService;

    /**
     * 计算功能权限
     */
    public PermissionResult calculateFunctionalPermission(
            String appId, String userId, String permissionCode) {

        // 获取用户所有角色
        List<Role> roles = roleService.getUserRoles(appId, userId);

        // 合并角色的功能权限
        Set<String> permissions = roles.stream()
            .flatMap(r -> r.getFunctionalPermissions().stream())
            .collect(Collectors.toSet());

        boolean allowed = permissions.contains("*") || permissions.contains(permissionCode);
        return allowed ? PermissionResult.allow() : PermissionResult.deny();
    }

    /**
     * 计算数据权限 - 从角色获取
     */
    public DataPermissionResult calculateDataPermission(
            String appId, String userId, String entityCode, OperationType operationType) {

        List<Role> roles = roleService.getUserRoles(appId, userId);

        // 检查是否有无限制角色
        boolean hasAllAccess = roles.stream()
            .anyMatch(r -> r.getDataStrategy(entityCode) == DataStrategy.ALL);
        if (hasAllAccess) {
            return DataPermissionResult.noRestriction();
        }

        // 从角色获取数据规则
        List<DataRule> rules = roles.stream()
            .flatMap(r -> r.getDataRules().stream())
            .filter(r -> r.getEntity().equals(entityCode))
            .filter(r -> r.hasOperation(operationType))
            .collect(Collectors.toList());

        if (rules.isEmpty()) {
            return DataPermissionResult.noAccess();
        }

        return DataPermissionResult.withRules(rules);
    }
}
```

### 5.3 模式2计算器

```java
@Component
public class ModeUserDataCalculator {

    @Autowired
    private UserRoleRepository userRoleRepository;

    @Autowired
    private UserRoleDataPermissionRepository userRoleDataPermissionRepository;

    /**
     * 计算功能权限 - 与模式1相同
     */
    public PermissionResult calculateFunctionalPermission(
            String appId, String userId, String permissionCode) {

        List<UserRole> userRoles = userRoleRepository.findByAppIdAndUserId(appId, userId);

        // 获取所有角色的功能权限并集
        Set<String> permissions = userRoles.stream()
            .map(UserRole::getRole)
            .flatMap(r -> r.getFunctionalPermissions().stream())
            .collect(Collectors.toSet());

        boolean allowed = permissions.contains("*") || permissions.contains(permissionCode);
        return allowed ? PermissionResult.allow() : PermissionResult.deny();
    }

    /**
     * 计算数据权限 - 从用户-角色绑定关系获取
     */
    public DataPermissionResult calculateDataPermission(
            String appId, String userId, String entityCode, OperationType operationType) {

        List<UserRole> userRoles = userRoleRepository.findByAppIdAndUserId(appId, userId);

        // 从用户-角色绑定关系获取数据规则
        List<DataRule> rules = new ArrayList<>();

        for (UserRole userRole : userRoles) {
            List<UserRoleDataPermission> dataPerms = userRoleDataPermissionRepository
                .findByUserRoleId(userRole.getId());

            dataPerms.stream()
                .filter(dp -> dp.getEntityCode().equals(entityCode))
                .filter(dp -> dp.hasOperation(operationType))
                .forEach(dp -> {
                    DataRule rule = dataRuleRepository.findById(dp.getRuleId());
                    rules.add(rule);
                });
        }

        if (rules.isEmpty()) {
            return DataPermissionResult.noAccess();
        }

        return DataPermissionResult.withRules(rules);
    }
}
```

## 6. 管理界面设计

### 6.1 应用创建时选择权限模式

```
┌─────────────────────────────────────────────────────────────────┐
│                       创建新应用                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  应用名称:  [CRM客户管理系统                    ]                │
│                                                                 │
│  应用编码:  [app_crm                            ]                │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  权限配置模式:                                                   │
│                                                                 │
│  ○ 模式1：角色绑定数据权限                                       │
│    └─ 数据权限范围在角色上配置，用户绑定角色即继承数据范围         │
│    └─ 适用于：组织架构固定、角色权限标准化的企业                  │
│                                                                 │
│  ○ 模式2：用户绑定角色时配置数据权限                              │
│    └─ 角色只定义功能权限，数据范围在用户-角色绑定关系上配置        │
│    └─ 适用于：组织架构灵活、同一角色不同用户数据范围不同的场景     │
│                                                                 │
│  ⚠️ 注意：权限模式一旦选择后不可更改                              │
│                                                                 │
│           [  取消  ]    [  创建应用  ]                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 模式1：角色管理界面

```
┌─────────────────────────────────────────────────────────────────┐
│                       角色：销售经理                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────┐  ┌─────────────────────────────────────────┐ │
│  │ 功能权限       │  │ 数据权限                                 │ │
│  │               │  │                                         │ │
│  │ ☑ 页面管理     │  │ 实体: 订单                               │ │
│  │ ☑ 订单列表     │  │ 规则: 本部门及子部门                      │ │
│  │ ☑ 订单查看     │  │ 操作: ☑读 ☑写                           │ │
│  │ ☐ 订单编辑     │  │                                         │ │
│  │ ☐ 订单删除     │  │ 实体: 客户                               │ │
│  │               │  │ 规则: 本部门数据                          │ │
│  │               │  │ 操作: ☑读 ☐写                           │ │
│  └───────────────┘  │                                         │ │
│                      │ [+ 添加数据权限]                         │ │
│                      └─────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.3 模式2：用户角色绑定界面

```
┌─────────────────────────────────────────────────────────────────┐
│                    用户：张三 的角色管理                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  已绑定角色:                                                     │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ ☑ 销售代表                                            [删除] ││
│  │                                                             ││
│  │   功能权限: 页面管理、订单列表、订单查看                      ││
│  │                                                             ││
│  │   数据权限配置:                                              ││
│  │   ┌─────────────┬──────────────────────┬──────────────────┐ ││
│  │   │ 实体        │ 数据范围              │ 操作             │ ││
│  │   ├─────────────┼──────────────────────┼──────────────────┤ ││
│  │   │ 订单        │ 华东区数据            │ 读 ☑  写 ☑       │ ││
│  │   │ 客户        │ 华东区数据            │ 读 ☑  写 ☐       │ ││
│  │   └─────────────┴──────────────────────┴──────────────────┘ ││
│  │   [+ 添加实体]                                              ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ ☑ 数据分析师                                          [删除] ││
│  │                                                             ││
│  │   功能权限: 报表查看、数据导出                               ││
│  │                                                             ││
│  │   数据权限配置:                                              ││
│  │   ┌─────────────┬──────────────────────┬──────────────────┐ ││
│  │   │ 实体        │ 数据范围              │ 操作             │ ││
│  │   ├─────────────┼──────────────────────┼──────────────────┤ ││
│  │   │ 报表        │ 全部数据              │ 读 ☑  写 ☐       │ ││
│  │   └─────────────┴──────────────────────┴──────────────────┘ ││
│  │   [+ 添加实体]                                              ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│           [+ 添加角色]                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 7. 模式对比总结

| 维度 | 模式1：角色绑定数据权限 | 模式2：用户绑定角色时配置数据权限 |
|------|------------------------|----------------------------------|
| **核心思想** | 角色是权限的完整集合 | 角色是功能模板，数据范围个性化 |
| **角色数量** | 角色随数据范围增加而增加 | 角色数量精简，数据范围在用户层配置 |
| **管理复杂度** | 角色管理复杂，用户管理简单 | 角色管理简单，用户管理复杂 |
| **权限粒度** | 按角色批量分配 | 按用户精细分配 |
| **适用组织** | 层级清晰、标准化的企业 | 扁平化、灵活组织架构 |
| **典型场景** | 部门经理看本部门数据 | 大区经理各自负责不同区域 |
| **数据库表** | 4张表 | 5张表（多一张用户角色数据权限表） |
| **缓存策略** | 按角色缓存 | 按用户-角色关系缓存 |

## 8. 迁移与切换

### 8.1 模式切换的限制

```yaml
# 应用创建后，权限模式原则上不允许切换
# 原因：两种模式的数据存储结构不同

切换风险:
  MODE_ROLE_DATA → MODE_USER_DATA:
    - 需要将所有角色的数据权限迁移到每个用户的绑定关系上
    - 如果同一角色有100个用户，需要复制100份数据权限配置
    - 丢失"统一调整角色数据范围"的能力

  MODE_USER_DATA → MODE_ROLE_DATA:
    - 需要决定：同一角色不同用户的数据范围冲突如何解决
    - 可能需要创建更多角色来覆盖差异
    - 可能丢失个性化的数据范围配置
```

### 8.2 强制切换的处理方案

```java
/**
 * 权限模式迁移服务（谨慎使用）
 */
@Service
public class PermissionModeMigrationService {

    /**
     * 模式1 → 模式2：复制角色数据权限到用户绑定关系
     */
    @Transactional
    public void migrateRoleDataToUserData(String appId) {
        // 1. 获取应用下所有用户角色关系
        List<UserRole> userRoles = userRoleRepository.findByAppId(appId);

        // 2. 对每个用户-角色关系，复制角色的数据权限
        for (UserRole userRole : userRoles) {
            Role role = roleRepository.findById(userRole.getRoleId());

            for (RoleDataPermission roleDataPerm : role.getDataPermissions()) {
                UserRoleDataPermission userRoleDataPerm = new UserRoleDataPermission();
                userRoleDataPerm.setUserRoleId(userRole.getId());
                userRoleDataPerm.setEntityCode(roleDataPerm.getEntityCode());
                userRoleDataPerm.setRuleId(roleDataPerm.getRuleId());
                userRoleDataPerm.setOperations(roleDataPerm.getOperations());
                userRoleDataPermissionRepository.save(userRoleDataPerm);
            }
        }

        // 3. 更新应用模式
        appRepository.updatePermissionMode(appId, PermissionMode.MODE_USER_DATA);
    }
}
```
