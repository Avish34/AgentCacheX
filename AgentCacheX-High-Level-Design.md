# AgentCacheX: High-Level Design

**Date:** 2026-09-11  
**Status:** Proposed design; no implementation or measured performance claims  
**Initial scope:** One configured C# repository; file/symbol explanations and static dependency tracing

## 1. Recommendation

Build a **dependency-aware evidence cache**, not a general-purpose LLM answer cache.

Combine three mechanisms:

1. **Content-addressed source and tool-result caching** to avoid repeated repository work.
2. **Snapshot-qualified evidence retrieval** to reuse supported discoveries across differently worded requests.
3. **A deterministic context compiler** to send only relevant, cited evidence to the answering model.

Use a C#/.NET service with the official MCP C# SDK, Roslyn, Git, and SQLite FTS5. Add vector retrieval behind a feature flag after establishing the exact/lexical baseline. Keep the first implementation a modular application, not a collection of microservices.

The useful analogy is a **build cache for code understanding**: reuse an artifact only when its declared inputs and interpretation context still match. Bazel's action cache and content-addressable store illustrate this separation [bazel]. AgentCacheX borrows that pattern; it does not require Bazel, and an LLM-generated interpretation is not automatically correct or reproducible.

**Product boundary:** AgentCacheX supplies grounded context. The existing coding agent still reasons, answers, and decides what code to change. It is neither a replacement agent runtime nor a transparent universal LLM proxy.

### What is different from the existing project material?

The decks correctly focus on cited evidence rather than answer replay. This design makes four operational details explicit:

- Semantic similarity and AST similarity are candidate-selection signals, not freshness guarantees.
- An unchanged file and its previously observed callees are not always enough: project configuration, added overloads, partial classes, and other binding inputs can change meaning.
- A local cache proves the reuse mechanism, but cross-developer reuse needs a shared service.
- Registering an MCP server does not automatically intercept another tool's calls or expose IDE selection.

### Solution alternatives

These are alternative ways to reduce repeated work, not interchangeable products. The tradeoff ratings are engineering judgments for this MVP, not benchmark results.

| Approach | High-level design | What it solves | Main limitation | Verdict |
|---|---|---|---|---|
| **Provider prompt/KV caching** | Keep stable instructions/context prefixes; use the provider's cache | Repeated model input processing and eligible input charges | Does not avoid repository exploration or prove code freshness; still generates an answer | Enable as a complementary baseline, not the product [prompt-cache] |
| **Exact tool-result cache** | Canonical tool key -> SQLite/Redis -> original tool on miss | Identical file reads, searches, and symbol queries | Replaying the same large tool output need not reduce model context or agent reasoning | Best first vertical slice; insufficient alone |
| **Semantic response cache** | Prompt embedding + scope filters -> RedisVL/GPTCache -> replay answer on accepted hit | Can bypass a full LLM call for narrowly equivalent requests | Similar wording does not prove equivalent target, context, repository state, or requested behavior | Useful for tightly constrained immutable queries; not the default coding workflow [redisvl] [gptcache] |
| **Incremental code RAG / repository index** | Parse source -> exact/lexical/vector index -> retrieve current snippets -> answer | Avoids repeated discovery and full-file context loading | Must still solve snapshot freshness; does not inherently retain prior supported analysis | Strongest simpler competitor to the recommendation; benchmark it explicitly |
| **Agent-memory / temporal-graph system** | Ingest episodes or memories -> extract facts/relations -> retrieve relevant knowledge | Persistent discoveries, relationships, and historical context | Memory timestamps, expiration, and knowledge supersession do not establish local Git dependency freshness | Consider for history/relationship-heavy use cases, not as the core correctness layer [graphiti-edges] [mem0-implementation] |
| **Dependency-aware evidence cache** | Exact artifacts + reusable evidence + source manifests + scoped retrieval + context compiler | Avoids tool work, reuses supported discoveries, and reduces context while explaining freshness | Requires custom correctness logic and honest dependency coverage | **Recommended**, using exact caching and incremental retrieval as its foundation |

**Why not just a universal caching proxy?** An LLM proxy sees requests sent to that endpoint, not necessarily the tools, files, IDE buffers, or source versions that produced them. A proxy could participate in a larger host integration, but changing one endpoint cannot supply the missing repository contract.

**Why not vector search alone?** Vector indexes answer "which stored items are similar?" The important question here is "which supported evidence still applies to this target in this source snapshot?" Those are separate operations.

**Why not build everything ourselves?** Storage, lexical search, parsers, MCP transport, and optional vector search already have reusable implementations. Build the source/evidence identity, dependency policies, validation states, coverage checks, and context compiler that connect them.

### Concrete storage and cache choices

| Component | Verified capabilities / caveats | Decision |
|---|---|---|
| **SQLite + FTS5** | Transactions and embedded storage; FTS5 supports lexical ranking, phrase/prefix queries, and tokenization. External-content FTS indexes need application-maintained consistency. SQLite is public domain [sqlite-fts] [sqlite-license]. | Authoritative local data, exact keys, and lexical index |
| **sqlite-vec** | SQL-accessible vector search with metadata/partition features documented in its source; MIT/Apache-2.0 licensing. The project explicitly remains pre-v1, so development-branch features may differ from the packaged artifact [sqlite-vec]. | Optional local semantic index, pinned and isolated behind an interface |
| **Qdrant** | Vector retrieval and payload filtering; Apache-2.0. Requires a service and correctly configured access controls. Filtering does not discover Git dependencies or establish freshness [qdrant] [qdrant-filtering]. | Preferred later shared retrieval index, not the provenance authority |
| **Redis + RedisVL** | Exact cache keys/expiry plus a semantic-cache library with thresholds, filters, metadata, and TTL. RedisVL is MIT; Redis 8+ server licensing is a choice of AGPLv3, RSALv2, or SSPLv1 [redis-set] [redisvl] [redis-license]. | Reuse if already operated and useful; do not add a separate Redis service for the MVP |
| **GPTCache** | Embedding/search/similarity/response-cache pipeline; MIT. Research found default-branch activity in July 2025 and a most recent returned GitHub release in August 2024, warranting a maintenance review rather than an assumption of current activity [gptcache] [gptcache-release]. | Design reference, not the default dependency |

TTL is a retention policy, not source validation. In particular, RedisVL's semantic-cache implementation can refresh TTL on access, so a frequently used obsolete entry can remain present without an independent freshness gate [redisvl-cache].

The recommended implementation stack is:

```text
C#/.NET + official MCP C# SDK
Roslyn + Git + explicit saved-workspace snapshots
SQLite relational tables + exact identifier keys + FTS5
optional: embedding provider + sqlite-vec
later team deployment: authenticated service
later scale-out: PostgreSQL authority + optional Qdrant index
```

For semantic retrieval, version the embedding model, tokenizer/preprocessing, dimensions, and normalization alongside every vector. Use a reviewed local embedding model or an explicitly approved endpoint; do not send private code to an arbitrary service. Embed compact card text, not entire conversations. Benchmark model/runtime choices on this workload before selecting one; a vector database does not itself generate useful embeddings.

The research establishes architectural fit, not packaged dependency compatibility, operating cost, or a claim that no competing product exists.

### Memory and ingestion frameworks: adopt or replace?

| Framework | What the inspected implementation provides | Missing application contract | Recommendation |
|---|---|---|---|
| **LlamaIndex ingestion/cache** | Hashes supplied node content/metadata and transformation configuration, caches output nodes, and supports document-ID/hash-based change handling [llama-pipeline] [llama-cache] | Git/workspace acquisition, semantic dependencies, coherent snapshot identity, and read-time evidence validation | Closest reusable ingestion component; use selectively if a Python sidecar is already justified, not merely to persist a few records |
| **Graphiti** | Episodes, supporting episode IDs, embedded fact edges, temporal validity fields, and entity/edge extraction/resolution [graphiti-edges] [graphiti-ingestion] | Source-file observation and dependency-aware freshness; temporal validity is about supplied knowledge | Later extension for architecture history, supersession, or relationship-heavy questions |
| **Mem0 OSS** | Searchable text memories, hashes, metadata/scoping, embeddings, CRUD/history, and extraction [mem0-implementation] | Source lineage schema, working-tree validation, dependency invalidation, and evidence support | Useful if its memory API simplifies a broader application; not the authority for current code facts |

Graphiti and Mem0 OSS are Apache-2.0; LlamaIndex core is MIT. Database, model, and hosted-service licensing must be reviewed separately [graphiti-license] [mem0-license] [llama-license].

Important integration details from the inspected source snapshots:

- Graphiti includes an OSS MCP server, but accepted ingestion may be queued rather than immediately searchable. Its direct-triplet path can still generate embeddings and perform LLM-backed resolution; it is not automatically a zero-model-cost write [graphiti-mcp] [graphiti-ingestion].
- Mem0's documented hosted MCP endpoint stores memory in the Mem0 Platform account. An OSS implementation exists in OpenMemory's archive, but that does not establish a current drop-in pairing. A thin custom MCP adapter remains the controlled local option [mem0-hosted-mcp] [mem0-archived-mcp].
- The inspected Mem0 ingestion implementation is additive, despite older/docstring descriptions of automatic add/update/delete reconciliation. Pin and inspect the chosen implementation rather than relying on those descriptions. `infer=False` avoids extraction, not embedding work [mem0-implementation].
- LlamaIndex transformation keys depend on the supplied node sequence, so changing batch composition can affect reuse. Its inspected upsert/delete path requires the corresponding stores; deletion strategies must see the complete intended collection, not just a changed-file batch [llama-pipeline].
- LlamaIndex supplies an OSS workflow-to-MCP adapter. That can expose a custom validation workflow, but the ingestion cache alone is not a repository-evidence service [llama-mcp].

These distinctions favor the small C# implementation for this C#-focused project. Reuse a framework only when its additional capabilities repay the sidecar, storage, model, and integration overhead.

The source review found **no end-to-end local working-tree/dependency freshness guarantee in the inspected framework contracts**. This is a bounded finding about those contracts, not a claim that custom integrations or competing solutions cannot provide one.

## 2. Requirements and boundaries

| Requirement | Design response |
|---|---|
| Avoid repeating source reads, searches, and analysis | Cache structured tool artifacts and reusable evidence |
| Support rephrased questions | Normalize explicit intent/target/aspects; lexical retrieval first, optional semantic recall |
| Remain correct after code changes | Validate declared source, search-scope, and compilation dependencies |
| Reduce model context | Rank, deduplicate, select, and cite under a token budget |
| Preserve useful work across unrelated commits | Separate immutable artifacts from current-workspace validation |
| Explain every reuse decision | Return snapshot, evidence lineage, missing coverage, and rejection reasons |
| Support multiple agents | Explicit MCP tools with stable contracts; persistent storage independent of chat sessions |
| Eventually reuse across developers | Authenticated shared cache for clean repository snapshots, with local revalidation |

The MVP describes **static source behavior in a known repository and compilation context**. It does not assert live runtime behavior dependent on a database, remote service, secret, feature flag, or reflection target that it has not observed.

Exclude final-patch replay, shell-trajectory replay, arbitrary non-code questions, multi-repository semantic merging, graph-database deployment, and automatic whole-repository LLM summarization.

Use saved files initially. Unsaved editor buffers require an explicit host adapter carrying buffer content/version; without one, report the saved-file snapshot rather than pretending to know the editor state.

## 3. High-level architecture

Open [the interactive architecture diagram](AgentCacheX-Architecture.html) for the request flow, storage/invalidation, and optional team-sharing views. It works offline and includes zoom, print/PDF, and Mermaid-source export.

```text
 Developer / IDE / coding agent
   request + explicit target + requested aspects
                         |
                 MCP over local stdio
                         |
 +----------------------------------------------------------+
 | AgentCacheX local adapter                                |
 |                                                          |
 | Request/scope resolver -> exact + lexical [+ vector]      |
 |                                candidate retrieval       |
 |                                      |                   |
 | Git/workspace snapshot <------ freshness + coverage gate  |
 | Roslyn syntax/semantics                |                   |
 |                                      v                   |
 | Cached read/search tools <---- missing-evidence plan       |
 |          |                                               |
 |          v                                               |
 | Source artifacts + evidence -> context compiler           |
 |                                      |                   |
 | L1 SQLite: metadata, artifacts, dependencies, FTS5         |
 | Optional vectors; audit and usage metrics                 |
 +----------------------------------------------------------+
                         |
           compact, cited, snapshot-qualified context
                         |
                Existing answering LLM

 Optional cross-developer path:

 Local adapter <--- authenticated API ---> Team cache service
                                           |
                              clean immutable snapshots only
                              trusted source acquisition
                              server-local relational store
```

### Module responsibilities

| Module | Responsibility |
|---|---|
| Scope resolver | Resolve configured repository ID, checkout, target, ref, project, principal, and requested aspects |
| Snapshot provider | Read actual source content; fingerprint workspace and compilation inputs |
| Roslyn analyzer | Produce declarations, symbol bindings, reference evidence, and anchored excerpts |
| Artifact cache | Cache supported reads/searches and parser outputs with dependency manifests |
| Evidence store | Preserve source lineage, derivation versions, evidence strength, and snapshot associations |
| Retriever | Apply hard scope filters; exact/FTS5 retrieval; optional semantic candidate recall |
| Freshness/coverage gate | Validate dependencies and determine whether requested aspects are actually covered |
| Context compiler | Select useful cards, preserve citations, deduplicate, and enforce the output budget |
| Metrics/audit | Explain hits and misses; record overhead, saved work, and rejected evidence |

Roslyn is preferred over a generic parser for this C# scope because its semantic model resolves symbols using source files, references, and compiler options [roslyn-semantics]. Its workspace model represents projects and their dependencies [roslyn-workspace]. Tree-sitter is an attractive later multi-language syntax frontend, not a substitute for C# semantic binding [tree-sitter].

## 4. Request and execution flows

### Cold request

1. The agent supplies intent, target path or symbol, requested aspects, and context budget.
2. AgentCacheX resolves the repository and a saved-workspace snapshot.
3. Lookup returns no usable evidence or identifies missing aspects.
4. Cached read/search tools acquire source and record the exact bytes, scope, and versions consumed.
5. Roslyn produces deterministic cards and source excerpts. The context compiler returns relevant evidence to the agent.
6. The existing LLM generates the answer. Optional narrative discoveries can be recorded as derived evidence, not self-certified facts.
7. Evidence is stored transactionally with its dependencies. If the source changed during the operation, the artifact remains tied to the observed snapshot, not relabeled as current.

Do not require a separate LLM summarization call for every file. Begin with deterministic extraction and lazy population of the requested subset.

### Warm request, including a paraphrase

1. Resolve scope and target again; do not infer identity from prompt similarity.
2. Retrieve exact intent/target/aspect candidates, then lexical candidates; use vectors only when useful.
3. Validate candidate dependency manifests against the requested snapshot.
4. Check coverage: a cached explanation of purpose is not a complete answer about error handling or transitive behavior.
5. Compile the valid subset with current citations. Return an explicit missing-evidence plan for uncovered aspects.
6. The answering LLM receives the original request plus compact evidence and generates a new answer.

A paraphrase can hit a structured `intent + target + aspects` key without vector search. Do not call such a demo proof of vector-based semantic matching.

### Source change

1. Detect changed files, source-set membership, and compilation/configuration inputs.
2. Invalidate current-workspace associations for affected evidence, including downstream derived records.
3. Preserve immutable historical artifacts.
4. Recompute only the necessary artifacts, using conservative project-level invalidation when fine-grained dependencies are incomplete.
5. Reuse unaffected file-local cards and unaffected project evidence.

File watchers are latency optimizations, not the source of truth. Lookup still validates the dependency manifest. Git status has a stable porcelain format and distinguishes tracked and untracked changes [git-status]; neither a watcher notification nor HEAD alone establishes freshness.

### Team reuse

1. Developer A's supported analysis of a clean, immutable snapshot becomes eligible for the team tier.
2. The hub acquires or verifies the canonical source and admits appropriately supported evidence.
3. Developer B searches the same authorized repository scope.
4. B's local adapter checks the returned dependencies against B's checkout before treating the evidence as current.
5. B's uncommitted modifications remain local; mismatching shared cards are rejected or partially refreshed.

Local L1 caches alone do **not** provide cross-developer reuse. This path is required before attributing team-wide savings to the product.

## 5. What to cache

Keep three types of records separate.

| Layer | Examples | Why separate? |
|---|---|---|
| Immutable artifacts | Source bytes, parser output, search results, minimal excerpts | Content identity and reproducibility of acquisition |
| Evidence | Declared symbol, resolved call, supported file responsibility, derived explanation | Reusable meaning with explicit scope and dependencies |
| Compiled context | Selected evidence IDs and rendered text for a particular request/budget | Request-specific; cannot become the authoritative source |

**Freshness is not semantic correctness.** A content hash proves that an input has not changed. It does not prove that an LLM's interpretation of that input was correct.

Track evidence strength separately:

- `source_excerpt`: exact source with verified span and content hash.
- `deterministic_fact`: a fact produced by a versioned analyzer, with its analysis limitations.
- `derived_summary`: generated prose linked to supporting evidence and derivation versions.
- `unverified`: insufficiently supported input, excluded from trusted reusable context.

The default context should prioritize excerpts and deterministic facts. Derived summaries remain labeled as derived and must not silently replace their supporting evidence. Confidence scores or approval by another model are not substitutes for source support.

### Conceptual data model

| Record | Essential fields |
|---|---|
| `repository_scope` | Immutable/configured repository ID, provider, principal partition, allowed roots |
| `snapshot` | Repository, checkout ID, ref, resolved commit, workspace manifest, project context hash, observation time |
| `source_artifact` | Content hash, source path, bytes/excerpt, encoding, parser version |
| `tool_result` | Tool/version, canonical arguments, scope, snapshot, result artifact IDs, completeness/index metadata |
| `evidence` | Type, claim/excerpt, target, aspects, source IDs, strength, extractor/prompt/model versions |
| `dependency` | Evidence/artifact ID, dependency kind, source/scope identity, expected fingerprint |
| `evidence_validation` | Evidence ID, requested snapshot, freshness state, coverage, reasons, validated time |
| `usage` | Request ID, decision, tool calls, emitted tokens, timings, refresh/extraction/embedding costs |

Do not store one global `fresh` flag on an evidence record. The same immutable evidence can be fresh for one checkout and historical or stale for another.

FTS and vector indexes are derived from authoritative records and can be rebuilt. Publish evidence, dependencies, and index updates coherently; an incomplete record is not queryable as supported current evidence.

## 6. Identity, freshness, and invalidation

### Separate acquisition identity from reuse eligibility

An exact acquisition key includes:

```text
repository ID + principal/scope + tool/version
+ canonical arguments + source snapshot/index version
+ schema version
```

An evidence candidate lookup uses:

```text
repository scope + target identity + intent/aspects
+ analyzer/derivation compatibility
```

The candidate is reusable only after checking its full dependency manifest. Do not put the current HEAD into every candidate key and invalidate the entire cache on every commit.

Authorization partitions use stable identities and policy versions, never raw tokens or secrets.

### Dependency classes

| Evidence/tool type | Minimum invalidation scope |
|---|---|
| File read or purely syntactic declaration | Exact consumed file bytes, path semantics, parser/options version |
| Symbol binding or behavioral explanation | Source dependencies plus applicable project/reference/compilation inputs |
| Project-wide or repository-wide search | Searched corpus membership/content and search implementation/index version |
| Negative search | Same searched corpus; newly added matching files invalidate the result |
| Derived summary | All supporting evidence plus extractor/prompt/model policy versions |
| Runtime-dependent observation | Explicit external snapshot; otherwise outside current-source guarantees |

For the MVP, fingerprint the relevant project and referenced-project source closure for semantic cards. This is deliberately more conservative than hashing only observed callees. Adding an overload, an extension method, or another partial declaration can change binding even when the old dependency files are unchanged.

Finer declaration/binding fingerprints can be introduced later when mutation tests demonstrate equivalent safety. Do not claim fully precise invalidation before this work exists.

Include applicable project files, imported props/targets, references and lockfiles, compilation options/defines, global usings, source-set membership, and relevant generated source. Surface unresolved workspace/build diagnostics. If analysis cannot establish the needed context, return limited syntax evidence or `suspect`, not a trusted behavioral explanation.

Avoid executing arbitrary repository build steps as part of an untrusted lookup. Semantic project loading requires an explicitly trusted repository and controlled build context.

### Freshness states versus response decisions

Per-evidence freshness:

| State | Meaning |
|---|---|
| `fresh` | Dependencies were validated for the requested observed snapshot |
| `historical` | Valid only for the recorded immutable snapshot |
| `suspect` | Required validation or dependency coverage could not be established |
| `stale` | A required dependency no longer matches |
| `tombstoned` | Source was removed or the record was revoked |

Per-request decision: `hit`, `partial`, or `miss`, with diagnostic reasons such as `dependency_changed`, `coverage_missing`, or `validation_unavailable`. A partial response contains only usable cards; stale and suspect candidates are not silently served as current.

### Edge cases that must be designed in

- **Dirty working tree:** use actual consumed bytes, including staged/unstaged changes and relevant new files. HEAD is only one part of the snapshot.
- **Unsaved buffer:** require an explicit versioned overlay or declare saved-file-only scope.
- **Branch switch:** resolve and validate the new scope. Do not inherit a previous branch's current status.
- **Comments/formatting:** AST equality may suggest reuse candidates, but source hashes and citations must be updated. Comments can be relevant evidence themselves.
- **Rename/deletion:** verify source identity, project membership, path effects, and citation re-anchoring. Matching bytes alone do not prove a move preserved semantics.
- **New overload/configuration change:** invalidate or rebind semantic cards even if the target file is unchanged.
- **Incomplete call graph:** conservatively include project context or withhold unsupported transitive behavior.
- **Negative/incomplete remote search:** do not claim current completeness if the provider's index snapshot cannot be established.
- **Concurrent edit:** bind the response to an observed snapshot and recheck before publication. If inputs changed, retry within a bounded policy or report an unstable snapshot. No guarantee extends to edits after the response.

## 7. Retrieval and context compilation

Use this order:

1. Enforce repository, principal, target, language/project, ref, and requested-aspect constraints.
2. Attempt exact structured lookup.
3. Search FTS5 for symbols, identifiers, paths, and related prose.
4. Optionally retrieve semantic candidates within the same allowed scope.
5. Validate source dependencies and discard incompatible candidates.
6. Rank by coverage, evidence strength, source relevance, and diversity.
7. Deduplicate overlapping spans and redundant claims.
8. Emit essential supporting snippets, citations, snapshot, and missing aspects within the budget.

Keep exact symbol/path indexes as well as FTS5: text tokenization alone should not decide identifier identity. Escape or parameterize user search input rather than treating arbitrary prompts as FTS query syntax.

Semantic similarity is neither a probability of correctness nor a universal threshold. Calibrate retrieval on labeled questions; do not adopt a blanket "95% similar means safe" rule.

AST fingerprints are a secondary matching/indexing aid. Preserve identifiers and literals; do not automatically merge differently named functions or behaviorally similar-looking code.

The context budget applies to the serialized tool result, including citations and metadata. Preserve whole evidence units rather than truncating identifiers, JSON, or source spans. If the model tokenizer is unavailable, report an estimate, not an exact token count.

The host still adds its system prompt, tool schemas, conversation, and user request. A 1,500-token evidence budget is **not** necessarily a 1,500-token total model request.

## 8. MCP integration contract

MCP exposes discoverable tools; those tools are generally model-controlled [mcp-tools]. Copilot supports explicitly configured local stdio servers [copilot-mcp]. The official C# SDK provides hosting and HTTP integration packages [mcp-csharp].

Proposed tool names below are AgentCacheX design contracts, not existing SDK features.

| Tool | Input | Output |
|---|---|---|
| `cache_lookup` | Query, intent, target, requested aspects, scope, token budget | Snapshot-qualified cards, hit/partial/miss, missing aspects, reasons |
| `code_read_cached` | Configured scope, path, optional symbol/span | Versioned source artifact, minimal excerpt, provenance |
| `code_search_cached` | Scope, query, filters | Cached or newly acquired search result with scope/completeness metadata |
| `cache_record` | Evidence type, source artifact IDs, claim/aspects, derivation metadata | Accepted evidence ID or explicit rejection; never automatic trust |
| `cache_explain` | Evidence/lookup ID | Dependencies, source hashes, validation history, selection/rejection reasons |

The source adapter should generate hashes and source IDs itself. Do not accept a model-supplied commit, citation, or `fresh=true` as authoritative.

Start with agent instructions to call `cache_lookup` before exploration and to use the cached read/search tools for supported work. The adapter captures those results directly rather than relying on the model to remember every write.

**Integration limitation:** an MCP server cannot automatically wrap all other MCP servers, native IDE reads, shell commands, or arbitrary LLM traffic. Unsupported tools remain outside capture. Report the percentage of eligible exploration routed through AgentCacheX.

A guaranteed interception path would require an explicit host/tool-router integration. Changing an LLM `BASE_URL` alone does not provide the selected file, filesystem snapshot, or complete tool dependencies.

## 9. Deployment and concurrency

### Local proof of concept

Use a C# stdio MCP process and SQLite on a local disk. Persist the cache outside the repository and outside cloud-synced folders, for example:

```text
%LOCALAPPDATA%\AgentCacheX\<repository-id>\cache.db
```

This project folder is currently under OneDrive. Keep design/code there if desired, but do not synchronize the live database, WAL, or lock files.

Multiple local MCP processes may use a correctly configured SQLite database on the same machine. Use WAL, short transactions, bounded busy handling, unique/idempotent inserts, and snapshot-aware publication. SQLite still has one writer at a time [sqlite-wal].

Do not hold a database write transaction while calling an LLM, computing embeddings, or doing network I/O. An in-process single-flight mechanism can reduce duplicate work; correctness must not depend on it across processes.

Set size limits and retention for raw artifacts; preserve references while evidence is retained. Tombstoning removes an item from retrieval, while explicit deletion must also clean associated indexes and stored content according to policy.

### Team demonstration

If the hackathon promises reuse across developers, add a small authenticated hub after the local kernel works. A single service instance with server-local SQLite can demonstrate this; multiple machines access the service, never the database file.

The hub serves clean, immutable snapshots from a trusted repository source. Keep dirty worktree evidence in the local principal/checkout partition. Treat client-generated narrative as unverified until admitted under the same source-support rules.

Even for the demo, network exposure requires authentication, repository authorization, transport protection, and bounded requests. Enterprise role administration can remain out of scope; access isolation cannot.

### Scale-out, only when justified

Move authoritative shared records to a server database such as PostgreSQL when multiple service writers, availability requirements, or operational needs justify it. Add Qdrant as a rebuildable hybrid retrieval index if measured retrieval volume requires it.

Use an outbox/versioned publication mechanism between the authoritative database and an external index. Index staleness may reduce recall, but must never bypass authoritative access, revocation, or freshness checks.

No Redis, message broker, graph database, or vector cluster is required to prove the core idea.

## 10. Safety and failure behavior

- Treat repository text and cached narrative as untrusted data, not instructions to execute.
- Confine paths to configured roots and honor source exclusion/access policies before acquisition and reuse.
- Store no credentials or secret-bearing tool arguments. Limit source capture to approved files and redact sensitive telemetry.
- Default to local processing; remote embedding or storage requires an approved destination and data policy.
- Distinguish an ordinary miss from cache/provider failure. Surface diagnostics and let the agent explicitly continue with normal exploration.
- Do not serve last-known-good evidence as current when validation fails.
- Namespace all data by repository and principal. Recheck authorization on lookup and source access.
- Keep mutations, shell execution, and patch application out of the reusable evidence path.

## 11. Evaluation and decision gates

Compare four configurations on the same tasks and repository states:

1. Existing agent with its normal provider prompt caching.
2. Exact tool-result cache only.
3. Incremental source retrieval plus deterministic context compilation, without learned narrative evidence.
4. Full evidence reuse, with lexical-only and optional vector variants.

This isolates whether the additional evidence layer actually beats simpler retrieval. If configuration 3 delivers the same quality and savings, ship that simpler implementation first.

Use a labeled set of approximately 30 representative file/symbol tasks, with multiple paraphrases and distinct requested aspects. Separate calibration from evaluation questions. Repeat runs to report distributions, not a single favorable trace.

Include mutations for direct edits, transitive changes, unrelated project edits, same-project binding changes, deletes/renames, new search matches, branch switches, configuration changes, missing build context, and source edits during lookup. Include cases where two similarly named symbols have different behavior.

| Metric | Proposed gate or interpretation |
|---|---|
| Cache precision | At least 95% of served reusable evidence is relevant and supported under the evaluation rubric |
| Known stale current-state serves | Zero in the mutation suite; any observed stale serve blocks rollout |
| Citation coverage | Every reusable factual card has a resolvable source anchor for its stated snapshot |
| Answer quality | No material regression versus the baseline on the same rubric |
| Lookup latency | Target p95 below one second for the stated corpus/hardware; separate startup from steady state |
| Total task cost | Include all model calls, cached-input pricing, embeddings, refresh work, and service overhead |
| Context reduction | Count actual full model inputs where available, not just evidence payload size |
| Exploration reduction | Separate backend requests, MCP invocations, local hash reads, and full-file reads |
| Coverage | Report eligible task fraction and fraction actually routed through the cache |

Zero observed stale results in a finite suite is not a proof that stale results are impossible. Source-support judgments also need a rubric; matching hashes alone do not establish cache precision.

### Interpret the deck numbers correctly

The existing 8,000 -> 1,500 input tokens, 18 -> 4 seconds, and 12 -> 3 repository calls are illustrative **per-warm-hit** assumptions.

At a hypothetical 60% usable hit rate, with misses unchanged and no added overhead:

| Metric | Baseline per request | Workload average with cache | Overall reduction |
|---|---:|---:|---:|
| Input tokens | 8,000 | 4,100 | 48.75% |
| Latency | 18 seconds | 9.6 seconds | 46.67% |
| Repository tool calls | 12 | 6.6 | 45% |

Thus an 81.25% reduction on a warm hit is not an 81.25% reduction across the entire workload.

For 20 developers x 15 requests/day x 220 days, those same assumptions yield 66,000 requests, 39,600 warm hits, 257.4 million avoided input tokens, 356,400 avoided repository calls, and 154 hours of aggregate avoided wait. These are models, not product results; parallel wait time is not automatically recovered developer labor.

Let `H` be the usable hit rate, `C0` baseline cost, `Ch` hit cost including validation, `Om` extra miss cost, and `A` amortized indexing/extraction cost per request:

```text
Expected cost = H * Ch + (1 - H) * (C0 + Om) + A
Net saving requires H * (C0 - Ch) > (1 - H) * Om + A
```

Apply the model separately to dollars, compute, and latency as appropriate. Account for provider cached-input pricing rather than multiplying all avoided tokens by the uncached rate. Provider prompt caching reuses a matching prefix and still produces a new response; it is complementary to AgentCacheX [prompt-cache].

## 12. Hackathon implementation sequence

| Checkpoint | Deliverable | Stop/go criterion |
|---|---|---|
| 1. Capture and exact reuse | MCP host, explicit target, SQLite schema, cached read/search, metrics | A second identical request reuses correctly scoped artifacts |
| 2. Useful context | Roslyn cards, FTS5, citations, budgeted context compiler | Same-task answers need less context without losing required evidence |
| 3. Correctness under change | Workspace/project fingerprints, invalidation, partial results, mutation cases | No known stale current-state serves; uncertain cases visibly miss |
| 4. Broader reuse | Optional semantic retrieval; calibrated candidates; evidence recording policy | Improves recall or cost beyond structured/lexical matching |
| 5. Team path | Authenticated hub for clean snapshots; local overlay validation | A separate user/checkout can reuse supported shared evidence safely |
| 6. Demo and report | Cold/warm/change flow with cost breakdown | Results distinguish modeled assumptions from actual observations |

Checkpoints are scope gates, not promised day estimates. If time is limited, complete 1-3 rather than replacing freshness with a similarity threshold. If only one machine is demonstrated, explicitly describe team reuse as the next deployment step.

### Parallel team work

Agree on the snapshot, evidence-card, and lookup-response contracts first.

| Workstream | Owned boundary |
|---|---|
| Storage/retrieval | SQLite schema, exact keys, FTS5, optional vectors, retrieval APIs |
| Source/freshness | Git snapshots, Roslyn extraction, dependency manifests, mutation fixtures |
| Agent integration | MCP contracts, cached acquisition tools, context compiler, capture coverage |
| Evaluation/demo | Baseline runner, labeled questions, metrics report, demo script |

Use separate branches and sessions. Integrate against the shared contracts rather than independently inventing card formats.

## 13. Main risks and deliberate tradeoffs

| Risk | Decision |
|---|---|
| Incomplete dependency graph creates false freshness | Start with conservative semantic scope fingerprints; optimize later |
| Accurate cache is slower than normal exploration | Benchmark end-to-end; bounded refresh and normal exploration on explicit failure |
| Narrative is stale or unsupported despite good citations | Separate source freshness from evidence strength; prioritize deterministic evidence |
| New tools are not consistently invoked | Explicit integration, cached acquisition tools, and capture-rate measurement |
| Embeddings add complexity without useful recall | Feature flag and lexical baseline; require measured incremental value |
| Entire repository analysis is too expensive | Lazy population and a declared pilot subset |
| Local success is mistaken for team-wide reuse | Separate local and shared deployments and report their hit rates independently |
| Concurrent edits make "current" ambiguous | Snapshot-qualified answers and bounded revalidation, not a timeless guarantee |

**Decision:** build the smallest source-grounded reuse kernel that beats incremental retrieval. Add semantic recall and team distribution only where they produce measurable value without weakening freshness or access boundaries.

## Source references

Public documentation and source were reviewed for this design on 2026-09-11. Source-pinned implementation links below distinguish inspected behavior from potentially different package releases. No project source was uploaded to public research services; SVG/PNG assets were not opened.

The local inputs were `GeminiResponse.txt`, `sharedrp-agent-memory-layer-research-2026-08-31.md`, and slide text from `AgentCacheX-Coding-Agent-Cache.pptx` and `AgentCacheX-Neon-Architecture.pptx`. The Keynote-Style deck was unavailable due to a file lock during the initial review.

[bazel]: https://bazel.build/remote/caching
[roslyn-semantics]: https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/work-with-semantics
[roslyn-workspace]: https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/work-with-workspace
[tree-sitter]: https://github.com/tree-sitter/tree-sitter
[git-status]: https://git-scm.com/docs/git-status
[mcp-tools]: https://modelcontextprotocol.io/specification/2025-06-18/server/tools
[mcp-csharp]: https://github.com/modelcontextprotocol/csharp-sdk
[copilot-mcp]: https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-mcp-servers
[prompt-cache]: https://developers.openai.com/api/docs/guides/prompt-caching
[sqlite-fts]: https://sqlite.org/fts5.html
[sqlite-license]: https://sqlite.org/copyright.html
[sqlite-wal]: https://sqlite.org/wal.html
[sqlite-vec]: https://github.com/asg017/sqlite-vec
[qdrant]: https://github.com/qdrant/qdrant
[qdrant-filtering]: https://qdrant.tech/documentation/search/filtering/
[redis-set]: https://redis.io/docs/latest/commands/set/
[redisvl]: https://github.com/redis/redis-vl-python
[redisvl-cache]: https://github.com/redis/redis-vl-python/blob/ff12ec2fbdc1e23ca0e413467e4953516f7ab0c9/redisvl/extensions/cache/llm/semantic.py
[redis-license]: https://redis.io/legal/licenses/
[gptcache]: https://github.com/zilliztech/GPTCache
[gptcache-release]: https://github.com/zilliztech/GPTCache/releases/tag/0.1.44
[graphiti-edges]: https://github.com/getzep/graphiti/blob/c64e45c111fd43ca7f5536a7f84a73f6acafd5b1/graphiti_core/edges.py#L263-L285
[graphiti-ingestion]: https://github.com/getzep/graphiti/blob/c64e45c111fd43ca7f5536a7f84a73f6acafd5b1/graphiti_core/graphiti.py#L1704-L1803
[graphiti-mcp]: https://github.com/getzep/graphiti/blob/c64e45c111fd43ca7f5536a7f84a73f6acafd5b1/mcp_server/src/graphiti_mcp_server.py#L446-L493
[graphiti-license]: https://github.com/getzep/graphiti/blob/c64e45c111fd43ca7f5536a7f84a73f6acafd5b1/LICENSE
[mem0-implementation]: https://github.com/mem0ai/mem0/blob/c7ee362aff94a369af70f13f2b4f853f6793ff4c/mem0/memory/main.py#L879-L1060
[mem0-hosted-mcp]: https://github.com/mem0ai/mem0/blob/c7ee362aff94a369af70f13f2b4f853f6793ff4c/docs/platform/mem0-mcp.mdx#L8-L30
[mem0-archived-mcp]: https://github.com/mem0ai/openmemory/blob/2df78cc48e5f688a771fee5b39899325accc2450/openmemory-archive/api/app/mcp_server.py#L24-L95
[mem0-license]: https://github.com/mem0ai/mem0/blob/c7ee362aff94a369af70f13f2b4f853f6793ff4c/LICENSE
[llama-pipeline]: https://github.com/run-llama/llama_index/blob/5fb3f398b1271edd0ba297912dbfa3cce7d58632/llama-index-core/llama_index/core/ingestion/pipeline.py
[llama-cache]: https://github.com/run-llama/llama_index/blob/5fb3f398b1271edd0ba297912dbfa3cce7d58632/llama-index-core/llama_index/core/ingestion/cache.py
[llama-mcp]: https://github.com/run-llama/llama_index/blob/5fb3f398b1271edd0ba297912dbfa3cce7d58632/llama-index-integrations/tools/llama-index-tools-mcp/llama_index/tools/mcp/utils.py#L77-L142
[llama-license]: https://github.com/run-llama/llama_index/blob/5fb3f398b1271edd0ba297912dbfa3cce7d58632/LICENSE
