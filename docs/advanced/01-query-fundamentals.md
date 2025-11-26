# Query Fundamentals

**Understanding PathRAG and GraphRAG: the two approaches to finding information in SemStreams**

---

## Why Two Query Approaches?

Knowledge graphs contain two types of valuable information:

1. **Structural relationships** - explicit connections between entities (A depends on B, drone-001 is near drone-002)
2. **Semantic similarity** - conceptual relatedness based on content (two documents about authentication, even if not linked)

SemStreams provides two complementary query approaches to leverage both:

| Approach | What It Finds | How It Works |
|----------|--------------|--------------|
| **PathRAG** | Structurally connected entities | Walks graph edges from a starting point |
| **GraphRAG** | Semantically similar entities | Searches within pre-computed community clusters |

Neither approach is "better" - they answer different questions. Understanding when to use each (or both together) is key to effective querying.

---

## PathRAG: Following the Structure

### What Is PathRAG?

PathRAG (Path-based Retrieval-Augmented Generation) traverses the graph by following explicit relationships. Think of it as walking through a building by following hallways and doors.

```text
You start at Room A
You follow doors (edges) to connected rooms
You stop after N doors or when time runs out
```

### PathRAG Traversal

```text
Starting Entity: drone-001
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
    belongs_to    near      reports_to
        │           │           │
        ▼           ▼           ▼
   fleet-rescue  drone-002  base-station
        │                       │
        ▼                       ▼
   stationed_at            monitors
        │                       │
        ▼                       ▼
    base-west              sector-7
```

PathRAG follows the edges (belongs_to, near, reports_to, etc.) to discover connected entities. Each hop increases the "distance" from the starting point.

### PathRAG Characteristics

- **Requires a starting entity** - you must know where to begin
- **Follows explicit relationships** - only traverses edges that exist in the graph
- **Bounded by resources** - max depth, max nodes, max time prevent runaway queries
- **Relevance decay** - entities farther away score lower (configurable)
- **No pre-computation** - queries execute at runtime against live graph

### When to Use PathRAG

| Question Type | Example |
|--------------|---------|
| Dependency tracing | "What services depend on this config?" |
| Impact analysis | "If this fails, what's affected?" |
| Spatial proximity | "What drones are near this one?" |
| Relationship chains | "How is entity A connected to entity B?" |

### Example Query

```json
{
  "start_entity": "c360.platform1.robotics.gcs1.drone.1",
  "max_depth": 3,
  "edge_filter": ["belongs_to", "near", "reports_to"],
  "max_nodes": 100,
  "max_time": "200ms",
  "decay_factor": 0.85
}
```

**Result**: All entities reachable within 3 hops via the specified edge types, with relevance scores that decay with distance.

---

## GraphRAG: Finding Semantic Clusters

### What Is GraphRAG?

GraphRAG (Graph-based Retrieval-Augmented Generation) organizes entities into semantic communities based on content similarity, then enables search within and across those communities. Think of it as a library with sections.

```text
Books are organized into sections (communities)
You can browse within a section (local search)
Or search across all section summaries (global search)
```

### GraphRAG Processing

**Step 1: Community Detection (Pre-computation)**

```text
All Entities
     │
     ▼ (Label Propagation Algorithm)
┌────────────────────────────────────────┐
│                                        │
│  ┌──────────┐  ┌──────────┐  ┌──────┐ │
│  │Community │  │Community │  │Comm. │ │
│  │   Auth   │  │ Database │  │ API  │ │
│  │          │  │          │  │      │ │
│  │ jwt.go   │  │ pool.go  │  │route │ │
│  │ oauth.go │  │ query.go │  │ ctrl │ │
│  │ session  │  │ migrate  │  │ midw │ │
│  └──────────┘  └──────────┘  └──────┘ │
│                                        │
│  Level 1 (fine-grained)               │
└────────────────────────────────────────┘
     │
     ▼ (Hierarchical clustering)
┌────────────────────────────────────────┐
│  ┌─────────────────┐  ┌─────────────┐  │
│  │  Backend Infra  │  │  Frontend   │  │
│  │  (Auth + DB)    │  │  (API + UI) │  │
│  └─────────────────┘  └─────────────┘  │
│                                        │
│  Level 2 (coarse)                     │
└────────────────────────────────────────┘
```

**Step 2: Summarization**

Each community gets a summary (statistical TF-IDF keywords, or LLM-generated if SemSummarize is enabled).

**Step 3: Query**

- **Local Search**: Start from an entity, search within its community
- **Global Search**: Search across all community summaries

### GraphRAG Characteristics

- **Pre-computed communities** - clustering happens at index time, not query time
- **Semantic grouping** - entities clustered by content similarity, not explicit edges
- **Hierarchical levels** - fine-grained (Level 1) to coarse (Level 3) communities
- **Two search modes** - local (within community) and global (across communities)
- **Works without relationships** - can find similar entities even if they're not linked

### When to Use GraphRAG

| Question Type | Example |
|--------------|---------|
| Similarity search | "Find code similar to this auth module" |
| Topic exploration | "What's in the system about authentication?" |
| Pattern discovery | "Show me all error handling patterns" |
| Related content | "What docs relate to this feature?" |

### Example Queries

**Local Search** (starting from an entity):

```graphql
query {
  localSearch(
    entityID: "c360.platform1.dev.repo1.file.auth-go"
    query: "token validation jwt"
    level: 1
  ) {
    id
    score
  }
}
```

**Global Search** (no starting entity):

```graphql
query {
  globalSearch(
    query: "authentication security oauth jwt"
    level: 2
    maxCommunities: 5
  ) {
    id
    score
    community
  }
}
```

---

## Comparing the Two Approaches

### Side-by-Side

| Aspect | PathRAG | GraphRAG |
|--------|---------|----------|
| **Finds** | Connected entities | Similar entities |
| **Requires** | Starting entity | Starting entity (local) or nothing (global) |
| **Traverses** | Explicit edges | Semantic communities |
| **Pre-computation** | None | Community detection |
| **Best for** | Relationship tracing | Similarity search |
| **Latency** | 10-200ms | 50-500ms |

### What Each Misses

**PathRAG alone misses:**

- Semantically similar entities that aren't explicitly linked
- Content-based patterns across unconnected entities
- "Related" content without explicit relationships

**GraphRAG alone misses:**

- Explicit dependency chains
- Relationship semantics (the meaning of specific edge types)
- Spatial/topological structure

### The Power of Combining Both

**Hybrid queries** combine PathRAG and GraphRAG for comprehensive results:

```text
Query: "Show me everything related to this authentication bug"

PathRAG finds:                    GraphRAG finds:
├── Services depending on auth    ├── Similar past auth bugs
├── Config files linked to auth   ├── Documentation about auth
├── Tests referencing the bug     ├── PRs with similar keywords
└── Downstream failures           └── Related specs in same cluster

Combined: Complete context from both structural AND semantic perspectives
```

---

## Quick Decision Guide

```text
What are you trying to find?
│
├─ "What DEPENDS ON this?" ──────────────► PathRAG
│  "What's CONNECTED to this?"
│  "Trace this failure IMPACT"
│
├─ "What's SIMILAR to this?" ─────────────► GraphRAG Local
│  "What else is in this CLUSTER?"
│  "Related items in this COMMUNITY?"
│
├─ "What's in the system ABOUT X?" ───────► GraphRAG Global
│  "Show me all AUTH code"
│  "Find PATTERNS across projects"
│
└─ "EVERYTHING related to this" ──────────► Hybrid (both)
   "Full CONTEXT for this entity"
```

---

## Resource Requirements

### PathRAG Only

- **Memory**: ~200 bytes per entity
- **Pre-computation**: None
- **External services**: None
- **Works on**: Raspberry Pi, edge devices, anywhere

### GraphRAG (with BM25 embeddings)

- **Memory**: ~2KB per entity
- **Pre-computation**: Community detection at startup
- **External services**: None (native Go)
- **Works on**: Laptop, server, cloud

### GraphRAG (with neural embeddings)

- **Memory**: ~2KB per entity
- **Pre-computation**: Community detection + embedding generation
- **External services**: SemEmbed (optional GPU)
- **Works on**: Server, cloud (2GB+ RAM for embedding service)

---

## Learning Path

Now that you understand the fundamentals, proceed through the advanced documentation in order:

1. **[Architecture Deep Dive](02-architecture-deep-dive.md)** - How SemStreams implements these concepts
2. **[Algorithm Reference](03-algorithm-reference.md)** - Native algorithms (BM25, LPA, PageRank)
3. **[Configuration Guide](04-configuration-guide.md)** - When to enable PathRAG, GraphRAG, or both
4. **[Query Strategies](05-query-strategies.md)** - Practical patterns for each approach
5. **[Hybrid Queries](06-hybrid-queries.md)** - Combining approaches for maximum insight
6. **[Performance Tuning](07-performance-tuning.md)** - Optimization strategies
7. **[Production Patterns](08-production-patterns.md)** - Deployment best practices

---

**Key Takeaway**: PathRAG and GraphRAG are complementary tools. PathRAG follows the graph's structure; GraphRAG finds semantic clusters. Use the right tool for your question, or combine both for comprehensive results.
