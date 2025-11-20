# Edge Deployment Patterns

**Running SemStreams at the edge with selective cloud sync**

---

## Edge-to-Cloud Architecture

```text
┌─────────────────────────────────────┐
│ EDGE (Vehicle, Drone, IoT Gateway) │
│ • PathRAG only (lightweight)       │
│ • Local graph, low latency         │
│ • Selective sync (exceptions only) │
└──────────────┬──────────────────────┘
               │ WebSocket Output
               ▼
┌─────────────────────────────────────┐
│ CLOUD (Laptop, Server)              │
│ • WebSocket Input                   │
│ • Full stack (PathRAG + GraphRAG)   │
│ • Aggregate multi-edge data         │
│ • Cross-edge queries & analytics    │
└─────────────────────────────────────┘
```

---

## Edge Configuration

### Minimal (Raspberry Pi)

```json
{
  "platform": {
    "type": "edge",
    "region": "vehicle-001"
  },
  "graph": {
    "config": {
      "workers": 2,
      "queue_size": 500,
      "indexer": {
        "indexes": {
          "predicate": true,
          "incoming": true
        },
        "embedding": {"enabled": false}
      }
    }
  }
}
```

**Resources:**
- CPU: 2 cores
- Memory: ~500MB
- Throughput: ~500 msg/sec

---

## Sync Patterns

### Selective Sync

Only send exceptions to cloud:

```json
{
  "federation": {
    "sync": {
      "subjects": ["events.alert.*"],
      "filters": {
        "priority": {"gte": 3},
        "battery.level": {"lte": 20}
      }
    }
  }
}
```

**Saves bandwidth** - Only critical events sync.

---

### Bidirectional Sync

Edge ⟷ Cloud sync:

```
Edge → Cloud: Alerts, anomalies
Cloud → Edge: Config updates, commands
```

---

## Use Cases

- **Fleet Management:** Vehicles with local processing, cloud aggregation
- **IoT Networks:** Sensors at edge, analytics in cloud
- **Distributed Sensing:** Drones/robots with local decisions, cloud coordination
- **Offline-First:** Edge continues when disconnected

---

## Next Steps

- **[Resource Constraints](02-constraints.md)** - Edge hardware guidelines
- **[Federation](03-federation.md)** - NATS leaf node setup
- **[Offline Operation](04-offline.md)** - Handling disconnections

---

**Process at the edge** - Low latency, bandwidth efficiency, offline capability.
