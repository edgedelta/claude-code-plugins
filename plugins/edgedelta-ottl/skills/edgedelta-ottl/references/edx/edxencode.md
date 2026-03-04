# EDXEncode - Encode Strings and Byte Arrays

**Function Type**: EDX Converter
**Purpose**: Encodes string or byte array data to specified encoding (base64 or IANA character encodings)
**Signature**: `EDXEncode(input, encoding, asBytes) -> string|[]byte`
**Return Type**: string or []byte (based on asBytes parameter)
**Minimum Agent Version**: v1.25.0
**Source**: https://docs.edgedelta.com/ottl-extensions/

---

## Quick Copy

```yaml
# Encode to base64 string
set(attributes["encoded"], EDXEncode(cache["data"], "base64", false))
```

---

## Overview

EDXEncode transforms data into encoded formats for storage, transmission, or compatibility. Essential for:
- Base64 encoding for text-safe binary data
- Character set conversion (UTF-8, ISO-8859-1, etc.)
- Preparing data for HTTP headers/JSON
- Encoding before compression or encryption

**Supported Encodings:**
- **base64**: Standard base64 encoding (RFC 4648)
- **IANA character encodings**: utf-8, iso-8859-1, windows-1252, etc.

---

## Parameters

### input
- **Type**: string or []byte
- **Required**: Yes  
- **Description**: Data to encode

### encoding
- **Type**: string
- **Required**: Yes
- **Description**: Target encoding format
- **Valid Values**: `"base64"` or any IANA encoding name

### asBytes  
- **Type**: bool
- **Required**: No
- **Default**: false
- **Description**: Return as byte array vs string

---

## Return Value

Returns encoded data as string (default) or []byte.

---

## Examples

### Example 1: Base64 Encode Binary Data
```yaml
processors:
  - name: encode-binary
    transform:
      statements:
        - set(cache["compressed"], EDXCompress(body, "gzip"))
        - set(attributes["payload"], EDXEncode(cache["compressed"], "base64", false))
```

### Example 2: Character Set Conversion
```yaml
processors:
  - name: convert-charset
    transform:
      statements:
        - set(cache["latin1"], EDXEncode(attributes["utf8_text"], "iso-8859-1", true))
```

### Example 3: Encode for JSON
```yaml
processors:
  - name: encode-for-json
    transform:
      statements:
        - set(cache["binary_data"], attributes["file_content"])
        - set(attributes["file_base64"], EDXEncode(cache["binary_data"], "base64", false))
```

### Example 4: Encode Multiple Fields
```yaml
processors:
  - name: batch-encode
    transform:
      statements:
        - set(attributes["field1_enc"], EDXEncode(attributes["field1"], "base64", false))
        - set(attributes["field2_enc"], EDXEncode(attributes["field2"], "base64", false))
```

### Example 5: Conditional Encoding
```yaml
processors:
  - name: conditional-encode
    transform:
      statements:
        - set(cache["needs_encoding"], Len(body) > 1024)
        - set(cache["final"], EDXIfElse(
            cache["needs_encoding"],
            EDXEncode(body, "base64", false),
            String(body)
          ))
```

---

## Common Use Cases

### 1. HTTP Header Encoding
```yaml
processors:
  - name: encode-header
    transform:
      statements:
        - set(cache["auth_str"], Concat([attributes["username"], ":", attributes["password"]]))
        - set(attributes["authorization"], Concat([
            "Basic ",
            EDXEncode(cache["auth_str"], "base64", false)
          ]))
```

### 2. Store Binary in Text Field
```yaml
processors:
  - name: binary-to-text
    transform:
      statements:
        - set(cache["compressed"], EDXCompress(body, "gzip"))
        - set(attributes["payload_b64"], EDXEncode(cache["compressed"], "base64", false))
```

---

## Validation Rules

1. **Input Required**: Must provide input data
2. **Valid Encoding**: Encoding must be supported
3. **IANA Accuracy**: IANA names must be valid

---

## Common Pitfalls

### 1. Wrong Encoding Name
```yaml
# WRONG: Invalid encoding
set(cache["encoded"], EDXEncode(data, "base-64", false))  # ERROR

# CORRECT: Use "base64"
set(cache["encoded"], EDXEncode(data, "base64", false))
```

---

## Best Practices

1. **Use for Binary Data**
```yaml
set(cache["safe_binary"], EDXEncode(cache["binary"], "base64", false))
```

2. **Specify Return Type Correctly**
```yaml
# For attributes (string)
set(attributes["encoded"], EDXEncode(data, "base64", false))

# For cache (can be bytes)
set(cache["encoded_bytes"], EDXEncode(data, "base64", true))
```

---

## Related Functions

- **EDXDecode**: Decoding counterpart
- **EDXCompress**: Often encode after compression
- **EDXEncrypt**: Often encode after encryption

---

## Cross-References

- [MASTER_INDEX.md](../../MASTER_INDEX.md#edxencode)
- [EDXDecode](./edxdecode.md)
- [EDXCompress](./edxcompress.md)

---

## Notes

- **Version**: Introduced in v1.25.0
- **Base64 Variant**: Uses standard base64 (not URL-safe)
- **Thread Safety**: Thread-safe
