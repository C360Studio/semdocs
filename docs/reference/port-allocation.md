# Port Allocation - SemStreams Ecosystem

This document tracks default port assignments across the SemStreams ecosystem to prevent collisions.

## Port Registry

### Core Infrastructure

| Port | Service | Module | Purpose | Configurable |
|------|---------|--------|---------|--------------|
| 4222 | NATS Server | NATS | Core messaging | Yes |
| 8222 | NATS Monitor | NATS | HTTP monitoring | Yes |
| 6222 | NATS Cluster | NATS | Cluster routing | Yes |

### SemStreams Framework

| Port | Service | Module | Purpose | Configurable |
|------|---------|--------|---------|--------------|
| 8080 | ServiceManager | semstreams | HTTP API + Swagger UI | Yes |
| 8081 | SemEmbed | semembed | Semantic embeddings (fastembed-rs, optional) | Yes |
| 8082 | TEI Embeddings | semstreams | Text embedding service (HF TEI, optional) | Yes |
| 8083 | SemSummarize | semsummarize | Text summarization (Candle + T5/BART, optional) | Yes |

### SemMem Application

| Port | Service | Module | Purpose | Configurable |
|------|---------|--------|---------|--------------|
| 8090 | GraphQL Server | semmem | GraphQL API + Playground | Yes |
| TBD | MCP Server | semmem | stdio transport (no port) | N/A |

### SemOps Application

| Port | Service | Module | Purpose | Configurable |
|------|---------|--------|---------|--------------|
| 14550 | MAVLink UDP | semops | Drone telemetry | Yes |

## Port Ranges

### Reserved Ranges

- **8080-8089**: SemStreams core services
- **8090-8099**: SemMem services
- **8100-8109**: SemOps services
- **14550-14559**: Robotics/MAVLink protocols
- **18000-19000**: Test infrastructure (dynamic allocation)

### Test Ports

Tests should use `getAvailablePort()` or similar to avoid conflicts:

- Integration tests: Dynamic allocation
- E2E tests: Fixed ports in 18000+ range with cleanup

## Configuration

All port assignments should be configurable via:

1. Configuration files (`*.json`)
2. Environment variables
3. CLI flags

### Example Configuration

```json
{
  "http_port": 8090,
  "websocket_port": 8081,
  "nats_url": "nats://localhost:4222"
}
```

### Environment Variables

```bash
SEMMEM_GRAPHQL_PORT=8090
SEMSTREAMS_HTTP_PORT=8080
NATS_PORT=4222
```

## Adding New Services

When adding a new HTTP service:

1. **Check this document** for available ports
2. **Pick from appropriate range**:
   - SemStreams core: 8080-8089
   - SemMem: 8090-8099
   - SemOps: 8100-8109
3. **Update this document** with your allocation
4. **Make port configurable** in service config
5. **Document default** in README and config schema

## Avoiding Collisions

### Development

- Run services on different ports
- Use Docker Compose port mapping
- Check `netstat -an | grep LISTEN` before starting

### Testing

- Use dynamic port allocation (`net.Listen(":0")`)
- Tests clean up resources properly
- Parallel test execution isolated

### Production

- Use reverse proxy (nginx, Caddy) for TLS termination
- Map external ports differently than internal
- Document port requirements in deployment guides

## Common Collisions

**Problem**: "address already in use"

**Solutions**:

1. Check if service is already running: `lsof -i :<port>`
2. Change port in configuration
3. Stop conflicting service
4. Use dynamic port allocation in tests

## References

- NATS default ports: <https://docs.nats.io/running-a-nats-service/configuration>
- TEI (Text Embeddings Inference): Port 8082 (avoids Ollama collision on 11434)
- MAVLink/ArduPilot: Port 14550 (SITL default)
- HTTP common ports: 8080, 8081, 8090, 9090 (avoid 80, 443 which require root)

## Notes

**Why not port 11434 for TEI?**
Port 11434 is Ollama's default port. Using it for TEI would cause collisions on developer machines with local Ollama installations. TEI uses 8082 from the SemStreams core services range instead.

## Updates

When ports change:

1. Update this document
2. Update configuration files in `configs/`
3. Update documentation in affected modules
4. Update tests using hardcoded ports
5. Announce changes in CHANGELOG

---

*Last Updated: 2025-11-17*
*Maintainer: SemStreams Team*
