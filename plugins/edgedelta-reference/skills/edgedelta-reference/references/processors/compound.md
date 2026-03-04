# compound Processor

**Type**: `compound`
**Category**: Nested Structures / Advanced
**Sequence Compatible**: ✗ No (top-level node only)
**Source**: `internalv3/processors/compound/processor.go`
**Config Struct**: `configv3.Node` (with `nodes` and `links` fields)

## Quick Copy

```yaml
- name: my_compound
  type: compound
  nodes:
    - name: input
      type: compound_input
    - name: my_processor
      type: ottl_transform
      statements:
        - 'set(attributes["processed"], true)'
    - name: output
      type: compound_output
  links:
    - from: input
      to: my_processor
    - from: my_processor
      to: output
```

## Overview

The `compound` processor creates custom graph-based processing pipelines by encapsulating multiple processor nodes and their connections into a single reusable unit. Unlike simple sequences that process data linearly, compound nodes allow for complex routing with conditional branching, fan-out patterns, and multiple exit paths.

**Important**: Compound nodes are an advanced feature primarily used for creating reusable processor packs. For most use cases, simple sequences are recommended. Use compound nodes only when you need complex multi-path routing or plan to reuse the same processing logic across multiple pipelines.

## Use Cases

- **Reusable Processor Packs**: Package common processing logic (e.g., PII masking + enrichment) for use across multiple pipelines
- **Multi-Path Routing**: Send data to different outputs based on processing results (success/failure paths)
- **Conditional Branching**: Route logs through different processors based on conditions or processor outcomes
- **Team-Specific Processing**: Create isolated processing graphs for different teams or use cases
- **Complex Data Flows**: Build sophisticated processing patterns that cannot be expressed with linear sequences
- **Error Handling Isolation**: Separate success and failure paths for robust error handling
- **Modular Pipeline Design**: Break large pipelines into manageable, testable compound units
- **Fan-Out Processing**: Split data streams to multiple parallel processing paths

## Parameters

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | string | Unique name for the compound node |
| `type` | string | Must be `compound` |
| `nodes` | []Node | Array of processor nodes within the compound (must include exactly 1 `compound_input` and at least 1 `compound_output`) |
| `links` | []Link | Array of connections between nodes within the compound |

### Node Fields (within nodes array)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Name of the internal node (unique within the compound) |
| `type` | string | Yes | Node type (`compound_input`, `compound_output`, or any processor type) |
| *processor-specific* | varies | Depends | Additional fields based on processor type |

### Link Fields (within links array)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `from` | string | Yes | Name of the source node (within the compound) |
| `to` | string | Yes | Name of the destination node (within the compound) |
| `path` | string | No | Optional path name (e.g., `failure` for error handling) |

### Optional Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `description` | string | "" | Human-readable description of the compound node |
| `metadata` | string | "" | Arbitrary metadata for tracking/debugging |
| `disabled` | bool | false | Disable this compound node without removing from config |

## Graph Structure

### Nodes

A compound node must contain:
- **Exactly 1** `compound_input` node - Entry point for data into the compound
- **At least 1** `compound_output` node - Exit point(s) for data from the compound
- **0 or more** processor nodes - Any processor type except input/output nodes

### Links

Links define the data flow within the compound:
- **Internal Links**: Connect nodes within the same compound (e.g., `input` → `processor` → `output`)
- **External Links**: Connect compound nodes to other nodes in the pipeline (e.g., `external_input` → `my_compound->input`)
- **Named Paths**: Use the `path` field to route data through specific exit points (e.g., success vs failure)

### Node Naming Convention

When referencing compound nodes from outside, use the delimiter `->`:
- Internal node: `processor` (within compound definition)
- External reference: `my_compound->processor` (from pipeline links)

## Examples

### Example 1: Basic Compound with 2 Nodes

```yaml
- name: simple_transform
  type: compound
  description: "Basic transformation compound"
  nodes:
    - name: input
      type: compound_input
    - name: add_tag
      type: ottl_transform
      statements:
        - 'set(attributes["processed"], true)'
    - name: output
      type: compound_output
  links:
    - from: input
      to: add_tag
    - from: add_tag
      to: output

# External links
links:
  - from: my_input
    to: simple_transform->input
  - from: simple_transform->output
    to: my_output
```

**What it does**: Creates a simple compound that adds a `processed` attribute to all logs.

### Example 2: Multi-Path Routing (Success/Failure)

```yaml
- name: json_parser
  type: compound
  description: "Parse JSON with error handling"
  nodes:
    - name: input
      type: compound_input
    - name: parser
      type: parse_json_attributes
      source_field: "body"
      target_field: "parsed"
    - name: success
      type: compound_output
    - name: failure
      type: compound_output
  links:
    - from: input
      to: parser
    - from: parser
      to: success
    - from: parser
      to: failure
      path: failure

# External links - route successes and failures differently
links:
  - from: my_input
    to: json_parser->input
  - from: json_parser->success
    to: processed_logs_destination
  - from: json_parser->failure
    to: error_logs_destination
```

**What it does**: Parses JSON logs and routes successful parses to one destination and failures to another.

### Example 3: Conditional Branching

```yaml
- name: log_routing
  type: compound
  description: "Route logs based on severity"
  nodes:
    - name: input
      type: compound_input
    - name: detect_level
      type: log_level_detector
    - name: high_severity
      type: compound_output
    - name: low_severity
      type: compound_output
  links:
    - from: input
      to: detect_level
    - from: detect_level
      to: high_severity
    - from: detect_level
      to: low_severity
      path: low_priority

# External links
links:
  - from: app_logs
    to: log_routing->input
  - from: log_routing->high_severity
    to: alert_system
  - from: log_routing->low_severity
    to: archive_storage
```

**What it does**: Routes logs to different destinations based on detected severity level.

### Example 4: Fan-Out Processing

```yaml
- name: multi_destination
  type: compound
  description: "Send logs to multiple processors"
  nodes:
    - name: input
      type: compound_input
    - name: add_metadata
      type: ottl_transform
      statements:
        - 'set(attributes["timestamp"], Now())'
    - name: metrics_output
      type: compound_output
    - name: archive_output
      type: compound_output
  links:
    - from: input
      to: add_metadata
    - from: add_metadata
      to: metrics_output
      path: metrics
    - from: add_metadata
      to: archive_output
      path: archive

# External links - fan out to multiple destinations
links:
  - from: app_input
    to: multi_destination->input
  - from: multi_destination->metrics_output
    to: extract_metric_node
  - from: multi_destination->archive_output
    to: s3_output
```

**What it does**: Adds metadata then sends the same data to both metrics extraction and archival storage.

### Example 5: Complex Custom Logic

```yaml
- name: pii_and_enrichment
  type: compound
  description: "PII masking followed by enrichment"
  nodes:
    - name: input
      type: compound_input
    - name: mask_emails
      type: generic_mask
      capture_group_masks:
        - capture_group: "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}"
          enabled: true
          mask: "***EMAIL***"
          name: "email"
    - name: mask_cards
      type: generic_mask
      capture_group_masks:
        - capture_group: "\\b\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}\\b"
          enabled: true
          mask: "***CC***"
          name: "credit_card"
    - name: add_environment
      type: ottl_transform
      statements:
        - 'set(attributes["environment"], "production")'
    - name: output
      type: compound_output
  links:
    - from: input
      to: mask_emails
    - from: mask_emails
      to: mask_cards
    - from: mask_cards
      to: add_environment
    - from: add_environment
      to: output

# External links
links:
  - from: application_logs
    to: pii_and_enrichment->input
  - from: pii_and_enrichment->output
    to: elasticsearch_output
```

**What it does**: Chains multiple processors (email masking, credit card masking, enrichment) in a reusable unit.

### Example 6: Nested Compound Nodes

```yaml
- name: outer_compound
  type: compound
  description: "Compound containing another compound"
  nodes:
    - name: input
      type: compound_input
    - name: inner_compound
      type: compound
      nodes:
        - name: inner_input
          type: compound_input
        - name: transform
          type: ottl_transform
          statements:
            - 'set(attributes["nested"], true)'
        - name: inner_output
          type: compound_output
      links:
        - from: inner_input
          to: transform
        - from: transform
          to: inner_output
    - name: output
      type: compound_output
  links:
    - from: input
      to: inner_compound->inner_input
    - from: inner_compound->inner_output
      to: output

# External links
links:
  - from: my_input
    to: outer_compound->input
  - from: outer_compound->output
    to: my_output
```

**What it does**: Demonstrates compound nodes can be nested, though this is rarely needed in practice.

### Example 7: Error Handling with Multiple Processors

```yaml
- name: robust_parser
  type: compound
  description: "Multi-stage parsing with error handling"
  nodes:
    - name: input
      type: compound_input
    - name: parse_json
      type: parse_json_attributes
      source_field: "body"
    - name: validate
      type: ottl_filter
      condition: 'attributes["parsed.status"] != nil'
    - name: enrich
      type: ottl_transform
      statements:
        - 'set(attributes["validated"], true)'
    - name: success
      type: compound_output
    - name: parse_failure
      type: compound_output
    - name: validation_failure
      type: compound_output
  links:
    - from: input
      to: parse_json
    - from: parse_json
      to: validate
    - from: parse_json
      to: parse_failure
      path: failure
    - from: validate
      to: enrich
    - from: validate
      to: validation_failure
      path: filtered
    - from: enrich
      to: success

# External links - three exit paths
links:
  - from: app_logs
    to: robust_parser->input
  - from: robust_parser->success
    to: processed_destination
  - from: robust_parser->parse_failure
    to: parse_error_logs
  - from: robust_parser->validation_failure
    to: validation_error_logs
```

**What it does**: Multi-stage processing with separate error paths for different failure types.

### Example 8: Team-Specific Processing Pack

```yaml
- name: team_a_pack
  type: compound
  description: "Standard processing for Team A logs"
  nodes:
    - name: input
      type: compound_input
    - name: add_team_tag
      type: ottl_transform
      statements:
        - 'set(attributes["team"], "team-a")'
        - 'set(attributes["cost_center"], "engineering")'
    - name: mask_pii
      type: generic_mask
      capture_group_masks:
        - capture_group: "\\b\\d{3}-\\d{2}-\\d{4}\\b"
          enabled: true
          mask: "***SSN***"
          name: "ssn"
    - name: extract_errors
      type: extract_metric
      extract_metric_rules:
        - name: "team_a_errors_total"
          unit: "1"
          conditions:
            - 'IsMatch(body, "(?i)ERROR")'
          sum:
            aggregation_temporality: delta
            is_monotonic: true
            value: 1
      interval: 1m
    - name: output
      type: compound_output
  links:
    - from: input
      to: add_team_tag
    - from: add_team_tag
      to: mask_pii
    - from: mask_pii
      to: extract_errors
    - from: extract_errors
      to: output

# Multiple teams can use the same pattern
- name: team_b_pack
  type: compound
  description: "Standard processing for Team B logs"
  nodes:
    - name: input
      type: compound_input
    - name: add_team_tag
      type: ottl_transform
      statements:
        - 'set(attributes["team"], "team-b")'
        - 'set(attributes["cost_center"], "operations")'
    - name: output
      type: compound_output
  links:
    - from: input
      to: add_team_tag
    - from: add_team_tag
      to: output

# External links - each team's logs go through their pack
links:
  - from: team_a_logs_input
    to: team_a_pack->input
  - from: team_a_pack->output
    to: shared_destination
  - from: team_b_logs_input
    to: team_b_pack->input
  - from: team_b_pack->output
    to: shared_destination
```

**What it does**: Creates reusable processing packs for different teams with standardized tagging, PII masking, and metrics extraction.

## Validation Rules

1. **Required Fields**: Must specify `name`, `type: compound`, `nodes`, `links`
2. **Exactly One Input**: Compound must have exactly one `compound_input` node
3. **At Least One Output**: Compound must have at least one `compound_output` node
4. **Processor Nodes Only**: Internal nodes can only be processors (not input/output nodes except `compound_input`/`compound_output`)
5. **Unique Node Names**: All node names within a compound must be unique
6. **Valid Links**: All links must reference nodes that exist within the compound
7. **No Self-Links**: Links cannot connect a node to itself (prevents loops)
8. **Proper Node References**: External links to compound nodes must use `compound_name->node_name` format
9. **Input Entry Only**: External traffic can only enter via `compound_input` nodes
10. **Output Exit Only**: External traffic can only exit via `compound_output` nodes
11. **No Skip-Level Links**: Links cannot skip compound nesting levels (must flow through each level)
12. **Connected Graph**: All internal nodes should be reachable from input and able to reach an output
13. **No Empty Names**: Node names cannot be empty after the delimiter (e.g., `CN->` is invalid)

## Common Pitfalls

### 1. Invalid Node References in Links

**Problem**: Referencing nodes that don't exist within the compound.

**Wrong**:
```yaml
nodes:
  - name: input
    type: compound_input
  - name: output
    type: compound_output
links:
  - from: input
    to: processor  # processor doesn't exist
  - from: processor
    to: output
```

**Correct**:
```yaml
nodes:
  - name: input
    type: compound_input
  - name: processor
    type: ottl_transform
    statements:
      - 'set(attributes["processed"], true)'
  - name: output
    type: compound_output
links:
  - from: input
    to: processor
  - from: processor
    to: output
```

### 2. Circular Dependencies

**Problem**: Creating loops in the processing graph.

**Wrong**:
```yaml
links:
  - from: input
    to: processor_a
  - from: processor_a
    to: processor_b
  - from: processor_b
    to: processor_a  # Loop!
```

**Correct**: Ensure data flows in one direction from input to output without cycles.

### 3. Missing Output Connections

**Problem**: Processors that don't connect to any output node.

**Wrong**:
```yaml
nodes:
  - name: input
    type: compound_input
  - name: processor
    type: ottl_transform
    statements:
      - 'set(attributes["x"], 1)'
  - name: output
    type: compound_output
links:
  - from: input
    to: processor
  # Missing: processor -> output
```

**Correct**:
```yaml
links:
  - from: input
    to: processor
  - from: processor
    to: output
```

### 4. Multiple Input Nodes

**Problem**: Defining more than one `compound_input` node.

**Wrong**:
```yaml
nodes:
  - name: input1
    type: compound_input
  - name: input2
    type: compound_input
  - name: output
    type: compound_output
```

**Correct**: Only one input node allowed.
```yaml
nodes:
  - name: input
    type: compound_input
  - name: output
    type: compound_output
```

### 5. No Output Nodes

**Problem**: Forgetting to include at least one `compound_output` node.

**Wrong**:
```yaml
nodes:
  - name: input
    type: compound_input
  - name: processor
    type: ottl_transform
    statements:
      - 'set(attributes["x"], 1)'
```

**Correct**:
```yaml
nodes:
  - name: input
    type: compound_input
  - name: processor
    type: ottl_transform
    statements:
      - 'set(attributes["x"], 1)'
  - name: output
    type: compound_output
```

### 6. Node Naming Conflicts

**Problem**: Using the delimiter `->` in internal node names or empty names after delimiter.

**Wrong**:
```yaml
nodes:
  - name: my->processor  # Don't use -> in internal names
    type: ottl_transform
  - name: CN->  # Empty name after delimiter
    type: compound_input
```

**Correct**:
```yaml
nodes:
  - name: my_processor
    type: ottl_transform
  - name: input
    type: compound_input
```

### 7. Incorrect External Link References

**Problem**: Not using the `->` delimiter when referencing compound nodes externally.

**Wrong**:
```yaml
links:
  - from: my_input
    to: my_compound.input  # Wrong delimiter
```

**Correct**:
```yaml
links:
  - from: my_input
    to: my_compound->input  # Use ->
```

### 8. Direct External to Internal Processor Links

**Problem**: Connecting external nodes directly to internal processors (must go through compound_input).

**Wrong**:
```yaml
links:
  - from: external_node
    to: my_compound->internal_processor  # Must go through input
```

**Correct**:
```yaml
links:
  - from: external_node
    to: my_compound->input
```

## Best Practices

1. **Use Simple Sequences First**: Only use compound nodes when you need complex routing or reusability - simple sequences are easier to understand and debug
2. **Meaningful Names**: Choose descriptive names for compound nodes and internal nodes that indicate their purpose
3. **Document Purpose**: Always use the `description` field to explain what the compound node does and when to use it
4. **Single Responsibility**: Each compound should have a clear, focused purpose (e.g., "PII masking" or "Team A processing")
5. **Minimal Internal Nodes**: Keep compound nodes as simple as possible - too many internal nodes make them hard to maintain
6. **Named Output Paths**: Use descriptive path names (e.g., `success`, `failure`, `high_priority`) for multiple outputs
7. **Error Handling**: Include explicit error handling paths using the `path` field on links
8. **Reusability Focus**: Design compounds to be reusable across different pipelines and teams
9. **Testing**: Test compound nodes thoroughly before deploying - use demo inputs to validate all paths
10. **Avoid Deep Nesting**: While nesting is supported, deeply nested compounds are hard to understand - keep hierarchies shallow

## Performance Considerations

- **Graph Overhead**: Compound nodes add routing overhead compared to simple sequences - use only when necessary
- **Memory Usage**: Each compound node maintains its own internal graph state - minimize the number of internal nodes
- **Path Evaluation**: Multiple output paths are evaluated for each item - complex routing patterns can impact throughput
- **Processor Count**: Total performance depends on the processors within the compound - expensive processors (regex, parsing) affect overall speed

## Compound vs Sequence

| Aspect | Compound Node | Sequence Processor |
|--------|---------------|-------------------|
| **Structure** | Graph-based with nodes and links | Linear list of processors |
| **Routing** | Multi-path with conditional branching | Single path (success/failure only) |
| **Complexity** | High - requires understanding graph connections | Low - straightforward list |
| **Reusability** | Designed for reuse across pipelines | Typically defined inline in pipeline |
| **Use Cases** | Complex routing, packs, team-specific logic | Simple linear transformations |
| **Performance** | Slightly higher overhead due to graph | Minimal overhead |
| **Error Handling** | Multiple named exit paths | Single failure path |
| **Recommended For** | Advanced users, reusable packs | Most common use cases |

**When to use Compound**: Creating reusable processor packs, complex multi-path routing, team-specific processing patterns

**When to use Sequence**: Linear processing, simple transformations, most everyday use cases

## Related Processors

- **sequence**: Linear processor chains (simpler alternative to compound for most use cases)
- **ottl_filter**: Can be used within compounds for conditional routing
- **ottl_transform**: Common processor used within compounds for data transformation
- All processors can be used within compound nodes (except input/output node types)

## Cross-References

- **edgedelta-pipelines skill**: Compound nodes can encapsulate entire processing pipelines
- **Processor Packs**: Compounds are the foundation for creating reusable processor packs
- **Node Specifications**: See `shared/pipeline/nodetype` for valid node types

## Documentation References

See https://docs.edgedelta.com for compound processor documentation.

## Notes

- Compound nodes cannot be used within sequences - they are top-level pipeline nodes only
- The delimiter `->` is reserved for referencing nodes within compounds (`compound_name->node_name`)
- Internal links use simple node names; external links use the `->` delimiter format
- Compound nodes are flattened during pipeline validation (see `NewWithFlattenCompoundNodes()`)
- Each compound must have exactly 1 input but can have multiple outputs for different routing paths
- Only processor node types are allowed inside compounds (not standalone inputs/outputs)
- Node references with the `->` delimiter are automatically created when flattening the graph
- For simple linear processing, always prefer sequence processors over compound nodes
- Compound nodes are most valuable when creating reusable processor packs for organizational standards
- The `PathNameAttributeField` can be set to track which path items took through the compound
