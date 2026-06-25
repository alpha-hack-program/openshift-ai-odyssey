# Mission 4 — Mars · Distributed Workloads

> [!NOTE]
> **Did you know?** Mars is home to Olympus Mons, the tallest volcano in the solar system at 21 km high and 600 km wide — roughly the size of France. It is so wide that if you stood on the rim, the opposite edge would be below the horizon. A Martian day (sol) is just 37 minutes longer than an Earth day, which made it easier for NASA to schedule rover operations.

## Mission Briefing

Mars is the last frontier of the inner system — red, rugged, and demanding real engineering muscle. This mission transforms your cluster from a single-model server into a distributed AI training platform. You'll enable Kueue for workload queuing, configure hardware profiles, and submit your first multi-node training job. This is the last stop before the asteroid belt. Cross it only when everything here is working. ⚔️🪐

## Pre-launch Checklist

- ☀️ Sun mission complete — `DataScienceCluster` is `Ready`
- 🌋 Venus mission complete — at least one GPU node pool is available (real GPUs recommended for actual training; fake GPUs work for scheduling tests)
- `cluster-admin` access to enable operators and configure quotas

## Flight Plan

### Systems Engineering ⚙️

#### 1. Enable distributed workloads components in DataScienceCluster

```bash
oc edit datasciencecluster default-dsc
```

Set these components to `Managed`:
```yaml
spec:
  components:
    kueue:
      managementState: Managed
    trainingoperator:
      managementState: Managed
    ray:
      managementState: Managed
```

Wait for all pods to be ready:
```bash
oc get pods -n redhat-ods-applications | grep -E 'kueue|training|ray'
```

#### 2. Create a ClusterQueue and ResourceFlavour

```bash
# Define a resource flavour for GPU nodes
cat <<EOF | oc apply -f -
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata:
  name: gpu-flavor
spec:
  nodeLabels:
    nvidia.com/gpu.present: "true"
EOF

# Create a ClusterQueue with GPU quota
cat <<EOF | oc apply -f -
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: ai-training-queue
spec:
  namespaceSelector: {}
  resourceGroups:
    - coveredResources: ["cpu", "memory", "nvidia.com/gpu"]
      flavors:
        - name: gpu-flavor
          resources:
            - name: cpu
              nominalQuota: "16"
            - name: memory
              nominalQuota: 64Gi
            - name: nvidia.com/gpu
              nominalQuota: "4"
EOF

# Create a LocalQueue in your project namespace
cat <<EOF | oc apply -f -
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata:
  name: training-queue
  namespace: mission-control
spec:
  clusterQueueName: ai-training-queue
EOF
```

#### 3. Configure hardware profiles in RHOAI dashboard

In the dashboard: **Settings** → **Hardware profiles** → **Create hardware profile**

Create a profile named `GPU Training Node`:
- **Identifier**: `nvidia.com/gpu`
- **Count**: `1` (minimum)
- **Tolerations**: Add toleration for GPU taint if nodes are tainted

#### 4. Verify Kueue is admitting workloads

```bash
oc get clusterqueue ai-training-queue
```

Expected:
```
NAME                  COHORT   PENDING WORKLOADS   ADMITTED WORKLOADS
ai-training-queue              0                   0
```

### Science Crew 💻

#### 1. Submit a sample distributed PyTorchJob

```bash
cat <<EOF | oc apply -f -
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: mars-training-job
  namespace: mission-control
  labels:
    kueue.x-k8s.io/queue-name: training-queue
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      restartPolicy: OnFailure
      template:
        spec:
          containers:
            - name: pytorch
              image: quay.io/modh/training:rhoai-3.4
              command:
                - python
                - -c
                - |
                  import torch
                  import torch.distributed as dist
                  dist.init_process_group(backend='gloo')
                  rank = dist.get_rank()
                  world_size = dist.get_world_size()
                  print(f"Rank {rank}/{world_size} — Mars training job online!")
                  dist.barrier()
                  if rank == 0:
                      print("Distributed training complete. All ranks reporting in.")
              resources:
                requests:
                  cpu: "2"
                  memory: 4Gi
    Worker:
      replicas: 1
      restartPolicy: OnFailure
      template:
        spec:
          containers:
            - name: pytorch
              image: quay.io/modh/training:rhoai-3.4
              command:
                - python
                - -c
                - |
                  import torch
                  import torch.distributed as dist
                  dist.init_process_group(backend='gloo')
                  rank = dist.get_rank()
                  world_size = dist.get_world_size()
                  print(f"Rank {rank}/{world_size} — worker node reporting in!")
                  dist.barrier()
              resources:
                requests:
                  cpu: "2"
                  memory: 4Gi
EOF
```

#### 2. Observe Kueue queuing and admission

```bash
# Watch the Kueue Workload object
oc get workloads -n mission-control -w
```

Observe the workload moving through `Pending` → `Admitted` → deleted (once the job completes).

#### 3. Monitor the job

```bash
oc get pytorchjob mars-training-job -n mission-control -w
```

Expected final state: `Succeeded`

```bash
# Check logs from the master
oc logs -n mission-control \
  $(oc get pod -n mission-control -l training.kubeflow.org/job-name=mars-training-job,training.kubeflow.org/replica-type=master -o name)
```

Expected log lines:
```
Rank 0/2 — Mars training job online!
Distributed training complete. All ranks reporting in.
```

## GO / NO-GO Checks

- [ ] GO — `oc get clusterqueue ai-training-queue` shows `ClusterQueue` exists with non-zero quota
- [ ] GO — `oc get localqueue training-queue -n mission-control` exists and references the ClusterQueue
- [ ] GO — `oc get workloads -n mission-control` showed the job moving through `Admitted` state
- [ ] GO — `oc get pytorchjob mars-training-job -n mission-control -o jsonpath='{.status.conditions[-1].type}'` returns `Succeeded`
- [ ] GO — Hardware profile appears in RHOAI dashboard Settings

## Abort Conditions & Recovery

| Symptom | Likely cause | Recovery procedure |
|---------|-------------|--------------------|
| Workload stays `Pending` indefinitely | ClusterQueue quota exhausted or LocalQueue not linked correctly | `oc describe workload -n mission-control` → check `Status.Conditions` for admission reason; verify LocalQueue `clusterQueueName` matches |
| PyTorchJob pods `CrashLoopBackOff` | Training image pull failure or entrypoint error | `oc logs <pod> -n mission-control` — check for Python import errors or image issues |
| `kueue` controller not found | Component not yet `Managed` in DataScienceCluster | `oc get datasciencecluster default-dsc -o jsonpath='{.spec.components.kueue}'` — verify `Managed` and wait for pod |
| `dist.init_process_group` hangs | Network policy blocking inter-pod communication | Check NetworkPolicy in `mission-control` namespace; ensure master/worker pods can reach each other |

## Mission Success Criteria

Kueue ClusterQueue and LocalQueue are configured, a PyTorchJob is submitted, Kueue admits it through the queue, and the job completes with `Succeeded` status. A hardware profile is visible in the RHOAI dashboard.

## Bonus Mission Objective

Submit two PyTorchJobs simultaneously with the ClusterQueue set to only admit one at a time (lower the `nominalQuota` for GPU to `1`). Observe Kueue queue the second job and admit it only after the first completes. This simulates real multi-team resource governance.
