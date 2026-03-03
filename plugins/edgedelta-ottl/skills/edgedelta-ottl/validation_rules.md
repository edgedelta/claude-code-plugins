# OTTL Syntax Validation Rules

**Purpose**: Strict syntax enforcement for OTTL statements in EdgeDelta pipelines
**Last Updated**: 2025-10-27
**Enforcement**: These rules MUST be followed or compilation will fail

---

## Critical Validation Rules

These rules are **non-negotiable**. Violations will cause OTTL parser errors.

---

## 1. Lowercase Operator Requirement (CRITICAL)

**Rule**: All logical operators MUST be lowercase

**Valid**: `and`, `or`, `not`, `where`
**Invalid**: `AND`, `OR`, `NOT`, `WHERE`, `And`, `Or`, `Not`, `Where`

**Error Pattern**: `WHERE`, `AND`, `OR`, `NOT` in OTTL statements
**Fix**: Replace with lowercase equivalents

**Examples**:
```yaml
# ❌ INVALID - Will cause parser error
set(attributes["match"], true) WHERE attributes["level"] == "ERROR"
set(cache["x"], true) where attributes["a"] == "1" AND attributes["b"] == "2"

# ✅ VALID
set(attributes["match"], true) where attributes["level"] == "ERROR"
set(cache["x"], true) where attributes["a"] == "1" and attributes["b"] == "2"
```

**Detection Regex**: `\b(WHERE|AND|OR|NOT)\b`
**Automated Fix**: Replace uppercase with lowercase

**Why This Matters**: OTTL parser is case-sensitive. Uppercase operators will fail with `syntax error near WHERE` or `undefined identifier AND`.

---

## 2. Single-Line Condition Requirement

**Rule**: All OTTL conditions must fit on a single line in YAML

**Error Pattern**: Multi-line where clauses
**Fix**: Combine into single line with `and`/`or`

**Examples**:
```yaml
# ❌ INVALID - Multi-line condition
set(cache["match"], true) where
  attributes["level"] == "ERROR" and
  attributes["source"] == "api"

# ❌ INVALID - YAML multi-line string
set(cache["match"], true) where >
  attributes["level"] == "ERROR" and
  attributes["source"] == "api"

# ✅ VALID - Single line
set(cache["match"], true) where attributes["level"] == "ERROR" and attributes["source"] == "api"

# ✅ VALID - Long line with proper formatting
set(cache["match"], true) where attributes["level"] == "ERROR" and attributes["source"] == "api" and attributes["status_code"] >= 400
```

**Why This Matters**: YAML multi-line parsing can introduce unexpected whitespace or newlines that break OTTL syntax.

---

## 3. String Value Quoting

**Rule**: String values MUST be enclosed in double quotes

**Valid**: `"value"`, `"ERROR"`, `"true"` (string literal)
**Invalid**: `value`, `ERROR`, `true` (as string - boolean true/false is exception)

**Error Pattern**: Unquoted strings in comparisons or assignments
**Fix**: Add double quotes

**Examples**:
```yaml
# ❌ INVALID
set(attributes["status"], error)
set(cache["match"], true) where attributes["level"] == ERROR
set(attributes["env"], production)

# ✅ VALID
set(attributes["status"], "error")
set(cache["match"], true) where attributes["level"] == "ERROR"
set(attributes["env"], "production")

# ✅ VALID - Booleans are NOT quoted
set(attributes["enabled"], true)
set(cache["match"], false)

# ✅ VALID - Numbers are NOT quoted
set(attributes["count"], 42)
set(attributes["ratio"], 3.14)
```

**Why This Matters**: Unquoted strings are treated as identifiers, causing `undefined identifier` errors.

---

## 4. Field Access Bracket Notation

**Rule**: Field access MUST use bracket notation, not dot notation

**Valid**: `attributes["field"]`, `cache["key"]`, `resource["name"]`
**Invalid**: `attributes.field`, `cache.key`, `resource.name`

**Error Pattern**: Dot notation in field paths
**Fix**: Convert to bracket notation

**Examples**:
```yaml
# ❌ INVALID
set(attributes["user"], cache.parsed.user.id)
set(cache["host"], resource.host.name)
delete_key(attributes, attributes.temp_field)

# ✅ VALID
set(attributes["user"], cache["parsed"]["user"]["id"])
set(cache["host"], resource["host"]["name"])
delete_key(attributes, "temp_field")

# ✅ VALID - Nested access
set(attributes["user_email"], cache["parsed"]["users"][0]["email"])
```

**Why This Matters**: OTTL does not support dot notation for field access. Dots cause `unexpected '.'` syntax errors.

---

## 5. Boolean Value Casing

**Rule**: Boolean values MUST be lowercase without quotes

**Valid**: `true`, `false`
**Invalid**: `"true"`, `"false"`, `TRUE`, `FALSE`, `True`, `False`

**Examples**:
```yaml
# ❌ INVALID
set(attributes["enabled"], "true")
set(cache["match"], TRUE)
set(attributes["is_active"], True)

# ✅ VALID
set(attributes["enabled"], true)
set(cache["match"], false)

# ⚠️ IMPORTANT - String literal "true" vs boolean true
set(attributes["text"], "true")  # ✅ String value "true"
set(attributes["bool"], true)    # ✅ Boolean value true
```

**Why This Matters**:
- `"true"` is a string, not a boolean
- `TRUE` causes `undefined identifier TRUE` error
- Type mismatches can break conditions

---

## 6. Body Field Decoding

**Rule**: body field MUST be decoded before string operations

**Error Pattern**: String functions applied directly to body
**Fix**: Wrap with `Decode(body, "utf-8")`

**Examples**:
```yaml
# ❌ INVALID - body is byte array
set(cache["json"], ParseJSON(body))
set(cache["match"], IsMatch(body, "pattern"))
set(attributes["message"], Substring(body, 0, 100))

# ✅ VALID
set(cache["json"], ParseJSON(Decode(body, "utf-8")))
set(cache["match"], IsMatch(Decode(body, "utf-8"), "pattern"))
set(attributes["message"], Substring(Decode(body, "utf-8"), 0, 100))

# ✅ VALID - Reuse decoded value
set(cache["decoded"], Decode(body, "utf-8"))
set(cache["json"], ParseJSON(cache["decoded"]))
set(cache["match"], IsMatch(cache["decoded"], "ERROR.*"))
```

**Why This Matters**:
- `body` is a byte array (pcommon.Value of type Bytes)
- String functions expect string type
- Type mismatch causes `cannot convert` errors

**Supported Encodings**: `"utf-8"`, `"ascii"`, `"utf-16"`, `"base64"`

---

## 7. Field Existence Checks

**Rule**: Check field existence using `!= nil`, not boolean conversion

**Examples**:
```yaml
# ❌ INVALID - May not work as expected
set(cache["has_user"], attributes["user_id"])

# ❌ INVALID - No IsNull function in OTTL
set(cache["has_user"], IsNull(attributes["user_id"]))

# ✅ VALID
set(cache["has_user"], true) where attributes["user_id"] != nil
delete_key(attributes, "temp") where attributes["temp"] != nil

# ✅ VALID - Check before access
set(cache["user"], attributes["user_id"]) where attributes["user_id"] != nil
```

**Why This Matters**:
- Accessing non-existent fields can cause errors
- `!= nil` is the standard OTTL existence check
- Prevents runtime errors from missing fields

---

## 8. Nil Comparison Syntax

**Rule**: Use `== nil` or `!= nil` for null checks

**Valid**: `attributes["field"] != nil`, `cache["key"] == nil`
**Invalid**: `attributes["field"] != null`, `IsNull(attributes["field"])`

**Examples**:
```yaml
# ❌ INVALID - 'null' is not recognized
set(cache["empty"], true) where attributes["value"] == null

# ❌ INVALID - No IsNull function
set(cache["empty"], true) where IsNull(attributes["value"])

# ✅ VALID
set(cache["empty"], true) where attributes["value"] == nil
set(cache["has_value"], true) where attributes["value"] != nil
delete_key(cache, "temp") where cache["temp"] == nil
```

**Why This Matters**: OTTL uses `nil` (Go-style), not `null` (JSON/JavaScript-style).

---

## 9. Function Argument Syntax

**Rule**: Function arguments must follow strict syntax rules

**Requirements**:
- No trailing commas
- Correct number of arguments
- Proper type matching
- Valid escape sequences in strings

**Examples**:
```yaml
# ❌ INVALID - Trailing comma
set(cache["value"], Substring(cache["text"], 0, 10,))

# ❌ INVALID - Wrong argument count
set(cache["lower"], ToLower(cache["text"], "extra_arg"))

# ❌ INVALID - Invalid escape sequence
set(cache["match"], IsMatch(cache["text"], "\d+\s+\w+"))

# ✅ VALID
set(cache["value"], Substring(cache["text"], 0, 10))
set(cache["lower"], ToLower(cache["text"]))
set(cache["match"], IsMatch(cache["text"], "\\d+\\s+\\w+"))

# ✅ VALID - Complex nested functions
set(cache["result"], ToLower(Substring(Decode(body, "utf-8"), 0, 100)))
```

**Why This Matters**: Parser errors from malformed function calls are difficult to debug.

---

## 10. YAML String Escaping

**Rule**: Properly escape special characters in YAML strings

**Special Cases**:
- Backslashes must be doubled: `\\`
- Quotes in strings must be escaped: `\"`
- Regex patterns need double escaping: `\\d`, `\\s`, `\\w`

**Examples**:
```yaml
# ❌ INVALID - Single backslash in regex
set(cache["match"], IsMatch(cache["text"], "\d+"))

# ❌ INVALID - Unescaped quotes
set(attributes["message"], "Error: "invalid"")

# ✅ VALID - Doubled backslashes for regex
set(cache["match"], IsMatch(cache["text"], "\\d+"))

# ✅ VALID - Escaped quotes
set(attributes["message"], "Error: \"invalid\"")

# ✅ VALID - Complex regex pattern
set(cache["match"], IsMatch(Decode(body, "utf-8"), "ERROR\\s+\\[\\w+\\]\\s+.*"))

# ✅ ALTERNATIVE - Use single quotes in YAML (no escaping needed inside)
set(cache["match"], IsMatch(Decode(body, "utf-8"), 'ERROR\s+\[\w+\]\s+.*'))
```

**Why This Matters**: YAML processes escape sequences before OTTL sees them. Single backslashes get consumed.

---

## 11. Cache vs Attributes Usage

**Rule**: Understand the difference between cache and attributes

**cache**: Temporary processing variables, not exported
**attributes**: Persistent fields, exported to destinations

**Examples**:
```yaml
# ✅ VALID - Use cache for intermediate values
set(cache["decoded"], Decode(body, "utf-8"))
set(cache["json"], ParseJSON(cache["decoded"]))
set(attributes["user_id"], cache["json"]["user"]["id"])

# ✅ VALID - Use attributes for final values
set(attributes["log_level"], cache["json"]["level"])
set(attributes["timestamp"], cache["json"]["timestamp"])

# ⚠️ IMPORTANT - cache values are NOT sent to destinations
set(cache["temp_match"], true)  # Only used in pipeline, not exported
set(attributes["matched"], true)  # Exported to Splunk, S3, etc.
```

**Why This Matters**:
- cache is scoped to pipeline processing
- attributes are exported to destinations
- Confusing them leads to missing data

---

## 12. EDXRedis Field Naming

**Rule**: EDXRedis hash fields MUST match exact key names

**Error Pattern**: Mismatched field names in EDXRedisHashSet and EDXRedisHashGet
**Fix**: Use consistent field naming

**Examples**:
```yaml
# ❌ INVALID - Field name mismatch
- statements:
  - EDXRedisHashSet(cache, "alerts", attributes["alert_id"], "count", 1)
  - set(cache["alert_count"], EDXRedisHashGet(cache, "alerts", attributes["alert_id"], "counter"))

# ✅ VALID - Matching field names
- statements:
  - EDXRedisHashSet(cache, "alerts", attributes["alert_id"], "count", 1)
  - set(cache["alert_count"], EDXRedisHashGet(cache, "alerts", attributes["alert_id"], "count"))

# ✅ VALID - Multiple fields
- statements:
  - EDXRedisHashSet(cache, "alerts", attributes["alert_id"], "count", 1)
  - EDXRedisHashSet(cache, "alerts", attributes["alert_id"], "first_seen", Time())
  - set(cache["count"], EDXRedisHashGet(cache, "alerts", attributes["alert_id"], "count"))
  - set(cache["first_seen"], EDXRedisHashGet(cache, "alerts", attributes["alert_id"], "first_seen"))
```

**Why This Matters**: Field name mismatches cause silent nil returns or missing data.

---

## 13. Type Conversion Rules

**Rule**: Explicit type conversion when mixing types

**Available Conversions**:
- `Int()` - Convert to integer
- `String()` - Convert to string
- `Double()` - Convert to float

**Examples**:
```yaml
# ❌ INVALID - Type mismatch in comparison
set(cache["match"], true) where attributes["status_code"] == "500"

# ✅ VALID - Proper type matching
set(cache["match"], true) where attributes["status_code"] == 500
set(cache["match"], true) where String(attributes["status_code"]) == "500"

# ✅ VALID - String to int conversion
set(cache["code"], Int(attributes["status_str"]))

# ✅ VALID - Int to string conversion
set(attributes["code_str"], String(attributes["status_code"]))
```

**Why This Matters**: Type mismatches cause comparison failures or runtime errors.

---

## 14. EDX Function Naming Convention (CRITICAL)

**Rule**: EDX function names follow strict naming conventions based on function type

**Naming Pattern**:
- **Standard OTTL Type Checkers**: CamelCase (e.g., `IsMap`, `IsString`, `IsBool`, `IsInt`)
- **Standard OTTL Converters**: CamelCase (e.g., `ParseJSON`, `ToUpperCase`, `Concat`)
- **Standard OTTL Editors**: snake_case (e.g., `set`, `delete_key`, `keep_keys`)
- **EDX Converters** (return values): `EDX[Name]` in CamelCase (e.g., `EDXCoalesce`, `EDXIfElse`, `EDXRedis`)
- **EDX Editors** (modify in place): `edx_[name]` in snake_case (e.g., `edx_code`, `edx_delete_keys`, `edx_keep_keys`)

**Error Pattern**: Using incorrect casing for functions
**Fix**: Match the correct naming convention based on function type

**Examples**:
```yaml
# ❌ INVALID - Wrong casing for EDX editor function
statements: |-
  EDXCode("item['attributes']['value'] = 'test';")

# ✅ VALID - Correct snake_case for EDX editor
statements: |-
  edx_code("item['attributes']['value'] = 'test';")

# ✅ VALID - Standard type checkers use CamelCase
set(cache["is_map"], true) where IsMap(attributes)
set(cache["is_string"], true) where IsString(attributes["value"])

# ✅ VALID - EDX Converters use CamelCase
set(attributes["result"], EDXIfElse(condition, value_if_true, value_if_false))
set(cache["data"], EDXRedis({"url": "redis://localhost"}, [{"command": "GET", "key": "mykey"}]))
set(attributes["default"], EDXCoalesce(attr1, attr2, "fallback"))

# ✅ VALID - EDX Editors use snake_case
edx_delete_keys(attributes, ["temp", "debug"])
edx_keep_matching_keys(attributes, ["^service\\.", "^host\\."])
```

**Common Errors**:
| ❌ Wrong | ✅ Correct | Type |
|---------|-----------|------|
| `EDXCode` | `edx_code` | EDX Editor |
| `edx_if_else` | `EDXIfElse` | EDX Converter |
| `is_map` | `IsMap` | Standard Type Checker |
| `edx_redis` | `EDXRedis` | EDX Converter |

**Why This Matters**: Function names are case-sensitive. Using the wrong casing results in `undefined function` errors. The naming convention helps identify whether a function returns a value (Converter) or modifies data in place (Editor).

---

## Validation Checklist

Before deploying OTTL statements, verify:

- [ ] All logical operators are lowercase (`and`, `or`, `not`, `where`)
- [ ] All conditions fit on single lines
- [ ] String values are in double quotes
- [ ] Field access uses bracket notation
- [ ] Boolean values are lowercase without quotes
- [ ] body field is decoded before string operations
- [ ] Field existence checked with `!= nil`
- [ ] No trailing commas in function calls
- [ ] Nested functions have correct parentheses matching
- [ ] YAML indentation is correct (spaces, not tabs)
- [ ] Regex patterns use double backslashes (`\\d`, `\\s`, `\\w`)
- [ ] cache vs attributes used appropriately
- [ ] EDXRedis field names are consistent
- [ ] Type conversions applied where needed
- [ ] EDX function names follow naming convention (EDX Converters: EDXCamelCase, EDX Editors: edx_snake_case)
- [ ] Standard OTTL functions follow naming convention (Type Checkers & Converters: CamelCase, Editors: snake_case)

---

## Common Error Patterns and Fixes

### Error 1: Uppercase WHERE
**Error Message**: `syntax error near WHERE`
**Pattern**: `WHERE`
**Fix**: Replace with `where`

```yaml
# Before
set(attributes["match"], true) WHERE attributes["level"] == "ERROR"

# After
set(attributes["match"], true) where attributes["level"] == "ERROR"
```

---

### Error 2: Multi-line condition
**Error Message**: `unexpected newline`
**Pattern**: Newline within where clause
**Fix**: Combine into single line

```yaml
# Before
set(cache["match"], true) where
  attributes["level"] == "ERROR" and
  attributes["source"] == "api"

# After
set(cache["match"], true) where attributes["level"] == "ERROR" and attributes["source"] == "api"
```

---

### Error 3: Unquoted string
**Error Message**: `undefined identifier ERROR`
**Pattern**: Bare string without quotes
**Fix**: Add double quotes

```yaml
# Before
set(cache["match"], true) where attributes["level"] == ERROR

# After
set(cache["match"], true) where attributes["level"] == "ERROR"
```

---

### Error 4: Dot notation
**Error Message**: `unexpected '.'`
**Pattern**: `field.subfield`
**Fix**: Convert to `field["subfield"]`

```yaml
# Before
set(attributes["user"], cache.parsed.user.id)

# After
set(attributes["user"], cache["parsed"]["user"]["id"])
```

---

### Error 5: Body not decoded
**Error Message**: `type mismatch: expected string, got []byte`
**Pattern**: String function on body
**Fix**: Wrap body with `Decode(body, "utf-8")`

```yaml
# Before
set(cache["json"], ParseJSON(body))

# After
set(cache["json"], ParseJSON(Decode(body, "utf-8")))
```

---

### Error 6: Uppercase boolean
**Error Message**: `undefined identifier TRUE`
**Pattern**: `TRUE`, `FALSE`
**Fix**: Replace with `true`, `false`

```yaml
# Before
set(cache["match"], TRUE)

# After
set(cache["match"], true)
```

---

### Error 7: Single backslash in regex
**Error Message**: `invalid escape sequence` or pattern doesn't match
**Pattern**: `\d`, `\s`, `\w` in regex
**Fix**: Double backslashes

```yaml
# Before
set(cache["match"], IsMatch(cache["text"], "\d+"))

# After
set(cache["match"], IsMatch(cache["text"], "\\d+"))
```

---

### Error 8: Null instead of nil
**Error Message**: `undefined identifier null`
**Pattern**: `== null`, `!= null`
**Fix**: Replace with `== nil`, `!= nil`

```yaml
# Before
set(cache["empty"], true) where attributes["value"] == null

# After
set(cache["empty"], true) where attributes["value"] == nil
```

---

### Error 9: Missing field check
**Error Message**: Runtime error or unexpected nil
**Pattern**: Direct field access without existence check
**Fix**: Add `!= nil` condition

```yaml
# Before
set(attributes["user"], cache["parsed"]["user"]["id"])

# After
set(attributes["user"], cache["parsed"]["user"]["id"]) where cache["parsed"]["user"]["id"] != nil
```

---

### Error 10: Type mismatch in comparison
**Error Message**: `type mismatch in comparison`
**Pattern**: Comparing different types
**Fix**: Add type conversion

```yaml
# Before
set(cache["match"], true) where attributes["status_code"] == "500"

# After
set(cache["match"], true) where attributes["status_code"] == 500
# OR
set(cache["match"], true) where String(attributes["status_code"]) == "500"
```

---

### Error 11: Incorrect Function Casing
**Error Message**: `undefined function EDXCode` or `undefined function is_map`
**Pattern**: Wrong casing for functions
**Fix**: Use correct naming convention

```yaml
# Before - Wrong casing for EDX editor
statements: |-
  EDXCode("item['attributes']['value'] = 'test';")

# After - Correct snake_case for EDX editor
statements: |-
  edx_code("item['attributes']['value'] = 'test';")

# Before - Wrong casing for standard type checker
set(cache["is_map"], true) where is_map(attributes)

# After - Correct CamelCase for standard type checker
set(cache["is_map"], true) where IsMap(attributes)

# Reminder:
# ✅ EDXIfElse (EDX Converter - EDXCamelCase)
# ✅ edx_code (EDX Editor - edx_snake_case)
# ✅ IsMap (Standard Type Checker - CamelCase)
# ✅ set (Standard Editor - snake_case)
```

---

## Automated Validation Script

Use this bash script to validate OTTL syntax before deployment:

```bash
#!/bin/bash
# ottl-validate.sh - Validate OTTL syntax in EdgeDelta pipeline YAML files

validate_ottl() {
    local file="$1"
    local errors=0

    echo "Validating OTTL syntax in: $file"

    # Check 1: Uppercase WHERE/AND/OR/NOT
    if grep -nE '\b(WHERE|AND|OR|NOT)\b' "$file"; then
        echo "❌ ERROR: Found uppercase logical operators (must be lowercase)"
        ((errors++))
    fi

    # Check 2: Uppercase TRUE/FALSE
    if grep -nE '\b(TRUE|FALSE|True|False)\b' "$file"; then
        echo "❌ ERROR: Found uppercase boolean values (must be lowercase)"
        ((errors++))
    fi

    # Check 3: Dot notation in field access
    if grep -nE '(attributes|cache|resource)\.[a-zA-Z_]' "$file"; then
        echo "❌ ERROR: Found dot notation (must use bracket notation)"
        ((errors++))
    fi

    # Check 4: Use of 'null' instead of 'nil'
    if grep -nE '(==|!=)\s*null\b' "$file"; then
        echo "❌ ERROR: Found 'null' (must use 'nil')"
        ((errors++))
    fi

    # Check 5: Single backslash in regex patterns (basic check)
    if grep -nE 'IsMatch.*\\d|IsMatch.*\\s|IsMatch.*\\w' "$file" | grep -v '\\\\'; then
        echo "⚠️  WARNING: Possible single backslash in regex (may need doubling)"
    fi

    # Check 6: Direct body usage without Decode
    if grep -nE 'ParseJSON\(body\)|IsMatch\(body|Substring\(body' "$file"; then
        echo "❌ ERROR: Found body used without Decode() wrapper"
        ((errors++))
    fi

    if [ $errors -eq 0 ]; then
        echo "✅ Validation passed: No OTTL syntax errors found"
        return 0
    else
        echo "❌ Validation failed: Found $errors error(s)"
        return 1
    fi
}

# Usage
if [ $# -eq 0 ]; then
    echo "Usage: $0 <pipeline.yaml>"
    exit 1
fi

validate_ottl "$1"
```

**Save as**: `/usr/local/bin/ottl-validate`
**Make executable**: `chmod +x /usr/local/bin/ottl-validate`
**Usage**: `ottl-validate pipeline.yaml`

---

## Testing Patterns

### 1. Syntax Validation Test

```yaml
# Test file: ottl-syntax-test.yaml
processors:
  - type: ottl
    name: syntax_test
    statements:
      # Test 1: Lowercase operators
      - set(cache["test1"], true) where attributes["level"] == "ERROR" and attributes["code"] == 500

      # Test 2: Boolean values
      - set(cache["test2"], true)
      - set(cache["test3"], false)

      # Test 3: Field access
      - set(cache["test4"], attributes["nested"]["field"])

      # Test 4: Nil checks
      - set(cache["test5"], true) where attributes["optional"] != nil

      # Test 5: Body decoding
      - set(cache["decoded"], Decode(body, "utf-8"))
      - set(cache["json"], ParseJSON(cache["decoded"]))
```

**Validation**: `ottl-validate ottl-syntax-test.yaml`

---

### 2. Type Checking Test

```yaml
# Test different data types
processors:
  - type: ottl
    name: type_test
    statements:
      # String type
      - set(attributes["string_field"], "value")

      # Integer type
      - set(attributes["int_field"], 42)

      # Float type
      - set(attributes["float_field"], 3.14)

      # Boolean type
      - set(attributes["bool_field"], true)

      # Type conversion
      - set(attributes["int_to_string"], String(attributes["int_field"]))
      - set(attributes["string_to_int"], Int(attributes["string_field"]))
```

---

### 3. Field Existence Verification Test

```yaml
# Test field existence checks
processors:
  - type: ottl
    name: existence_test
    statements:
      # Safe access with existence check
      - set(cache["has_user"], true) where attributes["user_id"] != nil
      - set(attributes["user"], attributes["user_id"]) where attributes["user_id"] != nil

      # Conditional processing
      - set(cache["has_error"], true) where attributes["error"] != nil and attributes["error"] != ""
```

---

### 4. Edge Case Handling Test

```yaml
# Test edge cases
processors:
  - type: ottl
    name: edge_case_test
    statements:
      # Empty string handling
      - set(cache["is_empty"], true) where attributes["value"] == ""

      # Zero value handling
      - set(cache["is_zero"], true) where attributes["count"] == 0

      # Nil vs empty string
      - set(cache["nil_check"], true) where attributes["field"] == nil
      - set(cache["empty_check"], true) where attributes["field"] == ""

      # Complex conditions
      - set(cache["complex"], true) where (attributes["a"] == "1" or attributes["b"] == "2") and attributes["c"] != nil
```

---

## Quick Reference Card

```
OTTL Syntax Rules - Quick Reference
====================================

✅ DO:
- Use lowercase: where, and, or, not
- Quote strings: "ERROR", "value"
- Bracket notation: attributes["field"]
- Lowercase booleans: true, false
- Decode body: Decode(body, "utf-8")
- Nil checks: field != nil
- Double backslashes in regex: \\d+

❌ DON'T:
- Uppercase: WHERE, AND, OR, NOT
- Unquoted strings: ERROR, value
- Dot notation: attributes.field
- Uppercase booleans: TRUE, FALSE, "true"
- Direct body access: ParseJSON(body)
- Null checks: field != null
- Single backslashes: \d+

Common Errors:
- "syntax error near WHERE" → Use lowercase where
- "undefined identifier ERROR" → Add quotes "ERROR"
- "unexpected '.'" → Use bracket notation
- "undefined identifier TRUE" → Use lowercase true
- "type mismatch" → Decode body first
- "undefined identifier null" → Use nil
```

---

## Additional Resources

- **OTTL Functions Reference**: See edgedelta-ottl/FUNCTION_INDEX.md
- **EDX Extensions Guide**: See edgedelta-ottl/EDX_EXTENSIONS.md
- **Pipeline Examples**: See examples/ directory
- **Common Patterns**: See edgedelta-ottl/COMMON_PATTERNS.md

---

## Version History

- **2025-10-20**: Initial creation with 13 critical validation rules
- **Coverage**: All major OTTL syntax requirements
- **Testing**: Automated validation script + 4 test patterns

---

**ENFORCEMENT NOTICE**: These rules are automatically checked in CI/CD pipelines. All OTTL syntax violations will cause deployment failures. Validate before committing.
