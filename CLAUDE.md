# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **Edge Delta Official Plugins** marketplace — a collection of Claude Code plugins that provide AI-powered observability automation for EdgeDelta customers. The repository is published as a Claude Code plugin marketplace at `edgedelta/claude-code-plugins`.

## Repository Structure

```
.claude-plugin/marketplace.json    # Marketplace manifest (lists all plugins)
plugins/
  edgedelta-ottl/                  # OTTL function reference (124 functions)
  edgedelta-pipelines/             # Pipeline creation, validation, deployment
  edgedelta-reference/             # Component reference (sources/processors/destinations)
  edgedelta-teammates/             # AI teammate configuration
  edgedelta-workflows/             # Workflow automation configs
  edgedelta-docker-image/          # Custom agent Docker images + SELinux
```

Each plugin follows the same internal layout:
```
plugins/<name>/
  .claude-plugin/plugin.json       # Plugin manifest (name, version, description, skills list)
  skills/<name>/
    SKILL.md                       # Main skill file (frontmatter + instructions)
    references/                    # Reference docs loaded by the skill
    assets/                        # Templates, scripts, examples (pipelines plugin)
```

## Architecture

**Marketplace manifest** (`.claude-plugin/marketplace.json`): Central registry listing all plugins with version, description, source path, category, and tags. Each plugin entry's `source` field points to its directory under `plugins/`.

**Plugin manifests** (`plugins/<name>/.claude-plugin/plugin.json`): Each plugin declares its own metadata and a `skills` array pointing to skill directories. Currently all plugins expose exactly one skill.

**Skills** are the core component type used — markdown files with YAML frontmatter (`name`, `version`, `description`, `dependencies`) that Claude auto-invokes based on context matching. The `description` field in both `plugin.json` and `SKILL.md` frontmatter is critical for triggering — it must contain the phrases users would say.

## Key Conventions

- **Version sync**: Plugin version in `plugin.json` must match the version in `marketplace.json` and `SKILL.md` frontmatter.
- **Skill naming**: Skill directory name matches plugin name (e.g., `edgedelta-ottl/skills/edgedelta-ottl/`).
- **SKILL.md structure**: Always includes "When to Use" and "Do NOT Use" sections to guide activation boundaries and cross-skill delegation.
- **Cross-skill delegation**: Skills reference each other by name (e.g., OTTL skill says "use `edgedelta-pipelines` for deployment"). Keep these references accurate when adding/renaming plugins.
- **Progressive disclosure**: Skills use master index files and per-topic reference files to avoid loading everything into context at once.

## Adding a New Plugin

1. Create `plugins/<name>/.claude-plugin/plugin.json` following existing plugin manifest format.
2. Create `plugins/<name>/skills/<name>/SKILL.md` with frontmatter and activation instructions.
3. Add the plugin entry to `.claude-plugin/marketplace.json` with matching version and description.
4. Add the plugin to the table in `README.md`.
5. Ensure `.gitignore` exclusions don't hide required files (`.claude-plugin/` directories must be committed despite the `.claude/` exclusion pattern).

## Git Workflow

- **Main branch**: `master` (PRs target this)
- **Working branch**: `main`
- Remote: `edgedelta/claude-code-plugins` on GitHub

## Files Excluded from Git (but present locally)

The `.gitignore` excludes `docs/`, `asciiart`, `edge-delta-plugin-marketplace-plan.md`, and `.claude/` — these are internal planning/development files not shipped to consumers.
