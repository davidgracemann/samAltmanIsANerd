## **I. Layer 0: The Silicon & Firmware Foundation**

Before the OS is initialized, the hardware must be decoupled from vendor-specific abstractions to expose the raw microarchitecture.

### **1. Microarchitecture Target**

For the **Intel Raptor Lake-S (i9-14900HX)**, we bypass generic x86-64 logic in favor of the **v4** ISA.

* **Target:** `x86-64-v4`
* **Instruction Sets enabled:** AVX-512, VNNI, and FMA3.
* **Impact:** Critical for deep learning inference and high-frequency quantitative simulations.

### **2. UEFI/Firmware Constraints**

| Feature | State | Rationale |
| --- | --- | --- |
| **Secure Boot** | **Disabled** | Allows loading of unsigned out-of-tree kernel modules. |
| **Intel VMD** | **Disabled** | Exposes NVMe to the Linux kernel via AHCI/NVMe protocol rather than a proprietary RAID layer. |
| **Fast Startup** | **Disabled** | Prevents Windows from leaving NTFS partitions in a "dirty" state, ensuring Btrfs/NTFS interoperability. |

---

## **II. Layer 1: Kernel & Filesystem Architecture**

The deployment utilizes **CachyOS** as the base, chosen for its "O3" optimized repositories and specific scheduler tuning for hybrid-core CPUs.

### **1. The BORE Scheduler (Burst-Oriented Response Enhancer)**

The i9-14900HX features an asymmetrical P-core/E-core layout. The **BORE scheduler** in the `linux-cachyos` kernel treats high-intensity bursts with priority, preventing "stutter" during heavy mathematical background tasks.

### **2. Recursive Btrfs Hierarchy**

We implement a subvolume strategy that separates system binaries from persistent research data to allow for **Atomic Snapshots**.

* **@ (Root):** System-level binaries.
* **@home:** Personal configs and `Gracemann` project files.
* **@var:** High-churn data (Logs/Pacman cache).
* **Mount Options:** `compress=zstd:1,noatime,discard=async`.
* *Note: `zstd:1` provides the best balance of CPU overhead vs. SSD lifespan.*



---

## **III. Layer 2: Hardware-Specific Hardening**

This phase resolves conflicts specific to the **Lenovo Legion Pro** and **NVIDIA Blackwell** subsystems.

### **1. Wireless Stack Deconfliction**

The `ideapad_laptop` module often triggers a "Hard Block" on the Wi-Fi 7 (RTW89) card.

* **Path:** `/etc/modprobe.d/lenovo-wifi.conf`
* **Configuration:** `blacklist ideapad_laptop`

### **2. The Blackwell (RTX 50-Series) Interface**

To utilize the Blackwell memory subsystem, the driver handshake must be verified at the hardware level.

* **Requirement:** CUDA 13.2+ / Driver 595.x+.
* **Verification:** `nvidia-smi` must show **PCIe Gen 5** speeds to confirm maximum throughput between the CPU and GPU.

---

## **IV. Layer 3: Memory & I/O Optimization**

To prevent disk "thrashing" and latency spikes, we move from physical swap to an in-memory compression model.

### **Memory Hierarchy Table**

| Type | Device | Algorithm | Speed |
| --- | --- | --- | --- |
| **Physical RAM** | 32GB DDR5 | N/A | ~5600 MT/s |
| **ZRAM (Swap)** | 31GB Virtual | `zstd` | ~50,000 MB/s |
| **Physical Swap** | **Disabled** | N/A | N/A |

---

## **V. Layer 4: The Dual-Boot Bridge (Gracemann Loader)**

The **alphaNode** must coexist with Windows without requiring manual BIOS intervention at every boot.

1. **Detection:** Install `os-prober`.
2. **GRUB Config:** Modify `/etc/default/grub` to set `GRUB_DISABLE_OS_PROBER=false`.
3. **Manual NVRAM Registration:** ```bash
efibootmgr --create --disk /dev/nvme0n1 --part 1 --label "Gracemann" --loader /EFI/cachyos/grubx64.efi
```


```



---

## **VI. Post-Deployment Verification Checklist**

| Category | Component | Validation Command |
| --- | --- | --- |
| **Compute** | x86-64-v4 / AVX-512 | `lscpu |
| **Graphics** | Blackwell / P0 State | `nvidia-smi -q -d PERFORMANCE` |
| **Display** | 240Hz / VRR | `wlr-randr` |
| **Network** | Wi-Fi 7 | `nmcli device show` |
| **IO** | ZRAM Efficiency | `zramctl` |

---

