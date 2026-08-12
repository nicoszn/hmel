# hmel — Hierarchical Memory Engine (Multi-Agent LLM System)

A tiered memory architecture for multi-agent LLM systems: **working memory**, **episodic memory**, and **shared team memory**, each with a distinct scope, lifecycle, and storage mechanism — built as a lightweight, Vercel-hosted system.

## Architecture

Three agents — **Planner → Worker → Reviewer** — run sequentially via LangGraph.js, with a revision loop (Reviewer can send work back to Worker). Every memory read/write passes through a rule-based **Memory Router** (no LLM call) into one of three tiers, all backed by a single Neon Postgres database.

```
Next.js UI (API routes + SSE, no Server Actions)
        │
        ▼
LangGraph.js graph (Planner → Worker → Reviewer, sequential)
        │  reads/writes via Memory Router
        ▼
┌───────────────┬──────────────────────┬───────────────────────┐
│ Working memory│ Episodic memory      │ Team memory           │
│ in-process,    │ [agentId, taskType], │ explicit commits,     │
│ one request    │ hash-embeddings +    │ Reviewer arbitrates   │
│                │ pgvector             │ conflicts             │
└───────────────┴──────────────────────┴───────────────────────┘
        │
        ▼
Neon Postgres (single database)
```

## Memory tiers

| Tier | Scope | Storage |
|---|---|---|
| Working | One task run, discarded after | LangGraph `MemorySaver` (in-process) |
| Episodic | Per agent + task type | Neon Postgres, `pgvector` column, deterministic hash-based embeddings |
| Team | Whole agent collective, persistent | Neon Postgres, explicit commits only, conflicts resolved by Reviewer arbitration |

## Tech stack

| Purpose | Package |
|---|---|
| Orchestration | `@langchain/langgraph` |
| LLM provider | `@langchain/openrouter` (`OPENROUTER_MODEL` env-configured, no hardcoded default) |
| Database | Neon Postgres via `@neondatabase/serverless` |
| ORM | `drizzle-orm` (with `pgvector` support) |
| Validation | `zod` |
| Frontend | Next.js 16.2 (App Router, API Route Handlers, no Server Actions) |
| Language | TypeScript 6.0.3 |

No ML embedding model, no separate vector DB service, no test runner — kept intentionally minimal for Vercel hosting.

## Design notes

- **Embeddings** are deterministic feature-hashing vectors (FNV-1a hash + sign, 256-dim), computed in-process — no model download, no cold-start cost.
- **Similarity search** uses `pgvector` + Drizzle's `cosineDistance`, brute-force (no PQ/ADC — unnecessary at this scale).
- **Tool sandbox** (`write_file`, `run_tests` for the code-generation task) writes to `os.tmpdir()`, which is ephemeral per Vercel invocation — a natural fit since sandbox files only need to live as long as one task run.
- **Streaming** uses `ReadableStream` + a controller in a Route Handler (SSE), not Server Actions.
- All operations (start a run, browse memory, resolve a team-memory conflict) are driven from the UI — no CLI scripts.

## Environment variables

```
DATABASE_URL=
OPENROUTER_API_KEY=
OPENROUTER_MODEL=
SEARCH_API_URL=
SEARCH_API_KEY=
```

## Status

Architecture and technical design finalized. Implementation in progress.
