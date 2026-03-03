# EDXHmac - Generate HMAC Signatures

**Function Type**: EDX Converter
**Purpose**: Generate HMAC (Hash-based Message Authentication Code) signatures for API authentication and data integrity
**Signature**: `EDXHmac(message, key, algorithm, encoding) -> string`
**Return Type**: string (encoded HMAC signature)
**Minimum Agent Version**: v2.6.0
**Source**: /pkg/ottlext/funcs/func_edx_hmac.go

---

## Quick Copy

```yaml
# Generate SHA-256 HMAC signature
set(attributes["signature"], EDXHmac(body, EDXEnv("API_SECRET"), "sha256", "hex"))
```

---

## Overview

EDXHmac generates cryptographic signatures for:
- API request signing
- Webhook verification
- Data integrity checking
- OAuth/JWT signature generation

**Supported Hash Algorithms (12 total):**
- **MD5**: `"md5"` (legacy, not recommended)
- **SHA-1**: `"sha1"` (legacy)
- **SHA-2**: `"sha224"`, `"sha256"` (default), `"sha384"`, `"sha512"`
- **SHA-512 Variants**: `"sha512/224"`, `"sha512/256"`
- **SHA-3**: `"sha3-224"`, `"sha3-256"`, `"sha3-384"`, `"sha3-512"`

**Output Encodings:**
- **hex**: Hexadecimal (default)
- **base64**: Base64 encoding

---

## Parameters

### message
- **Type**: string or []byte
- **Required**: Yes
- **Description**: Data to sign
- **Common Values**: HTTP body, query string, timestamp+nonce

### key
- **Type**: string or []byte
- **Required**: Yes
- **Description**: Secret key for HMAC
- **Source**: EDXEnv, attributes, or hardcoded
- **Security**: Keep secret, never log

### algorithm
- **Type**: string
- **Required**: No
- **Default**: `"sha256"`
- **Description**: Hash algorithm to use
- **Valid Values**: See list above
- **Case Insensitive**: `"SHA256"` = `"sha256"`

### encoding
- **Type**: string
- **Required**: No
- **Default**: `"hex"`
- **Description**: Output encoding format
- **Valid Values**: `"hex"`, `"base64"`
- **Case Insensitive**: `"HEX"` = `"hex"`

---

## Return Value

Returns HMAC signature as encoded string.

**Examples:**
- SHA-256 + hex: `"3bcebf43c85d20bba6e3b6ba278af1d2ba3ab0d57de271b0ad30b833e851c5a6"`
- SHA-256 + base64: `"O86/Q8hdILum47a6J4rx0ro6sNV94nGwrTC4M+hRxaY="`

---

## Examples

### Example 1: Sign API Request
```yaml
processors:
  - name: sign-request
    transform:
      statements:
        - set(cache["secret"], EDXEnv("API_SECRET"))
        - set(cache["signature"], EDXHmac(body, cache["secret"], "sha256", "hex"))
        - set(attributes["X-Signature"], cache["signature"])
```

### Example 2: Verify Webhook
```yaml
processors:
  - name: verify-webhook
    transform:
      statements:
        - set(cache["secret"], EDXEnv("WEBHOOK_SECRET"))
        - set(cache["computed_sig"], EDXHmac(body, cache["secret"], "sha256", "hex"))
        - set(cache["is_valid"], cache["computed_sig"] == attributes["X-Hub-Signature-256"])
```

### Example 3: AWS Signature V4 Style
```yaml
processors:
  - name: aws-sign
    transform:
      statements:
        - set(cache["string_to_sign"], Concat([
            attributes["http_method"], "\n",
            attributes["canonical_uri"], "\n",
            attributes["canonical_query"]
          ]))
        - set(cache["signature"], EDXHmac(
            cache["string_to_sign"],
            EDXEnv("AWS_SECRET_KEY"),
            "sha256",
            "hex"
          ))
```

### Example 4: JWT HS256 Signature
```yaml
processors:
  - name: jwt-sign
    transform:
      statements:
        - set(cache["header_b64"], EDXEncode(ToJSON({"alg":"HS256","typ":"JWT"}), "base64-url", false))
        - set(cache["payload_b64"], EDXEncode(ToJSON(attributes["claims"]), "base64-url", false))
        - set(cache["signing_input"], Concat([cache["header_b64"], ".", cache["payload_b64"]]))
        - set(cache["signature"], EDXHmac(
            cache["signing_input"],
            EDXEnv("JWT_SECRET"),
            "sha256",
            "base64"
          ))
```

### Example 5: GitHub Webhook Verification
```yaml
processors:
  - name: github-webhook
    transform:
      statements:
        - set(cache["secret"], EDXEnv("GITHUB_WEBHOOK_SECRET"))
        - set(cache["expected"], EDXHmac(body, cache["secret"], "sha256", "hex"))
        - set(cache["actual"], Substring(attributes["X-Hub-Signature-256"], 7))  # Remove "sha256=" prefix
        - set(cache["valid"], cache["expected"] == cache["actual"])
```

### Example 6: Stripe Webhook Verification
```yaml
processors:
  - name: stripe-webhook
    transform:
      statements:
        - set(cache["secret"], EDXEnv("STRIPE_WEBHOOK_SECRET"))
        - set(cache["payload"], Concat([
            attributes["timestamp"], ".",
            body
          ]))
        - set(cache["signature"], EDXHmac(cache["payload"], cache["secret"], "sha256", "hex"))
```

### Example 7: Different Hash Algorithms
```yaml
processors:
  - name: multi-algorithm
    transform:
      statements:
        - set(attributes["hmac_md5"], EDXHmac(body, "secret", "md5", "hex"))
        - set(attributes["hmac_sha1"], EDXHmac(body, "secret", "sha1", "hex"))
        - set(attributes["hmac_sha256"], EDXHmac(body, "secret", "sha256", "hex"))
        - set(attributes["hmac_sha512"], EDXHmac(body, "secret", "sha512", "hex"))
```

### Example 8: Base64 Encoding
```yaml
processors:
  - name: base64-hmac
    transform:
      statements:
        - set(cache["signature"], EDXHmac(
            attributes["data"],
            EDXEnv("SECRET"),
            "sha256",
            "base64"
          ))
        - set(attributes["Authorization"], Concat(["HMAC ", cache["signature"]]))
```

---

## Common Use Cases

### 1. API Request Signing
```yaml
processors:
  - name: api-auth
    transform:
      statements:
        - set(cache["timestamp"], String(UnixSeconds(Now())))
        - set(cache["nonce"], attributes["request_id"])
        - set(cache["payload"], Concat([
            attributes["http_method"],
            attributes["path"],
            cache["timestamp"],
            cache["nonce"],
            body
          ]))
        - set(cache["signature"], EDXHmac(
            cache["payload"],
            EDXEnv("API_SECRET"),
            "sha256",
            "base64"
          ))
        - set(attributes["X-Signature"], cache["signature"])
        - set(attributes["X-Timestamp"], cache["timestamp"])
```

### 2. Webhook Verification
```yaml
processors:
  - name: webhook-verify
    transform:
      statements:
        - set(cache["expected_sig"], EDXHmac(
            body,
            EDXEnv("WEBHOOK_SECRET"),
            "sha256",
            "hex"
          ))
        - set(cache["valid"], cache["expected_sig"] == attributes["X-Webhook-Signature"])
        - set(attributes["webhook_verified"], cache["valid"])
```

### 3. OAuth 1.0 Signing
```yaml
processors:
  - name: oauth1-sign
    transform:
      statements:
        - set(cache["base_string"], Concat([
            attributes["method"], "&",
            EDXEncode(attributes["url"], "url", false), "&",
            EDXEncode(attributes["params"], "url", false)
          ]))
        - set(cache["signing_key"], Concat([
            EDXEnv("CONSUMER_SECRET"), "&",
            attributes["token_secret"]
          ]))
        - set(cache["signature"], EDXHmac(
            cache["base_string"],
            cache["signing_key"],
            "sha1",
            "base64"
          ))
```

---

## Validation Rules

1. **Message Required**: Cannot be nil or empty
2. **Key Required**: Secret key must be provided and non-empty
3. **Valid Algorithm**: Must be one of 12 supported algorithms
4. **Valid Encoding**: Must be "hex" or "base64"
5. **Case Insensitive**: Algorithm and encoding names

---

## Common Pitfalls

### 1. Logging Secret Keys
```yaml
# DANGEROUS: Secret in attributes
set(attributes["secret"], EDXEnv("API_SECRET"))
set(attributes["sig"], EDXHmac(body, attributes["secret"], "sha256", "hex"))
# Secret now in logs!

# SAFE: Use cache
set(cache["secret"], EDXEnv("API_SECRET"))
set(attributes["sig"], EDXHmac(body, cache["secret"], "sha256", "hex"))
```

### 2. Algorithm Mismatch
```yaml
# Signature won't match if algorithms differ
# Sign: sha256, Verify: sha1 = FAIL
```

### 3. Encoding Mismatch
```yaml
# WRONG: Different encodings
set(cache["sig"], EDXHmac(body, key, "sha256", "hex"))
# Expecting base64 on verification side = FAIL

# CORRECT: Match encoding
set(cache["sig"], EDXHmac(body, key, "sha256", "base64"))
```

---

## Best Practices

### 1. Use Strong Algorithms
```yaml
# RECOMMENDED: SHA-256 or higher
EDXHmac(data, key, "sha256", "hex")

# AVOID: MD5, SHA-1 (deprecated)
```

### 2. Store Secrets Securely
```yaml
# Use environment variables
set(cache["key"], EDXEnv("HMAC_SECRET"))

# Never hardcode
# WRONG: set(cache["key"], "my-secret-key")
```

### 3. Include Timestamp/Nonce
```yaml
# Prevent replay attacks
set(cache["timestamp"], String(UnixSeconds(Now())))
set(cache["payload"], Concat([body, cache["timestamp"]]))
set(cache["sig"], EDXHmac(cache["payload"], key, "sha256", "hex"))
```

---

## Security Considerations

1. **Secret Management**: Never log or expose HMAC keys
2. **Algorithm Choice**: Use SHA-256 or stronger
3. **Replay Protection**: Include timestamp in signed data
4. **Constant-Time Comparison**: Use secure comparison for verification
5. **Key Rotation**: Rotate HMAC secrets periodically

---

## Performance Considerations

- **SHA-256**: Good balance of speed and security
- **SHA-512**: Slower but more secure
- **MD5/SHA-1**: Faster but deprecated
- **Hex Encoding**: Slightly faster than base64
- **Caching**: Cache signatures when possible

---

## Related Functions

- **EDXEnv**: Load secret keys from environment
- **EDXEncrypt**: For data encryption (vs signing)
- **EDXEncode**: Encode message before signing

---

## Cross-References

- [MASTER_INDEX.md](../../MASTER_INDEX.md#edxhmac)
- [EDXEnv](./edxenv.md)
- [EDXEncode](./edxencode.md)

---

## Notes

- **Version**: Introduced in v2.6.0
- **Algorithms**: 12 hash functions supported
- **Thread Safety**: Thread-safe
- **Constant Time**: Not guaranteed constant-time comparison
