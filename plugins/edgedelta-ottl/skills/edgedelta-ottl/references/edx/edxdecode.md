# EDXDecode - Decode Encoded Strings

**Function Type**: EDX Converter
**Purpose**: Decodes strings from various encoding formats including URL, hex, base64 variants, and IANA character encodings
**Signature**: `EDXDecode(value, encoding_type) -> string`
**Return Type**: string (decoded text)
**Minimum Agent Version**: v2.5.0
**Source**: https://docs.edgedelta.com/ottl-extensions/

---

## Quick Copy

```yaml
# Decode URL-encoded string
set(attributes["decoded"], EDXDecode(attributes["encoded_url"], "url"))
```

---

## Overview

EDXDecode transforms encoded strings back to their original form. This is essential for:
- Processing URL-encoded query parameters
- Decoding base64-encoded payloads
- Handling hex-encoded binary data
- Converting between character encodings (UTF-8, ISO-8859-1, etc.)
- Normalizing data from various sources

**Supported Encoding Types:**

**Binary Encodings:**
- `"base64"` - Standard base64 (RFC 4648)
- `"base64-url"` - URL-safe base64 (RFC 4648 §5)
- `"base64-raw"` - Base64 without padding
- `"base64-raw-url"` - URL-safe base64 without padding
- `"hex"` - Hexadecimal encoding
- `"url"` - URL percent-encoding (RFC 3986)

**Character Encodings (IANA):**
- `"utf-8"` - UTF-8 (default)
- `"iso-8859-1"` - Latin-1
- `"windows-1252"` - Windows Western European
- Any IANA-registered character encoding

**Common Workflows:**
```
URL-encoded → EDXDecode("url") → Plain text
Base64 → EDXDecode("base64") → Original bytes → String
Hex → EDXDecode("hex") → Binary data → String
```

---

## Parameters

### value
- **Type**: string
- **Required**: Yes
- **Description**: The encoded string to decode
- **Valid Values**: Any string in the specified encoding format
- **Validation**: Must be valid for the specified encoding type

### encoding_type
- **Type**: string
- **Required**: Yes
- **Description**: The encoding format of the input
- **Valid Values**:
  - `"url"` - URL percent-encoding (%20, %3A, etc.)
  - `"hex"` - Hexadecimal (e.g., "48656c6c6f")
  - `"base64"` - Standard base64
  - `"base64-url"` - URL-safe base64 (- and _ instead of + and /)
  - `"base64-raw"` - Base64 without = padding
  - `"base64-raw-url"` - URL-safe base64 without padding
  - IANA encoding names (e.g., "iso-8859-1", "windows-1252")
- **Case Sensitivity**: Case-insensitive

---

## Return Value

Returns the decoded string.

**Decoding Behavior:**
- URL: Decodes percent-encoded characters (%XX)
- Hex: Converts hex string to bytes, then to UTF-8 string
- Base64: Decodes base64, interprets as UTF-8 string
- IANA: Converts from source encoding to UTF-8

**Error Conditions:**
- Invalid base64/hex format → error
- Unknown encoding type → error
- Unsupported IANA encoding → error
- Malformed percent-encoding → error

---

## Examples

### Example 1: Decode URL Parameters
```yaml
# Decode URL-encoded query string
processors:
  - name: decode-query
    transform:
      statements:
        - set(attributes["search_raw"], "q=hello%20world&type=search%2Bquery")
        - set(cache["decoded"], EDXDecode(attributes["search_raw"], "url"))
        # Result: "q=hello world&type=search+query"
```

### Example 2: Decode Base64 Payload
```yaml
# Decode base64-encoded HTTP body
processors:
  - name: decode-body
    transform:
      statements:
        - set(cache["decoded"], EDXDecode(attributes["base64_body"], "base64"))
        - set(cache["parsed"], ParseJSON(cache["decoded"]))
```

### Example 3: Decode Hex String
```yaml
# Convert hex-encoded data to string
processors:
  - name: decode-hex
    transform:
      statements:
        - set(cache["hex_data"], "48656c6c6f20576f726c64")
        - set(attributes["text"], EDXDecode(cache["hex_data"], "hex"))
        # Result: "Hello World"
```

### Example 4: Decode URL-Safe Base64
```yaml
# Handle URL-safe base64 (JWT payloads, etc.)
processors:
  - name: decode-jwt-payload
    transform:
      statements:
        - set(cache["parts"], Split(attributes["jwt_token"], "."))
        - set(cache["payload_encoded"], cache["parts"][1])
        - set(cache["payload_json"], EDXDecode(cache["payload_encoded"], "base64-url"))
        - set(cache["payload"], ParseJSON(cache["payload_json"]))
```

### Example 5: Decode Multiple Encodings
```yaml
# Data that is URL-encoded then base64-encoded
processors:
  - name: double-decode
    transform:
      statements:
        - set(cache["decoded_b64"], EDXDecode(attributes["data"], "base64"))
        - set(cache["decoded_url"], EDXDecode(cache["decoded_b64"], "url"))
```

### Example 6: Handle Character Encoding
```yaml
# Convert from ISO-8859-1 to UTF-8
processors:
  - name: charset-conversion
    transform:
      statements:
        - set(cache["utf8_text"], EDXDecode(attributes["latin1_text"], "iso-8859-1"))
```

### Example 7: Decode with Error Handling
```yaml
# Safe decoding with fallback
processors:
  - name: safe-decode
    transform:
      statements:
        - set(cache["decoded"], EDXDecode(attributes["encoded"], "base64") or attributes["encoded"])
        - set(attributes["decode_success"], cache["decoded"] != attributes["encoded"])
```

### Example 8: Decode Email Headers
```yaml
# Decode MIME encoded-word format
processors:
  - name: decode-mime
    transform:
      statements:
        - set(cache["subject"], "=?UTF-8?B?SGVsbG8gV29ybGQ=?=")
        - set(cache["decoded"], EDXDecode("SGVsbG8gV29ybGQ=", "base64"))
```

### Example 9: Decode Cookie Values
```yaml
# Decode URL-encoded cookie
processors:
  - name: decode-cookie
    transform:
      statements:
        - set(cache["cookie_val"], "user_pref=theme%3Ddark%26lang%3Den")
        - set(cache["decoded"], EDXDecode(cache["cookie_val"], "url"))
        # Result: "user_pref=theme=dark&lang=en"
```

### Example 10: Batch Decode Multiple Fields
```yaml
# Decode multiple base64 fields
processors:
  - name: batch-decode
    transform:
      statements:
        - set(attributes["username"], EDXDecode(attributes["username_b64"], "base64"))
        - set(attributes["email"], EDXDecode(attributes["email_b64"], "base64"))
        - set(attributes["note"], EDXDecode(attributes["note_b64"], "base64"))
```

---

## Common Use Cases

### 1. HTTP Query String Parsing
**Scenario**: Extract and decode URL parameters from request logs.

```yaml
processors:
  - name: parse-query-string
    transform:
      statements:
        - set(cache["query"], attributes["http.query_string"])
        - set(cache["decoded"], EDXDecode(cache["query"], "url"))
        - set(cache["params"], EDXParseKeyValue(cache["decoded"], "=", "&", false, "last"))
        - set(attributes["query.search"], cache["params"]["q"])
        - set(attributes["query.page"], cache["params"]["page"])
```

### 2. JWT Token Parsing
**Scenario**: Decode and parse JWT payload for logging/enrichment.

```yaml
processors:
  - name: extract-jwt-claims
    transform:
      statements:
        - set(cache["token_parts"], Split(attributes["authorization"], "."))
        - set(cache["payload_b64"], cache["token_parts"][1])
        - set(cache["payload_json"], EDXDecode(cache["payload_b64"], "base64-url"))
        - set(cache["claims"], ParseJSON(cache["payload_json"]))
        - set(attributes["user_id"], cache["claims"]["sub"])
        - set(attributes["token_exp"], cache["claims"]["exp"])
```

### 3. Binary Data Handling
**Scenario**: Decode hex-encoded binary data from logs.

```yaml
processors:
  - name: decode-binary-log
    transform:
      statements:
        - set(cache["hex_payload"], attributes["binary_data"])
        - set(cache["bytes"], EDXDecode(cache["hex_payload"], "hex"))
        - set(attributes["payload_text"], cache["bytes"])
```

### 4. Email Processing
**Scenario**: Decode base64-encoded email attachments or bodies.

```yaml
processors:
  - name: decode-email-body
    transform:
      statements:
        - set(cache["is_base64"], attributes["content_transfer_encoding"] == "base64")
        - set(cache["body"], EDXIfElse(
            cache["is_base64"],
            EDXDecode(attributes["body"], "base64"),
            attributes["body"]
          ))
```

---

## Validation Rules

1. **Value Required**: Input string must be provided (can be empty)
2. **Encoding Required**: Encoding type must be specified
3. **Format Validation**: Input must be valid for specified encoding
4. **Base64 Padding**: Standard base64 expects proper padding (use -raw variants if no padding)
5. **Hex Format**: Hex strings must have even number of characters
6. **URL Encoding**: Percent-encoding must be properly formatted (%XX)

---

## Common Pitfalls

### 1. Base64 Padding Issues
**Problem**: Using standard base64 decoder on unpadded data.

```yaml
# FAILS: Unpadded base64 with standard decoder
set(cache["decoded"], EDXDecode("SGVsbG8", "base64"))  # Error!

# CORRECT: Use raw variant for unpadded data
set(cache["decoded"], EDXDecode("SGVsbG8", "base64-raw"))
```

### 2. URL vs URL-Safe Base64 Confusion
**Problem**: Using wrong base64 variant.

```yaml
# FAILS: URL-safe base64 (- and _) with standard decoder
set(cache["decoded"], EDXDecode("SGVs-bG8_", "base64"))  # Error!

# CORRECT: Use base64-url for URL-safe encoding
set(cache["decoded"], EDXDecode("SGVs-bG8_", "base64-url"))
```

### 3. Double Encoding Not Handled
**Problem**: Forgetting to decode multiple encoding layers.

```yaml
# INCOMPLETE: Data is base64(url-encoded)
set(cache["decoded"], EDXDecode(attributes["data"], "base64"))
# Still URL-encoded!

# CORRECT: Decode both layers
set(cache["decoded_b64"], EDXDecode(attributes["data"], "base64"))
set(cache["decoded_url"], EDXDecode(cache["decoded_b64"], "url"))
```

### 4. Character Encoding Assumptions
**Problem**: Assuming decoded bytes are UTF-8.

```yaml
# RISKY: Latin-1 data decoded as if UTF-8
set(cache["decoded"], EDXDecode(attributes["data"], "base64"))
# May produce mojibake if original was not UTF-8

# BETTER: Specify source encoding
set(cache["decoded"], EDXDecode(attributes["latin1_data"], "iso-8859-1"))
```

---

## Best Practices

### 1. Match Encoding to Data Format
Always use the correct encoding type:

```yaml
# URLs and query strings
EDXDecode(data, "url")

# JWT payloads, API tokens
EDXDecode(data, "base64-url")

# Email attachments, file uploads
EDXDecode(data, "base64")

# Binary protocol dumps
EDXDecode(data, "hex")
```

### 2. Handle Errors Gracefully
Provide fallbacks for malformed data:

```yaml
processors:
  - name: safe-decode
    transform:
      statements:
        - set(cache["decoded"], EDXDecode(attributes["data"], "base64") or "")
        - set(cache["valid"], cache["decoded"] != "")
        - set(attributes["status"], EDXIfElse(cache["valid"], "ok", "decode_failed"))
```

### 3. Chain with Parsing Functions
Decode then parse in sequence:

```yaml
processors:
  - name: decode-and-parse
    transform:
      statements:
        - set(cache["decoded"], EDXDecode(attributes["payload"], "base64"))
        - set(cache["parsed"], ParseJSON(cache["decoded"]))
        - set(attributes["data"], cache["parsed"])
```

### 4. Validate Before Decoding
Check format before attempting decode:

```yaml
processors:
  - name: validate-then-decode
    transform:
      statements:
        - set(cache["looks_like_base64"], MatchesPattern(attributes["data"], "^[A-Za-z0-9+/=]+$"))
        - set(cache["decoded"], EDXIfElse(
            cache["looks_like_base64"],
            EDXDecode(attributes["data"], "base64"),
            attributes["data"]
          ))
```

---

## Performance Considerations

- **Base64**: Fast, minimal allocation
- **URL**: Very fast, in-place decoding
- **Hex**: Moderate speed, allocates result buffer
- **IANA Encodings**: Slower, requires charset conversion

**Performance Tips:**
- Cache decoded results if used multiple times
- Decode only when necessary (check if already decoded)
- Use simpler encodings (url, hex) over complex character sets when possible

---

## Related Functions

- **EDXEncode**: Encoding counterpart
- **EDXDecompress**: Use after decoding compressed base64 data
- **ParseJSON**: Common next step after decoding JSON strings
- **EDXParseKeyValue**: Parse decoded query strings

---

## Cross-References

- [MASTER_INDEX.md](../../MASTER_INDEX.md#edxdecode)
- [EDXEncode](./edxencode.md) - Encoding counterpart
- [syntax_guide.md](../syntax/syntax_guide.md)

---

## Notes

- **Version History**: Introduced in v2.5.0
- **Encoding Support**: Supports all standard base64 variants + IANA charsets
- **UTF-8 Output**: Always returns UTF-8 encoded strings
- **Error Handling**: Returns error for invalid encoding formats
- **No Binary Output**: Always returns string (use EDXDecompress for binary handling)
