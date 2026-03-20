# 数据表设计

## 1. 功能权限相关表

### 1.1 权限定义表

```sql
CREATE TABLE sys_permission (
    id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '主键ID',
    app_id VARCHAR(64) NOT NULL COMMENT '应用ID',
    code VARCHAR(128) NOT NULL COMMENT '权限编码',
    name VARCHAR(128) NOT NULL COMMENT '权限名称',
    type VARCHAR(32) NOT NULL COMMENT '类型: page|button|field|api',
    parent_code VARCHAR(128) COMMENT '父权限编码',
    path VARCHAR(256) COMMENT '页面路径/API路径',
    icon VARCHAR(64) COMMENT '图标（页面类型）',
    location VARCHAR(32) COMMENT '位置（按钮类型）: toolbar|row|form',
    hidden TINYINT DEFAULT 0 COMMENT '是否隐藏: 0否 1是',
    danger TINYINT DEFAULT 0 COMMENT '是否危险操作: 0否 1是',
    config JSON COMMENT '扩展配置（JSON格式）',
    sort_order INT DEFAULT 0 COMMENT '排序序号',
    status TINYINT DEFAULT 1 COMMENT '状态: 0禁用 1启用',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    created_by VARCHAR(64) COMMENT '创建人',
    updated_by VARCHAR(64) COMMENT '更新人',
    
    UNIQUE KEY uk_app_code (app_id, code),
    INDEX idx_app_type (app_id, type),
    INDEX idx_app_parent (app_id, parent_code),
    INDEX idx_app_status (app_id, status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='权限定义表';
```

### 1.2 角色表

```sql
CREATE TABLE sys_role (
    id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '主键ID',
    app_id VARCHAR(64) NOT NULL COMMENT '应用ID',
    code VARCHAR(128) NOT NULL COMMENT '角色编码',
    name VARCHAR(128) NOT NULL COMMENT '角色名称',
    type VARCHAR(32) NOT NULL DEFAULT 'custom' COMMENT '类型: system|template|custom',
    source VARCHAR(256) COMMENT '来源: template:xxx|custom',
    description TEXT COMMENT '角色描述',
    config JSON COMMENT '权限配置JSON（DSL存储）',
    sort_order INT DEFAULT 0 COMMENT '排序序号',
    status TINYINT DEFAULT 1 COMMENT '状态: 0禁用 1启用',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    created_by VARCHAR(64) COMMENT '创建人',
    updated_by VARCHAR(64) COMMENT '更新人',
    
    UNIQUE KEY uk_app_code (app_id, code),
    INDEX idx_app_type (app_id, type),
    INDEX idx_app_status (app_id, status),
    INDEX idx_source (source)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='角色表';
```

### 1.3 角色-权限关联表

```sql
CREATE TABLE sys_role_permission (
    id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '主键ID',
    role_id BIGINT NOT NULL COMMENT '角色ID',
    permission_code VARCHAR(128) NOT NULL COMMENT '权限编码',
    permission_type VARCHAR(32) COMMENT '权限类型缓存',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    created_by VARCHAR(64) COMMENT '创建人',
    
    UNIQUE KEY uk_role_perm (role_id, permission_code),
    INDEX idx_role (role_id),
    INDEX idx_perm_code (permission_code)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='角色权限关联表';
```

### 1.4 数据权限维度表

```sql
CREATE TABLE sys_data_dimension (
    id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '主键ID',
    app_id VARCHAR(64) NOT NULL COMMENT '应用ID',
    code VARCHAR(64) NOT NULL COMMENT '维度编码',
    name VARCHAR(128) NOT NULL COMMENT '维度名称',
    type VARCHAR(32) NOT NULL COMMENT '类型: organization|department|owner|custom',
    entity_field VARCHAR(128) COMMENT '实体字段名',
    source_type VARCHAR(32) COMMENT '数据源类型: table|enum|api',
    source_config JSON COMMENT '数据源配置（表名、枚举值等）',
    description TEXT COMMENT '维度说明',
    status TINYINT DEFAULT 1 COMMENT '状态: 0禁用 1启用',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    
    UNIQUE KEY uk_app_code (app_id, code),
    INDEX idx_app_type (app_id, type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='数据权限维度表';
```

### 1.5 数据权限规则表

```sql
CREATE TABLE sys_data_rule (
    id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '主键ID',
    app_id VARCHAR(64) NOT NULL COMMENT '应用ID',
    code VARCHAR(128) NOT NULL COMMENT '规则编码',
    name VARCHAR(128) NOT NULL COMMENT '规则名称',
    entity_code VARCHAR(128) NOT NULL COMMENT '实体编码（表名或实体名）',
    rule_type VARCHAR(32) NOT NULL COMMENT '规则类型: template|field_based|custom',
    template_code VARCHAR(128) COMMENT '引用的模板编码（template类型）',
    expression TEXT COMMENT '规则表达式（custom类型）',
    conditions JSON COMMENT '字段条件（field_based类型）',
    priority INT DEFAULT 0 COMMENT '优先级（数字越大优先级越高）',
    parameters JSON COMMENT '规则参数（用于表达式变量替换）',
    description TEXT COMMENT '规则说明',
    status TINYINT DEFAULT 1 COMMENT '状态: 0禁用 1启用',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    created_by VARCHAR(64) COMMENT '创建人',
    updated_by VARCHAR(64) COMMENT '更新人',
    
    UNIQUE KEY uk_app_code (app_id, code),
    INDEX idx_app_entity (app_id, entity_code),
    INDEX idx_rule_type (rule_type),
    INDEX idx_priority (priority)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='数据权限规则表';
```

### 1.6 角色-数据规则关联表

```sql
CREATE TABLE sys_role_data_rule (
    id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '主键ID',
    role_id BIGINT NOT NULL COMMENT '角色ID',
    entity_code VARCHAR(128) NOT NULL COMMENT '实体编码',
    rule_id BIGINT COMMENT '规则ID（null表示ALL策略）',
    strategy VARCHAR(32) NOT NULL DEFAULT 'rule_based' COMMENT '策略: all|rule_based|custom',
    custom_expression TEXT COMMENT '自定义表达式（custom策略）',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    created_by VARCHAR(64) COMMENT '创建人',
    
    UNIQUE KEY uk_role_entity (role_id, entity_code),
    INDEX idx_role (role_id),
    INDEX idx_rule (rule_id),
    INDEX idx_entity (entity_code)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='角色数据规则关联表';
```

## 2. 用户权限相关表

### 2.1 用户-角色关联表

```sql
CREATE TABLE sys_user_role (
    id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '主键ID',
    app_id VARCHAR(64) NOT NULL COMMENT '应用ID',
    user_id VARCHAR(64) NOT NULL COMMENT '用户ID',
    role_id BIGINT NOT NULL COMMENT '角色ID',
    effective_time TIMESTAMP COMMENT '生效时间（null表示立即生效）',
    expire_time TIMESTAMP COMMENT '过期时间（null表示永不过期）',
    source VARCHAR(32) DEFAULT 'manual' COMMENT '来源: manual|dept_default|inherit',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    created_by VARCHAR(64) COMMENT '创建人',
    
    UNIQUE KEY uk_app_user_role (app_id, user_id, role_id),
    INDEX idx_app_user (app_id, user_id),
    INDEX idx_role (role_id),
    INDEX idx_effective (effective_time),
    INDEX idx_expire (expire_time),
    INDEX idx_time_range (effective_time, expire_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户角色关联表';
```

### 2.2 部门-角色关联表

```sql
CREATE TABLE sys_dept_role (
    id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '主键ID',
    app_id VARCHAR(64) NOT NULL COMMENT '应用ID',
    dept_id VARCHAR(64) NOT NULL COMMENT '部门ID',
    role_id BIGINT NOT NULL COMMENT '角色ID',
    is_default TINYINT DEFAULT 0 COMMENT '是否默认角色: 0否 1是',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    created_by VARCHAR(64) COMMENT '创建人',
    
    UNIQUE KEY uk_app_dept_role (app_id, dept_id, role_id),
    INDEX idx_app_dept (app_id, dept_id),
    INDEX idx_role (role_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='部门角色关联表';
```

### 2.3 用户权限缓存表

```sql
CREATE TABLE sys_user_permission_cache (
    id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '主键ID',
    app_id VARCHAR(64) NOT NULL COMMENT '应用ID',
    user_id VARCHAR(64) NOT NULL COMMENT '用户ID',
    permission_codes JSON COMMENT '权限码列表',
    data_rules JSON COMMENT '数据规则配置',
    role_ids JSON COMMENT '角色ID列表',
    version BIGINT DEFAULT 1 COMMENT '缓存版本（用于乐观锁）',
    expire_at TIMESTAMP NOT NULL COMMENT '过期时间',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    
    UNIQUE KEY uk_app_user (app_id, user_id),
    INDEX idx_expire (expire_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户权限缓存表';
```

## 3. 审计相关表

### 3.1 权限操作审计表

```sql
CREATE TABLE sys_permission_audit (
    id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '主键ID',
    app_id VARCHAR(64) NOT NULL COMMENT '应用ID',
    user_id VARCHAR(64) NOT NULL COMMENT '操作人ID',
    username VARCHAR(128) COMMENT '操作人名称',
    action VARCHAR(64) NOT NULL COMMENT '操作类型: CREATE|UPDATE|DELETE|ASSIGN|REVOKE',
    target_type VARCHAR(64) COMMENT '目标类型: permission|role|data_rule|user_role',
    target_id VARCHAR(64) COMMENT '目标ID',
    target_name VARCHAR(256) COMMENT '目标名称',
    old_value JSON COMMENT '变更前值',
    new_value JSON COMMENT '变更后值',
    change_summary TEXT COMMENT '变更摘要',
    ip_address VARCHAR(64) COMMENT '操作IP',
    user_agent TEXT COMMENT 'User-Agent',
    request_id VARCHAR(64) COMMENT '请求ID（链路追踪）',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    
    INDEX idx_app_time (app_id, created_at),
    INDEX idx_user (user_id),
    INDEX idx_action (action),
    INDEX idx_target (target_type, target_id),
    INDEX idx_time (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='权限操作审计日志表';
```

### 3.2 访问审计表

```sql
CREATE TABLE sys_access_audit (
    id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '主键ID',
    app_id VARCHAR(64) NOT NULL COMMENT '应用ID',
    user_id VARCHAR(64) COMMENT '用户ID（未登录可能为空）',
    username VARCHAR(128) COMMENT '用户名',
    permission_code VARCHAR(128) COMMENT '权限码',
    resource VARCHAR(256) COMMENT '资源路径',
    action VARCHAR(64) COMMENT '操作类型: VIEW|CREATE|UPDATE|DELETE|EXPORT',
    result TINYINT COMMENT '结果: 0拒绝 1允许',
    reason VARCHAR(256) COMMENT '拒绝原因',
    ip_address VARCHAR(64) COMMENT '访问IP',
    request_method VARCHAR(16) COMMENT 'HTTP方法',
    request_url VARCHAR(512) COMMENT '请求URL',
    request_params JSON COMMENT '请求参数',
    response_data JSON COMMENT '响应数据（可配置是否记录）',
    data_scope TEXT COMMENT '应用的数据权限范围',
    response_time_ms INT COMMENT '响应时间（毫秒）',
    user_agent TEXT COMMENT 'User-Agent',
    request_id VARCHAR(64) COMMENT '请求ID',
    trace_id VARCHAR(64) COMMENT '链路追踪ID',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    
    INDEX idx_app_time (app_id, created_at),
    INDEX idx_user (user_id),
    INDEX idx_permission (permission_code),
    INDEX idx_result (result),
    INDEX idx_time (created_at),
    INDEX idx_trace (trace_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='访问审计日志表';
```

## 4. 特殊配置表

### 4.1 数据脱敏白名单表

```sql
CREATE TABLE sys_masking_exemption (
    id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '主键ID',
    app_id VARCHAR(64) NOT NULL COMMENT '应用ID',
    user_id VARCHAR(64) NOT NULL COMMENT '用户ID',
    field_codes JSON COMMENT '豁免字段列表（["*"]表示全部）',
    reason TEXT COMMENT '豁免原因',
    effective_time TIMESTAMP COMMENT '生效时间',
    expire_time TIMESTAMP COMMENT '过期时间',
    created_by VARCHAR(64) COMMENT '创建人',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    
    UNIQUE KEY uk_app_user (app_id, user_id),
    INDEX idx_expire (expire_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='数据脱敏白名单表';
```

### 4.2 API限流配置表

```sql
CREATE TABLE sys_rate_limit_config (
    id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '主键ID',
    app_id VARCHAR(64) NOT NULL COMMENT '应用ID',
    target_type VARCHAR(32) NOT NULL COMMENT '目标类型: role|user|api',
    target_id VARCHAR(128) NOT NULL COMMENT '目标ID（角色编码/用户ID/API编码）',
    api_code VARCHAR(128) COMMENT 'API编码（target_type为role/user时必填）',
    limit_count INT NOT NULL COMMENT '限制次数',
    time_window VARCHAR(32) NOT NULL COMMENT '时间窗口: 1s|1m|1h|1d',
    window_seconds INT NOT NULL COMMENT '时间窗口（秒）',
    strategy VARCHAR(32) DEFAULT 'reject' COMMENT '超限策略: reject|throttle|queue',
    status TINYINT DEFAULT 1 COMMENT '状态: 0禁用 1启用',
    description TEXT COMMENT '说明',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    
    UNIQUE KEY uk_app_target_api (app_id, target_type, target_id, api_code),
    INDEX idx_app_api (app_id, api_code)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='API限流配置表';
```

### 4.3 应用权限版本表

```sql
CREATE TABLE sys_permission_version (
    id BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '主键ID',
    app_id VARCHAR(64) NOT NULL COMMENT '应用ID',
    version VARCHAR(32) NOT NULL COMMENT '版本号',
    version_type VARCHAR(32) COMMENT '版本类型: config|runtime',
    dsl_content LONGTEXT COMMENT 'DSL内容（JSON格式）',
    change_log TEXT COMMENT '变更说明',
    created_by VARCHAR(64) COMMENT '创建人',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    
    UNIQUE KEY uk_app_version (app_id, version),
    INDEX idx_app_type (app_id, version_type),
    INDEX idx_time (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='应用权限版本表';
```

## 5. ER 关系图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           权限数据模型关系图                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────┐         ┌──────────────────┐                          │
│  │ sys_permission   │         │ sys_role         │                          │
│  │ 权限定义表        │◄────────┤ 角色表           │                          │
│  │ (功能权限)        │  N:M    │                  │                          │
│  └──────────────────┘         └────────┬─────────┘                          │
│           ▲                            │                                     │
│           │                            │                                     │
│           │                            │ 1:N                                 │
│           │                    ┌───────┴─────────┐                          │
│           │                    │ sys_user_role   │                          │
│           │                    │ 用户角色关联表   │                          │
│           │                    └───────┬─────────┘                          │
│           │                            │                                     │
│           │                            │ N:1                                 │
│           │                            ▼                                     │
│           │                   ┌─────────────────┐                           │
│           │                   │ 用户系统（外部） │                           │
│           │                   └─────────────────┘                           │
│           │                                                                 │
│           │                            ┌──────────────────┐                │
│           └────────────────────────────┤ sys_role_permission│                │
│                                        │ 角色权限关联表     │                │
│                                        └──────────────────┘                │
│                                                                              │
│  ┌──────────────────┐         ┌──────────────────┐                          │
│  │ sys_data_dimension│        │ sys_data_rule    │                          │
│  │ 数据权限维度表    │         │ 数据权限规则表    │                          │
│  └──────────────────┘         └────────┬─────────┘                          │
│                                         │                                    │
│                                         │ 1:N                                │
│                                         ▼                                    │
│                               ┌──────────────────┐                          │
│                               │sys_role_data_rule│                          │
│                               │角色数据规则关联表 │◄───────────────────────┐│
│                               └──────────────────┘                        ││
│                                                                          ││
│  ┌──────────────────┐         ┌──────────────────┐         ┌────────────┐││
│  │ sys_user_permission_cache│  │ sys_permission_audit│    │ sys_access_audit││
│  │ 用户权限缓存表    │         │ 权限操作审计表    │         │ 访问审计表  │││
│  └──────────────────┘         └──────────────────┘         └────────────┘││
│                                                                          ││
│  ┌──────────────────┐         ┌──────────────────┐                       ││
│  │ sys_masking_exemption│      │ sys_rate_limit_config│                   ││
│  │ 脱敏白名单表      │         │ API限流配置表     │                       ││
│  └──────────────────┘         └──────────────────┘                       ││
│                                                                          ││
│  ┌──────────────────┐                                                    ││
│  │ sys_dept_role    │◄───────────────────────────────────────────────────┘│
│  │ 部门角色关联表    │                                                     │
│  └──────────────────┘                                                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```
