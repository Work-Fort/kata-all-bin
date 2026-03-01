# kata-all-bin

AUR package that installs all [Kata Containers](https://katacontainers.io/) binaries from upstream static releases.

## What's included

Everything from the `kata-static` and `kata-tools-static` release tarballs, installed to `/opt/kata/`:

- **Runtime:** `kata-runtime`, `containerd-shim-kata-v2` (Go + Rust shims)
- **Hypervisors:** `qemu-system-x86_64`, `cloud-hypervisor`, `firecracker`, `jailer`
- **Support daemons:** `virtiofsd`, `nydusd`
- **VM kernels:** standard, Dragonball, NVIDIA GPU variants
- **VM images:** Ubuntu Noble rootfs, Alpine initrd, confidential computing variants
- **Firmware:** OVMF (standard, Intel TDX, AMD SEV)
- **Tools:** `kata-ctl`, `genpolicy`, `kata-agent-ctl`, `kata-trace-forwarder`, `kata-manager`
- **Configuration:** TOML configs for all hypervisor backends

## Install

```bash
yay -S kata-all-bin
```

Or manually:

```bash
git clone https://aur.archlinux.org/kata-all-bin.git
cd kata-all-bin
makepkg -si
```

## Architecture support

| Arch | Runtime | Tools |
|------|---------|-------|
| x86_64 | Prebuilt | Prebuilt |
| aarch64 | Prebuilt | Built from source |

On aarch64, the tools are compiled from source since upstream doesn't ship a prebuilt tools tarball for ARM.

## Configuration

After install, `/opt/kata/bin` is added to your `PATH` via `/etc/profile.d/kata-containers.sh`. You may need to log out and back in, or run:

```bash
source /etc/profile.d/kata-containers.sh
```

To use Kata with containerd, add to `/etc/containerd/config.toml`:

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata]
  runtime_type = "io.containerd.kata.v2"
```

## Automatic updates

A GitHub Actions workflow checks daily for new Kata Containers releases and pushes updates to the AUR automatically.

## License

The packaging files in this repository are provided under [Apache-2.0](https://www.apache.org/licenses/LICENSE-2.0), matching the upstream Kata Containers license.
