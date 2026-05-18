---
name: plugin-scaffolder
description: >
  Scaffold a plugin repository that works as a plugin marketplace and plugin source for one, two,
  or all three coding agents: Claude Code, GitHub Copilot CLI, and Codex (OpenAI). Generates
  marketplace files, plugin manifests, shared skills, MCP server configs, Copilot direct-install
  shim, and README with install instructions — only for the agents the user selects. Use this skill
  whenever the user wants to create a plugin repo, plugin marketplace, agent plugin, coding agent
  plugin, or multi-agent plugin structure. Also trigger when users mention scaffolding plugin files,
  creating .claude-plugin, .codex-plugin, .github/plugin, or marketplace.json — even if they only
  name one agent.
---

# Plugin Scaffolder

Generate a plugin repository that one, two, or all three major coding agents (Claude Code, GitHub Copilot CLI, and Codex) can discover, install, and use.

The core idea: one shared plugin root for runtime assets (skills, MCP configs), with agent-specific manifests and marketplace files that point to that shared root. Only generate files for the agents the user wants to target.

## Interactive Flow

When invoked, gather the following information from the user via `AskUserQuestion`. Collect as much as you can from context before asking — if the user already provided details, skip those questions.

### Step 1: Repo metadata

Ask for:
- **Target agents** — which agents should this repo support? Present as a multi-select: Claude Code, GitHub Copilot CLI, Codex. Default: all three selected. If the user's request already names specific agents (e.g., "create a Claude Code plugin"), pre-select only those and confirm.
- **GitHub owner/repo** (e.g., `myorg/my-plugins`) — used in homepage, repository URLs, and README install commands
- **Marketplace name** (e.g., `my-marketplace`) — the name marketplace files use
- **Marketplace display name** (e.g., `My Plugins`) — human-readable label
- **Owner name and email** (e.g., `My Team`, `plugins@example.com`)
- **License** (default: `MIT`)

### Step 2: Plugin details

Ask how many plugins to include (default: 1). For each plugin, ask:
- **Plugin name** (e.g., `my-plugin`) — used as directory name and in all manifests
- **Plugin description** (one sentence)
- **Plugin version** (default: `1.0.0`)
- **Codex category** (default: `Productivity`) — only ask if Codex is a target agent
- **Keywords** (default: `["coding-agent", "plugin"]`)

### Step 3: Optional components

For each plugin, ask which optional components to include:
- **Skills** — if yes, ask for skill names (at least one). Each becomes `plugins/<plugin>/skills/<skill-name>/SKILL.md`
- **MCP server configs** — if yes, ask for server names and commands. Generates `.mcp.json` (for Claude/Copilot) and/or `.codex.mcp.json` (for Codex), depending on target agents
- **Copilot direct-install shim** — only offer if Copilot CLI is a target agent. `.github/plugin/plugin.json` so `/plugin install owner/repo` works
- **README** — with install instructions for the selected target agents

Present these as a multi-select with all applicable options selected by default.

## File Generation

After collecting inputs, generate only the files relevant to the selected target agents. Use the exact structures below — these are validated against the latest agent docs and local testing.

### Which files to generate per agent

| File | Claude Code | Copilot CLI | Codex |
|------|:-----------:|:-----------:|:-----:|
| `.claude-plugin/marketplace.json` | Yes | — | — |
| `.github/plugin/marketplace.json` | — | Yes | — |
| `.agents/plugins/marketplace.json` | — | — | Yes |
| `.github/plugin/plugin.json` (shim) | — | Optional | — |
| `plugins/<name>/plugin.json` | — | Yes | — |
| `plugins/<name>/.claude-plugin/plugin.json` | Yes | — | — |
| `plugins/<name>/.codex-plugin/plugin.json` | — | — | Yes |
| `plugins/<name>/.mcp.json` | Yes | Yes | — |
| `plugins/<name>/.codex.mcp.json` | — | — | Yes |
| `plugins/<name>/skills/` | Yes | Yes | Yes |
| `README.md` | Yes | Yes | Yes |
| `LICENSE` | Yes | Yes | Yes |

Skills, README, and LICENSE are shared — always generate them regardless of agent selection. Only generate marketplace files, manifests, and MCP configs for the selected agents.

When only one or two agents are targeted, the directory tree is simpler. For example, a Claude Code-only plugin skips `.agents/`, `.github/plugin/`, `.codex-plugin/`, `.codex.mcp.json`, and the Copilot-style `plugin.json` at the plugin root.

### Directory layout (all three agents selected)

For a repo with one plugin called `my-plugin` and one skill called `my-skill`:

```
repo/
|-- .agents/
|   `-- plugins/
|       `-- marketplace.json              # Codex marketplace
|-- .claude-plugin/
|   `-- marketplace.json                  # Claude Code marketplace
|-- .github/
|   `-- plugin/
|       |-- marketplace.json              # Copilot CLI marketplace
|       `-- plugin.json                   # Copilot direct-install shim (optional)
|-- plugins/
|   `-- my-plugin/
|       |-- plugin.json                   # Copilot CLI plugin manifest
|       |-- .claude-plugin/
|       |   `-- plugin.json               # Claude Code plugin manifest
|       |-- .codex-plugin/
|       |   `-- plugin.json               # Codex plugin manifest
|       |-- .mcp.json                     # Claude/Copilot MCP config (optional)
|       |-- .codex.mcp.json              # Codex MCP config (optional)
|       `-- skills/
|           `-- my-skill/
|               `-- SKILL.md              # Shared skill file
|-- README.md                             # Install docs (optional)
`-- LICENSE
```

For multiple plugins, repeat the `plugins/<name>/` subtree for each one. All marketplace files list every plugin.

### Marketplace files

#### Codex: `.agents/plugins/marketplace.json`

```json
{
  "name": "<marketplace-name>",
  "interface": {
    "displayName": "<marketplace-display-name>"
  },
  "plugins": [
    {
      "name": "<plugin-name>",
      "source": {
        "source": "local",
        "path": "./plugins/<plugin-name>"
      },
      "policy": {
        "installation": "AVAILABLE",
        "authentication": "ON_INSTALL"
      },
      "category": "<codex-category>"
    }
  ]
}
```

Important: `source.path` must point to a plugin subfolder like `./plugins/<name>`. Using `./` alone does not reliably surface plugins in Codex.

Note: Codex also reads `.claude-plugin/marketplace.json` as a legacy-compatible marketplace. If the repo targets both Claude Code and Codex, the Claude marketplace file may be discovered by Codex automatically — but keep `.agents/plugins/marketplace.json` as the explicit Codex-facing catalog.

#### Claude Code: `.claude-plugin/marketplace.json`

```json
{
  "name": "<marketplace-name>",
  "owner": {
    "name": "<owner-name>",
    "email": "<owner-email>"
  },
  "plugins": [
    {
      "name": "<plugin-name>",
      "source": "./plugins/<plugin-name>",
      "description": "<plugin-description>",
      "version": "<plugin-version>",
      "author": {
        "name": "<owner-name>"
      }
    }
  ]
}
```

#### Copilot CLI: `.github/plugin/marketplace.json`

```json
{
  "name": "<marketplace-name>",
  "owner": {
    "name": "<owner-name>",
    "email": "<owner-email>"
  },
  "metadata": {
    "description": "<marketplace-display-name>",
    "version": "1.0.0"
  },
  "plugins": [
    {
      "name": "<plugin-name>",
      "description": "<plugin-description>",
      "version": "<plugin-version>",
      "source": "./plugins/<plugin-name>"
    }
  ]
}
```

### Plugin manifests

#### Copilot CLI: `plugins/<plugin>/plugin.json`

```json
{
  "name": "<plugin-name>",
  "description": "<plugin-description>",
  "version": "<plugin-version>",
  "author": {
    "name": "<owner-name>",
    "email": "<owner-email>"
  },
  "homepage": "https://github.com/<owner>/<repo>",
  "repository": "https://github.com/<owner>/<repo>",
  "license": "<license>",
  "keywords": <keywords>,
  "skills": ["skills/"],
  "mcpServers": ".mcp.json"
}
```

If no MCP servers, omit `"mcpServers"`. If no skills, omit `"skills"`.

#### Claude Code: `plugins/<plugin>/.claude-plugin/plugin.json`

```json
{
  "name": "<plugin-name>",
  "description": "<plugin-description>",
  "version": "<plugin-version>",
  "author": {
    "name": "<owner-name>"
  },
  "homepage": "https://github.com/<owner>/<repo>",
  "repository": "https://github.com/<owner>/<repo>",
  "license": "<license>",
  "keywords": <keywords>,
  "skills": "./skills/",
  "mcpServers": "./.mcp.json"
}
```

Paths must start with `./`. Omit `"mcpServers"` or `"skills"` if not included.

#### Codex: `plugins/<plugin>/.codex-plugin/plugin.json`

```json
{
  "name": "<plugin-name>",
  "description": "<plugin-description>",
  "version": "<plugin-version>",
  "author": {
    "name": "<owner-name>"
  },
  "homepage": "https://github.com/<owner>/<repo>",
  "repository": "https://github.com/<owner>/<repo>",
  "license": "<license>",
  "keywords": <keywords>,
  "skills": "./skills/",
  "mcpServers": "./.codex.mcp.json",
  "interface": {
    "displayName": "<plugin-display-name>",
    "shortDescription": "<plugin-description>",
    "longDescription": "<plugin-description>",
    "developerName": "<owner-name>",
    "category": "<codex-category>"
  }
}
```

Codex uses a separate `.codex.mcp.json` because its wrapper key differs from Claude/Copilot. Omit `"mcpServers"` or `"skills"` if not included.

### Copilot direct-install shim: `.github/plugin/plugin.json`

Only generate if the user selected this option. This enables `/plugin install owner/repo` in Copilot CLI. Paths are relative to the repo root, not the plugin root.

For a single plugin:

```json
{
  "name": "<plugin-name>",
  "description": "<plugin-description>",
  "version": "<plugin-version>",
  "author": {
    "name": "<owner-name>"
  },
  "homepage": "https://github.com/<owner>/<repo>",
  "repository": "https://github.com/<owner>/<repo>",
  "license": "<license>",
  "keywords": <keywords>,
  "skills": ["plugins/<plugin-name>/skills/"],
  "mcpServers": "plugins/<plugin-name>/.mcp.json"
}
```

For multiple plugins, point to the first plugin's assets (Copilot direct install only supports one plugin). Note in the README that marketplace install is preferred for multi-plugin repos.

Copilot CLI checks manifest locations in order: `.plugin/plugin.json`, root `plugin.json`, `.github/plugin/plugin.json`, `.claude-plugin/plugin.json`. The shim only activates if the first two are absent.

### MCP server configs

#### Claude/Copilot: `plugins/<plugin>/.mcp.json`

```json
{
  "mcpServers": {
    "<server-name>": {
      "command": "<command>",
      "args": ["--stdio"]
    }
  }
}
```

#### Codex: `plugins/<plugin>/.codex.mcp.json`

Codex uses a flat map without the `mcpServers` wrapper:

```json
{
  "<server-name>": {
    "command": "<command>",
    "args": ["--stdio"]
  }
}
```

### Skill files

Generate `plugins/<plugin>/skills/<skill-name>/SKILL.md` with frontmatter:

```markdown
---
name: <skill-name>
description: <brief description — the user should fill this in>
---

# <Skill Name>

Follow these instructions when this skill is triggered.

<!-- Replace this with the actual skill instructions -->
```

Include both `name` and `description` in frontmatter — this is the safest cross-agent approach.

### README.md

Generate a README with:
1. Repo title and description
2. Available plugins table (name, description, version)
3. Install instructions only for the selected target agents, using the actual owner/repo and marketplace/plugin names

**Example install section:**

```markdown
## Installation

### GitHub Copilot CLI

Direct install (single plugin):
\`\`\`
/plugin install <owner>/<repo>
\`\`\`

Marketplace install:
\`\`\`
/plugin marketplace add <owner>/<repo>
/plugin install <plugin-name>@<marketplace-name>
\`\`\`

### Claude Code

\`\`\`
/plugin marketplace add <owner>/<repo>
/plugin install <plugin-name>@<marketplace-name>
\`\`\`

### Codex

\`\`\`bash
codex plugin marketplace add <owner>/<repo>
\`\`\`

Then install from the Codex plugin directory.
```

### LICENSE

Generate an MIT license file (or the user's chosen license) with the current year and owner name.

## Path rules

These matter and are a common source of bugs:

| Context | Paths relative to |
|---------|-------------------|
| Marketplace `source` fields | Repository root |
| Plugin manifest `skills`/`mcpServers` fields | Plugin root (`plugins/<name>/`) |
| Copilot shim `skills`/`mcpServers` fields | Repository root |
| Claude/Codex manifest paths | Must start with `./` |

## After generation

Once all files are written:

1. List every file created so the user can review
2. Remind them to validate JSON: all `.json` files should parse cleanly
3. Suggest local testing commands for each targeted agent:

**Claude Code:**
```
claude --plugin-dir <path-to-repo>/plugins/<plugin-name>
```
Then verify with `/help` and `/<plugin-name>:<skill-name>`. Use `/reload-plugins` after edits. Marketplace test:
```
/plugin marketplace add <path-to-repo>
/plugin install <plugin-name>@<marketplace-name>
```

**Copilot CLI:**
```
copilot plugin install <path-to-repo>/plugins/<plugin-name>
```
Then verify with `/plugin list`, `/skills list`, `/mcp`. If the shim exists, also test: `copilot plugin install <path-to-repo>`. Start a new Copilot CLI session if the installed plugin is not visible. Marketplace test:
```
copilot plugin marketplace add <path-to-repo>
copilot plugin install <plugin-name>@<marketplace-name>
```

**Codex:**
```
codex plugin marketplace add <path-to-repo>
```
Then restart Codex, open the plugin directory, and install from the marketplace. Useful maintenance:
```
codex plugin marketplace upgrade
codex plugin marketplace remove <marketplace-name>
```
If Codex says the marketplace was added but the plugin doesn't appear, check that the marketplace entry points to a subfolder like `./plugins/<name>`, not `./`.

4. Mention the maintenance checklist: keep versions aligned across all manifests and marketplace entries, test in all targeted clients before publishing, update README install commands if owner/repo or names change

## Common mistakes to watch for

- Putting skills or `.mcp.json` inside `.claude-plugin/` or `.codex-plugin/` — only `plugin.json` belongs there
- Using `mcp.json` instead of `.mcp.json` (dot prefix required)
- Using `source.path: "./"` in Codex marketplace — always use a subfolder like `./plugins/<name>`
- Forgetting that marketplace paths resolve from repo root, not from the directory containing `marketplace.json`
- Generating one marketplace file for all agents — each agent needs its own
