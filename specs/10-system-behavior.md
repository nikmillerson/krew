# Krew System-Wide Behavior Specification

## Global Behaviors

### PATH Checking

**When:**
- Before every command execution (persistent pre-run hook)

**Behavior:**
1. Check if `~/.krew/bin` is in user's $PATH
2. If not present: Print warning to stderr
3. Warning includes setup instructions for user's shell
4. Command continues execution (warning only, not an error)

**Warning Format:**
```
WARNING: krew is not in your PATH. To add krew to your PATH, add the following line to your shell configuration:
  export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```

### Directory Initialization

**When:**
- Before every command execution (persistent pre-run hook)

**Behavior:**
Create required directories if they don't exist:
- `~/.krew/` (base path)
- `~/.krew/store/` (install path)
- `~/.krew/bin/` (binary symlinks)
- `~/.krew/index/` (index base)
- `~/.krew/receipts/` (installation receipts)

All directories created with mode `0755`.

### Migration System

**When:**
- Before every command execution (persistent pre-run hook)

**Migrations:**

1. **Receipts Migration (v0.2.x → v0.3.x):**
   - Checks if migration complete via marker file
   - If incomplete: Forces user to manually reinstall
   - Blocks all operations if old receipt format detected

2. **Index Migration:**
   - Checks if index structure needs updating
   - Automatically migrates if needed
   - Updates directory layout to new format

**Behavior:**
- Migrations run automatically and transparently
- Old versions blocked with helpful upgrade instructions
- Migration errors stop command execution

### Upgrade Notifications

**When:**
- After every command execution (persistent post-run hook)
- Randomly (~40% of executions to avoid API rate limits)
- Only for released versions (not development builds)
- Skipped if `KREW_NO_UPGRADE_CHECK` environment variable set

**Behavior:**
1. Background goroutine checks GitHub for latest release tag
2. Compare with current version using semver
3. If newer version available: Print notification to stderr
4. Non-blocking (doesn't delay command completion)

**Notification Format:**
```
A newer version of krew is available (v0.4.3 -> v0.4.4).
Run "kubectl krew upgrade" to get the newest version!
```

### Error Handling Philosophy

**Standard Output (stdout):**
- Used only for command results (lists, tables, etc.)
- Never used for errors or warnings

**Standard Error (stderr):**
- All progress messages
- All warnings
- All error messages
- Upgrade notifications

**Exit Codes:**
- 0: Success
- Non-zero: Error (specific code not guaranteed)

**Error Message Format:**
- Clear, actionable messages
- Include context about what failed
- Suggest remediation when possible

### Logging

**Verbosity Levels:**
Uses `klog` (Kubernetes logging) with `-v` flag:
- `-v=0`: Default (errors and warnings)
- `-v=1`: Info messages, warnings about issues
- `-v=2`: More detailed operational info
- `-v=3`: Debug info about flows
- `-v=4`: Detailed debug (file operations, git commands)

**Log Destination:**
- All logs go to stderr
- Controlled by `--logtostderr` flag (always true)

### Plugin Execution

**Not part of krew itself, but krew enables:**

After installation, plugins are executed as:
```bash
kubectl plugin-name [args]
```

**How it works:**
1. kubectl searches $PATH for `kubectl-plugin-name`
2. Finds symlink in `~/.krew/bin/`
3. Follows symlink to actual binary in `~/.krew/store/plugin/version/`
4. Executes binary with provided arguments

**krew is not involved in plugin execution** - this is pure kubectl behavior.

## Cross-Cutting Concerns

### Platform Detection

**OS Detection:**
- Linux: `GOOS=linux`
- macOS: `GOOS=darwin`
- Windows: `GOOS=windows`

**Architecture Detection:**
- `GOARCH`: amd64, arm64, 386, arm, etc.

**Override for Testing:**
- `KREW_OS` environment variable
- `KREW_ARCH` environment variable

**Platform Matching:**
- Uses Kubernetes label selectors
- Manifest specifies required OS/ARCH
- Matching logic in `installation.GetMatchingPlatform()`

### Index/Plugin Naming

**Index Names:**
- Default: "default" (reserved)
- Custom: any valid identifier
- Invalid: containing `/`, `..`, `.`, or path separators
- Validated by `indexoperations.IsValidIndexName()`

**Plugin Names:**
- Simple: `plugin-name` (letters, numbers, hyphens)
- Invalid: containing `/`, `\`, `..`, `.`, or parent directory references
- Validated by `validation.IsSafePluginName()`

**Canonical Names:**
- Format: `index-name/plugin-name`
- Used internally and in some commands
- Parsed by `pathutil.CanonicalPluginName()`

**Display Names:**
- Default index plugins: `plugin-name`
- Custom index plugins: `index-name/plugin-name`
- Detached plugins: `plugin-name` (with note about detached status)

### File Organization

**Versioned Installations:**
```
~/.krew/store/
├── plugin1/
│   ├── v1.0.0/  # Old version kept
│   │   └── ...
│   └── v1.1.0/  # Current version
│       └── ...
└── plugin2/
    └── v2.0.0/
        └── ...
```

**Symlink Structure:**
```
~/.krew/bin/kubectl-plugin1 -> ../store/plugin1/v1.1.0/plugin-binary
```

**Receipt Files:**
```
~/.krew/receipts/
├── plugin1.yaml  # Includes full manifest + metadata
└── plugin2.yaml
```

### Concurrency and Safety

**No Locking:**
- krew doesn't implement file locking
- Concurrent operations may conflict
- Last write wins for most operations

**Idempotent Operations:**
- Install of installed plugin: skipped
- Update of up-to-date index: no-op
- Uninstall of missing plugin: error

**Race Conditions:**
- Concurrent install/uninstall: undefined
- Concurrent upgrade: last one wins
- Concurrent index update: git may conflict

### Security Considerations

**Download Verification:**
- Mandatory SHA256 verification for all downloads
- Fails if checksum doesn't match
- No fallback or retry with different checksum

**Plugin Name Validation:**
- Prevents path traversal attacks
- Rejects `..`, `/`, `\` in names
- Validates before file operations

**Index Trust:**
- Default index: Trusted by krew maintainers
- Custom indexes: Warning displayed to user
- No code signing or GPG verification (relies on HTTPS + SHA256)

**Execution:**
- krew never executes plugin code
- Plugins executed by kubectl, not krew
- Binary permissions preserved from archive

### Terminal vs Non-Terminal Output

**Detection:**
Uses `isatty.IsTerminal()` and `isatty.IsCygwinTerminal()` to detect:
- Terminal: stdout connected to TTY
- Non-Terminal: stdout piped or redirected

**Behavior Changes:**

**list command:**
- Terminal: Table with headers and columns
- Non-terminal: Plain list of names (one per line)

**Search/List tables:**
- Always use table format (even when piped)
- list command special case for plugin name export

**Color Output:**
- Upgrade notifications use colored output
- Uses `github.com/fatih/color` library
- Respects NO_COLOR environment variable

## Environment Variables

**KREW_ROOT:**
- Override default `~/.krew` directory
- Useful for testing and isolation
- Must be absolute path

**KREW_NO_UPGRADE_CHECK:**
- Disable upgrade check (any value)
- Useful for CI/CD environments
- Improves command execution time

**KREW_BINARY:**
- Path to krew binary for testing
- Used by integration tests
- Not typically set by users

**KREW_OS / KREW_ARCH:**
- Override platform detection
- Used for testing different platforms
- Format: same as GOOS/GOARCH

**PATH:**
- Must include `~/.krew/bin` for plugins to work
- Checked and warned about by krew
- User responsible for configuration

## Common Error Patterns

### "Plugin not found in index"
- Cause: Typo in plugin name, or index not updated
- Solution: Run `kubectl krew update` then try again

### "No matching platform"
- Cause: Plugin doesn't support current OS/ARCH
- Solution: Check plugin info, may not be available

### "Index not initialized"
- Cause: First run before any update
- Solution: Run `kubectl krew update` to initialize

### "Permission denied"
- Cause: Insufficient permissions to write to ~/.krew
- Solution: Check directory ownership and permissions

### "Already installed"
- Cause: Plugin already installed (not an error)
- Solution: Use upgrade if newer version desired

### "Failed to update index"
- Cause: Network issues or git conflicts
- Solution: Check network, may need to delete and re-add index

## Testing Strategies

### Unit Tests
- Test individual functions and components
- Mock filesystem and network operations
- Fast execution, no external dependencies

### Integration Tests
- Test complete command flows
- Use real krew binary
- Isolated KREW_ROOT for each test
- May hit real network (default index)

### Test Helpers
Located in `integration_test/testutil_test.go`:
- `NewTest(t)`: Create isolated test environment
- `WithDefaultIndex()`: Clone real index
- `WithCustomIndexFromDefault()`: Create custom index
- Assertion helpers for file existence, plugin installation

## Backward Compatibility

### Receipt Format Changes
- Old format: Flat structure
- New format: Nested with status/source
- Migration handles upgrade automatically
- Old krew versions rejected

### Index Structure Changes
- Old: Single index at `~/.krew/index/`
- New: Multiple indexes at `~/.krew/index/{name}/`
- Migration creates "default" subdirectory

### Plugin Manifest Evolution
- API version in manifest: `krew.googlecontainertools.github.com/v1alpha2`
- Future versions may introduce breaking changes
- krew version must be compatible with manifest version

## Performance Considerations

### Index Update
- Git pull operation (network-bound)
- Can be slow with large history
- Incremental updates are fast

### Plugin Search
- Scans all indexes and manifests
- No caching (re-scans each time)
- O(n) where n = total plugins across indexes

### Plugin Installation
- Network download (dominant factor)
- SHA256 verification (CPU-bound)
- File extraction (I/O-bound)
- Typically seconds to minutes

### Receipt Reading
- Small YAML files
- Fast operation
- No optimization needed
