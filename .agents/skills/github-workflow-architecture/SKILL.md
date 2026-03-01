---
name: GitHub Actions Architecture
description: When you want to modify or debug the github actions/workflows in this repo, read this.
---

# GitHub Actions Workflow Architecture

## Overview

GitHub Actions workflow YAML files are **generated from Rust code**, not hand-edited. The source of truth is `tooling/xtask/src/tasks/workflows/`. Regenerate all workflows with:

```
cargo xtask workflows
```

Never edit `.github/workflows/*.yml` directly for generated workflows. Check the first line of a YAML file — generated ones start with `# Generated from xtask::workflows::`.

## Source structure

```
tooling/xtask/src/tasks/
  workflows.rs          # Entry point: registers all WorkflowFile entries, calls generate_file()
  workflows/
    runners.rs          # Runner labels (ubuntu-*, macos-*, windows-*) and Arch/Platform enums
    steps.rs            # Reusable step builders (checkout, sccache, sentry, etc.) and NamedJob struct
    vars.rs             # Secrets, vars, asset name constants, concurrency helpers
    run_bundling.rs     # PR bundling test workflow (bundle_linux, bundle_mac, bundle_windows)
    release.rs          # Production release workflow, reuses bundle functions from run_bundling.rs
    release_nightly.rs  # Nightly release workflow
    run_tests.rs        # CI test workflow
    nix_build.rs        # Nix build jobs
    ...                 # Other workflow definitions
```

## Key abstractions

- **`gh_workflow` crate**: External crate providing `Workflow`, `Job`, `Step<Run>`, `Step<Use>`, etc.
- **`NamedJob`** (`steps.rs`): Pairs a job name string with a `Job`, used for dependency tracking between jobs.
- **`Arch` enum** (`runners.rs`): `X86_64` or `AARCH64`. Has `Display` impl and `linux_bundler()` method.
- **`Platform` enum** (`runners.rs`): `Windows`, `Linux`, or `Mac`.
- **`FluentBuilder` trait** (`steps.rs`): Provides `.map()`, `.when()`, `.when_some()` for conditional job/step building.
- **`named::` module** (`steps.rs`): Auto-names steps/workflows using backtrace-based function name detection. Call `named::bash("cmd")` inside a function named `setup_linux` and the step gets named `steps::setup_linux`.

## How bundling works

`run_bundling.rs` defines `bundle_linux()`, `bundle_mac()`, `bundle_windows()` which are **reused** by `release.rs` and `release_nightly.rs`. Each function:

1. Selects a runner via `arch.linux_bundler()` or platform constants
2. Adds platform-specific setup steps
3. Calls the bundle shell script (`./script/bundle-linux`, `./script/bundle-mac {triple}`, `script/bundle-windows.ps1 -Architecture {arch}`)
4. Uploads artifacts with paths from `vars::assets` constants

The Mac and Windows bundle scripts accept an architecture argument. The Linux bundle script (`script/bundle-linux`) detects architecture from the host via `uname -m` and `rustc --version --verbose`, so it must run on a runner matching the target architecture.

## Runner configuration

Defined in `runners.rs`. Key runners:

- `LINUX_X86_BUNDLER`: `ubuntu-20.04` (older glibc for compatibility)
- `LINUX_ARM_BUNDLER`: `ubuntu-22.04-arm` (ARM runner for aarch64 builds)
- `MAC_DEFAULT`: `macos-latest`
- `WINDOWS_DEFAULT`: `windows-latest`

## Asset naming

`vars.rs` defines canonical artifact filenames in `vars::assets`:

- `zed-linux-{arch}.tar.gz`, `Zed-{arch}.dmg`, `Zed-{arch}.exe`
- `zed-remote-server-{platform}-{arch}.gz` (or `.zip` for Windows)

These names are also used by `zed.dev` — keep them in sync.

## Workflow generation flow

1. `cargo xtask workflows` calls `run_workflows()` in `workflows.rs`
2. Each `WorkflowFile` entry calls its `source` function (e.g., `run_bundling::run_bundling()`) to build a `Workflow`
3. The workflow is serialized to YAML, prepended with a disclaimer comment, and written to `.github/workflows/{name}.yml`
