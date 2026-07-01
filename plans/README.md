# Review of memory-related projects

This folder holds a comprehensive description of four locally cloned open-source projects, each of which gives an AI coding agent a **persistent memory about a codebase under development** — so the agent enriches its context from an index instead of repeatedly running `find` / `rg` / `grep` / whole-file reads.

| Project | File | Language | Memory store | Interface |
|---|---|---|---|---|
| **OntoShip** (`shared-memory-gitmark`) | [gitmark.md](./gitmark.md) | Python (stdlib) | Plain markdown + SQLite FTS5 (derived) | Claude Code plugin: slash commands + skills |
| **GitNexus** (`shared-memory-gitnexus`) | [gitnexus.md](./gitnexus.md) | TypeScript / Node.js | LadybugDB graph DB + HNSW vectors | CLI + MCP + HTTP + web UI |
| **Gortex** (`shared-memory-gortex`) | [gortex.md](./gortex.md) | Go | In-memory graph → embedded SQLite (default daemon backend); gob+gzip snapshot only for the `memory` backend | 100+ MCP tools + CLI + HTTP + daemon |
| **graphify** (`shared-memory-graphify`) | [graphify.md](./graphify.md) | Python | NetworkX `graph.json` (no DB, no embeddings) | `/graphify` skill + CLI + MCP |

Each per-project file follows the four chapters requested in the issue: **(1)** main purpose of project creation, **(2)** main functionality, **(3)** how it works under the hood, and **(4)** how memory about the codebase is implemented.

## Framing: the agent-memory taxonomy

The reviews are grounded in the survey collected in this repo — *Hu et al. (2026), "Memory in the Age of AI Agents"* (`data/Hu_et_al_2026_Memory_in_the_Age_of_AI_agents.md`), which organizes agent memory along three axes:

- **Form — what carries memory** (§3): *token-level* (explicit external units), *parametric* (baked into model weights), or *latent* (hidden states / embeddings).
- **Function — why memory exists** (§4): *factual* ("what the agent knows"), *experiential* ("how the agent improves"), or *working* ("what the agent is thinking about now").
- **Dynamics — how memory operates** (§5): *formation* → *evolution* (consolidation / updating / forgetting) → *retrieval*.

All four projects are close cousins of the survey's §6.2 open-source memory frameworks and its §7.5 *shared memory for multi-agent systems* — except the shared substrate here is not chat history but the **structure of the codebase itself**.

## Where each project sits in the taxonomy

- **Form.** All four are overwhelmingly **token-level**: memory is external, human/agent-readable, and injected verbatim (markdown docs, graph nodes/edges, generated `CLAUDE.md`). **None is parametric** — no project fine-tunes a model. Two add a *latent* slice via embeddings for semantic search: **GitNexus** (384-dim HNSW vectors) and **Gortex** (baked GloVe-50d by default over `coder/hnsw`, with MiniLM/Ollama/OpenAI opt-in). **OntoShip** and **graphify** deliberately avoid embeddings entirely (FTS5 and graph-structure-as-similarity, respectively).

- **Function.** All four center on **factual/environment memory** — durable facts about what the code is and how its parts connect. Two go further into **experiential memory** that learns across sessions: **Gortex** (feedback / frecency / combo stores that rerank future context) and **graphify** (`save-result` outcomes distilled by `reflect` into `LESSONS.md`). The three graph tools (GitNexus, Gortex, graphify) add a **working-memory injection** layer — Claude Code hooks plus generated instruction files that push the right slice into the live context window (and, in Gortex, a PreCompact hook that preserves orientation across context compaction). **OntoShip is the exception**: it has no injection hooks, steering the agent instead through its `kb-search` skill and slash commands, which prompt the agent to search the KB and then read the top hits itself.

- **Dynamics.** *Formation* is a parse/index pass (tree-sitter graphs for GitNexus/Gortex/graphify; heading-chunked markdown for OntoShip; LLM-assisted for graphify's docs). *Evolution* differs by project: **Gortex and graphify do true incremental indexing** — the graph/index is keyed to file mtimes or content hashes and only stale files are re-parsed. **GitNexus is commit-gated rather than truly incremental**: `analyze` early-exits when `meta.json.lastCommit == HEAD` and caches embeddings by content hash, but a re-analyze rebuilds the whole graph from scratch (incremental graph indexing is still on its roadmap). **OntoShip is not commit-keyed at all**: `gitmark index` simply rebuilds the markdown FTS tables on demand, and staleness is handled *socially* through per-doc `status`/`updated`/`supersedes` frontmatter rather than automatic invalidation. *Retrieval* is a scoped query (BM25 / trigram / hybrid / graph traversal) that returns a small, ranked, `file:line`-anchored slice instead of a filesystem scan.

## How they reduce `find` / `rg`

The common mechanism: replace a recursive text scan with a **precomputed, queryable index**. The three graph tools go further and install **Claude Code hooks** that intercept `Grep`/`Glob`/`Read`/`Bash` and redirect the agent to the memory tool first (GitNexus and graphify inject `additionalContext` before the search runs; Gortex's PreToolUse redirects to graph tools). **OntoShip has no such hook** — its only hook is the unrelated destructive-command guard — so it relies on the `kb-search` skill's instructions to make the agent search before grepping. Either way, the index returns *relationships and intent* (callers, call chains, impact/blast-radius, design rationale, curated prose) that grep fundamentally cannot surface, and it does so under an explicit token budget — the shared value proposition across all four.

## Method note

Each description was produced by reading the project's own README / ARCHITECTURE / CLAUDE.md / AGENTS.md docs and its source code directly (schemas, indexers, hooks, MCP tool handlers, storage formats). Concrete file paths and line references are cited inline so every claim is traceable to the source. Where a documented feature was found to be an RFC, mid-migration, or otherwise not fully implemented, it is called out as a caveat rather than presented as shipped behavior.
