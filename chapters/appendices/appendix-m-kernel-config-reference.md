# Appendix M: Kernel Configuration Reference for Graphics

> **Status**: Stub — content to be populated during final editing pass

This appendix lists the key Linux kernel `CONFIG_` options relevant to the graphics stack, with notes on which chapters discuss the features they enable.

---

## M.1 DRM Core

| Config | Description | Chapter |
|---|---|---|
| `CONFIG_DRM` | DRM subsystem core; required for all GPU drivers | Ch1 |
| `CONFIG_DRM_KMS_HELPER` | KMS helper library; required for modesetting | Ch2 |
| `CONFIG_DRM_KMS_CMA_HELPER` | CMA-based framebuffer allocator for simple display drivers | Ch2 |
| `CONFIG_DRM_PANEL` | Panel driver framework (`drm_panel`) | Ch6 |
| `CONFIG_DRM_BRIDGE` | Bridge chain framework (`drm_bridge`) | Ch6 |
| `CONFIG_DRM_DISPLAY_CONNECTOR` | Generic display connector driver | Ch2 |
| `CONFIG_DRM_FBDEV_EMULATION` | Legacy fbdev emulation layer over KMS | Ch2 |

---

## M.2 GPU Drivers

| Config | Description | Chapter |
|---|---|---|
| `CONFIG_DRM_AMDGPU` | AMD GPU driver (GCN+, RDNA, DCN) | Ch5 |
| `CONFIG_DRM_AMDGPU_SI` | Southern Islands (GCN 1.0) support in amdgpu | Ch5 |
| `CONFIG_DRM_AMDGPU_CIK` | Sea Islands (GCN 2.0) support in amdgpu | Ch5 |
| `CONFIG_DRM_I915` | Intel i915 driver (Gen 2 through Gen 12) | Ch5 |
| `CONFIG_DRM_XE` | Intel Xe driver (Gen 12.5+ / Meteor Lake+) | Ch5 |
| `CONFIG_DRM_NOUVEAU` | Nouveau (open-source NVIDIA) driver | Ch5, Ch7-Ch11 |
| `CONFIG_NOUVEAU_PLATFORM_DRIVER` | Nouveau platform driver for Tegra SoCs | Ch6 |
| `CONFIG_DRM_PANFROST` | Panfrost driver (Mali Midgard/Bifrost) | Ch6 |
| `CONFIG_DRM_PANTHOR` | Panthor driver (Mali Valhall CSF) | Ch6 |
| `CONFIG_DRM_LIMA` | Lima driver (Mali-400/450) | Ch6 |
| `CONFIG_DRM_ETNAVIV` | Etnaviv driver (Vivante GC series) | Ch6 |
| `CONFIG_DRM_MSM` | MSM driver (Qualcomm Adreno) | Ch6 |
| `CONFIG_DRM_V3D` | V3D driver (Broadcom VideoCore VI — Raspberry Pi 4+) | Ch6 |
| `CONFIG_DRM_VC4` | VC4 driver (Broadcom VideoCore IV — Raspberry Pi 3) | Ch6 |
| `CONFIG_DRM_VIRTIO_GPU` | VirtIO GPU driver (QEMU/crosvm virtual GPU) | Ch5 |

---

## M.3 Memory Management and Buffer Sharing

| Config | Description | Chapter |
|---|---|---|
| `CONFIG_DMA_SHARED_BUFFER` | DMA-BUF framework | Ch4 |
| `CONFIG_SYNC_FILE` | Sync file / explicit fence framework | Ch3, Ch4 |
| `CONFIG_SW_SYNC` | Software sync timeline (for testing) | Ch4 |
| `CONFIG_DMABUF_HEAPS` | DMA-BUF heap allocator framework | Ch4 |
| `CONFIG_DMABUF_HEAPS_SYSTEM` | System heap for DMA-BUF | Ch4 |
| `CONFIG_DRM_GEM_SHMEM_HELPER` | Shared memory GEM helper (used by many simple drivers) | Ch4 |
| `CONFIG_DRM_TTM` | Translation Table Manager (used by amdgpu, nouveau) | Ch4 |

---

## M.4 IOMMU and Virtualisation

| Config | Description | Chapter |
|---|---|---|
| `CONFIG_IOMMU_SUPPORT` | IOMMU subsystem core | Ch5, App K |
| `CONFIG_AMD_IOMMU` | AMD IOMMU driver | Ch5 |
| `CONFIG_INTEL_IOMMU` | Intel VT-d IOMMU driver | Ch5 |
| `CONFIG_VFIO` | VFIO framework for GPU passthrough | Ch5, App K |
| `CONFIG_VFIO_PCI` | VFIO PCI device driver | App K |
| `CONFIG_HMM_MIRROR` | Heterogeneous Memory Management mirror (AMD SVM/amdgpu_mn) | Ch5 |

---

## M.5 Video and Camera

| Config | Description | Chapter |
|---|---|---|
| `CONFIG_MEDIA_SUPPORT` | V4L2 media framework | Ch26 |
| `CONFIG_VIDEO_V4L2` | V4L2 core | Ch26 |
| `CONFIG_MEDIA_CONTROLLER` | Media Controller API (required for libcamera) | Ch26 |
| `CONFIG_V4L2_MEM2MEM_DEV` | Memory-to-memory video device framework (stateless codecs) | Ch26 |
| `CONFIG_VIDEO_HANTRO` | Hantro VPU driver (stateless H.264/H.265/VP8/VP9) | Ch26 |
| `CONFIG_VIDEO_RASPBERRYPI_HEVC` | Raspberry Pi HEVC codec driver | Ch26 |

---

## M.6 Useful Debug Options

| Config | Description | Notes |
|---|---|---|
| `CONFIG_DRM_DEBUG_SELFTEST` | DRM core self-tests | Use with `make kselftest` |
| `CONFIG_DRM_DP_AUX_CHARDEV` | DisplayPort AUX channel char device | Useful for display debugging |
| `CONFIG_DRM_DEBUG_MM` | GEM/TTM memory manager debug | High overhead; CI use only |
| `CONFIG_LOCKDEP` | Lock dependency validator | Catches GPU scheduler deadlocks |
| `CONFIG_KASAN` | Kernel Address Sanitiser | Driver memory safety testing |
| `CONFIG_KCSAN` | Kernel Concurrency Sanitiser | Race condition detection in DRM |

---

*[Minimum version notes per feature and distro-default status to be added during editing pass.]*

---

## References

- [Kernel GPU documentation](https://www.kernel.org/doc/html/latest/gpu/) — authoritative driver-specific config notes
- [Mesa on Linux — driver prerequisites](https://docs.mesa3d.org/install.html)

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
