# JETSON NANO P3450

**Development Reference & Planning Guide**

Serial: 1420321005520 | Model: P3450 | Module: P3448 | Last updated: February 2026

---

## 1. Hardware Identity & Specs

| Parameter | Value |
|-----------|-------|
| Board Model | P3450 (Carrier Board) |
| Compute Module | P3448-0000 (SD card) or P3448-0002 (eMMC) |
| Serial Number | 1420321005520 |
| SoC | Tegra X1 (T210) — 4x ARM Cortex-A57 @ 1.43 GHz |
| GPU | 128-core Maxwell |
| RAM | 4 GB LPDDR4 (shared CPU/GPU) |
| Storage | microSD slot (SD variant) / 16 GB eMMC (eMMC variant) |
| USB | 4x USB 3.0 (Type-A), 1x Micro-USB (device/power) |
| Camera | 2x MIPI CSI-2 (via J13 connector) |
| Network | Gigabit Ethernet |
| Power | 5V via Micro-USB (2A) or barrel jack (4A with J48 jumper) |

---

## 2. Operating System & Software Ceiling

> ⚠️ **This board is end-of-life. JetPack 4.6.6 / L4T R32.7.6 is the final supported release.**

| Component | Pinned Version | Notes |
|-----------|---------------|-------|
| Base OS | Ubuntu 18.04 LTS (Bionic) | Cannot upgrade to 20.04 — breaks GPU stack |
| Kernel | 4.9.337-tegra (NVIDIA custom) | Cannot replace with mainline — no GPU support |
| L4T (Linux for Tegra) | R32.7.6 | Final release for Nano |
| JetPack | 4.6.6 | Final release for Nano |
| CUDA | 10.2.300 | Pinned to L4T R32.x — no upgrade path |
| cuDNN | 8.2.1 | Matches CUDA 10.2 |
| TensorRT | 8.2.1 | Matches CUDA 10.2 |
| OpenCV | 4.1.1 (system) | Can build newer from source |
| Python | 3.6 (system) | Use Nix or containers for newer |
| GCC | 7.5 (system) | Use Nix or containers for newer |

---

## 3. Hard Constraints — Do NOT Do This

> ⚠️ **Breaking any of these will likely brick GPU/CUDA support or render the board unbootable.**

- ⚠️ **`do-release-upgrade`** — never upgrade Ubuntu 18.04 to 20.04+
- ⚠️ **Replace or upgrade the kernel** beyond L4T R32.7.6 — GPU driver is a custom kernel module
- ⚠️ **Install upstream mesa/libdrm/X11 packages** — these conflict with NVIDIA's userspace drivers
- ⚠️ **Add PPAs that pull in newer glibc, libstdc++, or kernel headers** not matching 4.9-tegra
- ⚠️ **Uninstall `nvidia-l4t-*` packages** — these ARE the GPU driver stack
- ⚠️ **Use pip/apt to globally upgrade numpy/scipy** beyond what CUDA 10.2 wheels support

---

## 4. Safe Upgrade Path After Flashing

After flashing the JetPack 4.6.1 SD card image, this is the safe sequence to reach 4.6.6:

```bash
sudo apt update
sudo apt dist-upgrade          # Upgrades within L4T R32.x track only
head -n 1 /etc/nv_tegra_release  # Verify: R32 REVISION: 7.6
nvcc --version                  # Verify: CUDA 10.2
```

✓ **This is safe** — stays within the same Ubuntu 18.04 base and NVIDIA BSP

---

## 5. Strategies for Newer Software

### 5a. Nix Package Manager on L4T Ubuntu (Recommended)

Install the Nix package manager on top of Ubuntu 18.04 L4T. This gives you nix-shell, flakes, and access to modern packages without touching the system base.

- [x] aarch64-linux is officially supported by Nix — binary cache available
- [x] nix-shell gives isolated environments with Python 3.11+, GCC 13, Node 20, etc.
- [x] Does not interfere with system packages, CUDA, or GPU drivers
- [x] Reproducible dev shells via shell.nix or flake.nix

Install Nix (multi-user, on the Jetson):

```bash
curl -L https://nixos.org/nix/install | sh -s -- --daemon
```

Example `shell.nix` for camera/bot development:

```nix
{ pkgs ? import <nixpkgs> {} }:
pkgs.mkShell {
  buildInputs = with pkgs; [
    python311  python311Packages.pip
    python311Packages.pillow  python311Packages.requests
    ffmpeg  git  curl  jq
  ];
}
```

> ⚠️ **CUDA/cuDNN/TensorRT must still come from the system (apt), not from Nix.** Nix-managed CUDA packages won't match the Tegra GPU driver. Use `LD_LIBRARY_PATH` or wrapper scripts to bridge Nix shells with system CUDA libs.

### 5b. Full NixOS on Jetson Nano?

**Status: Not practically viable.** Here's why:

- The **jetpack-nixos** project (by Anduril) explicitly does NOT support Jetson Nano — it targets JetPack 5+ devices only (Orin, Xavier)
- There is a **jetson-nano-nix** repo (mirrexagon) but it's essentially a collection of links with no working config
- The Nano requires NVIDIA's custom kernel 4.9-tegra. Mainline kernels lack Tegra X1 GPU support
- Even if you built a NixOS image with the NVIDIA kernel, the proprietary userspace blobs (`libdrm_nvidia`, `libnvgrd`, etc.) would need manual packaging and are tightly coupled to the L4T rootfs structure

**Verdict:** Use Nix-the-package-manager on L4T Ubuntu. Don't try to replace the OS with NixOS on this board. Save NixOS for a future Orin-based board if you want full Nixification.

### 5c. NVIDIA L4T Containers

NVIDIA publishes L4T-base Docker containers on NGC. These let you run newer stacks (Ubuntu 20.04 userspace, newer TensorRT) inside containers while keeping the host OS intact.

- [x] Container runtime (`nvidia-docker2`) comes with JetPack
- [x] GPU passthrough works via `--runtime=nvidia`
- ⚠️ **4 GB RAM is tight for containers — keep images minimal**

---

## 6. Camera + Bot Project Planning Notes

### 6a. Camera Options

| Interface | Examples | Notes |
|-----------|----------|-------|
| MIPI CSI-2 | Raspberry Pi Camera v2 (IMX219), IMX477 HQ Camera | Best performance, direct ISP access, lowest latency. Supported via Jetson-IO config tool |
| USB (UVC) | Logitech C920/C930e, any USB webcam | Easiest to set up, broader selection, higher latency |
| IP/RTSP | Reolink, Hikvision, Amcrest PoE cameras | Network-based, can handle many cameras, decoded on CPU/GPU via GStreamer or ffmpeg |

### 6b. Streaming Pipeline

GStreamer with NVIDIA hardware acceleration (`nvarguscamerasrc` for CSI, `nvv4l2decoder` for RTSP) is the native approach on L4T:

```bash
# CSI camera live preview
gst-launch-1.0 nvarguscamerasrc ! nvoverlaysink

# RTSP stream capture to file
gst-launch-1.0 rtspsrc location=rtsp://cam/stream ! \
  rtph264depay ! h264parse ! nvv4l2decoder ! nvvidconv ! \
  jpegenc ! multifilesink location=frame_%05d.jpg
```

**DeepStream SDK** (included in JetPack) can handle multi-camera pipelines with AI inference if needed later.

### 6c. Telegram / Signal Bot

Both can run in a Nix shell with modern Python:

| Platform | Library | Notes |
|----------|---------|-------|
| Telegram | `python-telegram-bot` | Mature, async, easy image/video sending. Bot API has no E2E encryption |
| Signal | `signal-cli` + `pysignalclirestapi` | Requires signal-cli daemon (Java-based, RAM-heavy). E2E encrypted |

**Recommendation:** Start with Telegram — it's lighter on resources, easier to set up, and the Bot API supports sending images, videos, and documents natively. Signal-cli's Java runtime is heavy for 4 GB RAM.

---

## 7. Resource Budget (4 GB RAM)

> ⚠️ **4 GB shared between CPU and GPU. Plan memory carefully.**

| Component | Approx. RAM | Notes |
|-----------|-------------|-------|
| L4T Ubuntu desktop | ~800 MB | Reduce by using CLI-only (no GUI) |
| GStreamer + 1 camera | ~200–400 MB | Depends on resolution/codec |
| Python bot process | ~50–150 MB | Telegram bot is lightweight |
| CUDA/TensorRT inference | ~500 MB–1.5 GB | Only if doing AI/detection |
| Docker container | ~200–500 MB overhead | On top of container workload |
| signal-cli daemon | ~300–500 MB | Java is memory-hungry |

For a headless camera-bot setup without AI inference, budget roughly 1.0–1.5 GB used, leaving room for the GPU. Switch to 10W MAXN power mode for full performance, or use 5W mode for lower power.

---

## 8. Pre-Development Checklist

- [ ] Flash JetPack 4.6.1 SD card image
- [ ] Run `apt dist-upgrade` to reach L4T R32.7.6 (JetPack 4.6.6)
- [ ] Verify GPU: `nvcc --version`, `head -n 1 /etc/nv_tegra_release`
- [ ] Install Nix package manager (`curl -L https://nixos.org/nix/install | sh -s -- --daemon`)
- [ ] Set up nix-shell with Python 3.11+, ffmpeg, and bot libraries
- [ ] Decide camera type (CSI vs USB vs RTSP) and acquire hardware
- [ ] Set up headless mode if not using a display (saves ~500 MB RAM)
- [ ] Configure power mode: `sudo nvpmodel -m 0` (MAXN) or `-m 1` (5W)
- [ ] Enable fan control: `sudo jetson_clocks`
- [ ] Set up SSH access for remote management

---

## 9. Future Upgrade Path

When this board's limitations become blockers, the natural successor is:

| | Jetson Nano (current) | Jetson Orin Nano (upgrade path) |
|---|---|---|
| JetPack | 4.6.6 (EOL) | 6.x (active support) |
| CUDA | 10.2 | 12.x |
| Ubuntu | 18.04 | 22.04 / 24.04 |
| RAM | 4 GB | 4 / 8 GB |
| AI Performance | 472 GFLOPS | Up to 67 TOPS (INT8) |
| NixOS Support | Not viable | Supported via jetpack-nixos (Anduril) |
| Price | ~$99 (discontinued) | ~$249 |

The Orin Nano would let you fully **nixify** the board using the Anduril jetpack-nixos project, run modern CUDA workloads, and have significantly more headroom for multi-camera + AI + bot setups.

---

*— End of Reference Document —*
