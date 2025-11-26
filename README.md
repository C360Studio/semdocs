# SemDocs

## Work In Progress

All repos are under **heavy and active** development

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

**Examples available:**

- **[quickstart/](examples/quickstart/)** - NATS + SemStreams (minimal setup)
- **[with-ui/](examples/with-ui/)** - Add visual flow builder
- **[with-embeddings/](examples/with-embeddings/)** - Add semantic search
- **[production/](examples/production/)** - Hardened edge deployment

See **[Examples Overview](examples/README.md)** for full comparison and setup guides.

**Start here:**

- **New to SemStreams?** → [What is SemStreams](docs/basics/01-what-is-semstreams.md)
- **Want to run it now?** → [Examples](examples/README.md)
- **Building flows?** → [Your First Flow](docs/basics/04-first-flow.md)
- **Production deployment?** → [Production Guide](docs/deployment/production.md)

**Repositories:**
[semstreams](https://github.com/C360Studio/semstreams) (core)
[semembed](https://github.com/C360Studio/semembed) (embeddings)
[semsummarize](https://github.com/C360Studio/semsummarize) (LLM)
[semmem](https://github.com/C360Studio/semmem) (agentic memory)

---

**SemStreams** - Event-based semantic graph runtime.
