# Signals and Exit Handling Specification

This specification outlines the approach for handling Unix signals and exit
codes in the application, ensuring robust and predictable behavior during
termination or interruption.

## Goal

To implement a signal handling mechanism that:

1. Gracefully handles interruptions (e.g., Ctrl-C).
2. Ensures cleanup of resources (e.g., stopping worker tasks, flushing data).
3. Propagates signals to child processes where appropriate.
4. Provides correct exit codes based on the termination reason.

## Core Concepts

### 1. Signal Handling

The application will listen for specific Unix signals to determine when to shut
down or perform other actions.

- **SIGINT (Ctrl-C)**:
  - **First occurrence**: Initiate a graceful shutdown. This involves notifying
    all running tasks to cancel their work and waiting for them to finish
    cleanup.
  - **Second occurrence**: Force an immediate exit. This provides a "safety
    valve" for users if the graceful shutdown gets stuck.
- **SIGTERM**:
  - Treat similarly to SIGINT (first occurrence), initiating a graceful
    shutdown. This is standard for service managers like systemd or Kubernetes.
- **SIGQUIT**:
  - behave similarly to SIGINT/SIGTERM or perform a core dump if configured
    (optional, but good for debugging). For now, treat as graceful shutdown.

### 2. Architecture using Async Rust (Tokio)

We will utilize `tokio::signal` and `tokio::select!` to manage concurrency
between application logic and signal handling.

#### Components:

1. **Main Loop**:
   - Runs the primary application logic.
   - Uses `tokio::select!` to listen for:
     - Application completion.
     - Signal events.

2. **Broadcast Channel (`tokio::sync::broadcast`)**:
   - Used to notify all worker tasks of a shutdown request.
   - When a signal is received, a message (e.g., `Shutdown`) is sent over this
     channel.

3. **JoinSet (`tokio::task::JoinSet`)**:
   - Manages worker tasks.
   - Allows waiting for all tasks to complete during the graceful shutdown
     phase.

#### Control Flow:

1. **Startup**:
   - Initialize `JoinSet` for workers.
   - Create a broadcast channel `(shutdown_tx, shutdown_rx)`.
   - Spawn signal listeners for SIGINT, SIGTERM, etc.

2. **Running**:
   - Spawn worker tasks, passing each a `shutdown_rx`.
   - In the main loop, `select!` on:
     - `join_set.join_next()`: Handle worker completion.
     - `signal_stream.recv()`: Handle incoming signals.

3. **Graceful Shutdown**:
   - On first signal (SIGINT/SIGTERM):
     - Log "Initiating graceful shutdown...".
     - Drop the `shutdown_tx` (or send a message) to signal workers.
     - Wait for `join_set` to drain (all workers to exit).
     - If `join_set` is empty, exit with code 0 (or appropriate code).

4. **Forced Shutdown (Double Ctrl-C)**:
   - If a second SIGINT is received while waiting for workers to drain:
     - Log "Forced shutdown received. Exiting immediately.".
     - Exit process with a non-zero code (e.g., 130 for SIGINT).

### 3. Child Process Handling

If the application spawns child processes (e.g., via `std::process::Command` or
`tokio::process::Command`):

- **Process Groups**:
  - Use process groups (`setpgid`) to isolate children if they shouldn't receive
    the same signals directly from the shell, OR rely on the shell's default
    behavior where Ctrl-C sends SIGINT to the entire process group.
  - If managing process groups manually, the application must forward signals to
    the child process group.

- **Cleanup**:
  - On shutdown, ensure child processes are terminated (e.g., send SIGTERM to
    children, wait, then SIGKILL if necessary).

### 4. Exit Codes

- **0**: Successful completion.
- **1**: Generic error.
- **130**: Terminated by SIGINT (standard convention: 128 + signal number).
- **143**: Terminated by SIGTERM (128 + 15).

## Implementation Details

### Dependencies

- `tokio` with `signal`, `sync`, `macros`, `rt-multi-thread` features.
- `anyhow` for error handling.
- `tracing` for logging.

### Example Code Structure

```rust
use tokio::signal::unix::{signal, SignalKind};
use tokio::sync::broadcast;
use tokio::task::JoinSet;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // Setup signals
    let mut sigint = signal(SignalKind::interrupt())?;
    let mut sigterm = signal(SignalKind::terminate())?;

    // Setup broadcast for cancellation
    let (tx, _rx) = broadcast::channel(1);

    // Setup JoinSet for workers
    let mut join_set = JoinSet::new();

    // Spawn example workers
    for i in 0..5 {
        let mut rx = tx.subscribe();
        join_set.spawn(async move {
            tokio::select! {
                _ = tokio::time::sleep(std::time::Duration::from_secs(10)) => {
                    println!("Worker {} finished naturally", i);
                }
                _ = rx.recv() => {
                    println!("Worker {} cancelling...", i);
                    // Perform cleanup...
                }
            }
        });
    }

    // Main loop
    loop {
        tokio::select! {
            // Handle worker completion
            Some(res) = join_set.join_next() => {
                match res {
                    Ok(_) => {}, // Task finished successfully
                    Err(e) => eprintln!("Task execution error: {}", e),
                }
                if join_set.is_empty() {
                    println!("All workers finished.");
                    break;
                }
            }
            
            // Handle SIGINT
            _ = sigint.recv() => {
                println!("Received SIGINT. Initiating graceful shutdown...");
                // Signal workers to stop
                let _ = tx.send(());
                // Don't break immediately; wait for join_set to drain.
                // Depending on logic, we might switch to a "draining" state
                // where we only wait for join_set and listen for a 2nd SIGINT.
                handle_shutdown(&mut join_set, &mut sigint).await;
                return Ok(());
            }

            // Handle SIGTERM
            _ = sigterm.recv() => {
                println!("Received SIGTERM. Initiating graceful shutdown...");
                let _ = tx.send(());
                handle_shutdown(&mut join_set, &mut sigint).await;
                return Ok(());
            }
        }
    }

    Ok(())
}

async fn handle_shutdown(join_set: &mut JoinSet<()>, sigint: &mut tokio::signal::unix::Signal) {
     loop {
        tokio::select! {
            Some(_) = join_set.join_next() => {
                if join_set.is_empty() {
                    println!("Graceful shutdown complete.");
                    break;
                }
            }
            _ = sigint.recv() => {
                 println!("Forced shutdown!");
                 std::process::exit(130);
            }
        }
    }
}
```

## References

- [Beyond Ctrl-C: The dark corners of Unix signal handling](https://sunshowers.io/posts/beyond-ctrl-c-signals/)
- Tokio Documentation: `tokio::signal`
