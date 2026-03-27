# 🏆 Final Capstone Project: Enterprise Conversational AI Platform
## Weeks 7–9 Consolidated Capstone




> 📝 **Your Company Here:** Before you write a single line of code, fill in the box below. Every deliverable will reference these decisions:
> - **Company Name**: ___________________________
> - **Industry** (e.g., Healthcare, Insurance, Retail, Telecom): ___________________________
> - **Primary Customer Channel** (Voice / Chat / Both): ___________________________
> - **Two Core Use Cases your bot will handle** (e.g., "Check claim status" and "Schedule appointment"):
>   - Use Case A: ___________________________
>   - Use Case B: ___________________________
> - **Three customer-facing service tiers** (e.g., Premium / Standard / General, or Gold / Silver / Bronze): ___________________________

---

## 🗂️ How This Links to Your Daily Labs

| Day | Lab You Built | What You'll Extend |
|---|---|---|
| **W7 D1** | GCP Project + IAM + Service Accounts | Use the same GCP project as your foundation |
| **W7 D2** | Cloud Function → Firestore integration | This **is** your webhook backend |
| **W7 D3** | IVR routing tree + skills-based routing simulation | Inform your Dialogflow CX flow architecture |
| **W7 D4** | Conversational design — intents, entities, personas, error paths | Scaffold your CX agent's NLU layer |
| **W7 D5** | End-to-end architecture simulation | Validate that all layers connect |
| **W8 D1** | Agent Desktop walkthrough + screen pop + warm transfer | Documented as your escalation design |
| **W8 D2** | Skills-based routing matrix + escalation rules | Embedded into your CX escalation flow |
| **W8 D3** | Queue overflow strategy + EWT + callbacks | Reflected in your fallback route design |
| **W8 D4** | KPI analysis — AHT, SLA, Abandon Rate | Final submission includes analytics evidence |
| **W9 D1** | Dialogflow CX Agent + Flows + Pages + Routes | **Your agent hub** |
| **W9 D2** | Custom entities + intents + multi-turn form filling | Applied across all your agent's intents |
| **W9 D3** | Static responses + custom payloads + conditional responses | UI layer of your agent |
| **W9 D4** | Authentication flow + session params + conditional routing by tier | Required vertical in your agent |

---

## 🏗️ Project Architecture

Your platform must implement the following end-to-end architecture. Every layer maps to labs you have already completed.

```
Customer (Voice / Chat / Web)
  ↓
Channel Layer (PSTN → SIP Trunk / WebRTC)
  ↓
Dialogflow CX Agent (NLU, Flows, Pages, Routes, Event Handlers)
  ↓
Webhook Layer (Cloud Functions Gen 2 — Business Logic)
  ↓
Data Layer (Firestore — Customer Accounts, Session State)
  ↓
Observability (Cloud Logging — Structured JSON Logs)
  ↓
Security (IAM + Least-Privilege Service Accounts)
  ↓
Human Escalation (Agent Desktop → Warm Transfer → ACW)
```

---

## 📦 Deliverable 1: GCP Infrastructure (W7 D1 + D2)

**The rule**: Your entire project must run on the GCP project and Service Account infrastructure you set up in Week 7.

> 📝 **Your Company Here:**
> - **GCP Project ID** (from your W7 D1 lab): ___________________________
> - **Cloud Function name** you will deploy: ___________________________
> - **Firestore collection names** for your domain (e.g., `customers`, `claims`, `orders`): ___________________________

### Tasks

**1.1 — Verify Your Foundation (from W7 D1)**
- [X] Confirm your GCP project is active with billing linked.
- [X] Confirm the `dialogflow.googleapis.com`, `cloudfunctions.googleapis.com`, and `firestore.googleapis.com` APIs are enabled.
- [X] Confirm your `webhook-sa` Service Account exists with **only** `roles/datastore.user`. Do not over-provision.

**1.2 — Evolve Your Webhook (from W7 D2)**
Your webhook must now handle **at least three distinct intents**, not just a greeting. Adapt your `main.py` Cloud Function:

```python
import functions_framework
from flask import jsonify
from google.cloud import firestore
import json

db = firestore.Client()

@functions_framework.http
def agent_webhook(request):
    req = request.get_json(silent=True)
    tag = req.get("fulfillmentInfo", {}).get("tag", "default")

    if tag == "lookup_account":
        return handle_account_lookup(req)
    elif tag == "submit_request":
        return handle_request_submission(req)
    elif tag == "check_status":
        return handle_status_check(req)
    else:
        return jsonify({"fulfillmentResponse": {
            "messages": [{"text": {"text": ["I'm not sure how to handle that."]}}]
        }})
```

> 📝 **Your Company Here:** Replace the three webhook tags (`lookup_account`, `submit_request`, `check_status`) with tags that match your domain. Examples for a healthcare company might be: `lookup_patient`, `submit_appointment`, `check_claim_status`. Write your three tags here:
> - Tag 1: ___________________________
> - Tag 2: ___________________________
> - Tag 3: ___________________________

Each handler reads from and writes to **your own Firestore collection(s)** that you named in Deliverable 1.

**1.3 — Structured Logging**
Every webhook invocation must emit a structured JSON log entry, as you practiced in W7 D2:
```python
print(json.dumps({
    "severity": "INFO",
    "event": "webhook_called",
    "tag": tag,
    "session_id": req.get("sessionInfo", {}).get("session", "unknown")
}))
```

**Submission Requirement**: A screenshot of your Cloud Logging panel showing at least 3 structured log entries from live test runs.

---

## 📦 Deliverable 2: Dialogflow CX Agent Architecture (W9 D1 + D4)

Your Dialogflow CX Agent must implement a **multi-flow architecture**. A single-flow agent will **fail this requirement**.

### Agent Requirements

**2.1 — Minimum Flow Structure**

> 📝 **Your Company Here:** Replace the placeholder flow names in the table below with names specific to your use cases (e.g., `Claims Flow`, `Appointment Booking Flow`, `Order Tracking Flow`). Fill in this table as your **design document** before building in Dialogflow CX:

| Flow | Purpose | Your Flow Name | Required Pages |
|---|---|---|---|
| `Default Start Flow` | Entry point + intent routing | *(keep as-is)* | Start (Greeting + Menu) |
| `Authentication Flow` | Collect + verify account | *(keep as-is)* | Collect Account, Verify, Tier Router |
| `Primary Service Flow A` | Your core use case #1 | ___________________________ | At minimum 2 pages |
| `Primary Service Flow B` | Your core use case #2 | ___________________________ | At minimum 2 pages |
| `Escalation Flow` | Human handoff | *(keep as-is)* | Confirm Escalation, Handoff |

**2.2 — Authentication Flow (Mandatory, from W9 D4)**
This must be the **first** flow all users enter. Re-use your W9 D4 lab directly:

> 📝 **Your Company Here:** Customize the entry greeting and the tier names to match your company and the tier names you defined in the Overview:
> - **Bot greeting text**: ___________________________
> - **Tier 1 name** (top tier, e.g., "premium", "gold", "vip"): ___________________________
> - **Tier 2 name** (mid tier, e.g., "standard", "silver"): ___________________________
> - **Tier 3 name** (default, e.g., "general", "basic"): ___________________________
> - **Webhook tag for account lookup**: ___________________________

```
Flow: Authentication Flow
Page: Collect Account
  Entry: "[YOUR GREETING HERE — e.g., 'Welcome to [Your Company]. To get started, I'll need to verify your account.']"
  Form:
    account_number (@sys.number) — Required
      Reprompt (1st): "I didn't catch that. Please say or type your account number."
      Reprompt (2nd): "I'm having trouble. Let me connect you to an agent." → Escalate
  Route: $page.params.status = "FINAL"
    → Transition: Verify Page (calls webhook tag: "[YOUR TAG HERE]")

Page: Tier Router
  Route 1: $session.params.loyalty_tier = "[YOUR TIER 1 NAME]"
    → Transition: [Tier 1 Entry] (in your Primary Service Flow A)
  Route 2: $session.params.loyalty_tier = "[YOUR TIER 2 NAME]"
    → Transition: [Tier 2 Entry]
  Route 3: true (default)
    → Transition: [General Entry]
```

**2.3 — Route Groups (Mandatory)**
Create a Route Group named `Global Handlers` and attach it to **every flow**:

```
Route Group: Global Handlers
  Route 1: Intent = escalate_to_human
    → Transition: Escalation Flow
  Route 2: Intent = start_over
    → Param Preset: Clear all session params ($session.params = null)
    → Transition: Default Start Flow
  Route 3: sys.no-match-3
    → "I'm having trouble understanding. Let me get a specialist for you."
    → Transition: Escalation Flow
```

---

## 📦 Deliverable 3: Advanced NLU Layer (W7 D4 + W9 D2)

You must have **at least 8 intents** across your agent, each with a minimum of **15 training phrases**. Poor NLU coverage is the #1 cause of bot failure.

### Intent Requirements

> 📝 **Your Company Here:** The first five intents below use generic names. Rename them to match your domain and add phrasing that your actual customers would use. For example, a telecom company might rename `start_new_request` to `report_outage` and `check_status` to `check_ticket_status`. Fill in your intent names:
> - `start_new_request` → your name: ___________________________
> - `check_status` → your name: ___________________________
> - `modify_request` → your name: ___________________________
> - `cancel_request` → your name: ___________________________
> - `billing_inquiry` → your name (or replace with a domain-relevant intent): ___________________________

| Intent | Min Phrases | Must Use Entity |
|---|---|---|
| `start_new_request` *(rename for your domain)* | 15 | — |
| `check_status` *(rename for your domain)* | 15 | `@sys.number` or custom ID entity |
| `modify_request` *(rename for your domain)* | 15 | Custom entity (at least 3 synonyms each value) |
| `cancel_request` *(rename for your domain)* | 15 | — |
| `billing_inquiry` *(rename or replace)* | 15 | — |
| `escalate_to_human` | 20 | — |
| `start_over` | 10 | — |
| `confirm_yes` | 10 | — |
| `confirm_no` | 10 | — |

### Entity Requirements (from W9 D2)

You must define **at least 2 custom entities** relevant to your chosen domain:

> 📝 **Your Company Here:** The example entities below are generic. You must replace the **entity names** and **values** with ones specific to your industry. Examples:
> - Telecom: `@issue-type` (outage, billing, device, account), `@service-type` (mobile, broadband, TV)
> - Healthcare: `@appointment-type` (consultation, follow-up, lab), `@specialty` (cardiology, general, pediatrics)
> - Insurance: `@claim-type` (auto, health, home), `@policy-status` (active, expired, pending)
>
> Fill in your two custom entity designs here before building in Dialogflow CX:
> - **Entity 1 name**: ___________________________ Values + synonyms: ___________________________
> - **Entity 2 name**: ___________________________ Values + synonyms: ___________________________

```text
Entity: @[YOUR-ENTITY-1] (Map kind)              ← replace with your domain-specific name
  "value_1" → ["synonym", "synonym", "synonym"]  ← replace with your actual values
  "value_2" → ["synonym", "synonym", "synonym"]
  "value_3" → ["synonym", "synonym", "synonym"]

Entity: @[YOUR-ENTITY-2] (Map kind)              ← replace with your domain-specific name
  "urgent" → ["emergency", "critical", "ASAP", "blocking"]
  "normal" → ["whenever", "no rush", "standard", "regular"]
  "low" → ["low priority", "minor", "when you have time"]
```

### Multi-Turn Form (Mandatory, from W9 D2)

Your primary service flow must include a **3-parameter form**:

> 📝 **Your Company Here:** Replace the form parameters with the right entities for your domain. The prompts must use natural language your customers would hear from a bot in your industry. Fill in:
> - **Page name**: ___________________________
> - **Parameter 1** (entity + prompt question): ___________________________
> - **Parameter 2** (entity + prompt question): ___________________________
> - **Parameter 3** (entity + prompt question): ___________________________
> - **Webhook tag for submission**: ___________________________

```
Page: [YOUR PAGE NAME — e.g., "Collect Request Details", "Book Appointment", "Report Outage"]
  Form Parameters:
    [param_1] (@[YOUR-ENTITY-1]) — Required
      Prompt: "[YOUR DOMAIN-SPECIFIC PROMPT — e.g., 'What type of issue are you experiencing?']"
    [param_2] (@[YOUR-ENTITY-2 or @priority-level]) — Required
      Prompt: "[YOUR DOMAIN-SPECIFIC PROMPT — e.g., 'How urgent is this?']"
    description (@sys.any) — Required
      Prompt: "[YOUR DOMAIN-SPECIFIC PROMPT — e.g., 'Briefly describe the problem.']"
  Route: $page.params.status = "FINAL"
    → Webhook call (tag: "[YOUR WEBHOOK TAG]")
    → Transition: Confirmation Page
```

---

## 📦 Deliverable 4: Fulfillment & Response Layer (W9 D3)

Every page must have a **purposeful response strategy**. A bare text message is not sufficient.

### Response Requirements

**4.1 — Rich Responses with Custom Payloads**
At least **two pages** must send suggestion chip payloads:

> 📝 **Your Company Here:** Replace the chip labels with your actual service options. The chips on your Start page should reflect your two core use cases. Examples for an insurance company: `"File a Claim"`, `"Check Claim Status"`, `"Update Policy"`, `"Speak to Agent"`.
> - Chip 1: ___________________________
> - Chip 2: ___________________________
> - Chip 3: ___________________________
> - Chip 4 (optional): ___________________________

```json
{
  "richContent": [[
    {
      "type": "chips",
      "options": [
        { "text": "[YOUR OPTION 1]" },
        { "text": "[YOUR OPTION 2]" },
        { "text": "[YOUR OPTION 3]" },
        { "text": "Speak to Agent" }
      ]
    }
  ]]
}
```

**4.2 — Conditional Responses (Mandatory)**
Your confirmation page must vary its message based on session parameters:

> 📝 **Your Company Here:** Rewrite the confirmation messages in your company's actual tone and brand voice. Replace the SLA time estimates ("1 hour", "24 hours") with your real or realistic service commitments. Also replace `ref_id` with whatever you name the reference field in Firestore.
> - Urgent confirmation message: ___________________________
> - Normal confirmation message: ___________________________
> - Default confirmation message: ___________________________

```
IF $session.params.priority_level = "[YOUR URGENT TIER NAME]":
  "[YOUR URGENT CONFIRMATION — e.g., '⚠️ Your urgent request has been logged. Reference: $session.params.ref_id. You'll hear back within 1 hour.']"

IF $session.params.priority_level = "[YOUR NORMAL TIER NAME]":
  "[YOUR NORMAL CONFIRMATION — e.g., '✅ Request logged. Reference: $session.params.ref_id. Estimated response: 24 hours.']"

DEFAULT:
  "[YOUR DEFAULT CONFIRMATION — e.g., 'Your request has been submitted. Reference: $session.params.ref_id.']"
```

**4.3 — Webhook Error Handling**
Configure a `webhook.error` event handler on every page that calls a webhook:

```
Event: webhook.error
  Param Preset: $session.params.webhook_failed = true
  Response: "I'm having trouble reaching our system right now. Your information is safe. [Option to retry or speak to agent.]
```

---

## 📦 Deliverable 5: Session & State Management (W9 D4)

Demonstrate mastery of the three parameter scopes. Incorrect scope usage means your agent will **fail at scale**.

### State Management Requirements

**5.1 — Session-Level Tracking**
These parameters must persist for the **entire session**:
- `account_number`, `customer_name`, `loyalty_tier`, `authenticated`
- `request_ref_id` (generated by webhook on submission)
- `attempt_counter` (used for escalation after 3 failures)

**5.2 — Counter Pattern (Mandatory)**
Implement a retry counter for your form to prevent infinite loops:

```
Page: Collect Request Details (no-match handler):
  Param Preset: $session.params.attempt_counter = $session.params.attempt_counter + 1

Route Group: Global Handlers
  Route: $session.params.attempt_counter >= 3
    → Response: "I'm struggling to understand. Connecting you to a specialist."
    → Param Preset: $session.params.attempt_counter = 0
    → Transition: Escalation Flow
```

**5.3 — Cross-Flow Parameter Evidence**
Your submission must include a **simulator screenshot** proving that a parameter set in Flow A (e.g., `customer_name` from Authentication Flow) is still available and displayed correctly on a page in Flow B.

---

## 📦 Deliverable 6: Routing, Escalation & Contact Center Logic (W7 D3 + W8 D1/D2/D3)

This deliverable synthesizes your entire Week 7 D3, Week 8 D1, D2, and D3 labs into a documented design.

### Routing Design Document

You must submit a **routing design document** (markdown or table) covering the following:

**6.1 — Skills Matrix (from W8 D2)**
Define the human agent team that will handle escalations. You need at least 4 agents:

> 📝 **Your Company Here:** Replace the specializations, tier labels, and queue names with your company's actual (or realistic) human agent structure. What skills would your agents need? What languages would your customer base require? Replace "General", "Technical", "Billing" with your domain terms (e.g., for insurance: "Claims Processing", "Underwriting", "Fraud").

| Agent ID | Specialization | Tier | Languages | Priority Queues |
|---|---|---|---|------|
| Agent-1 | [YOUR SPECIALIZATION] | 1 | [Language] | [your queue name] |
| Agent-2 | [YOUR SPECIALIZATION] | 2 | [Languages] | [your queue names] |
| Agent-3 | [YOUR SPECIALIZATION] | 2 | [Language] | [your queue names] |
| Agent-4 | [YOUR SPECIALIZATION] | 1 | [Language] | [your queue name] |

**6.2 — Queue Strategy (from W8 D3)**
Define your queue overflow logic:

> 📝 **Your Company Here:** Name your queues based on your company's service structure (e.g., `Claims-Urgent`, `Billing-Standard`). Adjust SLA targets and wait times to reflect realistic expectations for your industry. Healthcare may need stricter SLAs than retail. Write your queue names and SLA targets here:
> - Queue 1 name: ___________________________ SLA target: ___________________________
> - Queue 2 name: ___________________________ SLA target: ___________________________

```
Queue: [YOUR URGENT QUEUE NAME]
  SLA Target: [e.g., 90/10]
  Max Wait: [e.g., 2 min]
  Overflow 1 (60s): Relax skill requirements to Tier 1
  Overflow 2 (120s): Offer callback
  Overflow 3 (180s): Voicemail + ticket created in Firestore ([your collection name])

Queue: [YOUR STANDARD QUEUE NAME]
  SLA Target: [e.g., 80/20]
  Max Wait: [e.g., 4 min]
  Overflow 1 (120s): General agents
  Overflow 2 (240s): Callback offer
```

**6.3 — Escalation Flow in CX**
Your Dialogflow CX `Escalation Flow` must:
1. Confirm the customer wants to speak with a human.
2. Set a session parameter `$session.params.escalated = true`.
3. Pass a **custom payload** with the conversation context (account number, issue type, tier) to the agent desktop via the webhook.

---

## 📦 Deliverable 7: Analytics & Validation (W8 D4 + W9 D1)

No agent ships without evidence. Your submission must include proof of testing.

### 7.1 — Simulator Test Log

Run and document the following **minimum test scenarios** in the CX Simulator:

> 📝 **Your Company Here:** Columns **Input** and **Expected Outcome** must be filled in with your company-specific values before testing. Replace the generic inputs (e.g., "Account + urgent request") with realistic utterances a customer of your company would say, and the expected outcomes with the actual page/response names in your agent.

| Test ID | Scenario | Your Input (fill in) | Your Expected Outcome (fill in) |
|---|---|---|---|
| T-01 | Happy path (top-tier user) | ___________________________ | Routes to [your Tier 1 flow], webhook confirms |
| T-02 | Happy path (standard user) | ___________________________ | [Your standard flow], confirmation with ref ID |
| T-03 | Form reprompt | ___________________________ | Bot reprompts with your custom reprompt text |
| T-04 | 3-strike escalation | 3 consecutive no-match inputs | Counter reaches 3, escalates to [your Escalation Flow] |
| T-05 | Start over | ___________________________ | Session params cleared, returns to Start |
| T-06 | Explicit escalation | ___________________________ | Escalation Flow triggered, context passed |
| T-07 | Webhook error | Simulate webhook timeout | Your `webhook.error` handler fires gracefully |
| T-08 | Condition branching | Set tier = "[your tier 1 name]" | Conditional response shows your Tier 1 message |

Submit a **screenshot** or **transcript** for each test.

### 7.2 — KPI Simulation (from W8 D4)

Based on a hypothetical traffic pattern, fill in this KPI table in your submission:

| KPI | Simulated Value | Target | Pass/Fail |
|---|---|---|---|
| Bot Containment Rate | _%  | ≥ 40% | |
| Escalation Rate | _% | ≤ 30% | |
| Form Completion Rate | _% | ≥ 80% | |
| Webhook Error Rate | _% | ≤ 5% | |
| 3-Strike Trigger Rate | _% | ≤ 10% | |

---

## 📋 Submission Checklist

Before submitting, confirm every item below:

### Infrastructure ✅
- [ ] GCP project active with correct APIs enabled
- [ ] `webhook-sa` service account with ONLY `roles/datastore.user`
- [ ] Cloud Function deployed and returning `200 OK`
- [ ] Firestore `customers` and `requests` collections exist with test data
- [ ] Cloud Logging shows structured JSON entries from 3+ real test runs

### Dialogflow CX Agent ✅
- [ ] Minimum 5 flows (Start, Auth, Service A, Service B, Escalation)
- [ ] Authentication Flow is entry point for all users
- [ ] Tier-based routing (premium/standard/general) is functional
- [ ] Global Route Group attached to all flows
- [ ] `attempt_counter` pattern implemented and tested

### NLU Layer ✅
- [ ] Minimum 8 intents, each with 15+ training phrases
- [ ] Minimum 2 custom entities with Map kind and synonyms
- [ ] Multi-turn 3-parameter form on at least one service page

### Response & Fulfillment ✅
- [ ] Minimum 2 pages use custom payload (suggestion chips)
- [ ] Minimum 1 conditional response that varies by session parameter
- [ ] Webhook error handler configured on every webhook-calling page
- [ ] Webhook routes on at least 3 distinct tags

### State Management ✅
- [ ] Session params persist across flow transitions (evidenced by screenshot)
- [ ] Page-scoped params used correctly (not session-scoped where inappropriate)
- [ ] Parameter cleared on "start over" intent

### Contact Center Logic ✅
- [ ] Skills matrix table with 4+ agents documented
- [ ] Queue overflow strategy documented for Urgent + Standard queues
- [ ] Escalation Flow passes structured context to webhook

### Testing & Analytics ✅
- [ ] All 8 test scenarios (T-01 to T-08) documented with screenshots/transcripts
- [ ] KPI simulation table filled in with honest estimates
- [ ] Cloud Logging screenshot showing real test log entries

---

## 📦 Deliverable 8: Generative AI Layer (Bonus Concept — Mandatory)

Every modern Dialogflow CX agent ships with a hybrid architecture: deterministic flows **plus** a generative safety net. You must implement both halves.

> 📝 **Your Company Here:** What type of documents would your company maintain for customer self-service? (e.g., FAQs, product manuals, policy documents, onboarding guides). List 2–3 documents you will add to your Data Store:
> - Document 1: ___________________________
> - Document 2: ___________________________
> - Document 3 (optional): ___________________________

### Tasks

**8.1 — Create a Data Store**
1. In your Dialogflow CX Agent, go to **Manage → Data Stores → Create**.
2. Select **Website** or **File upload** as the data source.
3. Upload at least **2 content sources** (PDFs, HTML pages, or plain text) relevant to your domain.
4. Wait for indexing to complete (usually 5–15 minutes).

**8.2 — Enable Generative Fallback**
Attach the Data Store to your `Default Start Flow` as a fallback handler:
```
Default Start Flow → Agent Responses → Generative Fallback:
  Enabled: true
  Data Store: [YOUR DATA STORE NAME]
  Prompt: "You are a helpful assistant for [YOUR COMPANY]. 
           Only answer from the provided documents. 
           If unsure, say 'I'll connect you with someone who can help.'"
```

> 📝 **Your Company Here:** Write your generative fallback prompt here (it must match your company's tone):
> ___________________________

**8.3 — Test Generative vs. Deterministic Routing**
Run test cases that prove both paths work:

| Test | Input | Expected Path | Expected Response |
|---|---|---|---|
| G-01 | A question that matches an intent | Deterministic | Goes to your flow |
| G-02 | A question answered by your Data Store | Generative | Cites the document |
| G-03 | A question outside all documents | Generative | Graceful "I don't know" |

**Submission Requirement**: Screenshots for G-01, G-02, and G-03 from the CX Simulator.

---

## 📦 Deliverable 9: Voice-Ready SSML Responses

A production contact center agent handles **voice callers**, not just chat users. Your agent must be voice-ready.

> 📝 **Your Company Here:** Which pages in your agent do customers spend the most time on? Those are the priority pages for SSML tuning. List 2 pages you will make voice-optimized:
> - Page 1: ___________________________
> - Page 2: ___________________________

### Tasks

**9.1 — Apply SSML to the Verification Page**
On your Authentication Flow's Verify page, replace the plain text response with an SSML response:
```xml
<speak>
  Account verified.
  <break time="400ms"/>
  Welcome back,
  <emphasis level="moderate">$session.params.customer_name</emphasis>.
  <break time="200ms"/>
  You are on our
  <prosody rate="slow">$session.params.loyalty_tier</prosody>
  plan.
  How can I help you today?
</speak>
```

> 📝 **Your Company Here:** Rewrite this SSML block using your company's vocabulary and tier names. Use at least one `<break>`, one `<emphasis>`, and one `<prosody>` tag:
> *(Paste your SSML here)*

**9.2 — Apply SSML to the Confirmation Page**
Your submission confirmation must also be voice-optimized:
```xml
<speak>
  Your request has been submitted.
  <break time="300ms"/>
  Your reference number is
  <say-as interpret-as="characters">$session.params.ref_id</say-as>.
  <break time="500ms"/>
  Is there anything else I can help you with?
</speak>
```

**9.3 — SSML Validation**
Test each SSML response using the **CX Simulator's voice preview** feature. Confirm it does not throw markup errors.

**Submission Requirement**: A transcript showing the rendered text of your SSML on at least 2 pages.

---

## 📦 Deliverable 10: Dynamic Entity Injection

Static entities cannot personalize at runtime. This deliverable proves you can inject user-specific entities into the conversation based on real session data.

> 📝 **Your Company Here:** What are examples of user-specific values in your domain? (e.g., past order IDs, open ticket numbers, enrolled policy IDs, appointment dates). List 2 types:
> - Dynamic value type 1: ___________________________
> - Dynamic value type 2: ___________________________

### Tasks

**10.1 — Return Dynamic Values from Your Webhook**
When your `lookup_account` webhook handler runs, it should return a session entity override alongside the account data:

```python
def handle_account_lookup(req):
    # Fetch customer data from Firestore
    account_num = req["sessionInfo"]["parameters"]["account_number"]
    doc = db.collection("[YOUR COLLECTION]").document(str(account_num)).get()
    data = doc.to_dict() if doc.exists else {}

    # Build session entity for user's specific items
    session_path = req["sessionInfo"]["session"]
    entity_overrides = [{
        "entityOverrideMode": "ENTITY_OVERRIDE_MODE_SUPPLEMENT",
        "entityType": {
            "displayName": "@[YOUR-ENTITY]",   # e.g., @ticket-id or @order-id
            "entities": [
                {"value": item_id, "synonyms": [item_id]}
                for item_id in data.get("[YOUR FIELD]", [])  # e.g., "open_tickets"
            ]
        }
    }]

    return jsonify({
        "sessionInfo": {
            "parameters": {
                "customer_name": data.get("name", "Valued Customer"),
                "loyalty_tier": data.get("tier", "general")
            }
        },
        "sessionEntityTypes": entity_overrides  # <-- dynamic injection
    })
```

> 📝 **Your Company Here:** Fill in the `[YOUR COLLECTION]`, `[YOUR-ENTITY]`, and `[YOUR FIELD]` placeholders with your Firestore collection name, entity name, and the Firestore field that holds the dynamic list.

**10.2 — Use the Injected Entity in a Form**
On a page **after** authentication, add a form parameter that uses the injected entity:
```
Form Parameter: selected_item ([YOUR-ENTITY]) — Required
  Prompt: "Which [ticket/order/policy] would you like help with?
           Your open ones are: [$session.params.[YOUR FIELD]]"
```

**10.3 — Prove It Works**
In the simulator: log in with an account that has 2 items in Firestore. Confirm the bot recognizes the exact IDs when the user says them.

**Submission Requirement**: Screenshot showing the entity override values appearing in the simulator's debug panel.

---

## 📦 Deliverable 11: Full IVR Tree Design

Dialogflow CX is not deployed in isolation — it sits on top of a telephony layer. You must design the complete IVR tree that a **voice caller** experiences before, during, and after their conversation with the bot.

> 📝 **Your Company Here:** What are the main DTMF menu options a caller hears before the bot takes over? What are your business hours? What happens after hours? Fill in:
> - Business hours: ___________________________
> - After-hours message: ___________________________
> - DTMF menu options (1, 2, 3, ...): ___________________________

### Tasks

**11.1 — Draw the Full IVR Tree**
Create a diagram (Mermaid, Lucidchart, draw.io, or hand-drawn and photographed) that includes:

```
Incoming Call
  ↓
[Business Hours Check]
  ├── YES → IVR Greeting: "Welcome to [Your Company]."
  │         ├── Press 1: [Use Case A] → CX Agent: [Flow A]
  │         ├── Press 2: [Use Case B] → CX Agent: [Flow B]
  │         ├── Press 3: Billing → CX Agent: Billing Intent
  │         ├── Press 0: Speak to Agent → Queue: [Urgent Queue]
  │         └── No Input (×2) → CX Agent: Start Flow (NLU mode)
  └── NO  → After-Hours Message + Voicemail
              → Ticket created in Firestore
```

**11.2 — SLA Annotation**
Annotate your diagram with the SLA targets and overflow thresholds from your Deliverable 6 queue design. Every queue branch must show:
- Target SLA (e.g., 80/20)
- Max wait before callback offered
- Max wait before voicemail

**11.3 — EWT Calculation**
For your highest-traffic queue, show the EWT calculation:
```
EWT = (Queue Depth × AHT) / Available Agents

Your values:
  Queue depth (peak estimate): ___
  AHT (minutes): ___
  Available agents (peak): ___
  Calculated EWT: ___ minutes
  → Action: [Offer callback / Continue / Emergency IVR]
```

**Submission Requirement**: The completed IVR tree diagram + the annotated EWT calculation.

---

## 📦 Deliverable 12: Webhook Multi-Tag Dispatcher Architecture

A production webhook is not a single function — it is a dispatch system. This deliverable forces you to engineer your backend like a real engineer.

> 📝 **Your Company Here:** List all the webhook operations your agent currently needs. You must have at least 5. Name them precisely as they will appear as tags in Dialogflow CX:
> - Tag 1: ___________________________
> - Tag 2: ___________________________
> - Tag 3: ___________________________
> - Tag 4: ___________________________
> - Tag 5: ___________________________

### Tasks

**12.1 — Implement the Dispatcher Pattern**
Refactor your `main.py` to use a clean handler registry:

```python
import functions_framework
from flask import jsonify, make_response
from google.cloud import firestore
import json, logging

db = firestore.Client()
logger = logging.getLogger(__name__)

# Handler registry — add new tags here without touching dispatch logic
HANDLER_REGISTRY = {
    "[YOUR TAG 1]": handle_tag_1,
    "[YOUR TAG 2]": handle_tag_2,
    "[YOUR TAG 3]": handle_tag_3,
    "[YOUR TAG 4]": handle_tag_4,
    "[YOUR TAG 5]": handle_tag_5,
}

@functions_framework.http
def agent_webhook(request):
    req = request.get_json(silent=True)
    tag = req.get("fulfillmentInfo", {}).get("tag", "")
    session_id = req.get("sessionInfo", {}).get("session", "unknown")

    logger.info(json.dumps({"event": "dispatch", "tag": tag, "session": session_id}))

    handler = HANDLER_REGISTRY.get(tag)
    if not handler:
        return make_response(jsonify({"error": f"Unknown tag: {tag}"}), 400)

    try:
        return handler(req)
    except Exception as e:
        logger.error(json.dumps({"event": "handler_error", "tag": tag, "error": str(e)}))
        return make_response(jsonify({"error": "Internal error"}), 500)
```

**12.2 — Each Handler Must**:
- Read at least one session parameter from `req["sessionInfo"]["parameters"]`
- Perform a Firestore read **or** write
- Return a properly structured `fulfillmentResponse` JSON
- Emit one structured Cloud Logging entry

**12.3 — Document Every Handler**
Submit a table documenting each handler:

| Tag | Firestore Operation | Session Params Read | Session Params Written | Cloud Log Event |
|---|---|---|---|---|
| [Tag 1] | read | [...] | [...] | [...] |
| [Tag 2] | write | [...] | [...] | [...] |
| ... | ... | ... | ... | ... |

**Submission Requirement**: Your complete `main.py` code + the handler documentation table.

---

## 📋 Submission Checklist

Before submitting, confirm every item below:

### Infrastructure ✅
- [ ] GCP project active with correct APIs enabled
- [ ] `webhook-sa` service account with ONLY `roles/datastore.user`
- [ ] Cloud Function deployed and returning `200 OK`
- [ ] Firestore collections exist with test data
- [ ] Cloud Logging shows structured JSON entries from 3+ real test runs

### Dialogflow CX Agent ✅
- [ ] Minimum 5 flows (Start, Auth, Service A, Service B, Escalation)
- [ ] Authentication Flow is entry point for all users
- [ ] Tier-based routing is functional (3 tiers)
- [ ] Global Route Group attached to all flows
- [ ] `attempt_counter` pattern implemented and tested

### NLU Layer ✅
- [ ] Minimum 8 intents, each with 15+ training phrases
- [ ] Minimum 2 custom entities with Map kind and synonyms
- [ ] Multi-turn 3-parameter form on at least one service page

### Response & Fulfillment ✅
- [ ] Minimum 2 pages use custom payload (suggestion chips)
- [ ] Minimum 1 conditional response that varies by session parameter
- [ ] Webhook error handler configured on every webhook-calling page
- [ ] Webhook routes on at least 3 distinct tags

### State Management ✅
- [ ] Session params persist across flow transitions (evidenced by screenshot)
- [ ] Page-scoped params used correctly
- [ ] Parameter cleared on "start over" intent

### Contact Center Logic ✅
- [ ] Skills matrix table with 4+ agents documented
- [ ] Queue overflow strategy documented for 2+ queues
- [ ] Escalation Flow passes structured context to webhook

### Testing & Analytics ✅
- [ ] All 8 test scenarios (T-01 to T-08) documented with screenshots/transcripts
- [ ] KPI simulation table filled in with honest estimates
- [ ] Cloud Logging screenshot showing real test log entries

### Generative AI (D8) ✅
- [ ] Data Store created with 2+ content sources
- [ ] Generative Fallback enabled on Default Start Flow with company-specific prompt
- [ ] G-01, G-02, G-03 test screenshots submitted

### Voice & SSML (D9) ✅
- [ ] SSML applied to Verification page (break + emphasis + prosody)
- [ ] SSML applied to Confirmation page (say-as for reference ID)
- [ ] SSML validated (no markup errors in simulator)

### Dynamic Entities (D10) ✅
- [ ] Webhook returns `sessionEntityTypes` override on account lookup
- [ ] At least one form uses the dynamically injected entity
- [ ] Simulator screenshot shows injected entity values in debug panel

### IVR Tree (D11) ✅
- [ ] Full IVR tree diagram submitted (all branches labeled)
- [ ] SLA targets and overflow times annotated on diagram
- [ ] EWT calculation completed for peak-traffic queue

### Webhook Dispatcher (D12) ✅
- [ ] `HANDLER_REGISTRY` pattern implemented
- [ ] 5+ distinct handler functions, each with Firestore + logging
- [ ] Handler documentation table submitted

---



