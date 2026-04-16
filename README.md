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

- Setting up your API key (generated at https://ai.accionbreeze.com/mcp/generate/key)
- Linking to an existing project or creating a new one
- Checking ontology status
- Optionally uploading your repository or documents

Your credentials are saved to `.breeze.json` in the project root (gitignored).

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
| **Setup Project**      | `/breeze:setup-project`                  | Initialize the Breeze workspace — API key, project link, ontology status check, next-step guidance. Does **not** upload repos or documents.                                                                                                                           |
| **Onboard Repository** | `/breeze:onboard-repository [repo-path]` | Upload a source repository into the Breeze code graph. Wraps `breeze-code-ontology-generator` with `--capture-statements`, verifies Node.js 22+, and resolves the target repo from an argument or the current directory. Run once per repo (frontend + each backend). |

### Search & analysis

| Skill                    | Command                              | Description                                                                                                                                                          |
| ------------------------ | ------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Search**               | `/breeze:search <query>`             | Search the functional graph or code graph for feature discovery, impact analysis, and cross-cutting queries                                                          |
| **Impact Analysis**      | `/breeze:impact-analysis`            | Deep analysis across functional, design, and code graphs — summarizes cross-layer impact and optionally generates a detailed analysis document with Mermaid diagrams |
| **Analyze Functional**   | `/breeze:analyze-functional`         | Analyze a requirement against the existing functional graph — coverage gaps, conflicts, dependencies, impact                                                         |
| **Analyze Architecture** | `/breeze:analyze-architecture`       | Analyze a requirement against the architecture graph — impacted layers and components across 8 architecture layers                                                   |
| **Analyze Design**       | `/breeze:analyze-design <Figma URL>` | Analyze UI/UX designs from Figma frames — functional summary, components, mapping to the functional graph, gap flags                                                 |

### Generate from designs

| Skill              | Command                  | Description                                                                                                                                                |
| ------------------ | ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Visual to Text** | `/breeze:visual-to-text` | Generate user stories from UI design visuals (Figma frames, PDF screens, images) — outputs structured stories in persona/outcome/scenario/step/action form |

### Spec generation

| Skill             | Command                 | Description                                                           |
| ----------------- | ----------------------- | --------------------------------------------------------------------- |
| **Generate Spec** | `/breeze:generate-spec` | Generate functional specification documents from the functional graph |

### Recommended pipelines

```
First-time onboarding of a brownfield full-stack project:
  /breeze:setup-project                               # API key + project link
  /breeze:onboard-repository <frontend repo>          # index code graph (once per repo)
  /breeze:onboard-repository <backend repo 1>
  /breeze:onboard-repository <backend repo 2>
  ...
  /breeze:generate-spec

Greenfield project (no code yet):
  /breeze:setup-project
  /breeze:visual-to-text           # Figma / PDF / images → user stories
  /breeze:analyze-functional       # also ingests requirement documents

Adding a new requirement to an existing project:
  /breeze:impact-analysis
  /breeze:analyze-functional
  /breeze:analyze-architecture
  /breeze:analyze-design <Figma URL>
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

- `.breeze.json` contains your API key — add it to `.gitignore`
- The **Design** skill requires a Figma MCP server to be configured separately
- All skills except `setup-project` require a valid `.breeze.json` with `apiKey` and `projectUuid`
