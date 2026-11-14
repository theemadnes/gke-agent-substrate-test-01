# Deploying Agent Substrate on Google Kubernetes Engine (GKE)

A hands-on tutorial and curated set of YAML examples for deploying **[Agent Substrate](https://github.com/agent-substrate/substrate)** (`v1.0.0`) onto a **GKE Standard** cluster, running stateful gVisor and microVM Actors, scaling worker capacity with ComputeClasses and CapacityBuffers, and configuring mTLS ingress, zero-trust egress, and external CSI volumes.

## Repository Contents

| Path | Purpose |
| --- | --- |
| [`examples/00-install/substrate-install.yaml`](examples/00-install/substrate-install.yaml) | Declarative `SubstrateInstall` configuration for `ate-setup` |
| [`examples/01-cluster-infra/system-config.yaml`](examples/01-cluster-infra/system-config.yaml) | GKE node `sysctl` settings for high sandbox density (>800 sandboxes/node) |
| [`examples/02-gvisor-workload/`](examples/02-gvisor-workload/) | Namespace, `WorkerPool` (`ate.dev/v1`), and `ActorTemplate` for a stateful gVisor agent |
| [`examples/03-microvm-workload/`](examples/03-microvm-workload/) | `ComputeClass`, `WorkerPool`, and `ActorTemplate` for Cloud Hypervisor microVMs |
| [`examples/04-production-scaling/`](examples/04-production-scaling/) | Auto-provisioned `ComputeClass` (`minimumCapacity`) and standby `CapacityBuffer` |
| [`examples/05-networking-security/`](examples/05-networking-security/) | In-cluster PodCertificate mTLS ingress client, MITM trust bundle projection, and `EgressPolicy` |
| [`examples/06-csi-volumes/`](examples/06-csi-volumes/) | Filestore `StorageClass`, `CSIDriverConfig` (`ate.dev/v1`), and CSI-backed `ActorTemplate` |

---

## 1. How Agent Substrate Works on GKE

Standard Kubernetes deployments bind each agent session to a dedicated Pod, which leaves CPU and RAM allocated while agents wait for user input or LLM responses and hits Kubernetes control-plane scaling limits at high concurrency.

Agent Substrate decouples **logical agent state** from **physical compute**:

- **Atespace**: A globally unique isolation boundary (analogous to a tenant namespace) for Actors, ActorTemplates, and Tags.
- **ActorTemplate**: An immutable workload blueprint (container images with digests, compute limits, volumes, wakeup probes, and snapshot fidelity). *Not* a Kubernetes CRD—managed via `kubectl ate` in the Substrate PostgreSQL store.
- **Actor**: A single stateful agent instance created from an `ActorTemplate`. When suspended, its memory, rootfs, and `durableDir` volumes are snapshotted to Google Cloud Storage (GCS) (`gs://<bucket>/...`).
- **WorkerPool**: A Kubernetes CRD (`ate.dev/v1`) managed by `ate-controller` that maintains warm sandboxed Worker Pods (`ateworker-gvisor` or `ateworker-microvm`).
- **SandboxConfig**: A cluster-scoped CRD (`ate.dev/v1`) naming the gVisor (`runsc`) or microVM (Kata + Cloud Hypervisor) runtime binaries and digests fetched by the `atelet` node DaemonSet.
- **atenet-router & atenet-egress**: Envoy-based ingress and egress gateways. `atenet-router` authenticates callers by SPIFFE ID and wakes suspended Actors on demand; `atenet-egress` enforces default-deny `EgressPolicy` rules and injects secrets outside the sandbox.

---

## 2. Prerequisites & GKE Cluster Requirements

### Cluster & Node Requirements

- **GKE Mode**: **GKE Standard** only (GKE Autopilot is not supported).
- **Kubernetes Version**: **1.37 or later** on both the control plane and every node pool (1.37 serves `PodCertificateRequest` and `ClusterTrustBundle` APIs).
- **Release Channel**: Use `rapid` or `regular`. Do **not** use the `extended` release channel (the installer enables beta PodCertificate APIs, which Extended does not support).
- **Networking & Identity**: **Workload Identity Federation for GKE** (`--workload-pool=${PROJECT_ID}.svc.id.goog`) and **GKE Dataplane V2** (`--enable-dataplane-v2`) must be enabled.
- **Machine Types**: Use x86 machine series (`c3-standard-4` or larger; e.g., `c3`, `c4`, `n2`, `n4`).
  - Mixed CPU microarchitecture series such as **E2** and **Arm** VMs are **not** supported.
  - Do **not** enable GKE Sandbox (`--sandbox=type=gvisor` or `nodePoolConfig.sandbox.type: microvm`) on Agent Substrate worker node pools—Agent Substrate ships and manages its own gVisor/microVM runtimes.
  - For **microVM** workloads, worker nodes require `/dev/kvm` via `--enable-nested-virtualization` (only configurable at node-pool creation) or x86 bare-metal (`-metal`) machines.

### Local Workstation Tools

> **Important:** Run the installation from a local workstation rather than Cloud Shell—Cloud Shell's 5 GB persistent disk is too small for the checkout and container tooling.

Install the following locally:
- `gcloud` CLI with `kubectl` and `gke-gcloud-auth-plugin` (`gcloud components install kubectl gke-gcloud-auth-plugin`)
- `git`, `go` (1.27+), and `make`
- `docker` with `buildx` (required only if building images from source)
- `sha256sum`, `zstd`, `unzip`, and `jq` (required if staging microVM assets)

Authenticate `gcloud` and Application Default Credentials (ADC):

```bash
gcloud auth login
gcloud auth application-default login
gcloud config set project YOUR_PROJECT_ID
```

Ensure your user or deployment identity holds the required project IAM roles:
- `roles/serviceusage.serviceUsageAdmin`
- `roles/container.admin`
- `roles/storage.admin`
- `roles/artifactregistry.admin`
- `roles/resourcemanager.projectIamAdmin`
- `roles/iam.serviceAccountAdmin`
- `roles/monitoring.editor`

---

## 3. Install Agent Substrate on GKE

Choose **Option A** for a guided setup that provisions GCP resources and deploys the control plane, or **Option B** for an explicit CLI / declarative workflow.

### Option A: Automated Installer (`substrate-gke`, Recommended)

Google maintains the [`ai-on-gke/substrate-gke`](https://github.com/ai-on-gke/substrate-gke) interactive terminal installer, which provisions the GKE Standard 1.37 cluster (or validates an existing one), GCS snapshot/asset buckets, `ate-snapshot-transfer` IAM bindings, Cloud Monitoring dashboards, and the `ate-system` control plane.

```bash
# 1. Clone the GKE installer (or run via curl)
git clone https://github.com/ai-on-gke/substrate-gke.git
cd substrate-gke

# 2. Run preflight checks and preview the wizard without touching GCP
make doctor
make dry-run

# 3. Pre-allow SPIFFE IDs that will call atenet-router
#    Always include ate-system/atenet-client if you will use `kubectl ate proxy actor`
export ATE_INGRESS_ALLOWED_SPIFFE_IDS="spiffe://cluster.local/ns/ate-system/sa/atenet-client,spiffe://cluster.local/ns/agent-client/sa/agent-gateway"

# 4. Run the interactive installer
make run
```

Wizard choices for a standard setup:
1. **Images**: Choose **Pre-built images** (`ghcr.io/agent-substrate/substrate`, tag `v1.0.0`) or **Build from source** (pushes to Artifact Registry `<region>-docker.pkg.dev/<project>/ate-images`).
2. **Cluster**: Create a new GKE 1.37 Rapid cluster (`c3-standard-4`) or select an existing GKE 1.37+ Standard cluster.
3. **Ingress clients**: Confirm the SPIFFE IDs from `ATE_INGRESS_ALLOWED_SPIFFE_IDS`.
4. **Optional steps**: Choose whether to install the Filestore CSI driver, enable node pool autoscaling, or stage the **Micro-VM** runtime assets.

### Option B: Declarative / Manual Installation (`setup-gcp` + `ate-setup`)

If you prefer managing GCP provisioning and Substrate installation as repeatable CLI steps using [`examples/00-install/substrate-install.yaml`](examples/00-install/substrate-install.yaml):

#### Step 1: Provision GCP & GKE prerequisites

Either use `substrate-gke`'s `setup-gcp` tool:

```bash
git clone https://github.com/ai-on-gke/substrate-gke.git
cd substrate-gke
cp tools/setup-gcp/ate-dev-env.sh.example .ate-dev-env.sh
# Edit .ate-dev-env.sh (PROJECT_ID, CLUSTER_NAME, CLUSTER_LOCATION, BUCKET_NAME, etc.)
source .ate-dev-env.sh

go -C tools/setup-gcp run . bootstrap
```

Or create the GKE Standard 1.37 cluster directly with `gcloud` (using [`examples/01-cluster-infra/system-config.yaml`](examples/01-cluster-infra/system-config.yaml) for high sandbox density):

```bash
export PROJECT_ID="YOUR_PROJECT_ID"
export ZONE="us-west1-c"
export CLUSTER_NAME="substrate-cluster"

gcloud services enable \
  container.googleapis.com \
  artifactregistry.googleapis.com \
  storage.googleapis.com \
  logging.googleapis.com \
  monitoring.googleapis.com \
  cloudtrace.googleapis.com \
  telemetry.googleapis.com

gcloud container clusters create "${CLUSTER_NAME}" \
  --location="${ZONE}" \
  --release-channel=rapid \
  --cluster-version=1.37 \
  --machine-type=c3-standard-4 \
  --num-nodes=2 \
  --workload-pool="${PROJECT_ID}.svc.id.goog" \
  --enable-dataplane-v2 \
  --system-config-from-file=examples/01-cluster-infra/system-config.yaml \
  --managed-otel-scope=COLLECTION_AND_INSTRUMENTATION_COMPONENTS
```

> **Note:** If you create the cluster manually with `gcloud` instead of `setup-gcp bootstrap`, also create the snapshot bucket (`gs://<bucket>`), asset bucket (`gs://<bucket>-assets`), and the `ate-snapshot-transfer` Workload Identity IAM bindings documented in [`setup-gcp`](https://github.com/ai-on-gke/substrate-gke/blob/main/tools/setup-gcp/README.md#what-create-iam-actually-grants).

#### Step 2: Deploy the Agent Substrate control plane

Clone the upstream `v1.0.0` Substrate tree (required so `ate-setup` can read the deployment manifests), update placeholders in [`examples/00-install/substrate-install.yaml`](examples/00-install/substrate-install.yaml), and deploy:

```bash
git clone --depth 1 --branch v1.0.0 https://github.com/agent-substrate/substrate.git
cd substrate

go run ./cmd/ate-setup deploy ate-system \
  --config /path/to/this-repo/examples/00-install/substrate-install.yaml
```

### Verify Control Plane & Install `kubectl-ate`

1. Verify all control-plane Pods are `Running`:

```bash
kubectl get pods -n ate-system
kubectl get pods -n podcertificate-controller-system
kubectl get sandboxconfigs
```

2. Install the `kubectl-ate` CLI plugin from the matching `v1.0.0` release:

```bash
git clone --depth 1 --branch v1.0.0 https://github.com/agent-substrate/substrate.git
cd substrate
go install ./cmd/kubectl-ate
export PATH="$PATH:$(go env GOPATH)/bin"

kubectl ate --version
```

3. Mint a 24-hour bearer token for `ate-api-server` (recommended to avoid 1-hour expiry and `--token-file` warnings):

```bash
kubectl create token -n ate-system ate-client \
  --audience api.ate-system.svc \
  --duration=24h > ~/.ate-token
```

---

## 4. Deploy Your First Stateful Agent (gVisor)

This walkthrough uses [`examples/02-gvisor-workload/`](examples/02-gvisor-workload/) to deploy a warm gVisor `WorkerPool`, register an `ActorTemplate`, and verify process memory + filesystem persistence across a suspend/resume cycle.

### Step 1: Deploy the `WorkerPool`

```bash
kubectl apply -f examples/02-gvisor-workload/worker-pool.yaml
```

Wait for the 2 warm gVisor worker Pods to become ready:

```bash
kubectl get workerpools -n agent-demo
kubectl ate --token-file ~/.ate-token get workers -n agent-demo
```

### Step 2: Create the Atespace and `ActorTemplate`

`ActorTemplate` is a Substrate API object (not a Kubernetes CRD), so it is applied with `kubectl ate`:

```bash
kubectl ate --token-file ~/.ate-token create atespace agent-demo
kubectl ate --token-file ~/.ate-token create actor-template \
  -f examples/02-gvisor-workload/actor-template.yaml
```

Wait until the template's golden snapshot finishes building (`GOLDEN TAG` populates):

```bash
kubectl ate --token-file ~/.ate-token get actor-templates -a agent-demo
```

### Step 3: Create, Wake, Suspend, and Resume an Actor

1. **Create an Actor** (starts in `ACTOR_STATE_SUSPENDED`, consuming zero worker CPU/RAM):

```bash
kubectl ate --token-file ~/.ate-token create actor my-agent-1 \
  -a agent-demo \
  --template=python-agent-template

kubectl ate --token-file ~/.ate-token get actors -a agent-demo
```

2. **Start a local mTLS proxy** to `atenet-router` (requires `spiffe://cluster.local/ns/ate-system/sa/atenet-client` in `ingressAllowedSPIFFEIDs`):

```bash
kubectl ate --token-file ~/.ate-token proxy actor my-agent-1 18088:8888 -a agent-demo &
PROXY_PID=$!
```

3. **Send an HTTP request**—`atenet-router` automatically claims a warm worker Pod, restores the Actor, and waits for `wakeupProbe` before forwarding the request:

```bash
curl -sS -X POST -H "Content-Type: application/json" \
  -d '{"command": "sh -c \"echo persisted-from-gvisor > /data/state.txt && cat /data/state.txt\""}' \
  http://127.0.0.1:18088/execute
```

Verify the Actor is now `ACTOR_STATE_RUNNING` on a worker Pod:

```bash
kubectl ate --token-file ~/.ate-token get actors -a agent-demo
```

4. **Suspend the Actor** synchronously to GCS (`--mode=sync` waits for the snapshot upload to finish before returning and frees the worker slot):

```bash
kubectl ate --token-file ~/.ate-token suspend actor my-agent-1 \
  -a agent-demo \
  --mode=sync

kubectl ate --token-file ~/.ate-token get actors -a agent-demo
```

5. **Send another request**—the Actor wakes onto any available worker Pod with both memory and `/data/state.txt` intact:

```bash
curl -sS -X POST -H "Content-Type: application/json" \
  -d '{"command": "cat /data/state.txt"}' \
  http://127.0.0.1:18088/execute
```

6. **Inspect logs or clean up the Actor**:

```bash
kubectl ate --token-file ~/.ate-token logs actor my-agent-1 -a agent-demo
kubectl ate --token-file ~/.ate-token delete actor my-agent-1 -a agent-demo
kill "${PROXY_PID}"
```

---

## 5. Deploy MicroVM Workloads (Cloud Hypervisor + Kata)

MicroVM Workers run each Actor inside a dedicated Cloud Hypervisor VM and request the `ate.dev/kvm: 1` extended resource advertised by `atelet` on nodes with `/dev/kvm`.

### Step 1: Pre-create 0-Node Nested Virtualization Pools (Scale-from-Zero)

Because GKE node auto-provisioning cannot infer `ate.dev/kvm` from an empty unprovisioned template, pre-create 0-node pools labeled with `clusterautoscaler-nodetemplate-resources.ate.dev/kvm=4096`:

```bash
# Primary N4 pool (0 nodes, scales up to 10)
gcloud container node-pools create microvm-n4-pool \
  --cluster="${CLUSTER_NAME}" \
  --location="${ZONE}" \
  --machine-type=n4-standard-4 \
  --enable-nested-virtualization \
  --system-config-from-file=examples/01-cluster-infra/system-config.yaml \
  --num-nodes=0 \
  --enable-autoscaling --total-min-nodes=0 --total-max-nodes=10 \
  --node-labels=cloud.google.com/compute-class=substrate-microvm-zero,clusterautoscaler-nodetemplate-resources.ate.dev/kvm=4096 \
  --node-taints=cloud.google.com/compute-class=substrate-microvm-zero:NoSchedule

# Fallback C3 pool (0 nodes, scales up to 10)
gcloud container node-pools create microvm-c3-pool \
  --cluster="${CLUSTER_NAME}" \
  --location="${ZONE}" \
  --machine-type=c3-standard-4 \
  --enable-nested-virtualization \
  --system-config-from-file=examples/01-cluster-infra/system-config.yaml \
  --num-nodes=0 \
  --enable-autoscaling --total-min-nodes=0 --total-max-nodes=10 \
  --node-labels=cloud.google.com/compute-class=substrate-microvm-zero,clusterautoscaler-nodetemplate-resources.ate.dev/kvm=4096 \
  --node-taints=cloud.google.com/compute-class=substrate-microvm-zero:NoSchedule
```

### Step 2: Apply ComputeClass, WorkerPool, and ActorTemplate

```bash
kubectl apply -f examples/03-microvm-workload/compute-class.yaml
kubectl apply -f examples/03-microvm-workload/worker-pool-microvm.yaml

kubectl ate --token-file ~/.ate-token create atespace agent-microvm-demo
kubectl ate --token-file ~/.ate-token create actor-template \
  -f examples/03-microvm-workload/actor-template-microvm.yaml
```

Verify that `atelet` discovers `/dev/kvm` once a node boots:

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.capacity.ate\.dev/kvm}{"\n"}{end}'
```

---

## 6. Production Infrastructure & Burst Scaling

For production clusters, isolate the Substrate database and control plane from high-churn worker nodes and keep standby worker capacity warm:

### Isolate PostgreSQL and System Control-Plane Node Pools

Create dedicated pools before installing with `cordonSystemInfraComponents: true` in [`examples/00-install/substrate-install.yaml`](examples/00-install/substrate-install.yaml):

```bash
# Single-zone node pool for the PostgreSQL StatefulSet
gcloud container node-pools create ate-postgres-pool \
  --cluster="${CLUSTER_NAME}" \
  --location="${ZONE}" \
  --node-locations="${ZONE}" \
  --machine-type=c3-standard-4 \
  --num-nodes=1 \
  --node-labels=ate.dev/workloadType=ate-postgres \
  --node-taints=ate.dev/workloadType=ate-postgres:NoSchedule

# Dedicated pool for atenet-router, ate-api-server, atenet-egress, ate-controller, ate-idp-server
gcloud container node-pools create ate-system-infra-pool \
  --cluster="${CLUSTER_NAME}" \
  --location="${ZONE}" \
  --machine-type=c3-standard-8 \
  --num-nodes=2 \
  --node-labels=ate.dev/workloadType=ate-system-infra \
  --node-taints=ate.dev/workloadType=ate-system-infra:NoSchedule
```

### Auto-Created ComputeClass + Standby CapacityBuffer

- [`examples/04-production-scaling/compute-class-min-capacity.yaml`](examples/04-production-scaling/compute-class-min-capacity.yaml): Auto-provisions `n4-standard-4` -> `c3-standard-4` -> `n2-standard-4` nodes with nested virtualization, high-density `sysctls`, and `minimumCapacity.targetNodeCount: 1`.
- [`examples/04-production-scaling/capacity-buffer-standby.yaml`](examples/04-production-scaling/capacity-buffer-standby.yaml): Pre-provisions and pre-warms nodes with `atelet` assets, then suspends the VMs (`buffer.gke.io/standby-capacity`) so burst Worker Pods resume in ~50–85 seconds end-to-end at a fraction of running compute cost.

```bash
kubectl apply -f examples/04-production-scaling/compute-class-min-capacity.yaml
kubectl apply -f examples/04-production-scaling/capacity-buffer-standby.yaml
```

> **Important:**
> - Ensure the `atelet` DaemonSet tolerates `buffer.gke.io/standby-capacity-node-suspended` (`operator: "Exists"`) when using standby buffers.
> - Avoid Spot VMs for stateful `WorkerPool` nodes—Spot preemption gives only 15s–2m notice before termination, whereas standard GKE drains allow the Worker's 30-minute graceful `SIGTERM` checkpoint window.

---

## 7. In-Cluster Ingress Clients & Zero-Trust Egress

### Call Actors from an In-Cluster Service (`atenet-router`)

In production, application services authenticate to `atenet-router.ate-system.svc:8444` using a `podCertificate` projected volume and verify the router using a `clusterTrustBundle` projected volume. See [`examples/05-networking-security/ingress-client-deployment.yaml`](examples/05-networking-security/ingress-client-deployment.yaml):

```bash
kubectl apply -f examples/05-networking-security/ingress-client-deployment.yaml

kubectl -n agent-client exec deploy/agent-client -- curl --proxytunnel \
  --proxy https://atenet-router.ate-system.svc:8444 \
  --proxy-cacert /run/router-client/router-ca.pem \
  --proxy-cert /run/router-client/credential-bundle.pem \
  --proxy-key /run/router-client/credential-bundle.pem \
  --proxy-header "ate-target-actor: agent-demo/my-agent-1" \
  http://actor-upstream:8888/
```

### Enforce EgressPolicy & Inject Credentials Outside the Sandbox

By default, `atenet-egress` denies all outbound Actor network traffic. To allow an Actor to reach an HTTPS endpoint and receive injected secrets (without ever exposing the secret inside the sandbox):

1. Use an `ActorTemplate` that projects the Substrate MITM CA bundle (`egress-mitm.ate.dev` + `system-roots.ate.dev`) via `systemInfo.dataSources[].trustBundle`:
   [`examples/05-networking-security/actor-template-with-trust-bundle.yaml`](examples/05-networking-security/actor-template-with-trust-bundle.yaml).
2. Grant the Actor's atespace read access to the target Kubernetes Secret namespace in `k8s-credential-provider-namespace-policy`:

```bash
kubectl -n ate-system patch configmap k8s-credential-provider-namespace-policy --type merge -p '
data:
  namespace-policy.yaml: |
    policies:
      - atespace: agent-demo
        allowedNamespaces:
          - ns1
'
kubectl -n ate-system rollout restart deployment/k8s-credential-provider
```

3. Apply the Actor's `EgressPolicy` from [`examples/05-networking-security/egress-policy.yaml`](examples/05-networking-security/egress-policy.yaml):

```bash
kubectl ate --token-file ~/.ate-token create egress-policy my-agent-1 \
  -a agent-demo \
  -f examples/05-networking-security/egress-policy.yaml
```

---

## 8. External CSI Volumes (Google Cloud Filestore)

For workloads needing persistent network file storage attached dynamically across Worker Pods (preview feature gated by `ExternalVolumes`):

1. Install the Filestore CSI driver with Substrate support (either via the `substrate-gke` wizard step or the [Filestore CSI Substrate overlay](https://github.com/kubernetes-sigs/gcp-filestore-csi-driver/tree/master/deploy/kubernetes/overlays/substrate)).
2. Enable `--preview=ExternalVolumes` on `ate-api-server` (`preview: ExternalVolumes` in [`examples/00-install/substrate-install.yaml`](examples/00-install/substrate-install.yaml)).
3. Update placeholders (`PROJECT_ID`, `LOCATION`, `VOLUME_POOL_NAME`) in [`examples/06-csi-volumes/storageclass-filestore.yaml`](examples/06-csi-volumes/storageclass-filestore.yaml) and apply:

```bash
kubectl apply -f examples/06-csi-volumes/storageclass-filestore.yaml
kubectl apply -f examples/06-csi-volumes/csi-driver-config.yaml

kubectl ate --token-file ~/.ate-token --preview create actor-template \
  -f examples/06-csi-volumes/actor-template-csi.yaml
kubectl ate --token-file ~/.ate-token --preview create actor my-csi-agent \
  -a agent-demo \
  --template=python-agent-csi
```

---

## 9. Cleanup & Teardown

### Remove Workload Examples

```bash
kubectl ate --token-file ~/.ate-token delete actor my-agent-1 -a agent-demo
kubectl ate --token-file ~/.ate-token delete actor-template python-agent-template -a agent-demo
kubectl ate --token-file ~/.ate-token delete atespace agent-demo
kubectl delete -f examples/02-gvisor-workload/worker-pool.yaml
```

### Remove Agent Substrate & GCP Resources

1. **Delete the Substrate control plane** (including the PostgreSQL PersistentVolumeClaim so GKE does not orphan a 500 GiB disk):

```bash
# Run the ate-setup delete command printed at install time, or from the substrate checkout:
go run ./cmd/ate-setup delete ate-system

# Wait until the PostgreSQL PV is deleted
kubectl get pv | grep data-postgres-0
```

2. **Delete GCP resources** (from the `substrate-gke` directory):

```bash
# Delete cluster, buckets, IAM bindings, service accounts, and dashboards:
./tools/cleanup-gcp --project "${PROJECT_ID}" --cluster "${CLUSTER_NAME}" --location "${ZONE}" --bucket "${BUCKET_NAME}"

# Or keep an existing GKE cluster and delete only Substrate buckets/IAM:
./tools/cleanup-gcp --project "${PROJECT_ID}" --cluster "${CLUSTER_NAME}" --location "${ZONE}" --bucket "${BUCKET_NAME}" --keep-cluster
```

---

## References

- [About Agent Substrate on GKE](https://docs.cloud.google.com/kubernetes-engine/ai-ml/about-agent-substrate)
- [Install Agent Substrate on GKE](https://docs.cloud.google.com/kubernetes-engine/ai-ml/install-overview-substrate)
- [Design your infrastructure for Agent Substrate on GKE](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/design-substrate-infrastructure)
- [Upstream `agent-substrate/substrate`](https://github.com/agent-substrate/substrate)
- [GKE Installer `ai-on-gke/substrate-gke`](https://github.com/ai-on-gke/substrate-gke)
