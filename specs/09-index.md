# Krew Index Management Commands Specification

## Command Group

```bash
kubectl krew index [subcommand]
```

## Purpose

Manage custom plugin indexes - the Git repositories that contain plugin manifests. Allows users to add, list, and remove plugin sources beyond the default krew index.

## Subcommands

- `kubectl krew index list` (alias: `ls`)
- `kubectl krew index add NAME URL`
- `kubectl krew index remove NAME` (alias: `rm`)

---

## kubectl krew index list

### Purpose

Display all configured plugin indexes with their names and URLs.

### Arguments

None.

### Flags

None.

### Behavior

1. Scan `~/.krew/index/` directory for subdirectories
2. For each directory:
   - Check if it's a valid git repository
   - Read remote URL from git config
   - Collect index name and URL
3. Display in table format sorted by name

### Output Format

Table with columns: `INDEX` and `URL`

```
INDEX     URL
default   https://github.com/kubernetes-sigs/krew-index.git
myindex   https://github.com/myorg/krew-plugins.git
```

### Exit Codes

- **0**: Success
- **Non-zero**: Error listing indexes

### Edge Cases

1. **No Indexes**: Empty table (only headers)
2. **Non-Git Directory**: Should skip or error
3. **Git Repo Without Remote**: May show empty URL
4. **Corrupted Git Repo**: Error or skip

### Examples

```bash
kubectl krew index list
kubectl krew index ls  # alias
```

---

## kubectl krew index add

### Purpose

Add a new custom plugin index by cloning its Git repository.

### Arguments

- **NAME** (required): Name for the index (must be valid identifier)
- **URL** (required): Git repository URL

### Flags

None.

### Behavior

**Name Validation:**
- Check if name is valid using `indexoperations.IsValidIndexName()`
- Invalid names: containing `/`, `..`, `.`, or other special characters
- Reserved name: "default" (can be re-added if deleted)

**Index Addition:**
1. Check if index with same name already exists
   - If exists: Error `index "{name}" already exists`
2. Clone git repository from URL to `~/.krew/index/{name}/`
3. Verify repository structure (should have `plugins/` directory)
4. On success: Print security warning about custom indexes

**Security Warning:**
After successful addition, always display:
```
You have added a new index from "{url}"
The plugins in this index are not audited for security by the Krew maintainers.
Install them at your own risk.
```

### Output Format

**Success (stderr):**
```
You have added a new index from "https://github.com/myorg/plugins.git"
The plugins in this index are not audited for security by the Krew maintainers.
Install them at your own risk.
```

**Error (stderr):**
```
index "myindex" already exists
invalid index name
failed to clone index repository
```

### Exit Codes

- **0**: Index added successfully
- **Non-zero**: Error adding index

### Edge Cases

1. **Name Already Exists**: Error without cloning
2. **Invalid Git URL**: Git clone fails with error
3. **Network Failure**: Clone fails
4. **Empty Repository**: May succeed but have no plugins
5. **Repository Without plugins/ Directory**: May succeed but plugins won't be found
6. **Name Conflicts with Future Reserved Names**: Allowed if currently valid
7. **URL with Authentication Required**: Git may prompt for credentials

### Examples

```bash
kubectl krew index add default https://github.com/kubernetes-sigs/krew-index.git
kubectl krew index add mycompany https://github.com/mycompany/krew-plugins.git
kubectl krew index add private git@github.com:private/plugins.git
```

---

## kubectl krew index remove

### Purpose

Remove a configured custom plugin index. Warns if plugins from that index are still installed.

### Arguments

- **NAME** (required): Name of index to remove

### Flags

- `--force`: Remove index even if plugins from it are installed

### Behavior

**Name Validation:**
- Check if name is valid using `indexoperations.IsValidIndexName()`
- If invalid: Error `invalid index name`

**Safety Check:**
1. Query all installed plugins to find those from this index
2. If any plugins found from this index AND `--force` not specified:
   - List affected plugins in warning
   - Error: `there are still plugins installed from this index`
   - Do not remove index
3. If `--force` specified OR no plugins installed: Proceed with removal

**Removal Process:**
1. Delete directory at `~/.krew/index/{name}/`
2. Recursive deletion of entire directory
3. If directory doesn't exist:
   - With `--force`: Succeed silently
   - Without `--force`: Error `index "{name}" does not exist`

**Warning Format (when plugins installed):**
```
Plugins [plugin1, plugin2, plugin3] are still installed from index "name"!
Removing indexes while there are plugins installed from is not recommended
(you can use --force to ignore this check).
```

### Output Format

**Success (silent):**
No output on success.

**Warning (stderr):**
```
Plugins [ctx, ns] are still installed from index "myindex"!
Removing indexes while there are plugins installed from is not recommended
(you can use --force to ignore this check).
```

**Error (stderr):**
```
index "myindex" does not exist
invalid index name
there are still plugins installed from this index
```

### Exit Codes

- **0**: Index removed successfully
- **Non-zero**: Error removing index or safety check failed

### Edge Cases

1. **Index Doesn't Exist**: Error (unless --force)
2. **Plugins Still Installed**: Warning + error (unless --force)
3. **Default Index Removal**: Allowed (can break plugin installations)
4. **Index Directory Manually Deleted**: With --force succeeds, without --force errors
5. **Insufficient Permissions**: Error during deletion
6. **Concurrent Remove Operations**: Race condition possible
7. **Index With Git Lock Files**: May fail deletion

### Examples

```bash
kubectl krew index remove myindex
kubectl krew index rm myindex  # alias
kubectl krew index remove myindex --force
kubectl krew index remove default  # Dangerous but allowed
```

---

## Testing Scenarios

### Existing Tests

Based on `index_test.go` analysis (not fully reviewed):
- Tests for basic index operations exist

### Recommended Test Scenarios

**list:**
1. List with default index only
2. List with no indexes
3. List with multiple indexes
4. List with corrupted git repo
5. List sorted by name
6. List with very long URLs

**add:**
1. Add valid custom index
2. Add with invalid name (special chars)
3. Add with name that already exists
4. Add with invalid git URL
5. Add with network failure
6. Add default index (re-adding after removal)
7. Add from private git repo (SSH)
8. Add with very long name
9. Verify security warning displayed
10. Add then list shows new index

**remove:**
1. Remove index with no installed plugins
2. Remove index with installed plugins (without --force)
3. Remove index with installed plugins (with --force)
4. Remove nonexistent index (without --force)
5. Remove nonexistent index (with --force)
6. Remove default index
7. Remove then add same index
8. Remove with insufficient permissions
9. Verify plugin list after force remove
10. Remove all indexes then add back

**Integration:**
1. Add index, install plugin, remove index
2. Add multiple indexes, list shows all
3. Remove index doesn't affect other indexes
4. Add, remove, add same index
5. Install from custom index, verify source
6. Search shows plugins from all indexes
7. Remove index, plugins remain functional
8. Remove index with --force, upgrade fails gracefully

## Implementation Notes

- Index operations in `internal/index/indexoperations` package
- Name validation prevents path traversal and special characters
- Safety check queries `installation.InstalledPluginsFromIndex()`
- Git operations use `gitutil.IsGitCloned()` for validation
- Index directory structure: `~/.krew/index/{name}/`
- Plugin manifests expected at: `~/.krew/index/{name}/plugins/*.yaml`
- `--force` flag on remove bypasses safety checks (not recommended)
- Security warning uses `internal.PrintWarning()` helper

## Related Commands

- `kubectl krew update`: Updates all indexes including custom ones
- `kubectl krew search`: Shows plugins from all indexes
- `kubectl krew install INDEX/PLUGIN`: Install from specific index
