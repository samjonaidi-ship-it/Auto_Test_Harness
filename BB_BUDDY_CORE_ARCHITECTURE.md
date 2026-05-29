# BB Buddy Core Architecture | v2.6 | 2026-04-02 | BB

**This is the architecture reference document.** For implementation plans, see companion files:
- `BB_BUDDY_TRACK_0_DELIVERY.md` — Test harness (Track 0 delivery plan)
- `BB_BUDDY_CREW_PLATFORM.md` — Crew platform (A0-A3 phases)
- `BB_HOME_PLATFORM_EXPANSION.md` — Future homeowner platform (B1-B7 design)

---

## Execution Boundary Matrix (Committed vs. Design-Only vs. Forbidden)

| Area | Track | Status | Build Now? | Owner | Gate to Activate |
|------|-------|--------|-----------|-------|-----------------|
| **Test harness core** (runner, waves, contracts) | Track 0 | Committed | YES | Claude Code | None — critical path |
| **Observer + synthetic data** | Track 0 | Committed | YES | Claude Code | T0.1 complete |
| **AI evaluators** (voice + RAG + scale) | Track 0 | Committed | YES | Claude Code | T0.3 COMPLETE |
| **OpenAI Agents SDK** | Track A (A0) | Committed | YES | Claude Code | None — can start immediately |
| **MCP tool layer** | Track A (A1) | Committed | YES | Claude Code | T0.1 complete |
| **RAG crew knowledge** | Track A (A2) | Committed | YES | Claude Code | T0.3 + T0.5 COMPLETE |
| **Operations + write workflows** | Track A (A3) | Committed | YES | Claude Code | T0.5 complete |
| **Customer trust + auth** | Track B (B1) | Design-only | NO | Future team | A3 production validation (4+ weeks) |
| **Customer ingestion** | Track B (B2) | Design-only | NO | Future team | B1 design + security review |
| **Property intelligence** | Track B (B3) | Design-only | NO | Future team | B2 + 50+ customers |
| **Scheduling + service UX** | Track B (B4) | Design-only | NO | Future team | Property intelligence stable |
| **Proactive assistant** | Track B (B5) | Design-only | NO | Future team | B4 operational + seasonal data |
| **Multi-tenant RAG** | Track B | Design-only | NO | Future team | B1 design (RLS enforcement) |
| **Stripe billing integration** | Track B | Forbidden now | NO | None | Only after A3 production validation |
| **Jobber scheduling** | Track B | Forbidden now | NO | None | Only after A3 production validation |
| **Homeowner document upload** | Track B | Forbidden now | NO | None | Only after A3 production validation |
| **DBSCAN geo-clustering** | Track B (B5+) | Forbidden now | NO | None | Activate at 150+ subscribers (Year 2) |

---

## Now / Next / Forbidden (Summary)

### NOW (Committed Build)
- **Track 0:** Test harness infrastructure (6 stages, 6-9 weeks)
- **A0-A3:** Crew-only platform (SDK eval → MCP → RAG crew docs → crew operations)
- **Crew features:** Voice + vision + knowledge lookup + financial queries + crew scheduling + CalExp integration

### NEXT (After A3 production validation)
- **Track B:** Homeowner subscription platform (design-only until A3 proven in production)
- **Homeowner features:** Document upload, property intelligence, service scheduling, subscriptions

### FORBIDDEN NOW
- ❌ Homeowner tenancy features (B1+)
- ❌ Stripe billing / subscription management (B4+)
- ❌ Customer document upload flows (B2+)
- ❌ Multi-tenant UI / homeowner apps (Track B)
- ❌ Property intelligence / home asset tracking (B3+)
- ❌ Customer scheduling via Jobber (B4+)
- ❌ DBSCAN geo-clustering / spatial features (B5+, activate only at 150+ subscribers)

---

## Vision

BB Buddy evolves from a hardcoded 3-model client into a **scalable agentic mesh** where models can be swapped into most roles (with OpenAI Realtime as a permanent lock-in for voice), agents communicate via MCP, and BB's entire business knowledge — receipts, invoices, contracts, emails, SOPs, vendor pricing — is available via RAG. This vision is divided into three execution tracks:

- **Track 0 (committed now):** Test harness + quality infrastructure (6 stages, 6-9 weeks)
- **Track A (committed now):** Crew-only platform with voice + RAG + operations (A0-A3, 9 weeks, parallel with Track 0)
- **Track B (design-only until A3 production validation):** Homeowner subscription platform (B1-B7, deferred to Q3 2026+)

Zero UX penalty. Crew points, talks, gets answers.

---

## Current State (v3.17) — What Works

```
PHONE (bb-scan-openai.html — single monolithic client)
  ├── OpenAI Realtime Mini ← WebRTC audio/video, function calling
  ├── Claude Sonnet 4.6 ← direct browser REST for expert knowledge + web search
  ├── SerpAPI ← via Bridge for asset search
  └── 5 hardcoded tools (ask_expert, log_item, deliver_report, calexp_action, find_asset)

BRIDGE (BB_Micro_Bridge — data persistence + context only)
  ├── /init → returns API keys to client
  ├── /v2/transcripts → sendBeacon batch storage
  ├── /v2/session/* → session lifecycle
  ├── /v2/crew-memory → per-employee memory
  └── /v2/search → SerpAPI proxy
```

**What's good:** Zero-latency voice (WebRTC direct to OpenAI), working function call lifecycle, echo suppression, hallucination filtering, crew memory, session persistence.

**What's rigid:** Models hardcoded, tools defined in HTML, no RAG, no model abstraction, API keys sent to client, no agent protocol.

---

## Track 0 → Track A Validation Chain

**Core principle: Nothing ships from Track A without passing Track 0 gates.**

```
TRACK 0 (Test Infrastructure)          TRACK A (Crew Product)
────────────────────────────            ────────────────────────

T0.1: Harness Core                     ← blocks until passed
↓
T0.2: Observer Dashboard               (T0.2 parallel work, not gate)
T0.3: Synthetic Data                   ← A1 requires T0.1 + T0.3
↓
T0.4: SDV + Assets                     ← A2 requires T0.3 + T0.5
T0.5: AI Evaluators                    
↓
T0.6: Scale + Cross-Platform           ← A3 requires T0.5 + T0.6
↓
T0.7: Post-Launch Hardening (Fix Agent)

Key insight:
- T0 always runs ~1 phase ahead of A
- A gates = T0 completion gates (not soft dependencies)
- If T0 fails, A is blocked until fixed
- No "ship anyway" override — gates are hard stops
```

**Gate table (canonical — must match Track 0 + Crew Platform docs exactly):**

| A Phase | REQUIRES | Why | Timeline |
|---------|----------|-----|----------|
| A0 | None (parallel start) | SDK eval doesn't need harness | Weeks 1-2 |
| A1 | **T0.1 COMPLETE + A0 COMPLETE** (hard gates) | MCP tools must validate in harness. A0 COMPLETE = evaluation done, path chosen (SDK or raw WebRTC). | Week 3+ |
| A2 | **T0.3 COMPLETE + T0.5 COMPLETE** (hard gates) | Seeded data required for golden retrieval set. Working evaluators required. | Week 5+ |
| A3 | **T0.5 COMPLETE + T0.6 COMPLETE** (hard gates) | Full AI eval + scale testing required for production validation. | Week 7+ |

**All gates are HARD STOPS.** Phase cannot advance without gate satisfaction. No "partial", no "baseline".

---

## Target Architecture — Agentic Mesh

### The Dual Orchestrator Architecture (CANONICAL DEFINITION)

**CRITICAL: Two completely independent orchestrators exist. They have different jobs. Both are required.**

**Use this naming everywhere in code, docs, and agent prompts.**

#### 1. **Conversational Orchestrator** (OpenAI Realtime) ← DECIDES WHAT TO DO
**Job:** Understand speech, reason about what the user needs, decide WHICH TOOL to call.

OpenAI Realtime handles:
- Voice I/O (WebRTC audio/video)
- Natural language reasoning (LLM decides intent)
- Tool discovery via MCP (auto-discovers Bridge tools)
- Async function calling (speaks "Let me check" while waiting)
- Sequential tool chaining (calls tool A, reasons, calls tool B)
- Response generation (narrates results back to crew)

**Technology:** `gpt-realtime-mini` (GA) with `@openai/agents-realtime` SDK. This is a **permanent lock-in dependency** — no fallback orchestrator documented.

#### 2. **Execution Orchestrator** (Bridge) ← DECIDES HOW TO DO IT SAFELY
**Job:** Validate that the tool call is safe, execute it with proper controls, handle failures gracefully.

Bridge enforces:
- Authentication + authorization (is this user allowed to call this tool?)
- Idempotency (safe to retry if network fails)
- Workflow state machines (proposed → validated → approved → executing)
- Approval tiers (read-only=auto, composite=human, financial=explicit)
- Outbox pattern (durable queue for external calls)
- Retry + compensation (exponential backoff, rollback on failure)
- Cost tracking + budget gates (stop if spend exceeds limits)

**Technology:** Fastify server on Bridge. No custom orchestrator — existing v3.17 infrastructure reused + extended with execution control layer.

**CRITICAL INSIGHT (use in all agent prompts):**
- **Conversational Orchestrator (OpenAI Realtime):** Decides WHAT to do (understands user intent, chooses tool)
- **Execution Orchestrator (Bridge):** Decides HOW to do it safely (validates, approves, executes, handles failure)
- **Both must exist.** One without the other = broken system.
- **OpenAI Realtime is NOT a full orchestrator.** It's the conversation layer only.
- **Bridge is NOT a conversation engine.** It's the safety + execution layer only.

Agents building either system must understand this separation and never blur the boundaries.

### Architecture Diagram

```
PHONE (thin client — just WebRTC + camera)
  │
  │ WebRTC audio + camera frames (via addImage)
  │
  ▼
┌──────────────────────────────────────────────────────────────┐
│               OPENAI REALTIME (GA)                            │
│           gpt-realtime-mini / gpt-realtime-full               │
│                                                               │
│  Voice I/O ← WebRTC (same as today)                          │
│  Orchestration ← model decides which tools to call            │
│  Async tools ← speaks while waiting for results               │
│  Tool chaining ← calls A, gets result, calls B if needed      │
│  MCP client ← connects to BB's MCP server on Bridge           │
│  Tracing ← built-in debugging + audit                         │
└───────────────┬──────────────────────────────────────────────┘
                │ MCP protocol (tool discovery + execution)
                ▼
┌──────────────────────────────────────────────────────────────┐
│               BB BRIDGE — MCP TOOL SERVER                     │
│                                                               │
│  ┌──────────────┐  ┌───────────────┐  ┌───────────────────┐ │
│  │ Vision Agent  │  │ Knowledge     │  │ Search Agent      │ │
│  │              │  │ Agent (RAG)   │  │                   │ │
│  │ Claude 4.6   │  │ pgvector +    │  │ SerpAPI /         │ │
│  │ GPT-4o       │  │ hybrid BM25 + │  │ Perplexity        │ │
│  │ Gemini Flash │  │ Claude        │  │                   │ │
│  │ (config)     │  │ (config)      │  │ (config)          │ │
│  └──────────────┘  └───────────────┘  └───────────────────┘ │
│                                                               │
│  ┌──────────────┐  ┌───────────────┐  ┌───────────────────┐ │
│  │ Financial    │  │ Action Agent  │  │ Logging Agent     │ │
│  │ Agent        │  │               │  │                   │ │
│  │ QBO/QBT →    │  │ CalExp5 API / │  │ Receipts / tools /│ │
│  │ SQL query +  │  │ Drive / Email │  │ vehicles / audit  │ │
│  │ summarize    │  │ / Estimates   │  │ (no model needed) │ │
│  └──────────────┘  └───────────────┘  └───────────────────┘ │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │                SHARED INFRASTRUCTURE                    │  │
│  │  Neon Postgres: sessions, transcripts, crew memory     │  │
│  │  pgvector: RAG knowledge base (hybrid search)          │  │
│  │  Cost tracker: per-agent, per-model, per-session       │  │
│  │  Audit log: every agent call with model/latency/tokens │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

---

## RAG — BB Knowledge Base (Crew Docs Only)

### Two Systems, Not One

This data splits into two fundamentally different retrieval patterns:

**System A: RAG (unstructured documents → vector search)**
- SOPs, safety plans, contracts, codes, manuals, vendor pricing sheets
- Chunked, embedded, stored in pgvector
- Retrieved via hybrid semantic + keyword search
- ~5K-15K chunks for crew knowledge

**System B: SQL Agent (structured data → query + summarize)**
- Receipts, invoices, estimates, timesheets, crew data
- Already in Neon Postgres (cal_receipts, employees, etc.)
- Already accessible via QBO/QBT Bridge endpoints
- Retrieved via SQL queries, summarized by LLM

### RAG Stack

| Component | Choice | Why |
|-----------|--------|-----|
| **Vector DB** | pgvector on Neon (BBInc_1) | Already have it. `CREATE EXTENSION vector`. $0-5/mo. |
| **Embedding** | OpenAI text-embedding-3-small | Already have API key. $0.02/1M tokens. 15K chunks ≈ $0.15. |
| **Search** | Hybrid: pgvector (semantic) + tsvector/BM25 (keyword) | Construction queries need both semantic ("waterproof deck") AND exact keyword ("NEC 210.52"). |
| **Chunking** | 512 tokens, 64 overlap, tables kept whole | 2026 Vectara benchmark winner (69% accuracy). |
| **Table extraction** | Docling (Hugging Face, open-source, free) | 97.9% table accuracy. Handles invoices, estimates, BOQs. |
| **Ingestion** | Google Drive polling every 15-30 min | Simpler than webhooks. Adequate for doc update frequency. |

---

## MCP Tool Registry

Each tool becomes an MCP-compatible server endpoint on Bridge. OpenAI Realtime auto-discovers them. **Tool classification determines governance: read-only tools require no approval; writes route through approval tiers (simple=auto, composite=human, financial=explicit).**

| Tool | Purpose | Model Used | Classification | Latency | Approval Tier |
|------|---------|------------|---|---------|---|
| `vision` | Identify tools/materials/text from camera frame | Claude 4.6 → GPT-4o fallback | **READ-ONLY** | 2-5s | None |
| `knowledge` | Search BB's document knowledge base (RAG) | pgvector search + GPT-4o-mini summarizer | **READ-ONLY** | 1-3s | None |
| `search_web` | Find external resources (videos, manuals, pricing) | SerpAPI → Perplexity fallback | **READ-ONLY** | 1-3s | None |
| `query_data` | Query structured business data (receipts, invoices) | SQL query + GPT-4o-mini summarizer | **READ-ONLY** | 1-2s | None |
| `log_item` | Record a detection (receipt, tool, vehicle, permit) | No model — direct DB write | **SIMPLE WRITE** | <500ms | Auto (idempotent) |
| `deliver_report` | Generate long-form report (notification/email/PDF) | Any model for formatting | **SIMPLE WRITE** | 1-2s | Auto (cached result) |
| `calexp_action` | Execute CalExp5 operations (log hours, check PTO, etc.) | No model — API routing | **COMPOSITE** | <1s | Risk-based (crew data=human if >$100) |
| `remember` | Store tribal knowledge from voice | Embedding + upsert to pgvector | **SIMPLE WRITE** | <1s | Auto (idempotent upsert) |

---

## Unified Role / Tool / Approval Matrix (CANONICAL — single source of truth)

**All agents implementing auth, tool routing, or approval logic MUST use this table. Do not derive approval logic from any other section.**

| Tool | Admin | Lead | Crew | Approval Tier | $ Threshold | Auto-Execute? | Notes |
|------|-------|------|------|---------------|-------------|--------------|-------|
| `vision` | ✓ | ✓ | ✓ | None (read) | — | Yes | Camera identify only — no writes |
| `knowledge` | ✓ | ✓ | ✓ | None (read) | — | Yes | RAG lookup only — no writes |
| `search_web` | ✓ | ✓ | ✓ | None (read) | — | Yes | SerpAPI proxy — no writes |
| `query_data` | ✓ | ✓ | Own data | None (read) | — | Yes | Crew sees own timesheets only; lead sees team; admin sees all |
| `log_item` | ✓ | ✓ | ✓ | Auto (simple write) | — | Yes, idempotent | Same key = no duplicate |
| `deliver_report` | ✓ | ✓ | ✓ | Auto (simple write) | — | Yes, cached | Queued via outbox pattern |
| `calexp_action` | ✓ | ✓ (own team) | Own data | Human if >$100 | $100 | No if >$100 | CalExp5 PIN scope enforced |
| `calexp_action` (financial write) | ✓ | Explicit approval | ✗ | Explicit (lead+admin) | Any $ | No | QBO write, invoice create/edit |
| `remember` | ✓ | ✓ | ✓ | Auto (simple write) | — | Yes, idempotent | Upsert to bb_knowledge_chunks |

**Role hierarchy:** admin > lead > crew. Role is set at session open from CalExp5 PIN, immutable for session duration.

**$ threshold rule:** Any `calexp_action` that creates, modifies, or deletes a financial record >$100 value requires explicit lead+admin approval before execution. Bridge checks this pre-execution.

**Approval SLAs:**
- Auto: <500ms
- Human (composite): escalate via Slack, max 1h wait → safe default (deny write)
- Explicit (financial): escalate via Slack to lead+admin, max 5min wait → deny write

**Enforcement point:** Bridge validates role + approval tier on EVERY tool call, before execution. OpenAI Realtime proposes, Bridge gates.

---

## Execution Control Backbone (Required for Crew Operations)

**The critical decision:** Bridge governs write and composite actions. LLM proposes, Bridge validates and executes.

| Component | Pattern | Purpose |
|-----------|---------|---------|
| **Idempotency** | Postgres-backed keys, 24hr expiry | All write tools are idempotent. Same request 3× = 1 row. |
| **Workflow states** | proposed → validated → approved → queued → executing → completed/failed/compensated | All multi-step actions follow state machine. |
| **Approval tiers** | Risk-based: read (none) / simple write (auto) / composite (human) / financial (explicit) | Write tools routed by risk profile. |
| **Outbox pattern** | Durable queue in Neon, async relay to external systems | No orphaned notifications or webhooks. |
| **Advisory locks** | `SELECT ... FOR UPDATE` on conflict check | Safe concurrent writes without blocking. |
| **Retry + Compensation** | Exponential backoff, bounded retries, compensation chain | Graceful degradation. Crew sees "pending" not "error." |

### Retry / Backoff / Timeout Constants (Canonical — implement exactly)

**Per-tool timeouts (Bridge enforces, hard cutoff):**

| Tool | Timeout | On Timeout |
|------|---------|-----------|
| `vision` | 8s | Return error, trigger fallback (text description) |
| `knowledge` (RAG) | 5s | Return error, fall back to BM25-only search (3s limit) |
| `search_web` | 6s | Return error, use RAG knowledge only |
| `query_data` | 10s | Return error, serve cached result (24h max) |
| `log_item` | 3s | Queue to IndexedDB, retry on reconnect |
| `deliver_report` | 5s | Queue via outbox, guaranteed delivery within 24h |
| `calexp_action` | 5s | Transition workflow to `failed`, trigger compensation |
| `remember` | 3s | Queue locally, sync on next session |
| **MCP server total** | 15s | Bridge returns 504, Realtime offers text fallback |

**Retry schedule (exponential backoff):**

```
Attempt 1: immediate
Attempt 2: 1s delay
Attempt 3: 2s delay
Attempt 4: 4s delay
Max attempts: 4 (then fail permanently for this request)
Max total wait: ~7s before giving up
```

**Outbox relay retry (durable, async — not real-time):**
```
Attempt 1: immediate
Attempt 2: 30s delay
Attempt 3: 5 min delay
Attempt 4: 30 min delay
Attempt 5: 2 hour delay
Max attempts: 5 over 24h window
After 24h: alert ops, mark as dead-letter, require manual resolution
```

**Idempotency key TTL:** 24 hours from first use. After expiry, same request generates new workflow_id (not a duplicate).

**Session inactivity timeout:** 30 minutes from last voice input. No grace period — session closes, transcript flushed.

**Cost circuit breaker:** If per-session cost reaches $1.80 (90% of $2.00 limit), Bridge warns crew via Buddy voice. At $2.00, Bridge stops executing new tool calls for that session (existing in-flight calls complete).

---

## Decision Log

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Orchestrator | OpenAI Realtime (not custom) | Native voice + MCP + async tools. Zero UX penalty. **This is a permanent lock-in dependency — no fallback orchestrator documented.** |
| Voice model | Mini (production), Full (evaluation) | Mini proven in v3.17. Test Full for accuracy before 3x cost. |
| Voice model fallback | None documented | **RISK: If OpenAI Realtime becomes unavailable, no plan B exists. Consider future: Gemini Live, Claude Realtime (TBD 2027).** |
| Vector DB | pgvector on Neon | Already running. One SQL command. $0 additional. Hybrid search. |
| Embedding | OpenAI text-embedding-3-small | Already have key. Cheapest. $0.15 for entire corpus. |
| RAG pattern | Hybrid search (vector + BM25) | Construction needs semantic AND exact keyword retrieval. |
| Structured data | SQL Agent (not RAG) | Receipts, invoices are already in Neon. Query, don't embed. |
| Drive sync | Polling every 15-30 min | Simpler than webhooks. Docs don't change per-minute. |
| API keys | Server-side only | Security. Model flexibility. ~100-200ms acceptable in "thinking" pause. |
| Agent protocol | MCP (native OpenAI support) | Standard. Auto-discovered. Testable independently. |
| Build tool | Claude Code (not Codex/GPT) | See `BB_BUDDY_TRACK_0_DELIVERY.md` for implementation strategy. |

---

## Cost Projections

### Current (v3.17)

| Component | Per 5-min Session | Monthly (10 crew x 4/day) |
|-----------|-------------------|---------------------------|
| OpenAI mini audio | $0.50 | $440 |
| Claude ask_expert (~3/session) | $0.05 | $44 |
| Claude compaction (~2/session) | $0.03 | $26 |
| Neon | ~$0 | $19 |
| **Total** | **~$0.58** | **~$529** |

### V2 Architecture (estimated, crew-only)

| Component | Per 5-min Session | Monthly (10 crew x 4/day) |
|-----------|-------------------|---------------------------|
| OpenAI mini audio (unchanged) | $0.50 | $440 |
| MCP vision tool (Claude/GPT-4o) | $0.05 | $44 |
| MCP knowledge tool (RAG query) | $0.01 | $9 |
| MCP data tool (SQL + summarize) | $0.01 | $9 |
| Compaction (unchanged) | $0.03 | $26 |
| pgvector storage | ~$0 | $0-5 |
| Neon (unchanged) | ~$0 | $19 |
| **Execution Control Infrastructure** | **+$0.02** | **~$17** |
|  — Idempotency key lookups + DB writes | — | ~$8 |
|  — Workflow state machine tracking | — | ~$5 |
|  — Approval queue operations | — | ~$2 |
|  — Advisory lock waits (concurrent safety) | — | ~$1 |
|  — Retry + compensation chain logging | — | ~$1 |
| **Observability Infrastructure** | **+$0.01** | **~$10** |
|  — OpenTelemetry tracing | — | ~$3 |
|  — Pino structured logging (storage) | — | ~$2 |
|  — Langfuse LLM-specific tracing (Railway) | — | ~$5 |
| **Total** | **~$0.63** | **~$579** |

**Net cost increase: ~$50/month** from v3.17 (vs. earlier projection of $23/month). Visibility into execution control + observability infrastructure now explicit. Voice model (OpenAI Realtime) remains the dominant cost.

---

## Critical Dependencies & Risk Factors

### OpenAI Realtime Availability
- **Dependency:** Permanent lock-in (no alternative orchestrator documented)
- **Risk:** API outage, pricing change, EOL
- **Mitigation:** Monitor OpenAI status page; fallback plan (Gemini Live eval for 2027) deferred to Track B6
- **Timeline:** Must evaluate alternatives if Realtime unavailable >24 hours

### pgvector Scaling Limits
- **Current spec:** ~5K-15K chunks for crew knowledge ($0-5/mo on Neon)
- **Scaling decision point:** If chunks exceed 100K, evaluate Pinecone/Weaviate migration
- **Crew-to-homeowner transition:** B1-B7 multi-tenant RAG will require separate vector DB (tenant isolation per `tenant_id`)
- **Timeline:** Scaling decision needed by end of Q2 2026 (A2 completion)

### Docling PDF Table Extraction Dependency
- **Dependency:** Hugging Face open-source library (97.9% accuracy)
- **Risk:** Incompatibility, platform EOL, Python version drift
- **Fallback:** Native PDF text parsing (lower table accuracy)
- **Current risk level:** Medium (used only during A2 crew doc ingestion, not critical path)

### Assistants API Sunset (Mid-2026)
- **Current status:** v3.17 may have Assistants API dependencies
- **Action required:** A0 must audit v3.17 for Assistants API usage and migrate to Agents SDK before sunset
- **Gate:** A0 success criteria includes "Assistants API deprecated" checkpoint
- **Timeline:** Must complete by June 2026

---

## Operational Semantics (Resolved)

| Question | Answer | Defined In |
|----------|--------|-----------|
| Session lifecycle | Opens on first voice input. Closes on 30-min timeout or explicit end. | A1 DONE WHEN (Crew Platform) |
| Transcript retention | 30-day rolling window. Audio archived 2 years. | Data Lifecycle Policy (below) |
| Cost aggregation | Per-tool invocation → rolled up per-session → per-crew-day. Failed/retried calls included. | A1 DONE WHEN (Crew Platform) |
| Approval SLAs | Read: none. Simple write: auto (<500ms). Composite: human, 1h max → deny write. Financial: explicit lead+admin, **5min max** → deny write. | Unified Role/Tool/Approval Matrix (above) |
| Approver identity | Admin > Lead > Crew. Escalate to Sam if composite approver unavailable >1h, or financial approver unavailable >5min. | Security Model v1 (below) |

---

## Phase Completion Criteria (Definition of Done)

**All phases must meet these binary criteria before advancing. "Good enough" = FAIL.**

### Track 0 Completion Criteria

| Stage | DONE WHEN |
|-------|-----------|
| **T0.1** | Harness core runs autonomously overnight. All node test suites pass 3-zero. Cost tracking accurate within ±10%. No manual intervention required. |
| **T0.2** | Observer UI shows live progress, pause/skip/rerun controls work, cost metrics displayed. Can connect via WebSocket from desktop without errors. |
| **T0.3** | Relational seeder generates golden-10 fixtures in <30s. Validator confirms all constraints satisfied. Snapshots reproducible. |
| **T0.4** | SDV model trains on golden fixtures. Generated 200-property portfolio statistically equivalent to input (KL divergence <0.1 per feature). Assets generated with realistic degradation. |
| **T0.5** | Voice sessions: 30/30 passes with >80% accuracy on persona-scripted questions. RAG: 50 golden queries pass faithfulness >0.7, zero hallucinated citations. Latency <3s p95. Ingestion stable for 48h. |
| **T0.6** | BrowserStack passes on iOS Safari + Android Chrome + Firefox. k6 load test: 200 VUs sustained without 5xx errors. Trend dashboard shows 4+ weeks of history. All 3 core projects (Buddy, Bridge, CalExp5) have passing test suites. |
| **T0.7** | Fix agent detects failures, proposes patches, all patches human-reviewed before merge. MTTR reduced by 80%. No autonomous merges to main. |

### Track A Phase Completion Criteria

| Phase | DONE WHEN |
|-------|-----------|
| **A0** | Agents SDK installed, RealtimeAgent working on desktop + iOS Safari. Connection time + first-audio latency measured and documented. v3.17 Assistants API audit complete + migration plan written. **A0 COMPLETE = evaluation finished AND path chosen. Path ∈ {SDK, raw WebRTC}. Either outcome satisfies the A1 gate.** |
| **A1** | MCP server deployed on Bridge, auto-discovered by OpenAI Realtime. All 8 tools live with working primary/fallback models. Anthropic API key removed from client. Tool latency <5s p95. Cost tracking accurate ±10%. Session lifecycle defined (30-min timeout, 30-day retention). Approval tiers defined per tool per role. T0.1 harness validates A1 tools passing 3-zero. |
| **A2** | pgvector enabled on Neon. `bb_knowledge_chunks` table created (crew-only, no tenant_id). Google Drive → extraction → chunking → embedding pipeline operational. 50 golden queries pass RAG faithfulness >0.7. Zero hallucinated citations. Latency <3s p95. `remember` tool stores + upserts successfully. Ingestion pipeline stable 48h. |
| **A3** | **Build complete:** Financial queries return correct summaries (invoice counts, expense totals verified against QBO). CalExp5 actions (log hours, check PTO) execute + rollback correctly. Write workflows respect approval tiers (auto for simple, human for composite, explicit for financial). Idempotency proven (same request 3× = 1 row). All crew data remaining single-tenant (no tenant_id in schema). T0.6 scale testing passes. **Production validation (Sam's sign-off):** 48h real crew shadow (5+ sessions/day), cost <$20/day, zero critical errors, satisfaction >4/5. |

---

## Failure Modes & Degraded Mode Behaviors

**Principle:** System never halts. It always offers a useful path forward, even if degraded.

### MCP Tool Failures

| Failure | Behavior | Fallback |
|---------|----------|----------|
| Vision tool unavailable | "I can't see the image right now. Describe it?" | Text description from user, retry later |
| Knowledge (RAG) down | "I don't have access to that reference. Try asking me directly." | LLM memory + web search (slower, less accurate) |
| Search web unavailable | "I can't browse external sources right now." | RAG knowledge base only (no web context) |
| Query data (SQL) fails | "I can't access the business data right now. Try later." | Cached summary from last successful query (max 24h old) |
| Log item fails | Queue locally on device, retry on next connection | IndexedDB backup queue (automatic sync) |
| Deliver report fails | "Report queued. I'll try sending it again." | Outbox pattern (guaranteed eventual delivery, max 24h) |
| CalExp action fails | "That action is pending approval. Checking..." | Approval workflow + manual crew fallback |
| Remember (RAG store) fails | "Got it, saved locally. Will sync when ready." | Local cache + eventual consistency sync |

### Conversational Orchestrator (OpenAI Realtime) Failures

| Failure | Behavior | Fallback |
|---------|----------|----------|
| OpenAI Realtime unavailable (>1 min) | Offer text interface to same tools (no voice) | Bridge HTTP API direct (same MCP tools, text I/O) |
| Model returns nonsensical response | Validation gate catches it, returns "I didn't understand. Can you rephrase?" | Force user confirmation + retry |
| Tool decision is unsafe (e.g., financial write without approval) | Execution orchestrator blocks it, explains why | Offer alternative safer action (read instead of write) |
| Audio connection drops mid-session | Reconnect automatically, pick up conversation where it left off | Session state persisted, transcript available for context |

### Execution Orchestrator (Bridge) Failures

| Failure | Behavior | Fallback |
|---------|----------|----------|
| Bridge unavailable (>2 min) | Client reads cached tool results (last 24h valid) | Stale data better than no data. Warn crew. |
| Idempotency key lookup fails | Abort write (never execute without dedup guarantee) | Manual crew review + admin override only |
| Approval queue jammed | Composite: escalate to Sam after 1h, deny write. Financial: escalate to Sam after 5min, deny write. | OOB Slack alert → safe default (deny write) |
| Outbox relay failing (3+ consecutive failures) | Retry with exponential backoff, bounded to 24h max | Pager alert (ops team, not crew). Manual investigation. |
| Cost budget exceeded | Pause new sessions, drain queue, alert Sam | Crew can override for critical actions (human approval only) |

### RAG-Specific Failures

| Failure | Behavior | Fallback |
|---------|----------|----------|
| Embedding service down | Use BM25 keyword-only search (no semantic) | Accurate but less contextual |
| pgvector unavailable | Fallback to BM25 tsvector (PostgreSQL native) | Built-in, guaranteed to exist |
| RAG returns hallucinated source | Validation catches it: "Citation not found in knowledge base" | LLM admits uncertainty: "I'm not confident in that source" |
| Knowledge base stale (ingestion cron missed >24h) | Alert ops, serve stale knowledge with timestamp | "This knowledge is from {date}. May be outdated." |

### Production Response Expectations

**Crew sees:** System is available and useful even when degraded. Transparency about what's degraded. "Try again later" only after exhausting fallbacks.

**Ops sees:** Automated alerts on all failures. Degraded mode metrics tracked separately. Cost impact visible per failure type.

**Architect sees:** Fallback chains defined. No single point of failure. Recovery path documented for each failure.

---

## Cost Gates & Budget Enforcement

**Cost tracking is not advisory — it is a hard brake on execution.**

| Budget Level | Limit | Action on Exceed | Owner | Rollback? |
|---|---|---|---|---|
| **Per-session** | $2.00 | Warn crew, offer degraded mode (read-only, no complex operations) | Client-side (pre-flight check) | Yes — don't execute the expensive call |
| **Per-crew-day** | $20.00 | Pause new sessions, drain approval queue | Bridge outbox (SLA enforced) | No — existing sessions complete |
| **Per-nightly-run** (T0.5+) | $50.00 | Abort harness, alert Sam, stop all test waves | Harness orchestrator | Yes — revert to last stable run |
| **Per-month (projected)** | $600 (budget) vs $579 (projected) | Monthly review, forecast drift analysis | Sam (architect approval) | No — adjust next month's allocation |

**If cost exceeds budget:**
1. ✓ Automatic Slack alert (ops channel, not crew)
2. ✓ Session/run continues (crew not interrupted)
3. ✓ Root cause analysis required before next run
4. ✓ No "just pay more" override — find the leak first
5. ✓ Financial writes require explicit cost pre-approval from crew lead

---

## Data Lifecycle Policy (Track A, Crew-Only)

| Data Type | Retention | Re-embed Trigger | Versioning | Deletion Policy | Notes |
|---|---|---|---|---|---|
| **Raw PDFs** (Google Drive) | Permanent (crew reference) | N/A (not embedded) | File revision + timestamp | Manual crew delete only | Crew docs stay forever for reference |
| **Transcripts** | 30 days rolling window | Daily (recency bias, latest context) | session_id + sequence | Auto-purge after 30 days | Audio archived separately (2yr retention for audit) |
| **Receipts** | Permanent (financial audit) | Monthly (sync with QBO source truth) | document_id + qbo_tx_id | Permanent | Keep forever for tax + audit compliance |
| **RAG chunks** (embeddings) | Until doc removal (manual) | On doc update OR every 7 days (batch re-embed) | chunk_id + doc_version | Delete with source doc | Drift risk: stale chunk >30 days = wrong answer |
| **Knowledge embeddings** | Match chunk retention | When chunk updates | embedding_id + hash(content) | Delete with chunk | Vector cache expires same as source doc |
| **Session cache** | 24h | N/A (expires automatically) | Per-session, no versions | Auto-purge after 24h | Cost + latency optimization only |
| **Cost records** | 7 years | N/A | Per-transaction, immutable | Never delete | Tax + audit trail compliance |
| **Approval audit log** | 2 years | N/A | Per-approval + timestamp | Auto-purge after 2 years | Legal compliance for writes |

**Stale knowledge risk:** RAG chunks not re-embedded for >30 days = potential drift. **Ops cron must validate freshness weekly.** Stale chunk alert if age >30 days.

---

## Security Model v1 (Track A Commitment)

**All writes require three sequential gates:**

### Gate 1: Authentication
**Question:** Who is the user?
- CalExp5 PIN → crew_id + role lookup
- Session state persisted → crew context available to all tool calls
- Failure: abort with "Not authenticated"

### Gate 2: Authorization
**Question:** Is this user's role allowed to call this tool?
- Role matrix: admin > lead > crew
- Tool matrix: some tools restricted to admin/lead only
- Failure: abort with "Not authorized for this tool"

### Gate 3: Approval Tier Evaluation
**Question:** Does this specific action require human approval?
- Risk-based routing: read (none) / simple_write (auto) / composite (human) / financial (explicit)
- Amount-based: actions >$100 require explicit lead approval
- Type-based: writes to QBO always require lead signature

### Execution Rule
**No direct LLM write access.** LLM proposes. Bridge validates + approves + executes.

| Action Type | Auth | Role Check | Approval | SLA | Examples |
|---|---|---|---|---|---|
| **READ** | Required | Required | None | <100ms | Ask about timesheets, get reports |
| **SIMPLE WRITE** | Required | Required | Auto (idempotent) | <500ms | Log receipt, save tribal knowledge |
| **COMPOSITE** | Required | Required | Varies (human if risk high) | <1min | Approve timesheet, modify schedule |
| **FINANCIAL** | Required | Required | **Explicit (lead + admin)** | <5min | Write-back to QBO, create invoice |

---

## Track A Module Ownership (Agent Scope Boundaries)

**Each agent building a Track A phase owns exactly the files below. No agent touches another phase's files without explicit cross-phase approval.**

| Agent / Phase | Owns | Cannot Touch |
|--------------|------|-------------|
| **A0 agent** | `bb-scan-openai.html`, client JS bundle, SDK integration code | Bridge source, Neon schema, MCP routes |
| **A1 agent** | `BB_Micro_Bridge/src/mcp/` (all files), `src/mcp/server.js`, tool handler files | RAG modules (`src/rag/`), operations routes (`src/operations/`), client HTML |
| **A2 agent** | `BB_Micro_Bridge/src/rag/` (all files), `bb_knowledge_chunks` schema migration, ingestion pipeline, embedder | MCP tool handlers (except `knowledge` tool stub → full impl), operations routes, client HTML |
| **A3 agent** | `BB_Micro_Bridge/src/operations/` (all files), workflow state machine, approval queue, outbox, QBO/QBT routing | RAG modules, MCP server registration (only adds routes, doesn't rewrite server.js) |
| **Client agent** | `bb-scan-openai.html`, client-side JS, CSS | All Bridge source, Neon schema |

**Shared files (require both agents to coordinate):**
- `BB_Micro_Bridge/src/mcp/server.js` — A1 creates it, A2/A3 add routes to it. Never rewrite — only append.
- `BB_Micro_Bridge/src/db/schema.ts` — A2 adds `bb_knowledge_chunks`. A3 adds workflow/session tables. Each phase owns its own tables.
- `.env` / Railway env vars — Any phase may add keys for its own services. No phase removes another phase's keys.

**File conflict rule:** If two agents need to edit the same file, the later-phase agent opens a PR against the earlier-phase agent's branch. Sam merges. No direct overwrites.

---

## Track A Frozen Contracts Appendix (Canonical — Agents Must Implement Exactly)

**Purpose:** These contracts are the immutable interface layer for Track A (A0–A3). Agents may implement internals however they choose, but they **must not change these shapes** without explicit approval from Sam.

**Scope:** MCP server request/response handling, Bridge execution orchestration, approval workflows, session/transcript persistence, cost tracking, RAG citations, query_data normalized output.

**Non-goals:** These are not DB migration specs for Track B, not multi-tenant schemas, and not homeowner-facing contracts. Track A remains single-tenant only — no `tenant_id` may appear in any Track A contract.

**Override rule:** These contracts override prose if prose is ambiguous. Agents may add internal fields, but may not remove or rename canonical fields. Public API surfaces must serialize exactly to these names.

**Companion files:** Machine-readable copies live at `BB_Micro_Bridge/src/contracts/track-a/` (TypeScript). These docs and those files must stay in sync.

---

### Contract 1: MCP Tool Request Envelope

```ts
export type McpToolName =
  | 'vision'
  | 'knowledge'
  | 'search_web'
  | 'query_data'
  | 'log_item'
  | 'deliver_report'
  | 'calexp_action'
  | 'remember';

export interface McpToolRequestEnvelope {
  request_id: string;              // uuid
  session_id: string;              // uuid
  tool_name: McpToolName;
  crew_id: string;                 // authenticated CalExp5 crew/user id
  crew_role: 'admin' | 'lead' | 'crew';
  timestamp: string;               // ISO 8601 UTC
  idempotency_key?: string;        // REQUIRED for all writes, forbidden for read-only calls
  input: Record<string, unknown>;  // tool-specific payload (see per-tool contracts below)
  trace_id: string;                // distributed tracing id
  client_context: {
    platform: 'ios_safari' | 'android_chrome' | 'desktop_chrome' | 'desktop_firefox' | 'unknown';
    app_version: string;
    network_state?: 'online' | 'offline' | 'degraded';
  };
}
```

**Rules:**
- `idempotency_key` is **required** for `log_item`, `deliver_report`, `calexp_action`, `remember`
- `idempotency_key` **must not** be sent for `vision`, `knowledge`, `search_web`, `query_data`
- `crew_role` is immutable for the session once authenticated
- `tool_name` must match the canonical tool registry exactly

---

### Contract 2: MCP Tool Success Response Envelope

```ts
export interface McpToolSuccessEnvelope {
  request_id: string;                // echoes request
  session_id: string;
  tool_name: McpToolName;
  status: 'success';
  trace_id: string;
  duration_ms: number;
  cost_cents: number;                // 0 allowed for non-LLM tools
  approval: {
    tier: 'none' | 'simple_write' | 'composite' | 'financial';
    required: boolean;
    status: 'not_needed' | 'auto_approved' | 'human_approved';
    approver_id?: string;            // present only if human approved
  };
  cache: {
    hit: boolean;
    source?: 'session_cache' | 'indexeddb' | 'bridge_cache' | 'none';
    staleness_seconds?: number;      // present if cached
  };
  output: Record<string, unknown>;   // normalized tool result (see per-tool contracts)
  warnings: string[];                // non-fatal warnings only
  timestamp: string;                 // ISO 8601 UTC
}
```

**Rules:**
- `status: 'success'` only if tool completed OR returned an accepted degraded-mode fallback
- Cached fallback results still return `success` if valid and disclosed in `cache`
- `cost_cents` must reflect the actual executed path, not the ideal path

---

### Contract 3: MCP Tool Error Envelope

```ts
export type McpErrorCode =
  | 'NOT_AUTHENTICATED'
  | 'NOT_AUTHORIZED'
  | 'APPROVAL_REQUIRED'
  | 'APPROVAL_TIMEOUT'
  | 'TOOL_TIMEOUT'
  | 'UPSTREAM_UNAVAILABLE'
  | 'INVALID_INPUT'
  | 'IDEMPOTENCY_LOOKUP_FAILED'
  | 'WORKFLOW_FAILED'
  | 'COST_LIMIT_REACHED'
  | 'RATE_LIMITED'
  | 'UNKNOWN_ERROR';

export interface McpToolErrorEnvelope {
  request_id: string;
  session_id: string;
  tool_name: McpToolName;
  status: 'error';
  trace_id: string;
  error: {
    code: McpErrorCode;
    message: string;                 // user-safe message (no stack traces)
    retryable: boolean;
    degraded_mode_available: boolean;
    degraded_mode_reason?: string;
  };
  fallback?: {
    offered: boolean;
    type?: 'text_only' | 'cached_result' | 'bm25_only' | 'manual_review' | 'queue_for_retry';
    expires_at?: string;             // ISO 8601 if relevant
  };
  duration_ms: number;
  cost_cents: number;
  timestamp: string;
}
```

**Rules:**
- No raw stack traces in this envelope
- If degraded mode exists, `degraded_mode_available: true` AND `fallback.offered: true`
- If approval is **pending** (not denied), do not use this envelope — use Contract 4/5

---

### Contract 4: Approval Request Object

```ts
export interface ApprovalRequest {
  approval_request_id: string;       // uuid
  workflow_id: string;               // links to workflow state machine
  session_id: string;
  tool_name: McpToolName;
  requested_by_crew_id: string;
  requested_by_role: 'admin' | 'lead' | 'crew';
  required_tier: 'composite' | 'financial';
  threshold_amount_usd?: number;     // present for financial or threshold-based actions
  reason: string;                    // concise explanation
  proposed_action_summary: string;   // human-readable summary for approver
  approver_roles_required: Array<'admin' | 'lead'>;
  status: 'pending' | 'approved' | 'rejected' | 'expired';
  created_at: string;
  expires_at: string;                // matches SLA from Unified Role/Tool/Approval Matrix
  trace_id: string;
}
```

**Rules:**
- `required_tier: 'financial'` → `approver_roles_required` must include both `'lead'` and `'admin'`
- `expires_at` must match the SLA from the Unified Role/Tool/Approval Matrix (composite = 1h, financial = 5min)

---

### Contract 5: Approval Decision Object

```ts
export interface ApprovalDecision {
  approval_request_id: string;
  workflow_id: string;
  decision: 'approved' | 'rejected' | 'expired';
  decided_by: Array<{
    approver_id: string;
    approver_role: 'admin' | 'lead';
    decided_at: string;
  }>;
  notes?: string;
  trace_id: string;
}
```

**Rules:**
- Financial approvals require both `lead` and `admin` entries before `decision: 'approved'`
- Expiry resolves to safe default: deny write

---

### Contract 6: Workflow State Object

```ts
export type WorkflowState =
  | 'proposed'
  | 'validated'
  | 'approved'
  | 'queued'
  | 'executing'
  | 'completed'
  | 'failed'
  | 'compensated';

export interface ExecutionWorkflow {
  workflow_id: string;                 // uuid
  request_id: string;
  session_id: string;
  tool_name: McpToolName;
  crew_id: string;
  state: WorkflowState;
  idempotency_key?: string;
  approval_tier: 'none' | 'simple_write' | 'composite' | 'financial';
  external_side_effects: boolean;
  retry_count: number;                 // 0–4 for real-time path
  compensation_required: boolean;
  last_error_code?: McpErrorCode;
  created_at: string;
  updated_at: string;
  trace_id: string;
}
```

**Allowed state transitions only:**
```
proposed → validated → approved → queued → executing → completed
                                                      → failed → compensated
```

**Rules:**
- No skipping states
- `compensated` is terminal
- Read-only tools may use a transient internal workflow, but if persisted must use valid states

---

### Contract 7: Session Record

```ts
export interface BuddySessionRecord {
  session_id: string;                  // uuid
  crew_id: string;
  crew_role: 'admin' | 'lead' | 'crew';
  opened_at: string;
  closed_at?: string;
  close_reason?: 'timeout' | 'explicit_end' | 'disconnect' | 'system_shutdown';
  status: 'open' | 'closed';
  platform: 'ios_safari' | 'android_chrome' | 'desktop_chrome' | 'desktop_firefox' | 'unknown';
  app_version: string;
  transcript_retention_days: 30;
  audio_archive_retention_years: 2;
  total_cost_cents: number;
  total_tool_calls: number;
  trace_id: string;
}
```

**Rules:**
- One role per session, immutable after open
- Session inactivity timeout: 30 minutes from last voice input

---

### Contract 8: Transcript Item Record

```ts
export interface BuddyTranscriptItem {
  transcript_item_id: string;          // uuid
  session_id: string;
  sequence_no: number;                 // strictly increasing within session
  speaker: 'crew' | 'buddy' | 'system' | 'tool';
  content_type: 'text' | 'tool_call' | 'tool_result' | 'system_event';
  content: string;                     // normalized text payload
  tool_name?: McpToolName;
  request_id?: string;
  timestamp: string;
  trace_id: string;
}
```

**Rules:**
- `sequence_no` must be gapless within a session
- Tool invocations and tool results must each get their own transcript items
- No binary/audio blobs in this table

---

### Contract 9: Cost Event Record

```ts
export interface CostEventRecord {
  cost_event_id: string;               // uuid
  session_id: string;
  request_id?: string;
  tool_name?: McpToolName;
  provider: 'openai' | 'anthropic' | 'serpapi' | 'perplexity' | 'neon' | 'browserstack' | 'other';
  model?: string;                      // e.g. 'gpt-realtime-mini', 'claude-sonnet-4-6'
  input_tokens?: number;
  output_tokens?: number;
  cost_cents: number;                  // integer cents, never floating point
  event_type: 'session_audio' | 'tool_call' | 'cache_hit' | 'workflow' | 'observability' | 'test_run';
  timestamp: string;
  trace_id: string;
}
```

**Rules:**
- `cost_cents` is integer cents, never floating point
- Cache hits may log `cost_cents: 0` but must still emit an event if they affect totals or tracing

---

### Contract 10: RAG Citation Object

```ts
export interface RagCitation {
  citation_id: string;                 // uuid
  doc_id: string;
  doc_title: string;
  section?: string;
  chunk_id: string;
  source_type: 'drive' | 'manual' | 'voice' | 'session';
  confidence_score: number;            // 0.0–1.0
  excerpt: string;                     // max 240 chars, paraphrased or short quoted snippet
}
```

**Rules:**
- Every `knowledge` response that claims source-backed facts must return `citations: RagCitation[]`
- Zero hallucinated citations — every returned citation must map to a real stored chunk
- If `chunk_id` cannot be resolved, suppress the citation and Buddy says "I'm not confident in that source"

---

### Contract 11: Knowledge Tool Normalized Output

```ts
export interface KnowledgeToolOutput {
  answer: string;
  citations: RagCitation[];
  retrieval_mode: 'hybrid' | 'bm25_only' | 'cached';
  latency_ms: number;
  degraded: boolean;
  degraded_reason?: string;
}
```

**Rules:**
- `citations` may be empty only if the answer explicitly states uncertainty or lack of source support
- If `retrieval_mode: 'bm25_only'`, `degraded` must be `true`

---

### Contract 12: Query Data Normalized Output

```ts
export interface QueryDataOutput {
  answer: string;
  query_scope: 'own_data' | 'team_data' | 'all_data';
  source_systems: Array<'neon' | 'qbo' | 'qbt' | 'calexp5'>;
  records_examined: number;
  currency?: 'USD';
  aggregates?: Array<{
    label: string;
    value: number;
    unit: 'usd' | 'count' | 'hours';
  }>;
  degraded: boolean;
  degraded_reason?: string;
  cache_age_seconds?: number;
}
```

**Rules:**
- `query_scope` must reflect the caller's role restrictions from the Unified Role/Tool/Approval Matrix
- Aggregates are normalized regardless of the source system's native response format
- RAG tool (`knowledge`) **never** returns this shape. SQL Agent (`query_data`) **never** returns `RagCitation[]`. Separate systems, separate types.

---

### Contract 13: Deliver Report Output

```ts
export interface DeliverReportOutput {
  report_id: string;                   // uuid
  delivery_status: 'queued' | 'sent' | 'failed';
  delivery_channel: 'email' | 'notification' | 'pdf';
  outbox_id?: string;
  degraded: boolean;
  degraded_reason?: string;
}
```

---

### Contract 14: Log Item Output

```ts
export interface LogItemOutput {
  log_id: string;                      // uuid
  deduplicated: boolean;
  persisted: boolean;
  storage_target: 'neon' | 'indexeddb';
  degraded: boolean;
  degraded_reason?: string;
}
```

---

### Contract 15: Remember Output

```ts
export interface RememberOutput {
  knowledge_chunk_id?: string;
  upsert_status: 'inserted' | 'updated' | 'queued_for_sync' | 'failed';
  deduplicated: boolean;
  degraded: boolean;
  degraded_reason?: string;
}
```

---

### RAG Chunk Schema (`bb_knowledge_chunks` table)

```ts
// Neon table definition — A2 creates this migration
{
  chunk_id: string,         // uuid, PK
  doc_id: string,           // uuid, references source document registry
  doc_title: string,        // human-readable document name
  doc_path: string,         // Google Drive path
  doc_version: string,      // Drive revision ID or content hash
  chunk_index: number,      // 0-based position within document
  content: string,          // raw text (512 tokens, 64 overlap)
  embedding: number[],      // 1536-dim from text-embedding-3-small
  tsvector: string,         // PostgreSQL tsvector for BM25
  tenant_id: string,        // ALWAYS 'bb_crew' in Track A. NEVER null.
  source_type: 'drive' | 'manual' | 'voice' | 'session',
  created_at: string,       // ISO 8601
  updated_at: string,
}
```

**RULE: `tenant_id` is ALWAYS `'bb_crew'` in Track A. Multi-tenant RAG is Track B only.**

---

## Open Items: ZERO (Architectural)

All 36+ items from earlier v1.5 versions are resolved. All ChatGPT Tier 1-3 fixes applied. Architecture is ready for Track 0 construction. Operational semantics clarifications (session lifecycle fine-tuning, approval SLA specifics, cost tracker granularity) deferred to phase-specific implementation docs (A0, A1, A3).

---

*Core Architecture v2.0. For delivery plans, see companion documents.*
*Reference docs: BB_Buddy Track 0 Delivery | BB_Buddy Crew Platform | BB_Home Platform Expansion*
