# Core Concepts

Understanding the fundamental concepts behind SemStreams.

## Semantic Streaming

**Semantic streaming** combines real-time data processing with semantic graph patterns to create structured knowledge graphs from continuous data streams. Using a flow-based architecture with modular components, SemStreams can ingest from any source and parse any format.

### Traditional Streaming vs Semantic Streaming

| Traditional Streaming | Semantic Streaming |
|----------------------|-------------------|
| Process events | Build knowledge graphs |
| Schema-on-write | Schema-on-read |
| Point-in-time queries | Graph traversal patterns |
| No semantic relationships | Rich entity relationships |

### Key Properties

1. **Continuous Processing**: Events processed as they arrive
2. **Graph Construction**: Entities and relationships built incrementally
3. **Semantic Enrichment**: RDF vocabularies add meaning and context
4. **Temporal Awareness**: Track entity evolution over time

## Entity-Centric Model

Everything in SemStreams is an **entity** with:
- **ID**: Unique identifier (IRI or string)
- **Type**: RDF type (e.g., `schema:Person`, `Sensor`)
- **Properties**: Key-value attributes
- **Edges**: Relationships to other entities
- **Timestamp**: When the entity was observed

### Example Entity

```json
{
  "id": "sensor-123",
  "type": "https://schema.org/Device",
  "properties": {
    "name": "Temperature Sensor",
    "location": "Building A, Floor 2",
    "value": "72.5",
    "unit": "fahrenheit"
  },
  "edges": [
    {
      "predicate": "installedIn",
      "target": "room-201"
    }
  ],
  "timestamp": "2025-11-17T15:30:00Z"
}
```

## Graph Processing Pipeline

```
1. Ingestion    → Receive data from modular inputs (UDP, TCP, HTTP, WebSocket)
2. Parsing      → Extract entities and relationships from any format (JSON, CSV, raw bytes)
3. Storage      → Persist to NATS KV store
4. Indexing     → Build search indexes (semantic + spatial)
5. Querying     → GraphQL/REST access
6. Analysis     → PathRAG, clustering, summaries
```

## PathRAG: Pattern-Based Knowledge Extraction

**PathRAG** (Path-based Retrieval Augmented Generation) extracts knowledge by traversing graph patterns rather than full-text search.

### How It Works

1. **Define Pattern**: Specify entity types and relationship paths
2. **Traverse Graph**: Find all matching subgraphs
3. **Extract Context**: Gather entities and their neighborhoods
4. **Generate Response**: Use extracted context for LLM prompts

### Example Pattern

```
Person → worksFor → Company → locatedIn → City
```

This pattern finds people, their employers, and employer locations in one traversal.

**Benefits:**
- More precise than keyword search
- Captures relationship context
- Reduces LLM hallucination (structured data)
- Efficient (indexed graph traversal)

See [PathRAG Guide](../guides/pathrag.md) for implementation details.

## Semantic Search

SemStreams provides hybrid search combining:

### BM25 (Lexical Search)
- Fast keyword-based matching
- Pure Go implementation (no dependencies)
- Fallback when neural embeddings unavailable

### Neural Embeddings (Semantic Search)
- Vector similarity using transformer models
- Understands semantic relationships
- Powered by [semembed](https://github.com/c360/semembed) service

### Hybrid Strategy
```go
// Try neural embeddings first
if embeddingServiceAvailable {
    results = semanticSearch(query)
} else {
    // Automatic fallback to BM25
    results = bm25Search(query)
}
```

See [Embedding Strategies](../guides/embedding-strategies.md) for configuration.

## Federation

**Federation** enables distributed graph processing across multiple SemStreams instances.

### Use Cases
- **Geographic Distribution**: Process data at edge locations
- **Scale**: Partition large graphs across instances
- **Isolation**: Separate tenant data

### How It Works
1. Each instance maintains a local graph
2. Federated queries span multiple instances
3. Results merged and deduplicated
4. NATS provides cross-instance messaging

See [Federation Guide](../guides/federation.md) for setup.

## Schema Tags

**Schema tags** are inline annotations that define:
- Entity types
- Property semantics
- Validation rules
- Index configuration

### Example

```go
type Sensor struct {
    ID       string  `semstreams:"id,type=schema:Device"`
    Name     string  `semstreams:"prop=schema:name,index=text"`
    Location GeoJSON `semstreams:"prop=schema:location,index=spatial"`
    Value    float64 `semstreams:"prop=schema:value"`
}
```

**Benefits:**
- Type safety (compile-time validation)
- Automatic index configuration
- Self-documenting schemas
- Code generation support

See [Schema Tags Guide](../guides/schema-tags.md) for full syntax.

## Vocabulary System

SemStreams uses standard RDF vocabularies:

| Vocabulary | Purpose | Example |
|------------|---------|---------|
| **schema.org** | General-purpose types | Person, Organization, Event |
| **SSN/SOSA** | Sensors and observations | Sensor, Observation, Platform |
| **GeoSPARQL** | Geospatial | Feature, Geometry |
| **PROV-O** | Provenance | Activity, Entity, Agent |

Custom vocabularies can be defined and registered.

See [Vocabulary Reference](../reference/vocabulary.md) for details.

## NATS as the Backbone

**NATS JetStream** provides:
- **Streaming**: Persistent message streams
- **KV Store**: Entity and metadata storage
- **Object Store**: Large binary storage
- **Pub/Sub**: Event distribution

### Key Patterns

```
Streams:
  entities.events        → Entity change events
  queries.results        → Query result streams

KV Buckets:
  entities               → Current entity state
  indexes.semantic       → Embedding vectors
  indexes.spatial        → Geo indexes

Object Store:
  embeddings.cache       → Cached embedding vectors
```

## Event-Driven Architecture

Services communicate via events:

```
Publisher              NATS              Subscriber
────────────────────────────────────────────────────
semstreams ────entity.created────→ indexmanager
                                   ↓
                            Update semantic index
                                   ↓
                            Publish index.updated
                                   ↓
querymanager ←────index.updated────┘
                                   ↓
                            Invalidate cache
```

**Benefits:**
- Loose coupling
- Async processing
- Horizontal scaling
- Fault tolerance

## Next Steps

- **[Architecture Overview](./architecture.md)**: See how components connect
- **[Integration Guide](../integration/)**: Connect your applications
- **[Deployment Guide](../deployment/)**: Run in production
