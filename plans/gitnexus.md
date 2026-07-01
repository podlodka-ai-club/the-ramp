# GitNexus (`shared-memory-gitnexus`)

| | |
|---|---|
| **Local path** | `~/git/shared-memory-gitnexus/` |
| **Vendor / license** | AkonLabs · PolyForm Noncommercial (commercial/enterprise tier available) |
| **Language / stack** | TypeScript / Node.js monorepo (`gitnexus`, `gitnexus-web`, `gitnexus-shared`); version 1.6.8 |
| **Storage** | LadybugDB embedded graph DB (`@ladybugdb/core`, formerly KuzuDB) + HNSW vector index, per-repo under `.gitnexus/` |
| **Primary interface** | CLI + MCP server (17 tools) + HTTP bridge + React/WASM web UI |

## What is the main purpose of project creation

GitNexus exists to give AI coding agents (Cursor, Claude Code, Codex, Windsurf, OpenCode) a **persistent, structural memory of a codebase** so they stop "shipping blind edits." The README frames the problem concretely (`README.md:523-532`): an agent edits `UserService.validate()`, "doesn't know 47 functions depend on its return type," and breaking changes ship. Text-based tools (`grep`, `rg`, `find`) reveal *strings* but not *relationships* — callers, call chains, inheritance, execution flows.

The stated mission is "**Building nervous system for agent context**" (`README.md:29`): GitNexus "indexes any codebase into a knowledge graph — every dependency, call chain, cluster, and execution flow — then exposes it through smart tools so AI agents never miss code." Its differentiating design is **"Precomputed Relational Intelligence"** (`README.md:560`): unlike traditional Graph RAG that hands an LLM raw edges and hopes it explores enough, GitNexus precomputes clustering, tracing, and confidence scoring *at index time* so a single tool call returns complete context. The claimed payoffs are reliability ("LLM can't miss context"), token efficiency (no 10-query chains), and "model democratization" — smaller models get architectural clarity comparable to frontier models (`README.md:42, 561-564`).

Audiences (`plans/docs/project-overview.md:17-23`): AI agents (impact analysis, safe edits), developers (local exploration/visualization), repo maintainers (stale-index detection, generated agent instructions, wikis), and multi-repo teams (cross-service contract analysis).

## Short description of main functionality

The primary interface is a CLI plus an MCP server (`README.md:97-228`). `gitnexus analyze` indexes a repo — "indexes the codebase, installs agent skills, registers Claude Code hooks, and creates `AGENTS.md`/`CLAUDE.md` context files — all in one command" (`README.md:108`). `gitnexus setup` writes MCP config for detected editors; `gitnexus mcp` starts a stdio MCP server serving all indexed repos; `gitnexus serve` starts an HTTP server (port 4747) for the browser UI. Other commands: `list`, `status`, `clean`, `wiki` (LLM-generated docs from the graph), and `group` subcommands for multi-repo analysis.

Agents get **17 MCP tools** — the registry in `gitnexus/src/mcp/tools.ts` defines `list_repos`, `query`, `cypher`, `context`, `detect_changes`, `check`, `rename`, `impact`, `explain`, `pdg_query`, `route_map`, `tool_map`, `shape_check`, `api_impact`, `group_list`, `group_sync`, and `trace`. The most important:

- `query` — process-grouped hybrid BM25 + semantic search with Reciprocal Rank Fusion.
- `context` — 360-degree symbol view (incoming/outgoing calls, process participation).
- `impact` — blast-radius analysis (upstream/downstream) with confidence and risk grouping.
- `detect_changes` — maps git diffs to affected symbols/processes.
- `trace` — walks execution paths / call chains through the graph.
- `rename` — graph-assisted multi-file rename with `dry_run`.
- `explain` / `pdg_query` — natural-language symbol explanation and program-dependence-graph queries.
- `cypher` — raw graph queries; plus route/API/tool-mapping (`route_map`, `tool_map`, `api_impact`, `shape_check`) and the two cross-repo group tools `group_list` / `group_sync`.

**MCP resources** (`gitnexus://repo/{name}/context|clusters|processes|process/{name}|schema`) give instant read-only context. Four bundled **agent skills** (Exploring, Debugging, Impact Analysis, Refactoring) plus `--skills`-generated per-module skills are installed to `.claude/skills/`. A **browser web UI** (React/Vite) offers a visual graph explorer and AI chat, running either fully in-browser (WASM) or bridged to a local `gitnexus serve` backend. Docker Compose ships two Cosign-signed images (CLI/server + web UI).

## How it works under the hood

**Monorepo** (`ARCHITECTURE.md:5-14`): `gitnexus/` (npm package: CLI, MCP server, HTTP API, ingestion, graph, embeddings), `gitnexus-web/` (thin React client), `gitnexus-shared/` (shared TS types/constants).

**Indexing pipeline.** `analyze.ts` → `runFullAnalysis` (`run-analyze.ts`) → `runPipelineFromRepo` (`pipeline.ts`) runs a **14-phase DAG** (`ARCHITECTURE.md:83`): `scan → structure → [markdown, cobol] → parse → [routes, tools, orm] → crossFile → scopeResolution → pruneLocalSymbols → mro → communities → processes`. Parsing uses **Tree-sitter** native bindings (per-language S-expression queries producing *unified capture tags* like `@definition.class`, `@call.name`), covering 14–16 languages. Cross-file resolution handles imports, calls, heritage, MRO (C3/first-wins), and constructor/`self`/`this` receiver inference. The dedicated **`scopeResolution`** phase (`scope-resolution/pipeline/phase.ts`) performs registry-primary binding/reference and inheritance resolution for migrated languages (Python, C#), gated by `MIGRATED_LANGUAGES` with CI parity checks; **`pruneLocalSymbols`** then drops inert block-local `Const`/`Variable`/`Static` nodes left with only a `File→DEFINES` edge (`ARCHITECTURE.md:213-343`).

**Storage** (`ARCHITECTURE.md:443-455`): the graph loads into **LadybugDB**, an embedded graph database with vector support (formerly KuzuDB, `@ladybugdb/core` ^0.16.1). Persistence lives per-repo in `<repo>/.gitnexus/`: `lbug` (DB), `lbug.wal`, `lbug.lock`, `meta.json` (lastCommit, indexedAt, stats). A global `~/.gitnexus/registry.json` lets one MCP server discover all indexed repos (`repo-manager.ts`). Loading uses CSV streaming (`csv-generator.ts`, `lbug-adapter.ts`).

**Query layer** — three interfaces to the same backend (`ARCHITECTURE.md:22-27`): MCP stdio (`mcp.ts` → `LocalBackend` → `tools.ts`/`resources.ts`), HTTP bridge (`serve.ts` → Express `api.ts`/`mcp-http.ts`), and CLI-direct. Connection pooling opens LadybugDB lazily and evicts after 5 min idle, max 5 concurrent (`README.md:335`).

**Tech stack** (`README.md:731-745`): Node.js native runtime; Tree-sitter parsing; LadybugDB storage; **HuggingFace transformers.js** embeddings (Snowflake `arctic-embed-xs`, 384-dim); BM25 + semantic + RRF search; MCP SDK; Express; Graphology (clustering/Leiden); worker threads for concurrency. The web mirror runs Tree-sitter WASM, LadybugDB WASM, in-browser embeddings, Sigma.js/Graphology visualization, and a LangChain ReAct agent.

## How memory about a codebase under development is implemented

**What is stored as memory.** The memory is a **knowledge graph**. The LadybugDB schema (`lbug/schema.ts`) defines ~30 node tables (File, Folder, Function, Class, Interface, Method, Constructor, Struct, Enum, Community, Process, Route, Tool, Section, Embedding, plus per-language types) and a *single* `CodeRelation` edge table carrying `type`, `confidence`, `reason`, and `step` properties (`schema.ts:229-438`). Relation types include CONTAINS, DEFINES, CALLS, IMPORTS, EXTENDS, IMPLEMENTS, HAS_METHOD, ACCESSES, METHOD_OVERRIDES, MEMBER_OF, STEP_IN_PROCESS, HANDLES_ROUTE, HANDLES_TOOL. Node rows store the actual `content`, `startLine`/`endLine`, and `description`.

This is primarily **factual/semantic memory** — durable, objective facts about code structure. Two derived layers add higher-order memory: **Community** nodes (Leiden clustering, with `cohesion`, `keywords`, `heuristicLabel`) group symbols into functional areas, and **Process** nodes trace execution flows across communities (`STEP_IN_PROCESS` edges with ordered `step` indices). These precomputed clusters and flows are the "relational intelligence" that lets one query answer what would otherwise take many.

**Form: token-level, not parametric.** GitNexus memory is entirely **token-level / externalized** — nothing is baked into model weights. It is a queryable graph plus a **vector store**: a separate `Embedding` table holds `FLOAT[384]` vectors with an HNSW cosine index (`schema.ts:463-481`), embedding File/Function/Class/Method/Interface nodes. Search (`hybrid-search.ts`) fuses BM25 keyword ranking and semantic vector ranking via Reciprocal Rank Fusion (K=60). The codebase's memory therefore spans a graph DB, a vector store, and content fields — all under `.gitnexus/`.

**How it is written / updated (lifecycle).** Creation happens on `gitnexus analyze` via the 14-phase pipeline. **Evolution/invalidation** is git-commit-driven, but *not yet a true incremental graph index*: `runFullAnalysis` early-exits when `meta.json.lastCommit == HEAD` unless `--force`, so an unchanged commit skips re-analysis entirely — but any re-analyze **rebuilds the whole graph** from scratch. Only the *embeddings* are incremental: they are cached and restored across runs, keyed by a **SHA1 content hash** (`contentHashForNode`, `embedding-pipeline.ts:87`), so just the stale nodes are re-embedded ("Incremental embeddings: N total, M cached, K stale"). Staleness detection is explicit: `git-staleness.ts` runs `git rev-list --count lastCommit..HEAD` and surfaces "Index is N commits behind HEAD" hints through MCP resources; it even detects *sibling-clone drift* by matching `remoteUrl` across on-disk clones (`git-staleness.ts:116-187`). A **Claude Code PostToolUse hook** (`gitnexus-claude-plugin/hooks/gitnexus-hook.js:254-305`) fires after `git commit/merge/rebase/pull`, compares HEAD to `meta.json.lastCommit`, and tells the agent to re-run analyze — closing the invalidation loop without blocking on a full reindex.

**How it is retrieved / injected — reducing `find`/`rg`.** Three injection mechanisms:

1. **Working-memory injection at edit time.** A **PreToolUse hook** intercepts `Grep`/`Glob`/`Bash rg|grep` calls, extracts the search pattern, and shells out to `gitnexus augment <pattern>` (`gitnexus-hook.js:217-243`). The augmentation engine (`augmentation/engine.ts`) runs BM25 (no embeddings, for <500 ms cold start), maps hits to graph symbols, then batch-fetches callers, callees, process participation, and cluster cohesion, returning a compact block such as `validateUser (src/auth/validate.ts) / Called by: handleLogin, handleRegister / Flows: LoginFlow (step 2/7)`. This is injected as `additionalContext` *before the grep even runs*, directly substituting graph edges for repeated grep chains.
2. **On-demand tool calls.** `query`, `context`, `impact`, `detect_changes` return precomputed, categorized results in one call.
3. **Persistent instructions as always-on priming.** `analyze` writes a delimited (`<!-- gitnexus:start/end -->`) block into `AGENTS.md`/`CLAUDE.md` (`cli/ai-context.ts`) with RFC-2119 imperatives ("MUST run impact analysis before editing any symbol," "NEVER rename with find-and-replace — use `gitnexus_rename`"), plus skill files under `.claude/skills/`. These sit permanently in the agent's system context, steering it toward graph memory over grep.

**Cross-repo memory.** Groups extend memory across services: `group sync` extracts provider/consumer contracts into a `contracts.json` **Contract Registry** and a bridge graph (`group/storage.ts:18`, `service.ts:148`), letting `impact` fan out across repo boundaries via the Contract Bridge (`cross-impact.ts`). This is a concrete instance of the survey's *shared memory for multi-agent / multi-service systems* (§7.5) — a common structural substrate spanning otherwise isolated repos.

**Caveats grounded in the code.** The augment fast-path deliberately uses BM25 only and hides cluster scores (`engine.ts:11-14`); memory freshness is commit-granular, so uncommitted working-tree edits are not reflected until re-analyze; incremental *indexing* is still on the roadmap ("Only re-index changed files," `README.md:754`) — today embeddings are incremental but the graph is rebuilt each analyze. All memory is local and gitignored (`README.md:769`), and no *experiential* memory of past agent sessions is retained — GitNexus stores purely factual codebase memory.
