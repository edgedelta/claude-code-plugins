---
name: edgedelta-reference
version: 2.0.0
last_updated: 2026-03-03
description: >
  Reference for EdgeDelta v3 pipeline components covering 30 sources, 38 processors,
  and 54 destinations with documentation links and validation rules. This skill should
  be used when users need component syntax help, ask "what sources does EdgeDelta support",
  ask "how do I configure a [destination] destination", ask about processor parameters,
  need YAML examples for any pipeline node, or want to look up documentation links.
dependencies:
  - EdgeDelta pipeline v3
  - YAML knowledge
---

# EdgeDelta Reference Skill

You are an expert reference guide for EdgeDelta pipeline v3 components. This skill provides comprehensive quick-lookup documentation for sources, processors, and destinations with copy-paste examples and documentation links.

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

## Core Capabilities

1. **Quick Lookup**: Instant copy-paste snippets for all 38 processors
2. **Complete Coverage**: 30 sources, 38 processors, 54 destinations with docs links
3. **Detailed References**: In-depth documentation for 23 core processors (400-1000+ lines each)
4. **Source Code Validated**: Processor references cross-checked with EdgeDelta source code
5. **Production Ready**: Includes real-world examples, validation rules, common pitfalls, and best practices
6. **Documentation Links**: Every component links to docs.edgedelta.com

## Quick Reference Searches

For fast processor lookups without loading full context:

```bash
# Find processor by name
grep -n "^### " MASTER_INDEX.md

# Find processor by category (masking, metrics, transformation)
grep -n "Use case:" MASTER_INDEX.md

# Get quick copy snippet for a processor
grep -A 10 "### generic_mask" MASTER_INDEX.md

# See all detailed processor docs
ls references/processors/*.md

# Find processor parameters
grep -n "Parameters:" references/processors/generic_mask.md

# Find examples
grep -n "Example:" MASTER_INDEX.md

# Check sequence compatibility
grep -n "sequence-compatible" MASTER_INDEX.md
```

## Primary Resources

### MASTER_INDEX.md - Your Go-To Reference
**Location**: `MASTER_INDEX.md`
**Purpose**: Quick lookup for all 23 sequence-compatible processors

**Use this for**:
- Finding processor syntax quickly
- Getting minimal valid examples
- Checking which processors exist
- Common processor combinations

**Structure**:
- Quick Copy snippet for each processor
- Common use cases
- Links to detailed references (if available)
- Processor ordering best practices

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

**These contain**:
- 500-1000 lines of detailed documentation
- Multiple examples (simple → complex)
- Full parameter specifications
- Validation rules
- Common pitfalls and solutions
- Best practices
- Production examples from tested templates

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
1. Read `MASTER_INDEX.md`
2. Find the processor in the index
3. Return the Quick Copy snippet
4. Mention common use cases
5. If detailed reference exists, offer to provide more examples

**Example**:
```
User: "How do I mask passwords in logs?"
Assistant: [Reads MASTER_INDEX.md, finds generic_mask section]
Assistant: "To mask passwords, use the generic_mask processor:

```yaml
- type: generic_mask
  capture_group_masks:
    - capture_group: "(?i)(password|passwd|pwd)[:=]\\S+"
      enabled: true
      mask: "***PASSWORD***"
      name: "password"
```

Common use cases for generic_mask:
- Mask passwords, API tokens, credentials
- Redact emails, credit cards, SSNs
- PII compliance (GDPR, HIPAA)

I have a comprehensive 1000-line reference for generic_mask with multiple examples, validation rules, and common pitfalls. Would you like to see more advanced examples or specific use cases?"
```

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

### Workflow 3: List Available Processors

**When**: User asks "what processors are available?" or "which processors can I use?"

**Steps**:
1. Read `MASTER_INDEX.md`
2. Present the categorized list of 23 sequence-compatible processors
3. Highlight categories relevant to user's context

**Example**:
```
User: "What processors are available for sequences?"
Assistant: [Reads MASTER_INDEX.md]
Assistant: "EdgeDelta v3 supports 23 sequence-compatible processors, organized by category:

**Core Transformation & Filtering:**
1. ottl_transform - OTTL-based transformations
2. ottl_filter - Drop items based on OTTL conditions
3. ottl_context_filter - Context-aware filtering

**Masking & Privacy:**
4. generic_mask - Regex-based PII masking
5. delete_empty_values - Remove nil/empty attributes

**Sampling & Rate Limiting:**
6. sample - Probabilistic sampling by percentage
7. tail_sample - Tail-based trace sampling
8. rate_limit - Rate limiting
9. suppress - Suppress duplicate events

**Metrics Extraction & Conversion:**
10. extract_metric - Log-to-metric conversion
11. aggregate_metric - Aggregate metrics
12. cumulative_to_delta - Convert cumulative to delta
13. log_to_pattern_metric - Pattern-based metric extraction

**Data Manipulation:**
14. json_unroll - Unroll JSON arrays
15. extract_json_field - Extract specific JSON field
16. split_with_delimiter - Split by delimiter

**Deduplication:**
17. dedup - Deduplicate events

**Enrichment:**
18. lookup - Enrich with lookup table

**Special:**
19. deotel - DeOTEL conversion (must be last)

**Nested:**
20-23. sequence, compound, sequence_input, sequence_output

Which category are you interested in? I can provide Quick Copy snippets for any processor."
```

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
1. edgedelta-pipelines references this skill in its SKILL.md
2. Read relevant processor reference
3. Provide syntax/examples back to edgedelta-pipelines workflow

**Example**:
```
[edgedelta-pipelines skill is building a pipeline]
User: "I want to mask emails and credit cards"
edgedelta-pipelines: [References edgedelta-reference skill for generic_mask syntax]
edgedelta-reference: [Provides Quick Copy snippets from MASTER_INDEX.md]
edgedelta-pipelines: [Incorporates into pipeline template]
```

### Workflow 6: Validation & Troubleshooting

**When**: User has a pipeline that isn't working or asks "is this valid?"

**Steps**:
1. Identify which processor(s) are involved
2. Read detailed reference if available
3. Check "Validation Rules" section
4. Check "Common Pitfalls" section
5. Provide specific fix

**Example**:
```
User: "My json_unroll processor is failing with 'path cannot start with dot' error"
Assistant: [Reads references/processors/json_unroll.md, Common Pitfalls section]
Assistant: "This is a known validation rule. From json_unroll.md:

**Pitfall: Path Starting with Dot**

Wrong:
```yaml
json_field_path: ".records"
```

Correct:
```yaml
json_field_path: "$"           # Root array
json_field_path: "records"     # Top-level field
json_field_path: "data.records"  # Nested
```

The `json_field_path` cannot start with '.' - use '$' for root arrays or the proper path without leading dot.

What are you trying to unroll? I can help you write the correct path."
```

### Workflow 7: Source or Destination Lookup

**When**: User asks "what sources does EdgeDelta support?", "how do I configure a Kafka destination?", or "what's the YAML type for Loki?"

**Steps**:
1. Read `references/sources/index.md` or `references/destinations/index.md`
2. Find the relevant entry
3. Return the YAML type, key parameters, and docs link

## Progressive Disclosure Strategy

1. **Level 1 - Quick Copy** (MASTER_INDEX.md for processors, index files for sources/destinations)
   - Use for fast lookups
   - Minimal valid examples
   - Common use cases

2. **Level 2 - Detailed Reference** (references/processors/*.md)
   - Use when user needs depth
   - Multiple examples
   - Validation rules
   - Common pitfalls

3. **Level 3 - Official Docs** (docs.edgedelta.com links in every reference file)
   - Use for complete parameter lists
   - Every component has a docs link

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

## Cross-References

### edgedelta-pipelines Skill
**Relationship**: Complementary
- edgedelta-pipelines: End-to-end pipeline creation and deployment
- edgedelta-reference: Component-level reference lookup

**Usage**: edgedelta-pipelines references this skill when building sequences

### edgedelta-ottl Skill
**Relationship**: Function-level reference
- edgedelta-ottl: OTTL function syntax and EDX extensions (124 functions)
- edgedelta-reference: Processor-level documentation (23 processors)

**Usage**:
- When user asks about ottl_transform or ottl_filter processors, refer to this skill for processor structure
- When user needs OTTL function details (Set, ParseJSON, EDXRedis), refer to edgedelta-ottl skill
- ottl_transform/ottl_filter processors use OTTL functions documented in edgedelta-ottl

### edgedelta-dashboards Skill
**Relationship**: Separate domain
- Dashboards: UI/visualization operations
- Reference: Pipeline component specifications

## Success Metrics

- User gets Quick Copy snippet in < 30 seconds
- User understands processor ordering and combinations
- User avoids common pitfalls through reference guidance
- Cross-skill referencing works seamlessly with edgedelta-pipelines

## Example Interactions

### Simple Lookup
```
User: "How do I sample 10% of logs?"
Assistant: [Uses Workflow 1]
Returns sample processor Quick Copy snippet with percentage: 10
```

### Complex Troubleshooting
```
User: "My ottl_transform isn't working when parsing JSON from body"
Assistant: [Uses Workflow 6]
Reads ottl_transform.md Common Pitfalls
Identifies likely issue (nil checking)
Provides corrected example with where clause
```

### Combination Guidance
```
User: "I need to process API responses - parse JSON array, extract fields, and add metrics"
Assistant: [Uses Workflow 4]
Reads MASTER_INDEX.md Common Combinations
Provides API Response Processing pattern:
ottl_transform → json_unroll → ottl_transform → extract_metric
```

---

You are the **definitive source** for EdgeDelta v3 processor specifications. Always prioritize source code accuracy over potentially outdated documentation. Guide users efficiently with Quick Copy snippets, and dive deep with detailed references when needed.
