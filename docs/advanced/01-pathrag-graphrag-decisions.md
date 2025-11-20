# PathRAG vs GraphRAG: Configuration Decision Guide

**Choosing the right configuration for your data**

---

## Overview

SemStreams excels when you need **structured relationships (PathRAG) OR semantic search (GraphRAG) OR both** - but you don't need both for every use case. The flexibility to use only what you need is a core feature.

This guide helps you decide which capabilities to enable based on your data characteristics.

---

## Understanding Fit: It's About Your Data

Different data types map to different capabilities. Here's how to think about it:

| Data Type | PathRAG (Relationships) | GraphRAG (Semantic Search) | Best Configuration |
|-----------|------------------------|---------------------------|-------------------|
| **Robotics telemetry** (battery: 85.2, gps: [lat,lon]) | ✅ Excellent (topology, "what's near?") | ❌ Poor (no text to search) | PathRAG only, skip embeddings |
| **Robotics + incident logs** (telemetry + "battery degraded...") | ✅ Excellent (which drone, mission) | ✅ Excellent (find similar incidents) | Full stack (PathRAG + GraphRAG) |
| **Software specs/docs** (dependencies + descriptions) | ✅ Excellent (dependency chains) | ✅ Excellent (semantic similarity) | Full stack (PathRAG + GraphRAG) |
| **IoT sensor networks** (temp, humidity, device IDs) | ✅ Excellent (device topology) | ❌ Poor (numeric only) | PathRAG only, skip embeddings |
| **DevOps service mesh** (services + dependencies) | ✅ Excellent (call graphs) | ⚠️ Maybe (if you have logs/docs) | PathRAG primary, GraphRAG optional |
| **Time-series only** (metrics over time) | ❌ Poor fit | ❌ Poor fit | **Use InfluxDB/Prometheus instead** |

---

## When SemStreams Excels (Full Stack)

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

---

## When to Use PathRAG Only (Skip GraphRAG)

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

**Configuration:**

```json
{
  "components": {
    "graph": {
      "type": "processor",
      "config": {
        "indexer": {
          "indexes": {
            "predicate": true,
            "incoming": true,
            "spatial": true,
            "temporal": true
          },
          "embedding": {
            "enabled": false
          }
        }
      }
    }
  }
}
```

---

## When to Use GraphRAG Only (Skip PathRAG)

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

Configuration: Focus on embeddings, lightweight relationship tracking
```

**Use cases:**

- Document search (rich text, minimal relationships)
- Log analysis (semantic patterns in unstructured logs)
- Content recommendation (similarity-based)

---

## When SemStreams Is NOT the Right Fit

**Be honest with yourself - sometimes alternatives are better:**

| Your Need | Better Alternative | Why |
|-----------|-------------------|-----|
| **Pure time-series metrics** (CPU, memory over time) | InfluxDB, Prometheus, TimescaleDB | Optimized for time-based aggregation, downsampling |
| **Pure full-text search** (search engine) | Elasticsearch, Meilisearch | Mature ranking, highlighting, faceting |
| **Traditional OLAP** (sum, group, aggregate) | ClickHouse, Druid | Columnar storage, SQL analytics |
| **Simple key-value lookups** (cache, session store) | Redis, Memcached | Faster, simpler for basic get/set |
| **Batch ETL pipelines** (nightly data warehouse loads) | Airflow + dbt + Snowflake | Purpose-built for batch workflows |

---

## Anti-Patterns (Common Mistakes)

### 1. Embedding everything in high-volume telemetry

❌ **Problem**: 10 drones × 10 Hz = 600 embeddings/sec → 103GB in 24 hours
✅ **Solution**: Use PathRAG only, skip semantic_enrichment

### 2. Using it as a general database

❌ **Problem**: No SQL, no transactions, no joins
✅ **Solution**: Use PostgreSQL for transactional data, SemStreams for graph queries

### 3. Trying semantic search on numeric-only data

❌ **Problem**: "battery: 85.2" has no semantic meaning
✅ **Solution**: Skip GraphRAG, use PathRAG for relationships

### 4. Not filtering by entity type

❌ **Problem**: Embedding noise data (heartbeats, health checks)
✅ **Solution**: Configure type filters, selective semantic indexing

---

## The Killer Feature: Use What You Need

**SemStreams is composable** - you're not locked into using every capability:

**Configuration A: PathRAG only (IoT fleet)**
```json
{
  "components": {
    "graph": {
      "config": {
        "indexer": {
          "indexes": {
            "predicate": true,
            "incoming": true,
            "spatial": true
          },
          "embedding": {"enabled": false}
        }
      }
    }
  }
}
```

**Configuration B: GraphRAG only (document search)**
```json
{
  "components": {
    "graph": {
      "config": {
        "indexer": {
          "indexes": {
            "predicate": false,
            "incoming": false
          },
          "embedding": {"enabled": true}
        }
      }
    }
  }
}
```

**Configuration C: Full stack (software knowledge graph)**
```json
{
  "components": {
    "graph": {
      "config": {
        "indexer": {
          "indexes": {
            "predicate": true,
            "incoming": true,
            "alias": true,
            "spatial": true,
            "temporal": true
          },
          "embedding": {"enabled": true}
        }
      }
    }
  }
}
```

**Pick your configuration based on your data characteristics, not the framework's capabilities.**

---

## Decision Tree

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

---

## Performance Impact

### Memory Usage Per Entity

| Configuration | Memory per Entity | Example (1M entities) |
|---------------|-------------------|----------------------|
| PathRAG only | ~200 bytes | 200 MB |
| GraphRAG only | ~2KB (with embeddings) | 2 GB |
| Full stack | ~2.2KB | 2.2 GB |

### Processing Throughput

| Configuration | Messages/sec (single instance) | Bottleneck |
|---------------|-------------------------------|------------|
| PathRAG only | ~10,000 msg/sec | CPU (relationship indexing) |
| GraphRAG only | ~1,000 msg/sec | Embedding service (SemEmbed) |
| Full stack | ~1,000 msg/sec | Embedding service |

**Optimization tips:**

- PathRAG only: Scale horizontally, partition by entity type
- GraphRAG only: GPU acceleration for SemEmbed, batch embeddings
- Full stack: Separate instances for PathRAG (edge) and GraphRAG (cloud)

---

## Edge Deployment Considerations

**Edge (resource-constrained):**
- Default: PathRAG only (native Go algorithms)
- Memory: 200 bytes/entity
- Works on: Raspberry Pi, IoT gateways, drones

**Cloud/Server (resource-rich):**
- Default: Full stack (PathRAG + GraphRAG)
- Memory: 2KB/entity
- Works on: Laptop, server, cloud instances

**Hybrid Pattern:**
```
Edge (PathRAG only) → Selective sync → Cloud (Full stack)
                                           ↓
                                    Semantic search
                                    Cross-edge queries
```

---

## Next Steps

- **[Native vs Enhanced Semantic](../semantic/01-overview.md)** - Understanding semantic capabilities
- **[Graph Indexing](../graph/04-indexing.md)** - Index types and configuration
- **[Edge Federation](../edge/03-federation.md)** - Edge-to-cloud patterns
- **[Performance Tuning](02-performance-tuning.md)** - Optimization strategies

---

**Remember**: This is a feature, not a limitation. Configure SemStreams based on what your data needs, not what the framework offers.
