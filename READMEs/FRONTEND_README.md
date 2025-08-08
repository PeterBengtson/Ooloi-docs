# Ooloi Frontend

This directory contains the frontend server code for Ooloi, a high-performance music notation software.

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

## Building the Backend

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

**Current Status**: Frontend has access to shared model contracts from the backend. Ready for Phase 3 gRPC client integration to enable frontend-backend communication.

## Notes

- Ensure all tests pass before creating the final package.
- The packaged application is platform-specific and ready for distribution.
- JavaFX is included in the project dependencies, ensuring GUI capabilities.

Remember to run tests (`lein midje`) before packaging to ensure everything is working correctly.
