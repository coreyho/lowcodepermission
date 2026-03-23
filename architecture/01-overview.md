# 整体架构设计

## 1. 权限模型

采用 **RBAC + ABAC 混合模型**：

- **RBAC（基于角色的访问控制）**：处理功能权限
  - 角色继承模板
  - 权限码集合管理
  - 简化权限分配

- **ABAC（基于属性的访问控制）**：处理数据权限
  - 基于用户属性动态计算
  - 支持复杂规则表达式
  - 灵活的数据范围控制

## 2. 核心概念

```
┌─────────────────────────────────────────────────────────────┐
│                      平台底座 (Platform)                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              应用配置空间 (App Config Space)         │   │
│  │  • 页面设计器  • 实体设计器  • 流程设计器            │   │
│  │  • 权限配置面板  • 应用发布管理                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                          ↓ 发布（写入权限管理服务）           │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              应用运行空间 (App Runtime)              │   │
│  │  • 独立URL访问  • 调用权限管理服务（HTTP）            │   │
│  │  • 携带 app_id 区分租户，无独立权限后端               │   │
│  └─────────────────────────────────────────────────────┘   │
│                          ↑ HTTP (X-App-Id 请求头)           │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         权限管理服务 (Permission Service)            │   │
│  │  • 唯一后端权限服务  • 所有应用共享                   │   │
│  │  • 按 app_id 隔离权限数据（页面/按钮/字段/规则/角色） │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 关键概念说明

| 概念 | 说明 |
|------|------|
| **app_id** | 应用唯一标识，作为租户键；所有权限资源（页面/按钮/字段/规则/角色）均以 app_id 为命名空间隔离，DSL 和数据库表均以此区分应用 |
| **App Config Space** | 应用配置期空间，开发者配置权限模板 |
| **App Runtime** | 应用运行期空间，运营管理员配置具体权限；无独立权限后端，调用共享权限管理服务 |
| **Permission Service** | 唯一的后端权限管理服务，多应用共享，按 app_id 实现数据隔离 |
| **Permission Code** | 权限码，在同一 app_id 内唯一标识一个权限点 |
| **Role Template** | 角色模板，配置期定义，运行期继承 |
| **Data Dimension** | 数据权限维度（组织/部门/Owner等） |
| **Data Rule** | 数据权限规则表达式 |

## 3. 权限层级

```
功能权限层级:
├── 页面级 (Page)
│   ├── 菜单可见性
│   └── 路由访问控制
├── 按钮级 (Button)
│   ├── 工具栏按钮
│   ├── 行操作按钮
│   └── 表单按钮
├── 字段级 (Field)
│   ├── 字段可见性
│   ├── 字段可编辑性
│   └── 数据脱敏
└── API级 (API)
    ├── HTTP方法控制
    └── 接口路径控制

数据权限层级:
├── 组织维度 (Organization)
├── 部门维度 (Department)
├── Owner维度
└── 自定义维度
```

## 4. 两阶段权限配置模型

权限管理采用**配置期生成 → 运行期分配**的两阶段模型：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        应用配置期 (Config Space)                        │
│                                                                         │
│  使用者：应用开发者 / 页面设计人员                                       │
│                                                                         │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐ │
│  │  设计页面    │   │  配置按钮    │   │  定义字段    │   │  标记API    │ │
│  │  (Page)     │   │  (Button)   │   │  (Field)    │   │  (API)      │ │
│  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘ │
│         │                 │                 │                 │        │
│         └─────────────────┴─────────────────┴─────────────────┘        │
│                                   │                                     │
│                                   ↓                                     │
│                    ┌─────────────────────────────┐                     │
│                    │   生成功能权限配置项 DSL      │                     │
│                    │   （定义有哪些权限可用）       │                     │
│                    └──────────────┬──────────────┘                     │
│                                   │                                     │
└───────────────────────────────────┼─────────────────────────────────────┘
                                    │ 发布 / 同步
                                    ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                        应用运行期 (Runtime)                             │
│                                                                         │
│  使用者：运营管理员 / 权限管理员                                         │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     权限管理服务 (Permission Service)            │   │
│  │                                                                 │   │
│  │   ┌─────────────────┐        ┌─────────────────────────────┐   │   │
│  │   │  读取配置项       │──────→│  创建角色并分配权限            │   │   │
│  │   │  (可用权限码列表)  │        │  (将权限项分配给具体角色)      │   │   │
│  │   └─────────────────┘        │                             │   │   │
│  │                              │  • 销售经理：page_order,      │   │   │
│  │                              │    btn_order_create,          │   │   │
│  │                              │    btn_order_edit...          │   │   │
│  │                              │                             │   │   │
│  │                              │  • 销售代表：page_order,      │   │   │
│  │                              │    btn_order_view...          │   │   │
│  │                              │                             │   │   │
│  │                              └─────────────────────────────┘   │   │
│  │                                          │                      │   │
│  │                                          ↓                      │   │
│  │                              ┌─────────────────────┐           │   │
│  │                              │  用户登录时计算权限  │           │   │
│  │                              │  缓存至 Redis       │           │   │
│  │                              └─────────────────────┘           │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.1 阶段对比

| 维度 | 应用配置期 (Config) | 应用运行期 (Runtime) |
|------|---------------------|----------------------|
| **使用者** | 应用开发者、页面设计师 | 运营管理员、权限管理员 |
| **操作对象** | 权限配置项（定义有哪些权限） | 角色与权限分配（谁能用什么） |
| **输出物** | 权限 DSL（pages/buttons/fields/apis） | 角色-权限绑定关系 |
| **存储位置** | 设计器数据库 → 发布到权限服务 | 权限服务数据库 |
| **变更频率** | 低（随应用版本发布） | 高（日常权限调整） |
| **典型操作** | 新增页面、添加按钮、配置脱敏字段 | 创建角色、为用户分配角色、调整角色权限 |

### 4.2 配置项类型

配置期生成的权限配置项作为运行期分配的基础：

| 配置项类型 | 配置期定义内容 | 运行期分配方式 |
|------------|----------------|----------------|
| **页面 (Page)** | 页面编码、名称、路由、菜单显示 | 角色勾选可访问的页面 |
| **按钮 (Button)** | 按钮编码、名称、所属页面、位置 | 角色勾选可操作的功能 |
| **字段 (Field)** | 字段编码、名称、所属页面、脱敏配置 | 角色勾选可见/可编辑的字段 |
| **API** | API编码、方法、路径、限流配置 | 角色勾选可调用的接口 |
| **数据规则** | 规则编码、表达式、适用实体 | 角色按实体绑定规则（READ/WRITE） |

## 5. 权限生效流程

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   用户请求   │────→│  权限校验层  │────→│  业务处理层  │
└─────────────┘     └─────────────┘     └─────────────┘
                           │
                    ┌──────┴──────┐
                    ↓             ↓
              ┌─────────┐   ┌─────────┐
              │功能权限 │   │数据权限 │
              │  校验   │   │  过滤   │
              └─────────┘   └─────────┘
                    │             │
              检查权限码      注入SQL条件
```

## 6. 技术选型

| 组件 | 选型 | 说明 |
|------|------|------|
| 权限引擎 | 自研 | 基于 Spring Boot 3 |
| 规则引擎 | QLExpress / Drools | 数据权限表达式计算 |
| SQL解析 | JSqlParser | 动态修改SQL |
| 缓存 | Redis | 权限缓存，key 格式：`perm:{appId}:{userId}` |
| 数据库 | MySQL 8 | 权限配置存储，所有权限表均含 `app_id` 列 |
| 搜索引擎 | Elasticsearch | 审计日志 |

## 7. 多租户隔离设计

运行时只有一个共享权限管理服务，所有应用（租户）通过 `app_id` 隔离权限数据。

### 7.1 数据库表隔离（行级）

所有权限表均以 `app_id` 作为第一分区键：

```sql
-- 示例：功能权限表（页面/按钮/字段/API 同理）
CREATE TABLE perm_page (
  id        BIGINT       NOT NULL AUTO_INCREMENT,
  app_id    VARCHAR(64)  NOT NULL COMMENT '应用唯一标识（租户键）',
  code      VARCHAR(128) NOT NULL COMMENT '权限码，同一应用内唯一',
  name      VARCHAR(128) NOT NULL,
  -- ...其他字段...
  PRIMARY KEY (id),
  UNIQUE KEY uk_app_code (app_id, code),  -- 联合唯一，不同应用可有同名权限码
  KEY idx_app_id (app_id)
) COMMENT = '页面级权限';

-- 角色表
CREATE TABLE perm_role (
  id        BIGINT       NOT NULL AUTO_INCREMENT,
  app_id    VARCHAR(64)  NOT NULL COMMENT '应用唯一标识（租户键）',
  code      VARCHAR(128) NOT NULL,
  name      VARCHAR(128) NOT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uk_app_code (app_id, code),
  KEY idx_app_id (app_id)
);
```

### 7.2 Redis 缓存隔离（key 前缀）

| 缓存类型 | Key 格式 | 说明 |
|----------|----------|------|
| 用户功能权限 | `perm:{appId}:{userId}` | 按应用 + 用户缓存权限码集合 |
| API 限流计数 | `ratelimit:{appId}:{userId}:{apiCode}` | 按应用隔离限流状态 |

### 7.3 DSL 顶层字段

配置期生成的权限 DSL 文件顶层必须携带 `app_id`，发布时作为导入键，权限管理服务以此归属所有资源：

```yaml
app_id: "app_crm"          # 必填，应用唯一标识
version: "1.0.0"           # DSL 版本，用于变更追踪
```

## 8. 应用配置导出与导入

应用支持跨平台导出和导入，权限配置项作为应用的核心配置之一，必须随应用一并导出和导入。

### 8.1 导出包结构

应用导出为一个 ZIP 包，包含应用的所有配置：

```
app_export_{app_id}_{version}_{timestamp}.zip
├── manifest.json                    # 导出清单，描述包元数据
├── metadata/
│   └── app_info.json               # 应用基本信息（名称、描述、图标）
├── permissions/
│   ├── functional_permission.yaml  # 功能权限配置（pages/buttons/fields/apis）
│   └── data_permission.yaml        # 数据权限配置（dimensions/rules/templates）
├── entities/
│   ├── entity_order.json           # 实体定义
│   └── entity_customer.json
├── pages/
│   ├── page_order_list.json        # 页面设计器配置
│   └── page_order_detail.json
└── workflows/
    └── workflow_approval.json      # 流程配置（如适用）
```

### 8.2 权限项导出内容

**功能权限配置项** (`permissions/functional_permission.yaml`)：

```yaml
# 导出格式与配置期 DSL 一致
export_metadata:
  export_version: "1.0.0"           # 导出格式版本
  exported_at: "2024-01-15T10:30:00Z"
  source_platform: "lowcode-platform-v2"
  app_id: "app_crm"                 # 原应用 ID（导入时可映射为新 ID）

# 权限配置项（配置期生成，运行期分配的基础）
permissions:
  pages:
    - code: "page_dashboard"
      name: "仪表盘"
      type: "page"
      path: "/dashboard"
      # ... 其他页面属性

  buttons:
    - code: "btn_order_create"
      name: "新建订单"
      page_code: "page_order_list"
      location: "toolbar"
      # ...

  fields:
    - code: "field_customer_phone"
      name: "客户电话"
      page_code: "page_order_detail"
      data_masking:
        enable: true
        type: "regex"
        match_pattern: "(\d{3})\d{4}(\d{4})"
        replace_with: "$1****$2"
        masking_side: "backend"
      # ...

  apis:
    - code: "api_order_export"
      method: "POST"
      path: "/api/orders/export"
      rate_limit:
        limit: 100
        window: "1h"
```

**数据权限配置项** (`permissions/data_permission.yaml`)：

```yaml
export_metadata:
  export_version: "1.0.0"
  exported_at: "2024-01-15T10:30:00Z"
  source_platform: "lowcode-platform-v2"

data_permissions:
  dimensions:
    - code: "dim_org"
      name: "组织维度"
      type: "organization"
      entity_field: "org_id"

  rule_templates:
    - code: "rule_self_only"
      name: "仅本人数据"
      expression: "created_by == ${currentUserId}"

  data_rules:
    - id: "rule_dept_and_sub"
      name: "本部门及子部门数据"
      entity: "order"
      type: "template"
      template: "rule_dept_and_sub_template"
      operations: ["READ", "WRITE"]
```

### 8.3 导出流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          应用导出流程                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. 收集配置                                                             │
│     ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │
│     │  读取页面配置 │  │ 读取实体配置 │  │ 读取权限配置 │  │ 读取流程配置 │  │
│     │  (pages)    │  │ (entities)  │  │(permissions)│  │ (workflows) │  │
│     └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  │
│            │                │                │                │         │
│            └────────────────┴────────────────┴────────────────┘         │
│                                     │                                    │
│                                     ↓                                    │
│  2. 组装导出包                                                          │
│     ┌─────────────────────────────────────────────────────────────┐     │
│     │  生成 manifest.json（包含权限项校验和）                      │     │
│     │  序列化所有配置为 YAML/JSON                                  │     │
│     │  打包为 ZIP                                                  │     │
│     └─────────────────────────────────────────────────────────────┘     │
│                                     │                                    │
│                                     ↓                                    │
│  3. 下载导出包                                                          │
│     app_export_app_crm_v1.2.3_20240115103000.zip                        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.4 导入流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          应用导入流程                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. 上传与校验                                                           │
│     ┌─────────────────────────────────────────────────────────────┐     │
│     │  • 解析 manifest.json，检查格式版本兼容性                    │     │
│     │  • 校验必要文件是否存在（permissions/*.yaml）                │     │
│     │  • 校验权限项 code 命名规范（避免与现有冲突）                │     │
│     │  • 显示预览：列出所有权限配置项供确认                        │     │
│     └─────────────────────────────────────────────────────────────┘     │
│                                     │                                    │
│                                     ↓                                    │
│  2. ID 映射与冲突处理                                                    │
│     ┌─────────────────────────────────────────────────────────────┐     │
│     │  app_id 映射：                                               │     │
│     │    原 app_id: "app_crm"  →  新 app_id: "app_crm_imported"   │     │
│     │                                                              │     │
│     │  冲突检测（同一目标平台）：                                  │     │
│     │    • 若 permission code 已存在 → 提示：跳过/覆盖/重命名      │     │
│     │    • 若 entity code 已存在    → 提示：跳过/覆盖/合并         │     │
│     └─────────────────────────────────────────────────────────────┘     │
│                                     │                                    │
│                                     ↓                                    │
│  3. 执行导入                                                             │
│     ┌─────────────────────────────────────────────────────────────┐     │
│     │  事务性导入：                                                │     │
│     │    a. 导入功能权限配置项（pages/buttons/fields/apis）        │     │
│     │    b. 导入数据权限配置项（dimensions/rules）                 │     │
│     │    c. 导入实体和页面配置                                     │     │
│     │    d. 生成新的 app_id 下的完整配置                           │     │
│     └─────────────────────────────────────────────────────────────┘     │
│                                     │                                    │
│                                     ↓                                    │
│  4. 导入完成                                                             │
│     • 新应用创建成功，权限配置项已就绪                                   │
│     • 运营管理员可在运行期基于此配置创建角色并分配权限                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.5 跨平台兼容性设计

**版本兼容策略：**

| 场景 | 处理策略 |
|------|----------|
| 导出版本 = 导入平台版本 | 完全兼容，直接导入 |
| 导出版本 < 导入平台版本 | 自动迁移：平台提供 migrator 将旧格式升级为新格式 |
| 导出版本 > 导入平台版本 | 拒绝导入，提示用户升级平台版本 |

**格式标准化：**

```yaml
# manifest.json
{
  "format_version": "1.0.0",           # 导出格式版本（语义化版本）
  "platform": {
    "name": "lowcode-platform",        # 平台标识
    "version": "2.5.0"                 # 平台版本
  },
  "app": {
    "original_id": "app_crm",
    "name": "CRM系统",
    "version": "1.2.3"
  },
  "permissions": {
    "export_scope": "config_only",     # config_only | config_with_roles
    "page_count": 12,
    "button_count": 35,
    "field_count": 78,
    "api_count": 45,
    "data_rule_count": 8,
    "checksum": "sha256:abc123..."     # 校验和，防篡改
  }
}
```

**跨平台适配映射：**

当导入到不同平台时，可能需要适配：

```yaml
# permissions/platform_mappings.yaml（可选，由目标平台提供）
mappings:
  # 图标映射（若平台图标体系不同）
  icons:
    "DashboardOutlined": "mdi-view-dashboard"
    "UserOutlined": "mdi-account"

  # 组件类型映射（若页面设计器组件不同）
  components:
    "antd/Table": "element-plus/Table"
    "antd/Form": "element-plus/Form"

  # 权限属性映射
  permission_attrs:
    "masking_side":  # 若目标平台不支持 backend 模式
      "backend": "frontend"  # 降级处理
```

### 8.6 权限项冲突处理

导入时可能遇到权限项 code 冲突，提供以下处理策略：

| 冲突类型 | 策略选项 | 说明 |
|----------|----------|------|
| **Page Code 冲突** | 跳过 | 保留现有页面配置，跳过导入的页面 |
| | 覆盖 | 用导入的配置替换现有配置 |
| | 重命名 | 自动添加后缀（如 `page_order_list_imported`） |
| **Button Code 冲突** | 合并 | 若页面相同，合并按钮列表（去重） |
| | 替换 | 替换整个页面的按钮配置 |
| **Field Code 冲突** | 覆盖 | 更新字段配置（如脱敏规则） |
| **Data Rule 冲突** | 跳过 | 保留现有规则 |
| | 新建版本 | 创建新版本的规则（如 `rule_self_only_v2`） |

**冲突检测算法：**

```java
public class ImportConflictDetector {

    public List<Conflict> detectConflicts(AppImportContext context) {
        List<Conflict> conflicts = new ArrayList<>();
        String targetAppId = context.getTargetAppId();

        // 1. 检测功能权限冲突
        for (PermissionItem item : context.getImportPermissions()) {
            if (permissionService.exists(targetAppId, item.getCode())) {
                conflicts.add(new Conflict(
                    ConflictType.PERMISSION_CODE_DUPLICATE,
                    item.getCode(),
                    "权限码已存在: " + item.getCode()
                ));
            }
        }

        // 2. 检测数据规则冲突
        for (DataRule rule : context.getImportDataRules()) {
            if (dataRuleService.exists(targetAppId, rule.getId())) {
                conflicts.add(new Conflict(
                    ConflictType.DATA_RULE_ID_DUPLICATE,
                    rule.getId(),
                    "数据规则ID已存在: " + rule.getId()
                ));
            }
        }

        return conflicts;
    }
}
```

### 8.7 运行时角色配置导出（可选）

除了配置期的权限项，运行期的角色配置也可选择性导出：

```yaml
# 导出范围：config_only（仅配置项）或 full（配置项+角色分配）
export_scope: "full"

# 运行时角色配置（仅当 export_scope=full 时包含）
runtime_roles:
  - code: "role_sales_manager"
    name: "销售经理"
    source: "template:role_manager"
    functional_permissions:           # 分配的功能权限码列表
      - "page_dashboard"
      - "page_order_list"
      - "btn_order_create"
      - "btn_order_edit"
      - "field_cost_price"
    data_permissions:
      - entity: "order"
        rule: "rule_dept_and_sub"
        operations: ["READ", "WRITE"]

  - code: "role_sales_rep"
    name: "销售代表"
    functional_permissions:
      - "page_order_list"
      - "btn_order_view"
    data_permissions:
      - entity: "order"
        rule: "rule_self_only"
        operations: ["READ", "WRITE"]
```

**导入运行时角色的注意事项：**

1. **依赖校验**：导入角色前，必须确保其引用的所有权限码已存在
2. **模板映射**：若引用了角色模板（`source: "template:xxx"`），需在目标平台存在同名模板，或转换为独立角色
3. **数据规则映射**：角色绑定的数据规则 ID 需映射为目标平台的新 ID

### 8.8 导入验证与回滚

```java
@Service
public class AppImportService {

    @Transactional(rollbackFor = Exception.class)
    public ImportResult importApp(AppPackage appPackage, ImportOptions options) {
        String newAppId = generateAppId();

        try {
            // 1. 导入权限配置项
            List<PermissionItem> permissions = appPackage.getPermissions();
            for (PermissionItem perm : permissions) {
                perm.setAppId(newAppId);
                permissionService.save(perm);
            }

            // 2. 导入数据权限配置
            List<DataRule> dataRules = appPackage.getDataRules();
            for (DataRule rule : dataRules) {
                rule.setAppId(newAppId);
                dataRuleService.save(rule);
            }

            // 3. 导入其他配置（页面、实体等）
            // ...

            // 4. 验证导入完整性
            validateImport(newAppId, appPackage.getManifest());

            return ImportResult.success(newAppId);

        } catch (Exception e) {
            // 事务回滚，清理已导入数据
            throw new ImportException("应用导入失败", e);
        }
    }

    private void validateImport(String appId, Manifest manifest) {
        // 校验导入的权限项数量与 manifest 一致
        int actualPageCount = permissionService.countPages(appId);
        assert actualPageCount == manifest.getPermissions().getPageCount();

        // 校验数据规则完整性
        // ...
    }
}
```

### 8.9 关键设计原则

1. **权限配置项必导**：pages、buttons、fields、apis、data_rules 等权限配置项必须完整导出，确保目标平台具备相同的权限控制能力

2. **角色配置可选导**：运行期的角色-权限绑定关系为可选导出，避免不同平台的角色体系冲突

3. **ID 重新生成**：导入时生成新的 `app_id`，避免与现有应用冲突

4. **Code 保持一致**：权限码（permission code）保持原值，确保跨平台一致性；冲突时由用户决定处理策略

5. **向后兼容**：导出格式版本化管理，支持旧版本导入到新平台（自动迁移）
