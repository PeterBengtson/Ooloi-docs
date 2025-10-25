# Ooloi Shared

The shared project has a unique **dual nature** serving both as a common code library and as the integration testing/combined application platform. This dual architecture enables both code reuse and sophisticated deployment scenarios.

## Table of Contents

<img src="../img/shared-ooloi.png" alt="Ooloi Shared Architecture" align="right" width="500">

1. [Core Architecture](#core-architecture)
2. [System Architecture](#system-architecture)
3. [Architecture Roles](#architecture-roles)
4. [Directory Structure](#directory-structure)
5. [Prerequisites](#prerequisites)
6. [gRPC Infrastructure](#grpc-infrastructure)
7. [Installation](#installation)
8. [Building the Combined Application](#building-the-combined-application)
9. [Build Process Details](#build-process-details)
10. [Version Handling](#version-handling)
11. [Development](#development)
12. [Configuration and Deployment](#configuration-and-deployment)
13. [Development Commands](#development-commands)
14. [Notes](#notes)
15. [Related Documentation](#related-documentation)

## Core Architecture

The shared project is **the complete Ooloi data model and API**:

### **Primary Role: Complete Data Model**
Contains the entire Ooloi data model, business logic, and API - this IS Ooloi's core functionality.

### **Secondary Role: Network Transport & Combined Applications**
Provides gRPC infrastructure for network access and packages combined applications for deployment.

### **Project Relationship**
- **Shared**: The complete Ooloi system (data models, business logic, API)
- **Backend**: gRPC wrapper around shared (network transport layer)
- **Frontend**: Consumer of shared data model (local + remote via gRPC)

### **CRITICAL: Unified Data Representation**
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

KEY: Same data structures, same functions, same everything.
```

## Architecture Roles

See [ADR-0023: Shared Model Contracts](../ADRs/0023-Shared-Model-Contracts.md) for complete shared model architecture, unified system design, and multi-project integration details.

### Future Combined System Architecture

**Note:** The shared project will eventually include `system.clj` for combined deployment scenarios:

**Combined System Components (Planned):**
- **Backend Components**: piece-manager, grpc-server, http-server, cache-daemon (from backend project)
- **Frontend Components**: grpc-clients, ui-manager (from frontend project)
- **Startup Order**: Backend components initialize first, then frontend components connect

**Current Status:**
- Backend system.clj always starts all backend components (piece-manager, grpc-server, http-server, cache-daemon)
- Frontend system.clj always starts all frontend components (grpc-clients, ui-manager)
- Combined deployment will be implemented in shared/system.clj when frontend stabilizes
- Test helpers (with-server, with-clients) remain fully functional for component-level testing

**Dependency:** Combined system architecture depends on stable component definitions in both backend and frontend projects. The implementation awaits frontend component stabilization.

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
│       │   │   ├── grace.clj         ; Grace - ornamental notes
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

### Platform Setup

**Requirements**: Java 22+ and Leiningen 2.9.0+

#### macOS
```bash
# Install latest OpenJDK (22+), Leiningen, and Protocol Buffers
brew install openjdk leiningen protobuf

# Set JAVA_HOME (add to ~/.zshrc or ~/.bash_profile for persistence)
export JAVA_HOME="$(brew --prefix openjdk)"
export PATH="$JAVA_HOME/bin:$PATH"
```

#### Linux (Ubuntu/Debian)
```bash
sudo apt install openjdk-22-jdk openjfx protobuf-compiler
curl https://raw.githubusercontent.com/technomancy/leiningen/stable/bin/lein > ~/bin/lein
chmod +x ~/bin/lein
export JAVA_HOME="/usr/lib/jvm/java-22-openjdk-amd64"
```

#### Windows
- Install OpenJDK 22 from [Adoptium](https://adoptium.net/)
- Download `lein.bat` from [Leiningen](https://leiningen.org/) and run `lein self-install`
- Set `JAVA_HOME` environment variable

#### Dependencies
Protocol Buffers, JavaFX, and Skija are included in project dependencies.

#### Verification
```bash
java -version && lein version
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

**First-time setup requires Protocol Buffer compilation**:

```bash
cd shared
lein deps         # Install dependencies
lein protoc       # Compile Protocol Buffers (required once for gRPC)
```

The `lein protoc` step generates gRPC client/server stubs from `src/main/proto/ooloi_service.proto`. This is required once during initial setup, and rarely needed afterward (only if the `.proto` schema itself changes).

**Verification**:
```bash
lein midje        # Run tests to verify installation (takes a few minutes)
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

## Configuration and Deployment

**Note:** The combined application described here is the **typical end-user desktop application** - what musicians download and run on their computers. It works like any other desktop application (e.g., Sibelius, MuseScore) with all components in a single process. For distributed deployments (separate backend server + multiple frontend clients), use the backend and frontend projects directly.

The combined application supports configuration for development and advanced use cases:

### Command-Line Arguments

The combined application is a simple wrapper around backend and frontend components running in a single JVM:

```bash
# Basic combined application
lein run

# With custom configuration
lein run -- --port 10700 --ui-mode graphical
```

#### Available CLI Arguments

The combined application accepts the **union** of all backend and frontend CLI arguments:

**Backend Component Configuration:**

| **Argument** | **Values** | **Default** | **Description** |
|--------------|------------|-------------|-----------------|
| `--port PORT` | 1-65535 | 10700 | Backend gRPC server port |
| `--timeout-ms MS` | milliseconds | 5000 | Network timeout in milliseconds |
| `--tls FLAG` | true, false | false | Enable/disable TLS encryption (backend) |
| `--cert-path PATH` | file path | platform default | Path to server's public certificate |
| `--key-path PATH` | file path | platform default | Path to server's private key (backend only) |
| `--health-port PORT` | 1-65535 | 10701 | HTTP health endpoint port |

**Frontend Component Configuration:**

| **Argument** | **Values** | **Default** | **Description** |
|--------------|------------|-------------|-----------------|
| `--ui-mode MODE` | graphical, headless | graphical | User interface display mode |

**Note on Transport:** The combined application always uses in-process transport for maximum performance. Transport is not configurable - if you need network transport, run backend and frontend separately.

### Environment Variables

The combined application accepts the **union** of all backend and frontend environment variables:

**Backend Component Environment Variables:**

| **Environment Variable** | **CLI Equivalent** | **Default** | **Description** |
|-------------------------|-------------------|-------------|-----------------|
| `OOLOI_PORT` | --port | 10700 | Backend gRPC server port |
| `OOLOI_TIMEOUT_MS` | --timeout-ms | 5000 | Network timeout in milliseconds |
| `OOLOI_TLS` | --tls | false | Enable/disable TLS encryption (backend) |
| `OOLOI_CERT_PATH` | --cert-path | platform default | Path to server's public certificate |
| `OOLOI_KEY_PATH` | --key-path | platform default | Path to server's private key (backend only) |
| `OOLOI_HEALTH_PORT` | --health-port | 10701 | HTTP health endpoint port |

**Frontend Component Environment Variables:**

| **Environment Variable** | **CLI Equivalent** | **Default** | **Description** |
|-------------------------|-------------------|-------------|-----------------|
| `OOLOI_FRONTEND_UI_MODE` | --ui-mode | graphical | User interface display mode |

**Note on Transport:** The combined application always uses in-process transport. No transport-related environment variables are supported - if you need network transport, run backend and frontend separately.

**Configuration Precedence:** CLI arguments override environment variables, which override application defaults.

### TLS Configuration

The combined application supports secure TLS connections for internal client-server communication.

#### TLS Infrastructure Overview

The shared project provides the **complete TLS infrastructure** used by both backend (server) and frontend (client):

**Three-Tier Certificate Trust Strategy:**

1. **Priority 1: Insecure Development Mode** (`:insecure-dev-mode true`)
   - Bypasses ALL certificate validation (self-signed certificates)
   - Backend: Accepts any client without validation
   - Frontend: Accepts any server without validation
   - **Use for**: Development with self-signed certificates
   - **Security**: ⚠️  NOT FOR PRODUCTION

2. **Priority 2: Explicit Certificate** (`:cert-path` + `:key-path`)
   - Backend: Uses specified certificate/key for server identity
   - Frontend: Trusts specified certificate with proper validation
   - **Use for**: Custom CA or enterprise certificates
   - **Security**: ✅ Secure with proper validation

3. **Priority 3: System Trust Store** (default when TLS enabled)
   - Backend: Auto-generates self-signed certificate in `~/.ooloi/certs/`
   - Frontend: Uses Java's built-in trusted CAs (~150 authorities)
   - **Use for**: Production with Let's Encrypt or commercial CA
   - **Security**: ✅ Fully secure - standard PKI validation

#### Common Scenarios

**Combined Mode - No TLS Needed**

Combined mode always uses in-process transport with no network communication:

```bash
# Combined mode - in-process communication (always)
lein run
```

**No TLS overhead** - components communicate directly in same JVM via method calls, not network.

#### TLS Configuration Summary

| **Scenario** | **Transport** | **Backend TLS** | **Frontend TLS** | **Security** |
|--------------|---------------|-----------------|------------------|--------------|
| Combined (Always) | in-process | N/A | N/A | N/A (no network) |

#### TLS Implementation Details

**Server-Side (Backend):**
- **Certificate Generation**: Auto-generates self-signed certificates in `~/.ooloi/certs/` when TLS enabled
- **Certificate/Key Pair**: Requires both public certificate and private key
- **Platform Support**: macOS, Linux, Windows certificate storage
- **Bouncy Castle**: Uses Bouncy Castle library for certificate generation

**Client-Side (Frontend):**
- **Certificate Discovery**: Auto-discovers certificates from `~/.ooloi/certs/` when TLS enabled
- **One-Way TLS**: Clients verify server identity (server authentication only)
- **System Trust Store**: Uses Java's built-in ~150 trusted CAs for production
- **Custom Certificates**: Supports explicit certificate paths for enterprise deployments

**For detailed TLS scenarios, see:**
- Backend README: Server-side TLS configuration and certificate generation
- Frontend README: Client-side TLS configuration and certificate trust
- ADR-0020: TLS Infrastructure and Deployment Architecture

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

**No compilation needed for**:
- New API methods in backend
- New data models or shared records
- Plugin installations
- Shared model contract changes

**Rare recompilation** only when `ooloi_service.proto` itself changes:
```bash
# Only if the unified schema structure is modified
lein clean && lein compile
```

**Development Reality**: Simple workflow:
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

## Related Documentation

### Architecture Guides
- **[Polymorphic API Guide](/guides/POLYMORPHIC_API_GUIDE.md)** - Type system foundations underlying the shared model architecture
- **[Ooloi Server Architectural Guide](/guides/OOLOI_SERVER_ARCHITECTURAL_GUIDE.md)** - Server architecture built on shared model contracts
- **[gRPC Communication and Flow Control](/guides/GRPC_COMMUNICATION_AND_FLOW_CONTROL.md)** - How shared types enable network serialization

### Technical Documentation
- **[GitHub Project Board](https://github.com/users/PeterBengtson/projects/4)** - Current development status and implementation roadmap
- **[ADR-0023: Shared Model Contracts](/ADRs/0023-Shared-Model-Contracts.md)** - Multi-project architecture decisions
- **[gRPC Streaming & Threading Guide](/guides/GRPC_STREAMING_THREADING_GUIDE.md)** - Advanced streaming implementation patterns
