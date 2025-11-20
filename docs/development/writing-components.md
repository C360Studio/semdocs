# Writing Custom Components

Complete guide to implementing SemStreams components

---

## Overview

SemStreams components are self-contained processing units that consume messages from NATS subjects, process them, and optionally publish results. This guide shows you how to write, test, and deploy custom components.

**What You'll Learn:**
- Component interface and lifecycle
- Configuration with schema tags
- Port configuration (inputs/outputs)
- Message handling patterns
- Testing and registration

---

## Component Types

SemStreams supports four component types:

| Type | Purpose | Examples |
|------|---------|----------|
| **Input** | Ingest data from external sources | UDP, TCP, HTTP, MQTT, File |
| **Processor** | Transform, filter, enrich data | JSON parser, filter, graph processor |
| **Output** | Send data to external systems | HTTP POST, File writer, Database |
| **Storage** | Persist data | NATS KV, Object Store |

---

## Component Interface

All components implement the `Component` interface:

```go
type Component interface {
    // Lifecycle
    Initialize(ctx context.Context, config interface{}) error
    Start(ctx context.Context) error
    Stop(ctx context.Context) error

    // Processing
    Process(ctx context.Context, msg message.Message) error

    // Discovery
    ConfigSchema() ConfigSchema
    PortsSchema() PortsSchema

    // Health
    IsHealthy() bool
    GetStatus() Status
}
```

### Lifecycle Methods

**Initialize(ctx, config)**
- Called once during component creation
- Parse and validate configuration
- Set up internal state (no external connections)

**Start(ctx)**
- Called when component should begin processing
- Open connections, start goroutines, subscribe to NATS
- Must be idempotent (safe to call multiple times)

**Stop(ctx)**
- Called for graceful shutdown
- Close connections, stop goroutines, cleanup resources
- Must respect context timeout

**Process(ctx, msg)**
- Called for each message (for processors/outputs)
- Implement message processing logic
- Return error if processing fails

---

## Step-by-Step: Building a Component

Let's build a complete custom component: an HTTP webhook output that posts messages to an endpoint.

### Step 1: Create Package Structure

```text
output/httpwebhook/
├── httpwebhook.go      # Component implementation
├── config.go           # Configuration struct with schema tags
└── httpwebhook_test.go # Tests
```

### Step 2: Define Configuration

**output/httpwebhook/config.go:**

```go
package httpwebhook

import "time"

// Config defines HTTP webhook configuration
type Config struct {
    // Basic settings
    URL         string `schema:"type:string,desc:Webhook endpoint URL,category:basic"`
    Method      string `schema:"type:enum,desc:HTTP method,enum:POST|PUT|PATCH,default:POST,category:basic"`

    // Headers
    Headers     map[string]string `schema:"type:object,desc:HTTP headers,category:basic"`

    // Advanced settings
    Timeout     int    `schema:"type:int,desc:Request timeout in seconds,min:1,max:300,default:30,category:advanced"`
    Retries     int    `schema:"type:int,desc:Retry attempts,min:0,max:10,default:3,category:advanced"`
    RetryDelay  int    `schema:"type:int,desc:Delay between retries in ms,min:100,default:1000,category:advanced"`

    // Content
    ContentType string `schema:"type:string,desc:Content-Type header,default:application/json,category:advanced"`

    // Ports configuration
    Ports       component.Ports `schema:"type:ports,desc:Input ports for messages"`
}
```

**See:** [Schema Tags Guide](./schema-tags.md) for complete tag reference

### Step 3: Implement Component

**output/httpwebhook/httpwebhook.go:**

```go
package httpwebhook

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "log/slog"
    "net/http"
    "time"

    "github.com/c360/semstreams/component"
    "github.com/c360/semstreams/message"
)

// HTTPWebhook sends messages to HTTP endpoints
type HTTPWebhook struct {
    config Config
    client *http.Client
    logger *slog.Logger
}

// ConfigSchema returns the schema for this component
func (h *HTTPWebhook) ConfigSchema() component.ConfigSchema {
    return component.ExtractSchema(Config{})
}

// PortsSchema returns the ports schema
func (h *HTTPWebhook) PortsSchema() component.PortsSchema {
    return component.PortsSchema{
        Inputs: []component.PortDefinition{
            {
                Name:        "webhook_in",
                Subject:     "{configurable}",
                Type:        "jetstream",
                Description: "Messages to send to webhook",
            },
        },
        Outputs: []component.PortDefinition{}, // No outputs
    }
}

// Initialize configures the component
func (h *HTTPWebhook) Initialize(ctx context.Context, config interface{}) error {
    cfg, ok := config.(map[string]interface{})
    if !ok {
        return fmt.Errorf("invalid config type")
    }

    // Parse configuration
    configJSON, err := json.Marshal(cfg)
    if err != nil {
        return fmt.Errorf("failed to marshal config: %w", err)
    }

    if err := json.Unmarshal(configJSON, &h.config); err != nil {
        return fmt.Errorf("failed to unmarshal config: %w", err)
    }

    // Validate required fields
    if h.config.URL == "" {
        return fmt.Errorf("url is required")
    }

    // Create HTTP client with timeout
    h.client = &http.Client{
        Timeout: time.Duration(h.config.Timeout) * time.Second,
    }

    h.logger = slog.Default().With("component", "httpwebhook", "url", h.config.URL)

    return nil
}

// Start begins processing
func (h *HTTPWebhook) Start(ctx context.Context) error {
    h.logger.Info("HTTP webhook output started",
        "url", h.config.URL,
        "method", h.config.Method,
        "timeout", h.config.Timeout)
    return nil
}

// Stop performs cleanup
func (h *HTTPWebhook) Stop(ctx context.Context) error {
    h.logger.Info("HTTP webhook output stopped")
    if h.client != nil {
        h.client.CloseIdleConnections()
    }
    return nil
}

// Process sends a message to the webhook
func (h *HTTPWebhook) Process(ctx context.Context, msg message.Message) error {
    // Serialize message
    payload, err := json.Marshal(msg)
    if err != nil {
        return fmt.Errorf("failed to marshal message: %w", err)
    }

    // Send with retries
    var lastErr error
    for attempt := 0; attempt <= h.config.Retries; attempt++ {
        if attempt > 0 {
            // Wait before retry
            select {
            case <-ctx.Done():
                return ctx.Err()
            case <-time.After(time.Duration(h.config.RetryDelay) * time.Millisecond):
            }
            h.logger.Debug("Retrying webhook request", "attempt", attempt)
        }

        // Send request
        if err := h.sendRequest(ctx, payload); err != nil {
            lastErr = err
            h.logger.Warn("Webhook request failed",
                "error", err,
                "attempt", attempt+1,
                "max_attempts", h.config.Retries+1)
            continue
        }

        // Success
        return nil
    }

    return fmt.Errorf("webhook request failed after %d attempts: %w", h.config.Retries+1, lastErr)
}

// sendRequest sends HTTP request
func (h *HTTPWebhook) sendRequest(ctx context.Context, payload []byte) error {
    // Create request
    req, err := http.NewRequestWithContext(ctx, h.config.Method, h.config.URL, bytes.NewReader(payload))
    if err != nil {
        return fmt.Errorf("failed to create request: %w", err)
    }

    // Set headers
    req.Header.Set("Content-Type", h.config.ContentType)
    for key, value := range h.config.Headers {
        req.Header.Set(key, value)
    }

    // Send request
    resp, err := h.client.Do(req)
    if err != nil {
        return fmt.Errorf("request failed: %w", err)
    }
    defer resp.Body.Close()

    // Read response body (for logging)
    body, _ := io.ReadAll(resp.Body)

    // Check status
    if resp.StatusCode < 200 || resp.StatusCode >= 300 {
        return fmt.Errorf("unexpected status: %d, body: %s", resp.StatusCode, string(body))
    }

    h.logger.Debug("Webhook request successful", "status", resp.StatusCode)
    return nil
}

// IsHealthy returns health status
func (h *HTTPWebhook) IsHealthy() bool {
    return h.client != nil
}

// GetStatus returns component status
func (h *HTTPWebhook) GetStatus() component.Status {
    return component.Status{
        Name:    "httpwebhook",
        Healthy: h.IsHealthy(),
        Details: map[string]interface{}{
            "url":     h.config.URL,
            "method":  h.config.Method,
            "timeout": h.config.Timeout,
        },
    }
}
```

### Step 4: Write Tests

**output/httpwebhook/httpwebhook_test.go:**

```go
package httpwebhook

import (
    "context"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"
    "time"

    "github.com/c360/semstreams/message"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestHTTPWebhook_Process(t *testing.T) {
    // Create test server
    var receivedPayload []byte
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        receivedPayload, _ = io.ReadAll(r.Body)
        w.WriteHeader(http.StatusOK)
    }))
    defer server.Close()

    // Create component
    webhook := &HTTPWebhook{}
    config := map[string]interface{}{
        "url":    server.URL,
        "method": "POST",
        "timeout": 5,
    }

    // Initialize
    err := webhook.Initialize(context.Background(), config)
    require.NoError(t, err)

    // Start
    err = webhook.Start(context.Background())
    require.NoError(t, err)

    // Create test message
    payload := &TestPayload{Data: "test"}
    msg := message.NewBaseMessage(
        payload.Schema(),
        payload,
        "test-source",
    )

    // Process message
    err = webhook.Process(context.Background(), msg)
    require.NoError(t, err)

    // Verify payload received
    assert.NotEmpty(t, receivedPayload)

    // Stop
    err = webhook.Stop(context.Background())
    require.NoError(t, err)
}

func TestHTTPWebhook_Retry(t *testing.T) {
    attempts := 0
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        attempts++
        if attempts < 3 {
            w.WriteHeader(http.StatusInternalServerError)
        } else {
            w.WriteHeader(http.StatusOK)
        }
    }))
    defer server.Close()

    webhook := &HTTPWebhook{}
    config := map[string]interface{}{
        "url":         server.URL,
        "retries":     3,
        "retry_delay": 100,
    }

    webhook.Initialize(context.Background(), config)
    webhook.Start(context.Background())

    payload := &TestPayload{Data: "test"}
    msg := message.NewBaseMessage(payload.Schema(), payload, "test")

    err := webhook.Process(context.Background(), msg)
    require.NoError(t, err)
    assert.Equal(t, 3, attempts, "should retry until success")
}

type TestPayload struct {
    Data string `json:"data"`
}

func (t *TestPayload) Schema() message.Type {
    return message.Type{Domain: "test", Category: "payload", Version: "v1"}
}

func (t *TestPayload) Validate() error { return nil }
func (t *TestPayload) MarshalJSON() ([]byte, error) {
    type Alias TestPayload
    return json.Marshal((*Alias)(t))
}
func (t *TestPayload) UnmarshalJSON(data []byte) error {
    type Alias TestPayload
    return json.Unmarshal(data, (*Alias)(t))
}
```

### Step 5: Register Component

**componentregistry/register.go:**

```go
package componentregistry

import (
    "github.com/c360/semstreams/component"
    "github.com/c360/semstreams/output/httpwebhook"
)

func RegisterAll(registry *component.Registry) error {
    // ... other registrations ...

    // Register HTTP webhook
    if err := registry.RegisterFactory("httpwebhook", &component.Registration{
        Description: "HTTP webhook output for posting messages to HTTP endpoints",
        Type:        "output",
        Protocol:    "http",
        Domain:      "network",
        Version:     "1.0.0",
        Factory: func() (component.Component, error) {
            return &httpwebhook.HTTPWebhook{}, nil
        },
        Schema: httpwebhook.Config{},
    }); err != nil {
        return err
    }

    return nil
}
```

### Step 6: Generate Schema

```bash
task schema:generate
```

This generates `schemas/httpwebhook.v1.json` and updates `specs/openapi.v3.yaml`.

### Step 7: Use in Configuration

**config.json:**

```json
{
  "components": {
    "my-webhook": {
      "type": "output",
      "component_type": "httpwebhook",
      "enabled": true,
      "config": {
        "url": "https://example.com/webhook",
        "method": "POST",
        "headers": {
          "Authorization": "Bearer ${WEBHOOK_TOKEN}"
        },
        "timeout": 30,
        "retries": 3,
        "ports": {
          "inputs": [{
            "name": "webhook_in",
            "subject": "events.alerts.*",
            "type": "jetstream"
          }]
        }
      }
    }
  }
}
```

---

## Port Configuration

Components communicate via NATS subjects defined in ports configuration.

### Input Ports

**Define subjects to subscribe to:**

```go
type Config struct {
    Ports component.Ports `schema:"type:ports,desc:Input and output ports"`
}
```

**In config.json:**

```json
{
  "ports": {
    "inputs": [
      {
        "name": "primary_in",
        "subject": "raw.udp.messages",
        "type": "jetstream",
        "pattern": "pubsub"
      }
    ]
  }
}
```

### Output Ports

**Define subjects to publish to:**

```json
{
  "ports": {
    "outputs": [
      {
        "name": "primary_out",
        "subject": "parsed.json",
        "type": "jetstream",
        "pattern": "pubsub"
      }
    ]
  }
}
```

### Subject Templates

Use templates to include message fields in subjects:

```json
{
  "subject": "events.graph.entity.{entity_type}"
}
```

Message with `{"entity_type": "drone"}` → Published to `events.graph.entity.drone`

**See:** [Routing Guide](../basics/03-routing.md) for complete port documentation

---

## Message Handling

### Creating Messages

```go
import "github.com/c360/semstreams/message"

// Create payload
payload := &MyPayload{
    Field1: "value",
    Field2: 123,
}

// Create message
msg := message.NewBaseMessage(
    payload.Schema(),
    payload,
    "my-component", // source
)

// Validate
if err := msg.Validate(); err != nil {
    return err
}

// Publish to NATS
nc.Publish("my.subject", msg)
```

### Processing Messages

```go
func (c *MyComponent) Process(ctx context.Context, msg message.Message) error {
    // Extract payload
    payload := msg.Payload()

    // Type assertion
    myPayload, ok := payload.(*MyPayload)
    if !ok {
        return fmt.Errorf("unexpected payload type")
    }

    // Process
    result := processData(myPayload)

    // Create output message
    outputMsg := message.NewBaseMessage(
        result.Schema(),
        result,
        "my-component",
    )

    // Publish result
    return c.publishMessage(outputMsg)
}
```

**See:** [Message System Guide](../basics/05-message-system.md) for complete message documentation

---

## Best Practices

### 1. Configuration Validation

```go
func (c *MyComponent) Initialize(ctx context.Context, config interface{}) error {
    // Parse config
    if err := c.parseConfig(config); err != nil {
        return fmt.Errorf("parse config: %w", err)
    }

    // Validate required fields
    if c.config.RequiredField == "" {
        return fmt.Errorf("required_field is required")
    }

    // Validate constraints
    if c.config.Port < 1 || c.config.Port > 65535 {
        return fmt.Errorf("port must be 1-65535")
    }

    return nil
}
```

### 2. Context Handling

```go
func (c *MyComponent) Process(ctx context.Context, msg message.Message) error {
    // Respect context cancellation
    select {
    case <-ctx.Done():
        return ctx.Err()
    default:
    }

    // Pass context to downstream operations
    result, err := c.processWithTimeout(ctx, msg)
    if err != nil {
        return err
    }

    return c.publishResult(ctx, result)
}
```

### 3. Error Handling

```go
import "github.com/c360/semstreams/errors"

func (c *MyComponent) Process(ctx context.Context, msg message.Message) error {
    // Transient errors (retry)
    if err := c.sendRequest(ctx); err != nil {
        return errors.WrapTransient(err, "MyComponent", "Process", "send request")
    }

    // Fatal errors (don't retry)
    if err := c.parseConfig(config); err != nil {
        return errors.WrapFatal(err, "MyComponent", "Initialize", "parse config")
    }

    // Invalid input (client error)
    if err := msg.Validate(); err != nil {
        return errors.WrapInvalid(err, "MyComponent", "Process", "validate message")
    }

    return nil
}
```

### 4. Logging

```go
import "log/slog"

type MyComponent struct {
    logger *slog.Logger
}

func (c *MyComponent) Initialize(ctx context.Context, config interface{}) error {
    // Create logger with component context
    c.logger = slog.Default().With(
        "component", "my-component",
        "instance", c.config.Name,
    )

    c.logger.Info("Component initialized",
        "port", c.config.Port,
        "timeout", c.config.Timeout)

    return nil
}

func (c *MyComponent) Process(ctx context.Context, msg message.Message) error {
    c.logger.Debug("Processing message",
        "message_id", msg.ID(),
        "type", msg.Type())

    if err := c.processMessage(msg); err != nil {
        c.logger.Error("Processing failed",
            "error", err,
            "message_id", msg.ID())
        return err
    }

    c.logger.Info("Message processed successfully",
        "message_id", msg.ID())
    return nil
}
```

### 5. Testing

```go
// Use testcontainers for integration tests
func TestMyComponent_Integration(t *testing.T) {
    // Start NATS testcontainer
    natsContainer := startNATSContainer(t)
    defer natsContainer.Terminate(context.Background())

    // Create component
    component := &MyComponent{}
    config := map[string]interface{}{
        "nats_url": natsContainer.URL(),
    }

    // Initialize and start
    component.Initialize(context.Background(), config)
    component.Start(context.Background())
    defer component.Stop(context.Background())

    // Test behavior
    // ...
}
```

---

## Common Patterns

### Input Component Pattern

```go
type MyInput struct {
    config Config
    conn   net.Conn
    cancel context.CancelFunc
}

func (i *MyInput) Start(ctx context.Context) error {
    ctx, i.cancel = context.WithCancel(ctx)

    // Open connection
    conn, err := net.Dial("tcp", i.config.Address)
    if err != nil {
        return err
    }
    i.conn = conn

    // Start reading in goroutine
    go i.readLoop(ctx)

    return nil
}

func (i *MyInput) readLoop(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            // Read data
            data, err := i.readData()
            if err != nil {
                continue
            }

            // Create message and publish
            msg := i.createMessage(data)
            i.publishMessage(msg)
        }
    }
}

func (i *MyInput) Stop(ctx context.Context) error {
    if i.cancel != nil {
        i.cancel()
    }
    if i.conn != nil {
        i.conn.Close()
    }
    return nil
}
```

### Processor Component Pattern

```go
type MyProcessor struct {
    config Config
    nc     *nats.Conn
    sub    *nats.Subscription
}

func (p *MyProcessor) Start(ctx context.Context) error {
    // Subscribe to input subject
    sub, err := p.nc.Subscribe(p.config.Ports.Inputs[0].Subject, func(m *nats.Msg) {
        // Deserialize message
        var msg message.Message
        json.Unmarshal(m.Data, &msg)

        // Process
        p.Process(context.Background(), msg)
    })
    if err != nil {
        return err
    }
    p.sub = sub

    return nil
}

func (p *MyProcessor) Process(ctx context.Context, msg message.Message) error {
    // Transform data
    result := p.transform(msg.Payload())

    // Create output message
    outputMsg := message.NewBaseMessage(result.Schema(), result, "my-processor")

    // Publish to output subject
    data, _ := json.Marshal(outputMsg)
    p.nc.Publish(p.config.Ports.Outputs[0].Subject, data)

    return nil
}

func (p *MyProcessor) Stop(ctx context.Context) error {
    if p.sub != nil {
        p.sub.Unsubscribe()
    }
    return nil
}
```

---

## Troubleshooting

### Component Not Loading

**Check:**
1. Registered in `componentregistry/register.go`
2. Factory function returns correct type
3. Schema struct provided in registration
4. No compilation errors

### Configuration Errors

**Check:**
1. Schema tags valid: `type:`, `desc:` required
2. JSON config matches schema
3. Required fields present (no default = required)
4. Validation logic in Initialize()

### Port Connection Issues

**Check:**
1. Port subjects configured in config.json
2. Subject patterns match (wildcards correct)
3. NATS connection healthy
4. Component subscribed to correct subjects

---

## Next Steps

**Essential Reading:**
- [Message System](../basics/05-message-system.md) - Message and payload creation
- [Schema Tags](./schema-tags.md) - Configuration schema reference
- [Routing](../basics/03-routing.md) - Port configuration and NATS subjects

**Testing:**
- [Testing Guide](./testing.md) - Component testing strategies
- [Contributing](./contributing.md) - Development workflow

**Examples:**
- `semstreams/input/udp/` - Input component example
- `semstreams/processor/json_generic/` - Processor component example
- `semstreams/output/httppost/` - Output component example

---

**Key Takeaway:** Components are self-contained processing units with clear lifecycle, configuration via schema tags, and communication via NATS subjects. Follow the patterns in this guide for reliable, testable components.
