# Production Deployment Patterns

**Battle-tested patterns for running SemStreams in production**

---

## Overview

This guide covers production deployment patterns, operational best practices, and lessons learned from real-world SemStreams deployments.

---

## Deployment Topologies

### 1. Single-Instance (Development/Small Scale)

```
┌─────────────────────────────────────┐
│          Single Server              │
│                                     │
│  ┌──────────────────────────────┐  │
│  │  SemStreams Runtime          │  │
│  │  • All components            │  │
│  │  • NATS embedded             │  │
│  │  • Local storage             │  │
│  └──────────────────────────────┘  │
│                                     │
│  ┌──────────────────────────────┐  │
│  │  Optional Services           │  │
│  │  • SemEmbed                  │  │
│  │  • SemSummarize              │  │
│  └──────────────────────────────┘  │
└─────────────────────────────────────┘
```

**Use cases:**
- Development environments
- Small deployments (<1000 msgs/sec)
- Resource-constrained environments

**Configuration:**
```json
{
  "nats": {
    "urls": ["nats://localhost:4222"]
  },
  "graph": {
    "config": {
      "workers": 4,
      "queue_size": 1000
    }
  }
}
```

**Pros:**
- Simple operations
- Low resource overhead
- Fast iteration

**Cons:**
- No high availability
- Limited scalability
- Single point of failure

---

### 2. NATS Cluster (High Availability)

```
┌─────────────────────────────────────────────────────────┐
│                    NATS Cluster                         │
│  ┌─────────┐      ┌─────────┐      ┌─────────┐         │
│  │ NATS 1  │◄────►│ NATS 2  │◄────►│ NATS 3  │         │
│  │ (leader)│      │         │      │         │         │
│  └────┬────┘      └────┬────┘      └────┬────┘         │
└───────┼────────────────┼────────────────┼──────────────┘
        │                │                │
   ┌────▼────┐      ┌────▼────┐      ┌────▼────┐
   │SemStreams│      │SemStreams│      │SemStreams│
   │Instance 1│      │Instance 2│      │Instance 3│
   └──────────┘      └──────────┘      └──────────┘
```

**Use cases:**
- Production workloads
- High availability requirements
- Multi-region deployments

**NATS cluster configuration:**
```
# nats-server-1.conf
port: 4222
cluster {
  name: semstreams-cluster
  listen: 0.0.0.0:6222
  routes: [
    nats://nats-2:6222,
    nats://nats-3:6222
  ]
}
jetstream {
  store_dir: /data/jetstream
  max_memory: 4GB
  max_file: 100GB
}
```

**SemStreams configuration:**
```json
{
  "nats": {
    "urls": [
      "nats://nats-1:4222",
      "nats://nats-2:4222",
      "nats://nats-3:4222"
    ],
    "max_reconnect": -1,
    "reconnect_wait": "2s"
  }
}
```

**Pros:**
- High availability (no single point of failure)
- Automatic failover
- Distributed state

**Cons:**
- More complex operations
- Network coordination overhead
- Requires 3+ nodes for quorum

---

### 3. Edge-to-Cloud Federation

```text
┌─────────────────────────────────────┐
│ EDGE 1 (Vehicle/Drone)              │
│  • PathRAG only                     │
│  • Local queries                    │
│  • Selective sync                   │
└──────────────┬──────────────────────┘
               │
┌──────────────┴──────────────────────┐
│ EDGE 2 (IoT Gateway)                │
│  • PathRAG only                     │
│  • Bandwidth-aware                  │
└──────────────┬──────────────────────┘
               │
               ▼ WebSocket Output
┌─────────────────────────────────────┐
│ CLOUD (Central Server)              │
│  • WebSocket Input                  │
│  • Full stack (PathRAG + GraphRAG)  │
│  • Aggregate multi-edge data        │
│  • Cross-edge analytics             │
│  • Long-term storage                │
└─────────────────────────────────────┘
```

**Use cases:**
- IoT deployments
- Fleet management
- Distributed sensing
- Edge AI

**Edge configuration (Raspberry Pi):**
```json
{
  "platform": {
    "type": "edge",
    "region": "vehicle-001"
  },
  "nats": {
    "urls": ["nats://localhost:4222"]
  },
  "graph": {
    "config": {
      "workers": 2,
      "queue_size": 500,
      "indexer": {
        "indexes": {
          "predicate": true,
          "incoming": true,
          "spatial": false,
          "temporal": false
        },
        "embedding": {
          "enabled": false
        }
      }
    }
  },
  "components": [
    {
      "id": "edge-to-cloud",
      "type": "websocket_output",
      "config": {
        "url": "wss://cloud-hub:8080/edge-sync",
        "filters": {
          "priority": {"gte": 3}
        }
      }
    }
  ]
}
```

**Cloud hub configuration:**
```json
{
  "platform": {
    "type": "cloud-hub",
    "region": "us-west-2"
  },
  "components": [
    {
      "id": "receive-from-edges",
      "type": "websocket_input",
      "config": {
        "port": 8080,
        "path": "/edge-sync"
      }
    }
  ],
  "graph": {
    "config": {
      "workers": 16,
      "indexer": {
        "indexes": {
          "predicate": true,
          "incoming": true,
          "alias": true,
          "spatial": true,
          "temporal": true
        },
        "embedding": {
          "enabled": true,
          "service_url": "http://semembed:8081"
        }
      }
    }
  }
}
```

**Pros:**
- Low-latency local queries
- Bandwidth efficiency
- Offline operation
- Privacy/compliance (data stays local)

**Cons:**
- Complex federation setup
- Sync conflict resolution
- Multiple system versions

---

## Scaling Patterns

### Horizontal Scaling by Entity Type

```
┌─────────────────────────────────────┐
│          NATS Cluster               │
└──┬───────────────┬─────────────┬────┘
   │               │             │
   ▼               ▼             ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│SemStreams│  │SemStreams│  │SemStreams│
│Drone     │  │Sensor    │  │User      │
│Processor │  │Processor │  │Processor │
└──────────┘  └──────────┘  └──────────┘
   │               │             │
   ▼               ▼             ▼
events.graph      events.graph  events.graph
.entity.drone     .entity.sensor .entity.user
```

**Configuration per instance:**

```json
{
  "graph": {
    "config": {
      "input_subject": "events.graph.entity.drone"
    }
  }
}
```

**Pros:**
- Independent scaling per entity type
- Fault isolation
- Optimized config per workload

**Cons:**
- More complex deployment
- Cross-entity queries harder
- More instances to manage

---

### Vertical Scaling (Resource Tuning)

**Memory-optimized (large entity count):**
```json
{
  "graph": {
    "config": {
      "workers": 4,
      "indexer": {
        "indexes": {
          "predicate": true,
          "incoming": true,
          "alias": false,
          "spatial": false,
          "temporal": false
        },
        "embedding": {
          "enabled": false
        }
      },
      "querier": {
        "cache_enabled": true,
        "max_cache_size": 50000
      }
    }
  }
}
```

**Throughput-optimized (high message rate):**
```json
{
  "graph": {
    "config": {
      "workers": 16,
      "queue_size": 10000,
      "data_manager": {
        "edge_batch_size": 200
      },
      "indexer": {
        "batch_processing": {
          "size": 100
        }
      }
    }
  }
}
```

---

## High Availability Patterns

### Active-Active (Multiple Writers)

```
Load Balancer
      │
   ┌──┴──┐
   ▼     ▼
Instance 1  Instance 2
   │        │
   └────┬───┘
        ▼
   NATS Cluster
   (conflict-free)
```

**Key considerations:**
- NATS handles message ordering per subject
- JetStream provides exactly-once delivery
- Last-write-wins for entity updates
- Use entity versioning for conflict detection

**Entity versioning pattern:**
```json
{
  "id": "UAV-001",
  "version": 42,
  "updated_at": "2024-01-15T10:30:00Z",
  "data": {...}
}
```

---

### Active-Passive (Failover)

```
Primary Instance (Active)
      │
      ▼
 NATS Cluster
      │
      ▼
Standby Instance (Passive)
  (health check, ready to take over)
```

**Health check pattern:**
```bash
#!/bin/bash
# health-check.sh
STATUS=$(curl -sf http://primary:8080/health)
if [ $? -ne 0 ]; then
  echo "Primary failed, promoting standby..."
  systemctl start semstreams
fi
```

---

## Security Patterns

### NATS Authentication

**NKey authentication (recommended):**

```
# Generate seed
nk -gen user -pubout

# nats-server.conf
authorization {
  users: [
    {
      nkey: UAABC123...
    }
  ]
}
```

**SemStreams configuration:**
```json
{
  "nats": {
    "urls": ["nats://nats:4222"],
    "nkey_seed": "/secrets/nkey.seed"
  }
}
```

---

### TLS Encryption

**NATS TLS configuration:**
```
tls {
  cert_file: "/certs/server-cert.pem"
  key_file: "/certs/server-key.pem"
  ca_file: "/certs/ca.pem"
  verify: true
}
```

**SemStreams configuration:**
```json
{
  "nats": {
    "urls": ["tls://nats:4222"],
    "tls": {
      "cert_file": "/certs/client-cert.pem",
      "key_file": "/certs/client-key.pem",
      "ca_file": "/certs/ca.pem"
    }
  }
}
```

---

### API Gateway Security

**JWT authentication:**
```json
{
  "services": {
    "service-manager": {
      "config": {
        "http_port": 8080,
        "auth": {
          "type": "jwt",
          "jwt_secret": "${JWT_SECRET}",
          "jwt_issuer": "semstreams"
        }
      }
    }
  }
}
```

**Rate limiting:**
```json
{
  "api-gateway": {
    "config": {
      "rate_limit": {
        "enabled": true,
        "requests_per_minute": 1000,
        "burst": 50
      }
    }
  }
}
```

---

## Observability Patterns

### Metrics Export (Prometheus)

```json
{
  "services": {
    "metrics": {
      "enabled": true,
      "config": {
        "port": 9090,
        "path": "/metrics",
        "include_go_metrics": true
      }
    }
  }
}
```

**Key metrics to monitor:**

```promql
# Throughput
rate(semstreams_messages_processed_total[5m])

# Error rate
rate(semstreams_errors_total[5m])

# Queue depth (backlog indicator)
semstreams_queue_depth

# P95 latency
histogram_quantile(0.95, rate(semstreams_processing_duration_seconds_bucket[5m]))

# Cache hit rate
rate(semstreams_cache_hits_total[5m]) / rate(semstreams_cache_requests_total[5m])
```

---

### Structured Logging

```json
{
  "logging": {
    "level": "INFO",
    "format": "json",
    "output": "stdout",
    "fields": {
      "service": "semstreams",
      "instance_id": "${INSTANCE_ID}",
      "region": "${REGION}"
    }
  }
}
```

**Log aggregation (Loki/Elasticsearch):**

```yaml
# promtail.yaml
scrape_configs:
  - job_name: semstreams
    static_configs:
      - targets:
          - localhost
        labels:
          job: semstreams
          __path__: /var/log/semstreams/*.log
```

---

### Distributed Tracing

**OpenTelemetry integration:**

```json
{
  "telemetry": {
    "enabled": true,
    "exporter": "otlp",
    "endpoint": "http://otel-collector:4317",
    "sample_rate": 0.1
  }
}
```

**Trace query patterns:**
```
Find slow queries:
  duration > 1s AND operation = "graph.query"

Find error chains:
  status = error AND trace.span.count > 5
```

---

## Backup and Recovery

### NATS JetStream Backup

```bash
#!/bin/bash
# backup-jetstream.sh

BACKUP_DIR="/backups/jetstream-$(date +%Y%m%d)"
JETSTREAM_DIR="/data/jetstream"

# Stop SemStreams (graceful shutdown)
systemctl stop semstreams

# Backup JetStream data
tar czf ${BACKUP_DIR}/jetstream.tar.gz ${JETSTREAM_DIR}

# Restart SemStreams
systemctl start semstreams
```

**Automated backup (cron):**
```
0 2 * * * /scripts/backup-jetstream.sh
```

---

### Graph State Export/Import

**Export entities:**
```bash
curl -X POST http://localhost:8080/admin/export \
  -H "Content-Type: application/json" \
  -d '{"entity_type": "drone", "format": "jsonl"}' \
  -o backup-drones.jsonl
```

**Import entities:**
```bash
curl -X POST http://localhost:8080/admin/import \
  -H "Content-Type: application/x-ndjson" \
  --data-binary @backup-drones.jsonl
```

---

## Operational Runbooks

### 1. Deployment

```bash
#!/bin/bash
# deploy-semstreams.sh

# 1. Pull latest version
docker pull ghcr.io/c360/semstreams:latest

# 2. Run database migrations (if any)
./scripts/migrate.sh

# 3. Update configuration
cp configs/production.json /etc/semstreams/config.json

# 4. Restart service (rolling restart for zero downtime)
systemctl restart semstreams

# 5. Verify health
sleep 5
curl -f http://localhost:8080/health || exit 1

echo "Deployment successful"
```

---

### 2. Scaling Up

```bash
#!/bin/bash
# scale-up.sh

# Add new instance to load balancer
aws elb register-instances-with-load-balancer \
  --load-balancer-name semstreams-lb \
  --instances i-0abc123

# Wait for health checks
sleep 30

# Verify instance is healthy
STATUS=$(aws elb describe-instance-health \
  --load-balancer-name semstreams-lb \
  --instances i-0abc123 \
  --query 'InstanceStates[0].State' --output text)

if [ "$STATUS" = "InService" ]; then
  echo "Scale-up successful"
else
  echo "Scale-up failed: $STATUS"
  exit 1
fi
```

---

### 3. Incident Response

**High latency alert:**

```bash
# 1. Check queue depth
curl http://localhost:9090/metrics | grep semstreams_queue_depth

# 2. Check NATS pending
nats stream info GRAPH_ENTITIES

# 3. Check resource usage
top -b -n 1 | grep semstreams
free -h

# 4. If queue is growing, scale workers
curl -X POST http://localhost:8080/admin/config \
  -d '{"graph.workers": 16}'

# 5. If memory is high, restart
systemctl restart semstreams
```

---

## Disaster Recovery

### Scenario: Complete Data Loss

1. **Stop all SemStreams instances**
   ```bash
   systemctl stop semstreams
   ```

2. **Restore NATS JetStream backup**
   ```bash
   tar xzf /backups/jetstream-20240115/jetstream.tar.gz -C /
   ```

3. **Start NATS cluster**
   ```bash
   systemctl start nats-server
   ```

4. **Verify stream integrity**
   ```bash
   nats stream ls
   nats stream info GRAPH_ENTITIES
   ```

5. **Start SemStreams instances**
   ```bash
   systemctl start semstreams
   ```

6. **Verify health**
   ```bash
   curl http://localhost:8080/health
   curl http://localhost:8080/graph/stats
   ```

**Recovery Time Objective (RTO):** <30 minutes
**Recovery Point Objective (RPO):** Last backup (24 hours max)

---

## Cost Optimization

### Cloud Deployment (AWS)

**Small deployment (dev/test):**
- Instance: t3.medium (2 CPU, 4GB RAM)
- Cost: ~$30/month
- Throughput: ~1,000 msg/sec

**Medium deployment (production):**
- Instance: c6i.2xlarge (8 CPU, 16GB RAM)
- Cost: ~$250/month
- Throughput: ~10,000 msg/sec

**Large deployment (high scale):**
- Instance: c6i.4xlarge (16 CPU, 32GB RAM) × 3
- Cost: ~$1,500/month
- Throughput: ~50,000 msg/sec

---

### Edge Deployment (On-Premise)

**Raspberry Pi 4 (8GB):**
- Cost: $75 (one-time)
- Power: <15W
- Throughput: ~500 msg/sec (PathRAG only)

**Intel NUC (16GB):**
- Cost: $400 (one-time)
- Power: ~25W
- Throughput: ~5,000 msg/sec (full stack)

---

## Next Steps

- **[Performance Tuning](02-performance-tuning.md)** - Optimize your deployment
- **[Architecture Deep Dive](03-architecture-deep-dive.md)** - Understand internals
- **[PathRAG vs GraphRAG Decisions](01-pathrag-graphrag-decisions.md)** - Configuration guidance
- **[Algorithm Reference](05-algorithm-reference.md)** - Native algorithm details

---

**Production is about operations** - Start simple, monitor everything, scale based on data, automate recovery.
