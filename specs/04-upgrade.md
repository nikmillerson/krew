# Krew Upgrade Command Specification

## Command

```bash
kubectl krew upgrade [PLUGIN...]
```

## Purpose

Upgrade installed plugins to newer versions available in the index. Without arguments, upgrades all installed plugins. With arguments, upgrades only specified plugins.

## Arguments

- **PLUGIN** (optional): One or more plugin names to upgrade
  - Must be simple names (no index/ prefix)
  - Using canonical form (index/plugin) returns error

## Flags

- `--no-update-index`: Skip automatic index update before upgrading (experimental)

## Behavior

### Pre-Upgrade

**Automatic Index Update:**
- Unless `--no-update-index` specified
- Runs full index update to get latest versions
- Failure in update doesn't block upgrade

### Upgrade Modes

**1. Upgrade All Plugins (No Arguments):**
```bash
kubectl krew upgrade
```
- Reads all installed plugin receipts
- Attempts to upgrade each one
- Ignores already-up-to-date plugins (silent skip)
- Continues on errors, warns at end if any failed
- Uses: `index-name/plugin-name` from receipt

**2. Upgrade Specific Plugins (With Arguments):**
```bash
kubectl krew upgrade plugin1 plugin2
```
- Only upgrades specified plugins
- Must use simple names (not index/plugin)
- Reads receipt to determine which index plugin came from
- Fails immediately on first error
- Canonical name not allowed (returns error)

### Plugin Resolution

For each plugin to upgrade:
1. Read installation receipt from `~/.krew/receipts/{plugin}.yaml`
2. Extract source index name from receipt
3. Construct canonical name: `{index}/{plugin}`
4. Load current manifest from index
5. Compare versions

### Version Comparison

Uses semantic versioning comparison:
- If index version > installed version: Upgrade needed
- If versions equal: Already upgraded (skip in "all" mode, warn in specific mode)
- If index version < installed version: No downgrade (treated as up-to-date)

### Special Cases

**Detached Plugins:**
- Plugins installed via `--manifest`
- Have source index "detached" in receipt
- Cannot be upgraded (no index to pull from)
- Logged: `Skipping upgrade for "{plugin}" because it was installed via manifest`
- Continue with next plugin

**Deleted Plugins:**
- Plugin no longer exists in index
- In "all" mode: Warn and skip
- In specific mode: Return error
- Message: `plugin "{name}" does not exist in the plugin index`

**Platform Mismatch:**
- Plugin manifest no longer has matching platform
- In "all" mode: Warn and continue
- In specific mode: Return error
- Happens if OS/ARCH support dropped

### Upgrade Process

Uses same installation process as `install` command:
1. Download new version archive
2. Verify SHA256
3. Extract to new versioned directory
4. Execute file operations
5. Update symlink to point to new version
6. Update receipt with new version/timestamp (preserves creationTimestamp)
7. Old version remains in store (at different version path)

## Security Requirements for Reimplementation

### Critical: Use Go 1.24 os.Root for All File Operations

**Upgrade uses the same file extraction and installation logic as install**, therefore:

**All file operations during upgrade MUST use Go 1.24's `os.Root` functionality** to prevent:
- Path traversal attacks (Zip Slip vulnerability)
- Symlink escape attacks
- Absolute path exploits during archive extraction
- Directory traversal during file operations

See **[12-archive-handling.md](./12-archive-handling.md)** and **[03-install.md](./03-install.md#security-requirements-for-reimplementation)** for complete security implementation details.

**Key points:**
- Create `os.Root` for new version directory during extraction
- Validate all archive entry names
- Skip or reject symlinks in archives
- Use root-scoped file operations for copying files

### Progress Output

For each plugin:
```
Upgrading plugin: plugin-name
Upgraded plugin: plugin-name
```

Or if already up-to-date (in "all" mode):
```
Upgrading plugin: plugin-name
Skipping plugin plugin-name, it is already on the newest version
```

### Error Messages

**All Mode Warnings:**
```
WARNING: failed to upgrade plugin "plugin-name", skipping (error: <details>)
WARNING: Some plugins failed to upgrade, check logs above.
```

**Specific Mode Errors:**
```
failed to upgrade plugin "plugin-name"
upgrade command does not support INDEX/PLUGIN syntax; just specify PLUGIN
```

### Security Notices

For plugins from default index only:
- Print security notice after upgrade
- Same as install command
- Not shown for custom index plugins

## Output Format

### Standard Error (stderr)

**Progress:**
```
Upgrading plugin: ctx
Upgraded plugin: ctx
```

**Skipped:**
```
Skipping plugin ctx, it is already on the newest version
Skipping upgrade for "custom-plugin" because it was installed via manifest
```

**Warnings (All Mode):**
```
WARNING: failed to upgrade plugin "foo/bar", skipping (error: plugin "bar" does not exist in the plugin index)
WARNING: Some plugins failed to upgrade, check logs above.
```

### Standard Output (stdout)

Silent (no output).

## Exit Codes

- **0**: All specified plugins upgraded or were already up-to-date
- **Non-zero**: At least one plugin failed to upgrade (in specific mode)
- **Non-zero**: One or more plugins failed in all mode (but continues upgrading others)

## Edge Cases & Error Handling

1. **No Plugins Installed**: Succeeds silently (nothing to upgrade)
2. **All Plugins Up-to-Date**: Shows skip messages, exits 0
3. **Plugin Receipt Missing**: Fails for that plugin
4. **Index Deleted After Install**: Plugin can't be upgraded
5. **Network Failure**: Fails upgrade for affected plugins
6. **SHA256 Mismatch**: Fails upgrade with security error
7. **Platform No Longer Supported**: All mode warns; specific mode fails
8. **Concurrent Upgrades**: Last one wins (no locking)
9. **Receipt Corruption**: Fails to read receipt, can't upgrade
10. **Index in Detached HEAD**: May affect version detection

## Examples

### Upgrade All Plugins
```bash
kubectl krew upgrade
```

### Upgrade Specific Plugin
```bash
kubectl krew upgrade krew
```

### Upgrade Multiple Plugins
```bash
kubectl krew upgrade ctx ns kubectx
```

### Upgrade Without Index Update
```bash
kubectl krew upgrade --no-update-index
```

### Invalid: Using Canonical Name
```bash
kubectl krew upgrade default/ctx
# Error: upgrade command does not support INDEX/PLUGIN syntax
```

## Testing Scenarios

### Existing Tests

1. **TestKrewUpgrade_WithoutIndexInitialized**: Upgrade works even without index
2. **TestKrewUpgrade**: Basic upgrade changes symlink to new version
3. **TestKrewUpgradePluginsFromCustomIndex**: Upgrade from non-default index
4. **TestKrewUpgradeSkipsManifestPlugin**: Detached plugins skipped
5. **TestKrewUpgradeNoSecurityWarningForCustomIndex**: No warning for custom indexes
6. **TestKrewUpgrade_CannotUseIndexSyntax**: Canonical form rejected
7. **TestKrewUpgradeUnsafe**: Reject unsafe plugin names
8. **TestKrewUpgradeWhenPlatformNoLongerMatches**: Handle platform removal
9. **TestKrewUpgrade_ValidPluginInstalledFromManifest**: Plugin removed from index

### Additional Test Scenarios

1. **Upgrade when already on latest**: Should show skip message
2. **Upgrade with no network**: Should fail gracefully
3. **Upgrade with corrupted index**: Should handle error
4. **Upgrade with version downgrade in index**: Should skip (no downgrade)
5. **Upgrade preserves creationTimestamp**: Verify receipt timestamp unchanged
6. **Upgrade changes symlink location**: New version path used
7. **Upgrade leaves old version in store**: Old version directory remains
8. **Upgrade with manifest version format change**: Handle version comparison
9. **Upgrade all with mix of successes/failures**: Partial upgrade completes
10. **Upgrade specific plugin that doesn't exist**: Clear error message
11. **Upgrade after manual index modification**: Detect version change
12. **Upgrade with very large new version**: Handle download properly
13. **Upgrade immediately after install**: Already up-to-date
14. **Upgrade after index rollback**: May downgrade (if newer in index)
15. **Upgrade with non-semver versions**: Handle comparison gracefully
16. **Upgrade with alpha/beta versions**: Semver pre-release handling
17. **Upgrade multiple plugins where some already up-to-date**: Mixed output
18. **Upgrade when receipt exists but binary deleted**: Re-create binary
19. **Upgrade with --no-update-index and stale index**: Uses stale versions
20. **Verify upgrade error count**: Correct count of failures in all mode
21. **Upgrade concurrently in two terminals**: Last writer wins
22. **Upgrade when download times out**: Appropriate timeout error
23. **Upgrade custom index plugin shows correct display name**: With index prefix
24. **Upgrade after custom index URL change**: Should still work
25. **Upgrade with broken symlink**: Should replace symlink

## Implementation Notes

- Uses `installation.Upgrade()` as main upgrade function
- Version comparison via `internal/installation/semver` package
- Semantic version parsing strict (some flexibility for non-semver)
- Receipt loading via `receipt.Load()`
- Upgrade reuses install logic (download, extract, link)
- creationTimestamp preserved from original installation
- Index name "detached" triggers special skip logic
- Error accumulation in "all" mode for final summary

## Related Commands

- `kubectl krew update`: Update indexes to see available upgrades
- `kubectl krew list`: See currently installed versions
- `kubectl krew install`: Install plugins
- `kubectl krew uninstall`: Remove plugins before/after upgrade
