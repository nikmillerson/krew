# Krew Uninstall Command Specification

## Command

```bash
kubectl krew uninstall PLUGIN...
```

Aliases: `remove`, `rm`

## Purpose

Remove one or more installed kubectl plugins from the system.

## Arguments

- **PLUGIN** (required): One or more plugin names to uninstall
  - Must be simple names (no index/ prefix)
  - Using canonical form (index/plugin) returns error
  - At least one plugin name required

## Flags

None.

## Behavior

### Pre-Uninstall

**Index Check:**
- Ensures at least one index is initialized
- If no index exists, fails with error
- (Note: This is somewhat arbitrary since uninstall doesn't need index)

**Name Validation:**
- Check for canonical form (index/plugin)
  - If detected: Error `uninstall command does not support INDEX/PLUGIN syntax; just specify PLUGIN`
- Check for unsafe characters (path traversal patterns)
  - If detected: Error `plugin name "{name}" not allowed`

### Uninstall Process

For each plugin (processed sequentially):

**1. Locate Installation:**
- Check for receipt file at `~/.krew/receipts/{plugin}.yaml`
- If not found: Error `plugin "{plugin}" is not installed`

**2. Remove Binary Symlink:**
- Delete symlink at `~/.krew/bin/kubectl-{plugin}`
- On Windows: May be a copy instead of symlink
- Ignore if already missing (idempotent)

**3. Remove Installation Files:**
- Delete plugin directory at `~/.krew/store/{plugin}/`
- Removes all versions (entire plugin directory)
- Recursive deletion

**4. Remove Receipt:**
- Delete receipt file at `~/.krew/receipts/{plugin}.yaml`

**5. Output:**
```
Uninstalled plugin: plugin-name
```

### Error Handling

**Immediate Failure:**
- Unlike install/upgrade, uninstall fails immediately on first error
- Does not continue with remaining plugins
- Returns error explaining failure

**Common Errors:**
- Plugin not installed
- Permission denied (can't delete files)
- Plugin files already deleted but receipt exists
- Receipt deleted but files remain (will still attempt cleanup)

## Security Requirements for Reimplementation

### Use Go 1.24 os.Root for Safe File Deletion

**All file deletion operations SHOULD use Go 1.24's `os.Root` functionality** to:
- Ensure deletions stay within krew's managed directories
- Prevent accidental deletion of system files if symlinks manipulated
- Validate paths before deletion

**Recommended approach:**

```go
// Create roots for each managed directory
binRoot := os.NewRoot(paths.BinPath())
storeRoot := os.NewRoot(paths.InstallPath())
receiptsRoot := os.NewRoot(paths.InstallReceiptsPath())

// Delete symlink safely
err := binRoot.Remove(fmt.Sprintf("kubectl-%s", pluginName))

// Delete installation directory safely
err := storeRoot.RemoveAll(pluginName)

// Delete receipt safely
err := receiptsRoot.Remove(fmt.Sprintf("%s.yaml", pluginName))
```

**Benefits:**
- Cannot delete files outside krew's directories
- Protection against symlink attacks where attacker replaces plugin directory with symlink to system directory
- Safer even if path validation is bypassed

**Validation:**
- Still validate plugin name for safety (no `..`, `/`, etc.)
- Use `os.Root` as defense-in-depth, not sole protection

See **[12-archive-handling.md](./12-archive-handling.md)** for general file operation security principles.

## Output Format

### Standard Error (stderr)

**Success:**
```
Uninstalled plugin: ctx
Uninstalled plugin: ns
```

**Errors:**
```
failed to uninstall plugin ctx: plugin "ctx" is not installed
uninstall command does not support INDEX/PLUGIN syntax; just specify PLUGIN
plugin name "../ctx" not allowed
```

### Standard Output (stdout)

Silent (no output).

## Exit Codes

- **0**: All specified plugins uninstalled successfully
- **Non-zero**: At least one plugin failed to uninstall (stops at first failure)

## Edge Cases & Error Handling

1. **Plugin Not Installed**: Fails with clear error
2. **Plugin Files Deleted Manually**: Still tries to clean up, may partially fail
3. **Receipt Missing But Files Present**: Can't uninstall (needs receipt)
4. **Broken Symlink**: Deletes anyway (idempotent)
5. **Plugin In Use**: OS may prevent deletion (Windows) or allow it (Unix)
6. **Insufficient Permissions**: Fails with permission error
7. **Multiple Versions Installed**: All versions removed (entire directory)
8. **Concurrent Uninstall**: Race condition possible, may fail
9. **Plugin Name Case Sensitivity**: Exact match required (case-sensitive on Unix)

## Examples

### Uninstall Single Plugin
```bash
kubectl krew uninstall ctx
```

### Uninstall Multiple Plugins
```bash
kubectl krew uninstall ctx ns kubectx
```

### Using Alias
```bash
kubectl krew rm ctx
kubectl krew remove ctx
```

### Invalid: Using Canonical Name
```bash
kubectl krew uninstall default/ctx
# Error: uninstall command does not support INDEX/PLUGIN syntax
```

### Invalid: No Arguments
```bash
kubectl krew uninstall
# Error: requires at least 1 arg(s), only received 0
```

## Testing Scenarios

### Existing Tests

Currently, no dedicated uninstall integration tests found in test files reviewed. Tests exist in `uninstall_test.go` but not examined in detail.

### Recommended Test Scenarios

1. **Uninstall installed plugin**: Should succeed cleanly
2. **Uninstall non-installed plugin**: Should fail with clear error
3. **Uninstall multiple plugins**: All removed successfully
4. **Uninstall with canonical name**: Should reject with error
5. **Uninstall with unsafe name**: Should reject
6. **Uninstall with no arguments**: Should show error
7. **Uninstall then reinstall**: Reinstall works correctly
8. **Uninstall plugin with broken symlink**: Should succeed
9. **Uninstall plugin where files were manually deleted**: Handle gracefully
10. **Uninstall plugin where only receipt exists**: Handle gracefully
11. **Uninstall while plugin binary is running**: Behavior varies by OS
12. **Uninstall with insufficient permissions**: Appropriate error
13. **Uninstall from custom index**: Works same as default index
14. **Uninstall detached plugin**: Works normally
15. **Verify all files removed after uninstall**: Complete cleanup
16. **Verify symlink removed after uninstall**: No broken links remain
17. **Verify receipt removed after uninstall**: Receipt file gone
18. **Uninstall first of multiple args succeeds, second fails**: First stays uninstalled
19. **Uninstall with spaces in plugin name**: Handled if valid name
20. **Concurrent uninstall of same plugin**: One succeeds, one may fail
21. **Uninstall krew itself**: Should work (self-removal)
22. **Uninstall using 'remove' alias**: Works identically
23. **Uninstall using 'rm' alias**: Works identically
24. **Uninstall plugin then search for it**: Shows as not installed
25. **Uninstall all plugins then upgrade**: Upgrade finds nothing

## Implementation Notes

- Uses `installation.Uninstall()` as main uninstall function
- Simple name validation prevents path traversal
- Canonical name check prevents confusion with index syntax
- Receipt file existence determines if plugin is "installed"
- Deletion is OS-specific (handles Windows vs Unix differences)
- No rollback mechanism (deletion is permanent)
- Cobra's `Args: cobra.MinimumNArgs(1)` enforces argument requirement

## Related Commands

- `kubectl krew list`: See what plugins are installed before uninstalling
- `kubectl krew install`: Reinstall after uninstall
- `kubectl krew search`: Find plugins after uninstalling
