# NATS Event Schemas

SemStreams uses NATS JetStream for event-driven communication between services.

## Event Streams

### entities.events

**Purpose:** Entity lifecycle events (create, update, delete)

**Subjects:**
```
entities.created.{type}     # New entity created
entities.updated.{type}     # Entity modified
entities.deleted.{type}     # Entity removed
```

**Schema:**
```json
{
  "event_type": "entity.created",
  "entity_id": "sensor-123",
  "entity_type": "schema:Device",
  "timestamp": "2025-11-17T15:30:00Z",
  "properties": {
    "name": "Temperature Sensor",
    "value": "72.5"
  },
  "edges": [
    {
      "predicate": "installedIn",
      "target": "room-201"
    }
  ]
}
```

### queries.results

**Purpose:** Query result streams for long-running queries

**Subjects:**
```
queries.results.{query_id}
```

## KV Buckets

### entities

**Purpose:** Current entity state (source of truth)

**Keys:**
```
{entity_id}                 # Entity data
```

### indexes.semantic

**Purpose:** Semantic search index data

**Keys:**
```
vectors.{entity_id}         # Embedding vector
metadata.{entity_id}        # Search metadata
```

### indexes.spatial

**Purpose:** Geospatial index data

**Keys:**
```
locations.{entity_id}       # Geo coordinates
bounds.{region_id}          # Bounding boxes
```

## Object Store

### embeddings.cache

**Purpose:** Cached embedding vectors for large batches

**Keys:**
```
{content_hash}.bin          # Binary embedding data
```

## Event Patterns

### Publish-Subscribe

```go
// Publisher
nc.Publish("entities.created.Device", entityData)

// Subscriber
nc.Subscribe("entities.created.*", handleEntityCreated)
```

### Request-Reply

```go
// Server
nc.QueueSubscribe("query.execute", "workers", handleQuery)

// Client
response, err := nc.Request("query.execute", query, 5*time.Second)
```

### Stream Processing

```go
// Create consumer
js.AddConsumer("entities.events", &nats.ConsumerConfig{
    Durable:       "indexer",
    AckPolicy:     nats.AckExplicitPolicy,
    FilterSubject: "entities.created.*",
})

// Process messages
msgs, _ := sub.Fetch(10)
for _, msg := range msgs {
    process(msg)
    msg.Ack()
}
```

## Integration Examples

See individual service documentation for implementation details:
- [semstreams](https://github.com/c360/semstreams) - Entity event publishing
- [indexmanager](https://github.com/c360/semstreams/tree/main/processor/graph/indexmanager) - Event consumption
- [semmem](https://github.com/c360/semmem) - Context management events
