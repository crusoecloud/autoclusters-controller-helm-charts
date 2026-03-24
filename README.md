# AutoClusters Controller Helm Charts

Helm charts for automated GPU node health validation on Kubernetes. The system automatically runs GPU diagnostics (DCGM, NVBandwidth) on idle nodes, reports results as Prometheus metrics and Kubernetes node conditions, and enables automated remediation of unhealthy nodes.

Tests only run on GPU nodes with no active customer workloads. Test pods use the `scavenger` PriorityClass (value -1000), so customer workloads always take scheduling priority and will preempt any running tests.

## Charts

| Chart | Description | Version |
|-------|-------------|---------|
| `autoclusters-argo-workflows` | Argo Workflows controller + GPU test WorkflowTemplates + scavenger PriorityClass | 1.1.0 |
| `autoclusters-controller` | Controller that schedules and submits GPU health test workflows on idle nodes | 1.0.0 |
| `crusoe-npd` | Node Problem Detector DaemonSet with GPU health monitoring plugins | 1.0.0 |

## GPU Tests

### DCGM Level 2 Diagnostics

Runs `dcgmi diag -r 2` on each idle GPU node, covering:

- GPU compute correctness
- HBM memory and ECC checks
- PCIe bandwidth measurement
- NVLink pairwise bandwidth

Results are reported per-GPU with subtest-level granularity.

### NVBandwidth

Runs `nvbandwidth -t device_to_device_memcpy_read_ce` to measure NVLink bandwidth between every GPU pair on a node. Results are compared against per-SKU thresholds:

| SKU Pattern | Threshold (GB/s) |
|-------------|------------------|
| `*a100*sxm*` | 220 |
| `*h100*sxm*` | 316 |
| `*h200*sxm*` | 316 |
| Default (unknown SKU) | 50 |

> **Warning:** B200 SKUs (`b200*`) are currently skipped — neither DCGM nor NVBandwidth tests will run on these nodes.

> **Note:** PCIe-only GPUs (e.g., L40S) automatically skip the NVBandwidth test. These SKUs have no NVLink interconnect, so device-to-device bandwidth testing is not applicable — DCGM Level 2 already validates PCIe bandwidth. The `GPUNVBandwidthUnhealthy` node condition will still be present on these nodes with status `False` (healthy), but it reflects a skipped test rather than an actual NVBandwidth measurement.

Thresholds are configurable via the `nvlinkThresholds` Helm value.

## Prerequisites

- Kubernetes cluster with [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/index.html) installed
- Helm 3.x
- `kubectl` configured for your cluster

## Installation

> **Note:** If your cluster was created with `--add-ons=autocluster_active_testing`, these charts are already installed. The steps below are for manual installation only.

### Step 1: Add the Helm repository

```bash
helm repo add autoclusters https://crusoecloud.github.io/autoclusters-controller-helm-charts
helm repo update
```

### Step 2: Pre-install Argo Workflows CRDs

Argo Workflows CRDs must be installed before the chart because Helm validates WorkflowTemplate resources during install. The CRDs are too large for client-side apply (262KB annotation limit), so server-side apply is required.

```bash
helm pull autoclusters/autoclusters-argo-workflows --destination /tmp/
tar xzf /tmp/autoclusters-argo-workflows-*.tgz -C /tmp/ \
  autoclusters-argo-workflows/charts/argo-workflows/files/crds/minimal/
kubectl apply --server-side \
  -f /tmp/autoclusters-argo-workflows/charts/argo-workflows/files/crds/minimal/
rm -rf /tmp/autoclusters-argo-workflows /tmp/autoclusters-argo-workflows-*.tgz
```

### Step 3: Install Argo Workflows

```bash
helm install autoclusters-argo-workflows autoclusters/autoclusters-argo-workflows \
  -n crusoe-system --create-namespace \
  --set argo-workflows.crds.install=false
```

The `argo-workflows.crds.install=false` flag disables CRD rendering in the chart since they were pre-installed in Step 2.

### Step 4: Install Crusoe NPD

```bash
helm install crusoe-npd autoclusters/crusoe-npd \
  --version 1.0.0 \
  -n crusoe-system
```

### Step 5: Install AutoClusters Controller

```bash
helm install autoclusters-controller autoclusters/autoclusters-controller \
  --version 1.0.0 \
  -n crusoe-system
```

## Configuration

### autoclusters-controller

| Value | Default | Description |
|-------|---------|-------------|
| `replicaCount` | `2` | Number of controller replicas (leader election required for HA) |
| `image.repository` | `ghcr.io/crusoecloud/autoclusters-controller` | Controller container image |
| `image.tag` | `""` | Image tag (defaults to chart appVersion: `1.0.0`) |
| `image.digest` | `""` | Image digest for immutable pinning (overrides tag when set) |
| `controller.schedule` | `"0 * * * *"` | Cron schedule for GPU health tests (default: hourly) |
| `controller.leaderElect` | `true` | Enable leader election (required for multi-replica) |
| `controller.metrics.enabled` | `true` | Enable Prometheus metrics endpoint |
| `controller.metrics.bindAddress` | `":8443"` | Metrics server bind address |
| `debugPort` | `8082` | Debug server port |
| `resources.requests.cpu` | `100m` | CPU request |
| `resources.requests.memory` | `128Mi` | Memory request |
| `resources.limits.cpu` | `500m` | CPU limit |
| `resources.limits.memory` | `512Mi` | Memory limit |
| `nodeSelector` | `{}` | Node selector for controller pods |
| `tolerations` | `[]` | Tolerations for controller pods |

### autoclusters-argo-workflows

| Value | Default | Description |
|-------|---------|-------------|
| `workflowTemplates.enabled` | `true` | Deploy built-in GPU test WorkflowTemplates |
| `images.dcgm.repository` | `ghcr.io/crusoecloud/crusoe-dcgm` | DCGM diagnostic image |
| `images.dcgm.tag` | `"1.0.0"` | DCGM image tag (statically set, not derived from chart appVersion) |
| `images.nvbandwidth.repository` | `ghcr.io/crusoecloud/nvbandwidth` | NVBandwidth test image |
| `images.nvbandwidth.tag` | `"1.0.0"` | NVBandwidth image tag (statically set, not derived from chart appVersion) |
| `nvlinkThresholds` | *(see below)* | Per-SKU NVLink bandwidth thresholds (GB/s) |
| `defaultNvlinkThreshold` | `50` | Fallback threshold for unrecognized SKUs (GB/s) |
| `skippedSKUPatterns` | `["b200*"]` | SKU glob patterns to skip all tests |
| `nvbandwidthSkippedSKUPatterns` | `["*l40s*"]` | SKU glob patterns to skip only the NVBandwidth test (PCIe-only GPUs) |
| `priorityClass.value` | `-1000` | Priority value for test pods (lower = more preemptible) |

**NVLink thresholds** default configuration:

```yaml
nvlinkThresholds:
  - pattern: "*a100*sxm*"
    threshold: 220
  - pattern: "*h100*sxm*"
    threshold: 316
  - pattern: "*h200*sxm*"
    threshold: 316
```

Patterns are matched case-insensitively against the node's `node.kubernetes.io/instance-type` label. PCIe variants (not matching `*sxm*`) will use the `defaultNvlinkThreshold`.

### crusoe-npd

| Value | Default | Description |
|-------|---------|-------------|
| `image.repository` | `ghcr.io/crusoecloud/crusoe-npd` | NPD container image |
| `image.tag` | `""` | Image tag (defaults to chart appVersion: `1.0.0`) |
| `port` | `20258` | Metrics/health port (avoids conflict with standard NPD on 20256) |
| `workflowNamespace` | `crusoe-system` | Namespace where Argo Workflows run |
| `pluginDefaults.invokeInterval` | `"60s"` | How often NPD checks workflow results |
| `tolerations` | `[{operator: Exists}]` | Runs on all nodes by default |
| `nodeSelector` | `{}` | Optional node selector to restrict NPD placement |
| `monitors` | *(2 entries)* | GPU health monitor definitions (see below) |

**Default monitors:**

| Monitor | Condition Type | Healthy Reason | Unhealthy Reason |
|---------|---------------|----------------|------------------|
| `gpu-dcgm-diag` | `GPUDCGMUnhealthy` | `GPUDCGMTestPassed` | `GPUDCGMTestFailed` |
| `nvbandwidth` | `GPUNVBandwidthUnhealthy` | `NVBandwidthTestPassed` | `NVBandwidthTestFailed` |

## Observability

### Prometheus Metrics

The controller exposes metrics on port 8443. Node-level metrics are available on all replicas; test metrics are only emitted by the leader.

**Node metrics:**

| Metric | Type | Description |
|--------|------|-------------|
| `autoclusters_node_gpu_total` | Gauge | Total number of GPU nodes in the cluster |
| `autoclusters_node_idle_gpu_count` | Gauge | Number of GPU nodes currently idle |

**Test metrics** (labels: `node`, `node_pool_id`, `instance_id`, `instance_type`, `test_type`, `test_name`):

| Metric | Type | Description |
|--------|------|-------------|
| `autoclusters_test_runs_total` | Counter | Total test runs (additional `status` label: success/failure) |
| `autoclusters_test_duration_seconds` | Histogram | Test run duration in seconds |
| `autoclusters_test_result` | Gauge | Latest test result (1=pass, 0=fail) |
| `autoclusters_test_failure_alert` | Gauge | Set to 1 on failure, 0 on pass (for alerting) |
| `autoclusters_test_node_health_status` | Gauge | Overall node health (1=healthy, 0=unhealthy) |
| `autoclusters_test_last_test_timestamp_seconds` | Gauge | Unix timestamp of last test run |
| `autoclusters_test_created_total` | Counter | Total number of tests created |
| `autoclusters_test_running_current` | Gauge | Number of currently running tests |
| `autoclusters_test_completed_total` | Counter | Completed tests (additional `result` label) |
| `autoclusters_test_failures_total` | Counter | Total test failure count |

**NVBandwidth metrics** (labels: `node`, `node_pool_id`, `instance_id`, `instance_type`):

| Metric | Type | Description |
|--------|------|-------------|
| `autoclusters_test_nvbandwidth_min_gbps` | Gauge | Minimum NVLink bandwidth across all GPU pairs (GB/s) |
| `autoclusters_test_nvbandwidth_threshold_gbps` | Gauge | Threshold used for pass/fail comparison (GB/s) |
| `autoclusters_test_nvbandwidth_failed_pairs` | Gauge | Number of GPU pairs below threshold |

**DCGM metrics:**

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `autoclusters_test_dcgm_failed_gpu_count` | Gauge | node, node_pool_id, instance_id, instance_type | Number of GPUs that failed diagnostics |
| `autoclusters_test_dcgm_gpu_status` | Gauge | + `gpu_id` | Per-GPU result (1=pass, 0=fail) |
| `autoclusters_test_dcgm_subtest_status` | Gauge | + `gpu_id`, `subtest` | Per-GPU per-subtest result |
| `autoclusters_test_dcgm_pcie_bandwidth_gbps` | Gauge | + `gpu_id`, `direction` | Per-GPU PCIe bandwidth (gpu_to_host, host_to_gpu, bidirectional) |

### Node Conditions

NPD translates test results into Kubernetes node conditions:

| Condition | Status: True | Status: False |
|-----------|-------------|---------------|
| `GPUDCGMUnhealthy` | DCGM diagnostics failed | DCGM diagnostics passed |
| `GPUNVBandwidthUnhealthy` | NVBandwidth test failed | NVBandwidth test passed (or skipped on PCIe-only SKUs) |

Check node conditions:

```bash
kubectl get nodes -o custom-columns=\
'NODE:.metadata.name,'\
'DCGM:.status.conditions[?(@.type=="GPUDCGMUnhealthy")].status,'\
'NVBW:.status.conditions[?(@.type=="GPUNVBandwidthUnhealthy")].status'
```

## Upgrading

```bash
helm repo update

helm upgrade autoclusters-argo-workflows autoclusters/autoclusters-argo-workflows \
  -n crusoe-system --set argo-workflows.crds.install=false

helm upgrade crusoe-npd autoclusters/crusoe-npd \
  -n crusoe-system

helm upgrade autoclusters-controller autoclusters/autoclusters-controller \
  -n crusoe-system
```

Upgrade in the same order as installation: Argo Workflows first, then NPD, then the controller.

## Uninstallation

Remove charts in reverse order:

```bash
helm uninstall autoclusters-controller -n crusoe-system
helm uninstall crusoe-npd -n crusoe-system
helm uninstall autoclusters-argo-workflows -n crusoe-system
```

Argo Workflows CRDs are not removed by `helm uninstall` (they are configured with `keep: true`). To remove them manually:

```bash
kubectl delete crd workflows.argoproj.io workflowtemplates.argoproj.io \
  cronworkflows.argoproj.io clusterworkflowtemplates.argoproj.io \
  workflowtasksets.argoproj.io workflowtaskresults.argoproj.io \
  workflowartifactgctasks.argoproj.io workfloweventbindings.argoproj.io
```

## Docker Images

Container images are published to GitHub Container Registry:

| Image | Description |
|-------|-------------|
| `ghcr.io/crusoecloud/autoclusters-controller` | Controller binary |
| `ghcr.io/crusoecloud/crusoe-npd` | NPD with GPU health check plugin |
| `ghcr.io/crusoecloud/crusoe-dcgm` | DCGM diagnostic image |
| `ghcr.io/crusoecloud/nvbandwidth` | NVBandwidth test image |

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
