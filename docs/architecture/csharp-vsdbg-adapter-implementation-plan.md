# C# vsdbg Adapter Implementation Plan

**Status**: Planning Phase  
**Target Version**: 0.16.0  
**Estimated Effort**: Medium  
**Priority**: High

## Executive Summary

This document outlines the implementation plan for adding C# debugging support to mcp-debugger using Microsoft's vsdbg (Visual Studio Debugger). The adapter will follow the established patterns from the Python adapter while accommodating .NET-specific requirements.

## Background

### What is vsdbg?

vsdbg is Microsoft's command-line debugger for .NET applications. It implements the Debug Adapter Protocol (DAP) and is used by VS Code's C# extension. Key characteristics:

- **Protocol**: Implements DAP natively
- **Platforms**: Cross-platform (Windows, Linux, macOS)
- **Installation**: Distributed with VS Code C# extension or standalone via scripts
- **Target Runtimes**: .NET Core, .NET 5+, .NET Framework (Windows only)
- **Architecture**: Runs as a separate process that communicates via stdin/stdout

### Why vsdbg?

1. **Official Support**: Maintained by Microsoft
2. **Full DAP Compliance**: Native DAP implementation
3. **Cross-Platform**: Works on all major OS platforms
4. **Mature**: Battle-tested in VS Code
5. **Feature-Rich**: Supports advanced debugging features

## Architecture Overview

### Package Structure

```
packages/adapter-csharp/
├── package.json              # Package manifest with dependencies
├── tsconfig.json             # TypeScript configuration
├── README.md                 # Adapter documentation
├── src/
│   ├── index.ts              # Public exports
│   ├── csharp-debug-adapter.ts       # Main adapter implementation
│   ├── csharp-adapter-factory.ts     # Factory for adapter creation
│   └── utils/
│       ├── dotnet-utils.ts           # .NET CLI utilities
│       ├── vsdbg-utils.ts            # vsdbg-specific utilities
│       └── project-utils.ts          # C# project detection
├── tests/
│   ├── csharp-adapter.test.ts        # Unit tests
│   ├── dotnet-utils.test.ts          # Utility tests
│   └── fixtures/                     # Test fixtures
└── vitest.config.ts          # Test configuration
```

### Key Components

#### 1. CSharpDebugAdapter (Main Adapter Class)

Implements `IDebugAdapter` interface with C#-specific logic:

**Responsibilities**:
- Environment validation (.NET SDK presence, vsdbg installation)
- vsdbg executable discovery and version checking
- Launch configuration transformation for C# projects
- DAP event handling with C#-specific state management
- Error message translation for .NET-specific errors

**State Management**:
- Tracks adapter state (uninitialized → ready → connected → debugging)
- Manages current thread ID for debugging operations
- Caches executable paths and project configurations

#### 2. CSharpAdapterFactory (Factory Class)

Extends `AdapterFactory` base class:

**Responsibilities**:
- Metadata provision (language, version, capabilities)
- Environment validation before adapter creation
- Dependency injection for adapter instances

#### 3. Utility Modules

**dotnet-utils.ts**:
- Find .NET CLI executable (`dotnet` command)
- Get .NET SDK version
- Detect installed runtimes
- Resolve .NET installation paths

**vsdbg-utils.ts**:
- Find vsdbg executable
- Download/install vsdbg if missing
- Validate vsdbg version compatibility
- Get vsdbg installation paths

**project-utils.ts**:
- Detect C# project type (.csproj, .sln)
- Parse project files for build configuration
- Detect framework target (net6.0, net7.0, etc.)
- Find build output paths (bin/Debug, bin/Release)

## Implementation Details

### 1. Language Enumeration Update

**File**: `packages/shared/src/models/index.ts`

```typescript
export enum DebugLanguage {
  PYTHON = 'python',
  MOCK = 'mock',
  CSHARP = 'csharp',  // ADD THIS
}
```

### 2. Executable Discovery

**Priority Order for .NET CLI**:
1. User-specified path (config parameter)
2. Environment variable: `DOTNET_ROOT`
3. System PATH: `dotnet` command
4. Platform-specific default locations:
   - **Windows**: `C:\Program Files\dotnet\dotnet.exe`
   - **Linux**: `/usr/share/dotnet/dotnet`
   - **macOS**: `/usr/local/share/dotnet/dotnet`

**Priority Order for vsdbg**:
1. User-specified path (config parameter)
2. Environment variable: `VSDBG_PATH`
3. VS Code extension directory:
   - **Windows**: `%USERPROFILE%\.vscode\extensions\ms-dotnettools.csharp-*\debugger\vsdbg.exe`
   - **Linux/macOS**: `~/.vscode/extensions/ms-dotnettools.csharp-*/debugger/vsdbg`
4. Common installation paths:
   - `~/.vsdbg/vsdbg` (standalone installation)
   - `/usr/local/vsdbg/vsdbg` (system-wide)

### 3. vsdbg Installation

If vsdbg is not found, provide installation helper:

**Installation Methods**:

1. **Automatic (Recommended)**:
   ```bash
   curl -sSL https://aka.ms/getvsdbgsh | bash /dev/stdin -v latest -l ~/.vsdbg
   ```

2. **Via VS Code**:
   - Install "C# Dev Kit" or "C#" extension
   - vsdbg is bundled automatically

3. **Manual**:
   - Download from Microsoft's releases
   - Extract to known location
   - Set `VSDBG_PATH` environment variable

**Adapter Behavior**:
- Check for vsdbg on initialization
- If missing, throw clear error with installation instructions
- Optionally: Implement automatic download (with user consent)

### 4. Launch Configuration

**Generic Configuration → C# Specific**:

```typescript
interface CSharpLaunchConfig extends LanguageSpecificLaunchConfig {
  // Project configuration
  program: string;              // Path to DLL or EXE
  cwd?: string;                 // Working directory
  args?: string[];              // Program arguments
  
  // .NET runtime
  console?: 'internalConsole' | 'integratedTerminal' | 'externalTerminal';
  justMyCode?: boolean;         // Step through user code only
  requireExactSource?: boolean; // Require source to match binary
  
  // Advanced
  sourceFileMap?: Record<string, string>;  // Map build paths to source paths
  symbolPath?: string[];        // Additional symbol search paths
  processId?: number;           // Attach to existing process
  
  // Logging
  logging?: {
    moduleLoad?: boolean;
    exceptions?: boolean;
    programOutput?: boolean;
  };
}
```

**Default Configuration**:
```typescript
{
  stopOnEntry: false,
  justMyCode: true,
  console: 'integratedTerminal',
  requireExactSource: true,
  logging: {
    moduleLoad: false,
    exceptions: true,
    programOutput: true
  }
}
```

### 5. Environment Validation

**Validation Checks**:

1. **.NET SDK Presence**:
   - Check `dotnet --version` succeeds
   - Verify minimum version (e.g., .NET 6.0+)
   - Warn if no SDK (only runtime) detected

2. **vsdbg Installation**:
   - Check vsdbg executable exists
   - Verify it's executable
   - Check version compatibility (if applicable)

3. **Project Configuration**:
   - Validate program path exists
   - Check if it's a built binary (.dll or .exe)
   - Warn if source files don't match compiled version

4. **Platform Compatibility**:
   - Verify .NET runtime matches target platform
   - Check for platform-specific requirements

**Validation Result Example**:
```typescript
{
  valid: true,
  errors: [],
  warnings: [
    {
      code: 'DEBUG_SYMBOLS_MISSING',
      message: 'Debug symbols not found. Rebuild with /debug:full for better debugging experience.'
    }
  ]
}
```

### 6. Adapter Command Building

**Command Structure**:
```bash
vsdbg --interpreter=vscode --connection=127.0.0.1:4711
```

**Parameters**:
- `--interpreter=vscode`: Use VS Code protocol
- `--connection=<host>:<port>`: Listen on specific host/port

**Implementation**:
```typescript
buildAdapterCommand(config: AdapterConfig): AdapterCommand {
  return {
    command: config.executablePath, // Path to vsdbg
    args: [
      '--interpreter=vscode',
      `--connection=${config.adapterHost}:${config.adapterPort}`
    ],
    env: {
      ...process.env,
      DOTNET_CLI_UI_LANGUAGE: 'en-US',  // Consistent error messages
    }
  };
}
```

### 7. C#-Specific Features

**Supported Features**:
- ✅ Conditional breakpoints
- ✅ Function breakpoints
- ✅ Exception breakpoints (with filters)
- ✅ Data breakpoints (watch expressions)
- ✅ Variable evaluation and modification
- ✅ Hot reload (Edit and Continue) - .NET 6+
- ✅ Async debugging
- ✅ Multi-threaded debugging
- ✅ Remote debugging
- ✅ Attach to process

**Exception Breakpoint Filters**:
```typescript
exceptionBreakpointFilters: [
  {
    filter: 'all',
    label: 'All Exceptions',
    description: 'Break on all thrown exceptions',
    default: false
  },
  {
    filter: 'user-unhandled',
    label: 'User-Unhandled Exceptions',
    description: 'Break on exceptions not handled by user code',
    default: true
  }
]
```

### 8. Error Handling

**Common Error Scenarios**:

1. **.NET Not Found**:
   ```
   .NET SDK not found. Please install .NET from:
   https://dotnet.microsoft.com/download
   
   Or specify the dotnet path in your configuration.
   ```

2. **vsdbg Not Found**:
   ```
   vsdbg not found. Install it using:
   curl -sSL https://aka.ms/getvsdbgsh | bash /dev/stdin -v latest -l ~/.vsdbg
   
   Or install VS Code with the C# extension.
   ```

3. **Program Not Found**:
   ```
   Program not found: /path/to/program.dll
   
   Build your project first:
   dotnet build
   ```

4. **Incompatible Runtime**:
   ```
   Program targets net7.0 but .NET 7.0 runtime not installed.
   Install from: https://dotnet.microsoft.com/download/dotnet/7.0
   ```

### 9. Capabilities Declaration

```typescript
getCapabilities(): AdapterCapabilities {
  return {
    supportsConfigurationDoneRequest: true,
    supportsFunctionBreakpoints: true,
    supportsConditionalBreakpoints: true,
    supportsHitConditionalBreakpoints: true,
    supportsEvaluateForHovers: true,
    supportsStepBack: false,
    supportsSetVariable: true,
    supportsRestartFrame: false,
    supportsGotoTargetsRequest: true,
    supportsStepInTargetsRequest: true,
    supportsCompletionsRequest: true,
    completionTriggerCharacters: ['.', '['],
    supportsModulesRequest: true,
    supportsRestartRequest: true,
    supportsExceptionOptions: true,
    supportsValueFormattingOptions: true,
    supportsExceptionInfoRequest: true,
    supportTerminateDebuggee: true,
    supportsDelayedStackTraceLoading: true,
    supportsLoadedSourcesRequest: true,
    supportsDataBreakpoints: true,
    supportsReadMemoryRequest: true,
    supportsDisassembleRequest: true,
    supportsBreakpointLocationsRequest: true,
    supportsClipboardContext: true,
  };
}
```

## Dependencies

### Runtime Dependencies

```json
{
  "dependencies": {
    "@debugmcp/shared": "workspace:*",
    "@vscode/debugprotocol": "^1.68.0",
    "which": "^5.0.0",
    "xml2js": "^0.6.0"  // For parsing .csproj files
  }
}
```

### Development Dependencies

```json
{
  "devDependencies": {
    "@types/node": "^22.15.29",
    "@types/which": "^3.0.4",
    "@types/xml2js": "^0.4.14",
    "typescript": "^5.2.2",
    "vitest": "^3.2.1"
  }
}
```

### External Dependencies

- **.NET SDK**: Version 6.0 or higher (required)
- **vsdbg**: Latest version (required)
- **C# project**: Must be built before debugging

## Testing Strategy

### 1. Unit Tests

**Test Coverage Areas**:
- Adapter initialization and disposal
- State transitions
- Executable discovery logic
- Configuration transformation
- Error message translation
- Feature support checking

**Mock Requirements**:
- File system operations
- Process spawning
- Network operations
- Logger interface

### 2. Integration Tests

**Test Scenarios**:
- Full debug session lifecycle
- Breakpoint setting and hitting
- Variable inspection
- Expression evaluation
- Exception handling
- Multi-threaded scenarios

**Test Fixtures**:
- Simple console application
- Web application (ASP.NET Core)
- Class library project
- Multi-project solution

### 3. End-to-End Tests

**Real-World Scenarios**:
- Debug .NET 6 console app
- Debug ASP.NET Core web app
- Attach to running process
- Cross-platform testing (Windows, Linux, macOS)

**Test Environment Setup**:
- Docker containers with .NET SDK
- Sample C# projects in `examples/csharp/`
- Automated test scripts

## Implementation Phases

### Phase 1: Core Infrastructure (Week 1)
- ✅ Add CSHARP to DebugLanguage enum
- ✅ Create package structure
- ✅ Implement dotnet-utils (executable discovery)
- ✅ Implement vsdbg-utils (vsdbg discovery)
- ✅ Write unit tests for utilities

### Phase 2: Adapter Implementation (Week 1-2)
- ✅ Implement CSharpDebugAdapter class
- ✅ Implement environment validation
- ✅ Implement configuration transformation
- ✅ Implement error handling
- ✅ Write unit tests for adapter

### Phase 3: Factory and Integration (Week 2)
- ✅ Implement CSharpAdapterFactory
- ✅ Add metadata and capabilities
- ✅ Write integration tests
- ✅ Test with real C# projects

### Phase 4: Documentation and Polish (Week 2-3)
- ✅ Write comprehensive README
- ✅ Add code examples
- ✅ Update main documentation
- ✅ Create troubleshooting guide
- ✅ Add to CI/CD pipeline

### Phase 5: Advanced Features (Week 3+)
- ⚠️ Optional: Auto-install vsdbg feature
- ⚠️ Optional: Project build integration
- ⚠️ Optional: Solution file support
- ⚠️ Optional: Remote debugging support

## File Checklist

### Required Files

- [ ] `packages/adapter-csharp/package.json`
- [ ] `packages/adapter-csharp/tsconfig.json`
- [ ] `packages/adapter-csharp/README.md`
- [ ] `packages/adapter-csharp/vitest.config.ts`
- [ ] `packages/adapter-csharp/src/index.ts`
- [ ] `packages/adapter-csharp/src/csharp-debug-adapter.ts`
- [ ] `packages/adapter-csharp/src/csharp-adapter-factory.ts`
- [ ] `packages/adapter-csharp/src/utils/dotnet-utils.ts`
- [ ] `packages/adapter-csharp/src/utils/vsdbg-utils.ts`
- [ ] `packages/adapter-csharp/src/utils/project-utils.ts`
- [ ] `packages/adapter-csharp/tests/csharp-adapter.test.ts`
- [ ] `packages/adapter-csharp/tests/dotnet-utils.test.ts`
- [ ] `packages/adapter-csharp/tests/vsdbg-utils.test.ts`

### Documentation Files

- [ ] `docs/architecture/csharp-adapter-guide.md`
- [ ] `examples/csharp/README.md`
- [ ] `examples/csharp/simple-console/Program.cs`
- [ ] `examples/csharp/simple-console/simple-console.csproj`

### Modified Files

- [ ] `packages/shared/src/models/index.ts` (add CSHARP enum)
- [ ] `README.md` (update language support list)
- [ ] `CHANGELOG.md` (add new feature)

## Potential Challenges

### 1. vsdbg Installation Complexity

**Challenge**: vsdbg is not as universally available as debugpy for Python  
**Mitigation**:
- Provide clear installation instructions
- Consider implementing auto-download with user consent
- Support multiple installation paths

### 2. .NET Version Fragmentation

**Challenge**: Multiple .NET versions (.NET Framework, .NET Core, .NET 5+)  
**Mitigation**:
- Focus on .NET 6+ (current LTS)
- Clearly document minimum version requirements
- Provide version detection and warnings

### 3. Project Build Requirements

**Challenge**: C# requires compilation before debugging  
**Mitigation**:
- Validate that program file exists
- Provide clear error messages about building
- Consider optional build integration

### 4. Cross-Platform Path Handling

**Challenge**: Different vsdbg locations on different platforms  
**Mitigation**:
- Use platform-specific default paths
- Leverage `which` library for PATH searching
- Support explicit path configuration

### 5. Symbol and Source Mapping

**Challenge**: Debug symbols must match source code  
**Mitigation**:
- Check for .pdb files
- Warn about mismatched symbols
- Support `sourceFileMap` configuration

## Success Criteria

### Functional Requirements
- ✅ Can create C# debug session
- ✅ Can set and hit breakpoints
- ✅ Can step through code (step in, step over, step out)
- ✅ Can inspect variables
- ✅ Can evaluate expressions
- ✅ Can handle exceptions
- ✅ Works on Windows, Linux, and macOS

### Quality Requirements
- ✅ >90% test coverage
- ✅ Clear error messages
- ✅ Comprehensive documentation
- ✅ Performance: <100ms initialization
- ✅ No memory leaks in long-running sessions

### User Experience Requirements
- ✅ Easy installation (clear instructions)
- ✅ Works out-of-box if .NET and vsdbg installed
- ✅ Helpful error messages with actionable steps
- ✅ Consistent with Python adapter UX

## References

### Official Documentation
- [vsdbg Documentation](https://github.com/OmniSharp/omnisharp-vscode/wiki/Debugging-with-VSCode)
- [.NET CLI Documentation](https://docs.microsoft.com/en-us/dotnet/core/tools/)
- [Debug Adapter Protocol Specification](https://microsoft.github.io/debug-adapter-protocol/)
- [C# Extension for VS Code](https://github.com/dotnet/vscode-csharp)

### Example Implementations
- [VS Code C# Extension](https://github.com/dotnet/vscode-csharp/tree/main/src/features)
- [OmniSharp Server](https://github.com/OmniSharp/omnisharp-roslyn)

### Related Projects
- [@debugmcp/adapter-python](../packages/adapter-python/) - Reference implementation
- [debugpy](https://github.com/microsoft/debugpy) - Python debugger (similar architecture)

## Appendix

### A. Example Usage

```typescript
// Create C# debug session
const session = await server.call('create_debug_session', {
  language: 'csharp',
  name: 'MyApp Debug',
  executablePath: '/usr/share/dotnet/dotnet' // Optional
});

// Start debugging
await server.call('start_debugging', {
  sessionId: session.sessionId,
  scriptPath: '/path/to/project/bin/Debug/net8.0/MyApp.dll',
  args: ['--verbose'],
  stopOnEntry: false
});
```

### B. Configuration Examples

**Simple Console App**:
```json
{
  "language": "csharp",
  "program": "${workspaceFolder}/bin/Debug/net8.0/MyApp.dll",
  "args": [],
  "cwd": "${workspaceFolder}",
  "stopOnEntry": false,
  "console": "integratedTerminal"
}
```

**ASP.NET Core Web App**:
```json
{
  "language": "csharp",
  "program": "${workspaceFolder}/bin/Debug/net8.0/WebApp.dll",
  "args": [],
  "cwd": "${workspaceFolder}",
  "stopOnEntry": false,
  "console": "internalConsole",
  "env": {
    "ASPNETCORE_ENVIRONMENT": "Development"
  }
}
```

**Attach to Process**:
```json
{
  "language": "csharp",
  "processId": "${command:pickProcess}",
  "justMyCode": false
}
```

### C. Platform-Specific Notes

**Windows**:
- vsdbg typically at: `%USERPROFILE%\.vscode\extensions\ms-dotnettools.csharp-*\debugger\vsdbg.exe`
- .NET SDK typically at: `C:\Program Files\dotnet\`
- Both .NET Framework and .NET Core supported

**Linux**:
- vsdbg typically at: `~/.vscode/extensions/ms-dotnettools.csharp-*/debugger/vsdbg`
- .NET SDK typically at: `/usr/share/dotnet/` or `/usr/local/share/dotnet/`
- Only .NET Core/.NET 5+ supported

**macOS**:
- vsdbg typically at: `~/.vscode/extensions/ms-dotnettools.csharp-*/debugger/vsdbg`
- .NET SDK typically at: `/usr/local/share/dotnet/`
- Homebrew installation: `/opt/homebrew/bin/dotnet`
- Only .NET Core/.NET 5+ supported

### D. Troubleshooting Guide

**Issue: vsdbg not found**
```
Solution:
1. Check if VS Code C# extension is installed
2. Run: ls ~/.vscode/extensions/ms-dotnettools.csharp-*/debugger/
3. If missing, install vsdbg:
   curl -sSL https://aka.ms/getvsdbgsh | bash /dev/stdin -v latest -l ~/.vsdbg
4. Set VSDBG_PATH environment variable
```

**Issue: .NET SDK not found**
```
Solution:
1. Verify installation: dotnet --version
2. If missing, install from: https://dotnet.microsoft.com/download
3. Add to PATH if needed
4. Restart terminal/IDE
```

**Issue: Program.dll not found**
```
Solution:
1. Build your project: dotnet build
2. Check build output path: ls bin/Debug/net*/
3. Verify program path in debug configuration
4. Ensure target framework matches installed runtime
```

---

**Document Version**: 1.0  
**Last Updated**: 2025-11-03  
**Author**: MCP Debugger Team  
**Status**: Ready for Implementation
