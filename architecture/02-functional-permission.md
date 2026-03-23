# 功能权限设计

## 1. 权限类型定义

### 1.1 页面级权限 (Page)

```yaml
# 功能权限配置 DSL（应用级别，每个应用独立一份）
app_id: "app_crm"                    # 应用唯一标识（租户键），所有权限资源按此隔离

pages:
  - code: "page_dashboard"           # 权限码：在同一 app_id 下唯一
    name: "仪表盘"                    # 显示名称
    type: "page"                     # 类型：page | menu
    path: "/dashboard"               # 路由路径
    icon: "DashboardOutlined"        # 图标
    default_visible: true            # 默认可见
    hidden: false                    # 是否在菜单隐藏
    
  - code: "page_order_manage"
    name: "订单管理"
    type: "menu"                     # 菜单类型，可包含子项
    children:
      - code: "page_order_list"
        name: "订单列表"
        type: "page"
        path: "/orders"
        
      - code: "page_order_detail"
        name: "订单详情"
        type: "page"
        path: "/orders/:id"
        hidden: true                 # 详情页不在菜单显示
```

### 1.2 按钮级权限 (Button)

```yaml
buttons:
  - code: "btn_order_create"
    name: "新建订单"
    page_code: "page_order_list"     # 所属页面
    location: "toolbar"              # 位置：toolbar | row | form
    
  - code: "btn_order_edit"
    name: "编辑订单"
    page_code: "page_order_list"
    location: "row"                  # 行操作按钮
    
  - code: "btn_order_delete"
    name: "删除订单"
    page_code: "page_order_list"
    location: "row"
    danger: true                     # 危险操作标记
    
  - code: "btn_order_export"
    name: "导出订单"
    page_code: "page_order_list"
    location: "toolbar"
```

**位置类型说明：**

| 类型 | 说明 | 示例 |
|------|------|------|
| `toolbar` | 工具栏按钮 | 新建、导出、批量操作 |
| `row` | 行操作按钮 | 编辑、删除、查看详情 |
| `form` | 表单按钮 | 保存、提交、重置 |

### 1.3 字段级权限 (Field)

```yaml
fields:
  - code: "field_order_amount"
    name: "订单金额"
    page_code: "page_order_detail"
    data_masking:                    # 数据脱敏配置
      enable: true
      type: "partial"                # partial | full | regex
      pattern: "***"
      masking_side: "backend"        # backend（推荐）| frontend
                                     # backend：API 响应前替换字段值，防止抓包泄露
                                     # frontend：仅视觉屏蔽，不保证数据安全
      
  - code: "field_customer_phone"
    name: "客户电话"
    page_code: "page_order_detail"
    data_masking:
      enable: true
      type: "regex"
      match_pattern: "(\d{3})\d{4}(\d{4})"  # 正则匹配原始值
      replace_with: "$1****$2"               # 替换后的显示格式（手机号脱敏：138****8888）
      masking_side: "backend"
      
  - code: "field_cost_price"
    name: "成本价"
    page_code: "page_order_detail"
    # 通过权限码控制可见性，与其他字段保持一致
    # 运行时仅拥有 field_cost_price 权限码的角色可见
```

**脱敏类型说明：**

| 类型 | 说明 | 示例 |
|------|------|------|
| `partial` | 部分脱敏 | 138****8888 |
| `full` | 完全脱敏 | *********** |
| `regex` | 正则脱敏 | `match_pattern` 匹配原始值，`replace_with` 定义替换结果 |

> **安全建议**：涉及敏感数据（手机号、身份证、金额等）的脱敏字段必须使用 `masking_side: backend`，在 API 响应序列化阶段完成替换，防止原始数据通过网络传输被抓包获取。`frontend` 模式仅适用于低敏感度的视觉提示场景。

### 1.4 API级权限 (API)

```yaml
apis:
  - code: "api_order_list"
    method: "GET"
    path: "/api/orders"
    auto_generated: true             # 自动生成标记
    
  - code: "api_order_create"
    method: "POST"
    path: "/api/orders"
    auto_generated: true
    
  - code: "api_order_export"
    method: "POST"
    path: "/api/orders/export"
    rate_limit:                      # API限流配置
      limit: 100
      window: "1h"
```

## 2. 权限继承与合并

### 2.1 角色权限继承

```yaml
roles:
  - code: "role_sales_manager"
    name: "销售经理"
    source: "template:role_manager"   # 继承模板
    extends:
      functional:
        add:                          # 额外添加的权限
          - "btn_order_delete"
        remove:                       # 移除的权限
          - "btn_order_export"

  - code: "role_finance"
    name: "财务"
    source: "template:role_viewer"
    extends:
      functional:
        add:
          - "field_cost_price"        # 财务角色额外拥有成本价字段权限
```

> **注意**：字段可见性统一通过权限码控制，不使用 `visible_roles` 直接绑定角色名。所有权限分配均在运行时角色配置中完成，保持权限检查逻辑的一致性。

### 2.2 多角色权限合并规则

当用户拥有多个角色时，权限采用**并集**策略：

```java
// 伪代码：权限合并逻辑
Set<String> userPermissions = user.getRoles().stream()
    .flatMap(role -> role.getPermissions().stream())
    .collect(Collectors.toSet());

// 只要任一角色拥有权限，用户即拥有
boolean hasPermission = userPermissions.contains(permissionCode);
```

## 3. 权限校验流程

```
┌─────────────┐
│  用户请求    │
└──────┬──────┘
       ↓
┌─────────────┐     ┌──────────────────────┐
│ 解析权限码   │────→│  查询缓存 (Redis)     │ cache key: perm:{appId}:{userId}
│             │     │  TTL: 5分钟           │
└──────┬──────┘     └──────────┬───────────┘
       │                        │
       │ cache miss              │ cache hit
       ↓                        ↓
┌─────────────┐          ┌─────────────┐
│ singleflight │          │  权限校验   │
│ 防击穿（同   │          │  通过/拒绝  │
│ key只查一次）│          └─────────────┘
└──────┬──────┘
       ↓
┌─────────────┐
│ 查询用户角色 │
│ 合并权限码   │
└──────┬──────┘
       ↓
┌─────────────┐     ┌─────────────┐
│  写入缓存   │────→│  权限校验   │
│  TTL: 5分钟 │     │  通过/拒绝  │
└─────────────┘     └─────────────┘
```

### 3.1 缓存失效策略

权限缓存通过两种机制保持一致性：

**① 主动失效（事件驱动）**

管理员执行以下操作时，发布权限变更事件，异步清除对应缓存：

| 操作 | 清除范围 |
|------|----------|
| 修改角色权限 | 该角色下所有用户的缓存 |
| 用户角色变更 | 该用户的缓存 |
| 发布新版本权限模板 | 应用下所有用户的缓存 |

```java
// 事件驱动清除示例
@EventListener
public void onRolePermissionChanged(RolePermissionChangedEvent event) {
    List<String> userIds = userRoleService.getUsersByRole(event.getRoleCode());
    userIds.forEach(uid -> redisCache.delete("perm:" + uid));
}
```

**② 被动过期（TTL 兜底）**

- 权限缓存 TTL 设为 **5 分钟**，防止主动失效遗漏时长期使用旧缓存
- 对安全要求极高的场景（如金融审批），可将 TTL 缩短至 **1 分钟**

**③ 缓存击穿防护（Singleflight）**

同一用户的权限 cache miss 时，多个并发请求只允许一个查询数据库，其余等待结果共享：

```java
// 使用 Guava Striped 或 Spring Cache 的 sync=true 实现
String permissions = permissionCache.get("perm:" + userId, key -> {
    // 只有一个线程执行，其他等待
    return loadPermissionsFromDB(userId);
});
```

## 4. 前端权限集成

### 4.1 React Hooks

```typescript
// usePermission.ts
export const usePermission = () => {
  const { user } = useAuth();
  
  const hasPermission = (permissionCode: string): boolean => {
    return user.permissions.includes(permissionCode) || 
           user.permissions.includes('*');
  };
  
  const hasAnyPermission = (codes: string[]): boolean => {
    return codes.some(code => hasPermission(code));
  };
  
  const hasAllPermissions = (codes: string[]): boolean => {
    return codes.every(code => hasPermission(code));
  };
  
  return { hasPermission, hasAnyPermission, hasAllPermissions };
};
```

### 4.2 权限组件

```tsx
// Permission.tsx
export const Permission: React.FC<{
  code: string;
  fallback?: React.ReactNode;
  children: React.ReactNode;
}> = ({ code, fallback = null, children }) => {
  const { hasPermission } = usePermission();
  return hasPermission(code) ? <>{children}</> : <>{fallback}</>;
};

// 使用示例
<Permission code="btn_order_create">
  <Button type="primary">新建订单</Button>
</Permission>
```

### 4.3 权限指令（Vue）

```typescript
// v-permission.ts
export const vPermission = {
  mounted(el: HTMLElement, binding: DirectiveBinding) {
    const { hasPermission } = usePermission();
    if (!hasPermission(binding.value)) {
      el.remove();
    }
  }
};

// 使用示例
<button v-permission="'btn_order_delete'">删除</button>
```

## 5. 权限配置界面

### 5.1 页面设计器集成

```typescript
interface PagePermissionConfig {
  // 页面访问权限
  access: {
    enabled: boolean;
    permissionCode: string;
    defaultRole: string[];
  };
  
  // 按钮权限
  buttons: Array<{
    id: string;
    permissionCode: string;
    visibleRoles: string[];
    enabledRoles: string[];
  }>;
  
  // 字段权限
  fields: Array<{
    id: string;
    permissionCode: string;
    visible: boolean;
    editable: boolean;
    masking?: {
      enabled: boolean;
      type: 'partial' | 'full' | 'regex';
      pattern?: string;
    };
  }>;
  
  // 数据权限（与实体关联）
  dataScope: {
    entity: string;
    dimension: string;
    rule: string;
  };
}
```
