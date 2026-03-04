# split_with_delimiter Processor

**Type**: `split_with_delimiter`
**Category**: Data Manipulation
**Sequence Compatible**: ✓ Yes
**Source**: `internalv3/processors/split/processor.go`
**Config Struct**: `configv3.SequenceProcessor.Delimiter`

## Quick Copy

```yaml
- type: split_with_delimiter
  delimiter: "\n"
```

## Overview

The `split_with_delimiter` processor takes a single log item and splits it into multiple log items based on a specified delimiter string. This is essential for processing batched logs, multi-line events, CSV data, and any scenario where multiple logical records are contained within a single log entry.

**Transformation**: 1 input → N outputs (where N = number of delimited segments)

**Before split**:
```
Record1,Record2,Record3
```

**After split** (delimiter: `,`):
```
Item 1: Record1
Item 2: Record2
Item 3: Record3
```

## Use Cases

- **Multi-line Logs**: Split logs containing multiple newline-separated events into individual log items
- **CSV Processing**: Parse comma-separated values into individual records
- **Batch Messages**: Split batched log messages delivered as a single payload
- **Array-like Strings**: Convert delimiter-separated lists into individual items
- **Custom Protocols**: Split messages using custom delimiters (pipe, semicolon, tab, etc.)
- **Log Aggregation**: Process logs that bundle multiple events for efficiency
- **Stream Processing**: Handle streaming protocols that use delimiters for message boundaries

## Parameters

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Must be `split_with_delimiter` |
| `delimiter` | string | The delimiter string used to split the log body. Can be literal string or special characters. |

### Optional Parameters (from SequenceProcessor)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `final` | bool | false | Mark as last processor in sequence (required for final processor) |
| `disabled` | bool | false | Disable this processor without removing from config |
| `condition` | string | "" | OTTL condition - only process items when condition is true |
| `metadata` | string | "" | Arbitrary metadata for tracking/debugging |
| `comment` | string | "" | Human-readable comment about this processor |

## Delimiter Patterns

### Common Delimiters

| Delimiter | YAML Value | Use Case |
|-----------|-----------|----------|
| Newline | `"\n"` | Unix/Linux multi-line logs |
| Windows newline | `"\r\n"` | Windows multi-line logs |
| Comma | `","` | CSV data, comma-separated lists |
| Pipe | `"\|"` | Pipe-separated values |
| Tab | `"\t"` | TSV (tab-separated values) |
| Semicolon | `";"` | Semicolon-separated records |
| Space | `" "` | Space-separated tokens |
| Custom | `":::"` | Custom protocol delimiters |

### YAML Escaping Rules

**Important**: YAML requires proper escaping for special characters.

| Character | YAML Representation | Notes |
|-----------|-------------------|-------|
| Newline `\n` | `"\n"` or `"\\n"` | Double quotes required |
| Tab `\t` | `"\t"` or `"\\t"` | Double quotes required |
| Backslash `\` | `"\\"` | Must escape in double quotes |
| Quote `"` | `'\"'` or `"\""` | Use single quotes or escape |

**Best Practice**: Always use double quotes around delimiter values in YAML.

## Examples

### Example 1: Split by Newline (Unix)

```yaml
- type: split_with_delimiter
  delimiter: "\n"
```

**Input** (1 item):
```
2025-10-20 10:00:01 ERROR: Connection timeout
2025-10-20 10:00:02 WARN: Retry attempt 1
2025-10-20 10:00:03 INFO: Connection restored
```

**Output** (3 items):
```
Item 1: 2025-10-20 10:00:01 ERROR: Connection timeout
Item 2: 2025-10-20 10:00:02 WARN: Retry attempt 1
Item 3: 2025-10-20 10:00:03 INFO: Connection restored
```

**What it does**: Converts a multi-line log entry into three separate log items.

### Example 2: Split CSV by Comma

```yaml
- type: split_with_delimiter
  delimiter: ","
```

**Input** (1 item):
```
user_id:101,user_id:102,user_id:103
```

**Output** (3 items):
```
Item 1: user_id:101
Item 2: user_id:102
Item 3: user_id:103
```

**What it does**: Splits comma-separated values into individual items.

### Example 3: Split by Custom Delimiter

```yaml
- type: split_with_delimiter
  delimiter: ":::"
```

**Input** (1 item):
```
event1:::event2:::event3
```

**Output** (3 items):
```
Item 1: event1
Item 2: event2
Item 3: event3
```

**What it does**: Uses custom triple-colon delimiter to split events.

### Example 4: Split Multi-line Logs with Parsing

```yaml
- name: multiline_processor
  type: sequence
  user_description: Split and Parse Multi-line Logs
  processors:
    - type: split_with_delimiter
      delimiter: "\n"
    - type: grok
      pattern: "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level}: %{GREEDYDATA:message}"
      final: true
```

**Input** (1 item):
```
2025-10-20T10:00:01Z ERROR: Database connection failed
2025-10-20T10:00:02Z WARN: Retrying connection
```

**Output** (2 items with parsed attributes):
```
Item 1:
  body: "2025-10-20T10:00:01Z ERROR: Database connection failed"
  attributes:
    timestamp: "2025-10-20T10:00:01Z"
    level: "ERROR"
    message: "Database connection failed"

Item 2:
  body: "2025-10-20T10:00:02Z WARN: Retrying connection"
  attributes:
    timestamp: "2025-10-20T10:00:02Z"
    level: "WARN"
    message: "Retrying connection"
```

**What it does**: Splits multi-line logs by newline, then parses each line with Grok.

### Example 5: Split CSV Lines

```yaml
- name: csv_processor
  type: sequence
  user_description: Process CSV Data
  processors:
    - type: split_with_delimiter
      delimiter: "\n"
    - type: split_with_delimiter
      delimiter: ","
      final: true
```

**Input** (1 item):
```
Alice,25,Engineer
Bob,30,Manager
Charlie,28,Designer
```

**Step 1 Output** (3 items after first split):
```
Item 1: Alice,25,Engineer
Item 2: Bob,30,Manager
Item 3: Charlie,28,Designer
```

**Step 2 Output** (9 items after second split):
```
Item 1: Alice
Item 2: 25
Item 3: Engineer
Item 4: Bob
Item 5: 30
Item 6: Manager
Item 7: Charlie
Item 8: 28
Item 9: Designer
```

**Note**: For CSV parsing, consider using `ottl_transform` with `ParseCSV()` instead for proper field mapping.

### Example 6: Split Batch Messages

```yaml
- type: split_with_delimiter
  delimiter: "|"
```

**Input** (1 item):
```
[ERROR] Service A failed|[WARN] Service B degraded|[INFO] Service C healthy
```

**Output** (3 items):
```
Item 1: [ERROR] Service A failed
Item 2: [WARN] Service B degraded
Item 3: [INFO] Service C healthy
```

**What it does**: Splits pipe-delimited batch messages into individual log items.

### Example 7: Conditional Split

```yaml
- type: split_with_delimiter
  condition: 'IsMatch(body, ".*\n.*")'
  delimiter: "\n"
```

**What it does**: Only splits logs that contain newlines (leaves single-line logs unchanged).

### Example 8: Split with Metadata Preservation

```yaml
- name: split_with_tracking
  type: sequence
  processors:
    - type: ottl_transform
      statements: |
        set(attributes["batch_id"], "12345")
        set(attributes["source"], "aggregated_logs")
    - type: split_with_delimiter
      delimiter: "\n"
    - type: ottl_transform
      statements: |
        set(attributes["split_item"], "true")
      final: true
```

**What it does**:
1. Adds batch metadata (batch_id, source) to the original item
2. Splits by newline - all split items inherit the batch metadata
3. Tags each split item with `split_item=true`

**Important**: Attributes from the original item are preserved on all split items.

## Validation Rules

1. **Required Field**: Must specify `delimiter` (cannot be empty string)
2. **Non-Empty Delimiter**: `delimiter` must contain at least one character
3. **String Body**: Only works on log items with string body (non-string bodies will be skipped)
4. **YAML Escaping**: Special characters must be properly escaped in YAML

**Config Validation** (from `config_validation.go:4817-4821`):
```go
func validateSplitWithDelimiterNode(errs *ValidationError, name, fieldPrefix string, delimiter string) {
    if delimiter == "" {
        errs.AddNodeError(name, fieldPrefix+"delimiter", fmt.Errorf("is required"))
    }
}
```

## Common Pitfalls

### 1. Missing Delimiter

**Problem**: Not specifying the `delimiter` parameter causes validation error.

**Wrong**:
```yaml
- type: split_with_delimiter
```

**Correct**:
```yaml
- type: split_with_delimiter
  delimiter: "\n"
```

**Error**: "delimiter is required" (validation fails before deployment)

### 2. Incorrect YAML Escaping for Newline

**Problem**: Using literal `\n` without proper YAML quoting.

**Wrong**:
```yaml
delimiter: \n  # Treated as literal backslash + n
```

**Correct**:
```yaml
delimiter: "\n"  # Properly interpreted as newline character
```

### 3. Regex vs Literal Delimiter

**Problem**: The delimiter is a **literal string match**, not a regex pattern.

**Wrong** (expecting regex):
```yaml
delimiter: "\s+"  # Literally matches "\s+" string, not whitespace
```

**Correct** (literal space):
```yaml
delimiter: " "  # Matches single space character
```

**Note**: Use `strings.Split()` literal matching logic, not regex.

### 4. Windows vs Unix Newlines

**Problem**: Using `\n` for Windows logs that use `\r\n`.

**Windows Logs**:
```yaml
delimiter: "\r\n"  # Windows CRLF
```

**Unix/Linux Logs**:
```yaml
delimiter: "\n"  # Unix LF
```

**Cross-Platform**:
```yaml
# Split by \r\n first, then \n to handle both
- type: split_with_delimiter
  delimiter: "\r\n"
- type: split_with_delimiter
  delimiter: "\n"
```

### 5. Empty Split Results

**Problem**: Not accounting for trailing delimiters creating empty items.

**Input**:
```
record1,record2,
```

**Split by `,`**:
```
Item 1: record1
Item 2: record2
Item 3: (empty string)
```

**Solution**: Filter empty items after split:
```yaml
- type: split_with_delimiter
  delimiter: ","
- type: ottl_filter
  drop_conditions:
    - 'body == ""'
```

### 6. Large Split Counts

**Problem**: Splitting very large batched logs can create thousands of items.

**Input**: 10,000 newline-separated log entries in one item.

**Output**: 10,000 individual log items.

**Impact**: Memory usage, downstream processing overhead, potential backpressure.

**Best Practice**: Monitor split ratios and consider rate limiting or sampling if necessary.

### 7. Delimiter Not in Body

**Problem**: Using split when delimiter doesn't exist in the body.

**Input**:
```
Single line log with no delimiter
```

**Split by `,`**:
```
Item 1: Single line log with no delimiter
```

**Result**: 1 output item (no split occurs, original item passed through).

**Note**: Not an error - the processor outputs the original item unchanged.

### 8. Delimiter Escaping in YAML

**Problem**: YAML special characters require proper escaping.

**Wrong**:
```yaml
delimiter: "  # Unterminated string
```

**Correct**:
```yaml
delimiter: "\""  # Escaped quote
# or
delimiter: '"'  # Single quotes
```

**Backslash**:
```yaml
delimiter: "\\"  # Literal backslash
```

## Best Practices

1. **Quote Delimiters**: Always use double quotes around delimiter values in YAML for clarity
2. **Test Escaping**: Validate special character escaping with sample data before deployment
3. **Preserve Metadata**: Use `ottl_transform` before split to add batch tracking attributes
4. **Handle Empty**: Filter empty split results if trailing delimiters are common
5. **Newline Handling**: Be explicit about `\n` vs `\r\n` based on log source
6. **Monitor Ratios**: Track split ratios (1 input → N outputs) to detect unexpected batching
7. **Combine with Parsing**: Chain with parsers (grok, ottl_transform) to extract fields from split items
8. **Conditional Split**: Use `condition` to only split multi-line logs (avoid unnecessary processing)
9. **Order of Operations**: Place split early in sequence before field extraction
10. **Documentation**: Comment complex delimiter patterns for future maintainability

## Algorithm

The processor uses Go's `strings.Split()` function for literal string splitting:

1. **Validate Input**: Ensure item is a log item with string body
2. **Extract Body**: Get body as string
3. **Split**: Use `strings.Split(body, delimiter)` to split into array
4. **Create Items**: For each split segment:
   - Create new log item (deep copy of original)
   - Update body with the segment
   - Preserve all attributes from original item
5. **Output Items**: Return array of split items
6. **Consume Original**: Original item is consumed (not in output)

**Key Implementation** (from `processor.go:95-101`):
```go
entries := strings.Split(body, p.delimiter)
items := make([]item.Item, 0, len(entries))
for _, e := range entries {
    newItem := logItem.DeepCopy().(*item.LogItem)
    newItem.UpdateBody(e)
    items = append(items, newItem)
}
```

## Performance Considerations

- **Multiplier Effect**: 1 input → N outputs (where N = number of segments)
- **Memory**: Each split creates a deep copy of the original item (attributes preserved)
- **Large Splits**: Splitting 10,000-line logs creates 10,000 items (monitor memory)
- **Delimiter Length**: Longer delimiters have negligible performance impact
- **String Operations**: Very fast (uses optimized `strings.Split()`)
- **Downstream Impact**: Ensure downstream processors can handle increased item count

**Typical Performance**:
- Splitting 10-line log: <1ms
- Splitting 1000-line log: ~5ms
- Splitting 10,000-line log: ~50ms

## Production Example

```yaml
- name: multiline_log_processor
  type: sequence
  user_description: Process Multi-line Application Logs
  processors:
    - type: ottl_transform
      statements: |
        set(attributes["log_batch_id"], Uuid())
        set(attributes["received_at"], Now())
    - type: split_with_delimiter
      delimiter: "\n"
    - type: grok
      pattern: "%{TIMESTAMP_ISO8601:timestamp} \\[%{LOGLEVEL:level}\\] %{DATA:logger}: %{GREEDYDATA:message}"
    - type: ottl_transform
      statements: |
        set(attributes["is_error"], true) where attributes["level"] == "ERROR"
        set(attributes["severity_number"], 17) where attributes["level"] == "ERROR"
        set(attributes["severity_number"], 13) where attributes["level"] == "WARN"
        set(attributes["severity_number"], 9) where attributes["level"] == "INFO"
    - type: ottl_filter
      drop_conditions:
        - 'attributes["level"] == "DEBUG"'
      final: true
```

**Workflow**:
1. Add batch tracking metadata (batch_id, received_at)
2. Split multi-line logs by newline
3. Parse each line with Grok pattern
4. Enrich with severity levels
5. Filter out DEBUG logs

## Related Processors

- **json_unroll**: Split JSON arrays into multiple items (similar concept for JSON)
- **ottl_transform**: Use `Split()` function for OTTL-based splitting (more flexible)
- **grok**: Parse split items using patterns
- **ottl_filter**: Filter empty or unwanted split results
- **extract_json_field**: Extract fields after splitting JSON-like logs

## Cross-References

- **json_unroll**: Similar concept for JSON array splitting
- **ottl_transform**: Can use `Split()` OTTL function for attribute-level splitting
- **Best Practices**: `.claude/skills/edgedelta-pipelines/assets/references/best-practices.md`

## Documentation References

See https://docs.edgedelta.com for split_with_delimiter processor documentation.

## Notes

- **Literal Matching**: Delimiter is **literal string**, not regex (use `strings.Split()` logic)
- **Preserves Attributes**: All attributes from original item are copied to split items
- **Empty Results**: Trailing delimiters create empty string items (filter if unwanted)
- **No Delimiter Found**: If delimiter not in body, outputs 1 item unchanged
- **YAML Escaping**: Special characters (`\n`, `\t`, `\r`, etc.) must be properly escaped in YAML
- **Windows Newlines**: Use `"\r\n"` for Windows logs, `"\n"` for Unix/Linux
- **Sequence Compatible**: Can be used in sequences with other processors
- **Body Type**: Only processes items with string body (skips non-string bodies)
- **Original Consumed**: Original item is consumed (not included in output)
- **Deep Copy**: Each split item is a deep copy (independent memory allocation)
