# EDXEncrypt - Encrypt Sensitive Data

**Function Type**: EDX Converter
**Purpose**: Encrypts data using AES-256-CBC/GCM with local key management and returns base64-encoded JSON envelope
**Signature**: `EDXEncrypt(value, keyclass, keyId, defaultVal) -> string`
**Return Type**: string (base64-encoded JSON envelope)
**Minimum Agent Version**: v2.5.0
**Source**: https://docs.edgedelta.com/ottl-extensions/

---

## Quick Copy

```yaml
# Encrypt SSN field
set(attributes["ssn_encrypted"], EDXEncrypt(attributes["ssn"], 1, "prod-key-001", ""))
```

---

## Overview

EDXEncrypt protects sensitive data (PII, credentials, secrets) using AES-256 encryption with local key management.

**Features:**
- AES-256-CBC and AES-256-GCM support
- JSON envelope with metadata
- Local file-based key storage
- Automatic key selection by class
- Base64-encoded output for text storage

**Output Format:**
```json
{
  "v": 1,
  "keyId": "prod-key-001",
  "data": "base64-encrypted-bytes",
  "iv": "base64-initialization-vector"
}
```

**Prerequisites:**
1. `ED_CRYPTO_PATH` environment variable
2. Keys file at `$ED_CRYPTO_PATH/auth/keys.json`

---

## Parameters

### value
- **Type**: any (string, bytes, or any value)
- **Required**: Yes
- **Description**: Data to encrypt

### keyclass
- **Type**: int
- **Required**: Yes
- **Description**: Key class identifier (matches keys.json)

### keyId
- **Type**: string
- **Required**: No
- **Default**: Auto-selected from keyclass
- **Description**: Specific key ID to use

### defaultVal
- **Type**: string
- **Required**: No
- **Default**: ""
- **Description**: Fallback value if encryption fails

---

## Return Value

Returns base64-encoded JSON envelope containing encrypted data.

---

## Environment Setup

```bash
export ED_CRYPTO_PATH=/opt/edgedelta/crypto
mkdir -p /opt/edgedelta/crypto/auth

cat > /opt/edgedelta/crypto/auth/keys.json <<EOF
{
  "keys": [
    {
      "keyclass": 1,
      "keyId": "prod-key-001",
      "key": "base64-encoded-32-byte-key",
      "algorithm": "AES-256-GCM"
    }
  ]
}
EOF
```

---

## Examples

### Example 1: Encrypt PII Field
```yaml
processors:
  - name: encrypt-pii
    transform:
      statements:
        - set(attributes["ssn_encrypted"], EDXEncrypt(
            attributes["ssn"],
            1,
            "prod-key-001",
            "<encryption-failed>"
          ))
        - delete_key(attributes, "ssn")
```

### Example 2: Encrypt API Credentials
```yaml
processors:
  - name: encrypt-credentials
    transform:
      statements:
        - set(cache["api_key_encrypted"], EDXEncrypt(
            attributes["api_key"],
            1,
            "prod-key-001",
            ""
          ))
```

### Example 3: Conditional Encryption
```yaml
processors:
  - name: conditional-encrypt
    transform:
      statements:
        - set(cache["is_sensitive"], attributes["data_classification"] == "pii")
        - set(attributes["data"], EDXIfElse(
            cache["is_sensitive"],
            EDXEncrypt(attributes["data"], 1, "prod-key-001", ""),
            attributes["data"]
          ))
```

### Example 4: Batch Encryption
```yaml
processors:
  - name: encrypt-batch
    transform:
      statements:
        - set(attributes["ssn_enc"], EDXEncrypt(attributes["ssn"], 1, "prod-key-001", ""))
        - set(attributes["dob_enc"], EDXEncrypt(attributes["dob"], 1, "prod-key-001", ""))
        - delete_key(attributes, "ssn")
        - delete_key(attributes, "dob")
```

### Example 5: Store Encrypted in Redis
```yaml
processors:
  - name: redis-encrypt
    transform:
      statements:
        - set(cache["encrypted"], EDXEncrypt(attributes["secret"], 1, "prod-key-001", ""))
        - set(cache["redis_result"], EDXRedis(
            {"url": "redis://localhost:6379"},
            [{"command": "setex", "key": attributes["key"], "args": [3600, cache["encrypted"]]}]
          ))
```

---

## Common Use Cases

### 1. PII Protection
```yaml
processors:
  - name: protect-pii
    transform:
      statements:
        - set(attributes["ssn_enc"], EDXEncrypt(attributes["ssn"], 1, "", ""))
        - set(attributes["email_enc"], EDXEncrypt(attributes["email"], 1, "", ""))
        - delete_key(attributes, "ssn")
        - delete_key(attributes, "email")
```

### 2. Credential Storage
```yaml
processors:
  - name: store-credentials
    transform:
      statements:
        - set(cache["token_enc"], EDXEncrypt(attributes["api_token"], 1, "prod-key-001", ""))
        - set(attributes["token_encrypted"], cache["token_enc"])
        - delete_key(attributes, "api_token")
```

---

## Validation Rules

1. **ED_CRYPTO_PATH Required**: Environment variable must be set
2. **Keys File Required**: Must exist at $ED_CRYPTO_PATH/auth/keys.json
3. **Valid Keyclass**: Must match a keyclass in keys.json
4. **Key Format**: 32-byte keys, base64-encoded

---

## Common Pitfalls

### 1. Missing Environment Variable
```yaml
# FAILS: No ED_CRYPTO_PATH
set(cache["enc"], EDXEncrypt(data, 1, "", ""))

# CORRECT: Set environment first
# export ED_CRYPTO_PATH=/opt/edgedelta/crypto
```

### 2. Storing Plaintext After Encryption
```yaml
# WRONG: Plaintext still in attributes
set(attributes["password_enc"], EDXEncrypt(attributes["password"], 1, "", ""))
# Password still accessible!

# CORRECT: Delete plaintext
set(attributes["password_enc"], EDXEncrypt(attributes["password"], 1, "", ""))
delete_key(attributes, "password")
```

---

## Best Practices

1. **Always Delete Plaintext**
```yaml
set(attributes["data_enc"], EDXEncrypt(attributes["sensitive"], 1, "", ""))
delete_key(attributes, "sensitive")
```

2. **Use Key Rotation**
```json
{
  "keys": [
    {"keyId": "key-v1", ...},
    {"keyId": "key-v2", ...}
  ]
}
```

3. **Handle Errors**
```yaml
set(cache["enc"], EDXEncrypt(data, 1, "", "<failed>"))
set(cache["success"], cache["enc"] != "<failed>")
```

---

## Security Considerations

1. Restrict keys.json permissions (0600)
2. Rotate keys regularly
3. Never log encrypted envelopes
4. Use different keys per environment
5. Audit encryption operations

---

## Related Functions

- **EDXDecrypt**: Decryption counterpart
- **EDXEnv**: Load keys from environment
- **EDXEncode**: Encode before storage

---

## Cross-References

- [MASTER_INDEX.md](../../MASTER_INDEX.md#edxencrypt)
- [EDXDecrypt](./edxdecrypt.md)

---

## Notes

- **Version**: Introduced in v2.5.0
- **Algorithms**: AES-256-GCM, AES-256-CBC
- **Key Storage**: Local file-based only
