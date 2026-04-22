# Breeze Plugin for Claude Code

A Claude Code plugin that integrates with the Breeze AI platform for functional graph management, code analysis, design analysis, and requirement tracing.

## Quick Start

### 1. Add the marketplace

In Claude Code, run:

```
/plugin marketplace add accionlabs/breezeai-hubexo-plugin
```

This registers the marketplace but does **not** install the plugin yet.

### 2. Install the plugin

```
/plugin install breeze
```

Pick `breeze` from the list when prompted. Claude Code downloads the plugin into your local plugin directory.

### 3. Activate the plugin

```
/plugin enable breeze
```

(If installation auto-enables it, this step is a no-op.)

### 4. Restart Claude Code

Plugin skills, hooks, and MCP servers are loaded at startup. **Quit Claude Code and start it again** so the new skills appear under `/breeze:*` and the Breeze MCP server is registered.

You can confirm everything loaded by running:

```
/plugin list
```

`breeze` should show as installed and enabled.

### 5. Initialize the workspace

```
/breeze:setup-project
```

This walks you through:

- Authenticating with the Breeze MCP (browser-based sign-in; no API key to paste)
- Linking to an existing project or creating a new one
- Checking ontology status

The linked `projectUuid` is saved to `.breeze.json` in the project root (gitignored). The MCP session is authenticated separately — no credentials are stored on disk.

### Updating the plugin

When a new version is released:

```
/plugin marketplace update accionlabs/breezeai-claude-plugin
/plugin update breeze
```

Then **restart Claude Code** again so the updated skills/hooks/MCP definitions are picked up.

## Available Skills

### Setup

| Skill                  | Command                                  | Description                                                                                                                                                                                                                                                           |
| ---------------------- | ---------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Setup Project** | `/breeze:setup-project` | Initialize the Breeze workspace — MCP authentication, project link, ontology status check, next-step guidance. Does **not** upload repos or documents. |

### Search & analysis

| Skill                    | Command                              | Description                                                                                                                                                          |
| ------------------------ | ------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Search**               | `/breeze:search <query>`             | Smart-search across functional, design, code, and architecture graphs — routes to one or many based on query intent. Default entry point for any question about the project. |
| **Impact Analysis**      | `/breeze:impact-analysis`            | Deep analysis across functional, design, and code graphs — summarizes cross-layer impact and optionally generates a detailed analysis document with Mermaid diagrams |

### Spec generation

| Skill             | Command                 | Description                                                           |
| ----------------- | ----------------------- | --------------------------------------------------------------------- |
| **Generate Spec** | `/breeze:generate-spec` | Generate functional specification documents from the functional graph |

### Recommended pipelines

```
First-time setup:
  /breeze:setup-project                               # MCP auth + project link
  /breeze:generate-spec                               # if the project already has a graph

Adding a new requirement to an existing project:
  /breeze:impact-analysis
  /breeze:search <query>
```

## Auto-Loading (No Flag Needed)

To avoid passing `--plugin-dir` every time, add this to your project's `.claude/settings.json`:

```json
{
  "plugins": ["./breeze-claude-plugin"]
}
```

## Setup for Teams

### Option A: Plugin inside your project repo

Place the `breeze-claude-plugin/` folder in your project repo. Everyone who clones the repo has it.

### Option B: Separate shared repo

Clone the plugin repo alongside your project:

```bash
git clone git@github.com:accionlabs/breeze-claude-plugin.git
claude --plugin-dir ../breeze-claude-plugin
```

## Notes

- `.breeze.json` holds only the linked `projectUuid`. It is gitignored by default. The Breeze MCP is authenticated separately via a browser sign-in flow — no API key or secret is stored on disk.
- The **Design** skill requires a Figma MCP server to be configured separately
- All skills except `setup-project` require a valid `.breeze.json` with a `projectUuid` and an authenticated Breeze MCP session. If a Breeze tool call fails with an auth error, re-run `/breeze:setup-project`.
