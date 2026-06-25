# Mission 3 — Earth · Model Serving

> [!NOTE]
> **Did you know?** Earth is the densest planet in the solar system and the only one not named after a Greek or Roman deity. "Earth" comes from Old English and Germanic roots meaning simply "the ground." It is also, so far, the only confirmed address for intelligence in the universe — though this cluster is working on that.

## Mission Briefing

Earth is the home of life — and this mission brings your cluster to life with AI inference. You'll deploy a language model from the RHOAI model catalog and expose it as an API endpoint. No specific model is required yet (save the GPU monster for the Moon); the goal is to understand the serving stack — KServe, vLLM, and llm-d — and get a model talking. First contact with AI in production. 🌍🤖

## Pre-launch Checklist

- Mission 2 (Venus) complete — at least one GPU node (or fake-GPU) available
- `kserve` component enabled in the `DataScienceCluster` (done in the Sun mission)
- S3-compatible object storage available for model weights (AWS S3, MinIO, or ODF)
- `oc` access with `cluster-admin`

## Flight Plan

### Systems Engineering ⚙️

#### 1. Verify KServe is enabled

```bash
oc get datasciencecluster default-dsc -o jsonpath='{.spec.components.kserve.managementState}'
```

Expected: `Managed`

```bash
oc get pods -n knative-serving
```

Expected: Knative Serving pods `Running`.

#### 2. Configure S3 storage for model weights

Create a secret in your project with S3 credentials:

```bash
oc create secret generic aws-s3-creds \
  --from-literal=AWS_ACCESS_KEY_ID=<your-key-id> \
  --from-literal=AWS_SECRET_ACCESS_KEY=<your-secret> \
  --from-literal=AWS_DEFAULT_REGION=us-east-1 \
  --from-literal=AWS_S3_BUCKET=<your-bucket> \
  --from-literal=AWS_S3_ENDPOINT=https://s3.amazonaws.com \
  -n mission-control

# Label it so RHOAI dashboard recognises it
oc label secret aws-s3-creds \
  opendatahub.io/dashboard=true \
  opendatahub.io/managed=true \
  -n mission-control
```

#### 3. Deploy a vLLM serving runtime

```bash
cat <<EOF | oc apply -f -
apiVersion: serving.kserve.io/v1alpha1
kind: ServingRuntime
metadata:
  name: vllm-runtime
  namespace: mission-control
  annotations:
    openshift.io/display-name: vLLM
spec:
  supportedModelFormats:
    - name: vLLM
      version: "1"
      autoSelect: true
  multiModel: false
  containers:
    - name: kserve-container
      image: quay.io/modh/vllm:rhoai-3.4
      args:
        - --model=/mnt/models
        - --served-model-name={{.Name}}
        - --max-model-len=4096
      resources:
        requests:
          cpu: "4"
          memory: 8Gi
        limits:
          cpu: "8"
          memory: 16Gi
          nvidia.com/gpu: "1"
EOF
```

#### 4. Open network routes

```bash
# If using RawDeployment mode (no Knative), ensure the service route is created
oc get route -n mission-control
```

For Serverless (Knative) mode, routes are created automatically when the `InferenceService` is deployed.

### Science Crew 💻

#### 1. Browse the model catalog and pick a model

In the dashboard: **Model catalog** → filter by `Red Hat AI validated models`

For a CPU-friendly first deploy, choose **TinyLlama-1.1B-Chat** or similar small model.
For GPU, any model in the catalog with matching hardware requirements.

#### 2. Deploy the model via the dashboard

In the dashboard: **Data Science Projects** → `mission-control` → **Models** → **Deploy model**

Settings:
- **Model name**: `earth-model`
- **Serving runtime**: `vLLM` (or `OpenVINO` for CPU)
- **Model server size**: Small
- **Source**: Connect to your S3 bucket and point to the model path
- **Accelerator**: Select GPU profile (if using GPU)

Click **Deploy** and wait for the model server to become `Ready`.

#### 3. Send a test prompt

```bash
# Get the inference endpoint
ENDPOINT=$(oc get inferenceservice earth-model -n mission-control \
  -o jsonpath='{.status.url}')

# Send a prompt (OpenAI-compatible format)
curl -X POST "$ENDPOINT/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "earth-model",
    "messages": [{"role": "user", "content": "What is OpenShift AI in one sentence?"}],
    "max_tokens": 100
  }'
```

Expected: JSON response with `choices[0].message.content` containing a coherent answer.

#### 4. (Optional) Compare with llm-d routing

If llm-d is enabled in your `DataScienceCluster`, you can point requests through the llm-d gateway to benefit from intelligent routing and load balancing. Ask your Systems Engineering crew to expose the llm-d gateway route, then substitute it as the endpoint in the curl above.

## GO / NO-GO Checks

- [ ] GO — `oc get inferenceservice earth-model -n mission-control -o jsonpath='{.status.modelStatus.states.targetModelState}'` returns `Loaded`
- [ ] GO — `oc get route -n mission-control` shows a route for the model (or Knative service URL is set)
- [ ] GO — `curl` to the `/v1/models` endpoint returns a JSON list including `earth-model`
- [ ] GO — A chat completion request returns HTTP `200` with a `choices` array

## Abort Conditions & Recovery

| Symptom | Likely cause | Recovery procedure |
|---------|-------------|--------------------|
| `InferenceService` stays `Pending` | Model download from S3 failed | `oc logs -n mission-control $(oc get pod -n mission-control -l serving.kserve.io/inferenceservice=earth-model -o name)` — check storage initialiser logs |
| Pod `OOMKilled` | Model too large for configured memory limit | Increase memory limit in the `ServingRuntime` or switch to a smaller model |
| `curl` returns 503 | KNative revision not yet ready | `oc get ksvc -n mission-control` — wait for `READY=True`; check Knative serving pod logs |
| Model returns garbled output | Wrong tokeniser or model format | Verify the model format matches the runtime (e.g. vLLM requires HuggingFace safetensors) |

## Mission Success Criteria

At least one model is deployed via the RHOAI dashboard, its `InferenceService` shows `Loaded`, and a `curl` request to the OpenAI-compatible endpoint returns a valid chat completion response.
