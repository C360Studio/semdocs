# SemStreams Unified Architecture

**Version**: 0.2.0-alpha
**Status**: Post-rename, pre-merge
**Date**: November 2025

## Overview

SemStreams is a two-layer framework for semantic stream processing at the edge.
It combines protocol-level data handling with generic semantic capabilities while
maintaining strict separation from domain-specific code.

## Architecture Philosophy

### Two Independent Layers

```
┌─────────────────────────────────────────────────────┐
│           SEMANTIC LAYER (domain agnostic)          │
│  Entity tracking | Graph structures | Vocabularies  │
│  Generic semantic enrichment pipeline               │
└──────────────────┬──────────────────────────────────┘
                   │ extends via standard interfaces
                   ↓
┌─────────────────────────────────────────────────────┐
│          PROTOCOL LAYER (network agnostic)          │
│  UDP | TCP | HTTP | WebSocket | JSON | NATS        │
│  Works standalone without semantic layer            │
└─────────────────────────────────────────────────────┘
```

**Key Principle**: Protocol layer has ZERO dependencies on semantic layer.

### What Belongs Where

**Protocol Layer** (in SemStreams framework):
- Network protocols: UDP, TCP, HTTP, WebSocket
- Data formats: JSON, CSV, raw bytes
- Messaging: NATS pub/sub, JetStream, KV
- Infrastructure: Metrics, health checks, logging
- Flow orchestration: Component lifecycle, registry

**Semantic Layer** (in SemStreams framework):
- Entity lifecycle and relationships
- Graph structures and patterns
- Vocabularies (SOSA/SSN compatible)
- Semantic enrichment pipeline
- Generic entity extraction

**Domain Layer** (separate modules):
- Domain protocols: MAVLink → `streamkit-robotics`
- ROS integration → `streamkit-robotics`
- Industrial protocols → `streamkit-industrial`
- Business logic → domain-specific modules

## Component Architecture

### Flow Engine

```
┌────────────────────────────────────┐
│         Flow Engine                │
│  - Component lifecycle             │
│  - State management                │
│  - Health monitoring               │
│  - Metrics collection              │
└───────────┬────────────────────────┘
            │ orchestrates
            ↓
┌────────────────────────────────────┐
│      Component Registry            │
│  - Protocol components             │
│  - Semantic components (optional)  │
│  - Domain components (external)    │
└───────────┬────────────────────────┘
            │ communicate via
            ↓
┌────────────────────────────────────┐
│         NATS Messaging             │
│  - Pub/Sub subjects                │
│  - JetStream (persistence)         │
│  - KV stores (state)               │
└────────────────────────────────────┘
```

### Component Types

**Input Components** (Protocol Layer):
- UDP Input: Raw socket data → NATS subject
- WebSocket Input: Federation endpoints
- HTTP Input: REST endpoints (future)
- TCP Input: Stream sockets (future)

**Processor Components** (Protocol Layer):
- JSON Parser: Parse raw bytes → JSON
- JSON Filter: JSONPath filtering
- JSON Map: Field mapping and transformation
- JSON Generic: Generic JSON processing

**Processor Components** (Semantic Layer - post-merge):
- Entity Extractor: JSON → Entities
- Entity Enricher: Add semantic metadata
- Graph Builder: Entity relationships
- Vocabulary Mapper: Domain vocabularies

**Output Components** (Protocol Layer):
- WebSocket Output: Broadcasting to clients
- File Output: Write to disk (JSON lines)
- HTTP POST Output: Webhook delivery
- Object Store: NATS bucket storage

**Service Components**:
- Component Manager: HTTP API for flow management
- Metrics Service: Prometheus metrics
- Message Logger: Debug logging

## Data Flow Patterns

### Protocol-Only Flow

```
┌─────────┐     ┌──────────┐     ┌────────────┐
│   UDP   │────→│JSON      │────→│ WebSocket  │
│  Input  │     │Filter    │     │   Output   │
└─────────┘     └──────────┘     └────────────┘
   14550         NATS subjects       :8082/ws

Protocol layer only - no semantic processing
```

### Semantic Flow (post-merge)

```
┌─────────┐     ┌──────────┐     ┌────────────┐     ┌──────────┐
│   UDP   │────→│ Entity   │────→│  Entity    │────→│ Object   │
│  Input  │     │Extractor │     │ Enricher   │     │  Store   │
└─────────┘     └──────────┘     └────────────┘     └──────────┘
   14550         JSON → Entity     + metadata         NATS KV

Protocol → Semantic → Storage
```

### Federation Pattern

```
┌────────────────┐                    ┌────────────────┐
│  Edge Instance │                    │ Cloud Instance │
│                │                    │                │
│  ┌──────────┐  │                    │  ┌──────────┐  │
│  │   UDP    │  │                    │  │WebSocket │  │
│  │  Input   │  │                    │  │  Input   │  │
│  └────┬─────┘  │                    │  └────┬─────┘  │
│       ↓        │    WebSocket       │       ↓        │
│  ┌──────────┐  │      :8080         │  ┌──────────┐  │
│  │WebSocket │  ├───────────────────→│  │ Entity   │  │
│  │ Output   │  │                    │  │Extractor │  │
│  └──────────┘  │                    │  └────┬─────┘  │
│                │                    │       ↓        │
└────────────────┘                    │  ┌──────────┐  │
                                      │  │ Storage  │  │
                                      │  └──────────┘  │
                                      └────────────────┘

Edge: Protocol processing only
Cloud: Protocol + Semantic processing
```

## Configuration Structure

### Protocol-Only Configuration

```json
{
  "name": "protocol-flow",
  "components": [
    {
      "id": "udp-in",
      "type": "udp_input",
      "config": {"port": 14550},
      "outputs": [{"subject": "raw.udp"}]
    },
    {
      "id": "ws-out",
      "type": "websocket_output",
      "config": {"port": 8082},
      "inputs": [{"subject": "raw.udp"}]
    }
  ]
}
```

### Semantic Configuration (post-merge)

```json
{
  "name": "semantic-flow",
  "components": [
    {
      "id": "udp-in",
      "type": "udp_input",
      "config": {"port": 14550},
      "outputs": [{"subject": "raw.data"}]
    },
    {
      "id": "entity-extract",
      "type": "entity_extractor",
      "config": {"vocabulary": "sosa"},
      "inputs": [{"subject": "raw.data"}],
      "outputs": [{"subject": "entities"}]
    },
    {
      "id": "entity-enrich",
      "type": "entity_enricher",
      "inputs": [{"subject": "entities"}],
      "outputs": [{"subject": "enriched"}]
    },
    {
      "id": "storage",
      "type": "objectstore",
      "inputs": [{"subject": "enriched"}]
    }
  ]
}
```

## Package Organization

```
semstreams/
├── cmd/
│   ├── semstreams/          # Main binary
│   └── e2e/                 # E2E test runner
│
├── pkg/                     # Public utilities
│   ├── buffer/              # Ring buffers
│   ├── cache/               # LRU caching
│   ├── retry/               # Retry policies
│   ├── worker/              # Worker pools
│   └── timestamp/           # Time utilities
│
├── engine/                  # Flow orchestration
├── component/               # Component framework
├── componentregistry/       # Component registration
├── flowstore/               # Flow persistence
├── config/                  # Configuration system
│
├── input/                   # Protocol layer inputs
│   ├── udp/
│   └── websocket_input/
│
├── processor/               # Protocol layer processors
│   ├── json_filter/
│   ├── json_map/
│   ├── json_generic/
│   └── parser/
│
├── output/                  # Protocol layer outputs
│   ├── websocket/
│   ├── file/
│   └── httppost/
│
├── service/                 # HTTP services
├── metric/                  # Metrics system
├── health/                  # Health checks
├── errors/                  # Error framework
├── natsclient/              # NATS client
├── message/                 # Message types
│
├── semantic/                # Semantic layer (post-merge)
│   ├── entity/              # Entity framework
│   ├── graph/               # Graph structures
│   ├── vocabulary/          # Vocabularies
│   └── enrichment/          # Enrichment pipeline
│
├── storage/                 # Storage components
│   └── objectstore/         # NATS object store
│
├── test/                    # Testing infrastructure
│   └── e2e/                 # E2E test framework
│
├── configs/                 # Example configurations
└── docs/                    # Documentation
```

## Extension Mechanisms

### Protocol Layer Extension

Add new protocol components:

```go
// Register custom input
func RegisterTCPInput(registry *component.Registry) error {
    return registry.RegisterWithConfig(component.RegistrationConfig{
        Name:        "tcp",
        Factory:     CreateTCPInput,
        Schema:      tcpSchema,
        Type:        "input",
        Protocol:    "tcp",
        Domain:      "network",
        Description: "TCP socket input",
        Version:     "1.0.0",
    })
}

// Register custom processor
func RegisterProtobufParser(registry *component.Registry) error {
    return registry.RegisterWithConfig(component.RegistrationConfig{
        Name:        "protobuf",
        Factory:     CreateProtobufParser,
        Schema:      protobufSchema,
        Type:        "processor",
        Protocol:    "protobuf",
        Domain:      "processing",
        Description: "Protobuf message parser",
        Version:     "1.0.0",
    })
}
```

### Domain-Specific Extension

Create separate modules:

```
streamkit-robotics/
├── mavlink/              # MAVLink protocol
├── ros/                  # ROS integration
└── robotics_components/  # Domain processors

Imports: github.com/c360/semstreams
Registers: Domain-specific components
```

## Merge Strategy

### Current State (Post-Rename)

- ✅ Old semstreams → semstreams-old (archived)
- ✅ streamkit → semstreams (renamed)
- ✅ All import paths updated
- ✅ Documentation updated for two-layer architecture
- ⏳ Semantic components in semstreams-old ready to merge

### Merge Phases

**Phase 1: Documentation** (Current)
- Update philosophy sections
- Rename configurations
- Document unified architecture
- Archive outdated specs

**Phase 2: Preparation**
- Categorize components (protocol vs semantic)
- Extract domain code to separate modules
- Update component registry

**Phase 3: Merge Execution**
- Copy semantic packages to semstreams/semantic/
- Update component registration
- Add semantic flow configurations

**Phase 4: Validation**
- Protocol-only E2E tests pass
- Semantic E2E tests pass
- Performance validation
- Documentation complete

## Quality Gates

### Protocol Layer Independence

```bash
# Protocol flows work without semantic components
./bin/semstreams --config configs/protocol-flow.json

# No domain-specific code
! grep -r 'MAVLink|ROS|robotics|drone' semstreams/ --include="*.go"
```

### Semantic Layer Tests

```bash
# Entity extraction works
INTEGRATION_TESTS=1 go test ./semantic/entity/...

# Graph operations work
INTEGRATION_TESTS=1 go test ./semantic/graph/...
```

### Full Stack Tests

```bash
# E2E protocol tests
task e2e:protocol

# E2E semantic tests (post-merge)
task e2e:semantic
```

## Migration Guide

### For Protocol-Only Users

No changes required. Continue using protocol configurations:

```bash
./bin/semstreams --config configs/protocol-flow.json
```

### For Semantic Users (post-merge)

Use semantic configurations:

```bash
./bin/semstreams --config configs/semantic-flow.json
```

### For Domain-Specific Development

Create separate module:

1. Import semstreams packages
2. Implement domain components
3. Register with component registry
4. Build custom binary with domain components

## Success Criteria

- ✅ Protocol layer works independently
- ✅ Semantic layer extends protocol cleanly
- ✅ No domain assumptions in framework
- ✅ Clear extension points documented
- ✅ Both layers fully tested
- ✅ Migration path documented

## Future Roadmap

**v0.3.0**: Semantic merge complete
- Entity extraction components
- Graph processing
- Vocabulary system

**v0.4.0**: Advanced federation
- Multi-instance coordination
- Distributed graph queries
- Cross-instance entity resolution

**v1.0.0**: Production ready
- Complete test coverage
- Performance optimized
- Security hardened
- Documentation complete

---

*This architecture document is the source of truth for SemStreams design.*
