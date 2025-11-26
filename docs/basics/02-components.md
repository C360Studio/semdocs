# Components

## Understanding SemStreams component types and how they work

---

## Overview

Components are the building blocks of SemStreams event flows. Each component has a specific role:

- **Input components:** Receive data from external sources
- **Processor components:** Transform, filter, or enrich events
- **Output components:** Send data to external systems
- **Storage components:** Persist data for later retrieval
- **Gateway components:** Provide HTTP/WebSocket access

Components communicate via **NATS subjects** (topics), not direct connections.

---

## Component Structure

Every component has the same basic structure:

```json
{
  "components": {
    "my_component": {
      "type": "input|processor|output|storage|gateway",
      "name": "unique-name",
      "enabled": true,
      "config": {
        "ports": {
          "inputs": [...],
          "outputs": [...]
        },
        // Component-specific config
      }
    }
  }
}
```

**Key fields:**

- **type:** Component category (determines behavior)
- **name:** Unique identifier for this component instance
- **enabled:** Set to `false` to disable without removing
- **config.ports:** Defines message routing (inputs/outputs)
- **config:** Component-specific configuration

---

## Input Components

Input components receive data from external sources and publish to NATS subjects.

### UDP Input

Listens for UDP packets on a port.

```json
{
  "udp": {
    "type": "input",
    "name": "udp_listener",
    "enabled": true,
    "config": {
      "ports": {
        "inputs": [],
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
  }
}
```

**Use cases:**

- MAVLink telemetry
- IoT sensor data
- Network monitoring

---

### HTTP Input

Receives HTTP POST requests.

```json
{
  "http": {
    "type": "input",
    "name": "http_listener",
    "enabled": true,
    "config": {
      "ports": {
        "outputs": [
          {
            "name": "http_out",
            "subject": "raw.http.messages"
          }
        ]
      },
      "bind": "0.0.0.0",
      "port": 8081,
      "path": "/ingest"
    }
  }
}
```

**Use cases:**

- Webhooks
- API integrations
- Mobile app events
- Browser analytics

---

## Processor Components

Processor components transform, filter, or enrich events.

### JSON Generic

Parses raw bytes into JSON messages.

```json
{
  "json_generic": {
    "type": "processor",
    "name": "json_parser",
    "enabled": true,
    "config": {
      "ports": {
        "inputs": [
          {
            "name": "parser_in",
            "subject": "raw.udp.messages"
          }
        ],
        "outputs": [
          {
            "name": "parser_out",
            "subject": "generic.messages",
            "interface": "core.json.v1"
          }
        ]
      }
    }
  }
}
```

**Interfaces:** Declares output interface type for downstream validation

---

### JSON Filter

Filters messages based on criteria.

```json
{
  "json_filter": {
    "type": "processor",
    "name": "telemetry_filter",
    "enabled": true,
    "config": {
      "ports": {
        "inputs": [
          {
            "name": "filter_in",
            "subject": "generic.messages",
            "interface": "core.json.v1"
          }
        ],
        "outputs": [
          {
            "name": "filter_out",
            "subject": "telemetry.filtered"
          }
        ]
      },
      "criteria": {
        "type": {"equals": "telemetry"},
        "battery.level": {"lte": 20.0}
      }
    }
  }
}
```

**Filter operators:**

- `equals`, `not_equals`
- `gt`, `gte`, `lt`, `lte` (numeric comparisons)
- `contains`, `starts_with`, `ends_with` (string matching)
- `in`, `not_in` (array membership)

---

### JSON to Entity

Converts JSON messages to graph entities.

```json
{
  "json_to_entity": {
    "type": "processor",
    "name": "entity_converter",
    "enabled": true,
    "config": {
      "ports": {
        "inputs": [
          {
            "name": "entity_in",
            "subject": "telemetry.filtered",
            "interface": "core.json.v1"
          }
        ],
        "outputs": [
          {
            "name": "entity_out",
            "subject": "events.graph.entity.telemetry",
            "interface": "graph.Entity.v1"
          }
        ]
      },
      "entity_id_field": "entity_id",
      "entity_type_field": "entity_type",
      "entity_class": "Object",
      "entity_role": "primary"
    }
  }
}
```

**Key mappings:**

- **entity_id_field:** JSON field containing entity ID
- **entity_type_field:** JSON field containing entity type
- **entity_class:** Object, Event, or Concept
- **entity_role:** primary, supporting, or context

---

### Rule Processor

Evaluates rules against messages and entities.

```json
{
  "rule": {
    "type": "processor",
    "name": "rule_processor",
    "enabled": true,
    "config": {
      "ports": {
        "inputs": [
          {
            "name": "rule_in",
            "subject": "generic.messages",
            "interface": "core.json.v1"
          }
        ],
        "outputs": [
          {
            "name": "rule_out",
            "subject": "events.rule.triggered"
          }
        ]
      },
      "enable_graph_integration": true,
      "graph_event_subject_prefix": "events.graph.entity"
    }
  }
}
```

**Rule integration:**

- Evaluates rules against incoming messages
- Optionally creates graph entities when rules trigger
- Publishes triggered rule events

---

### Graph Processor

Builds and maintains the semantic graph.

```json
{
  "graph": {
    "type": "processor",
    "name": "graph-processor",
    "enabled": true,
    "config": {
      "workers": 8,
      "queue_size": 5000,
      "input_subject": "events.graph.entity.*",
      "indexer": {
        "indexes": {
          "predicate": true,
          "incoming": true,
          "alias": true,
          "spatial": false,
          "temporal": false
        },
        "embedding": {
          "enabled": false
        }
      }
    }
  }
}
```

**Core functionality:**

- Stores entities in NATS KV
- Builds relationship indexes
- Enables graph queries
- Optional: Semantic search via embeddings

---

## Output Components

Output components send data to external systems.

### File Output

Writes messages to files.

```json
{
  "file": {
    "type": "output",
    "name": "file_writer",
    "enabled": true,
    "config": {
      "ports": {
        "inputs": [
          {
            "name": "file_in",
            "subject": "telemetry.filtered"
          }
        ]
      },
      "directory": "/data",
      "file_prefix": "telemetry",
      "format": "jsonl"
    }
  }
}
```

**Formats:** `json`, `jsonl` (newline-delimited)

---

### HTTP POST Output

Sends messages to HTTP endpoints.

```json
{
  "httppost": {
    "type": "output",
    "name": "webhook",
    "enabled": true,
    "config": {
      "ports": {
        "inputs": [
          {
            "name": "http_in",
            "subject": "events.rule.triggered"
          }
        ]
      },
      "url": "https://hooks.example.com/webhook",
      "retry_max": 3
    }
  }
}
```

**Use cases:**
- Webhook notifications
- External API calls
- Alert systems
- Integration with third-party services

---

### WebSocket Output

Streams messages to WebSocket clients.

```json
{
  "websocket": {
    "type": "output",
    "name": "websocket_server",
    "enabled": true,
    "config": {
      "ports": {
        "inputs": [
          {
            "name": "ws_in",
            "subject": "telemetry.*"
          }
        ]
      },
      "http_port": 8082,
      "path": "/ws"
    }
  }
}
```

**Use cases:**
- Real-time dashboards
- Live monitoring UIs
- Browser-based applications

---

## Storage Components

Storage components persist data for later retrieval.

### Object Store

Stores large data objects in NATS Object Store.

```json
{
  "objectstore": {
    "type": "storage",
    "name": "objectstore",
    "enabled": true,
    "config": {
      "ports": {
        "inputs": [
          {
            "name": "store_in",
            "subject": "large.objects"
          }
        ]
      },
      "bucket_name": "semstreams_objects",
      "data_cache": {
        "enabled": true,
        "strategy": "hybrid",
        "max_size": 5000,
        "ttl": "1h"
      }
    }
  }
}
```

**Use cases:**
- Large payloads (>1MB)
- Binary data (images, videos)
- Archived events

---

## Gateway Components

Gateway components provide HTTP/WebSocket access to graph queries.

### API Gateway

Exposes HTTP endpoints for graph queries.

```json
{
  "api-gateway": {
    "type": "gateway",
    "name": "http_gateway",
    "enabled": true,
    "config": {
      "enable_cors": true,
      "cors_origins": ["*"],
      "max_request_size": 1048576,
      "routes": [
        {
          "path": "/search/semantic",
          "method": "POST",
          "nats_subject": "graph.query.semantic",
          "timeout": "5s"
        },
        {
          "path": "/entity/:id",
          "method": "GET",
          "nats_subject": "graph.query.entity",
          "timeout": "2s"
        }
      ]
    }
  }
}
```

**HTTP → NATS mapping:**
- HTTP request → NATS request/reply
- Response timeout configurable per route
- Path parameters extracted and passed in request

---

## Component Lifecycle

### Startup Sequence

```
1. Load configuration
2. Initialize services (NATS, metrics, etc.)
3. Start enabled components (type order: input, processor, output)
4. Subscribe to NATS subjects
5. Begin processing messages
```

### Shutdown Sequence

```
1. Stop accepting new messages
2. Drain in-flight messages (graceful shutdown)
3. Close NATS subscriptions
4. Stop components (reverse order: output, processor, input)
5. Close NATS connection
```

---

## Component Communication

Components communicate via **NATS subjects**, not direct references.

**Example flow:**

```
UDP Input
  ↓ publish("raw.udp")
JSON Parser
  ↓ subscribe("raw.udp")
  ↓ publish("parsed.json")
Filter
  ↓ subscribe("parsed.json")
  ↓ publish("filtered.json")
Entity Converter
  ↓ subscribe("filtered.json")
  ↓ publish("events.graph.entity.telemetry")
Graph Processor
  ↓ subscribe("events.graph.entity.*")
```

**Key advantage:** Components can be added/removed without changing others.

---

## Best Practices

### Component Naming

✅ **Good:** `udp_mavlink`, `json_telemetry_parser`, `alert_webhook`
❌ **Bad:** `component1`, `proc`, `output`

Use descriptive names that indicate purpose.

### Port Naming

✅ **Good:** `telemetry_in`, `filtered_out`, `alert_events`
❌ **Bad:** `in1`, `out`, `port`

Port names help trace message flow.

### Subject Naming

Follow the convention: `{category}.{component}.{type}.{detail}`

✅ **Good:** `raw.udp.messages`, `events.graph.entity.drone`
❌ **Bad:** `data`, `stuff`, `output`

---

## Next Steps

- **[Routing](03-routing.md)** - Learn message routing patterns
- **[First Flow](04-first-flow.md)** - Build your first event flow
- **[Graph Processor](../graph/01-entities.md)** - Understanding entity graphs
- **[Configuration Reference](../api/03-configuration-reference.md)** - Complete component config

---

**Components are building blocks** - Combine them to create event flows that automatically build graphs.
