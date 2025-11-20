# SemDocs Reorganization Plan

**Status**: Planning
**Date**: 2025-01-19
**Goal**: Simplify SemStreams documentation from 816+ lines of buzzword-heavy content to <300 line approachable README with progressive disclosure

---

## Problem Statement

### Current Issues

**README.md (816 lines)**:
- ❌ Inverted information architecture (starts with advanced optimization)
- ❌ Buzzword overload: "distributed semantic stream processing platform" vs reality: "semantic graph runtime on message bus"
- ❌ Core concepts buried 200+ lines in
- ❌ Quick start is a footnote
- ❌ Uses "pipeline" terminology (linear) instead of event-based flow (non-linear)
- ❌ Semantic capabilities not explained until late, creating "why is it called SemStreams?" confusion

### Strategic Positioning

**Core Identity**:
- SemStreams = Semantic graph runtime built on event-based flows
- Graph building is THE product (not optional)
- Event routing is the mechanism, not the goal
- If you don't need graphs → use NATS directly (simpler, lighter)

**Target Audience**:
- Developers who need entity tracking across event streams
- Teams building edge-to-cloud architectures
- Users who want semantic search over streaming data
- NOT people looking for plain event bus (use NATS instead)

---

## New README Structure (<300 lines)

### 1. What is SemStreams? (60 lines)

**Opening Statement**:
```
SemStreams is a semantic graph runtime built on event-based flows.

As events flow through components, the runtime automatically builds entity graphs,
building semantic graphs from your event streams.

Unlike linear data flows, components can connect anywhere - multiple inputs,
branching outputs, flexible routing based on your needs.
```

**What does "building semantic graphs" mean?**
```
As events flow through your components, SemStreams automatically:

• Tracks entities mentioned in events (drones, sensors, users, etc.)
• Indexes relationships between entities (drone belongs_to fleet)
• Enables queries across entity connections (find all drones in fleet X)
• Optionally adds semantic search (find similar events by meaning)

Example:
  Event: {"drone_id": "UAV-001", "fleet": "rescue", "battery": 15}

  Graph automatically indexes:
  - Entity: UAV-001 (type: drone)
  - Relationship: UAV-001 belongs_to rescue (type: fleet)
  - Attribute: battery=15 (enables "find all low battery drones")

  Query later: "Show me all drones in rescue fleet with battery < 20%"
  Or: "Find events similar to this emergency pattern" (semantic search)

This happens in the background. You just route events - the graph builds itself.
```

**Core capabilities**:
```
• Event routing and transformation via message bus
• Automatic entity graph indexing from event streams
• Relationship tracking (spatial, temporal, predicates)
• Edge-to-cloud federation with selective sync
• Optional semantic enhancements:
  - SemEmbed: Vector similarity search
  - SemSummarize: AI-powered summarization
• Component-based architecture
• Real-time stream processing
```

### 2. Why Message Bus? (50 lines)

**Traditional Approaches**:
```
• Tight coupling between processing stages
• Linear pipelines - rigid, hard to extend
• Point-to-point connections don't scale
```

**Message Bus Pattern**:
```
• Loose coupling - components don't know about each other
• Publish/subscribe - add new consumers without changing producers
• Non-linear flows - route events based on content, not topology
• Independent scaling - scale busy components independently
```

**Implementation: NATS + JetStream** (appears ~50 lines in):
```
SemStreams uses NATS with JetStream for the message bus.

Why NATS?
✓ Lightweight - minimal overhead, fast message delivery
✓ Built-in persistence - JetStream provides durable streams
✓ Key-value storage - no external database for state
✓ Geographical distribution - multi-region federation built-in
✓ Developer friendly - simple ops, no Kafka/Zookeeper complexity

vs. Kafka: NATS is simpler to operate, lower resource overhead
vs. RabbitMQ: NATS has better clustering, simpler configuration
vs. Redis Streams: NATS has better persistence guarantees

Don't know NATS? No problem - SemStreams config abstracts it:
• Declare subjects (topics) in component config
• Components handle subscribe/publish automatically
• Optional: Direct NATS access for advanced use cases
```

### 3. Quick Start: Event Flow Example (80 lines)

**Basic Event Flow**:
```json
{
  "components": {
    "udp_listener": {
      "type": "input",
      "outputs": [{"subject": "raw.telemetry"}]
    },
    "json_parser": {
      "type": "processor",
      "inputs": [{"subject": "raw.telemetry"}],
      "outputs": [{"subject": "parsed.telemetry"}]
    },
    "file_writer": {
      "type": "output",
      "inputs": [{"subject": "parsed.telemetry"}]
    }
  }
}
```

**Non-Linear Flow Example**:
```
UDP Listener
    ↓ publish
Message Bus (NATS)
    ↓ subscribe
JSON Parser
    ↓ publish
Message Bus (NATS)
    ├─ subscribe → File Writer
    └─ subscribe → Graph Indexer → Semantic Search
```

**Shows**:
- Multiple subscribers on same subject
- Content-based routing
- Components added without changing producers

### 4. Progressive Enhancement (60 lines)

**Layer 1: Automatic Graph Building (CORE - Always On)**
```
✓ Events → Entity extraction
✓ Entities → Relationship indexing
✓ Native semantic indexing (Go algorithms, no external services)
✓ Queries across entity connections
Disable this? Use NATS instead - that's the whole point.
```

**Layer 2: Enhanced Indexing (OPTIONAL)**
```
✓ Spatial indexes (location-based queries)
✓ Temporal indexes (time-based queries)
✓ Predicate indexes (relationship types)
✓ Alias indexes (entity name resolution)
Configure what you need, skip what you don't.
```

**Layer 3: Semantic Enhancements (OPTIONAL - Better, Not Required)**
```
✓ SemEmbed - Enhanced vector embeddings (better than native)
  - Upgrades semantic similarity search quality
  - Resource: ~2GB RAM, GPU optional
  - Native semantic still works without it

✓ SemSummarize - AI-powered summarization (richer than native)
  - Upgrades entity/event summarization quality
  - Resource: LLM integration (local or any OpenAI API-compatible service)
  - Native summarization still works without it
```

**Layer 4: Edge Federation (OPTIONAL - Deployment Pattern)**
```
✓ Lightweight runtime at edge (native algorithms, no services)
✓ Full runtime on laptop/server (with or without enhancement services)
✓ Resource-constrained edge (Raspberry Pi) runs native only
✓ Selective sync, bi-directional federation
```

### 5. Edge-First Architecture (50 lines)

**Why Edge Matters**:
```
SemStreams is designed for edge deployment, not just centralized processing:

Typical Pattern:
┌─────────────────────────────────────────┐
│ EDGE (Drone, Vehicle, IoT Gateway)     │
│ • Lightweight SemStreams runtime        │
│ • Local graph building from sensors     │
│ • Local queries (low latency)           │
│ • Selective sync (bandwidth-aware)      │
└──────────────┬──────────────────────────┘
               │ Filtered events only
               ↓
┌─────────────────────────────────────────┐
│ LAPTOP/SERVER (Central Processing)     │
│ • Full SemStreams + semantic services   │
│ • Cross-edge aggregation                │
│ • Long-term graph storage               │
│ • Semantic search across all edges      │
└─────────────────────────────────────────┘

Why This Matters:
✓ Process at the edge, decide what goes upstream (not everything)
✓ Low-latency local queries (don't wait for server round-trip)
✓ Bandwidth efficiency (send summaries, not raw streams)
✓ Offline operation (edge continues when disconnected)
✓ Privacy/compliance (sensitive data stays local)

Example: Fleet Management
- Edge: Each vehicle runs SemStreams, tracks local entities, triggers alerts
- Server: Aggregates fleet-wide graphs, runs semantic analysis, fleet optimization
- Sync: Only send exceptions (low battery, geofence violations, anomalies)
```

### 6. When to Use / When Not to Use (30 lines)

**Use SemStreams when**:
```
✓ Entity graphs from event streams
✓ Relationships matter (not just individual events)
✓ Edge deployment (process locally, sync selectively)
✓ Temporal tracking (entity state over time)
✓ Optional: Semantic search, AI summarization
```

**Use NATS directly when**:
```
✓ Simple pub/sub event routing
✓ No entity tracking needed
✓ No edge requirements
✓ You just need message delivery
```

**Honest Positioning**:
```
Can you disable graph indexing? Technically yes.
Should you? No - you're using the wrong tool.

SemStreams = Graph runtime + Event flows
NATS = Event flows only

If you don't need graphs, NATS is simpler and lighter.
The "Sem" in SemStreams is the semantic graph, not optional add-on.
```

### 7. What's Next (20 lines)

**Links to detailed guides**:
```
→ Basics Guide - Component types, event routing, configuration
→ Graph Guide - Entity relationships, queries, indexing
→ Semantic Guide - Native vs enhanced, when to use which
→ Edge Guide - Federation patterns, sync strategies
→ Performance Guide - Optimization, caching, scaling
→ API Reference - Component schemas, REST endpoints
```

---

## Separate Documentation Files

### `/docs/basics/`

**01-components.md** - Component types and lifecycle
- Input components (UDP, HTTP, WebSocket, NATS)
- Processor components (Transform, Filter, Map, Rule)
- Output components (File, HTTP POST, WebSocket)
- Storage components (Object Store, KV)
- Gateway components (HTTP API)
- Component lifecycle (Start, Run, Stop, Health)

**02-event-routing.md** - Event flows and NATS subjects
- Subject patterns (wildcards, hierarchies)
- Port configuration (inputs, outputs, interfaces)
- Non-linear flows (fan-out, fan-in, conditional routing)
- Message interfaces (type safety between components)

**03-configuration.md** - JSON config structure
- Platform section (org, id, region, environment)
- NATS connection (URLs, credentials)
- Services (metrics, discovery, message logging)
- Components (wiring, configuration)

**04-first-flow.md** - Hands-on tutorial
- UDP → JSON parser → File output
- Adding graph indexing
- Querying entities
- Non-linear example (multiple outputs)

### `/docs/graph/`

**01-entities.md** - Entity model
- Entity structure (ID, type, class, role)
- Entity classes (Object, Event, Relationship, Abstract)
- Entity roles (Primary, Secondary, Derived)
- Source attribution (where entity came from)

**02-relationships.md** - Predicates and edges
- Predicate types (spatial, temporal, semantic)
- Edge creation (automatic vs explicit)
- Relationship queries (traversal, filtering)

**03-queries.md** - Querying the graph
- Entity lookup (by ID, by type)
- Relationship queries (find all X connected to Y)
- Temporal queries (entity state at time T)
- Spatial queries (entities within radius)

**04-indexing.md** - Index types
- Alias index (entity name resolution)
- Predicate index (relationship types)
- Incoming index (reverse relationships)
- Spatial index (location-based queries)
- Temporal index (time-based queries)

### `/docs/semantic/`

**01-overview.md** - Semantic features overview
- Native semantic (Go algorithms, always on)
- Enhanced semantic (SemEmbed, SemSummarize)
- When to use which

**02-native.md** - Native semantic capabilities
- Template-based summarization
- Basic similarity search (Go algorithms)
- Works everywhere (edge, laptop, server)
- No external dependencies

**03-semembed.md** - Enhanced vector embeddings
- What SemEmbed provides (transformer models)
- Resource requirements (~2GB RAM, GPU optional)
- Configuration and setup
- Quality improvements vs native

**04-semsummarize.md** - AI-powered summarization
- What SemSummarize provides (LLM integration)
- OpenAI API compatibility (swap with any compatible service)
- Resource requirements (local or API)
- Configuration and setup
- Use cases (entity history, event consolidation)

**05-when-to-enhance.md** - Decision guide
- Native is good enough when...
- SemEmbed adds value when...
- SemSummarize adds value when...
- Resource vs quality trade-offs

### `/docs/edge/`

**01-edge-patterns.md** - Edge deployment patterns
- Edge-only (standalone operation)
- Edge-to-server (selective sync)
- Edge-to-edge (peer federation)
- Multi-tier (edge → regional → central)

**02-resource-constraints.md** - Running on constrained devices
- Raspberry Pi deployment
- Native-only mode (no enhancement services)
- Memory optimization
- Storage management

**03-federation.md** - Edge-to-server federation
- Selective sync (filter, aggregate, summarize)
- Bi-directional communication
- Conflict resolution
- Bandwidth optimization

**04-offline-operation.md** - Disconnected edge
- Local graph building
- Local queries
- Sync when reconnected
- Queue management

### `/docs/advanced/`

**01-performance.md** - **MOVED HERE FROM README**
- PathRAG vs GraphRAG decision trees
- Query optimization
- Caching strategies
- Batch processing

**02-nats-integration.md** - Direct NATS usage
- Subject naming conventions
- JetStream streams
- Key-value storage
- Consumer groups

**03-monitoring.md** - Metrics and observability
- Prometheus metrics
- Grafana dashboards
- Health checks
- Message logging

**04-custom-components.md** - Building components
- Component interface
- Port configuration
- Health reporting
- Testing

---

## Terminology Guidelines

### Throughout All Documentation

**✅ Use**:
- "Event-based flow" (how data moves)
- "Event flow runtime" (what SemStreams is)
- "Message bus" (generic concept)
- "Component connections" (how pieces fit)
- "Pub/sub pattern" (architectural concept)
- "Event routing" (directing messages)
- "Semantic graph runtime" (product identity)
- "Native semantic" (Go algorithms, built-in)
- "Enhanced semantic" (SemEmbed, SemSummarize)
- "Edge" vs "resource-constrained" (Raspberry Pi)
- "Laptop/server" (sufficient resources)

**❌ Avoid**:
- "Pipeline" (implies linear)
- "Data pipeline" (too rigid)
- "ETL pipeline" (batch processing connotation)
- "Stage" (implies sequence)
- "Cloud-only" (works on laptop too)
- "Optional semantic" (native is always on)

### NATS-Specific References

- Mention NATS as implementation detail
- Explain why NATS (lightweight, simple ops, built-in features)
- Keep NATS terminology optional for basic usage
- Deep NATS knowledge only needed for advanced scenarios

---

## Content Migration Plan

### Phase 1: Create New README (<300 lines)

**Tasks**:
1. Write new opening (60 lines) - graph runtime identity
2. Add message bus explanation (50 lines) - why NATS
3. Create quick start example (80 lines) - hands-on
4. Add progressive enhancement (60 lines) - native → enhanced
5. Add edge-first section (50 lines) - deployment patterns
6. Add when to use (30 lines) - honest positioning
7. Add what's next links (20 lines)

**Total**: ~300 lines, focused, approachable

### Phase 2: Move Advanced Content (PRESERVE, DON'T DELETE)

**Extract from current README**:
- PathRAG vs GraphRAG → `/docs/advanced/01-performance.md`
- Optimization details → `/docs/advanced/01-performance.md`
- Deep NATS usage → `/docs/advanced/02-nats-integration.md`
- Monitoring setup → `/docs/advanced/03-monitoring.md`

**Preserve Existing Valuable Content**:
- Algorithm reference (user favorite) → `/docs/advanced/05-algorithm-reference.md`
- Good guides and snippets → Review and place in appropriate advanced sections
- Code examples → Keep in relevant topic areas
- Don't delete anything valuable - move to appropriate advanced/reference sections

### Phase 3: Create Basic Guides

**New files**:
- `/docs/basics/01-components.md` - Component types reference
- `/docs/basics/02-event-routing.md` - Subject patterns, non-linear flows
- `/docs/basics/03-configuration.md` - JSON config structure
- `/docs/basics/04-first-flow.md` - Hands-on tutorial

### Phase 4: Create Graph Guides

**New files**:
- `/docs/graph/01-entities.md` - Entity model
- `/docs/graph/02-relationships.md` - Predicates and edges
- `/docs/graph/03-queries.md` - Querying the graph
- `/docs/graph/04-indexing.md` - Index types

### Phase 5: Create Semantic Guides

**New files**:
- `/docs/semantic/01-overview.md` - Native vs enhanced
- `/docs/semantic/02-native.md` - Go algorithms, always on
- `/docs/semantic/03-semembed.md` - Enhanced embeddings
- `/docs/semantic/04-semsummarize.md` - AI summarization
- `/docs/semantic/05-when-to-enhance.md` - Decision guide

### Phase 6: Create Edge Guides

**New files**:
- `/docs/edge/01-edge-patterns.md` - Deployment patterns
- `/docs/edge/02-resource-constraints.md` - Raspberry Pi, IoT
- `/docs/edge/03-federation.md` - Edge-to-server sync
- `/docs/edge/04-offline-operation.md` - Disconnected edge

---

## Success Metrics

### Clarity
- ✅ Can a developer understand "semantic graph runtime on event flows" in <5 minutes?
- ✅ Is the non-linear nature clear from examples?
- ✅ Is the graph-first positioning obvious?

### Approachability
- ✅ Can someone start with native semantic (no external services)?
- ✅ Is edge deployment pattern clear and prominent?
- ✅ Are resource requirements explicit (laptop vs Raspberry Pi)?

### Progressive
- ✅ Does each layer build on the previous?
- ✅ Can advanced users skip to what they need?
- ✅ Is "native → enhanced" progression clear?

### Accuracy
- ✅ Does messaging match technical reality?
- ✅ Are resource requirements honest?
- ✅ Is the "graph is the product" message clear?
- ✅ Is NATS positioned correctly (implementation, not identity)?

### Strategic
- ✅ Does it attract the right users (graph needs)?
- ✅ Does it redirect wrong users (plain event bus → use NATS)?
- ✅ Is edge use case prominent (not buried)?
- ✅ Is semantic capability explained (justifies "Sem" in name)?

---

## Implementation Notes

### Do NOT Implement Yet

This is a **planning document only**. Do not:
- ❌ Modify existing README.md
- ❌ Create new documentation files
- ❌ Move content between files

### Next Steps

1. Review this plan with stakeholders
2. Validate terminology choices
3. Confirm progressive enhancement layers
4. Approve edge-first positioning
5. Then proceed with implementation in phases

### Execution Order

When approved:
1. Create new README.md (Phase 1)
2. Test with external reviewers (fresh eyes)
3. Iterate on README based on feedback
4. Only then start Phase 2-6 (detailed guides)

This ensures the core positioning (README) is solid before creating detailed documentation.

---

## Open Questions

1. Should we create a separate "Why Not NATS Directly?" page or keep in README?
2. How much detail about Go native algorithms in semantic docs?
3. Should edge federation have its own top-level section (not under "advanced")?
4. Do we need a migration guide for users of old documentation?

---

**End of Specification**
