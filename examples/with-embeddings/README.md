# With Embeddings Example

Semantic search capabilities powered by neural embeddings. **Minimal config, intelligent defaults**!

## What You Get

- **NATS**: Message broker
- **SemStreams**: Backend with semantic indexing enabled
- **SemEmbed**: Neural embedding service (fastembed-rs based)

## Prerequisites

- Docker & Docker Compose
- Ports 8080, 8081, and 9090 available
- **First run**: ~60MB model download (30-60 seconds)

## Quick Start

```bash
# Config file is provided (config.minimal.json)
# Start the stack
docker compose up -d

# First startup downloads model (~60MB, 30-60 sec) - check progress
docker compose logs -f semembed

# Once healthy, test embeddings API
curl http://localhost:8081/health
```

## Endpoints

| Service | URL | Purpose |
|---------|-----|---------|
| API | http://localhost:8080 | SemStreams HTTP API |
| Embeddings | http://localhost:8081/v1/embeddings | OpenAI-compatible embeddings |
| Models | http://localhost:8081/models | List available models |
| Health | http://localhost:8081/health | Embedding service health |

## What You Can Do

### 1. **Semantic Search**
Query messages by meaning, not just keywords:
```bash
# Traditional search: exact match
/search?q=robot

# Semantic search: finds "drone", "UAV", "autonomous vehicle"
/search?q=robot&type=semantic
```

### 2. **Similarity Clustering**
Group related messages automatically based on content similarity.

### 3. **OpenAI-Compatible API**
Use semembed with any OpenAI SDK:
```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:8081/v1", api_key="not-needed")

response = client.embeddings.create(
    model="BAAI/bge-small-en-v1.5",
    input="Your text here"
)
```

## Configuration

### Backend Config (`config.minimal.json`)

Same minimal 20-line config as quickstart:
```json
{
  "platform": { "org": "quickstart", "id": "demo-001" },
  "nats": { "urls": ["nats://nats:4222"] },
  "services": {
    "component-manager": { "enabled": true },
    "metrics": { "enabled": true, "config": { "port": 9090 } }
  },
  "components": {}
}
```

### SemEmbed Defaults

- **Model**: `BAAI/bge-small-en-v1.5`
  - Dimensions: 384
  - Speed: Fast (~5ms per text)
  - Quality: Good for most use cases
  - Size: ~60MB

- **Port**: 8081 (OpenAI-compatible)
- **Cache**: Models persisted in volume
- **API**: Compatible with OpenAI SDK

### SemStreams Integration

Automatically uses embeddings when:
1. IndexManager component enabled (auto-detected)
2. `EMBEDDING_PROVIDER=http` set (shown in compose)
3. Semantic queries requested

**Graceful Degradation**: If semembed unavailable, falls back to BM25 keyword search.

## Model Options

### Default: bge-small-en-v1.5 (Recommended)
```yaml
SEMEMBED_MODEL: BAAI/bge-small-en-v1.5
```
- Best balance of speed/quality/size
- 384 dimensions
- ~60MB download

### Higher Quality: bge-base-en-v1.5
```yaml
SEMEMBED_MODEL: BAAI/bge-base-en-v1.5
```
- Better accuracy
- 768 dimensions
- ~200MB download
- Slower (~10ms per text)

### Fastest: all-MiniLM-L6-v2
```yaml
SEMEMBED_MODEL: sentence-transformers/all-MiniLM-L6-v2
```
- Very fast (~3ms per text)
- 384 dimensions
- ~80MB download
- Slightly lower quality

## Testing Embeddings

```bash
# Check service health
curl http://localhost:8081/health

# List available models
curl http://localhost:8081/models

# Generate embeddings
curl -X POST http://localhost:8081/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{
    "input": "Hello world",
    "model": "BAAI/bge-small-en-v1.5"
  }'

# Batch embeddings
curl -X POST http://localhost:8081/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{
    "input": ["First text", "Second text", "Third text"],
    "model": "BAAI/bge-small-en-v1.5"
  }'
```

## Resource Usage

### SemEmbed
- **CPU**: 0.5-2 cores (depending on load)
- **RAM**: 512MB-1GB
- **Storage**: ~60-200MB (model cache)

### Disk Cache
Models are cached in named volume `semembed-cache`.
**First run**: Downloads model (~60MB, 30-60 sec)
**Subsequent runs**: Instant startup

## Troubleshooting

**Model download slow?**
```bash
# Check download progress
docker compose logs -f semembed

# Look for: "Downloading model BAAI/bge-small-en-v1.5..."
```

**Embedding service not healthy?**
```bash
# Check if model download succeeded
docker compose exec semembed ls -lh /home/semembed/.cache/fastembed

# View detailed logs
docker compose logs semembed | grep -i error
```

**Out of memory?**
```yaml
# Reduce model size
environment:
  SEMEMBED_MODEL: sentence-transformers/all-MiniLM-L6-v2
```

## Next Steps

**Add UI**:
- See `examples/with-ui/` to combine UI + semantic search

**Production Deployment**:
- See `examples/production/` for hardened config

**Custom Models**:
- Any fastembed-rs compatible model works
- See: https://huggingface.co/models?library=fastembed
