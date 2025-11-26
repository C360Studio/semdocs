# Relationships

**Connecting entities: the edges in your semantic graph**

---

## What are Relationships?

Relationships connect entities, forming the graph structure. In SemStreams, relationships are expressed in two ways:

1. **Triples** - Semantic facts where the Object is another EntityID
2. **Edges** - Computed relationships with weight/confidence (proximity, algorithms)

**Example graph:**

```text
acme.telemetry.robotics.gcs1.drone.001 ──ops.fleet.member_of──> acme.ops.logistics.hq.fleet.rescue
acme.telemetry.robotics.gcs1.drone.002 ──ops.fleet.member_of──> acme.ops.logistics.hq.fleet.rescue
acme.telemetry.robotics.gcs1.drone.001 ──geo.location.stationed_at──> acme.ops.facilities.west.base.main
```

---

## Relationship Sources

### From Triples (Semantic Facts)

The primary way to express relationships. When implementing `Graphable.Triples()`, return triples where the Object is another entity's ID. Triples include RDF*-like metadata for provenance and confidence.

```go
func (d *DronePayload) Triples() []Triple {
    entityID := d.EntityID()
    fleetID := fmt.Sprintf("acme.ops.logistics.hq.fleet.%s", d.Fleet)
    now := time.Now()

    return []Triple{
        // Property triple (value object)
        {
            Subject:    entityID,
            Predicate:  "robotics.battery.level",
            Object:     d.Battery,
            Source:     "mavlink_battery",
            Timestamp:  now,
            Confidence: 1.0,
        },

        // Relationship triple (entity reference)
        {
            Subject:    entityID,
            Predicate:  "ops.fleet.member_of",
            Object:     fleetID,  // Another entity ID = relationship
            Source:     "fleet_assignment",
            Timestamp:  now,
            Confidence: 1.0,
        },

        // Relationship triple with lower confidence (inferred)
        {
            Subject:    entityID,
            Predicate:  "robotics.operator.controlled_by",
            Object:     d.OperatorID,
            Source:     "session_inference",
            Timestamp:  now,
            Confidence: 0.8,  // Inferred from active session
        },
    }
}
```

**Key insight:** SemStreams doesn't "infer" relationships from properties. You explicitly declare them in Triples(). The system determines relationship vs property using `triple.IsRelationship()` which checks if Object is a valid 6-part EntityID.

### From Edges (Computed Relationships)

Edges are stored on EntityState for computed relationships like proximity:

```go
type Edge struct {
    ToEntityID string         `json:"to_entity_id"`
    EdgeType   string         `json:"edge_type"`            // "NEAR", "POWERED_BY", etc.
    Weight     float64        `json:"weight,omitempty"`     // Distance, strength
    Confidence float64        `json:"confidence,omitempty"` // 0.0-1.0
    Properties map[string]any `json:"properties,omitempty"`
    CreatedAt  time.Time      `json:"created_at"`
    ExpiresAt  *time.Time     `json:"expires_at,omitempty"` // Temporary relationships
}
```

**Example edge (proximity):**

```json
{
  "to_entity_id": "acme.telemetry.robotics.gcs1.drone.002",
  "edge_type": "NEAR",
  "weight": 150.5,
  "confidence": 0.95,
  "properties": {"distance_meters": 150.5},
  "created_at": "2024-01-15T10:30:00Z",
  "expires_at": "2024-01-15T10:35:00Z"
}
```

---

## Predicate Types

Use vocabulary registry predicates (dotted notation):

### Hierarchical (membership, containment)

```go
{Subject: droneID, Predicate: "ops.fleet.member_of", Object: fleetID}
{Subject: sensorID, Predicate: "robotics.component.part_of", Object: droneID}
```

**Query pattern:** "Find all drones in fleet X"

### Spatial (location, proximity)

```go
{Subject: droneID, Predicate: "geo.location.stationed_at", Object: baseID}
{Subject: droneID, Predicate: "geo.location.operating_in", Object: zoneID}
```

**Query pattern:** "Find all entities in zone X"

### Operational (control, assignment)

```go
{Subject: droneID, Predicate: "robotics.operator.controlled_by", Object: operatorID}
{Subject: droneID, Predicate: "ops.mission.assigned_to", Object: missionID}
```

**Query pattern:** "Find drones assigned to mission X"

### Causal (triggers, causes)

```go
{Subject: alertID, Predicate: "events.trigger.triggered_by", Object: sensorID}
{Subject: abortID, Predicate: "events.cause.caused_by", Object: failureID}
```

**Query pattern:** "What caused this alert?"

---

## Predicate Naming

Use **dotted notation** from the vocabulary registry:

```text
domain.category.property
```

**Examples:**

| Predicate | Meaning |
|-----------|---------|
| `ops.fleet.member_of` | Entity belongs to fleet |
| `geo.location.stationed_at` | Entity stationed at location |
| `robotics.operator.controlled_by` | Entity controlled by operator |
| `events.trigger.triggered_by` | Event triggered by entity |

**Not snake_case:**

```text
belongs_to          # Wrong
ops.fleet.member_of # Right
```

See: [Vocabulary Registry](../basics/06-vocabulary-registry.md)

---

## Querying Relationships

### Forward Traversal (Outgoing)

Find entities this entity points to.

**GraphQL:**

```graphql
query {
  entity(id: "acme.telemetry.robotics.gcs1.drone.001") {
    id
    triples(predicate: "ops.fleet.member_of") {
      object
    }
  }
}
```

### Reverse Traversal (Incoming)

Find entities that point to this entity. Requires INCOMING_INDEX.

**GraphQL:**

```graphql
query {
  entitiesWithRelationship(
    target: "acme.ops.logistics.hq.fleet.rescue"
    predicate: "ops.fleet.member_of"
  ) {
    id
    type
  }
}
```

**Enable incoming index:**

```json
{
  "graph": {
    "config": {
      "indexer": {
        "indexes": {
          "incoming": true
        }
      }
    }
  }
}
```

### Multi-Hop Traversal (PathRAG)

Find entities N hops away.

**Example:** Find sensors on drones in fleet-rescue (2 hops)

```text
fleet.rescue ←member_of← drone.001 →has_component→ sensor.42
fleet.rescue ←member_of← drone.002 →has_component→ sensor.43
```

See: [Query Fundamentals](../advanced/01-query-fundamentals.md) for PathRAG details.

---

## Triples vs Edges

| Aspect | Triples | Edges |
|--------|---------|-------|
| Source | Graphable.Triples() | Computed (proximity, algorithms) |
| Structure | Subject-Predicate-Object | ToEntityID, EdgeType, Weight, Confidence |
| Temporal | Always current | Can expire (ExpiresAt) |
| Semantic | Rich predicates | Simple types (NEAR, POWERED_BY) |
| Use case | Domain relationships | Computed relationships |

**Use Triples for:** Domain relationships declared by payloads

**Use Edges for:** Computed relationships (proximity, graph algorithms)

---

## Best Practices

### Declare Relationships in Triples

```go
func (d *DronePayload) Triples() []Triple {
    return []Triple{
        // Good: Explicit relationship declaration
        {Subject: d.EntityID(), Predicate: "ops.fleet.member_of", Object: d.FleetID()},
    }
}
```

Don't expect SemStreams to infer relationships from property names.

### Use Consistent Predicate Direction

Pick a direction and stick with it:

```text
drone ──member_of──> fleet      # Preferred direction
```

Use incoming index for reverse queries rather than creating inverse predicates.

### Use 6-Part Entity IDs in Relationships

```go
// Good: Full 6-part ID
{Subject: droneID, Predicate: "ops.fleet.member_of", Object: "acme.ops.logistics.hq.fleet.rescue"}

// Bad: Simple ID
{Subject: droneID, Predicate: "ops.fleet.member_of", Object: "rescue"}
```

---

## Next Steps

- **[Queries](03-queries.md)** - Query the graph
- **[Indexing](04-indexing.md)** - Index configuration for relationship queries
- **[Query Fundamentals](../advanced/01-query-fundamentals.md)** - PathRAG traversal strategies

---

**Key Takeaway:** Relationships come from Triples (declared in Graphable.Triples()) and Edges (computed). Use vocabulary predicates in dotted notation, not snake_case. Don't expect relationship inference - declare relationships explicitly.
