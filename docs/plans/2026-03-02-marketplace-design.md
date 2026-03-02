# EdgeDelta Claude Code Plugin Marketplace — Design Document

**Date**: 2026-03-02
**Status**: Approved
**Scope**: Initial launch — 2 plugins

---

## Overview

A GitHub-hosted Claude Code plugin marketplace (`edgedelta/claude-code-plugins`) distributing official EdgeDelta plugins. Customers install plugins via Claude Code's marketplace system to get AI-powered observability assistance directly in their development environment.

---

## Repository Identity

- **GitHub repo**: `edgedelta/claude-code-plugins`
- **Local path**: `/Users/danielbright/local_dev/edgedelta-marketplace`
- **Marketplace source**: `github` / `edgedelta/claude-code-plugins`

---

## Directory Structure

```
edgedelta/claude-code-plugins/
├── .claude-plugin/
│   └── marketplace.json               # Top-level marketplace manifest
├── plugins/
│   ├── edgedelta-ottl/                # Plugin 1
│   │   ├── .claude-plugin/
│   │   │   └── plugin.json
│   │   └── skills/
│   │       └── edgedelta-ottl/
│   │           ├── SKILL.md
│   │           ├── MASTER_INDEX.md
│   │           ├── validation_rules.md
│   │           ├── CHANGELOG.md
│   │           └── references/
│   │               ├── syntax/
│   │               ├── edx/
│   │               └── standard/
│   └── edgedelta-pipelines/           # Plugin 2
│       ├── .claude-plugin/
│       │   └── plugin.json
│       └── skills/
│           └── edgedelta-pipelines/
│               ├── SKILL.md
│               ├── CHANGELOG.md
│               └── assets/
│                   ├── templates/     # 7 production pipeline YAML templates
│                   ├── scripts/       # Python validation/deployment scripts
│                   └── references/    # Validation rules, best practices
├── docs/
│   └── plans/
│       └── 2026-03-02-marketplace-design.md
└── README.md                          # Includes ASCII art header
```

---

## Marketplace Manifest

**`.claude-plugin/marketplace.json`**:

```json
{
  "name": "edge-delta-official-plugins",
  "owner": {
    "name": "Edge Delta",
    "email": "plugins@edgedelta.com"
  },
  "metadata": {
    "description": "Official Edge Delta plugins for observability, SRE, and DevOps workflows in Claude Code",
    "version": "1.0.0",
    "pluginRoot": "./plugins"
  },
  "plugins": [
    {
      "name": "edgedelta-ottl",
      "source": "./plugins/edgedelta-ottl",
      "category": "developer-tools",
      "version": "1.0.0"
    },
    {
      "name": "edgedelta-pipelines",
      "source": "./plugins/edgedelta-pipelines",
      "category": "configuration",
      "version": "1.0.0"
    }
  ]
}
```

---

## Plugin Manifests

Each plugin's `.claude-plugin/plugin.json` uses standard metadata only (no custom path overrides — auto-discovery handles everything):

**edgedelta-ottl**:
```json
{
  "name": "edgedelta-ottl",
  "version": "1.0.1",
  "description": "OTTL function reference covering 124 functions (101 standard + 23 EDX). Use when users need OTTL syntax help, EDXRedis guidance, or transformation language validation.",
  "author": { "name": "Edge Delta", "email": "plugins@edgedelta.com" },
  "homepage": "https://docs.edgedelta.com",
  "repository": "https://github.com/edgedelta/claude-code-plugins",
  "license": "Apache-2.0",
  "keywords": ["ottl", "edx", "transformations", "observability"]
}
```

**edgedelta-pipelines**:
```json
{
  "name": "edgedelta-pipelines",
  "version": "2.2.0",
  "description": "Create, validate, and deploy EdgeDelta pipeline v3 configurations. Provides 7 production-tested templates, validation tools, and direct API deployment.",
  "author": { "name": "Edge Delta", "email": "plugins@edgedelta.com" },
  "homepage": "https://docs.edgedelta.com",
  "repository": "https://github.com/edgedelta/claude-code-plugins",
  "license": "Apache-2.0",
  "keywords": ["pipelines", "yaml", "observability", "telemetry", "deployment"]
}
```

---

## Content Strategy

### Source

Skills are copied from `/Users/danielbright/.claude/skills/`:
- `edgedelta-ottl/` → `plugins/edgedelta-ottl/skills/edgedelta-ottl/`
- `edgedelta-pipelines/` → `plugins/edgedelta-pipelines/skills/edgedelta-pipelines/`

### Sanitization Rules

Before copying, check and remediate:
1. **Hardcoded absolute paths** — any `/Users/danielbright/...` references → removed or replaced with `${CLAUDE_PLUGIN_ROOT}/...`
2. **API keys / tokens / org IDs** — strip entirely
3. **Internal-only URLs** — remove anything not in public docs
4. **Excluded files** — `README.md`, `SECURITY.md` from source skill directories (we write new versions at plugin level)

### Relative Path Preservation

All skill-internal references use relative paths (e.g., `MASTER_INDEX.md`, `references/edx/edxredis.md`, `assets/scripts/validate_pipeline.py`). These remain valid after copying because the full directory tree is preserved within each skill directory.

---

## README Design

The README uses the ASCII art header from `asciiart`, followed by:
- Marketplace description
- Plugin listing with descriptions
- Customer installation instructions (marketplace add + plugin install)
- Credential setup for edgedelta-pipelines

---

## Out of Scope (This Phase)

- `edgedelta-reference` plugin (deferred)
- MCP server (deferred to future phase)
- LSP server (deferred)
- Hooks (deferred)
- Remaining plugins from long-term plan (deferred)
- CI/CD workflows (deferred)
