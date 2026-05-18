# agent-skills

A personal collection of agent skills for coding workflows. This repo serves as a plugin marketplace for Claude Code, GitHub Copilot CLI, and Codex.

## Available Skills

| Plugin | Skill | Description |
|--------|-------|-------------|
| tianqi-skills | plugin-scaffolder | Scaffold a multi-agent plugin repo with marketplace files, manifests, and skills for Claude Code, Copilot CLI, and Codex |

## Installation

### GitHub Copilot CLI

Direct install:
```
/plugin install TianqiZhang/agent-skills
```

Marketplace install:
```
/plugin marketplace add TianqiZhang/agent-skills
/plugin install tianqi-skills@tianqi-marketplace
```

Start a new Copilot CLI session if the installed plugin is not available in the current session.

### Claude Code

```
/plugin marketplace add TianqiZhang/agent-skills
/plugin install tianqi-skills@tianqi-marketplace
```

### Codex

```bash
codex plugin marketplace add TianqiZhang/agent-skills
```

Then install `tianqi-skills` from the Codex plugin directory for the registered marketplace.

## License

MIT
