# OTTL & EDX Extensions Quick Reference

**Purpose**: Quick lookup for all 124 OTTL functions (101 standard + 23 EDX extensions)
**Source**: OpenTelemetry Collector OTTL + EdgeDelta EDX Extensions
**Last Updated**: 2025-10-27

## How to Use This Index

1. **Find Your Function**: Browse by category or search by name
2. **Copy Quick Snippet**: Use the minimal valid example
3. **Check Syntax**: Review parameters and return types
4. **Read Full Reference**: Link to detailed documentation if available

## Function Categories

- [OTTL Editors](#ottl-editors) (14 functions)
- [OTTL Converters](#ottl-converters) (87 functions)
- [Type Checkers](#type-checkers) (8 functions)
- [Hashing Functions](#hashing-functions) (6 functions)
- [Time Functions](#time-functions) (20 functions)
- [XML Functions](#xml-functions) (6 functions)
- [EDX Extensions](#edx-extensions) (23 functions)
  - [EDX Converters](#edx-converters) (15 functions)
  - [EDX Editors](#edx-editors) (8 functions)

---

## OTTL Editors

Functions that modify data in place.

### append
**Purpose**: Appends values to a list or creates a new list
**Signature**: `append(target, value) -> void`
**Full Reference**: [append.md](references/standard/append.md)

**Quick Copy**:
```yaml
- set(attributes["tags"], append(attributes["tags"], "new-tag"))
```

**Common Use Cases**:
- Adding tags to existing tag lists
- Accumulating values in arrays
- Building lists dynamically

**Parameters**:
- `target`: List to append to (created if doesn't exist)
- `value`: Value to append

---

### delete_key
**Purpose**: Deletes a specific key from a map
**Signature**: `delete_key(target, key) -> void`
**Full Reference**: [delete_key.md](references/standard/delete_key.md)

**Quick Copy**:
```yaml
- delete_key(attributes, "sensitive_field")
```

**Common Use Cases**:
- Removing sensitive data
- Cleaning up unwanted fields
- Normalizing data structures

**Parameters**:
- `target`: Map to delete from
- `key`: Key to delete

---

### delete_matching_keys
**Purpose**: Deletes all keys matching a pattern from a map
**Signature**: `delete_matching_keys(target, pattern) -> void`
**Full Reference**: [delete_matching_keys.md](references/standard/delete_matching_keys.md)

**Quick Copy**:
```yaml
- delete_matching_keys(attributes, "^temp_.*")
```

**Common Use Cases**:
- Bulk removal of temporary fields
- Cleaning debug attributes
- Removing fields by pattern

**Parameters**:
- `target`: Map to delete from
- `pattern`: Regex pattern to match keys

---

### flatten
**Purpose**: Flattens nested maps into a single level with dot notation
**Signature**: `flatten(target, Optional[prefix], Optional[depth]) -> void`
**Full Reference**: [flatten.md](references/standard/flatten.md)

**Quick Copy**:
```yaml
- set(attributes, flatten(attributes, "", 2))
```

**Common Use Cases**:
- Converting nested JSON to flat structure
- Creating searchable field names
- Simplifying complex objects

**Parameters**:
- `target`: Map to flatten
- `prefix`: Optional prefix for flattened keys
- `depth`: Optional max depth to flatten

---

### keep_keys
**Purpose**: Keeps only specified keys, removes all others
**Signature**: `keep_keys(target, keys[]) -> void`
**Full Reference**: [keep_keys.md](references/standard/keep_keys.md)

**Quick Copy**:
```yaml
- keep_keys(attributes, ["service.name", "host.name", "level"])
```

**Common Use Cases**:
- Allowlisting important fields
- Reducing attribute cardinality
- Creating minimal data views

**Parameters**:
- `target`: Map to filter
- `keys`: List of keys to keep

---

### keep_matching_keys
**Purpose**: Keeps only keys matching a pattern, removes all others
**Signature**: `keep_matching_keys(target, pattern) -> void`
**Full Reference**: [keep_matching_keys.md](references/standard/keep_matching_keys.md)

**Quick Copy**:
```yaml
- keep_matching_keys(attributes, "^(service|host|k8s)\\.")
```

**Common Use Cases**:
- Filtering by namespace prefix
- Keeping only infrastructure attributes
- Pattern-based allowlisting

**Parameters**:
- `target`: Map to filter
- `pattern`: Regex pattern to match keys

---

### limit
**Purpose**: Limits the number of items in a list
**Signature**: `limit(target, limit, Optional[priority_keys[]]) -> void`
**Full Reference**: [limit.md](references/standard/limit.md)

**Quick Copy**:
```yaml
- limit(attributes, 100, ["service.name", "host.name"])
```

**Common Use Cases**:
- Preventing attribute explosion
- Controlling cardinality
- Prioritizing important fields

**Parameters**:
- `target`: Map to limit
- `limit`: Maximum number of items
- `priority_keys`: Optional keys to prioritize

---

### merge_maps
**Purpose**: Merges two maps together
**Signature**: `merge_maps(target, source, strategy) -> void`
**Full Reference**: [merge_maps.md](references/standard/merge_maps.md)

**Quick Copy**:
```yaml
- merge_maps(attributes, resource.attributes, "upsert")
```

**Common Use Cases**:
- Combining resource and span attributes
- Merging configuration with defaults
- Enriching data from multiple sources

**Parameters**:
- `target`: Destination map
- `source`: Source map to merge
- `strategy`: "insert", "update", or "upsert"

---

### replace_match
**Purpose**: Replaces the first substring matching a pattern
**Signature**: `replace_match(target, pattern, replacement, Optional[function]) -> void`
**Full Reference**: [replace_match.md](references/standard/replace_match.md)

**Quick Copy**:
```yaml
- replace_match(body, "password=\\S+", "password=***")
```

**Common Use Cases**:
- Redacting sensitive data
- Normalizing log messages
- Cleaning up formatting

**Parameters**:
- `target`: String to modify
- `pattern`: Regex pattern to match
- `replacement`: Replacement string
- `function`: Optional replacement function

---

### replace_all_matches
**Purpose**: Replaces all substrings matching a pattern
**Signature**: `replace_all_matches(target, pattern, replacement, Optional[function]) -> void`
**Full Reference**: [replace_all_matches.md](references/standard/replace_all_matches.md)

**Quick Copy**:
```yaml
- replace_all_matches(body, "\\b\\d{16}\\b", "XXXX-XXXX-XXXX-XXXX")
```

**Common Use Cases**:
- Redacting credit card numbers
- Masking multiple occurrences
- Global search and replace

**Parameters**:
- `target`: String to modify
- `pattern`: Regex pattern to match
- `replacement`: Replacement string
- `function`: Optional replacement function

---

### replace_pattern
**Purpose**: Replaces the first occurrence of a literal string
**Signature**: `replace_pattern(target, pattern, replacement, Optional[function]) -> void`
**Full Reference**: [replace_pattern.md](references/standard/replace_pattern.md)

**Quick Copy**:
```yaml
- replace_pattern(body, "ERROR", "WARN")
```

**Common Use Cases**:
- Simple string replacement
- Normalizing severity levels
- Correcting typos

**Parameters**:
- `target`: String to modify
- `pattern`: Literal string to find
- `replacement`: Replacement string
- `function`: Optional replacement function

---

### replace_all_patterns
**Purpose**: Replaces all occurrences of a literal string
**Signature**: `replace_all_patterns(target, pattern, replacement, Optional[function]) -> void`
**Full Reference**: [replace_all_patterns.md](references/standard/replace_all_patterns.md)

**Quick Copy**:
```yaml
- replace_all_patterns(body, "localhost", "prod-server-01")
```

**Common Use Cases**:
- Global literal replacement
- Hostname normalization
- Environment-specific substitution

**Parameters**:
- `target`: String to modify
- `pattern`: Literal string to find
- `replacement`: Replacement string
- `function`: Optional replacement function

---

### set
**Purpose**: Sets a value at a target path
**Signature**: `set(target, value) -> void`
**Full Reference**: [set.md](references/standard/set.md)

**Quick Copy**:
```yaml
- set(attributes["http.status_code"], 200)
- set(severity_text, "INFO")
```

**Common Use Cases**:
- Setting attribute values
- Initializing fields
- Overwriting existing data

**Parameters**:
- `target`: Path to set
- `value`: Value to set

---

### truncate_all
**Purpose**: Truncates all string values in a map
**Signature**: `truncate_all(target, limit) -> void`
**Full Reference**: [truncate_all.md](references/standard/truncate_all.md)

**Quick Copy**:
```yaml
- truncate_all(attributes, 256)
```

**Common Use Cases**:
- Preventing oversized attributes
- Controlling data volume
- Enforcing size limits

**Parameters**:
- `target`: Map with string values
- `limit`: Maximum length for strings

---

## OTTL Converters

Functions that transform data and return new values.

### base64decode
**Purpose**: Decodes a base64 string
**Signature**: `base64decode(value) -> string`
**Full Reference**: [base64decode.md](references/standard/base64decode.md)

**Quick Copy**:
```yaml
- set(attributes["decoded"], base64decode(attributes["encoded_data"]))
```

**Common Use Cases**:
- Decoding encoded payloads
- Processing base64 tokens
- Extracting embedded data

**Parameters**:
- `value`: Base64 encoded string

---

### Concat
**Purpose**: Concatenates strings with a delimiter
**Signature**: `Concat(values[], delimiter) -> string`
**Full Reference**: [concat.md](references/standard/concat.md)

**Quick Copy**:
```yaml
- set(attributes["full_name"], Concat([attributes["first"], attributes["last"]], " "))
```

**Common Use Cases**:
- Building composite fields
- Creating display names
- Joining path components

**Parameters**:
- `values`: List of values to concatenate
- `delimiter`: String to insert between values

---

### contains_value
**Purpose**: Checks if a map contains a specific value
**Signature**: `contains_value(target, value) -> bool`
**Full Reference**: [contains_value.md](references/standard/contains_value.md)

**Quick Copy**:
```yaml
- set(attributes["has_error"], contains_value(attributes, "ERROR"))
```

**Common Use Cases**:
- Checking for specific values
- Validating data presence
- Conditional logic

**Parameters**:
- `target`: Map to search
- `value`: Value to find

---

### convert_case
**Purpose**: Converts string case (upper, lower, title, snake, camel)
**Signature**: `convert_case(value, case_type) -> string`
**Full Reference**: [convert_case.md](references/standard/convert_case.md)

**Quick Copy**:
```yaml
- set(attributes["service"], convert_case(attributes["service"], "lower"))
```

**Common Use Cases**:
- Normalizing service names
- Case-insensitive matching
- Formatting output

**Parameters**:
- `value`: String to convert
- `case_type`: "upper", "lower", "title", "snake", "camel"

---

### day
**Purpose**: Extracts day of month from a time
**Signature**: `day(time) -> int`
**Full Reference**: [day.md](references/standard/day.md)

**Quick Copy**:
```yaml
- set(attributes["day"], day(time_unix_nano))
```

**Common Use Cases**:
- Extracting day for aggregation
- Date-based routing
- Daily statistics

**Parameters**:
- `time`: Time value (Unix nano or Time object)

---

### decode
**Purpose**: Decodes a URL-encoded string
**Signature**: `decode(value) -> string`
**Full Reference**: [decode.md](references/standard/decode.md)

**Quick Copy**:
```yaml
- set(attributes["url_decoded"], decode(attributes["url_param"]))
```

**Common Use Cases**:
- Decoding URL parameters
- Processing query strings
- Cleaning encoded data

**Parameters**:
- `value`: URL-encoded string

---

### double
**Purpose**: Converts a value to double/float
**Signature**: `double(value) -> float64`
**Full Reference**: [double.md](references/standard/double.md)

**Quick Copy**:
```yaml
- set(attributes["latency_sec"], double(attributes["latency_ms"]) / 1000.0)
```

**Common Use Cases**:
- Type conversion for math
- Parsing numeric strings
- Unit conversion

**Parameters**:
- `value`: Value to convert to double

---

### duration
**Purpose**: Parses a duration string (e.g., "5m", "1h30m")
**Signature**: `duration(value) -> Duration`
**Full Reference**: [duration.md](references/standard/duration.md)

**Quick Copy**:
```yaml
- set(attributes["timeout"], duration("30s"))
```

**Common Use Cases**:
- Parsing timeout values
- Converting duration strings
- Time arithmetic

**Parameters**:
- `value`: Duration string (Go duration format)

---

### extract_grok_patterns
**Purpose**: Extracts fields using Grok patterns (includes error handling)
**Signature**: `extract_grok_patterns(text, pattern, namedCapturesOnly) -> map`
**Full Reference**: [extract_grok_patterns.md](references/standard/extract_grok_patterns.md)

**Quick Copy**:
```yaml
- set(attributes, extract_grok_patterns(body, "%{COMMONAPACHELOG}", true))
```

**Common Use Cases**:
- Parsing Apache/Nginx logs
- Extracting structured data from text
- Using predefined patterns

**Parameters**:
- `text`: String to parse
- `pattern`: Grok pattern
- `namedCapturesOnly`: Boolean to only return named captures

---

### extract_patterns
**Purpose**: Extracts fields using regex with named groups
**Signature**: `extract_patterns(text, pattern) -> map`
**Full Reference**: [extract_patterns.md](references/standard/extract_patterns.md)

**Quick Copy**:
```yaml
- set(attributes, extract_patterns(body, "level=(?P<level>\\w+) msg=(?P<message>.+)"))
```

**Common Use Cases**:
- Parsing custom log formats
- Extracting structured fields
- Creating attributes from regex

**Parameters**:
- `text`: String to parse
- `pattern`: Regex with named groups (?P<name>...)

---

### FNV
**Purpose**: Computes FNV-1a hash (32-bit)
**Signature**: `FNV(value) -> string`
**Full Reference**: [fnv.md](references/standard/fnv.md)

**Quick Copy**:
```yaml
- set(attributes["hash"], FNV(body))
```

**Common Use Cases**:
- Fast non-cryptographic hashing
- Creating unique IDs
- Deduplication keys

**Parameters**:
- `value`: String to hash

---

### format
**Purpose**: Formats a string using Go template syntax
**Signature**: `format(template, ...args) -> string`
**Full Reference**: [format.md](references/standard/format.md)

**Quick Copy**:
```yaml
- set(attributes["message"], format("User %s logged in from %s", attributes["user"], attributes["ip"]))
```

**Common Use Cases**:
- Building log messages
- Creating formatted output
- String interpolation

**Parameters**:
- `template`: Format string with %s, %d, etc.
- `args`: Values to insert

---

### formattime
**Purpose**: Formats a time value as a string
**Signature**: `formattime(time, format) -> string`
**Full Reference**: [formattime.md](references/standard/formattime.md)

**Quick Copy**:
```yaml
- set(attributes["timestamp"], formattime(time_unix_nano, "2006-01-02 15:04:05"))
```

**Common Use Cases**:
- Converting timestamps to strings
- Custom date formatting
- Human-readable timestamps

**Parameters**:
- `time`: Time value
- `format`: Go time layout string

---

### has_prefix
**Purpose**: Checks if a string starts with a prefix
**Signature**: `has_prefix(value, prefix) -> bool`
**Full Reference**: [has_prefix.md](references/standard/has_prefix.md)

**Quick Copy**:
```yaml
- set(attributes["is_error"], has_prefix(severity_text, "ERR"))
```

**Common Use Cases**:
- Filtering by prefix
- Namespace checking
- Conditional routing

**Parameters**:
- `value`: String to check
- `prefix`: Prefix to match

---

### has_suffix
**Purpose**: Checks if a string ends with a suffix
**Signature**: `has_suffix(value, suffix) -> bool`
**Full Reference**: [has_suffix.md](references/standard/has_suffix.md)

**Quick Copy**:
```yaml
- set(attributes["is_log_file"], has_suffix(attributes["filename"], ".log"))
```

**Common Use Cases**:
- File extension checking
- Suffix-based routing
- Pattern matching

**Parameters**:
- `value`: String to check
- `suffix`: Suffix to match

---

### hex
**Purpose**: Converts bytes to hexadecimal string
**Signature**: `hex(value) -> string`
**Full Reference**: [hex.md](references/standard/hex.md)

**Quick Copy**:
```yaml
- set(attributes["trace_id_hex"], hex(trace_id.bytes))
```

**Common Use Cases**:
- Converting trace/span IDs
- Displaying binary data
- Creating hex representations

**Parameters**:
- `value`: Bytes to convert

---

### hour
**Purpose**: Extracts hour from a time
**Signature**: `hour(time) -> int`
**Full Reference**: [hour.md](references/standard/hour.md)

**Quick Copy**:
```yaml
- set(attributes["hour"], hour(time_unix_nano))
```

**Common Use Cases**:
- Hour-based routing
- Time-of-day analysis
- Hourly aggregation

**Parameters**:
- `time`: Time value

---

### hours
**Purpose**: Converts duration to hours
**Signature**: `hours(duration) -> float64`
**Full Reference**: [hours.md](references/standard/hours.md)

**Quick Copy**:
```yaml
- set(attributes["duration_hours"], hours(duration("90m")))
```

**Common Use Cases**:
- Converting durations to hours
- Time-based calculations
- SLA tracking

**Parameters**:
- `duration`: Duration value

---

### index
**Purpose**: Returns the index of a substring
**Signature**: `index(value, substring) -> int`
**Full Reference**: [index.md](references/standard/index.md)

**Quick Copy**:
```yaml
- set(attributes["position"], index(body, "ERROR"))
```

**Common Use Cases**:
- Finding substring position
- String searching
- Conditional extraction

**Parameters**:
- `value`: String to search
- `substring`: Substring to find

---

### int
**Purpose**: Converts a value to integer
**Signature**: `int(value) -> int64`
**Full Reference**: [int.md](references/standard/int.md)

**Quick Copy**:
```yaml
- set(attributes["status_code"], int(attributes["status"]))
```

**Common Use Cases**:
- Type conversion
- Parsing numeric strings
- Mathematical operations

**Parameters**:
- `value`: Value to convert to int

---

### is_bool
**Purpose**: Checks if a value is a boolean
**Signature**: `is_bool(value) -> bool`
**Full Reference**: [is_bool.md](references/standard/is_bool.md)

**Quick Copy**:
```yaml
- set(attributes["is_bool_type"], is_bool(attributes["flag"]))
```

**Common Use Cases**:
- Type validation
- Schema checking
- Conditional logic

**Parameters**:
- `value`: Value to check

---

### is_double
**Purpose**: Checks if a value is a float/double
**Signature**: `is_double(value) -> bool`
**Full Reference**: [is_double.md](references/standard/is_double.md)

**Quick Copy**:
```yaml
- set(attributes["is_numeric"], is_double(attributes["value"]))
```

**Common Use Cases**:
- Type validation
- Numeric detection
- Data quality checks

**Parameters**:
- `value`: Value to check

---

### is_int
**Purpose**: Checks if a value is an integer
**Signature**: `is_int(value) -> bool`
**Full Reference**: [is_int.md](references/standard/is_int.md)

**Quick Copy**:
```yaml
- set(attributes["is_int_type"], is_int(attributes["count"]))
```

**Common Use Cases**:
- Type validation
- Integer detection
- Schema validation

**Parameters**:
- `value`: Value to check

---

### is_list
**Purpose**: Checks if a value is a list/array
**Signature**: `is_list(value) -> bool`
**Full Reference**: [is_list.md](references/standard/is_list.md)

**Quick Copy**:
```yaml
- set(attributes["is_array"], is_list(attributes["items"]))
```

**Common Use Cases**:
- Type checking
- Array validation
- Conditional processing

**Parameters**:
- `value`: Value to check

---

### is_map
**Purpose**: Checks if a value is a map/object
**Signature**: `is_map(value) -> bool`
**Full Reference**: [is_map.md](references/standard/is_map.md)

**Quick Copy**:
```yaml
- set(attributes["is_object"], is_map(attributes["metadata"]))
```

**Common Use Cases**:
- Type validation
- Object detection
- Structure validation

**Parameters**:
- `value`: Value to check

---

### is_match
**Purpose**: Checks if a string matches a regex pattern
**Signature**: `is_match(value, pattern) -> bool`
**Full Reference**: [is_match.md](references/standard/is_match.md)

**Quick Copy**:
```yaml
- set(attributes["is_email"], is_match(attributes["email"], "^[\\w.-]+@[\\w.-]+\\.[a-z]{2,}$"))
```

**Common Use Cases**:
- Pattern validation
- Filtering by regex
- Conditional routing

**Parameters**:
- `value`: String to check
- `pattern`: Regex pattern

---

### is_root_span
**Purpose**: Checks if the span is a root span (no parent)
**Signature**: `is_root_span() -> bool`
**Full Reference**: [is_root_span.md](references/standard/is_root_span.md)

**Quick Copy**:
```yaml
- set(attributes["is_root"], is_root_span())
```

**Common Use Cases**:
- Root span identification
- Trace entry point detection
- Sampling decisions

**Parameters**:
- None

---

### is_string
**Purpose**: Checks if a value is a string
**Signature**: `is_string(value) -> bool`
**Full Reference**: [is_string.md](references/standard/is_string.md)

**Quick Copy**:
```yaml
- set(attributes["is_text"], is_string(attributes["message"]))
```

**Common Use Cases**:
- Type validation
- String detection
- Schema checking

**Parameters**:
- `value`: Value to check

---

### keys
**Purpose**: Returns all keys from a map as a list
**Signature**: `keys(target) -> []string`
**Full Reference**: [keys.md](references/standard/keys.md)

**Quick Copy**:
```yaml
- set(attributes["field_names"], keys(attributes))
```

**Common Use Cases**:
- Extracting field names
- Schema discovery
- Dynamic processing

**Parameters**:
- `target`: Map to extract keys from

---

### len
**Purpose**: Returns the length of a string, list, or map
**Signature**: `len(value) -> int`
**Full Reference**: [len.md](references/standard/len.md)

**Quick Copy**:
```yaml
- set(attributes["attr_count"], len(attributes))
- set(attributes["msg_length"], len(body))
```

**Common Use Cases**:
- Counting attributes
- String length validation
- Size-based filtering

**Parameters**:
- `value`: String, list, or map

---

### log
**Purpose**: Returns natural logarithm
**Signature**: `log(value) -> float64`
**Full Reference**: [log.md](references/standard/log.md)

**Quick Copy**:
```yaml
- set(attributes["log_value"], log(double(attributes["value"])))
```

**Common Use Cases**:
- Mathematical transformations
- Log-scale calculations
- Statistical analysis

**Parameters**:
- `value`: Numeric value

---

### luhn_valid
**Purpose**: Validates credit card numbers using Luhn algorithm
**Signature**: `luhn_valid(value) -> bool`
**Full Reference**: [luhn_valid.md](references/standard/luhn_valid.md)

**Quick Copy**:
```yaml
- set(attributes["valid_cc"], luhn_valid(attributes["card_number"]))
```

**Common Use Cases**:
- Credit card validation
- Detecting card numbers for redaction
- PCI compliance

**Parameters**:
- `value`: String of digits

---

### MD5
**Purpose**: Computes MD5 hash
**Signature**: `MD5(value) -> string`
**Full Reference**: [md5.md](references/standard/md5.md)

**Quick Copy**:
```yaml
- set(attributes["content_hash"], MD5(body))
```

**Common Use Cases**:
- Content hashing
- Deduplication
- Checksum generation

**Parameters**:
- `value`: String to hash

---

### merge_maps
**Purpose**: Merges two maps and returns the result
**Signature**: `merge_maps(map1, map2, strategy) -> map`
**Full Reference**: [merge_maps.md](references/standard/merge_maps.md)

**Quick Copy**:
```yaml
- set(attributes, merge_maps(attributes, resource.attributes, "upsert"))
```

**Common Use Cases**:
- Combining attributes
- Merging configurations
- Data enrichment

**Parameters**:
- `map1`: First map
- `map2`: Second map
- `strategy`: "insert", "update", or "upsert"

---

### microseconds
**Purpose**: Converts duration to microseconds
**Signature**: `microseconds(duration) -> int64`
**Full Reference**: [microseconds.md](references/standard/microseconds.md)

**Quick Copy**:
```yaml
- set(attributes["duration_us"], microseconds(duration("5ms")))
```

**Common Use Cases**:
- Duration conversion
- Timestamp precision
- Performance metrics

**Parameters**:
- `duration`: Duration value

---

### milliseconds
**Purpose**: Converts duration to milliseconds
**Signature**: `milliseconds(duration) -> int64`
**Full Reference**: [milliseconds.md](references/standard/milliseconds.md)

**Quick Copy**:
```yaml
- set(attributes["duration_ms"], milliseconds(duration("5s")))
```

**Common Use Cases**:
- Duration conversion
- Latency metrics
- Timeout calculations

**Parameters**:
- `duration`: Duration value

---

### minute
**Purpose**: Extracts minute from a time
**Signature**: `minute(time) -> int`
**Full Reference**: [minute.md](references/standard/minute.md)

**Quick Copy**:
```yaml
- set(attributes["minute"], minute(time_unix_nano))
```

**Common Use Cases**:
- Time extraction
- Minute-based aggregation
- Time-of-day routing

**Parameters**:
- `time`: Time value

---

### minutes
**Purpose**: Converts duration to minutes
**Signature**: `minutes(duration) -> float64`
**Full Reference**: [minutes.md](references/standard/minutes.md)

**Quick Copy**:
```yaml
- set(attributes["duration_min"], minutes(duration("2h")))
```

**Common Use Cases**:
- Duration conversion
- Time-based calculations
- SLA tracking

**Parameters**:
- `duration`: Duration value

---

### month
**Purpose**: Extracts month from a time
**Signature**: `month(time) -> int`
**Full Reference**: [month.md](references/standard/month.md)

**Quick Copy**:
```yaml
- set(attributes["month"], month(time_unix_nano))
```

**Common Use Cases**:
- Monthly aggregation
- Date-based routing
- Seasonal analysis

**Parameters**:
- `time`: Time value

---

### MurmurHash3
**Purpose**: Computes MurmurHash3 (32-bit)
**Signature**: `MurmurHash3(value) -> string`
**Full Reference**: [murmur3_hash.md](references/standard/murmur3_hash.md)

**Quick Copy**:
```yaml
- set(attributes["hash"], MurmurHash3(Concat([attributes["user"], attributes["session"]], ":")))
```

**Common Use Cases**:
- Fast hashing for partitioning
- Consistent hashing
- Deduplication keys

**Parameters**:
- `value`: String to hash

---

### MurmurHash3_128
**Purpose**: Computes MurmurHash3 (128-bit)
**Signature**: `MurmurHash3_128(value) -> string`
**Full Reference**: [murmur3_hash128.md](references/standard/murmur3_hash128.md)

**Quick Copy**:
```yaml
- set(attributes["hash128"], MurmurHash3_128(body))
```

**Common Use Cases**:
- High-entropy hashing
- UUID generation
- Distributed systems

**Parameters**:
- `value`: String to hash

---

### nanosecond
**Purpose**: Extracts nanoseconds from a time
**Signature**: `nanosecond(time) -> int`
**Full Reference**: [nanosecond.md](references/standard/nanosecond.md)

**Quick Copy**:
```yaml
- set(attributes["nanos"], nanosecond(time_unix_nano))
```

**Common Use Cases**:
- High-precision timing
- Sub-second extraction
- Performance metrics

**Parameters**:
- `time`: Time value

---

### nanoseconds
**Purpose**: Converts duration to nanoseconds
**Signature**: `nanoseconds(duration) -> int64`
**Full Reference**: [nanoseconds.md](references/standard/nanoseconds.md)

**Quick Copy**:
```yaml
- set(attributes["duration_ns"], nanoseconds(duration("1ms")))
```

**Common Use Cases**:
- Duration conversion
- High-precision timing
- Performance metrics

**Parameters**:
- `duration`: Duration value

---

### now
**Purpose**: Returns current time
**Signature**: `now() -> Time`
**Full Reference**: [now.md](references/standard/now.md)

**Quick Copy**:
```yaml
- set(attributes["processed_at"], now())
```

**Common Use Cases**:
- Timestamping operations
- Calculating age
- Time-based decisions

**Parameters**:
- None

---

### parse_csv
**Purpose**: Parses CSV line into a list
**Signature**: `parse_csv(value, Optional[delimiter], Optional[header_row_index], Optional[mode]) -> []string or []map`
**Full Reference**: [parse_csv.md](references/standard/parse_csv.md)

**Quick Copy**:
```yaml
- set(attributes["fields"], parse_csv(body, ",", 0, "lazy"))
```

**Common Use Cases**:
- Parsing CSV logs
- Extracting structured data
- Batch processing

**Parameters**:
- `value`: CSV string
- `delimiter`: Field separator (default ",")
- `header_row_index`: Header row index (default -1)
- `mode`: "strict" or "lazy" (default "strict")

---

### parse_int
**Purpose**: Parses an integer from a string with base
**Signature**: `parse_int(value, base) -> int64`
**Full Reference**: [parse_int.md](references/standard/parse_int.md)

**Quick Copy**:
```yaml
- set(attributes["decimal"], parse_int("FF", 16))
```

**Common Use Cases**:
- Parsing hex values
- Converting number strings
- Base conversion

**Parameters**:
- `value`: String to parse
- `base`: Number base (2-36)

---

### parse_json
**Purpose**: Parses JSON string into a map or value
**Signature**: `parse_json(value) -> any`
**Full Reference**: [parse_json.md](references/standard/parse_json.md)

**Quick Copy**:
```yaml
- set(attributes, parse_json(body))
```

**Common Use Cases**:
- Parsing JSON logs
- Extracting structured data
- Converting JSON to attributes

**Parameters**:
- `value`: JSON string

---

### parse_key_value
**Purpose**: Parses key=value pairs from a string
**Signature**: `parse_key_value(value, Optional[delimiter], Optional[pair_delimiter]) -> map`
**Full Reference**: [parse_key_value.md](references/standard/parse_key_value.md)

**Quick Copy**:
```yaml
- set(attributes, parse_key_value(body, "=", " "))
```

**Common Use Cases**:
- Parsing logfmt logs
- Extracting key-value data
- Converting unstructured to structured

**Parameters**:
- `value`: String to parse
- `delimiter`: Key-value separator (default "=")
- `pair_delimiter`: Pair separator (default " ")

---

### parse_simplified_xml
**Purpose**: Parses simplified XML into a map
**Signature**: `parse_simplified_xml(value) -> map`
**Full Reference**: [parse_simplified_xml.md](references/standard/parse_simplified_xml.md)

**Quick Copy**:
```yaml
- set(attributes, parse_simplified_xml(body))
```

**Common Use Cases**:
- Parsing simple XML
- Converting XML to JSON-like structure
- Extracting XML data

**Parameters**:
- `value`: XML string

---

### parse_xml
**Purpose**: Parses XML with full structure preservation
**Signature**: `parse_xml(value) -> map`
**Full Reference**: [parse_xml.md](references/standard/parse_xml.md)

**Quick Copy**:
```yaml
- set(attributes["xml_data"], parse_xml(body))
```

**Common Use Cases**:
- Parsing complex XML
- Preserving XML structure
- XML to attribute conversion

**Parameters**:
- `value`: XML string

---

### profile_id
**Purpose**: Returns the profile ID from profiling data
**Signature**: `profile_id() -> ProfileID`
**Full Reference**: [profile_id.md](references/standard/profile_id.md)

**Quick Copy**:
```yaml
- set(attributes["profile_id"], profile_id().string)
```

**Common Use Cases**:
- Profiling data correlation
- Performance analysis
- Profile identification

**Parameters**:
- None

---

### second
**Purpose**: Extracts seconds from a time
**Signature**: `second(time) -> int`
**Full Reference**: [second.md](references/standard/second.md)

**Quick Copy**:
```yaml
- set(attributes["second"], second(time_unix_nano))
```

**Common Use Cases**:
- Time extraction
- Second-based routing
- Time alignment

**Parameters**:
- `time`: Time value

---

### seconds
**Purpose**: Converts duration to seconds
**Signature**: `seconds(duration) -> float64`
**Full Reference**: [seconds.md](references/standard/seconds.md)

**Quick Copy**:
```yaml
- set(attributes["duration_sec"], seconds(duration("5m")))
```

**Common Use Cases**:
- Duration conversion
- Time calculations
- Metric conversion

**Parameters**:
- `duration`: Duration value

---

### SHA1
**Purpose**: Computes SHA-1 hash
**Signature**: `SHA1(value) -> string`
**Full Reference**: [sha1.md](references/standard/sha1.md)

**Quick Copy**:
```yaml
- set(attributes["checksum"], SHA1(body))
```

**Common Use Cases**:
- Content hashing
- Checksums
- Deduplication

**Parameters**:
- `value`: String to hash

---

### SHA256
**Purpose**: Computes SHA-256 hash
**Signature**: `SHA256(value) -> string`
**Full Reference**: [sha256.md](references/standard/sha256.md)

**Quick Copy**:
```yaml
- set(attributes["hash"], SHA256(Concat([attributes["user"], attributes["timestamp"]], ":")))
```

**Common Use Cases**:
- Secure hashing
- Data integrity
- Cryptographic operations

**Parameters**:
- `value`: String to hash

---

### SHA512
**Purpose**: Computes SHA-512 hash
**Signature**: `SHA512(value) -> string`
**Full Reference**: [sha512.md](references/standard/sha512.md)

**Quick Copy**:
```yaml
- set(attributes["secure_hash"], SHA512(body))
```

**Common Use Cases**:
- Strong cryptographic hashing
- Security applications
- Data integrity

**Parameters**:
- `value`: String to hash

---

### slice_to_map
**Purpose**: Converts a slice of key-value pairs to a map
**Signature**: `slice_to_map(slice) -> map`
**Full Reference**: [slice_to_map.md](references/standard/slice_to_map.md)

**Quick Copy**:
```yaml
- set(attributes, slice_to_map(attributes["pairs"]))
```

**Common Use Cases**:
- Converting arrays to objects
- Restructuring data
- Dynamic attribute creation

**Parameters**:
- `slice`: Slice of key-value pairs

---

### sort
**Purpose**: Sorts a list
**Signature**: `sort(list, Optional[reverse]) -> []any`
**Full Reference**: [sort.md](references/standard/sort.md)

**Quick Copy**:
```yaml
- set(attributes["sorted_tags"], sort(attributes["tags"], false))
```

**Common Use Cases**:
- Sorting lists
- Normalizing arrays
- Ordering data

**Parameters**:
- `list`: List to sort
- `reverse`: Optional boolean to reverse sort

---

### span_id
**Purpose**: Returns the current span ID
**Signature**: `span_id() -> SpanID`
**Full Reference**: [span_id.md](references/standard/span_id.md)

**Quick Copy**:
```yaml
- set(attributes["span_id"], span_id().string)
```

**Common Use Cases**:
- Span identification
- Trace correlation
- Debugging

**Parameters**:
- None

---

### split
**Purpose**: Splits a string by delimiter
**Signature**: `split(value, delimiter) -> []string`
**Full Reference**: [split.md](references/standard/split.md)

**Quick Copy**:
```yaml
- set(attributes["tags"], split(attributes["tag_string"], ","))
```

**Common Use Cases**:
- Parsing delimited strings
- Creating arrays from strings
- Tag extraction

**Parameters**:
- `value`: String to split
- `delimiter`: Split delimiter

---

### string
**Purpose**: Converts a value to string
**Signature**: `string(value) -> string`
**Full Reference**: [string.md](references/standard/string.md)

**Quick Copy**:
```yaml
- set(attributes["status"], string(attributes["status_code"]))
```

**Common Use Cases**:
- Type conversion
- String formatting
- Display values

**Parameters**:
- `value`: Value to convert

---

### substring
**Purpose**: Extracts a substring
**Signature**: `substring(value, start, length) -> string`
**Full Reference**: [substring.md](references/standard/substring.md)

**Quick Copy**:
```yaml
- set(attributes["prefix"], substring(attributes["id"], 0, 8))
```

**Common Use Cases**:
- Extracting portions of strings
- Truncating values
- ID prefixing

**Parameters**:
- `value`: String to extract from
- `start`: Start index
- `length`: Length to extract

---

### time
**Purpose**: Parses a time string with a layout
**Signature**: `time(value, format) -> Time`
**Full Reference**: [time.md](references/standard/time.md)

**Quick Copy**:
```yaml
- set(attributes["parsed_time"], time(attributes["timestamp"], "2006-01-02 15:04:05"))
```

**Common Use Cases**:
- Parsing timestamp strings
- Converting date formats
- Time extraction

**Parameters**:
- `value`: Time string
- `format`: Go time layout

---

### to_camel_case
**Purpose**: Converts string to camelCase
**Signature**: `to_camel_case(value) -> string`
**Full Reference**: [to_camel_case.md](references/standard/to_camel_case.md)

**Quick Copy**:
```yaml
- set(attributes["fieldName"], to_camel_case("field_name"))
```

**Common Use Cases**:
- Normalizing field names
- API compatibility
- Formatting output

**Parameters**:
- `value`: String to convert

---

### to_key_value_string
**Purpose**: Converts a map to key=value string
**Signature**: `to_key_value_string(target, Optional[delimiter], Optional[pair_delimiter]) -> string`
**Full Reference**: [to_key_value_string.md](references/standard/to_key_value_string.md)

**Quick Copy**:
```yaml
- set(body, to_key_value_string(attributes, "=", " "))
```

**Common Use Cases**:
- Converting to logfmt
- Creating query strings
- Serializing attributes

**Parameters**:
- `target`: Map to convert
- `delimiter`: Key-value separator (default "=")
- `pair_delimiter`: Pair separator (default " ")

---

### to_lower_case
**Purpose**: Converts string to lowercase
**Signature**: `to_lower_case(value) -> string`
**Full Reference**: [to_lower_case.md](references/standard/to_lower_case.md)

**Quick Copy**:
```yaml
- set(attributes["service"], to_lower_case(attributes["service"]))
```

**Common Use Cases**:
- Case normalization
- Case-insensitive comparison
- Standardizing data

**Parameters**:
- `value`: String to convert

---

### to_snake_case
**Purpose**: Converts string to snake_case
**Signature**: `to_snake_case(value) -> string`
**Full Reference**: [to_snake_case.md](references/standard/to_snake_case.md)

**Quick Copy**:
```yaml
- set(attributes["field_name"], to_snake_case("fieldName"))
```

**Common Use Cases**:
- Normalizing field names
- Database compatibility
- Code style conversion

**Parameters**:
- `value`: String to convert

---

### to_upper_case
**Purpose**: Converts string to uppercase
**Signature**: `to_upper_case(value) -> string`
**Full Reference**: [to_upper_case.md](references/standard/to_upper_case.md)

**Quick Copy**:
```yaml
- set(severity_text, to_upper_case(severity_text))
```

**Common Use Cases**:
- Case normalization
- Formatting output
- Severity standardization

**Parameters**:
- `value`: String to convert

---

### trace_id
**Purpose**: Returns the current trace ID
**Signature**: `trace_id() -> TraceID`
**Full Reference**: [trace_id.md](references/standard/trace_id.md)

**Quick Copy**:
```yaml
- set(attributes["trace_id"], trace_id().string)
```

**Common Use Cases**:
- Trace identification
- Correlation
- Debugging

**Parameters**:
- None

---

### trim
**Purpose**: Trims whitespace from string
**Signature**: `trim(value) -> string`
**Full Reference**: [trim.md](references/standard/trim.md)

**Quick Copy**:
```yaml
- set(attributes["message"], trim(body))
```

**Common Use Cases**:
- Cleaning input
- Normalizing strings
- Removing whitespace

**Parameters**:
- `value`: String to trim

---

### truncate_time
**Purpose**: Truncates time to a specified duration unit
**Signature**: `truncate_time(time, duration) -> Time`
**Full Reference**: [truncate_time.md](references/standard/truncate_time.md)

**Quick Copy**:
```yaml
- set(attributes["hour_bucket"], truncate_time(time_unix_nano, duration("1h")))
```

**Common Use Cases**:
- Time bucketing
- Aggregation windows
- Rounding timestamps

**Parameters**:
- `time`: Time to truncate
- `duration`: Truncation unit

---

### unix
**Purpose**: Converts time to Unix timestamp (seconds)
**Signature**: `unix(time) -> int64`
**Full Reference**: [unix.md](references/standard/unix.md)

**Quick Copy**:
```yaml
- set(attributes["timestamp"], unix(time_unix_nano))
```

**Common Use Cases**:
- Unix timestamp conversion
- Epoch time generation
- Time serialization

**Parameters**:
- `time`: Time value

---

### unix_micro
**Purpose**: Converts time to Unix timestamp (microseconds)
**Signature**: `unix_micro(time) -> int64`
**Full Reference**: [unix_micro.md](references/standard/unix_micro.md)

**Quick Copy**:
```yaml
- set(attributes["timestamp_us"], unix_micro(time_unix_nano))
```

**Common Use Cases**:
- Microsecond timestamps
- High-precision timing
- Time conversion

**Parameters**:
- `time`: Time value

---

### unix_milli
**Purpose**: Converts time to Unix timestamp (milliseconds)
**Signature**: `unix_milli(time) -> int64`
**Full Reference**: [unix_milli.md](references/standard/unix_milli.md)

**Quick Copy**:
```yaml
- set(attributes["timestamp_ms"], unix_milli(time_unix_nano))
```

**Common Use Cases**:
- Millisecond timestamps
- JavaScript compatibility
- Time conversion

**Parameters**:
- `time`: Time value

---

### unix_nano
**Purpose**: Converts time to Unix timestamp (nanoseconds)
**Signature**: `unix_nano(time) -> int64`
**Full Reference**: [unix_nano.md](references/standard/unix_nano.md)

**Quick Copy**:
```yaml
- set(attributes["timestamp_ns"], unix_nano(now()))
```

**Common Use Cases**:
- Nanosecond timestamps
- High-precision timing
- Time conversion

**Parameters**:
- `time`: Time value

---

### unix_seconds
**Purpose**: Converts time to Unix timestamp (seconds, alias for unix)
**Signature**: `unix_seconds(time) -> int64`
**Full Reference**: [unix_seconds.md](references/standard/unix_seconds.md)

**Quick Copy**:
```yaml
- set(attributes["timestamp_sec"], unix_seconds(time_unix_nano))
```

**Common Use Cases**:
- Second timestamps
- Standard Unix time
- Time conversion

**Parameters**:
- `time`: Time value

---

### url
**Purpose**: Parses a URL into components
**Signature**: `url(value) -> URL`
**Full Reference**: [url.md](references/standard/url.md)

**Quick Copy**:
```yaml
- set(attributes["host"], url(attributes["url"]).host)
- set(attributes["path"], url(attributes["url"]).path)
```

**Common Use Cases**:
- URL parsing
- Extracting URL components
- Path analysis

**Parameters**:
- `value`: URL string

---

### useragent
**Purpose**: Parses user agent string
**Signature**: `useragent(value) -> UserAgent`
**Full Reference**: [useragent.md](references/standard/useragent.md)

**Quick Copy**:
```yaml
- set(attributes["browser"], useragent(attributes["user_agent"]).browser)
- set(attributes["os"], useragent(attributes["user_agent"]).os)
```

**Common Use Cases**:
- Browser detection
- Device identification
- Analytics

**Parameters**:
- `value`: User agent string

---

### uuid
**Purpose**: Generates a random UUID v4
**Signature**: `uuid() -> string`
**Full Reference**: [uuid.md](references/standard/uuid.md)

**Quick Copy**:
```yaml
- set(attributes["request_id"], uuid())
```

**Common Use Cases**:
- Generating unique IDs
- Request tracking
- Correlation IDs

**Parameters**:
- None

---

### uuidv7
**Purpose**: Generates a time-ordered UUID v7
**Signature**: `uuidv7() -> string`
**Full Reference**: [uuidv7.md](references/standard/uuidv7.md)

**Quick Copy**:
```yaml
- set(attributes["event_id"], uuidv7())
```

**Common Use Cases**:
- Time-ordered IDs
- Sortable UUIDs
- Event tracking

**Parameters**:
- None

---

### values
**Purpose**: Returns all values from a map as a list
**Signature**: `values(target) -> []any`
**Full Reference**: [values.md](references/standard/values.md)

**Quick Copy**:
```yaml
- set(attributes["all_values"], values(attributes))
```

**Common Use Cases**:
- Extracting values
- Data transformation
- Value aggregation

**Parameters**:
- `target`: Map to extract values from

---

### weekday
**Purpose**: Returns day of week (0=Sunday, 6=Saturday)
**Signature**: `weekday(time) -> int`
**Full Reference**: [weekday.md](references/standard/weekday.md)

**Quick Copy**:
```yaml
- set(attributes["day_of_week"], weekday(time_unix_nano))
```

**Common Use Cases**:
- Day-of-week analysis
- Weekend detection
- Schedule-based routing

**Parameters**:
- `time`: Time value

---

### year
**Purpose**: Extracts year from a time
**Signature**: `year(time) -> int`
**Full Reference**: [year.md](references/standard/year.md)

**Quick Copy**:
```yaml
- set(attributes["year"], year(time_unix_nano))
```

**Common Use Cases**:
- Year extraction
- Annual aggregation
- Date parsing

**Parameters**:
- `time`: Time value

---

## Type Checkers

Functions for checking value types.

### is_bool
See [OTTL Converters](#is_bool) section above.

### is_double
See [OTTL Converters](#is_double) section above.

### is_int
See [OTTL Converters](#is_int) section above.

### is_list
See [OTTL Converters](#is_list) section above.

### is_map
See [OTTL Converters](#is_map) section above.

### is_match
See [OTTL Converters](#is_match) section above.

### is_root_span
See [OTTL Converters](#is_root_span) section above.

### is_string
See [OTTL Converters](#is_string) section above.

---

## Hashing Functions

Functions for generating hashes.

### FNV
See [OTTL Converters](#fnv) section above.

### MD5
See [OTTL Converters](#md5) section above.

### MurmurHash3
See [OTTL Converters](#murmurhash3) section above.

### MurmurHash3_128
See [OTTL Converters](#murmurhash3_128) section above.

### SHA1
See [OTTL Converters](#sha1) section above.

### SHA256
See [OTTL Converters](#sha256) section above.

### SHA512
See [OTTL Converters](#sha512) section above.

---

## Time Functions

Functions for working with time and duration.

### day
See [OTTL Converters](#day) section above.

### duration
See [OTTL Converters](#duration) section above.

### formattime
See [OTTL Converters](#formattime) section above.

### hour
See [OTTL Converters](#hour) section above.

### hours
See [OTTL Converters](#hours) section above.

### microseconds
See [OTTL Converters](#microseconds) section above.

### milliseconds
See [OTTL Converters](#milliseconds) section above.

### minute
See [OTTL Converters](#minute) section above.

### minutes
See [OTTL Converters](#minutes) section above.

### month
See [OTTL Converters](#month) section above.

### nanosecond
See [OTTL Converters](#nanosecond) section above.

### nanoseconds
See [OTTL Converters](#nanoseconds) section above.

### now
See [OTTL Converters](#now) section above.

### second
See [OTTL Converters](#second) section above.

### seconds
See [OTTL Converters](#seconds) section above.

### time
See [OTTL Converters](#time) section above.

### truncate_time
See [OTTL Converters](#truncate_time) section above.

### unix
See [OTTL Converters](#unix) section above.

### unix_micro
See [OTTL Converters](#unix_micro) section above.

### unix_milli
See [OTTL Converters](#unix_milli) section above.

### unix_nano
See [OTTL Converters](#unix_nano) section above.

### unix_seconds
See [OTTL Converters](#unix_seconds) section above.

### weekday
See [OTTL Converters](#weekday) section above.

### year
See [OTTL Converters](#year) section above.

---

## XML Functions

Functions for working with XML data.

### convert_attributes_to_elements_xml
**Purpose**: Converts XML attributes to child elements
**Signature**: `convert_attributes_to_elements_xml(target) -> void`
**Full Reference**: [convert_attributes_to_elements_xml.md](references/standard/convert_attributes_to_elements_xml.md)

**Quick Copy**:
```yaml
- set(attributes["xml"], parse_xml(body))
- convert_attributes_to_elements_xml(attributes["xml"])
```

**Common Use Cases**:
- Simplifying XML structure
- Converting attributes to elements
- XML normalization

**Parameters**:
- `target`: Parsed XML map

---

### convert_text_to_elements_xml
**Purpose**: Converts text nodes to elements
**Signature**: `convert_text_to_elements_xml(target) -> void`
**Full Reference**: [convert_text_to_elements_xml.md](references/standard/convert_text_to_elements_xml.md)

**Quick Copy**:
```yaml
- set(attributes["xml"], parse_xml(body))
- convert_text_to_elements_xml(attributes["xml"])
```

**Common Use Cases**:
- XML structure transformation
- Normalizing text nodes
- Consistent XML format

**Parameters**:
- `target`: Parsed XML map

---

### get_xml
**Purpose**: Extracts value from XML using XPath
**Signature**: `get_xml(target, xpath) -> any`
**Full Reference**: [get_xml.md](references/standard/get_xml.md)

**Quick Copy**:
```yaml
- set(attributes["value"], get_xml(attributes["xml"], "/root/element"))
```

**Common Use Cases**:
- XPath queries
- Extracting XML values
- XML data extraction

**Parameters**:
- `target`: Parsed XML map
- `xpath`: XPath expression

---

### insert_xml
**Purpose**: Inserts or updates XML value at XPath
**Signature**: `insert_xml(target, xpath, value) -> void`
**Full Reference**: [insert_xml.md](references/standard/insert_xml.md)

**Quick Copy**:
```yaml
- insert_xml(attributes["xml"], "/root/new_element", "value")
```

**Common Use Cases**:
- Modifying XML
- Adding XML elements
- XML enrichment

**Parameters**:
- `target`: Parsed XML map
- `xpath`: XPath expression
- `value`: Value to insert

---

### parse_xml
See [OTTL Converters](#parse_xml) section above.

### parse_simplified_xml
See [OTTL Converters](#parse_simplified_xml) section above.

### remove_xml
**Purpose**: Removes XML element at XPath
**Signature**: `remove_xml(target, xpath) -> void`
**Full Reference**: [remove_xml.md](references/standard/remove_xml.md)

**Quick Copy**:
```yaml
- remove_xml(attributes["xml"], "/root/unwanted_element")
```

**Common Use Cases**:
- Removing XML elements
- XML cleanup
- Data filtering

**Parameters**:
- `target`: Parsed XML map
- `xpath`: XPath expression

---

## EDX Extensions

EdgeDelta-specific OTTL extensions for advanced processing.

### EDX Converters

EDX converter functions that transform and return values.

### EDXCoalesce
**Purpose**: Returns first non-null value from a list
**Signature**: `EDXCoalesce(values...) -> any`
**Full Reference**: [edxcoalesce.md](references/edx/edxcoalesce.md)

**Quick Copy**:
```yaml
- set(attributes["service"], EDXCoalesce(attributes["service.name"], attributes["service"], "unknown"))
```

**Common Use Cases**:
- Providing default values
- Handling missing fields
- Fallback chains

**Parameters**:
- `values`: Variable number of values to check

---

### EDXCompress
**Purpose**: Compresses data using gzip, zlib, or zstd
**Signature**: `EDXCompress(value, algorithm) -> string`
**Full Reference**: [edxcompress.md](references/edx/edxcompress.md)

**Quick Copy**:
```yaml
- set(attributes["compressed"], EDXCompress(body, "gzip"))
- set(attributes["compressed_zstd"], EDXCompress(body, "zstd"))
```

**Common Use Cases**:
- Compressing large payloads
- Reducing data size
- Storage optimization

**Parameters**:
- `value`: String to compress
- `algorithm`: "gzip", "zlib", or "zstd"

---

### EDXDataType
**Purpose**: Returns the type of a value as a string
**Signature**: `EDXDataType(value) -> string`
**Full Reference**: [edxdatatype.md](references/edx/edxdatatype.md)

**Quick Copy**:
```yaml
- set(attributes["field_type"], EDXDataType(attributes["value"]))
```

**Common Use Cases**:
- Type inspection
- Debugging data types
- Dynamic processing

**Parameters**:
- `value`: Value to check

**Returns**: "string", "int", "double", "bool", "map", "slice", "nil"

---

### EDXDecode
**Purpose**: Decodes base64, hex, or URL-encoded strings
**Signature**: `EDXDecode(value, encoding) -> string`
**Full Reference**: [edxdecode.md](references/edx/edxdecode.md)

**Quick Copy**:
```yaml
- set(attributes["decoded"], EDXDecode(attributes["data"], "base64"))
- set(attributes["url_decoded"], EDXDecode(attributes["param"], "url"))
- set(attributes["hex_decoded"], EDXDecode(attributes["hex_data"], "hex"))
```

**Common Use Cases**:
- Decoding encoded payloads
- Processing URL parameters
- Converting hex data

**Parameters**:
- `value`: Encoded string
- `encoding`: "base64", "hex", or "url"

---

### EDXDecrypt
**Purpose**: Decrypts data using AES
**Signature**: `EDXDecrypt(value, key, Optional[algorithm]) -> string`
**Full Reference**: [edxdecrypt.md](references/edx/edxdecrypt.md)

**Quick Copy**:
```yaml
- set(attributes["decrypted"], EDXDecrypt(attributes["encrypted"], EDXEnv("ENCRYPTION_KEY"), "aes"))
```

**Common Use Cases**:
- Decrypting sensitive data
- Processing encrypted payloads
- Security operations

**Parameters**:
- `value`: Encrypted string (base64)
- `key`: Encryption key
- `algorithm`: Optional, defaults to "aes"

---

### EDXDecompress
**Purpose**: Decompresses data using gzip, zlib, or zstd
**Signature**: `EDXDecompress(value, algorithm) -> string`
**Full Reference**: [edxdecompress.md](references/edx/edxdecompress.md)

**Quick Copy**:
```yaml
- set(body, EDXDecompress(attributes["compressed"], "gzip"))
- set(body, EDXDecompress(attributes["compressed"], "zstd"))
```

**Common Use Cases**:
- Decompressing payloads
- Processing compressed logs
- Data extraction

**Parameters**:
- `value`: Compressed data (base64 or raw)
- `algorithm`: "gzip", "zlib", or "zstd"

---

### EDXEncode
**Purpose**: Encodes strings to base64, hex, or URL encoding
**Signature**: `EDXEncode(value, encoding) -> string`
**Full Reference**: [edxencode.md](references/edx/edxencode.md)

**Quick Copy**:
```yaml
- set(attributes["encoded"], EDXEncode(body, "base64"))
- set(attributes["url_encoded"], EDXEncode(attributes["param"], "url"))
- set(attributes["hex_encoded"], EDXEncode(body, "hex"))
```

**Common Use Cases**:
- Encoding payloads
- URL parameter encoding
- Creating hex representations

**Parameters**:
- `value`: String to encode
- `encoding`: "base64", "hex", or "url"

---

### EDXEncrypt
**Purpose**: Encrypts data using AES
**Signature**: `EDXEncrypt(value, key, Optional[algorithm]) -> string`
**Full Reference**: [edxencrypt.md](references/edx/edxencrypt.md)

**Quick Copy**:
```yaml
- set(attributes["encrypted"], EDXEncrypt(attributes["sensitive"], EDXEnv("ENCRYPTION_KEY"), "aes"))
```

**Common Use Cases**:
- Encrypting sensitive data
- Secure data transmission
- PII protection

**Parameters**:
- `value`: String to encrypt
- `key`: Encryption key
- `algorithm`: Optional, defaults to "aes"

**Returns**: Base64-encoded encrypted string

---

### EDXEnv
**Purpose**: Reads environment variable value
**Signature**: `EDXEnv(variable_name, Optional[default]) -> string`
**Full Reference**: [edxenv.md](references/edx/edxenv.md)

**Quick Copy**:
```yaml
- set(attributes["api_key"], EDXEnv("API_KEY", "default-key"))
- set(attributes["env"], EDXEnv("ENVIRONMENT", "production"))
```

**Common Use Cases**:
- Reading configuration from environment
- Accessing secrets
- Dynamic configuration

**Parameters**:
- `variable_name`: Name of environment variable
- `default`: Optional default value if not found

---

### EDXExtractPatterns
**Purpose**: Extracts patterns using regex with named groups (advanced version)
**Signature**: `EDXExtractPatterns(text, pattern) -> map`
**Full Reference**: [edxextractpatterns.md](references/edx/edxextractpatterns.md)

**Quick Copy**:
```yaml
- set(attributes, EDXExtractPatterns(body, "(?P<timestamp>\\S+) (?P<level>\\w+) (?P<message>.+)"))
```

**Common Use Cases**:
- Advanced log parsing
- Complex pattern extraction
- Multi-field extraction

**Parameters**:
- `text`: String to parse
- `pattern`: Regex with named groups

---

### EDXHmac
**Purpose**: Computes HMAC hash with specified algorithm
**Signature**: `EDXHmac(value, key, algorithm) -> string`
**Full Reference**: [edxhmac.md](references/edx/edxhmac.md)

**Quick Copy**:
```yaml
- set(attributes["signature"], EDXHmac(body, EDXEnv("HMAC_KEY"), "sha256"))
- set(attributes["hmac"], EDXHmac(attributes["data"], "secret", "sha1"))
```

**Common Use Cases**:
- Message authentication
- API signatures
- Data integrity verification

**Parameters**:
- `value`: String to hash
- `key`: Secret key
- `algorithm`: "md5", "sha1", "sha256", "sha512"

---

### EDXIfElse
**Purpose**: Conditional expression (ternary operator)
**Signature**: `EDXIfElse(condition, true_value, false_value) -> any`
**Full Reference**: [edxifelse.md](references/edx/edxifelse.md)

**Quick Copy**:
```yaml
- set(severity_text, EDXIfElse(attributes["status"] >= 500, "ERROR", "INFO"))
- set(attributes["priority"], EDXIfElse(contains_value(attributes, "critical"), "high", "normal"))
```

**Common Use Cases**:
- Conditional value assignment
- Dynamic routing
- Status determination

**Parameters**:
- `condition`: Boolean expression
- `true_value`: Value if condition is true
- `false_value`: Value if condition is false

---

### EDXParseKeyValue
**Purpose**: Parses key-value pairs with advanced options
**Signature**: `EDXParseKeyValue(value, Optional[delimiter], Optional[pair_delimiter], Optional[options]) -> map`
**Full Reference**: [edxparsekeyvalue.md](references/edx/edxparsekeyvalue.md)

**Quick Copy**:
```yaml
- set(attributes, EDXParseKeyValue(body, "=", " "))
- set(attributes, EDXParseKeyValue(body, ":", ","))
```

**Common Use Cases**:
- Advanced logfmt parsing
- Custom key-value formats
- Flexible parsing

**Parameters**:
- `value`: String to parse
- `delimiter`: Key-value separator (default "=")
- `pair_delimiter`: Pair separator (default " ")
- `options`: Optional parsing options

---

### EDXRedis
**Purpose**: Executes Redis commands for caching, lookups, and state management
**Signature**: `EDXRedis(command, key, Optional[value], Optional[ttl]) -> string`
**Full Reference**: [edxredis.md](references/edx/edxredis.md)

**Quick Copy - Basic Commands**:
```yaml
# GET - Retrieve value
- set(attributes["cached_value"], EDXRedis("GET", "user:12345"))

# SET - Store value with TTL
- set(attributes["result"], EDXRedis("SET", "session:abc", attributes["session_data"], 3600))

# EXISTS - Check key existence
- set(attributes["user_exists"], EDXRedis("EXISTS", "user:12345"))

# DEL - Delete key
- set(attributes["deleted"], EDXRedis("DEL", "temp:key"))

# INCR - Increment counter
- set(attributes["request_count"], EDXRedis("INCR", "counter:requests"))

# EXPIRE - Set TTL on existing key
- set(attributes["ttl_set"], EDXRedis("EXPIRE", "session:abc", "300"))
```

**Quick Copy - TLS Connection**:
```yaml
# Environment variables needed:
# REDIS_HOST=redis.example.com:6379
# REDIS_PASSWORD=your-password
# REDIS_TLS_ENABLED=true
# REDIS_TLS_CERT=/path/to/client.crt
# REDIS_TLS_KEY=/path/to/client.key
# REDIS_TLS_CA=/path/to/ca.crt

- set(attributes["user_data"], EDXRedis("GET", concat(["user:", attributes["user_id"]], "")))
```

**Quick Copy - Advanced Patterns**:
```yaml
# Cache-aside pattern with TTL
- set(attributes["cache_key"], concat(["cache:", attributes["entity_id"]], ""))
- set(attributes["cached"], EDXRedis("GET", attributes["cache_key"]))
- set(attributes["value"], EDXIfElse(
    attributes["cached"] != "",
    attributes["cached"],
    EDXRedis("SET", attributes["cache_key"], attributes["fresh_data"], 300)
  ))

# Deduplication with EXISTS
- set(attributes["is_duplicate"], EDXRedis("EXISTS", concat(["seen:", attributes["event_id"]], "")))
- set(attributes["processed"], EDXIfElse(
    attributes["is_duplicate"] == "0",
    EDXRedis("SET", concat(["seen:", attributes["event_id"]], ""), "1", 86400),
    "skipped"
  ))

# Rate limiting with INCR
- set(attributes["rate_key"], concat(["rate:", attributes["user_id"], ":", year(now()), ":", month(now())], ""))
- set(attributes["request_count"], EDXRedis("INCR", attributes["rate_key"]))
- set(attributes["rate_limited"], EDXIfElse(int(attributes["request_count"]) > 1000, "true", "false"))
```

**Common Use Cases**:
- **Caching**: Store and retrieve processed data to avoid reprocessing
- **Deduplication**: Track seen events using EXISTS and SET
- **Rate Limiting**: Count requests per user/IP using INCR
- **Session Management**: Store session data with automatic expiration
- **Enrichment**: Join telemetry with external data from Redis
- **State Tracking**: Maintain processing state across pipeline runs
- **Feature Flags**: Store and retrieve feature toggles

**Supported Commands**:
- `GET`: Retrieve string value
- `SET`: Store string value (with optional TTL)
- `EXISTS`: Check if key exists (returns "1" or "0")
- `DEL`: Delete key (returns number of keys deleted)
- `INCR`: Increment integer value (returns new value)
- `EXPIRE`: Set TTL on existing key (seconds)
- `TTL`: Get remaining TTL (seconds)
- `HGET`: Get hash field value
- `HSET`: Set hash field value
- `LPUSH`: Push to list (left)
- `RPUSH`: Push to list (right)
- `LPOP`: Pop from list (left)
- `RPOP`: Pop from list (right)

**Parameters**:
- `command`: Redis command (GET, SET, EXISTS, DEL, INCR, EXPIRE, etc.)
- `key`: Redis key to operate on
- `value`: Optional value for SET, HSET, LPUSH, RPUSH
- `ttl`: Optional TTL in seconds for SET command

**Connection Configuration** (via environment variables):
- `REDIS_HOST`: Redis server address (default: "localhost:6379")
- `REDIS_PASSWORD`: Authentication password (optional)
- `REDIS_DB`: Database number (default: 0)
- `REDIS_TLS_ENABLED`: Enable TLS (default: false)
- `REDIS_TLS_CERT`: Client certificate path (for mutual TLS)
- `REDIS_TLS_KEY`: Client key path (for mutual TLS)
- `REDIS_TLS_CA`: CA certificate path (for TLS verification)
- `REDIS_TLS_INSECURE_SKIP_VERIFY`: Skip TLS verification (default: false)

**Return Values**:
- `GET`: String value or empty string if not found
- `SET`: "OK" on success
- `EXISTS`: "1" if exists, "0" if not
- `DEL`: Number of deleted keys as string
- `INCR`: New value as string
- `EXPIRE`: "1" if TTL set, "0" if key doesn't exist

**Error Handling**:
- Returns empty string on errors
- Check logs for connection/command errors
- Use EDXIfElse for fallback logic

---

### EDXUnescapeJSON
**Purpose**: Unescapes JSON escape sequences in strings
**Signature**: `EDXUnescapeJSON(value) -> string`
**Full Reference**: [edxunescapejson.md](references/edx/edxunescapejson.md)

**Quick Copy**:
```yaml
- set(body, EDXUnescapeJSON(attributes["escaped_json"]))
- set(attributes["clean"], EDXUnescapeJSON(attributes["json_string"]))
```

**Common Use Cases**:
- Cleaning escaped JSON
- Processing nested JSON strings
- Normalizing JSON data

**Parameters**:
- `value`: String with JSON escape sequences

---

### EDX Editors

EDX editor functions that modify data in place.

### edx_code
**Purpose**: Executes JavaScript code for custom transformations
**Signature**: `edx_code(code) -> void`
**Full Reference**: [edxcode.md](references/edx/edxcode.md)

**Quick Copy**:
```yaml
# Used in Code Processor - JavaScript modifies 'item' object directly
statements: |-
  edx_code("item['attributes']['severity'] = item['resource']['raw_data']['alert_type'] === 'error' ? 1 : item['resource']['raw_data']['alert_type'] === 'warning' ? 2 : 0;")
```

**Common Use Cases**:
- Custom transformations beyond OTTL
- Complex business logic
- Conditional field assignments

**Parameters**:
- `code`: JavaScript code string that modifies the `item` object

**Important Notes**:
- Function name is `edx_code` (lowercase_underscore), not `EDXCode`
- Used in Code Processor where `item` variable contains the data
- Code automatically wrapped in transform function with `return item;`
- Modifies data in place (Editor function)

**Warning**: Use sparingly - has performance overhead

---

### edx_delete_empty_values
**Purpose**: Removes all empty/null values from a map
**Signature**: `edx_delete_empty_values(target) -> void`
**Full Reference**: [edx_delete_empty_values.md](references/edx/edx_delete_empty_values.md)

**Quick Copy**:
```yaml
- edx_delete_empty_values(attributes)
```

**Common Use Cases**:
- Cleaning sparse data
- Removing null fields
- Data quality improvement

**Parameters**:
- `target`: Map to clean

---

### edx_delete_keys
**Purpose**: Deletes multiple keys from a map
**Signature**: `edx_delete_keys(target, keys[]) -> void`
**Full Reference**: [edx_delete_keys.md](references/edx/edx_delete_keys.md)

**Quick Copy**:
```yaml
- edx_delete_keys(attributes, ["temp_field", "debug_info", "internal_id"])
```

**Common Use Cases**:
- Bulk key removal
- Cleaning unwanted fields
- Data sanitization

**Parameters**:
- `target`: Map to modify
- `keys`: List of keys to delete

---

### edx_delete_matching_keys
**Purpose**: Deletes keys matching multiple patterns
**Signature**: `edx_delete_matching_keys(target, patterns[]) -> void`
**Full Reference**: [edx_delete_matching_keys.md](references/edx/edx_delete_matching_keys.md)

**Quick Copy**:
```yaml
- edx_delete_matching_keys(attributes, ["^temp_.*", "^debug_.*", ".*_internal$"])
```

**Common Use Cases**:
- Pattern-based bulk removal
- Cleaning by namespace
- Dynamic field removal

**Parameters**:
- `target`: Map to modify
- `patterns`: List of regex patterns

---

### edx_keep_keys
**Purpose**: Keeps specified keys, removes all others (improved version)
**Signature**: `edx_keep_keys(target, keys[]) -> void`
**Full Reference**: [edx_keep_keys.md](references/edx/edx_keep_keys.md)

**Quick Copy**:
```yaml
- edx_keep_keys(attributes, ["service.name", "host.name", "trace_id", "level"])
```

**Common Use Cases**:
- Strict allowlisting
- Reducing cardinality
- Essential fields only

**Parameters**:
- `target`: Map to filter
- `keys`: List of keys to keep

---

### edx_keep_matching_keys
**Purpose**: Keeps keys matching any of the patterns
**Signature**: `edx_keep_matching_keys(target, patterns[]) -> void`
**Full Reference**: [edx_keep_matching_keys.md](references/edx/edx_keep_matching_keys.md)

**Quick Copy**:
```yaml
- edx_keep_matching_keys(attributes, ["^service\\.", "^host\\.", "^k8s\\."])
```

**Common Use Cases**:
- Namespace-based filtering
- Pattern allowlisting
- Multi-pattern keeping

**Parameters**:
- `target`: Map to filter
- `patterns`: List of regex patterns to keep

---

### edx_map_keys
**Purpose**: Renames or transforms keys using a mapping
**Signature**: `edx_map_keys(target, mapping) -> void`
**Full Reference**: [edx_map_keys.md](references/edx/edx_map_keys.md)

**Quick Copy**:
```yaml
- edx_map_keys(attributes, {
    "old_name": "new_name",
    "legacy_field": "modern_field",
    "svc": "service.name"
  })
```

**Common Use Cases**:
- Field renaming
- Schema migration
- Standardizing field names

**Parameters**:
- `target`: Map to modify
- `mapping`: Map of old_key -> new_key

---

## Quick Reference Tables

### By Use Case

**Data Extraction & Parsing**:
- `extract_patterns`, `extract_grok_patterns`, `EDXExtractPatterns`
- `parse_json`, `parse_csv`, `parse_key_value`, `EDXParseKeyValue`
- `parse_xml`, `parse_simplified_xml`

**Type Conversion**:
- `int`, `double`, `string`
- `time`, `duration`
- `bool` (via `is_bool`)

**String Manipulation**:
- `concat`, `split`, `substring`, `trim`
- `to_upper_case`, `to_lower_case`, `to_camel_case`, `to_snake_case`
- `replace_match`, `replace_all_matches`

**Hashing & Encryption**:
- `MD5`, `SHA1`, `SHA256`, `SHA512`
- `FNV`, `MurmurHash3`, `MurmurHash3_128`
- `EDXEncrypt`, `EDXDecrypt`, `EDXHmac`

**Encoding/Decoding**:
- `base64decode`, `decode` (URL)
- `EDXEncode`, `EDXDecode` (base64/hex/URL)
- `EDXCompress`, `EDXDecompress`

**Time Operations**:
- `now`, `time`, `formattime`
- `unix`, `unix_milli`, `unix_micro`, `unix_nano`
- `year`, `month`, `day`, `hour`, `minute`, `second`
- `truncate_time`, `duration`

**Map/List Operations**:
- `keys`, `values`, `len`
- `merge_maps`, `flatten`
- `keep_keys`, `keep_matching_keys`, `edx_keep_keys`
- `delete_key`, `delete_matching_keys`, `edx_delete_keys`

**Conditional Logic**:
- `EDXIfElse`
- `is_match`, `has_prefix`, `has_suffix`
- `contains_value`, `is_bool`, `is_int`, `is_double`

**State & External Data**:
- `EDXRedis` (caching, counters, lookups)
- `EDXEnv` (environment variables)

**ID Generation**:
- `uuid`, `uuidv7`
- `trace_id`, `span_id`, `profile_id`

---

## Navigation Tips

1. **Search by keyword**: Use Cmd+F (Mac) or Ctrl+F (Windows) to find functions
2. **Copy examples**: Use Quick Copy snippets for rapid prototyping
3. **Check categories**: Browse by category for related functions
4. **Read full docs**: Follow reference links for detailed documentation
5. **Test incrementally**: Start with simple examples, build complexity

---

## Contributing

To add new functions or improve examples:
1. Follow the established format
2. Include realistic Quick Copy examples
3. Add common use cases
4. Link to detailed reference docs
5. Update category counts in header

---

**Total Functions**: 124 (101 OTTL standard + 23 EDX extensions)
**Last Verified**: 2025-10-20
**Maintained By**: EdgeDelta Documentation Team
