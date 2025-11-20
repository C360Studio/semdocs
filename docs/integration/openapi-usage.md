# OpenAPI Usage Guide

How to use OpenAPI specifications with SemStreams

---

## Overview

SemStreams provides **two distinct OpenAPI 3.0 specifications** for different purposes:

1. **Component Configuration API** - Schema discovery for building flows
2. **Service HTTP APIs** - Service endpoint documentation

Understanding which spec to use depends on your integration needs.

---

## Use Case 1: Component Configuration API

### Purpose

Discover available component types, their configuration schemas, and build flows programmatically.

### When to Use

- Building UI flow builders
- Generating client libraries for flow management
- Validating flow configurations
- Auto-completing component configs

### Specification File

**Location:** `specs/openapi.v3.yaml` (in semstreams repository)

**Generated From:** Component schema tags via `task schema:generate`

**Contains:**
- Component type metadata
- Configuration schemas (JSON Schema Draft-07)
- Component discovery endpoints
- Flow validation endpoints

### Key Endpoints

```text
GET /components/types           # List all component types
GET /components/types/{id}      # Get specific component schema
GET /components/status/{name}   # Component instance status
GET /components/flowgraph       # Flow connectivity graph
GET /components/validate        # Validate flow connectivity
```

### Example: List Component Types

```bash
curl http://localhost:8080/components/types
```

**Response:**
```json
[
  {
    "id": "udp",
    "name": "UDP Input",
    "type": "input",
    "protocol": "udp",
    "domain": "network",
    "description": "UDP input for receiving MAVLink and other UDP data",
    "version": "1.0.0",
    "schema": {
      "$ref": "../schemas/udp.v1.json"
    }
  },
  {
    "id": "graph-processor",
    "name": "Graph Processor",
    "type": "processor",
    "protocol": "nats",
    "domain": "semantic",
    "description": "Semantic graph processor with entity state management",
    "version": "1.0.0",
    "schema": {
      "$ref": "../schemas/graph-processor.v1.json"
    }
  }
]
```

### Example: Get Component Schema

```bash
curl http://localhost:8080/components/types/udp
```

**Response:**
```json
{
  "id": "udp",
  "name": "UDP Input",
  "type": "input",
  "schema": {
    "$schema": "http://json-schema.org/draft-07/schema#",
    "type": "object",
    "properties": {
      "port": {
        "type": "number",
        "description": "UDP port to listen on",
        "minimum": 1,
        "maximum": 65535,
        "default": 14550
      },
      "address": {
        "type": "string",
        "description": "Bind address",
        "default": "0.0.0.0"
      }
    },
    "required": ["port"]
  }
}
```

### Client Generation

Generate TypeScript types from OpenAPI spec:

```bash
# In semstreams-ui repository
npx openapi-typescript ../semstreams/specs/openapi.v3.yaml \
  -o src/lib/types/api.generated.ts
```

**Generated Types:**
```typescript
export interface components {
  schemas: {
    ComponentType: {
      id: string;
      name: string;
      type: 'input' | 'processor' | 'output' | 'storage';
      protocol?: string;
      domain?: string;
      description: string;
      version: string;
      schema: object;
    };
  };
}

export interface paths {
  '/components/types': {
    get: {
      responses: {
        200: {
          content: {
            'application/json': components['schemas']['ComponentType'][];
          };
        };
      };
    };
  };
}
```

### Schema Tag → OpenAPI Flow

Component configuration schemas are generated from Go struct tags:

```go
// Component implementation with schema tags
type Config struct {
    Port int `schema:"type:int,desc:UDP port,min:1,max:65535,default:14550"`
}

// ↓ task schema:generate

// Generated JSON Schema
{
  "type": "number",
  "description": "UDP port",
  "minimum": 1,
  "maximum": 65535,
  "default": 14550
}

// ↓ Included in OpenAPI spec

// OpenAPI component schema reference
{
  "$ref": "../schemas/udp.v1.json"
}
```

**See:** [Schema Tags Guide](../development/schema-tags.md) for complete schema tag documentation

---

## Use Case 2: Service HTTP APIs

### Purpose

Document HTTP endpoints exposed by SemStreams services for monitoring, management, and integration.

### When to Use

- Integrating with SemStreams services
- Building monitoring dashboards
- Creating admin tools
- Accessing service-specific APIs

### Specification Source

**Implementation:** Service Manager aggregates OpenAPI specs from all services

**How It Works:**

1. Service implements `HTTPHandler` interface:
   ```go
   type HTTPHandler interface {
       RegisterHTTPHandlers(prefix string, mux *http.ServeMux)
       OpenAPISpec() *OpenAPISpec
   }
   ```

2. Service provides OpenAPI spec:
   ```go
   func (s *MyService) OpenAPISpec() *service.OpenAPISpec {
       return &service.OpenAPISpec{
           Paths: map[string]service.PathItem{
               "/status": {
                   Get: &service.Operation{
                       Summary: "Get service status",
                       Responses: map[string]service.Response{
                           "200": {Description: "Service status"},
                       },
                   },
               },
           },
       }
   }
   ```

3. Service Manager aggregates all specs at runtime

### Example Services

**ComponentManager Service:**
- `/api/v1/components` - List managed components
- `/api/v1/components/{name}` - Component details
- `/api/v1/components/{name}/health` - Health check
- `/api/v1/flowgraph` - Flow connectivity graph

**FlowService Runtime Observability:**
- `/flowbuilder/flows/{id}/runtime/metrics` - Component metrics
- `/flowbuilder/flows/{id}/runtime/health` - Component health
- `/flowbuilder/flows/{id}/runtime/messages` - Message flow
- `/flowbuilder/flows/{id}/runtime/logs` - Component logs (SSE)

**MessageLogger Service:**
- `/api/v1/messages` - Query message log
- `/api/v1/messages/{id}` - Get specific message

### Example: Component Health

```bash
curl http://localhost:8080/api/v1/components/udp-source/health
```

**Response:**
```json
{
  "name": "udp-source",
  "type": "udp",
  "status": "running",
  "healthy": true,
  "message": "Listening on :14550",
  "start_time": "2025-11-20T14:07:33.123Z",
  "last_activity": "2025-11-20T14:23:04.567Z",
  "uptime_seconds": 932
}
```

### Example: Flow Runtime Metrics

```bash
curl http://localhost:8080/flowbuilder/flows/my-flow/runtime/metrics
```

**Response:**
```json
{
  "timestamp": "2025-11-20T14:23:05.123Z",
  "components": [
    {
      "name": "udp-source",
      "throughput": 1234.5,
      "error_rate": 0.0,
      "queue_depth": 0,
      "status": "healthy"
    },
    {
      "name": "json-parser",
      "throughput": 1234.5,
      "error_rate": 0.0,
      "queue_depth": 0,
      "status": "healthy"
    }
  ],
  "prometheus_available": true
}
```

**See:** [Service Package README](https://github.com/c360/semstreams/blob/main/service/README.md) for service implementation details

---

## Comparison Table

| Aspect | Component Config API | Service HTTP APIs |
|--------|---------------------|-------------------|
| **Purpose** | Component discovery & config | Service monitoring & management |
| **Spec Location** | `specs/openapi.v3.yaml` | Aggregated at runtime |
| **Generated From** | Schema tags (build-time) | Service implementations (runtime) |
| **Primary Users** | UI developers, client library authors | Operators, integrators, admins |
| **Endpoints** | `/components/*` | `/api/v1/*`, `/flowbuilder/*` |
| **Documentation** | [Schema Tags](../development/schema-tags.md) | [Service README](https://github.com/c360/semstreams/blob/main/service/README.md) |
| **Updates** | Regenerate with `task schema:generate` | Automatic (service registration) |

---

## Client Generation Workflow

### TypeScript (UI Development)

```bash
# In semstreams-ui repository

# Generate types from component config API
npx openapi-typescript ../semstreams/specs/openapi.v3.yaml \
  -o src/lib/types/components.generated.ts

# Use in code
import type { components } from '$lib/types/components.generated';

type ComponentType = components['schemas']['ComponentType'];

async function loadComponents(): Promise<ComponentType[]> {
  const response = await fetch('/components/types');
  return response.json();
}
```

### Go (Backend Integration)

```bash
# Generate Go client from OpenAPI spec
go install github.com/deepmap/oapi-codegen/cmd/oapi-codegen@latest

oapi-codegen -package client -generate types,client \
  specs/openapi.v3.yaml > client/generated.go
```

### Python (Automation/Scripts)

```bash
# Generate Python client
pip install openapi-python-client

openapi-python-client generate --url http://localhost:8080/components/types
```

---

## Best Practices

### For Component Config API

1. **Cache Component Types** - Component types rarely change, cache the list
2. **Validate Against Schema** - Use JSON Schema validation before submitting configs
3. **Handle Schema Evolution** - Support multiple schema versions (v1, v2)
4. **Type Generation** - Regenerate types when component schemas change

### For Service HTTP APIs

1. **Poll with Backoff** - Runtime metrics endpoints are designed for polling (2-5s intervals)
2. **Handle Timeouts** - Endpoints have strict timeouts (<100ms or <200ms)
3. **Graceful Degradation** - Services may be unavailable, handle errors gracefully
4. **Use SSE for Logs** - Log streaming uses Server-Sent Events, handle reconnection

### Security

1. **No Auth in Dev** - Development mode has no authentication
2. **Production Deployment** - Use reverse proxy or API gateway for auth
3. **Rate Limiting** - Production should implement rate limiting (100 req/min recommended)
4. **TLS** - Always use TLS in production for API endpoints

---

## Implementation Documentation

### For Go Developers

**Component Schema Tags:**
- [Schema Tags Guide](../development/schema-tags.md) - Writing schema tags
- [semstreams/docs/SCHEMA_GENERATION.md](https://github.com/c360/semstreams/blob/main/docs/SCHEMA_GENERATION.md) - Generation system
- [Writing Components](../development/writing-components.md) - Component development

**Service HTTP Handlers:**
- [semstreams/service/README.md](https://github.com/c360/semstreams/blob/main/service/README.md) - Service implementation
- [semstreams/service/openapi.go](https://github.com/c360/semstreams/blob/main/service/openapi.go) - OpenAPI types

### For Contract Testing

**Backend:**
- [semstreams/docs/CONTRACT_TESTING.md](https://github.com/c360/semstreams/blob/main/docs/CONTRACT_TESTING.md) - Schema validation tests
- `semstreams/test/contract/openapi_contract_test.go` - OpenAPI contract tests

**Frontend:**
- `semstreams-ui/src/lib/contract/openapi.contract.test.ts` - OpenAPI spec tests

---

## Troubleshooting

### Component Types Not Found

**Problem:** `GET /components/types` returns empty array

**Solution:**
```bash
# Check component manager is enabled
curl http://localhost:8080/health

# Verify config includes component-manager service
cat config.json | jq '.services."component-manager"'

# Check component registration
docker compose logs semstreams | grep "Registered component"
```

### Schema Not Generated

**Problem:** Component schema missing from OpenAPI spec

**Solution:**
```bash
# Regenerate schemas
cd semstreams
task schema:generate

# Check schema file exists
ls -la schemas/your-component.v1.json

# Verify OpenAPI spec includes component
cat specs/openapi.v3.yaml | grep "your-component"
```

### Type Generation Fails

**Problem:** `openapi-typescript` fails to generate types

**Solution:**
```bash
# Validate OpenAPI spec
npx @apidevtools/swagger-cli validate specs/openapi.v3.yaml

# Check for syntax errors
cat specs/openapi.v3.yaml | yq eval '.' -

# Regenerate from source
cd semstreams && task schema:generate
```

---

## See Also

- [REST API Reference](./rest-api.md) - Complete REST API documentation
- [GraphQL API](./graphql-api.md) - GraphQL alternative for queries
- [NATS Events](./nats-events.md) - Event-driven integration
- [Schema Tags Guide](../development/schema-tags.md) - Writing component schemas
- [Writing Components](../development/writing-components.md) - Component development

---

**Key Takeaway:** SemStreams provides two OpenAPI specs - one for component configuration (build-time generated from schema tags) and one for service HTTP APIs (runtime aggregated from service implementations). Use the right spec for your integration needs.
