# kata-all-bin Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Create an AUR package that installs all Kata Containers binaries, with a GitHub Actions workflow for automatic updates.

**Architecture:** Monolithic AUR package installing prebuilt static binaries to `/opt/kata/`. On x86_64, both `kata-static` and `kata-tools-static` tarballs are extracted directly. On aarch64, `kata-static` is extracted and tools are built from source (upstream doesn't ship an aarch64 tools tarball). A scheduled GitHub Actions workflow detects new releases and pushes updates to the AUR.

**Tech Stack:** PKGBUILD (bash), GitHub Actions YAML, `gh` CLI, `KSXGitHub/github-actions-deploy-aur` action

---

### Task 1: Initialize git repository

**Files:**
- Create: `.gitignore`

**Step 1: Initialize git repo**

Run: `git init`

**Step 2: Create .gitignore**

```
*.pkg.tar.zst
*.tar.zst
*.tar.gz
*.tar.xz
src/
pkg/
*.log
```

**Step 3: Commit**

```bash
git add .gitignore
git commit -m "chore: initialize repository"
```

---

### Task 2: Create profile.d script

**Files:**
- Create: `kata-containers.sh`

**Step 1: Write the profile script**

```bash
#!/bin/sh
export PATH="/opt/kata/bin:$PATH"
```

**Step 2: Commit**

```bash
git add kata-containers.sh
git commit -m "feat: add profile.d script for PATH setup"
```

---

### Task 3: Create the PKGBUILD

**Files:**
- Create: `PKGBUILD`

This is the core of the project. Key design decisions embedded in the PKGBUILD:

- **Architecture mapping:** Upstream uses `amd64`/`arm64`; Arch uses `x86_64`/`aarch64`
- **x86_64:** Downloads both `kata-static` and `kata-tools-static` prebuilt tarballs
- **aarch64:** Downloads `kata-static` prebuilt tarball + source + vendor tarball, builds Rust tools with gnu target (uses system `rust` package, no rustup needed)
- **Install path:** `/opt/kata/` (upstream layout, no path rewriting)
- **`!strip`:** Prevents stripping prebuilt binaries (could break QEMU/hypervisors)

**Step 1: Write the PKGBUILD**

```bash
# Maintainer: TODO <TODO>
pkgname=kata-all-bin
pkgver=3.27.0
pkgrel=1
pkgdesc='Kata Containers - lightweight VMs for container isolation (prebuilt binaries)'
arch=('x86_64' 'aarch64')
url='https://github.com/kata-containers/kata-containers'
license=('Apache-2.0')
depends=()
makedepends_aarch64=('rust' 'gcc' 'cmake' 'protobuf')
optdepends=(
    'containerd: container runtime interface'
    'docker: container runtime'
)
provides=('kata-containers' 'kata-runtime')
conflicts=('kata-runtime-bin' 'kata-containers-image-bin' 'linux-kata-bin')
options=('!strip' '!debug')

source=("kata-containers.sh")
source_x86_64=(
    "https://github.com/kata-containers/kata-containers/releases/download/${pkgver}/kata-static-${pkgver}-amd64.tar.zst"
    "https://github.com/kata-containers/kata-containers/releases/download/${pkgver}/kata-tools-static-${pkgver}-amd64.tar.zst"
)
source_aarch64=(
    "https://github.com/kata-containers/kata-containers/releases/download/${pkgver}/kata-static-${pkgver}-arm64.tar.zst"
    "kata-containers-${pkgver}.tar.gz::https://github.com/kata-containers/kata-containers/archive/refs/tags/${pkgver}.tar.gz"
    "https://github.com/kata-containers/kata-containers/releases/download/${pkgver}/kata-containers-${pkgver}-vendor.tar.gz"
)
sha256sums=('SKIP')
sha256sums_x86_64=('SKIP' 'SKIP')
sha256sums_aarch64=('SKIP' 'SKIP' 'SKIP')

prepare() {
    if [[ "$CARCH" == "aarch64" ]]; then
        # Overlay vendored dependencies onto source tree for offline Rust builds
        cd "$srcdir/kata-containers-${pkgver}"
        tar -xzf "$srcdir/kata-containers-${pkgver}-vendor.tar.gz" 2>/dev/null || true
    fi
}

build() {
    if [[ "$CARCH" == "aarch64" ]]; then
        cd "$srcdir/kata-containers-${pkgver}"
        for tool in kata-ctl agent-ctl genpolicy trace-forwarder; do
            make -C "src/tools/$tool" LIBC=gnu
        done
    fi
}

package() {
    # Install kata-static (common to both archs — extracts to opt/kata/ in srcdir)
    install -d "$pkgdir/opt"
    cp -dr --no-preserve=ownership "$srcdir/opt/kata" "$pkgdir/opt/kata"

    if [[ "$CARCH" == "aarch64" ]]; then
        # Install tools built from source
        cd "$srcdir/kata-containers-${pkgver}"
        local _target="aarch64-unknown-linux-gnu"
        install -Dm755 "src/tools/kata-ctl/target/${_target}/release/kata-ctl" "$pkgdir/opt/kata/bin/kata-ctl"
        install -Dm755 "src/tools/agent-ctl/target/${_target}/release/kata-agent-ctl" "$pkgdir/opt/kata/bin/kata-agent-ctl"
        install -Dm755 "src/tools/genpolicy/target/${_target}/release/genpolicy" "$pkgdir/opt/kata/bin/genpolicy"
        install -Dm755 "src/tools/trace-forwarder/target/${_target}/release/kata-trace-forwarder" "$pkgdir/opt/kata/bin/kata-trace-forwarder"

        # Install kata-manager shell script
        install -Dm755 "utils/kata-manager.sh" "$pkgdir/opt/kata/bin/kata-manager"

        # Install tool config files
        install -Dm644 "src/tools/genpolicy/rules.rego" \
            "$pkgdir/opt/kata/share/defaults/kata-containers/rules.rego"
        install -Dm644 "src/tools/genpolicy/genpolicy-settings.json" \
            "$pkgdir/opt/kata/share/defaults/kata-containers/genpolicy-settings.json"
    fi

    # Install profile.d script for PATH
    install -Dm644 "$srcdir/kata-containers.sh" "$pkgdir/etc/profile.d/kata-containers.sh"
}
```

**Notes:**
- On x86_64, both tarballs extract to `$srcdir/opt/kata/` and merge naturally — the `cp` in `package()` picks up everything.
- On aarch64, only `kata-static` extracts to `$srcdir/opt/kata/`. Tools are built from source and installed individually.
- `LIBC=gnu` on aarch64 avoids needing musl/rustup — uses the standard Arch `rust` package with `aarch64-unknown-linux-gnu` target.
- The vendor tarball overlay structure may need adjustment (`--strip-components`) — verify during first build.
- `csi-kata-directvolume` (Go, CSI plugin) is skipped on aarch64 since the vendor tarball doesn't cover Go deps and it's a niche Kubernetes component.

**Step 2: Commit**

```bash
git add PKGBUILD
git commit -m "feat: add PKGBUILD for kata-all-bin"
```

---

### Task 4: Create the GitHub Actions workflow

**Files:**
- Create: `.github/workflows/update.yml`

The workflow runs daily, checks for a new Kata Containers release, and if one is found, updates the PKGBUILD and pushes to the AUR.

**Step 1: Write the workflow**

```yaml
name: Update AUR Package

on:
  schedule:
    - cron: '0 6 * * *'
  workflow_dispatch:

jobs:
  check-and-update:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Check for new release
        id: check
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          LATEST=$(gh release view --repo kata-containers/kata-containers --json tagName -q .tagName)
          CURRENT=$(grep '^pkgver=' PKGBUILD | cut -d= -f2)
          echo "latest=$LATEST" >> "$GITHUB_OUTPUT"
          echo "current=$CURRENT" >> "$GITHUB_OUTPUT"
          if [ "$LATEST" != "$CURRENT" ]; then
            echo "update=true" >> "$GITHUB_OUTPUT"
            echo "::notice::New release found: $LATEST (current: $CURRENT)"
          else
            echo "update=false" >> "$GITHUB_OUTPUT"
            echo "::notice::Already up to date: $CURRENT"
          fi

      - name: Update PKGBUILD version
        if: steps.check.outputs.update == 'true'
        run: |
          VERSION="${{ steps.check.outputs.latest }}"
          sed -i "s/^pkgver=.*/pkgver=${VERSION}/" PKGBUILD
          sed -i "s/^pkgrel=.*/pkgrel=1/" PKGBUILD

      - name: Update checksums and generate .SRCINFO
        if: steps.check.outputs.update == 'true'
        uses: heyhusen/archlinux-package-action@v3
        with:
          path: .
          updpkgsums: true
          srcinfo: true
          namcap: false

      - name: Commit updated PKGBUILD to GitHub
        if: steps.check.outputs.update == 'true'
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add PKGBUILD .SRCINFO
          git commit -m "chore: update to ${{ steps.check.outputs.latest }}"
          git push

      - name: Deploy to AUR
        if: steps.check.outputs.update == 'true'
        uses: KSXGitHub/github-actions-deploy-aur@v4.1.1
        with:
          pkgname: kata-all-bin
          pkgbuild: ./PKGBUILD
          commit_username: ${{ secrets.AUR_USERNAME }}
          commit_email: ${{ secrets.AUR_EMAIL }}
          ssh_private_key: ${{ secrets.AUR_SSH_PRIVATE_KEY }}
          commit_message: "Update to ${{ steps.check.outputs.latest }}"
          ssh_keyscan_types: rsa,ecdsa,ed25519
          allow_empty_commits: false
```

**Notes:**
- The workflow also commits the updated PKGBUILD/.SRCINFO back to GitHub so the repo stays in sync.
- `updpkgsums` downloads all source files (~2 GB across both archs) to compute checksums. This is slow but runs at most once per upstream release.
- The `GH_TOKEN` for release checking uses the automatic `github.token` — no extra setup needed.
- The workflow needs write permission to push back to the repo. Ensure "Read and write permissions" is enabled in repo Settings > Actions > General > Workflow permissions.

**Step 2: Commit**

```bash
git add .github/workflows/update.yml
git commit -m "feat: add GitHub Actions workflow for automatic AUR updates"
```

---

### Task 5: Generate .SRCINFO and validate

**Step 1: Generate .SRCINFO**

On an Arch Linux system:
```bash
makepkg --printsrcinfo > .SRCINFO
```

Or via Docker on any system:
```bash
docker run --rm -v "$PWD":/pkg -w /pkg archlinux:latest bash -c \
  "pacman -Sy --noconfirm pacman-contrib && useradd -m builder && su builder -c 'makepkg --printsrcinfo' > .SRCINFO"
```

**Step 2: Validate with namcap (optional, on Arch)**

```bash
namcap PKGBUILD
```

**Step 3: Commit**

```bash
git add .SRCINFO
git commit -m "chore: add .SRCINFO"
```

---

### Task 6: Manual setup (not automated)

These are one-time manual steps the maintainer must do:

1. **Create the AUR package:**
   ```bash
   ssh aur@aur.archlinux.org setup-repo kata-all-bin
   ```

2. **Generate an SSH key for CI:**
   ```bash
   ssh-keygen -t ed25519 -C "github-actions-aur" -f aur_deploy -N ""
   ```

3. **Add the public key to your AUR account:**
   - Go to https://aur.archlinux.org/account/YOUR_USER/edit
   - Paste contents of `aur_deploy.pub` into SSH Public Keys

4. **Add GitHub repo secrets:**
   - `AUR_SSH_PRIVATE_KEY`: contents of `aur_deploy` (private key)
   - `AUR_USERNAME`: your AUR username
   - `AUR_EMAIL`: your AUR email

5. **Enable workflow write permissions:**
   - Repo Settings > Actions > General > Workflow permissions > Read and write permissions
