# Graph Indexing

**Index types, configuration, and when to use each**

---

## Index Overview

Indexes accelerate specific query patterns. Enable only what you need.

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

---

## Index Types

### Predicate Index

**Enables:** Find all entities with relationship type X

```bash
curl "http://localhost:8080/graph/query?predicate=belongs_to"
```

**Memory:** ~50 bytes/entity
**Recommendation:** Always enable (minimal cost, common queries)

---

### Incoming Index

**Enables:** Reverse relationship lookups

```bash
curl "http://localhost:8080/graph/query?target=fleet-rescue&predicate=belongs_to"
```

**Memory:** ~50 bytes/entity
**Recommendation:** Always enable (critical for graph traversal)

---

### Alias Index

**Enables:** Query by entity name

```bash
curl "http://localhost:8080/graph/entity/rescue-alpha"  # Resolves to UAV-001
```

**Memory:** ~30 bytes/entity
**Recommendation:** Enable if using human-readable names

---

### Spatial Index

**Enables:** Geospatial queries

```bash
curl "http://localhost:8080/graph/query?lat=37.7&lon=-122.4&radius_km=10"
```

**Memory:** ~40 bytes/entity
**Recommendation:** Enable only if querying by location

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

---

### Temporal Index

**Enables:** Time-based queries

```bash
curl "http://localhost:8080/graph/query?created_after=2024-01-15T00:00:00Z"
```

**Memory:** ~40 bytes/entity
**Recommendation:** Enable for event-driven systems

---

## Memory Impact

| Configuration | Memory/Entity | 1M Entities |
|---------------|---------------|-------------|
| Predicate + Incoming | ~100 bytes | 100 MB |
| + Alias | ~130 bytes | 130 MB |
| + Spatial | ~170 bytes | 170 MB |
| + Temporal | ~210 bytes | 210 MB |

**Embeddings add ~1.5KB/entity** (see [Semantic](../semantic/01-overview.md))

---

## Performance Characteristics

| Index | Build Time | Query Time | Use Case |
|-------|------------|------------|----------|
| Predicate | O(E) | O(1) | "Find all X relationships" |
| Incoming | O(E) | O(1) | "What points to this entity?" |
| Alias | O(N) | O(1) | "Find entity by name" |
| Spatial | O(N log N) | O(log N + K) | "Find nearby entities" |
| Temporal | O(N) | O(K) | "Find entities in time range" |

---

## Best Practices

### Start Minimal

```json
{
  "indexes": {
    "predicate": true,
    "incoming": true
  }
}
```

Add others only when needed.

### Monitor Index Usage

```bash
curl http://localhost:8080/metrics | grep index_queries
```

Disable unused indexes to save memory.

---

## Next Steps

- **[Performance Tuning](../advanced/02-performance-tuning.md)** - Optimize index configuration
- **[PathRAG Decisions](../advanced/01-pathrag-graphrag-decisions.md)** - Choose right indexes
- **[Architecture Deep Dive](../advanced/03-architecture-deep-dive.md)** - How indexes work

---

**Enable only the indexes you need** - Each index has a memory cost. Start minimal, add as required.
