# ADR-0020: TLS Infrastructure and Deployment Architecture

## Status

Accepted

## Context

[ADR-0002: gRPC Communication](0002-gRPC.md) establishes a comprehensive "TLS Everywhere Policy" with enterprise-grade certificate management, auto-generation tooling, and production-ready operational capabilities. However, this vision assumes infrastructure that doesn't currently exist in the codebase.

The current gRPC server implementation ([grpc_server.clj:67](../backend/src/main/clojure/ooloi/backend/components/grpc_server.clj)) creates insecure servers with no TLS support:

```clojure
(ServerBuilder/forPort port)  ; No TLS configuration
```

For production readiness, we need TLS capability immediately, but we also want to deliver ADR-0002's full enterprise vision systematically without compromising architectural integrity.

## Decision

We will implement TLS support through **two distinct phases**, each with specific deliverables and success criteria:

### Phase 1: Complete TLS Capability
**Goal**: Full TLS support for ANY deployment scenario - development, production, enterprise, cloud  
**Timeline**: Immediate (production readiness requirement)

**Deliverables:**
- Complete environment variable and CLI switch support for all TLS configuration
- TLS disabled by default for immediate developer productivity
- `OOLOI_TLS=true` or `--tls true` enables TLS with intelligent certificate management
- TLS-specific CLI overrides: `--tls true/false`, `--cert-path`, `--key-path`
- Auto-generated self-signed certificates for development and testing
- Custom certificate support for production and enterprise deployments
- Certificate creation at specified paths or platform-appropriate defaults
- TLS validation test in `system_integration_test.clj`
- Client TLS support for distributed deployments
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

**Implementation:**
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

## Deployment Scenarios and Certificate Management

### Single Developer (Combined Mode)
**Architecture**: Frontend and backend in same process  
**Certificate Strategy**: Auto-generated localhost certificates  
**Configuration**:
```bash
# TLS disabled by default for immediate productivity
./ooloi-combined

# TLS enabled with auto-generated certificates
OOLOI_TLS=true ./ooloi-combined
./ooloi-combined --tls true
```

**Certificate Characteristics**:
- **Generation**: Automatic on first TLS-enabled startup
- **Storage**: Via configuration (cert-path and key-path parameters)
- **SANs**: `localhost`, `127.0.0.1`, `::1`, `*.local`
- **Validity**: 20 years for long-term development
- **Trust**: Self-signed, requires manual trust or verification disable

**Certificate Management Behavior**:
1. **Explicit paths provided**: `--cert-path /path/cert.crt --key-path /path/key.key`
   - If certificates exist at those paths: Use existing certificates
   - If certificates don't exist: Create certificates at those paths
2. **No paths provided**: TLS enabled without certificate paths
   - Create certificates in platform-appropriate directory:
     - Unix/macOS: `~/.ooloi/certs/server.{crt,key}`
     - Windows: `%APPDATA%\Ooloi\certs\server.{crt,key}`
3. **Idempotent operation**: Safe to restart server multiple times, existing certificates reused

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
OOLOI_TLS=true OOLOI_CERT_PATH=/etc/certs/ooloi.crt OOLOI_KEY_PATH=/etc/certs/ooloi.key ./ooloi-server

# Production with commercial CA certificates
./ooloi-server --tls true --cert-path /etc/ssl/ooloi.crt --key-path /etc/ssl/ooloi.key --port 443
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
OOLOI_TLS=true OOLOI_CERT_PATH=/secrets/tls.crt OOLOI_KEY_PATH=/secrets/tls.key OOLOI_PORT=8443 ./ooloi-server
```

**Configuration Examples Summary**:
```bash
# Development: TLS disabled by default
./ooloi-server

# Development: TLS with auto-generated certificates
OOLOI_TLS=true ./ooloi-server
./ooloi-server --tls true

# Collaboration Development: Custom certificates for multi-client scenarios
./ooloi-server --tls true --cert-path ./dev-server.crt --key-path ./dev-server.key

# SaaS Production: TLS termination at load balancer
OOLOI_TLS=false OOLOI_PORT=8080 ./ooloi-server  # Behind AWS ALB

# Enterprise: Customer-managed certificates
OOLOI_TLS=true OOLOI_CERT_PATH=/etc/ssl/ooloi.crt OOLOI_KEY_PATH=/etc/ssl/ooloi.key ./ooloi-server

# Debugging: Explicit TLS disable
./ooloi-server --tls false
```

### Phase 2: Developer & Operations Tooling
**Goal**: Make Phase 1 easy to use and operationally robust  
**Timeline**: When tooling and automation become important for productivity and operations

**Deliverables:**
- Build tooling for certificate management (`make dev-certs`, `make prod-server`, `make debug-server`)
- Certificate validation and diagnostic tools
- Certificate expiration monitoring and health endpoint integration
- Automated certificate provisioning (ACME protocol support)
- Certificate rotation and renewal automation
- Enterprise monitoring integration (Prometheus metrics, alerting)
- Documentation and troubleshooting guides
- Team development certificate sharing (shared dev CA)

**Build Tooling:**
```makefile
dev-certs:
	# No external tools needed - certificates auto-generated using Java cryptography
	./ooloi-server --tls true --generate-certs

dev-server:
	./ooloi-server --tls true

debug-server:
	./ooloi-server

prod-server:
	./ooloi-server --tls true --port 443
```

**Automated Certificate Management:**
```bash
# ACME protocol support for automated provisioning
./ooloi-server --tls true --acme-provider letsencrypt --domain api.example.com
./ooloi-server --tls true --acme-provider internal-ca --domain internal.company.com
```

**Operational Features:**
- Health endpoint reports certificate expiration dates and validity
- Certificate validation warnings and diagnostics in server logs
- Prometheus metrics for certificate validity periods
- Automated certificate renewal with configurable thresholds
- Integration with enterprise PKI infrastructure
- mTLS (mutual TLS) support for service-to-service authentication

## Rationale

### Phased Approach Benefits

1. **Developer Experience First**: Phase 1 eliminates certificate setup friction while enabling TLS testing
2. **Immediate Productivity**: Developers can start working immediately without certificate management
3. **Progressive Enhancement**: Each phase adds capability without breaking previous functionality
4. **Production When Needed**: Phase 2 adds production deployment support when actually needed
5. **Risk Management**: Each phase can be validated independently before proceeding to the next

### Phase 1 Design Principles

**Developer Experience First**: TLS disabled by default, enabled with single environment variable
**Zero Setup Friction**: Auto-generated certificates eliminate manual certificate management
**Immediate Productivity**: Developers can start coding immediately without certificate setup
**Debug Capability**: TLS easily toggled on/off for testing and troubleshooting
**Progressive Enhancement**: TLS functionality builds from simple toggle to full production capability

### Integration with Existing Architecture

**Component Integration**: TLS configuration integrates cleanly with existing Integrant lifecycle management
**Deployment Model Support**: 
- **Combined mode**: In-process transport (no TLS needed)
- **Distributed mode**: Network transport with TLS requirement
- **Backend-only mode**: TLS-enabled server for external clients

**Configuration Precedence**: Environment variables → config files → defaults

## Consequences

### Positive

- **Developer productivity preserved** with zero-friction TLS toggle
- **Clear development roadmap** for enterprise features
- **No architectural debt** - each phase builds on previous phases
- **Maintains ADR-0002 vision** while enabling incremental delivery
- **Risk mitigation** through validated phase completion
- **Developer productivity** preserved through phased complexity introduction

### Negative

- **Multiple implementation phases** require sustained development effort
- **Coordination complexity** between phases and other feature development
- **Testing complexity** requiring validation across all phases
- **Documentation maintenance** for multiple configuration approaches

### Mitigations

- **Clear phase boundaries** with specific success criteria and deliverables
- **Comprehensive testing** for each phase before progression
- **Documentation strategy** that grows with phase implementation
- **Rollback capability** to previous phases if issues arise

## Success Criteria

### Phase 1 Success Criteria (COMPLETE)
- ✅ gRPC server starts immediately with TLS disabled by default
- ✅ Complete TLS environment variable support: `OOLOI_TLS`, `OOLOI_CERT_PATH`, `OOLOI_KEY_PATH`
- ✅ Complete TLS CLI flag support: `--tls true/false`, `--cert-path`, `--key-path`
- ✅ Auto-generated certificates work without manual intervention when no custom certs provided (Pure Java implementation)
- ✅ Custom certificate paths work for production and enterprise deployments
- ✅ Works transparently across development, testing, production, enterprise, and cloud scenarios
- ⏳ TLS validation test passes in `system_integration_test.clj` (final task for production readiness)
- ✅ Certificate generation uses pure Java cryptography (cross-platform, no external dependencies)
- ✅ All 70 gRPC server tests passing with comprehensive certificate management coverage including reuse behavior

### Phase 2 Success Criteria
- ✅ Build tooling (`make dev-certs`, `make debug-server`, `make prod-server`) functions correctly
- ✅ Certificate validation and diagnostic tools provide clear error messages
- ✅ Health endpoint reports certificate expiration dates and validity
- ✅ ACME protocol integration works with multiple providers (Let's Encrypt, internal CAs)
- ✅ Certificate renewal automation prevents expiration-related outages
- ✅ Enterprise monitoring integration provides operational visibility
- ✅ mTLS support enables secure service-to-service communication

## Implementation Dependencies

### Phase 1 Dependencies
- Current gRPC server component (`grpc_server.clj`)
- Existing Integrant configuration system
- Java gRPC TLS APIs (`ServerBuilder.useTransportSecurity`)
- Bouncy Castle cryptography libraries for pure Java certificate generation

### Phase 2 Dependencies
- Phase 1 completion
- Build system integration (Make or equivalent)
- ACME protocol client libraries (for Let's Encrypt, internal CAs, etc.)
- Enterprise monitoring infrastructure integration
- Production automation tooling

## Alternatives Considered

### 1. Implement Full ADR-0002 Vision Immediately
**Rejected**: Would delay production readiness milestone significantly while building comprehensive certificate infrastructure

### 2. Simple TLS Only (No Enterprise Features)
**Rejected**: Would leave gap between basic implementation and enterprise deployment requirements, potentially creating technical debt

### 3. Revise ADR-0002 to Remove Enterprise Features
**Rejected**: Enterprise features are legitimate requirements for production deployments; removing them would compromise long-term architecture

### 4. Third-Party Certificate Management Integration Only
**Rejected**: Would create external dependencies for basic TLS functionality and complicate development workflows

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

This phased approach prioritizes developer experience while building toward enterprise requirements. Phase 1 provides frictionless TLS capability that developers can use immediately, while Phase 2 adds production deployment support when needed.

The implementation phases can be adjusted based on deployment requirements and resource availability. Phase 1 completion enables both development TLS testing and production readiness through auto-generated certificate capability.

Each phase should be thoroughly tested and documented before progression to ensure stability and maintainability of the TLS implementation across all deployment scenarios.