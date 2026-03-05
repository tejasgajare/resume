# Kubernetes & Helm Chart Architecture

> **Tags**: `#interview-ready` `#kubernetes` `#helm` `#eks` `#istio` `#kustomize` `#prometheus`
> **Role**: Software Engineer — RAVE Platform, HPE GreenLake
> **Cross-references**: [aws-terraform.md](./aws-terraform.md), [secrets-management.md](./secrets-management.md), [cicd-pipelines.md](./cicd-pipelines.md)

---

## Overview

I manage the Kubernetes deployment infrastructure for the RAVE platform, which runs 20+ microservices across three Amazon EKS clusters (dev, stage, prod). My responsibilities include Helm chart architecture, Istio service mesh configuration, resource management, monitoring setup with Prometheus, and the overall deployment topology. The Helm chart is a single umbrella chart (`helm-rave`) that deploys all RAVE services into the `rave` namespace, with per-environment values files controlling replicas, resources, feature flags, and secrets.

---

## EKS Cluster Architecture

### Cluster Topology

Each environment runs a dedicated EKS cluster in us-west-2:

| Cluster                   | Environment | API Server                                                    |
|---------------------------|-------------|---------------------------------------------------------------|
| `rave-dev-us-west-2`      | dev         | `EC5B6C9E7628A2779ECDADBB1543F54F.gr7.us-west-2.eks.amazonaws.com` |
| `rave-stag-us-west-2`     | stage       | *(separate endpoint)*                                         |
| `rave-prod-us-west-2`     | prod        | *(separate endpoint)*                                         |

The clusters are managed by HPE's CCP (Common Cloud Platform) team, but I handle all application-layer Kubernetes resources within the `rave` namespace. I don't manage node groups or cluster-level addons directly—those are handled by CCP via Terraform.

### Namespace Strategy

RAVE services all run in a single `rave` namespace per cluster. This simplifies RBAC, network policies, and Istio configuration. Shared platform services (Consul, Istio control plane, monitoring stack) run in separate namespaces (`cloudops`, `istio-system`, `monitoring`).

### Multi-Tenancy Considerations

The EKS clusters host multiple teams' workloads. RAVE's isolation comes from:
- **Namespace-level RBAC**: Our ArgoCD application can only deploy to the `rave` namespace
- **Resource quotas**: Managed at the namespace level to prevent resource starvation
- **Network policies**: Istio mTLS and authorization policies control inter-service communication
- **Separate ServiceAccounts**: Each RAVE service has its own SA with scoped permissions

---

## Helm Chart Architecture

### Umbrella Chart Structure

The RAVE Helm chart (`helm-rave`) is a single umbrella chart that deploys all 20+ microservices:

```
helm-charts/rave/
├── Chart.yaml
├── templates/
│   ├── _helpers.tpl                          # Shared template helpers
│   ├── backtracefetchservice-deployment.yaml
│   ├── caserelay-deployment.yaml
│   ├── caserelay-service.yaml
│   ├── comenricher-deployment.yaml
│   ├── comenricher-prometheusrule.yaml
│   ├── comenricher-service.yaml
│   ├── common-alertManagerConfig.yaml
│   ├── common-secrets.yaml
│   ├── correlationengine-deployment.yaml
│   ├── crmrelay-deployment.yaml
│   ├── crmservice-deployment.yaml
│   ├── dataingestion-deployment.yaml
│   ├── dataparser-deployment.yaml
│   ├── dbupdate-cronjob.yaml
│   ├── decisionengine-deployment.yaml
│   ├── deviceshadow-deployment.yaml
│   ├── eventsDataProvider-deployment.yaml
│   ├── glpnotificationconsumer-deployment.yaml
│   ├── healthchecker-deployment.yaml
│   ├── homefleetcollector-deployment.yaml
│   ├── istio-envoy-sidecar-metrics.yaml
│   ├── istio-private-endpoint.yaml
│   ├── manualcase-attachments-sa.yaml
│   ├── monitoring/
│   ├── monitoring-prometheusrule.yaml
│   ├── pointnextcrm-deployment.yaml
│   ├── shared-externalsecrets.yaml
│   ├── wellnessapi-deployment.yaml
│   ├── wellnessproducer-deployment.yaml
│   └── wellnessproducer-prometheusrule.yaml
├── override/
│   ├── dev-values.yaml
│   ├── stag-values.yaml
│   └── prod-values.yaml
└── not-yet-activated/
    ├── wellnessapi-service.yaml
    └── templateeditor-service.yaml
```

### Why a Single Umbrella Chart?

We chose a monolithic umbrella chart over per-service charts because:
1. **Atomic deployments**: All services deploy together from a single image tag, ensuring version consistency
2. **Shared configuration**: Global values (MongoDB connection, Consul, Harmony endpoints) are defined once
3. **Simplified ArgoCD**: One ArgoCD Application per environment instead of 20+
4. **Trade-off**: Slower individual service deployments, but this matches our release cadence (all services build from the same monorepo `rave-cloud`)

### Deployment Template Pattern

Every service follows the same deployment template pattern. Here's the caserelay deployment as a representative example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: caserelay
  namespace: {{ .Values.global.namespace }}
  labels:
    {{- include "helm-rave.labels" . | nindent 4 }}
    app: caserelay
spec:
  replicas: {{ .Values.caserelay.replicaCount }}
  selector:
    matchLabels:
      app: caserelay
  template:
    metadata:
      labels:
        {{- include "helm-rave.labels" . | nindent 8 }}
        app: caserelay
      annotations:
        proxy.istio.io/config: '{ "holdApplicationUntilProxyStarts": true }'
    spec:
      serviceAccountName: {{ .Values.global.manualcases.s3.serviceAccount }}
      imagePullSecrets:
        - name: {{ .Values.global.dockerSecretName }}
      containers:
        - name: caserelay
          image: "{{ .Values.container.image.registry }}/{{ .Values.container.image.path }}/caserelay:{{ .Values.container.image.tag }}"
          imagePullPolicy: Always
          env:
            - name: CONSUL_HTTP_ADDR
              value: {{ .Values.global.consul }}
            - name: ENVIRONMENT
              value: {{ .Values.global.environment }}
```

Key patterns I maintain across all deployments:
- **Istio annotation** `holdApplicationUntilProxyStarts: true` — prevents app startup before the Envoy sidecar is ready, avoiding connection failures
- **ServiceAccount binding** — each service specifies its SA for IRSA AWS access
- **Image pull secrets** — `regcred` for JFrog Artifactory authentication
- **Consul integration** — every service connects to the Consul server for dynamic configuration
- **Environment injection** — the `ENVIRONMENT` env var lets services adapt behavior per environment

### Per-Environment Values

The ArgoCD deployment repo (`mcd-deploy-proj-rave`) contains environment-specific values:

**Dev** (`values-dev.yaml`):
```yaml
global:
  environment: dev
  awsAccountId: "590183756478"
  serviceAccount: default
  logLevel: "debug"

eso:
  enabled: true
  clusterSecretStore: rave-dev-secrets
  parameterStorePath: /eso/rave/rave-dev/rave-dev-secrets

caserelay:
  replicaCount: 1
```

**Prod** (`values-prod.yaml`):
```yaml
global:
  environment: prod
  awsAccountId: "481824204946"
  serviceAccount: default
  logLevel: "debug"

eso:
  enabled: true
  clusterSecretStore: rave-prod-secrets
  parameterStorePath: /eso/rave/rave-prod/rave-prod-secrets

crmrelay:
  replicaCount: 6
  resources:
    limits:
      cpu: "500m"
```

Notice how prod has higher replica counts (crmrelay: 6 vs dev: 1) and explicit resource limits. I tune these based on observed load patterns and OOM/CPU throttling incidents.

---

## Istio Service Mesh

### Istio Integration Points

I manage several Istio custom resources for the RAVE namespace:

#### 1. DestinationRules for MongoDB PrivateLink

MongoDB Atlas is accessed via AWS PrivateLink. I created DestinationRules to configure TCP keepalive settings that prevent idle connection drops:

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: mongo-privatelink1
  namespace: {{ .Values.global.namespace }}
spec:
  host: dev-rave-pl-0.ehjcd.mongodb.net
  trafficPolicy:
    connectionPool:
      tcp:
        tcpKeepalive:
          time: 300s
          interval: 60s
```

Each environment has its own MongoDB private endpoints:
- Dev: `dev-rave-pl-0.ehjcd.mongodb.net`, `dev-rave-pl-1.ehjcd.mongodb.net`
- Stage: `stag-rave-pl-0.xkgtz.mongodb.net`
- Prod: `rave-prod-pl-0.eummz.mongodb.net`

#### 2. ServiceEntry for External Access

Istio's default behavior blocks traffic to hosts not in the mesh. I define ServiceEntries to allow RAVE services to reach MongoDB:

```yaml
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: mongo-private-endpoint
spec:
  hosts:
    - dev-rave-pl-0.ehjcd.mongodb.net
    - dev-rave-pl-1.ehjcd.mongodb.net
    - stag-rave-pl-0.xkgtz.mongodb.net
    - rave-prod-pl-0.eummz.mongodb.net
  location: MESH_EXTERNAL
  ports:
    - number: 27017
      name: mongodb
      protocol: TLS
  resolution: DNS
```

#### 3. Envoy Sidecar Metrics (PodMonitor)

I configured Prometheus to scrape Istio sidecar metrics from RAVE pods:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: istio-sidecar-metrics
  namespace: {{ .Values.global.namespace }}
  labels:
    release: tenant-monitoring
spec:
  selector:
    matchLabels:
      security.istio.io/tlsMode: istio
  podMetricsEndpoints:
    - port: {{ .Values.global.httpEnvoyProm }}
      path: /stats/prometheus
      interval: 15s
      relabelings:
        - action: keep
          sourceLabels: [__meta_kubernetes_pod_container_name]
          regex: "istio-proxy"
```

This gives us Envoy-level metrics (request duration, error rates, connection counts) for every RAVE service without modifying application code.

#### 4. Istio Sidecar Annotations

Every RAVE deployment includes the critical Istio annotation:
```yaml
annotations:
  proxy.istio.io/config: '{ "holdApplicationUntilProxyStarts": true }'
```

I added this after debugging intermittent startup failures where services tried to connect to MongoDB before the Envoy proxy was ready, causing TLS handshake failures. This annotation ensures the application container waits for the sidecar to be healthy.

---

## Resource Management

### Replica Counts by Environment

I manage replica counts based on load patterns:

| Service             | Dev | Stage | Prod | Notes                              |
|---------------------|-----|-------|------|------------------------------------|
| backtracefetchservice | 1 | 1     | 2    | Moderate load                      |
| correlationengine   | 1   | 1     | 2    | CPU-intensive correlation logic    |
| crmrelay            | 1   | 1     | 6    | High volume — external CRM calls   |
| crmservice          | 1   | 1     | 1    | Internal CRM operations            |
| caserelay           | 1   | 1     | 2+   | Email/case processing              |

### Resource Limits

Prod services have explicit resource limits to prevent noisy-neighbor issues:
```yaml
crmrelay:
  resources:
    limits:
      cpu: "500m"
```

I set these based on monitoring data from Prometheus/New Relic, looking at P95 CPU and memory usage, then adding a 30-50% buffer for burst capacity.

### CronJobs

The `dbupdate-cronjob.yaml` runs periodic database maintenance tasks. I configure schedule, concurrency policy, and resource limits for these jobs.

---

## Kustomize for Prometheus

### `kustomize-prometheus` Repo

I contribute to the `kustomize-prometheus` repo, which uses Kustomize overlays to deploy Prometheus monitoring configurations across environments. This is separate from the Helm-based RAVE deployment because the monitoring stack is shared across teams.

My contributions include:
- **PrometheusRule definitions**: Alert rules for RAVE service health (high error rates, pod restarts, memory pressure)
- **ServiceMonitor configurations**: Scrape targets for RAVE service metrics endpoints
- **PodMonitor for Istio**: The Envoy sidecar metrics configuration described above
- **AlertManagerConfig**: PagerDuty integration for production alerts

The Kustomize structure:
```
kustomize-prometheus/
├── base/
│   ├── prometheusrules/
│   └── servicemonitors/
└── overlays/
    ├── dev/
    ├── stage/
    └── prod/
```

---

## Monitoring & Alerting Templates

### PrometheusRules

I maintain PrometheusRule CRDs for key RAVE services. Example for wellnessproducer:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: wellnessproducer-alerts
spec:
  groups:
    - name: wellnessproducer
      rules:
        - alert: WellnessProducerHighErrorRate
          expr: rate(http_requests_total{app="wellnessproducer", status=~"5.."}[5m]) > 0.1
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "High error rate on wellnessproducer"
```

### AlertManagerConfig

PagerDuty integration for production incident routing:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pagerduty-secret
  namespace: {{ .Values.global.namespace }}
type: Opaque
stringData:
  routingKey: <routing-key>
```

---

## ArgoCD Application Structure

### Metadata Configuration

Each environment's ArgoCD Application is defined in `mcd-deploy-proj-rave`:

```yaml
# applications/rave/dev/rave-dev-us-west-2/rave/metadata.yaml
cluster: https://EC5B6C9E7628A2779ECDADBB1543F54F.gr7.us-west-2.eks.amazonaws.com
github-group-names:
  teams:
    - rave
notification_email: peng.chen@hpe.com
helm-repo-url: https://hpeartifacts.jfrog.io/artifactory/api/helm/helm-harmony
chart: helm-rave
version: 1.0.1-cloudmaster-ccp-bddfa5517
helm-values-path: applications/rave/dev/rave-dev-us-west-2/rave/helm-values
namespace: rave
app-name: rave-d4e0ef
```

Key fields:
- **chart**: Points to `helm-rave` in JFrog Artifactory
- **version**: Matches the Git SHA from the build pipeline (e.g., `cloudmaster-ccp-bddfa5517`)
- **helm-values-path**: Relative path to the environment-specific values file
- **namespace**: Always `rave` — ArgoCD is scoped to this namespace

### Deployment Flow

```
Code merged to main → GitHub Actions builds images → 
  Helm chart published to JFrog → 
    Version updated in mcd-deploy-proj-rave metadata.yaml →
      ArgoCD detects change → Syncs to target cluster
```

---

## Interview Deep-Dives

### Q: Why a single umbrella Helm chart instead of per-service charts?

Our 20+ services are built from a single monorepo (`rave-cloud`) and always deploy together. A single chart ensures version consistency—you can't accidentally deploy wellnessproducer v1.2 with crmrelay v1.1. The trade-off is deployment granularity: you can't deploy one service without touching others. For us, this is acceptable because our release cadence is branch-based (all services share a tag). If we moved to independent release cycles, I'd refactor to per-service charts with a shared library chart for common templates.

### Q: How do you handle Istio proxy startup race conditions?

I added `holdApplicationUntilProxyStarts: true` to every deployment after debugging intermittent MongoDB connection failures. The root cause was the application container starting before the Envoy sidecar was ready—outbound TLS connections to MongoDB would fail because the proxy wasn't intercepting traffic yet. This annotation tells Kubernetes to hold the application container until the istio-proxy reports healthy. The downside is slightly slower pod startup (~2-3s), but it eliminates flaky initialization failures.

### Q: How would you scale this to 100 services?

I'd break the monolithic chart into a multi-chart architecture: a shared library chart with common templates (`_helpers.tpl`, standard deployment/service/HPA patterns), and per-service charts that inherit from it. Each service would get its own ArgoCD Application with independent sync. I'd also introduce ApplicationSets for pattern-based deployment across environments. For resource management at scale, I'd implement VPA (Vertical Pod Autoscaler) alongside HPA to right-size containers automatically.

### Q: Walk me through debugging a production pod crash.

First, `kubectl get pods -n rave` to identify the crashing pod. Then `kubectl describe pod <name>` to check events—is it OOMKilled, CrashLoopBackOff, or ImagePullBackOff? For OOM, I check `kubectl top pod` and Prometheus memory metrics to see if we need to increase limits. For CrashLoopBackOff, I check `kubectl logs <pod> -c <container> --previous` for the crash stack trace. If it's an Istio issue, I also check `kubectl logs <pod> -c istio-proxy` for sidecar errors. I cross-reference with Humio logs for the full application log trail.

---

## Key Repos

| Repo | Purpose | My Role |
|------|---------|---------|
| `rave-cloud` | Helm charts, service code | Chart templates, deployment configs |
| `mcd-deploy-proj-rave` | ArgoCD deployment | Environment values, version management |
| `kustomize-prometheus` | Monitoring configs | PrometheusRules, ServiceMonitors |

---

## Metrics & Impact

- **20+ microservices** deployed via a single Helm umbrella chart
- **3 EKS clusters** (dev/stage/prod) managed via ArgoCD GitOps
- **Istio service mesh** with mTLS, DestinationRules for MongoDB PrivateLink, and sidecar metrics scraping
- **Zero-downtime deployments** via rolling updates with readiness probes
- **Prometheus/PagerDuty alerting** for production service health
