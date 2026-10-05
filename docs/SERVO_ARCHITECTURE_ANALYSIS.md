# Servo Architecture Analysis (for wed-Ser)

**Pinned version studied:** Servo `v0.6.0` (LTS), released 2026-09-29 — the latest stable release.
**Method:** direct inspection of the pinned source tree (`servo/` submodule), the v0.6.0 release
manifests, Servo's own CI workflows at the same tag, and the Servo book. Facts below were verified
against the pinned sources; anything we could not verify is explicitly marked **unverified**.

---

## 1. Executive summary

Servo in late 2026 is a structurally different project from the "classic" Servo of 2015-2023.
Three changes matter most for wed-Ser:

1. **The engine has been componentized as published crates.** Stylo (CSS) and WebRender (GPU
   painting) are no longer in-tree forks: `v0.6.0` consumes `stylo 0.21.0` and `webrender 0.70`
   from crates.io, and SpiderMonkey bindings come from the published `mozjs` crate. The old
   `components/style`, `components/gfx`, and `components/compositing` crates no longer exist.
   Compositor responsibilities were folded into `libservo` (`components/servo`), and font handling
   lives in a dedicated `fonts` crate.
2. **Servo is embedder-first.** The supported product surface is `libservo` + the C API (`ffi/capi`),
   with `servoshell` (egui-based) as the reference browser. A formal **LTS program** exists — our
   pin `v0.6.0` is its first LTS line (`release/v0.6` branch, immutable release artifacts).
3. **The stack is fully modernized on the transport side except HTTP/3**: hyper 1.10 (HTTP/1+2),
   rustls 0.23 with the aws-lc-rs provider (which brings post-quantum key exchange), no OpenSSL
   anywhere. **QUIC/HTTP3 is absent from the entire workspace** — this is the single largest
   transport gap wed-Ser Phase 2 will close.

## 2. Toolchain and build system

| Item | Value at v0.6.0 | Source |
|---|---|---|
| Rust channel | `1.97.1` (pinned exact, **stable** — no longer nightly) | `rust-toolchain.toml` |
| Components | `clippy`, `rustfmt`, `rust-src`, `rustc-dev`, `llvm-tools` | `rust-toolchain.toml` |
| MSRV | 1.88.0 | workspace `Cargo.toml` |
| Edition | 2024 | workspace `Cargo.toml` |
| Build driver | `mach` (Python + `uv` virtualenv) | `python/` |
| Build profiles | `debug`, `release`, `checked-release`, `production` (+`-stripped`) | `mach build --profile` |

Notable: Servo's CI builds with `--locked` against a committed `Cargo.lock`, which makes our
pinned-engine builds reproducible. The default build target is `ports/servoshell`.
`crown`, Servo's custom rustc linter for GC rooting, is a separate crate under `support/` and is
compiled against the exact pinned rustc (`rustc-dev` component exists for this reason).

## 3. Crate map (what each part does)

Top-level layout at the pin: `components/`, `ports/servoshell`, `ffi/capi`, `support/` (incl.
`crown`), `tests/`, `python/` (mach), `resources/`.

Core engine crates under `components/`:

| Crate | Role |
|---|---|
| `servo` (libservo) | The embeddable engine: WebView API, event loop, **and the compositor** (post-refactor home of the former `compositing` crate) |
| `constellation` | The "traffic cop": owns the pipeline (frame) tree, session history, focus, and routes all cross-component messages |
| `script` | DOM implementations, event loops, SpiderMonkey glue, the JS-visible Web platform |
| `script_bindings` | Generated DOM↔JS bindings layer (split out of `script` in the 2025-2026 refactor) |
| `layout` (`servo-layout`) | Layout engine: block/inline layout plus flexbox/grid via **taffy 0.14**; consumes Stylo output, emits display lists |
| `net` (`servo-net`) | Fetch/HTTP(S), WebSockets, cookies, HSTS, HTTP cache, file/data/blob schemes |
| `fonts` | Font enumeration, fallback, and platform backends (successor to the font half of the old `gfx`) |
| `paint` | Painting abstraction over WebRender, with an optional **vello 0.10** backend |
| `pixels` | Raster/image primitives |
| `media` | Audio/video via **GStreamer 0.25** (play, audio, streams, webrtc sub-crates) |
| `canvas` / `webgl` / `webgpu` | 2D canvas; WebGL via ANGLE (`mozangle`) on `surfman`; WebGPU via **wgpu-core 30** |
| `webxr` | WebXR (openxr on desktop) |
| `storage` | localStorage/IndexedDB-family persistence on **rusqlite 0.38** + `sea-query 1.0` |
| `bluetooth` / `webvtt` / `xpath` / `svg` (resvg 0.48) / `timers` / `wakelock` / `metrics` / `config` / `url` | Feature crates added or promoted during the componentization push |
| `devtools` / `webdriver_server` | Remote debugging protocol; WebDriver endpoint (crates.io `webdriver 0.54`) |
| `profile` / `background_hang_monitor` / `servo_tracing` | Time/memory profilers, hang detection, tracing (perfetto) |
| `shared/*` (published as `servo-net-traits`, `servo-script-traits`, `embedder_traits`, `layout_api`, `paint_api`, `fonts_traits`, `constellation_traits`, …) | The message/trait contracts between crates crossing IPC |
| `allocator` | Global allocator wiring — **servoshell defaults to jemalloc** (`tikv-jemallocator 0.6.1`) |

Ports and FFI:

- `ports/servoshell` — the reference browser shell: **egui 0.34** + `egui_glow` + `winit 0.30`
  chrome, CLI via `bpaf`, preference system (`--prefs-file`, `--pref k=v`), headless mode
  (`-z`), exit-after-stable-image (`-x`), multiprocess (`-M`) and sandbox (`-S`) toggles.
- `ffi/capi` — the C embedding API (with `cargo-c`-based tests in `tests/capi`), the surface
  third-party embedders (and wed-Ser Phase 3) are meant to build on.

External engine components consumed as dependencies at the pin:

| Component | Source at v0.6.0 | Notes |
|---|---|---|
| Stylo (CSS) | crates.io `stylo 0.21.0` (+`stylo_traits`, `stylo_dom`, `stylo_atoms`, `selectors 0.40`, `servo_arc`, `cssparser 0.37`) | Same CSS engine Firefox ships; on Servo `main` these switched to a git pin of `servo/stylo` — watch for churn |
| WebRender | crates.io `webrender 0.70` / `webrender_api` | GPU scene builder + renderer, as used by Firefox |
| SpiderMonkey | crates.io `mozjs =0.26.3` (alias `js`, features `intl`, `libz-sys`) | Vendored C++ built by the crate; `js_jit` feature enables JIT |
| ANGLE | `mozangle 0.7` | WebGL/EGL on Windows |

## 4. Process and threading model

- One **event loop** per pipeline (a document+script context); the **constellation** owns the
  pipeline tree and mediates navigation, history, and focus.
- All cross-component communication flows through `ipc-channel 0.22` channels carrying
  serde payloads defined in the `*_traits` crates — the same abstraction works within a process
  and across processes.
- **Multiprocess mode** is a default-on feature of servoshell (`-M` toggle; `multiprocess` in
  default features). Sandbox support goes through **`gaol 0.2.1`** (strongest on macOS/Linux;
  Windows sandboxing remains comparatively limited — see weaknesses).
- `background_hang_monitor` watches script/layout event loops — useful telemetry for our Phase 3
  "near-zero idle CPU" work.

## 5. Rendering pipeline (end to end)

1. DOM mutations wake the script event loop; style is computed by **Stylo** (same cascade,
   selector matching, and computed-value machinery Firefox uses).
2. `servo-layout` produces the fragment tree and display lists; flexbox/grid geometry is solved
   by **taffy**, keeping those modes independent from block/inline code.
3. Display lists are handed to **WebRender 0.70**, which builds a retained scene and renders on
   the GPU through **surfman** GL contexts (ANGLE on Windows). An alternative **vello** paint
   backend exists behind a feature flag — interesting future option for CPU-side rendering paths.
4. The compositor (inside `libservo`) owns the WebRender transaction loop, scrolling, zoom, and
   presents frames; the egui chrome (servoshell) is composited alongside page content via
   `egui_glow`.
5. Frame scheduling is demand-driven: Servo only produces new frames when something changes —
   a good baseline for our idle-CPU goals, to be measured and tightened in Phase 3.

## 6. JavaScript engine integration

- **SpiderMonkey** via the published `mozjs` crate (pinned `=0.26.3`), with `intl` (ICU) and JIT
  features. Servo pins the crate version exactly (`=`), which is how it controls the SM ABI.
- DOM bindings are generated into `script_bindings`; GC-rooting correctness is enforced at review
  time by **crown**, a custom rustc linter (this is why the pinned toolchain carries `rustc-dev`
  and `llvm-tools`).
- WebAssembly comes "for free" with SpiderMonkey. **Service Workers: unverified** — status to be
  established with targeted tests in Phase 2 before we plan enhancements.

## 7. Networking stack

From `components/net` and the workspace manifest at the pin:

- **HTTP:** hyper **1.10** (client, http1, http2) + hyper-util (legacy client, proxy support).
  Async runtime is tokio.
- **TLS:** rustls **0.23** via hyper-rustls 0.27 / tokio-rustls 0.26, with **aws-lc-rs** as the
  crypto provider; certificate verification via **rustls-platform-verifier** (OS trust store) with
  webpki-roots as bundled fallback. aws-lc-rs brings post-quantum key exchange support
  (`ml-kem`/`ml-dsa` appear in the dependency tree) — a genuine 2026-grade default.
- **WebSockets:** tungstenite 0.30 / async-tungstenite.
- **Content encoding:** brotli, gzip, zlib, zstd (async-compression).
- **State:** cookies, HSTS, and the HTTP cache persist via SQLite (rusqlite/sea-query).
- **CSP:** parsed and enforced with the `content-security-policy 0.8` crate.
- **Missing:** **no QUIC/HTTP3 dependency anywhere in the workspace** (no quinn/h3/s2n-quic).
  DNS is plain resolver + HSTS; **no DNS-over-HTTPS/TLS**. These are the Phase 2 transport gaps.

## 8. Strengths / weaknesses assessment (2026-10)

**Strengths**

1. Memory-safe Rust engine with production-proven CSS (Stylo) and GPU rendering (WebRender).
2. Modern transport: hyper 1.x, rustls + aws-lc-rs (PQC), platform cert verification, SQLite-backed persistence.
3. Embedder-first architecture: libservo + documented C API + LTS releases with immutable artifacts.
4. Real WebGPU (wgpu-core), WebGL (ANGLE), WebXR, GStreamer media incl. WebRTC, WebDriver/DevTools.
5. Excellent CI hygiene: reproducible `--locked` builds, official smoketest, packaging per platform.
6. jemalloc default and demand-driven compositing — good starting points for Phase 3 resource work.

**Weaknesses / gaps wed-Ser must address**

1. **No HTTP/3, no QUIC, no DoH/DoT** — Phase 2 transport work.
2. **Windows process sandboxing is weak** (gaol is strongest on macOS/Linux) — Phase 2 hardening.
3. **Media needs the GStreamer runtime** on Windows — must be bundled in Phase 5 packaging.
4. Layout mode coverage (tables/floats corner cases, `:has()`-dependent layout, container queries
   end-to-end) is **unverified** at the pin — Phase 2 will establish a conformance baseline via WPT runs.
5. Service Worker support status **unverified** — Phase 2 investigation.
6. Small ecosystem relative to Chromium: expect to find and fix engine bugs ourselves — that is the
   cost of the wedge we chose, mitigated by the LTS line.

## 9. Extension points wed-Ser will use

| Phase | Need | Hook |
|---|---|---|
| 3 | Browser shell / UI | Fork `ports/servoshell` (egui) or a new port over `libservo`; tab lifecycle via constellation traits |
| 3 | Memory/CPU | `allocator` crate (jemalloc tuning); compositor frame scheduling; tab discard at embedder level |
| 2 | HTTP/3 + DoH | `components/net` request pipeline (scheme/connector layer behind hyper); DNS resolver swap |
| 4 | Ad/tracker blocking | Network filter inside `components/net` fetch path + cosmetic filtering at DOM level in `script` |
| 4 | Fingerprint resistance | DOM/WebIDL implementations in `script` (canvas, WebGL, navigator, screen) — per-site farbling at the bindings layer |
| 5 | Packaging | `mach package` outputs per platform (tar.gz / MSI / zip) as the base for wed-Ser packaging |

Because Stylo and WebRender are external crates at our pin, our engine fork diffs stay confined to
`servo/servo` itself — component upgrades remain normal `Cargo.lock` bumps unless we need to patch
them, in which case we use the documented `[patch.crates-io]` mechanism.

## 10. Integration decision for this repository

- The engine is integrated as a **git submodule pinned to `v0.6.0`** (tag checkout, shallow).
- Rationale: wed-Ser Phase 1 needs a reproducible, upgradable pin; a submodule keeps our repo
  small and the pin explicit, while `[patch.crates-io]` + a Phase 2 fork (under the project owner's
  GitHub account, referenced by the submodule) will carry engine modifications.
- We deliberately did **not** vendor the full servo tree into this repository: it is ~2 GB of
  history and would make every clone and CI run pay that cost for no benefit.

## 11. Phase 2 planning inputs (gap → plan)

| Gap (from §8) | Plan |
|---|---|
| HTTP/3 + QUIC | Introduce `quinn` + `h3` behind a feature flag; negotiate via Alt-Svc; keep hyper/h2 as fallback |
| DNS-over-HTTPS/TLS | Swap system resolver for `hickory-resolver` with DoH/DoT transports; bootstrap via IP, cache pinned |
| Service Workers / CSP3 / COOP-COEP status | Establish WPT conformance baseline first, then close measured gaps |
| Windows sandbox | Evaluate ` Restricted process` options; at minimum, harden file/network broker checks |
| Benchmarks | Baseline: `mach build` times, WPT pass rates, and per-tab RAM via a harness built in Phase 3 |

## 12. Sources

- Pinned sources: `servo/` submodule at `v0.6.0` (workspace `Cargo.toml`, `rust-toolchain.toml`,
  `components/*/Cargo.toml`)
- Servo v0.6.0 CI: `.github/workflows/linux-common.yml`, `.github/workflows/windows.yml` at the tag
- Servo book (2026 structure): building guides (`building/linux`, `building/windows`) and design
  documentation — https://book.servo.org/
- Release metadata: https://github.com/servo/servo/releases (v0.6.0, LTS, 2026-09-29)
