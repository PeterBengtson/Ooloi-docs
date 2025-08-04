# Ooloi Backend

This directory contains the backend server code for Ooloi, a high-performance music notation software.

## Directory structure

```
backend/
├── docs/              ; The HTML documentation for the backend, created by Codox
├── resources/         ; Resources for the application, notably icons
├── src/               ; The source code hierarchy for the backend
├── test/              ; The backend Midje tests
└── CHANGELOG.md       ; The CHANGELOG (as of yet unused)
└── README.md          ; This README
└── project.clj        ; The Leiningen project file for the backend
```

## Prerequisites

### System Requirements

- **Java Development Kit (JDK) 22 or later** - Required for running Clojure and building the application
- **Leiningen 2.9.0 or later** - Clojure build tool for dependency management and project operations
- **Git** - For version control and accessing the repository
- **Minimum 4GB RAM** - Recommended for development and large musical scores
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
# Install Java
sudo apt update
sudo apt install openjdk-22-jdk

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

Ensure you have the appropriate icon files in the `backend/resources/icons/` directory:

- **macOS**: `icon.icns`
- **Windows**: `icon.ico` 
- **Linux**: `icon.png`

These can be found in the root `icons/` directory and should be symlinked:
```bash
cd backend/resources/
ln -s ../../icons/ready/macos/icon.icns icons/
ln -s ../../icons/ready/windows/icon.ico icons/
ln -s ../../icons/ready/linux/icon.png icons/
```

## Installation

  ```bash
  cd backend
  lein deps
  ```

## Building the Backend

1. Navigate to the backend directory:
   ```bash
   cd backend
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
   - Run the build function to package the application

   You will see colored output indicating the progress of each step.

4. The packaged backend application will be in the `backend/target` directory.

## Build Process Details

The build process uses the following tools and steps:

1. **Cleaning**: Removes previous build artifacts.
2. **Uberjar Creation**: Compiles the code and creates a standalone jar file.
3. **Jpackage**: Packages the application for distribution, creating platform-specific installers or application images.

The build process handles both SNAPSHOT (development) versions and release versions:

- For SNAPSHOT versions, it creates an app image.
- For release versions, it creates platform-specific installers (DMG for macOS, DEB for Linux, MSI for Windows).

## Version Handling

The build process automatically adjusts version numbers for compatibility with different platforms:

- SNAPSHOT suffixes are removed for the final package version.
- If the version starts with "0", it's changed to "1.0.0" for macOS compatibility.
- The original version (including SNAPSHOT if applicable) is included in the application name.

## Development

### Running the Backend

#### Quick Start
```bash
lein run
```

The application will display startup messages:
```
Starting Ooloi backend server...
✅ All components started successfully
🎵 Ooloi backend ready for musical collaboration
```

#### Configuration Options

The backend supports flexible configuration through command-line arguments and environment variables:

**Command-Line Arguments** (recommended for development):
```bash
# Custom port
lein run -- --port 8080

# Deployment mode
lein run -- --deployment-mode combined

# TLS configuration (ADR-0020)
lein run -- --tls true
lein run -- --tls true --cert-path ./custom.crt --key-path ./custom.key

# gRPC transport optimization (ADR-0019)
lein run -- --grpc-transport in-process --health-port 10701

# Multiple options
lein run -- --port 8080 --deployment-mode backend --timeout-ms 3000 --grpc-transport network --tls true
```

**Environment Variables** (recommended for production):
```bash
export OOLOI_PORT=8080
export OOLOI_DEPLOYMENT_MODE=combined  
export OOLOI_TIMEOUT_MS=3000
export OOLOI_GRPC_TRANSPORT=in-process
export OOLOI_HEALTH_PORT=10701
# TLS configuration
export OOLOI_TLS=true
export OOLOI_CERT_PATH=/etc/ssl/ooloi.crt
export OOLOI_KEY_PATH=/etc/ssl/ooloi.key
lein run
```

#### Available Configuration

| Parameter | CLI Flag | Environment Variable | Default | Description |
|-----------|----------|---------------------|---------|-------------|
| **Port** | `--port 8080` | `OOLOI_PORT` | 10700 | gRPC server port for frontend communication |
| **Deployment Mode** | `--deployment-mode MODE` | `OOLOI_DEPLOYMENT_MODE` | backend | System deployment configuration |
| **Timeout** | `--timeout-ms 3000` | `OOLOI_TIMEOUT_MS` | 5000 | Network timeout in milliseconds |
| **TLS Enabled** | `--tls true/false` | `OOLOI_TLS` | false | Enable/disable TLS encryption |
| **TLS Certificate** | `--cert-path PATH` | `OOLOI_CERT_PATH` | platform default | Path to TLS certificate file (created if missing) |
| **TLS Private Key** | `--key-path PATH` | `OOLOI_KEY_PATH` | platform default | Path to TLS private key file (created if missing) |
| **gRPC Transport** | `--grpc-transport TYPE` | `OOLOI_GRPC_TRANSPORT` | auto | Transport optimization: `network` or `in-process` |
| **Health Port** | `--health-port 10701` | `OOLOI_HEALTH_PORT` | 10701 | HTTP health endpoint port for monitoring |

**Deployment Modes**:
- **`backend`** (default): Piece manager + gRPC server - typical server deployment
- **`combined`**: All components including UI support - single-process deployment  
- **`dev-engine-only`**: Only piece manager - minimal development/testing mode

**gRPC Transport Optimization** (ADR-0019):
- **`auto`** (default): Automatic selection - `in-process` for combined mode, `network` for backend mode
- **`in-process`**: Ultra-high-performance direct communication (37.5-75x faster, 98.7-99.3% latency reduction, for combined deployments)
- **`network`**: Standard TCP communication (for debugging or separate process deployment)

**Health Monitoring**:
- **Health Port**: HTTP endpoint for external monitoring tools (load balancers, ops dashboards)
- **gRPC Health**: Built-in gRPC health service for component coordination

**Configuration Precedence**: Command-line arguments override environment variables, which override defaults.

#### TLS Configuration and Test Certificates

**TLS Overview** (ADR-0020: TLS Infrastructure and Deployment Architecture):
- **Default**: TLS disabled for immediate developer productivity
- **Development**: Enable with `--tls true` or `OOLOI_TLS=true` - certificates auto-generated
- **Certificate Management**: Intelligent creation at specified paths or platform defaults
- **Production**: Full control via `--cert-path` and `--key-path` parameters

**Certificate Auto-Generation**:
Ooloi automatically generates TLS certificates when needed:
- **Implementation**: Pure Java with Bouncy Castle (cross-platform, no external dependencies)
- **Default Locations**: 
  - Unix/macOS: `~/.ooloi/certs/server.{crt,key}`
  - Windows: `%APPDATA%\Ooloi\certs\server.{crt,key}`
- **Properties**: RSA 2048-bit, 20-year validity, covers `localhost`, `127.0.0.1`, `::1`
- **Behavior**: Certificates created on first TLS startup, reused on subsequent starts

**TLS Configuration Examples**:
```bash
# Development: TLS disabled (default)
lein run

# Development: TLS with auto-generated certificates (platform defaults)
lein run -- --tls true
OOLOI_TLS=true lein run

# Development: TLS with certificates at custom locations
lein run -- --tls true --cert-path ./my.crt --key-path ./my.key

# Production: Environment variables with custom certificates
export OOLOI_TLS=true
export OOLOI_CERT_PATH=/etc/ssl/ooloi.crt
export OOLOI_KEY_PATH=/etc/ssl/ooloi.key
lein run

# Production: HTTPS port with TLS
export OOLOI_TLS=true
export OOLOI_PORT=443
export OOLOI_CERT_PATH=/etc/ssl/ooloi.crt  
export OOLOI_KEY_PATH=/etc/ssl/ooloi.key
java -jar target/ooloi-backend-*-standalone.jar
```

**Certificate Management**:
- **Automatic**: Certificates are generated automatically when TLS is enabled
- **Persistent**: Once created, certificates are reused across server restarts
- **Configurable**: Use `--cert-path` and `--key-path` to control certificate locations
- **Cross-platform**: Works seamlessly on Unix, macOS, and Windows

#### Production Deployment

For production deployment, use environment variables and proper process management:

```bash
# Set environment variables
export OOLOI_PORT=10700
export OOLOI_DEPLOYMENT_MODE=backend
export OOLOI_TIMEOUT_MS=5000

# Run with process management (e.g., systemd, supervisor, docker)
java -jar target/ooloi-backend-*-standalone.jar
```

#### Error Handling and Troubleshooting

The backend provides comprehensive error handling with actionable guidance:

**Port Conflicts**:
```bash
❌ Failed to start Ooloi backend:
Error: gRPC server failed to start on port 10700
💡 Suggestion: Is another instance already running? Try a different port or stop the conflicting service.
```

**Configuration Issues**:
```bash
❌ Failed to start Ooloi backend:
Error: Missing required configuration: piece-manager dependency
💡 Suggestion: Check your configuration files and environment variables.
```

**Exit Codes for Operational Monitoring**:
- **0**: Successful startup and operation
- **10**: Component initialization failure (check logs and dependencies)
- **11**: Port binding failure (port already in use)
- **12**: Configuration error (invalid or missing configuration)
- **13**: Missing dependency (required services unavailable)
- **14**: Resource exhaustion (insufficient system resources)
- **1**: Generic failure (check application logs)

#### Development vs Production

**Development Mode**:
- Use `lein run` with command-line arguments
- Automatic code reloading available
- Debug logging enabled by default

**Production Mode**:
- Use standalone JAR with environment variables
- Optimized performance settings
- Structured logging for monitoring

#### Health Monitoring

The backend includes built-in health monitoring accessible via gRPC:
- Component status tracking
- Resource usage monitoring  
- Automatic failure detection and isolation
- Graceful shutdown with proper cleanup

## Development Commands

### Running Tests

The backend uses Midje for comprehensive testing:

```bash
# Run all tests
lein midje

# Run specific test namespace
lein midje ooloi.backend.ops.timewalk-test

# Run tests with coverage report
lein coverage
```

**Important**: Use `lein midje` instead of `lein test`. The project is configured for Midje testing framework.

### Protocol Buffer Generation

The backend includes gRPC infrastructure with automatic Protocol Buffer compilation:

```bash
# Compile Protocol Buffers and generate Java classes
lein compile

# Clean and recompile (recommended when .proto files change)  
lein clean && lein compile

# Verify generated classes
ls target/generated-sources/protobuf/
```

#### Protocol Buffer Configuration

The backend automatically generates Java classes from shared Protocol Buffer definitions:

- **Proto source**: `../shared/src/main/proto/` (shared with frontend)
- **Generated output**: `target/generated-sources/protobuf/`
- **Compiler version**: protoc 3.25.3 (automatically downloaded)
- **Generated files**: 
  - `ooloi_domain.proto` → Domain model Java classes
  - `ooloi_service.proto` → gRPC service stubs
  - `vpd.proto` → VPD utility classes

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
4. **Run**: `lein run` (start backend server)
5. **Iterate**: Make changes and repeat steps 2-4

## Notes

- Ensure all tests pass before creating the final package.
- The packaged application is platform-specific and ready for distribution.
- JavaFX is included in the project dependencies, ensuring GUI capabilities.

Remember to run tests (`lein midje`) before packaging to ensure everything is working correctly.
