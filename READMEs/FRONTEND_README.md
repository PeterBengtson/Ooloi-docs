# Ooloi Frontend

**For initial development setup, see [../dev-setup/](../dev-setup/)**

This directory contains the frontend client code for Ooloi, a high-performance music notation software.

## Table of Contents

<img src="../img/frontend-ooloi.png" alt="Ooloi Frontend Architecture" align="right" width="500">

1. [Project Role](#project-role)
2. [System Architecture](#system-architecture)
3. [Directory Structure](#directory-structure)
4. [Prerequisites](#prerequisites)
5. [Installation](#installation)
6. [Configuration and Deployment](#configuration-and-deployment)
7. [Development](#development)
8. [Development Commands](#development-commands)
9. [Shared Model Integration](#shared-model-integration)
10. [Notes](#notes)
11. [Related Documentation](#related-documentation)

## Project Role

The **Ooloi Frontend** is the presentation layer code library providing:

1. **User Interface**: cljfx + JavaFX-based graphical interface with AtlantaFX dark theme for modern, futuristic music notation editing and visualization
2. **Score Rendering**: High-performance visual display of musical structures and notation
3. **gRPC Client Integration**: Communication with backend via protocol buffer-based unified interface
4. **User Interaction Management**: Click handling, form validation, real-time UI updates, and client-side state management

The frontend project does not produce a standalone deployable application. Instead, it is consumed by the **combined application** built from the [shared/](../shared/) project, which bundles frontend and backend code into a single JVM process using in-process gRPC transport. For standalone server deployment without a UI, see the [backend/](../backend/) project.

The frontend project retains `lein run` for development and testing purposes, along with full CLI argument and environment variable support.

## System Architecture

The frontend uses **Integrant dependency injection** for component lifecycle management. It defines five Integrant components:

- **event-bus**: Pure pub/sub message bus for internal frontend events
- **ui-manager**: User interface lifecycle — windows, dialogs, notifications, splash, theme
- **grpc-clients**: Connection to backend server (in-process transport in combined app)
- **event-router**: Routes backend events to the frontend event bus
- **fetch-coordinator**: Coordinates data fetches from the backend

The frontend project's own standalone configuration wires three of these (thread-pool, grpc-clients, ui-manager) with no declared dependencies between them — this is the test harness configuration. The full dependency graph, including event-bus, event-router, and fetch-coordinator, is assembled by the combined application in [shared/](../shared/).

## Directory structure

```
frontend/
├── docs/                            ; HTML documentation created by Codox
├── resources/                       ; Application resources, icons
├── src/main/clojure/ooloi/frontend/ ; Frontend source code
│   ├── app.clj                      ; Application entry point and main function
│   ├── app_settings.clj             ; App settings registry, persistence, event bus integration
│   ├── event_bus.clj                ; Pure pub/sub event bus (no UI concern)
│   ├── system.clj                   ; Integrant system configuration
│   ├── api/                         ; Frontend API
│   │   └── remote_api.clj           ; API functions delegating to backend
│   ├── components/                  ; Integrant system components
│   │   ├── event_bus.clj            ; Event bus component
│   │   ├── event_router.clj         ; Event router component
│   │   ├── fetch_coordinator.clj    ; Fetch coordinator component
│   │   ├── grpc_clients.clj         ; gRPC client component
│   │   └── ui_manager.clj           ; UI manager component
│   ├── event_router/                ; Event routing subsystem
│   │   ├── aggregator.clj           ; Event aggregation
│   │   └── core.clj                 ; Event router core
│   ├── grpc/                        ; gRPC client implementation
│   │   ├── api_client.clj           ; API client with connection pooling
│   │   └── event_client.clj         ; Event client for backend communication
│   ├── ops/                         ; Frontend operations
│   │   ├── atomic_handler.clj       ; Atomic operation handler
│   │   └── piece_ref.clj            ; Piece reference resolution
│   ├── rendering/                   ; Score rendering infrastructure
│   │   └── data_manager.clj         ; Rendering data management
│   ├── settings/                    ; App settings declarations (one file per tab)
│   │   ├── notifications.clj        ; Notifications settings
│   │   ├── ui.clj                   ; Appearance settings
│   │   └── user.clj                 ; User settings
│   └── ui/                          ; User interface (see ui/README.md)
│       ├── core/                    ; General reusable UI machinery
│       └── app/                     ; Ooloi-specific application UI
├── test/clojure/ooloi/frontend/     ; Frontend tests (mirror of src structure)
├── README.md
└── project.clj                      ; Dependencies: shared/ (complete Ooloi system)
```

## Prerequisites

See [../dev-setup/](../dev-setup/) for complete system requirements and platform-specific installation instructions.

## Installation

**See [../dev-setup/](../dev-setup/) for complete installation instructions.**

Quick reference for developers who have already completed initial setup:

```bash
cd frontend
lein deps         # Install dependencies
lein midje        # Verify installation
```

**Important**: Frontend requires shared/ to be built first (`lein protoc` in shared/ directory). See [../dev-setup/](../dev-setup/) for the correct build sequence.

## Configuration and Deployment

### Application Architecture

The Ooloi Frontend uses Integrant dependency injection for component lifecycle management. It provides the presentation layer consumed by the combined desktop application built from [shared/](../shared/).

**Key Components:**
- **event-bus**: Pure pub/sub message bus for internal frontend events
- **ui-manager**: User interface lifecycle — windows, dialogs, notifications, splash, theme
- **grpc-clients**: Connection to backend server (in-process transport in combined app)
- **event-router**: Routes backend events to the frontend event bus
- **fetch-coordinator**: Coordinates data fetches from the backend

### Command-Line Arguments

The frontend supports comprehensive command-line configuration:

```bash
# Basic usage
lein run

# With backend connection configuration
lein run -- --backend-host remote-server --backend-port 10700

# With UI options
lein run -- --ui-mode graphical

# With TLS security
lein run -- --tls true --cert-path /path/to/server-cert.pem

# Complete configuration example
lein run -- --backend-host localhost --backend-port 10700 --ui-mode graphical --timeout-ms 5000 --tls false
```

#### Available CLI Arguments

| **Argument** | **Values** | **Default** | **Description** |
|--------------|------------|-------------|-----------------|
| `--backend-host HOST` | hostname/IP | localhost | Backend server hostname or IP address |
| `--backend-port PORT` | 1-65535 | 10700 | Backend server port number |
| `--ui-mode MODE` | graphical, headless | graphical | User interface display mode |
| `--timeout-ms MS` | milliseconds | 5000 | Backend connection timeout |
| `--tls FLAG` | true, false | false | Enable TLS encryption for backend connection |
| `--cert-path PATH` | file path | auto-discovered | Path to server's public certificate (TLS only) |
| `--thread-pool-size N` | integer | -1 (cores−1) | Shared thread pool size |

**Argument Validation:**
- Port numbers are validated as integers within valid range
- UI mode is validated against allowed values
- TLS setting accepts only "true" or "false"
- Certificate path is optional when TLS enabled (auto-discovers from `~/.ooloi/certs/`)

### Environment Variables

All CLI arguments have corresponding environment variable alternatives:

| **Environment Variable** | **CLI Equivalent** | **Description** |
|-------------------------|-------------------|-----------------|
| `OOLOI_FRONTEND_BACKEND_HOST` | --backend-host | Backend server hostname |
| `OOLOI_FRONTEND_BACKEND_PORT` | --backend-port | Backend server port |
| `OOLOI_FRONTEND_UI_MODE` | --ui-mode | UI mode (graphical/headless) |
| `OOLOI_FRONTEND_TIMEOUT_MS` | --timeout-ms | Connection timeout in milliseconds |
| `OOLOI_FRONTEND_TLS` | --tls | Enable TLS (true/false) |
| `OOLOI_FRONTEND_CERT_PATH` | --cert-path | Path to server's public certificate |
| `OOLOI_THREAD_POOL_SIZE` | --thread-pool-size | Shared thread pool size |

**Configuration Precedence:** CLI arguments override environment variables, which override application defaults.

#### Environment Variable Usage Examples

```bash
# Basic development configuration
export OOLOI_FRONTEND_BACKEND_HOST=localhost
export OOLOI_FRONTEND_BACKEND_PORT=10700
export OOLOI_FRONTEND_UI_MODE=graphical
lein run

# Secure remote backend configuration
export OOLOI_FRONTEND_BACKEND_HOST=production-server.company.com
export OOLOI_FRONTEND_BACKEND_PORT=443
export OOLOI_FRONTEND_TLS=true
export OOLOI_FRONTEND_CERT_PATH=/etc/ssl/certs/ooloi-server.pem
lein run
```

### JVM Configuration

The frontend uses sensible JVM defaults (G1GC, compact object headers, string deduplication, 50ms GC pause target) configured in `project.clj`. The JVM automatically sizes memory based on available system resources.

**Advanced: Overriding JVM settings** (for specialized scenarios):
```bash
# Development override
JVM_OPTS="-Xmx16g" lein run

# Production: use the combined application from shared/
# See shared/README.md for production deployment
```

### Error Handling and Exit Codes

The frontend provides comprehensive error handling with specific exit codes for operational tooling:

| **Exit Code** | **Error Type** | **Description** |
|---------------|----------------|-----------------|
| 0 | Success | Application started successfully |
| 1 | Generic Failure | Unknown or unclassified error |
| 10 | Component Failure | Component initialization failed |
| 11 | Connection Failure | Cannot connect to backend server |
| 12 | Configuration Error | Invalid configuration provided |
| 13 | Dependency Missing | Required dependency unavailable |
| 14 | Resource Exhaustion | System resources exhausted |

**Error Resolution Guidance:**
The application provides actionable guidance for common errors:
- Connection failures: Check backend server status and network connectivity
- Configuration errors: Validate CLI arguments and environment variables
- Certificate errors: Verify TLS certificate and key file paths

### TLS and Security Configuration

**Client-Side TLS Support:**
The frontend supports secure connections to TLS-enabled backend servers using one-way TLS (server authentication).

#### Three-Tier Certificate Trust Strategy

The frontend uses a priority-based fallback system for validating server certificates:

1. **Priority 1: Insecure Development Mode** (`:insecure-dev-mode true`)
   - Bypasses ALL certificate validation (self-signed certificates)
   - **Use for**: Development with self-signed certificates
   - **Security**: ⚠️  NOT FOR PRODUCTION - accepts any certificate

2. **Priority 2: Explicit Certificate** (`:cert-path "/path/to/cert.pem"`)
   - Trusts specific certificate file with proper validation
   - **Use for**: Custom CA or enterprise certificates
   - **Security**: ✅ Secure - validates certificate properly

3. **Priority 3: System Trust Store** (default when TLS enabled)
   - Uses Java's built-in trusted CAs (~150 authorities)
   - **Use for**: Production with Let's Encrypt or commercial CA
   - **Security**: ✅ Fully secure - standard PKI validation

#### Common Scenarios

**1. Local Development (No TLS) - Default and Recommended**

Most developers work without TLS during local development:

```bash
# Default configuration - fastest development experience
lein run

# Or explicitly disable TLS
lein run -- --tls false
```

**No TLS overhead, direct plaintext communication.**

**2. Testing TLS with Self-Signed Certificates**

When testing multi-client setups or TLS functionality, use the backend's auto-generated self-signed certificate with insecure mode:

```bash
# Backend generates self-signed certificate in ~/.ooloi/certs/
# Frontend discovers it automatically and requires insecure mode

# Using CLI flag
lein run -- --tls true --insecure-dev-mode true

# Using environment variables
export OOLOI_FRONTEND_TLS=true
export OOLOI_FRONTEND_INSECURE_DEV_MODE=true
lein run
```

**Why insecure mode?** Self-signed certificates cannot be validated through a trust chain. They're fundamentally for development only.

**3. Custom Certificate Location**

If backend certificate is in non-default location:

```bash
# Explicit certificate path with insecure mode for self-signed
lein run -- --tls true --cert-path /custom/path/server.crt --insecure-dev-mode true

# Environment variables
export OOLOI_FRONTEND_TLS=true
export OOLOI_FRONTEND_CERT_PATH=/custom/path/server.crt
export OOLOI_FRONTEND_INSECURE_DEV_MODE=true
lein run
```

**4. Production with Let's Encrypt (Recommended)**

For production deployment with CA-signed certificates (Let's Encrypt, DigiCert, etc.):

```bash
# TLS enabled, system trust store validates automatically
lein run -- --tls true --backend-host production.example.com --backend-port 443

# Environment variables for containerized deployment
export OOLOI_FRONTEND_TLS=true
export OOLOI_FRONTEND_BACKEND_HOST=production.example.com
export OOLOI_FRONTEND_BACKEND_PORT=443
lein run
```

**No certificate path needed** - Java's built-in trust store (~150 CAs) validates the server's certificate automatically.

**5. Enterprise with Custom CA**

For enterprise environments with internal certificate authority:

```bash
# Explicit certificate path with proper validation (no insecure mode)
lein run -- --tls true --cert-path /etc/ssl/certs/company-ca.pem --backend-host enterprise.internal --backend-port 50051

# Environment variables
export OOLOI_FRONTEND_TLS=true
export OOLOI_FRONTEND_CERT_PATH=/etc/ssl/certs/company-ca.pem
export OOLOI_FRONTEND_BACKEND_HOST=enterprise.internal
export OOLOI_FRONTEND_BACKEND_PORT=50051
lein run
```

**Uses proper validation** - the certificate can be verified, so no insecure mode needed.

#### TLS Configuration Summary

| **Scenario** | **TLS Flag** | **Cert Path** | **Insecure Mode** | **Security** |
|--------------|--------------|---------------|-------------------|--------------|
| Local Development | `false` (default) | - | - | N/A (plaintext) |
| Testing TLS (Self-Signed) | `true` | auto-discovered | `true` | ⚠️  Development only |
| Custom Self-Signed Location | `true` | explicit path | `true` | ⚠️  Development only |
| Production (Let's Encrypt) | `true` | - | `false` (default) | ✅ Fully secure |
| Enterprise Custom CA | `true` | explicit path | `false` (default) | ✅ Fully secure |

#### Certificate Discovery

When `--cert-path` not specified with `--tls true`, the frontend automatically discovers certificates from `~/.ooloi/certs/`:

- Searches for `.crt` files in default directory
- Expects exactly one `.crt` file (throws exception if multiple found)
- Validates that file exists and is readable
- Fails cleanly with actionable error message if discovery fails

#### Security Notes

- **One-way TLS**: Clients verify server identity using server's public certificate
- **Server Certificate Only**: Clients only need the server's public certificate, never private keys
- **Insecure Mode Warning**: ⚠️  `--insecure-dev-mode true` prints warning and should NEVER be used in production
- **Certificate Validation**: Production deployments use full PKI validation through Java's trust store
- **Encryption**: All TLS modes encrypt gRPC communication between client and backend

### Integration Testing

The frontend can be started as a separate process for integration testing:

```bash
# Start frontend in background for testing
lein run -- &
FRONTEND_PID=$!

# Run integration tests
# ... test commands ...

# Clean shutdown
kill $FRONTEND_PID
```

### Monitoring and Health

The frontend provides health monitoring capabilities:

**Component Health Status:**
- Each component reports its health status (healthy/unhealthy)
- System-wide health aggregates component statuses  
- Failed components can be isolated or restarted individually

**Application Lifecycle:**
- **Startup**: Components initialize in dependency order
- **Runtime**: Health monitoring and error recovery
- **Shutdown**: Clean resource cleanup via JVM shutdown hooks

**Production Readiness:**
- Structured error messages with actionable guidance
- Specific exit codes for operational tooling integration
- Environment variable configuration for containerized deployments
- TLS support for secure client-server communication

## Development

### Running the Frontend

**Basic Development:**
```bash
lein run
```

**With Configuration:**
```bash
# Connect to local backend with debugging
lein run -- --backend-host localhost --backend-port 10700 --ui-mode graphical

# Remote backend connection
lein run -- --backend-host staging-server --backend-port 443 --tls true
```

### REPL

For interactive development, you can start a REPL:

```bash
lein repl
```

The REPL will start in the `ooloi.frontend.app` namespace.

## Development Commands

### Running Tests

The frontend uses Midje for comprehensive testing:

```bash
# Run all tests
lein midje

# Run specific test namespace
lein midje ooloi.frontend.components.some-test

# Run tests with coverage report
lein coverage
```

**Important**: Use `lein midje` instead of `lein test`. The project is configured for Midje testing framework.

### Translation Verification

The frontend uses PO files for localisation. During development, run `lein i18n` to keep translations synchronized with code:

```bash
# Development: auto-add missing translation keys with TODO placeholders
lein i18n

# Scan specific directory (e.g., test files)
lein i18n :source-dir "test/clojure"

# Strict mode (fails on missing keys or TODO entries)
lein i18n :strict true
```

**Parameters:**
- `:source-dir` — Directory to scan (default: `"src/main/clojure"`)
- `:po-file` — Translation catalog path (default: `"resources/i18n/en-GB.po"`)
- `:pattern` — File pattern to match (default: `#"\.clj$"`)
- `:strict` — Fail on incomplete translations (default: `false`)

**Development workflow:** Run `lein i18n` as you add new UI strings. Missing keys are automatically added with `[TODO: Translation needed]` placeholders.

**Build pipeline:** The combined application build (in shared/) runs verification in strict mode, failing if any keys are missing or contain TODO entries. This ensures all translations are complete before artifacts are created.

**See also:** [ADR-0039: Localisation Architecture](../ADRs/0039-Localisation-Architecture.md)

### gRPC Integration

**Essential Role**: Frontend is a **consumer of the unified Ooloi data model** located in shared/.

**⚡ CRITICAL**: Frontend uses **identical** data structures as backend:
- Same `Pitch` records, same `Piece` structures, same everything
- No separate frontend representations - just the unified data model
- Perfect fidelity: `(= frontend-pitch backend-pitch) => true`
- Works locally (direct shared calls) and remotely (gRPC)

The frontend uses Ooloi's unified gRPC architecture for remote backend communication:

```bash
# Compile application (includes automatic gRPC client generation)
lein compile

# Verify generated classes
ls target/generated-sources/protobuf/
```

#### Unified gRPC Architecture

The frontend uses Ooloi's **unified gRPC system** that eliminates complex protocol buffer management:

- **Unified Schema**: Single `OoloiValue` message handles all Clojure data types automatically
- **Dynamic API Access**: All backend methods available via `ExecuteMethod` - no per-method generation needed
- **Zero Maintenance**: New API methods and data types work immediately without regeneration
- **Perfect Type Fidelity**: Ratios, keywords, and custom records preserved across network boundaries

#### gRPC Client Configuration

- **Proto source**: `../shared/src/main/proto/ooloi_service.proto` (unified schema)
- **Generated output**: `target/generated-sources/protobuf/`  
- **Client stubs**: Universal `OoloiService` client with `ExecuteMethod`
- **Conversion**: Automatic Clojure ↔ `OoloiValue` conversion

#### When Compilation is Needed

**Automatic**: No manual steps needed for:
- New API methods in backend
- New data models or records
- Plugin installations  
- Shared model changes

**⚠️ Rare**: Recompilation only needed when `ooloi_service.proto` itself changes:
```bash
# Only if the unified schema is modified
lein clean && lein compile
```

**🎯 Normal Development**: Just code and test:
```bash
# Make any changes to shared models, backend API, etc.
lein midje  # Test - that's it!
```

### Documentation Generation

```bash
# Generate API documentation
lein codox
```
The documentation will be generated in the `docs` directory.

### Development Workflow

Recommended development sequence:

1. **Setup**: `lein deps` (install dependencies)
2. **Compile**: `lein compile` (compile application with gRPC client)
3. **Test**: `lein midje` (run test suite)
4. **Run**: `lein run` (start frontend for development/testing)
5. **Iterate**: Make changes and repeat steps 2-4

## Shared Model Integration

See [ADR-0023: Shared Model Contracts](../ADRs/0023-Shared-Model-Contracts.md) for frontend-specific shared model usage patterns, selective import architecture, and unified development workflow.

## Notes

- JavaFX is included in the project dependencies, ensuring GUI capabilities.
- The frontend project is consumed by the combined application built from [shared/](../shared/). It does not produce its own standalone artifact.
- Run `lein midje` to verify all frontend tests pass before committing changes.

## Related Documentation

### Architecture Guides  
- **[Polymorphic API Guide](/guides/POLYMORPHIC_API_GUIDE.md)** - Type system foundations that frontend development builds upon
- **[gRPC Communication and Flow Control](/guides/GRPC_COMMUNICATION_AND_FLOW_CONTROL.md)** - Client-server communication patterns for frontend integration
- **[Ooloi Server Architectural Guide](/guides/OOLOI_SERVER_ARCHITECTURAL_GUIDE.md)** - Understanding the server architecture that frontend connects to

### Technical Documentation
- **[ADR-0023: Shared Model Contracts](/ADRs/0023-Shared-Model-Contracts.md)** - Shared model architecture that frontend uses
- **[ADR-0022: Lazy Frontend-Backend Architecture](/ADRs/0022-Lazy-Frontend-Backend-Architecture.md)** - Frontend-backend interaction patterns
