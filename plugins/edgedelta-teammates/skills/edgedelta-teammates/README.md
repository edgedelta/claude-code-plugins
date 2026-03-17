# EdgeDelta Teammates Skill

Guides users through creating, configuring, and managing EdgeDelta AI teammates with effective system prompts, connector permissions, and channel scoping.

## What This Skill Does

- Designs teammate personas with audience, scope, and decision boundaries
- Engineers system prompts (masterPrompt, userPrompt, toolingPrompt) using Handlebars templates
- Configures connector permissions (Allow vs Ask Permission) per tool
- Scopes teammates to specific channels and workflows
- Manages teammate lifecycle (create, update, clone, deactivate, delete)
- Customizes built-in specialized teammates (SRE, Security, DevOps, etc.)
- Sets up periodic tasks for automated recurring checks

## Installation

```bash
cp -r skills/edgedelta-teammates ~/.claude/skills/
```

## File Structure

```
edgedelta-teammates/
├── SKILL.md                          # Main skill definition
├── README.md                         # This file
└── assets/
    └── references/
        ├── prompt-engineering.md      # System prompt patterns and examples
        └── teammate-model.md         # Data model, API, and validation rules
```

## Example Usage

```
"Create a teammate that monitors Kubernetes pod health"
"Help me write a system prompt for a security-focused teammate"
"How do I restrict what my DevOps teammate can access?"
"Customize the SRE specialist to also check PagerDuty"
"Set up a daily health check report"
```

## Related Skills

- `edgedelta-pipelines` - Pipeline creation and deployment
- `edgedelta-dashboards` - Dashboard CRUD operations
- `edgedelta-reference` - Processor syntax reference
- `edgedelta-ottl` - OTTL function reference

## Documentation

- [Custom Teammates](https://docs.edgedelta.com/teammates/)
- [Specialized Teammates](https://docs.edgedelta.com/specialized-teammates/)
- [AI Team Overview](https://docs.edgedelta.com/ai-team-overview/)
- [Connectors](https://docs.edgedelta.com/ai-connectors/)
