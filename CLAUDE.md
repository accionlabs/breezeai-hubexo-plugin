# Breeze (Hubexo) plugin — Claude Code preamble

Auto-loaded by Claude Code at the start of every session in this repo. Central precondition for skills under `skills/`. Skills should NOT duplicate this — they defer to this file.

## When a skill needs a project

If a skill needs `projectUuid` to do its work, resolve it using the flow below.

### Step 1: Derive from the prompt

Look for a project reference in the current invocation. Three forms are accepted, in this order of preference:

- **`--project <value>` (or `--project=<value>`)** anywhere in `$ARGUMENTS`.
- **Bare UUID** matching `[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}` anywhere in the prompt — use directly, no lookup needed.
- **Project name or substring** appearing in the prompt (e.g., *"for Lead Manager V2: ..."*, *"on OneBid: ..."*, *"... in the ETL & Harvestor project"*). Call `Call_List_Project_` and match against accessible project names (case-insensitive substring).

Then resolve:

- **Exactly one match** → use that UUID for this invocation only. Do NOT mutate `.breeze.json`. Strip the override tokens (`--project <value>` or the natural-language project phrase) from `$ARGUMENTS` before processing the rest of the prompt.
- **Multiple matches** → list them numbered, with names + UUIDs, and ask the user to pick one. Wait for the choice before continuing.
- **Zero matches / no project mentioned** → fall through to Step 2. Do **not** error — many prompts simply don't include a project hint.

This is the mechanism for cross-project parallel skill runs (different terminals / Claude sessions targeting different projects without re-linking `.breeze.json`).

### Step 2: `.breeze.json` fallback

If Step 1 didn't resolve a project, read `.breeze.json` from the repo root:

- **Has `projectUuid`** → use it.
- **File missing OR `projectUuid` absent** → call `Call_List_Project_`, present the accessible projects numbered with names + UUIDs, and ask the user to pick one. Use that UUID for this invocation only — do NOT mutate `.breeze.json`. To persist the choice, the user should follow up with `/breeze:project use <name|uuid>`.

### Step 3: Announce the active project

Begin the response with one line so the user can verify scope at a glance:

    Project: <name> (<uuid>)

## Persistent project mapping

`.breeze.json` is mutated ONLY by these skill modes:

- `/breeze:project setup` — initial link (or full bootstrap).
- `/breeze:project use <name|uuid>` — switch the persistent default.
- `/breeze:project create <name>` — create a new project and link it.

A successful `--project` override, bare UUID, or natural-language hint applies to that invocation only; it does not change `.breeze.json`.

## Auth (Breeze MCP)

MCP tool calls authenticate via Keycloak SSO, handled by Claude Code at sign-in. Tokens last roughly 7 days from the last successful handshake.

If a Breeze MCP tool call fails with a 401 / unauthenticated error (including the very first call when the session has never been authenticated), stop and tell the user:

> *"Breeze MCP server requires authentication. Please open this URL in your browser to authorize..."*

Claude Code typically surfaces the actual login URL automatically when an MCP server needs auth. If it doesn't appear in the user's terminal, escalate via `/breeze:project auth` to trigger the handshake explicitly.

The `apiKey` in `.breeze.json` is for non-MCP CLI consumers (the ontology-generator CLI, REST upsert path), not for MCP tool calls.

## Skills that don't need a project

These skills are exempt from the resolution flow above — run them as requested:

| Skill / mode | Why exempt |
|---|---|
| `/breeze:project setup` | Creates `.breeze.json`. |
| `/breeze:setup-project` | Deprecated alias for `/breeze:project setup`. Same exemption. |
| `/breeze:project auth` | OAuth handshake only. |
| `/breeze:project list` | Lists accessible projects. |
| `/breeze:project create <name>` | Creates a new project and links it. |

All other skills are project-bound and follow Steps 1–3 above.

## Notes

- Never print API keys or AWS credentials in output.
- When adding a new skill to `skills/`: if it makes any Breeze MCP call, it is project-bound by default (no inline preflight needed). If it genuinely doesn't need a project, add it to the exempt table above in the same commit.
