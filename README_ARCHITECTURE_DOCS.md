# BB Buddy Architecture Documentation

**This folder contains 4 focused architecture documents that replace the monolithic v1.6 spec.**

---

## Document Map

### 1. **BB_BUDDY_CORE_ARCHITECTURE.md** ← START HERE
- **Audience:** Architects, technical decision-makers
- **Length:** ~10 pages
- **Purpose:** Enduring architecture decisions, not implementation details
- **Contains:**
  - Vision + current state
  - Target architecture diagram
  - Key decisions (OpenAI Realtime, pgvector RAG, MCP, execution control)
  - Cost projections
  - **Execution Boundary Matrix** (what's committed, what's forbidden, what's future)

**When to read:** To understand architectural direction, strategic decisions, technology choices.

---

### 2. **BB_BUDDY_TRACK_0_DELIVERY.md** ← IF YOU'RE BUILDING THE TEST HARNESS
- **Audience:** Claude Code agent, QA engineers, reviewers
- **Length:** ~20 pages
- **Purpose:** Week-by-week test infrastructure delivery plan
- **Contains:**
  - 6 sequential stages (Stage 1 = freeze contracts through Stage 6 = hardening)
  - T0.1-T0.6 detailed specs (deliverables, success metrics, out-of-scope)
  - Frozen contracts (5 TypeBox schemas agents cannot change)
  - Dependency gates to Track A (when each crew phase can start)
  - Acceptance criteria standardized across all 6 stages

**When to read:** If you're implementing Track 0 (test harness), this is your spec.

---

### 3. **BB_BUDDY_CREW_PLATFORM.md** ← IF YOU'RE BUILDING THE CREW PRODUCT
- **Audience:** Claude Code agent, product engineers, reviewers
- **Length:** ~15 pages
- **Purpose:** A0-A3 phases (crew-only features, 9-week roadmap)
- **Contains:**
  - A0: SDK evaluation (framework migration test on iOS Safari)
  - A1: MCP tool layer (server-side execution, model config)
  - A2: RAG crew knowledge (SOPs, safety, pricing, tribal knowledge)
  - A3: Operations assistant (financial queries, crew scheduling, write workflows)
  - **Execution Boundary Matrix** (what's committed, what's forbidden)
  - **Execution Control Backbone** (idempotency, state machines, approvals, outbox)
  - Parallel timeline with Track 0
  - Cost breakdown

**When to read:** If you're implementing the crew platform (A0-A3), this is your spec.

---

### 4. **BB_HOME_PLATFORM_EXPANSION.md** ← DESIGN-ONLY (DO NOT BUILD YET)
- **Audience:** Architects, future product team
- **Length:** ~15 pages
- **Purpose:** Future homeowner subscription platform design (B1-B7)
- **Contains:**
  - **Gate criteria** (A3 must be in production + validated before B-track starts)
  - B1-B7 phases (customer trust, ingestion, property intelligence, scheduling, proactive assistant, model abstraction, multi-agent)
  - Three-audience architecture (crew / homeowners / providers)
  - Three-tenant RAG with RLS isolation
  - Subscription pricing + revenue projections
  - PNW seasonal calendar
  - Schema definitions (all B-track tables)
  - Hypothetical timeline (Q2 2026+ only after A3 validation)

**When to read:** For long-term vision, but DO NOT CODE anything in B1-B7 until A3 is production-validated.

---

## Key Changes from v1.6 Monolith

The original v1.6 monolith (6,500 lines) was split into 4 focused docs + reviewed by ChatGPT (3 tiers of fixes). All 12 issues resolved. For the full detailed changelog, see `FIXES_APPLIED_SUMMARY.md`.

**Summary of fixes:**
- Fix Agent deferred to T0.7 (post-launch only)
- A1 gate crystallized: T0.1 COMPLETE + A0 COMPLETE (no "partial")
- Dual orchestrators named: Conversational (OpenAI) vs Execution (Bridge)
- Crew-only enforced: NO tenant_id / RLS / multi-tenant in Track A
- DONE WHEN checklists added to every phase (T0.1-T0.7, A0-A3)
- Failure modes, cost gates, data lifecycle, security model v1 added
- Hard language (REQUIRED/FORBIDDEN/OPTIONAL) replaces soft language everywhere

---

## How to Use These Documents

### For Architects
- Read **Core Architecture** to understand decisions + tradeoffs
- Skim **Track 0 Delivery** + **Crew Platform** stage-by-stage to understand interdependencies
- Bookmark **Home Platform Expansion** for Q3 2026 planning

### For Claude Code Agent (Building Track 0)
- Read **Track 0 Delivery** end-to-end
- Use **Frozen Contracts** (Stage 1) as non-negotiable boundaries
- Reference **Execution Boundary Matrix** to know what's committed vs future
- Check **Dependency Gates** to understand when Track A phases can start

### For Claude Code Agent (Building Crew Platform)
- Read **Crew Platform** end-to-end
- Use **Execution Boundary Matrix** to know what's crew-only (A0-A3) vs forbidden
- Study **Execution Control Backbone** for A3 implementation (idempotency, state machines)
- Reference **Core Architecture** for decisions + RAG/MCP details

### For Future Product Team (Homeowner Platform)
- Archive **Home Platform Expansion** doc until A3 production validation gate opens
- Do not code against B1-B7 before gate criteria are met
- Use schemas as reference for future UX design

---

## Cross-Document References

| From | To | Reason |
|------|----|----|
| Core Architecture | Track 0 Delivery | "See implementation strategy for how Track 0 is built" |
| Core Architecture | Crew Platform | "See Phase A0-A3 for crew product roadmap" |
| Core Architecture | Home Expansion | "See B-track design for future homeowner platform" |
| Track 0 Delivery | Core Architecture | "See decisions + cost models" |
| Track 0 Delivery | Crew Platform | "See dependency gates to Track A" |
| Crew Platform | Core Architecture | "See RAG/MCP architecture + execution control" |
| Crew Platform | Track 0 Delivery | "See test harness dependencies" |
| Home Expansion | Crew Platform | "A3 must be production-validated before B-track starts" |

---

## Summary Table

| Document | Purpose | Audience | Lines | Read Time |
|----------|---------|----------|-------|-----------|
| Core Architecture | Decisions + tradeoffs | Architects | ~550 | 15 min |
| Track 0 Delivery | Test harness plan (6 stages + frozen schemas + dir tree) | Builders | ~500 | 30 min |
| Crew Platform | A0-A3 roadmap (9 weeks) | Builders | ~380 | 20 min |
| Home Expansion | B1-B7 design + B-track schemas (deferred) | Future team | ~350 | 20 min |

---

## Migration from v1.6

The original **BB_BUDDY_ARCHITECTURE_V2.md** (6,500 lines) has been split and archived:
- Core decisions → **BB_BUDDY_CORE_ARCHITECTURE.md**
- Track 0 spec → **BB_BUDDY_TRACK_0_DELIVERY.md**
- Crew features → **BB_BUDDY_CREW_PLATFORM.md**
- Homeowner design → **BB_HOME_PLATFORM_EXPANSION.md**

**Do not edit v1.6 directly.** All future updates go to the appropriate focused document above.

---

## Status

✅ All ChatGPT Tier 1-3 fixes applied (12/12). See `FIXES_APPLIED_SUMMARY.md` for full detail.

Ready to begin Track 0 Stage 1 (freeze contracts) immediately.

---

*Architecture Documentation v2.4 | 2026-04-02*
