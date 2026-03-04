# EDXRedis - Redis Integration for OTTL

**Function Type**: EDX Converter
**Purpose**: Execute Redis commands directly from OTTL for stateful processing, enrichment, caching, and deduplication
**Signature**: `EDXRedis(connection_config, commands) -> map[string]any`
**Return Type**: Map with results keyed by outField names
**Minimum Agent Version**: v2.5.0
**Source**: https://docs.edgedelta.com/ottl-extensions/

---

## Table of Contents

1. [Quick Start](#quick-start)
2. [Overview](#overview)
3. [Connection Configuration](#connection-configuration)
4. [Deployment Types](#deployment-types)
5. [Authentication](#authentication)
6. [TLS/mTLS Configuration](#tlsmtls-configuration)
7. [Command Structure](#command-structure)
8. [Supported Redis Commands](#supported-redis-commands)
9. [Client Pooling and Performance](#client-pooling-and-performance)
10. [Complete Examples](#complete-examples)
11. [Common Patterns](#common-patterns)
12. [Error Handling](#error-handling)
13. [Best Practices](#best-practices)
14. [Troubleshooting](#troubleshooting)
15. [Performance Considerations](#performance-considerations)

---

## Quick Start

### Minimal Standalone Example (No Auth, No TLS)

```yaml
processors:
  - name: redis_lookup
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            # Simple GET operation
            - set(cache["redis_result"], EDXRedis(
                {"url": "redis://localhost:6379"},
                [{"outField": "value", "command": "get", "key": "my_key"}]
              ))
```

### Standalone with Authentication

```yaml
processors:
  - name: redis_lookup_with_auth
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            - set(cache["redis_result"], EDXRedis(
                {
                  "url": "redis://localhost:6379",
                  "deploymentType": "Standalone",
                  "auth": {
                    "method": "Manual",
                    "username": "admin",
                    "password": "secret123"
                  }
                },
                [{"outField": "value", "command": "get", "key": "session_id"}]
              ))
```

### Standalone with TLS

```yaml
processors:
  - name: redis_lookup_tls
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            - set(cache["redis_result"], EDXRedis(
                {
                  "url": "rediss://redis.example.com:6380",
                  "deploymentType": "Standalone",
                  "tls": {
                    "enabled": true,
                    "validateCerts": true,
                    "caCertPath": "/certs/ca.pem",
                    "serverName": "redis.example.com"
                  }
                },
                [{"outField": "value", "command": "get", "key": "secure_key"}]
              ))
```

### Cluster Deployment

```yaml
processors:
  - name: redis_cluster
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            - set(cache["redis_result"], EDXRedis(
                {
                  "url": "redis://node1.redis.local:6379,node2.redis.local:6379,node3.redis.local:6379",
                  "deploymentType": "Cluster"
                },
                [{"outField": "value", "command": "get", "key": attributes["request_id"]}]
              ))
```

### Sentinel Deployment

```yaml
processors:
  - name: redis_sentinel
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            - set(cache["redis_result"], EDXRedis(
                {
                  "url": "redis://sentinel1:26379,sentinel2:26379,sentinel3:26379",
                  "deploymentType": "Sentinel",
                  "sentinelMasterName": "redis-master"
                },
                [{"outField": "value", "command": "get", "key": "session_key"}]
              ))
```

---

## Overview

### What EDXRedis Enables

EDXRedis is the **most powerful EDX function** for stateful processing in EdgeDelta pipelines. It provides production-grade Redis integration with:

1. **Client Pooling and Reuse**: Clients are cached and reused across function evaluations based on connection configuration
2. **Multi-Deployment Support**: Standalone, Cluster, and Sentinel Redis deployments
3. **Full TLS/mTLS Support**: Certificate validation, client certificates, and custom CA certificates
4. **Thread-Safe Operations**: Mutex-protected client management
5. **Health Checking**: Automatic detection and replacement of unhealthy clients
6. **Flexible Authentication**: Manual username/password (UserSecret planned)

### Use Cases

**1. Enrichment**
- CMDB lookups for adding asset information
- IP geolocation data
- User profile enrichment
- Application metadata

**2. Caching**
- Cache expensive API responses
- Store parsed data for reuse
- Session state management
- Configuration caching

**3. Deduplication**
- Exactly-once processing with SETNX
- Event deduplication based on message IDs
- Alert suppression
- Duplicate detection across distributed agents

**4. Rate Limiting**
- Per-user rate limiting with counters
- Sliding window rate limiting
- Throttling based on keys
- Token bucket implementation

**5. Stateful Processing**
- Counter aggregation across agents
- Feature flag retrieval
- Session tracking
- Distributed locks

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ OTTL Statement calls EDXRedis(connection_config, commands)  │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ RedisClientManager (Singleton)                              │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ 1. Generate SHA256 hash of connection_config            │ │
│ │ 2. Check if client exists in pool (read lock)           │ │
│ │ 3. If exists: Verify health with PING (1s timeout)      │ │
│ │ 4. If healthy: Return existing client                   │ │
│ │ 5. If unhealthy/missing: Create new client (write lock) │ │
│ │ 6. Cache client for future evaluations                  │ │
│ └─────────────────────────────────────────────────────────┘ │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ Execute Redis Commands                                      │
│ - Build command arguments (command + key + args)            │
│ - Execute via redis.UniversalClient.Process()               │
│ - Store results in map[outField]result                      │
└─────────────────────────────────────────────────────────────┘
```

### When to Use

✅ **Use EDXRedis when:**
- You need to enrich data with external state
- You want to deduplicate events across agents
- You need distributed counters or rate limiting
- You want to cache expensive operations
- You need session state management

❌ **Do NOT use EDXRedis when:**
- You can achieve the same result with stateless processing
- Redis latency would impact throughput requirements
- You need guaranteed delivery (Redis failures block processing)
- The data changes very infrequently (consider lookup tables instead)

---

## Connection Configuration

The `connection_config` parameter is a map containing all Redis connection settings.

### RedisConnection Struct

```go
type RedisConnection struct {
    URL                string         // Required: Redis server URL
    DeploymentType     string         // "Standalone", "Cluster", "Sentinel" (default: "Standalone")
    SentinelMasterName string         // Required for Sentinel deployments
    Auth               *RedisAuth     // Optional: Authentication configuration
    TLS                *RedisTLS      // Optional: TLS/mTLS configuration
    BlockingTimeout    int            // Timeout in seconds (default: 60)
    ClientSideCache    bool           // Enable client-side caching (default: false)
    Options            map[string]any // Additional options (reserved for future use)
}
```

### Required Fields

**url** (string, required)
- Format: `redis://host:port` for plaintext
- Format: `rediss://host:port` for TLS (note double 's')
- Multi-node: `redis://host1:port,host2:port,host3:port` (for Cluster/Sentinel)

Examples:
```yaml
"url": "redis://localhost:6379"
"url": "rediss://redis.prod.example.com:6380"
"url": "redis://10.0.1.10:6379,10.0.1.11:6379,10.0.1.12:6379"
```

### Optional Fields

**deploymentType** (string, optional, default: "Standalone")
- Values: `"Standalone"`, `"Cluster"`, `"Sentinel"`
- Controls which Redis client type is created

**sentinelMasterName** (string, required for Sentinel)
- Name of the Redis master in Sentinel configuration
- Default: `"mymaster"` if not specified
- Example: `"redis-master"`, `"production-redis"`

**blockingTimeout** (int, optional, default: 60)
- Timeout in seconds for read and write operations
- Sets both `ReadTimeout` and `WriteTimeout` on the client

**clientSideCache** (bool, optional, default: false)
- Reserved for future client-side caching feature
- Currently not implemented

**options** (map[string]any, optional)
- Reserved for future extensibility
- Not currently used

### Configuration Examples

**Minimal Configuration:**
```yaml
{
  "url": "redis://localhost:6379"
}
```

**Full Configuration:**
```yaml
{
  "url": "rediss://redis-cluster-1.example.com:6380,redis-cluster-2.example.com:6380,redis-cluster-3.example.com:6380",
  "deploymentType": "Cluster",
  "auth": {
    "method": "Manual",
    "username": "edgedelta-user",
    "password": "secure-password-123"
  },
  "tls": {
    "enabled": true,
    "validateCerts": true,
    "caCertPath": "/certs/ca.pem",
    "certPath": "/certs/client.crt",
    "keyPath": "/certs/client.key",
    "serverName": "redis-cluster.example.com"
  },
  "blockingTimeout": 30
}
```

---

## Deployment Types

EDXRedis supports three Redis deployment architectures, each optimized for different use cases.

### Standalone

**Overview**: Single Redis instance for simple deployments.

**URL Format**: `redis://hostname:port`

**When to Use**:
- Development and testing
- Low-traffic production workloads
- Single-region deployments
- Cost-sensitive environments
- Simple caching use cases

**Configuration**:
```yaml
{
  "url": "redis://redis.example.com:6379",
  "deploymentType": "Standalone"
}
```

**Client Type**: `redis.Client`

**Example**:
```yaml
processors:
  - name: standalone_redis
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            - set(cache["user_profile"], EDXRedis(
                {
                  "url": "redis://10.0.1.50:6379",
                  "deploymentType": "Standalone",
                  "blockingTimeout": 10
                },
                [
                  {"outField": "profile", "command": "hgetall", "key": attributes["user_id"]}
                ]
              ))
```

**Limitations**:
- No automatic failover
- No horizontal scaling
- Single point of failure
- Limited to single-instance throughput

---

### Cluster

**Overview**: Multi-node Redis cluster with automatic sharding and high availability.

**URL Format**: `redis://node1:port,node2:port,node3:port`

**When to Use**:
- High-throughput production workloads
- Data exceeds single-instance memory
- Need horizontal scaling
- Require automatic sharding
- Multi-region replication

**Configuration**:
```yaml
{
  "url": "redis://cluster-node-1:6379,cluster-node-2:6379,cluster-node-3:6379",
  "deploymentType": "Cluster"
}
```

**Client Type**: `redis.ClusterClient`

**Example**:
```yaml
processors:
  - name: cluster_redis
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            - set(cache["dedup_check"], EDXRedis(
                {
                  "url": "redis://10.0.1.10:6379,10.0.1.11:6379,10.0.1.12:6379,10.0.1.13:6379,10.0.1.14:6379,10.0.1.15:6379",
                  "deploymentType": "Cluster",
                  "blockingTimeout": 5
                },
                [
                  {
                    "outField": "is_duplicate",
                    "command": "setnx",
                    "key": attributes["message_id"],
                    "args": ["1"]
                  },
                  {
                    "command": "expire",
                    "key": attributes["message_id"],
                    "args": ["3600"]
                  }
                ]
              ))
```

**Features**:
- Automatic data sharding across nodes
- High availability with replica failover
- Horizontal scalability
- Client-side key routing
- Cluster topology discovery

**Limitations**:
- Multi-key operations limited to same hash slot
- More complex operations (Lua scripts) may have restrictions
- Requires minimum 3 master nodes

---

### Sentinel

**Overview**: High-availability Redis with automatic failover via Sentinel coordination.

**URL Format**: `redis://sentinel1:port,sentinel2:port,sentinel3:port`

**Required Fields**:
- `sentinelMasterName`: Name of the master to monitor

**When to Use**:
- High availability required
- Automatic failover needed
- Single master with replicas
- Cost-effective HA solution
- Simpler than Cluster

**Configuration**:
```yaml
{
  "url": "redis://sentinel-1:26379,sentinel-2:26379,sentinel-3:26379",
  "deploymentType": "Sentinel",
  "sentinelMasterName": "production-master"
}
```

**Client Type**: `redis.FailoverClient`

**Default Master Name**: If `sentinelMasterName` is omitted, defaults to `"mymaster"`

**Example**:
```yaml
processors:
  - name: sentinel_redis
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            - set(cache["session_data"], EDXRedis(
                {
                  "url": "redis://sentinel1.example.com:26379,sentinel2.example.com:26379,sentinel3.example.com:26379",
                  "deploymentType": "Sentinel",
                  "sentinelMasterName": "redis-master",
                  "auth": {
                    "method": "Manual",
                    "password": "sentinel-password"
                  },
                  "blockingTimeout": 15
                },
                [
                  {"outField": "session", "command": "get", "key": attributes["session_token"]}
                ]
              ))
```

**Features**:
- Automatic master failover (30-60 seconds typical)
- Replica promotion
- Health monitoring
- Sentinel quorum-based decisions
- Simpler than Cluster for HA

**Failover Behavior**:
1. Sentinel detects master failure (configured timeout)
2. Quorum of Sentinels agree master is down
3. Sentinels elect a replica to promote
4. Client receives notification of new master
5. Client reconnects to new master automatically

**Limitations**:
- No automatic sharding (single master holds all data)
- Failover takes time (not instant)
- Requires 3+ Sentinel instances for quorum
- Write operations blocked during failover

---

## Authentication

EDXRedis supports Redis authentication via ACLs (Access Control Lists) and legacy password auth.

### RedisAuth Struct

```go
type RedisAuth struct {
    Method   string // "Manual" or "UserSecret"
    Username string // Redis ACL username (Manual method)
    Password string // Redis password (Manual method)
    SecretID string // Secret ID for UserSecret method (not yet implemented)
}
```

### Manual Authentication (Currently Supported)

**Method**: `"Manual"`

**Fields**:
- `username` (string, optional): Redis ACL username
- `password` (string, required): Redis password

**Username/Password Auth (Redis 6+)**:
```yaml
{
  "url": "redis://localhost:6379",
  "auth": {
    "method": "Manual",
    "username": "edgedelta-user",
    "password": "secure-password-123"
  }
}
```

**Password-Only Auth (Legacy Redis < 6)**:
```yaml
{
  "url": "redis://localhost:6379",
  "auth": {
    "method": "Manual",
    "password": "legacy-password"
  }
}
```

**Complete Example**:
```yaml
processors:
  - name: authenticated_redis
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            - set(cache["user_info"], EDXRedis(
                {
                  "url": "redis://secure-redis.internal:6379",
                  "deploymentType": "Standalone",
                  "auth": {
                    "method": "Manual",
                    "username": "pipeline-user",
                    "password": "${REDIS_PASSWORD}"  # Use environment variable
                  }
                },
                [
                  {"outField": "info", "command": "hgetall", "key": "user:1234"}
                ]
              ))
```

### UserSecret Authentication (Planned, Not Yet Implemented)

**Method**: `"UserSecret"`

**Fields**:
- `secretId` (string): Reference to a secret stored in EdgeDelta secret manager

**Planned Configuration**:
```yaml
{
  "url": "redis://localhost:6379",
  "auth": {
    "method": "UserSecret",
    "secretId": "redis-prod-credentials"
  }
}
```

**Current Status**: Returns error:
```
UserSecret authentication not yet implemented
```

**Workaround**: Use `"Manual"` method with environment variables for credentials:
```yaml
"auth": {
  "method": "Manual",
  "username": "${REDIS_USERNAME}",
  "password": "${REDIS_PASSWORD}"
}
```

---

## TLS/mTLS Configuration

EDXRedis provides comprehensive TLS support for encrypted connections and mTLS for mutual authentication.

### RedisTLS Struct

```go
type RedisTLS struct {
    Enabled       bool   // Enable TLS
    ValidateCerts bool   // Validate server certificates
    CACertPath    string // Path to CA certificate file
    CertPath      string // Path to client certificate (for mTLS)
    KeyPath       string // Path to client private key (for mTLS)
    ServerName    string // Expected server name in certificate (SNI)
}
```

### TLS Configuration

**Basic TLS** (server authentication only):

```yaml
{
  "url": "rediss://redis.example.com:6380",  # Note: rediss:// (double 's')
  "tls": {
    "enabled": true,
    "validateCerts": true,
    "caCertPath": "/certs/ca.pem",
    "serverName": "redis.example.com"
  }
}
```

**Field Descriptions**:

**enabled** (bool, required for TLS)
- Set to `true` to enable TLS
- URL should use `rediss://` protocol (note double 's')

**validateCerts** (bool, recommended: true)
- When `true`: Validates server certificate against CA
- When `false`: Accepts any certificate (INSECURE - use only for testing)
- Production: Always use `true`

**caCertPath** (string, required for validation)
- Absolute path to CA certificate PEM file
- Used to validate Redis server certificate
- Example: `/etc/edgedelta/certs/ca.pem`

**serverName** (string, optional but recommended)
- Expected server name in certificate (for SNI - Server Name Indication)
- Must match Common Name (CN) or Subject Alternative Name (SAN) in certificate
- Example: `"redis.example.com"`

### mTLS Configuration (Mutual TLS)

**mTLS** requires both server and client authentication:

```yaml
{
  "url": "rediss://redis-secure.internal:6380",
  "tls": {
    "enabled": true,
    "validateCerts": true,
    "caCertPath": "/certs/ca.pem",
    "certPath": "/certs/client.crt",
    "keyPath": "/certs/client.key",
    "serverName": "redis-secure.internal"
  }
}
```

**Additional Fields for mTLS**:

**certPath** (string, required for mTLS)
- Path to client certificate PEM file
- Redis server validates this certificate
- Example: `/etc/edgedelta/certs/edgedelta-client.crt`

**keyPath** (string, required for mTLS)
- Path to client private key PEM file
- Must match the client certificate
- Example: `/etc/edgedelta/certs/edgedelta-client.key`

### CRITICAL REQUIREMENT: Subject Alternative Names (SANs)

```
⚠️  CERTIFICATES MUST INCLUDE SUBJECT ALTERNATIVE NAMES (SANs)
⚠️  Common Name (CN) alone is INSUFFICIENT
⚠️  This is a TLS best practice and strict requirement
```

**Why SANs are Required**:
- Modern TLS libraries (Go 1.15+) deprecate CN-only certificates
- SANs provide flexible, explicit hostname/IP matching
- Industry best practice per RFC 6125
- Certificate authorities require SANs

**Certificate Generation with SANs**:

```bash
# Create OpenSSL config with SANs
cat > redis-cert.conf <<EOF
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[req_distinguished_name]
C = US
ST = California
L = San Francisco
O = MyCompany
CN = redis.example.com

[v3_req]
keyUsage = keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = redis.example.com
DNS.2 = redis-node1.example.com
DNS.3 = redis-node2.example.com
IP.1 = 10.0.1.50
IP.2 = 10.0.1.51
EOF

# Generate certificate with SANs
openssl req -new -x509 -days 365 -key redis.key -out redis.crt -config redis-cert.conf -extensions v3_req
```

**Verify SANs in Certificate**:
```bash
openssl x509 -in redis.crt -text -noout | grep -A 5 "Subject Alternative Name"
```

Expected output:
```
X509v3 Subject Alternative Name:
    DNS:redis.example.com, DNS:redis-node1.example.com, DNS:redis-node2.example.com, IP Address:10.0.1.50, IP Address:10.0.1.51
```

### Kubernetes Certificate Mounting

When running EdgeDelta in Kubernetes, mount certificates as volumes:

**ConfigMap for CA Certificate**:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-ca-cert
  namespace: edgedelta
data:
  ca.pem: |
    -----BEGIN CERTIFICATE-----
    MIIDXTCCAkWgAwIBAgIJAK...
    -----END CERTIFICATE-----
```

**Secret for Client Certificates** (mTLS):
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: redis-client-certs
  namespace: edgedelta
type: Opaque
data:
  client.crt: <base64-encoded-certificate>
  client.key: <base64-encoded-private-key>
```

**DaemonSet Volume Mounts**:
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: edgedelta
  namespace: edgedelta
spec:
  template:
    spec:
      containers:
      - name: edgedelta
        image: edgedelta/agent:latest
        volumeMounts:
        - name: redis-ca
          mountPath: /certs/ca.pem
          subPath: ca.pem
          readOnly: true
        - name: redis-client-certs
          mountPath: /certs/client.crt
          subPath: client.crt
          readOnly: true
        - name: redis-client-certs
          mountPath: /certs/client.key
          subPath: client.key
          readOnly: true
      volumes:
      - name: redis-ca
        configMap:
          name: redis-ca-cert
      - name: redis-client-certs
        secret:
          secretName: redis-client-certs
          defaultMode: 0400  # Read-only for owner
```

**Pipeline Configuration with K8s Paths**:
```yaml
processors:
  - name: redis_tls_k8s
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            - set(cache["result"], EDXRedis(
                {
                  "url": "rediss://redis.default.svc.cluster.local:6380",
                  "tls": {
                    "enabled": true,
                    "validateCerts": true,
                    "caCertPath": "/certs/ca.pem",
                    "certPath": "/certs/client.crt",
                    "keyPath": "/certs/client.key",
                    "serverName": "redis.default.svc.cluster.local"
                  }
                },
                [{"outField": "value", "command": "get", "key": "test"}]
              ))
```

### TLS Troubleshooting

**Error**: `x509: certificate relies on legacy Common Name field, use SANs instead`
- **Cause**: Certificate missing Subject Alternative Names
- **Fix**: Regenerate certificate with SANs (see example above)

**Error**: `x509: certificate is valid for X, not Y`
- **Cause**: `serverName` doesn't match certificate SAN
- **Fix**: Ensure `serverName` matches a DNS name or IP in certificate SANs

**Error**: `x509: certificate signed by unknown authority`
- **Cause**: CA certificate not trusted
- **Fix**: Verify `caCertPath` points to correct CA file that signed the server certificate

**Error**: `tls: failed to find certificate PEM data`
- **Cause**: Invalid certificate file format
- **Fix**: Ensure certificate is in PEM format with proper headers/footers

---

## Command Structure

Each command in the `commands` array is a `RedisCommand` object.

### RedisCommand Struct

```go
type RedisCommand struct {
    OutField string // Optional: field name for storing result in output map
    Command  string // Required: Redis command name (case-insensitive)
    Key      any    // Optional: Redis key (string or expression)
    Args     any    // Optional: Command arguments (string, array, or JSON array string)
}
```

### Field Descriptions

**outField** (string, optional)
- Field name for storing command result in the returned map
- If omitted, result is not stored (useful for write-only commands like SET)
- Example: `"result"`, `"user_profile"`, `"cached_value"`

**command** (string, required)
- Redis command name (case-insensitive)
- Examples: `"get"`, `"SET"`, `"hgetall"`, `"INCR"`
- See [Supported Redis Commands](#supported-redis-commands) for full list

**key** (any, optional)
- Redis key to operate on
- Can be:
  - String literal: `"my_key"`
  - Quoted string: `"\"my_key\""` (unquoted during execution)
  - Attribute reference: `attributes["user_id"]`
  - Expression: `Concat([attributes["prefix"], ":", attributes["id"]])`

**args** (any, optional)
- Additional arguments for the command
- Supported formats:
  - **String**: `"value1"`
  - **Array**: `["value1", "value2"]`
  - **JSON array string**: `"[\"value1\", \"value2\"]"` (parsed automatically)
  - **Numeric**: `1`, `3600`

### Command Examples

**Simple GET (no args, with outField)**:
```yaml
{
  "outField": "cached_value",
  "command": "get",
  "key": "session:12345"
}
```

**SET with single arg**:
```yaml
{
  "command": "set",
  "key": "session:12345",
  "args": "active"
}
```

**SET with array args (key + value)**:
```yaml
{
  "command": "set",
  "key": "counter",
  "args": ["100"]
}
```

**SETEX with multiple args (key + TTL + value)**:
```yaml
{
  "command": "setex",
  "key": "temp:data",
  "args": ["3600", "temporary_value"]
}
```

**INCR without outField (fire-and-forget)**:
```yaml
{
  "command": "incr",
  "key": "page_views"
}
```

**HGETALL with outField (returns hash)**:
```yaml
{
  "outField": "user_data",
  "command": "hgetall",
  "key": "user:1234"
}
```

**Dynamic key from attribute**:
```yaml
{
  "outField": "profile",
  "command": "get",
  "key": attributes["user_id"]
}
```

**JSON array args (parsed automatically)**:
```yaml
{
  "command": "mget",
  "key": "key1",
  "args": "[\"key2\", \"key3\"]"
}
```

**Quoted key (unquoted during execution)**:
```yaml
{
  "outField": "result",
  "command": "incrby",
  "key": "\"edge_delta_counter\"",
  "args": "[1]"
}
```

### Multiple Commands in Single Call

Execute multiple Redis commands in one EDXRedis call:

```yaml
- set(cache["redis_ops"], EDXRedis(
    {"url": "redis://localhost:6379"},
    [
      {
        "outField": "user_name",
        "command": "get",
        "key": "user:name:1234"
      },
      {
        "outField": "user_email",
        "command": "get",
        "key": "user:email:1234"
      },
      {
        "command": "incr",
        "key": "user:login_count:1234"
      }
    ]
  ))
```

Result stored in `cache["redis_ops"]`:
```json
{
  "user_name": "John Doe",
  "user_email": "john@example.com"
}
```

Note: The INCR command has no `outField`, so its result is not stored.

---

## Supported Redis Commands

EDXRedis supports **all standard Redis commands**. Below are categorized lists with examples.

### String Commands

**GET** - Get value of key
```yaml
{"outField": "value", "command": "get", "key": "my_key"}
```

**SET** - Set key to value
```yaml
{"command": "set", "key": "my_key", "args": "my_value"}
```

**SETEX** - Set key with expiration
```yaml
{"command": "setex", "key": "temp_key", "args": ["3600", "temp_value"]}
```

**SETNX** - Set if not exists (deduplication)
```yaml
{"outField": "is_new", "command": "setnx", "key": "message_id", "args": "1"}
```

**MGET** - Get multiple keys
```yaml
{"outField": "values", "command": "mget", "key": "key1", "args": ["key2", "key3"]}
```

**MSET** - Set multiple keys
```yaml
{"command": "mset", "key": "key1", "args": ["value1", "key2", "value2"]}
```

**GETSET** - Set value and return old value
```yaml
{"outField": "old_value", "command": "getset", "key": "counter", "args": "0"}
```

**INCR** - Increment integer value by 1
```yaml
{"command": "incr", "key": "page_views"}
```

**INCRBY** - Increment by specific amount
```yaml
{"command": "incrby", "key": "total_bytes", "args": "1024"}
```

**DECR** - Decrement integer value by 1
```yaml
{"command": "decr", "key": "remaining_quota"}
```

**DECRBY** - Decrement by specific amount
```yaml
{"command": "decrby", "key": "credits", "args": "10"}
```

**APPEND** - Append to string value
```yaml
{"command": "append", "key": "log_buffer", "args": "new_log_line\n"}
```

**STRLEN** - Get length of string
```yaml
{"outField": "length", "command": "strlen", "key": "message"}
```

---

### Hash Commands

**HGET** - Get hash field value
```yaml
{"outField": "name", "command": "hget", "key": "user:1234", "args": "name"}
```

**HSET** - Set hash field
```yaml
{"command": "hset", "key": "user:1234", "args": ["name", "John Doe"]}
```

**HGETALL** - Get all hash fields and values
```yaml
{"outField": "user_data", "command": "hgetall", "key": "user:1234"}
```

**HMGET** - Get multiple hash fields
```yaml
{"outField": "fields", "command": "hmget", "key": "user:1234", "args": ["name", "email"]}
```

**HMSET** - Set multiple hash fields
```yaml
{"command": "hmset", "key": "user:1234", "args": ["name", "John", "email", "john@example.com"]}
```

**HDEL** - Delete hash field
```yaml
{"command": "hdel", "key": "user:1234", "args": "temp_field"}
```

**HEXISTS** - Check if hash field exists
```yaml
{"outField": "exists", "command": "hexists", "key": "user:1234", "args": "email"}
```

**HKEYS** - Get all hash field names
```yaml
{"outField": "fields", "command": "hkeys", "key": "user:1234"}
```

**HVALS** - Get all hash values
```yaml
{"outField": "values", "command": "hvals", "key": "user:1234"}
```

**HINCRBY** - Increment hash field integer
```yaml
{"command": "hincrby", "key": "stats:1234", "args": ["views", "1"]}
```

**HLEN** - Get number of hash fields
```yaml
{"outField": "count", "command": "hlen", "key": "user:1234"}
```

---

### List Commands

**LPUSH** - Push to head of list
```yaml
{"command": "lpush", "key": "queue:tasks", "args": "task1"}
```

**RPUSH** - Push to tail of list
```yaml
{"command": "rpush", "key": "queue:tasks", "args": "task2"}
```

**LPOP** - Pop from head of list
```yaml
{"outField": "task", "command": "lpop", "key": "queue:tasks"}
```

**RPOP** - Pop from tail of list
```yaml
{"outField": "task", "command": "rpop", "key": "queue:tasks"}
```

**LRANGE** - Get range of list elements
```yaml
{"outField": "items", "command": "lrange", "key": "recent:events", "args": ["0", "9"]}
```

**LLEN** - Get list length
```yaml
{"outField": "length", "command": "llen", "key": "queue:tasks"}
```

**LINDEX** - Get element by index
```yaml
{"outField": "item", "command": "lindex", "key": "list", "args": "0"}
```

**LSET** - Set element at index
```yaml
{"command": "lset", "key": "list", "args": ["0", "new_value"]}
```

**LTRIM** - Trim list to range
```yaml
{"command": "ltrim", "key": "recent:events", "args": ["0", "99"]}
```

---

### Set Commands

**SADD** - Add member to set
```yaml
{"command": "sadd", "key": "active:users", "args": "user123"}
```

**SREM** - Remove member from set
```yaml
{"command": "srem", "key": "active:users", "args": "user123"}
```

**SMEMBERS** - Get all set members
```yaml
{"outField": "users", "command": "smembers", "key": "active:users"}
```

**SISMEMBER** - Check if member in set
```yaml
{"outField": "is_active", "command": "sismember", "key": "active:users", "args": "user123"}
```

**SCARD** - Get set cardinality (size)
```yaml
{"outField": "count", "command": "scard", "key": "active:users"}
```

**SUNION** - Union of multiple sets
```yaml
{"outField": "all_users", "command": "sunion", "key": "set1", "args": ["set2", "set3"]}
```

**SINTER** - Intersection of sets
```yaml
{"outField": "common", "command": "sinter", "key": "set1", "args": "set2"}
```

**SDIFF** - Difference of sets
```yaml
{"outField": "unique", "command": "sdiff", "key": "set1", "args": "set2"}
```

**SPOP** - Remove and return random member
```yaml
{"outField": "member", "command": "spop", "key": "lottery:entries"}
```

---

### Sorted Set Commands

**ZADD** - Add member with score
```yaml
{"command": "zadd", "key": "leaderboard", "args": ["100", "player1"]}
```

**ZREM** - Remove member
```yaml
{"command": "zrem", "key": "leaderboard", "args": "player1"}
```

**ZRANGE** - Get range by rank
```yaml
{"outField": "top10", "command": "zrange", "key": "leaderboard", "args": ["0", "9"]}
```

**ZRANGEBYSCORE** - Get range by score
```yaml
{"outField": "high_scores", "command": "zrangebyscore", "key": "leaderboard", "args": ["100", "1000"]}
```

**ZRANK** - Get member rank
```yaml
{"outField": "rank", "command": "zrank", "key": "leaderboard", "args": "player1"}
```

**ZSCORE** - Get member score
```yaml
{"outField": "score", "command": "zscore", "key": "leaderboard", "args": "player1"}
```

**ZINCRBY** - Increment member score
```yaml
{"command": "zincrby", "key": "leaderboard", "args": ["10", "player1"]}
```

**ZCARD** - Get sorted set size
```yaml
{"outField": "count", "command": "zcard", "key": "leaderboard"}
```

**ZCOUNT** - Count members in score range
```yaml
{"outField": "count", "command": "zcount", "key": "leaderboard", "args": ["0", "100"]}
```

---

### Key Management Commands

**EXISTS** - Check if key exists
```yaml
{"outField": "exists", "command": "exists", "key": "my_key"}
```

**DEL** - Delete key
```yaml
{"command": "del", "key": "old_key"}
```

**EXPIRE** - Set key expiration (seconds)
```yaml
{"command": "expire", "key": "session:123", "args": "3600"}
```

**TTL** - Get time to live (seconds)
```yaml
{"outField": "ttl", "command": "ttl", "key": "session:123"}
```

**PERSIST** - Remove key expiration
```yaml
{"command": "persist", "key": "permanent_key"}
```

**TYPE** - Get key type
```yaml
{"outField": "type", "command": "type", "key": "my_key"}
```

**RENAME** - Rename key
```yaml
{"command": "rename", "key": "old_name", "args": "new_name"}
```

**SCAN** - Incrementally iterate keys (cursor-based)
```yaml
{"outField": "keys", "command": "scan", "key": "0", "args": ["MATCH", "user:*", "COUNT", "100"]}
```

---

## Client Pooling and Performance

EDXRedis implements sophisticated client pooling to maximize performance and minimize overhead.

### Architecture Overview

**RedisClientManager** is a singleton that manages all Redis client instances:

```go
type RedisClientManager struct {
    clients map[string]redis.UniversalClient  // Pooled clients by config hash
    mutex   sync.RWMutex                       // Thread-safe access
}
```

### How Client Pooling Works

**1. Configuration Hashing**

Each connection configuration is hashed to create a unique cache key:

```go
func (m *RedisClientManager) generateConfigKey(config *RedisConnection) string {
    configBytes, _ := jsonext.Marshal(config)
    hash := sha256.Sum256(configBytes)
    return hex.EncodeToString(hash[:])
}
```

- **Deterministic**: Same configuration always generates same hash
- **Collision-resistant**: SHA256 ensures uniqueness
- **Efficient**: Fast lookup in map

**2. GetOrCreateClient Pattern**

```go
func (m *RedisClientManager) GetOrCreateClient(config *RedisConnection) (redis.UniversalClient, error) {
    configKey := m.generateConfigKey(config)

    // Try read lock first (fast path)
    m.mutex.RLock()
    if client, exists := m.clients[configKey]; exists {
        m.mutex.RUnlock()

        // Health check existing client
        if m.isClientHealthy(client) {
            return client, nil  // Reuse healthy client
        }

        // Unhealthy - remove and recreate
        m.RemoveClient(config)
    } else {
        m.mutex.RUnlock()
    }

    // Create new client with write lock
    m.mutex.Lock()
    defer m.mutex.Unlock()

    // Double-check pattern (another goroutine may have created it)
    if client, exists := m.clients[configKey]; exists {
        if m.isClientHealthy(client) {
            return client, nil
        }
        _ = client.Close()
        delete(m.clients, configKey)
    }

    // Create and cache new client
    client, err := createRedisClient(config)
    if err != nil {
        return nil, fmt.Errorf("failed to create Redis client: %w", err)
    }

    m.clients[configKey] = client
    return client, nil
}
```

**Key Benefits**:
- **Read-lock fast path**: Most lookups use read-only lock (minimal contention)
- **Double-check locking**: Prevents race conditions during client creation
- **Automatic recovery**: Unhealthy clients replaced transparently

**3. Health Checking**

Every client reuse includes a health check:

```go
func (m *RedisClientManager) isClientHealthy(client redis.UniversalClient) bool {
    ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
    defer cancel()

    _, err := client.Ping(ctx).Result()
    return err == nil
}
```

- **Fast**: 1-second timeout prevents blocking
- **Simple**: PING command lightweight
- **Reliable**: Detects network failures, server issues

**4. Thread Safety**

All client operations protected by mutex:
- **Read lock** (`RLock`): Concurrent reads allowed
- **Write lock** (`Lock`): Exclusive for modifications
- **Prevents**: Race conditions, data corruption

### Performance Optimizations

**Clients NOT Created on Every Evaluation**

```
❌ WITHOUT client pooling (naive approach):
   Every EDXRedis() call → New TCP connection → TLS handshake → Auth → Command → Close

✅ WITH client pooling (EDXRedis implementation):
   First EDXRedis() call → Create client → Cache by config hash
   Subsequent calls → Retrieve from cache → Reuse connection → Execute command
```

**Benchmark Impact**:
- **Connection overhead eliminated**: No repeated TCP/TLS handshakes
- **Auth overhead eliminated**: No repeated AUTH commands
- **Memory efficient**: One client per unique configuration, not per evaluation
- **Throughput increase**: 100x-1000x faster than creating clients per call

**Configuration Key Generation Performance**:
```
Hash generation: ~100-200 nanoseconds
Map lookup: ~20-50 nanoseconds
Total overhead: <1 microsecond
```

**Health Check Overhead**:
```
PING command: ~0.5-2 milliseconds (local network)
PING command: ~10-50 milliseconds (cross-region)
Frequency: Only on cache hit (not every command)
```

**Memory Efficiency**:

One client handles connection pooling internally (go-redis default: 10 connections per client):
```
1 unique config = 1 RedisClientManager entry
               = 1 redis.UniversalClient
               = ~10 TCP connections (go-redis pool)
               = ~1-2 MB memory
```

**Multiple Pipelines, Same Config**:
```
Pipeline A: EDXRedis({url: "redis://host:6379"}, [...])  → Client 1 (cached)
Pipeline B: EDXRedis({url: "redis://host:6379"}, [...])  → Client 1 (reused)
Pipeline C: EDXRedis({url: "redis://host:6379"}, [...])  → Client 1 (reused)

Result: 1 client shared across all pipelines with identical config
```

### Client Lifecycle

**Creation**:
1. First EDXRedis call with new config
2. Config hashed (SHA256)
3. Client created based on deployment type
4. Client cached in manager
5. Client returned to caller

**Reuse**:
1. Subsequent EDXRedis call with same config
2. Config hashed (identical hash generated)
3. Client retrieved from cache
4. Health check (PING)
5. If healthy: Client reused
6. If unhealthy: Client removed, new client created

**Cleanup**:
- **Automatic**: Unhealthy clients removed on next access
- **Manual**: `CloseAll()` for graceful shutdown
- **Background**: `CleanupUnhealthyClients()` periodic cleanup (optional)

### Monitoring Client Pool

**Get Client Count**:
```go
count := clientManager.GetClientCount()
// Returns number of active clients in pool
```

**Typical Scenarios**:
- 1 client: Single Redis config across all pipelines
- 2-3 clients: Dev/staging/prod Redis instances
- 5-10 clients: Multiple clusters, different TLS configs
- 50+ clients: Likely configuration issue (review config uniqueness)

**Cleanup Unhealthy Clients** (optional background task):
```go
// Periodically remove unhealthy clients
clientManager.CleanupUnhealthyClients()
```

**Graceful Shutdown**:
```go
// Close all clients (e.g., on agent shutdown)
clientManager.CloseAll()
```

### Best Practices for Client Pooling

1. **Reuse configurations**: Identical configs share clients
2. **Avoid dynamic config values**: Don't include timestamps, random values in config
3. **Use environment variables**: For credentials, not inline in config (enables reuse)
4. **Monitor client count**: High count may indicate config duplication
5. **Trust the pool**: Don't manually create clients, let EDXRedis manage

**Good** (config reuse):
```yaml
# All logs use same client
- set(cache["user1"], EDXRedis({"url": "redis://host:6379"}, [...]))
- set(cache["user2"], EDXRedis({"url": "redis://host:6379"}, [...]))
```

**Bad** (creates new client every time):
```yaml
# Different config each time (due to timestamp)
- set(cache["result"], EDXRedis(
    {"url": "redis://host:6379", "timestamp": Now()},
    [...]
  ))
```

---

## Complete Examples

### Example 1: Basic GET/SET

```yaml
processors:
  - name: basic_redis_ops
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            # Set a value
            - set(cache["set_result"], EDXRedis(
                {"url": "redis://localhost:6379"},
                [
                  {"command": "set", "key": "app:status", "args": "healthy"}
                ]
              ))

            # Get the value
            - set(cache["get_result"], EDXRedis(
                {"url": "redis://localhost:6379"},
                [
                  {"outField": "status", "command": "get", "key": "app:status"}
                ]
              ))

            # Result: cache["get_result"]["status"] = "healthy"
```

---

### Example 2: Hash Operations (HGET/HSET)

```yaml
processors:
  - name: user_profile_enrichment
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            # Store user profile as hash
            - set(cache["store_profile"], EDXRedis(
                {"url": "redis://localhost:6379"},
                [
                  {
                    "command": "hmset",
                    "key": "user:12345",
                    "args": ["name", "John Doe", "email", "john@example.com", "role", "admin"]
                  }
                ]
              ))

            # Retrieve specific field
            - set(cache["user_name"], EDXRedis(
                {"url": "redis://localhost:6379"},
                [
                  {"outField": "name", "command": "hget", "key": "user:12345", "args": "name"}
                ]
              ))

            # Retrieve entire profile
            - set(cache["user_profile"], EDXRedis(
                {"url": "redis://localhost:6379"},
                [
                  {"outField": "profile", "command": "hgetall", "key": "user:12345"}
                ]
              ))

            # Enrich log with user name
            - set(attributes["user_name"], cache["user_name"]["name"])
```

---

### Example 3: TLS Connection

```yaml
processors:
  - name: secure_redis_lookup
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            - set(cache["secure_data"], EDXRedis(
                {
                  "url": "rediss://redis-prod.example.com:6380",
                  "deploymentType": "Standalone",
                  "tls": {
                    "enabled": true,
                    "validateCerts": true,
                    "caCertPath": "/etc/edgedelta/certs/ca.pem",
                    "serverName": "redis-prod.example.com"
                  },
                  "blockingTimeout": 10
                },
                [
                  {"outField": "api_key", "command": "get", "key": attributes["app_id"]}
                ]
              ))
```

---

### Example 4: mTLS Connection

```yaml
processors:
  - name: mtls_redis_lookup
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            - set(cache["mtls_result"], EDXRedis(
                {
                  "url": "rediss://redis-secure.internal:6380",
                  "deploymentType": "Standalone",
                  "auth": {
                    "method": "Manual",
                    "username": "edgedelta-client",
                    "password": "${REDIS_PASSWORD}"
                  },
                  "tls": {
                    "enabled": true,
                    "validateCerts": true,
                    "caCertPath": "/certs/ca.pem",
                    "certPath": "/certs/edgedelta-client.crt",
                    "keyPath": "/certs/edgedelta-client.key",
                    "serverName": "redis-secure.internal"
                  }
                },
                [
                  {"outField": "secret_config", "command": "hgetall", "key": "app:secrets"}
                ]
              ))
```

---

### Example 5: Cluster Deployment

```yaml
processors:
  - name: cluster_deduplication
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            # Attempt to deduplicate across cluster
            - set(cache["dedup"], EDXRedis(
                {
                  "url": "redis://cluster-node-1:6379,cluster-node-2:6379,cluster-node-3:6379,cluster-node-4:6379,cluster-node-5:6379,cluster-node-6:6379",
                  "deploymentType": "Cluster",
                  "blockingTimeout": 5
                },
                [
                  {
                    "outField": "is_new",
                    "command": "setnx",
                    "key": attributes["message_id"],
                    "args": "1"
                  },
                  {
                    "command": "expire",
                    "key": attributes["message_id"],
                    "args": ["86400"]
                  }
                ]
              ))

            # Drop duplicate messages
            - drop() where cache["dedup"]["is_new"] == 0
```

---

### Example 6: Sentinel Deployment

```yaml
processors:
  - name: sentinel_session_state
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            - set(cache["session"], EDXRedis(
                {
                  "url": "redis://sentinel-1.prod:26379,sentinel-2.prod:26379,sentinel-3.prod:26379",
                  "deploymentType": "Sentinel",
                  "sentinelMasterName": "production-master",
                  "auth": {
                    "method": "Manual",
                    "password": "${REDIS_SENTINEL_PASSWORD}"
                  },
                  "blockingTimeout": 15
                },
                [
                  {
                    "outField": "user_session",
                    "command": "hgetall",
                    "key": Concat(["session:", attributes["session_token"]])
                  }
                ]
              ))

            # Enrich with session data
            - set(attributes["user_id"], cache["session"]["user_session"]["user_id"]) where cache["session"]["user_session"] != nil
```

---

### Example 7: Multiple Commands in One Call

```yaml
processors:
  - name: multi_command_redis
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            - set(cache["multi_ops"], EDXRedis(
                {"url": "redis://localhost:6379"},
                [
                  # Get user name
                  {
                    "outField": "user_name",
                    "command": "get",
                    "key": Concat(["user:name:", attributes["user_id"]])
                  },
                  # Get user email
                  {
                    "outField": "user_email",
                    "command": "get",
                    "key": Concat(["user:email:", attributes["user_id"]])
                  },
                  # Increment login counter (no outField)
                  {
                    "command": "incr",
                    "key": Concat(["user:login_count:", attributes["user_id"]])
                  },
                  # Update last seen timestamp
                  {
                    "command": "set",
                    "key": Concat(["user:last_seen:", attributes["user_id"]]),
                    "args": [Now()]
                  }
                ]
              ))

            # Enrich log with user data
            - set(attributes["user_name"], cache["multi_ops"]["user_name"])
            - set(attributes["user_email"], cache["multi_ops"]["user_email"])
```

---

### Example 8: Cache-Aside Pattern

```yaml
processors:
  - name: cache_aside_api_response
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            # Check cache first
            - set(cache["api_data"], EDXRedis(
                {"url": "redis://localhost:6379"},
                [
                  {
                    "outField": "cached_response",
                    "command": "get",
                    "key": Concat(["api:response:", attributes["api_endpoint"]])
                  }
                ]
              ))

            # If cache miss, call API (pseudo-code - use actual API call mechanism)
            # Then store in cache with 5-minute TTL
            - set(cache["store_api"], EDXRedis(
                {"url": "redis://localhost:6379"},
                [
                  {
                    "command": "setex",
                    "key": Concat(["api:response:", attributes["api_endpoint"]]),
                    "args": ["300", attributes["api_response_body"]]
                  }
                ]
              )) where cache["api_data"]["cached_response"] == nil

            # Use cached data if available, otherwise use fresh API response
            - set(attributes["enriched_data"], cache["api_data"]["cached_response"]) where cache["api_data"]["cached_response"] != nil
```

---

### Example 9: Deduplication with SETNX

```yaml
processors:
  - name: deduplication
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            # Try to set message ID as dedup key
            - set(cache["dedup_check"], EDXRedis(
                {"url": "redis://localhost:6379"},
                [
                  {
                    "outField": "is_unique",
                    "command": "setnx",
                    "key": Concat(["dedup:", attributes["message_id"]]),
                    "args": "1"
                  },
                  {
                    "command": "expire",
                    "key": Concat(["dedup:", attributes["message_id"]]),
                    "args": ["3600"]
                  }
                ]
              ))

            # Drop duplicates (is_unique = 0 means key already existed)
            - drop() where cache["dedup_check"]["is_unique"] == 0
```

---

### Example 10: Rate Limiting with INCR + EXPIRE

```yaml
processors:
  - name: rate_limiting
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            # Increment request counter for user
            - set(cache["rate_limit"], EDXRedis(
                {"url": "redis://localhost:6379"},
                [
                  {
                    "outField": "request_count",
                    "command": "incr",
                    "key": Concat(["rate_limit:", attributes["user_id"], ":", Now().Truncate(60)])
                  },
                  {
                    "command": "expire",
                    "key": Concat(["rate_limit:", attributes["user_id"], ":", Now().Truncate(60)]),
                    "args": ["60"]
                  }
                ]
              ))

            # Drop requests exceeding 100 per minute
            - drop() where cache["rate_limit"]["request_count"] > 100
```

---

### Example 11: CMDB Enrichment

```yaml
processors:
  - name: cmdb_enrichment
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            # Lookup asset info from CMDB stored in Redis
            - set(cache["cmdb"], EDXRedis(
                {"url": "redis://cmdb-cache:6379"},
                [
                  {
                    "outField": "asset_info",
                    "command": "hgetall",
                    "key": Concat(["cmdb:asset:", attributes["hostname"]])
                  }
                ]
              ))

            # Enrich log with CMDB data
            - set(attributes["asset_owner"], cache["cmdb"]["asset_info"]["owner"]) where cache["cmdb"]["asset_info"] != nil
            - set(attributes["asset_location"], cache["cmdb"]["asset_info"]["location"]) where cache["cmdb"]["asset_info"] != nil
            - set(attributes["asset_criticality"], cache["cmdb"]["asset_info"]["criticality"]) where cache["cmdb"]["asset_info"] != nil
```

---

### Example 12: Session State Management

```yaml
processors:
  - name: session_tracking
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            # Update session state
            - set(cache["session_update"], EDXRedis(
                {"url": "redis://session-store:6379"},
                [
                  {
                    "command": "hmset",
                    "key": Concat(["session:", attributes["session_id"]]),
                    "args": [
                      "last_activity", Now(),
                      "ip_address", attributes["client_ip"],
                      "user_agent", attributes["user_agent"]
                    ]
                  },
                  {
                    "command": "expire",
                    "key": Concat(["session:", attributes["session_id"]]),
                    "args": ["1800"]
                  }
                ]
              ))
```

---

### Example 13: Batch Operations (MGET/MSET)

```yaml
processors:
  - name: batch_user_lookup
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            # Retrieve multiple user names in one call
            - set(cache["batch_users"], EDXRedis(
                {"url": "redis://localhost:6379"},
                [
                  {
                    "outField": "user_names",
                    "command": "mget",
                    "key": "user:name:123",
                    "args": ["user:name:456", "user:name:789"]
                  }
                ]
              ))

            # Result: cache["batch_users"]["user_names"] = ["Alice", "Bob", "Charlie"]
```

---

### Example 14: List-Based Queue (LPUSH/RPOP)

```yaml
processors:
  - name: event_queue
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            # Push event to queue
            - set(cache["queue_push"], EDXRedis(
                {"url": "redis://queue:6379"},
                [
                  {
                    "command": "lpush",
                    "key": "events:queue",
                    "args": [attributes["event_json"]]
                  }
                ]
              ))

            # Limit queue size to 1000 items
            - set(cache["queue_trim"], EDXRedis(
                {"url": "redis://queue:6379"},
                [
                  {
                    "command": "ltrim",
                    "key": "events:queue",
                    "args": ["0", "999"]
                  }
                ]
              ))
```

---

### Example 15: Sorted Set Leaderboard (ZADD/ZRANGE)

```yaml
processors:
  - name: leaderboard_tracking
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            # Update user score in leaderboard
            - set(cache["leaderboard_update"], EDXRedis(
                {"url": "redis://leaderboard:6379"},
                [
                  {
                    "command": "zincrby",
                    "key": "leaderboard:daily",
                    "args": [attributes["score_increment"], attributes["user_id"]]
                  }
                ]
              ))

            # Get top 10 players
            - set(cache["top_players"], EDXRedis(
                {"url": "redis://leaderboard:6379"},
                [
                  {
                    "outField": "top10",
                    "command": "zrevrange",
                    "key": "leaderboard:daily",
                    "args": ["0", "9", "WITHSCORES"]
                  }
                ]
              ))
```

---

## Common Patterns

### Pattern 1: Cache-Aside (Read-Through Cache)

**Use Case**: Reduce expensive API calls by caching responses.

**Implementation**:
```yaml
processors:
  - name: cache_aside
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            # 1. Check cache
            - set(cache["lookup"], EDXRedis(
                {"url": "redis://localhost:6379"},
                [
                  {
                    "outField": "cached_value",
                    "command": "get",
                    "key": Concat(["cache:", attributes["lookup_key"]])
                  }
                ]
              ))

            # 2. If cache hit, use cached value
            - set(attributes["enriched_data"], cache["lookup"]["cached_value"]) where cache["lookup"]["cached_value"] != nil

            # 3. If cache miss, fetch from source (pseudo-code)
            # ... call API or database ...

            # 4. Store in cache with TTL
            - set(cache["store"], EDXRedis(
                {"url": "redis://localhost:6379"},
                [
                  {
                    "command": "setex",
                    "key": Concat(["cache:", attributes["lookup_key"]]),
                    "args": ["300", attributes["fresh_data"]]
                  }
                ]
              )) where cache["lookup"]["cached_value"] == nil
```

**Benefits**:
- Reduces load on backing services
- Improves latency for repeated lookups
- Configurable TTL balances freshness vs performance

---

### Pattern 2: Write-Through Cache

**Use Case**: Keep cache synchronized with writes.

**Implementation**:
```yaml
processors:
  - name: write_through
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            # Update both database and cache atomically
            # 1. Write to database (pseudo-code)
            # ... database update ...

            # 2. Update cache immediately
            - set(cache["cache_update"], EDXRedis(
                {"url": "redis://localhost:6379"},
                [
                  {
                    "command": "set",
                    "key": Concat(["cache:", attributes["record_id"]]),
                    "args": [attributes["updated_value"]]
                  },
                  {
                    "command": "expire",
                    "key": Concat(["cache:", attributes["record_id"]]),
                    "args": ["3600"]
                  }
                ]
              ))
```

**Benefits**:
- Cache always consistent with database
- Subsequent reads served from cache
- Reduces cache miss rate

---

### Pattern 3: Exactly-Once Deduplication (SETNX + TTL)

**Use Case**: Ensure events are processed exactly once.

**Implementation**:
```yaml
processors:
  - name: exactly_once_dedup
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            # Try to acquire lock with SETNX
            - set(cache["dedup"], EDXRedis(
                {"url": "redis://localhost:6379"},
                [
                  {
                    "outField": "acquired",
                    "command": "setnx",
                    "key": Concat(["processed:", attributes["event_id"]]),
                    "args": "1"
                  },
                  {
                    "command": "expire",
                    "key": Concat(["processed:", attributes["event_id"]]),
                    "args": ["86400"]
                  }
                ]
              ))

            # Process only if lock acquired (acquired = 1)
            # Drop if already processed (acquired = 0)
            - drop() where cache["dedup"]["acquired"] == 0
```

**Benefits**:
- Guaranteed exactly-once processing
- TTL prevents unbounded growth
- Works across distributed agents

---

### Pattern 4: Sliding Window Rate Limiting

**Use Case**: Limit requests per user per time window.

**Implementation**:
```yaml
processors:
  - name: sliding_window_rate_limit
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            # Create time-based key (1-minute buckets)
            - set(cache["current_minute"], Now().Truncate(60))

            # Increment counter for current minute
            - set(cache["rate_check"], EDXRedis(
                {"url": "redis://localhost:6379"},
                [
                  {
                    "outField": "request_count",
                    "command": "incr",
                    "key": Concat(["rate:", attributes["user_id"], ":", cache["current_minute"]])
                  },
                  {
                    "command": "expire",
                    "key": Concat(["rate:", attributes["user_id"], ":", cache["current_minute"]]),
                    "args": ["120"]
                  }
                ]
              ))

            # Allow max 100 requests per minute
            - drop() where cache["rate_check"]["request_count"] > 100
```

**Benefits**:
- Fair per-user limits
- Sliding window (not fixed interval)
- Automatic cleanup with TTL

---

### Pattern 5: CMDB Enrichment with Fallback

**Use Case**: Enrich logs with asset data from CMDB, with fallback handling.

**Implementation**:
```yaml
processors:
  - name: cmdb_enrichment_with_fallback
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            # Lookup asset in CMDB cache
            - set(cache["cmdb"], EDXRedis(
                {"url": "redis://cmdb-cache:6379"},
                [
                  {
                    "outField": "asset",
                    "command": "hgetall",
                    "key": Concat(["cmdb:", attributes["hostname"]])
                  }
                ]
              ))

            # Enrich if found
            - set(attributes["asset_owner"], cache["cmdb"]["asset"]["owner"]) where cache["cmdb"]["asset"] != nil
            - set(attributes["asset_location"], cache["cmdb"]["asset"]["location"]) where cache["cmdb"]["asset"] != nil

            # Fallback to "unknown" if not found
            - set(attributes["asset_owner"], "unknown") where cache["cmdb"]["asset"] == nil
            - set(attributes["asset_location"], "unknown") where cache["cmdb"]["asset"] == nil
```

**Benefits**:
- Graceful degradation
- Logs always have enrichment fields (even if "unknown")
- No pipeline failures on cache miss

---

### Pattern 6: Session State Lookup

**Use Case**: Track user session state across requests.

**Implementation**:
```yaml
processors:
  - name: session_state_tracking
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            # Retrieve session state
            - set(cache["session"], EDXRedis(
                {"url": "redis://session-store:6379"},
                [
                  {
                    "outField": "state",
                    "command": "hgetall",
                    "key": Concat(["session:", attributes["session_token"]])
                  }
                ]
              ))

            # Enrich log with session data
            - set(attributes["user_id"], cache["session"]["state"]["user_id"]) where cache["session"]["state"] != nil
            - set(attributes["session_start"], cache["session"]["state"]["start_time"]) where cache["session"]["state"] != nil

            # Update last activity timestamp
            - set(cache["update_session"], EDXRedis(
                {"url": "redis://session-store:6379"},
                [
                  {
                    "command": "hset",
                    "key": Concat(["session:", attributes["session_token"]]),
                    "args": ["last_activity", Now()]
                  },
                  {
                    "command": "expire",
                    "key": Concat(["session:", attributes["session_token"]]),
                    "args": ["1800"]
                  }
                ]
              )) where cache["session"]["state"] != nil
```

**Benefits**:
- Stateful session tracking
- Automatic expiration (30-minute idle timeout)
- Enriches logs with user context

---

### Pattern 7: Feature Flag Retrieval

**Use Case**: Control feature enablement via Redis-backed feature flags.

**Implementation**:
```yaml
processors:
  - name: feature_flags
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            # Check if feature enabled for user
            - set(cache["feature_flag"], EDXRedis(
                {"url": "redis://feature-flags:6379"},
                [
                  {
                    "outField": "enabled",
                    "command": "sismember",
                    "key": "feature:new_ui:users",
                    "args": [attributes["user_id"]]
                  }
                ]
              ))

            # Tag log with feature flag status
            - set(attributes["feature_new_ui_enabled"], cache["feature_flag"]["enabled"] == 1)

            # Conditional processing based on flag
            - set(attributes["ui_version"], "v2") where cache["feature_flag"]["enabled"] == 1
            - set(attributes["ui_version"], "v1") where cache["feature_flag"]["enabled"] == 0
```

**Benefits**:
- Dynamic feature control without deployment
- Gradual rollout to specific users
- A/B testing support

---

### Pattern 8: Counter Aggregation

**Use Case**: Distributed counters across multiple agents.

**Implementation**:
```yaml
processors:
  - name: distributed_counter
    type: ottl_transform
    config:
      log_statements:
        - context: log
          statements:
            # Increment global counter
            - set(cache["counter"], EDXRedis(
                {"url": "redis://counters:6379"},
                [
                  {
                    "outField": "total_requests",
                    "command": "incrby",
                    "key": "global:request_count",
                    "args": ["1"]
                  },
                  {
                    "outField": "total_bytes",
                    "command": "incrby",
                    "key": "global:bytes_processed",
                    "args": [attributes["response_size"]]
                  }
                ]
              ))

            # Tag log with current totals
            - set(attributes["global_request_count"], cache["counter"]["total_requests"])
            - set(attributes["global_bytes_count"], cache["counter"]["total_bytes"])
```

**Benefits**:
- Accurate aggregation across agents
- Real-time counters
- Atomic operations prevent race conditions

---

## Error Handling

### Nil Response Handling

Redis returns `nil` for missing keys. EDXRedis handles this gracefully:

**GET Command with Missing Key**:
```yaml
- set(cache["result"], EDXRedis(
    {"url": "redis://localhost:6379"},
    [{"outField": "value", "command": "get", "key": "nonexistent_key"}]
  ))

# Result: cache["result"]["value"] = nil (not an error)
```

**Check for Nil Before Using**:
```yaml
- set(attributes["enriched_field"], cache["result"]["value"]) where cache["result"]["value"] != nil
- set(attributes["enriched_field"], "default_value") where cache["result"]["value"] == nil
```

### Connection Errors

**Error**: `failed to get Redis client: failed to create Redis client: ...`

**Causes**:
- Redis server unreachable
- Network issues
- Invalid URL format
- Port blocked by firewall

**Solutions**:
1. Verify Redis server is running: `redis-cli ping`
2. Check network connectivity: `telnet redis-host 6379`
3. Verify URL format: `redis://host:port` or `rediss://host:port`
4. Check firewall rules

**Graceful Handling**:
```yaml
# Use where clause to skip Redis on error
- set(cache["redis_result"], EDXRedis(...)) where IsMatch(attributes["critical_path"], "true")
```

### Command Errors

**Error**: `failed to execute command get: WRONGTYPE Operation against a key holding the wrong kind of value`

**Cause**: Using wrong command for key type (e.g., GET on a hash)

**Solutions**:
1. Verify key type: `redis-cli TYPE mykey`
2. Use correct command:
   - String: GET, SET
   - Hash: HGET, HSET, HGETALL
   - List: LPUSH, LPOP, LRANGE
   - Set: SADD, SMEMBERS
   - Sorted Set: ZADD, ZRANGE

### Timeout Errors

**Error**: `context deadline exceeded` or `i/o timeout`

**Causes**:
- Redis server slow to respond
- Network latency
- blockingTimeout too low
- Redis under heavy load

**Solutions**:
1. Increase `blockingTimeout`:
   ```yaml
   {"url": "redis://host:6379", "blockingTimeout": 30}
   ```
2. Optimize Redis queries (avoid KEYS *, use SCAN)
3. Scale Redis (add replicas, use Cluster)

### Authentication Errors

**Error**: `NOAUTH Authentication required`

**Cause**: Redis requires auth but none provided

**Solution**:
```yaml
{
  "url": "redis://localhost:6379",
  "auth": {
    "method": "Manual",
    "password": "your-password"
  }
}
```

**Error**: `ERR invalid username-password pair`

**Cause**: Incorrect credentials

**Solutions**:
1. Verify password: `redis-cli -a <password> PING`
2. Check Redis ACL: `redis-cli ACL LIST`
3. Ensure username correct (Redis 6+)

### TLS Errors

**Error**: `x509: certificate relies on legacy Common Name field, use SANs instead`

**Solution**: Regenerate certificates with Subject Alternative Names (see [TLS/mTLS Configuration](#tlsmtls-configuration))

**Error**: `tls: failed to verify certificate: x509: certificate signed by unknown authority`

**Solution**: Provide correct CA certificate:
```yaml
{
  "tls": {
    "enabled": true,
    "validateCerts": true,
    "caCertPath": "/path/to/ca.pem"
  }
}
```

**Error**: `remote error: tls: bad certificate`

**Cause**: mTLS enabled but client cert invalid

**Solution**: Verify client certificate and key:
```bash
openssl x509 -in client.crt -text -noout
openssl rsa -in client.key -check
```

---

## Best Practices

### 1. Use cache["body"] to Avoid Repeated Decoding

**Problem**: Parsing JSON from `body` multiple times is expensive.

**Bad**:
```yaml
- set(attributes["user_id"], ParseJSON(body)["user"]["id"])
- set(attributes["user_name"], ParseJSON(body)["user"]["name"])
- set(attributes["user_email"], ParseJSON(body)["user"]["email"])
# ParseJSON called 3 times!
```

**Good**:
```yaml
- set(cache["parsed_body"], ParseJSON(body))
- set(attributes["user_id"], cache["parsed_body"]["user"]["id"])
- set(attributes["user_name"], cache["parsed_body"]["user"]["name"])
- set(attributes["user_email"], cache["parsed_body"]["user"]["email"])
# ParseJSON called once
```

**Apply to Redis**:
```yaml
- set(cache["redis_result"], EDXRedis(...))
- set(attributes["field1"], cache["redis_result"]["field1"])
- set(attributes["field2"], cache["redis_result"]["field2"])
# Redis call made once, results reused
```

### 2. Set Appropriate TTLs

**Guideline**: Always set expiration on temporary keys.

**Pattern**:
```yaml
[
  {"command": "set", "key": "temp:data", "args": "value"},
  {"command": "expire", "key": "temp:data", "args": ["3600"]}
]
```

**Or use SETEX**:
```yaml
[
  {"command": "setex", "key": "temp:data", "args": ["3600", "value"]}
]
```

**TTL Recommendations**:
- Session data: 1800 seconds (30 minutes)
- Cache data: 300-3600 seconds (5-60 minutes)
- Dedup keys: 3600-86400 seconds (1-24 hours)
- Rate limit buckets: 60-120 seconds (1-2 minutes)

### 3. Use MGET/MSET for Batch Operations

**Bad** (multiple calls):
```yaml
- set(cache["user1"], EDXRedis({"url": "..."}, [{"outField": "name", "command": "get", "key": "user:1"}]))
- set(cache["user2"], EDXRedis({"url": "..."}, [{"outField": "name", "command": "get", "key": "user:2"}]))
- set(cache["user3"], EDXRedis({"url": "..."}, [{"outField": "name", "command": "get", "key": "user:3"}]))
# 3 round trips to Redis
```

**Good** (batch):
```yaml
- set(cache["users"], EDXRedis(
    {"url": "..."},
    [{"outField": "names", "command": "mget", "key": "user:1", "args": ["user:2", "user:3"]}]
  ))
# 1 round trip to Redis
```

### 4. Implement Fallback Logic for Missing Keys

**Pattern**:
```yaml
# Lookup in Redis
- set(cache["lookup"], EDXRedis(...))

# Use Redis value if present
- set(attributes["enriched_field"], cache["lookup"]["value"]) where cache["lookup"]["value"] != nil

# Fallback to default if missing
- set(attributes["enriched_field"], "default") where cache["lookup"]["value"] == nil
```

### 5. Monitor Connection Pool Health

**Check Client Count**:
- Expected: 1-5 clients for typical deployment
- Warning: 10+ clients (review config uniqueness)
- Alert: 50+ clients (likely configuration issue)

**Avoid Config Duplication**:
```yaml
# Good: Same config reuses client
{"url": "redis://host:6379"}
{"url": "redis://host:6379"}

# Bad: Different configs create separate clients
{"url": "redis://host:6379", "blockingTimeout": 10}
{"url": "redis://host:6379", "blockingTimeout": 20}
```

### 6. Use Consistent Field Naming in outField

**Convention**:
- Use snake_case: `user_profile`, `api_response`
- Be descriptive: `cached_user_name` not `val`
- Avoid conflicts: Don't overwrite existing cache keys

**Example**:
```yaml
[
  {"outField": "user_name", "command": "get", "key": "user:name:123"},
  {"outField": "user_email", "command": "get", "key": "user:email:123"},
  {"outField": "user_role", "command": "get", "key": "user:role:123"}
]
```

### 7. Avoid Blocking Commands in Production

**Dangerous Commands**:
- `BLPOP`, `BRPOP` (blocking pops)
- `BRPOPLPUSH` (blocking pop+push)
- `SAVE` (blocks server)
- `KEYS *` (scans all keys, blocks on large datasets)

**Safe Alternatives**:
- Use `LPOP`/`RPOP` (non-blocking)
- Use `BGSAVE` (background save)
- Use `SCAN` (cursor-based iteration)

### 8. Consider Redis Memory Limits

**Monitor Redis Memory**:
```bash
redis-cli INFO memory
```

**Set maxmemory policy** in Redis config:
```
maxmemory 2gb
maxmemory-policy allkeys-lru
```

**Policies**:
- `allkeys-lru`: Evict least recently used keys (good for cache)
- `volatile-lru`: Evict LRU keys with TTL (good for mixed use)
- `allkeys-lfu`: Evict least frequently used (Redis 4.0+)

**EdgeDelta Best Practice**:
- Always set TTLs on temporary keys
- Use eviction policies appropriate for use case
- Monitor Redis memory usage

---

## Troubleshooting

### Certificate Errors (SAN Requirement)

**Error**: `x509: certificate relies on legacy Common Name field, use SANs instead`

**Cause**: Certificate missing Subject Alternative Names (SANs)

**Diagnosis**:
```bash
openssl x509 -in redis.crt -text -noout | grep -A 5 "Subject Alternative Name"
```

If no output, certificate lacks SANs.

**Solution**: Regenerate certificate with SANs (see [TLS/mTLS Configuration](#tlsmtls-configuration))

### Connection Timeouts

**Symptoms**:
- `context deadline exceeded`
- `i/o timeout`
- Slow pipeline processing

**Diagnosis**:
1. Check Redis latency: `redis-cli --latency -h redis-host -p 6379`
2. Check network latency: `ping redis-host`
3. Check Redis load: `redis-cli INFO stats | grep instantaneous_ops_per_sec`

**Solutions**:
1. Increase `blockingTimeout`:
   ```yaml
   {"url": "redis://host:6379", "blockingTimeout": 30}
   ```
2. Reduce Redis load (optimize queries, scale horizontally)
3. Use Redis Cluster for distribution
4. Add Redis replicas for read scaling

### Authentication Failures

**Error**: `NOAUTH Authentication required`

**Solution**: Add auth configuration:
```yaml
{
  "url": "redis://host:6379",
  "auth": {
    "method": "Manual",
    "password": "your-password"
  }
}
```

**Error**: `ERR invalid username-password pair`

**Diagnosis**:
1. Test credentials: `redis-cli -h host -p 6379 -a password PING`
2. Check Redis ACL: `redis-cli ACL LIST`

**Solutions**:
1. Verify password correct
2. For Redis 6+ ACL, include username:
   ```yaml
   "auth": {
     "method": "Manual",
     "username": "edgedelta-user",
     "password": "correct-password"
   }
   ```

### Cluster Node Discovery

**Symptoms**:
- `MOVED` errors
- `CLUSTERDOWN` errors
- Inconsistent results

**Diagnosis**:
1. Check cluster status: `redis-cli --cluster check cluster-node:6379`
2. Verify all nodes reachable

**Solutions**:
1. Include all cluster nodes in URL:
   ```yaml
   "url": "redis://node1:6379,node2:6379,node3:6379,node4:6379,node5:6379,node6:6379"
   ```
2. Ensure network connectivity to all nodes
3. Check cluster topology: `redis-cli CLUSTER NODES`

### Sentinel Failover Delays

**Symptoms**:
- Temporary connection failures
- `READONLY You can't write against a read only replica` errors

**Diagnosis**:
1. Check Sentinel status: `redis-cli -p 26379 SENTINEL masters`
2. Check failover logs: `redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster`

**Solutions**:
1. Increase `blockingTimeout` to tolerate failover delays:
   ```yaml
   "blockingTimeout": 30
   ```
2. Verify Sentinel quorum configured correctly
3. Ensure all Sentinels can reach Redis instances
4. Check Sentinel `down-after-milliseconds` and `failover-timeout` settings

### Memory Issues

**Error**: `OOM command not allowed when used memory > 'maxmemory'`

**Cause**: Redis maxmemory limit reached

**Diagnosis**:
```bash
redis-cli INFO memory | grep maxmemory
redis-cli INFO memory | grep used_memory
```

**Solutions**:
1. Increase Redis maxmemory:
   ```
   redis-cli CONFIG SET maxmemory 4gb
   ```
2. Set eviction policy:
   ```
   redis-cli CONFIG SET maxmemory-policy allkeys-lru
   ```
3. Clean up old keys:
   ```bash
   redis-cli --scan --pattern "temp:*" | xargs redis-cli DEL
   ```
4. Review TTLs on keys (ensure expiration set)

### Performance Degradation

**Symptoms**:
- Slow queries
- High CPU on Redis server
- Pipeline backlog

**Diagnosis**:
1. Check slow queries: `redis-cli SLOWLOG GET 10`
2. Check CPU usage: `redis-cli INFO stats`
3. Check client connections: `redis-cli CLIENT LIST`

**Solutions**:
1. Optimize queries:
   - Avoid `KEYS *` (use `SCAN`)
   - Avoid `HGETALL` on large hashes (use `HSCAN`)
   - Batch operations with `MGET`/`MSET`
2. Scale Redis:
   - Add read replicas
   - Migrate to Cluster
3. Review client pool:
   ```go
   clientManager.GetClientCount()  // Should be low (1-10)
   ```
4. Enable Redis client-side caching (future feature)

---

## Performance Considerations

### Connection Pooling Impact

**Without Pooling** (naive approach):
- 1,000 EDXRedis calls = 1,000 new connections
- Each connection: TCP handshake (3-way) + TLS handshake (4-way) + AUTH
- Overhead: ~10-50ms per connection
- Total overhead: 10-50 seconds

**With Pooling** (EDXRedis implementation):
- 1,000 EDXRedis calls = 1 cached client (10 pooled connections)
- First call: Connection overhead (~10-50ms)
- Subsequent calls: Negligible overhead (<1ms)
- Total overhead: ~10-50ms

**Speedup**: 100x-1000x faster

### Network Latency

**Latency Impact**:
- Local Redis (same host): 0.1-0.5ms
- Same datacenter: 0.5-2ms
- Cross-region: 10-100ms
- Cross-continent: 100-300ms

**Optimization**:
1. Deploy Redis close to EdgeDelta agents
2. Use Redis Cluster for geo-distribution
3. Batch operations with MGET/MSET
4. Cache frequently accessed data

### Command Batching

**Single Command per Call**:
```yaml
# 3 round trips
- set(cache["r1"], EDXRedis({"url": "..."}, [{"command": "get", "key": "k1"}]))
- set(cache["r2"], EDXRedis({"url": "..."}, [{"command": "get", "key": "k2"}]))
- set(cache["r3"], EDXRedis({"url": "..."}, [{"command": "get", "key": "k3"}]))
```

**Batched Commands**:
```yaml
# 1 round trip (sequential execution within call)
- set(cache["results"], EDXRedis(
    {"url": "..."},
    [
      {"outField": "v1", "command": "get", "key": "k1"},
      {"outField": "v2", "command": "get", "key": "k2"},
      {"outField": "v3", "command": "get", "key": "k3"}
    ]
  ))
```

**Best**: Use Redis multi-key commands:
```yaml
# 1 round trip (parallel execution in Redis)
- set(cache["results"], EDXRedis(
    {"url": "..."},
    [{"outField": "values", "command": "mget", "key": "k1", "args": ["k2", "k3"]}]
  ))
```

### Redis Memory Usage

**Memory Per Key** (approximate):
- String (small): 50-100 bytes overhead + value size
- Hash (10 fields): 200-500 bytes overhead + field data
- List (100 items): 500-1000 bytes overhead + item data
- Set (100 members): 500-1000 bytes overhead + member data
- Sorted Set (100 members): 1000-2000 bytes overhead + member data

**Estimate Total Memory**:
```
Total = (Number of keys) × (Avg key size + Avg value size + Overhead)
```

**Example**:
- 1 million dedup keys
- Avg key size: 50 bytes
- Avg value size: 10 bytes (just "1")
- Overhead: 80 bytes
- Total: 1M × (50 + 10 + 80) = 140 MB

### Client-Side Caching

**Current Status**: `clientSideCache` field reserved but not implemented

**Planned Feature**:
- Cache frequently accessed keys on client
- Reduce Redis network round trips
- Invalidation via pub/sub

**Workaround**: Use cache["..."] in OTTL for pipeline-local caching:
```yaml
# First log: Fetch from Redis, store in cache
- set(cache["user_profile"], EDXRedis(...)) where cache["user_profile"] == nil

# Subsequent logs in same batch: Reuse from cache
- set(attributes["enriched"], cache["user_profile"]["name"])
```

### Blocking vs Non-Blocking Commands

**Blocking Commands** (avoid in production):
- `BLPOP`, `BRPOP`: Block until element available or timeout
- `BRPOPLPUSH`: Blocking pop and push
- Impact: Ties up connection, delays other operations

**Non-Blocking Alternatives**:
- `LPOP`, `RPOP`: Return immediately (nil if empty)
- `RPOPLPUSH`: Non-blocking variant

**EdgeDelta Recommendation**: Always use non-blocking commands in pipelines to prevent delays.

---

## Summary

EDXRedis is the most powerful EDX function for stateful processing in EdgeDelta pipelines. Key takeaways:

1. **Client Pooling**: Clients cached and reused based on connection config hash (100x-1000x performance improvement)
2. **Deployment Flexibility**: Standalone, Cluster, and Sentinel support
3. **Security**: Full TLS/mTLS with SAN requirement
4. **Comprehensive Command Support**: All Redis commands available
5. **Production-Ready**: Health checking, thread-safe, automatic recovery

**Critical Requirements**:
- Certificates MUST include Subject Alternative Names (SANs)
- Use `rediss://` for TLS connections
- Set TTLs on temporary keys
- Monitor client pool size
- Implement fallback logic for missing keys

**Common Use Cases**:
- Enrichment (CMDB, geolocation, user profiles)
- Caching (API responses, parsed data)
- Deduplication (exactly-once processing)
- Rate limiting (per-user, per-endpoint)
- Stateful processing (counters, sessions, feature flags)

For additional help, see:
- [OTTL Syntax Guide](../syntax_guide.md)
- [EDX Functions Master Index](../../MASTER_INDEX.md#edx-functions)
- EdgeDelta documentation: https://docs.edgedelta.com
