# Open Observability Stack Helm Chart

[中文文档](./README_zh.md)

**Open Observability Stack** is a complete observability stack using Apache Doris as the storage backend for traces, metrics, and logs, with OpenTelemetry and Grafana.

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

### 1. Add Helm Repository

```bash
helm repo add open-observability-stack https://charts.velodb.io
helm repo update
```

### 2. Create Namespace

```bash
kubectl create namespace open-observability-stack
```

### 3. Install Open Observability Stack

Install with default values (internal Doris):

```bash
helm install open-observability-stack open-observability-stack/open-observability-stack -n open-observability-stack
```

If you have an existing Doris cluster, use external mode:

```bash
helm install open-observability-stack open-observability-stack/open-observability-stack -n open-observability-stack \
  --set doris.mode=external \
  --set doris.external.host=<DORIS_FE_HOST> \
  --set doris.external.port=9030 \
  --set doris.external.feHttpPort=8030 \
  --set doris.internal.operator.enabled=false
```

---

## Verify the Installation

Check that all pods are running:

```bash
kubectl get pods -n open-observability-stack
```

Expected output (pod name prefix depends on release name):

```
NAME                                READY   STATUS    RESTARTS   AGE
open-observability-stack-doris-fe-0                 1/1     Running   0          2m
open-observability-stack-doris-be-0                 1/1     Running   0          1m
open-observability-stack-grafana-xxx                1/1     Running   0          2m
open-observability-stack-otel-collector-0           1/1     Running   0          2m
open-observability-stack-otel-collector-1           1/1     Running   0          2m
doris-operator-xxx                  1/1     Running   0          2m
```

Check DorisCluster status:

```bash
kubectl get doriscluster -n open-observability-stack
```

---

## Forward Ports

Port forwarding allows you to access and set up Open Observability Stack. For production deployments, configure ingress instead.

### Access Grafana

```bash
kubectl port-forward svc/open-observability-stack-grafana 3000:3000 -n open-observability-stack --address 0.0.0.0
```

Visit http://localhost:3000 (or http://YOUR_SERVER_IP:3000)

**Default credentials:**
- Username: `admin`
- Password: `admin`

### Access OTel Collector

```bash
kubectl port-forward svc/open-observability-stack-otel-collector 4317:4317 4318:4318 -n open-observability-stack --address 0.0.0.0
```

Send telemetry data to:
- OTLP gRPC: `localhost:4317`
- OTLP HTTP: `localhost:4318`

### Access Doris (Internal Mode)

```bash
# MySQL protocol
kubectl port-forward svc/open-observability-stack-doris-fe-service 9030:9030 -n open-observability-stack

# Connect via MySQL client
mysql -h 127.0.0.1 -P 9030 -u root

# Web UI
kubectl port-forward svc/open-observability-stack-doris-fe-service 8030:8030 -n open-observability-stack
```

---

## Customizing Values

You can customize settings by using `--set` flags:

```bash
helm install open-observability-stack open-observability-stack/open-observability-stack -n open-observability-stack \
  --set grafana.adminPassword=mysecretpassword
```

Or create a custom `values.yaml`:

```bash
# Get default values
helm show values open-observability-stack/open-observability-stack > my-values.yaml

# Edit and install
helm install open-observability-stack open-observability-stack/open-observability-stack -n open-observability-stack -f my-values.yaml
```

### Key Configuration Options

| Parameter | Description | Default |
|-----------|-------------|---------|
| `openObservabilityStack.timezone` | Timezone for the entire stack (Doris + OTel Collector) | `UTC` |
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

See [values.yaml](./open-observability-stack/values.yaml) for the complete list.

### Timezone Configuration

By default, all components use **UTC** timezone. To use a different timezone (e.g., `Asia/Shanghai`), set `openObservabilityStack.timezone`:

```bash
helm install open-observability-stack open-observability-stack/open-observability-stack -n open-observability-stack \
  --set openObservabilityStack.timezone=Asia/Shanghai
```

This single configuration applies to both Doris and OTel Collector, ensuring consistent time handling across the stack.

**Supported timezone formats:**
- `UTC` (default)
- `Asia/Shanghai`
- `America/New_York`
- `Europe/London`
- Any valid [IANA timezone](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)

> **Note:** Timezone consistency is critical. Mismatched timezones between Doris and OTel Collector will cause Grafana dashboards to display incorrect time ranges (e.g., "Last 15 minutes" queries returning no data).

---

## Using Secrets

For handling sensitive data such as database credentials, use Kubernetes secrets.

### Create a Secret

```bash
kubectl create secret generic doris-credentials \
  --from-literal=username=root \
  --from-literal=password=mysecretpassword \
  -n open-observability-stack
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
helm install open-observability-stack open-observability-stack/open-observability-stack -n open-observability-stack \
  --set doris.mode=external \
  --set doris.external.host=172.19.0.12 \
  --set doris.external.port=9030 \
  --set doris.external.feHttpPort=8030 \
  --set doris.external.beHttpPort=8040 \
  --set doris.external.user=root \
  --set doris.external.password="" \
  --set doris.internal.operator.enabled=false
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
helm install open-observability-stack . -n open-observability-stack -f values-external-doris.yaml
```

> **Note:** When using external Doris, `feHttpPort` is used for Stream Load operations (default 8030). If your Doris FE uses a different HTTP port, make sure to set it correctly.

---

## Environment-Specific Deployments

### Development

Minimal resources for local development:

```bash
helm install open-observability-stack open-observability-stack/open-observability-stack -n open-observability-stack -f values-dev.yaml
```

Features:
- Single replica for all components
- No persistence (emptyDir)
- Debug exporter enabled
- Minimal resource requests

### Production

High availability configuration:

```bash
helm install open-observability-stack open-observability-stack/open-observability-stack -n open-observability-stack -f values-prod.yaml \
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
    - host: oos.example.com
      paths:
        - path: /
          pathType: Prefix
          service: grafana
  tls:
    - secretName: oos-tls
      hosts:
        - oos.example.com
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
helm upgrade open-observability-stack open-observability-stack/open-observability-stack -n open-observability-stack -f your-values.yaml
```

To check current release:

```bash
helm list -n open-observability-stack
```

---

## Uninstalling Open Observability Stack

To remove the deployment:

```bash
helm uninstall open-observability-stack -n open-observability-stack
```

If using internal Doris, the DorisCluster CR may need manual deletion:

```bash
kubectl delete doriscluster -n open-observability-stack --all
```

Delete PVCs if you want to remove all data:

```bash
kubectl delete pvc -n open-observability-stack --all
```

Delete namespace (optional):

```bash
kubectl delete namespace open-observability-stack
```

---

## Troubleshooting

### Checking Logs

```bash
# OTel Collector logs
kubectl logs -l app.kubernetes.io/name=open-observability-stack-otel-collector -n open-observability-stack

# Grafana logs
kubectl logs -l app.kubernetes.io/name=open-observability-stack-grafana -n open-observability-stack

# Doris FE logs
kubectl logs -l app.kubernetes.io/component=fe -n open-observability-stack

# Doris BE logs
kubectl logs -l app.kubernetes.io/component=be -n open-observability-stack
```

### Debugging a Failed Install

```bash
helm install open-observability-stack open-observability-stack/open-observability-stack -n open-observability-stack --debug --dry-run
```

### Verifying Deployment

```bash
kubectl get pods -n open-observability-stack
kubectl get svc -n open-observability-stack
kubectl get doriscluster -n open-observability-stack
```

### Common Issues

| Issue | Solution |
|-------|----------|
| OTel Collector CrashLoopBackOff | Check Doris connectivity: `kubectl logs open-observability-stack-otel-collector-0 -n open-observability-stack` |
| Grafana plugin not loading | Verify plugin image is loaded: `kubectl describe pod -l app.kubernetes.io/name=open-observability-stack-grafana -n open-observability-stack` |
| Doris FE not ready | Check Doris Operator logs: `kubectl logs -l app.kubernetes.io/name=doris-operator -n open-observability-stack` |

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
│              │   (open-observability-stack-otel-collector)       │                       │
│              └───────────────────┬───────────────┘                       │
│                                  │ Doris Exporter (Stream Load)         │
│                                  ▼                                       │
│              ┌───────────────────────────────────┐                       │
│              │   Apache Doris                    │                       │
│              │   ┌─────────┐    ┌─────────┐      │                       │
│              │   │   FE    │    │   BE    │      │                       │
│              │   └─────────┘    └─────────┘      │                       │
│              │   (open-observability-stack-doris)                │                       │
│              └───────────────────┬───────────────┘                       │
│                                  │ MySQL Protocol (9030)                │
│                                  ▼                                       │
│              ┌───────────────────────────────────┐                       │
│              │   Grafana                         │                       │
│              │   (open-observability-stack-grafana)              │                       │
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
| `open-observability-stack-otel-collector` | 4317 | OTLP gRPC receiver |
| `open-observability-stack-otel-collector` | 4318 | OTLP HTTP receiver |
| `open-observability-stack-otel-collector` | 8888 | Prometheus metrics |
| `open-observability-stack-grafana` | 3000 | Grafana Web UI |
| `open-observability-stack-doris-fe-service` | 9030 | Doris MySQL protocol |
| `open-observability-stack-doris-fe-service` | 8030 | Doris FE HTTP (Stream Load) |
| `open-observability-stack-doris-be-service` | 8040 | Doris BE HTTP |

