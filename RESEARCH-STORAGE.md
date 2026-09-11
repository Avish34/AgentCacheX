# AgentCacheX: Storage and Cache Research

**Assessment date:** 2026-09-11  
**Scope:** Public infrastructure for local-first and shared repository-aware coding-agent evidence caching  
**Method:** Official documentation and source inspection; no workload benchmarks, managed-service quotes, or packaged compatibility measurements

This report preserves the detailed storage/cache findings used to produce the HLD.

## Recommendation

Use **SQLite + FTS5 locally, with sqlite-vec optional**. At shared scale, use **Qdrant as a rebuildable retrieval index**, not the authority for evidence or freshness.

Add Redis only when measured repetition justifies a shared response/tool-result cache. GPTCache offers reusable response-cache machinery, but its observed maintenance activity makes it a weaker default than RedisVL for that particular role.

## Distinguish the cache layers

| Layer | What gets reused | What it can save |
|---|---|---|
| Provider prompt/KV caching | Model-internal state for a matching prompt prefix | Repeated input processing and eligible input charges; still generates a new response |
| Response caching | A previously generated answer | A valid hit can bypass that model call, including generation |
| Evidence/tool-result caching | Searches, excerpts, symbol results, dependency analysis, or other tool outputs | Repeated retrieval/tool work; potentially fewer iterations and smaller prompts if the application selects bounded evidence |

A vector-search hit, response-cache hit, and provider-prefix hit are not interchangeable. These mechanisms can be complementary [10].

## Four infrastructure alternatives

| Alternative | Reusable capabilities | Limitations and operating burden |
|---|---|---|
| **SQLite FTS5 + optional sqlite-vec** | FTS5 lexical matching, phrases, prefixes, relevance ranking, and configurable tokenization. sqlite-vec adds SQL-accessible KNN with metadata constraints and partition keys. Ordinary relational tables retain authority [1] [2]. | Best local default: local storage/index CPU and optional embeddings, without a separate retrieval service. FTS tokenization is not exact programming-language identifier equality. External-content synchronization remains the application's responsibility. SQLite is public domain; sqlite-vec has MIT/Apache-2.0 licenses and warns that it is pre-v1 [1] [2] [3]. |
| **Qdrant + authoritative provenance elsewhere** | Vector retrieval, payload/ID filtering, and Boolean conditions. Predicates can restrict candidates by repository, snapshot, tenant, and evidence type [4] [5]. | Apache-2.0. Adds a service, ingestion/synchronization, authentication, monitoring, and recovery. RAM/disk, replicas, networking, and embeddings are cost drivers. An easy container startup is not a secure production deployment. Filters evaluate supplied conditions; they do not discover Git state. |
| **Redis exact/semantic response caching** | Exact keys with expiration; RedisVL semantic lookup, configurable distance thresholds, filterable fields, metadata, deletion, and TTL. Entry IDs include the prompt and supplied filters [6] [7]. | Attractive if Redis already exists; otherwise adds a service and memory/index capacity. Semantic lookup adds embedding overhead. RedisVL can refresh TTL on hits, so obsolete popular entries need a separate freshness gate. RedisVL is MIT; Redis 8+ licensing is AGPLv3 OR RSALv2 OR SSPLv1 [7] [8]. |
| **GPTCache** | Embedding -> candidate search -> similarity evaluation -> cached-answer/miss pipeline. Separates scalar/vector storage and provides eviction/session hooks. A hit returns a stored answer; a miss invokes the LLM handler [9]. | MIT. Adds Python/framework, model-adapter, embedding, and storage compatibility work. The date-bounded review returned a latest default-branch commit from July 11, 2025 and a most recent GitHub release from August 1, 2024. This is a maintenance caution, not proof of abandonment. |

The reviewed RedisVL repository had a September 10, 2026 release. Within this comparison, that makes RedisVL a better-supported response-cache framework candidate than assuming GPTCache is actively maintained. It does not establish either framework's fitness for the complete AgentCacheX problem.

## What remains custom

### Exact identifiers

Maintain case-preserving identifier/path keys and exact lookups separately from lexical or semantic ranking. Quoted FTS phrases still pass through tokenization; zero vector distance is not proof of string equality.

### Repository/ref isolation

Distinguish repository identity, immutable snapshot identity, authorization scope, and working-tree state. A branch name alone is insufficient. Include staged, dirty, and relevant untracked inputs.

Apply scope predicates before selecting candidates, then check authoritative manifests. The backend supplies filtering mechanisms; the application supplies correct scope values.

### Dependency invalidation

Attach manifests containing file/content hashes, applicable configuration/tool versions, and external-input versions.

Search results depend on the **searched universe**. A newly added matching file can invalidate a result even when every previously returned file is unchanged. Conservative whole-scope invalidation is preferable to incomplete precision.

### Bounded cited context

Store immutable content identity, path, snapshot, source range, and producing tool/arguments. Resolve authoritative bytes, verify ranges, deduplicate, and fit an explicit token budget.

A top-k result limit is not a token limit. Similarity scores are not source citations.

For an external Qdrant index, use versioned ingestion plus an authoritative published-generation marker. Reject stale or incomplete generations instead of equating retrieval success with current evidence.

## Safe semantic matching

Use semantics to discover candidate evidence, not to certify interchangeable answers. It can help with a question such as "where is retry handling?" after strict scope filtering.

Similarity alone is unsafe for choosing between related APIs, branches, dependencies, configurations, or requested mutations.

Recommended sequence:

```text
Scoped exact candidate
  -> lexical retrieval
  -> optional semantic recall
  -> source/provenance and coverage gate
  -> bounded context assembly
```

Semantic answer replay, if ever added, should be narrowly allowlisted and bound to the request contract and source dependencies. Model/prompt versions and relevant generation settings belong in response-cache identity; similarity does not capture conversation or tool state.

## Savings and measurements

- SQLite/Qdrant evidence reuse can avoid repeated scanning and tool work. It does not automatically remove model calls or reduce billed context.
- Redis/GPTCache response reuse can avoid generation on accepted hits. Misses incur lookup, possibly embedding/evaluation, then the original model call. An LLM verifier can itself consume model work.
- Provider KV reuse reduces eligible prefix-processing costs, not the need to produce the answer. Eligibility and pricing are model/provider-specific.

Measure accepted-hit rate after source gating, stale/incorrect reuse, avoided executions, actual input/output tokens, latency, and embedding/indexing costs.

No measured savings or break-even workload was established by this research.

## Primary sources

[1]: https://sqlite.org/fts5.html
[2]: https://github.com/asg017/sqlite-vec/blob/04d28bd21773981e2d266bbf6aa4efbd011eb4f6/site/features/vec0.md
[3]: https://sqlite.org/copyright.html
[4]: https://github.com/qdrant/qdrant/blob/6ab21cac18ebb6f4ae29102c7f8f5cc11affd5de/README.md
[5]: https://qdrant.tech/documentation/search/filtering/
[6]: https://redis.io/docs/latest/commands/set/
[7]: https://github.com/redis/redis-vl-python/blob/ff12ec2fbdc1e23ca0e413467e4953516f7ab0c9/redisvl/extensions/cache/llm/semantic.py
[8]: https://redis.io/legal/licenses/
[9]: https://github.com/zilliztech/GPTCache/blob/c59fb3a6152a4458b2a070ca183b61c4b614095f/gptcache/adapter/adapter.py
[10]: https://developers.openai.com/api/docs/guides/prompt-caching

Additional implementation and licensing anchors:

- [sqlite-vec pre-v1 notice](https://github.com/asg017/sqlite-vec/blob/04d28bd21773981e2d266bbf6aa4efbd011eb4f6/README.md#L10-L18), [MIT license](https://github.com/asg017/sqlite-vec/blob/main/LICENSE-MIT), [Apache license](https://github.com/asg017/sqlite-vec/blob/main/LICENSE-APACHE).
- [Qdrant filtering implementation](https://github.com/qdrant/qdrant/blob/6ab21cac18ebb6f4ae29102c7f8f5cc11affd5de/lib/segment/src/payload_storage/query_checker.rs#L46-L106), [September 4, 2026 release](https://github.com/qdrant/qdrant/releases/tag/v1.19.1).
- [RedisVL MIT license](https://github.com/redis/redis-vl-python/blob/main/LICENSE), [September 10, 2026 release](https://github.com/redis/redis-vl-python/releases/tag/v0.27.2).
- [GPTCache data manager](https://github.com/zilliztech/GPTCache/blob/c59fb3a6152a4458b2a070ca183b61c4b614095f/gptcache/manager/data_manager.py), [MIT license](https://github.com/zilliztech/GPTCache/blob/main/LICENSE), [observed commit](https://github.com/zilliztech/GPTCache/commit/c59fb3a6152a4458b2a070ca183b61c4b614095f), [release 0.1.44](https://github.com/zilliztech/GPTCache/releases/tag/0.1.44).
- [SQLite WAL](https://sqlite.org/wal.html): same-host concurrency, a single writer, and checkpointing; do not share the live database across machines or through cloud sync.

## Uncertainties

Source-pinned development behavior may differ from a packaged release. Pin and exercise the chosen artifacts, particularly pre-v1 sqlite-vec. This research does not establish managed-service cost, production capacity, or runtime compatibility.
