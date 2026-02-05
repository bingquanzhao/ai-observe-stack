# DOGStack Helm Chart

[English](./README.md)

**DOGStack**（Doris OpenTelemetry Grafana with Doris App Plugin）是一个完整的可观测性技术栈，使用 Apache Doris 作为 Traces、Metrics 和 Logs 的存储后端。

默认情况下，Helm Chart 会部署所有核心组件，包括：

- **Apache Doris**（通过 Doris Operator）
- **OpenTelemetry Collector**（带 Doris Exporter）
- **Grafana**（带 Doris App Plugin）

同时，也可以轻松配置为连接到现有的 Doris 集群。

Chart 支持 Kubernetes 标准最佳实践，包括：

- 通过 `values.yaml` 进行环境特定配置
- 资源限制和 Pod 级别扩展
- TLS 和 Ingress 配置
- Secrets 管理和认证设置

## 适用场景

- 概念验证（POC）
- 开发环境
- 生产部署

---

## 前置要求

- Kubernetes 1.20+
- Helm 3.0+
- `kubectl` 已配置并能与集群交互
- PV Provisioner 支持（用于持久化存储）

---

## 部署步骤

### 1. 下载并解压 DOGStack

```bash
wget https://justtmp-1308700295.cos.ap-hongkong.myqcloud.com/DOG-k8s.tar.gz
tar -xzf DOG-k8s.tar.gz
cd DOG-k8s
```

### 2. 添加 Helm 仓库

添加 Doris Operator 所需的 Helm 仓库：

```bash
helm repo add doris-repo https://charts.selectdb.com
helm repo update
```

### 3. 创建命名空间

```bash
kubectl create namespace dogstack
```

### 4. 构建并加载插件镜像（必需）

DOGStack 使用自定义的 Grafana 插件镜像，插件会在构建时从 S3/URL 自动下载：

```bash
cd images/doris-plugin

# 构建镜像（插件自动从默认 URL 下载）
docker build -t dog-grafana-assets:v1.0.1 .

# 或指定自定义插件 URL
docker build -t dog-grafana-assets:v1.0.1 \
  --build-arg DORIS_APP_URL=https://your-s3-bucket/doris-app-1.0.0.zip .

# Kind 集群加载镜像
kind load docker-image dog-grafana-assets:v1.0.1 --name kind

# 其他集群，推送到镜像仓库
# docker tag dog-grafana-assets:v1.0.1 your-registry/dog-grafana-assets:v1.0.1
# docker push your-registry/dog-grafana-assets:v1.0.1
```

### 5. 更新依赖

```bash
cd ../..
helm dependency update .
```

### 6. 安装 DOGStack

使用默认配置安装（内部 Doris 模式）：

```bash
helm install dogstack . -n dogstack \
  --set dorisPlugin.image.tag=v1.0.1
```

---

## 验证安装

检查所有 Pod 是否正常运行：

```bash
kubectl get pods -n dogstack
```

预期输出：

```
NAME                                READY   STATUS    RESTARTS   AGE
dogstack-doris-fe-0                 1/1     Running   0          2m
dogstack-doris-be-0                 1/1     Running   0          1m
dogstack-grafana-xxx                1/1     Running   0          2m
dogstack-otel-collector-0           1/1     Running   0          2m
dogstack-otel-collector-1           1/1     Running   0          2m
doris-operator-xxx                  1/1     Running   0          2m
```

检查 DorisCluster 状态：

```bash
kubectl get doriscluster -n dogstack
```

---

## 端口转发

端口转发可以让你访问和配置 DOGStack。生产环境建议配置 Ingress。

### 访问 Grafana

```bash
kubectl port-forward svc/dogstack-grafana 3000:3000 -n dogstack --address 0.0.0.0
```

访问 http://localhost:3000（或 http://服务器IP:3000）

**默认凭据：**
- 用户名：`admin`
- 密码：`admin`

### 访问 OTel Collector

```bash
kubectl port-forward svc/dogstack-otel-collector 4317:4317 4318:4318 -n dogstack --address 0.0.0.0
```

发送遥测数据到：
- OTLP gRPC：`localhost:4317`
- OTLP HTTP：`localhost:4318`

### 访问 Doris（内部模式）

```bash
# MySQL 协议
kubectl port-forward svc/dogstack-doris-fe-service 9030:9030 -n dogstack

# 通过 MySQL 客户端连接
mysql -h 127.0.0.1 -P 9030 -u root

# Web UI
kubectl port-forward svc/dogstack-doris-fe-service 8030:8030 -n dogstack
```

---

## 自定义配置

可以使用 `--set` 参数自定义设置：

```bash
helm install dogstack . -n dogstack \
  --set dorisPlugin.image.tag=v1.0.1 \
  --set grafana.adminPassword=mysecretpassword
```

或创建自定义 `values.yaml`：

```bash
# 获取默认配置
helm show values . > my-values.yaml

# 编辑后安装
helm install dogstack . -n dogstack -f my-values.yaml
```

### 主要配置参数

| 参数 | 描述 | 默认值 |
|------|------|--------|
| `doris.mode` | Doris 部署模式（`internal` / `external`） | `internal` |
| `doris.database` | 可观测性数据存储的数据库名 | `otel` |
| `doris.internal.cluster.fe.replicas` | FE 副本数 | `1` |
| `doris.internal.cluster.be.replicas` | BE 副本数 | `1` |
| `otel.enabled` | 启用 OpenTelemetry Collector | `true` |
| `otel.replicas` | OTel Collector 副本数 | `2` |
| `grafana.enabled` | 启用 Grafana | `true` |
| `grafana.adminPassword` | Grafana 管理员密码 | `admin` |
| `dorisPlugin.enabled` | 启用 Doris App 插件 | `true` |
| `ingress.enabled` | 启用 Ingress | `false` |

完整参数列表请参考 [values.yaml](./values.yaml)。

---

## 使用 Secrets

对于敏感数据（如数据库凭据），建议使用 Kubernetes Secrets。

### 创建 Secret

```bash
kubectl create secret generic doris-credentials \
  --from-literal=username=root \
  --from-literal=password=mysecretpassword \
  -n dogstack
```

### 在 values.yaml 中引用

```yaml
doris:
  mode: external
  external:
    host: "my-doris-fe.example.com"
    existingSecret: "doris-credentials"
    userKey: "username"
    passwordKey: "password"
```

---

## 使用外部 Doris

如果使用现有的 Doris 集群，需要禁用内部 Doris 并指定外部连接信息：

```bash
helm install dogstack . -n dogstack \
  --set doris.mode=external \
  --set doris.external.host=172.19.0.12 \
  --set doris.external.port=9030 \
  --set doris.external.feHttpPort=8030 \
  --set doris.external.beHttpPort=8040 \
  --set doris.external.user=root \
  --set doris.external.password="" \
  --set doris.internal.operator.enabled=false \
  --set dorisPlugin.image.tag=v1.0.1
```

或使用 `values.yaml` 文件：

```yaml
doris:
  mode: external
  database: otel
  external:
    host: "172.19.0.12"
    port: 9030
    feHttpPort: 8030
    beHttpPort: 8040
    user: "root"
    password: ""
  internal:
    operator:
      enabled: false
```

```bash
helm install dogstack . -n dogstack -f values-external-doris.yaml
```

> **注意：** 使用外部 Doris 时，`feHttpPort` 用于 Stream Load 操作（默认 8030）。如果你的 Doris FE 使用不同的 HTTP 端口，请确保正确设置。

---

## 环境特定部署

### 开发环境

适用于本地开发的最小资源配置：

```bash
helm install dogstack . -n dogstack -f values-dev.yaml \
  --set dorisPlugin.image.tag=v1.0.1
```

特点：
- 所有组件单副本
- 不使用持久化存储（emptyDir）
- 启用 Debug Exporter
- 最小资源请求

### 生产环境

高可用配置：

```bash
helm install dogstack . -n dogstack -f values-prod.yaml \
  --set dorisPlugin.image.tag=v1.0.1 \
  --set grafana.adminPassword="CHANGE_ME_IN_PRODUCTION"
```

特点：
- 高可用配置（3 FE、3 BE、3 OTel Collector）
- 启用持久化存储
- 更高的资源限制
- 禁用 Debug Exporter
- 启用 Ingress 和 TLS

---

## 生产注意事项

### 资源管理

```yaml
otel:
  resources:
    requests:
      cpu: "500m"
      memory: "512Mi"
    limits:
      cpu: "2"
      memory: "2Gi"

grafana:
  resources:
    requests:
      cpu: "200m"
      memory: "256Mi"
    limits:
      cpu: "1"
      memory: "1Gi"
```

### Ingress 配置

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  hosts:
    - host: dogstack.example.com
      paths:
        - path: /
          pathType: Prefix
          service: grafana
  tls:
    - secretName: dogstack-tls
      hosts:
        - dogstack.example.com
```

### 持久化存储

生产环境务必启用持久化：

```yaml
doris:
  internal:
    cluster:
      persistence:
        enabled: true
        storageClass: "your-storage-class"
        fe:
          size: 100Gi
        be:
          size: 500Gi

otel:
  persistence:
    enabled: true
    size: 50Gi

grafana:
  persistence:
    enabled: true
    size: 50Gi
```

---

## 升级 Chart

升级到新版本：

```bash
helm upgrade dogstack . -n dogstack -f your-values.yaml
```

查看当前 Release：

```bash
helm list -n dogstack
```

---

## 卸载 DOGStack

删除部署：

```bash
helm uninstall dogstack -n dogstack
```

如果使用内部 Doris，可能需要手动删除 DorisCluster CR：

```bash
kubectl delete doriscluster -n dogstack --all
```

如需删除所有数据，删除 PVC：

```bash
kubectl delete pvc -n dogstack --all
```

删除命名空间（可选）：

```bash
kubectl delete namespace dogstack
```

---

## 故障排除

### 查看日志

```bash
# OTel Collector 日志
kubectl logs -l app.kubernetes.io/name=dogstack-otel-collector -n dogstack

# Grafana 日志
kubectl logs -l app.kubernetes.io/name=dogstack-grafana -n dogstack

# Doris FE 日志
kubectl logs -l app.kubernetes.io/component=fe -n dogstack

# Doris BE 日志
kubectl logs -l app.kubernetes.io/component=be -n dogstack
```

### 调试安装失败

```bash
helm install dogstack . -n dogstack --debug --dry-run
```

### 验证部署

```bash
kubectl get pods -n dogstack
kubectl get svc -n dogstack
kubectl get doriscluster -n dogstack
```

### 常见问题

| 问题 | 解决方案 |
|------|----------|
| OTel Collector CrashLoopBackOff | 检查 Doris 连接：`kubectl logs dogstack-otel-collector-0 -n dogstack` |
| Grafana 插件未加载 | 验证插件镜像已加载：`kubectl describe pod -l app.kubernetes.io/name=dogstack-grafana -n dogstack` |
| Doris FE 未就绪 | 检查 Doris Operator 日志：`kubectl logs -l app.kubernetes.io/name=doris-operator -n dogstack` |

---

## 架构图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Kubernetes 集群                                 │
│                                                                          │
│   ┌────────────────┐     ┌────────────────┐     ┌────────────────┐      │
│   │  应用程序       │     │  应用程序       │     │  应用程序       │      │
│   │ (已接入探针)    │     │ (已接入探针)    │     │ (已接入探针)    │      │
│   └───────┬────────┘     └───────┬────────┘     └───────┬────────┘      │
│           │                      │                      │                │
│           │     OTLP (gRPC:4317 / HTTP:4318)           │                │
│           └──────────────────────┼──────────────────────┘                │
│                                  ▼                                       │
│              ┌───────────────────────────────────┐                       │
│              │   OpenTelemetry Collector         │                       │
│              │   (dogstack-otel-collector)       │                       │
│              └───────────────────┬───────────────┘                       │
│                                  │ Doris Exporter (Stream Load)         │
│                                  ▼                                       │
│              ┌───────────────────────────────────┐                       │
│              │   Apache Doris                    │                       │
│              │   ┌─────────┐    ┌─────────┐      │                       │
│              │   │   FE    │    │   BE    │      │                       │
│              │   └─────────┘    └─────────┘      │                       │
│              │   (dogstack-doris)                │                       │
│              └───────────────────┬───────────────┘                       │
│                                  │ MySQL 协议 (9030)                    │
│                                  ▼                                       │
│              ┌───────────────────────────────────┐                       │
│              │   Grafana                         │                       │
│              │   (dogstack-grafana)              │                       │
│              │   + Doris App 插件                │                       │
│              │   + 预置仪表盘                    │                       │
│              └───────────────────────────────────┘                       │
│                                  │ 端口 3000                            │
└──────────────────────────────────┼───────────────────────────────────────┘
                                   ▼
                              👤 用户访问
```

---

## 服务端点

安装完成后，以下服务可用：

| 服务 | 端口 | 描述 |
|------|------|------|
| `dogstack-otel-collector` | 4317 | OTLP gRPC 接收器 |
| `dogstack-otel-collector` | 4318 | OTLP HTTP 接收器 |
| `dogstack-otel-collector` | 8888 | Prometheus 指标 |
| `dogstack-grafana` | 3000 | Grafana Web UI |
| `dogstack-doris-fe-service` | 9030 | Doris MySQL 协议 |
| `dogstack-doris-fe-service` | 8030 | Doris FE HTTP (Stream Load) |
| `dogstack-doris-be-service` | 8040 | Doris BE HTTP |

