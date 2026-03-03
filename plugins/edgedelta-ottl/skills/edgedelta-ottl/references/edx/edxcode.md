# edx_code - Execute JavaScript Code

**Function Type**: EDX Editor
**Purpose**: Executes custom JavaScript code in a sandboxed runtime for dynamic transformations
**Signature**: `edx_code(code) -> void`
**Return Type**: void (modifies item in place)
**Minimum Agent Version**: v2.6.0
**Source**: /pkg/ottlext/funcs/func_edx_code.go

---

## Quick Copy

```yaml
# Execute JavaScript to transform data
# Note: Used in Code Processor - JavaScript modifies 'item' object directly
statements: |-
  edx_code("item['attributes']['severity'] = item['resource']['raw_data']['alert_type'] === 'error' ? 1 : item['resource']['raw_data']['alert_type'] === 'warning' ? 2 : 0;")
```

---

## Overview

edx_code provides a JavaScript runtime for complex transformations that are difficult or impossible with standard OTTL functions.

**Important**: The function name is `edx_code` (lowercase with underscore), not `EDXCode`. This follows the EDX Editor naming convention.

**⚠️ WARNING: Use Sparingly**
- Performance overhead
- Security considerations
- Maintenance complexity
- Prefer standard OTTL functions when possible

**Valid Use Cases:**
- Complex string manipulations
- Custom business logic
- Dynamic field transformations
- Prototyping before implementing native functions

**Available in JavaScript:**
- `item` object (current log/metric/trace)
- Standard JavaScript features (limited)
- No external libraries
- Sandboxed execution

---

## Parameters

### code
- **Type**: string
- **Required**: Yes
- **Description**: JavaScript code to execute
- **Context**: Has access to `item` object
- **Return**: Can return values or modify `item` directly

---

## Return Value

Returns the result of JavaScript execution.

**Behavior:**
- If code returns value → return that value
- If code modifies `item` → side effects applied
- Errors → returns error

---

## Examples

### Example 1: Add Computed Field
```yaml
processors:
  - name: computed-field
    transform:
      statements: |-
        edx_code("item['attributes']['full_name'] = item['attributes']['first_name'] + ' ' + item['attributes']['last_name'];")
```

### Example 2: Conditional Field Setting
```yaml
processors:
  - name: conditional-logic
    transform:
      statements: |-
        edx_code("if (item['attributes']['status'] === 'error') { item['attributes']['priority'] = 'high'; } else { item['attributes']['priority'] = 'low'; }")
```

### Example 3: Array Manipulation
```yaml
processors:
  - name: array-transform
    transform:
      statements: |-
        edx_code("var tags = item['attributes']['tags']; item['attributes']['tag_count'] = tags.length; item['attributes']['first_tag'] = tags[0];")
```

### Example 4: String Transformations
```yaml
processors:
  - name: string-ops
    transform:
      statements: |-
        edx_code("item['attributes']['message_upper'] = item['attributes']['message'].toUpperCase(); item['attributes']['message_length'] = item['attributes']['message'].length;")
```

### Example 5: Complex Calculations
```yaml
processors:
  - name: calculate
    transform:
      statements: |-
        edx_code("var price = item['attributes']['price']; var quantity = item['attributes']['quantity']; var tax_rate = 0.08; item['attributes']['total'] = price * quantity * (1 + tax_rate);")
```

---

## Common Use Cases

### 1. Dynamic Field Generation
```yaml
processors:
  - name: dynamic-fields
    transform:
      statements: |-
        edx_code("item['attributes']['timestamp_ms'] = Date.now(); item['attributes']['random_id'] = Math.random().toString(36);")
```

### 2. Complex Validation
```yaml
processors:
  - name: validate
    transform:
      statements: |-
        edx_code("var email = item['attributes']['email']; var isValid = /^[^@]+@[^@]+\\.[^@]+$/.test(email); item['attributes']['is_valid'] = isValid;")
```

---

## Validation Rules

1. **Code Required**: Must provide JavaScript code string
2. **Syntax Valid**: Code must be valid JavaScript
3. **Sandboxed**: Limited JavaScript features available
4. **No Async**: Cannot use async/await or Promises
5. **No External Libs**: No npm packages or external imports

---

## Common Pitfalls

### 1. Performance Overhead
```yaml
# AVOID: Using edx_code for simple operations
statements: |-
  edx_code("item['attributes']['upper'] = item['attributes']['text'].toUpperCase();")

# PREFER: Use standard OTTL functions
statements: |-
  set(attributes["upper"], to_upper_case(attributes["text"]))
```

### 2. Security Risks
```yaml
# DANGEROUS: Don't use user input directly in code
statements: |-
  set(cache["code"], concat(["item['attributes']['result'] = ", attributes["user_input"], ";"]))
  edx_code(cache["code"])  # UNSAFE!

# SAFE: Validate and sanitize first, or use OTTL functions instead
```

### 3. Maintenance Issues
```yaml
# HARD TO MAINTAIN: Complex inline JavaScript
statements: |-
  edx_code("complex 50-line script here...")

# BETTER: Keep code simple, or use native OTTL
```

---

## Best Practices

### 1. Use Only When Necessary
```yaml
# Prefer standard OTTL functions
# Use edx_code only for truly complex logic that can't be expressed otherwise
```

### 2. Keep Code Simple
```yaml
# GOOD: Short, clear code
statements: |-
  edx_code("item['attributes']['computed'] = a + b;")

# AVOID: Long, complex scripts
```

### 3. Document Thoroughly
```yaml
# Comment why edx_code is needed
processors:
  - name: special-calculation
    # Using edx_code because calculation involves complex business logic
    # that changes frequently based on customer requirements
    transform:
      statements: |-
        edx_code("...")
```

### 4. Consider Native Functions First
```yaml
# Before using edx_code, check if combination of standard functions works
```

---

## Security Considerations

1. **Sandboxed Execution**: Code runs in limited environment
2. **No File System Access**: Cannot read/write files
3. **No Network Access**: Cannot make HTTP requests
4. **No Process Access**: Cannot execute shell commands
5. **Input Validation**: Validate any user input used in code
6. **Code Review**: Review all edx_code usage carefully

---

## Performance Considerations

- **Slow**: JavaScript execution has overhead
- **No JIT**: May not have JIT compilation
- **Memory**: Each execution allocates VM resources
- **Alternative**: Consider implementing custom OTTL function

**When Performance Matters:**
- Avoid edx_code in high-throughput paths
- Cache results when possible
- Consider native OTTL functions
- Profile before deploying

---

## Related Functions

- **Standard OTTL Functions**: Always prefer these
- **EDXIfElse**: For conditional logic
- **Standard String/Math Functions**: For simple operations

---

## Cross-References

- [MASTER_INDEX.md](../../MASTER_INDEX.md#edx_code)
- [EDXIfElse](./edxifelse.md)

---

## Notes

- **Version**: Introduced in v2.6.0
- **Runtime**: Embedded JavaScript engine (goja)
- **Restrictions**: Limited API surface
- **Use Sparingly**: Performance and maintenance concerns
- **Type**: EDX Editor function (modifies item in place)
- **Naming**: Function name is `edx_code` (lowercase_underscore), not `EDXCode`
- **Framework**: Code automatically wrapped with `transform(item)` function and `return item;`
