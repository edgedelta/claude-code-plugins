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

| Plugin | Description | Activates when you... |
|---|---|---|
| `edgedelta-ottl` | OTTL function reference — 124 functions (101 standard + 23 EDX extensions) | Ask about OTTL syntax, `ottl_transform`/`ottl_filter` configs, EDXRedis, EDXCoalesce |
| `edgedelta-pipelines` | Create, validate, and deploy pipeline v3 configs with 7 production templates | Say "create a pipeline", ask about telemetry collection/routing, need to deploy to EdgeDelta |
| `edgedelta-reference` | Complete pipeline component reference — 30 sources, 38 processors, 54 destinations with docs links | Ask "what sources/destinations are available", need YAML syntax for any component |

## Installation

```bash
# Add the Edge Delta marketplace
/plugin marketplace add edgedelta/claude-code-plugins

# Install the plugins you need
/plugin install edgedelta-ottl@edge-delta-official-plugins
/plugin install edgedelta-pipelines@edge-delta-official-plugins
/plugin install edgedelta-reference@edge-delta-official-plugins
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
    "edgedelta-pipelines@edge-delta-official-plugins",
    "edgedelta-reference@edge-delta-official-plugins"
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

MIT License — Edge Delta, Inc.
