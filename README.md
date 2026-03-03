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
