# Project Plan: Egyptian-Arabic AI Voice Assistant for Restaurant Phone Orders

> **Instructions to the AI reading this:** You are the lead engineer for this project. Work phase by phase, in order. At the start of each phase, restate what you are about to build, list any inputs you are missing, and **ask rather than invent** (menu data, phone numbers, API keys, order destinations). Verify every API detail (model names, event names, parameters) against the **current official docs** before coding, because model names and APIs change. Keep everything configurable through environment variables and config files, not hard-coded.

---

## 1. Goal

Build a voice assistant that answers a restaurant's order calls in **Egyptian Arabic dialect**, takes the order, confirms it, and sends it to the restaurant (WhatsApp or cashier system). A human can take over at any time. The system must be cheap to run per call, easy to clone for a new restaurant, and observable (every call logged, reviewed and reported).

**Non-goals for now:** outbound calls, payments over the phone, delivery tracking, pricing/packaging logic.

## 2. Success criteria (agree before the pilot)

| Metric | Target (adjust with the client) |
|---|---|
| Order accuracy (items + quantities + modifiers correct) | ≥ 95% on test calls |
| Calls handled without human help | ≥ 70% |
| Response delay (caller stops talking → assistant starts) | < 1.5 s typical |
| Wrong-order complaints during pilot | Below an agreed number per week |
| Cost per call | Tracked and reported; compared across models |

## 3. High-level architecture

```
Customer ──► Restaurant's existing number
                  │  (call forwarding)
                  ▼
            Twilio number ──► Twilio Programmable Voice
                  │  TwiML <Connect><Stream> (Media Streams, WebSocket)
                  ▼
        Voice Gateway (our server, FastAPI + WebSocket)
          ├─► Voice AI model (realtime speech-to-speech)  ◄── swappable adapter
          │      tools: get_menu / add_item / remove_item / confirm_order /
          │             transfer_to_human / end_call
          ├─► Postgres (menu, calls, orders, transcripts, usage)
          └─► Webhook ──► n8n ──► WhatsApp / Cashier API
                              ├─► Post-call review (cheap text LLM)
                              ├─► Daily report
                              └─► Monthly consumption report
```

**Failsafe:** if the gateway is down or errors, Twilio's fallback URL forwards the call to the restaurant's staff number. The caller must never hit a dead line.

## 4. Tools and stack

| Layer | Choice | Notes |
|---|---|---|
| Phone line | **Twilio** (number + Programmable Voice + Media Streams) | Verify Egyptian number availability and forwarding cost; alternative is a SIP trunk (BYOC) with a local provider |
| Voice AI (default) | **OpenAI Realtime, `gpt-realtime-2.1-mini`** | Not the deprecated 2025-12-15 snapshot. Twilio's `g711_ulaw` audio is supported directly, so no transcoding |
| Voice AI (to test) | **Gemini Live** (Flash Live / native audio), **Qwen Omni realtime** | Same test calls on all; behind one adapter interface |
| Server | **Python 3.12 + FastAPI + asyncio websockets** | Bridges Twilio ↔ model |
| Database | **PostgreSQL** | Multi-tenant from day one (`restaurant_id`) |
| Automation | **n8n** (self-hosted) | Order delivery, reports, review trigger |
| Review / reports LLM | Cheap text model (GPT-4o-mini class or Gemini Flash) | Post-call only, not realtime |
| Order delivery | WhatsApp Business API (Twilio or Meta Cloud API) **or** the restaurant's cashier API | Decided per restaurant |
| Hosting | One **VPS** with Docker Compose (gateway, n8n, Postgres, Caddy/Nginx for HTTPS/WSS) | Twilio needs a public HTTPS + WSS endpoint |
| Secrets | `.env` + Docker secrets | Never commit keys |
| Monitoring | Structured logs, uptime check, error alerts to a WhatsApp/Telegram channel | |

## 5. Repository structure

```
voice-ordering/
├─ docker-compose.yml
├─ .env.example
├─ gateway/
│  ├─ main.py                 # FastAPI app: /voice (TwiML), /media-stream (WS), /status
│  ├─ twilio_handler.py       # TwiML, call transfer, fallback
│  ├─ session.py              # per-call state: cart, transcript, timers
│  ├─ adapters/
│  │  ├─ base.py              # VoiceModelAdapter interface
│  │  ├─ openai_realtime.py
│  │  ├─ gemini_live.py
│  │  └─ qwen_omni.py
│  ├─ tools.py                # tool/function implementations
│  ├─ prompts/                # system prompt templates per restaurant
│  ├─ costs.py                # usage → cost calculation
│  └─ db.py
├─ db/migrations/
├─ n8n/workflows/             # exported JSON: order_delivery, post_call_review, daily_report, monthly_report
├─ tests/                     # unit tests + call simulation scripts
└─ docs/                      # onboarding checklist, runbook
```

## 6. Data model (Postgres)

- `restaurants` (id, name, config JSON: language style, greeting, recording notice, max call seconds, handoff number, order destination)
- `branches` (id, restaurant_id, name, twilio_number, staff_number, hours, delivery areas)
- `menu_items` (id, restaurant_id, name_ar, aliases[] (slang/spelling variants), category, price, available bool)
- `modifiers` (id, item_id, name_ar, price_delta, required bool, group)
- `calls` (id, branch_id, twilio_call_sid, started_at, ended_at, duration_s, outcome [order / handoff / abandoned / error], model_used)
- `orders` (id, call_id, items JSON, total, customer_name, phone, address, notes, status, sent_at)
- `transcripts` (call_id, turns JSON, sampled bool)
- `reviews` (call_id, accuracy_score, issues[], flagged bool, reviewed_at)
- `usage` (call_id, audio_in_tokens, audio_out_tokens, text_tokens, cached_tokens, est_cost_egp, fx_rate)

## 7. Build phases

### Phase 0: Discovery and inputs (before coding)
Collect from the restaurant: full menu with prices, add-ons, unavailable items; current phone number and how it can be forwarded; number of branches; where orders should go (WhatsApp / cashier); a staff contact; busy hours; a few real call recordings if possible (with permission).
**Output:** `docs/onboarding-checklist.md` filled in, menu loaded into a CSV/JSON template.

### Phase 1: Infrastructure
1. Provision VPS, domain, TLS.
2. Docker Compose: gateway, Postgres, n8n, reverse proxy.
3. Create Twilio account/number; configure voice webhook → `/voice`, status callback → `/status`, **fallback URL → forward to staff number**.
4. Set up call forwarding from the restaurant's number (all calls, or busy/no-answer only; decide with the client).
5. Health-check endpoint and uptime monitor.

### Phase 2: Gateway and basic call loop
1. `/voice` returns TwiML `<Connect><Stream url="wss://.../media-stream">`.
2. `/media-stream` accepts Twilio frames, forwards audio to the model adapter, streams model audio back.
3. Handle interruption (caller speaks over assistant: clear Twilio's audio buffer and cancel the model response).
4. Max call duration guard and silence timeout.
5. Log every call to `calls`.
**Done when:** a test call has a live two-way voice conversation with the model.

### Phase 3: Model adapter layer
Define `VoiceModelAdapter` (connect, send_audio, receive events, send tool result, close, get_usage). Implement OpenAI first; add Gemini Live and Qwen later behind the same interface so models can be swapped by config. Configure input transcription cheaply (or sample only) and capture usage from response-complete events.

### Phase 4: Assistant behavior (prompt and tools)
**System prompt requirements** (per restaurant, templated):
- Persona: polite Egyptian-dialect restaurant order-taker, short natural sentences, no formal Modern Standard Arabic unless the caller uses it.
- Greeting includes the **call-recording notice**.
- Flow: greet → take items → ask for missing required modifiers → read back the full order and total → ask for name, phone, address (delivery) or pickup → final confirmation → close.
- Never invent menu items or prices; only use `get_menu` data. If the item is unavailable, offer alternatives.
- Keep replies **short** (audio output is the main cost driver).
- Handle numbers, quantities and slang item names via the `aliases` list.
- Escalate to a human when: the caller asks, the caller is angry, complaint/refund, the assistant fails to understand twice in a row, or anything outside ordering.

**Tools (function calling):** `get_menu`, `add_item(item_id, qty, modifiers[], notes)`, `remove_item`, `get_cart`, `set_customer_details`, `confirm_order`, `transfer_to_human(reason)`, `end_call(reason)`.
`confirm_order` validates the cart against the DB (prices, availability) before saving and firing the webhook.

### Phase 5: Human handoff
`transfer_to_human` updates the live call via Twilio REST with `<Dial>` to the branch staff number. Pass a short spoken line first ("هحولك لحد من الفريق"). Log reason and outcome. Test the case where nobody answers (return to the assistant or take a message).

### Phase 6: Order delivery (n8n)
1. Gateway posts `order.confirmed` JSON to an n8n webhook (signed with a shared secret).
2. n8n formats the order in Arabic and sends it to WhatsApp or the cashier API.
3. Retry on failure; if delivery fails, alert the team and keep the order in the DB as `unsent`.
4. Mark `orders.status` and `sent_at`.

### Phase 7: Post-call review
Triggered by the Twilio status callback after the call ends. n8n sends the transcript + final order JSON to the cheap text LLM, which scores accuracy, lists mismatches, and flags the call for human review. Store in `reviews`. Transcribe fully only a **sample (~10%) plus every flagged, handed-off or errored call**.

### Phase 8: Reports
- **Daily report** (n8n cron, sent to WhatsApp/email): calls, total minutes, orders, handoffs, flagged calls with reasons, average duration, estimated cost.
- **Monthly consumption report**: minutes by day, cost by component (voice model, line, review, server), trends, suggestions.
- Keep an `fx_rate` setting; costs are computed in USD and converted to EGP.

### Phase 9: Cost controls
Per-call usage logging; max call length; alert when daily or monthly minutes pass configured thresholds; hard monthly cap option; cached-input usage where the API supports it; concise prompt and short replies; alert on abnormal call lengths (possible loops or spam).

### Phase 10: Internal testing (our cost)
1. Write 30–50 scripted scenarios in Egyptian dialect: simple order, many items, modifiers, changes mid-order, unavailable item, unclear speech, noisy background, fast talkers, interruptions, angry caller, off-topic question, silence, wrong number.
2. Run the **same scenarios on each candidate model** (OpenAI mini, Gemini Live, Qwen options). Record accuracy, delay, handoff rate, cost per call.
3. Fix prompt/tool issues, rerun.
4. Produce a short comparison report and pick the cheapest model that meets the accuracy target.
5. Review the real measured cost and share it with the client before the pilot.

### Phase 11: Pilot (one branch)
Run on one branch with a backup staff member present, reduced scope agreed in advance. Weekly review of flagged calls and metrics. Tune menu aliases, prompt and handoff rules. Decide go/no-go against the agreed success criteria.

### Phase 12: Rollout and maintenance
Onboard remaining branches using the cloneable config (new branch = new row + Twilio number + prompt variables). Menu/price update procedure (CSV import or admin form). Monthly report. Runbook for incidents.

## 8. Security, privacy and compliance
- Recording notice in the greeting; store only what is needed.
- No sharing of call or customer data outside the system and the chosen AI/telephony providers; confirm each provider's data retention and region settings (especially for any provider hosted outside the region).
- Verify Twilio webhook signatures; sign n8n webhooks; restrict ports; keep API keys in secrets; encrypt backups.
- Retention policy for audio/transcripts (e.g., delete sampled transcripts after N days); allow deletion on request.

## 9. Testing checklist
- [ ] Forwarding works from the restaurant's number
- [ ] Fallback to staff works when the gateway is down
- [ ] Interruptions handled cleanly
- [ ] Dialect, numbers, quantities and slang item names recognized
- [ ] Cart totals match the DB prices
- [ ] Order reaches WhatsApp/cashier within seconds, with retries on failure
- [ ] Handoff connects to a human and logs a reason
- [ ] Review flags a deliberately wrong order
- [ ] Reports arrive on schedule and numbers match the DB
- [ ] Cost logs match the providers' invoices within a small margin

## 10. Risks and open questions
| Risk / question | Plan |
|---|---|
| Egyptian dialect quality differs by model | Decide only after the multi-model test |
| Twilio Egyptian number availability / forwarding cost | Verify early; fall back to SIP trunk or a number in another country if needed |
| Model names and prices change | Config-driven model name; re-check pricing pages before pilot |
| Noisy restaurant calls and bad lines | Test with real recordings; add noise-heavy scenarios |
| Order destination varies per restaurant | Keep delivery in n8n workflows, one per destination type |
| Provider outage | Fallback forwarding to staff; alerting |
| Data location / privacy concerns for some providers | Confirm region and retention before choosing |

## 11. Deliverables
Working gateway and database; n8n workflows (exported); prompt templates; model comparison report; onboarding checklist; runbook; daily and monthly report templates; pilot results summary.

## 12. How to start (first messages the AI should send)
1. Confirm the stack above or propose changes with reasons.
2. Ask for: restaurant menu and sample calls, order destination, existing phone line type, VPS/domain details, and API keys needed (Twilio, OpenAI, Google, Alibaba as applicable).
3. Begin with **Phase 1 and Phase 2**, and demo a working test call before moving on.