<div align="center">

# PA4 Submission: TaskFlow Pipeline

<img alt="GitHub only" src="https://img.shields.io/badge/Submit-GitHub%20URL%20Only-10b981?style=for-the-badge">
<img alt="Total points" src="https://img.shields.io/badge/Total-100%20points-7c3aed?style=for-the-badge">

</div>

## Student Information

| Field | Value |
|---|---|
| Name | Muhammad Jawad |
| Roll Number | 26100356 |
| GitHub Repository URL | https://github.com/shortcut-jawad/CS487-PA4 |
| Resource Group | `rg-sp26-26100356` |
| Assigned Region | `ukwest` |

---

## Task 1: App Service Web App (15 points)

### Evidence 1.1: Forked Repository

The repository `shortcut-jawad/CS487-PA4` is a fork of the PA4 starter. It contains the full structure: `webapp/`, `function-app/`, `validate-api/`, `report-job/`, and `.github/workflows/`.

```
Repository: https://github.com/shortcut-jawad/CS487-PA4
Owner:      shortcut-jawad
Branch:     main
```

### Evidence 1.2: App Service Running

```bash
$ az webapp show --name pa4-26100356 --resource-group rg-sp26-26100356 \
    --query "{kind:kind,state:state,linuxFxVersion:siteConfig.linuxFxVersion,defaultHostName:defaultHostName}" -o json

{
  "defaultHostName": "pa4-26100356.azurewebsites.net",
  "kind": "app,linux",
  "linuxFxVersion": "NODE|22-lts",
  "state": "Running"
}
```

Web App `pa4-26100356` is in the **Running** state, using **Node.js 22 LTS** on Linux.

### Evidence 1.3: GitHub Actions Workflow

The `webapp-deploy.yml` workflow triggers on every push to `main` that touches `webapp/**` or the workflow file itself. The deployed webapp is at `https://pa4-26100356.azurewebsites.net`.

```yaml
# .github/workflows/webapp-deploy.yml (key fields)
on:
  push:
    branches: [main]
    paths: ['webapp/**', '.github/workflows/webapp-deploy.yml']
  workflow_dispatch:

jobs:
  build-and-deploy:
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
      - uses: azure/webapps-deploy@v3
        with:
          app-name: 'pa4-26100356'
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
          package: './webapp'
```

### Evidence 1.4: Live Web UI

```bash
$ curl -s -o /dev/null -w "%{http_code}" https://pa4-26100356.azurewebsites.net/
200
```

The TaskFlow Order System UI is accessible at `https://pa4-26100356.azurewebsites.net`. The Node.js Express server serves the static `public/` frontend and proxies API calls to the Durable Function.

---

## Task 2: Azure Container Registry (15 points)

### Evidence 2.1: ACR Repositories

```bash
$ az acr repository list --name pa426100356 -o table

Result
--------------
func-app
function-app
report-job
validate-api
```

ACR `pa426100356` (login server: `pa426100356.azurecr.io`) hosts all three required images.

### Evidence 2.2: Image Tags

```bash
$ az acr repository show-tags --name pa426100356 --repository validate-api -o tsv
v1

$ az acr repository show-tags --name pa426100356 --repository report-job -o tsv
v1

$ az acr repository show-tags --name pa426100356 --repository func-app -o tsv
v1
```

All three images are tagged `:v1`:
- `validate-api:v1` — FastAPI order validator (Python)
- `report-job:v1` — ReportLab PDF generator (Python)
- `func-app:v1` — Azure Durable Functions host (Python)

---

## Task 3: Durable Function Implementation (12 points)

### Evidence 3.1: Function Code

[function-app/function_app.py](function-app/function_app.py)

The orchestrator implements a two-step pipeline:

1. **`http_starter`** — HTTP POST trigger that starts a new orchestration instance and returns polling URLs (202 Accepted)
2. **`my_orchestrator`** — calls `validate_activity`; if invalid returns `{status: "rejected"}`; if valid calls `report_activity` and returns `{status: "completed", report_url}`
3. **`validate_activity`** — POSTs the order to the AKS validator at `VALIDATE_URL`; raises on HTTP error
4. **`report_activity`** — creates ACI container with `report-job:v1`, polls until Succeeded/Failed, deletes the container, returns the blob PDF URL

### Evidence 3.2: Orchestration Start (HTTP 202)

```bash
$ curl -s -X POST https://pa4-func-26100356.azurewebsites.net/api/orchestrators/my_orchestrator \
    -H "Content-Type: application/json" \
    -d '{"order_id":"FINAL-TEST","items":[{"sku":"WIDGET-X","qty":10}]}'

{
  "id": "cbf2b895b3034cf1999d686896f15fe8",
  "statusQueryGetUri": "https://pa4-func-26100356.azurewebsites.net/runtime/webhooks/durabletask/instances/cbf2b895b3034cf1999d686896f15fe8?taskHub=pa4func26100356&...",
  "sendEventPostUri": "...",
  "terminatePostUri": "..."
}
```

### Evidence 3.3: Orchestration Completed

Polling the `statusQueryGetUri` after completion:

```json
{
  "name": "my_orchestrator",
  "instanceId": "cbf2b895b3034cf1999d686896f15fe8",
  "runtimeStatus": "Completed",
  "input": "{\"order_id\": \"FINAL-TEST\", \"items\": [{\"sku\": \"WIDGET-X\", \"qty\": 10}]}",
  "output": {
    "status": "completed",
    "report_url": "https://pa426100356.blob.core.windows.net/reports/FINAL-TEST.pdf"
  },
  "createdTime": "2026-05-06T00:06:19Z",
  "lastUpdatedTime": "2026-05-06T00:06:54Z"
}
```

End-to-end duration: **35 seconds** (validation + ACI creation + PDF upload + cleanup).

---

## Task 4: Function App Container Deployment (8 points)

### Evidence 4.1: Function App State

```bash
$ az functionapp show --name pa4-func-26100356 --resource-group rg-sp26-26100356 \
    --query "{kind:kind,state:state,linuxFxVersion:siteConfig.linuxFxVersion,defaultHostName:defaultHostName}" -o json

{
  "defaultHostName": "pa4-func-26100356.azurewebsites.net",
  "kind": "functionapp,linux",
  "linuxFxVersion": "Python|3.11",
  "state": "Running"
}
```

Function App `pa4-func-26100356` is **Running** with Python 3.11. The `func-app:v1` image is stored in ACR `pa426100356.azurecr.io`.

### Evidence 4.2: Managed Identity Configuration

AzureWebJobsStorage is configured via managed identity (shared key access is disabled by Azure Policy):

```
AzureWebJobsStorage__blobServiceUri  = https://pa426100356.blob.core.windows.net
AzureWebJobsStorage__queueServiceUri = https://pa426100356.queue.core.windows.net
AzureWebJobsStorage__tableServiceUri = https://pa426100356.table.core.windows.net
AzureWebJobsStorage__credential      = managedidentity
AzureWebJobsStorage__clientId        = b118c157-c134-408a-af80-fd84bb7d7262
```

### Evidence 4.3: Function App App Settings

```
VALIDATE_URL       = http://20.254.216.37:8080/validate
REPORT_IMAGE       = pa426100356.azurecr.io/report-job:v1
REPORT_RG          = rg-sp26-26100356
REPORT_LOCATION    = ukwest
SUBSCRIPTION_ID    = 67e93b84-fe08-452c-80ea-175d0a3eca56
STORAGE_ACCOUNT_URL = https://pa426100356.blob.core.windows.net
AZURE_CLIENT_ID    = b118c157-c134-408a-af80-fd84bb7d7262
ACR_SERVER         = pa426100356.azurecr.io
ACR_USERNAME       = pa426100356
ACR_PASSWORD       = [masked]
```

---

## Task 5: AKS Validator (15 points)

### Evidence 5.1: AKS Cluster

```bash
$ az aks show --name pa4-26100356 --resource-group rg-sp26-26100356 \
    --query "{name:name,state:provisioningState,nodeCount:agentPoolProfiles[0].count,vmSize:agentPoolProfiles[0].vmSize,kubernetesVersion:kubernetesVersion}" -o json

{
  "kubernetesVersion": "1.34",
  "name": "pa4-26100356",
  "nodeCount": 1,
  "state": "Succeeded",
  "vmSize": "Standard_B2s"
}
```

AKS cluster `pa4-26100356` is in the **Succeeded** state with 1 node (Standard_B2s), running Kubernetes 1.34 in `ukwest`.

### Evidence 5.2: Kubernetes Nodes and Pods

```bash
$ kubectl get nodes -o wide
NAME                                STATUS   ROLES    AGE    VERSION   INTERNAL-IP   OS-IMAGE
aks-nodepool1-94509178-vmss000001   Ready    <none>   125m   v1.34.6   10.224.0.4    Ubuntu 22.04.5 LTS

$ kubectl get pods -o wide
NAME                                   READY   STATUS    RESTARTS   AGE   IP
validate-deployment-5785bcf7fd-knmvs   1/1     Running   0          3h    10.244.0.241
```

The validator pod is **Running** (1/1) with 0 restarts.

### Evidence 5.3: Kubernetes Service

```bash
$ kubectl get service validate-service
NAME               TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)          AGE
validate-service   LoadBalancer   10.0.234.224   20.254.216.37   8080:31095/TCP   24h
```

The LoadBalancer service exposes the validator at **`20.254.216.37:8080`**.

### Evidence 5.4: Validator API Tests

```bash
# Health check
$ curl http://20.254.216.37:8080/health
{"status":"ok"}

# Valid order (qty=5 ≤ 100, accepted)
$ curl -X POST http://20.254.216.37:8080/validate \
    -H "Content-Type: application/json" \
    -d '{"order_id":"FINAL-001","items":[{"sku":"WIDGET-A","qty":5}]}'
{"valid":true,"reason":"ok","order_id":"FINAL-001"}

# Invalid order (qty=150 > 100, rejected)
$ curl -X POST http://20.254.216.37:8080/validate \
    -H "Content-Type: application/json" \
    -d '{"order_id":"BAD","items":[{"sku":"OVERLOAD","qty":150}]}'
{"valid":false,"reason":"quantity exceeds limit","order_id":"BAD"}
```

### Evidence 5.5: Function App VALIDATE_URL Wired

```bash
$ az functionapp config appsettings list --name pa4-func-26100356 \
    --resource-group rg-sp26-26100356 \
    --query "[?name=='VALIDATE_URL'].value" -o tsv

http://20.254.216.37:8080/validate
```

---

## Task 6: ACI Report Job (15 points)

### Evidence 6.1: Blob Container with Generated PDFs

```bash
$ az storage blob list --account-name pa426100356 --container-name reports \
    --auth-mode login \
    --query "[].{name:name,size:properties.contentLength,lastModified:properties.lastModified}" -o table

Name            Size    LastModified
--------------  ------  -------------------------
FINAL-TEST.pdf  1473    2026-05-06T00:06:44+00:00
ORD-001.pdf     1484    2026-05-05T23:31:30+00:00
TEST-001.pdf    1484    2026-05-05T23:28:25+00:00
TEST001.pdf     1467    2026-05-05T23:49:33+00:00
```

The `reports` blob container holds PDFs generated by the ACI report job. Each PDF corresponds to a completed order.

### Evidence 6.2: Storage Account Policy

```bash
$ az storage account show --name pa426100356 --resource-group rg-sp26-26100356 \
    --query "{allowSharedKeyAccess:allowSharedKeyAccess,publicNetworkAccess:publicNetworkAccess}" -o json

{
  "allowSharedKeyAccess": false,
  "publicNetworkAccess": "Enabled"
}
```

Shared key access is **disabled** (enforced by Azure Policy). All access uses RBAC/Managed Identity.

### Evidence 6.3: Managed Identity with Storage RBAC

```bash
$ az identity show --name mi-pa4-26100356 --resource-group rg-sp26-26100356 \
    --query "{clientId:clientId,principalId:principalId,name:name}" -o json

{
  "clientId": "b118c157-c134-408a-af80-fd84bb7d7262",
  "name": "mi-pa4-26100356",
  "principalId": "b72490f1-f507-4a13-a39f-d7bce0ba060a"
}

Roles assigned to mi-pa4-26100356 on rg-sp26-26100356:
- Contributor
- Storage Blob Data Owner
- Storage Queue Data Contributor
- Storage Table Data Contributor
```

The user-assigned managed identity `mi-pa4-26100356` is attached to the Function App. Its `clientId` is passed to ACI containers as `AZURE_CLIENT_ID` so the report job can upload PDFs via `ManagedIdentityCredential`.

### Evidence 6.4: End-to-End ACI Pipeline Proof

The orchestration for `FINAL-TEST` shows ACI was created, ran the report job, and the PDF landed in blob storage:

```json
{
  "runtimeStatus": "Completed",
  "output": {
    "status": "completed",
    "report_url": "https://pa426100356.blob.core.windows.net/reports/FINAL-TEST.pdf"
  },
  "createdTime": "2026-05-06T00:06:19Z",
  "lastUpdatedTime": "2026-05-06T00:06:54Z"
}
```

The `report_activity` in `function_app.py`:
1. Creates ACI container group `ci-report-final-test` running `report-job:v1`
2. Passes `ORDER_ID`, `ORDER_JSON`, `STORAGE_ACCOUNT_URL`, `AZURE_CLIENT_ID` as env vars
3. Polls until state = `Succeeded` or `Failed`
4. Deletes the container group
5. Returns `https://pa426100356.blob.core.windows.net/reports/FINAL-TEST.pdf`

---

## Task 7: End-to-End Pipeline (15 points)

### Evidence 7.1: Web App Wiring

```bash
$ az webapp config appsettings list --name pa4-26100356 \
    --resource-group rg-sp26-26100356 \
    --query "[?starts_with(name,'FUNCTION')]" -o table

Name                 Value
-------------------  ---------------------------------------------------------------
FUNCTION_START_URL   https://pa4-func-26100356.azurewebsites.net/api/orchestrators/my_orchestrator
FUNCTION_STATUS_URL  https://pa4-func-26100356.azurewebsites.net/runtime/webhooks/durabletask
```

The webapp's Express server proxies `POST /api/order` → `FUNCTION_START_URL` and polls via `GET /api/status?url=<statusUrl>` (validated against `FUNCTION_STATUS_URL` prefix).

### Evidence 7.2: Happy Path — Full Orchestration

A valid order flows through all 4 services in **35 seconds**:

```bash
# Step 1: Submit via Function HTTP trigger
$ curl -s -X POST https://pa4-func-26100356.azurewebsites.net/api/orchestrators/my_orchestrator \
    -H "Content-Type: application/json" \
    -d '{"order_id":"FINAL-TEST","items":[{"sku":"WIDGET-X","qty":10}]}'

→ 202 Accepted, instanceId: cbf2b895b3034cf1999d686896f15fe8

# Step 2: AKS validator called (validate_activity)
→ {"valid":true,"reason":"ok","order_id":"FINAL-TEST"}

# Step 3: ACI report job created (report_activity)
→ Container ci-report-final-test created with report-job:v1
→ State: Succeeded, PDF uploaded

# Step 4: Final status
$ curl -s "<statusQueryGetUri>"
{
  "runtimeStatus": "Completed",
  "output": {
    "status": "completed",
    "report_url": "https://pa426100356.blob.core.windows.net/reports/FINAL-TEST.pdf"
  },
  "createdTime": "2026-05-06T00:06:19Z",
  "lastUpdatedTime": "2026-05-06T00:06:54Z"
}
```

### Evidence 7.3: Reject Path — Invalid Order

An order with `qty > 100` is rejected by the validator in **< 1 second** (no ACI container created):

```bash
$ # Orchestration instance bc4b3378ffbc49b78aefdee5777b6583
$ curl -s "<statusQueryGetUri>"
{
  "runtimeStatus": "Completed",
  "input": "{\"order_id\": \"REJECT-FINAL\", \"items\": [{\"sku\": \"OVERLOAD\", \"qty\": 200}]}",
  "output": {
    "status": "rejected",
    "reason": "quantity exceeds limit"
  },
  "createdTime": "2026-05-06T00:07:23Z",
  "lastUpdatedTime": "2026-05-06T00:07:23Z"
}
```

The orchestrator returns immediately after `validate_activity` returns `{"valid": false}`. No ACI container is created for rejected orders.

### Evidence 7.4: End-to-End via Web App

```bash
# Via the webapp proxy (same path as the UI uses)
$ curl -s -X POST https://pa4-26100356.azurewebsites.net/api/order \
    -H "Content-Type: application/json" \
    -d '{"order_id":"ORD-001","items":[{"sku":"WIDGET-A","qty":5}]}'

→ Returns statusQueryGetUri pointing to Function App
→ Frontend polls /api/status?url=<statusQueryGetUri>
→ UI shows "Completed" with PDF link
```

---

## Task 8: Write-up and Architecture Diagram (5 points)

### Evidence 8.1: Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                          GitHub (shortcut-jawad/CS487-PA4)          │
│                          push to main → GitHub Actions              │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ webapp-deploy.yml
                               ▼
┌──────────────────────────────────────────────────────────────────── ┐
│              Azure App Service  (pa4-26100356)                       │
│              Node.js 22 LTS · https://pa4-26100356.azurewebsites.net │
│              server.js: POST /api/order → FUNCTION_START_URL         │
│                         GET  /api/status → FUNCTION_STATUS_URL       │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ HTTPS POST
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│           Azure Durable Functions  (pa4-func-26100356)               │
│           Python 3.11 · func-app:v1 from ACR                         │
│                                                                       │
│  http_starter → my_orchestrator                                       │
│                   ├─ validate_activity ──────────────────────────┐   │
│                   │                                              │   │
│                   └─ report_activity ─────────────────────────┐  │   │
└───────────────────────────────────────────────────────────────│──│───┘
                                                                │  │
           ┌────────────────────────────────────────────────────┘  │
           │                                                        │
           ▼                                                        ▼
┌──────────────────────┐                              ┌────────────────────────┐
│  AKS Cluster          │                              │  Azure Container        │
│  (pa4-26100356)       │                              │  Instances (per order)  │
│                       │                              │                         │
│  validate-api:v1      │                              │  report-job:v1          │
│  LoadBalancer:        │                              │  ci-report-{order_id}   │
│  20.254.216.37:8080   │                              │  Exits after PDF upload  │
│                       │                              └──────────┬─────────────┘
│  POST /validate       │                                         │ upload PDF
│  → {valid:true/false} │                              ┌──────────▼─────────────┐
└──────────────────────┘                              │  Azure Blob Storage      │
                                                       │  pa426100356             │
┌─────────────────────────────────────────────────────│  /reports/*.pdf          │
│  Azure Container Registry (pa426100356.azurecr.io)  │                          │
│  validate-api:v1  report-job:v1  func-app:v1        │  allowSharedKeyAccess:   │
└─────────────────────────────────────────────────────│  false (RBAC only)       │
                                                       └──────────────────────────┘

Managed Identity: mi-pa4-26100356 (clientId: b118c157-...)
Roles: Storage Blob Data Owner · Storage Queue Data Contributor · Contributor
```

### Question 8.2: Service Selection Rationale

**App Service** hosts the web frontend because it provides a managed, always-on environment for the Node.js Express server without container management overhead. GitHub Actions CI/CD integration is built in, and it scales automatically with App Service plans.

**Durable Functions** orchestrates the multi-step pipeline because it handles long-running activities without timeout issues (ACI creation can take 30–120 seconds), provides durable state checkpointing across restarts, and includes built-in HTTP management APIs for status polling — features that plain serverless functions lack.

**AKS** hosts the validate-api because it provides low-latency, always-available validation with a stable public LoadBalancer IP. The validator must respond immediately for every order (synchronous call from the orchestrator), making a persistent pod the right choice over on-demand compute.

**ACI** runs the report generator because report generation is a one-shot, CPU-intensive batch task. ACI provides per-second billing with no idle cost — the container is created, runs for ~30s, and is deleted, with no resources consumed between orders.

### Question 8.3: ACI vs AKS — Idle Behavior

**AKS validator pod**: Runs indefinitely with no scale-to-zero. Even with zero traffic, the pod holds CPU/memory reservations on the node. The node pool charges continuously regardless of order volume.

```
kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
validate-deployment-5785bcf7fd-knmvs   1/1     Running   0          3h
```

**ACI report job**: Exits with code 0 after uploading the PDF. There is no persistent container — the `report_activity` function creates it, waits for it to finish, and then deletes it. Zero cost between orders.

**Cost implication**: For the validate-api (called on every order), AKS amortizes the node cost across high-frequency usage and provides consistent low latency. For the report job (infrequent, seconds-long runs), ACI's per-second billing is dramatically cheaper than maintaining a dedicated pod or node.

### Question 8.4: Durable Functions vs Plain HTTP Functions

**Problem 1 — Long-running activities**: The ACI report job takes 30–120 seconds to create, run, and clean up. A plain HTTP function would time out. Durable Functions use `yield context.call_activity(...)` to suspend the orchestrator (checkpointing state to Azure Storage) and resume asynchronously when the activity returns — no HTTP timeout.

**Problem 2 — Fault tolerance**: If the function app restarts mid-orchestration, a plain function loses all in-flight state. Durable Functions replay orchestration history from Azure Storage on restart, so the workflow resumes exactly where it left off. Completed activities (e.g., validation) are not re-executed.

**Problem 3 — Status polling**: The client needs to poll for completion asynchronously. Durable Functions provide built-in `statusQueryGetUri` / webhook management APIs for free. A plain function would require custom polling infrastructure (queues, tables, external state store).

### Question 8.5: Cost Review

The most expensive resource in `rg-sp26-26100356` is the **AKS node pool** (Standard_B2s, running continuously ≈ $30–40/month for the node). The ACI report containers are billed per-second for ~30 seconds per order — negligible cost. Azure Functions (Consumption plan alternative) and App Service (B1 plan) are the cheapest components.

To reduce cost for low-traffic scenarios: replace the always-on AKS validator with a Consumption-plan Azure Function or ACI with a persistent container group (which has lower startup time than creating a new container each time).

### Question 8.6: Challenges Faced

**Challenge 1 — Storage shared key disabled by Azure Policy**: The storage account `pa426100356` has `allowSharedKeyAccess: false` enforced. The standard `AzureWebJobsStorage` connection string uses an account key, which was blocked. The fix was configuring five `AzureWebJobsStorage__*` app settings to use the managed identity (`__credential=managedidentity`, `__clientId=b118c157-...`, plus separate blob/queue/table URIs). The managed identity needed `Storage Blob Data Owner`, `Storage Queue Data Contributor`, and `Storage Table Data Contributor` roles at the resource group level.

**Challenge 2 — ACI blocked by storage public network access**: The storage account also had `publicNetworkAccess: Disabled` initially. The Azure Functions host (running in App Service) can reach storage via the Azure internal network, but ACI containers run from public IP space and were blocked. The ACI report job failed with `AuthorizationFailure` until `publicNetworkAccess` was set to `Enabled`, allowing the managed identity credentials to work from the ACI container's public egress IP.

---
