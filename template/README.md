# {{ project-name }}

This is the README for {{ project-name }}.

## Usage

### Building the project

To build the project, run the following command:

```bash
cargo build
```

### Running tests

This project uses [cargo-nextest](https://nexte.st/) for running tests.

First, install it:

```bash
cargo install cargo-nextest
```

Then, run the tests:

```bash
cargo nextest run
```

## Version Control with Jujutsu (jj)

This project uses [Jujutsu (jj)](https://github.com/jj-vcs/jj) for version
control. Here are some common commands:

- **Snapshot changes**: `jj log --no-pager`
- **Commit changes**: `jj commit -m "Your commit message"`
- **Set a branch/bookmark**: `jj bookmark set --revision @- main`
- **Push to remote**: `jj git push -r @-`

## Issue Tracking with Beads (bd)

All issue tracking is handled by
[beads (bd)](https://github.com/steveyegge/beads).

- **Check for ready work**: `bd ready --json`
- **Create a new issue**:
  `bd create "Issue title" -t bug|feature|task -p 0-4 --json`
- **Start working on an issue**:
  `bd update <issue-id> --status in_progress --json`
- **Close an issue**: `bd close <issue-id> --reason "Completed" --json`

## Specifications and Testing

The specifications and testing methodologies used in this template are based on
the following resources:

- [The Rust CLI Book](https://rust-cli.github.io/book/index.html)
- [Rust CLI Recommendations](https://rust-cli-recommendations.sunshowers.io/index.html)
- [Monads through PBT](https://sunshowers.io/posts/monads-through-pbt/)

Licensed under
[CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/).
