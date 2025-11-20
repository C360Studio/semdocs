# SemDocs Audit & Consolidation Plan

**Date:** 2025-11-20
**Status:** Documentation restructure in progress - needs consolidation
**Last Updated:** 2025-11-20 (Expanded audit per user feedback)

---

## Executive Summary

After the numbered structure refactor, we have **duplicate content** across old and new directories, **scattered documentation** for key features (message system, deployment), and **excellent examples/** that don't fully align with documentation references.

**Goal:** Create one tight, coherent documentation structure with no duplication, complete coverage, and examples that match docs.

### 🆕 Expanded Audit Findings (2025-11-20)

Based on user questions about OpenAPI, deployment/integration numbering, component dev guides, and schema tags:

**Critical Gaps Identified:**
1. ❌ **No "How to Write a Component" guide** - Developers lack guidance on component interface, lifecycle, registration, ConfigSchema()
2. ✅ **OpenAPI clarified** - Two distinct use cases both implemented (Service HTTP handlers + Component config discovery via schema tags). User-facing docs missing in semdocs, but implementation docs exist in semstreams/docs/
3. ✅ **Schema tags doc is EXCELLENT** - But buried in guides/ with broken references to 4 files (2 exist in semstreams/docs/, 2 don't exist anywhere)
4. ✅ **deployment/ and integration/ are high quality** - Recommendation: Keep as unnumbered reference docs (not learning progressions)

**Plan Updated:**
- Phase 1 expanded from 4 hours to 7 hours (+3 new tasks)
- Total timeline: 16 hours (was 13 hours)

---

## Current State Audit

### ✅ New Numbered Structure (KEEP - Good Foundation)

| Directory | Files | Status | Coverage |
|-----------|-------|--------|----------|
| **docs/basics/** | 01-04 | ✅ Complete | Core concepts, components, routing, first flow |
| **docs/advanced/** | 01-05 | ✅ Complete | PathRAG/GraphRAG, perf tuning, architecture, production, algorithms |
| **docs/graph/** | 01-04 | ✅ Complete | Entities, relationships, queries, indexing |
| **docs/edge/** | 01-04 | ✅ Complete | Patterns, constraints, federation, offline |
| **docs/semantic/** | 01-04 | ✅ Complete | Overview, semembed, semsummarize, decision guide |

**Quality:** High - numbered, logical progression, recent updates

---

### ⚠️ Old/Duplicate Structure (NEEDS CONSOLIDATION)

#### docs/getting-started/ (3 files)

| File | Content | Status | Action |
|------|---------|--------|--------|
| `architecture.md` | High-level architecture | 🔄 Duplicates basics/01-what-is-semstreams.md | **MERGE** into basics/01 or **DELETE** |
| `concepts.md` | Core concepts | 🔄 Duplicates basics/01 | **MERGE** into basics/01 or **DELETE** |
| `quickstart.md` | Git clone instructions | ❌ Conflicts with examples/quickstart/ | **REPLACE** with pointer to examples/ |

**Recommendation:** **DELETE** this directory entirely, integrate any unique content into basics/01 and point to examples/

---

#### docs/guides/ (10 files) - Scattered Content

| File | Content | Status | Action |
|------|---------|--------|--------|
| `message-system.md` | **Message pkg docs** ✅ | 🔥 **CRITICAL - Missing from main docs** | **MOVE** to basics/05-message-system.md |
| `rules-engine.md` | Rules engine guide | 📝 Partially in specs/ | **MOVE** to basics/06-rules-engine.md |
| `schema-tags.md` | Schema system | 🔄 Overlaps with message-system | **MERGE** into basics/05 or **DELETE** |
| `vocabulary-system.md` | Vocabulary tags | 🔄 Overlaps with message-system | **MERGE** into basics/05 or **DELETE** |
| `pathrag.md` | PathRAG guide | 🔄 Duplicates advanced/01 | **DELETE** (covered in advanced/01) |
| `graphrag.md` | GraphRAG guide | 🔄 Duplicates advanced/01 | **DELETE** (covered in advanced/01) |
| `hybrid-queries.md` | Hybrid query patterns | ✅ Useful supplement | **MOVE** to advanced/06-hybrid-queries.md |
| `embedding-strategies.md` | When to use embeddings | 🔄 Duplicates semantic/04 | **DELETE** (covered in semantic/04) |
| `choosing-query-strategy.md` | Query strategy decision | ✅ Useful supplement | **MOVE** to advanced/07-query-strategies.md |
| `federation.md` | **PKI/mTLS federation** ✅ | 🔥 **CRITICAL - Advanced topic** | **MOVE** to deployment/federation-advanced.md |

**Recommendation:** Consolidate into numbered docs, delete duplicates

---

#### docs/reference/ (3 files)

| File | Content | Status | Action |
|------|---------|--------|--------|
| `algorithms.md` | Algorithm reference | 🔄 **DUPLICATES** advanced/05 | **DELETE** (we just updated advanced/05) |
| `port-allocation.md` | Port assignments | ✅ Useful reference | **KEEP** as reference/port-allocation.md |
| `schema-versioning.md` | Schema version strategy | ✅ Useful reference | **KEEP** as reference/schema-versioning.md |

**Recommendation:** Keep port-allocation and schema-versioning, delete algorithms.md

---

#### docs/deployment/ (7 files) - KEEP, Well Organized

| File | Content | Status | Quality |
|------|---------|--------|---------|
| `production.md` | Production deployment | ✅ Core | High |
| `tls-setup.md` | **TLS/mTLS config** | ✅ Core | High |
| `acme-setup.md` | **ACME/Let's Encrypt** | ✅ Core | High |
| `configuration.md` | Config reference | ✅ Core | High |
| `monitoring.md` | Monitoring setup | ✅ Core | High |
| `operations.md` | Ops procedures | ✅ Core | High |
| `optional-services.md` | SemEmbed/SemSummarize | ✅ Core | High |

**Recommendation:** **KEEP** - This is excellent deployment documentation

---

#### docs/integration/ (3 files) - KEEP, Well Organized

| File | Content | Status | Quality |
|------|---------|--------|---------|
| `rest-api.md` | REST API reference | ✅ Core | High |
| `graphql-api.md` | GraphQL API | ✅ Core | High |
| `nats-events.md` | NATS event streams | ✅ Core | High |

**Recommendation:** **KEEP** - API integration docs are well structured

---

#### docs/development/ (3 files) - KEEP

| File | Content | Status | Quality |
|------|---------|--------|---------|
| `contributing.md` | Contribution guide | ✅ Core | High |
| `testing.md` | Testing guide | ✅ Core | High |
| `agents.md` | AI agent workflows | ✅ Core | High |

**Recommendation:** **KEEP** - Important for contributors

---

#### docs/api/ - EMPTY

| Status | Action |
|--------|--------|
| ❌ Empty directory | **DELETE** (API docs are in integration/) |

---

#### docs/specs/ (1 file)

| File | Content | Status | Action |
|------|---------|--------|--------|
| `SPEC-001-generic-rules-engine.md` | Rules engine spec | ✅ Design doc | **KEEP** as specs/ |

**Recommendation:** Keep specs/ for design documents

---

### ✅ Examples Directory (EXCELLENT - Just Needs Alignment)

| Example | Content | Docker Compose | Config | Quality |
|---------|---------|----------------|--------|---------|
| `quickstart/` | Minimal NATS + SemStreams | ✅ | ✅ config.minimal.json | High |
| `with-ui/` | + Visual flow builder | ✅ | ✅ config.minimal.json | High |
| `with-embeddings/` | + SemEmbed service | ✅ | ✅ config.minimal.json | High |
| `production/` | Hardened production | ✅ | ✅ config.example.json | High |

**Issues:**
1. ⚠️ Examples use `config.minimal.json` but docs reference `config.json`
2. ⚠️ Docs in `docs/getting-started/quickstart.md` conflict with `examples/quickstart/README.md`
3. ⚠️ No doc section clearly points to examples/

**Recommendation:**
- Add `docs/basics/05-examples.md` that showcases all examples
- Update all doc references to point to `examples/` directory
- Ensure terminology consistency (minimal config, docker compose)

---

## Missing Content Analysis

### 🔥 Critical Missing Content

1. **Message System** (`message` package)
   - **Currently:** Buried in `docs/guides/message-system.md`
   - **Should be:** `docs/basics/05-message-system.md` (core concept)
   - **Why critical:** Message structure is fundamental to understanding SemStreams

2. **Rules Engine**
   - **Currently:** Split between `docs/guides/rules-engine.md` and `docs/specs/SPEC-001`
   - **Should be:** `docs/basics/06-rules-engine.md` (core concept)
   - **Why critical:** Rules are a major feature, need clear docs

3. **Examples Overview**
   - **Currently:** Not prominently featured in docs
   - **Should be:** `docs/basics/07-examples.md` pointing to examples/
   - **Why critical:** Fastest way to get started

4. **Component Development Guide** 🆕
   - **Currently:** Does NOT exist
   - **Should be:** `docs/development/writing-components.md`
   - **Why critical:** Developers need to know how to write custom components
   - **Gaps:**
     - Component interface implementation
     - Lifecycle methods (Start/Stop/Process)
     - Component registration process
     - ConfigSchema() implementation with schema tags
     - Port configuration (inputs/outputs)
     - Testing custom components
   - **Partial coverage:**
     - `guides/message-system.md` covers payload creation
     - `contributing.md` has general workflow
     - `guides/schema-tags.md` covers config schema

5. **Schema Tags DevX Documentation** 🆕
   - **Currently:** `docs/guides/schema-tags.md` (814 lines, EXCELLENT)
   - **Should be:** `docs/development/schema-tags.md` or `docs/reference/schema-tags.md`
   - **Why critical:** Schema-as-code is core to component development
   - **Issues:**
     - References 4 non-existent files:
       - `SCHEMA_GENERATION.md` - How schema extraction works
       - `CONTRACT_TESTING.md` - Schema validation tests
       - `MIGRATION_GUIDE.md` - Migrating from manual schemas
       - `OPENAPI_INTEGRATION.md` - OpenAPI spec generation
     - Needs to be in numbered structure or as authoritative reference

### ⚠️ Important Missing Content

6. **OpenAPI Documentation** 🆕 ✅ **Clarified - Two Distinct Use Cases**

   **Use Case 1: Service Manager HTTP Handler Documentation**
   - **Purpose:** Documents HTTP endpoints exposed by services
   - **Implementation:** Services implement `HTTPHandler.OpenAPISpec()` interface
   - **Examples:** ComponentManager APIs, FlowService runtime endpoints
   - **Location:** `semstreams/service/openapi.go`
   - **Aggregation:** Service Manager collects specs from all services

   **Use Case 2: Component Configuration Schema Discovery**
   - **Purpose:** Documents component configuration APIs for UI/clients
   - **Implementation:** `cmd/schema-exporter` generates from schema tags
   - **Generated File:** `semstreams/specs/openapi.v3.yaml` ✅ EXISTS
   - **Endpoints:** `/components/types`, `/components/types/{id}`, `/components/flowgraph`
   - **Comprehensive Docs:** `semstreams/docs/SCHEMA_GENERATION.md` (907 lines!)

   **What's Missing in semdocs:**
   - ❌ No user guide explaining how to consume OpenAPI specs
   - ❌ `rest-api.md` claims `/api/openapi.json` endpoint (unclear if served)
   - ❌ No distinction between service HTTP APIs vs component config APIs

   **Action needed:**
   - Create `docs/integration/openapi-usage.md` explaining both use cases
   - Clarify in `rest-api.md` which OpenAPI spec it references
   - Link to component discovery API documentation

   **Note:** Implementation docs exist in `semstreams/docs/` (developer-facing), but user-facing docs are missing in `semdocs/`

7. **Hybrid Queries**
   - **Currently:** `docs/guides/hybrid-queries.md`
   - **Should be:** `docs/advanced/06-hybrid-queries.md`

8. **Query Strategy Decision Guide**
   - **Currently:** `docs/guides/choosing-query-strategy.md`
   - **Should be:** `docs/advanced/07-query-strategies.md`

9. **Advanced Federation (PKI/mTLS)**
   - **Currently:** `docs/guides/federation.md`
   - **Should be:** `docs/deployment/federation-advanced.md`

### ℹ️ Structural Questions 🆕

10. **Should deployment/ and integration/ be numbered?**
   - **Current state:** Unnumbered reference documentation
   - **Quality:** High quality, well-organized (7+3 files)
   - **Analysis:**
     - ✅ **Keep unnumbered** - These are reference docs, not learning progressions
     - Unlike basics/advanced/graph/semantic/edge (sequential learning), deployment/ and integration/ are consulted as-needed
     - Numbering makes sense for tutorials/learning paths, not for reference material
   - **Recommendation:** Keep deployment/ and integration/ as unnumbered reference sections

---

## Consolidation Plan

### Phase 1: Critical Fixes (Priority: 🔥 Immediate)

**Goal:** Add missing critical content to numbered structure

1. **Create docs/basics/05-message-system.md**
   - Source: `docs/guides/message-system.md`
   - Add to numbered sequence
   - Cover message structure, types, payloads, validation

2. **Create docs/basics/06-rules-engine.md**
   - Source: `docs/guides/rules-engine.md` + `docs/specs/SPEC-001`
   - Practical guide (not just spec)
   - Examples, configuration, use cases

3. **Create docs/basics/07-examples.md**
   - Point to `examples/` directory
   - Overview of all examples
   - When to use each example
   - Link docker-compose files

4. **Create docs/development/writing-components.md** 🆕
   - Complete component development guide
   - Component interface, lifecycle, registration
   - ConfigSchema() with schema tags
   - Port configuration (inputs/outputs)
   - Testing components
   - Pull from: contributing.md + message-system.md + schema-tags.md

5. **Move docs/guides/schema-tags.md → docs/development/schema-tags.md** 🆕
   - Keep as authoritative reference for schema-as-code
   - Update broken references to non-existent files
   - Add note about what's documented vs. what's planned

6. **Create OpenAPI user documentation** 🆕
   - Create `docs/integration/openapi-usage.md` explaining both use cases:
     - Service Manager HTTP handler documentation
     - Component configuration schema discovery (from schema tags)
   - Clarify in `docs/integration/rest-api.md` which OpenAPI spec(s) are available
   - Document how to access `/specs/openapi.v3.yaml` for component discovery

### Phase 2: Remove Duplicates (Priority: ⚠️ High)

**Goal:** Delete redundant content

1. **Delete docs/getting-started/** (3 files)
   - Merge any unique content into basics/01
   - Update all links pointing to getting-started/

2. **Delete duplicate guides** (5 files) 🔄
   - `guides/pathrag.md` (covered in advanced/01)
   - `guides/graphrag.md` (covered in advanced/01)
   - `guides/embedding-strategies.md` (covered in semantic/04)
   - `guides/vocabulary-system.md` (merge into basics/05)
   - `reference/algorithms.md` (covered in advanced/05)
   - NOTE: `guides/schema-tags.md` is MOVED to development/ in Phase 1, not deleted

3. **Delete empty docs/api/** directory

### Phase 3: Promote Valuable Content (Priority: ⚠️ High)

**Goal:** Move valuable guides into numbered structure

1. **Create docs/advanced/06-hybrid-queries.md**
   - Source: `docs/guides/hybrid-queries.md`
   - Clean up, ensure consistency

2. **Create docs/advanced/07-query-strategies.md**
   - Source: `docs/guides/choosing-query-strategy.md`
   - Decision trees, use case matrix

3. **Create docs/deployment/federation-advanced.md**
   - Source: `docs/guides/federation.md`
   - PKI/mTLS/ACME integration
   - Production federation patterns

### Phase 4: Align Examples with Docs (Priority: ⚠️ High)

**Goal:** Ensure examples/ matches documentation

1. **Update all doc references**
   - Change `config.json` → `config.minimal.json` (or vice versa)
   - Point to `examples/` not `docs/getting-started/quickstart.md`
   - Use consistent terminology

2. **Verify example configs**
   - Ensure all configs in examples/ are valid
   - Match configuration patterns shown in docs
   - Test all docker-compose files

3. **Add cross-references**
   - Link from basics/04-first-flow.md to examples/quickstart/
   - Link from deployment/ to examples/production/
   - Link from semantic/ to examples/with-embeddings/

### Phase 5: Final Cleanup (Priority: ✅ Medium)

**Goal:** Polish and verify

1. **Update README.md**
   - Clear documentation structure
   - Link to examples/
   - Quick start using examples/

2. **Update navigation**
   - Ensure all internal links work
   - No broken references
   - Clear progression through docs

3. **Add missing metadata**
   - Ensure all numbered docs have consistent frontmatter
   - Version tags where appropriate
   - Last updated dates

---

## Proposed Final Structure

```text
semdocs/
├── README.md                          # Updated with clear structure
├── CONTRIBUTING.md
├── examples/                          # ✅ KEEP - Excellent examples
│   ├── README.md                      # ✅ High quality overview
│   ├── quickstart/                    # ✅ Minimal setup
│   ├── with-ui/                       # ✅ + Visual builder
│   ├── with-embeddings/               # ✅ + Semantic search
│   └── production/                    # ✅ Hardened deployment
│
└── docs/
    ├── basics/                        # ✅ NUMBERED - Core concepts
    │   ├── 01-what-is-semstreams.md  # ✅ Complete
    │   ├── 02-components.md           # ✅ Complete
    │   ├── 03-routing.md              # ✅ Complete + NATS core/JetStream
    │   ├── 04-first-flow.md           # ✅ Complete
    │   ├── 05-message-system.md       # 🆕 ADD from guides/
    │   ├── 06-rules-engine.md         # 🆕 ADD from guides/ + specs/
    │   └── 07-examples.md             # 🆕 ADD - point to examples/
    │
    ├── graph/                         # ✅ NUMBERED - Graph operations
    │   ├── 01-entities.md             # ✅ Complete
    │   ├── 02-relationships.md        # ✅ Complete
    │   ├── 03-queries.md              # ✅ Complete
    │   └── 04-indexing.md             # ✅ Complete
    │
    ├── semantic/                      # ✅ NUMBERED - Semantic features
    │   ├── 01-overview.md             # ✅ Complete
    │   ├── 02-semembed.md             # ✅ Complete
    │   ├── 03-semsummarize.md         # ✅ Complete
    │   └── 04-decision-guide.md       # ✅ Complete
    │
    ├── edge/                          # ✅ NUMBERED - Edge deployment
    │   ├── 01-patterns.md             # ✅ Complete
    │   ├── 02-constraints.md          # ✅ Complete
    │   ├── 03-federation.md           # ✅ Complete + WebSocket
    │   └── 04-offline.md              # ✅ Complete
    │
    ├── advanced/                      # ✅ NUMBERED - Advanced topics
    │   ├── 01-pathrag-graphrag-decisions.md  # ✅ Complete
    │   ├── 02-performance-tuning.md   # ✅ Complete
    │   ├── 03-architecture-deep-dive.md # ✅ Complete
    │   ├── 04-production-patterns.md  # ✅ Complete
    │   ├── 05-algorithm-reference.md  # ✅ Complete + Actual algos
    │   ├── 06-hybrid-queries.md       # 🆕 MOVE from guides/
    │   └── 07-query-strategies.md     # 🆕 MOVE from guides/
    │
    ├── deployment/                    # ✅ KEEP - Deployment guides
    │   ├── configuration.md           # ✅ Complete
    │   ├── production.md              # ✅ Complete
    │   ├── tls-setup.md               # ✅ Complete
    │   ├── acme-setup.md              # ✅ Complete
    │   ├── federation-advanced.md     # 🆕 MOVE from guides/
    │   ├── monitoring.md              # ✅ Complete
    │   ├── operations.md              # ✅ Complete
    │   └── optional-services.md       # ✅ Complete
    │
    ├── integration/                   # ✅ KEEP - API reference
    │   ├── rest-api.md                # ✅ Complete
    │   ├── graphql-api.md             # ✅ Complete
    │   └── nats-events.md             # ✅ Complete
    │
    ├── development/                   # ✅ KEEP - Contributor docs
    │   ├── contributing.md            # ✅ Complete
    │   ├── testing.md                 # ✅ Complete
    │   ├── agents.md                  # ✅ Complete
    │   ├── writing-components.md      # 🆕 ADD - Complete component dev guide
    │   └── schema-tags.md             # 🆕 MOVE from guides/ - Schema-as-code reference
    │
    ├── reference/                     # ✅ KEEP - Technical reference
    │   ├── port-allocation.md         # ✅ Keep
    │   └── schema-versioning.md       # ✅ Keep
    │
    └── specs/                         # ✅ KEEP - Design specs
        └── SPEC-001-generic-rules-engine.md  # ✅ Keep

❌ DELETE:
├── docs/getting-started/          # DELETE (duplicate)
├── docs/guides/                   # DELETE (consolidate into numbered)
└── docs/api/                      # DELETE (empty)
```

---

## Execution Checklist

### Phase 1: Add Critical Content ✅
- [ ] Create `docs/basics/05-message-system.md` from `guides/message-system.md`
- [ ] Create `docs/basics/06-rules-engine.md` from `guides/rules-engine.md` + `specs/SPEC-001`
- [ ] Create `docs/basics/07-examples.md` pointing to `examples/`
- [ ] Update `docs/basics/04-first-flow.md` to reference examples/quickstart/
- [ ] 🆕 Create `docs/development/writing-components.md` (component dev guide)
- [ ] 🆕 Move `guides/schema-tags.md` → `docs/development/schema-tags.md`
- [ ] 🆕 Fix broken references in schema-tags.md to non-existent files
- [ ] 🆕 Create `docs/integration/openapi-usage.md` (explain both OpenAPI use cases)
- [ ] 🆕 Clarify in `rest-api.md` which OpenAPI specs are available

### Phase 2: Promote Valuable Content ✅
- [ ] Create `docs/advanced/06-hybrid-queries.md` from `guides/hybrid-queries.md`
- [ ] Create `docs/advanced/07-query-strategies.md` from `guides/choosing-query-strategy.md`
- [ ] Create `docs/deployment/federation-advanced.md` from `guides/federation.md`

### Phase 3: Remove Duplicates ✅
- [ ] Delete `docs/getting-started/` (after merging unique content to basics/01)
- [ ] Delete duplicate files in `docs/guides/`:
  - [ ] pathrag.md
  - [ ] graphrag.md
  - [ ] embedding-strategies.md
  - [ ] vocabulary-system.md (merge into basics/05 first)
  - [ ] 🔄 NOT schema-tags.md (moved to development/ in Phase 1)
- [ ] Delete `docs/reference/algorithms.md`
- [ ] Delete `docs/api/` directory

### Phase 4: Align Examples ✅
- [ ] Verify all example configs are valid and match docs
- [ ] Update all doc references to point to examples/
- [ ] Ensure terminology consistency (minimal config vs full config)
- [ ] Test all docker-compose files

### Phase 5: Final Polish ✅
- [ ] Update main README.md with new structure
- [ ] Update all internal cross-references
- [ ] Verify no broken links
- [ ] Add missing frontmatter/metadata
- [ ] Run final docs build/lint

---

## Success Criteria

1. ✅ **No duplicate content** - Each concept documented once in logical place
2. ✅ **Complete coverage** - Message system, rules engine, examples all documented
3. ✅ **Examples aligned** - Examples/ directory matches documentation references
4. ✅ **Clear structure** - Numbered progression through concepts
5. ✅ **Easy navigation** - Clear paths from basic → advanced topics
6. ✅ **Working links** - All internal references valid
7. ✅ **Tested examples** - All docker-compose files verified working

---

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Breaking links | Medium | Comprehensive link audit, redirects |
| Losing content | High | Git-based, can recover anything |
| Example mismatch | Medium | Test all examples before consolidation |
| Incomplete migration | Medium | Phased approach, verify each phase |

---

## Timeline Estimate

- **Phase 1** (Critical): 7 hours 🔄 (+3 hours for component guide, schema tags move, OpenAPI verification)
- **Phase 2** (Promote): 2 hours
- **Phase 3** (Delete): 2 hours
- **Phase 4** (Align): 3 hours
- **Phase 5** (Polish): 2 hours

**Total:** ~16 hours of focused work 🔄 (was 13 hours)

**New additions in Phase 1:**
- +2 hours: Create comprehensive component development guide
- +0.5 hours: Move and update schema-tags.md
- +0.5 hours: Verify OpenAPI implementation and document/fix

---

## Notes

- Git tracking means we can always recover deleted content
- Examples/ directory is excellent - use as template for doc quality
- Numbered structure is working well - extend it consistently
- Keep deployment/, integration/, development/, reference/ as-is (high quality)
- Focus consolidation on basics/ and advanced/ where duplication exists

---

## 🆕 Answers to Specific User Questions

### 1. "Are we missing OpenAPI mentions?"

**Answer:** ✅ **Clarified - Two distinct OpenAPI use cases both implemented**

**OpenAPI IS used in two ways:**

**Use Case 1: Service Manager HTTP Handler Documentation**
- Services implement `HTTPHandler` interface with `OpenAPISpec()` method
- Service Manager aggregates specs from all services
- Documented in `semstreams/service/openapi.go`, `semstreams/service/README.md`
- Examples: ComponentManager APIs, FlowService runtime endpoints

**Use Case 2: Component Configuration Schema Discovery**
- Schema exporter generates OpenAPI from schema tags
- Output: `semstreams/specs/openapi.v3.yaml` ✅ **EXISTS**
- Endpoints: `/components/types`, `/components/types/{id}`, `/components/flowgraph`
- Comprehensive developer docs: `semstreams/docs/SCHEMA_GENERATION.md` (907 lines!)

**What's missing in semdocs:**
- ❌ No **user-facing** guide on how to consume OpenAPI specs
- ❌ `rest-api.md` claims `/api/openapi.json` endpoint but doesn't clarify which use case
- ❌ No distinction between service HTTP APIs vs component config APIs

**Action needed:**
1. Create `docs/integration/openapi-usage.md` explaining both use cases
2. Clarify in `rest-api.md` which OpenAPI spec(s) are served
3. Document component discovery API for UI/client developers

**Note:** Implementation docs exist in `semstreams/docs/` (for Go developers writing services), but user-facing integration docs are missing in `semdocs/`

---

### 2. "Deployment and Integration docs are the old system and not numbered"

**Answer:** ✅ **Correct - but they should stay unnumbered**

**Analysis:**
- `docs/deployment/` (7 files) - High quality, production-focused
- `docs/integration/` (3 files) - High quality, API reference

**Recommendation: Keep unnumbered**

**Why?**
- These are **reference documentation**, not learning progressions
- Unlike basics/advanced/graph/semantic/edge (which teach concepts sequentially), deployment/ and integration/ are consulted as-needed
- Numbering makes sense for tutorials, not for reference material
- Current organization is logical and easy to navigate

**Comparison:**
- ✅ **basics/01-04**: Learning progression (numbered makes sense)
- ✅ **deployment/**: Reference docs (numbering would be arbitrary)
- ✅ **integration/**: API reference (numbering would be arbitrary)

---

### 3. "What about dev guides like how to write a component etc?"

**Answer:** ❌ **Critical gap - doesn't exist**

**What exists:**
- `docs/development/contributing.md` - General workflow, code standards
- `docs/guides/message-system.md` - How to create message payloads (partial)
- `docs/guides/schema-tags.md` - ConfigSchema() implementation (partial)

**What's missing:**
A complete "How to Write a Component" guide covering:
- ❌ Component interface implementation
- ❌ Lifecycle methods (Start/Stop/Process)
- ❌ Component registration process
- ❌ ConfigSchema() implementation with schema tags
- ❌ Port configuration (inputs/outputs on NATS subjects)
- ❌ Testing custom components
- ❌ Complete end-to-end example

**Action needed:**
Create `docs/development/writing-components.md` that synthesizes:
- Component interface from semstreams codebase
- Message creation from guides/message-system.md
- ConfigSchema from guides/schema-tags.md
- Ports/NATS from docs/basics/03-routing.md
- Testing from docs/development/testing.md

---

### 4. "There's a bunch going on with schema tags and code gen around some of our DevX"

**Answer:** ✅ **Excellent doc exists, but needs better placement**

**What exists:**
- `docs/guides/schema-tags.md` - **814 lines, EXCELLENT quality**
  - Explains schema-as-code philosophy
  - Complete tag reference (type, desc, default, min, max, enum, category)
  - Why schema-as-code vs separate JSON Schema files
  - Special types (ports, cache)
  - Best practices, troubleshooting
  - Complete examples

**Issues:**
1. ⚠️ **Buried in guides/** (old structure, hard to discover)
2. ❌ **References 4 non-existent files:**
   - `SCHEMA_GENERATION.md` - How schema extraction works internally
   - `CONTRACT_TESTING.md` - How to validate schemas in tests
   - `MIGRATION_GUIDE.md` - Migrating from manual schemas to tags
   - `OPENAPI_INTEGRATION.md` - How schemas become OpenAPI specs
3. ⚠️ **Not linked from main development docs**

**Action needed:**
1. Move `guides/schema-tags.md` → `docs/development/schema-tags.md`
2. Update broken references with notes: "TODO: Document" or remove references
3. Link from `docs/development/writing-components.md` (new guide)
4. Consider if referenced files should be created or are out of scope

---

**Status:** Ready for execution - awaiting approval to proceed with Phase 1
