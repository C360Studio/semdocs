# ADR-001: Tiered Security Architecture with step-ca and mTLS

**Status**: Proposed
**Date**: 2025-01-15
**Author**: SemStreams Architecture Team
**Deciders**: Engineering Leadership

## Context

SemStreams is an edge-first semantic streaming platform designed for deployment across multiple independent edge locations. The current TLS implementation supports only manual certificate management (`cert_file`/`key_file`), with no support for:

1. **Automated certificate lifecycle** - Manual renewals create operational burden
2. **Mutual TLS (mTLS)** - No client certificate validation for zero-trust security
3. **Federation trust** - No built-in trust model for multi-location deployments
4. **PKI automation** - No integration with ACME or certificate authorities

As SemStreams deployments grow to 3+ locations, manual certificate management becomes unsustainable. Organizations need automated PKI infrastructure while maintaining SemStreams' core principle: **simple by default, enterprise-ready when needed**.

### Current State

**Implemented** (`pkg/tlsutil`, `pkg/security`):
- Server TLS with manual certificates (websocket_output)
- Client TLS with CA validation (websocket_input, httppost)
- Static configuration (no runtime certificate updates)

**Planned but Not Integrated**:
- step-ca container defined in `docker-compose.services.yml`
- No ACME client code
- No mTLS support
- No automated certificate rotation

### Federation Use Case

From `docs/deployment/PRODUCTION.md:43-66`, SemStreams supports federated deployments:

```
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│  Location A  │       │  Location B  │       │  Location C  │
│              │       │              │       │              │
│  SemStreams  │──────►│  SemStreams  │──────►│  SemStreams  │
│  + NATS      │ wss:// │  + NATS      │ wss:// │  + NATS      │
└──────────────┘       └──────────────┘       └──────────────┘
```

**Security Challenge**: How to establish trust between locations without:
- Manual certificate distribution for each new location
- Centralized CA that creates single point of failure
- Complex PKI operations requiring expert staff

## Decision

Implement a **three-tier security model** that balances simplicity with enterprise PKI needs:

### Tier 1: None (Default)
- **Use Case**: Development, local testing, trusted internal networks
- **Security**: No TLS encryption
- **Complexity**: Minimal (zero configuration)
- **Status**: ✅ Already implemented

### Tier 2: Manual TLS with mTLS Support
- **Use Case**: 1-2 locations, existing PKI infrastructure, small deployments
- **Security**: TLS encryption + optional mutual authentication
- **Complexity**: Medium (manual certificate generation and renewal)
- **Implementation**: Add mTLS support to existing `pkg/tlsutil`
- **Status**: 🔨 To be implemented (Phase 1 - 5 days)

### Tier 3: Automated PKI with step-ca + ACME
- **Use Case**: 3+ locations, limited PKI expertise, enterprise scale
- **Security**: Full PKI automation with mTLS and federation support
- **Complexity**: High initial setup, low ongoing maintenance
- **Implementation**: ACME client integration, step-ca federation
- **Status**: 📋 Planned (Phase 2 - 18 days)

### Architecture: Federated PKI Model

**Recommended approach** for multi-location deployments:

```
                  Corporate Root CA (offline)
                          │
                          │ signs intermediates
          ┌───────────────┼───────────────┐
          │               │               │
    ┌─────▼─────┐   ┌─────▼─────┐   ┌─────▼─────┐
    │Location A │   │Location B │   │Location C │
    │           │   │           │   │           │
    │ step-ca   │   │ step-ca   │   │ step-ca   │
    │(Intermed.)│   │(Intermed.)│   │(Intermed.)│
    └─────┬─────┘   └─────┬─────┘   └─────┬─────┘
          │               │               │
    ┌─────▼─────┐   ┌─────▼─────┐   ┌─────▼─────┐
    │SemStreams │   │SemStreams │   │SemStreams │
    │+ NATS     │   │+ NATS     │   │+ NATS     │
    └───────────┘   └───────────┘   └───────────┘
```

**Trust Model**:
- Single offline root CA maintained by organization
- Each location runs step-ca as intermediate CA
- All locations automatically trust certificates from any location (shared root)
- Short-lived certificates (24-hour default) eliminate revocation complexity

### Configuration Schema

```json
{
  "security": {
    "tls": {
      "server": {
        "enabled": false,
        "mode": "manual",  // "manual" | "acme"

        // Tier 2: Manual TLS
        "cert_file": "/path/to/server.crt",
        "key_file": "/path/to/server.key",
        "min_version": "1.2",

        // Tier 3: ACME
        "acme": {
          "enabled": false,
          "directory_url": "https://step-ca:9000/acme/acme/directory",
          "email": "admin@example.com",
          "domains": ["semstreams.local"],
          "challenge_type": "http-01",
          "renew_before": "8h",
          "storage_path": "/data/acme"
        },

        // Tier 2/3: mTLS
        "mtls": {
          "enabled": false,
          "client_ca_files": ["/path/to/client-ca.crt"],
          "require_client_cert": true,
          "allowed_client_cns": ["location-b", "location-c"]
        }
      },

      "client": {
        "ca_files": ["/path/to/server-ca.crt"],
        "insecure_skip_verify": false,
        "min_version": "1.2",

        // Tier 2/3: Client mTLS
        "mtls": {
          "enabled": false,
          "cert_file": "/path/to/client.crt",
          "key_file": "/path/to/client.key",

          // Tier 3: ACME for client certs
          "acme": {
            "enabled": false,
            "directory_url": "https://step-ca:9000/acme/acme/directory",
            "email": "client@example.com",
            "domains": ["client.local"],
            "storage_path": "/data/acme-client"
          }
        }
      }
    }
  }
}
```

## Rationale

### Why Three Tiers?

**Tier 1 (None)** maintains SemStreams' "simple to run" philosophy:
- No security configuration required for dev/testing
- Plaintext acceptable for trusted networks
- Zero operational complexity

**Tier 2 (Manual TLS + mTLS)** serves small deployments:
- Familiar certificate management for existing PKI infrastructure
- One-time setup (2-4 hours) acceptable for 1-2 locations
- mTLS enables zero-trust without automation complexity

**Tier 3 (ACME + Federation)** addresses enterprise scale:
- Break-even point: 3+ locations (manual renewal burden too high)
- Automated 24-hour certificate rotation (no manual intervention)
- Built-in federation support for multi-location trust
- Operational savings: ~8-16 hours/year → ~2 hours/year

### Why step-ca?

**Evaluated Alternatives**:
1. **Let's Encrypt** - Public CA, not suitable for private edge networks
2. **HashiCorp Vault** - Heavy infrastructure, contradicts edge-first philosophy
3. **Manual OpenSSL** - No automation, high operational burden
4. **step-ca** - ✅ Selected

**step-ca Advantages**:
- Lightweight (single binary, minimal resources)
- ACME protocol support (standard, well-tested)
- Federation capabilities for multi-location deployments
- Short-lived certificate support (24-hour default)
- Edge-friendly (runs in container, no external dependencies)
- Active maintenance (Smallstep actively develops)

### Why ACME Over Manual at Scale?

**Operational Comparison** (per location):

| Aspect | Manual | ACME |
|--------|--------|------|
| **Initial Setup** | 2-4 hours | 8-16 hours |
| **Renewal (90 days)** | 2-4 hours | Automated |
| **Annual Burden** | 8-16 hours | ~2 hours (monitoring) |
| **Human Error Risk** | High | Low |
| **Multi-Location** | N × manual | Automated |

**Break-even Analysis**:
- 1-2 locations: Manual acceptable
- 3+ locations: ACME provides ROI within first year
- 5+ locations: ACME essential (manual unsustainable)

### Why mTLS for Federation?

**Zero-Trust Model Requirements**:
- Verify both server and client identity (not just server)
- Prevent unauthorized clients from connecting to WebSocket outputs
- Enable secure multi-location communication without VPN
- Support CN-based access control (whitelist trusted locations)

**Example Threat**: Without mTLS, any client with network access to Location A's WebSocket output port can connect. With mTLS, only clients with valid certificates signed by trusted CA can connect.

## Implementation Plan

### Phase 1: Tier 2 (mTLS Foundation) - 5 days

**Work Packages**:
1. Add `ServerMTLSConfig` and `ClientMTLSConfig` to `pkg/security/config.go`
2. Implement `LoadServerTLSConfigWithMTLS` in `pkg/tlsutil/tlsutil.go`
3. Implement `LoadClientTLSConfigWithMTLS` in `pkg/tlsutil/tlsutil.go`
4. Update `websocket_output`, `websocket_input`, `httppost` components
5. Unit tests for mTLS configuration loading
6. Integration tests for mTLS handshake validation
7. Update documentation (`OPTIONAL_SERVICES.md`, new `TLS_SETUP.md`)

**Deliverables**:
- mTLS support for federated WebSocket connections
- Backward compatible (existing configs continue working)
- Foundation for Tier 3 ACME integration

### Phase 2: Tier 3 (ACME + step-ca) - 18 days

**Work Packages**:
1. Add ACME configuration schema to `pkg/security/config.go`
2. Implement `pkg/acme/client.go` (obtain, renew, storage, account management)
3. Implement certificate hot-reload on renewal
4. Add `LoadServerTLSConfigWithACME` to `pkg/tlsutil/tlsutil.go`
5. Implement ACME → manual certificate fallback
6. HTTP-01 and TLS-ALPN-01 challenge support
7. Update components to support ACME mode
8. Unit tests (mocked ACME server)
9. Integration tests (real step-ca via testcontainers)
10. E2E tests (multi-location federation)
11. Update step-ca deployment configuration
12. Documentation (`FEDERATION_GUIDE.md`, `ACME_SETUP.md`)

**Deliverables**:
- Full PKI automation with step-ca
- Automated certificate lifecycle management
- Federation support for multi-location deployments
- Graceful degradation (ACME failure → manual certificates)

**Dependencies**:
- Go library: `github.com/go-acme/lego/v4` (ACME client)
- Infrastructure: step-ca (already defined in `docker-compose.services.yml`)

## Consequences

### Positive

**Operational**:
- Reduces certificate management burden for multi-location deployments
- Eliminates manual renewal toil (24-hour auto-renewal)
- Prevents certificate expiry outages
- Standardizes security configuration across all locations

**Security**:
- Enables zero-trust model with mTLS
- Short-lived certificates reduce compromise window
- Passive revocation (no CRL/OCSP complexity)
- Federation support without manual trust distribution

**Architectural**:
- Maintains SemStreams' "simple by default" principle (Tier 1 unchanged)
- Backward compatible (existing manual TLS configs continue working)
- Graceful degradation (ACME failure → manual certificate fallback)
- Edge-friendly (step-ca runs locally, no cloud dependencies)

### Negative

**Complexity**:
- Three configuration modes increase testing surface area
- ACME integration adds ~70 hours development effort
- step-ca deployment requires additional operational knowledge
- Hot certificate reload adds runtime state management

**Operational**:
- Initial setup cost: 8-16 hours for Tier 3 (vs 2-4 for Tier 2)
- Monitoring requirements: Track ACME renewal success/failure
- Debugging: ACME errors more complex than manual certificate issues
- Documentation burden: Need comprehensive federation guides

**Dependencies**:
- Adds `github.com/go-acme/lego/v4` dependency (~500KB)
- Requires step-ca infrastructure for Tier 3 deployments
- ACME protocol compliance (RFC 8555) - external standard

### Mitigations

**Complexity Mitigation**:
- Comprehensive integration tests with real step-ca instances
- Clear documentation with examples for each tier
- Fallback logic (ACME failure → manual certificates)
- Extensive logging for debugging ACME issues

**Operational Mitigation**:
- Taskfile commands for common operations (`task services:start:tls`)
- Health checks for step-ca availability
- Metrics for certificate expiry tracking
- Runbooks for common troubleshooting scenarios

**Dependency Mitigation**:
- Pin `lego` version to stable release
- Test ACME compliance with step-ca in CI
- Document step-ca version compatibility
- Support multiple ACME challenge types (http-01, tls-alpn-01)

## Alternatives Considered

### Alternative 1: Manual Certificates Only (Status Quo)

**Approach**: Keep current manual TLS, add only mTLS support

**Pros**:
- Minimal development effort (5 days)
- No new dependencies
- Familiar operational model

**Cons**:
- Manual renewal burden unsustainable at 3+ locations
- No federation automation
- High operational cost for growing deployments
- Certificate expiry outages likely

**Rejected**: Does not address scaling pain points for enterprise customers

### Alternative 2: HashiCorp Vault Integration

**Approach**: Integrate with Vault for PKI management

**Pros**:
- Enterprise-grade PKI platform
- Dynamic secrets management
- Existing adoption in large organizations

**Cons**:
- Heavy infrastructure (contradicts edge-first philosophy)
- Requires Vault deployment and expertise
- Network dependency (not offline-friendly)
- Complex operational model for edge devices

**Rejected**: Too heavyweight for edge deployments, violates SemStreams principles

### Alternative 3: Let's Encrypt Public Certificates

**Approach**: Use Let's Encrypt for automated certificates

**Pros**:
- Free, trusted by all browsers/systems
- Well-established ACME implementation
- No infrastructure to deploy

**Cons**:
- Requires public DNS and internet connectivity
- Not suitable for private edge networks
- 90-day certificate lifetime (vs step-ca 24-hour)
- Domain validation challenges difficult for edge devices

**Rejected**: Edge deployments often offline/private networks

### Alternative 4: Single-Tier "ACME Always"

**Approach**: Always use ACME, no manual certificate support

**Pros**:
- Simpler codebase (one configuration path)
- Forces best practices (automation)
- Reduced testing surface area

**Cons**:
- Violates "simple to run" principle
- Forces step-ca deployment even for dev/testing
- Breaks existing manual TLS deployments
- High barrier to entry for new users

**Rejected**: Contradicts SemStreams philosophy of simplicity by default

## Success Metrics

### Implementation Success (Development)

- [ ] All Tier 2 tests passing (unit + integration)
- [ ] All Tier 3 tests passing (unit + integration + E2E)
- [ ] Code review approval from go-reviewer
- [ ] Documentation complete and reviewed
- [ ] Zero breaking changes to existing configurations
- [ ] Graceful fallback tested (ACME failure → manual certs)

### Operational Success (Production)

- [ ] Certificate auto-renewal success rate > 99.9%
- [ ] Zero certificate expiry outages
- [ ] mTLS handshake failure rate < 1%
- [ ] ACME setup time < 16 hours (Tier 3)
- [ ] Manual renewal time eliminated (3+ location deployments)
- [ ] Positive user feedback on federation setup

### Adoption Metrics (6 months post-release)

- [ ] 50%+ of multi-location deployments use Tier 3
- [ ] 100% of 5+ location deployments use Tier 3
- [ ] <5 support tickets related to ACME/mTLS configuration
- [ ] <2 hours average time to resolve certificate issues

## References

### Internal Documentation

- Current TLS implementation: `pkg/tlsutil/tlsutil.go`, `pkg/security/config.go`
- Federation architecture: `docs/deployment/PRODUCTION.md:43-66`
- Optional services: `docs/OPTIONAL_SERVICES.md:94-122`
- step-ca configuration: `docker-compose.services.yml:72-95`

### External Standards

- **ACME Protocol**: RFC 8555 - Automatic Certificate Management Environment
- **TLS 1.2/1.3**: RFC 5246, RFC 8446
- **X.509 Certificates**: RFC 5280
- **mTLS**: RFC 8705 - OAuth 2.0 Mutual-TLS Client Authentication

### Tools & Libraries

- **step-ca**: https://github.com/smallstep/certificates
- **lego ACME client**: https://github.com/go-acme/lego
- **OpenSSL**: https://www.openssl.org/ (manual certificate generation)

### Related ADRs

- None (this is ADR-001)

## Appendix: Detailed Research

Full research document with implementation code examples, LOE breakdowns, and federation architecture diagrams available in project archive. Contact architecture team for access.

---

**Review Status**: Awaiting stakeholder approval
**Next Review Date**: 2025-01-22
**Approval Required From**: Engineering Lead, Product Owner, Security Team
