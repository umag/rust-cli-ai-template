# Logging Specification

This document specifies the logging strategy for the application, utilizing the
`log` crate facade and `env_logger` implementation.

## Overview

The application uses structured logging to provide runtime information about its
execution. This aids in debugging, monitoring, and understanding the
application's behavior without needing to attach a debugger.

## Dependencies

The following crates are used for logging:

- [`log`](https://crates.io/crates/log): A lightweight logging facade.
- [`env_logger`](https://crates.io/crates/env_logger): A logging implementation
  that can be configured via environment variables.

## Implementation Details

### Initialization

Logging must be initialized early in the application's startup process. The
initialization must configure the logger to provide extensive context for
`Debug` and `Trace` levels.

```rust
use std::io::Write;
use log::Level;

fn init_logging() {
    env_logger::Builder::from_default_env()
        .format(|buf, record| {
            if record.level() <= Level::Debug {
                // For Debug and Trace, include rich context: timestamp, level, thread, target, file:line
                writeln!(
                    buf,
                    "[{timestamp} {level} {thread} {target} {file}:{line}] {args}",
                    timestamp = buf.timestamp(),
                    level = record.level(),
                    thread = std::thread::current().name().unwrap_or("unnamed"),
                    target = record.target(),
                    file = record.file().unwrap_or("unknown"),
                    line = record.line().unwrap_or(0),
                    args = record.args()
                )
            } else {
                // For Info, Warn, Error, use a cleaner format
                writeln!(
                    buf,
                    "[{level}] {args}",
                    level = record.level(),
                    args = record.args()
                )
            }
        })
        .init();
}
```

### Usage

Use the macros provided by the `log` crate to emit log records.

```rust
use log::{error, warn, info, debug, trace};

fn some_function() {
    error!("Something went wrong: {}", error_details);
    warn!("This is a warning");
    info!("Operation completed successfully");
    debug!("Details for debugging: {:?}", internal_state);
    trace!("Low-level trace information");
}
```

### Log Levels

The application supports the following log levels, in decreasing order of
severity:

1. **Error**: Critical errors that cause a task or the application to fail.
2. **Warn**: Potential issues that are not immediately fatal but should be
   investigated.
3. **Info**: General operational events (e.g., startup, shutdown, major
   milestones).
4. **Debug**: Detailed information useful for debugging. **Must include**: File
   path, line number, module path/target, and thread information.
5. **Trace**: Very low-level information for tracing code execution. **Must
   include**: Full context similar to Debug.

### Configuration

Logging is configured via the `RUST_LOG` environment variable.

- Enable all logs at `info` level and above:
  ```bash
  RUST_LOG=info ./my_app
  ```
- Enable logs for a specific module:
  ```bash
  RUST_LOG=my_app::module=debug ./my_app
  ```
- Complex configuration:
  ```bash
  RUST_LOG=info,my_app::db=trace ./my_app
  ```

## User Interface

The application should be usable without seeing internal logs by default.
Important information for the user should be printed to `stdout` or `stderr`
directly (e.g., using `println!`), or `info` level logs should be formatted in a
user-friendly way.

## Testing

Tests should be able to capture logs to verify behavior. The logging
initialization should ensure it is only called once to prevent panic during
parallel test execution (e.g., using `std::sync::Once` or `try_init`).
