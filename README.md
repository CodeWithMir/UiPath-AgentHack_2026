# ClaimAgent: AI-Orchestrated Reimbursement Pipeline

**ClaimAgent** is an end-to-end, autonomous expense reimbursement pipeline built for the **UiPath AgentHack Track 1 (Reimbursement, Maestro Case)**. It leverages the full **UiPath Platform** — Coded Agents (Python + LangGraph), API Workflows (Serverless DSL), Document Understanding, and Coded Action Apps (React/Vite) — to process expense claims with speed, flexibility, and absolute auditability.

---

## 📖 Table of Contents
1. [Overview & Stage Architecture](#-overview--stage-architecture)
2. [UiPath Components Used](#-uipath-components-used)
3. [Folder Structure](#-folder-structure)
4. [Technology Stack](#-technology-stack)
5. [Deployment Instructions](#-deployment-instructions)
6. [Local Execution & Verification](#-local-execution--verification)
7. [Key Design Choices](#-key-design-choices)
8. [Challenges & Workarounds](#-challenges--workarounds)

---

## 🏗️ Overview & Stage Architecture

ClaimAgent automates the entire reimbursement lifecycle across **6 stages**, all orchestrated and deployed on the UiPath Platform:

```
[Stage 1: Intake RPA] ──► [Stage 2: IDP Extract] ──► [Stage 3: AI Classification Agent]
                                                             │
             ┌───────────────────────────────────────────────┘
             ▼
[Stage 4: Policy Check Bot] ──► Auto-Approve ──────────► [Stage 5a: Stripe Payout] ──► [Stage 6a: Success Notify]
             │
             ├──► Reject ────────────────────────────────► [Stage 6b: Rejection Notify]
             │
             └──► Manager Review ──► [Stage 5: HITL Action App]
                                          ├──► Approved ──► [Stage 5a: Stripe Payout]
                                          └──► Rejected ──► [Stage 6b: Rejection Notify]
```

| Stage | UiPath Artifact Type | Description |
|---|---|---|
| 1 — Intake | **Cross-Platform RPA Process** | Gmail inbox polling, attachment download, Storage Bucket upload |
| 2 — IDP Extract | **Cross-Platform RPA + Document Understanding** | OCR extraction of amount, vendor, date, currency from receipt image |
| 3 — AI Classify | **Coded Agent (Python)** | LangGraph agent — LLM classification + deterministic risk scoring |
| 4 — Policy Check | **API Workflow (JavaScript)** | Deterministic spend limit / date / pre-approval rules engine |
| 5 — Human Review | **Coded Action App (React)** | HITL approval form in Action Center |
| 5a — Payout | **API Workflow (JavaScript)** | Real Stripe PaymentIntent disbursement |
| 6a — Success Notify | **Coded Agent (Python)** | Branded HTML email via Gmail connector |
| 6b — Rejection Notify | **Coded Agent (Python)** | Rejection HTML email with reviewer notes via Gmail connector |

---

## 🟦 UiPath Components Used

This project is built end-to-end on the **UiPath Platform**. Below is how each UiPath feature is leveraged:

### 1. UiPath Orchestrator
The control plane for the entire solution. Used to:
- Host the unified **Solution package** (`ReimbursementApiSolution`) deploying all 5 backend components into `Shared/ReimbursementFullSolution`.
- Manage **Orchestrator Assets** (Text + Credential) for sender names, Reply-To addresses, and Gmail connection IDs — readable at runtime without code changes.
- Provide **Storage Buckets** as the handoff layer between the Intake RPA bot and the Receipt Extractor (bucket name: `Receipt`).
- Run all processes as **Serverless Jobs** (Platform Units), eliminating the need for dedicated Windows robots.

### 2. UiPath Coded Agents (Python + LangGraph)
Three separate coded agents are deployed as UiPath **Agent (python)** process types:

| Agent | LangGraph Nodes | Key UiPath Integration |
|---|---|---|
| `ReimbursementClassificationAgent` | `classify → score_risk` | UiPath LLM Gateway via `UiPathChat` |
| `NotificationAgent` | `write_note → send` | Gmail connector via `sdk.connections.invoke_activity()` |
| `RejectionNotificationAgent` | `write_note → send` | Same Gmail connector — rejection email path |

Each agent is authored with the `uipath` SDK and `uipath-langchain` packages. They are:
- **Evaluated locally** using `uip codedagent eval` with JSON eval sets and `JsonSimilarityEvaluator`.
- **Deployed** to the tenant Orchestrator feed via `uip codedagent deploy`.
- **Mocked** for Gmail sends using the `@mockable` decorator, allowing offline evaluation without live Integration Service connections.

### 3. UiPath LLM Gateway
The `gpt-4o-2024-11-20` model is accessed through the **UiPath LLM Gateway** — a secure, governed proxy. This means:
- No external API keys or direct OpenAI credentials are required.
- All LLM traffic is billed and audited through UiPath.
- Used in **two places**: the `classify` node (temperature `0` for determinism) and the `write_note` node in both notification agents (temperature `0.6` for personalized warmth).

```python
from uipath_langchain.chat.models import UiPathChat
llm = UiPathChat(model="gpt-4o-2024-11-20", temperature=0, max_tokens=1024)
```

### 4. UiPath API Workflows (JavaScript / CNCF Serverless Workflow DSL)
Two API Workflows handle deterministic logic and HTTP integrations:

- **`PolicyRuleCheckWorkflow`** — Pure JavaScript rules engine. Evaluates spend limits, date windows, and pre-approval thresholds. Runs as `type: "api"` in the solution bundle.
- **`StripePayoutWorkflow`** — Makes HTTP calls to `api.stripe.com` using the `UiPath.Http` implicit connector. Creates a Stripe Customer and confirms a PaymentIntent — no manual auth setup needed.

Both are authored as `.json` workflow files and tested locally:
```bash
uip api-workflow run ./PolicyRuleCheckWorkflow/PolicyRuleCheckWorkflow.json --no-auth --input-arguments '{...}'
```

### 5. UiPath Integration Service — Gmail Connector
Both notification agents send emails through the **UiPath Gmail Integration Service connector** — a native, OAuth-governed connection managed in Orchestrator. Invoked via:
```python
from uipath.platform.connections import ActivityMetadata
sdk.connections.invoke_activity(
    activity_metadata=GMAIL_SEND,
    connection_id=connection.id,
    activity_input={"To": to, "Subject": subject, "Body": html_body}
)
```
The connection ID is stored as an Orchestrator Asset in `Shared/ReimbursementFullSolution`, making it accessible to serverless jobs without personal workspace 403 errors.

### 6. UiPath Document Understanding
The `ReceiptExtractor_XP` process uses **Document Understanding** to extract structured receipt data (vendor, amount, date, currency) from uploaded images via the `ReimbursementDU` digitization model and the built-in Receipts extractor.

### 7. UiPath Action Center — Coded Action App
A **Coded Action App** (`classification-approval-app`) built with React + Vite is deployed to UiPath Action Center as the **Human-in-the-Loop (HITL) approval gate**:
- Reads `expenseType`, `riskScore`, `classificationConfidence` from the Maestro Case as read-only inputs.
- Reviewer fills optional `reviewerNotes` before submitting.
- Resolves with `Approve` or `Reject` outcome via `codedActionAppService.completeTask()`.

> **Critical:** Must be deployed to **both** the parent folder and the Case subfolder to avoid AppTasks error `170000`:
```bash
uip codedapp deploy -n classification-approval-app --folder-key 2c638be8-3a05-4f2a-902f-79a605afa6b3
uip codedapp deploy -n classification-approval-app --folder-key eb6b26a5-9d3e-48f7-9675-140651807ade
```

### 8. UiPath Solution Packaging
All 5 backend components are bundled into a single **`.uipx` Solution** (`ReimbursementApiSolution`):
```bash
uip solution pack ./ReimbursementApiSolution ./build --version 3.9.0
uip solution publish /absolute/path/to/build/ReimbursementApiSolution_3.9.0.zip --tenant DefaultTenant
uip solution deploy run \
  --package-name ReimbursementApiSolution \
  --package-version 3.9.0 \
  --name ReimbursementFullSolution \
  --folder-name ReimbursementFullSolution \
  --parent-folder-path Shared \
  --tenant DefaultTenant
```
> ⚠️ API Workflow projects require manually authored `entry-points.json` and `bindings_v2.json` files. The CLI packager does not auto-generate them — without these, deployment fails with error `2005`.

---

## 📁 Folder Structure

Only actively deployed components are listed:

```
├── data/
│   ├── mock_policy.json                  # Policy DB: spend limits, risk rules, keywords
│   └── case_schema.json                  # Shared field contract across all stages
│
├── ReimbursementClassificationAgent/     # Stage 3 — UiPath Coded Agent (Python + LangGraph)
│   ├── main.py                           # LangGraph graph: classify → score_risk
│   ├── data/mock_policy.json             # Local policy copy used by risk scorer
│   ├── evaluations/eval-sets/            # edge-cases.json + smoke-test.json (13 cases)
│   ├── entry-points.json                 # UiPath agent schema contract
│   └── pyproject.toml                    # Python deps (uipath, uipath-langchain, langgraph)
│
├── PolicyRuleCheckWorkflow/              # Stage 4 — UiPath API Workflow (JS DSL)
│   ├── PolicyRuleCheckWorkflow.json      # Workflow definition with JS rules engine
│   └── entry-points.json                 # Required for solution packaging
│
├── StripePayoutWorkflow/                 # Stage 5a — UiPath API Workflow (Stripe HTTP)
│   ├── StripePayoutWorkflow.json         # Workflow: Customer → PaymentIntent → Confirm
│   ├── entry-points.json                 # Required for solution packaging
│   └── bindings_v2.json                  # HTTP connector binding
│
├── NotificationAgent/                    # Stage 6a — UiPath Coded Agent (Gmail success)
│   └── main.py                           # LangGraph: write_note (LLM) → send (Gmail IS)
│
├── RejectionNotificationAgent/           # Stage 6b — UiPath Coded Agent (Gmail rejection)
│   └── main.py                           # Same pattern — rejection-themed HTML email
│
├── classification-approval-app/          # Stage 5 — UiPath Coded Action App (React/Vite)
│   ├── src/components/Form.tsx           # Approval form UI (Approve/Reject + notes)
│   └── action-schema.json                # UiPath action contract (inputs/outputs/outcomes)
│
└── ReimbursementApiSolution/             # Unified solution bundle (.uipx)
    ├── ReimbursementClassificationAgent/ # Bundled Stage 3
    ├── PolicyRuleCheckWorkflow/          # Bundled Stage 4
    ├── StripePayoutWorkflow/             # Bundled Stage 5a
    ├── NotificationAgent/               # Bundled Stage 6a
    └── RejectionNotificationAgent/      # Bundled Stage 6b
```

---

## 💻 Technology Stack

| Category | Technologies |
|---|---|
| **Languages** | Python 3.13, JavaScript, TypeScript |
| **AI / Agent Frameworks** | LangGraph, LangChain |
| **LLM** | GPT-4o via UiPath LLM Gateway |
| **Frontend (Action App)** | React, Vite |
| **UiPath Platform** | Orchestrator, Maestro/Case Management, Document Understanding, Action Center, Integration Service, LLM Gateway, Coded Agents, API Workflows |
| **Payment API** | Stripe (Customer + PaymentIntents, test mode) |
| **Email Connector** | Gmail via UiPath Integration Service |
| **CLI & Tooling** | UiPath CLI (`uip`), UV (Python package manager), NPM |

---

## 🚀 Deployment Instructions

### Prerequisites
```bash
uip --version    # UiPath CLI >= 1.196.0
uv --version     # UV Python package manager
uip user         # Confirm logged in to staging tenant
```

### Step 1: Login to Staging Tenant
```bash
uip login --authority "https://staging.uipath.com/identity_" \
  --organization "hackathon26_332" --tenant "DefaultTenant"
```

### Step 2: Deploy the Unified Solution (Stages 3, 4, 5a, 6a, 6b)
```bash
# Pack — bump --version if this version is already published
uip solution pack ./ReimbursementApiSolution ./build --version 3.9.0

# Publish to tenant feed (absolute path required)
uip solution publish /path/to/build/ReimbursementApiSolution_3.9.0.zip --tenant DefaultTenant

# Deploy + auto-activate under Shared folder
uip solution deploy run \
  --package-name ReimbursementApiSolution \
  --package-version 3.9.0 \
  --name ReimbursementFullSolution \
  --folder-name ReimbursementFullSolution \
  --parent-folder-path Shared \
  --tenant DefaultTenant
```

### Step 3: Deploy the Coded Action App (Stage 5)
```bash
cd classification-approval-app && npm run build

# Deploy to BOTH folders — prevents AppTasks 170000 errors
uip codedapp deploy -n classification-approval-app --folder-key 2c638be8-3a05-4f2a-902f-79a605afa6b3
uip codedapp deploy -n classification-approval-app --folder-key eb6b26a5-9d3e-48f7-9675-140651807ade
```

---

## 🧪 Local Execution & Verification

### Stage 3 — Classification Agent
```bash
cd ReimbursementClassificationAgent
uv venv --python 3.13 && source .venv/bin/activate && uv sync
uip codedagent setup --force   # required after every cd into a new agent project

# Single run — LLM-only path (keyword fallback can't handle "gaming equipment")
uip codedagent run agent '{"case_id":"DEMO-LLM","source_email_body":"gaming equipment for entertainment","amount":450,"document_attached":true,"employee_email":"e@co.com"}'
# Expect: expense_type = "others"

# Full 13-case edge-case evaluation
uip codedagent eval agent evaluations/eval-sets/edge-cases.json --no-report
uip codedagent eval agent evaluations/eval-sets/smoke-test.json --no-report
# Expect: all rows JsonSimilarityEvaluator = 1.0
```

### Stage 4 — Policy Check Workflow
```bash
# Auto-approve: food, low risk, below limit
uip api-workflow run ./PolicyRuleCheckWorkflow/PolicyRuleCheckWorkflow.json --no-auth \
  --input-arguments '{"case_id":"P1","expense_type":"food","amount":500,"currency":"INR","date":"2026-06-04","document_attached":true,"business_purpose_valid":true,"risk_score":"Low","duplicate_detected":false}' \
  --output json
# Expect: routing_decision = "auto_approve"

# Reject — travel over spend limit (>50,000 INR)
uip api-workflow run ./PolicyRuleCheckWorkflow/PolicyRuleCheckWorkflow.json --no-auth \
  --input-arguments '{"case_id":"P2","expense_type":"travel","amount":60000,"currency":"INR","date":"2026-06-04","document_attached":true,"business_purpose_valid":true,"risk_score":"Low","duplicate_detected":false}' \
  --output json
# Expect: routing_decision = "reject", policy_violations = ["over_spend_limit"]
```

### Stage 6 — Notification Agent (requires active Gmail IS connection)
```bash
cd NotificationAgent && source .venv/bin/activate
uip codedagent setup --force
export REIMBURSEMENT_GMAIL_CONNECTION_ID=9291e875-b63f-4d6b-aaf0-84b81f41aa14
uip codedagent run main '{"employee_email":"you@example.com","case_id":"DEMO","payout_status":"succeeded","amount":300,"currency":"INR","save_as_draft":true}'
# Expect: sent = true, Gmail message_id returned
```

---

## 🛡️ Key Design Choices

1. **LLM is a parser, not a decision-maker.** `gpt-4o` handles natural language interpretation (expense category + business purpose validation). Every money-routing decision is enforced by deterministic, auditable code — never by the LLM.

2. **Single policy source of truth.** All spend limits, risk thresholds, and classification keywords live in one editable `mock_policy.json`. Finance teams can update policy without redeploying or retraining.

3. **Full audit trail at every stage.** Every stage emits a structured `audit_entry` block (timestamp, actor, action, details) and an itemized `risk_factors[]` array, making every decision explainable and defensible.

---

## ⚠️ Challenges & Workarounds

| Problem | Root Cause | Fix |
|---|---|---|
| Cloud agents wouldn't execute | `AgentService = 0` on tenant | Built offline `@mockable` evaluation harness; proved logic end-to-end via `uip codedagent eval` |
| RPA bots failed in cloud | Tenant has 0 Robot Units (Windows only) | Converted to Portable (cross-platform) packages; uploaded DU as custom NuGet |
| HITL form gave error `170000` | Action App not deployed in Case's subfolder | Deployed app to **both** parent folder AND Case subfolder |
| `riskScore` displayed `—` in form | Case passes strings; form expected numbers | Added `rawLabel()` fallback + `toNumber()` coercion in React form |
| Stripe rejected payouts | Amount arrived as `0` (IDP didn't populate) | Added regex backfill in Stage 3 to extract amount from email body |
| Gmail `Invalid To header` error | Upstream passed a display name, not a bare address | Added `_clean_email()` parser to strip `Name <addr>` format |
| Solution deploy error `2005` | API Workflow projects lacked `entry-points.json` | Hand-authored `entry-points.json` + `bindings_v2.json` per workflow project |
| Double greeting in emails | LLM generated `Hi Name` + template also had `Hi Name` | Applied `_strip_greeting()` regex filter before injecting LLM output |
