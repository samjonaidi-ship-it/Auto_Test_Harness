# BB Buddy Crew Platform | A0-A3 Phases | v1.6 | 2026-04-02 | BB

**This is the committed crew-only product roadmap.** Homeowner features are in `BB_HOME_PLATFORM_EXPANSION.md` (design-only until A3 production validation).

> **Track A interface contracts are frozen in `BB_BUDDY_CORE_ARCHITECTURE.md` → "Track A Frozen Contracts Appendix" (Contracts 1–15).** Agents must implement those shapes exactly and may not modify them without explicit approval from Sam. Machine-readable TypeScript copies live at `BB_Micro_Bridge/src/contracts/track-a/`.

---

## Execution Boundary Matrix (Crew Platform A0-A3)

| Feature | Phase | Status | Build Now? | Notes |
|---------|-------|--------|-----------|-------|
| **OpenAI Agents SDK** | A0 | Committed | YES | Framework migration test, iOS Safari validation |
| **MCP tool layer** | A1 | Committed | YES | Server-side tool execution, model config |
| **RAG crew docs** | A2 | Committed | YES | SOPs, safety, pricing, manuals → crew knowledge |
| **Tribal knowledge capture** | A2 | Committed | YES | Voice → Buddy → crew memory + RAG |
| **Financial queries** | A3 | Committed | YES | QBO/QBT → SQL agent → crew answers |
| **Crew scheduling** | A3 | Committed | YES | CalExp5 integration, no homeowner features |
| **Crew auth/permissions** | A3 | Committed | YES | CalExp5 PIN → crew roles |
| **Write-back workflows** | A3 | Committed | YES | Idempotency, approvals, state machines |
| **Stripe billing** | — | FORBIDDEN NOW | NO | Future homeowner subscription platform |
| **Jobber scheduling** | — | FORBIDDEN NOW | NO | Future homeowner service platform |
| **Homeowner scheduling** | — | FORBIDDEN NOW | NO | Future homeowner platform |
| **Property intelligence** | — | FORBIDDEN NOW | NO | Future homeowner platform |
| **Customer uploads** | — | FORBIDDEN NOW | NO | Future homeowner platform |

---

## A0 (Phase 0): Agents SDK Evaluation — Effort: Small, Risk: Low

**Timeline:** 1-2 weeks (parallel with T0.1)

**Goal:** Validate that `@openai/agents-realtime` works on mobile browsers (iOS Safari) and doesn't degrade UX.

### What (REQUIRED)

- **REQUIRED:** Assistants API deprecation audit: Check v3.17 for Assistants API dependencies (thread mgmt, file retrieval, assistants endpoint calls). Migrate all to Agents SDK equivalents. Document migration plan.
- **REQUIRED:** Install `@openai/agents-realtime` in project
- **REQUIRED:** Rewrite `bb-scan-openai.html` to use `RealtimeAgent` + `RealtimeSession` + `OpenAIRealtimeWebRTC`
- **REQUIRED:** Keep same 5 tools (hardcoded), define via `tool()` with Zod schemas
- **REQUIRED:** Use SDK events (`audio_start`, `audio_stopped`) instead of manual `buddySpeaking` flag
- **REQUIRED:** Test on iOS Safari, Chrome Android, Chrome desktop (all three, not optional)
- **REQUIRED:** Measure + document: connection time, first-audio latency, tool call success rate, bundle size delta

**NOT CHANGING (FORBIDDEN in A0):**
- ❌ Tool execution moves to server (stays client-side in A0)
- ❌ MCP integration (A1 only)
- ❌ RAG knowledge base (A2 only)
- ❌ Multi-model strategy (A1 only)

**OPTIONAL (nice-to-have, do only if time allows):**
- Bundle size optimization (>15% bloat is acceptable if unavoidable)

**Success criteria:**
- **UX parity:** p95 first-audio latency ≤ v3.17 baseline (record v3.17 baseline during A0 test run). Task completion time (voice query → first spoken word of answer) ≤ v3.17 ±10%
- SDK works on iOS Safari (zero console errors, zero WebRTC ICE failures during test session)
- Bundle size delta measured and documented (>15% bloat requires justification note)
- Assistants API audit completed + migration plan written

**A0 is DONE when:**
- ✓ Agents SDK installed and basic RealtimeAgent working on desktop
- ✓ iOS Safari testing completed
- ✓ Connection time + first-audio latency measured and documented
- ✓ Bundle size increase measured (>15% bloat must be documented with justification)
- ✓ v3.17 Assistants API audit completed + migration plan written
- ✓ Decision recorded: SDK viable OR raw WebRTC (either outcome = A0 COMPLETE)
- ✓ Code merged to main, ready for A1 to begin

**Failure fallback:** If SDK fails on iOS Safari, A0 still COMPLETES — record outcome as "raw WebRTC path". A1 proceeds using raw WebRTC instead of SDK. A0 COMPLETE means "evaluation finished and path chosen", not "SDK succeeded". The A1 gate requirement "A0 COMPLETE" is satisfied by either outcome.

**Dependencies:** None. Can start immediately.

**Out of scope:** Server-side tools, RAG, model selection, load testing.

---

## A1 (Phase 1): MCP Tool Layer — Effort: Medium, Risk: Medium

**Timeline:** 2-3 weeks (start after T0.1 complete)

**Goal:** Move all AI tool execution from browser to Bridge. API keys stay server-side. Model selection becomes config-driven.

### What

- Build MCP server on Bridge (`src/mcp/server.js`) with these tools as endpoints:
  - `vision` (Claude 4.6 vision → identify from camera)
  - `knowledge` (placeholder, will be replaced by RAG in A2)
  - `search_web` (SerpAPI → external resources)
  - `query_data` (SQL query → structured data, **crew data only**)
  - `log_item` (receipt, tool, vehicle, permit)
  - `deliver_report` (long-form report generation)
  - `calexp_action` (CalExp5 operations)
  - `remember` (tribal knowledge capture)

- Each tool wraps AI provider with **primary/fallback config** (changeable without code):
  - Vision: Claude 4.6 → GPT-4o
  - Search: SerpAPI → Perplexity
  - Knowledge: GPT-4o-mini → Claude Haiku
  - Data summarization: GPT-4o-mini → Claude Haiku

- OpenAI Realtime connects to Bridge MCP server via `hostedMcpTool()`

- HTML client simplified: **only WebRTC + camera + UI**. No Claude/SerpAPI calls.

- Per-tool cost tracking: model, tokens, latency, cost → `cal_scan_transcripts`

- **Remove** Anthropic API key from client entirely

### Success criteria

- MCP server auto-discovered by OpenAI Realtime (no manual tool registration)
- Tool execution latency: same or better than v3.17 direct-to-API
- All crew tools work with config-driven model selection
- Cost tracking accurate within ±10%
- **Session lifecycle locked down:** Definition of session boundaries + transcript retention policy
- **Approval tiers fully specified:** Per tool, per crew role, with response SLAs

### A1 is DONE when:

- ✓ MCP server deployed on Bridge + auto-discovered by Realtime (no manual registration)
- ✓ All 8 tools live on Bridge with working primary/fallback config
- ✓ Anthropic API key removed from client entirely
- ✓ Tool execution latency verified: <5s p95 per tool (same or better vs v3.17)
- ✓ Cost tracking schema locked + accurate within ±10%
- ✓ Session lifecycle defined: on first input → close on 30min timeout, 30-day transcript retention
- ✓ Approval tiers defined per tool per role: read (none), simple_write (auto), composite (1 human), financial (explicit lead+admin)
- ✓ T0.1 harness validates A1 tools passing 3-zero
- ✓ Code merged to main, ready for A2

### Execution control (Governance)

**Required for A1 + A3:**
- **Idempotency:** Postgres-backed fingerprint keys, 24-hour expiry. All write tools are idempotent.
- **Workflow states:** proposed → validated → approved → queued → executing → completed/failed/compensated
- **Approval tiers:** Risk-based routing (read=none, simple_write=auto, composite=human, financial=explicit)
- **Outbox pattern:** Durable queue in Neon, async relay to external systems
- **Advisory locks:** `SELECT ... FOR UPDATE` for safe concurrent writes
- **Retry + Compensation:** Exponential backoff, bounded retries, compensation chain

### Dependencies

- A0 COMPLETE (SDK evaluation finished, path chosen: SDK or raw WebRTC)
- T0.1 COMPLETE (harness must be ready to validate MCP tools)

### Out of scope

- RAG (that's A2)
- Load testing (that's T0.5)
- Homeowner features
- Model abstraction beyond primary/fallback

---

## A2 (Phase 2): BB Knowledge System (Crew Docs Only) — Effort: Medium, Risk: Low

**Timeline:** 3-4 weeks (start after A1 complete)

**Goal:** BB's internal document knowledge available to crew via voice. **Crew docs only — no homeowner uploads.**

### What

- Enable pgvector on Neon BBInc_1: `CREATE EXTENSION vector`
- Create `bb_knowledge_chunks` table (crew-only, no tenant_id yet)
- Build ingestion pipeline: Google Drive `BB Knowledge Base/Crew/` folder → poll → extract text → chunk → embed → upsert
- New MCP tool: `knowledge` — hybrid vector + BM25 search → summarize → return to Buddy
- New MCP tool: `remember` — voice-captured tribal knowledge → embed → store
- Ingest Tier 1 (crew only): SOPs, safety plans, vendor pricing, equipment manuals
- Build golden retrieval set: 50 real crew questions → measure retrieval quality
- Source confidence scoring on all chunks

### Ingestion strategy (Tier 1 only in A2)

**A2 builds only Tier 3 (deep indexing) for crew docs.** Tier 1 and Tier 2 are defined here for reference — they apply to B2 customer ingestion (future).

| Tier | What | Cost | Timeline | Used In |
|------|------|------|----------|---------|
| **Tier 1** | Auto-classify + thumbnail (no AI model) | Free | <5 seconds | B2 (customer UX) |
| **Tier 2** | Fast Haiku classification + priority routing | ~$1.50 per 1,000 docs | <1 minute | B2 (customer UX) |
| **Tier 3** | Deep OCR + chunking + embedding | ~$8-15 per 2GB corpus | 30-90 minutes | **A2 crew docs** |

For crew docs (A2): typically 50-500 documents total, cost ~$5-50 for full Tier 3 ingestion. Only Tier 3 is built in A2.

### Success criteria

- Crew can ask real document questions and get useful, cited answers
- RAG faithfulness >0.7 (RAGAS metric)
- Latency <3s per query (p95)
- Zero hallucinated sources (LLM admits when it doesn't know)

### A2 is DONE when:

- ✓ pgvector enabled on Neon BBInc_1 (`CREATE EXTENSION vector`)
- ✓ `bb_knowledge_chunks` table created (crew-only, no tenant_id)
- ✓ Google Drive → extraction → chunking → embedding pipeline operational
- ✓ Golden retrieval set (50 real crew questions) passes RAG faithfulness >0.7
- ✓ Latency verified: <3s p95 for hybrid vector+BM25 query
- ✓ Zero hallucinated citations in golden set (perfect confidence scoring)
- ✓ Ingestion pipeline stable for 48h without error
- ✓ `remember` tool stores tribal knowledge + upserts to RAG successfully
- ✓ T0.5 AI evaluator confirms RAG metrics (harness validates)
- ✓ Code merged to main, ready for A3

### Explicitly OUT of scope for A2

- ❌ Multi-tenant schema / customer upload pipeline
- ❌ Property knowledge tables / home asset intelligence
- ❌ Homeowner document ingestion (email relay, customer Drive)
- ❌ Appliance knowledge base / recall monitoring

### Dependencies

- A1 (MCP tool layer must exist for the knowledge tool)
- T0.3 + T0.5 (seeded knowledge base required for evaluation)

---

## A3 (Phase 3): BB Operations Assistant (Crew Operations Only) — Effort: Large, Risk: Medium

**Timeline:** 4-5 weeks (start after A2 COMPLETE)

**Goal:** Buddy can query business data and take governed actions. **Crew operations only — no homeowner features.**

### What

- `query_data` MCP tool: route to existing QBO/QBT Bridge endpoints → summarize
  - "Run me a P&L for Q1" → QBO Reports API → GPT-4o-mini summary → speak
  - "What did we spend on Johnson?" → SQL query on cal_receipts + QBO invoices → summarize
  - "Which employees logged hours this week?" → cal_timesheets + employees → summarize

- Route `calexp_action` to **real** CalExp5 API endpoints (no stubs)
  - Crew auth: CalExp5 PIN login → scoped permissions
  - Sam sees financials, crew sees timesheets + their own data

- Write workflows with execution control:
  - Idempotency keys (24hr expiry)
  - Approval tiers (financial > composite > simple)
  - State machines (proposed → validated → approved → executing)
  - Outbox pattern for notifications
  - Retry + compensation on failure

- Role-based crew permissions
  - Admin: all data, all actions
  - Lead: crew timesheets, team operations
  - Crew: own data only

### Success criteria

- **Daily utility:** Crew initiates ≥5 Buddy sessions/day during 48h production shadow (measured via `cal_scan_transcripts` row count)
- **ROI:** (hours saved per day × $65/hr crew rate) > system daily cost ($20). Hours saved measured by crew self-report survey (N≥5, Likert 1-5, avg hours saved ≥1.5/day)
- **Satisfaction:** Crew satisfaction survey N≥5, Likert 1-5, avg ≥4.0
- All write actions idempotent (same request 3× = 1 DB row, verified by automated test)
- No orphaned state (outbox relay tested: simulated Bridge failure → all queued items delivered within 24h)

### A3 is DONE when: (BINARY CRITERIA — ALL MUST PASS)

**Part 1: Build complete (agent marks A3 "built" when all pass)**
- ✓ QBO/QBT SQL queries routed through `query_data` MCP tool. Queries: "P&L for Q1", "spend on {vendor}", "hours logged" all return correct summaries verified against QBO golden fixtures
- ✓ CalExp5 API integration complete: crew auth via PIN → scoped data access verified. Admin sees all data, lead sees team data, crew sees own data only
- ✓ Write workflows fully implemented: proposed → validated → approved → executing → completed/failed/compensated. State machine verified with automated test cases
- ✓ Idempotency keys enforced on ALL writes (24hr expiry, Postgres-backed). Same write request 3× in a row = 1 row (verified with tests)
- ✓ Approval tiers fully working: composite writes require human approval (SLA: 1h max → deny). Financial writes >$100 require explicit lead+admin approval (SLA: **5min max** → deny write)
- ✓ Outbox pattern tested: all external calls (notifications, QBO sync, email) guaranteed eventual delivery. Zero orphaned state after bridge failure simulation
- ✓ Role-based crew permissions enforced: role matrix checked on every tool call. Role-based schema queries validated
- ✓ All crew data verified single-tenant (zero tenant_id in schema, RLS deferred to B1)
- ✓ T0.6 scale testing passes: 200 VUs sustained load test. Zero 5xx errors at production scale. p95 latency <3s
- ✓ Code merged to main

**Part 2: Production validation (Sam validates — NOT agent's job to self-certify)**
- ✓ 48-hour production shadow: real crew actively using Buddy (5+ sessions/day observed)
- ✓ Cost trending <$20/day over 48-hour window
- ✓ User satisfaction: crew survey N≥5, Likert 1-5, avg ≥4.0
- ✓ Zero critical errors in production logs during shadow period
- ✓ Sam explicitly signs off: A3 production-validated, B-track gate criteria clock starts

**Note for agents:** Your job ends at Part 1. Part 2 is Sam's validation — you cannot self-certify it.

### Explicitly OUT of scope for A3

- ❌ Customer scheduling / subscription management
- ❌ Property CRUD / home asset inventory
- ❌ Warranty tracking / claim filing
- ❌ Homeowner-facing notifications or operations
- ❌ Stripe billing integration
- ❌ Jobber scheduling integration

### Dependencies

- A2 (shared infrastructure, RAG patterns)
- T0.5 (AI eval + scale testing must be stable)
- Existing QBO/QBT Bridge endpoints must be working

---

## Hard Scope Rules (Prevent Scope Leakage)

### Single-Tenant Only (A0-A3)

**FORBIDDEN now:**
- ❌ Add `tenant_id` to any schema (A0-A3)
- ❌ Implement RLS (row-level security) in A0-A3
- ❌ Design multi-tenant isolation (A0-A3)
- ❌ Reference homeowner data (A0-A3)
- ❌ Assume future multi-tenant in code (A0-A3)

**Schema rule:** All tables in Track A are single-tenant. Multi-tenant schema = Track B only (B1 gate).

### RAG vs SQL Agent Separation

**RULE: RAG NEVER queries structured data. SQL Agent NEVER uses embeddings.**

- `knowledge` tool: **ONLY** uses pgvector + hybrid search. Never touches cal_receipts, employees, invoices.
- `query_data` tool: **ONLY** uses SQL queries. Never embeds, never searches vectors.

This prevents spaghetti queries and keeps concerns clean.

### API Keys Stay Server-Side (Permanent)

**FORBIDDEN now:**
- ❌ Send Anthropic API key to client (A0+)
- ❌ Send OpenAI API key to client (A0+)
- ❌ Hardcode external API credentials in HTML/JS

**REQUIRED:** All external API calls route through Bridge MCP tools.

---

## A0-A3 Parallel Timeline with Track 0

```
Week  1  │  A0 (SDK eval)         │  T0.1 (harness core)
Week  2  │  A0                    │  T0.1 + T0.2/T0.3
Week  3  │  A0 → A1 start         │  T0.2 + T0.3
Week  4  │  A1                    │  T0.3 + T0.4
Week  5  │  A1 → A2 start         │  T0.4 + T0.5
Week  6  │  A2                    │  T0.5
Week  7  │  A2 → A3 start         │  T0.5 + T0.6
Week  8  │  A3                    │  T0.6
Week  9  │  A3 + production val   │  T0.6 (universal)
```

**Key:** Track 0 always slightly ahead. Every A phase deliverable must pass Track 0 harness before shipping.

---

## Cost Breakdown (A0-A3, Crew-Only)

| Phase | Primary Cost | Estimated Monthly |
|-------|--------------|-------------------|
| **A0** | OpenAI Realtime (unchanged) | $440 (voice only) |
| **A1** | Server-side MCP tools (Claude/GPT-4o) | +$50 (vision + search) |
| **A2** | RAG queries + embeddings | +$15 (small crew corpus) |
| **A3** | SQL queries + summarization | +$30 (QBO/QBT) |
| **Total A0-A3** | | **~$535/month (10 crew, 4 sessions/day)** |

Net increase: ~$95/month from v3.17. Voice remains dominant cost.

**Note:** Core Architecture projects ~$579/month (includes $17 execution control + $10 observability infrastructure not itemized here). See Core Architecture for full cost breakdown.

---

## CRITICAL GATE: A1 Start Condition

**A1 CANNOT START until:**
1. ✓ T0.1 is COMPLETE (harness core stable + running nightly)
2. ✓ A0 is COMPLETE (SDK evaluation finished, path chosen — SDK or raw WebRTC)
3. ✓ All 8 MCP tool specs written + frozen

**If T0.1 is incomplete, A1 cannot start.** Period. No exceptions. This is a hard gate — not a soft "recommended", not a "baseline".

**Why:** A1 tools must run against T0.1 harness to validate. Without T0.1, there's no way to test MCP tools.

---

## Open Items: ZERO

All gaps resolved (v1.3): A0 fallback contradiction fixed, ingestion tier naming aligned, A3 DONE WHEN split into build vs production validation. A0-A3 is ready to build.

---

*Crew Platform v1.3. See core architecture for decisions; see Track 0 for test harness.*
