# Production Example

Production-ready deployment with security hardening, resource limits, and proper operational practices.

## What You Get

- **NATS**: Message broker with resource limits
- **SemStreams**: Hardened container with security best practices
- **Resource Management**: CPU and memory limits for edge devices
- **Health Monitoring**: Proper health checks and restart policies
- **Network Isolation**: Internal network for NATS

## Prerequisites

- Docker & Docker Compose
- Ports 8080 and 9090 available
- **Required**: Create `config.json` before starting
- **Optional**: Set `NATS_PASSWORD` environment variable

## Quick Start

```bash
# 1. Create your config from example
cp config.example.json config.json

# 2. Edit config for your deployment
nano config.json

# 3. Set secrets (optional but recommended)
export NATS_PASSWORD="your-secure-password"

# 4. Start production stack
docker compose up -d

# 5. Verify health
curl http://localhost:8080/health

# 6. Check logs
docker compose logs -f
```

## Configuration

### Minimal Config (config.json)

The example config is **intentionally minimal** - relies on defaults:

```json
{
  "version": "1.0.0",
  "platform": {
    "org": "your-org",           // Your organization
    "id": "edge-device-001",     // Unique device ID
    "type": "edge-deployment",
    "environment": "production"
  },
  "nats": {
    "urls": ["nats://nats:4222"]
  },
  "services": {
    "component-manager": {
      "enabled": true,
      "config": {}                // Uses defaults
    },
    "metrics": {
      "enabled": true,
      "config": {
        "port": 9090              // Only override what you need
      }
    }
  },
  "components": {}                // Add your flow components here
}
```

### What Defaults Are Used

When you omit settings, these defaults apply:

**Platform Defaults:**
- Region: `local`
- Instance ID: Auto-generated from hostname

**Service Defaults:**
- Message Logger: Disabled (reduces I/O)
- Component Manager: Enabled
- Metrics: Enabled on port 9090

**Component Defaults:**
- No components enabled by default
- Add only what you need for your use case

### Adding Components

See `protocol-flow.json` in semstreams repo for full component examples.

Minimal UDP input example:
```json
{
  "components": {
    "udp": {
      "type": "input",
      "name": "udp",
      "enabled": true,
      "config": {
        "ports": {
          "outputs": [{
            "name": "udp_out",
            "subject": "raw.udp",
            "type": "nats"
          }]
        },
        "port": 14550
      }
    }
  }
}
```

## Security Features

### Container Hardening

```yaml
# Read-only filesystem (when possible)
read_only: false  # Set true if no filesystem writes needed

# Drop all capabilities, add only what's needed
cap_drop: [ALL]
cap_add: [NET_BIND_SERVICE]  # Only for ports < 1024

# No privilege escalation
security_opt:
  - no-new-privileges:true
```

### Network Isolation

```yaml
networks:
  internal:
    internal: true  # NATS not exposed to host
  external:
    # Only SemStreams API exposed
```

### Resource Limits

Optimized for edge devices (4GB RAM, 2 CPU):

**NATS:**
- Memory: 512MB-1GB
- CPU: 0.25-0.5 cores
- Storage: 2GB max

**SemStreams:**
- Memory: 512MB-1GB
- CPU: 0.5-1.0 cores

Adjust based on your hardware:
```yaml
deploy:
  resources:
    limits:
      memory: 2G      # Increase for larger workloads
      cpus: '2.0'
```

## Environment Variables

### Required

```bash
SEMSTREAMS_NATS_URL=nats://nats:4222  # Set in compose
```

### Optional (Security)

```bash
# NATS authentication (recommended)
export NATS_PASSWORD="your-secure-password"

# Or use Docker secrets
docker secret create nats_password /path/to/password
```

### Optional (Tuning)

```bash
# Adjust log level
SEMSTREAMS_LOG_LEVEL=warn  # info (default), debug, warn, error

# Disable metrics if not needed
SEMSTREAMS_METRICS_ENABLED=false
```

## Monitoring

### Health Checks

All services have health checks:

```bash
# Check all services
docker compose ps

# Individual health
curl http://localhost:8080/health
```

### Metrics

```bash
# Prometheus metrics
curl http://localhost:9090/metrics

# Check specific metrics
curl http://localhost:9090/metrics | grep semstreams_
```

### Logs

```bash
# All logs
docker compose logs -f

# Specific service
docker compose logs -f semstreams

# Last 50 lines
docker compose logs --tail=50 semstreams
```

## Backup & Recovery

### Persistent Data

```yaml
volumes:
  nats-data:       # NATS JetStream data
  semstreams-data: # Application data
```

### Backup

```bash
# Backup NATS data
docker run --rm \
  -v semstreams-nats-data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/nats-backup.tar.gz /data

# Backup config
cp config.json config-backup-$(date +%Y%m%d).json
```

### Restore

```bash
# Stop services
docker compose down

# Restore NATS data
docker run --rm \
  -v semstreams-nats-data:/data \
  -v $(pwd):/backup \
  alpine sh -c "rm -rf /data/* && tar xzf /backup/nats-backup.tar.gz -C /"

# Restart
docker compose up -d
```

## Updates

### Rolling Update

```bash
# Pull latest images
docker compose pull

# Recreate with new images (zero downtime if flows allow)
docker compose up -d --no-deps --build semstreams
```

### Pinned Versions (Recommended)

```yaml
semstreams:
  image: ghcr.io/c360studio/semstreams:v1.2.3  # Pin specific version
```

## Troubleshooting

**Service won't start?**
```bash
# Check config syntax
docker compose config

# View startup errors
docker compose logs semstreams | head -50
```

**Out of memory?**
```bash
# Check resource usage
docker stats

# Adjust limits in docker-compose.yml
```

**NATS connection issues?**
```bash
# Verify NATS is healthy
docker compose ps nats
docker compose logs nats

# Check network connectivity
docker compose exec semstreams wget -O- http://nats:8222/healthz
```

**Config errors?**
```bash
# Validate JSON
cat config.json | jq .

# Check for required fields
grep -E '"org"|"id"|"nats"' config.json
```

## Production Checklist

Before deploying to production:

- [ ] Config file created and validated
- [ ] NATS password set (if authentication enabled)
- [ ] Resource limits appropriate for hardware
- [ ] Health check URLs accessible
- [ ] Monitoring/alerting configured
- [ ] Backup strategy in place
- [ ] Update strategy documented
- [ ] Logs forwarded to central system (optional)
- [ ] Firewall rules configured
- [ ] SSL/TLS configured (if exposing externally)

## Next Steps

**Add Semantic Search**:
- Combine with `examples/with-embeddings/`
- Add semembed service to this compose file

**Add UI**:
- Combine with `examples/with-ui/`
- Add semstreams-ui service

**Federation**:
- See semstreams docs on multi-node federation
- Configure NATS clustering
