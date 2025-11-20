# SemStreams

## Work In Progress

All repos are under **heavy and active** development

## What is SemStreams?

SemStreams is a semantic graph runtime built on event-based flows.

As events flow through components, the runtime automatically builds semantic knowledge graphs from your event streams.

Unlike linear data flows, components can connect anywhere - multiple inputs, branching outputs, flexible routing based on your needs.

### What does "building semantic knowledge graphs" mean?

As events flow through your components, SemStreams automatically:

- Tracks entities mentioned in events (drones, sensors, users, etc.)
- Indexes relationships between entities (drone belongs_to fleet)
- Enables queries across entity connections (find all drones in fleet X)
- Optionally adds semantic search (find similar events by meaning)

**Example:**

```json
Event: {"drone_id": "UAV-001", "fleet": "rescue", "battery": 15}
```

**Graph automatically indexes:**

- Entity: UAV-001 (type: drone)
- Relationship: UAV-001 belongs_to rescue (type: fleet)
- Attribute: battery=15 (enables "find all low battery drones")

**Query later:**

- "Show me all drones in rescue fleet with battery < 20%"
- "Find events similar to this emergency pattern" (semantic search)

This happens in the background. You just route events - the graph builds itself.

---

## Why Semantics at the Source?

**The traditional approach:** Send raw data upstream, apply structure later.

```text
Raw Events → Network → Central Processing → Add Structure → Try to Merge
```

### The problem: Semantic Misalignment

When you have multiple sources—drones, sensors, logs, APIs—each creates its own understanding:

- CRM calls them "customers", logs call them "users", sensors call them "operators"
- One source uses lat/long, another uses geohash, another uses place names
- Relationships are implicit: "Which drone belongs to which fleet?" isn't captured
- When data arrives centrally, you can't reliably merge entities across sources
- Building cross-source knowledge graphs becomes a post-processing nightmare

**SemStreams approach:** Apply semantic structure at ingestion.

```text
Events → Semantic Processing → Aligned Entities → Cross-Source Fusion
```

### The Solution: Semantic interoperability enables fusion

When each source adds semantic structure at ingestion:

- **Consistent entity types**: All sources use agreed ontologies (lightweight or formal)
- **Explicit relationships**: "drone belongs_to fleet" captured where it's created
- **Semantic alignment**: Different sources can map to common semantics before fusion
- **Composable graphs**: Entities from multiple sources can be linked reliably

### Example: Fleet monitoring from 3 sources

```shell
Source 1 (Telemetry): UAV-001 → semantics → Entity(id=drone:uav-001, type=drone)
Source 2 (Maintenance): "uav 001" → semantics → Entity(id=drone:uav-001, type=drone)
Source 3 (Flight Logs): "UAV_001" → semantics → Entity(id=drone:uav-001, type=drone)

→ Fusion: Single unified entity across all sources
→ Query: "Show maintenance history + telemetry + flight logs for UAV-001"
```

Without semantics at source, you get three disconnected entities with different IDs.

**Key advantages:**

✅ **Semantic interoperability** - Sources speak common semantic language
✅ **Cross-source fusion** - Build knowledge graphs spanning heterogeneous sources
✅ **Flexible formality** - Support informal tags to formal ontologies, per-source
✅ **Context preservation** - Relationships captured where context exists
✅ **Composable graphs** - Entity resolution before fusion, not after

**Secondary benefits:**

✅ **Bandwidth efficiency** - Send structured entities, not raw streams
✅ **Edge autonomy** - Process offline, sync semantically-aligned data when connected
✅ **Privacy/compliance** - Semantic filtering at source keeps sensitive data local

This is what we mean by "semantics at the source": applying consistent semantic structure where data is created, enabling cross-source knowledge graphs that would be impossible from post-processed raw data.

---

## The SemStreams Ecosystem

SemStreams is the **core runtime** - the heart of the ecosystem.

**Core Runtime:**

- **[SemStreams](https://github.com/c360/semstreams)** - Event-based semantic graph runtime

**Optional Enhancement Services:**

- **[SemEmbed](https://github.com/c360/semembed)** - Transformer-based vector embeddings (OpenAI API-compatible)
- **[SemSummarize](https://github.com/c360/semsummarize)** - LLM-powered summarization (OpenAI API-compatible)

**Reference Implementations:**

- **SemMem** - Memory infrastructure for agentic coding
- **SemOps** - Common Operating Picture for field operations

Use SemStreams standalone or with enhancements. Reference implementations show real-world production patterns.

### Core capabilities

- Component-based architecture
- Real-time stream processing
- Event routing and transformation via message bus
- Automatic entity graph indexing from event streams
- Relationship tracking (spatial, temporal, predicates)
- Federation with selective sync
- Optional semantic enhancements (SemEmbed, SemSummarize)

---

## Why Message Bus?

**Traditional approaches:**

- Tight coupling between processing stages
- Linear flows - rigid, hard to extend
- Point-to-point connections don't scale

**Message bus pattern:**

- Loose coupling - components don't know about each other
- Publish/subscribe - add new consumers without changing producers
- Non-linear flows - route events based on content, not topology
- Independent scaling - scale busy components independently

### Implementation: NATS + JetStream

SemStreams uses NATS with JetStream for the message bus.

**Why NATS?**

- ✅ Lightweight - minimal overhead, fast message delivery
- ✅ Built-in persistence - JetStream provides durable streams
- ✅ Key-value storage - no external database for state
- ✅ Geographical distribution - multi-region federation built-in
- ✅ Developer friendly - simple ops, no Kafka/Zookeeper complexity

**Don't know NATS? No problem** - SemStreams config abstracts it:

- Declare subjects (topics) in component config
- Components handle subscribe/publish automatically
- Optional: Direct NATS access for advanced use cases

---

## Quick Start: Event Flow Example

**Basic Event Flow:**

```json
{
  "components": {
    "udp_listener": {
      "type": "input",
      "outputs": [{"subject": "raw.telemetry"}]
    },
    "json_parser": {
      "type": "processor",
      "inputs": [{"subject": "raw.telemetry"}],
      "outputs": [{"subject": "parsed.telemetry"}]
    },
    "file_writer": {
      "type": "output",
      "inputs": [{"subject": "parsed.telemetry"}]
    }
  }
}
```

**Event Flow Diagram:**

```text
UDP Listener
    ↓ publish
Message Bus (NATS)
    ↓ subscribe
JSON Parser
    ↓ publish
Message Bus (NATS)
    ├─ subscribe → File Writer
    └─ subscribe → Graph Indexer → Semantic Search
```

**Non-Linear Flow Example:**

```text
Temperature Sensor → sensor.temp.*
                          ↓
                    Message Bus
          ┌────────────┼────────────┐
          ↓            ↓            ↓
    Alert (>100°)  Logger (all)  Metrics (stats)
```

Shows:

- Multiple subscribers on same subject
- Content-based routing (alert only triggers on threshold)
- Components added without changing producers

---

## Progressive Enhancement

### Layer 1: Automatic Graph Building (CORE - Always On)

✅ Events → Entity extraction
✅ Entities → Relationship indexing
✅ Native semantic indexing (Go algorithms, no external services)
✅ Queries across entity connections

**Disable this?** Use vanilla NATS instead and avoid overhead.

### Layer 2: Enhanced Indexing (OPTIONAL)

✅ Spatial indexes (location-based queries)
✅ Temporal indexes (time-based queries)
✅ Predicate indexes (relationship types)
✅ Alias indexes (entity name resolution)

Configure what you need, skip what you don't.

### Layer 3: Semantic Enhancements (OPTIONAL)

**SemEmbed** - Enhanced vector embeddings

- Upgrades semantic similarity search quality
- Resource: Embed integration (local or any OpenAI API-compatible service)
- Native semantic still works without it

**SemSummarize** - AI-powered summarization

- Upgrades entity/event commuity summarization quality
- Resource: LLM integration (local or any OpenAI API-compatible service)
- Native summarization still works without it

### Layer 4: Edge Federation (OPTIONAL)

- Realtime push based
- Selective sync
- Backpressure supported

#### Key Distinction: Native vs Enhanced

| Feature | Native (Included) | Enhanced (Optional) |
|---------|-------------------|---------------------|
| Semantic indexing | ✅ Go algorithms | ✅ Transformer models (SemEmbed) |
| Similarity search | ✅ Basic | ✅ Higher quality |
| Summarization | ✅ Template-based | ✅ LLM-powered (SemSummarize) |
| Works on | ✅ Everywhere (laptop, server, Raspberry Pi) | ⚠️ Requires sufficient compute |
| External services | ❌ None | ✅ SemEmbed (~2GB), SemSummarize (LLM) |

**Both work.** Enhanced = better quality, native = works anywhere.

---

## Edge-First Architecture

SemStreams is designed for edge deployment, not just centralized processing.

**Typical Pattern:**

```text
┌─────────────────────────────────────────┐
│ EDGE (Drone, Vehicle, IoT Gateway)     │
│ • Lightweight SemStreams runtime        │
│ • Local graph building from sensors     │
│ • Local queries (low latency)           │
│ • Selective sync (bandwidth-aware)      │
└──────────────┬──────────────────────────┘
               │ Filtered events only
               ↓
┌─────────────────────────────────────────┐
│ LAPTOP/SERVER (Central Processing)     │
│ • Full SemStreams + semantic services   │
│ • Cross-edge aggregation                │
│ • Long-term graph storage               │
│ • Semantic search across all edges      │
└─────────────────────────────────────────┘
```

**Why This Matters:**

✅ Process at the edge, decide what goes upstream (not everything)
✅ Low-latency local queries (don't wait for server round-trip)
✅ Bandwidth efficiency (send summaries, not raw streams)
✅ Offline operation (edge continues when disconnected)
✅ Privacy/compliance (sensitive data stays local)

---

## When to Use / When Not to Use

### Use SemStreams when

✅ Entity graphs from event streams
✅ Relationships matter (not just individual events)
✅ Edge deployment (process locally, sync selectively)
✅ Temporal/Spatial tracking (entity state over time and space)
✅ Optional: Semantic search, AI summarization

### Use NATS directly when

✅ Simple pub/sub event routing
✅ No entity tracking needed
✅ No edge requirements
✅ You just need message delivery

---

## What's Next

### Quick Start with Docker Compose

**Fastest way to try SemStreams** - Docker Compose examples ready to run:

```bash
# Get running in 30 seconds
cd examples/quickstart/
docker compose up -d

# Verify it's working
curl http://localhost:8080/health
```

**Examples available:**

- **[quickstart/](examples/quickstart/)** - NATS + SemStreams (minimal setup)
- **[with-ui/](examples/with-ui/)** - Add visual flow builder
- **[with-embeddings/](examples/with-embeddings/)** - Add semantic search
- **[production/](examples/production/)** - Hardened edge deployment

See **[Examples Overview](examples/README.md)** for full comparison and setup guides.

### Documentation Structure

**Learning Path** (numbered sequences):

- **[Basics](docs/basics/)** (01-07) - Core concepts, components, routing, first flow, message system, rules engine, examples
- **[Graph](docs/graph/)** (01-04) - Entities, relationships, queries, indexing
- **[Semantic](docs/semantic/)** (01-04) - Overview, SemEmbed, SemSummarize, decision guide
- **[Edge](docs/edge/)** (01-04) - Patterns, constraints, federation, offline
- **[Advanced](docs/advanced/)** (01-07) - PathRAG vs GraphRAG, performance, architecture, production, algorithms, hybrid queries, query strategies

**Reference Documentation** (unnumbered):

- **[Deployment](docs/deployment/)** - Production, TLS, ACME, configuration, monitoring, operations, optional services, federation
- **[Integration](docs/integration/)** - REST API, GraphQL API, NATS events, OpenAPI usage
- **[Development](docs/development/)** - Contributing, testing, agents, writing components, schema tags

**Start here:**

- **New to SemStreams?** → [What is SemStreams](docs/basics/01-what-is-semstreams.md)
- **Want to run it now?** → [Examples](examples/README.md)
- **Building flows?** → [Your First Flow](docs/basics/04-first-flow.md)
- **Production deployment?** → [Production Guide](docs/deployment/production.md)

**Repositories:**
[semstreams](https://github.com/c360/semstreams) (core) · [semembed](https://github.com/c360/semembed) (embeddings) · [semsummarize](https://github.com/c360/semsummarize) (LLM) · [semmem](https://github.com/c360/semmem) (agentic memory) · [semops](https://github.com/c360/semops) (field operations)

---

**SemStreams** - Event-based semantic graph runtime.
