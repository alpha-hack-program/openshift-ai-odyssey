# Mission 7 — Uranus · Models-as-a-Service (MaaS)

## Mission Briefing

Uranus rotates on its side — unorthodox, and quietly powerful. MaaS is your model governance layer: token-gated access, usage tracking, and quota enforcement for any deployed model. Think of it as Mission Control issuing launch authorisations — not every crew member can fire the rockets whenever they want. This mission publishes your model through a governed API gateway and verifies that only authorised callers get through. 🔐🛸

## Pre-launch Checklist

- Mission 6 (Saturn) complete — you have a deployed, evaluated model
- A model endpoint available (GPT-OSS-20B from Moon recommended, or any Earth model)
- `oc` access with `cluster-admin`
- Database secret for the MaaS gateway (PostgreSQL or equivalent)
- TLS certificate available (or cluster wildcard cert)

## Flight Plan

### Systems Engineering ⚙️

#### 1. Verify MaaS prerequisites

```bash
# Check that MaaS (llm-d gateway / RHOAI MaaS component) is enabled
oc get datasciencecluster default-dsc \
  -o jsonpath='{.spec.components}'
```

MaaS in RHOAI 3.4 is managed via the dashboard and requires:
- A database backend (for storing subscriptions/tokens)
- TLS configured on the cluster ingress
- Dashboard feature flag enabled

Enable via dashboard feature flags:
```bash
oc edit configmap odh-dashboard-config -n redhat-ods-applications
```

Add under `dashboardConfig`:
```yaml
disableModelServing: false
disableKServe: false
disableModelMesh: false
```

#### 2. Create the database secret for MaaS gateway

```bash
# If using an in-cluster PostgreSQL (for learning purposes):
oc new-app postgresql-ephemeral \
  -p POSTGRESQL_USER=maas \
  -p POSTGRESQL_PASSWORD=maas-secret \
  -p POSTGRESQL_DATABASE=maasdb \
  -n redhat-ods-applications

# Create the MaaS gateway database secret
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: maas-db-secret
  namespace: redhat-ods-applications
type: Opaque
stringData:
  DATABASE_URL: "postgresql://maas:maas-secret@postgresql.redhat-ods-applications.svc.cluster.local:5432/maasdb"
EOF
```

#### 3. Verify MaaS gateway health

```bash
oc get pods -n redhat-ods-applications | grep maas
```

Expected: MaaS gateway pod(s) in `Running` state.

```bash
MAAS_ROUTE=$(oc get route maas-gateway -n redhat-ods-applications -o jsonpath='{.spec.host}')
curl "https://$MAAS_ROUTE/healthz"
```

Expected: `{"status":"ok"}` or HTTP 200.

### Science Crew 💻

#### 1. Publish a model through MaaS

In the dashboard: **Model serving** → select your deployed model (e.g. `gpt-oss-20b` or `earth-model`) → **Publish via MaaS**

Or via API:
```bash
MAAS_ROUTE=$(oc get route maas-gateway -n redhat-ods-applications -o jsonpath='{.spec.host}')

# Publish the model (admin token required)
curl -X POST "https://$MAAS_ROUTE/v1/models" \
  -H "Authorization: Bearer <admin-token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-published-model",
    "endpoint": "http://earth-model.mission-control.svc.cluster.local/v1",
    "description": "Earth mission model published through MaaS"
  }'
```

#### 2. Create a subscription token for a consumer

In the dashboard: **Models-as-a-Service** → select your model → **Create subscription**

Assign:
- **Consumer name**: `team-science-crew`
- **Rate limit**: 100 requests/hour
- **Quota**: 10,000 tokens/day

Note the generated token — you'll use it in the next step.

Or via API:
```bash
curl -X POST "https://$MAAS_ROUTE/v1/subscriptions" \
  -H "Authorization: Bearer <admin-token>" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "my-published-model",
    "consumer": "team-science-crew",
    "rate_limit": 100,
    "token_quota": 10000
  }' | jq '.access_token'
```

Save the returned `access_token`.

#### 3. Call the governed endpoint

**With a valid token (should succeed):**
```bash
USER_TOKEN="<access_token_from_step_2>"
MAAS_ROUTE=$(oc get route maas-gateway -n redhat-ods-applications -o jsonpath='{.spec.host}')

curl -X POST "https://$MAAS_ROUTE/v1/chat/completions" \
  -H "Authorization: Bearer $USER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "my-published-model",
    "messages": [{"role": "user", "content": "Tell me about mission Uranus."}],
    "max_tokens": 50
  }' | jq '{status: .choices[0].message.content}'
```

Expected: HTTP 200 with a response.

**Without a token (should fail):**
```bash
curl -X POST "https://$MAAS_ROUTE/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "my-published-model",
    "messages": [{"role": "user", "content": "Unauthorised call."}]
  }'
```

Expected: HTTP 401 Unauthorized.

## GO / NO-GO Checks

- [ ] GO — MaaS gateway pod is `Running` in `redhat-ods-applications`
- [ ] GO — `curl https://$MAAS_ROUTE/healthz` returns HTTP 200
- [ ] GO — Published model appears in the MaaS catalog in the dashboard
- [ ] GO — `curl` with a valid token returns HTTP 200 with a model response
- [ ] GO — `curl` without a token (or with an invalid token) returns HTTP 401

## Abort Conditions & Recovery

| Symptom | Likely cause | Recovery procedure |
|---------|-------------|--------------------|
| MaaS gateway pod fails to start | Database connection error | `oc logs -n redhat-ods-applications <maas-pod>` — check `DATABASE_URL` in the secret; verify PostgreSQL is running |
| Model publish fails with 404 | Dashboard config flags not set or restart needed | `oc rollout restart deployment rhods-dashboard -n redhat-ods-applications` after setting config flags |
| Valid token returns 401 | Token not yet propagated to gateway | Wait 30s for sync and retry; check gateway logs for token validation errors |
| Rate limiting not enforced | Gateway not in enforcing mode | Check MaaS gateway config — some deployments default to audit mode |

## Mission Success Criteria

A model is published through the MaaS gateway, a consumer token is created, a request with the valid token returns a model response (HTTP 200), and a request without a token is rejected (HTTP 401).
