# Vocabulary System Guide

## Semantic predicates for the knowledge graph

**Package:** `vocabulary`
**Status:** Core infrastructure (stable)

## Overview

The vocabulary system defines **predicates** (properties and relationships) used in the knowledge graph. It uses clean dotted notation internally while providing optional standards compliance at API boundaries.

**Key Insight**: SemStreams uses **pragmatic semantic web** - simple dotted notation for internal operations, with optional IRI mappings when you need standards compliance for RDF/OGC export.

## Ontological Support: Informal and Basic (By Design)

The vocabulary system provides **informal and basic ontological support** - this is a deliberate architectural choice that prioritizes simplicity and pragmatism over formal semantic rigor.

### What This Means

**Informal Ontology:**

- ✅ Predicate definitions with descriptive metadata
- ✅ Human-readable dotted notation (`sensor.temperature.celsius`)
- ✅ Simple constraints (data type, units, range)
- ✅ Alias semantics for entity resolution
- ❌ NOT formal OWL axioms or restrictions
- ❌ NOT automatic inference or reasoning
- ❌ NOT complex ontological constructs

**Basic Ontological Features:**

- ✅ Predicate registry and discovery
- ✅ Optional mappings to standard vocabularies (OWL, SKOS, Dublin Core, Schema.org)
- ✅ Bidirectional translation (dotted notation ↔ IRIs)
- ✅ Standards compliance at API boundaries
- ❌ NOT ontology imports or alignment
- ❌ NOT SPARQL queries or RDF reasoning
- ❌ NOT formal consistency checking

### Why This Approach?

**The Pragmatic Semantic Web Philosophy:**

| Full Ontologies (OWL/RDF) | SemStreams Vocabulary |
|---------------------------|----------------------|
| Complex & powerful | Simple & practical |
| Heavy dependencies (reasoning engines) | Zero dependencies (pure Go) |
| Steep learning curve | Approachable for developers |
| Standards-first | Developer experience first |
| RDF/SPARQL required internally | Clean Go code internally |
| Research & formal systems | Operational & edge systems |

**You get 80% of the value with 20% of the complexity.**

### When This Is Sufficient

**Perfect for:**

- ✅ **IoT/Robotics**: Simple sensor measurements, no complex reasoning
- ✅ **Edge Deployments**: Can't run heavyweight reasoning engines
- ✅ **Real-time Systems**: Need fast predicate lookups, not inference
- ✅ **Developer-focused**: Engineers building systems, not ontologists
- ✅ **Operational Data**: Measurements, events, state - not research
- ✅ **Standards Compliance**: Need RDF export, not RDF everywhere

**Example - Weather Station:**

```go
// This level of support is perfect for sensor data
vocabulary.Register("sensor.temperature.celsius",
    vocabulary.WithDescription("Ambient temperature"),
    vocabulary.WithDataType("float64"),
    vocabulary.WithUnits("celsius"),
    vocabulary.WithRange("-273.15 to +∞"))

// You don't need OWL reasoning to know temperature is a float!
```

### When You Need Full Ontologies

**Consider full RDF/OWL if you need:**

- ❌ **Formal reasoning**: Automatic inference of new facts
- ❌ **Complex restrictions**: Cardinality constraints, property chains, disjointness
- ❌ **Ontology alignment**: Merging multiple domain ontologies
- ❌ **Consistency checking**: Formal validation of semantic correctness
- ❌ **Academic research**: Formal semantics and proofs
- ❌ **Regulatory compliance**: Systems requiring formal verification (medical, legal)

**Example - Medical Ontology:**

```text
// Medical systems might need formal reasoning
If patient has symptom X AND symptom Y
  AND drug contraindication Z
  THEN infer potential adverse reaction

This requires OWL reasoning - not supported natively
```

### Bridging to Full Ontologies

**When you need formal reasoning, you can export:**

```go
// Step 1: Define predicates in SemStreams (dotted notation)
vocabulary.Register("robotics.flight.armed",
    vocabulary.WithIRI("http://example.org/ontology/armed"))

// Step 2: Use internally with clean dotted notation
triple := message.Triple{
    Subject:   droneID,
    Predicate: "robotics.flight.armed",  // Clean!
    Object:    true,
}

// Step 3: Export to RDF for external reasoning
func exportToRDF(triples []message.Triple) *RDFGraph {
    rdfGraph := NewRDFGraph()
    for _, triple := range triples {
        // Translate dotted → IRI using vocabulary metadata
        meta := vocabulary.GetPredicateMetadata(triple.Predicate)
        rdfTriple := RDFTriple{
            Subject:   triple.Subject,
            Predicate: meta.StandardIRI,  // IRI for reasoning
            Object:    triple.Object,
        }
        rdfGraph.Add(rdfTriple)
    }
    return rdfGraph
}

// Step 4: Run external OWL reasoner (Pellet, HermiT, etc.)
inferredTriples := owlReasoner.Infer(rdfGraph)

// Step 5: Import inferred facts back to SemStreams
func importFromRDF(rdfTriples []RDFTriple) []message.Triple {
    var triples []message.Triple
    for _, rdfTriple := range rdfTriples {
        // Translate IRI → dotted using vocabulary registry
        dotted := vocabulary.LookupByIRI(rdfTriple.Predicate)
        if dotted != "" {
            triples = append(triples, message.Triple{
                Subject:   rdfTriple.Subject,
                Predicate: dotted,  // Back to clean dotted notation
                Object:    rdfTriple.Object,
            })
        }
    }
    return triples
}
```

**This gives you:**

- ✅ Clean internal architecture (dotted notation)
- ✅ Formal reasoning when needed (export to RDF/OWL)
- ✅ Best of both worlds (simple + powerful)

### Progressive Enhancement

**The vocabulary system enables progressive ontological enhancement:**

```text
Level 0: Dotted predicates only
  ↓     (Add metadata)
Level 1: Predicates with descriptions, types, units
  ↓     (Add IRI mappings)
Level 2: Mapped to standard vocabularies (OWL, SKOS)
  ↓     (Export to RDF)
Level 3: External reasoning with OWL engines
  ↓     (Import inferred facts)
Level 4: Full semantic web integration
```

**Start simple, enhance as needed.**

### Design Philosophy Summary

The vocabulary package embodies **"informal and basic ontological support"** because:

1. **Simplicity First**: Most systems don't need formal reasoning
2. **Developer Experience**: Engineers shouldn't need PhDs in ontologies
3. **Edge Deployment**: Can't run reasoning engines on constrained devices
4. **Real-time Performance**: Predicate lookups must be fast
5. **Standards When Needed**: Export to RDF/OWL at boundaries, not everywhere
6. **Progressive Enhancement**: Add formality only where required

**This is a feature, not a limitation.**

If your domain requires formal ontological reasoning, SemStreams provides the bridge (IRI mappings, export/import) without forcing that complexity on systems that don't need it.

## Core Concepts

### Dotted Notation (Internal)

All predicates use three-level dotted notation:

```text
domain.category.property
```

**Examples:**

```go
"sensor.temperature.celsius"   // Sensor domain, temperature category, celsius property
"geo.location.latitude"        // Geo domain, location category, latitude property
"time.lifecycle.created"       // Time domain, lifecycle category, created property
"graph.rel.depends_on"         // Graph domain, relationship category, depends_on property
```

### Why Dotted Notation?

1. Human-Readable

```go
// ✅ GOOD: Clear and semantic
"robotics.battery.level"

// ❌ BAD: Opaque URI
"http://purl.org/stuff/1.0#batteryLevel"
```

2. NATS-Friendly

```go
// Query all temperature predicates
nc.Subscribe("sensor.temperature.*", handler)

// Query all sensor predicates
nc.Subscribe("sensor.>", handler)
```

3. Consistent with Message System

```go
// Message Type
Type{Domain: "sensors", Category: "temperature", Version: "v1"}
→ "sensors.temperature.v1"

// Entity Type
EntityType{Domain: "robotics", Type: "drone"}
→ "robotics.drone"

// Predicate
"sensor.temperature.celsius"
→ Same dotted pattern!
```

### IRI Mappings (External)

For standards compliance (RDF, OGC, semantic web), predicates can map to standard IRIs:

```go
vocabulary.Register("semantic.identity.alias",
    vocabulary.WithIRI("http://www.w3.org/2002/07/owl#sameAs"))
```

**Internal code uses dotted:** `"semantic.identity.alias"`
**RDF export uses IRI:** `"http://www.w3.org/2002/07/owl#sameAs"`

**Translation happens only at API boundaries** - your code never sees IRIs.

## Predicate Structure

### Three-Level Pattern

```text
domain.category.property
  ↓       ↓        ↓
sensor.temperature.celsius
```

**Naming conventions:**

- **domain**: Business domain (lowercase) - `sensor`, `geo`, `time`, `robotics`
- **category**: Groups related properties (lowercase) - `temperature`, `location`, `lifecycle`
- **property**: Specific property (lowercase) - `celsius`, `latitude`, `created`
- **No underscores or special characters** - dots only for level separation

### Built-In Predicate Domains

SemStreams provides example predicates in core domains:

**Sensor Domain** - Measurements and environmental data:

```go
sensor.temperature.celsius
sensor.pressure.pascals
sensor.humidity.percent
sensor.accel.x/y/z
sensor.gyro.x/y/z
```

**Geospatial Domain** - Location and positioning:

```go
geo.location.latitude
geo.location.longitude
geo.location.altitude
geo.velocity.ground
geo.accuracy.horizontal
```

**Temporal Domain** - Time and lifecycle:

```go
time.lifecycle.created
time.lifecycle.updated
time.lifecycle.seen
time.duration.active
time.schedule.start
```

**Network Domain** - Connectivity and communication:

```go
network.connection.status
network.protocol.type
network.traffic.bytes.in
```

**Quality Domain** - Data quality and validation:

```go
quality.confidence.score
quality.validation.status
quality.accuracy.absolute
```

**Graph Domain** - Entity relationships:

```go
graph.rel.contains        // Hierarchical containment
graph.rel.references      // Directional reference
graph.rel.depends_on      // Dependency
graph.rel.implements      // Implementation
graph.rel.triggered_by    // Event causation
graph.rel.blocked_by      // Blocking relationship
graph.rel.related_to      // General association
```

## Defining Custom Predicates

### Step 1: Define Constants

Create a package for your domain vocabulary:

```go
package robotics

// Define predicate constants
const (
    BatteryLevel   = "robotics.battery.level"
    BatteryVoltage = "robotics.battery.voltage"
    BatteryTemp    = "robotics.battery.temperature"

    FlightModeArmed  = "robotics.flight.armed"
    FlightModeActive = "robotics.flight.active"
    FlightAltitude   = "robotics.flight.altitude"

    CommunicationCallsign = "robotics.communication.callsign"
    CommunicationFreq     = "robotics.communication.frequency"
)
```

### Step 2: Register Predicates

Register during package initialization:

```go
package robotics

import "github.com/c360/semstreams/vocabulary"

func init() {
    // Register with metadata
    vocabulary.Register(BatteryLevel,
        vocabulary.WithDescription("Battery charge level percentage"),
        vocabulary.WithDataType("float64"),
        vocabulary.WithUnits("percent"),
        vocabulary.WithRange("0-100"),
        vocabulary.WithIRI("http://schema.org/batteryLevel"))

    vocabulary.Register(BatteryVoltage,
        vocabulary.WithDescription("Battery voltage"),
        vocabulary.WithDataType("float64"),
        vocabulary.WithUnits("volts"),
        vocabulary.WithRange("0-50"))

    vocabulary.Register(FlightModeArmed,
        vocabulary.WithDescription("Flight mode armed status"),
        vocabulary.WithDataType("bool"))

    vocabulary.Register(CommunicationCallsign,
        vocabulary.WithDescription("Radio call sign for ATC"),
        vocabulary.WithDataType("string"),
        vocabulary.WithAlias(vocabulary.AliasTypeCommunication, 0))  // Priority 0 = highest
}
```

### Step 3: Use in Triples

```go
package robotics

import "github.com/c360/semstreams/message"

type DronePayload struct {
    DroneID        string  `json:"drone_id"`
    BatteryPercent float64 `json:"battery_percent"`
    BatteryVolts   float64 `json:"battery_volts"`
    IsArmed        bool    `json:"is_armed"`
    Callsign       string  `json:"callsign"`
}

func (d *DronePayload) Triples() []message.Triple {
    return []message.Triple{
        {
            Subject:   d.DroneID,
            Predicate: BatteryLevel,      // "robotics.battery.level"
            Object:    d.BatteryPercent,
        },
        {
            Subject:   d.DroneID,
            Predicate: BatteryVoltage,    // "robotics.battery.voltage"
            Object:    d.BatteryVolts,
        },
        {
            Subject:   d.DroneID,
            Predicate: FlightModeArmed,   // "robotics.flight.armed"
            Object:    d.IsArmed,
        },
        {
            Subject:   d.DroneID,
            Predicate: CommunicationCallsign,  // "robotics.communication.callsign"
            Object:    d.Callsign,
        },
    }
}
```

## Predicate Metadata

### Available Options

```go
vocabulary.Register("sensor.temperature.celsius",
    // Human-readable description
    vocabulary.WithDescription("Temperature in degrees Celsius"),

    // Expected Go type
    vocabulary.WithDataType("float64"),

    // Measurement units
    vocabulary.WithUnits("celsius"),

    // Valid value range
    vocabulary.WithRange("-273.15 to +∞"),

    // Domain classification
    vocabulary.WithDomain("sensor"),

    // Category within domain
    vocabulary.WithCategory("temperature"),

    // Standard IRI mapping
    vocabulary.WithIRI("http://qudt.org/vocab/unit/DEG_C"),

    // Alias semantics
    vocabulary.WithAlias(vocabulary.AliasTypeAlternate, 1))
```

### Retrieving Metadata

```go
meta := vocabulary.GetPredicateMetadata("sensor.temperature.celsius")
if meta != nil {
    fmt.Printf("Description: %s\n", meta.Description)
    fmt.Printf("Data Type: %s\n", meta.DataType)
    fmt.Printf("Units: %s\n", meta.Units)
    fmt.Printf("Range: %s\n", meta.Range)
    if meta.StandardIRI != "" {
        fmt.Printf("Standard IRI: %s\n", meta.StandardIRI)
    }
}
```

## Alias Predicates for Entity Resolution

### Alias Types

Predicates can represent entity aliases for identification and resolution:

```go
// AliasTypeIdentity - Entity equivalence (owl:sameAs)
vocabulary.Register("semantic.identity.alias",
    vocabulary.WithAlias(vocabulary.AliasTypeIdentity, 0),
    vocabulary.WithIRI(vocabulary.OwlSameAs))

// AliasTypeAlternate - Secondary unique identifiers
vocabulary.Register("product.model.number",
    vocabulary.WithAlias(vocabulary.AliasTypeAlternate, 1),
    vocabulary.WithIRI(vocabulary.SkosNotation))

// AliasTypeExternal - External system identifiers
vocabulary.Register("product.serial.number",
    vocabulary.WithAlias(vocabulary.AliasTypeExternal, 2),
    vocabulary.WithIRI(vocabulary.DcIdentifier))

// AliasTypeCommunication - Communication identifiers
vocabulary.Register("robotics.communication.callsign",
    vocabulary.WithAlias(vocabulary.AliasTypeCommunication, 0))

// AliasTypeLabel - Display names (NOT for resolution - ambiguous)
vocabulary.Register("entity.label.preferred",
    vocabulary.WithAlias(vocabulary.AliasTypeLabel, 10),
    vocabulary.WithIRI(vocabulary.SkosPrefLabel))
```

### Priority for Conflict Resolution

Lower number = higher priority:

```go
// Priority 0 - Highest (identity assertions)
vocabulary.WithAlias(vocabulary.AliasTypeIdentity, 0)

// Priority 1-5 - High (unique identifiers)
vocabulary.WithAlias(vocabulary.AliasTypeAlternate, 1)

// Priority 6-9 - Medium (external references)
vocabulary.WithAlias(vocabulary.AliasTypeExternal, 7)

// Priority 10+ - Low (labels, display names)
vocabulary.WithAlias(vocabulary.AliasTypeLabel, 10)
```

### Discovering Alias Predicates

```go
// Get all alias predicates with their priorities
aliases := vocabulary.DiscoverAliasPredicates()
for predicate, priority := range aliases {
    meta := vocabulary.GetPredicateMetadata(predicate)
    fmt.Printf("%s (priority %d, type %s)\n",
        predicate, priority, meta.AliasType)
}
```

## Standard Vocabulary Mappings

### Common Standards

SemStreams provides constants for standard vocabulary IRIs:

**OWL (Web Ontology Language):**

```go
vocabulary.OwlSameAs              // Entity equivalence
vocabulary.OwlEquivalentClass     // Class equivalence
vocabulary.OwlEquivalentProperty  // Property equivalence
```

**SKOS (Simple Knowledge Organization System):**

```go
vocabulary.SkosPrefLabel    // Preferred label
vocabulary.SkosAltLabel     // Alternative label
vocabulary.SkosNotation     // Notation/identifier
```

**Dublin Core:**

```go
vocabulary.DcIdentifier   // Unambiguous reference
vocabulary.DcTitle        // Resource name
vocabulary.DcAlternative  // Alternative name
vocabulary.DcReferences   // References relationship
vocabulary.DcRequires     // Requires/depends relationship
```

**Schema.org:**

```go
vocabulary.SchemaName           // Item name
vocabulary.SchemaIdentifier     // Item identifier
vocabulary.SchemaSameAs         // Identity assertion
vocabulary.SchemaAlternateName  // Alternative name
vocabulary.SchemaHasPart        // Contains relationship
```

### Using Standard Mappings

```go
import "github.com/c360/semstreams/vocabulary"

func init() {
    // Identity assertion
    vocabulary.Register("semantic.identity.alias",
        vocabulary.WithIRI(vocabulary.OwlSameAs))

    // Preferred label
    vocabulary.Register("entity.label.preferred",
        vocabulary.WithIRI(vocabulary.SkosPrefLabel))

    // External identifier
    vocabulary.Register("product.serial.number",
        vocabulary.WithIRI(vocabulary.DcIdentifier))

    // Alternative name
    vocabulary.Register("entity.label.alternate",
        vocabulary.WithIRI(vocabulary.SchemaAlternateName))
}
```

## IRI Translation at API Boundaries

### Exporting to RDF

```go
// Internal triple with dotted predicate
triple := message.Triple{
    Subject:   "drone-001",
    Predicate: "robotics.battery.level",  // Dotted notation
    Object:    85.5,
}

// Export to RDF - translate to IRI
meta := vocabulary.GetPredicateMetadata(triple.Predicate)
if meta != nil && meta.StandardIRI != "" {
    rdfTriple.Predicate = meta.StandardIRI  // "http://schema.org/batteryLevel"
} else {
    // Generate IRI from dotted notation
    rdfTriple.Predicate = vocabulary.PredicateIRI(triple.Predicate)
}
```

### Importing from RDF

```go
// RDF triple with IRI predicate
rdfTriple := RDFTriple{
    Subject:   "drone-001",
    Predicate: "http://schema.org/batteryLevel",  // IRI
    Object:    85.5,
}

// Import - translate IRI to dotted
dottedPredicate := vocabulary.LookupByIRI(rdfTriple.Predicate)
if dottedPredicate != "" {
    triple.Predicate = dottedPredicate  // "robotics.battery.level"
} else {
    // Unknown IRI - store as-is or skip
    log.Printf("Unknown IRI: %s", rdfTriple.Predicate)
}
```

## Query Patterns with NATS

### Wildcard Queries

Dotted notation enables powerful wildcard queries:

```go
// All temperature predicates
nc.Subscribe("sensor.temperature.*", handler)
// Matches:
//   sensor.temperature.celsius
//   sensor.temperature.fahrenheit
//   sensor.temperature.kelvin

// All sensor predicates
nc.Subscribe("sensor.>", handler)
// Matches:
//   sensor.temperature.celsius
//   sensor.pressure.pascals
//   sensor.humidity.percent

// All lifecycle events
nc.Subscribe("time.lifecycle.*", handler)
// Matches:
//   time.lifecycle.created
//   time.lifecycle.updated
//   time.lifecycle.seen

// All graph relationships
nc.Subscribe("graph.rel.*", handler)
// Matches:
//   graph.rel.depends_on
//   graph.rel.implements
//   graph.rel.references
```

### Domain-Specific Subscriptions

```go
// Subscribe to robotics domain
nc.Subscribe("robotics.>", func(m *nats.Msg) {
    // Handle all robotics predicates
    var triple message.Triple
    json.Unmarshal(m.Data, &triple)

    // Route based on category
    parts := strings.Split(triple.Predicate, ".")
    if len(parts) == 3 {
        domain := parts[0]    // "robotics"
        category := parts[1]  // "battery", "flight", etc.
        property := parts[2]  // "level", "armed", etc.

        handleRoboticsPredicate(domain, category, property, triple)
    }
})
```

## Functional Options API Reference

The `Register()` function uses functional options for clean, composable configuration.

### Available Options

**`WithDescription(desc string)`** - Human-readable description

```go
vocabulary.Register("sensor.temperature.celsius",
    vocabulary.WithDescription("Temperature in degrees Celsius"))
```

**`WithDataType(dataType string)`** - Expected Go type

Common types: `"string"`, `"float64"`, `"int"`, `"bool"`, `"time.Time"`

```go
vocabulary.Register("sensor.temperature.celsius",
    vocabulary.WithDataType("float64"))
```

**`WithUnits(units string)`** - Measurement units

```go
vocabulary.Register("sensor.temperature.celsius",
    vocabulary.WithUnits("celsius"))
```

**`WithRange(valueRange string)`** - Valid value ranges

```go
vocabulary.Register("robotics.battery.level",
    vocabulary.WithRange("0-100"))
```

**`WithIRI(iri string)`** - W3C/RDF equivalent for standards compliance

```go
vocabulary.Register("entity.label.preferred",
    vocabulary.WithIRI(vocabulary.SkosPrefLabel))
```

**`WithAlias(aliasType, priority int)`** - Mark as entity alias for resolution

```go
vocabulary.Register("robotics.communication.callsign",
    vocabulary.WithAlias(vocabulary.AliasTypeCommunication, 0))  // Priority 0 = highest
```

### Complete Registration Example

```go
vocabulary.Register("robotics.battery.level",
    vocabulary.WithDescription("Battery charge level percentage"),
    vocabulary.WithDataType("float64"),
    vocabulary.WithUnits("percent"),
    vocabulary.WithRange("0-100"),
    vocabulary.WithIRI("http://schema.org/batteryLevel"))
```

## Registry API

### Query Registered Predicates

```go
// Get metadata for a predicate
meta := vocabulary.GetPredicateMetadata("robotics.battery.level")
if meta != nil {
    fmt.Println(meta.Description)     // "Battery charge level percentage"
    fmt.Println(meta.DataType)        // "float64"
    fmt.Println(meta.StandardIRI)     // "http://schema.org/batteryLevel"
}

// Check if predicate is registered
if vocabulary.IsRegistered("sensor.temperature.celsius") {
    // Use predicate
}

// List all registered predicates
predicates := vocabulary.ListRegisteredPredicates()

// Discover alias predicates
aliases := vocabulary.DiscoverAliasPredicates()
```

### Testing

```go
// Clear registry (testing only)
vocabulary.ClearRegistry()
```

## Graph Domain Predicates

Standard relationship predicates for linking entities in the semantic graph. All `graph.rel.*` predicates include mappings to Dublin Core, Schema.org, and PROV-O vocabularies.

### Hierarchical Relationships

```go
vocabulary.GraphRelContains     // "graph.rel.contains"
vocabulary.GraphRelDependsOn    // "graph.rel.depends_on"
```

**Uses**:

- `graph.rel.contains` → `prov:hadMember` - Parent contains child (platform contains sensors)
- `graph.rel.depends_on` → `dcterms:requires` - Dependency relationship (spec depends on spec)

### Reference Relationships

```go
vocabulary.GraphRelReferences   // "graph.rel.references"
vocabulary.GraphRelRelatedTo    // "graph.rel.related_to"
vocabulary.GraphRelDiscusses    // "graph.rel.discusses"
```

**Uses**:

- `graph.rel.references` → `dcterms:references` - Documentation references specifications
- `graph.rel.related_to` → `dcterms:relation` - Generic relationship
- `graph.rel.discusses` → `schema:about` - Discussion about a topic

### Causal Relationships

```go
vocabulary.GraphRelInfluences   // "graph.rel.influences"
vocabulary.GraphRelTriggeredBy  // "graph.rel.triggered_by"
```

**Uses**:

- `graph.rel.influences` - Decision influences implementation
- `graph.rel.triggered_by` - Alert triggered by threshold

### Implementation Relationships

```go
vocabulary.GraphRelImplements   // "graph.rel.implements"
vocabulary.GraphRelSupersedes   // "graph.rel.supersedes"
vocabulary.GraphRelBlockedBy    // "graph.rel.blocked_by"
```

**Uses**:

- `graph.rel.implements` - Code implements specification
- `graph.rel.supersedes` → `dcterms:replaces` - v2 supersedes v1
- `graph.rel.blocked_by` - Issue blocked by another issue

### Spatial/Communication Relationships

```go
vocabulary.GraphRelNear         // "graph.rel.near"
vocabulary.GraphRelCommunicates // "graph.rel.communicates"
```

**Uses**:

- `graph.rel.near` - Sensors near a location
- `graph.rel.communicates` - Services communicate with each other

### Usage Example

```go
import "github.com/c360/semstreams/vocabulary"

// Create relationship between specification and implementation
triple := message.Triple{
    Subject:   "spec-001",
    Predicate: vocabulary.GraphRelImplements,  // "graph.rel.implements"
    Object:    "pr-123",
}

// Query all relationships using NATS wildcards
nc.Subscribe("graph.rel.*", handler)  // All relationship types
nc.Subscribe("graph.rel.contains", handler)  // Only containment
```

## Internal vs External Usage

### Internal: Always Dotted Notation

**All internal code uses dotted predicates:**

```go
// Creating triples
triple := message.Triple{
    Subject:   "c360.platform1.robotics.drone.001",
    Predicate: "robotics.battery.level",  // Dotted, not IRI
    Object:    85.5,
}

// NATS subscriptions
nc.Subscribe("robotics.battery.*", handler)

// Entity properties
entityState.SetProperty("geo.location.latitude", 37.7749)

// NO IRIs in internal code!
```

### External: IRI Mappings at Boundaries

**Translate at API boundaries only:**

```go
// RDF export - translate dotted to IRI
func ExportToRDF(triples []message.Triple) []RDFTriple {
    rdfTriples := make([]RDFTriple, 0, len(triples))

    for _, triple := range triples {
        rdfTriple := RDFTriple{
            Subject: triple.Subject,
            Object:  triple.Object,
        }

        // Translate predicate to IRI if registered
        if meta := vocabulary.GetPredicateMetadata(triple.Predicate); meta != nil {
            if meta.StandardIRI != "" {
                rdfTriple.Predicate = meta.StandardIRI
            } else {
                rdfTriple.Predicate = triple.Predicate  // Use dotted as fallback
            }
        }

        rdfTriples = append(rdfTriples, rdfTriple)
    }

    return rdfTriples
}

// RDF import - translate IRI to dotted
func ImportFromRDF(rdfTriples []RDFTriple) []message.Triple {
    // Build reverse lookup: IRI -> dotted name
    iriToName := make(map[string]string)
    for _, name := range vocabulary.ListRegisteredPredicates() {
        if meta := vocabulary.GetPredicateMetadata(name); meta != nil {
            if meta.StandardIRI != "" {
                iriToName[meta.StandardIRI] = name
            }
        }
    }

    triples := make([]message.Triple, 0, len(rdfTriples))

    for _, rdfTriple := range rdfTriples {
        triple := message.Triple{
            Subject: rdfTriple.Subject,
            Object:  rdfTriple.Object,
        }

        // Translate IRI to dotted notation
        if dotted, ok := iriToName[rdfTriple.Predicate]; ok {
            triple.Predicate = dotted
        } else {
            // Unknown IRI - skip or handle as needed
            continue
        }

        triples = append(triples, triple)
    }

    return triples
}
```

## Best Practices

### 1. Define Predicates as Constants

```go
// ✅ GOOD: Package constants
package sensors

const (
    TemperatureCelsius = "sensor.temperature.celsius"
    PressurePascals    = "sensor.pressure.pascals"
    HumidityPercent    = "sensor.humidity.percent"
)

// ❌ BAD: Inline strings
triple := message.Triple{
    Predicate: "sensor.temperature.celsius",  // Typo-prone
}
```

### 2. Register in init()

```go
// ✅ GOOD: Register during initialization
func init() {
    vocabulary.Register(TemperatureCelsius,
        vocabulary.WithDescription("Temperature in Celsius"),
        vocabulary.WithDataType("float64"),
        vocabulary.WithUnits("celsius"))
}

// ❌ BAD: Register at usage site
func createTriple() {
    vocabulary.Register(TemperatureCelsius, ...)  // Don't do this
}
```

### 3. Follow Naming Conventions

```go
// ✅ GOOD: Lowercase, dotted notation
"sensor.temperature.celsius"
"geo.location.latitude"
"time.lifecycle.created"

// ❌ BAD: Mixed case, underscores, special chars
"Sensor.Temperature.Celsius"
"geo_location_latitude"
"time:lifecycle:created"
```

### 4. Only Map to IRIs When Needed

```go
// ✅ GOOD: Map customer-facing predicates
vocabulary.Register("entity.label.preferred",
    vocabulary.WithIRI(vocabulary.SkosPrefLabel))  // For RDF export

// ✅ GOOD: Skip IRI for internal predicates
vocabulary.Register("internal.cache.key",
    vocabulary.WithDescription("Internal cache key"))  // No IRI needed

// ❌ BAD: Mapping everything
vocabulary.Register("internal.debug.flag",
    vocabulary.WithIRI("http://example.com/debug"))  // Unnecessary
```

### 5. Use Appropriate Alias Types

```go
// ✅ GOOD: Identity for equivalence
vocabulary.Register("semantic.identity.alias",
    vocabulary.WithAlias(vocabulary.AliasTypeIdentity, 0))

// ✅ GOOD: External for system IDs
vocabulary.Register("product.serial.number",
    vocabulary.WithAlias(vocabulary.AliasTypeExternal, 2))

// ❌ BAD: Label for resolution
vocabulary.Register("entity.label.display",
    vocabulary.WithAlias(vocabulary.AliasTypeLabel, 0))  // Labels are ambiguous!
```

### 6. Provide Complete Metadata

```go
// ✅ GOOD: Comprehensive metadata
vocabulary.Register("sensor.temperature.celsius",
    vocabulary.WithDescription("Ambient temperature in degrees Celsius"),
    vocabulary.WithDataType("float64"),
    vocabulary.WithUnits("celsius"),
    vocabulary.WithRange("-273.15 to +∞"),
    vocabulary.WithIRI("http://qudt.org/vocab/unit/DEG_C"))

// ⚠️ ACCEPTABLE: Minimal metadata for internal use
vocabulary.Register("internal.state.flag",
    vocabulary.WithDataType("bool"))

// ❌ BAD: No metadata at all
vocabulary.Register("some.random.thing")  // What is it? What type?
```

## Complete Example

```go
package sensors

import (
    "github.com/c360/semstreams/message"
    "github.com/c360/semstreams/vocabulary"
)

// Define predicate constants
const (
    TemperatureCelsius = "sensor.temperature.celsius"
    HumidityPercent    = "sensor.humidity.percent"
    PressurePascals    = "sensor.pressure.pascals"
)

// Register predicates
func init() {
    vocabulary.Register(TemperatureCelsius,
        vocabulary.WithDescription("Ambient temperature in Celsius"),
        vocabulary.WithDataType("float64"),
        vocabulary.WithUnits("celsius"),
        vocabulary.WithRange("-273.15 to +∞"))

    vocabulary.Register(HumidityPercent,
        vocabulary.WithDescription("Relative humidity percentage"),
        vocabulary.WithDataType("float64"),
        vocabulary.WithUnits("percent"),
        vocabulary.WithRange("0-100"))

    vocabulary.Register(PressurePascals,
        vocabulary.WithDescription("Atmospheric pressure"),
        vocabulary.WithDataType("float64"),
        vocabulary.WithUnits("pascals"),
        vocabulary.WithRange("0 to +∞"))
}

// Use in payload
type WeatherPayload struct {
    StationID   string  `json:"station_id"`
    Temperature float64 `json:"temperature"`
    Humidity    float64 `json:"humidity"`
    Pressure    float64 `json:"pressure"`
}

func (w *WeatherPayload) EntityID() string {
    return message.EntityID{
        Org:      "acme",
        Platform: "platform1",
        Domain:   "sensors",
        System:   "weather",
        Type:     "station",
        Instance: w.StationID,
    }.Key()
}

func (w *WeatherPayload) Triples() []message.Triple {
    return []message.Triple{
        {
            Subject:   w.EntityID(),
            Predicate: TemperatureCelsius,  // Uses constant
            Object:    w.Temperature,
        },
        {
            Subject:   w.EntityID(),
            Predicate: HumidityPercent,
            Object:    w.Humidity,
        },
        {
            Subject:   w.EntityID(),
            Predicate: PressurePascals,
            Object:    w.Pressure,
        },
    }
}
```

## Next Steps

- **Message System**: Learn how messages and triples work together in [Message System Guide](message-system.md)
- **Graph Storage**: Understand how triples become the knowledge graph
- **Query Patterns**: Use PathRAG and GraphRAG to query the graph
- **Standards Compliance**: Export to RDF/Turtle for semantic web integration

---

**Key Takeaways:**

- ✅ Use dotted notation internally (`domain.category.property`)
- ✅ Map to standard IRIs only at API boundaries
- ✅ Register predicates in `init()` with metadata
- ✅ Use predicate constants, not inline strings
- ✅ Leverage NATS wildcards for powerful queries
- ✅ Choose appropriate alias types for entity resolution
