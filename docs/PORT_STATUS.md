# Port status

Waves 0 to 2 are merged (state of 7 November 2026, `main` after PR 13). The
shared utilities, the bit I/O, the DSP, SharpYUV, the lossless shared code,
both lossless codecs (VP8L decoder and encoder), the VP8 decoder core and the
encoder's picture and configuration units are translated and verified against
the C oracle in its release, debug and asan-ubsan variants, on macOS ARM64 and
Linux x86-64. Nothing of the public API is exported yet: the integration
packages (WP12, WP13), the demuxer and muxer, the animation code and the safe
Rust APIs are wave 3 and 4. The integrator aggregates this page from the package
pages in [docs/status](status/) after each wave; packages never edit it.

## Packages

| Package | Sub-wave | Status | Page |
| --- | --- | --- | --- |
| WP0 bootstrap | 0 | done (local host and CI) | [wp0](status/wp0.md) |
| WP1 types, memory, workers | 1a | done, PR 1 (PR 6 excludes zero-size allocations from the differential) | [wp1](status/wp1.md) |
| WP2 bit I/O | 1b | done, PR 4; readers store positions, the input is passed per call (ADR 0003, "Readers over input") | [wp2](status/wp2.md) |
| WP3 lossless shared | 1c | done, PR 7; four `lossless_common.h` inline functions moved to WP11 | [wp3](status/wp3.md) |
| WP4 pixel ops | 1b | done, PR 5; the rescaler owns its side rows and takes the output per call since PR 9, and exports by a signed stride since PR 12 | [wp4](status/wp4.md) |
| WP5 SharpYUV | 1b | done, PR 3 | [wp5](status/wp5.md) |
| WP6 VP8 decoder core | 2b | done, PR 12; the `put` hook and the alpha rows run on the calling thread at `Sync` (ADR 0003, "Threads") | [wp6](status/wp6.md) |
| WP7 VP8L decoder | 2a | done, PR 10; decoding context per call (ADR 0003) | [wp7](status/wp7.md) |
| WP8 demux | 3b | not started | [wp8](status/wp8.md) |
| WP9 encoder picture and config | 2a | done, PR 8 (PR 11 adapts `RescalePlane` to the PR 9 rescaler); shared views (ADR 0003) | [wp9](status/wp9.md) |
| WP10 VP8 encoder core | 2c | not started; its dependencies are merged | [wp10](status/wp10.md) |
| WP11 VP8L encoder | 2b | done, PR 13; the side context runs on a scoped thread | [wp11](status/wp11.md) |
| WP12 decoder integration | 3a | not started; its dependencies are merged | [wp12](status/wp12.md) |
| WP13 encoder integration | 3b | not started | [wp13](status/wp13.md) |
| WP14 animation decode and demux ABI | 3c | not started | [wp14](status/wp14.md) |
| WP15 mux and animation encode | 4 | not started | [wp15](status/wp15.md) |
| WP16 safe decoder API | 4 | not started | [wp16](status/wp16.md) |
| WP17 safe encoder API | 4 | not started | [wp17](status/wp17.md) |
| WP18 SIMD x86 decoder | 4 | not started | [wp18](status/wp18.md) |
| WP19 SIMD x86 encoder | 4 | not started | [wp19](status/wp19.md) |
| WP20 NEON | 4 | not started | [wp20](status/wp20.md) |
| WP21 Ferrixel `webp-read` | 5 | not started; tracked in the Ferrixel repository | none |
| WP22 Ferrixel `webp-write` | 5 | not started; tracked in the Ferrixel repository | none |

The sub-waves follow libwebp's call graph. [WORKPLAN.md](WORKPLAN.md) lists
each package's dependencies and the function- and type-level exceptions, and
`tools/check.py --oracle` checks them against the oracle's cross-unit
references.

## Integration notes

- **Representation decisions** taken between the packages are recorded as dated
  amendments of [ADR 0003](adr/0003-rust-representation.md): readers over input
  (WP2), untyped sample buffers (WP5), raw data and buffers and planes (WP1,
  WP9), shared views (WP9), decoding context per call (WP7) and the thread rule
  for workers that outlive a call (WP6) or live inside one (WP11). Each package
  page lists its not-applicable checks, its PORTING rule 8 exclusions and its
  recorded deviations, and ends with notes for the packages that build on it.
- **Interface changes after a merge** go through their own PR or commit, and the
  next PR is checked against the new `main` before it merges: PR 8 and PR 9
  were each green on their own branch and did not compile together (PR 11 fixed
  `RescalePlane`). PR 12 and PR 13 were compiled together before merging.
- **Open for wave 3.** WP10 (VP8 encoder core) and WP12 (decoder integration)
  can start; WP8 and WP13 follow WP12. The incremental decoder (WP12) must pass
  the same `data` and `alpha_data` slices to every call of one decoding, as the
  WP6 notes describe.

## Oracle baseline

`python3 tools/oracle_baseline.py` runs upstream's `test_dwebp.sh` and
`test_lossless.sh`, unmodified, with the oracle's own `dwebp` on a copy of the
pinned test data (`260804d`):

| Script | Result |
| --- | --- |
| `test_dwebp.sh` | passes: all 480 checksums of 96 vectors (five formats each) verified twice, with and without `-noasm`, 960 verifications |
| `test_lossless.sh` | passes: 33 vectors (32 `lossless_vec_*` and `lossless_color_transform.webp`) in five formats, with and without `-noasm`, 330 comparisons |

The test data was first pinned at `06ddd96`, whose last two commits regenerate
seven PGM checksums and three PGM reference images for libwebp `0d14d84b`, a
change made after v1.6.0. `--diagnose-post-v1.6.0` still compares against those
references: 473 of 480 checksums match, the seven others being exactly the
regenerated PGM entries (`bad_palette_index`, `color_cache_bits_11`,
`lossless1` to `lossless4`, `near_lossless_75`), and none of the 33 lossless PGM
outputs matches the regenerated images.

Recorded on 6 November 2026, after the re-pin, on macOS 27.0.1 (build 26A434), Apple M1 Ultra
(ARM64), with Apple clang 21.0.0 (clang-2100.3.34.2) and CMake 4.4.5. The
`release`, `asan-ubsan` and `debug` oracle variants give identical counts.
`test_dwebp.sh` needs an `md5sum` that accepts `-c -`. No substitution was
needed on this host: `md5sum` resolves to `/sbin/md5sum`, the FreeBSD-derived
`md5` of macOS in its GNU-compatible mode, which prints `name: OK` lines like
GNU coreutils. The script passes it explicitly as `--md5exec`; on macOS releases
before 14, which lack `md5sum`, it falls back to `gmd5sum` from Homebrew
coreutils. On the Ubuntu 26.04 CI runner, `md5sum` resolves to the uutils
coreutils binary (`/usr/lib/cargo/bin/coreutils/md5sum`), which accepts `-c -`
as well.

The first CI run reproduced the same counts for all three variants on the
Linux x86-64 and macOS ARM64 runners listed under Platform evidence.

## Platform evidence

| Host | Evidence |
| --- | --- |
| macOS ARM64, Apple clang 21, Rust 1.99.0 stable and 1.95.0 | `python3 tools/check.py --oracle --msrv`, the WP0 test against all three oracle variants, the baseline above |
| Linux x86-64, GitHub runner `ubuntu-26.04` (image 20261101.186.1), Ubuntu clang 21.1.8, CMake 4.4.5, Rust 1.99.1 stable and 1.95.0 | [CI run 42128339121](https://github.com/sebastian-software/libwebp-rs/actions/runs/42128339121) on `6e0315b`, 6 November 2026: `python3 tools/check.py`; the MSRV job (`--stage test --msrv`, build and tests on Rust 1.95.0); the oracle job, in which all three variants build natively (the first native evidence for the `-msse2` fix of ADR 0002), `tools/unit_dependencies.py --check` finds no undeclared reference, the WP0 test passes in all three profiles against the `release` and `asan-ubsan` oracles (LeakSanitizer active by default), and the baseline above passes for all three variants |
| macOS ARM64, GitHub runner `macos-26-arm64` (image 20261026.0455.2), Apple clang 21.0.0 (clang-2100.1.1.101), CMake 4.4.4, Rust 1.99.1 stable | same run: `python3 tools/check.py` and the oracle job as on Linux |
