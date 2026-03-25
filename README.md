# Gravitate Health Monitor

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

A complete, Helm-based monitoring stack for Kubernetes clusters featuring Prometheus, Grafana, Loki, and Promtail. Designed for containerized applications with pre-configured dashboards, log aggregation, and multi-environment support.

---

## Table of Contents

- [TLDR / Quickstart](#tldr--quickstart)
- [Architecture Overview](#architecture-overview)
- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration Guide](#configuration-guide)
- [Advanced Customization](#advanced-customization)
- [Verification & Post-Deployment](#verification--post-deployment)
- [Troubleshooting](#troubleshooting)
- [License](#license)

---

## TLDR / Quickstart

Deploy the complete monitoring stack in ~5 minutes:

```bash
# 1. Create namespace and setup (one-time)
kubectl create namespace monitoring
kubectl label namespace monitoring istio-injection=enabled --overwrite

# 2. Add secrets and dashboards
kubectl apply -f k8s/secrets/grafana-email-secret.yaml
kubectl create configmap -n monitoring grafana-dashboards-gh --from-file=./k8s/dashboards/ --dry-run=client -o yaml | kubectl apply -f -

# 3. Deploy with your base URL
helmfile sync --state-values-set baseUrl=https://your-domain.example.com
```

**That's it!** Access Grafana at `https://your-domain.example.com/grafana` (default credentials: admin/admin, change immediately).

---

## Architecture Overview

This stack provides comprehensive monitoring across three layers:

```
┌─────────────────────────────────────────────────────┐
│           Grafana (Visualization)                   │
│  • Pre-built dashboards for logs & metrics          │
│  • SMTP-enabled alerting                            │
│  • Multi-datasource support                         │
└────────────┬──────────────────┬──────────────────────┘
             │                  │
      ┌──────▼─────┐      ┌─────▼──────┐
      │ Prometheus │      │    Loki    │
      │ (Metrics)  │      │  (Logs)    │
      │ 10Gi store │      │ 50Gi store │
      └──────┬─────┘      └─────┬──────┘
             │                  │
      ┌──────▼──────┐    ┌──────▼──────┐
      │  Scrape     │    │  Promtail   │
      │  Targets    │    │  (Collector)│
      │  + Blackbox │    │             │
      └─────────────┘    └─────────────┘
```

**Key Components:**
- **Prometheus**: Scrapes metrics from cluster & external targets (HTTP/ICMP probes via Blackbox Exporter)
- **Loki**: Aggregates container logs from all namespaces via Promtail DaemonSet
- **Grafana**: Unified dashboard combining metrics and logs with email alerting
- **VirtualService**: Istio-based routing to expose all services via a single gateway at `/grafana`, `/prometheus`, `/loki` paths

---

## Repository Structure

```
.
├── helmfile.yaml                 # Main deployment orchestration
├── helm/
│   ├── values/                   # Helm chart values (templated & static)
│   │   ├── prometheus.yaml.gotmpl      # Prometheus config + external URL
│   │   ├── grafana.yaml.gotmpl         # Grafana config + SMTP + dashboards
│   │   ├── loki.yaml                   # Loki SingleBinary mode (filesystem)
│   │   ├── promtail.yaml               # Log collector config
│   │   └── blackbox-exporter.yaml      # HTTP/ICMP probe modules
│   └── charts/
│       └── monitor-routing/            # Local Helm chart for Istio VirtualService
│           ├── Chart.yaml
│           ├── values.yaml
│           └── templates/virtualservice.yaml
├── k8s/
│   ├── secrets/
│   │   └── grafana-email-secret.yaml   # SMTP credentials (apply manually)
│   └── dashboards/
│       ├── gh-logs.json                # Log aggregation dashboard
│       └── gh-monit.json               # gh_monit metrics dashboard
├── docs/
│   └── prometheus-grafana-stack.png    # Architecture diagram
├── deprecated_deployment/              # Legacy (non-Helm) configs - ignored
└── README.md                           # This file
```

**Key Design Decisions:**
- `.gotmpl` files use **Go templating** to parameterize `baseUrl` across multiple services
- **No environment files**: Single configuration set, parameterized via `--state-values-set`
- **Local Helm chart**: `monitor-routing` packages Istio VirtualService; avoids raw YAML manifests
- **Filesystem storage**: Loki & Prometheus store data in cluster volumes (not cloud backends)

---

## Prerequisites

1. **Kubernetes cluster** (v1.19+) with RBAC enabled
2. **Helm 3.x** installed locally
3. **Helmfile** installed ([installation guide](https://github.com/roboll/helmfile#installation))
4. **Istio** installed with:
   - Gateway `default/gh-gateway` configured in namespace where this is deployed
   - Namespace has `istio-injection=enabled` label
5. **kubectl** access to the cluster
6. **Domain name** pointing to your Istio ingress (for external access)
7. **SMTP server** details (for Grafana email alerts)

---

## Installation

### Step 1: Prepare Kubernetes Namespace

```bash
# Create monitoring namespace
kubectl create namespace monitoring

# Enable Istio injection (required for VirtualService routing)
kubectl label namespace monitoring istio-injection=enabled --overwrite

# Verify Istio gateway exists
kubectl get gateway -n default  # Confirm "gh-gateway" is listed
```

### Step 2: Configure Secrets

Edit [k8s/secrets/grafana-email-secret.yaml](k8s/secrets/grafana-email-secret.yaml) with your SMTP credentials:

```yaml
stringData:
  GF_SMTP_FROM_ADDRESS: your-email@example.com
  GF_SMTP_PASSWORD: your-smtp-password   # Can be plain text or use external secret management
```

Apply the secret:

```bash
kubectl apply -f k8s/secrets/grafana-email-secret.yaml
```

**Alternative: Pass secrets via CLI** (see [Advanced Customization](#passing-secrets-via-cli))

### Step 3: Create Dashboard ConfigMap

Grafana requires dashboards to be mounted as a ConfigMap:

```bash
kubectl create configmap \
  -n monitoring \
  grafana-dashboards-gh \
  --from-file=./k8s/dashboards/ \
  --dry-run=client -o yaml | kubectl apply -f -
```

### Step 4: Deploy via Helmfile

**Dry-run first** (recommended):

```bash
helmfile diff --state-values-set baseUrl=https://your-domain.example.com
```

**Deploy:**

```bash
helmfile sync --state-values-set baseUrl=https://your-domain.example.com
```

Monitor the rollout:

```bash
kubectl rollout status deployment/grafana -n monitoring
kubectl rollout status deployment/prometheus-server -n monitoring
kubectl rollout status statefulset/loki -n monitoring
```

---

## Configuration Guide

### Changing the Target Namespace

By default, all components deploy to the `monitoring` namespace. To use a different namespace:

```bash
# Option 1: Edit helmfile.yaml directly
# Change "namespace: monitoring" to your desired namespace in each release

# Option 2: Override via environment (not recommended)
export MONITORING_NAMESPACE=my-monitoring
sed -i "s/namespace: monitoring/namespace: $MONITORING_NAMESPACE/g" helmfile.yaml
helmfile sync --state-values-set baseUrl=...
```

**Note**: Ensure your custom namespace has `istio-injection=enabled` label.

### Excluding Specific Services from Deployment

Use Helmfile release filters. For example, to skip **Loki** (logs only, keep metrics):

```bash
helmfile sync \
  --state-values-set baseUrl=https://your-domain.example.com \
  --selector name!=loki \
  --selector name!=promtail
```

**Available releases to filter:**
- `monitor-routing` (Istio VirtualService)
- `prometheus` (metrics)
- `loki` (log aggregation backend)
- `promtail` (log collector daemon)
- `prometheus-blackbox-exporter` (external endpoint prober)
- `grafana` (dashboard UI)

**Example: Metrics-only setup** (skip Loki/Promtail):

```bash
helmfile sync --state-values-set baseUrl=... \
  --selector name=prometheus \
  --selector name=grafana \
  --selector name=prometheus-blackbox-exporter \
  --selector name=monitor-routing
```

### Modifying Values

#### Update Prometheus Scrape Config

Edit [helm/values/prometheus.yaml.gotmpl](helm/values/prometheus.yaml.gotmpl) to add custom `extraScrapeConfigs` or modify Blackbox probe targets.

#### Change Storage Sizes

```bash
# Override Prometheus PVC size (default 10Gi)
helmfile sync --state-values-set 'prometheus.persistentVolume.size=20Gi' ...

# Override Loki PVC size (default 50Gi)
helmfile sync --state-values-set 'loki.singleBinary.persistence.size=100Gi' ...
```

#### Disable Alertmanager

Edit [helm/values/prometheus.yaml.gotmpl](helm/values/prometheus.yaml.gotmpl):

```yaml
alertmanager:
  enabled: false  # Already disabled by default
```

#### Modify VirtualService Routing Paths

Edit [helm/charts/monitor-routing/values.yaml](helm/charts/monitor-routing/values.yaml):

```yaml
routes:
  grafana:
    prefix: /grafana        # Change to /monitoring/grafana
    host: grafana.monitoring.svc.cluster.local
    port: 80
```

---

## Advanced Customization

### Passing Secrets via CLI (Avoiding Files)

Instead of storing secrets in `k8s/secrets/grafana-email-secret.yaml`, pass them as Helm values:

```bash
helmfile sync --state-values-set baseUrl=https://your-domain.com \
  --state-values-set 'grafana.env.GF_SMTP_FROM_ADDRESS=your-email@example.com' \
  --state-values-set 'grafana.env.GF_SMTP_PASSWORD=your-password'
```

**For production**, use external secret management:

```bash
# Create secret from external vault
kubectl create secret generic grafana-email \
  --from-literal=GF_SMTP_FROM_ADDRESS=$SMTP_USER \
  --from-literal=GF_SMTP_PASSWORD=$SMTP_PASSWORD \
  -n monitoring

# This secret is already referenced in helmfile; no further steps needed
```

### Custom Dashboards

Add new Grafana dashboards:

1. **Export from Grafana**: Share → Export → Copy JSON
2. **Save to repo**: `k8s/dashboards/my-dashboard.json`
3. **Recreate ConfigMap**:
   ```bash
   kubectl create configmap \
     -n monitoring \
     grafana-dashboards-gh \
     --from-file=./k8s/dashboards/ \
     --dry-run=client -o yaml | kubectl apply -f -
   ```
4. **Restart Grafana**: `kubectl rollout restart deployment/grafana -n monitoring`

### Persistent Volume Provisioning

Both Prometheus and Loki require PersistentVolumes. Ensure your cluster has a default StorageClass:

```bash
# List available storage classes
kubectl get storageclass

# Set if not marked as default
kubectl patch storageclass <name> -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

If PVC gets stuck in "Pending", check available nodes:

```bash
kubectl describe pvc prometheus-server -n monitoring
kubectl get nodes
```

### Using External Prometheus or Loki

To use existing Prometheus/Loki instances instead of deploying new ones:

**Option 1: Skip deployment** (using Helmfile selectors - see Exclusion section)

**Option 2: Configure Grafana datasources manually**

Edit [helm/values/grafana.yaml.gotmpl](helm/values/grafana.yaml.gotmpl) and update datasource URLs:

```yaml
datasources:
  datasources.yaml:
    datasources:
      - name: Prometheus
        url: http://external-prometheus.example.com:9090  # Change here
      - name: Loki
        url: http://external-loki.example.com:3100        # Change here
```

---

## Verification & Post-Deployment

### 1. Verify All Pods Are Running

```bash
kubectl get pods -n monitoring

# Expected output (6 running pods):
# NAME                                              READY   STATUS
# monitor-routing-...                               1/1     Running
# prometheus-server-...                             2/2     Running
# grafana-...                                       1/1     Running
# loki-...                                          1/1     Running
# promtail-...                                      1/1     Running (DaemonSet)
# prometheus-blackbox-exporter-...                  1/1     Running
```

### 2. Verify Services Are Accessible

```bash
# Check services exist
kubectl get svc -n monitoring

# Expected: grafana, prometheus-server, loki, loki-gateway, prometheus-blackbox-exporter
```

### 3. Test Grafana Access

```bash
# Port-forward for quick test
kubectl port-forward svc/grafana -n monitoring 3000:80

# Visit http://localhost:3000
# Default login: admin / admin
```

### 4. Verify Datasources

In Grafana UI → Configuration → Data sources:

- **Prometheus** should be healthy (green checkmark)
- **Loki** should be healthy (green checkmark)
- Click "Test" on each to verify connectivity

### 5. Check Prometheus Targets

Via Prometheus UI (`https://your-domain.com/prometheus`):

1. Go to **Status → Targets**
2. Verify scrape jobs are in "UP" state:
   - `blackbox` (external probes)
   - `kubernetes-*` (cluster monitoring)
   - Any custom jobs

### 6. Monitor Logs in Loki

In Grafana, open **Explore** and switch to Loki datasource:

```
{namespace="monitoring"}
```

Should show logs from monitoring stack pods.

### 7. Check Storage Usage

```bash
# Monitor PVC usage
kubectl get pvc -n monitoring

# View actual storage consumption
kubectl exec -it loki-0 -n monitoring -- du -sh /loki
kubectl exec -it prometheus-server-... -n monitoring -- du -sh /prometheus
```

### Next Steps After Deployment

1. **Change Grafana Admin Password** (required for production)
   - Login as admin/admin
   - Profile (top right) → Change Password

2. **Configure Email Alerts**
   - **Test SMTP**: Grafana → Alerting → Notification channels → Test
   - **Create Alert Rules**: Grafana → Alerting → Alert Rules → Create

3. **Customize Dashboards**
   - Import community dashboards: Grafana → Dashboards → Import
   - Create custom dashboards for your application metrics

4. **Setup Log Aggregation**
   - Ensure Promtail is collecting from all namespaces (DaemonSet deployed cluster-wide)
   - Test log queries in Explore → Loki datasource

5. **Monitor Prometheus Retention**
   - Default retention: 15 days
   - Check actual usage: `kubectl top pod -n monitoring`
   - Increase if needed via Helm values

6. **Backup Dashboards & Alerts** (Production)
   ```bash
   # Export all dashboards and alert rules periodically
   # Use Grafana API or backup tools like https://github.com/grafana/cortex-tools
   ```

---

## Troubleshooting

### Prometheus Readiness Probe Failing

If Prometheus returns HTTP 503 for readiness checks (common with path prefix `/prometheus`):

```bash
kubectl patch deployment prometheus-server -n monitoring --type=json -p '[
  {"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/httpGet/path","value":"/prometheus/-/healthy"},
  {"op":"replace","path":"/spec/template/spec/containers/0/livenessProbe/httpGet/path","value":"/prometheus/-/healthy"}
]'
```

### Loki OOM Errors

Increase memory request:

```bash
helmfile sync --state-values-set baseUrl=... \
  --state-values-set 'loki.singleBinary.resources.requests.memory=2Gi'
```

### VirtualService Not Routing Traffic

Verify Istio gateway exists:

```bash
kubectl get gateway -n default
kubectl get virtualservice -n monitoring

# Check routing rules applied
kubectl logs -n istio-system -l app=istiod | grep "monitor"
```

### Dashboard Not Loading in Grafana

1. Check ConfigMap was created: `kubectl get cm grafana-dashboards-gh -n monitoring`
2. Restart Grafana: `kubectl rollout restart deployment/grafana -n monitoring`
3. Verify dashboard provisioning in **Grafana → Administration → Dashboards → Manage**

### Helm Dependency Errors

If Helm repos are not available:

```bash
helmfile repos
helmfile charts   # Updates chart dependencies

# If helmfile itself not found:
helmfile --version  # Verify installation
```

---

## License

Licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) for details.

---

## Support & Contributing

For issues, questions, or contributions, please refer to the project's GitHub repository.

**Disclaimer**: This monitoring stack is designed for Gravitate Health project environments. Adapt configurations to your specific security, compliance, and operational requirements before production deployment.
