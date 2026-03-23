# 权限引擎设计

## 1. 权限计算引擎架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                         权限计算引擎                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   ┌──────────────┐     ┌──────────────┐     ┌──────────────┐        │
│   │  权限加载器   │────→│  权限合并器   │────→│  权限缓存层   │        │
│   │  Loader      │     │  Merger      │     │  Redis       │        │
│   └──────────────┘     └──────────────┘     └──────────────┘        │
│          ↑                    ↑                    │                │
│          │                    │                    ↓                │
│   ┌──────────────┐     ┌──────────────┐     ┌──────────────┐        │
│   │  配置存储    │     │  规则引擎    │     │  决策点      │        │
│   │  MySQL       │     │  Drools/QLExpress  │  Enforcement │        │
│   └──────────────┘     └──────────────┘     └──────────────┘        │
│                                                    │                 │
│                              ┌─────────────────────┘                 │
│                              ↓                                      │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                    决策结果                                  │   │
│   │   { allow: true, fields: [...], dataScope: {...} }          │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.1 组件说明

| 组件 | 职责 | 技术选型 |
|------|------|----------|
| 权限加载器 | 从数据库加载权限配置 | MyBatis |
| 权限合并器 | 合并多角色权限 | 自研 |
| 规则引擎 | 解析数据权限表达式 | QLExpress / Drools |
| 权限缓存层 | 缓存权限计算结果 | Redis |
| 决策点 | 执行最终权限判断 | 自研 |

## 2. 功能权限校验流程

```java
/**
 * 权限决策流程
 */
@Component
public class PermissionEngine {
    
    @Autowired
    private PermissionCacheService cacheService;
    
    @Autowired
    private RoleService roleService;
    
    /**
     * 功能权限校验
     */
    public PermissionResult checkFunctionalPermission(
            String appId,
            String userId, 
            String permissionCode,
            PermissionContext context) {
        
        // 1. 尝试从缓存获取
        UserPermission cached = cacheService.getUserPermission(appId, userId);
        if (cached != null) {
            return checkFromCache(cached, permissionCode);
        }
        
        // 2. 获取用户所有角色
        List<Role> roles = roleService.getUserRoles(appId, userId);
        
        // 3. 收集所有权限码
        Set<String> permissions = roles.stream()
            .flatMap(r -> r.getPermissions().stream())
            .collect(Collectors.toSet());
        
        // 4. 超级管理员直接放行
        if (permissions.contains("*")) {
            return PermissionResult.allow();
        }
        
        // 5. 检查具体权限
        boolean allowed = permissions.contains(permissionCode);
        
        // 6. 更新缓存
        UserPermission userPermission = buildUserPermission(roles);
        cacheService.setUserPermission(appId, userId, userPermission);
        
        return allowed ? PermissionResult.allow() : PermissionResult.deny();
    }
    
    private PermissionResult checkFromCache(UserPermission cached, String permissionCode) {
        if (cached.getPermissions().contains("*")) {
            return PermissionResult.allow();
        }
        boolean allowed = cached.getPermissions().contains(permissionCode);
        return allowed ? PermissionResult.allow() : PermissionResult.deny();
    }
}
```

## 3. 数据权限 SQL 拦截器

### 3.1 拦截器实现

```java
/**
 * MyBatis Plus 数据权限拦截器
 */
@Component
@Intercepts({
    @Signature(type = Executor.class, method = "query", 
               args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class})
})
public class DataPermissionInterceptor implements Interceptor {
    
    @Autowired
    private PermissionEngine permissionEngine;
    
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        MappedStatement ms = (MappedStatement) invocation.getArgs()[0];
        Object parameter = invocation.getArgs()[1];
        
        // 获取当前用户
        String userId = SecurityContext.getCurrentUserId();
        String appId = TenantContext.getCurrentTenant();
        
        // 获取实体类型
        Class<?> entityClass = getEntityClass(ms);
        String entityCode = getEntityCode(entityClass);
        
        // 计算数据权限
        DataPermissionResult result = permissionEngine
            .calculateDataPermission(appId, userId, entityCode, buildContext(parameter));
        
        // 如果有数据权限限制，修改 SQL
        if (result.hasRestriction()) {
            String originalSql = getOriginalSql(ms, parameter);
            String filteredSql = applyDataFilter(originalSql, result.getFilter());
            return executeWithNewSql(invocation, filteredSql);
        }
        
        return invocation.proceed();
    }
    
    private String applyDataFilter(String sql, DataFilter filter) {
        try {
            Statement statement = CCJSqlParserUtil.parse(sql);
            
            if (statement instanceof Select) {
                Select select = (Select) statement;
                PlainSelect plainSelect = (PlainSelect) select.getSelectBody();
                
                // 构建权限条件
                Expression permissionExpr = buildPermissionExpression(filter);
                
                // 添加到 WHERE 子句
                Expression where = plainSelect.getWhere();
                if (where == null) {
                    plainSelect.setWhere(permissionExpr);
                } else {
                    plainSelect.setWhere(new AndExpression(where, permissionExpr));
                }
            }
            
            return statement.toString();
        } catch (JSQLParserException e) {
            log.error("SQL解析失败", e);
            throw new PermissionException("数据权限应用失败");
        }
    }
    
    private Expression buildPermissionExpression(DataFilter filter) {
        // 根据过滤器类型构建不同的表达式
        switch (filter.getType()) {
            case FIELD_EQUAL:
                return new EqualsTo(
                    new Column(filter.getField()),
                    new StringValue(filter.getValue().toString())
                );
            case FIELD_IN:
                ExpressionList items = new ExpressionList();
                for (Object value : filter.getValues()) {
                    items.add(new StringValue(value.toString()));
                }
                return new InExpression(new Column(filter.getField()), items);
            case CUSTOM:
                return CCJSqlParserUtil.parseCondExpression(filter.getExpression());
            default:
                throw new UnsupportedOperationException("不支持的过滤器类型");
        }
    }
}
```

### 3.2 拦截器注册

```java
@Configuration
public class MybatisConfig {
    
    @Autowired
    private DataPermissionInterceptor dataPermissionInterceptor;
    
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        
        // 数据权限拦截器
        interceptor.addInnerInterceptor(dataPermissionInterceptor);
        
        // 分页插件
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor());
        
        // 乐观锁插件
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
        
        return interceptor;
    }
}
```

## 4. 数据脱敏处理

### 4.1 脱敏服务实现

```java
@Component
public class DataMaskingService {
    
    public Object maskValue(Object value, MaskingConfig config) {
        if (value == null || !config.isEnable()) {
            return value;
        }
        
        String strValue = value.toString();
        
        switch (config.getType()) {
            case FULL:
                return StringUtils.repeat("*", strValue.length());
                
            case PARTIAL:
                int length = strValue.length();
                if (length <= 4) {
                    return "****";
                }
                return strValue.substring(0, 2) + 
                       StringUtils.repeat("*", length - 4) + 
                       strValue.substring(length - 2);
                       
            case REGEX:
                if (StringUtils.isNotBlank(config.getPattern())) {
                    return applyRegexMask(strValue, config.getPattern());
                }
                return "****";
                
            default:
                return value;
        }
    }
    
    /**
     * 对查询结果进行脱敏处理
     */
    public void maskQueryResult(List<Map<String, Object>> records, 
                                 String appId, 
                                 String pageCode,
                                 String userId) {
        // 检查用户是否有脱敏豁免权
        List<String> fieldCodes = getPageFields(appId, pageCode);
        if (hasMaskingExemption(appId, userId, fieldCodes)) {
            return;
        }
        
        // 获取字段脱敏配置
        Map<String, MaskingConfig> maskingConfigs = getMaskingConfigs(appId, fieldCodes);
        
        for (Map<String, Object> record : records) {
            for (Map.Entry<String, MaskingConfig> entry : maskingConfigs.entrySet()) {
                String field = entry.getKey();
                MaskingConfig config = entry.getValue();
                
                if (record.containsKey(field)) {
                    record.put(field, maskValue(record.get(field), config));
                }
            }
        }
    }
    
    private String applyRegexMask(String value, String pattern) {
        // 正则脱敏实现
        // pattern: "(\d{3})****(\d{4})"
        // value: "13812345678"
        // result: "138****5678"
        if (value.matches("^\\d{11}$")) {
            return value.substring(0, 3) + "****" + value.substring(7);
        }
        return value;
    }
}
```

### 4.2 响应结果处理器

```java
@RestControllerAdvice
public class MaskingResponseAdvice implements ResponseBodyAdvice<Object> {
    
    @Autowired
    private DataMaskingService maskingService;
    
    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        // 判断是否需要脱敏处理
        return returnType.hasMethodAnnotation(MaskingEnabled.class);
    }
    
    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType,
                                  MediaType selectedContentType,
                                  Class<? extends HttpMessageConverter<?>> selectedConverterType,
                                  ServerHttpRequest request, ServerHttpResponse response) {
        
        if (body instanceof Result) {
            Result<?> result = (Result<?>) body;
            if (result.getData() != null) {
                // 执行脱敏处理
                String pageCode = getPageCodeFromRequest(request);
                String appId = TenantContext.getCurrentTenant();
                String userId = SecurityContext.getCurrentUserId();
                
                Object maskedData = maskingService.maskData(result.getData(), appId, pageCode, userId);
                result.setData(maskedData);
            }
        }
        
        return body;
    }
}
```

## 5. API 限流控制

### 5.1 限流服务实现

```java
@Component
public class PermissionRateLimiter {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    /**
     * 检查是否超过限流阈值
     */
    public boolean isAllowed(String appId, String userId, String apiCode, RateLimitConfig config) {
        String key = String.format("ratelimit:%s:%s:%s", appId, userId, apiCode);
        
        Long current = redisTemplate.opsForValue().increment(key);
        
        if (current == 1) {
            // 第一次访问，设置过期时间
            redisTemplate.expire(key, config.getWindow());
        }
        
        return current <= config.getLimit();
    }
    
    /**
     * 获取剩余配额
     */
    public long getRemainingQuota(String appId, String userId, String apiCode, RateLimitConfig config) {
        String key = String.format("ratelimit:%s:%s:%s", appId, userId, apiCode);
        String value = redisTemplate.opsForValue().get(key);
        
        if (value == null) {
            return config.getLimit();
        }
        
        long current = Long.parseLong(value);
        return Math.max(0, config.getLimit() - current);
    }
    
    /**
     * 限流切面
     */
    @Aspect
    @Component
    public class RateLimitAspect {
        
        @Autowired
        private PermissionRateLimiter rateLimiter;
        
        @Around("@annotation(rateLimited)")
        public Object around(ProceedingJoinPoint point, RateLimited rateLimited) throws Throwable {
            String userId = SecurityContext.getCurrentUserId();
            String apiCode = rateLimited.value();
            
            RateLimitConfig config = getRateLimitConfig(apiCode);
            
            if (!rateLimiter.isAllowed(userId, apiCode, config)) {
                throw new RateLimitExceededException("请求过于频繁，请稍后重试");
            }
            
            return point.proceed();
        }
    }
}
```

## 6. 审计日志记录

### 6.1 访问审计

```java
@Component
public class PermissionAuditService {
    
    @Autowired
    private SysAccessAuditRepository auditRepository;
    
    /**
     * 记录权限访问
     */
    public void logAccess(AccessAuditEvent event) {
        SysAccessAudit audit = new SysAccessAudit();
        audit.setAppId(event.getAppId());
        audit.setUserId(event.getUserId());
        audit.setPermissionCode(event.getPermissionCode());
        audit.setResource(event.getResource());
        audit.setAction(event.getAction());
        audit.setResult(event.isAllowed() ? 1 : 0);
        audit.setReason(event.getReason());
        audit.setIpAddress(event.getIpAddress());
        audit.setRequestParams(event.getRequestParams());
        audit.setResponseTimeMs(event.getResponseTime());
        audit.setCreatedAt(LocalDateTime.now());
        
        auditRepository.save(audit);
    }
    
    /**
     * 记录权限变更
     */
    public void logPermissionChange(PermissionChangeEvent event) {
        SysPermissionAudit audit = new SysPermissionAudit();
        audit.setAppId(event.getAppId());
        audit.setUserId(event.getOperatorId());
        audit.setAction(event.getAction());
        audit.setTargetType(event.getTargetType());
        audit.setTargetId(event.getTargetId());
        audit.setOldValue(event.getOldValue());
        audit.setNewValue(event.getNewValue());
        audit.setIpAddress(event.getIpAddress());
        audit.setCreatedAt(LocalDateTime.now());
        
        // 异步保存到ES
        elasticsearchTemplate.save(audit);
    }
}
```

### 6.2 审计切面

```java
@Aspect
@Component
public class PermissionAuditAspect {
    
    @Autowired
    private PermissionAuditService auditService;
    
    @Around("@annotation(audited)")
    public Object around(ProceedingJoinPoint point, Audited audited) throws Throwable {
        long startTime = System.currentTimeMillis();
        
        try {
            Object result = point.proceed();
            
            // 记录成功访问
            recordAccess(point, audited, true, null, System.currentTimeMillis() - startTime);
            
            return result;
        } catch (PermissionDeniedException e) {
            // 记录拒绝访问
            recordAccess(point, audited, false, e.getMessage(), System.currentTimeMillis() - startTime);
            throw e;
        }
    }
    
    private void recordAccess(ProceedingJoinPoint point, Audited audited, 
                              boolean allowed, String reason, long responseTime) {
        AccessAuditEvent event = AccessAuditEvent.builder()
            .appId(TenantContext.getCurrentTenant())
            .userId(SecurityContext.getCurrentUserId())
            .permissionCode(audited.value())
            .resource(getResource(point))
            .action(audited.action())
            .allowed(allowed)
            .reason(reason)
            .ipAddress(WebUtils.getClientIp())
            .requestParams(getRequestParams(point))
            .responseTime(responseTime)
            .build();
        
        auditService.logAccess(event);
    }
}
```
