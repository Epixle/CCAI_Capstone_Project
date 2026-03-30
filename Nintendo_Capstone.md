# 🏆 Final Capstone Project: Enterprise Conversational AI Platform
## Weeks 7–9 Consolidated Capstone

> 📝 **Your Company Here:** Before you write a single line of code, fill in the box below. Every deliverable will reference these decisions:
> - **Company Name**: Nintendo
> - **Industry**: Entertainment
> - **Primary Customer Channel**: Chat, Voice Secondary
> - **Two Core Use Cases your bot will handle**:
>   - Use Case A: Purchase a Product
>   - Use Case B: Get information about a product/service
> - **Three customer-facing service tiers**: 
>   - Tier 1: Standard Customers
>   - Tier 2: Switch Online Members
>   - Tier 3: Switch Online Expansion Members

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

> 📝 **Nintendo**
> - **GCP Project ID**: nintendo-ccai-prod
> - **Cloud Function name**: agent-webhook
> - **Firestore collection names**:
>   - products
>   - accounts
>   - orders
>   - movies
>   - support_tickets
>   - returns


### Tasks

**1.1 — Verify Your Foundation (from W7 D1)**
- [X] Confirm your GCP project is active with billing linked.
```bash
gcloud projects create nintendo-ccai-prod \
    --name="Nintendo CCAI Production"

gcloud billing projects link nintendo-ccai-prod \
    --billing-account=01ABCD-234EFG-567HIJ
```

- [X] Confirm the `dialogflow.googleapis.com`, `cloudfunctions.googleapis.com`, and `firestore.googleapis.com` APIs are enabled.
```bash
gcloud services enable \
    dialogflow.googleapis.com \
    cloudfunctions.googleapis.com \
    firestore.googleapis.com \
    logging.googleapis.com \
    monitoring.googleapis.com \
    --project=nintendo-ccai-prod
```

- [X] Confirm your `webhook-sa` Service Account exists with **only** `roles/datastore.user`. Do not over-provision.
```bash
gcloud iam service-accounts create webhook-sa \
  --display-name="Webhooks Service Account" \
  --project=nintendo-ccai-prod

gcloud projects add-iam-policy-binding nintendo-ccai-prod \ 
	--member="serviceAccount:webhook-sa@nintendo-ccai-prod.iam.gserviceaccount.com" \
	--role="roles/datastore.user"
```

**1.2 — Evolve Your Webhook (from W7 D2)**
```python
import functions_framework
from flask import jsonify
from google.cloud import firestore
import json

db = firestore.Client()

def text_response(msg):
    return jsonify({
        "fulfillmentResponse": {
            "messages": [{"text": {"text": [msg]}}]
        }
    })

def get_params(req):
    return req.get("sessionInfo", {}).get("parameters", {})

@functions_framework.http
def agent_webhook(request):
    req = request.get_json(silent=True) or {}
    tag = req.get("fulfillmentInfo", {}).get("tag", "unknown")

    # Structured logging
    print(json.dumps({
        "severity": "INFO",
        "event": "webhook_called",
        "tag": tag,
        "session_id": req.get("sessionInfo", {}).get("session", "unknown")
    }))

    if tag == "account_lookup":
        return handle_account_lookup(req)
    elif tag == "product_lookup":
        return handle_product_lookup(req)
    elif tag == "order_status":
        return handle_order_status(req)
    elif tag == "movie_info":
        return handle_movie_info(req)
    elif tag == "report_issue":
        return handle_report_issue(req)
    elif tag == "return_request":
        return handle_return_request(req)

    return text_response("I'm not sure how to handle that.")

def handle_product_lookup(req):
    params = get_params(req)
    game = params.get("game")

    doc = db.collection("products").where("name", "==", game).limit(1).get()

    if not doc:
        return text_response("I couldn't find that product.")

    data = doc[0].to_dict()

    return text_response(f"{data['name']} is available for ${data['price']}.")

def handle_order_status(req):
    params = get_params(req)
    order_id = params.get("order_id")

    doc = db.collection("orders").document(str(order_id)).get()

    if not doc.exists:
        return text_response("I couldn't find that order.")

    status = doc.to_dict().get("status", "Processing")

    return text_response(f"Your order is currently {status}.")

def handle_report_issue(req):
    params = get_params(req)

    ticket = {
        "issue_type": params.get("issue_type"),
        "game": params.get("game"),
        "console": params.get("console"),
        "status": "OPEN"
    }

    db.collection("support_tickets").add(ticket)

    return text_response("Your issue has been submitted.")

def handle_return_request(req):
    params = get_params(req)

    db.collection("returns").add({
        "order_id": params.get("order_id"),
        "reason": params.get("reason"),
        "status": "PENDING"
    })

    return text_response("Your return request has been submitted.")
```

> 📝 **Nintendo** Webhook Tags:
> - Tag 1: product_lookup
> - Tag 2: order_status
> - Tag 3: movie_info
> - Tag 4: account_lookup
> - Tag 5: report_issue
> - Tag 6: return_request

Each handler reads from and writes to **your own Firestore collections** that you named in Deliverable 1.

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

| `Default Start Flow` | Entry point + intent routing | *(keep as-is)* | Start (Greeting + Menu) |
| `Authentication Flow` | Collect + verify account | *(keep as-is)* | Collect Account, Verify, Tier Router |
| `Primary Service Flow A` | Purchase Nintendo Product | Purchase Product Flow | Product Entry, View Product |
| `Primary Service Flow B` | Track Order Delivery | Order Tracking Flow | Collect Order ID, Tracking Info |
| `Escalation Flow` | Human handoff | *(keep as-is)* | Confirm Escalation, Handoff |

**2.2 — Authentication Flow (Mandatory, from W9 D4)**
This must be the **first** flow all users enter. Re-use your W9 D4 lab directly:

> 📝 **Nintendo:** Customize the entry greeting and the tier names to match your company and the tier names you defined in the Overview:
> - **Bot greeting text**: Greetings, I am R.O.B. *beep boop*! I can help with anything related to Nintendo.
> - **Tier 1**: Switch Online Expansion Members
> - **Tier 2**: Switch Online Members
> - **Tier 3**: Standard Customers
> - **Webhook tag for account lookup**: account_lookup

```
Flow: Authentication Flow
Page: Access Account
  Entry: "In order to access your account, you'll need to log in *buzz*! [Link/button/card to access Nintendo login portal]"
  Form:
    account_number (@sys.number) — Required
      Reprompt (1st): ""I'm sorry, I didn't quite understand that *beep*. Could you try again? Make sure the account number is a 12 digit number."
      Reprompt (2nd): "I'm having trouble. Let me connect you to an agent." → Escalate
  Route: $page.params.status = "FINAL"
    → Transition: Verify Page (calls webhook tag: "account_lookup")

Page: Tier Router
  Route 1: $session.params.loyalty_tier = "Switch Online Expansion Members"
    → Transition: [Tier 1 Entry] (in your Primary Service Flow A)
  Route 2: $session.params.loyalty_tier = "Switch Online Members"
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

> 📝 **Nintendo:** The first five intents below use generic names. Rename them to match your domain and add phrasing that your actual customers would use. For example, a telecom company might rename `start_new_request` to `report_outage` and `check_status` to `check_ticket_status`. Fill in your intent names:
> - `start_new_request` → your name: `open_shop`
> - `check_status` → your name: `order_status`
> - `modify_request` → your name: `change_acc_info`
> - `cancel_request` → your name: `return_request`
> - `billing_inquiry` → your name: `show_movie_details`

| Intent | Min Phrases | Must Use Entity |
|---|---|---|
| `open_shop` | 15 | `@game`, `@console`, `@accessory`, or `@franchise` |
| `order_status` | 15 | `@order_id` |
| `change_acc_info` | 15 | `@account` |
| `return_request` | 15 | `@order_id` |
| `show_movie_details` | 15 | `@movie` |
| `escalate_to_human` | 20 | — |
| `start_over` | 10 | — |
| `confirm_yes` | 10 | — |
| `confirm_no` | 10 | — |

### Entity Requirements (from W9 D2)

You must define **at least 2 custom entities** relevant to your chosen domain:

> 📝 **Nintendo:** The example entities below are generic. You must replace the **entity names** and **values** with ones specific to your industry. Examples:
> - Telecom: `@issue-type` (outage, billing, device, account), `@service-type` (mobile, broadband, TV)
> - Healthcare: `@appointment-type` (consultation, follow-up, lab), `@specialty` (cardiology, general, pediatrics)
> - Insurance: `@claim-type` (auto, health, home), `@policy-status` (active, expired, pending)
>
> Fill in your two custom entity designs here before building in Dialogflow CX:
> - **Entity 1 name**: `@console`
> - Values + synonyms:
Switch → ["switch", "nintendo switch", "original switch"]
Switch 2 → ["switch 2", "nintendo switch 2", "new switch"]
DS → ["ds", "nintendo ds"]
DSi → ["dsi"]
DSi XL → ["dsi xl", "large dsi"]
GameCube → ["gamecube", "gc"]
N64 → ["n64", "nintendo 64"]
NES Console → ["nes", "nintendo entertainment system"]
Virtual Boy → ["virtual boy"]

> - **Entity 2 name**: `@franchise`
> - Values + synonyms:
Pokémon → ["pokemon", "pokémon"]
Mario → ["mario", "super mario"]
Legend of Zelda → ["zelda", "legend of zelda"]
Kirby → ["kirby"]
Nintendo Sports → ["sports", "nintendo sports", "switch sports"]
Super Smash → ["smash", "super smash", "smash bros", "melee", "smash ultimate"]


### Multi-Turn Form (Mandatory, from W9 D2)

Your primary service flow must include a **3-parameter form**:

> 📝 **Your Company Here:** Replace the form parameters with the right entities for your domain. The prompts must use natural language your customers would hear from a bot in your industry. Fill in:
> - **Page name**: Collect Issue Details
> - **Parameter 1** (entity + prompt question): `@console` — “Which Nintendo console are you using?”
> - **Parameter 2** (entity + prompt question): `@game`, `@franchise` — “Which game is this related to?”
> - **Parameter 3** (entity + prompt question): `@sys.any` — “Please briefly describe the issue.”
> - **Webhook tag for submission**: `report_issue`

---

## 📦 Deliverable 4: Fulfillment & Response Layer (W9 D3)

Every page must have a **purposeful response strategy**. A bare text message is not sufficient.

### Response Requirements

**4.1 — Rich Responses with Custom Payloads**
At least **two pages** must send suggestion chip payloads:

> 📝 **Nintendo:** Replace the chip labels with your actual service options. The chips on your Start page should reflect your two core use cases. Examples for an insurance company: `"File a Claim"`, `"Check Claim Status"`, `"Update Policy"`, `"Speak to Agent"`.
> - Chip 1: Shop Products
> - Chip 2: Track Order
> - Chip 3: Movie Info
> - Chip 4: Speak to Agent

> - Page 1: Start Page

```json
{
  "richContent": [[
    {
      "type": "chips",
      "options": [
        { "text": "Shop Products" },
        { "text": "Track Order" },
        { "text": "Movie Info" },
        { "text": "Speak to Agent" }
      ]
    }
  ]]
}
```

> - Page 2: Post Resolution Page

```json
{
  "richContent": [[
    {
      "type": "chips",
      "options": [
        { "text": "Check Another Product" },
        { "text": "Track Another Order" },
        { "text": "Account Help" },
        { "text": "Goodbye" }
      ]
    }
  ]]
}
```

**4.2 — Conditional Responses (Mandatory)**
Your confirmation page must vary its message based on session parameters:

> 📝 **Nintendo:** Rewrite the confirmation messages in your company's actual tone and brand voice. Replace the SLA time estimates ("1 hour", "24 hours") with your real or realistic service commitments. Also replace `ref_id` with whatever you name the reference field in Firestore.
> - Urgent confirmation message: ⚠️ Your issue has been logged with priority handling. Reference: $session.params.request_ref_id. A Nintendo specialist should review it as soon as possible.
> - Normal confirmation message: ✅ Your request has been submitted. Reference: $session.params.request_ref_id. We’ll review it shortly.
> - Default confirmation message: Your request has been submitted successfully. Reference: $session.params.request_ref_id.

```
IF $session.params.priority_level = "urgent":
  "⚠️ Your issue has been logged with priority handling. Reference: $session.params.request_ref_id. A Nintendo specialist should review it as soon as possible."

IF $session.params.priority_level = "normal":
  "✅ Your request has been submitted. Reference: $session.params.request_ref_id. We’ll review it shortly."

DEFAULT:
  "Your request has been submitted successfully. Reference: $session.params.request_ref_id."
```

**4.3 — Webhook Error Handling**
Configure a `webhook.error` event handler on every page that calls a webhook:

```
Event: webhook.error
  Param Preset: $session.params.webhook_failed = true
  Response: "beep boop My hardware is circuiting! Please try again or connect to an agent for more help."
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
    → Response: "*beep boop* Sorry, I couldn't understand. Let me connect you to a specialist."
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
| Agent-1 | Technical Escalations | 1 | ES | Issues, VIP, System Failure |
| Agent-2 | Billing / Orders | 2 | FR, PT | Purchase, Info, Returns |
| Agent-3 | Billing Escalations / VIP | 1 | KL | VIP, Adv Billing, General |
| Agent-4 | General Support / Overflow | 2 | HI | Purchase, Info, General Overflow |

**6.2 — Queue Strategy (from W8 D3)**
Define your queue overflow logic:

> 📝 **Nintendo:** Name your queues based on your company's service structure (e.g., `Claims-Urgent`, `Billing-Standard`). Adjust SLA targets and wait times to reflect realistic expectations for your industry. Healthcare may need stricter SLAs than retail. Write your queue names and SLA targets here:
> - Queue 1 name: Nintendo-Issues-Urgent
> - SLA target: 90/10
> - Queue 2 name: Nintendo-Purchase-Standard
> - SLA target: 80/20

```
Queue: Nintendo-Issues-Urgent
  SLA Target: 90/10
  Max Wait: 3 min
  Overflow 1 (60s): Relax skill requirements to Tier 1
  Overflow 2 (120s): Offer callback
  Overflow 3 (180s): Voicemail + ticket created in Firestore (support_tickets)

Queue: Nintendo-Purchase-Standard
  SLA Target: 80/20
  Max Wait: 5 min
  Overflow 1 (120s): General agents
  Overflow 2 (240s): Callback offer
  Overflow 3 (300s): Voicemail
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

> 📝 **Nintendo:** Columns **Input** and **Expected Outcome** must be filled in with your company-specific values before testing. Replace the generic inputs (e.g., "Account + urgent request") with realistic utterances a customer of your company would say, and the expected outcomes with the actual page/response names in your agent.

| Test ID | Scenario | Your Input (fill in) | Your Expected Outcome (fill in) |
|---|---|---|---|
| T-01 | Happy path (top-tier user) | "I want to buy Zelda for Switch 2" | Routes to Product Purchase Flow (Expansion path), product shown via `product_lookup` |
| T-02 | Happy path (standard user) | "Track my order 123456789012" | Order Tracking Flow → webhook returns status → response shown |
| T-03 | Form reprompt | "My game is broken" → (no console given) | Bot reprompts with your custom reprompt text |
| T-04 | 3-strike escalation | 3 consecutive no-match inputs | Counter reaches 3, escalates to Escalation Flow |
| T-05 | Start over | "restart" | Session params cleared, returns to Start |
| T-06 | Explicit escalation | "I want a human" | Escalation Flow triggered, context passed |
| T-07 | Webhook error | Simulate webhook timeout | Your `webhook.error` handler fires gracefully |
| T-08 | Condition branching | priority_level = urgent | Urgent confirmation message displayed |

Submit a **screenshot** or **transcript** for each test.

### 7.2 — KPI Simulation (from W8 D4)

Based on a hypothetical traffic pattern, fill in this KPI table in your submission:

| KPI | Simulated Value | Target | Pass/Fail |
|---|---|---|---|
| Bot Containment Rate | 80%%  | ≥ 40% | PASS |
| Escalation Rate | 12% | ≤ 30% | PASS |
| Form Completion Rate | 90% | ≥ 80% | PASS |
| Webhook Error Rate | 1% | ≤ 5% | PASS |
| 3-Strike Trigger Rate | 5% | ≤ 3% | FAIL |

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

> 📝 **Nintendo:** What type of documents would your company maintain for customer self-service? (e.g., FAQs, product manuals, policy documents, onboarding guides). List 2–3 documents you will add to your Data Store:
> - Document 1: Nintendo Support FAQ
> - Document 2: Nintendo Product & Game Information
> - Document 3: Nintendo Online Membership & Subscription Guide

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
  Data Store: Nintendo Online Membership & Subscription Guide
  Prompt: "You are a helpful assistant for Nintendo. 
           Only answer from the provided documents. 
           If unsure, say '*beep boop* Sorry, my eyes are rusty. I'll connect you with someone who can help."
```

> 📝 **Nintendo:** Write your generative fallback prompt here (it must match your company's tone):
> "You are R.O.B., a helpful Nintendo assistant.

Only answer using the provided Nintendo support and product documents.
Be concise, friendly, and accurate.

If the answer is not clearly found in the documents, say:
"*beep boop* Sorry, my eyes are rusty. I'll connect you with someone who can help."

Do not make up information."

**8.3 — Test Generative vs. Deterministic Routing**
Run test cases that prove both paths work:

| Test | Input | Expected Path | Expected Response |
|---|---|---|---|
| G-01 | “I want to buy Zelda” | Deterministic | Routed to Product Purchase Flow |
| G-02 | “What is Nintendo Switch Online?” | Generative | Cites the document |
| G-03 | “What is the capital of France?” | Generative | Fallback |

**Submission Requirement**: Screenshots for G-01, G-02, and G-03 from the CX Simulator.

---

## 📦 Deliverable 9: Voice-Ready SSML Responses

A production contact center agent handles **voice callers**, not just chat users. Your agent must be voice-ready.

> 📝 **Nintendo:** Which pages in your agent do customers spend the most time on? Those are the priority pages for SSML tuning. List 2 pages you will make voice-optimized:
> - Page 1: Authenticate
> - Page 2: Issue Confirmation

### Tasks

**9.1 — Apply SSML to the Verification Page**
On your Authentication Flow's Verify page, replace the plain text response with an SSML response:
```xml
<speak>
  Account verified.
  <break time="400ms"/>
  Welcome back,
  <emphasis level="moderate">$session.params.customer_name</emphasis>.
  <break time="250ms"/>
  You are currently a
  <prosody rate="slow">$session.params.loyalty_tier</prosody>
  Nintendo member.
  <break time="250ms"/>
  How can I help you today?
</speak>
```

> 📝 **Nintendo:** Rewrite this SSML block using your company's vocabulary and tier names. Use at least one `<break>`, one `<emphasis>`, and one `<prosody>` tag:
> *(Paste your SSML here)*

**9.2 — Apply SSML to the Confirmation Page**
Your submission confirmation must also be voice-optimized:
```xml
<speak>
  Your Nintendo support request has been submitted.
  <break time="300ms"/>
  Your reference number is
  <say-as interpret-as="characters">$session.params.request_ref_id</say-as>.
  <break time="500ms"/>
  Is there anything else I can help you with today?
</speak>
```

**9.3 — SSML Validation**
Test each SSML response using the **CX Simulator's voice preview** feature. Confirm it does not throw markup errors.

**Submission Requirement**: A transcript showing the rendered text of your SSML on at least 2 pages.

---

## 📦 Deliverable 10: Dynamic Entity Injection

Static entities cannot personalize at runtime. This deliverable proves you can inject user-specific entities into the conversation based on real session data.

> 📝 **Nintendo:** What are examples of user-specific values in your domain? (e.g., past order IDs, open ticket numbers, enrolled policy IDs, appointment dates). List 2 types:
> - Dynamic value type 1: Order IDs
> - Dynamic value type 2: Support Ticket IDs

### Tasks

**10.1 — Return Dynamic Values from Your Webhook**
When your `lookup_account` webhook handler runs, it should return a session entity override alongside the account data:

```python
def handle_account_lookup(req):
    account_num = req["sessionInfo"]["parameters"]["account_number"]

    # Get account
    doc = db.collection("accounts").document(str(account_num)).get()
    data = doc.to_dict() if doc.exists else {}

    # Fetch user's orders (dynamic values)
    orders_ref = db.collection("orders").where("account_id", "==", account_num).stream()
    order_ids = [order.id for order in orders_ref]

    # Build session entity override for order IDs
    entity_overrides = [{
        "entityOverrideMode": "ENTITY_OVERRIDE_MODE_SUPPLEMENT",
        "entityType": {
            "displayName": "@order_id",
            "entities": [
                {"value": oid, "synonyms": [oid]}
                for oid in order_ids
            ]
        }
    }]

    return jsonify({
        "sessionInfo": {
            "parameters": {
                "customer_name": data.get("name", "Valued Customer"),
                "loyalty_tier": data.get("membership_tier", "Standard"),
                "order_list": order_ids
            }
        },
        "sessionEntityTypes": entity_overrides
    })
```

> 📝 **Nintendo:** Fill in the `[YOUR COLLECTION]`, `[YOUR-ENTITY]`, and `[YOUR FIELD]` placeholders with your Firestore collection name, entity name, and the Firestore field that holds the dynamic list.

**10.2 — Use the Injected Entity in a Form**
On a page **after** authentication, add a form parameter that uses the injected entity:
```
Form Parameter: selected_item (@order_id) — Required
  Prompt: "Which [ticket/order/policy] would you like help with?
           Your open ones are: [$session.params.order_list]"
```

**10.3 — Prove It Works**
In the simulator: log in with an account that has 2 items in Firestore. Confirm the bot recognizes the exact IDs when the user says them.

**Submission Requirement**: Screenshot showing the entity override values appearing in the simulator's debug panel.

---

## 📦 Deliverable 11: Full IVR Tree Design

Dialogflow CX is not deployed in isolation — it sits on top of a telephony layer. You must design the complete IVR tree that a **voice caller** experiences before, during, and after their conversation with the bot.

> 📝 **Your Company Here:** What are the main DTMF menu options a caller hears before the bot takes over? What are your business hours? What happens after hours? Fill in:
> - Business hours: Monday-Friday 8AM - 6PM, Saturday 10AM - 6PM
> - After-hours message: "Thank you for calling Nintendo. Our support team is currently closed. Please leave a voicemail after the tone, and we’ll create a support ticket for follow-up. You can also use our chat support for self-service options.”
> - DTMF menu options (1, 2, 3, ...):
```
Press 1: Shop for Nintendo products
Press 2: Track an order or delivery
Press 3: Account or billing help
Press 4: Report a game or console issue
Press 5: Returns and refunds
Press 0: Speak to an agent
No input ×2: send caller to CX Start Flow in NLU mode
```

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
  Queue depth (peak estimate): 10
  AHT (minutes): 4 min
  Available agents (peak): 5 agents
  Calculated EWT: 8 minutes
  → Action: [Offer callback / Continue / Emergency IVR]
```

**Submission Requirement**: The completed IVR tree diagram + the annotated EWT calculation.

---

## 📦 Deliverable 12: Webhook Multi-Tag Dispatcher Architecture

A production webhook is not a single function — it is a dispatch system. This deliverable forces you to engineer your backend like a real engineer.

> 📝 **Nintendo:** List all the webhook operations your agent currently needs. You must have at least 5. Name them precisely as they will appear as tags in Dialogflow CX:
> - Tag 1: account_lookup
> - Tag 2: product_lookup
> - Tag 3: order_status
> - Tag 4: movie_info
> - Tag 5: report_issue
> - Tag 6: return_request
> - Tag 7: create_handoff_context

### Tasks

**12.1 — Implement the Dispatcher Pattern**
Refactor your `main.py` to use a clean handler registry:

```python
import functions_framework
from flask import jsonify, make_response
from google.cloud import firestore
import json
import logging
from datetime import datetime

db = firestore.Client()
logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)


def log_event(severity: str, event: str, req: dict | None = None, **kwargs):
    payload = {
        "severity": severity,
        "event": event,
        "timestamp": datetime.utcnow().isoformat() + "Z",
    }

    if req:
        payload["session_id"] = req.get("sessionInfo", {}).get("session", "unknown")
        payload["tag"] = req.get("fulfillmentInfo", {}).get("tag", "unknown")

    payload.update(kwargs)
    logger.info(json.dumps(payload))


def text_response(message: str, session_params: dict | None = None, session_entities: list | None = None):
    response = {
        "fulfillmentResponse": {
            "messages": [
                {"text": {"text": [message]}}
            ]
        }
    }

    if session_params:
        response["sessionInfo"] = {"parameters": session_params}

    if session_entities:
        response["sessionEntityTypes"] = session_entities

    return jsonify(response)


def get_params(req: dict) -> dict:
    return req.get("sessionInfo", {}).get("parameters", {})


def handle_account_lookup(req: dict):
    params = get_params(req)
    account_number = str(params.get("account_number", "")).strip()

    log_event("INFO", "account_lookup_started", req, account_number=account_number)

    if not account_number:
        return text_response("I need your 12-digit Nintendo account number first.")

    doc = db.collection("accounts").document(account_number).get()
    log_event("INFO", "firestore_read", req, collection="accounts", document_id=account_number, found=doc.exists)

    if not doc.exists:
        log_event("WARNING", "account_not_found", req, account_number=account_number)
        return text_response("I couldn't find that Nintendo account.")

    data = doc.to_dict() or {}

    # Dynamic entity injection for this user's orders
    orders_ref = db.collection("orders").where("account_id", "==", account_number).stream()
    order_ids = [order.id for order in orders_ref]

    entity_overrides = [{
        "entityOverrideMode": "ENTITY_OVERRIDE_MODE_SUPPLEMENT",
        "entityType": {
            "displayName": "@order_id",
            "entities": [{"value": oid, "synonyms": [oid]} for oid in order_ids]
        }
    }]

    written_params = {
        "customer_name": data.get("name", "Valued Customer"),
        "loyalty_tier": data.get("membership_tier", "Standard"),
        "authenticated": True,
        "order_list": order_ids
    }

    log_event(
        "INFO",
        "handler_success",
        req,
        handler="account_lookup",
        written_params=list(written_params.keys())
    )

    return text_response(
        f"Account verified. Welcome back, {written_params['customer_name']}.",
        session_params=written_params,
        session_entities=entity_overrides
    )


def handle_product_lookup(req: dict):
    params = get_params(req)
    game = str(params.get("game", "")).strip()
    franchise = str(params.get("franchise", "")).strip()
    console = str(params.get("console", "")).strip()

    log_event("INFO", "product_lookup_started", req, game=game, franchise=franchise, console=console)

    if not game and not franchise:
        return text_response("Please tell me which Nintendo game, franchise, or product you want.")

    query = db.collection("products")
    if game:
        docs = query.where("name", "==", game).limit(1).get()
    else:
        docs = query.where("franchise", "==", franchise).limit(1).get()

    log_event("INFO", "firestore_read", req, collection="products", found=bool(docs))

    if not docs:
        log_event("WARNING", "product_not_found", req, game=game, franchise=franchise)
        return text_response("I couldn't find that Nintendo product. Please try a more specific product name.")

    data = docs[0].to_dict() or {}
    written_params = {
        "product_name": data.get("name", "Nintendo Product"),
        "product_url": data.get("url", ""),
        "product_price": data.get("price", "")
    }

    log_event("INFO", "handler_success", req, handler="product_lookup", product_name=written_params["product_name"])

    return text_response(
        f"I found {written_params['product_name']}. It is available for ${written_params['product_price']}.",
        session_params=written_params
    )


def handle_order_status(req: dict):
    params = get_params(req)
    order_id = str(params.get("selected_order") or params.get("order_id") or "").strip()

    log_event("INFO", "order_status_started", req, order_id=order_id)

    if not order_id:
        return text_response("Please provide your Nintendo order number.")

    doc = db.collection("orders").document(order_id).get()
    log_event("INFO", "firestore_read", req, collection="orders", document_id=order_id, found=doc.exists)

    if not doc.exists:
        log_event("WARNING", "order_not_found", req, order_id=order_id)
        return text_response("I couldn't find that order.")

    data = doc.to_dict() or {}
    status = data.get("status", "Processing")
    tracking_number = data.get("tracking_number", "")

    written_params = {
        "order_id": order_id,
        "tracking_number": tracking_number,
        "order_status_value": status
    }

    log_event("INFO", "handler_success", req, handler="order_status", order_status=status)

    return text_response(
        f"Your order {order_id} is currently {status}.",
        session_params=written_params
    )


def handle_movie_info(req: dict):
    params = get_params(req)
    franchise = str(params.get("franchise", "")).strip()

    log_event("INFO", "movie_info_started", req, franchise=franchise)

    docs = db.collection("movies").where("franchise", "==", franchise or "Mario").limit(1).get()
    log_event("INFO", "firestore_read", req, collection="movies", found=bool(docs))

    if not docs:
        log_event("WARNING", "movie_not_found", req, franchise=franchise)
        return text_response("I couldn't find movie information for that Nintendo title.")

    data = docs[0].to_dict() or {}
    title = data.get("title", "Nintendo Movie")
    release_date = data.get("release_date", "TBD")

    written_params = {
        "movie_title": title,
        "movie_release_date": release_date
    }

    log_event("INFO", "handler_success", req, handler="movie_info", movie_title=title)

    return text_response(
        f"{title} is scheduled for release on {release_date}.",
        session_params=written_params
    )


def handle_report_issue(req: dict):
    params = get_params(req)
    console = str(params.get("console", "")).strip()
    franchise = str(params.get("franchise", "")).strip()
    description = str(params.get("description", "")).strip()
    account_number = str(params.get("account_number", "")).strip()
    priority_level = str(params.get("priority_level", "normal")).strip()

    log_event("INFO", "report_issue_started", req, console=console, franchise=franchise, priority_level=priority_level)

    ticket = {
        "account_id": account_number,
        "console": console,
        "franchise": franchise,
        "description": description,
        "priority": priority_level,
        "status": "OPEN",
        "channel": "cx"
    }

    ref = db.collection("support_tickets").add(ticket)
    ticket_id = ref[1].id

    log_event("INFO", "firestore_write", req, collection="support_tickets", ticket_id=ticket_id)
    log_event("INFO", "handler_success", req, handler="report_issue", ticket_id=ticket_id)

    return text_response(
        f"Your Nintendo support request has been submitted. Reference: {ticket_id}.",
        session_params={"request_ref_id": ticket_id}
    )


def handle_return_request(req: dict):
    params = get_params(req)
    order_id = str(params.get("order_id", "")).strip()
    reason = str(params.get("reason", "")).strip()
    account_number = str(params.get("account_number", "")).strip()

    log_event("INFO", "return_request_started", req, order_id=order_id, reason=reason)

    return_doc = {
        "order_id": order_id,
        "account_id": account_number,
        "reason": reason,
        "status": "PENDING"
    }

    ref = db.collection("returns").add(return_doc)
    return_id = ref[1].id

    log_event("INFO", "firestore_write", req, collection="returns", return_id=return_id)
    log_event("INFO", "handler_success", req, handler="return_request", return_id=return_id)

    return text_response(
        f"Your return request has been submitted. Reference: {return_id}.",
        session_params={"request_ref_id": return_id}
    )


def handle_create_handoff_context(req: dict):
    params = get_params(req)

    handoff_context = {
        "account_number": params.get("account_number"),
        "customer_name": params.get("customer_name"),
        "loyalty_tier": params.get("loyalty_tier"),
        "issue_type": params.get("franchise"),
        "order_id": params.get("order_id"),
        "priority_level": params.get("priority_level"),
        "authenticated": params.get("authenticated", False),
        "escalated": True
    }

    ref = db.collection("handoff_contexts").add(handoff_context)
    context_id = ref[1].id

    log_event("INFO", "firestore_write", req, collection="handoff_contexts", context_id=context_id)
    log_event("INFO", "handler_success", req, handler="create_handoff_context", context_id=context_id)

    return text_response(
        "Alright, I'm connecting you to a Nintendo specialist now.",
        session_params={"escalated": True, "handoff_context_id": context_id}
    )


# Handler registry — add new tags here without touching dispatch logic
HANDLER_REGISTRY = {
    "account_lookup": handle_account_lookup,
    "product_lookup": handle_product_lookup,
    "order_status": handle_order_status,
    "movie_info": handle_movie_info,
    "report_issue": handle_report_issue,
    "return_request": handle_return_request,
    "create_handoff_context": handle_create_handoff_context,
}


@functions_framework.http
def agent_webhook(request):
    req = request.get_json(silent=True) or {}
    tag = req.get("fulfillmentInfo", {}).get("tag", "")
    session_id = req.get("sessionInfo", {}).get("session", "unknown")

    logger.info(json.dumps({
        "severity": "INFO",
        "event": "dispatch",
        "tag": tag,
        "session": session_id
    }))

    handler = HANDLER_REGISTRY.get(tag)
    if not handler:
        logger.error(json.dumps({
            "severity": "ERROR",
            "event": "unknown_tag",
            "tag": tag,
            "session": session_id
        }))
        return make_response(jsonify({"error": f"Unknown tag: {tag}"}), 400)

    try:
        return handler(req)
    except Exception as e:
        logger.error(json.dumps({
            "severity": "ERROR",
            "event": "handler_error",
            "tag": tag,
            "session": session_id,
            "error": str(e)
        }))
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
| `account_lookup`         | read `accounts`, read `orders` | `account_number`                                                                                              | `customer_name`, `loyalty_tier`, `authenticated`, `order_list` | `account_lookup_started`, `firestore_read`, `handler_success`  |
| `product_lookup`         | read `products`                | `game`, `franchise`, `console`                                                                                | `product_name`, `product_url`, `product_price`                 | `product_lookup_started`, `firestore_read`, `handler_success`  |
| `order_status`           | read `orders`                  | `selected_order` or `order_id`                                                                                | `order_id`, `tracking_number`, `order_status_value`            | `order_status_started`, `firestore_read`, `handler_success`    |
| `movie_info`             | read `movies`                  | `franchise`                                                                                                   | `movie_title`, `movie_release_date`                            | `movie_info_started`, `firestore_read`, `handler_success`      |
| `report_issue`           | write `support_tickets`        | `console`, `franchise`, `description`, `account_number`, `priority_level`                                     | `request_ref_id`                                               | `report_issue_started`, `firestore_write`, `handler_success`   |
| `return_request`         | write `returns`                | `order_id`, `reason`, `account_number`                                                                        | `request_ref_id`                                               | `return_request_started`, `firestore_write`, `handler_success` |
| `create_handoff_context` | write `handoff_contexts`       | `account_number`, `customer_name`, `loyalty_tier`, `franchise`, `order_id`, `priority_level`, `authenticated` | `escalated`, `handoff_context_id`                              | `firestore_write`, `handler_success`                           |


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



