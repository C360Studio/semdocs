# Message Routing

How events flow through SemStreams via NATS subjects

---

## NATS Core vs JetStream

SemStreams uses **two NATS patterns** depending on the communication type:

### NATS Core (Request/Reply)

Used for **synchronous operations** that need immediate responses:

- **Graph entity mutations**: Create, update, delete entities
- **Query operations**: Graph queries that return results immediately
- **Configuration updates**: Real-time config changes

**Characteristics:**

- Low latency (microseconds)
- No persistence
- Direct request/response
- Best for operations requiring immediate confirmation

**Example:**

```json
{
  "ports": {
    "inputs": [{
      "subject": "graph.entity.mutate",
      "type": "nats_core",
      "pattern": "request_reply"
    }]
  }
}
```

### JetStream (Pub/Sub)

Used for **asynchronous event streaming**:

- **Event flows**: Telemetry data, sensor readings, alerts
- **Component outputs**: Parser results, filtered events
- **Graph events**: Entity changes, relationship updates

**Characteristics:**

- Message persistence
- At-least-once delivery
- Replay capability
- Consumer groups for load balancing
- Best for event-driven architectures

**Example:**

```json
{
  "ports": {
    "outputs": [{
      "subject": "events.graph.entity.drone",
      "type": "jetstream",
      "pattern": "pubsub"
    }]
  }
}
```

**When to use what:**

- **Request/Reply (NATS Core)**: Need immediate response, synchronous operation, mutation commands
- **Pub/Sub (JetStream)**: Event streaming, multiple subscribers, need persistence/replay

---

## Subject-Based Routing

SemStreams uses **NATS subjects** (topics) for message routing, not direct component connections.

**Traditional approach (direct connections):**

```text
ComponentA.output → ComponentB.input
```

**SemStreams approach (publish/subscribe):**

```text
ComponentA → publish("raw.data") → NATS
ComponentB → subscribe("raw.data") → receives message
ComponentC → subscribe("raw.data") → also receives message
```

**Key advantage:** Multiple subscribers, loose coupling, dynamic routing.

---

## Subject Naming Convention

```text
{category}.{component}.{entity_type}.{detail}
```

**Examples:**

```text
raw.udp.messages              # Raw UDP input
events.graph.entity.drone     # Drone entity events
events.graph.entity.*         # All entity types (wildcard)
events.rule.triggered         # Rule trigger events
graph.query.semantic          # Semantic search queries
```

**Wildcards:**

- `*` matches single token: `events.graph.entity.*` → matches `events.graph.entity.drone`
- `>` matches multiple tokens: `events.>` → matches `events.graph.entity.drone.battery`

---

## Basic Routing Patterns

### 1. Simple Linear Flow

```text
UDP Input → Parser → Graph
```

Configuration:

```json
{
  "udp": {
    "config": {
      "ports": {
        "outputs": [{"subject": "raw.udp"}]
      }
    }
  },
  "json_generic": {
    "config": {
      "ports": {
        "inputs": [{"subject": "raw.udp"}],
        "outputs": [{"subject": "parsed.json"}]
      }
    }
  },
  "graph": {
    "config": {
      "input_subject": "parsed.json"
    }
  }
}
```

Message flow:

```text
UDP → publish("raw.udp")
       ↓
Parser → subscribe("raw.udp") → publish("parsed.json")
         ↓
Graph → subscribe("parsed.json")
```

---

### 2. Fan-Out (Multiple Subscribers)

```text
Parser
  ├─→ Filter
  ├─→ Graph
  └─→ File Output
```

Configuration:

```json
{
  "json_generic": {
    "config": {
      "ports": {
        "outputs": [{"subject": "parsed.json"}]
      }
    }
  },
  "json_filter": {
    "config": {
      "ports": {
        "inputs": [{"subject": "parsed.json"}],
        "outputs": [{"subject": "filtered.json"}]
      }
    }
  },
  "graph": {
    "config": {
      "input_subject": "parsed.json"
    }
  },
  "file": {
    "config": {
      "ports": {
        "inputs": [{"subject": "parsed.json"}]
      }
    }
  }
}
```

Message flow:

```text
Parser → publish("parsed.json")
          ↓
     NATS distributes to:
          ├─→ Filter (subscribes)
          ├─→ Graph (subscribes)
          └─→ File (subscribes)
```

All three components receive the same message independently.

---

### 3. Content-Based Routing

```text
Filter (battery < 20%)
  ├─→ Alert (if match)
  └─→ Logger (all)
```

Configuration:

```json
{
  "json_filter": {
    "config": {
      "ports": {
        "inputs": [{"subject": "telemetry"}],
        "outputs": [{"subject": "telemetry.low_battery"}]
      },
      "criteria": {
        "battery.level": {"lte": 20.0}
      }
    }
  },
  "httppost_alert": {
    "config": {
      "ports": {
        "inputs": [{"subject": "telemetry.low_battery"}]
      }
    }
  },
  "file_logger": {
    "config": {
      "ports": {
        "inputs": [{"subject": "telemetry"}]
      }
    }
  }
}
```

Message flow:

```text
Telemetry → publish("telemetry")
             ↓
        NATS distributes to:
             ├─→ Filter → (if battery < 20%) → publish("telemetry.low_battery") → Alert
             └─→ Logger (all messages)
```

---

### 4. Entity Type Routing

```text
Entity Converter
  ├─→ events.graph.entity.drone
  ├─→ events.graph.entity.sensor
  └─→ events.graph.entity.user
```

Configuration:

```json
{
  "json_to_entity": {
    "config": {
      "ports": {
        "outputs": [{"subject": "events.graph.entity.{entity_type}"}]
      }
    }
  },
  "graph": {
    "config": {
      "input_subject": "events.graph.entity.*"
    }
  }
}
```

Subject template: `{entity_type}` gets replaced with actual entity type from message.

---

## Advanced Routing Patterns

### 5. Aggregation (Multiple Inputs, Single Output)

```text
UDP → Parser ───┐
HTTP → Parser ──┤→ Graph
MQTT → Parser ──┘
```

Configuration:

```json
{
  "udp": {"config": {"ports": {"outputs": [{"subject": "raw.messages"}]}}},
  "http": {"config": {"ports": {"outputs": [{"subject": "raw.messages"}]}}},
  "mqtt": {"config": {"ports": {"outputs": [{"subject": "raw.messages"}]}}},
  "json_generic": {
    "config": {
      "ports": {
        "inputs": [{"subject": "raw.messages"}],
        "outputs": [{"subject": "parsed.json"}]
      }
    }
  }
}
```

All inputs publish to same subject → Single parser handles all sources.

---

### 6. Hierarchical Routing (Wildcards)

```text
events.graph.entity.drone        # Drone-specific handler
events.graph.entity.*             # All entity types
events.graph.>                    # All graph events
events.>                          # All events
```

Configuration:

```json
{
  "drone_processor": {
    "config": {
      "ports": {
        "inputs": [{"subject": "events.graph.entity.drone"}]
      }
    }
  },
  "all_entity_logger": {
    "config": {
      "ports": {
        "inputs": [{"subject": "events.graph.entity.*"}]
      }
    }
  }
}
```

Message to `events.graph.entity.drone` is received by:

- `drone_processor` (exact match)
- `all_entity_logger` (wildcard match)

---

## Port Configuration

### Input Ports

```json
{
  "ports": {
    "inputs": [
      {
        "name": "primary_in",
        "subject": "raw.udp.messages",
        "type": "nats",
        "interface": "core.json.v1"
      }
    ]
  }
}
```

**Fields:**

- **name:** Logical port name (for debugging/monitoring)
- **subject:** NATS subject to subscribe to
- **type:** `nats` (currently only type supported)
- **interface:** (Optional) Expected message format for validation

---

### Output Ports

```json
{
  "ports": {
    "outputs": [
      {
        "name": "primary_out",
        "subject": "parsed.json",
        "type": "nats",
        "interface": "core.json.v1"
      }
    ]
  }
}
```

**Subject templates:**

Use `{field_name}` to include message fields in subject:

```json
{
  "subject": "events.graph.entity.{entity_type}"
}
```

Message `{"entity_type": "drone", ...}` → Published to `events.graph.entity.drone`

---

## Interface Contracts

Interfaces declare message format for type safety.

**Example interfaces:**

```text
core.json.v1         # Generic JSON message
graph.Entity.v1      # Graph entity format
rule.Trigger.v1      # Rule trigger event
```

**Usage:**

```json
{
  "json_generic": {
    "config": {
      "ports": {
        "outputs": [
          {
            "subject": "parsed.json",
            "interface": "core.json.v1"
          }
        ]
      }
    }
  },
  "json_filter": {
    "config": {
      "ports": {
        "inputs": [
          {
            "subject": "parsed.json",
            "interface": "core.json.v1"
          }
        ]
      }
    }
  }
}
```

**Validation:** SemStreams validates input/output interface compatibility at startup.

---

## NATS JetStream Features

### Message Persistence

Enable persistence for durability:

```json
{
  "nats": {
    "jetstream": {
      "enabled": true,
      "stream_name": "SEMSTREAMS_EVENTS",
      "subjects": ["events.>"],
      "retention": "limits",
      "max_age": "24h"
    }
  }
}
```

**Benefits:**

- Messages survive restarts
- Replay capability
- At-least-once delivery

---

### Consumer Groups

Load balance across instances:

```json
{
  "graph": {
    "config": {
      "input_subject": "events.graph.entity.*",
      "consumer_group": "graph-processors"
    }
  }
}
```

**Behavior:**

- Multiple graph processor instances
- Each message delivered to ONE instance (load balanced)
- Automatic failover if instance crashes

---

## Routing Best Practices

### 1. Use Descriptive Subjects

✅ **Good:** `raw.udp.mavlink`, `events.graph.entity.drone`
❌ **Bad:** `data`, `messages`, `output`

### 2. Namespace by Category

```text
raw.*                  # Raw input data
parsed.*               # Parsed messages
events.graph.*         # Graph events
events.rule.*          # Rule events
graph.query.*          # Query requests
```

### 3. Avoid Subject Explosion

❌ **Bad:** `events.drone.UAV-001`, `events.drone.UAV-002`, ... (10,000 subjects)
✅ **Good:** `events.drone` with entity_id in message body

### 4. Use Wildcards Sparingly

Wildcard subscriptions (`*`, `>`) on high-volume subjects impact performance.

✅ **Good:** Specific subscriptions for high-volume
❌ **Bad:** `events.>` receiving millions of messages/sec

---

## Debugging Message Flow

### Enable Message Logger

```json
{
  "services": {
    "message-logger": {
      "enabled": true,
      "config": {
        "monitor_subjects": ["raw.*", "events.>"],
        "log_level": "INFO"
      }
    }
  }
}
```

### Query Message Log

```bash
# Via NATS CLI
nats kv get MESSAGE_LOG <message_id>

# Via HTTP
curl http://localhost:8080/admin/messages?subject=raw.udp
```

---

## Next Steps

- **[First Flow](04-first-flow.md)** - Build your first event flow
- **[Components](02-components.md)** - Component types reference
- **[Graph Queries](../graph/03-queries.md)** - Query the graph
- **[Architecture Deep Dive](../advanced/03-architecture-deep-dive.md)** - How routing works internally

---

**Subject-based routing decouples components** - Add new subscribers without changing publishers, route based on content not topology.
