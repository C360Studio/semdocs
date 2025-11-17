# Testing Philosophy

SemStreams testing standards and best practices.

## Core Principles

### 1. Test Behavior, Not Implementation

**Good:**
```go
func TestUserCanLogin(t *testing.T) {
    result := login("user@example.com", "password")
    assert.True(t, result.Success)
    assert.NotEmpty(t, result.Token)
}
```

**Bad:**
```go
func TestHashingAlgorithm(t *testing.T) {
    // Testing internal hash function instead of behavior
    hash := hashPassword("password")
    assert.Equal(t, "specific-hash-value", hash)
}
```

### 2. Real Dependencies Over Mocks

Use **testcontainers** for real services:

```go
func TestWithNATS(t *testing.T) {
    natsClient := getSharedNATSClient(t)  // Real NATS container
    // Test actual NATS behavior
}
```

**Why:** Mocks hide integration issues. Test against real dependencies when possible.

### 3. Explicit Synchronization

**Good:**
```go
func TestEventProcessing(t *testing.T) {
    done := make(chan struct{})

    go processor.Process(event, func() {
        close(done)
    })

    select {
    case <-done:
        // Success
    case <-time.After(5 * time.Second):
        t.Fatal("timeout")
    }
}
```

**Bad:**
```go
func TestEventProcessing(t *testing.T) {
    go processor.Process(event)
    time.Sleep(100 * time.Millisecond)  // Arbitrary sleep!
    // Check results
}
```

### 4. Table-Driven Tests (Go)

```go
func TestValidation(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        wantErr bool
    }{
        {"valid email", "user@example.com", false},
        {"missing @", "userexample.com", true},
        {"empty", "", true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := validate(tt.input)
            if (err != nil) != tt.wantErr {
                t.Errorf("got error %v, wantErr %v", err, tt.wantErr)
            }
        })
    }
}
```

## Test Types

### Unit Tests

**Scope:** Single function or package
**Dependencies:** Minimal (pure functions)
**Speed:** <1ms per test

```bash
# Go
go test ./pkg/cache -v

# Rust
cargo test

# Svelte
npm run test:unit
```

### Integration Tests

**Scope:** Multiple components working together
**Dependencies:** Real services (NATS, databases)
**Speed:** 100ms - 1s per test

```bash
# Go (with testcontainers)
INTEGRATION_TESTS=1 go test ./pkg/graphclustering

# Backend E2E
task e2e:semantic-basic
```

### End-to-End Tests

**Scope:** Full system from UI to backend
**Dependencies:** Entire stack running
**Speed:** 1-10s per test

```bash
# Full stack E2E (Playwright)
cd semstreams-ui
npm run test:e2e
```

## Coverage Standards

### Critical Paths: 80% Minimum

Critical paths include:
- Entity creation and retrieval
- Graph traversal (PathRAG)
- Semantic search
- Federation queries
- Error handling

### What to Cover

**Required:**
- ✅ Happy path (success case)
- ✅ Common errors (validation, not found)
- ✅ Edge cases (empty input, nil pointers)
- ✅ Concurrent access (if applicable)

**Optional:**
- Complex branches (if business-critical)
- Performance benchmarks
- Stress tests

**Don't Test:**
- Third-party library internals
- Generated code (unless complex)
- Trivial getters/setters

## Test Structure

### Arrange-Act-Assert (AAA)

```go
func TestSomething(t *testing.T) {
    // Arrange: Set up test data
    input := createTestInput()
    expected := createExpected()

    // Act: Execute the code under test
    result := doSomething(input)

    // Assert: Verify results
    assert.Equal(t, expected, result)
}
```

### Given-When-Then (BDD Style)

```typescript
describe('Flow validation', () => {
    it('should reject flows with cycles', () => {
        // Given: A flow with a cycle
        const flow = createFlowWithCycle()

        // When: Validating the flow
        const result = validateFlow(flow)

        // Then: Should fail with cycle error
        expect(result.valid).toBe(false)
        expect(result.errors).toContain('Cycle detected')
    })
})
```

## Common Patterns

### Testing Async Code (Go)

```go
func TestAsyncOperation(t *testing.T) {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    result := make(chan Result, 1)

    go func() {
        r := performOperation(ctx)
        result <- r
    }()

    select {
    case r := <-result:
        assert.NoError(t, r.Error)
    case <-ctx.Done():
        t.Fatal("operation timed out")
    }
}
```

### Testing Goroutine Cleanup

```go
func TestGoroutineCleanup(t *testing.T) {
    done := make(chan struct{})
    cleanup := func() { close(done) }
    defer cleanup()

    go func() {
        select {
        case <-done:
            // Clean exit
        case <-time.After(10 * time.Second):
            t.Error("goroutine did not exit")
        }
    }()

    // Perform test
    // ...
}
```

### Testing Context Cancellation

```go
func TestContextCancellation(t *testing.T) {
    ctx, cancel := context.WithCancel(context.Background())

    errCh := make(chan error, 1)
    go func() {
        errCh <- longRunningOperation(ctx)
    }()

    cancel()  // Cancel immediately

    select {
    case err := <-errCh:
        assert.Equal(t, context.Canceled, err)
    case <-time.After(1 * time.Second):
        t.Fatal("operation did not respect cancellation")
    }
}
```

### Component Testing (Svelte)

```typescript
import { render, fireEvent } from '@testing-library/svelte'
import Button from './Button.svelte'

test('button click triggers callback', async () => {
    const handleClick = vi.fn()
    const { getByRole } = render(Button, { onclick: handleClick })

    await fireEvent.click(getByRole('button'))

    expect(handleClick).toHaveBeenCalledOnce()
})
```

## Quality Checklist

Before merging:

- [ ] All tests pass (`task test`)
- [ ] No race conditions (`task test:race`)
- [ ] Integration tests pass (`task test:integration`)
- [ ] E2E tests pass (`task e2e`)
- [ ] Critical paths covered (80%+)
- [ ] Error cases tested
- [ ] Documentation updated
- [ ] No arbitrary sleeps or timeouts
- [ ] Goroutines cleaned up properly
- [ ] Context cancellation handled

## Continuous Integration

All tests run automatically on PR:

```yaml
# .github/workflows/ci.yml
- name: Unit Tests
  run: go test ./...

- name: Race Detector
  run: go test -race ./...

- name: Integration Tests
  run: INTEGRATION_TESTS=1 go test ./...

- name: E2E Tests
  run: task e2e
```

See service-specific CI workflows for details.

## Resources

- [Go Testing Best Practices](https://go.dev/doc/tutorial/add-a-test)
- [Testcontainers](https://testcontainers.com/)
- [Svelte Testing Library](https://testing-library.com/docs/svelte-testing-library/intro/)
- [Playwright E2E](https://playwright.dev/)

---

**Remember:** Good tests document behavior and catch regressions. Test what matters, skip what doesn't.
