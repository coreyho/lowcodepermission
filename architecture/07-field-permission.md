# 列权限（字段级权限）功能设计

## 1. 概念定义

列权限（Column-Level Permission）控制用户对数据表中特定字段的访问能力，包括：
- **可见性（Visible）**：是否能看到该字段的值
- **可编辑性（Editable）**：是否能修改该字段的值
- **数据脱敏（Masking）**：字段值是否需要脱敏显示

```
┌─────────────────────────────────────────────────────────────┐
│                     订单表 (order)                          │
├─────────────┬─────────────┬─────────────┬─────────────────┤
│   id        │ customer_name│   amount    │   cost_price    │
├─────────────┼─────────────┼─────────────┼─────────────────┤
│  销售角色可见 │  销售角色可见  │  销售角色可见 │   财务角色可见   │
│  只读       │   可编辑      │   可编辑     │    只读         │
│             │             │  敏感：脱敏   │   敏感：脱敏     │
└─────────────┴─────────────┴─────────────┴─────────────────┘
```

## 2. 权限模型

### 2.1 权限元数据

```yaml
# 列权限配置（应用配置期生成）
app_id: "app_crm"

field_permissions:
  # 实体级别的字段定义
  - entity: "order"
    fields:
      - code: "field_order_id"
        name: "订单编号"
        column: "id"                    # 对应数据库列名
        data_type: "string"
        required: true                  # 是否必填
        system_field: true              # 系统字段（不可删除）

      - code: "field_customer_name"
        name: "客户名称"
        column: "customer_name"
        data_type: "string"
        required: true

      - code: "field_amount"
        name: "订单金额"
        column: "amount"
        data_type: "decimal"
        sensitive: true                 # 标记为敏感字段
        default_masking: "partial"      # 默认脱敏策略

      - code: "field_cost_price"
        name: "成本价"
        column: "cost_price"
        data_type: "decimal"
        sensitive: true
        visible_roles: ["role_finance", "role_admin"]  # 默认可见角色

      - code: "field_created_by"
        name: "创建人"
        column: "created_by"
        data_type: "string"
        system_field: true
        editable: false                 # 系统字段不可编辑
```

### 2.2 运行时权限配置

```yaml
# 角色字段权限配置（应用运行期）
roles:
  - code: "role_sales_rep"
    name: "销售代表"
    field_permissions:
      # 实体级别的字段权限配置
      - entity: "order"
        fields:
          - field_code: "field_order_id"
            visible: true
            editable: false             # 订单编号不可编辑

          - field_code: "field_customer_name"
            visible: true
            editable: true

          - field_code: "field_amount"
            visible: true
            editable: true
            masking:                    # 字段级别的脱敏配置
              enabled: true
              type: "partial"           # 部分脱敏
              pattern: "***"

          - field_code: "field_cost_price"
            visible: false              # 销售代表看不到成本价

  - code: "role_finance"
    name: "财务"
    field_permissions:
      - entity: "order"
        fields:
          - field_code: "field_order_id"
            visible: true
            editable: false

          - field_code: "field_amount"
            visible: true
            editable: false             # 财务可看金额但不可改
            masking:
              enabled: false            # 财务看金额不脱敏

          - field_code: "field_cost_price"
            visible: true
            editable: false
```

## 3. 权限计算规则

### 3.1 权限维度

列权限包含三个独立维度：

| 维度 | 说明 | 默认值 | 优先级 |
|------|------|--------|--------|
| `visible` | 是否可见 | `true` | 最高 |
| `editable` | 是否可编辑 | `true` | 中 |
| `masking` | 是否脱敏 | `false` | 低 |

**规则：**
1. `visible = false` 时，字段完全不显示（API 不返回）
2. `visible = true, editable = false` 时，字段可见但只读
3. `visible = true, editable = true` 时，字段可见且可编辑
4. `masking.enabled = true` 时，字段值脱敏显示（不影响编辑权限）

### 3.2 多角色权限合并

当用户拥有多个角色时，列权限采用以下合并策略：

```java
/**
 * 列权限合并规则
 */
public FieldPermission mergeFieldPermissions(List<FieldPermission> permissions) {
    // 1. 可见性：任一角色可见则可见
    boolean visible = permissions.stream()
        .anyMatch(p -> p.isVisible());

    // 2. 可编辑性：所有角色都可编辑才可编辑
    // 解释：只要有一个角色设置为不可编辑，则该字段对此用户只读
    boolean editable = visible && permissions.stream()
        .allMatch(p -> p.isEditable());

    // 3. 脱敏：任一角色需要脱敏则脱敏
    // 解释：取最严格的脱敏策略
    MaskingConfig masking = permissions.stream()
        .filter(p -> p.getMasking() != null && p.getMasking().isEnabled())
        .map(FieldPermission::getMasking)
        .max(Comparator.comparing(MaskingConfig::getLevel))  // 按脱敏级别排序
        .orElse(MaskingConfig.none());

    return FieldPermission.builder()
        .visible(visible)
        .editable(editable)
        .masking(masking)
        .build();
}
```

**合并示例：**

```
用户张三拥有两个角色：
- 角色A：visible=true, editable=true, masking=NONE
- 角色B：visible=true, editable=false, masking=PARTIAL

合并结果：
- visible = true (任一可见)
- editable = false (所有可编辑才行)
- masking = PARTIAL (取最严格)
```

## 4. 后端实现

### 4.1 数据模型

```sql
-- 字段元数据表（配置期）
CREATE TABLE meta_field (
    id              VARCHAR(32) PRIMARY KEY,
    app_id          VARCHAR(32) NOT NULL COMMENT '应用ID',
    entity_code     VARCHAR(64) NOT NULL COMMENT '实体编码',
    field_code      VARCHAR(64) NOT NULL COMMENT '字段编码',
    field_name      VARCHAR(128) NOT NULL COMMENT '字段名称',
    column_name     VARCHAR(64) NOT NULL COMMENT '数据库列名',
    data_type       VARCHAR(32) COMMENT '数据类型',
    required        BOOLEAN DEFAULT FALSE COMMENT '是否必填',
    system_field    BOOLEAN DEFAULT FALSE COMMENT '是否系统字段',
    sensitive       BOOLEAN DEFAULT FALSE COMMENT '是否敏感字段',
    default_masking VARCHAR(32) COMMENT '默认脱敏策略',
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_entity_field (app_id, entity_code, field_code)
);

-- 角色字段权限表（运行期）
CREATE TABLE role_field_permission (
    id              VARCHAR(32) PRIMARY KEY,
    app_id          VARCHAR(32) NOT NULL COMMENT '应用ID',
    role_id         VARCHAR(32) NOT NULL COMMENT '角色ID',
    entity_code     VARCHAR(64) NOT NULL COMMENT '实体编码',
    field_code      VARCHAR(64) NOT NULL COMMENT '字段编码',
    visible         BOOLEAN DEFAULT TRUE COMMENT '是否可见',
    editable        BOOLEAN DEFAULT TRUE COMMENT '是否可编辑',
    masking_enabled BOOLEAN DEFAULT FALSE COMMENT '是否启用脱敏',
    masking_type    VARCHAR(32) COMMENT '脱敏类型',
    masking_config  JSON COMMENT '脱敏配置',
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_role_entity_field (app_id, role_id, entity_code, field_code)
);

-- 用户字段权限缓存表（运行时计算结果）
CREATE TABLE user_field_permission_cache (
    id              VARCHAR(32) PRIMARY KEY,
    app_id          VARCHAR(32) NOT NULL COMMENT '应用ID',
    user_id         VARCHAR(32) NOT NULL COMMENT '用户ID',
    entity_code     VARCHAR(64) NOT NULL COMMENT '实体编码',
    field_code      VARCHAR(64) NOT NULL COMMENT '字段编码',
    visible         BOOLEAN DEFAULT TRUE COMMENT '是否可见',
    editable        BOOLEAN DEFAULT TRUE COMMENT '是否可编辑',
    masking_enabled BOOLEAN DEFAULT FALSE COMMENT '是否启用脱敏',
    computed_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '计算时间',
    UNIQUE KEY uk_user_entity_field (app_id, user_id, entity_code, field_code)
);
```

### 4.2 权限计算服务

```java
/**
 * 列权限计算服务
 */
@Service
public class FieldPermissionService {

    @Autowired
    private RoleFieldPermissionRepository roleFieldPermissionRepo;

    @Autowired
    private UserRoleService userRoleService;

    @Autowired
    private CacheManager cacheManager;

    private static final String CACHE_KEY_PREFIX = "field_perm:";

    /**
     * 获取用户对指定实体的字段权限
     */
    public Map<String, FieldPermission> getFieldPermissions(
            String appId, String userId, String entityCode) {

        String cacheKey = CACHE_KEY_PREFIX + appId + ":" + userId + ":" + entityCode;

        // 1. 尝试从缓存获取
        Map<String, FieldPermission> cached = cacheManager.get(cacheKey);
        if (cached != null) {
            return cached;
        }

        // 2. 获取用户所有角色
        List<Role> roles = userRoleService.getUserRoles(appId, userId);

        // 3. 收集所有角色的字段权限并按字段分组
        Map<String, List<FieldPermission>> permissionGroups = new HashMap<>();

        for (Role role : roles) {
            List<RoleFieldPermission> rolePermissions = roleFieldPermissionRepo
                .findByAppIdAndRoleIdAndEntityCode(appId, role.getId(), entityCode);

            for (RoleFieldPermission rp : rolePermissions) {
                permissionGroups
                    .computeIfAbsent(rp.getFieldCode(), k -> new ArrayList<>())
                    .add(convertToFieldPermission(rp));
            }
        }

        // 4. 合并权限
        Map<String, FieldPermission> result = new HashMap<>();
        for (Map.Entry<String, List<FieldPermission>> entry : permissionGroups.entrySet()) {
            FieldPermission merged = mergeFieldPermissions(entry.getValue());
            result.put(entry.getKey(), merged);
        }

        // 5. 写入缓存
        cacheManager.set(cacheKey, result, Duration.ofMinutes(5));

        return result;
    }

    /**
     * 检查字段是否可见
     */
    public boolean isFieldVisible(String appId, String userId,
                                   String entityCode, String fieldCode) {
        Map<String, FieldPermission> permissions = getFieldPermissions(appId, userId, entityCode);
        FieldPermission permission = permissions.get(fieldCode);
        return permission != null && permission.isVisible();
    }

    /**
     * 检查字段是否可编辑
     */
    public boolean isFieldEditable(String appId, String userId,
                                    String entityCode, String fieldCode) {
        Map<String, FieldPermission> permissions = getFieldPermissions(appId, userId, entityCode);
        FieldPermission permission = permissions.get(fieldCode);
        return permission != null && permission.isVisible() && permission.isEditable();
    }

    /**
     * 合并字段权限
     */
    private FieldPermission mergeFieldPermissions(List<FieldPermission> permissions) {
        if (permissions.isEmpty()) {
            return FieldPermission.defaultPermission();
        }

        // 可见性：任一可见则可见
        boolean visible = permissions.stream().anyMatch(FieldPermission::isVisible);

        // 可编辑性：所有可编辑才可编辑
        boolean editable = permissions.stream().allMatch(FieldPermission::isEditable);

        // 脱敏：取最严格的
        MaskingConfig masking = permissions.stream()
            .map(FieldPermission::getMasking)
            .filter(Objects::nonNull)
            .filter(MaskingConfig::isEnabled)
            .max(Comparator.comparingInt(MaskingConfig::getLevel))
            .orElse(MaskingConfig.none());

        return FieldPermission.builder()
            .visible(visible)
            .editable(editable && visible)  // 不可见则一定不可编辑
            .masking(masking)
            .build();
    }
}
```

### 4.3 API 响应过滤

```java
/**
 * 字段权限响应过滤器
 * 自动根据用户权限过滤返回的字段
 */
@RestControllerAdvice
public class FieldPermissionResponseAdvice implements ResponseBodyAdvice<Object> {

    @Autowired
    private FieldPermissionService fieldPermissionService;

    @Override
    public boolean supports(MethodParameter returnType,
                           Class<? extends HttpMessageConverter<?>> converterType) {
        // 只对标记了 @EntityResponse 的方法生效
        return returnType.hasMethodAnnotation(EntityResponse.class);
    }

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

        // 获取实体注解信息
        EntityResponse entityAnnotation = returnType.getMethodAnnotation(EntityResponse.class);
        String entityCode = entityAnnotation.value();

        String appId = TenantContext.getCurrentTenant();
        String userId = SecurityContext.getCurrentUserId();

        // 获取字段权限
        Map<String, FieldPermission> fieldPermissions = fieldPermissionService
            .getFieldPermissions(appId, userId, entityCode);

        // 过滤数据
        Object filteredData = filterByFieldPermission(result.getData(), fieldPermissions);
        result.setData(filteredData);

        return body;
    }

    /**
     * 根据字段权限过滤数据
     */
    private Object filterByFieldPermission(Object data,
                                           Map<String, FieldPermission> permissions) {
        if (data instanceof List) {
            return ((List<?>) data).stream()
                .map(item -> filterSingleObject(item, permissions))
                .collect(Collectors.toList());
        } else {
            return filterSingleObject(data, permissions);
        }
    }

    private Map<String, Object> filterSingleObject(Object data,
                                                    Map<String, FieldPermission> permissions) {
        if (!(data instanceof Map)) {
            return convertToMap(data, permissions);
        }

        Map<String, Object> original = (Map<String, Object>) data;
        Map<String, Object> filtered = new HashMap<>();

        for (Map.Entry<String, Object> entry : original.entrySet()) {
            String fieldCode = entry.getKey();
            FieldPermission permission = permissions.get(fieldCode);

            // 1. 检查可见性
            if (permission == null || !permission.isVisible()) {
                continue;  // 不可见，跳过该字段
            }

            Object value = entry.getValue();

            // 2. 检查是否需要脱敏
            if (permission.getMasking() != null && permission.getMasking().isEnabled()) {
                value = applyMasking(value, permission.getMasking());
            }

            filtered.put(fieldCode, value);
        }

        return filtered;
    }

    /**
     * 应用脱敏
     */
    private Object applyMasking(Object value, MaskingConfig masking) {
        if (value == null) {
            return null;
        }

        String strValue = value.toString();

        switch (masking.getType()) {
            case FULL:
                return StringUtils.repeat("*", strValue.length());
            case PARTIAL:
                int len = strValue.length();
                if (len <= 4) return "****";
                return strValue.substring(0, 2) +
                       StringUtils.repeat("*", len - 4) +
                       strValue.substring(len - 2);
            case REGEX:
                if (StringUtils.isNotBlank(masking.getPattern())) {
                    return strValue.replaceAll(masking.getPattern(), masking.getReplacement());
                }
                return "****";
            default:
                return value;
        }
    }
}

/**
 * 实体响应注解
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface EntityResponse {
    String value();  // 实体编码
}
```

### 4.4 编辑权限校验

```java
/**
 * 字段编辑权限校验切面
 */
@Aspect
@Component
public class FieldEditPermissionAspect {

    @Autowired
    private FieldPermissionService fieldPermissionService;

    /**
     * 拦截带有 @CheckFieldEditPermission 注解的方法
     */
    @Around("@annotation(checkPermission)")
    public Object around(ProceedingJoinPoint point,
                         CheckFieldEditPermission checkPermission) throws Throwable {

        String appId = TenantContext.getCurrentTenant();
        String userId = SecurityContext.getCurrentUserId();
        String entityCode = checkPermission.entity();

        // 从方法参数中提取修改的字段
        Map<String, Object> changedFields = extractChangedFields(point);

        // 检查每个修改的字段是否有编辑权限
        for (String fieldCode : changedFields.keySet()) {
            if (!fieldPermissionService.isFieldEditable(appId, userId, entityCode, fieldCode)) {
                throw new PermissionDeniedException(
                    "无权限编辑字段: " + fieldCode);
            }
        }

        return point.proceed();
    }

    private Map<String, Object> extractChangedFields(ProceedingJoinPoint point) {
        // 从参数中提取字段变更信息
        // 实现细节取决于具体的参数结构
        // ...
    }
}

/**
 * 字段编辑权限检查注解
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface CheckFieldEditPermission {
    String entity();  // 实体编码
}
```

## 5. 前端实现

### 5.1 字段权限 Hook

```typescript
// useFieldPermission.ts
import { useMemo } from 'react';

interface FieldPermission {
  visible: boolean;
  editable: boolean;
  masking?: {
    enabled: boolean;
    type: 'full' | 'partial' | 'regex';
  };
}

interface UseFieldPermissionOptions {
  appId: string;
  entityCode: string;
}

export const useFieldPermission = (options: UseFieldPermissionOptions) => {
  const { appId, entityCode } = options;

  // 从用户权限上下文获取字段权限
  const { userPermissions } = useAuth();

  const fieldPermissions = useMemo(() => {
    return userPermissions.fieldPermissions?.[entityCode] || {};
  }, [userPermissions, entityCode]);

  /**
   * 检查字段是否可见
   */
  const isFieldVisible = (fieldCode: string): boolean => {
    const permission = fieldPermissions[fieldCode];
    return permission?.visible !== false;  // 默认可见
  };

  /**
   * 检查字段是否可编辑
   */
  const isFieldEditable = (fieldCode: string): boolean => {
    const permission = fieldPermissions[fieldCode];
    return permission?.visible !== false && permission?.editable !== false;
  };

  /**
   * 获取字段脱敏配置
   */
  const getFieldMasking = (fieldCode: string) => {
    const permission = fieldPermissions[fieldCode];
    return permission?.masking;
  };

  /**
   * 过滤字段列表
   */
  const filterVisibleFields = <T extends { code: string }>(
    fields: T[]
  ): T[] => {
    return fields.filter(field => isFieldVisible(field.code));
  };

  /**
   * 转换表单字段为只读
   */
  const convertToReadOnly = <T extends { code: string; props?: any }>(
    fields: T[]
  ): T[] => {
    return fields.map(field => ({
      ...field,
      props: {
        ...field.props,
        readOnly: !isFieldEditable(field.code),
        disabled: !isFieldEditable(field.code),
      },
    }));
  };

  return {
    fieldPermissions,
    isFieldVisible,
    isFieldEditable,
    getFieldMasking,
    filterVisibleFields,
    convertToReadOnly,
  };
};
```

### 5.2 权限表单组件

```tsx
// PermissionForm.tsx
import React from 'react';
import { Form, Input, InputNumber } from 'antd';
import { useFieldPermission } from './useFieldPermission';

interface PermissionFormProps {
  appId: string;
  entityCode: string;
  fields: FormField[];
  initialValues?: Record<string, any>;
  onSubmit?: (values: Record<string, any>) => void;
}

export const PermissionForm: React.FC<PermissionFormProps> = ({
  appId,
  entityCode,
  fields,
  initialValues,
  onSubmit,
}) => {
  const { isFieldVisible, isFieldEditable, getFieldMasking } = useFieldPermission({
    appId,
    entityCode,
  });

  const [form] = Form.useForm();

  // 过滤不可见字段
  const visibleFields = fields.filter(field => isFieldVisible(field.code));

  // 渲染字段
  const renderField = (field: FormField) => {
    const editable = isFieldEditable(field.code);
    const masking = getFieldMasking(field.code);

    const commonProps = {
      disabled: !editable,
      readOnly: !editable,
    };

    // 根据字段类型渲染不同组件
    switch (field.type) {
      case 'number':
        return (
          <InputNumber
            {...commonProps}
            style={{ width: '100%' }}
            formatter={masking?.enabled ? (value) => applyMasking(value, masking) : undefined}
          />
        );
      case 'text':
      default:
        return (
          <Input
            {...commonProps}
            type={masking?.enabled ? 'password' : 'text'}
          />
        );
    }
  };

  return (
    <Form
      form={form}
      initialValues={initialValues}
      onFinish={onSubmit}
    >
      {visibleFields.map(field => (
        <Form.Item
          key={field.code}
          name={field.code}
          label={field.name}
          rules={field.rules}
        >
          {renderField(field)}
        </Form.Item>
      ))}

      <Form.Item>
        <Button type="primary" htmlType="submit">
          保存
        </Button>
      </Form.Item>
    </Form>
  );
};

// 脱敏显示值
function applyMasking(value: any, masking: { type: string }): string {
  if (value == null) return '';
  const str = String(value);

  switch (masking.type) {
    case 'full':
      return '*'.repeat(str.length);
    case 'partial':
      if (str.length <= 4) return '****';
      return str.substring(0, 2) + '****' + str.substring(str.length - 2);
    default:
      return str;
  }
}
```

### 5.3 权限表格组件

```tsx
// PermissionTable.tsx
import React, { useMemo } from 'react';
import { Table } from 'antd';
import { useFieldPermission } from './useFieldPermission';

interface PermissionTableProps<T> {
  appId: string;
  entityCode: string;
  columns: TableColumn<T>[];
  dataSource: T[];
}

export function PermissionTable<T>({
  appId,
  entityCode,
  columns,
  dataSource,
}: PermissionTableProps<T>) {
  const { isFieldVisible, getFieldMasking } = useFieldPermission({
    appId,
    entityCode,
  });

  // 过滤不可见列
  const visibleColumns = useMemo(() => {
    return columns
      .filter(col => {
        // 如果列没有绑定字段，默认显示
        if (!col.dataIndex) return true;
        return isFieldVisible(col.dataIndex as string);
      })
      .map(col => {
        // 添加脱敏渲染
        if (col.dataIndex && getFieldMasking(col.dataIndex as string)?.enabled) {
          const originalRender = col.render;
          const masking = getFieldMasking(col.dataIndex as string);

          return {
            ...col,
            render: (value: any, record: T, index: number) => {
              const maskedValue = applyMasking(value, masking!);
              return originalRender
                ? originalRender(maskedValue, record, index)
                : maskedValue;
            },
          };
        }
        return col;
      });
  }, [columns, isFieldVisible, getFieldMasking]);

  return (
    <Table
      columns={visibleColumns}
      dataSource={dataSource}
      rowKey="id"
    />
  );
}
```

## 6. 管理界面设计

### 6.1 字段权限配置界面

```
┌─────────────────────────────────────────────────────────────────┐
│                    角色权限配置 - 销售代表                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  [页面权限] [按钮权限] [字段权限] [数据权限]                      │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ 实体: 订单 (order)                                          ││
│  │                                                             ││
│  │  ┌─────────────────┬─────────┬──────────┬────────────────┐  ││
│  │  │ 字段名称         │ 可见    │ 可编辑   │ 脱敏           │  ││
│  │  ├─────────────────┼─────────┼──────────┼────────────────┤  ││
│  │  │ 订单编号         │ ☑       │ ☐        │ ☐              │  ││
│  │  │ 客户名称         │ ☑       │ ☑        │ ☐              │  ││
│  │  │ 订单金额         │ ☑       │ ☑        │ ☑ 部分脱敏     │  ││
│  │  │ 成本价           │ ☐       │ ☐        │ ☐              │  ││
│  │  │ 创建人           │ ☑       │ ☐        │ ☐              │  ││
│  │  │ 创建时间         │ ☑       │ ☐        │ ☐              │  ││
│  │  └─────────────────┴─────────┴──────────┴────────────────┘  ││
│  │                                                             ││
│  │ [批量设置] [复制到其他实体]                                  ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ 实体: 客户 (customer)                                       ││
│  │ ...                                                         ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│                    [  保存  ]  [  取消  ]                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 批量配置界面

```
┌─────────────────────────────────────────────────────────────────┐
│                    批量设置字段权限                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  应用到字段:                                                     │
│  ☑ 订单金额    ☑ 折扣金额    ☑ 实付金额                          │
│  ☐ 成本价                                                      │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  权限设置:                                                       │
│                                                                 │
│  可见性:  ○ 可见  ● 隐藏                                        │
│                                                                 │
│  可编辑性:  ○ 可编辑  ● 只读                                    │
│                                                                 │
│  脱敏:  ○ 不脱敏  ● 部分脱敏  ○ 完全脱敏  ○ 正则脱敏            │
│                                                                 │
│           [  取消  ]    [  确定  ]                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 7. 使用示例

### 7.1 后端接口

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @GetMapping("/{id}")
    @EntityResponse("order")  // 启用字段权限过滤
    public Result<OrderVO> getOrder(@PathVariable String id) {
        OrderVO order = orderService.getOrder(id);
        return Result.success(order);
    }

    @GetMapping
    @EntityResponse("order")  // 列表也启用字段权限过滤
    public Result<List<OrderVO>> listOrders(OrderQuery query) {
        List<OrderVO> orders = orderService.listOrders(query);
        return Result.success(orders);
    }

    @PutMapping("/{id}")
    @CheckFieldEditPermission(entity = "order")  // 检查字段编辑权限
    public Result<Void> updateOrder(@PathVariable String id,
                                     @RequestBody OrderUpdateDTO dto) {
        orderService.updateOrder(id, dto);
        return Result.success();
    }
}
```

### 7.2 前端页面

```tsx
// OrderDetailPage.tsx
import React from 'react';
import { PermissionForm } from '@/components/PermissionForm';

const ORDER_FIELDS = [
  { code: 'orderNo', name: '订单编号', type: 'text' },
  { code: 'customerName', name: '客户名称', type: 'text' },
  { code: 'amount', name: '订单金额', type: 'number' },
  { code: 'costPrice', name: '成本价', type: 'number' },
  { code: 'createdBy', name: '创建人', type: 'text' },
];

export const OrderDetailPage: React.FC = () => {
  const handleSubmit = async (values: Record<string, any>) => {
    await updateOrder(orderId, values);
  };

  return (
    <PermissionForm
      appId="app_crm"
      entityCode="order"
      fields={ORDER_FIELDS}
      initialValues={orderData}
      onSubmit={handleSubmit}
    />
  );
};
```

## 8. 与其他权限的关系

```
┌─────────────────────────────────────────────────────────────┐
│                     权限体系层级                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 页面权限 (Page)                                         │
│     └─ 控制能否进入某个页面                                 │
│                                                             │
│  2. 按钮权限 (Button)                                       │
│     └─ 控制能否执行某个操作（如提交、导出）                  │
│                                                             │
│  3. 列权限 (Field)  ← 本设计                                │
│     ├─ visible: 控制能否看到某列数据                        │
│     ├─ editable: 控制能否编辑某列数据                       │
│     └─ masking: 控制数据是否需要脱敏显示                    │
│                                                             │
│  4. 数据权限 (Data)                                         │
│     └─ 控制能看到哪些行数据（数据范围）                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘

执行顺序：页面 → 数据 → 列 → 按钮
- 先检查是否有页面权限
- 再检查数据范围（能看哪些行）
- 然后过滤列（能看哪些字段）
- 最后检查按钮权限（能执行哪些操作）
```
