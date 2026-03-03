# EDXIfElse - Ternary Conditional Operator

**Function Type**: EDX Converter
**Purpose**: Conditional expression that returns one of two values based on a boolean condition (ternary operator)
**Signature**: `EDXIfElse(condition, valueIfTrue, valueIfFalse) -> any`
**Return Type**: any (type of returned value)
**Minimum Agent Version**: v2.0.0
**Source**: /pkg/ottlext/funcs/func_edx_if_else.go

---

## Quick Copy

```yaml
# Simple if-else logic
set(attributes["status"], EDXIfElse(
  attributes["error"] != nil,
  "failed",
  "success"
))
```

---

## Overview

EDXIfElse provides inline conditional logic similar to ternary operators in programming languages (`condition ? valueIfTrue : valueIfFalse`).

**Use Cases:**
- Conditional value assignment
- Dynamic routing
- Type-based processing
- Feature flags
- Default value selection

**vs EDXCoalesce:**
- EDXIfElse: Explicit condition evaluation
- EDXCoalesce: Null/empty checks only

---

## Parameters

### condition
- **Type**: bool expression
- **Required**: Yes
- **Description**: Boolean condition to evaluate
- **Examples**: `attributes["enabled"] == true`, `Len(body) > 1000`

### valueIfTrue
- **Type**: any
- **Required**: Yes
- **Description**: Value to return if condition is true
- **Can be**: Literal, attribute, function call

### valueIfFalse
- **Type**: any
- **Required**: Yes
- **Description**: Value to return if condition is false
- **Can be**: Literal, attribute, function call

---

## Return Value

Returns `valueIfTrue` if condition is true, otherwise `valueIfFalse`.

**Evaluation:**
- Condition always evaluated
- Only one value expression evaluated (short-circuit)
- Return type matches returned value's type

---

## Examples

### Example 1: Status Based on Error
```yaml
processors:
  - name: set-status
    transform:
      statements:
        - set(attributes["status"], EDXIfElse(
            attributes["error"] != nil,
            "error",
            "ok"
          ))
```

### Example 2: Environment-Based Routing
```yaml
processors:
  - name: route-by-env
    transform:
      statements:
        - set(cache["is_prod"], EDXEnv("ENVIRONMENT") == "production")
        - set(attributes["endpoint"], EDXIfElse(
            cache["is_prod"],
            "https://api.prod.com",
            "https://api.dev.com"
          ))
```

### Example 3: Size-Based Compression
```yaml
processors:
  - name: conditional-compress
    transform:
      statements:
        - set(cache["large"], Len(body) > 10240)
        - set(cache["final_body"], EDXIfElse(
            cache["large"],
            EDXCompress(body, "gzip"),
            body
          ))
```

### Example 4: Type-Based Processing
```yaml
processors:
  - name: type-handler
    transform:
      statements:
        - set(cache["is_string"], EDXDataType(attributes["value"]) == "string")
        - set(attributes["processed"], EDXIfElse(
            cache["is_string"],
            attributes["value"],
            String(attributes["value"])
          ))
```

### Example 5: Nested Conditions
```yaml
processors:
  - name: multi-level
    transform:
      statements:
        - set(attributes["priority"], EDXIfElse(
            attributes["severity"] == "critical",
            "P0",
            EDXIfElse(
              attributes["severity"] == "high",
              "P1",
              "P2"
            )
          ))
```

### Example 6: Feature Flag
```yaml
processors:
  - name: feature-flag
    transform:
      statements:
        - set(cache["enabled"], EDXEnv("FEATURE_X_ENABLED") == "true")
        - set(cache["result"], EDXIfElse(
            cache["enabled"],
            ProcessWithFeatureX(attributes["data"]),
            attributes["data"]
          ))
```

### Example 7: Validation with Default
```yaml
processors:
  - name: validate-age
    transform:
      statements:
        - set(cache["age_valid"], attributes["age"] >= 0 and attributes["age"] < 150)
        - set(attributes["age_normalized"], EDXIfElse(
            cache["age_valid"],
            attributes["age"],
            -1
          ))
```

### Example 8: Math Operations
```yaml
processors:
  - name: conditional-math
    transform:
      statements:
        - set(cache["discount"], EDXIfElse(
            attributes["is_premium"],
            attributes["price"] * 0.8,
            attributes["price"]
          ))
```

---

## Common Use Cases

### 1. Dynamic Configuration
```yaml
processors:
  - name: config-select
    transform:
      statements:
        - set(cache["env"], EDXEnv("ENVIRONMENT"))
        - set(attributes["batch_size"], EDXIfElse(
            cache["env"] == "production",
            1000,
            100
          ))
        - set(attributes["timeout"], EDXIfElse(
            cache["env"] == "production",
            60,
            10
          ))
```

### 2. Error Handling
```yaml
processors:
  - name: safe-processing
    transform:
      statements:
        - set(cache["parsed"], ParseJSON(body) or nil)
        - set(cache["success"], cache["parsed"] != nil)
        - set(attributes["result"], EDXIfElse(
            cache["success"],
            cache["parsed"]["data"],
            "parse_failed"
          ))
```

### 3. Rate Limiting
```yaml
processors:
  - name: rate-limit
    transform:
      statements:
        - set(cache["under_limit"], attributes["request_count"] < 1000)
        - set(attributes["action"], EDXIfElse(
            cache["under_limit"],
            "process",
            "throttle"
          ))
```

---

## Validation Rules

1. **All Parameters Required**: Must provide all three parameters
2. **Boolean Condition**: First parameter must evaluate to boolean
3. **Type Flexibility**: Value parameters can be any type
4. **Short Circuit**: Only evaluates one value expression

---

## Common Pitfalls

### 1. Non-Boolean Condition
```yaml
# WRONG: String is not boolean
set(cache["result"], EDXIfElse(
  attributes["status"],  # "ok" is not true/false
  "yes",
  "no"
))

# CORRECT: Use comparison
set(cache["result"], EDXIfElse(
  attributes["status"] == "ok",
  "yes",
  "no"
))
```

### 2. Deep Nesting Readability
```yaml
# HARD TO READ: Too many nested conditions
set(cache["value"], EDXIfElse(
  condition1,
  EDXIfElse(condition2, EDXIfElse(condition3, a, b), c),
  d
))

# BETTER: Use intermediate variables
set(cache["level3"], EDXIfElse(condition3, a, b))
set(cache["level2"], EDXIfElse(condition2, cache["level3"], c))
set(cache["value"], EDXIfElse(condition1, cache["level2"], d))
```

### 3. Type Mismatch Issues
```yaml
# RISKY: Different return types
set(attributes["value"], EDXIfElse(
  condition,
  123,  # int
  "abc"  # string
))
# Downstream code must handle both types

# SAFER: Consistent types
set(attributes["value"], EDXIfElse(
  condition,
  "123",
  "abc"
))
```

---

## Best Practices

### 1. Keep Conditions Simple
```yaml
# GOOD: Clear condition
set(cache["enabled"], attributes["feature_x"] == true)
set(cache["result"], EDXIfElse(cache["enabled"], "on", "off"))

# AVOID: Complex inline condition
set(cache["result"], EDXIfElse(
  attributes["a"] > 10 and attributes["b"] < 5 or attributes["c"] == "test",
  "yes",
  "no"
))
```

### 2. Use Intermediate Variables
```yaml
processors:
  - name: readable-conditions
    transform:
      statements:
        - set(cache["is_large"], Len(body) > 10000)
        - set(cache["is_json"], attributes["content_type"] == "application/json")
        - set(cache["should_compress"], cache["is_large"] and cache["is_json"])
        - set(cache["final"], EDXIfElse(
            cache["should_compress"],
            EDXCompress(body, "gzip"),
            body
          ))
```

### 3. Limit Nesting Depth
```yaml
# Maximum 2-3 levels of nesting
# For complex logic, use multiple EDXIfElse statements
```

---

## Performance Considerations

- **Fast Evaluation**: Minimal overhead
- **Short Circuit**: Only one value branch executed
- **No Lazy Evaluation**: Condition always evaluated
- **Type Agnostic**: No type conversion overhead

**Tips:**
- Place expensive operations in value branches (not condition)
- Use cache for repeated condition checks
- Prefer simple conditions over complex ones

---

## Related Functions

- **EDXCoalesce**: For null/empty checks (simpler syntax)
- **EDXDataType**: Type detection for conditional logic
- **Standard OTTL if**: Statement-level conditionals

---

## Cross-References

- [MASTER_INDEX.md](../../MASTER_INDEX.md#edxifelse)
- [EDXCoalesce](./edxcoalesce.md)
- [EDXDataType](./edxdatatype.md)

---

## Notes

- **Version**: Introduced in v2.0.0
- **Syntax**: Similar to `condition ? true : false` in many languages
- **Thread Safety**: Thread-safe
- **No Null Coalescing**: Use EDXCoalesce for null checks
