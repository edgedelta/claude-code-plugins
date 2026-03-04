# generic_mask Processor

**Type**: `generic_mask`
**Category**: Masking & Privacy
**Sequence Compatible**: ✓ Yes
**Source**: `internalv3/processors/mask/generic/processor.go`
**Config Struct**: `configv3.Filter.CaptureGroupMasks`

## Quick Copy

```yaml
- type: generic_mask
  capture_group_masks:
    - capture_group: "(?i)(password|passwd|pwd)[:=]\\S+"
      enabled: true
      mask: "***PASSWORD***"
      name: "password"
```

## Overview

The `generic_mask` processor uses regex patterns to identify and replace sensitive data within telemetry data (logs, traces, metrics attributes). It recursively processes resource attributes, body content, and nested maps/arrays to ensure comprehensive masking of PII and confidential information.

## Use Cases

- **PII Compliance**: Mask emails, credit cards, SSNs to meet GDPR/HIPAA requirements
- **Security**: Redact passwords, API tokens, authentication credentials
- **Data Privacy**: Remove sensitive business data before sharing logs with third parties
- **Regulatory**: Comply with data protection regulations across different jurisdictions

## Parameters

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Must be `generic_mask` |
| `capture_group_masks` | []CaptureGroupMask | List of masking rules to apply |

### CaptureGroupMask Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `capture_group` | string | Yes | Regex pattern to match sensitive data. Full regex syntax supported. |
| `enabled` | bool | Yes | Whether this mask is active. Set to `false` to temporarily disable. |
| `mask` | string | Yes | Replacement text (e.g., `"***EMAIL***"`, `"[REDACTED]"`). Use empty string to remove entirely. |
| `name` | string | Yes | Descriptive identifier for this mask rule (for logging/debugging). |

### Optional Parameters (from SequenceProcessor)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `included_field_paths` | []string | all fields | Only apply masking to these specific field paths (e.g., `["body", "attributes.user_data"]`) |
| `excluded_field_paths` | []string | none | Skip masking in these field paths |
| `final` | bool | false | Mark as last processor in sequence (required for final processor) |
| `disabled` | bool | false | Disable this processor without removing from config |
| `condition` | string | "" | OTTL condition - only apply masking when condition is true |
| `metadata` | string | "" | Arbitrary metadata for tracking/debugging |
| `comment` | string | "" | Human-readable comment about this processor |

### Deprecated Parameters

| Parameter | Status | Alternative |
|-----------|--------|-------------|
| `mask_captured` | Deprecated (07-06-23) | Use `capture_group_masks` instead |

## Algorithm

The `generic_mask` processor:

1. **Traverses** all fields in the telemetry data recursively (maps, slices, strings)
2. **Filters** based on `included_field_paths` / `excluded_field_paths` if specified
3. **Applies** each enabled `capture_group_mask` in order
4. **Replaces** all regex matches with the specified `mask` value
5. **Returns** the masked telemetry data

**Implementation**: Uses Go's `regexp.ReplaceAllString` for pattern replacement (processor.go:186-191).

## Examples

### Example 1: Basic Password Masking

```yaml
- type: generic_mask
  capture_group_masks:
    - capture_group: "(?i)(password|passwd|pwd)[:=]\\S+"
      enabled: true
      mask: "***PASSWORD***"
      name: "password"
```

**Input**:
```
password=MySecret123 pwd:admin123
```

**Output**:
```
***PASSWORD*** ***PASSWORD***
```

### Example 2: Email Masking

```yaml
- type: generic_mask
  capture_group_masks:
    - capture_group: "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}"
      enabled: true
      mask: "***EMAIL***"
      name: "email"
```

**Input**:
```
Contact: user@example.com or admin@company.org
```

**Output**:
```
Contact: ***EMAIL*** or ***EMAIL***
```

### Example 3: Credit Card Masking

```yaml
- type: generic_mask
  capture_group_masks:
    - capture_group: "\\b\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}\\b"
      enabled: true
      mask: "***CC***"
      name: "credit_card"
```

**Input**:
```
Card: 4532-1234-5678-9010 or 4532123456789010
```

**Output**:
```
Card: ***CC*** or ***CC***
```

### Example 4: Multiple Masks in Sequence

```yaml
- type: sequence
  processors:
    - type: generic_mask
      capture_group_masks:
        - capture_group: "(?i)(password|passwd|pwd)[:=]\\S+"
          enabled: true
          mask: "***PASSWORD***"
          name: "password"
    - type: generic_mask
      capture_group_masks:
        - capture_group: "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}"
          enabled: true
          mask: "***EMAIL***"
          name: "email"
    - type: generic_mask
      capture_group_masks:
        - capture_group: "\\b\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}\\b"
          enabled: true
          mask: "***CC***"
          name: "credit_card"
      final: true
```

### Example 5: Field-Specific Masking

```yaml
- type: generic_mask
  included_field_paths:
    - "body"
    - "attributes.request_data"
  capture_group_masks:
    - capture_group: "token[:=][\\w-]+"
      enabled: true
      mask: "token=***REDACTED***"
      name: "api_token"
```

Only masks tokens in `body` and `attributes.request_data` fields, leaving other fields untouched.

### Example 6: Conditional Masking

```yaml
- type: generic_mask
  condition: 'attributes["environment"] == "production"'
  capture_group_masks:
    - capture_group: "(?i)secret[:=]\\S+"
      enabled: true
      mask: "***SECRET***"
      name: "secret"
```

Only applies masking when the `environment` attribute equals `production`.

### Example 7: SSN Masking

```yaml
- type: generic_mask
  capture_group_masks:
    - capture_group: "\\b\\d{3}-\\d{2}-\\d{4}\\b"
      enabled: true
      mask: "***SSN***"
      name: "ssn"
```

**Input**:
```
SSN: 123-45-6789
```

**Output**:
```
SSN: ***SSN***
```

### Example 8: Complete Removal (Empty Mask)

```yaml
- type: generic_mask
  capture_group_masks:
    - capture_group: "internal_id[:=]\\S+"
      enabled: true
      mask: ""
      name: "internal_id_removal"
```

Completely removes matched text instead of replacing with placeholder.

## Validation Rules

1. **Required Fields**: Must specify `type`, `capture_group_masks`
2. **Non-Empty List**: `capture_group_masks` must contain at least one entry
3. **Valid Regex**: Each `capture_group` must be a valid Go regex pattern
4. **CaptureGroupMask Complete**: Each entry must have all 4 fields (`enabled`, `capture_group`, `mask`, `name`)
5. **Unique Names**: While not enforced, using unique `name` values improves debugging
6. **Field Paths**: If using `included_field_paths` / `excluded_field_paths`, they should be valid field references

## Common Pitfalls

### 1. Regex Escaping in YAML

**Problem**: Backslashes in regex patterns need double-escaping in YAML.

**Wrong**:
```yaml
capture_group: "\d{4}"  # YAML interprets \d as escape sequence
```

**Correct**:
```yaml
capture_group: "\\d{4}"  # Double backslash for YAML
```

### 2. Case Sensitivity

**Problem**: Patterns are case-sensitive by default.

**Solution**: Use `(?i)` flag for case-insensitive matching:
```yaml
capture_group: "(?i)(password|pwd)"  # Matches PASSWORD, Password, pwd
```

### 3. Greedy vs Non-Greedy

**Problem**: Greedy patterns may mask too much.

**Example**:
```yaml
capture_group: "token[:=].*"  # Greedy - masks entire rest of line
```

**Better**:
```yaml
capture_group: "token[:=]\\S+"  # Non-greedy - masks only the token value
```

### 4. Disabled Masks Still in Config

**Problem**: Setting `enabled: false` disables the mask but keeps it in config.

**Tip**: Use this for temporary disabling during debugging, remove entirely for production.

### 5. Performance with Many Masks

**Problem**: Each mask is a separate regex evaluation. Many complex patterns can impact performance.

**Solution**: Combine related patterns when possible:
```yaml
# Instead of 3 separate masks
capture_group: "(password|passwd|pwd|secret|token)[:=]\\S+"
```

### 6. Order Matters

**Problem**: Later masks may not match if earlier masks already replaced the text.

**Solution**: Apply masks in order from most specific to most general.

### 7. Nested JSON Field Paths

**Problem**: Complex nested JSON may require recursive masking.

**Note**: `generic_mask` handles this automatically - it traverses all nested structures.

## Best Practices

1. **Test Patterns**: Validate regex patterns with sample data before deployment
2. **Descriptive Names**: Use clear `name` values (e.g., `"ssn"`, `"credit_card"`) for debugging
3. **Mask Early**: Apply masking as early as possible in the sequence to prevent downstream leaks
4. **Use Predefined Patterns**: Consider `predefined_pattern` field for common cases (if available)
5. **Performance**: Use `included_field_paths` to limit scope when masking only specific fields
6. **Audit**: Maintain a list of what data is being masked for compliance documentation
7. **Empty Mask**: Use `mask: ""` for complete removal vs placeholder replacement
8. **Compliance**: Ensure regex patterns cover all format variations (e.g., credit cards with/without dashes)

## Production Example (from Template 1)

```yaml
- name: pii_masking_and_metrics
  type: sequence
  user_description: PII Masking and Metrics Extraction
  processors:
    - type: generic_mask
      capture_group_masks:
        - capture_group: "(?i)(password|passwd|pwd)[:=]\\S+"
          enabled: true
          mask: "***PASSWORD_REDACTED***"
          name: "password"
    - type: generic_mask
      capture_group_masks:
        - capture_group: "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}"
          enabled: true
          mask: "***EMAIL***"
          name: "email"
    - type: generic_mask
      capture_group_masks:
        - capture_group: "\\b\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}\\b"
          enabled: true
          mask: "***CC***"
          name: "credit_card"
    - type: extract_metric
      interval: 1m
      final: true
```

This sequence:
1. Masks passwords (case-insensitive)
2. Masks email addresses
3. Masks credit card numbers (with or without separators)
4. Extracts metrics (final processor)

**Testing Status**: ✓ Deployed and validated (Pipeline ID: 529198b9-2486-4f5c-9d38-3c2503756ce2)

## Related Processors

- **delete_empty_values**: Remove empty/nil attributes after masking
- **ottl_transform**: For conditional masking with complex logic
- **ottl_filter**: Filter out events based on sensitive data presence

## Cross-References

- **edgedelta-pipelines skill**: Template 1 (Log PII Masking), Template 3 (Mixed Telemetry)
- **Best Practices**: `.claude/skills/edgedelta-pipelines/assets/references/best-practices.md` (Security & Compliance section)

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/mask-processor/

## Notes

- Documentation uses "Mask Processor" terminology while source code uses `generic_mask` as the type value
- The processor recursively handles nested JSON/maps automatically
- Deprecated `mask_captured` field should not be used (use `capture_group_masks` instead)
- All regex patterns are evaluated using Go's `regexp` package
