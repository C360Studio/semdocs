# Practical Query Patterns

**Applying PathRAG, GraphRAG, and hybrid strategies to real-world scenarios**

---

> **Prerequisite**: This guide assumes you understand [Query Fundamentals](01-query-fundamentals.md) - what PathRAG and GraphRAG are. This document focuses on **practical application patterns**.

---

## Overview

This guide provides concrete examples of when and how to use each query strategy. Use the decision tree for quick reference, then see detailed patterns below.

## The Quick Decision Tree

```text
What are you trying to find?
│
├─ "What DEPENDS ON this?" ──────────────► PathRAG
│  "What's CONNECTED to this?"
│  "Trace this failure IMPACT"
│
├─ "What's SIMILAR to this?" ─────────────► GraphRAG (Local Search)
│  "What else is in this CLUSTER?"
│  "Related items in this COMMUNITY?"
│
├─ "What's in the system ABOUT X?" ───────► GraphRAG (Global Search)
│  "Show me all AUTHENTICATION code"
│  "Find PATTERNS across projects"
│
└─ "Full CONTEXT for this entity" ────────► Hybrid (PathRAG + GraphRAG)
   "Everything related AND similar"
```

---

## What's the Difference?

### PathRAG: Following the Graph's Structure

**What it does**: Walks along relationships (edges) from a starting point

**Think of it like**: Following a trail through the woods

- You start at a specific tree
- You follow marked paths (relationships)
- You stop after a certain number of steps or time limit

**Example Query**: "Starting from `config/database-url`, what services depend on it?"

```json
{
  "start_entity": "config/database-url",
  "max_depth": 3,
  "edge_filter": ["depends_on", "uses"],
  "max_time": "200ms"
}
```

**Returns**: Entities connected through explicit relationships:

- `api-service` (depth 1) - directly depends on config
- `auth-service` (depth 1) - directly depends on config
- `mobile-app` (depth 2) - depends on api-service
- `analytics-dashboard` (depth 3) - depends on auth-service

### GraphRAG: Finding Semantic Clusters

**What it does**: Pre-groups entities into communities, then searches within/across communities

**Think of it like**: Library sections

- Books are organized into sections (communities)
- You can browse within a section (local search)
- Or search across all section summaries (global search)

**Example Query**: "Find authentication-related code similar to this auth module"

```graphql
query {
  localSearch(
    entityID: "modules/auth.go"
    query: "token validation jwt"
    level: 1
  ) {
    id
    title
  }
}
```

**Returns**: Entities in the same semantic cluster:

- `middleware/jwt.go` (similar content, same community)
- `tests/auth_test.go` (related topic, same community)
- `docs/authentication.md` (similar keywords, same community)

**Key Difference**: GraphRAG doesn't require explicit relationships - it finds entities grouped by semantic similarity.

---

## When to Use Each Strategy

### Use PathRAG When

✅ **You have a specific starting entity**

- "What does this service call?"
- "What entities are near this drone?"

✅ **You need to trace relationships**

- "Show me the dependency chain"
- "What's the impact radius of this failure?"

✅ **Relationships are meaningful**

- "Follow the `triggered_by` edges"
- "Only traverse `depends_on` relationships"

✅ **You need real-time results with strict resource limits**

- "Find related entities in under 200ms"
- "Bounded by 100 nodes maximum"

**PathRAG Examples:**

```bash
# Dependency analysis
POST /entity/api-service/path
{"max_depth": 5, "edge_filter": ["depends_on"]}

# Spatial mesh network
POST /entity/drone-001/path
{"edge_filter": ["near", "communicates"], "max_depth": 3}

# Failure impact chain
POST /entity/alert-network-outage/path
{"edge_filter": ["triggers", "affects"], "max_depth": 4}
```

### Use GraphRAG Local Search When

✅ **You want semantically similar entities**

- "Show me code files similar to this one"
- "Find related specs in this project"

✅ **You have a starting entity**

- "What's in the same cluster as this PR?"

✅ **You want a focused, fast query**

- "Search within this entity's community"

✅ **Semantic similarity matters more than explicit relationships**

- "These entities share topics/keywords"
- "Similar content, even without direct links"

**GraphRAG Local Search Examples:**

```graphql
# Find related specs for a PR review
query {
  localSearch(
    entityID: "pr/1234"
    query: "breaking changes api contracts"
    level: 1
  )
}

# Similar documentation
query {
  localSearch(
    entityID: "docs/authentication.md"
    query: "security best practices"
    level: 1
  )
}
```

### Use GraphRAG Global Search When

✅ **You DON'T have a specific starting entity**

- "Show me all authentication code in the system"

✅ **You need comprehensive, system-wide results**

- "Find patterns across all projects"
- "What's in the system about microservices?"

✅ **You want diverse perspectives**

- "Search across multiple communities"
- "Show me different approaches to caching"

**GraphRAG Global Search Examples:**

```graphql
# System-wide pattern search
query {
  globalSearch(
    query: "authentication implementation jwt oauth"
    level: 2
    maxCommunities: 5
  )
}

# Cross-project exploration
query {
  globalSearch(
    query: "error handling retry patterns"
    level: 2
    maxCommunities: 10
  )
}
```

---

## Hybrid Queries: Best of Both Worlds

**The Most Powerful Approach**: Combine PathRAG and GraphRAG for comprehensive context.

### Pattern 1: Semantic Search → Path Expansion

**Use Case**: Find relevant entities, then explore their connections

**Workflow**:

```text
1. GraphRAG Local Search → Find semantically similar entities
2. PathRAG Expansion      → Explore relationships from top results
3. Merge Results          → Combine scores
```

**Example**:

```javascript
// Step 1: Find semantically similar entities
const similar = await localSearch({
  entityID: "drone-001",
  query: "battery performance degradation",
  level: 1
});

// Step 2: Expand from top result via relationships
const connected = await pathQuery({
  start_entity: similar[0].id,
  max_depth: 3,
  edge_filter: ["related_to", "triggered_by", "causes"]
});

// Step 3: Merge and rank
const results = mergeResults(similar, connected, {
  semanticWeight: 0.6,
  pathWeight: 0.4
});
```

**Returns**: Both semantically similar entities AND their connected neighbors!

### Pattern 2: Path Exploration → Community Context

**Use Case**: Follow relationships, then understand the semantic cluster

**Workflow**:

```text
1. PathRAG               → Find connected entities
2. GraphRAG Community    → See what community they're in
3. GraphRAG Local Search → Find similar entities in that community
```

**Example**:

```javascript
// Step 1: Follow dependency chain
const deps = await pathQuery({
  start_entity: "config/api-key",
  max_depth: 3,
  edge_filter: ["depends_on"]
});

// Step 2: Get community for top dependent
const community = await getCommunity({
  entityID: deps.entities[0].id,
  level: 1
});

// Step 3: Search within that community
const clusterContext = await localSearch({
  entityID: deps.entities[0].id,
  query: "configuration security",
  level: 1
});
```

**Returns**: Dependency chain PLUS semantic context from the cluster!

---

## How They Work Together Under the Hood

**Here's what many users don't realize**: GraphRAG actually uses the same graph traversal infrastructure as PathRAG!

### GraphRAG's Dependency on Graph Access

```text
GraphRAG Community Detection Process:

1. GraphRAG calls QueryManager to access the graph
2. QueryManager provides graph traversal (similar to PathRAG)
3. GraphRAG uses traversal to build community structure
4. Communities stored in NATS KV (COMMUNITY_INDEX)
5. Local/Global search queries use communities

In other words:
GraphRAG = Graph Clustering (via traversal) + Semantic Search
PathRAG  = Direct graph traversal for queries
```

**They share the same underlying graph storage**:

- Both read from: `ENTITY_STATES` bucket (NATS KV)
- Both use: Same entity and edge data
- PathRAG stores: Nothing (pure query-time)
- GraphRAG stores: Pre-computed communities in `COMMUNITY_INDEX`

---

## Practical Decision Guide

### Scenario: "I need to review this pull request"

**Question**: Do you want related code (relationships) or similar code (semantics)?

- **Related code**: PathRAG

  ```json
  POST /entity/pr-1234/path
  {
    "edge_filter": ["modifies", "affects", "depends_on"],
    "max_depth": 2
  }
  ```

- **Similar code**: GraphRAG Local Search

  ```graphql
  query {
    localSearch(
      entityID: "pr-1234"
      query: "api changes authentication"
      level: 1
    )
  }
  ```

- **Full context**: Hybrid (both!)

### Scenario: "This service just failed, what's impacted?"

**Answer**: PathRAG (trace the failure chain)

```json
POST /entity/service-api/path
{
  "edge_filter": ["depends_on", "calls", "affects"],
  "max_depth": 5,
  "decay_factor": 0.85
}
```

**Why PathRAG**: You need to follow explicit dependency relationships, not find semantically similar services.

### Scenario: "Show me all authentication code in the system"

**Answer**: GraphRAG Global Search (no starting point, comprehensive)

```graphql
query {
  globalSearch(
    query: "authentication authorization security jwt oauth"
    level: 2
    maxCommunities: 5
  )
}
```

**Why GraphRAG**: No specific starting entity, need system-wide semantic search across communities.

### Scenario: "What documentation is related to this error message?"

**Answer**: Hybrid

1. PathRAG: Follow `documented_by` edges
2. GraphRAG: Find semantically similar docs in the same community

```javascript
// Follow explicit documentation links
const docs = await pathQuery({
  start_entity: "error-auth-401",
  edge_filter: ["documented_by", "related_to"],
  max_depth: 2
});

// Find similar documentation
const similarDocs = await localSearch({
  entityID: "error-auth-401",
  query: "authentication failure troubleshooting",
  level: 1
});

// Combine results
return [...docs.entities, ...similarDocs];
```

---

## Performance Comparison

| Strategy | Typical Latency | Use Case | Pre-computation |
|----------|----------------|----------|----------------|
| **PathRAG** | 10-200ms | Trace relationships | None (query-time) |
| **GraphRAG Local** | 50-200ms | Similar entities in cluster | Community detection |
| **GraphRAG Global** | 200-500ms | System-wide semantic search | Community detection |
| **Hybrid** | 100-400ms | Comprehensive context | Community detection |

**Key Takeaway**: PathRAG is fastest (no pre-computation), GraphRAG Global is most comprehensive.

---

## Progressive Learning Path

### Level 1: Start Simple

**Just use PathRAG** if you:

- Have explicit relationships in your data
- Need dependency tracing
- Want simple, fast queries

```bash
curl -X POST /entity/my-entity/path \
  -d '{"max_depth": 3, "edge_filter": ["depends_on"]}'
```

### Level 2: Add GraphRAG Local Search

**Enable community detection** to unlock:

- Semantic similarity search
- Faster "find related" queries
- Community-based organization

```json
{
  "enable_clustering": true,
  "clustering": {
    "algorithm": "label_propagation",
    "embedder": "bm25"
  }
}
```

### Level 3: Use GraphRAG Global Search

**Explore system-wide** patterns:

- No starting entity needed
- Cross-community queries
- Comprehensive results

```graphql
query {
  globalSearch(
    query: "your search terms"
    level: 2
    maxCommunities: 5
  )
}
```

### Level 4: Master Hybrid Queries

**Combine strategies** for maximum insight:

- Semantic + Structural
- Fast initial results + comprehensive expansion
- Best of both approaches

---

## Common Pitfalls

### Pitfall 1: Using PathRAG for Semantic Search

**Problem**: "Find code similar to this" with PathRAG

```json
POST /entity/auth-module/path
{"max_depth": 5}
```

**Why it fails**: PathRAG follows edges blindly, not semantic similarity

**Solution**: Use GraphRAG Local Search

```graphql
localSearch(entityID: "auth-module", query: "authentication", level: 1)
```

### Pitfall 2: Using GraphRAG for Dependency Chains

**Problem**: "Trace dependencies" with GraphRAG

```graphql
localSearch(entityID: "service-api", query: "dependencies", level: 1)
```

**Why it fails**: GraphRAG finds semantically similar entities, not dependency chains

**Solution**: Use PathRAG

```json
POST /entity/service-api/path
{"edge_filter": ["depends_on"], "max_depth": 5}
```

### Pitfall 3: Not Setting Resource Limits

**Problem**: PathRAG query hangs

```json
{"start_entity": "node-1", "max_depth": 10}
```

**Why it fails**: Dense graphs explode without limits

**Solution**: Always set `max_time` and `max_nodes`

```json
{
  "start_entity": "node-1",
  "max_depth": 3,
  "max_nodes": 100,
  "max_time": "200ms"
}
```

---

## Summary

**PathRAG** = "Follow the relationships" (structural)

- Start from entity → walk edges → find connected entities
- Use for: Dependencies, impact analysis, spatial proximity

**GraphRAG Local** = "Find similar in cluster" (semantic)

- Start from entity → find community → search similar entities
- Use for: Related code, similar docs, topic clustering

**GraphRAG Global** = "Search everywhere" (semantic + comprehensive)

- No starting point → search all communities → comprehensive results
- Use for: System-wide patterns, exploratory queries

**Hybrid** = "Best of both worlds"

- Combine semantic and structural
- Maximum context and relevance

---

## Next Steps

- **[Hybrid Query Patterns](06-hybrid-queries.md)** - Advanced combined query examples
- **[Performance Tuning](07-performance-tuning.md)** - Optimize your queries
- **[Production Patterns](08-production-patterns.md)** - Deployment best practices

**Reference:**
- **[Query Fundamentals](01-query-fundamentals.md)** - Conceptual foundation
- **[Configuration Guide](04-configuration-guide.md)** - Enable/disable features based on your data

---

**Remember**: You don't have to choose just one. The most powerful queries combine both strategies to get comprehensive, relevant results. Start simple, add complexity as needed.
