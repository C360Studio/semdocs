# Configuration Management

SemStreams uses a simple, layered configuration approach designed for edge deployments where simplicity and reliability are paramount.

## Core Principle

**If you need Kubernetes to run SemStreams, something has gone architecturally wrong.** SemStreams is designed to run on edge devices with constrained resources using simple container orchestration (Docker Compose).

## Configuration Architecture

### Configuration Layers (Precedence: lowest → highest)

```
1. Code Defaults        → Built-in sensible defaults
2. Config File          → /etc/semstreams/config.json
3. Environment Variables → SEMSTREAMS_* overrides
4. NATS KV Store        → Runtime updates via semstreams-ui
```

### Why This Design?

- **Edge-First**: Minimal dependencies, works offline
- **Security**: Secrets in environment variables, not files
- **Flexibility**: Experts use files, non-experts use UI
- **Reliability**: Always falls back to working defaults

## Config File Location

**Container**: `/etc/semstreams/config.json`

**Non-Container**: Specified via `--config` flag
```bash
./semstreams --config /path/to/config.json
```

## Configuration Format

### Minimal Configuration (Recommended Start)

```json
{
  "port": 8080,
  "nats": {
    "url": "nats://localhost:4222"
  }
}
```

**That's it!** Everything else uses sensible defaults.

### Full Configuration Example

See `configs/deployments/production-full.json` for a complete example with all options documented.

## Environment Variable Overrides

### Pattern: Flat Snake Case

```bash
SEMSTREAMS_<CONFIG_KEY>=value
```

**No nesting notation needed** - environment variables map to top-level config keys only.

### Common Overrides

```bash
# Server configuration
SEMSTREAMS_PORT=8080
SEMSTREAMS_LOG_LEVEL=info

# NATS configuration  
SEMSTREAMS_NATS_URL=nats://nats:4222
SEMSTREAMS_NATS_USER=semstreams
SEMSTREAMS_NATS_PASSWORD=secret-from-vault

# TLS configuration
SEMSTREAMS_TLS_ENABLED=true
SEMSTREAMS_TLS_CERT_FILE=/certs/server.crt
SEMSTREAMS_TLS_KEY_FILE=/certs/server.key

# Feature flags
SEMSTREAMS_METRICS_ENABLED=true
SEMSTREAMS_DEBUG=false
```

### Usage Examples

**Docker:**
```bash
docker run -p 8080:8080 \
  -e SEMSTREAMS_NATS_URL=nats://nats:4222 \
  -e SEMSTREAMS_NATS_PASSWORD=$SECRET \
  ghcr.io/c360/semstreams:latest
```

**Docker Compose:**
```yaml
services:
  semstreams:
    image: ghcr.io/c360/semstreams:latest
    environment:
      SEMSTREAMS_PORT: 8080
      SEMSTREAMS_NATS_URL: nats://nats:4222
      SEMSTREAMS_NATS_PASSWORD: ${NATS_PASSWORD}
```

**Shell:**
```bash
export SEMSTREAMS_LOG_LEVEL=debug
./semstreams --config config.json
```

## Secrets Management

### Best Practices

**❌ NEVER put secrets in config files**
```json
{
  "nats": {
    "password": "my-secret-password"  // DON'T DO THIS
  }
}
```

**✅ ALWAYS use environment variables for secrets**
```bash
export SEMSTREAMS_NATS_PASSWORD=$(cat /run/secrets/nats_password)
docker run -e SEMSTREAMS_NATS_PASSWORD ghcr.io/c360/semstreams:latest
```

### Secret Sources

**Docker Secrets:**
```yaml
services:
  semstreams:
    image: ghcr.io/c360/semstreams:latest
    secrets:
      - nats_password
    environment:
      SEMSTREAMS_NATS_PASSWORD_FILE: /run/secrets/nats_password

secrets:
  nats_password:
    file: ./secrets/nats_password.txt
```

**Vault Integration:**
```bash
export SEMSTREAMS_NATS_PASSWORD=$(vault kv get -field=password secret/nats)
./semstreams
```

**Encrypted Files:**
```bash
export SEMSTREAMS_TLS_KEY_FILE=/encrypted/certs/server.key
export SEMSTREAMS_TLS_KEY_PASSWORD=$(decrypt-tool /path/to/encrypted-password)
./semstreams
```

## NATS KV Configuration Lifecycle

SemStreams stores runtime configuration in NATS KV for dynamic updates.

### Lifecycle Flow

```
1. Startup:
   - Read /etc/semstreams/config.json
   - Apply environment variable overrides
   - Validate configuration
   - Store in NATS KV bucket: CONFIG

2. Runtime:
   - Components read from NATS KV
   - semstreams-ui can update config via NATS KV
   - Components watch for changes (hot-reload)

3. Shutdown:
   - Config persisted in NATS JetStream
   - Survives container restarts
```

### NATS KV Configuration Bucket

**Bucket Name**: `CONFIG`

**Structure**:
```
CONFIG/
├── version              → Current config version
├── platform             → Platform-wide settings
└── components/
    ├── graph-processor  → Component-specific config
    ├── indexmanager
    └── ...
```

### Configuration Precedence During Runtime

1. **Initial Load**: File + Env Vars → NATS KV
2. **After Startup**: NATS KV is source of truth
3. **Updates via UI**: Written to NATS KV, components reload
4. **Restart**: File + Env Vars override NATS KV again

**Important**: Container restart resets config to file + env vars. This is intentional for infrastructure-as-code workflows.

## Configuration Validation

### Validation Stages

1. **Startup Validation**: Config file + env vars validated before storing in NATS KV
2. **Schema Validation**: All fields checked against schema
3. **Dependency Validation**: Required services checked (NATS connectivity)
4. **Runtime Validation**: Updates via UI validated before accepting

### Error Handling

**Fatal Errors** (prevent startup):
- Invalid JSON syntax
- Missing required fields (e.g., NATS URL)
- Invalid field types
- Failed NATS connection

**Warning Errors** (log and continue):
- Unknown fields (forward compatibility)
- Deprecated fields (legacy support)
- Optional service unavailable (graceful degradation)

### Validation Examples

```bash
# Valid config - starts successfully
{
  "port": 8080,
  "nats": {"url": "nats://localhost:4222"}
}

# Invalid config - fails startup
{
  "port": "not-a-number",  # Type error
  "nats": {}               # Missing required field: url
}

# Warning config - starts with warnings
{
  "port": 8080,
  "nats": {"url": "nats://localhost:4222"},
  "unknown_field": "ignored"  # Warning: unknown field
}
```

## Configuration Best Practices

### Development

```json
{
  "port": 8080,
  "log_level": "debug",
  "nats": {
    "url": "nats://localhost:4222"
  },
  "metrics": {
    "enabled": true
  }
}
```

**Characteristics**:
- Debug logging enabled
- Metrics exposed for development tools
- Local NATS instance
- No TLS (faster iteration)

### Production

```json
{
  "port": 8080,
  "log_level": "info",
  "nats": {
    "url": "nats://nats-cluster:4222"
  },
  "tls": {
    "enabled": true,
    "cert_file": "/certs/server.crt",
    "key_file": "/certs/server.key"
  },
  "metrics": {
    "enabled": true,
    "port": 9090
  }
}
```

**Characteristics**:
- Info-level logging
- TLS enabled
- Separate metrics port
- Production NATS cluster
- Secrets via environment variables

### Edge Deployment

```json
{
  "port": 8080,
  "log_level": "warn",
  "nats": {
    "url": "nats://localhost:4222",
    "max_reconnects": -1
  },
  "resources": {
    "max_memory_mb": 512,
    "max_cpu_percent": 50
  }
}
```

**Characteristics**:
- Minimal logging (storage constraints)
- Aggressive reconnection (unreliable networks)
- Resource limits (constrained hardware)
- Local NATS (offline operation)

## Configuration Schema

### Auto-generated Documentation

SemStreams components use struct tags to auto-generate schema documentation:

```go
type Config struct {
    Port     int    `json:"port" default:"8080" schema:"type:int,description:HTTP server port"`
    LogLevel string `json:"log_level" default:"info" schema:"type:enum,values:debug|info|warn|error"`
}
```

### Viewing Schema

```bash
# Get full schema as JSON
curl http://localhost:8080/api/schema

# Get component-specific schema
curl http://localhost:8080/api/schema/graph-processor
```

### Using Schema for Validation

```bash
# Validate config file before deployment
semstreams validate --config production.json

# Validate config against specific version
semstreams validate --config production.json --schema-version v1.0.0
```

## Troubleshooting

### Config Not Loading

**Symptom**: SemStreams uses defaults instead of config file

**Check**:
1. File exists at `/etc/semstreams/config.json`
2. File has correct permissions (readable by semstreams user)
3. JSON syntax is valid: `jq . < config.json`
4. Check logs for validation errors

```bash
# Test config file
docker run --rm -v $(pwd)/config.json:/etc/semstreams/config.json \
  ghcr.io/c360/semstreams:latest validate
```

### Environment Variables Not Working

**Symptom**: Environment variables not overriding config

**Check**:
1. Variable name matches pattern: `SEMSTREAMS_*`
2. Variable is exported: `env | grep SEMSTREAMS`
3. Value format is correct (no quotes for numbers/booleans)
4. Check precedence: NATS KV overrides env vars after startup

```bash
# Debug environment variables
docker run --rm -e SEMSTREAMS_LOG_LEVEL=debug \
  ghcr.io/c360/semstreams:latest env | grep SEMSTREAMS
```

### NATS KV Config Issues

**Symptom**: Config changes not persisting

**Check**:
1. NATS KV bucket created: `nats kv ls`
2. NATS permissions allow KV operations
3. Check NATS logs for KV errors
4. Verify JetStream enabled on NATS server

```bash
# Check NATS KV config bucket
nats kv get CONFIG version
nats kv ls CONFIG
```

### Secret Injection Failures

**Symptom**: Secrets not available to application

**Check**:
1. Secret file exists and is readable
2. Environment variable points to correct path
3. Secret value doesn't contain newlines (trim whitespace)
4. Docker secrets mounted to `/run/secrets/`

```bash
# Debug secret mounting
docker run --rm -v $(pwd)/secrets:/run/secrets:ro \
  alpine ls -la /run/secrets
```

## Migration Guide

### From File-Only Config

**Before** (all in config.json):
```json
{
  "port": 8080,
  "nats": {
    "url": "nats://nats:4222",
    "password": "secret123"
  }
}
```

**After** (secrets in env vars):
```json
{
  "port": 8080,
  "nats": {
    "url": "nats://nats:4222"
  }
}
```

```bash
export SEMSTREAMS_NATS_PASSWORD=secret123
```

### From Environment-Heavy Config

**Before** (everything in env vars):
```bash
export SEMSTREAMS_PORT=8080
export SEMSTREAMS_LOG_LEVEL=info
export SEMSTREAMS_NATS_URL=nats://nats:4222
export SEMSTREAMS_NATS_USER=semstreams
export SEMSTREAMS_NATS_PASSWORD=secret
# ... 50 more variables
```

**After** (structure in file):
```json
{
  "port": 8080,
  "log_level": "info",
  "nats": {
    "url": "nats://nats:4222",
    "user": "semstreams"
  }
}
```

```bash
# Only secrets in env vars
export SEMSTREAMS_NATS_PASSWORD=secret
```

## See Also

- [Production Deployment Guide](PRODUCTION.md) - Complete production setup
- [Operations Runbook](OPERATIONS.md) - Day-to-day operations
- [Example Configs](../../configs/deployments/) - Reference configurations
