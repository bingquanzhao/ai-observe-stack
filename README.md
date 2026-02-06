# Deployment

This guide covers three ways to deploy Open Observability Stack. Choose the method that fits your environment:

| Method | Best for |
|--------|----------|
| Docker Compose | Local testing, development, PoC |
| Kubernetes (Helm) | Production, scalable environments |
| Manual | Custom setups, existing infrastructure |

Each method deploys the core components: OpenTelemetry Collector, Apache Doris, and Grafana. For data collection configuration, see [Ingesting Data](docs/ingesting-data.md).

# Docker Compose

Best for local testing, development, and proof of concepts (PoC).

## Prerequisites

- Docker Engine (v20.10+)
- Docker Compose (v2.0+)

## Deploy

1. Clone the repository:

   ```bash
   git clone https://github.com/velodb/open-observability-stack.git
   cd open-observability-stack/docker
   ```

2. If you already have an existing Apache Doris cluster, configure the connection and start in external mode:

   ```bash
   cp .env.example .env
   ```

   Edit `.env` with your Doris connection details:

   ```bash
   DORIS_FE_HTTP_ENDPOINT=http://<DORIS_FE_HOST>:<FE_HTTP_PORT>
   DORIS_FE_MYSQL_ENDPOINT=<DORIS_FE_HOST>:<FE_MYSQL_PORT>
   DORIS_USERNAME=root
   DORIS_PASSWORD=
   ```

   Then start the services (OTel Collector + Grafana only):

   ```bash
   docker compose -f docker-compose-without-doris.yaml up -d
   ```

3. If you don't have a Doris cluster, simply start the full stack with the built-in Doris:

   ```bash
   docker compose up -d
   ```

4. Verify the services are running:

   ```bash
   docker compose ps
   ```

   All services should show `running` status.

5. Access Grafana at http://localhost:3000 and log in with `admin` / `admin`.

## Service endpoints

| Service | Endpoint | Credentials |
|---------|----------|-------------|
| Grafana | http://localhost:3000 | admin / admin |
| Doris FE UI | http://localhost:8030 | root / (empty) |
| Doris MySQL | localhost:9030 | root / (empty) |
| OTel gRPC | localhost:4317 | - |
| OTel HTTP | localhost:4318 | - |

> **Note:** Doris FE UI and Doris MySQL endpoints are only available with the built-in Doris. When using an external Doris, access your existing cluster directly.

## Stop and clean up

To stop services while preserving data:

```bash
docker compose down
```

To stop services and remove all data:

```bash
docker compose down -v
```

# Kubernetes (Helm)

Best for production deployments, development environments, and scalable setups.

## Prerequisites

- Kubernetes cluster (v1.20+)
- Helm (v3.0+)
- kubectl configured to access your cluster
- PersistentVolume provisioner (for data persistence)

## Deploy

1. Add the Open Observability Stack Helm repository:

   ```bash
   helm repo add open-observability-stack https://charts.velodb.io
   helm repo update
   ```

2. Create a namespace for Open Observability Stack:

   ```bash
   kubectl create namespace open-observability-stack
   ```

3. Install Open Observability Stack:

   ```bash
   helm install my-oos open-observability-stack/open-observability-stack -n open-observability-stack
   ```

   If you have an existing Doris cluster, use external mode instead:

   ```bash
   helm install my-oos open-observability-stack/open-observability-stack -n open-observability-stack \
     --set doris.mode=external \
     --set doris.external.host=<DORIS_FE_HOST> \
     --set doris.external.port=9030 \
     --set doris.external.feHttpPort=8030 \
     --set doris.internal.operator.enabled=false
   ```

4. Verify all pods are running:

   ```bash
   kubectl get pods -n open-observability-stack
   ```

   Wait until all pods show `Running` status.

5. Access Grafana:

   ```bash
   kubectl port-forward svc/my-oos-grafana 3000:3000 -n open-observability-stack
   ```

   Open http://localhost:3000 and log in with `admin` / `admin`.

## Service endpoints

| Service | Port-forward command |
|---------|---------------------|
| Grafana | `kubectl port-forward svc/my-oos-grafana 3000:3000 -n open-observability-stack` |
| Doris FE UI | `kubectl port-forward svc/my-oos-doris-fe 8030:8030 -n open-observability-stack` |
| Doris MySQL | `kubectl port-forward svc/my-oos-doris-fe 9030:9030 -n open-observability-stack` |

## Uninstall

```bash
helm uninstall my-oos -n open-observability-stack
kubectl delete namespace open-observability-stack
```

# Manually

Best for custom setups or integrating with existing infrastructure.

1. Deploy Apache Doris following the [Doris deployment documentation](https://doris.apache.org/docs/4.x/install/preparation/env-checking).
   Skip this step if you already have a Doris cluster.

2. Deploy OpenTelemetry Collector following the [installation guide](https://opentelemetry.io/docs/collector/install/).

3. Deploy Grafana following the [Grafana installation documentation](https://grafana.com/docs/grafana/latest/setup-grafana/installation/).

4. Install the Open Observability Stack Grafana plugin. See [Plugin Installation](docs/plugin-installation.md).

After completing all steps, access Grafana at http://localhost:3000.

