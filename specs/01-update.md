# Krew Update Command Specification

## Command

```bash
kubectl krew update
```

## Purpose

Synchronize the local copy of plugin indexes with their remote Git repositories. This updates the list of available plugins and their latest versions without installing or upgrading any plugins.

## Behavior

### Automatic Index Initialization

If no indexes exist:
1. Automatically add the default index (`https://github.com/kubernetes-sigs/krew-index.git`)
2. Print message: `Adding "default" plugin index from {URL}.`
3. Clone the repository to `~/.krew/index/default/`

### Index Update Process

For each configured index:
1. Perform `git pull` operation in the index directory
2. On success:
   - For default index: Print `Updated the local copy of plugin index.`
   - For custom indexes: Print `Updated the local copy of plugin index "{name}".`
3. On failure:
   - Log warning for that index
   - Continue with other indexes
   - Collect failed index names
   - Return error at end listing all failures

### Post-Update Information

After updating, if this is not the first run (plugins existed before):

**New Plugins Available:**
- Compare plugin lists before and after update
- Show plugins that didn't exist in the previous index
- Format:
  ```
  New plugins available:
    * plugin1
    * plugin2
  ```

**Upgrades Available:**
- Compare versions of installed plugins
- Show only installed plugins with version changes
- Format:
  ```
  Upgrades available for installed plugins:
    * plugin-name v1.0.0 -> v1.2.0
    * another-plugin v2.1.0 -> v2.2.0
  ```

### Implicit Invocation

The update command is automatically called before:
- `kubectl krew install` (unless `--no-update-index` flag is used)
- `kubectl krew upgrade` (unless `--no-update-index` flag is used)

## Output Format

### Success Output (stderr)

```
Updated the local copy of plugin index.
```

Or for custom indexes:
```
Updated the local copy of plugin index "custom-name".
```

With optional informational sections if applicable.

### Error Output (stderr)

```
failed to update the following indexes: index1, index2
```

Partial failures still update successful indexes and return error at end.

## Exit Codes

- **0**: All indexes updated successfully (or no indexes to update)
- **Non-zero**: At least one index failed to update

## Edge Cases & Error Handling

1. **No Git Available**: Fails with error about git requirement
2. **Network Failure**: Index update fails, error logged, continues with other indexes
3. **Corrupted Index Repository**:
   - Git operations fail
   - User must manually delete and re-add index
4. **Empty Directory in index/**:
   - Should be ignored or cleaned up
   - Only valid git repositories should be processed
5. **First Time Run**:
   - No "New plugins available" or "Upgrades available" sections
   - Only shows update success message
6. **Multiple Indexes with Same Plugin**:
   - Both listed in output
   - Distinguished by index name prefix

## Testing Scenarios

### Existing Tests

1. **TestKrewUpdate**: Basic update with automatic default index initialization
2. **TestKrewUpdateMultipleIndexes**: Update multiple indexes simultaneously
3. **TestKrewUpdateFailedIndex**: Handle index update failures gracefully
4. **TestKrewUpdateListsNewPlugins**: Show new plugins after update
5. **TestKrewUpdateListsUpgradesAvailable**: Show upgrade-able installed plugins

### Additional Test Scenarios

1. **Update with no network connectivity**: Should fail gracefully with clear error
2. **Update with corrupted git repository**: Should report error for specific index
3. **Update with diverged git history**: Should handle git conflicts
4. **Update with very old index (many commits behind)**: Should handle efficiently
5. **Update with GPG-signed commits**: Should verify signatures if configured
6. **Update when index repository has been force-pushed**: Handle non-fast-forward
7. **Concurrent update operations**: Handle file locking properly
8. **Update after manual git operations in index**: Should not corrupt state
9. **Update with slow network**: Should have reasonable timeout
10. **Update with renamed/moved plugins in index**: Show appropriate messages
11. **Update when index URL has changed**: Should fail with helpful message
12. **Update with detached HEAD in index**: Should attempt to recover
13. **Update showing both new plugins and upgrades**: Format correctly
14. **Update with no installed plugins**: Should not show upgrade section
15. **Re-run update immediately**: Should show "already up-to-date" behavior

## Implementation Notes

- Uses `internal/gitutil.EnsureUpdated()` for git operations
- Plugin list loading tolerates parse errors (logged, not fatal)
- Updates are performed sequentially, not in parallel
- No locking mechanism between concurrent krew processes
- Index paths constructed via `environment.Paths` abstraction

## Related Commands

- `kubectl krew index list`: View configured indexes
- `kubectl krew upgrade`: Install updates shown by this command
- `kubectl krew install`: Implicitly runs update first
