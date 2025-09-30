# ADR-0020: TLS Infrastructure and Deployment Architecture

## Table of Contents

1. [Status](#status)
2. [Context](#context)
3. [Decision](#decision)
   - [Complete TLS Capability](#complete-tls-capability)
   - [Server Implementation](#server-implementation)
   - [Client Implementation with Certificate Discovery](#client-implementation-with-certificate-discovery)
4. [Deployment Scenarios and Certificate Management](#deployment-scenarios-and-certificate-management)
   - [Single Developer (Combined Mode)](#single-developer-combined-mode)
   - [Collaboration Development (Distributed)](#collaboration-development-distributed)
   - [SaaS Production (AWS/Cloud)](#saas-production-awscloud)
   - [Enterprise/Self-Hosted Production](#enterpriseself-hosted-production)
   - [Container/Kubernetes Deployment](#containerkubernetes-deployment)
5. [Rationale](#rationale)
   - [Design Principles](#design-principles)
   - [Integration with Existing Architecture](#integration-with-existing-architecture)
6. [Consequences](#consequences)
   - [Positive](#positive)
   - [Negative](#negative)
   - [Mitigations](#mitigations)
7. [Success Criteria](#success-criteria)
   - [Server-Side TLS](#server-side-tls)
   - [Client-Side TLS](#client-side-tls)
   - [Integration & Deployment](#integration--deployment)
8. [Implementation Dependencies](#implementation-dependencies)
9. [Alternatives Considered](#alternatives-considered)
10. [References](#references)
11. [Notes](#notes)

## Status

Accepted

## Context

[ADR-0002: gRPC Communication](0002-gRPC.md) establishes a comprehensive "TLS Everywhere Policy" with enterprise-grade certificate management and production-ready operational capabilities.

Ooloi requires TLS support that:
- Works transparently across all deployment scenarios (development, production, enterprise, cloud)
- Enables TLS with minimal configuration friction
- Supports automatic certificate generation for development workflows
- Integrates with enterprise PKI infrastructure
- Works with both server and client components

The implementation must balance developer productivity (zero-friction local development) with production requirements (secure communication, proper certificate validation).

## Decision

Implement comprehensive TLS support for both gRPC server and client components:

### Complete TLS Capability
**Goal**: Full TLS support for ANY deployment scenario - development, production, enterprise, cloud  
**Timeline**: Immediate (production readiness requirement)

**Deliverables:**
- Complete environment variable and CLI switch support for all TLS configuration (server and client)
- TLS disabled by default for immediate developer productivity
- `OOLOI_TLS=true` or `--tls true` enables TLS with intelligent certificate management
- TLS-specific CLI overrides: `--tls true/false`, `--cert-path` (server's public certificate), `--key-path` (server's private key, backend only)
- Auto-generated self-signed certificates for development and testing (server-side)
- Custom certificate support for production and enterprise deployments
- Certificate creation at specified paths or platform-appropriate defaults
- TLS validation test in `system_integration_test.clj`
- Client TLS support using one-way TLS (server authentication only) with automatic certificate discovery
- Works transparently across all deployment scenarios

**Test Certificate Infrastructure:**
Ooloi automatically generates certificates for development and testing:
- **Generation**: Pure Java implementation using Bouncy Castle
- **Properties**: RSA 2048-bit, 20-year validity, self-signed
- **Subject**: `CN=localhost` with organizational unit `Ooloi`
- **SANs**: `localhost`, `127.0.0.1`, `::1` for comprehensive local coverage
- **Usage**: Automatic generation during TLS-enabled startup, no manual intervention required

**Production Certificate Auto-generation:**
- **Implementation**: Pure Java using Bouncy Castle cryptography (cross-platform, no external dependencies)
- **Generation**: Automatic when `OOLOI_TLS=true` and certificates don't exist at target locations
- **Certificate properties**: RSA 2048-bit, 20-year validity, X.509 with Subject Alternative Names  
- **Storage**: Certificates created at cert-path/key-path if specified, otherwise platform-appropriate defaults

### Server Implementation

```clojure
;; gRPC server with auto-generating TLS support
(let [tls-enabled? (get-tls-enabled config)  ; defaults to false
      tls-config (when tls-enabled? (ensure-dev-certificates))
      server-builder (if tls-config
                       (-> (ServerBuilder/forPort port)
                           (.useTransportSecurity
                            (File. (:cert-path tls-config))
                            (File. (:key-path tls-config))))
                       (ServerBuilder/forPort port))]
  ;; ... rest of server setup
```

### Client Implementation with Certificate Discovery

The client implements one-way TLS (server authentication only) with priority-based certificate resolution:

```clojure
;; gRPC client with automatic certificate discovery
(case transport
  :in-process
  ;; In-process: TLS irrelevant (no network communication)
  (build-in-process-channel server-name)

  :network
  (if (not tls-enabled?)
    ;; TLS disabled: plaintext (development default)
    (build-plaintext-channel host port)

    ;; TLS enabled: one-way TLS with automatic certificate discovery
    (cond
      ;; Priority 1: Explicit server cert provided
      cert-path
      (build-tls-channel-with-explicit-cert host port cert-path)

      ;; Priority 2: Server cert exists (same machine development)
      (server-cert-exists?)
      (build-tls-channel-with-server-cert host port (discover-server-cert))

      ;; Priority 3: System trust store (production CA-signed)
      :else
      (build-tls-channel-with-system-trust-store host port))))
```

**One-Way TLS Architecture:**
- **Server Authentication**: Client verifies server identity using server's public certificate
- **Client Identity**: Application-level authentication via client-id (not certificate-based)
- **Security Model**: Appropriate for collaboration software (not banking/military requiring mTLS)
- **Client Configuration**: Only needs server's public certificate, never private keys

**Certificate Discovery Strategy:**

| Scenario | Transport | TLS Flag | Cert Config | Server Cert Found | Client Behavior | Use Case |
|----------|-----------|----------|-------------|-------------------|-----------------|----------|
| **Dev (plaintext)** | `:network` | `false` | - | - | `.usePlaintext()` | Default development |
| **Dev (same machine)** | `:network` | `true` | None | Yes @ `~/.ooloi/certs/` | Trust server's self-signed cert | Local dev with TLS |
| **Dev (explicit cert)** | `:network` | `true` | Server cert path | - | Use explicit server cert | Custom dev setup |
| **Production (CA-signed)** | `:network` | `true` | None | No | System trust store (cacerts) | Production deployment |
| **Enterprise (custom CA)** | `:network` | `true` | CA cert path | - | Trust custom CA | Enterprise PKI |
| **Combined mode** | `:in-process` | Any | Any | Any | In-process (no network) | Single-JVM deployment |

**Certificate Discovery Locations:**
- **Unix/macOS**: `~/.ooloi/certs/*.crt` (discovers single .crt file)
- **Windows**: `%APPDATA%\Ooloi\certs\*.crt` (discovers single .crt file)
- **System trust store**: `$JAVA_HOME/lib/security/cacerts` (~150 trusted CAs)

## Deployment Scenarios and Certificate Management

### Single Developer (Combined Mode)
**Architecture**: Frontend and backend in same process with in-process gRPC communication  
**Certificate Strategy**: No TLS needed - in-process communication bypasses network layer  
**Configuration**:
```bash
# Combined mode - in-process transport, no network communication
./ooloi-combined

# TLS flags ignored in combined mode (no network transport)
OOLOI_TLS=true ./ooloi-combined  # TLS setting has no effect
./ooloi-combined --tls true      # TLS setting has no effect
```

**In-Process Transport Characteristics**:
- **Communication**: Direct Java method calls within same JVM process
- **Security**: Process isolation provides security boundary
- **Performance**: No network serialization overhead
- **TLS**: Not applicable - no network communication occurs
- **Transport**: Uses gRPC in-process transport (ADR-0019)

**Development Benefits**:
- **Zero configuration**: No certificate management needed
- **Immediate startup**: No TLS handshake or certificate validation delays  
- **Maximum performance**: Direct in-memory communication with 37.5-75x faster response times (98.7-99.3% latency reduction vs network gRPC, per ADR-0019)
- **No network dependencies**: Works offline or in restricted network environments

### Collaboration Development (Distributed)
**Architecture**: Multiple client machines connecting to shared development server  
**Certificate Strategy**: Proper certificates with server hostname/IP coverage  
**Use Case**: Team developing collaboration features, multi-client testing

```bash
# Development server with proper certificates
./ooloi-backend --tls true --cert-path ./dev-server.crt --key-path ./dev-server.key

# Clients connecting to dev-server.local
# Certificate must cover dev-server.local and be trusted by client machines
```

**Certificate Requirements**:
- **Coverage**: Must include server's actual hostname/IP addresses
- **Distribution**: Certificate must be trusted by all client development machines
- **Generation**: Manual certificate creation or team CA infrastructure
- **Rationale**: Multi-client scenarios mirror production networking, require realistic certificate management

### SaaS Production (AWS/Cloud)
**Architecture**: Backend on AWS behind Application Load Balancer  
**Certificate Strategy**: AWS Certificate Manager with TLS termination at load balancer  
**Client Experience**: Standard HTTPS with automatic certificate trust

```bash
# Backend configuration (behind ALB with TLS termination)
export OOLOI_TLS=false  # ALB handles TLS termination
export OOLOI_PORT=8080  # Internal HTTP port
export OOLOI_DEPLOYMENT_MODE=backend
java -jar ooloi-backend.jar
```

**SaaS Architecture Pattern**:
```
Client (Desktop App)
    ↓ HTTPS (ACM Certificate - api.ooloi-saas.com)
AWS Application Load Balancer
    ↓ HTTP (Internal AWS Network)
Ooloi Backend (EC2/ECS/EKS)
```

**Certificate Management Benefits**:
- **Automatic Trust**: AWS Certificate Manager provides certificates trusted by all browsers/OS
- **Zero Client Configuration**: Clients connect with standard HTTPS, no manual certificate management
- **Automatic Renewal**: AWS handles certificate lifecycle and renewal
- **Professional Security**: Standard CA-signed certificates provide production-grade security posture
- **Multi-Domain Support**: Single certificate covers `*.ooloi-saas.com` subdomains

### Enterprise/Self-Hosted Production
**Architecture**: Customer-managed infrastructure with internal certificate authority  
**Certificate Strategy**: Customer-provided certificates from internal PKI or commercial CA  
**Configuration**:

```bash
# Enterprise deployment with internal CA certificates
OOLOI_TLS=true OOLOI_CERT_PATH=/etc/certs/ooloi.crt OOLOI_KEY_PATH=/etc/certs/ooloi.key ./ooloi-backend

# Production with commercial CA certificates
./ooloi-backend --tls true --cert-path /etc/ssl/ooloi.crt --key-path /etc/ssl/ooloi.key --port 443
```

**Enterprise Certificate Characteristics**:
- **Source**: Internal PKI, commercial CA (DigiCert, GlobalSign, etc.), or Let's Encrypt
- **Trust**: Trusted by corporate infrastructure and client machines
- **Management**: Customer responsibility, integrated with existing certificate management processes
- **Compliance**: Meets corporate security policies and audit requirements

### Container/Kubernetes Deployment
**Architecture**: Containerized deployment with external certificate management  
**Certificate Strategy**: Kubernetes secrets, external certificate providers, or cert-manager integration

```bash
# Container deployment with mounted certificate secrets
OOLOI_TLS=true OOLOI_CERT_PATH=/secrets/tls.crt OOLOI_KEY_PATH=/secrets/tls.key OOLOI_PORT=8443 ./ooloi-backend
```

**Configuration Examples Summary**:
```bash
# Combined Mode: In-process transport, no TLS needed
./ooloi-combined  # TLS flags ignored

# Backend-only: TLS disabled by default
./ooloi-backend

# Backend-only: TLS with auto-generated certificates
OOLOI_TLS=true ./ooloi-backend
./ooloi-backend --tls true

# Collaboration Development: Custom certificates for multi-client scenarios
./ooloi-backend --tls true --cert-path ./dev-server.crt --key-path ./dev-server.key

# SaaS Production: TLS termination at load balancer
OOLOI_TLS=false OOLOI_PORT=8080 ./ooloi-backend  # Behind AWS ALB

# Enterprise: Customer-managed certificates
OOLOI_TLS=true OOLOI_CERT_PATH=/etc/ssl/ooloi.crt OOLOI_KEY_PATH=/etc/ssl/ooloi.key ./ooloi-backend

# Debugging: Explicit TLS disable
./ooloi-backend --tls false
```

## Rationale

### Design Principles

1. **Developer Experience First**: TLS disabled by default, enabled with single environment variable
2. **Zero Setup Friction**: Auto-generated certificates eliminate manual certificate management
3. **Immediate Productivity**: Developers can start coding immediately without certificate setup
4. **Debug Capability**: TLS easily toggled on/off for testing and troubleshooting
5. **Production Ready**: Supports enterprise PKI, custom certificates, and CA-signed certificates
6. **Client Intelligence**: Automatic certificate discovery for development, system trust store for production

### Integration with Existing Architecture

**Component Integration**: TLS configuration integrates cleanly with existing Integrant lifecycle management
**Deployment Model Support**: 
- **Combined mode**: In-process transport (no network communication, TLS not applicable)
- **Distributed mode**: Network transport with TLS requirement for secure communication
- **Backend-only mode**: Network server requiring TLS for external client connections

**Configuration Precedence**: Environment variables → config files → defaults

## Consequences

### Positive

- **Developer productivity preserved** with zero-friction TLS toggle
- **Production ready** with comprehensive certificate support
- **No architectural debt** - clean integration with existing component lifecycle
- **Maintains ADR-0002 vision** for TLS everywhere policy
- **Cross-platform** using pure Java cryptography (no external dependencies)
- **Intelligent defaults** handle common scenarios automatically

### Negative

- **Certificate management complexity** for users managing custom certificates
- **Testing complexity** requiring validation across multiple deployment scenarios
- **Documentation requirements** for different certificate configurations
- **Platform-specific paths** for certificate storage require conditional logic

### Mitigations

- **Clear error messages** guide users through certificate configuration issues
- **Comprehensive testing** across all deployment scenarios (dev, prod, enterprise)
- **Complete documentation** with examples for each deployment model
- **Automatic fallbacks** reduce configuration burden (auto-generation, discovery, system trust store)

## Success Criteria

### Server-Side TLS
- gRPC server starts immediately with TLS disabled by default
- Complete TLS environment variable support: `OOLOI_TLS`, `OOLOI_CERT_PATH`, `OOLOI_KEY_PATH`
- Complete TLS CLI flag support: `--tls true/false`, `--cert-path`, `--key-path`
- Auto-generated certificates work without manual intervention when no custom certs provided
- Custom certificate paths work for production and enterprise deployments
- Certificate generation uses pure Java cryptography (cross-platform, no external dependencies)
- All gRPC server tests passing with comprehensive certificate management coverage

### Client-Side TLS
- Complete TLS environment variable support: `OOLOI_FRONTEND_TLS`, `OOLOI_FRONTEND_CERT_PATH`
- Complete TLS CLI flag support: `--tls true/false`, `--cert-path`
- One-way TLS implementation (server authentication only, no client certificates)
- Client certificate discovery finds server certificates automatically (same-machine development)
- Client falls back to system trust store for CA-signed certificates (production)
- Explicit certificate configuration works for enterprise PKI scenarios
- All gRPC client tests passing with comprehensive certificate discovery coverage

### Integration & Deployment
- Works transparently across development, testing, production, enterprise, and cloud scenarios
- TLS validation test passes in `system_integration_test.clj`
- Combined mode (in-process transport) works without TLS configuration
- Distributed mode (network transport) supports TLS with proper certificate validation
- Clear error messages guide users through certificate configuration issues

## Implementation Dependencies

- gRPC server component (`backend/components/grpc_server.clj`)
- gRPC client components (`frontend/grpc/event_client.clj`, `frontend/grpc/api_client.clj`)
- Existing Integrant configuration system
- Java gRPC TLS APIs (`ServerBuilder.useTransportSecurity`, `ManagedChannelBuilder`)
- Bouncy Castle cryptography libraries for pure Java certificate generation
- Platform-specific file system APIs for certificate storage paths

## Alternatives Considered

### 1. TLS Always Enabled
**Rejected**: Would create friction for local development and combined-mode deployments where TLS adds no value

### 2. Third-Party Certificate Management Only
**Rejected**: Would complicate development workflows and require external dependencies for basic TLS functionality

## References

### Related ADRs
- [ADR-0001: Frontend-Backend Separation](0001-Frontend-Backend-Separation.md) - Establishes deployment models requiring TLS
- [ADR-0002: gRPC Communication](0002-gRPC.md) - Defines comprehensive TLS vision this ADR implements
- [ADR-0017: System Architecture](0017-System-Architecture.md) - Component architecture supporting TLS configuration
- [ADR-0019: In-Process gRPC Transport Optimization](0019-In-Process-gRPC-Transport-Optimization.md) - In-process transport that doesn't require TLS

### Technical Documentation
- [gRPC Java TLS Documentation](https://grpc.io/docs/guides/auth/)
- [Java ServerBuilder.useTransportSecurity API](https://grpc.github.io/grpc-java/javadoc/io/grpc/ServerBuilder.html#useTransportSecurity-java.io.File-java.io.File-)
- [Let's Encrypt Integration Guides](https://letsencrypt.org/docs/)

### Security Standards
- [OWASP Transport Layer Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Transport_Layer_Security_Cheat_Sheet.html)
- [gRPC Security Best Practices](https://grpc.io/docs/guides/security/)

## Notes

This implementation prioritizes developer experience while providing production-ready TLS support. The design balances:

- **Zero-friction local development**: TLS disabled by default, auto-generated certificates when enabled
- **Production readiness**: Full support for enterprise PKI, custom certificates, and CA-signed certificates
- **Intelligent defaults**: Automatic certificate discovery and system trust store fallbacks
- **Monolithic architecture**: No unnecessary complexity (ACME, mTLS, etc.) for features not needed in a monolithic server

The TLS implementation should be thoroughly tested across all deployment scenarios (development, production, enterprise, cloud) before considering it complete.