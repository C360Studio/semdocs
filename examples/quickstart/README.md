# Quickstart Example

The simplest way to run SemStreams - **minimal config, maximum defaults**!

## What You Get

- **NATS**: Message broker with JetStream enabled
- **SemStreams**: Stream processing engine with default settings

## Prerequisites

- Docker & Docker Compose
- Ports 8080, 9090, and 4222 available

## Quick Start

```bash
# Config file is already provided (config.minimal.json)
# Just start everything
docker compose up -d

# Check health
curl http://localhost:8080/health

# View logs
docker compose logs -f semstreams

# Stop
docker compose down
```

## Endpoints

| Service | URL | Purpose |
|---------|-----|---------|
| API | <http://localhost:8080> | Main HTTP API |
| Health | <http://localhost:8080/health> | Health check |
| Metrics | <http://localhost:9090/metrics> | Prometheus metrics |
| NATS | nats://localhost:4222 | Message broker |

## Configuration

This example uses **minimal config with maximum defaults**:

### The Config File (`config.minimal.json`)

Only 20 lines - just the essentials:

```json
{
  "platform": { "org": "quickstart", "id": "demo-001" },
  "nats": { "urls": ["nats://nats:4222"] },
  "services": {
    "component-manager": { "enabled": true },
    "metrics": { "enabled": true, "config": { "port": 9090 } }
  },
  "components": {}  // Empty - no flows yet!
}
```

### What This Gets You (Defaults)

- **Log Level**: `info` (clean, structured logging)
- **HTTP Port**: `8080` (standard web port)
- **Metrics Port**: `9090` (from config)
- **Metrics**: Enabled (from config)
- **Components**: None yet - add your flows to the config

### What's NOT Included (Optional)

- Custom flow configurations
- Semantic search (semembed)
- Text summarization (semsummarize)
- Persistent storage
- TLS/certificates
- Monitoring dashboards

See other examples for these features!

## Next Steps

**Add a Custom Flow**:

1. Create `config.json` (see production example)
2. Mount it: `./config.json:/etc/semstreams/config.json:ro`
3. Update command: `--config /etc/semstreams/config.json`

**Add Semantic Search**:

- See `examples/with-embeddings/`

**Production Deployment**:

- See `examples/production/`

**Connect a UI**:

- See `examples/with-ui/`

## Troubleshooting

**Port already in use?**

```bash
# Change the port mapping in docker-compose.yml
ports:
  - "8081:8080"  # Host:8081 -> Container:8080
```

**NATS connection failed?**

```bash
# Check NATS is healthy
docker compose ps
docker compose logs nats
```

**Need verbose logs?**

```yaml
environment:
  SEMSTREAMS_LOG_LEVEL: debug  # More detailed logging
```
