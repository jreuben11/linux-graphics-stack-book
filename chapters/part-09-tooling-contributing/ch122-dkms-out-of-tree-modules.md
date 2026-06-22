# Chapter 122: DKMS and Out-of-Tree GPU Kernel Modules

**Target audiences**: System administrators deploying GPU drivers; kernel and driver developers maintaining out-of-tree or downstream kernel trees; Linux distribution packagers managing GPU driver packages.

GPU kernel modules sit at the most turbulent intersection of hardware and software: they must be recompiled for every kernel the system runs, yet the hardware they drive is proprietary, complex, and changes slowly compared to kernel internals. This chapter explains why that problem exists, how DKMS solves it operationally, and how different GPU vendors — NVIDIA, AMD, and Intel — have each chosen a different point on the spectrum between fully proprietary and fully upstream. It covers the kernel module ABI problem, DKMS mechanics and configuration, NVIDIA's two module flavours (proprietary and open), AMD's upstream-first philosophy, Intel's firmware loading model, out-of-tree driver lifecycle, module signing under Secure Boot, and distribution packaging strategies.

---

## Table of Contents

1. [Introduction: The Recompilation Problem](#1-introduction-the-recompilation-problem)
2. [The Kernel Module ABI Problem](#2-the-kernel-module-abi-problem)
3. [DKMS: Dynamic Kernel Module Support](#3-dkms-dynamic-kernel-module-support)
4. [NVIDIA Proprietary Kernel Modules](#4-nvidia-proprietary-kernel-modules)
5. [NVIDIA Open Kernel Modules](#5-nvidia-open-kernel-modules)
6. [AMD's Upstream-First Approach](#6-amds-upstream-first-approach)
7. [Intel i915/Xe and Firmware Loading](#7-intel-i915xe-and-firmware-loading)
8. [Out-of-Tree Drivers: When Upstream is Delayed](#8-out-of-tree-drivers-when-upstream-is-delayed)
9. [Module Signing and Secure Boot](#9-module-signing-and-secure-boot)
10. [Packaging GPU Drivers for Distributions](#10-packaging-gpu-drivers-for-distributions)
11. [Integrations](#11-integrations)

---

## 1. Introduction: The Recompilation Problem

A Linux kernel module (`.ko` file) is not a standalone executable in the ordinary sense. It is a relocatable object that is linked into the running kernel image at load time. The kernel's internal structures — scheduler run-queues, DRM device objects, PCI descriptor tables — are not exposed through a stable ABI. When those structures change between kernel versions (and they do, often), any binary `.ko` compiled against the old layout will either refuse to load or corrupt memory if forced in.

For open-source drivers shipped inside the kernel tree, this is a non-problem: the driver is recompiled together with the kernel. For GPU vendors that ship kernel components separately — whether due to closed-source constraints, hardware embargoes, or simply a version mismatch between the driver release and the distro kernel — a mechanism is needed to recompile the module for every kernel the user might run. That mechanism is **DKMS** (Dynamic Kernel Module Support).

The landscape in 2026 spans several distinct strategies:

- **NVIDIA proprietary**: closed C source + binary stub, distributed via DKMS; requires rebuild on every kernel update.
- **NVIDIA open modules**: MIT/GPL dual-licensed source, distributed via DKMS or `.run` installer; supports Turing (RTX 20xx) and later only.
- **AMD amdgpu**: fully upstreamed; no DKMS needed for the open driver; `amdgpu-pro` uses DKMS for proprietary userspace shim extensions.
- **Intel i915/Xe**: fully upstreamed; no DKMS; but requires separate firmware blob packages from `linux-firmware`.

Understanding these strategies helps sysadmins choose the right packaging approach, and helps driver developers understand the tradeoffs before deciding whether to upstream their work.

---

## 2. The Kernel Module ABI Problem

### 2.1 No Stable Kernel ABI — By Design

Linux deliberately provides no stable in-kernel ABI. Linus Torvalds has stated this policy explicitly on numerous occasions; the rationale is that a frozen ABI calcifies design mistakes and prevents internal cleanups. Unlike the userspace ABI (where `POSIX` and `glibc` provide decades of compatibility), internal kernel interfaces change freely between any two kernel versions — including patch releases. A `.ko` built against kernel 6.12.1 may refuse to load on 6.12.2 if a single exported symbol changed prototype or if an embedded struct grew a field.

This is materially different from userspace shared libraries. `libc.so.6` maintains strict symbol versioning across decades. The kernel's internal interface — everything in `include/linux/` that is not explicitly published through `include/uapi/linux/` — is private and may change at will.

### 2.2 Version Magic and CRC Checksums

`modprobe` enforces compatibility through two mechanisms. The first is **version magic** (`vermagic`): a string embedded in every `.ko` encoding the exact kernel version string and key configuration flags (SMP, preemption model, etc.). If the magic string in the module does not match the running kernel's `UTS_RELEASE` string, `modprobe` rejects the module outright:

```
ERROR: could not insert 'nvidia': Exec format error
```

The second mechanism is **`CONFIG_MODVERSIONS`**, which adds a CRC checksum for every exported symbol. The CRC is computed over the full type signature of the exported function or variable. The `Module.symvers` file generated during a kernel build contains lines of the form:

```
0xdeadbeef    drm_dev_alloc    vmlinux    EXPORT_SYMBOL_GPL    (none)
```

When a module is loaded, the kernel compares the CRC stored in the module against the CRC in `vmlinux`. A mismatch triggers:

```
nvidia: disagrees about version of symbol drm_gem_object_put
```

This means that even a minor kernel point release that touched `struct drm_gem_object` will break a module compiled against the previous release, even if the change appears unrelated to GPU drivers. [Source](https://docs.kernel.org/kbuild/modules.html)

### 2.3 `EXPORT_SYMBOL` vs `EXPORT_SYMBOL_GPL`

The kernel distinguishes two export tiers. `EXPORT_SYMBOL(sym)` makes a symbol available to any loadable module. `EXPORT_SYMBOL_GPL(sym)` restricts access to modules that declare `MODULE_LICENSE("GPL")` or a compatible GPL variant. The check is enforced at build time by `modpost`: a non-GPL module that references a GPL-only symbol causes a hard build error.

Many GPU-relevant interfaces are GPL-only:

```c
/* From include/drm/drm_drv.h */
EXPORT_SYMBOL_GPL(drm_dev_alloc);
EXPORT_SYMBOL_GPL(drm_dev_register);
```

This prevents proprietary modules from using the modern DRM device registration path directly. NVIDIA's proprietary driver historically worked around this by declaring `MODULE_LICENSE("NVIDIA")` and routing around GPL-only symbols — accessing them through a GPL-licensed shim (`nvidia-drm.ko`) which bridges to the proprietary `nvidia.ko`. The ethical and legal debate around GPL symbol workarounds has run for two decades on LKML, with the kernel community maintaining studied ambiguity about whether all loadable modules constitute derived works. [Source](https://lwn.net/Articles/603131/)

### 2.4 `depmod` and Module Dependency Resolution

`depmod` reads all `.ko` files installed under `/lib/modules/$(uname -r)/` and produces the `modules.dep` database. It resolves inter-module symbol dependencies: if `nvidia-drm.ko` uses a symbol from `nvidia.ko`, `depmod` records the dependency so that `modprobe nvidia-drm` will first load `nvidia.ko`. The database lives at:

```
/lib/modules/6.12.1-amd64/modules.dep
/lib/modules/6.12.1-amd64/modules.dep.bin   # binary form
```

After installing or removing a module, running `depmod -A` (or the full `depmod -a`) updates these files. Package managers and DKMS invoke `depmod` automatically as part of their post-install hooks.

---

## 3. DKMS: Dynamic Kernel Module Support

DKMS was developed by Dell in 2003 to solve the exact problem described above. The core insight is simple: ship module *source* alongside a build recipe, and trigger a rebuild every time a new kernel is installed. DKMS handles the orchestration: storing source trees, invoking `make`, caching built modules, and installing them into the right location for each kernel. [Source](https://github.com/dkms-project/dkms)

### 3.1 Directory Layout

```
/usr/src/<module>-<version>/      ← module source tree (registered with DKMS)
    dkms.conf                     ← DKMS build configuration
    Makefile
    *.c / *.h

/var/lib/dkms/<module>/           ← DKMS state and built artefacts
    <version>/
        build/                    ← build directory per kernel
        <kernelver>/<arch>/       ← installed .ko files
        original_module/          ← displaced original module backups

/etc/dkms/framework.conf          ← global DKMS configuration
/etc/dkms/<module>.conf           ← per-module overrides
```

### 3.2 The `dkms.conf` File

`dkms.conf` is a shell script sourced by the `dkms` binary. Directives are uppercase variable assignments. A minimal example for a hypothetical GPU driver:

```bash
# /usr/src/mygpu-1.0.0/dkms.conf
PACKAGE_NAME="mygpu"
PACKAGE_VERSION="1.0.0"
AUTOINSTALL="yes"

BUILT_MODULE_NAME[0]="mygpu"
BUILT_MODULE_LOCATION[0]="src/"
DEST_MODULE_LOCATION[0]="/updates/dkms"

MAKE[0]="make -C ${kernel_source_dir} M=${dkms_tree}/${PACKAGE_NAME}/${PACKAGE_VERSION}/build"
CLEAN="make -C ${kernel_source_dir} M=${dkms_tree}/${PACKAGE_NAME}/${PACKAGE_VERSION}/build clean"
```

Key directives:

| Directive | Purpose |
|---|---|
| `PACKAGE_NAME` | Module name, used with `-m` flag |
| `PACKAGE_VERSION` | Module version, used with `-v` flag |
| `BUILT_MODULE_NAME[n]` | Compiled `.ko` name (without extension); supports array for multi-module packages |
| `BUILT_MODULE_LOCATION[n]` | Path within source tree where the `.ko` lives after build |
| `DEST_MODULE_LOCATION[n]` | Install destination under `/lib/modules/$(uname -r)/` |
| `MAKE[n]` | Build command; `${kernel_source_dir}` expands to the kernel build tree |
| `MAKE_MATCH[n]` | Regex against kernel version to select the right MAKE entry |
| `CLEAN` | Clean command (deprecated in newer DKMS; use `"true"` for compatibility) |
| `AUTOINSTALL` | Set to `"yes"` so DKMS rebuilds on every new kernel |
| `PRE_BUILD` | Script to run before compilation |
| `POST_BUILD` | Script to run after compilation (used for module signing, see §9) |
| `PRE_INSTALL` / `POST_INSTALL` | Scripts bracketing installation; non-zero `PRE_INSTALL` aborts |
| `BUILD_EXCLUSIVE_KERNEL` | Regex restricting which kernels the module will build for |
| `BUILD_EXCLUSIVE_ARCH` | Architecture filter (e.g., `"x86_64"`) |

The runtime variable `${kernelver}` expands to the target kernel version string (e.g., `6.12.1-amd64`), and `${kernel_source_dir}` expands to `/lib/modules/${kernelver}/build`.

### 3.3 Core DKMS Commands

**Register source with DKMS:**

```bash
# Source must already be in /usr/src/mygpu-1.0.0/
dkms add -m mygpu -v 1.0.0
```

Alternatively, if a `dkms.conf` is bundled in a tarball:

```bash
dkms ldtarball /path/to/mygpu-1.0.0-dkms.tar.gz
```

**Build for a specific kernel:**

```bash
dkms build -m mygpu -v 1.0.0 -k 6.12.1-amd64
# Build output lands in /var/lib/dkms/mygpu/1.0.0/build/
```

**Install into the module tree:**

```bash
dkms install -m mygpu -v 1.0.0 -k 6.12.1-amd64
# Module installed to /lib/modules/6.12.1-amd64/updates/dkms/mygpu.ko
# depmod is run automatically
```

**Check status of all registered DKMS modules:**

```bash
dkms status
# Output example:
mygpu/1.0.0, 6.12.1-amd64, x86_64: installed
nvidia/570.144, 6.12.1-amd64, x86_64: installed
```

**Rebuild all modules for all installed kernels (used in package post-install hooks):**

```bash
dkms autoinstall
```

**Remove a version:**

```bash
dkms remove -m mygpu -v 1.0.0 --all
```

### 3.4 Integration with Kernel Package Hooks

On Debian/Ubuntu, kernel packages (e.g., `linux-image-6.12.1-amd64`) include a post-install hook that calls `dkms autoinstall`. On Fedora/RHEL the integration is different: `akmods` (the Automatic Kernel Module system, used by RPMFusion) builds modules at first boot into a new kernel rather than at install time.

On Debian, a DKMS-aware package uses `dh-dkms` in its rules file:

```
# debian/rules
dh $@ --with dkms
```

The `dh_dkms` helper generates `postinst` and `prerm` scripts that call `dkms add`, `dkms build`, and `dkms install` during package installation, and `dkms remove` during removal. Module sources land in `/usr/src/${PACKAGE_NAME}-${PACKAGE_VERSION}/` by convention. [Source](https://wiki.debian.org/DkmsPackaging)

### 3.5 Framework Configuration

`/etc/dkms/framework.conf` controls global DKMS behaviour. Key options:

```bash
# /etc/dkms/framework.conf
sign_file="/lib/modules/${kernelver}/build/scripts/sign-file"
mok_signing_key="/var/lib/dkms/mok.key"
mok_certificate="/var/lib/dkms/mok.pub"
parallel_jobs=4
autoinstall_all_kernels="yes"
```

DKMS 3.x (the current series, latest stable 3.2.1 as of May 2025) auto-generates a self-signed 2048-bit RSA certificate for Secure Boot module signing when `generate_mok` is invoked. [Source](https://github.com/dkms-project/dkms)

---

## 4. NVIDIA Proprietary Kernel Modules

### 4.1 Architecture Overview

NVIDIA's GPU driver has always been split into userspace and kernel components. The userspace side (`libGL.so`, `libcuda.so`, `nvidia-smi`, `nvidia-settings`, `libEGL_nvidia.so`) is entirely proprietary but does not require recompilation for each kernel. The kernel side consists of four modules:

| Module | Role |
|---|---|
| `nvidia.ko` | Core GPU resource manager; hardware initialisation, memory management, context scheduling |
| `nvidia-drm.ko` | DRM primary node for KMS; bridges proprietary RM to the kernel DRM subsystem |
| `nvidia-modeset.ko` | Display engine state machine; mode-setting, HDCP, display port negotiation |
| `nvidia-uvm.ko` | Unified Virtual Memory; managed CUDA memory that migrates between CPU and GPU |

A fifth module, `nvidia-peermem.ko`, enables GPUDirect peer-to-peer memory access for InfiniBand RDMA.

### 4.2 Binary Stub Architecture

The proprietary driver is distributed as a self-extracting `.run` file. Inside it are two components for the kernel modules:

1. **Pre-compiled binary stubs** (`nv-kernel.o_binary`, `nv-modeset-kernel.o_binary`): architecture-specific binary blobs containing the core GPU Resource Manager logic. NVIDIA does not distribute source for these.
2. **C glue layer** (`kernel-open/` or `kernel/` directory): thin, kernel-version-sensitive wrappers that implement the Linux-specific interfaces (PCI probing, interrupt handling, memory mapping). This layer is compiled at install time against the running kernel headers.

The `./nvidia-installer` script (or DKMS configuration shipped by distributions) compiles the glue layer and links it with the pre-compiled stubs to produce the final `.ko` files. The resulting modules contain both open glue and closed binary, which is why `modinfo nvidia` shows a complex mix of symbols.

### 4.3 The `MODULE_LICENSE` Controversy

The proprietary `nvidia.ko` declares `MODULE_LICENSE("NVIDIA")`, which the kernel treats as proprietary (not GPL-compatible). This triggers `TAINT_PROPRIETARY_MODULE` — any kernel panic involving NVIDIA modules is marked as tainted, and bug reports to the kernel community are generally rejected for tainted kernels. More consequentially, a `MODULE_LICENSE("NVIDIA")` module cannot use `EXPORT_SYMBOL_GPL` symbols.

NVIDIA's solution: `nvidia-drm.ko` declares `MODULE_LICENSE("GPL")` and acts as the bridge between the proprietary `nvidia.ko` and the GPL-only DRM interfaces. This "GPL wrapper" pattern has generated significant debate on LKML, with many kernel developers arguing it constitutes a deliberate circumvention of copyleft requirements. [Source](https://lwn.net/Articles/154602/)

### 4.4 Distribution Packaging

**Ubuntu**: The `ubuntu-drivers-common` package provides:

```bash
ubuntu-drivers autoinstall   # detect GPU and install matching nvidia-driver-* meta-package
```

This installs packages such as `nvidia-driver-570` (userspace) and `nvidia-dkms-570` (DKMS module source). The DKMS source lands in `/usr/src/nvidia-570.*/`.

**Fedora (RPMFusion)**: Two packaging strategies:

```bash
# akmod: build-on-first-boot, preferred
dnf install akmod-nvidia

# kmod: pre-built for current kernel only
dnf install kmod-nvidia
```

The `akmod` system builds the module asynchronously on first boot after a kernel update, using akmods infrastructure separate from DKMS. [Source](https://rpmfusion-developers.rpmfusion.narkive.com/2xZ1hzx6/akmod-vs-kmod)

### 4.5 Enabling KMS

NVIDIA's `nvidia-drm.ko` supports KMS (Kernel Mode Setting), but it is disabled by default in the proprietary driver to avoid conflicts with older configurations. Enable it via:

```bash
# /etc/modprobe.d/nvidia-drm.conf
options nvidia-drm modeset=1
```

Without `modeset=1`, Wayland compositors cannot use the NVIDIA DRM primary node for display, and GBM allocations fail. Current NVIDIA driver releases (570+) enable modeset by default, but older installations may still have it disabled.

### 4.6 Blacklisting Nouveau

The upstream `nouveau` driver and the proprietary NVIDIA driver cannot coexist — both attempt to claim the same PCI device. Distributions that install NVIDIA proprietary drivers must blacklist `nouveau`:

```bash
# /etc/modprobe.d/blacklist-nouveau.conf
blacklist nouveau
options nouveau modeset=0
```

After modifying this file, regenerate the initramfs so the blacklist takes effect at early boot:

```bash
# Debian/Ubuntu:
update-initramfs -u

# Fedora/RHEL:
dracut --force
```

---

## 5. NVIDIA Open Kernel Modules

### 5.1 History and Significance

In May 2022, NVIDIA open-sourced the Linux GPU kernel modules under a dual MIT/GPL license. This was the most significant event in NVIDIA's Linux driver history since the GPU vendor had previously refused all cooperation with the open-source community. The repository is at [https://github.com/NVIDIA/open-gpu-kernel-modules](https://github.com/NVIDIA/open-gpu-kernel-modules).

The open modules cover the same four components (`nvidia.ko`, `nvidia-drm.ko`, `nvidia-modeset.ko`, `nvidia-uvm.ko`) as the proprietary modules, but with full source code. However, the architecture retains a separation:

- **Kernel interface layer** (fully open C source): the Linux-specific wrapping code in `src/nvidia/arch/nvalloc/unix/`
- **OS-agnostic GPU RM** (pre-built binary stubs): `nv-kernel.o_binary` and `nv-modeset-kernel.o_binary`, the same binary objects used in the proprietary driver

The binary stubs mean the open modules are not purely open — the core GPU resource manager remains closed. What changed is that the glue layer that connects the kernel to those blobs is now readable, modifiable, and patchable by the community. This enables kernel developers to fix compatibility issues without waiting for NVIDIA, and it allowed Nouveau developers to extract firmware upload sequences for the GSP-RM. [Source](https://github.com/NVIDIA/open-gpu-kernel-modules/blob/main/README.md)

### 5.2 Hardware Support Requirements

The open kernel modules require **Turing architecture or later** (GeForce RTX 20xx / GTX 16xx series, Quadro RTX, T4, A100, H100, and all subsequent generations). The reason is architectural: the open modules offload the GPU resource manager to the **GSP-RM** (GPU System Processor Resource Manager) — a dedicated Falcon microcontroller on-die that runs a firmware image rather than executing in the CPU-side driver.

As of driver series 595 and later:

- **Blackwell (RTX 50xx, B200, GB200)**: open modules *only*; proprietary modules do not support Blackwell. [Source](https://download.nvidia.com/XFree86/Linux-x86_64/595.58.03/README/kernel_open.html)
- **Turing through Hopper**: both open and proprietary modules work; NVIDIA recommends open modules.
- **Volta and older**: proprietary modules only; `NVreg_OpenRmEnableUnsupportedGpus=1` kernel parameter forces open modules onto older unsupported hardware but is not recommended.

### 5.3 GSP-RM Architecture

The GPU System Processor is a small RISC-V (formerly Falcon) core present in all Turing+ NVIDIA GPUs. It runs the Resource Manager firmware (`gsp.bin`) and exposes an IPC channel to the CPU-side driver via shared memory. The CPU-side `nvidia.ko` (open or proprietary) communicates with the GSP-RM via this channel:

```
CPU side                       GPU die
─────────────────              ───────────────────────────
nvidia.ko (kernel module)      GSP core (RISC-V)
  │                              │
  ├── IPC ring buffer ──────────>│ RM commands (alloc ctx, etc.)
  │<─────────────────────────────│ completions / events
  │
  ├── nvidia-uvm.ko (UVM)
  ├── nvidia-drm.ko (KMS)
  └── nvidia-modeset.ko
```

The firmware blob `nvidia/gsp_tu10x.bin` (Turing), `nvidia/gsp_ga10x.bin` (Ampere), etc., are delivered as part of the NVIDIA driver release package (not by `linux-firmware`). These blobs are loaded by `nvidia.ko` at initialisation time. Without the matching GSP firmware, the open modules cannot initialise the GPU.

### 5.4 Building the Open Modules

```bash
git clone https://github.com/NVIDIA/open-gpu-kernel-modules.git
cd open-gpu-kernel-modules

# Build against the running kernel
make modules -j$(nproc)

# Install
make modules_install -j$(nproc)
depmod -a
```

The build requires the kernel headers/build tree matching the target kernel. The toolchain must match what was used to build the kernel (`gcc` or `clang` as appropriate). Cross-compilation to `aarch64` is supported via standard cross-compilation variables.

Alternatively, the `.run` installer selects open modules:

```bash
./NVIDIA-Linux-x86_64-610.43.02.run -M=open
```

### 5.5 Verifying the Installed Flavour

```bash
# Check license string — "Dual MIT/GPL" indicates open modules
modinfo nvidia | grep license
# license:        Dual MIT/GPL

# Check /proc for module flavour
cat /proc/driver/nvidia/version
# NVRM version: NVIDIA UNIX x86_64 Kernel Module  610.43.02 ... (Open Kernel Module)
```

### 5.6 Open-Module-Exclusive Features

The open modules unlock capabilities that the proprietary modules cannot offer due to ABI constraints:

- **NVIDIA Confidential Computing** (H100, Blackwell): hardware TEE support
- **Magnum IO GPUDirect Storage**: zero-copy DMA between storage and GPU memory
- **Heterogeneous Memory Management (HMM)**: CPU page fault–driven GPU memory migration, uses `EXPORT_SYMBOL_GPL` interfaces
- **DMABUF support for CUDA allocations**: enables sharing CUDA buffers with V4L2, `dma-heap`, and other kernel subsystems
- **CPU affinity for GPU fault handlers**

### 5.7 Distribution Adoption

By late 2025, the open modules had become the default in major distributions for supported hardware:

- **Arch Linux**: `nvidia-open` package (open modules) is the default as of the R590 driver series; `nvidia` (proprietary) remains available.
- **Fedora**: Open modules ship by default for Turing+ GPUs via RPMFusion as of Fedora 41.
- **Ubuntu**: `nvidia-driver-*-open` variants available; `ubuntu-drivers` recommends open modules for RTX 20xx and later.
- **NixOS**: `hardware.nvidia.open = true` in `configuration.nix` selects open modules; required for Blackwell GPUs.

```nix
# /etc/nixos/configuration.nix
hardware.nvidia = {
  open = true;           # use open kernel modules (required for Blackwell)
  modesetting.enable = true;
  package = config.boot.kernelPackages.nvidiaPackages.stable;
};
```

[Source](https://wiki.nixos.org/wiki/NVIDIA)

---

## 6. AMD's Upstream-First Approach

### 6.1 The Strategy

AMD took the opposite path from NVIDIA: all AMD GPU kernel driver code lives in `drivers/gpu/drm/amd/` in the mainline Linux kernel tree. There is no DKMS package for the primary AMD GPU driver. When a user installs a kernel from their distribution, `amdgpu.ko` comes with it, already compiled and installed. When a new GPU generation ships, AMD engineers submit patches to `amd-gfx@lists.freedesktop.org`, which are reviewed by DRM maintainers and merged into the `drm-misc-next` or `drm-next` tree before being pulled into Linus's tree. [Source](https://en.wikipedia.org/wiki/AMDGPU_(Linux_kernel_module))

The practical benefit is enormous. A user who buys a new AMD GPU and runs a sufficiently recent kernel gets working hardware without installing any external packages. There is no version matrix (driver X requires kernel Y), no DKMS rebuild failure cascade, no initramfs-not-updated footguns.

### 6.2 amdgpu-pro: The Proprietary Stack

AMD's `amdgpu-pro` package suite targets enterprise and professional users who need certified OpenGL, specific OpenCL versions, or AMD's proprietary Vulkan ICD for workstation use cases. This package *does* use DKMS for any kernel module extensions it ships, but these are narrow shims rather than a complete replacement for the upstream driver. The base `amdgpu.ko` comes from the kernel; `amdgpu-pro` layers proprietary userspace components (`libGL_amd.so`, `libOpenCL_amd.so`) on top. [Source](https://amdgpu-install.readthedocs.io/en/latest/install-installing.html)

The DKMS usage in `amdgpu-pro` is a minor footnote. The main lesson from AMD's approach is architectural: the decision to upstream everything eliminates the entire DKMS problem class for that driver.

### 6.3 The Mailing List Workflow

AMD's upstream contribution flow:

1. Engineer writes patch against the current `amdgpu` tree (often based on `drm-tip`).
2. Patch posted to `amd-gfx@lists.freedesktop.org` with `Cc: dri-devel@lists.freedesktop.org`.
3. Review by AMD kernel team and external DRM reviewers.
4. Picked up by DRM maintainer (Alex Deucher for `amdgpu`) into `drm-misc-next` or `amd-staging-drm-next`.
5. Pulled into mainline during the next merge window.

This means new GPU support for AMD hardware lands in the upstream kernel before or shortly after retail availability, rather than through a separate driver download.

### 6.4 ROCm and the Compute Stack

AMD's ROCm HPC/ML compute stack (`amdgpu-dkms` in older releases) previously shipped its own version of `amdgpu.ko` via DKMS to backport features not yet in the distro kernel. As ROCm matured and upstream AMD support improved, ROCm 6.x primarily uses the kernel's built-in `amdgpu.ko` and no longer requires a DKMS kernel module replacement for most configurations. Some HMM and peer memory features may still require kernel version requirements that push users toward newer kernels.

---

## 7. Intel i915/Xe and Firmware Loading

### 7.1 Upstream Drivers

Intel's GPU kernel drivers are also fully upstream. The `i915.ko` driver supports all Intel GPUs from Sandy Bridge through Tiger Lake and some later generations. The newer `xe.ko` driver, merged in Linux 6.8, targets Meteorite Lake and later, providing a cleaner architecture designed from scratch (removing many legacy workarounds accumulated in `i915`). [Source](https://wiki.archlinux.org/title/Intel_graphics)

Neither driver requires DKMS; both are compiled into every mainstream Linux kernel.

### 7.2 The Firmware Challenge

The challenge for Intel GPU drivers is not source code — it is firmware. Modern Intel GPUs integrate dedicated microcontrollers:

| Firmware | Controller | Role |
|---|---|---|
| GuC (Graphics Microcontroller) | Dedicated Cortex-M (varies by generation) | GPU scheduling, context submission, power management |
| HuC (HEVC/AVC Microcontroller) | Similar core | Hardware video codec acceleration |
| DMC (Display Microcontroller) | Small dedicated core | Display power management during idle |

These firmware blobs are distributed in the `linux-firmware` package, *not* in the kernel source. They live under:

```
/lib/firmware/i915/
    dmc_ver2_14.bin          ← DMC for Gen12 (Tiger Lake)
    guc_70.6.4.bin           ← GuC for Gen12
    huc_gsc_cs_version_1.bin ← HuC with GSC signing

/lib/firmware/xe/
    guc_70.23.2.bin          ← GuC for Xe (Meteorite Lake+)
    huc_8.5.4.bin
```

The kernel announces required firmware file names in the module metadata:

```bash
modinfo i915 | grep firmware
# firmware:       i915/dmc_ver2_14.bin
# firmware:       i915/guc_70.6.4.bin
# firmware:       i915/huc_gsc_cs_version_1.bin
```

If the required firmware is not present in `/lib/firmware/`, the kernel module loads but logs warnings and disables features. GuC scheduling is now mandatory for many Xe operations; a missing GuC firmware means command submission will fail. [Source](https://dri.freedesktop.org/docs/drm/gpu/i915.html)

### 7.3 Ensuring Firmware is in the Initramfs

For systems where the root filesystem is encrypted or on a driver that itself requires GPU initialisation (rare but possible in embedded scenarios), the firmware must be included in the initramfs:

```bash
# Debian/Ubuntu: add firmware paths to initramfs
echo "FIRMWARE_DIRS=/lib/firmware" >> /etc/initramfs-tools/initramfs.conf
update-initramfs -u

# Fedora/RHEL: dracut includes linux-firmware automatically
dracut --force
```

### 7.4 Xe3 Firmware Upstreaming

In July 2025, Intel upstreamed GuC and HuC firmware for the Xe3 (Panther Lake) integrated graphics to `linux-firmware.git`. This illustrates that even firmware follows an upstream-first pattern: Intel submits firmware binaries to the `linux-firmware` repository, which distributions package separately from the kernel proper. [Source](https://www.phoronix.com/news/Intel-PTL-Xe3-GuC-HuC-Firmware)

The version-matching requirement between kernel and firmware adds a soft coupling: a new kernel may expect a newer firmware version. Distributions typically update `linux-firmware` and the kernel package in tandem for this reason.

---

## 8. Out-of-Tree Drivers: When Upstream is Delayed

### 8.1 Scenarios Requiring Out-of-Tree Development

Several legitimate scenarios require a kernel module that is not yet in the mainline kernel:

1. **New SoC GPU not yet upstreamed**: a new ARM SoC with a Mali or PowerVR variant where the DRM driver is under development.
2. **NDA hardware**: a GPU under NDA with a driver developed privately before public announcement.
3. **Proprietary display encoder**: MIPI DSI initialisation sequences for a custom panel, specific to one product.
4. **Vendor backport**: a newer driver backported to an older LTS kernel for an embedded product.
5. **In-house research GPU**: academic or industrial prototype.

### 8.2 The Standard Out-of-Tree Build

The canonical out-of-tree module build invocation is:

```bash
make -C /lib/modules/$(uname -r)/build M=$PWD modules
```

`-C /lib/modules/$(uname -r)/build` changes to the kernel build tree (a symlink to the kernel headers and Makefiles). `M=$PWD` tells kbuild that the module source lives in the current directory. The minimal `Kbuild` file:

```makefile
# Kbuild
obj-m := mygpu.o
mygpu-y := mygpu_drm.o mygpu_gem.o mygpu_display.o
```

Install the resulting module:

```bash
make -C /lib/modules/$(uname -r)/build M=$PWD modules_install
# Installs to /lib/modules/$(uname -r)/updates/
depmod -a
```

[Source](https://docs.kernel.org/kbuild/modules.html)

### 8.3 PCI Device Table

A DRM driver needs to declare which PCI device IDs it handles so the kernel can bind the driver automatically on device enumeration:

```c
static const struct pci_device_id mygpu_pci_ids[] = {
    { PCI_DEVICE(0xDEAD, 0xBEEF), .driver_data = MYGPU_VARIANT_A },
    { PCI_DEVICE(0xDEAD, 0xC0DE), .driver_data = MYGPU_VARIANT_B },
    { 0 }
};
MODULE_DEVICE_TABLE(pci, mygpu_pci_ids);
```

`MODULE_DEVICE_TABLE` generates the `alias` entries embedded in the `.ko` that `depmod` uses to populate `modules.alias`, enabling `modprobe` to load the driver automatically when `udev` detects the PCI device.

### 8.4 The Upstream Lifecycle

An out-of-tree driver follows a well-understood lifecycle towards upstream inclusion:

1. **Development phase**: module built and tested out-of-tree.
2. **RFC posting**: `[RFC PATCH 0/N] Add DRM driver for MyGPU` to `dri-devel@lists.freedesktop.org`.
3. **Review**: DRM maintainers review, request changes (often significant: no global state, proper PM ops, no `printk` spam, error path coverage).
4. **Repost**: iterated with `v2`, `v3`, ... patches.
5. **Staging (optional)**: if the driver is functional but rough, it may be merged into `drivers/staging/gpu/` as a stepping stone. Staging drivers are flagged for improvement or removal.
6. **Mainline merge**: accepted into `drm-misc-next` by a DRM maintainer, then pulled by Linus.

Until step 6, the out-of-tree maintainer must **forward-port** the driver to each new kernel release. This is the tax for staying out-of-tree: every kernel that changes `struct drm_device`, `struct gem_object_funcs`, or any API the driver uses requires a patch.

### 8.5 The Staging Area

`drivers/staging/` is an in-tree location for code not yet meeting mainline quality standards. Staging is technically in-tree (no DKMS needed) but is distinct from mainline:

- Staging drivers may use deprecated APIs.
- They carry a `TODO` file listing known issues.
- They are subject to removal if not improved within a few kernel cycles.
- `CONFIG_STAGING=y` is required to build them.

Notable GPU-adjacent code has passed through staging (e.g., older Vivante/etnaviv and Imagination PowerVR drivers before they matured).

---

## 9. Module Signing and Secure Boot

### 9.1 The Secure Boot Chain

Secure Boot establishes a chain of trust from UEFI firmware through bootloader to kernel:

```
UEFI firmware (Platform Key + db)
  └── shimx64.efi    (signed by Microsoft)
        └── grubx64.efi  (signed by distro key)
              └── vmlinuz  (signed by distro key)
                    └── kernel modules (.ko, signed)
```

If `CONFIG_MODULE_SIG_FORCE=y` is set in the kernel (as it is in most distribution kernels), any module that is not signed with a trusted key is rejected at load time.

### 9.2 `CONFIG_MODULE_SIG` and Kernel Build Options

Key kernel configuration options:

| Option | Effect |
|---|---|
| `CONFIG_MODULE_SIG=y` | Enable module signature verification support |
| `CONFIG_MODULE_SIG_FORCE=y` | Reject unsigned/untrusted modules (distro default) |
| `CONFIG_MODULE_SIG_SHA256=y` | Use SHA-256 for signature hashing |
| `CONFIG_MODULE_SIG_KEY` | Path to key used to sign in-tree modules |

Distribution kernels are built with `CONFIG_MODULE_SIG_FORCE=y`, which means DKMS-built modules must be signed to load.

### 9.3 Machine Owner Keys (MOK)

For user-signed modules (including DKMS modules), Linux uses the **MOK** (Machine Owner Key) facility provided by the `shim` bootloader. The workflow:

**Step 1: Generate a signing key pair**

```bash
openssl req -new -x509 \
  -newkey rsa:2048 \
  -keyout /etc/dkms/mok.key \
  -out /etc/dkms/mok.der \
  -outform DER \
  -days 36500 \
  -subj "/CN=DKMS Module Signing Key/" \
  -nodes
```

**Step 2: Enroll the public key with shim**

```bash
mokutil --import /etc/dkms/mok.der
# Enter a one-time password; the key will be enrolled on next reboot
```

**Step 3: Reboot and complete enrollment**

The `MOKManager.efi` screen appears during early boot (before GRUB), prompting for the password set in step 2. After enrollment, the key is stored in MokList NVRAM and trusted by the kernel.

**Step 4: Sign a module**

```bash
/lib/modules/$(uname -r)/build/scripts/sign-file \
  sha256 \
  /etc/dkms/mok.key \
  /etc/dkms/mok.der \
  /lib/modules/$(uname -r)/updates/dkms/mygpu.ko
```

### 9.4 DKMS Automatic Signing

DKMS 3.x automates this process. Configure `/etc/dkms/framework.conf`:

```bash
mok_signing_key="/var/lib/dkms/mok.key"
mok_certificate="/var/lib/dkms/mok.pub"
sign_file="/lib/modules/${kernelver}/build/scripts/sign-file"
```

Then run:

```bash
dkms generate_mok   # generates key pair in /var/lib/dkms/
```

With this configuration, DKMS automatically signs every module it builds using the `POST_BUILD` hook before installing it. The generated certificate must still be enrolled via `mokutil --import`.

Alternatively, use the `POST_BUILD` directive in `dkms.conf` to call a custom signing script:

```bash
POST_BUILD="sign_module.sh"
```

Where `sign_module.sh` contains the `sign-file` invocation above.

### 9.5 NVIDIA and Secure Boot

NVIDIA's proprietary modules were historically problematic with Secure Boot because signing a closed binary that could change between driver versions required each installed key to be trusted for that specific driver version. Modern distributions (Ubuntu 20.04+, Fedora 32+) handle this by:

1. Maintaining a long-lived DKMS MOK key enrolled once.
2. Using that key to sign all DKMS-built modules, including NVIDIA, regardless of driver version.

The NVIDIA open modules simplify this further: since the module sources are known and auditable, distributions can sign the built modules with their own distribution signing key, or users can use the standard DKMS signing flow. The `NVreg_EnableGpuFirmware=0` workaround (which disabled GSP firmware to avoid a signing-related initialisation issue) is no longer needed with current releases.

---

## 10. Packaging GPU Drivers for Distributions

### 10.1 Ubuntu

Ubuntu uses `ubuntu-drivers-common` for automatic detection and installation:

```bash
# Detect recommended driver
ubuntu-drivers devices
# == /sys/devices/pci0000:00/0000:00:01.0/0000:01:00.0 ==
# driver   : nvidia-driver-570 - third-party non-free recommended

# Install automatically
ubuntu-drivers autoinstall

# Packages installed:
# nvidia-driver-570          (meta-package: pulls userspace)
# nvidia-dkms-570            (DKMS source for kernel modules)
# nvidia-utils-570           (nvidia-smi, nvidia-settings)
# linux-modules-nvidia-570-$(uname -r)  (pre-built for HWE kernels)
```

DKMS source is installed to `/usr/src/nvidia-570.*/`. Ubuntu also ships pre-built modules for Hardware Enablement (HWE) kernels, reducing the need to build at install time for supported configurations.

For GL stack selection (switching between NVIDIA, Mesa, and other GL providers), Ubuntu uses `update-alternatives`:

```bash
update-alternatives --config x86_64-linux-gnu_gl_conf
# Selects which libGL.so.1 symlink is active
```

### 10.2 Fedora (RPMFusion)

```bash
# Enable RPMFusion free and non-free repositories
dnf install \
  https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
  https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

# Install akmods-based NVIDIA driver (open modules for Turing+)
dnf install akmod-nvidia nvidia-driver
```

`akmod-nvidia` triggers an automatic module build on first boot into a new kernel. The build is asynchronous — the system boots first, then builds in the background. A `systemctl status akmods` check shows build progress. `kmod-nvidia` (pre-built, specific to the kernel at packaging time) is the alternative for systems where the build toolchain should not be installed.

For GL alternative selection on Fedora:

```bash
alternatives --set gl /usr/lib64/nvidia/
```

### 10.3 Arch Linux

```bash
# Open kernel modules (recommended for RTX 20xx+)
pacman -S nvidia-open nvidia-utils

# DKMS variant (builds against any kernel, including linux-zen, linux-lts)
pacman -S nvidia-open-dkms nvidia-utils

# Proprietary modules (required for GTX 9xx-16xx, Kepler)
pacman -S nvidia nvidia-utils
```

Since late 2025, Arch defaults to `nvidia-open` for the stock `linux` kernel. `nvidia-open-dkms` is required for non-standard kernels (`linux-lts`, `linux-zen`, `linux-hardened`). [Source](https://www.phoronix.com/news/Arch-LInux-NVIDIA-Open-Default)

### 10.4 NixOS

NixOS handles NVIDIA driver packaging declaratively, avoiding the DKMS framework entirely. The NixOS module system manages kernel module compilation as part of the system build:

```nix
# /etc/nixos/configuration.nix
{ config, pkgs, ... }:
{
  services.xserver.videoDrivers = [ "nvidia" ];

  hardware.nvidia = {
    # Open modules: required for Blackwell, recommended for Turing+
    open = true;
    modesetting.enable = true;
    powerManagement.enable = false;
    package = config.boot.kernelPackages.nvidiaPackages.stable;
  };
}
```

NixOS generates a kernel-specific derivation for the NVIDIA module, building it during `nixos-rebuild switch` rather than at boot. The output is stored in the Nix store and symlinked into the active system. This gives reproducible builds and atomic rollbacks if a driver update breaks something. [Source](https://mynixos.com/nixpkgs/options/hardware.nvidia)

### 10.5 Gentoo

```bash
# Emerge NVIDIA drivers
emerge --ask x11-drivers/nvidia-drivers

# USE flags control features
# USE="kernel-open" selects open modules
# USE="dkms" enables DKMS-based installation
```

The Gentoo `ebuild` builds the module from source against the system's kernel headers at emerge time. Since Gentoo users compile their own kernels, there is typically no mismatch between the kernel headers and the running kernel.

### 10.6 Common Packaging Pitfalls

**Initramfs not updated after module install:**

If the NVIDIA (or any GPU) module needs to be available before the root filesystem is mounted (e.g., for a display output during early boot, or for resume from hibernation), the module must be in the initramfs. After installing or updating a DKMS module, initramfs must be regenerated:

```bash
# Debian/Ubuntu
update-initramfs -u -k all

# Fedora/RHEL
dracut --force
```

DKMS does not automatically trigger initramfs regeneration; this is handled by distro-specific hooks in the kernel package post-install scripts, which must explicitly call the initramfs tool after DKMS install completes.

**Kernel ABI break during distribution upgrade:**

When a distribution upgrades from kernel 6.11 to 6.12, DKMS modules compiled against 6.11 are not automatically rebuilt until the user reboots into 6.12 and DKMS triggers a build. If the build fails (due to a kernel API change in the NVIDIA glue layer), the user may boot without a working GPU driver. Always check `dkms status` after a kernel upgrade before rebooting.

**Conflicting nouveau and nvidia:**

Even with `blacklist nouveau` in `/etc/modprobe.d/`, `nouveau` may load if it is embedded in the initramfs before the blacklist is applied. The correct fix is to regenerate the initramfs *after* adding the blacklist, ensuring the blacklist file is included in the initrd.

**Version skew between kernel module and userspace:**

The NVIDIA kernel modules and userspace libraries must be exactly the same version. Installing `nvidia-dkms-570` (kernel) with `nvidia-utils-560` (userspace) will result in a version mismatch at runtime, typically manifesting as:

```
NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver.
Make sure that the latest NVIDIA driver is installed and running.
```

Distribution meta-packages declare strict version dependencies to prevent this, but manual installation or partial upgrades can break it.

---

## Roadmap

### Near-term (6–12 months)

- **DKMS 3.x maintenance and packaging integration**: The DKMS project (stable at 3.2.1 as of May 2025) continues incremental improvements focused on better integration with systemd-based distro packaging hooks, parallel builds for multi-kernel systems, and improved error reporting when kernel headers are missing. [Source](https://github.com/dkms-project/dkms/releases)
- **NVIDIA open-gpu-kernel-modules R570+ ABI stabilisation**: NVIDIA has stated that upstreaming the open kernel modules into the mainline kernel tree requires first stabilising the GSP firmware interface ABI; near-term driver releases (R570 and beyond) continue that stabilisation work before any upstream submission. Note: specific timeline needs verification. [Source](https://developer-stg.nvidia.com/blog/nvidia-releases-open-source-gpu-kernel-modules/)
- **Secure Boot certificate rotation (June 2026)**: Microsoft's original 2011 UEFI Driver Publisher certificate expires in June 2026; distributions shipping DKMS-signed modules via shim must migrate MOK infrastructure to the 2023 Microsoft UEFI CA key. Distro shim packages are being updated accordingly and DKMS documentation on `mokutil --import` will require refreshing for users. [Source](https://www.redhat.com/en/blog/expiration-secure-boot-signing-certificates-2026)
- **Nova-core landing in Linux 6.15–6.17**: The Rust-based Nova GPU driver skeleton was merged in Linux 6.15, with further GSP communication scaffolding landing in 6.17. As Nova matures it offers an in-tree Rust replacement for the DKMS-shipped NVIDIA open modules on Turing and later hardware. [Source](https://www.phoronix.com/news/Linux-6.17-NOVA-Driver)
- **Distribution adoption of NVIDIA open modules as default**: Arch Linux has already switched `nvidia-open` as the default for supported GPUs; Fedora and Ubuntu are evaluating the same transition for their 2026 release cycles, reducing dependence on the proprietary DKMS path. Note: exact release timelines need verification.

### Medium-term (1–3 years)

- **Nova-drm: full DRM driver on top of Nova-core**: The roadmap for Nova splits into `nova-core` (hardware/firmware abstraction) and `nova-drm` (DRM modesetting and render client). Completing `nova-drm` would allow a fully upstream, DKMS-free open driver for all Turing+ NVIDIA GPUs, eliminating the need for `open-gpu-kernel-modules` DKMS packaging entirely. [Source](https://rust-for-linux.com/nova-gpu-driver)
- **NVIDIA upstream submission to drm-misc**: Once the GSP firmware protocol is sufficiently documented and the nova-core/nova-drm split is stable, Red Hat and NVIDIA have indicated intent to submit the driver for inclusion in the `drm-misc` tree — following the same path that `amdgpu` and `xe` took. Note: needs verification against active mailing list discussion.
- **DKMS integration with systemd-sysupdate**: Proposals on the systemd mailing list discuss having `systemd-sysupdate` manage kernel + DKMS module bundles as atomic units, so that a kernel update and its associated DKMS rebuild happen together before the next boot, avoiding the window where a kernel is running but its DKMS module has not yet been rebuilt. Note: needs verification.
- **Improved out-of-tree driver co-installation tooling**: Fedora/RHEL's `akmods` (automatic `kmod` builds analogous to DKMS) and Debian's `dkms` framework are converging on common metadata formats; work is ongoing to allow a single source tree to produce both `.dkms` and `.akmod` packages from the same `dkms.conf`-compatible descriptor. Note: needs verification.
- **GSP firmware open documentation**: NVIDIA's commitment to open kernel modules implicitly requires documenting the GSP firmware protocol sufficiently for Nova to function without the binary GSP-RM blob. Progress on this documentation will determine how quickly Nova can replace the proprietary DKMS module in distributions. [Source](https://github.com/NVIDIA/open-gpu-kernel-modules)

### Long-term

- **End of DKMS for mainstream GPU drivers**: As AMD (`amdgpu`), Intel (`xe`), and NVIDIA (via Nova) all converge on fully upstreamed, in-tree kernel modules, the primary use case for DKMS shrinks to niche out-of-tree hardware and enterprise customisation. DKMS itself will persist for embedded, industrial, and proprietary peripheral drivers, but GPU workloads will likely no longer require it within the decade.
- **Stable kernel module ABI via Rust type safety**: The Rust-for-Linux project's longer-term goal is to express kernel internal interfaces through Rust trait bounds and stable API wrappers, providing a limited form of ABI stability without freezing C struct layouts. If this matures, it could reduce or eliminate the current requirement for full recompilation on every kernel version change. Note: highly speculative; not an official kernel commitment.
- **Hardware-enforced module attestation**: Future UEFI and TPM 2.0 workflows may allow kernel modules to be attested against a hardware-backed policy rather than only against static signing certificates, potentially simplifying the Secure Boot MOK dance for DKMS users on locked-down enterprise and consumer platforms. Note: speculative; no concrete upstream RFC at time of writing.
- **Unified firmware packaging across GPU vendors**: Long-term, `linux-firmware` and vendor firmware repositories may converge on a common metadata and update mechanism (analogous to `fwupd` for device firmware), allowing GPU firmware — currently split across NVIDIA GSP blobs, AMD `amdgpu` firmwares, and Intel GuC/HuC blobs — to be updated and verified through a single distribution channel independently of the kernel module. Note: needs verification against fwupd project roadmap.

---

## 11. Integrations

**Chapter 1 (DRM Architecture and the Driver Model)**: Every GPU kernel module — whether loaded via DKMS or compiled into the kernel tree — registers itself as a DRM driver using `drm_dev_alloc()` and `drm_dev_register()`. The DRM subsystem's driver model (probe/remove lifecycle, sysfs nodes, `/dev/dri/card0` creation) is the foundation that all GPU modules build on. The GPL-symbol restrictions that DKMS packages must navigate are directly tied to `EXPORT_SYMBOL_GPL` on these DRM registration functions.

**Chapter 5 (x86 GPU Drivers)**: The `amdgpu.ko` and Intel `i915.ko`/`xe.ko` covered in that chapter are the upstream counterparts to the DKMS-managed NVIDIA drivers discussed here. Understanding both helps system administrators decide whether a given driver deployment requires DKMS at all.

**Chapter 8 (Nouveau)**: The Nouveau kernel driver is the upstream open-source alternative to the NVIDIA proprietary DKMS module. Where DKMS delivers a closed driver, Nouveau delivers an open one — but historically at the cost of missing features (reclocking, Vulkan on newer hardware). The GSP-RM work described in Chapter 9 blurs this distinction.

**Chapter 10 (Nova — Rust NVIDIA Kernel Driver)**: Nova is the in-progress Rust-based kernel driver for NVIDIA hardware, intended to eventually replace both Nouveau and the need for NVIDIA's DKMS module entirely. It targets upstream inclusion and GSP-RM communication via an open firmware protocol, eliminating the binary stub architecture that the open-gpu-kernel-modules repository still requires.

**Chapter 76 (Contributing to Mesa)**: The upstream-first philosophy that AMD uses for `amdgpu.ko` mirrors the Mesa contribution philosophy. Both communities operate through mailing list review, iterative patch posting, and integration into shared review trees. The governance patterns and the tradeoffs between external packaging and upstream inclusion are closely analogous.

**Chapter 102 (DRM GPU Scheduler)**: The DRM GPU scheduler (`drivers/gpu/drm/scheduler/`) is used by `amdgpu`, `nouveau`, `lima`, `panfrost`, and increasingly by other drivers. Out-of-tree DRM drivers that want to use the scheduler must link against it, creating a `EXPORT_SYMBOL_GPL` dependency that forces a GPL-compatible module license.

---

*Sources consulted for this chapter:*
- [DKMS GitHub Repository (dkms-project/dkms)](https://github.com/dkms-project/dkms)
- [NVIDIA Open GPU Kernel Modules (NVIDIA/open-gpu-kernel-modules)](https://github.com/NVIDIA/open-gpu-kernel-modules)
- [NVIDIA Open Kernel Modules Documentation (595 series)](https://download.nvidia.com/XFree86/Linux-x86_64/595.58.03/README/kernel_open.html)
- [Linux Kernel: Building External Modules](https://docs.kernel.org/kbuild/modules.html)
- [Debian DKMS Packaging Guide](https://wiki.debian.org/DkmsPackaging)
- [Ubuntu DKMS Man Page](https://manpages.ubuntu.com/manpages/questing/en/man8/dkms.8.html)
- [AMDgpu (Linux kernel module) — Wikipedia](https://en.wikipedia.org/wiki/AMDGPU_(Linux_kernel_module))
- [Intel DRM i915 Driver Documentation](https://dri.freedesktop.org/docs/drm/gpu/i915.html)
- [NixOS NVIDIA Wiki](https://wiki.nixos.org/wiki/NVIDIA)
- [LWN: On the value of EXPORT\_SYMBOL\_GPL](https://lwn.net/Articles/154602/)
- [LWN: Questioning EXPORT\_SYMBOL\_GPL](https://lwn.net/Articles/603131/)
- [Arch Linux NVIDIA Open Modules Default (Phoronix)](https://www.phoronix.com/news/Arch-LInux-NVIDIA-Open-Default)
- [Intel Xe3/Panther Lake GuC/HuC Firmware Upstreamed (Phoronix)](https://www.phoronix.com/news/Intel-PTL-Xe3-GuC-HuC-Firmware)
- [RPMFusion akmod vs kmod Discussion](https://rpmfusion-developers.rpmfusion.narkive.com/2xZ1hzx6/akmod-vs-kmod)
- [AMDGPU Install Documentation](https://amdgpu-install.readthedocs.io/en/latest/install-installing.html)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
