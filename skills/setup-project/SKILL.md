---
name: setup-project
description: >
  Initialize or validate the Breeze workspace. Guides MCP
  authentication, links to a Breeze project, saves the projectUuid
  to .breeze.json, and checks ontology readiness. Does NOT upload
  repos or documents. Use when: first time setup, "init breeze",
  "setup breeze", or when any Breeze tool fails with authentication
  errors.
---

## Scope

This skill is responsible for **workspace bootstrap only**:

- Breeze MCP authentication check
- Project linking (select existing or create new)
- Ontology readiness check
- Pointing the user at the right next-step skill based on what they
  want to do

This skill does **NOT** upload repositories or documents.

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

### Existing project with a populated graph

If the graph is already populated:

> Your project is ready. You can:
>
> - `/breeze:search` to explore the graph
> - `/breeze:impact-analysis` to run cross-layer impact analysis
> - `/breeze:generate-spec` to export a functional specification

Setup is intentionally narrow: authenticate the MCP, link the
project, tell the user where to go next.
