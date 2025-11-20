# Performance Tuning Guide

**Optimizing SemStreams for throughput, latency, and resource efficiency**

---

## Overview

SemStreams performance depends on your configuration choices. This guide helps you optimize for different workload patterns.

---

## Memory Optimization

### Per-Entity Memory Footprint

| Configuration | Memory per Entity | 1M Entities | 10M Entities |
|---------------|-------------------|-------------|--------------|
| PathRAG only (no embeddings) | ~200 bytes | 200 MB | 2 GB |
| GraphRAG only (embeddings) | ~2KB | 2 GB | 20 GB |
| Full stack (PathRAG + GraphRAG) | ~2.2KB | 2.2 GB | 22 GB |

**Optimization strategies:**

```json
{
  "graph": {
    "config": {
      "indexer": {
        "indexes": {
          "predicate": true,    // Essential for relationship queries
          "incoming": true,     // Essential for reverse lookups
          "alias": false,       // Disable if not using entity aliases
          "spatial": false,     // Disable if no geo queries
          "temporal": false     // Disable if no time-based queries
        },
        "embedding": {
          "enabled": false      // Disable for PathRAG-only workloads
        }
      }
    }
  }
}
```

**Index selection impact:**

- **Predicate index**: ~50 bytes/entity (relationship type lookups)
- **Incoming index**: ~50 bytes/entity (reverse relationship queries)
- **Alias index**: ~30 bytes/entity (entity name resolution)
- **Spatial index**: ~40 bytes/entity (geospatial queries)
- **Temporal index**: ~40 bytes/entity (time-based queries)
- **Embeddings**: ~1.5KB/entity (vector similarity search)

**Recommendation**: Start with predicate + incoming only, add others as needed.

---

## Throughput Optimization

### Message Processing Rates

| Configuration | Msgs/sec (single instance) | Bottleneck | Scaling Strategy |
|---------------|---------------------------|------------|------------------|
| PathRAG only | ~10,000 msg/sec | CPU (indexing) | Horizontal (partition by entity type) |
| GraphRAG only | ~1,000 msg/sec | SemEmbed service | GPU acceleration, batch embeddings |
| Full stack | ~1,000 msg/sec | SemEmbed service | Separate PathRAG (edge) from GraphRAG (cloud) |

### Worker Pool Configuration

```json
{
  "graph": {
    "config": {
      "workers": 8,           // CPU cores - 1 (leave headroom)
      "queue_size": 5000,     // Buffer for traffic spikes
      "message_handler": {
        "processing_timeout": "5s",
        "max_retries": 5
      }
    }
  }
}
```

**Tuning guidelines:**

- **workers**: Set to CPU cores for CPU-bound workloads (PathRAG), lower for I/O-bound (GraphRAG with external services)
- **queue_size**: 1000-10000 depending on traffic variability (higher = more buffering during spikes)
- **processing_timeout**: Balance between slow operations and false timeouts (3-10s typical)

### Batch Processing

```json
{
  "graph": {
    "config": {
      "data_manager": {
        "edge_batch_size": 100    // Batch relationship updates
      },
      "indexer": {
        "batch_processing": {
          "size": 50              // Batch index updates
        }
      }
    }
  }
}
```

**Batch size tuning:**

- **Small batches (10-50)**: Lower latency, higher overhead
- **Large batches (100-500)**: Higher throughput, increased latency
- **Recommendation**: 50-100 for balanced latency/throughput

---

## Latency Optimization

### Query Performance

```json
{
  "graph": {
    "config": {
      "querier": {
        "cache_enabled": true,
        "cache_ttl": "15m",
        "max_cache_size": 5000,
        "query_timeout": "10s",
        "result_limit": 1000
      }
    }
  }
}
```

**Cache strategy:**

- **cache_enabled**: Always enable for read-heavy workloads
- **cache_ttl**: Balance freshness vs hit rate (5-30m typical)
- **max_cache_size**: Monitor cache hit rate, increase if evictions are frequent
- **result_limit**: Cap to prevent runaway queries (100-10000 depending on use case)

### Index Selection for Query Patterns

| Query Pattern | Required Indexes | Latency (1M entities) |
|--------------|------------------|----------------------|
| Find entity by ID | None (KV lookup) | <1ms |
| Find by relationship type | Predicate | 5-20ms |
| Find incoming relationships | Incoming | 5-20ms |
| Find by alias/name | Alias | 5-20ms |
| Geospatial query | Spatial | 10-50ms |
| Time-based query | Temporal | 10-50ms |
| Semantic similarity | Embedding | 50-200ms |

**Query optimization:**

1. Use specific filters (entity type, predicate type) to narrow search space
2. Limit result sets with `result_limit`
3. Enable query caching for repeated queries
4. Use batch queries when fetching multiple entities

---

## Network Optimization

### NATS Configuration

```json
{
  "nats": {
    "urls": ["nats://localhost:4222"],
    "max_reconnect": 60,
    "reconnect_wait": "2s",
    "max_pending": 1024,      // Messages buffered per subscriber
    "max_payload": 1048576    // 1MB max message size
  }
}
```

**Tuning for high-volume workloads:**

- **max_pending**: Increase for traffic spikes (1024-10000)
- **max_payload**: Decrease if messages are small (saves memory)
- **reconnect_wait**: Exponential backoff for network issues

### Subject Organization

```
events.graph.entity.drone     // Specific entity type
events.graph.entity.sensor    // Another entity type
events.graph.entity.*         // Subscribe to all entities
```

**Partitioning strategy:**

- Partition by entity type for independent scaling
- Use wildcards sparingly (performance impact on high-volume subjects)
- Separate high-volume subjects from low-volume subjects

---

## Storage Optimization

### NATS KV Bucket Configuration

```json
{
  "graph": {
    "config": {
      "data_manager": {
        "kv_bucket": "graph_entities",
        "kv_options": {
          "max_retries": 15,
          "timeout": "10s",
          "history": 5,            // Versions to keep (1-64)
          "ttl": 0                 // 0 = no expiration
        }
      }
    }
  }
}
```

**KV bucket tuning:**

- **history**: Set to 1 if you don't need version history (saves storage)
- **ttl**: Set expiration for ephemeral data (saves storage)
- **replicas**: Set to 3 for production (high availability)

### Object Store for Large Data

For entities with large payloads (>1MB):

```json
{
  "objectstore": {
    "config": {
      "bucket_name": "large_entities",
      "data_cache": {
        "enabled": true,
        "strategy": "hybrid",    // LRU + frequency
        "max_size": 5000,
        "ttl": "1h",
        "cleanup_interval": "5m"
      }
    }
  }
}
```

**Cache strategies:**

- **lru**: Evict least recently used (temporal locality)
- **lfu**: Evict least frequently used (popular items)
- **hybrid**: Balance between LRU and LFU

---

## Edge Deployment Optimization

### Resource-Constrained Edge (Raspberry Pi)

```json
{
  "graph": {
    "config": {
      "workers": 2,            // Limited CPU cores
      "queue_size": 1000,      // Limited memory
      "indexer": {
        "indexes": {
          "predicate": true,
          "incoming": true,
          "alias": false,
          "spatial": false,
          "temporal": false
        },
        "embedding": {
          "enabled": false       // No GPU, limited RAM
        }
      }
    }
  }
}
```

**Edge optimization checklist:**

- ✅ Disable unnecessary indexes
- ✅ Disable embeddings (use native semantic only)
- ✅ Reduce worker count (2-4 workers)
- ✅ Smaller queue sizes (500-1000)
- ✅ Selective sync (only send exceptions to cloud)

### Laptop/Server (Full Stack)

```json
{
  "graph": {
    "config": {
      "workers": 8,
      "queue_size": 5000,
      "indexer": {
        "indexes": {
          "predicate": true,
          "incoming": true,
          "alias": true,
          "spatial": true,
          "temporal": true
        },
        "embedding": {
          "enabled": true
        }
      }
    }
  }
}
```

**Full stack optimization:**

- ✅ Enable all indexes for comprehensive queries
- ✅ Enable SemEmbed for high-quality semantic search
- ✅ Increase worker pool (8-16 workers)
- ✅ Enable query caching
- ✅ Use GPU acceleration for embeddings

---

## Monitoring and Profiling

### Key Metrics to Track

**Throughput:**
- Messages processed/sec
- Index updates/sec
- Query rate/sec

**Latency:**
- Message processing time (P50, P95, P99)
- Query response time (P50, P95, P99)
- Embedding generation time (if using SemEmbed)

**Resources:**
- Memory usage (RSS, heap)
- CPU usage (per worker, total)
- Network bandwidth (in/out)
- NATS pending messages (backlog indicator)

### Prometheus Integration

```json
{
  "services": {
    "metrics": {
      "enabled": true,
      "config": {
        "port": 9090,
        "path": "/metrics",
        "include_go_metrics": true
      }
    }
  }
}
```

**Key Prometheus queries:**

```promql
# Message processing rate
rate(semstreams_messages_processed_total[5m])

# P95 query latency
histogram_quantile(0.95, rate(semstreams_query_duration_seconds_bucket[5m]))

# Cache hit rate
rate(semstreams_cache_hits_total[5m]) / rate(semstreams_cache_requests_total[5m])

# Memory usage
go_memstats_alloc_bytes / 1024 / 1024  # MB
```

---

## Performance Testing

### Load Testing Strategy

1. **Baseline**: Measure single-instance performance with default config
2. **Tune**: Adjust workers, queue size, batch sizes
3. **Scale**: Add instances, measure linear scaling
4. **Stress**: Push to failure point, identify bottlenecks

### E2E Performance Test

```bash
task e2e:rules-performance
```

This test validates:
- Throughput target (100 msg/sec)
- Latency P95 (<50ms)
- Stability under load (1000 messages)

### Benchmarking Tools

```bash
# Native Go benchmarks
go test -bench=. -benchmem ./...

# Integration benchmarks
INTEGRATION_TESTS=1 go test -bench=. ./processor/graph
```

---

## Common Performance Issues

### Issue 1: High Latency on Queries

**Symptoms**: Query response time >100ms consistently

**Diagnosis:**
- Check cache hit rate (should be >70% for read-heavy workloads)
- Check index coverage (missing indexes force full scans)
- Check result set size (large results = high serialization cost)

**Fix:**
```json
{
  "querier": {
    "cache_enabled": true,
    "cache_ttl": "30m",        // Increase TTL
    "max_cache_size": 10000,   // Increase cache size
    "result_limit": 500        // Cap result size
  }
}
```

### Issue 2: Growing NATS Pending Queue

**Symptoms**: NATS pending messages increasing over time

**Diagnosis:**
- Insufficient workers (CPU bottleneck)
- Slow external services (SemEmbed)
- Insufficient queue size (backpressure)

**Fix:**
```json
{
  "graph": {
    "config": {
      "workers": 16,          // Increase workers
      "queue_size": 10000     // Increase buffer
    }
  }
}
```

### Issue 3: High Memory Usage

**Symptoms**: Memory usage >10GB with <1M entities

**Diagnosis:**
- Too many indexes enabled
- Embeddings enabled for high-volume data
- Large cache sizes

**Fix:**
```json
{
  "indexer": {
    "indexes": {
      "spatial": false,       // Disable unused indexes
      "temporal": false
    },
    "embedding": {
      "enabled": false        // Disable if not needed
    }
  },
  "querier": {
    "max_cache_size": 1000    // Reduce cache size
  }
}
```

---

## Next Steps

- **[PathRAG vs GraphRAG Decisions](01-pathrag-graphrag-decisions.md)** - Choose the right configuration
- **[Architecture Deep Dive](03-architecture-deep-dive.md)** - Understand system internals
- **[Production Patterns](04-production-patterns.md)** - Deployment best practices
- **[Algorithm Reference](05-algorithm-reference.md)** - Native algorithm details

---

**Performance is a feature** - Start with reasonable defaults, measure, tune based on your workload.
