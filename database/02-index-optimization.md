# 索引优化建议

## 1. 索引设计原则

### 1.1 通用原则

1. **区分度高的字段建索引**：如 `app_id + code` 组合
2. **查询频繁的字段建索引**：如用户ID、角色ID
3. **避免冗余索引**：定期清理未使用的索引
4. **控制索引数量**：单表索引不超过5-6个
5. **使用覆盖索引**：减少回表查询

### 1.2 联合索引最左前缀原则

```sql
-- 正确：查询条件从左边开始匹配
SELECT * FROM sys_permission WHERE app_id = 'xxx' AND code = 'xxx';

-- 索引: (app_id, code)
```

## 2. 各表索引优化详情

### 2.1 sys_permission（权限定义表）

```sql
-- 主键索引（自动创建）
PRIMARY KEY (id)

-- 唯一索引：应用+权限码（核心查询）
UNIQUE KEY uk_app_code (app_id, code)

-- 联合索引：应用+类型（按类型查询权限列表）
INDEX idx_app_type (app_id, type)

-- 联合索引：应用+父权限（查询子权限）
INDEX idx_app_parent (app_id, parent_code)

-- 联合索引：应用+状态（查询启用中的权限）
INDEX idx_app_status (app_id, status)

-- 优化建议：
-- 1. 如果频繁按path查询，考虑添加: INDEX idx_path (path(100))
-- 2. 如果权限码有搜索需求，考虑添加全文索引
```

### 2.2 sys_role（角色表）

```sql
-- 主键索引
PRIMARY KEY (id)

-- 唯一索引：应用+角色编码
UNIQUE KEY uk_app_code (app_id, code)

-- 联合索引：应用+类型
INDEX idx_app_type (app_id, type)

-- 联合索引：应用+状态
INDEX idx_app_status (app_id, status)

-- 普通索引：来源（用于追溯模板继承）
INDEX idx_source (source)

-- 优化建议：
-- 1. 角色名称搜索频繁时，添加: INDEX idx_name (name)
-- 2. 如果按创建时间排序查询，添加: INDEX idx_created (app_id, created_at)
```

### 2.3 sys_role_permission（角色权限关联表）

```sql
-- 主键索引
PRIMARY KEY (id)

-- 唯一索引：角色+权限（避免重复绑定）
UNIQUE KEY uk_role_perm (role_id, permission_code)

-- 联合索引：角色ID（查询角色拥有的所有权限）
INDEX idx_role (role_id)

-- 普通索引：权限码（查询拥有某权限的所有角色）
INDEX idx_perm_code (permission_code)

-- 优化建议：
-- 1. 这是一个高频查询表，考虑使用覆盖索引
-- 2. 如果数据量巨大，考虑按app_id分区
```

### 2.4 sys_data_rule（数据权限规则表）

```sql
-- 主键索引
PRIMARY KEY (id)

-- 唯一索引：应用+规则编码
UNIQUE KEY uk_app_code (app_id, code)

-- 联合索引：应用+实体（查询某实体的所有规则）
INDEX idx_app_entity (app_id, entity_code)

-- 普通索引：规则类型（按类型统计）
INDEX idx_rule_type (rule_type)

-- 普通索引：优先级（排序）
INDEX idx_priority (priority)

-- 优化建议：
-- 1. expression字段较长，不做索引
-- 2. 如果频繁按template_code查询，添加: INDEX idx_template (template_code)
```

### 2.5 sys_role_data_rule（角色数据规则关联表）

```sql
-- 主键索引
PRIMARY KEY (id)

-- 唯一索引：角色+实体（一个角色对一个实体只有一个规则）
UNIQUE KEY uk_role_entity (role_id, entity_code)

-- 联合索引：角色ID
INDEX idx_role (role_id)

-- 普通索引：规则ID
INDEX idx_rule (rule_id)

-- 普通索引：实体编码（反向查询）
INDEX idx_entity (entity_code)

-- 优化建议：
-- 1. 核心查询：SELECT * FROM sys_role_data_rule WHERE role_id = ?
-- 2. 考虑创建覆盖索引: INDEX idx_role_cover (role_id, entity_code, rule_id, strategy)
```

### 2.6 sys_user_role（用户角色关联表）

```sql
-- 主键索引
PRIMARY KEY (id)

-- 唯一索引：应用+用户+角色（避免重复绑定）
UNIQUE KEY uk_app_user_role (app_id, user_id, role_id)

-- 联合索引：应用+用户（查询用户的所有角色）
INDEX idx_app_user (app_id, user_id)

-- 普通索引：角色ID（反向查询）
INDEX idx_role (role_id)

-- 普通索引：生效时间（定时任务清理过期数据）
INDEX idx_effective (effective_time)

-- 普通索引：过期时间（查询即将过期的角色）
INDEX idx_expire (expire_time)

-- 联合索引：时间范围（有效期查询）
INDEX idx_time_range (effective_time, expire_time)

-- 优化建议：
-- 1. 这是权限查询的核心表，必须优化
-- 2. 查询用户角色时的SQL: 
--    SELECT role_id FROM sys_user_role 
--    WHERE app_id = ? AND user_id = ? 
--    AND (effective_time IS NULL OR effective_time <= NOW())
--    AND (expire_time IS NULL OR expire_time >= NOW())
-- 3. 考虑创建覆盖索引覆盖上述查询
```

### 2.7 sys_user_permission_cache（用户权限缓存表）

```sql
-- 主键索引
PRIMARY KEY (id)

-- 唯一索引：应用+用户（一个用户一个缓存）
UNIQUE KEY uk_app_user (app_id, user_id)

-- 普通索引：过期时间（定时清理过期缓存）
INDEX idx_expire (expire_at)

-- 优化建议：
-- 1. 数据可以设置TTL，过期自动删除
-- 2. 考虑使用Redis代替MySQL存储缓存
-- 3. 如果保留MySQL，可以按expire_at分区
```

### 2.8 sys_access_audit（访问审计表）

```sql
-- 主键索引
PRIMARY KEY (id)

-- 联合索引：应用+时间（最常用的查询）
INDEX idx_app_time (app_id, created_at)

-- 普通索引：用户ID（查询用户访问记录）
INDEX idx_user (user_id)

-- 普通索引：权限码（统计权限使用）
INDEX idx_permission (permission_code)

-- 普通索引：结果（统计拒绝率）
INDEX idx_result (result)

-- 普通索引：创建时间（按时间范围查询）
INDEX idx_time (created_at)

-- 普通索引：链路追踪ID
INDEX idx_trace (trace_id)

-- 优化建议：
-- 1. 这是一个写入密集型表，索引不宜过多
-- 2. 数据量巨大，必须按时间分区或归档
-- 3. 考虑使用Elasticsearch存储审计日志
-- 4. 定期清理历史数据（保留90天）
```

### 2.9 sys_permission_audit（权限操作审计表）

```sql
-- 主键索引
PRIMARY KEY (id)

-- 联合索引：应用+时间
INDEX idx_app_time (app_id, created_at)

-- 普通索引：操作人
INDEX idx_user (user_id)

-- 普通索引：操作类型（按类型查询）
INDEX idx_action (action)

-- 联合索引：目标类型+目标ID（追踪变更历史）
INDEX idx_target (target_type, target_id)

-- 普通索引：创建时间
INDEX idx_time (created_at)

-- 优化建议：
-- 1. 同访问审计表，建议用ES存储
-- 2. 保留时间可以更长（1年）
-- 3. 考虑归档策略
```

## 3. 慢查询优化建议

### 3.1 权限校验查询

**慢查询场景：**
```sql
-- 查询用户所有权限（需要JOIN多张表）
SELECT p.code 
FROM sys_user_role ur
JOIN sys_role_permission rp ON ur.role_id = rp.role_id
JOIN sys_permission p ON rp.permission_code = p.code
WHERE ur.app_id = 'xxx' 
AND ur.user_id = 'xxx'
AND ur.status = 1;
```

**优化方案：**
1. 使用缓存表 `sys_user_permission_cache` 避免实时JOIN
2. 缓存失效策略：角色变更时刷新缓存
3. Redis缓存：权限码列表 + 数据规则

### 3.2 数据权限查询

**慢查询场景：**
```sql
-- 查询用户的数据权限规则
SELECT r.expression, r.entity_code
FROM sys_user_role ur
JOIN sys_role_data_rule rdr ON ur.role_id = rdr.role_id
JOIN sys_data_rule r ON rdr.rule_id = r.id
WHERE ur.app_id = 'xxx'
AND ur.user_id = 'xxx';
```

**优化方案：**
1. 同样使用缓存表缓存数据规则
2. 规则表达式预编译缓存
3. SQL拦截器从ThreadLocal获取规则，避免重复查询

## 4. 分区建议

### 4.1 按时间分区（审计表）

```sql
-- sys_access_audit 按月分区
CREATE TABLE sys_access_audit (
    -- ... 字段定义
) ENGINE=InnoDB 
PARTITION BY RANGE (YEAR(created_at) * 100 + MONTH(created_at)) (
    PARTITION p202401 VALUES LESS THAN (202402),
    PARTITION p202402 VALUES LESS THAN (202403),
    PARTITION p202403 VALUES LESS THAN (202404),
    -- ...
    PARTITION p_max VALUES LESS THAN MAXVALUE
);
```

### 4.2 按应用ID分区（权限配置表）

```sql
-- 如果应用数量多，可以考虑HASH分区
CREATE TABLE sys_permission (
    -- ... 字段定义
) ENGINE=InnoDB
PARTITION BY HASH(CRC32(app_id)) PARTITIONS 16;
```

## 5. 数据库参数建议

### 5.1 InnoDB 参数

```ini
# my.cnf

# 缓冲池大小（内存的50-70%）
innodb_buffer_pool_size = 4G

# 日志文件大小
innodb_log_file_size = 512M
innodb_log_files_in_group = 3

# 刷新日志策略（权衡性能与持久性）
innodb_flush_log_at_trx_commit = 2

# 并发线程数
innodb_read_io_threads = 8
innodb_write_io_threads = 8

# 打开表缓存
table_open_cache = 2000

# 连接数
max_connections = 500
```

### 5.2 查询缓存（MySQL 8.0已移除，使用Redis替代）

```ini
# 如果使用MySQL 5.7
query_cache_type = 1
query_cache_size = 256M
```

## 6. 监控指标

### 6.1 需要监控的SQL指标

| 指标 | 说明 | 告警阈值 |
|------|------|----------|
| 慢查询数量 | 执行时间>1s的查询 | > 10/min |
| 权限缓存命中率 | Redis缓存命中 | < 90% |
| 数据库连接数 | 当前连接数 | > 400 |
| 审计表写入QPS | 审计日志写入 | > 5000/s |

### 6.2 慢查询分析

```sql
-- 查看慢查询
SELECT * FROM mysql.slow_log 
WHERE start_time > DATE_SUB(NOW(), INTERVAL 1 HOUR)
ORDER BY query_time DESC
LIMIT 10;

-- 查看索引使用情况
SHOW STATUS LIKE 'Handler_read%';
```
