# GraphQL Gateway Abstraction - Architectural Analysis

**Author**: Claude (Architect)  
**Date**: 2025-11-16  
**Status**: Investigation Complete  
**Decision**: Recommendation Provided

## Executive Summary

**Question**: Can we create a generic GraphQL gateway abstraction based on SemMem's domain-specific GraphQL implementation?

**Answer**: **Partial Yes** - A hybrid approach is recommended. GraphQL query infrastructure can be generic, but resolver logic requires domain-specific code generation. The value proposition is strong for agentic workflows, but full abstraction faces fundamental GraphQL architecture constraints.

**Recommendation**: Build a **GraphQL Gateway Component** with:
1. Generic infrastructure (server, subscriptions, NATS integration)
2. Schema-driven code generation (gqlgen-based)
3. Declarative NATS subject mapping
4. Domain-specific resolver implementations (generated, not hand-coded)

## Background

### Problem Statement

Agentic workflows suffer from under/over-fetch problems when querying data stores:
- **Over-fetch**: REST endpoints return entire entities when agents need 2-3 fields
- **Under-fetch**: Agents make N+1 requests to assemble related entities
- **Token waste**: LLMs pay tokens for unused data in context windows

**Alternative considered**: Agent code execution (rejected as slow/complex)

**GraphQL solution**: Agents craft precise queries for exactly the data they need.

### Current Implementations

#### 1. SemMem GraphQL Server (Domain-Specific)

**Location**: `/semmem/output/graphql/`

**Schema**: 580 lines, 7 entity types (Spec, Doc, Issue, Discussion, PullRequest, Decision, PRReview)

**Pattern**:
- gqlgen code generation from `schema.graphql`
- Hand-written resolvers query GraphProcessor via NATS
- Type-specific NATS subjects (`semmem.graph.query.entity`)
- Relationship resolution via edge traversal
- Semantic search integration

**Key Files**:
- `schema.graphql` - Domain model (Specs, Docs, Issues, GitHub entities)
- `server.go` - HTTP server setup, gqlgen integration
- `resolver.go` - NATS client wrapper, base query functions
- `schema.resolvers.go` - 3156 lines of domain-specific resolver implementations
- `helpers.go` - Property extraction utilities

#### 2. HTTP Gateway (Generic)

**Location**: `/semstreams/gateway/http/`

**Pattern**: Configuration-driven route → NATS subject mapping

**Configuration**:
```json
{
  "routes": [
    {
      "path": "/search/semantic",
      "method": "POST",
      "nats_subject": "graph.query.semantic"
    }
  ]
}
```

**Characteristics**:
- Zero code generation
- Pure request forwarding
- No query language (REST endpoints)
- No relationship resolution
- Simple 1:1 mapping

## Analysis

### What's Domain-Specific in SemMem GraphQL?

#### 1. Schema Definition (100% Domain-Specific)

```graphql
type Spec {
  id: ID!
  title: String!
  status: String!
  version: String!
  content: String!
  dependencies: [Spec!]!      # Domain relationship
  implementedBy: [PullRequest!]!  # Domain relationship
}
```

**Why it can't be generic**: GraphQL requires explicit type definitions. There's no "generic entity" pattern that preserves GraphQL's type safety benefits.

#### 2. Resolver Implementations (95% Domain-Specific)

**Pattern in `schema.resolvers.go`**:
```go
func (r *queryResolver) Spec(ctx context.Context, id string) (*Spec, error) {
    // 1. Query NATS for entity (GENERIC)
    resp, err := r.natsClient.Request("semmem.graph.query.entity", data, 5*time.Second)
    
    // 2. Parse response (GENERIC)
    var entityState graph.EntityState
    json.Unmarshal(resp.Data, &entityState)
    
    // 3. Type checking (DOMAIN-SPECIFIC)
    if entityState.Node.Type != "org.semmem.spec" {
        return nil, errors.New("wrong type")
    }
    
    // 4. Map to GraphQL model (DOMAIN-SPECIFIC)
    spec := &Spec{
        ID:      entityState.Node.ID,
        Title:   getStringProp(entityState.Node.Properties, "title"),
        Status:  getStringProp(entityState.Node.Properties, "status"),
        Version: getStringProp(entityState.Node.Properties, "version"),
    }
    
    return spec, nil
}
```

**Domain-specific aspects**:
- Type discrimination (`org.semmem.spec`)
- Property mapping (`title`, `status`, `version` fields)
- Relationship edge types (`dependencies`, `implementedBy`)

**Generic aspects**:
- NATS request/reply pattern
- JSON marshaling/unmarshaling
- Property extraction helpers (`getStringProp`, `getIntProp`)

#### 3. Relationship Resolution (80% Domain-Specific)

**Field Resolver Pattern**:
```go
func (r *specResolver) Dependencies(ctx context.Context, obj *Spec) ([]*Spec, error) {
    // GENERIC: Query relationships by edge type
    targetIDs, err := r.queryRelationshipsByType(ctx, obj.ID, "depends_on", "outgoing")
    
    // GENERIC: Batch load entities
    entities, err := r.queryEntitiesByIDs(ctx, targetIDs)
    
    // DOMAIN-SPECIFIC: Convert to Spec type
    specs := make([]*Spec, 0, len(entities))
    for _, entity := range entities {
        specs = append(specs, convertToSpec(entity))
    }
    return specs, nil
}
```

**Domain-specific**:
- Edge type name (`depends_on`)
- Target entity type (Spec)
- Conversion function

**Generic**:
- Graph traversal pattern
- Batch loading optimization
- Direction filtering

#### 4. Search Integration (60% Generic)

**Semantic Search Resolver**:
```go
func (r *queryResolver) SearchSpecs(ctx context.Context, query string, limit *int) ([]*Spec, error) {
    // GENERIC: Semantic search via NATS
    results, err := r.querySemanticSearch(ctx, query, []string{"org.semmem.spec"}, limit)
    
    // GENERIC: Batch load full entities
    entities, err := r.queryEntitiesByIDs(ctx, entityIDs)
    
    // DOMAIN-SPECIFIC: Convert to Spec models
    return convertToSpecs(entities), nil
}
```

**Generic pattern**: Search → Extract IDs → Batch Load → Convert  
**Domain-specific**: Entity type filter, conversion logic

### What Could Be Generic?

#### 1. Infrastructure Layer (100% Generic)

**Components**:
- HTTP server setup (gqlgen handler)
- GraphQL Playground integration
- Subscription management (WebSocket)
- Context propagation
- Error handling
- CORS configuration
- Health checks

**Already implemented in**: `server.go` (mostly generic)

#### 2. NATS Integration (100% Generic)

**Reusable patterns**:
```go
// Generic NATS request helper
func queryNATS(subject string, payload interface{}) (json.RawMessage, error)

// Generic batch loading
func batchLoadEntities(ids []string) ([]*EntityState, error)

// Generic relationship traversal
func queryRelationships(entityID, edgeType, direction string) ([]string, error)

// Generic semantic search
func semanticSearch(query string, types []string, limit int) (SearchResults, error)
```

**Already extracted in**: `resolver.go`

#### 3. Property Extraction (100% Generic)

**Helper utilities** (from `helpers.go`):
```go
func getStringProp(props map[string]any, key string) string
func getIntProp(props map[string]any, key string) int
func getBoolProp(props map[string]any, key string) bool
func getStringArrayProp(props map[string]any, key string) []string
```

Already domain-agnostic.

#### 4. Configuration Format (80% Generic)

**Proposed**:
```json
{
  "graphql-gateway": {
    "type": "gateway",
    "name": "graphql",
    "config": {
      "schema_file": "./schema.graphql",
      "nats_subjects": {
        "entity_query": "domain.graph.query.entity",
        "predicate_query": "domain.graph.query.predicate",
        "relationships_query": "domain.graph.query.relationships",
        "semantic_search": "domain.graph.query.semantic"
      },
      "type_mappings": [
        {
          "graphql_type": "Spec",
          "entity_type": "org.domain.spec",
          "id_prefix": "org.domain.spec."
        }
      ]
    }
  }
}
```

**Generic**: Subject mapping pattern  
**Domain-specific**: Schema file, type mappings

## Proposed Architecture

### Hybrid Approach: Generic Infrastructure + Generated Resolvers

```
┌─────────────────────────────────────────────────────────────┐
│                 GraphQL Gateway Component                    │
│                      (GENERIC)                               │
├─────────────────────────────────────────────────────────────┤
│  • HTTP Server (gqlgen)                                      │
│  • Subscription Manager                                      │
│  • NATS Client Integration                                   │
│  • Health/Metrics                                            │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│              Resolver Generator (gqlgen)                     │
│                 (CODE GENERATION)                            │
├─────────────────────────────────────────────────────────────┤
│  Input:  schema.graphql + gqlgen.yml + config.json          │
│  Output: schema.resolvers.go (domain-specific)               │
│                                                              │
│  Templates:                                                  │
│  • Query resolver → NATS entity query                        │
│  • List resolver → NATS predicate query + filter             │
│  • Search resolver → NATS semantic search                    │
│  • Field resolver → NATS relationship query                  │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│           Generic Resolver Base (REUSABLE)                   │
├─────────────────────────────────────────────────────────────┤
│  • queryEntityByID(subject, id) → EntityState                │
│  • queryEntitiesByType(subject, type) → []EntityState        │
│  • queryRelationships(subject, id, edge, dir) → []string     │
│  • semanticSearch(subject, query, types) → SearchResults     │
│  • batchLoadEntities(ids) → []EntityState                    │
│  • convertProperties(entity, mapping) → GraphQLModel         │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
                   NATS Bus
                      │
┌─────────────────────┴───────────────────────────────────────┐
│              GraphProcessor Component                        │
│         (Domain-agnostic query engine)                       │
└─────────────────────────────────────────────────────────────┘
```

### Configuration Example

**1. GraphQL Schema** (`schema.graphql`):
```graphql
type Robot {
  id: ID!
  name: String!
  location: GeoPoint!
  sensors: [Sensor!]!
  missions: [Mission!]!
}

type Mission {
  id: ID!
  status: MissionStatus!
  robot: Robot!
}
```

**2. Gateway Config** (`gateway-config.json`):
```json
{
  "schema_file": "./schema.graphql",
  "nats_subjects": {
    "entity": "robotics.graph.query.entity",
    "predicate": "robotics.graph.query.predicate",
    "relationships": "robotics.graph.query.relationships",
    "semantic": "robotics.graph.query.semantic"
  },
  "type_mappings": {
    "Robot": {
      "entity_type": "org.robotics.robot",
      "id_prefix": "robot."
    },
    "Mission": {
      "entity_type": "org.robotics.mission",
      "id_prefix": "mission."
    }
  },
  "edge_mappings": {
    "Robot.sensors": "has_sensor",
    "Robot.missions": "assigned_to",
    "Mission.robot": "assigned_to"
  }
}
```

**3. gqlgen Config** (`gqlgen.yml`):
```yaml
schema:
  - schema.graphql

exec:
  filename: generated.go

model:
  filename: models_gen.go

resolver:
  filename: resolver.go
  type: Resolver

# Use plugin to generate NATS-backed resolvers
plugins:
  - name: semstreams-resolver
    config:
      base_resolver: github.com/c360/semstreams/gateway/graphql/base.Resolver
      config_file: gateway-config.json
```

### Generated Resolver Example

**Input** (schema.graphql):
```graphql
type Query {
  robot(id: ID!): Robot
  robots(status: String, limit: Int): [Robot!]!
  searchRobots(query: String!, limit: Int): [Robot!]!
}

type Robot {
  sensors: [Sensor!]!
}
```

**Generated** (schema.resolvers.go):
```go
// Robot query resolver (GENERATED)
func (r *queryResolver) Robot(ctx context.Context, id string) (*Robot, error) {
    // Use generic base query
    entity, err := r.base.QueryEntity(ctx, r.config.NATSSubjects.Entity, id)
    if err != nil {
        return nil, err
    }
    
    // Validate type
    expectedType := r.config.TypeMappings["Robot"].EntityType
    if entity.Node.Type != expectedType {
        return nil, fmt.Errorf("type mismatch: expected %s", expectedType)
    }
    
    // Convert using mapping
    return r.base.ConvertToRobot(entity), nil
}

// Robot list resolver (GENERATED)
func (r *queryResolver) Robots(ctx context.Context, status *string, limit *int) ([]*Robot, error) {
    entityType := r.config.TypeMappings["Robot"].EntityType
    entities, err := r.base.QueryEntitiesByType(ctx, r.config.NATSSubjects.Predicate, entityType, *limit)
    
    // Apply filters
    filtered := r.base.FilterByProperty(entities, "status", status)
    
    // Convert
    return r.base.ConvertToRobots(filtered), nil
}

// Robot field resolver (GENERATED)
func (r *robotResolver) Sensors(ctx context.Context, obj *Robot) ([]*Sensor, error) {
    edgeType := r.config.EdgeMappings["Robot.sensors"]
    targetIDs, err := r.base.QueryRelationships(ctx, obj.ID, edgeType, "outgoing")
    
    entities, err := r.base.BatchLoadEntities(ctx, targetIDs)
    return r.base.ConvertToSensors(entities), nil
}
```

### Generic Base Resolver (Reusable)

**Package**: `semstreams/gateway/graphql/base`

```go
package base

type Config struct {
    NATSSubjects struct {
        Entity        string
        Predicate     string
        Relationships string
        Semantic      string
    }
    TypeMappings  map[string]TypeMapping
    EdgeMappings  map[string]string
}

type TypeMapping struct {
    EntityType string
    IDPrefix   string
}

type Resolver struct {
    natsClient *natsclient.Client
    config     Config
}

// Generic query methods
func (r *Resolver) QueryEntity(ctx context.Context, subject, id string) (*graph.EntityState, error)
func (r *Resolver) QueryEntitiesByType(ctx context.Context, subject, entityType string, limit int) ([]*graph.EntityState, error)
func (r *Resolver) QueryRelationships(ctx context.Context, entityID, edgeType, direction string) ([]string, error)
func (r *Resolver) SemanticSearch(ctx context.Context, query string, types []string, limit int) (*SearchResults, error)
func (r *Resolver) BatchLoadEntities(ctx context.Context, ids []string) ([]*graph.EntityState, error)

// Generic filtering
func (r *Resolver) FilterByProperty(entities []*graph.EntityState, key string, value *string) []*graph.EntityState

// Conversion helpers (generated per domain)
// func (r *Resolver) ConvertToRobot(entity *graph.EntityState) *Robot
// func (r *Resolver) ConvertToMission(entity *graph.EntityState) *Mission
```

## Benefits Analysis

### For Agentic Workflows

#### Token Efficiency

**Before** (REST API):
```bash
# Agent needs: robot name + location
GET /robots/robot-42
Response: 5KB JSON (name, location, sensors[], missions[], telemetry{}, config{}, ...)
# Wasted: 4.5KB unused data in LLM context
```

**After** (GraphQL):
```graphql
query {
  robot(id: "robot-42") {
    name
    location { lat, lon }
  }
}
Response: 50 bytes JSON
# Savings: 99% reduction in token consumption
```

**Impact**: For an agent making 100 queries, this saves ~450KB = ~112K tokens = $0.56 (GPT-4) per workflow.

#### Relationship Efficiency

**Before** (N+1 queries):
```bash
GET /robots?status=active  # 1 query → 10 robots
GET /missions?robot=robot-1  # 10 queries for missions
GET /sensors/sensor-xyz    # 50 queries for sensor details
# Total: 61 HTTP requests
```

**After** (1 GraphQL query):
```graphql
query {
  robots(status: "active") {
    name
    missions { id, status }
    sensors { type, reading }
  }
}
# Total: 1 GraphQL query → 3 NATS queries (robots, missions batch, sensors batch)
```

**Impact**: Reduces round-trips by 95%, latency from ~3s to ~200ms.

#### Semantic Precision

**GraphQL enables**:
```graphql
query FindSimilarMissions {
  searchMissions(query: "autonomous navigation in warehouse") {
    id
    status
    robot { name }
    # Agent gets exactly what it needs for decision-making
  }
}
```

### For Development Teams

#### Code Reusability

**Generic components** (write once):
- NATS integration layer
- Subscription manager
- Property extraction utilities
- Batch loading optimization
- Error handling patterns

**Domain-specific** (generated):
- Type conversions
- Resolver implementations
- GraphQL models

**Savings**: 80% code reduction vs hand-writing resolvers.

#### Type Safety

GraphQL schema serves as API contract:
```graphql
type Robot {
  location: GeoPoint!  # Compiler enforces non-null
  sensors: [Sensor!]!  # Compiler enforces array type
}
```

**Benefits**:
- Compile-time errors vs runtime failures
- Auto-generated TypeScript types for frontend
- API documentation from schema

#### Discoverability

GraphQL introspection:
```graphql
query {
  __schema {
    types { name, fields { name, type } }
  }
}
```

**Enables**:
- Auto-generated documentation
- GraphQL Playground exploration
- IDE autocomplete for agents
- Schema validation tools

## Costs and Constraints

### Development Complexity

**Initial Setup**:
1. Create `gateway/graphql` package structure
2. Implement generic `BaseResolver`
3. Build gqlgen plugin for code generation
4. Write configuration schema
5. Create migration guide from HTTP gateway

**Estimate**: 2-3 weeks for senior Go developer.

### Fundamental GraphQL Limitations

#### 1. No True Polymorphism

**Problem**: GraphQL unions are awkward:
```graphql
union Entity = Robot | Mission | Sensor

type Query {
  entity(id: ID!): Entity  # Works
}

# But field resolution is clunky:
query {
  entity(id: "robot-42") {
    ... on Robot { name }
    ... on Mission { status }
  }
}
```

**HTTP Gateway equivalent**:
```json
{
  "path": "/entity/:id",
  "nats_subject": "graph.query.entity"
}
```
Returns generic JSON, no type constraints.

**Trade-off**: GraphQL requires type-specific handling, loses some flexibility.

#### 2. Schema Evolution

**GraphQL**: Breaking changes require coordination
```graphql
type Robot {
  location: GeoPoint!  # Can't remove without breaking clients
}
```

**HTTP Gateway**: JSON evolution is more forgiving
```json
{ "location": {...} }  # Clients ignore unknown fields
```

**Mitigation**: Use GraphQL deprecation + versioning.

#### 3. Complexity for Simple Queries

**For single-entity lookup**:
```graphql
query { robot(id: "robot-42") { name } }
```

vs HTTP:
```
GET /robots/robot-42
```

GraphQL adds overhead (query parsing, resolver execution) for minimal benefit.

**When GraphQL shines**: Complex queries with relationships.

### Operational Overhead

**GraphQL Playground**: Requires UI assets (~2MB)  
**Query Complexity Analysis**: Prevents denial-of-service attacks  
**Subscription Management**: WebSocket connections consume resources

**Recommendation**: Deploy GraphQL gateway separately from core processor components.

## Comparison Matrix

| Aspect | Domain-Specific GraphQL | Generic GraphQL Gateway | HTTP Gateway |
|--------|-------------------------|------------------------|--------------|
| **Code Generation** | Hand-written (3156 lines) | gqlgen + templates | None |
| **Configuration** | Schema file | Schema + NATS mappings | Route mappings |
| **Type Safety** | ✅ Full | ✅ Full | ❌ None |
| **Relationship Queries** | ✅ Native | ✅ Native | ❌ N+1 problem |
| **Token Efficiency** | ✅ Excellent | ✅ Excellent | ❌ Over-fetch |
| **Flexibility** | ❌ Rigid schema | ❌ Rigid schema | ✅ Dynamic |
| **Setup Complexity** | High | Medium | Low |
| **Runtime Overhead** | Medium | Medium | Low |
| **Agent Friendliness** | ✅ Excellent | ✅ Excellent | ⚠️ Fair |

## Decision Framework

### When to Use GraphQL Gateway

✅ **Strong Fit**:
- Agentic workflows (LLM-driven queries)
- Complex relationship graphs
- Frontend applications (React, Vue, Svelte)
- Mobile apps (bandwidth-constrained)
- Multi-client API (web, mobile, agents)

❌ **Weak Fit**:
- Simple CRUD operations
- Internal service-to-service (use NATS directly)
- Real-time telemetry ingestion (use Input components)
- Admin/control plane (HTTP gateway sufficient)

### When to Use HTTP Gateway

✅ **Strong Fit**:
- Simple request/reply
- Legacy system integration
- Webhooks/callbacks
- Admin operations
- Health checks

## Recommended Implementation Plan

### Phase 1: Generic Infrastructure (Week 1-2)

**Deliverables**:
```
semstreams/gateway/graphql/
├── base/
│   ├── resolver.go         # Generic BaseResolver
│   ├── nats.go             # NATS query helpers
│   ├── types.go            # Config types
│   └── helpers.go          # Property extraction
├── plugin/
│   └── semstreams.go       # gqlgen plugin
├── server.go               # HTTP server setup
├── config.go               # Configuration schema
└── doc.go                  # Package documentation
```

**Tests**:
- Unit tests for BaseResolver
- Integration tests with testcontainers NATS
- Example domain (robotics) for validation

### Phase 2: Code Generation (Week 3)

**Deliverables**:
- gqlgen plugin for resolver generation
- Template engine for type conversions
- Configuration validator
- Migration guide from hand-written resolvers

**Validation**:
- Port SemMem GraphQL to use generic gateway
- Compare generated vs hand-written code
- Benchmark performance (should be identical)

### Phase 3: Documentation & Examples (Week 4)

**Deliverables**:
- `docs/architecture/GRAPHQL_GATEWAY.md`
- Example: Robotics domain GraphQL gateway
- Configuration reference
- Best practices guide
- Agent integration examples (LangChain, AutoGPT)

### Phase 4: Production Hardening (Week 5-6)

**Deliverables**:
- Query complexity analysis (prevent DoS)
- Rate limiting integration
- Monitoring/metrics
- Subscription cleanup on disconnect
- Error handling improvements
- Performance optimization (DataLoader pattern)

## Final Recommendation

**Build the GraphQL Gateway Component**: ✅ Yes, with caveats

**Approach**: Hybrid (generic infrastructure + generated domain code)

**Rationale**:

1. **Agentic Value**: GraphQL's precision querying solves real token waste problems for LLM agents.

2. **Reusability**: 80% of code can be generic (NATS integration, server setup, property extraction).

3. **Maintainability**: Code generation reduces hand-written boilerplate from 3000+ lines to ~200 lines of config.

4. **Pragmatism**: Acknowledges that GraphQL schemas are inherently domain-specific, so we generate rather than abstract.

5. **Flexibility**: Teams can hand-customize generated resolvers for complex edge cases.

**Key Insight**: Don't try to make GraphQL "generic" like HTTP gateway. Instead, make the *scaffolding* generic and generate domain-specific resolvers from declarative config.

**Success Metric**: New domain GraphQL gateway setup time < 1 hour (vs 2+ weeks hand-coding).

## Open Questions

1. **Federation**: Should we support GraphQL federation for multi-domain graphs?
2. **Caching**: Integrate with Redis/Memcached for query result caching?
3. **Batching**: Implement DataLoader pattern in base resolver?
4. **Mutations**: Support write operations or keep read-only like HTTP gateway?
5. **Versioning**: Schema versioning strategy for breaking changes?

## References

- SemMem GraphQL: `/semmem/output/graphql/`
- HTTP Gateway: `/semstreams/gateway/http/`
- GraphQL Spec: https://spec.graphql.org/
- gqlgen Documentation: https://gqlgen.com/
- GraphQL Best Practices: https://graphql.org/learn/best-practices/
