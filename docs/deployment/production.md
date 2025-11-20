# Production Deployment Guide

SemStreams is designed for **edge deployments** on constrained hardware. This guide covers production deployment patterns that maintain this core principle.

## Core Principle

**If you need Kubernetes to run SemStreams, something has gone architecturally wrong.**

SemStreams is built for:
- **Edge devices** with limited CPU/memory
- **Offline-first** operation with intermittent connectivity
- **Simple operations** by non-expert users
- **Minimal dependencies** - no orchestration platform required

## Architecture Overview

### Single-Node Deployment (Recommended)

Most SemStreams deployments run on a **single edge device**:

```
┌─────────────────────────────────────┐
│      Edge Device (Single Node)     │
│                                     │
│  ┌──────────┐      ┌──────────┐   │
│  │ NATS     │◄────►│SemStreams│   │
│  │ Server   │      │ Container│   │
│  └──────────┘      └──────────┘   │
│                                     │
│  ┌──────────┐      ┌──────────┐   │
│  │ Optional │      │ Optional │   │
│  │ Services │      │ UI       │   │
│  └──────────┘      └──────────┘   │
└─────────────────────────────────────┘
```

**Why Single-Node?**
- Edge devices process local data streams
- Each instance is completely independent
- Simpler operations and troubleshooting
- Lower resource overhead

### Multi-Location Deployment

For multiple edge locations, each runs an **independent SemStreams instance**. Instances communicate using component outputs and inputs (WebSocket, HTTP, NATS pub/sub):

```
┌──────────────────────┐       ┌──────────────────────┐       ┌──────────────────────┐
│    Location A        │       │    Location B        │       │    Location C        │
│                      │       │                      │       │                      │
│  ┌────────────────┐  │       │  ┌────────────────┐  │       │  ┌────────────────┐  │
│  │  SemStreams    │  │       │  │  SemStreams    │  │       │  │  SemStreams    │  │
│  │  + NATS        │  │       │  │  + NATS        │  │       │  │  + NATS        │  │
│  └────────────────┘  │       │  └────────────────┘  │       │  └────────────────┘  │
│                      │       │                      │       │                      │
│  Flow:               │       │  Flow:               │       │  Flow:               │
│  Input → Process →  ─┼──────►│─ Input → Process →  ─┼──────►│─ Input → Sink       │
│         WebSocketOut │       │        WebSocketOut  │       │                      │
└──────────────────────┘       └──────────────────────┘       └──────────────────────┘
```

**Communication Pattern**:
- **Location A** has a flow with `WebSocketOutput` component listening on port 9000
- **Location B** has a flow with `WebSocketInput` component connecting to `ws://location-a:9000`
- Each instance operates completely independently with its own NATS server
- No NATS networking/federation - just component-based integration

**Example Flow Configuration** (Location A outputs):
```json
{
  "components": [
    {
      "id": "sensor-input",
      "type": "mavlink_input"
    },
    {
      "id": "send-to-location-b",
      "type": "websocket_output",
      "config": {
        "port": 9000
      }
    }
  ]
}
```

**Example Flow Configuration** (Location B inputs):
```json
{
  "components": [
    {
      "id": "receive-from-location-a",
      "type": "websocket_input",
      "config": {
        "url": "ws://location-a:9000"
      }
    },
    {
      "id": "process-data",
      "type": "json_filter"
    }
  ]
}
```

**Benefits**:
- Each location operates completely independently
- Simple component-based integration (no NATS networking complexity)
- Works with intermittent connectivity (buffering in output components)
- Advanced users can use NATS networking if needed, but SemStreams doesn't require or support it directly

## Why Not Kubernetes?

### Kubernetes Adds Complexity You Don't Need

**Kubernetes is for**:
- Large-scale multi-tenant workloads
- Complex service mesh architectures
- Frequent deployments with rolling updates
- Heterogeneous workload orchestration

**SemStreams is**:
- Single-tenant edge processing
- Simple container deployment
- Infrequent updates (stable edge software)
- Homogeneous workload (one app per device)

### Resource Overhead Comparison

**Docker Compose** (SemStreams approach):
- ~50MB memory overhead
- ~0.1 CPU cores for container runtime
- Zero control plane resources

**Kubernetes** (wrong approach):
- ~500MB+ memory overhead (kubelet, kube-proxy, etc.)
- ~0.5+ CPU cores for control plane
- etcd storage requirements
- API server resource consumption

On a device with **2GB RAM** and **2 CPU cores**, Kubernetes consumes **25% of your resources before running any application**.

### Operational Complexity

**Docker Compose**:
```bash
docker compose up -d           # Start
docker compose logs -f         # Monitor
docker compose down            # Stop
```

**Kubernetes** (don't do this):
```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl get pods --watch
kubectl describe pod semstreams-xyz
kubectl logs -f semstreams-xyz
# ... and so on
```

**Question**: Who maintains this Kubernetes cluster on an edge device? What happens when etcd corrupts? How do you debug kubelet issues remotely?

**Answer**: Use Docker Compose. It just works.

## Single-Node Production Deployment

### Hardware Requirements

**Minimum Edge Device**:
- **CPU**: 2 cores (ARM or x86_64)
- **RAM**: 2GB
- **Storage**: 8GB (4GB for containers, 4GB for data)
- **Network**: Intermittent connectivity acceptable

**Recommended Edge Device**:
- **CPU**: 4 cores
- **RAM**: 4GB
- **Storage**: 32GB (SSD preferred for NATS JetStream)
- **Network**: Reliable but not required

**Capacity Planning**:
- Each active flow: ~50-100MB RAM
- NATS JetStream: ~200-500MB RAM
- SemStreams base: ~100-200MB RAM
- Optional services: +500MB-2GB (if enabled)

### Container Setup

**1. Create deployment directory:**

```bash
mkdir -p /opt/semstreams
cd /opt/semstreams
```

**2. Create `docker-compose.yml`:**

```yaml
services:
  nats:
    image: nats:latest
    container_name: semstreams-nats
    ports:
      - "4222:4222"    # Client connections
      - "8222:8222"    # HTTP monitoring
    command:
      - "--jetstream"
      - "--store_dir=/data"
      - "--max_mem_store=512M"
      - "--max_file_store=2G"
    volumes:
      - nats-data:/data
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:8222/healthz"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  semstreams:
    image: ghcr.io/c360/semstreams:latest
    container_name: semstreams
    ports:
      - "8080:8080"
    environment:
      # Configuration overrides
      SEMSTREAMS_NATS_URL: nats://nats:4222
      SEMSTREAMS_LOG_LEVEL: info
      SEMSTREAMS_METRICS_ENABLED: true

      # Secrets (use Docker secrets or vault in production)
      SEMSTREAMS_NATS_PASSWORD: ${NATS_PASSWORD}
    volumes:
      - ./config.json:/etc/semstreams/config.json:ro
      - semstreams-data:/data
    depends_on:
      nats:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:8080/health"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    restart: unless-stopped
    # Resource limits for edge devices
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: '1.0'
        reservations:
          memory: 512M
          cpus: '0.5'

volumes:
  nats-data:
    driver: local
  semstreams-data:
    driver: local

networks:
  default:
    driver: bridge
```

**3. Create minimal `config.json`:**

```json
{
  "port": 8080,
  "log_level": "info",
  "nats": {
    "url": "nats://nats:4222"
  },
  "metrics": {
    "enabled": true,
    "port": 8080,
    "path": "/metrics"
  }
}
```

**4. Store secrets securely:**

```bash
# Option A: Environment file (development)
echo "NATS_PASSWORD=your-secure-password" > .env
chmod 600 .env

# Option B: Docker secrets (production)
echo "your-secure-password" | docker secret create nats_password -

# Option C: External secrets manager (enterprise)
export NATS_PASSWORD=$(vault kv get -field=password secret/semstreams/nats)
```

**5. Start services:**

```bash
docker compose up -d
```

**6. Verify deployment:**

```bash
# Check container status
docker compose ps

# Check SemStreams health
curl http://localhost:8080/health

# Check NATS connectivity
curl http://localhost:8222/healthz

# View logs
docker compose logs -f semstreams
```

## Resource Management

### Memory Limits

Set appropriate limits based on device capacity:

```yaml
services:
  semstreams:
    deploy:
      resources:
        limits:
          memory: 1G      # Hard limit
        reservations:
          memory: 512M    # Soft reservation
```

**Memory Budget Example** (4GB device):
- OS and system: 1GB
- NATS JetStream: 500MB
- SemStreams: 1GB
- Optional services: 1GB
- Buffer: 500MB

### CPU Limits

```yaml
services:
  semstreams:
    deploy:
      resources:
        limits:
          cpus: '1.0'     # Max 1 CPU core
        reservations:
          cpus: '0.5'     # Guarantee 0.5 cores
```

### Storage Management

**NATS JetStream Storage**:
```bash
# Limit JetStream storage
command:
  - "--max_file_store=2G"
  - "--max_mem_store=512M"
```

**Volume Cleanup**:
```bash
# Set up automatic cleanup in Docker
docker system prune --volumes --force --filter "until=720h"  # 30 days
```

## Network Architecture

### Port Requirements

**Required Ports**:
- `4222`: NATS client connections (internal only)
- `8080`: SemStreams HTTP API (internal or exposed)

**Optional Ports**:
- `8222`: NATS monitoring (internal only)
- `9090`: Prometheus metrics (if observability enabled)
- `3000`: Grafana UI (if observability enabled)
- `9000+`: WebSocket/HTTP outputs (if connecting instances)

**Firewall Rules**:
```bash
# Allow SemStreams HTTP API
iptables -A INPUT -p tcp --dport 8080 -j ACCEPT

# Allow WebSocket/HTTP outputs if connecting instances
iptables -A INPUT -p tcp --dport 9000 -j ACCEPT  # Adjust port as needed

# Block all other container ports from external access
iptables -A INPUT -p tcp --dport 4222 -j DROP
iptables -A INPUT -p tcp --dport 8222 -j DROP
```

### TLS Configuration

**Generate Certificates**:
```bash
# Self-signed for internal use
openssl req -x509 -newkey rsa:4096 -nodes \
  -keyout server.key -out server.crt -days 365 \
  -subj "/CN=semstreams.local"
```

**Mount in Docker Compose**:
```yaml
services:
  semstreams:
    environment:
      SEMSTREAMS_TLS_ENABLED: true
      SEMSTREAMS_TLS_CERT_FILE: /certs/server.crt
      SEMSTREAMS_TLS_KEY_FILE: /certs/server.key
    volumes:
      - ./certs:/certs:ro
```

**Automated Certificate Management** (Optional):

See [OPTIONAL_SERVICES.md](../OPTIONAL_SERVICES.md) for step-ca setup.

## Monitoring and Observability

### Health Checks

**SemStreams Health Endpoint**:
```bash
curl http://localhost:8080/health

# Expected response:
{
  "status": "healthy",
  "version": "v1.0.0",
  "uptime": "24h30m15s",
  "components": {
    "nats": "connected",
    "flows": "running"
  }
}
```

**NATS Health Check**:
```bash
curl http://localhost:8222/healthz

# Expected: 200 OK
```

### Metrics Collection

**Enable Prometheus Metrics**:
```json
{
  "metrics": {
    "enabled": true,
    "port": 8080,
    "path": "/metrics"
  }
}
```

**Access Metrics**:
```bash
curl http://localhost:8080/metrics
```

**Optional Observability Stack**:

If you want Prometheus + Grafana dashboards:
```bash
task services:start:observability

# Access dashboards at http://localhost:3000 (admin/admin)
```

See [OPTIONAL_SERVICES.md](../OPTIONAL_SERVICES.md) for details.

### Log Management

**View Logs**:
```bash
# Follow all logs
docker compose logs -f

# Follow SemStreams only
docker compose logs -f semstreams

# Last 100 lines
docker compose logs --tail=100 semstreams
```

**Log Levels**:
```bash
# Development: debug
SEMSTREAMS_LOG_LEVEL=debug

# Production: info
SEMSTREAMS_LOG_LEVEL=info

# Edge (constrained storage): warn
SEMSTREAMS_LOG_LEVEL=warn
```

**Log Rotation** (Prevent disk fill):
```yaml
services:
  semstreams:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

## Backup and Recovery

### What to Backup

**Critical Data**:
- NATS JetStream data: `/var/lib/docker/volumes/semstreams_nats-data/_data`
- Configuration: `/opt/semstreams/config.json`
- Secrets: External secret manager (never in backups)

**Non-Critical** (can be rebuilt):
- Container images (pull from registry)
- Application logs (ephemeral)

### Backup Strategy

**Simple File Backup**:
```bash
#!/bin/bash
# /opt/semstreams/backup.sh

BACKUP_DIR="/backup/semstreams/$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"

# Stop containers to ensure consistent backup
docker compose -f /opt/semstreams/docker-compose.yml stop

# Backup NATS data
tar -czf "$BACKUP_DIR/nats-data.tar.gz" \
  /var/lib/docker/volumes/semstreams_nats-data/_data

# Backup config
cp /opt/semstreams/config.json "$BACKUP_DIR/"

# Restart containers
docker compose -f /opt/semstreams/docker-compose.yml start

# Keep only last 7 days
find /backup/semstreams -type d -mtime +7 -exec rm -rf {} \;
```

**Automated Backup with Cron**:
```bash
# Run backup daily at 2 AM
0 2 * * * /opt/semstreams/backup.sh
```

### Recovery Procedure

**1. Stop running containers:**
```bash
cd /opt/semstreams
docker compose down
```

**2. Restore data:**
```bash
BACKUP_DATE="20250110-020000"
tar -xzf /backup/semstreams/$BACKUP_DATE/nats-data.tar.gz \
  -C /var/lib/docker/volumes/semstreams_nats-data/_data
```

**3. Restore configuration:**
```bash
cp /backup/semstreams/$BACKUP_DATE/config.json /opt/semstreams/
```

**4. Start containers:**
```bash
docker compose up -d
```

**5. Verify:**
```bash
curl http://localhost:8080/health
docker compose logs -f semstreams
```

## Updates and Maintenance

### Updating SemStreams

**1. Pull new image:**
```bash
docker compose pull semstreams
```

**2. Stop old container:**
```bash
docker compose stop semstreams
```

**3. Start new container:**
```bash
docker compose up -d semstreams
```

**4. Verify:**
```bash
curl http://localhost:8080/health
docker compose logs -f semstreams
```

**Zero-Downtime Update** (if you need it):
```bash
# Use Docker Swarm mode (not recommended for edge, but available)
docker stack deploy -c docker-compose.yml semstreams
docker service update --image ghcr.io/c360/semstreams:v1.1.0 semstreams_app
```

### Version Pinning

**Development** (track latest):
```yaml
services:
  semstreams:
    image: ghcr.io/c360/semstreams:latest
```

**Production** (pin versions):
```yaml
services:
  semstreams:
    image: ghcr.io/c360/semstreams:v1.0.0  # Explicit version
```

## Security Best Practices

### Secret Management

**Never do this:**
```yaml
environment:
  SEMSTREAMS_NATS_PASSWORD: "my-secret-password"  # ❌ NO!
```

**Always do this:**
```yaml
environment:
  SEMSTREAMS_NATS_PASSWORD: ${NATS_PASSWORD}  # From env file or vault
```

### Container Security

**Run as non-root:**
```yaml
services:
  semstreams:
    user: "1000:1000"  # Non-root user
```

**Read-only root filesystem:**
```yaml
services:
  semstreams:
    read_only: true
    tmpfs:
      - /tmp
      - /var/tmp
```

**Drop capabilities:**
```yaml
services:
  semstreams:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE  # Only if binding to port < 1024
```

### Network Security

**Use internal networks:**
```yaml
networks:
  internal:
    driver: bridge
    internal: true  # No external access

services:
  nats:
    networks:
      - internal  # NATS not exposed
  semstreams:
    networks:
      - internal
      - default   # Only SemStreams exposed
```

## Troubleshooting

### Container Won't Start

**Check logs:**
```bash
docker compose logs semstreams
```

**Common issues:**
- Configuration file syntax errors
- NATS connection failure
- Port already in use
- Insufficient memory

**Solutions:**
```bash
# Validate config
cat config.json | jq .

# Check port availability
lsof -i :8080

# Check NATS connectivity
docker compose logs nats
curl http://localhost:8222/healthz

# Check resource usage
docker stats
```

### High Memory Usage

**Check current usage:**
```bash
docker stats semstreams
```

**Identify cause:**
```bash
# Enable pprof profiling
curl http://localhost:8080/debug/pprof/heap > heap.prof
go tool pprof heap.prof
```

**Mitigation:**
- Reduce `max_mem_store` for NATS
- Lower memory limits in docker-compose.yml
- Check for memory leaks in flows

### NATS Connection Issues

**Test NATS connectivity:**
```bash
# From host
curl http://localhost:8222/varz

# From container
docker exec semstreams wget -O- http://nats:8222/varz
```

**Common causes:**
- NATS not healthy (check healthcheck)
- Wrong NATS URL in config
- Network connectivity issues
- NATS out of memory

### Performance Issues

**Check resource limits:**
```bash
docker stats
```

**Check NATS performance:**
```bash
curl http://localhost:8222/varz | jq '.cpu, .mem, .connections'
```

**Check SemStreams metrics:**
```bash
curl http://localhost:8080/metrics | grep -E 'semstreams_(flow|message)_'
```

## Migration from Other Deployments

### From Bare Metal to Containers

**1. Export configuration:**
```bash
# Copy existing config
cp /etc/semstreams/config.json /opt/semstreams/
```

**2. Export NATS data:**
```bash
# Stop NATS
systemctl stop nats

# Copy JetStream data
cp -r /var/lib/nats /opt/semstreams/nats-data/
```

**3. Start containers:**
```bash
cd /opt/semstreams
docker compose up -d
```

**4. Verify and decommission old:**
```bash
# Test new deployment
curl http://localhost:8080/health

# Stop old services
systemctl stop semstreams nats
systemctl disable semstreams nats
```

### From Docker Compose v1 to v2

Docker Compose v2 is built into Docker CLI:

```bash
# Old way
docker-compose up -d

# New way (automatically used)
docker compose up -d
```

No migration needed - v2 is backward compatible.

## Production Checklist

Before deploying to production:

- [ ] **Configuration**
  - [ ] Config file validated: `cat config.json | jq .`
  - [ ] Secrets in environment variables, not in config file
  - [ ] Log level set to `info` or `warn`
  - [ ] Metrics enabled for monitoring

- [ ] **Resources**
  - [ ] Memory limits configured based on device capacity
  - [ ] CPU limits configured to prevent resource starvation
  - [ ] Storage limits set for NATS JetStream
  - [ ] Log rotation configured to prevent disk fill

- [ ] **Security**
  - [ ] TLS enabled for external exposure
  - [ ] Firewall rules configured
  - [ ] Containers running as non-root
  - [ ] Internal network for NATS
  - [ ] Secrets stored in external vault or Docker secrets

- [ ] **Monitoring**
  - [ ] Health checks configured and tested
  - [ ] Metrics endpoint accessible
  - [ ] Log aggregation configured (if needed)
  - [ ] Alerting configured for critical failures

- [ ] **Backup**
  - [ ] Backup script created and tested
  - [ ] Backup schedule configured (cron)
  - [ ] Recovery procedure documented and tested
  - [ ] Backup retention policy defined

- [ ] **Operations**
  - [ ] Update procedure documented
  - [ ] Rollback procedure tested
  - [ ] On-call runbook created
  - [ ] Troubleshooting guide accessible

## See Also

- **Quick Start**: [Production Example](../../examples/production/) - Ready-to-run Docker Compose setup
- [Configuration Management](configuration.md) - Config file format and environment variables
- [Operations Runbook](operations.md) - Day-to-day operational procedures
- [Optional Services](optional-services.md) - Embedding, TLS, and observability services
- [TLS Setup Guide](tls-setup.md) - Certificate management
- [Federation Guide](federation-advanced.md) - Multi-location deployments with mTLS
