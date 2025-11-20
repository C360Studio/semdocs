# Algorithm Reference

## Native algorithms powering SemStreams semantic capabilities

---

## Overview

SemStreams includes **native Go algorithms** that work everywhere (laptop, server, Raspberry Pi) without external dependencies. This reference documents the algorithms **actually implemented** in the codebase, where they're used, and how to tune them.

**Key principle:** Native algorithms provide baseline functionality. Enhanced services (SemEmbed, SemSummarize) improve quality but are optional.

---

## Text Processing Algorithms

### Tokenization (Simple)

**Purpose:** Split text into searchable terms for BM25 embeddings

**Location:** `semstreams/pkg/embedding/bm25_embedder.go`

**Algorithm:** Unicode-aware whitespace + punctuation splitting

```go
func Tokenize(text string) []string {
    // 1. Normalize whitespace
    text = normalizeWhitespace(text)

    // 2. Split on word boundaries (Unicode-aware)
    tokens := wordBoundaryRegex.FindAllString(text, -1)

    // 3. Lowercase normalization
    for i, token := range tokens {
        tokens[i] = strings.ToLower(token)
    }

    return tokens
}
```

**Example:**

```text
Input:  "UAV-001 battery degraded to 15.2%!"
Output: ["uav", "001", "battery", "degraded", "to", "15", "2"]
```

**Used in:** BM25 lexical embeddings (fallback when neural embeddings unavailable)

**Tuning parameters:**

- `MinTokenLength`: Filter tokens shorter than N chars (default: 2)

**Note:** For neural embeddings, semembed uses transformer-based tokenization (BERT tokenizer via fastembed-rs).

---

## Semantic Similarity Algorithms

### BM25 (Best Matching 25)

**Purpose:** Lexical embeddings with term frequency saturation and document length normalization

**Location:** `semstreams/pkg/embedding/bm25_embedder.go`

**Algorithm:**

```text
BM25(query, doc) = Σ IDF(term) × (tf × (k1 + 1)) / (tf + k1 × (1 - b + b × (|doc| / avgDocLen)))

Where:
  k1 = term frequency saturation (default: 1.5)
  b  = document length normalization (default: 0.75)
  tf = term frequency in document
  |doc| = document length
  avgDocLen = average document length in corpus
```

**Implementation:**

```go
type BM25Embedder struct {
    dimensions int
    k1         float64 // Default: 1.5
    b          float64 // Default: 0.75

    // Document statistics
    docCount       int
    avgDocLength   float64
    termDocCount   map[string]int
}

func (b *BM25Embedder) Score(query []string, docID string) float64 {
    score := 0.0
    docLength := float64(b.docLengths[docID])

    for _, term := range query {
        // IDF component
        docsWithTerm := b.docFreq[term]
        if docsWithTerm == 0 {
            continue
        }
        idf := math.Log((float64(len(b.documents)) - float64(docsWithTerm) + 0.5) /
                        (float64(docsWithTerm) + 0.5) + 1.0)

        // TF component with saturation
        tf := float64(b.termCounts[docID][term])
        normalization := 1.0 - b.b + b.b*(docLength/b.avgDocLength)
        tfComponent := (tf * (b.k1 + 1.0)) / (tf + b.k1*normalization)

        score += idf * tfComponent
    }

    return score
}
```

**Used in:**

- Fallback embeddings when SemEmbed service is unavailable
- Native semantic similarity without neural models
- Edge devices with limited compute

**Performance:**

- **Indexing:** O(N×M) where N = docs, M = avg terms per doc
- **Query:** O(K) where K = query terms
- **Memory:** O(V×N) where V = vocabulary size

**Tuning parameters:**

- **k1** (1.2 - 2.0): Controls term frequency saturation
  - Higher k1 → More weight on repeated terms
  - Lower k1 → Faster saturation (less impact of repetition)
  - Default: 1.5
- **b** (0 - 1): Controls document length normalization
  - b=1 → Full normalization (penalize long docs)
  - b=0 → No normalization
  - Default: 0.75

---

### TF-IDF (Term Frequency-Inverse Document Frequency)

**Purpose:** Measure term importance for community summarization

**Location:** `semstreams/pkg/graphclustering/summarizer.go`

**Algorithm:**

```text
TF(term, doc) = (count of term in doc) / (total terms in doc)

IDF(term, corpus) = log(total documents / documents containing term)

TF-IDF(term, doc, corpus) = TF(term, doc) × IDF(term, corpus)
```

**Used in:**

- Statistical summarization of graph communities
- Identifying important terms in entity clusters
- Template-based summaries before LLM enhancement

**Performance:**

- **Indexing:** O(N×M) where N = docs, M = avg terms per doc
- **Query:** O(K) where K = query terms
- **Memory:** O(V×N) where V = vocabulary size

---

## Graph Algorithms

### PathRAG (Bounded DFS Traversal)

**Purpose:** Resource-constrained graph traversal for edge devices

**Location:** `semstreams/graph/query/interface.go`, `semstreams/graph/query/client.go`

**Algorithm:** Depth-First Search with resource limits and relevance decay

```go
type PathQuery struct {
    StartEntity string        // Starting point
    MaxDepth    int           // Maximum hops from start
    MaxNodes    int           // Maximum nodes to visit
    MaxTime     time.Duration // Timeout
    EdgeFilter  []string      // Which edge types to follow
    DecayFactor float64       // Relevance decay (0.0-1.0)
    MaxPaths    int           // Maximum paths to track
}

func (qc *natsClient) ExecutePathQuery(ctx context.Context, query PathQuery) (*PathResult, error) {
    state := &pathTraversalState{
        visited:  make(map[string]bool),
        entities: make(map[string]*gtypes.EntityState),
        scores:   make(map[string]float64),
    }

    // Initialize start entity with full relevance
    state.scores[query.StartEntity] = 1.0

    // Recursive DFS with bounds
    err := qc.traverseGraph(ctx, query, state, query.StartEntity, 0, []string{query.StartEntity})

    return &PathResult{
        Entities:  state.entities,
        Paths:     state.paths,
        Scores:    state.scores,  // Score = initial_score * (DecayFactor ^ depth)
        Truncated: state.truncated,
    }, err
}
```

**Key Features:**

- **Bounded DFS**: Not BFS or Dijkstra - recursive depth-first with limits
- **Relevance decay**: `score = parentScore * DecayFactor`
- **Resource constraints**: MaxDepth, MaxNodes, MaxTime prevent runaway queries
- **Edge filtering**: Follow only specific relationship types
- **Path tracking**: Maintains full paths from start to leaf nodes

**Used in:**

- Edge device queries (low latency, bounded resources)
- Local graph exploration without requiring full graph in memory
- Context retrieval for RAG (Retrieval-Augmented Generation)

**Performance:**

- **Time:** O(V + E) worst case, bounded by MaxNodes and MaxDepth
- **Space:** O(V) for visited set, O(MaxPaths × MaxDepth) for path storage

**Tuning:**

- **MaxDepth** (2-5): Controls how far from start to explore
  - Depth 2: Immediate neighbors and their neighbors
  - Depth 3: Good balance for most use cases
  - Depth 5+: May hit resource limits on edge devices
- **MaxNodes** (50-500): Prevents exponential growth in dense graphs
- **DecayFactor** (0.7-0.95): Controls relevance falloff with distance
  - 0.7: Aggressive decay, prioritizes nearby nodes
  - 0.85: Balanced decay (default)
  - 0.95: Minimal decay, treats distant nodes as relevant
- **MaxPaths** (10-100): Limits memory for path storage

**Example:**

```text
Graph:
  UAV-001 → belongs_to → fleet-rescue
  UAV-002 → belongs_to → fleet-rescue
  fleet-rescue → stationed_at → base-west

PathQuery(start=UAV-001, MaxDepth=2, DecayFactor=0.85):
  Depth 0: UAV-001 (score=1.0)
  Depth 1: fleet-rescue (score=0.85)
  Depth 2: base-west, UAV-002 (score=0.72)

Result: 4 entities, 2 paths (UAV-001→fleet-rescue→base-west, UAV-001→fleet-rescue→UAV-002)
```

---

### PageRank (Entity Importance)

**Purpose:** Rank entities by relationship importance (similar to Google PageRank)

**Location:** `semstreams/pkg/graphclustering/pagerank.go`

**Algorithm:**

```text
PR(A) = (1-d) + d × Σ(PR(Ti) / C(Ti))

Where:
  d = damping factor (0.85)
  Ti = entities linking to A
  C(Ti) = number of outbound links from Ti
```

**Implementation:**

```go
type PageRankConfig struct {
    Iterations    int     // Default: 20
    DampingFactor float64 // Default: 0.85
    Tolerance     float64 // Default: 1e-6
    TopN          int     // Number of top entities to return
}

func ComputePageRank(ctx context.Context, provider GraphProvider, config PageRankConfig) (*PageRankResult, error) {
    n := len(entities)
    pr := make(map[string]float64)

    // Initialize: equal probability
    for _, e := range entities {
        pr[e] = 1.0 / float64(n)
    }

    // Iterate until convergence
    for iter := 0; iter < maxIter; iter++ {
        newPR := make(map[string]float64)

        for _, entity := range entities {
            rank := (1.0 - damping) / float64(n)

            // Sum contributions from incoming links
            incoming := getIncomingEdges(entity)
            for _, source := range incoming {
                outbound := getOutgoingEdges(source)
                rank += damping * pr[source] / float64(len(outbound))
            }

            newPR[entity] = rank
        }

        pr = newPR
    }

    return &PageRankResult{
        Scores:     pr,
        Ranked:     sortedEntityIDs,
        Converged:  true,
    }, nil
}
```

**Used in:**

- Ranking search results by entity importance
- Identifying "hub" entities in knowledge graphs
- Community detection (combined with LPA)
- Influence propagation analysis

**Performance:**

- **Indexing:** O(I×E) where I = iterations, E = edges
- **Query:** O(1) after precomputation
- **Memory:** O(V) where V = vertices

**Tuning:**

- **Iterations** (10-50): More iterations = better convergence
- **DampingFactor** (0.8-0.9): Probability of following links
  - 0.85 is standard (from original PageRank paper)
- **Tolerance** (1e-6 to 1e-4): Convergence threshold

---

### Label Propagation Algorithm (LPA)

**Purpose:** Community detection in knowledge graphs

**Location:** `semstreams/pkg/graphclustering/lpa.go`

**Algorithm:** Iterative label propagation with hierarchical levels

```go
type LPADetector struct {
    maxIterations int // Default: 100
    levels        int // Default: 3 hierarchical levels
}

func (d *LPADetector) DetectCommunities(ctx context.Context) (map[int][]*Community, error) {
    // Level 1: Fine-grained communities
    // - Each entity starts with unique label
    // - Iteratively adopt most common neighbor label
    // - Converge when labels stabilize

    // Level 2: Medium communities (merge small communities)
    // Level 3: Coarse communities (top-level clusters)

    for level := 1; level <= d.levels; level++ {
        communities := d.detectAtLevel(ctx, level)
        // Optional: Progressive summarization
        if d.summarizer != nil {
            d.summarizer.Summarize(ctx, communities)
        }
    }

    return allCommunities, nil
}
```

**Used in:**

- Hierarchical community detection in entity graphs
- Clustering related entities for summarization
- Progressive summarization (statistical → LLM-enhanced)
- Graph partitioning for distributed processing

**Performance:**

- **Time:** O(I×E) where I = iterations, E = edges
- **Space:** O(V) where V = vertices

**Tuning:**

- **MaxIterations** (50-200): More iterations allow finer community detection
- **Levels** (2-5): Number of hierarchical clustering levels
  - Level 1: Fine-grained (many small communities)
  - Level 2-3: Medium granularity
  - Level 4+: Very coarse clustering

---

## Geospatial Algorithms

### Geohash (Simplified Grid-Based)

**Purpose:** Spatial indexing for location-based queries

**Location:** `semstreams/processor/graph/indexmanager/indexes.go`

**Algorithm:** Integer-based coordinate binning (not standard base32 geohash)

```go
type SpatialIndex struct {
    precision int // Default: 7 (30m resolution)
}

func (si *SpatialIndex) calculateGeohash(lat, lon float64, precision int) string {
    // Precision levels:
    // 4: ~2.5km bins
    // 5: ~600m bins
    // 6: ~120m bins
    // 7: ~30m bins (default)
    // 8: ~5m bins

    var multiplier float64
    switch precision {
    case 7:
        multiplier = 300.0 // ~30m resolution
    default:
        multiplier = 300.0
    }

    // Normalize coordinates to positive integers
    latInt := int(math.Floor((lat + 90.0) * multiplier))
    lonInt := int(math.Floor((lon + 180.0) * multiplier))

    return fmt.Sprintf("geo_%d_%d_%d", precision, latInt, lonInt)
}
```

**Example:**

```text
Location: San Francisco [37.7749, -122.4194]
Geohash (precision=7): geo_7_38324_17362

Nearby locations share similar coordinates:
  geo_7_38324_17362 → San Francisco
  geo_7_38324_17363 → Oakland (nearby)
  geo_7_38325_17362 → Adjacent grid cell
```

**Used in:**

- Spatial indexing of entity locations (latitude/longitude triples)
- Fast proximity queries ("find entities near this location")
- Grid-based clustering of entities by location

**Performance:**

- **Indexing:** O(1) per entity
- **Query:** O(R) where R = results in spatial region
- **Memory:** O(N) where N = number of entities with location

**Tuning:**

- **Precision** (4-8): Grid resolution
  - 4: Coarse (2.5km cells) - city-level queries
  - 7: Fine (30m cells) - street-level queries (default)
  - 8: Very fine (5m cells) - building-level queries

**Note:** This is a simplified geohash for fast binning. It does not calculate actual distances between points (no Haversine). For distance-based queries, you would need to:

1. Query all entities in nearby grid cells
2. Calculate actual distance using Haversine (not currently implemented)
3. Filter by distance threshold

---

## Performance Characteristics

### Algorithm Complexity Summary

| Algorithm | Indexing | Query | Memory | Use Case |
|-----------|----------|-------|--------|----------|
| BM25 | O(N×M) | O(K) | O(V×N) | Lexical embeddings, fallback search |
| TF-IDF | O(N×M) | O(K) | O(V×N) | Community summarization |
| PathRAG (DFS) | O(1) | O(V+E)* | O(V) | Bounded graph exploration |
| PageRank | O(I×E) | O(1) | O(V) | Entity importance ranking |
| LPA | O(I×E) | O(1) | O(V) | Community detection |
| Geohash | O(1) | O(R) | O(N) | Spatial indexing |

**Legend:**

- N = number of documents
- M = average terms per document
- K = query terms
- V = graph vertices (entities)
- E = graph edges (relationships)
- I = algorithm iterations
- R = results in spatial query
- \* Bounded by MaxNodes and MaxDepth

---

## Not Implemented

The following algorithms are **not currently implemented** in SemStreams:

### ❌ Stop Word Filtering

**Status:** Not implemented

**Reason:** Modern transformer models (in SemEmbed) handle this automatically. For native BM25, stop word filtering was deemed unnecessary for technical/domain-specific text.

**Alternative:** Use transformer-based embeddings (SemEmbed) for better semantic understanding.

---

### ❌ Porter Stemmer

**Status:** Not implemented

**Reason:** Stemming can reduce precision for technical terms. SemStreams prioritizes exact term matching (BM25) or neural embeddings (SemEmbed) over stemming.

**Alternative:** Use exact term matching or transformer-based embeddings.

---

### ❌ Dijkstra's Algorithm (Shortest Path)

**Status:** Not implemented

**Current alternative:** PathRAG uses bounded DFS, not shortest path

**Reason:** Edge devices prioritize bounded exploration over optimal paths. PathRAG's decay factor provides relevance-weighted traversal without the overhead of priority queues.

**If needed:** Could be added for use cases requiring true shortest path (e.g., routing, logistics).

---

### ❌ Haversine Distance

**Status:** Not implemented

**Current alternative:** Geohash grid-based proximity

**Reason:** Simplified geohash provides fast spatial binning without trigonometric distance calculations. Sufficient for "nearby entity" queries on edge devices.

**If needed:** Could be added for precise distance-based filtering (e.g., "entities within 500m radius").

---

### ❌ Breadth-First Search (BFS)

**Status:** Not implemented as standalone algorithm

**Current alternative:** PathRAG uses DFS with resource bounds

**Reason:** DFS is simpler to implement recursively and provides similar results with resource constraints. BFS would require queue management and more memory.

---

## Next Steps

- **[Performance Tuning](02-performance-tuning.md)** - Optimize algorithm parameters
- **[Architecture Deep Dive](03-architecture-deep-dive.md)** - How algorithms integrate
- **[PathRAG vs GraphRAG Decisions](01-pathrag-graphrag-decisions.md)** - When to use native vs enhanced

---

**Native algorithms are the foundation** - They provide baseline functionality that works everywhere without external dependencies. Enhanced services (SemEmbed, SemSummarize) build on these primitives for improved quality when resources allow.
