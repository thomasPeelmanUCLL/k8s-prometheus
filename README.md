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

## Pre-provisioned Grafana Dashboards

These dashboards are automatically loaded into Grafana on deploy — no manual importing needed.

| Dashboard | Grafana ID | Folder | What it shows |
|-----------|-----------|--------|---------------|
| **cert-manager** | [20340](https://grafana.com/grafana/dashboards/20340) | cert-manager | Certificate expiry, ACME requests, renewal rates |
| Node Exporter Full | [1860](https://grafana.com/grafana/dashboards/1860) | Kubernetes | CPU, memory, disk, network per node |
| Kubernetes Cluster | [7249](https://grafana.com/grafana/dashboards/7249) | Kubernetes | Cluster-wide pod/node overview |
| Kubernetes Pods | [6417](https://grafana.com/grafana/dashboards/6417) | Kubernetes | Per-pod resource usage |

> cert-manager metrics are scraped automatically via `additionalScrapeConfigs` in `helm/values.yaml`. Make sure `prometheus.enabled` is `true` in your [k8s-cert-manager](https://github.com/thomasPeelmanUCLL/k8s-cert-manager) `helm/values.yaml`.

## Pre-configured Alert Rules

| Alert | Severity | Triggers when |
|-------|----------|---------------|
| `NodeNotReady` | critical | Node down >2 min |
| `NodeMemoryHigh` | warning | Memory >85% for >5 min |
| `CertificateExpiringSoon` | critical | Cert expires in <7 days |
| `PodCrashLooping` | warning | Pod restarting frequently |

## Repository Structure

```
.
├── .github/
│   └── workflows/
│       ├── deploy.yml           # Manual deploy via workflow_dispatch
│       └── update-check.yml     # Weekly version check + auto PR
├── helm/
│   └── values.yaml              # All Helm config
├── alerts/
│   └── custom-rules.yaml        # PrometheusRule CRDs
├── docs/
│   └── wiki.md                  # Full ops & config guide
└── README.md
```

## Quick Start

### Prerequisites

- `kubectl` configured against your cluster
- `helm` v3 installed
- cert-manager installed with a `letsencrypt-prod` ClusterIssuer — see [k8s-cert-manager](https://github.com/thomasPeelmanUCLL/k8s-cert-manager)
- GitHub Actions secret `KUBECONFIG` set

### 1. Configure your domain

Edit `helm/values.yaml` and replace every `yourdomain.com` with your actual domain:

```yaml
grafana:
  ingress:
    hosts:
      - grafana.yourdomain.com
  grafana.ini:
    server:
      root_url: https://grafana.yourdomain.com
```

### 2. Deploy via GitHub Actions

1. **Actions → Deploy kube-prometheus-stack → Run workflow**
2. Leave version blank for latest, or pin e.g. `69.3.0`
3. First deploy takes ~3–5 min (CRD installation)

### 3. Local deploy

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f helm/values.yaml \
  --timeout 10m
```

## Accessing Services

| Service | Ingress URL | Port-forward |
|---------|------------|-------------|
| Grafana | `https://grafana.yourdomain.com` | `kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring` |
| Prometheus | disabled by default | `kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring` |
| Alertmanager | disabled by default | `kubectl port-forward svc/kube-prometheus-stack-alertmanager 9093:9093 -n monitoring` |

### Get Grafana admin password

```bash
kubectl get secret kube-prometheus-stack-grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode && echo
```

## Updating

Every Monday at 08:00 UTC the `update-check.yml` workflow compares the latest chart version to the pinned value in `helm/values.yaml`. If newer, it opens a PR for you to review and merge.

> ⚠️ kube-prometheus-stack updates often include **CRD changes**. Always check the [upgrade notes](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack#upgrading-chart) before merging.

See [docs/wiki.md](docs/wiki.md) for the full operations guide.

## Related Repos

| Repo | Purpose |
|------|---------|
| [k8s-cert-manager](https://github.com/thomasPeelmanUCLL/k8s-cert-manager) | TLS certificates via Let's Encrypt |
| [k8s-grafana](https://github.com/thomasPeelmanUCLL/k8s-grafana) | Standalone Grafana (optional) |

## License

MIT
