# DOGStack Helm Chart

[中文文档](./README_zh.md)

**DOGStack** (Doris OpenTelemetry Grafana with Doris App Plugin) is a complete observability stack using Apache Doris as the storage backend for traces, metrics, and logs.

By default, the Helm chart provisions all core components, including:

- **Apache Doris** (via Doris Operator)
- **OpenTelemetry Collector** (with Doris Exporter)
- **Grafana** (with Doris App Plugin)

However, it can be easily customized to integrate with an existing Doris deployment.

The chart supports standard Kubernetes best practices, including:

- Environment-specific configuration via `values.yaml`
- Resource limits and pod-level scaling
- TLS and ingress configuration
- Secrets management and authentication setup

## Suitable for

- Proof of concepts
- Development environments
- Production deployments

---

## Prerequisites

- Kubernetes 1.20+
- Helm 3.0+
- `kubectl` configured to interact with your cluster
- PV provisioner support (for persistence)

---

## Deployment Steps

### 1. Download and Extract DOGStack

```bash
wget https://justtmp-1308700295.cos.ap-hongkong.myqcloud.com/DOG-k8s.tar.gz
tar -xzf DOG-k8s.tar.gz
cd DOG-k8s
```

### 2. Add Helm Repository

Add the required Helm repository for Doris Operator:

```bash
helm repo add doris-repo https://charts.selectdb.com
helm repo update
```

### 3. Create Namespace

```bash
kubectl create namespace dogstack
```

### 4. Build and Load Plugin Image (Required)

DOGStack uses a custom Grafana plugin image. The plugin is downloaded from S3/URL during build:

```bash
cd images/doris-plugin

# Build image (plugin is auto-downloaded from default URL)
docker build -t dog-grafana-assets:v1.0.1 .

# Or specify a custom plugin URL
docker build -t dog-grafana-assets:v1.0.1 \
  --build-arg DORIS_APP_URL=https://your-s3-bucket/doris-app-1.0.0.zip .

# For Kind cluster
kind load docker-image dog-grafana-assets:v1.0.1 --name kind

# For other clusters, push to your registry
# docker tag dog-grafana-assets:v1.0.1 your-registry/dog-grafana-assets:v1.0.1
# docker push your-registry/dog-grafana-assets:v1.0.1
```

### 5. Update Dependencies

```bash
cd ../..
helm dependency update .
```

### 6. Install DOGStack

Install with default values (internal Doris):

```bash
helm install dogstack . -n dogstack \
  --set dorisPlugin.image.tag=v1.0.1
```

---

## Verify the Installation

Check that all pods are running:

```bash
kubectl get pods -n dogstack
```

Expected output:

```
NAME                                READY   STATUS    RESTARTS   AGE
dogstack-doris-fe-0                 1/1     Running   0          2m
dogstack-doris-be-0                 1/1     Running   0          1m
dogstack-grafana-xxx                1/1     Running   0          2m
dogstack-otel-collector-0           1/1     Running   0          2m
dogstack-otel-collector-1           1/1     Running   0          2m
doris-operator-xxx                  1/1     Running   0          2m
```

Check DorisCluster status:

```bash
kubectl get doriscluster -n dogstack
```

---

## Forward Ports

Port forwarding allows you to access and set up DOGStack. For production deployments, configure ingress instead.

### Access Grafana

```bash
kubectl port-forward svc/dogstack-grafana 3000:3000 -n dogstack --address 0.0.0.0
```

Visit http://localhost:3000 (or http://YOUR_SERVER_IP:3000)

**Default credentials:**
- Username: `admin`
- Password: `admin`

### Access OTel Collector

```bash
kubectl port-forward svc/dogstack-otel-collector 4317:4317 4318:4318 -n dogstack --address 0.0.0.0
```

Send telemetry data to:
- OTLP gRPC: `localhost:4317`
- OTLP HTTP: `localhost:4318`

### Access Doris (Internal Mode)

```bash
# MySQL protocol
kubectl port-forward svc/dogstack-doris-fe-service 9030:9030 -n dogstack

# Connect via MySQL client
mysql -h 127.0.0.1 -P 9030 -u root

# Web UI
kubectl port-forward svc/dogstack-doris-fe-service 8030:8030 -n dogstack
```

---

## Customizing Values

You can customize settings by using `--set` flags:

```bash
helm install dogstack . -n dogstack \
  --set dorisPlugin.image.tag=v1.0.1 \
  --set grafana.adminPassword=mysecretpassword
```

Or create a custom `values.yaml`:

```bash
# Get default values
helm show values . > my-values.yaml

# Edit and install
helm install dogstack . -n dogstack -f my-values.yaml
```

### Key Configuration Options

| Parameter | Description | Default |
|-----------|-------------|---------|
| `doris.mode` | Doris deployment mode (`internal` / `external`) | `internal` |
| `doris.database` | Database name for observability data | `otel` |
| `doris.internal.cluster.fe.replicas` | Number of FE replicas | `1` |
| `doris.internal.cluster.be.replicas` | Number of BE replicas | `1` |
| `otel.enabled` | Enable OpenTelemetry Collector | `true` |
| `otel.replicas` | Number of OTel Collector replicas | `2` |
| `grafana.enabled` | Enable Grafana | `true` |
| `grafana.adminPassword` | Grafana admin password | `admin` |
| `dorisPlugin.enabled` | Enable Doris App Plugin | `true` |
| `ingress.enabled` | Enable Ingress | `false` |

See [values.yaml](./values.yaml) for the complete list.

---

## Using Secrets

For handling sensitive data such as database credentials, use Kubernetes secrets.

### Create a Secret

```bash
kubectl create secret generic doris-credentials \
  --from-literal=username=root \
  --from-literal=password=mysecretpassword \
  -n dogstack
```

### Reference in values.yaml

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

## Using External Doris

If using an existing Doris cluster, disable the internal Doris and specify the external connection:

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

Or use a `values.yaml` file:

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

> **Note:** When using external Doris, `feHttpPort` is used for Stream Load operations (default 8030). If your Doris FE uses a different HTTP port, make sure to set it correctly.

---

## Environment-Specific Deployments

### Development

Minimal resources for local development:

```bash
helm install dogstack . -n dogstack -f values-dev.yaml \
  --set dorisPlugin.image.tag=v1.0.1
```

Features:
- Single replica for all components
- No persistence (emptyDir)
- Debug exporter enabled
- Minimal resource requests

### Production

High availability configuration:

```bash
helm install dogstack . -n dogstack -f values-prod.yaml \
  --set dorisPlugin.image.tag=v1.0.1 \
  --set grafana.adminPassword="CHANGE_ME_IN_PRODUCTION"
```

Features:
- HA configuration (3 FE, 3 BE, 3 OTel Collectors)
- Persistence enabled
- Higher resource limits
- Debug exporter disabled
- Ingress enabled with TLS

---

## Production Notes

For production deployments, consider the following:

### Resource Management

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

### Ingress Configuration

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

### Persistence

Ensure persistence is enabled for production:

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

## Upgrading the Chart

To upgrade to a newer version:

```bash
helm upgrade dogstack . -n dogstack -f your-values.yaml
```

To check current release:

```bash
helm list -n dogstack
```

---

## Uninstalling DOGStack

To remove the deployment:

```bash
helm uninstall dogstack -n dogstack
```

If using internal Doris, the DorisCluster CR may need manual deletion:

```bash
kubectl delete doriscluster -n dogstack --all
```

Delete PVCs if you want to remove all data:

```bash
kubectl delete pvc -n dogstack --all
```

Delete namespace (optional):

```bash
kubectl delete namespace dogstack
```

---

## Troubleshooting

### Checking Logs

```bash
# OTel Collector logs
kubectl logs -l app.kubernetes.io/name=dogstack-otel-collector -n dogstack

# Grafana logs
kubectl logs -l app.kubernetes.io/name=dogstack-grafana -n dogstack

# Doris FE logs
kubectl logs -l app.kubernetes.io/component=fe -n dogstack

# Doris BE logs
kubectl logs -l app.kubernetes.io/component=be -n dogstack
```

### Debugging a Failed Install

```bash
helm install dogstack . -n dogstack --debug --dry-run
```

### Verifying Deployment

```bash
kubectl get pods -n dogstack
kubectl get svc -n dogstack
kubectl get doriscluster -n dogstack
```

### Common Issues

| Issue | Solution |
|-------|----------|
| OTel Collector CrashLoopBackOff | Check Doris connectivity: `kubectl logs dogstack-otel-collector-0 -n dogstack` |
| Grafana plugin not loading | Verify plugin image is loaded: `kubectl describe pod -l app.kubernetes.io/name=dogstack-grafana -n dogstack` |
| Doris FE not ready | Check Doris Operator logs: `kubectl logs -l app.kubernetes.io/name=doris-operator -n dogstack` |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Kubernetes Cluster                              │
│                                                                          │
│   ┌────────────────┐     ┌────────────────┐     ┌────────────────┐      │
│   │  Your Apps     │     │  Your Apps     │     │  Your Apps     │      │
│   │ (instrumented) │     │ (instrumented) │     │ (instrumented) │      │
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
│                                  │ MySQL Protocol (9030)                │
│                                  ▼                                       │
│              ┌───────────────────────────────────┐                       │
│              │   Grafana                         │                       │
│              │   (dogstack-grafana)              │                       │
│              │   + Doris App Plugin              │                       │
│              │   + Dashboards                    │                       │
│              └───────────────────────────────────┘                       │
│                                  │ Port 3000                            │
└──────────────────────────────────┼───────────────────────────────────────┘
                                   ▼
                              👤 Users
```

---

## Endpoints

After installation, the following services are available:

| Service | Port | Description |
|---------|------|-------------|
| `dogstack-otel-collector` | 4317 | OTLP gRPC receiver |
| `dogstack-otel-collector` | 4318 | OTLP HTTP receiver |
| `dogstack-otel-collector` | 8888 | Prometheus metrics |
| `dogstack-grafana` | 3000 | Grafana Web UI |
| `dogstack-doris-fe-service` | 9030 | Doris MySQL protocol |
| `dogstack-doris-fe-service` | 8030 | Doris FE HTTP (Stream Load) |
| `dogstack-doris-be-service` | 8040 | Doris BE HTTP |

