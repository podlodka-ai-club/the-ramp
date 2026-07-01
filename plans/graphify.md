# graphify (`shared-memory-graphify`)

| | |
|---|---|
| **Local path** | `~/git/shared-memory-graphify/` |
| **Distribution** | Open-source Python package `graphifyy` on PyPI (YC S26; commercial "always-on" product Penpax on top) |
| **Language / stack** | Python ≥3.10; NetworkX graph engine; tree-sitter (36 grammars); optional MCP/Neo4j/FalkorDB/Whisper/Leiden extras |
| **Storage** | A single `graph.json` (NetworkX node-link format) in `graphify-out/`; no graph DB and no embeddings required |
| **Primary interface** | `/graphify` skill/command + CLI + MCP server; installs into ~20+ agent hosts |

## What is the main purpose of project creation

graphify exists to give AI coding assistants a **persistent, queryable map of a codebase** so they stop re-discovering structure from scratch on every turn. The README pitch (`README.md:27`): *"Type `/graphify` in your AI coding assistant and it maps your entire project — code, docs, PDFs, images, videos — into a knowledge graph you can query instead of grepping through files."* `pyproject.toml` echoes it: *"turn any folder of code, docs, papers, images, or videos into a queryable knowledge graph."*

The problem it solves is **context inefficiency**. Agents burn tokens reading whole files and running `find`/`rg`/`grep` to reconstruct what a codebase does. graphify builds the map once, then serves scoped subgraphs. `docs/how-it-works.md` reports a measured **71.5× token reduction per query** on a 52-file mixed corpus versus reading raw files, while conceding the win is structural (not compression) on tiny corpora that already fit a context window.

The audience is **developers using AI coding agents** — the tool ships install adapters for ~20+ hosts (Claude Code, Codex, Cursor, Gemini CLI, Copilot, Aider, OpenCode, Kilo, Kiro, Devin, Google Antigravity, and more; `README.md:29`, `graphify/skill-*.md`). It is deliberately a **local development tool** with a code-only offline path requiring no API key (`SECURITY.md:23-25`).

## Short description of main functionality

The primary entry point is the `/graphify` skill/command. Core workflows (`README.md:279-307`, `graphify/skill.md`):

- **Build**: `/graphify .` runs the full pipeline and emits three artifacts in `graphify-out/`: `graph.html` (interactive pyvis view), `GRAPH_REPORT.md` (god nodes, surprising connections, suggested questions, confidence audit), and `graph.json` (the full graph).
- **Query** (the token-saving path): `graphify query "<question>"` returns a scoped subgraph; `graphify path "A" "B"` finds the shortest path between two concepts; `graphify explain "X"` explains a node.
- **Incremental maintenance**: `graphify update .` re-extracts only changed files (AST-only, no LLM cost); `graphify hook install` adds git post-commit/post-checkout hooks that auto-rebuild; `graphify watch` rebuilds on filesystem events.
- **Ingest external material**: `graphify add <url>` fetches an arXiv paper, webpage, tweet, or transcribes a YouTube/video into the corpus.
- **Exports**: Obsidian vault, agent-crawlable markdown wiki (`--wiki`), Mermaid call-flow HTML, SVG, GraphML (Gephi/yEd), and Cypher for **Neo4j / FalkorDB** (`--neo4j-push`, `--falkordb-push`).
- **Serve as MCP**: `python -m graphify.serve graph.json` exposes the graph over MCP (stdio or shared HTTP) with structured tools.
- **Work-memory feedback**: `graphify save-result` records how a Q&A turned out; `graphify reflect` distills those outcomes into a `LESSONS.md`.
- **PR intelligence**: `graphify prs` maps changed files to graph communities to estimate merge "blast radius."
- **Cross-project**: `graphify global add` merges per-repo graphs into `~/.graphify/global-graph.json`.

It handles 36 tree-sitter code grammars plus docs, PDFs, Office/Google files, images, and video (`README.md:244-261`). Code is parsed **locally with no API calls**; only docs/papers/images use an LLM.

## How it works under the hood

**Pipeline** (`ARCHITECTURE.md`): `detect() → extract() → build_graph() → cluster() → analyze() → report() → export()`. Each stage is one function communicating via plain dicts / NetworkX graphs, with no side effects outside `graphify-out/`.

**Tech stack.** Python ≥3.10; **NetworkX** as the in-memory graph engine and serialization format (node-link JSON); **tree-sitter** grammars (one per language) for AST parsing; NumPy/rapidfuzz for dedup; optional `mcp`+`starlette`/`uvicorn`, `neo4j`, `falkordb`, `pypdf`, `faster-whisper`, `graspologic` (Leiden), and provider SDKs (`openai`, `anthropic`, `boto3`) as extras (`pyproject.toml`). The package is flat under `graphify/` (~60 modules) plus a partially-migrated `graphify/extractors/` subpackage.

**Storage / format.** The canonical store is a single **`graph.json`** in NetworkX node-link format — no graph DB required. Confirmed from `worked/httpx/graph.json`: `{"directed", "multigraph", "graph", "nodes":[...], "links":[...]}`. Nodes carry `{id, label, file_type, source_file, source_location, community}`; edges carry `{relation, confidence, source_file, source_location, weight, _src, _tgt, source, target}`. Neo4j/FalkorDB are optional *push* targets, not the primary store. There is **no vector database and no embedding step** — `docs/how-it-works.md` is explicit: *"No embeddings needed… The graph structure is the similarity signal."*

**Node model** (from `extract.py` and graph samples): `file_type ∈ {code, document, paper, image, rationale, concept}`. AST kind is encoded by **label convention**, not a field: a file is `models.py`; a class is a bare label `DigestAuth`; a free function is `name()`; a method is `.name()`. IDs come from `ids.py` `make_id`/`normalize_id` (NFKC-normalize → non-word runs to `_` → casefold), prefixed with the **full repo-relative path stem** so same-named files in different dirs don't collide (a breaking change in 0.9.0, `CHANGELOG.md`).

**Edge model.** Observed relations include `contains`, `method`, `calls`, `imports`, `imports_from`, `inherits`, `implements`, `extends`, `mixes_in`, `uses`, `references`, `instantiates`, `re_exports`, and `rationale_for`. Every edge is tagged **`EXTRACTED` / `INFERRED` / `AMBIGUOUS`** — an honesty audit trail. Same-file calls resolve to `EXTRACTED` (score 1.0); the cross-file second pass emits `EXTRACTED` when import evidence disambiguates, else `INFERRED` (0.8). Hyperedges (3+ node group relations) live in `G.graph["hyperedges"]`.

**Parsing/indexing pipeline** (`extract.py`, `detect.py`, `cache.py`): `detect()` walks the tree (respecting `.gitignore` merged with `.graphifyignore`, `followlinks=False`), classifies each file, and prunes noise dirs. `extract()` is two-pass: per-file tree-sitter extraction (parallelized with `ProcessPoolExecutor`), then cross-file import/call resolution. Code AST extraction never calls an LLM; docs/papers/images go through a **separate semantic pass** — the host agent dispatches parallel subagents (or a configured backend like Gemini) that emit JSON node/edge fragments. A **content-addressed SHA256 cache** (`cache.py`: `SHA256(content + \x00 + rel-path)`) skips unchanged files; the AST cache is version-keyed, while the semantic (LLM) cache is deliberately unversioned to avoid re-billing tokens.

**Clustering/analysis.** `cluster()` runs **Leiden/Louvain** community detection over graph structure (`PYTHONHASHSEED=0` pinned for reproducibility). `analyze.py` computes **god nodes** by degree centrality (filtering file-hubs and stdlib noise), **surprising connections** via a composite cross-file/cross-community/confidence score, and **suggested questions** from AMBIGUOUS edges, high-betweenness bridge nodes, and low-cohesion communities.

**Integration points.** (1) an **MCP server** (`serve.py`) over stdio or shared HTTP exposing `query_graph`, `get_node`, `get_neighbors`, `get_community`, `god_nodes`, `graph_stats`, `shortest_path`, plus PR tools and read-only resources (`graphify://report`), with hot-reload on `graph.json` mtime change; (2) a **CLI** (`graphify query/path/explain`); (3) **always-on host instructions and hooks** — `graphify claude install` writes a `CLAUDE.md`/`AGENTS.md` section and a **PreToolUse hook** that fires before Read/Glob/Grep/Bash-search and injects `additionalContext` nudging the agent to run `graphify query` first (`__main__.py:404-453`).

## How memory about a codebase under development is implemented

graphify implements **three distinct memory layers**, mapping cleanly onto the survey's factual / experiential / working taxonomy. All of it is **token-level, external, non-parametric** — the graph is JSON the agent reads, never weights it is trained into, and there is no latent/embedding memory (community structure *is* the similarity signal).

### 1. Factual / structural memory — the code graph

The core "memory" is the **`graph.json` code graph**: *what exists and how it connects*.

- **What is stored**: nodes for files, classes, functions, methods, imports, doc concepts; edges for `contains`, `calls`, `imports`/`imports_from`, `inherits`/`implements`, `uses`, `references`, `method`, `rationale_for`. Each node keeps `source_file` + `source_location` (e.g. `"L54"`), so the agent can cite exact locations without opening the file. Confirmed in `worked/httpx/graph.json` (144 nodes / 330 edges) and `worked/rsl-siege-manager/graph.json` (1886 nodes / 3876 edges; relation histogram `contains 1266, calls 1142, rationale_for 600, imports 312, uses 240, imports_from 196, inherits 64, method 56`).
- **The "why" sub-layer (design rationale)**: `extract.py:4204` defines `_RATIONALE_PREFIXES = ("# NOTE:", "# IMPORTANT:", "# HACK:", "# WHY:", "# RATIONALE:", "# TODO:", "# FIXME:")`. `_extract_python_rationale` (`extract.py:4228`) turns these comments **and docstrings** into standalone `file_type:"rationale"` nodes linked by `rationale_for` edges. In the rsl worked graph, **618 rationale nodes / 600 `rationale_for` edges** — the codebase's captured intent becomes first-class memory (the closest thing to *episodic memory of design decisions*).
- **How it's built/updated**: full build via the skill pipeline, or incremental via `build_merge` (`build.py:622`) where re-extracted files replace their prior contribution (stale nodes/edges for a changed `source_file` are dropped before merge; deleted files pruned via `prune_sources`).
- **How it's retrieved and injected**: `serve.py` `query_graph` runs an IDF-weighted, trigram-prefiltered node search (`_score_nodes`), picks up to 3 seeds (`_pick_seeds`), then does **BFS or DFS** (`_bfs`/`_dfs`) that *refuses to expand through hub nodes* (p99-degree threshold) to keep the frontier tight, and serializes the subgraph under a **token budget** (`_subgraph_to_text`, ~3 chars/token, default 2000). Results are injected as `NODE … [src=… loc=…]` / `EDGE u --relation [confidence]--> v` lines. Every label passes `sanitize_label` (prompt-injection defense).
- **How it reduces `find`/`rg`**: the PreToolUse hooks emit `MANDATORY: … run graphify query before grepping raw files` (`__main__.py:417`), and the always-on instruction files (`graphify/always_on/*.md`) tell the agent a scoped subgraph is *"usually much smaller than GRAPH_REPORT.md or raw grep output."*
- **Lifecycle / invalidation**: creation on first `/graphify`; evolution via `graphify update`, the git **post-commit/post-checkout hooks** (`hooks.py`) that run an incremental AST rebuild, and `watch.py`. Freshness is tracked via a `built_at_commit` field in `graph.json` (confirmed: `6085fd66…`) surfaced in `GRAPH_REPORT.md` ("Built from commit"). A SHA256 cache invalidates per-file on content change; concurrent commits are serialized by an advisory flock and a `.pending_changes` queue. A shrink-guard (`#479`) refuses to silently overwrite a larger graph.

### 2. Experiential / work memory — Q&A outcomes and reflections

This is graphify's explicit *experiential memory* — the system learns from **what you ask**, not just what's in the code. Docstring in `ingest.py:286`: *"This closes the feedback loop: the system grows smarter from both what you add AND what you ask."*

- **Capture**: `graphify save-result` (`ingest.py:save_query_result`, line 274) writes a markdown doc to `graphify-out/memory/query_<timestamp>_<slug>.md` with YAML frontmatter (`type, date, question, contributor, outcome, correction, source_nodes` capped at 10). Outcomes are constrained: `OUTCOMES = ("useful", "dead_end", "corrected")` (`ingest.py:271`). The outcome/correction are written to **both** frontmatter (for deterministic aggregation) **and** an `## Outcome` body section (so the signal round-trips back into the *graph* on the next semantic re-extraction).
- **Distillation**: `graphify reflect` (`reflect.py`) aggregates these docs into `graphify-out/reflections/LESSONS.md`, **deterministically, with no LLM**. Each source-node citation contributes a **signed, time-decayed score** (`useful` +1, `dead_end`/`corrected` −1, 30-day half-life), so a fresh dead end outweighs a months-old success. Nodes are bucketed into **Preferred** (≥2 corroborating useful results — "one save can't mint a trusted lesson"), **Tentative**, **Contested** (recency decides), **Known dead ends** ("don't re-derive"), and **Corrections**. With a graph in hand, lessons are grouped by community and stale nodes (deleted/renamed) are dropped.
- **Injection & lifecycle**: `LESSONS.md` is an orientation artifact an agent loads at session start. The git hooks call `reflect` after every rebuild, and `reflect --if-stale` skips redundant recomputation. It lives in `reflections/` precisely because the wiki export wipes `wiki/*.md` each run.
- Separately, **query logging** (`querylog.py`) appends every query to `~/.cache/graphify-queries.log` (JSONL) — passive, fail-silent telemetry with responses excluded by default.

### 3. Working / navigational memory — reports, wiki, MCP session state

The *working-memory* surface an agent uses mid-task: `GRAPH_REPORT.md` (god nodes, surprising connections, suggested questions, confidence spread — e.g. httpx "53% EXTRACTED · 47% INFERRED"), the `--wiki` export (`wiki.py`: `index.md` + per-community + per-god-node articles with an audit trail), and the MCP server's hot-reloaded in-memory graph plus cached trigram/IDF indexes per graph generation.

### Cross-session and cross-team persistence

`graph.json` and `graphify-out/` are meant to be **committed to git** so a whole team shares one map (`README.md:331`), with a `graphify merge-driver` CLI (union-merge via `networkx.compose`) preventing conflict markers — a concrete instance of the survey's *shared memory* (§7.5). Cross-project memory merges per-repo graphs into `~/.graphify/global-graph.json`.

### Observed gaps / caveats

- Per-node natural-language summaries are only an **RFC** (`docs/node-summaries-rfc.md`), not implemented — agents may still open a file to learn a node's responsibility.
- AST kind lives in the label convention, not a dedicated `kind` field, so consumers infer file/class/function/method from label suffixes.
- The `graphify/extractors/` package is mid-migration (`extractors/MIGRATION.md`); most language dispatch still runs through `extract.py`'s `_DISPATCH`.
- No `memory/`, `LESSONS.md`, or populated `reflections/` exist in the committed `worked/` examples — the experiential layer is exercised via tests (`tests/test_reflect.py`, `tests/test_rationale.py`), not shipped sample data.
