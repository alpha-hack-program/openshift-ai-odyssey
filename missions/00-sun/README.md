# ☀️ Sun · OpenShift + OpenShift AI

> [!IMPORTANT]
> If you are a Red Hatter, you can order a lab environment on the [Red Hat Demo Platform](https://catalog.demo.redhat.com/catalog?item=babylon-catalog-prod/sandboxes-gpte.sandbox-open.prod&utm_source=webapp&utm_medium=share-link). Request environment `Red Hat Open Environments` > `AWS Blank Open Environment`

## Mission Briefing

Every star system needs a sun — and ours is OpenShift. This mission ignites the core: an OCP 4.20 cluster on AWS IPI with Red Hat OpenShift AI 3.4 installed and ready for science. Without a functioning sun, nothing else in the solar system can orbit. Don't rush this one — a solid foundation means all future missions go smoothly. Countdown begins now. 🔥

## Pre-launch Checklist

- AWS account with sufficient quota for the chosen instance types
- `oc`, `aws` CLI, and `openshift-install` (or access to RHDP) available on your workstation
- A pull secret from [console.redhat.com](https://console.redhat.com/openshift/install/pull-secret)
- DNS zone accessible in Route 53 (required for AWS IPI)

## Flight Plan

### Systems Engineering ⚙️

#### 1. Deploy OpenShift 4.20 on AWS IPI

```bash
# Download the installer
curl -L https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable-4.20/openshift-install-linux.tar.gz | tar xz

# Create install config
./openshift-install create install-config --dir=./cluster

# Deploy (takes ~40 min)
./openshift-install create cluster --dir=./cluster --log-level=info
```

Expected output when complete:
```
INFO Install complete!
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.<cluster>.<domain>
INFO Login to the console with user: kubeadmin, and password: <password>
```

#### 2. Verify cluster health

```bash
export KUBECONFIG=./cluster/auth/kubeconfig
oc get nodes
oc get clusteroperators
```

Expected: all nodes `Ready`, all cluster operators `Available=True`.

#### 3. Install the OpenShift AI Operator

```bash
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: rhods-operator
  namespace: redhat-ods-operator
spec:
  channel: stable-3.x
  installPlanApproval: Automatic
  name: rhods-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

Wait for the operator to become ready:
```bash
oc get csv -n redhat-ods-operator -w
```

Expected:
```
rhods-operator.v3.x.x   Red Hat OpenShift AI   3.x.x   Succeeded
```

#### 4. Create the DataScienceCluster

```bash
cat <<EOF | oc apply -f -
apiVersion: datasciencecluster.opendatahub.io/v1
kind: DataScienceCluster
metadata:
  name: default-dsc
spec:
  components:
    dashboard:
      managementState: Managed
    workbenches:
      managementState: Managed
    datasciencepipelines:
      managementState: Managed
    kserve:
      managementState: Managed
      serving:
        managementState: Managed
        name: knative-serving
    modelmeshserving:
      managementState: Managed
EOF
```

```bash
oc get datasciencecluster default-dsc -o jsonpath='{.status.phase}'
```

Expected: `Ready`

#### 5. Verify storage classes

```bash
oc get storageclass
```

Confirm at least one storage class is set as default (marked with `(default)`). If using ODF or a custom provisioner, verify it can provision PVCs:
```bash
oc new-project test-storage
oc apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
  namespace: test-storage
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
EOF
oc get pvc -n test-storage
oc delete project test-storage
```

### Science Crew 💻

#### 1. Log in to the OpenShift AI dashboard

```bash
# Get dashboard URL
oc get route rhods-dashboard -n redhat-ods-applications -o jsonpath='{.spec.host}'
```

Open the URL in a browser. Log in with your OpenShift credentials.

#### 2. Create a project

In the dashboard: **Data Science Projects** → **Create project**

Name it `mission-control` (or your team name). This project will be used throughout all missions.

#### 3. Verify core sections appear

Confirm the following sections are visible in your project:
- **Workbenches**
- **Pipelines**
- **Models** (or Model Serving)

## GO / NO-GO Checks

- [ ] GO — `oc get nodes` shows all nodes as `Ready`
- [ ] GO — `oc get clusteroperators` shows all operators `Available=True`
- [ ] GO — `oc get csv -n redhat-ods-operator` shows operator as `Succeeded`
- [ ] GO — `oc get datasciencecluster default-dsc -o jsonpath='{.status.phase}'` returns `Ready`
- [ ] GO — Dashboard route is reachable in a browser
- [ ] GO — Team project exists in the dashboard with Workbenches and Pipelines sections visible

## Abort Conditions & Recovery

| Symptom | Likely cause | Recovery procedure |
|---------|-------------|--------------------|
| `openshift-install` times out at bootstrap | VPC/security group misconfiguration or quota issue | Run `openshift-install gather bootstrap --dir=./cluster` to collect logs; check EC2 console for failed instances |
| CSV stays in `Installing` | Operator dependency issue or image pull failure | `oc describe csv -n redhat-ods-operator` → look for `reason`; check `oc get events -n redhat-ods-operator` |
| `DataScienceCluster` stays in `Progressing` | Component image pull or dependency (e.g. cert-manager) not ready | `oc get pods -n redhat-ods-applications` and check for `CrashLoopBackOff` or `ImagePullBackOff` |
| Dashboard route returns 503 | Dashboard pod not yet running | `oc get pods -n redhat-ods-applications -l app=rhods-dashboard` — wait for `Running` |

## Mission Success Criteria

OCP 4.20 is running on AWS, the Red Hat OpenShift AI Operator is installed on the `stable-3.x` channel, the `DataScienceCluster` is `Ready`, the dashboard is reachable, and your team project is created.
