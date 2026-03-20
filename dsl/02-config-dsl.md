# 配置期 DSL 示例

## 1. 基础配置模板

```yaml
# permission-config.yaml
permission_schema:
  version: "1.0"
  app_id: "app_demo_20240320"
  
  # ========== 功能权限定义 ==========
  functional_permissions:
    
    # --------------------
    # 页面级权限
    # --------------------
    pages:
      # 仪表盘
      - code: "page_dashboard"
        name: "仪表盘"
        type: "page"
        path: "/dashboard"
        icon: "DashboardOutlined"
        default_visible: true
      
      # 订单管理菜单
      - code: "page_order_manage"
        name: "订单管理"
        type: "menu"
        icon: "ShoppingCartOutlined"
        children:
          - code: "page_order_list"
            name: "订单列表"
            type: "page"
            path: "/orders"
            
          - code: "page_order_detail"
            name: "订单详情"
            type: "page"
            path: "/orders/:id"
            hidden: true
            
          - code: "page_order_create"
            name: "新建订单"
            type: "page"
            path: "/orders/create"
            hidden: true
      
      # 客户管理菜单
      - code: "page_customer_manage"
        name: "客户管理"
        type: "menu"
        icon: "UserOutlined"
        children:
          - code: "page_customer_list"
            name: "客户列表"
            type: "page"
            path: "/customers"
            
          - code: "page_customer_detail"
            name: "客户详情"
            type: "page"
            path: "/customers/:id"
            hidden: true
      
      # 报表中心
      - code: "page_report_center"
        name: "报表中心"
        type: "menu"
        icon: "BarChartOutlined"
        children:
          - code: "page_sales_report"
            name: "销售报表"
            type: "page"
            path: "/reports/sales"
            
          - code: "page_performance_report"
            name: "业绩报表"
            type: "page"
            path: "/reports/performance"
      
      # 系统设置
      - code: "page_system_settings"
        name: "系统设置"
        type: "menu"
        icon: "SettingOutlined"
        children:
          - code: "page_user_management"
            name: "用户管理"
            type: "page"
            path: "/settings/users"
            
          - code: "page_role_management"
            name: "角色管理"
            type: "page"
            path: "/settings/roles"
    
    # --------------------
    # 按钮级权限 - 订单模块
    # --------------------
    buttons:
      # 订单列表页按钮
      - code: "btn_order_create"
        name: "新建订单"
        page_code: "page_order_list"
        location: "toolbar"
        
      - code: "btn_order_export"
        name: "导出订单"
        page_code: "page_order_list"
        location: "toolbar"
        
      - code: "btn_order_batch_delete"
        name: "批量删除"
        page_code: "page_order_list"
        location: "toolbar"
        danger: true
        
      - code: "btn_order_view"
        name: "查看"
        page_code: "page_order_list"
        location: "row"
        
      - code: "btn_order_edit"
        name: "编辑"
        page_code: "page_order_list"
        location: "row"
        
      - code: "btn_order_delete"
        name: "删除"
        page_code: "page_order_list"
        location: "row"
        danger: true
        
      - code: "btn_order_audit"
        name: "审核"
        page_code: "page_order_list"
        location: "row"
      
      # 订单详情页按钮
      - code: "btn_order_save"
        name: "保存"
        page_code: "page_order_detail"
        location: "form"
        
      - code: "btn_order_submit"
        name: "提交"
        page_code: "page_order_detail"
        location: "form"
      
      # 客户列表页按钮
      - code: "btn_customer_create"
        name: "新建客户"
        page_code: "page_customer_list"
        location: "toolbar"
        
      - code: "btn_customer_edit"
        name: "编辑"
        page_code: "page_customer_list"
        location: "row"
        
      - code: "btn_customer_delete"
        name: "删除"
        page_code: "page_customer_list"
        location: "row"
        danger: true
      
      # 报表按钮
      - code: "btn_report_export"
        name: "导出报表"
        page_code: "page_sales_report"
        location: "toolbar"
        
      - code: "btn_report_print"
        name: "打印"
        page_code: "page_sales_report"
        location: "toolbar"
    
    # --------------------
    # 字段级权限 - 敏感字段脱敏
    # --------------------
    fields:
      # 订单金额（部分脱敏）
      - code: "field_order_amount"
        name: "订单金额"
        page_code: "page_order_detail"
        data_masking:
          enable: true
          type: "partial"
          pattern: "***"
      
      # 客户电话（正则脱敏）
      - code: "field_customer_phone"
        name: "联系电话"
        page_code: "page_customer_detail"
        data_masking:
          enable: true
          type: "regex"
          pattern: "(\\d{3})****(\\d{4})"
      
      # 客户邮箱（部分脱敏）
      - code: "field_customer_email"
        name: "邮箱"
        page_code: "page_customer_detail"
        data_masking:
          enable: true
          type: "partial"
      
      # 成本价（完全脱敏）
      - code: "field_cost_price"
        name: "成本价"
        page_code: "page_order_detail"
        data_masking:
          enable: true
          type: "full"
      
      # 利润（部分脱敏）
      - code: "field_profit"
        name: "利润"
        page_code: "page_order_detail"
        data_masking:
          enable: true
          type: "partial"
      
      # 身份证号（正则脱敏）
      - code: "field_id_card"
        name: "身份证号"
        page_code: "page_customer_detail"
        data_masking:
          enable: true
          type: "regex"
          pattern: "(\\d{6})********(\\d{4})"
    
    # --------------------
    # API级权限（自动生成）
    # --------------------
    apis:
      # 订单API
      - code: "api_order_list"
        method: "GET"
        path: "/api/orders"
        auto_generated: true
        
      - code: "api_order_detail"
        method: "GET"
        path: "/api/orders/:id"
        auto_generated: true
        
      - code: "api_order_create"
        method: "POST"
        path: "/api/orders"
        auto_generated: true
        
      - code: "api_order_update"
        method: "PUT"
        path: "/api/orders/:id"
        auto_generated: true
        
      - code: "api_order_delete"
        method: "DELETE"
        path: "/api/orders/:id"
        auto_generated: true
        
      - code: "api_order_export"
        method: "POST"
        path: "/api/orders/export"
        auto_generated: true
        rate_limit:
          limit: 100
          window: "1h"
      
      # 客户API
      - code: "api_customer_list"
        method: "GET"
        path: "/api/customers"
        auto_generated: true
        
      - code: "api_customer_detail"
        method: "GET"
        path: "/api/customers/:id"
        auto_generated: true
        
      - code: "api_customer_create"
        method: "POST"
        path: "/api/customers"
        auto_generated: true
        
      - code: "api_customer_update"
        method: "PUT"
        path: "/api/customers/:id"
        auto_generated: true
      
      # 报表API
      - code: "api_report_sales"
        method: "GET"
        path: "/api/reports/sales"
        auto_generated: true

  # ========== 数据权限定义 ==========
  data_permissions:
    
    # --------------------
    # 数据权限维度
    # --------------------
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
        
      - code: "dim_customer_level"
        name: "客户等级维度"
        type: "custom"
        entity_field: "customer_level"
        source: "enum_customer_level"
    
    # --------------------
    # 数据规则模板
    # --------------------
    rule_templates:
      # 基础规则
      - code: "rule_self_only"
        name: "仅本人数据"
        expression: "created_by == ${currentUserId}"
        description: "只能查看自己创建的数据"
        
      - code: "rule_dept_only"
        name: "本部门数据"
        expression: "dept_id == ${currentDeptId}"
        description: "只能查看本部门的数据"
        
      - code: "rule_dept_and_sub"
        name: "本部门及子部门"
        expression: "dept_id IN ${currentDeptTree}"
        description: "查看本部门及子部门的数据"
        
      - code: "rule_org_only"
        name: "本组织数据"
        expression: "org_id == ${currentOrgId}"
        description: "只能查看本组织的数据"
        
      # 业务规则
      - code: "rule_pending_audit"
        name: "待审核数据"
        expression: "status == 'pending_audit'"
        description: "只能查看待审核状态的数据"
        
      - code: "rule_region_north"
        name: "华北区数据"
        expression: "region_code IN ('BJ', 'TJ', 'HEB', 'SX')"
        description: "华北区域数据"
        
      - code: "rule_vip_customer"
        name: "VIP客户数据"
        expression: "customer_level == 'VIP'"
        description: "VIP客户相关数据"
        
      # 自定义规则
      - code: "rule_custom"
        name: "自定义规则"
        expression: ""
        description: "由运营管理员自定义"

  # ========== 角色模板定义 ==========
  role_templates:
    
    # --------------------
    # 系统角色
    # --------------------
    - code: "role_admin"
      name: "系统管理员"
      type: "system"
      description: "拥有所有权限，不受任何限制"
      permissions:
        functional: ["*"]
        data:
          strategy: "all"
    
    # --------------------
    # 业务角色模板
    # --------------------
    - code: "role_sales_manager"
      name: "销售经理"
      type: "template"
      description: "管理部门所有销售数据"
      permissions:
        functional:
          - "page_dashboard"
          - "page_order_manage"
          - "page_order_list"
          - "page_order_detail"
          - "page_order_create"
          - "page_customer_manage"
          - "page_customer_list"
          - "page_customer_detail"
          - "page_report_center"
          - "page_sales_report"
          - "btn_order_create"
          - "btn_order_export"
          - "btn_order_edit"
          - "btn_order_delete"
          - "btn_order_audit"
          - "btn_customer_create"
          - "btn_customer_edit"
          - "btn_customer_delete"
          - "btn_report_export"
          - "btn_report_print"
          - "field_order_amount"
          - "field_profit"
        data:
          strategy: "rule_based"
          rules:
            - entity: "order"
              rule: "rule_dept_and_sub"
            - entity: "customer"
              rule: "rule_dept_and_sub"
    
    - code: "role_sales_rep"
      name: "销售代表"
      type: "template"
      description: "仅操作自己的销售数据"
      permissions:
        functional:
          - "page_dashboard"
          - "page_order_manage"
          - "page_order_list"
          - "page_order_detail"
          - "page_order_create"
          - "page_customer_manage"
          - "page_customer_list"
          - "page_customer_detail"
          - "btn_order_create"
          - "btn_order_view"
          - "btn_order_edit"
          - "btn_customer_create"
          - "btn_customer_edit"
        data:
          strategy: "rule_based"
          rules:
            - entity: "order"
              rule: "rule_self_only"
            - entity: "customer"
              rule: "rule_self_only"
    
    - code: "role_finance_auditor"
      name: "财务审核员"
      type: "template"
      description: "审核订单和查看财务报表"
      permissions:
        functional:
          - "page_dashboard"
          - "page_order_manage"
          - "page_order_list"
          - "page_order_detail"
          - "page_report_center"
          - "page_sales_report"
          - "page_performance_report"
          - "btn_order_audit"
          - "btn_report_export"
          - "field_order_amount"
          - "field_cost_price"
          - "field_profit"
        data:
          strategy: "custom"
          custom_rules:
            - entity: "order"
              expression: "status IN ('pending_audit', 'audited')"
    
    - code: "role_report_viewer"
      name: "报表查看员"
      type: "template"
      description: "仅查看报表，无操作权限"
      permissions:
        functional:
          - "page_dashboard"
          - "page_report_center"
          - "page_sales_report"
        data:
          strategy: "rule_based"
          rules:
            - entity: "order"
              rule: "rule_org_only"
    
    - code: "role_customer_service"
      name: "客服专员"
      type: "template"
      description: "处理客户相关事务"
      permissions:
        functional:
          - "page_dashboard"
          - "page_customer_manage"
          - "page_customer_list"
          - "page_customer_detail"
          - "page_order_list"
          - "page_order_detail"
          - "btn_customer_edit"
          - "btn_order_view"
        data:
          strategy: "rule_based"
          rules:
            - entity: "customer"
              rule: "rule_dept_only"
            - entity: "order"
              rule: "rule_dept_only"
```

## 2. 说明

### 2.1 配置原则

1. **权限码命名规范**
   - 页面：`page_{module}_{action}`
   - 按钮：`btn_{module}_{action}`
   - 字段：`field_{module}_{field}`
   - API：`api_{module}_{action}`

2. **继承机制**
   - `system` 类型角色：不可修改，拥有全部权限
   - `template` 类型角色：运行时可继承并自定义

3. **数据脱敏**
   - 敏感字段默认启用脱敏
   - 可通过白名单配置豁免

4. **数据规则**
   - 模板规则支持参数化
   - 支持自定义表达式
