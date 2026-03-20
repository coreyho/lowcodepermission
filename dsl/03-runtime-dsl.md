# 运行时 DSL 示例

## 1. 运行时权限配置

```yaml
# runtime-permission-config.yaml
runtime_permission:
  app_id: "app_demo_20240320"
  version: "1.0"
  
  # ========== 角色定义（继承模板并可自定义） ==========
  roles:
    # 继承模板的角色
    - id: "role_001"
      code: "sales_manager_north"
      name: "华北区销售经理"
      source: "template:role_sales_manager"  # 继承销售经理模板
      extends:
        functional:
          add:                          # 额外添加的权限
            - "btn_order_batch_delete"
          remove:                       # 移除的权限
            - "btn_report_print"
        data:
          custom_rules:                 # 自定义数据规则
            - entity: "order"
              expression: "region_code IN ('BJ', 'TJ') AND dept_id IN ${currentDeptTree}"
    
    # 继承模板的角色（无扩展）
    - id: "role_002"  
      code: "sales_rep_north"
      name: "华北区销售代表"
      source: "template:role_sales_rep"
    
    - id: "role_003"
      code: "sales_rep_south"
      name: "华南区销售代表"
      source: "template:role_sales_rep"
    
    # 自定义角色
    - id: "role_004"
      code: "finance_manager"
      name: "财务经理"
      source: "custom"
      permissions:
        functional:
          - "page_dashboard"
          - "page_order_manage"
          - "page_order_list"
          - "page_order_detail"
          - "page_report_center"
          - "page_sales_report"
          - "page_performance_report"
          - "page_system_settings"
          - "page_user_management"
          - "btn_order_audit"
          - "btn_order_export"
          - "btn_report_export"
          - "btn_report_print"
          - "field_order_amount"
          - "field_cost_price"
          - "field_profit"
        data:
          strategy: "all"              # 财务经理可以查看全部数据
    
    - id: "role_005"
      code: "regional_director"
      name: "大区总监"
      source: "custom"
      permissions:
        functional:
          - "page_dashboard"
          - "page_order_manage"
          - "page_customer_manage"
          - "page_report_center"
          - "btn_order_create"
          - "btn_order_edit"
          - "btn_order_delete"
          - "btn_order_export"
          - "btn_customer_create"
          - "btn_customer_edit"
          - "btn_report_export"
          - "field_order_amount"
          - "field_profit"
        data:
          strategy: "custom"
          custom_rules:
            - entity: "order"
              expression: "region_code IN ('BJ', 'TJ', 'HEB', 'SX', 'SH', 'JS', 'ZJ')"
            - entity: "customer"
              expression: "region_code IN ('BJ', 'TJ', 'HEB', 'SX', 'SH', 'JS', 'ZJ')"
    
    - id: "role_006"
      code: "vip_customer_manager"
      name: "VIP客户经理"
      source: "template:role_sales_rep"
      extends:
        functional:
          add:
            - "field_cost_price"
        data:
          strategy: "custom"
          custom_rules:
            - entity: "customer"
              expression: "customer_level == 'VIP' AND created_by == ${currentUserId}"
            - entity: "order"
              expression: "customer_level == 'VIP' AND created_by == ${currentUserId}"
  
  # ========== 用户-角色绑定 ==========
  user_roles:
    # 单个角色
    - user_id: "user_001"
      username: "张三"
      roles: ["role_001"]              # 华北区销售经理
      effective_time: "2024-01-01T00:00:00Z"
      expire_time: null
      
    - user_id: "user_002"
      username: "李四"
      roles: ["role_002"]              # 华北区销售代表
      effective_time: "2024-01-01T00:00:00Z"
      expire_time: "2024-12-31T23:59:59Z"  # 临时授权，年底到期
      
    - user_id: "user_003"
      username: "王五"
      roles: ["role_003"]              # 华南区销售代表
      
    - user_id: "user_004"
      username: "赵六"
      roles: ["role_004"]              # 财务经理
      
    - user_id: "user_005"
      username: "孙七"
      roles: ["role_005"]              # 大区总监
      
    # 多角色（权限取并集）
    - user_id: "user_006"
      username: "周八"
      roles: ["role_002", "role_006"]  # 既是销售代表，又是VIP客户经理
      
    - user_id: "user_007"
      username: "吴九"
      roles: ["role_003", "role_006"]  # 华南区销售代表 + VIP客户经理
      
    # 实习员工（受限权限）
    - user_id: "user_008"
      username: "郑十"
      roles: ["role_002"]              # 销售代表
      effective_time: "2024-03-01T00:00:00Z"
      expire_time: "2024-06-01T00:00:00Z"  # 实习期3个月
  
  # ========== 部门-角色绑定（默认角色） ==========
  dept_roles:
    # 销售一部（华北）
    - dept_id: "dept_sales_north_01"
      default_roles: ["role_002"]       # 默认销售代表
      
    # 销售二部（华北）
    - dept_id: "dept_sales_north_02"
      default_roles: ["role_002"]
      
    # 销售三部（华南）
    - dept_id: "dept_sales_south_01"
      default_roles: ["role_003"]       # 默认华南销售代表
      
    # 财务部
    - dept_id: "dept_finance"
      default_roles: ["role_004"]
      
    # 客服部
    - dept_id: "dept_cs"
      default_roles: ["role_customer_service"]
  
  # ========== 数据权限规则实例 ==========
  data_rules:
    # 基础规则实例
    - id: "rule_inst_001"
      name: "华北一区销售数据"
      entity: "order"
      type: "custom"
      expression: "region_code IN ('BJ', 'TJ') AND dept_id IN ${deptTree001}"
      parameters:
        deptTree001: ["dept_sales_north_01", "dept_sales_north_02"]
      
    - id: "rule_inst_002"
      name: "华南区销售数据"
      entity: "order"
      type: "custom"
      expression: "region_code IN ('GD', 'FJ') AND dept_id IN ${deptTree002}"
      parameters:
        deptTree002: ["dept_sales_south_01", "dept_sales_south_02"]
      
    # 字段条件规则
    - id: "rule_inst_003"
      name: "VIP客户订单"
      entity: "order"
      type: "field_based"
      conditions:
        - field: "customer_level"
          operator: "EQ"
          value: "VIP"
        - field: "amount"
          operator: "GTE"
          value: 100000
      logic: "AND"
      
    - id: "rule_inst_004"
      name: "大额待审核订单"
      entity: "order"
      type: "field_based"
      conditions:
        - field: "status"
          operator: "EQ"
          value: "pending_audit"
        - field: "amount"
          operator: "GTE"
          value: 500000
      logic: "AND"
      
    # 复杂表达式规则
    - id: "rule_inst_005"
      name: "华北区VIP客户数据"
      entity: "customer"
      type: "custom"
      expression: "region_code IN ('BJ', 'TJ', 'HEB', 'SX') AND customer_level == 'VIP' AND created_by IN ${userList}"
      parameters:
        userList: ["user_001", "user_002", "user_006", "user_008"]
      
    - id: "rule_inst_006"
      name: "本季度订单"
      entity: "order"
      type: "custom"
      expression: "created_at >= ${quarterStart} AND created_at <= ${quarterEnd}"
      parameters:
        quarterStart: "2024-01-01T00:00:00Z"
        quarterEnd: "2024-03-31T23:59:59Z"
      
    - id: "rule_inst_007"
      name: "特定产品线订单"
      entity: "order"
      type: "field_based"
      conditions:
        - field: "product_line"
          operator: "IN"
          value: ["line_a", "line_b", "line_c"]

  # ========== 特殊权限配置 ==========
  special_permissions:
    # 数据脱敏白名单
    masking_exemptions:
      # 财务经理可以查看完整金额
      - user_id: "user_004"
        fields: ["field_order_amount", "field_cost_price", "field_profit"]
        
      # 大区总监可以查看完整客户电话
      - user_id: "user_005"
        fields: ["field_customer_phone", "field_customer_email"]
        
      # 系统管理员可以查看所有字段
      - user_id: "user_admin"
        fields: ["*"]
    
    # API访问频率限制
    rate_limits:
      # 普通销售代表导出限制
      - role: "role_002"
        api: "api_order_export"
        limit: 10
        window: "1h"
        
      - role: "role_003"
        api: "api_order_export"
        limit: 10
        window: "1h"
        
      # 销售经理导出限制较宽松
      - role: "role_001"
        api: "api_order_export"
        limit: 50
        window: "1h"
        
      # 报表导出限制
      - role: "role_004"
        api: "api_report_sales"
        limit: 100
        window: "1h"
        
      - role: "role_005"
        api: "api_report_sales"
        limit: 100
        window: "1h"
        
      # 客服查看客户详情限制（防止数据泄露）
      - role: "role_customer_service"
        api: "api_customer_detail"
        limit: 200
        window: "1h"
    
    # 登录限制
    login_restrictions:
      # 实习员工只能在工作时间登录
      - user_id: "user_008"
        allowed_hours:
          - "09:00-18:00"
        allowed_days: ["MON", "TUE", "WED", "THU", "FRI"]
        
    # 数据导出审计
    export_audit:
      enabled: true
      sensitive_apis:
        - "api_order_export"
        - "api_report_sales"
        - "api_customer_export"
      require_reason: true          # 导出需要填写原因
      require_approval: false       # 是否需要审批
```

## 2. 角色权限矩阵

```yaml
# role-permission-matrix.yaml
# 用于可视化展示角色与权限的关系

permission_matrix:
  entities:
    - code: "order"
      name: "订单"
      operations: ["view", "create", "edit", "delete", "export", "audit"]
    - code: "customer"
      name: "客户"
      operations: ["view", "create", "edit", "delete"]
    - code: "report"
      name: "报表"
      operations: ["view", "export", "print"]
  
  matrix:
    - role: "role_001"              # 华北区销售经理
      permissions:
        order:
          view: { scope: "dept_tree", dept_ids: ["dept_sales_north_01", "dept_sales_north_02"] }
          create: { scope: "self" }
          edit: { scope: "dept_tree" }
          delete: { scope: "dept_tree" }
          export: { scope: "dept_tree" }
          audit: false
        customer:
          view: { scope: "dept_tree" }
          create: { scope: "self" }
          edit: { scope: "dept_tree" }
          delete: { scope: "dept_tree" }
        report:
          view: { scope: "org" }
          export: { scope: "org" }
          print: false
      
    - role: "role_002"              # 华北区销售代表
      permissions:
        order:
          view: { scope: "self" }
          create: { scope: "self" }
          edit: { scope: "self" }
          delete: false
          export: { scope: "self", limit: "10/h" }
          audit: false
        customer:
          view: { scope: "self" }
          create: { scope: "self" }
          edit: { scope: "self" }
          delete: false
        report:
          view: { scope: "dept" }
          export: false
          print: false
```

## 3. 权限变更审计

```yaml
# permission-audit-log.yaml
audit_logs:
  - timestamp: "2024-03-20T10:30:00Z"
    operator: "user_admin"
    action: "CREATE"
    target_type: "role"
    target_id: "role_005"
    details:
      name: "大区总监"
      permissions_count: 15
    
  - timestamp: "2024-03-20T10:35:00Z"
    operator: "user_admin"
    action: "ASSIGN"
    target_type: "user_role"
    target_id: "user_005"
    details:
      user: "孙七"
      roles_assigned: ["role_005"]
      previous_roles: ["role_002"]
    
  - timestamp: "2024-03-20T11:00:00Z"
    operator: "user_004"
    action: "MODIFY"
    target_type: "data_rule"
    target_id: "rule_inst_004"
    details:
      field: "amount"
      old_value: "500000"
      new_value: "300000"
      reason: "降低审核门槛"
    
  - timestamp: "2024-03-20T14:20:00Z"
    operator: "user_admin"
    action: "REVOKE"
    target_type: "user_role"
    target_id: "user_008"
    details:
      user: "郑十"
      role_revoked: "role_002"
      reason: "实习结束"
```

## 4. 权限测试用例

```yaml
# permission-test-cases.yaml
test_cases:
  - name: "销售经理查看本部门订单"
    user: "user_001"
    permission: "page_order_list"
    data_scope:
      entity: "order"
      expected_sql: "SELECT * FROM orders WHERE dept_id IN ('dept_sales_north_01', 'dept_sales_north_02')"
    expected: true
    
  - name: "销售代表查看他人订单（应被拒绝）"
    user: "user_002"
    permission: "page_order_detail"
    data_scope:
      entity: "order"
      record:
        id: "order_999"
        created_by: "user_999"        # 非本人创建
      expected_access: false
    
  - name: "销售代表查看自己订单"
    user: "user_002"
    permission: "page_order_detail"
    data_scope:
      entity: "order"
      record:
        id: "order_001"
        created_by: "user_002"        # 本人创建
      expected_access: true
    
  - name: "财务经理查看成本价（脱敏豁免）"
    user: "user_004"
    field: "field_cost_price"
    expected_masking: false           # 不脱敏
    
  - name: "销售代表查看成本价（应脱敏）"
    user: "user_002"
    field: "field_cost_price"
    expected_masking: true            # 脱敏
    expected_masked_value: "****"
    
  - name: "销售代表导出超限"
    user: "user_002"
    api: "api_order_export"
    request_count: 15                 # 超过10次限制
    expected_allowed: false
    expected_error: "RATE_LIMIT_EXCEEDED"
    
  - name: "大区总监跨区域查看数据"
    user: "user_005"
    permission: "page_order_list"
    data_scope:
      entity: "order"
      expected_sql: "SELECT * FROM orders WHERE region_code IN ('BJ', 'TJ', 'HEB', 'SX', 'SH', 'JS', 'ZJ')"
    expected: true
    
  - name: "已过期用户访问"
    user: "user_002"
    test_time: "2025-01-01T00:00:00Z"  # 超过2024-12-31过期时间
    permission: "page_order_list"
    expected: false
    expected_error: "ROLE_EXPIRED"
```
