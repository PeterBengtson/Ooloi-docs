# Ooloi Backend

This directory contains the backend server code for Ooloi, a high-performance music notation software.

## Table of Contents

<img src="../img/backend-ooloi.png" alt="Ooloi Backend Architecture" align="right" width="500">

1. [Project Role](#project-role)
2. [System Architecture](#system-architecture)
   - [Core Components](#core-components)
     - [Piece Manager Component](#piece-manager-component)
     - [gRPC Server Component](#grpc-server-component)
     - [Application Core](#application-core)
3. [Directory Structure](#directory-structure)
4. [Prerequisites](#prerequisites)
   - [System Requirements](#system-requirements)
   - [Platform-Specific Installation](#platform-specific-installation)
   - [Verification](#verification)
   - [Icon Files Setup](#icon-files-setup)
5. [Installation](#installation)
6. [Building the Backend](#building-the-backend)
7. [Build Process Details](#build-process-details)
8. [Version Handling](#version-handling)
9. [Shared Model Architecture](#shared-model-architecture)
   - [Shared Model Integration](#shared-model-integration)
   - [Backend-Specific Enhancements](#backend-specific-enhancements)
   - [Namespace Organization](#namespace-organization)
   - [Testing Architecture](#testing-architecture)
10. [Development](#development)
    - [Running the Backend](#running-the-backend)
    - [Monitoring and Health](#monitoring-and-health)
11. [Development Commands](#development-commands)
    - [Running Tests](#running-tests)
    - [gRPC Integration](#grpc-integration)
    - [Building and Packaging](#building-and-packaging)
    - [Documentation Generation](#documentation-generation)
    - [Development Workflow](#development-workflow)
    - [Integration Testing](#integration-testing)
    - [Production Deployment](#production-deployment)
12. [Notes](#notes)

## Project Role

The **Ooloi Backend** serves as the core server component providing:

1. **Musical Data Management**: STM-based concurrent piece storage and manipulation
2. **Complex Musical Operations**: Timewalk traversal, attachment resolution, and algorithmic processing
3. **gRPC Server Interface**: Unified API serving ~193 methods to frontend clients via protocol buffers
4. **Component Architecture**: Integrant-based system with piece manager, gRPC server, and monitoring components

**Key Architectural Responsibility**: Implements backend-specific multimethod behaviors for shared model contracts while providing a unified gRPC interface for frontend communication.

## System Architecture

The backend is a sophisticated server application using **Integrant dependency injection** for component lifecycle management:

```
┌─────────────────┐    ┌──────────────────┐
│   Application   │    │   Environment    │
│      Core       │◄──►│   Configuration  │
└─────────────────┘    └──────────────────┘
         │
         ▼
┌─────────────────┐    ┌──────────────────┐
│  Piece Manager  │◄──►│   gRPC Server    │
│   Component     │    │   Component      │
└─────────────────┘    └──────────────────┘
         │                       │
         ▼                       ▼
┌─────────────────┐    ┌──────────────────┐
│ STM Transaction │    │ Frontend Client  │
│ Musical Data    │    │ Communication    │
└─────────────────┘    └──────────────────┘
```

**Component Dependencies:**
- **Application Core** → **Piece Manager** → **gRPC Server**
- Configuration flows from CLI/environment through all components
- Clean shutdown ensures proper resource cleanup in reverse dependency order

### Core Components

#### Piece Manager Component
- **STM-based concurrent piece storage** with ACID transaction support
- **Musical data management** for complex scores and arrangements
- **VPD addressing system** for precise musical element navigation
- **Attachment system** supporting ties, slurs, dynamics, articulations

#### gRPC Server Component  
- **Unified API** serving ~193 methods via protocol buffers
- **Transport optimization** with in-process and network modes
- **TLS support** with automatic certificate generation
- **Health monitoring** with built-in gRPC health service

#### Application Core
- **CLI argument parsing** with comprehensive validation
- **Configuration management** supporting multiple deployment scenarios
- **Error handling** with specific exit codes for operational tooling
- **Lifecycle management** with JVM shutdown hooks

## Directory structure

```
backend/
├── docs/                            ; HTML documentation created by Codox
├── resources/                       ; Application resources, icons
├── src/main/clojure/ooloi/backend/  ; Backend-specific source code
│   ├── api.clj                      ; Backend API functions (delegates to shared)
│   ├── core.clj                     ; Application entry point and main function
│   ├── system.clj                   ; Integrant system configuration
│   ├── components/                  ; Integrant system components
│   │   ├── piece_manager.clj        ; STM-based piece storage and management
│   │   └── grpc_server.clj          ; gRPC server component
│   ├── grpc/                        ; gRPC server implementation
│   │   ├── server.clj               ; Universal ExecuteMethod endpoint
│   │   └── conversion.clj           ; OoloiValue conversion utilities
│   └── ops/                         ; Backend-specific operations
│       └── piece_manager.clj        ; Piece storage and STM coordination
├── test/clojure/ooloi/backend/      ; Backend-specific tests (~1,869 tests)
├── CHANGELOG.md
├── README.md
└── project.clj                      ; Dependencies: shared/, gRPC libraries
```

## Prerequisites

### System Requirements

- **Java Development Kit (JDK) 22 or later** - Required for running Clojure and building the application
- **Leiningen 2.9.0 or later** - Clojure build tool for dependency management and project operations
- **Git** - For version control and accessing the repository
- **Minimum 4GB RAM** - Recommended for development and large musical scores
- **Network access** - For downloading dependencies during initial setup

### Platform-Specific Installation

#### macOS
```bash
# Install Java using Homebrew
brew install openjdk@22

# Install Leiningen
brew install leiningen

# Set environment variables (add to ~/.zshrc or ~/.bash_profile)
export JAVA_HOME="/usr/local/opt/openjdk@22"
export PATH="$JAVA_HOME/bin:$PATH"
```

#### Linux (Ubuntu/Debian)
```bash
# Install Java
sudo apt update
sudo apt install openjdk-22-jdk

# Install Leiningen
curl https://raw.githubusercontent.com/technomancy/leiningen/stable/bin/lein > ~/bin/lein
chmod +x ~/bin/lein
lein --version  # This will download and install Leiningen

# Set environment variables (add to ~/.bashrc)
export JAVA_HOME="/usr/lib/jvm/java-22-openjdk-amd64"
export PATH="$JAVA_HOME/bin:$PATH"
```

#### Windows
1. **Install Java**:
   - Download OpenJDK 22 from [Adoptium](https://adoptium.net/)
   - Run installer and follow prompts
   - Set `JAVA_HOME` environment variable to installation directory

2. **Install Leiningen**:
   - Download `lein.bat` from [Leiningen website](https://leiningen.org/)
   - Place in a directory on your PATH
   - Run `lein self-install` from command prompt

### Verification

Verify your installation:
```bash
# Check Java version
java -version
# Should show: openjdk version "22.x.x" or later

# Check Leiningen
lein version
# Should show: Leiningen 2.9.0 or later on Java 22.x.x

# Check environment
echo $JAVA_HOME
# Should show path to Java installation
```

### Icon Files Setup

Ensure you have the appropriate icon files in the `backend/resources/icons/` directory:

- **macOS**: `icon.icns`
- **Windows**: `icon.ico` 
- **Linux**: `icon.png`

These can be found in the root `icons/` directory and should be symlinked:
```bash
cd backend/resources/
ln -s ../../icons/ready/macos/icon.icns icons/
ln -s ../../icons/ready/windows/icon.ico icons/
ln -s ../../icons/ready/linux/icon.png icons/
```

## Installation

  ```bash
  cd backend
  lein deps
  ```

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

**Completed Shared Model Contracts**: The backend uses shared model contracts from `../shared/src/main/clojure/` (ADR-0023 completed implementation):

- **Core Data Models**: All `defrecord` structures (Piece, Musician, Instrument, etc.) defined in shared
- **Interfaces & Predicates**: Shared multimethod contracts and type checking predicates  
- **Basic Operations**: Fundamental ops utilities in shared (access, pitches, rhythm, text)
- **All Traits**: Behavioral mixins (attachment, has-items, rhythmic-item, takes-attachment, transposable) in shared
- **Generator System**: Test data generators accessible from `ooloi.shared.specs.generators`

**Architecture Benefits**:
- ✅ **Type Fidelity**: Frontend and backend use identical data models for perfect gRPC round-trips
- ✅ **Interface Consistency**: Shared multimethod contracts prevent API drift  
- ✅ **Unified Plugin Ecosystem**: Plugins define models once, work everywhere
- ✅ **Zero Protocol Buffer Overhead**: Shared models work seamlessly with unified gRPC system

### Backend-Specific Enhancements

**VPD-Enhanced Operations**: Backend extends shared models with Vector Path Descriptor capabilities:

```clojure
;; Backend uses the same shared models but with additional server context
(ooloi.shared.models.core/create-pitch :note "C4" :duration 1/4)
;; → Backend provides gRPC server integration, STM transactions, piece management around shared models
```

**Enhanced Model Features**: Backend provides:
- **Server Integration**: gRPC service implementation and STM coordination
- **Piece Management**: Centralized piece storage and concurrent access control  
- **VPD Addressing**: Precise navigation within complex musical hierarchies
- **Advanced Algorithms**: Music theory, layout, and rendering computations

### Namespace Organization

**For 90% of your backend code, you only need this:**

```clojure
(ns my-namespace
  (:require [ooloi.shared.models.core :refer :all]))
```

This gives you access to:
- **All constructors**: `create-pitch`, `create-chord`, `create-measure`, etc. (from shared project)
- **All predicates**: `pitch?`, `chord?`, `measure?` (raw items), plus `pitch??`, `chord??`, `measure??` (timewalk tuples), etc. (from shared project)
- **All multimethods**: `get-duration`, `add-item`, `set-name`, etc. (interfaces from shared, implementations from backend)

**Why this works**: The shared `core` namespace imports shared model contracts and provides a unified entry point for backend development.

**Architecture Note**: Core data structures live in the `shared/` project. The shared `core` namespace serves as the complete Ooloi system that both frontend and backend consume.

**Shared Operations** (now in shared project):
```clojure
[ooloi.shared.ops.access :as xs]        ; Vector/attribute operations (was vectors-and-attributes)
[ooloi.shared.ops.pitches :as pitches]  ; Pitch normalization and conversion
[ooloi.shared.ops.rhythm :as rhythm]    ; Duration and rhythm utilities  
[ooloi.shared.ops.text :as text]        ; Text processing (pluralization, etc.)
```

**Backend-Specific Operations**:
```clojure
[ooloi.shared.ops.timewalk :as tw]      ; Musical structure traversal
[ooloi.backend.components.* :as *]      ; Integrant system components
```

**Shared Operations** (available to both frontend and backend):
```clojure
[ooloi.shared.models.changes :as ch]    ; Change-based attributes (time sigs, etc.) - now in shared
```

### Testing Architecture

**Test Data Generation**: Backend uses shared generators:
```clojure
[ooloi.shared.specs.generators :as generators]  ; All model generators

;; Available generators
(generators/create-random-piece)      ; Complete piece with all sub-models
(generators/create-random-musician)   ; Musician with instruments  
(generators/gen-pitch)                ; Pitch generator for property testing
```

**Backend Test Coverage includes**:
- **Shared Model Integration**: Backend-specific implementations of shared contracts
- **Complex Musical Operations**: Timewalk traversal, attachment resolution, algorithmic processing
- **gRPC Server Implementation**: Unified ExecuteMethod interface with dynamic API resolution
- **Component Lifecycle**: Integrant system startup/shutdown scenarios and piece manager operations  
- **STM Transactions**: Thread-safe concurrent piece modification and batch operations
- **API Integration**: All ~193 backend API methods accessible via gRPC unified interface


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
🎵 Ooloi backend ready for musical collaboration
```

#### Configuration Options

The backend supports flexible configuration through command-line arguments and environment variables:

**Command-Line Arguments** (recommended for development):
```bash
# Custom port
lein run -- --port 8080

# Deployment mode
lein run -- --deployment-mode combined

# TLS configuration (ADR-0020)
lein run -- --tls true
lein run -- --tls true --cert-path ./custom.crt --key-path ./custom.key

# gRPC transport optimization (ADR-0019)
lein run -- --grpc-transport in-process --health-port 10701

# Multiple options
lein run -- --port 8080 --deployment-mode backend --timeout-ms 3000 --grpc-transport network --tls true
```

**Environment Variables** (recommended for production):
```bash
export OOLOI_PORT=8080
export OOLOI_DEPLOYMENT_MODE=combined  
export OOLOI_TIMEOUT_MS=3000
export OOLOI_GRPC_TRANSPORT=in-process
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
| **Deployment Mode** | `--deployment-mode MODE` | `OOLOI_DEPLOYMENT_MODE` | backend | System deployment configuration |
| **Timeout** | `--timeout-ms 3000` | `OOLOI_TIMEOUT_MS` | 5000 | Network timeout in milliseconds |
| **TLS Enabled** | `--tls true/false` | `OOLOI_TLS` | false | Enable/disable TLS encryption |
| **TLS Certificate** | `--cert-path PATH` | `OOLOI_CERT_PATH` | platform default | Path to TLS certificate file (created if missing) |
| **TLS Private Key** | `--key-path PATH` | `OOLOI_KEY_PATH` | platform default | Path to TLS private key file (created if missing) |
| **gRPC Transport** | `--grpc-transport TYPE` | `OOLOI_GRPC_TRANSPORT` | auto | Transport optimization: `network` or `in-process` |
| **Health Port** | `--health-port 10701` | `OOLOI_HEALTH_PORT` | 10701 | HTTP health endpoint port for monitoring |

**Deployment Modes**:
- **`backend`** (default): Piece manager + gRPC server - typical server deployment
- **`combined`**: All components including UI support - single-process deployment  
- **`dev-engine-only`**: Only piece manager - minimal development/testing mode

**gRPC Transport Optimization** (ADR-0019):
- **`auto`** (default): Automatic selection - `in-process` for combined mode, `network` for backend mode
- **`in-process`**: Ultra-high-performance direct communication (37.5-75x faster, 98.7-99.3% latency reduction, for combined deployments)
- **`network`**: Standard TCP communication (for debugging or separate process deployment)

**Health Monitoring**:
- **Health Port**: HTTP endpoint for external monitoring tools (load balancers, ops dashboards)
- **gRPC Health**: Built-in gRPC health service for component coordination

**Configuration Precedence**: Command-line arguments override environment variables, which override defaults.

#### TLS Configuration and Test Certificates

**TLS Overview** (ADR-0020: TLS Infrastructure and Deployment Architecture):
- **Default**: TLS disabled for immediate developer productivity
- **Development**: Enable with `--tls true` or `OOLOI_TLS=true` - certificates auto-generated
- **Certificate Management**: Intelligent creation at specified paths or platform defaults
- **Production**: Full control via `--cert-path` and `--key-path` parameters

**Certificate Auto-Generation**:
Ooloi automatically generates TLS certificates when needed:
- **Implementation**: Pure Java with Bouncy Castle (cross-platform, no external dependencies)
- **Default Locations**: 
  - Unix/macOS: `~/.ooloi/certs/server.{crt,key}`
  - Windows: `%APPDATA%\Ooloi\certs\server.{crt,key}`
- **Properties**: RSA 2048-bit, 20-year validity, covers `localhost`, `127.0.0.1`, `::1`
- **Behavior**: Certificates created on first TLS startup, reused on subsequent starts

**TLS Configuration Examples**:
```bash
# Development: TLS disabled (default)
lein run

# Development: TLS with auto-generated certificates (platform defaults)
lein run -- --tls true
OOLOI_TLS=true lein run

# Development: TLS with certificates at custom locations
lein run -- --tls true --cert-path ./my.crt --key-path ./my.key

# Production: Environment variables with custom certificates
export OOLOI_TLS=true
export OOLOI_CERT_PATH=/etc/ssl/ooloi.crt
export OOLOI_KEY_PATH=/etc/ssl/ooloi.key
lein run

# Production: HTTPS port with TLS
export OOLOI_TLS=true
export OOLOI_PORT=443
export OOLOI_CERT_PATH=/etc/ssl/ooloi.crt  
export OOLOI_KEY_PATH=/etc/ssl/ooloi.key
java -jar target/ooloi-backend-*-standalone.jar
```

**Certificate Management**:
- **Automatic**: Certificates are generated automatically when TLS is enabled
- **Persistent**: Once created, certificates are reused across server restarts
- **Configurable**: Use `--cert-path` and `--key-path` to control certificate locations
- **Cross-platform**: Works seamlessly on Unix, macOS, and Windows

#### Production Deployment

For production deployment, use environment variables and proper process management:

```bash
# Set environment variables
export OOLOI_PORT=10700
export OOLOI_DEPLOYMENT_MODE=backend
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
- Multiple deployment modes for different operational scenarios

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

#### Unified gRPC Server Architecture

The backend implements Ooloi's **unified gRPC system** that eliminates complex protocol buffer management:

- **Universal ExecuteMethod**: Single endpoint serves all API methods via dynamic resolution
- **Real-Time Event Streaming**: Server-to-client event notification system with subscription management
- **Unified Schema**: `OoloiValue` message handles all Clojure data types automatically  
- **Zero Code Generation**: No per-method gRPC implementations - just delegate to `api.clj`
- **Dynamic Function Resolution**: `(ns-resolve 'ooloi.backend.api (symbol method-name))`
- **Perfect Type Fidelity**: Ratios, keywords, and custom records preserved automatically

#### gRPC Server Configuration

- **Proto source**: `../shared/src/main/proto/ooloi_service.proto` (unified schema)
- **Generated output**: `target/generated-sources/protobuf/`
- **Service implementation**: Universal `ExecuteMethod` with automatic API delegation
- **Conversion**: Automatic Clojure ↔ `OoloiValue` conversion

#### When Compilation is Needed

**✅ Automatic**: No manual steps needed for:
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
lein run -- --deployment-mode backend --port 10700 &
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

# Production deployment with environment configuration
export OOLOI_PORT=10700
export OOLOI_DEPLOYMENT_MODE=backend
export OOLOI_TLS=true
export OOLOI_CERT_PATH=/etc/ssl/ooloi.crt
export OOLOI_KEY_PATH=/etc/ssl/ooloi.key

# Run production server
java -jar target/ooloi-backend-*-standalone.jar
```

**Container Deployment:**
```dockerfile
# Example Dockerfile snippet
FROM openjdk:22-jre-slim
COPY target/ooloi-backend-*-standalone.jar /app/ooloi-backend.jar
ENV OOLOI_PORT=10700
ENV OOLOI_DEPLOYMENT_MODE=backend
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
- **[Development Plan](/DEV_PLAN.md)** - Current development status and implementation roadmap
- **[ADR-0019: STM-gRPC Batch Transactions](/ADRs/0019-STM-gRPC-Batch-Transactions.md)** - Atomic operation implementation
- **[ADR-0024: gRPC Concurrency and Flow Control Architecture](/ADRs/0024-gRPC-Concurrency-and-Flow-Control-Architecture.md)** - Technical decisions behind communication patterns
