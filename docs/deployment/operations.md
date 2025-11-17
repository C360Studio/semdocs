# Operations Runbook

This guide covers day-to-day operations for running SemStreams in production.

## Daily Operations

### Health Monitoring

**Check System Health:**
```bash
# SemStreams health endpoint
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

**Check NATS Health:**
```bash
# NATS monitoring endpoint
curl http://localhost:8222/varz | jq '.connections, .in_msgs, .out_msgs'

# Check JetStream status
curl http://localhost:8222/jsz | jq '.streams, .messages, .bytes'
```

**Check Container Status:**
```bash
# View running containers
docker compose ps

# Check resource usage
docker stats --no-stream

# Expected output:
CONTAINER           CPU %     MEM USAGE / LIMIT     MEM %     NET I/O
semstreams          15.5%     450MiB / 1GiB        44.0%     1.2MB / 2.5MB
semstreams-nats     5.2%      350MiB / 1GiB        34.2%     2.5MB / 1.2MB
```

### Log Management

**View Logs:**
```bash
# Follow all logs
docker compose logs -f

# Follow SemStreams only
docker compose logs -f semstreams

# Last 100 lines
docker compose logs --tail=100 semstreams

# Since specific time
docker compose logs --since 1h semstreams

# Search logs for errors
docker compose logs semstreams | grep -i error
```

**Log Rotation:**
```bash
# Check log sizes
docker compose logs semstreams | wc -l

# Force log rotation (if configured)
docker compose restart semstreams
```

### Metrics Collection

**Prometheus Metrics:**
```bash
# Get all metrics
curl http://localhost:8080/metrics

# Filter specific metrics
curl http://localhost:8080/metrics | grep semstreams_flow

# Check message throughput
curl http://localhost:8080/metrics | grep semstreams_messages_total
```

**Key Metrics to Monitor:**
- `semstreams_messages_total` - Total messages processed
- `semstreams_messages_errors_total` - Message processing errors
- `semstreams_flow_active` - Number of active flows
- `semstreams_nats_connection_status` - NATS connection health
- `process_resident_memory_bytes` - Memory usage

## Starting and Stopping

### Normal Startup

```bash
cd /opt/semstreams

# Start all services
docker compose up -d

# Wait for services to be ready
sleep 10

# Verify startup
curl http://localhost:8080/health
docker compose logs --tail=50 semstreams
```

### Normal Shutdown

```bash
cd /opt/semstreams

# Graceful shutdown (30 second timeout)
docker compose down --timeout 30

# Verify shutdown
docker compose ps
```

### Emergency Shutdown

```bash
# Immediate shutdown (no graceful drain)
docker compose kill

# Remove containers
docker compose down

# Verify all stopped
docker ps | grep semstreams
```

### Restart

```bash
# Restart all services
docker compose restart

# Restart specific service
docker compose restart semstreams

# Restart with new image
docker compose pull
docker compose up -d
```

## Configuration Management

### Viewing Active Configuration

```bash
# View mounted config file
docker exec semstreams cat /etc/semstreams/config.json

# View environment variables
docker exec semstreams env | grep SEMSTREAMS

# View effective configuration (from NATS KV)
curl http://localhost:8080/api/config
```

### Updating Configuration

**Static Config (Requires Restart):**
```bash
# 1. Edit config file
vim /opt/semstreams/config.json

# 2. Validate JSON
jq . < /opt/semstreams/config.json

# 3. Restart to apply
docker compose restart semstreams

# 4. Verify changes
curl http://localhost:8080/api/config
```

**Environment Variables (Requires Restart):**
```bash
# 1. Update .env file
echo "SEMSTREAMS_LOG_LEVEL=debug" >> /opt/semstreams/.env

# 2. Recreate containers with new env vars
docker compose up -d

# 3. Verify
docker exec semstreams env | grep SEMSTREAMS_LOG_LEVEL
```

**Runtime Config (Via UI - No Restart):**
```bash
# If semstreams-ui is running, configuration updates via UI
# automatically update NATS KV and components hot-reload

# View current NATS KV config
# (requires nats CLI)
nats kv get CONFIG version
```

### Configuration Rollback

```bash
# 1. Restore previous config from backup
cp /backup/semstreams/20250110-020000/config.json /opt/semstreams/

# 2. Restart services
docker compose restart semstreams

# 3. Verify rollback
docker compose logs -f semstreams
curl http://localhost:8080/health
```

## Backup and Recovery

### Manual Backup

```bash
#!/bin/bash
# /opt/semstreams/backup.sh

BACKUP_DIR="/backup/semstreams/$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"

echo "[BACKUP] Stopping containers for consistent backup..."
docker compose -f /opt/semstreams/docker-compose.yml stop

echo "[BACKUP] Backing up NATS JetStream data..."
tar -czf "$BACKUP_DIR/nats-data.tar.gz" \
  /var/lib/docker/volumes/semstreams_nats-data/_data

echo "[BACKUP] Backing up configuration..."
cp /opt/semstreams/config.json "$BACKUP_DIR/"
cp /opt/semstreams/.env "$BACKUP_DIR/" 2>/dev/null || true

echo "[BACKUP] Restarting containers..."
docker compose -f /opt/semstreams/docker-compose.yml start

echo "[BACKUP] Cleaning old backups (keeping last 7 days)..."
find /backup/semstreams -type d -mtime +7 -exec rm -rf {} \; 2>/dev/null || true

echo "[BACKUP] Backup complete: $BACKUP_DIR"
```

**Run Backup:**
```bash
chmod +x /opt/semstreams/backup.sh
/opt/semstreams/backup.sh
```

### Automated Backup (Cron)

```bash
# Edit crontab
crontab -e

# Add daily backup at 2 AM
0 2 * * * /opt/semstreams/backup.sh >> /var/log/semstreams-backup.log 2>&1
```

### Recovery from Backup

```bash
#!/bin/bash
# /opt/semstreams/restore.sh

BACKUP_DATE="${1:-latest}"

if [ "$BACKUP_DATE" = "latest" ]; then
  BACKUP_DIR=$(ls -td /backup/semstreams/* | head -1)
else
  BACKUP_DIR="/backup/semstreams/$BACKUP_DATE"
fi

if [ ! -d "$BACKUP_DIR" ]; then
  echo "[ERROR] Backup not found: $BACKUP_DIR"
  exit 1
fi

echo "[RESTORE] Using backup: $BACKUP_DIR"
echo "[RESTORE] Stopping containers..."
cd /opt/semstreams
docker compose down

echo "[RESTORE] Removing old data..."
docker volume rm semstreams_nats-data 2>/dev/null || true

echo "[RESTORE] Creating new volume..."
docker volume create semstreams_nats-data

echo "[RESTORE] Extracting backup..."
tar -xzf "$BACKUP_DIR/nats-data.tar.gz" \
  -C /var/lib/docker/volumes/semstreams_nats-data/_data \
  --strip-components=1

echo "[RESTORE] Restoring configuration..."
cp "$BACKUP_DIR/config.json" /opt/semstreams/
[ -f "$BACKUP_DIR/.env" ] && cp "$BACKUP_DIR/.env" /opt/semstreams/

echo "[RESTORE] Starting containers..."
docker compose up -d

echo "[RESTORE] Waiting for services..."
sleep 10

echo "[RESTORE] Verifying health..."
curl http://localhost:8080/health

echo "[RESTORE] Recovery complete!"
```

**Run Recovery:**
```bash
chmod +x /opt/semstreams/restore.sh

# Restore latest backup
./restore.sh

# Restore specific backup
./restore.sh 20250110-020000
```

## Updates and Upgrades

### Update SemStreams Container

**Check Current Version:**
```bash
docker exec semstreams cat /etc/semstreams/VERSION
# or
curl http://localhost:8080/health | jq '.version'
```

**Standard Update:**
```bash
cd /opt/semstreams

# 1. Backup before updating
./backup.sh

# 2. Pull new image
docker compose pull semstreams

# 3. Check what will change
docker compose config | grep image

# 4. Stop old container
docker compose stop semstreams

# 5. Start new container
docker compose up -d semstreams

# 6. Verify new version
docker compose logs -f semstreams
curl http://localhost:8080/health

# 7. If issues, rollback
# docker compose down
# docker image tag ghcr.io/c360/semstreams:v1.0.0 ghcr.io/c360/semstreams:latest
# docker compose up -d
```

**Pinned Version Update:**
```bash
# Edit docker-compose.yml to change version
vim /opt/semstreams/docker-compose.yml

# Change:
# image: ghcr.io/c360/semstreams:v1.0.0
# To:
# image: ghcr.io/c360/semstreams:v1.1.0

# Apply update
docker compose pull
docker compose up -d

# Verify
docker compose ps
```

### Update NATS Server

```bash
cd /opt/semstreams

# 1. Backup (NATS data is critical!)
./backup.sh

# 2. Pull new NATS image
docker compose pull nats

# 3. Restart NATS
docker compose stop nats
docker compose up -d nats

# 4. Verify NATS health
curl http://localhost:8222/healthz

# 5. Verify SemStreams reconnected
docker compose logs semstreams | grep -i nats
```

## Troubleshooting

### Container Won't Start

**Symptoms:**
- `docker compose up -d` fails
- Container exits immediately
- Health check failing

**Diagnosis:**
```bash
# Check logs
docker compose logs semstreams

# Check container exit code
docker compose ps -a

# Check configuration
jq . < /opt/semstreams/config.json

# Check port conflicts
lsof -i :8080
lsof -i :4222
```

**Common Fixes:**
```bash
# Invalid config
vim /opt/semstreams/config.json
jq . < /opt/semstreams/config.json  # Validate

# Port conflict
# Change port in config or docker-compose.yml

# Missing NATS
docker compose up -d nats
sleep 5
docker compose up -d semstreams

# Out of memory
docker stats
# Increase memory limits in docker-compose.yml
```

### High Memory Usage

**Symptoms:**
- Container using more than configured limit
- OOM kills
- System becoming unresponsive

**Diagnosis:**
```bash
# Check memory usage
docker stats --no-stream

# Check NATS JetStream usage
curl http://localhost:8222/jsz | jq '.memory, .store'

# Check for memory leaks
docker exec semstreams pprof -text /proc/1/cmdline heap
```

**Fixes:**
```bash
# Reduce NATS memory limits
# Edit docker-compose.yml or config:
# --max_mem_store=256M (reduce from 512M)

# Clear old JetStream data
docker exec semstreams-nats nats stream purge <stream-name> --force

# Restart to clear memory
docker compose restart semstreams
```

### NATS Connection Issues

**Symptoms:**
- `nats: connection refused`
- `nats: timeout`
- Health check showing `"nats": "disconnected"`

**Diagnosis:**
```bash
# Check NATS is running
docker compose ps nats

# Check NATS health
curl http://localhost:8222/healthz

# Check NATS connections
curl http://localhost:8222/connz | jq '.connections'

# Test from SemStreams container
docker exec semstreams wget -O- http://nats:8222/healthz
```

**Fixes:**
```bash
# NATS not running
docker compose up -d nats

# NATS crashed
docker compose restart nats

# Wrong NATS URL in config
# Check config: nats://nats:4222 (inside container)
# Not: nats://localhost:4222

# Network issue
docker network ls
docker network inspect semstreams_default
```

### Performance Issues

**Symptoms:**
- High latency
- Slow message processing
- CPU at 100%

**Diagnosis:**
```bash
# Check CPU usage
docker stats

# Check message backlog
curl http://localhost:8080/metrics | grep semstreams_messages_pending

# Check NATS performance
curl http://localhost:8222/varz | jq '.cpu, .mem, .slow_consumers'

# Check for slow flows
curl http://localhost:8080/api/flows | jq '.[] | select(.status == "slow")'
```

**Fixes:**
```bash
# Increase CPU limits
# Edit docker-compose.yml:
#   cpus: '2.0'  # Increase from 1.0

# Reduce message rate at source
# Check input component configuration

# Scale horizontally (multiple instances)
# Deploy additional instances at other locations

# Check for inefficient flows
# Review flow configuration for optimization
```

### Disk Space Issues

**Symptoms:**
- `no space left on device`
- NATS JetStream errors
- Backup failures

**Diagnosis:**
```bash
# Check disk usage
df -h

# Check Docker volumes
docker system df

# Check NATS data size
du -sh /var/lib/docker/volumes/semstreams_nats-data

# Check logs size
docker compose logs semstreams | wc -c
```

**Fixes:**
```bash
# Clean old Docker resources
docker system prune -a --volumes -f

# Reduce JetStream retention
# Edit NATS config:
# --max_file_store=1G  # Reduce from 2G

# Enable log rotation
# Edit docker-compose.yml:
# logging:
#   driver: "json-file"
#   options:
#     max-size: "10m"
#     max-file: "3"

# Clean old backups
find /backup/semstreams -mtime +7 -exec rm -rf {} \;
```

## Monitoring and Alerts

### Key Health Indicators

**System Health:**
- Container status (should be "Up" and "healthy")
- Memory usage (should be < 80% of limit)
- CPU usage (should be < 80% sustained)
- Disk usage (should be < 80%)

**SemStreams Health:**
- Health endpoint returns 200 OK
- NATS connection status: "connected"
- Active flows count matches expected
- No error spikes in logs

**NATS Health:**
- JetStream memory < max_mem_store
- JetStream storage < max_file_store
- No slow consumers
- Connection count stable

### Simple Health Check Script

```bash
#!/bin/bash
# /opt/semstreams/healthcheck.sh

echo "[$(date)] Running health checks..."

# Check container is running
if ! docker compose ps semstreams | grep -q "Up"; then
  echo "ERROR: SemStreams container not running"
  exit 1
fi

# Check health endpoint
if ! curl -sf http://localhost:8080/health > /dev/null; then
  echo "ERROR: Health endpoint not responding"
  exit 1
fi

# Check NATS connection
if ! curl -sf http://localhost:8080/health | grep -q '"nats":"connected"'; then
  echo "ERROR: NATS not connected"
  exit 1
fi

# Check memory usage
MEM_USAGE=$(docker stats --no-stream --format "{{.MemPerc}}" semstreams | sed 's/%//')
if [ ${MEM_USAGE%.*} -gt 90 ]; then
  echo "WARNING: Memory usage high: $MEM_USAGE%"
fi

# Check disk space
DISK_USAGE=$(df -h / | tail -1 | awk '{print $5}' | sed 's/%//')
if [ $DISK_USAGE -gt 80 ]; then
  echo "WARNING: Disk usage high: $DISK_USAGE%"
fi

echo "All health checks passed"
```

**Run Health Check:**
```bash
chmod +x /opt/semstreams/healthcheck.sh

# Manual check
./healthcheck.sh

# Automated check (every 5 minutes)
crontab -e
*/5 * * * * /opt/semstreams/healthcheck.sh >> /var/log/semstreams-health.log 2>&1
```

### Alerting

**Simple Email Alerts:**
```bash
#!/bin/bash
# /opt/semstreams/alert.sh

ALERT_EMAIL="ops@company.com"

if ! /opt/semstreams/healthcheck.sh; then
  echo "SemStreams health check failed" | \
    mail -s "ALERT: SemStreams Health Check Failed" $ALERT_EMAIL
fi
```

**Integration with Monitoring Systems:**
- **Prometheus + Alertmanager**: Use `/metrics` endpoint
- **Grafana**: Configure alerts in dashboards (see configs/grafana/)
- **External monitoring**: Use health endpoint for uptime checks

## Emergency Procedures

### Complete System Failure

**Symptoms:**
- All containers stopped
- Host system crashed
- Data corruption

**Recovery Steps:**
```bash
# 1. Check host system
df -h
free -h
docker ps -a

# 2. Check Docker daemon
systemctl status docker
systemctl restart docker  # If needed

# 3. Check for data corruption
docker volume inspect semstreams_nats-data

# 4. Attempt normal startup
cd /opt/semstreams
docker compose up -d

# 5. If startup fails, restore from backup
./restore.sh

# 6. Verify recovery
curl http://localhost:8080/health
docker compose logs -f
```

### NATS Data Corruption

**Symptoms:**
- NATS won't start
- JetStream errors
- Stream corruption messages

**Recovery:**
```bash
# 1. Backup current state (even if corrupted)
cp -r /var/lib/docker/volumes/semstreams_nats-data \
     /backup/corrupted-$(date +%Y%m%d)

# 2. Stop services
docker compose down

# 3. Remove corrupted volume
docker volume rm semstreams_nats-data

# 4. Restore from last good backup
./restore.sh

# 5. Verify
docker compose up -d
curl http://localhost:8222/healthz
```

### Split Brain (Multiple Instances)

**Note**: This shouldn't happen with SemStreams' output→input architecture, but if running custom NATS networking:

**Symptoms:**
- Multiple instances think they're primary
- Conflicting data
- Message duplication

**Resolution:**
```bash
# Stop all instances
ssh location-a "cd /opt/semstreams && docker compose down"
ssh location-b "cd /opt/semstreams && docker compose down"
ssh location-c "cd /opt/semstreams && docker compose down"

# Verify all stopped
# Wait 30 seconds

# Start primary location first
ssh location-a "cd /opt/semstreams && docker compose up -d"
sleep 30

# Start secondary locations
ssh location-b "cd /opt/semstreams && docker compose up -d"
ssh location-c "cd /opt/semstreams && docker compose up -d"
```

## Maintenance Windows

### Planned Downtime

```bash
#!/bin/bash
# /opt/semstreams/maintenance.sh

echo "[MAINT] Starting maintenance window..."

# 1. Backup
./backup.sh

# 2. Graceful shutdown
docker compose down --timeout 30

# 3. Perform maintenance
# - Update containers
# - Update host system
# - Disk cleanup
# - etc.

# 4. Restart
docker compose up -d

# 5. Verify
sleep 10
curl http://localhost:8080/health

echo "[MAINT] Maintenance complete"
```

### Zero-Downtime Updates

**For critical edge devices that can't have downtime:**

```bash
# Option 1: Use Docker Swarm rolling updates (if acceptable)
docker swarm init
docker stack deploy -c docker-compose.yml semstreams
docker service update --image ghcr.io/c360/semstreams:v1.1.0 semstreams_app

# Option 2: Deploy to standby instance, switch over
# This requires network-level routing changes
```

## Useful Commands Reference

```bash
# Quick Status
docker compose ps
curl http://localhost:8080/health

# Follow Logs
docker compose logs -f --tail=100

# Resource Usage
docker stats --no-stream

# Restart Everything
docker compose restart

# Clean Restart
docker compose down && docker compose up -d

# View Config
docker exec semstreams cat /etc/semstreams/config.json

# View Metrics
curl http://localhost:8080/metrics

# NATS Status
curl http://localhost:8222/varz | jq .

# Backup
./backup.sh

# Restore
./restore.sh [backup-date]

# Health Check
./healthcheck.sh

# Update
docker compose pull && docker compose up -d
```

## See Also

- [Configuration Guide](CONFIGURATION.md) - Configuration management
- [Production Deployment](PRODUCTION.md) - Deployment patterns and architecture
- [Optional Services](../OPTIONAL_SERVICES.md) - Embedding, observability, TLS services
