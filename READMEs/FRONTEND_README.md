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
    - [Protocol Buffer Generation](#protocol-buffer-generation)
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
├── docs/              ; The HTML documentation for the frontend, created by Codox
├── resources/         ; Resources for the application, notably icons
├── src/               ; The source code hierarchy for the frontend
├── test/              ; The frontend Midje tests
└── CHANGELOG.md       ; The CHANGELOG (as of yet unused)
└── README.md          ; This README
└── project.clj        ; The Leiningen project file for the frontend
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

**Frontend Test Coverage**: 131 passing tests including:
- **Application Infrastructure**: CLI argument parsing, configuration management, environment variables
- **Component Lifecycle**: Integrant component initialization, dependency injection, health monitoring
- **System Integration**: Deployment modes, configuration propagation, error handling
- **gRPC Client Infrastructure**: Protocol buffer message creation and gRPC client class instantiation
- **Shared Model Integration**: Tests for frontend-accessible shared functionality with mock backend support
- **UI Component Testing**: Form validation, user interaction handling, and component state management  
- **Client-side Functionality**: Connection management, error handling, client-side data processing
- **Security Configuration**: TLS validation, certificate path verification, secure connection setup
- **Mock Backend Integration**: Verification that shared traits and interfaces work with stubbed backend dependencies

**Total: 131 tests** covering complete application infrastructure including system architecture, configuration management, and client-side functionality.

### Protocol Buffer Generation

The frontend includes gRPC client infrastructure with automatic Protocol Buffer compilation:

```bash
# Compile Protocol Buffers and generate Java classes
lein compile

# Clean and recompile (recommended when .proto files change)  
lein clean && lein compile

# Verify generated classes
ls target/generated-sources/protobuf/
```

#### Protocol Buffer Configuration

The frontend automatically generates Java client classes from shared Protocol Buffer definitions:

- **Proto source**: `../shared/src/main/proto/` (shared with backend)
- **Generated output**: `target/generated-sources/protobuf/`
- **Compiler version**: protoc 3.25.3 (automatically downloaded)
- **Generated files**: 
  - `ooloi_domain.proto` → Domain model Java classes
  - `ooloi_service.proto` → gRPC client stubs
  - `vpd.proto` → VPD utility classes

#### When to Regenerate Protocol Buffers

**⚠️ Important**: Protocol Buffers must be regenerated after backend API changes:

**Regenerate when backend adds:**
- New API methods to `backend/models/interfaces.clj`, `models/core.clj`, or `api.clj`
- New gRPC service methods or message types
- Changes to existing API method signatures

**Frontend regeneration workflow:**
```bash
# 1. First ensure shared/ and backend/ are regenerated
cd ../shared && lein clean && lein compile
cd ../backend && lein clean && lein compile

# 2. Then regenerate frontend client stubs
cd ../frontend
lein clean
lein protoc

# 3. Verify client compilation
lein compile
lein midje
```

**Signs you need to regenerate:**
- Missing gRPC client methods for new backend API functions
- `NoSuchMethodError` when calling gRPC methods
- Protocol buffer compilation errors
- New backend methods not accessible from frontend

#### Manual Protocol Buffer Commands

```bash
# Force Protocol Buffer regeneration
lein clean
lein protoc

# View generated Java classes
find target/generated-sources/protobuf -name "*.java" | head -10
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
2. **Compile**: `lein compile` (generate Protocol Buffers + compile Clojure)
3. **Test**: `lein midje` (run test suite)
4. **Run**: `lein run` (start frontend client)
5. **Iterate**: Make changes and repeat steps 2-4

## Shared Model Integration

The frontend integrates with shared model contracts with important architectural constraints discovered during implementation.

### Selective Shared Import Architecture

**Key Discovery**: Not all shared code is frontend-accessible due to legitimate backend dependencies in some shared files. This is by design, not a limitation.

**Frontend-Safe Shared Code**:
```clojure
;; ✅ Safe for frontend use
[ooloi.shared.ops.text :as text]                    ; Pure utilities (pluralization, etc.)
[ooloi.shared.ops.pitches :as pitches]              ; Pitch normalization (with proper deps)  
[ooloi.shared.specs.generators :as generators]      ; Test data generators
[ooloi.shared.models.musical.pitch :refer [create-pitch]]  ; Basic model creation
```

**Backend-Dependent Shared Code** (Cannot be loaded by frontend):
```clojure
;; ⚠️ Contains backend dependencies - architecture-dependent, not bugs
[ooloi.shared.traits.attachment :as att]       ; Imports backend.models.musical.instrument
[ooloi.shared.proto-conversion :as proto]      ; Imports backend ops for VPD operations  
[ooloi.shared.models.* :as *]                  ; Many models depend on attachment system
```

### Testing Architecture

**Mock Backend Solution**: Frontend uses mock backend namespaces to enable full shared code access:

```clojure
;; Frontend test configuration  
:source-paths ["src/main/clojure" "../shared/src/main/clojure"]  ; Full shared access
:test-paths ["test/clojure"]

;; Mock backend implementations in frontend
frontend/src/main/clojure/ooloi/backend/ops/piece_manager.clj  ; Mock piece-manager functions
frontend/src/main/clojure/ooloi/backend/ops/timewalk.clj       ; Mock timewalk transducers

;; Test status: 20 passing tests with full shared imports working
```

**Architecture Solution**: Frontend provides stub implementations of backend dependencies that shared traits require. This allows:

- ✅ **Full shared code access**: Import predicates, interfaces, traits normally
- ✅ **`lein midje` auto-discovery**: No manual test file specification required  
- ✅ **Scalable test development**: Hundreds of tests can import shared code
- ✅ **Normal development workflow**: Test any frontend integration with shared functionality

```clojure
;; Frontend can now import any shared code
(ns my-frontend-test
  (:require
   [midje.sweet :refer :all]
   [ooloi.shared.predicates :as p]         ; ✅ Works - predicates load normally
   [ooloi.shared.interfaces :as i]))       ; ✅ Works - traits load with mock backend
```

**Trade-offs**: Mock backend requires maintenance if backend interfaces change, but enables normal frontend test development without architectural constraints.

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

**gRPC Client Readiness**: Frontend is positioned for gRPC client development with:
- ✅ **Test framework validated** (midje working correctly)
- ✅ **Selective shared imports working** (text utilities, generators)  
- ✅ **Architecture constraints understood** (backend-dependent shared modules identified)
- ✅ **Generator system accessible** (comprehensive test data creation)

**Next Step**: gRPC client integration to communicate with backend models via protocol buffers, avoiding direct shared model dependencies where backend integration exists.

## Notes

- Ensure all tests pass before creating the final package.
- The packaged application is platform-specific and ready for distribution.
- JavaFX is included in the project dependencies, ensuring GUI capabilities.

Remember to run tests (`lein midje`) before packaging to ensure everything is working correctly.
