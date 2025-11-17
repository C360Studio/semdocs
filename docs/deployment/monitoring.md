# Metrics Coverage Audit Report

**Date**: 2025-11-14
**Issue**: semstreams-7uti
**Status**: Initial Audit Complete

## Executive Summary

The SemStreams observability stack is **production-ready infrastructure** with Prometheus + Grafana configured, but **metrics instrumentation coverage is incomplete**. Core platform metrics exist, but several key components lack instrumentation.

### Infrastructure Status: ✅ READY

- Docker Compose with Prometheus (9090) + Grafana (3000)
- 4 pre-built Grafana dashboards
- Metrics service on port 9090 at `/metrics`
- Taskfile command: `task services:start:observability`

### Instrumentation Status: ⚠️ PARTIAL

- **Core Platform**: ✅ Fully instrumented
- **Graph Components**: ✅ Well instrumented
- **Cache/Buffer/Worker**: ✅ Well instrumented
- **Input/Output**: ✅ Partially instrumented
- **Processors**: ❌ Missing metrics
- **Storage**: ❌ Missing metrics
- **Flow Engine**: ❌ Missing metrics

---

## Detailed Findings

### ✅ Components WITH Metrics (9 components)

#### 1. **Core Platform Metrics** (`metric/core.go`)
**Status**: ✅ Complete
**Metrics**:
- `semstreams_service_status` - Service lifecycle (stopped/starting/running/stopping/failed)
- `semstreams_messages_received_total` - Messages by service and type
- `semstreams_messages_processed_total` - Processing status
- `semstreams_messages_published_total` - Published messages
- `semstreams_processing_duration_seconds` - Latency histograms
- `semstreams_errors_total` - Errors by service and type
- `semstreams_health_status` - Health check status
- `semstreams_nats_connected` - NATS connection status
- `semstreams_nats_rtt` - NATS round-trip time
- `semstreams_nats_reconnects_total` - Reconnection count
- `semstreams_nats_circuit_breaker` - Circuit breaker state

**Registration**: Via `MetricsRegistry` - automatically registered

---

#### 2. **IndexManager** (`processor/graph/indexmanager/metrics.go`)
**Status**: ✅ Complete
**Metrics**:
- Event processing (total, processed, failed, dropped)
- Processing latency histogram
- Deduplication (events, cache size)
- Index operations (updates by type, failures, latency)
- Query metrics (total, failed, latency)
- Buffer metrics (size, utilization, backlog)
- Health status and processing lag
- KV watch metrics (events, failures, reconnections, active watchers)

**Registration**: ✅ Registers via `MetricsRegistry`
**Dashboard**: `configs/grafana/dashboards/indexmanager-metrics.json` (13KB)

---

#### 3. **QueryManager** (`processor/graph/querymanager/metrics.go`)
**Status**: ✅ Complete
**Metrics**:
- Query execution (total, by type, duration)
- Cache performance (hits, misses, evictions)
- Result set sizes
- Error counts

**Registration**: ✅ Registers via `MetricsRegistry`
**Dashboard**: `configs/grafana/dashboards/graph-processor.json` (14KB)

---

#### 4. **Cache** (`pkg/cache/metrics.go`)
**Status**: ✅ Complete
**Metrics**:
- `semstreams_cache_hits_total` - Cache hits by component
- `semstreams_cache_misses_total` - Cache misses
- `semstreams_cache_sets_total` - Set operations
- `semstreams_cache_deletes_total` - Delete operations
- `semstreams_cache_evictions_total` - Evictions
- `semstreams_cache_size` - Current entry count

**Registration**: ✅ Registers via `MetricsRegistry`
**Dashboard**: `configs/grafana/dashboards/cache-performance.json` (11KB)

---

#### 5. **Buffer** (`pkg/buffer/metrics.go`)
**Status**: ✅ Complete
**Metrics**:
- Buffer operations (reads, writes, drops)
- Buffer utilization
- Wait times

**Registration**: ✅ Registers via `MetricsRegistry`

---

#### 6. **Worker Pool** (`pkg/worker/pool.go`)
**Status**: ✅ Complete
**Metrics**:
- Worker utilization
- Task queue depth
- Task processing time
- Idle/active worker counts

**Registration**: ✅ Registers via `MetricsRegistry`

---

#### 7. **UDP Input** (`input/udp/udp.go`)
**Status**: ✅ Complete
**Metrics**:
- Packets received
- Bytes received
- Parsing errors
- Processing latency

**Registration**: ✅ Registers via `MetricsRegistry`

---

#### 8. **WebSocket Input** (`input/websocket/websocket_input.go`)
**Status**: ✅ Complete
**Metrics**:
- Connection count
- Messages received
- Errors

**Registration**: ✅ Registers via `MetricsRegistry`

---

#### 9. **WebSocket Output** (`output/websocket/websocket.go`)
**Status**: ✅ Complete
**Metrics**:
- Connections
- Messages sent
- Errors
- Broadcasting metrics

**Registration**: ✅ Registers via `MetricsRegistry`

---

### ❌ Components WITHOUT Metrics (5+ areas)

#### 1. **JSON Filter Processor** (`processor/json_filter/`)
**Status**: ❌ No metrics
**Missing Metrics**:
- Messages filtered (matched vs rejected)
- Filter evaluation time
- Filter condition complexity
- Error rates

**Impact**: Cannot track filter performance or effectiveness
**Priority**: Medium

---

#### 2. **JSON Map Processor** (`processor/json_map/`)
**Status**: ❌ No metrics
**Missing Metrics**:
- Transformation operations count
- Mapping errors
- Field extraction success/failure
- Processing latency

**Impact**: Cannot track data transformation performance
**Priority**: Medium

---

#### 3. **ObjectStore** (`storage/objectstore/`)
**Status**: ❌ No metrics
**Missing Metrics**:
- Read/write operations
- Read/write latency
- Object counts by bucket
- Storage bytes used
- Error rates (NATS JetStream errors)

**Impact**: Cannot monitor storage layer health or performance
**Priority**: **High** - Storage is critical infrastructure

---

#### 4. **Flow Engine/Service Framework**
**Status**: ❌ No flow-level metrics
**Missing Metrics**:
- Flow execution count
- Flow execution time
- Flow validation errors
- Component initialization time
- Message routing efficiency

**Impact**: Cannot track end-to-end flow performance
**Priority**: **High** - Core functionality

---

#### 5. **Entity Graph Operations** (`graph/`)
**Status**: ⚠️ Partial (have IndexManager/QueryManager, missing mutation metrics)
**Missing Metrics**:
- Entity counts by type
- Relationship counts by type
- Graph mutation operations (create/update/delete)
- Federation correlation metrics
- Graph traversal depth/complexity

**Impact**: Limited visibility into graph state and operations
**Priority**: Medium

---

#### 6. **NATS JetStream Specifics**
**Status**: ❌ No JetStream metrics (only core NATS connection)
**Missing Metrics**:
- Stream lag/pending messages
- Consumer delivery rates
- Acknowledgment rates
- Stream storage usage
- Consumer backlogs

**Impact**: Cannot monitor message queue health
**Priority**: **High** - Message delivery is critical

---

## Prometheus Configuration Analysis

### Current Config (`configs/prometheus/prometheus.yml`)

```yaml
scrape_configs:
  - job_name: 'semstreams'
    static_configs:
      - targets: ['host.docker.internal:8080']  # ⚠️ WRONG PORT
    metrics_path: '/metrics'

  - job_name: 'semstreams-cache'
    static_configs:
      - targets: ['host.docker.internal:8080']  # ⚠️ WRONG PORT
    metrics_path: '/metrics/cache'  # ⚠️ WRONG PATH

  - job_name: 'semstreams-indexmanager'
    metrics_path: '/metrics/index'  # ⚠️ WRONG PATH

  - job_name: 'semstreams-graph'
    metrics_path: '/metrics/graph'  # ⚠️ WRONG PATH
```

### Issues Found

1. **❌ Wrong Port**: Config uses `8080`, but Metrics service defaults to `9090`
2. **❌ Wrong Paths**: Config expects separate paths (`/metrics/cache`), but all metrics exposed at `/metrics`
3. **❌ Redundant Jobs**: All metrics are on single endpoint, don't need separate jobs

### Recommended Fix

```yaml
scrape_configs:
  # Single job for all SemStreams metrics
  - job_name: 'semstreams'
    static_configs:
      - targets: ['host.docker.internal:9090']  # Correct port
    metrics_path: '/metrics'  # Single unified endpoint
    scrape_interval: 10s
    scrape_timeout: 5s

  # Prometheus self-monitoring
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

All metrics are exposed at a **single endpoint** (`/metrics`) with proper `component` labels to differentiate sources.

---

## Grafana Dashboard Analysis

### Existing Dashboards

1. **`semstreams-overview.json`** (9.4KB)
   - Status: ✅ Should work (uses core metrics)
   - Covers: Service status, message throughput, error rates

2. **`cache-performance.json`** (11KB)
   - Status: ✅ Should work (cache metrics exist)
   - Covers: Hit rates, evictions, cache size

3. **`indexmanager-metrics.json`** (13KB)
   - Status: ✅ Should work (IndexManager metrics exist)
   - Covers: Indexing operations, embedding metrics

4. **`graph-processor.json`** (14KB)
   - Status: ✅ Should work (QueryManager metrics exist)
   - Covers: Query performance, result sets

### Missing Dashboards

1. **Storage Dashboard** - ObjectStore metrics don't exist yet
2. **Flow Execution Dashboard** - Flow-level metrics don't exist yet
3. **NATS JetStream Dashboard** - JetStream metrics don't exist yet
4. **Processor Pipeline Dashboard** - Processor metrics incomplete

---

## Recommended Actions

### Immediate (Priority: High)

1. **Fix Prometheus Config**
   - Update port from `8080` to `9090`
   - Remove redundant scrape jobs
   - Use single `/metrics` endpoint

2. **Add ObjectStore Metrics**
   - Read/write operations and latency
   - Error rates
   - Storage usage

3. **Add NATS JetStream Metrics**
   - Stream lag and pending counts
   - Consumer delivery rates
   - Storage usage

4. **Add Flow Engine Metrics**
   - Flow execution counts and duration
   - Validation errors
   - Component initialization

### Short Term (Priority: Medium)

5. **Add Processor Metrics**
   - JSON Filter: match/reject counts, latency
   - JSON Map: transformation counts, errors

6. **Enhance Graph Metrics**
   - Entity/relationship counts by type
   - Mutation operation metrics

7. **Create Missing Dashboards**
   - Storage performance
   - Flow execution
   - NATS JetStream health

### Nice to Have (Priority: Low)

8. **Add Business Metrics**
   - Entity resolution rates
   - Federation correlation success
   - Semantic search quality metrics

9. **Add Alerting Rules**
   - High error rates
   - Service down
   - NATS disconnections
   - Storage failures

---

## Testing Checklist

### Infrastructure Test
- [ ] Start monitoring stack: `task services:start:observability`
- [ ] Verify Prometheus UI: http://localhost:9090
- [ ] Verify Grafana UI: http://localhost:3000 (admin/admin)
- [ ] Check Prometheus targets page for scrape status

### Metrics Test
- [ ] Start SemStreams with metrics service enabled
- [ ] Verify metrics endpoint responds: `curl http://localhost:9090/metrics`
- [ ] Verify core metrics present (search for `semstreams_service_status`)
- [ ] Verify component metrics present (search for `semstreams_cache_hits_total`)

### Dashboard Test
- [ ] Load overview dashboard in Grafana
- [ ] Generate some load (send messages)
- [ ] Verify panels populate with real data
- [ ] Check for any errors or missing queries

---

## Conclusion

**Infrastructure**: ✅ Production-ready
**Instrumentation**: ⚠️ ~60% coverage (9/15 major areas)
**Configuration**: ❌ Needs fixing (wrong port/paths)
**Dashboards**: ✅ Existing dashboards should work once config fixed

**Next Steps**:
1. Fix Prometheus configuration (10 min)
2. Test monitoring stack startup (5 min)
3. Add ObjectStore metrics (2-4 hours)
4. Add NATS JetStream metrics (2-4 hours)
5. Add Flow Engine metrics (2-4 hours)

**Total Effort Estimate**: ~1-2 days for complete coverage
