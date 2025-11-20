# SemStreams Docker Compose Examples

Ready-to-run Docker Compose configurations for common SemStreams deployment scenarios.

## Quick Navigation

| Example | What's Included | Best For | Config File |
|---------|----------------|----------|-------------|
| [quickstart](./quickstart/) | NATS + SemStreams | Learning, testing | ✅ Minimal (20 lines) |
| [with-ui](./with-ui/) | + Visual flow builder | Development, demos | ✅ Minimal (20 lines) |
| [with-embeddings](./with-embeddings/) | + Semantic search | Advanced search | ✅ Minimal (20 lines) |
| [production](./production/) | Hardened + resource limits | Production edge | ✅ Minimal + metadata |

## Philosophy: Minimal Config, Maximum Defaults

All examples follow the **"minimal config"** principle:

- ✅ **Tiny config files** (20 lines for most examples)
- ✅ **Only essential metadata** (platform ID, NATS URL)
- ✅ **Maximum defaults** (all services use sensible defaults)
- ✅ **Comments show what's configurable** (but you rarely need to change)
- ✅ **Empty components section** (add your flows as needed)

## Getting Started

### 1. Choose Your Example

**New to SemStreams?** Start here:
```bash
cd quickstart/
docker compose up -d
```

**Want visual flow building?**
```bash
cd with-ui/
docker compose up -d
```

**Need semantic search?**
```bash
cd with-embeddings/
docker compose up -d
```

**Deploying to production?**
```bash
cd production/
cp config.example.json config.json
# Edit config.json for your deployment
docker compose up -d
```

### 2. Verify It's Working

```bash
# Check all services are healthy
docker compose ps

# Test the API
curl http://localhost:8080/health

# View logs
docker compose logs -f
```

### 3. Stop When Done

```bash
docker compose down

# Or remove volumes too
docker compose down -v
```

## Example Details

### Quickstart
**Use case**: Get SemStreams running in 30 seconds

**What you get**:
- NATS message broker
- SemStreams with default components

**Minimal config** (20 lines) - just platform metadata and NATS URL.

[→ Read the quickstart guide](./quickstart/README.md)

---

### With UI
**Use case**: Visual flow development with drag-and-drop

**What you get**:
- Everything from quickstart
- SemStreams UI (visual flow builder)

**Same minimal config** - UI auto-discovers backend capabilities.

[→ Read the UI guide](./with-ui/README.md)

---

### With Embeddings
**Use case**: Semantic search and similarity matching

**What you get**:
- Everything from quickstart
- SemEmbed (neural embedding service)
- OpenAI-compatible embeddings API

**Same minimal config** + embedding env vars for integration.

**Note**: First startup downloads ~60MB model (30-60 seconds).

[→ Read the embeddings guide](./with-embeddings/README.md)

---

### Production
**Use case**: Production edge deployments

**What you get**:
- Hardened containers
- Resource limits (edge-optimized)
- Security best practices
- Network isolation
- Proper health checks

**Minimal config needed** - `config.json` with platform metadata only.

[→ Read the production guide](./production/README.md)

---

## Common Patterns

### Combining Examples

Mix and match features by combining compose files:

**UI + Embeddings:**
```bash
# Copy services from both files into new docker-compose.yml
# Make sure to:
# 1. Use same network name
# 2. No port conflicts
# 3. Update service dependencies
```

**Production + Everything:**
```bash
# Start with production/docker-compose.yml
# Add semembed and ui services
# Add proper resource limits to new services
```

### Port Customization

All examples use standard ports. To change:

```yaml
ports:
  - "8081:8080"  # Host:8081 -> Container:8080
```

Update any `BACKEND_HOST` or endpoint URLs accordingly.

### Volume Management

```bash
# List volumes
docker volume ls

# Inspect volume
docker volume inspect quickstart_nats-data

# Backup volume
docker run --rm -v quickstart_nats-data:/data -v $(pwd):/backup alpine tar czf /backup/backup.tar.gz /data

# Remove volumes (careful!)
docker compose down -v
```

## Default Ports

All examples use these standard ports:

| Port | Service | Purpose |
|------|---------|---------|
| 4222 | NATS | Message broker client |
| 8080 | SemStreams | HTTP API |
| 9090 | SemStreams | Prometheus metrics |
| 8081 | SemEmbed | Embeddings API (if enabled) |
| 3000 | UI | Web interface (if enabled) |

## Resource Requirements

### Minimal (Quickstart)
- **RAM**: 1GB
- **CPU**: 1 core
- **Disk**: 2GB

### With UI
- **RAM**: 1.5GB
- **CPU**: 1-2 cores
- **Disk**: 2GB

### With Embeddings
- **RAM**: 2GB
- **CPU**: 2 cores
- **Disk**: 3GB (includes model cache)

### Production
- **RAM**: 4GB (edge device spec)
- **CPU**: 2+ cores
- **Disk**: 8GB (SSD recommended for NATS)

## Configuration Philosophy

### Minimal Config Files Provided

All examples include **minimal 20-line configs**:

```json
{
  "platform": { "org": "quickstart", "id": "demo-001" },
  "nats": { "urls": ["nats://nats:4222"] },
  "services": {
    "component-manager": { "enabled": true },
    "metrics": { "enabled": true, "config": { "port": 9090 } }
  },
  "components": {}  // Empty - add your flows!
}
```

### What This Gives You

**Required fields** (must be in config):
- Platform metadata (org, id)
- NATS connection URL
- Enabled services

**Everything else defaults**:
- Log level: `info`
- HTTP port: `8080`
- Metrics: Enabled on port 9090
- Components: None (add via config or UI)

### When to Customize

**Add flows** to `components: {}`:
```json
{
  "components": {
    "your_flow": {
      "type": "processor",
      "enabled": true,
      "config": {...}
    }
  }
}
```

**Override defaults** via environment variables:
```yaml
environment:
  SEMSTREAMS_LOG_LEVEL: debug  # Override default 'info'
  SEMSTREAMS_HTTP_PORT: 8081   # Override default 8080
```

## Troubleshooting

### Services Won't Start

```bash
# Check config syntax
docker compose config

# View errors
docker compose logs

# Check port conflicts
netstat -an | grep LISTEN
```

### Performance Issues

```bash
# Check resource usage
docker stats

# View metrics
curl http://localhost:9090/metrics
```

### Network Issues

```bash
# Test connectivity between containers
docker compose exec semstreams wget -O- http://nats:8222/healthz

# Check DNS resolution
docker compose exec semstreams nslookup nats
```

## Contributing

Found an issue or have a suggestion?
- Report issues: [GitHub Issues](https://github.com/C360Studio/semdocs/issues)
- Improve examples: Submit a PR!

## License

See main repository LICENSE file.
