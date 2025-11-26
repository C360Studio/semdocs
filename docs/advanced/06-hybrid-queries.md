# Hybrid Query Patterns

Combining PathRAG and GraphRAG for maximum context and relevance

---

## Overview

Hybrid queries leverage both graph structure (PathRAG) and semantic clustering (GraphRAG) to provide comprehensive, relevant results. This guide shows practical patterns for combining these strategies.

## Why Hybrid Queries?

**PathRAG alone**: Finds structurally connected entities, might miss semantically relevant ones
**GraphRAG alone**: Finds semantically similar entities, might miss direct relationships
**Hybrid**: Gets the best of both - connected AND similar entities

### Real-World Example

**Query**: "Show me everything related to this authentication bug"

**PathRAG only** finds:

- Services that depend on the auth module
- Config files linked to auth
- Tests that explicitly reference the bug

**GraphRAG only** finds:

- Other auth-related bugs (similar content)
- Documentation about authentication
- PRs with similar keywords

**Hybrid** finds all of the above!

---

## Pattern 1: Semantic First, Then Expand

**Strategy**: Start with semantic search to find relevant entities, then use PathRAG to explore their connections.

### Use Case: Code Review Context

**Scenario**: You're reviewing a PR and want to understand both similar code and dependent code.

**Implementation**:

```javascript
async function getReviewContext(prID) {
  // Step 1: Find semantically similar entities in the PR's community
  const similar = await graphQLQuery(`
    query {
      localSearch(
        entityID: "${prID}"
        query: "api changes breaking changes contracts"
        level: 1
      ) {
        ... on Spec { id title status }
        ... on Issue { id number title }
        ... on PullRequest { id number title }
      }
    }
  `);

  // Step 2: For each similar entity, explore connected entities
  const expansions = await Promise.all(
    similar.slice(0, 3).map(entity =>
      pathQuery({
        start_entity: entity.id,
        max_depth: 2,
        edge_filter: ["depends_on", "modifies", "related_to"],
        max_nodes: 50,
        max_time: "200ms"
      })
    )
  );

  // Step 3: Merge results with scoring
  return mergeResults(similar, expansions, {
    semanticWeight: 0.6,  // Semantic similarity is primary
    pathWeight: 0.4,      // Connection strength is secondary
    deduplication: true
  });
}
```

**Result**:

- **Similar specs** that define related APIs (semantic)
- **Dependent services** that use those specs (structural)
- **Related issues** in the same community (semantic)
- **Modified tests** from the dependency chain (structural)

### Use Case: Incident Investigation

**Scenario**: A drone reported a battery failure. Find all related data.

**Implementation**:

```javascript
async function investigateIncident(alertID) {
  // Step 1: Find semantically similar alerts/issues
  const similarIncidents = await graphQLQuery(`
    query {
      localSearch(
        entityID: "${alertID}"
        query: "battery failure performance degradation"
        level: 1
      ) {
        ... on Alert { id timestamp severity message }
        ... on Telemetry { id entity_id metrics }
      }
    }
  `);

  // Step 2: Trace the impact chain from original alert
  const impactChain = await pathQuery({
    start_entity: alertID,
    max_depth: 4,
    edge_filter: ["triggered_by", "affects", "causes"],
    decay_factor: 0.85,
    max_time: "500ms"
  });

  // Step 3: Combine timeline
  return {
    similar_past_incidents: similarIncidents.map(inc => ({
      ...inc,
      relevance_score: inc.score,
      source: "semantic_search"
    })),
    current_impact_chain: impactChain.entities.map(ent => ({
      ...ent,
      distance: impactChain.paths.find(p => p.includes(ent.id))?.length,
      relevance_score: impactChain.scores[ent.id],
      source: "path_traversal"
    })),
    combined_score: calculateCombinedScore(similarIncidents, impactChain)
  };
}
```

**Result**:

- **Historical patterns**: Similar battery failures (semantic)
- **Current impact**: Affected drones and systems (structural)
- **Root causes**: Previous incidents with similar signatures (semantic)
- **Cascade effects**: Downstream failures (structural)

---

## Pattern 2: Structural First, Then Enrich

**Strategy**: Start with PathRAG to find connected entities, then use GraphRAG to understand their semantic context.

### Use Case: Dependency Analysis with Context

**Scenario**: A database config is changing. Find what depends on it AND understand the semantic context of those services.

**Implementation**:

```javascript
async function analyzeConfigChange(configID) {
  // Step 1: Trace dependency chain
  const dependents = await pathQuery({
    start_entity: configID,
    max_depth: 5,
    edge_filter: ["depends_on"],
    max_nodes: 200,
    max_time: "1s"
  });

  // Step 2: For top affected services, get their semantic community
  const topServices = dependents.entities
    .filter(e => e.entity_type === "service")
    .slice(0, 5);

  const communities = await Promise.all(
    topServices.map(service =>
      graphQLQuery(`
        query {
          entityCommunity(
            entityID: "${service.entity_id}"
            level: 1
          ) {
            id
            summary
            keywords
            members
          }
        }
      `)
    )
  );

  // Step 3: For each community, search for configuration-related entities
  const configContext = await Promise.all(
    communities.map(community =>
      graphQLQuery(`
        query {
          localSearch(
            entityID: "${topServices[0].entity_id}"
            query: "configuration database connection pool"
            level: 1
          ) {
            ... on Spec { id title }
            ... on Doc { id title }
          }
        }
      `)
    )
  );

  return {
    direct_dependents: dependents.entities,
    impact_score: dependents.scores,
    semantic_context: communities,
    related_documentation: configContext.flat()
  };
}
```

**Result**:

- **Direct dependents**: Services that use this config (structural)
- **Community context**: What semantic cluster each service belongs to (semantic)
- **Related docs**: Documentation in those communities about configuration (semantic)
- **Impact assessment**: Combined structural + semantic understanding

### Use Case: Spatial Network Analysis

**Scenario**: A communication relay went down. Find nearby drones AND similar outage patterns.

**Implementation**:

```javascript
async function analyzeNetworkOutage(relayID) {
  // Step 1: Find spatially connected entities
  const spatialNetwork = await pathQuery({
    start_entity: relayID,
    max_depth: 3,
    edge_filter: ["near", "communicates", "relays_through"],
    decay_factor: 0.8,
    max_nodes: 100,
    max_time: "300ms"
  });

  // Step 2: Find global patterns of similar outages
  const historicalPatterns = await graphQLQuery(`
    query {
      globalSearch(
        query: "relay outage communication failure mesh network"
        level: 2
        maxCommunities: 5
      ) {
        ... on Alert { id timestamp entity_id message }
        ... on Incident { id title resolution }
      }
    }
  `);

  // Step 3: Cross-reference: which affected drones had similar issues?
  const affectedDrones = spatialNetwork.entities
    .filter(e => e.entity_type === "drone");

  const droneHistory = await Promise.all(
    affectedDrones.slice(0, 10).map(drone =>
      graphQLQuery(`
        query {
          localSearch(
            entityID: "${drone.entity_id}"
            query: "connectivity issues relay failure"
            level: 0
          ) {
            ... on Alert { id timestamp }
          }
        }
      `)
    )
  );

  return {
    affected_area: spatialNetwork,
    historical_patterns: historicalPatterns,
    vulnerable_drones: droneHistory
      .map((hist, idx) => ({
        drone: affectedDrones[idx],
        past_issues: hist,
        risk_score: calculateRiskScore(hist, spatialNetwork.scores[affectedDrones[idx].entity_id])
      }))
      .sort((a, b) => b.risk_score - a.risk_score)
  };
}
```

**Result**:

- **Affected area**: Drones in communication range (structural)
- **Historical patterns**: Similar outages from the past (semantic)
- **Risk assessment**: Which drones are most vulnerable based on both proximity and history

---

## Pattern 3: Parallel Execution, Fused Results

**Strategy**: Run PathRAG and GraphRAG queries in parallel, then fuse results with weighted scoring.

### Use Case: Comprehensive Entity Context

**Scenario**: Show all context for an entity - both structural connections and semantic similarities.

**Implementation**:

```javascript
async function getFullContext(entityID, userQuery) {
  // Run queries in parallel
  const [pathResults, localResults, globalResults] = await Promise.all([
    // PathRAG: Structural connections
    pathQuery({
      start_entity: entityID,
      max_depth: 3,
      max_nodes: 100,
      max_time: "200ms",
      decay_factor: 0.85
    }),

    // GraphRAG Local: Similar entities in community
    graphQLQuery(`
      query {
        localSearch(
          entityID: "${entityID}"
          query: "${userQuery}"
          level: 1
        ) {
          id
          ... on Spec { title status }
          ... on Issue { number title state }
        }
      }
    `),

    // GraphRAG Global: Related entities across system
    graphQLQuery(`
      query {
        globalSearch(
          query: "${userQuery}"
          level: 2
          maxCommunities: 3
        ) {
          id
          ... on Doc { title }
          ... on PullRequest { number title }
        }
      }
    `)
  ]);

  // Rank fusion: Combine scores from all three strategies
  return rankFusion({
    structural: pathResults.entities.map(e => ({
      ...e,
      source: "path",
      score: pathResults.scores[e.entity_id]
    })),
    local_semantic: localResults.map(e => ({
      ...e,
      source: "local_search",
      score: e.score
    })),
    global_semantic: globalResults.map(e => ({
      ...e,
      source: "global_search",
      score: e.score
    })),
    weights: {
      structural: 0.4,
      local_semantic: 0.4,
      global_semantic: 0.2
    }
  });
}
```

**Result**: Comprehensive context from all three query strategies, ranked by combined relevance.

### Rank Fusion Algorithm

```javascript
function rankFusion(options) {
  const { structural, local_semantic, global_semantic, weights } = options;

  // Normalize scores to [0, 1]
  const normalize = (items) => {
    const maxScore = Math.max(...items.map(i => i.score));
    return items.map(i => ({ ...i, normalized_score: i.score / maxScore }));
  };

  const normStructural = normalize(structural);
  const normLocal = normalize(local_semantic);
  const normGlobal = normalize(global_semantic);

  // Collect all unique entities
  const entityMap = new Map();

  const addOrUpdate = (entity, score, source) => {
    const id = entity.id || entity.entity_id;
    if (!entityMap.has(id)) {
      entityMap.set(id, {
        ...entity,
        scores: { structural: 0, local: 0, global: 0 },
        sources: []
      });
    }

    const existing = entityMap.get(id);
    if (source === "path") existing.scores.structural = score;
    if (source === "local_search") existing.scores.local = score;
    if (source === "global_search") existing.scores.global = score;
    existing.sources.push(source);
  };

  normStructural.forEach(e => addOrUpdate(e, e.normalized_score, "path"));
  normLocal.forEach(e => addOrUpdate(e, e.normalized_score, "local_search"));
  normGlobal.forEach(e => addOrUpdate(e, e.normalized_score, "global_search"));

  // Calculate combined scores
  const results = Array.from(entityMap.values()).map(entity => ({
    ...entity,
    combined_score:
      (entity.scores.structural * weights.structural) +
      (entity.scores.local * weights.local_semantic) +
      (entity.scores.global * weights.global_semantic),
    diversity: entity.sources.length  // Entities found by multiple methods rank higher
  }));

  // Sort by combined score, with diversity as tiebreaker
  return results.sort((a, b) => {
    if (Math.abs(a.combined_score - b.combined_score) < 0.01) {
      return b.diversity - a.diversity;
    }
    return b.combined_score - a.combined_score;
  });
}
```

---

## Pattern 4: Iterative Refinement

**Strategy**: Start broad with GraphRAG Global, then refine with PathRAG exploration.

### Use Case: Exploratory Search

**Scenario**: "I want to understand how authentication works in this system" (no starting point).

**Implementation**:

```javascript
async function exploreAuthenticationArchitecture() {
  // Step 1: Global search to find authentication communities
  const authCommunities = await graphQLQuery(`
    query {
      globalSearch(
        query: "authentication authorization security jwt oauth sessions"
        level: 2
        maxCommunities: 5
      ) {
        id
        ... on Spec { title }
        ... on Service { name }
        ... on Doc { title }
      }
    }
  `);

  // Step 2: Pick the most relevant result as starting point
  const primaryAuthEntity = authCommunities[0];

  // Step 3: Explore connections from that entity
  const authArchitecture = await pathQuery({
    start_entity: primaryAuthEntity.id,
    max_depth: 3,
    edge_filter: ["implements", "uses", "depends_on", "calls"],
    max_nodes: 150,
    max_time: "500ms"
  });

  // Step 4: For each connected service, get its community context
  const services = authArchitecture.entities
    .filter(e => e.entity_type === "service");

  const serviceContexts = await Promise.all(
    services.map(svc =>
      graphQLQuery(`
        query {
          entityCommunity(entityID: "${svc.entity_id}", level: 1) {
            summary
            keywords
          }
        }
      `)
    )
  );

  return {
    entry_points: authCommunities,
    architecture_graph: authArchitecture,
    service_contexts: services.map((svc, idx) => ({
      service: svc,
      community: serviceContexts[idx],
      connections: authArchitecture.paths
        .filter(path => path.includes(svc.entity_id))
    }))
  };
}
```

**Result**:

- **Discovery**: Found auth-related entities without knowing where to start (global search)
- **Structure**: Mapped out how they connect (path traversal)
- **Context**: Understood what each component does (community summaries)

---

## Performance Optimization

### Caching Strategy

```javascript
class HybridQueryCache {
  constructor() {
    this.pathCache = new LRU({ max: 500, ttl: 5 * 60 * 1000 }); // 5 min TTL
    this.communityCache = new LRU({ max: 100, ttl: 30 * 60 * 1000 }); // 30 min TTL
  }

  async cachedPathQuery(params) {
    const key = JSON.stringify(params);
    if (this.pathCache.has(key)) {
      return this.pathCache.get(key);
    }

    const result = await pathQuery(params);
    this.pathCache.set(key, result);
    return result;
  }

  async cachedCommunity(entityID, level) {
    const key = `${entityID}:${level}`;
    if (this.communityCache.has(key)) {
      return this.communityCache.get(key);
    }

    const result = await getCommunity(entityID, level);
    this.communityCache.set(key, result);
    return result;
  }
}
```

### Parallel Execution Limits

```javascript
async function parallelHybridQuery(entityIDs, query) {
  // Limit concurrency to avoid overwhelming the system
  const concurrencyLimit = 5;
  const results = [];

  for (let i = 0; i < entityIDs.length; i += concurrencyLimit) {
    const batch = entityIDs.slice(i, i + concurrencyLimit);
    const batchResults = await Promise.all(
      batch.map(async (id) => {
        const [path, local] = await Promise.all([
          pathQuery({ start_entity: id, max_depth: 2, max_time: "100ms" }),
          localSearch({ entityID: id, query, level: 1 })
        ]);
        return { id, path, local };
      })
    );
    results.push(...batchResults);
  }

  return results;
}
```

### Progressive Results

```javascript
async function* streamingHybridQuery(entityID, query) {
  // Yield fast results first
  const pathPromise = pathQuery({
    start_entity: entityID,
    max_depth: 2,
    max_time: "100ms"
  });

  const localPromise = graphQLQuery(`
    query {
      localSearch(entityID: "${entityID}", query: "${query}", level: 1)
    }
  `);

  // Yield path results immediately when ready
  const pathResults = await pathPromise;
  yield { type: "structural", data: pathResults, partial: true };

  // Yield local results when ready
  const localResults = await localPromise;
  yield { type: "semantic_local", data: localResults, partial: true };

  // Optionally, fetch global results (slower)
  const globalResults = await graphQLQuery(`
    query {
      globalSearch(query: "${query}", level: 2, maxCommunities: 3)
    }
  `);
  yield { type: "semantic_global", data: globalResults, partial: false };
}
```

---

## Best Practices

### 1. Choose Appropriate Weights

```javascript
// For code review (relationships matter most)
const codeReviewWeights = {
  structural: 0.7,      // Dependencies are critical
  local_semantic: 0.2,  // Similar code is helpful
  global_semantic: 0.1  // Broader context is nice-to-have
};

// For incident investigation (patterns matter most)
const incidentWeights = {
  structural: 0.3,      // Current impact chain
  local_semantic: 0.5,  // Similar past incidents
  global_semantic: 0.2  // System-wide patterns
};

// For exploratory search (balance all)
const exploratoryWeights = {
  structural: 0.33,
  local_semantic: 0.33,
  global_semantic: 0.34
};
```

### 2. Set Appropriate Timeouts

```javascript
const hybridQueryConfig = {
  // PathRAG should be fast
  path: {
    max_time: "200ms",
    max_nodes: 100,
    max_depth: 3
  },

  // Local search is medium speed
  local: {
    timeout: "300ms"
  },

  // Global search is slowest
  global: {
    timeout: "500ms",
    max_communities: 5
  },

  // Overall hybrid query timeout
  total_timeout: "1s"
};
```

### 3. Deduplicate Results

```javascript
function deduplicateResults(results) {
  const seen = new Map();

  return results.filter(item => {
    const id = item.id || item.entity_id;
    if (seen.has(id)) {
      // Merge scores if entity found by multiple strategies
      const existing = seen.get(id);
      existing.combined_score = Math.max(existing.combined_score, item.combined_score);
      existing.sources.push(...item.sources);
      return false;
    }
    seen.set(id, item);
    return true;
  });
}
```

### 4. Fail Gracefully

```javascript
async function robustHybridQuery(entityID, query) {
  const results = {
    structural: null,
    semantic: null,
    error: null
  };

  try {
    // Try PathRAG
    results.structural = await pathQuery({
      start_entity: entityID,
      max_time: "200ms"
    });
  } catch (error) {
    console.warn("PathRAG failed, continuing with GraphRAG only", error);
    results.error = { pathrag: error.message };
  }

  try {
    // Try GraphRAG
    results.semantic = await localSearch({
      entityID,
      query,
      level: 1
    });
  } catch (error) {
    console.warn("GraphRAG failed, returning PathRAG results only", error);
    results.error = { ...results.error, graphrag: error.message };
  }

  // Return whatever succeeded
  if (!results.structural && !results.semantic) {
    throw new Error("All query strategies failed");
  }

  return results;
}
```

---

## Summary

**Hybrid queries provide the most comprehensive results** by combining:

- **Structural relationships** (PathRAG) - What's connected
- **Semantic similarity** (GraphRAG Local) - What's similar in the cluster
- **Global patterns** (GraphRAG Global) - What's relevant system-wide

**Key Patterns**:

1. **Semantic → Path**: Find similar, then explore connections
2. **Path → Semantic**: Find connected, then understand context
3. **Parallel Fusion**: Run all, combine scores
4. **Iterative Refinement**: Global discovery → local exploration

**Remember**: Start simple, add complexity as needed. A single PathRAG or GraphRAG query is often sufficient - use hybrid queries when you need maximum insight.

---

## Next Steps

- **[Performance Tuning](07-performance-tuning.md)** - Optimize hybrid query performance
- **[Production Patterns](08-production-patterns.md)** - Deploy hybrid query workloads

**Reference:**
- **[Query Fundamentals](01-query-fundamentals.md)** - Understand PathRAG and GraphRAG concepts
- **[Practical Query Patterns](05-query-strategies.md)** - Single-strategy patterns
- **[Configuration Guide](04-configuration-guide.md)** - Enable features based on your data

---

**Pro Tip**: The most powerful queries aren't always the most complex. Use the simplest strategy that meets your needs, and add hybrid techniques when you need deeper insight.
