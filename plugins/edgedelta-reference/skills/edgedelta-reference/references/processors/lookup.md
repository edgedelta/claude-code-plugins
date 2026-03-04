# lookup Processor

**Type**: `lookup`
**Category**: Enrichment & Data Augmentation
**Sequence Compatible**: ✓ Yes
**Source**: `internalv3/processors/lookup/processor.go`
**Config Struct**: `configv3.Lookup`

## Quick Copy

```yaml
- type: lookup
  location_path: "/path/to/lookup_table.csv"
  reload_period: 5m
  match_mode: exact
  key_fields:
    - event_field: 'attributes["error_code"]'
      lookup_field: "ErrorCode"
  out_fields:
    - event_field: 'attributes["error_message"]'
      lookup_field: "ErrorMessage"
```

## Overview

The `lookup` processor enriches telemetry items (logs, metrics, traces) by matching fields against CSV lookup tables and adding corresponding values from the table. This enables powerful data enrichment scenarios such as adding human-readable descriptions to error codes, mapping IP addresses to locations, enriching user IDs with profile information, or augmenting service names with team ownership data.

The processor supports multiple match modes (exact, regex, contain, prefix, suffix), composite key matching, multiple output fields, default values, and automatic reload of lookup tables for dynamic updates.

## Use Cases

- **Error Code Enrichment**: Add user-friendly messages and severity levels to numeric error codes
- **IP Geolocation**: Map IP addresses to geographic locations, ISPs, or network information
- **User Profile Enrichment**: Augment user IDs with names, departments, roles, or contact information
- **Service Metadata**: Add team ownership, SLA tiers, or support contacts to service names
- **Status Code Translation**: Convert numeric status codes to human-readable descriptions
- **Threat Intelligence**: Enrich security logs with threat actor information, malware signatures, or IOC data
- **Asset Management**: Add asset details like owner, location, or cost center to infrastructure logs
- **Product Catalog**: Augment product IDs with names, categories, prices, or inventory status

## Parameters

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Must be `lookup` |
| `location_path` | string | Path to CSV lookup table file (local file system path) |
| `key_fields` | []LookupField | Fields to match against lookup table (at least one required) |
| `out_fields` | []LookupField | Fields to enrich with values from lookup table (at least one required) |

### Optional Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `reload_period` | duration | `0s` (no reload) | How often to reload the lookup table from disk (e.g., `5m`, `1h`) |
| `match_mode` | string | `exact` | Match algorithm: `exact`, `regex`, `contain`, `prefix`, or `suffix` |
| `match_option` | string | `first` | For `contain`, `prefix`, `suffix` modes: `first` or `all` |
| `regex_option` | string | `first` | For `regex` mode: `first` or `all` |
| `ignore_case` | bool | false | Case-insensitive matching (not supported for `regex` mode) |
| `final` | bool | false | Mark as last processor in sequence (required for final processor) |
| `disabled` | bool | false | Disable this processor without removing from config |
| `condition` | string | "" | OTTL condition - only process items when condition is true |
| `metadata` | string | "" | Arbitrary metadata for tracking/debugging |
| `comment` | string | "" | Human-readable comment about this processor |

### LookupField Parameters (key_fields)

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `event_field` | string | Yes | Expression to extract value from telemetry item (CEL or OTTL) |
| `lookup_field` | string | Yes | Column name in CSV file to match against |

**Note**: `default_value` and `append_mode` are **not supported** for `key_fields`.

### LookupField Parameters (out_fields)

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `event_field` | string | Yes | Field path where enriched value will be written (e.g., `attributes["user_message"]`) |
| `lookup_field` | string | Yes | Column name in CSV file to extract value from |
| `default_value` | string | No | Value to use if no match is found (if not specified, field is not added) |
| `append_mode` | bool | false | When multiple matches occur, join values with comma instead of creating array |

## Match Modes

### exact (Default)

Performs exact string matching between event field and lookup field values. Case-sensitive unless `ignore_case: true`.

**When to use**: For exact matches like error codes, IDs, status codes.

**Performance**: Fastest match mode.

```yaml
match_mode: exact
ignore_case: false  # Optional: true for case-insensitive
```

### regex

Uses regular expressions defined in the lookup table to match against event field values. The CSV lookup field contains regex patterns.

**When to use**: For pattern-based matching like log message classification, flexible ID formats.

**Performance**: Slower than exact matching due to regex evaluation.

**Options**:
- `regex_option: first` - Stop after first match (default)
- `regex_option: all` - Find all matching patterns

**Important**: `ignore_case` is **not supported** with regex mode. Use `(?i)` flag in regex patterns for case-insensitive matching.

```yaml
match_mode: regex
regex_option: first  # or "all"
```

### contain

Checks if the lookup field value is contained within the event field value (substring match).

**When to use**: For partial matches like checking if a log message contains specific keywords.

**Performance**: Fast string search.

**Options**:
- `match_option: first` - Stop after first match (default)
- `match_option: all` - Find all containing matches

```yaml
match_mode: contain
match_option: first  # or "all"
ignore_case: false   # Optional: true for case-insensitive
```

### prefix

Checks if the event field value starts with the lookup field value.

**When to use**: For hierarchical identifiers, URL paths, or namespaced values.

**Performance**: Fast prefix comparison.

**Options**:
- `match_option: first` - Stop after first match (default)
- `match_option: all` - Find all prefix matches

```yaml
match_mode: prefix
match_option: first  # or "all"
ignore_case: false   # Optional: true for case-insensitive
```

### suffix

Checks if the event field value ends with the lookup field value.

**When to use**: For file extensions, domain names, or suffix-based classification.

**Performance**: Fast suffix comparison.

**Options**:
- `match_option: first` - Stop after first match (default)
- `match_option: all` - Find all suffix matches

```yaml
match_mode: suffix
match_option: first  # or "all"
ignore_case: false   # Optional: true for case-insensitive
```

## Examples

### Example 1: Basic CSV Lookup (Exact Match)

**CSV File** (`error_codes.csv`):
```csv
ErrorCode,ErrorMessage,Severity
1001,Invalid user input,Warning
1002,Database connection failed,Critical
1003,Timeout occurred,Error
1004,Resource not found,Warning
1005,Authentication failed,Critical
```

**Configuration**:
```yaml
- type: lookup
  location_path: "/etc/edgedelta/error_codes.csv"
  reload_period: 5m
  match_mode: exact
  key_fields:
    - event_field: 'attributes["error_code"]'
      lookup_field: "ErrorCode"
  out_fields:
    - event_field: 'attributes["error_message"]'
      lookup_field: "ErrorMessage"
    - event_field: 'attributes["severity"]'
      lookup_field: "Severity"
```

**What it does**: Matches `error_code` attribute against the ErrorCode column and enriches the log with ErrorMessage and Severity values.

**Input Log**:
```json
{
  "body": "Application error occurred",
  "attributes": {
    "error_code": "1002"
  }
}
```

**Output Log**:
```json
{
  "body": "Application error occurred",
  "attributes": {
    "error_code": "1002",
    "error_message": "Database connection failed",
    "severity": "Critical"
  }
}
```

### Example 2: Regex Match Mode

**CSV File** (`log_patterns.csv`):
```csv
Pattern,Category,Priority
^ERROR.*database.*,Database,High
^WARN.*memory.*,Memory,Medium
^INFO.*startup.*,Startup,Low
.*timeout.*,Network,High
```

**Configuration**:
```yaml
- type: lookup
  location_path: "/etc/edgedelta/log_patterns.csv"
  reload_period: 10m
  match_mode: regex
  regex_option: first
  key_fields:
    - event_field: 'body'
      lookup_field: "Pattern"
  out_fields:
    - event_field: 'attributes["log_category"]'
      lookup_field: "Category"
    - event_field: 'attributes["priority"]'
      lookup_field: "Priority"
```

**What it does**: Matches log body against regex patterns in the Pattern column and enriches with Category and Priority.

**Input Log**:
```json
{
  "body": "ERROR: database connection pool exhausted",
  "attributes": {}
}
```

**Output Log**:
```json
{
  "body": "ERROR: database connection pool exhausted",
  "attributes": {
    "log_category": "Database",
    "priority": "High"
  }
}
```

### Example 3: Case-Insensitive Matching

**CSV File** (`status_codes.csv`):
```csv
Status,Description,Action
success,Operation completed successfully,none
warning,Operation completed with warnings,review
error,Operation failed,investigate
fatal,Critical system failure,escalate
```

**Configuration**:
```yaml
- type: lookup
  location_path: "/etc/edgedelta/status_codes.csv"
  reload_period: 5m
  match_mode: exact
  ignore_case: true
  key_fields:
    - event_field: 'attributes["status"]'
      lookup_field: "Status"
  out_fields:
    - event_field: 'attributes["description"]'
      lookup_field: "Description"
    - event_field: 'attributes["action"]'
      lookup_field: "Action"
```

**What it does**: Performs case-insensitive exact matching on status field.

**Input Log** (note uppercase "ERROR"):
```json
{
  "attributes": {
    "status": "ERROR"
  }
}
```

**Output Log**:
```json
{
  "attributes": {
    "status": "ERROR",
    "description": "Operation failed",
    "action": "investigate"
  }
}
```

### Example 4: Multiple Key Fields (Composite Keys)

**CSV File** (`access_control.csv`):
```csv
User,System,IsAllowed,Reason
alice,prod_db,true,Admin user
alice,staging_db,true,Admin user
bob,prod_db,false,Insufficient privileges
bob,staging_db,true,Developer access
charlie,prod_db,true,DBA role
```

**Configuration**:
```yaml
- type: lookup
  location_path: "/etc/edgedelta/access_control.csv"
  reload_period: 2m
  match_mode: exact
  key_fields:
    - event_field: 'attributes["user"]'
      lookup_field: "User"
    - event_field: 'attributes["system"]'
      lookup_field: "System"
  out_fields:
    - event_field: 'attributes["is_allowed"]'
      lookup_field: "IsAllowed"
    - event_field: 'attributes["reason"]'
      lookup_field: "Reason"
```

**What it does**: Matches on both user AND system fields (composite key) to determine access rights.

**Input Log**:
```json
{
  "body": "Access attempt",
  "attributes": {
    "user": "bob",
    "system": "prod_db"
  }
}
```

**Output Log**:
```json
{
  "body": "Access attempt",
  "attributes": {
    "user": "bob",
    "system": "prod_db",
    "is_allowed": "false",
    "reason": "Insufficient privileges"
  }
}
```

### Example 5: Default Values

**CSV File** (`known_errors.csv`):
```csv
ErrorCode,Description
ERR_001,Network timeout
ERR_002,Invalid credentials
ERR_003,Resource exhausted
```

**Configuration**:
```yaml
- type: lookup
  location_path: "/etc/edgedelta/known_errors.csv"
  reload_period: 10m
  match_mode: exact
  key_fields:
    - event_field: 'attributes["error_code"]'
      lookup_field: "ErrorCode"
  out_fields:
    - event_field: 'attributes["description"]'
      lookup_field: "Description"
      default_value: "Unknown error code"
    - event_field: 'attributes["needs_investigation"]'
      lookup_field: "NeedInvestigation"  # Column doesn't exist
      default_value: "true"
```

**What it does**: Uses default values when no match is found or when lookup column doesn't exist.

**Input Log** (unknown error code):
```json
{
  "attributes": {
    "error_code": "ERR_999"
  }
}
```

**Output Log**:
```json
{
  "attributes": {
    "error_code": "ERR_999",
    "description": "Unknown error code",
    "needs_investigation": "true"
  }
}
```

### Example 6: Multiple Matches with append_mode

**CSV File** (`tags.csv`):
```csv
Keyword,Tag
payment,finance
payment,critical
database,infrastructure
database,backend
error,alert
```

**Configuration**:
```yaml
- type: lookup
  location_path: "/etc/edgedelta/tags.csv"
  reload_period: 5m
  match_mode: contain
  match_option: all
  ignore_case: true
  key_fields:
    - event_field: 'body'
      lookup_field: "Keyword"
  out_fields:
    - event_field: 'attributes["tags"]'
      lookup_field: "Tag"
      append_mode: true
```

**What it does**: Finds all keywords that appear in the log body and concatenates matching tags with commas.

**Input Log**:
```json
{
  "body": "Payment processing error in database"
}
```

**Output Log** (with append_mode):
```json
{
  "body": "Payment processing error in database",
  "attributes": {
    "tags": "finance,critical,infrastructure,backend,alert"
  }
}
```

**Without append_mode**, the output would be an array:
```json
{
  "attributes": {
    "tags": ["finance", "critical", "infrastructure", "backend", "alert"]
  }
}
```

### Example 7: Prefix Match for Hierarchical IDs

**CSV File** (`namespaces.csv`):
```csv
Prefix,Team,SLA
com.example.api,Platform,Tier1
com.example.web,Frontend,Tier2
com.example.data,DataEng,Tier1
com.example.ml,MLOps,Tier2
```

**Configuration**:
```yaml
- type: lookup
  location_path: "/etc/edgedelta/namespaces.csv"
  reload_period: 15m
  match_mode: prefix
  match_option: first
  key_fields:
    - event_field: 'attributes["service_name"]'
      lookup_field: "Prefix"
  out_fields:
    - event_field: 'attributes["team"]'
      lookup_field: "Team"
    - event_field: 'attributes["sla_tier"]'
      lookup_field: "SLA"
```

**What it does**: Matches service names by prefix to assign team ownership and SLA tier.

**Input Log**:
```json
{
  "attributes": {
    "service_name": "com.example.api.auth.v1"
  }
}
```

**Output Log**:
```json
{
  "attributes": {
    "service_name": "com.example.api.auth.v1",
    "team": "Platform",
    "sla_tier": "Tier1"
  }
}
```

### Example 8: IP Address to Location Mapping

**CSV File** (`ip_locations.csv`):
```csv
IPPrefix,Country,City,ISP
192.168,US,San Francisco,Internal
10.0,US,New York,Internal
203.0.113,GB,London,ExampleISP
198.51.100,DE,Berlin,EuroNet
```

**Configuration**:
```yaml
- type: lookup
  location_path: "/etc/edgedelta/ip_locations.csv"
  reload_period: 30m
  match_mode: prefix
  match_option: first
  key_fields:
    - event_field: 'attributes["client_ip"]'
      lookup_field: "IPPrefix"
  out_fields:
    - event_field: 'attributes["geo_country"]'
      lookup_field: "Country"
    - event_field: 'attributes["geo_city"]'
      lookup_field: "City"
    - event_field: 'attributes["isp"]'
      lookup_field: "ISP"
      default_value: "Unknown"
```

**What it does**: Enriches logs with geographic information based on IP address prefix.

**Input Log**:
```json
{
  "attributes": {
    "client_ip": "203.0.113.45"
  }
}
```

**Output Log**:
```json
{
  "attributes": {
    "client_ip": "203.0.113.45",
    "geo_country": "GB",
    "geo_city": "London",
    "isp": "ExampleISP"
  }
}
```

### Example 9: OTTL Expression Syntax

**CSV File** (`http_status.csv`):
```csv
StatusCode,Category,Severity
200,Success,Info
404,ClientError,Warning
500,ServerError,Critical
503,ServerError,Critical
```

**Configuration**:
```yaml
- type: lookup
  location_path: "/etc/edgedelta/http_status.csv"
  reload_period: 5m
  match_mode: exact
  key_fields:
    - event_field: 'attributes["http.status_code"]'
      lookup_field: "StatusCode"
  out_fields:
    - event_field: 'attributes["http.category"]'
      lookup_field: "Category"
    - event_field: 'attributes["severity"]'
      lookup_field: "Severity"
```

**What it does**: Uses OTTL syntax for field paths (when processor is in OTTL context).

**Note**: The processor automatically uses OTTL or CEL expressions based on the pipeline context. In a sequence processor using OTTL, use OTTL syntax. Otherwise, use CEL syntax with `item["attributes"]["field"]`.

### Example 10: Reload Period for Dynamic Updates

**Configuration**:
```yaml
- type: lookup
  location_path: "/etc/edgedelta/threat_intel.csv"
  reload_period: 1m  # Reload every minute for fresh threat data
  match_mode: exact
  ignore_case: true
  key_fields:
    - event_field: 'attributes["source_ip"]'
      lookup_field: "MaliciousIP"
  out_fields:
    - event_field: 'attributes["threat_level"]'
      lookup_field: "ThreatLevel"
      default_value: "none"
    - event_field: 'attributes["threat_type"]'
      lookup_field: "ThreatType"
      default_value: "unknown"
```

**What it does**: Reloads the threat intelligence CSV file every minute to get the latest malicious IP data without restarting the agent.

## Validation Rules

1. **Required Fields**: Must specify `type`, `location_path`, `key_fields`, `out_fields`
2. **Non-Empty Arrays**: `key_fields` and `out_fields` must each contain at least one entry
3. **Key Field Requirements**: Each key field must have both `event_field` and `lookup_field` defined
4. **Out Field Requirements**: Each out field must have both `event_field` and `lookup_field` defined
5. **Valid Match Mode**: `match_mode` must be one of: `exact`, `regex`, `contain`, `prefix`, `suffix`
6. **Valid Match Option**: `match_option` (for contain/prefix/suffix) must be `first` or `all`
7. **Valid Regex Option**: `regex_option` (for regex mode) must be `first` or `all`
8. **Ignore Case Restriction**: `ignore_case` is **not supported** with `regex` match mode (use `(?i)` in patterns)
9. **Key Field Restrictions**: `default_value` and `append_mode` are **not supported** for `key_fields`
10. **Protected Fields**: Cannot write to `body` or `resource` in `out_fields`
11. **Valid Expression**: `event_field` must be valid CEL or OTTL expression
12. **Field Path Syntax**: `event_field` in `out_fields` must have proper path syntax
13. **Reload Period**: Must be >= 0s (0s means no automatic reload)
14. **File Path**: `location_path` must be accessible by the EdgeDelta agent

## Common Pitfalls

### 1. Missing location_path

**Problem**: Not specifying the CSV file path causes the processor to fail.

**Wrong**:
```yaml
- type: lookup
  key_fields:
    - event_field: 'attributes["code"]'
      lookup_field: "Code"
```

**Correct**:
```yaml
- type: lookup
  location_path: "/etc/edgedelta/codes.csv"
  key_fields:
    - event_field: 'attributes["code"]'
      lookup_field: "Code"
```

### 2. Invalid match_mode

**Problem**: Using unsupported match mode values.

**Wrong**:
```yaml
match_mode: fuzzy  # Not supported
```

**Correct**:
```yaml
match_mode: exact  # or regex, contain, prefix, suffix
```

### 3. Key Field Path Errors (CEL vs OTTL)

**Problem**: Using wrong expression syntax for the context.

**Wrong (CEL context)**:
```yaml
key_fields:
  - event_field: 'attributes["status"]'  # Missing item prefix
    lookup_field: "Status"
```

**Correct (CEL context)**:
```yaml
key_fields:
  - event_field: 'item["attributes"]["status"]'
    lookup_field: "Status"
```

**Correct (OTTL context)**:
```yaml
key_fields:
  - event_field: 'attributes["status"]'
    lookup_field: "Status"
```

### 4. CSV Format Issues

**Problem**: CSV file doesn't match expected format or column names are wrong.

**Wrong CSV** (no header):
```csv
1001,Invalid input,Warning
1002,Connection failed,Critical
```

**Correct CSV** (with header row):
```csv
ErrorCode,Message,Severity
1001,Invalid input,Warning
1002,Connection failed,Critical
```

**Column Name Mismatch**:
```yaml
# CSV has "ErrorCode" but config uses "Code"
lookup_field: "Code"  # Won't match!
```

**Correct**:
```yaml
lookup_field: "ErrorCode"  # Matches CSV header exactly
```

### 5. Reload Period Too Short

**Problem**: Setting very short reload periods can cause performance issues.

**Wrong**:
```yaml
reload_period: 1s  # Too frequent, causes unnecessary I/O
```

**Correct**:
```yaml
reload_period: 5m  # Reasonable interval for most use cases
# or
reload_period: 0s  # No automatic reload if table is static
```

### 6. Regex Performance Issues

**Problem**: Using `regex_option: all` with complex patterns on large tables.

**Slow**:
```yaml
match_mode: regex
regex_option: all  # Evaluates ALL patterns even after match
```

**Faster**:
```yaml
match_mode: regex
regex_option: first  # Stops at first match
```

**Best (if possible)**:
```yaml
match_mode: exact  # Fastest option when appropriate
```

### 7. Case Sensitivity Confusion

**Problem**: Expecting case-insensitive matching without enabling it.

**Wrong**:
```yaml
match_mode: exact
# ignore_case not specified (defaults to false)
```

With input `"ERROR"` and lookup table containing `"error"`, no match occurs.

**Correct**:
```yaml
match_mode: exact
ignore_case: true
```

### 8. Using ignore_case with Regex

**Problem**: Trying to use `ignore_case: true` with regex mode.

**Wrong**:
```yaml
match_mode: regex
ignore_case: true  # Not supported, causes validation error
```

**Correct**:
```yaml
match_mode: regex
# Use (?i) flag in regex patterns for case-insensitive matching
```

**CSV File**:
```csv
Pattern,Category
(?i)^error.*,Error
(?i)^warn.*,Warning
```

### 9. Missing Key or Out Fields

**Problem**: Empty `key_fields` or `out_fields` arrays.

**Wrong**:
```yaml
- type: lookup
  location_path: "/path/to/file.csv"
  key_fields: []
  out_fields: []
```

**Correct**:
```yaml
- type: lookup
  location_path: "/path/to/file.csv"
  key_fields:
    - event_field: 'attributes["id"]'
      lookup_field: "ID"
  out_fields:
    - event_field: 'attributes["name"]'
      lookup_field: "Name"
```

### 10. Protected Field Overwrite

**Problem**: Attempting to write to protected fields like `body` or `resource`.

**Wrong**:
```yaml
out_fields:
  - event_field: 'body'  # Protected field
    lookup_field: "NewBody"
```

**Correct**:
```yaml
out_fields:
  - event_field: 'attributes["enriched_body"]'
    lookup_field: "NewBody"
```

## Best Practices

1. **CSV Organization**: Keep CSV files organized, versioned, and documented with clear column names
2. **File Permissions**: Ensure EdgeDelta agent has read access to the CSV file location
3. **Reload Period Selection**: Choose reload periods based on how frequently the lookup table changes
   - Static data: `0s` (no reload)
   - Hourly updates: `5m` or `10m`
   - Real-time threat feeds: `1m` or `2m`
4. **Match Mode Selection**: Use the simplest match mode that meets your requirements
   - Prefer `exact` over `contain`
   - Prefer `prefix`/`suffix` over `regex` when possible
   - Use `regex` only when pattern matching is truly needed
5. **Composite Keys**: Use multiple key fields for precise matching when single fields aren't unique enough
6. **Default Values**: Provide meaningful default values for optional enrichment fields
7. **Performance Testing**: Test with production-sized lookup tables to verify performance
8. **CSV Size Limits**: Keep lookup tables under 10,000 rows for best performance
   - For larger datasets, consider using a database integration or filtering CSV data
9. **Column Name Consistency**: Use consistent naming conventions between CSV columns and field names
10. **Expression Testing**: Test CEL/OTTL expressions independently before using in lookup processor
11. **Multiple Out Fields**: Enrich with multiple fields in a single lookup for efficiency
12. **Monitoring**: Monitor lookup processor metrics to detect missing files or reload failures
13. **CSV Validation**: Validate CSV format (proper headers, no malformed rows) before deployment
14. **Path Management**: Use absolute paths for `location_path` to avoid ambiguity
15. **Append Mode Usage**: Use `append_mode: true` for comma-separated string output instead of arrays when appropriate

## Performance Considerations

### CSV Size Impact

- **Small (< 100 rows)**: Negligible performance impact
- **Medium (100-1,000 rows)**: Minimal impact with proper match mode selection
- **Large (1,000-10,000 rows)**: Noticeable with `regex` or `all` options
- **Very Large (> 10,000 rows)**: Consider alternative solutions (database, filtering, sharding)

### Reload Frequency

- Each reload reads and parses the entire CSV file
- Balance freshness requirements vs I/O overhead
- For static data, set `reload_period: 0s` to disable reloading

### Match Mode Performance (fastest to slowest)

1. **exact**: O(1) hash lookup - best performance
2. **prefix/suffix**: O(n) string comparison - fast for small tables
3. **contain**: O(n) substring search - moderate performance
4. **regex**: O(n * m) pattern matching - slowest, especially with complex patterns

### Optimization Tips

- Order CSV rows by match likelihood (most common matches first)
- Use `first` option instead of `all` when only one match is needed
- Minimize the number of key fields (composite keys are slower)
- Keep regex patterns simple and specific
- Use `ignore_case` sparingly (adds overhead)
- Pre-filter logs with `condition` parameter to reduce lookups

## CSV Format Requirements

### File Structure

1. **Header Row**: First row must contain column names
2. **Data Rows**: Subsequent rows contain data values
3. **Encoding**: UTF-8 encoding recommended
4. **Line Endings**: Unix (LF) or Windows (CRLF) line endings supported
5. **Delimiter**: Comma (`,`) is the standard delimiter
6. **Quotes**: Use double quotes for values containing commas or newlines

### Example CSV Format

```csv
ErrorCode,ErrorMessage,Severity,ActionRequired
1001,Invalid input,Warning,false
1002,Connection failed,Critical,true
1003,"Timeout, retry later",Error,true
1004,Not found,Warning,false
```

### Special Cases

**Values with Commas**:
```csv
Field1,Field2
"value with, comma",normal_value
```

**Values with Quotes**:
```csv
Field1,Field2
"value with ""quotes""",normal_value
```

**Empty Values**:
```csv
Field1,Field2,Field3
value1,,value3
```

### CSV Best Practices

- Keep column names simple (alphanumeric, underscores)
- Avoid special characters in column names
- Use consistent data types within columns
- Remove trailing whitespace from values
- Test CSV parsing with sample data before production deployment

## Related Processors

- **ottl_transform**: Transform fields before lookup matching
- **ottl_filter**: Filter items before lookup processing
- **generic_mask**: Mask sensitive data in lookup keys or results
- **extract**: Extract values from unstructured data before lookup
- **parse_json**: Parse JSON fields before using in lookups

## Cross-References

- **edgedelta-pipelines skill**: Template examples using lookup processor
- **OTTL Reference**: https://docs.edgedelta.com/ottl-statements/
- **CEL Reference**: Expression syntax for key and out fields
- **Best Practices**: `.claude/skills/edgedelta-pipelines/assets/references/best-practices.md`

## Documentation References

- **EdgeDelta Docs**: "Lookup Processor" and "Data Enrichment"
- **Example Pages**:
  - https://docs.edgedelta.com/lookup-processor/
  - https://docs.edgedelta.com/data-enrichment/
- **CSV Format Guide**: https://datatracker.ietf.org/doc/html/rfc4180
- **OTTL Guide**: https://docs.edgedelta.com/ottl-statements/
- **Source Code**: `EdgeDelta source code repository/internalv3/processors/lookup/processor.go`

## Notes

- The processor loads the entire CSV file into memory on startup and reload
- Changes to the CSV file are only picked up after the reload period elapses
- If the CSV file is deleted or becomes inaccessible, the processor continues using the last loaded version
- Regex patterns in the CSV file are compiled once during load (cached for performance)
- Multiple matches (with `all` option) return values in the order they appear in the CSV
- The processor supports both CEL and OTTL expression types depending on the pipeline context
- Lookup operations are thread-safe and can process multiple items concurrently
- Failed lookups (no match) do not generate errors unless there's a configuration issue
- The processor is optimized for read-heavy workloads (many lookups, infrequent reloads)
