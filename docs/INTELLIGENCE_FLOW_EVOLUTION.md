# Intelligence Flow Evolution: Full Analysis & Next Generation Design

> Generated: 2026-02-23 | Codex peer-review applied: 2026-02-23
> Scope: Complete reverse-engineering of the current intelligence flow, redundancy audit, and optimal redesign

---

## Table of Contents
1. [Current System: What Exists](#1-current-system)
2. [Meta-Ralph vs EIDOS: The Two Gates](#2-meta-ralph-vs-eidos)
3. [Redundancy Audit: What's Done Twice](#3-redundancy-audit)
4. [Tuneable Audit: What to Keep, Merge, Kill](#4-tuneable-audit)
5. [18 Bypass Paths: Where Stages Get Skipped](#5-bypass-paths)
6. [8 Sequence Issues: Wrong Order of Operations](#6-sequence-issues)
7. [Codex Peer Review: Corrections & New Findings](#7-codex-review)
8. [Next Evolution: The Carmack Design (Revised)](#8-next-evolution)

---

## 1. Current System

### Bridge Cycle (Write Path) — 20 Sub-Stages, ALL Optional

Every stage wrapped in `try/except` that logs and continues. No stage blocks the next.

```
1.  Load events from queue
2.  Classify events (single-pass)
3.  Memory capture (importance_score, AUTO_SAVE if >= 0.65)
4.  Meta-Ralph quality check (only on captured memories)
5.  Cognitive learner injection (batch mode, deferred validation)
6.  EIDOS episode management
7.  EIDOS distillation (LLM-based, every 5th cycle)
8.  PatternDistiller (rule-based distillation)
9.  Chip processing
10. Mind sync
11. Prediction tracking
12. Opportunity scanning
13. Promotion evaluation
14. Stats/telemetry
15-20. Cleanup, rotation, save state
```

### Pre-Tool Advisory (Read Path) — Current Sequence

```
1.  Load state (shown_advice, last_tool, etc.)
2.  Generate trace_id
3.  Record tool call for telemetry
4.  Check packet store for pre-computed advice
5.  If no packet → advisor.advise() (12+ sources, ranked)
6.  Gate evaluation (10 filters: shown_ttl, tool_cooldown, budget)
7.  Global dedup check (redundant with gate's shown_ttl)
8.  LLM synthesis (100-300ms) — runs even if items will be deduped
9.  Text repeat check — runs AFTER synthesis
10. Emit advice
11. Mark as shown + save state
12. Track retrieval for feedback
13. Log and save
14. Safety check — runs AFTER all side effects
```

---

## 2. Meta-Ralph vs EIDOS: The Two Gates

### They Are NOT Redundant — They Serve Different Purposes

| Aspect | Meta-Ralph | EIDOS Distillation |
|--------|-----------|-------------------|
| **What it evaluates** | Proposed cognitive learnings (text insights) | Rules extracted from episodes (bounded problem-solving) |
| **Input** | Plain text strings from observations/patterns | Completed episodes with steps and outcomes, OR observation lists |
| **Processing** | 7-dimension scoring + auto-refinement + dedup | LLM-based analysis (every 5th cycle) OR rule-based reflection |
| **Quality criteria** | Score >= 4.5/12 across actionability, novelty, reasoning, tradeoff, temporal, domain, self-evidence | Basic validation (length, noise patterns) + advisory quality transformer |
| **Output** | RoastResult verdict (QUALITY/NEEDS_WORK/REJECTED/PRIMITIVE/DUPLICATE) | Distillation object (HEURISTIC/SHARP_EDGE/ANTI_PATTERN/PLAYBOOK/POLICY) |
| **Storage** | cognitive_insights.json | eidos_distillations.jsonl + eidos.db |
| **Deduplication** | Semantic hash matching (prevents same insight twice) | None (each distillation is unique by episode) |
| **Feedback loop** | Records outcomes on stored learnings | Tracks times_used, times_helped, validation_count |
| **Rigor** | VERY HIGH (multi-dimensional, catches tautologies, primitives) | MEDIUM (basic validation, advisory transformer) |
| **When it runs** | Every insight before CognitiveLearner stores it | Every 5th bridge cycle OR after episode completion |

### Data Flow — No Cross-Filtering

```
Observations/Patterns ──→ Meta-Ralph ──→ Cognitive Insights Store
                                              │
Episodes/Aggregates ──→ EIDOS Distillation ──→ EIDOS Store
                                              │
                          Both feed into ──→ Advisory Engine (RRF fusion)
```

Meta-Ralph does NOT evaluate EIDOS distillations. EIDOS does NOT evaluate Meta-Ralph learnings. They are complementary — Meta-Ralph catches "is this learning sound?" while EIDOS captures "what reusable rules did we learn from this experience?"

### Unique Value Each Provides

**Meta-Ralph only:**
- 7-dimensional scoring (catches tautologies, platitudes, vague advice)
- Auto-refinement (improves close-call insights instead of rejecting)
- Semantic deduplication (prevents same insight stored twice)
- Primitive pattern filtering (knows operational noise patterns)

**EIDOS only:**
- Episode context (bounded problems with goals and budgets)
- Causal inference (because X happened, Y resulted)
- Recovery patterns (failure → success sequences)
- Multi-step playbooks (not just single rules)
- LLM-powered nuance (understands context Meta-Ralph's regex misses)

### Where Improvement Is Needed

EIDOS distillation has a **weaker quality gate** than Meta-Ralph:
- Basic validation: length >= 24 chars, not matching noise patterns
- Advisory quality transformer (secondary check)
- But NO 7-dimensional scoring, NO tautology detection, NO dedup

**Recommendation**: Add a `MetaRalph.validate_insight(text, source="eidos")` call for EIDOS distillations before storage. This closes bypass #10 without merging the systems.

---

## 3. Redundancy Audit: What's Done Twice

### 3.1 Noise Filtering — 5 Separate Pattern Lists

| Module | File | Patterns | Scope |
|--------|------|----------|-------|
| Meta-Ralph | meta_ralph.py | ~17 PRIMITIVE_PATTERNS + ~19 QUALITY_SIGNALS | Pre-storage quality gate |
| Cognitive Learner | cognitive_learner.py | 41 patterns in `_is_noise_insight()` | Pre-cognitive storage |
| Primitive Filter | primitive_filter.py | ~10 patterns in `is_primitive_text()` | Standalone utility |
| Advisory Gate | advisory_gate.py | Score-based (no pattern list) | Post-retrieval filtering |
| EIDOS | bridge_cycle.py | _EIDOS_NOISE_PATTERNS | Pre-distillation storage |

**Problem**: 5 modules each maintain their OWN noise pattern lists. No shared constant. Copy-paste across files.
**Fix**: Extract `lib/noise_patterns.py` — single source of truth for all noise/quality patterns.

### 3.2 Storage — 38+ Files With Overlapping Data

**Critical duplications:**

| Data Type | Files | Overlap |
|-----------|-------|---------|
| Feedback | `advice_feedback.jsonl` + `advice_feedback_requests.jsonl` + `advisor/implicit_feedback.jsonl` | 3 files tracking same advice→outcome |
| Outcomes | `outcomes.jsonl` + `outcome_links.jsonl` + `meta_ralph/outcome_tracking.json` | 3 files tracking tool outcomes |
| Advisory logs | `advisory_engine.jsonl` + `advisory_decision_ledger.jsonl` + `advisory_emit.jsonl` | 3 operational logs of same decisions |
| Dedup logs | `advisory_low_auth_dedupe.jsonl` + `advisory_global_dedupe.jsonl` | Both tracking dedup events |
| Chip data | `chip_insights/*.jsonl` + `chip_learning_distillations.jsonl` | Chip knowledge in 2 places |

**Fix**: Consolidate to canonical streams:
- 1 feedback file (unified schema with explicit + implicit fields)
- 1 outcome file (tool_name, success, trace_id, advice_ids)
- 1 advisory operational log (replaces 3)

### 3.3 Feedback Loops — 8 Separate Systems

| System | Purpose | Authoritative? |
|--------|---------|----------------|
| advice_feedback.py | Explicit "was this helpful?" | Partial |
| advisory_packet_feedback.py | Packet-level feedback | Partial |
| implicit_outcome_tracker.py | Links advice→tool outcome | Partial |
| feedback_loop.py | General orchestration | Orchestrator only |
| meta_ralph.py outcome_tracking | Quality gate validation | For Meta-Ralph only |
| feedback_effectiveness_cache.py | Performance cache | Reader only |
| aha_tracker.py | "Aha" moments | Signal only |
| engagement_tracker.py | Engagement metrics | Growth only |

**Problem**: No single source of truth. 3 systems track "did advice help?" from different angles. Inconsistent state possible.
**Fix**: Single `advisory_feedback.jsonl` with unified schema. Multiple readers, one writer.

### 3.4 Retrieval — 15 Sources Reading from 6 Stores

| Source Function | Underlying Data | Overlap |
|----------------|-----------------|---------|
| `_get_cognitive_advice()` | cognitive_insights.json | |
| `_get_semantic_cognitive_advice()` | cognitive_insights.json (semantic index) | SAME data as above |
| `_get_cognitive_advice_keyword()` | cognitive_insights.json (keyword) | SAME data as above |
| `_get_eidos_advice()` | eidos_distillations.jsonl | |
| `_get_chip_advice()` | chip_insights/*.jsonl | Similar to EIDOS |
| `_get_bank_advice()` | memory_banks | |
| `_get_mind_advice()` | mind_bridge (external) | |
| `_get_tool_specific_advice()` | trigger_rules.yaml | |
| `_get_replay_counterfactual_advice()` | recent_advice.jsonl | |
| 6 more... | various | |

**Problem**: 3 sources reading same cognitive_insights.json with different retrieval strategies.
**Fix**: Single `_get_cognitive_advice()` with multi-strategy fallback (semantic → keyword → exhaustive).

### 3.5 Configuration — Dual Control Surfaces

- `config/tuneables.json` (version-controlled defaults) + `~/.spark/tuneables.json` (live runtime)
- 97 environment variables that OVERRIDE tuneables silently
- 20+ modules each have their own config loader (identical pattern)
- Code defaults ≠ config defaults in multiple places

**Fix**: Centralized config registry with env var documentation.

### 3.6 Code Duplication

| Pattern | Occurrences | Fix |
|---------|-------------|-----|
| Config loading `_load_*_config()` | 20+ modules | Extract `lib/config_loader.py` |
| Env parsing `_env_float/_env_int/_env_bool` | 3 implementations | Extract `lib/env_utils.py` |
| Advisory log writing | 3 files, different formats | Extract unified JSONL writer |
| Noise pattern detection | 5 files | Extract `lib/noise_patterns.py` |

---

## 4. Tuneable Audit: What to Keep, Merge, Kill

### 4.1 Kill List — Remove These (Verified Dead by Both Analyses + Codex Grep)

| Tuneable | Reason | Verified |
|----------|--------|----------|
| `values.min_occurrences_critical` | Never read by any code | Codex confirmed |
| `advisor.source_weights` | Only logged, never used for weighting | Codex confirmed |
| `eidos.max_steps` | DUPLICATE of `values.max_steps` | Both analyses agree |

### 4.1b Keep List — Codex Corrected (NOT Dead)

| Tuneable | Why We Thought Dead | Why It's Actually Live |
|----------|--------------------|-----------------------|
| `promotion.confidence_floor` | "Dead code path" | LIVE: `promoter.py:374, 397` |
| `synthesizer.cache_ttl_s` | "Synthesizer doesn't cache" | LIVE: `advisory_synthesizer.py:849` |
| `synthesizer.max_cache_entries` | "Synthesizer doesn't cache" | LIVE: `advisory_synthesizer.py:877` |
| `advisory_prefetch.*` (8 keys) | "Prefetch disabled" | WIRED: `advisory_engine.py:2671`, `advisory_prefetch_worker.py:51, 65` |
| `advisory_packet_store.packet_lookup_llm_*` (8 keys) | "Feature gated OFF" | WIRED: `advisory_packet_store.py:3022`, `advisory_packet_llm_reranker.py:204` |
| `values.advice_cache_ttl` | "Conflicts with advisor.cache_ttl" | Backward-compat fallback: `advisor.py:711` |

### 4.2 Merge List — Consolidate These

#### Cooldowns: 4 Systems → 3 (Unified Naming, Distinct Semantics)

> Codex correction: These 4 serve genuinely different purposes (text-level vs ID-level vs state-level vs cross-session). Merging to 2 would lose semantics. Instead: keep 3 layers, unify config naming.

**Current (4 overlapping, scattered naming):**
```
A. advisory_engine.advisory_text_repeat_cooldown_s = 420s (ENV OVERRIDES to 600s)
B. advisory_gate.advice_repeat_cooldown_s = 420s
C. advisory_gate.shown_advice_ttl_s = 420s (+ source_ttl_multipliers + category_cooldown_multipliers)
D. advisory_engine.global_dedupe_cooldown_s = 240s (ENV OVERRIDES to 600s)
```

**Proposed (3 layers, unified under `advisory_suppression`):**
```
Layer 1: advisory_suppression.text_repeat_cooldown_s = 420s
  - Exact text match (prevents same words re-emitted)
  - Merges A into unified section

Layer 2: advisory_suppression.shown_advice_ttl_s = 420s
  - Shown-state marker (advice_id + tool + phase)
  - Absorbs B (same advice_id) + C (shown-state) — these were already the same concept
  - Keep source_ttl_multipliers + category_cooldown_multipliers as sub-controls

Layer 3: advisory_suppression.global_cross_session_cooldown_s = 600s
  - Cross-session dedup (D + low_auth_global)

All env var overrides documented with deprecation warnings.
```

#### Source Ranking: Keep Structure, Tighten Bounds

> Codex correction: Core `_rank_score()` is already additive (`advisor.py:4891`). Multiplicative behavior only applies in category/noise penalties and auto-tuner composition. The "compound penalty" problem is real but the fix is tighter bounds, not architectural change.

**Current:**
```
Stage 1: advisor.min_reliability = 0.62 (pre-filter)
Stage 2: auto_tuner.source_boosts.{source} = 0.3-1.6 (bounded multiplicative)
Stage 3: advisor.min_rank_score = 0.45 (additive gate)
```

**Problem**: Boost range 0.3-1.6 is too wide. Sources with low effectiveness get penalized so hard they can never recover.

**Proposed (same structure, tighter bounds):**
```
Stage 1: advisor.min_reliability = 0.6 (pre-filter, unchanged)
Stage 2: auto_tuner.source_boosts bounded 0.8-1.1 (was 0.2-3.0)
         - Worst penalty: 0.8x (mild dampening, not silence)
         - Best boost: 1.1x (mild amplification, not dominance)
         - Sources can always recover from low effectiveness
Stage 3: advisor.min_rank_score = 0.45 (additive gate, unchanged)
```

#### Max Items: 2 → 1

**Current:**
- `advisor.max_items` = 3 (config: 4) — retrieval cap
- `advisor.max_advice_items` = 10 (config: 5) — emission cap? DRIFT from config

**Proposed:**
- `advisor.max_advice_items` = 5 (single control)

#### Duplicate EIDOS/Values Keys: 4 duplicates → 0

Move `max_steps`, `max_retries_per_error`, `max_file_touches`, `no_evidence_steps` exclusively to `eidos` section. Remove from `values`.

### 4.3 Keep List — These Are Well-Designed

| Section | Status | Notes |
|---------|--------|-------|
| `meta_ralph.*` | KEEP ALL | Well-used, no redundancy |
| `retrieval.*` | KEEP ALL | Elegant domain profiles |
| `production_gates.*` | KEEP ALL | Health checking |
| `observatory.*` | KEEP ALL | Export config |
| `promotion.*` (minus dead keys) | KEEP | After removing confidence_floor |

### 4.4 Environment Variable Strategy

**97 env vars currently shadow tuneables.** The env var wins silently.

**Phase 1**: Centralize in `lib/tuneable_env_overrides.py` — document all mappings
**Phase 2**: Deprecation warnings when env var overrides tuneable
**Phase 3**: Remove env shadowing — tuneables.json is the single control surface

---

## 5. Bypass Paths: Where Stages Get Skipped

18 paths where intelligence reaches advisory without proper quality gates:

| # | Path | What Gets Skipped | Severity | Source |
|---|------|-------------------|----------|--------|
| 1 | Pre-tool budget exhaustion → quick fallback | ALL gates | HIGH | advisory_engine.py |
| 2 | Advisory engine budget exhaustion → live quick fallback | Gate + synthesis | MEDIUM* | advisory_engine.py:1556 |
| 3 | Packet fallback emit after gate rejection | Gate overridden | HIGH | advisory_engine.py |
| 4 | Memory capture AUTO_SAVE (importance >= 0.65) | Meta-Ralph | MEDIUM | memory_capture.py |
| 5 | Cognitive batch mode deferred validation | Meta-Ralph timing | LOW | cognitive_learner.py |
| 6 | User prompt cognitive signals → direct learner write | Meta-Ralph | MEDIUM | cognitive_signals.py |
| 7 | Edit/Write content learning → direct store | Meta-Ralph | MEDIUM | memory_capture.py:417 |
| 8 | Validation loop outcome recording | Meta-Ralph | LOW | |
| 9 | Chip insight processing (own thresholds) | Meta-Ralph | MEDIUM | bridge_cycle.py |
| 10 | EIDOS distillation → direct JSONL append | Meta-Ralph | MEDIUM | bridge_cycle.py:1069 |
| 11 | LLM advisory synthesis without cognitive filtering | Cognitive noise filter | MEDIUM | |
| 12 | Advisory engine no-advice path | Telemetry gap | LOW | |
| 13 | Advice feedback loop records without validation | Quality gate | LOW | feedback_loop.py:159 |
| 14 | Session failure tracking auto-records | Quality gate | LOW | |
| 15 | Async prefetch populates packet store unvalidated | ALL gates | HIGH | advisory_engine.py:2671 |
| 16 | **Hook-level legacy fallback** (Codex finding) | ALL engine gates | HIGH | observe.py:593, 602 |
| 17 | **Direct non-roasted cognitive writes** (Codex finding) | Meta-Ralph | MEDIUM | feedback_loop.py:159, chip_merger.py:662, aggregator.py:462 |
| 18 | **Distillation floor explicit roast bypass** (Codex finding) | Meta-Ralph | MEDIUM | pipeline.py:838 |

*Bypass #2 partially phantom per Codex: quick advice object is built but `advise_on_tool()` runs immediately after and route resets to live.*

---

## 6. Sequence Issues: Wrong Order of Operations

8 issues in the current `on_pre_tool()` ordering:

1. **Synthesis before repeat check** — LLM synthesis (100-300ms) runs, THEN text repeat check kills the result. Wasted compute.
2. **Double filtering** — Gate evaluates shown_ttl, then separate global_dedupe re-checks. Redundant.
3. **Text repeat check after synthesis** — Should be BEFORE retrieval (cheapest check first).
4. **Synthesis on empty results** — If global dedupe removes all items after gate, synthesis already ran on them.
5. **Three overlapping dedup layers** — shown_ttl (gate) + global_dedupe (engine) + text_repeat (engine).
6. **Safety check after side effects** — State is saved, logs written, THEN safety check runs. Too late.
7. **Fragile multiplicative boosts** — Gate evaluates with compound boost interactions (hard to debug).
8. **Biased retrieval tracking** — Only tracks emitted items, creating feedback loop bias in auto-tuner.

---

## 7. Codex Peer Review: Corrections & New Findings

> Independent Codex review of this document and the full codebase. Corrections applied below.

### Validated (Codex Agrees)

- Write path is fail-open/optional — confirmed via `_run_step()` pattern (`bridge_cycle.py:214, 303, 616`)
- Read path sequence issues are real — synthesis before dedup (`advisory_engine.py:2058, 2116`), safety after side effects (`advisory_engine.py:2200, 2454`)
- Gate/engine dedupe overlap confirmed — gate has shown TTL/tool cooldown (`advisory_gate.py:613, 653`), engine re-runs global dedupe/text repeat (`advisory_engine.py:1920, 2116`)
- Meta-Ralph vs EIDOS correctly classified as complementary — confirmed via code (`meta_ralph.py:733`, `bridge_cycle.py:1069, 1093`)

### Corrections to Our Analysis

**Bypass #2 is partly phantom**: The "live_quick_fallback" builds a quick advice object, but `advise_on_tool()` still runs immediately after and route resets to live (`advisory_engine.py:1556, 1559, 1569`). Not a clean bypass — more of a pre-warm.

**Kill list has false positives** — these are NOT dead:
- `promotion.confidence_floor` — LIVE, read by promoter (`promoter.py:374, 397`)
- `synthesizer.cache_ttl_s / max_cache_entries` — LIVE, used by advisory_synthesizer (`advisory_synthesizer.py:849, 877`)
- `packet_lookup_llm_*` (8 keys) — feature-flagged but actively wired (`advisory_packet_store.py:3022`, `advisory_packet_llm_reranker.py:204`)
- `values.advice_cache_ttl` — still used as backward-compat fallback (`advisor.py:711`)

**Prefetch keys are NOT dead**: Prefetch is still wired even when disabled — `on_user_prompt` enqueues, inline worker calls consume (`advisory_engine.py:2671`, `advisory_prefetch_worker.py:51, 65`).

**Source ranking is NOT purely multiplicative**: Core `_rank_score()` is actually additive (`advisor.py:4891`). Multiplicative behavior only comes from category/noise penalties and auto-tuner source-quality composition (`advisor.py:800, 4904`). Our proposal to "fix" multiplicative → additive was solving a non-problem.

### New Bypass Paths Found by Codex

| # | Path | What Gets Skipped | Severity | Source |
|---|------|-------------------|----------|--------|
| 16 | Hook-level legacy bypass: if advisory engine throws, observe.py falls back to legacy advisor path | All engine gates | HIGH | `observe.py:593, 602` |
| 17 | Direct non-roasted cognitive writes from feedback_loop, chip_merger, aggregator, memory_capture | Meta-Ralph | MEDIUM | `feedback_loop.py:159`, `chip_merger.py:662`, `aggregator.py:462`, `memory_capture.py:417` |
| 18 | Distillation floor explicitly bypasses roast gate | Meta-Ralph | MEDIUM | `pipeline.py:838` |

### New Bugs Found

**Critical**: `cognitive_signals.py:283` passes `advisory_quality=` to `add_insight()`, but the signature (`cognitive_learner.py:1473`) does NOT accept that kwarg. This path likely errors and is silently swallowed. **Fixed in Batch 2 — removed invalid kwarg.**

**Dead code**: `_low_auth_recently_emitted()` has zero call sites (`advisory_engine.py:215`) but the low-auth dedupe log is still written (`advisory_engine.py:2285`).

**Race condition**: `outcome_links.jsonl` has two independent writers: `outcome_log.py:19,137` and `outcomes/linker.py:19,145`. Several append-only JSONL writers are lockless in hot paths (`advice_feedback.py:278`, `implicit_outcome_tracker.py:100`, `advisory_emitter.py:260`, `advisory_engine.py:172`).

### Design Challenges from Codex

**"Don't remove fallbacks entirely"**: During retrieval/gate outages, removing fallbacks drives emit rate to near-zero. Keep fallbacks but hard-rate-limit and metric-tag them. Fallbacks exist for graceful degradation (`advisory_engine.py:1646, 1855`).

**"Don't use retry queue — use quarantine"**: Fail-closed + retry queue adds operational complexity and backlog risks. Better: fail-open for session UX, quarantine failed writes, run async repair/replay job. Bridge is designed fail-open (`bridge_cycle.py:214`).

**"Don't merge 4 cooldowns to 2"**: shown TTL, tool cooldown, text-repeat, and cross-session global dedupe are different units and horizons. Alternative: keep 3 layers, unify config surface and naming (`advisory_gate.py:613, 653`, `advisory_engine.py:2116, 1920`).

**"Don't go purely additive on ranking"**: Bounded multiplicative penalties (0.8-1.1 range) are intentional for hard-penalizing unreliable sources. Removing them loses deliberate discrimination.

**"Tuneable kill list is not safe as written"**: Only provably safe removals are `advisor.source_weights` (no runtime consumer) and `values.min_occurrences_critical` (schema only, not wired).

### Codex Recommended Batch Order

1. **Batch 0 (NEW — MUST HAVE)**: Safety-net tests + telemetry baselines BEFORE any behavior changes
2. **Batch 1**: Read path reorder (validated as safe with dependency-order preserved)
3. **Batch 2**: Write path — `validate_and_store_insight()` unified path (highest-impact single change)
4. **Batch 3**: Fallback rate-limiting (not removal)
5. **Batch 4**: Storage consolidation (after read/write stabilized)
6. **Batch 5**: Tuneable naming unification (not structural merge)
7. **Batch 6**: Code dedup (lowest risk, incremental)

### What Codex Says NOT to Change
- Meta-Ralph/EIDOS dual-system model (correctly complementary)
- Fail-open bridge behavior for non-critical stages
- Additive core rank model in `_rank_score()`

---

## 8. Next Evolution: The Carmack Design (Revised)

> "The right stuff at the right places at the right times. Not less, not more."
> Revised after Codex peer review — incorporates corrections on fallbacks, cooldowns, ranking.

### Design Principles (Revised)

1. **One job per module** — No module does two things
2. **One path per data type** — All insight writes go through `validate_and_store_insight()`
3. **One config surface** — tuneables.json with unified naming; env vars documented but deprecated
4. **Cheap checks first** — Before spending 100ms on LLM synthesis, check if we even need to
5. **Fail-open with quarantine** — Critical gates quarantine failures for async repair, never block the session
6. **Single feedback stream** — One file tracks all advice effectiveness, multiple readers
7. **Rate-limited fallbacks** — Keep fallback emits for graceful degradation, but budget-cap and metric-tag them

### The Optimal Write Path (Bridge Cycle)

```
Queue Events
    │
    ▼
┌─────────────────────┐
│ 1. CLASSIFY          │  Single-pass event classification
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 2. IMPORTANCE SCORE  │  Score 0-1, all items proceed (no AUTO_SAVE bypass)
│    memory_capture.py │
└─────────┬───────────┘
          │
          ▼
┌──────────────────────────────────────────────────────────────┐
│ 3. validate_and_store_insight()  — SINGLE UNIFIED ENTRY POINT │
│                                                                │
│    a) Meta-Ralph roast (score >= 4.5 → pass)                  │
│       Exception → quarantine + async repair (fail-open)        │
│       Reject → logged with reason + rejection telemetry        │
│                                                                │
│    b) Cognitive noise filter (41 patterns + tautology)         │
│       Exception → quarantine + async repair (fail-open)        │
│       Reject → logged with pattern match                       │
│                                                                │
│    c) Store in cognitive_insights.json                          │
└─────────┬────────────────────────────────────────────────────┘
          │ (only validated items)
          │
    ┌─────┴─────┐
    │           │
    ▼           ▼
┌────────┐  ┌──────────────┐
│ EIDOS  │  │ CHIPS        │  Both route through validate_and_store_insight()
│        │  │              │  (closes bypass #9, #10, #17, #18)
└────┬───┘  └──────┬───────┘
     │             │
     └──────┬──────┘
            │
            ▼
┌─────────────────────┐
│ 4. PROMOTION EVAL   │  Reads only from validated stores
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 5. STATS + CLEANUP  │  Includes rejection + quarantine counters
└─────────────────────┘
```

**Changes from current:**
- Remove AUTO_SAVE bypass (closes #4)
- All insight writes funnel through `validate_and_store_insight()` (closes #5-10, #17, #18)
- Fail-OPEN with quarantine: exceptions don't block the session, failed writes queued for async repair
- EIDOS/Chips/feedback_loop/chip_merger/aggregator all use the same entry point
- Rejection telemetry on every path

### The Optimal Read Path (Pre-Tool Advisory)

```
Tool Call Arrives
    │
    ▼
┌─────────────────────┐
│ 1. LOAD STATE        │  shown_advice, last_tool
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 2. SAFETY CHECK      │  FIRST — before any side effects
│    (moved from #14)  │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 3. TEXT REPEAT CHECK │  Cheap string compare. If same → early exit
│    (moved from #9)   │  Saves entire retrieval + synthesis
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 4. TOOL COOLDOWN     │  Cheap timestamp check. If cooling → early exit
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 5. BUDGET CHECK      │  Counter check. If exhausted → rate-limited fallback
│                      │  Fallback: max 1 per 5 calls, metric-tagged
│                      │  (NOT removed — graceful degradation per Codex review)
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 6. RETRIEVAL         │  Packet store lookup OR advisor.advise()
│                      │  12 sources → ranked list
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 7. GATE              │  Keep 3 distinct suppression layers (per Codex):
│                      │  Layer 1: shown_ttl (advice_id + tool + phase)
│                      │  Layer 2: tool cooldown (per-tool family)
│                      │  Layer 3: cross-session global dedupe
│                      │  But UNIFIED config naming + single evaluation pass
│                      │  + dedup-by-ID absorbed into gate (no separate check)
│                      │  Budget cap: dynamic 2-4
│                      │  Result: 0-4 guaranteed-novel items
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 8. SYNTHESIS         │  LLM synthesis ONLY if items > 0
│    (moved from #8)   │  No wasted compute on empty/deduped results
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ 9. EMIT + TRACK      │  Emit → mark shown → track retrieval → log
│                      │  Packet fallback: rate-limited (max 1 per 10 calls)
│                      │  metric-tagged as "fallback_emit" for auto-tuner
└─────────────────────┘
```

**Changes from current:**
- Safety check moved to top (correct ordering)
- Text repeat check before retrieval (saves 100-300ms)
- Budget exhaustion: rate-limited fallback instead of removal (Codex correction)
- Global dedupe absorbed into gate (single evaluation pass, but 3 semantic layers preserved)
- Synthesis only runs when items exist and are novel
- Packet fallback: kept but rate-limited and metric-tagged (Codex correction)

### The Effectiveness Loop (NEW — Closing the Circle)

After advisory emission, the system needs to learn what worked. Currently 8 separate feedback systems track this inconsistently. The optimal design:

```
Advice Emitted
    │
    ▼
┌─────────────────────┐
│ TOOL EXECUTES        │  User runs the tool (Read, Edit, Bash, etc.)
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ IMPLICIT TRACKER     │  Single system records:
│                      │  - advice_id
│                      │  - tool_name + tool_success
│                      │  - was advice followed? (heuristic)
│                      │  - trace_id for linking
│                      │  → Writes to advisory_feedback.jsonl (SINGLE FILE)
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ EFFECTIVENESS CACHE  │  Reads advisory_feedback.jsonl
│                      │  Computes per-source effectiveness
│                      │  14-day exponential decay
│                      │  Used by: auto-tuner, advisor ranking
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ AUTO-TUNER           │  Reads effectiveness cache
│                      │  Adjusts source_boosts (additive, not multiplicative)
│                      │  Bounded: min_boost=0.8, max_boost