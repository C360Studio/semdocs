# Relationships

**Connecting entities: the edges in your semantic graph**

---

## What are Relationships?

Relationships connect entities, forming the graph structure. Also called **edges** or **predicates**.

**Example graph:**

```
UAV-001 ──belongs_to──> fleet-rescue
UAV-002 ──belongs_to──> fleet-rescue
UAV-001 ──located_at──> base-west
fleet-rescue ──stationed_at──> base-west
```

**PathRAG queries use these relationships** to traverse the graph (e.g., "find all drones in fleet-rescue").

---

## Relationship Structure

**In entity:**

```json
{
  "id": "UAV-001",
  "edges": [
    {
      "predicate": "belongs_to",
      "target": "fleet-rescue",
      "properties": {
        "assigned_at": "2024-01-01T00:00:00Z",
        "role": "primary-drone"
      }
    },
    {
      "predicate": "located_at",
      "target": "base-west"
    }
  ]
}
```

**Fields:**

- **predicate**: Relationship type (the verb)
- **target**: Target entity ID
- **properties**: (Optional) Metadata about the relationship

---

## Predicate Types

### Hierarchical (belongs_to, part_of, child_of)

Represents containment or membership.

```
UAV-001 ──belongs_to──> fleet-rescue
sensor-42 ──part_of──> UAV-001
```

**Query pattern:** "Find all drones in fleet X" (common in fleet management)

---

### Spatial (located_at, near, adjacent_to)

Represents physical location or proximity.

```
UAV-001 ──located_at──> base-west
UAV-001 ──near──> UAV-002
```

**Query pattern:** "Find all entities near this location"

---

### Temporal (follows, precedes, during)

Represents time-based relationships.

```
mission-042 ──follows──> mission-041
event-alert ──during──> mission-042
```

**Query pattern:** "Find events during mission X"

---

### Causal (triggered_by, caused_by, results_in)

Represents cause and effect.

```
alert-001 ──triggered_by──> UAV-002
mission-abort ──caused_by──> battery-failure
```

**Query pattern:** "What caused this alert?"

---

### Semantic (related_to, similar_to, variant_of)

Represents conceptual relationships.

```
spec-auth ──related_to──> spec-user-mgmt
design-v2 ──variant_of──> design-v1
```

**Query pattern:** "Find related documents"

---

## Creating Relationships

### Automatic (from properties)

Most common - SemStreams infers relationships from entity properties.

**Entity message:**

```json
{
  "entity_id": "UAV-001",
  "fleet": "rescue",
  "location": "base-west"
}
```

**Inferred relationships:**

```
UAV-001 ──belongs_to──> fleet-rescue
UAV-001 ──located_at──> base-west
```

**Property-to-predicate mapping:**

- `fleet` → `belongs_to` predicate
- `location` → `located_at` predicate
- `parent` → `child_of` predicate (inverse)

---

### Explicit (edges field)

Manually specify relationships.

```json
{
  "id": "UAV-001",
  "edges": [
    {
      "predicate": "assigned_to",
      "target": "mission-042",
      "properties": {"priority": "high"}
    }
  ]
}
```

---

### Via API

Create relationships via HTTP.

```bash
curl -X POST http://localhost:8080/graph/relationship \
  -d '{
    "source": "UAV-001",
    "predicate": "monitors",
    "target": "sensor-42"
  }'
```

---

## Querying Relationships

### Forward Traversal

Find entities this entity points to.

```bash
# Find what UAV-001 belongs to
curl "http://localhost:8080/graph/query?source=UAV-001&predicate=belongs_to"
```

**Response:**

```json
{
  "relationships": [
    {
      "source": "UAV-001",
      "predicate": "belongs_to",
      "target": "fleet-rescue"
    }
  ]
}
```

---

### Reverse Traversal (Incoming)

Find entities that point to this entity.

```bash
# Find all drones in fleet-rescue
curl "http://localhost:8080/graph/query?target=fleet-rescue&predicate=belongs_to"
```

**Requires incoming index** enabled:

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

---

### Multi-hop Traversal

Find entities N hops away.

```bash
# Find all sensors on drones in fleet-rescue (2 hops)
curl -X POST http://localhost:8080/graph/traverse \
  -d '{
    "start": "fleet-rescue",
    "hops": [
      {"predicate": "belongs_to", "direction": "incoming"},
      {"predicate": "has_sensor", "direction": "outgoing"}
    ]
  }'
```

**Result:**

```
fleet-rescue ←belongs_to← UAV-001 ─has_sensor→ sensor-42
fleet-rescue ←belongs_to← UAV-002 ─has_sensor→ sensor-43
```

---

## Relationship Properties

Relationships can have metadata.

```json
{
  "edges": [
    {
      "predicate": "assigned_to",
      "target": "mission-042",
      "properties": {
        "assigned_at": "2024-01-15T10:00:00Z",
        "assigned_by": "user-alice",
        "priority": "high",
        "expires_at": "2024-01-15T18:00:00Z"
      }
    }
  ]
}
```

**Query by relationship properties:**

```bash
# Find high-priority assignments
curl "http://localhost:8080/graph/query?predicate=assigned_to&properties.priority=high"
```

---

## Bidirectional Relationships

Some relationships are bidirectional.

**Example:**

```
UAV-001 ──near──> UAV-002
UAV-002 ──near──> UAV-001
```

**Automatic inverse:**

Configure automatic inverse creation:

```json
{
  "graph": {
    "config": {
      "relationship_inverses": {
        "near": "near",
        "belongs_to": "contains",
        "child_of": "parent_of"
      }
    }
  }
}
```

**Behavior:**

Create `UAV-001 ──belongs_to──> fleet-rescue`
→ Automatically creates `fleet-rescue ──contains──> UAV-001`

---

## Best Practices

### Predicate Naming

✅ **Good:** `belongs_to`, `located_at`, `triggered_by`
❌ **Bad:** `has`, `link`, `related`

Use descriptive verbs that clarify the relationship type.

### Relationship Direction

**Be consistent:**

```
✅ drone ──belongs_to──> fleet
❌ fleet ──has──> drone

✅ alert ──triggered_by──> sensor
❌ sensor ──triggers──> alert
```

Choose a primary direction and stick with it. Use incoming index for reverse queries.

### Property Usage

**Use relationship properties for metadata ABOUT the relationship:**

```json
{
  "predicate": "assigned_to",
  "target": "mission-042",
  "properties": {
    "assigned_at": "2024-01-15T10:00:00Z",  // When assigned
    "assigned_by": "user-alice"              // Who assigned
  }
}
```

**Don't use relationship properties for target entity properties:**

```json
❌ Bad:
{
  "predicate": "assigned_to",
  "target": "mission-042",
  "properties": {
    "mission_status": "active",     // This belongs on mission-042 entity
    "mission_priority": "high"      // This belongs on mission-042 entity
  }
}
```

---

## Next Steps

- **[Queries](03-queries.md)** - Advanced graph queries
- **[Indexing](04-indexing.md)** - Index configuration for relationship queries
- **[PathRAG Decisions](../advanced/01-pathrag-graphrag-decisions.md)** - When to use relationship traversal

---

**Relationships form the graph** - Entities are nodes, relationships are edges. Together they enable powerful traversal queries.
