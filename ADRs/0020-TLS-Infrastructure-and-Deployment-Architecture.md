# ADR-0020: TLS Infrastructure and Deployment Architecture

## Table of Contents

1. [Status](#status)
2. [Implementation](#implementation)
3. [Context](#context)
4. [Decision](#decision)
   - [Complete TLS Capability](#complete-tls-capability)
   - [Server Implementation](#server-implementation)
   - [Client Implementation with Certificate Discovery](#client-implementation-with-certificate-discovery)
5. [Deployment Scenarios and Certificate Management](#deployment-scenarios-and-certificate-management)
   - [Single Developer (Combined Mode)](#single-developer-combined-mode)
   - [Collaboration Development (Distributed)](#collaboration-development-distributed)
   - [SaaS Production (AWS/Cloud)](#saas-production-awscloud)
   - [Enterprise/Self-Hosted Production](#enterpriseself-hosted-production)
   - [Container/Kubernetes Deployment](#containerkubernetes-deployment)
6. [Health Endpoint Integration](#health-endpoint-integration)
   - [TLS Configuration Visibility](#tls-configuration-visibility)
   - [Certificate Monitoring](#certificate-monitoring)
   - [Operational Use Cases](#operational-use-cases)
7. [Rationale](#rationale)
   - [Design Principles](#design-principles)
   - [Integration with Existing Architecture](#integration-with-existing-architecture)
8. [Consequences](#consequences)
   - [Positive](#positive)
   - [Negative](#negative)
   - [Mitigations](#mitigations)
9. [Success Criteria](#success-criteria)
   - [Server-Side TLS](#server-side-tls)
   - [Client-Side TLS](#client-side-tls)
   - [Integration & Deployment](#integration--deployment)
10. [Implementation Dependencies](#implementation-dependencies)
11. [Alternatives Considered](#alternatives-considered)
12. [References](#references)
13. [Notes](#notes)

## Status

Accepted - Implemented

## Implementation

The TLS infrastructure is implemented as a unified module in the shared project with comprehensive test coverage:

**Architecture:**
- **Core Module**: `shared/src/main/clojure/ooloi/shared/grpc/tls.clj` - Unified TLS utilities for both server and client
- **Platform Support**: `shared/src/main/clojure/ooloi/shared/platform.clj` - Cross-platform directory management
- **Test Coverage**: `shared/test/clojure/ooloi/shared/grpc/tls_test.clj` - Comprehensive unit tests covering all TLS functionality
- **Integration Tests**: `shared/test/clojure/ooloi/shared/grpc/tls_integration_test.clj` - End-to-end component lifecycle testing

**Server-Side Implementation:**
- `ensure-certificates` - Complete certificate lifecycle management with automatic generation
- `generate-rsa-key-pair` - RSA 2048-bit key pair generation
- `create-self-signed-certificate` - X.509 certificates with SAN support
- `write-key-pair-to-files` - PEM format output with proper encoding
- Platform-appropriate certificate storage (Windows: `%APPDATA%/Ooloi/certs/`, Unix/macOS: `~/.ooloi/certs/`)
- Idempotent Bouncy Castle provider registration

**Client-Side Implementation:**
- `apply-tls-config` - Automatic certificate discovery and validation
- `configure-tls-for-channel` - gRPC channel TLS configuration with conditional authority override
- Three-tier trust strategy: insecure dev mode → explicit cert → system trust store
- Support for self-signed certificates (development), CA-signed certificates (production), and custom CA (enterprise)

**Test Infrastructure:**
- Unit tests for `ensure-certificates` covering generation, reuse, error handling, and platform paths
- Unit tests for `apply-tls-config` covering discovery logic, validation, and error scenarios
- Unit tests for health endpoint TLS visibility (disabled, enabled, expiry warnings, certificate parsing)
- Integration tests verifying behavior through component lifecycle including real TLS server with HTTP health endpoint
- Platform-specific path testing for Windows and Unix/macOS conventions

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
- **Generation**: Pure Java implementation using Bouncy Castle (idempotent provider registration)
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

The client implements one-way TLS (server authentication only) with three-tier trust strategy:

```clojure
;; Three-tier certificate trust strategy (configure-tls-for-channel)
(defn configure-tls-for-channel [builder config]
  (if (:tls config)
    (let [ssl-context-builder (GrpcSslContexts/forClient)
          ssl-context (cond
                        ;; Tier 1: Insecure development mode (explicit opt-in)
                        (:insecure-dev-mode config)
                        (do
                          (println "⚠️  WARNING: Running in insecure TLS mode")
                          (-> ssl-context-builder
                              (.trustManager InsecureTrustManagerFactory/INSTANCE)
                              (.build)))

                        ;; Tier 2: Explicit certificate (custom CA or enterprise)
                        (:cert-path config)
                        (-> ssl-context-builder
                            (.trustManager (io/file (:cert-path config)))
                            (.build))

                        ;; Tier 3: System trust store (production default)
                        :else
                        (.build ssl-context-builder))]  ; Uses Java's cacerts
      
      (cond-> builder
        ;; Always apply SSL context
        true (.sslContext ssl-context)
        
        ;; Override authority for self-signed certificate scenarios (tiers 1 & 2)
        ;; Tier 3 (system trust store) validates against actual target hostname
        (or (:insecure-dev-mode config) (:cert-path config))
        (.overrideAuthority (:backend-host config))))
    
    (.usePlaintext builder)))
```

**One-Way TLS Architecture:**
- **Server Authentication**: Client verifies server identity using server's public certificate
- **Client Identity**: Application-level authentication via client-id (not certificate-based)
- **Security Model**: Appropriate for collaboration software (not banking/military requiring mTLS)
- **Client Configuration**: Only needs server's public certificate, never private keys

**Three-Tier Trust Strategy:**

| Priority | Config | Trust Behavior | Authority Override | Use Case | Security Level |
|----------|--------|----------------|-------------------|----------|----------------|
| **1 (Highest)** | `:insecure-dev-mode true` | Bypass ALL validation | `:backend-host` | Self-signed certs in development | ⚠️ Insecure (explicit opt-in required) |
| **2** | `:cert-path "/path/to/cert"` | Trust specific certificate | `:backend-host` | Custom CA or enterprise PKI | ✅ Secure (validates certificate) |
| **3 (Default)** | No config | Use system trust store | None (validates actual hostname) | Production with Let's Encrypt, DigiCert, etc. | ✅ Secure (validates against 150+ trusted CAs) |

**Authority Override Behavior:**

The authority override controls which hostname the client checks against the server's certificate:

- **Tiers 1 & 2**: Override to `:backend-host` (the target you're connecting to)
- **Tier 3**: No override - validates against the actual target hostname

This design ensures:
- Development and enterprise scenarios work when the certificate matches the target host
- Production connections properly validate the server's CA-signed certificate

**Development Scenarios:**

| Scenario | Transport | TLS | insecure-dev-mode | cert-path | Behavior | When to Use |
|----------|-----------|-----|-------------------|-----------|----------|-------------|
| **Local dev (no TLS)** | `:network` | `false` | - | - | Plaintext | Default development, fastest |
| **Local dev (self-signed)** | `:network` | `true` | `true` | Auto-discovered | Insecure mode | Testing TLS with auto-generated certs |
| **Multi-client dev** | `:network` | `true` | `true` | Explicit path | Insecure mode | Testing collaboration with TLS |
| **Enterprise (custom CA)** | `:network` | `true` | `false` | CA cert path | Secure validation | Enterprise with internal PKI |
| **Production** | `:network` | `true` | `false` | None | System trust store | Production with CA-signed certs |
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

### Local Development (No TLS) - Default
**Architecture**: Backend and frontend on same machine, network transport
**Certificate Strategy**: No TLS - plaintext communication
**Use Case**: Default development workflow, fastest iteration

```bash
# Backend - no TLS
./ooloi-backend

# Frontend - no TLS
./ooloi-frontend
```

**Benefits**:
- **Fastest**: No TLS overhead
- **Simple**: No certificate management
- **Default**: Most developers work this way

### Local Development with TLS (Self-Signed Certificates)
**Architecture**: Backend and frontend on same or different machines, testing TLS functionality
**Certificate Strategy**: Server auto-generates self-signed certificate, client uses insecure mode
**Use Case**: Testing TLS features, multi-machine development, verifying encryption

```bash
# Backend - generates self-signed cert at ~/.ooloi/certs/server.crt
OOLOI_TLS=true ./ooloi-backend

# Frontend - uses insecure mode to accept self-signed cert
OOLOI_FRONTEND_TLS=true
OOLOI_FRONTEND_INSECURE_DEV_MODE=true
./ooloi-frontend
```

**Important Notes**:
- ⚠️ **Insecure mode required** for self-signed certificates (explicit opt-in)
- **Auto-discovery**: Client automatically finds server's certificate at `~/.ooloi/certs/`
- **Security**: Insecure mode bypasses ALL certificate validation - development only
- **When to use**: Testing TLS functionality, not production-like security

### Multi-Client Development (Distributed Testing)
**Architecture**: Multiple client machines connecting to shared development server
**Certificate Strategy**: Self-signed certificate shared across development team
**Use Case**: Testing collaboration features with TLS enabled

```bash
# Development server with auto-generated cert
OOLOI_TLS=true ./ooloi-backend

# Copy certificate to client machines:
# scp ~/.ooloi/certs/server.crt dev-machine-2:~/.ooloi/certs/

# Clients on different machines
OOLOI_FRONTEND_TLS=true
OOLOI_FRONTEND_INSECURE_DEV_MODE=true
OOLOI_FRONTEND_CERT_PATH=~/.ooloi/certs/server.crt
./ooloi-frontend
```

**Certificate Requirements**:
- **Distribution**: Share server's certificate with all client machines
- **Insecure mode**: Required for self-signed certificates
- **Testing focus**: Collaboration features, not production security model

### SaaS Production (AWS/Cloud)
**Architecture**: Backend on AWS behind Application Load Balancer  
**Certificate Strategy**: AWS Certificate Manager with TLS termination at load balancer  
**Client Experience**: Standard HTTPS with automatic certificate trust

```bash
# Backend configuration (behind ALB with TLS termination)
export OOLOI_TLS=false  # ALB handles TLS termination
export OOLOI_PORT=8080  # Internal HTTP port
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

**Enterprise Client with Custom CA (non-localhost hostname):**

```bash
# Client trusts enterprise CA certificate, authority derived from backend-host
OOLOI_FRONTEND_TLS=true \
OOLOI_FRONTEND_CERT_PATH=/etc/ssl/company-ca.crt \
OOLOI_FRONTEND_BACKEND_HOST=internal.corp.com \
./ooloi-frontend
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
# ============================================================================
# DEVELOPMENT SCENARIOS
# ============================================================================

# Default: No TLS (fastest, simplest)
./ooloi-backend
./ooloi-frontend

# Testing TLS with self-signed certificates (same machine)
OOLOI_TLS=true ./ooloi-backend
OOLOI_FRONTEND_TLS=true OOLOI_FRONTEND_INSECURE_DEV_MODE=true ./ooloi-frontend

# Testing TLS across multiple machines
OOLOI_TLS=true ./ooloi-backend  # Generates cert at ~/.ooloi/certs/
# Copy cert to other machines, then:
OOLOI_FRONTEND_TLS=true \
OOLOI_FRONTEND_INSECURE_DEV_MODE=true \
OOLOI_FRONTEND_CERT_PATH=~/.ooloi/certs/server.crt \
./ooloi-frontend

# Combined mode (in-process, no network)
./ooloi-combined  # TLS not applicable

# ============================================================================
# PRODUCTION SCENARIOS
# ============================================================================

# Production with Let's Encrypt (system trust store - no authority override)
OOLOI_TLS=true \
OOLOI_CERT_PATH=/etc/letsencrypt/live/api.example.com/fullchain.pem \
OOLOI_KEY_PATH=/etc/letsencrypt/live/api.example.com/privkey.pem \
./ooloi-backend

# Client connects with system trust (validates actual hostname)
OOLOI_FRONTEND_TLS=true \
OOLOI_FRONTEND_BACKEND_HOST=api.example.com \
./ooloi-frontend

# SaaS: TLS termination at load balancer
OOLOI_TLS=false OOLOI_PORT=8080 ./ooloi-backend  # Behind AWS ALB

# Enterprise with custom CA (non-localhost)
OOLOI_TLS=true \
OOLOI_CERT_PATH=/etc/ssl/company.crt \
OOLOI_KEY_PATH=/etc/ssl/company.key \
./ooloi-backend

# Enterprise client (trusts custom CA)
OOLOI_FRONTEND_TLS=true \
OOLOI_FRONTEND_CERT_PATH=/etc/ssl/company-ca.crt \
OOLOI_FRONTEND_BACKEND_HOST=internal.corp.com \
./ooloi-frontend
```

## Health Endpoint Integration

Enterprise operations require visibility into TLS configuration for monitoring, alerting, and compliance. Ooloi exposes TLS status and certificate details through standard HTTP health endpoints.

### TLS Configuration Visibility

The `/health` and `/health/server` endpoints include a `security` section with TLS configuration:

**TLS Disabled Response:**
```json
{
  "status": "SERVING",
  "timestamp_unix_seconds": 1727800000.0,
  "security": {
    "tls": {
      "enabled": false
    }
  },
  "server": {
    "uptime_seconds": 3600.5,
    "clients_connected_current": 5
  }
}
```

**TLS Enabled Response:**
```json
{
  "status": "SERVING",
  "timestamp_unix_seconds": 1727800000.0,
  "security": {
    "tls": {
      "enabled": true,
      "certificate": {
        "path": "/etc/ssl/ooloi.crt",
        "subject": "C=US,ST=Auto,L=Auto,O=Ooloi,OU=AutoGenerated,CN=localhost",
        "issuer": "C=US,ST=Auto,L=Auto,O=Ooloi,OU=AutoGenerated,CN=localhost",
        "serial": "17A2B3C4D5E6F",
        "valid_from": "2025-10-01T12:00:00Z",
        "valid_until": "2045-10-01T12:00:00Z",
        "days_until_expiry": 7299
      }
    }
  },
  "server": {
    "uptime_seconds": 3600.5,
    "clients_connected_current": 5
  }
}
```

**Implementation:**
- **Certificate Parsing**: `parse-certificate-info` in `shared/src/main/clojure/ooloi/shared/grpc/tls.clj` extracts X.509 details
- **Health Integration**: `build-tls-security-info` in `backend/src/main/clojure/ooloi/backend/grpc/stats.clj` builds security section
- **Graceful Degradation**: Parse failures return `{:tls {:enabled true}}` without certificate details

### Certificate Monitoring

**Expiry Warning System:**

When certificates expire within 30 days, a warning is automatically included:

```json
{
  "status": "SERVING",
  "security": {
    "tls": {
      "enabled": true,
      "warning": "Certificate expires in 15 days",
      "certificate": {
        "days_until_expiry": 15,
        "valid_until": "2025-10-16T12:00:00Z",
        ...
      }
    }
  }
}
```

**Certificate Fields:**
- `path`: Absolute path to certificate file
- `subject`: Certificate subject DN (Distinguished Name)
- `issuer`: Certificate issuer DN
- `serial`: Certificate serial number (hexadecimal)
- `valid_from`: Start of validity period (ISO 8601)
- `valid_until`: End of validity period (ISO 8601)
- `days_until_expiry`: Days remaining until expiration

**Configurable Validity:**

The `create-self-signed-certificate` function accepts an optional `validity-days` parameter:
```clojure
;; Default 20-year validity for auto-generated certificates
(create-self-signed-certificate key-pair)

;; Custom validity for testing expiry scenarios
(create-self-signed-certificate key-pair 30)  ; 30 days
```

### Operational Use Cases

**1. Monitoring and Alerting**

Prometheus-based certificate expiry monitoring:
```yaml
# Alert when certificate expires within 30 days
- alert: OoloiCertificateExpiringSoon
  expr: ooloi_tls_certificate_days_until_expiry < 30
  annotations:
    summary: "Certificate expires in {{ $value }} days"
    description: "Ooloi server certificate at {{ $labels.instance }} expires soon"
```

**2. Startup Verification**

Verify TLS configuration immediately after deployment:
```bash
# Check TLS is enabled in production
curl http://localhost:10701/health | jq .security.tls.enabled
# Expected: true

# Verify certificate details
curl http://localhost:10701/health | jq .security.tls.certificate
# Review subject, issuer, expiry date
```

**3. Compliance Auditing**

Security audits can verify TLS configuration remotely:
```bash
# Verify TLS enabled
# Verify certificate from approved CA
# Verify certificate not expired or expiring soon
curl http://ooloi-server:10701/health | \
  jq '{enabled: .security.tls.enabled,
       issuer: .security.tls.certificate.issuer,
       days_left: .security.tls.certificate.days_until_expiry}'
```

**4. Automated Certificate Rotation**

Integration with certificate management automation:
```bash
#!/bin/bash
# Certificate rotation trigger script
DAYS_LEFT=$(curl -s http://localhost:10701/health | \
            jq .security.tls.certificate.days_until_expiry)

if [ "$DAYS_LEFT" -lt 30 ]; then
  echo "Certificate expires in $DAYS_LEFT days - triggering rotation"
  trigger_cert_rotation
fi
```

**5. Load Balancer Health Checks**

Cloud load balancers can verify both service health and TLS status:
```yaml
# AWS ALB health check
# Target: /health
# Success: HTTP 200 + security.tls.enabled == true

# GCP Load Balancer health probe
# Path: /health
# Validation: Parse JSON, check security.tls.certificate.days_until_expiry > 7
```

**See Also:** [ADR-0025: Server Statistics Architecture](0025-Server-Statistics-Architecture.md) for complete health endpoint specification and data model details.

## Rationale

### Design Principles

1. **Developer Experience First**: TLS disabled by default, enabled with single environment variable
2. **Zero Setup Friction**: Auto-generated certificates eliminate manual certificate management
3. **Immediate Productivity**: Developers can start coding immediately without certificate setup
4. **Debug Capability**: TLS easily toggled on/off for testing and troubleshooting
5. **Production Ready**: Supports enterprise PKI, custom certificates, and CA-signed certificates
6. **Client Intelligence**: Automatic certificate discovery for development, system trust store for production
7. **Correct Authority Handling**: Authority override only for self-signed scenarios; production validates actual hostname

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
- **Correct hostname validation** in production (tier 3 validates actual target host)

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
- Idempotent Bouncy Castle provider registration
- All gRPC server tests passing with comprehensive certificate management coverage

### Client-Side TLS
- Complete TLS environment variable support: `OOLOI_FRONTEND_TLS`, `OOLOI_FRONTEND_CERT_PATH`
- Complete TLS CLI flag support: `--tls true/false`, `--cert-path`
- One-way TLS implementation (server authentication only, no client certificates)
- Client certificate discovery finds server certificates automatically (same-machine development)
- Client falls back to system trust store for CA-signed certificates (production)
- Explicit certificate configuration works for enterprise PKI scenarios
- Authority override only applied for tiers 1 & 2 (self-signed scenarios)
- Tier 3 (system trust store) validates against actual target hostname
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
- Bouncy Castle cryptography libraries for pure Java certificate generation (idempotent registration)
- Platform-specific file system APIs for certificate storage paths

## Alternatives Considered

### 1. TLS Always Enabled
**Rejected**: Would create friction for local development and combined-mode deployments where TLS adds no value

### 2. Third-Party Certificate Management Only
**Rejected**: Would complicate development workflows and require external dependencies for basic TLS functionality

### 3. Always Override Authority to Localhost
**Rejected**: Would break production deployments with CA-signed certificates for non-localhost hostnames. The three-tier strategy with conditional authority override provides correct behavior for all scenarios.

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
- **Correct authority handling**: Self-signed scenarios use target hostname; production validates actual hostname
- **Monolithic architecture**: No unnecessary complexity (ACME, mTLS, etc.) for features not needed in a monolithic server

The TLS implementation should be thoroughly tested across all deployment scenarios (development, production, enterprise, cloud) before considering it complete.