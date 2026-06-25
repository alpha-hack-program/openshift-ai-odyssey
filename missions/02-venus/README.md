# Mission 2 — Venus · GPU Power-up

> [!NOTE]
> **Did you know?** A day on Venus is longer than a year on Venus. It rotates so slowly — and backwards compared to most planets — that the Sun rises in the west and sets in the east. Also, at 465 °C on the surface, it is hotter than Mercury despite being twice as far from the Sun.

## Mission Briefing

Venus is shrouded in heat and mystery — and so is the GPU. This mission electrifies the cluster: you'll either bring real NVIDIA GPU nodes online on AWS, or deploy the fake-gpu-operator to simulate them for learning purposes. Without this mission, model serving stays firmly in CPU territory. Pick your track, fire your thrusters, and get those accelerators online. ⚡🔥

## Pre-launch Checklist

- Mission 1 (Mercury) complete — workbench is running
- AWS quota for GPU instances approved (Track A only) — check `g6` or `p3` limits in your region via the AWS console
- `oc` access with `cluster-admin` privileges

## Flight Plan

---

### Track A — Real GPUs on AWS

#### Systems Engineering ⚙️

##### 1. Add a GPU MachineSet

```bash
# Get the existing worker MachineSet as a template
oc get machineset -n openshift-machine-api -o name | head -1

# Export and edit it
oc get machineset <existing-machineset> -n openshift-machine-api -o yaml > gpu-machineset.yaml
```

Edit `gpu-machineset.yaml`:
- Change `.metadata.name` to `<cluster-id>-gpu-worker-<az>`
- Change `.spec.selector.matchLabels` and `.spec.template.metadata.labels` to match the new name
- Change `.spec.template.spec.providerSpec.value.instanceType` to `g6.2xlarge` (or your preferred GPU instance)
- Set `.spec.replicas` to `1`

Apply:
```bash
oc apply -f gpu-machineset.yaml
oc get machineset -n openshift-machine-api -w
```

Wait until `READY` count equals `DESIRED`.

##### 2. Install Node Feature Discovery (NFD) Operator

```bash
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: nfd
  namespace: openshift-nfd
spec:
  channel: stable
  installPlanApproval: Automatic
  name: nfd
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Create NodeFeatureDiscovery CR
cat <<EOF | oc apply -f -
apiVersion: nfd.openshift.io/v1
kind: NodeFeatureDiscovery
metadata:
  name: nfd-instance
  namespace: openshift-nfd
spec:
  operand:
    image: registry.redhat.io/openshift4/ose-node-feature-discovery
    imagePullPolicy: Always
  workerConfig:
    configData: |
      sources:
        pci:
          deviceClassWhitelist: ["0200","03","12"]
          deviceLabelFields: ["vendor"]
EOF
```

##### 3. Install NVIDIA GPU Operator

```bash
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: gpu-operator-certified
  namespace: nvidia-gpu-operator
spec:
  channel: v24.9
  installPlanApproval: Automatic
  name: gpu-operator-certified
  source: certified-operators
  sourceNamespace: openshift-marketplace
EOF

# Create ClusterPolicy
cat <<EOF | oc apply -f -
apiVersion: nvidia.com/v1
kind: ClusterPolicy
metadata:
  name: gpu-cluster-policy
spec:
  operator:
    defaultRuntime: crio
  driver:
    enabled: true
  toolkit:
    enabled: true
  devicePlugin:
    enabled: true
  dcgmExporter:
    enabled: true
  gfd:
    enabled: true
EOF
```

Wait for `ClusterPolicy` to become `Ready`:
```bash
oc get clusterpolicy gpu-cluster-policy -o jsonpath='{.status.state}'
```

Expected: `ready`

##### 4. Verify `nvidia-smi` from a GPU pod

```bash
oc run gpu-test --image=nvidia/cuda:12.3.0-base-ubi9 \
  --restart=Never --rm -it \
  --limits='nvidia.com/gpu=1' \
  -n mission-control \
  -- nvidia-smi
```

Expected: NVIDIA-SMI output showing the GPU model and driver version.

#### Science Crew 💻

##### 1. Verify accelerator profile in the dashboard

In the dashboard: **Settings** → **Accelerator profiles**

Confirm `NVIDIA GPU` appears as an available profile.

##### 2. Launch a GPU workbench

Create a new workbench in `mission-control`:
- **Image**: `Standard Data Science` (latest)
- **Accelerator**: `NVIDIA GPU` — set count to `1`

Wait for `Running`, then open JupyterLab and run:

```python
import subprocess
result = subprocess.run(['nvidia-smi'], capture_output=True, text=True)
print(result.stdout)
```

Expected: nvidia-smi output inside the notebook.

---

### Track B — Fake GPUs for learning

#### Systems Engineering ⚙️

##### 1. Deploy fake-gpu-operator

```bash
# Clone the repo
git clone https://github.com/run-ai/fake-gpu-operator.git
cd fake-gpu-operator

# Deploy using the provided manifests
kubectl apply -f deployments/fake-gpu-operator/
```

Or use Helm:
```bash
helm repo add fake-gpu-operator https://run-ai.github.io/fake-gpu-operator
helm repo update
helm install fake-gpu-operator fake-gpu-operator/fake-gpu-operator \
  --namespace fake-gpu-operator \
  --create-namespace \
  --set gpu.count=2 \
  --set gpu.vram=24576
```

##### 2. Confirm nodes advertise GPUs

```bash
oc describe node <worker-node> | grep -A5 "Capacity:"
```

Expected: `nvidia.com/gpu: 2` (or your configured count) under `Capacity` and `Allocatable`.

#### Science Crew 💻

##### 1. Schedule a GPU-requesting pod

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: fake-gpu-test
  namespace: mission-control
spec:
  containers:
  - name: test
    image: busybox
    command: ["sleep", "300"]
    resources:
      limits:
        nvidia.com/gpu: "1"
  restartPolicy: Never
EOF

oc get pod fake-gpu-test -n mission-control
```

Expected: Pod reaches `Running` status (fake GPUs are allocated, no CUDA execution happens).

```bash
# Cleanup
oc delete pod fake-gpu-test -n mission-control
```

##### 2. Verify dashboard shows GPU capacity

In the dashboard: **Settings** → **Accelerator profiles** — confirm a GPU profile appears.

## GO / NO-GO Checks

**Track A**
- [ ] GO — `oc get nodes -l nvidia.com/gpu.present=true` returns GPU nodes
- [ ] GO — `oc get clusterpolicy -o jsonpath='{.items[0].status.state}'` returns `ready`
- [ ] GO — `nvidia-smi` from a GPU pod shows driver version and GPU model
- [ ] GO — Accelerator profile visible in RHOAI dashboard Settings

**Track B**
- [ ] GO — `oc describe node <worker> | grep nvidia.com/gpu` shows non-zero capacity
- [ ] GO — Pod requesting `nvidia.com/gpu: 1` reaches `Running` state
- [ ] GO — RHOAI dashboard shows GPU accelerator profile

## Abort Conditions & Recovery

| Symptom | Likely cause | Recovery procedure |
|---------|-------------|--------------------|
| MachineSet stuck in `Provisioning` (Track A) | AWS quota exceeded or wrong instance type | Check EC2 console for provisioning errors; request quota increase for `g6` via AWS Service Quotas |
| `ClusterPolicy` stays `notReady` (Track A) | Driver install failed or node not tainted correctly | `oc get pods -n nvidia-gpu-operator` → find failing pod; `oc logs <pod>` for details |
| `nvidia-smi` fails with "no NVIDIA devices" | GPU Operator toolkit not fully initialised | Wait 5 min after ClusterPolicy `ready` and retry; toolkit validation sometimes lags |
| Fake GPU pod stays `Pending` | fake-gpu-operator DaemonSet not running on target node | `oc get pods -n fake-gpu-operator` — ensure DaemonSet pod is `Running` on target node |

## Mission Success Criteria

At least one node in the cluster exposes `nvidia.com/gpu` capacity, an accelerator profile is visible in the RHOAI dashboard, and a workload (real or fake) successfully requests and is allocated a GPU unit.
