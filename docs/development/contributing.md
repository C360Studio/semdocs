# Contributing to SemStreams

Thank you for your interest in contributing to the SemStreams ecosystem!

## Development Workflow

SemStreams uses **specialized AI agents** to coordinate development work. See [Agent Workflow](./agents.md) for the full TDD-based process.

### Quick Overview

1. **architect** → Designs API contracts and system architecture
2. **go-developer** → Implements Go backend with tests
3. **go-reviewer** → Reviews Go code quality
4. **svelte-developer** → Implements frontend with tests
5. **svelte-reviewer** → Reviews Svelte code quality
6. **technical-writer** → Updates documentation

## Getting Started

### Prerequisites

- Go 1.21+ (for semstreams, semmem)
- Rust 1.75+ (for semembed, semsummarize)
- Node.js 20+ (for semstreams-ui)
- Docker & Docker Compose
- Task runner: `brew install go-task` or see [taskfile.dev](https://taskfile.dev)

### Development Setup

```bash
# Clone repositories
git clone https://github.com/c360/semstreams
git clone https://github.com/c360/semstreams-ui
git clone https://github.com/c360/semembed

# Start dependencies (NATS, etc.)
cd semstreams
task integration:start

# Run service in dev mode
task dev
```

## Code Standards

### Go (semstreams, semmem)

**Error Handling:**
```go
import "github.com/c360/semstreams/errors"

// Use typed errors
return errors.WrapTransient(err, "Component", "Method", "action")  // Temporary
return errors.WrapFatal(err, "Component", "Method", "action")      // Programming error
return errors.WrapInvalid(err, "Component", "Method", "action")    // Invalid input
```

**Context Management:**
```go
// ALWAYS pass context as first parameter
func ProcessData(ctx context.Context, input Data) (Result, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()
    // ...
}
```

**Testing:**
```go
// Use testcontainers for integration tests
func TestWithNATS(t *testing.T) {
    natsClient := getSharedNATSClient(t)  // Real NATS, not mocks
    // Test actual behavior
}
```

See individual repo CONTRIBUTING.md for more details.

### Svelte 5 (semstreams-ui)

**Use Runes System:**
```svelte
<script>
    let { data, onUpdate = $bindable() } = $props();
    let processed = $derived(transform(data));

    $effect(() => {
        // Side effects with cleanup
    });
</script>
```

**Event Handling:**
```svelte
<!-- Events as properties -->
<button onclick={() => handleClick()}>Click me</button>
```

See [semstreams-ui/CONTRIBUTING.md](https://github.com/c360/semstreams-ui/blob/main/CONTRIBUTING.md).

### Rust (semembed, semsummarize)

**No local installation required** - all development via Docker:

```bash
cd semembed
task dev          # Build + run + test
task test:all     # Run all tests
task logs         # View logs
```

See service-specific READMEs for details.

## Testing Requirements

### Test Coverage
- **Critical paths**: 80% minimum
- **Happy path + error cases**: Required
- **Edge cases**: Document in test comments

### Test Quality
- Test **behavior**, not implementation
- Avoid arbitrary sleeps - use explicit synchronization
- Table-driven tests for Go
- Component tests for Svelte

See [Testing Philosophy](./testing.md) for details.

## Pull Request Process

1. **Create issue** (or pick from `bd ready`)
2. **Create branch**: `feat/description` or `fix/description`
3. **Write tests first** (TDD)
4. **Implement feature**
5. **Run quality checks**:
   ```bash
   task lint
   task test
   task test:race
   task e2e
   ```
6. **Update documentation**
7. **Submit PR** with:
   - Clear description
   - Link to issue
   - Test coverage proof
   - Screenshots (if UI changes)

### PR Review
- **go-reviewer** checks Go code
- **svelte-reviewer** checks Svelte code
- Must pass CI (tests, lints, security scans)
- Requires 1 approval (maintainer)

## Commit Messages

Use **conventional commits**:

```
<type>(scope): subject

feat(indexmanager): add semantic search caching
fix(graphql): handle nil pointer in resolver
docs(pathrag): update examples
test(clustering): add integration tests
refactor(embedder): extract BM25 to separate package
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`

## Issue Tracking

SemStreams uses **bd (beads)** for issue tracking:

```bash
# Find ready work
bd ready

# Show issue details
bd show semstreams-abc

# Update status
bd update semstreams-abc --status in_progress

# Close when done
bd close semstreams-abc
```

See [Agent Workflow](./agents.md) for the full process.

## Documentation

- **Ecosystem docs**: Update this repository (semdocs)
- **Implementation docs**: Update service repository
- **API changes**: Update OpenAPI/GraphQL schemas
- **Breaking changes**: Update CHANGELOG.md

## Code of Conduct

- Be respectful and inclusive
- Focus on constructive feedback
- Help newcomers
- Follow the technical standards
- Keep discussions professional

## Questions?

- **General questions**: [Discussions](https://github.com/c360/semstreams/discussions)
- **Bug reports**: Service-specific issue tracker
- **Documentation**: [semdocs issues](https://github.com/c360/semdocs/issues)

Thank you for contributing to SemStreams!
