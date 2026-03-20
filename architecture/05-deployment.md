# 部署架构

## 1. K8s 部署架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              K8s Cluster                               │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     Ingress Controller                           │   │
│  │              (路由分发: /designer/* 和 /app/{id}/*)              │   │
│  │                                                                  │   │
│  │   域名: designer.lowcode-platform.com                           │   │
│  │         app-{id}.lowcode-platform.com                           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                          │
│  ┌─────────────────────────┐    ┌─────────────────────────────────┐    │
│  │   页面设计器服务         │    │        应用运行服务群            │    │
│  │   (Designer Service)    │    │   (多个应用实例，独立域名)        │    │
│  │   Replicas: 3           │    │                                 │    │
│  │                         │    │  ┌─────────┐ ┌─────────┐       │    │
│  │  - 权限配置面板          │    │  │  App-1  │ │  App-2  │ ...   │    │
│  │  - DSL生成器            │    │  │ (Pod×2) │ │ (Pod×2) │       │    │
│  │  - 配置存储             │    │  └─────────┘ └─────────┘       │    │
│  │                         │    │       ↓                         │    │
│  │   ConfigMap:            │    │  ┌─────────────────────────┐    │    │
│  │   - permission-dsl      │    │  │   运行时权限引擎        │    │    │
│  └─────────────────────────┘    │  │   (Sidecar模式)         │    │    │
│                                 │  │   - 权限校验API         │    │    │
│  ┌─────────────────────────┐    │  │   - 数据权限拦截器       │    │    │
│  │   权限服务集群           │    │  │   - 脱敏处理器          │    │    │
│  │   (Permission Service)  │    │  └─────────────────────────┘    │    │
│  │   Replicas: 3           │    └─────────────────────────────────┘    │
│  │   Service: permission   │                                           │
│  │                         │                                           │
│  │  - 权限管理API          │                                           │
│  │  - 角色管理API          │                                           │
│  │  - DSL解析器            │                                           │
│  │  - 规则引擎             │                                           │
│  └─────────────────────────┘                                           │
│           ↓                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                         共享存储层                               │   │
│  │                                                                  │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │   │
│  │  │  MySQL      │  │  Redis      │  │  Elasticsearch          │  │   │
│  │  │  (主从集群)  │  │  (哨兵模式)  │  │  (3节点集群)             │  │   │
│  │  │  权限配置    │  │  权限缓存    │  │  审计日志                │  │   │
│  │  │  StatefulSet│  │  StatefulSet│  │  StatefulSet            │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────────────┘  │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## 2. 服务拆分

### 2.1 权限服务 (permission-service)

| 功能模块 | 说明 | API 路径 |
|----------|------|----------|
| 权限管理 | 权限配置 CRUD | `/api/v1/permissions/**` |
| 角色管理 | 角色配置 CRUD | `/api/v1/roles/**` |
| 数据规则 | 数据权限规则管理 | `/api/v1/data-rules/**` |
| 权限校验 | 运行时权限校验 | `/api/v1/check/**` |
| DSL 解析 | 权限 DSL 解析 | `/api/v1/dsl/**` |

### 2.2 应用运行时 (app-runtime)

| 组件 | 部署方式 | 说明 |
|------|----------|------|
| 业务应用 | Deployment | 用户发布的应用 |
| 权限 Sidecar | Sidecar | 与应用同 Pod 部署 |
| 配置初始化 | InitContainer | 加载权限配置 |

## 3. 配置文件

### 3.1 权限服务配置

```yaml
# permission-service-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: permission-service-config
data:
  application.yml: |
    server:
      port: 8080
    
    spring:
      datasource:
        url: jdbc:mysql://mysql-cluster:3306/permission?useUnicode=true&characterEncoding=utf8
        username: ${DB_USERNAME}
        password: ${DB_PASSWORD}
      
      redis:
        sentinel:
          master: mymaster
          nodes: redis-sentinel-0:26379,redis-sentinel-1:26379,redis-sentinel-2:26379
        password: ${REDIS_PASSWORD}
      
      elasticsearch:
        uris: http://elasticsearch-cluster:9200
    
    permission:
      cache:
        user-ttl: 1800  # 30分钟
        app-ttl: 3600   # 1小时
      
      engine:
        rule-parser: qlexpress
        sql-parser: jsqlparser
      
      audit:
        enabled: true
        storage: elasticsearch
        retention-days: 90
```

### 3.2 Sidecar 配置

```yaml
# permission-sidecar.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-runtime-{appId}
spec:
  replicas: 2
  template:
    spec:
      initContainers:
        - name: permission-init
          image: permission-sidecar:latest
          command: ['sh', '-c', 'permission-init --app-id={{appId}}']
          volumeMounts:
            - name: permission-config
              mountPath: /etc/permission
      
      containers:
        - name: app
          image: app-runtime:latest
          volumeMounts:
            - name: permission-config
              mountPath: /etc/permission
              readOnly: true
        
        - name: permission-sidecar
          image: permission-sidecar:latest
          env:
            - name: APP_ID
              value: "{{appId}}"
            - name: PERMISSION_SERVICE_URL
              value: "http://permission-service:8080"
      
      volumes:
        - name: permission-config
          emptyDir: {}
```

## 4. 网络策略

### 4.1 Service 定义

```yaml
# permission-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: permission-service
spec:
  selector:
    app: permission-service
  ports:
    - port: 8080
      targetPort: 8080
  type: ClusterIP

---
# permission-service-headless.yaml
apiVersion: v1
kind: Service
metadata:
  name: permission-service-headless
spec:
  selector:
    app: permission-service
  ports:
    - port: 8080
  clusterIP: None
```

### 4.2 Ingress 配置

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lowcode-platform-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: designer.lowcode-platform.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: designer-service
                port:
                  number: 80
    
    - host: "app-*.lowcode-platform.com"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-runtime-service
                port:
                  number: 80
```

## 5. 资源限制

### 5.1 资源配额

```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: permission-quota
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    persistentvolumeclaims: "5"
```

### 5.2 服务资源限制

| 服务 | CPU Request | CPU Limit | Memory Request | Memory Limit |
|------|-------------|-----------|----------------|--------------|
| permission-service | 500m | 2000m | 1Gi | 4Gi |
| designer-service | 300m | 1000m | 512Mi | 2Gi |
| app-runtime | 200m | 500m | 256Mi | 1Gi |
| permission-sidecar | 100m | 200m | 128Mi | 256Mi |

## 6. 监控与日志

### 6.1 Prometheus 监控

```yaml
# service-monitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: permission-metrics
spec:
  selector:
    matchLabels:
      app: permission-service
  endpoints:
    - port: metrics
      interval: 30s
      path: /actuator/prometheus
```

### 6.2 关键指标

| 指标 | 说明 | 告警阈值 |
|------|------|----------|
| `permission_check_total` | 权限校验次数 | - |
| `permission_check_duration_seconds` | 校验耗时 | P99 > 100ms |
| `permission_cache_hit_ratio` | 缓存命中率 | < 80% |
| `permission_audit_log_queue_size` | 审计日志队列 | > 1000 |

### 6.3 日志收集

```yaml
# fluentd-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/containers/permission-*.log
      pos_file /var/log/fluent/permission.log.pos
      tag permission.*
      <parse>
        @type json
      </parse>
    </source>
    
    <match permission.**>
      @type elasticsearch
      host elasticsearch-cluster
      logstash_format true
      logstash_prefix permission
    </match>
```
