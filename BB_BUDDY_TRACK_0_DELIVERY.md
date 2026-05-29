# BB Buddy Track 0 Delivery Plan | v1.6 | 2026-04-02 | BB

**Track 0 = Quality Infrastructure.** This is the foundation for all features. Nothing ships without passing Track 0's test harness.

---

## Track 0 Overview

**Goal:** Build autonomous test harness + synthetic data factory + AI evaluation layer + cross-platform validation infrastructure.

**Timeline:** 6-9 weeks (5 MVP stages + 1 post-launch stage). Stage 1 = contracts, Stages 2-5 = T0.1–T0.6 MVP, Stage 6 = T0.7 post-launch only (not counted in 6-9 week estimate).

**Owner:** Claude Code (80% implementation) + Rcodex agents (review + 3-zero) + Sam (architect/merge)

**Success criteria:** Track 0 complete = nightly autonomous testing with voice + RAG + scale validation, all results tracked, no critical gaps.

---

## Execution Boundary Matrix (Track 0 Scope)

| Area | Status | Build Now? | Gate | Notes |
|------|--------|-----------|------|-------|
| **Harness core** (runner, waves, manifest) | Committed | YES (Week 1) | None — critical path | T0.1 deliverable |
| **Observer dashboard** (live UI) | Committed | YES (Week 2) | T0.1 complete | T0.2 deliverable |
| **Synthetic data (relational)** | Committed | YES (Week 3) | T0.1 complete | T0.3 deliverable, enables RAG testing |
| **Synthetic data (SDV + assets)** | Committed | YES (Week 4) | T0.3 COMPLETE | T0.4 deliverable |
| **AI evaluators** (voice + RAG + judge) | Committed | YES (Week 5) | T0.3 complete | T0.5 deliverable, requires seeded data |
| **Cross-platform** (BrowserStack + k6) | Committed | YES (Week 8) | T0.5 complete | T0.6 deliverable |
| **Universal project discovery** | Future | NO (post-launch) | A3 + T0.6 stable | Out of scope for launch |
| **Fix agent** (autonomous patching) | Post-launch | NO (T0.7 only) | A3 production stable (4+ weeks) | Deferred to Stage 6 (T0.7) — NOT part of T0 MVP |

---

## Stage-by-Stage Delivery

### Stage 1: Freeze Contracts (Day 1)

**Goal:** Lock down 5 schemas that agents cannot change. These are the constitution — all downstream modules build to these shapes.

**Deliverables:**
- `Auto_Test_Harness/schemas/manifest.ts` — run manifest schema
- `Auto_Test_Harness/schemas/result-classification.ts` — PASS / WARN / FAIL taxonomy
- `Auto_Test_Harness/schemas/scenario-dsl.ts` — scenario YAML validator
- `Auto_Test_Harness/schemas/evaluator-api.ts` — evaluator input/output shape
- `Auto_Test_Harness/schemas/patch-proposal.ts` — safe fix proposal format

**SCHEMA DEFINITIONS (canonical — implement exactly as specified):**

**Schema 1: Run Manifest** (`manifest.ts`)
```ts
{
  run_id: string,           // uuid
  project: string,          // 'bb-buddy' | 'bb-micro-bridge' | 'calexp5'
  suite: string,            // 'voice-sessions' | 'rag-accuracy' | etc
  scenario_id: string,      // references scenario DSL name field
  build_sha: string,        // git commit sha
  trace_id: string,         // uuid, for distributed tracing
  result_class: string,     // 'deterministic_fail' | 'policy_fail' | 'golden_miss' | 'llm_warn' | 'pass'
  artifacts: string[],      // ['traces/...', 'logs/...']
  cost_cents: number,       // integer, total cost of this run
  duration_ms: number,      // integer, wall-clock time
  timestamp: string,        // ISO 8601 UTC
}
```

**Schema 2: Result Classification** (`result-classification.ts`)
```ts
const RESULT_CLASSES = {
  deterministic_fail: { blocks_build: true,  blocks_release: true,  requires: 'code_fix' },
  policy_fail:        { blocks_build: true,  blocks_release: true,  requires: 'policy_review' },
  golden_miss:        { blocks_build: false, blocks_release: true,  requires: 'investigation' },
  llm_warn:           { blocks_build: false, blocks_release: false, requires: 'human_review' },
  pass:               { blocks_build: false, blocks_release: false, requires: null },
}
// deterministic_fail / policy_fail = hard blocks (code MUST be fixed before proceeding)
// golden_miss = release-blocking (investigation required, not necessarily a code bug)
// llm_warn = advisory only (human reviews, can ship)
// pass = green
```

**Schema 3: Scenario DSL** (`scenario-dsl.ts`) — validated by TypeBox before YAML compilation
```ts
{
  name: string,                                       // unique scenario identifier
  seed: string,                                       // PRNG seed for reproducibility
  scale: {
    tenants: number,                                  // 10-500
    crew: number,                                     // 1-20
  },
  property_distribution: Record<string, number>,      // archetype → probability, must sum to 1.0
  documents: Record<string, number>,                  // doc_type → probability
  assets_per_property: {
    min: number,
    max: number,
    mandatory: string[],
  },
  edge_cases: Record<string, number>,                 // case_type → probability
  correlations: Record<string, boolean>,              // correlation_name → enabled
}
```

**Schema 4: Evaluator API** (`evaluator-api.ts`) — every evaluator (RAG, voice, deterministic) exposes this shape
```ts
{
  input: {
    scenario_id: string,
    question: string,
    context: string,
    expected: string,
  },
  metrics: {
    faithfulness: number,       // 0-1 (RAG metric: answer grounded in context)
    relevance: number,          // 0-1 (answer addresses the question)
    hallucination: number,      // 0-1 (lower = better, 0 = no hallucination)
    latency_ms: number,
    token_cost_cents: number,
  },
  verdict: string,              // result_class (one of RESULT_CLASSES keys)
  explanation: string,          // human-readable justification
  artifacts: string[],          // ['transcripts/...', 'scores/...']
}
```

**Schema 5: Patch Proposal** (`patch-proposal.ts`) — safe fix proposal format (T0.7 only, define now, use later)
```ts
{
  patch_id: string,             // uuid
  suite: string,                // 'voice-sessions' | 'rag-accuracy' | etc
  failure: {
    test_name: string,
    assertion_error: string,
    expected: string,
    actual: string,
  },
  diff: string,                 // unified diff string
  files_changed: string[],      // ['src/routes/scan-live-v2.js']
  lines_changed: number,
  confidence: number,           // 0-1
  sandbox_branch: string,       // 'fix/voice-sessions-1712345678'
  forbidden_paths_touched: boolean,   // MUST be false before any review
  re_test_result: string,       // result_class
  approval_status: string,      // 'pending' | 'approved' | 'rejected'
}
```

**Stage 1 DONE WHEN:**
- ✓ All 5 TypeBox schema files written at `Auto_Test_Harness/schemas/`
- ✓ Each schema passes standalone TypeBox validation (no build errors)
- ✓ `schemas/index.ts` exports all 5 schemas
- ✓ Zero TODO / placeholder / `any` types in schema files
- ✓ Checked into `main` branch

**Out of scope:** Implementation. Only contracts.

---

### Stage 2: Harness Core (T0.1) — Weeks 1-2

**Goal:** Rcodex-aligned test orchestrator running overnight with 3-zero completion.

**Deliverables:**
- `Auto_Test_Harness/core/orchestrator.js` — wave engine, 3-zero, checkpoints, manifests, stream files
- `Auto_Test_Harness/core/cost-tracker.js` — budget gating, cost-history.jsonl
- `Auto_Test_Harness/core/reporter.js` — Rcodex-format output files
- `Auto_Test_Harness/node-tests/` — 4 test files (bb-buddy-session, rag-accuracy, privacy-pipeline, scheduling-engine) copied into `BB_Micro_Bridge/tests/`. **Note: stripe-webhooks.test.js is a Bridge legacy regression test (pre-A-track), kept in `BB_Micro_Bridge/tests/` for infrastructure coverage only. It does NOT represent committed A-track scope and does NOT gate crew launch.**
- `Auto_Test_Harness/harness.js` — CLI (--auto, --observe, --project, --suite)
- `.harness/` runtime state directory (HARNESS_STATE.md, MANIFEST.json)

**Success metric:** `node harness.js --auto --project bb-micro-bridge` runs overnight, produces SESSION SUMMARY with 3-zero on all unit test suites.

**REQUIRED Dependencies:** None. Start immediately.

**FORBIDDEN in T0.1:**
- ❌ AI evaluators (T0.5)
- ❌ Synthetic data generation (T0.3+)
- ❌ Real device testing (T0.6)
- ❌ Observer UI (T0.2)
- ❌ Scale testing (T0.6)
- ❌ Fix agent (T0.7, post-launch only)

**OPTIONAL (if time allows, non-blocking):**
- Unit test framework selection (Jest vs Vitest)

---

### Stage 3: Observer + Seed Factory (T0.2 + T0.3) — Weeks 2-4

**Run in parallel using worktrees:**

**⚠️ DEPENDENCY: T0.1 must be COMPLETE before starting T0.2 or T0.3.**

**Worktree A (T0.2): Observer Dashboard + Governance**
- `Auto_Test_Harness/observer-ui/index.html` — single-page dashboard (vanilla HTML/CSS/JS, BB theme)
- WebSocket server on port 9801 streaming test events as JSON
- Dashboard features: progress bars, live logs, voice transcripts, fix diffs, cost
- Buttons: Pause, Skip Suite, Force Rerun, Open in VS Code
- Governance enforcement: authority model (deterministic first, LLM advisory), execution mode isolation

**Worktree B (T0.3): Synthetic Data Factory — Engine 1 (Relational)**
- `Auto_Test_Harness/synthetic/scenarios/golden-10.yaml` — 10-property golden fixtures
- `Auto_Test_Harness/synthetic/scenarios/bainbridge-200.yaml` — 200-property scenario
- `Auto_Test_Harness/synthetic/engine-1-relational/` — generators + compiler + loader
- `Auto_Test_Harness/synthetic/seeder.js`, `cleaner.js`, `validator.js`, `snapshotter.js`
- Templates: addresses-bainbridge.json, appliance-catalog.json, service-templates.json

**Success metrics:**
- T0.2: Dashboard opens, WebSocket streaming test events, controls (pause/skip/rerun) responsive
- T0.3: Relational seeder generates golden-10 fixtures in <30s, all constraints satisfied

**Dependencies:** **T0.1 COMPLETE (hard gate).** Cannot start T0.2 or T0.3 until T0.1 fully passes.

**Out of scope:** SDV/Tonic, advanced audio/video generation, massive datasets (>10K rows), fix agent.

---

### Stage 4: SDV + Asset Forge + AI Eval (T0.4 + T0.5) — Weeks 4-7

**Run sequentially (most complex modules):**

**T0.4: Synthetic Data Factory — Engine 2 (SDV) + Engine 3 (Asset Forge)**
- `Auto_Test_Harness/synthetic/engine-2-synthetic/sdv-trainer.py` — train SDV HMA model on golden fixtures
- `Auto_Test_Harness/synthetic/engine-2-synthetic/sdv-generator.py` — generate 200 correlated property portfolios
- `Auto_Test_Harness/synthetic/engine-3-asset-forge/` — see full spec in **CalExp5 Data Factory** section below
- `Auto_Test_Harness/synthetic/fixtures/` — pre-downloaded fixture pool (download-once, checked in)
- `Auto_Test_Harness/synthetic/download-fixtures.js` — one-time fixture download script
- `Auto_Test_Harness/e2e/fixtures/seed-indexeddb.js` — Playwright IndexedDB seeder fixture
- `Auto_Test_Harness/synthetic/embedder.js` — batch embed with caching (text-embedding-3-small)

**Engine 3 generates 4 asset classes. See CalExp5 Data Factory section for full agent implementation spec.**

**T0.5: AI Evaluation Layer**
- `Auto_Test_Harness/evaluators/voice-evaluator.js` — LangWatch Scenario wrapper for headless voice testing
- `Auto_Test_Harness/evaluators/rag-evaluator.js` — RAGAS metrics (faithfulness, recall, precision, hallucination) using Haiku as judge
- `Auto_Test_Harness/evaluators/llm-judge.js` — generic Haiku judge for custom assertions
- `Auto_Test_Harness/projects/bb-buddy/voice-sessions.suite.js` — 6 persona archetypes × 5 questions each
- `Auto_Test_Harness/projects/bb-buddy/rag-accuracy.suite.js` — 20-query battery against seeded knowledge base
- Nightly regression: Railway Cron → `harness.js --auto --project bb-buddy` → digest + SMS on critical failure

**Railway Cron wiring spec (T0.5 deliverable):**
```
Service name:   auto-test-harness
Runtime:        Node 22.x, start command: node harness.js
Cron schedule:  0 3 * * *   (3 AM Pacific daily)
Env vars required:
  OPENAI_API_KEY
  ANTHROPIC_API_KEY
  DATABASE_URL          (Neon BBInc_1 connection string)
  HARNESS_PROJECT       bb-buddy
  HARNESS_NOTIFY_SMS    +1XXXXXXXXXX (Sam's number)
  HARNESS_COST_LIMIT    5000  (cents — $50 abort threshold)
On critical failure:  harness.js exits code 1 → Railway logs + SMS alert
On success:           harness.js exits code 0 → SESSION SUMMARY written to .harness/MANIFEST.json
```

**Success metrics:**
- T0.4: 200-property dataset generated, all statistical features validated (KL divergence <0.1)
- T0.5: Voice: 30/30 persona sessions pass >80%. RAG: 50 golden queries >0.7 faithfulness. Zero hallucinations.

**Dependencies:** T0.3 COMPLETE (relational seeds must exist and be stable).

---

### Stage 5: Scale + Integration (T0.6) — Weeks 7-9

**Goal:** Cross-platform validation + all 3 core projects.

**Deliverables:**
- `Auto_Test_Harness/e2e/playwright.config.bs.js` — BrowserStack integration (iOS Safari, Android Chrome, Firefox desktop)
- `Auto_Test_Harness/scale/k6-load.js` — 200 VUs against Bridge endpoints
- `Auto_Test_Harness/scale/mock-ai-server.js` — Fastify on port 9999, deterministic AI responses ($0 scale testing)
- `Auto_Test_Harness/e2e/specs/bb-buddy/` — onboarding, voice-session, crew-scheduling Playwright specs
- `Auto_Test_Harness/projects/bb-micro-bridge/` — API integration, receipt pipeline, GPS suites
- `Auto_Test_Harness/projects/calexp5/` — E2E flows, PWA offline suites
- Cross-project regression detection (change in Bridge → also test CalExp5)
- Trend dashboard in observer UI (cost, pass rate over 4+ weeks)

**Success metric:** All 3 core projects have passing nightly test suites. BrowserStack: iOS Safari + Android Chrome pass. Trend dashboard: 4+ weeks of data. Zero flaky tests.

**Dependencies:** T0.5 COMPLETE (AI evaluators must be stable and producing consistent metrics).

**Out of scope:** Universal project discovery, arbitrary 55+ projects, fix agent (deferred to post-launch).

---

### Stage 6: Post-Launch Hardening (T0.7) — DEFERRED — After A3 Production Validation

**🚨 CRITICAL: T0.7 is NOT part of Track 0 MVP. Do NOT build until A3 is proven stable in production (4+ weeks real crew usage).**

**What T0.7 is:**
Autonomous fix agent that can detect test failures, propose safe patches, and submit for human review.

**What T0.7 is NOT:**
- Not autonomous patch merging (always human review)
- Not general-purpose code generation (only pre-approved fix patterns)
- Not part of launch build (post-launch enhancement only)

**T0.7 Gate Criteria (HARD STOPS — all must be true before starting):**
- ✓ A3 has been in production for 4+ weeks
- ✓ Crew is actively using Buddy daily (10+ sessions/day)
- ✓ Cost is trending stable (<$20/day)
- ✓ T0.6 test harness is running nightly with >95% pass rate
- ✓ Sam explicitly approves T0.7 work

**If any gate is not met, T0.7 remains deferred.**

**Deliverables (when gates are met):**
- `Auto_Test_Harness/fix-agent/` — Claude Code agent triggered on test failure
- Sandbox branch enforcement (fixes never commit to main directly)
- Trust suite gates (patches must pass all 3-zero test phases)
- Human review + approval flow (Sam approves before merge)
- Patch proposal feedback loop (fix agent learns from rejections)

**Success metric:** Fix agent reduces MTTR (mean time to repair) by 80%. All patches human-reviewed. Zero broken patches merged to main.

**Out of scope:** Fully autonomous merging, unbounded code generation, multi-agent fix loops.

---

## Track 0 Parallel Timeline

```
Week  1  │  T0.1 (harness core + node tests)      │  (critical path)
Week  2  │  T0.1 + T0.2 (observer)                │  T0.2 in worktree A
         │         + T0.3 (relational seeder)    │  T0.3 in worktree B
Week  3  │  T0.2 + T0.3 integration               │
Week  4  │  T0.3 + T0.4 (SDV + assets)            │
Week  5  │  T0.4 + T0.5 (AI eval)                 │
Week  6  │  T0.5 (evaluation layer)               │
Week  7  │  T0.5 + T0.6 (scale + cross-platform) │
Week  8  │  T0.6 (BrowserStack + k6)              │
Week  9  │  T0.6 (universal + hardening)          │
```

**Key insight:** T0.1 is critical path. Everything else chains on it. Stages 2-6 run sequentially with internal parallelism (worktrees at stage 3).

---

## 6-Stage Frozen Contracts (Implementation Agents Only)

Agents receive these contracts + corresponding module-specific prompt prefixes:

**Stage 1 contracts:** 5 TypeBox schemas (manifest, classification, DSL, evaluator, patch)

**Stage 2 contracts:** Manifest schema → Core interface (Runner, Wave, Result) → CLI shape

**Stage 3 contracts:** Scenario DSL → Generator output → Seeder interface

**Stage 4 contracts:** SDV model signature → Asset forge output → Embedder shape

**Stage 5 contracts:** Evaluator API input/output → Judge interface → Suite registry

**Stage 6 contracts:** Test report schema → BrowserStack device matrix → Trend data shape

---

## Track 0 → Track A Validation Mapping (Explicit)

**Each A phase has specific harness suites and pass criteria that must be green before the phase ships. "Must pass Track 0" is not enough — this table is the exact contract.**

| A Phase | Validated By | Pass Criteria | Blocks If Fails |
|---------|-------------|---------------|-----------------|
| **A0** | Manual measurement only (no harness suite yet) | First-audio latency ≤ v3.17 baseline. Zero iOS Safari console errors. Bundle delta documented. | A1 start |
| **A1** | `node-tests/bb-buddy-session.test.js` | 3-zero (3 consecutive passes, 0 failures) | A2 start |
| **A1** | Latency suite (ad-hoc: 10 tool calls, measure p95) | p95 tool call latency <5s | A2 start |
| **A1** | Cost tracking accuracy test (`node-tests/` manual) | Reported cost within ±10% of actual API usage | A2 start |
| **A2** | `projects/bb-buddy/rag-accuracy.suite.js` | 50 golden queries: faithfulness >0.7, hallucination = 0 | A3 start |
| **A2** | `node-tests/rag-accuracy.test.js` | 3-zero | A3 start |
| **A3** | `projects/bb-buddy/` workflow test suite | State machine transitions all verified, idempotency proven (3× same request = 1 row) | Production launch |
| **A3** | `scale/k6-load.js` (T0.6) | 200 VUs, zero 5xx, p95 latency <3s | Production launch |
| **A3** | `node-tests/scheduling-engine.test.js` | 3-zero | Production launch |

**How to read this:** When an A1 agent says "A1 is done," Sam runs the A1 rows. All three must be green. If any fail, A1 is not done — fix and re-run.

---

## Dependency Gates (Track 0 → Track A)

| Track A Phase | Requires | Why |
|---|---|---|
| **A0** (SDK eval) | None | Can start immediately (parallel with T0.1) |
| **A1** (MCP tools) | **T0.1 COMPLETE + A0 COMPLETE** (hard gates) | Harness core must be stable. SDK must be proven on iOS Safari. |
| **A2** (RAG knowledge) | **T0.3 COMPLETE + T0.5 COMPLETE** (hard gates) | Seeded data required for golden retrieval set. Working evaluators required to validate RAG metrics. |
| **A3** (crew operations) | **T0.5 COMPLETE + T0.6 COMPLETE** (hard gates) | Full AI eval required for approval tiers. Scale testing required for production load validation. |

**All gates are HARD STOPS.** No "partial", no "baseline". Phase cannot advance without gate satisfaction.

---

## Track 0 Acceptance Criteria (All Stages)

Every Track 0 stage must pass:

| Criterion | Stage 1 | T0.1 | T0.2 | T0.3 | T0.4 | T0.5 | T0.6 | T0.7 |
|-----------|---------|------|------|------|------|------|------|------|
| **Goal met** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Deliverables complete** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **DONE WHEN checklist passed** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Out-of-scope items excluded** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Dependencies satisfied** | N/A | N/A | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Tests pass (3-zero rule)** | N/A (schemas only) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Code reviewed + merged** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |

**Note on Stage 1:** No tests run (contracts only). DONE WHEN = all 5 schemas compile without errors, no `any` types, exported from `index.ts`, merged to `main`.

**Note on T0.7:** Not part of Track 0 MVP. Row included for completeness only. Gate criteria are listed in Stage 6.

---

## Implementation Strategy

**Build actor:** Claude Code (primary)

**Review actors:** Rcodex agents (BE/UI/Data/E2E)

**Approval:** Sam (merge authority)

**Token discipline:** Every coding session includes footer:
```
- Read only files directly relevant to this task.
- Do not summarize the whole repo.
- Reuse existing utilities whenever possible.
- Return only: (1) files to change, (2) code, (3) tests, (4) short notes.
```

**Caching strategy:** 4 high-ROI caches to prevent re-reading:
1. Context cache (module summaries with public interfaces)
2. Synthetic asset manifest (scenario + version → skip regeneration)
3. Embedding cache (content hash → vector)
4. Evaluation score cache (input hash → metrics)

**Module-specific prompt prefixes (stored in `Auto_Test_Harness/prompts/`):**

Each agent session that builds a Track 0 module MUST load the corresponding prompt prefix. This prevents scope drift — agents cannot wander into adjacent modules.

| Module | Prompt File | Allowed Scope | Forbidden Scope |
|--------|------------|---------------|-----------------|
| Harness Core | `prompts/core-harness.md` | runner, waves, results, artifacts, manifest, CLI | evaluators, synthetic, observer, fix-agent |
| Synthetic Data | `prompts/synthetic-data.md` | scenario DSL, generators, loader, validator, snapshotter, embedder | evaluators, observer, CI, fix-agent |
| AI Evaluation | `prompts/ai-eval.md` | RAG eval, voice eval, judges, scorecards | runner internals, seed generators, observer |
| Platform/CI | `prompts/platform-ci.md` | CI workflows, BrowserStack, k6, budgets, retention crons | evaluators, seed generators, observer |
| Observer/Fix | `prompts/observer-fix.md` | dashboard, replay, annotations, safe patch proposals | protected branches, auth/, billing/ — NEVER implement features |
| Review/Safety | `prompts/review-safety.md` | reviews diffs only, flags drift, policy violations, missing tests | NEVER implements features |

**Content of each prompt file:** Create these as short markdown files (5-10 lines each) that spell out the allowed/forbidden scope. The agent reads the file at session start, then restricts all edits to that scope.

---

## Auto_Test_Harness Project Structure

**Full directory tree (agents must match exactly):**

```
Auto_Test_Harness/
├── package.json                          # Node 22.x, see spec below
├── harness.js                            # CLI entry point (--auto, --observe, --project, --suite)
├── schemas/                              # Stage 1: frozen contracts — DO NOT MODIFY
│   ├── manifest.ts
│   ├── result-classification.ts
│   ├── scenario-dsl.ts
│   ├── evaluator-api.ts
│   ├── patch-proposal.ts
│   └── index.ts                          # exports all 5
├── core/                                 # T0.1: harness runtime
│   ├── orchestrator.js                   # wave engine, 3-zero, checkpoints, manifests
│   ├── cost-tracker.js                   # budget gating, cost-history.jsonl
│   └── reporter.js                       # Rcodex-format output files
├── observer-ui/                          # T0.2: live dashboard
│   └── index.html                        # single-page, vanilla HTML/CSS/JS, BB theme
├── synthetic/                            # T0.3-T0.4: data factory
│   ├── scenarios/
│   │   ├── golden-10.yaml
│   │   └── bainbridge-200.yaml
│   ├── engine-1-relational/              # T0.3: Node.js generators
│   │   ├── generators/
│   │   ├── compiler.js
│   │   └── loader.js
│   ├── engine-2-synthetic/               # T0.4: Python SDV (see Python env spec)
│   │   ├── sdv-trainer.py
│   │   └── sdv-generator.py
│   ├── engine-3-asset-forge/             # T0.4: asset generators (see CalExp5 Data Factory spec)
│   │   ├── receipt-forge.js              # Generator A: JPEG receipt images with EXIF
│   │   ├── streetview-forge.js           # Generator B: 180×180 JPEG from fixture pool
│   │   ├── tool-photo-forge.js           # Generator C: tool catalog photos from fixture pool
│   │   ├── pdf-forge.js                  # Generator D: PDF receipts via pdf-lib
│   │   └── audio-forge.js               # Generator E: voice session audio (WAV/MP3)
│   ├── fixtures/                         # pre-downloaded once, checked into repo (~20MB total)
│   │   ├── receipts/
│   │   │   ├── source/                   # 20 raw receipt JPEGs
│   │   │   ├── processed/                # 20 processed receipt JPEGs
│   │   │   └── thumbs/                   # 20 thumbnails (120px, JPEG q=0.6)
│   │   ├── streetview/                   # 50 real Street View tiles (180×180 JPEG)
│   │   │   └── meta.json                 # { "lat_lng_key": "filename.jpg" }
│   │   ├── tools/                        # 30 Creative Commons tool photos
│   │   │   └── catalog.json              # { "category": ["filename.jpg", ...] }
│   │   └── pdfs/                         # 10 sample receipt PDFs
│   ├── download-fixtures.js              # ONE-TIME ONLY: downloads/generates fixture pool
│   ├── templates/
│   │   ├── addresses-bainbridge.json
│   │   ├── appliance-catalog.json
│   │   └── service-templates.json
│   ├── seeder.js
│   ├── cleaner.js
│   ├── validator.js
│   ├── snapshotter.js
│   └── embedder.js                       # batch embed with caching
├── evaluators/                           # T0.5: AI evaluation layer
│   ├── voice-evaluator.js                # LangWatch Scenario wrapper
│   ├── rag-evaluator.js                  # RAGAS metrics via Haiku
│   └── llm-judge.js                      # generic Haiku judge
├── projects/                             # per-project test suites
│   ├── bb-buddy/
│   │   ├── voice-sessions.suite.js       # 6 personas × 5 questions
│   │   └── rag-accuracy.suite.js         # 20-query battery
│   ├── bb-micro-bridge/
│   │   └── *.suite.js
│   └── calexp5/
│       └── *.suite.js
├── node-tests/                           # T0.1: node test files (copied to BB_Micro_Bridge/tests/)
│   ├── bb-buddy-session.test.js
│   ├── rag-accuracy.test.js
│   ├── stripe-webhooks.test.js          # Bridge legacy regression ONLY — not A-track scope, does not gate crew launch
│   ├── privacy-pipeline.test.js
│   └── scheduling-engine.test.js
├── e2e/                                  # T0.6: cross-platform
│   ├── playwright.config.bs.js           # BrowserStack config
│   ├── fixtures/
│   │   └── seed-indexeddb.js             # Playwright IndexedDB seeder (see CalExp5 Data Factory spec)
│   └── specs/
│       └── bb-buddy/
├── scale/                                # T0.6: load testing
│   ├── k6-load.js
│   └── mock-ai-server.js                 # Fastify port 9999
├── fix-agent/                            # T0.7 ONLY — do not create until T0.7 gate opens
├── prompts/                              # module-specific agent prompt prefixes
│   ├── core-harness.md
│   ├── synthetic-data.md
│   ├── ai-eval.md
│   ├── platform-ci.md
│   ├── observer-fix.md
│   └── review-safety.md
└── .harness/                             # runtime state (gitignored)
    ├── HARNESS_STATE.md
    └── MANIFEST.json
```

**package.json starter spec:**
```json
{
  "name": "auto-test-harness",
  "version": "0.1.0",
  "engines": { "node": ">=22.0.0" },
  "type": "module",
  "scripts": {
    "start": "node harness.js",
    "test": "node harness.js --auto --project bb-micro-bridge"
  },
  "dependencies": {
    "fastify": "^5.0.0",
    "ws": "^8.18.0",
    "@anthropic-ai/sdk": "latest",
    "openai": "latest",
    "zod": "^3.24.0",
    "@sinclair/typebox": "^0.33.0",
    "js-yaml": "^4.1.0",
    "dotenv": "^16.4.0",
    "pino": "^9.0.0"
  },
  "devDependencies": {
    "vitest": "^3.0.0"
  }
}
```

**Python environment spec (for T0.4 SDV modules only):**
```
Python: 3.11.x (not 3.12 — SDV has known 3.12 incompatibilities as of 2026-04)
Virtual env: venv at Auto_Test_Harness/synthetic/engine-2-synthetic/.venv/
Requirements:
  sdv==1.15.0
  pandas==2.2.0
  numpy==1.26.4
  scikit-learn==1.4.0
Activation: source .venv/bin/activate  (Linux/Mac) or .venv\Scripts\activate (Windows)
Entry points: sdv-trainer.py --scenario bainbridge-200.yaml --output models/
             sdv-generator.py --model models/hma.pkl --count 200 --output output/
```

---

## CalExp5 Data Factory Spec (Agent Implementation Reference)

**Purpose:** This section is the complete agent-executable spec for producing synthetic assets for CalExp5 — the first project run through the test harness. Agents building Engine 3 and the IndexedDB seeder MUST implement exactly what is described here.

**Scope:** This covers Engine 3 (asset forge), the fixture pool, the IndexedDB seeder, and the mock-ai-server asset routes. Engine 1 (relational rows) and Engine 2 (SDV statistics) are unchanged.

**Agent scope boundary:** All files in this spec live under `synthetic/` or `e2e/fixtures/`. Do not touch `core/`, `evaluators/`, or `scale/`.

---

### Asset Inventory (What CalExp5 Uses)

Agents must understand what each asset is before generating it. These are the 8 asset types the test harness must produce:

| ID | Asset | Format | Dimensions/Size | Stored As | Used By |
|----|-------|--------|-----------------|-----------|---------|
| A1 | Receipt source photo | JPEG | ~1200px wide, 400–600KB | base64 string | POST `/cal/receipt/scan` body |
| A2 | Receipt processed image | JPEG | ~800KB | base64 string | `receipt-cache.js` IndexedDB `full` field |
| A3 | Receipt thumbnail | JPEG | 120px wide, ~40KB, quality=0.6 | base64 string | `receipt-cache.js` IndexedDB `thumbnail` field |
| A4 | Receipt PDF | PDF | A4 page, ~500KB | Buffer | GET `/cal/receipt/{id}/pdf` response |
| A5 | Street view photo | JPEG | 180×180px, ~15–30KB | ArrayBuffer | `bb-streetview-cache` IndexedDB |
| A6 | Tool scan photo | JPEG | ~300×300px, ~300KB | base64 string | POST `/assets/tools/scan` body |
| A7 | Tool catalog photo | JPEG/PNG | varies | file on disk | GET `/api/assets/media/{id}/thumb` |
| A8 | Neon BYTEA blobs | JPEG bytes | full=~800KB, thumb=~40KB | Node Buffer | `cal_receipts.processed_image`, `.thumbnail` columns |

---

### Fixture Pool — Download Once, Never Regenerate at Runtime

**File:** `synthetic/download-fixtures.js`

**Run condition:** ONLY run this script manually when setting up the harness for the first time, or when fixtures need refreshing. It is NOT called by the harness runner. All fixture files are checked into the repo.

**What it downloads/generates:**

```js
// synthetic/download-fixtures.js
// AGENT: implement this file exactly as described.
// Dependencies: node-fetch, @napi-rs/canvas, pdf-lib, sharp
// Run: node synthetic/download-fixtures.js
// Output: populates synthetic/fixtures/ — check ALL output into repo

import { createCanvas } from '@napi-rs/canvas';
import { PDFDocument } from 'pdf-lib';
import { writeFileSync, mkdirSync } from 'fs';
import path from 'path';

// 1. RECEIPT FIXTURES (20 source, 20 processed, 20 thumbs)
// Generate via receipt-forge.js (see Generator A below).
// Vendors: pull from templates/appliance-catalog.json (hardware, lumber, electrical)
// Amounts: $12.50 – $847.00
// Dates: 2025-09-01 through 2026-03-31
// Write to: fixtures/receipts/source/receipt-001.jpg ... receipt-020.jpg
//           fixtures/receipts/processed/receipt-001.jpg ... receipt-020.jpg
//           fixtures/receipts/thumbs/receipt-001.jpg ... receipt-020.jpg

// 2. STREET VIEW FIXTURES (50 tiles)
// Fetch from Bridge proxy: GET /api/properties/streetview/{lat}/{lng}
// Use lat/lng values from templates/addresses-bainbridge.json (first 50 entries)
// Write to: fixtures/streetview/sv-001.jpg ... sv-050.jpg
// Write index: fixtures/streetview/meta.json = { "{lat6}_{lng6}": "sv-001.jpg", ... }
// IMPORTANT: requires BRIDGE_URL and OPENAI_API_KEY env vars
// Fallback if Bridge unavailable: generate placeholder via canvas (grey gradient 180×180)

// 3. TOOL PHOTO FIXTURES (30 photos)
// Source: Creative Commons images from Wikimedia Commons (safe to check in)
// Categories: hammer(5), drill(5), tape_measure(5), level(4), saw(4), wrench(4), other(3)
// Resize to 300×300 via sharp before saving
// Write to: fixtures/tools/tool-001.jpg ... tool-030.jpg
// Write index: fixtures/tools/catalog.json = { "hammer": ["tool-001.jpg",...], ... }

// 4. PDF FIXTURES (10 files)
// Generate via pdf-forge.js (see Generator D below)
// Write to: fixtures/pdfs/receipt-001.pdf ... receipt-010.pdf
```

**meta.json format** (agents must produce exactly this shape):
```json
{
  "47.625650_-122.520100": "sv-001.jpg",
  "47.626210_-122.519830": "sv-002.jpg"
}
```

**catalog.json format:**
```json
{
  "hammer": ["tool-001.jpg", "tool-002.jpg"],
  "drill": ["tool-003.jpg", "tool-004.jpg"],
  "tape_measure": ["tool-005.jpg"],
  "level": ["tool-006.jpg"],
  "saw": ["tool-007.jpg"],
  "wrench": ["tool-008.jpg"],
  "other": ["tool-009.jpg"]
}
```

---

### Generator A: Receipt Image Forge

**File:** `synthetic/engine-3-asset-forge/receipt-forge.js`

**Dependencies:** `@napi-rs/canvas` (Node native canvas — NOT browser canvas, NOT jimp). Install: `npm install @napi-rs/canvas`.

**Inputs:**
```js
// receiptForge(options) → { sourceBuffer, processedBuffer, thumbBuffer }
{
  vendor: 'Ace Hardware',          // string — from appliance-catalog.json
  amount: 156.32,                  // number
  date: '2026-03-14',              // ISO date string
  items: [                         // array, 2–8 items
    { name: 'Wood screws 1"', price: 4.99 },
    { name: 'Paint roller kit', price: 23.47 }
  ],
  seed: 42,                        // integer — controls all randomness (REQUIRED for reproducibility)
  exif: {
    orientation: 1,                // 1|3|6|8 — JPEG EXIF orientation tag
    make: 'Apple',                 // 'Apple'|'Samsung'|'Google'
    model: 'iPhone 14'
  }
}
```

**Output contract:** returns `{ sourceBuffer: Buffer, processedBuffer: Buffer, thumbBuffer: Buffer }` — all valid JPEG Buffers, no base64 encoding inside this function.

**Implementation steps (agent must follow in order):**

```
Step 1 — Canvas setup:
  const canvas = createCanvas(900, 1400)   // portrait receipt shape
  const ctx = canvas.getContext('2d')

Step 2 — White background + margin:
  ctx.fillStyle = '#FAFAFA'
  ctx.fillRect(0, 0, 900, 1400)

Step 3 — Store header (top 200px):
  ctx.fillStyle = '#1A1A1A'
  ctx.font = 'bold 48px sans-serif'
  ctx.fillText(vendor.toUpperCase(), 60, 80)
  ctx.font = '28px sans-serif'
  ctx.fillText('123 Main St, Bainbridge Island WA 98110', 60, 130)
  ctx.fillText(`Date: ${date}`, 60, 175)

Step 4 — Divider line:
  ctx.strokeStyle = '#CCCCCC'
  ctx.lineWidth = 2
  ctx.beginPath(); ctx.moveTo(40, 210); ctx.lineTo(860, 210); ctx.stroke()

Step 5 — Line items (starting at y=250, 60px per row):
  for each item:
    ctx.fillText(item.name, 60, y)                        // left-aligned
    ctx.fillText(`$${item.price.toFixed(2)}`, 840, y)     // right-aligned

Step 6 — Subtotal / Tax / Total block (bottom of items):
  subtotal = sum(items)
  tax = subtotal * 0.1025  // WA state sales tax
  ctx.fillText('Subtotal', 60, y)
  ctx.fillText(`$${subtotal.toFixed(2)}`, 840, y)
  ctx.fillText('Tax (10.25%)', 60, y+50)
  ctx.fillText(`$${tax.toFixed(2)}`, 840, y+50)
  ctx.font = 'bold 40px sans-serif'
  ctx.fillText('TOTAL', 60, y+120)
  ctx.fillText(`$${amount.toFixed(2)}`, 840, y+120)

Step 7 — Camera noise (controlled by seed — use seeded PRNG, e.g. simple mulberry32):
  a) Slight perspective skew: ctx.transform(1, 0, tilt, 1, 0, 0)
     tilt = seededRandom(seed) * 0.04 - 0.02  // ±0.02 radians
  b) Edge vignette: radialGradient from corners, rgba(0,0,0,0) to rgba(0,0,0,0.15)
  c) Brightness variation: globalAlpha = 0.85 + seededRandom(seed+1) * 0.15

Step 8 — Export source JPEG:
  sourceBuffer = canvas.toBuffer('image/jpeg', { quality: 0.88 })

Step 9 — Inject EXIF into sourceBuffer:
  Use piexifjs (npm install piexifjs) to inject:
    Orientation: exif.orientation
    Make: exif.make
    Model: exif.model
    DateTime: `${date} 09:${(seed%60).toString().padStart(2,'0')}:00`
  sourceBuffer = piexifjs.insert(piexifjs.dump(exifData), sourceBuffer)

Step 10 — processedBuffer: same canvas, no skew, full brightness (server "corrected" version)
  Repeat steps 1–6 without step 7
  processedBuffer = canvas.toBuffer('image/jpeg', { quality: 0.92 })

Step 11 — thumbBuffer: scale processedBuffer down to 120px wide
  const thumbCanvas = createCanvas(120, Math.round(120 * 1400/900))
  thumbCtx.drawImage(image, 0, 0, 120, thumbCanvas.height)
  thumbBuffer = thumbCanvas.toBuffer('image/jpeg', { quality: 0.6 })
```

**Seeded PRNG (copy this exactly — no external library):**
```js
// mulberry32 — deterministic, seed-based, no imports needed
function mulberry32(seed) {
  return function() {
    seed |= 0; seed = seed + 0x6D2B79F5 | 0;
    let t = Math.imul(seed ^ seed >>> 15, 1 | seed);
    t = t + Math.imul(t ^ t >>> 7, 61 | t) ^ t;
    return ((t ^ t >>> 14) >>> 0) / 4294967296;
  };
}
// Usage: const rng = mulberry32(42); const val = rng();
```

**DONE WHEN:**
- `receipt-forge.js` exports a `receiptForge(options)` function
- Returns `{ sourceBuffer, processedBuffer, thumbBuffer }` — all valid JPEG Buffers
- Same `seed` always produces identical pixel output (verified by SHA-256 hash comparison)
- `sourceBuffer` passes `jpeg-js` decode without error
- EXIF orientation tag present in `sourceBuffer` (verify with `piexifjs.load()`)

---

### Generator B: Street View Forge

**File:** `synthetic/engine-3-asset-forge/streetview-forge.js`

**Strategy:** Serve from fixture pool only. No API calls at runtime.

```js
// streetviewForge(lat, lng) → Buffer (JPEG, 180×180)
// AGENT: implement exactly this lookup logic

import { readFileSync } from 'fs';
import path from 'path';
import { createCanvas } from '@napi-rs/canvas';

const FIXTURES_DIR = path.resolve('synthetic/fixtures/streetview');
let meta = null;  // lazy-load

function loadMeta() {
  if (!meta) meta = JSON.parse(readFileSync(path.join(FIXTURES_DIR, 'meta.json'), 'utf8'));
  return meta;
}

export function streetviewForge(lat, lng) {
  const key = `${lat.toFixed(6)}_${lng.toFixed(6)}`;
  const m = loadMeta();

  // 1. Exact match in fixture pool
  if (m[key]) {
    return readFileSync(path.join(FIXTURES_DIR, m[key]));
  }

  // 2. Nearest fixture by hash (deterministic fallback — no API calls)
  const keys = Object.keys(m);
  const idx = Math.abs(hashString(key)) % keys.length;
  const nearestKey = keys[idx];
  return readFileSync(path.join(FIXTURES_DIR, m[nearestKey]));
}

// Fallback if fixture pool is empty (should never happen after download-fixtures.js runs)
export function streetviewPlaceholder(lat, lng, seed) {
  const canvas = createCanvas(180, 180);
  const ctx = canvas.getContext('2d');
  const rng = mulberry32(seed);
  // Sky: blue-grey gradient top half
  const sky = ctx.createLinearGradient(0, 0, 0, 90);
  sky.addColorStop(0, `hsl(210,${20+rng()*20}%,${60+rng()*20}%)`);
  sky.addColorStop(1, `hsl(210,15%,75%)`);
  ctx.fillStyle = sky; ctx.fillRect(0, 0, 180, 90);
  // Building: grey rectangle centre
  ctx.fillStyle = `hsl(0,0%,${50+rng()*20}%)`;
  ctx.fillRect(30, 40, 120, 100);
  // Windows: 2×3 grid
  ctx.fillStyle = '#87CEEB';
  for (let r=0;r<2;r++) for (let c=0;c<3;c++)
    ctx.fillRect(40+c*35, 50+r*35, 20, 22);
  // Road: dark grey bottom
  ctx.fillStyle = '#555'; ctx.fillRect(0, 140, 180, 40);
  return canvas.toBuffer('image/jpeg', { quality: 0.82 });
}

function hashString(s) {
  let h = 0;
  for (let i = 0; i < s.length; i++) h = Math.imul(31, h) + s.charCodeAt(i) | 0;
  return h;
}
```

**DONE WHEN:**
- `streetviewForge(lat, lng)` returns a valid JPEG Buffer for any lat/lng
- Same lat/lng always returns identical bytes
- Returns fixture pool image when available, deterministic fallback when not
- Fixture pool has ≥50 entries after `download-fixtures.js` runs

---

### Generator C: Tool Photo Forge

**File:** `synthetic/engine-3-asset-forge/tool-photo-forge.js`

**Strategy:** Select from fixture pool by category. Overlay brand text for variety.

```js
// toolPhotoForge({ category, brand, seed }) → Buffer (JPEG)
// AGENT: implement exactly this

import { readFileSync } from 'fs';
import path from 'path';
import { createCanvas, loadImage } from '@napi-rs/canvas';

const FIXTURES_DIR = path.resolve('synthetic/fixtures/tools');
let catalog = null;

function loadCatalog() {
  if (!catalog) catalog = JSON.parse(readFileSync(path.join(FIXTURES_DIR, 'catalog.json'), 'utf8'));
  return catalog;
}

export async function toolPhotoForge({ category, brand = '', seed = 0 }) {
  const cat = loadCatalog();
  const options = cat[category] ?? cat['other'];
  const rng = mulberry32(seed);
  const filename = options[Math.floor(rng() * options.length)];
  const base = readFileSync(path.join(FIXTURES_DIR, filename));

  // Load base image, add brand text overlay
  const img = await loadImage(base);
  const canvas = createCanvas(img.width, img.height);
  const ctx = canvas.getContext('2d');
  ctx.drawImage(img, 0, 0);

  if (brand) {
    // Brand label: white pill bottom-left
    ctx.fillStyle = 'rgba(255,255,255,0.82)';
    ctx.fillRect(8, img.height - 34, brand.length * 9 + 16, 28);
    ctx.fillStyle = '#1A1A1A';
    ctx.font = 'bold 16px sans-serif';
    ctx.fillText(brand.toUpperCase(), 16, img.height - 14);
  }

  return canvas.toBuffer('image/jpeg', { quality: 0.88 });
}
```

**DONE WHEN:**
- Returns valid JPEG Buffer for any `{ category, brand, seed }` combination
- Same inputs → identical output
- Brand overlay visible in rendered JPEG

---

### Generator D: PDF Receipt Forge

**File:** `synthetic/engine-3-asset-forge/pdf-forge.js`

**Dependencies:** `pdf-lib` (pure JS, no native deps). Install: `npm install pdf-lib`.

```js
// pdfReceiptForge({ vendor, amount, date, receiptImageBuffer }) → Buffer (PDF)
// AGENT: implement exactly this

import { PDFDocument, StandardFonts, rgb } from 'pdf-lib';

export async function pdfReceiptForge({ vendor, amount, date, receiptImageBuffer }) {
  const pdfDoc = await PDFDocument.create();

  // Metadata
  pdfDoc.setTitle(`Receipt - ${vendor} - ${date}`);
  pdfDoc.setAuthor('BB Scan');
  pdfDoc.setCreationDate(new Date(date));

  const page = pdfDoc.addPage([595, 842]); // A4 points
  const font = await pdfDoc.embedFont(StandardFonts.Helvetica);
  const boldFont = await pdfDoc.embedFont(StandardFonts.HelveticaBold);

  // Embed receipt JPEG image
  const jpgImage = await pdfDoc.embedJpg(receiptImageBuffer);
  const { width, height } = jpgImage.scale(0.55);  // scale to fit page
  page.drawImage(jpgImage, {
    x: (595 - width) / 2,
    y: 842 - height - 60,
    width,
    height,
  });

  // Text header above image
  page.drawText(`${vendor.toUpperCase()}`, {
    x: 40, y: 810, size: 18, font: boldFont, color: rgb(0.1, 0.1, 0.1)
  });
  page.drawText(`Date: ${date}    Amount: $${amount.toFixed(2)}`, {
    x: 40, y: 788, size: 11, font, color: rgb(0.3, 0.3, 0.3)
  });
  page.drawText('Generated by BB Scan', {
    x: 40, y: 30, size: 9, font, color: rgb(0.6, 0.6, 0.6)
  });

  const pdfBytes = await pdfDoc.save();
  return Buffer.from(pdfBytes);
}
```

**DONE WHEN:**
- Returns valid PDF Buffer (verify with `pdf-lib` re-parse: `PDFDocument.load(output)` must not throw)
- PDF contains embedded JPEG image
- PDF text layer contains vendor, date, amount (verifiable via text extraction)

---

### Engine 1 Addition: BYTEA Column Generation

**File:** `synthetic/engine-1-relational/generators/receipt-image-generator.js`

Engine 1 seeds Neon rows. The `cal_receipts` table has `processed_image BYTEA` and `thumbnail BYTEA` columns. Engine 1 must write real JPEG bytes, not NULLs.

```js
// AGENT: add this to engine-1-relational/generators/
// Called by seeder.js when generating cal_receipts rows

import { receiptForge } from '../../engine-3-asset-forge/receipt-forge.js';

export async function generateReceiptImages(receiptData, seed) {
  // receiptData: { vendor, amount, date, items[] }
  const { sourceBuffer, processedBuffer, thumbBuffer } = await receiptForge({
    ...receiptData,
    seed,
    exif: {
      orientation: [1, 3, 6, 8][seed % 4],
      make: ['Apple', 'Samsung', 'Google'][seed % 3],
      model: ['iPhone 14', 'Galaxy S23', 'Pixel 8'][seed % 3]
    }
  });
  return {
    processed_image: processedBuffer,   // Buffer → Neon writes as BYTEA
    thumbnail: thumbBuffer              // Buffer → Neon writes as BYTEA
  };
}

// Neon insert (in seeder.js):
// await db.execute(
//   `INSERT INTO cal_receipts (vendor, amount, receipt_date, processed_image, thumbnail, ...)
//    VALUES ($1, $2, $3, $4, $5, ...)`,
//   [vendor, amount, date, processed_image, thumbnail, ...]
// );
// Note: @neondatabase/serverless accepts Buffer directly for BYTEA params.
```

**DONE WHEN:** Every seeded `cal_receipts` row has non-NULL `processed_image` and `thumbnail` columns containing valid JPEG bytes.

---

### IndexedDB Seeder — Playwright Fixture

**File:** `e2e/fixtures/seed-indexeddb.js`

**Purpose:** Prime browser IndexedDB before E2E tests run. Without this, every test hits the network for cached assets — making tests slow, flaky, and order-dependent.

**When to use:** Called from every E2E test file that exercises receipt history, street view, or tool crib UI.

```js
// AGENT: implement exactly this Playwright fixture
// Usage in test files:
//   import { seedIndexedDB } from '../fixtures/seed-indexeddb.js';
//   test.beforeEach(async ({ page }) => { await seedIndexedDB(page, seedData); });

import { readFileSync } from 'fs';
import path from 'path';

// Pre-built seed payload — generated once by seeder.js, written to:
// synthetic/fixtures/idb-seed.json
// Contains base64-encoded JPEG strings and ArrayBuffer data for IDB injection.
// Agents: do NOT inline large base64 strings here. Load from file.

export async function seedIndexedDB(page, options = {}) {
  const seedFile = path.resolve('synthetic/fixtures/idb-seed.json');
  const seedData = JSON.parse(readFileSync(seedFile, 'utf8'));

  // Step 1: Inject seed payload into page context before load
  await page.addInitScript((data) => {
    window.__HARNESS_IDB_SEED__ = data;
  }, seedData);

  // Step 2: After page load, populate IndexedDB programmatically
  await page.waitForLoadState('domcontentloaded');

  await page.evaluate(async () => {
    const data = window.__HARNESS_IDB_SEED__;
    if (!data) throw new Error('IDB seed not injected');

    function openDB(name, version, upgradeCallback) {
      return new Promise((resolve, reject) => {
        const req = indexedDB.open(name, version);
        if (upgradeCallback) req.onupgradeneeded = (e) => upgradeCallback(e.target.result);
        req.onsuccess = (e) => resolve(e.target.result);
        req.onerror = (e) => reject(e.target.error);
      });
    }
    function putRecord(db, storeName, record, key) {
      return new Promise((resolve, reject) => {
        const tx = db.transaction(storeName, 'readwrite');
        const req = key !== undefined
          ? tx.objectStore(storeName).put(record, key)
          : tx.objectStore(storeName).put(record);
        req.onsuccess = () => resolve();
        req.onerror = (e) => reject(e.target.error);
      });
    }

    // Seed bb-receipt-cache (receipts store)
    const rcDb = await openDB('bb-receipt-cache', 1, (db) => {
      if (!db.objectStoreNames.contains('images'))
        db.createObjectStore('images', { keyPath: 'receiptId' });
    });
    for (const r of data.receipts) {
      await putRecord(rcDb, 'images', {
        receiptId: r.id,
        thumbnail: r.thumb_base64,   // base64 string
        full: r.full_base64,         // base64 string
        savedAt: Date.now()
      });
    }

    // Seed bb-streetview-cache (photos store, keyed by "lat_lng")
    const svDb = await openDB('bb-streetview-cache', 1, (db) => {
      if (!db.objectStoreNames.contains('photos'))
        db.createObjectStore('photos');
    });
    for (const sv of data.streetviews) {
      // sv.array_buffer is base64 string — convert back to ArrayBuffer
      const binary = atob(sv.array_buffer_b64);
      const buf = new Uint8Array(binary.length);
      for (let i = 0; i < binary.length; i++) buf[i] = binary.charCodeAt(i);
      await putRecord(svDb, 'photos', {
        data: buf.buffer,
        savedAt: Date.now()
      }, sv.key);   // key = "47.625650_-122.520100"
    }
  });
}
```

**idb-seed.json format** (written by `seeder.js` after generating assets):
```json
{
  "receipts": [
    {
      "id": "receipt-001",
      "thumb_base64": "/9j/4AAQSkZJRgAB...",
      "full_base64": "/9j/4AAQSkZJRgAB..."
    }
  ],
  "streetviews": [
    {
      "key": "47.625650_-122.520100",
      "array_buffer_b64": "/9j/4AAQSkZJRgAB..."
    }
  ]
}
```

**DONE WHEN:**
- `seedIndexedDB(page)` runs without throwing in a Playwright test
- After calling it, `page.evaluate(() => indexedDB.databases())` shows `bb-receipt-cache` and `bb-streetview-cache` exist
- Receipt history UI renders thumbnails without any network requests (verify via `page.on('request', ...)` — no `/api/cal/receipt/.*/thumb` requests should fire)
- Street view images render on property cards without network requests

---

### Mock AI Server — Asset Route Contracts

**File:** `scale/mock-ai-server.js` — add these routes

All routes are **deterministic**: same input → same output, always. No randomness at request time. This is non-negotiable for reproducible test results.

```js
// AGENT: add these routes to the existing mock-ai-server.js Fastify instance
// All routes pull from the fixture pool — no generation at request time.

import { readFileSync, readdirSync } from 'fs';
import path from 'path';

const FIXTURES = path.resolve('synthetic/fixtures');

// Deterministic fixture selector — same input always picks same file
function pickFixture(dir, inputHash) {
  const files = readdirSync(dir).filter(f => f.endsWith('.jpg') || f.endsWith('.jpeg'));
  if (files.length === 0) throw new Error(`No fixtures in ${dir}`);
  return path.join(dir, files[Math.abs(inputHash) % files.length]);
}

function hashString(s) {
  let h = 0;
  for (let i = 0; i < s.length; i++) h = Math.imul(31, h) + s.charCodeAt(i) | 0;
  return h;
}

// Route 1: Receipt scan — returns realistic receipt analysis + processed image
fastify.post('/cal/receipt/scan', async (req, reply) => {
  const { image = '', crewName = '', dateHint = '' } = req.body ?? {};
  const h = hashString(image.slice(0, 64));  // hash first 64 chars of base64
  const imgFile = pickFixture(path.join(FIXTURES, 'receipts/processed'), h);
  const imgBuf = readFileSync(imgFile);
  const processedImage = imgBuf.toString('base64');

  // Deterministic vendor/amount from hash
  const vendors = ['Ace Hardware', 'Home Depot', 'Lowe\'s', 'Fastenal', 'Pacific Lumber'];
  const vendor = vendors[Math.abs(h) % vendors.length];
  const amount = ((Math.abs(h) % 8000) + 500) / 100;  // $5.00–$84.99

  return reply.send({
    id: `scan-${Math.abs(h).toString(16).slice(0,8)}`,
    processedImage,
    vendor,
    amount,
    receipt_date: dateHint || '2026-03-14',
    jobcode_match: 'Bainbridge - Renovation',
    confidence: { vendor: 0.95, amount: 0.91, date: 0.88 }
  });
});

// Route 2: Street view proxy
fastify.get('/api/properties/streetview/:lat/:lng', async (req, reply) => {
  const { lat, lng } = req.params;
  const meta = JSON.parse(readFileSync(path.join(FIXTURES, 'streetview/meta.json'), 'utf8'));
  const key = `${parseFloat(lat).toFixed(6)}_${parseFloat(lng).toFixed(6)}`;
  const filename = meta[key] ?? Object.values(meta)[Math.abs(hashString(key)) % Object.values(meta).length];
  const buf = readFileSync(path.join(FIXTURES, 'streetview', filename));
  return reply.type('image/jpeg').send(buf);
});

// Route 3: Receipt thumbnail
fastify.get('/api/cal/receipt/:id/thumb', async (req, reply) => {
  const h = hashString(req.params.id);
  const file = pickFixture(path.join(FIXTURES, 'receipts/thumbs'), h);
  return reply.type('image/jpeg').send(readFileSync(file));
});

// Route 4: Receipt PDF
fastify.get('/api/cal/receipt/:id/pdf', async (req, reply) => {
  const h = hashString(req.params.id);
  const files = readdirSync(path.join(FIXTURES, 'pdfs')).filter(f => f.endsWith('.pdf'));
  const file = path.join(FIXTURES, 'pdfs', files[Math.abs(h) % files.length]);
  return reply.type('application/pdf').send(readFileSync(file));
});

// Route 5: Tool photo thumbnail (by Drive file ID)
fastify.get('/api/assets/media/:driveFileId/thumb', async (req, reply) => {
  const h = hashString(req.params.driveFileId);
  const file = pickFixture(path.join(FIXTURES, 'tools'), h);
  return reply.type('image/jpeg').send(readFileSync(file));
});

// Route 6: Tool scan — returns deterministic tool identification
fastify.post('/assets/tools/scan', async (req, reply) => {
  const { image = '' } = req.body ?? {};
  const h = hashString(image.slice(0, 64));
  const tools = [
    { name: 'DeWalt DCD771C2 Drill', brand: 'DeWalt', model: 'DCD771C2', category: 'drill' },
    { name: 'Milwaukee M18 Hammer Drill', brand: 'Milwaukee', model: 'M18', category: 'drill' },
    { name: 'Stanley FatMax Hammer', brand: 'Stanley', model: 'FatMax', category: 'hammer' },
    { name: 'Stanley 25ft Tape Measure', brand: 'Stanley', model: '33-425', category: 'tape_measure' },
    { name: 'Empire Level 48"', brand: 'Empire', model: 'E55.48', category: 'level' },
  ];
  return reply.send({
    tool: tools[Math.abs(h) % tools.length],
    confidence: 0.91
  });
});
```

**DONE WHEN:**
- All 6 routes return valid binary responses (JPEG or PDF) without throwing
- Same request always returns same response (deterministic)
- Routes respond in <50ms (reading from disk, no generation)
- `GET /api/properties/streetview/47.625650/-122.520100` returns a valid 180×180 JPEG

---

### Dependency List for Engine 3 (package.json additions)

Agents must add exactly these packages. No alternatives without explicit approval:

```json
"@napi-rs/canvas": "^0.1.53",
"pdf-lib": "^1.17.1",
"piexifjs": "^1.0.6",
"sharp": "^0.33.3"
```

**Why each:**
- `@napi-rs/canvas`: Node-native canvas (works on Windows without browser). Used by receipt-forge, streetview-forge, tool-photo-forge.
- `pdf-lib`: Pure JS PDF creation, no native deps, Windows-safe. Used by pdf-forge.
- `piexifjs`: EXIF read/write for JPEG buffers. Used by receipt-forge to inject orientation + camera metadata.
- `sharp`: High-performance image resizing for tool photo preparation in download-fixtures.js.

**Do NOT use:** `node-canvas` (requires Cairo native build, breaks on Windows), `canvas` (same), `pdfkit` (different API), `jimp` (too slow for batch generation).

---

### Seeder Integration (How Engine 3 Hooks Into Engine 1)

`seeder.js` must call Engine 3 generators when seeding rows that require binary assets. Execution order:

```
1. seeder.js generates relational row data (vendor, amount, date, items) via Engine 1
2. For each cal_receipts row:
   a. Call receiptForge({ ...rowData, seed: rowIndex })
   b. Get { processedBuffer, thumbBuffer }
   c. Insert row with processed_image = processedBuffer, thumbnail = thumbBuffer
3. After all rows inserted:
   a. Build idb-seed.json:
      - For each receipt: base64-encode processedBuffer and thumbBuffer
      - For each property in addresses-bainbridge.json: call streetviewForge(lat, lng), base64-encode result
   b. Write synthetic/fixtures/idb-seed.json
4. seeder.js prints: "IDB seed written → synthetic/fixtures/idb-seed.json (N receipts, M streetviews)"
```

**idb-seed.json is gitignored** — it is generated fresh from the fixture pool on each `seeder.js` run. The fixture pool files (jpgs, pdfs) ARE checked in.

---

## Success and Failure Cases

**Success:** Track 0 runs nightly on Sam's Windows 11 PC, produces SESSION SUMMARY showing all 6 stages passing, cost under $5/run, no manual intervention.

**Failure:** Any stage blocks Track A advancement. Blocks are hard (deterministic_fail / policy_fail) or release-blocking (golden_miss). LLM warnings (llm_warn) are advisory only.

**Recovery:** If a stage fails, fix is committed immediately, stage re-run, results updated. No "it passed before" — every run validates current state.

---

*Track 0 Delivery Plan v1.0. See core architecture for decisions; see crew platform for feature roadmap.*
