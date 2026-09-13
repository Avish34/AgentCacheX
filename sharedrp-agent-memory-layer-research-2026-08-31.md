# SharedRP Agent Memory Layer Research

> **Historical input:** This August 31 research describes the earlier memory/evidence direction. AgentCacheX's current MVP is a shared tool-result cache plus token-efficient responses, without an owned semantic-analysis or evidence-store platform. See the [current HLD](AgentCacheX-High-Level-Design.md). The original research body below is preserved.

> **Date:** 2026-08-31  
> **Scope:** Open-source memory, retrieval, caching, provenance, and MCP integration patterns for the SharedRP agent  
> **Status:** Research and architecture recommendation; no implementation included

## Executive summary

The SharedRP agent needs more than conversational memory. It needs a Git-aware result cache and
provenance-first evidence store that can:

- Reuse prior code searches and source analysis.
- Avoid loading large KB documents for every request.
- Validate stored facts against the current repository state.
- Invalidate only evidence affected by source changes.
- Compile a small, task-specific context pack for each subagent.
- Preserve source paths, commits, file hashes, and validation history.

No investigated open-source project provides this complete capability as a turnkey product. The
recommended approach is therefore:

> Adopt open-source storage, indexing, ingestion, and MCP components, but build the
> SharedRP-specific provenance, freshness, invalidation, and prompt-compilation control plane.

For a local-first proof of concept:

1. Use SQLite as the authoritative store.
2. Use SQLite FTS5 for exact lexical retrieval.
3. Add `sqlite-vec` only if semantic retrieval proves necessary.
4. Expose the memory layer to GitHub Copilot through a small MCP server.
5. Store atomic evidence cards rather than complete conversations or generated answers.
6. Validate branch-qualified facts through commit and source-file hashes.
7. Return token-bounded context packs to the SharedRP agent.

LlamaIndex offers reusable ingestion and transformation-caching components. Qdrant is the preferred
scale-out retrieval backend. Graphiti is valuable only if temporal relationships, contradictions,
and supersession justify its graph database and LLM extraction overhead.

## Problem statement

The existing `migration/kb` directory already provides durable, Git-reviewed SharedRP knowledge:

- Shared implementation patterns.
- Partner-specific analyses and plans.
- Architect, Refactorer, and Validator learnings.
- Completed migration history.
- Invariants and reference implementations.

The remaining problem is retrieval efficiency. Agents may repeatedly:

- Search for the same controllers, handlers, states, and configuration.
- Load large Markdown files when only one section is relevant.
- Repeat legacy-versus-SharedRP contract analysis.
- Reconstruct dependency-injection and routing information.
- Send previous raw search results back into the model.

The memory layer should reduce this repeated work without allowing stale information to become
current architectural truth.

## Memory pattern taxonomy

The following patterns are complementary.

### 1. Exact tool-result cache

Caches normalized results from:

- Code search.
- File reads.
- Symbol and reference queries.
- Commit history.
- Pull request analysis.
- Configuration discovery.

The cache identity should include the provider, repository, branch or commit, tool version,
canonical arguments, index version, authorization scope, and schema version.

This layer directly reduces repeated ADO, GitHub, and local source queries.

### 2. Provenance-first evidence store

Stores small, independently reusable facts rather than chat transcripts.

Each evidence item should link to:

- Repository and immutable repository ID.
- Branch and resolved commit.
- Source path and line or symbol range.
- Blob SHA or file-content hash.
- Originating query.
- Extractor and prompt version.
- Confidence and validation state.

Generated summaries remain derived records and never replace their source evidence.

### 3. Hybrid retrieval index

Use both:

- Lexical/BM25 retrieval for symbols, paths, API versions, operation names, IDs, and errors.
- Embedding retrieval for conceptual similarity.

Apply metadata filtering before semantic ranking so evidence from the wrong partner, branch, API
version, or resource type cannot outrank the correct exact match.

### 4. Temporal or graph memory

Models:

- Relationships.
- Contradictions.
- Supersession.
- Validity periods.
- Historical architectural decisions.

This is useful for questions such as:

- What replaced this validation pattern?
- Which partner handlers depend on this common base?
- What was true before a migration?

Graph memory supplements source validation; it does not replace it.

### 5. Execution persistence

Stores:

- Workflow checkpoints.
- Pending operations.
- Approval state.
- Subagent progress.
- Retry state.

Execution persistence allows work to resume, but it does not prove that a stored source fact remains
valid.

### 6. Freshness and invalidation plane

Resolves current branch heads and compares stored dependencies with current blobs.

It should:

- Preserve immutable historical evidence.
- Mark branch-qualified evidence stale when dependencies change.
- Promote unaffected evidence when a branch advances but its source blobs do not change.
- Tombstone evidence from deleted files.
- Revalidate negative search results after relevant tree changes.

### 7. Context compiler

Retrieves, validates, ranks, deduplicates, and emits evidence under a strict token budget.

Lossy compression, if used at all, belongs after deterministic evidence selection and should apply
only to narrative prose.

## Candidate comparison

| Candidate | License and deployment | MCP integration | Provenance and freshness | SharedRP assessment |
|---|---|---|---|---|
| **SQLite FTS5 + custom schema** | SQLite is public domain; local embedded database | Requires a thin MCP server | Can enforce commits, blobs, paths, dependencies, tombstones, and validation states | **Best authoritative foundation** |
| **sqlite-vec** | MIT/Apache-2.0; embedded SQLite extension | Used behind the custom MCP service | Metadata can remain relational; vector index is rebuildable | Good optional semantic layer; pre-v1 |
| **LlamaIndex** | MIT; Python framework | Supports exposing workflows/functions through MCP | Document hashes, source relationships, ingestion and transformation caching | **Best reusable ingestion component** |
| **Graphiti** | Apache-2.0; FalkorDB or Neo4j plus LLM/embedder | Built-in experimental MCP server | Strong temporal facts, episodes, supersession, and history; Git semantics remain custom | Strong optional temporal sidecar |
| **Qdrant** | Apache-2.0; server, cloud, or embedded options | Thin REST/gRPC MCP adapter | Rich metadata filters but no Git freshness policy | **Best scale-out retrieval backend** |
| **Semantic Kernel** | MIT; strong .NET integration | MCP server samples and typed vector-store abstractions | Provenance can be modeled as fields; commit validation remains custom | Good if .NET alignment is a priority |
| **LangGraph Store** | MIT; workflow checkpointers and long-term store | Thin MCP adapter required | Useful timestamps and namespaces; no Git freshness semantics | Useful if SharedRP orchestration becomes LangGraph-based |
| **mcp-memory-service** | Apache-2.0; MCP/REST with multiple backends | MCP-native | Supports hashes and metadata, but not mandatory Git lineage | Good disposable prototype |
| **Mem0 OSS** | Apache-2.0; fact extraction and vector memory | OSS requires a wrapper; documented MCP is hosted | Metadata can hold sources; updates and contradictions need explicit handling | Better for preferences and decisions than code truth |
| **Letta Code / MemFS** | Apache-2.0; full stateful agent harness | Broader runtime adoption required | Git-versioned agent memory, but code-source lineage remains custom | Low fit unless replacing the agent runtime |
| **Basic Memory** | AGPL-3.0; local Markdown knowledge graph | MCP-native | Human-auditable but not inherently tied to source blobs | Useful for decisions and handoffs; legal review required |
| **Official MCP Memory server** | Reference implementation with transitional repository licensing | MCP-native | Basic JSONL entity/relation graph with substring matching | Educational reference only |

## Ranked shortlist

### 1. SQLite FTS5 with a custom provenance schema

This is the preferred foundation.

Reasons:

- SharedRP questions frequently contain exact identifiers and paths.
- FTS5 supports phrase, prefix, Boolean, proximity, and BM25 retrieval.
- SQLite provides transactions across cache entries, evidence, dependencies, and usage metrics.
- Source provenance can be mandatory instead of optional metadata.
- Local stdio deployment minimizes operational cost and source-code exposure.
- A later Qdrant index can be rebuilt from the authoritative relational data.

Start with FTS5 alone. Introduce vector search only after measuring lexical misses.

References:

- [SQLite FTS5 documentation](https://www.sqlite.org/fts5.html)
- [SQLite copyright and public-domain status](https://www.sqlite.org/copyright.html)

### 2. LlamaIndex ingestion and caching

Use LlamaIndex as an ingestion component, not as the source of truth.

Useful capabilities:

- Document chunking.
- Transformation pipelines.
- Node and transformation caching.
- Document hash comparison.
- Source relationships.
- Embedding and reranking integration.
- MCP exposure for workflows and functions.

SharedRP still needs custom interpretation of repository, branch, commit, and source validity.

References:

- [LlamaIndex ingestion pipeline](https://github.com/run-llama/llama_index/blob/main/docs/src/content/docs/framework/module_guides/loading/ingestion_pipeline/index.md)
- [LlamaIndex MCP conversion](https://github.com/run-llama/llama_index/blob/main/docs/src/content/docs/framework/module_guides/mcp/convert_existing.md)
- [LlamaIndex node and source schema](https://github.com/run-llama/llama_index/blob/main/llama-index-core/llama_index/core/schema.py)

### 3. Graphiti temporal graph

Use Graphiti only when the agent needs relationship-heavy or temporal questions:

- Historical versus current behavior.
- Pattern supersession.
- Cross-partner dependencies.
- Contradictory findings.
- Contract evolution.

Graphiti should store high-level relationships linked to authoritative evidence IDs. It should not
be the primary code-result cache.

References:

- [Graphiti repository](https://github.com/getzep/graphiti)
- [Graphiti MCP server](https://github.com/getzep/graphiti/tree/main/mcp_server)

### 4. Qdrant

Adopt Qdrant when:

- Multiple agents need concurrent shared access.
- The number of chunks or embeddings exceeds comfortable embedded operation.
- Sparse+dense fusion, sharding, replication, or operational observability becomes necessary.

Keep relational provenance authoritative and treat Qdrant as a rebuildable retrieval index.

Reference:

- [Qdrant repository](https://github.com/qdrant/qdrant)

### 5. Semantic Kernel

Semantic Kernel is a reasonable alternative when implementing the MCP service in .NET.

Benefits:

- Typed vector records.
- Collection abstractions.
- Metadata filtering.
- Strong fit with the existing .NET environment.
- MCP client/server examples.

Limitations:

- Vector-store APIs have preview or release-candidate surfaces in some languages.
- It does not provide the required Git freshness and exact query-result cache by itself.

References:

- [Semantic Kernel vector stores](https://learn.microsoft.com/semantic-kernel/concepts/vector-store-connectors/)
- [Semantic Kernel agent memory design](https://github.com/microsoft/semantic-kernel/blob/main/docs/decisions/0072-agents-with-memory.md)
- [Semantic Kernel MCP server sample](https://github.com/microsoft/semantic-kernel/tree/main/dotnet/samples/Demos/ModelContextProtocolClientServer)

### 6. mcp-memory-service

This is the fastest Apache-licensed option for a disposable proof of concept.

It already offers MCP/REST access, local embeddings, metadata, content hashes, and multiple storage
backends. SharedRP-specific cache keys, source lineage, and invalidation would still need to be
added or wrapped.

Reference:

- [mcp-memory-service](https://github.com/doobidoo/mcp-memory-service)

## Recommended architecture

```text
GitHub Copilot CLI custom SharedRP agent
                  |
             MCP stdio/HTTP
                  |
     SharedRP Memory/Cache Adapter
     +-- Exact query-result cache
     +-- Git/ADO freshness validator
     +-- Provenance/evidence service
     +-- Hybrid retriever and reranker
     +-- Context/evidence-card compiler
     +-- Audit, metrics, and invalidation
                  |
     +------------+-------------------+
     |            |                   |
 SQLite tables   FTS5 / sqlite-vec   Optional sidecars
 authoritative   rebuildable index   +-- LlamaIndex ingestion
                                     +-- Qdrant at scale
                                     +-- Graphiti temporal graph
```

### Adopt

- MCP SDK, FastMCP, or Semantic Kernel MCP transport.
- SQLite for authoritative local data.
- FTS5 for lexical retrieval.
- `sqlite-vec` for optional semantic recall.
- LlamaIndex if ingestion caching justifies its dependency footprint.
- Qdrant for shared deployment at scale.
- Graphiti only for temporal relationships.

### Build

- Exact query fingerprinting.
- Git and ADO freshness validation.
- Mandatory provenance schema.
- Selective invalidation.
- Negative-query caching.
- Access-scope isolation.
- Evidence validation states.
- Token-budgeted context compilation.
- Cache and token-usage metrics.

## Proposed data model

### `source_snapshot`

- Provider.
- Repository owner, name, and immutable ID.
- Ref type and name.
- Resolved commit.
- Tree SHA, if available.
- Path.
- Blob SHA or content hash.
- Start and end line.
- Context-anchor hash.
- Fetch and validation timestamps.
- Access classification.

### `query_cache`

- Tool and tool version.
- Canonical arguments.
- Query fingerprint.
- Source scope.
- Observed commit and tree.
- Raw-result hash and storage pointer.
- Negative-result flag.
- Dependency set.
- Creation, validation, and expiry timestamps.

### `evidence`

- Atomic claim or concise excerpt.
- Evidence type.
- Confidence.
- Source IDs.
- Extractor, prompt, and model versions.
- Current validity state.
- `valid_from`, `valid_to`, and `superseded_by`.

### `dependency`

- Query or evidence ID.
- Source path and stored blob.
- Dependency type:
  - Direct source.
  - Derived summary.
  - Imported contract.
  - Generated artifact.
  - Negative-search scope.

### `usage`

- Cache hit or miss.
- Retrieval rank.
- Last access.
- Raw-result token count.
- Emitted-evidence token count.
- Accepted, rejected, or revalidated result.

## Cache identity

An exact query-cache identity should include:

```text
provider
+ immutable repository ID
+ ref type and name
+ resolved commit SHA
+ tool name and version
+ canonical JSON arguments
+ search/index version
+ authorization scope
+ schema version
```

Keep the query fingerprint separate from source validation. This allows reuse after a branch moves
when all relevant source blobs remain unchanged.

## Freshness model

Every lookup should return an explicit freshness state.

| State | Meaning |
|---|---|
| `fresh` | Current ref was resolved and all dependencies match |
| `historical` | Valid for an immutable commit but not asserted as current |
| `suspect` | Branch moved or validation could not complete |
| `stale` | A dependency changed, disappeared, or could not be re-anchored |
| `tombstoned` | Removed from current truth but retained for audit |

Rules:

1. Commit-qualified facts remain valid for historical queries.
2. Branch-qualified facts must resolve the branch before being presented as current.
3. When a branch advances, compare only relevant blob hashes.
4. Promote unaffected evidence when dependencies have not changed.
5. Invalidate deleted sources.
6. Re-anchor renamed files only after content verification.
7. Invalidate negative results on any relevant tree change.
8. Treat TTL as a fallback, not proof of correctness.
9. If repository validation fails, return `suspect`, never `fresh`.
10. Invalidate derived summaries when evidence or summarization versions change.

## Retrieval and context compilation

Recommended retrieval order:

1. Filter by repository, ref, freshness, path, evidence type, and authorization scope.
2. Run exact identifier, path, API-version, and error matching through FTS5.
3. Run semantic retrieval only if exact retrieval is insufficient.
4. Fuse rankings using reciprocal-rank fusion or a simple weighted score.
5. Rerank by:
   - Freshness.
   - Exact symbol or path match.
   - Partner, resource type, operation, and API-version match.
   - Semantic relevance.
   - Source diversity.
6. Deduplicate evidence from the same source span.
7. Render evidence cards under a strict token budget.

Example:

```text
Fact: Validation failures must populate ErrorResponse.
Source: Liftr.Services:SharedRPWebPartnerValidation.cs:ValidateCoreAsync@abc123
Freshness: fresh; blob 45ef67; validated 2026-08-31
Why selected: exact resource contract and operation match
Excerpt: minimum source lines needed to establish the claim
```

## GitHub Copilot MCP proof of concept

GitHub Copilot CLI supports custom MCP servers. A local stdio service is the simplest initial
deployment.

Official reference:

- [Adding MCP servers to GitHub Copilot CLI](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-mcp-servers)

### Minimal tools

#### `memory_lookup`

Inputs:

- Query.
- Repository and ref scope.
- Path filters.
- Current-head hint.
- Result count.
- Token budget.

Output:

- Validated evidence cards.
- Freshness.
- Source lineage.
- Missing evidence.

#### `code_search_cached`

Checks the exact cache first. On a miss, it invokes the underlying ADO or GitHub search and records
the structured result.

Proxying common searches ensures capture does not depend on the model remembering to save results.

#### `memory_record_evidence`

Stores a sourced fact or excerpt. Repository, commit, path, and source span are required.

Unsourced writes should be rejected or marked `unverified` and excluded from authoritative answers.

#### `memory_validate`

Re-resolves repository refs and validates specified query or evidence IDs.

#### `memory_invalidate`

Accepts repository/ref and changed commits or paths, then revalidates dependent records.

#### `memory_explain`

Returns lineage, derivation versions, source hashes, and validation history.

#### `memory_forget`

Deletes or tombstones a repository, branch, query, evidence item, or access scope.

### Optional MCP resources

```text
memory://evidence/{id}
memory://repository/{owner}/{repo}/ref/{ref}/status
memory://query/{fingerprint}
```

## SharedRP agent policy

The SharedRP agent should be instructed to:

1. Call `memory_lookup` before broad code search.
2. Use only `fresh` evidence for current-state assertions.
3. Label `historical` evidence explicitly.
4. Treat `suspect` and `stale` results as cache misses.
5. Prefer `code_search_cached` for supported query types.
6. Include source evidence cards rather than unsourced memory text.
7. Never store secrets, credentials, customer data, or unsupported speculation.
8. Pass only a token-bounded context pack to specialized subagents.
9. Promote reusable facts only after Validator approval.

Initially, unsupported tools can use:

```text
normal search -> memory_record_evidence
```

High-volume searches should later move behind proxy tools.

## Proof-of-concept validation scenarios

Test the following:

1. First request misses; identical second request hits.
2. Semantically equivalent requests retrieve the same evidence after scope validation.
3. Unrelated source changes preserve cache hits.
4. A source-file edit marks dependent facts stale.
5. A deleted file tombstones its evidence.
6. A rename re-anchors only after content verification.
7. Branch switching does not leak evidence.
8. Historical commit evidence remains queryable.
9. A cached negative search is invalidated after a matching file is added.
10. Repository validation failure returns `suspect`.
11. Cross-repository and cross-user access controls prevent leakage.

Measure:

- Exact cache-hit rate.
- Semantic cache-hit rate.
- Underlying source-search calls per task.
- Prompt tokens before and after context compilation.
- Raw-result bytes versus emitted evidence bytes.
- Freshness-validation latency.
- Stale hits used for current assertions; target zero.
- Citation resolution rate.
- Answer correctness.

## Context compression

Use this order:

1. Cache raw tool results outside model context.
2. Validate source freshness.
3. Filter by repository, branch, partner, RT, operation, and evidence type.
4. Retrieve exact lexical hits.
5. Add semantic recall only when needed.
6. Fuse, rerank, and deduplicate.
7. Emit compact evidence cards.
8. Optionally compress remaining narrative prose.

Do not cache complete model answers as authoritative truth. Cache their evidence sets and regenerate
answers against fresh evidence.

### LLMLingua

[LLMLingua](https://github.com/microsoft/LLMLingua) is MIT licensed and supports prompt compression,
but it is lossy.

Never compress:

- Repository names.
- Source paths.
- Commits and hashes.
- Line ranges.
- Code or symbols.
- Diagnostics.
- API versions.
- JSON.
- Commands.
- Quoted evidence.

If used, restrict compression to explanatory prose and verify protected spans byte-for-byte.
Deterministic evidence selection is safer and likely provides most of the available token savings.

## GraphRAG assessment

[Microsoft GraphRAG](https://github.com/microsoft/graphrag) can help with offline architecture
discovery across a stable corpus:

- Entity extraction.
- Relationship discovery.
- Communities.
- Community summaries.

It is not recommended as the interactive query-result cache. The project describes itself as
research-oriented, indexing can be expensive, and its cache package is maintenance-oriented.

Potential use:

- Periodically generate architecture summaries for immutable release snapshots.
- Store summaries as derived artifacts tied to source commits.
- Never use graph summaries as proof of current source behavior.

## Licensing and project-status cautions

### Zep and Graphiti

- The current Zep repository contains cloud examples and integrations, not the production service.
- Zep Community Edition is deprecated and unsupported.
- The production Zep Context Graph Engine is proprietary.
- Graphiti is the relevant Apache-2.0 open-source framework.

References:

- [Zep repository](https://github.com/getzep/zep)
- [Graphiti repository](https://github.com/getzep/graphiti)

### Letta

- The older Letta/MemGPT implementation is retired.
- Evaluate [Letta Code](https://github.com/letta-ai/letta-code), not old deployment guides.
- Letta is a complete stateful-agent harness rather than a focused code-search cache.

### Mem0

- The documented Mem0 MCP service is hosted.
- OSS Mem0 requires a custom MCP adapter.
- Automatic fact extraction does not solve Git freshness or contradiction handling.

Reference:

- [Mem0 repository](https://github.com/mem0ai/mem0)

### Basic Memory

- [Basic Memory](https://github.com/basicmachines-co/basic-memory) is AGPL-3.0.
- Obtain legal guidance before modifying it or exposing it as an internal network service.

### Official MCP servers

- The [official MCP servers repository](https://github.com/modelcontextprotocol/servers) describes
  its servers as educational reference implementations.
- Its memory server uses a JSONL entity/relation graph and basic substring search.
- It has no native Git provenance, embeddings, TTL, or selective invalidation.

### sqlite-vec

- [`sqlite-vec`](https://github.com/asg017/sqlite-vec) is pre-v1.
- Pin the version.
- Place it behind an index abstraction.
- Keep vectors rebuildable.
- Preserve an FTS5-only fallback.

### Semantic Kernel

- Vector-store APIs are not uniformly stable across languages.
- Confirm current stability before selecting a production connector.

### Hosted versus open-source products

Do not assume features from the following hosted products exist in their open-source cores:

- Mem0 Platform.
- Zep.
- Chroma Cloud.
- Qdrant Cloud.
- LanceDB Enterprise.
- Basic Memory Cloud.

Licenses listed in this report are not legal advice. Review direct and transitive dependencies,
container images, model licenses, embedding services, and cloud terms before production adoption.

## Recommended implementation phases

### Phase 1: local lexical proof of concept

- SQLite authoritative schema.
- FTS5 retrieval.
- Local stdio MCP server.
- Exact query cache.
- Required source provenance.
- Branch and blob freshness validation.
- Evidence-card compiler.
- Metrics for cache hits and token savings.

### Phase 2: agent integration

- Add memory tools to `SharedRP-Agent`.
- Insert the memory gate after request classification.
- Prefer cached searches.
- Pass compact context packs to Architect, Refactorer, and Validator.
- Store only Validator-approved reusable evidence.

### Phase 3: semantic retrieval

- Measure lexical misses.
- Add `sqlite-vec` or another vector backend only where semantic retrieval improves recall.
- Store embedding model and version with every vector.
- Keep relational evidence authoritative.

### Phase 4: shared service

- Move the retrieval index to Qdrant if multi-agent concurrency or corpus size requires it.
- Add access controls and repository-scope isolation.
- Use remote HTTP MCP only after authentication, authorization, observability, backup, and recovery
  requirements are defined.

### Phase 5: temporal graph, if justified

- Add Graphiti only for relationship-heavy and temporal queries.
- Link every graph fact to authoritative evidence IDs.
- Keep current-source assertions behind Git validation.

## Final recommendation

Build a small provenance-first SharedRP MCP service:

- **Authoritative layer:** SQLite relational schema.
- **Exact retrieval:** FTS5.
- **Semantic retrieval:** optional `sqlite-vec` initially; Qdrant later.
- **Ingestion:** selectively reuse LlamaIndex.
- **Temporal graph:** optional Graphiti sidecar.
- **Prompt reduction:** deterministic evidence cards.
- **Freshness:** custom Git/ADO commit, blob, dependency, and negative-cache validation.
- **Transport:** local stdio MCP first.

This design adopts mature ecosystem components while keeping SharedRP-critical correctness rules
explicit, testable, and independent of any single memory framework.
