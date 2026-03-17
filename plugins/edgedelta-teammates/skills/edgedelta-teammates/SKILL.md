---
name: edgedelta-teammates
version: 1.0.0
last_updated: 2026-02-09
description: Guides creation and configuration of EdgeDelta AI teammates. Use when users need to build teammates, write system prompts, assign connectors, or scope permissions.
---

# EdgeDelta Teammates Skill

Creates, configures, and manages EdgeDelta AI teammates. Provides guidance on system prompt engineering, connector permissions, channel scoping, and teammate lifecycle management.

## When to Use This Skill

Activate this skill when the user:
- Wants to create a new EdgeDelta teammate
- Asks about teammate configuration or system prompts
- Needs to scope a teammate to specific responsibilities
- Wants to understand teammate permissions or connector access
- Asks about specialized vs custom teammates
- Mentions "teammate", "AI agent", "system prompt", "OnCall AI" in EdgeDelta context
- Wants to troubleshoot teammate behavior or refine prompts

## Core Capabilities

1. **Teammate Design**: Guide persona design with audience, scope, and decision boundaries
2. **System Prompt Engineering**: Build effective masterPrompt, userPrompt, and toolingPrompt templates
3. **Permission Scoping**: Configure connector-level permissions (Allow vs Ask Permission)
4. **Channel Assignment**: Scope teammates to relevant channels and workflows
5. **Lifecycle Management**: Create, update, clone, deactivate, and delete teammates

## Quick Reference Searches

For fast lookups without loading full skill context:

```bash
# Teammate configuration fields
grep -n "Configuration Fields" SKILL.md

# System prompt patterns
grep -n "Prompt Structure" SKILL.md

# Permission model
grep -n "Permission" SKILL.md

# Specialized teammates
grep -n "Specialized" SKILL.md

# Detailed prompt engineering guide
cat assets/references/prompt-engineering.md

# Teammate data model reference
cat assets/references/teammate-model.md
```

## Workflows

### Workflow 1: Create a Custom Teammate

**When**: User wants to build a new teammate from scratch

**Steps**:

1. **Design the persona** - Ask three questions before building:
   - **Audience and scope**: Who will interact with this teammate? What questions should it refuse?
   - **Data sources and tools**: Which connectors expose the telemetry or systems it needs?
   - **Decision boundaries**: Can it execute changes, execute with approval, or only analyze and advise?

2. **Gather required inputs**:
   - **Name**: 1-48 characters, alphanumeric with spaces, underscores, hyphens (`^[a-zA-Z0-9\s_-]+$`)
   - **Description**: One-line summary so colleagues understand the persona
   - **Avatar**: Optional, predefined selection or custom upload
   - **Model**: GPT-4o, GPT-4o-mini, Claude Sonnet, Claude Opus (org defaults apply)
   - **Model Temperature**: 0.1-1.0 (lower = more deterministic, default 0.1)

3. **Build the system prompt** - Construct three prompt layers:
   - **masterPrompt** (required): System-level identity and behavioral instructions
   - **userPrompt** (required): Template wrapping user questions with context
   - **toolingPrompt** (optional): Describes available tools when teammate is called as a tool
   - All prompts use Handlebars syntax for template variables
   - See `assets/references/prompt-engineering.md` for detailed patterns

4. **Assign connectors** - Select minimum required connectors:
   - `edgedelta-mcp` and `edgedelta-documentation` auto-attached by default
   - For each connector, configure per-tool permissions:
     - **Allow**: Execute without approval (read-only operations)
     - **Ask Permission**: Require human approval (write/delete operations)
   - Enable/disable individual tools within each connector

5. **Assign to channels** - Scope availability:
   - Add to specific channels (`#incident-response`, `#platform-ops`, etc.)
   - A DM channel is auto-created for direct 1:1 interaction
   - Tenant-wide access is reserved for broadly applicable personas

6. **Save and verify** - Teammate appears in the Teammates list, is available for DMs and channel interactions

### Workflow 2: Write an Effective System Prompt

**When**: User needs help crafting or refining teammate prompts

**Steps**:

1. Read `assets/references/prompt-engineering.md` for the full guide

2. **masterPrompt structure** - Build in this order:
   ```
   Identity: Who is this teammate and what is their expertise?
   Responsibilities: What specific tasks does it handle?
   Constraints: What should it NOT do?
   Communication style: Tone, format, verbosity
   Tool usage patterns: When/how to use available connectors
   Escalation rules: When to ask for human approval or defer
   ```

3. **userPrompt structure** - Wrap the user's question with context:
   ```
   Available template variables:
   - {{question}}: The user's actual question
   - {{memory_context}}: Retrieved conversation memories
   - {{shared.current_agent.full}}: Full agent object
   - {{shared.current_agent.reference}}: Agent id, name, description, type
   ```

4. **Validate templates** - All prompts are compiled as Handlebars templates at save time. Invalid syntax is rejected.

5. **Test and iterate** - Chat with the teammate, refine based on:
   - Generic answers -> Add concrete tasks, decision criteria, examples
   - Missing data -> Confirm connectors are assigned and credentials valid
   - Verbose responses -> Add formatting guidance ("Limit to 3 bullets")
   - Boundary issues -> Strengthen rules section, require approval for risky actions

### Workflow 3: Scope Teammate Permissions

**When**: User wants to restrict what a teammate can see and do

**Steps**:

1. **Connector-level scoping**: Only assign connectors the teammate needs
   - Each connector exposes specific tools (list resources, query data, create issues)
   - Removing a connector removes all its tools from the teammate

2. **Tool-level scoping**: Within each connector, enable/disable individual tools
   - Navigate: Teammates -> Edit -> Connectors section
   - Toggle specific tools on/off per connector
   - Set execution level per tool:
     - `unsupervised`: Executes automatically
     - `supervised`: Requires human approval before execution

3. **Channel scoping**: Limit where the teammate operates
   - Add only to channels where its expertise is relevant
   - Teammate can only respond in channels where it has membership
   - DM channel allows private 1:1 interaction

4. **Model constraints**:
   - Organization admins can set forbidden models
   - Tool count limited by model capability (with safety threshold)
   - Temperature controls response determinism

### Workflow 4: Customize a Specialized Teammate

**When**: User wants to modify a built-in specialized teammate

**Steps**:

1. Navigate to **Teammates** tab
2. Click kebab menu on the specialist tile -> **Configure**
3. Customizable fields:
   - **System Prompt**: Refine behavior, priorities, communication style
   - **Model Selection**: Change foundation model
   - **Connectors**: Add/remove data source access
4. Click **Save** (changes affect all org users)

**Built-in specialists** (maintained by EdgeDelta):

| Teammate | Domain | Recommended Connectors |
|----------|--------|------------------------|
| OnCall AI | Orchestration & routing | None (orchestrates others) |
| Cloud Engineer | Infrastructure & cost | AWS, Azure, GCP |
| Code Analyzer | Code quality | GitHub |
| DevOps Engineer | Reliability & CI/CD | Kubernetes, GitHub, Jenkins, CircleCI |
| Issue Coordinator | Workflow sync | GitHub, Jira, Linear |
| Security Engineer | Security posture | GitHub, AWS, Jira, CrowdStrike FDR |
| SRE | Incident response | EdgeDelta MCP, PagerDuty, Kubernetes |

### Workflow 5: Manage Teammate Lifecycle

**When**: User needs to update, clone, deactivate, or delete a teammate

**Steps**:

1. **Update**: Edit configuration, prompts, or connectors
   - Prompt changes automatically create a version history entry
   - Optional version notes explain what changed
   - Compare versions side-by-side in the UI

2. **Clone**: Duplicate a teammate for staging or regional variants
   - Creates copy with all configuration, prompts, and connectors
   - Adjust name and access for the new context

3. **Deactivate**: Toggle Online/Offline without deletion
   - Blocked if teammate is referenced by active workflows or monitors
   - Returns list of blocking references if denied
   - Audit log entry created

4. **Delete**: Permanently remove
   - Blocked if referenced by workflows, channels, or monitors
   - Deletes avatar, versions, DM channel, and memberships
   - Cannot be undone

### Workflow 6: Set Up Periodic Tasks

**When**: User wants a teammate to perform automated recurring checks

**Steps**:

1. Navigate to **Periodic Tasks** tab after teammate creation
2. Configure:
   - **What**: Work prompt describing the task
   - **When**: Schedule (15 min, hourly, daily, weekly, monthly, or custom cron)
   - **Where**: Target channel (must have appropriate teammate membership)
3. Ensure the channel has the right teammates assigned for execution
4. Monitor results in channel threads

## Configuration Fields Reference

### Required Fields
| Field | Rules | Example |
|-------|-------|---------|
| `name` | 1-48 chars, `^[a-zA-Z0-9\s_-]+$` | `Kubernetes SRE` |
| `masterPrompt` | Valid Handlebars template | `You are a Kubernetes specialist...` |
| `userPrompt` | Valid Handlebars template | `Context: {{question}}` |
| `status` | `active`, `inactive`, `error` | `active` |

### Optional Fields
| Field | Rules | Default |
|-------|-------|---------|
| `description` | Free text | None |
| `avatar` | Data URL or hosted URL | None |
| `model` | Org-allowed model name | Org default |
| `modelTemperature` | 0.1-1.0 | 0.1 |
| `toolingPrompt` | Valid Handlebars template | Auto-generated |
| `toolConfigurations` | Map of connector tools | `edgedelta-mcp` + `edgedelta-documentation` |
| `capabilities` | Array of capability strings | None |
| `expertise` | Array of expertise strings | None |
| `priority` | Numeric priority | None |

## Teammate Types

| Type | Purpose | Created By |
|------|---------|------------|
| `user-defined` | Custom teammates you create | Users via UI/API |
| `default` | Built-in specialists (SRE, Security, etc.) | EdgeDelta (auto-backfilled) |
| `internal` | System teammates | EdgeDelta platform only |
| `super-agent` | OnCall AI orchestrator | EdgeDelta platform only |

Only `user-defined` and `default` are visible to normal users. `internal` and `super-agent` are restricted.

## Permission Model

### Three Layers of Access Control

1. **Connector assignment**: Which external systems can the teammate reach?
2. **Tool-level permissions**: Within each connector, which operations are enabled?
   - `Allow` (unsupervised): Execute without approval
   - `Ask Permission` (supervised): Human must approve before execution
3. **Channel membership**: Where can the teammate participate?

### Best Practice
- Start with read-only connectors
- Mark write/delete operations as "Ask Permission"
- Grant only connectors required for the teammate's specific role
- Review access quarterly

## Progressive Disclosure

- **Level 1**: Use the AI-powered teammate builder (describe in natural language)
- **Level 2**: Customize prompts and connectors manually
- **Level 3**: Advanced prompt engineering with Handlebars templates
- **Level 4**: Full API integration and workflow orchestration

## Success Criteria

- Teammate activates and responds accurately in assigned channels
- System prompt produces focused, domain-specific responses (not generic)
- Permissions follow least-privilege: only required connectors and tools enabled
- Risky operations require human approval
- Version history tracks all prompt changes

## Example Interactions

**User**: "Create a teammate that monitors our Kubernetes cluster health"
**Agent**: Uses Workflow 1, designs K8s-focused persona, assigns EdgeDelta MCP + Kubernetes connectors, builds system prompt with incident detection focus

**User**: "My teammate gives generic answers about security"
**Agent**: Uses Workflow 2, refines masterPrompt with specific security tasks, decision criteria, tool usage patterns, and escalation rules

**User**: "How do I restrict what my DevOps teammate can do?"
**Agent**: Uses Workflow 3, explains connector/tool/channel scoping, helps set supervised execution for write operations

**User**: "I want the SRE to also check PagerDuty incidents"
**Agent**: Uses Workflow 4, adds PagerDuty connector to SRE specialist, configures tool permissions

**User**: "Set up a daily health check report"
**Agent**: Uses Workflow 6, creates periodic task with daily schedule, assigns to appropriate channel

## Cross-Skill Integration

### Using edgedelta-pipelines Skill
**When**: Teammate needs pipeline data access (streaming connectors feed EdgeDelta MCP)
- Configure streaming connectors to ingest telemetry into pipelines
- Teammates query pipeline data through the EdgeDelta MCP connector

### Using edgedelta-dashboards Skill
**When**: Teammate needs to reference or create dashboards
- Teammates can read dashboard definitions to explain metrics
- Use edgedelta-dashboards skill for dashboard CRUD operations

### Using edgedelta-reference Skill
**When**: Building a teammate that works with pipeline configuration
- Reference processor syntax for pipeline-aware teammates
- Use processor documentation to inform system prompts

## When NOT to Use This Skill

- Pipeline creation or validation -> use `edgedelta-pipelines` skill
- Dashboard operations -> use `edgedelta-dashboards` skill
- Processor syntax reference -> use `edgedelta-reference` skill
- OTTL function reference -> use `edgedelta-ottl` skill
- General EdgeDelta account management without teammate context

## Reference

- [Custom Teammates Documentation](https://docs.edgedelta.com/teammates/)
- [Specialized Teammates](https://docs.edgedelta.com/specialized-teammates/)
- [AI Team Overview](https://docs.edgedelta.com/ai-team-overview/)
- [Connectors Overview](https://docs.edgedelta.com/ai-connectors/)
- [Channels and Direct Messages](https://docs.edgedelta.com/channels/)
- [Periodic Tasks](https://docs.edgedelta.com/periodic-tasks/)

---

You are an expert in EdgeDelta teammate configuration. Guide users through persona design, system prompt engineering, connector scoping, and permission management. Always recommend starting narrow and granting least-privilege access. For prompt refinement, read `assets/references/prompt-engineering.md`. For data model details, read `assets/references/teammate-model.md`.
