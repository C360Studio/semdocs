# Schema Contract Versioning Strategy

## Overview

This document defines the versioning strategy for component configuration schemas, OpenAPI specifications, and TypeScript types in the SemStreams ecosystem. Proper versioning ensures backward compatibility and smooth upgrades.

## Versioning Principles

### 1. **Schemas as Contracts**

Component schemas are **contracts** between:

- Component implementation and configuration
- Backend API and frontend UI
- Different instances of semstreams/semmem

Once published, schemas should be treated as immutable contracts.

### 2. **Semantic Versioning**

We use semantic versioning (MAJOR.MINOR.PATCH) for schema versions:

- **MAJOR** (v2, v3): Breaking changes (field removal, type changes, required fields)
- **MINOR** (v1.1, v1.2): Backward-compatible additions (new optional fields)
- **PATCH** (v1.0.1): Documentation updates, clarifications (no schema changes)

### 3. **Version in Filename**

Schema files include version in the filename: `component-name.v1.json`

This allows multiple versions to coexist during migrations.

## Current Version: v1

All current schemas are version 1 (`v1`):

- `udp.v1.json`
- `graph-processor.v1.json`
- `httppost.v1.json`
- etc.

**Schema ID**: `"$id": "udp.v1.json"`

## Backward Compatible Changes (MINOR)

### Adding Optional Fields

**Allowed**: Adding new fields with defaults

**Example**:
```go
// v1.0
type Config struct {
    Port int `schema:"type:int,desc:Server port,default:8080"`
}

// v1.1 - Added optional timeout field
type Config struct {
    Port    int `schema:"type:int,desc:Server port,default:8080"`
    Timeout int `schema:"type:int,desc:Timeout in seconds,default:30"` // NEW
}
```

**Impact**: ✅ No breaking changes
- Old configs still valid
- New field has default value
- Existing deployments unaffected

### Adding Enum Values

**Allowed**: Adding new enum options

**Example**:
```go
// v1.0
LogLevel string `schema:"type:enum,desc:Log level,enum:info|warn|error"`

// v1.1 - Added debug level
LogLevel string `schema:"type:enum,desc:Log level,enum:debug|info|warn|error"` // NEW debug
```

**Impact**: ✅ No breaking changes
- Old configs use existing values
- New option available for new deployments

### Relaxing Constraints

**Allowed**: Making validation less strict

**Example**:
```go
// v1.0
Port int `schema:"type:int,desc:Port,min:8000,max:9000"`

// v1.1 - Wider port range
Port int `schema:"type:int,desc:Port,min:1,max:65535"`
```

**Impact**: ✅ No breaking changes
- Previously valid configs remain valid
- More options now available

### Improving Descriptions

**Allowed**: Clarifying field descriptions

**Example**:
```go
// v1.0
Timeout int `schema:"type:int,desc:Timeout"`

// v1.0.1 - Better description (patch version)
Timeout int `schema:"type:int,desc:Connection timeout in seconds"`
```

**Impact**: ✅ No schema changes
- Only documentation improvement
- Patch version increment

## Breaking Changes (MAJOR)

Breaking changes require a new major version (v2).

### Removing Fields

**Breaking**: Removing configuration fields

**Example**:
```go
// v1
type Config struct {
    Port    int    `schema:"type:int,desc:Port"`
    OldField string `schema:"type:string,desc:Deprecated field"`
}

// v2 - Field removed
type Config struct {
    Port int `schema:"type:int,desc:Port"`
    // OldField removed - BREAKING CHANGE
}
```

**Migration Path**:
1. Release v1.x with deprecation warning
2. Document migration in release notes
3. Release v2 with field removed
4. Support v1 for grace period

### Changing Field Types

**Breaking**: Changing field data type

**Example**:
```go
// v1
Port string `schema:"type:string,desc:Port number"`

// v2 - Type changed
Port int `schema:"type:int,desc:Port number"` // BREAKING CHANGE
```

**Migration Path**:
1. Add new field with new type in v1.x
2. Deprecate old field
3. Release v2 with only new field

### Making Fields Required

**Breaking**: Removing default values

**Example**:
```go
// v1
APIKey string `schema:"type:string,desc:API key,default:demo-key"`

// v2 - No default, field now required
APIKey string `schema:"type:string,desc:API key"` // BREAKING CHANGE
```

**Impact**: ❌ Old configs will fail validation

**Migration Path**:
1. Document in release notes
2. Provide migration tool/script
3. Grace period with warnings

### Tightening Constraints

**Breaking**: Making validation more strict

**Example**:
```go
// v1
Port int `schema:"type:int,desc:Port,min:1,max:65535"`

// v2 - Narrower range
Port int `schema:"type:int,desc:Port,min:8000,max:9000"` // BREAKING CHANGE
```

**Impact**: ❌ Previously valid configs may become invalid

## Version Migration Strategy

### Deprecation Process

1. **Mark as Deprecated** (v1.x release):
   ```go
   // DEPRECATED: Use NewField instead. Will be removed in v2.
   OldField string `schema:"type:string,desc:Deprecated - use NewField"`
   NewField string `schema:"type:string,desc:Replacement field"`
   ```

2. **Update Documentation**:
   - Add deprecation notice to README
   - Update migration guide
   - Include in release notes

3. **Grace Period**:
   - Minimum 3 months or 2 minor releases
   - Both fields supported during transition

4. **Remove in Major Version**:
   - Release v2 with field removed
   - Update meta-schema if needed

### Multi-Version Support

During major version transitions, support both versions:

**File Structure**:
```
schemas/
  udp.v1.json          # Current version
  udp.v2.json          # Next version (during transition)
```

**Component Code**:
```go
func (u *UDP) MigrateConfig(oldVersion int, config map[string]interface{}) (map[string]interface{}, error) {
    if oldVersion == 1 {
        // Migrate v1 -> v2
        if oldPort, ok := config["oldPort"].(string); ok {
            config["port"], _ = strconv.Atoi(oldPort)
            delete(config, "oldPort")
        }
    }
    return config, nil
}
```

## OpenAPI Versioning

### Specification Version

OpenAPI spec version matches schema version:

**File**: `specs/openapi.v3.yaml`

```yaml
openapi: 3.0.3
info:
  title: SemStreams Component API
  version: 1.0.0  # API version
```

### Schema References

OpenAPI references specific schema versions:

```yaml
components:
  schemas:
    ComponentType:
      properties:
        schema:
          oneOf:
            - $ref: ../schemas/udp.v1.json
            - $ref: ../schemas/graph-processor.v1.json
```

### API Version in URLs

For breaking API changes, version in URL path:

```
/v1/components/types      # Current API
/v2/components/types      # Future breaking changes
```

## TypeScript Type Versioning

### Generated Types

TypeScript types are generated from the OpenAPI spec:

**File**: `src/lib/types/api.generated.ts`

```typescript
// Generated from openapi.v3.yaml version 1.0.0
export interface components {
  schemas: {
    ComponentType: { /* ... */ }
  }
}
```

### Version Compatibility

Frontend must match backend OpenAPI version:

```typescript
// Check API version compatibility
const API_VERSION = '1.0.0';

async function checkVersion() {
  const response = await fetch('/openapi.yaml');
  const spec = await response.text();
  const version = spec.match(/version: (.+)/)?.[1];

  if (version !== API_VERSION) {
    console.warn(`API version mismatch: expected ${API_VERSION}, got ${version}`);
  }
}
```

## Meta-Schema Versioning

The meta-schema defines the structure all component schemas must follow.

**File**: `specs/component-schema-meta.json`

**Current Version**: JSON Schema Draft-07

**Versioning**:
- Meta-schema changes are RARE
- Require coordinated update across all repositories
- Must maintain backward compatibility

**If meta-schema must change**:
1. Propose change with RFC
2. Validate against all existing schemas
3. Update all repositories simultaneously
4. Run full test suite

## Release Process

### Minor Release (Backward Compatible)

1. **Add new fields** with schema tags
2. **Run schema generation**:
   ```bash
   task schema:generate
   ```

3. **Run tests**:
   ```bash
   go test ./test/contract -v
   ```

4. **Commit schemas with code**:
   ```bash
   git add schemas/ specs/
   git commit -m "feat: add timeout configuration (v1.1)"
   ```

5. **Tag release**:
   ```bash
   git tag v1.1.0
   git push --tags
   ```

### Major Release (Breaking Changes)

1. **Create v2 branch**:
   ```bash
   git checkout -b v2-migration
   ```

2. **Update component code**:
   ```go
   // Remove deprecated fields
   // Update schema tags
   ```

3. **Generate v2 schemas**:
   ```bash
   task schema:generate
   # This creates new v2.json files
   ```

4. **Update meta-schema if needed**:
   ```bash
   # Update specs/component-schema-meta.json
   ```

5. **Write migration guide**:
   ```markdown
   # Migration: v1 -> v2
   - Field X removed, use Y instead
   - Field Z type changed from string to int
   ```

6. **Test thoroughly**:
   ```bash
   go test ./...
   go test ./test/contract -v
   ```

7. **Merge and tag**:
   ```bash
   git tag v2.0.0
   git push --tags
   ```

## Version Compatibility Matrix

### semstreams

| Schema Version | Component Version | OpenAPI Version | Status |
|---|---|---|---|
| v1.x | 1.x.x | 1.0.0 | ✅ Current |
| v2.x | 2.x.x | 2.0.0 | 🚧 Future |

### Frontend (semstreams-ui)

| Types Version | OpenAPI Version | Backend Version | Status |
|---|---|---|---|
| 1.x | 1.0.0 | v1.x | ✅ Current |
| 2.x | 2.0.0 | v2.x | 🚧 Future |

## Deprecation Timeline Example

**Example**: Removing `legacyPort` field

**v1.5.0** (Month 1):
```go
type Config struct {
    Port       int    `schema:"type:int,desc:Server port,default:8080"`
    LegacyPort string `schema:"type:string,desc:DEPRECATED - Use Port instead"`
}
```
- Release notes warn about deprecation
- Both fields work

**v1.6.0** (Month 2):
- Same as v1.5.0
- Log warning when LegacyPort used

**v1.7.0** (Month 3):
- Same as v1.6.0
- Final warning in release notes

**v2.0.0** (Month 4):
```go
type Config struct {
    Port int `schema:"type:int,desc:Server port,default:8080"`
    // LegacyPort removed
}
```
- Field removed
- Migration guide provided

## Best Practices

### 1. **Default Everything Optional**

Make new fields optional with defaults:
```go
// ✅ Good - won't break existing configs
NewField int `schema:"type:int,desc:New feature,default:100"`

// ❌ Avoid - breaks existing configs
NewField int `schema:"type:int,desc:Required new field"`
```

### 2. **Never Remove Defaults**

Removing defaults makes fields required:
```go
// v1 - Optional
Timeout int `schema:"type:int,desc:Timeout,default:30"`

// v2 - Now required (BREAKING)
Timeout int `schema:"type:int,desc:Timeout"` // ❌ Don't do this
```

### 3. **Deprecate Before Removing**

Always have a deprecation period:
```go
// v1.x - Both fields work
Old string `schema:"type:string,desc:DEPRECATED - use New"`
New string `schema:"type:string,desc:Replacement"`

// v2 - Old removed
New string `schema:"type:string,desc:Field description"`
```

### 4. **Document Changes**

Always document in:
- Code comments
- Migration guide
- Release notes
- CHANGELOG.md

### 5. **Test Migrations**

Write migration tests:
```go
func TestConfigMigration(t *testing.T) {
    v1Config := map[string]interface{}{
        "oldPort": "8080",
    }

    v2Config, err := MigrateConfig(1, v1Config)
    require.NoError(t, err)

    assert.Equal(t, 8080, v2Config["port"])
    assert.NotContains(t, v2Config, "oldPort")
}
```

## Monitoring Schema Changes

### Git Hooks

Use pre-commit hooks to catch breaking changes:

```bash
#!/bin/bash
# .git/hooks/pre-commit

# Check if schemas changed
if git diff --cached --name-only | grep -q "schemas/"; then
    echo "Schema files changed - ensure version is correct!"

    # Run contract tests
    go test ./test/contract || exit 1
fi
```

### CI Checks

CI automatically validates schemas:
- Schema structure validation
- Contract tests
- Version compatibility checks

## Related Documentation

- [Schema Tags Guide](./SCHEMA_TAGS_GUIDE.md) - Writing schema tags
- [Migration Guide](./MIGRATION_GUIDE.md) - Migrating configurations
- [Contract Testing](./CONTRACT_TESTING.md) - Testing schema contracts
- [CI Integration](./CI_SCHEMA_INTEGRATION.md) - Automated validation

## Summary

**Key Principles**:
- ✅ Add optional fields freely (minor version)
- ✅ Deprecate before removing (grace period)
- ❌ Never change types without major version
- ❌ Never remove fields without deprecation
- ✅ Always test migrations
- ✅ Document all changes

**Version Bumps**:
- **Patch** (v1.0.1): Documentation only
- **Minor** (v1.1.0): New optional fields
- **Major** (v2.0.0): Breaking changes

Following this strategy ensures smooth upgrades and maintains backward compatibility across the SemStreams ecosystem.
