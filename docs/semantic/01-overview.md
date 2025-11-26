# Semantic Capabilities Overview

**Native vs Enhanced semantic processing**

---

## Two Levels of Semantic

### Native (Always Available)

Go algorithms built into the runtime.

**Algorithms:**

- TF-IDF (term frequency-inverse document frequency)
- BM25 (best matching 25)
- Cosine similarity

**Works on:** Laptop, server, Raspberry Pi - anywhere Go runs

**Quality:** Good for basic similarity search

**Dependencies:** None (pure Go)

See [Algorithm Reference](../advanced/05-algorithm-reference.md) for details.

---

### Enhanced (Optional)

External services for higher quality.

**SemEmbed:** Transformer-based vector embeddings

- Model: all-MiniLM-L6-v2 (384 dimensions)
- Much better semantic similarity than native
- ~2GB RAM, GPU optional
- OpenAI API-compatible

**SemSummarize:** LLM-powered summarization

- Richer entity descriptions
- Any OpenAI API-compatible service (local or cloud)
- Examples: Ollama, LM Studio, OpenAI API

**Works on:** Laptop/server (sufficient compute), not Raspberry Pi

---

## When to Use Each

| Scenario | Recommendation |
|----------|----------------|
| Edge deployment (Raspberry Pi) | Native only |
| Development/testing | Native only (faster iteration) |
| Production (laptop/server) | Enhanced (better quality) |
| High-volume telemetry | Native (lower resource overhead) |
| Document search | Enhanced (significantly better results) |

---

## Configuration

### Native Only (Default)

```json
{
  "graph": {
    "config": {
      "indexer": {
        "embedding": {
          "enabled": false
        }
      }
    }
  }
}
```

**Memory:** ~200 bytes/entity
**Throughput:** ~10,000 msg/sec

---

### Enhanced (SemEmbed + SemSummarize)

```json
{
  "graph": {
    "config": {
      "indexer": {
        "embedding": {
          "enabled": true,
          "service_url": "http://localhost:8081",
          "batch_size": 32,
          "fields": ["title", "description", "content"]
        }
      },
      "summarization": {
        "enabled": true,
        "service_url": "http://localhost:8082",
        "model": "llama2"
      }
    }
  }
}
```

**Memory:** ~2KB/entity
**Throughput:** ~1,000 msg/sec (limited by embedding service)

---

## Starting Optional Services

### SemEmbed

```bash
docker run -d -p 8081:8081 ghcr.io/c360/semembed:latest
```

See [SemEmbed Guide](02-semembed.md)

### SemSummarize

```bash
# Example: Ollama
docker run -d -p 8082:11434 ollama/ollama
ollama run llama2
```

See [SemSummarize Guide](03-semsummarize.md)

---

## Performance Impact

### Native

```
✅ Fast ingestion (~10K msg/sec)
✅ Low memory (~200 bytes/entity)
✅ Works everywhere
⚠️ Basic similarity quality
```

### Enhanced

```
✅ High-quality similarity
✅ Rich summarization
⚠️ Slower ingestion (~1K msg/sec)
⚠️ Higher memory (~2KB/entity)
⚠️ Requires external services
```

---

## Next Steps

- **[SemEmbed](02-semembed.md)** - Enhanced vector embeddings
- **[SemSummarize](03-semsummarize.md)** - LLM-powered summarization
- **[Decision Guide](04-decision-guide.md)** - Choose native vs enhanced
- **[Algorithm Reference](../advanced/05-algorithm-reference.md)** - Native algorithms explained

---

**Both work** - Native provides baseline functionality everywhere. Enhanced improves quality when you have the resources.
