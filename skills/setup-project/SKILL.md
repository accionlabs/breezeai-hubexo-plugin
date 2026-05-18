---
name: setup-project
description: >
  Backward-compatible alias for `/breeze:project setup`. Performs the
  full Breeze workspace bootstrap: MCP auth + project link/create +
  status report. Prefer `/breeze:project setup` going forward — this
  skill remains as a stable entry point for muscle memory and existing
  documentation.
---

## Scope

This skill is a backward-compatible alias. When invoked, execute the
same bootstrap flow that `/breeze:project setup` documents. The
behavior is intentionally identical.

The canonical home for this flow (and all other project management
operations — `show`, `list`, `use`, `create`, `auth`, `status`) is
`/breeze:project`. New documentation and prompts should point users
at `/breeze:project setup`; this skill remains so the older command
keeps working.

## Behavior — full bootstrap

Run these three phases in order. (For the canonical / future-proof
version, see Mode: setup in `/breeze:project`'s SKILL.md.)

### Phase 1 — MCP Authentication

1. Attempt a lightweight MCP call (`Call_List_Project_` with
   `limit: 1`) to test the current session.

2. If it succeeds → continue to Phase 2.

3. If it fails with an auth error:

   - Call `mcp__breeze-mcp__authenticate`. Share the returned URL
     with the user and ask them to complete sign-in in their browser.
   - Once they confirm, call
     `mcp__breeze-mcp__complete_authentication` to finalize.
   - Retry the lightweight call to verify. If verification still
     fails, ask the user to retry or check their network/SSO and
     stop.

### Phase 2 — Project Linking

Read `.breeze.json`. Remember whether `projectUuid` was already
present.

**No existing `projectUuid` (first-time link)** — ask:

    Would you like to:
    1. Select an existing project
    2. Create a new project

- **Select existing**: Call `Call_List_Project_`, display the list
  (name + UUID), let the user pick. Save the chosen UUID to
  `.breeze.json` (preserving any other keys).
- **Create new**: Ask for project name and optional description.
  Call `Call_Create_Project_`. Save the returned `projectUuid` to
  `.breeze.json` (preserving any other keys).

**Existing `projectUuid` (confirm / switch)** —

1. Call `Call_Get_Project_Details_` with the existing UUID to
   identify the current project.

2. Ask:

       You're currently linked to **{currentProject.name}**
       (`{currentProject.uuid}`). Would you like to:
       1. Keep this project (just show its status)
       2. Switch to a different existing project
       3. Create a new project

3. Route:

   - **Keep** → proceed to Phase 3.
   - **Switch** → run the "select existing" sub-flow above, then
     overwrite `projectUuid` in `.breeze.json`.
   - **Create new** → run the "create new" sub-flow above, then
     overwrite `projectUuid` in `.breeze.json`.

If the existing UUID returns empty / 404 from
`Call_Get_Project_Details_`, treat the link as gone and fall through
to the first-time link prompt above.

### Phase 3 — Status Report + Next Steps

1. Call `Call_Get_Project_Details_` with the active UUID. Render:

       Active project: <name>  (<uuid>)
       Status:         <status>
       Version:        <version>
       Author:         <author.firstName> <author.lastName> <author.email>
       Description:    <metadata.description or "—">

2. End with:

       Your project is linked. Next:

       - /breeze:project              view / list / switch project
       - /breeze:search <question>    explore the graph
       - /breeze:impact-analysis      cross-layer blast radius
       - /breeze:generate-spec        export a functional specification

       Each analysis skill also accepts --project <name|uuid> for a
       one-shot, non-persistent override.

## See also

- `/breeze:project setup` — canonical home for this exact flow
- `/breeze:project use <name|uuid>` — fast mid-session switch
- `/breeze:project auth` — re-authenticate without changing the link
- `/breeze:project list` — view all accessible projects
