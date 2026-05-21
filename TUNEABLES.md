# Spark Intelligence Tuneable Parameters

All configurable thresholds, limits, and weights across the system.
Use this to test and optimize learning quality.
Navigation hub: `docs/GLOSSARY.md`

---

## Configuration Precedence (Canonical)

Spark runtime now resolves tuneables using a single authority model:

1. `lib/tuneables_schema.py` defaults
2. `config/tuneables.json` baseline
3. `~/.spark/tuneables.json` runtime overrides
4. Explicit env overrides (allowlisted per key)

Reference: `docs/CONFIG_AUTHORITY.md`

Core runtime sections now routed through this model:
- `advisory_engine`, `advisory_gate`, `advisor`, `synthesizer`, `semantic`, `triggers`
- `meta_ralph`, `eidos`, `promotion`, `memory_emotion`, `memory_learning`, `memory_retrieval_guard`
- `bridge_worker`, `queue`, `pipeline`, `values`
- `advisory_packet_store`, `advisory_prefetch`, `sync`, `production_gates`
- `chip_merge`, `memory_capture`, `request_tracker`, `observatory`, `advisory_preferences`

---

## 0. Advisor Retrieval Router (Carmack Path)

**File:** `lib/advisor.py`

This controls when advisor stays on fast embeddings retrieval versus escalating to hybrid-agentic retrieval.

Canonical routing tuneables surface:
- Live: `~/.spark/tuneables.json` -> `retrieval.overrides.*`
- Benchmark overlays: also use `retrieval.overrides.*` (same schema)

### Core Strategy

- Fast path first: semantic retrieval on primary query.
- `auto` mode default gate (minimal):
  - escalate on weak primary count
  - escalate on weak primary top score
  - escalate on high-risk query terms
- Bounded escalation:
  - rate cap (`agentic_rate_limit`)
  - hard deadline (`agentic_deadline_ms`)

### Parameters

| Parameter | Default (Level 2) | Description |
|-----------|-------------------|-------------|
| `retrieval.level` | `"2"` | Profile baseline (`1` local-free, `2` balanced, `3` quality-max). |
| `retrieval.overrides.mode` | `auto` | `auto`, `embeddings_only`, or `hybrid_agentic`. |
| `retrieval.overrides.gate_strategy` | `minimal` | `minimal` uses weak_count/weak_score/high_risk; `extended` also uses complexity+trigger gates. |
| `retrieval.overrides.min_results_no_escalation` | `4` | If primary result count is below this, escalate. |
| `retrieval.overrides.min_top_score_no_escalation` | `0.72` | If primary top fusion score is below this, escalate. |
| `retrieval.overrides.escalate_on_high_risk` | `true` | Escalate when high-risk terms are present. |
| `retrieval.overrides.escalate_on_trigger` | `false` (L2) | Trigger-based escalation (mostly for extended/high-quality profiles). |
| `retrieval.overrides.agentic_rate_limit` | `0.20` | Max fraction of recent queries allowed to escalate agentically. |
| `retrieval.overrides.agentic_rate_window` | `80` | Rolling window size for rate cap. |
| `retrieval.overrides.agentic_deadline_ms` | `700` | Deadline for agentic facet fanout; stop on timeout. |
| `retrieval.overrides.fast_path_budget_ms` | `250` | Target budget marker for primary retrieval path telemetry. |
| `retrieval.overrides.prefilter_enabled` | `true` | Enables metadata/token prefilter before semantic retrieval. |
| `retrieval.overrides.prefilter_max_insights` | `500` | Max candidate insights after prefilter. |
| `retrieval.overrides.semantic_limit` | `10` | Number of semantic candidates returned from each retrieval call. |
| `retrieval.overrides.max_queries` | `3` | Max total retrieval queries (primary + facets). |
| `retrieval.overrides.agentic_query_limit` | `3` | Max extracted facet queries before clipping by `max_queries`. |
| `retrieval.overrides.lexical_weight` | `0.28` | Weight applied to lexical blend during rerank. |
| `retrieval.overrides.bm25_k1` | `1.2` | BM25 TF saturation parameter. |
| `retrieval.overrides.bm25_b` | `0.75` | BM25 length normalization parameter. |
| `retrieval.overrides.bm25_mix` | `0.75` | Blend ratio: BM25 vs overlap lexical signal. |
| `retrieval.overrides.semantic_context_min` | `0.18` | Minimum semantic similarity to treat a candidate as a context match. |
| `retrieval.overrides.semantic_lexical_min` | `0.05` | Minimum lexical overlap to keep a candidate when semantic similarity is weak. |
| `retrieval.overrides.semantic_strong_override` | `0.92` | If semantic similarity is this strong, keep the candidate even if lexical overlap is weak. |

---
## 0.5 Memory Emotion Fusion

**Files:** `lib/memory_banks.py`, `lib/memory_store.py`

This controls how emotional state is attached to memory writes and reused as a retrieval rerank signal.

Tuneable surface:
- Live: `~/.spark/tuneables.json` -> `memory_emotion.*`
- Environment overrides:
  - `SPARK_MEMORY_EMOTION_WRITE_CAPTURE`
  - `SPARK_MEMORY_EMOTION_ENABLED`
  - `SPARK_MEMORY_EMOTION_WEIGHT`
  - `SPARK_MEMORY_EMOTION_MIN_SIM`
  - `SPARK_ADVISORY_MEMORY_EMOTION_ENABLED`
  - `SPARK_ADVISORY_MEMORY_EMOTION_WEIGHT`
  - `SPARK_ADVISORY_MEMORY_EMOTION_MIN_SIM`

### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `memory_emotion.enabled` | `true` | Master switch for retrieval-time emotion/state rerank in memory retrieval. |
| `memory_emotion.write_capture_enabled` | `true` | Attach current emotion snapshot (`meta.emotion`) when writing memory-bank entries. |
| `memory_emotion.retrieval_state_match_weight` | `0.22` | Additive score weight applied to state similarity during retrieval rerank. |
| `memory_emotion.retrieval_min_state_similarity` | `0.30` | Minimum similarity required before state-match contributes to score. |
| `memory_emotion.advisory_rerank_weight` | `0.15` | Additive weight for emotion-state similarity in live advisory semantic reranking. |
| `memory_emotion.advisory_min_state_similarity` | `0.30` | Minimum similarity threshold before advisory rerank applies emotion boost. |

### When to Tune

| Scenario | Adjustment |
|----------|------------|
| Emotion signal overpowers relevance | Lower `retrieval_state_match_weight` (e.g. `0.10-0.18`) |
| Emotion signal has no practical effect | Raise `retrieval_state_match_weight` (e.g. `0.30-0.45`) |
| Too many weak emotional matches | Raise `retrieval_min_state_similarity` (e.g. `0.45`) |
| Want broader emotional recall | Lower `retrieval_min_state_similarity` (e.g. `0.15-0.25`) |

---
## 1. Memory Gate (Pattern → EIDOS)

**File:** `lib/pattern_detection/memory_gate.py`

The Memory Gate decides which Steps and Distillations are worth persisting to long-term memory. It prevents noise from polluting the knowledge base by scoring each item against multiple quality signals.

### How It Works

Every Step or Distillation is scored from 0.0 to 1.0+ based on weighted signals. Only items scoring above the `threshold` are persisted.

```
Final Score = Σ(signal_present × signal_weight)
```

### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `threshold` | **0.5** | **The gate cutoff.** Items scoring below this are discarded. At 0.5, an item needs at least 2-3 positive signals to pass. |
| `WEIGHTS["impact"]` | 0.30 | **Progress signal.** Did this action unblock progress or advance toward the goal? High when a stuck situation was resolved. |
| `WEIGHTS["novelty"]` | 0.20 | **New pattern signal.** Is this something we haven't seen before? Detects first-time tool combinations, new error types, or unique approaches. |
| `WEIGHTS["surprise"]` | 0.30 | **Prediction error signal.** Did the outcome differ from what was predicted? Surprises indicate learning opportunities - the system's model was wrong. |
| `WEIGHTS["recurrence"]` | 0.20 | **Frequency signal.** Has this pattern appeared 3+ times? Recurring patterns are likely stable and worth remembering. |
| `WEIGHTS["irreversible"]` | 0.60 | **Stakes signal.** Is this a high-stakes action (production deploy, security change, data deletion)? Irreversible actions get dominant weight because mistakes are costly. Raised from 0.40. |
| `WEIGHTS["evidence"]` | 0.10 | **Validation signal.** Is there concrete evidence (test pass, user confirmation) supporting this? Evidence-backed items are more trustworthy. |

### Scoring Examples

**High score (passes gate):**
```
Step: "Fixed authentication bug by adding token refresh"
- impact: 0.30 (unblocked login flow)
- surprise: 0.30 (expected different root cause)
- evidence: 0.10 (tests now pass)
Total: 0.70 ✓ PASSES
```

**Low score (rejected):**
```
Step: "Read config file"
- novelty: 0.0 (common action)
- impact: 0.0 (no progress made)
- surprise: 0.0 (expected outcome)
Total: 0.0 ✗ REJECTED
```

### When to Tune

| Scenario | Adjustment |
|----------|------------|
| Too much noise in memory | Raise `threshold` to 0.6-0.7 |
| Missing important learnings | Lower `threshold` to 0.4 |
| Want more emphasis on errors | Raise `surprise` weight |
| Learning too slowly | Lower `recurrence` weight |
| High-stakes project (finance, security) | Weight already at 0.60, raise to 0.7+ if needed |

---

## 2. Pattern Distiller

**File:** `lib/pattern_detection/distiller.py`

The Pattern Distiller analyzes completed Steps to extract reusable rules (Distillations). It looks for patterns in successes, failures, and user behavior to create actionable guidance.

### How It Works

1. Collects completed Steps from the Request Tracker
2. Groups by pattern type (user preferences, tool usage, surprises)
3. Requires minimum evidence before creating a Distillation
4. Passes Distillations through Memory Gate before storage

### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `min_occurrences` | **2** | **Evidence threshold.** A pattern must appear at least this many times before being distilled into a rule. Lowered from 3 for faster learning. |
| `min_occurrences_critical` | **1** | **Fast-track for CRITICAL tier.** Critical importance items (explicit "remember this", corrections) are learned from a single occurrence. |
| `min_confidence` | **0.6** | **Success rate threshold.** For heuristics (if X then Y), the pattern must have worked at least 60% of the time. Filters out unreliable patterns. |
| `gate_threshold` | **0.5** | **Memory gate threshold** (inherited from Memory Gate). Distillations must score above this to be stored. |

### Distillation Types Created

| Type | What It Captures | Example |
|------|------------------|---------|
| `HEURISTIC` | "When X, do Y" patterns | "When file not found, check path case sensitivity first" |
| `ANTI_PATTERN` | "Don't do X because Y" | "Don't use sed on Windows - syntax differs" |
| `SHARP_EDGE` | Gotchas and pitfalls | "Python venv activation differs between shells" |
| `PLAYBOOK` | Multi-step procedures | "To debug imports: 1. Check PYTHONPATH, 2. Verify __init__.py" |
| `POLICY` | User-defined rules | "Always run tests before committing" |

### When to Tune

| Scenario | Adjustment |
|----------|------------|
| Learning too slowly | Lower `min_occurrences` to 1 |
| Distillations are unreliable | Raise `min_occurrences` to 4-5 |
| Too many weak heuristics | Raise `min_confidence` to 0.7-0.8 |
| Missing edge case patterns | Lower `min_confidence` to 0.5 |
| Want more one-shot learning | Lower `min_occurrences_critical` (already at 1) |

---

## 3. Request Tracker

**File:** `lib/pattern_detection/request_tracker.py`

The Request Tracker wraps every user request in an EIDOS Step envelope, tracking the full lifecycle from intent → action → outcome. This creates the structured data needed for learning.

### How It Works

```
User Message → Step Created (with intent, hypothesis, prediction)
     ↓
Action Taken → Step Updated (with decision, tool used)
     ↓
Outcome Observed → Step Completed (with result, evaluation, lesson)
```

### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `max_pending` | **50** | **Concurrent request limit.** Maximum unresolved requests being tracked. Prevents memory bloat from abandoned requests. When exceeded, oldest pending requests are dropped. |
| `max_completed` | **200** | **Completed history limit.** How many completed Steps to retain for distillation analysis. Older completed Steps are pruned. |
| `max_age_seconds` | **3600** | **Timeout (1 hour).** Pending requests older than this are auto-closed as "timed_out". Prevents zombie requests from lingering forever. |

### When to Tune

| Scenario | Adjustment |
|----------|------------|
| Long-running sessions with many requests | Raise `max_pending` to 100 |
| Memory-constrained environment | Lower both limits |
| Want more history for distillation | Raise `max_completed` to 500 |
| Requests timing out too quickly | Raise `max_age_seconds` to 7200 (2 hours) |

---

## 4. Pattern Aggregator

**File:** `lib/pattern_detection/aggregator.py`

The Pattern Aggregator coordinates all pattern detectors (correction, sentiment, repetition, semantic, why) and routes detected patterns to the learning system. It's the central hub for pattern detection.

### How It Works

```
Event → All Detectors Run → Patterns Collected → Corroboration Check → Learning Triggered
                                    ↓
                         (Every N events) → Distillation Run
```

### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `CONFIDENCE_THRESHOLD` | **0.6** | **Learning trigger threshold.** Patterns must have at least 60% confidence to trigger learning. Lowered from 0.7 to let importance scorer do quality filtering. |
| `DEDUPE_TTL_SECONDS` | **600** | **Deduplication window (10 min).** The same pattern won't be processed twice within this window. Prevents spammy patterns from flooding the system. |
| `DISTILLATION_INTERVAL` | **20** | **Batch size for distillation.** After every 20 events processed, the distiller runs to analyze completed Steps. Lower = more frequent distillation. |

### Corroboration Boost

When multiple detectors agree, confidence is boosted:
- Correction + Frustration detected together → +15% confidence
- Repetition + Frustration detected together → +10% confidence

### When to Tune

| Scenario | Adjustment |
|----------|------------|
| Missing subtle patterns | Lower `CONFIDENCE_THRESHOLD` to 0.6 |
| Too many false positives | Raise `CONFIDENCE_THRESHOLD` to 0.8 |
| Same insight appearing repeatedly | Raise `DEDUPE_TTL_SECONDS` to 1800 (30 min) |
| Want faster learning cycles | Lower `DISTILLATION_INTERVAL` to 10 |
| System too slow | Raise `DISTILLATION_INTERVAL` to 50 |

---

## 5. EIDOS Budget (Episode Limits)

**File:** `lib/eidos/models.py` → `Budget` class

The EIDOS Budget enforces hard limits on episodes to prevent rabbit holes. When any limit is exceeded, the episode transitions to DIAGNOSE or HALT phase.

### How It Works

These are **circuit breakers** - when tripped, they force the system to stop and reassess rather than continuing blindly.

### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `max_steps` | **25** (code) / **40** (tuneables) | **Step limit per episode.** After N actions without completing the goal, force DIAGNOSE phase. Canonical key is `values.max_steps` (legacy `eidos.max_steps` may still be read if present in older runtime files). |
| `max_time_seconds` | **720** (code) / **1200** (tuneables) | **Time limit.** Episodes taking longer than this are force-stopped. Wired to `eidos.max_time_seconds`. |
| `max_retries_per_error` | **2** (code) / **3** (tuneables) | **Error retry limit.** Wired to `eidos.max_retries_per_error` (also reads `values.max_retries_per_error`). |
| `max_file_touches` | **3** (code) / **5** (tuneables) | **File modification limit.** Wired to `eidos.max_file_touches` (also reads `values.max_file_touches`). |
| `no_evidence_limit` | **5** (code) / **6** (tuneables) | **Evidence requirement.** After N steps without new evidence, force DIAGNOSE. Wired to `eidos.no_evidence_limit` (also reads `values.no_evidence_steps`). |

### What Happens When Limits Hit

| Limit Exceeded | Transition | Behavior |
|----------------|------------|----------|
| `max_steps` | → HALT | Episode ends, escalate to user |
| `max_time_seconds` | → HALT | Episode ends, escalate to user |
| `max_retries_per_error` | → DIAGNOSE | Stop modifying, only observe |
| `max_file_touches` | → DIAGNOSE | File frozen, must find another approach |
| `no_evidence_limit` | → DIAGNOSE | Must gather evidence before acting |

### When to Tune

| Scenario | Adjustment |
|----------|------------|
| Complex tasks need more steps | Raise `max_steps` to 40 |
| Want faster failure detection | Lower `max_steps` to 15 |
| Legitimate long-running tasks | Raise `max_time_seconds` to 1800 (30 min) |
| Frequent file thrashing | Lower `max_file_touches` to 1 |
| Tasks require iteration | Raise `max_file_touches` to 3 |

---

## 6. EIDOS Watchers

**File:** `lib/eidos/control_plane.py`

Watchers are real-time monitors that detect specific stuck patterns. When triggered, they force phase transitions to break out of unproductive loops.

### How It Works

Each watcher monitors a specific metric. When the threshold is exceeded, it fires an alert that triggers a phase transition (usually to DIAGNOSE).

### Watchers

| Watcher | Threshold | What It Detects | Response |
|---------|-----------|-----------------|----------|
| **Repeat Error** | **2** | Same error signature appearing twice. | → DIAGNOSE. Stop modifying, investigate root cause. |
| **No New Info** | **5** | Five consecutive steps without gathering new evidence. | → DIAGNOSE. Must read/test before acting. |
| **Diff Thrash** | **4** | Same file modified four times (after max_file_touches=3). | → SIMPLIFY. Freeze file, find alternative. |
| **Confidence Stagnation** | **0.05 × 3** | Confidence delta < 5% for three steps. | → PLAN. Step back, reconsider approach. |
| **Memory Bypass** | **1** | Action taken without citing retrieved memory. | BLOCK. Must acknowledge memory or declare absent. |
| **Budget Half No Progress** | **50%** | Budget >50% consumed with no progress. | → SIMPLIFY. Reduce scope, focus on core. |
| **Scope Creep** | varies | Plan grows but progress doesn't. | → PLAN. Re-scope to original goal. |
| **Validation Gap** | **2** | More than 2 steps without validation. | → VALIDATE. Must test before continuing. |

### When to Tune

| Scenario | Adjustment |
|----------|------------|
| False positives on error detection | Raise repeat error threshold to 3 |
| Missing repeated mistakes | Lower repeat error threshold to 1 |
| Tasks legitimately require file iteration | Raise diff thrash to 4-5 |
| Want stricter evidence requirements | Lower no new info to 3 |

---

## 7. Cognitive Learner (Decay)

**File:** `lib/cognitive_learner.py`

The Cognitive Learner stores insights with time-based decay. Older insights gradually lose reliability, ensuring the system stays current and doesn't over-rely on stale knowledge.

### How It Works

```
Effective Reliability = Base Reliability × 2^(-age_days / half_life)
```

After one half-life period, reliability drops to 50%. After two half-lives, 25%, etc.

### Half-Life by Category

| Category | Half-Life | Rationale |
|----------|-----------|-----------|
| `WISDOM` | **180 days** | Principles and wisdom are timeless, decay slowly. "Ship fast, iterate faster" stays true. |
| `META_LEARNING` | **120 days** | How to learn itself changes slowly. Learning strategies remain valid. |
| `USER_UNDERSTANDING` | **90 days** | User preferences are fairly stable but can evolve. |
| `COMMUNICATION` | **90 days** | Communication style preferences are sticky but not permanent. |
| `SELF_AWARENESS` | **60 days** | Blind spots need regular reassessment. What I struggled with before may not apply now. |
| `REASONING` | **60 days** | Assumptions and reasoning patterns should be questioned regularly. |
| `CREATIVITY` | **60 days** | Novel approaches may become stale as tech evolves. |
| `CONTEXT` | **45 days** | Environment-specific context changes frequently. Project structure, team practices, etc. |

### Pruning Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `max_age_days` | **365** | **Maximum age.** Insights older than 1 year are pruned regardless of reliability. |
| `min_effective` | **0.2** | **Minimum effective reliability.** When decay brings reliability below 20%, the insight is pruned. |

### When to Tune

| Scenario | Adjustment |
|----------|------------|
| Fast-changing project | Lower CONTEXT half-life to 30 days |
| Stable long-term project | Raise half-lives across the board |
| Want insights to last longer | Raise `max_age_days` to 730 (2 years) |
| Memory getting cluttered | Lower `min_effective` to 0.3 |

---

## 8. Structural Retriever

**File:** `lib/eidos/retriever.py`

The Structural Retriever fetches relevant Distillations before actions. Unlike text similarity search, it prioritizes by EIDOS structure (policies > playbooks > sharp edges > heuristics).

### How It Works

```
Intent/Error → Keyword Extraction → Match Against Distillations → Sort by Type Priority → Return Top N
```

### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `max_results` | **10** | **Result limit.** Maximum Distillations returned per query. More results = more context but also more noise. |
| `min_overlap` | **2** | **Keyword threshold.** Minimum number of keywords that must overlap between query and Distillation. Filters out weak matches. |

### Type Priority Order

1. **POLICY** (highest) - User-defined rules always come first
2. **PLAYBOOK** - Multi-step procedures for known situations
3. **SHARP_EDGE** - Gotchas and pitfalls to avoid
4. **HEURISTIC** - General "if X then Y" patterns
5. **ANTI_PATTERN** (lowest) - What not to do

### When to Tune

| Scenario | Adjustment |
|----------|------------|
| Retrieval returning irrelevant results | Raise `min_overlap` to 3 |
| Missing relevant Distillations | Lower `min_overlap` to 1 |
| Too much context overwhelming decisions | Lower `max_results` to 5 |
| Complex tasks need more guidance | Raise `max_results` to 15-20 |

---

## 9. Importance Scorer (Signal Detection)

**File:** `lib/importance_scorer.py`

The Importance Scorer evaluates incoming text at **ingestion time** (not promotion time) to determine what's worth learning. This ensures critical one-time insights are captured even if they never repeat.

### How It Works

Text is analyzed for signal patterns that indicate importance:
1. Check for CRITICAL signals (explicit requests, corrections)
2. Check for HIGH signals (preferences, principles)
3. Check for MEDIUM signals (observations, context)
4. Check for LOW signals (noise indicators)
5. Apply domain relevance boost
6. Apply first-mention elevation

### Importance Tiers

| Tier | Score Range | Behavior | Examples |
|------|-------------|----------|----------|
| **CRITICAL** | 0.9+ | Learn immediately, bypass normal thresholds | "Remember this", corrections, "never do X" |
| **HIGH** | 0.7-0.9 | Should learn, prioritize | Preferences, principles, reasoned explanations |
| **MEDIUM** | 0.5-0.7 | Consider learning | Observations, context, weak preferences |
| **LOW** | 0.3-0.5 | Store but don't promote | Acknowledgments, trivial statements |
| **IGNORE** | <0.3 | Don't store | Tool sequences, metrics, operational noise |

### Critical Signals (Immediate Learning)

| Pattern | Signal Type | Why It's Critical |
|---------|-------------|-------------------|
| "remember this" | explicit_remember | User explicitly requesting persistence |
| "always do it this way" | explicit_preference | Strong user directive |
| "never do this" | explicit_prohibition | Important constraint |
| "no, I meant..." | correction | User correcting misunderstanding |
| "because this works" | reasoned_decision | Outcome with explanation |

### High Signals

| Pattern | Signal Type |
|---------|-------------|
| "I prefer" | preference |
| "let's go with" | preference |
| "the key is" | principle |
| "the pattern here is" | pattern_recognition |
| "in general" | generalization |

### Low Signals (Noise)

| Pattern | Signal Type |
|---------|-------------|
| "Bash → Edit" | tool_sequence |
| "45% success" | metric |
| "timeout" | operational |
| "okay", "got it" | acknowledgment |

### When to Tune

Add domain-specific patterns to `DOMAIN_WEIGHTS` for your use case. See Section 15 for domain weight configuration.

---

## 10. Context Sync Defaults

**File:** `lib/context_sync.py`

Context Sync synchronizes high-value insights to Mind (persistent memory) for cross-session retrieval.

### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `DEFAULT_MIN_RELIABILITY` | **0.7** | **Quality threshold.** Only sync insights with 70%+ reliability to Mind. |
| `DEFAULT_MIN_VALIDATIONS` | **3** | **Evidence threshold.** Insights must be validated 3+ times before syncing. |
| `DEFAULT_MAX_ITEMS` | **12** | **Batch limit.** Maximum items to sync per operation. |
| `DEFAULT_MAX_PROMOTED` | **6** | **Promotion limit.** Maximum items to mark as "promoted" per sync. |

### When to Tune

| Scenario | Adjustment |
|----------|------------|
| Mind getting cluttered | Raise thresholds |
| Missing important context | Lower `DEFAULT_MIN_VALIDATIONS` to 2 |
| Want more cross-session memory | Raise `DEFAULT_MAX_ITEMS` to 20 |

---

## 11. Advisor (Action Guidance)

**File:** `lib/advisor.py`

The Advisor queries relevant insights **before** actions are taken, making stored knowledge actionable. It bridges the gap between learning and decision-making.

### How It Works

```
Tool + Context → Query Memory Banks + Cognitive Insights + Mind → Rank by Relevance → Return Advice
```

### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `MIN_RELIABILITY_FOR_ADVICE` | **0.5** | **Quality filter.** Only include insights with 50%+ reliability in advice. Lowered from 0.6 for more advice coverage. Wired to `tuneables.json` -> `advisor.min_reliability`. |
| `MIN_VALIDATIONS_FOR_STRONG_ADVICE` | **2** | **Strong advice threshold.** Insights validated 2+ times are marked as "strong" advice. Wired to `advisor.min_validations_strong`. |
| `MAX_ADVICE_ITEMS` | **3** | **Advice limit.** Runtime reads `advisor.max_items`. Keep `advisor.max_advice_items` mirrored for auto-tuner compatibility. |
| `ADVICE_CACHE_TTL_SECONDS` | **120** | **Cache duration (2 min).** Same query within 2 minutes returns cached advice. Wired to `advisor.cache_ttl` (also reads `values.advice_cache_ttl`). |
| `MIN_RANK_SCORE` | **0.55** | **Rank cutoff.** Drop advice below this score after ranking; prefer fewer, higher-quality items. Wired to `advisor.min_rank_score`. |
| `MIND_MAX_STALE_SECONDS` | **0** | **Mind freshness gate.** `0` disables staleness blocking; positive values block stale Mind retrieval when newer local evidence exists. Wired to `advisor.mind_max_stale_s`. |
| `MIND_STALE_ALLOW_IF_EMPTY` | **true** | **Cross-session fallback.** If Mind is stale but no other advice exists, still allow Mind retrieval. Wired to `advisor.mind_stale_allow_if_empty`. |
| `MIND_MIN_SALIENCE` | **0.5** | **Mind quality floor.** Ignore low-salience Mind memories below this threshold. Wired to `advisor.mind_min_salience`. |

Compatibility note:
- Runtime advisor uses `advisor.max_items`.
- Auto-tuner recommendation logic currently targets `advisor.max_advice_items`.
- Keep both keys equal to avoid drift.

### Advice Sources

| Source | What It Provides |
|--------|------------------|
| `cognitive` | Insights from cognitive_learner (preferences, self-awareness) |
| `mind` | Memories from Mind persistent storage |
| `bank` | Project/global memory banks |
| `self_awareness` | Cautions about known struggles |
| `surprise` | Warnings from past unexpected failures |
| `skill` | Relevant skill recommendations |

### When to Tune

| Scenario | Adjustment |
|----------|------------|
| Getting too much advice | Lower `advisor.max_items` (and mirror `advisor.max_advice_items`) to 3 |
| Missing relevant warnings | Lower `MIN_RELIABILITY_FOR_ADVICE` to 0.5 |
| Advice is stale | Lower `ADVICE_CACHE_TTL_SECONDS` to 60 (already lowered to 120) |
| Performance issues | Raise cache TTL to 600 (10 min) |

### Semantic Retrieval (Optional)

Semantic retrieval augments Advisor with embeddings + trigger rules. It is
**disabled by default** unless enabled in `~/.spark/tuneables.json` or via
`SPARK_SEMANTIC_ENABLED=1`.

| Parameter | Default | Description |
|----------|---------|-------------|
| `semantic.enabled` | **false** | Enable semantic retrieval for cognitive insights |
| `semantic.min_similarity` | **0.6** | Min cosine similarity to allow semantic candidates |
| `semantic.min_fusion_score` | **0.5** | Final decision threshold after fusion |
| `semantic.weight_recency` | **0.2** | Recency boost weight |
| `semantic.weight_outcome` | **0.3** | Outcome effectiveness boost weight |
| `semantic.mmr_lambda` | **0.5** | Diversity balance (1.0 = relevance only) |
| `semantic.dedupe_similarity` | **0.92** | Dedupe near-duplicate results by embedding cosine |
| `semantic.index_on_write` | **true** | Index embeddings on insight write |
| `semantic.index_on_read` | **true** | Backfill missing embeddings at retrieval time |
| `semantic.index_backfill_limit` | **300** | Max insights to backfill per run |
| `semantic.index_cache_ttl_seconds` | **120** | Cache duration for vector index |
| `semantic.exclude_categories` | **[]** | Categories to exclude from semantic results (e.g., `["context"]`) |
| `semantic.log_retrievals` | **true** | Log semantic retrieval events to `~/.spark/logs/semantic_retrieval.jsonl` |

Trigger rules (YAML):

| Parameter | Default | Description |
|----------|---------|-------------|
| `triggers.enabled` | **false** | Enable explicit trigger rules |
| `triggers.rules_file` | **~/.spark/trigger_rules.yaml** | YAML rules file |

Environment overrides:
- `SPARK_SEMANTIC_ENABLED=1`
- `SPARK_TRIGGERS_ENABLED=1`

---

## 12. Memory Capture

**File:** `lib/memory_capture.py`

Memory Capture scans user messages for statements worth persisting. It uses keyword triggers and heuristics to identify preferences, rules, and decisions.

### How It Works

```
User Message → Score Against Triggers → Above Auto-Save? → Save Automatically
                                     → Above Suggest? → Queue for Review
                                     → Below Suggest? → Ignore
```

### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `AUTO_SAVE_THRESHOLD` | **0.82** | **Auto-save cutoff.** Statements scoring 82%+ are saved without confirmation. High threshold ensures only clear signals auto-save. |
| `SUGGEST_THRESHOLD` | **0.55** | **Suggestion cutoff.** Statements scoring 55-82% are queued for user review. Below 55% is ignored. |
| `MAX_CAPTURE_CHARS` | **2000** | **Length limit.** Maximum characters to capture. Longer statements are truncated. |

### Hard Triggers (Explicit Signals)

These keywords trigger high scores immediately:

| Trigger Phrase | Score | Why |
|----------------|-------|-----|
| "remember this" | 1.0 | Explicit persistence request |
| "don't forget" | 0.95 | Strong persistence signal |
| "lock this in" | 0.95 | Commitment language |
| "non-negotiable" | 0.95 | Boundary/constraint |
| "hard rule" | 0.95 | Explicit rule definition |
| "hard boundary" | 0.95 | Constraint definition |
| "from now on" | 0.85 | Future-oriented preference |
| "