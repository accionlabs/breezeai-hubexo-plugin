---
name: search
description: >
  Search the functional, design, code, and architecture graphs to answer
  questions about the system. Smart-routes to one graph, a subset, or all of
  them based on query intent. Use for: feature discovery, "how does X work",
  "who handles Y", finding code or UI implementations, and cross-layer
  questions. Default entry point for any question about the project.
---

## Guard

Read `.breeze.json`. If missing or incomplete, tell the user to run `/breeze:setup-project`.
Extract `projectUuid`. The Breeze MCP is authenticated separately — if a tool call fails with an auth error, tell the user to re-run `/breeze:setup-project` to re-authenticate.

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
- `Code_Graph_Search` — semantic search across File / Function / Class.
- `Get_Code_File_Details` — full hierarchical structure of a single file (classes, methods, decorators, statements).

### Design graph
- `Design_Graph_Search` — semantic search across design nodes (journeys, flows, pages, components).
- `Get_all_Design_By_Label` — list design nodes filtered by label type.
- `Get_Design_Nodes_by_Ids` — fetch specific design nodes by ID.

### Architecture graph
- `Get_All_architecture_Graph` — full read of the 8-layer architecture graph (UserExperience, ApiGateway, Observability, Agents, Services, EventQueue, DataLake, Infrastructure). Small graph — prefer this over label-filtered lookups.
- `Get_Architecture_Nodes_By_Label` — fetch architecture nodes by a specific layer label (use when you only need one layer and the full graph would be overkill).

---

## Phase 1 — Classify Query Scope

Inspect `$ARGUMENTS`. Decide **which graphs to search**. You can pick one, a subset, or all. The goal is to cover the layers the user's question actually touches — no more, no less.

### 1a. Single-graph fast-paths

Stop at the first match:

| Query shape | Route to |
|---|---|
| Pure code lookup — "where is X implemented", "find function/class/file Y", "show me the route handler for Z" | `Code_Graph_Search` → then `Get_Code_File_Details` on top hits |
| Pure UI — "what page shows X", "which component", "what does the settings screen look like" | `Design_Graph_Search` → optionally `Get_Design_Nodes_by_Ids` for detail |
| Personas / roles only — "who manages X", "what roles exist" | `Get_all_personas` → drill down (see Phase 3) |
| Pure deployment / topology — "what services are there", "what's the tech stack", "where is X deployed", "what data stores exist" | `Get_All_architecture_Graph` (full read), or `Get_Architecture_Nodes_By_Label` if only one layer is needed |

### 1b. Multi-graph triggers

If the query doesn't fit a single-graph fast-path, pick the layers it implies and run them in parallel.

**Trigger words that add a layer:**
- *screen, page, component, UI, UX, flow, wireframe, design* → add **Design**
- *file, function, class, method, module, import, repository* → add **Code**
- *service, deploy, deployment, region, SPOF, queue, topic, bucket, index, DB, data store, tech stack, infra* → add **Architecture**
- *persona, role, outcome, scenario, workflow, business logic* → add **Functional**

**Shape-based routing:**
- "how does X work" / "explain Y" / "what happens when…" / "walk me through Z" → **Functional + Code** minimum (add Design if UI-facing, add Architecture if deployment-facing)
- "what is the impact of…" / "what breaks if…" / "what does X depend on" → **Functional + Code + Architecture** (add Design if UI-facing)
- "compare X and Y" (where X, Y are features) → **Functional + Code**

**Default when truly ambiguous** → **Functional + Code**.

> Running all four graphs is fine when the question is genuinely cross-cutting. Don't over-narrow out of caution, but don't fan out to all four for a question that's obviously single-layer.

---

## Phase 2 — Execute in Parallel

Run every chosen semantic search / full-graph read simultaneously. Do not sequentialize.

- `Functional_Graph_Search`
- `Design_Graph_Search`
- `Code_Graph_Search`
- `Get_All_architecture_Graph`

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
- **Top hits are Files** → call `Get_Code_File_Details` on each to see classes, methods, decorators, statements.
- **Top hits are Functions or Classes** → their source and call chain are already in the search payload; only fetch the parent file if you need surrounding context.

### Design drill-down
- **Top hits are design nodes** → if the search payload is thin, call `Get_Design_Nodes_by_Ids` on the top IDs for full detail, or `Get_all_Design_By_Label` to widen within a label type.

### Architecture drill-down
- `Get_All_architecture_Graph` already returns every node in every layer. No further drill calls needed — just use the nodes in synthesis. If you only pulled one layer via `Get_Architecture_Nodes_By_Label` and the question actually spans layers, fall back to the full read.

### Cross-Persona Drill-Down

When drilling into a functional Outcome, check if the **same Outcome name exists under other Personas** (common for features that span UI and backend). After finding an Outcome under a User persona, also fetch it under the System persona (and vice versa) via `Get_all_outcomes_for_a_persona_id`, then `Get_all_scenarios_for_a_outcome_id` on the matching outcome. Merge both into the narrative.

---

## Phase 4 — Synthesize into One Narrative

Present a **single coherent answer**, not a per-graph ranked list.

### For behavior / "how does X work" questions
Build a sequential flow:
1. **Trigger** — what initiates it (UI event, cron, webhook, queue message).
2. **UI interaction** — from Design / functional User persona.
3. **Backend processing** — from functional System persona + Code.
4. **Deployed path** — which services / queues / data stores carry it (from Architecture), if architecture was in scope.
5. **Completion** — side effects, confirmation.

### For discovery questions
Provide a ranked list including:
- Entity type (Persona / Outcome / Scenario / Step / Action / Component / File / Function / Service / DataLake node / …)
- Name + one-line description
- Pointer to drill down further if useful

### Always when multiple layers fire
Weave them: *functional flow → code file/function → deployed service/queue/data store*. Show the connection; don't just list per layer.

---

## End-to-End Flow Questions

For any question shaped like *"what happens when…"*, *"explain the flow of…"*, *"how does X process work"*:

1. **Always** search **Functional + Code** at minimum.
2. Add **Design** if the flow starts at a UI action.
3. Add **Architecture** if the flow crosses services / queues / data stores (which is usually true).
4. In the functional graph, query **twice** — once with user-centric terms (UI actions, clicks, forms) and once with system-centric terms ("System processes…", "backend handles…", "External System…"). This captures both the trigger side and the processing side of the dual-persona model.
5. Drill the top functional hits all the way to Steps/Actions via the Phase 3 hierarchy chain.
6. Merge everything into one sequential trace (Phase 4).

A single search returns an incomplete picture for process questions — always fan out.

---

## Don'ts

- **Don't present raw search results as the final answer.** Always drill down and synthesize.
- **Don't claim a feature doesn't exist** from one failed search — try a rephrase or widen to another graph first.
- **Don't mention empty layers or graph-completeness gaps** in user-visible output (e.g., "the architecture graph has 0 Agents"). Silently fall back; present what you do have.
- **Don't duplicate `/breeze:impact-analysis`.** `search` reads and synthesizes across layers. It does **not** do scenario-ID → architecture-node anchoring, blast-radius scoring, risk levels, or templated Context blocks. If the user asks for impact assessment, blast radius, or a detailed analysis doc, point them at `/breeze:impact-analysis`.
- **Don't run all four graphs for an obviously single-layer question** — it wastes tokens and blurs the answer.

---

## Output

- **Behavior questions** → numbered sequential flow (trigger → UI → backend → deployed path → completion).
- **Discovery questions** → ranked list with entity type, name, description, drill-down hints.
- **Always** name concrete entities (e.g., *Scenario "Run hourly ANZ ETL"*, *file `dags/apac/anz_planning_development_application_dag.py`*, *Service node "ETL Pipeline"*) rather than vague references.
- **Link the layers**: when functional + code + architecture all fire, show how they connect on this specific flow.
