# Ooloi Frontend

This directory contains the frontend client code for Ooloi, a high-performance music notation software.

## Project Role

The **Ooloi Frontend** serves as the user-facing client component providing:

1. **User Interface**: JavaFX-based graphical interface for music notation editing and visualization
2. **Score Rendering**: High-performance visual display of musical structures and notation
3. **gRPC Client Integration**: Communication with backend via protocol buffer-based unified interface
4. **User Interaction Management**: Click handling, form validation, real-time UI updates, and client-side state management

**Key Architectural Responsibility**: Implements presentation layer functionality and user experience while communicating with backend server through gRPC for all musical data operations.

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

## Development

### Running the Frontend

To run the frontend client for development:

```bash
lein run
```

This will start the Ooloi frontend client.

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

**Frontend Test Coverage**: 20 passing tests including:
- **gRPC Client Infrastructure**: Protocol buffer message creation and gRPC client class instantiation
- **Shared Model Integration**: Tests for frontend-accessible shared functionality with mock backend support
- **UI Component Testing**: Form validation, user interaction handling, and component state management  
- **Client-side Functionality**: Connection management, error handling, and client-side data processing
- **Mock Backend Integration**: Verification that shared traits and interfaces work with stubbed backend dependencies

**Total: 20 tests** focused on frontend presentation layer, user experience, and client-side gRPC communication setup.

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
