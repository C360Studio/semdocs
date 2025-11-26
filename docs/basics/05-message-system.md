# Message System Guide

## How data flows through SemStreams

**Package:** `message`
**Status:** Core infrastructure (stable)

## Overview

All data in SemStreams flows as **structured messages** - typed, validated containers with behavioral capabilities. This guide explains how to create custom message payloads and use the message system effectively.

**Key Insight**: Messages are not just data containers - they're self-describing, validated, routable entities that components can inspect and process based on their capabilities.

## Message Structure

Every message consists of five core elements:

```go
type Message interface {
    ID() string            // Unique identifier (UUID)
    Type() Type           // Schema (domain.category.version)
    Payload() Payload     // Your data + optional behaviors
    Meta() Meta          // Timestamps, source, federation info
    Hash() string        // Content-based deduplication
    Validate() error     // Validation at message + payload level
}
```

### 1. ID - Unique Tracking

```go
msg.ID()  // "550e8400-e29b-41d4-a716-446655440000"
```

- Globally unique UUID
- Enables deduplication, tracking, and correlation
- Immutable after creation

### 2. Type - Schema & Routing

```go
msg.Type()  // Type{Domain: "sensors", Category: "temperature", Version: "v1"}
```

**Type structure:**

```go
Type{
    Domain:   "sensors",      // Data domain (organizational unit)
    Category: "temperature",  // Message category (specific type)
    Version:  "v1",          // Schema version (evolution)
}
```

**Maps to NATS subject:**

```go
msg.Type().Key()  // "sensors.temperature.v1"
```

This enables:

- **Type-based routing**: Subscribe to `sensors.temperature.*` (all versions)
- **Domain isolation**: Subscribe to `sensors.>` (all sensor types)
- **Version-specific processing**: Handle `v1` differently than `v2`

### 3. Payload - Your Data + Behaviors

The payload contains your actual data and implements the `Payload` interface:

```go
type Payload interface {
    Schema() Type         // Returns message type
    Validate() error      // Domain-specific validation
    json.Marshaler        // MarshalJSON for serialization
    json.Unmarshaler      // UnmarshalJSON for deserialization
}
```

### 4. Meta - Lifecycle Information

```go
msg.Meta().CreatedAt()  // Message creation time
msg.Meta().Source()     // "temperature-monitor" (service that created it)
```

### 5. Hash - Content-Based Identity

```go
msg.Hash()  // "sha256:abc123..."
```

- Enables deduplication
- Content-addressable storage
- Computed from type + payload

## Creating Custom Payloads

### Step 1: Define Your Payload Struct

```go
package sensors

import (
    "encoding/json"
    "errors"
    "time"
    "github.com/c360/semstreams/message"
)

type TemperaturePayload struct {
    SensorID    string    `json:"sensor_id"`
    Temperature float64   `json:"temperature"`
    Unit        string    `json:"unit"`
    Timestamp   time.Time `json:"timestamp"`
    Location    struct {
        Lat float64 `json:"lat"`
        Lon float64 `json:"lon"`
    } `json:"location"`
}
```

### Step 2: Implement Required Payload Interface

```go
// Schema returns the message type
func (t *TemperaturePayload) Schema() message.Type {
    return message.Type{
        Domain:   "sensors",
        Category: "temperature",
        Version:  "v1",
    }
}

// Validate performs domain-specific validation
func (t *TemperaturePayload) Validate() error {
    if t.SensorID == "" {
        return errors.New("sensor_id is required")
    }
    if t.Temperature < -273.15 {  // Absolute zero check
        return errors.New("temperature below absolute zero")
    }
    if t.Unit != "celsius" && t.Unit != "fahrenheit" && t.Unit != "kelvin" {
        return errors.New("unit must be celsius, fahrenheit, or kelvin")
    }
    return nil
}

// MarshalJSON implements json.Marshaler
// Use alias to avoid infinite recursion
func (t *TemperaturePayload) MarshalJSON() ([]byte, error) {
    type Alias TemperaturePayload
    return json.Marshal((*Alias)(t))
}

// UnmarshalJSON implements json.Unmarshaler
// Use alias to avoid infinite recursion
func (t *TemperaturePayload) UnmarshalJSON(data []byte) error {
    type Alias TemperaturePayload
    return json.Unmarshal(data, (*Alias)(t))
}
```

### Step 3: Implement Graphable (Required for Graph Storage)

If your payload represents entities that should be stored in the knowledge graph, you **must** implement the `Graphable` interface:

```go
// Graphable - REQUIRED for graph storage
func (t *TemperaturePayload) EntityID() string {
    // Return deterministic 6-part federated ID
    return fmt.Sprintf("acme.platform1.sensors.iot.temperature.%s", t.SensorID)
}

func (t *TemperaturePayload) Triples() []message.Triple {
    entityID := t.EntityID()
    return []message.Triple{
        {
            Subject:    entityID,
            Predicate:  "rdf:type",
            Object:     "sensors:TemperatureSensor",
            Source:     "system",
            Timestamp:  t.Timestamp,
            Confidence: 1.0,
        },
        {
            Subject:    entityID,
            Predicate:  "sensor.temperature.value",
            Object:     t.Temperature,
            Source:     "sensor_reading",
            Timestamp:  t.Timestamp,
            Confidence: 1.0,  // Direct sensor data
        },
        {
            Subject:    entityID,
            Predicate:  "sensor.temperature.unit",
            Object:     t.Unit,
            Source:     "sensor_config",
            Timestamp:  t.Timestamp,
            Confidence: 1.0,
        },
    }
}
```

**Triple metadata fields:**

| Field | Purpose | Example |
|-------|---------|---------|
| `Source` | Provenance tracking | `"mavlink"`, `"operator"`, `"ai_inference"` |
| `Timestamp` | When assertion was made | `time.Now()` |
| `Confidence` | Reliability (0.0-1.0) | `1.0` for direct data, `0.5` for inferred |
| `Context` | Correlation ID (optional) | Batch ID, request ID |
| `Datatype` | RDF datatype hint (optional) | `"xsd:float"`, `"geo:point"` |

**Without Graphable, your payload cannot be stored in the graph.** The graph processor only accepts payloads that implement this interface.

### Step 4: Create Messages

```go
// Create payload
payload := &TemperaturePayload{
    SensorID:    "temp-sensor-001",
    Temperature: 22.5,
    Unit:        "celsius",
    Timestamp:   time.Now(),
    Location: struct {
        Lat float64 `json:"lat"`
        Lon float64 `json:"lon"`
    }{
        Lat: 37.7749,
        Lon: -122.4194,
    },
}

// Create message
msg := message.NewBaseMessage(
    payload.Schema(),
    payload,
    "temperature-monitor",  // Source service
)

// Validate before use
if err := msg.Validate(); err != nil {
    log.Fatalf("Invalid message: %v", err)
}

```

## Interfaces as Contracts

SemStreams uses Go interfaces as data contracts. This is standard Go practice - implement the interfaces you need, and the system discovers capabilities at runtime via type assertions.

The two core interfaces are:

| Interface | Purpose | Required For |
|-----------|---------|--------------|
| `Payload` | Message transport | All messages |
| `Graphable` | Graph storage | Storing entities in the knowledge graph |

Additional interfaces can be defined as needed for specific processing requirements. Components discover capabilities via type assertions:

```go
if graphable, ok := payload.(message.Graphable); ok {
    // Process as graph entity
}
```

## Complete Example

```go
package sensors

import (
    "encoding/json"
    "errors"
    "fmt"
    "time"

    "github.com/c360/semstreams/message"
)

// Payload definition
type TemperaturePayload struct {
    SensorID    string    `json:"sensor_id"`
    Temperature float64   `json:"temperature"`
    Unit        string    `json:"unit"`
    Timestamp   time.Time `json:"timestamp"`
    Location    struct {
        Lat float64 `json:"lat"`
        Lon float64 `json:"lon"`
    } `json:"location"`
}

// Type definition
var TemperatureType = message.Type{
    Domain:   "sensors",
    Category: "temperature",
    Version:  "v1",
}

// ===========================================
// Required: Payload interface (for transport)
// ===========================================

func (t *TemperaturePayload) Schema() message.Type {
    return TemperatureType
}

func (t *TemperaturePayload) Validate() error {
    if t.SensorID == "" {
        return errors.New("sensor_id is required")
    }
    if t.Temperature < -273.15 {
        return errors.New("temperature below absolute zero")
    }
    return nil
}

func (t *TemperaturePayload) MarshalJSON() ([]byte, error) {
    type Alias TemperaturePayload
    return json.Marshal((*Alias)(t))
}

func (t *TemperaturePayload) UnmarshalJSON(data []byte) error {
    type Alias TemperaturePayload
    return json.Unmarshal(data, (*Alias)(t))
}

// ===========================================
// Required: Graphable interface (for graph storage)
// ===========================================

func (t *TemperaturePayload) EntityID() string {
    // 6-part federated format: org.platform.domain.system.type.instance
    return fmt.Sprintf("acme.platform1.sensors.iot.temperature.%s", t.SensorID)
}

func (t *TemperaturePayload) Triples() []message.Triple {
    entityID := t.EntityID()
    return []message.Triple{
        {
            Subject:    entityID,
            Predicate:  "rdf:type",
            Object:     "sensors:TemperatureSensor",
            Source:     "system",
            Timestamp:  t.Timestamp,
            Confidence: 1.0,
        },
        {
            Subject:    entityID,
            Predicate:  "sensor.temperature.value",
            Object:     t.Temperature,
            Source:     "sensor_reading",
            Timestamp:  t.Timestamp,
            Confidence: 1.0,
        },
        {
            Subject:    entityID,
            Predicate:  "sensor.temperature.unit",
            Object:     t.Unit,
            Source:     "sensor_config",
            Timestamp:  t.Timestamp,
            Confidence: 1.0,
        },
        {
            Subject:    entityID,
            Predicate:  "geo.location.latitude",
            Object:     t.Location.Lat,
            Source:     "gps",
            Timestamp:  t.Timestamp,
            Confidence: 0.95,  // GPS accuracy varies
        },
        {
            Subject:    entityID,
            Predicate:  "geo.location.longitude",
            Object:     t.Location.Lon,
            Source:     "gps",
            Timestamp:  t.Timestamp,
            Confidence: 0.95,
        },
    }
}

```

## Next Steps

- **[Components](02-components.md)** - See how to use messages in components
- **[Routing](03-routing.md)** - Learn how to route messages in flows
- **[Vocabulary Registry](06-vocabulary-registry.md)** - Define predicates for your triples
- **[Rules Engine](07-rules-engine.md)** - React to entity changes

---

**Key Takeaways:**

- ✅ **Payload interface** is required for message transport
- ✅ **Graphable interface** is required for graph storage
- ✅ SemStreams uses interfaces as contracts - standard Go practice
- ✅ Always validate messages before processing
