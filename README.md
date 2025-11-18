# SemStreams Ecosystem Documentation

## Work In Progress

**All repos in ecosystem are under heavy and active dev.**

## Real-time semantic stream processing at the edge

Welcome to the SemStreams ecosystem documentation. This repository serves as the single source of truth for understanding the SemStreams platform architecture, concepts, and integration patterns.

## What is SemStreams?

SemStreams is a distributed semantic stream processing platform designed for edge deployments. It combines real-time graph processing, semantic search, and knowledge extraction to enable intelligent data processing at scale.

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
