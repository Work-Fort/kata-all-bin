# kata-all-bin AUR Package Design

## Overview

A monolithic AUR package that installs all Kata Containers binary tooling from upstream static releases, with a GitHub Actions workflow that auto-detects new releases and pushes updates to the AUR.

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Package scope | Single monolithic package | Full install of all Kata tooling |
| Install path | `/opt/kata/` | Upstream layout, avoids fragile path rewriting and QEMU firmware breakage |
| Architectures | x86_64 + aarch64 | Major Arch Linux architectures |
| aarch64 tools | Build from source | Upstream only ships tools tarball for amd64 |
| Update detection | Cron-scheduled GitHub Actions | Daily check via gh CLI |
| Repo strategy | GitHub as source of truth | AUR repo is a deploy target |
| PATH setup | `/etc/profile.d/kata-containers.sh` | Adds `/opt/kata/bin` to PATH |

## Repository Structure

```
kata-all-bin/
├── PKGBUILD
├── kata-containers.sh              # profile.d script
├── .SRCINFO                        # auto-generated
├── .github/
│   └── workflows/
│       └── update.yml              # scheduled release checker + AUR push
└── .gitignore
```

## PKGBUILD Design

- **pkgname:** `kata-all-bin`
- **Sources (x86_64):** `kata-static-$pkgver-amd64.tar.zst`, `kata-tools-static-$pkgver-amd64.tar.zst`
- **Sources (aarch64):** `kata-static-$pkgver-arm64.tar.zst`, `kata-containers-$pkgver-vendor.tar.gz` (for building tools)
- **makedepends (aarch64 only):** `go`, `rust`, `cargo`
- **Install:** Extracts tarballs to `$pkgdir/opt/kata/`, installs profile.d script
- **Conflicts/provides:** `kata-runtime`, `kata-containers-image`, `linux-kata`

### Architecture Mapping

Upstream uses `amd64`/`arm64` in filenames; Arch uses `x86_64`/`aarch64`. PKGBUILD maps between them.

### aarch64 Tools Build

On aarch64, the `build()` function compiles tools from the vendored source tarball since upstream doesn't ship a prebuilt tools binary for ARM. makedepends for Go/Rust are conditional.

## GitHub Actions Workflow

- **Trigger:** `schedule: cron '0 6 * * *'` (daily at 06:00 UTC) + `workflow_dispatch`
- **Steps:**
  1. Fetch latest release tag from `kata-containers/kata-containers` via gh CLI
  2. Compare against current `pkgver` in PKGBUILD
  3. If different: update `pkgver`, download new tarballs, compute SHA256 sums, update PKGBUILD
  4. Generate `.SRCINFO` via `makepkg --printsrcinfo`
  5. Commit and push to AUR via SSH
- **Secrets:** `AUR_SSH_PRIVATE_KEY`

## Profile Script

`/etc/profile.d/kata-containers.sh` adds `/opt/kata/bin` to `$PATH`.
