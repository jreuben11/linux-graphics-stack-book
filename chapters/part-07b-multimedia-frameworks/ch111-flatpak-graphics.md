# Chapter 111: Flatpak Graphics — GPU Access in Sandboxed Applications

**Part VII — Application APIs and Middleware**
**Audiences**: Linux application developers packaging with Flatpak (primary); compositor and portal developers; security-minded desktop engineers
**Status**: First draft — 2026-06-19

---

## Scope

This chapter is for application developers who need real GPU acceleration inside a Flatpak sandbox, portal implementers who mediate that access, and desktop engineers who must reason about the security trade-offs. It explains, with technical precision, why GPU driver access is hard to sandbox correctly: Mesa DRI drivers are architecture-specific shared objects that must version-match the kernel driver, but bubblewrap establishes a separate mount namespace in which host library paths are absent by default. The chapter traces the complete data path through:

- **`finish-args` section** — Flatpak manifest declarations governing device and socket access
- **bubblewrap namespace** — the mount namespace providing filesystem isolation
- **GL extension mounting mechanism** — versioned Mesa DRI driver bundles merged into the sandbox runtime
- **ICD discovery for both OpenGL and Vulkan** — how vendor ICDs are located inside the sandbox
- **xdg-desktop-portal** — mediated access for screen capture and camera
- **VA-API video decode** — hardware video decode inside the sandbox
- **Electron/Chromium nesting under Flatpak** — nested sandbox handling and packaging patterns

Packaging best practices and a comparison with Snap and AppImage close the chapter.

Readers who need the underlying Vulkan ICD mechanics (loader, JSON manifests, RADV/ANV/NVK) should see **Ch24**; PipeWire's node graph, DMA-BUF negotiation, and the ScreenCast portal implementation are covered in **Ch38**.

---

## Table of Contents

1. [Introduction: The Security-Functionality Tension](#1-introduction-the-security-functionality-tension)
   - [1.1 Why GPU Sandboxing Is Uniquely Hard](#11-why-gpu-sandboxing-is-uniquely-hard)
   - [1.2 What is Flatpak?](#12-what-is-flatpak)
   - [1.3 What is bubblewrap?](#13-what-is-bubblewrap)
   - [1.4 What is the GL extension?](#14-what-is-the-gl-extension)
   - [1.5 What is OSTree?](#15-what-is-ostree)
2. [Flatpak Sandbox Architecture](#2-flatpak-sandbox-architecture)
3. [DRI Device Access](#3-dri-device-access)
4. [Mesa Driver Discovery Inside the Sandbox](#4-mesa-driver-discovery-inside-the-sandbox)
5. [Vulkan ICD Discovery in the Sandbox](#5-vulkan-icd-discovery-in-the-sandbox)
6. [The xdg-desktop-portal and the Permissions Model](#6-the-xdg-desktop-portal-and-the-permissions-model)
7. [VA-API and Hardware Video Decode](#7-va-api-and-hardware-video-decode)
8. [Electron and CEF Applications Under Flatpak](#8-electron-and-cef-applications-under-flatpak)
9. [Snap, AppImage, and Nix Comparison](#9-snap-appimage-and-nix-comparison)
   - [9.1 Snap](#91-snap)
   - [9.2 AppImage](#92-appimage)
   - [9.3 Nix and NixOS](#93-nix-and-nixos)
   - [9.4 Comparison Summary](#94-comparison-summary)
   - [9.5 Pros and Cons](#95-pros-and-cons)
10. [Convergence Efforts](#10-convergence-efforts)
    - [10.1 Portal Universalisation](#101-portal-universalisation)
    - [10.2 OCI and CDI: Container GPU Convergence](#102-oci-and-cdi-container-gpu-convergence)
    - [10.3 Nix + Declarative Flatpak](#103-nix--declarative-flatpak)
    - [10.4 Immutable OS Convergence](#104-immutable-os-convergence)
    - [10.5 Virtio-GPU Venus: Driver ABI Decoupling](#105-virtio-gpu-venus-driver-abi-decoupling)
    - [10.6 `systemd-sysext` for Driver Overlays](#106-systemd-sysext-for-driver-overlays)
    - [10.7 Linyaps](#107-linyaps)
    - [10.8 Direction of Travel](#108-direction-of-travel)
11. [Packaging Best Practices](#11-packaging-best-practices)
12. [Integrations](#12-integrations)
13. [References](#13-references)

---

## 1. Introduction: The Security-Functionality Tension

Flatpak's central promise is that a Linux application can be distributed as a self-contained package that runs inside a well-defined security boundary regardless of the host distribution. That promise interacts with GPU access in a structurally difficult way.

A GPU on Linux is not a single device file and a single library. It is a layered stack: a kernel driver (`i915`, `amdgpu`, `nouveau`, `nvidia`), a render node character device (`/dev/dri/renderD128`), a primary node (`/dev/dri/card0`), and a userspace driver — Mesa on open-source hardware — that is a collection of architecture-specific shared objects whose ABI must match the kernel module running on the host. When Flatpak installs an application it mounts a *runtime* — a versioned read-only root filesystem (e.g., `org.freedesktop.Platform//24.08`) — into the sandbox, not the host root. That runtime ships its own Mesa build. The problem: Mesa links into the kernel via DRM ioctls, and the ioctl interface is not guaranteed stable across Mesa versions. On hardware like NVIDIA, the situation is even stricter: the kernel module version (`/sys/module/nvidia/version`) and the userspace GLVND libraries must match *exactly*, down to the minor release.

The sandbox must therefore solve three distinct sub-problems simultaneously:

1. **Device visibility**: the render node must be bind-mounted into the sandbox namespace.
2. **Driver version alignment**: the Mesa build (or NVIDIA userspace) in the sandbox must match the host kernel driver.
3. **Privileged operations**: some GPU operations (KMS modesetting, `CAP_SYS_ADMIN`-gated ioctls) must be handled outside the sandbox by the compositor.

The mechanism Flatpak uses to address (2) is the *GL extension* — a separately packaged, automatically-managed Flatpak extension bundle that carries the Mesa build for the active GPU. Understanding how it works, and where it falls short, is the core technical subject of this chapter.

### 1.1 Why GPU Sandboxing Is Uniquely Hard

Sandboxing a pure CPU process is conceptually straightforward: restrict filesystem visibility, deny dangerous syscalls, isolate namespaces. GPU access breaks these assumptions at every layer.

**Mesa DRI ABI**: Unlike libc, which maintains a stable public ABI, Mesa's internal DRI interface between the GL/EGL front-end and the Gallium driver back-end is internal-only and changes between Mesa releases. The freedesktop.Platform runtime ships a Mesa build at a specific version; if the host kernel `amdgpu` or `i915` driver module expects a particular TTM ioctl or GEM object layout that a different Mesa version does not produce, the result can be GPU hangs, corrupted rendering, or EFAULT failures from the kernel. This is why the GL extension version is pinned to the runtime version — the entire userspace driver stack must be internally consistent.

**NVIDIA strict ABI**: NVIDIA's proprietary userspace does not expose a stable public ABI at all. The NVKMS and NV-NVKMS kernel modules are compiled to match exactly the userspace libraries (`libEGL_nvidia.so.VERSION`, `libGL.so.1.7.0`, etc.) that ship with the same driver tarball. A one-version difference between the host NVIDIA kernel module and the Flatpak-loaded NVIDIA userspace causes immediate failures with `ERR_INVALID_STATE` from `nvidia-modprobe` or crash dumps from the loader. This is why the extension ID encodes the driver version: `org.freedesktop.Platform.GL.nvidia-565-77` for kernel module version 565.77.

**Firmware blobs**: The kernel driver may load GPU firmware from `/lib/firmware/` on the host. This is a host-side concern that does not enter the sandbox (firmware is loaded by the kernel before the DRM character device becomes usable), but it means a missing firmware blob on the host manifests as GPU unavailability inside every Flatpak on that host, not as a Flatpak-specific problem.

**DRM file descriptor lifetime**: GPU work is submitted via ioctls on a file descriptor opened on the render node. That file descriptor — and the GPU context attached to it — has a lifetime tied to the process that opened it. Inside a Flatpak sandbox, Mesa opens `/dev/dri/renderD128` at GL context creation time and holds it open for the lifetime of the application. This is identical to non-sandboxed behaviour; the sandbox boundary does not affect the kernel-side fd lifetime or GPU scheduling.

### 1.2 What is Flatpak?

Flatpak is a distribution-agnostic application packaging and deployment system for Linux that bundles an application together with a fixed-version runtime environment. Unlike traditional distribution packages (`.deb`, `.rpm`) that depend on system libraries at specific versions, a Flatpak application references a *runtime* — a versioned, read-only filesystem image distributed via OSTree — that provides the complete set of shared libraries the application expects. The application itself ships as a second OSTree-managed layer mounting at `/app` inside the sandbox, while the runtime mounts at `/usr`. This arrangement allows an application to run identically on Debian, Fedora, Arch, and any other distribution because the runtime provides a consistent, known library set. Flatpak manages installation, updates, and permission grants through the `flatpak` command-line tool and the `flatpakd` system daemon; the Flathub repository (`https://flathub.org`) serves as the primary curated distribution channel. Permissions — including GPU access — are declared in the application's manifest under the `finish-args` key and enforced by the sandbox machinery at launch time. For graphics applications, the critical Flatpak-specific constraints are that the runtime's Mesa build must align with the host kernel driver version, and that privileged display operations must be delegated to the compositor outside the sandbox. Both constraints arise directly from the layered architecture of the Linux graphics stack.

### 1.3 What is bubblewrap?

Bubblewrap (`bwrap`) is an unprivileged sandboxing tool that constructs Linux container namespaces without requiring root privileges or a setuid helper. It operates by calling `clone()` with `CLONE_NEWUSER | CLONE_NEWNS | CLONE_NEWPID` (and optionally `CLONE_NEWNET` and `CLONE_NEWIPC`), establishing a user namespace in which the calling process's UID and GID map to UID 0 and GID 0 inside the new namespace. Within the resulting mount namespace, bubblewrap assembles a synthetic root filesystem from caller-specified bind-mounts: read-only mounts from the Flatpak runtime and application bundles, writable mounts from per-application state directories, and selected host paths such as the Wayland socket and D-Bus bus socket. Optionally, a seccomp BPF filter restricts the set of syscalls available to the sandboxed process. Flatpak uses bubblewrap as its sandbox engine; the `flatpak run` command constructs a `bwrap` command line that encodes every permission grant from the manifest (`--device=dri`, `--socket=wayland`, `--share=network`, etc.) as corresponding `bwrap` flags. Because bubblewrap relies on user namespaces rather than setuid helpers, its security depends on the host kernel's implementation of `CLONE_NEWUSER`. Kernel hardening configurations that disable unprivileged user namespaces (`kernel.unprivileged_userns_clone=0` on some distributions) prevent bubblewrap from functioning without a privileged mode fallback.

### 1.4 What is the GL extension?

The GL extension is a Flatpak extension category specifically designed to carry versioned GPU userspace drivers — Mesa builds or NVIDIA proprietary libraries — matched to the kernel driver version on the host machine. The problem it addresses is fundamental: a Flatpak runtime ships a fixed Mesa version compiled into its read-only image, but the host GPU's kernel driver (e.g., `amdgpu`, `i915`, `nvidia`) may require a different Mesa or NVIDIA userspace version for correct ioctl compatibility. A GL extension carries the full Mesa DRI megadriver (`libgallium_dri.so`), GLVND vendor JSON files, and Vulkan ICD JSON manifests for a specific GPU class, and declares itself as a subdirectory extension of `org.freedesktop.Platform.GL`. At application launch, Flatpak detects the active GPU driver (via `/sys/class/drm/card0/device/driver/module/`) and automatically selects the matching GL extension, mounting its contents into `lib/GL/` inside the sandbox and using the extension's `merge-dirs` list to overlay GLVND configuration, Vulkan ICD descriptors, and the DRI driver search path. The version encoding in the extension ID (e.g., `org.freedesktop.Platform.GL.mesa-extra`, `org.freedesktop.Platform.GL.nvidia-565-77`) ensures that only a compatible userspace stack is loaded for the running kernel driver. When no matching extension is installed, Flatpak falls back to the runtime's bundled Mesa, which may or may not work correctly with the host kernel.

### 1.5 What is OSTree?

OSTree (formally **libostree**) is a content-addressed, immutable filesystem tree versioning system that Flatpak uses to distribute and update runtimes and applications [Source](https://ostreedev.github.io/ostree/). Its design is modelled on Git's object store, but applied to full filesystem trees of compiled binaries rather than source code.

**Object model.** Every file in an OSTree repository is identified by the SHA-256 hash of its content and stored once — identical files across different runtimes or application versions share a single on-disk inode via hard links. A *commit* object records the root directory tree hash, a parent commit reference, metadata (timestamp, subject, version string), and arbitrary key–value annotations. A *ref* (analogous to a Git branch) names a current commit hash. The full ancestry of refs in a repository constitutes the deployment history.

**Deployment mechanics.** On the client side, `ostree pull` fetches only the objects present in the new commit that are absent locally, then assembles the new root filesystem tree from hard links into a staging directory (e.g. `/ostree/deploy/default/deploy/<checksum>/`). The active deployment is swapped by atomically updating a symlink; the old deployment remains on disk until explicitly pruned, enabling instant rollback:

```bash
ostree admin status          # list deployments and their checksums
ostree admin rollback        # switch to the previous deployment
ostree refs                  # list all named refs in the local repo
ostree log org.freedesktop.Platform/x86_64/24.08   # commit history for a runtime ref
```

**Flatpak integration.** Flatpak wraps OSTree for application management. Each Flatpak remote (e.g. Flathub at `https://dl.flathub.org/repo/`) is an OSTree repository. `flatpak install` resolves the application ref to a commit, pulls missing objects, and hard-links them into `~/.local/share/flatpak/repo/` (user install) or `/var/lib/flatpak/repo/` (system install). The runtime, application, and any extensions (including GL extensions) are each separate OSTree refs that are mounted together at launch via the bubblewrap bind-mount layer described in §2.1.

**Relevance to GPU drivers.** The GL extension mechanism (§1.4) is entirely OSTree-based: `org.freedesktop.Platform.GL.nvidia-565-77` is an OSTree ref whose commit bundles the NVIDIA 565.77 userspace libraries. When a new NVIDIA host driver is installed, running `flatpak update` pulls the corresponding GL extension commit — only the delta objects, typically a few megabytes — and hard-links them into the repo. At the next application launch, bubblewrap mounts the updated extension. The OSTree hard-link model means a single copy of each Mesa `.so` or NVIDIA library serves every Flatpak application that references the same runtime version, keeping storage overhead bounded.

**Delta updates.** OSTree supports *static deltas* — pre-computed binary diffs between two specific commits distributed as a single blob, analogous to a Git pack file. Flathub generates static deltas for major runtime version upgrades (e.g. `24.08` → `25.08`), which reduces network traffic from gigabytes to tens of megabytes. `flatpak update --appstream` fetches repository metadata (AppStream XML) via a delta summary rather than re-fetching the full summary file.

---

## 2. Flatpak Sandbox Architecture

### 2.1 Bubblewrap and Linux Namespaces

Flatpak launches each application inside a container constructed by **bubblewrap** (`bwrap`), a low-level unprivileged sandboxing tool maintained under the `containers/` organisation on GitHub [Source](https://github.com/containers/bubblewrap). Bubblewrap requires no root privileges because it leverages Linux *user namespaces* (`CLONE_NEWUSER`), which since kernel 3.8 allow an unprivileged process to own a new UID/GID mapping and establish subordinate namespaces.

A typical Flatpak invocation causes `bwrap` to create the following namespace set:

| Namespace | Flag | Purpose |
|-----------|------|---------|
| User | `CLONE_NEWUSER` | Maps caller's UID 1000 to UID 0 inside sandbox |
| Mount | `CLONE_NEWNS` | Isolates filesystem view; root is a `tmpfs` |
| PID | `CLONE_NEWPID` | Sandbox sees PID 1 as its init |
| IPC | `CLONE_NEWIPC` | Isolates SysV IPC and POSIX message queues |
| Network | `CLONE_NEWNET` (optional) | Isolates network stack unless `--share=network` |

The new mount namespace starts empty on a `tmpfs`. Flatpak then uses `bwrap`'s `--ro-bind` and `--bind` flags to assemble the sandbox filesystem:

- `/usr` ← `org.freedesktop.Platform` runtime (read-only bind-mount)
- `/app` ← the application's own files (read-only bind-mount)
- `/var` ← `~/.var/app/APP_ID/` for per-app writable state
- `/run/host/` ← selected host paths (e.g., fonts, icons)
- `/dev/` ← synthesised device tree (only explicitly granted devices)

Because `CLONE_NEWNS` gives the sandbox a completely separate mount namespace, paths that exist on the host (such as `/usr/lib/x86_64-linux-gnu/dri/`) are invisible inside unless explicitly bind-mounted. This is the root of the Mesa driver discovery problem.

### 2.2 Seccomp Filtering

In addition to namespace isolation, Flatpak applies a **seccomp BPF filter** to the sandboxed process, loaded via `bwrap --seccomp FD` [Source](https://deepwiki.com/flatpak/flatpak/5-security). Flatpak currently uses a *deny-list* model that blocks a small set of dangerous syscalls rather than an allowlist:

- `chroot` → `EPERM`
- `TIOCSTI` ioctl (terminal injection) → `EPERM` (CVE-2017-5226 mitigation)
- `clone3` with `CLONE_NEWUSER` → `ENOSYS` (prevents namespace escapes, CVE-2021-41133)
- Older `clone` with `CLONE_NEWUSER` → allowed under conditions

Graphics-intensive applications (games, GPU compute) benefit from the fact that most graphics ioctls (`DRM_IOCTL_*`, `AMDGPU_INFO`, `I915_GEM_*`) are *not* blocked — the seccomp overhead per-syscall would be prohibitive, and the privilege model for render nodes does not require these to be restricted at the seccomp layer.

### 2.3 The Flatpak Runtime

The **runtime** is a versioned read-only filesystem image distributed via OSTree. Current major runtimes include:

- `org.freedesktop.Platform//24.08` — the GNOME platform runtime, ships Mesa 24.x, GLib, GTK4, systemd libraries
- `org.kde.Platform//6.7` — KDE Plasma runtime
- `org.electronjs.Electron2.BaseApp//24-08` — base for Electron applications

The runtime mounts at `/usr` inside the sandbox. Its Mesa build is compiled against a specific kernel ABI and with specific driver backends. This is why the runtime version number matters: `//24.08` indicates that the runtime was built and tested against the 24.08 release cycle of the freedesktop SDK.

The application's files install into `/app` — a second read-only overlay. When `flatpak-builder` builds an application from source, all compiled binaries, data files, icons, and `.desktop` entries land in `/app`. The two layers (`/usr` from the runtime, `/app` from the application) provide complete library separation: the application can depend on whatever version of a library the runtime ships, and the runtime is shared by all applications that use it, reducing disk usage.

User-writable state is isolated per application under `~/.var/app/APP_ID/`. The XDG base directories inside the sandbox resolve as:
- `XDG_DATA_HOME` → `~/.var/app/APP_ID/data/`
- `XDG_CONFIG_HOME` → `~/.var/app/APP_ID/config/`
- `XDG_CACHE_HOME` → `~/.var/app/APP_ID/cache/`

This isolation ensures one application's corrupted configuration does not affect others, and complete removal of `~/.var/app/APP_ID/` fully uninstalls the application's state — analogous to Android's per-app data partitions.

### 2.4 D-Bus and XDG Portals

Communication with services outside the sandbox happens over D-Bus. The sandboxed application connects to the *session bus* socket at `$XDG_RUNTIME_DIR/bus`, which is bind-mounted into the sandbox. The **xdg-desktop-portal** service owns the well-known D-Bus name `org.freedesktop.portal.Desktop` and exposes a collection of portal interfaces for operations that require host OS access: file dialogs, screen capture, camera, printing, and so forth.

The general flow for any portal request is:

```
App (inside sandbox)
  → D-Bus method call on org.freedesktop.portal.Desktop
    → xdg-desktop-portal (host service, validates caller's App ID)
      → compositor-specific backend (xdg-desktop-portal-gnome / -kde / -wlr)
        → response (fd, URI, or permission grant) passed back to app
```

The portal uses the caller's Flatpak App ID (derived from `$FLATPAK_ID` or the D-Bus sender's flatpak metadata) to scope permissions to a specific application, storing persistent decisions in the permission store at `~/.local/share/flatpak/db/` [Source](https://github.com/flatpak/xdg-desktop-portal/wiki/The-Permission-Store).

---

## 3. DRI Device Access

### 3.1 Render Nodes and Primary Nodes

The Linux DRM subsystem exposes two classes of device node for each GPU:

- **Render nodes** (`/dev/dri/renderD128`, `renderD129`, …): unprivileged; any process with read/write permission can submit GPU work. No capability is required. Access is controlled purely by file permissions (typically `crw-rw---- root:render`).
- **Primary nodes** (`/dev/dri/card0`, `card1`, …): required for KMS modesetting (display output, CRTC configuration). Becoming DRM master — needed to call `drmModeSetCrtc`, `drmModeAtomicCommit`, etc. — requires that the process either owns the VT or is granted master explicitly by the current master.

Inside Flatpak, the **compositor** is DRM master on the primary node; sandboxed applications never need to be DRM master themselves. A Wayland client submits buffers to the compositor's Wayland socket; the compositor performs the KMS pageflip. This privilege decomposition is fundamental to why Flatpak GPU access can be made safe.

### 3.2 The `--device=dri` Permission

When an application's manifest includes `--device=dri` in `finish-args`, Flatpak instructs bubblewrap to bind-mount `/dev/dri/` into the sandbox namespace [Source](https://docs.flatpak.org/en/latest/sandbox-permissions.html). This gives the sandbox access to both `renderD*` and `card*` nodes. The `card*` node *is visible* inside the sandbox — it is not hidden. However, the sandboxed app cannot become DRM master on it because (a) it is not DRM master at the time of sandbox entry, and (b) the privileged `DRM_IOCTL_SET_MASTER` ioctl requires that the process is the session VT owner or was explicitly granted master by one.

As of Flatpak 1.18 (June 2026), the `--device=dri` permission additionally grants access to `/dev/kfd`, the AMD Kernel Fusion Driver interface used by ROCm and OpenCL compute workloads [Source](https://linuxiac.com/flatpak-1-18-released-with-amd-compute-interface-support/). This means machine-learning applications requiring AMD GPU compute no longer need `--device=all` (which would expose USB devices, `/dev/mem`, and other unrelated hardware).

NVIDIA proprietary drivers also expose `/dev/nvidia0`, `/dev/nvidiactl`, and `/dev/nvidia-uvm` character devices. When `--device=dri` is set and the NVIDIA kernel module is loaded, Flatpak bind-mounts these as well, because they are detected as GPU-related devices.

### 3.3 Verifying GPU Access

To diagnose GPU availability inside a sandbox:

```bash
# Check which GPU driver Mesa selected
flatpak run --command=bash --device=dri org.example.App -- \
  env LIBGL_DEBUG=verbose glxinfo 2>&1 | head -20

# Force software rendering to test fallback
flatpak run --command=bash org.example.App -- \
  env LIBGL_ALWAYS_SOFTWARE=1 glxinfo 2>&1 | grep "renderer string"

# Check DRI device visibility
flatpak run --command=bash org.example.App -- ls -la /dev/dri/
```

Without `--device=dri`, `/dev/dri/` will be absent or empty inside the sandbox and any Mesa path that opens a render node will fail. The failure mode depends on the application: Mesa may fall back to software rasterisation via **llvmpipe** or **softpipe** (if those are compiled in), or the application may crash or report a black window. The fallback is not guaranteed because the runtime Mesa build may not have shipped with software rasterisation enabled.

### 3.4 Security Implications

Granting `--device=dri` is the standard path to GPU access and is considered safe for general use. The render node does not allow:

- Accessing GPU VRAM of other processes (GPU memory isolation is enforced by the kernel driver via GEM object handle tables scoped per `drm_file`)
- Becoming DRM master or modifying display outputs
- Reading framebuffer contents of other processes (the GPU's memory protection hardware prevents cross-process VRAM reads at the hardware level on modern AMD, Intel, and NVIDIA GPUs)
- Escaping the sandbox (no publicly disclosed privilege escalation via render-node ioctls as of June 2026)

It does allow an application to submit arbitrary GPU workloads and to allocate GPU memory, which is a necessary trade-off for any GPU-accelerated app. In a shared multi-user or cloud desktop environment, GPU workload isolation via mechanisms like `AMDGPU_INFO_MEMORY` and cgroup-based GPU scheduling is a separate concern beyond the Flatpak security model.

The `--device=all` permission (which grants access to every device node including `/dev/mem`, USB devices, and block devices) is a qualitatively different and much more dangerous grant. Flathub reviewers explicitly reject applications that request `--device=all` when only GPU access is needed.

---

## 4. Mesa Driver Discovery Inside the Sandbox

### 4.1 The ICD Discovery Problem

On a typical host Linux install, Mesa's DRI infrastructure uses a single megadriver library (`libgallium_dri.so`) — the result of Mesa's consolidation from per-driver `.so` files (`radeonsi_dri.so`, `i965_dri.so`) to a single megadriver that contains all supported Gallium/Iris drivers. This library lives in a path such as `/usr/lib/x86_64-linux-gnu/dri/` (Debian/Ubuntu) or `/usr/lib64/dri/` (Fedora). The EGL and GLX implementations (`libEGL_mesa.so`, `libGL.so.1` via GLVND) locate this megadriver at runtime using a combination of compile-time paths and environment variables (`LIBGL_DRIVERS_PATH`, `EGL_DRIVERS_PATH`).

The GLVND (GL Vendor-Neutral Dispatch) architecture that modern distributions use complicates the picture further. GLVND provides a vendor-neutral `libGL.so.1` and `libEGL.so.1` that dispatch to vendor-specific implementations at runtime. On the host, the GLVND EGL vendor configuration directory (`/usr/share/glvnd/egl_vendor.d/` or `/etc/glvnd/egl_vendor.d/`) contains JSON files telling GLVND which vendor library to load (e.g., `50_mesa.json` pointing to `libEGL_mesa.so`). Inside the Flatpak sandbox, `/usr` is the *runtime's* `/usr`, not the host's, so the GLVND vendor configuration must also come from the sandbox — and it does, via the GL extension's `glvnd/egl_vendor.d/` in `merge-dirs`.

The runtime ships its own Mesa build, which lives at the expected path `/usr/lib/x86_64-linux-gnu/dri/` *within the runtime*. For AMD (RADV/radeonsi) and Intel (ANV/iris) hardware, the runtime Mesa version is typically within a few months of the host kernel driver's expected Mesa version, and the DRM ioctl ABI is stable enough that this works in practice. However, the runtime cannot bundle GPU-vendor-specific firmware or NVIDIA proprietary blobs, and it cannot know in advance which GPU the application will run on.

### 4.2 The GL Extension Mechanism

Flatpak solves this with *GL extensions* — a special category of Flatpak extension that is automatically selected and mounted based on the active GPU driver. The extension point is declared in the runtime's metadata:

```ini
[Extension org.freedesktop.Platform.GL]
version=1.4
directory=lib/GL
add-ld-path=lib
merge-dirs=vulkan/icd.d;glvnd/egl_vendor.d;OpenCL/vendors;lib/dri;lib/d3d;
subdirectories=true
no-autodownload=true
autodelete=false
```

The key line is `merge-dirs`: when a GL extension is mounted, its contents are *merged* into the sandbox's directory tree rather than replacing existing paths. This allows an extension to inject its DRI drivers, Vulkan ICD files, and EGL vendor configurations alongside the runtime's own files.

The `version=1.4` here is a Flatpak extension ABI version, not a Mesa or driver version. NVIDIA extensions use `1.4` because NVIDIA's userspace ABI has no stability guarantee across minor releases, requiring exact version matching.

### 4.3 Mesa Extensions: `org.freedesktop.Platform.GL.default`

For Intel, AMD, and other open-source GPU hardware, the relevant extension is `org.freedesktop.Platform.GL.default`. This extension contains a Mesa build compiled specifically for the runtime's release cycle. When a runtime such as `org.freedesktop.Platform//24.08` is installed, Flatpak automatically downloads and activates `org.freedesktop.Platform.GL.default//24.08` [Source](https://docs.flatpak.org/en/latest/extension.html).

The extension mounts at `/usr/lib/x86_64-linux-gnu/GL/` inside the sandbox. Its directory structure mirrors what Mesa expects:

```
GL/
├── lib/
│   ├── dri/
│   │   └── libgallium_dri.so   ← Mesa megadriver for AMD/Intel/Nouveau
│   ├── libEGL_mesa.so.0
│   └── libGLX_mesa.so.0
├── vulkan/
│   └── icd.d/
│       ├── radeon_icd.x86_64.json
│       ├── intel_icd.x86_64.json
│       └── lvp_icd.x86_64.json
└── glvnd/
    └── egl_vendor.d/
        └── 50_mesa.json
```

The `merge-dirs` mechanism means these files are visible alongside anything the runtime itself provides in the same paths, with the extension taking precedence.

### 4.4 NVIDIA: `org.freedesktop.Platform.GL.nvidia-${VERSION}`

For NVIDIA proprietary drivers, the mechanism is more complex. Flatpak detects the NVIDIA kernel module version by reading `/sys/module/nvidia/version` from the host through `/run/host/` [Source](https://blog.tingping.se/2018/08/26/flatpak-host-extensions.html). It then maps that version string (e.g., `565.77`) to a branch of the `org.freedesktop.Platform.GL.nvidia-565-77` extension. The version tag is inserted into the extension ID to enforce the exact match between kernel driver and userspace.

This extension is generated automatically: rather than being built statically, a bootstrapping helper downloads the official NVIDIA driver tarball from NVIDIA's servers at install time and extracts the relevant userspace libraries into the Flatpak extension store. The helper must be statically compiled (linked against `libarchive`, `libz`, `liblzma`) to avoid dependency on any runtime [Source](https://blogs.igalia.com/vjaquez/digging-further-into-flatpak-with-nvidia/).

To list the active GL driver(s) Flatpak has selected:

```bash
flatpak --gl-drivers
# Output example:
# default
# nvidia-565-77
```

To override for testing:

```bash
FLATPAK_GL_DRIVERS=default flatpak run org.example.App
```

### 4.5 DRI_PRIME and Multi-GPU in Sandbox

**DRI_PRIME** (for selecting a secondary GPU on hybrid graphics systems) works inside the Flatpak sandbox provided the required render nodes are accessible via `--device=dri` and the GL extension includes both GPU drivers. On a system with an Intel integrated GPU and an AMD discrete GPU, `org.freedesktop.Platform.GL.default` includes `libgallium_dri.so` with both `radeonsi` and `i965`/`iris` Gallium drivers compiled in. Setting `DRI_PRIME=1` inside the sandbox selects the discrete GPU as usual.

```bash
flatpak run --env=DRI_PRIME=1 org.example.App
```

For debugging, `GALLIUM_DRIVER=softpipe` or `GALLIUM_DRIVER=llvmpipe` inside the sandbox overrides the Mesa driver selection to force software rendering.

---

## 5. Vulkan ICD Discovery in the Sandbox

### 5.1 How the Vulkan Loader Finds ICDs Inside the Sandbox

The Vulkan loader (`libvulkan.so.1`) discovers ICDs by scanning a set of directories for JSON manifest files. On the host, these directories include `/usr/share/vulkan/icd.d/`, `/etc/vulkan/icd.d/`, and `$XDG_DATA_HOME/vulkan/icd.d/` (full mechanics in **Ch24**). Inside the Flatpak sandbox, `/usr/share/vulkan/icd.d/` resolves to the *runtime's* ICD directory, not the host's.

The runtime itself ships a minimal Vulkan ICD: **Lavapipe** (`lvp_icd.x86_64.json`), the LLVM-based CPU Vulkan implementation. This provides a software-rendering fallback but not hardware acceleration.

Hardware Vulkan ICDs enter the sandbox via the same GL extension `merge-dirs` mechanism that handles OpenGL. The `org.freedesktop.Platform.GL.default` extension ships both OpenGL DRI drivers and Vulkan ICD JSON files, merged into the sandbox's ICD search paths:

- `radeon_icd.x86_64.json` → RADV (AMD Vulkan from Mesa)
- `intel_icd.x86_64.json` → ANV (Intel Vulkan from Mesa)
- `nouveau_icd.x86_64.json` → NVK (NVIDIA open-source Vulkan from Mesa, available from Mesa 24.0+)

Note the separation: **NVK** (the open-source Mesa Vulkan driver for NVIDIA) ships in `org.freedesktop.Platform.GL.default` like any other Mesa driver. The `org.freedesktop.Platform.GL.nvidia-${VERSION}` extension provides the **proprietary** NVIDIA Vulkan ICD (`libGLX_nvidia.so`, `nvidia_icd.json`). These are entirely separate code paths that must not be conflated.

### 5.2 Verifying Vulkan in the Sandbox

```bash
# Run vulkaninfo inside the sandbox
flatpak run --command=vulkaninfo --device=dri org.example.App 2>&1 | \
  grep -A2 "VkPhysicalDeviceProperties"

# Check which ICDs were loaded
flatpak run --command=bash --device=dri org.example.App -- \
  env VK_LOADER_DEBUG=all vulkaninfo 2>&1 | grep "Found ICD"

# Force a specific ICD for debugging
flatpak run --env=VK_ICD_FILENAMES=/usr/lib/x86_64-linux-gnu/GL/vulkan/icd.d/radeon_icd.x86_64.json \
  --device=dri org.example.App
```

If Vulkan hardware acceleration is absent despite `--device=dri` being set, the most common cause is that the GL extension has not been downloaded for the correct architecture. The Flatpak tooling will usually auto-download `GL.default` but may not auto-install GL extensions for 32-bit compatibility (`GL32.default`), which some games require.

### 5.3 Vulkan Layers in Sandbox

Vulkan layers (validation, RenderDoc capture, etc.) are loaded from `$VK_LAYER_PATH` or the standard layer directories (`/usr/share/vulkan/explicit_layer.d/`, `/usr/share/vulkan/implicit_layer.d/`). Inside Flatpak, the runtime's layer directories hold whatever the runtime ships (typically `VK_LAYER_KHRONOS_validation` for developer runtimes). `$VK_LAYER_PATH` can be overridden via `--env` in `finish-args` to point to extension-provided layers.

The `org.freedesktop.Platform.VulkanLayer` extension point allows packaged Vulkan layers to be injected into the sandbox automatically [Source](https://docs.flatpak.org/en/latest/extension.html). A Flatpak-packaged RenderDoc, for instance, can expose a `VulkanLayer` extension that the application can opt into:

```yaml
# In the application manifest's finish-args, to enable a Vulkan layer extension:
add-extensions:
  org.freedesktop.Platform.VulkanLayer.renderdoc:
    directory: lib/extensions/vulkan/renderdoc
    version: '24.08'
    add-ld-path: lib
    merge-dirs: share/vulkan/explicit_layer.d
```

The layer JSON files merged into `/usr/share/vulkan/explicit_layer.d/` are then visible to the Vulkan loader's layer enumeration, and `VK_INSTANCE_LAYERS=VK_LAYER_RENDERDOC_Capture` activates them as usual.

---

## 6. The xdg-desktop-portal and the Permissions Model

### 6.1 Purpose and Architecture

The **xdg-desktop-portal** (XDP) is a D-Bus service running on the host (outside any sandbox) that proxies privileged operations on behalf of sandboxed applications. Its primary job is to expose a compositing-environment-agnostic set of interfaces that work regardless of whether the desktop is GNOME, KDE, or a wlroots compositor [Source](https://flatpak.github.io/xdg-desktop-portal/docs/).

XDP consists of two layers:

- A **frontend** (`/usr/libexec/xdg-desktop-portal`) that implements the `org.freedesktop.portal.*` D-Bus interfaces and enforces the permission store
- One or more **backends** that implement the `org.freedesktop.impl.portal.*` interfaces and handle desktop-environment-specific UI (dialogs, pickers, screen capture):
  - `xdg-desktop-portal-gnome` for GNOME Shell / Mutter
  - `xdg-desktop-portal-kde` for KWin / KDE Plasma
  - `xdg-desktop-portal-wlr` for wlroots-based compositors (Sway, Wayfire, etc.)
  - `xdg-desktop-portal-hyprland` for Hyprland

### 6.2 Graphics-Relevant Portal Interfaces

**Screenshot portal** (`org.freedesktop.portal.Screenshot`): allows a sandboxed application to request a screenshot without holding any static manifest permission. The portal presents a user-facing confirmation dialog (the exact UI depends on the compositor backend), then instructs the compositor backend to capture the screen. Under Wayland, the backend uses `wlr-screencopy-unstable-v1` (on wlroots compositors such as Sway and Wayfire) or the newer standardised `ext-image-capture-source-v1` / `ext-image-copy-capture-v1` protocols, which became official Wayland extensions in 2024–2025. The captured buffer is returned to the application as a file URI pointing to a temporary file or, in newer portal versions, directly as a memory-backed file descriptor.

D-Bus call:
```
org.freedesktop.portal.Screenshot.Screenshot(
    parent_window: s,
    options: a{sv}    ← {"interactive": <b true>}
) → uri: s
```

**ScreenCast portal** (`org.freedesktop.portal.ScreenCast`): returns a PipeWire file descriptor for streaming screen content. The `OpenPipeWireRemote()` method grants the app access to a scoped PipeWire remote from which it can subscribe to a screen capture stream. PipeWire and the DMA-BUF mechanics behind this are covered in detail in **Ch38**.

**Camera portal** (`org.freedesktop.portal.Camera`): similar mechanism to ScreenCast but for camera devices. The app calls `OpenPipeWireRemote()` and receives a PipeWire fd connected to a camera source node managed by libcamera or V4L2 (see **Ch96**).

**FileChooser portal** (`org.freedesktop.portal.FileChooser`): enables open/save dialogs without granting filesystem permissions. Returns file descriptors into the sandbox for the selected files.

### 6.3 The Permission Store

Portal decisions made by the user (e.g., "always allow this app to access the camera") are stored in an SQLite-like permission database at `~/.local/share/flatpak/db/` with one file per portal type. The store uses the Flatpak App ID as the primary key, so permissions are app-scoped.

Permissions can be inspected and revoked:

```bash
# List all portal permissions for an app
flatpak permission-list org.example.App

# Remove a stored permission
flatpak permission-remove org.example.App camera
```

The XDP frontend validates the caller's identity using D-Bus sender metadata and the Flatpak metadata carried in the process's namespace, preventing one sandboxed app from impersonating another.

---

## 7. VA-API and Hardware Video Decode

### 7.1 VA-API Driver Discovery Inside the Sandbox

VA-API (Video Acceleration API) requires two things: a DRI render node for submitting video decode commands, and a VA-API driver library (`iHD_drv_video.so` for Intel, `radeonsi_drv_video.so` for AMD) that implements the `VADriverVTable`. The driver library path is controlled by `LIBVA_DRIVERS_PATH`; without it, libva scans a compile-time default (`/usr/lib/x86_64-linux-gnu/dri/` on Debian).

Inside the Flatpak sandbox, the VA-API driver libraries come from the GL extension, not the runtime itself. The `org.freedesktop.Platform.GL.default` extension includes VA-API driver modules merged via its `lib/dri/` directory in `merge-dirs`. This means:

1. `--device=dri` must be set (for the render node)
2. The GL extension must be installed (for the VA-API driver `.so` files)
3. `LIBVA_DRIVERS_PATH` inside the sandbox must resolve to where the extension mounted the drivers

The standard path after extension mounting is `/usr/lib/x86_64-linux-gnu/GL/lib/dri/` or a symlink to it.

### 7.2 Firefox Flatpak: A Case Study

The Firefox Flatpak (`org.mozilla.firefox`) is one of the highest-profile users of VA-API hardware decode inside a sandbox. Its manifest specifies:

```yaml
finish-args:
  - --device=dri
  - --socket=wayland
  - --socket=fallback-x11
  - --share=ipc
  - --share=network
  - --filesystem=xdg-download
```

Hardware decode in Firefox Flatpak requires two extensions:

1. **`org.freedesktop.Platform.GL.default`** (or the NVIDIA variant) — provides the VA-API driver libraries (iHD for Intel, radeonsi for AMD)
2. **`org.freedesktop.Platform.ffmpeg-full`** (branch matching the runtime, e.g., `24.08`) — provides FFmpeg with hardware decode codec support

In `about:config`, `media.ffmpeg.vaapi.enabled=true` activates the VA-API decode path. The renderer can be verified in `about:support` under "Media" → "VA-API Enabled" [Source](https://discourse.flathub.org/t/how-to-enable-video-hardware-acceleration-on-flatpak-firefox/3125).

Testing VA-API availability from inside the sandbox:

```bash
# Launch a shell inside Firefox Flatpak's sandbox environment
flatpak run --command=bash org.mozilla.firefox

# Then inside that shell:
LIBVA_DRIVER_NAME=radeonsi vainfo
# or for Intel:
LIBVA_DRIVER_NAME=iHD vainfo
```

The `LIBVA_DRIVER_NAME` override forces a specific driver and is useful when auto-detection fails because the sandbox's Mesa build identifies the hardware differently from the host.

### 7.3 PipeWire Video Integration

For screen capture and camera access (rather than file-based video decode), VA-API hardware encode/decode interoperates with PipeWire via DMA-BUF buffer sharing. A sandboxed application that receives a PipeWire DMA-BUF stream from the ScreenCast portal can import those buffers into a VA-API surface for hardware encode without any CPU copy. The full DMA-BUF ↔ PipeWire ↔ VA-API chain is documented in **Ch38**.

---

## 8. Electron and CEF Applications Under Flatpak

### 8.1 The Nested Sandbox Problem

Electron applications embed Chromium, which has its own GPU process and its own sandbox model. When an Electron app runs inside Flatpak, the result is a *nested sandbox*: Flatpak's bubblewrap sandbox wraps Chromium's own sandbox. The interaction between these two layers is non-trivial.

Chromium's sandbox on Linux uses one of two mechanisms:
- **Namespace sandbox**: creates unprivileged user namespaces for renderer/GPU processes
- **setuid sandbox** (`chrome-sandbox`): a setuid root binary that creates namespaces with elevated privilege

Inside Flatpak's bubblewrap, creating new user namespaces (`CLONE_NEWUSER`) is blocked by Flatpak's seccomp filter when the target process is already inside a user namespace (to prevent namespace escape chains). This breaks Chromium's namespace sandbox. The setuid approach fails because the Flatpak sandbox runs as an unprivileged user without `CAP_SETUID`.

The naive workaround — passing `--no-sandbox` to Electron — disables Chromium's own sandbox entirely, running renderer and GPU processes unsandboxed inside the Flatpak container. This is a security regression and should be avoided.

### 8.2 Zypak: The Flatpak-Native Chromium Sandbox

**Zypak** (by refi64, available at [github.com/refi64/zypak](https://github.com/refi64/zypak)) is the standard solution for Electron/Chromium applications on Flatpak. It intercepts Chromium's sandbox calls via `LD_PRELOAD` and redirects them to Flatpak's own sandbox mechanism.

Zypak implements two strategies [Source](https://github.com/refi64/zypak/blob/main/README.md):

**Mimic strategy** (Flatpak < 1.8.2): Zypak intercepts zygote protocol messages and replaces `fork`/`exec` operations with `flatpak-spawn --sandbox` calls. Each child process is a new Flatpak sub-sandbox rather than a Chromium namespace sandbox. This has higher startup latency and memory cost (no shared zygote memory pages).

**Spawn strategy** (Flatpak ≥ 1.8.2, recommended): uses the `expose-pids` feature and the `SpawnStarted` D-Bus signal to run the true Chromium zygote inside a Flatpak sandbox. The zygote process remains functional, preserving shared memory and startup speed. A dedicated D-Bus thread inside the sandbox manages `flatpak-spawn` calls for new child processes.

Most Electron-based Flathub applications use Zypak, including Visual Studio Code (`com.visualstudio.code`), which uses `org.electronjs.Electron2.BaseApp` as its base application. The manifest includes:

```yaml
command: zypak-wrapper.sh code
```

The `zypak-wrapper.sh` script prepends the necessary environment to redirect Chromium sandbox calls through Zypak.

### 8.3 GPU Acceleration in Electron Flatpak Applications

With `--device=dri` in `finish-args` and the GL extension installed, Electron Flatpak applications can use GPU acceleration for WebGL rendering (via ANGLE or native OpenGL) and GPU-accelerated compositing. Electron on Linux uses the `--ozone-platform=wayland` flag when running under Wayland (automatically set on modern Electron versions when `$WAYLAND_DISPLAY` is set and `--socket=wayland` is granted).

The GPU process inside Electron runs as a Zypak sandbox child. It opens `/dev/dri/renderD128` and loads Mesa from the GL extension's paths. The sandbox-within-sandbox setup means there are two layers of namespace isolation, which is acceptable: the GPU process's GL work is legitimate, and its access to the render node is necessary.

```bash
# Enable verbose GPU logging in a sandboxed Electron app
flatpak run --env=LIBGL_DEBUG=verbose com.visualstudio.code --verbose
```

If GPU acceleration is not working, the most common cause is that Zypak is not correctly wrapping the sandbox binary, causing Chromium to fall back to `--no-sandbox` mode and disabling its GPU sandbox — which paradoxically sometimes *enables* GPU acceleration but at a security cost. The canonical fix is to ensure the Flatpak manifest has `--device=dri` and that `org.freedesktop.Platform.GL.default` is installed.

---

## 9. Snap, AppImage, and Nix Comparison

Understanding Flatpak's GPU model is easier with a concrete comparison to the two other dominant universal packaging formats on Linux.

The choice between packaging formats has historically been driven by distribution politics as much as technical merit — Ubuntu/Canonical favour Snap, the Red Hat/GNOME ecosystem leans toward Flatpak, and AppImage serves the "just a binary" distribution model. This section evaluates the three purely on their GPU access and sandboxing mechanisms, leaving the political dimension to the reader.

### 9.1 Snap

Snap applications run under **AppArmor** profiles managed by `snapd`. GPU access in Snap uses *interfaces* — named capability grants. The relevant interface is `opengl`:

```yaml
# snapcraft.yaml
plugs:
  opengl:
    interface: opengl
```

Snaps with the `opengl` interface connect to the host's graphics stack via the **`graphics-core24`** Snap interface [Source](https://discourse.ubuntu.com/t/the-graphics-core20-snap-interface/23000), which bind-mounts a curated set of graphics libraries from a separate `mesa-core24` Snap content provider into the application Snap's namespace. This is conceptually similar to Flatpak's GL extension mechanism but implemented via AppArmor-mediated bind mounts rather than Flatpak's OSTree-based extension system.

The `graphics-core24` interface provides:
- Mesa DRI drivers
- GLVND libraries
- Vulkan ICDs (Mesa + vendor)
- VA-API drivers

NVIDIA in Snap uses the `hardware-observe` interface to read `/sys/module/nvidia/version`, then loads the matching proprietary driver from a separately distributed NVIDIA content Snap.

Snaps running with `--classic` confinement bypass all AppArmor restrictions and access the host filesystem directly — the equivalent of `--device=all --filesystem=host` in Flatpak, but with no portal layer.

### 9.2 AppImage

AppImage is not a sandbox format. An AppImage is a self-contained SquashFS image that mounts itself via FUSE and `exec`s its main binary with the bundled library paths prepended. Security confinement is absent by default.

For GPU access, an AppImage that bundles Mesa will have version-mismatch issues on hosts with a different Mesa version. Many AppImages solve this by *not* bundling Mesa and relying on the host's system Mesa (`LD_LIBRARY_PATH` manipulation or `RUNPATH`). This produces the widest compatibility but re-introduces host dependency and ABI fragility.

AppImage + Firejail provides manual sandboxing:

```bash
firejail --appimage ./MyApp.AppImage
```

Firejail restricts filesystem access and can limit network, but it does not have a portal layer for mediated out-of-sandbox access. It is a lower-effort solution than either Flatpak or Snap.

### 9.3 Nix and NixOS

Nix occupies a fundamentally different position from the three formats above: rather than a per-application sandbox, it is a purely functional package manager and (on NixOS) a full system configuration language. Its relevance to GPU graphics is real but operates at a different layer — driver management and reproducibility rather than runtime sandboxing.

**NixOS system-level GPU configuration.** On NixOS the entire graphics stack is declared in `/etc/nixos/configuration.nix`. Mesa, Wayland compositors, and hardware-specific VA-API drivers are all managed as a single reproducible closure:

```nix
# /etc/nixos/configuration.nix
hardware.graphics = {                       # renamed from hardware.opengl in NixOS 24.05
  enable = true;
  extraPackages = with pkgs; [
    intel-media-driver                      # iHD VA-API driver (Intel Gen 8+)
    intel-vaapi-driver                      # i965 VA-API driver (Intel Gen 4–7)
    vaapiVdpau                              # VA-API → VDPAU bridge
    libvdpau-va-gl                          # VDPAU → OpenGL bridge
    rocmPackages.clr                        # AMD ROCm OpenCL runtime
  ];
  extraPackages32 = with pkgs.pkgsi686Linux; [ intel-vaapi-driver ];
};
```

NVIDIA on NixOS uses the `hardware.nvidia` module, which manages DKMS kernel module installation, GLVND setup, and the proprietary userspace libraries as a coherent unit:

```nix
hardware.nvidia = {
  open = true;                              # open kernel module (Turing/Ampere+ recommended)
  modesetting.enable = true;               # required for Wayland
  powerManagement.enable = true;           # RTD3 runtime power management
  package = config.boot.kernelPackages.nvidiaPackages.stable;
};
services.xserver.videoDrivers = [ "nvidia" ];
```

The critical advantage: NixOS pins the kernel module version to the userspace driver version in the same `configuration.nix` commit. A `nixos-rebuild switch` atomically upgrades both, eliminating the ABI mismatch that afflicts rolling-release distributions and is the root cause of Flatpak's complex GL extension versioning scheme.

**The non-NixOS problem: nixGL.** Running Nix-packaged OpenGL or Vulkan applications on a non-NixOS host (e.g. Nix on Ubuntu or Fedora) hits the same driver-version problem as AppImage: Nix-packaged Mesa is compiled against a specific DRI driver path that does not match the host's `/usr/lib/x86_64-linux-gnu/dri/`. The result is a blank window or a segfault.

The canonical solution is **nixGL** [Source](https://github.com/nix-community/nixGL), a community wrapper that generates a shell script setting `LD_LIBRARY_PATH` and `LIBGL_DRIVERS_PATH` to the host's OpenGL libraries:

```bash
# Install nixGL into the user profile
nix profile install --impure github:nix-community/nixGL

# Run an OpenGL app via nixGL wrapper
nixGL glxgears

# Run with Vulkan
nixVulkan vkcube

# Or compose in a flake devShell
nix run --impure github:nix-community/nixGL#nixGLDefault -- ./myapp
```

The `--impure` flag is required because nixGL must inspect the host's `/proc/driver/nvidia/version` or Mesa DRI directory — information that is by definition outside the pure Nix input graph. A pure-Nix alternative, `nix-gl-host` [Source](https://github.com/numtide/nix-gl-host), achieves the same result with a Rust binary that reads `/etc/os-release` and `/proc/driver/nvidia/version` to locate host drivers at runtime without `--impure` flake evaluation.

**Packaging OpenGL applications in nixpkgs.** When an upstream application targets inclusion in nixpkgs, the convention is `wrapOpenGL` (part of `makeWrapper`) to bake the correct Mesa path into the binary's `RPATH` or a wrapper script:

```nix
# In a nixpkgs derivation
nativeBuildInputs = [ makeWrapper ];
postInstall = ''
  wrapProgram $out/bin/myapp \
    --prefix LD_LIBRARY_PATH : ${lib.makeLibraryPath [ mesa libGL ]} \
    --set VK_ICD_FILENAMES ${mesa.drivers}/share/vulkan/icd.d/radeon_icd.x86_64.json
'';
```

For applications that need the full GLVND + vendor ICD stack, `pkgs.addOpenGLRunpath` is the nixpkgs idiom since NixOS 23.11 — it patches RPATH post-install to include the OpenGL driver output of the `mesa` derivation.

**GPU compute in Nix devshells.** Nix's reproducibility makes it attractive for CUDA and ROCm development environments, where version coupling between compilers, runtime libraries, and kernel modules is notoriously fragile:

```nix
# flake.nix devShell for CUDA
devShells.default = pkgs.mkShell {
  buildInputs = with pkgs; [
    cudaPackages.cudatoolkit
    cudaPackages.cuda_nvcc
    cudaPackages.libcublas
    cudaPackages.cudnn
    linuxPackages.nvidia_x11   # kernel-matching userspace
  ];
  shellHook = ''
    export CUDA_PATH=${pkgs.cudaPackages.cudatoolkit}
    export LD_LIBRARY_PATH=${pkgs.linuxPackages.nvidia_x11}/lib:$LD_LIBRARY_PATH
  '';
};
```

ROCm in nixpkgs is packaged under `pkgs.rocmPackages` with modules for `rocblas`, `miopen`, `hipcc`, and `clr` (the HIP runtime). Because ROCm requires `/dev/kfd` and DRM render nodes, devshells typically add `--device=/dev/kfd --device=/dev/dri` when invoked inside a `nix develop` environment on a non-NixOS host.

**nixpkgs-wayland overlay.** For cutting-edge Wayland compositors and toolkits, the `nixpkgs-wayland` community overlay [Source](https://github.com/nix-community/nixpkgs-wayland) provides frequently updated packages for Sway, wlroots, wlr-randr, and related tools that lag behind upstream in the stable nixpkgs channel.

**Reproducibility versus flexibility tradeoff.** NixOS's all-or-nothing approach to GPU driver management delivers the strongest reproducibility guarantee of any Linux packaging system — the same `configuration.nix` produces the same driver stack across hardware generations. The cost is that per-application driver version isolation (as in Flatpak's GL.nvidia-${VERSION} extension) is not the Nix model; an NixOS system runs one Mesa and one NVIDIA userspace version system-wide. This is a deliberate design choice: in the Nix model, version mismatches are prevented at configuration time rather than mediated at runtime.

### 9.4 Comparison Summary

| Feature | Flatpak | Snap | AppImage | Nix/NixOS |
|---------|---------|------|---------|-----------|
| **Sandbox technology** | bubblewrap (user namespaces) | AppArmor + seccomp | None (optional: Firejail) | None (system config, not sandbox) |
| **GPU access mechanism** | `--device=dri` + GL extension | `opengl` interface + graphics-core content Snap | Host Mesa or bundled Mesa | Declarative `hardware.graphics.*`; nixGL on non-NixOS |
| **Driver version matching** | GL.nvidia-${VERSION} extension | NVIDIA content Snap | Relies on host ABI | Atomic system closure (NixOS); nixGL/nix-gl-host (non-NixOS) |
| **Portal layer** | xdg-desktop-portal (well-defined D-Bus API) | snapd-desktop-integration (limited) | None | None (not an app sandbox) |
| **Vulkan support** | Via GL extension `merge-dirs` | Via graphics-core content Snap | Host Vulkan loader | nixpkgs Mesa + vendor ICDs; nixVulkan wrapper |
| **VA-API** | Via GL extension + ffmpeg-full | Via graphics-core content Snap | Host libva | `hardware.graphics.extraPackages` |
| **CUDA / ROCm** | Explicit `/dev/nvidia*` grants | `hardware-observe` + NVIDIA content Snap | Host libs | `cudaPackages` / `rocmPackages` devShells |
| **Classic/unrestricted mode** | `--device=all --filesystem=host` (discouraged) | `--classic` | Default | Default (no sandbox concept) |
| **Reproducibility** | Runtime version-pinned by extension | Content Snap version-pinned | Bundled at image build time | Strongest: full closure including kernel module |
| **Security strength** | Highest (user namespace + portal) | Medium (AppArmor, no portal standard) | None by default | N/A (system-level, not per-app) |

The key differentiator for graphics is the **portal layer**: Flatpak's xdg-desktop-portal provides a standardised, user-consented mechanism for screen capture, camera, and file access, while Snap's equivalent is less standardised and AppImage has none. Nix/NixOS sits outside this axis entirely — it provides the strongest driver-version reproducibility guarantee but is a system-configuration tool rather than a per-application sandbox.

### 9.5 Pros and Cons

#### Flatpak

**Pros:**
- **Strongest security boundary** for a desktop app format: user namespaces + seccomp + portal mediation. A Flatpak GPU app cannot escape its sandbox without an explicit permission grant or a kernel namespace vulnerability.
- **GL extension mechanism** solves the hardest problem in Linux GPU packaging — matching the userspace driver to the running kernel module — in a way that is automatic, distribution-agnostic, and atomic (OSTree delta updates).
- **xdg-desktop-portal** provides a standardised, user-consented API surface for screen capture, camera, file access, and notifications. A Flatpak app on GNOME and KDE uses the same portal calls; the compositor provides the implementation.
- **Runtime sharing via OSTree hard links** means multiple Flatpak apps sharing the same `org.freedesktop.Platform//24.08` runtime store only one copy of Mesa, glibc, and GTK on disk.
- **Flathub** is the largest curated Linux application catalogue, with CI-enforced metadata validation, static analysis, and sandbox review.

**Cons:**
- **Startup latency**: bubblewrap namespace setup and OSTree bind-mount assembly add ~50–150 ms of process startup overhead visible in time-to-first-frame for GPU apps.
- **Mesa version lag**: the `org.freedesktop.Platform` runtime is released on a ~6-month cycle. Apps that require a Mesa feature newer than the current runtime version cannot ship it via the stable extension mechanism without maintaining a custom runtime fork.
- **D-Bus complexity**: portal interactions require the `xdg-desktop-portal` daemon and a compositor-specific backend (GNOME, KDE, wlroots) to be running. Headless or minimal Wayland compositor environments may lack the portal backend, breaking camera or screen-capture flows.
- **NVIDIA proprietary driver**: the GL extension ID must exactly match the installed kernel module version. A user who updates the NVIDIA host driver without immediately running `flatpak update` is left with a version mismatch that silently falls back to software rendering. The mismatch is not always surfaced clearly.
- **Compute workloads**: GPU compute access (`/dev/kfd` for ROCm, `/dev/nvidia-uvm` for CUDA) requires explicit `--device` grants that are not standardised and not enforced by Flathub policy — each upstream application manifest handles it differently.

---

#### Snap

**Pros:**
- **AppArmor policy** is enforced by the kernel and cannot be bypassed by userspace bugs in the sandbox tool — unlike user-namespace-based approaches, which depend on `CLONE_NEWUSER` being correctly implemented.
- **`graphics-core24` content Snap** provides a curated, tested Mesa + Vulkan + VA-API bundle maintained by Canonical that works with the `opengl` interface across all supported Ubuntu releases without application developers needing to understand driver versioning.
- **Automatic updates**: `snapd` performs background delta updates and transactional refresh, with rollback on failure. No manual `flatpak update` required.
- **`--classic` confinement** provides an escape hatch for developer tools and other software that legitimately needs unrestricted host access, with a clear opt-in model.

**Cons:**
- **Ubuntu/Canonical ecosystem bias**: the `graphics-core24` content Snap and `snapd` infrastructure are primarily developed and tested on Ubuntu. Non-Ubuntu distributions have experienced `snapd` packaging and integration issues.
- **No standardised portal layer**: `snapd-desktop-integration` exists but is not equivalent to xdg-desktop-portal. Screen capture, camera access, and file dialogs lack the same cross-compositor standardisation.
- **Squashfs mount overhead**: each Snap is a SquashFS image that must be loop-mounted at application launch. Systems with many installed Snaps accumulate loop devices, which can affect mount latency and `df` output readability.
- **No user-namespace sandbox on non-AppArmor kernels**: Snap security degrades to weaker confinement on distributions without AppArmor loaded (e.g. Fedora with SELinux, Arch with no LSM). AppArmor is Ubuntu-default but not universal.
- **NVIDIA driver matching** relies on the NVIDIA content Snap, which is a separate independently versioned package. Mismatches between the NVIDIA host kernel module and the content Snap version are possible and can be harder to diagnose than the Flatpak GL extension versioning.

---

#### AppImage

**Pros:**
- **Zero installation** and **zero daemon**: an AppImage is a self-contained executable. Running it requires only FUSE and execute permission. Suitable for CI environments, air-gapped systems, and one-off testing.
- **Maximum application control**: the developer decides every bundled library version. There is no runtime platform to lag behind upstream Mesa or toolchain releases.
- **Portable across distributions**: an AppImage built on a sufficiently old glibc (e.g. Ubuntu 20.04 base) runs on any Linux distribution with a compatible or newer glibc, without distribution packaging.
- **No update infrastructure required**: users can carry an AppImage on a USB drive and run it on any compatible machine.

**Cons:**
- **No sandbox**: AppImage provides zero security isolation. The application runs with full user-level access to the filesystem, network, and devices. Malicious or buggy AppImages have the same capabilities as any user process.
- **GPU driver ABI fragility**: if the AppImage bundles Mesa, it must be compatible with both the host kernel DRM driver and host glibc. Bundling an old Mesa can result in missing KMS features, broken sync, or corrupted rendering on newer GPU hardware. Not bundling Mesa re-introduces host dependency.
- **No portal integration**: screen capture, camera access, and privileged display operations require the AppImage to call host libraries directly, bypassing the standardised xdg-desktop-portal consent model.
- **Update story is per-application**: AppImageUpdate and the `update` section in the AppImage spec exist, but there is no centralised update infrastructure equivalent to Flathub or `snapd`. Each application developer must implement update checking independently.
- **Large file sizes** for applications that bundle a full Mesa + Qt/GTK stack, since there is no runtime sharing mechanism.

---

#### Nix / NixOS

**Pros:**
- **Strongest reproducibility guarantee** in the Linux ecosystem: the entire dependency closure — including the GPU kernel module version, Mesa, and compute runtimes — is pinned to a single `flake.lock` hash. The same flake input hash produces bit-identical builds across machines and time.
- **Atomic GPU driver upgrades**: on NixOS, `hardware.nvidia.package` and the kernel module are upgraded together in a single `nixos-rebuild switch`. There is no window where the userspace driver version and the kernel module version are mismatched — the new generation is fully assembled before activation.
- **Best CUDA/ROCm devShell ergonomics**: `cudaPackages` and `rocmPackages` in nixpkgs provide versioned, pinned GPU compute toolchain environments. `nix develop` drops into a shell with the exact NVCC, cuDNN, and library versions pinned, reproducible across developer machines.
- **Hard-link deduplication**: Nix's `/nix/store` uses the same content-addressed hard-link approach as OSTree. Multiple generations of NixOS configurations sharing a Mesa version store only one copy.
- **No runtime to lag behind**: there is no `org.freedesktop.Platform` equivalent with a 6-month release cycle. nixpkgs tracks upstream Mesa, GCC, and library releases continuously.

**Cons:**
- **Not a per-application sandbox**: NixOS provides no bubblewrap-equivalent isolation between applications. A GPU application on NixOS runs with full user privileges. For multi-tenant or kiosk use cases, additional tooling (systemd units with `PrivateDevices=`, `bubblewrap` manual invocation, or `nix-sandbox`) is required.
- **The non-NixOS problem**: running Nix-packaged GPU apps on Ubuntu, Fedora, or Arch requires nixGL or nix-gl-host and the `--impure` flag. The ergonomics are significantly worse than Flatpak's automatic GL extension selection.
- **Nix language learning curve**: `configuration.nix` and flake-based development environments require learning the Nix expression language, which has a steep initial curve compared to writing a Flatpak manifest YAML.
- **`nixpkgs` GPU packaging coverage gaps**: while Mesa, CUDA, and ROCm are well-packaged, some vendor-specific VA-API drivers, media SDK libraries, and proprietary compute libraries have incomplete or outdated nixpkgs packaging requiring overlays or `fetchurl` workarounds.
- **Binary cache dependency**: the Nix build model is source-first with binary caching via `cache.nixos.org`. GPU compute packages (CUDA, ROCm) are large; on a slow connection or in environments where the binary cache is unavailable, building from source is prohibitively slow.

---

## 10. Convergence Efforts

The four packaging models described in §9 are not converging toward a single winner; they are converging toward a common set of interfaces that any of them can implement. The convergence is happening simultaneously at the runtime layer (how GPU devices and display resources are accessed) and at the packaging/reproducibility layer (how software and its dependencies are managed and updated).

### 10.1 Portal Universalisation

The **xdg-desktop-portal** is the clearest convergence point across all formats. Snap's `snapd-desktop-integration` is gradually implementing the same D-Bus interfaces rather than maintaining a parallel API surface. The `wp_security_context_v1` Wayland protocol (Ch46) lets the compositor tag any sandboxed client connection — Flatpak, Snap, or a manually `bwrap`-wrapped process — with an app ID, enabling per-application protocol policy enforcement at the compositor level without format-specific knowledge. The result is that the portal layer is becoming the neutral lingua franca for display, camera, screen capture, and file access regardless of how the surrounding sandbox was constructed.

### 10.2 OCI and CDI: Container GPU Convergence

The **Container Device Interface (CDI)** [Source](https://github.com/cncf-tags/container-device-interface) is a vendor-neutral JSON specification for injecting GPU devices and their associated userspace libraries into OCI containers. Driven by the NVIDIA Container Toolkit and adopted by AMD ROCm for `podman` and `docker`, CDI defines a standard `DeviceSpec` that maps device nodes and host library paths into a container namespace:

```json
{
  "cdiVersion": "0.6.0",
  "kind": "nvidia.com/gpu",
  "devices": [{ "name": "0", "containerEdits": {
    "deviceNodes": [{"path": "/dev/nvidia0"}, {"path": "/dev/nvidiactl"}],
    "mounts": [{"hostPath": "/usr/lib/x86_64-linux-gnu/libcuda.so.1",
                "containerPath": "/usr/lib/x86_64-linux-gnu/libcuda.so.1"}]
  }}]
}
```

Flatpak's GL extension mechanism and Snap's `graphics-core24` content Snap both solve the same driver-injection problem with format-specific machinery. CDI is the candidate to unify them: any container runtime that speaks CDI can inject the correct GPU userspace without needing to know whether the container was built by `flatpak-builder`, `snapcraft`, or `buildah`. Alexander Larsson has discussed `flatpak-builder` emitting OCI images (rather than OSTree commits alone), which would allow Flatpak applications to be stored and distributed via standard OCI registries while retaining the portal security model — making a Flatpak app and a `podman` container interchangeable at the storage layer.

### 10.3 Nix + Declarative Flatpak

The `nix-flatpak` NixOS module [Source](https://github.com/gmodena/nix-flatpak) manages Flatpak application installations declaratively inside `configuration.nix`, combining Nix's reproducibility for *which* apps and versions are pinned with Flatpak's runtime sandboxing and GL extension GPU driver matching at runtime:

```nix
# /etc/nixos/configuration.nix
services.flatpak.enable = true;
services.flatpak.packages = [
  { appId = "com.valvesoftware.Steam"; origin = "flathub"; }
  { appId = "org.blender.Blender"; origin = "flathub"; commit = "abc123"; }
];
```

This is the most practically available convergence point today. The user gets Nix-level declarative system configuration (including pinned Flatpak commits) combined with Flatpak's portal mediation and automatic GL extension version matching — without having to choose between the two ecosystems.

### 10.4 Immutable OS Convergence

Fedora Atomic (Silverblue/Kinoite), **Universal Blue**, and **SteamOS 3** on the Steam Deck represent an architectural pattern that unifies three of the four models: an OSTree-managed immutable base (reproducibility analogous to NixOS), Flatpak for user applications (sandboxing + portal), and `distrobox`/`toolbx` OCI containers for mutable development environments with GPU passthrough.

Universal Blue extends this by layering custom `ublue-os` images on top of Fedora Atomic with NVIDIA or AMD drivers pre-baked into the base OSTree commit:

```bash
# Universal Blue image selection (GPU-specific images)
# nvidia: bakes NVIDIA kernel module + userspace into the OSTree base
rpm-ostree rebase ostree-unverified-registry:ghcr.io/ublue-os/silverblue-nvidia:latest

# After rebase: NVIDIA driver is part of the immutable base image —
# same atomicity guarantee as NixOS's hardware.nvidia module
```

This gives the NixOS guarantee — kernel module and userspace driver upgrade atomically — implemented with OSTree image layers instead of the Nix store, and available on any Fedora-compatible system without requiring the Nix toolchain.

### 10.5 Virtio-GPU Venus: Driver ABI Decoupling

Sebastian Wick's January 2026 proposal (referenced in §16 of this chapter) for Flatpak to use Mesa's **Venus Vulkan driver** over a `vtest` Unix-socket transport to `virgl_test_server` on the host would decouple the sandbox's Mesa version from the host kernel DRM driver entirely:

```
Flatpak sandbox (any Mesa version)
    │  Venus Vulkan ICD
    │  vtest Unix socket
    ▼
virgl_test_server (host Mesa, speaks host DRM ioctls)
    │  KMS / GEM / TTM ioctls
    ▼
/dev/dri/renderD128
```

The sandbox-side Venus speaks a stable virtual GPU protocol; the host-side server handles kernel driver ioctls with the correct Mesa version. This eliminates the entire GL extension version-matching problem — there is no longer a need for `org.freedesktop.Platform.GL.nvidia-565-77` because the sandbox never speaks kernel ioctls directly. It converges toward the NixOS model (one canonical Mesa on the host, all applications use it) but without requiring a system-level configuration manager or any format-specific extension infrastructure.

### 10.6 `systemd-sysext` for Driver Overlays

`systemd-sysext` [Source](https://www.freedesktop.org/software/systemd/man/systemd-sysext.html) provides signed, immutable overlay extensions for `/usr` that can be applied at boot to any systemd-based system — including OSTree-managed immutable distributions — without rebuilding the base image:

```bash
# A GPU vendor ships a sysext image containing Mesa 25.x or NVIDIA 570.x
# as a signed squashfs extension
systemd-sysext merge           # applies extensions into /usr overlay
systemd-sysext status          # list active extensions
```

GPU driver vendors could distribute Mesa or NVIDIA userspace as a `sysext` image, giving any immutable OSTree-based system (SteamOS, Fedora Atomic, Universal Blue) the same atomic driver upgrade guarantee NixOS achieves through store generations — without requiring a Nix toolchain or a Flatpak GL extension. The extension is GPG-signed, version-matched to the OS image via metadata, and applied atomically at boot.

### 10.7 Linyaps

**Linyaps** (formerly Linglong), developed by Deepin/UnionTech [Source](https://linglong.dev/), is a newer packaging format that uses OCI containers natively with xdg-desktop-portal integration from the ground up. Unlike Flatpak (which layered OCI export onto an OSTree-native design) or Snap (which uses SquashFS with a proprietary store), Linyaps treats an OCI image as its native unit and runs it via `bubblewrap` with full portal mediation. It is a smaller ecosystem than Flatpak or Snap but represents the cleanest implementation of the "OCI + portal" convergence direction without legacy design constraints.

### 10.8 Direction of Travel

The convergence is happening at two independent layers simultaneously:

| Layer | Converging toward |
|-------|-------------------|
| **Runtime (GPU + display access)** | xdg-desktop-portal + CDI + `wp_security_context_v1` + Virtio-GPU Venus |
| **Packaging (reproducibility + distribution)** | OCI images + declarative management (Nix modules, `nix-flatpak`, Universal Blue image layers, `systemd-sysext`) |

The end state several projects are moving toward is: **OCI images managed declaratively** (Nix, image-based OSTree, or `systemd-confext`) + **CDI for GPU device injection** + **xdg-desktop-portal for display/media mediation** — a combination that would deliver AppImage's portability, Flatpak's security model, Nix's reproducibility, and Snap's background update model without requiring any single packaging format to win.

---

## 11. Packaging Best Practices

### 10.1 Minimal Manifest for a GPU-Accelerated Wayland Application

```yaml
id: org.example.MyGLApp
runtime: org.freedesktop.Platform
runtime-version: '24.08'
sdk: org.freedesktop.Sdk
command: myglapp

finish-args:
  # Wayland display socket
  - --socket=wayland
  # X11 fallback when Wayland is not available
  - --socket=fallback-x11
  # SHM for X11 (required when --socket=x11 or fallback-x11 is set)
  - --share=ipc
  # GPU access (render nodes + /dev/dri/card* visible but not DRM master)
  - --device=dri
  # Network (omit if the application does not need it)
  - --share=network

modules:
  - name: myglapp
    buildsystem: meson
    sources:
      - type: archive
        url: https://example.org/myglapp-1.0.tar.xz
        sha256: abc123...
```

The `--socket=fallback-x11` flag is preferred over `--socket=x11` for Wayland-native applications: it grants X11 access only when `$WAYLAND_DISPLAY` is unset, pushing users toward Wayland while preserving XWayland compatibility.

### 10.2 Adding PipeWire and Portal Access

For applications needing screen capture, camera, or PipeWire audio:

```yaml
finish-args:
  - --socket=wayland
  - --socket=fallback-x11
  - --share=ipc
  - --device=dri
  # PipeWire socket for audio/video streaming
  - --filesystem=xdg-run/pipewire-0
  # D-Bus access for portal communication
  - --talk-name=org.freedesktop.portal.Desktop
  - --talk-name=org.freedesktop.Notifications
```

The `--filesystem=xdg-run/pipewire-0` grant allows the application to open the PipeWire UNIX socket at `$XDG_RUNTIME_DIR/pipewire-0`. The D-Bus `--talk-name` grants allow calling portal methods. Note that portal permissions (camera, screen capture) are still mediated by xdg-desktop-portal's permission store — granting D-Bus access to the portal hub does not bypass the user consent dialogs.

### 10.3 Electron/Zypak Configuration

For Electron applications, use `org.electronjs.Electron2.BaseApp` as the base application and include Zypak:

```yaml
id: com.example.MyElectronApp
runtime: org.freedesktop.Platform
runtime-version: '24.08'
sdk: org.freedesktop.Sdk
base: org.electronjs.Electron2.BaseApp
base-version: '24-08'
command: zypak-wrapper.sh my-electron-app

finish-args:
  - --socket=wayland
  - --socket=fallback-x11
  - --share=ipc
  - --share=network
  - --device=dri
  - --filesystem=xdg-run/pipewire-0
  - --talk-name=org.freedesktop.portal.Desktop

modules:
  - name: my-electron-app
    buildsystem: simple
    build-commands:
      - install -Dm755 my-electron-app.sh /app/bin/my-electron-app.sh
    sources:
      - type: archive
        url: https://example.org/my-electron-app-linux.tar.gz
        sha256: def456...
```

The `base: org.electronjs.Electron2.BaseApp` provides the Electron binary and Zypak. The `zypak-wrapper.sh` command handles sandbox redirection automatically.

### 10.4 AMD ROCm / GPU Compute

Applications requiring AMD GPU compute should use `--device=dri` (Flatpak 1.18+). This now automatically includes `/dev/kfd` access without requiring `--device=all`:

```yaml
finish-args:
  - --device=dri   # includes /dev/kfd since Flatpak 1.18
  - --socket=wayland
```

For OpenCL vendor configuration files (`.icd` files in `/etc/OpenCL/vendors/`), these are provided by the GL extension via the `merge-dirs=OpenCL/vendors` field in the extension point definition.

### 10.5 Debugging GPU Issues

```bash
# Enable Mesa verbose logging
flatpak run --env=LIBGL_DEBUG=verbose org.example.App

# Check which Mesa driver was selected
flatpak run --env=MESA_LOADER_DRIVER_OVERRIDE=radeonsi --device=dri org.example.App

# Dump Flatpak GL driver selection
FLATPAK_GL_VERBOSE=1 flatpak run org.example.App

# Test with software rendering to isolate GPU driver issues
flatpak run --env=LIBGL_ALWAYS_SOFTWARE=1 org.example.App

# Check extension installation status
flatpak list --runtime | grep Platform.GL
flatpak info --show-extensions org.freedesktop.Platform//24.08

# Enter sandbox shell for debugging
flatpak run --command=bash --device=dri org.example.App
# Inside the sandbox:
ls /dev/dri/
ls /usr/lib/x86_64-linux-gnu/GL/
vulkaninfo 2>&1 | head -30
vainfo
```

### 10.6 Flathub Submission Guidelines

Flathub (the primary Flatpak application repository) requires:

- FLOSS license for applications in the main repository (proprietary apps use the `flathub-beta` or verified publisher tracks)
- AppStream metadata (`MetaInfo.xml`) with application description, screenshots, and developer info
- Manifest linted with `flatpak-builder-lint` [Source](https://docs.flathub.org/docs/for-app-authors/linter)
- No use of `--device=all` without justification (reviewers flag this)
- `--filesystem=host` and `--filesystem=home` are discouraged; specific `xdg-*` paths are preferred
- Automated CI builds on Flathub's infrastructure validate that the application builds from source

GPU-intensive applications (games, 3D modellers, video editors) routinely use `--device=dri` and are accepted on Flathub without special review; this permission is on the "standard" list. Requests for `--device=all` or access to raw kernel interfaces trigger manual review.

---

## Roadmap

### Near-term (6–12 months)

- **Virtio-GPU / Venus vtest transport for driver-agnostic GPU access**: Sebastian Wick's January 2026 proposal describes using Mesa's Venus Vulkan driver over a `vtest` Unix-socket transport to `virgl_test_server` on the host, eliminating the hard version-coupling between the Flatpak runtime's Mesa and the host kernel driver. Flatpak would auto-start `virgl_test_server` when the runtime's native driver is unavailable. [Source](https://blog.sebastianwick.net/posts/flatpak-graphics-drivers/)
- **Backwards-compatible permission extensions**: A merged patch allows manifests to declare "new, more restricting permissions while not breaking compatibility when the app runs on older Flatpak versions," enabling finer-grained GPU and input device access (gamepads, USB portals) without breaking existing installs. [Source](https://blog.sebastianwick.net/posts/flatpak-happenings/)
- **xdg-desktop-portal 1.21+ incremental hardening**: The 1.21 release (January 2026) added a `ConfigureShortcuts` method to the Global Shortcuts Portal and Linyaps app format support; follow-on releases are expected to expand the Settings Portal's accessibility signals and printer capabilities. [Source](https://www.phoronix.com/news/XDG-Desktop-Portal-1.21)
- **bubblewrap overlay mounts**: Bubblewrap gained `--overlay`, `--tmp-overlay`, `--ro-overlay`, and `--overlay-src` options, enabling layered filesystem views inside the sandbox — relevant for compositing runtime + GL extension paths without separate bind-mounts. [Source](https://www.suse.com/support/update/announcement/2025/suse-ru-20250145-1/)
- **Flathub automated CI for GPU manifests**: Flathub's build infrastructure is expanding automated validation of `--device=dri` and VA-API paths, reducing the lag between upstream Mesa releases and extension availability on Flathub. Note: needs verification for exact timeline.

### Medium-term (1–3 years)

- **GPU hardware access daemon**: The graphics virtualisation proposal explicitly plans a dedicated daemon to manage device permissions more granularly — moving away from the coarse `--device=dri` blanket grant to dynamic, per-capability GPU access control. This would allow sandboxed apps to acquire GPU compute access (e.g., `/dev/kfd` for ROCm) on demand rather than at manifest-declaration time. [Source](https://blog.sebastianwick.net/posts/flatpak-graphics-drivers/)
- **Network sandboxing via pasta and PipeWire policies**: Flatpak's roadmap includes network isolation using `pasta` (unprivileged userspace networking) and PipeWire-enforced policies for sandboxed audio/video streams, complementing GPU isolation with tighter I/O boundaries. [Source](https://blog.sebastianwick.net/posts/flatpak-happenings/)
- **systemd-appd and nested sandboxing**: The planned `systemd-appd` service would track running Flatpak instances and enable properly-nested sandboxes — removing the requirement for a D-Bus proxy process and allowing Electron/CEF sub-processes (currently handled by Zypak) to participate in the same permission-store lifecycle as the parent. [Source](https://blog.sebastianwick.net/posts/flatpak-happenings/)
- **XDG Intents portal**: A new portal for deep-linking and thumbnail generation inside sandboxed apps, enabling file-manager thumbnailing of media formats (including GPU-decoded video frames via VA-API) without granting `--filesystem=host`. Note: needs verification for exact specification status.
- **Varlink as D-Bus alternative for portal IPC**: Active exploratory work exists to make varlink a production-ready alternative to D-Bus for portal communication, which would simplify the permission-store interface and reduce latency for high-frequency portal calls (e.g., repeated ScreenCast frame requests). [Source](https://blog.sebastianwick.net/posts/flatpak-happenings/)

### Long-term

- **Full GPU virtualisation without host driver bleed**: The logical endpoint of the Venus/virglrenderer approach is a Flatpak sandbox that never loads host userspace GPU driver code at all — the Mesa inside the runtime is the sole userspace driver, and it communicates to the host GPU exclusively through a stable, versioned virtio-gpu protocol. This would make the GL extension mechanism obsolete and remove the NVIDIA strict-ABI problem entirely. [Source](https://blog.sebastianwick.net/posts/flatpak-graphics-drivers/)
- **Declarative GPU capability requirements in manifests**: Future manifest syntax may allow applications to declare minimum Vulkan or OpenGL feature levels, letting the Flatpak installer select the appropriate runtime + driver combination and warn users on under-specified hardware at install time rather than at runtime. Note: needs verification.
- **Unified portal for GPU context sharing with compositors**: Long-term convergence of the ScreenCast, Camera, and a hypothetical GPU-compute portal under a single `org.freedesktop.portal.GPU` interface — analogous to how Android's `SurfaceFlinger` exposes a single surface-allocation channel — has been discussed but not formally proposed. Note: needs verification.
- **AppImage convergence on portal model**: As AppImage gains optional portal support (via `$APPIMAGE_EXTRACT_AND_RUN` and libfuse3 sandboxing), pressure is growing for convergence on the xdg-desktop-portal permission model rather than relying on ambient host library access — bringing AppImage's GPU access story closer to Flatpak's. Note: needs verification for timeline.

---

## 12. Integrations

This chapter connects to the following chapters in the book:

**Ch6 — EGL**: The EGL ICD (`libEGL_mesa.so`) is part of the GL extension bundle, mounted alongside DRI drivers. The GLVND dispatcher inside the sandbox resolves to the extension's `glvnd/egl_vendor.d/` configuration. EGL's `eglGetPlatformDisplayEXT` with `EGL_PLATFORM_WAYLAND_KHR` works inside Flatpak when `--socket=wayland` is granted and the GL extension provides `libEGL_mesa.so`.

**Ch24 — Vulkan for Application Developers**: The Vulkan loader's ICD JSON manifest discovery mechanics are the same inside and outside Flatpak; the sandbox difference is that `/usr/share/vulkan/icd.d/` inside the sandbox belongs to the runtime + GL extension, not the host. Ch24's description of RADV/ANV/NVK manifest files applies directly; this chapter explains how those manifests arrive in the sandbox via the GL extension `merge-dirs`.

**Ch38 — PipeWire**: The ScreenCast and Camera portal interfaces described in Section 6 of this chapter return PipeWire file descriptors. Ch38 covers the full PipeWire node graph, buffer negotiation, and DMA-BUF flow that follows once the sandbox has the PipeWire fd. For screen capture in sandboxed apps, both chapters are required reading.

**Ch39 — GTK and Qt Rendering**: GTK4 applications inside Flatpak use `org.freedesktop.portal.FileChooser` for file dialogs automatically when `$FLATPAK_ID` is set. Qt 6 similarly routes through portals via `QFileDialog`. The GL extension provides the GPU driver that GTK4's NGL renderer or Qt's Vulkan backend loads.

**Ch50 — Firefox Flatpak (cross-ref)**: Firefox's Flatpak packaging is one of the most complete examples of the GL extension + VA-API + ffmpeg-full pattern in production. The VA-API decode case study in Section 7 of this chapter covers Firefox specifically.

**Ch96 — libcamera**: The Camera portal (`org.freedesktop.portal.Camera`) bridges the Flatpak security boundary and passes a PipeWire fd that connects to a libcamera-backed source node managed by WirePlumber. A sandboxed application using libcamera's API indirectly does so via this portal chain, not by opening `/dev/video*` directly.

**Ch107 — Headless Rendering**: When `--device=dri` is absent from the manifest, Mesa falls back to software rasterisation via **Lavapipe** (if the GL extension is installed) or **llvmpipe**/**softpipe** (if compiled into the runtime's Mesa). Ch107 covers the headless and software rendering paths in detail; this chapter explains the sandbox conditions that trigger them unintentionally.

---

## 13. References

- Flatpak Documentation — Sandbox Permissions: [docs.flatpak.org/en/latest/sandbox-permissions.html](https://docs.flatpak.org/en/latest/sandbox-permissions.html)
- Flatpak Documentation — Extensions: [docs.flatpak.org/en/latest/extension.html](https://docs.flatpak.org/en/latest/extension.html)
- Flatpak Documentation — Manifests: [docs.flatpak.org/en/latest/manifests.html](https://docs.flatpak.org/en/latest/manifests.html)
- Bubblewrap (bwrap) source: [github.com/containers/bubblewrap](https://github.com/containers/bubblewrap)
- Flatpak security model (DeepWiki): [deepwiki.com/flatpak/flatpak/5-security](https://deepwiki.com/flatpak/flatpak/5-security)
- Flatpak 1.18 release — AMD /dev/kfd support: [linuxiac.com/flatpak-1-18-released-with-amd-compute-interface-support/](https://linuxiac.com/flatpak-1-18-released-with-amd-compute-interface-support/)
- TingPing blog — Using host NVIDIA driver with Flatpak: [blog.tingping.se/2018/08/26/flatpak-host-extensions.html](https://blog.tingping.se/2018/08/26/flatpak-host-extensions.html)
- Igalia blog — Digging further into Flatpak with NVIDIA: [blogs.igalia.com/vjaquez/digging-further-into-flatpak-with-nvidia/](https://blogs.igalia.com/vjaquez/digging-further-into-flatpak-with-nvidia/)
- xdg-desktop-portal documentation: [flatpak.github.io/xdg-desktop-portal/docs/](https://flatpak.github.io/xdg-desktop-portal/docs/)
- xdg-desktop-portal permission store: [github.com/flatpak/xdg-desktop-portal/wiki/The-Permission-Store](https://github.com/flatpak/xdg-desktop-portal/wiki/The-Permission-Store)
- Zypak — Flatpak Chromium sandbox: [github.com/refi64/zypak](https://github.com/refi64/zypak)
- Flathub Linter: [docs.flathub.org/docs/for-app-authors/linter](https://docs.flathub.org/docs/for-app-authors/linter)
- Firefox Flatpak hardware acceleration: [discourse.flathub.org/t/how-to-enable-video-hardware-acceleration-on-flatpak-firefox/3125](https://discourse.flathub.org/t/how-to-enable-video-hardware-acceleration-on-flatpak-firefox/3125)
- Snap graphics-core24 interface: [discourse.ubuntu.com/t/the-graphics-core20-snap-interface/23000](https://discourse.ubuntu.com/t/the-graphics-core20-snap-interface/23000)
- Flatpak GitHub — /dev/kfd feature request (resolved in 1.18): [github.com/flatpak/flatpak/issues/5383](https://github.com/flatpak/flatpak/issues/5383)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
