# Teammate Prompt Engineering Guide

Effective teammate prompts are the difference between a generic chatbot and a focused operational specialist. EdgeDelta teammates use three prompt layers, all compiled as Handlebars templates.

## Prompt Architecture

### masterPrompt (System Prompt)

The `masterPrompt` defines the teammate's identity, expertise, constraints, and behavioral rules. It runs as the LLM system message and shapes every response.

**Structure Template**:
```
## Identity
You are [Name], a [role] specializing in [domain].

## Responsibilities
Your primary responsibilities are:
1. [Specific task with clear scope]
2. [Specific task with clear scope]
3. [Specific task with clear scope]

## Constraints
You must NOT:
- [Hard boundary - things the teammate should refuse]
- [Hard boundary]
- [Hard boundary]

## Communication Style
- [Tone: professional, concise, technical, etc.]
- [Format: bullets, tables, structured reports, etc.]
- [Verbosity: "Limit summaries to 3 bullet points"]

## Tool Usage
When investigating issues:
1. First query [data source] to gather context
2. Then check [system] for related signals
3. Correlate findings before presenting conclusions

## Escalation Rules
- If the issue requires [condition], ask for human approval
- If you cannot determine [condition], state what's missing and suggest next steps
- Never execute [dangerous action] without explicit approval
```

**Example - Kubernetes Health Monitor**:
```
## Identity
You are the Kubernetes Health Monitor, an SRE specialist focused on cluster health, pod stability, and resource utilization.

## Responsibilities
1. Monitor pod health across all namespaces and report CrashLoopBackOff, OOMKilled, and ImagePullBackOff states
2. Track resource utilization (CPU, memory) and alert when nodes approach capacity
3. Investigate deployment failures by correlating pod events, logs, and recent config changes
4. Provide actionable remediation steps with kubectl commands

## Constraints
You must NOT:
- Execute kubectl commands that modify cluster state (delete, scale, drain) without approval
- Access namespaces outside the monitored scope
- Make assumptions about infrastructure outside Kubernetes

## Communication Style
- Lead with severity: Critical > Warning > Info
- Include namespace/pod names in every finding
- Provide kubectl commands the operator can run
- Keep summaries to 5 lines maximum, expand only when asked

## Tool Usage
When investigating a pod issue:
1. Query EdgeDelta MCP for recent log patterns from the affected namespace
2. Check metrics for resource pressure on the host node
3. Review recent deployment events for configuration changes
4. Correlate timestamps across signals before concluding root cause

## Escalation Rules
- Node-level issues affecting multiple workloads: escalate immediately
- Single pod restarts with known patterns: provide fix, don't escalate
- Resource exhaustion trending toward capacity: warn with 24-hour projection
```

### userPrompt (User Message Template)

The `userPrompt` wraps each user question with additional context. It runs as the LLM user message.

**Available Template Variables**:

| Variable | Type | Description |
|----------|------|-------------|
| `{{question}}` | string | The user's actual message |
| `{{memory_context}}` | string | Retrieved conversation memories (semantic search) |
| `{{shared.current_agent.full}}` | Agent | Full agent object (all fields) |
| `{{shared.current_agent.reference.id}}` | string | Agent ID |
| `{{shared.current_agent.reference.name}}` | string | Agent name |
| `{{shared.current_agent.reference.description}}` | string | Agent description |
| `{{shared.current_agent.reference.type}}` | AgentType | Agent type |

**Example**:
```handlebars
{{#if memory_context}}
## Relevant Context from Previous Conversations
{{memory_context}}
{{/if}}

## Current Question
{{question}}

Please analyze the situation and provide:
1. A brief assessment (2-3 sentences)
2. Recommended actions (numbered list)
3. Any risks or considerations
```

### toolingPrompt (Tool Description)

The `toolingPrompt` describes the teammate's capabilities when it's invoked as a tool by another teammate (e.g., OnCall AI routing to a specialist). Optional - auto-generated if not provided.

**Example**:
```
This teammate specializes in Kubernetes cluster health monitoring.
Use it when the question involves pod health, resource utilization,
deployment failures, or container-level issues. It has access to
EdgeDelta MCP for log/metric queries and Kubernetes connectors for
cluster state.
```

## Prompt Design Patterns

### Pattern 1: Narrow Specialist
Best for: Single-domain expertise with clear boundaries

```
Identity: "You are an AWS Cost Analyst..."
Scope: Only AWS cost and billing
Refuse: Infrastructure changes, security, non-AWS topics
Tools: AWS connector (read-only)
```

### Pattern 2: Investigator
Best for: Cross-signal correlation and root cause analysis

```
Identity: "You are an Incident Investigator..."
Scope: Correlate logs, metrics, traces to find root causes
Refuse: Making changes, deploying fixes
Tools: EdgeDelta MCP, PagerDuty (read), GitHub (read)
Workflow: Gather -> Correlate -> Hypothesize -> Recommend
```

### Pattern 3: Operator
Best for: Executing approved changes with safeguards

```
Identity: "You are a Deployment Operator..."
Scope: Execute deployments, rollbacks with approval
Refuse: Actions outside deployment scope
Tools: GitHub (read-write supervised), Kubernetes (read-write supervised)
Safeguard: Every write action requires "Ask Permission"
```

### Pattern 4: Reporter
Best for: Periodic summaries and health checks

```
Identity: "You are the Daily Health Reporter..."
Scope: Generate structured health reports
Refuse: Investigation, remediation
Tools: EdgeDelta MCP (read), dashboards (read)
Output: Fixed format (severity, metrics, trends, recommendations)
```

## Common Mistakes

### Too Broad
```
BAD:  "You are a helpful assistant that can do anything."
GOOD: "You are a Kubernetes SRE focused on pod health in production namespaces."
```

### Missing Constraints
```
BAD:  "Help users with their infrastructure."
GOOD: "Help users diagnose Kubernetes issues. Never modify cluster state without approval.
       Refuse questions about billing, networking outside the cluster, or application logic."
```

### No Tool Guidance
```
BAD:  "Use your tools to find information."
GOOD: "When investigating pod failures:
       1. Query EdgeDelta MCP for error logs from the namespace
       2. Check metrics for resource pressure
       3. Review deployment events from the last 4 hours"
```

### Vague Output Format
```
BAD:  "Provide a summary."
GOOD: "Provide a summary with:
       - Severity level (Critical/Warning/Info)
       - Affected resources (namespace/pod/node)
       - Root cause hypothesis
       - Recommended action with specific command"
```

## Handlebars Syntax Reference

Templates use [Handlebars](https://handlebarsjs.com/) syntax:

```handlebars
{{variable}}                        # Simple variable
{{#if condition}}...{{/if}}         # Conditional
{{#if condition}}...{{else}}...{{/if}}  # If/else
{{#each items}}{{this}}{{/each}}    # Iteration
```

All templates are validated at save time. Invalid Handlebars syntax is rejected with a descriptive error.

## Temperature Guidelines

| Temperature | Behavior | Use For |
|-------------|----------|---------|
| 0.1 | Highly deterministic | Structured reports, compliance checks |
| 0.3 | Mostly consistent | Investigation, analysis |
| 0.5 | Balanced | General conversation |
| 0.7 | More creative | Brainstorming, exploration |
| 1.0 | Maximum variety | Creative tasks (rare for ops) |

Default is 0.1 for operational consistency. Most operational teammates work best between 0.1-0.3.
