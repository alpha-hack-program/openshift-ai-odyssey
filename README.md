# Red Hat OpenShift AI Odyssey

Your space suit for Red Hat OpenShift AI. Embark on a hands-on journey from zero to MLOps hero through practical missions and challenges! 🚀🪐

![OpenShift AI Odyssey learning path](docs/images/rhoai-learning-path.png)

A hands-on path for small teams. Pair people from different backgrounds so each mission covers both **platform** and **application** work.

Follow the missions from the **Sun** outward — through the inner planets, across the **asteroid belt**, and into the outer system.

Like the real solar system, there is a divide between the **inner** and **outer** planets:

| Zone | Missions | Who leads |
|------|----------|-----------|
| **Sun + inner system** ☀️ | Sun → Mars *(+ optional Moon)* | Mostly **Systems Engineering** ⚙️ — install, configure, and prepare the cluster |
| **Asteroid belt** ☄️ | Between Mars and Jupiter | Transition — foundation is ready; experimentation and science take the wheel |
| **Outer system** 🪐 | Jupiter → Neptune | Mostly **Science Crew** 💻 — pipelines, training, evaluation, MaaS, and agentic apps |
| **Deep Space** 🌌 | Bonus explorations | Stretch goals for teams that finish early |

| Crew | Background | You own… |
|------|------------|----------|
| **Systems Engineering** ⚙️ | OpenShift infra — installation, nodes, storage, operators | Cluster plumbing that makes AI workloads run |
| **Science Crew** 💻 | CI/CD, middleware, app dev | Projects, notebooks, pipelines, models, and APIs |

> [!IMPORTANT]
> If you are a Red Hatter, you can order a lab environment on the [Red Hat Demo Platform](https://catalog.demo.redhat.com/catalog?item=babylon-catalog-prod/sandboxes-gpte.sandbox-open.prod&utm_source=webapp&utm_medium=share-link). Request environment `Red Hat Open Environments` > `AWS Blank Open Environment`

**Prerequisites:** OpenShift Container Platform 4.20 on [AWS IPI](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/installing_on_aws/index), cluster-admin access, and the [OpenShift AI 3.4 docs](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4) handy.

---

### ☀️ Sun · OpenShift + OpenShift AI *(shared · start here)*

Everything orbits from here. Goal: OCP 4.20 on AWS IPI with RHOAI 3.4 running on top.

**Systems Engineering** ⚙️

- Deploy or verify [OpenShift 4.20 on AWS IPI](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/installing_on_aws/index)
- Install the [Red Hat OpenShift AI Operator](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/installing_and_uninstalling_openshift_ai_self-managed) (`stable-3.x` channel — latest GA)
- Create a `DataScienceCluster` with core components (dashboard, workbenches, pipelines, kserve)
- Verify storage classes and PVC provisioning for user workloads

**Science Crew** 💻

- Log in to the OpenShift AI dashboard
- Create a project
- Confirm workbenches and pipelines appear in the UI

**Telemetry**
- `oc get datasciencecluster` shows phase `Ready`
- OpenShift AI dashboard is reachable at its route
- A project appears in the dashboard with workbench and pipeline sections visible

**Flight Notes**
- `oc get csv -n redhat-ods-operator` — verify operator is `Succeeded`
- Dashboard route: `oc get route rhods-dashboard -n redhat-ods-applications -o jsonpath='{.spec.host}'`

✅ **Done when:** Dashboard is reachable and your team project exists.

📂 [Full Mission Dossier](missions/00-sun/README.md)

---

### Mission 1 — Mercury · Notebook playground *(inner system ☀️)*

**Systems Engineering** ⚙️

- Confirm notebook image pull and default workbench sizes
- Verify PVC/storage for home directories
- Check SCCs and resource quotas in the project

**Science Crew** 💻

- Launch a workbench
- Run a short Python notebook (e.g. `pandas`, a simple plot)
- Install one extra package
- Restart the workbench and confirm your work persists

📖 [Getting started — workbenches](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/getting_started_with_red_hat_openshift_ai_self-managed)

**Telemetry**
- Workbench pod shows `Running` in the OpenShift console
- Notebook cell executes without error and renders output
- After restart, the notebook file is still present in the home directory

**Flight Notes**
- `oc get notebooks -n <your-project>` — check workbench CR status
- If image pull fails, verify the `ImageStream` exists in `redhat-ods-applications`

✅ **Done when:** A notebook runs code and survives a restart.

📂 [Full Mission Dossier](missions/01-mercury/README.md)

---

### Mission 2 — Venus · GPU power-up *(inner system ☀️)*

Pick **one** track per cluster (or run both on separate node pools).

#### Track A — Real GPUs on AWS *(Systems Engineering ⚙️ leads)*

**Systems Engineering** ⚙️

- Add GPU worker nodes (e.g. `g6` instances)
- Install [NVIDIA GPU Operator](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/installing_and_uninstalling_openshift_ai_self-managed) + Node Feature Discovery
- Label and taint GPU nodes
- Confirm `nvidia-smi` from a GPU pod

**Science Crew** 💻

- Verify the GPU appears in the OpenShift AI dashboard (accelerator profile)
- Launch a workbench with a GPU resource request
- Confirm the device is visible inside the pod

#### Track B — Fake GPUs for learning *(Systems Engineering ⚙️ leads)*

**Systems Engineering** ⚙️

- Deploy [fake-gpu-operator](https://github.com/run-ai/fake-gpu-operator) on selected worker nodes
- Confirm nodes advertise `nvidia.com/gpu`

**Science Crew** 💻

- Schedule a GPU-requesting workload on a fake-GPU node
- Confirm scheduling succeeds and the dashboard still shows GPU capacity

**Telemetry**
- `oc describe node <gpu-node> | grep nvidia.com/gpu` shows allocatable capacity
- Accelerator profile appears in the RHOAI dashboard under Settings
- Workbench with GPU request starts without `Pending` due to resource constraints

**Flight Notes**
- `oc get clusterpolicy` — verify NVIDIA GPU Operator ClusterPolicy is `Ready`
- For Track B: fake-gpu resources behave like real ones for scheduling; CUDA workloads won't run

✅ **Done when:** At least one node pool exposes GPUs and a workload can request them.

📂 [Full Mission Dossier](missions/02-venus/README.md)

---

### Mission 3 — Earth · Deploy models *(inner system ☀️)*

Learn model serving in general — not tied to one specific model yet.

**Systems Engineering** ⚙️

- Enable model serving (KServe) on the `DataScienceCluster`
- Ensure GPU nodes and enough PVC/object storage for model weights
- Open network routes for inference endpoints
- Deploy or configure serving runtimes: **vLLM**, **KServe**, and **llm-d** (pick at least two)

**Science Crew** 💻

- Open the [model catalog](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/working_with_the_model_catalog)
- Pick a small model suitable for your hardware
- Deploy it with a serving runtime (vLLM or KServe)
- Send a test prompt and capture the response
- *(Optional)* Compare inference behaviour with **llm-d** routing

**Telemetry**
- `InferenceService` CR shows `READY: True`
- `curl` to the inference endpoint returns a valid JSON response
- Model appears as `Deployed` in the RHOAI dashboard

**Flight Notes**
- `oc get inferenceservice -n <your-project>` — check serving status
- Small CPU-friendly models (e.g. TinyLlama-1B) work without GPU for initial testing

✅ **Done when:** At least one model serves inference from the catalog using a chosen runtime.

📂 [Full Mission Dossier](missions/03-earth/README.md)

---

### 🌙 Moon · GPT-OSS-20B *(optional · orbits Earth)*

Optional deep dive once Earth is complete. Deploy the Red Hat validated **GPT-OSS-20B** model specifically.

**Systems Engineering** ⚙️

- Confirm GPU capacity and storage for GPT-OSS-20B weights
- Review [Performance Insights](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/working_with_the_model_catalog) for your hardware

**Science Crew** 💻

- Find **GPT-OSS-20B** in the model catalog
- Deploy with vLLM (or your preferred runtime)
- Run a few representative prompts and save baseline responses for later missions

**Telemetry**
- GPT-OSS-20B `InferenceService` shows `READY: True`
- A prompt returns a coherent response within a reasonable latency

**Flight Notes**
- GPT-OSS-20B requires significant GPU VRAM — check Performance Insights before deploying
- Save a few baseline prompt/response pairs; you'll use them on Saturn (EvalHub)

✅ **Done when:** GPT-OSS-20B serves inference *(skip if your hardware cannot fit this model)*.

📂 [Full Mission Dossier](missions/03-moon/README.md)

---

### Mission 4 — Mars · Advanced platform *(inner system ☀️ — last stop before the belt)*

**Systems Engineering** ⚙️

- Enable the **training operator**, **Kueue**, and **Ray** on the `DataScienceCluster`
- Configure [hardware profiles / accelerator profiles](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/managing_resources_for_ai_workloads) for GPU workloads
- Set up Kueue quotas and queue management for distributed jobs

**Science Crew** 💻

- Submit a sample **distributed training** job (PyTorchJob or Training Hub + Kubeflow Trainer)
- Observe how Kueue queues and admits the workload
- Confirm the job runs on the expected hardware profile

📖 [Managing distributed workloads](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/managing_distributed_workloads)

**Telemetry**
- `oc get clusterqueue` shows quota capacity and admitted workloads
- PyTorchJob or TrainingJob CR reaches `Succeeded` status
- Hardware profile appears selectable in the RHOAI dashboard workbench launcher

**Flight Notes**
- `oc get localqueue -n <your-project>` — namespace-level queue must exist before submitting jobs
- Kueue admission can be observed with `oc get workloads -n <your-project> -w`

✅ **Done when:** A distributed workload is queued, scheduled, and completes on GPU nodes.

📂 [Full Mission Dossier](missions/04-mars/README.md)

---

## ☄️ Asteroid belt — crossing point

You made it through the inner system. OpenShift AI is installed, GPUs are ready, models are serving, and advanced platform features are enabled.

From here on, **Science Crew** 💻 takes the lead. Systems Engineering ⚙️ still supports (storage, routes, operators), but the missions are about *experimenting and building* on what you stood up.

---

### Mission 5 — Jupiter · Experimenting *(outer system 🪐)*

Broad experimentation mission — connect the dots between pipelines, tracking, and training.

**Systems Engineering** ⚙️

- Ensure the **pipelines** and **MLflow** components are enabled
- Provide S3-compatible storage for pipeline artifacts ([OpenShift Data Foundation](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/installing_and_uninstalling_openshift_ai_self-managed), MinIO, or AWS S3)
- Wire connection secrets and `MLFLOW_TRACKING_URI` into the namespace

**Science Crew** 💻

- Build or import a **Kubeflow Pipeline** and run it end-to-end
- Run an interactive **Training Hub** experiment in a workbench (SFT or LoRA on a small model)
- Track runs in **MLflow** — compare parameters and metrics across experiments
- *(Stretch)* Chain Training Hub into a pipeline step for a reproducible fine-tuning workflow

📖 [Working with AI pipelines](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/working_with_ai_pipelines) · [Training Hub](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/customize_models_for_gen_ai_and_agentic_ai_applications/train-the-model-by-using-your-prepared-data_custom-models)

**Telemetry**
- Pipeline run appears as `Succeeded` in the RHOAI Pipelines UI
- MLflow experiment shows at least two runs with logged metrics
- Training Hub job completes and a model checkpoint is saved to the configured storage

**Flight Notes**
- `oc get pipelinerun -n <your-project>` — check Tekton pipeline run status
- MLflow UI is accessible via its route in the `redhat-ods-applications` namespace

✅ **Done when:** A pipeline run and a Training Hub experiment both appear in MLflow.

📂 [Full Mission Dossier](missions/05-jupiter/README.md)

---

### Mission 6 — Saturn · TrustyAI *(outer system 🪐)*

**Systems Engineering** ⚙️

- Deploy **EvalHub** via the TrustyAI Operator
- Deploy and configure **NeMo Guardrails** for your model endpoint
- Configure MLflow tracking for EvalHub (if not already done on Jupiter)
- Expose EvalHub and Guardrails routes

**Science Crew** 💻

- Submit an **EvalHub** job against your model endpoint (REST API, notebook, or `evalhub` CLI)
- Pick a small benchmark collection and review metrics
- Send prompts **with and without NeMo Guardrails** — compare blocked vs allowed responses
- Verify guardrails catch at least one unsafe or off-policy input

📖 [Evaluate LLMs with EvalHub](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/evaluating_ai_systems/evaluating-llms-with-evalhub_evaluate)

**Telemetry**
- EvalHub job reaches `completed` status and metrics appear in MLflow
- NeMo Guardrails route is reachable and returns `200` for a safe prompt
- A known unsafe prompt returns a blocked/refused response through Guardrails

**Flight Notes**
- `oc get evalhub -n redhat-ods-applications` — check EvalHub CR status
- Use the `evalhub` CLI: `evalhub jobs list --url <evalhub-route>`

✅ **Done when:** An EvalHub job completes and NeMo Guardrails demonstrably filters a test prompt.

📂 [Full Mission Dossier](missions/06-saturn/README.md)

---

### Mission 7 — Uranus · Models-as-a-Service (MaaS) *(outer system 🪐)*

**Systems Engineering** ⚙️

- Follow the [MaaS prerequisites](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/govern_llm_access_with_models-as-a-service): database secret, TLS, and dashboard config flags
- Verify the MaaS gateway is healthy

**Science Crew** 💻

- Publish a model through MaaS *(GPT-OSS-20B if you completed the Moon mission)*
- Create a subscription/token for a consumer
- Call the governed endpoint with `curl` or a small app

**Telemetry**
- MaaS gateway pod is `Running` in `redhat-ods-applications`
- A published model appears in the MaaS catalog in the dashboard
- `curl` with a valid token returns a model response; without a token returns `401`

**Flight Notes**
- `oc get modelmesh -n redhat-ods-applications` — verify gateway health
- Token-based auth: include `Authorization: Bearer <token>` in `curl` requests

✅ **Done when:** A model is consumed through the MaaS API with authentication.

📂 [Full Mission Dossier](missions/07-uranus/README.md)

---

### Mission 8 — Neptune · Agentic applications with RAG and OGX *(outer system 🪐)*

Build an agentic app that retrieves from your own documents and orchestrates tool calls through [OGX](https://github.com/ogx-ai/ogx) — an OpenAI-compatible agentic API server.

**Systems Engineering** ⚙️

- Deploy [OGX](https://github.com/ogx-ai/ogx) on the cluster (or enable it as a model-serving backend)
- Provide vector-store backing storage (e.g. Milvus, PGVector, or OGX vector stores)
- Expose OGX and any dependent services via Routes
- Wire your model or MaaS endpoint as the LLM backend for OGX

**Science Crew** 💻

- Upload documents to a vector store via the OGX [Files](https://ogx-ai.github.io/docs/building_applications/rag) / Vector Stores API
- Build a RAG flow using the OGX [Responses API](https://ogx-ai.github.io/docs/building_applications/rag) (agentic file search + tool calling)
- Try an [agentic starter kit](https://github.com/red-hat-data-services/agentic-starter-kits) (e.g. LangGraph agentic RAG) on OpenShift
- Ask questions that require retrieving from your documents — confirm the agent cites the right sources

📖 [Enterprise RAG chatbot on OpenShift AI](https://developers.redhat.com/articles/2026/01/29/deploy-enterprise-rag-chatbot-red-hat-openshift-ai)

**Telemetry**
- OGX `/health` endpoint returns `200`
- A document uploaded to the vector store is retrievable via the Files API
- An agentic query returns an answer that cites content from your uploaded documents

**Flight Notes**
- `curl <ogx-route>/v1/models` — verify OGX sees your LLM backend
- Use `OPENAI_BASE_URL=<ogx-route>/v1` with any OpenAI-compatible client

✅ **Done when:** An agentic RAG application answers from your own data through OGX.

📂 [Full Mission Dossier](missions/08-neptune/README.md)

---

## Deep Space 🌌 — bonus explorations

Optional missions once you've completed the solar system. Split up and report back — each person picks one area.

**Model Registry**

- **Systems Engineering** ⚙️: Enable the component, provision backend storage
- **Science Crew** 💻: Register a model version from a pipeline or training run

**Feature store (Feast)**

- **Systems Engineering** ⚙️: Enable Feast operator
- **Science Crew** 💻: Connect a notebook to an online feature store

**Model catalog deep dive**

- **Science Crew** 💻: Use Model Performance view to compare validated models for a workload type

**TrustyAI — bias & drift**

- **Science Crew** 💻: Run a bias or data-drift check on a model beyond EvalHub benchmarks

📖 [OpenShift AI 3.4 documentation hub](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4)

✅ **Done when:** Each teammate demos one bonus feature in 5 minutes.

---

## Tips for Mission Control

- **Mix the crews** ⚙️ + 💻 throughout — but respect the zones: Systems Engineering ⚙️ leads from the Sun through Mars; Science Crew 💻 leads beyond the asteroid belt.
- **Venus track B** (fake GPU) is great when AWS GPU quota is tight; switch to track A before Earth.
- **Earth before Moon** — learn model serving in general first; GPT-OSS-20B on the Moon is optional and hardware-dependent.
- **The asteroid belt** ☄️ is your checkpoint: don't cross to Jupiter until Mars (distributed training + Kueue) is working.
- **Jupiter → Saturn → Uranus → Neptune** — experiment, evaluate and guardrail, govern with MaaS, then build agentic apps.
- **Deep Space** topics are optional stretch goals for teams that finish early.
- For automation examples, see [alvarolop/rhoai-gitops](https://github.com/alvarolop/rhoai-gitops).
