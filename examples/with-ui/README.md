# With UI Example

Visual flow builder for creating data processing pipelines with drag-and-drop. **Minimal config, maximum usability**!

## What You Get

- **NATS**: Message broker
- **SemStreams**: Stream processing backend
- **SemStreams UI**: Visual flow builder (SvelteKit app)

## Prerequisites

- Docker & Docker Compose
- Ports 3000, 8080, and 9090 available

## Quick Start

```bash
# Config file is provided (config.minimal.json)
# Just start the stack
docker compose up -d

# Wait for services (about 10-15 seconds)
docker compose ps

# Open the UI
open http://localhost:3000
```

## Endpoints

| Service | URL | Purpose |
|---------|-----|---------|
| **UI** | http://localhost:3000 | Visual flow builder |
| API | http://localhost:8080 | Backend HTTP API |
| Health | http://localhost:8080/health | Backend health |
| Metrics | http://localhost:9090/metrics | Prometheus metrics |

## What You Can Do

### 1. **Create Flows Visually**
- Drag components onto canvas
- Connect them to create pipelines
- Configure component settings
- Save and deploy flows

### 2. **Available Components**
- **Inputs**: UDP, HTTP, WebSocket
- **Processors**: JSON filter, mapper, transformer
- **Outputs**: File, HTTP POST, WebSocket, Object Storage

### 3. **Monitor Flows**
- View real-time metrics
- Check component health
- Debug message flow

## Configuration

### Backend Config (`config.minimal.json`)

Minimal 20-line config - same as quickstart:
```json
{
  "platform": { "org": "quickstart", "id": "demo-001" },
  "nats": { "urls": ["nats://nats:4222"] },
  "services": {
    "component-manager": { "enabled": true },
    "metrics": { "enabled": true, "config": { "port": 9090 } }
  },
  "components": {}  // Add flows here or via UI
}
```

### UI Auto-Discovery

The UI **automatically discovers** backend capabilities:

- Fetches available component types from `/components/types`
- Loads OpenAPI spec from `/openapi.yaml`
- Uses FlowBuilder API at `/flowbuilder/*`

### Defaults

**UI**:
- Port: 3000 (SvelteKit production)
- Backend: Auto-detected from `BACKEND_HOST` env var
- Theme: Light/dark mode toggle
- Auto-save: Flows saved to backend

**Backend**:
- Same minimal config as quickstart
- Component manager enabled (for UI discovery)
- Metrics enabled

## Common Tasks

### Save a Flow

Flows created in UI are automatically saved to the backend.

### Load Existing Flow

Navigate to "Flows" tab to see all saved flows.

### Export Flow as JSON

Use "Export" button to download flow configuration for version control or deployment.

## Troubleshooting

**UI shows "Backend Unavailable"?**
```bash
# Check backend health
curl http://localhost:8080/health

# View backend logs
docker compose logs semstreams
```

**Can't connect to UI?**
```bash
# Check UI is running
docker compose ps ui
docker compose logs ui
```

**Port conflict?**
```yaml
# Change UI port in docker-compose.yml
ports:
  - "3001:3000"  # Host:3001 -> Container:3000
```

## Next Steps

**Add Semantic Search**:
- See `examples/with-embeddings/`

**Production Deployment**:
- See `examples/production/`

**Custom Components**:
- Configure flows in backend
- UI automatically reflects changes
