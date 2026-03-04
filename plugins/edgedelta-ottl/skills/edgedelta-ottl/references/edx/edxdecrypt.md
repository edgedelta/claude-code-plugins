# EDXDecrypt - Decrypt Encrypted Data

**Function Type**: EDX Converter
**Purpose**: Decrypts data from base64-encoded JSON envelope using AES-256-CBC/GCM with local key management
**Signature**: `EDXDecrypt(encryptedValue, escape, escapeSeq) -> string`
**Return Type**: string (decrypted plaintext)
**Minimum Agent Version**: v2.5.0
**Source**: https://docs.edgedelta.com/ottl-extensions/

---

## Quick Copy

```yaml
# Decrypt sensitive field
set(attributes["ssn"], EDXDecrypt(attributes["encrypted_ssn"], false, ""))
```

---

## Overview

EDXDecrypt decrypts data that was encrypted using EDXEncrypt. This enables secure handling of sensitive data (PII, credentials, secrets) in observability pipelines.

**Key Features:**
- AES-256-CBC and AES-256-GCM encryption support
- Base64-encoded JSON envelope format
- Local key management via ED_CRYPTO_PATH
- Automatic key rotation support
- Escape sequence handling for output

**Encryption Envelope Format:**
```json
{
  "v": 1,
  "keyId": "prod-key-001",
  "data": "base64-encrypted-bytes",
  "iv": "base64-initialization-vector"
}
```

**Prerequisites:**
1. `ED_CRYPTO_PATH` environment variable set
2. Keys file at `$ED_CRYPTO_PATH/auth/keys.json`
3. Matching encryption keys available

---

## Parameters

### encryptedValue
- **Type**: string
- **Required**: Yes
- **Description**: Base64-encoded JSON envelope containing encrypted data
- **Format**: Must be output from EDXEncrypt function
- **Components**:
  - `v` (version): Envelope format version (currently 1)
  - `keyId`: Identifier of encryption key used
  - `data`: Base64-encoded ciphertext
  - `iv`: Base64-encoded initialization vector

### escape
- **Type**: bool
- **Required**: No
- **Default**: false
- **Description**: Whether to escape special characters in the decrypted output
- **Use Cases**: Escape quotes/newlines for JSON insertion

### escapeSeq
- **Type**: string
- **Required**: No
- **Default**: `"\""`
- **Description**: Escape sequence to prepend to special characters
- **Common Values**:
  - `"\\"` - Backslash escape for JSON
  - `"\"` - Quote escape (default)
  - `""` - No escape sequence

---

## Return Value

Returns the decrypted plaintext as a string.

**Success Cases:**
- Returns original unencrypted value
- Applies escape sequences if requested
- Maintains UTF-8 encoding

**Error Cases:**
- Missing/invalid ED_CRYPTO_PATH → error
- Key not found in keys.json → error
- Invalid envelope format → error
- Decryption failure (wrong key, corrupted data) → error
- Malformed base64 → error

---

## Environment Setup

### ED_CRYPTO_PATH Configuration
```bash
# Set environment variable
export ED_CRYPTO_PATH=/opt/edgedelta/crypto

# Create directory structure
mkdir -p /opt/edgedelta/crypto/auth

# Create keys.json
cat > /opt/edgedelta/crypto/auth/keys.json <<EOF
{
  "keys": [
    {
      "keyclass": 1,
      "keyId": "prod-key-001",
      "key": "base64-encoded-32-byte-key",
      "algorithm": "AES-256-GCM"
    },
    {
      "keyclass": 1,
      "keyId": "prod-key-002",
      "key": "base64-encoded-32-byte-key",
      "algorithm": "AES-256-CBC"
    }
  ]
}
EOF
```

### Keys.json Schema
```json
{
  "keys": [
    {
      "keyclass": 1,              // Key class identifier (integer)
      "keyId": "unique-key-id",   // Unique key identifier (string)
      "key": "base64-key-data",   // Base64-encoded 32-byte key
      "algorithm": "AES-256-GCM"  // AES-256-GCM or AES-256-CBC
    }
  ]
}
```

---

## Examples

### Example 1: Basic Decryption
```yaml
# Decrypt SSN field
processors:
  - name: decrypt-pii
    transform:
      statements:
        - set(attributes["ssn_plaintext"], EDXDecrypt(
            attributes["ssn_encrypted"],
            false,
            ""
          ))
```

### Example 2: Decrypt with Escaping for JSON
```yaml
# Decrypt and escape for JSON embedding
processors:
  - name: decrypt-escaped
    transform:
      statements:
        - set(cache["decrypted"], EDXDecrypt(
            attributes["encrypted_note"],
            true,
            "\\"
          ))
        - set(attributes["note_json"], Concat(["{\"note\":\"", cache["decrypted"], "\"}"]))
```

### Example 3: Conditional Decryption
```yaml
# Only decrypt if field is encrypted
processors:
  - name: conditional-decrypt
    transform:
      statements:
        - set(cache["is_encrypted"], attributes["encryption_status"] == "encrypted")
        - set(attributes["api_key"], EDXIfElse(
            cache["is_encrypted"],
            EDXDecrypt(attributes["api_key"], false, ""),
            attributes["api_key"]
          ))
```

### Example 4: Batch Decryption
```yaml
# Decrypt multiple PII fields
processors:
  - name: decrypt-pii-batch
    transform:
      statements:
        - set(attributes["ssn"], EDXDecrypt(attributes["ssn_enc"], false, ""))
        - set(attributes["email"], EDXDecrypt(attributes["email_enc"], false, ""))
        - set(attributes["phone"], EDXDecrypt(attributes["phone_enc"], false, ""))
```

### Example 5: Decrypt and Parse JSON
```yaml
# Decrypt JSON payload
processors:
  - name: decrypt-json
    transform:
      statements:
        - set(cache["json_str"], EDXDecrypt(attributes["encrypted_payload"], false, ""))
        - set(cache["parsed"], ParseJSON(cache["json_str"]))
        - set(attributes["user_id"], cache["parsed"]["userId"])
```

### Example 6: Error Handling with Fallback
```yaml
# Decrypt with error handling
processors:
  - name: safe-decrypt
    transform:
      statements:
        - set(cache["decrypted"], EDXDecrypt(attributes["encrypted"], false, "") or "<decryption_failed>")
        - set(cache["success"], cache["decrypted"] != "<decryption_failed>")
        - set(attributes["decryption_status"], EDXIfElse(
            cache["success"],
            "success",
            "failed"
          ))
```

### Example 7: Decrypt from Redis Cache
```yaml
# Retrieve and decrypt from Redis
processors:
  - name: redis-decrypt
    transform:
      statements:
        - set(cache["redis_result"], EDXRedis(
            {"url": "redis://localhost:6379"},
            [{"command": "get", "key": attributes["key"], "outField": "encrypted"}]
          ))
        - set(attributes["secret"], EDXDecrypt(
            cache["redis_result"]["encrypted"],
            false,
            ""
          ))
```

### Example 8: Decrypt and Re-encrypt with New Key
```yaml
# Key rotation: decrypt with old key, encrypt with new key
processors:
  - name: rotate-encryption
    transform:
      statements:
        - set(cache["plaintext"], EDXDecrypt(attributes["old_encrypted"], false, ""))
        - set(attributes["new_encrypted"], EDXEncrypt(
            cache["plaintext"],
            1,
            "new-key-002",
            ""
          ))
```

### Example 9: Decrypt URL-encoded Encrypted Data
```yaml
# Decrypt data from URL parameters
processors:
  - name: decrypt-url-param
    transform:
      statements:
        - set(cache["url_decoded"], EDXDecode(attributes["encrypted_param"], "url"))
        - set(cache["decrypted"], EDXDecrypt(cache["url_decoded"], false, ""))
```

### Example 10: Audit Logging with Decryption
```yaml
# Decrypt for audit log, mask for regular logs
processors:
  - name: audit-decrypt
    transform:
      statements:
        - set(cache["is_audit"], attributes["log_type"] == "audit")
        - set(attributes["sensitive_data"], EDXIfElse(
            cache["is_audit"],
            EDXDecrypt(attributes["encrypted_value"], false, ""),
            "***REDACTED***"
          ))
```

---

## Common Use Cases

### 1. PII Decryption for Processing
**Scenario**: Decrypt sensitive fields for enrichment or validation.

```yaml
processors:
  - name: decrypt-for-processing
    transform:
      statements:
        - set(cache["ssn"], EDXDecrypt(attributes["ssn_encrypted"], false, ""))
        - set(cache["email"], EDXDecrypt(attributes["email_encrypted"], false, ""))

        # Process decrypted data
        - set(attributes["ssn_last4"], Substring(cache["ssn"], -4, 4))
        - set(attributes["email_domain"], Split(cache["email"], "@")[1])

        # Don't store plaintext
        - delete_key(cache, "ssn")
        - delete_key(cache, "email")
```

### 2. Secure Credential Management
**Scenario**: Decrypt API keys/tokens for external API calls.

```yaml
processors:
  - name: decrypt-credentials
    transform:
      statements:
        - set(cache["api_key"], EDXDecrypt(
            EDXEnv("ENCRYPTED_API_KEY", ""),
            false,
            ""
          ))
        - set(cache["api_secret"], EDXDecrypt(
            EDXEnv("ENCRYPTED_API_SECRET", ""),
            false,
            ""
          ))
        # Use credentials for API call
```

### 3. Compliance Logging
**Scenario**: Decrypt only for compliance/audit logs.

```yaml
processors:
  - name: compliance-decrypt
    transform:
      statements:
        - set(cache["is_compliance"], attributes["log_category"] == "compliance")
        - set(cache["payment_info"], EDXIfElse(
            cache["is_compliance"],
            EDXDecrypt(attributes["payment_encrypted"], false, ""),
            "***MASKED***"
          ))
```

### 4. Key Rotation
**Scenario**: Migrate from old encryption keys to new ones.

```yaml
processors:
  - name: key-rotation
    transform:
      statements:
        - set(cache["plaintext"], EDXDecrypt(attributes["data_old_key"], false, ""))
        - set(attributes["data_new_key"], EDXEncrypt(cache["plaintext"], 1, "new-key-id", ""))
        - delete_key(attributes, "data_old_key")
```

---

## Validation Rules

1. **ED_CRYPTO_PATH Required**: Environment variable must be set
2. **Keys File Required**: `$ED_CRYPTO_PATH/auth/keys.json` must exist
3. **Valid Envelope**: Input must be valid JSON envelope from EDXEncrypt
4. **Key Availability**: Referenced keyId must exist in keys.json
5. **Algorithm Match**: Key algorithm must match encrypted data
6. **Base64 Format**: Envelope and components must be valid base64

---

## Common Pitfalls

### 1. Missing Environment Variable
**Problem**: ED_CRYPTO_PATH not set.

```yaml
# FAILS: No crypto path configured
set(cache["decrypted"], EDXDecrypt(attributes["encrypted"], false, ""))
# Error: ED_CRYPTO_PATH not set

# CORRECT: Ensure environment variable is set
# export ED_CRYPTO_PATH=/opt/edgedelta/crypto
```

### 2. Key Rotation Without Migration
**Problem**: Decrypting with old key after rotation.

```yaml
# PROBLEM: Old encrypted data, new keys.json
# keys.json only has new-key-002, but data encrypted with prod-key-001

# SOLUTION: Keep old keys during migration period
{
  "keys": [
    {"keyId": "prod-key-001", ...},  // Keep old key
    {"keyId": "prod-key-002", ...}   // New key
  ]
}
```

### 3. Storing Decrypted Data
**Problem**: Leaving sensitive data decrypted in attributes.

```yaml
# RISKY: Decrypted value stays in attributes
set(attributes["password"], EDXDecrypt(attributes["password_enc"], false, ""))
# Password now travels through entire pipeline!

# BETTER: Use cache, process, delete
set(cache["password"], EDXDecrypt(attributes["password_enc"], false, ""))
# ... use cache["password"] ...
delete_key(cache, "password")
```

### 4. Invalid Envelope Format
**Problem**: Trying to decrypt non-envelope data.

```yaml
# FAILS: Plain base64, not EDXEncrypt envelope
set(cache["decrypted"], EDXDecrypt("SGVsbG8=", false, ""))
# Error: Invalid envelope format

# CORRECT: Only decrypt EDXEncrypt output
# EDXDecrypt requires the full JSON envelope structure
```

---

## Best Practices

### 1. Use Cache for Sensitive Data
Never store decrypted secrets in attributes:

```yaml
processors:
  - name: secure-handling
    transform:
      statements:
        - set(cache["secret"], EDXDecrypt(attributes["encrypted_secret"], false, ""))
        # Use cache["secret"] here
        - delete_key(cache, "secret")  # Clean up
```

### 2. Implement Key Rotation Strategy
Maintain backward compatibility:

```json
{
  "keys": [
    {"keyId": "key-v1", ...},  // For decrypting old data
    {"keyId": "key-v2", ...}   // For new encryption
  ]
}
```

### 3. Error Handling for Compliance
Log decryption failures for audit:

```yaml
processors:
  - name: audit-decrypt
    transform:
      statements:
        - set(cache["decrypted"], EDXDecrypt(attributes["pii"], false, "") or "")
        - set(cache["success"], cache["decrypted"] != "")
        - set(attributes["decryption_status"], EDXIfElse(cache["success"], "ok", "failed"))
```

### 4. Environment-Specific Keys
Use different keys per environment:

```yaml
# Development: dev-key-001
# Staging: staging-key-001
# Production: prod-key-001
```

---

## Security Considerations

1. **Key Storage**: Store keys.json with restricted permissions (0600)
2. **Key Rotation**: Rotate keys periodically (e.g., every 90 days)
3. **Access Control**: Limit ED_CRYPTO_PATH access to Edge Delta process only
4. **Logging**: Never log decrypted sensitive data
5. **Memory**: Sensitive data in cache is cleared after use
6. **Transport**: Use TLS for any network transmission of encrypted data

---

## Performance Considerations

- **Decryption Speed**: AES-256-GCM faster than CBC
- **Key Loading**: Keys loaded once at startup, cached
- **Memory**: Each decryption allocates result buffer
- **CPU**: Moderate CPU usage per decryption

**Performance Tips:**
- Minimize decryption operations
- Batch decrypt if possible
- Use cache to avoid re-decryption
- Prefer GCM over CBC for better performance

---

## Related Functions

- **EDXEncrypt**: Encryption counterpart
- **EDXEnv**: Load encrypted values from environment
- **EDXDecode**: Decode base64 before inspection (debugging)

---

## Cross-References

- [MASTER_INDEX.md](../../MASTER_INDEX.md#edxdecrypt)
- [EDXEncrypt](./edxencrypt.md) - Encryption counterpart
- [syntax_guide.md](../syntax/syntax_guide.md)

---

## Notes

- **Version History**: Introduced in v2.5.0
- **Algorithms**: Supports AES-256-GCM and AES-256-CBC
- **Key Management**: Local file-based key storage only
- **Envelope Version**: Currently only version 1 supported
- **Thread Safety**: Function is thread-safe, keys cached globally
