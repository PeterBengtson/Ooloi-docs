# ADR-0021: Pluggable Authentication Architecture and JWT-Based Security

## Status

Accepted

## Context

Ooloi's multi-deployment architecture supports vastly different operational scenarios, from local music teachers to enterprise conservatories and cloud-based SaaS deployments. Each scenario has fundamentally different authentication requirements:

- **Local Music Teacher**: Simple or no authentication, minimal setup friction
- **School District**: Integration with existing LDAP/Active Directory infrastructure
- **SaaS Production**: OAuth social logins, scalable user management
- **Enterprise Self-Hosted**: Corporate PKI, SAML SSO, compliance requirements
- **Workshop/Demo**: Anonymous sessions, temporary access

The authentication system must integrate seamlessly with:
- **gRPC transport** (ADR-0002): JWT tokens in request metadata for distributed deployments
- **TLS infrastructure** (ADR-0020): Coordinated security architecture
- **Component lifecycle** (ADR-0017): Integrant-managed authentication providers
- **Multiple deployment modes**: Combined, backend-only, and distributed architectures

Key architectural challenges:
- **Deployment flexibility** without forcing architectural compromises
- **Security-first design** while maintaining developer productivity
- **Enterprise readiness** without complexity for simple use cases
- **Protocol consistency** across local and distributed deployments

## Decision

We will implement a **pluggable authentication architecture** based on **JWT stateless tokens** and **protocol-driven providers**, managed through **Integrant components** with **configuration-driven selection**.

### Core Architectural Decisions

1. **JWT as Authentication Foundation**: Stateless tokens work identically in local and distributed deployments
2. **Pluggable Provider Architecture**: Clean protocols enable deployment-specific authentication without core logic changes
3. **Integrant Component Management**: Authentication providers follow established component lifecycle patterns
4. **gRPC Metadata Transport**: JWT tokens transmitted in gRPC request metadata for distributed authentication
5. **Configuration-Driven Selection**: Deployment scenarios determined by configuration data, not code changes

### Authentication Protocol Design

```clojure
(defprotocol AuthProvider
  "Core authentication operations for all provider types"
  (authenticate [provider username password] 
    "Authenticates user credentials, returns JWT token on success")
  (validate-token [provider jwt-token] 
    "Validates JWT token, returns user information or nil")
  (refresh-token [provider refresh-token] 
    "Refreshes JWT token using refresh token"))

(defprotocol UserStore
  "User data persistence abstraction"
  (create-user [store user-data] "Creates new user account")
  (find-user [store identifier] "Retrieves user by ID/username/email")
  (update-user [store user-id updates] "Updates user information")
  (list-organization-users [store org-id] "Lists users in organization")
  (delete-user [store user-id] "Removes user account"))
```

### Component Integration Architecture

```clojure
;; System configuration with dependency injection
{:ooloi.backend.components/piece-manager {}
 
 :ooloi.backend.components/auth-provider 
 {:type :auto  ; or :none, :cognito, :ldap, :sqlite
  :config {:allow-anonymous true
           :session-timeout "24h"
           :database-url "sqlite:~/.ooloi/users.db"}}
 
 :ooloi.backend.components/grpc-server 
 {:piece-manager (ig/ref :ooloi.backend.components/piece-manager)
  :auth-provider (ig/ref :ooloi.backend.components/auth-provider)
  :port 10700
  :tls-enabled true}}

;; Component lifecycle implementation
(defmethod ig/init-key :ooloi.backend.components/auth-provider 
  [_ {:keys [type config]}]
  (case type
    :none     (->NoAuthProvider config)
    :cognito  (->CognitoAuthProvider config)
    :ldap     (->LDAPAuthProvider config)
    :sqlite   (->SQLiteAuthProvider config)
    :auto     (select-provider-automatically config)))

(defmethod ig/halt-key! :ooloi.backend.components/auth-provider
  [_ provider]
  (when (satisfies? Lifecycle provider)
    (stop provider)))
```

### gRPC Authentication Integration

```clojure
;; Server-side authentication interceptor
(defn auth-interceptor [auth-provider]
  (proxy [ServerInterceptor] []
    (interceptCall [call headers next]
      (if-let [token (extract-jwt-from-headers headers)]
        (if-let [user (auth/validate-token auth-provider token)]
          (-> (.startCall next call headers)
              (add-user-context user))
          (throw-unauthenticated "Invalid or expired token"))
        (handle-anonymous-access auth-provider call headers next)))))

;; Client-side token injection
(defn create-authenticated-client [auth-provider host port user-credentials]
  (let [token (auth/authenticate auth-provider (:username user-credentials) (:password user-credentials))
        channel (create-secure-channel host port)
        stub (-> (OoloiGrpc/newBlockingStub channel)
                 (.withCallCredentials (jwt-call-credentials token)))]
    {:stub stub :token token :channel channel}))
```

## Rationale

### Why JWT-Based Architecture

1. **Stateless Design**: Tokens contain all necessary user information, eliminating server-side session storage
2. **gRPC Compatibility**: JWT tokens transport efficiently in gRPC metadata without protocol modifications
3. **Deployment Scalability**: Works identically from single-process to globally distributed deployments
4. **Standard Compliance**: Industry-standard approach with extensive tooling and library support
5. **Performance**: No database lookups for token validation in high-frequency musical operations

### Why Pluggable Providers

1. **Deployment Diversity**: Music teachers need different authentication than enterprise conservatories
2. **Integration Requirements**: Existing organizational infrastructure varies dramatically
3. **Operational Constraints**: Some environments prohibit external authentication services
4. **Cost Considerations**: Authentication complexity should match organizational resources
5. **Future Adaptability**: New authentication methods can be added without architectural changes

### Why Integrant Component Architecture

1. **Architectural Consistency**: Follows established patterns from ADR-0017 system architecture
2. **Dependency Management**: Clear dependency injection between authentication and gRPC components
3. **Lifecycle Management**: Proper resource initialization and cleanup for all provider types
4. **Configuration Integration**: Authentication configuration integrates with existing system configuration
5. **Testing Support**: Component isolation enables comprehensive testing strategies

### Integration with Existing Architecture

**gRPC Transport Synergy**: Authentication tokens leverage ADR-0002's gRPC metadata transport, providing seamless security without protocol modifications.

**TLS Infrastructure Coordination**: Authentication integrates with ADR-0020's TLS implementation, providing defense-in-depth security architecture.

**Component Lifecycle Harmony**: Authentication providers follow ADR-0017's component patterns, ensuring consistent resource management and dependency injection.

## Implementation Approach

### Provider Implementation Tiers

**Tier 1: Essential Providers (Immediate Implementation)**
- **NoAuthProvider**: Anonymous sessions with optional nicknames for local/educational use
- **LocalFileProvider**: Simple username/password storage in configuration files
- **InMemoryProvider**: Temporary sessions for demonstrations and workshops

**Tier 2: Professional Providers (Near-term Implementation)**
- **SQLiteProvider**: Single-file user database with encryption support
- **DatabaseProvider**: PostgreSQL/MySQL integration for traditional deployments

**Tier 3: Enterprise Providers (Long-term Implementation)**
- **CognitoProvider**: AWS Cognito integration with social login support
- **OAuthProvider**: Generic OAuth2 (Google, GitHub, Microsoft, custom providers)
- **LDAPProvider**: Corporate directory integration (Active Directory, OpenLDAP)
- **SAMLProvider**: Enterprise SSO integration for institutional customers

### Configuration-Driven Provider Selection

```clojure
;; Automatic provider selection based on environment
{:auth {:provider :auto
        :config {:allow-anonymous true
                 :session-timeout "24h"}}}

;; Explicit provider configuration
{:auth {:provider :cognito
        :config {:user-pool-id "us-east-1_AbCdEfGhI"
                 :client-id "1234567890abcdef"
                 :region "us-east-1"}}}

;; Environment variable overrides
;; OOLOI_AUTH_PROVIDER=ldap OOLOI_AUTH_LDAP_URL=ldaps://dc.company.com
```

### Deployment Scenario Matrix

| Scenario | Auth Provider | User Store | Configuration |
|----------|---------------|------------|---------------|
| Music Teacher | NoAuth/Simple | SQLite | `{:provider :none :allow-anonymous true}` |
| School District | LDAP | External | `{:provider :ldap :ldap-url "ldaps://..."}` |
| SaaS Production | Cognito | JWT Claims | `{:provider :cognito :user-pool-id "..."}` |
| Enterprise | SAML/OAuth | Corporate DB | `{:provider :saml :idp-url "..."}` |
| Workshop/Demo | NoAuth | Memory | `{:provider :none :user-store :memory}` |

### JWT Token Structure

```json
{
  "sub": "user-uuid",
  "email": "musician@conservatory.edu",
  "given_name": "Elena",
  "family_name": "Rodriguez", 
  "ooloi:display_name": "Elena R.",
  "ooloi:institution": "Royal Academy of Music",
  "ooloi:instruments": ["violin", "viola"],
  "ooloi:roles": ["editor", "collaborator"],
  "ooloi:permissions": {
    "administration": ["read"]
  },
  "exp": 1672531200,
  "iat": 1672444800
}
```

### Google Docs-Style Piece Authorization

**Email-Based Identity and Piece-Scoped Permissions**:

Authentication (this ADR) handles **who the user is**, while piece-scoped authorization handles **what they can do with each piece**. This follows the Google Docs model where:

- **User Identity**: Email address from JWT token (`"email": "musician@conservatory.edu"`)
- **Piece Permissions**: Stored within each piece's collaboration metadata
- **Invitation-Based Sharing**: Users invite others by email address
- **Role-Based Access**: admin/write/read/comment permissions per piece

```clojure
;; Piece authorization metadata (stored within each piece)
{:piece-id "symphony-1"
 :musical-data {...}
 :collaboration {:owner "user@example.com"
                :permissions {"user@example.com"     {:role :admin :granted-by :owner}
                             "violinist@music.edu"  {:role :write :granted-by "user@example.com"}
                             "conductor@orchestra"   {:role :read  :granted-by "user@example.com"}}
                :active-sessions {"client-123" {:user "violinist@music.edu" :connected-at timestamp}}}}
```

**Authorization Integration with Authentication**:
- **gRPC Method Interception**: Extract email from JWT token, check piece permissions before VPD operations
- **STM Transaction Boundaries**: Authorization checks within transaction scope for consistency
- **Event Streaming**: Only broadcast events to users with appropriate piece permissions

### Security Implementation

**Token Security**:
- Configurable access token lifetime (default: 2 hours for music notation workflows, 15 minutes for high-security environments)
- Longer refresh tokens (7 days) with automatic rotation
- Configurable token expiration policies per deployment
- Token revocation support through provider-specific blacklists or external revocation services

**Token Revocation Strategy**:
- **Immediate Revocation**: Provider-specific token blacklists for compromised tokens
- **User Logout**: Client-initiated token invalidation with server-side revocation
- **Permission Changes**: Automatic token invalidation when user roles/permissions change
- **Account Suspension**: All user tokens revoked when account is disabled
- **Integration Points**: External revocation services (Redis, database) for distributed deployments

**Multi-Factor Authentication (MFA)**:
- **Enterprise Provider Support**: MFA integration through SAML, OAuth, and Cognito providers
- **TOTP Support**: Time-based one-time passwords for SQLite and database providers
- **SMS/Email Verification**: Optional second factor for user registration and password reset
- **Hardware Token Support**: FIDO2/WebAuthn integration for high-security environments
- **Configurable MFA Policies**: Per-organization MFA requirements and exemptions

**Rate Limiting and Attack Protection**:
- **Authentication Rate Limiting**: Configurable limits per IP/user for login attempts
- **Brute Force Protection**: Exponential backoff and account lockout policies
- **Distributed Rate Limiting**: Redis-backed rate limiting for multi-instance deployments  
- **Monitoring Integration**: Authentication failure alerts and metrics
- **IP Whitelisting**: Optional IP-based access restrictions for enterprise deployments

**Transport Security**:
- All JWT tokens transmitted over TLS-encrypted connections (ADR-0020)
- gRPC metadata provides secure token transport channel
- No tokens stored in logs or debug output

**Provider Security**:
- Configurable password policies for local providers
- Integration with existing enterprise security policies
- Comprehensive audit trail support for compliance requirements (FERPA, GDPR, SOX)

**Session Management and User Experience**:
- **Intelligent Token Refresh**: Automatic token renewal during active music editing sessions
- **Activity-Based Expiration**: Extend token lifetime during intensive score editing work
- **Graceful Degradation**: Seamless token refresh without interrupting musical workflows
- **Background Refresh**: Client-side token renewal before expiration to prevent session interruption
- **Work Session Protection**: Save work progress before token expiration warnings

## Consequences

### Positive

- **Deployment Flexibility**: Single authentication architecture adapts from music teacher to enterprise conservatory
- **Security Consistency**: JWT tokens provide uniform security model across all deployment scenarios  
- **Developer Productivity**: Zero-config authentication for development, sophisticated options for production
- **Operational Integration**: Authentication providers integrate seamlessly with existing system architecture
- **Enterprise Readiness**: LDAP, SAML, and OAuth support enables institutional deployment
- **Cost Efficiency**: Authentication complexity scales with organizational resources and requirements
- **Future Adaptability**: New authentication methods integrate without architectural changes

### Negative

- **Configuration Complexity**: Multiple provider types with deployment-specific configuration options
- **Learning Curve**: Developers must understand JWT tokens, provider protocols, and Integrant lifecycle
- **Testing Requirements**: Authentication flows require comprehensive testing across provider types
- **Token Management**: JWT tokens require careful handling for security and user experience
- **Provider Dependencies**: Enterprise providers introduce dependencies on external services

### Provider Migration Strategies

**Organizational Growth Paths**:
- **Teacher → School**: NoAuth/SQLite → LDAP with user data export/import utilities
- **School → District**: LDAP → Centralized LDAP/SAML with identity federation
- **Self-Hosted → SaaS**: Local providers → Cognito/OAuth with account linking
- **SaaS → Enterprise**: OAuth → SAML/Enterprise SSO with SSO account mapping

**Migration Tools**:
- **User Data Export**: Standardized JSON format for user accounts, roles, and permissions
- **Account Linking**: Link existing local accounts with enterprise identities during migration
- **Gradual Migration**: Support multiple authentication providers during transition periods  
- **Rollback Support**: Ability to revert to previous authentication provider if migration issues occur

### Mitigations

- **Intelligent Defaults**: `:auto` provider selection chooses appropriate authentication for deployment context
- **Comprehensive Documentation**: Authentication patterns documented with concrete examples for each deployment scenario
- **Testing Framework**: Established patterns for testing authentication flows with mock providers
- **Configuration Validation**: Runtime validation catches authentication configuration errors with actionable messages
- **Graceful Degradation**: Authentication failures provide clear user guidance for resolution

### Testing Strategy

**Authentication Flow Testing**:
- **Mock Provider Framework**: Configurable mock providers for unit testing without external dependencies
- **Token Lifecycle Testing**: Comprehensive tests for token generation, validation, refresh, and revocation
- **Provider Integration Testing**: Automated tests against real providers (LDAP, OAuth) in CI/CD pipeline
- **Security Testing**: Automated security tests for common vulnerabilities (brute force, token manipulation)
- **Load Testing**: Authentication performance testing under realistic user loads

**Test Provider Categories**:
- **Unit Tests**: Mock providers with configurable responses for isolated component testing
- **Integration Tests**: Real provider connections with test accounts and controlled environments
- **Security Tests**: Penetration testing scenarios for authentication bypass attempts
- **Performance Tests**: High-concurrency authentication scenarios with realistic token validation loads
- **User Experience Tests**: Token refresh timing and workflow interruption scenarios

### Performance Considerations

**JWT Validation Optimization**:
- **Signature Caching**: Cache JWT signature validation results to reduce cryptographic overhead
- **Provider-Specific Optimizations**: LDAP connection pooling, database query optimization
- **Token Validation Pipeline**: Efficient JWT parsing and validation without full deserialization
- **Distributed Caching**: Redis-based token validation caching for multi-instance deployments

**Scalability Metrics**:
- **Authentication Throughput**: Target 1000+ authentications/second for enterprise deployments
- **Token Validation Latency**: <10ms for JWT validation in high-frequency musical operations
- **Provider Response Times**: Configurable timeouts and fallback strategies for external providers
- **Memory Usage**: Efficient token storage and cleanup to prevent memory leaks

### Compliance and Regulatory Support

**Education Compliance (FERPA)**:
- **Student Data Protection**: Separate authentication for student vs. staff accounts
- **Audit Trail Requirements**: Complete authentication and access logging
- **Data Retention Policies**: Configurable log retention periods
- **Parent/Guardian Consent**: Optional consent workflows for student account creation

**International Compliance (GDPR)**:
- **Data Minimization**: Collect only necessary authentication data
- **Right to Erasure**: Complete user data deletion capabilities
- **Data Portability**: Export user authentication data in standardized formats
- **Consent Management**: Explicit consent tracking for data processing

**Enterprise Compliance (SOX, ISO 27001)**:
- **Access Control Matrices**: Role-based access control with separation of duties
- **Privileged Account Management**: Enhanced security for administrative accounts
- **Regular Access Reviews**: Automated reports for access certification processes
- **Security Incident Response**: Integration with enterprise security monitoring systems

## Alternatives Considered

### 1. OAuth-Only Authentication
**Rejected**: Too complex for simple local deployments where music teachers just want to start using Ooloi immediately.

### 2. Simple Authentication Only
**Rejected**: Insufficient for enterprise deployments requiring LDAP integration, SAML SSO, and compliance features.

### 3. Session-Based Authentication
**Rejected**: Server-side session storage conflicts with stateless architecture and complicates distributed deployments.

### 4. Third-Party Authentication Service Only
**Rejected**: Creates external dependencies for basic functionality and increases operational complexity for simple deployments.

### 5. Custom Authentication Protocol
**Rejected**: JWT provides industry-standard security with extensive tooling; custom protocols require significant security expertise and testing.

## References

### Related ADRs
- [ADR-0000: Clojure](0000-Clojure.md) - Language foundation enabling protocol-based architecture
- [ADR-0001: Frontend-Backend Separation](0001-Frontend-Backend-Separation.md) - Deployment architecture requiring authentication
- [ADR-0002: gRPC Communication](0002-gRPC.md) - Transport protocol for JWT token metadata
- [ADR-0017: System Architecture](0017-System-Architecture.md) - Component architecture patterns for authentication providers
- [ADR-0020: TLS Infrastructure](0020-TLS-Infrastructure-and-Deployment-Architecture.md) - Transport security coordinating with authentication

### Technical Documentation
- [JWT (RFC 7519)](https://tools.ietf.org/html/rfc7519) - JSON Web Token standard
- [gRPC Authentication Guide](https://grpc.io/docs/guides/auth/) - gRPC metadata-based authentication
- [AWS Cognito Developer Guide](https://docs.aws.amazon.com/cognito/) - Cloud authentication integration
- [Integrant Documentation](https://github.com/weavejester/integrant) - Component lifecycle management

### Security Standards
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OAuth 2.0 Security Best Practices](https://tools.ietf.org/html/draft-ietf-oauth-security-topics)

## Notes

This authentication architecture maintains Ooloi's core philosophy of "minimal core, adaptive periphery" - simple when you need simple, sophisticated when you need sophisticated. The pluggable provider design ensures that a music teacher can start using Ooloi immediately while an enterprise conservatory can integrate with existing security infrastructure.

The JWT-based approach provides security consistency across all deployment scenarios while the Integrant component architecture ensures proper resource management and dependency injection. Authentication becomes just another managed component in the system, with the same reliability guarantees as piece management and gRPC communication.

Implementation should proceed incrementally: Tier 1 providers enable immediate development and simple deployments, while Tier 2 and Tier 3 providers add enterprise capabilities as needed. Each tier builds on previous capabilities without breaking existing functionality.

The architecture supports future authentication evolution - new providers can be added, existing providers can be enhanced, and authentication policies can be updated without affecting core musical functionality or deployment flexibility.