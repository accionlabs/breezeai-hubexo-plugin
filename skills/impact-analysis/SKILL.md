---
name: impact-analysis
description: >
  Searches Breeze functional, design, code, and architecture graphs to perform
  deep analysis of any user prompt. Shows analysis summary and optionally
  generates a detailed document with diagrams. Use for: "impact analysis",
  "deep analysis", "analyze this prompt", cross-layer impact assessment from
  functional → design → code → architecture (deployed-system blast radius).
---

# Breeze Deep Analysis

Perform deep analysis of a user prompt across the Breeze functional, design, code, and architecture graphs to understand full context — from user intent down to deployed services, data stores, and operational blast radius — before acting.

## Your Task

Given a user prompt, search all four Breeze graphs, perform a deep analysis, present a summary, and — if the user wants — generate a comprehensive document with diagrams.

## Step 1 — Read Project Config

Read `.breeze.json` to get the `projectUuid`. If it doesn't exist, respond with: "No Breeze project configured. Skipping analysis."

## Step 2 — Read All Four Graphs (in parallel)

Run these four calls simultaneously using queries derived from the user's prompt:

1. **Functional Graph Search** (`Functional_Graph_Search`)
   Search for related personas, outcomes, scenarios, steps, and actions.

2. **Design Graph Search** (`Design_Graph_Search`)
   Search for related user journeys, flows, pages, and components.

3. **Code Graph Search** (`Code_Graph_Search`)
   Search for related code files, classes, methods, and modules.

4. **Architecture Graph read** (`Get_All_architecture_Graph`)
   Fetch the full 8-layer architecture graph once per invocation. This is a full-graph read, not a semantic search — the architecture graph is small (typically < 100 nodes) and the intersection logic in Step 2.2 needs every layer.

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

Skip this step if the Architecture Graph is empty.

This step is about **how the user's prompt impacts the deployed system** — not about auditing the architecture graph for metadata completeness. The goal is to answer: *"given what the user is asking about or proposing to change, which deployed nodes are on the hook, and what does that mean for them operationally?"*

Intersect the functional + code results with the Architecture Graph to identify touched nodes, then reason about what the prompt means for those nodes.

1. **Service / UX / ApiGateway anchoring by scenario.** For each functional scenario ID surfaced in Step 2.1, scan the `scenario` array on every `UserExperience`, `ApiGateway`, and `Services` node. Any node whose `scenario` array contains the ID is a **scenario-anchored** touchpoint. If scenario arrays are empty across the graph, fall back to name/description/technology matching; treat this as the anchoring mode of last resort and continue with the impact analysis.

2. **Service / UX / ApiGateway anchoring by code.** For each Code Graph hit, read its `codeOntologyId` (or `clusterId` on File nodes). Match against the `code_ontology_id` on every architecture node. Any match is a **code-anchored** touchpoint.

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

   This step's output is a set of operational consequences for nodes that exist in the graph. If an ontology-hygiene limitation surfaces along the way, route the user to `/breeze:analyze-architecture` in a single trailing line and keep the focus on the impact picture.

## Step 3 — Classify Prompt Intent

Before synthesizing, classify the user's prompt into one of two frames. This determines the output shape in Step 4.

- **Explanatory frame** — "how does X work?", "explain Y", "walk me through Z", "what happens when…". The user wants to *understand the system as it is*. Output focuses on the runtime trace across layers, per-node role in the flow, and data-freshness mechanics. Do NOT emit "Impact on touched nodes", "Architecture Blast Radius", or "Risk Level" lines — these only make sense when evaluating a change.
- **Change/impact frame** — "what if we change X?", "impact of removing Y", "what breaks if Z fails?", "blast radius of…". The user is proposing or evaluating a change. Output focuses on SPOFs for the flow, multi-service coordination, async-tail consequences, blast radius, and risk level.
- **Mixed** — include both, but as distinct sections so the explainer isn't polluted with risk language.

Default to explanatory when uncertain. Explanatory output is strictly a subset of the reasoning for the change-impact frame; emitting change-impact language on a benign question reads as alarmist.

## Step 3.1 — Deep Analysis

Synthesize results from all four graphs into a deep analysis. Think across layers:

- **What does this prompt mean functionally?** — Which personas are involved, what outcomes/scenarios are touched, what business logic is relevant.
- **What does this mean from a design perspective?** — Which UI flows, pages, components are related, what user journeys are affected.
- **What does this mean at the code level?** — Which files, modules, classes, methods implement the relevant functionality.
- **What does this mean at the deployed-system level?** — Which UserExperience / ApiGateway / Services / Agents / EventQueue / DataLake / ObservabilityMonitoring / Infrastructure nodes are touched *by this prompt*. What's the request path. What's the async tail (CDC, queues, alert chains). What data stores hold the source-of-truth vs. the indexed / cached copy. Which of the touched nodes are single points of failure for this flow. The frame is **"what does the user input mean for the architecture?"**, not "what is wrong with the architecture graph?".
- **How are these connected?** — Trace the thread from functional requirement → design element → code implementation → deployed architecture node. Focus on the impact path, not ontology gaps.
- **What are the dependencies and risks?** — Upstream/downstream effects, shared components, breaking change potential. At the architecture layer, quantify **blast radius in deployed nodes** (how many services redeploy, how many data stores migrate, how many regions affected) — not just code files.

## Step 4 — Return Analysis Summary

Return the deep analysis using the template that matches the prompt intent from Step 3.

### Template A — Explanatory prompts ("how does X work?")

```
Breeze Deep Analysis

Functional Context:
  - Personas: <list of relevant personas>
  - Scenarios: <list of relevant scenario names>
  - Key Actions: <most relevant actions/steps>

Design Context:
  - User Journeys: <related journeys>
  - Flows/Pages: <related flows and pages>
  - Components: <related UI components>

Code Context:
  - Files: <related source files>
  - Modules/Classes: <related modules>
  - Key Methods: <relevant methods/functions>

Architecture Context:
  - UserExperience: <delivery modalities on this flow>
  - ApiGateway path: <ordered request hop chain>
  - Services: <service nodes exercised, with tech stack>
  - DataLake: <stores read/written on this flow, noting source-of-truth vs index/cache>
  - EventQueue: <queues / CDC / event buses that carry this flow's async tail>
  - Observability: <log / metric surfaces for this flow>
  - Infrastructure: <compute / networking substrate>

How it works:
  <4–8 bullet runtime trace of the flow end-to-end: entry point → request hops →
   service(s) that execute the core logic → data stores read/written → async tail.
   Each bullet names concrete nodes/files and explains its role in *this specific flow*.
   This is a narrative explanation, not a risk assessment.>

Feature Traceability:
  <how functional → design → code → architecture are linked for this prompt>
```

### Template B — Change/impact prompts ("what if we change X?", "what breaks if Y fails?")

```
Breeze Deep Analysis

[Functional / Design / Code / Architecture Context blocks — same as Template A]

Impact on touched nodes:
  <2–4 bullets: SPOFs for this flow, source-of-truth vs. cache data roles,
   async-tail cost, which nodes absorb new load, multi-service coordination needs>

Feature Traceability:
  <how functional → design → code → architecture are linked for this prompt>

Architecture Blast Radius: <LOW / MEDIUM / HIGH / CRITICAL> — <one-line quantification>

Risk Level: <Low/Medium/High> — <one-line justification accounting for code-level risk
  AND deployed-system blast radius>
```

### Template C — Mixed prompts

Emit both an explanatory section (from A) and a change/impact section (from B), clearly separated so the explainer isn't polluted with risk language.

Notes on the Architecture Context block:
- Include a layer line only when at least one node in that layer is touched by the prompt. The block is a roster of what the prompt exercises.
- If the Architecture Graph is not populated, replace the whole block with a single line: `Architecture Context: Not available — architecture graph is not populated for this project. Run /breeze:analyze-architecture to baseline.`
- Mark each node with `✓` (exists, touched), `✗` (removed / failing in this scenario), `~` (modified), or `+` (new, proposed) — the same convention used in the analyze-architecture write-back format.
- Each `Impact on touched nodes` bullet (Template B only) names a specific touched node and describes an operational consequence — failure behavior, load characteristic, coordination requirement, or data-flow role — that the user's prompt produces for that node.

IMPORTANT: After presenting this summary, explicitly ask the user:

"Would you like me to generate a detailed analysis document with diagrams for this? (yes/no)"

## Rules

- Always read ALL four graphs, even if the prompt seems to only affect one area. Functional / Design / Code are semantic searches; Architecture is a full-graph read.
- The summary should be informative but concise — the detailed doc comes later if requested.
- If functional, design, AND code searches all return no results, report "No relevant context found in any graph" and skip the doc offer — even if the architecture graph is populated, there's nothing to anchor it to.
- Do not modify or interfere with the user's original prompt — your job is analysis only.
- Architecture analysis is read-only. This skill never mutates the Architecture Graph — for that, direct the user to `/breeze:analyze-architecture`.
- Architecture analysis is **impact-on-deployed-system**, not ontology audit. Frame every architecture observation as "what does the user's prompt mean for this node?" — not "what is missing from the graph?".
- **HARD RULE — never surface graph-hygiene gaps to the user.** This is output shown to non-Breeze readers who may misread coverage shortfalls as product defects. Do NOT include (in the summary, the detailed doc, or any user-visible line):
  - "empty scenario arrays", "unanchored", "orphan scenario", "graph-hygiene gap", "fully unanchored"
  - "Agents layer is empty / count=0" or any "layer X has 0 nodes" observation
  - "missing code_ontology_id", "missing regional counterpart", "region mismatch"
  - a `Gaps:` line inside the Architecture Context block, ever
  - any itemization of what the graph is missing

  If `scenario[]` arrays or `code_ontology_id` fields are empty, silently fall back to name/technology matching and proceed with the impact analysis — do not mention the fallback. If the architecture graph is entirely empty, use the single baseline line from the "Notes on the Architecture Context block" section and stop there.
- Match the output frame to the prompt intent (Step 3). Explanatory prompts must not carry Blast Radius / Risk / "Impact on touched nodes" lines — those are change/impact framings only.
- The detailed document (if requested) should include: Mermaid diagrams for the request path and the async tail, a per-layer touched-nodes table, a cross-layer traceability matrix (Functional → Design → Code → Architecture), and an explicit single-points-of-failure section when the prompt is a resilience / incident query. The document's architecture section should describe impact on touched nodes (change/impact prompts) or runtime mechanics (explanatory prompts) — not enumerate graph-completeness gaps in either case.
