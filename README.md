# agent-skills

A personal collection of agent skills for coding workflows. This repo serves as a plugin marketplace for Claude Code, GitHub Copilot CLI, and Codex.

## Available Skills

| Plugin | Skill | Description | Source | Sync |
|--------|-------|-------------|--------|------|
| tianqi-skills | plugin-scaffolder | Scaffold a multi-agent plugin repo with marketplace files, manifests, and skills for Claude Code, Copilot CLI, and Codex | Original | Manual |
| tianqi-skills | skill-creator | Create, edit, evaluate, and optimize agent skills | `https://github.com/anthropics/skills`, path `skills/skill-creator`, ref `origin/main`, last synced `6a5bb06904ab164a345e41c381fc9097954b83da` | Fetch source, copy `skills/skill-creator` to `plugins\tianqi-skills\skills\skill-creator`, preserve `LICENSE.txt`, then update this row's commit |
| tianqi-skills | test-health-audit | Review test suites to identify low-value tests that lock down implementation details instead of real specifications | Original | Manual |

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

### Non-vendored external skills

Some external skills are not copied into this repository because their license terms do not permit redistribution. Install them directly from their source repository in a new environment:

```bash
# docx: create, read, edit, and analyze Word .docx documents.
npx skills add https://github.com/anthropics/skills --skill docx
# pdf: read, create, edit, merge, split, OCR, and process PDF files.
npx skills add https://github.com/anthropics/skills --skill pdf
# pptx: create, read, edit, and analyze PowerPoint .pptx presentations.
npx skills add https://github.com/anthropics/skills --skill pptx
# xlsx: create, read, edit, analyze, and convert spreadsheet files.
npx skills add https://github.com/anthropics/skills --skill xlsx
# guizang-ppt-skill: generate polished single-file HTML slide decks, images, and social covers.
npx skills add https://github.com/op7418/guizang-ppt-skill --skill guizang-ppt-skill
```

## License

MIT for this repository's original content. Imported skills retain the terms in their included `LICENSE.txt` files.
