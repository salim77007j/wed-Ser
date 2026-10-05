# Contributing to wed-Ser

Thanks for your interest. wed-Ser is built in strict phases; contributions should target the
current phase's goals (see the roadmap in [README.md](README.md) and the latest `PHASE_N_REPORT.md`).

## Ground rules

1. **No fake implementations.** Every feature must be real, functional, and tested. No stubs, no
   mockups, no dead code. If a feature is partially complete, it belongs behind an explicit
   `unimplemented` error or a feature flag — never silently.
2. **Document every decision.** Non-obvious choices (library picks, algorithms, tradeoffs) get a
   short note in the relevant `docs/` file or in the phase report.
3. **Test continuously.** Tests accompany every feature. Do not push red builds.
4. **Respect the phase protocol.** One phase at a time; each phase ends with a report and a push.

## Code standards

### Rust

- **Toolchain**: use the pinned toolchain from `servo/rust-toolchain.toml` for engine work;
  `rustup` will select it automatically inside `servo/`.
- **Formatting**: `cargo fmt` must produce zero diffs. Run `cargo fmt --all` before committing.
- **Linting**: `cargo clippy` warnings are errors on new code. Do not add `#[allow(...)]` without
  a comment explaining why.
- **Engine code** (anything under `servo/`): run Servo's `crown` linter (`./mach test-crown` in the
  servo directory). GC-rooting discipline is not optional — unrooted `JS::Handle`/`JSRef` misuse
  will be rejected.
- **Style**: follow existing idioms in whichever codebase you are touching. In wed-Ser shell code,
  prefer small pure functions, `#[derive(Debug)]` on all types, and `anyhow`/`thiserror` for error
  plumbing (library crates use `thiserror`; the shell binary may use `anyhow`).

### Commits

- Meaningful, imperative-subject messages: `net: add DoH resolver with bootstrap cache`, not
  `update stuff`.
- One logical change per commit; group related files.
- Reference the phase and task where relevant: `phase2: add HTTP/3 transport via quinn`.

### Tests

- Unit tests live next to the code (`#[cfg(test)] mod tests`).
- Integration/engine tests run through `./mach` inside `servo/`.
- Performance changes must include before/after numbers in the commit message or phase report.

## Licensing

All new files are MPL-2.0. Engine files inherit Servo's existing headers; do not strip them.

## Submitting

Push to a branch and open a PR against `main`. CI must be green, including the Servo baseline build
(`.github/workflows/servo-baseline.yml`).
