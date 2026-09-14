# Google SecOps Multi-Tenant Batch Retrohunt Orchestrator

A robust, enterprise-ready automation framework to execute large-scale retrohunts across hundreds of YARA-L detection rules in Google SecOps (Chronicle) using the official [`secops-wrapper`](https://github.com/google/secops-wrapper#retrohunts) Python SDK.

Built for both single-instance operations and multi-tenant enterprise/MSSP architectures (e.g. **50+ multi-tenant SIEM instances**), this orchestrator manages the Google SecOps limit of **at most 3 concurrent retrohunts per instance**, implements automated continuous queueing without manual verification, prevents SOAR alert floods via Safe Mode, and provides checkpoint/resume capabilities.

Compatible with community and custom rules from [chronicle/detection-rules](https://github.com/chronicle/detection-rules).

---

## Architecture: Multi-Tenant Enterprise SIEM (50+ Instances)

In multi-tenant environments such as large enterprises or MSSPs (e.g., Acme Corp MSSP), security teams oversee numerous SIEM instances corresponding to various subsidiaries, business units, or client organizations.

```
                     ┌─────────────────────────────────────────────────────────┐
                     │ Central Orchestrator & Service Account                  │
                     │ (Partner API / Cross-Project Authority)                │
                     └───────────────────────────┬─────────────────────────────┘
                                                 │
            ┌────────────────────────────────────┼───────────────────────────────────┐
            │ Discovery via Partner API          │ Inventory File                    │
            │ (projects.locations.instances.     │ (--instances-file instances.json) │
            │  tenants.list)                     │                                   │
            ▼                                    ▼                                   ▼
┌───────────────────────┐            ┌───────────────────────┐           ┌───────────────────────┐
│ MSSP Tenant #001      │            │ MSSP Tenant #002      │   ...     │ MSSP Tenant #050      │
│ SIEM Instance         │            │ SIEM Instance         │           │ SIEM Instance         │
├───────────────────────┤            ├───────────────────────┤           ├───────────────────────┤
│ Active Worker Queue   │            │ Active Worker Queue   │           │ Active Worker Queue   │
│ [Job 1] [Job 2] [Job 3│            │ [Job 1] [Job 2] [Job 3│           │ [Job 1] [Job 2] [Job 3│
│ (Max 3 Concurrent)    │            │ (Max 3 Concurrent)    │           │ (Max 3 Concurrent)    │
│ Safe Mode: Alerts OFF │            │ Safe Mode: Alerts OFF │           │ Safe Mode: Alerts OFF │
└───────────────────────┘            └───────────────────────┘           └───────────────────────┘
```

### 1. Customer Management / Partner API Dynamic Discovery
Instead of manually collecting API keys or maintaining static lists of dozens of tenant instances, the orchestrator leverages Google SecOps's modern Customer Management API per the [SIEM Endpoint Mapping Table](https://docs.cloud.google.com/chronicle/docs/administration/siem-endpoint-mapping-table):

* **Legacy Endpoint**: `ListCustomersOfPartner`
* **Modern Chronicle API**: `GET /v1alpha/projects/{project}/locations/{location}/instances/{instance}/tenants` (`chronicle.googleapis.com/tenants.list`)
* **Behavior**: The script queries the parent project/instance, automatically discovers all active child tenants, and targets them with unified reporting.

### 2. Static Inventory File Support (JSON / CSV)
For staged rollouts, pilots, or air-gapped change-control workflows:
* Define tenant instances in `instances.json` or `instances.csv`.
* Filter specific agencies or units using regex (`--instance-filter "(?i)finance"`).
* Cap execution to a pilot group (`--instance-limit 5`).

### 3. Unified Authentication Across All Tenants
The same central Service Account credential (`GOOGLE_APPLICATION_CREDENTIALS` or GCP Application Default Credentials) is used to access all child tenants. The parent project's service account possesses administrative or partner authority over the child instances.

---

## Continuous Automated Execution & Concurrency Safety

### Continuous Sliding-Window Queue (No Manual Verification)
A common challenge in SIEM automation is handling the concurrency limit:
* **The Constraint**: Google SecOps enforces a hard limit of **at most 3 concurrent retrohunts per SIEM instance**. Exceeding 3 results in `RESOURCE_EXHAUSTED` (HTTP 429) errors.
* **Naive Batching (Inefficient)**: Waiting for all 3 rules in a batch to finish before triggering the next 3 creates significant dead time if one rule completes in 1 minute while another takes 15 minutes.
* **Orchestrator Continuous Queue (Efficient & Automated)**:
  The orchestrator maintains an active worker pool sized to `--max-concurrent-per-instance 3`. As soon as **any** single retrohunt completes, the next rule in the queue is immediately dispatched. No manual verification, no idle worker slots, and no risk of exceeding the limit.

### Cross-Tenant Parallelism
The 3-job concurrency limit is scoped **per tenant instance**. Tenant A executing 3 retrohunts does not consume the quota of Tenant B. The orchestrator allows cross-instance concurrency via `--max-parallel-instances` (default `3` to `5`), allowing multiple agencies to be processed in parallel.

---

## Operational Risks & Best Practice Safeguards

When automating retrohunts across dozens of instances and hundreds of rules, the following operational risks must be mitigated:

### 1. Alert Flood Prevention (Safe Mode)
* **Risk**: Running a retrohunt on a rule that has alerting enabled will cause Chronicle to generate real-time detections and forward them to Google SecOps SOAR (Siemplify), flooding SOC analyst queues with historical alerts.
* **Safeguard**: **Safe Mode** is enabled by default. Prior to initiating a retrohunt, the orchestrator inspects `ruleDeployments.alerting`. If alerting is active, it automatically sets `alerting: false` on the rule deployment, launches the retrohunt, collects matches strictly as analytical detection records, and restores the original alerting state upon completion.

### 2. Live Pipeline Isolation
* **Risk**: Staging new community detection rules into production instances could cause unverified alerts to fire on incoming live events.
* **Safeguard**: When new rules are staged from disk, the orchestrator creates them with `enabled: false`. They are executed purely against historical logs. An optional `--cleanup-created-rules` flag automatically deletes staged rules after the retrohunt finishes.

### 3. Checkpointing & Fault Tolerance
* **Risk**: A network disconnect or terminal closure during a multi-hour retrohunt across all tenants could cause completed runs to be lost or repeated.
* **Safeguard**: The orchestrator maintains atomic checkpoint state (`instance_id::rule_name`). If interrupted, simply rerun the same command with `--checkpoint-file <path>` to skip all previously completed rules.

---

## Required IAM Permissions & Roles

### Summary of Required Permissions

| API Resource / Domain | IAM Permission | Purpose in Script |
| :--- | :--- | :--- |
| **Partner Discovery** | `chronicle.tenants.list` | Discovers all child tenant instances from the parent instance. |
| | `chronicle.instances.get` | Validates instance health and location settings. |
| **Retrohunts** | `chronicle.retrohunts.create` | Initiates the retrohunt job for a rule over historical logs. |
| | `chronicle.retrohunts.get` | Polls progress (`state`, `progressPercentage`) until completion. |
| | `chronicle.retrohunts.list` | Inspects existing retrohunts across the instance. |
| **Detection Rules** | `chronicle.rules.list` | Pre-caches instance rules at startup for fast matching. |
| | `chronicle.rules.get` | Reads rule definitions and properties. |
| | `chronicle.rules.create` | Automatically stages missing rules from YARA-L files in disabled state. |
| | `chronicle.rules.delete` | Removes dynamically created rules when `--cleanup-created-rules` is set. |
| **Rule Deployments** | `chronicle.ruleDeployments.get` | Checks if alerting is enabled on the rule. |
| | `chronicle.ruleDeployments.update` | Temporarily disables alerting in **Safe Mode** to avoid alert floods and restores it afterwards. |
| **Detections** | `chronicle.detections.list` | Fetches historical detection results and timestamps for CSV/JSON reports. |
| **Operations** | `chronicle.operations.get` | Monitors long-running API operations for retrohunts. |

### IAM Role Configuration

#### Option A: Predefined Role (Recommended)
Grant **Chronicle API Editor** (`roles/chronicle.editor`) to the service account on the parent project and across the target tenant instances:
* `roles/chronicle.editor` (**Chronicle API Editor**)

#### Option B: Least-Privilege Custom Role
In hardened environments, define a custom role:
```bash
gcloud iam roles create SecOpsMultiTenantRetrohuntRunner \
    --project="YOUR_PARENT_PROJECT_ID" \
    --title="SecOps Multi-Tenant Retrohunt Runner" \
    --description="Minimal permissions to discover tenants, run batch retrohunts, and collect detections" \
    --permissions="chronicle.tenants.list,chronicle.instances.get,chronicle.rules.list,chronicle.rules.get,chronicle.rules.create,chronicle.rules.delete,chronicle.ruleDeployments.get,chronicle.ruleDeployments.update,chronicle.retrohunts.create,chronicle.retrohunts.get,chronicle.retrohunts.list,chronicle.detections.list,chronicle.operations.get" \
    --stage="GA"
```

---

## Step-by-Step Setup Guide

### 1. Clone Repository & Install Dependencies
```bash
git clone https://github.com/hzmndt/google-secops-batch-retrohunt.git
cd google-secops-batch-retrohunt
pip install -r requirements.txt
```

### 2. Configure Service Account & Authentication
```bash
# Set Google Application Credentials
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/service_account.json"

# (Optional) Alternatively rely on gcloud ADC:
# gcloud auth application-default login
```

### 3. Clone Sample Detection Rules
```bash
git clone https://github.com/chronicle/detection-rules.git
```

---

## Usage Examples

### 1. Multi-Tenant: Auto-Discover Tenant Instances via Partner API
Automatically queries the Customer Management Partner API to fetch all child tenant instances and executes 5 rules across each instance:
```bash
python3 retrohunt_batch.py \
  --discover-tenants \
  --parent-instance "YOUR_PARENT_INSTANCE_ID" \
  --parent-project "YOUR_PARENT_PROJECT_ID" \
  --region "asia-southeast1" \
  --rules-dir detection-rules/rules/community/microsoft \
  --limit 5 \
  --hours 24 \
  --max-concurrent-per-instance 3 \
  --max-parallel-instances 5 \
  --output-json multi_tenant_partner_results.json \
  --output-csv multi_tenant_partner_results.csv
```

### 2. Multi-Tenant: Run via Static Inventory File (JSON / CSV)
Use a curated list of tenant instances (see `example_instances.json` and `example_instances.csv`):
```bash
python3 retrohunt_batch.py \
  --instances-file example_instances.json \
  --rules-dir detection-rules/rules/community/workspace \
  --days 7 \
  --max-concurrent-per-instance 3 \
  --max-parallel-instances 3 \
  --checkpoint-file retrohunt_checkpoint.json \
  --output-json multi_tenant_results.json \
  --output-csv multi_tenant_results.csv
```

### 3. Filter Specific Tenant Instances (Regex)
Target only specific agencies or ministries from the inventory:
```bash
python3 retrohunt_batch.py \
  --instances-file example_instances.csv \
  --instance-filter "Ministry of Finance" \
  --rules-dir detection-rules/rules/community/aws \
  --limit 10 \
  --hours 48
```

### 4. Single-Instance Dry-Run (Preview & Syntax Validation)
Validates rule syntax and checks instance rule cache without triggering retrohunts:
```bash
python3 retrohunt_batch.py \
  --customer-id "YOUR_CUSTOMER_ID" \
  --project-id "YOUR_PROJECT_ID" \
  --region "asia-southeast1" \
  --rules-dir detection-rules/rules/community/workspace \
  --limit 5 \
  --dry-run
```

---

## Web UI Dashboard (Google Cloud Run)

An interactive web dashboard built with Flask, Bootstrap 5 (Dark Mode), and live polling is provided under `ui/` to visualize batch retrohunt execution, quota consumption, and detection results across tenants in real time.

### Features
* **Live KPI Cards**: Displays real-time counts of Target Tenants, Total Queued Rules, Completed Jobs, Active Jobs, and Total Detections.
* **Tenant Auto-Discovery**: Dynamically fetches child tenants from Google SecOps using the Customer Management / Partner API.
* **Active Execution Matrix**: Shows live running jobs per tenant with rule IDs, elapsed time, and animated status badges.
* **Job Results Table**: Real-time tabular breakdown showing status (`DONE`, `FAILED`, `TIMEOUT`), execution duration, and detection counts per rule.
* **Live Log Stream**: Terminal log console updating automatically with server-side events and Chronicle status transitions.

### Running Locally
```bash
export PARENT_INSTANCE_ID="YOUR_PARENT_INSTANCE_ID"
export PARENT_PROJECT_ID="YOUR_PARENT_PROJECT_ID"
export SECOPS_REGION="asia-southeast1"
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/service_account.json"

pip install -r requirements.txt flask gunicorn
python3 ui/app.py
```
Open `http://localhost:8080` in your browser.

### Deploying to Google Cloud Run
```bash
# Build and push the container image
gcloud builds submit --tag asia-southeast1-docker.pkg.dev/PROJECT_ID/REPO_NAME/secops-retrohunt-ui:latest

# Deploy to Cloud Run with Secret Manager mounting the service account key
gcloud run deploy secops-retrohunt-ui \
  --image=asia-southeast1-docker.pkg.dev/PROJECT_ID/REPO_NAME/secops-retrohunt-ui:latest \
  --region=asia-southeast1 \
  --set-secrets=GOOGLE_APPLICATION_CREDENTIALS=secops-retrohunt-sa-key:latest \
  --service-account=secops-retrohunt-sa@PROJECT_ID.iam.gserviceaccount.com \
  --no-cpu-throttling \
  --min-instances=1 \
  --allow-unauthenticated
```

---

## CLI Reference

| Parameter | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `--discover-tenants` | Flag | `False` | Discover child instances via Customer Management Partner API |
| `--parent-instance` | String | None | Parent Chronicle instance ID for partner tenant discovery |
| `--parent-project` | String | None | Parent GCP Project ID for partner tenant discovery |
| `--instances-file` | String | None | Path to JSON or CSV instance inventory file |
| `--instance-filter` | Regex | None | Filter target instances by display name or customer code |
| `--instance-limit` | Integer | None | Limit number of instances to target (useful for pilots) |
| `--customer-id` | String | `YOUR_CUSTOMER_ID` | Fallback single instance customer UUID |
| `--project-id` | String | `YOUR_PROJECT_ID` | Fallback single instance GCP project ID |
| `--region` | String | `us` | Chronicle region (`asia-southeast1`, `us`, `europe`, etc.) |
| `--credentials-path` | String | None | Path to Service Account JSON key (falls back to ADC) |
| `--rules-dir` | String | None | Path to directory containing `.yaral` rule files |
| `--rules-pattern` | String | `**/*.yaral` | Glob pattern for finding rule files in `--rules-dir` |
| `--category` | String | None | Filter rules by subfolder (e.g., `microsoft`, `workspace`, `aws`) |
| `--limit` | Integer | None | Max rules to process per instance |
| `--hours` | Integer | `24` | Lookback window in hours |
| `--days` | Integer | None | Lookback window in days (takes precedence over `--hours`) |
| `--start-time` | RFC3339 | None | Explicit start time (`YYYY-MM-DDTHH:MM:SSZ`) |
| `--end-time` | RFC3339 | None | Explicit end time (`YYYY-MM-DDTHH:MM:SSZ`) |
| `--max-concurrent-per-instance` | Choice (1, 2, 3) | `3` | Maximum active retrohunts per instance (hard limit is 3) |
| `--max-parallel-instances` | Integer | `3` | Number of instances to execute concurrently in parallel |
| `--poll-interval` | Integer | `10` | Status polling interval in seconds |
| `--timeout` | Integer | `600` | Timeout per retrohunt job in seconds |
| `--dry-run` | Flag | `False` | Check syntax and staging without executing retrohunts |
| `--no-safe-mode` | Flag | `False` | Disable Safe Mode (WARNING: alerts will fire into SOAR) |
| `--cleanup-created-rules` | Flag | `False` | Delete newly staged temporary rules upon completion |
| `--checkpoint-file` | String | Auto | Checkpoint path for resuming interrupted multi-tenant runs |
| `--output-json` | String | Auto | Output path for comprehensive JSON report |
| `--output-csv` | String | Auto | Output path for tabular CSV detection report |

---

## License

Apache 2.0
