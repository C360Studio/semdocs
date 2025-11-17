# SemStreams Ecosystem Documentation

**Real-time semantic stream processing at the edge**

Welcome to the SemStreams ecosystem documentation. This repository serves as the single source of truth for understanding the SemStreams platform architecture, concepts, and integration patterns.

## What is SemStreams?

SemStreams is a distributed semantic stream processing platform designed for edge deployments. It combines real-time graph processing, semantic search, and knowledge extraction to enable intelligent data processing at scale.

**Core Capabilities:**

- **Flow-Based Stream Processing**: Modular components for ingesting from any source (UDP, TCP, HTTP, WebSocket), parsing any format (JSON, CSV, raw bytes), and constructing semantic knowledge graphs
- **GraphRAG**: Hierarchical community detection with local/global search (see [GraphRAG Guide](docs/guides/graphrag.md))
  - **Zero-Dependency Core**: BM25 embeddings + statistical summaries (pure Go, works offline)
  - **Progressive Enhancement**: Optional HTTP embedder + async LLM enrichment when available
  - **Edge-Optimized Storage**: NATS KV replaces traditional vector/graph databases
- **PathRAG**: Knowledge extraction using graph traversal patterns
- **Semantic Search**: Vector embeddings + BM25 hybrid search
- **Graph Clustering**: Community detection with label propagation
- **Federation**: Distributed graph processing across multiple instances

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

- [GraphRAG Guide](docs/guides/graphrag.md) - Hierarchical community search (local/global)
- [Federation Guide](docs/guides/federation.md) - Distributed graph processing
- [PathRAG Guide](docs/guides/pathrag.md) - Knowledge extraction patterns
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

### GraphRAG: Edge-Optimized, Progressive Enhancement

**Unlike typical GraphRAG stacks** (requires Pinecone + Neo4j + S3 + LLM API), **SemStreams works offline**:

| Component | Typical GraphRAG | SemStreams |
|-----------|-----------------|------------|
| **Vector Storage** | Pinecone/Weaviate (required) | BM25 (pure Go) + optional HTTP embedder |
| **Graph Storage** | Neo4j/TigerGraph (required) | NATS KV + in-memory |
| **Summaries** | LLM API (required) | Statistical (TF-IDF) + optional async LLM |
| **Deployment** | Cloud-only (heavy deps) | Edge-first (zero external services) |

**Progressive Enhancement Pattern:**
- **BM25 → HTTP Embedder**: Lexical search always works, neural embeddings enhance accuracy
- **Statistical Summary → LLM Summary**: Label propagation always generates summaries, async LLM enrichment improves quality
- **Works Offline**: Full GraphRAG functionality without internet connection

See [GraphRAG Guide](docs/guides/graphrag.md) for local/global search patterns and configuration.

### Stream Processing Flow

```text
Input → Parser → Graph Builder → Indexer → Query Engine
  ↓                    ↓             ↓           ↓
NATS              Entity Store   Embeddings   GraphQL/REST
                    (KV)          (BM25/HTTP)
```

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
