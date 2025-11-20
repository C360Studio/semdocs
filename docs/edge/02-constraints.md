# Edge Resource Constraints

**Hardware guidelines and optimization for edge deployment**

---

## Hardware Recommendations

### Raspberry Pi 4 (8GB)

**Suitable for:**
- PathRAG only
- ~500 msg/sec
- 10K-50K entities

**Configuration:**
```json
{
  "workers": 2,
  "queue_size": 500,
  "indexer": {
    "indexes": {"predicate": true, "incoming": true},
    "embedding": {"enabled": false}
  }
}
```

**Memory:** ~500MB

---

### Intel NUC / Similar

**Suitable for:**
- PathRAG + optional GraphRAG
- ~2K msg/sec
- 100K-500K entities

**Configuration:**
```json
{
  "workers": 4,
  "queue_size": 2000,
  "indexer": {
    "indexes": {"predicate": true, "incoming": true, "spatial": true},
    "embedding": {"enabled": false}
  }
}
```

**Memory:** ~2GB

---

## Optimization Strategies

### Disable Unused Indexes

```json
{
  "indexes": {
    "alias": false,    // Disable if not using entity names
    "spatial": false,   // Disable if no geo queries
    "temporal": false   // Disable if no time queries
  }
}
```

**Memory savings:** ~100 bytes/entity

### Reduce Worker Count

```json
{"workers": 2}  // Raspberry Pi
```

**CPU savings:** Lower thread overhead

### Limit Queue Size

```json
{"queue_size": 500}  // Smaller buffer
```

**Memory savings:** Less message buffering

---

## Next Steps

- **[Patterns](01-patterns.md)** - Edge deployment patterns
- **[Federation](03-federation.md)** - Edge-to-cloud sync
- **[Performance Tuning](../advanced/02-performance-tuning.md)** - Optimization guide

---

**Optimize for your hardware** - Start minimal, add capabilities as needed.
