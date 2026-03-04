# EDXParseKeyValue - Parse Key-Value Pairs

**Function Type**: EDX Converter
**Purpose**: Parses delimited key-value pair strings into structured maps with type conversion and merge strategies
**Signature**: `EDXParseKeyValue(target, kvDelimiter, delimiter, shouldConvert, mergeStrategy) -> map[string]any`
**Return Type**: map[string]any (pcommon.Map)
**Minimum Agent Version**: v1.22.0
**Source**: https://docs.edgedelta.com/ottl-extensions/

---

## Quick Copy

```yaml
# Parse query string
set(cache["params"], EDXParseKeyValue(
  attributes["query_string"],
  "=",
  "&",
  true,
  "last"
))
```

---

## Overview

EDXParseKeyValue transforms delimited key-value strings into structured maps. Essential for:
- Parsing URL query strings
- Processing log formats (key1=val1 key2=val2)
- Parsing configuration strings
- Extracting structured data from unstructured text

**Features:**
- Custom delimiters for keys and pairs
- Automatic type conversion (string→int/float/bool)
- Multiple merge strategies for duplicate keys
- Quote handling (single/double)
- Whitespace trimming

---

## Parameters

### target
- **Type**: string
- **Required**: Yes
- **Description**: String to parse
- **Example**: `"key1=value1&key2=value2"`

### kvDelimiter
- **Type**: string
- **Required**: No
- **Default**: `"="`
- **Description**: Delimiter between keys and values
- **Common**: `"="`, `":"`, `"->"`

### delimiter
- **Type**: string
- **Required**: No
- **Default**: `" "` (space)
- **Description**: Delimiter between pairs
- **Common**: `"&"`, `","`, `" "`, `"\n"`

### shouldConvert
- **Type**: bool
- **Required**: No
- **Default**: false
- **Description**: Auto-convert values to int/float/bool
- **Behavior**:
  - `true`: `"123"` → `int64(123)`, `"true"` → `bool(true)`
  - `false`: All values remain strings

### mergeStrategy
- **Type**: string
- **Required**: No
- **Default**: `"last"`
- **Description**: How to handle duplicate keys
- **Valid Values**:
  - `"first"`: Keep first occurrence
  - `"last"`: Keep last occurrence (default)
  - `"append"`: Collect all values as array (deduplicates)
  - `"concat"`: Join values with ", " separator (deduplicates)
  - `"indexed"`: Create indexed keys (key_0, key_1, key_2)

---

## Return Value

Returns `pcommon.Map` containing parsed key-value pairs.

**Conversion Rules (when shouldConvert=true):**
- Integers: `"123"` → `int64(123)`
- Floats: `"3.14"` → `float64(3.14)`
- Booleans: `"true"`/`"false"` → `bool`
- Quoted strings: `"\"text\""` → `"text"` (quotes removed)
- Other: Remains as string

---

## Examples

### Example 1: Parse URL Query String
```yaml
processors:
  - name: parse-query
    transform:
      statements:
        - set(cache["params"], EDXParseKeyValue(
            attributes["query_string"],
            "=",
            "&",
            true,
            "last"
          ))
        - set(attributes["search_term"], cache["params"]["q"])
        - set(attributes["page"], cache["params"]["page"])
```

### Example 2: Parse Log Line
```yaml
processors:
  - name: parse-log
    transform:
      statements:
        - set(cache["fields"], EDXParseKeyValue(
            body,
            "=",
            " ",
            true,
            "last"
          ))
        - set(attributes["user"], cache["fields"]["user"])
        - set(attributes["duration_ms"], cache["fields"]["duration"])
```

### Example 3: Parse with Type Conversion
```yaml
processors:
  - name: typed-parse
    transform:
      statements:
        - set(cache["data"], EDXParseKeyValue(
            "age=25 height=5.9 active=true name=John",
            "=",
            " ",
            true,
            "last"
          ))
        # age → int64(25)
        # height → float64(5.9)
        # active → bool(true)
        # name → "John"
```

### Example 4: Merge Strategy - Append
```yaml
processors:
  - name: append-duplicates
    transform:
      statements:
        - set(cache["parsed"], EDXParseKeyValue(
            "tag=prod tag=web tag=api",
            "=",
            " ",
            false,
            "append"
          ))
        # cache["parsed"]["tag"] = ["prod", "web", "api"]
```

### Example 5: Merge Strategy - Concat
```yaml
processors:
  - name: concat-duplicates
    transform:
      statements:
        - set(cache["parsed"], EDXParseKeyValue(
            "label=a label=b label=c",
            "=",
            " ",
            false,
            "concat"
          ))
        # cache["parsed"]["label"] = "a, b, c"
```

### Example 6: Merge Strategy - Indexed
```yaml
processors:
  - name: indexed-duplicates
    transform:
      statements:
        - set(cache["parsed"], EDXParseKeyValue(
            "val=x val=y val=z",
            "=",
            " ",
            false,
            "indexed"
          ))
        # cache["parsed"]["val_0"] = "x"
        # cache["parsed"]["val_1"] = "y"
        # cache["parsed"]["val_2"] = "z"
```

### Example 7: Custom Delimiters
```yaml
processors:
  - name: custom-delimiters
    transform:
      statements:
        - set(cache["parsed"], EDXParseKeyValue(
            "key1:value1|key2:value2|key3:value3",
            ":",
            "|",
            false,
            "last"
          ))
```

### Example 8: Parse Cookie String
```yaml
processors:
  - name: parse-cookies
    transform:
      statements:
        - set(cache["cookies"], EDXParseKeyValue(
            attributes["cookie"],
            "=",
            "; ",
            false,
            "last"
          ))
        - set(attributes["session_id"], cache["cookies"]["session_id"])
```

---

## Common Use Cases

### 1. HTTP Request Parsing
```yaml
processors:
  - name: parse-http
    transform:
      statements:
        - set(cache["query"], EDXParseKeyValue(
            attributes["http.query"],
            "=",
            "&",
            true,
            "last"
          ))
        - set(cache["headers"], EDXParseKeyValue(
            attributes["http.headers_str"],
            ": ",
            "\n",
            false,
            "last"
          ))
```

### 2. Configuration String Parsing
```yaml
processors:
  - name: parse-config
    transform:
      statements:
        - set(cache["config"], EDXParseKeyValue(
            EDXEnv("APP_CONFIG", ""),
            "=",
            ",",
            true,
            "last"
          ))
        - set(attributes["timeout"], cache["config"]["timeout"])
        - set(attributes["retries"], cache["config"]["retries"])
```

### 3. Extract Tags from Logs
```yaml
processors:
  - name: extract-tags
    transform:
      statements:
        - set(cache["tags"], EDXParseKeyValue(
            attributes["tags_string"],
            "=",
            ",",
            false,
            "last"
          ))
        - edx_map_keys(attributes, cache["tags"], cache["tags"], "upsert")
```

---

## Validation Rules

1. **Target Required**: Input string must be provided
2. **Delimiter Validation**: Delimiters cannot be equal
3. **Delimiter Cannot Be Whitespace**: kvDelimiter cannot be space
4. **Empty Delimiters Invalid**: Delimiters must be non-empty
5. **Format Validation**: Each pair must split into exactly 2 parts

---

## Common Pitfalls

### 1. Equal Delimiters
```yaml
# WRONG: Same delimiter for kv and pairs
set(cache["parsed"], EDXParseKeyValue("a=b=c=d", "=", "=", false, "last"))
# Error: delimiters cannot be equal

# CORRECT: Different delimiters
set(cache["parsed"], EDXParseKeyValue("a=b&c=d", "=", "&", false, "last"))
```

### 2: Type Conversion Surprises
```yaml
# With shouldConvert=true
set(cache["parsed"], EDXParseKeyValue("id=007", "=", " ", true, "last"))
# cache["parsed"]["id"] = int64(7) NOT "007"!

# To keep as string:
set(cache["parsed"], EDXParseKeyValue("id=007", "=", " ", false, "last"))
# cache["parsed"]["id"] = "007"
```

### 3. Missing Pairs Cause Errors
```yaml
# WRONG: Malformed pair
set(cache["parsed"], EDXParseKeyValue("key1=val1&key2", "=", "&", false, "last"))
# Error: cannot split "key2" into 2 items

# Data must be well-formed
```

---

## Best Practices

### 1. Enable Type Conversion for Numeric Data
```yaml
# When parsing config/query params with numbers
set(cache["params"], EDXParseKeyValue(input, "=", "&", true, "last"))
```

### 2. Choose Appropriate Merge Strategy
```yaml
# For tags/labels that can repeat
set(cache["tags"], EDXParseKeyValue(input, "=", " ", false, "append"))

# For unique keys
set(cache["params"], EDXParseKeyValue(input, "=", "&", false, "last"))
```

### 3. Handle Errors Gracefully
```yaml
set(cache["parsed"], EDXParseKeyValue(input, "=", "&", true, "last") or map[string]any{})
set(cache["success"], Len(cache["parsed"]) > 0)
```

---

## Performance Considerations

- **Quote Handling**: Slight overhead for quoted values
- **Type Conversion**: Additional processing when enabled
- **Merge Strategies**: "indexed" and "append" allocate more memory
- **String Splitting**: Performance degrades with many pairs

**Tips:**
- Use "last" strategy for best performance
- Disable type conversion if not needed
- Cache parsed results

---

## Related Functions

- **EDXExtractPatterns**: For complex parsing with regex
- **Split()**: For simple delimiter splitting
- **ParseJSON()**: For JSON data

---

## Cross-References

- [MASTER_INDEX.md](../../MASTER_INDEX.md#edxparsekeyvalue)
- [EDXExtractPatterns](./edxextractpatterns.md)

---

## Notes

- **Version**: Introduced in v1.22.0
- **Quote Handling**: Supports single and double quotes
- **Whitespace**: Trims leading/trailing whitespace from values
- **Empty Values**: Preserved in output
- **Thread Safety**: Thread-safe
