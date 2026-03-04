# EDXDecompress - Decompress Compressed Data

**Function Type**: EDX Converter
**Purpose**: Decompresses byte array data using gzip, snappy, or zstd algorithms and returns string or byte array
**Signature**: `EDXDecompress(input, compression_type, asBytes) -> string|[]byte`
**Return Type**: string or []byte (based on asBytes parameter)
**Minimum Agent Version**: v1.31.0
**Source**: https://docs.edgedelta.com/ottl-extensions/

---

## Quick Copy

```yaml
# Decompress gzip data to string
set(cache["decompressed"], EDXDecompress(cache["compressed"], "gzip", false))
```

---

## Overview

EDXDecompress is the counterpart to EDXCompress, restoring compressed data to its original form. Essential for:
- Processing compressed log archives
- Decoding compressed HTTP responses
- Reading compressed cache values
- Processing batched compressed events

**Supported Algorithms:**
- **gzip**: RFC 1952 DEFLATE compression
- **snappy**: Google Snappy fast compression
- **zstd**: Zstandard high-performance compression

**Key Feature**: Can return either string (default) or byte array based on `asBytes` parameter.

---

## Parameters

### input
- **Type**: []byte
- **Required**: Yes
- **Description**: Compressed byte array to decompress
- **Source**: Typically from EDXCompress, Redis, file, or HTTP body

### compression_type
- **Type**: string
- **Required**: Yes
- **Description**: Compression algorithm used
- **Valid Values**: `"gzip"`, `"snappy"`, `"zstd"`
- **Must Match**: Algorithm used during compression

### asBytes
- **Type**: bool
- **Required**: No
- **Default**: false
- **Description**: Return type control
  - `false` (default): Return as string
  - `true`: Return as byte array

---

## Return Value

Returns decompressed data as string or []byte.

**Return Behavior:**
- `asBytes = false`: Interprets decompressed data as UTF-8 string
- `asBytes = true`: Returns raw byte array
- Empty input → empty result
- Invalid compression → error

---

## Examples

### Example 1: Decompress from Cache
```yaml
processors:
  - name: decompress-cached
    transform:
      statements:
        - set(cache["decompressed"], EDXDecompress(cache["compressed_data"], "gzip", false))
        - set(cache["parsed"], ParseJSON(cache["decompressed"]))
```

### Example 2: Decompress HTTP Body
```yaml
processors:
  - name: decompress-response
    transform:
      statements:
        - set(cache["is_gzip"], attributes["content_encoding"] == "gzip")
        - set(cache["body_text"], EDXIfElse(
            cache["is_gzip"],
            EDXDecompress(body, "gzip", false),
            String(body)
          ))
```

### Example 3: Decompress from Redis
```yaml
processors:
  - name: redis-decompress
    transform:
      statements:
        - set(cache["redis_result"], EDXRedis(
            {"url": "redis://localhost:6379"},
            [{"command": "get", "key": attributes["cache_key"], "outField": "compressed"}]
          ))
        - set(cache["decoded"], EDXDecode(cache["redis_result"]["compressed"], "base64"))
        - set(cache["decompressed"], EDXDecompress(cache["decoded"], "zstd", false))
```

### Example 4: Binary Decompression
```yaml
processors:
  - name: decompress-binary
    transform:
      statements:
        - set(cache["binary_data"], EDXDecompress(attributes["compressed"], "snappy", true))
        - set(attributes["data_size"], Len(cache["binary_data"]))
```

### Example 5: Full Decode and Decompress Pipeline
```yaml
processors:
  - name: decode-decompress
    transform:
      statements:
        - set(cache["decoded"], EDXDecode(attributes["payload"], "base64"))
        - set(cache["decompressed"], EDXDecompress(cache["decoded"], "gzip", false))
        - set(attributes["message"], cache["decompressed"])
```

### Example 6: Multiple Algorithm Support
```yaml
processors:
  - name: multi-algorithm-decompress
    transform:
      statements:
        - set(cache["algorithm"], attributes["compression_type"])
        - set(cache["decompressed"], EDXDecompress(
            cache["compressed_payload"],
            cache["algorithm"],
            false
          ))
```

### Example 7: Compression Metrics
```yaml
processors:
  - name: compression-stats
    transform:
      statements:
        - set(cache["compressed_size"], Len(cache["compressed"]))
        - set(cache["decompressed"], EDXDecompress(cache["compressed"], "zstd", false))
        - set(cache["original_size"], Len(cache["decompressed"]))
        - set(attributes["compression_ratio"],
            cache["compressed_size"] / cache["original_size"])
```

### Example 8: Error Handling
```yaml
processors:
  - name: safe-decompress
    transform:
      statements:
        - set(cache["decompressed"], EDXDecompress(cache["data"], "gzip", false) or "")
        - set(cache["success"], cache["decompressed"] != "")
```

---

## Common Use Cases

### 1. Log Archive Processing
```yaml
processors:
  - name: process-archived-logs
    transform:
      statements:
        - set(cache["log_data"], EDXDecompress(attributes["archived_log"], "zstd", false))
        - set(cache["parsed"], ParseJSON(cache["log_data"]))
```

### 2. Compressed Cache Retrieval
```yaml
processors:
  - name: cache-decompress
    transform:
      statements:
        - set(cache["redis_data"], EDXRedis(
            {"url": "redis://localhost:6379"},
            [{"command": "get", "key": attributes["key"], "outField": "value"}]
          ))
        - set(cache["decoded"], EDXDecode(cache["redis_data"]["value"], "base64"))
        - set(cache["response"], EDXDecompress(cache["decoded"], "snappy", false))
```

### 3. HTTP Response Decompression
```yaml
processors:
  - name: http-decompress
    transform:
      statements:
        - set(cache["encoding"], attributes["content_encoding"])
        - set(cache["decompressed"], EDXIfElse(
            cache["encoding"] == "gzip",
            EDXDecompress(body, "gzip", false),
            String(body)
          ))
```

---

## Validation Rules

1. **Input Required**: Must provide compressed byte array
2. **Algorithm Match**: Must use same algorithm as compression
3. **Valid Format**: Input must be validly compressed data
4. **Type Safety**: Input must be []byte (not string)

---

## Common Pitfalls

### 1. Algorithm Mismatch
```yaml
# WRONG: Different algorithms
set(cache["compressed"], EDXCompress(data, "gzip"))
set(cache["decompressed"], EDXDecompress(cache["compressed"], "snappy", false))  # ERROR

# CORRECT: Match algorithms
set(cache["compressed"], EDXCompress(data, "gzip"))
set(cache["decompressed"], EDXDecompress(cache["compressed"], "gzip", false))
```

### 2. Forgetting Base64 Decode
```yaml
# WRONG: Decompressing base64-encoded data
set(cache["decompressed"], EDXDecompress(attributes["payload"], "gzip", false))  # ERROR

# CORRECT: Decode first
set(cache["decoded"], EDXDecode(attributes["payload"], "base64"))
set(cache["decompressed"], EDXDecompress(cache["decoded"], "gzip", false))
```

---

## Best Practices

1. **Track Compression Metadata**
```yaml
set(attributes["compression.algorithm"], "gzip")
set(cache["decompressed"], EDXDecompress(data, attributes["compression.algorithm"], false))
```

2. **Handle Errors Gracefully**
```yaml
set(cache["decompressed"], EDXDecompress(data, "gzip", false) or "")
set(cache["valid"], cache["decompressed"] != "")
```

3. **Use Appropriate Return Type**
```yaml
# For text data
set(cache["text"], EDXDecompress(data, "gzip", false))

# For binary data
set(cache["binary"], EDXDecompress(data, "gzip", true))
```

---

## Performance Considerations

- **gzip**: Moderate speed, good for HTTP
- **snappy**: Fastest decompression
- **zstd**: Fast with excellent ratios

**Tips:**
- Cache decompressed results
- Use snappy for high-throughput
- Use zstd for best compression ratios

---

## Related Functions

- **EDXCompress**: Compression counterpart
- **EDXDecode**: Often used before decompression
- **ParseJSON**: Common next step after decompression

---

## Cross-References

- [MASTER_INDEX.md](../../MASTER_INDEX.md#edxdecompress)
- [EDXCompress](./edxcompress.md)
- [EDXDecode](./edxdecode.md)

---

## Notes

- **Version**: Introduced in v1.31.0
- **Thread Safety**: Thread-safe
- **Memory**: Allocates buffer for decompressed data
