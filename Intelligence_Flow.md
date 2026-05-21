```
# Intelligence_Flow.md

Generated: 2026-02-06
Navigation hub: `docs/GLOSSARY.md`
Repository: vibeship-spark-intelligence
Scope:
- Runtime Python modules: root *.py, lib/, hooks/, adapters/, spark/
- Configs: chips/*.chip.yaml, chips/examples/*.chip.yaml, config/learning_sources.yaml
Exclusions:
- tests/, benchmarks/, build/ artifacts (not part of runtime)

Notes:
- ASCII only.
- Auto-extracted sections (chip inventory, learning_sources config, tuneables index, import map) are appended at the end.

## 1) System overview (what talks to what)

Sources:
- hooks/observe.py (agent hooks)
- sparkd.py (HTTP ingest for SparkEventV1)
- adapters/* (stdin_ingest, clawdbot_tailer, moltbook)
- scripts/emit_event.py (manual event injector)

Central bus:
- lib/queue.py writes events to ~/.spark/queue/events.jsonl (append-only) with:
  - lock-contention spillover to ~/.spark/queue/events.overflow.jsonl
  - logical consume head in ~/.spark/queue/state.json (O(1) consume + periodic compaction)

Processing loop:
- bridge_worker.py -> lib/bridge_cycle.run_bridge_cycle

High-level flow (ASCII):
sources -> queue -> bridge_cycle -> {update_spark_context, memory capture, pattern detection, validation/prediction, content learning, chips}
                         -> {cognitive insights, memory banks/store, eidos store, advisor loop, outputs}

Mind integration:
- mind_server.py (Mind Lite+ API, sqlite at ~/.mind/lite/memories.db)
- lib/mind_bridge.py (retrieval used by advisor + lib.bridge for SPARK_CONTEXT)
- Mind sync is manual (spark sync / lib.mind_bridge.sync_all_to_mind); offline queue used when Mind is down.
- mind_bridge health checks are cached/backed-off to avoid repeated blocking calls while Mind is unavailable.
- If Mind CLI is unstable, Spark can run the built-in Mind server (see start_mind.ps1; SPARK_FORCE_BUILTIN_MIND=1).

Observability:
- Spark Pulse (primary web dashboard) via external vibeship-spark-pulse/app.py (port SPARK_PULSE_PORT, default 8765).
- Obsidian Observatory (file-based pipeline viewer) via lib/observatory/ — auto-syncs every 120s.
- CLI scripts: scripts/spark_dashboard.py, scripts/eidos_dashboard.py
- spark_watchdog.py + lib/service_control.py (monitor/restart services)

## 2) Primary workflows (end-to-end)

### 2.1 Event ingest -> queue
1) hooks/observe.py captures tool/user events and emits SparkEvent payloads.
2) sparkd.py validates SparkEventV1 and writes queue events via lib.queue.quick_capture.
3) adapters (stdin_ingest, clawdbot_tailer, moltbook) also feed sparkd or queue.
4) hooks/observe.py also runs EIDOS pre/post step tracking and Meta-Ralph roasting for cognitive signals (direct calls, not via bridge_cycle).

### 2.2 Queue -> bridge cycle
1) bridge_worker.py loops every interval (default 60s, min 10s sleep).
2) lib.bridge_cycle.run_bridge_cycle reads recent events and orchestrates learning tasks.
3) bridge_cycle enables deferred/batch writes for cognitive learner and Meta-Ralph to avoid repeated large JSON rewrites per event.
4) bridge_cycle classifies events in one pass and reuses those buckets for tastebank, content learning, cognitive signal extraction, and chips.
5) bridge_cycle now runs both prompt-based validation and outcome-linked validation each cycle.
6) bridge_cycle runs runtime hygiene cleanup each cycle (stale heartbeat files, stale PID state, stale tmp artifacts).
7) A heartbeat is written to ~/.spark/bridge_worker_heartbeat.json.

### 2.2.1 Trace context propagation (v1)
- trace_id is attached at ingest (hooks/observe.py, sparkd.py, lib.queue.quick_capture).
- trace_id is carried into pattern events and EIDOS Steps.
- evidence is linked to steps; outcomes include trace_id when available.
- trace binding enforcement: steps, evidence, and outcomes should record trace_id; TRACE_GAP watcher warns on missing bindings. Set `SPARK_TRACE_STRICT=1` to block actions on trace gaps.
- dashboards should drill down by trace_id for audit and validation.

### 2.3 Context sync + promotion (live vs durable)
1) lib.bridge.update_spark_context builds a live context pack (insights, warnings, advice, skills, taste, outcomes) and writes SPARK_CONTEXT.md.
2) lib.context_sync selects high-confidence insights and writes live context to output adapters.
   - default sync mode is `core`, which writes only `openclaw` + `exports`.
   - optional adapters (`claude_code`, `cursor`, `windsurf`, `clawdbot`) are opt-in and can be disabled without affecting core health.
   - context_sync can include promoted lines already present in CLAUDE.md / AGENTS.md / TOOLS.md / SOUL.md.
   - context_sync also injects recent high-quality chip highlights from ~/.spark/chip_insights.
   - context_sync does not sync to Mind (see 2.8).
3) lib.promoter is a separate, manual step (spark promote) that writes durable learnings into CLAUDE.md / AGENTS.md / TOOLS.md / SOUL.md.
   - promoter can run chip_merger first so high-quality chip insights enter cognitive promotion flow.

### 2.4 Memory capture + cognitive learning
1) lib.memory_capture scans user messages for memory triggers.
2) High-signal items are auto-saved or queued for review.
3) lib.cognitive_learner stores insights + reliability/validation + decay.
4) lib.memory_banks stores per-project and global memory.
5) lib.memory_store persists hybrid memory (SQLite FTS + embeddings + graph edges).
6) hooks/observe.py uses Meta-Ralph to roast cognitive signals before add_insight.

### 2.4.1 Semantic index (embeddings)
1) lib.cognitive_learner calls lib.semantic_retriever.index_insight on write (best-effort).
2) lib.semantic_retriever stores vectors in ~/.spark/semantic/insights_vec.sqlite.
3) Backfill command: python -m spark.index_embeddings --all
4) Legacy reindex script: python scripts/semantic_reindex.py
5) Harness: python scripts/semantic_harness.py (prints top results + why)

### 2.5 Pattern -> distillation -> EIDOS
1) lib.pattern_detection.aggregator runs detectors (correction, sentiment, repetition, semantic, why).
2) lib.pattern_detection.request_tracker wraps user requests as EIDOS Steps.
3) lib.pattern_detection.distiller creates distillations (heuristic, anti-pattern, sharp edge, playbook, policy).
4) lib.pattern_detection.memory_gate filters low-signal items.
5) lib.eidos.store persists to ~/.spark/eidos.db; lib.eidos.retriever retrieves for guidance.
6) lib.eidos.control_plane and lib.eidos.elevated_control enforce budgets and stuck detection.

### 2.6 Outcomes, predictions, validation, surprises
1) lib.prediction_loop builds and tracks predictions; matches outcomes later.
2) lib.outcome_log + lib.outcomes/* store outcomes and links.
3) lib.validation_loop updates reliability based on outcomes.
4) lib.aha_tracker captures surprises for learning.
5) lib.exposure_tracker records exposure timing for prediction evaluation.
   - sync-heavy sources are deduped/capped (`sync_context`, `sync_context:project`, `chip_merge`) to reduce noise/churn.

### 2.6.1 Advisory engine + packetized feedback loop
1) PreToolUse: hooks/observe.py calls lib.advisory_engine.on_pre_tool.
2) Advisory engine resolves intent and task plane via lib.advisory_intent_taxonomy.
3) Engine attempts packet lookup first (exact, then relaxed) via lib.advisory_packet_store.
4) On packet miss, engine falls back to live advisor retrieval, then gate + synthesis + emit.
   - packet no-emit fallback emission is opt-in (`SPARK_ADVISORY_PACKET_FALLBACK_EMIT=1` or `advisory_engine.packet_fallback_emit_enabled=true`).
   - packet no-emit fallback is additionally rate-guarded (`fallback_rate_guard_enabled`, `fallback_rate_max_ratio`, `fallback_rate_window`) to prevent fallback-heavy advisory loops.
   - live retrieval now receives the same Mind policy (`include_mind`) as memory fusion, so packet/live paths no longer drift on Mind usage.
5) Hot path is optimized for advisory speed: engine does not build the multi-source memory bundle during PreToolUse.
   - Packet lineage is inferred from emitted advice source labels.
   - `lib.advisory_memory_fusion` remains available for offline diagnostics and future non-hot-path use.
6) Engine persists baseline/live packets and enqueues background prefetch jobs from UserPromptSubmit.
7) PostToolUse/PostToolUseFailure call advisory_engine.on_post_tool for implicit feedback and packet invalidation on Edit/Write.
8) Advisor still logs retrievals/outcomes and Meta-Ralph updates quality via outcome-linked feedback.
9) Advisory event logs now carry diagnostics envelope fields for explainability and debugging:
   - `session_id`, `trace_id`, `session_context_key`, `scope`, `provider_path`, `source_counts`, `missing_sources` (may be empty on hot path).
10) Advisory actionability is enforced on emitted guidance:
   - when no concrete command/check is present, engine appends `Next check: <command>`.
11) Engine status derives a delivery badge from recent events:
   - `live | fallback | blocked | stale`.
12) Semantic retrieval path (when semantic.enabled=true):
   - intent = semantic_retriever._extract_intent(context)
   - triggers (optional) + semantic search (embeddings) + fusion scoring + dedupe + MMR + category caps
   - retrieval log: ~/.spark/logs/semantic_retrieval.jsonl (intent, candidates, triggers, top-N scores)
13) Advisor retrieval router (Carmack-style, retrieval section in tuneables):
   - default strategy: embeddings-first fast path, then selective agentic fanout
   - minimal escalation gate in auto mode:
     - weak primary count
     - weak primary top score
     - high-risk terms
   - hard controls:
     - agentic deadline (`agentic_deadline_ms`)
     - agentic rate cap (`agentic_rate_limit`, windowed)
     - prefilter cap (`prefilter_max_insights`) before semantic retrieval
   - route telemetry:
     - ~/.spark/advisor/retrieval_router.jsonl with route, reasons, elapsed, budget/cap flags

### 2.6.2 Learning usage in real work (semantic first)
1) Advisor is called before actions, so learnings can change the next decision (not just be stored).
2) Semantic retrieval matches meaning, not just keywords, then surfaces top items with a short reason (why) for fast human validation.
3) Trigger rules inject critical guardrails immediately (security, destructive ops, deploys).
4) Outcomes feed back into Meta-Ralph and cognitive reliability so future advice is prioritized by what actually helped.

### 2.7 Chips pipeline (domain intelligence)
1) lib.chips.loader discovers chips across formats:
   - single file (`chips/*.chip.yaml`)
   - multifile bundles (`chips/multifile/<chip>/chip.yaml` + components)
   - hybrid specs (`chips/hybrid/*.chip.yaml` + includes)
2) lib.chips.router normalizes event aliases (`PostToolUse` -> `post_tool`, etc.) and matches event/tool/pattern triggers.
3) lib.chips.runtime runs observers/learners and applies quality gates before storage.
   - low-value/primitive insights are filtered by score before write
   - balanced gate enforces confidence + safety + evidence/outcome checks
   - chip-level fallback matches are suppressed when observer matches exist
   - chip runtime/store apply size-based JSONL rotation to prevent unbounded chip insight growth
4) lib.chips.scoring computes cognitive value and promotion tier.
5) lib.chips.evolution records trigger quality and can deprecate/add triggers or suggest provisional chips.
6) lib.chip_merger merges accepted chip insights into cognitive categories.
   - unknown chip IDs use domain/content fallback category inference
   - merge dedupe uses stable content hash (timestamp-independent) to suppress repeat churn
   - low-quality repeats are suppressed with cooldown tracking before recounting skip noise
   - duplicate-only churn now enters throttle cooldown to avoid repeated no-yield merge cycles

### 2.8 Mind retrieval + manual sync
1) lib.mind_bridge retrieves from mind_server.py (keyword + optional FTS, RRF + salience).
2) lib.advisor and lib.bridge (SPARK_CONTEXT) read from Mind for advice/context.
3) Mind sync is manual via spark sync (lib.mind_bridge.sync_all_to_mind); offline queue stores when Mind is down.
4) mind_bridge uses per-endpoint timeouts plus health cache/backoff to keep hook paths responsive during Mind outages.
5) advisor can gate stale Mind reads (`advisor.mind_max_stale_s`) while still allowing Mind as fallback when no other evidence is found (`advisor.mind_stale_allow_if_empty=true`).

### 2.9 Self-evolution and meta-learning
- lib/meta_ralph.py: quality gate for observe.py cognitive capture + advisor outcome loop.
- lib/metalearning/*: strategy, evaluator, reporter, metrics (used by chips auto-activation).
- lib/resonance.py + lib/spark_voice.py: internal state and voice calibration (used by bridge context).
- lib/growth_tracker.py: milestones and long-term growth stats.
- lib/curiosity_engine.py, lib/contradiction_detector.py, lib/hypothesis_tracker.py: triggered in pattern_detection.aggregator.

### 2.10 Research / external sources
- lib/research/* and lib/x_research_events.py use config/learning_sources.yaml to drive external research.

### 2.11 Services / ops
- lib/service_control.py starts/stops sparkd, bridge_worker, dashboard, watchdog, and pulse.
- pulse process management targets external vibeship-spark-pulse/app.py.
- spark_watchdog.py checks health, queue size, and heartbeat freshness.

## 3) Data stores and artifacts (local)

Core queue + ingest:
- ~/.spark/queue/events.jsonl
- ~/.spark/queue/.queue.lock
- ~/.spark/queue/events.overflow.jsonl
- ~/.spark/queue/state.json
- ~/.spark/invalid_events.jsonl

Context + promotion:
- ~/.spark/cognitive_insights.json
- ~/.spark/.cognitive.lock
- ~/.spark/pending_memory.json
- ~/.spark/memory_capture_state.json
- ~/.spark/bridge_worker_heartbeat.json
Workspace context:
- SPARK_CONTEXT.md (workspace, written by lib.bridge.update_spark_context)

Pattern + EIDOS:
- ~/.spark/detected_patterns.jsonl
- ~/.spark/pattern_detection_state.json
- ~/.spark/eidos.db
- ~/.spark/truth_ledger.json
- ~/.spark/acceptance_plans.json
  - steps table includes trace_id (v1 trace context)

Memory banks + store:
- ~/.spark/banks/*.jsonl
- ~/.spark/memory_store.sqlite
Semantic retrieval:
- ~/.spark/semantic/insights_vec.sqlite
- ~/.spark/logs/semantic_retrieval.jsonl
Advisor metrics:
- ~/.spark/advisor/metrics.json
Advisor routing diagnostics:
- ~/.spark/advisor/retrieval_router.jsonl
Advisory foundation:
- ~/.spark/advisory_engine.jsonl
- ~/.spark/advisory_emit.jsonl
- ~/.spark/advisory_state/
- ~/.spark/advice_packets/index.json
- ~/.spark/advice_packets/*.json
- ~/.spark/advice_packets/prefetch_queue.jsonl
- `advisory_engine.jsonl` events include diagnostics envelope, actionability metadata, and route/delivery fields used by operator badges.

Outcomes + prediction:
- ~/.spark/predictions.jsonl
- ~/.spark/prediction_state.json
- ~/.spark/outcomes.jsonl
- ~/.spark/outcome_links.jsonl
- ~/.spark/outcome_requests.jsonl
- ~/.spark/outcome_checkin_state.json
  - outcomes may include trace_id when derived from queue events
- ~/.spark/outcome_tracker.json
- ~/.spark/exposures.jsonl
- ~/.spark/last_exposure.json

Chips:
- ~/.spark/chip_insights/
- ~/.spark/chip_registry.json
- ~/.spark/chip_evolution.yaml
- ~/.spark/provisional_chips/
- ~/.spark/chip_merge_state.json
- ~/.spark/chips/ (user-installed chips, including multifile bundles)

Skills + advisor + sync:
- ~/.spark/skills_index.json
- ~/.spark/skills_effectiveness.json
- ~/.spark/advisor/
- ~/.spark/sync_stats.json

Meta-Ralph:
- ~/.spark/meta_ralph/roast_history.json
- ~/.spark/meta_ralph/outcome_tracking.json
- ~/.spark/meta_ralph/learnings_store.json
- ~/.spark/meta_ralph/self_roast.json

Mind sync state:
- ~/.spark/mind_sync_state.json
- ~/.spark/mind_offline_queue.jsonl

Other:
- ~/.spark/projects.json
- ~/.spark/project_context.json
- ~/.spark/project_contexts/
- ~/.spark/taste/
- ~/.spark/research/
- ~/.spark/exports/
- ~/.spark/logs/
- ~/.spark/pids/
- ~/.spark/watchdog_state.json

Mind:
- ~/.mind/lite/memories.db

## 4) Key tuneables (curated, high leverage)

Ingest + servers:
- sparkd.py uses SPARKD_PORT (default 8787), SPARKD_MAX_BODY_BYTES=262144 (max /ingest payload).
- mind_server.py uses SPARK_MIND_PORT (default 8080), MIND_MAX_BODY_BYTES=262144, MIND_MAX_CONTENT_CHARS=4000, MIND_MAX_QUERY_CHARS=1000.
- mind_server.py RRF_K=60 (rank fusion constant).

Queue:
- lib.queue.py MAX_EVENTS=10000 (rotation threshold).
- lib.queue.py TAIL_CHUNK_BYTES=65536 (tail read size).
- lib.queue._queue_lock timeout_s=0.5 (lock wait).
- lib.queue.py MAX_QUEUE_BYTES=10485760 (10MB active queue budget).
- lib.queue.py SPARK_QUEUE_COMPACT_HEAD_BYTES=5242880 (logical-head compaction threshold).

Bridge cycle:
- bridge_worker.py --interval default 60s, enforced min sleep 10s.
- lib.bridge_cycle.run_bridge_cycle memory_limit=60, pattern_limit=200.
- bridge_cycle reads 40 recent events and checks last 10 user prompts for tastebank.
- bridge_cycle defers cognitive/meta writes until cycle end (batch flush).
  - bridge step timeout: SPARK_BRIDGE_STEP_TIMEOUT_S (default 45s)
  - bridge timeout disable: SPARK_BRIDGE_DISABLE_TIMEOUTS (default off)

Memory capture:
- lib.memory_capture.MAX_CAPTURE_CHARS=2000
- AUTO_SAVE_THRESHOLD=0.82, SUGGEST_THRESHOLD=0.55
- pending suggestions capped at 200; list_pending limit 20; process_recent_memory_events default limit 50
- HARD_TRIGGERS / SOFT_TRIGGERS weights drive scoring.

Pattern detection and distillation:
- aggregator CONFIDENCE_THRESHOLD=0.6, DEDUPE_TTL_SECONDS=600, DISTILLATION_INTERVAL=20
- distiller: min_occurrences=2, min_occurrences_critical=1, min_confidence=0.6, gate_threshold=0.5
- repetition detector: min_similarity=0.5, min length 10, keep 20, group size >=3, confidence starts 0.7
- sentiment detector: frustration/satisfaction pattern lists and thresholds
- correction detector: patterns list, confidence threshold >=0.6

Memory gate (pattern_detection/memory_gate.py):
- WEIGHTS: impact 0.30, novelty 0.20, surprise 0.30, recurrence 0.20, irreversible 0.60, evidence 0.10
- threshold=0.5, high_stakes keyword list.

EIDOS control and budgets:
- Budget defaults: max_steps=25, max_time_seconds=720, max_retries_per_error=2, max_file_touches=3, no_evidence_limit=5
- control_plane watcher thresholds: repeat_error=2, no_new_info=5, diff_thrash=3, confidence_delta=0.05, confidence_stagnation_steps=3
- elevated_control escape thresholds (see auto index for full list).

Cognitive learning and promotion:
- cognitive_learner half-lives by category (see auto index), max_age_days=365, min_effective=0.2
- promoter DEFAULT_PROMOTION_THRESHOLD=0.7, DEFAULT_MIN_VALIDATIONS=3, DEFAULT_CONFIDENCE_FLOOR=0.90
- context_sync DEFAULT_MIN_RELIABILITY=0.7, DEFAULT_MIN_VALIDATIONS=3, DEFAULT_MAX_ITEMS=12, DEFAULT_MAX_PROMOTED=6
  - context_sync also injects recent high-quality chip highlights

Advisor retrieval router:
- ~/.spark/tuneables.json -> retrieval:
  - level
  - overrides:
    - mode, gate_strategy
    - semantic_limit, max_queries, agentic_query_limit
    - agentic_deadline_ms, agentic_rate_limit, agentic_rate_window
    - fast_path_budget_ms
    - prefilter_enabled, prefilter_max_insights
    - lexical_weight, bm25_k1, bm25_b, bm25_mix
    - min_results_no_escalation, min_top_score_no_escalation
    - escalate_on_high_risk, escalate_on_trigger
    - semantic_context_min, semantic_lexical_min, semantic_strong_override

Advisor / skills:
- advisor tuneables: `~/.spark/tuneables.json` -> `advisor.*` (reliability floor, max_items, cache_ttl, min_rank_score, etc.)
- advisor Mind tuneables: `~/.spark/tuneables.json` -> `advisor.mind_*` (staleness + salience controls)
- skills_router scoring weights (query/name/desc/owns/etc) and limit clamp to 1..10
- advisor recent-advice lookup is tail-based (bounded by RECENT_ADVICE_MAX_LINES, no full-file scans)
- advisor cache key now includes `include_mind` to avoid mixed-policy cache reuse.
Advisory foundation:
- advisory engine enabled by default (SPARK_ADVISORY_ENGINE=1) with direct-path budget SPARK_ADVISORY_MAX_MS=4000.
- direct path: packet lookup -> live retrieval fallback -> deterministic/AI synthesis -> stdout emission.
- packet store defaults:
  - packet TTL DEFAULT_PACKET_TTL_S=900
  - max indexed packets MAX_INDEX_PACKETS=2000
- prefetch queue is enabled by default (SPARK_ADVISORY_PREFETCH_QUEUE=1) and fed from UserPromptSubmit.
- memory fusion can optionally include Mind retrieval (tuneable `advisory_engine.include_mind`; env `SPARK_ADVISORY_INCLUDE_MIND` sets the default).
- actionability enforcement is on by default (SPARK_ADVISORY_REQUIRE_ACTION=1).
- repeat suppression uses text fingerprint cooldown (tuneable `advisory_engine.advisory_text_repeat_cooldown_s`; env default is 1800s).
- delivery stale window defaults to 900s (SPARK_ADVISORY_STALE_S) for `live/fallback/blocked/stale` status.
Advisory synthesis:
- SPARK_SYNTH_MODE=auto|ai_only|programmatic (default auto)
- SPARK_SYNTH_TIMEOUT=3.0s
- SPARK_OLLAMA_MODEL default phi4-mini (override via SPARK_OLLAMA_MODEL)
- cloud fallback available when API keys exist and provider chain allows.
Semantic retrieval:
- semantic.dedupe_similarity default 0.92 (embedding cosine)
- semantic.log_retrievals default true (writes semantic_retrieval.jsonl)

Mind bridge:
- MIND_API_URL default from SPARK_MIND_PORT (default 8080)
- MIND_HEALTH_TIMEOUT_S=8.0
- MIND_POST_TIMEOUT_S=5.0
- MIND_RETRIEVE_TIMEOUT_S=3.0
- MIND_HEALTH_CACHE_TTL_S=30.0
- MIND_HEALTH_BACKOFF_MAX_S=15.0
- salience clamp 0.5..0.95, retrieve limit 5
- offline queue and sync state kept under ~/.spark

Chips:
- chip scoring weights: cognitive_value 0.30, outcome_linkage 0.20, uniqueness 0.15, actionability 0.15, transferability 0.10, domain_relevance 0.10
- evolution thresholds: deprecate triggers when matches>=10 and value_ratio<0.2; provisional chip rules (see auto index)
- runtime insight limit default 50
- runtime quality gate: SPARK_CHIP_MIN_SCORE (default 0.35)
- runtime confidence gate: SPARK_CHIP_MIN_CONFIDENCE (default 0.7)
- runtime gate mode: SPARK_CHIP_GATE_MODE (default balanced)
- runtime/store rotate JSONL files at size thresholds (runtime 2MB cap, observations 5MB cap)
- loader env SPARK_CHIP_SCHEMA_VALIDATION=warn|block
- loader preference env SPARK_CHIP_PREFERRED_FORMAT=single|multifile|hybrid (default multifile)
Auto-tuner hygiene:
- auto_tuner only writes tuneables when values actually change (no-op recommendation/boost writes are skipped)
- auto_tuner validates via `lib/tuneables_schema.py` before every write (clamps out-of-bounds)
- auto_tuner records drift distance after every write via `lib/tuneables_drift.py`

Tuneables infrastructure:
- `lib/tuneables_schema.py`: Central schema (25 sections, 153 keys) with type, bounds, defaults validation
- `lib/tuneables_reload.py`: Mtime-based hot-reload coordinator. `check_and_reload()` called at top of every bridge cycle
- `lib/tuneables_drift.py`: Normalized distance between runtime and `config/tuneables.json` baseline (alerts when >0.3)
- Hot-reload registered: meta_ralph, eidos, pipeline (values), queue, advisory_gate, advisor
- Reference doc: `docs/TUNEABLES_REFERENCE.md` (auto-generated from schema via `generate_reference_doc()`)

Outcomes + prediction:
- prediction_loop: prediction max age 6h, project prediction max age 14 days, match sim threshold 0.72
- outcome_linker: auto_link_outcomes min_similarity 0.25, get_linkable_candidates min_similarity 0.2

## 5) Environment variables (from code)

Ingest:
- SPARKD_TOKEN (optional bearer auth for /ingest)
- SPARKD_MAX_BODY_BYTES (default 262144, int)

Mind server:
- MIND_TOKEN (optional bearer auth)
- MIND_MAX_BODY_BYTES (default 262144, int)
- MIND_MAX_CONTENT_CHARS (default 4000, int)
- MIND_MAX_QUERY_CHARS (default 1000, int)

Hooks:
- SPARK_EIDOS_ENABLED (default "1")
- SPARK_OUTCOME_CHECKIN_MIN_S (default 1800, int)
- SPARK_OUTCOME_CHECKIN (enable/disable)
- SPARK_OUTCOME_CHECKIN_PROMPT (enable/disable)
Advisory:
- SPARK_ADVISORY_ENGINE (default "1")
- SPARK_ADVISORY_MAX_MS (default "4000")
- SPARK_ADVISORY_PREFETCH_QUEUE (default "1")
- SPARK_ADVISORY_INCLUDE_MIND (default "0")
- SPARK_ADVISOR_MIND_MAX_STALE_S (default "0", disabled)
- SPARK_ADVISOR_MIND_STALE_ALLOW_IF_EMPTY (default "1")
- SPARK_ADVISOR_MIND_MIN_SALIENCE (default "0.5")
- SPARK_ADVISORY_REQUIRE_ACTION (default "1")
- SPARK_ADVISORY_STALE_S (default "900")
- SPARK_ADVISORY_TEXT_REPEAT_COOLDOWN_S (default "1800")
- SPARK_ADVISORY_EMIT (default "1")
- SPARK_ADVISORY_MAX_CHARS (default "500")
- SPARK_ADVISORY_FORMAT (default "inline")
- SPARK_SYNTH_MODE (auto|ai_only|programmatic)
- SPARK_SYNTH_TIMEOUT (default "3.0")
- SPARK_OLLAMA_API (default http://localhost:11434)
- SPARK_OLLAMA_MODEL (default "phi4-mini")
- SPARK_OPENAI_MODEL / SPARK_ANTHROPIC_MODEL / SPARK_GEMINI_MODEL (optional overrides)

Embeddings:
- SPARK_EMBEDDINGS (default "1", set 0/false/no to disable)
- SPARK_EMBED_MODEL (default "BAAI/bge-small-en-v1.5")

Workspace/context:
- SPARK_WORKSPACE (default ~/clawd)
- SPARK_AGENT_CONTEXT_MAX_CHARS (optional override)
- SPARK_AGENT_CONTEXT_LIMIT (optional override)

Logging:
- SPARK_DEBUG (enables verbose logs)
- SPARK_LOG_DIR (override ~/.spark/logs)
- SPARK_LOG_TEE (default "1")

Chips:
- SPARK_CHIP_SCHEMA_VALIDATION (warn|block)
- SPARK_CHIP_MIN_SCORE (default 0.35, discard lower-scored chip insights)
- SPARK_CHIP_MIN_CONFIDENCE (default 0.7, balanced gate confidence floor)
- SPARK_CHIP_GATE_MODE (balanced|off)
- SPARK_CHIP_PREFERRED_FORMAT (single|multifile|hybrid, default multifile)
- SPARK_BRIDGE_STEP_TIMEOUT_S (default 45)
- SPARK_BRIDGE_DISABLE_TIMEOUTS (1=true to disable per-step bridge timeouts)
- SPARK_STARTUP_READY_TIMEOUT_S (default 12)
- SPARK_STARTUP_READY_POLL_S (default 0.4)

Skills:
- SPARK_SKILLS_DIR (path to skills repository)

Clawdbot adapter:
- SPARK_CLAWDBOT_WORKSPACE / CLAWDBOT_WORKSPACE
- SPARK_CLAWDBOT_TARGETS / CLAWDBOT_TARGETS
- SPARK_CLAWDBOT_CONTEXT_PATH / CLAWDBOT_CONTEXT_PATH
- CLAWDBOT_PROFILE

Moltbook adapter:
- MOLTBOOK_API_KEY

## 6) Known gaps / mismatches

- Spark Pulse is the external vibeship-spark-pulse/app.py (set SPARK_PULSE_DIR to override location).
- advisory delivery status is surfaced in Pulse advisory/status APIs and Observatory stage pages, but action routing still depends on the OpenClaw agent/plugin side consuming those signals.
- Mind host is fixed to localhost; port override supported via `SPARK_MIND_PORT` (see `lib/ports.py`).
- build/ contains duplicated code artifacts; excluded from analysis.

## 7) Auto-generated sections below
## Chip inventory (auto summary)

### chips\bench-core.chip.yaml
- id: bench_core
- name: Benchmark Core Intelligence
- version: 0.1.0
- activation: opt_in
- risk_level: low
- domains: ['benchmarking', 'tooling', 'workflow']
- triggers.patterns: ['tool', 'command', 'file', 'prompt']
- triggers.events: ['post_tool', 'post_tool_failure', 'user_prompt', 'PostToolUse', 'PostToolUseFailure', 'UserPromptSubmit']
- observers:
  - tool_event triggers=['tool', 'command', 'file', 'write', 'edit', 'bash']
  - user_prompt triggers=['user', 'prompt', 'prefer', 'rather', 'instead', 'why']
- outcomes.positive:
  - condition=status == success weight=1.0 insight=User/tool signaled success
- outcomes.negative:
  - condition=status == failure weight=1.0 insight=User/tool signaled failure

### chips\biz-ops.chip.yaml
- id: biz-ops
- name: Business Ops Intelligence
- version: 0.1.0
- activation: opt_in
- risk_level: medium
- domains: ['business_ops', 'strategy', 'pricing']
- triggers.patterns: ['pricing experiment', 'pricing test', 'revenue forecast', 'runway', 'ops brief', 'operational risk', 'budget forecast']
- triggers.events: ['post_tool', 'post_tool_failure', 'user_prompt', 'PostToolUse', 'PostToolUseFailure', 'UserPromptSubmit', 'pricing_experiment', 'forecast_created', 'ops_plan']
- observers:
  - ops_brief triggers=['ops brief', 'operational risk', 'ops dependency', 'operational dependency', 'ops dependencies', 'operational dependencies']
  - pricing_experiment triggers=['pricing experiment', 'pricing test', 'pricing plan', 'pricing variant', 'experiment plan']
  - forecast triggers=['forecast', 'runway', 'forecast assumptions', 'revenue projection', 'burn projection']
- outcomes.positive:
  - condition=success_metric == retention weight=1.0 insight=Pricing experiment uses retention metric
- outcomes.negative:
  - condition=assumption == missing weight=1.0 insight=Forecast missing assumptions
- questions:
  - pricing_ethics: What pricing behaviors are off-limits for this business?
  - ops_success: Which operational outcomes matter most for this sprint?

### chips\examples\marketing-growth.chip.yaml
- id: marketing-growth
- name: Marketing Growth Intelligence
- version: 1.0.0
- activation: opt_in
- domains: ['marketing', 'growth', 'campaigns', 'acquisition']
- triggers.patterns: ['CTR was', 'conversion rate', 'CAC is', 'ROAS', 'campaign performed', 'audience responded', 'A/B test showed', 'email open rate', 'click through']
- triggers.events: ['user_prompt']
- observers:
  - campaign_metric triggers=['CTR', 'conversion', 'CAC', 'ROAS', 'performed']
  - ab_test_result triggers=['A/B test', 'variant', 'control']
  - audience_signal triggers=['audience', 'segment', 'responded']
- learners:
  - channel_roi type=correlation
  - message_resonance type=pattern
  - timing_pattern