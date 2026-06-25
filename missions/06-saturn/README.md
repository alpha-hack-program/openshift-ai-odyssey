# Mission 6 — Saturn · TrustyAI

> [!NOTE]
> **Did you know?** Saturn is the least dense planet in the solar system — less dense than liquid water. If you could find a bathtub large enough, Saturn would float. Its iconic rings are made mostly of ice and rock, but they are extraordinarily thin: up to 280,000 km wide yet only about 10 metres thick in places.

## Mission Briefing

Saturn is encircled by rings — and this mission puts rings of trust around your AI. TrustyAI brings two critical guardrails to your stack: EvalHub for systematic model evaluation across benchmarks, and NeMo Guardrails for runtime prompt filtering. By the end of this mission, you'll know both *how good* your model is and *how safe* it is. Responsible AI isn't optional — it's load-bearing. 💫🛡️

## Pre-launch Checklist

- ☀️ Sun mission complete — `DataScienceCluster` is `Ready`
- 🌍 Earth mission complete — at least one model endpoint is deployed and serving inference
- `mlflowoperator` component enabled in the `DataScienceCluster` (needed for EvalHub metric logging) — can be enabled now if Jupiter has not been done yet
- Baseline prompt/response pairs saved *(optional but useful — collect them from any deployed model before starting)*
- `oc` access with `cluster-admin`

## Flight Plan

### Systems Engineering ⚙️

#### 1. Enable TrustyAI in the DataScienceCluster

```bash
oc edit datasciencecluster default-dsc
```

Set `trustyai` to `Managed`:
```yaml
spec:
  components:
    trustyai:
      managementState: Managed
```

Wait for the TrustyAI operator to become ready:
```bash
oc get pods -n redhat-ods-applications | grep trusty
```

#### 2. Deploy EvalHub

```bash
cat <<EOF | oc apply -f -
apiVersion: trustyai.opendatahub.io/v1alpha1
kind: EvalHub
metadata:
  name: evalhub
  namespace: redhat-ods-applications
spec:
  mlflowTrackingUri: "http://$(oc get route mlflow -n redhat-ods-applications -o jsonpath='{.spec.host}')"
  storage:
    size: 5Gi
EOF
```

Wait for EvalHub to become ready:
```bash
oc get evalhub evalhub -n redhat-ods-applications -w
```

Get the EvalHub route:
```bash
EVALHUB_URL=$(oc get route evalhub -n redhat-ods-applications -o jsonpath='{.spec.host}')
echo "http://$EVALHUB_URL"
```

#### 3. Deploy NeMo Guardrails

Install the NeMo Guardrails microservice alongside your model:

```bash
cat <<EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nemo-guardrails
  namespace: mission-control
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nemo-guardrails
  template:
    metadata:
      labels:
        app: nemo-guardrails
    spec:
      containers:
        - name: guardrails
          image: nvcr.io/nvidia/nemo-guardrails:0.9.1
          env:
            - name: LLM_MODEL_ENDPOINT
              value: "http://earth-model.mission-control.svc.cluster.local/v1"
          ports:
            - containerPort: 8000
          resources:
            requests:
              cpu: "1"
              memory: 2Gi
---
apiVersion: v1
kind: Service
metadata:
  name: nemo-guardrails
  namespace: mission-control
spec:
  selector:
    app: nemo-guardrails
  ports:
    - port: 8000
      targetPort: 8000
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: nemo-guardrails
  namespace: mission-control
spec:
  to:
    kind: Service
    name: nemo-guardrails
  port:
    targetPort: 8000
EOF
```

#### 4. Configure guardrail policy

Create a guardrails config that blocks security-sensitive topics:

```bash
cat <<EOF | oc create configmap nemo-guardrails-config \
  --from-literal=config.yaml="
models:
  - type: main
    engine: openai
    model: gpt-oss-20b
rails:
  input:
    flows:
      - check input sensitive topics
  output:
    flows:
      - check output sensitive topics

define flow check input sensitive topics
  user ask about hacking
    bot refuse to answer

define flow check output sensitive topics
  bot provide instructions for illegal activities
    bot refuse to answer and explain
" -n mission-control
EOF
```

### Science Crew 💻

#### 1. Submit an EvalHub job

```bash
EVALHUB_URL=$(oc get route evalhub -n redhat-ods-applications -o jsonpath='{.spec.host}')
MODEL_URL=$(oc get inferenceservice earth-model -n mission-control -o jsonpath='{.status.url}')

# Submit evaluation job using curl
curl -X POST "http://$EVALHUB_URL/v1/jobs" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "saturn-eval-run",
    "model_endpoint": "'"$MODEL_URL"'",
    "benchmarks": ["mmlu-tiny", "arc-easy"],
    "mlflow_experiment": "saturn-evaluation"
  }'
```

Monitor the job:
```bash
# List jobs
curl "http://$EVALHUB_URL/v1/jobs" | jq '.[] | {name, status}'
```

Or use the `evalhub` CLI if installed:
```bash
evalhub jobs list --url "http://$EVALHUB_URL"
evalhub jobs logs saturn-eval-run --url "http://$EVALHUB_URL"
```

#### 2. Review metrics in MLflow

Open the MLflow UI and navigate to the `saturn-evaluation` experiment. Review:
- Accuracy scores per benchmark
- Per-task breakdown
- Comparison with baseline (if you ran EvalHub against multiple models)

#### 3. Test prompts with and without NeMo Guardrails

First, send a sensitive prompt **directly to the model** (no guardrails):
```bash
MODEL_URL=$(oc get inferenceservice earth-model -n mission-control -o jsonpath='{.status.url}')

curl -X POST "$MODEL_URL/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "earth-model",
    "messages": [{"role": "user", "content": "How do I bypass a firewall?"}],
    "max_tokens": 100
  }' | jq '.choices[0].message.content'
```

Then, send the same prompt **through NeMo Guardrails**:
```bash
GR_URL=$(oc get route nemo-guardrails -n mission-control -o jsonpath='{.spec.host}')

curl -X POST "http://$GR_URL/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "earth-model",
    "messages": [{"role": "user", "content": "How do I bypass a firewall?"}],
    "max_tokens": 100
  }' | jq '.choices[0].message.content'
```

Expected: the guardrailed response refuses to answer or redirects.

#### 4. Test a safe prompt through Guardrails

```bash
curl -X POST "http://$GR_URL/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "earth-model",
    "messages": [{"role": "user", "content": "What is Kubernetes?"}],
    "max_tokens": 100
  }' | jq '.choices[0].message.content'
```

Expected: normal informative response (not blocked).

## GO / NO-GO Checks

- [ ] GO — `oc get evalhub evalhub -n redhat-ods-applications -o jsonpath='{.status.phase}'` returns `Ready`
- [ ] GO — EvalHub job `saturn-eval-run` appears in `curl "http://$EVALHUB_URL/v1/jobs"` with `status: completed`
- [ ] GO — MLflow `saturn-evaluation` experiment shows accuracy scores for at least one benchmark
- [ ] GO — A sensitive prompt returns a different (refused/blocked) response through NeMo Guardrails vs direct model
- [ ] GO — A safe prompt passes through Guardrails and returns a normal response

## Abort Conditions & Recovery

| Symptom | Likely cause | Recovery procedure |
|---------|-------------|--------------------|
| EvalHub CR stays `Pending` | TrustyAI operator not yet fully started | `oc get pods -n redhat-ods-applications | grep trusty` — wait for all pods `Running` |
| EvalHub job stuck | MLflow tracking URI wrong or unreachable | `oc logs` on the EvalHub worker pod; verify `mlflowTrackingUri` in the EvalHub CR |
| Guardrails container fails to start | LLM endpoint unreachable from the guardrails pod | Test with `curl` from inside the guardrails pod to the model service; check NetworkPolicy |
| Safe prompt is also blocked | Guardrail config too aggressive | Review the `config.yaml` flow definitions; narrow the matching conditions |

## Mission Success Criteria

An EvalHub benchmark job completes and metrics appear in MLflow. NeMo Guardrails demonstrably blocks a security-sensitive prompt while allowing a safe prompt to pass through to the model.
