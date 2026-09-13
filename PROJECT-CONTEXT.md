# AgentCacheX: Project Context and Team Handoff

**Updated:** 2026-09-13 - focused shared-cache MVP  
**Purpose:** Give a teammate or separate agent session the current decisions without requiring the chat.

## The agreed problem

Multiple developers working on the same repository repeat the same read-only code lookups and send oversized, repetitive results to their agents. AgentCacheX shares version-scoped tool results and returns only the details each agent needs.

**Build a shared, programming-language-agnostic tool-result cache plus a deterministic token optimizer.** Reuse existing providers. Do not build a semantic-analysis or agent-memory platform.

The two benefits are independent: shared results reduce repeated backend work; compact previews and selected details reduce unnecessary tool payload on misses as well as hits. The existing agent still produces a fresh answer.

## Current status

The repository contains design documents, research history, an offline interactive architecture viewer, three updated PowerPoint decks, and a presentation-generation utility. **There is no cache backend, MCP implementation, Redis deployment, benchmark harness, or measured savings yet.**

The video, captions, player, and video-generation scripts were removed at the user's request. Do not treat historical mentions as an instruction to recreate them.

The current design is [AgentCacheX-High-Level-Design.md](AgentCacheX-High-Level-Design.md); [README.md](README.md) is the concise overview. Earlier research and conversation sections are historical, not competing implementation plans.

## Fixed MVP decisions

| Decision | Meaning |
|---|---|
| Shared use is core | At least two developers, separate sessions, one repository, one authenticated service |
| Token optimization is core | Keep complete captured results outside model context; return previews and batched selected details |
| Language-agnostic boundary | Cache provider outputs and exact source ranges, not compiler-specific objects |
| One existing provider first | Recommended baseline: `git grep` and source reads pinned to a verified commit in a trusted mirror |
| Redis for shared results | One instance initially; no shared database file or required local L1 |
| Proposed .NET/ASP.NET Core MCP host | Host language does not restrict repository languages |
| Exact snapshot-qualified reuse | Match actual source/provider state and upstream arguments, not semantic question similarity |
| Coarse revision invalidation | New source revision means a new key; no fine-grained dependency graph |
| Deterministic output shaping | Preserve source text and citations; no LLM summaries or code minification |
| Honest outcome measurement | Measure total task usage and quality against already-bounded native tools |

The Git pilot supports **text matches**, not proven symbol references or call relationships. A later capable provider can supply those operations; AgentCacheX need not implement them. Roslyn is only relevant to an upstream C# analyzer and is not an MVP dependency.

## The complete request story

1. An agent explicitly invokes `search_compact` through the configured MCP wrapper.
2. The service authenticates the caller, authorizes repository/content scope, and resolves the actual searchable snapshot.
3. It checks the shared raw-result key. On a miss it runs the existing provider and stores the captured result.
4. It returns compact source-anchored previews, stable item IDs, and explicit coverage/pagination metadata.
5. The agent calls `fetch_details` for selected items in a batch, still bound to the same snapshot.
6. Another authorized developer's equivalent query reuses that result, potentially with a different preview budget.

If a detail expansion was not captured by the original search, perform and count a separate source read at the pinned revision. Do not assume a search artifact contains complete files.

MCP registration does not intercept every other tool. The demo must show actual wrapper calls from separate agent sessions, not merely two invocations inside one session.

## Contracts to freeze before coding

| Contract | Required information |
|---|---|
| Snapshot | Repository immutable ID, actual source revision, applicable provider/index revision, and explicit verifiability |
| Provider result | Provider/tool/version, canonical arguments, original payload, source locations, capabilities, and upstream completeness/continuation |
| Access scope | Server-derived effective visibility/content-policy identity and policy version; not a model-supplied role |
| Result set | Opaque ID, immutable items with stable IDs, snapshot, access binding, expiration, and full captured response |
| Search response | Hit/miss/bypass status; previews or complete-result page; coverage, cursor, snapshot, expiry, and budget usage |
| Detail response | Exact selected source ranges; fulfilled/pending selections; snapshot, cursor, expiry, and budget usage |
| Response budget | Supported tokenizer and token limit, or byte bound with labeled token estimates |

Proposed tools:

```text
search_compact(query, scope, response_budget, mode="preview", cursor=null)
fetch_details(result_set_id, selections, response_budget, cursor=null)
```

The raw cache key combines repository, actual snapshot, provider/tool version, canonical upstream arguments, effective access scope, and schema version. It does **not** automatically include developer username or presentation-only response budgets.

These are proposed contracts; the JSON examples and edge cases in the HLD must become typed API definitions during implementation.

## Non-negotiable boundaries

- Authenticate and authorize every search, detail fetch, and page. A cached handle does not grant access, including after permission changes.
- Shared results describe committed snapshots. Dirty, unsaved, or unverified workspace state bypasses shared reuse; unsupported states must not silently fall back to HEAD.
- Use the backend's actual source/index version. A client commit hint or `clean=true` is not proof.
- TTL is retention, not freshness. Expiration or eviction returns `RESULT_EXPIRED`, not "no matches."
- Preserve partial-provider status and paginate complete-mode requests. Do not claim grep matches are semantic references.
- Preserve exact source text and locations; deduplicate only safe same-source overlap. Do not assume earlier excerpts remain in a model's active context.
- Include metadata in response-budget accounting; label estimates. Count additional model turns and pinned reads.
- Coalesce concurrent misses with bounded, owner-checked leases; do not claim universal exactly-once execution or high availability.

## Team workstreams

| Workstream | Deliverable | Integration boundary |
|---|---|---|
| Provider/source | Existing search and pinned reads, trusted snapshot resolver, capability/completeness handling | Provider-result contract |
| Cache/access | Shared Redis records, permission-qualified keys, expiry, atomic publication, miss coalescing | Result-set lifecycle |
| MCP/optimizer | Two typed tools, previews, detail batches, pagination, budget accounting | Model-visible responses |
| Evaluation/demo | Two-session configuration, native baseline, workload, usage and quality measurements | Demo matrix and evidence |

Use separate clones/checkouts, branches, and agent sessions. Integrate through pull requests, not concurrent edits to a shared OneDrive directory. A shared repository or cache does not synchronize conversations or grant another person control of a live session.

## Demo and decision gates

Use a small repository with the same identifier, such as `VIP_DISCOUNT`, in C# and TypeScript files. This demonstrates language-neutral text caching, not semantic equivalence across languages.

| Scenario | Required observation |
|---|---|
| Developer A searches committed S1 | Cold provider execution and compact output |
| Developer B repeats at S1 | Shared hit, without repeating the search |
| B changes only the preview budget | Same captured result, differently packed output |
| Selected details are needed | Batched exact source, or separately counted pinned expansion |
| Source becomes S2 | Different key; source-correct new result |
| Dirty/unknown workspace is requested | Visible bypass/unsupported state |
| Unauthorized or expired handle is fetched | Denial or explicit expiration, never leaked/empty success |
| Many matches exceed one budget | Explicit pages, preserved completeness |

Compare native bounded output, optimizer-only, and optimizer-plus-cache. Record backend executions, all model input/output tokens across the task, repeated context, latency, actual billed cost where available, and answer quality.

The **10,000 -> 300 preview + 1,200 detail = 1,500 tokens** example is illustrative tool-payload arithmetic. The apparent 85% reduction is not a measured total-task saving. If the agent needs every result or performs more reasoning turns, total savings can be small or negative.

There is no current annual ROI, latency target achieved, or measured token-reduction claim. Acceptance requires correct access/snapshot/completeness behavior, no material answer-quality regression, and worthwhile measured benefit over existing tools.

## Still-open implementation choices

| Choice | Direction |
|---|---|
| Pilot repository and tasks | Select a permitted corpus with repeated multi-developer lookups |
| Deployment/authentication | One shared MCP endpoint; choose the approved identity and repository-access mechanism |
| Package versions | Pin supported .NET, MCP, Redis client/server, and tokenizer dependencies when implementing |
| Capacity and retention | Set entry size, total memory, TTL, timeout, lease, page, and source-range limits |
| Exact token accounting | Select the agent model/tokenizer and document byte-bounded estimate behavior |
| Success thresholds | Set workload-specific gates before measuring; do not assume a savings percentage |

No open choice reintroduces custom semantic analysis, vectors, graph memory, local-only caching, or makes token optimization optional.

## History and preserved context

The original materials proposed a broader evidence/memory layer, a C# pilot, dependency-aware freshness, and modeled savings. Later research compared storage and memory frameworks. The user clarified language independence, then asked to simplify around existing agent tools while retaining token optimization and shared use.

That progression **supersedes** the evidence-card/Roslyn-adapter/SQLite/FTS5/vector/dependency-graph implementation plan. The current README, HLD, diagram, and all three decks describe the focused shared-cache solution. Original deck versions remain in Git history; original brainstorming and dated research remain accessible.

[CONVERSATION.md](CONVERSATION.md) preserves the evolution, including repository creation under `Avish34`, the subsequent public-visibility decision, language clarification, video removal, and focused MVP decision. [GeminiResponse.txt](GeminiResponse.txt) remains brainstorming, not verified market evidence.

The intended repository is [Avish34/AgentCacheX](https://github.com/Avish34/AgentCacheX). Public visibility permits viewing/forking and proposing pull requests, not automatic write access. The earlier repository under the initially active account was left untouched.

The user's artifact-handling constraint remains: **do not read SVG or PNG files**. The updated decks use native editable shapes/text rather than image assets; video creation is not part of this task.
