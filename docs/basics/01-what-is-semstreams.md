# What is SemStreams?

## A semantic graph runtime built on event-based flows

---

## The Core Concept

SemStreams automatically builds semantic knowledge graphs from your event streams.

As events flow through your components:

1. **Events arrive** (UDP, HTTP, MQTT, etc.)
2. **Components process** (parse, filter, transform)
3. **Graph builds automatically** (entities, relationships, indexes)
4. **Query anytime** (relationships, semantic search, analytics)

**The key difference:** You don't manually build graphs. The runtime builds them in real time from your events.

---

## A Simple Example

**Your events** (with required `entity_id` and `entity_type` fields):

```json
{"entity_id": "acme.rescue.robotics.gcs1.drone.001", "entity_type": "robotics.drone", "fleet": "rescue", "battery": 85.2}
{"entity_id": "acme.rescue.robotics.gcs1.drone.002", "entity_type": "robotics.drone", "fleet": "rescue", "battery": 15.4}
{"entity_id": "acme.rescue.ops.hq.fleet.rescue", "entity_type": "ops.fleet", "status": "active", "location": "base-west"}
```

**Entity ID Format:** SemStreams uses 6-part federated entity IDs:

```text
org.platform.domain.system.type.instance
 │      │       │      │     │      │
 │      │       │      │     │      └─ Instance identifier (001, 002, rescue)
 │      │       │      │     └─ Entity type (drone, fleet)
 │      │       │      └─ Source system (gcs1, hq)
 │      │       └─ Data domain (robotics, ops)
 │      └─ Platform identifier (rescue)
 └─ Organization namespace (acme)
```

**What SemStreams creates:**

```text
Entities:
  - acme.rescue.robotics.gcs1.drone.001 (type: robotics.drone)
  - acme.rescue.robotics.gcs1.drone.002 (type: robotics.drone)
  - acme.rescue.ops.hq.fleet.rescue (type: ops.fleet)

Relationships (from triples):
  - drone.001 → belongs_to → fleet.rescue
  - drone.002 → belongs_to → fleet.rescue
  - fleet.rescue → located_at → base-west

Indexes (NATS KV buckets):
  - PREDICATE_INDEX: "belongs_to" → [drone.001, drone.002]
  - INCOMING_INDEX: fleet.rescue ← [drone.001, drone.002]
  - ALIAS_INDEX: "ALPHA-1" → drone.001 (searchable names)
  - SPATIAL_INDEX: geohash → [entities with geo.location.*]
  - TEMPORAL_INDEX: time ranges → [entities with time.lifecycle.*]
```

**Now you can query:**

```bash
# Find all drones in rescue fleet (PathRAG - relationship traversal)
curl "http://localhost:8080/api/v1/graph/traverse?start=acme.rescue.ops.hq.fleet.rescue&direction=incoming"

# Find entity by alias (call sign, serial number, etc.)
curl "http://localhost:8080/api/v1/graph/resolve?alias=ALPHA-1"

# Find entities semantically similar to "emergency" (GraphRAG)
curl -X POST http://localhost:8080/api/v1/search/semantic -d '{"query": "emergency response drone"}'
```

---

## Why Event-Based Flows?

Traditional approaches use **linear data flows** (also called "pipelines"):

```text
Input → Step1 → Step2 → Step3 → Output
```

**Problems:**

- Tight coupling between steps
- Hard to add new outputs without changing pipeline
- Can't route based on message content
- Difficult to scale individual steps

**SemStreams uses event-based flows:**

```text
Input
  ↓ publish
Message Bus
  ├─ subscribe → Step1
  │     ↓ publish
  ├─ subscribe → Step2
  ├─ subscribe → Graph Processor
  ├─ subscribe → Output1
  └─ subscribe → Output2
```

**Advantages:**

- **Loose coupling:** Components don't know about each other
- **Non-linear flows:** Multiple subscribers on same subject
- **Dynamic routing:** Route events based on content, not topology
- **Independent scaling:** Scale busy components without affecting others

---

## The Message Bus (NATS)

SemStreams uses **NATS + JetStream** as the message bus implementation.

**Why NATS?**

✅ **Lightweight** - Minimal overhead, fast message delivery
✅ **Built-in persistence** - JetStream provides durable streams
✅ **Key-value storage** - No external database needed for state
✅ **Simple operations** - No Kafka/Zookeeper complexity

**Don't know NATS?** No problem - SemStreams config abstracts it:

```json
{
  "components": {
    "udp_input": {
      "outputs": [{"subject": "raw.udp"}]
    },
    "parser": {
      "inputs": [{"subject": "raw.udp"}],
      "outputs": [{"subject": "parsed.json"}]
    }
  }
}
```

Components handle subscribe/publish automatically. NATS is an implementation detail.

---

## The Graph is THE Product

**Can you disable graph building?**

Technically yes. **Should you?** No - you're using the wrong tool.

**SemStreams = Graph runtime + Event flows**
**NATS = Event flows only**

If you don't need graphs, use NATS directly. It's simpler and lighter.

The "Sem" in SemStreams is the **semantic graph** - it's not an optional add-on.

---

## Native vs Enhanced Semantic

SemStreams has **two levels** of semantic capabilities:

### Native (Always Available)

**What it is:** Go algorithms (TF-IDF, BM25) built into the runtime that work well with structured data.

**Works on:**

- ✅ Laptop
- ✅ Server
- ✅ Raspberry Pi
- ✅ Any platform Go runs on

**Quality:** Good for basic similarity search.

**Dependencies:** None (pure Go)

### Enhanced (Optional)

**What it is:** External services for higher quality

**SemEmbed:** Transformer-based vector embeddings

- Better semantic similarity than native
- ~2GB RAM, GPU optional
- OpenAI API-compatible

**SemSummarize:** LLM-powered summarization

- Richer entity descriptions than native
- Any OpenAI API-compatible service (local or cloud)

**Works on:**

- ✅ Laptop (CPU works, GPU better)
- ✅ Server (production use)
- ❌ Raspberry Pi (insufficient resources)

**Key point:** Both native and enhanced work. Enhanced = better quality, not required for functionality.

---

## Progressive Enhancement Layers

SemStreams is designed in layers. Use what you need:

### Layer 1: Event-Based Flow (CORE)

```text
Events → Message Bus → Components
```

**Always on.** This is the foundation.

### Layer 2: Graph Building (CORE)

```text
Events → Graph Processor → Entity Indexes
```

**Always on.** Automatic entity and relationship tracking.

### Layer 3: Native Semantic Indexing (CORE)

```text
Entities → Go Algorithms (TF-IDF, BM25) → Similarity Scores
```

**Always on.** Basic semantic search without external services.

### Layer 4: Enhanced Indexing (OPTIONAL)

```text
Enable: Spatial, Temporal, Alias indexes
```

**Configure what you need.** Turn on geospatial queries, time-based queries, etc.

### Layer 5: Enhanced Semantic (OPTIONAL)

```text
Entities → SemEmbed → High-quality vector embeddings
Entities → SemSummarize → LLM-powered descriptions
```

**Better quality, not required.** Upgrade semantic capabilities when you need them.

### Layer 6: Edge Federation (OPTIONAL)

```text
Edge Runtime → Selective Sync → Cloud Hub
```

**Deployment pattern.** Run lightweight on edge, full stack when/where able.

---

## When to Use SemStreams

### ✅ Use SemStreams when

- You need **knowledge graphs from event streams**
- **Relationships matter** (not just individual events)
- You want **edge deployment** (process locally, sync selectively)
- You need **temporal and spatial tracking** (entity state over time and space)
- Optionally: **Semantic search, AI summarization**

### ⚠️ Use NATS directly when

- Simple pub/sub event routing
- No entity tracking needed
- No edge requirements
- You just need message delivery

### ❌ SemStreams is NOT for

| Your Need | Better Alternative | Why |
|-----------|-------------------|-----|
| Pure time-series metrics | InfluxDB, Prometheus | Optimized for time-based aggregation |
| Pure full-text search | Elasticsearch, Meilisearch | Mature ranking, highlighting, faceting |
| Traditional OLAP | ClickHouse, Druid | Columnar storage, SQL analytics |
| Simple key-value lookups | Redis, Memcached | Faster, simpler for basic get/set |
| Batch ETL pipelines | Airflow + dbt + Snowflake | Purpose-built for batch workflows |

---

## Key Terminology

**Entity:** A tracked object (drone, user, sensor, document, etc.)

**Relationship:** A connection between entities (belongs_to, located_at, depends_on)

**Predicate:** The type of relationship (the verb in "UAV-001 belongs_to fleet-rescue")

**Subject:** A NATS topic (e.g., "events.graph.entity.drone")

**Component:** A processing element (input, processor, output)

**PathRAG:** Relationship traversal queries ("find all entities that depend on this")

**GraphRAG:** Semantic similarity queries ("find entities semantically similar to this text")

---

## Next Steps

Ready to get started?

- **[Components](02-components.md)** - Understand component types
- **[Routing](03-routing.md)** - Learn message routing patterns
- **[First Flow](04-first-flow.md)** - Build your first event flow

Want to understand configuration options?

- **[Query Fundamentals](../advanced/01-query-fundamentals.md)** - PathRAG vs GraphRAG decisions
- **[Graph Indexing](../graph/04-indexing.md)** - Index types and when to use them
- **[Edge Deployment](../edge/01-patterns.md)** - Edge-to-cloud patterns

---

**SemStreams builds graphs from events** - Focus on your event flows, the graph builds itself.
