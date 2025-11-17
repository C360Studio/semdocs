# PathRAG: Graph Traversal for Contextual Retrieval

**Status**: Production-ready (since v0.6.0)
**Package**: `graph/query`
**HTTP Endpoint**: `POST /entity/:id/path`

---

## Not Sure If This Is the Right Tool?

**PathRAG is for tracing RELATIONSHIPS** (dependencies, connections, impact chains).

**If you need...**

- **Semantically similar entities** → Use [GraphRAG](graphrag.md) instead
- **System-wide pattern search** → Use [GraphRAG Global Search](graphrag.md#global-vs-local-search)
- **Both relationships AND similarity** → See [Hybrid Query Patterns](hybrid-queries.md)
- **Help choosing** → Read [Choosing Your Query Strategy](choosing-query-strategy.md)

---

## Overview

PathRAG (Path-based Retrieval-Augmented Generation) extends traditional semantic search by traversing entity relationships to discover contextually relevant information. Instead of relying solely on vector similarity, PathRAG explores the graph structure to find entities connected through meaningful relationships.

**Key Insight**: Sometimes the most relevant information isn't semantically similar to your query—it's structurally connected to entities that are.

## SemStreams Implementation

SemStreams implements PathRAG through the `PathQuery` API with production-grade resource protection:

```go
type PathQuery struct {
    StartEntity  string        // Entity to start traversal from
    MaxDepth     int          // Maximum hops from start (prevents infinite loops)
    MaxNodes     int          // Maximum nodes to visit (bounds memory)
    MaxTime      time.Duration // Query timeout (prevents runaway queries)
    EdgeFilter   []string     // Filter by relationship types
    DecayFactor  float64      // Relevance decay per hop (0.0-1.0)
    MaxPaths     int          // Limit paths tracked (prevents exponential growth)
}

type PathResult struct {
    Entities  []*EntityState      // All visited entities
    Paths     [][]string          // Sequences of entity IDs
    Scores    map[string]float64  // Relevance scores (with decay)
    Truncated bool                // Hit resource limits?
}
```

### Design Philosophy

**Bounded Exploration**: Unlike academic GraphRAG implementations that assume unlimited resources, SemStreams PathRAG is designed for edge deployment:

1. **MaxDepth**: Prevents infinite loops in cyclic graphs
2. **MaxNodes**: Bounds memory usage regardless of graph density
3. **MaxTime**: Ensures predictable latency for real-time queries
4. **MaxPaths**: Prevents exponential path growth in dense graphs
5. **EdgeFilter**: Selective traversal reduces irrelevant exploration

## Comparison to Microsoft GraphRAG

| Feature | Microsoft GraphRAG | SemStreams PathRAG |
|---------|-------------------|-------------------|
| **Traversal Strategy** | Local/Global/DRIFT modes | Bounded multi-hop with filters |
| **Resource Protection** | None (academic) | MaxDepth, MaxNodes, MaxTime, MaxPaths |
| **Edge Filtering** | Not mentioned | Explicit EdgeFilter per query |
| **Relevance Scoring** | Community-based | Distance-based decay (configurable) |
| **Deployment Target** | Cloud (unlimited resources) | Edge devices (resource-constrained) |
| **Graph Construction** | LLM-powered extraction | Schema-driven (deterministic) |
| **Real-time** | Batch processing | Streaming event-driven |

**Key Difference**: Microsoft GraphRAG relies on pre-computed community hierarchies for global queries. SemStreams uses on-demand bounded traversal, trading some query expressiveness for predictable resource usage.

## Use Cases

### 1. Dependency Chain Traversal

**Scenario**: Find all services affected by a configuration change

```json
{
  "start_entity": "config.database.credentials",
  "max_depth": 5,
  "edge_filter": ["depends_on"],
  "decay_factor": 0.9,
  "max_nodes": 100,
  "max_time": "500ms"
}
```

**Result**: Discovers:
- `api-service` (depth 1, score 0.9)
- `auth-service` (depth 1, score 0.9)
- `dashboard-frontend` (depth 2, score 0.81)
- `mobile-app` (depth 3, score 0.73)

### 2. Spatial Proximity Network

**Scenario**: Find all drones in communication range of a base station

```json
{
  "start_entity": "base.alpha",
  "max_depth": 3,
  "edge_filter": ["near", "communicates"],
  "decay_factor": 0.8,
  "max_nodes": 50
}
```

**Result**: Discovers:
- Direct: Drones within radio range (depth 1)
- Relay: Drones reachable via mesh network (depth 2-3)
- Scores: Higher for closer/fewer hops

### 3. Incident Impact Analysis

**Scenario**: Trace cascading effects of a system failure

```json
{
  "start_entity": "alert.network.outage.region-west",
  "max_depth": 4,
  "edge_filter": ["triggers", "affects", "depends_on"],
  "decay_factor": 0.85,
  "max_paths": 20
}
```

**Result**: Discovers:
- Immediately affected services (depth 1)
- Secondary failures (depth 2)
- User impact chains (depth 3-4)
- Paths limited to 20 most relevant

### 4. Knowledge Graph Exploration

**Scenario**: Discover related documentation and examples

```json
{
  "start_entity": "concept.graph-processor",
  "max_depth": 3,
  "edge_filter": ["references", "related_to", "example_of"],
  "decay_factor": 0.9,
  "max_nodes": 200
}
```

**Result**: Discovers:
- API documentation (depth 1)
- Usage examples (depth 2)
- Related concepts (depth 2-3)

## Resource Limits Explained

### Why MaxDepth?

**Problem**: Cyclic graphs (A→B→C→A) cause infinite traversal
**Solution**: Hard limit on hop count

**Tuning**:
- **Depth 2-3**: Local neighborhood exploration
- **Depth 4-5**: Medium-range dependency chains
- **Depth 6+**: Rarely needed; consider graph redesign

### Why MaxNodes?

**Problem**: Dense graphs cause memory explosion
**Solution**: Cap total nodes visited regardless of depth

**Tuning**:
- **50-100 nodes**: Quick queries, tight memory budget
- **200-500 nodes**: Thorough exploration
- **1000+ nodes**: Heavy queries; ensure adequate resources

### Why MaxTime?

**Problem**: Complex queries block real-time responses
**Solution**: Timeout ensures predictable latency

**Tuning**:
- **50-100ms**: Real-time user-facing queries
- **500ms-1s**: Background analysis
- **5s+**: Batch jobs only

### Why MaxPaths?

**Problem**: Exponential path growth (N nodes → N! paths)
**Solution**: Track only top-K paths by relevance

**Example**: Graph with 10 entities, depth 5:
- Without limit: Potentially 10^5 = 100,000 paths
- With MaxPaths=20: Track 20 highest-scoring paths

### Why DecayFactor?

**Problem**: Distant entities may be irrelevant
**Solution**: Exponential relevance decay with distance

**Formula**: `score = initial_score × (DecayFactor ^ depth)`

**Tuning**:
- **0.9**: Gentle decay (depth 5 → 59% relevance)
- **0.8**: Moderate decay (depth 5 → 33% relevance)
- **0.7**: Aggressive decay (depth 5 → 17% relevance)

## Performance Characteristics

### Latency by Graph Complexity

**Note**: The following table shows **projected performance estimates** based on theoretical analysis. See `graph/query/path_benchmark_test.go` for benchmark tests to validate these estimates.

| Graph Size | Depth | Nodes Visited | P50 | P95 | P99 |
|------------|-------|---------------|-----|-----|-----|
| 100 entities | 2 | 15 | 5ms | 12ms | 20ms |
| 100 entities | 3 | 40 | 15ms | 35ms | 60ms |
| 1000 entities | 2 | 20 | 8ms | 18ms | 30ms |
| 1000 entities | 3 | 80 | 25ms | 60ms | 100ms |
| 10000 entities | 2 | 25 | 12ms | 25ms | 45ms |
| 10000 entities | 3 | 150 | 50ms | 120ms | 200ms |

**Run benchmarks**:
```bash
# Run all PathRAG benchmarks
go test -bench=BenchmarkPathQuery -benchmem -benchtime=10s ./graph/query/

# Run specific graph size
go test -bench=BenchmarkPathQuery_MediumGraph -benchmem ./graph/query/

# Compare performance across changes
go test -bench=. -count=10 ./graph/query/ > baseline.txt
# (make changes)
go test -bench=. -count=10 ./graph/query/ > optimized.txt
benchstat baseline.txt optimized.txt
```

**Actual performance depends on**:
- Entity size (properties, triple count)
- NATS KV latency
- Cache hit rates
- Edge density
- Relationship fanout (edges per entity)

### Memory Usage

**Per-Query Overhead**:
- Entity cache: ~2KB per entity visited
- Path tracking: ~100 bytes per path
- Score map: ~50 bytes per entity

**Example**: 100 entities, 20 paths
- Memory: (100 × 2KB) + (20 × 100B) + (100 × 50B) ≈ 207KB

### Scalability Patterns

**Horizontal**: PathQuery is stateless; scales with query load

**Vertical**: Memory-bound by MaxNodes setting

**Recommendation**: Use MaxTime as primary throttle for unpredictable queries

## Query Patterns

### Pattern 1: Fixed-Depth Exploration

**Use Case**: "Show me all entities within N hops"

```go
query := PathQuery{
    StartEntity: "entity-123",
    MaxDepth:    3,           // Fixed depth
    MaxNodes:    1000,        // Large enough to not truncate
    MaxTime:     1 * time.Second,
    DecayFactor: 1.0,         // No decay (all equal relevance)
}
```

### Pattern 2: Filtered Relationship Walk

**Use Case**: "Follow only specific relationship types"

```go
query := PathQuery{
    StartEntity: "service-api",
    MaxDepth:    5,
    EdgeFilter:  []string{"depends_on", "calls"},  // Only these edges
    MaxNodes:    200,
    DecayFactor: 0.85,
}
```

### Pattern 3: Time-Bounded Discovery

**Use Case**: "Find as much as possible in 100ms"

```go
query := PathQuery{
    StartEntity: "alert-001",
    MaxDepth:    10,          // High depth
    MaxNodes:    10000,       // High limit
    MaxTime:     100 * time.Millisecond,  // Primary constraint
    DecayFactor: 0.8,
}
```

### Pattern 4: Top-K Path Ranking

**Use Case**: "Show me the 10 most relevant paths"

```go
query := PathQuery{
    StartEntity: "concept-graphrag",
    MaxDepth:    4,
    MaxNodes:    500,
    MaxPaths:    10,          // Only track 10 best paths
    DecayFactor: 0.9,
}
```

## Integration with Semantic Search

### Hybrid Query: Semantic + Path

**Workflow**:

1. **Semantic Search**: Find initial relevant entities
   ```json
   POST /search/semantic
   {
     "query": "drone battery failure",
     "threshold": 0.3,
     "limit": 5
   }
   ```

2. **Path Expansion**: Explore from top result
   ```json
   POST /entity/drone-001/path
   {
     "max_depth": 3,
     "edge_filter": ["related_to", "triggered_by"]
   }
   ```

3. **Result Fusion**: Combine semantic scores + path scores
   ```
   Final Score = (semantic_score × 0.6) + (path_score × 0.4)
   ```

### When to Use Each

| Query Type | Use Semantic Search | Use PathRAG |
|------------|-------------------|-------------|
| **Keyword match** | ✅ Excellent | ❌ Poor |
| **Conceptual similarity** | ✅ Excellent | ⚠️ Depends on graph |
| **Relationship discovery** | ❌ Cannot do | ✅ Excellent |
| **Dependency analysis** | ❌ Cannot do | ✅ Excellent |
| **Impact chains** | ⚠️ Indirect only | ✅ Direct |
| **Real-time queries** | ✅ Fast (1-10ms) | ⚠️ Slower (10-200ms) |

## Configuration

### NATS Subject

PathQuery is exposed via NATS request/reply:

```go
// Subscribe to path queries
nc.Subscribe("graph.query.path", func(msg *nats.Msg) {
    var query PathQuery
    json.Unmarshal(msg.Data, &query)

    result, err := queryClient.ExecutePathQuery(ctx, query)

    json.Marshal(result)
    msg.Respond(resultJSON)
})
```

### HTTP Gateway Route

Configured in gateway component:

```json
{
  "path": "/entity/:id/path",
  "method": "POST",
  "nats_subject": "graph.query.path",
  "timeout": "10s",
  "description": "Traverse graph path from entity"
}
```

**Example Request**:

```bash
curl -X POST http://localhost:8080/api-gateway/entity/drone-001/path \
  -H "Content-Type: application/json" \
  -d '{
    "max_depth": 3,
    "max_nodes": 100,
    "max_time": "500ms",
    "edge_filter": ["near", "communicates"],
    "decay_factor": 0.8,
    "max_paths": 10
  }'
```

**Example Response**:

```json
{
  "entities": [
    {
      "entity_id": "drone-001",
      "entity_type": "robotics.drone",
      "properties": {...}
    },
    {
      "entity_id": "base-alpha",
      "entity_type": "robotics.base",
      "properties": {...}
    }
  ],
  "paths": [
    ["drone-001", "relay-003", "base-alpha"],
    ["drone-001", "drone-002", "base-alpha"]
  ],
  "scores": {
    "drone-001": 1.0,
    "relay-003": 0.8,
    "drone-002": 0.8,
    "base-alpha": 0.64
  },
  "truncated": false
}
```

## Best Practices

### 1. Start with Tight Limits

```go
// ✅ GOOD: Conservative defaults
query := PathQuery{
    MaxDepth:    2,
    MaxNodes:    50,
    MaxTime:     100 * time.Millisecond,
    DecayFactor: 0.8,
}

// ❌ BAD: Unbounded query
query := PathQuery{
    MaxDepth:    10,
    MaxNodes:    100000,
    MaxTime:     30 * time.Second,
}
```

### 2. Use EdgeFilter Aggressively

```go
// ✅ GOOD: Selective traversal
EdgeFilter: []string{"depends_on", "calls"}

// ❌ BAD: Traverse all edges
EdgeFilter: nil  // Explores ALL relationship types
```

### 3. Monitor Truncation

```go
result, err := client.ExecutePathQuery(ctx, query)
if result.Truncated {
    log.Warn("Query hit resource limits - results incomplete",
        "start_entity", query.StartEntity,
        "max_depth", query.MaxDepth)
    // Consider increasing limits or filtering differently
}
```

### 4. Tune DecayFactor for Use Case

```go
// Strong locality (nearby entities most relevant)
DecayFactor: 0.7

// Moderate (balance local vs distant)
DecayFactor: 0.85

// Weak decay (distant entities still relevant)
DecayFactor: 0.95
```

### 5. Combine with Caching

```go
// Query client uses LRU/TTL cache
config := &query.Config{
    EntityCache: cache.Config{
        Strategy: cache.StrategyHybrid,
        MaxSize:  1000,
        TTL:      5 * time.Minute,
    },
}
client, _ := query.NewClient(natsClient, config)

// Repeated path queries benefit from cached entities
```

## Troubleshooting

### Query Returns Empty Results

**Possible Causes**:
1. StartEntity has no outgoing edges
2. EdgeFilter too restrictive
3. MaxDepth too small
4. MaxNodes too small (truncated before finding results)

**Debug**:
```go
// Remove filters temporarily
query.EdgeFilter = nil
query.MaxDepth = 5
query.MaxNodes = 500

result, _ := client.ExecutePathQuery(ctx, query)
log.Info("Debug traversal", "entities_found", len(result.Entities))
```

### Query Times Out

**Possible Causes**:
1. Graph too dense (exponential explosion)
2. MaxTime too aggressive
3. Slow NATS KV responses

**Solutions**:
- Reduce MaxDepth (each hop multiplies work)
- Add EdgeFilter to prune exploration
- Increase MaxTime or reduce MaxNodes
- Check NATS KV latency metrics

### Truncated Results

**Meaning**: Query hit MaxNodes, MaxTime, or MaxPaths limit before completing

**Solutions**:
1. Increase the limiting factor (check which triggered)
2. Use tighter EdgeFilter to focus search
3. Reduce MaxDepth if deep traversal not needed
4. Accept truncation for approximate results

### High Memory Usage

**Cause**: Large MaxNodes or dense graphs

**Solutions**:
- Reduce MaxNodes to bound memory
- Use MaxTime as primary limit (frees memory on timeout)
- Clear query client cache periodically
- Monitor `query.GetCacheStats()` for cache size

## Future Enhancements

**With Community Detection** (Tier 3):
- Global PathRAG: Start from community representatives
- Multi-source PathRAG: Parallel traversal from multiple entities
- Community-bounded PathRAG: Limit exploration to community boundaries

**With Rank Fusion** (Tier 2):
- Hybrid scoring: Semantic + Path + BM25
- Multi-strategy queries: Try semantic first, expand with path on low recall

## Related Documentation

- [Graph Query Library](../../graph/query/README.md) - API documentation
- [Semantic Search](../EMBEDDING_ARCHITECTURE.md) - Vector similarity search
- [HTTP Gateway Usage](../../configs/HTTP_GATEWAY_USAGE.md) - REST API examples
- [GraphRAG Lessons Learned](../architecture/GRAPHRAG_LESSONS_LEARNED.md) - Research comparison

---

**Implementation**: `graph/query/client.go:ExecutePathQuery()`
**Tests**: `graph/query/client_test.go:TestExecutePathQuery()`
**HTTP Route**: `POST /entity/:id/path`
**NATS Subject**: `graph.query.path`
