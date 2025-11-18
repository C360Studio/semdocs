# Embedding Architecture

## Overview

SemStreams provides semantic search capabilities through a flexible embedding system that prioritizes **reliability** and **graceful degradation**. The architecture uses BM25 (pure Go lexical search) as the primary provider, with optional HTTP-based neural embeddings for enhanced semantic understanding.

## Design Philosophy

**BM25-Primary, Embeddings-Optional:**

- BM25 is the **recommended production default** (no external dependencies)
- HTTP embeddings (semembed, LocalAI, OpenAI) are opt-in enhancements
- Automatic fallback from HTTP to BM25 if service unavailable
- System remains operational regardless of embedding service status

**Rationale** (from GraphRAG research):

- **Deterministic > Probabilistic**: BM25 provides reliable, repeatable results
- **Fast**: 0.1-1ms latency vs 5-10ms for HTTP neural embeddings
- **Zero dependencies**: No external services to fail or manage
- **Production-proven**: Lexical search is well-understood and debuggable
- **Neural embeddings as enhancement**: Add HTTP provider only when semantic similarity improves results

### When to Use Each Provider

| Use Case | Recommended Provider | Rationale |
|----------|---------------------|-----------|
| **Production deployments** | BM25 | Reliable, fast, zero dependencies |
| **Exact keyword matching** | BM25 | Lexical search excels at exact terms |
| **Low-resource environments** | BM25 | No GPU, minimal memory overhead |
| **Semantic similarity queries** | HTTP (optional) | Neural embeddings understand context |
| **Multi-language support** | HTTP (optional) | Multilingual embedding models |
| **Development/testing** | BM25 | Fast iteration, no service setup |

## Provider Architecture

### Three Provider Modes

```go
// Configuration (processor/graph/indexmanager/config.go)
type EmbeddingConfig struct {
    Enabled  bool   `json:"enabled"`  // Default: true (using BM25)
    Provider string `json:"provider"` // "bm25", "http", or "disabled"

    // HTTP-specific config (optional)
    HTTPEndpoint string `json:"http_endpoint"` // e.g., "http://semembed:8081/v1/embeddings"
    HTTPModel    string `json:"http_model"`    // e.g., "BAAI/bge-small-en-v1.5"

    // Shared config
    TextFields      []string      `json:"text_fields"`
    RetentionWindow time.Duration `json:"retention_window"` // Default: 24h
    CacheBucket     string        `json:"cache_bucket"`     // Optional NATS KV cache
}
```

**1. BM25 Provider (Default)**

- Pure Go implementation, no external service needed
- 384-dimensional vectors for compatibility
- Standard BM25 parameters (k1=1.5, b=0.75)
- Always available, predictable performance

**2. HTTP Provider (Optional)**

- Neural embeddings via HTTP API (semembed, LocalAI, OpenAI)
- OpenAI-compatible `/v1/embeddings` endpoint
- Automatic BM25 fallback if service unreachable
- Tested on startup with connectivity check

**3. Disabled**

- Semantic search completely disabled
- No embedding generation or storage
- Minimal memory footprint

### Provider Selection Logic

```go
// Initialization (processor/graph/indexmanager/semantic.go:initializeSemanticSearch)

if !config.Embedding.Enabled || config.Embedding.Provider == "disabled" {
    // Semantic search disabled
    return nil
}

provider := config.Embedding.Provider
if provider == "" {
    provider = "bm25" // Default to BM25
}

switch provider {
case "http":
    // Try HTTP, fallback to BM25 if unreachable
    embedder, err := embedding.NewHTTPEmbedder(httpConfig)
    if err != nil {
        return err
    }

    // Test connectivity
    _, testErr := embedder.Generate(ctx, []string{"connectivity test"})
    if testErr != nil {
        log.Warn("HTTP service unavailable - falling back to BM25")
        embedder = embedding.NewBM25Embedder(bm25Config)
        metrics.embeddingFallbacks.Inc() // Track fallback event
    }

case "bm25":
    embedder = embedding.NewBM25Embedder(bm25Config)
}
```

## Storage Architecture

### Two-Tier Caching System

**L1 Cache: In-Memory TTL Caches** (Fast, Volatile)

- **vectorCache**: Stores entity embeddings ([]float32)
- **metadataCache**: Stores entity metadata (EntityMetadata)
- Default retention: 24 hours (configurable via `retention_window`)
- Automatic eviction on expiry
- Lost on service restart

**L2 Cache: NATS KV Cache** (Persistent, Optional)

- Content-addressed storage (SHA-256 hash of text)
- Survives service restarts
- Enables deduplication across entities with identical text
- Configured via `cache_bucket` (e.g., "EMBEDDINGS_CACHE")

```text
┌─────────────────────────────────────────────────────────────┐
│  Entity Processing                                          │
│                                                             │
│  Entity → Extract Text → Generate Embedding                │
│              ↓              ↓                               │
│        Text Fields    Provider (BM25 or HTTP)             │
└─────────────────────────┬───────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│  L1 Cache (In-Memory TTL)                                   │
│  ┌─────────────────┐  ┌──────────────────┐                 │
│  │  vectorCache    │  │  metadataCache   │                 │
│  │  entityID → []  │  │  entityID →      │                 │
│  │  float32        │  │  EntityMetadata  │                 │
│  └─────────────────┘  └──────────────────┘                 │
│  Retention: 24h (default)                                   │
└─────────────────────────┬───────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│  L2 Cache (NATS KV - Optional)                              │
│  ┌──────────────────────────────────────────┐               │
│  │  EMBEDDINGS_CACHE bucket                 │               │
│  │  contentHash (SHA-256) → []float32       │               │
│  │  Content-addressed, persistent           │               │
│  └──────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────┘
```

### Storage Flow

```go
// Entity Embedding Generation (processor/graph/indexmanager/manager.go:generateEmbedding)

func (m *Manager) generateEmbedding(ctx context.Context, entityID string, entityState *EntityState) error {
    // 1. Extract text from entity properties
    text := m.extractText(entityState.Node.Properties)
    if text == "" {
        return nil // Skip entities with no text
    }

    // 2. Generate embedding via provider
    embeddings, err := m.embedder.Generate(ctx, []string{text})
    if err != nil {
        return err
    }
    vector := embeddings[0]

    // 3. Store in L1 cache (TTL-based eviction)
    m.vectorCache.Set(entityID, vector)

    // 4. Store metadata (for search result enrichment)
    metadata := &EntityMetadata{
        EntityID:   entityID,
        EntityType: entityState.Node.Type,
        Properties: entityState.Node.Properties,
        Updated:    entityState.UpdatedAt,
    }
    m.metadataCache.Set(entityID, metadata)

    // 5. Optional: Store in L2 NATS KV cache (if configured)
    if m.embeddingCache != nil {
        contentHash := embedding.ContentHash(text)
        m.embeddingCache.Put(ctx, contentHash, vector)
    }

    return nil
}
```

## Text Extraction

Text is extracted from entity properties using configured `text_fields`:

**Default Fields:**

```go
[]string{"title", "content", "description", "summary", "text", "name"}
```

**Extraction Process:**

1. Iterate through `text_fields` in order
2. Check if entity property exists for each field
3. Convert property value to string
4. Concatenate all found text with spaces
5. Skip embedding if no text found

**Type Filtering:**

- `enabled_types`: Allow list of message types to embed (supports wildcards)
- `skip_types`: Deny list of message types to skip (evaluated first)
- Format: `"domain.category.version"` (e.g., `"alerts.critical.v1"`)

## Semantic Search

### Search Flow

```go
// Semantic Search (processor/graph/indexmanager/semantic.go:SearchSemantic)

func (m *Manager) SearchSemantic(ctx context.Context, query string, opts *SemanticSearchOptions) (*SearchResults, error) {
    // 1. Generate query embedding
    queryEmbedding, err := m.embedder.Generate(ctx, []string{query})
    if err != nil {
        return nil, err
    }

    // 2. Compute similarity scores for all cached vectors
    var hits []scoredHit
    for _, entityID := range m.vectorCache.Keys() {
        vec, ok := m.vectorCache.Get(entityID)
        if !ok {
            continue // Evicted between Keys() and Get()
        }

        // Cosine similarity
        score := embedding.CosineSimilarity(queryEmbedding, vec)

        // Apply threshold filter
        if score >= opts.Threshold {
            hits = append(hits, scoredHit{
                entityID: entityID,
                score:    score,
            })
        }
    }

    // 3. Sort by score (descending) and limit
    sort.Slice(hits, func(i, j int) bool {
        return hits[i].score > hits[j].score
    })
    if len(hits) > opts.Limit {
        hits = hits[:opts.Limit]
    }

    // 4. Enrich results with metadata
    results := &SearchResults{
        Query:     query,
        Threshold: opts.Threshold,
        Hits:      make([]SearchHit, len(hits)),
    }
    for i, hit := range hits {
        metadata, _ := m.metadataCache.Get(hit.entityID)
        results.Hits[i] = SearchHit{
            EntityID:   hit.entityID,
            Score:      hit.score,
            EntityType: metadata.EntityType,
            Properties: metadata.Properties,
            Updated:    metadata.Updated,
        }
    }

    return results, nil
}
```

### Search Options

```go
type SemanticSearchOptions struct {
    Threshold float64 // Minimum cosine similarity (0.0-1.0, default: 0.3)
    Limit     int     // Maximum results (default: 10)

    // Future: Type filters, property filters, etc.
}
```

## Prometheus Metrics

### Embedding Metrics

```prometheus
# Provider status
indexengine_embedding_provider{component="indexengine"}
# Values: 0=disabled, 1=bm25, 2=http

# Generation metrics
indexengine_embeddings_generated_total{component="indexengine"}
indexengine_embedding_text_extractions_total{component="indexengine"}
indexengine_embedding_generation_latency_seconds{component="indexengine"}

# Cache metrics
indexengine_embeddings_active{component="indexengine"}
indexengine_embedding_cache_hits_total{component="indexengine"}      # L2 NATS cache
indexengine_embedding_cache_misses_total{component="indexengine"}   # L2 NATS cache

# Reliability metrics
indexengine_embedding_fallbacks_total{component="indexengine"}
# Tracks HTTP → BM25 fallback events
```

### Query Metrics

```prometheus
# Search operations
indexengine_queries_total{component="indexengine",query_type="semantic"}
indexengine_queries_failed_total{component="indexengine",query_type="semantic"}
indexengine_query_latency_seconds{component="indexengine",query_type="semantic"}
```

## Configuration Examples

### BM25-Only (Default)

```json
{
  "indexer": {
    "embedding": {
      "enabled": true,
      "provider": "bm25",
      "text_fields": ["title", "content", "description"],
      "retention_window": "24h"
    }
  }
}
```

### HTTP Neural Embeddings (Optional Enhancement)

Use HTTP embeddings when semantic similarity is critical and you can manage the external service dependency.

```json
{
  "indexer": {
    "embedding": {
      "enabled": true,
      "provider": "http",
      "http_endpoint": "http://semembed:8081/v1/embeddings",
      "http_model": "BAAI/bge-small-en-v1.5",
      "text_fields": ["title", "content", "description", "summary"],
      "retention_window": "24h",
      "cache_bucket": "EMBEDDINGS_CACHE"
    }
  }
}
```

**Note**: System automatically falls back to BM25 if HTTP service unavailable (graceful degradation).

### Environment Variable Override

```bash
# Override provider via environment
EMBEDDING_PROVIDER=http
EMBEDDING_HTTP_ENDPOINT=http://semembed:8081/v1/embeddings
EMBEDDING_HTTP_MODEL=BAAI/bge-small-en-v1.5
```

### Why Type Filtering Is Critical

**Memory management in production deployments** requires careful type filtering. Without proper configuration, high-volume data streams can **quickly exhaust in-memory caches** and cause system instability.

#### The Problem: High-Volume Telemetry

**Scenario**: Real-time robotics deployment with continuous telemetry

```text
Fleet: 10 drones
Telemetry frequency: 10 Hz per drone
Entity types: position, velocity, battery, temperature, GPS, IMU
```

**Without type filtering:**

```text
Messages per second:  10 drones × 10 Hz × 6 entity types = 600 entities/sec
Messages per hour:    600 × 3,600 = 2,160,000 entities
Memory per entity:    ~2KB (vector + metadata)

Total memory (1 hour):  2,160,000 × 2KB = 4.3GB
Total memory (24 hours): 4.3GB × 24 = 103GB ⚠️
```

**Result**: Cache exhaustion, OOM kills, system failure

**With proper type filtering:**

```json
{
  "skip_types": [
    "robotics.telemetry.*",    // Skip high-frequency telemetry
    "sensors.raw.*",           // Skip raw sensor readings
    "position.*.*",            // Skip position updates
    "velocity.*.*"             // Skip velocity data
  ],
  "enabled_types": [
    "alerts.*.*",              // Keep alerts
    "events.mission.*",        // Keep mission events
    "notes.*.*",               // Keep operator notes
    "anomaly.*.*"              // Keep anomaly detections
  ]
}
```

**Result**: Only semantically meaningful entities cached

```text
Meaningful events: ~10-100 per hour
Memory (24 hours):  2,400 entities × 2KB = 4.8MB ✅
```

#### Memory Growth Patterns

**Common high-volume entity types to filter:**

| Entity Type | Frequency | Memory Impact (24h) | Filter? |
|------------|-----------|---------------------|---------|
| **robotics.telemetry.*** | 10-50 Hz | 10-100GB | ✅ Always skip |
| **sensors.raw.*** | 5-20 Hz | 5-50GB | ✅ Always skip |
| **position.*** | 1-10 Hz | 1-10GB | ✅ Usually skip |
| **metrics.*** | 1 Hz | 1-5GB | ✅ Usually skip |
| **alerts.*** | 0.01 Hz | 1-10MB | ❌ Keep for search |
| **events.*** | 0.1 Hz | 10-100MB | ❌ Keep for search |
| **notes.*** | 0.001 Hz | <1MB | ❌ Keep for search |

#### When Type Filtering Fails

**Symptoms of improper filtering:**

1. **Rapid memory growth**: Container memory steadily increases
2. **OOM kills**: Process terminated by kernel
3. **Slow queries**: Scanning millions of cached vectors
4. **Cache churn**: Frequent evictions, poor hit rates

**Monitoring:**

```prometheus
# Watch active embeddings count
indexengine_embeddings_active{component="indexengine"}

# Alert if growing too fast
rate(indexengine_embeddings_active[5m]) > 1000  # More than 1000/sec
```

#### Best Practices for IoT/Robotics

**1. Default to skip high-frequency types:**

```json
{
  "skip_types": [
    "telemetry.*.*",
    "sensors.*.*",
    "position.*.*",
    "velocity.*.*",
    "metrics.*.*"
  ]
}
```

**2. Explicitly enable semantic types:**

```json
{
  "enabled_types": [
    "alerts.*.*",           // Operator alerts
    "events.mission.*",     // Mission lifecycle
    "events.anomaly.*",     // Anomaly detection
    "notes.*.*",            // Human annotations
    "waypoints.*.*"         // Mission waypoints
  ]
}
```

**3. Consider aggregation for telemetry:**

Instead of embedding every telemetry message, create **summary entities** at lower frequency:

```text
❌ BAD:  Embed every position update (10 Hz)
✅ GOOD: Embed mission summaries (0.01 Hz)
        "Mission 123: Flew 5km, battery used 40%, 2 waypoints completed"
```

**4. Monitor and adjust:**

```bash
# Check memory usage
docker stats semstreams

# Check active embeddings
curl localhost:9090/metrics | grep embeddings_active

# Adjust filters based on actual patterns
```

#### Configuration Examples

**Robotics deployment (conservative):**

```json
{
  "embedding": {
    "enabled": true,
    "provider": "bm25",
    "retention_window": "12h",  // Shorter retention for memory
    "skip_types": [
      "robotics.telemetry.*",
      "sensors.*.*",
      "position.*.*",
      "velocity.*.*",
      "imu.*.*",
      "gps.raw.*"
    ],
    "enabled_types": [
      "alerts.*.*",
      "events.mission.*",
      "events.anomaly.*",
      "notes.*.*"
    ]
  }
}
```

**IoT deployment (sensor-heavy):**

```json
{
  "embedding": {
    "enabled": true,
    "provider": "bm25",
    "retention_window": "6h",   // Very short for edge devices
    "skip_types": [
      "sensors.*.*",
      "metrics.*.*",
      "telemetry.*.*"
    ],
    "enabled_types": [
      "alerts.critical.*",       // Only critical alerts
      "events.alarm.*",          // Alarm events
      "device.status.*"          // Device status changes
    ]
  }
}
```

### Type Filtering Configuration

**Filter syntax:**

- `enabled_types`: Allow list of message types to embed (supports wildcards)
- `skip_types`: Deny list of message types to skip (evaluated first)
- Format: `"domain.category.version"` (e.g., `"alerts.critical.v1"`)

**Example configuration:**

```json
{
  "indexer": {
    "embedding": {
      "enabled": true,
      "provider": "bm25",
      "enabled_types": [
        "alerts.*.*",      // All alerts
        "events.incident.*", // Incident events
        "notes.*.*"        // All notes
      ],
      "skip_types": [
        "telemetry.*.*",   // Skip raw telemetry
        "sensors.*.*",     // Skip sensor data
        "metrics.*.*"      // Skip numeric metrics
      ]
    }
  }
}
```

## Performance Characteristics

### BM25 Provider

| Metric | Performance |
|--------|-------------|
| Latency (single) | ~0.1-1ms |
| Throughput (batch-10) | ~1-5ms |
| Memory (per embedding) | ~1.5KB (384 floats) |
| Startup time | Instant |
| Dependencies | None |

### HTTP Provider (semembed)

| Metric | Performance |
|--------|-------------|
| Latency (single) | ~5-10ms (local network) |
| Throughput (batch-10) | ~20-50ms |
| Memory (per embedding) | ~1.5KB (384 floats) |
| Startup time | ~3s (model load) |
| Dependencies | semembed service (HTTP) |

### Memory Usage Estimation

**L1 Cache:**

- Vector: 384 floats × 4 bytes = 1,536 bytes
- Metadata: ~500 bytes (varies by entity)
- **Total per entity**: ~2KB

**Example: 10,000 entities**

- L1 Cache: 10,000 × 2KB = ~20MB
- Retention: 24h (auto-eviction)

**L2 Cache (NATS KV):**

- Content-addressed (deduplicated)
- Persistent storage (survives restarts)
- Size depends on unique text content

## Troubleshooting

### HTTP Service Unavailable

**Symptom:** Logs show "HTTP embedding service unavailable - falling back to BM25"

**Cause:** semembed service not running or unreachable

**Solution:**

```bash
# Check service status
docker ps | grep semembed

# Start embedding service
docker compose -f docker-compose.services.yml --profile embedding up -d

# Check health
curl http://localhost:8081/health

# Monitor fallback metric
curl http://localhost:9090/metrics | grep embedding_fallbacks_total
```

### No Search Results

**Symptom:** Semantic search returns empty results

**Possible Causes:**

1. **No embeddings cached:** Check `indexengine_embeddings_active` metric
2. **Threshold too high:** Lower `threshold` in search options (default: 0.3)
3. **Text extraction failed:** Check logs for "No text content found"
4. **Type filters:** Verify entity types match `enabled_types` / `skip_types`

**Debugging:**

```bash
# Check active embeddings
curl http://localhost:9090/metrics | grep embeddings_active

# Check generation stats
curl http://localhost:9090/metrics | grep embeddings_generated_total

# Check text extractions
curl http://localhost:9090/metrics | grep embedding_text_extractions_total
```

### High Memory Usage

**Symptom:** IndexManager using excessive memory

**Cause:** Large retention window + many entities

**Solutions:**

1. Reduce `retention_window` (e.g., from 24h to 12h)
2. Add type filters to skip embedding certain message types
3. Reduce `text_fields` to only essential fields
4. Enable L2 NATS cache and reduce L1 retention

```json
{
  "embedding": {
    "retention_window": "12h",  // Reduced from 24h
    "skip_types": ["telemetry.*.*", "sensors.*.*"]
  }
}
```

## PathRAG Integration

### Complementary Query Strategies

SemStreams provides two complementary retrieval strategies:

**Semantic Search** (this document): Content-based similarity using BM25 or neural embeddings
**PathRAG**: Relationship-based discovery through graph traversal

### When to Use Each

| Query Type | Semantic Search | PathRAG |
|------------|----------------|---------|
| **"Find similar content"** | ✅ Primary | ❌ Not applicable |
| **"What's related to X?"** | ⚠️ Indirect | ✅ Direct |
| **Keyword matching** | ✅ Excellent (BM25) | ❌ Poor |
| **Dependency chains** | ❌ Cannot do | ✅ Excellent |
| **Impact analysis** | ❌ Cannot do | ✅ Excellent |
| **Real-time (<50ms)** | ✅ Yes | ⚠️ Depends (10-200ms) |

### Hybrid Query Pattern

**Workflow**: Combine semantic + path for comprehensive retrieval

```go
// 1. Semantic search finds initial candidates
semanticResults, _ := indexManager.SearchSemantic(ctx, &SemanticSearchOptions{
    Query:     "drone battery failure",
    Threshold: 0.3,
    Limit:     5,
})

// 2. PathRAG expands from top result
topEntity := semanticResults.Hits[0].EntityID
pathResults, _ := queryClient.ExecutePathQuery(ctx, PathQuery{
    StartEntity: topEntity,
    MaxDepth:    3,
    EdgeFilter:  []string{"related_to", "triggered_by"},
    DecayFactor: 0.8,
})

// 3. Merge results with score fusion
finalScore := (semanticScore * 0.6) + (pathScore * 0.4)
```

### PathRAG Performance vs Semantic Search

| Metric | Semantic Search (BM25) | Semantic Search (HTTP) | PathRAG |
|--------|----------------------|----------------------|---------|
| **Latency (P50)** | 0.1-1ms | 5-10ms | 10-50ms |
| **Latency (P95)** | 2-5ms | 15-30ms | 50-200ms |
| **Memory** | 2KB per entity | 2KB per entity | 2KB per visited node |
| **Scalability** | O(N) entities | O(N) entities | O(B^D) (breadth^depth) |
| **Dependencies** | None (BM25) | HTTP service | NATS KV buckets |

**Recommendation**: Use semantic search for initial filtering, PathRAG for relationship exploration.

### Example Use Cases

**Use Case 1: Alert Context Discovery**

```bash
# Semantic search: Find similar past alerts
curl -X POST /search/semantic -d '{"query": "network timeout", "limit": 3}'

# PathRAG: Explore dependencies of affected service
curl -X POST /entity/service-api/path -d '{"max_depth": 3, "edge_filter": ["depends_on"]}'
```

**Use Case 2: Spatial Awareness**

```bash
# Semantic search: Find "drones with low battery"
curl -X POST /search/semantic -d '{"query": "low battery drone", "limit": 5}'

# PathRAG: Find nearby base stations within 2 hops
curl -X POST /entity/drone-001/path -d '{"max_depth": 2, "edge_filter": ["near"]}'
```

**Use Case 3: Documentation Discovery**

```bash
# Semantic search: Find relevant docs
curl -X POST /search/semantic -d '{"query": "graph processor API", "limit": 10}'

# PathRAG: Explore related examples and guides
curl -X POST /entity/doc-graphprocessor/path \
  -d '{"max_depth": 3, "edge_filter": ["references", "example_of"]}'
```

## Related Documentation

**Query Strategy Guides**:

- [PathRAG Guide](pathrag.md) - Graph traversal for contextual retrieval
- [GraphRAG Guide](graphrag.md) - Semantic similarity search with community detection
- [Choosing Your Query Strategy](choosing-query-strategy.md) - Decision tree for query approaches
- [Hybrid Query Patterns](hybrid-queries.md) - Combining PathRAG + GraphRAG

**System Guides**:

- [Message System](message-system.md) - How data flows through SemStreams
- [Vocabulary System](vocabulary-system.md) - Semantic predicates and graph language
- [Federation Guide](federation.md) - Distributed deployments

**Integration**:

- [GraphQL API](../integration/graphql-api.md) - GraphQL endpoint contracts
- [REST API](../integration/rest-api.md) - OpenAPI specifications
- [NATS Events](../integration/nats-events.md) - Event schemas and messaging

**Architecture**:

- [Architecture Overview](../getting-started/architecture.md) - High-level system design

---

**Implementation**: See [semstreams](https://github.com/c360/semstreams) repository

- BM25 Embedder: `processor/graph/indexmanager/embedding/bm25_embedder.go`
- HTTP Embedder: `processor/graph/indexmanager/embedding/http_embedder.go`
- Semantic Search: `processor/graph/indexmanager/semantic.go`
- Index Manager: `processor/graph/indexmanager/manager.go`

**Embedding Services**:

- [semembed](https://github.com/c360/semembed) - HTTP embedding service (Rust + fastembed)
