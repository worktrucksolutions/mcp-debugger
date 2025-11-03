# C# vsdbg Adapter Implementation Plan

## Overview

This document outlines the implementation plan for adding C# debugging support to mcp-debugger using the Visual Studio Code debugger (vsdbg). The vsdbg debugger is the cross-platform .NET debugger used by Visual Studio Code for C#, F#, and VB.NET debugging.

## Package Structure

```
packages/adapter-csharp/
  package.json
  tsconfig.json
  README.md
  src/
    index.ts
    csharp-adapter-factory.ts
    csharp-debug-adapter.ts
    utils/
      vsdbg-utils.ts
      dotnet-utils.ts
  tests/
    csharp-adapter.test.ts
    vsdbg-utils.test.ts
    dotnet-utils.test.ts
  vitest.config.ts
```

## Package Metadata

- **Package Name**: `@debugmcp/adapter-csharp`
- **Version**: `0.1.0` (initial release)
- **Factory Class**: `CSharpAdapterFactory`
- **Adapter Class**: `CSharpDebugAdapter`
- **Language**: `DebugLanguage.CSHARP` (to be added to shared models)

## Core Components

### 1. CSharpDebugAdapter

Implements `IDebugAdapter` interface with C#/.NET-specific logic:

**Key Responsibilities:**
- vsdbg executable discovery and validation
- .NET SDK version detection
- Launch configuration transformation for .NET projects
- Capabilities declaration for .NET debugging features
- Error message translation for .NET-specific issues

**Key Methods:**
- `resolveExecutablePath()`: Find vsdbg executable
- `validateEnvironment()`: Check .NET SDK and vsdbg installation
- `buildAdapterCommand()`: Build command to launch vsdbg
- `transformLaunchConfig()`: Convert generic config to .NET launch config
- `getCapabilities()`: Return .NET-specific DAP capabilities

### 2. CSharpAdapterFactory

Extends `AdapterFactory` base class:

**Key Responsibilities:**
- Factory validation for .NET SDK and vsdbg
- Adapter instance creation with dependencies
- Metadata about the C# adapter

**Metadata:**
```typescript
{
  language: DebugLanguage.CSHARP,
  displayName: 'C#',
  version: '0.1.0',
  author: 'mcp-debugger team',
  description: 'Debug C#/.NET applications using vsdbg',
  documentationUrl: 'https://github.com/debugmcp/mcp-debugger/docs/csharp',
  minimumDebuggerVersion: '1.0.0',
  fileExtensions: ['.cs', '.csx', '.fs', '.fsx', '.vb'],
  icon: '...' // C# icon data URL
}
```

### 3. Utility Modules

#### vsdbg-utils.ts
- `findVsdbgExecutable()`: Locate vsdbg on different platforms
- `getVsdbgVersion()`: Get vsdbg version
- `validateVsdbgInstallation()`: Verify vsdbg is properly installed
- `getVsdbgInstallPaths()`: Platform-specific search paths

#### dotnet-utils.ts
- `findDotnetExecutable()`: Find .NET SDK executable
- `getDotnetVersion()`: Get .NET SDK version
- `validateDotnetSdk()`: Check .NET SDK installation
- `getDotnetInstallPaths()`: Platform-specific search paths
- `detectProjectType()`: Detect .NET project type (Framework, Core, .NET 5+)

## Implementation Details

### vsdbg Discovery

vsdbg location varies by platform and installation method:

**Windows:**
- Visual Studio Code: `%USERPROFILE%\.vscode\extensions\ms-dotnettools.csharp-*\debugAdapters\vsdbg.exe`
- Visual Studio: `%ProgramFiles%\Microsoft Visual Studio\*\Common7\IDE\Extensions\*\debugAdapters\vsdbg.exe`
- Standalone: User-specified path

**Linux/macOS:**
- Visual Studio Code: `~/.vscode/extensions/ms-dotnettools.csharp-*/debugAdapters/vsdbg`
- Standalone: User-specified path

**Search Strategy:**
1. Check `VSDBG_PATH` environment variable
2. Check user-specified path (from config)
3. Search VS Code extensions directory (user and global)
   - User: `~/.vscode/extensions/` (Linux/macOS) or `%USERPROFILE%\.vscode\extensions\` (Windows)
   - Global: Platform-specific global extension paths
   - Search for `ms-dotnettools.csharp-*` extension directories
   - Check multiple versions (latest preferred)
4. Search Visual Studio installation (Windows only)
   - `%ProgramFiles%\Microsoft Visual Studio\*\Common7\IDE\Extensions\*\debugAdapters\vsdbg.exe`
5. Check common installation paths
   - May need to search for `.NET Core Debugger` extension as well

**Note**: vsdbg version detection may require:
- Reading extension manifest/package.json
- Parsing executable metadata
- Checking extension directory version numbers

### .NET SDK Detection

**Required:**
- .NET SDK 2.1+ (for .NET Core debugging)
- .NET SDK 5.0+ (for modern .NET debugging) - recommended
- .NET Framework 4.5+ (for Framework debugging on Windows only)

**Detection Methods:**
1. **.NET SDK**:
   - Check `DOTNET_ROOT` environment variable
   - Find `dotnet` command in PATH
   - Platform-specific paths:
     - Windows: `%ProgramFiles%\dotnet\dotnet.exe`
     - Linux: `/usr/share/dotnet/dotnet` or `/usr/local/share/dotnet/dotnet`
     - macOS: `/usr/local/share/dotnet/dotnet`
   - Use `dotnet --version` to get SDK version
   - Use `dotnet --list-runtimes` to verify runtime installation

2. **.NET Framework** (Windows only):
   - Check registry: `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\NET Framework Setup\NDP\`
   - Check installed programs
   - Check for `%WINDIR%\Microsoft.NET\Framework\*` directories
   - Note: .NET Framework debugging requires Windows-specific vsdbg setup

### vsdbg Communication Method

**CRITICAL**: vsdbg uses **stdio** (NDJSON) for DAP communication, NOT TCP like debugpy.

The adapter manager spawns vsdbg with stdio communication:
- vsdbg is invoked with no TCP arguments (no `--host`/`--port`)
- DAP messages are sent/received via stdin/stdout (NDJSON format)
- The adapter manager handles stdio communication
- vsdbg may output logs to stderr (separate from DAP protocol)

**Command Building:**
```typescript
buildAdapterCommand(config: AdapterConfig): AdapterCommand {
  return {
    command: config.executablePath,
    args: [],  // vsdbg uses stdio, no TCP args needed
    env: {
      ...process.env,
      // vsdbg-specific environment variables if needed
      // May need to configure logging to avoid stdout pollution
    }
  };
}
```

**Important**: Ensure vsdbg logging doesn't pollute stdout (must be NDJSON-only per DAP spec).

### Launch Configuration

**Generic → .NET Transformation:**

```typescript
interface CSharpLaunchConfig extends LanguageSpecificLaunchConfig {
  type: 'coreclr' | 'clr';  // Note: 'netcoredbg' is a different debugger, not vsdbg
  request: 'launch' | 'attach';
  program?: string;           // Path to executable
  args?: string[];            // Command-line arguments
  cwd?: string;               // Working directory
  env?: Record<string, string>; // Environment variables
  console?: 'internalConsole' | 'integratedTerminal' | 'externalTerminal';
  stopAtEntry?: boolean;       // Stop at entry point
  justMyCode?: boolean;        // Debug only user code
  enableStepFiltering?: boolean;
  symbolOptions?: {
    searchPaths?: string[];
    searchMicrosoftSymbolServer?: boolean;
  };
  // Additional .NET-specific options
}
```

**Note**: The launch configuration is sent via DAP `launch` request after initialization, not via command-line arguments.

### Capabilities

vsdbg supports extensive DAP capabilities:

**Supported Features:**
- Conditional breakpoints
- Function breakpoints
- Exception breakpoints (.NET exceptions)
- Variable evaluation
- Set variable values
- Step operations (step over, step into, step out)
- Evaluate expressions
- Log points
- Exception filters (CLR exceptions)
- Modules request
- Loaded sources request
- Restart request
- Terminate request

**Exception Breakpoint Filters:**
- `System.Exception`: All exceptions
- `System.NullReferenceException`: Null reference exceptions
- `System.ArgumentException`: Argument exceptions
- `System.ArgumentNullException`: Argument null exceptions
- `System.IndexOutOfRangeException`: Index out of range exceptions
- `System.InvalidOperationException`: Invalid operation exceptions
- Custom exception types (user-defined exceptions)

**Note**: vsdbg supports filtering by exception type name, allowing breakpoints on specific exception types or all exceptions.

**Capabilities Declaration:**
```typescript
{
  supportsConfigurationDoneRequest: true,
  supportsFunctionBreakpoints: true,
  supportsConditionalBreakpoints: true,
  supportsHitConditionalBreakpoints: true,
  supportsEvaluateForHovers: true,
  supportsSetVariable: true,
  supportsRestartRequest: true,
  supportsTerminateRequest: true,
  supportsExceptionOptions: true,
  supportsExceptionInfoRequest: true,
  supportsModulesRequest: true,
  supportsLoadedSourcesRequest: true,
  supportsLogPoints: true,
  supportsBreakpointLocationsRequest: true,
  exceptionBreakpointFilters: [
    {
      filter: 'System.Exception',
      label: 'All Exceptions',
      default: false,
      supportsCondition: true
    },
    // ... more filters
  ]
}
```

### Environment Validation

**Validation Checks:**
1. .NET SDK installation
   - Check `dotnet` command availability
   - Verify .NET SDK version (2.1+ minimum)
   - Check for .NET Framework (Windows only)

2. vsdbg installation
   - Locate vsdbg executable
   - Verify vsdbg version
   - Check executable permissions

3. Project validation (optional)
   - Detect project type
   - Verify project targets supported .NET version
   - Check for required dependencies

**Validation Result:**
```typescript
{
  valid: boolean,
  errors: [
    {
      code: 'DOTNET_SDK_NOT_FOUND',
      message: '...',
      recoverable: false
    },
    {
      code: 'VSDBG_NOT_FOUND',
      message: '...',
      recoverable: true
    }
  ],
  warnings: [
    {
      code: 'DOTNET_VERSION_OLD',
      message: '...'
    }
  ]
}
```

### Error Handling

**Common Error Scenarios:**

1. **vsdbg not found**
   - Error: `VSDBG_NOT_FOUND`
   - Message: "vsdbg debugger not found. Install the C# extension for VS Code or download vsdbg manually."
   - Recovery: Install VS Code C# extension or download vsdbg

2. **.NET SDK not found**
   - Error: `DOTNET_SDK_NOT_FOUND`
   - Message: ".NET SDK not found. Install .NET SDK from https://dotnet.microsoft.com/download"
   - Recovery: Install .NET SDK

3. **Incompatible .NET version**
   - Error: `DOTNET_VERSION_INCOMPATIBLE`
   - Message: ".NET SDK version X.Y.Z is not supported. Requires .NET SDK 2.1 or higher."
   - Recovery: Update .NET SDK

4. **Project type mismatch**
   - Error: `PROJECT_TYPE_UNSUPPORTED`
   - Message: "Project type not supported for debugging."
   - Recovery: Use supported project type

**Error Translation:**
```typescript
translateErrorMessage(error: Error): string {
  const message = error.message.toLowerCase();
  
  if (message.includes('vsdbg') && message.includes('not found')) {
    return 'vsdbg debugger not found. Install the C# extension for VS Code.';
  }
  
  if (message.includes('dotnet') && message.includes('not found')) {
    return '.NET SDK not found. Install .NET SDK from https://dotnet.microsoft.com/download';
  }
  
  // ... more translations
}
```

## File Structure

### src/index.ts
```typescript
export { CSharpAdapterFactory } from './csharp-adapter-factory.js';
export { CSharpDebugAdapter } from './csharp-debug-adapter.js';
export {
  findVsdbgExecutable,
  getVsdbgVersion,
  findDotnetExecutable,
  getDotnetVersion
} from './utils/vsdbg-utils.js';
export type { CommandFinder } from './utils/vsdbg-utils.js';

export default {
  name: 'csharp',
  factory: CSharpAdapterFactory
};
```

### src/csharp-debug-adapter.ts

Main adapter implementation following the Python adapter pattern:
- Extends `EventEmitter`
- Implements `IDebugAdapter`
- Handles lifecycle, state, validation, executable discovery
- Builds adapter commands
- Transforms launch configurations
- Declares capabilities

### src/csharp-adapter-factory.ts

Factory implementation:
- Extends `AdapterFactory`
- Implements `createAdapter()`
- Provides metadata
- Validates environment (checks .NET SDK and vsdbg)

### src/utils/vsdbg-utils.ts

vsdbg-specific utilities:
- Executable discovery
- Version detection
- Installation validation
- Platform-specific path resolution

### src/utils/dotnet-utils.ts

.NET SDK utilities:
- SDK detection
- Version checking
- Project type detection
- Framework detection

## Testing Strategy

### Unit Tests

**csharp-adapter.test.ts:**
- Factory instantiation
- Adapter creation
- Export validation

**vsdbg-utils.test.ts:**
- vsdbg discovery on different platforms
- Version detection
- Path resolution
- Error handling

**dotnet-utils.test.ts:**
- .NET SDK detection
- Version parsing
- Project type detection
- Framework detection

### Integration Tests

- End-to-end debugging workflow
- Launch configuration validation
- Error message verification
- Environment validation

## Dependencies

### Runtime Dependencies
- `@debugmcp/shared`: Core interfaces and types
- `@vscode/debugprotocol`: DAP protocol types
- `which`: Command resolution (optional, can use Node.js built-ins)

### Dev Dependencies
- `typescript`: TypeScript compiler
- `@types/node`: Node.js type definitions
- `vitest`: Testing framework

## Configuration Files

### package.json
```json
{
  "name": "@debugmcp/adapter-csharp",
  "version": "0.1.0",
  "description": "C# debug adapter for MCP debugger using vsdbg",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": ["dist"],
  "scripts": {
    "build": "tsc -b",
    "build:ci": "tsc -b -f",
    "clean": "rimraf dist && rimraf tsconfig.tsbuildinfo",
    "test": "vitest run",
    "test:watch": "vitest watch"
  },
  "dependencies": {
    "@debugmcp/shared": "workspace:*",
    "@vscode/debugprotocol": "^1.68.0"
  },
  "devDependencies": {
    "@types/node": "^22.15.29",
    "typescript": "^5.2.2",
    "vitest": "^3.2.1"
  },
  "peerDependencies": {
    "@debugmcp/shared": "workspace:*"
  }
}
```

### tsconfig.json
```json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "composite": true,
    "tsBuildInfoFile": "./dist/tsconfig.tsbuildinfo",
    "outDir": "dist",
    "rootDir": "src",
    "baseUrl": ".",
    "paths": {
      "@debugmcp/shared": ["../shared/src"],
      "@debugmcp/shared/*": ["../shared/src/*"]
    },
    "declaration": true,
    "declarationMap": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true
  },
  "references": [
    { "path": "../shared" }
  ],
  "include": ["src/**/*"],
  "exclude": ["dist", "node_modules", "**/*.test.ts", "**/*.spec.ts"]
}
```

## Required Changes to Shared Package

### packages/shared/src/models/index.ts

Add `CSHARP` to `DebugLanguage` enum:

```typescript
export enum DebugLanguage {
  PYTHON = 'python',
  MOCK = 'mock',
  CSHARP = 'csharp',  // Add this
}
```

## Implementation Steps

### Phase 1: Foundation
1. Create package structure
2. Set up package.json and tsconfig.json
3. Add `CSHARP` to `DebugLanguage` enum in shared package
4. Create basic factory and adapter classes (skeleton)

### Phase 2: Core Implementation
5. Implement vsdbg discovery utilities
6. Implement .NET SDK detection utilities
7. Implement environment validation
8. Implement executable path resolution

### Phase 3: Adapter Implementation
9. Implement `CSharpDebugAdapter` class
10. Implement lifecycle methods
11. Implement launch configuration transformation
12. Implement capabilities declaration

### Phase 4: Factory Implementation
13. Implement `CSharpAdapterFactory` class
14. Implement factory validation
15. Implement metadata

### Phase 5: Testing
16. Write unit tests
17. Write integration tests
18. Test on different platforms (Windows, Linux, macOS)

### Phase 6: Documentation
19. Write README.md
20. Add usage examples
21. Document installation requirements

## Platform-Specific Considerations

### Windows
- **Executable**: `vsdbg.exe`
- **Visual Studio paths**: `%ProgramFiles%\Microsoft Visual Studio\*\Common7\IDE\Extensions\*\debugAdapters\vsdbg.exe`
- **VS Code paths**: `%USERPROFILE%\.vscode\extensions\ms-dotnettools.csharp-*\debugAdapters\vsdbg.exe`
- **.NET Framework support**: Available (Windows-only)
- **Registry checks**: Required for .NET Framework detection
- **Error messages**: Windows-specific paths and registry references

### Linux
- **Executable**: `vsdbg` (no extension)
- **VS Code paths**: `~/.vscode/extensions/ms-dotnettools.csharp-*/debugAdapters/vsdbg`
- **Executable permissions**: Must be executable (chmod +x)
- **.NET SDK paths**: `/usr/share/dotnet/dotnet` or `/usr/local/share/dotnet/dotnet`
- **.NET Framework**: Not available (Windows-only)
- **Error messages**: Linux-specific paths and permission references

### macOS
- **Executable**: `vsdbg` (no extension)
- **VS Code paths**: `~/.vscode/extensions/ms-dotnettools.csharp-*/debugAdapters/vsdbg`
- **Executable permissions**: Must be executable
- **.NET SDK paths**: `/usr/local/share/dotnet/dotnet`
- **.NET Framework**: Not available (Windows-only)
- **Error messages**: macOS-specific paths and permission references

## Stdio Mode Requirements

**CRITICAL**: vsdbg must output only NDJSON to stdout (per DAP specification).

**Requirements:**
- vsdbg logs should be configured to output to stderr or files, not stdout
- No non-JSON output to stdout (would corrupt DAP protocol)
- May need to configure vsdbg via environment variables or configuration
- The adapter manager uses `stdio: ['ignore', 'inherit', 'inherit', 'ipc']` which inherits stdout/stderr

**Configuration:**
- Check if vsdbg has environment variables for log configuration
- May need to redirect stderr to log files
- Ensure vsdbg doesn't output diagnostic messages to stdout

## Known Challenges

1. **vsdbg Installation Location**
   - Varies by VS Code extension version
   - May require VS Code extension installation
   - Solution: Comprehensive search paths with fallback

2. **.NET SDK Version Detection**
   - Multiple SDK versions may be installed
   - Need to detect latest compatible version
   - Solution: Use `dotnet --version` and parse output

3. **Project Type Detection**
   - .NET Framework vs .NET Core vs .NET 5+
   - Different launch configurations
   - Solution: Parse project file or use `dotnet` commands

4. **Cross-Platform Compatibility**
   - Different path separators
   - Different executable names
   - Different installation locations
   - Solution: Platform-specific utilities with common interface

## Future Enhancements

1. **Project Type Auto-Detection**
   - Automatically detect project type from .csproj/.sln files
   - Suggest appropriate launch configuration

2. **Symbol Server Support**
   - Configure symbol server for debugging
   - Download symbols automatically
   - Configure PDB file locations

3. **Source Map Support**
   - Support for source maps in debugging
   - Map generated code to source

4. **Multi-Targeting Support**
   - Support projects targeting multiple frameworks (TFM)
   - Allow selection of target framework
   - Auto-detect from project file

5. **Attach Mode Support**
   - Attach to running processes
   - Process ID selection
   - Process name matching

6. **Project Type Auto-Detection**
   - Automatically detect project type from .csproj/.sln files
   - Suggest appropriate launch configuration
   - Validate project compatibility

7. **Container Mode Support**
   - vsdbg discovery in Docker containers
   - .NET SDK installation in containers
   - Path resolution in containerized environments

## References

- [vsdbg Documentation](https://github.com/Microsoft/vscode-docs/blob/main/docs/languages/csharp.md)
- [Debug Adapter Protocol](https://microsoft.github.io/debug-adapter-protocol/)
- [.NET SDK Documentation](https://docs.microsoft.com/en-us/dotnet/core/)
- [Python Adapter Implementation](../adapter-python/) (reference implementation)
