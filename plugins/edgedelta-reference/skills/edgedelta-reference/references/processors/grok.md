# grok Processor

**Type**: `grok`
**Category**: Data Manipulation & Parsing
**Sequence Compatible**: ✓ Yes
**Source**: `internalv3/processors/grok/processor.go`
**Config Struct**: `configv3.Node` (Pattern, CustomPattern, PatternFieldPath, Transformation fields)

## Quick Copy

```yaml
- type: grok
  pattern: "%{IPORHOST:clientip} %{USER:ident} %{USER:auth} \\[%{HTTPDATE:timestamp}\\] \"(?:%{WORD:verb} %{NOTSPACE:request}(?: HTTP/%{NUMBER:httpversion})?|%{DATA:rawrequest})\" %{NUMBER:response} (?:%{NUMBER:bytes}|-)"
```

## Overview

The `grok` processor parses unstructured log lines into structured attributes using Grok patterns. Grok patterns are named regular expressions that simplify complex log parsing by providing reusable building blocks for common log formats. This processor is similar to Logstash Grok and supports parsing Apache, Nginx, Syslog, Java, Redis, MongoDB, and custom application logs.

## Use Cases

- **Web Server Log Parsing**: Extract structured data from Apache, Nginx, and IIS access/error logs
- **Application Log Parsing**: Parse Java, Python, Node.js, and other application logs into searchable fields
- **System Log Parsing**: Structure Syslog, systemd journal, and other system logs
- **Cloud Service Logs**: Parse AWS VPC Flow Logs, S3 Access Logs, ELB Access Logs
- **Database Logs**: Extract fields from PostgreSQL, MySQL, MongoDB, Redis, and Elasticsearch logs
- **Custom Format Parsing**: Create custom patterns for proprietary or unique log formats
- **Log Enrichment**: Add structured attributes to unstructured logs for better filtering and analysis
- **Multi-Pattern Parsing**: Handle logs with varying formats using pattern field path (dynamic pattern selection)

## Parameters

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Must be `grok` |
| `pattern` OR `custom_pattern` | string | Grok pattern to match log lines (exactly one required, unless using `pattern_field_path`) |

### Optional Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `pattern_field_path` | string | "" | Expression path to a field containing the pattern (enables dynamic pattern selection) |
| `disabled` | bool | false | Disable this processor without removing from config |
| `condition` | string | "" | OTTL condition - only process items when condition is true |
| `final` | bool | false | Mark as last processor in sequence (required for final processor) |
| `metadata` | string | "" | Arbitrary metadata for tracking/debugging |
| `comment` | string | "" | Human-readable comment about this processor |

### Parameter Notes

- **pattern vs custom_pattern**: Both fields accept the same Grok pattern syntax. `custom_pattern` is deprecated but still supported. Use `pattern` for all new configurations.
- **Mutual Exclusivity**: Cannot specify both `pattern` and `custom_pattern` simultaneously
- **Pattern Field Path**: When using `pattern_field_path`, neither `pattern` nor `custom_pattern` should be specified
- **Named Grok Patterns**: The `pattern` field can reference built-in named patterns (e.g., `nginx`, `apache_common`) by their ID from the Grok knowledge base

## Grok Pattern Syntax

Grok patterns use the syntax `%{PATTERN_NAME:field_name}` where:
- **PATTERN_NAME**: A built-in or custom pattern (e.g., `IP`, `NUMBER`, `WORD`)
- **field_name**: The attribute name to store the extracted value (lowercase with underscores recommended)

### Basic Syntax

```
%{PATTERN:field}           # Extract value as field
%{PATTERN:field:type}      # Extract and convert type (int, float, etc.)
(?:pattern)?               # Optional pattern (may or may not be present)
\\[                        # Escape special regex characters
.                          # Any single character
.*                         # Zero or more of any character (greedy)
.*?                        # Zero or more of any character (non-greedy)
```

### Pattern Modifiers

Grok patterns support type conversion through the semantic suffix:
- `%{NUMBER:bytes:int}` - Parse as integer
- `%{NUMBER:duration:float}` - Parse as float
- `%{WORD:enabled:bool}` - Parse as boolean (automatic type detection also works)

The EdgeDelta Grok processor automatically attempts to convert extracted values to native types (int, float, bool) when possible.

## Examples

### Example 1: Apache Common Log Format

```yaml
- type: grok
  pattern: "%{IPORHOST:clientip} %{USER:ident} %{USER:auth} \\[%{HTTPDATE:timestamp}\\] \"(?:%{WORD:verb} %{NOTSPACE:request}(?: HTTP/%{NUMBER:httpversion})?|%{DATA:rawrequest})\" %{NUMBER:response} (?:%{NUMBER:bytes}|-)"
```

**Input Log**:
```
172.17.0.1 - - [06/Jan/2017:16:16:37 +0000] "GET /datadoghq/company?test=var1%20Pl HTTP/1.1" 200 612
```

**Extracted Attributes**:
```yaml
clientip: "172.17.0.1"
ident: "-"
auth: "-"
timestamp: "06/Jan/2017:16:16:37 +0000"
verb: "GET"
request: "/datadoghq/company?test=var1%20Pl"
httpversion: 1.1  # automatically converted to float
response: 200     # automatically converted to int
bytes: 612        # automatically converted to int
```

### Example 2: Nginx Error Log

```yaml
- type: grok
  pattern: "%{DATA:timestamp} \\[%{WORD:level}\\] %{NUMBER:process_pid}#%{NUMBER:process_thread_id}: (\\*%{NUMBER:nginx_error_connection_id} )?%{GREEDYDATA:message}"
```

**Input Log**:
```
2024-04-17T15:23:45.678Z [ERROR] 123#456: *789 connection failed, host: example.com, port: 443
```

**Extracted Attributes**:
```yaml
timestamp: "2024-04-17T15:23:45.678Z"
level: "ERROR"
process_pid: 123
process_thread_id: 456
nginx_error_connection_id: 789
message: "connection failed, host: example.com, port: 443"
```

### Example 3: Syslog Format

```yaml
- type: grok
  pattern: "%{SYSLOGLINE}"
```

**Input Log**:
```
Jan 17 10:05:43 server1 sshd[12345]: Failed password for invalid user admin from 192.168.1.100 port 22 ssh2
```

**What it does**: Uses the built-in `SYSLOGLINE` pattern to extract timestamp, hostname, program, process ID, and message from standard syslog format.

### Example 4: Custom Application Log with Multiple Patterns

```yaml
- type: grok
  pattern: "%{TIMESTAMP_ISO8601:timestamp} \\[%{LOGLEVEL:level}\\] \\[%{DATA:component}\\] %{GREEDYDATA:message}"
```

**Input Log**:
```
2024-01-17T14:23:45.123Z [INFO] [AuthService] User john.doe authenticated successfully
```

**Extracted Attributes**:
```yaml
timestamp: "2024-01-17T14:23:45.123Z"
level: "INFO"
component: "AuthService"
message: "User john.doe authenticated successfully"
```

### Example 5: AWS VPC Flow Logs

```yaml
- type: grok
  pattern: "%{INT:vpc_version} %{NOTSPACE:aws_account_id} (?:%{NOTSPACE:network_interface}|-) (?:%{NOTSPACE:network_client_ip}|-) (?:%{NOTSPACE:network_destination_ip}|-) (?:%{INT:network_client_port}|-) (?:%{INT:network_destination_port}|-) (?:%{NOTSPACE:network_protocol}|-) (?:%{INT:network_packet}|-) (?:%{INT:network_bytes_written}|-) %{INT:vpc_interval_start} %{INT:vpc_interval_end} (?:%{WORD:vpc_action}|-) %{WORD:vpc_status}.*"
```

**Input Log**:
```
2 123456789010 eni-0123456789abcdef 203.0.113.12 172.31.16.139 20641 22 6 20 4248 1418530010 1418530070 ACCEPT OK
```

**Extracted Attributes**:
```yaml
vpc_version: 2
aws_account_id: 123456789010
network_interface: "eni-0123456789abcdef"
network_client_ip: "203.0.113.12"
network_destination_ip: "172.31.16.139"
network_client_port: 20641
network_destination_port: 22
network_protocol: 6
network_packet: 20
network_bytes_written: 4248
vpc_interval_start: 1418530010
vpc_interval_end: 1418530070
vpc_action: "ACCEPT"
vpc_status: "OK"
```

### Example 6: Redis Logs

```yaml
- type: grok
  pattern: "%{INT:pid}:%{WORD:role} %{REDIS_TIMESTAMP:timestamp} # %{NOTSPACE:level}(?:: )%{GREEDYDATA:message}"
```

**Input Log**:
```
12115:M 08 Jan 17:55:41.572 # WARNING: The TCP backlog setting of 511 cannot be enforced
```

**Extracted Attributes**:
```yaml
pid: 12115
role: "M"
timestamp: "08 Jan 17:55:41.572"
level: "WARNING"
message: "The TCP backlog setting of 511 cannot be enforced"
```

### Example 7: Java Application Logs

```yaml
- type: grok
  pattern: "\\[%{WORD:level}\\] %{JAVATHREAD:thread} %{JAVACLASS:class}: %{GREEDYDATA:message}"
```

**Input Log**:
```
[INFO] AB-Processor1 com.example.SomeClass: This is a log message from the Java application. SUCCESS
```

**Extracted Attributes**:
```yaml
level: "INFO"
thread: "AB-Processor1"
class: "com.example.SomeClass"
message: "This is a log message from the Java application. SUCCESS"
```

### Example 8: CoreDNS Query Logs

```yaml
- type: grok
  pattern: "\\[%{WORD:level}\\] %{IP:network_client_ip}:%{INT:network_client_port} - %{NUMBER:dns_id} \"%{WORD:dns_question_type} %{WORD:dns_question_class} %{HOSTNAME:dns_question_name} %{WORD:dns_protocol} %{NUMBER:dns_question_size} %{WORD:dns_dnssec} %{NUMBER:dns_buffer}\" %{WORD:dns_flags_rcode} %{NOTSPACE:dns_flags_list} %{NUMBER:dns_answer_size} %{NUMBER:duration}s.*"
```

**Input Log**:
```
[INFO] 127.0.0.1:50759 - 29008 "A IN example.org. udp 41 false 4096" NOERROR qr,rd,ra,ad 68 0.037990251s
```

**Extracted Attributes**:
```yaml
level: "INFO"
network_client_ip: "127.0.0.1"
network_client_port: 50759
dns_id: 29008
dns_question_type: "A"
dns_question_class: "IN"
dns_question_name: "example.org."
dns_protocol: "udp"
dns_question_size: 41
dns_dnssec: false
dns_buffer: 4096
dns_flags_rcode: "NOERROR"
dns_flags_list: "qr,rd,ra,ad"
dns_answer_size: 68
duration: 0.037990251
```

### Example 9: Complex Custom Log with Optional Fields

```yaml
- type: grok
  pattern: "%{IPORHOST:client_ip} - %{USER:username} \\[%{HTTPDATE:timestamp}\\] \"%{WORD:method} %{URIPATHPARAM:request_path} HTTP/%{NUMBER:http_version}\" %{NUMBER:response_code} %{NUMBER:response_bytes} \"%{DATA:referrer}\" \"%{DATA:user_agent}\" %{NUMBER:duration} %{GREEDYDATA:additional_info}"
```

**Input Log**:
```
203.0.113.12 - john.doe [17/Apr/2024:14:36:50 +0000] "GET /api/v1/users HTTP/1.1" 200 1234 "http://example.com" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" 0.254 additional_info: some additional info here
```

**Extracted Attributes**:
```yaml
client_ip: "203.0.113.12"
username: "john.doe"
timestamp: "17/Apr/2024:14:36:50 +0000"
method: "GET"
request_path: "/api/v1/users"
http_version: 1.1
response_code: 200
response_bytes: 1234
referrer: "http://example.com"
user_agent: "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
duration: 0.254
additional_info: "additional_info: some additional info here"
```

### Example 10: Dynamic Pattern Selection with Pattern Field Path

```yaml
- type: grok
  pattern_field_path: "item.attributes.log_format"
```

**Use Case**: When different log sources send different formats, store the pattern selection in an attribute. For example, Kubernetes pods might have an annotation specifying their log format.

**Input Log Item**:
```yaml
body: "172.17.0.1 - - [06/Jan/2017:16:16:37 +0000] \"GET /api HTTP/1.1\" 200 612"
attributes:
  log_format: "%{IPORHOST:clientip} %{USER:ident} %{USER:auth} \\[%{HTTPDATE:timestamp}\\] \"%{WORD:verb} %{NOTSPACE:request} HTTP/%{NUMBER:httpversion}\" %{NUMBER:response} %{NUMBER:bytes}"
```

**What it does**: Reads the Grok pattern from `item.attributes.log_format` and applies it to parse the log body.

### Example 11: Using Named Grok Knowledge Patterns

```yaml
- type: grok
  pattern: "nginx"  # References built-in NGINX_DEFAULT pattern
```

**What it does**: Uses the pre-defined Nginx pattern from EdgeDelta's Grok knowledge base. Other available named patterns include:
- `apache_common` - Apache Common Log Format
- `apache_combined` - Apache Combined Log Format
- `apache_error` - Apache Error Logs
- `nginx` - Nginx Access Logs
- `coredns` - CoreDNS Query Logs
- `redis` - Redis Server Logs

### Example 12: Unix Millisecond Timestamp Parsing

```yaml
- type: grok
  pattern: "%{NUMBER:timestamp} %{LOGLEVEL:loglevel} %{GREEDYDATA:message}"
```

**Input Log**:
```
1622547805123 INFO User login successful
```

**Extracted Attributes**:
```yaml
timestamp: 1622547805123  # automatically converted to int
loglevel: "INFO"
message: "User login successful"
```

### Example 13: Failed Parsing Handling

```yaml
- type: grok
  pattern: "%{IPORHOST:clientip} %{USER:ident} %{USER:auth}"
  condition: 'IsMatch(body, "^[0-9]+")'  # Only process logs starting with IP
```

**Input Log (Matches)**:
```
192.168.1.1 - frank
```

**Input Log (Doesn't Match - Processor Terminates)**:
```
ERROR: Something went wrong
```

**What it does**: Uses a condition to pre-filter logs before Grok parsing. Logs that don't match the pattern will terminate with an error metric recorded, but won't crash the pipeline.

## Validation Rules

1. **Pattern Required**: Must specify exactly one of `pattern`, `custom_pattern`, or `pattern_field_path`
2. **Pattern Mutual Exclusivity**: Cannot specify both `pattern` and `custom_pattern` simultaneously
3. **Pattern Field Path Exclusivity**: When using `pattern_field_path`, neither `pattern` nor `custom_pattern` should be specified
4. **Valid Grok Syntax**: Pattern must be compilable by the Grok parser
5. **Escaped Special Characters**: Regex special characters in literal strings must be escaped (e.g., `\\[`, `\\]`, `\\.`)
6. **Named Capture Groups**: All capture groups must be named using `:field_name` syntax
7. **Pattern Availability**: Referenced pattern names (e.g., `%{IP}`) must exist in built-in patterns or custom patterns
8. **Log Items Only**: Processor only operates on log items (terminates for metrics and traces)
9. **Field Path Expression**: When using `pattern_field_path`, the expression must be valid and evaluate to a string

## Common Pitfalls

### 1. Pattern Order Doesn't Matter (Single Pattern)

**Note**: Unlike some Grok implementations, EdgeDelta's Grok processor uses a single pattern per processor. For multiple pattern attempts, use multiple Grok processors with conditions or use `pattern_field_path` for dynamic selection.

**Single Pattern Approach**:
```yaml
- type: grok
  pattern: "%{APACHE_COMMON}"  # Single pattern
```

**Multi-Pattern Approach (Use Conditions)**:
```yaml
- type: grok
  condition: 'IsMatch(body, "^[0-9]{1,3}\\.")'  # IP-based logs
  pattern: "%{APACHE_COMMON}"
- type: grok
  condition: 'IsMatch(body, "^\\[")'  # Bracket-based logs
  pattern: "\\[%{TIMESTAMP_ISO8601:timestamp}\\] %{GREEDYDATA:message}"
```

### 2. Named Capture Group Syntax

**Problem**: Forgetting the colon before field name.

**Wrong**:
```yaml
pattern: "%{IP clientip}"  # Missing colon
```

**Correct**:
```yaml
pattern: "%{IP:clientip}"  # Colon before field name
```

### 3. Regex Escaping in YAML

**Problem**: YAML string escaping conflicts with regex escaping.

**Wrong**:
```yaml
pattern: "[%{TIMESTAMP}]"  # Brackets not escaped
```

**Correct**:
```yaml
pattern: "\\[%{TIMESTAMP}\\]"  # Double backslash for YAML
```

**Best Practice**: Use double quotes for patterns and escape special regex characters with `\\`.

### 4. Pattern Not Found Error

**Problem**: Referencing a pattern name that doesn't exist.

**Wrong**:
```yaml
pattern: "%{MYCOMPANYLOG:log}"  # Custom pattern not defined
```

**Correct**:
```yaml
# Use built-in patterns or define custom patterns through EdgeDelta API
pattern: "%{GREEDYDATA:log}"  # Use generic pattern
```

### 5. Greedy vs Non-Greedy Matching

**Problem**: Using greedy patterns that consume too much.

**Greedy (Takes Everything)**:
```yaml
pattern: "%{DATA:first_field} %{DATA:second_field}"
# If log is "foo bar baz", first_field gets "foo bar", second_field gets "baz"
```

**Non-Greedy (More Precise)**:
```yaml
pattern: "%{NOTSPACE:first_field} %{GREEDYDATA:second_field}"
# If log is "foo bar baz", first_field gets "foo", second_field gets "bar baz"
```

**Guidance**:
- Use `%{NOTSPACE}` for single words without spaces
- Use `%{DATA}` for non-greedy matching (stops at next pattern)
- Use `%{GREEDYDATA}` only at the end of patterns (consumes everything remaining)

### 6. Performance with Complex Patterns

**Problem**: Overly complex patterns with many optional groups slow down parsing.

**Slow**:
```yaml
pattern: "(%{WORD:field1})?(%{WORD:field2})?(%{WORD:field3})?(%{WORD:field4})?(%{WORD:field5})?"
```

**Better**:
```yaml
# Be more specific about the log structure
pattern: "%{WORD:field1} %{GREEDYDATA:remaining}"
# Then use additional processors to further parse if needed
```

### 7. Both Pattern and Custom Pattern Defined

**Problem**: Specifying both `pattern` and `custom_pattern` causes validation error.

**Wrong**:
```yaml
pattern: "%{APACHE_COMMON}"
custom_pattern: "%{NGINX_DEFAULT}"  # Both defined - ERROR
```

**Correct**:
```yaml
pattern: "%{APACHE_COMMON}"  # Use only one
```

### 8. Pattern Field Path Expression Errors

**Problem**: Invalid expression syntax when using `pattern_field_path`.

**Wrong**:
```yaml
pattern_field_path: "attributes.pattern"  # Missing item prefix
```

**Correct**:
```yaml
pattern_field_path: "item.attributes.pattern"  # Full expression path
```

### 9. Missing Field Names in Capture Groups

**Problem**: Using patterns without naming the captured values.

**Wrong**:
```yaml
pattern: "%{IP} %{USER} %{NUMBER}"  # No field names
```

**Correct**:
```yaml
pattern: "%{IP:clientip} %{USER:username} %{NUMBER:response_code}"
```

### 10. Optional Pattern Syntax

**Problem**: Incorrect syntax for optional patterns.

**Wrong**:
```yaml
pattern: "%{WORD:optional}?"  # Wrong - the ? is outside pattern
```

**Correct**:
```yaml
pattern: "(%{WORD:optional} )?"  # Entire group is optional
```

## Best Practices

1. **Start with Built-In Patterns**: Check if EdgeDelta's Grok knowledge base has a pattern for your log format before creating custom ones
2. **Test Patterns Incrementally**: Build complex patterns piece by piece, testing each addition with sample logs
3. **Use Specific Patterns**: Prefer `%{NOTSPACE}` over `%{DATA}`, and `%{INT}` over `%{NUMBER}` when you know the exact format
4. **Name Fields Descriptively**: Use lowercase with underscores (e.g., `client_ip`, `response_time_ms`)
5. **End with GREEDYDATA**: Place `%{GREEDYDATA}` at the end of patterns to capture remaining content as a message field
6. **Validate with Sample Logs**: Always test patterns against real log samples before deployment
7. **Use Conditions for Pre-Filtering**: Add `condition` to skip logs that won't match your pattern (improves performance)
8. **Document Custom Patterns**: Add `comment` field to explain what your pattern extracts
9. **Automatic Type Conversion**: Leverage automatic int/float/bool conversion instead of manual type specifications
10. **Handle Parse Failures Gracefully**: Monitor `pipeline_node_error` metrics to detect logs that fail parsing
11. **Escape Regex Special Characters**: Always escape `[`, `]`, `(`, `)`, `.`, `*`, `+`, `?`, `{`, `}`, `|`, `\` with `\\` in YAML
12. **Use Pattern Libraries**: For common formats (Apache, Nginx, Syslog), use named patterns from Grok knowledge base
13. **Avoid Over-Extraction**: Only extract fields you'll actually use for filtering, analysis, or alerting
14. **Consider Performance**: Complex patterns with many optional groups can slow parsing - keep patterns simple
15. **Dynamic Patterns for Multi-Format Logs**: Use `pattern_field_path` when processing logs from multiple sources with different formats

## Performance Considerations

- **Pattern Complexity**: Simple patterns are faster than complex patterns with many optional groups or alternations
- **Pattern Compilation**: Patterns are compiled and cached per unique pattern string - reuse patterns where possible
- **Greedy Matching**: `GREEDYDATA` patterns can be slow on large log lines - use more specific patterns when possible
- **Failed Matches**: Logs that don't match the pattern will increment error metrics but won't block the pipeline
- **Type Conversion**: Automatic type conversion (string to int/float/bool) adds minimal overhead
- **Pattern Field Path**: Dynamic pattern selection has slightly more overhead than static patterns due to expression evaluation

## Common Grok Patterns

EdgeDelta includes built-in Grok patterns from the standard Grok pattern library. Here are the most commonly used patterns:

### Fundamental Patterns

| Pattern | Description | Example Match |
|---------|-------------|---------------|
| `USERNAME` | User identifier | `john_doe`, `admin`, `user123` |
| `USER` | User identifier (same as USERNAME) | `frank`, `root`, `-` |
| `INT` | Integer number | `42`, `123`, `-5` |
| `NUMBER` | Integer or decimal | `3.14`, `42`, `-0.5` |
| `WORD` | Single word (letters, digits, underscore) | `INFO`, `Error`, `test_123` |
| `NOTSPACE` | Non-whitespace string | `example.com`, `192.168.1.1` |
| `DATA` | Any character (non-greedy) | Stops at next pattern |
| `GREEDYDATA` | Any character (greedy) | Consumes all remaining text |
| `QUOTEDSTRING` | Quoted string | `"hello world"`, `'test'` |
| `UUID` | UUID identifier | `a05c4495-ebdb-4f1d-894a-523a8d3a1d5f` |

### Network Patterns

| Pattern | Description | Example Match |
|---------|-------------|---------------|
| `IP` | IPv4 or IPv6 address | `192.168.1.1`, `::1` |
| `IPV4` | IPv4 address | `192.168.1.1` |
| `IPV6` | IPv6 address | `2001:0db8::1` |
| `IPORHOST` | IP address or hostname | `192.168.1.1`, `example.com` |
| `HOSTNAME` | DNS hostname | `server1.example.com` |
| `URIHOST` | URI host component | `example.com:8080` |
| `URIPATH` | URI path component | `/api/v1/users` |
| `URIPATHPARAM` | URI path with parameters | `/search?q=test&page=1` |
| `MAC` | MAC address | `00:1B:63:84:45:E6` |

### Time and Date Patterns

| Pattern | Description | Example Match |
|---------|-------------|---------------|
| `TIMESTAMP_ISO8601` | ISO 8601 timestamp | `2024-01-17T14:30:00.123Z` |
| `HTTPDATE` | HTTP date format | `17/Apr/2024:14:36:50 +0000` |
| `SYSLOGTIMESTAMP` | Syslog timestamp | `Jan 17 10:05:43` |
| `DATESTAMP` | Generic date stamp | `17/Apr/2024:14:36:50` |
| `YEAR` | Four-digit year | `2024` |
| `MONTH` | Month name | `January`, `Jan` |
| `MONTHNUM` | Month number | `01`, `12` |
| `MONTHDAY` | Day of month | `01`, `31` |
| `HOUR` | Hour (00-23) | `14`, `00` |
| `MINUTE` | Minute (00-59) | `30`, `05` |
| `SECOND` | Second (00-59) | `45`, `00` |
| `TIME` | Time HH:MM:SS | `14:30:45` |

### HTTP and Web Patterns

| Pattern | Description | Example Match |
|---------|-------------|---------------|
| `QS` | Quoted string | `"Mozilla/5.0..."` |
| `URI` | Complete URI | `http://example.com/path?q=1` |
| `URIPROTO` | URI protocol | `http`, `https`, `ftp` |

### Log Level Patterns

| Pattern | Description | Example Match |
|---------|-------------|---------------|
| `LOGLEVEL` | Log level | `INFO`, `ERROR`, `DEBUG`, `WARN` |

### Application-Specific Patterns

| Pattern | Description | Example Match |
|---------|-------------|---------------|
| `JAVACLASS` | Java class name | `com.example.MyClass` |
| `JAVAFILE` | Java file name | `MyClass.java` |
| `JAVAMETHOD` | Java method name | `doSomething`, `<init>` |
| `JAVATHREAD` | Java thread name | `AB-Processor1` |

### EdgeDelta Custom Patterns

| Pattern | Description | Example Match |
|---------|-------------|---------------|
| `CLICKHOUSE_TIMESTAMP` | ClickHouse timestamp | `2019.11.06 05:19:07.489819` |
| `REDIS_TIMESTAMP` | Redis timestamp | `08 Jan 17:55:41.572` |
| `MONGO3_LOG` | MongoDB 3.x log | Full MongoDB log line |
| `ELB_ACCESS_LOG` | AWS ELB access log | Full ELB access log line |
| `APACHE_COMMON` | Apache common format | Full Apache common log |
| `NGINX_DEFAULT` | Nginx default format | Full Nginx access log |
| `SYSLOGLINE` | Syslog line | Full syslog message |

### Named Knowledge Base Patterns

EdgeDelta provides pre-built patterns in the Grok knowledge base that can be referenced by ID:

| Pattern ID | Description | Use Case |
|------------|-------------|----------|
| `apache_common` | Apache Common Log Format | Standard Apache access logs |
| `apache_combined` | Apache Combined Log Format | Apache logs with referrer and user agent |
| `apache_error` | Apache/Nginx Error Logs | Web server error logs |
| `nginx` | Nginx Access Logs | Standard Nginx access logs |
| `coredns` | CoreDNS Query Logs | Kubernetes DNS query logs |
| `redis` | Redis Server Logs | Redis database logs |

## Related Processors

- **parse_json**: Parse JSON-formatted log lines (use when logs are already JSON)
- **extract_json_field**: Extract specific fields from JSON logs
- **regex_filter**: Filter logs using regex patterns (use for filtering, not parsing)
- **ottl_transform**: Transform extracted attributes after parsing
- **extract_metric**: Convert parsed log attributes into metrics
- **log_to_pattern**: Cluster logs by patterns after parsing

## Cross-References

- **edgedelta-pipelines skill**: Log parsing and enrichment pipelines
- **OTTL Reference**: https://docs.edgedelta.com/ottl-statements/
- **Grok Pattern Library**: Standard Grok patterns inherited from Logstash
- **Best Practices**: `.claude/skills/edgedelta-pipelines/assets/references/best-practices.md`

## Documentation References

- **Official Docs**: https://docs.edgedelta.com/parse-grok-processor/

## Notes

- The Grok processor only operates on log items - metrics and traces are terminated with a log process result
- Parse failures increment the `pipeline_node_error` metric but do not crash the pipeline
- Extracted values are automatically converted to native types (int, float, bool) when possible
- The processor uses the `elastic/go-grok` library under the hood
- Patterns are compiled and cached per unique pattern string for performance
- All patterns from Logstash's default pattern library are available
- Custom patterns can be added through the EdgeDelta Grok knowledge base API
- The `custom_pattern` field is deprecated - use `pattern` instead for all configurations
- Pattern field path enables dynamic pattern selection based on log attributes (e.g., Kubernetes pod annotations)
- Failed parsing is rate-limited for logging to prevent log flooding
- The processor supports both static patterns and dynamic pattern selection through expressions
- When using `pattern_field_path`, the pattern is evaluated for each log item, allowing different log formats to be processed by the same processor
