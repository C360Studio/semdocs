# Optional External Services

SemStreams supports optional containerized microservices that enhance functionality. The platform gracefully degrades when these services are unavailable.

## Available Services

### Embedding Service (SemEmbed)

**Optional**: Semantic search works by default with BM25 (pure Go, no service needed)

**Provides**: Neural text embeddings via SemEmbed (lightweight Rust-based service using fastembed-rs)

**What is SemEmbed?**

- Lightweight embedding service built on fastembed-rs
- OpenAI-compatible API (/v1/embeddings endpoint)
- Optimized for performance with Rust and ONNX runtime
- Supports popular sentence-transformers models (BAAI/bge-*, sentence-transformers/*)
- Native arm64 + amd64 support
- Smaller memory footprint than alternatives (~512MB-1GB vs 2GB+)

**Default Behavior**:

- Semantic search is **enabled by default** using BM25 (pure Go lexical search)
- No external service required
- Performance: ~1.4μs per text
- Best for: exact keyword matching, offline operation

**Upgrade to Neural Embeddings (SemEmbed)**:

1. **Start SemEmbed service** (one command):

   ```bash
   task services:start:embedding
   # or
   docker compose -f docker-compose.services.yml --profile embedding up -d
   ```

2. **Configure SemStreams** to use SemEmbed:

   ```json
   {
     "indexmanager": {
       "embedding": {
         "enabled": true,
         "provider": "http",
         "http_endpoint": "http://localhost:8081",
         "http_model": "BAAI/bge-small-en-v1.5"
       }
     }
   }
   ```

3. **That's it!** System automatically:
   - Tests SemEmbed connectivity on startup
   - Uses neural embeddings if SemEmbed available
   - Falls back to BM25 if SemEmbed unavailable
   - No manual intervention required

**Automatic Fallback**: If SemEmbed service is unavailable, automatically falls back to BM25 (no disruption)

**Alternative**: TEI (Text Embeddings Inference) from Hugging Face can be used instead - see Dockerfile.tei for setup

**Explicit BM25 Only** (if you want to skip HTTP attempt):

```json
{
  "indexmanager": {
    "embedding": {
      "enabled": true,
      "provider": "bm25"
    }
  }
}
```

**Model Selection** (edit `docker-compose.services.yml`):

```yaml
environment:
  # Fast, good quality (default)
  - SEMEMBED_MODEL=BAAI/bge-small-en-v1.5  # 384 dims

  # Higher quality
  # - SEMEMBED_MODEL=BAAI/bge-base-en-v1.5  # 768 dims

  # Best English quality
  # - SEMEMBED_MODEL=BAAI/bge-large-en-v1.5  # 1024 dims

  # Alternative: sentence-transformers models
  # - SEMEMBED_MODEL=sentence-transformers/all-MiniLM-L6-v2  # 384 dims
  # - SEMEMBED_MODEL=sentence-transformers/all-mpnet-base-v2  # 768 dims
```

**Performance Comparison**:

- **BM25** (default): Fast (~1.4μs), works offline, exact keyword matching
- **SemEmbed Neural**: Better semantic understanding, synonyms, concepts (~50-100ms per batch)

**Graceful Degradation**:

- **HTTP provider**: Automatically falls back to BM25 if SemEmbed is unavailable (seamless)
- **BM25 provider**: Always works (no external dependencies)
- All other indexing (predicate, spatial, temporal, etc.) continues normally regardless of embedding provider

### Summarization Service (SemSummarize)

**Optional**: Community summarization works by default with statistical methods (no service needed)

**Provides**: LLM-based natural language summaries via SemSummarize (lightweight Rust-based service using Candle + T5/BART)

**What is SemSummarize?**

- Lightweight summarization service built on Candle (Rust ML framework)
- Supports T5 and BART models from HuggingFace Hub
- Native arm64 + amd64 support with CPU-optimized inference
- Smaller memory footprint (~512MB-1GB)
- Used for GraphRAG community summarization

**Default Behavior**:

- Community summarization is **enabled by default** using statistical methods
- No external service required
- Generates summaries from entity types, properties, and keywords
- Best for: offline operation, deterministic results

**Upgrade to LLM Summarization (SemSummarize)**:

1. **Start SemSummarize service** (one command):

   ```bash
   task services:start:summarization
   # or
   docker compose -f docker-compose.services.yml --profile summarization up -d
   ```

2. **Use in code** (GraphRAG community detection):

   ```go
   // Create LLM summarizer pointing to semsummarize service
   llmSummarizer := graphclustering.NewHTTPLLMSummarizer("http://localhost:8083")

   // Summarize community (automatically falls back to statistical if service unavailable)
   community, err := llmSummarizer.SummarizeCommunity(ctx, community, entities)

   // Check which summarizer was used
   if community.Summarizer == "llm" {
       // LLM service was available and used
   } else if community.Summarizer == "statistical-fallback" {
       // Fell back to statistical summarization
   }
   ```

3. **That's it!** System automatically:
   - Tests SemSummarize connectivity
   - Uses LLM summaries if available
   - Falls back to statistical methods if unavailable
   - No errors or disruption

**Automatic Fallback**: If SemSummarize service is unavailable, automatically falls back to statistical summarization (no disruption)

**Model Selection** (edit `docker-compose.services.yml`):

```yaml
environment:
  # Fast, good quality (default)
  - SEMSUMMARIZE_MODEL=google/flan-t5-small  # 77M params

  # Higher quality
  # - SEMSUMMARIZE_MODEL=google/flan-t5-base  # 250M params

  # Alternative: BART models for narrative style
  # - SEMSUMMARIZE_MODEL=facebook/bart-base  # 140M params
```

**Performance Comparison**:

- **Statistical** (default): Fast (<1ms), deterministic, keyword-based
- **SemSummarize LLM**: Natural language, contextual understanding (~500ms-2s per summary)

**Use Cases**:

- GraphRAG community summarization (better understanding of entity clusters)
- Generating human-readable descriptions of graph neighborhoods
- Creating narrative summaries for monitoring dashboards

**Graceful Degradation**:

- **HTTP LLM**: Automatically falls back to statistical if SemSummarize is unavailable (seamless)
- **Statistical**: Always works (no external dependencies)
- GraphRAG search works normally regardless of summarization provider

### Certificate Authority (step-ca) - Tiered Security Model

**Status**: 📋 **Planned** - Infrastructure defined, implementation tracked in `semstreams-83ff`

SemStreams implements a **three-tier security model** that balances simplicity with enterprise PKI needs:

#### Tier 1: None (Default) - ✅ Current Implementation

- **Use Case**: Development, local testing, trusted internal networks
- **Configuration**: No security configuration required
- **Security**: Plaintext communication (acceptable for dev/trusted networks)
- **Complexity**: Zero configuration

#### Tier 2: Manual TLS + mTLS - 🔨 Planned (5 days)

- **Use Case**: 1-2 locations, existing PKI infrastructure, small deployments
- **Security**: TLS encryption + optional mutual authentication (zero-trust)
- **Complexity**: Medium (manual certificate generation and renewal)
- **Certificate Lifecycle**: Manual renewal (typically 90 days)

**Configuration Example** (Manual TLS):

```json
{
  "security": {
    "tls": {
      "server": {
        "enabled": true,
        "cert_file": "/path/to/server.crt",
        "key_file": "/path/to/server.key",
        "min_version": "1.2"
      },
      "client": {
        "ca_files": ["/path/to/ca.crt"]
      }
    }
  }
}
```

**Configuration Example** (Manual TLS + mTLS):

```json
{
  "security": {
    "tls": {
      "server": {
        "enabled": true,
        "cert_file": "/path/to/server.crt",
        "key_file": "/path/to/server.key",
        "mtls": {
          "enabled": true,
          "client_ca_files": ["/path/to/client-ca.crt"],
          "require_client_cert": true,
          "allowed_client_cns": ["location-b", "location-c"]
        }
      },
      "client": {
        "ca_files": ["/path/to/server-ca.crt"],
        "mtls": {
          "enabled": true,
          "cert_file": "/path/to/client.crt",
          "key_file": "/path/to/client.key"
        }
      }
    }
  }
}
```

**Benefits**:

- Zero-trust security for federated WebSocket connections
- Client certificate validation (only authorized clients can connect)
- CN-based access control (whitelist specific locations)
- Backward compatible with existing manual TLS configs

#### Tier 3: Automated PKI (step-ca + ACME) 

- **Use Case**: 3+ locations, enterprise scale, limited PKI expertise
- **Security**: Full PKI automation with mTLS and federation support
- **Complexity**: High initial setup, low ongoing maintenance
- **Certificate Lifecycle**: Automated 24-hour renewal (no manual intervention)

**Configuration Example** (ACME):

```json
{
  "security": {
    "tls": {
      "server": {
        "enabled": true,
        "mode": "acme",
        "acme": {
          "enabled": true,
          "directory_url": "https://step-ca:9000/acme/acme/directory",
          "email": "admin@example.com",
          "domains": ["semstreams-location-a.local"],
          "challenge_type": "http-01",
          "renew_before": "8h",
          "storage_path": "/data/acme"
        },
        "mtls": {
          "enabled": true,
          "client_ca_files": ["/certs/federation-root-ca.crt"],
          "require_client_cert": true,
          "allowed_client_cns": ["location-b.local", "location-c.local"]
        }
      }
    }
  }
}
```

**Federation Architecture**:

```shell
Corporate Root CA (offline)
    │
    ├─── Location A: step-ca (intermediate) → SemStreams A
    ├─── Location B: step-ca (intermediate) → SemStreams B
    └─── Location C: step-ca (intermediate) → SemStreams C

All locations automatically trust each other's certificates
```

**Benefits**:

- Set-and-forget certificate renewal (automated every 16 hours)
- Short-lived certificates (24h) eliminate revocation complexity
- Built-in federation support for multi-location deployments
- Graceful fallback (ACME failure → manual certificates)
- Zero manual intervention for certificate lifecycle

**Start step-ca** (when Tier 3 is implemented):

```bash
task services:start:tls
# or
docker compose -f docker-compose.services.yml --profile tls up -d
```

**Tier Comparison**:

| Aspect | Tier 1: None | Tier 2: Manual | Tier 3: ACME |
|--------|-------------|---------------|--------------|
| **Setup Time** | 0 hours | 2-4 hours | 8-16 hours |
| **Annual Maintenance** | 0 hours | 8-16 hours | ~2 hours (monitoring) |
| **Multi-Location** | N/A | High burden | Low (automated) |
| **mTLS Support** | No | Yes | Yes |
| **Federation** | No | Manual trust | Automatic trust |
| **Best For** | Dev/testing | 1-2 locations | 3+ locations |

**Recommendation**:

- **1-2 locations**: Use Tier 2 (manual certificates with optional mTLS)
- **3+ locations**: Use Tier 3 (ACME automation provides ROI within first year)
- **Existing PKI**: Use Tier 2 with corporate CA-issued certificates

**Graceful Degradation**:

- Tier 2: TLS components work with manually-managed certificates (standard OpenSSL workflow)
- Tier 3: ACME failure automatically falls back to manual certificates if configured
- All tiers: System operates without TLS if security not configured (Tier 1 default)

### Observability Stack (Prometheus + Grafana)

**Optional**: Provides local metrics collection and visualization

**Provides**:

- Prometheus metrics scraping and storage
- Pre-configured Grafana dashboards
- Real-time monitoring of SemStreams components

**Components**:

- **Prometheus**: Metrics collection server (port 9090)
- **Grafana**: Visualization and dashboards (port 3000)

**Pre-configured Dashboards**:

1. **SemStreams Overview**: Service health, throughput, error rates, latency
2. **Cache Performance**: L1/L2/L3 hit rates, sizes, eviction rates
3. **IndexManager Metrics**: Index sizes, query latency, backlog, deduplication
4. **Graph Processor**: Entity/edge operations, latency, worker pool status

**Configuration**:

```json
{
  "metrics": {
    "enabled": true,
    "port": 8080,
    "path": "/metrics"
  }
}
```

**Start**:

```bash
task services:start:observability
# or
docker compose -f docker-compose.services.yml --profile observability up -d
```

**Access**:

- Prometheus: http://localhost:9090
- Grafana: http://localhost:3000 (default credentials: admin/admin)
- Change password on first login via: `GRAFANA_PASSWORD=<your-password>`

**Custom Config**:

- Edit `configs/prometheus/prometheus.yml` to add scrape targets
- Edit dashboards in `configs/grafana/dashboards/*.json`
- Changes take effect on restart

**Graceful Degradation**: Metrics endpoints work independently. Prometheus/Grafana are only needed for centralized monitoring and dashboards.

## Quick Reference

### Start Services

```bash
# Start embedding service only
task services:start:embedding

# Start TLS certificate authority only
task services:start:tls

# Start observability stack only
task services:start:observability

# Start all optional services
task services:start:all
```

### Manage Services

```bash
# Check status
task services:status

# View logs
task services:logs

# Stop all optional services
task services:stop
```

### Direct Docker Compose Usage

```bash
# Start specific profile
docker compose -f docker-compose.services.yml --profile embedding up -d

# Start multiple profiles
docker compose -f docker-compose.services.yml --profile embedding --profile observability up -d

# Stop all
docker compose -f docker-compose.services.yml --profile all down
```

## Architecture: Smart Defaults with Automatic Fallback

SemStreams follows the **Zero-Config Semantic Search** pattern:

1. **Default Enabled**: Semantic search is enabled by default using BM25 (pure Go)
2. **Automatic Fallback**: HTTP provider automatically falls back to BM25 if service unavailable
3. **No Disruption**: Users always get semantic search functionality (lexical or neural)
4. **Opt-Out**: Explicitly set `provider: "disabled"` to disable entirely

### Provider Resolution Flow

```text
┌─────────────────────────────────────┐
│ IndexManager.initializeSemanticSearch│
└────────────┬────────────────────────┘
             │
        ┌────▼────┐
        │Provider?│
        └────┬────┘
             │
   ┌─────────┼──────────┐
   │         │          │
 Empty     "bm25"     "http"
   │         │          │
   └──┬──────┘          │
      │                 │
 Use BM25          Test HTTP
      │                 │
      │         ┌───────┴───────┐
      │         │               │
      │      Success         Timeout
      │         │               │
      │     Use HTTP      Fallback to BM25
      │         │               │
      └─────────┴───────────────┘
                │
        Semantic Search Ready
```

### Example: Automatic SemEmbed → BM25 Fallback

**What happens when you start SemStreams without SemEmbed running:**

```go
// indexmanager/semantic.go:83-103
// System tests SemEmbed connectivity on startup
_, testErr := m.embedder.Generate(testCtx, []string{"connectivity test"})
if testErr != nil {
    m.logger.Warn("HTTP embedding service unavailable - falling back to BM25",
        "endpoint", "http://localhost:8081",  // SemEmbed endpoint
        "error", testErr.Error(),
        "fallback", "bm25",
        "hint", "Start embedding service with: task services:start:embedding")

    // Automatic fallback to BM25 instead of disabling
    m.embedder = embedding.NewBM25Embedder(embedding.BM25Config{
        Dimensions: 384,
        K1:         1.5,
        B:          0.75,
    })
    m.logger.Info("Fallback to BM25 embedder successful",
        "dimensions", m.embedder.Dimensions(),
        "model", "bm25-lexical")
}
```

**Log output example:**

```shell
WARN HTTP embedding service unavailable - falling back to BM25
  endpoint=http://localhost:8081
  error=connection refused
  fallback=bm25
  hint="Start embedding service with: task services:start:embedding"
INFO Fallback to BM25 embedder successful
  dimensions=384
  model=bm25-lexical
```

**Result**: Semantic search always works - either with neural embeddings (SemEmbed) or lexical search (BM25)

## Adding New Optional Services

To add a new optional service:

1. **Add to `docker-compose.services.yml`**:

   ```yaml
   services:
     myservice:
       profiles: ["myservice", "all"]
       image: myservice:latest
       # ... configuration
   ```

2. **Add Taskfile commands**:

   ```yaml
   services:start:myservice:
     desc: Start my service
     cmds:
       - docker compose -f docker-compose.services.yml --profile myservice up -d
   ```

3. **Implement graceful handling** in component:
   - Test connectivity during Initialize()
   - Log warning if unavailable
   - Set feature flag to disabled
   - Return clear errors when disabled feature is used

4. **Update documentation** with configuration and degradation behavior

## Best Practices

1. **Always test without services**: Verify your component works in degraded mode
2. **Clear error messages**: Tell users exactly what's missing and how to enable it
3. **No cascading failures**: One missing service should not break other features
4. **Document gracefully**: Make it obvious what's optional vs required
5. **Monitor in production**: Log when running in degraded mode for visibility

## Troubleshooting

### SemEmbed service won't start

```bash
# Check if port is available
lsof -i :8081

# View SemEmbed logs
docker compose -f docker-compose.services.yml logs semembed

# Restart with fresh state
docker compose -f docker-compose.services.yml down -v
docker compose -f docker-compose.services.yml --profile embedding up -d

# Watch SemEmbed download model (first start only)
docker compose -f docker-compose.services.yml logs -f semembed
```

**Common issues:**

- **Port conflict**: Another service using port 8081
- **Memory**: SemEmbed needs ~512MB-1GB RAM depending on model
- **First start**: Model download can take 1-2 minutes (one-time, cached to volume)

### Semantic search not working

1. **Check service is running**:

   ```bash
   task services:status
   # or
   docker compose -f docker-compose.services.yml ps
   ```

2. **Check IndexManager logs** for connectivity warnings:

   ```bash
   # Look for fallback messages
   grep "fallback to BM25" logs/semstreams.log
   ```

3. **Verify configuration**:

   ```json
   {
     "embedding": {
       "enabled": true,
       "provider": "http",
       "http_endpoint": "http://localhost:8081",
       "http_model": "BAAI/bge-small-en-v1.5"
     }
   }
   ```

4. **Test SemEmbed manually**:
   ```bash
   # Health check
   curl http://localhost:8081/health

   # Test embedding generation
   curl -X POST http://localhost:8081/v1/embeddings \
     -H "Content-Type: application/json" \
     -d '{"input": "test query", "model": "BAAI/bge-small-en-v1.5"}'
   ```

5. **Expected response** (healthy SemEmbed):
   ```json
   {
     "object": "list",
     "data": [
       {
         "object": "embedding",
         "embedding": [0.123, -0.456, ...],  # 384 floats
         "index": 0
       }
     ],
     "model": "BAAI/bge-small-en-v1.5",
     "usage": {"prompt_tokens": 2, "total_tokens": 2}
   }
   ```

### step-ca certificate issues

```bash
# View step-ca logs
docker compose -f docker-compose.services.yml logs step-ca

# Check CA health
docker exec semstreams-step-ca step ca health
```

## Production Considerations

For production deployments:

1. **Use container orchestration**: Kubernetes, Docker Swarm, or Nomad
2. **Health checks**: Configure liveness/readiness probes
3. **Resource limits**: Set memory/CPU limits (see docker-compose.services.yml)
4. **High availability**: Run multiple replicas of optional services
5. **Monitoring**: Track when running in degraded mode
6. **Graceful startup**: Use init containers or startup dependencies in K8s

## See Also

- [TLS Configuration](./TLS_SETUP.md) - Configuring TLS with or without step-ca
- [Semantic Search](../processor/graph/indexmanager/README.md) - IndexManager embedding configuration
- [Docker Compose Reference](https://docs.docker.com/compose/profiles/) - Profile documentation
