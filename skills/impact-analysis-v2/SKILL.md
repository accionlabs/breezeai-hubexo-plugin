---
name: impact-analysis-v2
description: >
  Searches Breeze functional, design, code, and architecture graphs to perform
  impact analysis of a proposed change. Shows an analysis summary and optionally
  generates a detailed document with diagrams. Use for: "impact analysis", "what
  if we change X", "what breaks if Y fails", "blast radius of Z" — cross-layer
  change assessment from functional → design → code → architecture (deployed-
  system blast radius). Supports optional `--project <uuid|name>` to target a
  Breeze project other than the one in `.breeze.json`, and `--detailed` to skip
  the summary and emit the full diagrammed document directly.
---

# Breeze Impact Analysis

Perform impact analysis of a proposed change across the Breeze functional, design, code, and architecture graphs to understand full context — from user intent down to deployed services, data stores, and operational blast radius — before acting.

## Your Task

Given a user prompt that implies a change (new feature, modification, removal, failure scenario), search all four Breeze graphs, perform an impact analysis, present a summary, and — if the user wants — generate a comprehensive document with diagrams.

## Step 1 — Project & Output Mode

### 1a. Project resolution

Follow CLAUDE.md's project-resolution rules (per-invocation override via `--project` flag or natural-language hint → `.breeze.json` fallback → announce active project). This skill prints `Breeze Impact Analysis` on the line above the `Project: <name> (<uuid>)` header so the report title appears first. Auth handling on Breeze MCP failures is also covered in CLAUDE.md.

### 1b. Detailed-mode flag

The user may pass `--detailed` (or `--full` / `--report`, or natural phrasing like "give me the full report", "detailed report", "skip the summary") anywhere in `$ARGUMENTS`. Treat this as a one-shot output mode override:

- **SUMMARY mode (default, no flag)** → run all steps as written. Step 4 emits the summary template, then asks the user whether to generate the detailed document.
- **DETAILED mode (flag present)** → run all steps as written, but in Step 4 **skip the summary template entirely** and emit the detailed document **directly into the conversation output (not to a file)**. The detailed report must appear inline in the response so it can be (a) read by the user, and (b) consumed as the return value when this skill is invoked from another skill or agent via `/breeze:impact-analysis-v2 --detailed ...`. Strip `--detailed` (and any synonyms) from `$ARGUMENTS` before processing the prompt.

Do not produce both a summary and a detailed document in the same invocation.

**Example invocations:**
- `/impact-analysis-v2 expand keyword search to status text and description` → SUMMARY mode (default).
- `/impact-analysis-v2 --detailed expand keyword search to status text and description` → DETAILED mode — emit the diagrammed document directly, no summary, no yes/no prompt.
- `/impact-analysis-v2 --project "Lead Manager V2" --detailed what breaks if Redis is offline` → DETAILED mode against an explicit project override.

## Step 2 — Read All Four Graphs (in parallel)

First, call `Call_List_Repositories_` to enumerate the project's repositories. Cache the inventory (name + id + metadata). You'll use it for two things: (a) scoping `Code_Graph_Search` per-repo when the prompt clearly maps to one or two subsystems, and (b) anchoring code hits back to their parent repo when reasoning about cross-repo coordination in Step 2.3, Step 3.1, and the final output.

Then run these four calls simultaneously using queries derived from the user's prompt:

1. **Functional Graph Search** (`Functional_Graph_Search`)
   Search for related personas, outcomes, scenarios, steps, and actions.

2. **Design Graph Search** (`Design_Graph_Search`)
   Search for related user journeys, flows, pages, and components.

3. **Code Graph Search** (`Code_Graph_Search`)
   Search for related code files, classes, methods, and modules. If the prompt strongly implies one or two repos from the inventory above (e.g., a frontend-only change → the `frontendweb_*` repo, a search-feature change → `backend_*_search_*`), issue parallel `repository_name=`-filtered searches per repo instead of one broad call. Otherwise do one broad call.

4. **Architecture Graph read — per label, in parallel** (`Get_Architecture_Nodes_By_Label`)
   Issue 8 parallel `Get_Architecture_Nodes_By_Label` calls, one per layer: `UserExperience`, `ApiGateway`, `ObservabilityMonitoring`, `Agents`, `Services`, `EventQueue`, `DataLake`, `Infrastructure`. **This tool is an enumeration, not a search** — it takes only `uuid` and `label` (no `query` parameter) and returns every node in that label. Step 2.2's intersection logic filters them down by relevance. Splitting per-label avoids the token-overflow that `Get_All_architecture_Graph` hits on populated projects (per-layer payloads can still be sizeable — e.g., DataLake with many DB schemas). Do NOT use `Architecture_Graph_Search` — that's similarity-filtered search and will silently drop anchors whose embedding doesn't match the prompt.

If a search returns no results, note "No matches found" for that graph. If the Architecture Graph is empty (all 8 layer counts are 0), note "Architecture Graph not populated" and skip Step 2.2.

## Step 2.1 — Functional Drill-Down

Based on the Functional_Graph_Search results, drill deeper into the functional graph:

1. ALWAYS call `Get_all_personas` first to get the full persona list with their IDs.
2. For outcomes/scenarios returned by the search, match the `personaId` field in each outcome back to the persona list to identify which personas are affected.
3. Call `Get_all_outcomes_for_a_persona_id` for each affected persona to get their full outcome list.
4. Call `Get_all_scenarios_for_a_outcome_id` for each relevant outcome.
5. For each relevant scenario, call `Get_all_steps_actions_for_a_scenario_id` to get the complete steps and actions.

This gives you the full functional chain: Persona → Outcome → Scenario → Steps → Actions.

## Step 2.2 — Architecture Impact

Skip this step if every architecture layer returned zero nodes. If only some layers are empty, silently skip those layers and proceed with the populated ones — do not surface "layer X is empty" anywhere in the output (see HARD RULE).

This step is about **how the user's prompt impacts the deployed system** — not about auditing the architecture graph for metadata completeness. The goal is to answer: *"given what the user is proposing to change, which deployed nodes are on the hook, and what does that mean for them operationally?"*

Intersect the functional + code results with the Architecture Graph to identify touched nodes, then reason about what the prompt means for those nodes.

1. **Anchor by name / description / technology / domain.** Cross-reference keywords from the functional scenarios, design journeys/pages/components, and the user's prompt itself against every architecture node's `name`, `description`, `domain`, and `technologies` fields. This is the primary anchoring path. (Note: architecture nodes no longer carry a `scenario` array — scenario-based anchoring is not used.)

2. **Anchor by code.** For each Code Graph hit, read its `codeOntologyId` (or `clusterId` on File nodes). Match against the `code_ontology_id` on every architecture node. Any match is a **code-anchored** touchpoint.

3. **One-hop supporting-layer expansion.** For each touched Service, include the supporting-layer nodes it depends on — DataLake (databases, indexes, caches, object stores), EventQueue (queues, CDC streams, event buses), ObservabilityMonitoring (logs, metrics, alerts), and Infrastructure (compute, networking, IaC). Use name patterns, shared technologies, and cross-reference with Code Graph imports (e.g., `loopback-connector-oracle` → Oracle DataLake node; `ElastiCache` imports → Redis node) when explicit relationship edges are not available.

4. **Request-path ordering.** If the prompt touches user-facing flows, order the ApiGateway nodes into a request path (typically edge → DNS → CDN/WAF → gateway → load balancer → ingress) so the summary can show the full hop chain.

5. **Impact assessment on touched nodes.** For the set of nodes surfaced in steps 1–4, reason about what the user's prompt means for them. Useful lenses:
   - **Failure domains touched by this prompt** — which of the touched nodes are single points of failure *for this specific flow* (e.g., a lone ES gateway that all search traffic funnels through).
   - **Data roles for this prompt** — which touched data store is the source of truth, which is the indexed / cached / replicated copy; what reads vs. writes the prompt implies.
   - **Regional reach** — does the prompt cross regional shards, and if so, how does region routing happen (env var, header, tenant lookup).
   - **Async tail implications** — how writes from this prompt propagate through CDC / queues / notifications, and what the lag consequences are for the user-facing surface.
   - **Deployment coordination** — how many services must be redeployed together for a change implied by the prompt; whether shared code (e.g., duplicated query builders) forces multi-repo synchronization.
   - **Capacity / scaling pressure** — if the prompt implies a traffic-shape change (new filter, heavier query, fanout), which nodes absorb that pressure first.
   - **Reuse opportunities** — if a proposed change implies new cross-cutting capability (auth, notifications, search) but an existing Platform Service already provides it, call that out so the user doesn't build a duplicate.

   This step's output is a set of operational consequences for nodes that exist in the graph. Keep the focus on the impact picture.

## Step 2.3 — Code Graph Deep-Dive

For the top 3–5 `Code_Graph_Search` hits, call `Get_Code_File_Details` when any of the following holds:

- The hit is a `File` node and you need its method/class structure to pinpoint the actual touchpoint inside the file.
- The prompt implies modifying or extending a specific function or route that the search returned only at file granularity.
- You need explicit repo + clusterId + module info to anchor a Code Context entry confidently.

For each enriched hit, retain the parent repo (from the inventory in Step 2). When the touched code spans multiple repos, render Code Context entries with their parent repo prefix (e.g. `frontendweb_react_tnlm: src/utils/posthog.ts`) so cross-repo coordination is visible at a glance — never bare paths when more than one repo is involved.

If `Code_Graph_Search` returned zero hits, skip this step. Steps 2.1 and 2.3 are independent and can run in parallel.

## Step 2.4 — Data Layer (Schema-Side) Impact

This step surfaces DDL-level impact — columns, procedures, triggers, materialized views, indexes, constraints — that the architecture-layer search alone misses. It runs in three stages with bounded call cost. Stage 1 is the trigger; Stages 2 and 3 are conditional.

### Stage 1 — Broad DDL Discovery (always when triggered)

**Trigger:** Step 2.2 surfaced ≥1 touched DataLake. Run Stage 1 against ALL touched DataLakes — non-relational lakes (ES, Redis, Kafka) naturally produce zero DDL hits and drop out. Do NOT pre-filter by `category` field — the field is AI-generated and may vary across projects.

**Action:** 5 parallel `Architecture_Graph_Search` calls, one per DDL label.

```
parallel:
  Architecture_Graph_Search(uuid, query=<prompt>, include_labels=["DDLColumn"],     threshold=0.4)
  Architecture_Graph_Search(uuid, query=<prompt>, include_labels=["DDLProcedure"],  threshold=0.4)
  Architecture_Graph_Search(uuid, query=<prompt>, include_labels=["DDLView"],       threshold=0.4)
  Architecture_Graph_Search(uuid, query=<prompt>, include_labels=["DDLConstraint"], threshold=0.4)
  Architecture_Graph_Search(uuid, query=<prompt>, include_labels=["DDLIndex"],      threshold=0.4)
```

**Filter & gate:**
- Drop hits whose `dbOntologyId ∉ touched_datalakes` (filter to the subset surfaced in Step 2.2)
- If total surviving hits < 3 → **suppress the Data Context block** in the summary AND **the Data Layer table** in the detailed doc (change isn't schema-touching)
- Otherwise proceed

### Stage 2 — Focus-Table Detail (selective, default cost = 0)

**Identify focus tables:** cluster Stage 1 hits by `tableId`. A focus table has ≥2 hits across any DDL labels. **Cap at 5 focus tables.** If more than 5, surface this in the output as a scope-too-broad signal — do not fan out.

**For each focus table, in order, stop when you have enough:**

1. **Parse `ddlText` from the Stage 1 table hit (default, 0 extra calls).** Every `DDLTable` returned by Stage 1 includes the full `CREATE TABLE` statement. Parse for columns + types + lengths + nullability + defaults + inline PK + inline constraints. Covers ~80% of impact-analysis questions.

2. **Targeted semantic column search (1 call, only if needed).** When the prompt implies cross-table column comparison (e.g. "find all `STATUS_TEXT` columns"):
   ```
   Architecture_Graph_Search(uuid, query="<TABLE_NAME> columns",
                             include_labels=["DDLColumn"], threshold=0.3)
   ```
   Approximate but cheap. May miss generic columns (CREATED_AT, IS_DELETED).

3. **DataLake-wide bulk fetch (last resort, amortized per DataLake).** Only when ≥2 focus tables share a parent DataLake AND structured column nodes are essential:
   ```
   Get_DB_Schema_Nodes_By_Label(data_lake_id, label="column")  ← one call per DataLake
   → cache result, client-side filter by tableId for each focus table
   ```
   One expensive call (~thousands of nodes) reused across all focus tables in that DataLake.

### Stage 3 — Cross-Reference Search (selective, only for drops / renames / type-changes / FK additions)

**Trigger:** prompt implies a structural change that needs downstream-dependency scan. Skip for purely additive changes.

**Action per affected DataLake (parallel, cached per DataLake):**
```
Get_DB_Schema_Nodes_By_Label(data_lake_id, label="constraint")  → filter incoming FKs by referenced table
Get_DB_Schema_Nodes_By_Label(data_lake_id, label="view")        → grep definition for table name (word-boundary)
Get_DB_Schema_Nodes_By_Label(data_lake_id, label="procedure")   → grep body for table name (word-boundary)
```

### Classify each touched DataLake into one of three tiers

For the output, classify each touched DataLake by combining its writability with the nature of its findings:

| Tier | Trigger | Render |
|---|---|---|
| **🚨 Blocking (RO)** | DataLake is read-only (pattern matches `rmsowned-readonly` or similar) AND Stage 1 surfaced new columns/constraints/views being proposed | Full detail + "cross-team coordination needed" callout — appears FIRST in the Data Context / Data Layer section |
| **● V2-writable** | DataLake's pattern indicates V2 can mutate it (`full-crud`, `v2-modifies-via-migration`) AND DDL changes are needed | Full detail — application/migration work |
| **✓ Confirmed (RO)** | DataLake is read-only AND Stage 1 only surfaced EXISTING columns the prompt references — no DDL needed | One-line summary: "RO ✓ — columns X, Y already present, no DDL required" |
| **— Tangential** | DataLake had < 3 hits all with score 0.40-0.45 (weak match) | **Skip entirely from output** |

Detect writability via the DataLake's `pattern` array (e.g., `"rmsowned-readonly"` → read-only; `"full-crud-v1-and-v2"`, `"v2-modifies-via-migration"` → writable). Pattern strings are project-specific but consistently descriptive.

### Verdict

After tier classification, emit a single-line **Verdict** that takes one of these states:

- **🔴 BLOCKING** — at least one DataLake is in 🚨 tier (cross-team coordination required before V2 work can begin)
- **🟡 V2 WORK** — at least one ● V2-writable DataLake needs DDL, no blocking dependencies
- **🟢 NO DDL** — all findings are in ✓ Confirmed tier; application-layer change only

The Verdict is the most actionable single piece of output from this step. It belongs as the **last line** of the Data Context block (summary) and as a **callout box** at the bottom of the Data Layer section (detailed doc).

### Call-budget summary

| Scenario | Calls |
|---|---|
| UI-only ticket (Stage 1 < 3 hits) | 5 (Stage 1 only, then skip) |
| Schema-touching, additive, 3 focus tables | 5 (Stage 1) + 0 (ddlText parse) = **5** |
| Cross-table column comparison needed | 5 + 3 = **8** |
| Drop/rename, 1 DataLake, full dependency scan | 5 + 0 + 3 = **8** |
| Heavy: drop/rename across 2 DataLakes | 5 + 0 + 6 = **11** |

Bounded in every realistic case. Compare to a naive "always bulk fetch every DataLake" approach which could hit 50+ calls per analysis.

## Step 3 — Deep Analysis

Synthesize results from all four graphs into a deep analysis. Think across layers:

- **What does this prompt mean functionally?** — Which personas are involved, what outcomes/scenarios are touched, what business logic is relevant.
- **What does this mean from a design perspective?** — Which UI flows, pages, components are related, what user journeys are affected.
- **What does this mean at the code level?** — Which files, modules, classes, methods implement the relevant functionality.
- **What does this mean at the deployed-system level?** — Which UserExperience / ApiGateway / Services / Agents / EventQueue / DataLake / ObservabilityMonitoring / Infrastructure nodes are touched *by this prompt*. What's the request path. What's the async tail (CDC, queues, alert chains). What data stores hold the source-of-truth vs. the indexed / cached copy. Which of the touched nodes are single points of failure for this flow. The frame is **"what does the user input mean for the architecture?"**, not "what is wrong with the architecture graph?".
- **How are these connected?** — Trace the thread from functional requirement → design element → code implementation → deployed architecture node. Focus on the impact path, not ontology gaps.
- **What are the dependencies and risks?** — Upstream/downstream effects, shared components, breaking change potential. At the architecture layer, quantify **blast radius in deployed nodes** (how many services redeploy, how many data stores migrate, how many regions affected) — not just code files.

## Step 4 — Return Analysis Summary

**If DETAILED mode was set in Step 1b, skip this template entirely** and emit the detailed document directly **into the conversation output** (described in the last bullet of the Rules section). Do not also produce the summary. Do NOT write the report to a file unless the user explicitly asks for a file path (e.g. "save it to docs/impact.md"). The default is always inline output, so parent skills and agents can consume this skill's return value directly.

Otherwise emit the summary using the single template below.

```
Breeze Impact Analysis
Project: <name> (<uuid>)

Functional Context:
  - Personas: <list of relevant personas>
  - Scenarios: <list of relevant scenario names>
  - Key Actions: <most relevant actions/steps>

Design Context:
  - User Journeys: <related journeys>
  - Flows/Pages: <related flows and pages>
  - Components: <related UI components>

Code Context:
  - Files: <related source files, prefixed with parent repo when more than one repo is involved>
  - Modules/Classes: <related modules>
  - Key Methods: <relevant methods/functions>

  ** When 3+ files share a pattern that's being changed (multiple DSL builders, parallel
     controllers, similar React components), REPLACE the bullets above with a markdown
     table — significantly more scannable than bullets when the change is "same edit
     across N similar sites": **

  | Repo | File | Endpoint / Page | Current | Adds |
  |---|---|---|---|---|

  Column meanings:
  - **Repo** — which repo to clone/edit (`backend_nodejs_global_tnlm`, `frontendweb_react_tnlm`, etc.).
  - **File** — path with line number (e.g. `src/.../search-result-project.dsl.ts:18`). No symbol name in this column.
  - **Endpoint / Page** — the user-facing thing this file backs: a route (`POST /v1/search/project`), a page (`Dashboard search bar`), or a component slot. One concept per cell, not both.
  - **Current** — what the code does today (fields searched, behavior, fields rendered).
  - **Adds** — what the change introduces (new fields, new behavior, copy diff).

  If multiple symbols inside the same file change, list multiple rows — don't try to cram symbols into the `File` cell.

Architecture Context:
  - UserExperience: <delivery modalities on this flow>
  - ApiGateway path: <ordered request hop chain>
  - Services: <service nodes exercised, with tech stack>
  - Agents: <agent / LLM-workflow nodes exercised on this flow>
  - DataLake: <stores read/written on this flow, noting source-of-truth vs index/cache>
  - EventQueue: <queues / CDC / event buses that carry this flow's async tail>
  - Observability: <log / metric surfaces for this flow>
  - Infrastructure: <compute / networking substrate>

Data Context (conditional — only emit when Step 2.4 Stage 1 returned ≥3 hits):
  - 🚨 Cross-team (RO): <list RO DataLakes where DDL is needed but V2 can't ship it — name DataLake + 1-line "what needs to change + who owns it">
  - ● V2 work: <list V2-writable DataLakes where DDL changes are needed — name DataLake + 1-line summary>
  - ✓ Confirmed (RO): <list RO DataLakes where existing columns/structures suffice — single-line "RO ✓ — columns X, Y already present">
  - Verdict: <🔴 BLOCKING / 🟡 V2 WORK / 🟢 NO DDL> — <one-line justification>

  Omit any of the three lines (Cross-team / V2 work / Confirmed) that have no entries.
  Omit the whole Data Context block if Step 2.4 was suppressed (< 3 Stage 1 hits).

Impact on touched nodes:
  <2–4 bullets: SPOFs for this flow, source-of-truth vs. cache data roles,
   async-tail cost, which nodes absorb new load, multi-service coordination needs>

Feature Traceability:
  <how functional → design → code → architecture are linked for this prompt>

Architecture Blast Radius: <LOW / MEDIUM / HIGH / CRITICAL> — <one-line quantification>

Risk Level: <Low/Medium/High> — <one-line justification accounting for code-level risk
  AND deployed-system blast radius>

Cross-checks vs the prompt (include this block ONLY when the user's prompt itself
includes a stated analysis, count, or assertion the skill can validate or dispute):
  - ✅ <items the analysis confirmed — one bullet per confirmed claim>
  - ⚠️ <items the prompt understated, missed, or got wrong — one bullet each
        with a one-line explanation of the delta>
```

Notes on the template:
- Include a layer line in the Architecture Context block only when at least one node in that layer is touched by the prompt. The block is a roster of what the prompt exercises — silently omit empty layers (do NOT print "(none)" or "Agents: empty").
- If every architecture layer is empty, replace the whole Architecture Context block with a single line: `Architecture Context: Not available — architecture graph is not populated for this project.`
- Mark each architecture node with `✓` (exists, touched), `✗` (removed / failing in this scenario), `~` (modified), or `+` (new, proposed).
- Each `Impact on touched nodes` bullet names a specific touched node and describes an operational consequence — failure behavior, load characteristic, coordination requirement, or data-flow role.
- **Emit `Impact on touched nodes`, `Architecture Blast Radius`, and `Risk Level` lines only when the prompt clearly implies a change to the system** (new feature, modification, removal, failure scenario). If the prompt turns out to be purely exploratory ("how does X work?"), omit these three blocks and add a one-line note suggesting `/breeze:search` for explanatory questions — this skill is impact-analysis only.

IMPORTANT: After presenting this summary, explicitly ask the user:

"Would you like me to generate a detailed analysis document with diagrams for this? (yes/no)"

## Rules

- Always read all four graphs, even if the prompt seems to only affect one area. Functional / Design / Code are semantic searches; Architecture is per-label enumeration via 8 parallel `Get_Architecture_Nodes_By_Label` calls (one per architecture label).
- The summary should be informative but concise — the detailed doc comes later if requested (or directly in DETAILED mode).
- **DETAILED mode (Step 1b):** when the user passes `--detailed` (or a synonym), Step 4 skips the summary template and emits the detailed document directly. Never produce both a summary and a detailed document in the same invocation.
- **Output target — always conversation, never a file (unless explicitly requested):** both the SUMMARY template (Step 4) and the DETAILED document are emitted inline as the response. Do NOT use the Write tool or any file-creation path to save the report. The only exception is an explicit user instruction with a file path (e.g. *"save the detailed report to `docs/impact-2026-05-18.md`"* or *"write it to `/tmp/x.md`"*). Default = inline conversation output, every time. This is critical because this skill may be invoked from another skill or agent via `/breeze:impact-analysis-v2 --detailed ...` — the parent expects v2's response to BE the report, not a path to a file the parent then has to read back.
- **Project targeting (Step 1a + CLAUDE.md):** project resolution follows CLAUDE.md's rules (`--project <uuid|name>` override → `.breeze.json` fallback). Step 1a adds a skill-specific natural-language hint form. Do not mutate `.breeze.json`.
- If functional, design, AND code searches all return no results, report "No relevant context found in any graph" and skip the doc offer — even if the architecture graph is populated, there's nothing to anchor it to.
- Do not modify or interfere with the user's original prompt — your job is analysis only.
- Architecture analysis is read-only. This skill never mutates the Architecture Graph.
- Architecture analysis is **impact-on-deployed-system**, not ontology audit. Frame every architecture observation as "what does the user's prompt mean for this node?" — not "what is missing from the graph?".
- **HARD RULE — never surface graph-hygiene gaps to the user.** This is output shown to non-Breeze readers who may misread coverage shortfalls as product defects. Do NOT include (in the summary, the detailed doc, or any user-visible line):
  - "empty scenario arrays", "unanchored", "orphan scenario", "graph-hygiene gap", "fully unanchored"
  - "Agents layer is empty / count=0" or any "layer X has 0 nodes" observation
  - "missing code_ontology_id", "missing regional counterpart", "region mismatch"
  - a `Gaps:` line inside the Architecture Context block, ever
  - any itemization of what the graph is missing

  If `scenario[]` arrays or `code_ontology_id` fields are empty, silently fall back to name/technology matching and proceed with the impact analysis — do not mention the fallback. If the architecture graph is entirely empty, use the single baseline line from the "Notes on the Architecture Context block" section and stop there.
- **The detailed document (if requested, or auto-emitted in DETAILED mode)** is a structured artifact, not a long-form essay. Emit the sections below in this order. Mandatory sections appear in every detailed doc; conditional sections appear only when applicable.

  **MANDATORY HEADER — print verbatim, immediately under the title/metadata block:**

  > `Legend: + new · ~ modified · ✓ existing/touched · ✗ removed`

  This legend covers markers across all four ontologies (Functional, Design, Code, Architecture). It MUST appear in every detailed doc — readers from outside the team rely on it to interpret the per-layer tables.

  **MANDATORY SECTIONS (in this order):**

  1. **Executive Summary** — 2–4 paragraphs naming the change, the surface area, and the headline risk.

  2. **Per-layer touched-nodes tables — one per ontology, markers on every row.** All five ontologies get the same marker treatment in the detailed doc (this differs from the SUMMARY template above, which marks only Architecture nodes):
     - *Functional Layer table* — columns: `Marker | Type (Outcome/Scenario/Action) | Name | Notes`
     - *Design Layer table* — columns: `Marker | Type (Page/Component/UserJourney/Flow) | Name | Notes`
     - *Code Layer table* — columns: `Marker | Repo | File | Endpoint / Page | Current | Adds` (same shape as the summary's Code Context table, plus the marker column up front — see Step 4 for column meanings)
     - *Architecture Layer table* — columns: `Marker | Layer | Node | Touched because…`
     - *Data Layer (Schema-Side) section* — **conditional on Step 2.4 Stage 1 returning ≥3 hits.** Render as one block per touched DataLake (not a flat table — the per-DataLake grouping is the point). Each block uses the tier classification from Step 2.4:
       * 🚨 Blocking (RO) blocks appear FIRST. Include: DataLake name + access pattern; tables touched with current `ddlText`-parsed column summary; what DDL is being requested; dependent procedures / materialized views / triggers / incoming FKs; the "RMS-or-equivalent coordination ticket needed" callout.
       * ● V2-writable blocks appear second. Same content shape; framed as work V2 can ship via its migration repo.
       * ✓ Confirmed (RO) blocks appear last as one-liners: "<DataLake> (RO): <columns> already present, no DDL required".
       * Skip — Tangential entirely.
       * Close the section with the **Verdict callout box**: 🔴 BLOCKING / 🟡 V2 WORK / 🟢 NO DDL with a 1-2 sentence justification. Same Verdict line that appeared in the summary's Data Context — repeated here so the detailed doc is self-contained.

  3. **Request-path Mermaid diagram** — MUST use `classDef` color-coding (e.g., `changed` / `unchanged` / `storage`) so every node visually carries its status. Do not emit an uncolored diagram. Example:

     ```
     classDef changed fill:#fff4b8,stroke:#b8860b,color:#000
     classDef unchanged fill:#e8f4ff,stroke:#3a78a8,color:#000
     classDef storage fill:#e8ffe8,stroke:#2e8b57,color:#000
     class GT,IDX1,IDX2 changed
     class R,CF,AG,NG unchanged
     class ES,DB storage
     ```

  4. **Async-tail Mermaid diagram** — full pipeline from upstream source through queue/CDC/worker/destination, including external sinks (SES, third-party APIs, Kafka topics, downstream DBs) when relevant.

  5. **Cross-layer traceability matrix** — table with one row per scenario; columns `Functional | Design | Code | Architecture`.

  6. **Single Points of Failure for this flow** — list nodes whose failure breaks *this flow specifically* (not generic platform SPOFs). One paragraph per SPOF: failure mode → user-visible effect → mitigation in this design.

  7. **Risk taxonomy — structured.** List 3–7 risks. Each risk uses this sub-structure:
     - **Heading** with a `<Likelihood> × <Consequence>` rating (e.g., "Latency on the ES hot path — LOW likelihood × HIGH consequence")
     - **Why** — one paragraph
     - **Worst case** — one sentence
     - **Mitigation** — numbered list of concrete actions

     Do NOT emit a generic risk paragraph or a flat risk table — the Why/Worst case/Mitigation breakdown is what makes this section actionable for triage.

  8. **QA test plan — concrete table.** Markdown table with columns `# | Test | Expected`. 6–12 specific test cases with concrete inputs and concrete expected outputs. Include regression-guard cases (empty input, behavior unchanged on existing paths) and edge cases (special characters, pagination boundaries, large payloads).

  9. **Operational considerations** — alarms to add or tune, capacity test plan, rollback path. Bulleted; concrete; name specific dashboards/services where possible.

  10. **Out of Scope** — explicit bulleted list of what the change does NOT affect. Pre-empts scope creep and is often as valuable as the in-scope analysis.

  11. **Open Questions for the change author** — numbered list of 3–6 specific questions that surfaced during analysis and need answering before merging. Questions must have concrete implementation impact (field placement, indexer behavior, UX behavior, pagination, auth) — not generic "have you considered…" prompts.

  **CONDITIONAL SECTIONS (include when applicable):**

  - **Implementation Options** — include when 2+ viable paths exist for shipping the change (msearch fanout vs unified index, sync vs async, in-place migration vs blue-green, shared base class vs duplicated edits). Each option: pros / cons / cost-to-ship / risk + a recommendation with reasoning. Omit when there's only one sensible path.

  - **Multi-Service Deploy Coordination** — include whenever more than one service or repo must redeploy. Cover ordering, what makes each step backwards-compatible (or not), which deploys can lag behind the others, and rollback path per service.

  - **Staged-rollout plumbing** — include if the prompt implies a feature flag or cohort rollout: flag storage location, where the gate lives in code, kill-switch behavior, rollout stage definition (Stage 0 internal → Stage 1 design partners → …).

  - **Suggested ticket slicing** — include for non-trivial changes. Use IDs with explicit dependencies (e.g., `DB-01 → B-01 → T-01`). One line per ticket: scope + dependency.

  The architecture section describes impact on touched nodes — never enumerates graph-completeness gaps.
