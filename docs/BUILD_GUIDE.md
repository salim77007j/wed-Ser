# Building wed-Ser / Servo baseline

wed-Ser tracks the Servo engine as a git submodule pinned to the latest stable release
(`v0.6.0` at the time of Phase 1). Building the browser today means building `servoshell`
inside the submodule; the wed-Ser shell replaces it in Phase 3.

The authoritative upstream instructions live in the
[Servo book](https://book.servo.org/hacking/choosing-a-recipe.html). This guide summarizes
the steps we verified in CI (`.github/workflows/servo-baseline.yml`, which mirrors Servo's
own GitHub Actions workflows at `v0.6.0`).

## All platforms (common)

```bash
git clone --recurse-submodules https://github.com/salim77007j/wed-Ser
cd wed-Ser/servo
```

Servo pins its own Rust toolchain via `rust-toolchain.toml` (stable **1.97.1** for `v0.6.0`
with `clippy`, `rustfmt`, `rust-src`, `rustc-dev`, `llvm-tools` components). `rustup`
installs it automatically on the first `mach` invocation — a working `rustup` is required.

Python is required for `mach` (which manages its own virtualenv via `uv`).

## Linux (Ubuntu 24.04 — what CI uses)

```bash
sudo apt-get update
./mach bootstrap --yes --skip-lints --skip-nextest   # installs all required apt packages
```

`mach bootstrap` handles the full native dependency set (build tools, X11/XCB client
libraries, GL/EGL, fontconfig, ALSA, GStreamer, etc.). Two non-obvious requirements that
Servo's own CI calls out on GitHub-hosted runners:

- **LLVM/Clang ≥ 20.1** with `LIBCLANG_PATH` pointing at its `lib/` directory — the
  SpiderMonkey bindgen step needs a modern libclang; the distro clang on 24.04 is too old.
- `tshark` and `shellcheck` are installed in CI before bootstrap.

Build, smoke-test, package:

```bash
./mach build --locked --profile debug     # fastest; use --profile release for real use
xvfb-run ./mach smoketest --profile debug # official headless smoke test
./mach package --profile debug            # produces target/<profile>/servo-tech-demo.tar.gz
```

## Windows (x86_64, MSVC)

Prerequisites: Visual Studio (or Build Tools) with the **Desktop development with C++**
workload, `rustup`, Git, Python 3.x, and CMake.

```powershell
.\mach fetch                 # fetch vendored dependencies
.\mach bootstrap-gstreamer   # download the GStreamer MSVC runtime Servo builds against
.\mach build --locked --profile debug
.\mach smoketest --profile debug
```

As on Linux, install **LLVM ≥ 20.1** and set `LIBCLANG_PATH` to its `bin\` directory if
bindgen complains about clang versions. Packaging into an MSI additionally needs WiX 7 —
that is Phase 5 territory.

### Windows notes

- Enable git long paths if checkout fails on deep Web Platform Test directories:
  `git config core.longpaths true` (the submodule in this repo already sets it).
- Keep `core.autocrlf false` for the engine checkout; mixed line endings slow builds and
  can break SpiderMonkey's build scripts.

## Resource expectations

- Disk: a debug build needs roughly 20 GB (`target/` dominates). Release profiles less.
- RAM: linking `servoshell` in debug wants 6-8 GB headroom. Machines with ≤ 8 GB total RAM
  should limit parallelism, e.g. `CARGO_BUILD_JOBS=4 ./mach build`, or build in CI instead.
- Time: 30-60 min on a modern 8+ core machine; 1.5-3 h on 4-core CI runners.

## CI

`.github/workflows/servo-baseline.yml` builds the pinned engine from source on every push
to `main`, runs the official smoketest on a virtual display, packages the result, and
uploads it as an artifact. The Windows job runs on demand (`workflow_dispatch`).
