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
7. [Grafana vs k8s-grafana repo](#grafana-note)
8. [Updating](#updating)
9. [Custom Alerts](#custom-alerts)
10. [Helm Values Reference](#helm-values-reference)
11. [Troubleshooting](#troubleshooting)
12. [Useful Commands](#useful-commands)

---

## Overview

This repo deploys the full Kubernetes monitoring stack in one Helm chart:
- **Prometheus** — collects and stores metrics
- **Grafana** — dashboards and visualization
- **Alertmanager** — routes alerts to Slack, email, etc.
- **node-exporter** — node-level OS/hardware metrics
- **kube-state-metrics** — Kubernetes object health metrics
- **Prometheus Operator** — manages Prometheus via Kubernetes CRDs

---

## Prerequisites

| Tool | Version |
|------|--------|
| Kubernetes | 1.25+ |
| Helm | 3.x |
| cert-manager | Installed (for TLS) |
| Traefik | Installed as ingress controller |

---

## Secrets

**Settings → Secrets and variables → Actions:**

| Secret | Description |
|--------|-------------|
| `KUBECONFIG` | Your cluster kubeconfig (K3s: `cat /etc/rancher/k3s/k3s.yaml`) |

---

## Before Deploying

Edit `helm/values.yaml` and replace all instances of `yourdomain.com` with your real domain.

If you already have [k8s-grafana](https://github.com/thomasPeelmanUCLL/k8s-grafana) deployed, disable the built-in Grafana:
```yaml
grafana:
  enabled: false
```
Then point your existing Grafana at `http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090` as a datasource.

---

## Deploying

### Via GitHub Actions
1. **Actions → Deploy kube-prometheus-stack → Run workflow**
2. Leave version blank for latest, or enter e.g. `69.3.0`

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

> ⚠️ First install takes longer (~3-5 min) due to CRD installation and pod startup.

---

## Accessing Services

### Via Ingress (configured in values.yaml)
- Grafana: `https://grafana.yourdomain.com`
- Prometheus: disabled by default, enable in values.yaml
- Alertmanager: disabled by default, enable in values.yaml

### Via port-forward (no domain needed)
```bash
# Grafana
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring

# Prometheus
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring

# Alertmanager
kubectl port-forward svc/kube-prometheus-stack-alertmanager 9093:9093 -n monitoring
```

### Get Grafana admin password
```bash
kubectl get secret kube-prometheus-stack-grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode && echo
```

---

## Grafana Note

This chart includes its own Grafana. You have two options:

| Option | When to use |
|--------|------------|
| Use built-in Grafana (`grafana.enabled: true`) | Fresh setup, simplest approach |
| Disable built-in Grafana (`grafana.enabled: false`) | You already deployed k8s-grafana separately |

If disabling, add this datasource to your k8s-grafana `values.yaml`:
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

> ⚠️ kube-prometheus-stack updates often include **CRD changes**. Always check the [upgrade notes](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack#upgrading-chart) before merging update PRs.

Version priority:
1. Workflow dispatch input
2. `chartVersion` in values.yaml
3. Live latest from Helm repo

The `update-check.yml` workflow opens a PR every Monday when a newer version exists.

---

## Custom Alerts

Custom alert rules live in `alerts/custom-rules.yaml` as a `PrometheusRule` CRD.
Apply them with:
```bash
kubectl apply -f alerts/custom-rules.yaml
```

Pre-included rules:
- `NodeNotReady` — node down for >2m
- `NodeMemoryHigh` — memory >85% for >5m
- `CertificateExpiringSoon` — cert expires in <7 days (integrates with cert-manager)
- `PodCrashLooping` — pod restarting frequently

To route alerts to Slack, edit the `alertmanager.config` section in `helm/values.yaml`.

---

## Helm Values Reference

| Key | Default | Description |
|-----|---------|-------------|
| `chartVersion` | `""` | Pin version; empty = latest |
| `grafana.enabled` | `true` | Deploy Grafana with the stack |
| `grafana.adminPassword` | `""` | Empty = auto-generated |
| `prometheus.prometheusSpec.retention` | `30d` | Metrics retention period |
| `prometheus.prometheusSpec.storageSpec` | 20Gi PVC | Persistent storage for metrics |
| `nodeExporter.enabled` | `true` | Node-level metrics |
| `kubeStateMetrics.enabled` | `true` | K8s object metrics |

---

## Troubleshooting

### Pods not starting
```bash
kubectl get pods -n monitoring
kubectl describe pod <pod-name> -n monitoring
kubectl logs <pod-name> -n monitoring
```

### PVCs pending
```bash
kubectl get pvc -n monitoring
# Check your storageClass supports dynamic provisioning
kubectl get storageclass
```

### CRD conflicts on upgrade
```bash
# Manually apply CRDs before upgrading
helm show crds prometheus-community/kube-prometheus-stack | kubectl apply --server-side -f -
```

### ServiceMonitor not being scraped
```bash
# Check Prometheus targets
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring
# Open http://localhost:9090/targets
```

---

## Useful Commands

```bash
# All monitoring pods
kubectl get pods -n monitoring

# All Helm releases
helm list -n monitoring

# Current deployed values
helm get values kube-prometheus-stack -n monitoring

# Uninstall (keeps PVCs)
helm uninstall kube-prometheus-stack -n monitoring

# Remove CRDs (WARNING: destructive)
kubectl get crds | grep monitoring.coreos.com | awk '{print $1}' | xargs kubectl delete crd

# Check PrometheusRules loaded
kubectl get prometheusrule -n monitoring

# Check ServiceMonitors
kubectl get servicemonitor -n monitoring -A

# Restart Prometheus
kubectl rollout restart statefulset/prometheus-kube-prometheus-stack-prometheus -n monitoring
```
