# aslice Package Format

**Status:** Format draft, v0.1 — September 2026
**Companion to:** [DESIGN.md](DESIGN.md) — this document is the authoritative specification for §6 (Package Format). Where they disagree, this document wins.
**Scope:** the `package.toml` definition format, `build.star` build API, dependency and version semantics, transitive resolution, and lock files.

---

## 1. Philosophy

If you know npm's `package.json`, you know the shape of this format — and you should also know where we deliberately part ways with it:

| npm convention | aslice decision | Why |
|---|---|---|
| `scripts.postinstall` — arbitrary code at install | **Does not exist.** Binary installs execute zero package code (DESIGN §10.1) | npm's install scripts are its largest supply-chain hole; we start without one |
| Semver, loosely enforced | Semver-derived, **strictly normalized and validated** (§4) | A solver is only as good as its version algebra |
| `package-lock.json` | `aslice.lock`, first-class and machine-aware (§7) | Locks record µarch flavor — a dimension npm doesn't have |
| `dependencies` / `devDependencies` | `runtime` / `build` / `test`, plus **conditional** dependencies (§5.3) | Native builds have three distinct dependency lifecycles |
| Anything goes in unknown fields | Schema-validated, unknown fields **rejected** | Silent typos in package metadata are a real supply-chain bug class |

The format is **data first**: `package.toml` is pure TOML, validatable without executing anything. Build *logic*, where needed, lives in a separate hermetic Starlark file that runs only inside the build sandbox (DESIGN §10.5).

---

## 2. Files of a package

```
orchards/core/ffmpeg/
 ├── package.toml      # this specification (required)
 ├── build.star        # build logic (required iff [build].system = "custom")
 ├── tests.star        # smoke tests (required for core orchard)
 ├── patches/          # patch files, each checksummed in package.toml
 └── files/            # auxiliary files: launchd plists, default configs
```

Directory name **must** equal `package.name`. One directory = one package lineage (all versions of `ffmpeg` live in one formula; the orchard git history is the version history).

---

## 3. `package.toml` — full schema

Every formula carries `spec = 1` — the format version. Readers reject unknown `spec` values and unknown fields, so the format can evolve without silent misreads.

### 3.1 `[package]` — identity

```toml
spec = 1

[package]
name         = "ffmpeg"
version      = "7.1.0"          # normalized; see §4
revision     = 0                # orchard-side packaging revision (same upstream, new recipe)
epoch        = 0                # last-resort ordering override; see §4.4
license      = "LGPL-2.1-or-later"   # SPDX expression, validated against the SPDX list
description  = "Play, record, convert, and stream audio and video"
homepage     = "https://ffmpeg.org"
documentation = "https://ffmpeg.org/documentation.html"   # optional
maintainers  = ["alice <alice@example.com>"]              # at least one for core/extended
keywords     = ["video", "codec", "transcode"]            # powers `aslice search`
tier         = "core"           # core | extended
```

| Field | Required | Rules |
|---|---|---|
| `name` | yes | `[a-z0-9][a-z0-9-]*`, ≤ 64 chars, unique per orchard |
| `version` | yes | Normalized aslice version (§4); stored normalized, not verbatim-upstream |
| `revision` | yes | Integer ≥ 0; resets to 0 on any `version` change; bumped for packaging-only changes |
| `epoch` | no | Integer ≥ 0, default 0; §4.4 |
| `license` | yes | SPDX expression; `LicenseRef-` for unlisted licenses |
| `min_os` / `max_os` | no | §3.2 |
| `flavors` | no | §3.3 |
| `eol` / `deprecated` | no | Booleans; surfaced loudly by `audit` and at install time |

### 3.2 Platform bounds — minimum OS, maximum OS

```toml
[package]
min_os = "10.12"    # oldest macOS this formula builds/runs on; default "10.11"
max_os = "12"       # optional; omit unless upstream genuinely breaks on newer
```

- `min_os` sets the package's deployment target **and** its index visibility: a 10.11 machine simply never sees packages with `min_os = "10.12"` — filtered at solve time with an explicit "requires macOS ≥ 10.12" message, never a runtime surprise.
- The value is one of `"10.11"`, `"10.12"`, `"10.13"`, `"10.14"`, `"10.15"`, `"11"`, `"12"` — the supported window, verbatim.
- Honesty rule (DESIGN §4.1): declare the real floor. Do not contort a build to claim 10.11.

### 3.3 `flavors` — minimum instruction set

```toml
[package]
flavors = ["v2", "v3"]    # omit → all of ["v1", "v2", "v3"]
```

- A package that genuinely requires AVX2 (e.g., hand-written AVX2 kernels without a dispatch fallback) declares `flavors = ["v3"]` and the farm skips its `v1`/`v2` slices; v1/v2 machines get a clear solve-time message.
- The toolchain injects the flavor's `-march=x86-64-vN` floor automatically (§6.3); formula authors never write `-march` themselves. User `-march=native` requests layer on top at install time (DESIGN §7.4) without touching identity.
- `min_os` and `flavors` are orthogonal and both part of the build identity (DESIGN §7.2).

### 3.4 `[[source]]` — where the bits come from

```toml
[[source]]
url     = "https://ffmpeg.org/releases/ffmpeg-7.1.0.tar.xz"
sha256  = "40973d44…"           # required, always
mirrors = ["https://mirror.example/ffmpeg-7.1.0.tar.xz"]

[[source]]                        # multiple sources allowed; unpacked in order
git    = "https://github.com/example/plugin.git"
commit = "d34db33f…"            # full commit hash, required for git sources; tags alone are not pins
into   = "contrib/plugin"       # optional target subdirectory
```

Rules:

- **Every source is content-pinned.** Archives: sha256. Git: full commit hash. Mutable refs (`branch = "main"`) are rejected by `aslice lint`.
- **PGP verification** is optional and additional: `[source.pgp] key_url = "…", fingerprint = "…"` — the fingerprint is pinned in the formula, so key substitution still fails closed.
- Git submodules are forbidden; express them as additional `[[source]]` entries so they are pinned and mirrored.
- Patches live in `patches/` and are listed with their hashes:

```toml
[[patch]]
file   = "patches/0001-fix-darwin15-clock.patch"
sha256 = "a1b2c3…"
```

### 3.5 `[variants.*]` — feature switches with ABI honesty

```toml
[variants.x265]
default     = true
abi         = true        # changes the exported interface → enters build identity (DESIGN §7.2)
description = "HEVC encoding via x265"

[variants.debug]
default     = false
abi         = false       # build-flavor only; never enters identity
description = "Build with debug symbols"

[variants.lto]
default     = false
abi         = false
conflicts   = ["debug"]               # optional: mutually exclusive variants
requires    = []                      # optional: variant prerequisites
```

- `abi = true` variants are capped at 6 per package by policy (DESIGN §13.2) and each must name, in `description`, the interface it changes.
- Variants may carry their own platform bounds: `min_os = "10.13"`, `flavors = ["v2", "v3"]` inside a `[variants.*]` table narrow that variant's availability.

### 3.6 `[depends]` — dependencies

```toml
[depends]
runtime = [
  "x264 ^0.164",                   # version constraint (§5.2)
  "x265 ^3.5 ?variant.x265",       # conditional: only when our x265 variant is on (§5.3)
  "openssl ^3.0 || ^3.2",
  "sdl2 +metal ^2.28",             # require the provider built with its metal variant
  "macfuse ?os>=10.14",            # OS-conditional dependency
]
build   = ["nasm ^2.16", "pkgconf"]
test    = ["ffprobe-selftest"]
```

Full dependency semantics in §5.

### 3.7 Interop declarations — provides, conflicts, replaces

```toml
[provides]
blas = "3.11"          # virtual package provided, with version; satisfies `depends: ["blas ^3"]`

conflicts = ["ffmpeg4", "libav"]      # cannot be installed into the same profile
replaces  = ["ffmpeg4"]               # rename/supersede: upgrades replace the old package atomically
aliases   = ["ff"]                    # search/install aliases, no semantics
```

- `provides` is how interchangeable implementations interop: `openblas`, `blis`, and Accelerate-shim all `provide blas`; dependents name the virtual, the solver picks a provider (default: configurable per profile, `aslice profile prefer blas openblas`).
- `conflicts` is for *identity* conflicts. **File collisions are detected automatically** from manifests and never need declaration; profile-level priority resolves them (DESIGN §8.2).

### 3.8 `[install]` — declarative post-install behavior

Everything here is applied by aslice itself — not by package code:

```toml
[install]
links_priority = 50                     # collision priority in profiles; default 50

[[install.data_dir]]
path = "var/lib/postgresql"             # created at install, survives uninstall unless --purge
mode = "0750"

[[install.service]]                     # launchd, declaratively
label    = "org.aslice.postgresql"
plist    = "files/org.aslice.postgresql.plist"   # checksummed file from files/
scope    = "user"                       # user | system (system requires multi-user mode, DESIGN §10.4)

[install.completions]
bash = "share/bash-completion/ffmpeg"
zsh  = "share/zsh/site-functions/_ffmpeg"
```

### 3.9 `[audit]` — vulnerability matching and lifecycle

```toml
[audit]
cpe      = "cpe:2.3:a:ffmpeg:ffmpeg"   # binds the package to CVE feeds
eol      = false
eol_date = "2027-03-01"                # optional upstream EOL announcement
```

`aslice audit` joins installed packages against OSV/GitHub Advisory data via CPE and name aliases. Packages with `eol = true` require `--allow-eol` to install and are excluded from the core orchard (DESIGN §13.1).

### 3.10 `[build]` — the declarative build shortcut

```toml
[build]
system = "cmake"                        # autotools | cmake | meson | cargo | go | make | custom
args   = ["-DENABLE_GPL=ON", "-DENABLE_LIBX265=ON"]
```

If `system` is one of the known build systems, **no `build.star` is needed** — aslice runs the canonical phase sequence (configure with the system-typical flags, parallel build, DESTDIR install, then the ABI scan). `system = "custom"` requires `build.star` (§6). An autotools hello-world formula is literally `[package]` + `[[source]]` + `[build] system = "autotools"`.

---

## 4. Versioning

### 4.1 The version type

aslice versions are **SemVer 2.0 plus one extension**: an optional fourth numeric component for lettered-patch upstreams. Formally:

```
version  := epoch? core ("." patch4)? ("-" prerelease)? 
epoch    := <uint> "!"          (rarely needed; §4.4)
core     := <uint> "." <uint> "." <uint>
patch4   := <uint>              (only for upstreams with letter/scheme patches)
```

Ordering: epoch first, then numeric core components, then patch4, then prerelease (SemVer rules). Build metadata (`+…`) is **not** used — identity is the build_id's job (DESIGN §7.2), not the version's.

### 4.2 Normalization (upstream → aslice)

The orchard records `version` in normalized form; the upstream spelling lives in `source.upstream_version` when they differ:

| Upstream tag | Normalized | Rule |
|---|---|---|
| `v7.1` | `7.1.0` | Strip leading `v`; pad to three components |
| `1.3` | `1.3.0` | Pad |
| `2.0.0-rc.2` | `2.0.0-rc.2` | Prerelease passes through (SemVer spelling required) |
| `1.1.1k` | `1.1.1.11` | Letter patch → patch4 (`a`=1 … `z`=26) |
| `2024.09.1` | `2024.9.1` | CalVer with numeric components passes through |
| `r4520` / `git describe` hashes | **rejected** | Use epoch + a synthetic version, chosen by the maintainer |

Prereleases sort before their release (`7.1.0-rc.1 < 7.1.0`) and are **excluded from default resolution**: you get one only if you ask (`aslice install ffmpeg --prerelease` or an explicit constraint).

### 4.3 Constraint syntax

| Syntax | Meaning | Example matches |
|---|---|---|
| `"x264"` | any version | — |
| `"x264 ^0.164"` | compatible: `>=0.164.0 <0.165.0` (caret; for `0.x`, minor is pinned per SemVer convention) | `0.164.5` |
| `"openssl ~3.0.8"` | patch-level: `>=3.0.8 <3.1.0` | `3.0.14` |
| `"zlib =1.3.1"` | exactly this version | `1.3.1` |
| `"libpng >=1.6, <1.7"` | range intersection | `1.6.43` |
| `"openssl ^3.0 \|\| ^3.2"` | union | `3.0.15`, `3.2.1` |
| `"python *"` | any (explicit wildcard; same as bare name) | — |

Constraints compose with conditionals and provider-variant requirements in any order: `"x265 ^3.5 +asm ?variant.x265"`.

### 4.4 Revision and epoch

- **`revision`** bumps when the recipe changes but upstream doesn't (patch added, dependency range fixed, rebuild against a new ABI). `7.1.0-1` sorts after `7.1.0-0`. Revisions are orchard-visible and *not* part of upstream identity.
- **`epoch`** exists for the day an upstream changes its versioning scheme or a maintainer ships a mistaken higher version that must be rolled back: `epoch = 1` outranks every `epoch = 0` version regardless of the rest. It is a controlled break-glass mechanism, logged loudly at install time.

---

## 5. Dependency semantics

### 5.1 The three kinds

| Kind | Needed when | Propagates transitively? |
|---|---|---|
| `build` | compiling this package | **No** — build tools never leak into a consumer's closure |
| `runtime` | linking against or executing this package | **Yes** — the runtime closure is recursive |
| `test` | `tests.star` only | No |

A library consumer records *which provider build it linked against* via the ABI contract (DESIGN §7.3). Upgrading a provider whose `compatibility_version` and symbol fingerprint still cover its clients requires **no** dependent rebuilds; a provider that regresses forces the solver to either hold it back or plan dependent rebuilds — stated explicitly in the plan, never discovered later.

### 5.2 Transitive resolution

Resolution is PubGrub over the full graph (DESIGN §7.5), with these aslice-specific rules:

1. **Single-version-per-profile by default.** A profile links one build of a given name (the store can hold many; the *profile* points at one). Libraries needing side-by-side majors are separate package names: `openssl@3` and (if ever needed) `openssl@4` — an orchard naming convention, not a solver exception.
2. **Version unification.** If `a` needs `dep ^1.2` and `b` needs `dep ^1.4`, the profile gets one `dep` satisfying both (`^1.4`), or the solve fails with the conflict rendered as a derivation tree (`--explain`).
3. **Build deps float.** Two packages may build against different `nasm` versions without conflict; only runtime identity is unified in a profile.
4. **Cycles** are rejected at lint time for runtime edges; build-time cycles (rare, e.g., bootstrap compilers) require an explicit `bootstrap = true` edge annotation with a pinned seed slice.

### 5.3 Conditional dependencies

| Conditional | Meaning |
|---|---|
| `"x265 ?variant.x265"` | only when *this package's* `x265` variant is enabled |
| `"macfuse ?os>=10.14"` | only on matching OS releases |
| `"intel-mkl ?flavor>=v2"` | only when the target flavor is at least v2 (flavors are ordered `v1 < v2 < v3`) |

Conditionals are evaluated at solve time against the machine's OS and flavor and the chosen variant assignment — so the dependency graph the solver sees is exactly the graph that will be built.

### 5.4 Provider-variant requirements

`"sdl2 +metal ^2.28"` requires a provider whose build identity includes `+metal` (an `abi = true` variant). Because identity covers ABI variants, this is a hash-level match — the solver either finds a slice with that identity or plans a source build of the provider with that variant. There is no "close enough."

---

## 6. Build instructions

### 6.1 Phase model

Every build, declarative or custom, runs the same phase sequence under the sandbox profiles of DESIGN §10.5:

```
fetch → verify → unpack → patch → configure → build → install(staging) → abi-scan → test → pack(.slice) → sign
```

`fetch` is the only phase with network access. `abi-scan` is always run by aslice itself and cannot be skipped by a formula — the ABI contract is not optional metadata.

### 6.2 Declarative builds

`[build].system` covers the common cases with the system-idiomatic defaults: `configure`-with-prefix for autotools, out-of-tree `-DCMAKE_INSTALL_PREFIX` for cmake, `meson setup --prefix`, `cargo build --release` with `--locked` enforced (vendored or lockfile-pinned dependencies only — see §6.5), and so on. Per-phase overrides without going full `custom`:

```toml
[build]
system = "cmake"
args   = ["-DENABLE_GPL=ON"]
skip_tests = false
```

### 6.3 `build.star` — the custom API

Starlark, deterministic, no network, filesystem confined to the build dir (DESIGN §6.1). The `ctx` object is the entire capability surface:

| Member | Type | Meaning |
|---|---|---|
| `ctx.prefix` | string | Final store path this build will occupy |
| `ctx.staging` | string | DESTDIR staging directory |
| `ctx.jobs` | int | Parallelism granted by the scheduler |
| `ctx.flavor` | string | `"v1"` / `"v2"` / `"v3"` — the `-march=x86-64-vN` floor is already in `CC`/`CXX` wrappers |
| `ctx.min_os` | string | Deployment target; already exported as `MACOSX_DEPLOYMENT_TARGET` |
| `ctx.variant(name)` | fn → bool | Variant assignment for this build |
| `ctx.deps` | dict | name → store path of each resolved build+runtime dependency |
| `ctx.env` | map | Controlled environment; `ctx.env.set(k, v)` / `ctx.env.append(k, v)`; reads of host env are denied |
| `ctx.run(argv…)` | fn | Exec, argv-array only — **no shell**, no string interpolation attacks |
| `ctx.make(*args)`, `ctx.cmake(*args)`, `ctx.meson(*args)` | fn | Tool helpers with correct defaults |
| `ctx.user_cflags` / `ctx.user_ldflags` | string | User flags from install time (DESIGN §7.4); appended, recorded, never identity-affecting |
| `ctx.patch(file)` | fn | Apply a checksummed patch from `patches/` |

Environment determinism is set by the builder, not the formula: `LC_ALL=C`, `TZ=UTC`, `SOURCE_DATE_EPOCH` pinned to the source timestamp, prefix-mapping flags for reproducibility (DESIGN §9.5). A formula that needs something outside this API is a bug report against aslice, not a sandbox escape.

### 6.4 What the sandbox guarantees

Restating the security-relevant invariants that this format relies on: no network after `fetch`; no writes outside the build dir; no reads of the host environment; no code execution at slice-install time; `tests.star` runs network-free unless the formula declares `test_network = true` (loudly logged).

### 6.5 Language-ecosystem sub-managers

Crates, modules, and gems that fetch their own dependencies are allowed **only** in lockfile-pinned, vendored form: `cargo build --locked --offline` against a `[[source]]`-supplied vendor tarball, `go build` with a pinned `go.sum` and vendored modules, etc. Network access during build is denied, so any ecosystem dependency that isn't pinned in the formula fails closed and loudly.

---

## 7. Lock files

### 7.1 Who locks what

- **The orchard doesn't lock.** The signed index snapshot *is* the global lock: every package in it names exact versions, build identities, and source hashes.
- **The machine locks.** Every profile has a lock recording the exact resolved state. It lives at `/opt/aslice/profiles/<name>/aslice.lock` and is rewritten atomically with every generation (so `aslice rollback` restores both the symlinks *and* the lock that describes them).
- **Projects and fleets export locks.** `aslice lock export > aslice.lock` captures a profile for reproduction elsewhere; `aslice apply aslice.lock` replays it.

### 7.2 Format

```toml
lock_version = 1
generated_by = "aslice 0.3.0"
index_snapshot = "sha256:8f3a…"        # the TUF snapshot this lock resolved against

[machine]
os     = "12.7"
flavor = "v3"

[[package]]
name     = "ffmpeg"
version  = "7.1.0"
revision = 0
build_id = "2f4a9c1e"
flavor   = "v3"
min_os   = "10.11"
origin   = "slice"                     # slice | local-build
variants = { x265 = true, debug = false }
digest   = "sha256:9be4…"              # slice manifest digest (what was verified)

[[package]]
name     = "x264"
version  = "0.164.0"
revision = 1
build_id = "77aa10b2"
flavor   = "v3"
min_os   = "10.11"
origin   = "slice"
variants = {}
digest   = "sha256:c001…"
```

### 7.3 Portability semantics — exact by default, intent-preserving across flavors

A lock applied on a machine matching `machine.os`/`machine.flavor` reproduces **bit-identical build_ids** (verified against the same index snapshot or a newer one that still contains them). Applied on a *different* flavor or older OS, aslice re-resolves with the same versions and variants, swapping only build identities to the available flavor — and reports exactly what changed. Locks are thus exact where they can be and honest where they can't.

`--frozen` mode refuses any re-resolution: mismatch is an error, not an adaptation. That's the CI mode.

---

## 8. Validation and tooling

- **`aslice lint <formula>`** — full schema validation plus policy checks (name rules, license validity, unpinned sources, submodule use, cycle detection, variant caps, `min_os` plausibility against the toolchain). Orchard CI runs lint + a sandboxed build on every PR (DESIGN §13.4).
- **`spec` evolution** — new format versions are additive-only within a `spec` major; readers reject higher `spec` values rather than guessing. Breaking changes bump `spec` and ship with a mechanical migrator.
- **Unknown fields are errors** — including misplaced ones (`min_os` inside `[source]` fails lint, not silently ignored).

---

## 9. Worked examples

### 9.1 Minimal — zlib (no build.star at all)

```toml
spec = 1

[package]
name        = "zlib"
version     = "1.3.1"
revision    = 0
license     = "Zlib"
description = "Compression library"
homepage    = "https://zlib.net"
maintainers = ["maintainer <m@example.com>"]
tier        = "core"

[[source]]
url    = "https://zlib.net/zlib-1.3.1.tar.gz"
sha256 = "9a93b2b7…"

[build]
system = "cmake"
```

### 9.2 Version normalization — openssl-style letter patches

```toml
[package]
name    = "openssl1"
version = "1.1.1.23"            # upstream 1.1.1w (w = 23)
revision = 0
# …
[[source]]
url              = "https://www.openssl.org/source/old/1.1.1/openssl-1.1.1w.tar.gz"
sha256           = "cf309895…"
upstream_version = "1.1.1w"
```

### 9.3 Virtual provider — BLAS

```toml
[package]
name = "openblas"
version = "0.3.27"
# …
[provides]
blas = "3.11"
```

A dependent says `runtime = ["blas ^3"]`; the profile's provider choice (`openblas` by default) is recorded in the lock as the concrete package.

### 9.4 Full-featured — ffmpeg

See `orchards/core/ffmpeg/` in §2/§3: conditional deps, provider-variant requirements, variant caps, audit CPE, declarative service-free install. It is the reference formula the linter's test suite round-trips.

---

## Appendix. Field index

| Section | Fields |
|---|---|
| top-level | `spec` |
| `[package]` | `name` `version` `revision` `epoch` `license` `description` `homepage` `documentation` `maintainers` `keywords` `tier` `min_os` `max_os` `flavors` `eol` `deprecated` |
| `[[source]]` | `url` `git` `commit` `sha256` `mirrors` `into` `upstream_version` + `[source.pgp]` (`key_url`, `fingerprint`) |
| `[[patch]]` | `file` `sha256` |
| `[variants.*]` | `default` `abi` `description` `conflicts` `requires` `min_os` `flavors` |
| `[depends]` | `runtime` `build` `test` — entries: `name [constraint] [+variant] [?condition]` |
| interop | `provides` (map), `conflicts`, `replaces`, `aliases` |
| `[install]` | `links_priority`, `[[install.data_dir]]`, `[[install.service]]`, `[install.completions]` |
| `[audit]` | `cpe`, `eol`, `eol_date` |
| `[build]` | `system`, `args`, `skip_tests` |
| lock file | `lock_version`, `generated_by`, `index_snapshot`, `[machine]`, `[[package]]` |
