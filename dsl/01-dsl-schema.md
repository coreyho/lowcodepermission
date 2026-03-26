# DSL Schema 定义

## 1. DSL 版本说明

| 版本 | 日期 | 说明 |
|------|------|------|
| 1.0 | 2026-03-20 | 初始版本 |

## 2. 配置期 DSL Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Permission Config DSL Schema",
  "type": "object",
  "required": ["version", "app_id"],
  "properties": {
    "version": {
      "type": "string",
      "enum": ["1.0"],
      "description": "DSL版本号"
    },
    "app_id": {
      "type": "string",
      "pattern": "^app_[a-zA-Z0-9]+$",
      "description": "应用唯一标识"
    },
    
    "functional_permissions": {
      "type": "object",
      "description": "功能权限定义",
      "properties": {
        "pages": {
          "type": "array",
          "description": "页面级权限",
          "items": {
            "type": "object",
            "required": ["code", "name", "type"],
            "properties": {
              "code": { 
                "type": "string",
                "description": "权限码，全局唯一"
              },
              "name": { 
                "type": "string",
                "description": "显示名称"
              },
              "type": { 
                "type": "string",
                "enum": ["page", "menu"],
                "description": "页面类型"
              },
              "path": { 
                "type": "string",
                "description": "路由路径"
              },
              "icon": { 
                "type": "string",
                "description": "图标标识"
              },
              "hidden": { 
                "type": "boolean",
                "default": false,
                "description": "是否在菜单隐藏"
              },
              "default_visible": { 
                "type": "boolean",
                "default": true,
                "description": "默认是否可见"
              },
              "children": { 
                "type": "array",
                "description": "子页面列表"
              }
            }
          }
        },
        
        "buttons": {
          "type": "array",
          "description": "按钮级权限",
          "items": {
            "type": "object",
            "required": ["code", "name", "page_code"],
            "properties": {
              "code": { 
                "type": "string",
                "description": "按钮权限码"
              },
              "name": { 
                "type": "string",
                "description": "按钮名称"
              },
              "page_code": { 
                "type": "string",
                "description": "所属页面权限码"
              },
              "location": {
                "type": "string",
                "enum": ["toolbar", "row", "form"],
                "description": "按钮位置"
              },
              "danger": { 
                "type": "boolean",
                "default": false,
                "description": "是否危险操作"
              }
            }
          }
        },
        
        "fields": {
          "type": "array",
          "description": "字段级权限",
          "items": {
            "type": "object",
            "required": ["code", "name", "page_code"],
            "properties": {
              "code": { 
                "type": "string",
                "description": "字段权限码"
              },
              "name": { 
                "type": "string",
                "description": "字段名称"
              },
              "page_code": { 
                "type": "string",
                "description": "所属页面权限码"
              },
              "data_masking": {
                "type": "object",
                "description": "数据脱敏配置",
                "properties": {
                  "enable": { 
                    "type": "boolean",
                    "description": "是否启用脱敏"
                  },
                  "type": { 
                    "type": "string",
                    "enum": ["partial", "full", "regex"],
                    "description": "脱敏类型"
                  },
                  "pattern": { 
                    "type": "string",
                    "description": "脱敏规则（正则或模式）"
                  }
                }
              }
            }
          }
        },
        
        "apis": {
          "type": "array",
          "description": "API级权限",
          "items": {
            "type": "object",
            "required": ["code", "method", "path"],
            "properties": {
              "code": { 
                "type": "string",
                "description": "API权限码"
              },
              "method": { 
                "type": "string",
                "enum": ["GET", "POST", "PUT", "DELETE", "PATCH"],
                "description": "HTTP方法"
              },
              "path": { 
                "type": "string",
                "description": "API路径"
              },
              "auto_generated": { 
                "type": "boolean",
                "default": false,
                "description": "是否自动生成"
              }
            }
          }
        }
      }
    },
    
    "data_permissions": {
      "type": "object",
      "description": "数据权限定义",
      "properties": {
        "dimensions": {
          "type": "array",
          "description": "数据权限维度",
          "items": {
            "type": "object",
            "required": ["code", "name", "type"],
            "properties": {
              "code": {
                "type": "string",
                "description": "维度编码"
              },
              "name": {
                "type": "string",
                "description": "维度名称"
              },
              "type": {
                "type": "string",
                "enum": ["organization", "department", "owner", "custom"],
                "description": "维度类型"
              },
              "entity_field": {
                "type": "string",
                "description": "实体字段名"
              },
              "source_type": {
                "type": "string",
                "enum": ["table", "enum", "api"],
                "description": "数据源类型"
              },
              "source_config": {
                "type": "object",
                "description": "数据源配置"
              }
            }
          }
        },
        
        "rule_templates": {
          "type": "array",
          "description": "数据规则模板",
          "items": {
            "type": "object",
            "required": ["code", "name"],
            "properties": {
              "code": {
                "type": "string",
                "description": "模板编码"
              },
              "name": {
                "type": "string",
                "description": "模板名称"
              },
              "expression": {
                "type": "string",
                "description": "规则表达式"
              },
              "param_schema": {
                "type": "object",
                "description": "参数定义（JSON Schema格式）"
              },
              "example_params": {
                "type": "object",
                "description": "示例参数"
              },
              "description": {
                "type": "string",
                "description": "模板说明"
              }
            }
          }
        }
      }
    },
    
    "role_templates": {
      "type": "array",
      "description": "角色模板定义",
      "items": {
        "type": "object",
        "required": ["code", "name"],
        "properties": {
          "code": { 
            "type": "string",
            "description": "角色编码"
          },
          "name": { 
            "type": "string",
            "description": "角色名称"
          },
          "type": {
            "type": "string",
            "enum": ["system", "template"],
            "default": "template",
            "description": "角色类型"
          },
          "description": { 
            "type": "string",
            "description": "角色描述"
          },
          "permissions": {
            "type": "object",
            "description": "权限配置",
            "properties": {
              "functional": {
                "type": "array",
                "items": { 
                  "type": "string",
                  "description": "功能权限码"
                }
              },
              "data": {
                "type": "object",
                "description": "数据权限配置",
                "properties": {
                  "strategy": {
                    "type": "string",
                    "enum": ["all", "rule_based", "custom"],
                    "description": "数据权限策略"
                  },
                  "rules": { 
                    "type": "array",
                    "description": "数据规则列表"
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

## 3. 运行时 DSL Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Runtime Permission DSL Schema",
  "type": "object",
  "required": ["app_id", "version"],
  "properties": {
    "app_id": {
      "type": "string",
      "description": "应用ID"
    },
    "version": {
      "type": "string",
      "description": "配置版本"
    },
    
    "roles": {
      "type": "array",
      "description": "角色定义",
      "items": {
        "type": "object",
        "required": ["id", "code", "name"],
        "properties": {
          "id": { 
            "type": "string",
            "description": "角色唯一ID"
          },
          "code": { 
            "type": "string",
            "description": "角色编码"
          },
          "name": { 
            "type": "string",
            "description": "角色名称"
          },
          "source": { 
            "type": "string",
            "description": "来源：template:xxx | custom"
          },
          "extends": {
            "type": "object",
            "description": "继承扩展配置",
            "properties": {
              "functional": {
                "type": "object",
                "properties": {
                  "add": {
                    "type": "array",
                    "items": { "type": "string" }
                  },
                  "remove": {
                    "type": "array",
                    "items": { "type": "string" }
                  }
                }
              }
            }
          },
          "permissions": {
            "type": "object",
            "description": "权限配置（自定义角色时使用）"
          }
        }
      }
    },
    
    "user_roles": {
      "type": "array",
      "description": "用户-角色绑定",
      "items": {
        "type": "object",
        "required": ["user_id", "roles"],
        "properties": {
          "user_id": { 
            "type": "string",
            "description": "用户ID"
          },
          "username": { 
            "type": "string",
            "description": "用户名"
          },
          "roles": {
            "type": "array",
            "items": { "type": "string" },
            "description": "角色ID列表"
          },
          "effective_time": {
            "type": "string",
            "format": "date-time",
            "description": "生效时间"
          },
          "expire_time": {
            "type": "string",
            "format": "date-time",
            "description": "过期时间"
          }
        }
      }
    },
    
    "dept_roles": {
      "type": "array",
      "description": "部门-角色绑定",
      "items": {
        "type": "object",
        "properties": {
          "dept_id": { 
            "type": "string",
            "description": "部门ID"
          },
          "default_roles": {
            "type": "array",
            "items": { "type": "string" },
            "description": "默认角色列表"
          }
        }
      }
    },
    
    "data_rules": {
      "type": "array",
      "description": "数据规则实例",
      "items": {
        "type": "object",
        "required": ["id", "name", "entity"],
        "properties": {
          "id": { 
            "type": "string",
            "description": "规则ID"
          },
          "name": { 
            "type": "string",
            "description": "规则名称"
          },
          "entity": { 
            "type": "string",
            "description": "实体编码"
          },
          "type": {
            "type": "string",
            "enum": ["template", "field_based", "custom"],
            "description": "规则类型"
          },
          "expression": { 
            "type": "string",
            "description": "规则表达式"
          },
          "conditions": {
            "type": "array",
            "description": "字段条件列表",
            "items": {
              "type": "object",
              "properties": {
                "field": { "type": "string" },
                "operator": { 
                  "type": "string",
                  "enum": ["EQ", "NE", "GT", "GTE", "LT", "LTE", "IN", "NOT_IN", "LIKE", "BETWEEN"]
                },
                "value": {},
                "logic": {
                  "type": "string",
                  "enum": ["AND", "OR"],
                  "default": "AND"
                }
              }
            }
          },
          "parameters": {
            "type": "object",
            "description": "规则参数"
          }
        }
      }
    },
    
    "special_permissions": {
      "type": "object",
      "description": "特殊权限配置",
      "properties": {
        "masking_exemptions": {
          "type": "array",
          "description": "脱敏白名单",
          "items": {
            "type": "object",
            "properties": {
              "user_id": { "type": "string" },
              "fields": {
                "type": "array",
                "items": { "type": "string" }
              }
            }
          }
        },
        "rate_limits": {
          "type": "array",
          "description": "API限流配置",
          "items": {
            "type": "object",
            "properties": {
              "role": { "type": "string" },
              "api": { "type": "string" },
              "limit": { "type": "number" },
              "window": { "type": "string" }
            }
          }
        }
      }
    }
  }
}
```

## 4. 校验工具

### 4.1 Java 校验示例

```java
@Component
public class DSLValidator {
    
    private final JsonSchema configSchema;
    private final JsonSchema runtimeSchema;
    
    public DSLValidator() throws IOException {
        ObjectMapper mapper = new ObjectMapper();
        JsonSchemaFactory factory = JsonSchemaFactory.getInstance(SpecVersion.VersionFlag.V7);
        
        // 加载 Schema
        InputStream configStream = getClass().getResourceAsStream("/schema/config-dsl-schema.json");
        this.configSchema = factory.getSchema(configStream);
        
        InputStream runtimeStream = getClass().getResourceAsStream("/schema/runtime-dsl-schema.json");
        this.runtimeSchema = factory.getSchema(runtimeStream);
    }
    
    public ValidationResult validateConfigDSL(String dslContent) {
        try {
            ObjectMapper mapper = new ObjectMapper();
            JsonNode jsonNode = mapper.readTree(dslContent);
            
            Set<ValidationMessage> errors = configSchema.validate(jsonNode);
            
            if (errors.isEmpty()) {
                return ValidationResult.success();
            } else {
                List<String> errorMessages = errors.stream()
                    .map(ValidationMessage::getMessage)
                    .collect(Collectors.toList());
                return ValidationResult.fail(errorMessages);
            }
        } catch (IOException e) {
            return ValidationResult.fail(Collections.singletonList("JSON解析失败: " + e.getMessage()));
        }
    }
    
    public ValidationResult validateRuntimeDSL(String dslContent) {
        // 类似实现...
    }
}
```
