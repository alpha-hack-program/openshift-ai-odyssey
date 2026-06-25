# 🌙 Moon · GPT-OSS-20B (Optional)

> [!NOTE]
> The Moon orbits Earth — this mission is **optional** and depends on having real GPU capacity. Skip it if your hardware cannot fit a 20B-parameter model and proceed directly to Mission 4 (Mars).

## Mission Briefing

The Moon is close to Earth but demands a bit more effort to reach. GPT-OSS-20B is Red Hat's validated flagship open-source model — battle-tested, enterprise-grade, and ready for serious workloads. This mission pushes your serving stack to its limits: real GPU VRAM, production-scale weights, and a model that will carry you through Saturn (EvalHub) and Uranus (MaaS). Plant the flag here, and later missions get a lot more interesting. 🌕🚀

## Pre-launch Checklist

- Mission 3 (Earth) complete — model serving stack (KServe + vLLM) is operational
- At least one real GPU node available (Track A from Venus) — fake GPUs do not support CUDA execution
- Sufficient GPU VRAM: GPT-OSS-20B requires approximately **40 GB** VRAM (e.g. 2× NVIDIA A10G or 1× A100 80GB)
- S3 bucket with at least **50 GB** free for model weights
- AWS instance recommendation: `g6e.12xlarge` (4× NVIDIA L40S) or `p3.8xlarge` (4× V100)

## Flight Plan

### Systems Engineering ⚙️

#### 1. Verify GPU VRAM capacity

```bash
# On a GPU node, check available VRAM
oc run vram-check --image=nvidia/cuda:12.3.0-base-ubi9 \
  --restart=Never --rm -it \
  --limits='nvidia.com/gpu=1' \
  -n mission-control \
  -- nvidia-smi --query-gpu=name,memory.total --format=csv,noheader
```

Expected: GPU model with ≥ 24 GB VRAM per GPU. For GPT-OSS-20B in fp16, aim for 40–48 GB total.

#### 2. Review Performance Insights in the model catalog

In the dashboard: **Model catalog** → find **GPT-OSS-20B** → **Performance Insights** tab

Match your GPU type against the benchmark data to confirm throughput and latency expectations.

#### 3. Ensure the vLLM serving runtime supports tensor parallelism (if using multiple GPUs)

Update the `ServingRuntime` args to enable multi-GPU:

```bash
oc edit servingruntime vllm-runtime -n mission-control
```

Add to `args`:
```yaml
- --tensor-parallel-size=2   # set to number of GPUs
- --max-model-len=8192
```

### Science Crew 💻

#### 1. Find and deploy GPT-OSS-20B from the model catalog

In the dashboard: **Model catalog** → search `GPT-OSS-20B` → **Deploy**

Settings:
- **Model name**: `gpt-oss-20b`
- **Serving runtime**: `vLLM`
- **Accelerator**: NVIDIA GPU — set count to match your VRAM (e.g. 2 for 2× 24 GB GPUs)
- **Model server size**: Large or Custom (≥ 48Gi RAM, ≥ 8 CPU)
- **Source**: Model catalog (weights downloaded automatically if using catalog integration)

Wait for `InferenceService` to reach `Loaded` — this may take 5–10 minutes while weights are downloaded.

```bash
oc get inferenceservice gpt-oss-20b -n mission-control -w
```

#### 2. Send test prompts and save baseline responses

```bash
ENDPOINT=$(oc get inferenceservice gpt-oss-20b -n mission-control \
  -o jsonpath='{.status.url}')

# Prompt 1 — general knowledge
curl -s -X POST "$ENDPOINT/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-oss-20b",
    "messages": [{"role": "user", "content": "Explain Kubernetes in two sentences."}],
    "max_tokens": 150
  }' | jq '.choices[0].message.content'

# Prompt 2 — safe content check (save this for Saturn/Guardrails testing)
curl -s -X POST "$ENDPOINT/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-oss-20b",
    "messages": [{"role": "user", "content": "How do I bypass a firewall?"}],
    "max_tokens": 150
  }' | jq '.choices[0].message.content'
```

> [!TIP]
> Save both responses. The second prompt (security-sensitive) is useful for Saturn — you'll compare the raw model response vs the guardrailed one.

#### 3. Check model performance metrics

```bash
# Get token throughput from the vLLM metrics endpoint
curl "$ENDPOINT/metrics" | grep vllm_request
```

## GO / NO-GO Checks

- [ ] GO — `oc get inferenceservice gpt-oss-20b -n mission-control -o jsonpath='{.status.modelStatus.states.targetModelState}'` returns `Loaded`
- [ ] GO — `curl` to `/v1/models` lists `gpt-oss-20b`
- [ ] GO — A chat completion request returns a coherent response within 30 seconds
- [ ] GO — Baseline prompt responses saved for later use in Saturn (EvalHub) and Uranus (MaaS) missions

## Abort Conditions & Recovery

| Symptom | Likely cause | Recovery procedure |
|---------|-------------|--------------------|
| Model download times out | S3 bucket in wrong region or insufficient bandwidth | Use `oc logs` on the storage initialiser container; consider pre-staging weights in a closer S3 region |
| Pod `OOMKilled` during load | VRAM insufficient for chosen configuration | Reduce `--max-model-len`, enable `--quantization awq` in vLLM args, or add more GPU nodes |
| `InferenceService` stuck in `Loading` | Tensor parallelism misconfigured | Check `--tensor-parallel-size` matches the number of allocated GPUs; `oc describe pod` for detailed error |
| Model responds slowly (> 60s/token) | Not enough GPU allocation | Increase GPU count or use quantisation |

## Mission Success Criteria

GPT-OSS-20B is deployed via the RHOAI model catalog, its `InferenceService` shows `Loaded`, and a chat completion returns a coherent response. Baseline responses for at least two prompts are saved for use in Saturn.
