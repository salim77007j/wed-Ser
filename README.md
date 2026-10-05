# wed-Ser

**An ultra-lightweight, ultra-fast, privacy-first web browser built on the [Servo](https://servo.org) engine.**

wed-Ser pairs the Servo rendering engine — a from-scratch, Rust-native browser engine developed under the
Linux Foundation — with a purpose-built shell and a hard privacy layer. The project pursues three goals,
in priority order:

1. **Lightweight** — a minimal memory footprint per tab, aggressive tab hibernation, and near-zero idle CPU.
   Target: **under 100 MB of RAM per tab**, and the lowest idle resource usage of any 2026 browser.
2. **Fast** — GPU-accelerated rendering via WebRender, parallel layout via Stylo, and modern transport
   (HTTP/3, QUIC, DNS-over-HTTPS) once those land in Phase 2.
3. **Private** — a network-level ad/tracker filter, per-site fingerprint resistance, third-party cookie
   partitioning, and URL tracking-parameter stripping. All blocking happens locally; no telemetry.

## Why Servo?

- **Rust end-to-end**: memory safety without a garbage collector for the engine itself.
- **Proven components**: Stylo (Servo's CSS engine) powers Firefox; WebRender powers Firefox's GPU pipeline.
- **Embeddable**: Servo exposes `libservo`, a clean embedding API that wed-Ser builds its shell on.
- **Small and hackable**: ~10× smaller than Chromium, with a fraction of the build complexity.

## Engine integration

The engine lives in [`servo/`](servo) as a git submodule pinned to the **latest stable Servo release**:

| Component | Version |
|---|---|
| Servo | `v0.6.0` (2026-09-29) — see [releases](https://github.com/servo/servo/releases) |
| License | MPL-2.0 (engine); wed-Ser itself is MPL-2.0 |

wed-Ser does not fork the engine in Phase 1. Engine enhancements (Phase 2) will be carried as patches on
a forked Servo repository, re-pinned here, so upstream integration stays clean.

## Build

See [docs/BUILD_GUIDE.md](docs/BUILD_GUIDE.md) for per-platform dependency lists.

```bash
git clone --recurse-submodules https://github.com/salim77007j/wed-Ser
cd wed-Ser/servo
./mach build --release        # Linux / macOS
./mach build                  # first build; add --release for optimized
./mach run https://example.com
```

CI builds Servo from source on every push that touches `servo/` or the workflow — see
[`.github/workflows/servo-baseline.yml`](.github/workflows/servo-baseline.yml).

## Roadmap

| Phase | Goal | Status |
|---|---|---|
| 1 | Environment, Servo integration, architecture study, CI | ✅ Complete |
| 2 | Engine enhancement: HTTP/3, DoH, modern CSS/JS, hardening | ⬜ Awaiting go-ahead |
| 3 | Ultra-lightweight UI + extreme resource optimization | ⬜ |
| 4 | Stealth ad blocker & privacy engine | ⬜ |
| 5 | Build, packaging, Windows + Linux distribution | ⬜ |
| 6 | Testing vs Chrome on 10 real sites | ⬜ |

Phase reports live in the repo root (`PHASE_N_REPORT.md`); architecture notes in [`docs/`](docs).

## Contributing

Read [CONTRIBUTING.md](CONTRIBUTING.md). All code is Rust; formatting and linting are enforced
(`rustfmt`, `clippy`, and Servo's `crown` GC-rooting linter for engine code).

## License

MPL-2.0. Servo is © the Servo project developers and Mozilla, also MPL-2.0.
