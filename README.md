# ClaimAgent — AI-Orchestrated Reimbursements

> **UiPath AgentHack 2026 · Track 1 · Enterprise Agents**
> Built by Team ClaimAgent (Mir Jasimuddin · Subharjun Bose · Rashmi · Avishek)

---

## Project Description

### The Problem

Corporate expense reimbursement is broken. Employees collect paper receipts, manually type line items into a spreadsheet, email it to their manager, and wait two weeks — often following up multiple times before money arrives. On the finance side, someone manually cross-checks every claim against a policy document, hunts for duplicates, and chases employees for missing receipts. There is no audit trail, policy violations slip through, and valid claims get delayed.

### The Solution

**ClaimAgent** is an end-to-end, no-touch reimbursement pipeline that takes an employee from "I have a receipt" to "money is in my account" — with zero manual data entry and a human always in command before any money moves.

An employee submits an expense via a **web portal** or by **emailing a receipt** to a designated inbox. The system then automatically:

1. **Ingests and Extracts** — downloads the receipt and uses Document Understanding to extract vendor, date, amount, currency, and OCR confidence as structured fields
2. **Classifies and Validates** — a LangGraph AI Agent (gpt-4o via UiPath LLM Gateway) semantically classifies the expense type, validates business justification, and flags potential duplicates; a deterministic fallback keeps the case moving if the LLM is unavailable
3. **Applies Policy Rules** — evaluates the claim against category-specific spend limits, date windows, receipt requirements, and pre-approval thresholds from a policy configuration file
4. **Routes the Decision** — auto-approves low-risk claims under threshold; immediately rejects clear violations; routes borderline cases to a human reviewer in Action Center
5. **Pays and Closes** — on Approve, creates and confirms a Stripe PaymentIntent in real time; sends polished HTML emails to the claimant and finance team on both Approve and Reject paths

All stages are orchestrated by a **UiPath Maestro Case** running on the hackathon cloud tenant as fully serverless jobs.

### Pipeline at a Glance

```
Employee ──▶ Web Form / Email Inbox
                │
                ▼
         ┌─────────────┐    ┌──────────────┐    ┌───────────────┐    ┌──────────────┐
         │  1. Intake  │───▶│ 2. IDP       │───▶│ 3. Classify   │───▶│ 4. Policy    │
         │  (RPA Bot)  │    │  Extraction  │    │  AI Agent     │    │  Check       │
         └─────────────┘    │  (Doc U.)    │    │  (LangGraph)  │    │  (API Wflow) │
                            └──────────────┘    └───────────────┘    └──────┬───────┘
                                                                            │
                                              ┌─────────────────────────────┘
                                              ▼
                                     ┌─────────────────┐
                                     │ 5. Human Review │
                                     │  (Action Center │
                                     │   Approval App) │
                                     └────────┬────────┘
                                    Approve   │   Reject
                              ┌───────────────┘      └───────────────┐
                              ▼                                       ▼
                     ┌────────────────┐                   ┌────────────────────┐
                     │ 6. Stripe      │                   │ 7b. Rejection      │
                     │    Payout      │                   │     Notify Email   │
                     └───────┬────────┘                   └────────────────────┘
                             ▼
                     ┌────────────────┐
                     │ 7. Success     │
                     │    Notify Email│
                     └────────────────┘
```

---

## UiPath Components Used

| Component | How It Is Used |
|---|---|
| **UiPath Maestro (Case Management)** | Orchestrates all 7 stages end-to-end as a Maestro Case (`ReimbursementProcessCase`). Manages stage sequencing, variable passing, HITL routing, and case lifecycle. |
| **UiPath Agent Builder (Coded Agents)** | Three Python + LangGraph coded agents: `ReimbursementClassificationAgent` (Stage 3), `NotificationAgent` (Stage 7), `RejectionNotificationAgent` (Stage 7b). |
| **UiPath API Workflows** | Two JavaScript API workflows: `PolicyRuleCheckWorkflow` (Stage 4 — deterministic policy evaluation) and `StripePayoutWorkflow` (Stage 6 — Stripe payment disbursement). |
| **UiPath Action Center** | Hosts the `classification-approval-app` — a Coded Action App (React/Vite) that presents expense details, risk score, and classification confidence to a human reviewer for Approve / Reject. |
| **UiPath Document Understanding** | `ReceiptExtractor_XP` uses Document Understanding AI models to extract structured fields (vendor, date, amount, currency) from uploaded receipt images. |
| **UiPath RPA (Cross-Platform Bots)** | `ReimbursementIntakeBot_XP` reads Gmail inbox, downloads receipt attachments, uploads to Azure Blob Storage bucket. `ReceiptExtractor_XP` downloads and processes them. Both run serverless (Portable packages). |
| **UiPath Orchestrator** | Hosts all deployed processes, manages serverless job execution, stores the `Receipt` storage bucket, and houses the deployed `ReimbursementApiSolution`. |
| **UiPath Integration Service** | Provides the Gmail connector used by `NotificationAgent` and `RejectionNotificationAgent` to send HTML emails (connection id in solution folder). |
| **UiPath LLM Gateway** | Powers gpt-4o inference for the Classification Agent (expense type, risk scoring, business purpose validation) and Notification Agents (personalised email note generation). |
| **UiPath Solution (`.uipx`)** | `ReimbursementApiSolution` v3.9.0 bundles all 5 sub-projects (Classify + Policy + Stripe + Notify + RejectNotify) into one deployable artifact, active in `Shared/ReimbursementFullSolution`. |

---

## Agent Type

This solution uses **both Coded Agents and Low-code components**:

**Coded Agents (Python + LangGraph):**
- `ReimbursementClassificationAgent` — Stage 3, classifies expense type and scores risk using gpt-4o
- `NotificationAgent` — Stage 7, writes personalised HTML email on payout success
- `RejectionNotificationAgent` — Stage 7b, emails claimant with reviewer's rejection reason

**Low-code / No-code Components:**
- `ReimbursementIntakeBot_XP` — cross-platform RPA workflow (XAML) for Gmail intake
- `ReceiptExtractor_XP` — cross-platform RPA workflow (XAML) with Document Understanding
- `PolicyRuleCheckWorkflow` — API workflow (JavaScript) for deterministic policy rules
- `StripePayoutWorkflow` — API workflow (JavaScript) for Stripe payment disbursement
- `classification-approval-app` — Coded Action App (React/Vite) for human-in-the-loop approval
- `ReimbursementProcessCase` — Maestro Case orchestrating all stages

---

## Repository Layout

```
ClaimAgent/
├── IntakeAndExtraction/              # Stage 1 & 2 — RPA bots (cross-platform / serverless)
│   ├── ReimbursementIntakeBot/       # Gmail → receipt → Azure Blob Storage bucket
│   └── ReceiptExtractor/             # Blob → Document Understanding → out_JSON
├── ReimbursementClassificationAgent/ # Stage 3 — Coded AI Agent (LangGraph + GPT-4o)
├── PolicyRuleCheckWorkflow/          # Stage 4 — API Workflow (deterministic policy check)
├── classification-approval-app/      # Stage 5 — Coded Action App (HITL in Action Center)
├── StripePayoutWorkflow/             # Stage 6 — API Workflow (Stripe PaymentIntent)
├── NotificationAgent/                # Stage 7 — Coded Agent (HTML email on success)
├── RejectionNotificationAgent/       # Stage 7b — Coded Agent (HTML email on reject)
├── ReimbursementApiSolution/         # Deployable solution bundling stages 3–7b
├── MaestroCase/_unpacked/            # Maestro Case source (caseplan.json)
├── IntakeForm/                       # Web intake form — React/Vite + FastAPI (Render)
│   ├── api.py                        # FastAPI backend — bucket upload + Case trigger
│   └── src/IntakeForm.tsx            # React frontend
├── data/
│   ├── mock_policy.json              # Policy rules configuration
│   └── case_schema.json              # Case data schema
└── xp/ReimbursementIntakeBot/        # Cross-platform source for the intake RPA bot
```

---

## Setup Instructions

### Prerequisites

Before running anything, ensure you have:

- **UiPath Cloud account** with Orchestrator, Action Center, Integration Service, Document Understanding, and LLM Gateway enabled
- **UiPath CLI (`uip`)** installed: `npm install -g @uipath/cli`
- **Node.js 18+** and **Python 3.11+**
- A **Gmail Integration Service connection** created in your UiPath tenant
- A free **Stripe test-mode secret key** (`sk_test_…`) from [dashboard.stripe.com](https://dashboard.stripe.com)
- An **Azure Blob Storage bucket** named `Receipt` created in your Orchestrator folder

### Step 1 — Authenticate

```bash
uip login \
  --authority "https://cloud.uipath.com/identity_" \
  --organization "<your-org-name>" \
  --tenant "<your-tenant-name>"
```

Verify with `uip user`. Note your folder key from Orchestrator → Folders.

### Step 2 — Deploy the RPA Bots (Stage 1 & 2)

These run as **serverless cross-platform processes** on Orchestrator.

```bash
# Pack and upload the Intake Bot
uip rpa pack IntakeAndExtraction/ReimbursementIntakeBot ./build
uip or packages upload ./build/ReimbursementIntakeBot.*.nupkg

# Create the serverless process
uip or processes create \
  --name ReimbursementIntakeBot_XP \
  --package-key ReimbursementIntakeBot \
  --package-version <version> \
  --folder-key <your-folder-key>

# Run it (reads Gmail inbox, uploads receipt to bucket)
uip or jobs start <process-key> \
  --folder-key <your-folder-key> \
  --runtime-type Serverless \
  --wait-for-completion
```

> **Note:** Run IntakeBot first — it fills the `Receipt` bucket. ReceiptExtractor reads from it.
> ReceiptExtractor also requires the Document Understanding `ReimbursementDUReceiptsV1` bundle uploaded to your tenant feed.

### Step 3 — Deploy the Solution (Stages 3–7b)

The `ReimbursementApiSolution` bundles all coded agents and API workflows into one package.

```bash
# Pack the solution
uip solution pack ReimbursementApiSolution ./build --version 3.9.0

# Publish to your tenant
uip solution publish ./build/ReimbursementApiSolution_3.9.0.zip --tenant <your-tenant>

# Deploy and activate
uip solution deploy run \
  --package-name ReimbursementApiSolution \
  --package-version 3.9.0 \
  --name ReimbursementFullSolution \
  --folder-name ReimbursementFullSolution \
  --parent-folder-path Shared \
  --tenant <your-tenant>
```

**Before packing**, update the Gmail connection ID in:
- `ReimbursementApiSolution/NotificationAgent/bindings.json` → `ConnectionId.defaultValue`
- `ReimbursementApiSolution/RejectionNotificationAgent/bindings.json` → same

Set it to your Gmail Integration Service connection ID.

### Step 4 — Deploy the Approval Action App (Stage 5)

```bash
cd classification-approval-app
npm install
npm run build

# Pack and publish as an Action App
uip codedapp pack dist -n classification-approval-app --version 0.0.3
uip codedapp publish -t Action \
  --name classification-approval-app \
  --version 0.0.3

# Deploy to your Case's folder (replace with your folder key)
uip codedapp deploy \
  -n classification-approval-app \
  --folder-key <your-case-folder-key>
```

### Step 5 — Configure and Publish the Maestro Case

1. Open `MaestroCase/_unpacked/content/caseplan.json` in UiPath Studio Web
2. Rebind each stage node to point at **your deployed process keys**:
   - Stage 1: `ReimbursementIntakeBot_XP`
   - Stage 2: `ReceiptExtractor_XP`
   - Stage 3: `ReimbursementApiSolution.Agent.ReimbursementClassificationAgent`
   - Stage 4: `ReimbursementApiSolution.Api.PolicyRuleCheckWorkflow`
   - Stage 5 (HITL): `classification-approval-app` (your deployment)
   - Stage 6: `ReimbursementApiSolution.Api.StripePayoutWorkflow`
   - Stage 7: `ReimbursementApiSolution.Agent.NotificationAgent`
   - Stage 7b: `ReimbursementApiSolution.Agent.RejectionNotificationAgent`
3. Set your Stripe secret key as an Orchestrator Credential Asset named `STRIPE_SECRET_KEY`
4. **Publish the Case** from Studio Web

### Step 6 — Run the Web Intake Form (Optional)

The web form at `IntakeForm/` is an alternative to email intake, deployed on Render.

```bash
cd IntakeForm

# Install Python dependencies
pip install -r requirements.txt

# Set environment variables
export UIPATH_ACCESS_TOKEN=<your-pat-token>   # from UiPath Cloud → Profile → PAT
export RESEND_API_KEY=<your-resend-key>        # from resend.com (free tier)

# Run the API backend
uvicorn api:app --reload --port 8000

# In a separate terminal — run the frontend
npm install
npm run dev
# Opens at http://localhost:5174
```

> **Getting a UiPath PAT:** Cloud Portal → Profile icon → Personal Access Tokens → Create → Scope: Orchestrator → Max expiry. Format is `pat_…` (not `rt_…`).

### Step 7 — Test the Full Pipeline

```bash
# Trigger the Maestro Case via CLI
uip maestro case process run \
  ReimbursementProcessMaestro.caseManagement.ReimbursementProcessCase \
  <your-case-folder-key> \
  --release-key <your-release-key> \
  -i '{"employeeEmail":"you@example.com","expenseAmount":248.50,"expenseCurrency":"USD","expenseTypeConfirmed":"Travel","expenseVendor":"Acme Hotels","expenseDate":"2026-06-29","expenseReason":"Client meeting","documentAttached":true,"ocrConfidence":0.95,"duplicateDetected":false,"businessPurposeValid":true}'
```

Watch in **Orchestrator → Jobs** as each stage runs in sequence. When it reaches Stage 5, open **Action Center** in your browser to Approve or Reject.

---

## Validated Results

| Stage | Status | Evidence |
|---|---|---|
| Stage 1 — Email Intake | ✅ Successful | Serverless job ran, receipt uploaded to bucket |
| Stage 2 — IDP Extraction | ✅ Successful | `out_JSON` returned with vendor/amount/date |
| Stage 3 — Classification | ✅ Successful | Smoke + edge evals 100%; cloud job Successful |
| Stage 4 — Policy Check | ✅ Successful | 5 routing scenarios correct |
| Stage 5 — HITL Approval | ✅ Deployed | Action App v0.0.3 live in Action Center |
| Stage 6 — Stripe Payout | ✅ Successful | Real PaymentIntent `succeeded` in test mode |
| Stage 7 — Notify Email | ✅ Successful | HTML email delivered, Gmail message ID returned |
| Stage 7b — Reject Email | ✅ Successful | Rejection reason surfaced in claimant email |
| Full solution | ✅ Active | `ReimbursementApiSolution` v3.9.0 deployed on tenant |

---

## Technology Stack

| Category | Technologies |
|---|---|
| **Orchestration** | UiPath Maestro Case Management |
| **AI Agents** | UiPath Agent Builder, Python, LangGraph, GPT-4o via UiPath LLM Gateway |
| **RPA** | UiPath Studio (cross-platform / Portable), Document Understanding |
| **API Workflows** | UiPath API Workflows (JavaScript) |
| **Human-in-the-Loop** | UiPath Action Center, Coded Action App (React/Vite) |
| **Integration** | UiPath Integration Service, Gmail connector |
| **Payment** | Stripe API (test mode) |
| **Web Intake** | React/Vite, FastAPI, Docker, Render.com |
| **Storage** | Azure Blob Storage (via UiPath Orchestrator bucket) |
| **Deployment** | UiPath Orchestrator, Serverless Platform Units |

---

## Security

No credentials are committed to this repository. All secrets (UiPath PAT, Stripe key, Gmail App Password) are stored as:
- Orchestrator Credential Assets (for tenant-deployed components)
- Environment variables in `.env` (gitignored locally)
- Render.com environment variables (for the web intake)

Connection IDs visible in source files are tenant identifiers, not credentials — they are inert without valid authentication to that specific tenant.

---

## Team

| Name | Contribution |
|---|---|
| **Mir Jasimuddin** | Maestro Case authoring, orchestration, overall architecture |
| **Subharjun Bose** | AI Agents (Classify / Notify / Reject), API Workflows (Policy / Stripe), Web Intake Form |
| **Rashmi** | Document Understanding, RPA intake bot, IDP extraction |
| **Avishek** | Demo, presentation deck, video walkthrough |

---

*UiPath AgentHack 2026 · Track 1 · Reimbursement Process Automation*
