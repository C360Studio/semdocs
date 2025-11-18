# GraphQL API

SemStreams provides a **schema-based GraphQL gateway** that automatically maps domain-specific GraphQL schemas to NATS-based knowledge graph queries. This enables agents and applications to query semantic graphs with type-safe, efficient GraphQL queries without writing custom resolvers.

## The Schema-Based Magic

### How It Works

**Traditional approach**: Write resolvers for every GraphQL field manually

**SemStreams approach**: Define your domain schema, the gateway generates everything

```text
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Define Domain Schema (GraphQL)                      │
│                                                              │
│   type Spec {                                               │
│     id: ID!                                                 │
│     title: String!                                          │
│     dependencies: [Spec!]!  # Relationship                  │
│   }                                                         │
└───────────────────────────┬─────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 2: Configure Type Mappings                             │
│                                                              │
│   Spec → entity type "org.semmem.spec"                     │
│   dependencies → edge type "depends_on"                     │
└───────────────────────────┬─────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 3: Gateway Auto-Generates (gqlgen)                     │
│                                                              │
│   - Type-safe Go resolvers                                  │
│   - NATS query clients                                      │
│   - Property extraction                                     │
│   - Relationship traversal                                  │
└───────────────────────────┬─────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Result: Query Your Domain Model                             │
│                                                              │
│   query { spec(id: "123") { title dependencies { title }}} │
│          ↓ NATS                                             │
│   graph.query.entity → Knowledge Graph                      │
└─────────────────────────────────────────────────────────────┘
```

**The magic**: Your GraphQL schema IS your API. No resolver boilerplate, just domain modeling.

## Endpoint

```text
http://localhost:8080/graphql
```

**GraphQL Playground**: http://localhost:8080/graphql (in development mode)

## Architecture

### Three-Layer Model

**1. GraphQL Schema Layer** (Your Domain)

- Define your types (Spec, Doc, Issue, etc.)
- Declare relationships
- Add custom queries

**2. Gateway Layer** (Auto-Generated)

- gqlgen code generation from schema
- NATS query integration
- Generic property extraction
- Automatic relationship resolution

**3. Knowledge Graph Layer** (SemStreams Core)

- Entity storage (NATS KV)
- Graph traversal (PathRAG)
- Semantic search (GraphRAG)
- Real-time subscriptions

## Configuration

### Gateway Component

```json
{
  "components": [
    {
      "type": "gateway",
      "name": "graphql",
      "config": {
        "http_port": 8090,
        "path": "/graphql",
        "playground": true,
        "schema_file": "./schema.graphql",

        "nats_subjects": {
          "entity": "graph.query.entity",
          "predicate": "graph.query.predicate",
          "relationships": "graph.query.relationships",
          "semantic": "graph.query.semantic"
        },

        "type_mappings": {
          "Spec": {
            "entity_type": "org.semmem.spec",
            "id_prefix": "org.semmem.spec.",
            "properties": {
              "title": "title",
              "status": "status",
              "content": "content"
            }
          }
        },

        "edge_mappings": {
          "Spec.dependencies": {
            "edge_type": "depends_on",
            "direction": "outgoing"
          }
        }
      }
    }
  ]
}
```

### Field Reference

| Field | Type | Description |
|-------|------|-------------|
| `schema_file` | string | Path to GraphQL schema file |
| `nats_subjects` | object | NATS subject routing for queries |
| `type_mappings` | object | Map GraphQL types to entity types |
| `edge_mappings` | object | Map GraphQL fields to graph edges |
| `playground` | boolean | Enable GraphiQL playground (dev only) |

## Authentication

Currently no authentication required (development mode). Production deployments should configure authentication via reverse proxy or API gateway.

## Why Schema-Based Gateway?

### The Agent Problem

**Traditional REST APIs cause token waste for LLMs:**

```text
Agent needs: Spec title + dependencies
REST returns: Entire Spec object (5KB)
Agent pays:   1000+ tokens for 2 fields ❌

GraphQL request: Exactly title + dependencies
GraphQL returns: Only requested fields (200 bytes)
Agent pays:     50 tokens ✅
```

**Query precision = Cost savings + Faster responses**

### Benefits

| Benefit | Description |
|---------|-------------|
| **Type Safety** | gqlgen generates type-safe resolvers from schema |
| **Zero Boilerplate** | No manual resolver writing for relationships |
| **NATS Integration** | Automatic mapping to knowledge graph queries |
| **Agentic-Friendly** | Agents craft precise queries, minimize tokens |
| **Domain-Driven** | Model your domain, gateway handles the rest |
| **Real-time** | GraphQL subscriptions for graph updates |

## Example Schemas

### Knowledge Graph Schema (SemMem)

**Domain**: Software specifications and GitHub integration

```graphql
type Spec {
  id: ID!
  title: String!
  status: SpecStatus!
  version: String!
  content: String!

  # Relationships (auto-resolved via edge_mappings)
  dependencies: [Spec!]!
  implementedBy: [PullRequest!]!
}

enum SpecStatus {
  DRAFT
  ACTIVE
  DEPRECATED
}

type PullRequest {
  id: ID!
  number: Int!
  title: String!
  state: PRState!
  implements: [Spec!]!
}

enum PRState {
  OPEN
  MERGED
  CLOSED
}

type Query {
  # Entity queries
  spec(id: ID!): Spec
  specs(status: SpecStatus, limit: Int): [Spec!]!

  # Semantic search
  searchSpecs(query: String!, limit: Int): [SearchResult!]!

  # Relationship traversal (PathRAG)
  specDependencyChain(id: ID!, maxDepth: Int): [Spec!]!
}

type SearchResult {
  spec: Spec!
  score: Float!
  snippet: String
}
```

### IoT Monitoring Schema

**Domain**: Sensor networks and device management

```graphql
type Device {
  id: ID!
  name: String!
  type: DeviceType!
  location: Location!
  status: DeviceStatus!

  # Relationships
  sensors: [Sensor!]!
  nearbyDevices(radius: Float): [Device!]!  # Spatial query
}

enum DeviceType {
  DRONE
  BASE_STATION
  SENSOR_NODE
}

enum DeviceStatus {
  ONLINE
  OFFLINE
  MAINTENANCE
}

type Sensor {
  id: ID!
  type: String!
  lastReading: Float
  unit: String
  device: Device!
}

type Location {
  latitude: Float!
  longitude: Float!
  altitude: Float
}

type Query {
  device(id: ID!): Device
  devices(status: DeviceStatus, type: DeviceType): [Device!]!

  # Semantic search
  searchDevices(query: String!, threshold: Float): [DeviceSearchResult!]!

  # PathRAG query
  deviceNetwork(startDeviceId: ID!, maxHops: Int): [Device!]!
}
```

## Example Queries

### Precise Field Selection (Agent-Optimized)

**Agent needs**: Just spec titles and their dependency count

```graphql
query SpecSummaries {
  specs(status: ACTIVE, limit: 10) {
    title
    dependencies {
      title
    }
  }
}
```

**Result**: ~200 bytes (vs 5KB+ for full REST endpoint)

### Relationship Traversal with PathRAG

**Find all dependencies of a spec recursively:**

```graphql
query SpecDependencyChain {
  specDependencyChain(id: "spec-123", maxDepth: 3) {
    id
    title
    status
    version
  }
}
```

**Behind the scenes**: Gateway uses PathRAG to traverse graph edges

### Semantic Search Integration

**Find related specs using GraphRAG:**

```graphql
query FindRelatedSpecs {
  searchSpecs(query: "authentication OAuth2", limit: 5) {
    score
    snippet
    spec {
      id
      title
      status
      dependencies {
        title
      }
    }
  }
}
```

**Result**: Semantic similarity scores + relationship context in one query

### Spatial Queries (IoT)

**Find devices near a location:**

```graphql
query NearbyDevices {
  device(id: "drone-001") {
    name
    location {
      latitude
      longitude
    }
    nearbyDevices(radius: 5000.0) {
      name
      type
      status
    }
  }
}
```

## Subscriptions

**Real-time updates via GraphQL subscriptions** (powered by NATS JetStream):

```graphql
type Subscription {
  # Watch for spec updates
  specUpdated(id: ID): Spec!

  # Watch for new dependencies
  dependencyAdded(specId: ID): Spec!

  # Watch device status changes
  deviceStatusChanged(deviceId: ID): Device!
}
```

### Example Subscription

```graphql
subscription WatchSpecChanges {
  specUpdated(id: "spec-123") {
    id
    title
    status
    dependencies {
      title
    }
  }
}
```

**How it works**: NATS JetStream streams → WebSocket subscriptions → Real-time GraphQL updates

## Code Generation with gqlgen

### How the Gateway Generates Resolvers

**1. Define Schema** (`schema.graphql`):

```graphql
type Spec {
  id: ID!
  title: String!
  dependencies: [Spec!]!
}
```

**2. Run gqlgen** (automatic):

```bash
go run github.com/99designs/gqlgen generate
```

**3. Generated Code** (automatic):

```go
// schema.resolvers.go (generated)
func (r *specResolver) Dependencies(ctx context.Context, obj *Spec) ([]*Spec, error) {
    // Auto-generated: queries NATS for edges with type "depends_on"
    edges, err := r.natsClient.QueryRelationships(ctx, obj.ID, "depends_on")
    // Resolves each edge to Spec entity
    return r.resolveSpecs(ctx, edges)
}
```

**Result**: Zero manual resolver code for relationships

### Configuration-Driven Resolution

**Type mapping** tells gateway how GraphQL types map to entities:

```json
{
  "type_mappings": {
    "Spec": {
      "entity_type": "org.semmem.spec",  // NATS entity type
      "id_prefix": "org.semmem.spec.",    // ID format
      "properties": {
        "title": "title",                 // GraphQL → entity property
        "status": "status",
        "content": "content"
      }
    }
  }
}
```

**Edge mapping** defines relationship resolution:

```json
{
  "edge_mappings": {
    "Spec.dependencies": {
      "edge_type": "depends_on",  // NATS edge type
      "direction": "outgoing"     // Follow outgoing edges
    },
    "Spec.implementedBy": {
      "edge_type": "implements",
      "direction": "incoming"     // Follow incoming edges
    }
  }
}
```

## Implementation Details

### NATS Integration

**Gateway translates GraphQL queries to NATS requests:**

| GraphQL Query | NATS Request | Subject |
|--------------|--------------|---------|
| `spec(id: "123")` | Get entity by ID | `graph.query.entity` |
| `spec.dependencies` | Query relationships | `graph.query.relationships` |
| `searchSpecs(query: "auth")` | Semantic search | `graph.query.semantic` |
| `specDependencyChain(...)` | PathRAG traversal | `graph.query.path` |

### Property Extraction

**Gateway automatically extracts properties from entity JSON:**

```json
{
  "entity_id": "org.semmem.spec.auth-spec",
  "properties": {
    "title": "OAuth2 Authentication",
    "status": "ACTIVE",
    "content": "..."
  }
}
```

**GraphQL schema field** → **Entity property lookup** → **Type coercion**

### Performance

| Operation | Latency | Notes |
|-----------|---------|-------|
| Single entity query | 1-5ms | NATS KV lookup |
| Relationship resolution | 5-20ms | PathRAG (depends on depth) |
| Semantic search | 10-50ms | GraphRAG with BM25 |
| Subscription delivery | <10ms | NATS JetStream → WebSocket |

## Error Handling

GraphQL errors follow the standard format:

```json
{
  "errors": [
    {
      "message": "Entity not found",
      "path": ["entity"],
      "extensions": {
        "code": "NOT_FOUND",
        "entityId": "sensor-999"
      }
    }
  ]
}
```

**Error Codes:**

- `NOT_FOUND`: Resource doesn't exist
- `INVALID_INPUT`: Validation failed
- `UNAUTHORIZED`: Authentication required
- `INTERNAL_ERROR`: Server error

## Rate Limiting

Development: No rate limiting
Production: Recommended 100 req/min per client

Configure via reverse proxy or API gateway.

## Client Libraries

### JavaScript/TypeScript

```typescript
import { ApolloClient, InMemoryCache, gql } from '@apollo/client'

const client = new ApolloClient({
  uri: 'http://localhost:8080/graphql',
  cache: new InMemoryCache()
})

const GET_ENTITY = gql`
  query GetEntity($id: ID!) {
    entity(id: $id) {
      id
      type
      properties
    }
  }
`

const { data } = await client.query({
  query: GET_ENTITY,
  variables: { id: 'sensor-123' }
})
```

### Go

```go
import "github.com/machinebox/graphql"

client := graphql.NewClient("http://localhost:8080/graphql")

req := graphql.NewRequest(`
  query GetEntity($id: ID!) {
    entity(id: $id) {
      id
      type
      properties
    }
  }
`)
req.Var("id", "sensor-123")

var resp struct {
    Entity struct {
        ID         string
        Type       string
        Properties map[string]interface{}
    }
}

if err := client.Run(ctx, req, &resp); err != nil {
    log.Fatal(err)
}
```

### Python

```python
from gql import gql, Client
from gql.transport.requests import RequestsHTTPTransport

transport = RequestsHTTPTransport(url='http://localhost:8080/graphql')
client = Client(transport=transport, fetch_schema_from_transport=True)

query = gql('''
  query GetEntity($id: ID!) {
    entity(id: $id) {
      id
      type
      properties
    }
  }
''')

result = client.execute(query, variable_values={'id': 'sensor-123'})
```

## Schema Introspection

Get the full schema:

```graphql
query IntrospectionQuery {
  __schema {
    types {
      name
      kind
      description
    }
  }
}
```

Or use GraphQL Playground for interactive exploration: http://localhost:8080/graphql

## Reference Implementations

- **SemMem**: Knowledge graph for software specs ([github.com/c360/semmem](https://github.com/c360/semmem))
- **SemOps**: Operational picture with device tracking

## Related Documentation

**Query Strategies**:

- [PathRAG Guide](../guides/pathrag.md) - Graph traversal for dependencies
- [GraphRAG Guide](../guides/graphrag.md) - Semantic similarity search
- [Choosing Your Query Strategy](../guides/choosing-query-strategy.md) - Decision tree

**System Guides**:

- [Message System](../guides/message-system.md) - How data flows through SemStreams
- [Vocabulary System](../guides/vocabulary-system.md) - Semantic predicates

**Integration**:

- [REST API](./rest-api.md) - HTTP/JSON endpoints for simple queries
- [NATS Events](./nats-events.md) - Event-driven integration

**Architecture**:

- [Architecture Overview](../getting-started/architecture.md) - High-level system design

---

**Implementation**: See [semstreams](https://github.com/c360/semstreams) repository

- Gateway Component: `gateway/graphql/`
- Code Generation: Uses [gqlgen](https://gqlgen.com/)
- NATS Integration: `gateway/graphql/resolver.go`
