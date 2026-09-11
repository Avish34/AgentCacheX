# AgentCacheX: Project Conversation

**Date:** 2026-09-11

This captures the project-facing prompts and final responses from the discovery, research, design, and diagram discussions. Typography is normalized to plain ASCII; the original documents and presentations are retained separately. Research details are in the two research reports and HLD. Credentials, machine-specific runtime state, and tool execution logs are not project documentation.

## 1. Initial folder review

### User

> We are creating a hackathon project - please check this folder and contents to understand, and let me know what did you understand. Also, don't read svg or png files. Name this sesion "AgentCacheX".

### Assistant

**AgentCacheX is a repository-aware evidence cache for coding agents: reuse previously discovered code knowledge safely, rather than repeatedly exploring unchanged code or blindly replaying old LLM answers.**

I read both text documents and extracted slide text from the Coding-Agent-Cache and Neon-Architecture decks. **No SVG or PNG files were read**, including embedded images. The Keynote-Style deck was locked by another process, so I couldn't read it.

#### What I understand

- **The problem:** Different developers and agents repeatedly read the same files, search symbols, trace dependencies, and rebuild context. This wastes tool calls, tokens, and time, even when the relevant code hasn't changed.
- **The solution:** Put AgentCacheX between coding agents and repository exploration, exposed through **MCP tools**. Store reusable **evidence cards** containing file responsibilities, symbols, control flow, dependencies, tool results, and precise source citations.
- **How reuse works:** Semantic matching finds similarly intended requests; exact identifiers and AST fingerprints help establish code identity; Git state, working-tree changes, and dependency hashes determine freshness. **AST similarity alone is not proof of equivalent behavior.**
- **What the LLM receives:** A small, relevant, cited context pack instead of full files and repeated raw search output. **The coding agent still reasons and generates the current answer.** Final patches and shell-command sequences are not blindly replayed.
- **How changes are handled:** Reuse fresh evidence, refresh only missing or changed portions, and invalidate evidence affected by source or dependency changes. Unrelated changes should preserve useful cache entries.

#### Hackathon scope and architecture

The decks describe a focused MVP: **one C# repository, file/symbol explanations, dependency tracing, semantic matching, AST fingerprints, Git-aware freshness, MCP integration, and metrics.** Cross-repository memory and enterprise-scale deployment are outside that scope.

The research document supplies a practical foundation: **SQLite as the authoritative store, FTS5 for exact retrieval, and custom provenance, invalidation, and context-compilation logic.** It recommends adding vectors only after measuring lexical misses, whereas the decks include semantic matching in the MVP: one scope difference to reconcile.

The intended demo is clear: **cold request -> rephrased warm hit -> code change -> selective invalidation.** The advertised reductions, 8,000 to 1,500 input tokens, 18 to 4 seconds, and 12 to 3 tool calls, are explicitly **illustrative targets, not measured results**.

This folder currently contains **research and pitch material, not an implementation**. The strongest differentiator is *safe reuse tied to actual repository state*, rather than generic semantic response caching.

**Session name:** I can't rename it directly with the available tools. Use `/rename AgentCacheX`.

## 2. Team collaboration

### User

> Can I share this sessions to my team so they can work paralelly? Can this session be used concurently?

### Assistant

**You can share the conversation with your team, but don't use one live session as a concurrent, multi-person workspace.** For parallel development, each teammate should use their own session.

| Goal | How |
|---|---|
| Share our discussion and findings | Run `/share` and choose a Markdown export. Share it through your approved team location. |
| Control this session from another device | `/remote` supports remote access, but GitHub web/mobile access requires **the same GitHub account** that started the session, not teammates' accounts. [Documentation](https://docs.github.com/en/copilot/how-tos/copilot-cli/use-copilot-cli/steer-remotely) |
| Have teammates work simultaneously | Give everyone the project files and shared context, then use **separate sessions and Git branches**. |

Sharing the conversation **doesn't grant access to your local folder or tools**, and separate sessions won't automatically receive each other's new discussions.

**For AgentCacheX, I recommend:**

1. Put the project in a team-accessible private Git repository. This folder isn't a Git repository yet.
2. Share the session export and keep agreed architecture, interfaces, and task ownership in the repository. Review the export for internal or sensitive content before sharing.
3. Each teammate clones the repository, opens their own session, and works on a separate branch, for example, storage/retrieval, Git/AST freshness, MCP integration, or demo/metrics.
4. Integrate through pull requests rather than having multiple agents edit the same OneDrive working folder concurrently.

## 3. Research and high-level design

### User

> Now that you have problem statement, try to think through and find me multiple solutions which can solve this problem. Get whatever you need from web, open source projects, db, caches. Create a high level design and present me the best solution to solve this problem.

### Assistant

**My recommendation is a Git-aware, dependency-aware evidence cache, not a generic semantic response cache.** It should reuse the source evidence behind an answer, then let the coding agent generate a fresh answer.

Created the full design in **[AgentCacheX-High-Level-Design.md](AgentCacheX-High-Level-Design.md)**, including architecture, alternatives, data model, MCP contracts, team deployment, implementation stages, and source references.

#### Six approaches considered

| Solution | What it saves | Assessment for AgentCacheX |
|---|---|---|
| **Provider prompt caching** | Reprocessing identical prompt prefixes | Useful alongside our solution, but doesn't prevent repeated repository exploration |
| **Exact tool-result cache** | Repeated reads, searches, and symbol queries | Simplest starting point; returning large cached results still consumes model context |
| **Semantic response cache: RedisVL/GPTCache** | Can bypass an entire LLM call | Risky for coding: similar questions can concern different code states or requirements |
| **Incremental code RAG** | Repeated discovery and full-file context loading | Strong alternative: maintain a current code index and retrieve relevant snippets |
| **Agent memory/graph systems: Mem0/Graphiti** | Persistent discoveries, relationships, historical knowledge | Useful components, but their memory-update mechanisms don't establish working-tree freshness |
| **Dependency-aware evidence cache** | Repository exploration, repeated analysis, and unnecessary context | **Best fit:** combines exact caching and incremental retrieval with explicit source dependencies |

The strongest simpler alternative is **incremental code RAG**. If it delivers equivalent quality and savings without storing derived discoveries, we should ship that simpler version first.

#### Recommended high-level design

```text
Coding agent: question + target file/symbol + requested aspects
                              |
                         MCP adapter
                              |
                  Exact / lexical lookup
                  Optional semantic retrieval
                              |
                  Source freshness + coverage gate
                       /                 \
                  Usable evidence     Missing/stale evidence
                       |                 |
                       |          Cached source reads/search
                       |          Git + Roslyn analysis
                       \                 /
                        Evidence store
                              |
                  Token-budgeted context compiler
                              |
                    Existing answering LLM
```

An **evidence card** contains a supported fact or excerpt, target symbol, source location, content hash, dependency fingerprints, and analysis version. The cache returns those cards, not an old answer or patch.

#### Technology choices

- **C#/.NET + [official MCP SDK](https://github.com/modelcontextprotocol/csharp-sdk):** one application with separate modules, avoiding unnecessary services.
- **[Roslyn](https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/work-with-semantics) + Git:** C# syntax, symbol resolution, project context, and actual working-tree contents.
- **[SQLite + FTS5](https://sqlite.org/fts5.html):** authoritative evidence, exact identifiers, dependencies, and lexical retrieval.
- **Optional sqlite-vec:** add semantic retrieval only when it improves recall beyond structured and lexical matching.
- **Later:** authenticated shared service; PostgreSQL/Qdrant only when concurrency or retrieval requirements justify them.

LlamaIndex offers useful ingestion caching, but introducing a Python sidecar solely for persistence is unnecessary here. Graphiti becomes attractive if architecture history and relationship exploration become major requirements.

#### The important correctness decisions

**Similarity finds candidates; source dependencies determine reuse.** Neither a high embedding score nor matching AST shape proves an answer still applies.

**HEAD alone is insufficient.** Uncommitted edits, changed dependencies, added overloads, project configuration, and newly added search matches can invalidate evidence. Start with conservative project-level dependencies for semantic claims; optimize precision later.

**Freshness and truth are separate.** Unchanged source does not prove an earlier LLM interpretation was correct. Prefer exact excerpts and deterministic analyzer facts; label generated summaries as derived.

**MCP integration must be explicit.** Registering our server does not automatically intercept existing tools or expose IDE buffers. Supported reads/searches must go through AgentCacheX.

#### Local versus team reuse

Start with a local cache for each working tree. To demonstrate **Developer B benefiting from Developer A's work**, add an authenticated team service for clean, immutable snapshots. Each client checks shared evidence against its own checkout; dirty changes stay local.

Do **not** share the live SQLite database through OneDrive. Keep runtime storage outside the synced project folder.

The hackathon should demonstrate **cold request -> rephrased warm request -> dependency change -> selective refresh**, while reporting total costs, including cache overhead. The deck's approximately 81% token reduction is a warm-hit assumption; at 60% reuse, it represents approximately **49% workload-wide reduction before overhead**, not an established product result.

## 4. Architecture diagram

### User

> Create a digram with the flow and all the component involved.

### Assistant

Created **[AgentCacheX-Architecture.html](AgentCacheX-Architecture.html)**. Open it in a browser to explore three views:

1. **Request flow:** cold requests, warm hits, partial refresh, and answer generation.
2. **Storage and invalidation:** Git, Roslyn, evidence cards, dependencies, SQLite, FTS5, and optional vectors.
3. **Team sharing:** private local caches, authenticated shared cache, and optional scale-out.

Works offline with **zoom, Print/PDF, and editable Mermaid export**. Also linked from the high-level design.

## 5. Repository request

### User

> Let's create a github repo and include all this docs there, make it private. Add readme.md and all the context regarding research, ppt, responses, EVERYTHING into the repo.

### Repository handoff

The README indexes the original inputs, design, diagram, research reports, and this conversation. `PROJECT-CONTEXT.md` provides the current implementation status, decisions, assumptions, open questions, and proposed team workstreams.

The original presentation and response files remain separate from the later research so that brainstorming, source-backed findings, proposed architecture, and eventual measured results are not conflated.

## 6. GitHub account correction

### User

> I want Avish34 github account to create the repo

### Repository ownership

The intended private project repository is [Avish34/AgentCacheX](https://github.com/Avish34/AgentCacheX). The repository under the initially active `avishporwal_microsoft` account is not deleted or transferred. Existing commit history is preserved; the README clone URL and local remote are updated for `Avish34`.
