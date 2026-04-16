---
name: setup-project
description: >
  Initialize or validate the Breeze workspace. Guides MCP
  authentication, links to a Breeze project, saves the projectUuid
  to .breeze.json, and checks ontology readiness. Does NOT upload
  repos or documents — use /breeze:analyze-functional or
  /breeze:visual-to-text for document and design ingestion. Use
  when: first time setup, "init breeze", "setup breeze", or when
  any Breeze tool fails with authentication errors.
---

## Scope

This skill is responsible for **workspace bootstrap only**:

- Breeze MCP authentication check
- Project linking (select existing or create new)
- Ontology readiness check
- Pointing the user at the right next-step skill based on what they
  want to do

This skill does **NOT** upload repositories or documents. That
responsibility lives in dedicated skills:

| You want to… | Use this skill instead |
|---|---|
| Ingest a PDF / markdown / text document | `/breeze:analyze-functional` (handles document input as part of the analysis flow) |
| Convert a UI design visual into user stories | `/breeze:visual-to-text` |

If the user asks for any of those during setup, finish the bootstrap
steps first and then point them at the right skill in Step 4.

## Prerequisites

Read `.breeze.json` from the project root. If it exists and contains
`projectUuid`, skip to **Step 3 — Ontology Status Check**.

## Step 1 — Breeze MCP Authentication

The Breeze MCP server handles authentication on its own — no API
key is stored locally. Before calling any Breeze MCP tool, confirm
the session is authenticated.

1. Attempt a lightweight MCP call (e.g. `Call_List_Project_`).
2. If it succeeds → the MCP is already authenticated. Continue to
   Step 2.
3. If it fails with an auth error, guide the user through the MCP
   auth handshake:
   - Call `mcp__breeze-mcp__authenticate` to start the flow. The
     tool returns an authentication URL.
   - Share the URL with the user and ask them to complete sign-in
     in their browser.
   - Once they confirm they've finished, call
     `mcp__breeze-mcp__complete_authentication` to finalize the
     session.
   - Retry the lightweight MCP call to verify the session is live.

If authentication still fails after this flow, stop and ask the
user to retry `/breeze:setup-project` or check their network/SSO.

## Step 2 — Project Linking

If `projectUuid` is missing from `.breeze.json`:

Ask: "Would you like to:
1. Select an existing project
2. Create a new project"

**Option 1 — Select existing:**

- Call `Call_List_Project_`
- Display the project list (name + UUID)
- User selects one → save `projectUuid` to `.breeze.json`

**Option 2 — Create new:**

- Ask for project name and description (optional)
- Call `Call_Create_Project_` with name and description
- Save returned `projectUuid` to `.breeze.json`

Resulting `.breeze.json`:

    {
      "projectUuid": "<PROJECT_UUID>"
    }

Confirm: "Project linked successfully."

## Step 3 — Ontology Status Check

With `projectUuid` in hand, check what the project already contains
so the user knows where to go next.

1. Call `Call_Get_Project_Details_` to see what's already indexed.
   Do **not** call `Get_complete_functional_graph` here — it pulls
   the entire graph and is far too heavy for a status check.
2. Report the state plainly from the project details response:

   - **Code ontology:** present or missing
   - **Functional graph:** populated or empty
   - **Design graph:** present or missing (if applicable)

**Do NOT attempt to upload anything from this skill.** Just report
what's there.

## Step 4 — Next-step guidance

Based on Step 3's findings, point the user at the right follow-up.

### Greenfield project (no code yet)

If the user is starting from designs or documents:

> To seed Breeze without code:
>
> - **UI designs** → use `/breeze:visual-to-text` to convert Figma
>   frames / PDFs / screenshots into structured user stories.
> - **Requirement documents** → use `/breeze:analyze-functional`,
>   which ingests documents as part of its analysis flow and can
>   upsert the extracted intent into the functional graph.

### Existing project with a populated graph

If the graph is already populated:

> Your project is ready. You can:
>
> - `/breeze:search` to explore the graph
> - `/breeze:analyze-functional` to analyze a new requirement
> - `/breeze:analyze-architecture` to check architecture impact
> - `/breeze:analyze-design` to generate design graph nodes
> - `/breeze:generate-spec` to export a functional specification

## What this skill does NOT do

- **Upload or ingest documents** — use `/breeze:analyze-functional`
- **Convert visuals to user stories** — use `/breeze:visual-to-text`

Setup is intentionally narrow: authenticate the MCP, link the
project, tell the user where to go next. Every other responsibility
lives in its own skill.

## See also

- `/breeze:analyze-functional` — analyze requirements (including from
  documents) against the graph
- `/breeze:visual-to-text` — convert UI design visuals into user
  stories
