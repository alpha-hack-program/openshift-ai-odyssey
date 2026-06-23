# Red Hat OpenShift AI Odyssey

Your space suit for Red Hat OpenShift AI. Embark on a hands-on journey from zero to MLOps hero through practical missions and challenges! 🚀🪐

![OpenShift AI Odyssey learning path](docs/images/rhoai-learning-path.png)

A hands-on path for small teams. Pair people from different backgrounds so each mission covers both **platform** and **application** work.

Follow the missions from the **Sun** outward — through the inner planets, across the **asteroid belt**, and into the outer system.

Like the real solar system, there is a divide between the **inner** and **outer** planets:

| Zone | Missions | Who leads |
|------|----------|-----------|
| **Sun + inner system** ☀️ | Sun → Mars *(+ optional Moon)* | Mostly **Platform crew** ⚙️ — install, configure, and prepare the cluster |
| **Asteroid belt** ☄️ | Between Mars and Jupiter | Transition — foundation is ready; experimentation and app dev take the wheel |
| **Outer system** 🪐 | Jupiter → Neptune | Mostly **App crew** 💻 — pipelines, training, evaluation, MaaS, and agentic apps |
| **Deep Space** 🌌 | Bonus explorations | Stretch goals for teams that finish early |

| Crew | Background | You own… |
|------|------------|----------|
| **Platform crew** ⚙️ | OpenShift infra — installation, nodes, storage, operators | Cluster plumbing that makes AI workloads run |
| **App crew** 💻 | CI/CD, middleware, app dev | Projects, notebooks, pipelines, models, and APIs |

**Prerequisites:** OpenShift Container Platform 4.20 on [AWS IPI](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/installing_on_aws/index), cluster-admin access, and the [OpenShift AI 3.4 docs](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4) handy.

---

### ☀️ Sun · OpenShift + OpenShift AI *(shared · start here)*

Everything orbits from here. Goal: OCP 4.20 on AWS IPI with RHOAI 3.4 running on top.

**Platform crew** ⚙️

- Deploy or verify [OpenShift 4.20 on AWS IPI](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/installing_on_aws/index)
- Install the [Red Hat OpenShift AI Operator](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/installing_and_uninstalling_openshift_ai_self-managed) (`stable-3.x` channel — latest GA)
- Create a `DataScienceCluster` with core components (dashboard, workbenches, pipelines, kserve)
- Verify storage classes and PVC provisioning for user workloads

**App crew** 💻

- Log in to the OpenShift AI dashboard
- Create a project
- Confirm workbenches and pipelines appear in the UI

✅ **Done when:** Dashboard is reachable and your team project exists.

---

### Mission 1 — Mercury · Notebook playground *(inner system ☀️)*

**Platform crew** ⚙️

- Confirm notebook image pull and default workbench sizes
- Verify PVC/storage for home directories
- Check SCCs and resource quotas in the project

**App crew** 💻

- Launch a workbench
- Run a short Python notebook (e.g. `pandas`, a simple plot)
- Install one extra package
- Restart the workbench and confirm your work persists

📖 [Getting started — workbenches](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/getting_started_with_red_hat_openshift_ai_self-managed)

✅ **Done when:** A notebook runs code and survives a restart.

---

### Mission 2 — Venus · GPU power-up *(inner system ☀️)*

Pick **one** track per cluster (or run both on separate node pools).

#### Track A — Real GPUs on AWS *(Platform crew ⚙️ leads)*

**Platform crew** ⚙️

- Add GPU worker nodes (e.g. `g6` instances)
- Install [NVIDIA GPU Operator](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/installing_and_uninstalling_openshift_ai_self-managed) + Node Feature Discovery
- Label and taint GPU nodes
- Confirm `nvidia-smi` from a GPU pod

**App crew** 💻

- Verify the GPU appears in the OpenShift AI dashboard (accelerator profile)
- Launch a workbench with a GPU resource request
- Confirm the device is visible inside the pod

#### Track B — Fake GPUs for learning *(Platform crew ⚙️ leads)*

**Platform crew** ⚙️

- Deploy [fake-gpu-operator](https://github.com/run-ai/fake-gpu-operator) on selected worker nodes
- Confirm nodes advertise `nvidia.com/gpu`

**App crew** 💻

- Schedule a GPU-requesting workload on a fake-GPU node
- Confirm scheduling succeeds and the dashboard still shows GPU capacity

✅ **Done when:** At least one node pool exposes GPUs and a workload can request them.

---

### Mission 3 — Earth · Deploy models *(inner system ☀️)*

Learn model serving in general — not tied to one specific model yet.

**Platform crew** ⚙️

- Enable model serving (KServe) on the `DataScienceCluster`
- Ensure GPU nodes and enough PVC/object storage for model weights
- Open network routes for inference endpoints
- Deploy or configure serving runtimes: **vLLM**, **KServe**, and **llm-d** (pick at least two)

**App crew** 💻

- Open the [model catalog](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/working_with_the_model_catalog)
- Pick a small model suitable for your hardware
- Deploy it with a serving runtime (vLLM or KServe)
- Send a test prompt and capture the response
- *(Optional)* Compare inference behaviour with **llm-d** routing

✅ **Done when:** At least one model serves inference from the catalog using a chosen runtime.

---

### 🌙 Moon · GPT-OSS-20B *(optional · orbits Earth)*

Optional deep dive once Earth is complete. Deploy the Red Hat validated **GPT-OSS-20B** model specifically.

**Platform crew** ⚙️

- Confirm GPU capacity and storage for GPT-OSS-20B weights
- Review [Performance Insights](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/working_with_the_model_catalog) for your hardware

**App crew** 💻

- Find **GPT-OSS-20B** in the model catalog
- Deploy with vLLM (or your preferred runtime)
- Run a few representative prompts and save baseline responses for later missions

✅ **Done when:** GPT-OSS-20B serves inference *(skip if your hardware cannot fit this model)*.

---

### Mission 4 — Mars · Advanced platform *(inner system ☀️ — last stop before the belt)*

**Platform crew** ⚙️

- Enable the **training operator**, **Kueue**, and **Ray** on the `DataScienceCluster`
- Configure [hardware profiles / accelerator profiles](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/managing_resources_for_ai_workloads) for GPU workloads
- Set up Kueue quotas and queue management for distributed jobs

**App crew** 💻

- Submit a sample **distributed training** job (PyTorchJob or Training Hub + Kubeflow Trainer)
- Observe how Kueue queues and admits the workload
- Confirm the job runs on the expected hardware profile

📖 [Managing distributed workloads](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/managing_distributed_workloads)

✅ **Done when:** A distributed workload is queued, scheduled, and completes on GPU nodes.

---

## ☄️ Asteroid belt — crossing point

You made it through the inner system. OpenShift AI is installed, GPUs are ready, models are serving, and advanced platform features are enabled.

From here on, **App crew** 💻 takes the lead. Platform crew ⚙️ still supports (storage, routes, operators), but the missions are about *experimenting and building* on what you stood up.

---

### Mission 5 — Jupiter · Experimenting *(outer system 🪐)*

Broad experimentation mission — connect the dots between pipelines, tracking, and training.

**Platform crew** ⚙️

- Ensure the **pipelines** and **MLflow** components are enabled
- Provide S3-compatible storage for pipeline artifacts ([OpenShift Data Foundation](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/installing_and_uninstalling_openshift_ai_self-managed), MinIO, or AWS S3)
- Wire connection secrets and `MLFLOW_TRACKING_URI` into the namespace

**App crew** 💻

- Build or import a **Kubeflow Pipeline** and run it end-to-end
- Run an interactive **Training Hub** experiment in a workbench (SFT or LoRA on a small model)
- Track runs in **MLflow** — compare parameters and metrics across experiments
- *(Stretch)* Chain Training Hub into a pipeline step for a reproducible fine-tuning workflow

📖 [Working with AI pipelines](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/working_with_ai_pipelines) · [Training Hub](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/customize_models_for_gen_ai_and_agentic_ai_applications/train-the-model-by-using-your-prepared-data_custom-models)

✅ **Done when:** A pipeline run and a Training Hub experiment both appear in MLflow.

---

### Mission 6 — Saturn · TrustyAI *(outer system 🪐)*

**Platform crew** ⚙️

- Deploy **EvalHub** via the TrustyAI Operator
- Deploy and configure **NeMo Guardrails** for your model endpoint
- Configure MLflow tracking for EvalHub (if not already done on Jupiter)
- Expose EvalHub and Guardrails routes

**App crew** 💻

- Submit an **EvalHub** job against your model endpoint (REST API, notebook, or `evalhub` CLI)
- Pick a small benchmark collection and review metrics
- Send prompts **with and without NeMo Guardrails** — compare blocked vs allowed responses
- Verify guardrails catch at least one unsafe or off-policy input

📖 [Evaluate LLMs with EvalHub](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/evaluating_ai_systems/evaluating-llms-with-evalhub_evaluate)

✅ **Done when:** An EvalHub job completes and NeMo Guardrails demonstrably filters a test prompt.

---

### Mission 7 — Uranus · Models-as-a-Service (MaaS) *(outer system 🪐)*

**Platform crew** ⚙️

- Follow the [MaaS prerequisites](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/govern_llm_access_with_models-as-a-service): database secret, TLS, and dashboard config flags
- Verify the MaaS gateway is healthy

**App crew** 💻

- Publish a model through MaaS *(GPT-OSS-20B if you completed the Moon mission)*
- Create a subscription/token for a consumer
- Call the governed endpoint with `curl` or a small app

✅ **Done when:** A model is consumed through the MaaS API with authentication.

---

### Mission 8 — Neptune · Agentic applications with RAG and OGX *(outer system 🪐)*

Build an agentic app that retrieves from your own documents and orchestrates tool calls through [OGX](https://github.com/ogx-ai/ogx) — an OpenAI-compatible agentic API server.

**Platform crew** ⚙️

- Deploy [OGX](https://github.com/ogx-ai/ogx) on the cluster (or enable it as a model-serving backend)
- Provide vector-store backing storage (e.g. Milvus, PGVector, or OGX vector stores)
- Expose OGX and any dependent services via Routes
- Wire your model or MaaS endpoint as the LLM backend for OGX

**App crew** 💻

- Upload documents to a vector store via the OGX [Files](https://ogx-ai.github.io/docs/building_applications/rag) / Vector Stores API
- Build a RAG flow using the OGX [Responses API](https://ogx-ai.github.io/docs/building_applications/rag) (agentic file search + tool calling)
- Try an [agentic starter kit](https://github.com/red-hat-data-services/agentic-starter-kits) (e.g. LangGraph agentic RAG) on OpenShift
- Ask questions that require retrieving from your documents — confirm the agent cites the right sources

📖 [Enterprise RAG chatbot on OpenShift AI](https://developers.redhat.com/articles/2026/01/29/deploy-enterprise-rag-chatbot-red-hat-openshift-ai)

✅ **Done when:** An agentic RAG application answers from your own data through OGX.

---

## Deep Space 🌌 — bonus explorations

Optional missions once you've completed the solar system. Split up and report back — each person picks one area.

**Model Registry**

- **Platform crew** ⚙️: Enable the component, provision backend storage
- **App crew** 💻: Register a model version from a pipeline or training run

**Feature store (Feast)**

- **Platform crew** ⚙️: Enable Feast operator
- **App crew** 💻: Connect a notebook to an online feature store

**Model catalog deep dive**

- **App crew** 💻: Use Model Performance view to compare validated models for a workload type

**TrustyAI — bias & drift**

- **App crew** 💻: Run a bias or data-drift check on a model beyond EvalHub benchmarks

📖 [OpenShift AI 3.4 documentation hub](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4)

✅ **Done when:** Each teammate demos one bonus feature in 5 minutes.

---

## Tips for team leads

- **Mix the crews** ⚙️ + 💻 throughout — but respect the zones: Platform crew ⚙️ leads from the Sun through Mars; App crew 💻 leads beyond the asteroid belt.
- **Venus track B** (fake GPU) is great when AWS GPU quota is tight; switch to track A before Earth.
- **Earth before Moon** — learn model serving in general first; GPT-OSS-20B on the Moon is optional and hardware-dependent.
- **The asteroid belt** ☄️ is your checkpoint: don't cross to Jupiter until Mars (distributed training + Kueue) is working.
- **Jupiter → Saturn → Uranus → Neptune** — experiment, evaluate and guardrail, govern with MaaS, then build agentic apps.
- **Deep Space** topics are optional stretch goals for teams that finish early.
- For automation examples, see [alvarolop/rhoai-gitops](https://github.com/alvarolop/rhoai-gitops).
