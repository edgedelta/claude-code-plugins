# EDXCompress - Compress Data with Multiple Algorithms

**Function Type**: EDX Converter
**Purpose**: Compresses string or byte array data using specified compression algorithm (gzip, snappy, or zstd)
**Signature**: `EDXCompress(input, compression_type) -> []byte`
**Return Type**: []byte (compressed data as byte array)
**Minimum Agent Version**: v1.31.0
**Source**: https://docs.edgedelta.com/ottl-extensions/

---

## Quick Copy

```yaml
# Compress HTTP body with gzip
set(cache["compressed"], EDXCompress(body, "gzip"))
```

---

## Overview

EDXCompress compresses data using industry-standard compression algorithms. This is essential for:
- Reducing payload sizes before transmission
- Storing data efficiently in databases or caches
- Optimizing network bandwidth usage
- Implementing compression pipelines for log aggregation

The function supports three compression algorithms:
1. **gzip**: RFC 1952 standard, widely compatible, good compression ratio
2. **snappy**: Google's fast compression, optimized for speed over ratio
3. **zstd**: Modern algorithm with excellent compression ratio and speed

**Input Handling:**
- Accepts both strings and byte arrays
- Automatically converts strings to bytes before compression
- Returns compressed data as byte array (not base64 encoded)

**Typical Workflow:**
```
Raw Data (string/bytes) → EDXCompress → Compressed []byte → EDXEncode (optional base64)
```

---

## Parameters

### input
- **Type**: string or []byte
- **Required**: Yes
- **Description**: The data to compress
- **Valid Values**: Any string or byte array
- **Size Limits**: No explicit limits, but consider agent memory constraints for very large inputs
- **Encoding**: If string, uses UTF-8 encoding before compression

### compression_type
- **Type**: string
- **Required**: Yes
- **Description**: The compression algorithm to use
- **Valid Values**:
  - `"gzip"` - DEFLATE algorithm (RFC 1952), wide compatibility
  - `"snappy"` - Google Snappy, optimized for speed
  - `"zstd"` - Zstandard, modern high-performance compression
- **Case Sensitivity**: Case-sensitive (use lowercase)
- **Invalid Values**: Any other string causes error

---

## Return Value

Returns compressed data as a byte array (`[]byte`).

**Important Notes:**
- Result is raw binary data, not base64-encoded
- To send over HTTP or store as text, use `EDXEncode(result, "base64")`
- Byte array can be stored in cache or attributes
- Use `EDXDecompress()` with the same algorithm to decompress

**Size Characteristics:**
- **gzip**: 60-70% compression for text, slower speed
- **snappy**: 40-50% compression, 5-10x faster than gzip
- **zstd**: 65-75% compression, 2-3x faster than gzip

---

## Examples

### Example 1: Compress HTTP Response Body
```yaml
# Compress large JSON response before caching
processors:
  - name: compress-response
    transform:
      statements:
        - set(cache["compressed_body"], EDXCompress(body, "gzip"))
        - set(attributes["compressed_size"], Len(cache["compressed_body"]))
        - set(attributes["original_size"], Len(body))
```

### Example 2: Compress and Base64 Encode for Storage
```yaml
# Compress and encode for storing in text field
processors:
  - name: compress-and-encode
    transform:
      statements:
        - set(cache["compressed"], EDXCompress(body, "zstd"))
        - set(attributes["payload"], EDXEncode(cache["compressed"], "base64", false))
```

### Example 3: Conditional Compression Based on Size
```yaml
# Only compress if body is larger than 1KB
processors:
  - name: conditional-compression
    transform:
      statements:
        - set(cache["body_size"], Len(body))
        - set(cache["should_compress"], cache["body_size"] > 1024)
        - set(cache["final_body"], EDXIfElse(
            cache["should_compress"],
            EDXCompress(body, "gzip"),
            body
          ))
```

### Example 4: Using Snappy for Speed
```yaml
# Fast compression for high-throughput scenarios
processors:
  - name: fast-compress
    transform:
      statements:
        - set(cache["compressed"], EDXCompress(attributes["log_data"], "snappy"))
        - set(attributes["compression_type"], "snappy")
```

### Example 5: Compress JSON Before Sending to S3
```yaml
# Compress structured data
processors:
  - name: compress-for-s3
    transform:
      statements:
        - set(cache["json_str"], ToJSON(attributes))
        - set(cache["compressed"], EDXCompress(cache["json_str"], "zstd"))
        - set(cache["encoded"], EDXEncode(cache["compressed"], "base64", false))
```

### Example 6: Compress Multiple Fields
```yaml
# Compress multiple log fields
processors:
  - name: compress-fields
    transform:
      statements:
        - set(cache["msg_compressed"], EDXCompress(attributes["message"], "gzip"))
        - set(cache["stack_compressed"], EDXCompress(attributes["stack_trace"], "gzip"))
        - set(attributes["compressed.message"], EDXEncode(cache["msg_compressed"], "base64"))
        - set(attributes["compressed.stack"], EDXEncode(cache["stack_compressed"], "base64"))
```

### Example 7: Compression Ratio Calculation
```yaml
# Calculate and log compression efficiency
processors:
  - name: compression-metrics
    transform:
      statements:
        - set(cache["original_size"], Len(body))
        - set(cache["compressed"], EDXCompress(body, "zstd"))
        - set(cache["compressed_size"], Len(cache["compressed"]))
        - set(attributes["compression_ratio"],
            cache["original_size"] / cache["compressed_size"])
```

### Example 8: Error Handling with Fallback
```yaml
# Handle compression errors gracefully
processors:
  - name: safe-compress
    transform:
      statements:
        - set(cache["compressed"], EDXCompress(body, "gzip") or body)
        - set(attributes["is_compressed"], cache["compressed"] != body)
```

### Example 9: Algorithm Selection Based on Data Type
```yaml
# Choose algorithm based on content
processors:
  - name: smart-compression
    transform:
      statements:
        - set(cache["is_json"], attributes["content_type"] == "application/json")
        - set(cache["algorithm"], EDXIfElse(cache["is_json"], "zstd", "snappy"))
        - set(cache["compressed"], EDXCompress(body, cache["algorithm"]))
```

### Example 10: Pipeline with Decompression
```yaml
# Full compress/decompress pipeline
processors:
  - name: compress-stage
    transform:
      statements:
        - set(cache["compressed"], EDXCompress(body, "gzip"))
        - set(cache["encoded"], EDXEncode(cache["compressed"], "base64", false))

  - name: decompress-stage
    transform:
      statements:
        - set(cache["decoded"], EDXDecode(cache["encoded"], "base64"))
        - set(cache["decompressed"], EDXDecompress(cache["decoded"], "gzip", false))
```

---

## Common Use Cases

### 1. Log Aggregation Optimization
**Scenario**: Reduce bandwidth and storage costs for log shipping.

```yaml
processors:
  - name: compress-logs
    transform:
      statements:
        # Compress log messages before sending to backend
        - set(cache["log_json"], ToJSON(attributes))
        - set(cache["compressed"], EDXCompress(cache["log_json"], "zstd"))
        - set(attributes["payload"], EDXEncode(cache["compressed"], "base64"))
        - set(attributes["compression"], "zstd")
        - set(attributes["original_bytes"], Len(cache["log_json"]))
        - set(attributes["compressed_bytes"], Len(cache["compressed"]))
```

### 2. Caching Large Responses
**Scenario**: Store API responses in Redis with compression to save memory.

```yaml
processors:
  - name: cache-with-compression
    transform:
      statements:
        - set(cache["response_json"], ToJSON(attributes["api_response"]))
        - set(cache["compressed"], EDXCompress(cache["response_json"], "snappy"))
        - set(cache["cache_value"], EDXEncode(cache["compressed"], "base64"))
        - set(cache["redis_result"], EDXRedis(
            {"url": "redis://localhost:6379"},
            [{"command": "setex", "key": attributes["cache_key"], "args": [3600, cache["cache_value"]]}]
          ))
```

### 3. Batch Processing with Size Limits
**Scenario**: Compress batches to stay under transmission size limits.

```yaml
processors:
  - name: batch-compression
    transform:
      statements:
        - set(cache["batch_json"], ToJSON(attributes["events"]))
        - set(cache["compressed"], EDXCompress(cache["batch_json"], "gzip"))
        - set(cache["size_ok"], Len(cache["compressed"]) < 1000000)
        - set(attributes["batch_payload"], EDXIfElse(
            cache["size_ok"],
            cache["compressed"],
            nil  # Reject if still too large
          ))
```

### 4. Archival Storage
**Scenario**: Compress old logs before moving to cold storage.

```yaml
processors:
  - name: archive-compression
    transform:
      statements:
        - set(cache["is_old"], UnixSeconds(Now()) - attributes["timestamp"] > 604800)
        - set(cache["payload"], EDXIfElse(
            cache["is_old"],
            EDXCompress(body, "zstd"),  # High compression for archives
            body
          ))
        - set(attributes["storage_tier"], EDXIfElse(cache["is_old"], "cold", "hot"))
```

---

## Validation Rules

1. **Input Required**: The `input` parameter must not be nil or empty
2. **Valid Algorithm**: `compression_type` must be one of: "gzip", "snappy", "zstd"
3. **Case Sensitive**: Algorithm names are case-sensitive (must be lowercase)
4. **Type Validation**: Input must be string or []byte
5. **Output Type**: Always returns []byte (never string)
6. **Error on Invalid**: Returns error if compression fails or algorithm is unsupported

---

## Common Pitfalls

### 1. Forgetting to Encode for Text Storage
**Problem**: Compressed byte arrays cannot be stored as text without encoding.

```yaml
# INCORRECT: Storing raw bytes as attribute
set(attributes["compressed"], EDXCompress(body, "gzip"))
# This may cause issues if attribute is stored as string

# CORRECT: Base64 encode for text storage
set(cache["compressed"], EDXCompress(body, "gzip"))
set(attributes["compressed"], EDXEncode(cache["compressed"], "base64", false))
```

### 2. Algorithm Mismatch on Decompression
**Problem**: Using different algorithms for compression and decompression.

```yaml
# INCORRECT: Mismatch causes errors
set(cache["compressed"], EDXCompress(body, "gzip"))
set(cache["decompressed"], EDXDecompress(cache["compressed"], "snappy", false))  # ERROR!

# CORRECT: Use same algorithm
set(cache["compressed"], EDXCompress(body, "gzip"))
set(cache["decompressed"], EDXDecompress(cache["compressed"], "gzip", false))

# BEST: Store algorithm with data
set(cache["compressed"], EDXCompress(body, "gzip"))
set(attributes["compression_algorithm"], "gzip")
```

### 3. Compressing Already Compressed Data
**Problem**: Double compression provides no benefit and wastes CPU.

```yaml
# INCORRECT: Compressing gzip'd data again
set(cache["compressed_once"], EDXCompress(body, "gzip"))
set(cache["compressed_twice"], EDXCompress(cache["compressed_once"], "gzip"))  # Wasteful!

# CORRECT: Check if already compressed
set(cache["is_compressed"], attributes["content_encoding"] == "gzip")
set(cache["final"], EDXIfElse(
  cache["is_compressed"],
  body,
  EDXCompress(body, "gzip")
))
```

### 4. Not Considering Compression Overhead
**Problem**: Compressing small payloads may increase size due to algorithm overhead.

```yaml
# RISKY: Compressing tiny payloads
set(cache["compressed"], EDXCompress("Hi", "gzip"))  # May be larger than original!

# BETTER: Only compress if size exceeds threshold
set(cache["should_compress"], Len(body) > 512)
set(cache["final"], EDXIfElse(
  cache["should_compress"],
  EDXCompress(body, "gzip"),
  body
))
```

---

## Best Practices

### 1. Choose Algorithm Based on Use Case
Match algorithm to requirements:

```yaml
# High throughput, speed critical → snappy
set(cache["compressed"], EDXCompress(body, "snappy"))

# Maximum compression, archival → zstd or gzip
set(cache["compressed"], EDXCompress(body, "zstd"))

# Wide compatibility, HTTP → gzip
set(cache["compressed"], EDXCompress(body, "gzip"))
```

### 2. Always Track Compression Metadata
Store algorithm and sizes for debugging:

```yaml
processors:
  - name: compress-with-metadata
    transform:
      statements:
        - set(cache["original_size"], Len(body))
        - set(cache["compressed"], EDXCompress(body, "zstd"))
        - set(attributes["compression.algorithm"], "zstd")
        - set(attributes["compression.original_size"], cache["original_size"])
        - set(attributes["compression.compressed_size"], Len(cache["compressed"]))
        - set(attributes["compression.ratio"],
            cache["original_size"] / Len(cache["compressed"]))
```

### 3. Use Size Thresholds
Only compress when beneficial:

```yaml
processors:
  - name: threshold-compression
    transform:
      statements:
        - set(cache["size"], Len(body))
        - set(cache["should_compress"], cache["size"] > 1024)
        - set(cache["compressed"], EDXIfElse(
            cache["should_compress"],
            EDXCompress(body, "gzip"),
            body
          ))
        - set(attributes["is_compressed"], cache["should_compress"])
```

### 4. Combine with Encoding for Transport
Full pipeline for HTTP or storage:

```yaml
processors:
  - name: compress-encode-pipeline
    transform:
      statements:
        - set(cache["compressed"], EDXCompress(body, "gzip"))
        - set(cache["encoded"], EDXEncode(cache["compressed"], "base64", false))
        - set(attributes["content_encoding"], "gzip")
        - set(attributes["transfer_encoding"], "base64")
        - set(attributes["payload"], cache["encoded"])
```

---

## Performance Considerations

### Algorithm Performance Characteristics

| Algorithm | Speed       | Ratio      | CPU Usage | Best For                    |
|-----------|-------------|------------|-----------|----------------------------|
| snappy    | Fastest     | Lowest     | Low       | Real-time, high-throughput |
| gzip      | Moderate    | Good       | Moderate  | HTTP, general purpose      |
| zstd      | Fast        | Best       | Moderate  | Modern systems, archives   |

**Performance Tips:**
- **snappy**: Use for logs > 1000/sec, CPU-constrained environments
- **gzip**: Use for HTTP compatibility, moderate traffic
- **zstd**: Use for best compression, modern infrastructure

**Memory Usage:**
- Input data is copied during compression
- Compressed output allocated as new byte array
- Peak memory = input size + output size
- Consider streaming for very large inputs (> 10MB)

**CPU Impact:**
- Compression is CPU-intensive
- Consider compression threshold to avoid overhead on small payloads
- Monitor agent CPU usage under load

---

## Related Functions

- **EDXDecompress**: Decompresses data compressed by EDXCompress
- **EDXEncode**: Encode compressed bytes as base64 for text storage/transport
- **EDXDecode**: Decode base64 before decompression
- **Len()**: Standard OTTL function to check payload sizes

---

## Cross-References

- [MASTER_INDEX.md](../../MASTER_INDEX.md#edxcompress)
- [EDXDecompress](./edxdecompress.md) - Decompression counterpart
- [EDXEncode](./edxencode.md) - Encoding for transport
- [syntax_guide.md](../syntax/syntax_guide.md)

---

## Notes

- **Version History**: Introduced in v1.31.0
- **Algorithm Support**: All three algorithms are always available (no dependencies)
- **Compression Levels**: Uses default compression levels (not configurable)
- **Thread Safety**: Function is thread-safe
- **Binary Data**: Handles binary data correctly (not just text)
- **Empty Input**: Returns empty byte array if input is empty string
