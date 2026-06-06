# Appendix K: Remote Display and GPU Virtualisation

> **Status**: Stub — research and writing pending

## Table of Contents

- [Overview](#overview)
- [K.1 RDP: Remote Desktop Protocol on Linux](#k1-rdp-remote-desktop-protocol-on-linux)
- [K.2 VNC and the Wayland Capture Chain](#k2-vnc-and-the-wayland-capture-chain)
- [K.3 PipeWire-Based Remote Desktop](#k3-pipewire-based-remote-desktop)
- [K.4 GPU Virtualisation: VirtIO-GPU, VirGL, and Venus](#k4-gpu-virtualisation-virtio-gpu-virgl-and-venus)
- [K.5 VFIO and GPU Passthrough](#k5-vfio-and-gpu-passthrough)
- [K.6 WSL2 and the dxgkrnl Bridge](#k6-wsl2-and-the-dxgkrnl-bridge)
- [References](#references)

---

## Overview

Remote display on Linux spans two distinct problem domains: *screen capture and encoding* (sending a running desktop session to a remote viewer) and *GPU virtualisation* (giving a virtual machine or container access to GPU resources). Both domains intersect the graphics stack at the compositor and kernel layers. This appendix surveys the principal technologies in each domain, with cross-references to the chapters where the underlying mechanisms — PipeWire, DMA-BUF, VirtIO, KMS — are covered in depth.

*[Detailed content to be written.]*

---

## K.1 RDP: Remote Desktop Protocol on Linux

*[To be written — FreeRDP, GNOME RDP via Mutter, KWin RDP backend, GPU-accelerated encode via VA-API (Ch26), PipeWire capture (Ch38).]*

---

## K.2 VNC and the Wayland Capture Chain

*[To be written — wayvnc, KMS CRTC readback vs. screencopy protocol, DMA-BUF CPU copy overhead.]*

---

## K.3 PipeWire-Based Remote Desktop

*[To be written — xdg-desktop-portal remote-desktop interface, OBS Studio streaming pipeline, DMA-BUF encode handoff.]*

---

## K.4 GPU Virtualisation: VirtIO-GPU, VirGL, and Venus

*[To be written — cross-reference Ch5 virtio-gpu section; VirGL OpenGL passthrough; Venus Vulkan passthrough; Mesa guest-side drivers; QEMU/crosvm integration.]*

---

## K.5 VFIO and GPU Passthrough

*[To be written — VFIO IOMMU groups, VFIO-PCI driver, QEMU GPU passthrough setup, Looking Glass for low-latency frame capture.]*

---

## K.6 WSL2 and the dxgkrnl Bridge

*[To be written — dxgkrnl kernel module, D3D12 compute under WSL2, Mesa's d3d12 Gallium driver, limitation on display output.]*

---

## References

*[To be populated during research phase.]*

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
