file_path="/home/heankang/lowcode-permission-design/architecture/06-permission-code-generator.md"
content="# 权限码自动生成算法

## 1. 设计目标

- **无感生成**：用户在页面设计器中添加页面/按钮/字段时，系统自动生成符合规范的权限码
- **唯一性保证**：同一 `app_id` 内权限码唯一，避免冲突
- **可读性优先**：生成的code应能直观反映其含义
- **跨平台兼容**：支持导出导入时的code映射和冲突处理
- **平台与应用分离**：区分**平台权限**（IAM自管理）和**应用权限**（各应用独立）

## 2. 命名规范

### 2.1 权限类型与命名空间

```
┌─────────────────────────────────────────────────────────────────┐
│                        权限服务体系 (IAM)                        │
├─────────────────────────────┬───────────────────────────────────┤
│      平台权限 (Platform)      │        应用权限 (App)              │
│   命名空间: _system          │    命名空间: {app_id}              │
├─────────────────────────────┼───────────────────────────────────┤
│  • 应用管理                  │  • 页面访问 (page_*)               │
│  • 用户管理                  │  • 按钮操作 (btn_*)                │
│  • 角色模板管理              │  • 字段可见 (field_*)              │
│  • 平台配置                  │  • 字段可见 (field_*)              │
│  • 审计日志查看              │  • 数据规则                        │
├─────────────────────────────┴───────────────────────────────────┤
│  数据库隔离: app_id = "_system"  |  app_id = "app_crm" 等        │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 格式标准

#### 应用权限格式（运行时应用）

```
{prefix}_{entity}_{action}
{prefix}_{entity}_{action}_{location}
{prefix}_{entity}_{field_name}
```

| 类型 | 前缀 | 命名规则 | 示例 |
|------|------|----------|------|
| **页面** | `page` | `page_{entity}_{action}` | `page_order_list`, `page_dashboard` |
| **按钮** | `btn` | `btn_{entity}_{action}` | `btn_order_create`, `btn_order_delete` |
| **字段** | `field` | `field_{entity}_{field_name}` | `field_order_amount`, `field_customer_phone` |
| **数据规则** | `rule` | `rule_{entity}_{scope}` | `rule_order_dept`, `rule_customer_self` |

#### 平台权限格式（IAM平台本身）

```
sys_{module}_{resource}_{action}
```

| 类型 | 前缀 | 命名规则 | 示例 |
|------|------|----------|------|
| **应用管理** | `sys_app` | `sys_app_{action}` | `sys_app_create`, `sys_app_publish` |
| **用户管理** | `sys_user` | `sys_user_{action}` | `sys_user_create`, `sys_user_assign_role` |
| **角色模板** | `sys_role_tpl` | `sys_role_tpl_{action}` | `sys_role_tpl_create`, `sys_role_tpl_edit` |
| **平台配置** | `sys_config` | `sys_config_{action}` | `sys_config_modify`, `sys_config_view_audit` |
| **导入导出** | `sys_import` | `sys_import_{action}` | `sys_import_app`, `sys_export_app` |

### 2.3 权限码生成入口区分

```java
/**
 * 权限码生成器入口
 */
@Service
public class PermissionCodeGenerator {

    @Autowired
    private PlatformPermissionGenerator platformGenerator;

    @Autowired
    private AppPermissionGenerator appGenerator;

    /**
     * 生成权限码（自动识别类型）
     */
    public String generate(PermissionGenerateRequest request) {
        // 平台权限：appId = "_system" 或 type = PLATFORM
        if (isPlatformPermission(request)) {
            return platformGenerator.generate(request);
        }

        // 应用权限：appId = 具体应用ID
        return appGenerator.generate(request);
    }

    private boolean isPlatformPermission(PermissionGenerateRequest request) {
        return "_system".equals(request.getAppId()) ||
               request.getType().startsWith("sys_");
    }
}

### 2.2 命名转换规则

```java
/**
 * 中文名称转换为code（核心算法）
 */
public class PermissionCodeGenerator {

    private static final int MAX_CODE_LENGTH = 64;
    private static final int MAX_ENTITY_LENGTH = 20;
    private static final int MAX_ACTION_LENGTH = 30;

    /**
     * 主入口：生成权限码
     */
    public String generate(String prefix, String entity, String action,
                          GenerateOptions options) {
        // 1. 标准化输入
        String normalizedEntity = normalize(entity);
        String normalizedAction = normalize(action);

        // 2. 截断处理（优先保证可读性）
        normalizedEntity = truncate(normalizedEntity, MAX_ENTITY_LENGTH);
        normalizedAction = truncate(normalizedAction, MAX_ACTION_LENGTH);

        // 3. 组装基础code
        String baseCode = String.format("%s_%s_%s",
            prefix, normalizedEntity, normalizedAction);

        // 4. 冲突检测与处理
        return resolveConflict(baseCode, options);
    }

    /**
     * 标准化处理
     */
    private String normalize(String input) {
        if (StringUtils.isBlank(input)) {
            return "unnamed";
        }

        // 1. 中文转拼音（多音字处理见 2.3 节）
        String pinyin = PinyinHelper.toPinyin(input, PinyinStyle.NORMAL);

        // 2. 特殊字符替换
        pinyin = pinyin
            .replaceAll("[^a-zA-Z0-9\\u4e00-\\u9fa5]", "_")  // 特殊字符转下划线
            .replaceAll("_{2,}", "_")                       // 连续下划线合并
            .toLowerCase();

        // 3. 移除首尾下划线
        pinyin = StringUtils.strip(pinyin, "_");

        return pinyin;
    }

    /**
     * 智能截断（保留语义完整性）
     */
    private String truncate(String input, int maxLength) {
        if (input.length() <= maxLength) {
            return input;
        }

        // 按语义单元截断（优先保留后半部分，通常是动作）
        String[] parts = input.split("_");
        StringBuilder result = new StringBuilder();

        // 从后往前组装，确保动作词完整
        for (int i = parts.length - 1; i >= 0; i--) {
            String candidate = parts[i] + (result.length() > 0 ? "_" + result : "");
            if (candidate.length() <= maxLength) {
                result.insert(0, parts[i]);
                if (i > 0) result.insert(0, "_");
            } else {
                break;
            }
        }

        return result.toString();
    }

    /**
     * 冲突解决
     */
    private String resolveConflict(String baseCode, GenerateOptions options) {
        String appId = options.getAppId();
        String finalCode = baseCode;
        int suffix = 1;

        // 检查同一app_id下是否已存在
        while (permissionRepository.exists(appId, finalCode)) {
            finalCode = baseCode + "_" + suffix;
            suffix++;

            // 超出长度限制时截断
            if (finalCode.length() > MAX_CODE_LENGTH) {
                int overflow = finalCode.length() - MAX_CODE_LENGTH;
                baseCode = baseCode.substring(0, baseCode.length() - overflow - 2);
                finalCode = baseCode + "_" + suffix;
            }
        }

        return finalCode;
    }
}
```

### 2.3 多音字与特殊字符处理

```java
/**
 * 多音字处理策略
 */
@Component
public class PinyinDictionary {

    // 常用多音字映射（业务语境优先）
    private static final Map<String, String> POLYPHONE_MAP = Map.of(
        "订单", "dingdan",      // 不是 "dingdan"
        "角色", "juese",        // 不是 "jiaose"
        "行长", "hangzhang",    // 银行场景
        "重载", "chongzai",     // 编程场景
        "重构", "chonggou",
        "行内", "hangnei",      // 银行内部
        "还款", "huankuan",
        "重新", "chongxin"
    );

    /**
     * 智能分词+拼音转换
     */
    public String toPinyin(String chinese) {
        // 1. 优先匹配多音字词组
        for (Map.Entry<String, String> entry : POLYPHONE_MAP.entrySet()) {
            chinese = chinese.replace(entry.getKey(), entry.getValue());
        }

        // 2. 剩余部分使用HanLP分词+拼音
        List<Term> terms = HanLP.segment(chinese);
        StringBuilder pinyin = new StringBuilder();

        for (Term term : terms) {
            String word = term.word;
            if (POLYPHONE_MAP.containsKey(word)) {
                pinyin.append(POLYPHONE_MAP.get(word));
            } else {
                pinyin.append(PinyinHelper.toPinyin(word, PinyinStyle.NORMAL));
            }
        }

        return pinyin.toString();
    }
}

/**
 * 同义词/近义词标准化（提高一致性）
 */
@Component
public class SynonymNormalizer {

    // 同义词映射（保证相同含义生成相同code）
    private static final Map<String, String> SYNONYM_MAP = Map.ofEntries(
        Map.entry("新增", "create"),
        Map.entry("添加", "create"),
        Map.entry("新建", "create"),
        Map.entry("录入", "create"),
        Map.entry("登记", "create"),

        Map.entry("编辑", "edit"),
        Map.entry("修改", "edit"),
        Map.entry("更新", "edit"),
        Map.entry("变更", "edit"),

        Map.entry("删除", "delete"),
        Map.entry("移除", "delete"),
        Map.entry("废弃", "delete"),
        Map.entry("作废", "delete"),

        Map.entry("查询", "list"),
        Map.entry("列表", "list"),
        Map.entry("查看", "view"),
        Map.entry("浏览", "view"),
        Map.entry("详情", "detail"),

        Map.entry("导出", "export"),
        Map.entry("下载", "export"),
        Map.entry("导入", "import"),
        Map.entry("上传", "import"),

        Map.entry("审核", "approve"),
        Map.entry("审批", "approve"),
        Map.entry("核准", "approve"),
        Map.entry("批准", "approve"),

        Map.entry("提交", "submit"),
        Map.entry("保存", "save"),
        Map.entry("确认", "confirm")
    );

    public String normalize(String action) {
        return SYNONYM_MAP.getOrDefault(action, action);
    }
}
```

## 3. 各类型生成规则

### 3.1 页面权限生成

```java
/**
 * 页面权限码生成
 */
@Component
public class PagePermissionGenerator {

    @Autowired
    private PermissionCodeGenerator codeGenerator;

    /**
     * 从页面配置生成权限码
     */
    public String generate(PageDesignConfig config, String appId) {
        String entity = extractEntity(config.getPath());
        String action = extractAction(config.getName(), config.getPath());

        return codeGenerator.generate("page", entity, action,
            GenerateOptions.builder()
                .appId(appId)
                .sourceId(config.getId())
                .build());
    }

    /**
     * 从路由路径提取实体名
     * /orders -> order
     * /orders/:id -> order
     * /customer-management/list -> customer
     */
    private String extractEntity(String path) {
        if (StringUtils.isBlank(path)) {
            return "unnamed";
        }

        String[] segments = path.split("/");
        for (String segment : segments) {
            if (StringUtils.isBlank(segment) || segment.startsWith(":")) {
                continue;
            }
            // 取第一个有效段
            return segment.replaceAll("-", "_");
        }
        return "unnamed";
    }

    /**
     * 从页面名称提取动作
     */
    private String extractAction(String name, String path) {
        // 优先从名称识别
        if (name.contains("列表") || name.contains("查询")) {
            return "list";
        }
        if (name.contains("详情") || name.contains("明细")) {
            return "detail";
        }
        if (name.contains("表单") || name.contains("录入")) {
            return "form";
        }

        // 从路径推断
        if (path.contains("list") || path.endsWith("s")) {
            return "list";
        }
        if (path.contains("detail") || path.contains(":id")) {
            return "detail";
        }

        return "view";
    }
}
```

### 3.2 按钮权限生成

```java
/**
 * 按钮权限码生成
 */
@Component
public class ButtonPermissionGenerator {

    @Autowired
    private PermissionCodeGenerator codeGenerator;

    /**
     * 生成按钮权限码
     */
    public String generate(ButtonConfig config, PagePermission page, String appId) {
        // 从页面code提取实体
        String entity = extractEntityFromPage(page.getCode());

        // 从按钮名称提取动作
        String action = extractAction(config.getName(), config.getLocation());

        String baseCode = codeGenerator.generate("btn", entity, action,
            GenerateOptions.builder()
                .appId(appId)
                .sourceId(config.getId())
                .build());

        // 行操作按钮添加位置标识
        if (config.getLocation() == ButtonLocation.ROW) {
            baseCode = baseCode + "_row";
            // 重新检查冲突
            baseCode = resolveConflict(appId, baseCode);
        }

        return baseCode;
    }

    /**
     * 从页面code提取实体
     * page_order_list -> order
     */
    private String extractEntityFromPage(String pageCode) {
        if (pageCode.startsWith("page_")) {
            String[] parts = pageCode.split("_");
            if (parts.length >= 2) {
                return parts[1]; // 取实体部分
            }
        }
        return "unnamed";
    }

    /**
     * 提取动作词
     */
    private String extractAction(String name, ButtonLocation location) {
        String action = normalizeAction(name);

        // 行操作按钮特殊处理
        if (location == ButtonLocation.ROW) {
            if (name.contains("批量")) {
                action = "batch_" + action;
            }
        }

        return action;
    }

    private String normalizeAction(String name) {
        // 使用 SynonymNormalizer 处理
        return synonymNormalizer.normalize(name);
    }
}
```

### 3.3 字段权限生成

```java
/**
 * 字段权限码生成
 */
@Component
public class FieldPermissionGenerator {

    @Autowired
    private PermissionCodeGenerator codeGenerator;

    /**
     * 生成字段权限码
     */
    public String generate(FieldConfig config, PagePermission page, String appId) {
        String entity = extractEntityFromPage(page.getCode());
        String fieldName = normalizeFieldName(config.getName(), config.getFieldKey());

        return codeGenerator.generate("field", entity, fieldName,
            GenerateOptions.builder()
                .appId(appId)
                .sourceId(config.getId())
                .build());
    }

    /**
     * 字段名标准化
     */
    private String normalizeFieldName(String name, String fieldKey) {
        // 优先使用fieldKey（数据库字段名通常更规范）
        if (StringUtils.isNotBlank(fieldKey)) {
            return fieldKey.toLowerCase()
                .replaceAll("^is_", "")     // 移除布尔前缀
                .replaceAll("_id$", "");     // 移除ID后缀
        }

        // 从名称提取
        return codeGenerator.normalize(name);
    }
}
```

### 3.4 平台权限生成（IAM平台自管理）

平台权限用于管理低代码平台本身的功能，采用固定的 `sys_` 前缀，直接存入 `app_id = "_system"` 的命名空间。

```java
/**
 * 平台权限码生成器（IAM平台自身使用）
 */
@Component
public class PlatformPermissionGenerator {

    @Autowired
    private PermissionCodeGenerator codeGenerator;

    private static final String SYSTEM_APP_ID = "_system";

    /**
     * 生成平台权限码
     * 格式：sys_{module}_{resource}_{action}
     */
    public String generate(PlatformPermissionRequest request) {
        String prefix = "sys_" + request.getModule();
        String resource = normalizeResource(request.getResource());
        String action = normalizeAction(request.getAction());

        return codeGenerator.generate(prefix, resource, action,
            GenerateOptions.builder()
                .appId(SYSTEM_APP_ID)
                .sourceId(request.getId())
                .build());
    }

    /**
     * 预定义的平台权限（系统启动时初始化）
     */
    public List<PlatformPermissionDefinition> getPredefinedPermissions() {
        return List.of(
            // 应用管理
            PlatformPermissionDefinition.of("app", "create", "创建应用"),
            PlatformPermissionDefinition.of("app", "edit", "编辑应用"),
            PlatformPermissionDefinition.of("app", "delete", "删除应用"),
            PlatformPermissionDefinition.of("app", "publish", "发布应用"),
            PlatformPermissionDefinition.of("app", "config", "配置应用权限"),

            // 用户管理
            PlatformPermissionDefinition.of("user", "create", "创建用户"),
            PlatformPermissionDefinition.of("user", "edit", "编辑用户"),
            PlatformPermissionDefinition.of("user", "delete", "删除用户"),
            PlatformPermissionDefinition.of("user", "reset_password", "重置密码"),
            PlatformPermissionDefinition.of("user", "assign_role", "分配角色"),

            // 角色模板管理
            PlatformPermissionDefinition.of("role_tpl", "create", "创建角色模板"),
            PlatformPermissionDefinition.of("role_tpl", "edit", "编辑角色模板"),
            PlatformPermissionDefinition.of("role_tpl", "delete", "删除角色模板"),
            PlatformPermissionDefinition.of("role_tpl", "assign_perm", "分配权限"),

            // 数据维度管理
            PlatformPermissionDefinition.of("dimension", "create", "创建维度"),
            PlatformPermissionDefinition.of("dimension", "edit", "编辑维度"),
            PlatformPermissionDefinition.of("dimension", "delete", "删除维度"),

            // 审计日志
            PlatformPermissionDefinition.of("audit", "view", "查看审计日志"),
            PlatformPermissionDefinition.of("audit", "export", "导出审计日志"),

            // 平台配置
            PlatformPermissionDefinition.of("config", "modify", "修改平台配置"),
            PlatformPermissionDefinition.of("config", "view", "查看平台配置"),

            // 应用导入导出
            PlatformPermissionDefinition.of("import", "execute", "导入应用"),
            PlatformPermissionDefinition.of("export", "execute", "导出应用")
        );
    }

    private String normalizeResource(String resource) {
        // 标准化资源名
        return resource.toLowerCase()
            .replaceAll("[^a-z0-9]", "_")
            .replaceAll("_{2,}", "_");
    }

    private String normalizeAction(String action) {
        // 使用同义词标准化
        return SynonymNormalizer.normalize(action);
    }
}
```

#### 平台权限与应用权限的对比

| 维度 | 平台权限 (Platform) | 应用权限 (App) |
|------|---------------------|----------------|
| **app_id** | `_system` | `app_crm`, `app_erp` 等 |
| **前缀** | `sys_{module}` | `page`, `btn`, `field` |
| **管理界面** | IAM平台管理后台 | 各应用的权限管理页面 |
| **使用者** | 平台管理员 | 应用运营管理员 |
| **权限范围** | 管理所有应用 | 仅管理当前应用 |
| **数据来源** | 系统预定义 | 页面设计器自动生成 |
| **存储表** | `perm_functional` (app_id=`_system`) | `perm_functional` (app_id=具体值) |

#### 平台权限使用示例

```java
/**
 * 平台权限校验（IAM服务自管理）
 */
@Service
public class PlatformPermissionService {

    @Autowired
    private PermissionEngine permissionEngine;

    private static final String SYSTEM_APP_ID = "_system";

    /**
     * 校验当前用户是否有指定平台权限
     */
    public boolean hasPlatformPermission(String userId, String permissionCode) {
        // 平台权限使用固定的 _system app_id
        PermissionResult result = permissionEngine.checkFunctionalPermission(
            SYSTEM_APP_ID,
            userId,
            permissionCode,
            PermissionContext.empty()
        );
        return result.isAllowed();
    }

    /**
     * 校验用户是否是平台管理员
     */
    public boolean isPlatformAdmin(String userId) {
        return hasPlatformPermission(userId, "*"); // 超级管理员拥有所有权限
    }
}

/**
 * 平台权限注解（用于Controller）
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RequirePlatformPermission {
    String value(); // 权限码，如 "sys_app_create"
}

/**
 * 平台权限切面
 */
@Aspect
@Component
public class PlatformPermissionAspect {

    @Autowired
    private PlatformPermissionService permissionService;

    @Around("@annotation(requirePermission)")
    public Object around(ProceedingJoinPoint point,
                        RequirePlatformPermission requirePermission) throws Throwable {
        String userId = SecurityContext.getCurrentUserId();
        String permissionCode = requirePermission.value();

        if (!permissionService.hasPlatformPermission(userId, permissionCode)) {
            throw new PermissionDeniedException("缺少平台权限: " + permissionCode);
        }

        return point.proceed();
    }
}

// 使用示例
@RestController
@RequestMapping("/api/platform/apps")
public class AppManagementController {

    @PostMapping
    @RequirePlatformPermission("sys_app_create")
    public Result<App> createApp(@RequestBody CreateAppRequest request) {
        // 只有拥有 sys_app_create 权限的平台管理员才能创建应用
        return Result.success(appService.create(request));
    }

    @DeleteMapping("/{appId}")
    @RequirePlatformPermission("sys_app_delete")
    public Result<Void> deleteApp(@PathVariable String appId) {
        // 只有拥有 sys_app_delete 权限的平台管理员才能删除应用
        appService.delete(appId);
        return Result.success();
    }
}
```

## 4. 冲突处理策略

### 4.1 冲突检测流程

```java
/**
 * 权限码冲突检测器
 */
@Component
public class PermissionCodeConflictDetector {

    /**
     * 检测潜在冲突
     */
    public ConflictReport detect(String appId, List<PermissionItem> newItems) {
        ConflictReport report = new ConflictReport();

        // 1. 检测内部冲突（新列表内部重复）
        Map<String, List<PermissionItem>> internalDuplicates = newItems.stream()
            .collect(Collectors.groupingBy(PermissionItem::getCode));

        internalDuplicates.forEach((code, items) -> {
            if (items.size() > 1) {
                report.addInternalConflict(new InternalConflict(code, items));
            }
        });

        // 2. 检测外部冲突（与现有权限冲突）
        for (PermissionItem item : newItems) {
            if (permissionRepository.exists(appId, item.getCode())) {
                PermissionItem existing = permissionRepository.find(appId, item.getCode());
                report.addExternalConflict(new ExternalConflict(item, existing));
            }
        }

        // 3. 检测语义冲突（不同但相似的code）
        detectSemanticConflicts(appId, newItems, report);

        return report;
    }

    /**
     * 语义冲突检测（警告级别）
     */
    private void detectSemanticConflicts(String appId, List<PermissionItem> newItems,
                                         ConflictReport report) {
        // 获取所有现有权限码
        List<String> existingCodes = permissionRepository.findAllCodes(appId);

        for (PermissionItem item : newItems) {
            for (String existing : existingCodes) {
                // Levenshtein距离 < 2 视为相似
                if (calculateLevenshtein(item.getCode(), existing) <= 2 &&
                    !item.getCode().equals(existing)) {
                    report.addSemanticWarning(
                        new SemanticWarning(item.getCode(), existing));
                }
            }
        }
    }

    private int calculateLevenshtein(String s1, String s2) {
        // Levenshtein距离算法实现
        int[][] dp = new int[s1.length() + 1][s2.length() + 1];
        // ... 算法实现
        return dp[s1.length()][s2.length()];
    }
}
```

### 4.2 冲突解决策略

```java
/**
 * 冲突解决策略枚举
 */
public enum ConflictStrategy {
    AUTO_INCREMENT,      // 自动添加序号后缀
    PREFIX_WITH_PAGE,    // 添加页面前缀（按钮/字段）
    USE_HASH_SUFFIX,     // 使用短hash后缀
    MANUAL_RESOLVE,      // 用户手动指定
    SKIP                 // 跳过（保留现有）
}

/**
 * 冲突解决器
 */
@Component
public class PermissionCodeConflictResolver {

    /**
     * 自动解决冲突
     */
    public String resolve(String baseCode, ConflictStrategy strategy,
                         ResolutionContext context) {
        switch (strategy) {
            case AUTO_INCREMENT:
                return resolveWithIncrement(baseCode, context);

            case PREFIX_WITH_PAGE:
                return resolveWithPagePrefix(baseCode, context);

            case USE_HASH_SUFFIX:
                return resolveWithHash(baseCode, context);

            case MANUAL_RESOLVE:
                return context.getUserSpecifiedCode();

            case SKIP:
                return null; // 返回null表示跳过

            default:
                return resolveWithIncrement(baseCode, context);
        }
    }

    /**
     * 添加页面前缀
     * btn_edit -> btn_order_edit
     */
    private String resolveWithPagePrefix(String baseCode, ResolutionContext context) {
        String pagePrefix = extractPageEntity(context.getPageCode());
        String newCode = baseCode.replaceFirst("^btn_", "btn_" + pagePrefix + "_");
        return newCode;
    }

    /**
     * 使用短hash
     * btn_order_create -> btn_order_create_a7x2
     */
    private String resolveWithHash(String baseCode, ResolutionContext context) {
        String hash = DigestUtils.md5Hex(context.getSourceId())
            .substring(0, 4);
        return baseCode + "_" + hash;
    }
}
```

## 5. 生成事件与监听

### 5.1 生成事件

```java
/**
 * 权限码生成事件
 */
public class PermissionCodeGeneratedEvent {
    private String appId;
    private String permissionCode;
    private PermissionType type;
    private String sourceId;          // 来源对象ID（页面ID、按钮ID等）
    private String sourceName;        // 来源对象名称
    private String generationRule;    // 使用的生成规则
    private boolean hasConflict;      // 是否发生过冲突
    private String conflictStrategy;  // 冲突解决策略
    private LocalDateTime generatedAt;
}

/**
 * 生成监听器
 */
@Component
public class PermissionCodeGenerationListener {

    @EventListener
    public void onPermissionCodeGenerated(PermissionCodeGeneratedEvent event) {
        // 1. 记录生成日志（审计）
        auditService.logCodeGeneration(event);

        // 2. 如果发生过冲突，发送警告
        if (event.isHasConflict()) {
            notificationService.sendConflictWarning(event);
        }

        // 3. 缓存新权限码（用于快速冲突检测）
        codeCache.put(event.getAppId(), event.getPermissionCode());
    }
}
```

## 6. 配置与扩展

### 6.1 生成规则配置

```yaml
# permission-code-generator.yml
generator:
  # 全局配置
  max-code-length: 64
  max-entity-length: 20
  max-action-length: 30

  # 多音字词典（支持热加载）
  polyphone-dictionary:
    path: "classpath:polyphone-dict.txt"
    hot-reload: true
    reload-interval: 3600  # 秒

  # 同义词映射
  synonym:
    create: ["新增", "添加", "新建", "录入"]
    edit: ["编辑", "修改", "更新", "变更"]
    delete: ["删除", "移除", "废弃"]
    list: ["查询", "列表", "浏览"]

  # 冲突解决策略优先级
  conflict-strategies:
    button: [PREFIX_WITH_PAGE, AUTO_INCREMENT]
    field: [PREFIX_WITH_PAGE, AUTO_INCREMENT]
    page: [AUTO_INCREMENT]

  # 保留词（禁止使用的code）
  reserved-words:
    - "admin"
    - "root"
    - "system"
    - "public"
    - "private"
    - "all"
    - "*"
    - "null"
    - "undefined"
```

### 6.2 自定义生成规则

```java
/**
 * 自定义生成规则接口
 */
public interface CustomCodeGenerator {
    boolean supports(PermissionType type, String appId);
    String generate(GenerateContext context);
}

/**
 * 示例：电商业务自定义生成规则
 */
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class EcommerceCodeGenerator implements CustomCodeGenerator {

    @Override
    public boolean supports(PermissionType type, String appId) {
        // 只对特定应用生效
        return appId.startsWith("app_ec_");
    }

    @Override
    public String generate(GenerateContext context) {
        // 电商业务特殊处理
        if (context.getType() == PermissionType.BUTTON) {
            ButtonConfig btn = (ButtonConfig) context.getSource();
            if (btn.getName().contains("上架")) {
                return "btn_" + context.getEntity() + "_shelve";
            }
            if (btn.getName().contains("下架")) {
                return "btn_" + context.getEntity() + "_unshelve";
            }
        }
        return null; // 返回null表示使用默认生成器
    }
}
```

## 7. 使用示例

### 7.1 页面设计器集成

```typescript
// PermissionCodeService.ts
export class PermissionCodeService {

  /**
   * 实时生成权限码预览
   */
  async previewPageCode(pageName: string, path: string): Promise<CodePreview> {
    const response = await fetch('/api/permission/code/preview', {
      method: 'POST',
      body: JSON.stringify({
        type: 'page',
        name: pageName,
        path: path
      })
    });
    return response.json();
  }

  /**
   * 批量生成页面及其所有按钮、字段的权限码
   */
  async generatePagePermissions(pageId: string): Promise<PermissionSet> {
    const response = await fetch(`/api/permission/code/generate/page/${pageId}`, {
      method: 'POST'
    });
    return response.json();
  }
}

// 页面设计器中使用
interface CodePreviewProps {
  pageName: string;
  path: string;
}

const CodePreview: React.FC<CodePreviewProps> = ({ pageName, path }) => {
  const [preview, setPreview] = useState<CodePreview | null>(null);

  useEffect(() => {
    PermissionCodeService.previewPageCode(pageName, path)
      .then(setPreview);
  }, [pageName, path]);

  return (
    <div className="code-preview">
      <div className="preview-item">
        <span className="label">页面权限码：</span>
        <code>{preview?.pageCode}</code>
        {preview?.hasConflict && (
          <Tag color="warning">已自动解决冲突</Tag>
        )}
      </div>
      <div className="preview-list">
        <span className="label">预计生成按钮权限码：</span>
        {preview?.buttonCodes?.map(btn => (
          <div key={btn.code} className="preview-subitem">
            <code>{btn.code}</code>
            <span className="desc">{btn.name}</span>
          </div>
        ))}
      </div>
    </div>
  );
};
```

### 7.2 批量导入场景

```java
/**
 * 批量导入时的code生成
 */
@Service
public class BatchImportService {

    @Transactional
    public ImportResult importWithCodeGeneration(AppImportPackage pkg, String targetAppId) {
        ImportContext context = new ImportContext(targetAppId);

        // 1. 收集所有需要生成code的项
        List<CodeGenerationTask> tasks = collectGenerationTasks(pkg);

        // 2. 批量预生成code（不保存到数据库）
        Map<String, String> generatedCodes = new HashMap<>();
        for (CodeGenerationTask task : tasks) {
            String code = codeGenerator.generate(task.getType(),
                task.getEntity(), task.getAction(),
                GenerateOptions.builder()
                    .appId(targetAppId)
                    .conflictStrategy(ConflictStrategy.AUTO_INCREMENT)
                    .build());
            generatedCodes.put(task.getSourceId(), code);
        }

        // 3. 显示预览，让用户确认
        CodePreview preview = new CodePreview(generatedCodes);

        // 4. 用户确认后保存
        return saveWithCodes(pkg, generatedCodes);
    }
}
```

## 8. 校验与测试

### 8.1 Code格式校验

```java
/**
 * 权限码格式校验器
 */
@Component
public class PermissionCodeValidator {

    /**
     * 应用权限：page_*, btn_*, field_*
     * 平台权限：sys_{module}_*
     */
    private static final Pattern CODE_PATTERN = Pattern.compile(
        "^((page|btn|field)|sys_[a-z]+)_[a-z][a-z0-9_]*$"
    );

    private static final int MAX_LENGTH = 64;

    public ValidationResult validate(String code) {
        List<String> errors = new ArrayList<>();

        if (StringUtils.isBlank(code)) {
            errors.add("权限码不能为空");
        } else {
            // 长度校验
            if (code.length() > MAX_LENGTH) {
                errors.add(String.format("权限码长度不能超过%d字符", MAX_LENGTH));
            }

            // 格式校验
            if (!CODE_PATTERN.matcher(code).matches()) {
                errors.add("权限码格式错误，应用权限应为：page|btn|field_实体_动作；" +
                          "平台权限应为：sys_模块_资源_动作");
            }

            // 连续下划线校验
            if (code.contains("__")) {
                errors.add("权限码不能包含连续下划线");
            }

            // 保留词校验
            String[] parts = code.split("_");
            for (String part : parts) {
                if (ReservedWords.contains(part)) {
                    errors.add("权限码包含保留词：" + part);
                }
            }
        }

        return errors.isEmpty()
            ? ValidationResult.success()
            : ValidationResult.fail(errors);
    }

    /**
     * 判断是否为平台权限
     */
    public boolean isPlatformPermission(String code) {
        return code != null && code.startsWith("sys_");
    }

    /**
     * 判断是否为应用权限
     */
    public boolean isAppPermission(String code) {
        return code != null && (
            code.startsWith("page_") ||
            code.startsWith("btn_") ||
            code.startsWith("field_")
        );
    }
}
```

### 8.2 单元测试

```java
@SpringBootTest
class PermissionCodeGeneratorTest {

    @Autowired
    private PermissionCodeGenerator generator;

    @Test
    void testBasicGeneration() {
        String code = generator.generate("page", "订单", "列表",
            GenerateOptions.builder().appId("app_test").build());

        assertEquals("page_dingdan_list", code);
    }

    @Test
    void testPolyphone() {
        // 测试多音字
        String code1 = generator.generate("btn", "订单", "新建",
            GenerateOptions.builder().appId("app_test").build());
        assertEquals("btn_dingdan_create", code1);

        String code2 = generator.generate("btn", "角色", "编辑",
            GenerateOptions.builder().appId("app_test").build());
        assertEquals("btn_juese_edit", code2);
    }

    @Test
    void testConflictResolution() {
        // 先插入一个权限码
        permissionRepository.save("app_conflict", "btn_order_create", ...);

        // 再次生成相同code
        String code = generator.generate("btn", "订单", "新建",
            GenerateOptions.builder()
                .appId("app_conflict")
                .conflictStrategy(ConflictStrategy.AUTO_INCREMENT)
                .build());

        assertEquals("btn_order_create_1", code);
    }

    @Test
    void testTruncate() {
        // 测试超长名称截断
        String code = generator.generate("field",
            "非常非常长的实体名称超过了二十个字符",
            "非常非常长的字段名称超过了三十个字符",
            GenerateOptions.builder().appId("app_test").build());

        assertTrue(code.length() <= 64);
        assertTrue(code.startsWith("field_"));
    }
}
```
