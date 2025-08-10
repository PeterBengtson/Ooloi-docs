# Ooloi Frontend

This directory contains the frontend client code for Ooloi, a high-performance music notation software.

## Table of Contents

1. [Project Role](#project-role)
2. [System Architecture](#system-architecture)
3. [Directory Structure](#directory-structure)
4. [Prerequisites](#prerequisites)
   - [System Requirements](#system-requirements)
   - [Platform-Specific Installation](#platform-specific-installation)
   - [Frontend-Specific Requirements](#frontend-specific-requirements)
   - [Verification](#verification)
   - [Icon Files Setup](#icon-files-setup)
5. [Installation](#installation)
6. [Building the Frontend](#building-the-frontend)
7. [Build Process Details](#build-process-details)
8. [Version Handling](#version-handling)
9. [Configuration and Deployment](#configuration-and-deployment)
   - [Application Architecture](#application-architecture)
   - [Command-Line Arguments](#command-line-arguments)
   - [Environment Variables](#environment-variables)
   - [Deployment Modes](#deployment-modes)
   - [Error Handling and Exit Codes](#error-handling-and-exit-codes)
   - [TLS and Security Configuration](#tls-and-security-configuration)
   - [Integration Testing](#integration-testing)
   - [Monitoring and Health](#monitoring-and-health)
10. [Development](#development)
    - [Running the Frontend](#running-the-frontend)
    - [REPL](#repl)
11. [Development Commands](#development-commands)
    - [Running Tests](#running-tests)
    - [gRPC Integration](#grpc-integration)
    - [Building and Packaging](#building-and-packaging)
    - [Documentation Generation](#documentation-generation)
    - [Development Workflow](#development-workflow)
12. [Shared Model Integration](#shared-model-integration)
    - [Selective Shared Import Architecture](#selective-shared-import-architecture)
    - [Testing Architecture](#testing-architecture)
    - [Generator System Access](#generator-system-access)
    - [Frontend Development Strategy](#frontend-development-strategy)
13. [Notes](#notes)

## Project Role

The **Ooloi Frontend** serves as the user-facing client component providing:

1. **User Interface**: JavaFX-based graphical interface for music notation editing and visualization
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
# Install Java and JavaFX dependencies
sudo apt update
sudo apt install openjdk-22-jdk openjfx

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

### Frontend-Specific Requirements

The frontend requires additional components for UI and graphics:

- **JavaFX** - Included in project dependencies, handles windowing and UI components
- **Skija** - Java bindings for Skia graphics library (handles high-quality 2D rendering)
- **Platform-specific graphics drivers** - Ensure graphics drivers are up to date for optimal rendering

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

Ensure you have the appropriate icon files in the `frontend/resources/icons/` directory:

- **macOS**: `icon.icns`
- **Windows**: `icon.ico` 
- **Linux**: `icon.png`

These can be found in the root `icons/` directory and should be symlinked:
```bash
cd frontend/resources/
ln -s ../../icons/ready/macos/icon.icns icons/
ln -s ../../icons/ready/windows/icon.ico icons/
ln -s ../../icons/ready/linux/icon.png icons/
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

# With security configuration
lein run -- --tls true --cert-path /path/to/cert.pem --key-path /path/to/key.pem

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
| `--cert-path PATH` | file path | - | TLS certificate file path (requires --key-path) |
| `--key-path PATH` | file path | - | TLS private key file path (requires --cert-path) |

**Argument Validation:**
- Port numbers are validated as integers within valid range
- Transport and UI modes are validated against allowed values
- TLS setting accepts only "true" or "false"
- Certificate paths must be provided together (both --cert-path and --key-path)

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
| `OOLOI_FRONTEND_CERT_PATH` | --cert-path | TLS certificate path |
| `OOLOI_FRONTEND_KEY_PATH` | --key-path | TLS private key path |

**Configuration Precedence:** CLI arguments override environment variables, which override application defaults.

#### Environment Variable Usage Examples

```bash
# Basic development configuration
export OOLOI_FRONTEND_BACKEND_HOST=localhost
export OOLOI_FRONTEND_BACKEND_PORT=10700
export OOLOI_FRONTEND_UI_MODE=graphical
lein run

# Remote backend configuration
export OOLOI_FRONTEND_BACKEND_HOST=production-server.company.com
export OOLOI_FRONTEND_BACKEND_PORT=443
export OOLOI_FRONTEND_TLS=true
export OOLOI_FRONTEND_CERT_PATH=/etc/ssl/certs/client.pem
export OOLOI_FRONTEND_KEY_PATH=/etc/ssl/private/client.key
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
The frontend supports secure connections to backend servers:

```bash
# Enable TLS with auto-generated certificates (development)
lein run -- --tls true

# Production TLS with custom certificates
lein run -- --tls true --cert-path ./client.pem --key-path ./client.key

# Environment variable configuration
export OOLOI_FRONTEND_TLS=true
export OOLOI_FRONTEND_CERT_PATH=/etc/ssl/ooloi/client.pem
export OOLOI_FRONTEND_KEY_PATH=/etc/ssl/ooloi/client.key
lein run
```

**TLS Configuration Notes:**
- Both certificate and key paths must be provided together
- Frontend validates certificate path pairs at startup
- TLS setting applies to gRPC client connection to backend
- Invalid certificate configurations result in clear error messages

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

The REPL will start in the `ooloi.frontend.mon.core` namespace.

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

**✅ Automatic**: No manual steps needed for:
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

The frontend integrates with shared model contracts with important architectural constraints discovered during implementation.

### Quick Start: What Most Frontend Code Needs

**For 90% of your frontend code, you only need this:**

```clojure
(ns my-frontend-namespace
  (:require [ooloi.shared.models.core :refer :all]))
```

This gives you access to:
- **All constructors**: `create-pitch`, `create-chord`, `create-measure`, etc. (same as backend)
- **All predicates**: `pitch?`, `chord?`, `measure?` (raw items), plus `pitch??`, `chord??`, `measure??` (timewalk tuples), etc. (same as backend)
- **All multimethods**: `get-duration`, `add-item`, `set-name`, etc. (same interfaces as backend)

**Why this works**: The shared `core` namespace provides the complete Ooloi system that frontend uses identically to backend.

**Architecture Note**: Frontend uses the SAME data model as backend - no conversion, no separate representations. `(= frontend-pitch backend-pitch) => true`

### Selective Shared Import Architecture

**Key Discovery**: Not all shared code is frontend-accessible due to legitimate backend dependencies in some shared files. This is by design, not a limitation.

**Frontend-Safe Shared Code**:
```clojure
;; ✅ Safe for frontend use - completed shared model contracts
[ooloi.shared.ops.text :as text]                    ; Pure utilities (pluralization, etc.)
[ooloi.shared.ops.pitches :as pitches]              ; Pitch normalization utilities
[ooloi.shared.specs.generators :as generators]      ; Test data generators
[ooloi.shared.models.musical.pitch :refer [create-pitch]]  ; Shared model constructors
[ooloi.shared.interfaces :as interfaces]            ; Multimethod interface contracts
[ooloi.shared.predicates :as predicates]            ; Type checking predicates
```

**Selective Import Pattern**:
```clojure
;; ✅ Frontend selectively imports backend-free shared modules
[ooloi.shared.traits.has-items :as has-items]       ; Collection behaviors
[ooloi.shared.traits.rhythmic-item :as rhythmic]    ; Duration-based elements  
;; ⚠️ Some shared files have legitimate backend dependencies - use selective import
```

### Testing Architecture

**Unified System Access**: Frontend uses the complete shared Ooloi system directly:

```clojure
;; Frontend configuration - unified data model access
:source-paths ["src/main/clojure" "../shared/src/main/clojure"]  ; Complete Ooloi system
:test-paths ["test/clojure"]

;; Frontend uses shared system directly - no mocks needed for unified architecture
;; All data models, traits, interfaces, operations now in shared/
;; Test status: 131 passing tests with direct shared system access
```

**Unified Architecture**: Frontend uses the complete shared system directly:

- ✅ **Direct Shared Access**: All models, traits, interfaces, operations in shared/
- ✅ **No Mock Dependencies**: Timewalk, attachments, traits all moved to shared  
- ✅ **Identical Data Model**: Same records, same functions, same everything as backend
- ✅ **Clean Architecture**: Frontend is pure consumer of unified system

```clojure
;; Frontend uses shared system directly (same as backend)
(ns my-frontend-component
  (:require
   [ooloi.shared.models.musical.pitch :refer [create-pitch]]   ; Same as backend
   [ooloi.shared.ops.timewalk :as tw]                          ; Now in shared
   [ooloi.shared.traits.attachment :as att]                    ; Now in shared  
   [ooloi.shared.interfaces :as i]))                           ; Same as backend

;; Creates identical objects to backend
(def my-pitch (create-pitch "C4" "1/4"))  ; Same function, same Pitch record
```

**Architecture Benefits**: Unified system eliminates mock complexity and ensures perfect data fidelity across frontend and backend.

### Generator System Access

**Achievement**: Frontend now has access to comprehensive test data generators:
```clojure
[ooloi.shared.specs.generators :as generators]

;; Available for frontend development and testing
(generators/create-random-pitch)      ; Generate random valid pitch
(generators/create-random-chord)      ; Generate random valid chord  
(generators/gen-note)                 ; Property-based testing generator
(generators/gen-duration)             ; Duration generators with dot support
```

### Frontend Development Strategy

**Unified Data Model Consumer**: Frontend ready for production with:
- ✅ **Identical Data Model** (same records as backend, zero conversion)
- ✅ **Dual Usage Patterns** (local shared calls + remote gRPC calls)
- ✅ **Perfect Object Fidelity** (frontend objects ARE backend objects)
- ✅ **Zero-maintenance system** (unified architecture eliminates complexity)

**Development Reality**: Simple, unified workflow:
```bash
# Frontend uses SAME shared system as backend
(def my-pitch (create-pitch "C4" "1/4"))   ; Same function as backend uses
(pitch? my-pitch)                           ; Same predicate as backend uses

# Make any changes to the unified system
lein midje  # Test - frontend automatically gets all shared improvements
```

**Architecture Principle**: Frontend doesn't have separate data models - it IS a consumer of the complete Ooloi system in shared/, just like backend is.

## Notes

- Ensure all tests pass before creating the final package.
- The packaged application is platform-specific and ready for distribution.
- JavaFX is included in the project dependencies, ensuring GUI capabilities.

Remember to run tests (`lein midje`) before packaging to ensure everything is working correctly.
