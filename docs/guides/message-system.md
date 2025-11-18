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

### Step 3: Implement Optional Behavioral Interfaces

Add capabilities that make sense for your payload:

```go
// Timeable - Enables time-series indexing
func (t *TemperaturePayload) Timestamp() time.Time {
    return t.Timestamp
}

// Locatable - Enables spatial indexing
func (t *TemperaturePayload) Location() (float64, float64) {
    return t.Location.Lat, t.Location.Lon
}

// Observable - Enables sensor reading processing
func (t *TemperaturePayload) ObservedEntity() string {
    return t.SensorID
}

func (t *TemperaturePayload) ObservedProperty() string {
    return "temperature"
}

func (t *TemperaturePayload) ObservedValue() any {
    return t.Temperature
}

func (t *TemperaturePayload) ObservedUnit() string {
    return t.Unit
}
```

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

// Validate before sending
if err := msg.Validate(); err != nil {
    log.Fatalf("Invalid message: %v", err)
}

// Publish to NATS
subject := msg.Type().Key()  // "sensors.temperature.v1"
data, _ := json.Marshal(msg)
nc.Publish(subject, data)
```

## Behavioral Interfaces

### Graphable - Entity Graph Storage

**Use when:** Payload represents entities that should be stored in the knowledge graph.

```go
type Graphable interface {
    EntityID() string      // Federated entity identifier
    Triples() []Triple    // Semantic facts about the entity
}
```

**Example:**

```go
func (t *TemperaturePayload) EntityID() string {
    return message.EntityID{
        Org:      "acme",
        Platform: "platform1",
        Domain:   "sensors",
        System:   "iot",
        Type:     "temperature",
        Instance: t.SensorID,
    }.Key()  // "acme.platform1.sensors.iot.temperature.temp-sensor-001"
}

func (t *TemperaturePayload) Triples() []message.Triple {
    return []message.Triple{
        {
            Subject:   t.EntityID(),
            Predicate: "sosa:hasValue",
            Object:    t.Temperature,
        },
        {
            Subject:   t.EntityID(),
            Predicate: "sosa:hasUnit",
            Object:    t.Unit,
        },
        {
            Subject:   t.EntityID(),
            Predicate: "sosa:resultTime",
            Object:    t.Timestamp,
        },
    }
}
```

### Locatable - Spatial Indexing

**Use when:** Payload has geographic location data.

```go
type Locatable interface {
    Location() (lat, lon float64)
}
```

**Example:**

```go
func (t *TemperaturePayload) Location() (float64, float64) {
    return t.Location.Lat, t.Location.Lon
}
```

**Enables:**

- Spatial queries ("Find all sensors within 10km")
- Geographic clustering
- Location-based routing

### Timeable - Time-Series Analysis

**Use when:** Payload represents time-series data or historical events.

```go
type Timeable interface {
    Timestamp() time.Time
}
```

**Note:** This is the event/observation time, not message creation time.

**Example:**

```go
func (t *TemperaturePayload) Timestamp() time.Time {
    return t.Timestamp
}
```

**Enables:**

- Time-series indexing
- Temporal queries
- Historical analysis

### Observable - Sensor Readings

**Use when:** Payload is a sensor reading or observation.

```go
type Observable interface {
    ObservedEntity() string  // ID of observed entity
    ObservedProperty() string // Property being observed
    ObservedValue() any      // Observed value
    ObservedUnit() string    // Measurement unit
}
```

**Example:**

```go
func (t *TemperaturePayload) ObservedEntity() string {
    return t.SensorID
}

func (t *TemperaturePayload) ObservedProperty() string {
    return "temperature"
}

func (t *TemperaturePayload) ObservedValue() any {
    return t.Temperature
}

func (t *TemperaturePayload) ObservedUnit() string {
    return t.Unit
}
```

### Correlatable - Distributed Tracing

**Use when:** Messages need to be correlated across requests/responses.

```go
type Correlatable interface {
    CorrelationID() string
}
```

**Example:**

```go
type RequestPayload struct {
    // ... fields ...
    CorrelationID string `json:"correlation_id"`
}

func (r *RequestPayload) CorrelationID() string {
    return r.CorrelationID
}
```

### Processable - Priority & Deadlines

**Use when:** Messages need priority-based or deadline-aware processing.

```go
type Processable interface {
    Priority() int          // 0-10, higher = more important
    Deadline() time.Time    // Processing deadline
}
```

**Example:**

```go
type AlertPayload struct {
    // ... fields ...
    Severity  string    `json:"severity"`
    ExpiresAt time.Time `json:"expires_at"`
}

func (a *AlertPayload) Priority() int {
    switch a.Severity {
    case "critical":
        return 10
    case "high":
        return 7
    case "medium":
        return 5
    default:
        return 3
    }
}

func (a *AlertPayload) Deadline() time.Time {
    return a.ExpiresAt
}
```

## Processing Messages

### Basic Message Handling

```go
func messageHandler(m *nats.Msg) {
    // Deserialize message
    var msg message.BaseMessage
    if err := json.Unmarshal(m.Data, &msg); err != nil {
        log.Printf("Failed to unmarshal message: %v", err)
        return
    }

    // Validate
    if err := msg.Validate(); err != nil {
        log.Printf("Invalid message: %v", err)
        return
    }

    // Process based on type
    switch msg.Type().Domain {
    case "sensors":
        processSensorMessage(&msg)
    case "alerts":
        processAlertMessage(&msg)
    default:
        log.Printf("Unknown domain: %s", msg.Type().Domain)
    }
}
```

### Capability Discovery

```go
func processSensorMessage(msg *message.BaseMessage) {
    payload := msg.Payload()

    // Check for graph storage capability
    if graphable, ok := payload.(message.Graphable); ok {
        entityID := graphable.EntityID()
        triples := graphable.Triples()
        // Store in knowledge graph
        storeEntity(entityID, triples)
    }

    // Check for spatial capability
    if locatable, ok := payload.(message.Locatable); ok {
        lat, lon := locatable.Location()
        // Build spatial index
        spatialIndex.Add(msg.ID(), lat, lon)
    }

    // Check for time-series capability
    if timeable, ok := payload.(message.Timeable); ok {
        timestamp := timeable.Timestamp()
        // Index by time
        timeSeriesDB.Store(timestamp, payload)
    }

    // Check for observation capability
    if observable, ok := payload.(message.Observable); ok {
        entity := observable.ObservedEntity()
        property := observable.ObservedProperty()
        value := observable.ObservedValue()
        unit := observable.ObservedUnit()
        // Process sensor reading
        processSensorReading(entity, property, value, unit)
    }
}
```

## Type System

### Message Type (Schema)

**Format:** `domain.category.version`

```go
Type{
    Domain:   "sensors",      // Organizational domain
    Category: "temperature",  // Specific message type
    Version:  "v1",          // Schema version
}
```

**Use for:**

- NATS routing (`sensors.temperature.v1`)
- Schema validation
- Version evolution

### Entity Type (Graph Classification)

**Format:** `domain.type`

```go
EntityType{
    Domain: "robotics",
    Type:   "drone",
}
```

**Use for:**

- Graph queries ("Find all `robotics.drone` entities")
- Entity classification
- Derived from EntityID

### Entity ID (Federated Identity)

**Format:** `org.platform.domain.system.type.instance`

```go
EntityID{
    Org:      "acme",
    Platform: "platform1",
    Domain:   "robotics",
    System:   "mav",
    Type:     "drone",
    Instance: "001",
}.Key()  // "acme.platform1.robotics.mav.drone.001"
```

**Use for:**

- Globally unique entity identity
- Multi-platform deployments
- Federated knowledge graphs

## Best Practices

### 1. Implement Required Methods First

```go
// ✅ GOOD: All required methods implemented
func (t *TemperaturePayload) Schema() message.Type { /* ... */ }
func (t *TemperaturePayload) Validate() error { /* ... */ }
func (t *TemperaturePayload) MarshalJSON() ([]byte, error) { /* ... */ }
func (t *TemperaturePayload) UnmarshalJSON([]byte) error { /* ... */ }
```

### 2. Only Implement Relevant Behavioral Interfaces

```go
// ✅ GOOD: Implements Locatable because it has real GPS data
func (d *DronePayload) Location() (float64, float64) {
    return d.GPS.Lat, d.GPS.Lon
}

// ❌ BAD: Implements Locatable with fake data
func (u *UserPayload) Location() (float64, float64) {
    return 0, 0  // User has no location!
}
```

### 3. Validate Meaningfully

```go
// ✅ GOOD: Validates business rules
func (t *TemperaturePayload) Validate() error {
    if t.SensorID == "" {
        return errors.New("sensor_id is required")
    }
    if t.Temperature < -273.15 {
        return errors.New("temperature below absolute zero")
    }
    if t.Unit == "" {
        return errors.New("unit is required")
    }
    return nil
}

// ❌ BAD: No validation
func (t *TemperaturePayload) Validate() error {
    return nil
}
```

### 4. Use Type Constants

```go
// ✅ GOOD: Type as constant
var TemperatureType = message.Type{
    Domain:   "sensors",
    Category: "temperature",
    Version:  "v1",
}

func (t *TemperaturePayload) Schema() message.Type {
    return TemperatureType
}

// ❌ BAD: Inline construction
func (t *TemperaturePayload) Schema() message.Type {
    return message.Type{
        Domain: "sensors", Category: "temperature", Version: "v1",
    }
}
```

### 5. Always Check Validation Errors

```go
// ✅ GOOD: Validate before processing
msg := message.NewBaseMessage(payload.Schema(), payload, "my-service")
if err := msg.Validate(); err != nil {
    log.Printf("Invalid message: %v", err)
    return
}
processMessage(msg)

// ❌ BAD: Skip validation
msg := message.NewBaseMessage(payload.Schema(), payload, "my-service")
processMessage(msg)  // May process invalid data!
```

## Complete Example

```go
package sensors

import (
    "encoding/json"
    "errors"
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

// Type constant
var TemperatureType = message.Type{
    Domain:   "sensors",
    Category: "temperature",
    Version:  "v1",
}

// Required: Payload interface
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

// Optional: Behavioral interfaces
func (t *TemperaturePayload) Timestamp() time.Time {
    return t.Timestamp
}

func (t *TemperaturePayload) Location() (float64, float64) {
    return t.Location.Lat, t.Location.Lon
}

func (t *TemperaturePayload) ObservedEntity() string {
    return t.SensorID
}

func (t *TemperaturePayload) ObservedProperty() string {
    return "temperature"
}

func (t *TemperaturePayload) ObservedValue() any {
    return t.Temperature
}

func (t *TemperaturePayload) ObservedUnit() string {
    return t.Unit
}
```

## Next Steps

- **Component Development**: See how to use messages in [Component Guide](components.md)
- **Flow Configuration**: Learn how to route messages in flows
- **Graph Storage**: Understand how `Graphable` payloads become entities
- **Spatial Indexing**: Use `Locatable` for geographic queries

---

**Key Takeaways:**

- ✅ Messages are typed, validated containers
- ✅ Behavioral interfaces enable runtime capability discovery
- ✅ Type system enables routing and schema evolution
- ✅ Only implement behavioral interfaces that make semantic sense
- ✅ Always validate messages before processing
