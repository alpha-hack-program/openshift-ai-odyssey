# Mission 8 — Neptune · Agentic RAG with OGX

## Mission Briefing

Neptune is the last planet — cold, distant, and the most complex to reach. This mission is the culmination of the Odyssey: you'll deploy OGX (an OpenAI-compatible agentic API server), wire it to your model endpoint, upload documents to a vector store, and build a RAG application that retrieves from your own data. The agent reasons across documents, calls tools, and cites its sources. That's not just AI — that's enterprise-grade agentic intelligence running on your cluster. 🌊🤖

## Pre-launch Checklist

- Mission 7 (Uranus) complete — a model endpoint is accessible (via MaaS or direct)
- A deployed model endpoint accessible from inside the cluster (GPT-OSS-20B or any capable LLM)
- `oc` access with `cluster-admin`
- Vector store backend available (Milvus, PGVector, or built-in OGX vector stores)
- Python 3.11 or later on your workstation (for the agentic starter kit)

## Flight Plan

### Systems Engineering ⚙️

#### 1. Deploy OGX on the cluster

```bash
# Clone the OGX repo (or apply the Helm chart)
helm repo add ogx https://ogx-ai.github.io/helm-charts
helm repo update

helm install ogx ogx/ogx \
  --namespace ogx \
  --create-namespace \
  --set llm.endpoint="http://earth-model.mission-control.svc.cluster.local/v1" \
  --set llm.model="earth-model" \
  --set vectorStore.type=pgvector \
  --set vectorStore.pgvector.connectionString="postgresql://ogx:ogx@postgresql.ogx.svc.cluster.local:5432/ogxdb"
```

Verify OGX is running:
```bash
oc get pods -n ogx
```

#### 2. Deploy a vector store backend (PGVector)

```bash
oc new-app postgresql-ephemeral \
  -p POSTGRESQL_USER=ogx \
  -p POSTGRESQL_PASSWORD=ogx \
  -p POSTGRESQL_DATABASE=ogxdb \
  -n ogx

# Enable the pgvector extension
oc exec -it -n ogx \
  $(oc get pod -n ogx -l name=postgresql -o jsonpath='{.items[0].metadata.name}') \
  -- psql -U ogx -d ogxdb -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

#### 3. Expose OGX routes

```bash
oc expose svc ogx -n ogx
OGX_ROUTE=$(oc get route ogx -n ogx -o jsonpath='{.spec.host}')
echo "http://$OGX_ROUTE"
```

#### 4. Wire your model or MaaS endpoint as OGX backend

If using the MaaS endpoint from Uranus instead of a direct model service:

```bash
oc set env deployment/ogx -n ogx \
  LLM_ENDPOINT="https://$(oc get route maas-gateway -n redhat-ods-applications -o jsonpath='{.spec.host}')/v1" \
  LLM_API_KEY="$USER_TOKEN"
```

### Science Crew 💻

#### 1. Verify OGX is healthy

```bash
OGX_ROUTE=$(oc get route ogx -n ogx -o jsonpath='{.spec.host}')

curl "http://$OGX_ROUTE/health"
```

Expected: `{"status":"ok"}`

```bash
# Verify OGX sees your LLM backend
curl "http://$OGX_ROUTE/v1/models" | jq '.data[].id'
```

#### 2. Upload documents to the vector store

Prepare a set of markdown or text documents (e.g. your team's architecture docs, or the OpenShift AI docs).

```python
import httpx
import os

OGX_URL = f"http://{os.environ.get('OGX_ROUTE', 'ogx-route')}"

# Create a vector store
r = httpx.post(f"{OGX_URL}/v1/vector_stores", json={"name": "odyssey-docs"})
vs_id = r.json()["id"]
print(f"Vector store ID: {vs_id}")

# Upload a document
with open("my-architecture.md", "rb") as f:
    r = httpx.post(
        f"{OGX_URL}/v1/files",
        files={"file": ("my-architecture.md", f, "text/markdown")},
    )
file_id = r.json()["id"]

# Add the file to the vector store
httpx.post(
    f"{OGX_URL}/v1/vector_stores/{vs_id}/files",
    json={"file_id": file_id},
)
print("Document indexed in vector store")
```

Wait for the file to be processed:
```python
import time

while True:
    r = httpx.get(f"{OGX_URL}/v1/vector_stores/{vs_id}/files/{file_id}")
    status = r.json().get("status")
    print(f"Status: {status}")
    if status == "completed":
        break
    time.sleep(5)
```

#### 3. Build a RAG flow using the OGX Responses API

```python
response = httpx.post(
    f"{OGX_URL}/v1/responses",
    json={
        "model": "earth-model",
        "input": "What is the architecture of our AI platform?",
        "tools": [
            {
                "type": "file_search",
                "vector_store_ids": [vs_id]
            }
        ]
    },
    timeout=60
)

result = response.json()
print("Answer:", result["output"][0]["content"][0]["text"])

# Check citations
for annotation in result["output"][0]["content"][0].get("annotations", []):
    if annotation["type"] == "file_citation":
        print(f"  → cited from: {annotation.get('filename')}")
```

Expected: a relevant answer that references your uploaded document.

#### 4. Try an agentic starter kit

Clone the Red Hat agentic starter kits:
```bash
git clone https://github.com/red-hat-data-services/agentic-starter-kits.git
cd agentic-starter-kits/langgraph-rag
```

Configure the kit to point at your OGX endpoint:
```bash
export OPENAI_BASE_URL="http://$OGX_ROUTE/v1"
export OPENAI_API_KEY="dummy"  # OGX uses its own auth model
export VECTOR_STORE_ID="$vs_id"
```

Run the starter kit:
```bash
pip install -r requirements.txt
python app.py
```

Ask the agent questions that require document retrieval — confirm it cites your uploaded files.

## GO / NO-GO Checks

- [ ] GO — `curl "http://$OGX_ROUTE/health"` returns `{"status":"ok"}`
- [ ] GO — `curl "http://$OGX_ROUTE/v1/models"` returns a list including your model
- [ ] GO — A document uploaded to the vector store reaches `status: completed`
- [ ] GO — A Responses API call returns an answer with at least one `file_citation` annotation
- [ ] GO — An agentic RAG query returns a contextually relevant answer from your documents

## Abort Conditions & Recovery

| Symptom | Likely cause | Recovery procedure |
|---------|-------------|--------------------|
| OGX pod `CrashLoopBackOff` | LLM endpoint unreachable or misconfigured | `oc logs -n ogx <ogx-pod>` — check `LLM_ENDPOINT` value; test with `curl` from inside the pod |
| Document stuck in `processing` | PGVector extension not enabled | `oc exec` into PostgreSQL pod and run `\dx` to verify `vector` extension is listed |
| Responses API returns empty citations | File not added to vector store, or embedding failed | Check `GET /v1/vector_stores/<id>/files/<file_id>` for error status; re-upload the document |
| Agent hallucinating instead of retrieving | `file_search` tool not included in the request | Verify the `tools` array in the Responses API call includes `{"type": "file_search"}` |
| MaaS token rejected by OGX | API key not passed in request headers | Ensure `LLM_API_KEY` env var is set in the OGX deployment and matches your MaaS token |

## Mission Success Criteria

OGX is deployed and healthy, at least one document is indexed in a vector store, and an agentic Responses API call returns an answer with citations pointing to your uploaded document. The mission is complete when your agent can answer questions from your own data.

## Bonus Mission Objective

Extend the RAG agent with a custom tool that queries a live API (e.g. a Prometheus metrics endpoint or a REST service from your cluster) alongside document retrieval. The agent should decide dynamically whether to query documents, call the API, or combine both sources to answer a question — demonstrating true multi-tool agentic reasoning.
