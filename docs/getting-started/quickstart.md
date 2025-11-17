# Quick Start Guide

Get the SemStreams ecosystem running in 5 minutes.

## Prerequisites

- Docker & Docker Compose
- [Task](https://taskfile.dev/#/installation) (optional but recommended)
- Git

## 1. Clone Repositories

```bash
# Core engine
git clone https://github.com/c360/semstreams
cd semstreams

# Optional: UI and supporting services
git clone https://github.com/c360/semstreams-ui ../semstreams-ui
git clone https://github.com/c360/semembed ../semembed
```

## 2. Start Services

### Option A: Full Stack (Recommended)

```bash
cd semstreams
docker compose -f docker-compose.dev.yml up
```

This starts:
- SemStreams core engine (GraphQL + REST APIs)
- NATS JetStream (message broker)
- SemStreams UI (web interface)
- Embedding service (optional)

### Option B: Core Only

```bash
cd semstreams
task dev
```

## 3. Verify Services

```bash
# Health checks
curl http://localhost:8080/health          # Core engine
curl http://localhost:3000                  # UI (if running)
curl http://localhost:4222                  # NATS

# GraphQL Playground
open http://localhost:8080/graphql
```

## 4. Try a Simple Flow

### Via GraphQL

```graphql
mutation {
  createEntity(input: {
    id: "sensor-1"
    type: "Sensor"
    properties: {
      name: "Temperature Sensor"
      location: "Building A"
      value: "72.5"
    }
  }) {
    id
    type
  }
}

query {
  entity(id: "sensor-1") {
    id
    type
    properties
  }
}
```

### Via REST API

```bash
# Create entity
curl -X POST http://localhost:8080/api/entities \
  -H "Content-Type: application/json" \
  -d '{
    "id": "sensor-1",
    "type": "Sensor",
    "properties": {
      "name": "Temperature Sensor",
      "location": "Building A",
      "value": "72.5"
    }
  }'

# Query entity
curl http://localhost:8080/api/entities/sensor-1
```

## 5. Explore the UI

Open http://localhost:3000 to access:
- **Flow Builder**: Visual flow construction
- **Entity Explorer**: Browse the knowledge graph
- **Query Console**: Interactive GraphQL queries
- **Monitoring**: Real-time metrics and health

## Next Steps

- **[Core Concepts](./concepts.md)**: Understand semantic streaming fundamentals
- **[Architecture Overview](./architecture.md)**: Learn how components work together
- **[Integration Guide](../integration/)**: Connect your applications
- **[PathRAG Guide](../guides/pathrag.md)**: Extract knowledge from graphs

## Common Issues

### Port Already in Use

```bash
# Check what's using ports
lsof -i :8080  # SemStreams
lsof -i :3000  # UI
lsof -i :4222  # NATS

# Change ports in docker-compose.dev.yml or config files
```

### Services Not Starting

```bash
# Check logs
docker compose logs semstreams
docker compose logs nats
docker compose logs ui

# Rebuild from scratch
docker compose down -v
docker compose build --no-cache
docker compose up
```

### Out of Memory

```bash
# Check Docker resources
docker stats

# Increase Docker memory limit in Docker Desktop preferences
# Minimum recommended: 4GB RAM
```

## Development Mode

For active development on a specific service:

```bash
# SemStreams core
cd semstreams
task dev           # Run with hot reload
task test          # Run tests
task e2e           # End-to-end tests

# UI
cd semstreams-ui
npm run dev        # Vite dev server
npm test           # Run tests
```

See individual repository READMEs for service-specific development guides.
