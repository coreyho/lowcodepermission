# 服务接口定义

## 1. PermissionService（权限服务）

### 1.1 接口定义

```java
/**
 * 权限服务接口
 */
public interface PermissionService {
    
    // ==================== 应用初始化 ====================
    
    /**
     * 初始化应用权限（应用发布时调用）
     * @param appId 应用ID
     * @param dsl 权限配置DSL
     */
    void initAppPermissions(String appId, PermissionConfigDSL dsl);
    
    /**
     * 升级应用权限配置
     * @param appId 应用ID
     * @param newDsl 新的权限配置DSL
     * @param migrateStrategy 迁移策略
     */
    void upgradeAppPermissions(String appId, PermissionConfigDSL newDsl, 
                               MigrateStrategy migrateStrategy);
    
    // ==================== 功能权限校验 ====================
    
    /**
     * 校验单个权限
     * @param appId 应用ID
     * @param userId 用户ID
     * @param permissionCode 权限码
     * @return 是否拥有权限
     */
    boolean checkPermission(String appId, String userId, String permissionCode);
    
    /**
     * 批量校验权限
     * @param appId 应用ID
     * @param userId 用户ID
     * @param permissionCodes 权限码列表
     * @return 权限校验结果映射
     */
    Map<String, Boolean> checkPermissions(String appId, String userId, 
                                          List<String> permissionCodes);
    
    /**
     * 校验任意权限（拥有其中一个即可）
     * @param appId 应用ID
     * @param userId 用户ID
     * @param permissionCodes 权限码列表
     * @return 是否拥有任意一个权限
     */
    boolean checkAnyPermission(String appId, String userId, 
                               List<String> permissionCodes);
    
    /**
     * 校验所有权限（必须全部拥有）
     * @param appId 应用ID
     * @param userId 用户ID
     * @param permissionCodes 权限码列表
     * @return 是否拥有全部权限
     */
    boolean checkAllPermissions(String appId, String userId, 
                                List<String> permissionCodes);
    
    /**
     * 获取用户菜单（根据权限过滤）
     * @param appId 应用ID
     * @param userId 用户ID
     * @return 菜单树
     */
    List<MenuTreeNode> getUserMenus(String appId, String userId);
    
    /**
     * 获取用户所有权限码
     * @param appId 应用ID
     * @param userId 用户ID
     * @return 权限码集合
     */
    Set<String> getUserPermissionCodes(String appId, String userId);
    
    /**
     * 获取用户完整权限信息
     * @param appId 应用ID
     * @param userId 用户ID
     * @return 用户权限对象
     */
    UserPermission getUserPermissions(String appId, String userId);
    
    // ==================== 数据权限 ====================
    
    /**
     * 获取数据权限过滤器
     * @param appId 应用ID
     * @param userId 用户ID
     * @param entityCode 实体编码
     * @return 数据过滤器
     */
    DataFilter getDataFilter(String appId, String userId, String entityCode);
    
    /**
     * 校验数据访问权限
     * @param appId 应用ID
     * @param userId 用户ID
     * @param entityCode 实体编码
     * @param dataId 数据ID
     * @param action 操作类型
     * @return 是否有权限访问
     */
    boolean checkDataAccess(String appId, String userId, String entityCode, 
                            String dataId, DataAccessAction action);
    
    /**
     * 获取数据权限SQL条件
     * @param appId 应用ID
     * @param userId 用户ID
     * @param entityCode 实体编码
     * @return SQL条件片段
     */
    String getDataPermissionCondition(String appId, String userId, String entityCode);
    
    // ==================== 缓存管理 ====================
    
    /**
     * 刷新用户权限缓存
     * @param appId 应用ID
     * @param userId 用户ID
     */
    void refreshUserCache(String appId, String userId);
    
    /**
     * 批量刷新用户权限缓存
     * @param appId 应用ID
     * @param userIds 用户ID列表
     */
    void refreshUserCacheBatch(String appId, List<String> userIds);
    
    /**
     * 刷新应用下所有用户缓存
     * @param appId 应用ID
     */
    void refreshAppCache(String appId);
    
    /**
     * 清除用户缓存
     * @param appId 应用ID
     * @param userId 用户ID
     */
    void clearUserCache(String appId, String userId);
}
```

### 1.2 DTO定义

```java
/**
 * 用户权限对象
 */
@Data
public class UserPermission {
    
    /** 用户ID */
    private String userId;
    
    /** 应用ID */
    private String appId;
    
    /** 权限码集合 */
    private Set<String> permissionCodes;
    
    /** 数据权限规则映射（实体 -> 规则） */
    private Map<String, DataRule> dataRules;
    
    /** 角色ID列表 */
    private List<Long> roleIds;
    
    /** 缓存版本 */
    private Long version;
    
    /** 缓存过期时间 */
    private LocalDateTime expireAt;
}

/**
 * 数据过滤器
 */
@Data
public class DataFilter {
    
    /** 实体编码 */
    private String entityCode;
    
    /** 过滤器类型 */
    private FilterType type;
    
    /** 字段名（FIELD_EQUAL/FIELD_IN类型） */
    private String field;
    
    /** 单值（FIELD_EQUAL类型） */
    private Object value;
    
    /** 多值（FIELD_IN类型） */
    private List<Object> values;
    
    /** 自定义表达式（CUSTOM类型） */
    private String expression;
    
    /** 是否无限制 */
    private boolean noRestriction;
    
    public static DataFilter noRestriction() {
        DataFilter filter = new DataFilter();
        filter.setNoRestriction(true);
        return filter;
    }
    
    public static DataFilter withCondition(String field, Object value) {
        DataFilter filter = new DataFilter();
        filter.setType(FilterType.FIELD_EQUAL);
        filter.setField(field);
        filter.setValue(value);
        return filter;
    }
}

/**
 * 数据权限规则
 */
@Data
public class DataRule {
    
    /** 规则ID */
    private Long id;
    
    /** 实体编码 */
    private String entityCode;
    
    /** 策略类型 */
    private DataStrategy strategy;
    
    /** 规则表达式 */
    private String expression;
    
    /** 字段条件（field_based类型） */
    private List<FieldCondition> conditions;
}

/**
 * 字段条件
 */
@Data
public class FieldCondition {
    
    /** 字段名 */
    private String field;
    
    /** 操作符 */
    private Operator operator;
    
    /** 值 */
    private Object value;
    
    /** 逻辑关系（AND/OR） */
    private Logic logic;
}

/**
 * 菜单树节点
 */
@Data
public class MenuTreeNode {
    
    /** 权限码 */
    private String code;
    
    /** 名称 */
    private String name;
    
    /** 路径 */
    private String path;
    
    /** 图标 */
    private String icon;
    
    /** 子菜单 */
    private List<MenuTreeNode> children;
    
    /** 是否隐藏 */
    private boolean hidden;
    
    /** 排序序号 */
    private Integer sortOrder;
}
```

## 2. RoleService（角色服务）

### 2.1 接口定义

```java
/**
 * 角色服务接口
 */
public interface RoleService {
    
    // ==================== 角色CRUD ====================
    
    /**
     * 创建角色
     * @param appId 应用ID
     * @param request 创建请求
     * @return 创建的角色
     */
    Role createRole(String appId, RoleCreateRequest request);
    
    /**
     * 更新角色
     * @param appId 应用ID
     * @param roleId 角色ID
     * @param request 更新请求
     * @return 更新后的角色
     */
    Role updateRole(String appId, Long roleId, RoleUpdateRequest request);
    
    /**
     * 删除角色
     * @param appId 应用ID
     * @param roleId 角色ID
     */
    void deleteRole(String appId, Long roleId);
    
    /**
     * 获取角色详情
     * @param appId 应用ID
     * @param roleId 角色ID
     * @return 角色详情
     */
    Role getRole(String appId, Long roleId);
    
    /**
     * 查询角色列表
     * @param appId 应用ID
     * @param query 查询条件
     * @return 角色列表
     */
    List<Role> listRoles(String appId, RoleQuery query);
    
    /**
     * 分页查询角色
     * @param appId 应用ID
     * @param query 查询条件
     * @return 分页结果
     */
    PageResult<Role> pageRoles(String appId, RolePageQuery query);
    
    // ==================== 模板继承 ====================
    
    /**
     * 从模板克隆角色
     * @param appId 应用ID
     * @param templateCode 模板编码
     * @param request 克隆请求
     * @return 克隆的角色
     */
    Role cloneFromTemplate(String appId, String templateCode, RoleCloneRequest request);
    
    /**
     * 获取应用的角色模板列表
     * @param appId 应用ID
     * @return 模板列表
     */
    List<RoleTemplate> getRoleTemplates(String appId);
    
    /**
     * 应用角色模板到角色
     * @param appId 应用ID
     * @param roleId 角色ID
     * @param templateCode 模板编码
     */
    void applyTemplate(String appId, Long roleId, String templateCode);
    
    // ==================== 角色-权限关联 ====================
    
    /**
     * 分配权限给角色
     * @param appId 应用ID
     * @param roleId 角色ID
     * @param permissionCodes 权限码列表
     */
    void assignPermissions(String appId, Long roleId, List<String> permissionCodes);
    
    /**
     * 移除角色的权限
     * @param appId 应用ID
     * @param roleId 角色ID
     * @param permissionCodes 权限码列表
     */
    void revokePermissions(String appId, Long roleId, List<String> permissionCodes);
    
    /**
     * 设置角色权限（全量替换）
     * @param appId 应用ID
     * @param roleId 角色ID
     * @param permissionCodes 权限码列表
     */
    void setPermissions(String appId, Long roleId, List<String> permissionCodes);
    
    /**
     * 获取角色的权限列表
     * @param appId 应用ID
     * @param roleId 角色ID
     * @return 权限列表
     */
    List<Permission> getRolePermissions(String appId, Long roleId);
    
    // ==================== 角色-数据规则关联 ====================
    
    /**
     * 绑定数据规则到角色
     * @param appId 应用ID
     * @param roleId 角色ID
     * @param entityCode 实体编码
     * @param ruleId 规则ID
     */
    void bindDataRule(String appId, Long roleId, String entityCode, Long ruleId);
    
    /**
     * 解绑角色的数据规则
     * @param appId 应用ID
     * @param roleId 角色ID
     * @param entityCode 实体编码
     */
    void unbindDataRule(String appId, Long roleId, String entityCode);
    
    /**
     * 设置角色的数据策略为ALL
     * @param appId 应用ID
     * @param roleId 角色ID
     * @param entityCode 实体编码
     */
    void setDataStrategyAll(String appId, Long roleId, String entityCode);
    
    // ==================== 用户-角色关联 ====================
    
    /**
     * 分配角色给用户
     * @param appId 应用ID
     * @param userId 用户ID
     * @param roleIds 角色ID列表
     */
    void assignRolesToUser(String appId, String userId, List<Long> roleIds);
    
    /**
     * 移除用户的角色
     * @param appId 应用ID
     * @param userId 用户ID
     * @param roleIds 角色ID列表
     */
    void revokeRolesFromUser(String appId, String userId, List<Long> roleIds);
    
    /**
     * 设置用户的角色（全量替换）
     * @param appId 应用ID
     * @param userId 用户ID
     * @param roleIds 角色ID列表
     */
    void setUserRoles(String appId, String userId, List<Long> roleIds);
    
    /**
     * 获取用户的角色列表
     * @param appId 应用ID
     * @param userId 用户ID
     * @return 角色列表
     */
    List<Role> getUserRoles(String appId, String userId);
    
    /**
     * 获取拥有某角色的所有用户
     * @param appId 应用ID
     * @param roleId 角色ID
     * @return 用户ID列表
     */
    List<String> getRoleUsers(String appId, Long roleId);
    
    // ==================== 部门-角色关联 ====================
    
    /**
     * 设置部门的默认角色
     * @param appId 应用ID
     * @param deptId 部门ID
     * @param roleIds 角色ID列表
     */
    void setDeptDefaultRoles(String appId, String deptId, List<Long> roleIds);
    
    /**
     * 获取部门的默认角色
     * @param appId 应用ID
     * @param deptId 部门ID
     * @return 角色列表
     */
    List<Role> getDeptDefaultRoles(String appId, String deptId);
}
```

## 3. DataPermissionService（数据权限服务）

```java
/**
 * 数据权限服务接口
 */
public interface DataPermissionService {
    
    // ==================== 数据维度管理 ====================
    
    /**
     * 创建数据权限维度
     * @param appId 应用ID
     * @param request 创建请求
     * @return 维度对象
     */
    DataDimension createDimension(String appId, DimensionCreateRequest request);
    
    /**
     * 更新数据权限维度
     * @param appId 应用ID
     * @param dimensionId 维度ID
     * @param request 更新请求
     */
    void updateDimension(String appId, Long dimensionId, DimensionUpdateRequest request);
    
    /**
     * 删除数据权限维度
     * @param appId 应用ID
     * @param dimensionId 维度ID
     */
    void deleteDimension(String appId, Long dimensionId);
    
    /**
     * 获取维度列表
     * @param appId 应用ID
     * @return 维度列表
     */
    List<DataDimension> listDimensions(String appId);
    
    // ==================== 数据规则管理 ====================
    
    /**
     * 创建数据规则
     * @param appId 应用ID
     * @param request 创建请求
     * @return 规则对象
     */
    DataRule createRule(String appId, DataRuleCreateRequest request);
    
    /**
     * 更新数据规则
     * @param appId 应用ID
     * @param ruleId 规则ID
     * @param request 更新请求
     */
    void updateRule(String appId, Long ruleId, DataRuleUpdateRequest request);
    
    /**
     * 删除数据规则
     * @param appId 应用ID
     * @param ruleId 规则ID
     */
    void deleteRule(String appId, Long ruleId);
    
    /**
     * 获取规则详情
     * @param appId 应用ID
     * @param ruleId 规则ID
     * @return 规则对象
     */
    DataRule getRule(String appId, Long ruleId);
    
    /**
     * 查询规则列表
     * @param appId 应用ID
     * @param query 查询条件
     * @return 规则列表
     */
    List<DataRule> listRules(String appId, DataRuleQuery query);
    
    /**
     * 测试规则
     * @param appId 应用ID
     * @param ruleId 规则ID
     * @param testData 测试数据
     * @return 测试结果
     */
    RuleTestResult testRule(String appId, Long ruleId, Map<String, Object> testData);
    
    // ==================== 规则模板 ====================
    
    /**
     * 创建规则模板
     * @param appId 应用ID
     * @param request 创建请求
     * @return 模板对象
     */
    RuleTemplate createRuleTemplate(String appId, RuleTemplateCreateRequest request);
    
    /**
     * 获取规则模板列表
     * @param appId 应用ID
     * @return 模板列表
     */
    List<RuleTemplate> listRuleTemplates(String appId);
    
    /**
     * 从模板创建规则
     * @param appId 应用ID
     * @param templateCode 模板编码
     * @param request 创建请求
     * @return 规则对象
     */
    DataRule createRuleFromTemplate(String appId, String templateCode, 
                                    RuleFromTemplateRequest request);
    
    // ==================== 数据权限计算 ====================
    
    /**
     * 计算用户的数据权限条件
     * @param appId 应用ID
     * @param userId 用户ID
     * @param entityCode 实体编码
     * @return SQL条件
     */
    String calculateDataCondition(String appId, String userId, String entityCode);
    
    /**
     * 验证数据访问权限
     * @param appId 应用ID
     * @param userId 用户ID
     * @param entityCode 实体编码
     * @param data 数据对象
     * @param action 操作类型
     * @return 是否有权限
     */
    boolean validateDataAccess(String appId, String userId, String entityCode,
                                Map<String, Object> data, DataAccessAction action);
}
```

## 4. PermissionAdminService（权限管理服务）

```java
/**
 * 权限管理服务接口（供管理后台使用）
 */
public interface PermissionAdminService {
    
    // ==================== 权限管理 ====================
    
    /**
     * 创建权限
     * @param appId 应用ID
     * @param request 创建请求
     * @return 权限对象
     */
    Permission createPermission(String appId, PermissionCreateRequest request);
    
    /**
     * 批量创建权限
     * @param appId 应用ID
     * @param requests 创建请求列表
     * @return 权限对象列表
     */
    List<Permission> batchCreatePermissions(String appId, 
                                            List<PermissionCreateRequest> requests);
    
    /**
     * 更新权限
     * @param appId 应用ID
     * @param permissionId 权限ID
     * @param request 更新请求
     */
    void updatePermission(String appId, Long permissionId, PermissionUpdateRequest request);
    
    /**
     * 删除权限
     * @param appId 应用ID
     * @param permissionId 权限ID
     */
    void deletePermission(String appId, Long permissionId);
    
    /**
     * 获取权限树
     * @param appId 应用ID
     * @return 权限树
     */
    List<PermissionTreeNode> getPermissionTree(String appId);
    
    /**
     * 导入权限DSL
     * @param appId 应用ID
     * @param dslContent DSL内容
     * @param importStrategy 导入策略
     */
    void importPermissionDSL(String appId, String dslContent, ImportStrategy importStrategy);
    
    /**
     * 导出权限DSL
     * @param appId 应用ID
     * @return DSL内容
     */
    String exportPermissionDSL(String appId);
    
    // ==================== 权限审计 ====================
    
    /**
     * 查询权限操作日志
     * @param appId 应用ID
     * @param query 查询条件
     * @return 日志列表
     */
    PageResult<PermissionAuditLog> queryPermissionAuditLogs(String appId, 
                                                             AuditLogQuery query);
    
    /**
     * 查询访问审计日志
     * @param appId 应用ID
     * @param query 查询条件
     * @return 日志列表
     */
    PageResult<AccessAuditLog> queryAccessAuditLogs(String appId, AccessLogQuery query);
    
    // ==================== 特殊配置 ====================
    
    /**
     * 添加脱敏白名单
     * @param appId 应用ID
     * @param request 添加请求
     */
    void addMaskingExemption(String appId, MaskingExemptionRequest request);
    
    /**
     * 移除脱敏白名单
     * @param appId 应用ID
     * @param exemptionId 白名单ID
     */
    void removeMaskingExemption(String appId, Long exemptionId);
    
    /**
     * 配置API限流
     * @param appId 应用ID
     * @param request 配置请求
     */
    void configureRateLimit(String appId, RateLimitConfigRequest request);
}
```
