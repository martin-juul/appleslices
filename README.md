# appleslices

A package manager for Intel Macs — macOS 10.15 (Catalina) through 12 (Monterey), with and without AVX2.

A successor-in-spirit to Homebrew for the platform Homebrew is leaving behind:

- **Prebuilt binaries** for the common flavors (x86-64-v2 baseline and x86-64-v3/AVX2), hosted on GitHub with mirror-friendly fallback
- **User-selectable build flags and variants** — compile your own without losing interoperability with the prebuilt world (ABI-aware substitution)
- **A stronger security model** — declarative formulae, sandboxed builds, TUF-signed metadata, code-free binary installs, no sudo in steady state
- **A stronger performance model** — single C++20 binary, sub-10ms startup, parallel solver and downloads, zstd payloads, atomic rollback-capable generations

## Status

Design phase. See the [design document](docs/DESIGN.md) (written under the working title "Scrumpy" — `appleslices` is the candidate project name).

## Scope

- macOS 10.15, 11, 12 — Intel x86_64 only
- No Apple Silicon, no macOS 13+ targets, by explicit design
