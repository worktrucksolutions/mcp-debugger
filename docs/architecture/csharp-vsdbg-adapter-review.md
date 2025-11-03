# C# vsdbg Adapter Plan Review

## Review Summary

This document reviews the C# vsdbg adapter implementation plan for accuracy, clarity, gaps, and other issues.

## ✅ Strengths

1. **Comprehensive Structure**: Well-organized with clear phases and components
2. **Follows Patterns**: Correctly follows the Python adapter pattern
3. **Good Documentation**: Includes checklist and summary documents
4. **Platform Awareness**: Addresses Windows, Linux, and macOS differences

## ⚠️ Issues Found

### 1. **CRITICAL: vsdbg Communication Method**

**Issue**: The plan assumes vsdbg uses TCP (like debugpy), but vsdbg typically uses **stdio** for DAP communication.

**Current Plan Says**:
```typescript
buildAdapterCommand(config: AdapterConfig): AdapterCommand {
  return {
    command: config.executablePath,
    args: [
      '--host', config.adapterHost,
      '--port', config.adapterPort.toString()
    ],
    // ...
  };
}
```

**Reality**: vsdbg communicates via stdio (NDJSON), not TCP. The command should be:
```typescript
buildAdapterCommand(config: AdapterConfig): AdapterCommand {
  return {
    command: config.executablePath,
    args: [],  // No host/port args - uses stdio
    env: {
      ...process.env,
      // vsdbg-specific environment variables if needed
    }
  };
}
```

**Impact**: High - This would cause the adapter to fail completely
**Fix Required**: Update `buildAdapterCommand()` to use stdio instead of TCP

**Reference**: The adapter manager uses stdio: `stdio: ['ignore', 'inherit', 'inherit', 'ipc']`

### 2. **vsdbg Launch Configuration**

**Issue**: The plan doesn't specify how vsdbg receives the launch configuration.

**Gap**: vsdbg needs the launch configuration passed via DAP `launch` request, not command-line arguments. The adapter doesn't need to pass `--host`/`--port` because it uses stdio.

**Clarification Needed**: 
- vsdbg is invoked with no arguments (or minimal args like `--interpreter=vscode`)
- The launch configuration is sent via DAP protocol after initialization
- The adapter manager handles stdio communication

**Fix Required**: Update plan to reflect stdio-based communication

### 3. **Missing: vsdbg Command-Line Arguments**

**Issue**: The plan doesn't document actual vsdbg command-line arguments.

**Gap**: Need to specify:
- `--interpreter=vscode` (optional, for VS Code compatibility)
- Other vsdbg-specific arguments if any
- Environment variables that vsdbg recognizes

**Fix Required**: Research and document actual vsdbg invocation

### 4. **Incomplete vsdbg Discovery Paths**

**Issue**: The plan lists discovery paths but may be incomplete.

**Current Plan Lists**:
- VS Code extensions directory
- Visual Studio (Windows only)
- Environment variable `VSDBG_PATH`

**Missing**:
- VS Code may install vsdbg in different locations depending on version
- Check for `.NET Core Debugger` extension
- May need to search multiple extension versions
- Global vs user extension locations

**Fix Required**: Add more comprehensive search strategy

### 5. **vsdbg Version Detection**

**Issue**: The plan mentions `getVsdbgVersion()` but doesn't specify how to get it.

**Gap**: vsdbg doesn't have a `--version` flag like many tools. Version detection may require:
- Reading from extension manifest
- Parsing vsdbg executable metadata
- Checking extension directory version

**Fix Required**: Clarify version detection method

### 6. **Launch Configuration Type**

**Issue**: The plan shows `type: 'coreclr' | 'clr' | 'netcoredbg'` but `netcoredbg` is a different debugger.

**Clarification**: 
- `coreclr` = .NET Core / .NET 5+ (uses vsdbg)
- `clr` = .NET Framework (uses vsdbg on Windows)
- `netcoredbg` = Alternative debugger (not vsdbg)

**Fix Required**: Remove `netcoredbg` from type options or clarify it's not for vsdbg

### 7. **Missing: .NET Framework Detection**

**Issue**: The plan mentions .NET Framework but doesn't detail how to detect it.

**Gap**: 
- .NET Framework is Windows-only
- Detection requires checking registry or installed programs
- Different from .NET SDK detection
- May need separate utility function

**Fix Required**: Add .NET Framework detection details

### 8. **Incomplete Error Codes**

**Issue**: The plan lists error codes but may be missing some.

**Current Codes**:
- `VSDBG_NOT_FOUND`
- `DOTNET_SDK_NOT_FOUND`
- `DOTNET_VERSION_INCOMPATIBLE`
- `PROJECT_TYPE_UNSUPPORTED`

**Missing**:
- `VSDBG_VERSION_INCOMPATIBLE`
- `DOTNET_FRAMEWORK_NOT_FOUND` (for .NET Framework projects)
- `VSDBG_PERMISSION_DENIED` (executable permissions)
- `VSDBG_EXECUTION_FAILED`

**Fix Required**: Add missing error codes

### 9. **Missing: Stdio Mode Requirements**

**Issue**: The plan doesn't mention stdio mode requirements.

**Gap**: 
- vsdbg must not output non-JSON to stdout
- Need to handle stderr separately
- May need to configure vsdbg logging to avoid stdout pollution
- Reference: The guide mentions stdout must be NDJSON-only

**Fix Required**: Add stdio mode requirements section

### 10. **Missing: vsdbg Logging Configuration**

**Issue**: The plan doesn't specify how vsdbg logging is configured.

**Gap**: 
- vsdbg may output logs to stderr or files
- Need to configure logging to avoid stdout pollution
- May need environment variables for log location

**Fix Required**: Add logging configuration details

### 11. **Incomplete Capabilities List**

**Issue**: The capabilities declaration is incomplete.

**Current Plan Shows**: Partial list with "// ... more filters"

**Missing**:
- Complete list of all vsdbg capabilities
- Specific .NET exception filters
- Step granularity options
- Data breakpoint support (if any)

**Fix Required**: Complete the capabilities declaration

### 12. **Missing: .NET Project Type Detection**

**Issue**: The plan mentions `detectProjectType()` but doesn't detail implementation.

**Gap**:
- How to detect .NET Framework vs .NET Core vs .NET 5+
- Parse .csproj file? Use `dotnet` commands?
- May need to check `TargetFramework` in project file
- Could use `dotnet --info` or project inspection

**Fix Required**: Add implementation details for project type detection

### 13. **Missing: .NET SDK vs Runtime**

**Issue**: The plan mentions .NET SDK but may need to distinguish SDK vs Runtime.

**Clarification**:
- Debugging requires .NET Runtime (not just SDK)
- SDK includes Runtime, but detection should verify Runtime
- May need `dotnet --list-runtimes` check

**Fix Required**: Clarify SDK vs Runtime requirements

### 14. **Missing: Multi-Targeting Framework**

**Issue**: The plan doesn't address projects targeting multiple frameworks.

**Gap**:
- Projects can target multiple frameworks (TFM)
- Need to select which framework to debug
- May need user selection or default to first

**Fix Required**: Add multi-targeting handling (or note as future enhancement)

### 15. **Missing: .NET Core vs .NET 5+ Naming**

**Issue**: The plan uses ".NET Core" but .NET 5+ is now just ".NET".

**Clarification**:
- .NET Core 3.1 and earlier
- .NET 5, 6, 7, 8+ (dropped "Core" name)
- May need to update terminology

**Fix Required**: Update terminology for accuracy

### 16. **Missing: Launch Configuration Validation**

**Issue**: The plan doesn't specify validation of launch configuration.

**Gap**:
- Need to validate required fields (e.g., `program`)
- Validate .NET-specific options
- Check file paths exist
- Verify project compatibility

**Fix Required**: Add validation requirements

### 17. **Missing: Attach Mode Support**

**Issue**: The plan mentions `request: 'launch' | 'attach'` but doesn't detail attach mode.

**Gap**:
- How to attach to running process
- Process ID selection
- Process name matching
- Attach-specific configuration

**Fix Required**: Add attach mode details or note as future enhancement

### 18. **Missing: Symbol Loading**

**Issue**: The plan mentions symbol options but doesn't detail implementation.

**Gap**:
- How to configure symbol paths
- Symbol server configuration
- PDB file location
- Symbol loading options

**Fix Required**: Add symbol loading details or simplify

### 19. **Missing: .NET Core Debugger vs vsdbg**

**Issue**: The plan uses "vsdbg" but may need to clarify relationship to "CoreCLR Debugger".

**Clarification**:
- vsdbg is the VS Code debugger for .NET
- Also called ".NET Core Debugger" in some contexts
- May need to clarify naming

**Fix Required**: Clarify naming if needed

### 20. **Missing: Container Mode Considerations**

**Issue**: The plan doesn't mention container mode considerations.

**Gap**:
- How vsdbg discovery works in containers
- .NET SDK installation in containers
- Path resolution in Docker
- Volume mounting requirements

**Fix Required**: Add container mode section or note as future consideration

## 📋 Clarity Issues

### 1. **Unclear: Executable vs Module**

The plan shows `getAdapterModuleName()` returning `'vsdbg'`, but vsdbg is an executable, not a module like Python's `debugpy.adapter`. This is fine, but the naming might be confusing.

**Fix**: Clarify that vsdbg is an executable, not a module

### 2. **Unclear: Factory vs Adapter Validation**

The plan shows both factory validation and adapter validation. Need to clarify:
- Factory validation: Checks if adapter CAN be created (environment ready)
- Adapter validation: Checks if adapter IS ready (runtime checks)

**Fix**: Clarify distinction

### 3. **Unclear: Launch Config Transformation**

The plan shows transforming generic config to .NET config, but doesn't show:
- What happens if required fields are missing
- How to handle partial configs
- Default value application

**Fix**: Add transformation details

## 🔍 Gaps and Omissions

### 1. **Missing: Testing Strategy Details**

The plan mentions testing but doesn't detail:
- How to test without actual vsdbg installation
- Mock vsdbg for unit tests
- Integration test setup
- Platform-specific test requirements

### 2. **Missing: Performance Considerations**

The plan doesn't mention:
- Caching strategy for executable discovery
- Performance of path searching
- Startup time optimization

### 3. **Missing: Error Recovery**

The plan shows error codes but doesn't detail:
- Retry strategies
- Fallback options
- User guidance for recovery

### 4. **Missing: Logging Strategy**

The plan doesn't specify:
- What to log and when
- Log levels for different scenarios
- Integration with core logger

### 5. **Missing: Dependency Version Requirements**

The plan mentions .NET SDK 2.1+ but doesn't specify:
- Recommended version
- Known issues with specific versions
- Compatibility matrix

## ✅ Accuracy Checks

### 1. **Package Structure** ✅
Correct - follows Python adapter pattern

### 2. **Factory Pattern** ✅
Correct - extends `AdapterFactory` base class

### 3. **Interface Implementation** ✅
Correct - implements `IDebugAdapter` interface

### 4. **Export Format** ✅
Correct - follows naming convention

### 5. **Metadata Structure** ✅
Correct - matches `AdapterMetadata` interface

## 🔧 Recommendations

### Priority 1 (Critical - Must Fix)
1. **Fix vsdbg communication method** - Change from TCP to stdio
2. **Update buildAdapterCommand()** - Remove host/port args
3. **Add stdio mode requirements** - Document stdout purity

### Priority 2 (High - Should Fix)
4. **Complete vsdbg discovery paths** - Add more search locations
5. **Add vsdbg version detection** - Document method
6. **Fix launch configuration type** - Remove `netcoredbg`
7. **Add .NET Framework detection** - Detail implementation

### Priority 3 (Medium - Nice to Have)
8. **Complete capabilities list** - Fill in all capabilities
9. **Add project type detection** - Detail implementation
10. **Add error recovery** - Detail strategies
11. **Add testing strategy** - Detail test setup

### Priority 4 (Low - Future Enhancements)
12. **Add attach mode** - Detail attach configuration
13. **Add symbol loading** - Detail symbol configuration
14. **Add container mode** - Detail container considerations

## 📝 Action Items

1. Research actual vsdbg invocation and command-line arguments
2. Verify vsdbg uses stdio (not TCP) for DAP communication
3. Test vsdbg discovery paths on actual systems
4. Document vsdbg version detection method
5. Complete capabilities list with actual vsdbg capabilities
6. Add .NET Framework detection implementation details
7. Add stdio mode requirements and logging configuration
8. Complete error code list with all scenarios
9. Add launch configuration validation details
10. Add testing strategy with mock vsdbg if needed

## 📚 Additional Research Needed

1. **vsdbg Documentation**: Official docs on invocation and configuration
2. **VS Code C# Extension**: How it invokes vsdbg
3. **.NET SDK Detection**: Best practices for SDK detection
4. **DAP Protocol**: vsdbg-specific DAP capabilities and options
5. **Platform Differences**: Actual paths on Windows/Linux/macOS

## ✅ Conclusion

The plan is comprehensive and well-structured, but has several critical issues that need to be addressed:

1. **CRITICAL**: vsdbg communication method (stdio vs TCP)
2. **HIGH**: Incomplete vsdbg discovery and configuration
3. **MEDIUM**: Missing implementation details for several features

With these fixes, the plan will be ready for implementation.
