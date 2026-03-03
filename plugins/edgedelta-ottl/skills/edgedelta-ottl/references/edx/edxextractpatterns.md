# EDXExtractPatterns - Dynamic Regex Extraction

**Function Type**: EDX Converter
**Purpose**: Parses strings using regular expressions and returns named capture groups as a map
**Signature**: `EDXExtractPatterns(input, pattern) -> map[string]string`
**Return Type**: map[string]string (named groups)
**Minimum Agent Version**: v1.22.0
**Source**: /pkg/ottlext/funcs/func_edx_extract_patterns.go

---

## Quick Copy

```yaml
# Extract filename from path
set(cache["parsed"], EDXExtractPatterns(
  resource["ed.filepath"],
  "^(.+)/(?P<filename>[^/]+)$"
))
set(attributes["filename"], cache["parsed"]["filename"])
```

---

## Overview

EDXExtractPatterns enables dynamic pattern extraction using regular expressions with named capture groups.

**Key Features:**
- Named capture groups support
- Returns map of extracted values
- Go regex syntax (RE2)
- Efficient pattern matching

**Common Use Cases:**
- Parse file paths
- Extract structured data from logs
- Parse URLs
- Extract version numbers
- Parse custom log formats

---

## Parameters

### input
- **Type**: string
- **Required**: Yes
- **Description**: String to parse
- **Can be**: Log message, file path, URL, etc.

### pattern
- **Type**: string
- **Required**: Yes
- **Description**: Regular expression with named groups
- **Syntax**: Go regex (RE2)
- **Named Groups**: `(?P<name>pattern)`
- **Example**: `"^(?P<protocol>https?)://(?P<domain>[^/]+)(?P<path>/.*)?$"`

---

## Return Value

Returns `map[string]string` containing named capture groups.

**Behavior:**
- Only named groups returned (unnamed groups ignored)
- If pattern doesn't match → empty map
- Missing groups → not included in map
- All values are strings

---

## Examples

### Example 1: Parse File Path
```yaml
processors:
  - name: parse-filepath
    transform:
      statements:
        - set(cache["parsed"], EDXExtractPatterns(
            resource["ed.filepath"],
            "^(?P<directory>.+)/(?P<filename>[^/]+)\\.(?P<extension>\\w+)$"
          ))
        - set(attributes["dir"], cache["parsed"]["directory"])
        - set(attributes["file"], cache["parsed"]["filename"])
        - set(attributes["ext"], cache["parsed"]["extension"])
```

### Example 2: Parse URL
```yaml
processors:
  - name: parse-url
    transform:
      statements:
        - set(cache["url_parts"], EDXExtractPatterns(
            attributes["url"],
            "^(?P<protocol>https?)://(?P<domain>[^:/?]+)(:(?P<port>\\d+))?(?P<path>/[^?]*)?(\\?(?P<query>.*))?$"
          ))
        - set(attributes["domain"], cache["url_parts"]["domain"])
        - set(attributes["path"], cache["url_parts"]["path"])
```

### Example 3: Extract Version Number
```yaml
processors:
  - name: parse-version
    transform:
      statements:
        - set(cache["version"], EDXExtractPatterns(
            attributes["user_agent"],
            "Version/(?P<major>\\d+)\\.(?P<minor>\\d+)\\.(?P<patch>\\d+)"
          ))
        - set(attributes["version_major"], cache["version"]["major"])
```

### Example 4: Parse Log Line
```yaml
processors:
  - name: parse-apache-log
    transform:
      statements:
        - set(cache["log"], EDXExtractPatterns(
            body,
            "^(?P<ip>[\\d.]+) - - \\[(?P<timestamp>[^\\]]+)\\] \"(?P<method>\\w+) (?P<path>[^ ]+) HTTP/[^\"]+\" (?P<status>\\d+)"
          ))
        - set(attributes["client_ip"], cache["log"]["ip"])
        - set(attributes["http.method"], cache["log"]["method"])
        - set(attributes["http.status"], cache["log"]["status"])
```

### Example 5: Extract Email Parts
```yaml
processors:
  - name: parse-email
    transform:
      statements:
        - set(cache["email"], EDXExtractPatterns(
            attributes["email"],
            "^(?P<user>[^@]+)@(?P<domain>.+)$"
          ))
        - set(attributes["email_user"], cache["email"]["user"])
        - set(attributes["email_domain"], cache["email"]["domain"])
```

### Example 6: Parse JSON Error Message
```yaml
processors:
  - name: parse-error
    transform:
      statements:
        - set(cache["error"], EDXExtractPatterns(
            attributes["error_message"],
            "Error: (?P<code>\\d+) - (?P<message>.+)"
          ))
        - set(attributes["error_code"], cache["error"]["code"])
        - set(attributes["error_msg"], cache["error"]["message"])
```

### Example 7: Extract IPv4 Address
```yaml
processors:
  - name: extract-ip
    transform:
      statements:
        - set(cache["ip_match"], EDXExtractPatterns(
            body,
            "src_ip=(?P<ip>\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}\\.\\d{1,3})"
          ))
        - set(attributes["source_ip"], cache["ip_match"]["ip"])
```

### Example 8: Parse Docker Image Tag
```yaml
processors:
  - name: parse-image
    transform:
      statements:
        - set(cache["image"], EDXExtractPatterns(
            attributes["docker_image"],
            "^(?P<registry>[^/]+)/(?P<repo>[^:]+):(?P<tag>.+)$"
          ))
        - set(attributes["registry"], cache["image"]["registry"])
        - set(attributes["repository"], cache["image"]["repo"])
        - set(attributes["tag"], cache["image"]["tag"])
```

---

## Common Use Cases

### 1. Log Parsing
```yaml
processors:
  - name: structured-log-parse
    transform:
      statements:
        - set(cache["parsed"], EDXExtractPatterns(
            body,
            "\\[(?P<timestamp>[^\\]]+)\\] (?P<level>\\w+): (?P<message>.+)"
          ))
        - set(attributes["log.level"], cache["parsed"]["level"])
        - set(attributes["log.message"], cache["parsed"]["message"])
```

### 2. Metric Name Parsing
```yaml
processors:
  - name: parse-metric
    transform:
      statements:
        - set(cache["metric"], EDXExtractPatterns(
            attributes["metric_name"],
            "^(?P<namespace>[^.]+)\\.(?P<subsystem>[^.]+)\\.(?P<name>.+)$"
          ))
```

### 3. Extract Build Info
```yaml
processors:
  - name: parse-build
    transform:
      statements:
        - set(cache["build"], EDXExtractPatterns(
            attributes["version"],
            "^v(?P<version>\\d+\\.\\d+\\.\\d+)-(?P<commit>[a-f0-9]{7})$"
          ))
```

---

## Validation Rules

1. **Valid Regex**: Pattern must be valid Go regex
2. **Named Groups**: Use `(?P<name>...)` syntax
3. **Input Required**: Input string must be provided
4. **Return Always**: Always returns map (empty if no match)

---

## Common Pitfalls

### 1. Forgetting Named Groups
```yaml
# WRONG: Unnamed groups not returned
set(cache["parsed"], EDXExtractPatterns(body, "(\\d+)-(\\w+)"))
# Returns empty map!

# CORRECT: Use named groups
set(cache["parsed"], EDXExtractPatterns(body, "(?P<id>\\d+)-(?P<name>\\w+)"))
```

### 2. Not Escaping Special Characters
```yaml
# WRONG: Dot matches any character
set(cache["ip"], EDXExtractPatterns(body, "(?P<ip>\\d+.\\d+.\\d+.\\d+)"))

# CORRECT: Escape dots
set(cache["ip"], EDXExtractPatterns(body, "(?P<ip>\\d+\\.\\d+\\.\\d+\\.\\d+)"))
```

### 3. Not Handling No Match
```yaml
# RISKY: Accessing non-existent key
set(cache["parsed"], EDXExtractPatterns(body, "(?P<user>\\w+)"))
set(attributes["user"], cache["parsed"]["user"])  # May be nil!

# BETTER: Check existence
set(cache["parsed"], EDXExtractPatterns(body, "(?P<user>\\w+)"))
set(attributes["user"], cache["parsed"]["user"] or "unknown")
```

---

## Best Practices

### 1. Use Descriptive Names
```yaml
set(cache["url"], EDXExtractPatterns(
  input,
  "(?P<protocol>https?)://(?P<hostname>[^/]+)(?P<path>/.*)"
))
```

### 2. Combine with EDXCoalesce
```yaml
set(cache["parsed"], EDXExtractPatterns(body, "user=(?P<user>\\w+)"))
set(attributes["user"], EDXCoalesce(cache["parsed"]["user"], "anonymous"))
```

### 3. Test Patterns Thoroughly
```yaml
# Use tools like regex101.com to test patterns before deployment
```

---

## Performance Considerations

- **Regex Compilation**: Patterns compiled at runtime
- **Complexity**: Avoid overly complex patterns
- **Backtracking**: RE2 prevents catastrophic backtracking
- **Caching**: Consider caching parsed results

---

## Related Functions

- **Grok Processor**: Use for predefined patterns
- **EDXParseKeyValue**: Simpler key-value parsing
- **Split()**: Use for simple delimiter-based parsing

---

## Cross-References

- [MASTER_INDEX.md](../../MASTER_INDEX.md#edxextractpatterns)
- [Grok Processor](../../processors/grok.md)

---

## Notes

- **Version**: Introduced in v1.22.0
- **Regex Engine**: Uses Go's RE2
- **Unicode**: Full Unicode support
- **Thread Safety**: Thread-safe
