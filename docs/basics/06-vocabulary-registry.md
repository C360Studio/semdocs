# Vocabulary Registry

**Teaching SemStreams what your data means**

---

## Why Vocabulary Matters

When you send data to SemStreams, it stores triples like:

```json
{
  "subject": "c360.platform1.robotics.gcs1.drone.1",
  "predicate": "robotics.communication.callsign",
  "object": "ALPHA-1"
}
```

But how does SemStreams know that `robotics.communication.callsign` is something you can search by? That "ALPHA-1" should resolve to this drone when someone queries for it?

**The vocabulary registry tells SemStreams what your predicates mean.**

Without it:

- The alias index doesn't know which fields are searchable names
- The spatial index doesn't know which fields contain coordinates
- The temporal index doesn't know which fields are timestamps
- Embedding extraction doesn't know which fields contain meaningful text

---

## Predicate Structure

All predicates in SemStreams follow **three-level dotted notation**:

```text
domain.category.property
```

| Part | Description | Examples |
|------|-------------|----------|
| **domain** | Business domain | `sensor`, `geo`, `robotics`, `time` |
| **category** | Group of related properties | `temperature`, `location`, `battery` |
| **property** | Specific property | `celsius`, `latitude`, `level` |

**Examples:**

```text
sensor.temperature.celsius     # Temperature in Celsius
geo.location.latitude          # GPS latitude
robotics.battery.level         # Battery percentage
time.lifecycle.created         # Creation timestamp
robotics.communication.callsign # Radio call sign
```

**Why dotted notation?**

- NATS wildcards work naturally: `robotics.battery.*` matches all battery predicates
- Human-readable without URI complexity
- Consistent with entity IDs (`c360.platform1.robotics.gcs1.drone.1`)

---

## Registering Predicates

Register your domain predicates during application initialization:

```go
package main

import "github.com/c360/semstreams/vocabulary"

func init() {
    // Register a simple predicate
    vocabulary.Register("robotics.battery.level",
        vocabulary.WithDescription("Battery charge percentage"),
        vocabulary.WithDataType("float64"),
        vocabulary.WithUnits("percent"),
        vocabulary.WithRange("0-100"))

    // Register an alias predicate (searchable by name)
    vocabulary.Register("robotics.communication.callsign",
        vocabulary.WithDescription("Radio call sign for ATC"),
        vocabulary.WithDataType("string"),
        vocabulary.WithAlias(vocabulary.AliasTypeCommunication, 0))
}
```

### Registration Options

| Option | Purpose | Example |
|--------|---------|---------|
| `WithDescription(string)` | Human-readable description | `"Battery charge percentage"` |
| `WithDataType(string)` | Expected Go type | `"float64"`, `"string"`, `"bool"` |
| `WithUnits(string)` | Measurement units | `"percent"`, `"meters"`, `"celsius"` |
| `WithRange(string)` | Valid value range | `"0-100"`, `"-90 to 90"` |
| `WithAlias(type, priority)` | Mark as entity alias | See [Alias Predicates](#alias-predicates) |
| `WithIRI(string)` | RDF/OWL IRI mapping | See [Standards Compliance](#standards-compliance-optional) |

---

## Alias Predicates

Alias predicates are fields that can be used to **find entities by name**. When you register a predicate as an alias, SemStreams indexes it for fast lookup.

### Alias Types

| Type | Description | Resolvable | Example |
|------|-------------|------------|---------|
| `AliasTypeIdentity` | Entity equivalence | Yes | UUIDs, federated IDs |
| `AliasTypeAlternate` | Secondary unique IDs | Yes | Model numbers, registration IDs |
| `AliasTypeExternal` | External system IDs | Yes | Serial numbers, legacy IDs |
| `AliasTypeCommunication` | Communication IDs | Yes | Call signs, hostnames |
| `AliasTypeLabel` | Display names | **No** | Human-readable names (ambiguous) |

**Important:** `AliasTypeLabel` is NOT used for entity resolution because labels are ambiguous (many entities can share the same name).

### Registration Example

```go
// Call sign - highest priority (0), resolvable
vocabulary.Register("robotics.communication.callsign",
    vocabulary.WithDescription("Radio call sign for ATC"),
    vocabulary.WithDataType("string"),
    vocabulary.WithAlias(vocabulary.AliasTypeCommunication, 0))

// Serial number - lower priority (1), resolvable
vocabulary.Register("robotics.identifier.serial",
    vocabulary.WithDescription("Manufacturer serial number"),
    vocabulary.WithDataType("string"),
    vocabulary.WithAlias(vocabulary.AliasTypeExternal, 1))

// Display name - lowest priority, NOT resolvable
vocabulary.Register("entity.label.display",
    vocabulary.WithDescription("Human-readable display name"),
    vocabulary.WithDataType("string"),
    vocabulary.WithAlias(vocabulary.AliasTypeLabel, 10))
```

### Priority

Lower numbers = higher priority. When multiple aliases exist for an entity, the highest priority one is used for resolution conflicts.

### How Alias Resolution Works

```text
Query: "Find entity ALPHA-1"

1. Look up "alias--alpha-1" in ALIAS_INDEX
2. Get entity ID: "c360.platform1.robotics.gcs1.drone.1"
3. Return entity from ENTITY_STATES
```

The alias index is populated automatically when entities with registered alias predicates are stored.

---

## Built-in Predicates

SemStreams provides example predicates for common domains. These are **examples only** - you should define your own domain-specific vocabulary.

### Sensor Domain (`sensor.*`)

```go
sensor.temperature.celsius     // float64, degrees Celsius
sensor.temperature.fahrenheit  // float64, degrees Fahrenheit
sensor.pressure.pascals        // float64, pascals
sensor.humidity.percent        // float64, 0-100%
sensor.accel.x                 // float64, m/s²
sensor.gyro.x                  // float64, rad/s
```

### Geospatial Domain (`geo.*`)

```go
geo.location.latitude          // float64, -90 to 90
geo.location.longitude         // float64, -180 to 180
geo.location.altitude          // float64, meters ASL
geo.velocity.ground            // float64, m/s ground speed
geo.velocity.heading           // float64, degrees 0-360
geo.accuracy.horizontal        // float64, meters CEP
```

### Temporal Domain (`time.*`)

```go
time.lifecycle.created         // time.Time, creation timestamp
time.lifecycle.updated         // time.Time, last update
time.lifecycle.seen            // time.Time, last observed
time.duration.active           // float64, seconds active
time.schedule.start            // time.Time, scheduled start
```

### Graph Relationships (`graph.rel.*`)

```go
graph.rel.contains             // Hierarchical containment
graph.rel.depends_on           // Dependency relationship
graph.rel.references           // Directional reference
graph.rel.implements           // Implementation relationship
graph.rel.near                 // Spatial proximity
graph.rel.triggered_by         // Event causation
graph.rel.blocked_by           // Blocking relationship
```

---

## Index Integration

The vocabulary registry integrates with SemStreams indexes:

### Alias Index

```go
// The alias index discovers which predicates to index
aliases := vocabulary.DiscoverAliasPredicates()
// Returns: map["robotics.communication.callsign"]int(0), ...
```

Only predicates registered with `WithAlias()` and a resolvable type are indexed.

### Spatial Index

The spatial index looks for these predicate patterns:

- `geo.location.latitude`
- `geo.location.longitude`
- `geo.location.altitude`

Entities with these predicates are automatically indexed for geospatial queries.

### Temporal Index

The temporal index looks for these predicate patterns:

- `time.lifecycle.created`
- `time.lifecycle.updated`
- `time.lifecycle.seen`

Entities with these predicates are indexed for time-based queries.

### Embedding Index

The embedding configuration specifies which fields to extract text from:

```json
{
  "embedding": {
    "text_fields": ["title", "content", "description", "summary", "text", "name"]
  }
}
```

---

## Defining Domain Vocabularies

For production applications, define your vocabulary in a dedicated package:

```go
// pkg/vocabulary/robotics/predicates.go
package robotics

import "github.com/c360/semstreams/vocabulary"

// Predicate constants
const (
    BatteryLevel    = "robotics.battery.level"
    BatteryVoltage  = "robotics.battery.voltage"
    FlightArmed     = "robotics.flight.armed"
    Callsign        = "robotics.communication.callsign"
    SerialNumber    = "robotics.identifier.serial"
    Registration    = "robotics.identifier.registration"
)

// RegisterVocabulary registers all robotics predicates
func RegisterVocabulary() {
    // Telemetry predicates
    vocabulary.Register(BatteryLevel,
        vocabulary.WithDescription("Battery charge percentage"),
        vocabulary.WithDataType("float64"),
        vocabulary.WithUnits("percent"),
        vocabulary.WithRange("0-100"))

    vocabulary.Register(BatteryVoltage,
        vocabulary.WithDescription("Battery voltage"),
        vocabulary.WithDataType("float64"),
        vocabulary.WithUnits("volts"))

    vocabulary.Register(FlightArmed,
        vocabulary.WithDescription("Flight mode armed status"),
        vocabulary.WithDataType("bool"))

    // Alias predicates (searchable)
    vocabulary.Register(Callsign,
        vocabulary.WithDescription("Radio call sign for ATC"),
        vocabulary.WithDataType("string"),
        vocabulary.WithAlias(vocabulary.AliasTypeCommunication, 0))

    vocabulary.Register(SerialNumber,
        vocabulary.WithDescription("Manufacturer serial number"),
        vocabulary.WithDataType("string"),
        vocabulary.WithAlias(vocabulary.AliasTypeExternal, 1))

    vocabulary.Register(Registration,
        vocabulary.WithDescription("FAA registration number"),
        vocabulary.WithDataType("string"),
        vocabulary.WithAlias(vocabulary.AliasTypeExternal, 2))
}
```

Then register during application startup:

```go
package main

import "myapp/pkg/vocabulary/robotics"

func main() {
    // Register domain vocabulary before starting SemStreams
    robotics.RegisterVocabulary()

    // ... start SemStreams
}
```

---

## Standards Compliance (Optional)

For integration with RDF/OWL systems, predicates can include IRI mappings:

```go
vocabulary.Register("robotics.communication.callsign",
    vocabulary.WithDescription("Radio call sign"),
    vocabulary.WithAlias(vocabulary.AliasTypeCommunication, 0),
    vocabulary.WithIRI(vocabulary.FoafAccountName))  // foaf:accountName
```

### Common Standard IRIs

| Constant | IRI | Use Case |
|----------|-----|----------|
| `OwlSameAs` | `owl:sameAs` | Entity equivalence |
| `SkosPrefLabel` | `skos:prefLabel` | Preferred label |
| `SchemaName` | `schema:name` | Display name |
| `DcIdentifier` | `dc:identifier` | External identifier |
| `FoafAccountName` | `foaf:accountName` | Communication ID |

**Key principle:** IRIs are only used at API boundaries (RDF export/import). Internal code always uses dotted notation.

```go
// Internal: Always dotted
triple.Predicate = "robotics.battery.level"

// External: Translate at boundaries
if meta := vocabulary.GetPredicateMetadata(predicate); meta != nil {
    if meta.StandardIRI != "" {
        rdfPredicate = meta.StandardIRI  // "http://schema.org/batteryLevel"
    }
}
```

---

## Registry API

### Registration

```go
// Register with functional options
vocabulary.Register(name string, opts ...Option)
```

### Retrieval

```go
// Get metadata for a predicate
meta := vocabulary.GetPredicateMetadata("robotics.battery.level")
if meta != nil {
    fmt.Println(meta.Description)  // "Battery charge percentage"
    fmt.Println(meta.DataType)     // "float64"
    fmt.Println(meta.Units)        // "percent"
}

// List all registered predicates
predicates := vocabulary.ListRegisteredPredicates()

// Discover alias predicates (for index manager)
aliases := vocabulary.DiscoverAliasPredicates()
```

### Validation

```go
// Check if predicate follows dotted notation
valid := vocabulary.IsValidPredicate("robotics.battery.level")  // true
valid := vocabulary.IsValidPredicate("invalid")                  // false
```

---

## Best Practices

### 1. Use Package Constants

```go
// Good: Constants prevent typos
triple.Predicate = robotics.BatteryLevel

// Bad: Inline strings are error-prone
triple.Predicate = "robotics.battery.level"
```

### 2. Register in init()

```go
func init() {
    vocabulary.Register("myapp.sensor.reading", ...)
}
```

### 3. Follow Naming Conventions

- **domain**: lowercase, business domain (`robotics`, `sensors`, `logistics`)
- **category**: lowercase, property group (`battery`, `location`, `status`)
- **property**: lowercase, specific property (`level`, `latitude`, `active`)
- **No underscores** in the three parts (dots only)

### 4. Choose Correct Alias Types

- Use `AliasTypeIdentity` for true equivalence (same entity in different systems)
- Use `AliasTypeExternal` for external IDs (serial numbers, vendor refs)
- Use `AliasTypeCommunication` for communication identifiers (call signs)
- **Never use `AliasTypeLabel` for resolution** - labels are for display only

### 5. Set Appropriate Priorities

```go
// Most reliable identifier first
vocabulary.WithAlias(vocabulary.AliasTypeIdentity, 0)      // Highest
vocabulary.WithAlias(vocabulary.AliasTypeCommunication, 1)
vocabulary.WithAlias(vocabulary.AliasTypeExternal, 2)
vocabulary.WithAlias(vocabulary.AliasTypeAlternate, 3)
```

---

## Next Steps

- **[Rules Engine](07-rules-engine.md)** - React to entity changes
- **[Examples](08-examples.md)** - Complete working examples

**Reference:**

- **[Architecture Deep Dive](../advanced/02-architecture-deep-dive.md)** - How indexes use vocabulary
- **[Configuration Guide](../advanced/04-configuration-guide.md)** - Enable/disable index types

---

**The vocabulary registry is the semantic layer** - It tells SemStreams what your predicates mean, enabling intelligent indexing, alias resolution, and semantic search.
