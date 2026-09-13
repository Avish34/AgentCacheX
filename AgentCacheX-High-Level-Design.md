# AgentCacheX: High-Level Design

**Scope revision:** 2026-09-13 - shared tool-result cache plus token-efficient progressive disclosure  
**Status:** Proposed hackathon architecture; no cache service or measured product results yet  
**Audience:** Teammates implementing one focused MVP

## 1. Problem and recommendation

Multiple developers working on the same repository repeat the same read-only code lookups and send oversized, repetitive results to their agents. AgentCacheX shares version-scoped tool results and returns only the details each agent needs.

Build a **shared MCP service with Redis-backed exact tool-result caching and a deterministic token optimizer**. Start with one repository and one existing search provider. Keep full captured results outside model context; expose compact discovery and batched detail retrieval.

Two independent benefits must be measured: fewer duplicate backend executions, and less unnecessary model-visible tool output. A warm cache returning the same enormous payload solves only the first. Progressive disclosure can help even on the first cache miss.

### What changed from the earlier design?

| Earlier direction | Current MVP |
|---|---|
| Cache learned evidence cards and semantic claims | Cache exact captured tool results and source ranges |
| Build language-specific analyzers | Reuse an existing provider; do not implement semantic analysis |
| SQLite/FTS5 authority with optional vector retrieval | Shared Redis result cache; no new search index |
| Local-first cache, team sharing later | Multiple developers use the same service from the start |
| Dependency-aware selective invalidation | Coarse, immutable snapshot-qualified keys |
| Broad acquisition/record/lookup API | `search_compact` and `fetch_details` |

The historical research is retained and labeled in the [README](README.md). It explains alternatives, not additional MVP dependencies.

### Why this option?

| Alternative | Assessment |
|---|---|
| Use existing tool limits, file ranges, and provider caching | Essential baseline; may already solve enough of the problem |
| Shared exact cache only | Simple backend savings, but no inherent reduction in tool payload |
| **Shared exact cache + progressive disclosure** | **Selected:** addresses both costs without another analysis engine |
| Provider prompt caching | Complementary pricing/processing optimization; does not itself deduplicate repository tool executions |
| Semantic answer cache | Similar questions can require different code states or answers; not needed here |
| New RAG, evidence, or graph-memory platform | Larger ingestion, correctness, and operations scope than this problem requires |

Ship the wrapper only if it provides incremental value over realistic native tools. Do not deliberately disable their filters or pagination to manufacture a favorable baseline.

## 2. Scope and boundaries

| Included | Excluded |
|---|---|
| One configured repository; multiple authenticated developers | Cross-repository memory or arbitrary filesystem access |
| One existing read-only provider | Building a code search engine, language server, or compiler integration |
| Shared immutable tool results | Caching final answers, edits, shell trajectories, or arbitrary client-written results |
| Compact previews and selected exact details | LLM-generated summaries or source-code rewriting |
| Explicit snapshots, access scopes, expiry, and pagination | Shared dirty/unsaved workspace state |
| Metrics and a reproducible team demo | Highly available multi-region infrastructure |

The cache is **programming-language-agnostic because its boundary is the provider result**, not a programming-language AST. An upstream provider might use Roslyn for C#, another language server, or a semantic index. None becomes an AgentCacheX core dependency.

The recommended pilot wraps existing **`git grep` at a resolved commit** in a trusted repository mirror. This supports text search across languages and makes source identity observable. It does **not** establish semantic references, implementations, or call relationships. Requests requiring those capabilities must use a capable provider or return `UNSUPPORTED_CAPABILITY`.

Git search is often already fast. This pilot establishes correctness and token behavior, not a promise of large latency savings. A subsequent expensive remote provider is eligible only when its actual searched source/index version and access semantics can be established.

## 3. Components and deployment

```text
Developer A      Developer B      Developer C
     \               |               /
       Configured AgentCacheX MCP tools
                     |
         Shared authenticated MCP service
                     |
       Repository access + snapshot resolver
                     |
          Exact-result lookup / coordination
             /                       \
      Shared Redis             Existing provider
   full captured results       search/read on miss
             \                       /
            Deterministic token optimizer
                     |
          Preview / selected exact details
                     |
          Existing agent reasons and answers
```

The [interactive diagram](AgentCacheX-Architecture.html) provides component, token-flow, and lifecycle views.

| Component | Responsibility and reason |
|---|---|
| MCP host | Publish the two tools over an authenticated remote endpoint. Proposed stack: ASP.NET Core and the official MCP C# SDK [3][4]. The host language is not an input-language restriction. |
| Access/snapshot gate | Resolve a configured repository, enforce current permissions, and establish the source the backend actually searches. It is the authority for cache eligibility. |
| Provider adapter | Invoke one existing read-only tool with canonical arguments; retain original results, locations, capability limits, and completeness metadata. |
| Redis | Hold result payloads, query pointers, and expiring coordination leases shared across users. It is a disposable cache, not the repository's source of truth. |
| Token optimizer | Build bounded previews and detail batches without inventing summaries, dropping hidden matches, or changing source text. |
| Metrics | Separate cache effects from output-shaping effects and account for the entire agent task. |

Deploy **one service and one Redis instance** for the hackathon. Keep Redis private to the service, with credentials, transport protection, and bounded memory. Repository mirrors and runtime data belong outside the synced project folder and outside Git.

This is a shared network cache, not a claim of fault-tolerant distributed operation. Additional stateless service replicas and managed Redis availability are later deployment choices. Never use a OneDrive/network-shared SQLite file as the coordination mechanism.

## 4. Snapshot and cache identity

The conceptual key is a hash of a versioned, canonical record:

```text
repository immutable ID
+ actual source snapshot and applicable provider index revision
+ provider identity, implementation version, and tool operation
+ canonical upstream arguments
+ effective authorization/content-policy scope and policy version
+ stored-result schema version
```

Canonicalization normalizes the structured request, not its meaning. Sort object keys and resolve documented defaults; do not remove meaningful query whitespace or change a regular expression. Include path filters, context ranges, upstream limits, and continuation parameters that affect the captured result.

**Exclude presentation-only settings** such as preview size and response budget from the raw-result key. Developer B can reuse Developer A's underlying result while asking for a smaller preview. There is no automatic username partition: callers with the same effective visibility may share; user-specific or more restricted results stay isolated.

The service determines scope after authentication. The model cannot claim another user's scope, invent an access group, write cache entries, or establish correctness with `clean=true`.

### Source-state rules

| Requested state | Behavior |
|---|---|
| Explicit committed revision, searched by the provider at that revision | Eligible for shared caching; label the actual snapshot |
| Branch/ref | Resolve to an immutable revision first and search that revision, not a moving ref |
| Different commit | Different key; deliberately coarse invalidation |
| Historical committed revision | Valid only as that named historical snapshot |
| Dirty or unsaved workspace | Bypass shared reuse; use a matching uncached provider if available, otherwise return `UNSUPPORTED_SOURCE_STATE` |
| Unknown actual backend/index revision | Do not publish or reuse a shared entry; label a permitted uncached result as unverifiable |
| Provider searches an older index than requested | Report the mismatch/lag; do not label its result as the requested revision |

A commit that changes an unrelated file also creates a new key. This deliberately sacrifices some reuse to keep source correctness simple; cross-revision dependency proofs are outside the MVP.

A clean flag from an agent is not workspace attestation. A request for "my current workspace" requires trustworthy host/workspace integration; absent that, require an explicit committed-snapshot request or report the limitation. Results at a commit do not assert that the developer's current checkout is unchanged.

For the Git pilot, resolve and verify the commit object first, then pass that exact object ID to the existing tool. Illustrative invocation after resolution:

```text
git grep -n -I -F -e "VIP_DISCOUNT" <resolved-commit> --
```

Git supports searching supplied tree objects [5]. The adapter must use an argument-list API, not interpolate requests into shell commands; restrict repositories, operations, and paths to authorized configuration. Captured context can be requested through existing tool options. The sample is text search, not a new semantic analyzer.

## 5. What Redis stores

| Record | Contents |
|---|---|
| Query pointer | Cache-key digest -> opaque result-set ID; expiry no later than its payload |
| Immutable result set | Full captured provider response, normalized item index, snapshot, access-policy identity, provider/version, source locations, completeness/continuation data, creation and expiration |
| Pinned detail result | A separately cached source read when a requested expansion needs text not included in the original search response |
| In-flight lease | Key, unpredictable owner token, and bounded expiry for coalescing simultaneous misses |

**Full captured response does not mean every file or every match in the repository.** If the upstream provider returns one page, store the complete page and its continuation state. Do not relabel it as an exhaustive search.

Assign stable item IDs within each immutable result set. Keep result sets, cursors, and detail reads bound to their repository, snapshot, and authorized visibility. Opaque handles are references, not access credentials.

Publish payload and pointer atomically or in a payload-first sequence that never exposes an incomplete entry. A missing payload behind a pointer is a miss, never an empty result. Store expiry metadata consistently, bound entry size and total memory, and disclose truncation or an uncacheable-size response rather than silently discarding data.

Redis expiry and eviction control retention. They do not prove source freshness. Expired or evicted detail handles return `RESULT_EXPIRED`; clients must repeat the search. Eviction of an acceleration cache must not destroy source data.

### Concurrent misses

Use bounded, owner-checked expiring leases, such as Redis `SET ... NX PX ...` plus an atomic ownership check when renewing or releasing [6]. Recheck the cache after lease acquisition. Publish complete results only under the applicable ownership policy; a former owner must not delete a newer owner's lease.

Waiters wait for a bounded interval, then retry the lookup or receive an explicit retryable condition. Provider timeouts, failed owners, and lease expiration must not block indefinitely. Coalescing reduces duplicate work; it is not a universal exactly-once guarantee under failures.

## 6. Request flows

### Cold search

1. Authenticate; authorize the configured repository and effective content scope.
2. Resolve the actual searchable snapshot and provider capabilities.
3. Canonicalize upstream arguments and compute the raw-result key.
4. Look up Redis. On a miss, acquire/recheck coordination and run the provider.
5. Capture exact output, locations, snapshot, and completeness. Publish a cacheable result set.
6. Apply the response budget and return a compact preview or a complete-result page.

### Another developer's warm search

Developer B submits the same lookup for the same snapshot and equivalent access scope. The service reauthorizes B, finds the result, and runs the optimizer for B's budget. The provider search is not rerun for that retained result.

Only equivalent upstream requests hit. Different wording or a paraphrased natural-language question does not imply a semantic cache hit.

### Selected details

Resolve and authorize the result set; reject expiry or snapshot mismatch. Fetch the selected item ranges together, merge safe overlap, and return exact text with paths and line ranges.

If the search captured only short snippets, a broader expansion requires an explicit read at **the same immutable snapshot**. Cache that read separately and count its backend cost. Never silently fetch from the latest branch, nor imply that retaining search metadata retained whole files. A provider unable to expand the pinned snapshot returns an explicit limitation.

Retain source objects for the lifetime of expandable handles where the provider supports that. If a pinned object has become unavailable, return `SNAPSHOT_UNAVAILABLE`; the continued existence of a Redis search artifact does not make the missing source available.

If the requested batch cannot fit, return the included and pending selections plus a continuation cursor. An item itself may need multiple source-range chunks. Mark partial ranges; do not present a cut-off function as its full definition.

## 7. Proposed MCP contracts

These names and shapes describe the design, not implemented endpoints:

```text
search_compact(query, scope, response_budget, mode="preview", cursor=null)
fetch_details(result_set_id, selections, response_budget, cursor=null)
```

`scope` identifies a requested repository/revision and optional search filters; the service verifies it and derives authorization internally. `mode="complete"` requests explicit traversal of the available match set, not a small relevance sample. Completeness remains limited by the provider's capability.

`response_budget` specifies either a supported tokenizer and token limit or a byte limit. With no matching tokenizer, return labeled estimates under a byte bound or `UNSUPPORTED_TOKENIZER`; do not advertise an exact token guarantee.

### Search example

```text
search_compact(
  query="VIP_DISCOUNT",
  scope={repository_id: "demo-shop", revision: "abc123"},
  response_budget={max_tokens: 600, tokenizer_id: "configured-agent-model"},
  mode="preview"
)
```

Illustrative response excerpt; the commit is abbreviated for readability, and counts are not benchmark results:

```json
{
  "result_set_id": "R17",
  "cache_status": "miss",
  "snapshot": {
    "repository_id": "demo-shop",
    "commit": "abc123",
    "provider_version": "git-grep-adapter-v1"
  },
  "items": [
    {
      "id": "r1",
      "path": "Pricing.cs",
      "start_line": 12,
      "end_line": 12,
      "preview": "const decimal VIP_DISCOUNT = 0.20m;"
    },
    {
      "id": "r2",
      "path": "pricing.ts",
      "start_line": 8,
      "end_line": 8,
      "preview": "const VIP_DISCOUNT = 0.20;"
    }
  ],
  "coverage": {
    "captured_match_count": 40,
    "shown_match_count": 2,
    "provider_complete": true
  },
  "more_available": true,
  "cursor": "opaque-preview-cursor",
  "expires_in_seconds": 600
}
```

The response says **two of forty captured matches**, not "these are the only relevant files." `provider_complete` describes upstream coverage; `more_available` and the cursor describe the remaining response traversal. If the upstream total is unknown, report it as unknown rather than inventing a count. A complete response contract also includes the applied budget, exact count or labeled estimate, and counting/tokenizer identity.

### Detail example

```text
fetch_details(
  result_set_id="R17",
  selections=[{item_id: "r1"}, {item_id: "r2"}],
  response_budget={max_tokens: 1500, tokenizer_id: "configured-agent-model"}
)
```

The result contains source snapshot, exact source ranges, fulfilled/pending selections, continuation state, expiry, and budget usage. Optional expanded ranges must remain attached to stored item identity and the pinned source. Do not accept arbitrary filesystem paths through a detail handle.

Cursors distinguish response pagination from upstream pagination and are bound to the search, snapshot, and access scope. Authenticate and authorize every page. Re-query after expiration; do not retarget an old cursor to a new revision.

## 8. Token optimizer

Use deterministic output shaping, not another model call:

| Technique | Required invariant |
|---|---|
| Compact preview | Keep enough source identity, ranking, and exact preview text to choose the right items |
| Group repeated metadata | Keep repository/snapshot/provider once per response, without losing item-to-source association |
| Deduplicate snippets | Merge only identical or overlapping text from the same source and snapshot; preserve locations |
| Selective expansion | Batch requested details, rather than force one agent turn per match |
| Budgeted packing | Count the final serialized model-visible payload, including envelope and coverage metadata |
| Complete-mode pagination | Make all captured matches discoverable; disclose upstream truncation and pending pages |

Retain the complete original provider response outside context. Do not silently throw away fields that could alter interpretation; expose necessary diagnostics/meaningful fields or make them retrievable. Preserve original code, identifiers, comments, and literals rather than minifying allegedly equivalent code.

The initial optimizer deduplicates **within a response**. It must not assume that a prior excerpt remains in active context after compaction, a new session, or delegation. A handle alone cannot replace evidence the model needs to reason.

If the envelope alone exceeds the budget, return `BUDGET_TOO_SMALL` with the minimum requirement rather than produce misleading empty success. Configure maximum requests, source ranges, entries, and byte sizes. A response budget limits our tool payload, not host-added prompts/tool definitions or cumulative task context.

### Illustrative arithmetic

```text
Full raw tool response:        10,000 tokens
Compact preview:                 300 tokens
Selected details:              1,200 tokens
Total selected tool payload:   1,500 tokens
Illustrative payload reduction:   85%
```

This is not a measured benchmark. If the agent ultimately fetches everything, payload savings may disappear. Added model turns can repeat previous context and increase total input even with smaller individual tool responses. Billed cost also depends on provider cached-input pricing and output tokens.

## 9. Access and failure behavior

Authenticate over the shared endpoint; authorize repository and content access on every search, detail request, and page. Reevaluate current permissions even for an old handle. Where two users cannot see the same content, separate or filter cached results under an effective policy-qualified scope.

Do not place credentials, raw code, full queries, or result payloads in routine metrics. Keep code-bearing Redis storage and logs within the approved environment. Treat repository/tool text as untrusted data, not instructions that can change permissions or invoke new operations.

| Condition | Required behavior |
|---|---|
| Unauthorized repository/result/page | Deny access without leaking payload or unnecessary existence information |
| Dirty/unknown requested source unsupported by backend | `UNSUPPORTED_SOURCE_STATE`; no silent committed fallback |
| Requested and actual snapshots disagree | `SNAPSHOT_MISMATCH` or explicit index-lag condition |
| A pinned source object needed for expansion no longer exists | `SNAPSHOT_UNAVAILABLE`; do not read a newer version |
| Missing/expired detail result | `RESULT_EXPIRED`, requiring a new search |
| Provider error or timeout | Surface the provider failure; never cache it as successful "no matches" |
| Redis unavailable | Explicit cache-unavailable/retry condition; no success-shaped or silently uncached fallback in the initial MVP |
| Partial upstream results | Preserve partial status, diagnostics, and continuation; do not claim exhaustive coverage |
| Too-small budget or unavailable exact tokenizer | `BUDGET_TOO_SMALL` or `UNSUPPORTED_TOKENIZER`, or an explicitly selected byte-bounded estimate mode |
| Oversized provider result | Explicit size/truncation/continuation condition; publish no incomplete success entry |

A successfully completed search with zero matches is a valid result for its exact snapshot and scope. It is distinct from all of the failure cases above.

## 10. Evaluation and demo

Compare three variants using the **same source snapshot, task, provider, model, and native filters**:

| Variant | What it isolates |
|---|---|
| Existing provider with reasonable limits/ranges | Honest baseline |
| Wrapper + token optimizer, shared caching disabled | Output-shaping benefit and extra-turn overhead |
| Wrapper + optimizer + shared cache | Incremental cross-developer backend reuse |

Record backend searches and pinned detail reads, cache hit/miss/bypass/coalescing, raw and emitted payload bytes/tokens, model input and output usage across **all turns**, repeated history, latency distribution, actual provider pricing, and answer quality. Keep raw, estimated, provider-reported, and billed quantities distinguishable.

| Demo case | Expected observable outcome |
|---|---|
| A searches snapshot S1 cold | Existing provider executes; compact response still applies |
| B repeats the same query at S1 with equivalent access | Shared result reused; no repeated provider search |
| B uses a different preview budget | Same raw cache identity; newly packed response |
| A and B issue concurrent identical misses | Controlled run demonstrates coalescing; failure tests exercise lease expiry |
| Source changes to S2 | New key and source-correct results, not an S1 hit |
| Dirty/unverifiable workspace is requested | Visible bypass or unsupported-state result |
| Another access scope or revoked permission requests a handle | No unauthorized disclosure |
| R17 expires or is evicted | Explicit expiration and re-query path |
| Complete-mode query exceeds one response | No hidden dropped matches; traverse explicit pages |
| Selected details exceed budget | Exact partial ranges and pending selections, no silent truncation |

Acceptance requires correct source/access/completeness behavior, faithfully preserved selected text, no material answer-quality regression, and a measured benefit in total task tokens or backend work worth the operational overhead. Do not set an unsupported universal savings percentage.

## 11. Implementation sequence and team split

1. Select the pilot repository, trusted commit-pinned provider, authorization source, and representative questions. Freeze the snapshot/result/coverage/budget contracts.
2. Implement the read-only provider adapter and shared Redis result lifecycle, including actual source identity and permission-qualified keys.
3. Implement both MCP tools and deterministic packing, including pinned expansion, byte/token accounting, and complete-mode pagination.
4. Add bounded concurrent-miss coordination, expiration, failure handling, and safe metrics.
5. Run two independent agent sessions against one service and execute the demo matrix against the native baseline.

| Workstream | Boundary |
|---|---|
| Provider and source identity | Existing Git/search integration, exact source reads, snapshots, capabilities, and completeness |
| Shared cache and access | Redis schema/lifecycle, effective scopes, expiry, publication, and miss coalescing |
| MCP and optimizer | Typed tools, preview/detail packing, pagination, and response budgets |
| Evaluation and demo | Agent configuration, baseline tasks, usage telemetry, quality review, and team-reuse demonstration |

Open implementation choices are package pins, deployment/authentication configuration, pilot corpus, tokenizer integration, concrete limits/TTL, and measured acceptance thresholds. Shared use and token optimization are **not optional later phases**.

## 12. Sources and historical context

These sources support individual building blocks, not AgentCacheX performance claims:

1. [VS Code agent workspace context](https://code.visualstudio.com/docs/agents/reference/workspace-context) - existing file, text, semantic, and language-aware context tools.
2. [Writing effective tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents) and [Code execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp) - efficient tool interfaces and keeping unnecessary intermediate data out of model context.
3. [MCP tools specification](https://modelcontextprotocol.io/specification/2025-06-18/server/tools) - tool request/result integration.
4. [Official MCP C# SDK](https://github.com/modelcontextprotocol/csharp-sdk) - proposed .NET service integration.
5. [Git grep documentation](https://git-scm.com/docs/git-grep) - existing text search against supplied source trees.
6. [Redis SET](https://redis.io/docs/latest/commands/set/) - conditional creation and expiration primitives.
7. [Redis licensing](https://redis.io/legal/licenses/) - review the selected server/distribution's license before deployment; Redis server and client-library licenses are separate.
8. [Storage research](RESEARCH-STORAGE.md), [memory-framework research](RESEARCH-MEMORY-FRAMEWORKS.md), and [conversation history](CONVERSATION.md) - preserved earlier comparisons and the decision to simplify.
