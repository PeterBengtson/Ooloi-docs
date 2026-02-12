# Ooloi Backend

**For initial development setup, see [../dev-setup/](../dev-setup/)**

The backend server component providing gRPC API services, STM-based piece management, and real-time client communication.

## Table of Contents

<img src="../img/backend-ooloi.png" alt="Ooloi Backend Architecture" align="right" width="500">

1. [Project Role](#project-role)
2. [System Architecture](#system-architecture)
3. [Directory Structure](#directory-structure)
4. [Prerequisites](#prerequisites)
5. [Installation](#installation)
6. [Building the Backend](#building-the-backend)
7. [Shared Model Architecture](#shared-model-architecture)
8. [Development](#development)
   - [Running the Backend](#running-the-backend)
   - [Configuration Options](#configuration-options)
   - [JVM Configuration](#jvm-configuration)
   - [TLS Configuration and Test Certificates](#tls-configuration-and-test-certificates)
   - [Monitoring and Health](#monitoring-and-health)
   - [Development Commands](#development-commands)
9. [Related Documentation](#related-documentation)

## Project Role

The backend server component providing:
- **gRPC API Server**: Unified interface serving frontend clients
- **STM-based Piece Management**: Concurrent musical data storage and operations  
- **Real-time Communication**: Event streaming and client coordination
- **Health Monitoring**: Component status and operational metrics

See [ADR-0023: Shared Model Contracts](../ADRs/0023-Shared-Model-Contracts.md) for architecture details.

## System Architecture

The backend is a sophisticated server application using **Integrant dependency injection** for component lifecycle management:

```
┌─────────────────┐    ┌──────────────────┐
│   Application   │    │   Environment    │
│      Core       │◄──►│   Configuration  │
└─────────────────┘    └──────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│              Piece Manager Component                │
│         (STM-based Musical Data Store)              │
└─────────────────────────────────────────────────────┘
    │                │                │
    ▼                ▼                ▼
┌──────────┐  ┌──────────┐  ┌──────────────┐
│  Cache   │  │  gRPC    │  │ HTTP Server  │
│  Daemon  │  │  Server  │  │  Component   │
└──────────┘  └──────────┘  └──────────────┘
    │              │              │
    ▼              ▼              ▼
┌──────────┐  ┌──────────┐  ┌──────────────┐
│Background│  │ Frontend │  │   Health     │
│Hash-Cons │  │  Client  │  │  Endpoints   │
│Optimize  │  │  Comms   │  │& Monitoring  │
└──────────┘  └──────────┘  └──────────────┘
```

**Component Dependencies:**
- **Application Core** → **Piece Manager** → **{Cache Daemon, gRPC Server}** → **HTTP Server**
- Cache Daemon, gRPC Server can start in parallel after Piece Manager
- HTTP Server depends on gRPC Server for health monitoring integration
- Configuration flows from CLI/environment through all components
- Clean shutdown ensures proper resource cleanup in reverse dependency order

### Core Components

#### Piece Manager Component
- **STM-based concurrent piece storage** with ACID transaction support
- **Piece lifecycle management** for concurrent access and modifications
- **Thread-safe operations** coordinating multiple client requests
- **Integration with shared models** for consistent data representation

#### gRPC Server Component
- **Unified API** serving ~193 methods via protocol buffers
- **TLS support** with automatic certificate generation
- **Health monitoring** with built-in gRPC health service

#### HTTP Server Component
- **Health monitoring endpoints** for load balancers and operations dashboards
- **Statistics access** over HTTP for external monitoring tools
- **Dependency on gRPC Server** for accessing health manager and port configuration
- **Port configuration** via `--health-port` (default 10701)

#### Cache Daemon Component
- **Hash-consing optimization** for memory efficiency through object canonicalization
- **Background maintenance** processes stored pieces to canonicalize duplicate musical objects
- **Reduces memory footprint** by ensuring identical Pitches, Rests, Chords, and Articulations share canonical instances
- **Configurable interval** with default 60-second maintenance cycles (`:maintenance-interval-ms`)
- **STM-safe operations** using proper transaction semantics for cache modifications
- **Automatic operation** requires no manual intervention once started

#### Application Core
- **CLI argument parsing** with comprehensive validation
- **Error handling** with specific exit codes for operational tooling
- **Lifecycle management** with JVM shutdown hooks

## Directory structure

```
backend/
├── docs/                            ; HTML documentation created by Codox
├── monitoring/                      ; Monitoring and statistics
├── resources/                       ; Application resources, icons
├── src/main/clojure/ooloi/backend/  ; Backend-specific source code
│   ├── api.clj                      ; Backend API functions (delegates to shared)
│   ├── core.clj                     ; Application entry point and main function
│   ├── system.clj                   ; Integrant system configuration
│   ├── components/                  ; Integrant system components
│   │   ├── piece_manager.clj        ; STM-based piece storage and management
│   │   ├── grpc_server.clj          ; gRPC server component
│   │   ├── http_server.clj          ; HTTP health/statistics server component
│   │   └── cache_daemon.clj         ; Hash-consing optimization daemon
│   ├── grpc/                        ; gRPC server implementation
│   │   ├── server.clj               ; Universal ExecuteMethod endpoint
│   │   └── stats.clj                ; Server statistics collection
│   └── ops/                         ; Backend-specific operations
│       └── piece_manager.clj        ; Piece storage and STM coordination
├── test/clojure/ooloi/backend/      ; Backend-specific tests
├── CHANGELOG.md
├── README.md
└── project.clj                      ; Dependencies: shared/, gRPC libraries
```

## Prerequisites

See [../dev-setup/](../dev-setup/) for complete system requirements and platform-specific installation instructions.

## Installation

**See [../dev-setup/](../dev-setup/) for complete installation instructions.**

Quick reference for developers who have already completed initial setup:

```bash
cd backend
lein deps         # Install dependencies
lein midje        # Verify installation (takes a few minutes)
```

**Important**: Backend requires shared/ to be built first (`lein protoc` in shared/ directory). See [../dev-setup/](../dev-setup/) for the correct build sequence.

## Building the Backend

1. Navigate to the backend directory:
   ```bash
   cd backend
   ```

2. Run tests:
   ```bash
   lein midje
   ```

3. Build the application:
   ```bash
   lein build
   ```

   This command will perform the following steps:
   - Clean the project
   - Create an uberjar
   - Run the build function to package the application

   You will see colored output indicating the progress of each step.

4. The packaged backend application will be in the `backend/target` directory.

## Build Process Details

The build process uses the following tools and steps:

1. **Cleaning**: Removes previous build artifacts.
2. **Uberjar Creation**: Compiles the code and creates a standalone jar file.
3. **Jpackage**: Packages the application for distribution, creating platform-specific installers or application images.

The build process handles both SNAPSHOT (development) versions and release versions:

- For SNAPSHOT versions, it creates an app image.
- For release versions, it creates platform-specific installers (DMG for macOS, DEB for Linux, MSI for Windows).

## Version Handling

The build process automatically adjusts version numbers for compatibility with different platforms:

- SNAPSHOT suffixes are removed for the final package version.
- If the version starts with "0", it's changed to "1.0.0" for macOS compatibility.
- The original version (including SNAPSHOT if applicable) is included in the application name.

## Shared Model Architecture

The backend implements shared model contracts, establishing clean separation between shared and backend-specific functionality.

### Shared Model Integration

See [ADR-0023: Shared Model Contracts](../ADRs/0023-Shared-Model-Contracts.md) for shared model architecture, namespace organization, and integration details.


## Development

### Running the Backend

#### Quick Start
```bash
lein run
```

The application will display startup messages:
```
Starting Ooloi backend server...
✅ All components started successfully
🎵 Ooloi backend ready for musical operations
```

#### Configuration Options

The backend supports flexible configuration through command-line arguments and environment variables:

**Command-Line Arguments** (recommended for development):
```bash
# Custom port
lein run -- --port 8080

# TLS configuration ([ADR-0020](../ADRs/0020-TLS-Infrastructure-and-Deployment-Architecture.md))
lein run -- --tls true
lein run -- --tls true --cert-path ./custom.crt --key-path ./custom.key

# Multiple options
lein run -- --port 8080 --timeout-ms 3000 --health-port 10701 --tls true
```

**Environment Variables** (recommended for production):
```bash
export OOLOI_PORT=8080
export OOLOI_TIMEOUT_MS=3000
export OOLOI_HEALTH_PORT=10701
# TLS configuration
export OOLOI_TLS=true
export OOLOI_CERT_PATH=/etc/ssl/ooloi.crt
export OOLOI_KEY_PATH=/etc/ssl/ooloi.key
lein run
```

#### Available Configuration

| Parameter | CLI Flag | Environment Variable | Default | Description |
|-----------|----------|---------------------|---------|-------------|
| **Port** | `--port 8080` | `OOLOI_PORT` | 10700 | gRPC server port for frontend communication |
| **Timeout** | `--timeout-ms 3000` | `OOLOI_TIMEOUT_MS` | 5000 | Network timeout in milliseconds |
| **TLS Enabled** | `--tls true/false` | `OOLOI_TLS` | false | Enable/disable TLS encryption |
| **TLS Certificate** | `--cert-path PATH` | `OOLOI_CERT_PATH` | platform default | Path to TLS certificate file (created if missing) |
| **TLS Private Key** | `--key-path PATH` | `OOLOI_KEY_PATH` | platform default | Path to TLS private key file (created if missing) |
| **Health Port** | `--health-port 10701` | `OOLOI_HEALTH_PORT` | 10701 | HTTP health endpoint port for monitoring |
| **Thread Pool Size** | `--thread-pool-size 4` | `OOLOI_THREAD_POOL_SIZE` | -1 (cores−1) | Shared thread pool size |

**Health Monitoring**:
- **Health Port**: HTTP endpoint for external monitoring tools (load balancers, ops dashboards)
- **gRPC Health**: Built-in gRPC health service for component coordination

**Configuration Precedence**: Command-line arguments override environment variables, which override defaults.

#### JVM Configuration

The backend uses sensible JVM defaults (G1GC, compact object headers, string deduplication, 50ms GC pause target) configured in `project.clj`. The JVM automatically sizes memory based on available system resources.

**Running the backend**:
```bash
# Development (uses project.clj defaults)
lein run

# Production (standalone JAR)
java -jar target/ooloi-backend-*-standalone.jar
```

**Advanced: Overriding JVM settings** (for specialized deployment scenarios):
```bash
# Development override
JVM_OPTS="-Xmx16g" lein run

# Production override
JAVA_OPTS="-Xmx24g" java -jar target/ooloi-backend-*-standalone.jar
```

#### TLS Configuration and Test Certificates

**TLS Overview** ([ADR-0020: TLS Infrastructure and Deployment Architecture](../ADRs/0020-TLS-Infrastructure-and-Deployment-Architecture.md)):
- **Default**: TLS disabled for immediate developer productivity
- **Development**: Auto-generates self-signed certificates for testing TLS functionality
- **Production**: Supports CA-signed certificates with secure validation
- **Enterprise**: Custom CA certificates with proper certificate chain validation

**Development Scenarios**:

**1. Local Development (No TLS) - Default and Recommended**
```bash
# Fastest development experience, no TLS overhead
lein run
# Or with environment variable
OOLOI_TLS=false lein run
```
Most developers work this way. Use this for normal development.

**2. Testing TLS Functionality (Self-Signed Certificates)**
```bash
# Server auto-generates certificate at ~/.ooloi/certs/server.crt
OOLOI_TLS=true lein run

# Note: Clients connecting to this server will need OOLOI_FRONTEND_INSECURE_DEV_MODE=true
# because self-signed certificates cannot be validated through a trust chain
```
Use this when testing TLS features or multi-machine setups.

**3. Custom Certificate Locations**
```bash
# Server uses certificates at specified paths
lein run -- --tls true --cert-path ./dev.crt --key-path ./dev.key
```

**Production Scenarios**:

**4. Production with Let's Encrypt (Recommended)**
```bash
# Production deployment with CA-signed certificate
export OOLOI_TLS=true
export OOLOI_PORT=443
export OOLOI_CERT_PATH=/etc/letsencrypt/live/api.example.com/fullchain.pem
export OOLOI_KEY_PATH=/etc/letsencrypt/live/api.example.com/privkey.pem
java -jar target/ooloi-backend-*-standalone.jar

# Clients connect with zero configuration - system trust store validates Let's Encrypt
```

**5. Enterprise with Custom CA**
```bash
# Enterprise deployment with company CA
export OOLOI_TLS=true
export OOLOI_CERT_PATH=/etc/ssl/company-server.crt
export OOLOI_KEY_PATH=/etc/ssl/company-server.key
java -jar target/ooloi-backend-*-standalone.jar

# Clients use OOLOI_FRONTEND_CERT_PATH=/etc/ssl/company-ca.crt for validation
```

**Certificate Auto-Generation**:
When TLS is enabled without explicit cert paths, Ooloi auto-generates self-signed certificates:
- **Implementation**: Pure Java with Bouncy Castle (cross-platform, no dependencies)
- **Default Locations**:
  - Unix/macOS: `~/.ooloi/certs/server.{crt,key}`
  - Windows: `%APPDATA%\Ooloi\certs\server.{crt,key}`
- **Properties**: RSA 2048-bit, 20-year validity, covers `localhost`, `127.0.0.1`, `::1`
- **Reuse**: Once generated, certificates persist across restarts
- **Purpose**: Development and testing only (not for production)

**Important Security Notes**:
- ⚠️ Self-signed certificates are for **development/testing only**
- ✅ Production deployments should use CA-signed certificates
- ✅ Let's Encrypt provides free, automated CA-signed certificates
- ✅ Enterprise deployments can use internal CA infrastructure

#### Production Deployment

For production deployment, use environment variables and proper process management:

```bash
# Set environment variables
export OOLOI_PORT=10700
export OOLOI_TIMEOUT_MS=5000

# Run with process management (e.g., systemd, supervisor, docker)
java -jar target/ooloi-backend-*-standalone.jar
```

#### Error Handling and Troubleshooting

The backend provides comprehensive error handling with actionable guidance:

**Port Conflicts**:
```bash
❌ Failed to start Ooloi backend:
Error: gRPC server failed to start on port 10700
💡 Suggestion: Is another instance already running? Try a different port or stop the conflicting service.
```

**Configuration Issues**:
```bash
❌ Failed to start Ooloi backend:
Error: Missing required configuration: piece-manager dependency
💡 Suggestion: Check your configuration files and environment variables.
```

**Exit Codes for Operational Monitoring**:
- **0**: Successful startup and operation
- **10**: Component initialization failure (check logs and dependencies)
- **11**: Port binding failure (port already in use)
- **12**: Configuration error (invalid or missing configuration)
- **13**: Missing dependency (required services unavailable)
- **14**: Resource exhaustion (insufficient system resources)
- **1**: Generic failure (check application logs)

#### Development vs Production

**Development Mode**:
- Use `lein run` with command-line arguments
- Automatic code reloading available
- Debug logging enabled by default

**Production Mode**:
- Use standalone JAR with environment variables
- Optimized performance settings
- Structured logging for monitoring

### Monitoring and Health

The backend provides comprehensive monitoring capabilities for production deployment:

**Component Health Status:**
- Each component reports its health status (healthy/unhealthy)
- System-wide health aggregates component statuses  
- Failed components can be isolated or restarted individually
- Built-in gRPC health service for component coordination

**Application Lifecycle:**
- **Startup**: Components initialize in dependency order
- **Runtime**: Health monitoring and error recovery
- **Shutdown**: Clean resource cleanup via JVM shutdown hooks

**Production Monitoring:**
- **Health Port**: HTTP endpoint for external monitoring tools (load balancers, ops dashboards)
- **gRPC Health**: Built-in gRPC health service for component coordination
- **Component Status**: Real-time health reporting for piece manager and gRPC server
- **Resource Monitoring**: Automatic failure detection and isolation

**Production Readiness:**
- Structured error messages with actionable guidance
- Specific exit codes for operational tooling integration
- Environment variable configuration for containerized deployments
- TLS support with automatic certificate generation

## Development Commands

### Running Tests

The backend uses Midje for comprehensive testing:

```bash
# Run all tests
lein midje

# Run specific test namespace
lein midje ooloi.backend.ops.piece-manager-test

# Run tests with coverage report
lein coverage
```

**Important**: Use `lein midje` instead of `lein test`. The project is configured for Midje testing framework.

### gRPC Integration

The backend implements Ooloi's unified gRPC server architecture:

```bash
# Compile application (includes automatic gRPC server generation)
lein compile

# Verify generated classes
ls target/generated-sources/protobuf/
```

#### gRPC Architecture

See [ADR-0024: gRPC Concurrency and Flow Control Architecture](../ADRs/0024-gRPC-Concurrency-and-Flow-Control-Architecture.md) for unified server design and implementation details.

#### When Compilation is Needed

**Automatic**: No manual steps needed for:
- New API methods in `api.clj`
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
# Add new API methods, modify data models, etc.
lein midje  # Test - no compilation cascades needed
```

### Building and Packaging

```bash
# Full build process (clean, compile, test, package)
lein build

# Create standalone JAR only
lein uberjar

# Clean build artifacts
lein clean
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
2. **Compile**: `lein compile` (compile application with gRPC server)
3. **Test**: `lein midje` (run test suite)
4. **Run**: `lein run` (start backend server)
5. **Iterate**: Make changes and repeat steps 2-4

### Integration Testing

The backend can be started as a separate process for integration testing with frontend clients:

```bash
# Start backend in development mode for testing
lein run -- --port 10700 &
BACKEND_PID=$!

# Run integration tests with frontend client
# ... test commands ...

# Clean shutdown
kill $BACKEND_PID
```

**Multi-Process Architecture:**
- Backend runs as independent server process
- Frontend connects via gRPC network transport
- Enables realistic integration testing scenarios
- Supports multi-user collaboration and distributed deployment

### Production Deployment

**Standalone JAR Deployment:**
```bash
# Build production JAR
lein uberjar

# Run production server (see Configuration section for environment setup)
java -jar target/ooloi-backend-*-standalone.jar
```

**Container Deployment:**
```dockerfile
# Example Dockerfile snippet
FROM openjdk:22-jre-slim
COPY target/ooloi-backend-*-standalone.jar /app/ooloi-backend.jar
ENV OOLOI_PORT=10700
EXPOSE 10700
CMD ["java", "-jar", "/app/ooloi-backend.jar"]
```

**Load Balancer Health Checks:**
```bash
# Health endpoint for load balancer monitoring
curl http://localhost:10701/health

# gRPC health check
grpc_health_probe -addr=localhost:10700
```

## Notes

- Ensure all tests pass before creating the final package.
- The packaged application is platform-specific and ready for distribution.
- JavaFX is included in the project dependencies, ensuring GUI capabilities.

Remember to run tests (`lein midje`) before packaging to ensure everything is working correctly.

## Related Documentation

### Architecture Guides
- **[Ooloi Server Architectural Guide](/guides/OOLOI_SERVER_ARCHITECTURAL_GUIDE.md)** - Complete server architecture analysis and enterprise patterns
- **[gRPC Communication and Flow Control](/guides/GRPC_COMMUNICATION_AND_FLOW_CONTROL.md)** - Communication patterns and collaborative scenarios
- **[Piece Manager Guide](/guides/PIECE_MANAGER_GUIDE.md)** - STM-based storage operations and lifecycle management
- **[Advanced Concurrency Patterns](/guides/ADVANCED_CONCURRENCY_PATTERNS.md)** - STM coordination patterns used in server implementation

### Technical Documentation
- **[ADR-0019: In-Process gRPC Transport Optimization](/ADRs/0019-In-Process-gRPC-Transport-Optimization.md)** - Transport performance optimization
- **[ADR-0024: gRPC Concurrency and Flow Control Architecture](/ADRs/0024-gRPC-Concurrency-and-Flow-Control-Architecture.md)** - Technical decisions behind communication patterns
