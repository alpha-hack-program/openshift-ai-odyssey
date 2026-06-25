# Mission 5 — Jupiter · Experimenting

> [!NOTE]
> **Did you know?** Jupiter's Great Red Spot is a storm that has been raging for at least 350 years — and it is wide enough to swallow three Earths. Despite being the largest planet, Jupiter has the shortest day: it completes one full rotation in just under 10 hours. It also acts as the solar system's bouncer, its gravity deflecting or capturing many comets and asteroids that might otherwise head toward the inner planets.

## Mission Briefing

Jupiter is the largest planet in the solar system — and this mission is the broadest in the Odyssey. You've crossed the asteroid belt. Now the Science Crew takes the helm. Jupiter is where experiments run at scale: Kubeflow Pipelines for reproducibility, MLflow for tracking, and Training Hub for fine-tuning. Connect the dots between data, models, and metrics. This is where good science becomes great engineering. 🔭🧪

## Pre-launch Checklist

- Mission 4 (Mars) complete — Kueue and distributed training are operational
- `datasciencepipelines` component enabled in `DataScienceCluster` (confirmed in Sun mission)
- S3-compatible storage for pipeline artifacts (AWS S3, MinIO, or ODF — configured in Earth or Mars)
- `MLFLOW_TRACKING_URI` accessible from inside the cluster

## Flight Plan

### Systems Engineering ⚙️

#### 1. Verify pipelines and MLflow components

```bash
oc get datasciencecluster default-dsc \
  -o jsonpath='{.spec.components.datasciencepipelines.managementState}'
```

Expected: `Managed`

```bash
# Verify pipeline backend pods
oc get pods -n redhat-ods-applications | grep -E 'pipeline|mlflow'
```

#### 2. Connect S3 storage for pipeline artifacts

In the dashboard: **Data Science Projects** → `mission-control` → **Data connections** → **Add data connection**

Fill in your S3 credentials (key, secret, region, endpoint, bucket). This connection is used by both pipelines and MLflow.

#### 3. Wire the MLflow tracking URI into the namespace

```bash
# Get the MLflow route
MLFLOW_URL=$(oc get route mlflow -n redhat-ods-applications -o jsonpath='{.spec.host}')
echo "http://$MLFLOW_URL"

# Create a ConfigMap so notebooks and jobs can pick it up
cat <<EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: mlflow-config
  namespace: mission-control
data:
  MLFLOW_TRACKING_URI: "http://$MLFLOW_URL"
EOF
```

#### 4. Expose MLflow UI for the Science Crew

The MLflow UI is accessible at the route above. Share the URL with the Science Crew.

### Science Crew 💻

#### 1. Run your first Kubeflow Pipeline

In the dashboard: **Data Science Projects** → `mission-control` → **Pipelines** → **Create pipeline**

**Option A — Import a sample YAML pipeline:**

Download the sample pipeline from the RHOAI examples:
```bash
curl -L https://raw.githubusercontent.com/kubeflow/pipelines/master/samples/core/condition/condition.py \
  -o condition_pipeline.py
```

Or use the RHOAI Python SDK to compile a pipeline:

```python
from kfp import dsl, compiler

@dsl.component(base_image='python:3.11')
def prepare_data(message: str) -> str:
    return f"Jupiter data prepared: {message}"

@dsl.component(base_image='python:3.11')
def train_model(data: str) -> str:
    import time
    time.sleep(5)
    return f"Model trained on: {data}"

@dsl.pipeline(name='jupiter-sample-pipeline')
def sample_pipeline(message: str = 'Hello from Jupiter!'):
    prep = prepare_data(message=message)
    train_model(data=prep.output)

compiler.Compiler().compile(sample_pipeline, 'jupiter_pipeline.yaml')
```

Upload `jupiter_pipeline.yaml` via the dashboard and create a run.

**Monitor the run:**
```bash
oc get pipelinerun -n mission-control -w
```

Expected final state: `Succeeded`

#### 2. Run a Training Hub experiment

In the dashboard: **Training Hub** → **Create training run**

Settings:
- **Base model**: a small HuggingFace model (e.g. `facebook/opt-125m` or from the model catalog)
- **Training type**: SFT (Supervised Fine-Tuning) or LoRA
- **Dataset**: use a small sample dataset (or upload a JSONL file with 50–100 examples)
- **Compute**: use the Kueue `training-queue` from Mars

While the Training Hub job runs:
```bash
oc get trainingjob -n mission-control -w
```

#### 3. Track experiments in MLflow

In your workbench, create a notebook:

```python
import mlflow
import os

mlflow.set_tracking_uri(os.environ.get("MLFLOW_TRACKING_URI", "http://mlflow-route"))
mlflow.set_experiment("jupiter-experiments")

# Log a simulated run
with mlflow.start_run(run_name="manual-experiment-1"):
    mlflow.log_param("model", "opt-125m")
    mlflow.log_param("lr", 0.001)
    mlflow.log_param("epochs", 3)
    mlflow.log_metric("train_loss", 2.34)
    mlflow.log_metric("eval_loss", 2.41)
    print("Run logged to MLflow")
```

Open the MLflow UI and navigate to the `jupiter-experiments` experiment — verify the run appears with all logged params and metrics.

#### 4. (Stretch) Chain Training Hub into a pipeline step

Add a pipeline step that submits a Training Hub job and waits for completion before logging results to MLflow. This creates a fully reproducible training pipeline.

```python
@dsl.component(base_image='python:3.11', packages_to_install=['mlflow'])
def log_to_mlflow(tracking_uri: str, run_name: str, loss: float):
    import mlflow
    mlflow.set_tracking_uri(tracking_uri)
    with mlflow.start_run(run_name=run_name):
        mlflow.log_metric("final_loss", loss)
        print(f"Logged {run_name} with loss={loss}")
```

## GO / NO-GO Checks

- [ ] GO — `oc get pipelinerun -n mission-control` shows at least one run with `Succeeded` condition
- [ ] GO — MLflow UI shows the `jupiter-experiments` experiment with at least two logged runs
- [ ] GO — Training Hub job reaches `Succeeded` state
- [ ] GO — S3 data connection is visible in the dashboard project

## Abort Conditions & Recovery

| Symptom | Likely cause | Recovery procedure |
|---------|-------------|--------------------|
| Pipeline run stuck in `Running` | Component step failing or image pull error | `oc get pods -n mission-control -l pipelines.kubeflow.org/v2_component=true` → check failing pod logs |
| MLflow logs not appearing | Wrong `MLFLOW_TRACKING_URI` in the notebook | Print `mlflow.get_tracking_uri()` in the notebook — ensure it matches the route |
| Training Hub job `Pending` indefinitely | Kueue queue from Mars not correctly linked | Verify `LocalQueue` `training-queue` exists in `mission-control`; check quota |
| S3 upload fails in pipeline | IAM permissions on the S3 bucket | Test with `aws s3 cp test.txt s3://<bucket>/test.txt` from a pod; check bucket policy |

## Mission Success Criteria

A Kubeflow Pipeline run completes successfully, a Training Hub experiment finishes, and both appear as tracked runs in the MLflow UI with logged parameters and metrics.

## Bonus Mission Objective

Build a pipeline that: (1) downloads a small dataset, (2) runs a Training Hub fine-tuning job, (3) evaluates the fine-tuned model on a test set, and (4) logs all metrics to MLflow — creating a fully end-to-end reproducible ML workflow.
