# Rules Engine

Ingestion-time intelligence for dynamic knowledge graphs

---

## What is the Rules Engine?

The Rules Engine adds **conditional logic and pattern detection** as data flows through SemStreams. While PathRAG and GraphRAG provide query-time intelligence, rules enable automatic graph enrichment at ingestion time.

**Without Rules:** Static entity extraction only

**With Rules:** Conditional entities, derived relationships, semantic enrichment, pattern detection

---

## How It Works

```text
Ingestion Time                         Query Time
────────────────────────              ──────────────────────
Stream → Parse → Rules Engine →       Graph → PathRAG/GraphRAG
        (extract) (enrich/decide)    (store)   (query)
```

### Rule Lifecycle

1. **Entity arrives** via NATS KV bucket update
2. **Rules Engine checks:** Does entity pattern match?
3. **Matching rules evaluate conditions**
4. **Conditions pass** → Execute event generation
5. **Events published** to NATS graph subjects
6. **Cooldown period** prevents re-triggering

---

## Use Cases

### 1. Status Monitoring & Alerts

**Scenario:** Create alert entities when battery drops below threshold

**Example:**
- Drone battery < 20% → Create "low battery" alert entity
- Temperature > 80°C → Create "overheat" alert
- Connection lost > 5 minutes → Create "offline" alert

### 2. Threshold Detection

**Scenario:** Track when values cross thresholds

**Example:**
- Altitude > 400 feet → Tag entity as "high altitude"
- Speed < 1 m/s → Tag entity as "stationary"
- Battery charging → Update entity status

### 3. Pattern Matching

**Scenario:** Detect patterns across multiple entities

**Example:**
- Multiple drones in same area → Create "swarm" entity
- Repeated errors from device → Create "maintenance needed" alert
- Geo-fence violations → Create "boundary breach" alert

### 4. Derived Relationships

**Scenario:** Automatically create relationships between entities

**Example:**
- Drone near operator → Create "nearby" relationship
- Device in fleet → Create "member of" relationship
- Sensor on platform → Create "mounted on" relationship

---

## Configuration

### Enable Rules Engine

Add to your `config.json`:

```json
{
  "components": {
    "rule-processor": {
      "type": "processor",
      "enabled": true,
      "config": {
        "ports": {
          "inputs": [{
            "subject": "events.graph.entity.>",
            "type": "jetstream"
          }],
          "outputs": [{
            "subject": "events.graph.{event_type}",
            "type": "jetstream"
          }]
        },
        "rules": [
          {
            "name": "battery-alert",
            "type": "battery_threshold",
            "enabled": true,
            "config": {
              "entity_pattern": "*.*.robotics.*.drone.*",
              "threshold": 20.0,
              "cooldown": "5m"
            }
          }
        ]
      }
    }
  }
}
```

---

## Built-In Rule Types

SemStreams includes several built-in rule types:

### Battery Threshold Rule

**Type:** `battery_threshold`

**Purpose:** Monitor battery levels and create alerts

**Configuration:**
```json
{
  "name": "low-battery",
  "type": "battery_threshold",
  "enabled": true,
  "config": {
    "entity_pattern": "*.*.robotics.*.drone.*",
    "threshold": 20.0,
    "cooldown": "5m",
    "severity": "warning"
  }
}
```

**Generated Events:**
- Creates alert entity: `alert.battery.{device_id}`
- Properties: severity, battery_level, device_id, timestamp
- Relationship: alert → originatedFrom → drone

### Threshold Monitor Rule

**Type:** `threshold_monitor`

**Purpose:** Generic threshold monitoring for any numeric field

**Configuration:**
```json
{
  "name": "altitude-monitor",
  "type": "threshold_monitor",
  "enabled": true,
  "config": {
    "entity_pattern": "*.*.robotics.*.drone.*",
    "field": "altitude",
    "operator": "gt",
    "threshold": 400.0,
    "cooldown": "1m",
    "tag": "high-altitude"
  }
}
```

**Operators:** `gt` (>), `lt` (<), `gte` (>=), `lte` (<=), `eq` (=), `neq` (!=)

---

## Rule Anatomy

Every rule has four parts:

### 1. Trigger

**When to evaluate:**
```json
{
  "entity_pattern": "*.*.robotics.*.drone.*",
  "kv_buckets": ["graph_entities"]
}
```

- **entity_pattern**: Glob pattern for entity IDs to watch
- **kv_buckets**: NATS KV buckets to monitor

### 2. Conditions

**What to check:**
```json
{
  "field": "battery.level",
  "operator": "lt",
  "threshold": 20.0
}
```

### 3. Event Generation

**What to emit:**
- Creates new entities
- Updates existing entities
- Creates relationships
- Adds semantic tags

### 4. Metadata

**Configuration:**
```json
{
  "cooldown": "5m",
  "description": "Battery low alert",
  "tags": ["monitoring", "battery"]
}
```

**Cooldown:** Prevents re-triggering for specified duration

---

## Example: Complete Flow with Rules

### Configuration

```json
{
  "components": {
    "udp": {
      "type": "input",
      "enabled": true,
      "config": {
        "port": 14550,
        "ports": {
          "outputs": [{"subject": "raw.udp"}]
        }
      }
    },
    "json_generic": {
      "type": "processor",
      "enabled": true,
      "config": {
        "ports": {
          "inputs": [{"subject": "raw.udp"}],
          "outputs": [{"subject": "parsed.json"}]
        }
      }
    },
    "graph": {
      "type": "processor",
      "enabled": true,
      "config": {
        "input_subject": "parsed.json"
      }
    },
    "rule-processor": {
      "type": "processor",
      "enabled": true,
      "config": {
        "ports": {
          "inputs": [{"subject": "events.graph.entity.>"}]
        },
        "rules": [
          {
            "name": "battery-alert",
            "type": "battery_threshold",
            "enabled": true,
            "config": {
              "entity_pattern": "*.*.robotics.*.drone.*",
              "threshold": 20.0,
              "cooldown": "5m"
            }
          },
          {
            "name": "high-altitude",
            "type": "threshold_monitor",
            "enabled": true,
            "config": {
              "entity_pattern": "*.*.robotics.*.drone.*",
              "field": "altitude",
              "operator": "gt",
              "threshold": 400.0,
              "cooldown": "1m",
              "tag": "high-altitude"
            }
          }
        ]
      }
    }
  }
}
```

### What Happens

1. **UDP receives telemetry** → `raw.udp`
2. **JSON parser extracts data** → `parsed.json`
3. **Graph processor creates/updates drone entity** → `events.graph.entity.drone`
4. **Rules engine evaluates:**
   - Battery < 20%? → Create alert entity
   - Altitude > 400ft? → Tag entity as "high-altitude"
5. **Alert entities and tags added to graph**
6. **PathRAG/GraphRAG can query** enriched graph

---

## Performance

### Evaluation Budget

Rules are evaluated with strict timing constraints:

- **Per-rule budget:** 10ms
- **Total budget:** 100ms
- **Cooldown:** Prevents excessive re-evaluation

### Optimization

- Rules only evaluate when entity pattern matches
- Cooldown periods prevent re-triggering
- Parallel rule evaluation (when safe)
- Efficient NATS KV bucket watching

### Metrics

Monitor rule performance:

```bash
curl http://localhost:9090/metrics | grep rule_
```

**Key Metrics:**
- `rule_evaluations_total` - Total evaluations
- `rule_matches_total` - Conditions matched
- `rule_events_generated_total` - Events created
- `rule_evaluation_duration_seconds` - Evaluation latency

---

## Best Practices

### 1. Use Specific Entity Patterns

```json
// ✅ Good - Specific pattern
"entity_pattern": "acme.platform1.robotics.fleet1.drone.*"

// ❌ Bad - Too broad (performance impact)
"entity_pattern": "*.*.*.*.*.*"
```

### 2. Set Appropriate Cooldowns

```json
// ✅ Good - Prevents alert spam
"cooldown": "5m"

// ❌ Bad - May generate too many alerts
"cooldown": "10s"
```

### 3. Keep Conditions Simple

```json
// ✅ Good - Single clear condition
{
  "field": "battery.level",
  "operator": "lt",
  "threshold": 20.0
}

// ❌ Bad - Complex nested conditions (not supported)
// Use separate rules instead
```

### 4. Meaningful Rule Names

```json
// ✅ Good
"name": "battery-critical-alert"

// ❌ Bad
"name": "rule1"
```

---

## Troubleshooting

### Rules Not Triggering

**Check:**
1. Rule enabled: `"enabled": true`
2. Entity pattern matches: Use `*` wildcards correctly
3. Conditions accurate: Field names and operators correct
4. Cooldown expired: Wait for cooldown period

**Debug:**
```bash
# View rule evaluations
docker compose logs rule-processor | grep "evaluate"

# Check entity patterns
curl http://localhost:8080/api/v1/components/rule-processor
```

### Performance Issues

**Symptoms:**
- Slow event processing
- High CPU usage
- Metrics show long evaluation times

**Solutions:**
- Narrow entity patterns
- Increase cooldown periods
- Reduce number of active rules
- Check for expensive conditions

---

## Custom Rules

For advanced use cases, you can create custom rule types in Go.

**See:** [Writing Custom Rules](../../examples/custom-rules/) (TODO) for implementation guide

**Developer Documentation:**
- `docs/guides/rules-engine.md` - Complete implementation guide
- `docs/specs/SPEC-001-generic-rules-engine.md` - Architecture specification

---

## Next Steps

**Basic Usage:**
- [Components](02-components.md) - Other processor types
- [Message System](05-message-system.md) - Understanding message flow
- [Graph Operations](../graph/) - Working with enriched graphs

**Advanced Topics:**
- [PathRAG vs GraphRAG](../advanced/01-pathrag-graphrag-decisions.md) - Query-time intelligence
- [Performance Tuning](../advanced/02-performance-tuning.md) - Optimizing rule evaluation
- [Production Patterns](../advanced/04-production-patterns.md) - Production deployment

---

**Key Takeaway:** Rules Engine provides ingestion-time intelligence for dynamic knowledge graphs. Use it to create alerts, detect patterns, and automatically enrich your graph as data flows through the system.
