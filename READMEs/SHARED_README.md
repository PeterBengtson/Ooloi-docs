# Ooloi Shared

The shared project has a unique **dual nature** serving both as a common code library and as the integration testing/combined application platform. This dual architecture enables both code reuse and sophisticated deployment scenarios.

## Table of Contents

1. [Dual Nature Architecture](#dual-nature-architecture)
   - [Role 1: Common Code Library](#role-1-common-code-library)
   - [Role 2: Integration Platform & Combined Application Builder](#role-2-integration-platform--combined-application-builder)
2. [System Architecture](#system-architecture)
   - [Current Architecture (Code Library Mode)](#current-architecture-code-library-mode)
   - [Future Architecture (Combined Application Mode)](#future-architecture-combined-application-mode)
3. [Three Critical Architectural Roles](#three-critical-architectural-roles)
   - [Shared Model Contracts](#1-shared-model-contracts)
   - [gRPC Communication Layer](#2-grpc-communication-layer)
   - [Combined Application Builder](#3-combined-application-builder)
4. [Directory Structure](#directory-structure)
5. [Prerequisites](#prerequisites)
   - [System Requirements](#system-requirements)
   - [Platform-Specific Installation](#platform-specific-installation)
   - [Combined Application Requirements](#combined-application-requirements)
   - [Build Dependencies](#build-dependencies)
   - [Verification](#verification)
   - [Icon Files Setup](#icon-files-setup)
6. [gRPC Infrastructure](#grpc-infrastructure)
   - [Protocol Buffer Schema](#protocol-buffer-schema)
   - [Conversion Utilities](#conversion-utilities)
   - [Build Integration](#build-integration)
7. [Installation](#installation)
8. [Building the Combined Application](#building-the-combined-application)
9. [Build Process Details](#build-process-details)
10. [Version Handling](#version-handling)
11. [Development](#development)
    - [Running the Combined Application](#running-the-combined-application)
12. [Future Configuration and Deployment](#future-configuration-and-deployment)
    - [Projected Command-Line Arguments](#projected-command-line-arguments)
    - [Projected Environment Variables](#projected-environment-variables)
    - [Projected Deployment Modes](#projected-deployment-modes)
    - [Future Component Architecture](#future-component-architecture)
    - [Integration Testing Architecture](#integration-testing-architecture)
    - [REPL](#repl)
    - [Monitoring and Health (Future)](#monitoring-and-health-future)
13. [Development Commands](#development-commands)
    - [Running Tests](#running-tests)
    - [Architecture Insights](#architecture-insights)
    - [Dual Nature Development Implications](#dual-nature-development-implications)
    - [Development Workflow Coordination](#development-workflow-coordination)
    - [Protocol Buffer Management](#protocol-buffer-management)
    - [Building and Packaging](#building-and-packaging)
    - [Documentation Generation](#documentation-generation)
    - [Development Workflow](#development-workflow)
14. [Notes](#notes)

## Dual Nature Architecture

The shared project serves **two distinct but complementary roles**:

### 🧩 **Role 1: Common Code Library**
Provides shared model contracts, interfaces, and utilities that both frontend and backend depend on, ensuring perfect type fidelity and eliminating circular dependencies.

### 🏗️ **Role 2: Integration Platform & Combined Application Builder**
Packages and orchestrates both frontend and backend components into a unified application, enabling integration testing and single-process deployment scenarios.

## System Architecture

The shared project enables multiple deployment architectures through its dual nature:

### Current Architecture (Code Library Mode)
```
┌─────────────────┐    ┌─────────────────┐
│    Frontend     │    │     Backend     │
│   Components    │    │   Components    │
└─────────────────┘    └─────────────────┘
         │                       │
         └───────────┬───────────┘
                     ▼
         ┌─────────────────────────┐
         │      Shared Code        │
         │   (Models, Traits,      │
         │  Interfaces, Utils)     │
         └─────────────────────────┘
```

### Future Architecture (Combined Application Mode)
When the shared project gets its own system components:
```
┌─────────────────────────────────────────────────────────┐
│                Shared System Manager                    │
│            (Future Application Core)                    │
└─────────────────┬───────────────────────────────────────┘
                  │
    ┌─────────────▼─────────────┐
    │    Integration Orchestrator    │
    │        Component         │
    └─────────────┬─────────────┘
                  │
        ┌─────────┴─────────┐
        ▼                   ▼
┌─────────────────┐  ┌─────────────────┐
│   Frontend      │  │    Backend      │
│   Components    │  │   Components    │
│                 │  │                 │
│ • gRPC Client   │  │ • Piece Manager │
│ • UI Manager    │  │ • gRPC Server   │
└─────────────────┘  └─────────────────┘
        │                       │
        └───────────┬───────────┘
                    ▼
        ┌─────────────────────────┐
        │     Shared Code         │
        │ (Models, Traits, Utils) │
        └─────────────────────────┘
```

## Three Critical Architectural Roles

### 1. **Shared Model Contracts** 
Contains all data model definitions (`defrecord` structures), interfaces, predicates, and traits that both frontend and backend use, ensuring perfect type fidelity and eliminating circular dependencies.

**Key Components**:
- **Core Data Models**: All musical and visual model `defrecord` definitions
- **Interface Contracts**: Shared multimethod definitions preventing API drift
- **All Trait Implementations**: Behavioral mixins used by both frontend and backend
- **Basic Operations**: Fundamental utilities (access, pitches, rhythm, text)
- **Generator System**: Comprehensive test data generators for all models
- **Selective Backend Integration**: Some shared files legitimately import backend functionality

### 2. **gRPC Communication Layer** 
Provides Protocol Buffer domain model and conversion utilities for frontend-backend communication.

### 3. **Combined Application Builder** 
Combines the backend and frontend components into a single, distributable application.

## Directory structure

```
shared/
├── src/main/proto/                    ; Universal Protocol Buffer definitions
│   ├── ooloi_service.proto           ; Universal gRPC schema (single OoloiValue message)
│   └── vpd.proto                     ; VPD addressing structures
├── src/main/clojure/ooloi/shared/    ; Shared contracts and utilities
│   ├── models/                       ; **Complete shared model contracts**
│   │   ├── musical/                  ; Musical data models (Piece, Musician, Instrument, etc.)  
│   │   ├── visual/                   ; Visual models (Layout, PageView, StaffView, etc.)
│   │   └── changes.clj               ; ChangeSet data structure for time sigs, key sigs, tempos
│   ├── traits/                       ; **ALL trait implementations** (attachment, has-items, rhythmic-item, takes-attachment, transposable)
│   ├── specs/                        ; Shared generator system
│   │   └── generators.clj            ; Test data generators for all models
│   ├── interfaces.clj                ; Shared multimethod interface contracts  
│   ├── predicates.clj                ; Shared predicate functions (musical?, visual?, etc.)
│   ├── hierarchy.clj                 ; Shared type hierarchy and dispatch values
│   ├── ops/                          ; Shared operations
│   │   ├── access.clj                ; Vector/attribute operations
│   │   ├── pitches.clj               ; Pitch normalization and conversion utilities
│   │   ├── rhythm.clj                ; Duration and rhythm processing utilities  
│   │   ├── text.clj                  ; Text processing (pluralization, singularization)
│   │   └── vpd.clj                   ; VPD operations and addressing utilities
│   ├── proto_conversion.clj          ; Clojure ↔ Protocol Buffer conversion utilities
│   ├── vpd_utils.clj                 ; VPD manipulation and protobuf conversion
│   ├── core.clj                      ; Combined application entry point
│   └── build.clj                     ; Build utilities
├── resources/                        ; Resources for the application, notably icons
├── test/                             ; The Midje tests for the shared code
└── project.clj                       ; The Leiningen project file for the application
```

## Prerequisites

### System Requirements

- **Java Development Kit (JDK) 22 or later** - Required for running the combined application
- **Leiningen 2.9.0 or later** - Clojure build tool for dependency management and building
- **Protocol Buffers Compiler (protoc) 3.25.3 or later** - For gRPC code generation (automatically downloaded by lein-protoc)
- **Git** - For version control and accessing the repository
- **Minimum 8GB RAM** - Recommended for combined application with both backend and frontend
- **Network access** - For downloading dependencies and gRPC communication

### Platform-Specific Installation

#### macOS
```bash
# Install Java using Homebrew
brew install openjdk@22

# Install Leiningen
brew install leiningen

# Install Protocol Buffers (optional - lein-protoc will download automatically)
brew install protobuf

# Set environment variables (add to ~/.zshrc or ~/.bash_profile)
export JAVA_HOME="/usr/local/opt/openjdk@22"
export PATH="$JAVA_HOME/bin:$PATH"
```

#### Linux (Ubuntu/Debian)
```bash
# Install Java and dependencies
sudo apt update
sudo apt install openjdk-22-jdk openjfx

# Install Protocol Buffers (optional - lein-protoc will download automatically)
sudo apt install protobuf-compiler

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

3. **Protocol Buffers** (optional):
   - Download from [Protocol Buffers releases](https://github.com/protocolbuffers/protobuf/releases)
   - Extract and add to PATH (lein-protoc will download automatically if not found)

### Combined Application Requirements

The shared project combines both backend and frontend, requiring:

- **gRPC Infrastructure** - Protocol Buffers compilation and Java class generation
- **JavaFX Support** - For frontend UI components when running in combined mode
- **Skija Graphics** - High-performance 2D rendering capabilities
- **Component Lifecycle Management** - Integrant-based system coordination

### Build Dependencies

The shared project automatically handles complex build dependencies:

- **lein-protoc 0.4.1** - Automatically downloads protoc 3.25.3 if not installed
- **protobuf-java 4.31.1** - Runtime protobuf support with version compatibility
- **javax.annotation-api 1.3.2** - Required for generated code annotations
- **gRPC libraries** - Complete gRPC stack for network and in-process communication

### Verification

Verify your installation:
```bash
# Check Java version
java -version
# Should show: openjdk version "22.x.x" or later

# Check Leiningen
lein version
# Should show: Leiningen 2.9.0 or later on Java 22.x.x

# Check Protocol Buffers (optional)
protoc --version
# Should show: libprotoc 3.x.x or later (or will be downloaded automatically)

# Check environment
echo $JAVA_HOME
# Should show path to Java installation
```

### Icon Files Setup

Ensure you have the appropriate icon files in the `shared/resources/icons/` directory:

- **macOS**: `icon.icns`
- **Windows**: `icon.ico` 
- **Linux**: `icon.png`

These can be found in the root `icons/` directory and should be symlinked:
```bash
cd shared/resources/
ln -s ../../icons/ready/macos/icon.icns icons/
ln -s ../../icons/ready/windows/icon.ico icons/
ln -s ../../icons/ready/linux/icon.png icons/
```

## gRPC Infrastructure

The shared project contains Protocol Buffer definitions for frontend-backend communication:

### Protocol Buffer Schema

- **`ooloi_service.proto`**: gRPC service definitions
- **`vpd.proto`**: Vector Path Descriptor addressing structures

### Conversion Utilities

- **`proto_conversion.clj`**: Clojure ↔ Protocol Buffer conversion
- **`vpd_utils.clj`**: VPD manipulation and protobuf conversion

### Build Integration

- `lein compile` generates Java classes in `target/generated-sources/protobuf/`
- Generated classes are shared across all projects (shared, backend, frontend)
- Supports both distributed (client-server) and combined (single-JVM) deployment modes

## Installation

  ```bash
  cd shared
  lein deps
  ```

## Building the Combined Application

1. Navigate to the shared directory:
   ```bash
   cd shared
   ```

2. Run tests:
   ```bash
   lein midje
   ```
   
   The test suite includes comprehensive coverage of:
   - Protocol Buffer conversion utilities (62 tests)
   - VPD manipulation functions (12 tests)  
   - Round-trip conversion symmetry for all data types
   - Context-aware VPD auto-detection

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

4. The packaged shared application will be in the `shared/target` directory.

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

## Development

### Running the Combined Application

To run the application for development:

```bash
lein run
```

This will start the Ooloi combined application.

## Future Configuration and Deployment

When the shared project evolves to have its own system components and CLI, it will likely support comprehensive configuration for orchestrating both frontend and backend components:

### Projected Command-Line Arguments

Based on the dual nature and integration requirements, the shared project will likely support:

```bash
# Basic combined application
lein run

# Integration testing mode
lein run -- --mode integration-test --frontend-port 10800 --backend-port 10700

# Single-process combined deployment
lein run -- --mode combined --ui-mode graphical --transport in-process

# Distributed coordination mode  
lein run -- --mode distributed --frontend-host remote-frontend --backend-host remote-backend

# Development orchestration
lein run -- --mode dev-orchestrator --auto-restart true --log-level debug
```

#### Anticipated CLI Arguments

| **Argument** | **Values** | **Default** | **Description** |
|--------------|------------|-------------|-----------------|
| `--mode MODE` | integration-test, combined, distributed, dev-orchestrator | combined | Deployment orchestration mode |
| `--frontend-host HOST` | hostname/IP | localhost | Frontend component hostname |
| `--frontend-port PORT` | 1-65535 | 10800 | Frontend component port |
| `--backend-host HOST` | hostname/IP | localhost | Backend component hostname |
| `--backend-port PORT` | 1-65535 | 10700 | Backend component port |
| `--transport MODE` | network, in-process, auto | auto | Inter-component transport |
| `--ui-mode MODE` | graphical, headless, auto | auto | UI display mode for combined deployment |
| `--integration-timeout MS` | milliseconds | 30000 | Integration test timeout |
| `--auto-restart BOOL` | true, false | false | Auto-restart components on failure |
| `--log-level LEVEL` | debug, info, warn, error | info | Application logging level |
| `--health-port PORT` | 1-65535 | 10801 | Combined application health endpoint |
| `--tls BOOL` | true, false | false | Enable TLS for all communications |

### Projected Environment Variables

All CLI arguments would have corresponding environment variable alternatives:

| **Environment Variable** | **CLI Equivalent** | **Description** |
|-------------------------|-------------------|-----------------|
| `OOLOI_SHARED_MODE` | --mode | Deployment orchestration mode |
| `OOLOI_SHARED_FRONTEND_HOST` | --frontend-host | Frontend component hostname |
| `OOLOI_SHARED_FRONTEND_PORT` | --frontend-port | Frontend component port |
| `OOLOI_SHARED_BACKEND_HOST` | --backend-host | Backend component hostname |
| `OOLOI_SHARED_BACKEND_PORT` | --backend-port | Backend component port |
| `OOLOI_SHARED_TRANSPORT` | --transport | Inter-component transport mode |
| `OOLOI_SHARED_UI_MODE` | --ui-mode | UI display mode |
| `OOLOI_SHARED_INTEGRATION_TIMEOUT` | --integration-timeout | Integration test timeout |
| `OOLOI_SHARED_AUTO_RESTART` | --auto-restart | Auto-restart on component failure |
| `OOLOI_SHARED_LOG_LEVEL` | --log-level | Application logging level |
| `OOLOI_SHARED_HEALTH_PORT` | --health-port | Health monitoring port |
| `OOLOI_SHARED_TLS` | --tls | Enable TLS communications |

### Projected Deployment Modes

#### integration-test
**Components:** Integration Orchestrator + Test Frontend + Test Backend  
**Use Case:** Deep integration testing between frontend and backend components
```bash
lein run -- --mode integration-test
```
- Starts both frontend and backend in test mode
- Runs comprehensive integration test suite
- Validates gRPC communication paths
- Tests all transport modes and failure scenarios

#### combined
**Components:** All components in single JVM  
**Use Case:** Single-process deployment with maximum performance
```bash
lein run -- --mode combined --transport in-process
```
- Ultra-high-performance in-process communication
- Single JVM with all components
- Minimal resource usage
- Ideal for standalone deployments

#### distributed
**Components:** Coordination of remote frontend and backend  
**Use Case:** Multi-machine deployment coordination
```bash
lein run -- --mode distributed --frontend-host client-machine --backend-host server-machine
```
- Coordinates remote component deployment
- Manages inter-machine communication
- Handles distributed failure scenarios
- Load balancing and health coordination

#### dev-orchestrator
**Components:** Development tooling and component management  
**Use Case:** Development environment management
```bash
lein run -- --mode dev-orchestrator --auto-restart true
```
- Auto-restart components on code changes
- Development-friendly logging and debugging
- Hot reloading coordination
- Developer productivity optimizations

### Future Component Architecture

The shared system would likely include these Integrant components:

#### Integration Orchestrator Component
- **Component coordination** between frontend and backend
- **Health monitoring** across all components
- **Failure detection** and automatic recovery
- **Transport optimization** based on deployment mode

#### Configuration Manager Component  
- **Environment detection** and mode selection
- **Component configuration** propagation
- **Runtime reconfiguration** support
- **Deployment validation** and compatibility checking

#### Test Coordination Component
- **Integration test execution** across frontend-backend boundary
- **Test data management** using shared generators
- **Performance benchmarking** of different transport modes
- **Regression test automation** for gRPC communication

### Integration Testing Architecture

The shared project would serve as the **integration testing hub**:

```bash
# Comprehensive integration testing
export OOLOI_SHARED_MODE=integration-test
export OOLOI_SHARED_INTEGRATION_TIMEOUT=60000
lein run

# Performance benchmarking
lein run -- --mode integration-test --transport in-process --benchmark true

# Multi-user collaboration testing
lein run -- --mode integration-test --multi-client true --client-count 5
```

### REPL

For interactive development, you can start a REPL:

```bash
lein repl
```

The REPL will start in the `ooloi.shared.mon.core` namespace.

### Monitoring and Health (Future)

When the shared project gets its own system components, it will provide comprehensive monitoring for the combined application:

**Multi-Component Health Status:**
- Health aggregation across frontend and backend components
- Inter-component communication monitoring
- Distributed system health coordination
- Cross-boundary error tracking and recovery

**Integration Test Monitoring:**
- Real-time integration test execution status
- Performance metrics across transport modes
- Regression detection and alerting
- Multi-user collaboration test results

**Application Lifecycle Coordination:**
- **Startup**: Orchestrates component initialization in correct dependency order
- **Runtime**: Monitors health across component boundaries, handles partial failures
- **Shutdown**: Coordinates graceful shutdown of all managed components

**Production Readiness (Future):**
- **Distributed health endpoints**: Coordination across multiple machines
- **Integration monitoring**: Real-time gRPC communication health
- **Performance telemetry**: Transport optimization metrics and recommendations
- **Component isolation**: Failure isolation preventing cascade failures

## Development Commands

### Running Tests

The shared project uses Midje for comprehensive testing:

```bash
# Run all tests
lein midje

# Run specific test namespace
lein midje ooloi.shared.proto-conversion-test

# Run tests with coverage report
lein coverage
```

The test suite includes comprehensive coverage of:
- **Protocol Buffer conversion utilities** (62 tests) - Clojure ↔ protobuf conversion logic
- **VPD manipulation functions** (12 tests) - Vector Path Descriptor utilities  
- **Shared model contracts** (~1,587 tests) - All shared model functionality and generator correctness
- **gRPC Integration tests** - Real client-server communication testing (in-process, network, distributed)
- **Build utilities** - Cross-project packaging and deployment tools

**Total: ~1,587 tests** covering all shared functionality plus integration testing between frontend and backend.

### Architecture Insights

**Frontend Integration Constraints**: Not all shared code is frontend-accessible due to legitimate architectural dependencies:

```clojure
;; Frontend-safe shared code
[ooloi.shared.ops.text :as text]           ; ✅ Pure utilities
[ooloi.shared.models.musical.pitch :refer [create-pitch]]  ; ✅ Basic models
[ooloi.shared.specs.generators :as gen]    ; ✅ Test generators

;; Backend-dependent shared code (legitimate architectural pattern)  
[ooloi.shared.traits.attachment :as att]   ; ⚠️ Imports backend.models.musical.instrument
[ooloi.shared.proto-conversion :as proto]  ; ⚠️ Imports backend ops for VPD conversion
```

**Design Principle**: Shared files CAN import backend dependencies when architecturally justified (attachment resolution, VPD operations, etc.), but frontend must selectively import only backend-free shared modules.

**Generator Accessibility**: Test data generators in `shared/src` are available for frontend development and testing.

### Dual Nature Development Implications

**Code Library Development:**
```bash
# Developing shared code used by both projects
cd shared/
# Edit shared models, traits, interfaces
lein midje  # Test shared code in isolation
cd ../backend && lein midje  # Test backend integration
cd ../frontend && lein midje  # Test frontend integration
```

**Integration Platform Development:**  
```bash
# Developing combined application features
cd shared/
lein run  # Test combined application
# Edit integration orchestration components
lein midje  # Test integration scenarios
```

**Deployment Flexibility Benefits:**
- **Single Repository**: All three projects in one repo with shared development workflow
- **Selective Dependencies**: Frontend/backend can import only needed shared modules
- **Integration Testing**: Comprehensive testing across component boundaries
- **Combined Packaging**: Single JAR for deployment scenarios requiring both components
- **Code Reuse**: Maximum reuse while maintaining architectural boundaries

### Development Workflow Coordination

**Protocol Buffer Changes:**
1. Edit `.proto` files in `shared/src/main/proto/`  
2. Compile in shared: `lein clean && lein compile`
3. Propagate to backend: `cd ../backend && lein clean && lein compile`
4. Propagate to frontend: `cd ../frontend && lein clean && lein compile`
5. Test integration: `cd ../shared && lein midje`

**Shared Model Changes:**
1. Edit models/traits/interfaces in `shared/src/main/clojure/`
2. Test shared: `cd shared && lein midje`  
3. Test backend compatibility: `cd ../backend && lein midje`
4. Test frontend compatibility: `cd ../frontend && lein midje`
5. Test integration scenarios: `cd ../shared && lein midje`

### Protocol Buffer Management

The shared project contains the **source Protocol Buffer definitions** used by both backend and frontend:

```bash
# Compile Protocol Buffers and generate Java classes
lein compile

# Clean and recompile (recommended when editing .proto files)  
lein clean && lein compile

# View generated Protocol Buffer Java classes
ls target/generated-sources/protobuf/
```

#### Protocol Buffer Source Files

- **`src/main/proto/ooloi_service.proto`** - gRPC service definitions
  - Service methods: `ExecuteMethod` and `ExecuteBatch` 
  - Universal data handling via `OoloiValue` message type

- **`src/main/proto/vpd.proto`** - VPD addressing structures
  - Vector Path Descriptor message definitions for navigation

#### When to Recompile Protocol Buffers

Recompile when:
- Modifying `.proto` files in `src/main/proto/`
- After `git pull` that updates protobuf definitions
- Updating protobuf compiler or dependencies

**Compilation workflow**:

```bash
# 1. In shared/ directory
lein clean && lein compile

# 2. In backend/ directory
cd ../backend && lein clean && lein compile

# 3. In frontend/ directory
cd ../frontend && lein clean && lein compile

# 4. Test all projects
cd ../backend && lein midje  # Test backend
cd ../frontend && lein midje # Test frontend  
cd ../shared && lein midje   # Test shared
```

### Building and Packaging

```bash
# Full build process (clean, compile, test, package combined application)
lein build

# Create standalone JAR for combined application
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

Recommended development sequence for gRPC infrastructure changes:

1. **Edit Protocol Buffers**: Modify .proto files in `src/main/proto/`
2. **Regenerate**: `lein clean && lein compile` (in shared/)
3. **Update backend**: `cd ../backend && lein clean && lein compile`
4. **Update frontend**: `cd ../frontend && lein clean && lein compile`  
5. **Test**: `lein midje` (run conversion utility tests)
6. **Integration test**: Test backend/frontend communication

## Notes

- Ensure all tests pass before creating the final package.
- The packaged application is platform-specific and ready for distribution.
- JavaFX is included in the project dependencies, ensuring GUI capabilities.

Remember to run tests (`lein midje`) before packaging to ensure everything is working correctly.
