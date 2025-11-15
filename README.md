# Rust CLI Application Template with AI Specs

This is an opinionated template for creating a Rust CLI application with AI
specifications built-in.

## Prerequisites

Before you begin, ensure you have the following installed:

- [Rust](https://www.rust-lang.org/tools/install)
- [cargo-generate](https://github.com/cargo-generate/cargo-generate)

## Installation

To install `cargo-generate`, run the following command:

```bash
cargo install cargo-generate
```

## Usage

To create a new project from this template, run the following command:

```bash
cargo generate --git https://github.com/umag/rust-cli-ai-template.git --name my-cli-app --branch main template
```

This will create a new directory named `my-cli-app` with the template's
contents.

## Specifications and Testing

The specifications and testing methodologies used in this template are based on
the following resources:

- [The Rust CLI Book](https://rust-cli.github.io/book/index.html)
- [Rust CLI Recommendations](https://rust-cli-recommendations.sunshowers.io/index.html)
- [Monads through PBT](https://sunshowers.io/posts/monads-through-pbt/)

Licensed under
[CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/).
