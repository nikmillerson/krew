# Krew Version Command Specification

## Command

```bash
kubectl krew version
```

## Purpose

Display version information and diagnostic details about the krew installation, including build information and directory paths.

## Arguments

None.

## Flags

None.

## Behavior

### Information Displayed

Outputs a table with the following fields:

**Build Information:**
- **GitTag**: Release version tag (e.g., "v0.4.4")
  - From build-time variable
  - Empty or "unknown" for development builds
- **GitCommit**: Git commit SHA that krew was built from
  - Short or full commit hash
  - From build-time variable

**Index Configuration:**
- **IndexURI**: Default index repository URL
  - Value: `https://github.com/kubernetes-sigs/krew-index.git`
  - Constant from `pkg/constants`

**Directory Paths:**
- **BasePath**: Root directory for krew
  - Default: `~/.krew` (or `$KREW_ROOT` if set)
  - Absolute path
- **IndexPath**: Directory containing default index
  - Path: `{BasePath}/index/default`
  - Where plugin manifests are stored
- **InstallPath**: Directory where plugins are installed
  - Path: `{BasePath}/store`
  - Contains versioned plugin directories
- **BinPath**: Directory containing plugin symlinks
  - Path: `{BasePath}/bin`
  - Should be in user's $PATH

**Platform Information:**
- **DetectedPlatform**: Current OS and architecture
  - Format: `{os}/{arch}` (e.g., "linux/amd64", "darwin/arm64", "windows/amd64")
  - Detected at runtime

### Output Format

Table with two columns: `OPTION` and `VALUE`

```
OPTION             VALUE
GitTag             v0.4.4
GitCommit          abc123def
IndexURI           https://github.com/kubernetes-sigs/krew-index.git
BasePath           /Users/username/.krew
IndexPath          /Users/username/.krew/index/default
InstallPath        /Users/username/.krew/store
BinPath            /Users/username/.krew/bin
DetectedPlatform   darwin/arm64
```

## Exit Codes

- **0**: Always succeeds (version info always available)

## Edge Cases & Error Handling

1. **Development Build**: GitTag may be empty or "unknown"
2. **Custom KREW_ROOT**: Paths reflect custom root
3. **Symlinked Directories**: Paths shown as-is (not resolved)
4. **Non-existent Directories**: Paths shown even if directories don't exist yet
5. **Permission Issues**: Version still displays (doesn't test directory access)

## Examples

### Basic Usage
```bash
kubectl krew version
```

### Piping to Check Version
```bash
kubectl krew version | grep GitTag
```

### Checking Custom KREW_ROOT
```bash
KREW_ROOT=/custom/path kubectl krew version
```

### Getting Path Information
```bash
kubectl krew version | grep BinPath
```

## Use Cases

1. **Bug Reports**: Users include version output
2. **Debugging**: Check configured paths
3. **Verification**: Confirm krew installation location
4. **Documentation**: Show installed version
5. **Automation**: Parse version for scripts
6. **Support**: Verify correct PATH configuration

## Testing Scenarios

### Recommended Test Scenarios

1. **Version output format**: Verify table structure
2. **Version with default paths**: Correct default paths shown
3. **Version with KREW_ROOT**: Custom root reflected in paths
4. **Version with development build**: GitTag shows appropriately
5. **Version output is stable**: Consistent across runs
6. **Version shows correct platform**: Matches runtime OS/ARCH
7. **Version IndexURI**: Shows default index URL
8. **Version GitCommit**: Non-empty for releases
9. **Version all paths absolute**: No relative paths shown
10. **Version output to file**: Correctly redirected
11. **Version in different locales**: Works correctly
12. **Version when krew not fully initialized**: Still works
13. **Version when BasePath doesn't exist**: Shows path anyway
14. **Version table alignment**: Columns properly aligned
15. **Parse version programmatically**: Output format stable for parsing
16. **Version with very long custom path**: Long paths display correctly
17. **Version on Windows**: Backslashes in paths if applicable
18. **Version shows symlink resolution**: Or shows as-is
19. **Compare version across different systems**: Platform varies appropriately
20. **Version when binary is symlinked**: Still shows correct info

## Implementation Notes

- Uses `version.GitTag()` and `version.GitCommit()` for build info
  - Set via `-ldflags` at build time
  - From `internal/version` package
- Uses `index.DefaultIndex()` for index URI
  - Constant: `constants.DefaultIndexURI`
- Uses `environment.Paths` for directory paths
  - Reads `KREW_ROOT` environment variable
  - Falls back to `~/.krew`
- Uses `installation.OSArch()` for platform detection
  - Returns `runtime.GOOS` and `runtime.GOARCH`
- Output via `printTable()` helper (same as list command)
- No external dependencies or network calls
- Pure diagnostic output (read-only)

## Related Commands

- `kubectl krew update`: Update index at IndexPath
- `kubectl krew list`: See plugins in InstallPath
- `kubectl krew install`: Add plugins to BinPath
