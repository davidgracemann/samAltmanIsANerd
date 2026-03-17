# cachyAdmin
> windows → CachyOS · migration 101

> **alphaNode build guide** — Silicon-level tuning through dual-boot deployment
> A reproducible, layer-by-layer migration reference for the Lenovo Legion Pro 5 (i9-14900HX · RTX 5070 · Wi-Fi 7).
> Corrections from peer review applied throughout.

![Platform](https://img.shields.io/badge/Platform-CachyOS%20Linux-blue?style=flat-square)
![Arch](https://img.shields.io/badge/ISA%20Target-x86--64--v3-informational?style=flat-square)
![Kernel](https://img.shields.io/badge/Kernel-linux--cachyos%20%28BORE%29-green?style=flat-square)
![GPU](https://img.shields.io/badge/GPU-RTX%205070%20Mobile%20%28Blackwell%29-76b900?style=flat-square)
![FS](https://img.shields.io/badge/Filesystem-Btrfs%20%2B%20ZRAM-orange?style=flat-square)
![Boot](https://img.shields.io/badge/Boot-GRUB%20%2B%20efibootmgr-lightgrey?style=flat-square)

---

## Table of Contents

1. [Prerequisites & Hardware Reference](#0-prerequisites--hardware-reference)
2. [Layer 0 — Silicon & Firmware Foundation](#i-layer-0-silicon--firmware-foundation)
3. [Layer 1 — Kernel & Filesystem Architecture](#ii-layer-1-kernel--filesystem-architecture)
4. [Layer 2 — Hardware-Specific Hardening](#iii-layer-2-hardware-specific-hardening)
5. [Layer 3 — Memory & I/O Optimization](#iv-layer-3-memory--io-optimization)
6. [Layer 4 — Dual-Boot Bridge](#v-layer-4-the-dual-boot-bridge-gracemann-loader)
7. [Layer 5 — Post-Deployment Verification](#vi-layer-5-post-deployment-verification-checklist)
8. [Reference Links](#reference-links)

---

## 0. Prerequisites & Hardware Reference

This guide targets the following hardware exactly. ISA flags, driver versions, and UEFI settings are all specific to this platform.

| Component | Specification |
|:---|:---|
| **Device** | Lenovo Legion Pro 5 16IRX10 (83NN003CIN) |
| **CPU** | Intel Core i9-14900HX — **Raptor Lake-HX** (BGA, mobile) · 24C/32T |
| **GPU** | NVIDIA RTX 5070 Mobile — Blackwell architecture |
| **RAM** | 32 GB DDR5-5600 |
| **Storage** | 1 TB NVMe PCIe Gen 4 |
| **WiFi** | Realtek RTW89 (8922AE) — Wi-Fi 7 |
| **Target OS** | [CachyOS](https://cachyos.org) · x86-64-v3 repository |

> **Before you start:** Back up your Windows installation using [Macrium Reflect Free](https://www.macrium.com/reflectfree.aspx) or create a system image via `Settings → System → Recovery → Backup`. Resize your Windows partition in **Disk Management** before proceeding — shrink to leave ≥ 100 GB unallocated for Linux.

---

## I. Layer 0: Silicon & Firmware Foundation

Before the OS is initialized, the hardware must be decoupled from vendor-specific abstractions to expose the raw microarchitecture.

### 1. Microarchitecture Target

> **Correction notice:** Earlier versions of this document incorrectly stated `x86-64-v4` and AVX-512 as targets. The **i9-14900HX is Raptor Lake-HX**. Intel removed AVX-512 from the entire Raptor Lake family (both desktop `-S` and mobile `-HX` dies). The correct and maximum achievable ISA level for this CPU is **`x86-64-v3`**.

| Attribute | Value |
|:---|:---|
| **Correct ISA target** | `x86-64-v3` |
| **Instruction sets present** | AVX, AVX2, FMA3, BMI1/2, POPCNT, MOVBE, AVX-VNNI (non-512) |
| **AVX-512** | NOT present — removed by Intel in Raptor Lake |
| **x86-64-v4** | NOT achievable — requires AVX-512F/BW/CD/DQ/VL |

**Why this matters for CachyOS:** CachyOS offers separate package repositories for `x86-64-v3` and `x86-64-v4`. Installing from the v4 repo on this hardware will produce `Illegal instruction` crashes at runtime because v4 packages contain AVX-512 opcodes the CPU cannot execute. Confirm your repo tier during the CachyOS installer — select **v3**.

To verify post-install:

```bash
# Should show avx2 and fma, but NOT avx512f
lscpu | grep -E 'avx|fma|vnni'

# Confirm the installed repo tier
grep -r "x86-64" /etc/pacman.conf
```

**Impact on workloads:** For deep learning inference and quantitative simulation, `x86-64-v3` with AVX2 + FMA3 still provides a substantial SIMD advantage over scalar code. ONNX Runtime, PyTorch, and llama.cpp all have optimized AVX2 paths. The practical performance gap vs v4 is small for inference-heavy workloads where the GPU is the bottleneck.

---

### 2. UEFI/Firmware Configuration

Access the Legion BIOS with `F2` at POST, or via `Settings → System → Recovery → Advanced startup → UEFI Firmware Settings` in Windows.

| Feature | Target State | Rationale |
|:---|:---:|:---|
| **Secure Boot** | Disabled | Required for unsigned out-of-tree kernel modules (NVIDIA DKMS). Can be re-enabled after setting up [sbctl](https://github.com/Foxboron/sbctl) with custom keys. |
| **Intel VMD** | Disabled | Exposes the NVMe directly to the Linux kernel via the native NVMe driver rather than behind Intel's proprietary Virtual RAID layer. Without this, the NVMe is invisible to the Linux installer. |
| **Fast Startup** | Disabled | Prevents Windows from writing a hibernation state to the EFI partition, which leaves NTFS in a "dirty" state and blocks Linux from mounting shared partitions. |
| **CSM** | Disabled | Force UEFI-only boot. Required for `efibootmgr` to work correctly. |

> **VMD note:** Intel VMD (`/sys/bus/pci/drivers/vmd`) is the most common reason the CachyOS installer shows "no disks found." Disabling it in BIOS is the fix. If you need it re-enabled later for a Windows RAID setup, Linux has limited VMD support via `drivers/vmd` but it is not recommended for a daily-driver config.

---

## II. Layer 1: Kernel & Filesystem Architecture

CachyOS is chosen as the base distribution for its `O3`-optimized package repositories and its first-class support for performance-tuned kernels targeting hybrid-core desktop and mobile CPUs.

**CachyOS resources:**
- [CachyOS Download](https://cachyos.org/download/)
- [CachyOS Wiki](https://wiki.cachyos.org/)
- [CachyOS GitHub](https://github.com/CachyOS)
- [linux-cachyos kernel repo](https://github.com/CachyOS/linux-cachyos)

---

### 1. The BORE Scheduler

The i9-14900HX has an asymmetrical core layout: 8 P-cores (Raptor Cove, hyperthreaded) + 16 E-cores (Gracemont, single-threaded) = 32 logical threads. The Linux default CFS scheduler was designed for symmetric SMP and does not model burst workloads well on this topology.

**BORE (Burst-Oriented Response Enhancer)** is a CFS modification that assigns a `burst_score` to each task based on its recent CPU utilization pattern. High-intensity mathematical processes accumulate burst score and receive scheduling priority during peak demand, preventing stutter during concurrent heavy background tasks (e.g., an ollama inference job running behind an active agentic coding session).

The `linux-cachyos` kernel ships BORE-EEVDF (Earliest Eligible Virtual Deadline First with BORE) as the default scheduler. No configuration is needed — it is active on boot.

```bash
# Confirm BORE is active
cat /sys/kernel/debug/sched/features | grep -i bore

# Check current scheduler
cat /sys/block/nvme0n1/queue/scheduler
```

**Further reading:**
- [BORE scheduler source](https://github.com/firelzrd/bore-scheduler)
- [EEVDF paper (Lameter, 1997)](https://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.47.8197)

---

### 2. Btrfs Subvolume Hierarchy

We implement a flat subvolume layout that separates system binaries from user data to enable **atomic snapshots** via `snapper` or `btrfs send/receive`. This means you can roll back a broken system update without touching your `@home` data.

#### Subvolume map

| Subvolume | Mountpoint | Contents | Snapshotted |
|:---|:---|:---|:---:|
| `@` | `/` | System binaries, `/etc`, `/boot` | Yes |
| `@home` | `/home` | User configs, Gracemann project files | Yes |
| `@var` | `/var` | High-churn data: logs, pacman cache, databases | Optional |
| `@snapshots` | `/.snapshots` | Snapshot storage (snapper convention) | No |

#### Mount options

```
compress=zstd:1,noatime,discard=async,space_cache=v2
```

| Option | Effect |
|:---|:---|
| `compress=zstd:1` | Level 1 is the inflection point: fastest compression speed, meaningful ratio, lowest CPU overhead. Level 3+ gives diminishing ratio gains at significant CPU cost. |
| `noatime` | Disables inode access-time updates on every read. Eliminates a major source of unnecessary write amplification on NVMe. |
| `discard=async` | Issues TRIM commands asynchronously in the background rather than synchronously per-`unlink`. Prevents latency spikes during large delete operations. |
| `space_cache=v2` | Uses the v2 free space cache (B-tree based). More reliable and faster than v1 for large filesystems. |

#### `/etc/fstab` reference

```fstab
# Get your UUID with: blkid /dev/nvme0n1p2
UUID=<your-btrfs-uuid>  /          btrfs  subvol=@,compress=zstd:1,noatime,discard=async,space_cache=v2  0 0
UUID=<your-btrfs-uuid>  /home      btrfs  subvol=@home,compress=zstd:1,noatime,discard=async,space_cache=v2  0 0
UUID=<your-btrfs-uuid>  /var       btrfs  subvol=@var,compress=zstd:1,noatime,discard=async,space_cache=v2  0 0
UUID=<your-efi-uuid>    /boot/efi  vfat   umask=0077  0 2
```

**Btrfs resources:**
- [Btrfs Wiki — Getting Started](https://btrfs.wiki.kernel.org/index.php/Getting_started)
- [Btrfs mount options reference](https://btrfs.readthedocs.io/en/latest/ch-mount-options.html)
- [snapper — snapshot manager](https://github.com/openSUSE/snapper)

---

## III. Layer 2: Hardware-Specific Hardening

This layer resolves conflicts specific to the Lenovo Legion Pro platform and the NVIDIA Blackwell subsystem.

### 1. Wireless Stack Deconfliction

The `ideapad_laptop` kernel module manages power and function keys on Lenovo IdeaPad/Legion systems. A known conflict causes it to assert a **hard RF-kill** on the Realtek RTW89 Wi-Fi 7 adapter (8922AE), making the interface appear but rendering it permanently blocked — `rfkill list` shows `Hard blocked: yes` despite no physical switch being toggled.

**Fix:**

```bash
# Create the blacklist file
sudo tee /etc/modprobe.d/lenovo-wifi.conf << 'EOF'
blacklist ideapad_laptop
EOF

# Regenerate initramfs to apply at boot
sudo mkinitcpio -P

# Reboot, then verify
rfkill list
# Should show: Hard blocked: no  /  Soft blocked: no
```

> **Note:** Blacklisting `ideapad_laptop` disables the Fn-key backlight toggle and battery conservation mode on some Legion units. If you need those, the alternative is to use `ideapad_laptop` with `rfkill unblock wifi` in a `udev` rule, though the blacklist is the more stable solution.

**RTW89 driver resources:**
- [linux-firmware (rtw89)](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git)
- [RTW89 upstream driver](https://github.com/lwfinger/rtw89)
- [Arch Wiki — Lenovo Legion](https://wiki.archlinux.org/title/Lenovo_Legion)

---

### 2. NVIDIA Blackwell (RTX 50-Series) Driver Setup

The RTX 5070 Mobile is a Blackwell-architecture GPU (`GB205` die). Driver support requires the `nvidia-open` or `nvidia` proprietary driver from NVIDIA. CachyOS packages both.

```bash
# Install the open kernel module (recommended for Blackwell)
sudo pacman -S nvidia-open nvidia-utils lib32-nvidia-utils

# Or the proprietary closed module
sudo pacman -S nvidia nvidia-utils lib32-nvidia-utils
```

**Minimum requirements for Blackwell:**

| Component | Minimum Version | Notes |
|:---|:---|:---|
| **NVIDIA Driver** | 570.x+ | Blackwell support begins in 570 series |
| **CUDA Toolkit** | 12.4+ | Required for CUDA 12 compute capabilities |
| **Kernel module** | `nvidia-open` preferred | Blackwell has full open-source kernel module support |

```bash
# Verify driver installation and GPU state
nvidia-smi

# Full query including PCIe link speed (see note below)
nvidia-smi -q | grep -E 'PCIe|Gen|Link'
```

> **PCIe Gen 5 note:** The i9-14900HX PCIe controller is Gen 5 capable, but laptop OEMs frequently route the discrete GPU through a Gen 4 x8 or x16 lane for thermal and board-routing reasons. The actual negotiated speed depends on the Lenovo Legion Pro 5 board design. Run `nvidia-smi -q | grep -i pcie` to confirm. A Gen 4 result is normal and does not indicate a problem — GPU workloads are almost never PCIe-bandwidth-limited.

**NVIDIA resources:**
- [NVIDIA open-gpu-kernel-modules (GitHub)](https://github.com/NVIDIA/open-gpu-kernel-modules)
- [CachyOS NVIDIA setup wiki](https://wiki.cachyos.org/configuration/nvidia/)
- [Arch Wiki — NVIDIA](https://wiki.archlinux.org/title/NVIDIA)
- [nvidia-smi reference](https://developer.nvidia.com/nvidia-system-management-interface)

---

## IV. Layer 3: Memory & I/O Optimization

To prevent disk thrashing and latency spikes under heavy concurrent workloads (multiple model inference sessions, large dataset I/O), we replace physical swap with an in-memory compression layer.

### Memory Hierarchy

| Type | Device | Algorithm | Effective Speed | Notes |
|:---|:---|:---:|---:|:---|
| **Physical RAM** | 32 GB DDR5-5600 | — | ~89,600 MB/s | Theoretical peak |
| **ZRAM (virtual swap)** | 16 GB virtual | `zstd` | ~40,000–60,000 MB/s | Compresses to RAM; speed varies with data compressibility |
| **Physical swap** | Disabled | — | — | Eliminated to protect NVMe write endurance |

### ZRAM Setup

CachyOS ships `zram-generator`. Configure it:

```bash
# /etc/systemd/zram-generator.conf
sudo tee /etc/systemd/zram-generator.conf << 'EOF'
[zram0]
zram-size = ram / 2
compression-algorithm = zstd
swap-priority = 100
EOF

# Apply
sudo systemctl daemon-reload
sudo systemctl start systemd-zram-setup@zram0.service

# Verify
zramctl
# Should show /dev/zram0 with zstd algorithm and ~16G disksize
```

### Disable physical swap (if present)

```bash
# Check for active swap
swapon --show

# Disable a swap partition permanently
sudo swapoff /dev/nvme0n1pX
# Remove its entry from /etc/fstab
```

### vm.swappiness tuning

```bash
# /etc/sysctl.d/99-zram.conf
sudo tee /etc/sysctl.d/99-zram.conf << 'EOF'
vm.swappiness = 180
vm.watermark_boost_factor = 0
vm.watermark_scale_factor = 125
vm.page-cluster = 0
EOF

sudo sysctl --system
```

> `vm.swappiness = 180` is the correct value for ZRAM-only systems (counterintuitively high — ZRAM is fast enough that aggressive swapping to it is beneficial; it reduces memory pressure without the latency penalty of disk swap).

**References:**
- [zram-generator (systemd)](https://github.com/systemd/zram-generator)
- [Fedora ZRAM tuning rationale](https://fedoraproject.org/wiki/Changes/Scale_ZRAM_to_matched_RAM)
- [Linux vm.swappiness docs](https://www.kernel.org/doc/html/latest/admin-guide/sysctl/vm.html)

---

## V. Layer 4: The Dual-Boot Bridge (Gracemann Loader)

**Goal:** Boot CachyOS or Windows without entering the BIOS at every restart. GRUB acts as the bootloader, with a custom NVRAM entry ensuring the Legion boots to GRUB by default.

### Step-by-step

**1. Install os-prober**

```bash
sudo pacman -S os-prober
```

**2. Enable os-prober in GRUB config**

```bash
sudo tee -a /etc/default/grub << 'EOF'
GRUB_DISABLE_OS_PROBER=false
EOF
```

**3. Mount the Windows EFI partition and regenerate GRUB**

```bash
# Identify your EFI partition (usually /dev/nvme0n1p1)
lsblk -o NAME,FSTYPE,LABEL,SIZE,MOUNTPOINT

# Mount if not already mounted
sudo mount /dev/nvme0n1p1 /boot/efi

# Regenerate GRUB config (os-prober will find Windows Boot Manager)
sudo grub-mkconfig -o /boot/grub/grub.cfg

# Confirm Windows entry was detected
grep -i windows /boot/grub/grub.cfg
```

**4. Register the custom NVRAM boot entry**

```bash
# Create the "Gracemann" boot entry pointing to the CachyOS GRUB EFI binary
sudo efibootmgr \
  --create \
  --disk /dev/nvme0n1 \
  --part 1 \
  --label "Gracemann" \
  --loader /EFI/cachyos/grubx64.efi

# Verify the entry was created and set as first in boot order
efibootmgr -v
```

**5. Set boot order (optional)**

```bash
# List current order
efibootmgr

# Set Gracemann (e.g., Boot0003) as first
sudo efibootmgr --bootorder 0003,0000,0001
```

> **Windows Fast Startup:** If Windows boots after a shutdown and the NTFS partition shows as dirty in Linux, confirm Fast Startup is disabled: `Control Panel → Power Options → Choose what the power buttons do → Turn on fast startup` — **uncheck it**. Hibernation (`shutdown /h`) must also be disabled: run `powercfg /h off` in an elevated Windows terminal.

**Resources:**
- [efibootmgr man page](https://linux.die.net/man/8/efibootmgr)
- [Arch Wiki — GRUB](https://wiki.archlinux.org/title/GRUB)
- [Arch Wiki — Dual boot with Windows](https://wiki.archlinux.org/title/Dual_boot_with_Windows)
- [GRUB upstream docs](https://www.gnu.org/software/grub/manual/grub/)

---

## VI. Layer 5: Post-Deployment Verification Checklist

Run these after first boot into CachyOS to confirm each layer is functioning correctly.

| Layer | Component | Validation Command | Expected Output |
|:---|:---|:---|:---|
| **0** | ISA level (v3, not v4) | `lscpu \| grep -E 'avx2\|fma'` | `avx2`, `fma` present; no `avx512f` |
| **0** | CachyOS repo tier | `grep x86-64 /etc/pacman.conf` | `x86-64-v3` |
| **1** | BORE scheduler | `cat /sys/kernel/debug/sched/features` | `BORE` in output |
| **1** | Btrfs subvolumes | `btrfs subvolume list /` | `@`, `@home`, `@var` listed |
| **1** | Mount options | `mount \| grep btrfs` | `zstd`, `noatime`, `discard=async` |
| **2** | Wi-Fi hard block cleared | `rfkill list` | `Hard blocked: no` |
| **2** | NVIDIA driver loaded | `nvidia-smi` | GPU listed, driver version shown |
| **2** | PCIe link speed (check, don't assume) | `nvidia-smi -q \| grep -i pcie` | Gen 4 or Gen 5 (board-dependent) |
| **3** | ZRAM active | `zramctl` | `/dev/zram0` with `zstd`, ~16 GB |
| **3** | Physical swap disabled | `swapon --show` | Empty (no output) |
| **4** | Dual-boot entry | `efibootmgr -v` | `Gracemann` entry present |
| **4** | GRUB detects Windows | `grep -i windows /boot/grub/grub.cfg` | `Windows Boot Manager` entry |
| **System** | Kernel version | `uname -r` | `*-cachyos` suffix |
| **System** | Display / refresh rate | `wlr-randr` (Wayland) | `2560x1600 @ 240.000 Hz` |
| **System** | Network interface | `nmcli device show wlan0` | `GENERAL.STATE: 100 (connected)` |

---

## Reference Links

### CachyOS

| Resource | URL |
|:---|:---|
| Official site | https://cachyos.org |
| Download | https://cachyos.org/download/ |
| Wiki | https://wiki.cachyos.org/ |
| GitHub org | https://github.com/CachyOS |
| BORE scheduler | https://github.com/firelzrd/bore-scheduler |
| Forum | https://discuss.cachyos.org/ |

### Arch Linux (upstream reference)

| Resource | URL |
|:---|:---|
| Arch Wiki — Installation guide | https://wiki.archlinux.org/title/Installation_guide |
| Arch Wiki — Btrfs | https://wiki.archlinux.org/title/Btrfs |
| Arch Wiki — GRUB | https://wiki.archlinux.org/title/GRUB |
| Arch Wiki — NVIDIA | https://wiki.archlinux.org/title/NVIDIA |
| Arch Wiki — Lenovo Legion | https://wiki.archlinux.org/title/Lenovo_Legion |
| Arch Wiki — Dual boot | https://wiki.archlinux.org/title/Dual_boot_with_Windows |
| Arch Wiki — ZRAM | https://wiki.archlinux.org/title/Zram |

### Hardware & Drivers

| Resource | URL |
|:---|:---|
| NVIDIA open kernel modules | https://github.com/NVIDIA/open-gpu-kernel-modules |
| RTW89 driver | https://github.com/lwfinger/rtw89 |
| linux-firmware | https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git |
| sbctl (Secure Boot management) | https://github.com/Foxboron/sbctl |
| efibootmgr docs | https://linux.die.net/man/8/efibootmgr |

### Kernel & Filesystem

| Resource | URL |
|:---|:---|
| Btrfs documentation | https://btrfs.readthedocs.io/ |
| zram-generator | https://github.com/systemd/zram-generator |
| Linux sysctl vm docs | https://www.kernel.org/doc/html/latest/admin-guide/sysctl/vm.html |
| x86-64 ISA levels spec | https://gitlab.com/x86-64-abi |

### Windows Side

| Resource | URL |
|:---|:---|
| Macrium Reflect Free (backup) | https://www.macrium.com/reflectfree.aspx |
| Disable Fast Startup (Microsoft) | https://docs.microsoft.com/en-us/windows-hardware/design/device-experiences/oem-uefi |

---

## Corrections Log

| Version | Change |
|:---|:---|
| v1.1 | **Critical:** Corrected ISA target from `x86-64-v4` to `x86-64-v3`. Raptor Lake (all variants) does not support AVX-512. |
| v1.1 | **Minor:** Corrected CPU die name from "Raptor Lake-S" to "Raptor Lake-HX". `-S` is the desktop LGA1700 die; `-HX` is the BGA mobile die. |
| v1.1 | **Clarification:** PCIe Gen 5 GPU link noted as board-dependent, not guaranteed. Added verification command. |
| v1.1 | Added `space_cache=v2` to Btrfs mount options. |
| v1.1 | Added `vm.swappiness = 180` rationale for ZRAM-only systems. |

---

*Maintainer: David Grace (@davidgracemann) · Hardware: Gracemann365 · Last updated: March 2026*
