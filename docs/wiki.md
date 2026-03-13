# kube-prometheus-stack GitOps Wiki

> This file mirrors the GitHub Wiki. Keep both in sync.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Secrets Setup](#secrets)
4. [Before Deploying](#before-deploying)
5. [Deploying](#deploying)
6. [Accessing Services](#accessing-services)
7. [cert-manager Dashboard](#cert-manager-dashboard)
8. [Grafana vs k8s-grafana repo](#grafana-note)
9. [Updating](#updating)
10. [Custom Alerts](#custom-alerts)
11. [Helm Values Reference](#helm-values-reference)
12. [Troubleshooting](#troubleshooting)
13. [Useful Commands](#useful-commands)

---

## Overview

This repo deploys the full Kubernetes monitoring stack in one Helm chart:

| Component | What it does |
|-----------|-------------|
| **Prometheus** | Scrapes and stores metrics from all cluster components |
| **Grafana** | Visualizes metrics via dashboards |
| **Alertmanager** | Routes firing alerts to Slack, email, PagerDuty, etc. |
| **node-exporter** | Exposes per-node CPU, memory, disk, network metrics |
| **kube-state-metrics** | Exposes Kubernetes object states (pod restarts, deployment health) |
| **Prometheus Operator** | Manages Prometheus config via Kubernetes CRDs |

---

## Prerequisites

| Tool | Version |
|------|--------|
| Kubernetes | 1.25+ |
| Helm | 3.x |
| cert-manager | Installed with `letsencrypt-prod` ClusterIssuer |
| Traefik | Installed as ingress controller |

Deploy [k8s-cert-manager](https://github.com/thomasPeelmanUCLL/k8s-cert-manager) first.

---

## Secrets

**Settings → Secrets and variables → Actions:**

| Secret | Description |
|--------|-------------|
| `KUBECONFIG` | Your cluster kubeconfig |

For K3s clusters:
```bash
cat /etc/rancher/k3s/k3s.yaml
# Replace 127.0.0.1 with your cluster's public IP
```

---

## Before Deploying

1. Replace all `yourdomain.com` in `helm/values.yaml` with your real domain
2. Optionally set `grafana.adminPassword` to a strong password (or leave empty for auto-generated)
3. Make sure the `letsencrypt-prod` ClusterIssuer is Ready:
   ```bash
   kubectl get clusterissuer letsencrypt-prod
   # READY should be True
   ```
4. Enable cert-manager metrics in your cert-manager repo:
   ```yaml
   # k8s-cert-manager/helm/values.yaml
   prometheus:
     enabled: true
   ```

---

## Deploying

### Via GitHub Actions
1. **Actions → Deploy kube-prometheus-stack → Run workflow**
2. Leave version blank for latest, or enter e.g. `69.3.0`
3. First deploy takes ~3–5 minutes

### Locally
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f helm/values.yaml \
  --timeout 10m
```

> ⚠️ First install is slower due to CRD creation. Do not cancel mid-install.

---

## Accessing Services

### Via Ingress
- **Grafana:** `https://grafana.yourdomain.com`
- Prometheus & Alertmanager ingress is disabled by default — enable in `helm/values.yaml` if needed

### Via port-forward
```bash
# Grafana — http://localhost:3000
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring

# Prometheus — http://localhost:9090
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring

# Alertmanager — http://localhost:9093
kubectl port-forward svc/kube-prometheus-stack-alertmanager 9093:9093 -n monitoring
```

### Get Grafana admin password
```bash
kubectl get secret kube-prometheus-stack-grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode && echo
```

---

## cert-manager Dashboard

The cert-manager dashboard (Grafana ID [20340](https://grafana.com/grafana/dashboards/20340)) is automatically provisioned in the **cert-manager** folder in Grafana.

### What it shows
- ✅ Certificate expiry countdowns per certificate
- ✅ ACME challenge success/failure rates
- ✅ Certificate renewal history
- ✅ Controller error rates

### How metrics get into Prometheus

Prometheus scrapes cert-manager via `additionalScrapeConfigs` in `helm/values.yaml`. cert-manager exposes metrics on port `9402`.

**Verify cert-manager is being scraped:**
```bash
# 1. Check cert-manager exposes a metrics port
kubectl get svc -n cert-manager
# Should see port 9402

# 2. Check Prometheus targets
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring
# Open http://localhost:9090/targets and look for 'cert-manager'
```

**If you see no data in the dashboard:**
1. Make sure `prometheus.enabled: true` in `k8s-cert-manager/helm/values.yaml`
2. Redeploy cert-manager via its workflow
3. Wait ~1 scrape interval (30s) and reload the Grafana dashboard

---

## Grafana Note

This chart includes its own Grafana (`grafana.enabled: true` by default).

| Option | When to use |
|--------|------------|
| `grafana.enabled: true` | Default — full stack in one deploy |
| `grafana.enabled: false` | You already have k8s-grafana deployed |

If you disable it, add this to your k8s-grafana `helm/values.yaml`:
```yaml
datasources:
  datasources.yaml:
    datasources:
      - name: Prometheus
        type: prometheus
        url: http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090
        isDefault: true
```

---

## Updating

> ⚠️ kube-prometheus-stack updates often include **CRD changes**. Always read the [upgrade notes](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack#upgrading-chart) before merging update PRs.

Version priority order:
1. Workflow dispatch input (highest)
2. `chartVersion` in `helm/values.yaml`
3. Live latest from Helm repo (lowest)

The `update-check.yml` workflow runs every Monday at 08:00 UTC and opens a PR when a newer chart version is available.

---

## Custom Alerts

Custom alert rules live in `alerts/custom-rules.yaml` as `PrometheusRule` CRDs.

```bash
kubectl apply -f alerts/custom-rules.yaml
```

The Prometheus Operator picks them up automatically (no restart needed).

### Pre-included rules

| Alert | Severity | Condition |
|-------|----------|----------|
| `NodeNotReady` | critical | Node NotReady >2 min |
| `NodeMemoryHigh` | warning | Memory usage >85% for >5 min |
| `CertificateExpiringSoon` | critical | Cert expires in <7 days |
| `PodCrashLooping` | warning | Pod restarting frequently |

### Add Slack alerting

Uncomment and fill in the Slack section in `helm/values.yaml`:

```yaml
alertmanager:
  config:
    receivers:
      - name: slack
        slack_configs:
          - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK'
            channel: '#alerts'
```

Then set the default receiver to `slack` in the `route` section.

---

## Helm Values Reference

| Key | Default | Description |
|-----|---------|-------------|
| `chartVersion` | `""` | Pin chart version; empty = latest |
| `grafana.enabled` | `true` | Deploy Grafana with the stack |
| `grafana.adminPassword` | `""` | Empty = auto-generated |
| `grafana.persistence.size` | `5Gi` | Grafana PVC size |
| `prometheus.prometheusSpec.retention` | `30d` | Metrics retention period |
| `prometheus.prometheusSpec.storageSpec` | `20Gi` | Prometheus PVC size |
| `prometheus.prometheusSpec.additionalScrapeConfigs` | cert-manager | Extra scrape jobs |
| `nodeExporter.enabled` | `true` | Node-level OS/hardware metrics |
| `kubeStateMetrics.enabled` | `true` | K8s object state metrics |
| `alertmanager.config` | null receiver | Alert routing config |

---

## Troubleshooting

### Pods not starting
```bash
kubectl get pods -n monitoring
kubectl describe pod <pod-name> -n monitoring
kubectl logs <pod-name> -n monitoring
```

### PVCs stuck in Pending
```bash
kubectl get pvc -n monitoring
kubectl describe pvc <pvc-name> -n monitoring
kubectl get storageclass  # check dynamic provisioning is available
```

### CRD conflicts on upgrade
```bash
helm show crds prometheus-community/kube-prometheus-stack \
  | kubectl apply --server-side -f -
```

### cert-manager not showing in Grafana
```bash
# Check cert-manager metrics port exists
kubectl get svc -n cert-manager

# Check Prometheus is scraping it
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring
# http://localhost:9090/targets → look for cert-manager job

# Check Prometheus scrape errors
# http://localhost:9090/targets → look for any 'DOWN' targets
```

### Grafana dashboard shows no data
1. Confirm cert-manager target is UP in Prometheus
2. Check the dashboard time range (top right) — set to Last 1h
3. Check the datasource in the dashboard is set to `Prometheus`

### ServiceMonitor not being scraped
```bash
kubectl get servicemonitor -A
# Verify labels match prometheusSpec.serviceMonitorSelector
```

---

## Useful Commands

```bash
# All monitoring pods
kubectl get pods -n monitoring

# All Helm releases in monitoring
helm list -n monitoring

# Show currently deployed values
helm get values kube-prometheus-stack -n monitoring

# Uninstall stack (keeps PVCs and CRDs)
helm uninstall kube-prometheus-stack -n monitoring

# Remove CRDs (WARNING: destroys all Prometheus/Alertmanager configs)
kubectl get crds | grep monitoring.coreos.com | awk '{print $1}' | xargs kubectl delete crd

# Apply custom alert rules
kubectl apply -f alerts/custom-rules.yaml

# Check loaded PrometheusRules
kubectl get prometheusrule -n monitoring

# Check all ServiceMonitors
kubectl get servicemonitor -A

# Restart Prometheus
kubectl rollout restart statefulset/prometheus-kube-prometheus-stack-prometheus -n monitoring

# Restart Grafana
kubectl rollout restart deployment/kube-prometheus-stack-grafana -n monitoring

# Check certificate status (from cert-manager)
kubectl get certificate -A
```
