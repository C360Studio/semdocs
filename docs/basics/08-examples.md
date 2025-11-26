# Running Examples

Docker Compose examples to get SemStreams running in minutes

---

## Quick Start

The fastest way to learn SemStreams is through hands-on examples. All examples follow the **minimal config** principle - tiny configuration files with sensible defaults.

**Location:** [`examples/` directory](../../examples/)

---

## Available Examples

### [Quickstart](../../examples/quickstart/)

**Use Case:** Get SemStreams running in 30 seconds

**What You Get:**
- NATS message broker with JetStream
- SemStreams with default components
- Minimal 20-line configuration

**When to Use:** Learning basics, testing locally, proof of concept

```bash
cd examples/quickstart/
docker compose up -d
curl http://localhost:8080/health
```

---

### [With UI](../../examples/with-ui/)

**Use Case:** Visual flow development with drag-and-drop

**What You Get:**
- Everything from quickstart
- SemStreams UI (visual flow builder)
- Interactive component configuration

**When to Use:** Building flows without editing JSON, demos, development

```bash
cd examples/with-ui/
docker compose up -d
open http://localhost:3000
```

---

### [With Embeddings](../../examples/with-embeddings/)

**Use Case:** Semantic search and similarity matching

**What You Get:**
- Everything from quickstart
- SemEmbed (neural embedding service)
- OpenAI-compatible embeddings API

**When to Use:** Semantic search, similarity detection, advanced RAG

**Note:** First startup downloads ~60MB model (30-60 seconds)

```bash
cd examples/with-embeddings/
docker compose up -d
curl http://localhost:8081/v1/embeddings # SemEmbed API
```

---

### [Production](../../examples/production/)

**Use Case:** Production edge deployments

**What You Get:**
- Hardened containers (non-root, read-only)
- Resource limits (edge-optimized)
- Security best practices
- Network isolation
- Proper health checks

**When to Use:** Production deployments, edge devices, security-conscious environments

```bash
cd examples/production/
cp config.example.json config.json
# Edit config.json for your deployment
docker compose up -d
```

---

## Example Comparison

| Example | NATS | SemStreams | UI | Embeddings | Production-Ready | Config Lines |
|---------|------|------------|----|-----------

|------------------|--------------|
| quickstart | ✅ | ✅ | ❌ | ❌ | ❌ | 20 |
| with-ui | ✅ | ✅ | ✅ | ❌ | ❌ | 20 |
| with-embeddings | ✅ | ✅ | ❌ | ✅ | ❌ | 20 |
| production | ✅ | ✅ | ❌ | ❌ | ✅ | 20 + metadata |

---

## Common Tasks

### Checking Health

```bash
# SemStreams health
curl http://localhost:8080/health

# NATS monitoring
curl http://localhost:8222/healthz

# SemEmbed (if running)
curl http://localhost:8081/health
```

### Viewing Logs

```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f semstreams
docker compose logs -f nats
docker compose logs -f semembed
```

### Stopping Services

```bash
# Stop and remove containers
docker compose down

# Also remove volumes (careful - deletes data!)
docker compose down -v
```

---

## Resource Requirements

### Quickstart (Minimal)
- **RAM:** 1GB
- **CPU:** 1 core
- **Disk:** 2GB

### With UI
- **RAM:** 1.5GB
- **CPU:** 1-2 cores
- **Disk:** 2GB

### With Embeddings
- **RAM:** 2GB
- **CPU:** 2 cores
- **Disk:** 3GB (includes model cache)

### Production
- **RAM:** 4GB (edge device spec)
- **CPU:** 2+ cores
- **Disk:** 8GB (SSD recommended)

---

## Configuration Philosophy

All examples use **minimal config with maximum defaults**:

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

**Only 20 lines** - just the essentials. Everything else uses sensible defaults.

### What You Get (Defaults)

- **Log Level:** `info` (clean, structured logging)
- **HTTP Port:** `8080` (standard web port)
- **Metrics Port:** `9090` (Prometheus)
- **Components:** None yet - add your flows to config

### What's NOT Included (Optional)

Add these by extending the examples:

- Custom flow configurations
- Persistent storage beyond NATS
- TLS/certificates
- Monitoring dashboards (Grafana, etc.)

See [Production Patterns](../advanced/08-production-patterns.md) and [Deployment Guides](../deployment/) for these features.

---

## Combining Examples

Mix and match features by combining docker-compose files:

**UI + Embeddings:**
```bash
# Copy services from both files into new docker-compose.yml
# Ensure:
# 1. Same network name
# 2. No port conflicts
# 3. Updated service dependencies
```

**Production + Everything:**
```bash
# Start with production/docker-compose.yml
# Add semembed and ui services
# Add resource limits to new services
```

---

## Troubleshooting

### Port Already in Use?

```bash
# Change port mapping in docker-compose.yml
ports:
  - "8081:8080"  # Host:8081 -> Container:8080
```

### NATS Connection Failed?

```bash
# Check NATS is healthy
docker compose ps
docker compose logs nats

# Verify network connectivity
docker compose exec semstreams ping nats
```

### Need Verbose Logs?

```yaml
# In docker-compose.yml
environment:
  SEMSTREAMS_LOG_LEVEL: debug  # More detailed logging
```

### Container Won't Start?

```bash
# Check configuration syntax
docker compose config

# View detailed errors
docker compose logs

# Verify resource availability
docker stats
```

---

## Next Steps

**After Running Examples:**

1. **Understand Configuration** → [Components](02-components.md)
2. **Build Your First Flow** → [First Flow](04-first-flow.md)
3. **Message Routing** → [Routing](03-routing.md)
4. **Production Deployment** → [Deployment Guides](../deployment/)

**Advanced Topics:**

- [Semantic Features](../semantic/) - Embeddings, summarization
- [Edge Deployment](../edge/) - Federation, offline operation
- [Graph Operations](../graph/) - Entities, relationships, queries

---

**Examples Directory:** [`examples/`](../../examples/)

All examples are ready to run with `docker compose up -d` - pick the one that matches your use case and start exploring!
