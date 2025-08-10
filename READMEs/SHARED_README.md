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
    - [gRPC Infrastructure Management](#grpc-infrastructure-management)
    - [Building and Packaging](#building-and-packaging)
    - [Documentation Generation](#documentation-generation)
    - [Development Workflow](#development-workflow)
14. [Notes](#notes)

## Core Architecture

The shared project is **the complete Ooloi data model and API**:

### 🎯 **Primary Role: Complete Data Model**
Contains the entire Ooloi data model, business logic, and API - this IS Ooloi's core functionality.

### 🌐 **Secondary Role: Network Transport & Combined Applications**
Provides gRPC infrastructure for network access and packages combined applications for deployment.

### 📐 **Project Relationship**
- **Shared**: The complete Ooloi system (data models, business logic, API)
- **Backend**: gRPC wrapper around shared (network transport layer)
- **Frontend**: Consumer of shared data model (local + remote via gRPC)

### ⚡ **CRITICAL: Unified Data Representation**
**There are NO separate frontend/backend data representations.**
- Same `Pitch` record used by frontend and backend
- Same `Piece` structure, same `Musician` model, same everything
- Frontend creates identical data structures to backend
- Perfect round-trip fidelity: frontend ↔ gRPC ↔ backend use identical objects

## System Architecture

The shared project enables multiple deployment architectures through its dual nature:

### Unified Data Model Architecture
```
         ┌─────────────────────────┐
         │     SHARED PROJECT      │
         │  (Complete Ooloi Core)  │
         │                         │
         │ • All Data Models       │
         │ • All Business Logic    │
         │ • Complete API          │
         │ • Traits & Interfaces   │
         │ • Operations & Utils    │
         └─────────────┬───────────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  Frontend   │ │   Backend   │ │  Combined   │
│  Consumer   │ │gRPC Wrapper │ │Application  │
│             │ │             │ │             │
│• UI Layer   │ │• Network    │ │• Both       │
│• Local Use  │ │  Transport  │ │  Together   │
│• gRPC Client│ │• Delegation │ │• Single JVM │
└─────────────┘ └─────────────┘ └─────────────┘
```

### Unified Data Model Usage
```
SHARED (Complete Ooloi System)
├── Frontend Local Usage
│   (create-pitch "C4" "1/4")           ← SAME Pitch record as backend
│   (pitch? some-pitch)                ← SAME predicates as backend
│
├── Frontend Remote Usage (via gRPC)
│   Frontend ──gRPC──→ Backend ──delegates──→ SAME shared functions
│   (grpc-call :create-pitch {...})     ← Returns SAME Pitch record
│
└── Backend Usage
    (create-pitch "C4" "1/4")           ← IDENTICAL to frontend
    (pitch? some-pitch)                ← IDENTICAL to frontend

⚡ KEY: Same data structures, same functions, same everything!
```

## Architecture Roles

### 1. **Complete Ooloi System** 
Shared IS Ooloi - contains all functionality:

**Complete Implementation**:
- **Single Data Model**: ONE set of `defrecord` structures used by ALL projects
- **Unified API**: ONE set of functions called identically by frontend and backend  
- **Shared Business Logic**: Interfaces, traits, predicates, operations, algorithms
- **Identical Objects**: Same `Pitch`, `Piece`, `Musician` records everywhere
- **Generator System**: Creates identical test data for all projects
- **Core Operations**: Access, pitches, rhythm, text processing, timewalk, attachments

### 2. **Network Transport Layer** 
Provides unified gRPC infrastructure for remote access to the complete system.

### 3. **Combined Application Builder** 
Packages complete applications by combining shared core with frontend/backend wrappers.

## Directory structure

```
shared/
├── docs/                             ; HTML documentation generated by Codox
├── resources/                        ; Application resources
│   └── icons/                        ; Platform-specific icons (icns, ico, png)
├── src/main/                         ; Source code
│   ├── proto/                        ; Unified gRPC Schema
│   │   └── ooloi_service.proto       ; Complete unified gRPC schema (OoloiValue + ExecuteMethod)
│   └── clojure/ooloi/shared/         ; **Complete Ooloi System**
│       ├── build.clj                 ; Build utilities and packaging
│       ├── core.clj                  ; Combined application entry point
│       ├── hierarchy.clj             ; Shared type hierarchy and dispatch values
│       ├── interfaces.clj            ; Shared multimethod interface contracts
│       ├── predicates.clj            ; Shared predicate functions (musical?, visual?, etc.)
│       ├── proto_conversion.clj      ; Clojure ↔ Protocol Buffer conversion utilities
│       ├── models/                   ; **Complete Data Model**
│       │   ├── core.clj              ; Model core and macro system
│       │   ├── changes.clj           ; ChangeSet data structure for time sigs, key sigs, tempos
│       │   ├── musical/              ; Musical data models
│       │   │   ├── piece.clj         ; Piece - root musical container
│       │   │   ├── musician.clj      ; Musician - performer representation
│       │   │   ├── instrument.clj    ; Instrument - musical instrument definition
│       │   │   ├── staff.clj         ; Staff - staff representation
│       │   │   ├── voice.clj         ; Voice - voice within staff
│       │   │   ├── measure.clj       ; Measure - musical measure
│       │   │   ├── pitch.clj         ; Pitch - musical pitch
│       │   │   ├── chord.clj         ; Chord - multiple simultaneous pitches
│       │   │   ├── rest.clj          ; Rest - musical rest
│       │   │   ├── tuplet.clj        ; Tuplet - rhythmic grouping
│       │   │   ├── tremolando.clj    ; Tremolando - rapid repetition
│       │   │   └── attachments/      ; Musical attachments
│       │   │       ├── articulation.clj ; Articulation markings
│       │   │       ├── dynamic.clj   ; Dynamic markings (forte, piano, etc.)
│       │   │       ├── glissando.clj ; Glissando markings
│       │   │       ├── hairpin.clj   ; Hairpin crescendo/diminuendo
│       │   │       ├── ottava.clj    ; Octave displacement markings
│       │   │       ├── slur.clj      ; Slur markings
│       │   │       └── tie.clj       ; Tie markings
│       │   └── visual/               ; Visual layout models
│       │       ├── layout.clj        ; Overall layout configuration
│       │       ├── page_view.clj     ; Page layout view
│       │       ├── system_view.clj   ; System layout view
│       │       ├── staff_view.clj    ; Staff layout view
│       │       └── measure_view.clj  ; Measure layout view
│       ├── ops/                      ; **Complete Operations**
│       │   ├── access.clj            ; Vector/attribute operations
│       │   ├── attachment_resolver.clj ; Attachment resolution operations
│       │   ├── changes.clj           ; Change-based operations
│       │   ├── persistence.clj       ; Persistence operations
│       │   ├── piece_ref.clj         ; Piece reference operations
│       │   ├── pitches.clj           ; Pitch normalization and conversion utilities
│       │   ├── rhythm.clj            ; Duration and rhythm processing utilities
│       │   ├── text.clj              ; Text processing (pluralization, singularization)
│       │   ├── timewalk.clj          ; Musical structure traversal and analysis operations
│       │   └── vpd.clj               ; VPD operations and addressing utilities
│       ├── specs/                    ; Test data generation
│       │   └── generators.clj        ; Test data generators for all models
│       └── traits/                   ; **All Behavioral Traits**
│           ├── attachment.clj        ; Attachment behavior trait
│           ├── has_items.clj         ; Collection behavior trait
│           ├── rhythmic_item.clj     ; Rhythmic behavior trait
│           ├── takes_attachment.clj  ; Attachment acceptance trait
│           └── transposable.clj      ; Transposition behavior trait
├── test/                             ; Comprehensive test suite (~15,048 tests)
├── util/                             ; Utility scripts
└── project.clj                       ; Project configuration and dependencies
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

### Conversion Utilities

- **`proto_conversion.clj`**: Clojure ↔ Protocol Buffer conversion

### Build Integration

- `lein compile` generates unified gRPC infrastructure in `target/generated-sources/protobuf/`
- Universal `OoloiService` client/server classes used by all projects
- Supports distributed (client-server), combined (single-JVM), and in-process deployment modes

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
   - Protocol Buffer conversion utilities
   - Round-trip conversion symmetry for all data types

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

### Quick Start: What You Need for 90% of Your Code

**Whether you're developing frontend or backend, 90% of your code only needs this:**

```clojure
(ns my-namespace
  (:require [ooloi.shared.models.core :refer :all]))
```

This gives you access to:
- **All constructors**: `create-pitch`, `create-chord`, `create-measure`, etc.
- **All predicates**: `pitch?`, `chord?`, `measure?` (raw items), plus `pitch??`, `chord??`, `measure??` (timewalk tuples), etc.
- **All multimethods**: `get-duration`, `add-item`, `set-name`, etc.
- **Complete Ooloi system**: Everything you need for frontend AND backend development

**Why this works**: The shared `core` namespace IS the complete Ooloi system - frontend and backend both consume it identically.

**Architecture Reality**: Frontend and backend use SAME functions, SAME data structures, SAME everything from shared/. No conversion, no separate representations.

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

### Architecture Insights

**Unified Data Model Access**: Frontend and backend use IDENTICAL shared system:

```clojure
;; Frontend code (identical to backend)
[ooloi.shared.models.musical.pitch :refer [create-pitch]]     ; Same Pitch record
[ooloi.shared.interfaces :as i]                              ; Same API functions
[ooloi.shared.predicates :as p]                              ; Same predicates

;; Backend code (identical to frontend)  
[ooloi.shared.models.musical.pitch :refer [create-pitch]]     ; SAME Pitch record
[ooloi.shared.interfaces :as i]                              ; SAME API functions
[ooloi.shared.predicates :as p]                              ; SAME predicates

;; Result: (= frontend-pitch backend-pitch) => true
```

**Design Principle**: ONE data model, ONE API, used identically by frontend and backend. No conversion, no mapping, no separate representations - just the same shared objects everywhere.

**Complete System Access**: All Ooloi functionality in `shared/src` available to both frontend (local use + gRPC client) and backend (gRPC wrapper).

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

**gRPC Schema Changes** (extremely rare):
1. Edit `shared/src/main/proto/ooloi_service.proto` only if adding fundamental Clojure data types
2. Compile: `lein clean && lein compile`  
3. Test: `lein midje`

**Shared Model Changes** (normal development):
1. Edit models/traits/interfaces in `shared/src/main/clojure/`
2. Test: `lein midje` - changes automatically work across all projects
3. No compilation cascades needed - unified system handles everything

### gRPC Infrastructure Management

The shared project contains the **unified gRPC schema** used by both backend and frontend:

```bash
# Compile application (includes automatic gRPC infrastructure)
lein compile

# View generated gRPC infrastructure
ls target/generated-sources/protobuf/
```

#### Unified gRPC Schema

The shared project provides Ooloi's **unified gRPC architecture**:

- **`src/main/proto/ooloi_service.proto`** - Complete unified schema
  - **Universal OoloiValue**: Handles all Clojure data types automatically
  - **ExecuteMethod**: Single endpoint for all API methods via dynamic resolution
  - **ExecuteBatch**: Streaming operations for atomic STM transactions

#### Zero-Maintenance Architecture

**✅ No compilation needed for**:
- New API methods in backend
- New data models or shared records
- Plugin installations
- Shared model contract changes

**⚠️ Rare recompilation** only when `ooloi_service.proto` itself changes:
```bash
# Only if the unified schema structure is modified
lein clean && lein compile
```

**🎯 Development Reality**: Simple workflow:
```bash
# Make any changes - new models, API methods, plugins, etc.
lein midje  # Test your changes - no compilation cascades needed
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

Recommended development sequence:

1. **Make Changes**: Edit shared models, backend API methods, frontend code, etc.
2. **Test**: `lein midje` in relevant project - unified system handles everything automatically
3. **Integration**: All changes work immediately across projects via unified gRPC architecture

**Reality**: The unified system eliminates complex development workflows - just code and test!

## Notes

- Ensure all tests pass before creating the final package.
- The packaged application is platform-specific and ready for distribution.
- JavaFX is included in the project dependencies, ensuring GUI capabilities.

Remember to run tests (`lein midje`) before packaging to ensure everything is working correctly.
