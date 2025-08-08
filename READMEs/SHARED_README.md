# Ooloi Shared

This directory contains the configuration and build scripts for creating a combined package of Ooloi, including both the backend server and frontend client.

## Purpose

The shared project serves three critical architectural roles in Ooloi:

### 1. **Shared Model Contracts** 
Contains all data model definitions (`defrecord` structures), interfaces, predicates, and traits that both frontend and backend use, ensuring perfect type fidelity and eliminating circular dependencies.

**Key Components**:
- **Core Data Models**: All musical and visual model `defrecord` definitions
- **Interface Contracts**: Shared multimethod definitions preventing API drift
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
│   ├── models/                       ; **Phase 2.6**: Complete shared model contracts
│   │   ├── musical/                  ; Musical data models (Piece, Musician, Instrument, etc.)  
│   │   ├── visual/                   ; Visual models (Layout, PageView, StaffView, etc.)
│   │   └── changes.clj               ; ChangeSet data structure for time sigs, key sigs, tempos
│   ├── traits/                       ; Shared trait definitions (rhythmic-item, takes-attachment)
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

This will start the Ooloi application.

### REPL

For interactive development, you can start a REPL:

```bash
lein repl
```

The REPL will start in the `ooloi.shared.mon.core` namespace.

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
- **Shared model contracts** (1,587 tests) - All shared model functionality and generator correctness
- **Build utilities** - Cross-project packaging and deployment tools

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
