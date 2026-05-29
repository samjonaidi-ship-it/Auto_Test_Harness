# BB Home Platform Expansion | B1-B7 Design | v1.3 | 2026-04-02 | BB

**⚠️ DESIGN-ONLY UNTIL A3 PRODUCTION VALIDATION**

This document describes the future homeowner subscription platform (BB Home). **Do not build any B-track features until:**
1. Track A (crew platform) is deployed and stable in production
2. A3 has run for 4+ weeks with crew actively using it
3. Business case for homeowner expansion is explicitly validated
4. Sam approves gate-opening for B-track implementation

Violation of this gate = scope creep that blocks crew product launch.

---

## B-Track Gate Criteria

**A3 must be in production AND:**
- ✓ Crew using Buddy 5+ times per day
- ✓ Cost per session trending downward (learning + caching)
- ✓ Crew satisfaction score >4/5
- ✓ Zero critical defects in last 2 weeks
- ✓ MCP tool ecosystem stable (no model swaps needed)
- ✓ RAG faithfulness >0.8 on crew corpus (A2 ships at >0.7 — B-track gate requires higher bar: 4+ weeks of production data must show sustained >0.8)
- ✓ Sam explicitly approves B-track activation

Only then do we begin B1 **implementation**. (B1 design is already in this doc — the gate governs when we start coding it.) B2+ implementation starts Q3 2026 at earliest.

---

## Market Opportunity

- U.S. home services market: **$842 billion** (2026), growing to $989B by 2031
- 62% of U.S. consumers already use recurring service plans
- Home services: tree, window washing, gutter cleaning, landscaping, pressure washing, roof cleaning
- **Nobody has an AI voice+vision assistant for home services.** BB Buddy is genuinely novel.

---

## B1: Customer Trust Platform — Design Phase

**Timeline:** Q2 2026 (design only, no implementation)

**Gate:** A3 production validation

### What

- JWT auth for homeowners (Clerk individual users)
- Tenant isolation beyond DB (storage, cache, model context)
- Privacy dashboard, delete/export/revoke
- Trust suites (release gates from Track 0 governance spec)

### Why Separate

Homeowner data is sensitive. Crew data isolation is nice-to-have. Customer data isolation is non-negotiable. RLS must be enforced at DB layer, not app layer.

---

## B2: Customer Ingestion — Design Phase

**Timeline:** Q2 2026 (design only, no implementation)

**Gate:** B1 design complete + security review

### What

- Document upload (homeowner Drive folders)
- Email relay (Postmark inbound webhook)
- SMS/MMS ingestion
- Object storage isolation (signed URLs, tenant-scoped cache)
- Customer-specific RAG (per-property knowledge base)

### Ingestion Strategy (Tier 1-3)

Same three-tier approach as crew docs, but with customer onboarding UX:

1. **Tier 1 (instant, free):** Auto-classify + thumbnail
2. **Tier 2 (fast, ~$1-3):** Haiku classification + priority routing
3. **Tier 3 (deep, ~$5-15 per customer):** OCR + enrichment + embedding

UX: Customer uploads 2GB folder → appears in gallery instantly → indexed in background → "247 of 412 documents ready"

### Why Separate

Customer documents contain PII (inspection reports, insurance, loan docs). Ingestion pipeline must never contaminate crew knowledge base. Separate tenant_id strategy required.

---

## B3: Property Intelligence — Design Phase

**Timeline:** Q3 2026 (design only, no implementation)

**Gate:** B2 implementation + 50+ customers on-boarded

### What

- `bb_properties`, `bb_home_assets`, `bb_property_trees` tables (defined in schema appendix)
- Service history + line items
- Confidence/provenance fields (Schema Provenance section)
- Homeowner verification gamification loop

### Why Separate

Property data requires rich structured schema (trees, assets, services, line items, warranties). Crew-only platform doesn't need this complexity. B3 is where homeowner-specific intelligence activates.

---

## B4: Scheduling & Service UX — Design Phase

**Timeline:** Q3 2026 (design only, no implementation)

**Gate:** Property intelligence stable, first 100 customers acquired

### What

- Customer scheduling with advance notification + approval
- Date-blocking, communication preferences, calendar
- Provider workflow (dispatch, checkin, completion)
- Jobber/Housecall Pro integration (Year 1)

### Why Separate

Scheduling affects crew operations (calendar + dispatch). Only build after A3 crew scheduling is stable and non-interruptible.

---

## B5: Proactive Home Assistant — Design Phase

**Timeline:** Q4 2026+ (design only)

**Gate:** B4 operational + seasonal testing (4+ seasons of data)

### What

- Maintenance reminders, seasonal recommendations
- Recall monitoring, warranty expiry alerts
- Claims/warranty workflows
- Neighborhood clustering (DBSCAN at 150+ subscribers)

### Why Separate

Requires PNW seasonal calendar + historical service patterns. Can't accurately predict without 1+ year of customer data. Build after proof of concept.

---

## B6: Voice Model Abstraction (Future Phase 4)

**Timeline:** 2027+ (design only)

**Gate:** Crew + homeowner platform both stable, clear business case for alternatives

### What

- Abstract transport: OpenAI WebRTC, Gemini Live, future Claude Realtime
- Provider selection UI
- Model fairness analysis (cost vs quality vs latency per provider)

### Why Defer

OpenAI Realtime dominates market. Abstraction adds complexity. Only pursue if:
- Clear customer demand for alternative
- Regulatory requirement (EU AI Act, etc.)
- Cost pressure forces provider diversification

---

## B7: Multi-Agent Orchestration (Future Phase 5)

**Timeline:** 2027+ (design only)

**Gate:** Single-agent platform (A+B1-B5) fully mature

### What

- Task decomposition, parallel execution, handoff chains
- A2A protocol (agent-to-agent communication)
- Long-context session management
- Meta-reasoning (agent reasons about which agent to call)

### Why Defer

Single-agent (OpenAI Realtime as orchestrator) is sufficient for crew + homeowner. Multi-agent adds operational complexity. Only pursue at massive scale or for novel product features not possible with single orchestrator.

---

## Three-Audience Architecture (Post-A3)

**After A3 validates crew product, B1-B5 will introduce three audiences:**

```
                    BB BUDDY PLATFORM
                         │
        ┌────────────────┼────────────────┐
        │                │                │
   BB CREW          HOMEOWNERS      SERVICE PROVIDERS
  "BB Buddy"        "BB Home"         "BB Pro"
        │                │                │
  All tools        Their property    Their catalog
  All data         Their docs        Their customers
  Financials       Global knowledge  Global knowledge
  GPS/timesheets   Linked providers  Scheduling
```

### Three-Tenant RAG Architecture (Post-A3)

| tenant_id | What | Who Sees | Purpose |
|-----------|------|----------|---------|
| `bb_global` | Building codes, PNW calendar, safety standards | Everyone | Universal knowledge |
| `bb_crew` | BB internal SOPs, vendor pricing, crew procedures | BB crew only | Crew knowledge |
| `cust_{uuid}` | Homeowner's inspection reports, warranties, contracts | That homeowner only | Customer knowledge |
| `prov_{uuid}` | Provider's service catalog, certifications, pricing | Provider + linked customers | Provider knowledge |

**Data isolation guaranteed by 5 layers:**
1. Postgres RLS — database enforces tenant_id filtering
2. Application filter — every query includes `WHERE tenant_id IN (...)`
3. JWT authentication — tenant_id extracted from signed token
4. Tool-level gating — LLM never sees tools it can't use
5. Audit log — every RAG query logged with tenant context

---

## PNW Seasonal Calendar (B5 Content)

Annual service schedule embedded in RAG as `tenant_id = 'bb_global'`:

| Season | Services | BB Buddy Proactive |
|--------|----------|-------------------|
| **Spring (Mar-May)** | Gutter clean, tree prune, aeration, first mow, mulch, irrigation startup | "Spring is here — gutters need post-winter cleaning. Book?" |
| **Summer (Jun-Aug)** | Weekly mowing, window washing, hedge trim, irrigation tune-up | "Best window for exterior windows. Shall I book?" |
| **Fall (Sep-Nov)** | Gutter clean, leaf removal, roof moss treatment, winterize irrigation | "November gutter cleaning is critical. Heavy leaves accumulating..." |
| **Winter (Dec-Feb)** | Storm response, tree removal (best pricing), drainage inspection | "Winter is cheapest for tree removal. That leaning maple..." |

---

## Subscription Tiers (B4 Pricing)

| Tier | Monthly | Annual (10% off) | Services |
|------|---------|------------------|----------|
| **BB Essential** | $149/mo | $1,609/yr | Bi-weekly mowing (seasonal), 2x gutter clean, 1x window wash |
| **BB Complete** | $299/mo | $3,229/yr | Weekly mowing, 2x gutter clean, 2x window wash, 1x pressure wash, seasonal cleanup |
| **BB Premium** | $499/mo | $5,389/yr | All Complete + tree care, roof cleaning, irrigation management, priority scheduling |

**Revenue projection:** 100 subscribers at BB Complete = $29,900/month = **$358,800/year recurring**.

---

## B-Track Schema (Design Intent — Implement at B3 Gate)

**These table definitions are design intent only. Do not create them until B3 gate opens.**

All B-track tables require `tenant_id` (RLS-enforced, per-homeowner isolation). They do NOT exist in the current Neon BBInc_1 schema — that schema is crew-only (Track A).

```sql
-- bb_properties: one row per homeowner property
CREATE TABLE bb_properties (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id    UUID NOT NULL,                          -- homeowner's tenant
  address      TEXT NOT NULL,
  city         TEXT NOT NULL,
  state        TEXT NOT NULL DEFAULT 'WA',
  zip          TEXT NOT NULL,
  lot_sqft     NUMERIC,
  structure_sqft NUMERIC,
  year_built   INT,
  meta         JSONB DEFAULT '{}',
  created_at   TIMESTAMPTZ DEFAULT NOW()
);

-- bb_property_trees: tree inventory per property
CREATE TABLE bb_property_trees (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  property_id  UUID NOT NULL REFERENCES bb_properties(id) ON DELETE CASCADE,
  tenant_id    UUID NOT NULL,
  species      TEXT,
  height_ft    NUMERIC,
  diameter_in  NUMERIC,
  health       TEXT,                                   -- 'good' | 'fair' | 'poor' | 'dead'
  last_trimmed DATE,
  notes        TEXT,
  location_geo JSONB,                                  -- {lat, lng}
  created_at   TIMESTAMPTZ DEFAULT NOW()
);

-- bb_home_assets: appliances, systems, equipment
CREATE TABLE bb_home_assets (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  property_id  UUID NOT NULL REFERENCES bb_properties(id) ON DELETE CASCADE,
  tenant_id    UUID NOT NULL,
  asset_type   TEXT NOT NULL,                          -- 'hvac' | 'roof' | 'appliance' | etc
  brand        TEXT,
  model        TEXT,
  serial_no    TEXT,
  install_date DATE,
  warranty_exp DATE,
  notes        TEXT,
  meta         JSONB DEFAULT '{}',
  created_at   TIMESTAMPTZ DEFAULT NOW()
);

-- bb_service_projects: work completed at property
CREATE TABLE bb_service_projects (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  property_id  UUID NOT NULL REFERENCES bb_properties(id) ON DELETE CASCADE,
  tenant_id    UUID NOT NULL,
  service_type TEXT NOT NULL,                          -- 'gutter_clean' | 'mowing' | etc
  scheduled_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  crew_id      UUID,                                   -- references employees table
  status       TEXT DEFAULT 'scheduled',               -- 'scheduled' | 'completed' | 'cancelled'
  notes        TEXT,
  created_at   TIMESTAMPTZ DEFAULT NOW()
);

-- bb_service_line_items: granular costs per project
CREATE TABLE bb_service_line_items (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id   UUID NOT NULL REFERENCES bb_service_projects(id) ON DELETE CASCADE,
  tenant_id    UUID NOT NULL,
  description  TEXT NOT NULL,
  quantity     NUMERIC DEFAULT 1,
  unit_price   NUMERIC NOT NULL,
  total        NUMERIC GENERATED ALWAYS AS (quantity * unit_price) STORED,
  created_at   TIMESTAMPTZ DEFAULT NOW()
);
```

**RLS policy pattern (apply to all B-track tables at B1 gate):**
```sql
ALTER TABLE bb_properties ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON bb_properties
  USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
-- Repeat for all B-track tables
```

---

## Implementation Timeline (Hypothetical Post-A3)

| Timeline | Gate | Gate Criteria |
|----------|------|---------------|
| **Q2 2026** | B1 | A3 production validation (4+ weeks) |
| **Q3 2026** | B2 | B1 design complete + security review |
| **Q3 2026** | B3 | B2 implementation + 50+ customers |
| **Q4 2026** | B4 | Property intelligence stable |
| **Q1 2027** | B5 | B4 operational + seasonal data collected |
| **2027+** | B6-B7 | Mature single-agent platform + business case |

---

## Competitive Moat

**No existing home service platform has this:**
- LawnStarter: satellite imagery (passive, no conversation)
- SingleOps: tree inventory (manual, arborist-only)
- Jobber/Housecall Pro: scheduling + CRM (no AI)
- Lowe's HomeCare+: basic indoor tasks ($99/year, no exterior)

**BB Buddy pointing at a tree and telling you its species, health, when last trimmed — while cross-referencing inspection reports and the PNW seasonal calendar — is genuinely novel.**

---

**This document is design intent only. No implementation until A3 production validation. Do not commit B-track code to master until gate criteria met.**

*B-Track Design v1.0. See BB_Buddy_Crew_Platform for committed roadmap.*
