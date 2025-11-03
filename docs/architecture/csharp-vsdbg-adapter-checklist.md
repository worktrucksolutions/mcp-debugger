# C# vsdbg Adapter Implementation Checklist

## Phase 1: Foundation Setup

### Package Structure
- [ ] Create `packages/adapter-csharp/` directory
- [ ] Create `src/` directory structure
- [ ] Create `tests/` directory structure
- [ ] Create `src/utils/` directory

### Configuration Files
- [ ] Create `package.json` with correct metadata
- [ ] Create `tsconfig.json` extending root config
- [ ] Create `vitest.config.ts` for testing
- [ ] Create `README.md` with basic documentation

### Shared Package Updates
- [ ] Add `CSHARP = 'csharp'` to `DebugLanguage` enum in `packages/shared/src/models/index.ts`

## Phase 2: Utility Implementation

### vsdbg-utils.ts
- [ ] Implement `findVsdbgExecutable()` function
  - [ ] Windows path resolution (VS Code extensions, Visual Studio)
  - [ ] Linux path resolution (VS Code extensions)
  - [ ] macOS path resolution (VS Code extensions)
  - [ ] Environment variable support (`VSDBG_PATH`)
  - [ ] User-specified path support
  - [ ] Error handling for not found
- [ ] Implement `getVsdbgVersion()` function
- [ ] Implement `validateVsdbgInstallation()` function
- [ ] Implement `getVsdbgInstallPaths()` function (platform-specific)
- [ ] Add unit tests for vsdbg-utils

### dotnet-utils.ts
- [ ] Implement `findDotnetExecutable()` function
  - [ ] Environment variable support (`DOTNET_ROOT`)
  - [ ] PATH resolution
  - [ ] Platform-specific default paths
- [ ] Implement `getDotnetVersion()` function
- [ ] Implement `validateDotnetSdk()` function
  - [ ] Version parsing and validation
  - [ ] Minimum version check (2.1+)
- [ ] Implement `getDotnetInstallPaths()` function
- [ ] Implement `detectProjectType()` function (optional for Phase 1)
- [ ] Add unit tests for dotnet-utils

## Phase 3: Adapter Implementation

### CSharpDebugAdapter Class
- [ ] Implement class structure extending EventEmitter
- [ ] Implement `IDebugAdapter` interface
- [ ] Implement lifecycle methods:
  - [ ] `initialize()`
  - [ ] `dispose()`
- [ ] Implement state management:
  - [ ] `getState()`
  - [ ] `isReady()`
  - [ ] `getCurrentThreadId()`
  - [ ] `transitionTo()` (private helper)
- [ ] Implement environment validation:
  - [ ] `validateEnvironment()`
  - [ ] `getRequiredDependencies()`
- [ ] Implement executable management:
  - [ ] `resolveExecutablePath()`
  - [ ] `getDefaultExecutableName()` (return 'vsdbg')
  - [ ] `getExecutableSearchPaths()`
- [ ] Implement adapter configuration:
  - [ ] `buildAdapterCommand()`
  - [ ] `getAdapterModuleName()` (return 'vsdbg')
  - [ ] `getAdapterInstallCommand()` (return installation instructions)
- [ ] Implement debug configuration:
  - [ ] `transformLaunchConfig()`
  - [ ] `getDefaultLaunchConfig()`
- [ ] Implement DAP protocol operations:
  - [ ] `sendDapRequest()` (validation only)
  - [ ] `handleDapEvent()`
  - [ ] `handleDapResponse()`
- [ ] Implement connection management:
  - [ ] `connect()`
  - [ ] `disconnect()`
  - [ ] `isConnected()`
- [ ] Implement error handling:
  - [ ] `getInstallationInstructions()`
  - [ ] `getMissingExecutableError()`
  - [ ] `translateErrorMessage()`
- [ ] Implement feature support:
  - [ ] `supportsFeature()`
  - [ ] `getFeatureRequirements()`
  - [ ] `getCapabilities()` (full .NET capabilities)

## Phase 4: Factory Implementation

### CSharpAdapterFactory Class
- [ ] Extend `AdapterFactory` base class
- [ ] Implement constructor with metadata
- [ ] Implement `createAdapter()` method
- [ ] Implement `getMetadata()` method
  - [ ] Language: `DebugLanguage.CSHARP`
  - [ ] Display name: 'C#'
  - [ ] Version: '0.1.0'
  - [ ] Description
  - [ ] File extensions: ['.cs', '.csx', '.fs', '.fsx', '.vb']
  - [ ] Icon (optional)
- [ ] Implement `validate()` method
  - [ ] Check .NET SDK installation
  - [ ] Check vsdbg installation
  - [ ] Return validation result with errors/warnings

## Phase 5: Export and Integration

### index.ts
- [ ] Export `CSharpAdapterFactory`
- [ ] Export `CSharpDebugAdapter`
- [ ] Export utility functions
- [ ] Export types
- [ ] Default export: `{ name: 'csharp', factory: CSharpAdapterFactory }`

## Phase 6: Testing

### Unit Tests
- [ ] `csharp-adapter.test.ts`
  - [ ] Factory instantiation
  - [ ] Adapter creation
  - [ ] Export validation
- [ ] `vsdbg-utils.test.ts`
  - [ ] Windows path resolution
  - [ ] Linux path resolution
  - [ ] macOS path resolution
  - [ ] Version detection
  - [ ] Error handling
- [ ] `dotnet-utils.test.ts`
  - [ ] SDK detection
  - [ ] Version parsing
  - [ ] Error handling

### Integration Tests
- [ ] End-to-end debugging workflow
- [ ] Launch configuration validation
- [ ] Environment validation
- [ ] Error message verification

## Phase 7: Documentation

### README.md
- [ ] Package description
- [ ] Installation instructions
- [ ] Usage examples
- [ ] Requirements (vsdbg, .NET SDK)
- [ ] Platform-specific notes
- [ ] Troubleshooting section

### Code Documentation
- [ ] JSDoc comments for public APIs
- [ ] Inline comments for complex logic
- [ ] Type definitions

## Phase 8: Build and Verification

### Build
- [ ] Run `npm run build` in package directory
- [ ] Verify TypeScript compilation succeeds
- [ ] Check for type errors
- [ ] Verify dist files are generated

### Integration with Core
- [ ] Build shared package
- [ ] Verify package is discoverable by dynamic loader
- [ ] Test `list_supported_languages` includes 'csharp'
- [ ] Test `create_debug_session` with language 'csharp'

### Platform Testing
- [ ] Test on Windows
- [ ] Test on Linux
- [ ] Test on macOS (if available)

## Phase 9: Cleanup and Polish

### Code Quality
- [ ] Run linter and fix issues
- [ ] Ensure consistent code style
- [ ] Remove debug logs
- [ ] Remove commented code

### Error Messages
- [ ] Verify all error messages are user-friendly
- [ ] Add recovery instructions where applicable
- [ ] Test error scenarios

### Performance
- [ ] Verify executable discovery is cached appropriately
- [ ] Check for performance bottlenecks
- [ ] Optimize path resolution if needed

## Phase 10: Final Verification

### Checklist Verification
- [ ] All Phase 1-9 items completed
- [ ] All tests passing
- [ ] Documentation complete
- [ ] Code reviewed
- [ ] Ready for merge

## Notes

- Reference Python adapter implementation for patterns
- Follow adapter development guide: `docs/architecture/adapter-development-guide.md`
- Ensure stdout is clean in stdio mode (no logs to stdout)
- Test with actual .NET projects if possible
- Consider edge cases (missing SDK, wrong version, etc.)
