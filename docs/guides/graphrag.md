# GraphRAG: Hierarchical Community Search

**Status**: Production-ready (since v0.8.0)
**Package**: `pkg/graphclustering`, `processor/graph/querymanager`
**GraphQL API**: `localSearch`, `globalSearch`, `community`, `entityCommunity`

## Overview

GraphRAG (Graph Retrieval-Augmented Generation) combines hierarchical community detection with semantic search to enable both fast, focused queries (local search) and comprehensive, multi-community queries (global search). Unlike traditional RAG that searches a flat vector space, GraphRAG understands the graph's cluster structure.

**Key Insight**: Entity communities reveal semantic structure. Search within relevant communities (local) for speed, or across community summaries (global) for comprehensiveness.

## SemStreams Implementation

SemStreams implements GraphRAG with **progressive enhancement** - full functionality works offline with zero external dependencies:

### Storage Architecture

```go
// Entity States - NATS KV bucket ENTITY_STATES
type EntityState struct {
    Node  NodeProperties  // Entity data
    Edges []Edge         // Relationships
}

// Community Index - NATS KV bucket COMMUNITY_INDEX
type Community struct {
    ID                 string   // comm-{level}-{label}
    Level              int      // Hierarchy level (0=finest)
    Members            []string // Entity IDs in community
    Summary            string   // Combined summary
    StatisticalSummary string   // TF-IDF keywords (always available)
    LLMSummary         string   // Async LLM enhancement (optional)
    Keywords           []string // Extracted terms
    RepEntities        []string // Most central entities
}
```

### Progressive Enhancement Pattern

| Component | Baseline (Always Works) | Enhancement (Optional) |
|-----------|------------------------|----------------------|
| **Embeddings** | BM25 (pure Go, lexical) | HTTP embedder (neural) |
| **Summaries** | Statistical (TF-IDF + LPA) | Async LLM enrichment |
| **Network** | Offline (NATS KV local) | Distributed (NATS federation) |
| **Dependencies** | Zero (pure Go) | semembed, semsummarize services |

**Design Philosophy**: Unlike typical GraphRAG (requires Pinecone + Neo4j + LLM API), SemStreams works **offline** with graceful enhancement when resources are available.

## Comparison to Microsoft GraphRAG

| Feature | Microsoft GraphRAG | SemStreams GraphRAG |
|---------|-------------------|---------------------|
| **Vector Storage** | Pinecone/Weaviate (required) | BM25 baseline + optional HTTP embedder |
| **Graph Storage** | Neo4j/TigerGraph (required) | NATS KV + in-memory |
| **Summaries** | LLM API (required) | Statistical baseline + async LLM |
| **Deployment** | Cloud-only (heavy deps) | Edge-first (offline capable) |
| **Community Detection** | Leiden/Louvain (Python) | Label Propagation (pure Go) |
| **Real-time** | Batch processing | Streaming event-driven |
| **Cost** | $$$$ (API calls + databases) | $ (self-hosted, optional services) |

**Key Difference**: Microsoft GraphRAG requires external APIs and databases. SemStreams provides full GraphRAG functionality with zero external dependencies, progressively enhancing when services are available.

## Local vs Global Search

### Local Search: Fast, Focused

Query within an entity's community for **fast, targeted results**.

**Use When:**
- You have a specific starting entity
- Need low-latency responses
- Want semantically related entities
- Context is bounded (e.g., "similar specs to auth.md")

**Example (GraphQL):**
```graphql
query {
  localSearch(
    entityID: "org.semmem.spec.auth"
    query: "security vulnerabilities"
    level: 1
  ) {
    ... on Spec { id title status }
    ... on Issue { number title state }
  }
}
```

**How It Works:**
1. Find entity's community at specified level
2. Get all community members
3. Search within members using BM25/embeddings
4. Rank by semantic similarity + community centrality

**Performance:** ~50-200ms (searches 10-100 entities)

### Global Search: Comprehensive, Multi-Community

Query across **community summaries** for comprehensive results.

**Use When:**
- No specific starting entity
- Need comprehensive coverage
- Exploring unfamiliar domains
- Want diverse perspectives (e.g., "authentication patterns across all projects")

**Example (GraphQL):**
```graphql
query {
  globalSearch(
    query: "authentication implementation patterns"
    level: 2
    maxCommunities: 5
  ) {
    ... on Spec { id title }
    ... on PullRequest { number title }
  }
}
```

**How It Works:**
1. Search across community summaries (not individual entities)
2. Rank communities by relevance
3. Retrieve top entities from each community
4. Re-rank combined results

**Performance:** ~200-500ms (searches summaries, then top communities)

## Community Hierarchy

Communities are organized in hierarchical levels:

```text
Level 2 (High-level)
  ├─ comm-2-frontend-architecture    (10 entities)
  ├─ comm-2-backend-services          (15 entities)
  └─ comm-2-data-infrastructure       (12 entities)
      ↓
Level 1 (Mid-level)
  ├─ comm-1-api-gateway               (5 entities)
  ├─ comm-1-auth-service              (4 entities)
  ├─ comm-1-database-layer            (6 entities)
      ↓
Level 0 (Fine-grained)
  ├─ comm-0-user-endpoints            (2 entities)
  ├─ comm-0-session-management        (3 entities)
```

**Level Selection Guidelines:**
- **Level 0**: Tight clusters (2-10 entities) - Very focused
- **Level 1**: Medium clusters (5-20 entities) - Balanced
- **Level 2+**: Broad clusters (10-50+ entities) - High-level overview

## Use Cases

### 1. Code Review Context (Local Search)

**Scenario**: Find related specs and issues for a PR review

```json
{
  "entity_id": "org.semmem.pr.1234",
  "query": "breaking changes api contracts",
  "level": 1
}
```

**Result**: Discovers within PR's community:
- Related specs that define affected APIs
- Previous issues about API stability
- Other PRs that modified same contracts

**Why Local**: PR has a defined community of related work.

### 2. Architecture Exploration (Global Search)

**Scenario**: Understand authentication patterns across entire codebase

```json
{
  "query": "jwt token validation middleware patterns",
  "level": 2,
  "max_communities": 5
}
```

**Result**: Finds entities across multiple communities:
- Frontend auth components (comm-2-frontend)
- Backend auth services (comm-2-backend)
- Infrastructure security configs (comm-2-infra)

**Why Global**: No specific starting point, want comprehensive view.

### 3. Impact Analysis (Community Inspection)

**Scenario**: See what's in a spec's community

```graphql
query {
  entityCommunity(
    entityID: "org.semmem.spec.auth"
    level: 1
  ) {
    id
    level
    members
    keywords
    summary
  }
}
```

**Result**: Community metadata showing:
- All entities in auth spec's cluster
- Keywords: ["authentication", "jwt", "sessions", "oauth"]
- Summary: "Authentication and authorization infrastructure..."

**Why Useful**: Understand spec's semantic neighborhood.

## Configuration

### Basic Setup (Zero Dependencies)

```json
{
  "components": [
    {
      "type": "graphprocessor",
      "config": {
        "enable_clustering": true,
        "clustering": {
          "algorithm": "label_propagation",
          "max_levels": 3,
          "embedder": "bm25"
        }
      }
    }
  ]
}
```

**Result**: Full GraphRAG with BM25 + statistical summaries.

### Enhanced Setup (with HTTP Embedder)

```json
{
  "components": [
    {
      "type": "graphprocessor",
      "config": {
        "enable_clustering": true,
        "clustering": {
          "algorithm": "label_propagation",
          "max_levels": 3,
          "embedder": "http",
          "embedder_config": {
            "url": "http://localhost:8080/embed",
            "model": "all-MiniLM-L6-v2",
            "fallback": "bm25"
          }
        }
      }
    }
  ]
}
```

**Result**: Neural embeddings when semembed available, BM25 fallback.

### Full Enhancement (with LLM Summaries)

```json
{
  "components": [
    {
      "type": "graphprocessor",
      "config": {
        "enable_clustering": true,
        "clustering": {
          "algorithm": "label_propagation",
          "max_levels": 3,
          "embedder": "http",
          "enable_llm_enhancement": true,
          "llm_config": {
            "url": "http://localhost:8081/summarize",
            "model": "llama-3.2-1b",
            "async": true
          }
        }
      }
    }
  ]
}
```

**Result**: Best quality - neural embeddings + LLM-enriched summaries.

## GraphQL Integration

Generate GraphQL resolvers using `semstreams-gqlgen`:

**Schema:**
```graphql
type Community {
  id: ID!
  level: Int!
  members: [String!]!
  summary: String
  keywords: [String!]
}

type Query {
  localSearch(entityID: ID!, query: String!, level: Int!): [Entity!]!
  globalSearch(query: String!, level: Int!, maxCommunities: Int!): [Entity!]!
  community(id: ID!): Community
  entityCommunity(entityID: ID!, level: Int!): Community
}
```

**Config:**
```json
{
  "queries": {
    "localSearch": {
      "resolver": "LocalSearch",
      "subject": "graph.semmem.localsearch"
    },
    "globalSearch": {
      "resolver": "GlobalSearch",
      "subject": "graph.semmem.globalsearch"
    }
  }
}
```

See [semstreams-gqlgen README](../../semstreams/cmd/semstreams-gqlgen/README.md) for complete code generation guide.

## Community Detection Algorithm

SemStreams uses **Label Propagation Algorithm (LPA)** for community detection:

### Why LPA?

| Algorithm | Time Complexity | Dependencies | Deterministic | Quality |
|-----------|----------------|--------------|---------------|---------|
| **LPA** | O(n) - O(n log n) | None | With seed | Good |
| Leiden | O(n log n) - O(n²) | igraph (Python) | Yes | Excellent |
| Louvain | O(n log n) | NetworkX | No | Very Good |
| Girvan-Newman | O(n³) | NetworkX | Yes | Excellent |

**Rationale**: LPA provides **good-enough quality** with **pure Go implementation** and **linear time complexity**. Perfect for edge deployment where external dependencies are problematic.

### How LPA Works

```go
// Simplified LPA pseudocode
func LabelPropagation(graph Graph) Communities {
    // 1. Initialize: Each node starts in own community
    labels := map[string]string{}
    for node := range graph.Nodes {
        labels[node] = node
    }

    // 2. Iterate until convergence
    for iteration := 0; iteration < maxIterations; iteration++ {
        changed := false
        for node := range graph.Nodes {
            // 3. Adopt most common neighbor label
            neighborLabels := countNeighborLabels(node, labels)
            mostCommon := findMostCommon(neighborLabels)

            if labels[node] != mostCommon {
                labels[node] = mostCommon
                changed = true
            }
        }

        if !changed {
            break // Converged
        }
    }

    return groupByCommunity(labels)
}
```

**Convergence**: Typically 5-15 iterations for most graphs.

## Performance Characteristics

### Storage

```text
Entities:    1000 → ~500KB in NATS KV
Communities: ~50  → ~25KB in NATS KV
Embeddings:  1000 × 384 dims → ~1.5MB in memory (BM25 cache)
```

**Total for 1000-entity graph**: ~2MB (edge-friendly!)

### Query Latency

| Operation | Latency (p50) | Latency (p99) |
|-----------|---------------|---------------|
| Local Search (level 0) | 50ms | 150ms |
| Local Search (level 2) | 120ms | 300ms |
| Global Search (3 communities) | 200ms | 500ms |
| Global Search (10 communities) | 400ms | 1000ms |
| Get Community | 10ms | 30ms |

**Measured on**: Apple M1, 1000-entity graph, BM25 embeddings

### Clustering Performance

```text
Community Detection (LPA):
  100 entities  → ~20ms
  1000 entities → ~200ms
  10000 entities → ~3s

Statistical Summary Generation:
  Per community (50 entities) → ~50ms

LLM Summary Enhancement (async):
  Per community → ~2-5s (doesn't block queries)
```

## Best Practices

### 1. Choose Appropriate Level

**Problem**: How to pick the right hierarchy level?

**Solution**:
- Start with **Level 1** (balanced)
- Use **Level 0** if you need very tight clusters
- Use **Level 2+** for high-level architecture queries

### 2. Balance Local vs Global

**Problem**: When to use local vs global search?

**Heuristic**:
```
if (have_specific_entity && want_fast_response) {
    use_local_search(entity, level=1)
} else if (exploring_domain && want_comprehensive) {
    use_global_search(level=2, max=5)
}
```

### 3. Progressive Enhancement Strategy

**Problem**: How to configure for different environments?

**Solution**:
```text
Development:    BM25 only (fast, no services)
                ↓
Staging:        BM25 + HTTP embedder (better quality)
                ↓
Production:     BM25 + HTTP embedder + async LLM (best quality)
```

### 4. Monitor Community Quality

**Metrics to track**:
- Community sizes (should be balanced, not dominated by one huge cluster)
- Keyword diversity (communities should have distinct keywords)
- Re-clustering frequency (stable communities = good quality)

## Troubleshooting

### Issue: Global Search Returns No Results

**Symptoms**: `globalSearch` returns empty array

**Causes**:
1. No communities generated yet (clustering not run)
2. Query doesn't match any community summaries
3. `maxCommunities` set too low

**Solutions**:
```graphql
# Check if communities exist
query {
  entityCommunity(entityID: "any-entity-id", level: 1) {
    id  # Should return a community
  }
}

# Try broader query
globalSearch(query: "*", level: 1, maxCommunities: 10)

# Check clustering config
```

### Issue: Local Search Too Slow

**Symptoms**: Local search takes >500ms

**Causes**:
1. Community too large (>100 entities)
2. Using high hierarchy level with BM25
3. No embedding cache

**Solutions**:
- Use **Level 0** for smaller communities
- Enable HTTP embedder with caching
- Consider re-clustering with stricter parameters

### Issue: Summaries Are Poor Quality

**Symptoms**: Community summaries don't make sense

**Causes**:
1. Only statistical summaries (no LLM)
2. Communities too heterogeneous (bad clustering)
3. Not enough entity content

**Solutions**:
- Enable async LLM enhancement
- Tune LPA parameters (iterations, convergence threshold)
- Ensure entities have rich property content

## Real-World Example: SemMem

SemMem uses GraphRAG for semantic issue tracking:

**Local Search Use Case**: "Find related issues when reviewing a spec"
```graphql
query {
  spec(id: "org.semmem.spec.auth") {
    title

    # Get spec's community context
    community: entityCommunity(level: 1) {
      summary
      keywords
    }

    # Find related entities in community
    related: localSearch(
      query: "implementation security"
      level: 1
    ) {
      ... on Issue { number title }
      ... on PullRequest { number title }
    }
  }
}
```

**Global Search Use Case**: "Find authentication patterns across all projects"
```graphql
query {
  authPatterns: globalSearch(
    query: "jwt session oauth authentication"
    level: 2
    maxCommunities: 5
  ) {
    ... on Spec { id title }
    ... on Doc { id title }
  }
}
```

## References

- **Implementation**: `pkg/graphclustering` (LPA algorithm)
- **Storage**: `pkg/graphclustering/storage.go` (NATS KV integration)
- **Query API**: `processor/graph/querymanager/graphrag_search.go`
- **GraphQL**: `cmd/semstreams-gqlgen` (code generator)
- **Example**: `semmem/` (production GraphRAG deployment)

## Next Steps

1. **Try It**: Use SemMem as reference implementation
2. **Tune It**: Adjust hierarchy levels for your graph
3. **Enhance It**: Enable HTTP embedder when ready
4. **Scale It**: Use NATS federation for distributed deployment

---

**Related Guides**:
- [PathRAG Guide](pathrag.md) - Graph traversal patterns (complements GraphRAG)
- [Embedding Strategies](embedding-strategies.md) - BM25 vs HTTP embedder configuration
- [Federation Guide](federation.md) - Distributed GraphRAG across instances
