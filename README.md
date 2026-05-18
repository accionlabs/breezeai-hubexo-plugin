# Breeze Plugin for Claude Code

A Claude Code plugin that integrates with the Breeze AI platform for functional graph management, code analysis, design analysis, and requirement tracing.

## Quick Start

### 1. Add the marketplace

In Claude Code, run:

```
/plugin marketplace add accionlabs/breezeai-hubexo-plugin
```

This registers the marketplace from GitHub. It does **not** install the plugin yet.

> For plugin developers / local testing, use a local-directory marketplace instead:
> `/plugin marketplace add /absolute/path/to/breezeai-hubexo-plugin`
> Claude Code writes `"source": "directory"` into `known_marketplaces.json` and re-installs always pull from that local path.

### 2. Install the plugin

```
/plugin install breeze@breezeai-plugins
```

Claude Code copies the plugin into your local plugin cache.

### 3. Reload (or restart) Claude Code

```
/reload-plugins
```

Plugin skills, hooks, and MCP servers are picked up. Restart only if `/reload-plugins` doesn't surface the skills (rare).

You can confirm everything loaded by running:

```
/plugin list
```

`breeze` should show as installed and enabled. The available skills will appear with the `/breeze:*` namespace.

### 4. Initialize the workspace

```
/breeze:project setup
```

This walks you through:

- Authenticating with the Breeze MCP (browser-based Keycloak sign-in; no API key to paste)
- Linking to an existing project or creating a new one
- Checking ontology status

The linked `projectUuid` is saved to `.breeze.json` in the project root (gitignored). The MCP session is authenticated separately — no credentials are stored on disk. The Keycloak token lasts roughly **7 days** from the last successful sign-in; after that, any `/breeze:*` skill will prompt you to re-run `/breeze:project auth`.

## Available Skills

### Project management

| Skill | Command | Description |
|---|---|---|
| **Project** | `/breeze:project [show \| list \| use <name> \| create <name> \| auth \| status \| setup]` | Canonical home for all project management. Sub-modes: `show` (no args, print active project), `list` (all accessible projects), `use <name\|uuid>` (switch and persist), `create <name> [--desc "..."]` (create a new project and link), `auth` (re-authenticate MCP), `status` (full metadata report — alias of show), `setup` (full bootstrap = auth + link/create + status). The `auth` and `setup` modes handle the MCP auth handshake; all others assume an active session. |
| **Setup Project** | `/breeze:setup-project` | Backward-compatible alias for `/breeze:project setup`. Identical behavior. Prefer `/breeze:project setup` for new docs and prompts. |

### Search & analysis

| Skill | Command | Description |
|---|---|---|
| **Search** | `/breeze:search <query>` | Smart-search across functional, design, code, and architecture graphs — routes to one or many based on query intent. Default entry point for any question about the project. |
| **Impact Analysis** | `/breeze:impact-analysis` | Original impact analysis. Cross-layer summary + optional detailed document with Mermaid diagrams. |
| **Impact Analysis v2** | `/breeze:impact-analysis-v2 [--detailed] <prompt>` | Newer, more structured impact analysis. Same cross-layer scope, plus: legend on every detailed doc, marker tables for all four ontologies, structured Risk Taxonomy (Why / Worst case / Mitigation per risk), concrete QA Test Plan as a table, Implementation Options (A/B/C with pros/cons), Multi-Service Deploy Coordination, Open Questions, Out of Scope sections, Schema-Side Impact (Data Context block) with 🔴/🟡/🟢 verdict. Pass `--detailed` to skip the summary and emit the full document directly. |

### Spec generation

| Skill | Command | Description |
|---|---|---|
| **Generate Spec** | `/breeze:generate-spec` | Generate functional specification documents from the functional graph. |

## Project resolution

Every analysis skill (`search`, `impact-analysis`, `impact-analysis-v2`, `generate-spec`) resolves the active project for each invocation using the flow below. The full rule lives in `CLAUDE.md` at the plugin root — skills defer to it rather than duplicating logic.

### How a skill picks the project

1. **Explicit `--project <uuid|name>` argument** anywhere in the prompt. UUIDs are used directly. Names are resolved via `Call_List_Project_` (case-insensitive substring match).
2. **Bare UUID** anywhere in the prompt (matches the standard UUID pattern). Used directly, no lookup.
3. **Natural-language hint** at the start of the prompt — `for <project>: ...`, `on <project>: ...`, `<project> impact: ...`, `in the <project> project ...`. Resolved the same way as `--project`.
4. **`.breeze.json` fallback** — the persisted default, written by `/breeze:project setup` / `use` / `create`.
5. **If nothing matches** — the skill calls `Call_List_Project_`, shows the accessible projects, and asks you to pick one. No hard error.

Resolution outcomes for (1), (2), (3):
- **Exactly one match** → use that UUID for the invocation only. `.breeze.json` is NOT touched.
- **Multiple matches** → the skill lists them numbered (name + UUID) and waits for your pick.
- **Zero matches** → falls through to step 4 (the prompt may not contain a project hint at all).

### Persistent switch vs one-shot override

```
# One-shot — uses Lead Manager V2 for this invocation only
/breeze:search --project "Lead Manager V2" how does keyword search work

# Persistent — writes Lead Manager V2 to .breeze.json
/breeze:project use "Lead Manager V2"
```

`.breeze.json` is **only** mutated by `/breeze:project setup`, `/breeze:project use`, and `/breeze:project create`. Per-invocation overrides (steps 1-3 above) never touch the file.

### Cross-project parallel runs

Different terminals or Claude Code sessions can target different projects simultaneously by using `--project` (or the natural-language hint) on each invocation — no need to re-link `.breeze.json` between them.

## Recommended pipelines

```
First-time setup:
  /breeze:project setup                               # MCP auth + project link
  /breeze:generate-spec                               # if the project already has a graph

Adding a new requirement to an existing project:
  /breeze:impact-analysis-v2                          # newer, structured output
  /breeze:search <query>

Switching projects mid-session:
  /breeze:project list                                # see what's available
  /breeze:project use "Lead Manager V2"               # persist the switch

Cross-project comparison without persisting:
  /breeze:search --project v1 how does X work
  /breeze:search --project v2 how does X work

Re-authenticate after the 7-day Keycloak token expires:
  /breeze:project auth
```

## CLAUDE.md preamble

The plugin ships a `CLAUDE.md` at the marketplace root. When you install the plugin into Claude Code, that file lives at `~/.claude/plugins/cache/breezeai-plugins/breeze/<version>/CLAUDE.md`. Claude Code auto-loads it for sessions running in that directory.

It centralizes:

- The project-resolution flow described above
- The exempt-skills list (which skills don't need a configured project)
- The auth handling rule (what to do on a 401)
- The persistent-mapping rule (which skill modes are allowed to mutate `.breeze.json`)

Individual skill files (`skills/*/SKILL.md`) defer to `CLAUDE.md` rather than duplicating the rule, so the behavior stays consistent and there's one place to update.

## Updating the plugin

When a new version is released:

```
/plugin marketplace update breezeai-plugins
/plugin uninstall breeze@breezeai-plugins
/plugin install breeze@breezeai-plugins
/reload-plugins
```

The uninstall-then-install pair is required because `/plugin install` is a no-op on an already-installed plugin — it won't refresh the cached copy on its own.

## Local development (plugin maintainers)

For active development of the plugin itself:

1. **Point the marketplace at your local working clone:**
   ```
   /plugin marketplace remove breezeai-plugins
   /plugin marketplace add /absolute/path/to/breezeai-hubexo-plugin
   /plugin install breeze@breezeai-plugins
   ```
   Claude Code writes `"source": "directory"` into `known_marketplaces.json` and the install cache is sourced from your local path.

2. **After editing skill files, refresh the install cache:**
   ```
   /plugin uninstall breeze@breezeai-plugins
   /plugin install breeze@breezeai-plugins
   /reload-plugins
   ```

3. **For zero-command edit cycles** (advanced), symlink the install cache to the working clone:
   ```
   rm -rf ~/.claude/plugins/cache/breezeai-plugins/breeze/<version>
   ln -s /absolute/path/to/breezeai-hubexo-plugin ~/.claude/plugins/cache/breezeai-plugins/breeze/<version>
   ```
   After that, every edit is immediately reflected; only `/reload-plugins` is needed. Caveat: re-running `/plugin install` blows away the symlink.

## Notes

- `.breeze.json` holds only the linked `projectUuid` and the API key for non-MCP CLI consumers. It is gitignored by default. MCP authentication uses Keycloak SSO via Claude Code — no API key or secret needs to be pasted for MCP tools.
- All skills except `/breeze:project` (in `setup`, `auth`, `list`, `create` modes) and the legacy `/breeze:setup-project` alias require a project to be resolvable (see Project resolution above). If a Breeze MCP tool call fails with an auth error, run `/breeze:project auth`.
- The Keycloak token expires after roughly 7 days. Expect to re-authenticate periodically; the skills will tell you when.
- The `--detailed` flag is supported only by `/breeze:impact-analysis-v2` today. It skips the summary and emits the full diagrammed document directly.
- The **Design** skill requires a Figma MCP server to be configured separately.
