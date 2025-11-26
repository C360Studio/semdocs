# Graph Indexing

Index types, configuration, and when to use each

---

## Index Overview

Indexes accelerate specific query patterns. Each index is a separate NATS KV bucket.

```json
{
  "graph": {
    "config": {
      "indexer": {
        "indexes": {
          "predicate": true,     // Relationship type queries
          "incoming": true,      // Reverse relationship queries
          "alias": false,        // Entity name resolution
          "spatial": false,      // Geospatial queries
          "temporal": false      // Time-based queries
        }
      }
    }
  }
}
```

**NATS KV Buckets:**

| Index | Bucket Name | Purpose |
|-------|-------------|---------|
| Entity States | `ENTITY_STATES` | Primary entity storage |
| Predicate | `PREDICATE_INDEX` | Predicate → entities lookup |
| Incoming | `INCOMING_INDEX` | Target → source entities |
| Alias | `ALIAS_INDEX` | Alias → entity ID |
| Spatial | `SPATIAL_INDEX` | Geohash → entities |
| Temporal | `TEMPORAL_INDEX` | Time bucket → entities |
| Community | `COMMUNITY_INDEX` | Community detection results |

---

## Index Types

### ENTITY_STATES (Primary Storage)

All entities stored by 6-part ID.

**Key:** `acme.telemetry.robotics.gcs1.drone.001`
**Value:** EntityState JSON

**Always enabled** - This is primary storage, not optional.

---

### PREDICATE_INDEX

**Enables:** Find all entities with a specific predicate value.

**Structure:**

```text
Key: "ops.fleet.member_of:acme.ops.logistics.hq.fleet.rescue"
Value: ["acme.telemetry.robotics.gcs1.drone.001", "acme.telemetry.robotics.gcs1.drone.002"]
```

**Query pattern:** "Find all drones in rescue fleet"

**Memory:** ~50 bytes/entity
**Recommendation:** Always enable (minimal cost, common queries)

---

### INCOMING_INDEX

**Enables:** Reverse relationship lookups (what points TO this entity).

**Structure:**

```text
Key: "acme.ops.logistics.hq.fleet.rescue"
Value: ["acme.telemetry.robotics.gcs1.drone.001", "acme.telemetry.robotics.gcs1.drone.002"]
```

**Query pattern:** "Find all entities that reference this fleet"

**Memory:** ~50 bytes/entity
**Recommendation:** Always enable (critical for graph traversal)

---

### ALIAS_INDEX

**Enables:** Query by human-readable names.

**Structure:**

```text
Key: "rescue-alpha"
Value: "acme.telemetry.robotics.gcs1.drone.001"
```

**Source:** From triples with predicates registered as aliases in vocabulary:

```go
{Subject: entityID, Predicate: "robotics.communication.callsign", Object: "rescue-alpha"}
```

Predicates must be registered with `vocabulary.WithAlias()` to be indexed. See [Vocabulary Registry](../basics/06-vocabulary-registry.md).

**Query pattern:** "Resolve alias to entity ID"

**Memory:** ~30 bytes/alias
**Recommendation:** Enable if using human-readable names

---

### SPATIAL_INDEX

**Enables:** Geospatial queries.

**Structure:**

```text
Key: "9q8yy6"  // geohash
Value: ["acme.telemetry.robotics.gcs1.drone.001", "acme.telemetry.robotics.gcs1.drone.002"]
```

**Source:** From entity Position field or geo predicates.

**Query pattern:** "Find entities within radius of coordinates"

**Configuration:**

```json
{
  "indexer": {
    "indexes": {
      "spatial": true
    },
    "spatial_config": {
      "geohash_precision": 6
    }
  }
}
```

**Memory:** ~40 bytes/entity
**Recommendation:** Enable only if querying by location

---

### TEMPORAL_INDEX

**Enables:** Time-based queries.

**Structure:**

```text
Key: "2024-01-15T10:00"  // time bucket
Value: ["acme.telemetry.robotics.gcs1.drone.001"]
```

**Query pattern:** "Find entities updated in time range"

**Configuration:**

```json
{
  "indexer": {
    "indexes": {
      "temporal": true
    },
    "temporal_config": {
      "bucket_duration": "1m"
    }
  }
}
```

**Memory:** ~40 bytes/entity
**Recommendation:** Enable for event-driven systems

---

### COMMUNITY_INDEX

**Enables:** GraphRAG community-based search.

**Structure:**

```text
Key: "graph.community.0.comm-0-auth"
Value: {"id": "comm-0-auth", "level": 0, "members": ["entity1", "entity2"], "summary": "..."}

Key: "graph.community.entity.0.acme.telemetry.robotics.gcs1.drone.001"
Value: "comm-0-auth"
```

**Source:** Computed by Label Propagation Algorithm (LPA) during community detection.

**Query pattern:** "Find entities semantically similar to this one" (via community membership)

**Key patterns:**

| Pattern | Purpose |
|---------|---------|
| `graph.community.{level}.{id}` | Community data (members, summary, keywords) |
| `graph.community.entity.{level}.{entityID}` | Entity → Community mapping |

**Memory:** Variable based on community count and summaries
**Recommendation:** Enable if using GraphRAG semantic search

**Note:** Community membership is stored as a separate index, not as triples on entities. See [Query Fundamentals](../advanced/01-query-fundamentals.md) for GraphRAG details.

**Architecture Note:** Unlike other indexes (managed by IndexManager), COMMUNITY_INDEX is managed by the `graphclustering` package. This is a known inconsistency being reviewed.

---

## Memory Impact

| Configuration | Memory/Entity | 1M Entities |
|---------------|---------------|-------------|
| Entity States only | ~500 bytes | 500 MB |
| + Predicate + Incoming | ~600 bytes | 600 MB |
| + Alias | ~630 bytes | 630 MB |
| + Spatial | ~670 bytes | 670 MB |
| + Temporal | ~710 bytes | 710 MB |

**Embeddings add ~1.5KB/entity** (see [Semantic Overview](../semantic/01-overview.md))

---

## Performance Characteristics

| Index | Build Time | Query Time | Use Case |
|-------|------------|------------|----------|
| Predicate | O(T) | O(1) | "Find by predicate value" |
| Incoming | O(T) | O(1) | "What points to this entity?" |
| Alias | O(A) | O(1) | "Resolve alias to entity" |
| Spatial | O(N) | O(log N + K) | "Find nearby entities" |
| Temporal | O(N) | O(K) | "Find in time range" |

Where:

- **T** = number of triples
- **N** = number of entities
- **A** = number of aliases
- **K** = result set size

---

## Configuration Example

**Minimal (recommended start):**

```json
{
  "graph": {
    "config": {
      "indexer": {
        "indexes": {
          "predicate": true,
          "incoming": true,
          "alias": false,
          "spatial": false,
          "temporal": false
        }
      }
    }
  }
}
```

**Full indexing:**

```json
{
  "graph": {
    "config": {
      "indexer": {
        "indexes": {
          "predicate": true,
          "incoming": true,
          "alias": true,
          "spatial": true,
          "temporal": true
        },
        "spatial_config": {
          "geohash_precision": 6
        },
        "temporal_config": {
          "bucket_duration": "1m"
        }
      }
    }
  }
}
```

---

## Best Practices

### Start Minimal

Enable only predicate and incoming indexes initially:

```json
{
  "indexes": {
    "predicate": true,
    "incoming": true
  }
}
```

Add others when you have specific query patterns that need them.

### Monitor Index Usage

```bash
curl http://localhost:9090/metrics | grep index
```

**Key metrics:**

- `index_queries_total` - Index query count
- `index_query_duration_seconds` - Query latency
- `index_entries_total` - Index size

Disable unused indexes to save memory.

### Consider Query Patterns

| Query Pattern | Required Index |
|---------------|----------------|
| "Find by predicate value" | PREDICATE_INDEX |
| "What references this entity?" | INCOMING_INDEX |
| "Resolve alias to ID" | ALIAS_INDEX |
| "Find nearby entities" | SPATIAL_INDEX |
| "Find in time range" | TEMPORAL_INDEX |

---

## Next Steps

- **[Queries](03-queries.md)** - Query patterns using indexes
- **[Performance Tuning](../advanced/07-performance-tuning.md)** - Optimize index configuration
- **[Query Fundamentals](../advanced/01-query-fundamentals.md)** - PathRAG index usage

---

**Key Takeaway:** Each index is a NATS KV bucket. Start with predicate + incoming, add others based on query patterns. Monitor usage and disable unused indexes.
