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
      "name": "rule-processor",
      "enabled": true,
      "config": {
        "ports": {
          "inputs": [
            {
              "name": "entity_states",
              "type": "kv-watch",
              "required": true,
              "description": "Watch entity state changes from ENTITY_STATES KV bucket"
            }
          ],
          "outputs": [
            {
              "name": "control_commands",
              "type": "nats",
              "subject": "control.*.commands",
              "description": "Control commands based on rules"
            }
          ]
        },
        "rules_files": ["rules/battery-alerts.json"],
        "entity_watch_patterns": ["*.*.robotics.*.drone.*"],
        "enable_graph_integration": true
      }
    }
  }
}
```

---

## Rule Definition Structure

Rules use a **condition-based expression system**. Each rule defines conditions that evaluate against entity state.

### Rule Definition

```json
{
  "id": "low-battery-alert",
  "type": "alert",
  "name": "Low Battery Alert",
  "description": "Alert when drone battery drops below 20%",
  "enabled": true,
  "entity": {
    "pattern": "*.*.robotics.*.drone.*",
    "watch_buckets": ["ENTITY_STATES"]
  },
  "conditions": [
    {
      "field": "robotics.battery.level",
      "operator": "lte",
      "value": 20.0,
      "required": true
    }
  ],
  "logic": "and",
  "cooldown": "5m",
  "metadata": {
    "severity": "warning",
    "category": "battery"
  }
}
```

### Condition Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `eq` | Equal | `{"field": "status", "operator": "eq", "value": "active"}` |
| `ne` | Not equal | `{"field": "status", "operator": "ne", "value": "offline"}` |
| `lt` | Less than | `{"field": "battery", "operator": "lt", "value": 20.0}` |
| `lte` | Less than or equal | `{"field": "battery", "operator": "lte", "value": 20.0}` |
| `gt` | Greater than | `{"field": "altitude", "operator": "gt", "value": 400.0}` |
| `gte` | Greater than or equal | `{"field": "speed", "operator": "gte", "value": 50.0}` |
| `between` | In range | `{"field": "temp", "operator": "between", "value": [10, 30]}` |
| `contains` | String contains | `{"field": "name", "operator": "contains", "value": "rescue"}` |
| `starts_with` | String prefix | `{"field": "id", "operator": "starts_with", "value": "UAV"}` |
| `ends_with` | String suffix | `{"field": "type", "operator": "ends_with", "value": "drone"}` |
| `regex` | Regex match | `{"field": "callsign", "operator": "regex", "value": "^ALPHA-\\d+$"}` |
| `in` | In array | `{"field": "status", "operator": "in", "value": ["active", "standby"]}` |
| `not_in` | Not in array | `{"field": "mode", "operator": "not_in", "value": ["disabled"]}` |

### Logic Operators

Combine multiple conditions:

- `"logic": "and"` - All conditions must be true (default)
- `"logic": "or"` - Any condition can be true

### Example: Battery Alert Rule

```json
{
  "id": "battery-critical",
  "type": "alert",
  "name": "Critical Battery Alert",
  "description": "Alert when battery is critically low and drone is airborne",
  "enabled": true,
  "entity": {
    "pattern": "*.*.robotics.*.drone.*"
  },
  "conditions": [
    {
      "field": "robotics.battery.level",
      "operator": "lte",
      "value": 10.0,
      "required": true
    },
    {
      "field": "robotics.flight.armed",
      "operator": "eq",
      "value": true,
      "required": false
    }
  ],
  "logic": "and",
  "cooldown": "2m",
  "metadata": {
    "severity": "critical",
    "action": "land_immediately"
  }
}
```

### Example: Altitude Monitor Rule

```json
{
  "id": "high-altitude-warning",
  "type": "warning",
  "name": "High Altitude Warning",
  "description": "Warn when drone exceeds safe altitude",
  "enabled": true,
  "entity": {
    "pattern": "*.*.robotics.*.drone.*"
  },
  "conditions": [
    {
      "field": "geo.location.altitude",
      "operator": "gt",
      "value": 120.0
    }
  ],
  "logic": "and",
  "cooldown": "1m"
}
```

---

## Rule Anatomy

Every rule has four parts:

### 1. Entity Matching

**Which entities to watch:**

```json
{
  "entity": {
    "pattern": "*.*.robotics.*.drone.*",
    "watch_buckets": ["ENTITY_STATES"]
  }
}
```

- **pattern**: Glob pattern matching 6-part entity IDs
- **watch_buckets**: KV buckets to monitor (defaults to ENTITY_STATES)

### 2. Conditions

**What to check (uses predicate fields):**

```json
{
  "conditions": [
    {
      "field": "robotics.battery.level",
      "operator": "lte",
      "value": 20.0,
      "required": true
    }
  ],
  "logic": "and"
}
```

- **field**: Predicate path in dotted notation (from vocabulary registry)
- **operator**: Comparison operator (see table above)
- **value**: Value to compare against
- **required**: If false, missing field doesn't fail evaluation

### 3. Event Generation

**What happens when conditions match:**

- Creates graph events on configured output subjects
- Events can trigger downstream processors
- Integration with graph processor for entity updates

### 4. Cooldown & Metadata

**Prevents spam and adds context:**

```json
{
  "cooldown": "5m",
  "metadata": {
    "severity": "warning",
    "category": "battery",
    "action": "notify_operator"
  }
}
```

**Cooldown:** Per-entity cooldown prevents re-triggering for specified duration

---

## Example: Complete Flow with Rules

### Configuration

```json
{
  "components": {
    "udp": {
      "type": "input",
      "name": "udp",
      "enabled": true,
      "config": {
        "port": 14550,
        "ports": {
          "outputs": [{"name": "udp_out", "subject": "raw.udp", "type": "nats"}]
        }
      }
    },
    "json_generic": {
      "type": "processor",
      "name": "json_generic",
      "enabled": true,
      "config": {
        "ports": {
          "inputs": [{"subject": "raw.udp", "type": "nats"}],
          "outputs": [{"subject": "parsed.json", "type": "nats", "interface": "core.json.v1"}]
        }
      }
    },
    "json_to_entity": {
      "type": "processor",
      "name": "json_to_entity",
      "enabled": true,
      "config": {
        "ports": {
          "inputs": [{"subject": "parsed.json", "type": "nats", "interface": "core.json.v1"}],
          "outputs": [{"subject": "events.graph.entity", "type": "nats", "interface": "graph.Entity.v1"}]
        },
        "entity_id_field": "entity_id",
        "entity_type_field": "entity_type"
      }
    },
    "graph": {
      "type": "processor",
      "name": "graph-processor",
      "enabled": true,
      "config": {
        "input_subject": "events.graph.entity"
      }
    },
    "rule-processor": {
      "type": "processor",
      "name": "rule-processor",
      "enabled": true,
      "config": {
        "ports": {
          "inputs": [{"name": "entity_states", "type": "kv-watch"}],
          "outputs": [{"name": "alerts", "subject": "events.alerts", "type": "nats"}]
        },
        "inline_rules": [
          {
            "id": "battery-alert",
            "type": "alert",
            "name": "Low Battery Alert",
            "enabled": true,
            "entity": {"pattern": "*.*.robotics.*.drone.*"},
            "conditions": [
              {"field": "robotics.battery.level", "operator": "lte", "value": 20.0}
            ],
            "logic": "and",
            "cooldown": "5m"
          },
          {
            "id": "high-altitude",
            "type": "warning",
            "name": "High Altitude Warning",
            "enabled": true,
            "entity": {"pattern": "*.*.robotics.*.drone.*"},
            "conditions": [
              {"field": "geo.location.altitude", "operator": "gt", "value": 120.0}
            ],
            "logic": "and",
            "cooldown": "1m"
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
3. **json_to_entity creates EntityPayload** (implements Graphable) → `events.graph.entity`
4. **Graph processor stores entity** in ENTITY_STATES KV bucket
5. **Rules engine watches** ENTITY_STATES for changes matching `*.*.robotics.*.drone.*`
6. **Rules evaluate conditions:**
   - `robotics.battery.level` ≤ 20? → Publish alert event
   - `geo.location.altitude` > 120m? → Publish warning event
7. **Events published** to `events.alerts` subject
8. **PathRAG/GraphRAG can query** the enriched graph

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
- [Vocabulary Registry](06-vocabulary-registry.md) - Predicate definitions used in conditions
- [Graph Operations](../graph/) - Working with enriched graphs

**Advanced Topics:**

- [Query Fundamentals](../advanced/01-query-fundamentals.md) - PathRAG vs GraphRAG query-time intelligence
- [Performance Tuning](../advanced/07-performance-tuning.md) - Optimizing rule evaluation
- [Production Patterns](../advanced/08-production-patterns.md) - Production deployment

---

**Key Takeaway:** Rules Engine provides ingestion-time intelligence for dynamic knowledge graphs. Use it to create alerts, detect patterns, and automatically enrich your graph as data flows through the system.
