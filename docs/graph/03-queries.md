# Graph Queries

**Querying entities and relationships**

---

## Query Interfaces

SemStreams provides three query interfaces:

| Interface | Use Case | Protocol |
|-----------|----------|----------|
| **GraphQL** | Interactive queries, UI dashboards | HTTP (GraphQL) |
| **NATS Request/Reply** | Programmatic access, service-to-service | NATS |
| **Semantic Search** | Natural language queries | HTTP Gateway |

---

## GraphQL Gateway

The primary query interface. Define domain-specific schemas with typed queries.

### Example Schema

```graphql
type Query {
  # Single entity by ID
  drone(id: ID!): Drone

  # List queries
  allDrones(limit: Int, status: DroneStatus): [Drone!]!

  # Relationship queries
  dronesInFleet(fleetID: ID!): [Drone!]!

  # Search
  searchDrones(query: String!, limit: Int): [Drone!]!
}

type Drone {
  id: ID!
  status: DroneStatus!
  batteryLevel: Float!
  location: Location!
  fleet: Fleet
  sensors: [Sensor!]
}
```

### Query Examples

**Single entity:**

```graphql
query {
  drone(id: "acme.telemetry.robotics.gcs1.drone.001") {
    id
    status
    batteryLevel
    fleet {
      id
      name
    }
  }
}
```

**With filters:**

```graphql
query {
  allDrones(status: ACTIVE, limit: 10) {
    id
    batteryLevel
    location {
      latitude
      longitude
    }
  }
}
```

**Relationship traversal:**

```graphql
query {
  dronesInFleet(fleetID: "acme.ops.logistics.hq.fleet.rescue") {
    id
    status
    batteryLevel
  }
}
```

### GraphQL Configuration

```json
{
  "services": {
    "graphql-gateway": {
      "enabled": true,
      "config": {
        "schema_path": "./schemas/robotics.graphql",
        "http_port": 8080,
        "playground_enabled": true,
        "queries": {
          "drone": {
            "resolver": "QueryEntityByID",
            "subject": "graph.drone.get"
          },
          "allDrones": {
            "resolver": "QueryEntitiesByType",
            "entity_type": "robotics.drone"
          }
        }
      }
    }
  }
}
```

---

## NATS Request/Reply

For programmatic access within services.

### Query by ID

```go
// Request
request := QueryRequest{
    EntityID: "acme.telemetry.robotics.gcs1.drone.001",
}

// Send to graph processor
response, err := nc.Request("graph.entity.get", requestBytes, 5*time.Second)
```

### Query by Type

```go
request := QueryRequest{
    EntityType: "robotics.drone",
    Limit:      100,
}

response, err := nc.Request("graph.entity.query", requestBytes, 5*time.Second)
```

### Query by Predicate

```go
request := PredicateQueryRequest{
    Predicate: "ops.fleet.member_of",
    Object:    "acme.ops.logistics.hq.fleet.rescue",
}

response, err := nc.Request("graph.predicate.query", requestBytes, 5*time.Second)
```

---

## Semantic Search

Natural language queries via embeddings.

**Requires embedding enabled:**

```json
{
  "graph": {
    "config": {
      "indexer": {
        "embedding": {
          "enabled": true,
          "service_url": "http://semembed:8081"
        }
      }
    }
  }
}
```

**Gateway endpoint:**

```bash
curl -X POST http://localhost:8080/search/semantic \
  -H "Content-Type: application/json" \
  -d '{
    "query": "drones with low battery in rescue fleet",
    "limit": 10,
    "entity_types": ["robotics.drone"]
  }'
```

**Response:**

```json
{
  "results": [
    {
      "entity_id": "acme.telemetry.robotics.gcs1.drone.002",
      "score": 0.92,
      "entity": {
        "node": {
          "id": "acme.telemetry.robotics.gcs1.drone.002",
          "type": "robotics.drone",
          "properties": {
            "robotics.battery.level": 15.4
          }
        }
      }
    }
  ]
}
```

See: [SemEmbed](../semantic/02-semembed.md) for embedding configuration.

---

## Index-Based Queries

Queries leverage NATS KV indexes for performance.

### PREDICATE_INDEX

Query entities by predicate value:

```text
Key: "ops.fleet.member_of:acme.ops.logistics.hq.fleet.rescue"
Value: ["drone.001", "drone.002", "drone.003"]
```

**Use case:** Find all entities with a specific predicate value.

### INCOMING_INDEX

Query reverse relationships (incoming edges):

```text
Key: "acme.ops.logistics.hq.fleet.rescue"
Value: ["drone.001", "drone.002", "drone.003"]
```

**Use case:** Find all entities pointing TO this entity.

**Enable incoming index:**

```json
{
  "graph": {
    "config": {
      "indexer": {
        "indexes": {
          "incoming": true
        }
      }
    }
  }
}
```

### SPATIAL_INDEX

Query by geographic location:

```json
{
  "graph": {
    "config": {
      "indexer": {
        "indexes": {
          "spatial": true
        }
      }
    }
  }
}
```

### TEMPORAL_INDEX

Query by time range:

```json
{
  "graph": {
    "config": {
      "indexer": {
        "indexes": {
          "temporal": true
        }
      }
    }
  }
}
```

See: [Indexing](04-indexing.md) for index configuration details.

---

## PathRAG Traversal

Multi-hop relationship traversal.

**Example:** Find sensors on drones in a fleet (2 hops)

```text
fleet.rescue ←member_of← drone.001 →has_component→ sensor.42
```

**GraphQL:**

```graphql
query {
  fleet(id: "acme.ops.logistics.hq.fleet.rescue") {
    drones {
      id
      sensors {
        id
        type
        lastReading {
          value
          timestamp
        }
      }
    }
  }
}
```

See: [Query Fundamentals](../advanced/01-query-fundamentals.md) for PathRAG vs GraphRAG decisions.

---

## Query Performance

### Use Indexes

Enable appropriate indexes for your query patterns:

```json
{
  "indexer": {
    "indexes": {
      "predicate": true,    // For predicate-based queries
      "incoming": true,     // For reverse relationship queries
      "spatial": true,      // For geo queries
      "temporal": true      // For time-range queries
    }
  }
}
```

### Limit Results

Always specify limits to avoid unbounded queries:

```graphql
query {
  allDrones(limit: 100) {
    id
  }
}
```

### Use Specific Predicates

Query specific predicates rather than scanning all entities:

```go
// Good: Query by predicate
request := PredicateQueryRequest{
    Predicate: "ops.fleet.member_of",
    Object:    fleetID,
}

// Avoid: Scan all entities and filter
```

### Cache Considerations

Query results are not cached by default. Enable caching for read-heavy workloads:

```json
{
  "graph": {
    "config": {
      "query_cache": {
        "enabled": true,
        "ttl": "30s",
        "max_entries": 10000
      }
    }
  }
}
```

See: [Performance Tuning](../advanced/07-performance-tuning.md) for optimization details.

---

## Next Steps

- **[Indexing](04-indexing.md)** - Configure indexes for query performance
- **[Query Fundamentals](../advanced/01-query-fundamentals.md)** - PathRAG vs GraphRAG decisions
- **[Performance Tuning](../advanced/07-performance-tuning.md)** - Optimize queries

---

**Key Takeaway:** Use GraphQL for interactive queries, NATS for programmatic access, and semantic search for natural language. Enable appropriate indexes for your query patterns.
