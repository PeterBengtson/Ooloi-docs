# Ooloi Shared

This directory contains the configuration and build scripts for creating a combined package of Ooloi, including both the backend server and frontend client.

## Purpose

The shared project serves to:

1. Hold code common to both the backend and the frontend.
2. Provide Protocol Buffer domain model and conversion utilities for gRPC communication.
3. Combine the backend and frontend components into a single, distributable application.

## Directory structure

```
shared/
├── src/main/proto/                    ; Protocol Buffer definitions
│   ├── ooloi_domain.proto            ; Complete Ooloi domain model (Musical + Visual hierarchies)
│   ├── ooloi_service.proto           ; gRPC service definitions (165 VPD methods)
│   └── vpd.proto                     ; VPD addressing structures
├── src/main/clojure/ooloi/shared/    ; Shared Clojure utilities
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

The shared project contains the complete Protocol Buffer infrastructure for gRPC communication between frontend and backend:

### Protocol Buffer Domain Model

- **`ooloi_domain.proto`**: Complete domain model covering all Ooloi data structures
  - Musical hierarchy: Piece → Musicians → Instruments → Staves → Voices → Measures → Items
  - Visual hierarchy: Layouts → PageViews → SystemViews → StaffViews → MeasureViews  
  - Attachment system: All 7 attachment types (Articulation, Dynamic, Slur, Tie, etc.)
  - Polymorphic Entity message with oneof for flexible type handling

- **`ooloi_service.proto`**: gRPC service definitions with 165 VPD-based methods
  - Generated from backend API namespace introspection
  - Request/response message pairs for all VPD-capable API methods
  - Consistent naming: kebab-case API methods → CamelCase protobuf messages

- **`vpd.proto`**: Vector Path Descriptor addressing structures for navigating musical hierarchies

### Conversion Utilities

- **`proto_conversion.clj`**: Comprehensive Clojure ↔ Protocol Buffer conversion
  - Context-aware conversion (VPD navigation vs musical data)
  - Support for all musical data types: ratios, keywords, numbers, strings, collections
  - Auto-detection of VPD collections based on first element
  - Protocol-based extensible design for future data types

- **`vpd_utils.clj`**: VPD manipulation and protobuf string conversion
  - VPD ↔ protobuf string array conversion
  - Compact/verbose VPD form handling
  - Empty VPD handling ([] represents whole piece)

### Build Integration

The Protocol Buffer files are automatically compiled during the build process:
- `lein protoc` generates Java classes in `target/generated-sources/protobuf/`
- Generated classes are identical across all projects (shared, backend, frontend)
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
- **Build utilities** - Cross-project packaging and deployment tools

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

The shared project defines the complete gRPC protocol:

- **`src/main/proto/ooloi_domain.proto`** - Complete Ooloi domain model (85+ message types)
  - Musical hierarchy: Piece→Musicians→Instruments→Staves→Voices→Measures→Items
  - Visual hierarchy: Layouts→PageViews→SystemViews→StaffViews→MeasureViews
  - Attachment system: Cross-reference resolution and shared musical elements

- **`src/main/proto/ooloi_service.proto`** - gRPC service definitions (165 VPD method definitions)
  - All API methods from `ooloi.backend.api` with proper Request/Response pairs
  - Categorized by operation type: getters, setters, complex operations

- **`src/main/proto/vpd.proto`** - VPD addressing structures
  - Vector Path Descriptor message definitions for navigation

#### When to Regenerate Protocol Buffers

**⚠️ Critical**: Protocol Buffers must be regenerated after API changes:

**Always regenerate after adding:**
- New multimethod definitions in `backend/src/main/clojure/ooloi/backend/models/interfaces.clj`
- New VPD-dispatching methods in `backend/src/main/clojure/ooloi/backend/models/core.clj`
- New functions exported via `backend/src/main/clojure/ooloi/backend/api.clj`
- New message types or service methods in `.proto` files

**Regeneration workflow** (must be done in correct order):

```bash
# 1. In shared/ directory - regenerate source definitions
lein clean && lein compile

# 2. In backend/ directory - regenerate server stubs  
cd ../backend && lein clean && lein compile

# 3. In frontend/ directory - regenerate client stubs
cd ../frontend && lein clean && lein compile

# 4. Verify all projects compile
cd ../backend && lein midje  # Test backend
cd ../frontend && lein midje # Test frontend  
cd ../shared && lein midje   # Test shared
```

**Signs you forgot to regenerate:**
- `NoSuchMethodError` in gRPC tests
- Missing service methods in generated stubs
- New API methods not accessible via gRPC
- Protocol buffer compilation errors

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
