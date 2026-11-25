# CLAUDE.md — Crazy Research Wall

Read this first. This is the standing brief for Claude Code and other agents working in this repo.

The model below is the **current working summary**. Canonical Cypher (constraints, indexes, and the frozen property set) will live in `packages/cypher/migrations/` once it is checked in. If this file and a migration disagree, the migration wins — update this document and `docs/DECISIONS.md` rather than inventing a parallel schema. When the schema is ambiguous, **ask rather than guess**.

## Purpose

**Crazy Research Wall** is a Neo4j-powered, Maltego-inspired node-based link analysis and evidence corroboration tool for cold-case investigations. Investigators pin entities on a wall, relate them, attach evidence, accept or reject AI-suggested links, and run graph analytics (contradictions, link prediction, community detection).

Every assertion on the wall must be traceable to Evidence, with a source and a confidence or reliability score.

## Core data model (working summary)

Canonical Cypher will live in `packages/cypher/migrations/`. Property names here are the working contract (camelCase) and may be remapped if the official model uses a different convention.

```cypher
(:Case {caseId, title, status, summary, createdAt, updatedAt})

(:Node {id, caseId, type, label, properties, status, createdAt, updatedAt})

(:Node)-[:RELATED_TO {
  id, type, status, confidence, caseId, createdBy, createdAt
}]->(:Node)

(:Evidence {
  id, caseId, claim, sourceTitle, sourceUrl, sourceDate, sourceType, reliability
})
```

- `:Case` is the isolation boundary. `caseId` is the public key (e.g. `atadero-1999`).
- `:Node` is a generic entity. `type` distinguishes kinds (`Person`, `Location`, `Vehicle`, `Organization`, `Event`, `Document`, `Item`, … — extend only when needed and document it). `label` is the display name. Extra attributes go in `properties`, not as unsourced top-level claims.
- `:RELATED_TO` is the investigation link. `type` is a controlled vocabulary (`SEEN_AT`, `KNOWS`, `MENTIONED_IN`, …), not a new relationship type per kind. `confidence` is 0.0–1.0. `createdBy` is `investigator` | `ai` | `analytics`.
- **Status enum (working):** `suggested` | `confirmed` | `disputed` | `rejected`. AI and analytics output starts as `suggested` only. Rejected and disputed links are retained for audit; do not delete them to "clean up" the graph.
- `:Evidence` is a sourced claim or artifact. `reliability` is 0.0–1.0.

### `SUPPORTED_BY` and relationship-level evidence (TBD)

`:SUPPORTED_BY` ties a node or a relationship-assertion to `:Evidence`. Standard Neo4j property graphs **cannot** point a relationship at a relationship, so "this `RELATED_TO` is supported by that Evidence" needs an explicit pattern. Two candidates:

1. **Reify the assertion** (e.g. `(:Node)-[:LINK_FROM]->(:Assertion)-[:LINK_TO]->(:Node)` and `(:Assertion)-[:SUPPORTED_BY]->(:Evidence)`), with `RELATED_TO` either dropped or kept as a denormalized edge.
2. **Store the relationship id on the support edge or a support node**, e.g. `(:Support {targetKind, targetId, caseId})-[:SUPPORTED_BY]->(:Evidence)`, or a `relationshipId` property on a case-scoped support structure.

**TBD:** the exact choice is pending the canonical model. Do not bury support only in id arrays on the relationship. Entity-level support is `(:Node)-[:SUPPORTED_BY]->(:Evidence)` either way. Every `RELATED_TO` still needs a stable `id` so evidence can attach.

Conflicts are first-class: prefer `(:Evidence)-[:CONTRADICTS]->(:Evidence)` (and/or status flags such as `disputed`) over deleting or silently overwriting a claim.

## Hard rules

These are non-negotiable.

- **Parameterized Cypher only.** Never string-concatenate or interpolate user input into Cypher text. Queries live in `packages/cypher` and are called as named functions from the API.
- **Every query filters by `caseId`.** MATCH, CREATE, MERGE, DELETE, and GDS projections are case-scoped. No cross-case leakage.
- **AI-suggested links are never confirmed automatically.** Only an explicit human action moves `suggested` → `confirmed` (or `rejected` / `disputed`). Suggestions must store provenance: model, prompt/version, score, and timestamp (plus algorithm/run id for analytics).
- **Every claim needs evidence/provenance.** Confirmed links still need Evidence. If an investigator accepts a suggestion without evidence, surface that gap in the UI; do not silently treat it as sourced.
- **Represent conflicts explicitly.** Use `CONTRADICTS` between Evidence and/or status flags. Never delete a conflicting source to resolve a disagreement.
- **Never silently overwrite Evidence.** Updates are additive or explicit revisions with history; do not clobber `claim`, source fields, or reliability in place without a trace.
- **Sensitive media is flagged and not displayed by default** (e.g. remains / skull photos). The UI must require an intentional reveal.
- **No secrets in the repo.** Commit `.env.example` only. Credentials come from the environment (`NEO4J_URI`, `NEO4J_USER`, `NEO4J_PASSWORD`, etc.).
- The UI does not run Cypher. All business writes go through the API.

## Stack

TypeScript everywhere, strict mode.

| Layer | Guidance |
|-------|----------|
| Monorepo | pnpm workspaces |
| API | Node.js + official `neo4j-driver` + **Fastify** (or similar). Validate all input with **zod**. Prefer this over stuffing Neo4j calls into Next.js route handlers. |
| Web | Vite + React + TypeScript. Graph wall: **xyflow/React Flow** *or* **Sigma.js / graphology**. Decision is open — see tradeoff below. |
| Database | Neo4j 5 + Graph Data Science (GDS) plugin via Docker Compose. Use GDS Community algorithms. No Enterprise-only or APOC-only features unless a decision doc says so. |
| Shared types | `packages/shared` — entity, relationship, evidence, suggestion types. Do not duplicate across apps. |

**Graph viz tradeoff (open decision):** React Flow is better for rich, editable node cards and investigator UX at small/medium scale (typical case graphs: hundreds to low thousands of nodes). Sigma.js / graphology is better for large graphs and WebGL performance. Record the choice in `docs/DECISIONS.md` when it is made; do not swap libraries casually after scaffolding.

### Suggested layout

```
apps/web/                  React UI (graph wall, inspector, evidence, timeline, suggestion queue)
apps/api/                  Fastify API
packages/cypher/           migrations/, queries/, analytics/
packages/shared/           shared TypeScript types
seed/                      case seed data (see Seed data)
docker/                    docker-compose.yml for Neo4j 5 + GDS
docs/                      ARCHITECTURE, SCHEMA, BACKLOG, DECISIONS
```

Default Neo4j: `bolt://localhost:7687`, HTTP `7474`. Compose may ship a **dev-only** password; change it. Never commit real credentials.

## Algorithms of interest

Analytics run on **case-scoped projections** and write results as **suggestions**, never as confirmed facts. Persist provenance (algorithm, parameters, run id, timestamp, score).

- **AI-suggested links** — model proposes `RELATED_TO` with status `suggested` and full provenance.
- **Contradiction detection** — find conflicting Evidence (and disputed links); materialize `CONTRADICTS` or an equivalent review queue.
- **Topological link prediction** — common neighbors, Adamic-Adar, preferential attachment, via `gds.alpha.linkprediction` or equivalent plain Cypher.
- **GDS link prediction pipelines** — supervised / pipeline workflows when a case graph is rich enough; still written as suggestions.
- **Community detection** — Louvain and Leiden; communities are analytical overlays, not ground truth.

Document algorithm names and Community vs Enterprise requirements next to the query.

## Workflow

- **SuperBot** — orchestrator (scope, backlog, handoffs, standing context).
- **Research Analyst** — case research, evidence, source tracking, seed data.
- **Graph Specialist** — Neo4j schema, Cypher, algorithms, investigative questions as graph operations.
- **Claude Code** — primary coding agent for this repository.
- **Mystic** — human reviewer; reviews and merges.

Work on feature branches. Keep PRs small and focused. Use conventional commits (`feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, `test:`). Update `docs/DECISIONS.md` for any architectural choice (stack, schema, library, status machine). When the schema is ambiguous, ask rather than guess.

## Conventions

### Testing

- **vitest** for unit tests (query builders, status-machine rules, validators, seed parsers) — these should not need Neo4j.
- Integration tests against Dockerized Neo4j or **testcontainers** (Neo4j 5 + GDS). Never run destructive tests against a researcher's database.
- Cover at least: `caseId` isolation, parameterized queries, suggestion accept/reject (no auto-confirm), evidence attachment, migration apply, and a seed-import smoke test once seed files are in the repo.

### Migrations

Numbered, idempotent `.cypher` files in `packages/cypher/migrations/` (`001_*.cypher`, `002_*.cypher`, …). Constraints and indexes live here. Prefer `IF NOT EXISTS` / idempotent forms. No APOC-only dependencies unless noted in the migration and `docs/SCHEMA.md`.

### Lint, format, errors, logging

- Shared lint + format (eslint + prettier, or the repo standard once scaffolded) and `tsc` in CI.
- API errors are structured (`code`, `message`, `details`). Fail closed on missing `caseId` or invalid status transitions.
- Log request/operation metadata, not sensitive evidence text, personal data, or credentials. No PII leakage in logs.

### Seed data

Seed case: **Jaryd Atadero** (disappeared 2 Oct 1999, Big South Trail, Roosevelt National Forest, Colorado; remains recovered 2003). Sourced files live in `seed/atadero/` and are rebuilt/validated by `seed/atadero/scripts/build.py`. Do not invent case facts.

The Larimer County Sheriff's report is reserved as document **D002** and is pending; when it arrives, Research Analyst will update the seed. Do not fabricate D002.

Do not copy or commit seed data until it is deliberately added to this repo.

### Commands (TODO until scaffolded)

These are the intended scripts. Names may match package scripts once the monorepo exists. Until then, treat this list as a checklist, not a promise that the commands run.

```bash
# TODO: scaffold pnpm workspaces, then:
pnpm install
pnpm dev                      # API + web; assumes Neo4j is up
pnpm --filter api dev
pnpm --filter web dev
docker compose -f docker/docker-compose.yml up -d
pnpm migrate                  # apply packages/cypher/migrations/*.cypher in order
pnpm seed:atadero             # import seed/atadero/ (idempotent, case-scoped)
pnpm test
pnpm lint && pnpm typecheck
```

## Start here (Claude Code)

1. Read this file. Treat the data model as a working summary until Cypher lands in `packages/cypher/migrations/`. If canonical Cypher is provided, put it there and align docs — do not invent a parallel model.
2. Scaffold the pnpm workspace, `apps/api`, `apps/web`, `packages/cypher`, `packages/shared`, `.env.example`, and `docker/docker-compose.yml` with a healthcheck. `docker compose up -d` should yield Neo4j 5 with GDS (`RETURN gds.version()`).
3. Add constraints/indexes and named, parameterized, `caseId`-scoped queries for case, node, relationship, evidence, and the graph payload for the wall.
4. Resolve or explicitly defer the `SUPPORTED_BY` pattern (reify vs relationship id) in `docs/DECISIONS.md` before writing support-attachment code.
5. API: cases, graph-for-case, nodes, relationships, evidence, suggestion accept/reject (human-only confirm).
6. UI: graph wall, entity inspector, evidence panel, then timeline and a suggestion review queue. Sensitive media hidden by default.
7. Analytics endpoints + UI entry points: contradictions, link prediction, community detection — results written as suggestions.
8. Add an idempotent importer for `seed/atadero/` once those files are in the repo. Skip or clearly refuse if they are absent; do not fabricate case data.
9. Keep `docs/DECISIONS.md` updated when a stack or schema choice is closed.
