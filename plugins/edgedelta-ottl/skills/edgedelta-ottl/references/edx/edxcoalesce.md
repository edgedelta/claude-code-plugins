# EDXCoalesce - Return First Non-Empty Value

**Function Type**: EDX Converter
**Purpose**: Returns the first non-empty/non-null value from a list of expressions, similar to SQL COALESCE or JavaScript's nullish coalescing operator
**Signature**: `EDXCoalesce(expression, fallbackExpression) -> any`
**Return Type**: any (returns the type of the first non-empty expression)
**Minimum Agent Version**: v2.0.0
**Source**: /pkg/ottlext/funcs/func_edx_coalesce.go

---

## Quick Copy

```yaml
# Basic coalesce with fallback
set(attributes["value"], EDXCoalesce(attributes["primary"], attributes["fallback"]))
```

---

## Overview

EDXCoalesce evaluates expressions in order and returns the first non-empty value. This is essential for handling optional fields, providing default values, and implementing fallback chains in pipeline processing.

Unlike simple null checks, EDXCoalesce considers multiple "empty" states:
- `nil` values
- Empty strings (`""`)
- Empty byte slices (`[]byte{}`)
- Empty maps (`map[string]any{}`)
- Empty slices/arrays (`[]any{}`)
- Zero numeric values (`0`, `int32(0)`, `int64(0)`)
- Empty pcommon types (Map, Slice, ByteSlice, etc.)

This makes it ideal for data enrichment scenarios where you want to use a primary source but fall back to secondary sources if the primary is unavailable or empty.

**Common Use Cases:**
- Providing default values for missing attributes
- Implementing fallback chains (primary → secondary → default)
- Handling optional configuration values
- Normalizing data from multiple possible sources
- Graceful degradation in data pipelines

---

## Parameters

### expression
- **Type**: ottl.Getter[any]
- **Required**: Yes
- **Description**: Primary expression to evaluate. If this returns a non-empty value, it will be returned
- **Valid Values**: Any OTTL expression (attribute access, function call, literal)
- **Evaluation**: Evaluated first; if non-empty, short-circuits and returns immediately

### fallbackExpression
- **Type**: ottl.Getter[any]
- **Required**: Yes
- **Description**: Fallback expression to evaluate if the primary expression is empty or null
- **Valid Values**: Any OTTL expression
- **Evaluation**: Only evaluated if the primary expression is empty
- **Error Handling**: If both expressions error, the fallback error is returned

---

## Return Value

Returns the value of the first non-empty expression. The return type matches the type of the returned value (can be string, int, map, etc.).

**Return Logic:**
1. Evaluate `expression`
2. If `expression` returns a non-empty value → return that value
3. If `expression` is empty or null → evaluate `fallbackExpression`
4. Return `fallbackExpression` value (even if empty)
5. If both expressions error → return the fallback expression's error

**Empty Value Detection:**
- `nil` → empty
- `""` (empty string) → empty
- `[]byte{}` → empty
- `map[string]any{}` → empty
- `[]any{}` → empty
- `0` (any zero numeric type) → empty
- Empty pcommon types → empty

---

## Examples

### Example 1: Basic Fallback for Missing Attribute
```yaml
# Use user-provided name, fall back to default
processors:
  - name: normalize-username
    transform:
      statements:
        - set(attributes["username"], EDXCoalesce(attributes["user.name"], "anonymous"))
```

### Example 2: Multi-Level Fallback Chain
```yaml
# Try multiple possible field names
processors:
  - name: normalize-ip
    transform:
      statements:
        - set(attributes["client_ip"], EDXCoalesce(
            attributes["http.client_ip"],
            EDXCoalesce(
              attributes["x_forwarded_for"],
              "0.0.0.0"
            )
          ))
```

### Example 3: Handling Zero Values with Defaults
```yaml
# Use configured timeout, fall back to default if zero
processors:
  - name: set-timeout
    transform:
      statements:
        - set(attributes["timeout_ms"], EDXCoalesce(
            attributes["config.timeout"],
            5000
          ))
```

### Example 4: Environment Variable with Fallback
```yaml
# Use environment variable or configuration value
processors:
  - name: config-from-env
    transform:
      statements:
        - set(attributes["log_level"], EDXCoalesce(
            EDXEnv("LOG_LEVEL"),
            attributes["config.log_level"]
          ))
```

### Example 5: Conditional Fallback
```yaml
# Use premium features if available
processors:
  - name: feature-selection
    transform:
      statements:
        - set(attributes["feature_set"], EDXCoalesce(
            attributes["premium_features"],
            attributes["basic_features"]
          ))
```

### Example 6: Fallback for Parsed Data
```yaml
# Use parsed value or raw value
processors:
  - name: parse-with-fallback
    transform:
      statements:
        - set(cache["parsed"], EDXParseKeyValue(body, "=", ",", true))
        - set(attributes["value"], EDXCoalesce(
            cache["parsed"]["field"],
            body
          ))
```

### Example 7: Handling Empty Strings vs Null
```yaml
# Both null and empty string will trigger fallback
processors:
  - name: normalize-status
    transform:
      statements:
        - set(attributes["status"], EDXCoalesce(
            attributes["response.status"],
            "unknown"
          ))
```

### Example 8: Complex Object Fallback
```yaml
# Fall back to empty map if parsing fails
processors:
  - name: safe-json-parse
    transform:
      statements:
        - set(cache["headers"], EDXCoalesce(
            ParseJSON(attributes["http_headers"]),
            map[string]any{}
          ))
```

### Example 9: Numeric Fallback (Zero is Empty)
```yaml
# Zero counts as empty, use default
processors:
  - name: retry-count
    transform:
      statements:
        - set(attributes["max_retries"], EDXCoalesce(
            attributes["config.retries"],
            3
          ))
```

### Example 10: Chaining with Other Functions
```yaml
# Combine with EDXIfElse for complex logic
processors:
  - name: advanced-fallback
    transform:
      statements:
        - set(attributes["result"], EDXCoalesce(
            EDXIfElse(
              attributes["enabled"],
              attributes["primary_value"],
              nil
            ),
            attributes["default_value"]
          ))
```

---

## Common Use Cases

### 1. Configuration Management
**Scenario**: Provide default configuration values when user configuration is missing or empty.

```yaml
processors:
  - name: config-defaults
    transform:
      statements:
        - set(attributes["batch_size"], EDXCoalesce(attributes["config.batch_size"], 100))
        - set(attributes["timeout_sec"], EDXCoalesce(attributes["config.timeout"], 30))
        - set(attributes["region"], EDXCoalesce(EDXEnv("AWS_REGION"), "us-east-1"))
```

### 2. Data Normalization
**Scenario**: Handle data from multiple sources with different field names.

```yaml
processors:
  - name: normalize-fields
    transform:
      statements:
        # Different log formats use different field names
        - set(attributes["timestamp"], EDXCoalesce(
            attributes["@timestamp"],
            EDXCoalesce(attributes["time"], attributes["log_time"])
          ))
        - set(attributes["message"], EDXCoalesce(
            attributes["msg"],
            EDXCoalesce(attributes["message"], attributes["log"])
          ))
```

### 3. Error Recovery
**Scenario**: Provide safe defaults when parsing or extraction fails.

```yaml
processors:
  - name: safe-extraction
    transform:
      statements:
        - set(cache["parsed"], EDXExtractPatterns(body, "user=(\\w+)"))
        - set(attributes["user"], EDXCoalesce(
            cache["parsed"]["user"],
            "system"
          ))
```

### 4. Multi-Tier Fallback
**Scenario**: Implement cascading fallback for high-availability data sources.

```yaml
processors:
  - name: ha-lookup
    transform:
      statements:
        # Try cache, then database, then static default
        - set(cache["redis_result"], EDXRedis(
            {"url": "redis://primary:6379"},
            [{"command": "get", "key": attributes["lookup_key"], "outField": "value"}]
          ))
        - set(attributes["enriched_data"], EDXCoalesce(
            cache["redis_result"]["value"],
            EDXCoalesce(
              attributes["database_value"],
              "default"
            )
          ))
```

---

## Validation Rules

1. **Both Parameters Required**: Both `expression` and `fallbackExpression` must be provided
2. **Type Safety**: No type restrictions - can coalesce different types, but consider downstream usage
3. **Error Handling**: If both expressions error, the function returns the fallback expression's error
4. **Evaluation Order**: Primary expression is always evaluated first
5. **Short-Circuit Evaluation**: If primary expression is non-empty, fallback is never evaluated
6. **Empty Detection**: Zero numeric values (0, 0.0) are considered empty and trigger fallback

---

## Common Pitfalls

### 1. Zero Values Considered Empty
**Problem**: Numeric zero is treated as empty, which may not always be desired.

```yaml
# INCORRECT: Zero is a valid value but triggers fallback
set(attributes["count"], EDXCoalesce(attributes["item_count"], 10))
# If item_count = 0, this incorrectly returns 10

# CORRECT: Use EDXIfElse when zero is valid
set(attributes["count"], EDXIfElse(
  attributes["item_count"] != nil,
  attributes["item_count"],
  10
))
```

### 2. Error Propagation Confusion
**Problem**: Understanding which error is returned when both expressions fail.

```yaml
# The FALLBACK error is returned, not the primary error
set(attributes["value"], EDXCoalesce(
  attributes["missing_field"],  # Error: field not found
  attributes["also_missing"]    # Error: field not found (this error is returned)
))

# BETTER: Use explicit error handling
set(cache["primary"], attributes["field"] or nil)
set(cache["fallback"], attributes["backup"] or nil)
set(attributes["value"], EDXCoalesce(cache["primary"], cache["fallback"]))
```

### 3. Deep Nesting Reduces Readability
**Problem**: Too many nested coalesce calls become hard to maintain.

```yaml
# POOR: Hard to read
set(attributes["value"], EDXCoalesce(
  attributes["a"],
  EDXCoalesce(
    attributes["b"],
    EDXCoalesce(attributes["c"], "default")
  )
))

# BETTER: Use intermediate variables
set(cache["fallback_bc"], EDXCoalesce(attributes["b"], attributes["c"]))
set(cache["final"], EDXCoalesce(attributes["a"], cache["fallback_bc"]))
set(attributes["value"], EDXCoalesce(cache["final"], "default"))
```

### 4. Type Mixing Without Validation
**Problem**: Coalescing different types can cause downstream issues.

```yaml
# RISKY: May return string or number
set(attributes["value"], EDXCoalesce(
  attributes["string_field"],  # Returns "abc"
  attributes["number_field"]   # Returns 123
))

# BETTER: Ensure consistent types
set(attributes["value"], EDXCoalesce(
  String(attributes["string_field"]),
  String(attributes["number_field"])
))
```

---

## Best Practices

### 1. Use for Optional Fields
Always provide sensible defaults for optional configuration or data fields.

```yaml
processors:
  - name: with-defaults
    transform:
      statements:
        - set(attributes["region"], EDXCoalesce(attributes["aws.region"], "us-east-1"))
        - set(attributes["env"], EDXCoalesce(EDXEnv("ENVIRONMENT"), "production"))
```

### 2. Keep Fallback Chains Shallow
Limit nesting to 2-3 levels maximum for maintainability.

```yaml
# GOOD: Two-level fallback
set(attributes["ip"], EDXCoalesce(
  attributes["client_ip"],
  attributes["remote_addr"]
))

# ACCEPTABLE: Three-level with intermediates
set(cache["backup"], EDXCoalesce(attributes["secondary"], "default"))
set(attributes["value"], EDXCoalesce(attributes["primary"], cache["backup"]))
```

### 3. Document Empty Semantics
When zero is a valid value, use EDXIfElse instead.

```yaml
# When 0 is valid (age, count, etc.)
set(attributes["age"], EDXIfElse(
  attributes["user.age"] != nil,
  attributes["user.age"],
  -1  # Use sentinel value instead
))
```

### 4. Combine with EDXEnv for Configuration
Use environment variables as the ultimate fallback.

```yaml
processors:
  - name: config-hierarchy
    transform:
      statements:
        # Priority: attribute → env var → default
        - set(attributes["api_key"], EDXCoalesce(
            attributes["config.api_key"],
            EDXEnv("API_KEY", "default-key")
          ))
```

### 5. Safe Parsing with Fallbacks
Always provide fallbacks when parsing untrusted data.

```yaml
processors:
  - name: safe-parse
    transform:
      statements:
        - set(cache["parsed"], ParseJSON(body) or map[string]any{})
        - set(attributes["user_id"], EDXCoalesce(
            cache["parsed"]["user_id"],
            "unknown"
          ))
```

---

## Performance Considerations

- **Short-Circuit Optimization**: If the primary expression returns a non-empty value, the fallback expression is never evaluated, saving computation
- **Error Evaluation**: Both expressions may be evaluated even if one errors (to determine which error to return)
- **Type Checking Overhead**: Each evaluation checks multiple type conditions to determine if a value is empty
- **Nested Calls**: Deep nesting creates multiple function call stacks; consider using intermediate cache variables
- **Memory Allocation**: Empty detection for slices and maps requires minimal memory inspection

**Performance Tips:**
- Place most likely successful expressions first (primary position)
- Use cache variables to avoid re-evaluating expensive expressions
- Limit nesting depth to reduce call stack overhead
- Consider EDXIfElse for simple conditions instead of Coalesce

---

## Related Functions

- **EDXIfElse**: Use when you need conditional logic beyond null/empty checks (e.g., when 0 is valid)
- **EDXEnv**: Often used as the fallback expression to provide environment-based defaults
- **Standard OTTL nil coalescing**: EDXCoalesce provides more comprehensive empty checking
- **ParseJSON**: Commonly paired with Coalesce to provide fallback for parsing failures

---

## Cross-References

- [MASTER_INDEX.md](../../MASTER_INDEX.md#edxcoalesce)
- [EDXIfElse](./edxifelse.md) - Conditional logic with explicit conditions
- [EDXEnv](./edxenv.md) - Environment variable access for fallback values
- [syntax_guide.md](../syntax/syntax_guide.md) - OTTL expression syntax

---

## Notes

- **Version History**: Introduced in v2.0.0
- **Empty vs Null**: The function treats many "falsy" values as empty, including zero numbers
- **Error Handling**: When both expressions fail, the fallback expression's error is returned
- **Type Flexibility**: No type constraints on return values - can return different types
- **Evaluation Guarantee**: Primary expression is always evaluated; fallback is conditionally evaluated
- **pcommon Types**: Fully compatible with OpenTelemetry pcommon data types (Map, Slice, etc.)
