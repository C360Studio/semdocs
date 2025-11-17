# SemStreams Ecosystem Documentation

**Real-time semantic stream processing at the edge**

Welcome to the SemStreams ecosystem documentation. This repository serves as the single source of truth for understanding the SemStreams platform architecture, concepts, and integration patterns.

## What is SemStreams?

SemStreams is a distributed semantic stream processing platform designed for edge deployments. It combines real-time graph processing, semantic search, and knowledge extraction to enable intelligent data processing at scale.

**Core Capabilities:**

### Real-Time Stream Processing

- **Flexible Ingestion**: UDP, TCP, HTTP, WebSocket sources
- **Universal Parsing**: JSON, CSV, raw bytes, custom formats
- **Graph Construction**: Real-time entity extraction and relationship building
- **Flow-Based Architecture**: Modular, composable processing components

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

**Query Strategies** - Choosing the right approach:

- [Choosing Your Query Strategy](docs/guides/choosing-query-strategy.md) - **START HERE** - Decision tree for PathRAG vs GraphRAG
- [PathRAG Guide](docs/guides/pathrag.md) - Graph traversal for dependencies and relationships
- [GraphRAG Guide](docs/guides/graphrag.md) - Hierarchical community search (local/global) for semantic similarity
- [Hybrid Query Patterns](docs/guides/hybrid-queries.md) - Combining PathRAG + GraphRAG for maximum insight

**Other Guides**:

- [Federation Guide](docs/guides/federation.md) - Distributed graph processing
- [Embedding Strategies](docs/guides/embedding-strategies.md) - Semantic search configuration
- [Schema Tags](docs/guides/schema-tags.md) - Schema annotation system

### 🏗️ [Architecture](docs/architecture/)

- [ADR-001: Tiered Security](docs/architecture/ADR-001-tiered-security-step-ca.md) - Step-CA integration
- [GraphQL Gateway Analysis](docs/architecture/GRAPHQL_GATEWAY_ANALYSIS.md) - Gateway architecture
- [GraphQL Gateway Config](docs/architecture/GRAPHQL_GATEWAY_CONFIG_EXAMPLE.md) - Configuration examples

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

- [Vocabulary Specification](docs/reference/vocabulary.md) - RDF vocabularies and ontologies
- [Schema Versioning](docs/reference/schema-versioning.md) - Schema evolution strategy
- [Port Allocation](docs/reference/port-allocation.md) - Service port registry
- [Configuration Schema](docs/reference/configuration-reference.md) - Complete config reference

### 🛠️ [Development](docs/development/)

- [Agent Workflow](docs/development/agents.md) - TDD workflow with specialized agents
- [Contributing Guide](docs/development/contributing.md) - How to contribute
- [Testing Philosophy](docs/development/testing.md) - Testing standards and patterns

## Architecture Highlights

### Stream Processing Pipeline

**Data flows through modular, composable components:**

```text
┌─────────────────────────────────────────────────────────────┐
│                      Input Sources                           │
│   UDP │ TCP │ HTTP │ WebSocket │ File │ NATS │ Custom      │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│                       Parsers                                │
│      JSON │ CSV │ Binary │ Custom │ Streaming               │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│                   Graph Builder                              │
│   Entity Extraction │ Relationship Discovery │ Validation   │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│                  Storage & Indexing                          │
│   NATS KV │ Semantic Index │ Graph Index │ Communities      │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│                  Query Engines                               │
│   PathRAG │ GraphRAG │ Semantic Search │ GraphQL │ REST     │
└─────────────────────────────────────────────────────────────┘
```

**Key Design Principles:**

- **Modular**: Each component is independently deployable
- **Composable**: Mix and match to build custom flows
- **Event-Driven**: NATS-based messaging throughout
- **Resource-Aware**: Configurable limits for edge deployment

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
    ┌────┴────┬──────────┬──────────┐
    ▼         ▼          ▼          ▼
┌────────┐ ┌─────────┐ ┌─────────┐
│ NATS   │ │semembed │ │semsumm. │
│JetStrm │ │(embeds) │ │ (LLM)   │
└────────┘ └─────────┘ └─────────┘

Optional observability stack:
┌───────────┐ ┌─────────┐
│Prometheus │ │ Grafana │
│ (metrics) │ │  (viz)  │
└───────────┘ └─────────┘
```

**Note**: semstreams can run headless with JSON configuration. The UI, semembed, semsummarize, Prometheus, and Grafana are optional services that can be enabled via Docker Compose profiles.

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
