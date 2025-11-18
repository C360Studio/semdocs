# Algorithm Reference

## A plain-language guide to the algorithms powering SemStreams

SemStreams uses well-established algorithms chosen for their reliability, performance, and suitability for edge deployment. This guide explains what these algorithms do, why they were chosen, and when they're used.

## Overview

| Algorithm | Purpose | Where Used | Time Complexity |
|-----------|---------|------------|-----------------|
| [BM25](#bm25) | Text search and ranking | Semantic search, embeddings | O(n × m) |
| [Label Propagation (LPA)](#label-propagation-lpa) | Community detection | GraphRAG clustering | O(n × k) |
| [PageRank](#pagerank) | Entity importance ranking | Community summaries, entity weighting | O(n × k) |
| [TF-IDF](#tf-idf) | Statistical summarization | Community summaries | O(n × m) |
| [Dijkstra's Algorithm](#dijkstras-algorithm) | Shortest path finding | PathRAG graph traversal | O((V + E) log V) |

**Design Philosophy**: Choose algorithms that are:

- **Deterministic** over probabilistic (repeatable results)
- **Efficient** for real-time processing (streaming, not batch)
- **Embeddable** in pure Go (no Python/GPU dependencies for core features)
- **Well-understood** and battle-tested (not experimental)

---

## BM25

**Best Match 25** - Probabilistic text ranking algorithm

### What It Does

BM25 ranks documents by how relevant they are to a search query. It's like a smart version of "find words in text" that understands:

- **Word frequency**: Words that appear more often are more relevant (but with diminishing returns)
- **Document length**: Shorter documents with matches rank higher than longer ones
- **Word rarity**: Rare words matching are more significant than common words

### Example

**Query**: "battery failure drone"

```text
Document A: "The battery failed during the drone mission."
  - Contains: battery (rare), failed (relates to failure), drone (rare)
  - Length: Short (11 words)
  - Score: HIGH ✅

Document B: "The system uses a battery for power. Many systems have batteries."
  - Contains: battery (common word, appears 2x)
  - Length: Longer (11 words)
  - Missing: failure, drone
  - Score: LOW ❌
```

### Why BM25?

**Chosen over neural embeddings as default** because:

| Aspect | BM25 | Neural Embeddings |
|--------|------|-------------------|
| **Dependencies** | Pure Go, zero dependencies | Requires embedding service (semembed) |
| **Performance** | ~1.4μs per text | ~50-100ms per batch |
| **Deterministic** | Same input = same output (always) | Can vary with model versions |
| **Offline** | Works without internet | May need model downloads |
| **Resource Use** | Minimal (~10MB RAM) | Significant (~512MB-1GB) |
| **Results** | Exact keyword matches guaranteed | Semantic similarity, may miss exact terms |

**When to upgrade to neural**: When you need semantic understanding (synonyms, concepts) and have resources for embedding service

### Configuration

```json
{
  "embedding": {
    "provider": "bm25"  // Default, always available
  }
}
```

### Used In

- **Semantic search** (default mode)
- **GraphRAG** local/global search (when semembed unavailable)
- **Entity similarity** for deduplication

### Trade-offs

✅ **Strengths**:

- Fast, reliable, deterministic
- No external dependencies
- Works offline
- Excellent for exact keyword matching

❌ **Limitations**:

- Doesn't understand synonyms ("car" ≠ "automobile")
- No concept similarity ("battery failure" doesn't match "power issue")
- Vocabulary mismatch problems (technical jargon)

**Solution**: Use BM25 as baseline, optionally enhance with neural embeddings when semantic understanding needed

---

## Label Propagation (LPA)

**Label Propagation Algorithm** - Fast community detection in graphs

### What It Does

LPA finds groups (communities) of entities that are more connected to each other than to the rest of the graph. Think of it like finding cliques in a social network - people who interact frequently form communities.

**How it works**:

1. **Initialize**: Give every entity a unique label (its own ID)
2. **Propagate**: Each entity adopts the most common label among its neighbors
3. **Iterate**: Repeat until labels stop changing
4. **Result**: Entities with the same label belong to the same community

### Example

**Graph**: Software specifications and their dependencies

```text
Before LPA:
┌──────┐     ┌──────┐     ┌──────┐
│Auth  │────→│Users │────→│Perms │
│Spec  │     │Spec  │     │Spec  │
└──────┘     └──────┘     └──────┘
   ↓                          ↓
┌──────┐                  ┌──────┐
│OAuth │                  │RBAC  │
│Spec  │                  │Spec  │
└──────┘                  └──────┘

After LPA (3 iterations):
Community 1 (Authentication):
  - Auth Spec
  - OAuth Spec
  - Users Spec

Community 2 (Authorization):
  - Perms Spec
  - RBAC Spec
  - Users Spec (bridge - appears in both)
```

### Why LPA?

**Chosen over Leiden/Louvain** (used by Microsoft GraphRAG) because:

| Algorithm | Time Complexity | Deterministic | Pure Go | Streaming | Quality |
|-----------|----------------|---------------|---------|-----------|---------|
| **LPA** | O(n × k) | No* | ✅ Yes | ✅ Yes | Good |
| **Leiden** | O(n log n) | Yes | ❌ Needs Python | ❌ Batch | Better |
| **Louvain** | O(n log n) | No | ❌ Needs Python | ❌ Batch | Better |

*LPA is non-deterministic due to tie-breaking, but **stabilized** in SemStreams with sorted neighbor iteration

**Why this matters for SemStreams**:

- **Edge deployment**: No Python dependencies
- **Real-time**: Process entities as they arrive (streaming)
- **Performance**: Fast enough for large graphs (millions of entities)
- **Good enough**: Quality difference vs Leiden/Louvain is minimal for most graphs

### Configuration

```json
{
  "graph_processor": {
    "community_detection": {
      "enabled": true,
      "algorithm": "label_propagation",
      "max_levels": 3,  // Build hierarchy (level 0 → level 1 → level 2)
      "min_community_size": 5,
      "max_iterations": 10
    }
  }
}
```

### Hierarchical Communities

LPA builds **multi-level hierarchies** for GraphRAG:

```text
Level 0 (Fine-grained):
  Community 0.1: [spec-1, spec-2, spec-3]
  Community 0.2: [spec-4, spec-5]
  Community 0.3: [spec-6, spec-7, spec-8, spec-9]

Level 1 (Medium):
  Community 1.1: [Community 0.1, Community 0.2]
  Community 1.2: [Community 0.3]

Level 2 (Coarse):
  Community 2.1: [Community 1.1, Community 1.2]
```

**Local search**: Queries level 0 (fine-grained communities)
**Global search**: Queries level 2 (coarse-grained summaries)

### Used In

- **GraphRAG** community detection
- **Hierarchical summaries** (local → global search)
- **Entity clustering** for organization

### Trade-offs

✅ **Strengths**:

- Very fast (linear time complexity)
- Pure Go (no external dependencies)
- Streaming-friendly (incremental updates)
- Scales to millions of entities

❌ **Limitations**:

- Non-deterministic (tie-breaking randomness) - mitigated with sorted iteration
- Produces uneven community sizes
- May create singleton communities

**Mitigation**: Configure `min_community_size` to merge small communities

---

## PageRank

**Google's Page Ranking Algorithm** - Importance scoring based on graph structure

### What It Does

PageRank measures the importance of entities in a graph by counting the number and quality of links pointing to them. Think of it as a "vote" system where:

- **Each incoming link is a vote** for the entity's importance
- **Votes from important entities count more** than votes from unimportant ones
- **Entities that link to many others** spread their vote thinly

**The insight**: Important entities are linked to by other important entities.

### Example

**Graph**: Software specifications with cross-references

```text
Initial scores (all equal):
┌────────┐ score: 1.0
│Auth    │────────┐
│Spec    │        │
└────────┘        ↓
                ┌────────┐ score: 1.0
                │Users   │
┌────────┐      │Spec    │
│OAuth   │──────►        │
│Spec    │ 1.0  └────────┘
└────────┘        ↑
                  │
┌────────┐        │
│Perms   │────────┘
│Spec    │ 1.0
└────────┘

After PageRank iterations:
┌────────┐ score: 0.8
│Auth    │────────┐
│Spec    │        │
└────────┘        ↓
                ┌────────┐ score: 2.4 ← HIGHEST (3 incoming links)
                │Users   │
┌────────┐      │Spec    │
│OAuth   │──────►        │
│Spec    │ 0.9  └────────┘
└────────┘        ↑
                  │
┌────────┐        │
│Perms   │────────┘
│Spec    │ 0.9
└────────┘

Result: Users Spec is most important (central to authentication/authorization)
```

### Why PageRank?

**Chosen for entity importance weighting** because:

| Algorithm | Measures | Time Complexity | Best For |
|-----------|----------|-----------------|----------|
| **PageRank** | Structural importance | O(n × k) | Graph-based relevance |
| **Degree Centrality** | Simple link count | O(n) | Quick approximation |
| **Betweenness** | Path importance | O(n³) | Critical connectors |
| **Eigenvector** | Network influence | O(n × k) | Similar to PageRank |

**Why PageRank wins**:

- Considers link quality (not just quantity)
- Fast convergence (typically 10-20 iterations)
- Well-understood and proven (Google Search for 25 years)
- Pure Go implementation (no dependencies)

### How It Works with TF-IDF

**Community summaries use BOTH algorithms together:**

```text
Step 1: PageRank → Rank entities by importance
  Users Spec: 2.4
  Auth Spec: 0.8
  OAuth Spec: 0.9

Step 2: TF-IDF → Extract key terms from entity descriptions
  Users Spec: ["authentication", "authorization", "user", "roles"]
  Auth Spec: ["oauth", "tokens", "claims"]

Step 3: Combine → Weight TF-IDF scores by PageRank
  "authentication" (from Users Spec): TF-IDF 0.8 × PageRank 2.4 = 1.92
  "oauth" (from Auth Spec): TF-IDF 0.9 × PageRank 0.8 = 0.72

Result: Summary focuses on terms from important entities
  "This community covers authentication and authorization with user roles..."
```

**Why combine them?**

- **PageRank alone**: Knows WHICH entities are important, not WHAT they're about
- **TF-IDF alone**: Knows WHAT words are important, not FROM WHERE
- **Together**: Important words from important entities = best summary

### Configuration

```json
{
  "graph_processor": {
    "community_detection": {
      "use_pagerank": true,  // Enable PageRank weighting
      "pagerank_iterations": 20,
      "pagerank_damping": 0.85  // Standard damping factor
    }
  }
}
```

### Used In

- **Community summaries** - Weight entity importance
- **GraphRAG global search** - Rank community significance
- **Entity deduplication** - Choose canonical entity from duplicates (keep highest PageRank)
- **Query results** - Rank search results by entity importance

### The Damping Factor (0.85)

**What it means**: 85% probability a user follows a link, 15% they jump randomly

```text
PageRank formula:
PR(A) = (1-d) + d × Σ(PR(T) / C(T))

Where:
- d = damping factor (0.85)
- PR(T) = PageRank of pages linking to A
- C(T) = number of outbound links from T
```

**Why 0.85?**: Google's original value, empirically proven optimal for most graphs

**Effect of damping**:

- **Higher (0.9-0.99)**: More weight to link structure, slower convergence
- **Lower (0.5-0.7)**: More random distribution, faster convergence
- **0.85**: Good balance of accuracy and speed

### Trade-offs

✅ **Strengths**:

- Measures true importance (not just link count)
- Fast convergence (typically 10-20 iterations)
- Pure Go (no external dependencies)
- Deterministic results (same graph = same scores)
- Proven algorithm (25+ years in production)

❌ **Limitations**:

- Requires multiple iterations (not instant)
- Can favor older entities (more time to accumulate links)
- Sensitive to graph structure changes

**Mitigation**: Run PageRank incrementally, cache results, recompute on significant graph changes

---

## TF-IDF

**Term Frequency-Inverse Document Frequency** - Statistical importance scoring

### What It Does

TF-IDF finds the most important words in a document by balancing:

- **Term Frequency (TF)**: How often a word appears in the document
- **Inverse Document Frequency (IDF)**: How rare the word is across all documents

**Formula**: `TF-IDF = (word frequency in doc) × log(total docs / docs containing word)`

### Example

**3 Documents** about drones:

```text
Doc 1: "The drone battery failed during flight."
Doc 2: "The drone completed the mission successfully."
Doc 3: "The system uses a battery for power."

Word "drone":
  - TF in Doc 1: 1/7 = 0.14
  - IDF: log(3/2) = 0.18  (appears in 2 of 3 docs)
  - TF-IDF: 0.14 × 0.18 = 0.025

Word "battery":
  - TF in Doc 1: 1/7 = 0.14
  - IDF: log(3/2) = 0.18  (appears in 2 of 3 docs)
  - TF-IDF: 0.14 × 0.18 = 0.025

Word "failed":
  - TF in Doc 1: 1/7 = 0.14
  - IDF: log(3/1) = 0.48  (appears in only 1 doc - RARE!)
  - TF-IDF: 0.14 × 0.48 = 0.067  ← HIGHEST SCORE

Result: "failed" is the most significant word in Doc 1
```

### Why TF-IDF?

**Chosen as baseline for community summaries** because:

| Aspect | TF-IDF + PageRank | LLM Summaries |
|--------|-------------------|---------------|
| **Speed** | Instant (pure math) | Slow (API calls) |
| **Cost** | Free (local compute) | $$$ (tokens) |
| **Dependencies** | None | semsummarize service |
| **Quality** | Good (statistical + importance) | Better (semantic understanding) |
| **Offline** | ✅ Works anywhere | ❌ Needs service |

**Progressive enhancement**: Start with TF-IDF + PageRank, optionally upgrade to LLM summaries

**How they work together**: See [PageRank + TF-IDF combination](#how-it-works-with-tf-idf) above

### Configuration

```json
{
  "graph_processor": {
    "community_detection": {
      "summary_method": "tfidf",  // Default
      "llm_enhancement": false     // Optional: enable semsummarize
    }
  }
}
```

### Used In

- **GraphRAG community summaries** (combined with PageRank for entity weighting)
- **Global search** topic extraction
- **Entity description** generation (when no LLM available)

**Note**: TF-IDF is always combined with PageRank for better summaries (important words from important entities)

### Trade-offs

✅ **Strengths**:

- Extremely fast (milliseconds)
- Zero dependencies
- Deterministic results
- Works offline

❌ **Limitations**:

- Keyword-based (not semantic)
- Doesn't understand context
- May include stop words if not filtered

**Enhancement**: Use semsummarize (LLM) for semantic summaries when quality > speed

---

## Dijkstra's Algorithm

**Shortest path finding** - Graph traversal with weighted edges

### What It Does

Dijkstra's algorithm finds the shortest path between two nodes in a graph with weighted edges. In SemStreams, this enables **PathRAG queries** that traverse entity relationships.

**How it works**:

1. **Start**: Begin at source entity, distance = 0
2. **Explore**: Visit unvisited neighbors, calculate distances
3. **Choose**: Always visit the closest unvisited node next
4. **Repeat**: Until destination found or all reachable nodes visited

### Example

**Query**: Find dependency chain from `spec-A` to `spec-Z`

```text
Graph:
spec-A → spec-B (weight: 1)
spec-A → spec-C (weight: 5)
spec-B → spec-D (weight: 2)
spec-C → spec-D (weight: 1)
spec-D → spec-Z (weight: 1)

Dijkstra's traversal:
1. Start: spec-A (distance = 0)
2. Visit: spec-B (distance = 1) ← closest neighbor
3. Visit: spec-D (distance = 3) via spec-B
4. Visit: spec-Z (distance = 4) via spec-D
   Alternative path via spec-C: 5 + 1 + 1 = 7 (longer)

Result: spec-A → spec-B → spec-D → spec-Z (total distance: 4)
```

### Why Dijkstra's?

**Chosen for PathRAG traversal** because:

| Algorithm | Use Case | Time Complexity | Best For |
|-----------|----------|-----------------|----------|
| **Dijkstra's** | Weighted shortest path | O((V + E) log V) | Real-time queries with edge weights |
| **BFS** | Unweighted shortest path | O(V + E) | Simple reachability |
| **A*** | Heuristic-guided search | O(E) best case | Known target with distance estimate |

**SemStreams needs**: Weighted edges (importance scores) + real-time performance

### Configuration

PathRAG uses Dijkstra's automatically when edge weights are present:

```json
{
  "graph_query": {
    "use_edge_weights": true,  // Enable Dijkstra's
    "max_depth": 5
  }
}
```

### Used In

- **PathRAG** shortest path queries
- **Dependency chain** analysis (spec dependencies)
- **Impact analysis** (what depends on this entity?)
- **Relationship traversal** with importance scores

### Trade-offs

✅ **Strengths**:

- Optimal shortest path guaranteed
- Handles weighted edges naturally
- Efficient for sparse graphs
- Deterministic results

❌ **Limitations**:

- Slower than BFS for unweighted graphs
- Requires positive edge weights
- Memory overhead for priority queue

**Optimization**: SemStreams uses BFS when all edges have equal weight (faster)

---

## Algorithm Selection Guidelines

### When to Use Which Algorithm

**Semantic Search**:

- **Start with**: BM25 (default)
- **Upgrade to**: Neural embeddings (semembed) when semantic understanding needed

**Community Detection**:

- **Use**: Label Propagation (LPA) - only option, optimized for edge deployment

**Entity Importance**:

- **Use**: PageRank - automatic, runs with community detection

**Summarization**:

- **Start with**: TF-IDF + PageRank (default, weighted by entity importance)
- **Upgrade to**: LLM summaries (semsummarize) when quality > speed

**Graph Traversal**:

- **Weighted paths**: Dijkstra's algorithm
- **Unweighted paths**: BFS (automatic optimization)

### Progressive Enhancement Pattern

SemStreams follows a **progressive enhancement** pattern for all algorithms:

```text
Baseline (Always Works)          Enhancement (Optional)
─────────────────────────        ──────────────────────
BM25                       →     Neural embeddings (semembed)
LPA                        →     (No enhancement - LPA is optimal)
PageRank                   →     (No enhancement - always used with LPA)
TF-IDF + PageRank          →     LLM summaries (semsummarize)
Dijkstra's/BFS             →     (No enhancement - optimal for use case)
```

**Philosophy**: Start simple and reliable, enhance when resources allow

---

## Performance Characteristics

### Benchmarks (10,000 entities)

| Algorithm | Operation | Time | Memory |
|-----------|-----------|------|--------|
| **BM25** | Single search | 1.4μs | ~10MB |
| **LPA** | Full clustering | 45ms | ~50MB |
| **PageRank** | Importance scoring (20 iterations) | 35ms | ~30MB |
| **TF-IDF + PageRank** | Weighted summary generation | 5ms | ~10MB |
| **Dijkstra's** | Path query (depth=5) | 8ms | ~20MB |

**Takeaway**: All algorithms are designed for **real-time, streaming** performance

---

## Research Papers & References

### BM25

- **Paper**: Robertson & Zaragoza (2009) - "The Probabilistic Relevance Framework: BM25 and Beyond"
- **Why it matters**: Industry standard for 20+ years (Elasticsearch, Lucene)

### Label Propagation

- **Paper**: Raghavan et al. (2007) - "Near linear time algorithm to detect community structures in large-scale networks"
- **Why it matters**: Fast enough for real-time graph processing

### PageRank

- **Paper**: Page et al. (1998) - "The PageRank Citation Ranking: Bringing Order to the Web"
- **Why it matters**: Google's foundation algorithm, 25+ years proven at web scale

### TF-IDF

- **Paper**: Salton & Buckley (1988) - "Term-weighting approaches in automatic text retrieval"
- **Why it matters**: Foundation of information retrieval (40+ years proven)

### Dijkstra's Algorithm

- **Paper**: Dijkstra (1959) - "A note on two problems in connexion with graphs"
- **Why it matters**: Classic algorithm, proven optimal for shortest path

---

## Related Documentation

**Query Strategies**:

- [Choosing Your Query Strategy](../guides/choosing-query-strategy.md) - When to use PathRAG vs GraphRAG
- [GraphRAG Guide](../guides/graphrag.md) - LPA community detection deep dive
- [PathRAG Guide](../guides/pathrag.md) - Dijkstra's traversal patterns

**Configuration**:

- [Embedding Strategies](../guides/embedding-strategies.md) - BM25 vs neural embeddings
- [Configuration Reference](../deployment/configuration.md) - Algorithm tuning parameters

---

**Summary**: SemStreams uses battle-tested algorithms chosen for **reliability over novelty**, **performance over perfection**, and **embeddability over external dependencies**.

**Core algorithm stack**:

- **BM25** for text search (baseline, with optional neural enhancement)
- **LPA** for community detection (pure Go, streaming-friendly)
- **PageRank** for entity importance (combined with TF-IDF)
- **TF-IDF** for summarization (weighted by PageRank scores)
- **Dijkstra's** for graph traversal (optimal shortest path)

Every algorithm has a fast baseline with optional enhancements via external services.
