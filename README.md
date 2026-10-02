# Project Plan: Multi-Tenant Egyptian-Arabic AI Voice Assistant for Restaurants (B2B SaaS)

> **Instructions to the AI reading this:** You are the lead platform engineer for this project. This product is a **multi-tenant B2B SaaS platform** built to be marketed and sold to **multiple independent restaurant brands and chains**. Work phase by phase, in order. At the start of each phase, restate what you are about to build, list missing inputs, and **ask rather than invent** (tenant configurations, menus, telephony details, API keys). Verify every API detail (model names, event names, parameters) against **current official documentation**. Every component must support multi-tenancy: strict tenant isolation, dynamic routing by phone number, configurable restaurant personas/menus, and per-client usage tracking & billing.

---

## 1. Product Vision & Goal

Build and operate a multi-tenant voice AI platform that serves **multiple restaurant clients** across Egypt and the MENA region. 

For each subscribed restaurant, the system:
1. Answers customer phone calls in natural **Egyptian Arabic dialect**.
2. Understands customer intent, takes orders from that restaurant's specific dynamic menu, manages modifiers, and confirms customer details (delivery address/pickup).
3. Delivers confirmed orders instantly to that specific restaurant's destination (WhatsApp Business number, POS, or cashier system).
4. Provides seamless human escalation to branch staff when needed.
5. Logs every call, records metrics, runs automated QA audits, and generates **client-ready daily/monthly usage and billing reports**.

### Business & Operational Model
- **Product Type:** B2B Voice AI SaaS / Managed Solution for F&B.
- **Tenants:** Multiple independent restaurant brands/owners, each with one or more branches.
- **Onboarding Speed:** Fast, template-driven onboarding. Adding a new restaurant requires zero code changes—only database configuration (menu, branches, phone mapping, prompt variables).
- **Revenue & Margins:** Subscriptions + per-minute or per-order usage fee, tracking raw provider costs (Twilio + LLM + TTS/STT) against restaurant billing rates.
- **Non-goals for MVP:** Outbound telemarketing, in-call credit card payments, delivery rider dispatch.

---

## 2. Success Criteria

| Metric | Target | Notes |
|---|---|---|
| **Multi-Tenant Isolation** | 100% | Zero data leakage or cross-talk between restaurant tenants |
| **Order Accuracy** | ≥ 95% on test calls | Correct items, quantities, required modifiers, address |
| **Autonomous Resolution** | ≥ 70% | Calls completed without human staff intervention |
| **Response Latency** | < 1.5 s typical | Turn-around time from caller silence to speech response |
| **New Tenant Onboarding Time** | < 30 minutes | From receiving menu/numbers to live test call |
| **Cost & Margin Visibility** | 100% auditable | Every call maps exact provider costs to the respective tenant |
| **Wrong-Order Rate** | < 2% of total orders | Flagged and reported via automated post-call review |

---

## 3. Multi-Tenant Architecture & Call Flow

```
[ Customer Calling Restaurant A ]         [ Customer Calling Restaurant B ]
               │                                         │
               ▼                                         ▼
   Restaurant A Public Number                Restaurant B Public Number
               │ (Call Forwarding)                       │ (Call Forwarding)
               ▼                                         ▼
    Twilio Number A (Tenant A)                Twilio Number B (Tenant B)
               │                                         │
               └────────────────────┬────────────────────┘
                                    │
                                    ▼
                         Twilio Programmable Voice
                         (Media Streams WebSocket)
                                    │
                                    ▼
                 Multi-Tenant Voice Gateway (FastAPI)
      ┌──────────────────────────────────────────────────────────┐
      │ 1. Dynamic Tenant Resolution (Called Number -> Tenant)   │
      │ 2. Hydrate Tenant Context (Menu, Prompts, Branch Staff)  │
      │ 3. Voice AI Session (OpenAI Realtime / Gemini Live / Qwen)│
      │ 4. Scoped Tools (Tenant-isolated menu search & cart)     │
      └──────────────┬────────────────────────────┬──────────────┘
                     │                            │
                     ▼                            ▼
       PostgreSQL (Multi-Tenant DB)      Tenant Webhook / n8n
     - restaurants (tenants)              ├─► WhatsApp (Restaurant A/B)
     - branches                           ├─► Cashier / POS API
     - menu_items & modifiers             ├─► Automated Call QA Review
     - calls, orders & transcripts        └─► Tenant Daily & Monthly Reports
     - per-tenant usage & billing
```

### Dynamic Tenant Resolution Flow
1. **Inbound Call:** Twilio sends a webhook request to `/voice` with `Called` (the dialed Twilio number) and `From` (customer's caller ID).
2. **Tenant & Branch Lookup:** The gateway queries PostgreSQL to resolve which `restaurant_id` and `branch_id` own the dialed number.
3. **Session Hydration:** The gateway initializes the session with:
   - The restaurant's customized Egyptian dialect system prompt and rules (tone, brand name, recording disclaimer).
   - The restaurant's branch operational rules (delivery radius, working hours, minimum order).
   - Tool bindings scoped strictly to `restaurant_id` (so `get_menu` and `add_item` only access that restaurant's inventory).
4. **Failsafe Redirection:** If the gateway is down, Twilio's fallback action immediately redirects the call to that specific branch's emergency staff phone number.

### High-Concurrency & Simultaneous Calls Architecture
The platform is designed natively for high-throughput concurrency during peak lunch and dinner rushes:
- **Simultaneous Calls to the SAME Restaurant (Rush-Hour Scale):**
  - Unlike traditional single-line phone setups where callers receive a busy signal, Twilio supports elastic concurrent incoming streams on the exact same virtual number.
  - If 10 or 50 customers call "Restaurant A" simultaneously, the gateway immediately spawns 10 or 50 independent `CallSession` instances.
  - Each caller interacts with their own isolated AI ordering assistant with an independent shopping cart, audio stream, and state.
- **Simultaneous Calls Across DIFFERENT Restaurants:**
  - Calls to Restaurant A and Restaurant B execute concurrently within the asynchronous event loop without cross-talk or performance interference.
  - Tenant data (menus, carts, prompts, and order webhooks) is strictly segregated per session using `restaurant_id`.
- **How Concurrency is Handled Across Every Layer:**
  1. **Voice Gateway (FastAPI + Asyncio + Uvicorn Workers):**
     - Non-blocking async event loop multiplexes hundreds of concurrent audio WebSockets with minimal CPU/RAM.
     - Multi-worker deployment (e.g. 4 Uvicorn workers behind reverse proxy) utilizes all CPU cores.
     - Isolated in-memory `CallSession` keyed by unique Twilio `CallSid` ensures zero state bleeding.
  2. **Workflow Automation (n8n in Queue Mode with Redis):**
     - When multiple calls complete orders at the exact same second, n8n webhooks acknowledge instantly with `200 OK` (non-blocking).
     - n8n runs in **Queue Mode** with Redis (`EXECUTIONS_MODE=queue`) and dedicated background workers. High bursts of concurrent orders are queued and processed asynchronously without dropping or delaying orders.
     - **WhatsApp Outbound Throttling & Queue:** Respects Meta Cloud API / Twilio WhatsApp message rate limits (e.g., 20–80 msg/sec per sender number) via worker-level pacing so high bursts don't trigger HTTP 429 rate limit errors.
     - **Idempotency & Deduplication:** Orders carry a unique `order_id` / `call_id` to prevent double-dispatch during rapid concurrent retries.
  3. **Database (PostgreSQL + Async Connection Pooling):**
     - Async pool (`asyncpg`) with tuned `pool_size=30, max_overflow=50` handles concurrent menu lookups and order commits without blocking the voice event loop.
     - Strict indexing on `restaurant_id`, `branch_id`, and `twilio_phone_number` ensures sub-millisecond tenant resolution even under heavy read load.
  4. **Reverse Proxy & OS Tuning (Caddy / Nginx):**
     - Configured with `worker_connections 8192` and elevated OS file descriptors (`ulimit -n 65535`) to keep thousands of concurrent WebSockets open without dropping frames.
  5. **Provider Concurrency & Overflow Fallback:**
     - AI Realtime model concurrency quotas are monitored per tenant.
     - If concurrent calls exceed a restaurant's subscription tier or platform safety threshold, excess calls automatically and gracefully failover to the branch's human staff number via Twilio fallback.

---

## 4. Technology Stack & Sizing

| Layer | Technology | Multi-Tenant Architecture Role |
|---|---|---|
| **Telephony** | **Twilio Voice / SIP Trunking** | Dedicated or forwarded virtual numbers mapped to distinct restaurant tenants and branches; elastic concurrent line capacity. |
| **Voice AI Engine (Default)** | **OpenAI Realtime (`gpt-realtime-2.1-mini`)** | Direct `g711_ulaw` audio streaming, zero transcoding overhead, fast conversational response. |
| **Voice AI Engine (Alternatives)** | **Gemini Live**, **Qwen Omni** | Pluggable behind a unified `VoiceModelAdapter` interface for performance & cost benchmarking. |
| **Application Server** | **Python 3.12 + FastAPI + asyncio** | High-concurrency async WebSocket gateway (Uvicorn multi-worker) handling simultaneous calls across all tenants. |
| **Database** | **PostgreSQL** | Strict tenant isolation via indexed `restaurant_id` on all entities; `asyncpg` connection pooling. |
| **Workflow & Integration** | **n8n (Queue Mode) + Redis** | Distributed queue mode for burst order delivery to WhatsApp/POS, scheduled audits, and reports without dropped payloads. |
| **Call Audit & QA** | **GPT-4o-mini / Gemini Flash** | Automated post-call evaluation of order accuracy, flagging anomalies and issues. |
| **Reverse Proxy & TLS** | **Caddy / Nginx** | TLS termination, WebSocket connection multiplexing, and ulimit tuning (`65535` sockets). |
| **Secrets & Config** | `.env` + Secure Vault / DB Config | Global infrastructure credentials kept separate from per-tenant integration keys. |

### Recommended Production VPS Sizing
- **Pilot Phase (1–3 Restaurants, up to 10 concurrent calls):**
  - 4 vCPUs, 8 GB RAM, 80 GB NVMe SSD, 1 Gbps network port.
  - Single-node Docker Compose with standalone n8n and Postgres.
- **Commercial SaaS Scale (10–50 Restaurants, 50–200 concurrent calls):**
  - 8–16 vCPUs, 16–32 GB RAM, 160 GB+ NVMe SSD.
  - Multi-worker gateway + Redis-backed n8n worker nodes + dedicated Postgres with connection pooling (`PgBouncer`).

### Resource Optimization & Production Hardening (8 GB RAM / 80 GB NVMe)
1. **n8n Memory Management:**
   - By default, n8n saves all execution data, which will quickly exhaust 8 GB RAM during heavy call volumes.
   - **Prune Execution Data:** Set `EXECUTIONS_DATA_PRUNE=true`.
   - **Set Retention Hours:** Set `EXECUTIONS_DATA_MAX_AGE=24` (or lower) to delete old logs automatically.
   - **Failures Only (Optional):** Set `EXECUTIONS_DATA_SAVE_ON_SUCCESS=none` to audit failed call flows without storing successful payloads in memory/DB.
2. **PostgreSQL Tuning (for 8 GB RAM VPS):**
   - The default Postgres configuration is tailored for low-resource environments and won't utilize 8 GB RAM effectively.
   - **Shared Buffers:** Set `shared_buffers = 2GB` (25% of total system RAM).
   - **Work Memory:** Set `work_mem = 64MB` to handle complex queries efficiently without swapping to disk.
   - **Maintenance Work Memory:** Set `maintenance_work_mem = 512MB`.
   - **Effective Cache Size:** Set `effective_cache_size = 6GB`.
3. **Storage & I/O Protection:**
   - An 80 GB NVMe drive provides excellent read/write speeds for database logging, but unconstrained container logs will deplete storage.
   - **Docker Log Rotation:** Configure Docker's log-driver with `max-size=10m` and `max-file=3` across all services in `docker-compose.yml` to prevent logs from eating up disk space.

---

## 5. Repository Structure

```
voice-ordering/
├─ docker-compose.yml
├─ .env.example
├─ gateway/
│  ├─ main.py                 # FastAPI app: /voice (TwiML routing), /media-stream (WS), /status
│  ├─ twilio_handler.py       # TwiML generation, tenant fallback forwarding, transfer handling
│  ├─ session.py              # Per-call session state: tenant context, cart, transcript, timers
│  ├─ tenant_manager.py       # Tenant resolution, caching, and config loading
│  ├─ adapters/
│  │  ├─ base.py              # Abstract VoiceModelAdapter interface
│  │  ├─ openai_realtime.py   # OpenAI Realtime implementation
│  │  ├─ gemini_live.py       # Google Gemini Live implementation
│  │  └─ qwen_omni.py         # Alibaba Qwen Omni implementation
│  ├─ tools.py                # Tenant-scoped function tools (menu search, cart, validation)
│  ├─ prompts/
│  │  ├─ base_template.py     # Base Egyptian Arabic ordering prompt with variable placeholders
│  │  └─ builder.py           # Injects tenant brand, greeting, recording notices, and rules
│  ├─ billing.py              # Per-tenant cost calculation (telephony + token usage + platform markup)
│  └─ db.py                   # Async database operations and tenant queries
├─ db/
│  ├─ migrations/             # Schema definitions and migrations
│  └─ seed_tenant.py          # CLI script to bootstrap new restaurant menus and branches
├─ n8n/workflows/             # Tenant-parameterized workflows (order delivery, QA review, reporting)
├─ scripts/
│  ├─ onboard_restaurant.py   # Onboarding CLI: validate menu CSV/JSON and register new tenant
│  └─ simulate_call.py        # Automated test calls simulating customer orders across tenants
├─ tests/                     # Unit and integration test suite
└─ docs/
   ├─ tenant-onboarding.md    # Standard operating procedure (SOP) to onboard a new restaurant
   └─ runbook.md              # Incident response and platform operations guide
```

---

## 6. Multi-Tenant Data Model

Every operational table is scoped by `restaurant_id` to guarantee tenant isolation:

```mermaid
erDiagram
    RESTAURANTS ||--o{ BRANCHES : owns
    RESTAURANTS ||--o{ MENU_ITEMS : catalogs
    BRANCHES ||--o{ CALLS : receives
    CALLS ||--o{ ORDERS : generates
    CALLS ||--o{ TRANSCRIPTS : records
    CALLS ||--o{ REVIEWS : evaluates
    CALLS ||--o{ USAGE_LOGS : tracks

    RESTAURANTS {
        uuid id PK
        string name
        string slug
        string status "active | suspended | trial"
        jsonb config "prompts, tone, recording_notice, max_call_duration"
        jsonb billing_settings "currency, rate_per_minute, plan"
        timestamp created_at
    }

    BRANCHES {
        uuid id PK
        uuid restaurant_id FK
        string branch_name
        string twilio_phone_number "Indexed for inbound tenant resolution"
        string staff_backup_number "Emergency fallback/handoff line"
        jsonb operating_hours
        jsonb delivery_zones
        jsonb order_destination "WhatsApp number, webhook, or POS credentials"
    }

    MENU_ITEMS {
        uuid id PK
        uuid restaurant_id FK
        string name_ar
        string[] aliases "Slang, alternative pronunciations, and typos"
        string category
        decimal price
        boolean is_available
    }

    MODIFIERS {
        uuid id PK
        uuid item_id FK
        string name_ar
        decimal price_delta
        boolean is_required
        string group_name "e.g., Size, Spice level, Extras"
    }

    CALLS {
        uuid id PK
        uuid restaurant_id FK
        uuid branch_id FK
        string twilio_call_sid
        string caller_phone
        timestamp started_at
        timestamp ended_at
        integer duration_seconds
        string outcome "order_placed | staff_handoff | caller_abandoned | system_error"
        string model_used
    }

    ORDERS {
        uuid id PK
        uuid restaurant_id FK
        uuid branch_id FK
        uuid call_id FK
        jsonb order_items "Items, modifiers, quantities, unit prices"
        decimal total_amount
        string customer_name
        string customer_phone
        string delivery_address
        string order_type "delivery | pickup"
        string dispatch_status "pending | delivered | failed"
        timestamp sent_at
    }

    TRANSCRIPTS {
        uuid id PK
        uuid call_id FK
        jsonb turns
        boolean sampled
    }

    REVIEWS {
        uuid id PK
        uuid call_id FK
        uuid restaurant_id FK
        integer accuracy_score "1 to 100"
        jsonb detected_issues
        boolean flagged_for_human_review
        timestamp reviewed_at
    }

    USAGE_LOGS {
        uuid id PK
        uuid call_id FK
        uuid restaurant_id FK
        integer audio_in_tokens
        integer audio_out_tokens
        integer text_tokens
        decimal provider_cost_usd
        decimal tenant_charge_egp
        decimal fx_rate
    }
```

---

## 7. Implementation Phases

### Phase 0: Multi-Tenant Foundation & Onboarding Design
- Establish standardized onboarding templates (`docs/tenant-onboarding.md` and `menu_template.csv`).
- Design the onboarding workflow: ingest restaurant menu, clean Arabic aliases, configure branches and numbers, set delivery targets (WhatsApp/POS).
- Build the onboarding CLI (`scripts/onboard_restaurant.py`) for rapid setup of new restaurants.

### Phase 1: Infrastructure & Telephony Setup
- Provision VPS, domain, TLS reverse proxy (Caddy/Nginx with `65535` socket ulimits), and Docker environment.
- Configure Docker Compose: FastAPI gateway (multi-worker), PostgreSQL (tuned connection pool), Redis, and n8n (Queue Mode).
- Configure Twilio Programmable Voice with media stream endpoints (`/voice`, `/media-stream`, `/status`).
- Implement dynamic fallback URLs pointing to each branch's emergency staff line.

### Phase 2: Core Voice Gateway & Dynamic Resolution
- Implement FastAPI WebSocket server handling Twilio Media Streams across concurrent connections.
- Implement **Tenant Resolution Engine**: extract dialed number from Twilio handshake, resolve `restaurant_id` & `branch_id`, and load tenant config.
- Implement stream bidirectional audio exchange with interruption handling (barge-in support) and silence timeouts.

### Phase 3: Pluggable Voice AI Adapters
- Implement `VoiceModelAdapter` abstraction.
- Connect **OpenAI Realtime API** (`gpt-realtime-2.1-mini`) with direct `g711_ulaw` streaming.
- Build benchmarking harness to test **Gemini Live** and **Qwen Omni** using the same suite of recorded Egyptian order calls.

### Phase 4: Tenant Prompt Engine & Scoped Function Calling
- Build dynamic system prompt builder:
  - Inject restaurant persona, brand identity, Egyptian dialect nuances, and mandatory call-recording disclosure.
  - Enforce concise replies to minimize audio generation latency and cost.
- Implement tenant-isolated tools:
  - `get_menu`: queries only the active tenant's items and categories.
  - `add_item` / `remove_item` / `get_cart`: cart state scoped to call session.
  - `confirm_order`: verifies cart items against current DB prices and availability before committing.
  - `transfer_to_human(reason)`: routes caller to the branch staff line.

### Phase 5: Resilient Human Handoff
- Execute live `<Dial>` transfers to the branch staff number on trigger conditions (caller request, angry customer, repeated misunderstanding, complex inquiries).
- Announce transfer politely to caller in Egyptian dialect prior to dial.
- Log escalation reasons for tenant quality reporting.

### Phase 6: Multi-Tenant Order Delivery (n8n in Queue Mode & POS Connectors)
- Gateway publishes `order.confirmed` payload containing `restaurant_id`, `branch_id`, and unique `order_id` (idempotency key).
- n8n webhook returns immediate `200 OK` and dispatches the task to the Redis job queue (`EXECUTIONS_MODE=queue`).
- Dedicated n8n background workers evaluate tenant configuration and route the order:
  - Format 1: WhatsApp Business message formatted in Arabic to the branch cashier (with outbound message rate limiting to prevent Meta API throttling).
  - Format 2: Webhook post to the restaurant's POS / order-management API.
- Automated delivery retries with failure alerting to operations; orders tracked in DB as `pending`, `delivered`, or `failed`.

### Phase 7: Automated Post-Call QA & Audit Engine
- Background evaluation triggered after call termination.
- Fast, low-cost text model audits the call transcript against the final cart:
  - Validates correct item capture, modifier accuracy, and address clarity.
  - Flags problematic calls for operational inspection.
- Sample 10% of routine calls and 100% of escalated or flagged calls.

### Phase 8: Tenant Reporting & Billing Engine
- **Daily Performance Digest:** automated daily summary per restaurant (call volume, orders placed, handoff rate, peak order hours).
- **Monthly Client Invoicing Report:** detailed consumption breakdown (minutes utilized, call count, cost per minute/call, platform fee in EGP).
- Real-time internal gross margin dashboard comparing raw telephony + AI API costs against client billing.

### Phase 9: Multi-Scenario Benchmark & Concurrent Load Testing
- Execute 50+ scripted Egyptian Arabic test calls across different restaurant profiles (e.g., Fast Food, Koshary, Grill/BBQ, Cafe).
- **Concurrent Load Testing:**
  - Simulate 10–30 simultaneous calls hitting the *same restaurant* to verify zero audio stutter, cart isolation, and prompt responsiveness during dinner rush.
  - Simulate concurrent calls hitting *different restaurants* to confirm strict multi-tenant data segregation.
  - Burst-test n8n order queues and WhatsApp dispatch to ensure zero dropped orders under peak load.

### Phase 10: Multi-Restaurant Pilot Program
- **Tenant 1 Pilot:** Launch on one branch of the first restaurant client with on-site staff supervision.
- **Tenant 2 Pilot:** Onboard a second independent restaurant brand to validate onboarding speed and concurrent tenant isolation.
- Evaluate KPIs against targets and optimize menu aliases and speech prompts.

### Phase 11: Scale & Self-Service Operations
- Expand to additional branches and new restaurant clients.
- Provide restaurant owners with webhooks or a simple dashboard to update menu prices and toggle item availability.
- Deploy operational runbooks for 24/7 uptime monitoring.

---

## 8. Multi-Tenant Security, Privacy & Compliance

- **Strict Tenant Data Segregation:** Database queries always enforce tenant criteria (`WHERE restaurant_id = :id`). 
- **Customer Privacy:** Call recording disclaimer played in the opening greeting. Audio recordings and raw transcripts subject to configurable tenant retention periods (e.g., auto-purge transcripts after 30 days).
- **API Security:** All incoming Twilio webhooks verified with signature validation. Webhooks between gateway and n8n secured via HMAC SHA256 signatures.
- **Encrypted Storage:** Database backups and sensitive credentials encrypted at rest; no customer phone numbers or PII exposed in public logs.

---

## 9. Quality & Readiness Checklist

- [ ] Inbound calls to different Twilio numbers resolve the correct restaurant brand and branch
- [ ] Simultaneous calls to the *same* restaurant run without busy signals or cart bleeding
- [ ] Concurrent calls across *different* restaurants maintain strict tenant isolation
- [ ] Emergency fallback successfully dials branch staff if the AI gateway is unresponsive
- [ ] Caller speech barge-in immediately halts AI voice output without stream stutter
- [ ] Egyptian dialect, colloquial phrasing, numbers, and slang aliases correctly parsed
- [ ] Cart calculations strictly match current database prices with zero hallucinations
- [ ] Confirmed orders arrive at the correct restaurant WhatsApp/POS within 5 seconds
- [ ] n8n Redis queue absorbs high-burst order spikes without dropping payloads or exceeding WhatsApp rate limits
- [ ] Human handoff smoothly transfers the call and logs the escalation reason
- [ ] Post-call QA flags inaccurate orders and summarizes mismatch causes
- [ ] Usage tracking calculates telephony and AI costs accurately per restaurant
- [ ] Onboarding a new restaurant takes less than 30 minutes using the CLI tool

---

## 10. Platform Risks & Mitigations

| Risk | Impact | Mitigation Strategy |
|---|---|---|
| **Cross-Tenant Data Leakage** | Critical | Strict database constraints, tenant-scoped session factories, and automated multi-tenant regression tests. |
| **Telephony Line Availability in Egypt** | High | Support both Twilio international/local numbers and direct SIP Trunking (BYOC) with local Egyptian telecom providers. |
| **High Model Audio Token Costs** | Medium | Keep AI system prompts and voice responses crisp; leverage prompt caching; use cheaper text models for post-call tasks. |
| **Loud Background Noise in Calls** | Medium | Tune VAD (Voice Activity Detection) parameters and noise suppression thresholds on the audio stream. |
| **Diverse Restaurant POS Systems** | Medium | Decouple order dispatch into modular n8n workflows supporting WhatsApp, email, webhook, and custom POS APIs. |

---

## 11. Core Deliverables

1. **Multi-Tenant Voice Gateway:** Async FastAPI service supporting dynamic tenant routing and media streaming.
2. **Pluggable Voice AI Adapters:** Production integration with OpenAI Realtime, with evaluation benchmarks for Gemini Live and Qwen.
3. **Tenant Onboarding Toolkit:** Automated CLI and validation scripts to import restaurant menus, aliases, and branches.
4. **Order Dispatch & Notification System:** Ready-to-deploy n8n workflows for WhatsApp and POS order forwarding.
5. **Auditing & Billing System:** Post-call QA analysis pipeline and automated client usage/billing reports.
6. **Documentation & Runbooks:** Client onboarding SOP, incident handling guide, and production deployment configuration.

---

## 12. Next Steps to Begin Development

1. **Review Architecture:** Confirm the multi-tenant architecture and chosen telephony/AI adapters.
2. **Prepare Initial Tenant Data:** Collect the first pilot restaurant's menu, branches, staff numbers, and preferred order destination.
3. **Execute Phase 1 & 2:** Spin up the Docker stack and deliver a working prototype demonstrating dynamic tenant resolution on an inbound test call.