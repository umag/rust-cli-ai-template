# Atomic Writes Specification

This specification details the implementation of atomic file writes to ensure
data integrity across different output modes.

## Overview

"Atomic writes" imply different strategies depending on the target and mode:

1. **File Replacement**: Replacing an existing file entirely.
2. **Appends**: Adding to an existing file.
3. **Streams**: Writing to stdout, stderr, or pipes.

## Strategies

### 1. File Replacement (Write-Replace)

**Use case**: Saving configuration, state files, or documents where the entire
content is rewritten.

**Mechanism**:

1. Write data to a temporary file in the same directory.
2. Sync changes to disk.
3. Atomically rename the temporary file to the destination.

**Dependencies**: `tempfile` crate.

```rust
pub fn atomic_write_file<P: AsRef<Path>, C: AsRef<[u8]>>(path: P, content: C) -> io::Result<()> {
    let path = path.as_ref();
    let dir = path.parent().unwrap_or_else(|| Path::new("."));
    let mut temp_file = NamedTempFile::new_in(dir)?;
    temp_file.write_all(content.as_ref())?;
    temp_file.persist(path).map_err(|e| e.error)?;
    Ok(())
}
```

### 2. File Append

**Use case**: Logging, history files.

**Mechanism**: The "Write-Replace" strategy **cannot** be used for appending.
Instead, we rely on the operating system's `O_APPEND` flag. On POSIX systems,
writes to a file opened with `O_APPEND` are atomic (for writes smaller than
`PIPE_BUF`).

**Implementation**: Use `std::fs::OpenOptions`.

```rust
use std::fs::OpenOptions;

pub fn atomic_append<P: AsRef<Path>, C: AsRef<[u8]>>(path: P, content: C) -> io::Result<()> {
    let mut file = OpenOptions::new()
        .create(true)
        .append(true)
        .open(path)?;
    file.write_all(content.as_ref())?;
    // Note: fsync/sync_all is still needed for durability, but atomicity is provided by O_APPEND
    file.sync_data()?; 
    Ok(())
}
```

### 3. Streams (Stdout/Pipes)

**Use case**: CLI output piped to other tools (`|`), redirection (`>`).

**Mechanism**: We cannot "rename" over a pipe. Atomicity here means preventing
interleaved output from multiple threads or processes.

1. **Single Thread**: `write_all` is generally safe.
2. **Multi-threaded**: Use `stdout().lock()` or buffer the entire output in
   memory and write in one call.

**Implementation**: Check if the output is a file or a TTY/Pipe. If it's a pipe,
we cannot use the tempfile dance.

```rust
use std::io::{self, Write, IsTerminal};

pub fn write_output(content: &[u8], path: Option<&Path>, append: bool) -> io::Result<()> {
    match path {
        Some(p) => {
            if append {
                atomic_append(p, content)
            } else {
                atomic_write_file(p, content)
            }
        }
        None => {
            // Write to stdout
            let stdout = io::stdout();
            let mut handle = stdout.lock(); // Prevent interleaving
            handle.write_all(content)?;
            Ok(())
        }
    }
}
```

## Security Considerations: TOCTTOU

Time-of-Check Time-of-Use (TOCTTOU) race conditions are a concern when file
operations depend on paths that might change between operations (e.g., between a
`stat` check and a `rename`).

While standard `rename` and `unlink` operations require paths, we can mitigate
race conditions (such as directory swapping or symlink attacks) through the
following strategies:

1. **Tight Directory Permissions**: Ensure the directory containing the files
   has strict access controls. If an attacker cannot write to the directory,
   they cannot exploit race conditions by manipulating entries.

2. **Directory File Descriptors (`*at` syscalls)**: Instead of using full paths
   for every operation, open a file descriptor for the directory once and use
   "at" system calls (`openat`, `renameat`, `unlinkat`) to operate relative to
   that descriptor.

   ```rust
   // Conceptual example using openat/renameat logic
   let dir_fd = open(parent_dir, O_RDONLY | O_DIRECTORY)?;
   // Operations are now relative to dir_fd, pinning the directory
   renameat(dir_fd, "temp_file", dir_fd, "target_file")?;
   ```

   This ensures that even if the directory path is modified external to the
   process, the operations continue to act on the original directory handle.

## Requirements

1. **Detection**: The system must distinguish between target types (File vs
   Stream) and Modes (Overwrite vs Append).
2. **Consistency**:
   - **Overwrite File**: Must be atomic via rename.
   - **Append File**: Must use `O_APPEND`.
   - **Stream**: Must use locking or single-write buffering.
3. **Durability**: Files must be synced. Streams typically flush on newline or
   buffer fill; manual flush may be required.
4. **Failure Safety**: Temp files used for replacement must be cleaned up on
   failure.
