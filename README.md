# AutoClusters Controller Helm Charts

Helm charts for the AutoClusters active GPU testing system.

## Charts

| Chart | Description | Version |
|-------|-------------|---------|
| `autoclusters-controller` | GPU node health test controller | 0.1.0 |
| `crusoe-npd` | Node Problem Detector with GPU health plugins | 0.1.0 |
| `autoclusters-argo-workflows` | Argo Workflows + WorkflowTemplates + PriorityClass | 0.1.0 |

## Installation

Charts are published as OCI artifacts to `ghcr.io/crusoecloud/autoclusters-controller-helm-charts`.

### Install Argo Workflows (with GPU test templates)

```bash
helm install autoclusters-argo-workflows \
  oci://ghcr.io/crusoecloud/autoclusters-controller-helm-charts/autoclusters-argo-workflows \
  --version 0.1.0 \
  -n crusoe-system --create-namespace
```

### Install AutoClusters Controller

```bash
helm install autoclusters-controller \
  oci://ghcr.io/crusoecloud/autoclusters-controller-helm-charts/autoclusters-controller \
  --version 0.1.0 \
  -n crusoe-system
```

### Install Crusoe NPD

```bash
helm install crusoe-npd \
  oci://ghcr.io/crusoecloud/autoclusters-controller-helm-charts/crusoe-npd \
  --version 0.1.0 \
  -n crusoe-system
```

## Docker Images

Container images are built from the private `autoclusters-controller` repo and published to:

| Image | Description |
|-------|-------------|
| `ghcr.io/crusoecloud/autoclusters-controller` | Controller binary |
| `ghcr.io/crusoecloud/crusoe-npd` | NPD with GPU health check script |

## CI/CD

On merge to `main`:
1. All charts are linted with `helm lint`
2. Charts are packaged and pushed to the OCI registry

Required CI variables: `GHCR_USERNAME`, `GHCR_TOKEN`
