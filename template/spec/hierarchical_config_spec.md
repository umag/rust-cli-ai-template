# Hierarchical Configuration Specification

Based on
[Rust Hierarchical Configuration](https://steezeburger.com/2023/03/rust-hierarchical-configuration/),
this specification details the implementation of a hierarchical configuration
system for the application.

## Overview

The application will support configuration from three sources, ordered by
precedence (lowest to highest):

1. Configuration file (`Config.toml`)
2. Environment variables (prefixed with `APP_`)
3. Command line arguments

This hierarchy allows for:

- **Easier testing**: Configure differently for different environments
- **Improved portability**: Override specific values via environment variables
  (12 Factor App)
- **Flexibility**: Provide defaults while allowing power user overrides

## Technical Implementation

### Dependencies

- **Figment**: For layered configuration handling (`figment`)
- **Clap**: For command line argument parsing (`clap`)
- **Serde**: For serialization/deserialization (`serde`)

### Architecture

The implementation requires separating the CLI argument structure from the
internal configuration structure to handle optional overrides correctly.

#### 1. CLI Structure (`cli.rs`)

Responsible for parsing command line arguments. Fields must be `Option<T>` and
skip serialization if `None`.

```rust
use clap::Parser;
use serde::Serialize;

#[derive(Debug, Parser, Serialize)]
pub(crate) struct Cli {
    /// The name
    #[arg(long = "name")]
    #[serde(skip_serializing_if = "::std::option::Option::is_none")]
    pub(crate) name: Option<String>,

    /// The count
    #[arg(long = "count")]
    #[serde(skip_serializing_if = "::std::option::Option::is_none")]
    pub(crate) count: Option<u8>,
}
```

#### 2. Configuration Structure (`config.rs`)

Defines the final, validated configuration used by the application.

```rust
use serde::{Deserialize, Serialize};

/// The global configuration for the application
#[derive(Serialize, Deserialize)]
pub(crate) struct Config {
    /// The name
    pub(crate) name: String,

    /// The count
    pub(crate) count: u8,
}
```

#### 3. Integration (`main.rs`)

Merges configuration sources in the correct order.

```rust
use clap::Parser;
use color_eyre::eyre::Result;
use figment::{
    Figment,
    providers::{Env, Format, Serialized, Toml},
};
use crate::cli::Cli;
use crate::config::Config;

pub fn load_config() -> Result<Config> {
    // hierarchical config. cli args override envars which override toml config values
    let conf: Config = Figment::new()
        .merge(Toml::file("Config.toml"))
        .merge(Env::prefixed("APP_"))
        .merge(Serialized::defaults(Cli::parse()))
        .extract()?;
        
    Ok(conf)
}
```

## Requirements

1. **Precedence**: CLI args > Env vars > Config file
2. **Partial Updates**: CLI args should only override specified values (using
   `Option` and `skip_serializing_if`)
3. **Type Safety**: Final configuration must be fully populated and valid
4. **Environment Variables**: Must support `APP_` prefix
