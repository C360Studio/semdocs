# SemEmbed Service

**Enhanced vector embeddings for high-quality semantic search**

---

## What is SemEmbed?

Transformer-based embedding service that converts text to vector representations for semantic similarity.

**Model:** all-MiniLM-L6-v2 (384 dimensions)
**API:** OpenAI-compatible
**Performance:** ~1000 embeddings/sec (CPU), ~5000/sec (GPU)

---

## Starting SemEmbed

### Docker

```bash
docker run -d \
  --name semembed \
  -p 8081:8081 \
  ghcr.io/c360/semembed:latest
```

### With GPU

```bash
docker run -d \
  --name semembed \
  --gpus all \
  -p 8081:8081 \
  ghcr.io/c360/semembed:latest-gpu
```

---

## Configuration

```json
{
  "graph": {
    "config": {
      "indexer": {
        "embedding": {
          "enabled": true,
          "service_url": "http://localhost:8081",
          "batch_size": 32,
          "timeout": "5s",
          "fields": ["title", "description", "content"]
        }
      }
    }
  }
}
```

**Fields:**
- **service_url:** SemEmbed endpoint
- **batch_size:** Number of texts to embed per request (16-64)
- **timeout:** Request timeout
- **fields:** Entity fields to embed (concatenated)

---

## Usage

### Automatic (via Graph Processor)

When enabled, graph processor automatically:
1. Extracts text from configured fields
2. Sends to SemEmbed
3. Stores vector in entity

**No manual code required.**

### Manual (HTTP API)

```bash
curl -X POST http://localhost:8081/embed \
  -H "Content-Type: application/json" \
  -d '{
    "input": ["emergency battery situation", "low power alert"],
    "model": "all-MiniLM-L6-v2"
  }'
```

**Response:**

```json
{
  "data": [
    {"embedding": [0.123, -0.456, ...]},
    {"embedding": [0.111, -0.442, ...]}
  ]
}
```

---

## Semantic Search

Once embeddings are enabled, query semantically:

```bash
curl -X POST http://localhost:8080/search/semantic \
  -d '{
    "query": "emergency battery situation",
    "limit": 10,
    "threshold": 0.7
  }'
```

**Response:**

```json
{
  "results": [
    {
      "entity_id": "UAV-002",
      "score": 0.89,
      "properties": {"battery": 15.4, "status": "critical"}
    },
    {
      "entity_id": "alert-battery-001",
      "score": 0.82,
      "properties": {...}
    }
  ]
}
```

---

## Performance Tuning

### Batch Size

```json
{"batch_size": 32}  // Good for CPU
{"batch_size": 64}  // Better for GPU
```

Larger batches = higher throughput, higher latency

### Field Selection

```json
{"fields": ["title", "description"]}  // Essential fields only
```

Fewer fields = faster embedding, less complete semantic index

### Async Processing

SemEmbed calls are async by default - don't block entity indexing.

---

## Monitoring

```bash
# SemEmbed health
curl http://localhost:8081/health

# Embedding metrics
curl http://localhost:9090/metrics | grep embedding
```

**Key metrics:**
- `semstreams_embeddings_total` - Total embeddings generated
- `semstreams_embedding_duration_seconds` - Embedding latency
- `semstreams_embedding_errors_total` - Embedding failures

---

## Troubleshooting

### "Connection refused"

**Cause:** SemEmbed not running

**Fix:**
```bash
docker ps | grep semembed
docker start semembed
```

### High latency

**Cause:** CPU bottleneck

**Fix:** Use GPU or increase batch size
```json
{"batch_size": 64}
```

### Out of memory

**Cause:** Too many concurrent embeddings

**Fix:** Reduce worker count or batch size
```json
{"workers": 4, "batch_size": 16}
```

---

## Next Steps

- **Quick Start**: [With Embeddings Example](../../examples/with-embeddings/) - Try SemEmbed with Docker Compose
- **[SemSummarize](03-semsummarize.md)** - LLM-powered summarization
- **[Decision Guide](04-decision-guide.md)** - When to use enhanced semantic
- **[Performance Tuning](../advanced/02-performance-tuning.md)** - Optimize embedding throughput

---

**SemEmbed upgrades semantic quality** - Use when you need better similarity search than native algorithms provide.
