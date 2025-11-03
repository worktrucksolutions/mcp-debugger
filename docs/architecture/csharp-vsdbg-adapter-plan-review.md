# C# vsdbg Adapter Implementation Plan - Review

**Review Date**: 2025-11-03  
**Reviewer**: AI Assistant  
**Plan Version**: 1.0  
**Status**: Issues Found - Requires Revision

## Executive Summary

The implementation plan is comprehensive and well-structured, but contains several **critical technical inaccuracies** and **gaps** that need to be addressed before implementation begins. The plan demonstrates good understanding of the adapter pattern and MCP debugger architecture, but misunderstands some key aspects of vsdbg's operation and has inconsistencies with the actual codebase patterns.

**Overall Assessment**: 6/10 - Good structure but significant technical errors

---

## Critical Issues (Must Fix)

### 1. ❌ CRITICAL: Incorrect Export Pattern in index.ts

**Location**: Section "Implementation Details" → "Step-by-Step Implementation" (Line 214-218 in guide)

**Issue**: The plan states to export with this format:
```typescript
export default { name: 'example', factory: ExampleAdapterFactory };
```

**Reality**: Looking at the actual adapter implementations:
- `packages/adapter-python/src/index.ts` - NO default export
- `packages/adapter-mock/src/index.ts` - NO default export  
- `src/adapters/adapter-loader.ts` - Looks for NAMED export only

**Correct Pattern**:
```typescript
// src/index.ts - CORRECT implementation
export { CSharpAdapterFactory } from './csharp-adapter-factory.js';
export { CSharpDebugAdapter } from './csharp-debug-adapter.js';
// NO DEFAULT EXPORT NEEDED
```

The loader dynamically constructs the factory class name as `${Capitalized}AdapterFactory` and imports it as a named export.

**Impact**: HIGH - Would cause complete adapter load failure

---

### 2. ❌ CRITICAL: Misunderstanding of vsdbg Architecture

**Location**: Section "Adapter Command Building" (Lines 245-271)

**Issue**: The plan shows:
```bash
vsdbg --interpreter=vscode --connection=127.0.0.1:4711
```

**Problem**: This command-line interface is **unverified** and may not be accurate. vsdbg has different modes:

1. **stdin/stdout mode** (what debugpy adapter uses)
2. **TCP server mode** (listening on a port)
3. **VS Code integration mode**

**What Actually Happens**:
- debugpy has TWO components: `debugpy.adapter` (DAP proxy) + debug server
- vsdbg is a SINGLE component that IS the debug adapter
- vsdbg typically runs in **stdin/stdout mode** for DAP communication
- The "connection" parameter might be for connecting TO a debuggee, not FOR the adapter itself

**Required Investigation**:
- [ ] Verify vsdbg command-line parameters from official sources
- [ ] Determine if vsdbg needs to be launched differently than debugpy adapter
- [ ] Check if vsdbg supports TCP mode for DAP communication
- [ ] Review VS Code C# extension source code for actual launch command

**Recommended Approach**:
```typescript
// Likely more accurate for vsdbg
buildAdapterCommand(config: AdapterConfig): AdapterCommand {
  return {
    command: config.executablePath, // Path to vsdbg
    args: [
      // vsdbg operates in stdin/stdout mode by default
      // No --interpreter or --connection flags needed
    ],
    env: {
      ...process.env,
      DOTNET_CLI_UI_LANGUAGE: 'en-US',
    }
  };
}
```

**Impact**: HIGH - Incorrect command = adapter won't start

---

### 3. ⚠️ MAJOR: Missing Clarification on Adapter vs Runtime Confusion

**Location**: Throughout the plan

**Issue**: The plan conflates the .NET **runtime** (dotnet CLI) with the **debug adapter** (vsdbg).

**Clarification Needed**:
- **dotnet CLI**: Used to run the program being debugged (e.g., `dotnet MyApp.dll`)
- **vsdbg**: The debug adapter that sits between MCP server and the dotnet runtime
- **Relationship**: vsdbg launches dotnet as a child process, not the other way around

**What this means for implementation**:

1. The `executablePath` in `AdapterConfig` should point to **vsdbg**, not dotnet
2. The `program` in the launch config is the .NET DLL/EXE being debugged
3. vsdbg internally figures out how to launch the dotnet runtime

**Correction needed in**:
- Section "2. Executable Discovery" - Clarify what each executable is for
- Section "6. Adapter Command Building" - executablePath is vsdbg, not dotnet
- Throughout - Distinguish "runtime discovery" from "adapter discovery"

**Impact**: MEDIUM-HIGH - Conceptual confusion could lead to wrong implementation

---

### 4. ⚠️ MAJOR: Incomplete File Extensions Validation

**Location**: Section "Implementation Details" → CSharpAdapterFactory metadata (implied)

**Issue**: Plan mentions file extensions but doesn't specify them correctly.

**Reality**: C# files have multiple extensions:
- `.cs` - Primary C# source files
- `.csx` - C# script files
- `.razor` - Razor components (Blazor)
- `.cshtml` - Razor views (ASP.NET MVC)

**Correction**:
```typescript
getMetadata(): AdapterMetadata {
  return {
    // ...
    fileExtensions: ['.cs', '.csx'],  // Add these
    // ...
  };
}
```

**Impact**: LOW - Mostly cosmetic, but affects file type detection

---

### 5. ⚠️ MAJOR: vsdbg Installation Script May Be Outdated

**Location**: Section "3. vsdbg Installation" (Lines 144-147)

**Issue**: The plan shows:
```bash
curl -sSL https://aka.ms/getvsdbgsh | bash /dev/stdin -v latest -l ~/.vsdbg
```

**Problems**:
1. This URL may be outdated or deprecated
2. Microsoft may have changed installation methods
3. Different vsdbg versions for different .NET versions
4. No mention of architecture (x64, ARM64, etc.)

**Required Verification**:
- [ ] Check if `https://aka.ms/getvsdbgsh` still works
- [ ] Verify if VS Code C# extension is now the preferred distribution method
- [ ] Check if vsdbg versioning matters (.NET 6 vs 7 vs 8)
- [ ] Document architecture-specific installation

**Safer Approach**:
Recommend VS Code extension installation as primary method, with script as fallback:
```typescript
getInstallationInstructions(): string {
  return `vsdbg Installation Options:

1. Via VS Code (Recommended):
   - Install the "C#" extension from Microsoft
   - vsdbg is automatically installed
   - Location: ~/.vscode/extensions/ms-dotnettools.csharp-*/debugger/

2. Standalone (Advanced):
   - Visit: https://github.com/dotnet/vscode-csharp
   - Follow official installation instructions
   - Set VSDBG_PATH environment variable

3. Manual:
   - Download from VS Code marketplace or GitHub releases
   - Extract to ~/.vsdbg/
   - Ensure executable permissions (chmod +x)`;
}
```

**Impact**: MEDIUM - Users may fail to install vsdbg correctly

---

## Major Gaps and Omissions

### 6. ⚠️ MISSING: Launch vs Attach Configuration Distinction

**Location**: Section "4. Launch Configuration"

**Issue**: The plan shows launch configuration but doesn't clearly distinguish between:
- **Launch mode**: Start a new process and debug it
- **Attach mode**: Attach to an existing running process

**What's Missing**:
```typescript
interface CSharpAttachConfig extends LanguageSpecificLaunchConfig {
  processId: number | string;        // Process ID or "${command:pickProcess}"
  processName?: string;              // Or process name to search for
  justMyCode?: boolean;
}

// transformLaunchConfig needs to handle both cases
transformLaunchConfig(config: GenericLaunchConfig): LanguageSpecificLaunchConfig {
  if (config.processId !== undefined) {
    return this.transformAttachConfig(config);
  } else {
    return this.transformLaunchConfigForRun(config);
  }
}
```

**Impact**: MEDIUM - Attach debugging won't work without this

---

### 7. ⚠️ MISSING: Pre-Launch Task Configuration

**Location**: Section "4. Launch Configuration"

**Issue**: C# requires compilation before debugging, but there's no discussion of:
- Automatic build triggering (optional)
- Build verification before launch
- Handling of build failures

**Should Add**:
```typescript
interface CSharpLaunchConfig extends LanguageSpecificLaunchConfig {
  // ... existing fields ...
  
  preLaunchTask?: string;           // Name of task to run before launch
  suppressBuildBeforeDebug?: boolean; // Skip build check (dangerous)
}

// In adapter:
async validateBeforeLaunch(config: CSharpLaunchConfig): Promise<ValidationResult> {
  // Check if program DLL exists
  // Check if it's newer than source files
  // Warn if out of date
}
```

**Impact**: MEDIUM - Poor UX without build guidance

---

### 8. ⚠️ MISSING: Container/Docker Debugging Considerations

**Location**: Not addressed in plan

**Issue**: The mcp-debugger documentation emphasizes container support, but the C# plan doesn't address:
- vsdbg installation in Docker images
- Remote debugging setup
- Path mapping for containerized apps
- Volume mounts for source code

**Should Add Section**:
```markdown
### Container Deployment

**Docker Image Requirements**:
- .NET SDK or Runtime (depending on use case)
- vsdbg installed at known location
- Source files mounted or built into image

**Example Dockerfile**:
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0
RUN curl -sSL https://aka.ms/getvsdbgsh | bash /dev/stdin -v latest -l /vsdbg
ENV VSDBG_PATH=/vsdbg/vsdbg
WORKDIR /workspace
```

**Path Mapping**:
- Container path: `/workspace/src/Program.cs`
- Host path: `/home/user/project/src/Program.cs`
- Use `sourceFileMap` to map paths
```

**Impact**: MEDIUM - Container users will struggle

---

### 9. ⚠️ MISSING: Exception Breakpoint Filter Details

**Location**: Section "7. C#-Specific Features" → Exception Breakpoint Filters (Lines 287-303)

**Issue**: Shows only 2 filters, but .NET supports more granular exception filtering:

**Actually Available in vsdbg**:
```typescript
exceptionBreakpointFilters: [
  {
    filter: 'all',
    label: 'All Exceptions',
    default: false
  },
  {
    filter: 'user-unhandled',
    label: 'User-Unhandled Exceptions',
    default: true
  },
  {
    filter: 'never',
    label: 'Never Break',
    default: false
  },
  // Plus support for specific exception types:
  // "System.NullReferenceException", "System.ArgumentException", etc.
]
```

**Also Missing**: Support for exception conditions (break only if certain conditions met)

**Impact**: LOW-MEDIUM - Limited exception debugging capabilities

---

### 10. ⚠️ MISSING: Multi-Project Solution Support

**Location**: Section "Utility Modules" → project-utils.ts

**Issue**: Plan mentions `.csproj` and `.sln` but doesn't detail:
- How to handle .sln files with multiple projects
- Which project to debug when multiple exist
- Project reference resolution
- Multi-targeting (e.g., net6.0 and net7.0 in same project)

**Recommendation**: 
- Phase 1: Support single .csproj only
- Phase 2+: Add solution-level support
- Document this limitation clearly

**Impact**: LOW - Can be deferred to later phases

---

## Clarity Issues

### 11. ℹ️ UNCLEAR: Relationship Between Components

**Location**: Section "Architecture Overview"

**Issue**: Diagram doesn't show how components interact with the existing mcp-debugger infrastructure.

**Should Add**:
```
MCP Client (Claude)
    ↓ JSON-RPC
MCP Server (mcp-debugger)
    ↓
SessionManager
    ↓
ProxyManager
    ↓ spawns
ProxyWorker (separate process)
    ↓ spawns
vsdbg (debug adapter)
    ↓ launches & controls
dotnet runtime (debuggee)
    ↓ runs
MyApp.dll (user's program)
```

**Impact**: LOW - Helpful for understanding but not critical

---

### 12. ℹ️ UNCLEAR: Performance Characteristics

**Location**: Section "Success Criteria" → Performance

**Issue**: States "<100ms initialization" but doesn't explain what's being measured.

**Should Clarify**:
- Adapter construction: < 10ms
- Environment validation: < 100ms  
- vsdbg process spawn: < 500ms
- Initial DAP handshake: < 200ms
- Total session creation: < 1000ms

**Impact**: LOW - Clarifies expectations

---

### 13. ℹ️ UNCLEAR: xml2js Dependency Justification

**Location**: Section "Dependencies" (Line 383)

**Issue**: Lists `xml2js` for parsing .csproj but:
1. .csproj files are simple XML
2. May be overkill for initial implementation
3. Adds unnecessary dependency

**Recommendation**:
- Phase 1: Skip .csproj parsing entirely (just validate file exists)
- Phase 2: Add if needed for advanced features
- Consider lighter alternatives (fast-xml-parser)

**Impact**: LOW - Minor dependency bloat

---

## Positive Aspects ✅

### What the Plan Gets Right:

1. **✅ Overall Structure**: Excellent package layout following established patterns
2. **✅ Factory Pattern**: Correctly identified need for factory extending AdapterFactory
3. **✅ Phased Approach**: Sensible breakdown into implementation phases
4. **✅ Test Strategy**: Comprehensive testing approach (unit, integration, e2e)
5. **✅ Error Handling**: Good coverage of common error scenarios
6. **✅ Platform Awareness**: Addresses Windows, Linux, macOS differences
7. **✅ Documentation**: Thorough documentation plan
8. **✅ Examples**: Good variety of usage examples (console, web, attach)
9. **✅ Capabilities**: Comprehensive capability declaration
10. **✅ Path Discovery**: Good priority order for executable discovery

---

## Recommendations

### Immediate Actions (Before Implementation)

1. **Research Phase** (1-2 days):
   - [ ] Study VS Code C# extension source code for vsdbg usage
   - [ ] Test vsdbg manually to verify command-line interface
   - [ ] Confirm installation methods still work
   - [ ] Review actual DAP communication flow with vsdbg

2. **Plan Revision** (1 day):
   - [ ] Fix index.ts export pattern (Critical Issue #1)
   - [ ] Correct vsdbg command building (Critical Issue #2)
   - [ ] Clarify runtime vs adapter distinction (Critical Issue #3)
   - [ ] Add launch vs attach modes (Gap #6)
   - [ ] Add container considerations (Gap #8)

3. **Create Research Document**:
   ```
   docs/architecture/vsdbg-research-findings.md
   - Actual vsdbg command-line interface
   - DAP communication modes
   - Installation methods verification
   - Known limitations and quirks
   ```

### Phase Adjustments

**Phase 1 Should Be Simplified**:
- ❌ Remove: project-utils.ts (.csproj parsing)
- ❌ Remove: xml2js dependency
- ✅ Focus: Basic launch debugging only
- ✅ Focus: Simple path validation only

**Add Phase 0: Research & Validation** (before current Phase 1):
- Manually test vsdbg with sample C# project
- Document actual command-line usage
- Verify installation on all platforms
- Create minimal working example outside mcp-debugger

### Testing Recommendations

Add to test strategy:
```typescript
// Essential vsdbg behavior tests
describe('vsdbg integration', () => {
  it('should launch vsdbg with correct arguments', async () => {
    // Mock process spawn
    // Verify exact command and args
  });
  
  it('should handle vsdbg startup failures', async () => {
    // Test error scenarios
  });
  
  it('should communicate via stdin/stdout', async () => {
    // Test DAP message exchange
  });
});
```

---

## Severity Summary

| Severity | Count | Description |
|----------|-------|-------------|
| ❌ Critical | 3 | Must fix before implementation |
| ⚠️ Major | 7 | Should fix for quality implementation |
| ℹ️ Clarity | 3 | Would improve understanding |
| ✅ Correct | 10+ | Well-designed aspects |

---

## Revised Implementation Checklist

### Pre-Implementation (NEW - Phase 0)
- [ ] Research vsdbg command-line interface
- [ ] Test vsdbg manually with sample C# app
- [ ] Verify installation methods on each platform
- [ ] Document findings in vsdbg-research-findings.md
- [ ] Update plan based on research

### Phase 1: Core Infrastructure
- [ ] Add CSHARP to DebugLanguage enum
- [ ] Create package structure
- [ ] Implement dotnet-utils (runtime discovery) ⚠️ Note: Runtime, not adapter
- [ ] Implement vsdbg-utils (adapter discovery) ✅ Correct
- [ ] **REMOVE** project-utils.ts (defer to Phase 2+)
- [ ] Write unit tests for utilities
- [ ] **FIX** index.ts export pattern (named exports only)

### Phase 2: Adapter Implementation  
- [ ] Implement CSharpDebugAdapter class
- [ ] Implement environment validation
- [ ] Implement configuration transformation
  - [ ] Launch mode support
  - [ ] Attach mode support ⚠️ NEW
- [ ] Implement error handling
- [ ] **FIX** buildAdapterCommand with correct vsdbg args
- [ ] Write unit tests for adapter

### Phase 3: Factory and Integration
- [ ] Implement CSharpAdapterFactory
- [ ] Add metadata and capabilities
  - [ ] Include correct file extensions (.cs, .csx)
  - [ ] Add comprehensive exception filters
- [ ] Write integration tests
- [ ] Test with real C# projects (console app minimum)

### Phase 4: Documentation and Polish
- [ ] Write comprehensive README
- [ ] Add code examples (launch and attach)
- [ ] Update main documentation
- [ ] Create troubleshooting guide
- [ ] **ADD** Container deployment guide ⚠️ NEW
- [ ] Add to CI/CD pipeline

### Phase 5: Advanced Features (Future)
- [ ] Auto-install vsdbg feature
- [ ] Project build integration
- [ ] Solution file (.sln) support
- [ ] .csproj parsing (project-utils.ts)
- [ ] Remote debugging
- [ ] Hot reload support

---

## Conclusion

The plan demonstrates good understanding of the adapter pattern and overall architecture but contains several critical technical errors that would prevent successful implementation. **The plan should not be followed as-is.**

**Recommendation**: PAUSE implementation and conduct a research phase to:
1. Verify vsdbg command-line interface
2. Test vsdbg manually  
3. Fix critical issues #1-3
4. Address major gaps #6-8
5. Simplify Phase 1 scope

**Estimated Research Time**: 1-2 days  
**Estimated Plan Revision Time**: 4-6 hours  
**Revised Implementation Time**: 2-3 weeks (same as original with better foundation)

**Risk if proceeding without fixes**: HIGH - Core functionality would not work correctly.

---

**Next Steps**:
1. ✅ Review this critique
2. ⏸️ Pause current plan
3. 🔬 Conduct vsdbg research phase
4. ✏️ Revise plan with findings
5. ✅ Re-review revised plan
6. 🚀 Begin implementation

---

**Review Completed**: 2025-11-03  
**Confidence Level**: High (based on codebase analysis and pattern matching)  
**Recommendation**: Significant revision required before implementation
