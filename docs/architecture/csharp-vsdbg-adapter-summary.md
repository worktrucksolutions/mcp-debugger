# C# vsdbg Adapter - Quick Reference Summary

## Overview

This is a quick reference summary for implementing the C# vsdbg adapter for mcp-debugger. For detailed information, see [csharp-vsdbg-adapter-plan.md](./csharp-vsdbg-adapter-plan.md).

## Package Details

- **Package Name**: `@debugmcp/adapter-csharp`
- **Version**: `0.1.0` (initial)
- **Location**: `packages/adapter-csharp/`
- **Factory Class**: `CSharpAdapterFactory`
- **Adapter Class**: `CSharpDebugAdapter`
- **Language Enum Value**: `DebugLanguage.CSHARP` (to be added)

## Key Components

### Files to Create

```
packages/adapter-csharp/
├── src/
│   ├── index.ts                    # Exports
│   ├── csharp-adapter-factory.ts   # Factory implementation
│   ├── csharp-debug-adapter.ts     # Adapter implementation
│   └── utils/
│       ├── vsdbg-utils.ts          # vsdbg discovery utilities
│       └── dotnet-utils.ts         # .NET SDK utilities
├── tests/
│   ├── csharp-adapter.test.ts
│   ├── vsdbg-utils.test.ts
│   └── dotnet-utils.test.ts
├── package.json
├── tsconfig.json
└── README.md
```

## vsdbg Discovery

vsdbg is typically found in:
- **VS Code**: `~/.vscode/extensions/ms-dotnettools.csharp-*/debugAdapters/vsdbg`
- **Windows VS**: `%ProgramFiles%\Microsoft Visual Studio\*\Common7\IDE\Extensions\*\debugAdapters\vsdbg.exe`
- **Environment**: Check `VSDBG_PATH` environment variable

## .NET SDK Requirements

- Minimum: .NET SDK 2.1+
- Recommended: .NET SDK 5.0+
- Detection: Use `dotnet --version` command
- Path: Check `DOTNET_ROOT` env var or find `dotnet` in PATH

## Launch Configuration

Transform generic config to .NET launch config:

```typescript
{
  type: 'coreclr' | 'clr',
  request: 'launch',
  program: string,        // Path to executable
  args?: string[],
  cwd?: string,
  env?: Record<string, string>,
  stopAtEntry?: boolean,
  justMyCode?: boolean,
  // ... more options
}
```

## Capabilities

vsdbg supports:
- ✅ Conditional breakpoints
- ✅ Function breakpoints
- ✅ Exception breakpoints (.NET exceptions)
- ✅ Variable evaluation and modification
- ✅ Step operations
- ✅ Expression evaluation
- ✅ Log points
- ✅ Exception filters
- ✅ Modules and loaded sources
- ✅ Restart and terminate

## Required Changes

### Shared Package Update

Add to `packages/shared/src/models/index.ts`:

```typescript
export enum DebugLanguage {
  PYTHON = 'python',
  MOCK = 'mock',
  CSHARP = 'csharp',  // ← Add this
}
```

## Implementation Steps (High-Level)

1. **Setup**: Create package structure and config files
2. **Utilities**: Implement vsdbg and .NET SDK discovery utilities
3. **Adapter**: Implement `CSharpDebugAdapter` class
4. **Factory**: Implement `CSharpAdapterFactory` class
5. **Exports**: Set up proper exports in `index.ts`
6. **Tests**: Write unit and integration tests
7. **Docs**: Create README and documentation
8. **Integration**: Test with core debugger system

## Key Differences from Python Adapter

1. **Executable**: vsdbg vs debugpy
2. **SDK Detection**: .NET SDK vs Python version
3. **Launch Config**: .NET-specific options vs Python-specific
4. **Exception Filters**: .NET exception types vs Python exceptions
5. **Platforms**: vsdbg is cross-platform (Windows/Linux/macOS)

## Testing Checklist

- [ ] vsdbg discovery on Windows
- [ ] vsdbg discovery on Linux
- [ ] vsdbg discovery on macOS
- [ ] .NET SDK detection
- [ ] Environment validation
- [ ] Launch configuration transformation
- [ ] Capabilities declaration
- [ ] Error handling and messages
- [ ] Integration with core debugger

## References

- **Python Adapter**: `packages/adapter-python/` (reference implementation)
- **Development Guide**: `docs/architecture/adapter-development-guide.md`
- **Shared Interfaces**: `packages/shared/src/interfaces/debug-adapter.ts`
- **Detailed Plan**: `docs/architecture/csharp-vsdbg-adapter-plan.md`
- **Checklist**: `docs/architecture/csharp-vsdbg-adapter-checklist.md`

## Common Issues and Solutions

### vsdbg Not Found
- **Solution**: Install VS Code C# extension or download vsdbg manually
- **Check**: `VSDBG_PATH` environment variable

### .NET SDK Not Found
- **Solution**: Install .NET SDK from https://dotnet.microsoft.com/download
- **Check**: `DOTNET_ROOT` environment variable

### Wrong .NET Version
- **Solution**: Update to .NET SDK 2.1 or higher
- **Check**: Run `dotnet --version`

## Next Steps

1. Review the detailed plan: `csharp-vsdbg-adapter-plan.md`
2. Follow the implementation checklist: `csharp-vsdbg-adapter-checklist.md`
3. Reference Python adapter for patterns
4. Start with Phase 1 (Foundation Setup)
