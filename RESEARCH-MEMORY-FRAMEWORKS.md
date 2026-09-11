# AgentCacheX: Memory Framework Research

**Assessment date:** 2026-09-11  
**Scope:** Graphiti, Mem0 OSS, and LlamaIndex ingestion/caching for one C# repository  
**Method:** Public source inspection, not runtime compatibility or performance measurement

Inspected snapshots: Graphiti `c64e45c`, Mem0 `c7ee362`, and LlamaIndex `5fb3f39`. These identify reviewed source, not installed package releases.

## Conclusion

**Reuse selected components; do not replace the evidence-cache design with an entire memory framework.**

LlamaIndex's ingestion cache is the closest reusable component. Graphiti is attractive if temporal graph exploration is itself the demo. Mem0 OSS is useful primarily for searchable memory storage.

None of the inspected contracts establishes the end-to-end guarantee that a retrieved claim matches the current local working tree and changed dependencies. They offer different update mechanisms, not interchangeable notions of freshness. Custom metadata can participate in such a guarantee, but AgentCacheX must calculate, check, and enforce it.

## Candidate comparison

| Candidate | Actual model and freshness mechanism | Reusable pieces and remaining work |
|---|---|---|
| **Graphiti** | Episodes carry content, source description, ingestion time, and supplied reference time. Fact edges include embeddings, supporting episode IDs, attributes, and validity/expiration timestamps. Ingestion extracts/resolves entities and facts, including facts invalidated by new information [1] [2]. | Useful provenance links, typed entities/edges, temporal retrieval, and graph storage. Add repository identity, source locations, fingerprints, source-change detection, dependency invalidation, and retrieval-time source gating. Temporal knowledge validity is not observation of local file changes. |
| **Mem0 OSS** | Vector-backed memories have IDs, text, text hashes, metadata, timestamps, user/agent/run scoping, and SQLite history. The inspected ingestion path is additive: retrieve context, extract memories with an LLM, deduplicate text hashes, and insert. Explicit update/delete APIs remain. Expiration hides old memory; neither expiration nor the memory-text hash validates source files [3]. | Reuse search, metadata filtering, CRUD, and history. `infer=False` preserves supplied text without extraction but still generates embeddings. Supply evidence IDs, source lineage, source/dependency fingerprints, and invalidation. |
| **LlamaIndex** | A transformation cache key includes the supplied node sequence's content/metadata and serialized transformation configuration. Cached outputs are serialized nodes. Document management uses stable IDs and hashes to skip unchanged inputs, reprocess changes, and optionally delete missing documents [6] [7]. | Reuse transformation caching, persistence, and document-change handling. Supply stable file/symbol identities, dependency fingerprints, extraction/model/prompt versions, repository enumeration, and a read-time source validator. |

## Implementation qualifications

### Graphiti

The direct-triplet API is not a zero-model-cost insertion path. It bypasses extraction but can generate missing embeddings and perform entity/edge resolution, including an LLM-backed edge-resolution call [2].

The OSS MCP server includes ingestion/search/provenance/delete capabilities. Its `add_memory` queues ingestion, so acceptance is not proof that evidence is already searchable. Readiness and publication policies remain necessary [4].

### Mem0

There is a documentation inconsistency: the `add()` docstring describes ADD/UPDATE/DELETE decisions, while the inspected implementation explicitly follows the additive pipeline. Base integration behavior on the pinned implementation, not older descriptions [3].

The official `https://mcp.mem0.ai/mcp` endpoint is hosted Mem0 Platform and stores memories in the platform account, not locally.

An OSS Mem0-based MCP implementation exists under OpenMemory's `openmemory-archive` directory. That demonstrates an integration approach, not a verified current drop-in pairing with the latest SDK [5].

### LlamaIndex

Cache granularity is not always one independent entry per node: the actual key covers the node sequence passed to a transformation. Batch composition therefore affects reuse.

The inspected upsert/delete path requires the corresponding document and vector stores. Without the vector store, the pipeline falls back to duplicate-only handling. Use deletion strategies with the complete intended collection, not a changed-file-only batch [6].

OSS MCP utilities include `workflow_as_mcp`, which can expose a custom retrieval/validation workflow. An ingestion cache is not by itself a ready-made repository-evidence MCP service [8].

## MCP and operational footprint

| Framework | Integration | Costs and dependencies |
|---|---|---|
| Graphiti | Included OSS MCP server | Python, graph database, embeddings, LLM-backed extraction/resolution, optional summarization |
| Mem0 OSS | Controlled custom MCP adapter; distinguish from hosted platform and archived application | Python, vector storage, embeddings, SQLite history; extraction LLM for normal ingestion; optional reranking/entity work |
| LlamaIndex | OSS workflow-to-MCP utilities | Python sidecar if the main application is C#; local default KV ingestion cache; embeddings/LLM extraction only when selected, although the default pipeline includes an embedding model |

LlamaIndex deterministic transformations do not inherently require a generation LLM. Mem0 raw insertion can avoid extraction but not its embedding work. Graphiti direct triplets can reduce extraction without removing all model work.

## Required AgentCacheX correctness layer

1. **Snapshot identity:** fingerprint actual relevant working-tree content, not only HEAD. Define staged/dirty/new/generated input and unsaved-buffer scope.
2. **Evidence cards:** retain claims/excerpts, file/symbol/span, source digest, dependency digest, analysis versions, and evidence strength.
3. **Dependency-aware invalidation:** account for source dependencies and build-resolution inputs such as project files, props/targets, framework/defines, resolved packages, and generators. Use conservative scope when coverage is unknown.
4. **Read-time source gating:** bind reuse to a coherent observed snapshot. Watchers accelerate discovery but are not the correctness authority.
5. **Freshness separate from truth:** unchanged source proves unchanged inputs, not a correct generated interpretation. Keep supporting evidence and regenerate the final response.

## Adoption decision

Retain a small local evidence store and explicit MCP boundary.

Adopt LlamaIndex selectively if a Python sidecar is already convenient. Otherwise implement the narrow content-addressed cache in the C# service rather than adding a framework solely for persistence.

Keep Graphiti for a later graph/history-focused extension. Choose Mem0 only if its memory interface materially simplifies the application, not as the source-freshness authority.

## Licensing

The inspected Graphiti and Mem0 OSS projects are Apache-2.0. LlamaIndex core is MIT. These licenses do not automatically cover every deployed database, provider, model, or hosted service.

- [Graphiti license](https://github.com/getzep/graphiti/blob/c64e45c111fd43ca7f5536a7f84a73f6acafd5b1/LICENSE).
- [Mem0 license](https://github.com/mem0ai/mem0/blob/c7ee362aff94a369af70f13f2b4f853f6793ff4c/LICENSE).
- [LlamaIndex license](https://github.com/run-llama/llama_index/blob/5fb3f398b1271edd0ba297912dbfa3cce7d58632/LICENSE).

## Primary source anchors

[1]: https://github.com/getzep/graphiti/blob/c64e45c111fd43ca7f5536a7f84a73f6acafd5b1/graphiti_core/edges.py#L263-L285
[2]: https://github.com/getzep/graphiti/blob/c64e45c111fd43ca7f5536a7f84a73f6acafd5b1/graphiti_core/graphiti.py#L1704-L1803
[3]: https://github.com/mem0ai/mem0/blob/c7ee362aff94a369af70f13f2b4f853f6793ff4c/mem0/memory/main.py#L879-L1060
[4]: https://github.com/getzep/graphiti/blob/c64e45c111fd43ca7f5536a7f84a73f6acafd5b1/mcp_server/src/graphiti_mcp_server.py#L446-L493
[5]: https://github.com/mem0ai/mem0/blob/c7ee362aff94a369af70f13f2b4f853f6793ff4c/docs/platform/mem0-mcp.mdx#L8-L30
[6]: https://github.com/run-llama/llama_index/blob/5fb3f398b1271edd0ba297912dbfa3cce7d58632/llama-index-core/llama_index/core/ingestion/pipeline.py
[7]: https://github.com/run-llama/llama_index/blob/5fb3f398b1271edd0ba297912dbfa3cce7d58632/llama-index-core/llama_index/core/ingestion/cache.py#L17-L74
[8]: https://github.com/run-llama/llama_index/blob/5fb3f398b1271edd0ba297912dbfa3cce7d58632/llama-index-integrations/tools/llama-index-tools-mcp/llama_index/tools/mcp/utils.py#L77-L142

Additional inspected source ranges:

- [Graphiti episode ingestion](https://github.com/getzep/graphiti/blob/c64e45c111fd43ca7f5536a7f84a73f6acafd5b1/graphiti_core/graphiti.py#L1156-L1239).
- [Graphiti MCP deployment](https://github.com/getzep/graphiti/blob/c64e45c111fd43ca7f5536a7f84a73f6acafd5b1/mcp_server/README.md#L79-L111) and [tool documentation](https://github.com/getzep/graphiti/blob/c64e45c111fd43ca7f5536a7f84a73f6acafd5b1/mcp_server/README.md#L556-L579).
- [Mem0 ingestion API](https://github.com/mem0ai/mem0/blob/c7ee362aff94a369af70f13f2b4f853f6793ff4c/mem0/memory/main.py#L487-L514), [hash/scoping behavior](https://github.com/mem0ai/mem0/blob/c7ee362aff94a369af70f13f2b4f853f6793ff4c/mem0/memory/main.py#L760-L802), and [expiration behavior](https://github.com/mem0ai/mem0/blob/c7ee362aff94a369af70f13f2b4f853f6793ff4c/mem0/memory/main.py#L1961-L1990).
- [Archived OpenMemory MCP server](https://github.com/mem0ai/openmemory/blob/2df78cc48e5f688a771fee5b39899325accc2450/openmemory-archive/api/app/mcp_server.py#L24-L95) and [OSS client construction](https://github.com/mem0ai/openmemory/blob/2df78cc48e5f688a771fee5b39899325accc2450/openmemory-archive/api/app/utils/memory.py#L478-L486).

## Uncertainty

This is source inspection, not proof of runtime compatibility or performance. The bounded finding is "native working-tree/dependency freshness guarantee not established by the inspected contracts," not "custom integrations are impossible" or "the ecosystem has no competing solutions."
