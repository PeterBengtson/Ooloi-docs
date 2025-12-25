# Ooloi Development Setup

Complete guide for setting up Ooloi development environment on macOS, Windows, and Linux.

## Overview

Ooloi is a multi-project Clojure system consisting of three interconnected components that must be set up in a specific order:

1. **shared/** - Core models, gRPC infrastructure (must be set up first)
2. **backend/** - Server component (depends on shared)
3. **frontend/** - Client component (depends on shared)

**Important**: This guide is for **developers** setting up a development environment. End users will install pre-built packages.

## Quick Navigation

Choose your platform for detailed installation instructions:

- **[macOS Setup](MACOS.md)** - Tested ✅
- **[Windows Setup](WINDOWS.md)** - Tested ✅
- **[Linux Setup](LINUX.md)** - Tested ✅ (Ubuntu/Debian via Docker)

## Prerequisites Summary

All platforms require:

- **Java 25+** (OpenJDK recommended)
- **Leiningen 2.9.0+** (Clojure build tool)
- **Git** (for cloning the repository)

## The Critical Sequence

Ooloi's three projects **must be set up in this exact order**:

### Step 1: Shared Project (Required First)

```bash
cd shared
lein deps           # Install dependencies
lein protoc         # Compile Protocol Buffers (generates gRPC stubs)
lein midje          # Verify installation (takes a few minutes)
```

**Why first?** The `lein protoc` command compiles Protocol Buffer definitions that both backend and frontend depend on. If you skip this, backend and frontend will fail to compile.

**Generated files**: Protocol Buffer compilation creates Java classes in `shared/target/generated-sources/protobuf/` that backend and frontend projects import.

### Step 2: Backend Project

```bash
cd ../backend
lein deps           # Install dependencies
lein midje          # Verify installation
```

**Dependencies**: Backend imports the gRPC stubs generated in Step 1.

### Step 3: Frontend Project

```bash
cd ../frontend
lein deps           # Install dependencies
lein midje          # Verify installation
```

**Dependencies**: Frontend imports the gRPC stubs generated in Step 1.

## Verification

After completing all three steps, verify the complete system:

```bash
# Run all test suites (must be run sequentially to avoid port conflicts)
cd /path/to/Ooloi/shared && lein midje
cd /path/to/Ooloi/backend && lein midje
cd /path/to/Ooloi/frontend && lein midje
```

All test suites must pass for a successful installation.

## Common Issues

### "Protocol Buffer classes not found"

**Symptom**: Backend or frontend compilation fails with missing gRPC classes.

**Solution**: Run `lein protoc` in `shared/` directory first. This generates required classes.

### Port conflicts during tests

**Symptom**: Tests fail with "address already in use" errors.

**Solution**: Run test suites **sequentially** (one at a time), not in parallel. Tests use ports 10700-10701 which can conflict if run simultaneously.

### PATH not updated

**Symptom**: `java` or `lein` commands not found after installation.

**Solution**: Close and reopen **all** terminal windows and your IDE completely. Environment variables only take effect in new processes. See platform-specific guides for PATH configuration.

### lein deps hangs or fails

**Symptom**: Dependency download stalls or times out.

**Solution**:
- Check network connectivity
- Antivirus software may block Leiningen downloads - whitelist `~/.lein/` directory
- On slow connections, increase timeout: `export LEIN_HTTP_TIMEOUT=60000`

## What's Different About Each Project?

### Shared Project
- **Runs `lein protoc`** - generates gRPC infrastructure
- **Contains all data models** - the complete Ooloi system
- **Core functionality tests** - data models, operations, and gRPC infrastructure

### Backend Project
- **Provides gRPC server** - network wrapper around shared models
- **STM-based piece management** - concurrent musical data storage
- **Server functionality tests** - server components and integration

### Frontend Project
- **Provides UI** - JavaFX + Skija rendering
- **gRPC client** - connects to backend server
- **Client functionality tests** - UI and client components

## Development Workflow

After initial setup, typical development workflow:

```bash
# Make changes to any project
# Run tests in that project
cd shared && lein midje          # Test shared changes
cd backend && lein midje         # Test backend changes
cd frontend && lein midje        # Test frontend changes

# If you modified shared models or interfaces,
# test integration across all projects
cd shared && lein midje && cd ../backend && lein midje && cd ../frontend && lein midje
```

**Note**: You rarely need to run `lein protoc` again - only if the Protocol Buffer schema itself (`shared/src/main/proto/ooloi_service.proto`) changes, which is extremely rare.

## Next Steps

After successful installation:

1. See main [README.md](../README.md) for project overview and architecture
2. See [shared/README.md](../shared/README.md), [backend/README.md](../backend/README.md), [frontend/README.md](../frontend/README.md) for project-specific development
3. Check [ADRs/](../ADRs/) for architectural decisions and design rationale
4. Review [guides/](../guides/) for detailed technical documentation

## Getting Help

If you encounter issues not covered here:

1. Check the [GitHub Issues](https://github.com/PeterBengtson/Ooloi/issues) for similar problems
2. Review platform-specific guides for detailed troubleshooting

## Platform-Specific Notes

Each platform has unique considerations:

- **macOS**: Homebrew makes installation straightforward, but JAVA_HOME must be set correctly
- **Windows**: PATH configuration often requires manual intervention - installers frequently fail to update environment variables
- **Linux**: Package manager variations (apt vs yum vs pacman) require different commands

See platform-specific guides for detailed instructions.
