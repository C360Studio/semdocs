# Offline Operation

**Edge autonomy and sync recovery**

---

## Offline-First Design

Edge instances continue operating when disconnected from cloud.

**Key capabilities:**
- ✅ Local graph building continues
- ✅ Local queries work
- ✅ Events buffered for later sync
- ✅ Automatic reconnection
- ✅ Conflict-free sync on reconnect

---

## Buffering Strategy

### NATS JetStream Persistence

Messages persist locally during disconnection:

```json
{
  "nats": {
    "jetstream": {
      "enabled": true,
      "max_age": "7d",
      "max_bytes": "10GB"
    }
  }
}
```

**Behavior:**
- Events stored locally while offline
- Automatically sync when reconnected
- No message loss

---

## Reconnection Behavior

```
1. Detect disconnect
2. Continue local operation
3. Buffer outbound messages
4. Attempt reconnect (exponential backoff)
5. On reconnect: Sync buffered messages
6. Resume normal operation
```

**Configuration:**

```json
{
  "nats": {
    "max_reconnect": -1,
    "reconnect_wait": "2s"
  }
}
```

---

## Conflict Resolution

**Last-write-wins** for entity updates.

**Example:**

```
Edge (offline): UAV-001.battery = 15.4
Cloud: UAV-001.battery = 20.1

On reconnect:
  - Both updates visible
  - Latest timestamp wins
  - Version tracking prevents data loss
```

---

## Monitoring Offline Status

```bash
# Check connection status
curl http://localhost:8080/health

# Response (offline):
{
  "status": "degraded",
  "nats_connected": false,
  "buffered_messages": 142
}
```

---

## Best Practices

### Set Appropriate Buffer Limits

```json
{
  "jetstream": {
    "max_age": "7d",      // How long offline before data loss
    "max_bytes": "10GB"   // How much to buffer
  }
}
```

### Monitor Buffer Usage

```bash
curl http://localhost:9090/metrics | grep buffer
```

**Alert when buffer >80% full**

### Selective Sync for Bandwidth

```json
{
  "sync": {
    "subjects": ["events.alert.*"],  // Only critical events
    "filters": {"priority": {"gte": 3}}
  }
}
```

---

## Recovery Scenarios

### Short Disconnect (<1 hour)

- Buffer fills <10%
- Quick sync on reconnect
- No intervention needed

### Long Disconnect (1-24 hours)

- Buffer may fill 50-80%
- Sync takes minutes
- Monitor buffer metrics

### Extended Disconnect (>24 hours)

- Risk buffer overflow
- May need to increase buffer size
- Consider compression/filtering

---

## Next Steps

- **[Patterns](01-patterns.md)** - Edge deployment patterns
- **[Federation](03-federation.md)** - Edge-to-cloud connectivity
- **[Production Patterns](../advanced/04-production-patterns.md)** - Production deployment

---

**Edge continues offline** - Local processing, buffered sync, automatic recovery. Design for intermittent connectivity.
