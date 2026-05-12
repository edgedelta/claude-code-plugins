---
name: edgedelta-reference
version: 2.0.0
last_updated: 2026-03-03
description: "Reference for EdgeDelta v3 pipeline components covering 30 sources, 38 processors, and 54 destinations with documentation links and validation rules. Use when users need component syntax help, ask 'what sources does EdgeDelta support', ask 'how do I configure a destination', ask about processor parameters, need YAML examples for pipeline nodes, or want documentation links."
dependencies:
  - EdgeDelta pipeline v3
  - YAML knowledge
---

# EdgeDelta Reference Skill

Quick-lookup documentation for EdgeDelta pipeline v3 sources, processors, and destinations with copy-paste examples and documentation links.

## Component Categories

| Category | Count | Reference |
|---|---|---|
| Sources | 30 | [references/sources/index.md](references/sources/index.md) |
| Processors | 38 | [references/processors/](references/processors/) |
| Destinations | 54 | [references/destinations/index.md](references/destinations/index.md) |

## When to Use This Skill

Activate this skill when:
- User asks about EdgeDelta processors, inputs, outputs, or sources
- User asks "what sources does EdgeDelta support?" or "what destinations are available?"
- User needs processor specifications, parameters, or YAML syntax
- User asks "what processors are available" or "how do I use X processor"
- User is building a pipeline and needs component syntax
- User asks for examples of any pipeline node (source, processor, destination)
- User asks "how do I configure a [Kafka/Loki/Splunk/etc] destination?"
- Another skill (like edgedelta-pipelines) needs component reference lookup
- User wants to know which processors are sequence-compatible
- User wants to look up a documentation link for any EdgeDelta component

## Do NOT Use This Skill When

- User wants to deploy a complete pipeline → use `edgedelta-pipelines` skill instead
- User asks about EdgeDelta dashboards → use `edgedelta-dashboards` skill
- User asks about OTTL functions or syntax → use `edgedelta-ottl` skill instead
- User needs OTTL function reference (ParseJSON, EDXRedis, etc.) → use `edgedelta-ottl` skill
- General observability questions without EdgeDelta context

## Primary Resources

### MASTER_INDEX.md
Quick lookup for all 23 sequence-compatible processors: copy-paste snippets, use cases, ordering best practices, and links to detailed references.

### Detailed Processor References
**Location**: `references/processors/`

**38 Processors Documented (23 with full depth, 15 stubs with doc links)**:

**Core Transformation & Filtering (3)**:
- `ottl_transform.md` - OTTL transformations, attribute manipulation
- `ottl_filter.md` - OTTL-based filtering and dropping
- `ottl_context_filter.md` - Log context collection for debugging

**Masking & Privacy (2)**:
- `generic_mask.md` - PII masking, regex-based redaction
- `delete_empty_values.md` - Remove nil/empty attributes and placeholders

**Sampling & Rate Limiting (4)**:
- `sample.md` - Probabilistic hash-based sampling
- `tail_sample.md` - Intelligent trace sampling (10 policy types)
- `rate_limit.md` - Throughput rate limiting
- `suppress.md` - Stateful duplicate suppression

**Metrics Extraction & Conversion (4)**:
- `extract_metric.md` - Log-to-metric conversion, OTTL conditions
- `aggregate_metric.md` - Metric aggregation and roll-up (10 aggregation types)
- `cumulative_to_delta.md` - Cumulative to delta conversion
- `log_to_pattern_metric.md` - DRAIN pattern clustering with metrics

**Data Manipulation (5)**:
- `json_unroll.md` - Array unrolling, batch processing
- `extract_json_field.md` - Extract JSON field with array expansion
- `split_with_delimiter.md` - Split logs by delimiter (1 → N)
- `grok.md` - Grok pattern-based log parsing (30+ patterns)
- `lookup.md` - CSV/DB enrichment with multiple match modes

**Deduplication (1)**:
- `dedup.md` - Content-based exact deduplication (xxHash)

**Special Processors (1)**:
- `deotel.md` - DeOTEL format conversion (MUST be last)

**Nested Structures (3)**:
- `sequence.md` - Nested sequences for modular organization (1067 lines)
- `compound.md` - Graph-based custom routing (755 lines)
- `sequence_input_output.md` - Internal processors (reference)

**Stub Processors (15 with doc links, basic examples)**:
- `parse_csv.md`, `parse_json.md`, `parse_key_value.md`, `parse_regex.md`, `parse_severity.md`
- `parse_timestamp.md`, `parse_xml.md`
- `add_field.md`, `copy_field.md`, `delete_field.md`, `pack.md`
- `redis_enrichment.md`, `rollup_metric.md`, `stateful_alert.md`
- `route.md`, `code.md`, `comment.md`, `conditional_group.md`

### Sources Reference
**Location**: `references/sources/index.md`

Index of all 30 EdgeDelta v3 sources organized by category (Log, Metric, Trace, Cloud Native, Hybrid) with YAML type, description, key parameters, and docs.edgedelta.com link.

### Destinations Reference
**Location**: `references/destinations/index.md`

Index of all 54 EdgeDelta v3 destinations organized by category (Observability, Storage, Security/SIEM, Custom/Streaming) with YAML type, description, key parameters, and docs.edgedelta.com link.

## How to Use This Skill

### Workflow 1: Quick Processor Lookup

**When**: User asks "How do I use X processor?" or "What's the syntax for X?"

**Steps**:
1. Read `MASTER_INDEX.md` and find the processor
2. Return the Quick Copy snippet with common use cases
3. If a detailed reference exists in `references/processors/`, offer deeper examples

### Workflow 2: Detailed Processor Documentation

**When**: User needs in-depth info about generic_mask, extract_metric, ottl_transform, or json_unroll

**Steps**:
1. Read the specific processor reference file (e.g., `references/processors/generic_mask.md`)
2. Provide relevant section based on user's need:
   - Examples for "how do I..." questions
   - Validation Rules for "is this valid?" questions
   - Common Pitfalls for "this isn't working" questions
   - Best Practices for "what's the right way?" questions

**Example**:
```
User: "I'm getting errors when masking credit cards. What am I doing wrong?"
Assistant: [Reads references/processors/generic_mask.md, goes to Common Pitfalls section]
Assistant: "Common issues with credit card masking:

1. **Regex Escaping in YAML**: Backslashes need double-escaping in YAML.

   Wrong:
   ```yaml
   capture_group: "\d{4}"  # YAML interprets \d as escape
   ```

   Correct:
   ```yaml
   capture_group: "\\d{4}"  # Double backslash for YAML
   ```

2. **Matching with/without separators**:
   ```yaml
   capture_group: "\\b\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}\\b"
   ```
   This pattern handles both `4532-1234-5678-9010` and `4532123456789010`.

Can you share your current configuration? I can help debug the specific issue."
```

### Workflow 3: List Available Components

**When**: User asks "what processors/sources/destinations are available?"

**Steps**:
1. Read `MASTER_INDEX.md` (processors) or the relevant index file
2. Present the categorized list

### Workflow 4: Processor Combination Guidance

**When**: User asks how to combine processors or "best practices for X"

**Steps**:
1. Read `MASTER_INDEX.md` "Common Combinations" section
2. Provide relevant example
3. Explain the ordering rationale

**Example**:
```
User: "I need to mask PII and extract error metrics from logs. What's the best way?"
Assistant: [Reads MASTER_INDEX.md, Common Combinations section]
Assistant: "For PII masking + metrics extraction, use this pattern:

```yaml
processors:
  - type: generic_mask
    capture_group_masks:
      - capture_group: "(?i)(password|passwd)[:=]\\S+"
        enabled: true
        mask: "***PASSWORD***"
        name: "password"
  - type: extract_metric
    extract_metric_rules:
      - name: "errors_total"
        conditions:
          - 'IsMatch(body, "ERROR")'
        sum:
          aggregation_temporality: delta
          is_monotonic: true
          value: 1
    interval: 1m
    final: true
```

**Why this order?**
1. **Mask first**: Ensures sensitive data is redacted before any processing
2. **Metrics second**: Extract metrics from already-masked logs
3. **Final flag**: mark extract_metric as the last processor

This follows best practice: Filter → Parse → Mask → Transform → Sample → Metrics."
```

### Workflow 5: Cross-Skill Reference (from edgedelta-pipelines)

**When**: The `edgedelta-pipelines` skill needs processor reference while building a pipeline

**Steps**:
1. Read the relevant processor reference from `MASTER_INDEX.md` or `references/processors/`
2. Provide syntax/examples for incorporation into the pipeline

### Workflow 6: Validation & Troubleshooting

**When**: User has a broken config or asks "is this valid?"

**Steps**:
1. Identify which processor(s) are involved
2. Read the detailed reference's Validation Rules and Common Pitfalls
3. Check the user's config against documented constraints
4. Provide the corrected YAML
5. Verify the fix satisfies all validation rules before presenting

### Workflow 7: Source or Destination Lookup

**When**: User asks "what sources does EdgeDelta support?", "how do I configure a Kafka destination?", or "what's the YAML type for Loki?"

**Steps**:
1. Read `references/sources/index.md` or `references/destinations/index.md`
2. Find the relevant entry
3. Return the YAML type, key parameters, and docs link

## Important Notes

### Sequence-Compatible Processors Only

**Only these 24 processors** can be used inside `sequence` nodes:
- ottl_transform, ottl_filter, ottl_context_filter
- generic_mask, delete_empty_values
- sample, tail_sample, rate_limit, suppress
- extract_metric, aggregate_metric, cumulative_to_delta, log_to_pattern_metric
- json_unroll, extract_json_field, split_with_delimiter, grok
- dedup
- lookup
- deotel
- sequence, compound, sequence_input, sequence_output

**route_ottl is NOT sequence-compatible** - it's used for top-level multi-path routing.

### Source Code Validation

All processor references are validated against EdgeDelta source code:
- **Source**: `EdgeDelta source code repository`
- **Processor List**: `internalv3/discover/sequence_node.go` (lines 52-76)
- **Config Structs**: `configv3/config.go`
- **Validation**: `configv3/config_validation.go`

### Documentation Disparities

Check `documentation_disparities.md` when:
- User reports conflicting information
- Processor name in docs differs from YAML `type`
- Parameters documented differently than source code

**Example Disparity**:
- Source code: `type: generic_mask`
- Docs.edgedelta.com: "Mask Processor" or "Mask node"

## Cross-Skill Delegation

- **`edgedelta-pipelines`**: End-to-end pipeline creation/deployment. This skill provides the component-level reference it consumes.
- **`edgedelta-ottl`**: OTTL function syntax and EDX extensions (124 functions). Refer there for Set, ParseJSON, EDXRedis; refer here for ottl_transform/ottl_filter processor structure.
