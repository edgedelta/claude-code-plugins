# OTTL Syntax Guide

**Purpose**: Complete OTTL (OpenTelemetry Transformation Language) syntax specification for EdgeDelta pipelines
**Last Updated**: 2025-10-20
**Source**: OpenTelemetry OTTL + EdgeDelta Extensions

---

## Table of Contents

1. [Statement Structure](#statement-structure)
2. [Function Categories](#function-categories)
3. [Operators](#operators)
4. [Conditional Logic](#conditional-logic)
5. [Field Access](#field-access)
6. [Data Types](#data-types)
7. [Special Fields](#special-fields)
8. [Critical Syntax Rules](#critical-syntax-rules)
9. [Common Patterns](#common-patterns)
10. [Syntax Errors and Fixes](#syntax-errors-and-fixes)

---

## Statement Structure

Every OTTL statement follows this pattern:

```
<function_call>(<target_specification>, <argument_parameters>)
```

### Basic Statement Anatomy

**Components**:
- **Function Call**: The OTTL function name (e.g., `set`, `ParseJSON`, `delete_key`)
- **Target Specification**: The field to operate on, using bracket notation
- **Argument Parameters**: Additional parameters specific to the function

**Examples**:

```yaml
# Simple assignment
set(attributes["key"], "value")

# Function with multiple parameters
set(attributes["result"], Concat([attributes["first"], " ", attributes["last"]]))

# Nested function calls (converter inside editor)
set(attributes["parsed"], ParseJSON(Decode(body, "utf-8")))

# With conditional clause
set(attributes["alert"], "Critical") where attributes["status"] == "error"
```

### Target Specification

Targets use **bracket notation** to specify fields:

```yaml
attributes["field_name"]    # Log/span/metric attributes
resource["field_name"]      # Resource attributes
cache["field_name"]         # Temporary storage
body                        # Raw telemetry data (byte array)
```

### Return Values

- **Editor functions**: Modify data in-place, no return value
- **Converter functions**: Return transformed data without modifying originals

### Chaining Functions

Converter functions can be nested inside other functions:

```yaml
# Three functions chained: Decode → ParseJSON → set
set(attributes["data"], ParseJSON(Decode(body, "utf-8")))

# Complex chaining
set(attributes["timestamp"], UnixMilli(Time(cache["parsed"]["@timestamp"], "%Y-%m-%dT%H:%M:%S")))

# Multiple nested converters
set(attributes["full_name"], Concat([ToUpperCase(attributes["first"]), " ", ToUpperCase(attributes["last"])]))
```

---

## Function Categories

OTTL functions fall into two main categories:

### Editors

**Purpose**: Modify telemetry data directly (in-place mutations)

**Common Editor Functions**:
- `set(target, value)` - Assign a value to a field
- `delete_key(target, key)` - Remove a key from a map
- `delete_matching_keys(target, pattern)` - Remove keys matching a pattern
- `keep_keys(target, keys[])` - Keep only specified keys
- `limit(target, limit)` - Limit collection size
- `merge_maps(target, source)` - Merge maps together
- `replace_match(target, pattern, replacement)` - Replace matched text
- `replace_pattern(target, regex, replacement)` - Replace regex matches
- `truncate_all(target, limit)` - Truncate all strings in a map

**Key Characteristics**:
- Modify data in-place
- No return value
- Always used as top-level statements
- Can contain converter functions as parameters

**Example**:
```yaml
# Editor function (set) with converter function (ParseJSON)
set(attributes["json_data"], ParseJSON(Decode(body, "utf-8")))
```

### Converters

**Purpose**: Transform data without altering originals (pure functions)

**Common Converter Functions**:
- `ParseJSON(target)` - Parse JSON string to map
- `Decode(target, encoding)` - Decode byte array to string
- `Concat(values[])` - Concatenate strings
- `Int(value)` - Convert to integer
- `Double(value)` - Convert to double
- `String(value)` - Convert to string
- `ToUpperCase(target)` - Convert to uppercase
- `ToLowerCase(target)` - Convert to lowercase
- `Time(time, format)` - Parse time string
- `UnixMilli(time)` - Convert to Unix milliseconds

**Key Characteristics**:
- Return transformed data
- Don't modify originals
- Used inside other functions (nested)
- Can be chained together

**Example**:
```yaml
# Converter functions chained together
set(attributes["count"], Int(attributes["count_str"]))
set(attributes["upper"], ToUpperCase(attributes["name"]))
```

### When to Use Each Type

**Use Editors when**:
- You want to modify telemetry data
- You need to set, delete, or merge fields
- You're performing top-level operations

**Use Converters when**:
- You need to transform data for assignment
- You're parsing, converting, or manipulating values
- You want to chain transformations together

---

## Operators

### Comparison Operators

Used in `where` clauses to compare values:

| Operator | Meaning | Example |
|----------|---------|---------|
| `==` | Equal to | `attributes["status"] == "error"` |
| `!=` | Not equal to | `attributes["level"] != "DEBUG"` |
| `>` | Greater than | `attributes["count"] > 100` |
| `<` | Less than | `attributes["latency"] < 500` |
| `>=` | Greater than or equal | `attributes["severity"] >= 5` |
| `<=` | Less than or equal | `attributes["retry"] <= 3` |

**Examples**:
```yaml
set(attributes["high_priority"], true) where attributes["severity"] > 7
set(attributes["excluded"], true) where attributes["log_type"] == "TRAFFIC"
set(attributes["valid"], true) where attributes["count"] >= 1 and attributes["count"] <= 100
```

### Logical Operators

**CRITICAL**: All logical operators **must be lowercase**!

| Operator | Meaning | Example |
|----------|---------|---------|
| `and` | Logical AND | `attributes["a"] == "x" and attributes["b"] == "y"` |
| `or` | Logical OR | `attributes["level"] == "ERROR" or attributes["level"] == "CRITICAL"` |
| `not` | Logical NOT | `not (attributes["status"] == "success")` |

**IMPORTANT**: Uppercase operators (`AND`, `OR`, `NOT`) will cause validation errors!

**Correct Examples**:
```yaml
# ✅ CORRECT - lowercase operators
set(cache["match"], true) where attributes["level"] == "ERROR" and attributes["source"] == "api"
set(cache["alert"], true) where attributes["severity"] > 8 or attributes["urgent"] == true
set(cache["invalid"], true) where not (attributes["status"] == "success")
```

**Incorrect Examples**:
```yaml
# ❌ WRONG - uppercase operators will fail
set(cache["match"], true) where attributes["level"] == "ERROR" AND attributes["source"] == "api"
set(cache["alert"], true) where attributes["severity"] > 8 OR attributes["urgent"] == true
set(cache["invalid"], true) where NOT (attributes["status"] == "success")
```

### Operator Precedence

1. **Parentheses**: `()`
2. **NOT**: `not`
3. **Comparison**: `==`, `!=`, `>`, `<`, `>=`, `<=`
4. **AND**: `and`
5. **OR**: `or`

**Example**:
```yaml
# Evaluates as: (severity > 5) AND ((level == "ERROR") OR (level == "CRITICAL"))
set(cache["alert"], true) where attributes["severity"] > 5 and attributes["level"] == "ERROR" or attributes["level"] == "CRITICAL"

# Use parentheses for clarity
set(cache["alert"], true) where attributes["severity"] > 5 and (attributes["level"] == "ERROR" or attributes["level"] == "CRITICAL")
```

---

## Conditional Logic

### where Clause Syntax

The `where` clause filters when statements execute:

**Basic Pattern**:
```
<statement> where <condition>
```

**Key Rules**:
1. **Single-line requirement**: In YAML, entire condition must be on one line
2. **Placement**: Always after the function call
3. **Lowercase keywords**: Use `where`, `and`, `or`, `not` (lowercase only)
4. **No semicolons**: YAML doesn't need statement terminators

### Single Conditions

```yaml
# Simple equality check
set(attributes["excluded"], true) where attributes["log_type"] == "TRAFFIC"

# Numeric comparison
set(attributes["high_latency"], true) where attributes["response_time"] > 1000

# Field existence check
set(attributes["has_user"], true) where attributes["user_id"] != nil
```

### Multiple Conditions with `and`

Combine conditions that must **all be true**:

```yaml
# Two conditions
set(attributes["alert"], "high") where attributes["level"] == "ERROR" and attributes["retry_count"] > 3

# Three conditions
set(cache["match"], true) where attributes["log_type"] != "TRAFFIC" and attributes["log_type"] != "THREAT" and attributes["severity"] > 3

# With numeric ranges
set(attributes["warning"], true) where attributes["temperature"] > 80 and attributes["temperature"] < 100
```

### Multiple Conditions with `or`

Combine conditions where **at least one must be true**:

```yaml
# Two conditions
set(attributes["critical"], true) where attributes["level"] == "ERROR" or attributes["level"] == "CRITICAL"

# Three conditions
set(cache["exclude"], true) where attributes["type"] == "TRAFFIC" or attributes["type"] == "DEBUG" or attributes["type"] == "TRACE"

# Mixed with comparison
set(attributes["alert"], true) where attributes["severity"] > 8 or attributes["urgent"] == true
```

### Combining `and` with `or`

Use parentheses for clarity:

```yaml
# Alert if: (high severity) AND (error OR critical)
set(cache["alert"], true) where attributes["severity"] > 7 and (attributes["level"] == "ERROR" or attributes["level"] == "CRITICAL")

# Exclude if: (traffic logs) OR (debug AND low priority)
set(cache["exclude"], true) where attributes["type"] == "TRAFFIC" or (attributes["type"] == "DEBUG" and attributes["priority"] < 3)
```

### Negation with `not`

Invert conditions:

```yaml
# Simple negation
set(attributes["valid"], true) where not (attributes["status"] == "failed")

# Negation with compound condition
set(cache["process"], true) where not (attributes["log_type"] == "TRAFFIC" or attributes["log_type"] == "DEBUG")

# Double negative (avoid if possible)
set(cache["match"], true) where not (attributes["excluded"] != true)
```

### Field Existence Checks

Check if fields exist or are nil:

```yaml
# Field exists (not nil)
set(attributes["has_email"], true) where attributes["email"] != nil

# Field is nil (doesn't exist or null)
set(attributes["email"], "unknown@example.com") where attributes["email"] == nil

# Combined with other conditions
set(cache["valid"], true) where attributes["user_id"] != nil and attributes["user_id"] != ""
```

### Complex Conditional Examples

```yaml
# Multi-level filtering
set(cache["alert"], true) where attributes["environment"] == "production" and attributes["severity"] > 7 and (attributes["service"] == "auth" or attributes["service"] == "payment")

# Range checks with exclusions
set(cache["monitor"], true) where attributes["response_time"] > 100 and attributes["response_time"] < 5000 and attributes["endpoint"] != "/health"

# Status-based processing
set(attributes["retry"], true) where (attributes["status_code"] >= 500 and attributes["status_code"] < 600) or attributes["status_code"] == 429

# Type and value validation
set(cache["valid"], true) where attributes["event_type"] != nil and attributes["event_type"] != "" and attributes["timestamp"] != nil
```

---

## Field Access

### Bracket Notation

OTTL uses **bracket notation** for all field access. Dot notation is **not supported**.

**Field Types**:

| Field Type | Syntax | Description |
|------------|--------|-------------|
| `attributes` | `attributes["field_name"]` | Log/span/metric attributes |
| `resource` | `resource["field_name"]` | Resource attributes |
| `cache` | `cache["field_name"]` | Temporary storage (ephemeral) |
| `body` | `body` | Raw telemetry data (byte array) |

**Examples**:
```yaml
# Attributes access
set(attributes["user_id"], "12345")
set(attributes["log_level"], ToUpperCase(attributes["level"]))

# Resource attributes
set(resource["service.version"], "2.1.0")
set(resource["deployment.environment"], "production")

# Cache (temporary storage)
set(cache["parsed"], ParseJSON(Decode(body, "utf-8")))
set(cache["match_count"], 0)

# Body (raw data)
set(cache["body_str"], Decode(body, "utf-8"))
```

### Nested Fields

After parsing JSON or maps, access nested fields using chained brackets:

**Pattern**:
```yaml
# First parse JSON into cache
set(cache["parsed"], ParseJSON(Decode(body, "utf-8")))

# Then access nested fields
cache["parsed"]["level1"]["level2"]["field"]
```

**Examples**:
```yaml
# Parse JSON body
set(cache["json"], ParseJSON(Decode(body, "utf-8")))

# Access top-level field
set(attributes["user_id"], cache["json"]["userId"])

# Access nested object
set(attributes["email"], cache["json"]["user"]["profile"]["email"])

# Access deeply nested field
set(attributes["city"], cache["json"]["user"]["address"]["location"]["city"])

# Access array element (if supported)
set(attributes["first_tag"], cache["json"]["tags"][0])
```

### Field Naming Rules

**Valid field names**:
- Alphanumeric characters: `a-z`, `A-Z`, `0-9`
- Underscores: `_`
- Hyphens/dashes: `-`
- Dots: `.` (common in resource attributes like `service.name`)

**Examples**:
```yaml
# Standard names
attributes["user_id"]
attributes["log-level"]
attributes["eventType"]

# Dotted names (common in OpenTelemetry)
resource["service.name"]
resource["service.version"]
resource["deployment.environment"]

# Mixed formats
attributes["http.status_code"]
attributes["db.connection-pool.size"]
```

### Special Character Handling

If field names contain special characters, they must still use brackets:

```yaml
# Fields with dots
set(attributes["response.time.ms"], 150)

# Fields with dashes
set(attributes["X-Request-ID"], cache["request_id"])

# Fields with spaces (avoid if possible)
set(attributes["Event Type"], "user_login")
```

---

## Data Types

OTTL supports standard data types from OpenTelemetry:

### String

Text values enclosed in double quotes.

**Syntax**:
```yaml
"text value"
```

**Examples**:
```yaml
set(attributes["status"], "active")
set(attributes["message"], "User login successful")
set(attributes["level"], "ERROR")
```

**Rules**:
- Must be enclosed in double quotes
- Can contain spaces and special characters
- Empty strings are valid: `""`

### Int

Whole numbers (64-bit integers).

**Syntax**:
```yaml
123
-456
0
```

**Examples**:
```yaml
set(attributes["count"], 42)
set(attributes["retry_attempts"], 3)
set(attributes["status_code"], 200)
```

**Conversion**:
```yaml
# Convert string to int
set(attributes["count"], Int(attributes["count_str"]))

# Convert float to int (truncates)
set(attributes["rounded"], Int(attributes["price"]))
```

### Double

Floating-point numbers (64-bit).

**Syntax**:
```yaml
123.45
-67.89
0.0
```

**Examples**:
```yaml
set(attributes["price"], 19.99)
set(attributes["temperature"], 98.6)
set(attributes["rate"], 0.025)
```

**Conversion**:
```yaml
# Convert string to double
set(attributes["price"], Double(attributes["price_str"]))

# Convert int to double
set(attributes["percentage"], Double(attributes["count"]) / 100.0)
```

### Boolean

True or false values.

**Syntax**:
```yaml
true
false
```

**IMPORTANT**:
- Must be lowercase
- No quotes
- Case-sensitive

**Examples**:
```yaml
# ✅ CORRECT
set(attributes["is_active"], true)
set(attributes["has_error"], false)

# ❌ WRONG
set(attributes["is_active"], "true")   # String, not boolean
set(attributes["has_error"], TRUE)      # Uppercase not allowed
set(attributes["enabled"], False)       # Uppercase not allowed
```

**Conversion**:
```yaml
# String to boolean (if function exists)
set(attributes["flag"], Bool(attributes["flag_str"]))
```

### Byte Array

Raw binary data (body field).

**Key Characteristics**:
- The `body` field is always a byte array
- Must decode before string operations
- Use `Decode(body, "utf-8")` to convert to string

**Examples**:
```yaml
# Decode body to string
set(cache["body_str"], Decode(body, "utf-8"))

# Parse JSON from body
set(cache["json"], ParseJSON(Decode(body, "utf-8")))

# Search in body (after decoding)
set(cache["contains_error"], Contains(Decode(body, "utf-8"), "ERROR"))
```

### Map

Key-value pairs (objects/dictionaries).

**Examples**:
```yaml
# Parsed JSON becomes a map
set(cache["json"], ParseJSON(Decode(body, "utf-8")))

# Access map values
set(attributes["user_id"], cache["json"]["userId"])

# Check if key exists
set(attributes["has_email"], cache["json"]["email"] != nil)
```

**Operations**:
```yaml
# Merge maps
merge_maps(attributes, cache["additional_fields"])

# Delete key from map
delete_key(attributes, "sensitive_field")

# Keep only specific keys
keep_keys(attributes, ["user_id", "timestamp", "level"])
```

### Slice (Array)

Ordered collections of values.

**Examples**:
```yaml
# Concat function takes a slice of strings
set(attributes["full_name"], Concat([attributes["first"], " ", attributes["last"]]))

# Multiple values in array
set(attributes["tags"], Concat(["prod", "api", "v2"]))
```

**Array Access**:
```yaml
# Access array element (if supported)
set(attributes["first_item"], cache["json"]["items"][0])
```

### Nil (Null)

Represents absence of value.

**Usage**:
```yaml
# Check if field exists
set(attributes["has_user"], true) where attributes["user_id"] != nil

# Set default if nil
set(attributes["level"], "INFO") where attributes["level"] == nil

# Conditional assignment based on nil
set(attributes["name"], attributes["username"]) where attributes["username"] != nil
```

### Type Conversion Functions

| Function | Converts To | Example |
|----------|-------------|---------|
| `Int(value)` | Integer | `Int("123")` → `123` |
| `Double(value)` | Float | `Double("19.99")` → `19.99` |
| `String(value)` | String | `String(123)` → `"123"` |
| `Decode(bytes, encoding)` | String | `Decode(body, "utf-8")` |

**Examples**:
```yaml
# String to number
set(attributes["count"], Int(attributes["count_str"]))
set(attributes["price"], Double(attributes["price_str"]))

# Number to string
set(attributes["user_id"], String(attributes["uid_number"]))

# Byte array to string
set(cache["body"], Decode(body, "utf-8"))
```

---

## Special Fields

### body Field

The `body` field contains raw telemetry data and has special characteristics:

**Key Properties**:
- **Type**: Always a byte array
- **Content**: Raw log message, span data, or metric data
- **Encoding**: Typically UTF-8, but can vary

**Critical Rule**: Must decode before string operations!

**Common Pattern**:
```yaml
# ❌ WRONG - body is byte array, not string
set(cache["json"], ParseJSON(body))

# ✅ CORRECT - decode first
set(cache["json"], ParseJSON(Decode(body, "utf-8")))
```

**Examples**:
```yaml
# Decode body to string
set(cache["body_str"], Decode(body, "utf-8"))

# Parse JSON from body
set(cache["parsed"], ParseJSON(Decode(body, "utf-8")))
set(attributes["user"], cache["parsed"]["user"])

# Search in body
set(cache["has_error"], Contains(Decode(body, "utf-8"), "ERROR"))

# Extract substring from body
set(attributes["prefix"], Substring(Decode(body, "utf-8"), 0, 10))
```

**Optimization Tip**: Decode once, reuse multiple times:
```yaml
# Decode once, store in cache
set(cache["body"], Decode(body, "utf-8"))

# Use cached decoded value multiple times
set(cache["has_error"], Contains(cache["body"], "ERROR"))
set(cache["has_warning"], Contains(cache["body"], "WARN"))
set(cache["json"], ParseJSON(cache["body"]))
```

### cache Field

Temporary storage that exists only during processing of a single telemetry item.

**Key Properties**:
- **Lifetime**: Exists only during current processing pipeline
- **Scope**: Private to current telemetry item
- **Output**: Never appears in output metadata
- **Purpose**: Avoid repeated operations, store intermediate results

**When to Use cache**:
1. Store decoded body to avoid repeated decoding
2. Store parsed JSON to avoid repeated parsing
3. Store intermediate calculation results
4. Store temporary flags for conditional logic

**Examples**:
```yaml
# Store decoded body (avoid repeated Decode calls)
set(cache["body"], Decode(body, "utf-8"))
set(cache["json"], ParseJSON(cache["body"]))  # Use cached decoded body

# Store parsed JSON (access multiple fields)
set(cache["parsed"], ParseJSON(Decode(body, "utf-8")))
set(attributes["user_id"], cache["parsed"]["userId"])
set(attributes["timestamp"], cache["parsed"]["timestamp"])
set(attributes["level"], cache["parsed"]["level"])

# Store intermediate calculations
set(cache["total"], attributes["count"] * attributes["price"])
set(attributes["tax"], cache["total"] * 0.08)
set(attributes["grand_total"], cache["total"] + attributes["tax"])

# Store temporary flags
set(cache["is_error"], attributes["level"] == "ERROR")
set(cache["is_critical"], attributes["severity"] > 8)
set(attributes["alert"], "HIGH") where cache["is_error"] and cache["is_critical"]
```

**Important Notes**:
- Cache values **disappear** after processing completes
- Cache values **don't appear** in output attributes
- Cache is **not shared** between telemetry items
- Cache is **not persisted** (use EDXRedisSet for persistence)

**cache vs attributes**:

| Feature | cache | attributes |
|---------|-------|------------|
| Appears in output | ❌ No | ✅ Yes |
| Persists after processing | ❌ No | ✅ Yes |
| Visible in destinations | ❌ No | ✅ Yes |
| Use for temporary data | ✅ Yes | ❌ No |
| Use for metadata enrichment | ❌ No | ✅ Yes |

### resource Field

Resource attributes describe the source of telemetry data.

**Common Resource Attributes** (OpenTelemetry semantic conventions):
- `service.name` - Service name
- `service.version` - Service version
- `deployment.environment` - Environment (prod, dev, staging)
- `host.name` - Hostname
- `cloud.provider` - Cloud provider (aws, gcp, azure)
- `cloud.region` - Cloud region

**Examples**:
```yaml
# Read resource attributes
set(attributes["service"], resource["service.name"])
set(attributes["env"], resource["deployment.environment"])

# Conditional based on resource
set(attributes["monitor"], true) where resource["deployment.environment"] == "production"

# Set resource attributes (if needed)
set(resource["service.version"], "2.1.0")
set(resource["custom.tag"], "monitoring")
```

### attributes Field

Main metadata storage for telemetry items.

**Key Properties**:
- **Persists**: Appears in output and destinations
- **Searchable**: Can be indexed and queried
- **Type**: Map of key-value pairs

**Examples**:
```yaml
# Set attributes
set(attributes["user_id"], "12345")
set(attributes["request_id"], cache["parsed"]["requestId"])

# Modify existing attributes
set(attributes["level"], ToUpperCase(attributes["level"]))
set(attributes["count"], Int(attributes["count_str"]))

# Delete attributes
delete_key(attributes, "sensitive_data")

# Keep only specific attributes
keep_keys(attributes, ["user_id", "timestamp", "level", "message"])
```

---

## Critical Syntax Rules

### RULE 1: Lowercase Operators Only

**All OTTL keywords and operators MUST be lowercase**.

**Keywords**:
- ✅ `where`, `and`, `or`, `not`
- ❌ `WHERE`, `AND`, `OR`, `NOT`

**Examples**:
```yaml
# ✅ CORRECT
set(cache["match"], true) where attributes["level"] == "ERROR"
set(cache["alert"], true) where attributes["severity"] > 5 and attributes["urgent"] == true
set(cache["valid"], true) where not (attributes["status"] == "failed")

# ❌ WRONG - Will cause validation errors
set(cache["match"], true) WHERE attributes["level"] == "ERROR"
set(cache["alert"], true) where attributes["severity"] > 5 AND attributes["urgent"] == true
set(cache["valid"], true) where NOT (attributes["status"] == "failed")
```

### RULE 2: Single-Line Conditions in YAML

**All conditions must fit on one line**. YAML doesn't support multi-line conditions without special syntax.

**Examples**:
```yaml
# ✅ CORRECT - Single line
set(cache["match"], true) where attributes["level"] == "ERROR" and attributes["source"] == "api" and attributes["severity"] > 5

# ❌ WRONG - Multi-line will fail
set(cache["match"], true) where
  attributes["level"] == "ERROR" and
  attributes["source"] == "api"

# ❌ WRONG - YAML interprets this incorrectly
set(cache["match"], true) where attributes["level"] == "ERROR"
  and attributes["source"] == "api"
```

**Workaround for long conditions**: Use multiple statements instead:
```yaml
# Instead of one long condition, use multiple statements
set(cache["is_error"], true) where attributes["level"] == "ERROR"
set(cache["is_api"], true) where attributes["source"] == "api"
set(cache["is_severe"], true) where attributes["severity"] > 5
set(cache["match"], true) where cache["is_error"] and cache["is_api"] and cache["is_severe"]
```

### RULE 3: String Values Must Be Quoted

**All string literals must be enclosed in double quotes**.

**Examples**:
```yaml
# ✅ CORRECT
set(attributes["status"], "active")
set(attributes["level"], "ERROR")
set(attributes["message"], "User login successful")

# ❌ WRONG - Unquoted strings cause errors
set(attributes["status"], active)
set(attributes["level"], ERROR)
set(attributes["message"], User login successful)
```

**Special Cases**:
```yaml
# Boolean values - NO quotes
set(attributes["is_active"], true)    # ✅ CORRECT
set(attributes["is_active"], "true")  # ❌ WRONG - This is a string, not boolean

# Numbers - NO quotes
set(attributes["count"], 42)          # ✅ CORRECT - Integer
set(attributes["count"], "42")        # ❌ WRONG - This is a string
```

### RULE 4: Field Paths Use Brackets

**All field access must use bracket notation**. Dot notation is not supported.

**Examples**:
```yaml
# ✅ CORRECT - Bracket notation
attributes["field_name"]
cache["parsed"]["user"]["id"]
resource["service.name"]

# ❌ WRONG - Dot notation not supported
attributes.field_name
cache.parsed.user.id
resource.service.name
```

**Nested field access**:
```yaml
# ✅ CORRECT - Chain brackets
set(attributes["user_id"], cache["parsed"]["user"]["id"])
set(attributes["city"], cache["json"]["address"]["location"]["city"])

# ❌ WRONG - Mixed notation
set(attributes["user_id"], cache["parsed"].user.id)
set(attributes["city"], cache.json["address"]["location"]["city"])
```

### RULE 5: Boolean Values Lowercase, No Quotes

**Boolean values must be lowercase and unquoted**.

**Examples**:
```yaml
# ✅ CORRECT
set(attributes["is_active"], true)
set(attributes["has_error"], false)
set(cache["match"], true) where attributes["valid"] == true

# ❌ WRONG - Quoted (becomes string)
set(attributes["is_active"], "true")
set(attributes["has_error"], "false")

# ❌ WRONG - Uppercase
set(attributes["is_active"], True)
set(attributes["has_error"], FALSE)
set(cache["match"], TRUE) where attributes["valid"] == TRUE
```

### RULE 6: Decode Body Before String Operations

**The `body` field is a byte array and must be decoded before string operations**.

**Examples**:
```yaml
# ✅ CORRECT - Decode first
set(cache["json"], ParseJSON(Decode(body, "utf-8")))
set(cache["contains_error"], Contains(Decode(body, "utf-8"), "ERROR"))
set(attributes["message"], Decode(body, "utf-8"))

# ❌ WRONG - Operating on byte array
set(cache["json"], ParseJSON(body))
set(cache["contains_error"], Contains(body, "ERROR"))
```

**Best Practice**: Decode once, reuse multiple times:
```yaml
# Decode once
set(cache["body"], Decode(body, "utf-8"))

# Reuse decoded value
set(cache["json"], ParseJSON(cache["body"]))
set(cache["has_error"], Contains(cache["body"], "ERROR"))
set(attributes["length"], Len(cache["body"]))
```

### RULE 7: Function Names Are Case-Sensitive

**OTTL function names are case-sensitive**.

**Examples**:
```yaml
# ✅ CORRECT - Proper case
set(attributes["data"], ParseJSON(Decode(body, "utf-8")))
set(attributes["upper"], ToUpperCase(attributes["name"]))
set(attributes["count"], Int(attributes["value"]))

# ❌ WRONG - Incorrect case
set(attributes["data"], parseJSON(Decode(body, "utf-8")))
set(attributes["upper"], touppercase(attributes["name"]))
set(attributes["count"], int(attributes["value"]))
```

### RULE 8: Nil Checks for Optional Fields

**Check for nil before accessing optional fields**.

**Examples**:
```yaml
# ✅ CORRECT - Check for nil first
set(attributes["has_user"], true) where attributes["user_id"] != nil
set(attributes["name"], attributes["username"]) where attributes["username"] != nil

# Set default if nil
set(attributes["level"], "INFO") where attributes["level"] == nil

# ❌ RISKY - May fail if field doesn't exist
set(attributes["upper"], ToUpperCase(attributes["name"]))  # Fails if name is nil
```

**Safe pattern**:
```yaml
# Check existence before transformation
set(attributes["upper_name"], ToUpperCase(attributes["name"])) where attributes["name"] != nil
set(attributes["upper_name"], "UNKNOWN") where attributes["name"] == nil
```

---

## Common Patterns

### Pattern 1: Parse JSON and Extract Fields

**Use Case**: Extract structured data from JSON log body

**Implementation**:
```yaml
# Step 1: Decode body to string
set(cache["body"], Decode(body, "utf-8"))

# Step 2: Parse JSON
set(cache["json"], ParseJSON(cache["body"]))

# Step 3: Extract fields
set(attributes["user_id"], cache["json"]["userId"])
set(attributes["timestamp"], cache["json"]["timestamp"])
set(attributes["event_type"], cache["json"]["eventType"])
```

**Optimized Version** (fewer statements):
```yaml
# Combine decode and parse
set(cache["json"], ParseJSON(Decode(body, "utf-8")))

# Extract multiple fields
set(attributes["user_id"], cache["json"]["userId"])
set(attributes["event"], cache["json"]["eventType"])
set(attributes["level"], cache["json"]["level"])
```

**With Nested Fields**:
```yaml
set(cache["json"], ParseJSON(Decode(body, "utf-8")))
set(attributes["email"], cache["json"]["user"]["profile"]["email"])
set(attributes["city"], cache["json"]["user"]["address"]["city"])
set(attributes["country"], cache["json"]["user"]["address"]["country"])
```

### Pattern 2: Conditional Transformation

**Use Case**: Set attributes based on conditions

**Implementation**:
```yaml
# Set severity levels
set(attributes["level"], "CRITICAL") where attributes["severity"] > 8
set(attributes["level"], "WARNING") where attributes["severity"] > 4 and attributes["severity"] <= 8
set(attributes["level"], "INFO") where attributes["severity"] <= 4

# Set alert flags
set(attributes["alert"], true) where attributes["level"] == "CRITICAL" and resource["deployment.environment"] == "production"

# Categorize response times
set(attributes["latency_category"], "fast") where attributes["response_time"] < 100
set(attributes["latency_category"], "medium") where attributes["response_time"] >= 100 and attributes["response_time"] < 500
set(attributes["latency_category"], "slow") where attributes["response_time"] >= 500
```

### Pattern 3: Type Conversion

**Use Case**: Convert string fields to appropriate types

**Implementation**:
```yaml
# String to integer
set(attributes["count"], Int(attributes["count_str"]))
set(attributes["status_code"], Int(attributes["status_str"]))

# String to double
set(attributes["price"], Double(attributes["price_str"]))
set(attributes["temperature"], Double(attributes["temp_str"]))

# Time conversion
set(attributes["timestamp_ms"], UnixMilli(Time(attributes["timestamp_str"], "%Y-%m-%dT%H:%M:%S")))

# Boolean conversion (if supported)
set(attributes["is_active"], Bool(attributes["active_str"]))
```

**With Error Handling** (check for nil):
```yaml
# Safe conversion
set(attributes["count"], Int(attributes["count_str"])) where attributes["count_str"] != nil
set(attributes["count"], 0) where attributes["count_str"] == nil
```

### Pattern 4: Field Existence and Fallback

**Use Case**: Set defaults when fields are missing

**Implementation**:
```yaml
# Use existing value if present, otherwise set default
set(attributes["name"], attributes["username"]) where attributes["username"] != nil
set(attributes["name"], "unknown") where attributes["username"] == nil

# Multiple fallback options
set(attributes["identifier"], attributes["user_id"]) where attributes["user_id"] != nil
set(attributes["identifier"], attributes["session_id"]) where attributes["user_id"] == nil and attributes["session_id"] != nil
set(attributes["identifier"], "anonymous") where attributes["user_id"] == nil and attributes["session_id"] == nil

# Copy with transformation if exists
set(attributes["upper_level"], ToUpperCase(attributes["level"])) where attributes["level"] != nil
set(attributes["upper_level"], "UNKNOWN") where attributes["level"] == nil
```

### Pattern 5: String Manipulation

**Use Case**: Combine, transform, and extract string data

**Implementation**:
```yaml
# Concatenate strings
set(attributes["full_name"], Concat([attributes["first_name"], " ", attributes["last_name"]]))
set(attributes["full_address"], Concat([attributes["street"], ", ", attributes["city"], ", ", attributes["state"]]))

# Case conversion
set(attributes["upper_level"], ToUpperCase(attributes["level"]))
set(attributes["lower_email"], ToLowerCase(attributes["email"]))

# Extract substring
set(attributes["area_code"], Substring(attributes["phone"], 0, 3))
set(attributes["first_10"], Substring(Decode(body, "utf-8"), 0, 10))

# Trim whitespace (if function exists)
set(attributes["clean_name"], Trim(attributes["name"]))

# Replace text
replace_match(attributes["message"], "password", "****")
replace_pattern(attributes["email"], "@.*", "@redacted.com")
```

### Pattern 6: Filtering and Exclusion

**Use Case**: Mark items for filtering or exclusion

**Implementation**:
```yaml
# Mark for exclusion
set(cache["exclude"], true) where attributes["log_type"] == "TRAFFIC"
set(cache["exclude"], true) where attributes["log_type"] == "DEBUG" and resource["deployment.environment"] == "development"

# Mark for inclusion (opposite logic)
set(cache["include"], true) where attributes["level"] == "ERROR" or attributes["level"] == "CRITICAL"
set(cache["include"], true) where attributes["severity"] > 5 and resource["deployment.environment"] == "production"

# Multi-condition exclusion
set(cache["exclude"], true) where attributes["log_type"] != "APPLICATION" and attributes["log_type"] != "ERROR"
```

### Pattern 7: Metadata Enrichment

**Use Case**: Add context and metadata to telemetry

**Implementation**:
```yaml
# Add environment context
set(attributes["environment"], resource["deployment.environment"])
set(attributes["service"], resource["service.name"])
set(attributes["version"], resource["service.version"])

# Add derived metadata
set(attributes["is_production"], true) where resource["deployment.environment"] == "production"
set(attributes["is_critical"], true) where attributes["severity"] > 8
set(attributes["requires_alert"], true) where attributes["is_production"] and attributes["is_critical"]

# Add timestamps
set(attributes["processed_at"], Now())
set(attributes["ingestion_time"], UnixMilli(Now()))

# Add categorization
set(attributes["category"], "security") where Contains(Decode(body, "utf-8"), "authentication") or Contains(Decode(body, "utf-8"), "authorization")
set(attributes["category"], "performance") where attributes["response_time"] != nil
```

### Pattern 8: Data Cleanup

**Use Case**: Remove sensitive data and clean up attributes

**Implementation**:
```yaml
# Delete sensitive fields
delete_key(attributes, "password")
delete_key(attributes, "api_key")
delete_key(attributes, "secret")

# Delete matching pattern
delete_matching_keys(attributes, ".*_token")
delete_matching_keys(attributes, ".*_secret")

# Keep only specific fields
keep_keys(attributes, ["user_id", "timestamp", "level", "message", "request_id"])

# Redact sensitive data
replace_match(attributes["url"], "apikey=.*", "apikey=REDACTED")
replace_pattern(attributes["message"], "password=\\S+", "password=****")
```

### Pattern 9: Aggregation and Calculation

**Use Case**: Perform calculations and create derived values

**Implementation**:
```yaml
# Simple math
set(cache["total"], attributes["quantity"] * attributes["unit_price"])
set(attributes["tax"], cache["total"] * 0.08)
set(attributes["grand_total"], cache["total"] + attributes["tax"])

# Percentages
set(attributes["error_rate"], Double(attributes["error_count"]) / Double(attributes["total_count"]) * 100.0)

# Time calculations (if supported)
set(cache["duration_ms"], attributes["end_time"] - attributes["start_time"])
set(attributes["duration_sec"], cache["duration_ms"] / 1000)

# Conditional calculation
set(attributes["discount"], cache["total"] * 0.10) where attributes["is_member"] == true and cache["total"] > 100
```

### Pattern 10: Multi-Step Processing

**Use Case**: Complex processing with multiple stages

**Implementation**:
```yaml
# Stage 1: Parse and decode
set(cache["body"], Decode(body, "utf-8"))
set(cache["json"], ParseJSON(cache["body"]))

# Stage 2: Extract core fields
set(attributes["event_type"], cache["json"]["eventType"])
set(attributes["user_id"], cache["json"]["userId"])
set(attributes["timestamp"], cache["json"]["timestamp"])

# Stage 3: Derive metadata
set(cache["is_error"], attributes["event_type"] == "error")
set(cache["is_production"], resource["deployment.environment"] == "production")

# Stage 4: Set alerts
set(attributes["alert_level"], "high") where cache["is_error"] and cache["is_production"]
set(attributes["alert_level"], "low") where cache["is_error"] and not cache["is_production"]

# Stage 5: Cleanup
delete_key(attributes, "internal_metadata")
keep_keys(attributes, ["event_type", "user_id", "timestamp", "alert_level"])
```

---

## Syntax Errors and Fixes

### Error 1: Uppercase Operator

**Problem**: Using uppercase operators like `WHERE`, `AND`, `OR`, `NOT`

```yaml
# ❌ WRONG - Uppercase WHERE
set(cache["match"], true) WHERE attributes["level"] == "ERROR"

# ✅ CORRECT - Lowercase where
set(cache["match"], true) where attributes["level"] == "ERROR"
```

```yaml
# ❌ WRONG - Uppercase AND
set(cache["alert"], true) where attributes["severity"] > 5 AND attributes["urgent"] == true

# ✅ CORRECT - Lowercase and
set(cache["alert"], true) where attributes["severity"] > 5 and attributes["urgent"] == true
```

### Error 2: Multi-Line Condition

**Problem**: Attempting to split conditions across multiple lines in YAML

```yaml
# ❌ WRONG - Multi-line condition
set(cache["match"], true) where
  attributes["level"] == "ERROR" and
  attributes["source"] == "api"

# ✅ CORRECT - Single line
set(cache["match"], true) where attributes["level"] == "ERROR" and attributes["source"] == "api"
```

```yaml
# ❌ WRONG - YAML indentation confusion
set(cache["match"], true) where attributes["level"] == "ERROR"
  and attributes["source"] == "api"
  and attributes["severity"] > 5

# ✅ CORRECT - All on one line
set(cache["match"], true) where attributes["level"] == "ERROR" and attributes["source"] == "api" and attributes["severity"] > 5
```

**Alternative**: Break into multiple statements:
```yaml
# ✅ ACCEPTABLE - Multiple statements
set(cache["is_error"], true) where attributes["level"] == "ERROR"
set(cache["is_api"], true) where attributes["source"] == "api"
set(cache["is_severe"], true) where attributes["severity"] > 5
set(cache["match"], true) where cache["is_error"] and cache["is_api"] and cache["is_severe"]
```

### Error 3: Unquoted String

**Problem**: Using string values without quotes

```yaml
# ❌ WRONG - Unquoted string
set(attributes["status"], error)
set(attributes["level"], ERROR)

# ✅ CORRECT - Quoted string
set(attributes["status"], "error")
set(attributes["level"], "ERROR")
```

```yaml
# ❌ WRONG - Boolean as string
set(attributes["is_active"], "true")  # This is a string, not boolean

# ✅ CORRECT - Boolean without quotes
set(attributes["is_active"], true)
```

### Error 4: Dot Notation Instead of Brackets

**Problem**: Using dot notation for field access

```yaml
# ❌ WRONG - Dot notation
set(attributes["user"], cache.parsed.user)
set(attributes["level"], attributes.level)

# ✅ CORRECT - Bracket notation
set(attributes["user"], cache["parsed"]["user"])
set(attributes["level"], attributes["level"])
```

```yaml
# ❌ WRONG - Mixed notation
set(attributes["email"], cache["json"].user.profile.email)

# ✅ CORRECT - All brackets
set(attributes["email"], cache["json"]["user"]["profile"]["email"])
```

### Error 5: Parsing Body Without Decode

**Problem**: Attempting to parse body as string when it's a byte array

```yaml
# ❌ WRONG - Body is byte array, not string
set(cache["json"], ParseJSON(body))

# ✅ CORRECT - Decode first
set(cache["json"], ParseJSON(Decode(body, "utf-8")))
```

```yaml
# ❌ WRONG - String operations on byte array
set(cache["contains_error"], Contains(body, "ERROR"))

# ✅ CORRECT - Decode before string operation
set(cache["contains_error"], Contains(Decode(body, "utf-8"), "ERROR"))
```

**Best Practice**: Decode once, reuse:
```yaml
# ✅ OPTIMAL - Decode once
set(cache["body"], Decode(body, "utf-8"))
set(cache["json"], ParseJSON(cache["body"]))
set(cache["has_error"], Contains(cache["body"], "ERROR"))
```

### Error 6: Incorrect Boolean Casing

**Problem**: Using uppercase or quoted boolean values

```yaml
# ❌ WRONG - Uppercase boolean
set(attributes["is_active"], True)
set(attributes["has_error"], FALSE)

# ✅ CORRECT - Lowercase boolean
set(attributes["is_active"], true)
set(attributes["has_error"], false)
```

```yaml
# ❌ WRONG - Quoted boolean (becomes string)
set(attributes["enabled"], "true")
set(cache["match"], "false")

# ✅ CORRECT - Unquoted lowercase
set(attributes["enabled"], true)
set(cache["match"], false)
```

### Error 7: Missing Nil Check

**Problem**: Not checking if field exists before using it

```yaml
# ❌ RISKY - May fail if field is nil
set(attributes["upper_name"], ToUpperCase(attributes["name"]))

# ✅ SAFE - Check for nil first
set(attributes["upper_name"], ToUpperCase(attributes["name"])) where attributes["name"] != nil
set(attributes["upper_name"], "UNKNOWN") where attributes["name"] == nil
```

```yaml
# ❌ RISKY - Accessing nested field without checking
set(attributes["email"], cache["json"]["user"]["email"])

# ✅ SAFE - Check parent exists
set(attributes["email"], cache["json"]["user"]["email"]) where cache["json"]["user"] != nil
set(attributes["email"], "unknown") where cache["json"]["user"] == nil
```

### Error 8: Incorrect Function Casing

**Problem**: Using wrong case for function names

```yaml
# ❌ WRONG - Incorrect function casing
set(cache["json"], parseJSON(Decode(body, "utf-8")))
set(attributes["upper"], TOUPPERCASE(attributes["name"]))
set(attributes["count"], int(attributes["value"]))

# ✅ CORRECT - Proper function casing
set(cache["json"], ParseJSON(Decode(body, "utf-8")))
set(attributes["upper"], ToUpperCase(attributes["name"]))
set(attributes["count"], Int(attributes["value"]))
```

### Error 9: Invalid Field Names

**Problem**: Using invalid characters in field names or incorrect syntax

```yaml
# ❌ WRONG - Invalid field syntax
set(attributes[level], "ERROR")  # Missing quotes
set(attributes['level'], "ERROR")  # Single quotes not supported

# ✅ CORRECT - Proper field syntax
set(attributes["level"], "ERROR")
```

```yaml
# ❌ WRONG - Spaces without quotes
set(attributes[log level], "ERROR")

# ✅ CORRECT - Quotes with spaces (though avoid spaces if possible)
set(attributes["log level"], "ERROR")
```

### Error 10: Operator Precedence Issues

**Problem**: Not understanding operator precedence

```yaml
# ❌ UNCLEAR - May not work as expected
set(cache["alert"], true) where attributes["severity"] > 5 and attributes["level"] == "ERROR" or attributes["urgent"] == true

# ✅ CLEAR - Use parentheses
set(cache["alert"], true) where attributes["severity"] > 5 and (attributes["level"] == "ERROR" or attributes["urgent"] == true)
```

```yaml
# ❌ WRONG - NOT has higher precedence than AND
set(cache["exclude"], true) where not attributes["level"] == "ERROR" and attributes["source"] == "api"
# This evaluates as: (not (attributes["level"] == "ERROR")) and (attributes["source"] == "api")

# ✅ CORRECT - Use parentheses for clarity
set(cache["exclude"], true) where not (attributes["level"] == "ERROR" and attributes["source"] == "api")
```

---

## Cross-References

### Related Documentation

- **[MASTER_INDEX.md](../../MASTER_INDEX.md)** - Complete index of all 124 OTTL functions (101 standard + 23 EDX extensions)
- **[validation_rules.md](../../validation_rules.md)** - Strict syntax validation rules and error checking
- **[edxredis.md](../edx/edxredis.md)** - Comprehensive EDXRedis reference for stateful processing

### Function Categories

- **[Standard Functions](../standard/)** - All 101 standard OTTL functions
- **[EDX Extensions](../edx/)** - All 23 EdgeDelta custom extensions
- **[Syntax Reference](../syntax/)** - This document and related syntax guides

### Additional Resources

- **OpenTelemetry OTTL Documentation**: Official OTTL specification
- **EdgeDelta Pipeline v3 Docs**: EdgeDelta-specific implementation details
- **EdgeDelta Examples**: Production examples using OTTL in pipelines

---

## Quick Reference Card

### Essential Syntax Rules

| Rule | ✅ Correct | ❌ Wrong |
|------|-----------|----------|
| Operators | `where`, `and`, `or`, `not` | `WHERE`, `AND`, `OR`, `NOT` |
| Strings | `"value"` | `value` (unquoted) |
| Booleans | `true`, `false` | `"true"`, `TRUE`, `False` |
| Field Access | `attributes["field"]` | `attributes.field` |
| Body Decode | `Decode(body, "utf-8")` | `body` (in string ops) |
| Conditions | Single line | Multi-line split |

### Common Function Patterns

```yaml
# Parse JSON from body
set(cache["json"], ParseJSON(Decode(body, "utf-8")))

# Extract nested field
set(attributes["value"], cache["json"]["parent"]["child"])

# Conditional assignment
set(attributes["level"], "HIGH") where attributes["severity"] > 8

# Type conversion
set(attributes["count"], Int(attributes["count_str"]))

# String concatenation
set(attributes["full"], Concat([attributes["first"], " ", attributes["last"]]))

# Nil check with fallback
set(attributes["name"], attributes["username"]) where attributes["username"] != nil
set(attributes["name"], "unknown") where attributes["username"] == nil
```

### Operator Precedence (High to Low)

1. Parentheses `()`
2. NOT `not`
3. Comparison `==`, `!=`, `>`, `<`, `>=`, `<=`
4. AND `and`
5. OR `or`

---

**End of OTTL Syntax Guide**
