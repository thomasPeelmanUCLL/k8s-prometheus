# k8s-prometheus

GitOps-style deployment of the full Kubernetes monitoring stack via [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack).

## What gets deployed

| Component | Purpose |
|-----------|---------|
| **Prometheus** | Metrics collection & storage |
| **Grafana** | Dashboards & visualization |
| **Alertmanager** | Alert routing (Slack, email, etc.) |
| **node-exporter** | Per-node OS/hardware metrics |
| **kube-state-metrics** | Kubernetes object state metrics |
| **Prometheus Operator** | Manages Prometheus via CRDs |

> **Note:** This deploys its own Grafana. If you already have [k8s-grafana](https://github.com/thomasPeelmanUCLL/k8s-grafana) deployed, set `grafana.enabled: false` in `helm/values.yaml` to avoid conflicts.

## Repository Structure

```
.
├── .github/
│   └── workflows/
│       ├── deploy.yml
│       └── update-check.yml
├── helm/
│   └── values.yaml
├── alerts/
│   └── custom-rules.yaml     # Custom PrometheusRule examples
├── docs/
│   └── wiki.md
└── README.md
```

## Quick Start

### Prerequisites

- `kubectl` configured against your cluster
- `helm` v3 installed
- cert-manager + `letsencrypt-prod` ClusterIssuer ready (for TLS)
- GitHub Actions secret `KUBECONFIG` set

### Deploy via GitHub Actions

1. **Actions → Deploy kube-prometheus-stack → Run workflow**
2. Optionally pin a chart version (blank = latest)

### Local Deploy

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f helm/values.yaml
```

## Accessing services

```bash
# Grafana
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring

# Prometheus UI
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring

# Alertmanager UI
kubectl port-forward svc/kube-prometheus-stack-alertmanager 9093:9093 -n monitoring
```

Or use the configured Ingress (see `helm/values.yaml`).

## Default Grafana credentials

```bash
kubectl get secret kube-prometheus-stack-grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode && echo
```

## License

MIT
