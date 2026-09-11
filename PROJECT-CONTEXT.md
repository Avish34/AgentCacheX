# AgentCacheX: Project Context and Team Handoff

**Captured:** 2026-09-11  
**Purpose:** Let another teammate or coding-agent session understand the project without needing the original chat.

## What we are building

AgentCacheX is a hackathon project for repository-aware reuse of coding-agent discoveries. Different questions about unchanged code should not require repeating all file reads, searches, dependency exploration, and context construction.

The proposed cache supplies source-backed evidence to the existing agent. It does not replace the agent runtime and does not blindly replay generated answers, patches, or shell commands.

The initial materials combined broad commercial brainstorming with a narrower, source-provenance-first memory design. The current recommendation follows the narrower design.

## Source material and chronology

1. The original folder contained three PowerPoint decks, `GeminiResponse.txt`, and the August 31 SharedRP memory-layer research.
2. The initial review identified a single-repository C# MVP, evidence cards, MCP integration, AST/Git signals, selective invalidation, and modeled token/latency savings.
3. Team collaboration was discussed. The recommended workflow is separate sessions and branches, with repository documentation as shared context.
4. Public-source research compared six architectures and concrete storage/memory components. The detailed findings are preserved in `RESEARCH-STORAGE.md` and `RESEARCH-MEMORY-FRAMEWORKS.md`.
5. `AgentCacheX-High-Level-Design.md` captured the recommended architecture, contracts, correctness boundaries, deployment, and evaluation.
6. `AgentCacheX-Architecture.html` added three offline diagram views, including Mermaid export and print/PDF.
7. The project owner requested a private GitHub repository containing all materials, context, presentations, and responses.

The initial request excluded reading SVG/PNG files. Slide text was extracted without opening embedded images. The Keynote-Style deck was locked during that review; its original file is retained without claiming its content was reviewed.

## Current implementation status

This is a research/design repository. No backend cache service, package manifest, MCP server implementation, benchmark harness, or measured savings has been created yet.

The HTML architecture viewer is implemented documentation. It is not a cache-service UI.

## Scope

| In the first implementation | Outside the first implementation |
|---|---|
| One configured C# repository | General non-code or multi-repository memory |
| File/symbol explanation and static dependency tracing | Runtime truth about unobserved databases/services/flags |
| Saved working-tree files and explicit snapshots | Unsaved buffers without a host adapter |
| Exact/lexical retrieval and source-qualified cards | Blind semantic answer/patch/trajectory replay |
| Conservative dependency freshness and partial results | An assertion that AST similarity proves behavioral equivalence |
| MCP tools and measured capture coverage | Transparent interception of every host tool |
| Optional measured semantic recall | Mandatory vectors, graph database, or Redis cluster |
| Optional authenticated clean-snapshot team cache | Sharing dirty edits or a live database through OneDrive |

The original decks include semantic matching in the MVP. The current recommendation sequences a correctness-first exact/lexical baseline before adding vectors. A paraphrase of the same structured intent/target can match without vector search; that must not be presented as proof of a semantic-vector feature.

## Architectural decisions

| Decision | Reason |
|---|---|
| Reuse evidence, not final answers | Current answers should be grounded in the requested source state |
| Keep immutable artifacts separate from evidence and compiled context | Acquisition identity, reusable meaning, and per-request output have different lifetimes |
| C#/.NET plus official MCP SDK and Roslyn | Direct fit for the initial C# repository and semantic-analysis requirements |
| SQLite authority, exact keys, FTS5 | Minimal operational footprint and coherent evidence/dependency transactions |
| Optional sqlite-vec behind an interface | Semantic retrieval should demonstrate value; pre-v1 packaging needs care |
| Freshness per evidence/snapshot association | A fact may be current in one checkout and historical or stale in another |
| Fingerprint relevant compilation/source closure for semantic cards | New overloads, partial declarations, or configuration can change binding |
| Prefer excerpts and deterministic facts | Unchanged hashes do not prove an LLM interpretation was correct |
| Treat suspect/stale records as unusable for current claims | Validation failure is not permission to serve old information as current |
| Separate local L1 and authenticated team L2 | Local dirty state and cross-user clean-snapshot reuse have different boundaries |
| Defer PostgreSQL/Qdrant until justified | A single service with server-local SQLite can prove the team path |
| Compare against incremental code RAG | Ship the simpler design if learned evidence adds no quality or cost benefit |

## Proposed shared contracts

Agree on these before dividing implementation work.

**Snapshot:** repository ID, principal/scope, checkout/ref/commit, actual source manifest, project context fingerprint, and observation time.

**Evidence card:** ID, type, target, covered aspects, exact source IDs/spans, claim or excerpt, evidence strength, dependency manifest, and extractor/derivation versions.

**Freshness association:** evidence ID, requested snapshot, state, coverage, reason codes, and observation/validation time.

**Lookup response:** `hit`, `partial`, or `miss`; usable cards; observed snapshot; missing aspects; explicit reasons; budget/cost metadata.

**Tools:** `cache_lookup`, `code_read_cached`, `code_search_cached`, `cache_record`, and `cache_explain`.

Source acquisition generates hashes and anchors. Do not trust a model-supplied `fresh=true`, commit, or citation as authoritative.

## Expected request flow

1. The agent supplies query, target, requested aspects, and budget.
2. AgentCacheX resolves repository/project/access scope and the observed workspace.
3. Exact/lexical retrieval, with optional semantic recall, produces candidates.
4. Source freshness and coverage determine usable evidence.
5. A miss or partial result triggers only the necessary supported acquisition and analysis.
6. Evidence is admitted with source support and dependencies.
7. The compiler returns bounded cited context.
8. The existing LLM produces a new answer.

Normal tool exploration remains available when the cache cannot help, but failures and bypasses must be visible in metrics.

## Demo narrative

- **Cold:** ask what a selected file/symbol does; acquire evidence and record the work.
- **Warm:** rephrase the same request and demonstrate reuse without repeated broad exploration.
- **Change:** modify a source dependency and show affected evidence being rejected/refreshed.
- **Unrelated change:** show preserved file-local or unrelated-project evidence.
- **Team extension:** use a separate authorized checkout to reuse clean-snapshot evidence, then demonstrate that local dirty changes invalidate it.

## Performance assumptions, not results

The decks model 20 developers x 15 queries/day x 220 working days = 66,000 annual requests, with 60% reusable.

| Metric | Cold assumption | Warm assumption |
|---|---:|---:|
| Model input tokens | 8,000 | 1,500 |
| Elapsed time | 18 seconds | 4 seconds |
| Repository tool calls | 12 | 3 |

At 60% usable reuse and no added overhead, the average is 4,100 input tokens, 9.6 seconds, and 6.6 repository calls. Modeled annual avoidance is 257.4 million input tokens, 356,400 repository calls, and 154 aggregate hours of wait.

Include provider cached-input pricing, embeddings, cache lookup, extraction, invalidation, extra miss work, fixed system/tool prompts, and service overhead before claiming savings. Aggregate waiting time is not automatically recovered developer labor.

Proposed gates include at least 95% supported/relevant evidence precision, zero known stale current-state serves in the mutation suite, complete source anchors, no material answer-quality regression, and a steady-state p95 lookup target below one second on stated hardware/corpus.

These are target gates, not completed measurements or mathematical guarantees.

## Parallel workstreams

| Workstream | Ownership |
|---|---|
| Storage/retrieval | Schema, exact cache keys, FTS5, optional vectors, retrieval APIs |
| Source/freshness | Git snapshots, Roslyn extraction, dependency manifests, mutation fixtures |
| Agent integration | MCP tools, cached acquisition, context compiler, capture coverage |
| Evaluation/demo | Baselines, labeled questions, cost/quality metrics, demo narrative |

Each teammate should read this handoff and the HLD, work in their own branch/session, and integrate through pull requests. A shared repository does not automatically synchronize agent conversations.

## Decisions still open

- Select the pilot repository and initial file/symbol subset.
- Choose and pin the supported .NET, Roslyn, MCP, SQLite, and optional vector packages.
- Define the exact evidence schema and structured MCP response types.
- Decide whether semantic recall is needed for the first demo.
- Select an approved embedding runtime/model only if it is needed.
- Choose the trusted project-loading/build context and handling of unresolved dependencies.
- Decide whether the hackathon demonstrates local-only or actual cross-developer reuse.
- Select deployment and authentication for the team hub if included.
- Assign workstream owners and agree on acceptance criteria.

## How to interpret the research

The original Gemini response is brainstorming, not proof of market exclusivity or competitor limitations. The later research uses official documentation and pinned source references, but it did not establish workload performance or installed-package compatibility.

Graphiti's temporal validity, Mem0's memory expiration/hash, and LlamaIndex's document/transform hashes are not the same as local Git dependency freshness. Custom integrations may supply that policy; the inspected native contracts do not establish the complete guarantee.

The design should evolve with evidence. Preserve the original inputs, date revised findings, and keep measured results distinct from proposals.
