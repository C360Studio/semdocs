# Your First Event Flow

**Hands-on tutorial: Build a complete event flow that automatically creates a graph**

> **Quick Start Alternative:** Want to run SemStreams immediately with Docker? See [Examples](08-examples.md) for ready-to-run Docker Compose configurations. The [Quickstart Example](../../examples/quickstart/) gets you running in 30 seconds with minimal configuration.

---

## Goal

Build a working SemStreams flow that:

1. Receives UDP telemetry messages
2. Parses JSON
3. Filters for low battery alerts
4. Builds a graph of drones and fleets
5. Enables graph queries

---

## Prerequisites

```bash
# Install SemStreams
go install github.com/c360/semstreams/cmd/semstreams@latest

# Start NATS (Docker)
docker run -d --name nats -p 4222:4222 nats:latest
```

---

## Step 1: Create Configuration

Create `config.json`:

```json
{
  "version": "1.0.0",
  "platform": {
    "org": "tutorial",
    "id": "first-flow",
    "type": "development",
    "region": "local",
    "instance_id": "dev-001",
    "environment": "dev"
  },
  "nats": {
    "urls": ["nats://localhost:4222"]
  },
  "services": {
    "service-manager": {
      "name": "service-manager",
      "enabled": true,
      "config": {
        "http_port": 8080
      }
    }
  },
  "components": {
    "udp": {
      "type": "input",
      "name": "udp",
      "enabled": true,
      "config": {
        "ports": {
          "outputs": [
            {
              "name": "udp_out",
              "subject": "raw.udp.messages",
              "type": "nats"
            }
          ]
        },
        "bind": "0.0.0.0",
        "port": 14550,
        "buffer_size": 65536
      }
    },
    "json_generic": {
      "type": "processor",
      "name": "json_generic",
      "enabled": true,
      "config": {
        "ports": {
          "inputs": [
            {
              "name": "parser_in",
              "subject": "raw.udp.messages",
              "type": "nats"
            }
          ],
          "outputs": [
            {
              "name": "parser_out",
              "subject": "parsed.json",
              "type": "nats",
              "interface": "core.json.v1"
            }
          ]
        }
      }
    },
    "json_filter": {
      "type": "processor",
      "name": "json_filter",
      "enabled": true,
      "config": {
        "ports": {
          "inputs": [
            {
              "name": "filter_in",
              "subject": "parsed.json",
              "type": "nats",
              "interface": "core.json.v1"
            }
          ],
          "outputs": [
            {
              "name": "filter_out",
              "subject": "telemetry.alerts",
              "type": "nats",
              "interface": "core.json.v1"
            }
          ]
        },
        "criteria": {
          "battery.level": {
            "lte": 20.0
          }
        }
      }
    },
    "json_to_entity": {
      "type": "processor",
      "name": "json_to_entity",
      "enabled": true,
      "config": {
        "ports": {
          "inputs": [
            {
              "name": "entity_in",
              "subject": "parsed.json",
              "type": "nats",
              "interface": "core.json.v1"
            }
          ],
          "outputs": [
            {
              "name": "entity_out",
              "subject": "events.graph.entity.{entity_type}",
              "type": "nats",
              "interface": "graph.Entity.v1"
            }
          ]
        },
        "entity_id_field": "entity_id",
        "entity_type_field": "entity_type",
        "entity_class": "Object",
        "entity_role": "primary"
      }
    },
    "graph": {
      "type": "processor",
      "name": "graph-processor",
      "enabled": true,
      "config": {
        "workers": 4,
        "queue_size": 1000,
        "input_subject": "events.graph.entity.*",
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
        }
      }
    },
    "file": {
      "type": "output",
      "name": "file",
      "enabled": true,
      "config": {
        "ports": {
          "inputs": [
            {
              "name": "file_in",
              "subject": "telemetry.alerts",
              "type": "nats",
              "interface": "core.json.v1"
            }
          ]
        },
        "directory": "/tmp",
        "file_prefix": "alerts",
        "format": "jsonl"
      }
    }
  }
}
```

---

## Step 2: Start SemStreams

```bash
semstreams --config config.json
```

**Expected output:**

```text
INFO Starting SemStreams runtime
INFO Service Manager listening on :8080
INFO UDP input listening on 0.0.0.0:14550
INFO JSON Generic processor started
INFO JSON Filter processor started
INFO JSON to Entity converter started
INFO Graph Processor started (workers: 4)
INFO File output started
```

---

## Step 3: Send Test Data

Create `send-telemetry.sh`:

```bash
#!/bin/bash

# Send drone telemetry with 6-part federated entity IDs
# Format: org.platform.domain.system.type.instance
echo '{"entity_id":"tutorial.first-flow.robotics.gcs1.drone.001","entity_type":"robotics.drone","fleet":"rescue","battery":{"level":85.2},"status":"active"}' | nc -u localhost 14550
echo '{"entity_id":"tutorial.first-flow.robotics.gcs1.drone.002","entity_type":"robotics.drone","fleet":"rescue","battery":{"level":15.4},"status":"critical"}' | nc -u localhost 14550
echo '{"entity_id":"tutorial.first-flow.robotics.gcs1.drone.003","entity_type":"robotics.drone","fleet":"rescue","battery":{"level":92.1},"status":"active"}' | nc -u localhost 14550
echo '{"entity_id":"tutorial.first-flow.ops.hq.fleet.rescue","entity_type":"ops.fleet","status":"active","location":"base-west"}' | nc -u localhost 14550
```

Run it:

```bash
chmod +x send-telemetry.sh
./send-telemetry.sh
```

---

## Step 4: Verify Event Flow

### Check Logs

```text
INFO [UDP] Received 4 messages
INFO [JSON Generic] Parsed 4 messages
INFO [JSON Filter] Filtered: 1 low battery alert (tutorial.first-flow.robotics.gcs1.drone.002)
INFO [Entity Converter] Created 4 entities
INFO [Graph] Indexed 4 entities
INFO [File] Wrote 1 alert to /tmp/alerts-20240115.jsonl
```

### Check Alert File

```bash
cat /tmp/alerts-*.jsonl
```

**Output:**

```json
{"entity_id":"tutorial.first-flow.robotics.gcs1.drone.002","entity_type":"robotics.drone","fleet":"rescue","battery":{"level":15.4},"status":"critical"}
```

---

## Step 5: Query the Graph

### Health Check

```bash
curl http://localhost:8080/health
```

**Response:**

```json
{"status":"healthy","timestamp":"2024-01-15T10:30:00Z"}
```

### Graph Statistics

```bash
curl http://localhost:8080/api/v1/graph/stats
```

**Response:**

```json
{
  "entity_count": 4,
  "entity_types": {
    "robotics.drone": 3,
    "ops.fleet": 1
  },
  "relationship_count": 3
}
```

### Find All Drones

```bash
curl "http://localhost:8080/api/v1/graph/query?type=robotics.drone"
```

**Response:**

```json
{
  "entities": [
    {
      "id": "tutorial.first-flow.robotics.gcs1.drone.001",
      "type": "robotics.drone",
      "properties": {"fleet": "rescue", "battery": {"level": 85.2}, "status": "active"}
    },
    {
      "id": "tutorial.first-flow.robotics.gcs1.drone.002",
      "type": "robotics.drone",
      "properties": {"fleet": "rescue", "battery": {"level": 15.4}, "status": "critical"}
    },
    {
      "id": "tutorial.first-flow.robotics.gcs1.drone.003",
      "type": "robotics.drone",
      "properties": {"fleet": "rescue", "battery": {"level": 92.1}, "status": "active"}
    }
  ]
}
```

### Find Drones in Rescue Fleet (PathRAG Traversal)

```bash
curl "http://localhost:8080/api/v1/graph/traverse?start=tutorial.first-flow.ops.hq.fleet.rescue&direction=incoming"
```

**Response:**

```json
{
  "entities": [
    {"id": "tutorial.first-flow.robotics.gcs1.drone.001", "type": "robotics.drone", ...},
    {"id": "tutorial.first-flow.robotics.gcs1.drone.002", "type": "robotics.drone", ...},
    {"id": "tutorial.first-flow.robotics.gcs1.drone.003", "type": "robotics.drone", ...}
  ]
}
```

### Find Low Battery Drones

```bash
curl "http://localhost:8080/api/v1/graph/query?type=robotics.drone&battery.level.lte=20"
```

**Response:**

```json
{
  "entities": [
    {"id": "tutorial.first-flow.robotics.gcs1.drone.002", "type": "robotics.drone", "properties": {"battery": {"level": 15.4}, ...}}
  ]
}
```

---

## What Just Happened?

**Event flow:**

```text
1. UDP Input received 4 JSON messages
2. JSON Generic parsed them
3. JSON Filter identified 1 low battery alert (battery ≤ 20%)
4. JSON to Entity converted all 4 to EntityPayload (implements Graphable interface)
5. Graph Processor:
   - Stored entities in ENTITY_STATES KV bucket
   - Built PREDICATE_INDEX (belongs_to → fleet relationships)
   - Built INCOMING_INDEX (fleet.rescue ← drones)
6. File Output wrote 1 alert to disk
```

**Graph automatically created:**

```text
Entities (6-part federated IDs):
  tutorial.first-flow.robotics.gcs1.drone.001 (robotics.drone)
  tutorial.first-flow.robotics.gcs1.drone.002 (robotics.drone)
  tutorial.first-flow.robotics.gcs1.drone.003 (robotics.drone)
  tutorial.first-flow.ops.hq.fleet.rescue (ops.fleet)

Relationships (from Graphable.Triples()):
  drone.001 → belongs_to → fleet.rescue
  drone.002 → belongs_to → fleet.rescue
  drone.003 → belongs_to → fleet.rescue

Indexes (NATS KV buckets):
  PREDICATE_INDEX: "belongs_to" → [drone.001, drone.002, drone.003]
  INCOMING_INDEX: "fleet.rescue" ← [drone.001, drone.002, drone.003]
```

You didn't manually create the graph - it built itself from your events via the **Graphable interface**.

---

## More Advanced Flows

### Add More Components

**WebSocket output** for real-time dashboards:

```json
{
  "websocket": {
    "type": "output",
    "config": {
      "ports": {
        "inputs": [{"subject": "telemetry.alerts"}]
      },
      "http_port": 8082,
      "path": "/ws"
    }
  }
}
```

**HTTP webhook** for external alerts:

```json
{
  "httppost": {
    "type": "output",
    "config": {
      "ports": {
        "inputs": [{"subject": "telemetry.alerts"}]
      },
      "url": "https://alerts.example.com/webhook",
      "retry_max": 3
    }
  }
}
```

### Enable Semantic Search

Add SemEmbed service:

```json
{
  "graph": {
    "config": {
      "indexer": {
        "embedding": {
          "enabled": true,
          "service_url": "http://localhost:8081"
        }
      }
    }
  }
}
```

Start SemEmbed:

```bash
docker run -d -p 8081:8081 ghcr.io/c360/semembed:latest
```

Query semantically:

```bash
curl -X POST http://localhost:8080/search/semantic \
  -d '{"query": "emergency battery situation", "limit": 5}'
```

### Add Rules

Enable rule processor:

```json
{
  "rule": {
    "type": "processor",
    "config": {
      "ports": {
        "inputs": [{"subject": "parsed.json"}],
        "outputs": [{"subject": "events.rule.triggered"}]
      }
    }
  }
}
```

Define rules via API or config files.

---

## Common Issues

### "Port already in use"

```shell
Error: bind: address already in use (port 14550)
```

**Fix:** Change UDP port in config:

```json
{"udp": {"config": {"port": 14551}}}
```

### "NATS connection refused"

```shell
Error: dial tcp 127.0.0.1:4222: connection refused
```

**Fix:** Start NATS:

```bash
docker run -d -p 4222:4222 nats:latest
```

### "No entities in graph"

**Check:**

1. Are messages being sent? (Check logs)
2. Is JSON valid? (Use `jq` to validate)
3. Are entity fields present? (entity_id, entity_type required)

---

## Complete Examples

Ready-to-run Docker Compose examples with minimal configuration:

### [Quickstart](../../examples/quickstart/)
**NATS + SemStreams** - Get running in 30 seconds
```bash
cd examples/quickstart/ && docker compose up -d
```

### [With UI](../../examples/with-ui/)
**+ Visual Flow Builder** - Drag-and-drop component configuration
```bash
cd examples/with-ui/ && docker compose up -d
open http://localhost:3000
```

### [With Embeddings](../../examples/with-embeddings/)
**+ Semantic Search** - Neural embeddings for similarity matching
```bash
cd examples/with-embeddings/ && docker compose up -d
```

### [Production](../../examples/production/)
**Hardened Deployment** - Resource limits, security best practices
```bash
cd examples/production/
cp config.example.json config.json
docker compose up -d
```

**See:** [Examples Guide](08-examples.md) for detailed comparison and usage

---

## Next Steps

- **[Components](02-components.md)** - Learn all component types
- **[Routing](03-routing.md)** - Advanced routing patterns
- **[Graph Queries](../graph/03-queries.md)** - Query the graph in detail
- **[Query Fundamentals](../advanced/01-query-fundamentals.md)** - PathRAG vs GraphRAG decisions

---

**Congratulations!** You've built your first event flow that automatically creates a queryable graph.
