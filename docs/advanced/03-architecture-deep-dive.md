# Architecture Deep Dive

**Understanding SemStreams internals: event flows, graph building, and semantic indexing**

---

## System Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        SemStreams Runtime                    │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐            │
│  │  Input   │────▶│ Processor│────▶│  Output  │            │
│  │Components│     │Components│     │Components│            │
│  └──────────┘     └──────────┘     └──────────┘            │
│        │                │                 │                  │
│        └────────────────┼─────────────────┘                  │
│                         ▼                                     │
│              ┌─────────────────────┐                         │
│              │   Message Bus       │                         │
│              │   (NATS+JetStream)  │                         │
│              └─────────────────────┘                         │
│                         │                                     │
│                         ▼                                     │
│              ┌─────────────────────┐                         │
│              │  Graph Processor    │                         │
│              │  • Entity Tracking  │                         │
│              │  • Relationship     │                         │
│              │    Indexing         │                         │
│              │  • Semantic Search  │                         │
│              └─────────────────────┘                         │
│                         │                                     │
│                         ▼                                     │
│              ┌─────────────────────┐                         │
│              │  Storage Layer      │                         │
│              │  • NATS KV (graph)  │                         │
│              │  • NATS ObjectStore │                         │
│              │  • Index Buckets    │                         │
│              └─────────────────────┘                         │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
              ┌─────────────────────┐
              │ Optional Services   │
              │ • SemEmbed (GPU)    │
              │ • SemSummarize (LLM)│
              └─────────────────────┘
```

---

## Event Flow Architecture

### Message Bus Pattern

SemStreams uses a **publish/subscribe message bus** (NATS) instead of direct component connections.

**Traditional linear approach:**
```
UDP Input → Parser → Filter → Graph → File Output
  (tight coupling, rigid flow)
```

**SemStreams message bus approach:**
```
UDP Input
    ↓ publish("raw.udp")
Message Bus
    ├─ subscribe("raw.udp") → Parser
    │      ↓ publish("parsed.json")
    ├─ subscribe("parsed.json") → Filter
    │      ↓ publish("entities")
    ├─ subscribe("entities") → Graph Processor
    ├─ subscribe("entities") → File Output
    └─ subscribe("entities") → HTTP Output
```

**Key advantages:**

- **Loose coupling**: Components don't know about each other
- **Non-linear flows**: Multiple subscribers on same subject
- **Dynamic routing**: Route based on message content, not topology
- **Independent scaling**: Scale busy components without affecting others

### Subject Naming Convention

```
{category}.{component}.{entity_type}.{detail}

Examples:
  raw.udp.messages              # Raw UDP input
  events.graph.entity.drone     # Drone entity events
  events.graph.entity.*         # All entity events (wildcard)
  events.rule.triggered         # Rule trigger events
  graph.query.semantic          # Semantic search queries
```

**Wildcards:**
- `*` matches single token: `events.graph.entity.*` → all entity types
- `>` matches multiple tokens: `events.>` → all events

---

## Graph Processor Architecture

The graph processor is the heart of SemStreams, automatically building entity graphs from event streams.

### Component Structure

```
┌─────────────────────────────────────────────────────────┐
│                    Graph Processor                       │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌───────────────┐      ┌───────────────┐               │
│  │ Message       │      │ Worker Pool   │               │
│  │ Handler       │─────▶│ (8 workers)   │               │
│  │ (subscriber)  │      └───────┬───────┘               │
│  └───────────────┘              │                        │
│                                  ▼                        │
│                       ┌──────────────────┐               │
│                       │  Data Manager    │               │
│                       │  • Entity CRUD   │               │
│                       │  • Edge Tracking │               │
│                       │  • KV Operations │               │
│                       └────────┬─────────┘               │
│                                │                          │
│                 ┌──────────────┼──────────────┐          │
│                 ▼              ▼              ▼           │
│          ┌──────────┐   ┌──────────┐   ┌──────────┐     │
│          │ Indexer  │   │ Querier  │   │ Embedding│     │
│          │ Manager  │   │          │   │ Manager  │     │
│          └──────────┘   └──────────┘   └──────────┘     │
│                 │              │              │           │
│                 └──────────────┼──────────────┘           │
│                                ▼                          │
│                       ┌─────────────────┐                │
│                       │  NATS Storage   │                │
│                       │  • Entity KV    │                │
│                       │  • Index KV     │                │
│                       └─────────────────┘                │
└─────────────────────────────────────────────────────────┘
```

### Message Handler Flow

```go
1. Subscribe to "events.graph.entity.*"
2. Receive Entity message
3. Validate message structure
4. Enqueue to worker pool (buffered channel)
5. Worker picks up message
6. Data Manager: Upsert entity to KV
7. Index Manager: Update all enabled indexes
8. Embedding Manager: Generate/update embedding (if enabled)
9. Acknowledge message to NATS
```

**Worker pool pattern:**

```go
type Processor struct {
    workers    int
    queueSize  int
    workQueue  chan *types.EntityEvent
}

func (p *Processor) Start(ctx context.Context) error {
    // Create buffered work queue
    p.workQueue = make(chan *types.EntityEvent, p.queueSize)

    // Start N workers
    for i := 0; i < p.workers; i++ {
        go p.worker(ctx, i)
    }

    // Subscribe to NATS subject
    p.subscribe("events.graph.entity.*", func(msg *nats.Msg) {
        entity := parseEntity(msg.Data)
        p.workQueue <- entity  // Enqueue for processing
    })
}

func (p *Processor) worker(ctx context.Context, id int) {
    for {
        select {
        case entity := <-p.workQueue:
            p.processEntity(ctx, entity)
        case <-ctx.Done():
            return
        }
    }
}
```

### Data Manager: Entity Storage

**KV storage pattern:**

```
Key: entity:{entity_type}:{entity_id}
Value: JSON-serialized Entity

Example:
  Key: entity:drone:UAV-001
  Value: {"id":"UAV-001","type":"drone","battery":85.2,...}
```

**Edge tracking:**

```
Key: edges:{entity_id}
Value: Set of relationship IDs

Example:
  Key: edges:UAV-001
  Value: ["rel-001", "rel-002", "rel-003"]
```

**Batch operations for performance:**

```go
// Instead of individual updates:
for _, edge := range edges {
    kv.Put(edge.ID, edge.Data)  // N round-trips
}

// Batch updates:
batch := make([]KVPair, len(edges))
for i, edge := range edges {
    batch[i] = KVPair{Key: edge.ID, Value: edge.Data}
}
kv.PutBatch(batch)  // Single round-trip
```

---

## Index Architecture

### Index Types

SemStreams supports multiple index types, each optimized for specific query patterns.

#### 1. Predicate Index

**Purpose**: Find all relationships of a specific type

**Structure:**
```
Key: predicate:{predicate_type}
Value: Set of entity IDs

Example:
  Key: predicate:belongs_to
  Value: ["UAV-001", "UAV-002", "UAV-003"]
```

**Query:**
```
Find all entities with "belongs_to" relationships
→ Lookup "predicate:belongs_to" → Get entity ID list
```

**Performance**: O(1) lookup, returns all entities with that predicate

#### 2. Incoming Index

**Purpose**: Find entities that have relationships pointing TO this entity (reverse lookup)

**Structure:**
```
Key: incoming:{entity_id}:{predicate_type}
Value: Set of source entity IDs

Example:
  Key: incoming:fleet-rescue:belongs_to
  Value: ["UAV-001", "UAV-002"]  # Drones that belong to this fleet
```

**Query:**
```
Find all drones that belong to fleet-rescue
→ Lookup "incoming:fleet-rescue:belongs_to"
```

**Performance**: O(1) lookup for reverse relationships (critical for graph traversal)

#### 3. Alias Index

**Purpose**: Resolve entity names to IDs

**Structure:**
```
Key: alias:{alias_name}
Value: entity_id

Example:
  Key: alias:rescue-alpha
  Value: UAV-001
```

**Query:**
```
Find entity by name "rescue-alpha"
→ Lookup "alias:rescue-alpha" → Get "UAV-001"
→ Lookup entity "UAV-001"
```

**Performance**: O(1) name resolution

#### 4. Spatial Index

**Purpose**: Geospatial queries (entities within bounding box)

**Structure**: Geohash-based indexing

```
Key: spatial:{geohash}
Value: Set of entity IDs

Example:
  Key: spatial:9q8yy  # San Francisco area
  Value: ["UAV-001", "UAV-003"]
```

**Query:**
```
Find entities near [37.7749, -122.4194] within 10km
→ Compute geohashes for bounding box
→ Lookup all matching geohash buckets
→ Filter by exact distance
```

**Performance**: O(log N) for geohash lookup, O(M) for distance filtering

#### 5. Temporal Index

**Purpose**: Time-based queries (entities created/updated in time range)

**Structure**: Time bucket indexing

```
Key: temporal:{bucket}
Value: Set of entity IDs

Example:
  Key: temporal:2024-01-15T10:00
  Value: ["UAV-001", "UAV-002"]  # Entities active in this hour
```

**Query:**
```
Find entities active between 10:00-11:00
→ Lookup temporal buckets for time range
→ Union all entity sets
```

**Performance**: O(K) where K = number of time buckets in range

#### 6. Embedding Index (GraphRAG)

**Purpose**: Semantic similarity search

**Structure**: Vector database (in-memory or external)

```
Entity ID → Vector (1536 dimensions)

Example:
  "UAV-001" → [0.12, -0.34, 0.56, ...]
```

**Query:**
```
Find entities semantically similar to "low battery emergency"
→ Generate query embedding
→ Compute cosine similarity with all entity embeddings
→ Return top K most similar
```

**Performance**: O(N) for brute force, O(log N) with HNSW/IVF indexes

### Index Update Flow

```
1. Entity created/updated in KV
2. DataManager emits index update event
3. IndexManager receives event
4. For each enabled index:
   a. Extract relevant fields from entity
   b. Compute index keys
   c. Batch update KV bucket
5. If embedding enabled:
   a. Extract text content
   b. Send to SemEmbed service (async)
   c. Store vector in embedding index
```

**Batch processing optimization:**

```go
// Collect index updates
updates := make(map[string][]KVPair)
for _, entity := range entities {
    // Predicate index
    for _, edge := range entity.Edges {
        key := fmt.Sprintf("predicate:%s", edge.Predicate)
        updates["predicate"] = append(updates["predicate"],
            KVPair{Key: key, Value: entity.ID})
    }

    // Incoming index
    for _, edge := range entity.Edges {
        key := fmt.Sprintf("incoming:%s:%s", edge.Target, edge.Predicate)
        updates["incoming"] = append(updates["incoming"],
            KVPair{Key: key, Value: entity.ID})
    }
}

// Batch update each index bucket
for bucket, pairs := range updates {
    kv.PutBatch(bucket, pairs)
}
```

---

## Query Architecture

### Query Flow

```
1. HTTP Request → API Gateway
2. API Gateway publishes to "graph.query.{type}"
3. Graph Processor Querier subscribes
4. Parse query parameters
5. Check cache (if enabled)
6. Cache miss → Execute query:
   a. Determine required indexes
   b. Fetch from index KV buckets
   c. Resolve entity IDs to full entities
   d. Apply filters and limits
7. Cache result
8. Return to API Gateway
9. API Gateway returns HTTP response
```

### Query Types

#### 1. Entity by ID

```json
GET /entity/UAV-001

Query Plan:
  → Direct KV lookup: "entity:drone:UAV-001"
  → O(1) performance
```

#### 2. Entities by Type

```json
GET /entities?type=drone

Query Plan:
  → Scan KV prefix: "entity:drone:*"
  → O(N) where N = number of drones
  → Cache recommended
```

#### 3. Relationship Traversal

```json
GET /entities?predicate=belongs_to&target=fleet-rescue

Query Plan:
  → Lookup predicate index: "predicate:belongs_to"
  → Filter by target: "fleet-rescue"
  → Resolve entity IDs
  → O(M) where M = entities with predicate
```

#### 4. Reverse Relationship

```json
GET /entities?incoming=fleet-rescue&predicate=belongs_to

Query Plan:
  → Lookup incoming index: "incoming:fleet-rescue:belongs_to"
  → Resolve entity IDs
  → O(1) index lookup, O(K) entity resolution
```

#### 5. Semantic Search

```json
POST /search/semantic
{"query": "low battery emergency", "limit": 10}

Query Plan:
  → Generate query embedding
  → Compute similarity with all entity embeddings
  → Sort by score descending
  → Return top K
  → O(N) for N entities (no index acceleration yet)
```

### Query Cache

```go
type QueryCache struct {
    cache     map[string]CachedResult
    ttl       time.Duration
    maxSize   int
}

type CachedResult struct {
    Result    interface{}
    Timestamp time.Time
}

func (qc *QueryCache) Get(key string) (interface{}, bool) {
    result, exists := qc.cache[key]
    if !exists {
        return nil, false
    }

    // Check TTL
    if time.Since(result.Timestamp) > qc.ttl {
        delete(qc.cache, key)
        return nil, false
    }

    return result.Result, true
}
```

**Cache invalidation strategies:**

- **TTL-based**: Expire after fixed time (simple, works for most cases)
- **Event-based**: Invalidate when entities change (complex, high accuracy)
- **Hybrid**: TTL + invalidate on writes to specific entities

---

## Semantic Enhancement Architecture

### Native vs Enhanced Semantic

**Native Semantic (Always Available):**

```
Entity text → Go algorithms (TF-IDF, BM25) → Similarity score
```

- No external dependencies
- Works on Raspberry Pi
- Good enough for basic similarity
- Fast (pure Go)

**Enhanced Semantic (Optional):**

```
Entity text → SemEmbed service → Transformer model → Vector embedding
```

- Requires SemEmbed service (~2GB RAM)
- Much higher quality similarity
- Slower (model inference)
- GPU optional (faster with GPU)

### SemEmbed Integration

```
┌─────────────────┐
│ Graph Processor │
└────────┬────────┘
         │ HTTP POST /embed
         ▼
┌─────────────────┐
│ SemEmbed Service│
│ • Model: all-   │
│   MiniLM-L6-v2  │
│ • Vector: 384d  │
│ • GPU: optional │
└────────┬────────┘
         │ JSON response: {"embedding": [0.12, ...]}
         ▼
┌─────────────────┐
│ Embedding Index │
│ (in-memory)     │
└─────────────────┘
```

**Configuration:**

```json
{
  "indexer": {
    "embedding": {
      "enabled": true,
      "service_url": "http://localhost:8081",
      "batch_size": 32,
      "timeout": "5s",
      "fields": ["title", "description", "content"]
    }
  }
}
```

**Async embedding pattern:**

```go
// Don't block entity indexing on embedding
go func() {
    embedding, err := embeddingClient.Embed(ctx, entity.Text)
    if err != nil {
        log.Error("embedding failed", "error", err)
        return  // Continue without embedding
    }
    embeddingIndex.Store(entity.ID, embedding)
}()
```

### SemSummarize Integration

```
┌─────────────────┐
│ Graph Processor │
└────────┬────────┘
         │ HTTP POST /summarize
         ▼
┌──────────────────┐
│ SemSummarize API │
│ (OpenAI-compat)  │
│ • Local LLM      │
│ • Cloud API      │
└────────┬─────────┘
         │ JSON: {"summary": "..."}
         ▼
┌─────────────────┐
│ Entity.summary  │
│ (field update)  │
└─────────────────┘
```

**Use cases:**

- Summarize long entity descriptions
- Generate entity titles from content
- Extract key metadata from unstructured text

---

## Edge Federation Architecture

### Edge-to-Cloud Pattern

```
┌───────────────────────────────────────┐
│ EDGE (Raspberry Pi, Vehicle)          │
│ • PathRAG only (no embeddings)        │
│ • Local graph, low latency queries    │
│ • Selective sync (exceptions only)    │
└─────────────────┬─────────────────────┘
                  │
                  │ WebSocket Output
                  │ (bandwidth-aware sync)
                  │
                  ▼
┌───────────────────────────────────────┐
│ CLOUD (Laptop, Server)                │
│ • WebSocket Input                     │
│ • Full stack (PathRAG + GraphRAG)     │
│ • Aggregate multi-edge graphs         │
│ • Semantic search, analytics          │
└───────────────────────────────────────┘
```

**Federation configuration:**

```json
{
  "components": [
    {
      "id": "edge-output",
      "type": "websocket_output",
      "config": {
        "port": 8080,
        "path": "/edge-data",
        "filters": {
          "priority": {"gte": 3},
          "battery.level": {"lte": 20}
        }
      }
    }
  ]
}
```

**WebSocket federation pattern:**

```text
Edge (WebSocket Output)
  ↓ TLS connection
Cloud (WebSocket Input)
  ← Receives filtered events
  → Optional: Send commands back
```

---

## Next Steps

- **[Performance Tuning](02-performance-tuning.md)** - Optimize for your workload
- **[Production Patterns](04-production-patterns.md)** - Deployment best practices
- **[Algorithm Reference](05-algorithm-reference.md)** - Native algorithm details
- **[Graph Indexing](../graph/04-indexing.md)** - Index configuration guide

---

**Understanding the architecture helps you configure SemStreams effectively** - Know what runs where, when to use which index, and how to scale.
