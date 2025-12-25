# macOS Development Setup for Ooloi

**Status**: Tested ✅

Complete guide for setting up Ooloi development environment on macOS.

## Overview

Setting up Ooloi on macOS is straightforward using Homebrew. This guide covers installation of all prerequisites and the correct sequence for building Ooloi's three-project structure.

## Prerequisites

- macOS 10.15 (Catalina) or later
- Homebrew package manager
- Xcode Command Line Tools (installed automatically by Homebrew)

## Step 1: Install Homebrew (if not already installed)

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Follow the post-installation instructions to add Homebrew to your PATH.

## Step 2: Install Java (OpenJDK 25+) and Leiningen

```bash
# Install OpenJDK 25+, Leiningen, and Protocol Buffers compiler
brew install openjdk leiningen protobuf
```

### Configure JAVA_HOME

Add to your shell configuration file (`~/.zshrc` for zsh or `~/.bash_profile` for bash):

```bash
# Set JAVA_HOME to Homebrew-installed OpenJDK
export JAVA_HOME="$(brew --prefix openjdk)"
export PATH="$JAVA_HOME/bin:$PATH"
```

**Apply changes**:
```bash
# For zsh (default on modern macOS)
source ~/.zshrc

# For bash
source ~/.bash_profile
```

### Verify Installation

```bash
java -version
# Should show: openjdk version "25" or higher

lein version
# Should show: Leiningen 2.x.x on Java 25 OpenJDK 64-Bit Server VM

protoc --version
# Should show: libprotoc 3.x.x or higher
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

All tests should pass on a clean macOS installation.

## Common Issues

### Homebrew Not in PATH

**Symptom**: `brew: command not found`

**Solution**: Add Homebrew to PATH. On Apple Silicon Macs:
```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zshrc
source ~/.zshrc
```

On Intel Macs:
```bash
echo 'eval "$(/usr/local/bin/brew shellenv)"' >> ~/.zshrc
source ~/.zshrc
```

### JAVA_HOME Not Set

**Symptom**: Java commands work but tools complain about missing JAVA_HOME.

**Solution**: Ensure JAVA_HOME is set in your shell configuration:
```bash
# Find Java location
brew --prefix openjdk

# Add to ~/.zshrc
echo 'export JAVA_HOME="$(brew --prefix openjdk)"' >> ~/.zshrc
echo 'export PATH="$JAVA_HOME/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
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

### M1/M2/M3 (Apple Silicon) Considerations

Ooloi runs natively on Apple Silicon. All dependencies support ARM64:
- OpenJDK has native Apple Silicon builds
- Leiningen runs on any JVM
- All Clojure dependencies are platform-independent

**Note**: If you encounter "bad CPU type" errors, ensure you're using native ARM64 Java, not Rosetta-translated x86_64 Java.

### Permission Denied on lein

**Symptom**: `lein: Permission denied`

**Solution**: Ensure lein script is executable:
```bash
chmod +x $(which lein)
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

Ooloi targets three platforms: macOS, Windows, and Linux. When developing on macOS, be aware of platform-specific issues:

- **File paths**: macOS uses `/` separators (like Linux)
- **Case sensitivity**: By default macOS filesystems are case-insensitive
- **Graphics rendering**: JavaFX/Skija behavior may differ from Windows/Linux
- **Font rendering**: macOS has excellent font rendering (often looks best here)
- **Native libraries**: macOS-specific .dylib files vs .so (Linux) or .dll (Windows)

## Next Steps

After successful installation and test verification:

1. Return to [main setup guide](README.md) for development workflow
2. See [main README](../README.md) for project overview
3. Check [ADRs](../ADRs/) for architecture documentation
4. Review [guides](../guides/) for technical details

## Getting Help

If you encounter macOS-specific issues:

1. Check [GitHub Issues](https://github.com/PeterBengtson/Ooloi/issues) for similar problems
2. Verify Homebrew is up to date: `brew update && brew upgrade`
3. Ensure JAVA_HOME and PATH are set correctly
