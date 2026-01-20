# Archive Extraction and File Operations Specification

## Purpose

This document specifies how krew handles archive extraction and file operations during plugin installation and upgrade. It details the exact mechanics used in the current implementation and security requirements for reimplementation.

## Archive Format Support

### Supported Formats

**TAR.GZ (.tar.gz, .tgz):**
- Most common format for Unix/Linux/macOS
- Preserves file permissions
- Preserves symlinks (though symlinks in archives are not recommended)
- Uses standard library: `archive/tar` and `compress/gzip`

**ZIP (.zip):**
- Common format for Windows
- May not preserve Unix permissions correctly
- Uses standard library: `archive/zip`

### Format Detection

Detection based on file extension from URI:
- `.tar.gz` or `.tgz` → TAR.GZ handler
- `.zip` → ZIP handler
- Other extensions → Error

No content-based detection (magic bytes) performed.

## Current Implementation Mechanics

### Download Phase

Located in `internal/download/fetch.go`:

1. **HTTP GET Request:**
   - Create HTTP GET request for plugin URI
   - No custom headers
   - Follow redirects (Go's default behavior)
   - No timeout specified (uses system defaults)

2. **Stream to Temporary File:**
   - Download to temporary file in `os.TempDir()`
   - File created with pattern: `krew-download-*`
   - Streaming download (not loaded into memory)
   - File created with mode `0600` (owner read/write only)

3. **SHA256 Verification:**
   - Compute SHA256 hash during download (streaming)
   - Compare with manifest's expected hash
   - Case-insensitive comparison
   - On mismatch: Delete temp file, return error
   - Uses `crypto/sha256` from standard library

### Extraction Phase

Located in `internal/download/downloader.go` and internal extraction logic:

**For TAR.GZ:**

1. **Open Archive:**
   - Open downloaded file
   - Create gzip reader: `gzip.NewReader(file)`
   - Create tar reader: `tar.NewReader(gzipReader)`

2. **Iterate Entries:**
   ```go
   for {
       header, err := tarReader.Next()
       if err == io.EOF {
           break
       }
       // Process header
   }
   ```

3. **Extract Each Entry:**
   - Read header to get: name, size, mode, type
   - For regular files:
     - Create parent directories as needed
     - Create file with mode from header
     - Copy content from tar reader to file
     - Close file
   - For directories:
     - Create directory with mode from header
   - For symlinks:
     - **Current implementation may create symlinks** (security issue)
     - Linkname from header used as-is

4. **Path Handling:**
   - Paths are relative to extraction root
   - **Current implementation joins paths with `filepath.Join()`**
   - **No explicit validation against path traversal via `..`**
   - **No validation against absolute paths in archive**

**For ZIP:**

1. **Open Archive:**
   - Open downloaded file
   - Open as zip: `zip.OpenReader(filename)`

2. **Iterate Entries:**
   ```go
   for _, file := range zipReader.File {
       // Process file
   }
   ```

3. **Extract Each Entry:**
   - For each file in zip:
     - Open file from archive
     - Read file info (name, size, mode)
     - Create parent directories
     - Create output file
     - Copy content
     - Close file

4. **Permission Handling:**
   - ZIP may not preserve Unix permissions correctly
   - Uses default permissions if not set in archive
   - On Windows: Permissions largely ignored

### File Operations Phase

Located in `internal/installation/move.go`:

After extraction, manifest's `files` operations are executed:

1. **Glob Pattern Matching:**
   - For each file operation with `from` pattern
   - Use `filepath.Glob()` to match files
   - Supports: `*`, `**`, `?`, `[...]`, `{...}`

2. **Move/Copy Operations:**
   - For each matched file:
     - Compute destination path (join `to` with filename)
     - Create parent directories if needed
     - **Current: Copy file then delete source** (not atomic move)
     - Preserve file mode/permissions

3. **Error Handling:**
   - Fail on first file operation error
   - No rollback mechanism
   - Partial state possible on failure

## Security Vulnerabilities in Current Implementation

### Path Traversal (Zip Slip)

**Current Issues:**
1. **No validation of entry names** before extraction
2. **Allows `..` in paths** - can extract outside intended directory
3. **Allows absolute paths** - can write to any location
4. **Symlinks followed** - can be used to escape directory

**Example Attack:**
```
Archive containing:
  ../../etc/cron.d/malicious
  symlink -> /etc/
  symlink/cron.d/malicious
```

**Current Defense:**
- None in extraction code
- Relies on manifest validation (plugin names checked)
- But manifest files operations not validated for traversal

### Symlink Attacks

**Current Issues:**
1. **Symlinks preserved** from archive
2. **Symlinks followed** during operations
3. Can point outside installation directory
4. Can be used for privilege escalation

**Example Attack:**
```
Archive containing:
  symlink -> /root/.ssh/
  files/authorized_keys
Then file operation: from symlink/authorized_keys
```

### Race Conditions

**Current Issues:**
1. **TOCTOU vulnerabilities** - time of check vs time of use
2. **No atomic operations** for file moves
3. **Concurrent installs** can corrupt state

## Required Security Improvements for Reimplementation

### Use Go 1.24 os.Root for All File Operations

**Go 1.24 introduces `os.Root` functionality** for secure file operations:

```go
import "os"

// Create Root for installation directory
root := os.NewRoot(installDir)

// All operations scoped to this directory
file, err := root.Open("file.txt")
file, err := root.Create("newfile.txt")
dir, err := root.OpenDir("subdir")
```

**Benefits:**
- **Prevents path traversal** - cannot escape root directory
- **Prevents symlink escapes** - symlinks constrained to root
- **Prevents absolute path exploits** - absolute paths rejected
- **Safe by default** - no additional validation needed

### Implementation Requirements

**For Archive Extraction:**

```go
// Pseudo-code for secure extraction
func ExtractArchive(archivePath string, destDir string) error {
    // Create Root for destination directory
    root := os.NewRoot(destDir)

    // Open archive
    archive := openArchive(archivePath)

    for entry := range archive.Entries() {
        // Validate entry name (no .., no absolute paths)
        if !isValidEntryName(entry.Name) {
            return fmt.Errorf("invalid entry name: %s", entry.Name)
        }

        // Skip symlinks entirely (security)
        if entry.IsSymlink() {
            log.Warning("skipping symlink: %s", entry.Name)
            continue
        }

        switch entry.Type {
        case Directory:
            // Create directory within root
            err := root.MkdirAll(entry.Name, entry.Mode)

        case RegularFile:
            // Create parent directories
            err := root.MkdirAll(filepath.Dir(entry.Name), 0755)

            // Create and write file within root
            file, err := root.Create(entry.Name)
            if err != nil {
                return err
            }
            defer file.Close()

            // Set permissions
            err = file.Chmod(entry.Mode)

            // Copy content
            _, err = io.Copy(file, entry.Reader())
        }
    }

    return nil
}
```

**For File Operations:**

```go
// Pseudo-code for secure file operations
func ExecuteFileOperations(ops []FileOperation, installDir string) error {
    // Create Root for installation directory
    root := os.NewRoot(installDir)

    for _, op := range ops {
        // Glob within root
        matches, err := root.Glob(op.From)

        for _, match := range matches {
            // Compute destination (still within root)
            dest := filepath.Join(op.To, filepath.Base(match))

            // Open source file within root
            src, err := root.Open(match)
            defer src.Close()

            // Create destination within root
            dst, err := root.Create(dest)
            defer dst.Close()

            // Copy
            _, err = io.Copy(dst, src)

            // Preserve permissions
            info, _ := src.Stat()
            dst.Chmod(info.Mode())
        }
    }

    return nil
}
```

### Validation Requirements

**Entry Name Validation:**

Must reject:
- Names containing `..` components
- Absolute paths (starting with `/` or drive letter on Windows)
- Names with backslashes on Unix (could be path traversal on Windows)
- Empty names
- Names longer than reasonable limit (e.g., 1024 chars)

```go
func isValidEntryName(name string) bool {
    // Reject empty
    if name == "" {
        return false
    }

    // Reject absolute paths
    if filepath.IsAbs(name) {
        return false
    }

    // Reject parent directory references
    if strings.Contains(name, "..") {
        return false
    }

    // Clean and compare (detect normalized parent refs)
    cleaned := filepath.Clean(name)
    if strings.HasPrefix(cleaned, "..") {
        return false
    }

    // Reject overly long names
    if len(name) > 1024 {
        return false
    }

    return true
}
```

### Symlink Handling

**Recommended approach:**

1. **Skip symlinks during extraction** - don't extract them at all
2. **Log warning** when symlinks encountered
3. **Document** that symlinks in archives are not supported
4. **Rationale:** Symlinks in archives are rarely needed and are a significant security risk

**Alternative (if symlinks needed):**

1. Extract symlinks but validate target
2. Ensure target is within installation directory (relative path)
3. Use `os.Root` to prevent escape even if validation fails
4. Never follow symlinks during file operations

## Archive Extraction Detailed Flow

### Step-by-Step Process

**Phase 1: Download**
1. HTTP GET request to manifest's platform.uri
2. Stream response body to temporary file
3. Calculate SHA256 hash while downloading
4. Close file and compare hashes
5. On success: Keep temp file, on failure: Delete temp file

**Phase 2: Validation**
1. Verify temp file exists and is readable
2. Verify file size is reasonable (not empty, not too large)
3. Detect format based on URI extension

**Phase 3: Extraction Setup**
1. Create temporary extraction directory: `${TMPDIR}/krew-extract-{random}`
2. Set directory mode: `0700` (owner only)
3. Create `os.Root` for extraction directory (in secure implementation)

**Phase 4: Archive Extraction**
1. Open downloaded archive file
2. For TAR.GZ: Create gzip reader, then tar reader
3. For ZIP: Open zip reader
4. For each entry in archive:
   a. Read entry metadata (name, size, mode, type)
   b. Validate entry name (reject `..`, absolute paths, symlinks)
   c. For directories: Create directory within root
   d. For files:
      - Create parent directories within root
      - Create file within root
      - Set permissions from archive
      - Copy content from archive reader
      - Close file
5. Close archive readers

**Phase 5: File Operations**
1. Create `os.Root` for final installation directory
2. For each file operation in manifest:
   a. Glob pattern match in extraction directory
   b. For each matched file:
      - Validate destination path
      - Create parent directories in install dir
      - Copy file from extract dir to install dir (within roots)
      - Preserve file permissions
      - Verify copy succeeded
3. Validate binary file exists at expected location

**Phase 6: Cleanup**
1. Delete temporary extraction directory (recursive)
2. Delete downloaded archive file
3. On error: Also delete partial installation directory

### Error Handling at Each Phase

**Download Errors:**
- Network failures: Retry not implemented, fail immediately
- SHA256 mismatch: Delete temp file, return security error
- Disk full: Fail during write, delete partial file

**Extraction Errors:**
- Invalid archive: Fail with format error, cleanup temp files
- Invalid entry name: Skip entry with warning, or fail (configurable)
- Permission denied: Fail, cleanup partial extraction
- Disk full: Fail, cleanup partial extraction

**File Operation Errors:**
- Glob no match: Warning or error (based on strictness)
- Copy failure: Fail immediately, no rollback
- Permission denied: Fail with clear error

**Cleanup Errors:**
- Cleanup failure logged but doesn't fail overall operation
- User may need to manually clean temporary directories

## Testing Requirements

### Security Tests

1. **Path Traversal Prevention:**
   - Archive with `../../etc/malicious` entry
   - Should: Reject entry or extract to safe location
   - Must not: Write outside installation directory

2. **Absolute Path Prevention:**
   - Archive with `/etc/malicious` entry
   - Should: Reject entry or extract to safe location
   - Must not: Write to absolute path

3. **Symlink Escape Prevention:**
   - Archive with symlink pointing outside directory
   - Should: Skip symlink or validate target
   - Must not: Allow escape via symlink

4. **Large File Handling:**
   - Archive with very large file (e.g., 10GB)
   - Should: Handle gracefully or reject with clear error
   - Must not: Exhaust memory or disk

5. **Malformed Archive:**
   - Corrupted tar.gz or zip file
   - Should: Detect and fail with clear error
   - Must not: Panic or leave partial extraction

### Functional Tests

1. **TAR.GZ Extraction:**
   - Extract archive with files and directories
   - Verify all files extracted correctly
   - Verify permissions preserved

2. **ZIP Extraction:**
   - Extract ZIP archive
   - Verify files extracted
   - Handle permission differences on Windows

3. **Nested Directories:**
   - Archive with deep directory structure
   - Verify all directories created

4. **File Operations:**
   - Glob pattern matching works correctly
   - Files copied to correct destinations
   - Permissions preserved

5. **Binary Verification:**
   - After extraction and file ops, binary is executable
   - Symlink points to correct binary

### Integration Tests

1. **Full Install Flow:**
   - Download → Extract → File Ops → Symlink → Receipt
   - Verify plugin is usable

2. **Cleanup on Failure:**
   - Inject failure at various points
   - Verify no leftover temporary files

3. **Concurrent Operations:**
   - Multiple installs simultaneously
   - Verify no corruption

## Implementation Checklist

For secure reimplementation:

- [ ] Use Go 1.24's `os.Root` for all file operations
- [ ] Validate all archive entry names before extraction
- [ ] Reject or skip symlinks in archives
- [ ] Prevent path traversal with `..` in entry names
- [ ] Prevent absolute paths in entry names
- [ ] Use streaming for downloads (don't load into memory)
- [ ] Verify SHA256 hash matches exactly
- [ ] Create temporary directories with secure permissions (0700)
- [ ] Clean up temporary files on success and failure
- [ ] Set appropriate file permissions from archive
- [ ] Test with malicious archives (path traversal, symlinks)
- [ ] Handle disk full gracefully
- [ ] Handle corrupted archives gracefully
- [ ] Log operations at appropriate verbosity levels
- [ ] Document symlink handling (not supported)

## Current Implementation References

**Download & Verification:**
- `internal/download/fetch.go`: HTTP download
- `internal/download/verifier.go`: SHA256 verification
- `internal/download/downloader.go`: Extraction logic

**File Operations:**
- `internal/installation/move.go`: File copying and moving
- `internal/installation/util.go`: Utility functions

**Installation Flow:**
- `internal/installation/install.go`: Main installation logic

**Platform Detection:**
- `internal/installation/platform.go`: OS/ARCH detection
