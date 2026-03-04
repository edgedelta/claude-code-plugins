# EDXEnv - Access Environment Variables

**Function Type**: EDX Converter
**Purpose**: Retrieves environment variable values with optional default fallback
**Signature**: `EDXEnv(env_var_name, default_value) -> string`
**Return Type**: string
**Minimum Agent Version**: v2.5.0
**Source**: https://docs.edgedelta.com/ottl-extensions/

---

## Quick Copy

```yaml
# Get environment variable with fallback
set(attributes["api_url"], EDXEnv("API_ENDPOINT", "https://api.default.com"))
```

---

## Overview

EDXEnv provides access to environment variables in OTTL pipelines. Essential for:
- External configuration management
- Environment-specific settings
- Secret injection from environment
- Dynamic pipeline configuration

**Key Features:**
- Safe environment variable access
- Optional default values
- Empty string handling
- Runtime configuration

---

## Parameters

### env_var_name
- **Type**: string
- **Required**: Yes
- **Description**: Name of environment variable to read
- **Case Sensitive**: Yes
- **Examples**: `"API_KEY"`, `"LOG_LEVEL"`, `"REGION"`

### default_value
- **Type**: string
- **Required**: No
- **Default**: ""
- **Description**: Value to return if variable not found or empty
- **Fallback Behavior**: Returns default if variable unset or empty string

---

## Return Value

Returns string value of environment variable or default.

**Return Logic:**
1. If env var exists and non-empty → return value
2. If env var missing or empty → return default
3. If no default provided → return ""

---

## Examples

### Example 1: API Configuration
```yaml
processors:
  - name: api-config
    transform:
      statements:
        - set(attributes["api_endpoint"], EDXEnv("API_ENDPOINT", "https://api.prod.com"))
        - set(attributes["api_key"], EDXEnv("API_KEY", ""))
```

### Example 2: Environment-Specific Routing
```yaml
processors:
  - name: env-routing
    transform:
      statements:
        - set(cache["env"], EDXEnv("ENVIRONMENT", "production"))
        - set(cache["is_prod"], cache["env"] == "production")
        - set(attributes["destination"], EDXIfElse(
            cache["is_prod"],
            "prod-endpoint",
            "staging-endpoint"
          ))
```

### Example 3: Log Level Configuration
```yaml
processors:
  - name: log-config
    transform:
      statements:
        - set(attributes["log_level"], EDXEnv("LOG_LEVEL", "INFO"))
        - set(cache["debug_enabled"], attributes["log_level"] == "DEBUG")
```

### Example 4: Secret Injection
```yaml
processors:
  - name: inject-secrets
    transform:
      statements:
        - set(cache["db_password"], EDXEnv("DB_PASSWORD", ""))
        - set(cache["encryption_key"], EDXEnv("ENCRYPTION_KEY", ""))
```

### Example 5: Region Configuration
```yaml
processors:
  - name: region-config
    transform:
      statements:
        - set(attributes["aws_region"], EDXEnv("AWS_REGION", "us-east-1"))
        - set(attributes["az"], EDXEnv("AWS_AZ", "us-east-1a"))
```

### Example 6: Feature Flags
```yaml
processors:
  - name: feature-flags
    transform:
      statements:
        - set(cache["feature_enabled"], EDXEnv("ENABLE_FEATURE_X", "false"))
        - set(cache["should_process"], cache["feature_enabled"] == "true")
```

### Example 7: Combine with Coalesce
```yaml
processors:
  - name: config-hierarchy
    transform:
      statements:
        - set(attributes["timeout"], EDXCoalesce(
            attributes["config.timeout"],
            EDXEnv("DEFAULT_TIMEOUT", "30")
          ))
```

### Example 8: Build Info
```yaml
processors:
  - name: build-info
    transform:
      statements:
        - set(attributes["version"], EDXEnv("BUILD_VERSION", "unknown"))
        - set(attributes["commit"], EDXEnv("GIT_COMMIT", ""))
```

---

## Common Use Cases

### 1. External Configuration
```yaml
processors:
  - name: external-config
    transform:
      statements:
        - set(attributes["kafka_brokers"], EDXEnv("KAFKA_BROKERS", "localhost:9092"))
        - set(attributes["redis_url"], EDXEnv("REDIS_URL", "redis://localhost:6379"))
        - set(attributes["batch_size"], EDXEnv("BATCH_SIZE", "100"))
```

### 2. Secrets Management
```yaml
processors:
  - name: secrets
    transform:
      statements:
        - set(cache["api_key"], EDXEnv("API_KEY", ""))
        - set(cache["has_key"], cache["api_key"] != "")
        - set(attributes["auth_header"], EDXIfElse(
            cache["has_key"],
            Concat(["Bearer ", cache["api_key"]]),
            ""
          ))
```

### 3. Environment Detection
```yaml
processors:
  - name: env-detection
    transform:
      statements:
        - set(cache["env"], EDXEnv("ENVIRONMENT", "production"))
        - set(attributes["is_dev"], cache["env"] == "development")
        - set(attributes["is_prod"], cache["env"] == "production")
```

---

## Validation Rules

1. **Name Required**: Environment variable name must be provided
2. **String Only**: Returns string (use Int() to convert if needed)
3. **Empty Check**: Empty string treated as missing
4. **Case Sensitive**: Environment variable names are case-sensitive

---

## Common Pitfalls

### 1. Type Conversion
```yaml
# WRONG: Treating as number
set(cache["port"], EDXEnv("PORT", "8080"))
set(attributes["port_num"], cache["port"] + 1)  # ERROR: string + int

# CORRECT: Convert to int
set(cache["port_str"], EDXEnv("PORT", "8080"))
set(attributes["port_num"], Int(cache["port_str"]))
```

### 2. Empty String vs Missing
```yaml
# Both return default
# export EMPTY_VAR=""
set(cache["value"], EDXEnv("EMPTY_VAR", "default"))  # Returns "default"
set(cache["value"], EDXEnv("MISSING_VAR", "default"))  # Returns "default"
```

### 3. Security - Logging Secrets
```yaml
# DANGEROUS: Logging secret
set(attributes["api_key"], EDXEnv("API_KEY", ""))
# api_key now in logs!

# BETTER: Use cache
set(cache["api_key"], EDXEnv("API_KEY", ""))
# Use cache["api_key"] but don't log
```

---

## Best Practices

### 1. Provide Sensible Defaults
```yaml
set(attributes["timeout"], EDXEnv("TIMEOUT", "30"))
set(attributes["retry_count"], EDXEnv("RETRIES", "3"))
```

### 2. Use for Environment-Specific Config
```yaml
set(cache["env"], EDXEnv("ENVIRONMENT", "production"))
set(cache["endpoint"], EDXIfElse(
  cache["env"] == "production",
  EDXEnv("PROD_ENDPOINT", ""),
  EDXEnv("DEV_ENDPOINT", "")
))
```

### 3. Validate Required Variables
```yaml
set(cache["api_key"], EDXEnv("API_KEY", ""))
set(cache["has_key"], cache["api_key"] != "")
# Assert cache["has_key"] or error
```

### 4. Keep Secrets in Cache
```yaml
# Don't expose secrets in attributes
set(cache["secret"], EDXEnv("SECRET", ""))
# Use cache["secret"] for processing only
```

---

## Security Considerations

1. **Never Log Secrets**: Don't set secrets in attributes
2. **Use Cache**: Store sensitive env vars in cache only
3. **Validate Presence**: Check that required secrets exist
4. **Environment Isolation**: Use different variables per environment

---

## Performance Considerations

- **Very Fast**: O(1) environment variable lookup
- **No Caching**: Each call reads from environment
- **Minimal Overhead**: Negligible performance impact

---

## Related Functions

- **EDXCoalesce**: Combine with EDXEnv for fallback chains
- **EDXIfElse**: Use for environment-based conditionals
- **EDXDecrypt**: Decrypt encrypted environment variables

---

## Cross-References

- [MASTER_INDEX.md](../../MASTER_INDEX.md#edxenv)
- [EDXCoalesce](./edxcoalesce.md)
- [EDXIfElse](./edxifelse.md)

---

## Notes

- **Version**: Introduced in v2.5.0
- **Platform**: Works on all platforms
- **Thread Safety**: Thread-safe
- **Caching**: No internal caching, reads environment each call
