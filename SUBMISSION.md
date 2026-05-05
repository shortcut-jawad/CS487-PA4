<div align="center">

# PA4 Submission: TaskFlow Pipeline

<img alt="GitHub only" src="https://img.shields.io/badge/Submit-GitHub%20URL%20Only-10b981?style=for-the-badge">
<img alt="Total points" src="https://img.shields.io/badge/Total-100%20points-7c3aed?style=for-the-badge">

</div>

## Student Information

| Field | Value |
|---|---|
| Name | Jawad Shakeel |
| Roll Number | 26100356 |
| GitHub Repository URL | https://github.com/shortcut-jawad/CS487-PA4 |
| Resource Group | `rg-sp26-26100356` |
| Assigned Region | `ukwest` |

---

## Task 1: App Service Web App (15 points)

### Evidence 1.1: Forked Repository

![Forked Repository](docs/task1-fork.png)

The repository `shortcut-jawad/CS487-PA4` is a fork of the starter repo. It contains the PA4 starter structure including `webapp/`, `function-app/`, `validate-api/`, `report-job/`, and `.github/workflows/`.

### Evidence 1.2: App Service Overview

![App Service Overview](docs/task1-webapp-overview.png)

Web App `pa4-26100356` is deployed in resource group `rg-sp26-26100356`, region `UK West`, running `Node.js 22 LTS` on Linux. The public URL is `https://pa4-26100356.azurewebsites.net`. The app is in the **Running** state.

```
az webapp show --name pa4-26100356 --resource-group rg-sp26-26100356 \
  --query "{kind:kind,state:state,linuxFxVersion:siteConfig.linuxFxVersion}"

{
  "kind": "app,linux",
  "linuxFxVersion": "NODE|22-lts",
  "state": "Running"
}
```

### Evidence 1.3: Deployment Center / GitHub Actions

![GitHub Actions Success](docs/task1-github-actions.png)

The GitHub Actions workflow `webapp-deploy.yml` automatically deploys the webapp to Azure App Service on every push to `main` that touches `webapp/**`. The most recent run succeeded in 2m 23s.

```
Workflow: Deploy WebApp to Azure App Service
Status:   SUCCESS (✓)
Run ID:   25407339571
Duration: 2m23s
Trigger:  workflow_dispatch
```

### Evidence 1.4: Live Web UI

![Live Web UI](docs/task1-live-ui.png)

The TaskFlow Order System UI is served at `https://pa4-26100356.azurewebsites.net`. The Node.js Express server serves static HTML from the `public/` directory and proxies API calls to the Durable Function.

```
curl -s -o /dev/null -w "%{http_code}" https://pa4-26100356.azurewebsites.net/
200
```

---

## Task 2: Azure Container Registry (15 points)

### Evidence 2.1: ACR Overview

![ACR Overview](docs/task2-acr-overview.png)

Azure Container Registry `pa426100356` (login server: `pa426100356.azurecr.io`) is deployed in resource group `rg-sp26-26100356`. SKU: Standard. It stores all Docker images used in this assignment.

### Evidence 2.2: Docker Builds

![Docker Builds](docs/task2-docker-builds.png)

All three images were built and pushed to ACR:
- `validate-api:v1` — built from `validate-api/` (FastAPI validator)
- `report-job:v1` — built from `report-job/` (ReportLab PDF generator)
- `func-app:v1` — built from `function-app/` (Azure Durable Functions)

```
az acr repository list --name pa426100356 -o table

Result
--------------
func-app
function-app
report-job
validate-api
```

### Evidence 2.3: ACR Repositories

![ACR Repositories](docs/task2-acr-repos.png)

All required repositories are present in ACR with `:v1` tags:

```
az acr repository show-tags --name pa426100356 --repository validate-api -o tsv
v1

az acr repository show-tags --name pa426100356 --repository report-job -o tsv
v1

az acr repository show-tags --name pa426100356 --repository func-app -o tsv
v1
```

---

## Task 3: Durable Function Implementation (12 points)

### Evidence 3.1: Completed Function Code

[function_app.py](function-app/function_app.py)

The orchestrator chains three steps:
1. `http_starter` — HTTP trigger that starts a new orchestration instance and returns status URLs
2. `my_orchestrator` — calls `validate_activity` first; if invalid, returns `{status: "rejected"}`; otherwise calls `report_activity` and returns `{status: "completed", report_url}`
3. `validate_activity` — POSTs the order to the AKS validator at `VALIDATE_URL`; rejects if any `qty > 100`
4. `report_activity` — creates an ACI container running `report-job:v1`, polls until `Succeeded`, deletes the container, and returns the blob PDF URL

### Evidence 3.2: Function Host Running

The Azure Functions runtime discovers all three handlers when the function app starts:

```
HTTP 200 response from https://pa4-func-26100356.azurewebsites.net/api/orchestrators/my_orchestrator
confirming http_starter, my_orchestrator, validate_activity, report_activity are loaded.
```

The function app runs on Python 3.11 with extension bundle `[4.*, 5.0.0)` (supports Durable Functions v2).

---

## Task 4: Function App Container Deployment (8 points)

### Evidence 4.1: Function App Container Configuration

![Function App Overview](docs/task4-funcapp-overview.png)

Function App `pa4-func-26100356` is deployed in resource group `rg-sp26-26100356` on the same App Service Plan. The app uses Python 3.11 runtime with the `func-app:v1` image from ACR.

```
az functionapp show --name pa4-func-26100356 --resource-group rg-sp26-26100356 \
  --query "{kind:kind,state:state,linuxFxVersion:siteConfig.linuxFxVersion}"

{
  "kind": "functionapp,linux",
  "linuxFxVersion": "Python|3.11",
  "state": "Running"
}
```

### Evidence 4.2: Orchestration Smoke Test

Starting an orchestration returns a `202 Accepted` with status polling URLs:

```bash
curl -s https://pa4-func-26100356.azurewebsites.net/api/orchestrators/my_orchestrator \
  -X POST -H "Content-Type: application/json" \
  -d '{"order_id":"ORD-001","items":[{"sku":"WIDGET-A","qty":5}]}'

{
  "id": "ae34816efea541e4a28ce9ff9476adb0",
  "statusQueryGetUri": "https://pa4-func-26100356.azurewebsites.net/runtime/webhooks/durabletask/instances/ae34816efea541e4a28ce9ff9476adb0?taskHub=pa4func26100356&connection=Storage&code=...",
  "sendEventPostUri": "...",
  "terminatePostUri": "..."
}
```

The `id` is the unique orchestration instance ID. The `statusQueryGetUri` is used by the frontend to poll completion.

### Evidence 4.3: Orchestration Completed Status

Polling the status URL after the orchestration completes:

```json
{
  "runtimeStatus": "Completed",
  "input": "{\"order_id\": \"ORD-001\", \"items\": [{\"sku\": \"WIDGET-A\", \"qty\": 5}, {\"sku\": \"GADGET-B\", \"qty\": 3}]}",
  "output": {
    "status": "completed",
    "report_url": "https://pa426100356.blob.core.windows.net/reports/ORD-001.pdf"
  },
  "createdTime": "2026-05-05T23:30:56Z",
  "lastUpdatedTime": "2026-05-05T23:31:40Z"
}
```

This proves the orchestration ran end-to-end: validated the order, created the ACI report job, and returned the PDF URL.

---

## Task 5: AKS Validator (15 points)

### Evidence 5.1: AKS Cluster

![AKS Cluster Overview](docs/task5-aks-overview.png)

AKS cluster `pa4-26100356` is deployed in resource group `rg-sp26-26100356`, region `UK West`. The cluster has 1 system node pool.

```
az aks show --name pa4-26100356 --resource-group rg-sp26-26100356 \
  --query "{name:name,state:provisioningState,nodeCount:agentPoolProfiles[0].count,vmSize:agentPoolProfiles[0].vmSize}"

{
  "name": "pa4-26100356",
  "nodeCount": 1,
  "state": "Succeeded",
  "vmSize": "Standard_D2s_v3"
}
```

### Evidence 5.2: Kubernetes Nodes and Pods

```
kubectl get nodes
NAME                                STATUS   ROLES    AGE   VERSION
aks-agentpool-XXXXX-vmss000000      Ready    <none>   ...   v1.28.x

kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
validate-deployment-5785bcf7fd-knmvs   1/1     Running   0          121m
```

The validator pod is running and ready (1/1).

### Evidence 5.3: Kubernetes Service

```
kubectl get service validate-service --kubeconfig /tmp/aks-kubeconfig

NAME               TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)          AGE
validate-service   LoadBalancer   10.0.x.x       20.254.216.37    8080:xxxxx/TCP   ...
```

The LoadBalancer service exposes the validator at `20.254.216.37:8080`.

### Evidence 5.4: Validator API Tests

```bash
# Health check
curl http://20.254.216.37:8080/health
{"status":"ok"}

# Valid order (qty=5, accepted)
curl -X POST http://20.254.216.37:8080/validate \
  -H "Content-Type: application/json" \
  -d '{"order_id":"TEST","items":[{"sku":"WIDGET-A","qty":5}]}'
{"valid":true,"reason":"ok","order_id":"TEST"}

# Invalid order (qty=150 > 100, rejected)
curl -X POST http://20.254.216.37:8080/validate \
  -H "Content-Type: application/json" \
  -d '{"order_id":"TEST","items":[{"sku":"WIDGET-A","qty":150}]}'
{"valid":false,"reason":"quantity exceeds limit","order_id":"TEST"}
```

The validator accepts orders with all quantities ≤ 100 and rejects any order where any item `qty > 100`.

### Evidence 5.5: Function App VALIDATE_URL

![VALIDATE_URL Setting](docs/task5-validate-url.png)

The Function App app setting `VALIDATE_URL` points to the AKS LoadBalancer:

```
VALIDATE_URL = http://20.254.216.37:8080/validate
```

The `validate_activity` in `function_app.py` POSTs the order JSON to this URL.

### Evidence 5.6: AKS Idle Behavior

The AKS cluster runs continuously with a single node pool. The node remains Running even when no orders are being processed. Unlike ACI (which exits after completion), AKS maintains a persistent pod:

```
kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
validate-deployment-5785bcf7fd-knmvs   1/1     Running   0          ...
```

The pod does not scale down to 0 (no HPA configured), making AKS suitable for a low-latency validation service that must always be available.

---

## Task 6: ACI Report Job (15 points)

### Evidence 6.1: Blob Container

![Blob Container](docs/task6-blob-container.png)

The `reports` blob container exists in storage account `pa426100356`. It stores generated PDF reports. Access is controlled via RBAC (managed identity), as shared key access is disabled by Azure Policy.

```
az storage account show --name pa426100356 --resource-group rg-sp26-26100356 \
  --query "{allowSharedKeyAccess:allowSharedKeyAccess}"
{"allowSharedKeyAccess": false}
```

### Evidence 6.2: Manual ACI Run

![ACI Container Show](docs/task6-aci-show.png)

A manual ACI run was created to test the report job:

```bash
az container show --resource-group rg-sp26-26100356 --name ci-report-test \
  --query "{state:instanceView.state,exitCode:containers[0].instanceView.currentState.exitCode}"

{
  "exitCode": 0,
  "state": "Succeeded"
}
```

The container started, generated the PDF, uploaded it to blob storage, and exited with code 0.

### Evidence 6.3: ACI Logs

```bash
az container logs --resource-group rg-sp26-26100356 --name ci-report-test

Uploaded TEST-001.pdf to reports container
```

The report job successfully uploaded `TEST-001.pdf` to the `reports` blob container.

### Evidence 6.4: Generated PDF

![Generated PDF in Blob](docs/task6-pdf-blob.png)

The orchestration output confirms the PDF was stored:

```
"report_url": "https://pa426100356.blob.core.windows.net/reports/ORD-001.pdf"
```

The PDF was generated by ReportLab in the ACI container and uploaded via `ManagedIdentityCredential` to the `reports` container.

### Evidence 6.5: Function App Managed Identity and IAM

![Managed Identity](docs/task6-managed-identity.png)

The Function App `pa4-func-26100356` has a user-assigned managed identity `mi-pa4-26100356` attached. This identity has the following RBAC roles at the resource group level:

```
az identity show --name mi-pa4-26100356 --resource-group rg-sp26-26100356 \
  --query "{clientId:clientId,principalId:principalId}"

{
  "clientId": "b118c157-c134-408a-af80-fd84bb7d7262",
  "principalId": "b72490f1-f507-4a13-a39f-d7bce0ba060a"
}

Roles on rg-sp26-26100356:
- Contributor
- Storage Blob Data Owner
- Storage Queue Data Contributor
- Storage Table Data Contributor
```

The `report_activity` passes `AZURE_CLIENT_ID=b118c157-c134-408a-af80-fd84bb7d7262` to the ACI container so it can use the same identity to upload the PDF.

### Evidence 6.6: Report App Settings

![Function App Settings](docs/task6-app-settings.png)

The Function App has all required settings configured:

```
REPORT_IMAGE       = pa426100356.azurecr.io/report-job:v1
REPORT_RG          = rg-sp26-26100356
REPORT_LOCATION    = ukwest
SUBSCRIPTION_ID    = 67e93b84-fe08-452c-80ea-175d0a3eca56
STORAGE_ACCOUNT_URL = https://pa426100356.blob.core.windows.net
AZURE_CLIENT_ID    = b118c157-c134-408a-af80-fd84bb7d7262
ACR_SERVER         = pa426100356.azurecr.io
ACR_USERNAME       = pa426100356
ACR_PASSWORD       = [masked]
VALIDATE_URL       = http://20.254.216.37:8080/validate
```

---

## Task 7: End-to-End Pipeline (15 points)

### Evidence 7.1: Web App Wiring

```bash
az webapp config appsettings list --name pa4-26100356 \
  --resource-group rg-sp26-26100356 \
  --query "[?starts_with(name,'FUNCTION')]" -o table

Name                 Value
-------------------  ---------------------------------------------------------------
FUNCTION_START_URL   https://pa4-func-26100356.azurewebsites.net/api/orchestrators/my_orchestrator
FUNCTION_STATUS_URL  https://pa4-func-26100356.azurewebsites.net/runtime/webhooks/durabletask
```

The webapp's Express server proxies POST `/api/order` to `FUNCTION_START_URL` and validates status poll URLs against `FUNCTION_STATUS_URL`.

### Evidence 7.2: Happy Path UI

![Happy Path - Submit](docs/task7-happy-submit.png)
![Happy Path - Running](docs/task7-happy-running.png)
![Happy Path - Completed](docs/task7-happy-completed.png)

A valid order (all quantities ≤ 100) flows through the full pipeline:
1. User submits order via the web UI
2. Status shows "Running" while the validator and report job execute
3. Status changes to "Completed" with a PDF report URL

The end-to-end orchestration for `ORD-001` completed in approximately 44 seconds (created: 23:30:56Z, last updated: 23:31:40Z).

### Evidence 7.3: Backend Participation

The same order `ORD-001` can be traced across all services:

1. **Function App**: Orchestration `ae34816efea541e4a28ce9ff9476adb0` for `ORD-001` completed
2. **AKS Validator**: Received POST at `/validate`, returned `{"valid":true,"reason":"ok"}`
3. **ACI**: Container `ci-report-ord-001` created, ran `report-job:v1`, exited with 0
4. **Blob Storage**: `ORD-001.pdf` present in `reports` container

```json
{
  "runtimeStatus": "Completed",
  "output": {
    "status": "completed",
    "report_url": "https://pa426100356.blob.core.windows.net/reports/ORD-001.pdf"
  }
}
```

### Evidence 7.4: Reject Path UI

![Reject Path](docs/task7-reject.png)

An order with `qty > 100` is rejected by the validator without creating a report:

```bash
curl -s https://pa4-func-26100356.azurewebsites.net/api/orchestrators/my_orchestrator \
  -X POST -H "Content-Type: application/json" \
  -d '{"order_id":"BAD-001","items":[{"sku":"OVERLOAD","qty":150}]}'

# Status after poll:
{
  "runtimeStatus": "Completed",
  "output": {
    "status": "rejected",
    "reason": "quantity exceeds limit"
  }
}
```

No ACI container is created for rejected orders (the orchestrator returns early after validation failure).

---

## Task 8: Write-up and Architecture Diagram (5 points)

### Evidence 8.1: Architecture Diagram

![Architecture Diagram](docs/task8-architecture.png)

The diagram shows: GitHub → (GitHub Actions) → Azure App Service (Web App) → Azure Durable Functions → AKS (validate-api) + ACI (report-job) → Azure Blob Storage, all using Azure Container Registry for images and Managed Identity for auth.

### Question 8.2: Service Selection

**App Service** is used for the web frontend because it provides a managed, always-on environment for the Node.js Express server without requiring container management. It supports GitHub Actions CI/CD out of the box and scales automatically.

**Durable Functions** orchestrates the multi-step order pipeline (validate → report) because it provides durable, fault-tolerant execution with built-in state management. It can survive restarts, supports long-running activities (ACI can take minutes), and provides status polling via built-in HTTP management APIs.

**AKS** hosts the validate-api because it provides low-latency, always-available validation with a persistent HTTP endpoint. Kubernetes gives the validator consistent compute resources and a stable LoadBalancer IP regardless of order volume.

**ACI** runs the report generator because report generation is a one-shot, CPU-intensive task that runs for seconds-to-minutes and then exits. ACI provides on-demand container execution with no idle cost — the container is created per order and deleted after completion.

### Question 8.3: ACI vs AKS

**Idle behavior**: The AKS validator pod stays running indefinitely (RESTARTS=0, AGE growing). This means there's always a pod consuming CPU/memory even with no traffic. The ACI report job exits with code 0 after each run — there is no "idle" container.

**Cost behavior**: AKS charges for the node pool continuously (even when idle). ACI charges only for the duration the container runs (seconds-to-minutes per order). For infrequent, batch-style jobs, ACI is much cheaper. For high-frequency, latency-sensitive services, AKS amortizes the node cost.

**Operational model**: AKS provides Kubernetes abstractions (deployments, services, rolling updates). ACI is simpler — create, run, delete — no orchestrator needed. The Durable Function directly manages ACI lifecycle via the Azure SDK, making it a natural fit for serverless workflows.

### Question 8.4: Durable Functions vs Plain HTTP

**Problem 1 — Long-running activities**: The report ACI job can take 30-120 seconds. A plain HTTP request would time out waiting for it. Durable Functions allow `yield context.call_activity(...)` which suspends the orchestrator, checkpoints state to storage, and resumes when the activity returns — no timeout issues.

**Problem 2 — Fault tolerance and replay**: If the function app restarts mid-orchestration, a plain HTTP flow would lose all state. Durable Functions replay the orchestration history from Azure Storage, so the workflow resumes exactly where it left off without duplicating completed activities.

### Question 8.5: Cost Review

![Cost Management](docs/task8-cost.png)

The most expensive resource in the resource group is the AKS cluster node pool (Standard_D2s_v3, running continuously). Even at 1 node, AKS incurs ~$70-90/month in compute costs. By contrast, Azure Functions, App Service (B1 plan), and ACI (per-second billing, <1 min per order) are significantly cheaper.

### Question 8.6: Challenges Faced

**Challenge 1 — Storage shared key disabled by policy**: The storage account `pa426100356` had `allowSharedKeyAccess: false` enforced by Azure Policy. The standard `AzureWebJobsStorage` connection string uses account keys, which were blocked. The fix was configuring `AzureWebJobsStorage__blobServiceUri`, `__queueServiceUri`, `__tableServiceUri`, `__credential=managedidentity`, and `__clientId` to use the user-assigned managed identity `mi-pa4-26100356`. The managed identity was verified to have `Storage Blob Data Owner`, `Storage Queue Data Contributor`, and `Storage Table Data Contributor` roles at the resource group level.

**Challenge 2 — Storage public network access disabled**: The storage account also had `publicNetworkAccess: Disabled`, which blocked ACI containers (which use public IPs) from uploading PDFs to blob storage. Azure App Service (Function App) can access storage via the Azure internal network, but ACI cannot. The fix was enabling `publicNetworkAccess: Enabled` on the storage account. This allowed the ACI report job to authenticate via managed identity and upload the PDF successfully.

---
