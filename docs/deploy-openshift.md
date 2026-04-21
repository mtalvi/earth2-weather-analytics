# Deploying Earth-2 Weather Analytics on Red Hat OpenShift AI

This guide walks through deploying the Earth-2 Weather Analytics Blueprint on an
OpenShift cluster.  All OpenShift-specific resources are gated behind
`openshift.enabled: false` in the Helm chart, so vanilla Kubernetes deployments
are completely unaffected.

## 1. What We're Deploying

The Earth-2 Weather Analytics Blueprint is a geospatial data analysis service
that combines AI weather forecasting with Omniverse visualization.  On OpenShift
we deploy the **Data Federation Mesh (DFM)** backend and optionally the
**FourCastNet NIM** for AI inference.

| Component | Image | GPU | Purpose |
|-----------|-------|-----|---------|
| **process** | earth2-weather-analytics-process | — | FastAPI gateway; accepts pipeline requests |
| **execute** | earth2-weather-analytics-execute | — | Worker; runs adapter graph (data fetch, xarray, NIM calls) |
| **scheduler** | earth2-weather-analytics-scheduler | — | Timing / batching coordinator |
| **redis** | redis/redis-stack-server:7.2.0-v11 | — | Pub/sub message bus between services |
| **FourCastNet NIM** | nvcr.io/nim/nvidia/fourcastnet:1.0.0 | 1 × GPU | AI global weather forecasting |

### Data Flow

```
Client / E2CC
      │
      ▼  HTTP :8080
   process  ──── Redis pub/sub ────  scheduler
      │                                  │
      └──────── Redis pub/sub ───── execute
                                       │
                                       ▼
                               FourCastNet NIM
```

### FourCastNet NIM

FourCastNet is deployed as a standard Kubernetes Deployment.  The NIM
container handles model downloading internally at startup (no separate
caching step).  The Helm chart also includes optional NIM Operator templates
(`NIMCache` + `NIMService`) for environments where the NIM Operator is
installed, but note that FourCastNet's container does not include the
`download-to-cache` entrypoint that the NIM Operator expects (that mechanism
is designed for LLM NIMs).  The raw Deployment is the recommended path.

## 2. Recommended Hardware

| Resource | Minimum | Notes |
|----------|---------|-------|
| GPU nodes | 1 × NVIDIA L40S 48 GB (or equivalent) | For FourCastNet NIM |
| CPU | 8 cores | For DFM services + Redis |
| Memory | 32 GB | |
| Storage | 128 GB NVMe SSD | Cache PVC (RWX) + NIM model PVC |
| OpenShift | 4.14+ | With GPU Operator and (optionally) NIM Operator |

API keys required:

- **NGC API key** — for pulling the FourCastNet container and model weights
- **ESRI API key** *(optional)* — for topographic / wind data pipelines

## 3. What's Different from Upstream

| Area | Upstream Default | OpenShift Overlay | Impact |
|------|-----------------|-------------------|--------|
| External access | `kubectl port-forward` | OpenShift Route to process service | Persistent URL, no manual port-forward |
| NIM deployment | Raw Deployment + Service | NIM Operator (NIMCache + NIMService) | Managed model caching, GPU scheduling |
| Streamer | Enabled (`hostNetwork: true`) | **Disabled** | WebRTC streaming unavailable (requires `hostnetwork` SCC) |
| Image pull policy | `Never` (local MicroK8s builds) | `IfNotPresent` | Images pulled from a container registry |
| Process service | Headless (`clusterIP: None`) | Normal ClusterIP | Required for Route targeting |
| Security | No SCC configured | `anyuid` SCC RoleBinding | Allows containers to run with their expected UIDs |

## 4. Prerequisites

### CLI Tools

- `oc` (OpenShift CLI) — logged in to your cluster
- `helm` ≥ 3.12

### Cluster Requirements

- **GPU Operator** — installed and functional
  ```bash
  oc get nodes -l nvidia.com/gpu.present=true
  ```
- **NIM Operator** *(if using NIM Operator path)* — provides the
  `apps.nvidia.com/v1alpha1` API
  ```bash
  oc api-versions | grep apps.nvidia.com
  ```
- **RWX-capable StorageClass** — for the `dfm-cache` PVC
  (`ReadWriteMany`).  Examples: ODF (CephFS), NFS provisioner.
  ```bash
  oc get sc
  ```

### Container Images

The DFM services (process, execute, scheduler) are custom-built images.  You
must build and push them to a registry accessible from your OpenShift cluster:

```bash
# Build the images (from the repo root)
./src/build_images.sh -f

# Tag and push to your registry
for svc in process execute scheduler; do
  podman tag earth2-weather-analytics-${svc}:0.1.0 \
    <your-registry>/earth2-weather-analytics-${svc}:0.1.0
  podman push <your-registry>/earth2-weather-analytics-${svc}:0.1.0
done
```

## 5. Configuration Reference

### The `openshift:` Block

| Key | Default | Description |
|-----|---------|-------------|
| `openshift.enabled` | `false` | Master toggle for all OpenShift resources |
| `openshift.route.enabled` | `false` | Create a Route for the process API |
| `openshift.route.host` | `""` | Custom hostname (auto-generated if empty) |
| `openshift.route.tls.termination` | `edge` | `edge`, `passthrough`, or `reencrypt` |
| `openshift.route.tls.insecureEdgeTerminationPolicy` | `Redirect` | How to handle HTTP (non-TLS) traffic |
| `openshift.scc.create` | `false` | Create `anyuid` SCC RoleBinding |
| `openshift.scc.priority` | `10` | SCC priority (higher = preferred) |

### The `nimOperator:` Block

| Key | Default | Description |
|-----|---------|-------------|
| `nimOperator.ngcApiSecretName` | `ngc-catalog-secret` | K8s Secret with NGC API key for model download |
| `nimOperator.fourcastnet.enabled` | `false` | Deploy FourCastNet via NIM Operator |
| `nimOperator.fourcastnet.replicas` | `1` | NIMService replica count |
| `nimOperator.fourcastnet.image.repository` | `nvcr.io/nim/nvidia/fourcastnet` | NIM container image |
| `nimOperator.fourcastnet.image.tag` | `1.0.0` | Image tag |
| `nimOperator.fourcastnet.resources` | 1 GPU | GPU resource requests/limits |
| `nimOperator.fourcastnet.storage.pvc.size` | `50Gi` | Model cache PVC size |
| `nimOperator.fourcastnet.storage.pvc.storageClass` | `""` | StorageClass for model cache (cluster default if empty) |
| `nimOperator.fourcastnet.expose.service.port` | `8086` | Port exposed by the NIMService |

## 6. Deployment

### Create the Namespace

```bash
oc new-project earth2
```

### Create Secrets

Create the image pull secret and NGC API secret **with Helm ownership labels**
so Helm can manage them:

```bash
# Image pull secret (for nvcr.io)
oc create secret docker-registry docker-secret \
  --docker-server=nvcr.io \
  --docker-username='$oauthtoken' \
  --docker-password="${NGC_API_KEY}" \
  -n earth2

oc label secret docker-secret \
  app.kubernetes.io/managed-by=Helm -n earth2
oc annotate secret docker-secret \
  meta.helm.sh/release-name=earth2-weather-analytics \
  meta.helm.sh/release-namespace=earth2 -n earth2

# NGC API secret (for model download)
oc create secret generic ngc-catalog-secret \
  --from-literal=api-key="${NGC_API_KEY}" \
  -n earth2

oc label secret ngc-catalog-secret \
  app.kubernetes.io/managed-by=Helm -n earth2
oc annotate secret ngc-catalog-secret \
  meta.helm.sh/release-name=earth2-weather-analytics \
  meta.helm.sh/release-namespace=earth2 -n earth2
```

Optionally, create the ESRI API secret:

```bash
oc create secret generic esri-arcgis-secret \
  --from-literal=api-key="${DFM_ESRI_API_KEY}" \
  -n earth2
```

### Build Helm Dependencies

```bash
helm dependency build deploy/helm/
```

### Install the Chart

```bash
helm install earth2-weather-analytics deploy/helm/ \
  -f deploy/helm/values.yaml \
  -f deploy/helm/values-openshift.yaml \
  --set image.process.repository=<your-registry>/earth2-weather-analytics-process \
  --set image.execute.repository=<your-registry>/earth2-weather-analytics-execute \
  --set image.scheduler.repository=<your-registry>/earth2-weather-analytics-scheduler \
  -n earth2
```

To simulate NIM Operator presence with `helm template` (for local testing):

```bash
helm template earth2-weather-analytics deploy/helm/ \
  -f deploy/helm/values.yaml \
  -f deploy/helm/values-openshift.yaml \
  --api-versions apps.nvidia.com/v1alpha1 \
  -n earth2
```

## 7. Verification

### Check Pod Status

```bash
oc get pods -n earth2
```

Expected pods (names will include the release name):

| Pod | Status |
|-----|--------|
| `earth2-weather-analytics-process-*` | Running |
| `earth2-weather-analytics-execute-*` | Running |
| `earth2-weather-analytics-scheduler-*` | Running |
| `earth2-weather-analytics-redis-master-0` | Running |
| `earth2-weather-analytics-redis-replicas-0` | Running |

### Check NIM Operator Resources (if enabled)

```bash
oc get nimcache -n earth2
oc get nimservice -n earth2
```

The NIMCache will show a `Ready` status once the model download completes
(this can take 15–30 minutes for FourCastNet).  The NIMService pod will start
after the cache is ready.

### Check the Route

```bash
oc get route -n earth2
```

## 8. Accessing the API

Get the Route URL:

```bash
PROCESS_URL=$(oc get route earth2-weather-analytics-process \
  -n earth2 -o jsonpath='{.spec.host}')
echo "https://${PROCESS_URL}"
```

Test connectivity:

```bash
curl -k "https://${PROCESS_URL}/version"
```

## 9. Testing the Deployment

Pods being `Running` does not mean the pipeline works end-to-end.  Validate
actual functionality using the CLI pipeline scripts and the Swagger UI.

### Quick Smoke Test

```bash
curl -k "https://${PROCESS_URL}/status"
curl -k "https://${PROCESS_URL}/version"
```

### Swagger UI

Open `https://${PROCESS_URL}/docs` in a browser to see the interactive API
documentation.  You can test the `/status` and `/version` endpoints directly
from the UI.

### Set Up the Python Environment

The pipeline scripts require the DFM client library, which needs Python
3.10-3.12.  If your system Python is newer, create a virtual environment:

```bash
python3.12 -m venv .venv
source .venv/bin/activate
pip install -e src/
```

### Run the ERA5 Pipeline (no GPU required)

This fetches GFS weather data and generates a weather map image:

```bash
export ROUTE="https://${PROCESS_URL}"
python src/pipelines/era5.py --site "$ROUTE" --verbose
```

### Run the FourCastNet Pipeline (GPU inference)

This runs end-to-end AI weather forecasting through the FourCastNet NIM:

```bash
python src/pipelines/fourcastnet.py \
  --site "$ROUTE" \
  --variable t2m \
  --date "01/15/25 00:00:00" \
  --verbose
```

Available variables: `t2m` (temperature), `u10m`/`v10m` (wind), `tcwv`
(water vapor), `tp` (precipitation).

### Verify Cache Population

After a successful pipeline run, check that data was written to the cache PVC:

```bash
oc exec deploy/earth2-weather-analytics-execute -n earth2 -- ls /cache/
```

## 10. Visualization: CLI Pipelines vs E2CC 3D Globe

The blueprint includes two ways to interact with the DFM backend:

### CLI Pipelines (available now)

The Python scripts in `src/pipelines/` submit pipeline graphs to the DFM API
and receive weather data and images in response.  This is the primary way to
validate the deployment on OpenShift.  No additional infrastructure is needed
beyond what the Helm chart deploys.

### Earth-2 Command Center / E2CC (requires additional setup)

E2CC is an NVIDIA Omniverse Kit application that renders weather data on an
interactive 3D globe.  It connects to the same DFM API that the CLI scripts
use.  There are two ways to run it:

**Desktop mode:** Run E2CC locally on a workstation with an NVIDIA GPU and
point it at the OpenShift Route URL.  This requires the Omniverse Kit SDK
installed locally.

**Streamer mode:** The Helm chart includes a streamer container
(`streamer.enabled`) that runs E2CC headless inside a pod and streams the
rendered output to a browser via WebRTC.  This is disabled in the OpenShift
overlay because:

1. The streamer requires `hostNetwork: true` for WebRTC media transport,
   which conflicts with OpenShift's default restricted SCC.
2. Building the streamer image requires NVIDIA's internal `linbuild` tool
   and an NVIDIA AI Enterprise (NIAE) license for the base container.  The
   pre-built image is not publicly available on NGC.
3. The streamer requires an additional GPU for RTX rendering.

To enable the streamer once the image is available:

```bash
oc adm policy add-scc-to-user hostnetwork -z default -n earth2
```

Then set `streamer.enabled: true` in your values override and run
`helm upgrade`.

## 11. OpenShift-Specific Challenges and Solutions


### 1. Headless Service Cannot Be a Route Target

**What happened:** OpenShift Routes require a ClusterIP to forward traffic to.
The upstream process Service uses `clusterIP: None` (headless), which has no
virtual IP for the router.

**Error:** Route admitted but returns 503 — no endpoints reachable.

**Fix:** The process Service template now conditionally drops `clusterIP: None`
when `openshift.enabled` is true.  The service becomes a normal ClusterIP
service, which Routes can target.  No impact on vanilla Kubernetes — the
headless behavior is preserved when the flag is false.

### 2. Streamer Requires `hostNetwork: true`

**What happened:** The Omniverse streamer binds ports directly on the host for
WebRTC media transport.  OpenShift's default `restricted` SCC blocks
`hostNetwork`.

**Affected services:** streamer

**Fix:** The streamer is disabled in `values-openshift.yaml`.  If streaming is
required, grant the `hostnetwork` SCC to the service account:

```bash
oc adm policy add-scc-to-user hostnetwork \
  -z default -n earth2
```

Then set `streamer.enabled: true` in your values override.

### 3. Container UID / SCC Constraints

**What happened:** NVIDIA containers typically run as root or a fixed UID.
OpenShift's `restricted` SCC forces containers to run as an arbitrary UID from
the namespace's allocated range, causing permission errors on startup.

**Affected services:** process, execute, scheduler, Redis, NIM

**Fix:** The chart creates an `anyuid` SCC RoleBinding (via
`openshift.scc.create: true`) that binds the `default` service account to the
`system:openshift:scc:anyuid` ClusterRole.  This must exist **before** pods
start — the Helm chart handles this declaratively.

### 4. ReadWriteMany PVC Requires RWX StorageClass

**What happened:** The `dfm-cache` PVC requests `ReadWriteMany` access mode.
Not all OpenShift storage classes support RWX.

**Fix:** Ensure a RWX-capable StorageClass is available (ODF/CephFS, NFS
provisioner).  If only RWO is available, you can change the access mode but
must ensure only one pod mounts the PVC at a time (reduce execute replicas
to 1).

### 5. GPU Scheduling and Tolerations

**What happened:** GPU nodes often carry taints (e.g.,
`nvidia.com/gpu=present:NoSchedule`) that prevent non-GPU pods from landing
on them, and vice versa.

**Fix:** Check GPU node taints and add tolerations if needed:

```bash
oc get nodes -l nvidia.com/gpu.present=true \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.taints}{"\n"}{end}'
```

Add tolerations to the NIM Operator values:

```yaml
nimOperator:
  fourcastnet:
    tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
```

### 6. Image Pull Policy

**What happened:** Upstream defaults use `pullPolicy: Never`, designed for
MicroK8s where images are imported directly.  On OpenShift, images must come
from a registry.

**Fix:** The OpenShift overlay sets `pullPolicy: IfNotPresent` for all DFM
service images.  Users must push images to an accessible registry and pass the
repository via `--set`.

### 7. Helm Secret Ownership

**What happened:** If secrets are created with `oc create secret` before
`helm install`, Helm refuses to adopt them during upgrades because they lack
Helm ownership metadata.

**Fix:** Label and annotate pre-created secrets with Helm ownership (shown in
the Deployment section above).

### 8. NIM Operator Incompatibility with FourCastNet

**What happened:** The NIM Operator's `NIMCache` controller expects a
`download-to-cache` binary in the container image.  This mechanism is designed
for LLM NIMs.  The FourCastNet NIM does not include this binary; it handles
model downloading internally at startup.

**Error:** `CreateContainerError: exec: "download-to-cache": executable file
not found in $PATH`

**Fix:** Deploy FourCastNet as a raw Deployment (`nim.enabled: true`) instead
of through the NIM Operator (`nimOperator.fourcastnet.enabled: false`).  The
`values-openshift.yaml` overlay is configured this way by default.

### 9. RWO Storage and Pod Co-location

**What happened:** When only `ReadWriteOnce` storage is available (e.g., EBS
gp3-csi), multiple pods cannot mount the same PVC from different nodes.  DFM
pods (process, execute, scheduler) all mount `dfm-cache`.

**Error:** `Multi-Attach error for volume`

**Fix:** Set `dfm.cacheAccessMode: ReadWriteOnce` and pin all DFM pods to the
same node using `dfm.nodeSelector`.  The `values-openshift.yaml` overlay
demonstrates this.

## 12. Cleanup

### Uninstall the Chart

```bash
helm uninstall earth2-weather-analytics -n earth2
```

### Delete NIMCache PVC (if applicable)

The NIMCache is annotated with `helm.sh/resource-policy: keep`, so it persists
after uninstall to avoid re-downloading the model (which can be 50+ GB).  To
fully clean up:

```bash
oc delete nimcache --all -n earth2
oc delete pvc -l app.kubernetes.io/instance=earth2-weather-analytics -n earth2
```

### Delete the Namespace

```bash
oc delete project earth2
```

---

<div align="center">

| Previous | Next |
|:---------:|:-----:|
| [Troubleshooting](./07_troubleshooting.md) | — |

</div>
