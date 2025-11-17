# GraphQL API

SemStreams exposes a GraphQL API for querying the knowledge graph, managing flows, and accessing semantic search.

## Endpoint

```
http://localhost:8080/graphql
```

**GraphQL Playground**: http://localhost:8080/graphql (in development mode)

## Authentication

Currently no authentication required (development mode). Production deployments should configure authentication via reverse proxy or API gateway.

## Core Schema

### Entities

**Query entities from the knowledge graph:**

```graphql
type Query {
  # Get entity by ID
  entity(id: ID!): Entity

  # List entities with filtering
  entities(
    type: String
    limit: Int = 10
    offset: Int = 0
  ): EntityConnection!

  # Semantic search
  searchEntities(
    query: String!
    limit: Int = 10
    threshold: Float = 0.3
  ): [SearchResult!]!
}

type Entity {
  id: ID!
  type: String!
  properties: JSON!
  edges: [Edge!]!
  timestamp: DateTime!
}

type Edge {
  predicate: String!
  target: ID!
}

type SearchResult {
  entity: Entity!
  score: Float!
  snippet: String
}
```

### Flows

**Manage processing flows:**

```graphql
type Query {
  # Get flow by ID
  flow(id: ID!): Flow

  # List all flows
  flows: [Flow!]!
}

type Mutation {
  # Create new flow
  createFlow(input: FlowInput!): Flow!

  # Update existing flow
  updateFlow(id: ID!, input: FlowInput!): Flow!

  # Delete flow
  deleteFlow(id: ID!): Boolean!

  # Start/stop flow
  startFlow(id: ID!): Flow!
  stopFlow(id: ID!): Flow!
}

type Flow {
  id: ID!
  name: String!
  description: String
  components: [Component!]!
  connections: [Connection!]!
  status: FlowStatus!
}

enum FlowStatus {
  STOPPED
  RUNNING
  ERROR
}
```

## Example Queries

### Get Entity with Relationships

```graphql
query GetSensor {
  entity(id: "sensor-123") {
    id
    type
    properties
    edges {
      predicate
      target
    }
    timestamp
  }
}
```

### Semantic Search

```graphql
query SearchDevices {
  searchEntities(
    query: "temperature sensor building A"
    limit: 5
    threshold: 0.5
  ) {
    score
    snippet
    entity {
      id
      type
      properties
    }
  }
}
```

### List Flows

```graphql
query AllFlows {
  flows {
    id
    name
    status
    components {
      id
      type
      config
    }
  }
}
```

### Create Flow

```graphql
mutation CreateProcessingFlow {
  createFlow(input: {
    name: "Sensor Processing"
    description: "Process temperature sensor data"
    components: [
      {
        id: "input-1"
        type: "UDPInput"
        config: {
          port: 14550
          bind: "0.0.0.0"
        }
      }
      {
        id: "parser-1"
        type: "JSONParser"
      }
      {
        id: "output-1"
        type: "NATSOutput"
        config: {
          subject: "sensors.temperature"
        }
      }
    ]
    connections: [
      { from: "input-1", to: "parser-1" }
      { from: "parser-1", to: "output-1" }
    ]
  }) {
    id
    name
    status
  }
}
```

## Subscriptions

**Real-time updates via GraphQL subscriptions:**

```graphql
type Subscription {
  # Subscribe to entity changes
  entityUpdated(type: String): Entity!

  # Subscribe to flow status changes
  flowStatusChanged(flowId: ID): Flow!
}
```

### Example Subscription

```graphql
subscription WatchSensors {
  entityUpdated(type: "Sensor") {
    id
    type
    properties
    timestamp
  }
}
```

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

## See Also

- [REST API](./rest-api.md) - HTTP/JSON endpoints
- [NATS Events](./nats-events.md) - Event-driven integration
- [Deployment Guide](../deployment/production.md) - Production setup
