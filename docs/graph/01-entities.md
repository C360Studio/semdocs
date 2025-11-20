# Entities

**Understanding entities: the nodes in your semantic graph**

---

## What are Entities?

Entities are the **objects** tracked in your graph. Each entity represents something that exists and has properties.

**Examples:**
- Drones (UAV-001, UAV-002)
- Fleets (rescue-fleet, surveying-fleet)
- Sensors (temp-sensor-42, pressure-gauge-12)
- Users (user-alice, user-bob)
- Documents (spec-auth, design-database)

---

## Entity Structure

```json
{
  "id": "UAV-001",
  "type": "drone",
  "class": "Object",
  "role": "primary",
  "properties": {
    "fleet": "rescue",
    "battery": {
      "level": 85.2,
      "voltage": 24.5
    },
    "location": {
      "lat": 37.7749,
      "lon": -122.4194
    },
    "status": "active"
  },
  "edges": [
    {
      "predicate": "belongs_to",
      "target": "fleet-rescue",
      "properties": {"assigned_at": "2024-01-01T00:00:00Z"}
    }
  ],
  "created_at": "2024-01-15T10:00:00Z",
  "updated_at": "2024-01-15T10:30:00Z"
}
```

**Core fields:**

- **id**: Unique identifier (required)
- **type**: Entity type/category (required)
- **class**: Object, Event, or Concept
- **role**: primary, supporting, or context
- **properties**: Arbitrary key-value data
- **edges**: Relationships to other entities
- **created_at**: Timestamp of creation
- **updated_at**: Timestamp of last update

---

## Entity Classes

### Object (Physical/Logical Things)

Represents tangible or conceptual objects that exist over time.

**Examples:**
- Drones, vehicles, devices
- Users, teams, organizations
- Documents, specifications, designs
- Infrastructure (servers, databases)

**Characteristics:**
- Persistent identity
- State changes over time
- Can have multiple relationships

---

### Event (Occurrences)

Represents something that happened at a specific time.

**Examples:**
- "Battery low alert triggered"
- "Mission started"
- "User logged in"
- "Sensor reading received"

**Characteristics:**
- Timestamp-centric
- Typically immutable after creation
- Often linked to Objects (what happened to what)

---

### Concept (Abstract Ideas)

Represents abstract ideas, categories, or classifications.

**Examples:**
- "Emergency Protocol"
- "Safety Guidelines"
- "Flight Zone North"
- "Access Level Admin"

**Characteristics:**
- Abstract, not physical
- Often used for categorization
- May represent rules or policies

---

## Entity Roles

Roles indicate importance/centrality in context.

**primary**: The main entity of interest
**supporting**: Related entity that provides context
**context**: Background information entity

**Example event:**

```json
{
  "id": "event-battery-alert",
  "type": "alert",
  "class": "Event",
  "role": "primary",
  "edges": [
    {"predicate": "triggered_by", "target": "UAV-002", "target_role": "supporting"},
    {"predicate": "in_mission", "target": "mission-042", "target_role": "context"}
  ]
}
```

**Query pattern:**
- Find events where UAV-002 is **supporting** → Find what happened TO this drone
- Find events in mission-042 (**context**) → Find all events during this mission

---

## Creating Entities

### Via JSON to Entity Converter

Most common approach - convert events to entities automatically.

**Configuration:**

```json
{
  "json_to_entity": {
    "config": {
      "entity_id_field": "entity_id",
      "entity_type_field": "entity_type",
      "entity_class": "Object",
      "entity_role": "primary"
    }
  }
}
```

**Input message:**

```json
{
  "entity_id": "UAV-001",
  "entity_type": "drone",
  "battery": 85.2,
  "fleet": "rescue"
}
```

**Generated entity:**

```json
{
  "id": "UAV-001",
  "type": "drone",
  "class": "Object",
  "role": "primary",
  "properties": {
    "battery": 85.2,
    "fleet": "rescue"
  }
}
```

---

### Via HTTP API

Direct entity creation via REST API.

```bash
curl -X POST http://localhost:8080/graph/entity \
  -H "Content-Type: application/json" \
  -d '{
    "id": "UAV-003",
    "type": "drone",
    "class": "Object",
    "properties": {
      "battery": 92.1,
      "fleet": "survey"
    }
  }'
```

---

### Via NATS Subject

Publish directly to graph subject.

```bash
nats pub events.graph.entity.drone '{
  "id": "UAV-004",
  "type": "drone",
  "class": "Object",
  "properties": {"battery": 78.5}
}'
```

---

## Entity Properties

Properties are arbitrary JSON data attached to entities.

**Flat properties:**

```json
{
  "properties": {
    "name": "Rescue Alpha",
    "status": "active",
    "battery": 85.2
  }
}
```

**Nested properties:**

```json
{
  "properties": {
    "battery": {
      "level": 85.2,
      "voltage": 24.5,
      "temperature": 42.3
    },
    "location": {
      "lat": 37.7749,
      "lon": -122.4194,
      "altitude": 120.5
    }
  }
}
```

**Query nested properties:**

```bash
curl "http://localhost:8080/graph/query?type=drone&battery.level.lte=20"
```

---

## Entity Updates

Entities are **upserted** - create if new, update if exists.

**Update behavior:**

- **id** match → Update existing entity
- Properties **merge** (new fields added, existing fields updated)
- **edges** replaced (not merged)
- **updated_at** timestamp updated

**Example:**

**Initial entity:**
```json
{"id": "UAV-001", "properties": {"battery": 85.2, "status": "active"}}
```

**Update message:**
```json
{"id": "UAV-001", "properties": {"battery": 15.4}}
```

**Result:**
```json
{"id": "UAV-001", "properties": {"battery": 15.4, "status": "active"}}
```

Note: `status` preserved, `battery` updated.

---

## Entity Aliases

Aliases allow multiple names for the same entity.

**Configuration:**

```json
{
  "graph": {
    "config": {
      "indexer": {
        "indexes": {
          "alias": true
        }
      }
    }
  }
}
```

**Entity with alias:**

```json
{
  "id": "UAV-001",
  "aliases": ["rescue-alpha", "drone-1"],
  "properties": {...}
}
```

**Query by alias:**

```bash
curl "http://localhost:8080/graph/entity/rescue-alpha"
```

Resolves to `UAV-001` automatically.

---

## Best Practices

### Entity ID Design

✅ **Good:** `UAV-001`, `user-alice`, `sensor-temp-42`
❌ **Bad:** UUID strings like `a1b2c3d4-e5f6-...` (hard to read/debug)

**Use readable IDs when possible** - UUIDs are fine when external system provides them, but human-readable IDs improve debugging.

### Entity Type Naming

✅ **Good:** `drone`, `fleet`, `sensor`, `user`
❌ **Bad:** `Drone`, `FLEET`, `sensor_device`

**Use lowercase, singular nouns** - Consistent naming aids queries.

### Property Organization

✅ **Good:** Nest related properties
```json
{
  "battery": {"level": 85.2, "voltage": 24.5},
  "location": {"lat": 37.7, "lon": -122.4}
}
```

❌ **Bad:** Flat with prefixes
```json
{
  "battery_level": 85.2,
  "battery_voltage": 24.5,
  "location_lat": 37.7,
  "location_lon": -122.4
}
```

---

## Next Steps

- **[Relationships](02-relationships.md)** - Connect entities
- **[Queries](03-queries.md)** - Query the graph
- **[Indexing](04-indexing.md)** - Index configuration for performance

---

**Entities are the foundation** - Everything else (relationships, queries, semantic search) builds on entities.
