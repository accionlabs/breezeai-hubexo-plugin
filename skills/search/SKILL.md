---
name: search
description: >
  Search the functional, design, code, and architecture graphs to answer
  questions about the system. Smart-routes to one graph, a subset, or all of
  them based on query intent. Use for: feature discovery, "how does X work",
  "who handles Y", finding code or UI implementations, schema lookups, and
  cross-layer questions. Default entry point for any question about the project.
---

## Project

This skill needs a `projectUuid` — follow CLAUDE.md's project-resolution rules (`--project` override → `.breeze.json` fallback → announce active project header). Auth handling on Breeze MCP failures is also covered in CLAUDE.md.

---

## Tools Available

All tools below are read-only. Pass `projectUuid` (or `project_uuid` / `uuid`, per each tool's schema) on every call.

### Functional graph
- `Functional_Graph_Search` — semantic search across Persona / Outcome / Scenario / Step / Action.
- `Get_all_personas` — list every Persona with its ID.
- `Get_all_outcomes_for_a_persona_id` — outcomes under a Persona.
- `Get_all_scenarios_for_a_outcome_id` — scenarios under an Outcome.
- `Get_all_steps_actions_for_a_scenario_id` — full Step → Action tree under a Scenario.

### Code graph
- `Call_List_Repositories_` — list every indexed repository in the project (name, id, file/class/function counts, language, repo URL, commit, branch). Use first for "which codebases are indexed" questions, and to enable per-repo scoping on subsequent `Code_Graph_Search` calls.
- `Code_Graph_Search` — semantic search across File / Function / Class. Supports `repository_name=` to scope a search to one repo when the question clearly maps to a single subsystem (e.g., a frontend-only change → the `frontendweb_*` repo). When the touched code spans multiple repos, anchor each hit back to its parent repo from the inventory.
- `Get_Code_File_Details` — full hierarchical structure of a single file (classes, methods, decorators, statements).

### Design graph
- `Design_Graph_Search` — semantic search across design nodes (journeys, flows, pages, components).
- `Get_all_Design_By_Label` — list design nodes filtered by label type.
- `Get_Design_Nodes_by_Ids` — fetch specific design nodes by ID.

### Architecture + DB-schema graph
- `Architecture_Graph_Search` — semantic search across architecture-layer nodes (UserExperience, ApiGateway, ObservabilityMonitoring, Agents, Services, EventQueue, DataLake, Infrastructure) **AND** DDL nodes attached to each DataLake (DDLTable, DDLColumn, DDLConstraint, DDLIndex, DDLView, DDLSequence, DDLProcedure). Filter via `include_labels=[...]` and tune `threshold` (default ~0.4). **Embedding-filtered — silently drops nodes whose name/description doesn't match the query.** For *enumeration* questions ("list every service", "every table in DataLake X"), fall back to the by-label tools below — those don't filter.
- `Get_Architecture_Nodes_By_Label` — enumerate every architecture node in one layer (no semantic filter, no silent drops). Use for "list all services / queues / data stores".
- `Get_DB_Schema_Nodes_By_Label` — list DDL entities under one DataLake by label (`table`, `column`, `constraint`, `index`, `view`, `sequence`, `procedure`). Supports pagination, sort, equality/regex filters (e.g. `{"tableId": {"$eq": "<uuid>"}}`), parent/child tree walks. Use for "all tables in DataLake X", "FKs referencing table Y", "columns of table Z".

### Label routing for `Architecture_Graph_Search`

Pick `include_labels` by query intent — narrow when you can, omit when truly ambiguous:

| Query shape | `include_labels` |
|---|---|
| "which service handles X" | `["Services"]` |
| "what queue/topic carries X" | `["EventQueue"]` |
| "what data stores hold X" (store-level) | `["DataLake"]` |
| "what alerts on X" / "where do X logs go" | `["ObservabilityMonitoring"]` |
| "what compute / VPC / region runs X" | `["Infrastructure"]` |
| "which frontend / app exposes X" | `["UserExperience"]` |
| "which API / route / gateway handles X" | `["ApiGateway"]` |
| "which agent / LLM workflow runs X" | `["Agents"]` |
| "find tables for X" / "which tables store Y" | `["DDLTable"]` |
| "find columns named X" | `["DDLColumn"]` |
| "FKs referencing X" / "unique constraint on Y" | `["DDLConstraint"]` |
| "indexes on X" | `["DDLIndex"]` |
| "views / materialized views including X" | `["DDLView"]` |
| "stored procs / triggers doing X" | `["DDLProcedure"]` |
| "sequences for X" | `["DDLSequence"]` |
| "everything related to the auth schema" | `["DDLTable","DDLColumn","DDLConstraint","DDLView","DDLProcedure"]` |
| "what stores AND schema back X" | `["DataLake","DDLTable","DDLView"]` |
| Truly ambiguous one-word queries | omit `include_labels` |

When a question demands completeness ("are these ALL the services / tables?"), don't rely on this tool — use `Get_Architecture_Nodes_By_Label` or `Get_DB_Schema_Nodes_By_Label`.

---

## Phase 1 — Classify Query Scope

Inspect `$ARGUMENTS`. Decide **which graphs to search**. You can pick one, a subset, or all. The goal is to cover the layers the user's question actually touches — no more, no less.

### 1a. Single-graph fast-paths

Stop at the first match:

| Query shape | Route to |
|---|---|
| Pure code lookup — "where is X implemented", "find function/class/file Y", "show me the route handler for Z" | `Code_Graph_Search` → then `Get_Code_File_Details` on top hits. If the query names or implies a single repo, list repos via `Call_List_Repositories_` first and scope the search via `repository_name=`. |
| Repo inventory — "which codebases", "list the repos", "what languages does this project use", "what's the repo for X" | `Call_List_Repositories_` |
| Pure UI — "what page shows X", "which component", "what does the settings screen look like" | `Design_Graph_Search` → optionally `Get_Design_Nodes_by_Ids` for detail |
| Personas / roles only — "who manages X", "what roles exist" | `Get_all_personas` → drill down (see Phase 3) |
| Targeted deployment question — "which service handles X", "what queue carries Y", "what data store holds Z" | `Architecture_Graph_Search` with matching `include_labels` (see routing table above) |
| Enumerate one architecture layer — "list every service / queue / data store / topic" | `Get_Architecture_Nodes_By_Label` |
| Targeted DDL question — "find tables for X", "columns named Y", "FKs to Z", "views including W" | `Architecture_Graph_Search` with DDL `include_labels` |
| Enumerate schema under one DataLake — "all tables in DataLake X", "every view in Y" | `Get_DB_Schema_Nodes_By_Label` (find the DataLake's id first via `Architecture_Graph_Search` or `Get_Architecture_Nodes_By_Label`) |

### 1b. Multi-graph triggers

If the query doesn't fit a single-graph fast-path, pick the layers it implies and run them in parallel.

**Trigger words that add a layer:**
- *screen, page, component, UI, UX, flow, wireframe, design* → add **Design**
- *file, function, class, method, module, import* → add **Code**
- *repo, repository, codebase, language* → add **Code** (start with `Call_List_Repositories_`)
- *service, deploy, deployment, region, SPOF, queue, topic, bucket, cache, tech stack, infra* → add **Architecture**
- *table, column, schema, DDL, foreign key, FK, materialized view, procedure, trigger, constraint, sequence, index* → add **Architecture (DDL labels)**
- *persona, role, outcome, scenario, workflow, business logic* → add **Functional**

**Shape-based routing:**
- "how does X work" / "explain Y" / "what happens when…" / "walk me through Z" → **Functional + Code** minimum (add Design if UI-facing, add Architecture if deployment-facing, add DDL labels if data-flow involves specific tables)
- "what is the impact of…" / "what breaks if…" / "what does X depend on" → **Functional + Code + Architecture** (add Design if UI-facing) — but for full impact analysis prefer `/breeze:impact-analysis-v2`
- "compare X and Y" (where X, Y are features) → **Functional + Code**

**Default when truly ambiguous** → **Functional + Code**.

> Running all graphs is fine when the question is genuinely cross-cutting. Don't over-narrow out of caution, but don't fan out for a question that's obviously single-layer.

---

## Phase 2 — Execute in Parallel

Run every chosen semantic search / enumeration call simultaneously. Do not sequentialize.

Typical parallel batch:
- `Functional_Graph_Search`
- `Design_Graph_Search`
- `Code_Graph_Search` (optionally per-repo splits if the query maps to ≤2 repos from the `Call_List_Repositories_` inventory)
- `Architecture_Graph_Search` (with `include_labels` per the routing table — for cross-cutting queries you may issue 2 parallel calls, e.g. one architecture-layer set and one DDL-label set)

If a graph returns no results, note "No matches in {graph}" internally and continue — don't fail the whole search.

---

## Phase 3 — Drill Down

After the parallel reads, drill into top hits to build a complete picture. No magic relevance thresholds — just shape-based rules.

### Functional drill-down — top-down hierarchy

The functional graph is **Persona → Outcome → Scenario → Step → Action**. Drill along this chain as far as the question requires:

1. **Search hit is (or references) a Persona** → call `Get_all_outcomes_for_a_persona_id` to list its outcomes.
2. **Search hit is (or drills into) an Outcome** → call `Get_all_scenarios_for_a_outcome_id` to list its scenarios.
3. **Search hit is (or drills into) a Scenario** → call `Get_all_steps_actions_for_a_scenario_id` to get the full Step → Action tree.
4. **Search hit is an Action or Step** → identify the parent Scenario (via `scenarioId` in the result) and run `Get_all_steps_actions_for_a_scenario_id` on it to get the full flow.

For "how does X work" questions, drill all the way to Steps/Actions. For "who does X" or "what outcomes exist", stopping at Outcomes or Scenarios is usually enough.

### Code drill-down
- **Always anchor hits to their parent repo** from the `Call_List_Repositories_` inventory. When the touched code spans multiple repos, prefix file paths with the repo name (e.g., `frontendweb_react_tnlm: src/utils/posthog.ts`) so cross-repo coordination is visible at a glance.
- **Top hits are Files** → call `Get_Code_File_Details` on each to see classes, methods, decorators, statements.
- **Top hits are Functions or Classes** → their source and call chain are already in the search payload; only fetch the parent file if you need surrounding context.

### Design drill-down
- **Top hits are design nodes** → if the search payload is thin, call `Get_Design_Nodes_by_Ids` on the top IDs for full detail, or `Get_all_Design_By_Label` to widen within a label type.

### Architecture + DDL drill-down
- **Top hits are architecture nodes** → already self-contained (name, description, technologies, pattern, regions); use directly in synthesis.
- **Top hits are DDL nodes (DDLTable / DDLColumn / DDLView / DDLProcedure …)** → the `ddlText` (for tables) or `definition` (for views/procs) is typically in the search payload. For a complete schema picture under one DataLake — every column of a table, every FK pointing at it, every view that references it — call `Get_DB_Schema_Nodes_By_Label` with the matching `data_lake_id` and the appropriate label + `filters`.
- **Enumeration follow-up** — if the semantic search returned an incomplete set and the question demands completeness ("are these ALL the tables?"), fall back to `Get_Architecture_Nodes_By_Label` (one architecture layer) or `Get_DB_Schema_Nodes_By_Label` (schema-side, one DataLake).

### Cross-Persona Drill-Down

When drilling into a functional Outcome, check if the **same Outcome name exists under other Personas** (common for features that span UI and backend). After finding an Outcome under a User persona, also fetch it under the System persona (and vice versa) via `Get_all_outcomes_for_a_persona_id`, then `Get_all_scenarios_for_a_outcome_id` on the matching outcome. Merge both into the narrative.

---

## Phase 4 — Synthesize into One Narrative

Present a **single coherent answer**, not a per-graph ranked list.

### For behavior / "how does X work" questions
Build a sequential flow:
1. **Trigger** — what initiates it (UI event, cron, webhook, queue message).
2. **UI interaction** — from Design / functional User persona.
3. **Backend processing** — from functional System persona + Code (anchored to repo).
4. **Deployed path** — which services / queues / data stores / tables carry it (from Architecture + DDL), if architecture was in scope.
5. **Completion** — side effects, confirmation.

### For discovery questions
Provide a ranked list including:
- Entity type (Persona / Outcome / Scenario / Step / Action / Component / File / Function / Service / DataLake / DDLTable / DDLColumn / …)
- Name + one-line description (and parent repo for code entries, parent DataLake for DDL entries)
- Pointer to drill down further if useful

### Always when multiple layers fire
Weave them: *functional flow → code file/function (with repo) → deployed service/queue/data store → underlying table/view*. Show the connection; don't just list per layer.

---

## End-to-End Flow Questions

For any question shaped like *"what happens when…"*, *"explain the flow of…"*, *"how does X process work"*:

1. **Always** search **Functional + Code** at minimum.
2. Add **Design** if the flow starts at a UI action.
3. Add **Architecture** if the flow crosses services / queues / data stores (which is usually true). Add DDL labels too if the prompt names specific tables or asks about persistence shape.
4. In the functional graph, query **twice** — once with user-centric terms (UI actions, clicks, forms) and once with system-centric terms ("System processes…", "backend handles…", "External System…"). This captures both the trigger side and the processing side of the dual-persona model.
5. Drill the top functional hits all the way to Steps/Actions via the Phase 3 hierarchy chain.
6. Merge everything into one sequential trace (Phase 4).

A single search returns an incomplete picture for process questions — always fan out.

---

## Don'ts

- **Don't present raw search results as the final answer.** Always drill down and synthesize.
- **Don't claim a feature doesn't exist** from one failed search — try a rephrase or widen to another graph first.
- **Don't trust an empty `Architecture_Graph_Search` result for an enumeration question.** Embedding-filtered search silently drops nodes — if the user asked to enumerate (all services, all tables), use `Get_Architecture_Nodes_By_Label` / `Get_DB_Schema_Nodes_By_Label` instead.
- **Don't mention empty layers or graph-completeness gaps** in user-visible output (e.g., "the architecture graph has 0 Agents"). Silently fall back; present what you do have.
- **Don't duplicate `/breeze:impact-analysis-v2`.** `search` reads and synthesizes across layers. It does **not** do scenario-ID → architecture-node anchoring, blast-radius scoring, risk levels, tier classification, or templated Context blocks. If the user asks for impact assessment, blast radius, or a detailed analysis doc, point them at `/breeze:impact-analysis-v2`.
- **Don't run all graphs for an obviously single-layer question** — it wastes tokens and blurs the answer.

---

## Output

- **Behavior questions** → numbered sequential flow (trigger → UI → backend → deployed path → completion).
- **Discovery questions** → ranked list with entity type, name, description, parent (repo for code, DataLake for DDL), drill-down hints.
- **Always** name concrete entities (e.g., *Scenario "Run hourly ANZ ETL"*, *file `backend_nodejs_global_tnlm: src/project/.../search-result-project.dsl.ts`*, *Service node "global_tnlm"*, *DDLTable `RESEARCH_PROJECT_VERSION`*) rather than vague references.
- **Link the layers**: when functional + code + architecture + DDL all fire, show how they connect on this specific flow.
