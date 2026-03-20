# 缓存策略

## 1. 缓存架构设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                         多级缓存架构                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   ┌──────────────┐      ┌──────────────┐      ┌──────────────┐     │
│   │   L1 缓存    │      │   L2 缓存    │      │   数据库     │     │
│   │  Caffeine    │─────→│   Redis      │─────→│   MySQL      │     │
│   │  (本地缓存)   │      │  (分布式)     │      │  (持久化)     │     │
│   └──────────────┘      └──────────────┘      └──────────────┘     │
│         ↑                      ↑                      ↑             │
│         │                      │                      │             │
│    应用进程内              跨进程共享              最终一致性        │
│    10ms响应               1-5ms响应               10-50ms          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## 2. 缓存层级说明

| 层级 | 技术 | 作用域 | TTL | 适用场景 |
|------|------|--------|-----|----------|
| L1 | Caffeine | 进程内 | 5分钟 | 热点数据、极高频访问 |
| L2 | Redis | 分布式 | 30分钟 | 用户权限、角色权限 |
| DB | MySQL | 持久化 | 永久 | 配置数据、审计日志 |

## 3. 缓存 Key 设计

### 3.1 Key 命名规范

```
格式: {namespace}:{appId}:{resource}:{identifier}

示例:
- permission:user:{appId}:{userId}          # 用户权限
- permission:app:{appId}:metadata           # 应用元数据
- permission:role:{roleId}                  # 角色权限
- permission:dsl:{appId}:{version}          # DSL配置
```

### 3.2 具体 Key 定义

```java
public class CacheKeys {
    
    // ==================== 用户权限缓存 ====================
    
    /** 用户权限缓存 */
    public static String userPermission(String appId, String userId) {
        return String.format("permission:user:%s:%s", appId, userId);
    }
    
    /** 用户权限码集合缓存 */
    public static String userPermissionCodes(String appId, String userId) {
        return String.format("permission:user:%s:%s:codes", appId, userId);
    }
    
    /** 用户数据规则缓存 */
    public static String userDataRules(String appId, String userId) {
        return String.format("permission:user:%s:%s:rules", appId, userId);
    }
    
    /** 用户菜单缓存 */
    public static String userMenus(String appId, String userId) {
        return String.format("permission:user:%s:%s:menus", appId, userId);
    }
    
    // ==================== 应用级缓存 ====================
    
    /** 应用权限元数据 */
    public static String appMetadata(String appId) {
        return String.format("permission:app:%s:metadata", appId);
    }
    
    /** 应用权限定义映射（code -> Permission） */
    public static String appPermissions(String appId) {
        return String.format("permission:app:%s:permissions", appId);
    }
    
    /** 应用数据维度 */
    public static String appDataDimensions(String appId) {
        return String.format("permission:app:%s:dimensions", appId);
    }
    
    /** 应用数据规则模板 */
    public static String appRuleTemplates(String appId) {
        return String.format("permission:app:%s:templates", appId);
    }
    
    /** 应用角色模板 */
    public static String appRoleTemplates(String appId) {
        return String.format("permission:app:%s:role-templates", appId);
    }
    
    // ==================== 角色级缓存 ====================
    
    /** 角色权限缓存 */
    public static String rolePermissions(Long roleId) {
        return String.format("permission:role:%d:permissions", roleId);
    }
    
    /** 角色数据规则缓存 */
    public static String roleDataRules(Long roleId) {
        return String.format("permission:role:%d:rules", roleId);
    }
    
    // ==================== DSL 缓存 ====================
    
    /** DSL配置缓存 */
    public static String dslConfig(String appId, String version) {
        return String.format("permission:dsl:%s:%s", appId, version);
    }
    
    /** DSL 解析结果缓存 */
    public static String dslParsed(String appId, String version) {
        return String.format("permission:dsl:%s:%s:parsed", appId, version);
    }
    
    // ==================== 限流缓存 ====================
    
    /** 用户API限流计数 */
    public static String rateLimit(String userId, String apiCode) {
        return String.format("ratelimit:%s:%s", userId, apiCode);
    }
}
```

## 4. 缓存配置

### 4.1 Caffeine 本地缓存配置

```java
@Configuration
public class LocalCacheConfig {
    
    /**
     * 用户权限本地缓存
     * - 最大容量: 10000
     * - 过期时间: 5分钟
     * - 统计功能开启
     */
    @Bean("userPermissionLocalCache")
    public Cache<String, UserPermission> userPermissionLocalCache() {
        return Caffeine.newBuilder()
            .maximumSize(10000)
            .expireAfterWrite(Duration.ofMinutes(5))
            .recordStats()
            .build();
    }
    
    /**
     * 应用元数据本地缓存
     * - 最大容量: 1000（应用数量不会太多）
     * - 过期时间: 10分钟
     */
    @Bean("appMetadataLocalCache")
    public Cache<String, AppPermissionMetadata> appMetadataLocalCache() {
        return Caffeine.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(Duration.ofMinutes(10))
            .recordStats()
            .build();
    }
    
    /**
     * 角色权限本地缓存
     * - 最大容量: 50000
     * - 过期时间: 10分钟
     */
    @Bean("rolePermissionLocalCache")
    public Cache<String, RolePermission> rolePermissionLocalCache() {
        return Caffeine.newBuilder()
            .maximumSize(50000)
            .expireAfterWrite(Duration.ofMinutes(10))
            .recordStats()
            .build();
    }
}
```

### 4.2 Redis 缓存配置

```java
@Configuration
public class RedisCacheConfig {
    
    @Bean
    public RedisTemplate<String, Object> permissionRedisTemplate(
            RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        
        // Key序列化
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        
        // Value序列化（JSON）
        Jackson2JsonRedisSerializer<Object> serializer = 
            new Jackson2JsonRedisSerializer<>(Object.class);
        ObjectMapper mapper = new ObjectMapper();
        mapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        mapper.activateDefaultTyping(LaissezFaireSubTypeValidator.instance);
        serializer.setObjectMapper(mapper);
        
        template.setValueSerializer(serializer);
        template.setHashValueSerializer(serializer);
        
        template.afterPropertiesSet();
        return template;
    }
    
    @Bean
    public CacheManager permissionCacheManager(
            RedisTemplate<String, Object> redisTemplate) {
        
        // 用户权限缓存配置
        RedisCacheConfiguration userCacheConfig = RedisCacheConfiguration
            .defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .prefixCacheNameWith("permission:")
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()));
        
        // 应用元数据缓存配置（更长TTL）
        RedisCacheConfiguration appCacheConfig = RedisCacheConfiguration
            .defaultCacheConfig()
            .entryTtl(Duration.ofHours(1))
            .prefixCacheNameWith("permission:");
        
        Map<String, RedisCacheConfiguration> configMap = new HashMap<>();
        configMap.put("userPermission", userCacheConfig);
        configMap.put("appMetadata", appCacheConfig);
        
        return RedisCacheManager.builder(redisTemplate.getConnectionFactory())
            .cacheDefaults(userCacheConfig)
            .withInitialCacheConfigurations(configMap)
            .build();
    }
}
```

## 5. 缓存操作封装

### 5.1 多级缓存服务

```java
@Service
public class PermissionCacheService {
    
    @Autowired
    private Cache<String, UserPermission> userPermissionLocalCache;
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    private static final long USER_CACHE_TTL_MINUTES = 30;
    
    // ==================== 用户权限缓存 ====================
    
    /**
     * 获取用户权限（多级缓存）
     */
    public UserPermission getUserPermission(String appId, String userId) {
        String cacheKey = CacheKeys.userPermission(appId, userId);
        
        // L1: 本地缓存
        UserPermission cached = userPermissionLocalCache.getIfPresent(cacheKey);
        if (cached != null) {
            return cached;
        }
        
        // L2: Redis缓存
        String json = redisTemplate.opsForValue().get(cacheKey);
        if (json != null) {
            try {
                UserPermission permission = objectMapper.readValue(json, UserPermission.class);
                // 回填L1缓存
                userPermissionLocalCache.put(cacheKey, permission);
                return permission;
            } catch (JsonProcessingException e) {
                log.error("反序列化用户权限缓存失败", e);
            }
        }
        
        return null;
    }
    
    /**
     * 设置用户权限缓存
     */
    public void setUserPermission(String appId, String userId, UserPermission permission) {
        String cacheKey = CacheKeys.userPermission(appId, userId);
        
        try {
            String json = objectMapper.writeValueAsString(permission);
            
            // L2: Redis
            redisTemplate.opsForValue().set(cacheKey, json, 
                Duration.ofMinutes(USER_CACHE_TTL_MINUTES));
            
            // L1: 本地缓存
            userPermissionLocalCache.put(cacheKey, permission);
            
        } catch (JsonProcessingException e) {
            log.error("序列化用户权限缓存失败", e);
        }
    }
    
    /**
     * 清除用户权限缓存（L1 + L2）
     */
    public void clearUserPermission(String appId, String userId) {
        String cacheKey = CacheKeys.userPermission(appId, userId);
        
        // 清除L1
        userPermissionLocalCache.invalidate(cacheKey);
        
        // 清除L2
        redisTemplate.delete(cacheKey);
        
        // 清除相关缓存
        redisTemplate.delete(CacheKeys.userPermissionCodes(appId, userId));
        redisTemplate.delete(CacheKeys.userDataRules(appId, userId));
        redisTemplate.delete(CacheKeys.userMenus(appId, userId));
    }
    
    // ==================== 应用级缓存 ====================
    
    /**
     * 获取应用权限元数据
     */
    public AppPermissionMetadata getAppMetadata(String appId) {
        String cacheKey = CacheKeys.appMetadata(appId);
        
        // L1
        // ... 类似实现
        
        // L2
        String json = redisTemplate.opsForValue().get(cacheKey);
        if (json != null) {
            try {
                return objectMapper.readValue(json, AppPermissionMetadata.class);
            } catch (JsonProcessingException e) {
                log.error("反序列化应用元数据失败", e);
            }
        }
        
        return null;
    }
    
    /**
     * 刷新应用缓存（清除该应用下所有用户缓存）
     */
    public void refreshAppCache(String appId) {
        // 清除应用元数据缓存
        String appKey = CacheKeys.appMetadata(appId);
        userPermissionLocalCache.invalidate(appKey);
        redisTemplate.delete(appKey);
        
        // 清除该应用下所有用户缓存
        String userPattern = CacheKeys.userPermission(appId, "*");
        Set<String> keys = redisTemplate.keys(userPattern);
        if (keys != null && !keys.isEmpty()) {
            redisTemplate.delete(keys);
            keys.forEach(userPermissionLocalCache::invalidate);
        }
    }
}
```

## 6. 缓存失效策略

### 6.1 主动失效

```java
@Service
public class PermissionCacheInvalidationService {
    
    @Autowired
    private PermissionCacheService cacheService;
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    /**
     * 角色变更时刷新缓存
     */
    @EventListener
    public void onRoleChanged(RoleChangedEvent event) {
        String appId = event.getAppId();
        Long roleId = event.getRoleId();
        
        // 1. 清除角色缓存
        cacheService.clearRoleCache(roleId);
        
        // 2. 清除拥有该角色的所有用户缓存
        List<String> userIds = getUsersByRole(appId, roleId);
        for (String userId : userIds) {
            cacheService.clearUserPermission(appId, userId);
        }
        
        // 3. 发送消息通知其他节点
        notifyOtherNodes(appId, userIds);
    }
    
    /**
     * 权限定义变更时刷新缓存
     */
    @EventListener
    public void onPermissionChanged(PermissionChangedEvent event) {
        String appId = event.getAppId();
        
        // 清除整个应用的缓存（影响范围大）
        cacheService.refreshAppCache(appId);
        
        // 通知其他节点
        notifyOtherNodes(appId, null);
    }
    
    /**
     * 数据规则变更时刷新缓存
     */
    @EventListener
    public void onDataRuleChanged(DataRuleChangedEvent event) {
        String appId = event.getAppId();
        Long ruleId = event.getRuleId();
        
        // 获取使用该规则的角色
        List<Long> roleIds = getRolesByDataRule(appId, ruleId);
        
        // 清除相关用户缓存
        for (Long roleId : roleIds) {
            List<String> userIds = getUsersByRole(appId, roleId);
            for (String userId : userIds) {
                cacheService.clearUserPermission(appId, userId);
            }
        }
    }
    
    /**
     * 用户角色变更时刷新缓存
     */
    @EventListener
    public void onUserRoleChanged(UserRoleChangedEvent event) {
        String appId = event.getAppId();
        String userId = event.getUserId();
        
        cacheService.clearUserPermission(appId, userId);
    }
    
    /**
     * 跨节点缓存同步
     */
    private void notifyOtherNodes(String appId, List<String> userIds) {
        CacheInvalidationMessage message = new CacheInvalidationMessage();
        message.setAppId(appId);
        message.setUserIds(userIds);
        message.setTimestamp(System.currentTimeMillis());
        
        redisTemplate.convertAndSend("permission:cache:invalidate", message);
    }
}

/**
 * 跨节点缓存失效监听器
 */
@Component
public class CacheInvalidationListener implements MessageListener {
    
    @Autowired
    private PermissionCacheService cacheService;
    
    @Override
    public void onMessage(Message message, byte[] pattern) {
        try {
            String json = new String(message.getBody());
            CacheInvalidationMessage invalidateMsg = 
                JSON.parseObject(json, CacheInvalidationMessage.class);
            
            String appId = invalidateMsg.getAppId();
            List<String> userIds = invalidateMsg.getUserIds();
            
            if (userIds == null || userIds.isEmpty()) {
                // 清除整个应用的本地缓存
                cacheService.clearAppLocalCache(appId);
            } else {
                // 清除指定用户的本地缓存
                for (String userId : userIds) {
                    cacheService.clearUserLocalCache(appId, userId);
                }
            }
        } catch (Exception e) {
            log.error("处理缓存失效消息失败", e);
        }
    }
}
```

### 6.2 被动失效（TTL）

```java
// Redis缓存TTL配置
public class CacheTTLConfig {
    
    /** 用户权限缓存：30分钟 */
    public static final long USER_PERMISSION_TTL = 30;
    
    /** 应用元数据缓存：1小时 */
    public static final long APP_METADATA_TTL = 60;
    
    /** 角色权限缓存：1小时 */
    public static final long ROLE_PERMISSION_TTL = 60;
    
    /** DSL配置缓存：24小时（配置变更少） */
    public static final long DSL_CONFIG_TTL = 24 * 60;
    
    /** 限流计数缓存：根据配置动态 */
    public static final long RATE_LIMIT_TTL = 1;
}
```

## 7. 缓存监控

### 7.1 缓存命中率监控

```java
@Component
public class CacheMetricsService {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    @Autowired
    private Cache<String, UserPermission> userPermissionLocalCache;
    
    @Scheduled(fixedRate = 60000) // 每分钟上报
    public void reportCacheMetrics() {
        // Caffeine缓存统计
        CacheStats stats = userPermissionLocalCache.stats();
        
        meterRegistry.gauge("permission.cache.l1.hitRate", 
            stats.hitRate() * 100);
        meterRegistry.gauge("permission.cache.l1.hitCount", 
            stats.hitCount());
        meterRegistry.gauge("permission.cache.l1.missCount", 
            stats.missCount());
        meterRegistry.gauge("permission.cache.l1.evictionCount", 
            stats.evictionCount());
        
        // Redis缓存统计（通过INFO命令）
        // ...
    }
}
```

### 7.2 关键指标

| 指标 | 目标值 | 告警阈值 |
|------|--------|----------|
| L1缓存命中率 | > 90% | < 80% |
| L2缓存命中率 | > 80% | < 70% |
| 缓存平均响应时间 | < 5ms | > 10ms |
| 缓存驱逐率 | < 5% | > 20% |
