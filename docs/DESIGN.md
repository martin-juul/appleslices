# Scrumpy — A Package Manager for Intel macOS

**Working title.** "Scrumpy" (traditional farmhouse cider) is a placeholder — a nod to the Macintosh apple, deliberately distinct from Homebrew's beer vocabulary to avoid community confusion and trademark friction. Alternatives considered: *Wort*, *Grist*, *Press*, *Intelbrew* (coined publicly by a Homebrew maintainer, but likely to breed confusion). Final naming is an open decision.

- **Status:** Design draft, v0.1 — September 2026
- **Scope:** macOS 10.15 (Catalina) through 12 (Monterey), Intel x86_64 only
- **Implementation:** C++20 core, single self-contained binary
- **Audience:** Maintainers, founding contributors, and early reviewers

---

## 1. Context and Opportunity

### 1.1 The gap that is opening

The Intel-Mac package-management ecosystem is losing its maintainer on a known schedule:

- **Homebrew 7.0.0 (September 2026)** moved Intel x86_64 to Tier 3 ("not supported"): no new bottles are built for Intel, CI coverage is gone, and macOS 10.15 support was removed outright. Existing bottles stay hosted but freeze in time.
- **September 2027:** Homebrew plans to remove the ability to run on Intel systems at all.
- **Autumn 2027:** GitHub Actions retires its Intel macOS runners (`macos-13` images), eliminating the last hosted CI capable of natively building x86_64 macOS binaries. This is one of the explicit reasons Homebrew cites for its exit.
- **Apple:** macOS 26 Tahoe is the final release for Intel Macs; macOS 27 Golden Gate is Apple-silicon-only. Intel Macs receive security updates only, on a countdown.

Meanwhile a large installed base remains: Homebrew's own analytics, discussed publicly in mid-2026, put Intel at roughly a quarter of active Homebrew Mac installations. Catalina, Big Sur, and Monterey are precisely the releases these machines are stranded on — and precisely the window Homebrew has already abandoned.

### 1.2 Who the users are

Three populations, all underserved:

1. **Owners of 2012–2020 Intel Macs** used as daily drivers, home servers, audio rigs, and build machines. Many are maxed-out machines (Mac Pro 2013, iMac 5K, 16" MBP 2019) that remain genuinely capable.
2. **CI and legacy-maintenance shops** that must keep building and testing x86_64 macOS software through Tahoe's support window.
3. **OpenEmu / retro / audio / lab environments** pinned to old macOS releases for driver or 32-bit-adjacent reasons (Catalina is the first 64-bit-only release and the last before Big Sur's dyld-cache changes broke some workflows).

### 1.3 Why not MacPorts or Nix?

Both are suggested by Homebrew itself as migration paths. Both leave the opening intact:

| | MacPorts | Nix (via Determinate/nixpkgs) |
|---|---|---|
| Binary coverage for 10.15–12 Intel | Partial, shrinking; builds from source as the norm | x86_64-darwin support degrading; Hydra builds for old OS releases not a goal |
| Variant/flag model | Excellent (`+variants`) — but every variant is a local compile | Binary cache keyed on exact derivation; any flag change = full local rebuild |
| UX | Functional but austere | Steep learning curve |
| Philosophy | Build on the *target* OS version | Hermetic, prefix-independent |

The opening is a manager that combines **Homebrew's ergonomics**, **MacPorts' variant flexibility**, and **Nix's correctness ideas** (store paths, generations, atomic switching) — scoped tightly to a platform the incumbents are vacating, small enough to be excellent.

### 1.4 Design thesis

> **One OS axis collapses; one µarch axis matters. Variants are safe if you separate ABI from optimization. Security comes from making packages declarative and installs code-free.**

Each clause is developed in its own section below.

---

## 2. Goals and Non-Goals

### 2.1 Goals

- **G1 — Full coverage of macOS 10.15, 11, and 12 on Intel**, treated as first-class citizens, not legacy tiers.
- **G2 — Two µarch flavors:** `v2` (baseline: SSE4.2/POPCNT — every CPU that can run Catalina) and `v3` (AVX2 — Haswell and later). See §4.
- **G3 — Precompiled binaries for the common flavors**, hosted on GitHub infrastructure with a mirror-friendly fallback. Default install path is binary and near-instant.
- **G4 — User-selectable build flags and feature variants** with local compilation — *without* forfeiting interoperability with prebuilt packages (§7).
- **G5 — A materially better security model than Homebrew's** (§10): declarative package definitions, sandboxed builds, signed everything, code-free binary installs, no `/usr/local` chown, no sudo in steady state.
- **G6 — A materially better performance profile** (§11): sub-10 ms CLI startup, parallel solver and downloads, zstd payloads, APFS-aware linking.
- **G7 — Atomic, rollback-capable installations** via generations (§8).
- **G8 — Low maintainer burden.** The platform is frozen by Apple; the design exploits that stability instead of fighting it (§15).

### 2.2 Non-goals

- **N1 — Apple Silicon.** Not now, not by accident. The architecture must not preclude it, but no engineering effort goes to it. Homebrew owns that space.
- **N2 — macOS 13+ on Intel.** Tahoe-era Intel machines (2019–2020) are welcome, but the build targets remain 10.15–12; Ventura+ Intel gets whatever falls out naturally.
- **N3 — GUI applications (Casks) at launch.** A declarative app-install format is designed (§12.4) but core packages come first.
- **N4 — Linux/Windows.** The codebase should stay portable, but no effort is spent there.
- **N5 — Replacing the system.** Scrumpy never touches `/usr`, `/System`, or `/usr/local`'s ownership. It lives in its own prefix.

---

## 3. Positioning: "What Homebrew Could Not"

Homebrew's structural constraints — not its maintainers — produced its weaknesses. Scrumpy's founding decisions target each one:

| Homebrew constraint | Consequence | Scrumpy's founding decision |
|---|---|---|
| Formulae are arbitrary Ruby executed at install time | Taps and `post_install` are a code-execution supply chain; audit is impossible to automate fully | Formulae are **declarative TOML + hermetic Starlark build scripts** (§6); binary installs execute **zero** package code (§10.3) |
| Bottles exist only for default options; `homebrew-core` removed options entirely (v2.0, 2019) | Users needing flags lose binaries *and* break interop | **ABI-aware variant model** (§7): optimization flags never affect identity; feature flags affect it only when they change the exported interface |
| One linked version per package in the Cellar | Upgrades are destructive; rollback is archaeology | **Store paths + generations** (§8): any number of variants coexist; switching is atomic |
| Prefix ownership of `/usr/local` chowned to the user | Security researchers have criticized this for a decade | Private prefix `/opt/scrumpy`, created once by an installer, never world-writable, never sudo thereafter (§10.4) |
| Ruby runtime, git-cloned taps | Slow startup, slow `brew update` | Single C++ binary, content-addressed TUF-signed index with snapshot diffs (§11) |
| CI hostage to GitHub-hosted Intel runners | The current collapse | **Self-hosted build farm on real Intel hardware** from day one (§9.3) |

---

## 4. Platform Matrix and Microarchitecture Strategy

### 4.1 The OS axis collapses

Naively, 3 OS versions × 2 µarch flavors = 6 builds per package. In practice it is **2**:

macOS has a mature deployment-target mechanism. A binary compiled with `-mmacosx-version-min=10.15` against the Catalina SDK (or a later SDK with disciplined use of availability annotations and weak linking) runs correctly on 10.15, 11, and 12. This is how MacPorts-adjacent projects and most commercial software ship single binaries across OS releases.

So the default is: **build once against the 10.15 target, per µarch flavor.**

Exceptions exist and are handled explicitly:

- Packages that *require* post-Catalina APIs (e.g., need `NSFileProvider` revision, newer Security framework surfaces) declare `min_os = "11"` or `"12"` in their formula. The index simply filters them per-OS, and their binaries are built against the newer target.
- Big Sur's dyld shared cache change (system dylibs no longer exist on disk) breaks build systems that probe for `/usr/lib/libfoo.dylib` file existence. Because we build *on the oldest supported OS* (or with the oldest SDK), this classic failure mode is avoided by construction, and formulae carry `patch` stanzas for the stubborn cases.

### 4.2 The µarch axis is real and worth serving

The Catalina–Monterey Intel population splits cleanly on AVX2 (introduced with Haswell, 2013):

- **No AVX2 (v2 required):** Mac Pro 2013 (Ivy Bridge-EP — the big one), iMac/MacBook Pro/Mac mini 2012, and the 2010–2012 Mac Pros with upgraded GPUs. These machines top out at Catalina or Big Sur.
- **AVX2 (v3 capable):** everything 2014+ — iMac 5K, MacBook Pro 2015–2020, Mac mini 2018, iMac Pro, Mac Pro 2019. Monterey's supported list is entirely AVX2-capable.

Scrumpy adopts the x86-64 psABI microarchitecture levels as its flavor vocabulary:

| Flavor | Level | Key ISA | Who needs it |
|---|---|---|---|
| `v2` | x86-64-v2 | SSE4.2, POPCNT, SSSE3 | Baseline — runs on every Catalina-capable Mac |
| `v3` | x86-64-v3 | AVX2, BMI2, FMA | Haswell+; 10–40% faster on codecs, crypto, compression, math |

Notes:

- **x86-64-v4 (AVX-512) is deliberately absent.** No Intel Mac ever shipped AVX-512. The flavor space is exactly two, keeping the binary matrix and the UX small.
- Detection is one `sysctlbyname("hw.optional.avx2_0")` at install time. The manager itself is built `v2` (it gains nothing from AVX2) and selects flavors on the user's behalf; `scrumpy config set flavor v2` overrides.
- Binaries are clearly tagged (`+v2` / `+v3` in the build identity, §7.2) so a v3 binary can never be selected on a v2 machine — the solver treats flavor as a hard constraint, not a preference.

### 4.3 Toolchain floor

- Minimum build host: Xcode 12.4 CLT (last Catalina SDK) for the baseline builders; any later CLT that can still target 10.15 works elsewhere.
- The package manager *core* is C++20, built with `-mmacosx-version-min=10.15`, statically linking libc++ and all third-party libraries, dynamically linking only `libSystem`. Result: one Mach-O binary that runs on all three OS versions with zero runtime dependencies. (Fully static linking is impossible on macOS — `libSystem` must be dynamic — but nothing else need be.)
- A self-hosted `scrumpy-toolchain` package (modern Clang/LLD, CMake, Ninja, pkgconf) is a phase-2 deliverable so package *builds* aren't hostage to aging CLTs (§14).

---

## 5. Core Architecture

### 5.1 Process layout

```
scrumpy (single binary, unprivileged)
 ├── scrumpy-fetch     ── sandboxed helper: network + disk cache only
 ├── scrumpy-extract   ── sandboxed helper: archive extraction only
 ├── scrumpy-build     ── sandboxed helper: runs Starlark build scripts
 └── scrumpy-link      ── the only component that writes the store/profile
```

Privilege separation is structural: the helpers are separate executables (spawned by the main binary, which re-executes itself with a subcommand) running under Seatbelt profiles (§10.5) with exactly the capabilities their phase requires. The fetch helper can't touch the store; the extractor has no network; the linker has no network and no compiler. Each helper is small (a few hundred lines) and independently auditable — this is where the C++ attack-surface discipline pays for itself.

### 5.2 Major components

| Component | Responsibility | Notes |
|---|---|---|
| **Index client** | Fetches and caches the package index | TUF metadata + zstd-compressed JSON snapshots; incremental updates via snapshot diffs, not git |
| **Solver** | Version + variant resolution | PubGrub-style CDCL algorithm over (name, version, variant) space; flavors as hard constraints (§7.5) |
| **Store** | Content- and identity-addressed package trees | `/opt/scrumpy/store/<name>-<version>-<buildid>/` (§8) |
| **Profiles / generations** | Atomic merged views | Symlink forests with rename-swap; rollback = flip a symlink (§8.3) |
| **Builder** | Fetch→unpack→patch→configure→build→install in sandbox | Deterministic environment; DESTDIR staging; ABI scan on output (§7.3) |
| **Verifier** | Signature, hash, ABI, and policy checks before linking | Nothing reaches the profile without passing (§10.2) |
| **Database** | Installed-set and metadata | Single SQLite file, WAL mode, prepared statements; the only mutable state besides the store |
| **Reporter** | SBOM generation, `audit`, provenance display | SPDX SBOM per package; OSV feed integration (§10.6) |

### 5.3 Why C++ — and what it costs

The user-facing case for C++ is startup time, single-binary deployment across 10.15–12 with no runtime story, direct Mach-O/dyld/Seatbelt API access, and world-class tooling for the performance goals. The honest cost is memory safety, which is a security-goal liability. Scrumpy treats that as an engineering constraint, not an embarrassment:

- **Disciplined subset:** no owning raw pointers (RAII everywhere, `std::unique_ptr`/`shared_ptr` at boundaries), bounds-checked views (`std::span`, `string_view` with explicit lifetime rules), no C arrays, no `str*`/`mem*` libc string calls, exceptions banned across module boundaries.
- **Hardened build:** `-fstack-protector-strong -fstack-clash-protection -D_FORTIFY_SOURCE=2` (via libc++ equivalents), full RELRO-analog (`-Wl,-bind_at_load` where tolerable), PIE, CFI under LTO (`-fsanitize=cfi`) for release builds once lld/ld64 support is verified per-OS.
- **CI sanitizers:** every PR runs the test suite under ASan+UBSan on both flavors; parsers and the archive extractor are continuously fuzzed with libFuzzer — formula parsing, manifest parsing, tar/zip extraction, and the index parser are all *untrusted-input surfaces* and are treated accordingly.
- **The trust-critical helpers are tiny.** fetch/extract/link together are the only code paths that touch hostile data with ambient authority, and each is kept small enough to review line-by-line.

---

## 6. Package Format

### 6.1 Formulae are data, with a hermetic build script

A Scrumpy package is a directory in a tap (git repo):

```
taps/core/ffmpeg/
 ├── package.toml      # metadata, sources, dependencies, variants
 ├── build.star        # Starlark build script (sandboxed, no IO escape)
 ├── patches/          # optional, checksummed
 └── tests.star        # optional smoke tests
```

`package.toml`:

```toml
[package]
name        = "ffmpeg"
version     = "7.1"
revision    = 0
license     = "LGPL-2.1-or-later"
description = "Play, record, convert, and stream audio and video"
homepage    = "https://ffmpeg.org"

[source]
url    = "https://ffmpeg.org/releases/ffmpeg-7.1.tar.xz"
sha256 = "40973d449e3c3a4a551b3e2e05f5a28f8ff74a2f2e0c2e6ec4f7f4b9c0f2a1c9"
# mirrors = ["https://mirror.example/..."]   # optional fallback mirrors

[variants.x265]          # feature variant
default = true
abi     = true           # changes exported interface → part of build identity (§7)
description = "HEVC encoding via x265"

[variants.debug]
default = false
abi     = false          # optimization/debug flags never affect identity
description = "Build with debug symbols"

[depends]
runtime = ["x264", "x265?variant.x265", "lame", "opus", "srt"]
build   = ["nasm", "pkgconf"]
```

`build.star` (Starlark: deterministic, no network, no filesystem access outside the build dir, no `eval`):

```python
def configure(ctx):
    args = [
        "--prefix=" + ctx.prefix,
        "--enable-gpl",
        "--enable-libx264",
    ]
    if ctx.variant("x265"):
        args.append("--enable-libx265")
    ctx.env.append("CFLAGS", ctx.user_cflags)   # user flags honored, recorded, non-ABI (§7.4)
    ctx.run("./configure", *args)

def build(ctx):
    ctx.make(jobs = ctx.jobs)

def install(ctx):
    ctx.make("install", destdir = ctx.staging)
```

Key properties:

- **No Turing-complete host code at install time.** Starlark executes only during *builds*, inside the sandbox, with capabilities enumerated in `ctx`. There is no `post_install` hook that runs on the user's machine — post-install behavior (creating data dirs, registering launch agents) is expressed declaratively in `package.toml` and executed by Scrumpy itself. (Homebrew 7.0 is migrating the same direction with `*_steps`; Scrumpy simply starts there.)
- **Everything is pinned.** Source URLs carry hashes; patches are checksummed files; the index records the full closure.
- **Variants are declared, typed, and ABI-tagged** by the package author — the foundation of the interop model in §7.

### 6.2 Binary package format (`.spk`)

A binary package ("**flagon**" in the working vocabulary — cider ships in flagons) is:

```
ffmpeg-7.1-0+core.v3.2f4a9c1e.spk
 ├── manifest.json     # identity, ABI contract, file list w/ hashes, SBOM, provenance
 ├── payload.tar.zst   # the tree, zstd-19 --long compressed
 └── signature         # minisign/cosign signature over the above (§10.2)
```

Install of a `.spk` is: verify signature → verify payload hashes → extract into store path → ABI-check against the packages that will link to it → register in SQLite → link into profile. **No code from the package executes at any point.**
