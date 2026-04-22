---
name: setup-project
description: >
  Initialize or validate the Breeze workspace. Guides MCP
  authentication, links to a Breeze project, saves the projectUuid
  to .breeze.json, and reports project metadata. Also used to
  switch the linked project. Does NOT upload repos or documents.
  Use when: first time setup, "init breeze", "setup breeze",
  switching between projects, or when any Breeze tool fails with
  authentication errors.
---

## Scope

This skill is responsible for **workspace bootstrap only**:

- Breeze MCP authentication check
- Project linking (select existing, create new, or switch)
- Project metadata report
- Pointing the user at the right next-step skill

This skill does **NOT** upload repositories or documents, and it
does **NOT** probe individual graphs (functional / code / design /
architecture). Graph-level readiness is surfaced by `/breeze:search`
the first time it actually hits a graph.

## Prerequisites

Read `.breeze.json` from the project root. Remember whether a
`projectUuid` was already present — this drives Step 2 behaviour.

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

The behaviour here depends on whether `.breeze.json` already had a
`projectUuid` when this skill started.

### 2a. No existing `projectUuid` — first-time link

Ask:

> Would you like to:
> 1. Select an existing project
> 2. Create a new project

Handle the selected option (see **Selection flows** below), then
write `projectUuid` to `.breeze.json` and continue to Step 3.

### 2b. `projectUuid` already present — confirm / switch

The user running `/breeze:setup-project` on an already-linked
workspace is almost always trying to **switch** to a different
project (or sanity-check the current one). Do **not** silently skip
to the status report.

1. Call `Call_Get_Project_Details_` with the existing
   `projectUuid` so you can name the current project.
2. Ask:

   > You're currently linked to **{currentProject.name}**
   > (`{currentProject.uuid}`). Would you like to:
   > 1. Keep this project (just show its status)
   > 2. Switch to a different existing project
   > 3. Create a new project

3. Route on the answer:

   - **Keep** → continue to Step 3 with the existing `projectUuid`.
   - **Switch** → run the *Select existing* flow below, overwrite
     `projectUuid` in `.breeze.json`, then Step 3.
   - **Create new** → run the *Create new* flow below, overwrite
     `projectUuid` in `.breeze.json`, then Step 3.

If the existing `projectUuid` no longer resolves (Get Project
Details returns empty/404), tell the user the previously linked
project is gone and fall through to the *first-time link* prompt
in 2a.

### Selection flows

**Select existing:**

- Call `Call_List_Project_`
- Display the project list (name + UUID)
- User selects one → save `projectUuid` to `.breeze.json`

**Create new:**

- Ask for project name and description (optional)
- Call `Call_Create_Project_` with name and description
- Save the returned `projectUuid` to `.breeze.json`

### Resulting `.breeze.json`

Minimal shape:

    {
      "projectUuid": "<PROJECT_UUID>"
    }

If the file already had other keys (e.g. legacy `apiKey`,
`apiBase`, AWS creds), **preserve them** — only update
`projectUuid`.

Confirm: `Project linked successfully.` (or
`Switched to {name}.` when coming from 2b).

## Step 3 — Project Metadata Report

`Call_Get_Project_Details_` returns **only project metadata** — it
does NOT expose per-graph readiness flags. Report exactly the
fields it gives you. Do not probe individual graphs from this
skill; doing so inflates cost and lies about graph state when the
user only wanted a status ping.

Call `Call_Get_Project_Details_` (skip the call if you just made it
in Step 2b) and surface:

| Field | Source |
|---|---|
| Name | `data[0].name` |
| UUID | `data[0].uuid` |
| Internal ID | `data[0]._id` |
| Status | `data[0].status` |
| Version | `data[0].version` |
| Description | `data[0].metadata.description` (show "—" if empty) |
| Tags | `data[0].tags` (show "—" if empty) |
| User Story Status | `data[0].userStoryStatus` |
| Author | `data[0].author.firstName` + `lastName` + `email` |

If any of these fields are absent from the payload, omit that row.
Do **not** invent graph-readiness rows.

If the caller asks specifically whether graphs are populated, tell
them that `/breeze:search` or `/breeze:impact-analysis` will
surface that the first time it hits a graph.

## Step 4 — Next-step guidance

End with a short menu:

> Your project is linked. Next:
>
> - `/breeze:search <question>` — explore the graph (also tells you
>   which graphs are populated the first time it hits them)
> - `/breeze:impact-analysis <change>` — cross-layer blast-radius
> - `/breeze:generate-spec` — export a functional specification

Setup is intentionally narrow: authenticate the MCP, link or
switch the project, report project metadata, and hand off.
