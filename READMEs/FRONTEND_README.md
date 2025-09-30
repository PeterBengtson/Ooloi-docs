# Ooloi Frontend

This directory contains the frontend client code for Ooloi, a high-performance music notation software.

## Table of Contents

<img src="../img/frontend-ooloi.png" alt="Ooloi Frontend Architecture" align="right" width="500">

1. [Project Role](#project-role)
2. [System Architecture](#system-architecture)
3. [Directory Structure](#directory-structure)
4. [Prerequisites](#prerequisites)
5. [Installation](#installation)
6. [Building the Frontend](#building-the-frontend)
7. [Build Process Details](#build-process-details)
8. [Version Handling](#version-handling)
9. [Configuration and Deployment](#configuration-and-deployment)
10. [Development](#development)
11. [Development Commands](#development-commands)
12. [Shared Model Integration](#shared-model-integration)
13. [Notes](#notes)
14. [Related Documentation](#related-documentation)

## Project Role

The **Ooloi Frontend** serves as the user-facing client component providing:

1. **User Interface**: cljfx + JavaFX-based graphical interface with AtlantaFX dark theme for modern, futuristic music notation editing and visualization
2. **Score Rendering**: High-performance visual display of musical structures and notation
3. **gRPC Client Integration**: Communication with backend via protocol buffer-based unified interface
4. **User Interaction Management**: Click handling, form validation, real-time UI updates, and client-side state management

**Key Architectural Responsibility**: Implements presentation layer functionality and user experience while communicating with backend server through gRPC for all musical data operations.

## System Architecture

The frontend is a full-featured application using **Integrant dependency injection** for component lifecycle management:

```
┌─────────────────┐    ┌──────────────────┐
│   Application   │    │   Environment    │
│      Core       │◄──►│   Configuration  │
└─────────────────┘    └──────────────────┘
         │
         ▼
┌─────────────────┐    ┌──────────────────┐
│   gRPC Client   │◄──►│   UI Manager     │
│   Component     │    │   Component      │
└─────────────────┘    └──────────────────┘
         │                       │
         ▼                       ▼
┌─────────────────┐    ┌──────────────────┐
│  Backend Server │    │ User Interface   │
│  Communication  │    │ & Interactions   │
└─────────────────┘    └──────────────────┘
```

**Component Dependencies:**
- **Application Core** → **gRPC Client** → **UI Manager**
- Configuration flows from CLI/environment through all components
- Clean shutdown ensures proper resource cleanup in reverse dependency order

## Directory structure

```
frontend/
├── docs/                            ; HTML documentation created by Codox
├── resources/                       ; Application resources, icons
├── src/main/clojure/ooloi/frontend/ ; Frontend consumer source code
│   ├── components/                  ; UI components using shared data model
│   │   ├── score_view.clj           ; Score rendering with shared Piece records
│   │   └── measure_editor.clj       ; Measure editing using shared API
│   ├── grpc/                        ; gRPC client for remote backend access
│   │   ├── client.clj               ; OoloiService client wrapper
│   │   └── connection.clj           ; Connection management
│   └── core.clj                     ; Application entry point
├── test/clojure/ooloi/frontend/     ; Frontend consumer tests (131 tests)
│   ├── components/                  ; UI component tests
│   ├── grpc/                        ; gRPC client tests
│   └── integration/                 ; Unified data model integration tests
├── CHANGELOG.md
├── README.md
└── project.clj                      ; Dependencies: shared/ (complete Ooloi system)
```

## Prerequisites

### System Requirements

- **Java Development Kit (JDK) 22 or later** - Required for running Clojure and JavaFX components
- **Leiningen 2.9.0 or later** - Clojure build tool for dependency management and project operations
- **Git** - For version control and accessing the repository
- **Minimum 8GB RAM** - Recommended for UI development and rendering large musical scores
- **Graphics acceleration support** - For optimal Skija rendering performance
- **Network access** - For downloading dependencies during initial setup

### Platform Setup

**Requirements**: Java 22+ and Leiningen 2.9.0+

#### macOS
```bash
brew install openjdk@22 leiningen
export JAVA_HOME="/usr/local/opt/openjdk@22"
```

#### Linux (Ubuntu/Debian)
```bash
sudo apt install openjdk-22-jdk openjfx
curl https://raw.githubusercontent.com/technomancy/leiningen/stable/bin/lein > ~/bin/lein
chmod +x ~/bin/lein
export JAVA_HOME="/usr/lib/jvm/java-22-openjdk-amd64"
```

#### Windows
- Install OpenJDK 22 from [Adoptium](https://adoptium.net/)
- Download `lein.bat` from [Leiningen](https://leiningen.org/) and run `lein self-install`
- Set `JAVA_HOME` environment variable

#### Frontend Dependencies
UI components (JavaFX, AtlantaFX, Skija) are included in project dependencies.

#### Verification
```bash
java -version && lein version
```

## Installation

  ```bash
  cd frontend
  lein deps
  ```

## Building the Frontend

1. Navigate to the frontend directory:
   ```bash
   cd frontend
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
   - Run jlink to create a custom runtime
   - Run the build function to package the application

   You will see colored output indicating the progress of each step.

4. The packaged frontend application will be in the `frontend/target` directory.

## Build Process Details

The build process uses the following tools and steps:

1. **Cleaning**: Removes previous build artifacts.
2. **Uberjar Creation**: Compiles the code and creates a standalone jar file.
3. **Jlink**: Creates a custom Java runtime with only the necessary modules.
4. **Jpackage**: Packages the application for distribution, creating platform-specific installers or application images.

The build process handles both SNAPSHOT (development) versions and release versions:

- For SNAPSHOT versions, it creates an app image.
- For release versions, it creates platform-specific installers (DMG for macOS, DEB for Linux, MSI for Windows).

## Version Handling

The build process automatically adjusts version numbers for compatibility with different platforms:

- SNAPSHOT suffixes are removed for the final package version.
- If the version starts with "0", it's changed to "1.0.0" for macOS compatibility.
- The original version (including SNAPSHOT if applicable) is included in the application name.

## Configuration and Deployment

### Application Architecture

The Ooloi Frontend is now a full-featured application with comprehensive system architecture matching the backend. It uses Integrant dependency injection for component lifecycle management and supports multiple deployment scenarios.

**Key Components:**
- **gRPC Client**: Manages connection to backend server
- **UI Manager**: Handles user interface lifecycle and user interactions
- **Application Core**: CLI argument parsing, configuration management, error handling

### Command-Line Arguments

The frontend supports comprehensive command-line configuration:

```bash
# Basic usage
lein run

# With backend connection configuration
lein run -- --backend-host remote-server --backend-port 10700

# With UI and transport options  
lein run -- --ui-mode graphical --transport network

# With TLS security
lein run -- --tls true --cert-path /path/to/server-cert.pem

# Complete configuration example
lein run -- --backend-host localhost --backend-port 10700 --transport network --ui-mode graphical --deployment-mode frontend --timeout-ms 5000 --tls false
```

#### Available CLI Arguments

| **Argument** | **Values** | **Default** | **Description** |
|--------------|------------|-------------|-----------------|
| `--backend-host HOST` | hostname/IP | localhost | Backend server hostname or IP address |
| `--backend-port PORT` | 1-65535 | 10700 | Backend server port number |
| `--transport MODE` | network, in-process | network | Communication transport mechanism |
| `--ui-mode MODE` | graphical, headless | graphical | User interface display mode |
| `--deployment-mode MODE` | frontend, combined-client, dev-client-only | frontend | Application deployment configuration |
| `--timeout-ms MS` | milliseconds | 5000 | Backend connection timeout |
| `--tls FLAG` | true, false | false | Enable TLS encryption for backend connection |
| `--cert-path PATH` | file path | auto-discovered | Path to server's public certificate (TLS only) |

**Argument Validation:**
- Port numbers are validated as integers within valid range
- Transport and UI modes are validated against allowed values
- TLS setting accepts only "true" or "false"
- Certificate path is optional when TLS enabled (auto-discovers from `~/.ooloi/certs/`)

### Environment Variables

All CLI arguments have corresponding environment variable alternatives:

| **Environment Variable** | **CLI Equivalent** | **Description** |
|-------------------------|-------------------|-----------------|
| `OOLOI_FRONTEND_BACKEND_HOST` | --backend-host | Backend server hostname |
| `OOLOI_FRONTEND_BACKEND_PORT` | --backend-port | Backend server port |
| `OOLOI_FRONTEND_TRANSPORT` | --transport | Transport mode (network/in-process) |
| `OOLOI_FRONTEND_UI_MODE` | --ui-mode | UI mode (graphical/headless) |
| `OOLOI_FRONTEND_DEPLOYMENT_MODE` | --deployment-mode | Deployment configuration |
| `OOLOI_FRONTEND_TIMEOUT_MS` | --timeout-ms | Connection timeout in milliseconds |
| `OOLOI_FRONTEND_TLS` | --tls | Enable TLS (true/false) |
| `OOLOI_FRONTEND_CERT_PATH` | --cert-path | Path to server's public certificate |

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

# Headless mode for automated testing
export OOLOI_FRONTEND_UI_MODE=headless
export OOLOI_FRONTEND_DEPLOYMENT_MODE=dev-client-only
lein run
```

### Deployment Modes

The frontend supports multiple deployment configurations:

#### frontend (Default)
**Components:** gRPC Client + UI Manager  
**Use Case:** Standard client deployment connecting to remote backend server
```bash
lein run -- --deployment-mode frontend
```
- Connects to external backend server
- Full graphical user interface
- Typical production client configuration

#### combined-client
**Components:** All client components  
**Use Case:** Single-process client with comprehensive functionality
```bash
lein run -- --deployment-mode combined-client
```
- All available client-side components
- Enhanced local functionality
- Suitable for standalone client deployments

#### dev-client-only
**Components:** gRPC Client only (minimal)  
**Use Case:** Development and testing scenarios
```bash
lein run -- --deployment-mode dev-client-only
```
- Minimal client for development
- No UI components (lightweight)
- Ideal for integration testing and debugging

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
The frontend supports secure connections to TLS-enabled backend servers using one-way TLS (server authentication):

```bash
# Enable TLS with automatic certificate discovery
lein run -- --tls true

# Explicit certificate path
lein run -- --tls true --cert-path /path/to/server-cert.pem

# Environment variable configuration
export OOLOI_FRONTEND_TLS=true
export OOLOI_FRONTEND_CERT_PATH=/etc/ssl/ooloi/server.pem
lein run
```

**TLS Configuration Notes:**
- **One-way TLS**: Clients verify server identity using server's public certificate
- **Certificate Discovery**: When `--cert-path` not specified, automatically discovers certificate from `~/.ooloi/certs/`
- **Requirements**: Expects exactly one `.crt` file in certificate directory for auto-discovery
- **Explicit Path**: Use `--cert-path` to specify custom certificate location
- **Server Certificate Only**: Clients only need the server's public certificate, never private keys
- **Security**: TLS encrypts all gRPC communication between client and backend

### Integration Testing

The frontend can be started as a separate process for integration testing:

```bash
# Start frontend in background for testing
lein run -- --deployment-mode dev-client-only &
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

The REPL will start in the `ooloi.frontend.core` namespace.

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
2. **Compile**: `lein compile` (compile application with gRPC client)
3. **Test**: `lein midje` (run test suite)
4. **Run**: `lein run` (start frontend client)
5. **Iterate**: Make changes and repeat steps 2-4

## Shared Model Integration

See [ADR-0023: Shared Model Contracts](../ADRs/0023-Shared-Model-Contracts.md) for frontend-specific shared model usage patterns, selective import architecture, and unified development workflow.

## Notes

- Ensure all tests pass before creating the final package.
- The packaged application is platform-specific and ready for distribution.
- JavaFX is included in the project dependencies, ensuring GUI capabilities.

Remember to run tests (`lein midje`) before packaging to ensure everything is working correctly.

## Related Documentation

### Architecture Guides  
- **[Polymorphic API Guide](/guides/POLYMORPHIC_API_GUIDE.md)** - Type system foundations that frontend development builds upon
- **[gRPC Communication and Flow Control](/guides/GRPC_COMMUNICATION_AND_FLOW_CONTROL.md)** - Client-server communication patterns for frontend integration
- **[Ooloi Server Architectural Guide](/guides/OOLOI_SERVER_ARCHITECTURAL_GUIDE.md)** - Understanding the server architecture that frontend connects to

### Technical Documentation
- **[GitHub Project Board](https://github.com/users/PeterBengtson/projects/4)** - Current development status and implementation roadmap
- **[ADR-0023: Shared Model Contracts](/ADRs/0023-Shared-Model-Contracts.md)** - Shared model architecture that frontend uses
- **[ADR-0022: Lazy Frontend-Backend Architecture](/ADRs/0022-Lazy-Frontend-Backend-Architecture.md)** - Frontend-backend interaction patterns
