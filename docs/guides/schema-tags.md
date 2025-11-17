# Schema Tags Developer Guide

## Overview

Schema tags are Go struct field tags that define how component configuration fields are exposed in JSON Schemas and the OpenAPI specification. This guide shows you how to write effective schema tags for your components.

## Basic Syntax

Schema tags use the `schema` key in Go struct tags:

```go
type Config struct {
    FieldName string `schema:"type:string,desc:Field description"`
}
```

**Tag Format**: `schema:"key:value,key:value,..."`

## Available Tag Keys

### Required Tags

#### `type` - Field Type

Specifies the JSON Schema type for the field.

**Supported Types**:
- `string` - Text values
- `int` - Integer numbers
- `float` - Floating point numbers
- `bool` - Boolean values
- `array` - Arrays/slices
- `object` - Nested objects/structs
- `enum` - Enumerated string values
- `ports` - Special type for NATS port configuration
- `cache` - Special type for cache configuration

**Example**:
```go
type Config struct {
    Name     string  `schema:"type:string,desc:Component name"`
    Count    int     `schema:"type:int,desc:Number of items"`
    Enabled  bool    `schema:"type:bool,desc:Enable feature"`
    Tags     []string `schema:"type:array,desc:Tag list"`
}
```

#### `desc` - Description

Human-readable description of the field's purpose.

**Best Practices**:
- Be concise but clear
- Explain the field's purpose, not just its name
- Include units if applicable (e.g., "Timeout in seconds")
- Mention defaults if important

**Examples**:
```go
// ❌ Bad - just repeats field name
Timeout int `schema:"type:int,desc:Timeout"`

// ✅ Good - explains purpose and units
Timeout int `schema:"type:int,desc:Connection timeout in seconds"`

// ✅ Good - provides context
MaxRetries int `schema:"type:int,desc:Maximum retry attempts before failing"`
```

### Optional Tags

#### `default` - Default Value

Specifies the default value if the field is omitted.

**Examples**:
```go
type Config struct {
    Host    string `schema:"type:string,desc:Server hostname,default:localhost"`
    Port    int    `schema:"type:int,desc:Server port,default:8080"`
    Enabled bool   `schema:"type:bool,desc:Enable feature,default:true"`
}
```

**Generated JSON Schema**:
```json
{
    "properties": {
        "host": {
            "type": "string",
            "description": "Server hostname",
            "default": "localhost"
        },
        "port": {
            "type": "number",
            "description": "Server port",
            "default": 8080
        }
    }
}
```

#### `min` / `max` - Numeric Constraints

Specify minimum and maximum values for numeric fields.

**Examples**:
```go
type Config struct {
    Port       int `schema:"type:int,desc:Server port,min:1,max:65535"`
    Threads    int `schema:"type:int,desc:Worker threads,min:1,max:100"`
    Percentage int `schema:"type:int,desc:Success rate,min:0,max:100"`
}
```

#### `enum` - Enumerated Values

Restricts field to specific allowed values.

**Example**:
```go
type Config struct {
    LogLevel string `schema:"type:enum,desc:Logging level,enum:debug|info|warn|error"`
    Format   string `schema:"type:enum,desc:Output format,enum:json|yaml|xml"`
}
```

**Generated JSON Schema**:
```json
{
    "properties": {
        "logLevel": {
            "type": "string",
            "description": "Logging level",
            "enum": ["debug", "info", "warn", "error"]
        }
    }
}
```

#### `category` - UI Organization

Groups fields for better UI organization.

**Values**:
- `basic` - Essential configuration (shown by default)
- `advanced` - Advanced settings (collapsed by default)

**Example**:
```go
type Config struct {
    // Basic settings
    Host string `schema:"type:string,desc:Server host,category:basic"`
    Port int    `schema:"type:int,desc:Server port,category:basic"`

    // Advanced settings
    Timeout      int  `schema:"type:int,desc:Timeout in ms,category:advanced"`
    MaxRetries   int  `schema:"type:int,desc:Retry attempts,category:advanced"`
    EnableDebug  bool `schema:"type:bool,desc:Debug logging,category:advanced"`
}
```

## Special Types

### Ports Configuration

Use `type:ports` for NATS port configuration fields.

**Example**:
```go
type ProcessorConfig struct {
    Ports component.Ports `schema:"type:ports,desc:Input and output ports"`
}
```

**What This Generates**:
- Validates port structure (Input/Output maps)
- Includes port metadata in schema
- Provides proper JSON Schema for port configuration

### Cache Configuration

Use `type:cache` for cache/KV bucket configuration.

**Example**:
```go
type ProcessorConfig struct {
    Cache component.Cache `schema:"type:cache,desc:State storage configuration"`
}
```

## Complete Examples

### Simple Input Component

```go
package udp

type Config struct {
    Port    int    `schema:"type:int,desc:UDP port to listen on,min:1,max:65535,default:14550"`
    Address string `schema:"type:string,desc:Bind address,default:0.0.0.0"`
    Buffer  int    `schema:"type:int,desc:Buffer size in bytes,min:1024,default:8192,category:advanced"`
}

func (u *UDP) ConfigSchema() component.ConfigSchema {
    return component.ExtractSchema(Config{})
}
```

**Generated Schema**:
```json
{
    "$schema": "http://json-schema.org/draft-07/schema#",
    "$id": "udp.v1.json",
    "type": "object",
    "title": "udp Configuration",
    "description": "UDP input component for receiving MAVLink and other UDP data",
    "properties": {
        "port": {
            "type": "number",
            "description": "UDP port to listen on",
            "minimum": 1,
            "maximum": 65535,
            "default": 14550
        },
        "address": {
            "type": "string",
            "description": "Bind address",
            "default": "0.0.0.0"
        },
        "buffer": {
            "type": "number",
            "description": "Buffer size in bytes",
            "minimum": 1024,
            "default": 8192,
            "category": "advanced"
        }
    },
    "required": []
}
```

### Processor with Ports and Cache

```go
package graph

type ProcessorConfig struct {
    Ports component.Ports `schema:"type:ports,desc:Input and output ports for message flow"`
    Cache component.Cache `schema:"type:cache,desc:Entity state storage"`

    BatchSize    int  `schema:"type:int,desc:Batch size for bulk operations,min:1,max:1000,default:100"`
    EnableIndex  bool `schema:"type:bool,desc:Enable semantic indexing,default:true"`
    IndexModel   string `schema:"type:string,desc:Embedding model name,default:all-MiniLM-L6-v2,category:advanced"`
}

func (g *GraphProcessor) ConfigSchema() component.ConfigSchema {
    return component.ExtractSchema(ProcessorConfig{})
}
```

### Output Component with Enum

```go
package httppost

type Config struct {
    URL         string `schema:"type:string,desc:HTTP endpoint URL"`
    Method      string `schema:"type:enum,desc:HTTP method,enum:POST|PUT|PATCH,default:POST"`
    Timeout     int    `schema:"type:int,desc:Request timeout in seconds,min:1,max:300,default:30"`
    Retries     int    `schema:"type:int,desc:Number of retry attempts,min:0,max:10,default:3"`
    ContentType string `schema:"type:string,desc:Content-Type header,default:application/json,category:advanced"`
}

func (h *HTTPPost) ConfigSchema() component.ConfigSchema {
    return component.ExtractSchema(Config{})
}
```

## Best Practices

### 1. **Always Provide Descriptions**

```go
// ❌ Bad
Port int `schema:"type:int"`

// ✅ Good
Port int `schema:"type:int,desc:Server port for incoming connections"`
```

### 2. **Use Sensible Defaults**

```go
// ✅ Good - users rarely need to change this
Timeout int `schema:"type:int,desc:Connection timeout in seconds,default:30"`

// ✅ Good - common use case
LogLevel string `schema:"type:enum,desc:Logging verbosity,enum:debug|info|warn|error,default:info"`
```

### 3. **Add Constraints for Validation**

```go
// ✅ Good - prevents invalid port numbers
Port int `schema:"type:int,desc:Server port,min:1,max:65535"`

// ✅ Good - reasonable thread count
Workers int `schema:"type:int,desc:Worker thread count,min:1,max:100,default:4"`
```

### 4. **Use Categories for Complex Configs**

```go
type Config struct {
    // Basic - always visible
    URL  string `schema:"type:string,desc:API endpoint,category:basic"`
    Port int    `schema:"type:int,desc:Listen port,category:basic"`

    // Advanced - collapsed by default
    Timeout        int    `schema:"type:int,desc:Timeout in ms,category:advanced,default:5000"`
    MaxConnections int    `schema:"type:int,desc:Connection pool size,category:advanced,default:10"`
    EnableDebug    bool   `schema:"type:bool,desc:Verbose logging,category:advanced,default:false"`
}
```

### 5. **Document Required vs Optional**

Schema generation automatically detects required fields based on whether they have defaults:

```go
type Config struct {
    // Required - no default
    APIKey string `schema:"type:string,desc:Authentication API key"`

    // Optional - has default
    Region string `schema:"type:string,desc:AWS region,default:us-east-1"`
}
```

## Common Patterns

### Connection Configuration

```go
type ConnectionConfig struct {
    Host           string `schema:"type:string,desc:Server hostname,default:localhost"`
    Port           int    `schema:"type:int,desc:Server port,min:1,max:65535,default:8080"`
    Timeout        int    `schema:"type:int,desc:Connection timeout in seconds,min:1,default:30,category:advanced"`
    MaxRetries     int    `schema:"type:int,desc:Maximum retry attempts,min:0,max:10,default:3,category:advanced"`
    RetryDelay     int    `schema:"type:int,desc:Delay between retries in ms,min:100,default:1000,category:advanced"`
}
```

### Feature Flags

```go
type FeatureConfig struct {
    EnableMetrics    bool `schema:"type:bool,desc:Collect performance metrics,default:true"`
    EnableTracing    bool `schema:"type:bool,desc:Enable distributed tracing,default:false,category:advanced"`
    EnableValidation bool `schema:"type:bool,desc:Validate input messages,default:true"`
}
```

### Resource Limits

```go
type ResourceConfig struct {
    MaxMemory     int `schema:"type:int,desc:Maximum memory usage in MB,min:100,max:10000,default:1024"`
    MaxCPU        int `schema:"type:int,desc:CPU limit as percentage,min:1,max:100,default:80"`
    WorkerThreads int `schema:"type:int,desc:Worker thread pool size,min:1,max:100,default:4"`
    QueueSize     int `schema:"type:int,desc:Message queue capacity,min:100,max:100000,default:1000"`
}
```

## Validation

After defining your schema tags, validate them:

### 1. **Generate Schemas**

```bash
task schema:generate
```

### 2. **Run Schema Tests**

```bash
go test ./cmd/schema-exporter -v
```

### 3. **Run Contract Tests**

```bash
go test ./test/contract -v
```

### 4. **Check Generated Schema**

```bash
cat schemas/your-component.v1.json
```

Verify:
- All fields have proper types
- Descriptions are clear
- Constraints are correct
- Defaults make sense

## Troubleshooting

### Schema Not Generated

**Problem**: Component schema not appearing in `schemas/` directory

**Solution**:
1. Ensure `ConfigSchema()` method is implemented
2. Check component is registered in `componentregistry/register.go`
3. Run `task schema:generate`

### Invalid Schema Structure

**Problem**: Schema validation fails

**Solution**:
Check for:
- Missing required tags (`type`, `desc`)
- Invalid enum syntax (use `|` separator)
- Invalid constraints (min > max)
- Typos in tag keys

### Type Mapping Issues

**Problem**: Go type doesn't map correctly to JSON Schema

**Solution**:
Use explicit `type:` tag:
```go
// Explicit type for clarity
Count int64 `schema:"type:int,desc:Item count"`

// Arrays need explicit type
Tags []string `schema:"type:array,desc:Tag list"`
```

## Migration from Manual Schemas

If you have existing manual JSON schemas, convert them to schema tags:

**Before** (manual JSON):
```json
{
    "properties": {
        "port": {
            "type": "number",
            "description": "Server port",
            "minimum": 1,
            "maximum": 65535,
            "default": 8080
        }
    }
}
```

**After** (schema tags):
```go
type Config struct {
    Port int `schema:"type:int,desc:Server port,min:1,max:65535,default:8080"`
}
```

See `docs/MIGRATION_GUIDE.md` for detailed migration instructions.

## Related Documentation

- [Schema Generation System](./SCHEMA_GENERATION.md) - Overall architecture
- [Contract Testing](./CONTRACT_TESTING.md) - Validating schemas
- [Migration Guide](./MIGRATION_GUIDE.md) - Migrating from manual schemas
- [OpenAPI Integration](./OPENAPI_INTEGRATION.md) - OpenAPI spec generation

## Summary

**Key Points**:
- Always use `type:` and `desc:` tags
- Add constraints (`min`, `max`, `enum`) for validation
- Provide sensible `default:` values
- Use `category:` for UI organization
- Test with `task schema:generate` and `go test ./test/contract`

**Tag Template**:
```go
FieldName Type `schema:"type:jsontype,desc:Clear description,default:value,min:X,max:Y,category:basic|advanced"`
```

Following these guidelines ensures your component schemas are clear, valid, and provide excellent developer experience for users of your components.
