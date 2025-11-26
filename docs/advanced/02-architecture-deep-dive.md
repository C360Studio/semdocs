# Architecture Deep Dive

**Understanding SemStreams internals: event flows, graph building, and semantic indexing**

---

## System Overview

```text
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

## Entity ID Architecture

### Federated Entity IDs

SemStreams uses **6-part federated entity IDs** to enable multi-tenant, multi-platform deployments:

```text
org.platform.domain.system.type.instance
```

**Components:**

| Part | Description | Example |
|------|-------------|---------|
| `org` | Organization identifier | `c360` |
| `platform` | Platform/deployment | `platform1` |
| `domain` | Business domain | `robotics` |
| `system` | System within domain | `gcs1` |
| `type` | Entity type | `drone` |
| `instance` | Unique instance ID | `1` |

**Examples:**

```text
c360.platform1.robotics.gcs1.drone.1
c360.platform1.robotics.mav1.battery.0
c360.platform1.logistics.warehouse1.sensor.temp42
```

**Validation:**

Entity IDs must match the pattern: `^[a-zA-Z0-9]+\.[a-zA-Z0-9]+\.[a-zA-Z0-9]+\.[a-zA-Z0-9]+\.[a-zA-Z0-9]+\.[a-zA-Z0-9]+$`

---

## Event Flow Architecture

### Message Bus Pattern

SemStreams uses a **publish/subscribe message bus** (NATS) instead of direct component connections.

**Traditional linear approach:**

```text
UDP Input → Parser → Filter → Graph → File Output
  (tight coupling, rigid flow)
```

**SemStreams message bus approach:**

```text
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

```text
{category}.{component}.{entity_type}.{detail}

Examples:
  raw.udp.messages              # Raw UDP input
  storage.*.events              # ObjectStore events (default input)
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

```text
┌─────────────────────────────────────────────────────────┐
│                    Graph Processor                       │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌───────────────┐      ┌───────────────┐               │
│  │ Message       │      │ Worker Pool   │               │
│  │ Manager       │─────▶│ (10 workers)  │               │
│  │ (subscriber)  │      └───────┬───────┘               │
│  └───────────────┘              │                        │
│                                  ▼                        │
│                       ┌──────────────────┐               │
│                       │  Data Manager    │               │
│                       │  • Entity CRUD   │               │
│                       │  • Edge Tracking │               │
│                       │  • L1/L2 Caching │               │
│                       └────────┬─────────┘               │
│                                │                          │
│                 ┌──────────────┼──────────────┐          │
│                 ▼              ▼              ▼           │
│          ┌──────────┐   ┌──────────┐   ┌──────────┐     │
│          │ Index    │   │ Query    │   │ Embedding│     │
│          │ Manager  │   │ Manager  │   │ (BM25/   │     │
│          │ (5 types)│   │          │   │  HTTP)   │     │
│          └──────────┘   └──────────┘   └──────────┘     │
│                 │              │              │           │
│                 └──────────────┼──────────────┘           │
│                                ▼                          │
│                       ┌─────────────────┐                │
│                       │  NATS KV Storage│                │
│                       │  • ENTITY_STATES│                │
│                       │  • Index Buckets│                │
│                       └─────────────────┘                │
└─────────────────────────────────────────────────────────┘
```

### Message Handler Flow

```text
1. Subscribe to "storage.*.events" (ObjectStore events)
2. Receive message data
3. Validate message structure
4. Submit to worker pool (buffered queue)
5. Worker processes message:
   a. MessageManager: Transform message → EntityState
   b. DataManager: Upsert entity to ENTITY_STATES KV
   c. IndexManager: Update all enabled indexes
   d. EmbeddingManager: Generate/update embedding (if enabled)
6. Acknowledge message to NATS
```

**Worker pool architecture:**

```go
type Processor struct {
    workerPool *worker.Pool[[]byte]
    config     *Config
}

// Default configuration
Workers:      10
QueueSize:    10000
InputSubject: "storage.*.events"
```

### Data Manager: Entity Storage

**KV storage pattern (ENTITY_STATES bucket):**

```text
Key:   {entity_id}
Value: JSON-serialized EntityState

Example:
  Key:   c360.platform1.robotics.gcs1.drone.1
  Value: {
    "node": {
      "id": "c360.platform1.robotics.gcs1.drone.1",
      "type": "robotics.drone"
    },
    "edges": [...],
    "triples": [
      {"subject": "c360.platform1.robotics.gcs1.drone.1",
       "predicate": "robotics.battery.level",
       "object": 85.2},
      ...
    ],
    "version": 3,
    "updated_at": "2024-01-15T10:30:00Z"
  }
```

**L1/L2 cache architecture:**

```text
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Request    │────▶│  L1 Cache   │────▶│  L2 Cache   │
│             │     │  (LRU)      │     │  (TTL)      │
└─────────────┘     └──────┬──────┘     └──────┬──────┘
                           │ miss              │ miss
                           └───────────┬───────┘
                                       ▼
                              ┌─────────────┐
                              │  NATS KV    │
                              │  Bucket     │
                              └─────────────┘
```

**Write coalescing for performance:**

```go
// Buffered writes are coalesced before flushing
// Multiple updates to same entity become single write
buffer := []*EntityWrite{
    {Entity: A, Op: Update},  // Entity A
    {Entity: A, Op: Update},  // Entity A (merged)
    {Entity: B, Op: Create},  // Entity B
}
// After coalescing: 2 writes instead of 3
```

---

## Index Architecture

### KV Bucket Overview

SemStreams uses separate NATS KV buckets for each index type:

| Bucket | Purpose | Key Pattern |
|--------|---------|-------------|
| `ENTITY_STATES` | Entity storage | `{entity_id}` |
| `PREDICATE_INDEX` | Triple predicate lookup | `{sanitized_predicate}` |
| `INCOMING_INDEX` | Reverse relationships | `{target_entity_id}` |
| `ALIAS_INDEX` | Name resolution | `alias--{alias}` or `entity--{id}` |
| `SPATIAL_INDEX` | Geospatial queries | `geo_{precision}_{lat}_{lon}` |
| `TEMPORAL_INDEX` | Time-based queries | `YYYY.MM.DD.HH` |

### Index Types

#### 1. Predicate Index

**Purpose**: Find all entities with a specific triple predicate

**Structure:**

```text
Bucket: PREDICATE_INDEX
Key:    {sanitized_predicate}
Value:  JSON array of entity IDs

Example:
  Key:   robotics.battery.level
  Value: ["c360.platform1.robotics.gcs1.drone.1",
          "c360.platform1.robotics.gcs1.drone.2",
          "c360.platform1.robotics.mav1.drone.0"]
```

**Query:**

```text
Find all entities with battery level triples
→ Lookup "robotics.battery.level" in PREDICATE_INDEX
→ Returns array of entity IDs
```

**Key sanitization:** Predicates are sanitized for NATS KV compatibility (spaces → underscores, invalid chars removed, max 255 chars).

#### 2. Incoming Index

**Purpose**: Find entities that have edges pointing TO this entity (reverse lookup)

**Structure:**

```text
Bucket: INCOMING_INDEX
Key:    {target_entity_id}
Value:  JSON array of source entity IDs

Example:
  Key:   c360.platform1.robotics.fleet1.fleet.rescue
  Value: ["c360.platform1.robotics.gcs1.drone.1",
          "c360.platform1.robotics.gcs1.drone.2"]
```

**Query:**

```text
Find all drones that belong to fleet "rescue"
→ Lookup "c360.platform1.robotics.fleet1.fleet.rescue" in INCOMING_INDEX
→ Returns array of source entity IDs
```

**Performance**: O(1) lookup for reverse relationships (critical for graph traversal)

#### 3. Alias Index

**Purpose**: Resolve entity names/identifiers to full entity IDs

**Structure (bidirectional):**

```text
Bucket: ALIAS_INDEX

Forward index:
  Key:   alias--{sanitized_alias}
  Value: entity_id (string)

Reverse index (for cleanup):
  Key:   entity--{entity_id}
  Value: JSON array of aliases

Examples:
  Key:   alias--rescue-alpha
  Value: "c360.platform1.robotics.gcs1.drone.1"

  Key:   entity--c360.platform1.robotics.gcs1.drone.1
  Value: ["rescue-alpha", "UAV-001"]
```

**Query:**

```text
Find entity by name "rescue-alpha"
→ Lookup "alias--rescue-alpha" in ALIAS_INDEX
→ Get "c360.platform1.robotics.gcs1.drone.1"
→ Lookup entity in ENTITY_STATES
```

**Alias predicates:** The system uses vocabulary registry to discover which triple predicates represent aliases (e.g., `identifier.callsign`, `identifier.tail_number`).

#### 4. Spatial Index

**Purpose**: Geospatial queries (entities within geographic region)

**Structure:**

```text
Bucket: SPATIAL_INDEX
Key:    geo_{precision}_{latInt}_{lonInt}
Value:  JSON object with entities map

Example:
  Key:   geo_7_53382_12819
  Value: {
    "entities": {
      "c360.platform1.robotics.gcs1.drone.1": {
        "lat": 37.7749,
        "lon": -122.4194,
        "alt": 100.5,
        "updated": 1705312200
      }
    },
    "last_update": 1705312200
  }
```

**Precision levels:**

| Precision | Resolution | Multiplier |
|-----------|------------|------------|
| 4 | ~2.5km | 10 |
| 5 | ~600m | 50 |
| 6 | ~120m | 100 |
| 7 (default) | ~30m | 300 |
| 8 | ~5m | 1000 |

**Geohash calculation:**

```go
latInt := int(math.Floor((lat + 90.0) * multiplier))
lonInt := int(math.Floor((lon + 180.0) * multiplier))
key := fmt.Sprintf("geo_%d_%d_%d", precision, latInt, lonInt)
```

#### 5. Temporal Index

**Purpose**: Time-based queries (entities active in time range)

**Structure:**

```text
Bucket: TEMPORAL_INDEX
Key:    YYYY.MM.DD.HH
Value:  JSON object with events array

Example:
  Key:   2024.01.15.10
  Value: {
    "events": [
      {
        "entity": "c360.platform1.robotics.gcs1.drone.1",
        "type": "update",
        "timestamp": "2024-01-15T10:15:30Z"
      },
      {
        "entity": "c360.platform1.robotics.gcs1.drone.2",
        "type": "update",
        "timestamp": "2024-01-15T10:22:45Z"
      }
    ],
    "entity_count": 2
  }
```

**Query:**

```text
Find entities active between 10:00-12:00 on 2024-01-15
→ Lookup "2024.01.15.10", "2024.01.15.11" in TEMPORAL_INDEX
→ Collect all entity IDs from events arrays
→ Deduplicate
```

#### 6. Embedding Index (GraphRAG)

**Purpose**: Semantic similarity search

**Providers:**

- **BM25** (default): Pure Go lexical search, no external service
- **HTTP**: External embedding service (SemEmbed, OpenAI, LocalAI)

**Structure:**

```text
Bucket: EMBEDDING_INDEX
Key:    {entity_id}
Value:  {
  "vector": [0.12, -0.34, 0.56, ...],
  "model": "all-MiniLM-L6-v2",
  "text_hash": "abc123...",
  "created_at": "2024-01-15T10:30:00Z"
}
```

**Deduplication bucket:**

```text
Bucket: EMBEDDING_DEDUP
Key:    {content_hash}
Value:  {entity_id}
```

### Index Update Flow

```text
1. Entity created/updated in ENTITY_STATES
2. IndexManager processes entity via KV watcher
3. For each enabled index:
   a. Extract relevant fields from EntityState
   b. Compute index keys
   c. Update KV bucket with CAS (Compare-And-Swap)
4. If embedding enabled:
   a. Extract text from configured fields
   b. Generate embedding (BM25 or HTTP)
   c. Store in EMBEDDING_INDEX
```

**CAS update pattern:**

```go
// Race-free index updates using Compare-And-Swap
err := kvStore.UpdateWithRetry(ctx, key, func(current []byte) ([]byte, error) {
    entities := parseEntityList(current)
    if !contains(entities, entityID) {
        entities = append(entities, entityID)
    }
    return json.Marshal(entities)
})
```

---

## Query Architecture

### Query Flow

```text
1. Request arrives (NATS request/reply or HTTP)
2. QueryManager receives query
3. Rate limiting check (100 queries/sec default)
4. Check cache (if enabled)
5. Cache miss → Execute query:
   a. Determine required indexes
   b. Fetch from index KV buckets
   c. Resolve entity IDs to full EntityState
   d. Apply filters and limits
6. Cache result
7. Return response
```

### Query Types

#### 1. Entity by ID

```text
GET entity by "c360.platform1.robotics.gcs1.drone.1"

Query Plan:
  → Direct KV lookup in ENTITY_STATES
  → Key: "c360.platform1.robotics.gcs1.drone.1"
  → O(1) performance
```

#### 2. Entity by Alias

```text
GET entity by alias "rescue-alpha"

Query Plan:
  → Lookup "alias--rescue-alpha" in ALIAS_INDEX
  → Get entity ID
  → Lookup entity in ENTITY_STATES
  → O(1) + O(1) = O(1) performance
```

#### 3. Entities by Predicate

```text
GET entities with predicate "robotics.battery.level"

Query Plan:
  → Lookup "robotics.battery.level" in PREDICATE_INDEX
  → Get array of entity IDs
  → Resolve each entity from ENTITY_STATES
  → O(1) + O(N) where N = matching entities
```

#### 4. Reverse Relationship

```text
GET entities pointing to "c360.platform1.robotics.fleet1.fleet.rescue"

Query Plan:
  → Lookup target in INCOMING_INDEX
  → Get array of source entity IDs
  → Resolve each entity from ENTITY_STATES
  → O(1) + O(N) where N = incoming entities
```

#### 5. Semantic Search

```text
POST /search/semantic
{"query": "low battery emergency", "limit": 10}

Query Plan:
  → Generate query embedding (BM25 or HTTP)
  → Compute similarity with indexed embeddings
  → Sort by score descending
  → Return top K entities
  → O(N) for N entities (brute force)
```

### Query Cache

The QueryManager implements LRU caching with TTL:

```go
type QueryCache struct {
    cache   *lru.Cache[string, *CachedResult]
    ttl     time.Duration
    maxSize int
}
```

**Cache invalidation strategies:**

- **TTL-based**: Expire after fixed time (simple, default)
- **Event-based**: Invalidate on entity changes via KV watcher
- **Hybrid**: TTL + invalidate on specific entity updates

---

## Semantic Enhancement Architecture

### Native vs Enhanced Semantic

**Native Semantic (Always Available):**

```text
Entity text → Go algorithms (TF-IDF, BM25) → Similarity score
```

- No external dependencies
- Works on Raspberry Pi
- Good for basic similarity
- Fast (pure Go)

**Enhanced Semantic (Optional):**

```text
Entity text → SemEmbed service → Transformer model → Vector embedding
```

- Requires SemEmbed service (~2GB RAM)
- Higher quality similarity
- Slower (model inference)
- GPU optional (faster with GPU)

### SemEmbed Integration

```text
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
│ EMBEDDING_INDEX │
│ (NATS KV)       │
└─────────────────┘
```

**Configuration:**

```json
{
  "indexer": {
    "embedding": {
      "enabled": true,
      "provider": "http",
      "http_endpoint": "http://localhost:8082",
      "http_model": "all-MiniLM-L6-v2",
      "text_fields": ["title", "description", "content", "summary", "text", "name"],
      "retention_window": "24h"
    }
  }
}
```

**Fallback behavior:** If HTTP provider fails, system automatically falls back to BM25.

---

## Edge Federation Architecture

### Edge-to-Cloud Pattern

```text
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

---

## Next Steps

- **[Algorithm Reference](03-algorithm-reference.md)** - Native algorithm details
- **[Configuration Guide](04-configuration-guide.md)** - Enable/disable features based on your data
- **[Practical Query Patterns](05-query-strategies.md)** - Apply what you've learned

**Later:**

- **[Performance Tuning](07-performance-tuning.md)** - Optimize for your workload
- **[Production Patterns](08-production-patterns.md)** - Deployment best practices

**Related:**

- **[Graph Indexing](../graph/04-indexing.md)** - Index configuration guide

---

**Understanding the architecture helps you configure SemStreams effectively** - Know what runs where, when to use which index, and how to scale.
