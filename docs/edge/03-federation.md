# Edge Federation

WebSocket-based edge-to-cloud connectivity

---

## WebSocket Federation Pattern

SemStreams federation uses WebSocket output and input components to connect edge instances to central cloud hubs.

```text
Edge NATS (local processing)
  ↓ WebSocket Output component
Cloud NATS (central hub)
  ← WebSocket Input component
  → Receives filtered events
```

**Benefits:**
- Edge operates independently
- Selective event filtering at source
- Automatic reconnection with backpressure
- TLS/mTLS for secure communication
- Simple configuration

---

## Edge Configuration

```json
{
  "components": [
    {
      "id": "edge-to-cloud",
      "type": "websocket_output",
      "config": {
        "port": 8080,
        "path": "/edge-data",
        "filters": {
          "priority": {"gte": 3}
        }
      }
    }
  ],
  "security": {
    "tls": {
      "server": {
        "enabled": true,
        "cert_file": "/certs/server.crt",
        "key_file": "/certs/server.key"
      }
    }
  }
}
```

---

## Cloud Hub Configuration

```json
{
  "components": [
    {
      "id": "receive-from-edge",
      "type": "websocket_input",
      "config": {
        "url": "wss://edge-device:8080/edge-data",
        "reconnect": true,
        "reconnect_interval": "5s"
      }
    }
  ],
  "security": {
    "tls": {
      "client": {
        "ca_files": ["/certs/ca.crt"]
      }
    }
  }
}
```

---

## Selective Sync

Only sync critical events:

```json
{
  "components": [
    {
      "id": "filtered-output",
      "type": "websocket_output",
      "config": {
        "port": 8080,
        "path": "/edge-data",
        "filters": {
          "event_type": ["alert", "anomaly"],
          "battery.level": {"lte": 20},
          "priority": {"gte": 3}
        }
      }
    }
  ]
}
```

**Result:** Bandwidth efficiency, only exceptions sync to cloud.

---

## Bidirectional Communication

### Edge → Cloud

Alerts, anomalies, exceptions

### Cloud → Edge

Config updates, commands, queries (requires bidirectional WebSocket setup)

---

## Connection Resilience

WebSocket components handle connection failures gracefully:

- **Automatic reconnection** with exponential backoff
- **Backpressure support** when cloud is unavailable
- **Local buffering** continues at edge during disconnections
- **Resume on reconnect** picks up from last successfully sent event

---

## Next Steps

- **[Patterns](01-patterns.md)** - Edge deployment patterns
- **[Offline Operation](04-offline.md)** - Handling disconnections
- **[Production Patterns](../advanced/04-production-patterns.md)** - Federation at scale
- **[Federation Guide](../guides/federation.md)** - Secure multi-location setup with mTLS

---

**WebSocket federation enables edge autonomy** - Process locally, sync selectively, operate offline.
