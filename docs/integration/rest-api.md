# REST API

SemStreams provides a RESTful HTTP API for entity management, flow control, and system health.

## Base URL

```
http://localhost:8080/api
```

## Authentication

Currently no authentication required (development mode). Production deployments should configure authentication via reverse proxy or API gateway.

## Endpoints

### Health & Status

#### GET /health

Health check endpoint.

**Response:**
```json
{
  "status": "healthy",
  "timestamp": "2025-11-17T15:30:00Z",
  "version": "0.2.0-alpha",
  "services": {
    "nats": "connected",
    "embedding": "available"
  }
}
```

#### GET /metrics

Prometheus metrics endpoint.

**Response:** Prometheus text format
```
# HELP semstreams_entities_total Total number of entities
# TYPE semstreams_entities_total counter
semstreams_entities_total 1234

# HELP semstreams_flows_active Number of active flows
# TYPE semstreams_flows_active gauge
semstreams_flows_active 3
```

### Entities

#### GET /api/entities

List entities with optional filtering.

**Query Parameters:**
- `type` (string): Filter by entity type
- `limit` (int): Max results (default: 10, max: 100)
- `offset` (int): Pagination offset (default: 0)

**Example:**
```bash
curl "http://localhost:8080/api/entities?type=Sensor&limit=20"
```

**Response:**
```json
{
  "entities": [
    {
      "id": "sensor-123",
      "type": "schema:Device",
      "properties": {
        "name": "Temperature Sensor",
        "value": "72.5",
        "unit": "fahrenheit"
      },
      "edges": [
        {
          "predicate": "installedIn",
          "target": "room-201"
        }
      ],
      "timestamp": "2025-11-17T15:30:00Z"
    }
  ],
  "total": 42,
  "limit": 20,
  "offset": 0
}
```

#### GET /api/entities/{id}

Get entity by ID.

**Example:**
```bash
curl "http://localhost:8080/api/entities/sensor-123"
```

**Response:**
```json
{
  "id": "sensor-123",
  "type": "schema:Device",
  "properties": {
    "name": "Temperature Sensor",
    "value": "72.5"
  },
  "edges": [
    {
      "predicate": "installedIn",
      "target": "room-201"
    }
  ],
  "timestamp": "2025-11-17T15:30:00Z"
}
```

**Error Response (404):**
```json
{
  "error": "entity not found",
  "entity_id": "sensor-999"
}
```

#### POST /api/entities

Create new entity.

**Request:**
```json
{
  "id": "sensor-456",
  "type": "schema:Device",
  "properties": {
    "name": "Pressure Sensor",
    "value": "101.3",
    "unit": "kPa"
  },
  "edges": [
    {
      "predicate": "installedIn",
      "target": "room-202"
    }
  ]
}
```

**Example:**
```bash
curl -X POST http://localhost:8080/api/entities \
  -H "Content-Type: application/json" \
  -d '{
    "id": "sensor-456",
    "type": "schema:Device",
    "properties": {
      "name": "Pressure Sensor"
    }
  }'
```

**Response (201 Created):**
```json
{
  "id": "sensor-456",
  "type": "schema:Device",
  "properties": {
    "name": "Pressure Sensor"
  },
  "edges": [],
  "timestamp": "2025-11-17T15:35:00Z"
}
```

#### PUT /api/entities/{id}

Update existing entity.

**Request:**
```json
{
  "properties": {
    "value": "72.8",
    "lastUpdate": "2025-11-17T15:40:00Z"
  }
}
```

**Example:**
```bash
curl -X PUT http://localhost:8080/api/entities/sensor-123 \
  -H "Content-Type: application/json" \
  -d '{"properties": {"value": "72.8"}}'
```

**Response (200 OK):**
```json
{
  "id": "sensor-123",
  "type": "schema:Device",
  "properties": {
    "name": "Temperature Sensor",
    "value": "72.8"
  },
  "timestamp": "2025-11-17T15:40:00Z"
}
```

#### DELETE /api/entities/{id}

Delete entity.

**Example:**
```bash
curl -X DELETE http://localhost:8080/api/entities/sensor-123
```

**Response (204 No Content)**

### Search

#### POST /api/search

Semantic search across entities.

**Request:**
```json
{
  "query": "temperature sensor building A",
  "limit": 10,
  "threshold": 0.5,
  "types": ["schema:Device"]
}
```

**Example:**
```bash
curl -X POST http://localhost:8080/api/search \
  -H "Content-Type: application/json" \
  -d '{
    "query": "temperature sensor",
    "limit": 5
  }'
```

**Response:**
```json
{
  "results": [
    {
      "entity": {
        "id": "sensor-123",
        "type": "schema:Device",
        "properties": {
          "name": "Temperature Sensor"
        }
      },
      "score": 0.92,
      "snippet": "Temperature Sensor in Building A, Floor 2"
    }
  ],
  "total": 3,
  "query_time": "15ms"
}
```

### Flows

#### GET /api/flows

List all flows.

**Response:**
```json
{
  "flows": [
    {
      "id": "flow-1",
      "name": "Sensor Processing",
      "description": "Process temperature data",
      "status": "running",
      "components_count": 3
    }
  ]
}
```

#### GET /api/flows/{id}

Get flow details.

**Response:**
```json
{
  "id": "flow-1",
  "name": "Sensor Processing",
  "status": "running",
  "components": [
    {
      "id": "input-1",
      "type": "UDPInput",
      "config": {
        "port": 14550
      }
    }
  ],
  "connections": [
    {
      "from": "input-1",
      "to": "parser-1"
    }
  ]
}
```

#### POST /api/flows

Create new flow.

**Request:**
```json
{
  "name": "New Flow",
  "components": [...],
  "connections": [...]
}
```

#### POST /api/flows/{id}/start

Start flow.

**Response:**
```json
{
  "id": "flow-1",
  "status": "running",
  "started_at": "2025-11-17T15:45:00Z"
}
```

#### POST /api/flows/{id}/stop

Stop flow.

**Response:**
```json
{
  "id": "flow-1",
  "status": "stopped",
  "stopped_at": "2025-11-17T15:50:00Z"
}
```

## Error Responses

All errors follow consistent format:

```json
{
  "error": "error description",
  "code": "ERROR_CODE",
  "details": {
    "field": "additional context"
  }
}
```

**HTTP Status Codes:**
- `200 OK`: Success
- `201 Created`: Resource created
- `204 No Content`: Success, no response body
- `400 Bad Request`: Invalid input
- `404 Not Found`: Resource doesn't exist
- `409 Conflict`: Resource already exists
- `500 Internal Server Error`: Server error

**Error Codes:**
- `INVALID_INPUT`: Validation failed
- `NOT_FOUND`: Resource not found
- `ALREADY_EXISTS`: Duplicate resource
- `INTERNAL_ERROR`: Server error

## Rate Limiting

Development: No rate limiting
Production: Recommended 100 req/min per client

**Rate Limit Headers:**
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1700237400
```

## OpenAPI Specification

Full OpenAPI 3.0 specification available at:

```
GET /api/openapi.json
GET /api/openapi.yaml
```

Interactive API documentation (Swagger UI):
```
http://localhost:8080/api/docs
```

## Client Examples

### cURL

```bash
# Health check
curl http://localhost:8080/health

# Create entity
curl -X POST http://localhost:8080/api/entities \
  -H "Content-Type: application/json" \
  -d '{"id":"test-1","type":"Test","properties":{}}'

# Search
curl -X POST http://localhost:8080/api/search \
  -H "Content-Type: application/json" \
  -d '{"query":"sensor","limit":5}'
```

### JavaScript/TypeScript

```typescript
const baseURL = 'http://localhost:8080/api'

// Get entity
const response = await fetch(`${baseURL}/entities/sensor-123`)
const entity = await response.json()

// Create entity
await fetch(`${baseURL}/entities`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    id: 'sensor-456',
    type: 'schema:Device',
    properties: { name: 'New Sensor' }
  })
})

// Search
const searchResults = await fetch(`${baseURL}/search`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    query: 'temperature',
    limit: 10
  })
}).then(r => r.json())
```

### Python

```python
import requests

BASE_URL = 'http://localhost:8080/api'

# Get entity
response = requests.get(f'{BASE_URL}/entities/sensor-123')
entity = response.json()

# Create entity
new_entity = {
    'id': 'sensor-456',
    'type': 'schema:Device',
    'properties': {'name': 'New Sensor'}
}
response = requests.post(f'{BASE_URL}/entities', json=new_entity)

# Search
search_result = requests.post(f'{BASE_URL}/search', json={
    'query': 'temperature',
    'limit': 10
}).json()
```

### Go

```go
import (
    "bytes"
    "encoding/json"
    "net/http"
)

const baseURL = "http://localhost:8080/api"

// Get entity
resp, err := http.Get(baseURL + "/entities/sensor-123")
var entity Entity
json.NewDecoder(resp.Body).Decode(&entity)

// Create entity
newEntity := Entity{
    ID:   "sensor-456",
    Type: "schema:Device",
    Properties: map[string]interface{}{
        "name": "New Sensor",
    },
}
body, _ := json.Marshal(newEntity)
http.Post(baseURL+"/entities", "application/json", bytes.NewReader(body))
```

## See Also

- [GraphQL API](./graphql-api.md) - Graph-based queries
- [NATS Events](./nats-events.md) - Event-driven integration
- [Deployment Guide](../deployment/production.md) - Production setup
