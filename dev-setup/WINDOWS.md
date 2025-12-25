# Windows Development Setup for Ooloi

**Status**: Tested ✅ (December 2024)

Complete guide for setting up Ooloi development environment on Windows 10 or later.

## Overview

Setting up Ooloi on Windows requires Java 25+ and Leiningen. The main complication is that Windows installers often fail to set PATH environment variables correctly, requiring manual configuration.

## Prerequisites

- Windows 10 or later
- Administrator access (for some installation methods)
- Git for Windows (recommended)

## Step 1: Install Java (OpenJDK 25+)

### Installation

**Using winget (recommended)**:
```powershell
winget install EclipseAdoptium.Temurin.25.JDK
```

**Or download manually** from [Adoptium](https://adoptium.net/temurin/releases/) and install the MSI.

### Verify Installation

After installing, verify Java is accessible:
```powershell
java -version
```

If you get `'java' is not recognized`, the installer failed to set PATH correctly. Continue to PATH configuration below.

### Configure JAVA_HOME and PATH

**Method 1: GUI (Recommended for most users)**

1. Press `Win+R`, type `rundll32 sysdm.cpl,EditEnvironmentVariables`, press Enter
2. In the **top section** (User variables):
   - Click **New**
   - Variable name: `JAVA_HOME`
   - Variable value: `C:\Program Files\Eclipse Adoptium\jdk-25.0.x.x-hotspot` (verify actual path)
   - Click OK
3. Still in top section, find **Path** variable:
   - Click **Path**, then **Edit**
   - Click **New**
   - Add: `%JAVA_HOME%\bin`
   - Click OK on all dialogs
4. **Close and reopen your terminal/VS Code completely**
5. Verify: `java -version` should now work

**Method 2: PowerShell (requires admin)**

Find your Java installation path:
```powershell
Get-ChildItem "C:\Program Files\Eclipse Adoptium" -Recurse -Filter java.exe | Select-Object FullName
```

Set environment variables (run PowerShell as Administrator):
```powershell
# Replace path with your actual Java installation path
[System.Environment]::SetEnvironmentVariable('JAVA_HOME', 'C:\Program Files\Eclipse Adoptium\jdk-25.0.x.x-hotspot', 'Machine')
[System.Environment]::SetEnvironmentVariable('Path', $env:Path + ';C:\Program Files\Eclipse Adoptium\jdk-25.0.x.x-hotspot\bin', 'Machine')
```

Close and reopen terminal, then verify with `java -version`.

## Step 2: Install Leiningen

### Installation

1. Download `lein.bat` from [Leiningen](https://raw.githubusercontent.com/technomancy/leiningen/stable/bin/lein.bat)
2. Create a permanent directory for it (e.g., `C:\lein\`)
3. Save `lein.bat` to that directory

### Add Leiningen to PATH

**Method 1: GUI**

1. `Win+R` → `rundll32 sysdm.cpl,EditEnvironmentVariables` → Enter
2. In top section (User variables), find **Path**
3. Edit → New → Add `C:\lein` (or wherever you saved lein.bat)
4. OK on all dialogs
5. Close and reopen terminal

**Method 2: PowerShell (as admin)**

```powershell
[System.Environment]::SetEnvironmentVariable('Path', $env:Path + ';C:\lein', 'Machine')
```

### Complete Leiningen Installation

Run the self-install process:
```powershell
lein self-install
```

This downloads Leiningen dependencies (may take a few minutes on first run).

Verify:
```powershell
lein version
```

Should show: `Leiningen 2.x.x on Java 25 OpenJDK 64-Bit Server VM`

## Step 3: Clone and Build Ooloi

Ooloi consists of three separate Leiningen projects that must be built in sequence:

```powershell
git clone https://github.com/PeterBengtson/Ooloi.git
cd Ooloi
```

### Install Dependencies for Each Project

**Critical**: Follow this exact sequence:

```powershell
# 1. Shared project (MUST BE FIRST)
cd shared
lein deps
lein protoc         # Compiles Protocol Buffers (required for backend/frontend)

# 2. Backend project
cd ..\backend
lein deps

# 3. Frontend project
cd ..\frontend
lein deps
cd ..
```

**Why this order?** The `lein protoc` command in `shared/` generates gRPC protocol buffer definitions required by both `backend/` and `frontend/`. Skip this and the other projects will fail to compile.

### Run Tests

Verify everything works by testing each project (run sequentially to avoid port conflicts):

```powershell
# Run tests sequentially
cd shared
lein midje          # Takes a few minutes

cd ..\backend
lein midje

cd ..\frontend
lein midje

cd ..
```

All tests should pass on a clean Windows installation.

## Common Issues

### PowerShell Execution Policy

If PowerShell blocks running scripts:
```powershell
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
```

### Antivirus Interference

Some antivirus software may flag or slow down Leiningen downloads. Consider whitelisting:
- `C:\lein\` directory
- `%USERPROFILE%\.lein\` directory

### PATH Changes Not Taking Effect

Environment variable changes only affect new processes. You must:
1. Close all terminal windows completely
2. Close VS Code or your IDE completely
3. Reopen your terminal/IDE

Simply opening a new terminal tab is insufficient - the parent process must reload.

### Finding Java Installation Path

If unsure where Java installed:
```powershell
Get-ChildItem "C:\Program Files" -Recurse -Filter java.exe -ErrorAction SilentlyContinue | Select-Object FullName
```

Look for paths containing "Eclipse Adoptium" or "Temurin".

### Port Conflicts During Tests

**Symptom**: Tests fail with "address already in use" errors.

**Solution**: Run test suites **sequentially** (one project at a time), not in parallel windows. Tests use ports 10700-10701 which will conflict if multiple test suites run simultaneously.

### Long Path Issues

Windows has a 260-character path limit by default. Enable long paths:

1. Open Group Policy Editor: `Win+R` → `gpedit.msc`
2. Navigate to: Computer Configuration → Administrative Templates → System → Filesystem
3. Enable "Enable Win32 long paths"
4. Restart

Or via registry (requires admin):
```powershell
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" -Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force
```

## Windows Subsystem for Linux (WSL) - The Better Option

**Important**: WSL runs Linux, not Windows. This section is for Windows users who want to avoid the complexity of native Windows development.

### Why Use WSL?

Native Windows development is significantly more difficult than macOS or Linux:
- PATH configuration frequently fails and requires manual fixes
- Installers don't reliably set environment variables
- File path handling differs (`\` vs `/`)
- PowerShell vs CMD vs Git Bash inconsistencies
- Antivirus interference with development tools

Microsoft themselves have acknowledged these issues by creating WSL - a Linux environment that runs on Windows. For Ooloi development, WSL provides a cleaner, more Unix-like experience.

### Setting Up WSL

**This installs Linux on Windows**:

```powershell
wsl --install
```

After reboot, you're running Ubuntu Linux. **Follow the [Linux setup guide](LINUX.md)** for all installation steps:

```bash
# You're now in Linux, not Windows
sudo apt update
sudo apt install openjdk-25-jdk openjfx protobuf-compiler git leiningen
git clone https://github.com/PeterBengtson/Ooloi.git
cd Ooloi
# Continue with Linux setup guide
```

### Critical Distinction

- **WSL = Linux on Windows** - You're developing on Linux, just hosted by Windows
- **Native Windows = actual Windows** - Required for testing Windows platform compatibility

**If you use WSL, you are testing Linux compatibility, not Windows compatibility.** For true Windows platform validation, use the native Windows installation instructions above.

## Platform Testing

Ooloi targets three platforms: macOS, Windows, and Linux. When developing on Windows, be aware of platform-specific issues:

- **File path handling**: Windows uses `\` separators vs `/` on Unix
- **Graphics rendering**: JavaFX/Skija behavior varies by platform
- **Font rendering**: Font availability and rendering differ
- **DPI scaling**: Windows-specific high-DPI considerations
- **Native library loading**: Platform-specific native dependencies

## Next Steps

After successful installation and test verification:

1. Return to [main setup guide](README.md) for development workflow
2. See [main README](../README.md) for project overview
3. Check [ADRs](../ADRs/) for architecture documentation
4. Review [guides](../guides/) for technical details

## Getting Help

If you encounter Windows-specific issues:

1. Check [GitHub Issues](https://github.com/PeterBengtson/Ooloi/issues) for similar problems
2. Verify Java and Leiningen are correctly installed and in PATH
3. Ensure all environment changes took effect (restart terminal/IDE)
