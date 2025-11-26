# Entities

## Understanding entities: the nodes in your semantic graph

---

## What are Entities?

Entities are the **nodes** in your knowledge graph. Each entity represents something that exists and has properties expressed as semantic triples.

**Examples (6-part federated IDs):**

- Drones: `acme.telemetry.robotics.gcs1.drone.001`
- Fleets: `acme.ops.logistics.hq.fleet.rescue`
- Sensors: `acme.telemetry.sensors.building1.temperature.42`
- Users: `acme.platform.auth.main.user.alice`

---

## How Entities are Created

Entities come from payloads that implement the **Graphable interface**.

```go
type Graphable interface {
    // EntityID returns deterministic 6-part ID: org.platform.domain.system.type.instance
    EntityID() string

    // Triples returns all facts about this entity
    Triples() []Triple
}
```

**The graph processor only accepts payloads implementing Graphable.** This is not optional.

### Example: Drone Payload

```go
type DronePayload struct {
    SystemID  uint8   `json:"system_id"`
    Battery   float64 `json:"battery"`
    Latitude  float64 `json:"latitude"`
    Longitude float64 `json:"longitude"`
    Fleet     string  `json:"fleet"`
}

func (d *DronePayload) EntityID() string {
    // 6-part format: org.platform.domain.system.type.instance
    return fmt.Sprintf("acme.telemetry.robotics.gcs1.drone.%d", d.SystemID)
}

func (d *DronePayload) Triples() []Triple {
    entityID := d.EntityID()
    fleetID := fmt.Sprintf("acme.ops.logistics.hq.fleet.%s", d.Fleet)
    now := time.Now()

    return []Triple{
        {
            Subject:    entityID,
            Predicate:  "rdf:type",
            Object:     "robotics:Drone",
            Source:     "system",
            Timestamp:  now,
            Confidence: 1.0,
        },
        {
            Subject:    entityID,
            Predicate:  "robotics.battery.level",
            Object:     d.Battery,
            Source:     "mavlink_battery",
            Timestamp:  now,
            Confidence: 1.0,
        },
        {
            Subject:    entityID,
            Predicate:  "geo.location.latitude",
            Object:     d.Latitude,
            Source:     "gps",
            Timestamp:  now,
            Confidence: 0.95,
        },
        {
            Subject:    entityID,
            Predicate:  "geo.location.longitude",
            Object:     d.Longitude,
            Source:     "gps",
            Timestamp:  now,
            Confidence: 0.95,
        },
        {
            Subject:    entityID,
            Predicate:  "ops.fleet.member_of",
            Object:     fleetID,  // Relationship: Object is another EntityID
            Source:     "fleet_assignment",
            Timestamp:  now,
            Confidence: 1.0,
        },
    }
}
```

---

## Entity ID Format

Entity IDs use a **6-part federated format**:

```text
org.platform.domain.system.type.instance
```

| Part | Description | Example |
|------|-------------|---------|
| org | Organization namespace | `acme` |
| platform | Platform/deployment | `telemetry` |
| domain | Data domain | `robotics` |
| system | System identifier | `gcs1` |
| type | Entity type | `drone` |
| instance | Unique instance | `001` |

**Examples:**

```text
acme.telemetry.robotics.gcs1.drone.001
acme.ops.logistics.hq.fleet.rescue
acme.platform.auth.main.user.alice
```

**Why 6-part IDs?**

- **Globally unique** across federated deployments
- **Hierarchical** for pattern matching (e.g., `*.*.robotics.*.*.*`)
- **Self-documenting** - IDs explain what and where
- **Collision-free** without coordination

---

## Entity State Structure

When stored in the graph, entities become `EntityState`:

```go
type EntityState struct {
    Node      NodeProperties   `json:"node"`       // Entity properties
    Edges     []Edge           `json:"edges"`      // Outgoing edges
    Triples   []Triple         `json:"triples"`    // Semantic facts
    ObjectRef string           `json:"object_ref"` // Reference to full message
    Version   uint64           `json:"version"`    // For conflict resolution
    UpdatedAt time.Time        `json:"updated_at"`
}

type NodeProperties struct {
    ID         string         `json:"id"`         // 6-part entity ID
    Type       string         `json:"type"`       // e.g., "robotics.drone"
    Properties map[string]any `json:"properties"` // Query-essential only
    Position   *Position      `json:"position,omitempty"`
    Status     EntityStatus   `json:"status"`
}
```

**Stored in NATS KV:**

```json
{
  "node": {
    "id": "acme.telemetry.robotics.gcs1.drone.001",
    "type": "robotics.drone",
    "properties": {
      "robotics.battery.level": 85.2,
      "ops.fleet.member_of": "rescue"
    },
    "position": {
      "lat": 37.7749,
      "lon": -122.4194
    },
    "status": "active"
  },
  "edges": [
    {
      "to_entity_id": "acme.ops.logistics.hq.fleet.rescue",
      "edge_type": "MEMBER_OF",
      "weight": 1.0,
      "confidence": 1.0,
      "created_at": "2024-01-15T10:00:00Z"
    }
  ],
  "triples": [
    {"subject": "acme.telemetry.robotics.gcs1.drone.001", "predicate": "rdf:type", "object": "robotics:Drone"},
    {"subject": "acme.telemetry.robotics.gcs1.drone.001", "predicate": "robotics.battery.level", "object": 85.2}
  ],
  "object_ref": "objects/drone/001/2024-01-15T10:30:00Z",
  "version": 42,
  "updated_at": "2024-01-15T10:30:00Z"
}
```

---

## Triples: Semantic Facts

Triples are RDF*-like facts with rich metadata: **Subject-Predicate-Object + Provenance**.

```go
type Triple struct {
    // Core SPO (Subject-Predicate-Object)
    Subject   string `json:"subject"`   // EntityID (6-part federated format)
    Predicate string `json:"predicate"` // Dotted notation (domain.category.property)
    Object    any    `json:"object"`    // Value or EntityID (relationship)

    // RDF*-like metadata
    Source     string    `json:"source"`              // Provenance ("mavlink", "operator", "ai_inference")
    Timestamp  time.Time `json:"timestamp"`           // When assertion was made
    Confidence float64   `json:"confidence"`          // 0.0 to 1.0 reliability score
    Context    string    `json:"context,omitempty"`   // Correlation/batch ID
    Datatype   string    `json:"datatype,omitempty"`  // RDF datatype hint ("xsd:float", "geo:point")
}
```

### Triple Types

**Property triples** (entity has value):

```go
{
    Subject:    "acme.telemetry.robotics.gcs1.drone.001",
    Predicate:  "robotics.battery.level",
    Object:     85.2,
    Source:     "mavlink_battery",
    Timestamp:  time.Now(),
    Confidence: 1.0,  // Direct telemetry = high confidence
}
```

**Relationship triples** (entity relates to entity):

```go
{
    Subject:    "acme.telemetry.robotics.gcs1.drone.001",
    Predicate:  "ops.fleet.member_of",
    Object:     "acme.ops.logistics.hq.fleet.rescue",  // Another entity ID
    Source:     "fleet_assignment",
    Timestamp:  time.Now(),
    Confidence: 1.0,
}
```

**Type triples** (entity classification):

```go
{
    Subject:    "acme.telemetry.robotics.gcs1.drone.001",
    Predicate:  "rdf:type",
    Object:     "robotics:Drone",
    Source:     "system",
    Timestamp:  time.Now(),
    Confidence: 1.0,
}
```

**Alias triples** (alternative identifiers):

```go
{
    Subject:    "acme.telemetry.robotics.gcs1.drone.001",
    Predicate:  "robotics.communication.callsign",  // Registered as alias in vocabulary
    Object:     "rescue-alpha",
    Source:     "operator_input",
    Timestamp:  time.Now(),
    Confidence: 1.0,
}
```

Alias predicates must be registered in the vocabulary registry with `WithAlias()` to be indexed. See [Vocabulary Registry](../basics/06-vocabulary-registry.md).

### Confidence Levels

| Level | Meaning | Example |
|-------|---------|---------|
| 1.0 | Direct telemetry or explicit data | Sensor readings, operator input |
| 0.9 | High-confidence sensor readings | GPS with good signal |
| 0.7 | Calculated or derived values | Estimated time of arrival |
| 0.5 | Inferred relationships | AI-detected proximity |
| 0.0 | Uncertain or placeholder | Default values |

### Relationship Detection

The system determines if a triple is a relationship (vs property) using `IsRelationship()`:

```go
// Returns true if Object is a valid 6-part EntityID
triple.IsRelationship()
```

Relationship triples are indexed in INCOMING_INDEX for reverse traversal.

---

## Edges: Outgoing Relationships

Edges represent explicit relationships between entities:

```go
type Edge struct {
    ToEntityID string         `json:"to_entity_id"`         // Target entity
    EdgeType   string         `json:"edge_type"`            // "MEMBER_OF", "POWERED_BY", etc.
    Weight     float64        `json:"weight,omitempty"`     // Distance, strength
    Confidence float64        `json:"confidence,omitempty"` // 0.0-1.0
    Properties map[string]any `json:"properties,omitempty"`
    CreatedAt  time.Time      `json:"created_at"`
    ExpiresAt  *time.Time     `json:"expires_at,omitempty"` // For temporary relationships
}
```

**Example edges:**

```json
[
  {
    "to_entity_id": "acme.ops.logistics.hq.fleet.rescue",
    "edge_type": "MEMBER_OF",
    "weight": 1.0,
    "confidence": 1.0,
    "created_at": "2024-01-15T10:00:00Z"
  },
  {
    "to_entity_id": "acme.telemetry.robotics.gcs1.drone.002",
    "edge_type": "NEAR",
    "weight": 150.5,
    "confidence": 0.95,
    "properties": {"distance_meters": 150.5},
    "created_at": "2024-01-15T10:30:00Z",
    "expires_at": "2024-01-15T10:35:00Z"
  }
]
```

**Edges vs Triples:**

- **Triples** are semantic facts from Graphable.Triples()
- **Edges** are computed relationships (proximity, graph algorithms)
- Both are queryable, but edges support weight/confidence/expiration

---

## Entity Updates

Entities are **upserted** - created if new, updated if exists.

**Update behavior:**

- **Node.ID** match → Update existing entity
- **Properties** merge (new fields added, existing updated)
- **Triples** replaced with new set from Graphable.Triples()
- **Version** incremented for conflict resolution
- **UpdatedAt** timestamp updated

---

## Pattern Matching

Entity IDs support **glob pattern matching** for rules and queries:

```text
*.*.robotics.*.drone.*       # All drones from any org/platform
acme.*.*.*.*.001            # Entity 001 from any domain in acme
acme.telemetry.robotics.*.*.*  # All robotics entities from telemetry platform
```

**Used in:**

- Rules engine entity patterns
- NATS KV bucket watches
- Query filters

---

## Best Practices

### Use 6-Part Federated IDs

```text
acme.telemetry.robotics.gcs1.drone.001
```

Not simple IDs like `UAV-001` or UUIDs like `a1b2c3d4-...`

### Implement Graphable Properly

```go
func (p *MyPayload) EntityID() string {
    // Deterministic - same input always produces same ID
    return fmt.Sprintf("myorg.platform.domain.system.type.%s", p.UniqueField)
}

func (p *MyPayload) Triples() []Triple {
    // Return all semantic facts about this entity
    // Use vocabulary predicates for consistency
}
```

### Use Vocabulary Predicates

Define predicates in the vocabulary registry for consistency:

```text
robotics.battery.level      # Not "battery" or "battery_level"
geo.location.latitude       # Not "lat" or "latitude"
ops.fleet.member_of         # Not "fleet" or "belongs_to"
```

See: [Vocabulary Registry](../basics/06-vocabulary-registry.md)

---

## Next Steps

- **[Relationships](02-relationships.md)** - Relationship patterns and traversal
- **[Queries](03-queries.md)** - Query the graph
- **[Indexing](04-indexing.md)** - Index configuration for performance
- **[Message System](../basics/05-message-system.md)** - Graphable interface details

---

**Key Takeaway:** Entities come from the Graphable interface. Implement `EntityID()` returning a 6-part federated ID and `Triples()` returning semantic facts. The graph processor handles the rest.
