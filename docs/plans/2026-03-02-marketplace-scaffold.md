# EdgeDelta Plugin Marketplace — Scaffold Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Scaffold the `edgedelta/claude-code-plugins` marketplace repo with two production-ready plugins (`edgedelta-ottl` and `edgedelta-pipelines`), copying and sanitizing the existing skill content from `~/.claude/skills/`.

**Architecture:** Monorepo with a top-level marketplace manifest listing plugins in `plugins/<name>/`. Each plugin follows the Claude Code plugin-structure convention: `.claude-plugin/plugin.json` at plugin root, skills in `skills/<name>/SKILL.md` with supporting files alongside. No custom path overrides — auto-discovery handles everything.

**Tech Stack:** JSON (manifests), Markdown (skills, README), Python (pipeline scripts — already written), Bash (copy/sanitize steps)

**Skills:** @plugin-dev:plugin-structure throughout for structure validation.

---

## Task 1: Rename default branch to `main`

**Files:**
- None (git operation only)

**Step 1: Rename the branch**

```bash
git -C /Users/danielbright/local_dev/edgedelta-marketplace branch -m master main
```

Expected: no output (success)

**Step 2: Verify**

```bash
git -C /Users/danielbright/local_dev/edgedelta-marketplace branch --show-current
```

Expected: `main`

**Step 3: Commit** (nothing to commit yet — just a branch rename, no commit needed)

---

## Task 2: Create root `.gitignore`

**Files:**
- Create: `plugins/edgedelta-pipelines/skills/edgedelta-pipelines/.gitignore` (will be handled in Task 7 — skip here)
- Create: `.gitignore` at repo root

**Step 1: Write the root `.gitignore`**

Create `/Users/danielbright/local_dev/edgedelta-marketplace/.gitignore`:

```
# Python
__pycache__/
*.py[cod]
*.egg-info/
.venv/
dist/
build/

# Environment
.env
.env.local
*.env

# macOS
.DS_Store

# Editor
.idea/
*.swp
```

**Step 2: Stage and commit**

```bash
git -C /Users/danielbright/local_dev/edgedelta-marketplace add .gitignore
git -C /Users/danielbright/local_dev/edgedelta-marketplace commit -m "chore: add root gitignore"
```

---

## Task 3: Create the marketplace manifest

**Files:**
- Create: `.claude-plugin/marketplace.json`

**Step 1: Create the directory**

```bash
mkdir -p /Users/danielbright/local_dev/edgedelta-marketplace/.claude-plugin
```

**Step 2: Write the manifest**

Create `/Users/danielbright/local_dev/edgedelta-marketplace/.claude-plugin/marketplace.json`:

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
      "description": "OTTL function reference covering 124 functions (101 standard + 23 EDX). Use when users need OTTL syntax help, EDXRedis guidance, or transformation language validation.",
      "version": "1.0.1",
      "category": "developer-tools",
      "tags": ["ottl", "edx", "transformations", "observability"],
      "author": { "name": "Edge Delta" },
      "license": "Apache-2.0"
    },
    {
      "name": "edgedelta-pipelines",
      "source": "./plugins/edgedelta-pipelines",
      "description": "Create, validate, and deploy EdgeDelta pipeline v3 configurations with 7 production-tested templates and direct API deployment.",
      "version": "2.2.0",
      "category": "configuration",
      "tags": ["pipelines", "yaml", "observability", "telemetry", "deployment"],
      "author": { "name": "Edge Delta" },
      "license": "Apache-2.0"
    }
  ]
}
```

**Step 3: Validate JSON is well-formed**

```bash
python3 -c "import json; json.load(open('/Users/danielbright/local_dev/edgedelta-marketplace/.claude-plugin/marketplace.json')); print('valid')"
```

Expected: `valid`

**Step 4: Commit**

```bash
git -C /Users/danielbright/local_dev/edgedelta-marketplace add .claude-plugin/marketplace.json
git -C /Users/danielbright/local_dev/edgedelta-marketplace commit -m "feat: add marketplace manifest with two plugins"
```

---

## Task 4: Scaffold the `edgedelta-ottl` plugin

**Files:**
- Create: `plugins/edgedelta-ottl/.claude-plugin/plugin.json`

**Step 1: Create directory structure**

```bash
mkdir -p /Users/danielbright/local_dev/edgedelta-marketplace/plugins/edgedelta-ottl/.claude-plugin
mkdir -p /Users/danielbright/local_dev/edgedelta-marketplace/plugins/edgedelta-ottl/skills/edgedelta-ottl
```

**Step 2: Write plugin manifest**

Create `/Users/danielbright/local_dev/edgedelta-marketplace/plugins/edgedelta-ottl/.claude-plugin/plugin.json`:

```json
{
  "name": "edgedelta-ottl",
  "version": "1.0.1",
  "description": "OTTL function reference covering 124 functions (101 standard + 23 EDX). Use when users need OTTL syntax help, EDXRedis guidance, or transformation language validation. Includes critical naming conventions for EDX functions.",
  "author": {
    "name": "Edge Delta",
    "email": "plugins@edgedelta.com"
  },
  "homepage": "https://docs.edgedelta.com",
  "repository": "https://github.com/edgedelta/claude-code-plugins",
  "license": "Apache-2.0",
  "keywords": ["ottl", "edx", "transformations", "observability", "pipeline"]
}
```

**Step 3: Validate JSON**

```bash
python3 -c "import json; json.load(open('/Users/danielbright/local_dev/edgedelta-marketplace/plugins/edgedelta-ottl/.claude-plugin/plugin.json')); print('valid')"
```

Expected: `valid`

**Step 4: Commit scaffold**

```bash
git -C /Users/danielbright/local_dev/edgedelta-marketplace add plugins/edgedelta-ottl/
git -C /Users/danielbright/local_dev/edgedelta-marketplace commit -m "feat: scaffold edgedelta-ottl plugin"
```

---

## Task 5: Copy and sanitize `edgedelta-ottl` skill files

Source: `/Users/danielbright/.claude/skills/edgedelta-ottl/`
Destination: `/Users/danielbright/local_dev/edgedelta-marketplace/plugins/edgedelta-ottl/skills/edgedelta-ottl/`

**Files to copy:**
- `SKILL.md`
- `MASTER_INDEX.md`
- `validation_rules.md`
- `references/syntax/syntax_guide.md`
- `references/edx/` (23 EDX function docs — entire directory)

**Step 1: Copy files**

```bash
DEST=/Users/danielbright/local_dev/edgedelta-marketplace/plugins/edgedelta-ottl/skills/edgedelta-ottl
SRC=/Users/danielbright/.claude/skills/edgedelta-ottl

cp "$SRC/SKILL.md" "$DEST/"
cp "$SRC/MASTER_INDEX.md" "$DEST/"
cp "$SRC/validation_rules.md" "$DEST/"
mkdir -p "$DEST/references/syntax" "$DEST/references/edx"
cp "$SRC/references/syntax/syntax_guide.md" "$DEST/references/syntax/"
cp -r "$SRC/references/edx/." "$DEST/references/edx/"
```

**Step 2: Sanitization check — hardcoded user paths**

```bash
grep -r "/Users/danielbright" /Users/danielbright/local_dev/edgedelta-marketplace/plugins/edgedelta-ottl/
```

Expected: no output. If any matches found, open the file and remove the hardcoded path or replace with `${CLAUDE_PLUGIN_ROOT}`.

**Step 3: Sanitization check — API keys / tokens**

```bash
grep -rEi "(api[_-]?key|api[_-]?token|secret|password)\s*[:=]\s*['\"]?[A-Za-z0-9+/]{20,}" \
  /Users/danielbright/local_dev/edgedelta-marketplace/plugins/edgedelta-ottl/
```

Expected: no output.

**Step 4: Sanitization check — UUID org IDs**

```bash
grep -rE "[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}" \
  /Users/danielbright/local_dev/edgedelta-marketplace/plugins/edgedelta-ottl/
```

Expected: no output. If matches appear, review each — only remove if it's an actual org/account identifier (not a placeholder UUID in docs).

**Step 5: Verify file count**

```bash
find /Users/danielbright/local_dev/edgedelta-marketplace/plugins/edgedelta-ottl/skills/ -type f | wc -l
```

Expected: approximately 26 files (SKILL.md + MASTER_INDEX.md + validation_rules.md + syntax_guide.md + 23 EDX reference files = 27 — close to this number is fine).

**Step 6: Commit**

```bash
git -C /Users/danielbright/local_dev/edgedelta-marketplace add plugins/edgedelta-ottl/skills/
git -C /Users/danielbright/local_dev/edgedelta-marketplace commit -m "feat: add edgedelta-ottl skill content"
```

---

## Task 6: Scaffold the `edgedelta-pipelines` plugin

**Files:**
- Create: `plugins/edgedelta-pipelines/.claude-plugin/plugin.json`

**Step 1: Create directory structure**

```bash
mkdir -p /Users/danielbright/local_dev/edgedelta-marketplace/plugins/edgedelta-pipelines/.claude-plugin
mkdir -p /Users/danielbright/local_dev/edgedelta-marketplace/plugins/edgedelta-pipelines/skills/edgedelta-pipelines
```

**Step 2: Write plugin manifest**

Create `/Users/danielbright/local_dev/edgedelta-marketplace/plugins/edgedelta-pipelines/.claude-plugin/plugin.json`:

```json
{
  "name": "edgedelta-pipelines",
  "version": "2.2.0",
  "description": "Create, validate, and deploy EdgeDelta pipeline v3 configurations. Provides 7 production-tested templates, validation tools, environment inspection, and direct API deployment.",
  "author": {
    "name": "Edge Delta",
    "email": "plugins@edgedelta.com"
  },
  "homepage": "https://docs.edgedelta.com",
  "repository": "https://github.com/edgedelta/claude-code-plugins",
  "license": "Apache-2.0",
  "keywords": ["pipelines", "yaml", "observability", "telemetry", "deployment", "ottl"]
}
```

**Step 3: Validate JSON**

```bash
python3 -c "import json; json.load(open('/Users/danielbright/local_dev/edgedelta-marketplace/plugins/edgedelta-pipelines/.claude-plugin/plugin.json')); print('valid')"
```

Expected: `valid`

**Step 4: Commit scaffold**

```bash
git -C /Users/danielbright/local_dev/edgedelta-marketplace add plugins/edgedelta-pipelines/
git -C /Users/danielbright/local_dev/edgedelta-marketplace commit -m "feat: scaffold edgedelta-pipelines plugin"
```

---

## Task 7: Copy and sanitize `edgedelta-pipelines` skill files

Source: `/Users/danielbright/.claude/skills/edgedelta-pipelines/`
Destination: `/Users/danielbright/local_dev/edgedelta-marketplace/plugins/edgedelta-pipelines/skills/edgedelta-pipelines/`

**Files to copy:** SKILL.md, CHANGELOG.md, full `assets/` tree.
**Files to exclude:** README.md, SECURITY.md (replaced by repo-level README).

**Step 1: Copy files**

```bash
DEST=/Users/danielbright/local_dev/edgedelta-marketplace/plugins/edgedelta-pipelines/skills/edgedelta-pipelines
SRC=/Users/danielbright/.claude/skills/edgedelta-pipelines

cp "$SRC/SKILL.md" "$DEST/"
cp "$SRC/CHANGELOG.md" "$DEST/"
cp -r "$SRC/assets/." "$DEST/assets/"
```

**Step 2: Sanitization check — hardcoded user paths**

```bash
grep -r "/Users/danielbright" /Users/danielbright/local_dev/edgedelta-marketplace/plugins/edgedelta-pipelines/
```

Expected: no output. If any appear (e.g., in Python scripts), replace with a comment or parameterize.

**Step 3: Sanitization check — API keys / tokens**

```bash
grep -rEi "(api[_-]?key|api[_-]?token|secret)\s*[:=]\s*['\"][^'\"]{20,}['\"]" \
  /Users/danielbright/local_dev/edgedelta-marketplace/plugins/edgedelta-pipelines/
```

Expected: no output. Note: the Python scripts accept credentials as CLI args or from `.env` — that's correct behavior, not a hardcoded secret.

**Step 4: Sanitization check — UUID org IDs**

```bash
grep -rE "[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}" \
  /Users/danielbright/local_dev/edgedelta-marketplace/plugins/edgedelta-pipelines/
```

Expected: no output for real org IDs. Template YAML files may contain placeholder UUIDs — inspect each match and confirm it's a documented placeholder before leaving it in.

**Step 5: Sanitization check — internal-only URLs**

```bash
grep -rE "https://[a-z0-9.-]+\.(internal|corp|local|dev)[/\"]" \
  /Users/danielbright/local_dev/edgedelta-marketplace/plugins/edgedelta-pipelines/
```

Expected: no output.

**Step 6: Verify file count**

```bash
find /Users/danielbright/local_dev/edgedelta-marketplace/plugins/edgedelta-pipelines/skills/ -type f | wc -l
```

Expected: approximately 17 files (SKILL.md + CHANGELOG.md + 4 scripts + 7 templates + 3 references + examples summary = ~17).

**Step 7: Commit**

```bash
git -C /Users/danielbright/local_dev/edgedelta-marketplace add plugins/edgedelta-pipelines/skills/
git -C /Users/danielbright/local_dev/edgedelta-marketplace commit -m "feat: add edgedelta-pipelines skill content"
```

---

## Task 8: Write the README

**Files:**
- Create: `README.md`

**Step 1: Write README**

Create `/Users/danielbright/local_dev/edgedelta-marketplace/README.md` with this content:

````markdown
```
▄▖ ▌      ▄   ▜ ▗
▙▖▛▌▛▌█▌  ▌▌█▌▐ ▜▘▀▌
▙▖▙▌▙▌▙▖  ▙▘▙▖▐▖▐▖█▌
    ▄▌
▄▖▜      ▌    ▄▖▜     ▘
▌ ▐ ▀▌▌▌▛▌█▌  ▙▌▐ ▌▌▛▌▌▛▌▛▘
▙▖▐▖█▌▙▌▙▌▙▖  ▌ ▐▖▙▌▙▌▌▌▌▄▌
                    ▄▌
```

# Edge Delta Official Plugins

Official Claude Code plugins from Edge Delta — AI-powered observability automation inside your development environment.

## Plugins

### `edgedelta-ottl`

OTTL (OpenTelemetry Transformation Language) function reference for Edge Delta pipelines.

**Auto-activates when you:**
- Ask about OTTL syntax, operators, or language rules
- Need documentation for any of 124 OTTL functions (standard + EDX extensions)
- Work on `ottl_transform` or `ottl_filter` processor configurations
- Encounter OTTL syntax errors and need strict validation
- Ask about EDXRedis, EDXCoalesce, or any other EDX extension

### `edgedelta-pipelines`

Create, validate, and deploy Edge Delta pipeline v3 configurations.

**Auto-activates when you:**
- Want to create or deploy an Edge Delta pipeline
- Need to validate a pipeline YAML configuration
- Ask about telemetry collection, routing, or processing
- Say "what can I monitor" in your environment

**Includes:**
- 7 production-tested pipeline templates
- Validation and deployment scripts
- Environment inspection tools

## Installation

```bash
# Add the Edge Delta marketplace
/plugin marketplace add edgedelta/claude-code-plugins

# Install the plugins you need
/plugin install edgedelta-ottl@edge-delta-official-plugins
/plugin install edgedelta-pipelines@edge-delta-official-plugins
```

### Team Rollout

Add to `.claude/settings.json` in your shared repo:

```json
{
  "extraKnownMarketplaces": {
    "edge-delta-official-plugins": {
      "source": {
        "source": "github",
        "repo": "edgedelta/claude-code-plugins"
      }
    }
  },
  "enabledPlugins": [
    "edgedelta-ottl@edge-delta-official-plugins",
    "edgedelta-pipelines@edge-delta-official-plugins"
  ]
}
```

## Credentials (edgedelta-pipelines)

Set in your environment or `~/.edgedelta.env`:

```bash
ED_ORG_ID=your-org-id
ED_ORG_API_TOKEN=your-api-token
```

Get these from [app.edgedelta.com](https://app.edgedelta.com).

## License

Apache-2.0 — Edge Delta, Inc.
````

**Step 2: Commit**

```bash
git -C /Users/danielbright/local_dev/edgedelta-marketplace add README.md
git -C /Users/danielbright/local_dev/edgedelta-marketplace commit -m "docs: add README with ASCII art and install instructions"
```

---

## Task 9: Final verification

**Step 1: Confirm complete repo structure**

```bash
find /Users/danielbright/local_dev/edgedelta-marketplace \
  -not -path '*/.git/*' \
  -not -path '*/__pycache__/*' \
  | sort
```

Verify the output includes:
- `.claude-plugin/marketplace.json`
- `plugins/edgedelta-ottl/.claude-plugin/plugin.json`
- `plugins/edgedelta-ottl/skills/edgedelta-ottl/SKILL.md`
- `plugins/edgedelta-pipelines/.claude-plugin/plugin.json`
- `plugins/edgedelta-pipelines/skills/edgedelta-pipelines/SKILL.md`
- `README.md`
- `docs/plans/2026-03-02-marketplace-design.md`

**Step 2: Validate all JSON manifests**

```bash
for f in $(find /Users/danielbright/local_dev/edgedelta-marketplace -name "*.json" -not -path '*/.git/*'); do
  python3 -c "import json; json.load(open('$f')); print('OK: $f')"
done
```

Expected: all files print `OK: ...`

**Step 3: Run final sanitization sweep across entire repo**

```bash
grep -r "/Users/danielbright" /Users/danielbright/local_dev/edgedelta-marketplace \
  --exclude-dir=.git --exclude-dir=__pycache__
```

Expected: no output.

**Step 4: Check git log**

```bash
git -C /Users/danielbright/local_dev/edgedelta-marketplace log --oneline
```

Expected output (most recent first):
```
<hash> docs: add README with ASCII art and install instructions
<hash> feat: add edgedelta-pipelines skill content
<hash> feat: scaffold edgedelta-pipelines plugin
<hash> feat: add edgedelta-ottl skill content
<hash> feat: scaffold edgedelta-ottl plugin
<hash> feat: add marketplace manifest with two plugins
<hash> chore: add root gitignore
<hash> Add marketplace design document
```

**Step 5: Set up GitHub remote (if repo exists)**

```bash
git -C /Users/danielbright/local_dev/edgedelta-marketplace \
  remote add origin https://github.com/edgedelta/claude-code-plugins.git
```

If the GitHub repo doesn't exist yet, skip this step and create it via GitHub UI or `gh repo create edgedelta/claude-code-plugins --public`.

---

## Done

The marketplace repo is ready. Next steps (out of scope for this plan):
- Push to GitHub: `git push -u origin main`
- Add remaining plugins from the long-term roadmap
- Set up CI to lint manifests on PRs
