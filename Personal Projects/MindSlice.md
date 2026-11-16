---
sort: 1
---

# MindSlice: Designing a Layered Memory Engine for AI Agents

## Background

Recently, in my spare time, I have been building an independent open-source project, [MindSlice](https://github.com/pancerZH/MindSlice), along with a few companion tools around it. The motivation came from a recurring pain point when using AI coding agents across personal side projects.

Most existing agent memory setups fall into one of two extremes:

1. **A single flat `MEMORY.md` file**: easy to read and edit, but everything lives in one bucket. Notes from one side project leak into another, outdated facts stay around next to newer ones, and sharing part of a knowledge base means manually copying lines around.
2. **Full-stack memory services**: backed by vector databases and internal LLM summarization pipelines. They are powerful, but the storage format becomes opaque, behavior is non-deterministic, and the engine quietly makes policy decisions about what is "true" behind the scenes.

In my view, long-term memory for agents needs a clean **filesystem-like layer** in the middle: human-readable files on disk, strict isolation boundaries between shared baselines and runtime learnings, and explicit mechanisms for knowledge to evolve over time. That is what MindSlice is designed to be.

## Core Design Principles

Before writing code, I settled on four rules for the engine:

- **Mechanism separated from policy**: There are **no LLM API calls**, no external database daemons, and no hardcoded truth heuristics inside the engine. MindSlice provides addressing, layered storage, lineage resolution, lexical search, and transactional staging; judgment stays in prompts, skills, and caller-defined trust rules.
- **Plain Markdown and TOML as the single source of truth**: Every card and manifest is a plain `.md` or `.toml` file on disk that can be tracked with Git or edited in any IDE. The local SQLite database (`.cache/index.sqlite`) is strictly a disposable cache that can be rebuilt from disk at any time.
- **Immutable baselines, isolated growth**: Mounted memory *slices* are read-only at runtime. All new learnings go into a writable `@overlay` layer (or `staging/` when human approval is required).
- **Zero runtime dependencies**: The engine is built exclusively on the Python 3.12+ standard library (`sqlite3`, `tomllib`, `pathlib`, `hashlib`, `fcntl`, `unicodedata`), and exposes the same workspace through a Python API, the `mindslice` CLI, and a 4-tool MCP server.

## Engine Architecture

### 1. Workspace, Slices, and Mounts

In MindSlice, **a directory is a slice**. There is no custom binary archive format. A workspace simply coordinates read-only mounted `.slice/` folders, a writable `overlay/`, a `staging/` buffer, and engine-owned history/cache directories:

```text
workspace/
├── mounts.toml                 # @alias, path, precedence, and provenance role per mount
├── slices/
│   ├── core-prefs.slice/       # Precedence: 10  | @prefs   | role = "baseline"
│   └── project-kb.slice/       # Precedence: 50  | @project | role = "curated"
├── overlay/                    # Precedence: 100 | @overlay | role = "runtime" (writable)
├── staging/                    # Uncommitted edits & base_sha256 preconditions
├── .history/changes.jsonl      # Engine-owned write reasons and lifecycle events
└── .cache/
    ├── index.sqlite            # Disposable FTS5 + link graph cache
    └── workspace.lock          # Reader/writer lock (fcntl.flock)
```

One deliberate detail in `mounts.toml` is **decoupling `precedence` from authority (`role`)**. `precedence` only decides which mount wins when a caller passes an unqualified path (such as `read("cli/output-format")`) and breaks exact BM25 score ties—it never inflates search scores or forces an agent to trust one card over another. Meanwhile, `role` (e.g., `"curated"` vs. `"runtime"`) is returned as provenance metadata so callers can enforce their own trust policies.

To play nicely with manual editor saves and concurrent processes, every operation coordinates through `fcntl.flock` (`LOCK_SH` for reads, `LOCK_EX` for writes). Before serving a query, the engine replays any interrupted `staging/.apply_journal.json` transaction and checks `(mtime_ns, size)` on mounted files to incrementally re-index normal editor changes on the fly.

### 2. Memory Cards and Scoped Addressing

Each `.md` file represents **one focused sub-topic** (typically 5–15 atomic bullet statements). Because Python 3.12 ships `tomllib` in the standard library, cards use `+++` TOML frontmatter instead of pulling in a third-party YAML parser (though plain Markdown and flat legacy YAML files are still indexed as-is without conversion):

```markdown
+++
title = "CLI JSON Output Conventions"
aliases = ["json-flag", "structured-output"]
tags = ["cli", "design"]
extends = []
supersedes = ["@project/cli/legacy-text-output"]
+++

All inspection subcommands support `--json` with deterministic keys and non-zero exit codes on error.

- Emit structured diagnostics to stdout under `--json` and keep human-only hints on stderr.
- Pagination fields follow the conventions in [[cli/pagination]].
```

Addressing inside cards follows a strict scoping rule:

- **Qualified references** (`@project/cli/legacy-text-output`) bind to a named mount.
- **Unqualified references** (`[[cli/pagination]]`) inside a card resolve **strictly within that card's own slice**, never through global workspace `precedence`.

This rule turned out to be essential: once a slice is packaged and shared, an unrelated card with the same relative path in `@overlay` can never accidentally hijack the slice's internal links or lineage.

### 3. Three-Stage Retrieval Funnel

Instead of dumping whole folders into an agent's context window, the MCP server exposes only three read primitives (`manifest`, `search`, `read`) plus `write`, each with bounded output sizes and pagination:

```text
manifest()  -->  mount map, card counts, top-level folders, and always-loaded cards
    |
search(q)   -->  top-k 240-char card previews, 1-hop [[neighbors]], coverage, unmatched_terms
    |
read(addr)  -->  full card body + composed baselines/extensions + backlinks + sha256 token
```

For `search()`, MindSlice uses SQLite FTS5 BM25 weighted across fields (`title: 10`, `aliases: 5`, `tags: 3`, `summary: 2`, `body: 1`). Because standard FTS5 tokenizers struggle with unspaced languages without native extensions, MindSlice normalizes text with NFKC and rewrites unspaced scripts (such as Chinese, Japanese, and Korean) into grapheme unigrams plus overlapping bigrams in pure Python before indexing. Every search response also reports `unmatched_terms`, giving the agent a clear signal to rephrase when a lexical query misses.

### 4. Explicit Lineage: `supersedes` and `extends`

Handling changing facts is where flat note systems usually break down: old notes still match keywords and compete with updated ones. MindSlice models change with two explicit directed edges:

- **`supersedes` (replacement)**: When card `B` supersedes `A`, `A` remains in the FTS5 index as a retrieval route. If an agent searches using old terminology from `A`, `search()` matches through `A` and redirects the hit to `B` (`matched_via_superseded`). Callers that do not trust runtime writes to override curated slices can pass `follow_supersedes=["curated"]`, which stops automatic forwarding while still surfacing untrusted overrides in metadata.
- **`extends` (refinement without shadowing)**: When `E` extends baseline `B` (for example, adding a local environment exception), `read(E)` returns `B` and `E` composed together rather than letting `E` shadow `B`. Crucially, extensions are **never silently rebased**: if `B` is later superseded by `R`, reading `E` or `R` flags `baseline_superseded_by` / `unrebased_extensions` so the agent can inspect whether the exception still holds.

Whenever a `write()` supersedes or deletes a card, the engine returns an **inbound impact report** (`affected_backlinks` and `affected_extensions`) under the same write lock. Again, the engine does not rewrite dependent cards on its own—it reports exact impact and lets the agent or maintainer decide how to update them.

### 5. Boot Set, Approval Staging, and `bake`

Not all memory is query-driven. Cards marked `load = "always"` (such as coding preferences or standing communication rules) form the **boot set**, delivered automatically at session start.

Because boot-set cards shape every future turn, any agent write that creates, modifies, extends, or supersedes a card in the boot-set footprint is **automatically routed to `staging/`** instead of live `@overlay` memory. Staged files stay invisible to live `search()` and `read()`, and `apply` / `discard` are CLI-only commands—an agent over MCP cannot self-approve changes to its own standing instructions.

For normal knowledge evolution, editing a card that lives in a read-only slice performs **copy-on-write** into `@overlay` and records a `supersedes` edge to the original (`forked_from`). Once `@overlay` accumulates verified learnings, `mindslice bake` packages selected cards into a new versioned `.slice` directory (`0.1.0` -> `0.1.1`), rewrites internal references to slice-relative form, atomically switches `mounts.toml`, and keeps the previous slice directory intact for instant `rollback`.

## Companion Tools Around the Engine

Keeping the core engine small and policy-free made it straightforward to build specialized tools on top without complicating the storage layer:

- **[MindSlice-plugin](https://github.com/pancerZH/MindSlice-plugin)**: Packages the pinned MindSlice engine, the 4-tool MCP server, review/maintenance skills, and `SessionStart` hooks into native plugins for Claude Code, Antigravity, and Codex.
- **[MindSlice-forge](https://github.com/pancerZH/MindSlice-forge)**: A zero-dependency CLI pipeline (`profile -> plan -> split -> extract -> finalize`) that drives any unattended agent harness via a simple `--harness-cmd` contract to distill large raw documents into validated slices, backed by per-step Git checkpoints and automated repair loops.
- **[MindSlice-flow](https://github.com/pancerZH/MindSlice-flow)**: A lightweight state-machine orchestrator for staged memory mounting, settlement, and baking. Each stage mounts its required background slices plus `load = "always"` director cards, lets the agent accumulate runtime state in `@overlay`, and settles/bakes each completed stage into a cumulative `@story` slice—first validated on multi-act interactive tabletop RPG scripts.
- **[MindSlice-tui](https://github.com/pancerZH/MindSlice-tui)**: A read-only three-pane terminal browser built with Textual on top of MindSlice's Python API, making it easy to inspect slice trees, card contents, `extends`/`supersedes`/backlink graphs, and `.history/changes.jsonl` audit trails.

## Takeaways

Building and benchmarking MindSlice reinforced a few lessons for me:

1. **Explicit lineage beats heuristic recency**: On four compiled external memory benchmarks, tracking `supersedes` reduced cases where a stale fact outranked the current one on state questions from **62% down to 0 / 442**.
2. **Measure before keeping "clever" retrieval tricks**: Early on, I tried adding `did_you_mean` spelling suggestions and coverage-first result re-ranking. Deterministic retrieval benchmarks showed spelling suggestions misfired far more often than they helped, and coverage-first ordering lost `-0.026 nDCG@5` against plain BM25, while authored card `aliases` gained `+0.16 nDCG@5` on vocabulary-gap queries. Both heuristics were deleted.
3. **Deterministic engines make agent workflows testable**: Because the engine itself has no LLM calls or hidden state, every engine change after our frozen agent evaluation could be regression-checked by replaying **17,947 logged `write` calls** and **4,044 `search` pages** across 631 workspace snapshots and verifying identical digests.

There is still plenty of room to explore—especially around hybrid retrieval for heavy paraphrase queries where pure lexical search is weak—but keeping the foundation as plain files, strict mount boundaries, and explicit lineage has made both personal agent workflows and downstream tooling much cleaner to build.
