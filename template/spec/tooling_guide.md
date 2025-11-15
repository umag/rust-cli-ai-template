# Tooling Guide

This document provides a brief overview of the tools used in this project.

## Issue Tracking with `bd` (beads)

This project uses `bd` (beads) for all issue tracking.

### Quick Start

**Check for ready work:**

```bash
bd ready --json
```

**Create new issues:**

```bash
bd create "Issue title" -t bug|feature|task -p 0-4 --json
bd create "Issue title" -p 1 --deps discovered-from:bd-123 --json
```

**Claim and update:**

```bash
bd update bd-42 --status in_progress --json
bd update bd-42 --priority 1 --json
```

**Complete work:**

```bash
bd close bd-42 --reason "Completed" --json
```

## Testing with `cargo nextest`

This project uses `cargo nextest` as the test runner for improved performance
and better output.

To run all tests, use:

```bash
cargo nextest run
```

## Version Control with `jj`

This project uses `jj` for version control.

### Making Changes

1. Make your changes to the code.
2. Snapshot the changes: `jj log --no-pager`
3. Create a commit: `jj commit -m "description of the changes made"`
4. Update the `main` bookmark: `jj bookmark set --revision @- main`
5. Push the changes: `jj git push -r @-`
