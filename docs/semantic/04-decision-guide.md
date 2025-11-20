# Native vs Enhanced Decision Guide

**When to use native, when to use enhanced semantic**

---

## Quick Decision Matrix

| Your Scenario | Recommendation |
|---------------|----------------|
| Development/testing | Native (faster iteration) |
| Edge (Raspberry Pi) | Native (only option) |
| Laptop/server with resources | Enhanced (better quality) |
| High-volume telemetry (>5K msg/sec) | Native (lower overhead) |
| Document/knowledge search | Enhanced (significantly better) |
| IoT sensor data (numeric only) | Native (no semantic needed) |

---

## Native Semantic

**When to use:**
- Edge deployment
- Resource-constrained environments
- High-volume workloads (>5K msg/sec)
- Quick iteration during development
- Numeric/telemetry data (minimal text)

**Pros:**
- ✅ No external dependencies
- ✅ Low memory (~200 bytes/entity)
- ✅ High throughput (~10K msg/sec)
- ✅ Works everywhere (Raspberry Pi → server)
- ✅ Simple operations

**Cons:**
- ⚠️ Basic similarity quality (keyword matching)
- ⚠️ No deep semantic understanding

---

## Enhanced Semantic

**When to use:**
- Document/knowledge management
- Production systems with available resources
- Workloads requiring high-quality similarity
- Complex semantic queries
- Cross-lingual similarity (future)

**Pros:**
- ✅ High-quality semantic similarity
- ✅ Deep contextual understanding
- ✅ Better handling of synonyms/paraphrases
- ✅ Richer summarization

**Cons:**
- ⚠️ Higher memory (~2KB/entity)
- ⚠️ Lower throughput (~1K msg/sec)
- ⚠️ Requires external services
- ⚠️ More complex operations

---

## Migration Path

### Start Native

```json
{
  "indexer": {
    "embedding": {"enabled": false}
  }
}
```

### Add Enhanced Later

```json
{
  "indexer": {
    "embedding": {
      "enabled": true,
      "service_url": "http://localhost:8081"
    }
  }
}
```

**Schema evolution is built-in** - you can change your mind.

---

## Hybrid Approach

**Edge:** Native only (resource-constrained)
**Cloud:** Enhanced (resource-rich)

```
┌─────────────────────────┐
│ Edge (Raspberry Pi)     │
│ • Native semantic       │
│ • PathRAG queries       │
└──────────┬──────────────┘
           │ Selective sync
           ▼
┌─────────────────────────┐
│ Cloud (Server)          │
│ • Enhanced semantic     │
│ • Cross-edge queries    │
│ • Semantic search       │
└─────────────────────────┘
```

See [Edge Patterns](../edge/01-patterns.md) for more.

---

## Cost Comparison

### Native (per instance)

- Infrastructure: $0 (included in runtime)
- Memory: ~200 MB (1M entities)
- CPU: Moderate
- **Total: Part of base deployment**

### Enhanced (per instance)

- Infrastructure: +1 service (SemEmbed)
- Memory: +2GB (SemEmbed) + 2GB (entity embeddings)
- CPU/GPU: Higher (embedding generation)
- **Total: +$50-200/month (cloud) or +$0 (self-hosted)**

---

## Next Steps

- **[SemEmbed](02-semembed.md)** - Setup enhanced embeddings
- **[PathRAG vs GraphRAG](../advanced/01-pathrag-graphrag-decisions.md)** - Complete decision guide
- **[Algorithm Reference](../advanced/05-algorithm-reference.md)** - Understand native algorithms

---

**Both work** - Start with native, upgrade to enhanced when quality matters more than resources.
