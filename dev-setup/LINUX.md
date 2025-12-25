# Linux Development Setup for Ooloi

**Status**: Untested ⚠️ - Awaiting validation

Complete guide for setting up Ooloi development environment on Linux. These instructions are based on the macOS and Windows tested setups but have not yet been validated on actual Linux systems.

## Overview

Setting up Ooloi on Linux uses your distribution's package manager. This guide covers common distributions (Ubuntu/Debian, Fedora/RHEL, Arch) and the correct sequence for building Ooloi's three-project structure.

## Prerequisites

- Modern Linux distribution (Ubuntu 20.04+, Fedora 35+, Arch, etc.)
- Sudo access for package installation

## Step 1: Install Java (OpenJDK 25+)

### Ubuntu/Debian

```bash
sudo apt update
sudo apt install openjdk-25-jdk openjfx protobuf-compiler git
```

### Fedora/RHEL

```bash
sudo dnf install java-25-openjdk java-25-openjdk-devel protobuf-compiler git
```

### Arch Linux

```bash
sudo pacman -S jdk25-openjdk protobuf git
```

### Configure JAVA_HOME

Add to your shell configuration file (`~/.bashrc`, `~/.zshrc`, or `~/.profile`):

```bash
# Set JAVA_HOME (adjust path for your distribution)
export JAVA_HOME="/usr/lib/jvm/java-25-openjdk-amd64"    # Ubuntu/Debian
# export JAVA_HOME="/usr/lib/jvm/java-25-openjdk"        # Fedora/Arch
export PATH="$JAVA_HOME/bin:$PATH"
```

**Apply changes**:
```bash
source ~/.bashrc    # or ~/.zshrc, ~/.profile
```

### Verify Installation

```bash
java -version
# Should show: openjdk version "25" or higher

protoc --version
# Should show: libprotoc 3.x.x or higher
```

## Step 2: Install Leiningen

Leiningen is not typically available in distribution repositories, so install it manually:

```bash
# Download lein script
curl https://raw.githubusercontent.com/technomancy/leiningen/stable/bin/lein > ~/bin/lein

# Make it executable
chmod +x ~/bin/lein

# Ensure ~/bin is in PATH (add to ~/.bashrc if needed)
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Self-install Leiningen
lein self-install
```

### Verify Installation

```bash
lein version
# Should show: Leiningen 2.x.x on Java 25 OpenJDK 64-Bit Server VM
```

## Step 3: Clone and Build Ooloi

Ooloi consists of three separate Leiningen projects that must be built in sequence:

```bash
git clone https://github.com/PeterBengtson/Ooloi.git
cd Ooloi
```

### Install Dependencies for Each Project

**Critical**: Follow this exact sequence:

```bash
# 1. Shared project (MUST BE FIRST)
cd shared
lein deps
lein protoc         # Compiles Protocol Buffers (required for backend/frontend)

# 2. Backend project
cd ../backend
lein deps

# 3. Frontend project
cd ../frontend
lein deps
cd ..
```

**Why this order?** The `lein protoc` command in `shared/` generates gRPC protocol buffer definitions required by both `backend/` and `frontend/`. Skip this and the other projects will fail to compile.

### Run Tests

Verify everything works by testing each project (run sequentially to avoid port conflicts):

```bash
# Run tests sequentially
cd shared
lein midje          # Takes a few minutes

cd ../backend
lein midje

cd ../frontend
lein midje

cd ..
```

All tests should pass on a clean Linux installation.

## Common Issues (Anticipated)

### JAVA_HOME Not Set

**Symptom**: Java commands work but tools complain about missing JAVA_HOME.

**Solution**: Find your Java installation and set JAVA_HOME:
```bash
# Find Java installation
update-alternatives --list java    # Ubuntu/Debian
ls /usr/lib/jvm/                  # Check available Java versions

# Add to ~/.bashrc
echo 'export JAVA_HOME="/usr/lib/jvm/java-25-openjdk-amd64"' >> ~/.bashrc
source ~/.bashrc
```

### Port Conflicts During Tests

**Symptom**: Tests fail with "address already in use" errors.

**Solution**: Run test suites **sequentially** (one project at a time), not in parallel terminal tabs. Tests use ports 10700-10701 which will conflict if multiple test suites run simultaneously.

### Leiningen Self-Install Fails

**Symptom**: `lein` fails to download dependencies.

**Solution**: Ensure network connectivity and try:
```bash
# Clear Leiningen cache and retry
rm -rf ~/.lein
lein self-install
```

### Missing JavaFX on Headless Systems

**Symptom**: Frontend tests fail with JavaFX errors.

**Solution**: Install JavaFX (OpenJFX):
```bash
# Ubuntu/Debian
sudo apt install openjfx

# Fedora
sudo dnf install openjfx

# Arch
sudo pacman -S java-openjfx
```

### Graphics Acceleration Issues

**Symptom**: Tests or rendering performance is poor.

**Solution**: Ensure graphics drivers are properly installed. For development VMs or containers, JavaFX may run in software rendering mode (slower but functional).

### File Permissions

**Symptom**: Permission denied errors when running lein or tests.

**Solution**:
```bash
# Ensure lein is executable
chmod +x ~/bin/lein

# Ensure proper ownership of project files
cd ~/Ooloi
sudo chown -R $USER:$USER .
```

## Development Tools

### REPL Development

All three projects support REPL-driven development:

```bash
# Start REPL in any project
cd shared    # or backend, or frontend
lein repl
```

The REPL loads the project namespace and allows interactive development.

## Platform Testing

Ooloi targets three platforms: macOS, Windows, and Linux. When developing on Linux, be aware of platform-specific issues:

- **File paths**: Linux uses `/` separators (like macOS)
- **Case sensitivity**: Linux filesystems are case-sensitive (unlike Windows/macOS default)
- **Graphics rendering**: JavaFX/Skija behavior may differ from Windows/macOS
- **Font rendering**: Quality varies by distribution and font configuration
- **Native libraries**: Linux uses .so files vs .dylib (macOS) or .dll (Windows)
- **Distribution variations**: Different package managers and paths across distros

## Docker Alternative

For consistent development environments across systems, use Docker to test these instructions without a native Linux installation.

**Create Dockerfile** in the Ooloi repository root:

```dockerfile
# Dockerfile for testing Ooloi Linux installation
FROM eclipse-temurin:25-jdk

RUN apt-get update && apt-get install -y \
    git \
    protobuf-compiler \
    openjfx \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Install Leiningen
RUN curl https://raw.githubusercontent.com/technomancy/leiningen/stable/bin/lein > /usr/local/bin/lein && \
    chmod +x /usr/local/bin/lein && \
    lein version

WORKDIR /workspace
CMD ["/bin/bash"]
```

**Build and run**:

```bash
# Build the image (run from Ooloi repository root)
docker build -t ooloi-linux-test .

# Start container with Ooloi source mounted
docker run -it --rm -v $(pwd):/workspace ooloi-linux-test bash

# Inside container, follow Step 3 onwards from this guide
# (Java, protobuf, and Leiningen are already installed)
cd shared
lein deps
lein protoc
# ... continue with build sequence
```

## Next Steps

After successful installation and test verification:

1. Return to [main setup guide](README.md) for development workflow
2. See [main README](../README.md) for project overview
3. Check [ADRs](../ADRs/) for architecture documentation
4. Review [guides](../guides/) for technical details

## Validation Needed

This guide has not yet been validated on actual Linux systems. If you're setting up Ooloi on Linux:

1. **Please report success or failures** via GitHub Issues
2. Note your distribution and version
3. Document any differences or additional steps needed
4. Help us improve this guide with your feedback

Your contributions will help other Linux developers!

## Getting Help

If you encounter Linux-specific issues:

1. Check [GitHub Issues](https://github.com/PeterBengtson/Ooloi/issues) for similar problems
2. Verify all dependencies are installed for your distribution
3. Ensure JAVA_HOME and PATH are set correctly
