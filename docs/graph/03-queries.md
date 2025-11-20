# Graph Queries

**Querying entities and relationships**

---

## Query Methods

- HTTP REST API
- NATS request/reply
- GraphQL (optional)

---

## Entity Queries

### By ID

```bash
curl "http://localhost:8080/graph/entity/UAV-001"
```

### By Type

```bash
curl "http://localhost:8080/graph/query?type=drone"
```

### By Properties

```bash
curl "http://localhost:8080/graph/query?type=drone&battery.level.lte=20"
```

**Operators:** `eq`, `ne`, `gt`, `gte`, `lt`, `lte`, `contains`

---

## Relationship Queries

### Forward (outgoing)

```bash
curl "http://localhost:8080/graph/query?source=UAV-001&predicate=belongs_to"
```

### Reverse (incoming)

```bash
curl "http://localhost:8080/graph/query?target=fleet-rescue&predicate=belongs_to"
```

---

## Advanced Queries

### Multi-hop Traversal

```bash
curl -X POST http://localhost:8080/graph/traverse \
  -d '{"start":"fleet-rescue","hops":[{"predicate":"belongs_to","direction":"incoming"}]}'
```

### Geospatial Queries

```bash
curl "http://localhost:8080/graph/query?lat=37.7749&lon=-122.4194&radius_km=10"
```

Requires spatial index enabled.

### Temporal Queries

```bash
curl "http://localhost:8080/graph/query?created_after=2024-01-15T00:00:00Z"
```

Requires temporal index enabled.

---

## Semantic Search

```bash
curl -X POST http://localhost:8080/search/semantic \
  -d '{"query":"emergency battery situation","limit":10}'
```

Requires embedding enabled.

---

## Query Performance

- Enable caching for read-heavy workloads
- Use specific filters to narrow search space
- Limit result sets
- Enable appropriate indexes

See [Performance Tuning](../advanced/02-performance-tuning.md).

---

## Next Steps

- **[Indexing](04-indexing.md)** - Configure indexes for query performance
- **[Performance Tuning](../advanced/02-performance-tuning.md)** - Optimize queries
- **[PathRAG Decisions](../advanced/01-pathrag-graphrag-decisions.md)** - Query patterns guide
