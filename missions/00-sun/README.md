# ☀️ Sun · OpenShift + OpenShift AI

> [!NOTE]
> **Did you know?** The Sun contains 99.86 % of all the mass in the solar system. Every planet, moon, asteroid, and comet combined makes up just the remaining 0.14 %. Everything in this Odyssey literally orbits around it.

## Mission Briefing

Every star system needs a sun — and ours is OpenShift. This mission ignites the core: an OCP 4.20 cluster on AWS with Red Hat OpenShift AI 3.4 installed and ready for science. The cluster arrives pre-installed; your job is to configure it, install the AI stack, and hand the keys to the Science Crew. Without a functioning sun, nothing else in the solar system can orbit. Don't rush this one — a solid foundation means all future missions go smoothly. Countdown begins now. 🔥

## Pre-launch Checklist

- `oc` CLI installed on your workstation ([download](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable-4.20/))
- Access to the cluster API URL and kubeadmin credentials (provided by RHDP or your cluster admin)
- OpenShift 4.20 cluster running on AWS (see "Launch Sequence" below for provisioning options)

## Launch Sequence — Obtain Your Cluster

### Option A — Red Hatters: RHDP Catalog

> [!IMPORTANT]
> Order your lab environment from the [Red Hat Demo Platform](https://catalog.demo.redhat.com/catalog/babylon-catalog-prod?item=babylon-catalog-prod/sandboxes-gpte.sandbox-ocp.prod&utm_source=webapp&utm_medium=share-link) — select **OpenShift on AWS Sandbox**.

When configuring the catalog request, use these settings:

| Setting | Value |
|---------|-------|
| **Activity** | Practice / Enablement — Trying out a technical solution |
| **Cert-manager** | **Uncheck** (do not install cert-manager) |
| **Configure Authentication** | **Check** (enable) |
| **Region** | `eu-central-1` |
| **OpenShift Version** | `4.20` |
| **Control Plane count** | `1` |
| **Control Plane Instance Type** | `m6a.4xlarge` *(the default does not have sufficient compute for RHOAI and related operators)* |

Submit the order and wait for the environment to provision (typically 20–40 minutes). You will receive:
- Cluster console URL
- kubeadmin credentials
- A service account with cluster-admin privileges

### Option B — Everyone else: Bring your own cluster

You need an OpenShift 4.20 cluster on AWS. The cluster must have:
- At least 3 worker nodes with a minimum of `m6a.4xlarge` instance type (or equivalent)
- `cluster-admin` access

If you do not have a cluster, follow the [Installing on AWS IPI](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/installing_on_aws/index) documentation and return here once the cluster is up.

---

## Flight Plan

### Systems Engineering ⚙️

#### 1. Log in to the cluster

```bash
# Export the API URL provided by RHDP or your cluster admin
export OCP_API="https://api.<cluster-name>.<domain>:6443"

# Log in as kubeadmin
oc login "$OCP_API" -u kubeadmin -p <kubeadmin-password>
```

Expected:
```
Login successful.
You have access to N projects, the list has been suppressed.
```

#### 2. Verify cluster health

```bash
oc get nodes
oc get clusteroperators
```

Expected: all nodes `Ready`, all cluster operators `Available=True, Progressing=False, Degraded=False`.

#### 3. Create the RHOAI operator namespace

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: redhat-ods-operator
EOF
```

#### 4. Install the Red Hat OpenShift AI Operator

```bash
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: OperatorGroup
metadata:
  name: rhods-operator
  namespace: redhat-ods-operator
spec:
  upgradeStrategy: Default
---
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
NAME                      DISPLAY                  VERSION   PHASE
rhods-operator.v3.x.x    Red Hat OpenShift AI     3.x.x     Succeeded
```

> [!NOTE]
> This step can take 5–10 minutes as the operator pod pulls its images. The CSV will transition through `Installing` before reaching `Succeeded`.

#### 5. Create the DataScienceCluster

The `DataScienceCluster` enables individual RHOAI components. In this mission we bring up only the core baseline — additional components are activated in the mission that needs them.

```bash
cat <<EOF | oc apply -f -
apiVersion: datasciencecluster.opendatahub.io/v2
kind: DataScienceCluster
metadata:
  name: default-dsc
spec:
  components:
    # --- Sun baseline (enabled now) ---
    dashboard:
      managementState: Managed
    workbenches:
      workbenchNamespace: rhods-notebooks
      managementState: Managed
    aipipelines:
      managementState: Managed
    kserve:
      managementState: Managed
      rawDeploymentServiceConfig: Headless

    # --- Mission 4 · Mars (distributed workloads) ---
    ray:
      managementState: Removed
    trainingoperator:
      managementState: Removed
    kueue:
      managementState: Removed

    # --- Mission 5 · Jupiter (experimentation & tracking) ---
    mlflowoperator:
      managementState: Removed

    # --- Mission 6 · Saturn (model evaluation & guardrails) ---
    trustyai:
      managementState: Removed

    # --- Deep Space (optional stretch goals) ---
    modelregistry:
      managementState: Removed
    feastoperator:
      managementState: Removed
    llamastackoperator:
      managementState: Removed
EOF
```

Monitor progress:
```bash
oc get datasciencecluster default-dsc -o jsonpath='{.status.phase}'
```

Expected: `Ready` (this may take several minutes while all component pods start).

### Science Crew 💻

#### 1. Log in to the OpenShift AI dashboard

```bash
# Get the dashboard route
oc get route rhods-dashboard -n redhat-ods-applications -o jsonpath='{.spec.host}'
```

Open the URL in a browser. Log in with your OpenShift credentials (kubeadmin or your configured identity provider).

#### 2. Create the team project

In the dashboard: **Data Science Projects** → **Create project**

Name it `mission-control` (or your team name). This namespace is used throughout all missions.

#### 3. Verify core sections appear

Confirm the following sections are visible inside the project:
- **Workbenches**
- **Pipelines**
- **Models** (or Model Serving)

## GO / NO-GO Checks

- [ ] GO — `oc get nodes` shows all nodes as `Ready`
- [ ] GO — `oc get clusteroperators` shows all operators `Available=True`
- [ ] GO — `oc get csv -n redhat-ods-operator` shows operator as `Succeeded`
- [ ] GO — `oc get datasciencecluster default-dsc -o jsonpath='{.status.phase}'` returns `Ready`
- [ ] GO — Dashboard route is reachable in a browser
- [ ] GO — Team project `mission-control` exists with Workbenches and Pipelines sections visible

## Abort Conditions & Recovery

| Symptom | Likely cause | Recovery procedure |
|---------|-------------|--------------------|
| `oc login` fails with `x509` error | Self-signed cluster cert | Add `--insecure-skip-tls-verify=true` to the login command; download the CA cert once logged in |
| CSV stays in `Installing` | Operator image pull failure or dependency missing | `oc describe csv -n redhat-ods-operator` → check `reason`; `oc get events -n redhat-ods-operator --sort-by='.lastTimestamp'` |
| `DataScienceCluster` stays in `Progressing` | Component image pull failing or cert-manager missing | `oc get pods -n redhat-ods-applications` — identify pods in `CrashLoopBackOff` or `ImagePullBackOff` and check their logs |
| Dashboard route returns 503 | Dashboard pod not yet running | `oc get pods -n redhat-ods-applications -l app=rhods-dashboard` — wait for `Running`; check events if stuck |

## Mission Success Criteria

An OpenShift 4.20 cluster is running on AWS, the Red Hat OpenShift AI Operator is installed on the `stable-3.x` channel, the `DataScienceCluster` is `Ready`, the dashboard is reachable, and the team project `mission-control` is created.
