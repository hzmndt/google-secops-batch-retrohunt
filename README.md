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
* Filter specific tenant units or divisions using regex (`--instance-filter "(?i)finance"`).
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
The 3-job concurrency limit is scoped **per tenant instance**. Tenant A executing 3 retrohunts does not consume the quota of Tenant B. The orchestrator allows cross-instance concurrency via `--max-parallel-instances` (default `3` to `5`), allowing multiple tenant instances to be processed in parallel.

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
| **Data Access Scopes** | `chronicle.dataAccessScopes.permit` | **CRITICAL**: Authorizes access to rules and logs bound to custom Data Access Scopes (e.g. `Scope_Restricted_Endpoints`, `Scope_Finance_Data`). Without this, scoped rules return HTTP 403 *"user does not have access to scope"*. |
| | `chronicle.globalDataAccessScopes.permit` | Authorizes access to rules and logs in the default/global scope (`Scope: None`). |
| | `chronicle.dataAccessScopes.list` | Lists and verifies configured scopes across the instance. |
| **Retrohunts** | `chronicle.retrohunts.create` | Initiates the retrohunt job for a rule over historical logs. |
| | `chronicle.retrohunts.get` | Polls progress (`state`, `progressPercentage`) until completion. |
| | `chronicle.retrohunts.list` | Inspects existing retrohunts across the instance. |
| **Detection Rules** | `chronicle.rules.list` | Pre-caches instance rules at startup for fast matching. |
| | `chronicle.rules.get` | Reads rule definitions and properties. |
| | `chronicle.rules.create` | Automatically stages missing rules from YARA-L files in disabled state. |
| | `chronicle.rules.delete` | Removes dynamically created rules when `--cleanup-created-rules` is set. |
| | `chronicle.rules.listRevisions` | Reads and binds specific rule revision versions during retrohunt execution. |
| | `chronicle.rules.verifyRuleText` | Validates YARA-L rule syntax during staging via `:verifyRuleText`. |
| **Rule Deployments** | `chronicle.ruleDeployments.get` | Checks if alerting is enabled on the rule. |
| | `chronicle.ruleDeployments.update` | Temporarily disables alerting in **Safe Mode** to avoid alert floods and restores it afterwards. |
| **Detections & Legacy APIs** | `chronicle.legacies.legacySearchDetections` | Fetches historical detection matches and timestamps for CSV/JSON reports. |
| | `chronicle.legacies.legacyTestRuleStreaming` | Validates and streams rule evaluation across historical telemetry. |
| **Operations** | `chronicle.operations.get` | Monitors long-running API operations for retrohunts. |

### IAM Role Configuration

#### Option A: Predefined Roles
* **`roles/chronicle.admin` (Recommended for Full Access)**:
  Contains all necessary permissions including `chronicle.dataAccessScopes.permit`. Best for central service accounts managing multi-tenant environments.
* **`roles/chronicle.editor`**:
  Contains rule and retrohunt execution permissions, but **lacks `chronicle.dataAccessScopes.permit`**. If your environment uses custom Data Access Scopes (e.g. Scoped Rules), you must also assign a custom role providing `chronicle.dataAccessScopes.permit`.

#### Option B: Least-Privilege Custom Role (Recommended for Automation)
In hardened environments requiring least privilege, create a custom role with the verified 19 permissions:
```bash
gcloud iam roles create SecOpsMultiTenantRetrohuntRunner \
    --project="YOUR_PROJECT_ID" \
    --title="SecOps Multi-Tenant Retrohunt Runner" \
    --description="Permissions to discover tenants, access scoped rules, run batch retrohunts, and collect detections" \
    --permissions="chronicle.tenants.list,chronicle.instances.get,chronicle.dataAccessScopes.permit,chronicle.globalDataAccessScopes.permit,chronicle.dataAccessScopes.list,chronicle.rules.list,chronicle.rules.get,chronicle.rules.create,chronicle.rules.delete,chronicle.rules.listRevisions,chronicle.rules.verifyRuleText,chronicle.ruleDeployments.get,chronicle.ruleDeployments.update,chronicle.retrohunts.create,chronicle.retrohunts.get,chronicle.retrohunts.list,chronicle.legacies.legacySearchDetections,chronicle.legacies.legacyTestRuleStreaming,chronicle.operations.get" \
    --stage="GA"
```

---

## Service Account Permission & Scope Diagnostic Tool

Before launching long-running batch retrohunts, you can verify that your Service Account key possesses all required IAM permissions and Chronicle Data Access Scopes using `test_secops_permissions.py`:

```bash
# 1. Test using your target instances file (e.g. instances.csv) and probe a specific rule
python3 test_secops_permissions.py \
  --credentials-path path/to/service_account.json \
  --instances-file instances.csv \
  --rule-id "ru_11111111-2222-3333-4444-555555555555"

# 2. Test a standalone instance directly
python3 test_secops_permissions.py \
  --credentials-path path/to/service_account.json \
  --customer-id "00000000-0000-0000-0000-000000000000" \
  --project-id "sample-secops-project" \
  --region "asia-southeast1" \
  --rule-id "ru_11111111-2222-3333-4444-555555555555"
```

### Diagnostic Output Example
```
===============================================================================================
GOOGLE SECOPS PERMISSION & SCOPE DIAGNOSTIC REPORT
===============================================================================================
  Target Instance : Instance-Tenant01 (00000000-0000-0000-0000-000000000000)
  GCP Project ID  : sample-secops-project | Region: asia-southeast1
  Auth Identity   : sa-retrohunt@sample-secops-project.iam.gserviceaccount.com
-----------------------------------------------------------------------------------------------
DOMAIN             | IAM PERMISSION                             | STATUS  | DETAILS
-----------------------------------------------------------------------------------------------
Instance Health    | chronicle.instances.get                    | PASS    | Connected to instance...
Data Scopes        | chronicle.globalDataAccessScopes.permit    | PASS    | Global/Default Scope access granted (Scope: None).
Data Scopes        | chronicle.dataAccessScopes.permit          | PASS    | Access verified for 1 custom scope(s).
Detection Rules    | chronicle.rules.list                       | PASS    | Successfully listed rules.
Rule Staging       | chronicle.rules.verifyRuleText             | PASS    | Syntax validation (:verifyRuleText) succeeded.
Rule Access        | chronicle.rules.get                        | PASS    | Successfully retrieved target rule.
Safe Mode          | chronicle.ruleDeployments.get              | PASS    | Deployment state readable. Safe Mode can toggle alerting.
Detections Search  | chronicle.legacies.legacySearchDetections  | PASS    | Detection search endpoint verified.
Partner Discovery  | chronicle.tenants.list                     | PASS    | Customer Management Partner API verified.
-----------------------------------------------------------------------------------------------
✔ ALL REQUIRED PERMISSIONS & DATA ACCESS SCOPES ARE VERIFIED AND READY.
```

If any permission or scope is missing, the tool automatically outputs exact, copy-pasteable remediation steps for both **Google Cloud IAM** and the **Google SecOps Console UI**.

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
Target only specific business units or divisions from the inventory:
```bash
python3 retrohunt_batch.py \
  --instances-file example_instances.csv \
  --instance-filter "Finance Division" \
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

### 5. Running Against Existing Deployed Rules (No Local Files Needed)
Directly retrohunt rules already active in Chronicle using regex filters or explicit rule IDs:
```bash
# Retrohunt deployed rules matching a regex filter
python3 retrohunt_batch.py \
  --customer-id "YOUR_CUSTOMER_ID" \
  --project-id "YOUR_PROJECT_ID" \
  --region "asia-southeast1" \
  --use-instance-rules \
  --rule-filter "(?i)ioc|ransomware" \
  --hours 24

# Retrohunt explicit rule IDs
python3 retrohunt_batch.py \
  --customer-id "YOUR_CUSTOMER_ID" \
  --project-id "YOUR_PROJECT_ID" \
  --region "asia-southeast1" \
  --rule-ids "ru_11111111-2222-3333-4444-555555555555,ru_66666666-7777-8888-9999-000000000000" \
  --hours 24
```

---

## Real-World Validation & Verification

The orchestrator was verified against live enterprise Google SecOps environments, validating customer inventory discovery, scoped data access, safe-mode alerting toggles, quota management (≤3 concurrent jobs), and detection retrieval:

### Environment Test Configuration
* **GCP Project ID**: `sample-secops-project`
* **GCP Project Number**: `123456789012`
* **Customer ID**: `00000000-0000-0000-0000-000000000000`
* **Region**: `asia-southeast1`
* **API Endpoint**: `https://asia-southeast1-chronicle.googleapis.com`

### 1. Discovery & Dry-Run Preview
```bash
python3 retrohunt_batch.py \
  --customer-id "00000000-0000-0000-0000-000000000000" \
  --project-id "sample-secops-project" \
  --region "asia-southeast1" \
  --use-instance-rules \
  --limit 5 \
  --hours 24 \
  --dry-run
```
**Outcome**:
* Successfully connected and cached **1,600 deployed rules** across the instance.
* Evaluated 5 candidate rules and staged dry-run preview entries without triggering active retrohunts or consuming tenant execution slots:
```
---------------------------------------------------------------------------------------------------
INSTANCE                  | RULE NAME                           | STATUS    | DETECTIONS | DURATION
---------------------------------------------------------------------------------------------------
Instance-Tenant01         | test_gti_ioc_domain_alert           | DRY_RUN   | 0          | 0.0    s
Instance-Tenant01         | test_gti_ioc_hostname_match         | DRY_RUN   | 0          | 0.0    s
Instance-Tenant01         | test_cross_instance_alert_verific.. | DRY_RUN   | 0          | 0.0    s
Instance-Tenant01         | Test_Composite_Rule_Detections_V2   | DRY_RUN   | 0          | 0.0    s
Instance-Tenant01         | Test_Sub_Rule_2_Process             | DRY_RUN   | 0          | 0.0    s
---------------------------------------------------------------------------------------------------
TOTAL INSTANCES: 1 | TOTAL RUNS: 5
OUTCOMES: Done: 0, Dry-Run: 5, Failed: 0, Skipped: 0, Timeout: 0
```

### 2. Live Retrohunt Execution
```bash
python3 retrohunt_batch.py \
  --customer-id "00000000-0000-0000-0000-000000000000" \
  --project-id "sample-secops-project" \
  --region "asia-southeast1" \
  --rule-ids "ru_11111111-2222-3333-4444-555555555555" \
  --hours 2 \
  --output-json sample_retrohunt_test.json \
  --output-csv sample_retrohunt_test.csv
```
**Outcome**:
* **Operation Initiated**: `oh_00000000-0000-0000-0000-000000000000`
* **Safe Mode**: Verified and suppressed real-time alerting during retrohunt execution to protect downstream SOAR analysts.
* **Polling & Lifecycle**: Transitioned `RUNNING` (0s) → `DONE` (10s) smoothly.
* **Detections Retrieval**: Queried detections through `chronicle.legacies.legacySearchDetections`.
* **Export**: Generated structured JSON and CSV reports with execution timestamps, status, detection counts, and rule revisions.
```
---------------------------------------------------------------------------------------------------
INSTANCE                  | RULE NAME                           | STATUS    | DETECTIONS | DURATION
---------------------------------------------------------------------------------------------------
Instance-Tenant01         | test_gti_ioc_domain_alert           | DONE      | 0          | 21.4   s
---------------------------------------------------------------------------------------------------
TOTAL INSTANCES: 1 | TOTAL RUNS: 1
OUTCOMES: Done: 1, Dry-Run: 0, Failed: 0, Skipped: 0, Timeout: 0
```

---

## Web UI Dashboard (Google Cloud Run)

A dedicated, real-time web dashboard for visualizing multi-tenant retrohunt execution, quota consumption, and historical detections is available in its own repository:

👉 **[hzmndt/google-secops-retrohunt-ui](https://github.com/hzmndt/google-secops-retrohunt-ui)**

### Highlights
* **Live KPI Cards**: Real-time counts of Target Tenants, Total Queued Rules, Completed Jobs, Active Jobs, and Total Detections.
* **Tenant Auto-Discovery**: Dynamically queries Google SecOps Customer Management / Partner API.
* **Active Execution Matrix**: Sliding-window visualization of active retrohunt worker slots (strictly honoring the ≤3 concurrent jobs limit per instance).
* **Safe Mode & Dry Run**: Web controls to test syntax or run hunts without flooding SOAR alerts.
* **Google Cloud Run Ready**: Complete Dockerfile, Secret Manager integration, and Cloud Run deployment scripts.

See the [Google SecOps Retrohunt UI repository](https://github.com/hzmndt/google-secops-retrohunt-ui) for local setup instructions, REST API documentation, and Cloud Run deployment guides.

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
| `--use-instance-rules` | Flag | `False` | Use rules already deployed in target Chronicle instance(s) |
| `--rule-filter` | Regex | None | Regex filter on deployed rule displayName or rule ID |
| `--rule-ids` | String | None | Comma-separated list of specific Chronicle rule IDs to retrohunt |
| `--rules-pattern` | String | `**/*.yaral` | Glob pattern for finding rule files in `--rules-dir` |
| `--category` | String | None | Filter rules by subfolder (e.g., `microsoft`, `workspace`, `aws`) |
| `--limit` | Integer | None | Max rules to process per instance |
| `--hours` | Integer | `24` | Lookback window in hours (auto-buffered by 1h) |
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
