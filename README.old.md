# SemStreams Ecosystem Documentation

## Work In Progress

**All repos in ecosystem are under heavy and active dev.**

## Real-time semantic stream processing at the edge

Welcome to the SemStreams ecosystem documentation. This repository serves as the single source of truth for understanding the SemStreams platform architecture, concepts, and integration patterns.

## What is SemStreams?

SemStreams is a distributed semantic stream processing platform designed for edge deployments. It combines real-time graph processing, semantic search, and knowledge extraction to enable intelligent data processing at scale.

## Is SemStreams Right for You?

**TL;DR**: SemStreams excels when you need **structured relationships (PathRAG) OR semantic search (GraphRAG) OR both** - but you don't need both for every use case. The flexibility to use only what you need is a core feature.

### Understanding Fit: It's About Your Data

Different data types map to different capabilities. Here's how to think about it:

| Data Type | PathRAG (Relationships) | GraphRAG (Semantic Search) | Best Configuration |
|-----------|------------------------|---------------------------|-------------------|
| **Robotics telemetry** (battery: 85.2, gps: [lat,lon]) | ✅ Excellent (topology, "what's near?") | ❌ Poor (no text to search) | PathRAG only, skip embeddings |
| **Robotics + incident logs** (telemetry + "battery degraded...") | ✅ Excellent (which drone, mission) | ✅ Excellent (find similar incidents) | Full stack (PathRAG + GraphRAG) |
| **Software specs/docs** (dependencies + descriptions) | ✅ Excellent (dependency chains) | ✅ Excellent (semantic similarity) | Full stack (PathRAG + GraphRAG) |
| **IoT sensor networks** (temp, humidity, device IDs) | ✅ Excellent (device topology) | ❌ Poor (numeric only) | PathRAG only, skip embeddings |
| **DevOps service mesh** (services + dependencies) | ✅ Excellent (call graphs) | ⚠️ Maybe (if you have logs/docs) | PathRAG primary, GraphRAG optional |
| **Time-series only** (metrics over time) | ❌ Poor fit | ❌ Poor fit | **Use InfluxDB/Prometheus instead** |

### When SemStreams Excels (Full Stack)

**Best fit when you have BOTH structure AND semantics:**

```yaml
# Example: Software engineering knowledge graph
Entity: Spec "Authentication System"
Properties:
  - title: "OAuth2 Authentication Implementation"       # ← Text for GraphRAG
  - description: "Implements RFC 6749 with PKCE..."     # ← Text for GraphRAG
  - status: "active"                                     # ← Metadata
Relationships:
  - depends_on: ["User Management Spec"]                # ← Structure for PathRAG
  - implemented_by: ["PR-123", "PR-456"]                # ← Structure for PathRAG

Queries you can run:
- PathRAG: "Show all specs that depend on Auth" (relationship traversal)
- GraphRAG: "Find specs similar to OAuth2" (semantic similarity)
- Hybrid: "Find auth-related specs and their dependencies"
```

**Use cases:**

- Knowledge management (docs with dependencies)
- Software architecture (specs, PRs, issues with relationships)
- Research systems (papers with citations + semantic search)
- Operational intelligence (incidents + infrastructure topology)

### When to Use PathRAG Only (Skip GraphRAG)

**Best fit when you have structured relationships but minimal text:**

```yaml
# Example: IoT device fleet
Entity: Drone "drone-001"
Properties:
  - battery: 85.2              # ← Numeric, not searchable text
  - location: [37.7749, -122.4194]
  - status: "active"
Relationships:
  - near: ["drone-002", "drone-003"]          # ← Structure for PathRAG
  - reports_to: ["base-station-west"]         # ← Structure for PathRAG

Queries you can run:
- PathRAG: "Show all drones near drone-001" (spatial relationships)
- PathRAG: "Find fleet topology for base-station-west" (hierarchy)
- GraphRAG: ❌ Not useful (no semantic content)

Configuration: Skip semantic_enrichment processor, no embeddings
Memory savings: ~2KB → ~200 bytes per entity (10x reduction)
```

**Use cases:**

- IoT sensor networks (device topology, no rich descriptions)
- Robotics telemetry (fleet coordination, spatial relationships)
- Service mesh (microservices dependencies, no semantic search)
- Network topology (routers, switches, connections)

**Configuration tip:**

```yaml
processors:
  - type: entity_extractor
  - type: relationship_builder
  # SKIP: semantic_enrichment (no embeddings needed)

storage:
  - type: nats_kv
    config:
      enable_graph_index: true      # PathRAG ✅
      enable_semantic_index: false  # GraphRAG ❌ (save memory)
```

### When to Use GraphRAG Only (Skip PathRAG)

**Best fit when you have rich text but minimal explicit relationships:**

```yaml
# Example: Document search system
Entity: Doc "incident-report-2024-03-15"
Properties:
  - title: "Database Connection Pool Exhaustion"
  - content: "At 3:47 AM PST, we observed..."    # ← Rich text for GraphRAG
  - tags: ["database", "production", "p1"]
Relationships:
  # Minimal or no explicit edges between docs

Queries you can run:
- GraphRAG: "Find incidents similar to connection pool issues"
- GraphRAG: "Summarize all database-related incidents"
- PathRAG: ❌ Limited (no relationship structure to traverse)

Configuration: Skip relationship_builder, focus on embeddings
```

**Use cases:**

- Document search (rich text, minimal relationships)
- Log analysis (semantic patterns in unstructured logs)
- Content recommendation (similarity-based)

### When SemStreams Is NOT the Right Fit

**Be honest with yourself - sometimes alternatives are better:**

| Your Need | Better Alternative | Why |
|-----------|-------------------|-----|
| **Pure time-series metrics** (CPU, memory over time) | InfluxDB, Prometheus, TimescaleDB | Optimized for time-based aggregation, downsampling |
| **Pure full-text search** (search engine) | Elasticsearch, Meilisearch | Mature ranking, highlighting, faceting |
| **Traditional OLAP** (sum, group, aggregate) | ClickHouse, Druid | Columnar storage, SQL analytics |
| **Simple key-value lookups** (cache, session store) | Redis, Memcached | Faster, simpler for basic get/set |
| **Batch ETL pipelines** (nightly data warehouse loads) | Airflow + dbt + Snowflake | Purpose-built for batch workflows |

**Anti-patterns (common mistakes):**

1. **Embedding everything** in high-volume telemetry
   - ❌ Problem: 10 drones × 10 Hz = 600 embeddings/sec → 103GB in 24 hours
   - ✅ Solution: Use PathRAG only, skip semantic_enrichment

2. **Using it as a general database**
   - ❌ Problem: No SQL, no transactions, no joins
   - ✅ Solution: Use PostgreSQL for transactional data, SemStreams for graph queries

3. **Trying semantic search on numeric-only data**
   - ❌ Problem: "battery: 85.2" has no semantic meaning
   - ✅ Solution: Skip GraphRAG, use PathRAG for relationships

4. **Not filtering by entity type**
   - ❌ Problem: Embedding noise data (heartbeats, health checks)
   - ✅ Solution: Configure type filters (see [Embedding Strategies](docs/guides/embedding-strategies.md))

### The Killer Feature: Use What You Need

**SemStreams is composable** - you're not locked into using every capability:

```yaml
# Configuration A: PathRAG only (IoT fleet)
processors:
  - type: entity_extractor
  - type: relationship_builder

# Configuration B: GraphRAG only (document search)
processors:
  - type: entity_extractor
  - type: semantic_enrichment

# Configuration C: Full stack (software knowledge graph)
processors:
  - type: entity_extractor
  - type: relationship_builder
  - type: semantic_enrichment

# Configuration D: Just stream processing (no graph)
processors:
  - type: json_parser
  - type: custom_validator
outputs:
  - type: nats_publisher  # No graph storage
```

**Pick your configuration based on your data characteristics, not the framework's capabilities.**

### Decision Tree

```text
Do you have structured relationships (dependencies, topology, hierarchy)?
│
├─ YES + Rich text descriptions (specs, docs, incidents)
│  └─ ✅ Full Stack (PathRAG + GraphRAG)
│
├─ YES + Minimal text (sensor IDs, numeric telemetry)
│  └─ ✅ PathRAG Only (skip embeddings, save memory)
│
├─ NO + Rich text (documents, logs, articles)
│  └─ ✅ GraphRAG Only (semantic search)
│
└─ NO + Minimal text (pure metrics, time-series)
   └─ ❌ Consider InfluxDB/Prometheus instead
```

**Still unsure?** Start with PathRAG only (lowest overhead), add GraphRAG later if you need semantic search. Schema evolution is built in - you can change your mind.

**Core Capabilities:**

### Real-Time Stream Processing

**Flow-based architecture with first-class component types:**

- **Inputs**: Ingest from any source (UDP, TCP, HTTP, WebSocket, File, NATS, custom)
- **Processors**: Transform, enrich, extract, validate data at any point in the flow
- **Outputs**: Send results anywhere (NATS, HTTP, files, databases, custom sinks)
- **Storage**: Persist entities and graphs (NATS KV, custom backends)
- **Gateways**: Expose query interfaces (GraphQL, REST, NATS subjects)

**Build custom flows by composing components:**

```yaml
# Example: Multi-stage processing with branching
inputs:
  - type: udp_listener
    port: 9001

processors:
  - type: json_parser
  - type: entity_extractor
  - type: relationship_builder
  - type: semantic_enrichment  # Optional: hook in embeddings
  - type: custom_validator     # Hook in your own logic

outputs:
  - type: nats_publisher
  - type: http_webhook        # Send to multiple destinations

storage:
  - type: nats_kv             # Entity graph storage

gateways:
  - type: graphql             # Query interface
  - type: rest_api            # Alternative query interface
```

**Hook in at any point**: Add processors, outputs, or custom logic anywhere in the flow to build exactly the processing flow you need.

### How Data Flows: The Message System

**All data in SemStreams flows as structured messages** - not raw bytes or untyped JSON, but typed, validated containers with behavioral capabilities.

**Message Structure:**

```go
type Message interface {
    ID() string            // Unique identifier (UUID)
    Type() Type           // Schema (domain.category.version)
    Payload() Payload     // Your data + optional behaviors
    Meta() Meta          // Timestamps, source, federation info
    Hash() string        // Content-based deduplication
    Validate() error     // Validation at message + payload level
}
```

**Type System** enables routing and schema evolution:

```go
Type{
    Domain:   "sensors",      // Data domain
    Category: "temperature",  // Message category
    Version:  "v1",          // Schema version
}
// Results in NATS subject: "sensors.temperature.v1"
```

**Behavioral Interfaces** - Messages discover capabilities at runtime:

- **Graphable**: Entities for knowledge graph storage (`EntityID()`, `Triples()`)
- **Locatable**: Geographic coordinates for spatial indexing (`Location()`)
- **Timeable**: Event timestamps for time-series (`Timestamp()`)
- **Observable**: Sensor readings (`ObservedEntity()`, `ObservedValue()`)
- **Correlatable**: Distributed tracing (`CorrelationID()`, `TraceID()`)
- **Processable**: Priority and deadlines (`Priority()`, `Deadline()`)

**Runtime capability discovery:**

```go
// Components discover what messages can do
if graphable, ok := msg.Payload().(Graphable); ok {
    entityID := graphable.EntityID()
    triples := graphable.Triples()
    // Store in knowledge graph
}

if locatable, ok := msg.Payload().(Locatable); ok {
    lat, lon := locatable.Location()
    // Build spatial index
}
```

**Why this matters:**

- ✅ **Type-safe** - Compile-time safety with runtime flexibility
- ✅ **Extensible** - Add new behaviors without breaking existing code
- ✅ **Routable** - Message types map directly to NATS subjects
- ✅ **Discoverable** - Components learn what messages can do at runtime
- ✅ **Validated** - Messages validate themselves before processing

See [Message System Guide](docs/guides/message-system.md) for details on creating custom payloads and using behavioral interfaces.

### Dynamic Knowledge Graphs: Stream to Graph

**SemStreams automatically builds and maintains knowledge graphs from streaming data** - no batch ETL, no manual modeling, just real-time semantic graphs.

**How it works**:

```text
Incoming Stream        Entity Extraction       Knowledge Graph
──────────────────    ─────────────────────    ───────────────────
UDP packets      →    Graphable interface  →   Entity nodes
JSON events      →    EntityID() method    →   + Properties
WebSocket data   →    Triples() method     →   + Relationships
NATS messages    →    Auto-storage         →   + Temporal metadata

Result: Queryable semantic graph updated in real-time
```

**Key Characteristics**:

1. **Dynamic** - Graphs update as data streams in (not batch processed)
2. **Semantic** - Entities and relationships have meaning (via vocabulary)
3. **Temporal** - Full history tracked (every update timestamped)
4. **Queryable** - PathRAG for relationships, GraphRAG for semantic search
5. **Federated** - Sync graphs across edge/cloud instances

#### Example: IoT Sensor Graph

```go
// Incoming message (UDP packet from drone)
msg := &DroneStatusMessage{
    DroneID: "drone-001",
    Battery: 85.5,
    Location: LatLon{37.7749, -122.4194},
}

// Message implements Graphable → automatic graph construction
triples := msg.Triples()
// Returns:
// [
//   {Subject: "drone-001", Predicate: "robotics.battery.level", Object: 85.5},
//   {Subject: "drone-001", Predicate: "geo.location.latitude", Object: 37.7749},
//   {Subject: "drone-001", Predicate: "graph.rel.near", Object: "drone-002"},
// ]

// Stored in NATS KV → queryable via PathRAG/GraphRAG immediately
```

**From Stream to Graph in < 5ms**:

1. Message arrives (UDP/TCP/WebSocket/NATS)
2. `Graphable` interface extracts entities + relationships
3. Triples persisted to NATS KV
4. Graph indexes updated (PathRAG + GraphRAG)
5. Ready for query

**What you get**:

- **Entity graph** - Nodes (drones, sensors, specs) with typed properties
- **Relationship graph** - Edges (depends_on, near, implements) between entities
- **Semantic index** - BM25/neural embeddings for similarity search
- **Community structure** - LPA clustering for GraphRAG global search
- **Temporal views** - Query graph state at any point in time

**Why "dynamic" matters**:

Traditional knowledge graphs require batch ETL pipelines that run hourly/daily. **SemStreams graphs update in real-time** as events happen - query the current state of your world, not yesterday's snapshot.

**Use cases**:

- **IoT**: Real-time device topology and status
- **Robotics**: Mission planning with live fleet status
- **DevOps**: Live service dependency graphs
- **Knowledge management**: Continuously updated documentation graphs

See [PathRAG Guide](docs/guides/pathrag.md) and [GraphRAG Guide](docs/guides/graphrag.md) for querying your dynamic knowledge graph.

### Rules Engine: Dynamic Graph Enrichment

**The missing piece**: Dynamic knowledge graphs need BOTH ingestion-time intelligence (rules) AND query-time intelligence (PathRAG/GraphRAG).

**Without rules, you're limited to:**

- Static entity extraction via `Graphable` interface
- No conditional logic or pattern detection
- No derived entities or relationships
- Manual graph construction

**With rules, you get:**

- Conditional entity creation ("if battery < 20%, create alert")
- Derived relationships ("if 5+ shared dependencies, link as related")
- Semantic enrichment ("if mentions OAuth, tag as security")
- Pattern detection ("if 3 errors in 5 min, create incident")
- Automatic graph evolution based on data patterns

#### The Complete Picture

```text
Ingestion Time                    Query Time
──────────────────               ──────────────────
Stream → Graphable →  Rules  →   Graph  →  PathRAG + GraphRAG
        (extract)    (enrich)   (store)    (query)
                        ↓
              Conditional logic
              Pattern detection
              Derived entities
              Smart relationships
```

#### Rules Architecture

Rules are implemented using a factory pattern where each rule type (e.g., battery_monitor) has its own RuleFactory that creates Rule instances.

**How it works:**

1. **Define Rule** (JSON):
   ```json
   {
     "id": "low-battery-alert",
     "type": "battery_monitor",
     "name": "Low Battery Alert",
     "enabled": true,
     "conditions": [
       {
         "field": "robotics.battery.level",
         "operator": "lte",
         "value": 20.0
       }
     ],
     "logic": "and",
     "cooldown": "2m"
   }
   ```

2. **Implement Rule** (Go):
   ```go
   func (r *BatteryMonitorRule) ExecuteEvents(messages []message.Message) ([]gtypes.Event, error) {
       // Direct event generation in Go code
       event := gtypes.Event{
           Type:     gtypes.EventAlert,
           EntityID: "alert.battery." + deviceID,
           Properties: map[string]interface{}{
               "severity": "high",
               "level":    batteryLevel,
           },
       }
       return []gtypes.Event{event}, nil
   }
   ```

3. **Rule Evaluates** - Conditions checked against messages
4. **Events Generated** - ExecuteEvents() creates gtypes.Event array
5. **Events Published** - Sent to NATS graph subjects

**For complete rule implementation details**, see:
- [Rules Engine Guide](docs/guides/rules-engine.md) - User documentation with examples
- [SPEC-001: Rules Engine](docs/specs/SPEC-001-generic-rules-engine.md) - Technical specification

#### Why Rules + GraphRAG = Power

**Ingestion-time rules structure the graph:**

- Add semantic tags → GraphRAG finds similar entities more accurately
- Create derived edges → PathRAG traverses richer relationships
- Detect patterns → Graph contains insights, not just raw data

**Query-time intelligence leverages structure:**

- GraphRAG semantic search benefits from rule-added tags/categories
- PathRAG traversal uses rule-generated relationships
- Communities reflect rule-based categorization

**Example flow:**

1. Spec entity created with content about "OAuth2 implementation"
2. **Rule fires** → Adds `security` tag, creates edge to `category:security`
3. Entity stored in graph with enriched metadata
4. **GraphRAG query** → "Find security-related specs" returns this entity (tag match)
5. **PathRAG query** → "Show all specs in security category" traverses the edge

**Status**: Rules engine implemented and in use. See comprehensive guides above for creating custom rule types.

### Semantic Vocabulary: The Knowledge Graph Language

**Predicates define properties and relationships** in the knowledge graph using clean dotted notation.

**Pragmatic Semantic Web Approach:**

- **Internal**: Dotted notation everywhere (`domain.category.property`)
- **External**: Optional IRI mappings for standards compliance
- **No Leakage**: Standards complexity stays at API boundaries

**Predicate Structure:**

```go
"sensor.temperature.celsius"   // domain.category.property
"geo.location.latitude"        // Clean, human-readable
"graph.rel.depends_on"         // Relationship predicate
```

**NATS-Friendly Wildcards:**

```go
// Query all temperature predicates
nc.Subscribe("sensor.temperature.*", handler)

// Query entire sensor domain
nc.Subscribe("sensor.>", handler)
```

**Standard Vocabulary Support** - Optional IRI mappings for RDF/OGC export:

- **OWL**: `owl:sameAs`, `owl:equivalentClass`
- **SKOS**: `skos:prefLabel`, `skos:altLabel`
- **Dublin Core**: `dc:identifier`, `dc:references`
- **Schema.org**: `schema:name`, `schema:sameAs`
- **SSN/SOSA**: Sensor ontology for IoT/robotics

**Define custom vocabularies:**

```go
const BatteryLevel = "robotics.battery.level"

func init() {
    vocabulary.Register(BatteryLevel,
        vocabulary.WithDescription("Battery charge percentage"),
        vocabulary.WithDataType("float64"),
        vocabulary.WithUnits("percent"),
        vocabulary.WithRange("0-100"),
        vocabulary.WithIRI("http://schema.org/batteryLevel"))  // Optional
}
```

**Use in triples:**

```go
triple := message.Triple{
    Subject:   droneID,
    Predicate: BatteryLevel,  // Dotted notation
    Object:    85.5,
}
// Internal: "robotics.battery.level"
// RDF export: "http://schema.org/batteryLevel" (if mapped)
```

See [Vocabulary System Guide](docs/guides/vocabulary-system.md) for predicate registration, alias types, and standards mappings.

### Graph Query & Search

- **PathRAG**: Fast graph traversal for tracing dependencies, impact chains, and spatial relationships ([guide](docs/guides/pathrag.md))
- **GraphRAG**: Semantic search using hierarchical community detection with local/global modes ([guide](docs/guides/graphrag.md))
- **Hybrid Queries**: Combine structural + semantic approaches for comprehensive results ([patterns](docs/guides/hybrid-queries.md))
- **Not sure which to use?** See [Choosing Your Query Strategy](docs/guides/choosing-query-strategy.md)

### Progressive Enhancement Philosophy

**Works offline with zero dependencies, gets better with optional services:**

- **Embeddings**: BM25 (pure Go, always works) → HTTP neural embeddings (optional enhancement)
- **Summaries**: Statistical TF-IDF (always works) → Async LLM enrichment (optional enhancement)
- **Storage**: Local NATS KV (always works) → Distributed federation (optional enhancement)

This means SemStreams runs in constrained environments while gracefully scaling up when resources are available.

### Distributed & Edge-Ready

- **Federation**: Multi-instance graph synchronization
- **Edge Deployment**: Minimal resource footprint, offline capable
- **Cloud Scale**: Optional neural embeddings, LLM services, distributed storage

## Ecosystem Components

| Service | Purpose | Repository |
|---------|---------|------------|
| **semstreams** | Core stream processing engine | [semstreams](https://github.com/c360/semstreams) |
| **semstreams-ui** | Web-based UI for flow building and monitoring | [semstreams-ui](https://github.com/c360/semstreams-ui) |
| **semembed** | Embedding service (Rust + fastembed) | [semembed](https://github.com/c360/semembed) |
| **semsummarize** | Summarization service (Rust + Candle) | [semsummarize](https://github.com/c360/semsummarize) |
| **semmem** | Memory infrastructure for agentic coding (reference implementation using github spec-kit) | [semmem](https://github.com/c360/semmem) |
| **semops** | Semantic common operational picture (reference implementation) | [semops](https://github.com/c360/semops) |

## Quick Start

```bash
# Clone the ecosystem
git clone https://github.com/c360/semstreams
git clone https://github.com/c360/semstreams-ui
git clone https://github.com/c360/semembed

# Start core services with Docker Compose
cd semstreams
docker compose -f docker-compose.dev.yml up
```

**Next Steps:**

- [Getting Started Guide](docs/getting-started/quickstart.md) - 5-minute quickstart
- [Architecture Overview](docs/getting-started/architecture.md) - High-level system design
- [Core Concepts](docs/getting-started/concepts.md) - Semantic streaming fundamentals

## Documentation Structure

### 📚 [Getting Started](docs/getting-started/)

- [Quickstart Guide](docs/getting-started/quickstart.md) - Get up and running in 5 minutes
- [Architecture Overview](docs/getting-started/architecture.md) - System architecture and design
- [Core Concepts](docs/getting-started/concepts.md) - Semantic streaming fundamentals

### 📖 [Guides](docs/guides/)

**Core Concepts**:

- [Message System Guide](docs/guides/message-system.md) - How data flows through SemStreams (typed messages, behavioral interfaces)
- [Vocabulary System Guide](docs/guides/vocabulary-system.md) - Semantic predicates for the knowledge graph (dotted notation, IRI mappings)

**Query Strategies** - Choosing the right approach:

- [Choosing Your Query Strategy](docs/guides/choosing-query-strategy.md) - **START HERE** - Decision tree for PathRAG vs GraphRAG
- [PathRAG Guide](docs/guides/pathrag.md) - Graph traversal for dependencies and relationships
- [GraphRAG Guide](docs/guides/graphrag.md) - Hierarchical community search (local/global) for semantic similarity
- [Hybrid Query Patterns](docs/guides/hybrid-queries.md) - Combining PathRAG + GraphRAG for maximum insight

**Other Guides**:

- [Federation Guide](docs/guides/federation.md) - Distributed graph processing
- [Embedding Strategies](docs/guides/embedding-strategies.md) - Semantic search configuration
- [Schema Tags](docs/guides/schema-tags.md) - Schema annotation system

### 🔌 [Integration](docs/integration/)

- [NATS Events](docs/integration/nats-events.md) - Event schemas and messaging
- [GraphQL API](docs/integration/graphql-api.md) - GraphQL endpoint contracts
- [REST API](docs/integration/rest-api.md) - OpenAPI specifications
- [Service Mesh](docs/integration/service-mesh.md) - Inter-service communication

### 🚀 [Deployment](docs/deployment/)

- [Production Setup](docs/deployment/production.md) - Production architecture
- [Operations Guide](docs/deployment/operations.md) - Running and maintaining
- [Configuration Reference](docs/deployment/configuration.md) - All configuration options
- [TLS Setup](docs/deployment/tls-setup.md) - ACME/Let's Encrypt configuration
- [Monitoring](docs/deployment/monitoring.md) - Observability and metrics

### 📋 [Reference](docs/reference/)

- [Algorithm Reference](docs/reference/algorithms.md) - Plain-language guide to BM25, LPA, TF-IDF, Dijkstra's
- [Schema Versioning](docs/reference/schema-versioning.md) - Schema evolution strategy
- [Port Allocation](docs/reference/port-allocation.md) - Service port registry
- [Configuration Schema](docs/reference/configuration-reference.md) - Complete config reference

### 🛠️ [Development](docs/development/)

- [Agent Workflow](docs/development/agents.md) - TDD workflow with specialized agents
- [Contributing Guide](docs/development/contributing.md) - How to contribute
- [Testing Philosophy](docs/development/testing.md) - Testing standards and patterns

## Architecture Highlights

### Component-Based Flow Architecture

**SemStreams uses a flexible, composable component model:**

```text
┌─────────────────────────────────────────────────────────────┐
│                    INPUTS (Ingest)                           │
│  UDP │ TCP │ HTTP │ WebSocket │ File │ NATS │ Custom        │
└───────────────┬─────────────────────────────────────────────┘
                │
                ├─────────────────────────────────────┐
                │                                     │
┌───────────────▼────────┐              ┌─────────────▼──────┐
│  PROCESSORS (Transform) │              │ PROCESSORS (Branch)│
│  • Parser               │              │ • Filter           │
│  • Entity Extractor     │              │ • Aggregator       │
│  • Relationship Builder │              │ • Custom Logic     │
│  • Validator            │              └─────────┬──────────┘
│  • Enrichment           │                        │
│  • Custom Hooks         │                        │
└───────────┬─────────────┘                        │
            │                                      │
            ├──────────┬───────────────────────────┘
            │          │
┌───────────▼─────┐    ┌───▼────────┐
│  STORAGE        │    │  OUTPUTS   │
│  • NATS KV      │    │  • NATS    │
│  • Graph Index  │    │  • HTTP    │
│  • Communities  │    │  • File    │
│  • Custom       │    │  • Custom  │
└───────┬─────────┘    └────────────┘
        │
┌───────▼──────────────────────────────────────────────────────┐
│                    GATEWAYS (Query)                           │
│  GraphQL │ REST API │ NATS Subjects │ Custom Endpoints       │
│    ↓          ↓            ↓                                  │
│  PathRAG │ GraphRAG │ Semantic Search │ Direct Access        │
└───────────────────────────────────────────────────────────────┘
```

**Key Design Principles:**

- **Composable**: Chain, branch, and fork data flows as needed
- **Pluggable**: Hook custom processors anywhere in the pipeline
- **Multi-Output**: Send data to multiple destinations simultaneously
- **Independently Deployable**: Each component can run standalone or distributed
- **Event-Driven**: NATS-based messaging enables decoupled architecture
- **Resource-Aware**: Configure limits per component for edge deployment

**Example Flows:**

- **IoT Edge**: UDP → Parser → Entity Extractor → Local NATS KV → Query via REST
- **Data Integration**: Multiple inputs → Processor chain → Branch to both storage and webhook
- **Distributed Processing**: Input on edge → NATS federation → Cloud storage → Query anywhere
- **Custom Flow**: Your input → Your processors → SemStreams storage → PathRAG/GraphRAG queries

### Progressive Enhancement in Practice

**Unlike typical RAG/search stacks that require heavy cloud dependencies, SemStreams works offline:**

| Capability | Baseline (Always Works) | Enhancement (Optional) |
|-----------|------------------------|----------------------|
| **Embeddings** | BM25 lexical (pure Go) | HTTP neural embeddings |
| **Summaries** | Statistical TF-IDF | Async LLM enrichment |
| **Storage** | Local NATS KV | Federated multi-instance |
| **Network** | Offline operation | Distributed sync |
| **Query** | PathRAG + BM25 GraphRAG | Neural embeddings GraphRAG |

This means you can:

- ✅ Develop locally without internet
- ✅ Deploy to edge devices with minimal resources
- ✅ Scale up to cloud with neural models when available
- ✅ Degrade gracefully when services are unavailable

See [GraphRAG Guide](docs/guides/graphrag.md) and [PathRAG Guide](docs/guides/pathrag.md) for query patterns.

### Core Service Architecture

```text
┌─────────────────┐
│  semstreams-ui  │  ← Visual flow builder + runtime dashboard (Svelte 5, optional)
└────────┬────────┘
         │
┌────────▼────────┐
│   semstreams    │  ← Core engine (Go) - runs headless with JSON config
└────────┬────────┘
         │
    ┌────┴────┬──────────┬──────────┬──────────┐
    ▼         ▼          ▼          ▼          ▼
┌────────┐ ┌─────────┐ ┌─────────┐ ┌────────┐
│ NATS   │ │semembed │ │semsumm. │ │step-ca │
│JetStrm │ │(embeds) │ │ (LLM)   │ │ (PKI)  │
└────────┘ └─────────┘ └─────────┘ └────────┘

Optional observability stack:
┌───────────┐ ┌─────────┐
│Prometheus │ │ Grafana │
│ (metrics) │ │  (viz)  │
└───────────┘ └─────────┘
```

**Note**: semstreams can run headless with JSON configuration. The UI, semembed, semsummarize, step-ca, Prometheus, and Grafana are optional services that can be enabled via Docker Compose profiles.

**Optional Services**:

- **semembed**: Neural embeddings (defaults to BM25 without it)
- **semsummarize**: LLM summaries (defaults to statistical without it)
- **step-ca**: Automated PKI with ACME for mTLS ([setup guide](docs/deployment/acme-setup.md))
- **Prometheus + Grafana**: Metrics and visualization

### Reference Implementations

Applications built using SemStreams:

- **semmem**: Memory infrastructure for agentic coding with github spec-kit
- **semops**: Semantic common operational picture

## Getting Help

- **Issues**: Report bugs or request features in the respective repository
- **Discussions**: Join the discussion in [semstreams Discussions](https://github.com/c360/semstreams/discussions)
- **Documentation Issues**: Report doc issues in [semdocs](https://github.com/c360/semdocs/issues)

## Contributing

We welcome contributions! See the [Contributing Guide](docs/development/contributing.md) for details on:

- Development workflow with specialized agents (go-developer, svelte-developer, etc.)
- TDD best practices
- Code review process
- Documentation standards

## License

[License information to be added]

---

**Note**: This is the ecosystem documentation. For implementation details, see individual service repositories.
