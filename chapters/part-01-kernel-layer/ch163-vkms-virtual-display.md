# Chapter 163: VKMS and Virtual Display Drivers for Testing

**Target audiences**: Kernel DRM developers writing tests and CI pipelines for graphics code; compositors and display stack engineers needing a headless virtual display; and CI infrastructure engineers running GPU tests without physical hardware.

---

## Table of Contents

1. [Introduction](#introduction)
2. [VKMS: Virtual Kernel Modesetting Driver](#vkms-virtual-kernel-modesetting-driver)
3. [VKMS Architecture and Capabilities](#vkms-architecture-and-capabilities)
4. [Using VKMS in CI and Testing](#using-vkms-in-ci-and-testing)
5. [VKMS Advanced Features: Writeback and CRC](#vkms-advanced-features-writeback-and-crc)
6. [Other Virtual Display Drivers](#other-virtual-display-drivers)
7. [DRM Leases and VR Direct Mode](#drm-leases-and-vr-direct-mode)
8. [GPU Testing with LLVMpipe and softpipe](#gpu-testing-with-llvmpipe-and-softpipe)
9. [Container and VM Display Stacks](#container-and-vm-display-stacks)
10. [Integrations](#integrations)

---

## Introduction

Testing graphics code without a physical display is essential for CI pipelines. Linux provides several virtual display solutions:

- **VKMS** (Virtual Kernel Modesetting) — a software DRM driver in the kernel that implements full KMS without hardware
- **evdi** — userspace-backed virtual display (Ch155, also used by DisplayLink)
- **virtio-gpu** — paravirtualized GPU for VMs
- **DRM writeback connector** — captures composed output to a buffer
- **LLVMpipe/softpipe** — software Mesa rasterisers for GPU-less rendering

Together, these enable running Wayland compositors, IGT tests, and Vulkan/OpenGL applications in a CI environment with no GPU.

Sources: [VKMS in kernel](https://github.com/torvalds/linux/tree/master/drivers/gpu/drm/vkms) | [IGT GPU tests](https://gitlab.freedesktop.org/drm/igt-gpu-tools)

---

## VKMS: Virtual Kernel Modesetting Driver

### What VKMS Provides

VKMS (`drivers/gpu/drm/vkms/`) is a software-only DRM driver that:
- Creates a virtual `DRM_MODE_CONNECTOR_VIRTUAL` connector
- Implements a full atomic modesetting stack (CRTC, encoder, connector, planes)
- Simulates vertical blank interrupts via hrtimer
- Provides a pixel pipeline: composition of planes → framebuffer output (in software)
- Supports CRC (cyclic redundancy check) for pixel-perfect test assertions

```bash
# Load VKMS:
sudo modprobe vkms
# Check:
ls /dev/dri/  # new card* appears
dmesg | grep -i vkms
# Output: vkms: loaded

# Run Weston on VKMS:
DISPLAY= weston --backend=drm-backend.so --drm-device=card0 &
# or:
weston --backend=headless-backend.so  # weston's own headless (not VKMS)
```

### Module Parameters

```bash
# VKMS module options:
modprobe vkms enable_cursor=1      # enable hardware cursor plane
modprobe vkms enable_overlay=1     # enable overlay plane
modprobe vkms enable_writeback=1   # enable writeback connector

# Or via sysfs:
echo 1 > /sys/module/vkms/parameters/enable_cursor
```

---

## VKMS Architecture and Capabilities

### Source Structure

```
drivers/gpu/drm/vkms/
├── vkms_drv.c        # Module init, drm_driver
├── vkms_crtc.c       # CRTC: vblank simulation, composition
├── vkms_plane.c      # Primary, overlay, cursor planes
├── vkms_output.c     # Connector and encoder
├── vkms_writeback.c  # Writeback connector
├── vkms_composer.c   # Software plane composition (blend)
├── vkms_crc.c        # CRC computation from composed frame
└── vkms_config.c     # Configurable output (resolution, depth)
```

### Vblank Simulation

VKMS uses an hrtimer to simulate the vertical blank interrupt:

```c
/* vkms/vkms_crtc.c */
static enum hrtimer_restart vkms_vblank_simulate(struct hrtimer *timer)
{
    struct vkms_output *output = container_of(timer, struct vkms_output, vblank_hrtimer);
    struct drm_crtc *crtc = &output->crtc;

    /* Advance frame counter: */
    drm_crtc_handle_vblank(crtc);

    /* If writeback enabled: compose and output frame: */
    if (output->wb_pending)
        vkms_composer_worker(output);

    hrtimer_forward_now(timer, output->period_ns);
    return HRTIMER_RESTART;
}
```

### Software Plane Composition

VKMS composites planes in software (no GPU) to produce a framebuffer:

```c
/* vkms/vkms_composer.c */
static void vkms_compose_planes(struct vkms_output *output,
    struct vkms_writeback_job *wb)
{
    struct drm_framebuffer *primary_fb = output->composer_state->primary_fb;
    /* For each plane (primary, overlays, cursor): */
    list_for_each_entry(plane, &planes, head) {
        /* Alpha-blend plane onto accumulation buffer: */
        vkms_blend(wb->data, plane->pixel_data,
                   plane->src_rect, plane->dst_rect,
                   plane->pixel_blend_mode);
    }
    /* Compute CRC over result: */
    crc32(wb->crc, wb->data, wb->size);
}
```

This software composition is intentionally slow (CPU-only) — VKMS is for correctness testing, not performance.

---

## Using VKMS in CI and Testing

### IGT GPU Tests on VKMS

IGT (Intel GPU Tools) is the primary DRM test suite. Many IGT tests run on VKMS:

```bash
# Run IGT on VKMS:
sudo modprobe vkms
sudo igt_runner --test-list kms_atomic kms_plane kms_cursor \
    --device /dev/dri/card0

# Or run specific tests:
sudo kms_atomic --run-subtest plane-primary-overlay-mutable-zpos

# Check which tests support VKMS:
igt_runner --list-tests 2>/dev/null | head -40
```

VKMS passes the majority of KMS-level IGT tests, making it suitable for kernel CI without hardware.

### Wayland Compositor Testing

```bash
# Run a Wayland compositor on VKMS:
sudo modprobe vkms
WLR_BACKENDS=drm WLR_DRM_DEVICES=/dev/dri/card0 sway &

# Run a Wayland client:
WAYLAND_DISPLAY=wayland-1 weston-terminal

# Or use Weston's own test backend:
weston --backend=headless-backend.so --width=1920 --height=1080 &
```

### Rendercheck and Pixel Verification

```bash
# rendercheck uses VKMS CRC for pixel-exact verification:
rendercheck -t blend -f a8r8g8b8 -o /dev/dri/card0

# The CRC check ensures the compositor's output matches
# the expected pixel values — catches rendering regressions
```

---

## VKMS Advanced Features: Writeback and CRC

### Writeback Connector

The writeback connector captures the composed frame to a framebuffer, enabling pixel readback without a physical display:

```c
/* Enable writeback (userspace): */
uint32_t writeback_fb_id = create_framebuffer(drm_fd, 1920, 1080,
    DRM_FORMAT_XRGB8888);

/* Add writeback FB to atomic commit: */
drmModeAtomicAddProperty(req, writeback_connector_id,
    writeback_fb_id_prop, writeback_fb_id);
drmModeAtomicAddProperty(req, writeback_connector_id,
    writeback_pixel_formats_prop, 0);
drmModeAtomicCommit(drm_fd, req, DRM_MODE_ATOMIC_NONBLOCK, NULL);

/* Wait for writeback completion fence: */
int fence_fd;
drmModeAtomicGetProperty(req, writeback_connector_id,
    writeback_out_fence_prop, &fence_fd);
sync_wait(fence_fd, -1);
/* Now writeback_fb contains the composed frame */
```

### CRC Capture

VKMS exposes per-frame CRC values via debugfs:

```bash
# Enable CRC capture:
echo auto > /sys/kernel/debug/dri/0/crtc-0/crc/control

# Read CRC values:
cat /sys/kernel/debug/dri/0/crtc-0/crc/data
# Output: 0x1234 0xabcd 0x5678  (per-frame CRC)

# In IGT tests:
igt_pipe_crc_new(fd, 0, INTEL_PIPE_CRC_SOURCE_AUTO);
igt_pipe_crc_start(pipe_crc);
/* ... render something ... */
igt_pipe_crc_get_current(fd, pipe_crc, &crc);
igt_assert_crc_equal(&crc, &expected_crc);
```

CRC capture is used in IGT tests like `kms_plane --subtest pixel-format` to verify that a plane displays the correct colours.

---

## Other Virtual Display Drivers

### virtio-gpu

`virtio-gpu` is the paravirtualized display driver for QEMU/KVM virtual machines:

```bash
# QEMU with virtio-gpu:
qemu-system-x86_64 \
    -device virtio-gpu-pci \
    -display sdl,gl=on \
    -kernel vmlinuz -initrd initrd.img

# In the guest: /dev/dri/card0 is virtio-gpu
# Supports: virgl (OpenGL via host GPU), 2D display
```

virtio-gpu + virgl passes OpenGL commands to the host GPU, enabling guest VMs to use hardware acceleration without GPU passthrough.

### DRM Dummy Driver

The `drm_dummy` driver (proposed, not merged) creates a minimal DRM device for testing without any display output or pixel composition:

```c
/* Minimal DRM driver for testing: */
static const struct drm_driver dummy_driver = {
    .driver_features = DRIVER_MODESET | DRIVER_ATOMIC,
    .name = "drm-dummy",
};
```

### simpledrm

`simpledrm` (`drivers/gpu/drm/tiny/simpledrm.c`) claims a simple framebuffer from the UEFI boot screen and creates a DRM device over it — useful for booting to a display before GPU drivers load:

```bash
dmesg | grep simpledrm
# Output: simpledrm: bound to /dev/dri/card0 (efifb at ...)
```

---

## DRM Leases and VR Direct Mode

### DRM Lease Concept

A DRM lease allows one DRM master to delegate exclusive access to a subset of DRM resources (CRTCs, connectors, planes) to another process. Used for VR direct mode:

```
SteamVR / OpenXR runtime
  ├─ leases connector 0 (headset) from the desktop compositor
  ├─ runs its own modesetting on the leased connector
  └─ desktop compositor retains control of other connectors
```

### Creating a DRM Lease

```c
/* Lessor (compositor): grant lease to VR runtime: */
uint32_t objects[] = { connector_id, crtc_id, plane_id };
int lessee_fd = drmModeCreateLease(fd, objects, 3, 0, &lease_id);
/* Send lessee_fd to the VR runtime process (e.g. via SCM_RIGHTS) */

/* Lessee (VR runtime): use the leased resources exclusively: */
/* The lessee uses the received fd as if it were the DRM master: */
drmModeSetCrtc(lessee_fd, crtc_id, fb_id, 0, 0, &connector_id, 1, &mode);
/* VR headset renders at 90Hz without compositor involvement */
```

### Lease Lifecycle

```c
/* Compositor: revoke lease: */
drmModeRevokeLease(fd, lease_id);

/* List active leases: */
struct drm_mode_list_lessees list;
drmModeListLessees(fd, &list);
```

OpenXR runtimes (Monado) use DRM leases on Linux for direct-to-HMD rendering, bypassing the desktop compositor entirely for minimum latency.

---

## GPU Testing with LLVMpipe and softpipe

### LLVMpipe

LLVMpipe is Mesa's LLVM-JIT software rasteriser. It provides full OpenGL 4.5 and Vulkan (via lavapipe) on CPU:

```bash
# Force LLVMpipe:
LIBGL_ALWAYS_SOFTWARE=1 glxinfo | grep "OpenGL renderer"
# Output: OpenGL renderer string: llvmpipe (LLVM 17, 256 bits)

# For Vulkan:
VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/lvp_icd.x86_64.json vulkaninfo
# or:
MESA_VK_DEVICE_SELECT_FORCE_DEFAULT_DEVICE=1 \
MESA_LOADER_DRIVER_OVERRIDE=zink \
VK_DRIVER_FILES=/usr/share/vulkan/icd.d/lvp_icd.x86_64.json vulkan_app
```

### lavapipe

lavapipe is Mesa's Vulkan software rasteriser (LLVMpipe + Vulkan frontend):

```bash
# Check lavapipe:
VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/lvp_icd.x86_64.json vulkaninfo --summary
# Shows: Vulkan 1.3 on llvmpipe

# CI use: run Vulkan CTS on lavapipe:
cts-runner --vulkan-cts-path=VK-GL-CTS/build \
    VK_ICD_FILENAMES=.../lvp_icd.x86_64.json \
    --run suite/dEQP-VK.api.*
```

lavapipe supports Vulkan 1.3 + most extensions, making it suitable for CTS conformance testing without a GPU.

---

## Container and VM Display Stacks

### Running Wayland in Docker/Podman

```bash
# Run a Wayland compositor in a container:
docker run --device /dev/dri --device /dev/snd \
    -e XDG_RUNTIME_DIR=/tmp/runtime \
    -v /tmp/runtime:/tmp/runtime \
    ubuntu weston --backend=drm-backend.so

# For software-only (no DRI device needed):
docker run -e LIBGL_ALWAYS_SOFTWARE=1 \
    ubuntu weston --backend=headless-backend.so
```

### X Virtual Framebuffer (Xvfb)

For legacy X11 applications:

```bash
# Start Xvfb:
Xvfb :99 -screen 0 1920x1080x24 &
DISPLAY=:99 some_x11_app
# Capture screenshot:
DISPLAY=:99 import -window root screenshot.png
```

### GPU Passthrough (VFIO)

For VMs needing real GPU performance:

```bash
# Bind GPU to VFIO:
echo "10de 2684" > /sys/bus/pci/drivers/vfio-pci/new_id  # NVIDIA A100
# QEMU command:
qemu-system-x86_64 \
    -device vfio-pci,host=01:00.0 \
    -m 8G -smp 8
# Guest uses the real GPU via VFIO passthrough
```

---

## Integrations

- **Ch01 (DRM Architecture)** — VKMS is a DRM driver implementing the same `drm_driver` and `drm_crtc_helper_funcs` interfaces as real GPU drivers; it serves as a clean reference implementation
- **Ch06 (KMS Atomic)** — VKMS implements the full atomic modesetting path; IGT's `kms_atomic` test suite runs against VKMS to verify atomic commit semantics
- **Ch04 (DRM Planes)** — VKMS implements primary, overlay, and cursor planes; software composition in `vkms_composer.c` demonstrates how planes blend
- **Ch155 (evdi)** — Both VKMS and evdi are virtual DRM drivers; VKMS uses hrtimer for vblank, evdi uses a userspace-driven update model
- **Ch34 (Wayland)** — Weston and sway can run on VKMS for headless CI testing; the display protocol stack is fully exercised without physical hardware
- **Ch125 (RenderDoc)** — RenderDoc captures work from lavapipe (software Vulkan) for shader debugging without a GPU
