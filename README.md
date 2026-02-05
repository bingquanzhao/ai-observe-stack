As shown in the architecture, DOG Stack consists of four components: data collection tools, OpenTelemetry Collector, Doris and Grafana with Doris App. This deployment documentation will focus on the last three components. And the 'Ingesting Data' documentations will focus on the first components.

DOG Stack offers multiple deployment options for different environment and purpose.

# Docker Compose

Suitable for local testing, development, and proof of concepts (PoC).

## Prerequisites

- Docker and Docker Compose installed

## Deployment Steps

### 1. Clone Repository

```bash
git clone https://github.com/velodb/DogStack.git
cd DogStack/docker
```

### 2. Start Services

```bash
docker compose up -d
```

### 3. View Logs (Optional)

```bash
docker compose logs -f
```

### 4. Access Grafana

Visit http://localhost:3000 to access the Grafana UI.

Default credentials:
- Username: `admin`
- Password: `admin`

## Access Endpoints

| Service | URL/Endpoint | Credentials |
|---------|--------------|-------------|
| Grafana | http://localhost:3000 | admin / admin |
| Doris FE UI | http://localhost:8030 | root / (empty) |
| Doris MySQL | localhost:9030 | root / (empty) |
| OTel gRPC | localhost:4317 | - |
| OTel HTTP | localhost:4318 | - |

## Stop Services

```bash
# Stop services (preserve data)
docker compose down

# Stop services and remove data volumes
docker compose down -v
```

# Kubernetes (Helm)

Suitable for production deployments, development environments, and proof of concepts.

## Prerequisites

- Kubernetes 1.20+
- Helm 3.0+
- `kubectl` configured to interact with your cluster
- PV provisioner support (for persistence)

## Deployment Steps

### 1. Clone Repository

```bash
git clone https://github.com/velodb/DogStack.git
cd DogStack/helm-charts
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

Or install with external Doris:

```bash
helm install dogstack . -n dogstack \
  --set dorisPlugin.image.tag=v1.0.1 \
  --set doris.mode=external \
  --set doris.external.host=<DORIS_FE_HOST> \
  --set doris.external.port=9030 \
  --set doris.external.feHttpPort=8030 \
  --set doris.internal.operator.enabled=false
```

### 7. Verify Installation

```bash
kubectl get pods -n dogstack
```

### 8. Access Grafana

```bash
kubectl port-forward svc/dogstack-grafana 3000:3000 -n dogstack
```

Visit http://localhost:3000 (admin / admin)

## Uninstall

```bash
helm uninstall dogstack -n dogstack
kubectl delete namespace dogstack
```

# Manually

1. Deploy a Doris cluster following the Doris [deployment documentation](https://doris.apache.org/docs/4.x/install/preparation/env-checking). If there is a Doris cluster, this step can be skipped.
2. Deploy the OpenTelemetry Collector following the [OpenTelemetry deployment documentation](https://opentelemetry.io/docs/collector/install/).
3. Deploy the Grafana UI following the [Grafana deployment documentation](https://grafana.com/docs/grafana/latest/setup-grafana/installation/).
4. Navigate the DOG Stack web UI

Visit http://localhost:3000 to access the Grafana UI inside DOG Stack.

