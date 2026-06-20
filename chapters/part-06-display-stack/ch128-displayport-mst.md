# Chapter 128: DisplayPort MST and Multi-Monitor Topology

This chapter targets **kernel display driver developers** implementing or debugging DisplayPort Multi-Stream Transport (MST) support, **system integrators** connecting multiple monitors through docking stations or daisy-chained displays, **docking station engineers** needing to understand how Linux enumerates and manages MST branch devices, and **embedded display engineers** who must configure bandwidth allocation across multiple streams in constrained-bandwidth environments. It assumes familiarity with the KMS atomic modesetting framework (Chapter 2) and the DRM connector model.

---

## Table of Contents

1. [Introduction: What MST Solves](#1-introduction-what-mst-solves)
2. [DisplayPort Physical and Link Layer](#2-displayport-physical-and-link-layer)
3. [SST vs MST Architecture](#3-sst-vs-mst-architecture)
4. [Kernel drm_dp_mst_topology](#4-kernel-drm_dp_mst_topology)
5. [USB-C and DP Alt Mode](#5-usb-c-and-dp-alt-mode)
6. [Docking Stations: Linux Support Landscape](#6-docking-stations-linux-support-landscape)
7. [Bandwidth Allocation and DSC](#7-bandwidth-allocation-and-dsc)
8. [Kernel Debugging MST](#8-kernel-debugging-mst)
9. [Userspace and Compositor Interaction](#9-userspace-and-compositor-interaction)
10. [Real-World Multi-Monitor Setup](#10-real-world-multi-monitor-setup)
11. [USB4 and Thunderbolt Display Connectivity](#11-usb4-and-thunderbolt-display-connectivity)
12. [Integrations](#12-integrations)

---

## 1. Introduction: What MST Solves

A laptop with a single DisplayPort output—or a desktop GPU with four ports already occupied—faces an immediate constraint: one cable, one display. DisplayPort Multi-Stream Transport (MST), introduced in DP 1.2 (2010) and extended through DP 2.1, dissolves that constraint. It multiplexes multiple independent video streams onto a single DP link by dividing the link's time slots among virtual channels (VCs), one per downstream display. A daisy-chainable monitor, a USB-C docking station, or a dedicated MST hub all appear to the GPU as a tree of branch devices, each with one or more sink ports.

On Linux the entire MST machinery lives in `drivers/gpu/drm/display/drm_dp_mst_topology.c` [Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/display/drm_dp_mst_topology.c) and its header `include/drm/display/drm_dp_mst_helper.h` [Source](https://github.com/torvalds/linux/blob/master/include/drm/display/drm_dp_mst_helper.h). This subsystem:

- **Enumerates** the branch/port tree using the DP sideband message protocol.
- **Exposes** each discovered sink as a KMS connector (`DP-1-1`, `DP-1-2`, etc.).
- **Allocates** VC payload time slots across streams via the KMS atomic state.
- **Handles hotplug** by processing upstream request messages from branch devices.

This chapter walks through the complete stack: from DP electrical layers and sideband protocol, through the kernel topology manager and its atomic integration, through USB-C Alt Mode and Thunderbolt docking, to real-world bandwidth budgeting and debugging.

---

## 2. DisplayPort Physical and Link Layer

### 2.1 Main Link

DisplayPort carries video on a differential, source-synchronous main link of 1, 2, or 4 lane pairs. The lane count combined with the per-lane symbol rate (each symbol = 10 bits at 8b/10b or 128b/132b at UHBR) determines total bandwidth:

| Generation | Symbol rate/lane | Coding | Net Gbps/lane | 4-lane total |
|---|---|---|---|---|
| HBR (DP 1.1) | 2.7 Gbd | 8b/10b | 2.16 Gbps | 8.64 Gbps |
| HBR2 (DP 1.2) | 5.4 Gbd | 8b/10b | 4.32 Gbps | 17.28 Gbps |
| HBR3 (DP 1.4) | 8.1 Gbd | 8b/10b | 6.48 Gbps | 25.92 Gbps |
| UHBR10 (DP 2.0) | 10 Gbd | 128b/132b | 9.70 Gbps | 38.69 Gbps |
| UHBR13.5 (DP 2.0) | 13.5 Gbd | 128b/132b | 13.11 Gbps | 52.22 Gbps |
| UHBR20 (DP 2.1) | 20 Gbd | 128b/132b | 19.42 Gbps | 77.37 Gbps |

The VESA DP 2.1 specification [Source](https://www.vesa.org/vesa-standards/standards/displayport/) adds UHBR rates and mandates 128b/132b encoding (channel coding efficiency ≈ 96.97% vs 80% for 8b/10b), nearly doubling effective throughput relative to HBR3 at the same baud rate.

### 2.2 Link Training

Before any stream can be transmitted, the source and sink negotiate a working link rate and lane count through a two-phase link training procedure using the AUX channel:

1. **Clock recovery (CR phase):** source sends a training pattern; sink adjusts its clock recovery loop and reports lock status via DPCD register `DP_LANE0_1_STATUS` (address `0x0202`).
2. **Channel equalisation (EQ phase):** source and sink negotiate pre-emphasis and voltage swing levels, then verify symbol lock and inter-lane alignment.

The maximum trained link rate is stored in DPCD `DP_MAX_LINK_RATE` (`0x0001`); the actual trained rate after negotiation lands in `DP_LINK_BW_SET` (`0x0100`) and `DP_LANE_COUNT_SET` (`0x0101`).

### 2.3 AUX Channel

The bidirectional 1 Mbps AUX channel (or 720 Mbps in DP 2.0 UHBR mode) multiplexes two functions on a single differential pair:

- **DPCD access:** reads and writes to the DisplayPort Configuration Data register map beginning at address `0x00000`. Common registers include `DP_DPCD_REV` (`0x0000`), `DP_MSTM_CAP` (`0x0021`), and `DP_DEVICE_SERVICE_IRQ_VECTOR_ESI0` (`0x2003`).
- **MST sideband messaging:** framed messages exchange topology control commands between source and branch/sink devices (see Section 3).

AUX transfers in the kernel go through `struct drm_dp_aux` and its `.transfer` callback, which drivers implement to invoke their hardware AUX engine. The AMDGPU implementation (`amdgpu_dm_mst_types.c`) calls `dc_link_aux_transfer_raw()` [Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_mst_types.c).

### 2.4 DSC — Display Stream Compression

When the required resolution or refresh rate exceeds available link bandwidth, Display Stream Compression (DSC, VESA DSC 1.1/1.2) reduces the data rate. DSC is a visually lossless or near-lossless block-based codec operating at a configurable bits-per-pixel (bpp) target, typically 8–24 bpp. DSC 1.2 adds support for YCbCr 4:2:0 and deeper colour, enabling 8K@60Hz over HBR3 4-lane links. The kernel DSC helper lives in `drivers/gpu/drm/display/drm_dp_dsc_helper.c`.

---

## 3. SST vs MST Architecture

### 3.1 Single-Stream Transport (SST)

In SST mode, one DP source port drives exactly one display. The entire link bandwidth—minus protocol overhead—belongs to the single video stream. SST is the default and only mode on DP 1.1 hardware. A sink is SST-only when `DP_DPCD_REV` < `0x12` or when `DP_MST_CAP` is clear in `DP_MSTM_CAP`.

The kernel reads this with `drm_dp_read_mst_cap()`:

```c
/* include/drm/display/drm_dp_mst_helper.h */
enum drm_dp_mst_mode {
    DRM_DP_SST,                 /* no MST, no sideband messaging */
    DRM_DP_MST,                 /* full MST, multiple streams */
    DRM_DP_SST_SIDEBAND_MSG,    /* single stream but sideband capable */
};

enum drm_dp_mst_mode drm_dp_read_mst_cap(struct drm_dp_aux *aux,
    const u8 dpcd[DP_RECEIVER_CAP_SIZE]);
```

### 3.2 MST Topology Model

MST introduces a tree topology:

```
GPU (DP source)
 └── DP cable → MST hub (branch device, root)
                 ├── port 0 → Monitor A (sink)
                 ├── port 1 → Daisy-chain monitor (branch + sink)
                 │             └── port 0 → Monitor B (sink)
                 └── port 2 → Monitor C (sink)
```

Each **branch device** (hub, daisy-chain capable monitor, Thunderbolt dock) manages one or more ports. Each **sink** is a display. The tree depth is limited to 7 hops by the DP 1.2 specification (limited by the Relative Address field width).

### 3.3 Sideband Addressing: LCT, LCR, and RAD

Every sideband message carries a routing header that tells each branch device how to forward it toward the target. Three fields control routing:

- **LCT (Link Count Total):** total number of links on the path from the source to the target device. The root branch sits at LCT=1; the first downstream branch at LCT=2; and so on.
- **LCR (Link Count Remaining):** decremented at each hop. When a branch receives a message with LCR=1, it is the intended target. When LCR > 1 the branch forwards the message downstream on the port indicated by the RAD nibble for that hop.
- **RAD (Relative Address):** up to 8 bytes encoding the port number at each hop. Each byte encodes two nibbles: the more-significant nibble covers the odd-numbered hop and the less-significant nibble the even-numbered hop. For the root branch, RAD bytes are all zero (unused).

This encoding means that to address a second-level branch (e.g., a hub downstream of another hub) the source sets LCT=3, LCR=2, and places the root-branch port number in the first RAD nibble and the second-level port number in the second RAD nibble. The `drm_dp_mst_branch.rad[]` array stores the Relative Address for each discovered branch device.

In the kernel struct:

```c
struct drm_dp_mst_branch {
    u8 rad[8];  /* RAD nibbles packed MSB-first at each level */
    u8 lct;     /* Link Count Total to reach this branch */
    /* ... */
};
```

The sideband message header struct mirrors this:

```c
struct drm_dp_sideband_msg_hdr {
    u8 lct;
    u8 lcr;
    u8 rad[8];
    bool broadcast;
    bool path_msg;  /* true for payload-table messages (broadcast) */
    u8 msg_len;
    bool somt;      /* Start of Message Transaction */
    bool eomt;      /* End of Message Transaction */
    bool seqno;     /* sequence number (alternates 0/1) */
};
```

### 3.3 Time Slots and Virtual Channels

The DP main link is divided into time slots. Every 650 μs the link transmits one **Multi-stream Transport Packet (MTP)** containing 64 payload slots, each carrying 54 bytes of payload data. A given video stream (VC payload) occupies one or more consecutive slots per MTP; the GPU assigns slot ranges by sending `ALLOCATE_PAYLOAD` sideband messages.

Bandwidth per stream = time_slots × 54 bytes × (MTP_rate) × 8 bits

The **Payload Bandwidth Number (PBN)** is the kernel's unit for expressing stream bandwidth requirements. `drm_dp_calc_pbn_mode(clock_khz, bpp)` converts a mode's pixel clock and colour depth into PBN:

```c
/* From drivers/gpu/drm/display/drm_dp_mst_topology.c */
int drm_dp_calc_pbn_mode(int clock, int bpp);
```

The VC payload bandwidth per time slot per MTP is computed by `drm_dp_get_vc_payload_bw()`:

```c
/* Returns bandwidth per timeslot in fixed-point 20.12 format */
fixed20_12 drm_dp_get_vc_payload_bw(int link_rate, int link_lane_count);
```

Internally it applies the channel coding efficiency factor and divides by the 8b/10b reference rate (5400 Mbaud, i.e., HBR2 per lane) scaled to 12 fractional bits. UHBR rates use 128b/132b efficiency (~96.97%).

### 3.4 Sideband Message Protocol

All MST control traffic flows over the AUX channel using a framed sideband protocol. Messages are routed to branch devices using the LCT/LCR/RAD address fields described above. Key message types:

| Message | Direction | Purpose |
|---|---|---|
| `LINK_ADDRESS` | Down | Ask a branch device for its port list and GUIDs |
| `ENUM_PATH_RESOURCES` | Down | Query available PBN on a given path |
| `ALLOCATE_PAYLOAD` | Down | Assign VC payload slots to a stream |
| `CLEAR_PAYLOAD_ID_TABLE` | Down | Reset all payload assignments (after error or hotplug) |
| `REMOTE_DPCD_READ/WRITE` | Down | Access DPCD registers of a downstream device |
| `REMOTE_I2C_READ` | Down | Read I2C (EDID) from a downstream device |
| `CONNECTION_STATUS_NOTIFY` | Up | Branch reports port connect/disconnect |
| `RESOURCE_STATUS_NOTIFY` | Up | Branch reports PBN changes |

The kernel encodes and decodes these in `drm_dp_mst_topology.c`. The set of request type strings is defined as:

```c
static const char * const req_type_str[] = {
    [DP_LINK_ADDRESS]           = "LINK_ADDRESS",
    [DP_ENUM_PATH_RESOURCES]    = "ENUM_PATH_RESOURCES",
    [DP_ALLOCATE_PAYLOAD]       = "ALLOCATE_PAYLOAD",
    [DP_CLEAR_PAYLOAD_ID_TABLE] = "CLEAR_PAYLOAD_ID_TABLE",
    [DP_REMOTE_DPCD_READ]       = "REMOTE_DPCD_READ",
    [DP_REMOTE_DPCD_WRITE]      = "REMOTE_DPCD_WRITE",
    [DP_REMOTE_I2C_READ]        = "REMOTE_I2C_READ",
    /* ... */
};
```

---

## 4. Kernel drm_dp_mst_topology

### 4.1 Core Data Structures

The MST subsystem defines three primary object types, each with dual reference counts: a `topology_kref` tracking lifetime in the enumerated tree, and a `malloc_kref` protecting the memory allocation.

**`struct drm_dp_mst_port`** represents one port on a branch device [Source](https://github.com/torvalds/linux/blob/master/include/drm/display/drm_dp_mst_helper.h):

```c
struct drm_dp_mst_port {
    struct kref topology_kref;
    struct kref malloc_kref;

    u8 port_num;        /* port number on parent branch */
    bool input;         /* true if this is an input port */
    bool mcs;           /* message capability status */
    bool ddps;          /* DisplayPort Device Plug Status */
    u8 pdt;             /* Peer Device Type */
    u8 dpcd_rev;        /* DPCD revision of attached device */
    u8 num_sdp_streams;
    uint16_t full_pbn;  /* maximum PBN available on this port */

    struct drm_dp_mst_branch *mstb; /* downstream branch, if any */
    struct drm_dp_aux aux;          /* I2C-over-AUX bus for this port */
    struct drm_dp_aux *passthrough_aux; /* for DSC pass-through */
    struct drm_dp_mst_branch *parent;
    struct drm_connector *connector; /* KMS connector for this sink */
    struct drm_dp_mst_topology_mgr *mgr;
    const struct drm_edid *cached_edid; /* tiling EDID cache */
    bool fec_capable;   /* FEC capable up to this point */
};
```

**`struct drm_dp_mst_branch`** represents a branch device (hub, daisy-chain monitor):

```c
struct drm_dp_mst_branch {
    struct kref topology_kref;
    struct kref malloc_kref;

    u8 rad[8];      /* Relative Address (path to this branch) */
    u8 lct;         /* Link Count Total */
    int num_ports;

    struct list_head ports; /* list of drm_dp_mst_port children */
    struct drm_dp_mst_port *port_parent; /* parent port, NULL if root */
    struct drm_dp_mst_topology_mgr *mgr;
    bool link_address_sent;
    guid_t guid;    /* unique identifier for this branch */
};
```

**`struct drm_dp_mst_topology_mgr`** is the top-level manager, one per MST-capable DP connector on the GPU:

```c
struct drm_dp_mst_topology_mgr {
    struct drm_private_obj base;    /* atomic private object */
    struct drm_device *dev;
    const struct drm_dp_mst_topology_cbs *cbs; /* connector add/destroy callbacks */
    int max_dpcd_transaction_bytes;
    struct drm_dp_aux *aux;         /* AUX channel for root DP port */
    int max_payloads;               /* GPU's maximum concurrent payloads */
    int conn_base_id;               /* root KMS connector ID for path naming */

    struct drm_dp_sideband_msg_rx up_req_recv;
    struct drm_dp_sideband_msg_rx down_rep_recv;

    struct mutex lock;              /* protects mst_state, mst_primary */
    struct mutex probe_lock;        /* serialises topology discovery work */
    bool mst_state : 1;
    bool payload_id_table_cleared : 1;

    u8 payload_count;               /* active payloads in hardware */
    u8 next_start_slot;             /* next slot to assign */

    struct drm_dp_mst_branch *mst_primary; /* root branch device */
    u8 dpcd[DP_RECEIVER_CAP_SIZE];  /* cached root DPCD */

    struct work_struct work;         /* topology probe work */
    struct work_struct tx_work;      /* sideband TX work */
    struct work_struct up_req_work;  /* upstream request processing */
    /* ... delayed destroy lists, workqueue, etc. */
};
```

### 4.2 Initialisation

A driver initialises the topology manager during `drm_connector_init()` for the root DP connector:

```c
int drm_dp_mst_topology_mgr_init(
    struct drm_dp_mst_topology_mgr *mgr,
    struct drm_device *dev,
    struct drm_dp_aux *aux,
    int max_dpcd_transaction_bytes,
    int max_payloads,
    int conn_base_id);
```

`max_payloads` is the GPU's hardware limit on simultaneous VC payloads (typically 63 on modern GPUs). `conn_base_id` is used to construct the KMS connector path property (`DP-1`, `DP-1-1`, etc.).

To activate MST mode after link training confirms the sink is MST-capable:

```c
int drm_dp_mst_topology_mgr_set_mst(struct drm_dp_mst_topology_mgr *mgr,
                                     bool mst_state);
```

Setting `mst_state = true` clears the payload ID table (`CLEAR_PAYLOAD_ID_TABLE` sideband message to the root branch), then queues `mgr->work` to begin topology enumeration.

### 4.3 Topology Enumeration

The work item `drm_dp_mst_link_probe_work()` calls `drm_dp_send_link_address()` on the root branch:

```c
/* Internal: sends LINK_ADDRESS down-request to mstb */
static int drm_dp_send_link_address(struct drm_dp_mst_topology_mgr *mgr,
                                    struct drm_dp_mst_branch *mstb);
```

The reply (`struct drm_dp_link_address_ack_reply`) contains the port list: each port reports its `peer_device_type` (indicating whether it is another branch or a sink), DPCD revision, SDP stream counts, and a GUID. For each port hosting a downstream branch, the enumeration recurses. For sink ports, the topology manager invokes the driver callback `cbs->add_connector()` to create a KMS connector.

This recursive, depth-first discovery populates the linked list `mst_primary->ports` and, for each discovered branch, its `ports` list in turn.

### 4.4 Hotplug Handling

When a monitor is connected or disconnected from a branch port, the branch device sends a `CONNECTION_STATUS_NOTIFY` upstream request. The kernel interrupt handler (driven by HPD) calls:

```c
int drm_dp_mst_hpd_irq_handle_event(
    struct drm_dp_mst_topology_mgr *mgr,
    const u8 *esi,         /* ESI register bytes read from DPCD */
    u8 *ack,               /* ESI bits to acknowledge */
    bool *handled);        /* set true if MST handled this IRQ */

void drm_dp_mst_hpd_irq_send_new_request(
    struct drm_dp_mst_topology_mgr *mgr);
```

`drm_dp_mst_hpd_irq_handle_event()` parses the Event Status Indicator (ESI) registers and queues upstream requests to `mgr->up_req_list`. The `up_req_work` work item then processes these, updating the branch/port tree and calling `drm_kms_helper_hotplug_event()` to notify userspace.

### 4.5 Atomic Payload Management

MST payload allocation integrates with KMS atomic state through `struct drm_dp_mst_topology_state` and `struct drm_dp_mst_atomic_payload`:

```c
struct drm_dp_mst_atomic_payload {
    struct drm_dp_mst_port *port;
    s8  vc_start_slot;   /* assigned at commit time, not check time */
    u8  vcpi;            /* Virtual Channel Payload Identifier */
    int time_slots;      /* timeslots allocated on the DP link */
    int pbn;             /* payload bandwidth number */
    bool delete : 1;     /* true if removing this payload */
    bool dsc_enabled : 1;
    enum drm_dp_mst_payload_allocation payload_allocation_status;
    struct list_head next;
};

struct drm_dp_mst_topology_state {
    struct drm_private_state base;
    struct drm_dp_mst_topology_mgr *mgr;
    u32 pending_crtc_mask;       /* CRTCs touched by this state */
    struct drm_crtc_commit **commit_deps;
    size_t num_commit_deps;
    u32 payload_mask;            /* bitmask of allocated VCPIs */
    struct list_head payloads;   /* list of drm_dp_mst_atomic_payload */
    u8 total_avail_slots;        /* 63 or 64 depending on link */
    u8 start_slot;               /* first usable slot (0 or 1) */
    fixed20_12 pbn_div;          /* PBN divisor, set by driver */
};
```

The atomic check flow for an MST encoder:

1. **Encoder `atomic_check`:** calls `drm_dp_atomic_find_time_slots(state, mgr, port, pbn)` to reserve slots in the new topology state.
2. **`drm_dp_mst_atomic_check(state)`:** validates that total PBN across all payloads does not exceed path bandwidth. Returns `-ENOSPC` if insufficient.
3. **`drm_dp_mst_atomic_setup_commit(state)`:** builds the `commit_deps` list so async commits to the same topology serialise correctly.

The commit sequence uses a two-phase payload add to handle the ACT (Allocation Change Trigger) handshake. This two-phase design exists because all branch devices along the path must latch the new payload table at exactly the same MTP boundary — a strict timing requirement that cannot be met if the software writes to each device individually. The ACT mechanism solves this:

**Phase 1 — Sideband negotiation:** Send `ALLOCATE_PAYLOAD` sideband messages down the tree to each branch device, informing them of the new VC payload assignment (VCPI, slot count, PBN). Each branch stores the pending allocation but does not yet switch to it.

```c
/* Phase 1: reserve slot range; send ALLOCATE_PAYLOAD sideband msg to root branch */
int drm_dp_add_payload_part1(
    struct drm_dp_mst_topology_mgr *mgr,
    struct drm_dp_mst_topology_state *mst_state,
    struct drm_dp_mst_atomic_payload *payload);
```

`drm_dp_add_payload_part1()` assigns a start slot (`payload->vc_start_slot`) from `mgr->next_start_slot`, increments `mgr->next_start_slot` by the slot count, and calls `drm_dp_send_allocate_payload()` which sends the `ALLOCATE_PAYLOAD` message routed via the branch's RAD.

**ACT pulse:** The GPU writes `DP_PAYLOAD_ACT_SET` into the DPCD register `DP_PAYLOAD_ALLOCATIONS_UPDATE` (`0x02C0`) on the root branch. This causes all branch devices to simultaneously latch the pending payload table at the next MTP boundary, guaranteeing atomic slot reassignment across the entire tree. The root branch device then sets `DP_PAYLOAD_ACT_HANDLED` in `DP_PAYLOAD_TABLE_UPDATE_STATUS` (`0x02C1`).

```c
/* Poll for ACT_HANDLED bit after pulsing ACT register */
int drm_dp_check_act_status(struct drm_dp_mst_topology_mgr *mgr);
```

`drm_dp_check_act_status()` reads DPCD `0x02C1` in a polling loop (with microsecond delays) until `DP_PAYLOAD_ACT_HANDLED` is set, confirming all branch devices have committed the new payload table.

**Phase 2 — Hardware programming:** With the branch payload table now committed, the GPU can safely program its own internal payload table register to begin transmitting video data in the assigned slots.

```c
/* Phase 2: wait for sink ACT confirmation, then program GPU hardware payload table */
int drm_dp_add_payload_part2(
    struct drm_dp_mst_topology_mgr *mgr,
    struct drm_dp_mst_atomic_payload *payload);
```

The removal sequence mirrors this:

```c
void drm_dp_remove_payload_part1(struct drm_dp_mst_topology_mgr *mgr,
                                  struct drm_dp_mst_topology_state *mst_state,
                                  struct drm_dp_mst_atomic_payload *payload);
void drm_dp_remove_payload_part2(struct drm_dp_mst_topology_mgr *mgr,
                                  struct drm_dp_mst_topology_state *mst_state,
                                  const struct drm_dp_mst_atomic_payload *old_payload,
                                  struct drm_dp_mst_atomic_payload *new_payload);
```

Part 1 sends `ALLOCATE_PAYLOAD` with `pbn=0` to free the slots; part 2 again pulses ACT and waits for `ACT_HANDLED`, then frees the GPU's payload table entry. The `new_payload` parameter in part 2 allows slot reassignment: freed slots can be reallocated to another stream in the same atomic commit.

### 4.6 Drivers Using MST

- **`amdgpu_dm`**: `drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_mst_types.c` — most mature MST path, supports DSC over MST.
- **`i915`**: `drivers/gpu/drm/i915/display/intel_dp_mst.c` — extensive platform-specific quirks for Tiger Lake and newer.
- **`nouveau`**: `drivers/gpu/drm/nouveau/nouveau_dp.c` — NVK (Rust NVIDIA driver) will eventually replace this.
- **`vmwgfx`**: virtual DP MST for VMware SVGA.

---

## 5. USB-C and DP Alt Mode

### 5.1 USB-C Connector and Lane Assignment

A USB Type-C connector carries up to four SuperSpeed lane pairs (TX1, RX1, TX2, RX2), plus a USB 2.0 pair, CC lines, and power. DisplayPort Alternate Mode (DP Alt Mode) reassigns some or all SuperSpeed lanes to carry DP differential signals:

- **2-lane DP Alt Mode**: lanes TX1/RX1 carry DP (2 DP lanes) + lanes TX2/RX2 carry USB 3.x (up to USB 3.2 Gen 2×1).
- **4-lane DP Alt Mode**: all four lanes carry DP, USB 3.x is suspended (only USB 2.0 remains).

The physical mux between USB and DP is implemented in a **re-timer/mux IC** (e.g., Synaptics VMM9434, Parade PS186, TI TCA9535 combo chips). The kernel driver for DP Alt Mode negotiation is `drivers/usb/typec/altmodes/displayport.c` [Source](https://github.com/torvalds/linux/blob/master/drivers/usb/typec/altmodes/displayport.c). It registers a `typec_altmode_driver` and calls into the connector manager to advertise DP Alt Mode capabilities via VDM (Vendor Defined Message) on the CC line.

### 5.2 DP Alt Mode 2.1

DP Alt Mode 2.1 (standardised alongside DP 2.1 in 2022) enables UHBR over USB-C. The additional cable configuration parameters—UHBR13.5 support, cable type (active optical, active electrical, passive), and DPAM version reporting—are negotiated during USB Power Delivery VDM exchanges. Intel contributed patches in 2023 to add DP Alt Mode 2.1 support to the kernel's TypeC driver [Source](https://lkml.iu.edu/hypermail/linux/kernel/2308.3/07136.html). The key addition is the cable identification flow required to advertise UHBR support to the DisplayPort source.

```c
/* drivers/usb/typec/altmodes/displayport.c (simplified) */
static void dp_altmode_work(struct work_struct *work)
{
    struct dp_altmode *dp = container_of(work, struct dp_altmode, work);
    /* Sends DP_CMD_CONFIGURE VDM to set pin assignment (C, D, E, F) */
    typec_altmode_vdm(dp->alt, dp->header,
                      &dp->data.conf, 1);
}
```

### 5.3 USB4 and Thunderbolt 3/4

Thunderbolt 3 and 4 (TB3/TB4) tunnel PCIe and DisplayPort over a 40 Gbps USB-C connection. USB4 is the open-standard successor to TB3 and is backward compatible. From the kernel's perspective:

- The **`thunderbolt`** driver (`drivers/thunderbolt/`) manages device authorisation, PCIe tunnelling, and DP tunnelling.
- **`boltctl`** and the **`bolt`** daemon handle userspace device authorisation for Thunderbolt security levels.
- A Thunderbolt docking station exposes itself as a PCIe device hosting a USB hub plus a **DP MST hub**, which the GPU then treats as a standard DP MST topology.

DP 2.1 over USB4 Gen 4 supports up to 80 Gbps of raw bandwidth (two 40 Gbps tunnels), sufficient for multiple 8K@60Hz MST streams. The GPU's KMS connector for a Thunderbolt DP output uses `DRM_MODE_CONNECTOR_DisplayPort`; the branch topology enumeration proceeds identically to a wired DP MST hub.

### 5.4 CEC Over AUX

HDMI-to-DP adapters can tunnel CEC (Consumer Electronics Control) over the AUX channel, implemented in `drivers/gpu/drm/display/drm_dp_cec.c`. This is separate from MST but uses the same AUX infrastructure.

---

## 6. Docking Stations: Linux Support Landscape

### 6.1 Common Chipsets

| Chipset | Manufacturer | Notes |
|---|---|---|
| VMM9434 | Synaptics (formerly Parade) | 4-port MST hub, USB-C re-timer |
| PS186 | Parade Tech | 4K@60Hz, HDCP 2.2, DP 1.4 |
| TCA9535 | Texas Instruments | I2C expander companion, GPIO |
| IT6564 | ITE Tech | 4-lane DP 1.4 MST hub |
| VMM9510 | Synaptics | DP 2.0/2.1 capable |

These chipsets appear to the kernel's DRM MST layer as standard DP branch devices — there is no `synaptics-mst` or `parade-mst` kernel driver per se; the generic `drm_dp_mst_topology.c` handles the sideband protocol regardless of the hub ASIC.

### 6.2 Dock Hotplug Sequence

When a laptop is connected to a dock:

1. USB-C PD negotiation → DP Alt Mode activated → lanes routed to GPU DP input.
2. GPU HPD pin asserts → interrupt → `drm_dp_mst_hpd_irq_handle_event()`.
3. ESI registers indicate MST topology change → `LINK_ADDRESS` probe queued.
4. Topology enumeration discovers branch ports → KMS connectors created via `cbs->add_connector()`.
5. `drm_kms_helper_hotplug_event()` → udev event → compositor picks up new connectors.
6. EDID read via `drm_dp_mst_edid_read()` (uses `REMOTE_I2C_READ` sideband) → modes populated.

For ACPI-based hotplug on traditional desktop GPUs, an ACPI notifier calls `drm_kms_helper_hotplug_event()` directly. The MST re-enumeration occurs within `drm_dp_mst_topology_mgr_set_mst()` → `drm_dp_mst_link_probe_work()`.

### 6.3 Common Compatibility Issues

**AUX timeout:** Some cheaper docks ship with buggy firmware that does not respond to `LINK_ADDRESS` within the DP-specified 400 μs AUX reply window, causing `drm_dp_dpcd_read()` to return `-EIO`. The symptom is:

```
[drm:drm_dp_send_link_address] *ERROR* failed to send link address
```

**Payload allocation failure:** Docks that misreport `full_pbn` in `ENUM_PATH_RESOURCES` replies cause `drm_dp_mst_atomic_check()` to incorrectly estimate available bandwidth, leading to `-ENOSPC` at modesetting time even when physical bandwidth exists.

**DSC negotiation failure:** On some docks, the branch device reports DSC capability in its port DPCD but the actual ASIC firmware rejects DSC-compressed payloads. The fix is either a dock firmware update or forcing `drm.dp_dsc=0` on the kernel command line.

### 6.4 Sysfs Inspection

```bash
# List MST connectors (branch port connectors appear as DP-1-1, DP-1-2, etc.)
ls /sys/class/drm/ | grep DP

# Check connector status
cat /sys/class/drm/card0-DP-1/status
cat /sys/class/drm/card0-DP-1-1/status
cat /sys/class/drm/card0-DP-1-2/status

# Inspect EDID
cat /sys/class/drm/card0-DP-1-1/edid | edid-decode

# Read supported modes
cat /sys/class/drm/card0-DP-1-1/modes
```

### 6.5 Userspace Configuration

```bash
# Configure individual MST monitors with xrandr
xrandr --output DP-1-1 --mode 2560x1440 --rate 144
xrandr --output DP-1-2 --mode 1920x1080 --rate 60 --right-of DP-1-1

# Wayland: wlr-randr (wlroots compositors)
wlr-randr --output DP-1-1 --mode 2560x1440@144Hz

# KDE: kscreen-doctor
kscreen-doctor output.DP-1-1.mode.2560x1440@144
```

---

## 7. Bandwidth Allocation and DSC

### 7.1 Path Resource Enumeration

Before committing a modeset, the kernel queries available bandwidth on each path using the `ENUM_PATH_RESOURCES` sideband message, which returns `full_pbn` (the path's maximum PBN capacity) and `avail_pbn` (currently unallocated PBN). Internally this is `drm_dp_send_enum_path_resources()`. The result is stored in `drm_dp_mst_port.full_pbn`.

### 7.2 PBN Calculation and Time Slot Allocation

The kernel's PBN unit is defined by `drm_dp_calc_pbn_mode()`, which normalises the stream bandwidth against a reference clock and adds a 0.6% peak overhead factor (`1006/1000`). The formula, taken directly from `drivers/gpu/drm/display/drm_dp_mst_topology.c`, computes:

```
PBN = ceil(pixel_clock_kHz × bpp × 1006 / (8 × 54 × 1000 × 1000))
```

For a 2560×1440@144 Hz mode at 24 bpp, using a pixel clock of approximately 580,000 kHz:

```c
/* pixel_clock in kHz, bpp = bits per pixel (e.g. 24 for 8bpc RGB) */
int pbn = drm_dp_calc_pbn_mode(580000, 24);
/* Result: 33 PBN */
```

The VC payload bandwidth per time slot per MTP is obtained from `drm_dp_get_vc_payload_bw()`, which returns a `fixed20_12` value (`pbn_div`) representing how many PBN units one timeslot can carry on the given link:

```c
/* link_rate in 10 kbps units; link_lane_count = 1, 2, or 4 */
fixed20_12 pbn_div = drm_dp_get_vc_payload_bw(810000, 4); /* HBR3 4-lane */
/* Returns 60.0 in fixed20_12 format (60 PBN per timeslot per MTP) */
```

The number of timeslots required for the stream is then:

```c
int req_slots = DIV_ROUND_UP(pbn, drm_fixp2int(pbn_div));
/* For pbn=33, pbn_div=60: req_slots = 1 */
```

To understand the total link capacity: at HBR3 × 4 lanes, 63 usable slots × 60 PBN/slot = 3780 PBN total. Since 1 PBN corresponds to approximately 6.86 Mbps of raw data bandwidth in this normalisation:

- 2560×1440@144@24bpp: **33 PBN** (≈ 13.9 Gbps pixel data after overhead factor)
- 1920×1080@60@24bpp: **9 PBN** (≈ 3.77 Gbps)
- 3840×2160@60@24bpp: **34 PBN** (≈ 14.3 Gbps)

The total PBN for all three streams is 76, well within the 3780 PBN capacity of an HBR3 4-lane link. The `drm_dp_mst_atomic_check()` function enforces this bound and rejects atomic commits that exceed it.

A practical cross-check using raw bandwidth (pixel_clock × bpp / 8 × 10/8 for 8b/10b encoding):

| Stream | Pixel clock | Raw link BW |
|---|---|---|
| 2560×1440@144 | 580 MHz | ≈ 17.4 Gbps |
| 1920×1080@60 | 148.5 MHz | ≈ 4.5 Gbps |
| 3840×2160@60 | 594 MHz | ≈ 17.8 Gbps |
| **Total** | | **≈ 39.7 Gbps** |

The HBR3 4-lane line rate is 32.4 Gbps, so all three streams together would exceed HBR3. In practice the GPU's `drm_dp_mst_atomic_check()` rejects this configuration and the user must either reduce refresh rates or enable DSC on the highest-bandwidth stream.

### 7.3 Atomic Bandwidth Check

The encoder's `atomic_check` callback calls:

```c
int drm_dp_atomic_find_time_slots(
    struct drm_atomic_commit *state,
    struct drm_dp_mst_topology_mgr *mgr,
    struct drm_dp_mst_port *port,
    int pbn);
```

This adds or updates a `drm_dp_mst_atomic_payload` entry in the topology state. Later, `drm_dp_mst_atomic_check()` sums all payload PBNs and verifies the sum does not exceed the link's `pbn_div × total_avail_slots`. On failure it returns `-ENOSPC`, causing the atomic commit to be rejected before any hardware state changes.

```c
/* Connector atomic_check: release slots if connector is going inactive */
int drm_dp_atomic_release_time_slots(
    struct drm_atomic_commit *state,
    struct drm_dp_mst_topology_mgr *mgr,
    struct drm_dp_mst_port *port);
```

### 7.4 DSC Integration

When `drm_dp_mst_atomic_check()` detects insufficient bandwidth, drivers may call `drm_dp_mst_atomic_enable_dsc()` to mark a payload as DSC-compressed, halving its effective PBN requirement (at a 2:1 compression ratio):

```c
int drm_dp_mst_atomic_enable_dsc(
    struct drm_atomic_commit *state,
    struct drm_dp_mst_port *port,
    int pbn,
    bool enable);
```

DSC capability at the sink port is probed via `drm_dp_dsc_sink_caps_init()` (reading DPCD registers `DP_DSC_SUPPORT` through `DP_DSC_MAX_BITS_PER_PIXEL_LOW` from the MST port's AUX), and the actual DSC configuration (slice count, bits-per-pixel, chunk size) is computed by `drm_dp_compute_dsc_config()` in `drivers/gpu/drm/display/drm_dp_dsc_helper.c`.

The `drm_dp_mst_atomic_payload.dsc_enabled` flag records whether DSC is active for this payload, ensuring the ACT handshake and hardware programming reflect the correct compressed or uncompressed bandwidth.

### 7.5 Slot Count Limits

DP 1.2–1.4 defines 64 total payload slots per MTP, with slot 0 reserved in some configurations (leaving 63 usable). DP 2.0/2.1 with UHBR encoding uses 64 usable slots but a different `pbn_div` reflecting the higher throughput per slot. `drm_dp_mst_update_slots()` sets `total_avail_slots` and `start_slot` based on the link encoding capability reported in `DP_MAIN_LINK_CHANNEL_CODING_SET`:

```c
void drm_dp_mst_update_slots(
    struct drm_dp_mst_topology_state *mst_state,
    uint8_t link_encoding_cap);
```

---

## 8. Kernel Debugging MST

### 8.1 Topology Dump

The function `drm_dp_mst_dump_topology()` writes a human-readable branch/port tree to a `seq_file` (exposed via debugfs):

```c
void drm_dp_mst_dump_topology(struct seq_file *m,
                               struct drm_dp_mst_topology_mgr *mgr);
```

On AMDGPU, the MST topology debugfs file is created by `amdgpu_dm_debugfs.c` at [Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_debugfs.c):

```bash
cat /sys/kernel/debug/dri/0/amdgpu_mst_topology
```

This file is registered with `debugfs_create_file("amdgpu_mst_topology", ...)` which calls `drm_dp_mst_dump_topology()` internally. On Intel i915, MST topology information is available via the dedicated debugfs entry (registered in `intel_display_debugfs.c` at `i915_dp_mst_info`) [Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_display_debugfs.c):

```bash
cat /sys/kernel/debug/dri/0/i915_dp_mst_info
```

This calls `drm_dp_mst_dump_topology()` for each MST-capable DP port found by iterating `intel_dp.mst.mgr`.

### 8.2 Dynamic Debug

Enable verbose AUX and sideband message tracing at runtime:

```bash
# Enable drm dp_mst debug category
echo "module drm +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drm_dp_mst_topology.c +p" > /sys/kernel/debug/dynamic_debug/control

# Or at boot time:
# drm.debug=0x10   (DP: bit 4 = DRM_UT_DP)
```

Kernel dmesg output includes sideband message decodes:

```
[drm:drm_dp_send_link_address] Sending LINK_ADDRESS to mstb rad=[]
[drm:drm_dp_mst_handle_down_rep] received LINK_ADDRESS reply: 3 ports
[drm:drm_dp_send_enum_path_resources] port 0: full_pbn=2520 avail_pbn=2520
```

### 8.3 Common dmesg Patterns

```bash
# Watch for MST events
dmesg -w | grep -i "mst\|dp_aux\|payload\|DPRX\|alloc"

# AUX timeout — usually a firmware bug in hub:
# [drm:drm_dp_dpcd_access] *ERROR* dp_aux i2c transfer timed out

# Payload allocation failure:
# [drm:drm_dp_mst_atomic_check] not enough bandwidth for all MST payloads

# Hotplug loop (common with buggy dock firmware):
# [drm:drm_dp_mst_hpd_irq_handle_event] MST HPD irq event
# (repeated hundreds of times per minute)
```

### 8.4 AMDGPU-Specific

AMDGPU provides several targeted debugfs controls for MST [Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_debugfs.c):

```bash
# Trigger MST re-probe (simulates hotplug) via the HPD-trigger file:
echo 1 > /sys/kernel/debug/dri/0/amdgpu_dm_trigger_hpd_mst

# Visual confirm overlay (marks MST active streams with on-screen debug info):
echo 1 > /sys/kernel/debug/dri/0/amdgpu_dm_visual_confirm

# MST progress status for a specific connector (per-connector debugfs at DP-x/):
cat /sys/kernel/debug/dri/0/DP-1/mst_progress_status

# Dump full MST topology (calls drm_dp_mst_dump_topology):
cat /sys/kernel/debug/dri/0/amdgpu_mst_topology
```

### 8.5 Intel i915-Specific

```bash
# Read MST topology state (verified present in intel_display_debugfs.c):
cat /sys/kernel/debug/dri/0/i915_dp_mst_info

# General display info including MST connector state:
cat /sys/kernel/debug/dri/0/i915_display_info

# Dynamic debug for DP: enable DRM_UT_DP category at runtime
echo "file intel_dp_mst.c +p" > /sys/kernel/debug/dynamic_debug/control
```

### 8.6 Compile-Time Debug Kconfig

`CONFIG_DRM_DEBUG_DP_MST_TOPOLOGY_REFS=y` enables reference-count history for all `drm_dp_mst_port` and `drm_dp_mst_branch` objects, storing stack traces for every `topology_kref` get/put. This is invaluable for tracking use-after-free bugs in topology teardown:

```bash
# With CONFIG_DRM_DEBUG_DP_MST_TOPOLOGY_REFS enabled, on WARN/BUG:
# [drm] DP MST topology ref history for port 0x...
# Entry 0: GET at drm_dp_mst_topology.c:1234
# Entry 1: PUT at drm_dp_mst_topology.c:1456
```

### 8.7 Emergency Reset

If payload state becomes inconsistent after repeated hotplug events, the primary recovery path is to reset the MST manager entirely. Sysfs does not expose a direct "reset MST" knob from userspace; the correct approach is to physically disconnect and reconnect the dock, which triggers a full HPD cycle.

In driver code, the reset is:

```c
/* Internal reset: tears down all payloads, destroys all MST connectors */
drm_dp_mst_topology_mgr_set_mst(mgr, false);
/* Re-enable: clears payload ID table, re-probes topology */
drm_dp_mst_topology_mgr_set_mst(mgr, true);
```

Drivers trigger this sequence on link loss detection (e.g., when `drm_dp_dpcd_read()` fails on the root AUX, indicating the dock has been disconnected mid-session). The call to `drm_dp_mst_topology_mgr_set_mst(mgr, false)` sends `CLEAR_PAYLOAD_ID_TABLE` to the root branch (if it is still reachable), destroys all managed connectors, and resets `mgr->next_start_slot` to its initial value.

On AMDGPU, an HPD trigger can be simulated for testing:

```bash
# Simulate HPD trigger to force re-enumeration (AMDGPU only):
echo 1 > /sys/kernel/debug/dri/0/amdgpu_dm_trigger_hpd_mst
```

### 8.8 The "MST Manager Probe Stuck" Bug Class

A recurring class of bugs involves the `probe_lock` mutex held by `drm_dp_mst_link_probe_work()` while a sideband transaction stalls (e.g., because the hub disconnected mid-probe). The kernel watchdog eventually fires an RCU stall warning. Workarounds include:

- Reducing `MAX_WAIT_TIME` for AUX transactions in the driver.
- Using `drm_dp_mst_topology_queue_probe(mgr)` (added in 6.x) which takes a less aggressive lock path.
- Dock firmware update to fix AUX response timeouts.

---

## 9. Userspace and Compositor Interaction

### 9.1 Connector Naming

KMS names MST connectors using the path from root to port, separated by hyphens:

- `DP-1` — the root DP connector (GPU output)
- `DP-1-1` — port 1 on the root branch (first downstream display)
- `DP-1-2` — port 2 on the root branch (second downstream display)
- `DP-1-1-1` — port 1 on a second-level branch (daisy-chained hub)

The path is stored as a KMS connector property `PATH` (string), alongside the standard `TILE` property for display tiling.

### 9.2 Connector Enumeration

Compositors iterate connectors with `drmModeGetResources()` and `drmModeGetConnector()`. MST connectors appear in the connector list exactly like physical connectors. The compositor must handle connectors appearing and disappearing at runtime (not just at startup) to support dock hotplug:

```c
/* libdrm: enumerate all connectors including MST */
drmModeRes *res = drmModeGetResources(fd);
for (int i = 0; i < res->count_connectors; i++) {
    drmModeConnector *conn = drmModeGetConnector(fd, res->connectors[i]);
    if (conn->connector_type == DRM_MODE_CONNECTOR_DisplayPort &&
        conn->connection == DRM_MODE_CONNECTED) {
        /* Could be SST or MST port — check PATH property for MST */
        drmModePropertyBlobPtr path_blob =
            get_connector_property_blob(fd, conn, "PATH");
        /* "mst:<connector_id>-<branch>-<port>" for MST connectors */
    }
}
```

### 9.3 Atomic Modesetting with MST

The compositor adds CRTC/plane/connector state to an atomic request identically for MST and SST connectors:

```c
drmModeAtomicReqPtr req = drmModeAtomicAlloc();
drmModeAtomicAddProperty(req, connector_id, crtc_id_prop, crtc_id);
drmModeAtomicAddProperty(req, crtc_id, mode_id_prop, blob_id);
drmModeAtomicAddProperty(req, crtc_id, active_prop, 1);
drmModeAtomicCommit(fd, req, DRM_MODE_ATOMIC_ALLOW_MODESET, NULL);
```

The kernel driver's `encoder->atomic_check()` internally calls `drm_dp_atomic_find_time_slots()`, and `drm_dp_mst_atomic_check()` validates bandwidth before the commit proceeds.

### 9.4 EDID Retrieval Over MST

Reading EDID from a downstream MST sink requires routing an I2C read through the sideband channel using `REMOTE_I2C_READ` messages:

```c
/* drm_dp_mst_edid_read: preferred new API */
const struct drm_edid *drm_dp_mst_edid_read(
    struct drm_connector *connector,
    struct drm_dp_mst_topology_mgr *mgr,
    struct drm_dp_mst_port *port);

/* drm_dp_mst_get_edid: legacy API returning raw struct edid * */
struct edid *drm_dp_mst_get_edid(
    struct drm_connector *connector,
    struct drm_dp_mst_topology_mgr *mgr,
    struct drm_dp_mst_port *port);
```

The EDID is **not** cached between hotplug cycles in the MST manager itself (unlike SST connectors). On each `connector->detect()`, the driver must re-read it. However, for tiled displays (`drm_dp_mst_port.cached_edid`), the MST manager does cache the EDID to ensure all tile connectors share consistent EDID data.

### 9.5 Compositor-Specific MST Behaviour

**Mutter (GNOME Shell):** auto-detects MST connectors from the `PATH` property. Monitor layout is stored per-dock using the dock's EDID serial number as a key, so connecting the same dock restores the saved arrangement.

**KWin (KDE Plasma):** implements saved monitor profiles per physical configuration, identified by connector name + EDID. MST connectors participate in the normal KScreen profile persistence.

**wlroots-based compositors (Sway, Hyprland):** expose each MST port as a `wlr_output`. The compositor author manages multi-monitor layout via the `wlr-output-management` protocol.

---

## 10. Real-World Multi-Monitor Setup

### 10.1 Typical Dock + 3-Monitor Configuration

Consider a laptop with a single USB-C port driving three monitors through a Thunderbolt 4 dock:

- `card0-eDP-1`: integrated laptop panel, 1920×1200@60Hz (driven by internal link, not MST)
- `card0-DP-1-1`: dock monitor A, 2560×1440@144Hz
- `card0-DP-1-2`: dock monitor B, 1920×1080@60Hz

Link negotiated: HBR3 × 4 lanes = 32.4 Gbps line rate, 25.92 Gbps effective data BW (after 8b/10b overhead).

Bandwidth budget (raw link bandwidth = pixel_clock × bpp × 10/8 for 8b/10b):

| Stream | Resolution | Refresh | Pixel clock | Link BW (24 bpp) |
|---|---|---|---|---|
| DP-1-1 | 2560×1440 | 144 Hz | ~580 MHz | ~17.4 Gbps |
| DP-1-2 | 1920×1080 | 60 Hz | ~148.5 MHz | ~4.5 Gbps |
| **Total** | | | | **~21.9 Gbps** |

This fits within 25.92 Gbps effective (HBR3 4-lane). Adding a third 4K@60Hz monitor (pixel clock ~594 MHz, link BW ~17.8 Gbps) brings total to ~39.7 Gbps — exceeding the 25.92 Gbps HBR3 budget. The kernel rejects this modeset with `-ENOSPC` from `drm_dp_mst_atomic_check()`.

With DSC enabled on the 4K@60Hz stream at a 2:1 compression ratio, its effective bandwidth requirement drops to ~8.9 Gbps, bringing total to ~30.8 Gbps — still exceeding HBR3. The combination fits only on an UHBR10 link (38.79 Gbps effective). Alternatively, dropping the 4K@60Hz to 4K@30Hz (pixel clock ~297 MHz, ~8.9 Gbps) brings total to ~30.8 Gbps at HBR3 — still over budget, so DSC or reduced resolution is required for three high-resolution streams on HBR3.

### 10.2 Checking Bandwidth Allocation at Runtime

```bash
# On AMDGPU: check current MST topology and payload state
cat /sys/kernel/debug/dri/0/amdgpu_mst_topology

# On Intel: check MST topology and payload state
cat /sys/kernel/debug/dri/0/i915_dp_mst_info

# On Intel: broader display state including CRTC/pipe config
cat /sys/kernel/debug/dri/0/i915_display_info | grep -A 20 "MST\|MST port"
```

### 10.3 Diagnosing "Monitor Not Detected"

```bash
# Step 1: check connector status
cat /sys/class/drm/card0-DP-1/status

# Step 2: look for AUX errors or MST probe failures
dmesg | grep -i "aux\|mst\|link address\|payload"

# Step 3: verify MST capability bit was found in DPCD
dmesg | grep -i "mst cap\|MST_CAP\|mst mode"

# Step 4: check if branch device enumerated (LINK_ADDRESS response)
dmesg | grep -i "drm_dp_send_link_address\|nports\|received LINK_ADDRESS"

# Step 5: force re-probe by simulating HPD (AMDGPU)
echo 1 > /sys/kernel/debug/dri/0/amdgpu_dm_trigger_hpd_mst
# Or on Intel: enable dynamic debug then reconnect dock:
echo "file drm_dp_mst_topology.c +p" > /sys/kernel/debug/dynamic_debug/control
```

### 10.4 "Cannot Allocate All CRTCs" Failure

When the compositor requests more simultaneous monitors than the GPU's CRTC count supports, the atomic commit fails with `EINVAL`. On most modern GPUs:

- Intel Tiger Lake/Alder Lake: 4 CRTCs (pipes).
- AMD Navi: 6 CRTCs (display pipes).
- NVIDIA (nouveau/NVK): varies by GPU generation.

Check available CRTCs:

```bash
modetest -M amdgpu | grep "^Crtcs"
# Or:
drm_info | grep crtc
```

The compositor must deactivate some CRTCs before activating new ones if the CRTC limit is reached.

### 10.5 Monitor Firmware Updates Over DP

Some Dell UltraSharp and LG UltraFine monitors accept firmware updates delivered over the DP AUX channel. The utility `ddcutil` (and display-specific flashing tools) can communicate over the AUX I2C bus exposed by `/dev/i2c-*` to perform such updates, which sometimes resolve MST sideband protocol bugs in monitor firmware.

### 10.6 Example: Full Setup with wlr-randr

```bash
# Query all outputs via wlr-randr
wlr-randr

# Example output:
# eDP-1  "BOE 0x095F" (1920x1200, 60.00 Hz, enabled)
# DP-1-1 "Dell U2722D" (2560x1440, 144.00 Hz, enabled)
# DP-1-2 "LG ULTRAFINE" (1920x1080, 60.00 Hz, disabled)

# Enable all three with layout
wlr-randr \
    --output eDP-1  --mode 1920x1200@60Hz  --pos 0,0 \
    --output DP-1-1 --mode 2560x1440@144Hz --pos 1920,0 \
    --output DP-1-2 --mode 1920x1080@60Hz  --pos 4480,0
```

---

## Roadmap

### Near-term (6–12 months)

- **UHBR MST over Thunderbolt/USB4 tunnels (Intel i915/Xe):** Intel's graphics driver is preparing UHBR DP tunnelling support, enabling DP 2.1 UHBR20 (up to 80 Gbps) through USB4/Thunderbolt 4 tunnels to drive 8K or 4K@240Hz outputs via docking stations. Preliminary prep patches landed ahead of Linux 7.1. [Source](https://www.phoronix.com/news/Intel-Linux-7.1-UHBR-DP-Prep)
- **AMDGPU DP 2.0 MST maturation:** AMDGPU received initial DP 2.0 MST wiring in recent DRM-next cycles; remaining work includes full UHBR13.5/UHBR20 MST slot allocation, link training at 128b/132b for MST topologies, and robust fallback when an MST hub only trains at HBR3. [Source](https://www.phoronix.com/news/AMDGPU-DP-2.0-MST-PR)
- **Qualcomm MSM DP MST controller initialisation:** Qualcomm upstream developers posted a v2 patch series (June 2025) to initialise a per-controller `dp_mst` module in the MSM drm driver, bringing MST support to Snapdragon-based ARM laptops and docking stations. [Source](https://lkml.iu.edu/hypermail/linux/kernel/2506.1/00906.html)
- **RK3576/RK3588 DisplayPort MST enablement:** Rockchip SoC DP controller patches (v3 posted February 2026) add MST support for the RK3576; RK3588 DP MST work is in parallel development, targeting Chromebook-style ARM docks. [Source](https://lkml.iu.edu/hypermail/linux/kernel/2602.0/08130.html)
- **DSC-over-MST reliability fixes (Intel):** Intel has revised MST DSC (Display Stream Compression) support across several patch revisions, addressing slice geometry negotiation failures and `drm_dp_mst_atomic_enable_dsc()` edge cases on Tiger Lake/Alder Lake hardware. [Source](https://www.phoronix.com/news/Intel-DP-MST-DSC-Linux)

### Medium-term (1–3 years)

- **DP 2.1 UHBR MST in the shared `drm_dp_mst_topology` helper:** The shared topology manager currently only models 63 time slots (DP 1.2 allocation table). Extending it to the DP 2.1 64-time-slot model (plus the expanded payload bandwidth table for 128b/132b links) is an architectural rework required before any driver can expose UHBR MST to user space generically. Note: needs verification of current patch status on LKML.
- **USB4 DisplayPort Alt Mode 2.1 sideband routing:** DP Alt Mode 2.1 adds USB4 Gen3×2 lane configuration and UHBR13.5 signalling; the USB Type-C Alt Mode driver (`usb/typec/altmodes/displayport.c`) needs extensions to communicate the cable's UHBR capability to the MST topology manager. A v2 patch set was posted in August 2023; upstreaming remains incomplete. [Source](https://lkml.iu.edu/hypermail/linux/kernel/2308.3/07136.html)
- **NVK (Nouveau Vulkan) MST integration with `drm_dp_mst_topology.c`:** The open-source NVK driver currently delegates MST to the legacy Nouveau code path, which implements its own partial topology manager. Full migration to the shared DRM MST helper—required for atomic DSC, UHBR, and proper vblank fence sequencing—is a medium-term goal tracked in Nouveau maintainer discussions. Note: needs verification of current issue tracker status on freedesktop GitLab.
- **Improved hot-unplug resilience and port re-enumeration:** Current MST sideband handling has known races between topology removal and payload teardown (CVE-2024-57798 addressed one NULL-deref vector). Work is ongoing to harden `drm_dp_mst_handle_up_req()` and related paths against concurrent port removal, using fine-grained reference counting rather than topology-wide locks.
- **`drm_dp_mst` debugfs expansion:** Planned additions to `drm/*/MST/*` debugfs entries (time-slot maps, per-port bandwidth budgets, sideband message traces) to make in-kernel state more observable without requiring custom vendor debug tools.

### Long-term

- **Adaptive-sync (FreeSync/VRR) over MST:** The DisplayPort 1.4a specification allows Adaptive-Sync on MST streams in principle, but the kernel's MST atomic path does not yet propagate VRR enable/disable to individual VC payloads. Long-term, each MST connector's `drm_connector_state.vrr_enabled` must influence the payload's time-slot recalculation on every refresh-rate change, a non-trivial interaction with the two-phase commit model.
- **Panel Self-Refresh (PSR2) on downstream MST sinks:** PSR2 selective update requires direct AUX communication with the sink's ALPM (Autonomous Link Power Management) logic. For MST sinks behind a branch device, AUX messages must be routed through the sideband path, which conflicts with PSR2's latency requirements. Architectural solutions (branch-device-local PSR coordination) are speculative and depend on future VESA specification work.
- **Automated MST topology validation tooling:** Long-term vision from DRM maintainers includes a kernel self-test (`KUnit`) suite that exercises the sideband protocol state machine against a software-emulated branch device, enabling CI regression testing of topology enumeration, hot-plug, and bandwidth allocation without physical hardware.

---

## 11. USB4 and Thunderbolt Display Connectivity

This section targets **kernel driver developers** working on the Thunderbolt/USB4 subsystem and its interaction with DRM, **hardware engineers** integrating USB4 docks with display outputs, and **system integrators** debugging bandwidth contention on multi-device Thunderbolt setups. It builds directly on Section 5 (USB-C and DP Alt Mode) and the bandwidth allocation material in Section 7.

### 11.1 USB4 Architecture

USB4 is the open specification (published by the USB Implementers Forum in 2019, with USB4 Version 2.0 in 2022) that standardises the Thunderbolt 3 protocol as an interoperable, royalty-free standard [Source](https://www.usb.org/sites/default/files/USB4%20Specification_0.zip). The key differentiator from USB 3.x is the **tunneled-protocol model**: instead of dedicating physical lanes to a single protocol, the USB4 fabric multiplexes three protocol tunnels over a shared 40 Gbps (USB4 Gen 2×2) or 80 Gbps (USB4 Gen 3×2) link using a packet-switched architecture.

The three tunneled protocols are:

- **USB 3.2 tunnel**: carries USB SuperSpeed traffic (up to USB 3.2 Gen 2×2, 20 Gbps) over a dedicated tunnel with guaranteed bandwidth. Up to two SuperSpeed tunnels can be active simultaneously.
- **DisplayPort tunnel (DP tunneling)**: wraps a DP 2.1 stream inside USB4 tunnel packets, delivered to a DisplayPort sink downstream of the USB4 fabric. This is fundamentally different from DP Alt Mode (see Section 11.2).
- **PCIe Gen 3 tunnel**: carries PCIe traffic (×4 lanes for TB3, ×8 lanes for TB4, at PCIe Gen 3 speeds) to attach NVMe enclosures, eGPUs, or network adapters.

Bandwidth allocation across these three protocol tunnels is **dynamic and negotiated at connection time** by the USB4 Connection Manager (implemented in firmware — the ICM, see Section 11.4 — or as a Software Connection Manager in the kernel). The total link capacity must be shared: a USB4 Gen 2×2 link at 40 Gbps (after encoding overhead, ~32 Gbps effective) allocates bandwidth in Gbps chunks to competing tunnels. A saturated USB 3.2 Gen 2 transfer coexisting with a 4K@144Hz DP tunnel and a PCIe NVMe device must share those 32 effective Gbps, and the Connection Manager enforces minimum guaranteed bandwidths per protocol to prevent starvation.

USB4 Version 2.0 (2022) doubles throughput to 80 Gbps (USB4 Gen 3×2) using PAM-4 signalling over the same 40 Gbps cable (backward compatible, but only at the higher speed if both ends and the cable support it). Gen 3×2 allows asymmetric bandwidth mode at up to 120 Gbps in one direction (at the cost of halving bandwidth in the other direction) — useful for display-heavy workloads [Source](https://www.usb.org/sites/default/files/USB4_Version_2.0_Specification_20221110.zip).

### 11.2 DP Alt Mode vs. DP Tunneling

These two mechanisms both carry DisplayPort signals over a USB-C connector but are architecturally distinct:

**DP Alt Mode (USB-C, Section 5)**:
- Negotiated via USB Power Delivery (USB PD) Vendor Defined Messages (VDMs) on the CC line.
- The USB-C connector physically re-maps its SuperSpeed lane pairs to carry **raw DP differential signal** — no USB4 fabric, no packetisation, no Thunderbolt required.
- Works with USB 3.x cables (passive or active), as long as they carry the four SuperSpeed lane pairs.
- A 4-lane DP Alt Mode configuration (pin assignment C or E) places all four SuperSpeed pairs on DP duty; USB 3.x is suspended and only USB 2.0 remains active through the cable.
- Latency is essentially zero beyond the PHY: the DP signal enters the cable at one end and exits at the other with propagation delay only. There is no packetisation overhead.
- On Linux, DP Alt Mode negotiation is handled by `drivers/usb/typec/altmodes/displayport.c` [Source](https://github.com/torvalds/linux/blob/master/drivers/usb/typec/altmodes/displayport.c), which sends `DP_CMD_CONFIGURE` VDMs to set the pin assignment.

**DP Tunneling (USB4 / Thunderbolt 3/4)**:
- Requires a USB4 or Thunderbolt 3/4 connection — a Thunderbolt cable or a USB4 Gen 2/3 cable.
- The DP stream is **packetised** by the upstream USB4 router and delivered as a sequence of tunnel packets over the USB4 fabric to a downstream USB4 router, which reassembles the DP stream and drives the display.
- Bandwidth is shared dynamically with concurrent USB 3.2 and PCIe tunnels — the Connection Manager can reclaim DP tunnel bandwidth if it is needed for USB 3.2 bursts, subject to minimum bandwidth guarantees.
- Supports DP 2.1 (including UHBR rates) when both routers implement USB4 v2.0, enabling DisplayPort streams at up to 80 Gbps through a single USB4 cable.
- Adds a small but measurable latency (a few microseconds of buffering and packetisation) compared to DP Alt Mode. For display use, this is below the pixel clock period and effectively invisible.
- On Linux, DP tunneling is managed by the `drivers/thunderbolt/` subsystem in cooperation with the DRM layer (see Section 11.4).

The practical consequence: a laptop with a standard USB-C port (USB 3.2, no Thunderbolt) will use DP Alt Mode for external display; a laptop with a Thunderbolt 3/4/USB4 port will use DP tunneling, because the Thunderbolt/USB4 controller intercepts the connection before the DP Alt Mode VDM exchange can put the lanes into Alt Mode.

### 11.3 Thunderbolt 3, 4, and 5

Intel's Thunderbolt specification sits on top of USB4 as a certification program and capability superset:

| Standard | Bandwidth | PCIe | DP version | Notes |
|---|---|---|---|---|
| **Thunderbolt 3** | 40 Gbps | Gen 3 ×4 | DP 1.2 (HBR2) | First USB-C form factor, Intel proprietary |
| **Thunderbolt 4** | 40 Gbps | Gen 3 ×8 (mandatory) | DP 1.4 HBR3 (mandatory) | USB4 v1.0 compliant; all features mandatory |
| **Thunderbolt 5** | 80 Gbps (120 Gbps asymmetric) | Gen 4 ×4 or ×8 | DP 2.1 UHBR20 | USB4 v2.0; bandwidth boost mode |

TB3 required 40 Gbps but had optional features: PCIe ×4 was mandatory while ×8 was optional. A TB3 dock might therefore expose only ×4 PCIe despite the cable bandwidth supporting more. TB4 tightened the requirements, mandating ×8 PCIe and DP 1.4 HBR3 (8.1 Gbps/lane × 4 lanes = 32.4 Gbps) on all certified products [Source](https://thunderbolttechnology.net/sites/default/files/Thunderbolt4_Spec_R1.pdf).

TB5, announced with Intel Lunar Lake (2024), doubles the raw bandwidth to 80 Gbps using the USB4 Gen 3×2 physical layer. In **bandwidth boost mode**, the asymmetric allocation can reach 120 Gbps downstream and 40 Gbps upstream — ideal for driving two 8K@60Hz displays. TB5 supports DP 2.1 UHBR20 tunneling [Source](https://www.intel.com/content/www/us/en/architecture-and-technology/thunderbolt/thunderbolt-technology-general.html).

**Thunderbolt security and daisy-chaining**: Thunderbolt adds two features beyond bare USB4:

1. **DMA protection via IOMMU**: each Thunderbolt PCIe tunnel connects a downstream device to the host PCIe root complex. Without protection, a malicious dock could DMA arbitrary host memory. Thunderbolt uses the IOMMU (Intel VT-d) to restrict each tunnel to its allocated physical address ranges. The Linux `thunderbolt` driver configures these IOMMU mappings through the kernel's DMA remapping layer when a device is authorised.

2. **Daisy-chaining**: Thunderbolt docks and displays can be daisy-chained up to six devices deep. Each device in the chain contains a Thunderbolt switch (router) with one upstream port and one or more downstream ports. The kernel's domain model represents this as a tree of `tb_switch` objects, mirroring the physical topology.

### 11.4 Linux Kernel Thunderbolt Driver

The Thunderbolt subsystem lives in `drivers/thunderbolt/` [Source](https://github.com/torvalds/linux/tree/master/drivers/thunderbolt). Its primary components are:

**Connection Manager models**: the driver supports two modes:

- **ICM (Internal Connection Manager)**: firmware in the Thunderbolt controller handles device enumeration and tunnel management. The kernel driver (`icm.c`) communicates with the ICM via a mailbox interface, sending commands like `TB_ICM_CMD_APPROVE_DEVICE` and receiving `TB_ICM_EVENT_DEVICE_CONNECTED` notifications [Source](https://github.com/torvalds/linux/blob/master/drivers/thunderbolt/icm.c). The ICM is used on most Intel client platforms (Skylake through Tiger Lake).

- **Software Connection Manager (SCM)**: the kernel manages the Thunderbolt fabric directly, without firmware assistance. Used on newer platforms (Alder Lake and later) and on Apple Silicon Macs. The SCM implements domain enumeration, switch configuration, and tunnel allocation entirely in `tb.c` and friends [Source](https://github.com/torvalds/linux/blob/master/drivers/thunderbolt/tb.c).

**Key data structures** (`drivers/thunderbolt/tb.h`):

```c
/* A Thunderbolt domain — one per Thunderbolt controller */
struct tb_domain {
    struct device dev;
    int index;                    /* controller index (0, 1, ...) */
    const struct tb_cm_ops *cm_ops; /* ICM or SCM ops */
    struct tb *tb;
    struct mutex lock;
    struct notifier_block iommu_nb;
};

/* A Thunderbolt switch (router) — one per physical Thunderbolt device */
struct tb_switch {
    struct device dev;
    struct tb_regs_switch_header config; /* config space header */
    struct tb_port *ports;              /* array of ports */
    struct tb_dma_port *dma_port;       /* DMA port for FW updates */
    struct tb *tb;
    u64 uid;                            /* unique 64-bit identifier */
    uuid_t uuid;
    u16 vendor;
    u16 device;
    int generation;                     /* 1=TB1, 2=TB2/TB3, 3=TB3/TB4, 4=USB4 */
    int tunnel_count;                   /* active tunnels through this switch */
    bool is_unplugged;
    bool rpm;                           /* runtime PM enabled */
    /* ... */
};

/* A Thunderbolt port on a switch */
struct tb_port {
    struct tb_regs_port_header config;
    struct tb_switch *sw;
    struct tb_port *remote;             /* connected port on remote switch */
    struct tb_tunnel *tunnel;           /* active tunnel through this port */
    int port;                           /* port number */
    bool disabled;
    bool bonded;                        /* lane bonded (two ports = one 40G link) */
    /* ... */
};
```

**Tunnel allocation** (`drivers/thunderbolt/tunnel.c`): when the Connection Manager decides to create a DP tunnel to a downstream display, it calls `tb_tunnel_alloc_dp()` [Source](https://github.com/torvalds/linux/blob/master/drivers/thunderbolt/tunnel.c):

```c
struct tb_tunnel *tb_tunnel_alloc_dp(struct tb *tb,
                                     struct tb_port *in,
                                     struct tb_port *out,
                                     int link_nr,
                                     int max_up,
                                     int max_down);
```

Here `in` is the upstream DP adapter port (on the host controller's switch) and `out` is the downstream DP adapter port (on the dock's switch). `max_up` and `max_down` specify the maximum bandwidth in Mbps in each direction. Internally the function allocates a `struct tb_tunnel` and configures the path through the switch fabric using hop configuration registers.

Once the DP tunnel is established, its downstream endpoint presents itself as a standard DisplayPort sink to the GPU. The GPU's AUX transactions to that display travel through the DP tunnel's AUX path (a separate, lower-bandwidth tunnel alongside the main DP link tunnel). The DRM layer then sees a standard DP MST or SST sink and enumerates it using `drm_dp_mst_topology.c` exactly as it would for a wired DP hub.

The Thunderbolt driver notifies the DRM layer of DP bandwidth changes through `drm_dp_update_payload_part1()` — if the USB4 Connection Manager needs to reclaim bandwidth from the DP tunnel (e.g., to service a USB 3.2 burst), it informs the DRM driver, which must reduce the DP link rate or resolution. This bandwidth-change notification path is still evolving in the upstream kernel. Note: needs verification of the exact callback path in current upstream (Linux 6.10+).

### 11.5 USB4 Dock Behavior on Linux

When a USB4 dock with DP output is plugged in, DRM can see it in one of three ways depending on the dock hardware:

**(a) DRM connector via DP tunneling through the USB4/Thunderbolt fabric** (most common for TB3/TB4/USB4 docks): The Thunderbolt driver establishes a DP tunnel; the GPU sees the tunnel endpoint as a native DP sink. The resulting KMS connector is `DRM_MODE_CONNECTOR_DisplayPort`. The path from GPU to display is: GPU DP output → Thunderbolt controller → USB4 fabric → dock switch → DP output → cable → display. Enumeration uses `drm_dp_mst_topology.c` if the dock's DP output drives an MST hub, or SST if it drives a single display.

**(b) HDMI/DP Alt Mode adapter**: cheaper docks use a USB-C-to-HDMI/DP Alt Mode chip (e.g., a passive re-timer or an active converter like the ITE IT6263). In this case, no Thunderbolt fabric is involved; the USB PD VDM exchange configures DP Alt Mode, and the GPU sees the display directly as if it were a native DP or HDMI connector. Alt Mode docks appear to DRM as `DRM_MODE_CONNECTOR_HDMIA` or `DRM_MODE_CONNECTOR_DisplayPort`.

**(c) DisplayLink USB device** (Section 6 compatibility note): some docks include a DisplayLink chip (e.g., DL-6xxx series from Synaptics/DisplayLink) that implements a USB display adapter. The display is driven by the CPU over USB, with the `evdi` kernel module (Chapter 155) creating a virtual DRM device. This path bypasses the GPU's display engine entirely and has much lower performance and higher CPU overhead.

**Hotplug path for case (a)**: when a USB4/Thunderbolt dock is connected, the sequence is:

1. USB-C PD negotiation completes; the Thunderbolt controller asserts Thunderbolt presence on the cable.
2. The kernel Thunderbolt driver enumerates the dock's switch: udev emits `SUBSYSTEM=thunderbolt ACTION=add` for each discovered switch.
3. Depending on security level, the user (or `bolt` daemon) authorises the device: `boltctl enroll <uuid>` or automatic `BOLTD_POLICY=auto` in `bolt.conf`.
4. The Connection Manager creates a DP tunnel: `tb_tunnel_alloc_dp()` configures the path through the switch fabric.
5. The Thunderbolt driver asserts the GPU's DP HPD (Hot-Plug Detect) line via a platform interrupt or a DP AUX HPD pulse.
6. The GPU DRM interrupt handler processes the HPD → `drm_dp_mst_hpd_irq_handle_event()` if the dock presents as MST, or a standard HPD connector update if SST.
7. Topology enumeration, EDID read, KMS connector creation, and udev `SUBSYSTEM=drm ACTION=change` events proceed as in Section 6.2.

### 11.6 Bandwidth Allocation Challenges

A TB4 dock running at 40 Gbps (after encoding: ~32 Gbps effective) must share its bandwidth among all active tunnels. Consider a demanding but realistic configuration:

| Device | Tunnel type | Bandwidth requirement |
|---|---|---|
| 4K@144Hz display (DP 1.4 HBR3) | DP tunnel | ~18 Gbps |
| External USB 3.2 Gen 2×2 SSD | USB 3.2 tunnel | ~20 Gbps peak |
| PCIe NVMe enclosure | PCIe Gen 3 ×4 tunnel | ~16 Gbps peak |

The total peak demand (~54 Gbps) far exceeds the 32 Gbps effective TB4 budget. The Connection Manager handles this through **bandwidth negotiation**: each tunnel type has a minimum guaranteed bandwidth (e.g., DP tunnel minimum is the bandwidth required to sustain the current display mode; USB 3.2 minimum is negotiated with the USB host controller; PCIe has no hard minimum). Excess bandwidth is allocated on a best-effort basis.

In practice, the NVMe drive and USB SSD will never simultaneously saturate their peak rates. But the display DP tunnel **must** sustain its allocated bandwidth continuously or the display will glitch. The kernel enforces this hierarchy: the DP tunnel minimum bandwidth is reserved at tunnel creation and cannot be reclaimed by other protocols without dropping the display to a lower mode.

The Linux USB4/Thunderbolt stack exposes bandwidth allocation state through the `sysfs` interface [Source](https://www.kernel.org/doc/html/latest/driver-api/thunderbolt.html):

```bash
# List all Thunderbolt domains and devices
ls /sys/bus/thunderbolt/devices/

# Check bandwidth allocation for a specific domain (domain0 = first controller)
cat /sys/bus/thunderbolt/devices/domain0/0-0/rx_bandwidth_allocated
cat /sys/bus/thunderbolt/devices/domain0/0-0/tx_bandwidth_allocated

# Check available bandwidth on a link
cat /sys/bus/thunderbolt/devices/domain0/0-1/rx_bandwidth
cat /sys/bus/thunderbolt/devices/domain0/0-1/tx_bandwidth
```

When bandwidth allocation fails — for example, because the DP tunnel cannot negotiate enough bandwidth for the requested display mode — the kernel emits a dmesg warning:

```
[thunderbolt] bandwidth allocation failed for 0-1: not enough bandwidth for DP tunnel
[drm:intel_dp_tunnel_atomic_check_link] *ERROR* Not enough BW for DP tunnel
```

In this case, the DRM atomic commit is rejected with `-EINVAL` or `-ENOSPC`, and the compositor must fall back to a lower resolution or refresh rate. On Intel platforms, `intel_dp_tunnel.c` implements the DRM ↔ Thunderbolt bandwidth negotiation callbacks [Source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_dp_tunnel.c).

For AMD platforms and the generic case, the bandwidth negotiation between the Thunderbolt driver and DRM is handled through a notifier chain: the Thunderbolt driver calls `drm_dp_tunnel_notify_bw_alloc_change()` (or equivalent, depending on kernel version) to signal available bandwidth changes to the DP driver. Note: the exact API surface here is still evolving — check `drivers/thunderbolt/` and `drivers/gpu/drm/display/drm_dp_tunnel.c` for current upstream state.

### 11.7 Debugging Thunderbolt and USB4 Display Issues

A systematic toolkit for diagnosing display connectivity through Thunderbolt/USB4:

**`boltctl`** — the primary userspace tool for Thunderbolt device management:

```bash
# List all Thunderbolt devices and their authorisation state
boltctl list

# Example output:
#  ● Caldigit TS4 Thunderbolt 4 Element Hub
#    ├─ type:          peripheral
#    ├─ name:          Thunderbolt 4 Element Hub
#    ├─ vendor:        CalDigit
#    ├─ uuid:          00000000-0000-0000-0000-<mac>
#    ├─ status:        authorized
#    ├─ authorized:    2026-06-20 12:34:56
#    └─ stored:        yes

# Enroll (permanently authorise) a device by UUID
boltctl enroll --policy auto <uuid>

# Check Thunderbolt security level
boltctl info | grep -i security
# security: user          <- requires manual authorisation each session
# security: secure        <- cryptographic challenge-response required
# security: dponly        <- only DP tunnels allowed, no PCIe
# security: none          <- all devices auto-authorised (insecure)

# Change security level (requires reboot to take effect on most platforms)
# Set via BIOS/UEFI Thunderbolt settings, not boltctl
```

**`dmesg` Thunderbolt tracing**:

```bash
# Monitor Thunderbolt events in real time
dmesg -w | grep -i "thunderbolt\|usb4\|tb_tunnel\|dp_tunnel"

# Common messages:
# thunderbolt 0000:00:0d.2: AMD USB4 v2 router found
# thunderbolt 0-1: new device found, vendor=... device=...
# thunderbolt 0-1: DP tunnel created
# thunderbolt 0-1: DP tunnel bandwidth allocation: 17920 Mb/s
# [drm] Thunderbolt DP tunnel BW allocation failed

# Enable verbose Thunderbolt debug at boot:
# thunderbolt.dyndbg=+pmf (or add to /etc/modprobe.d/thunderbolt.conf)
echo "module thunderbolt +p" > /sys/kernel/debug/dynamic_debug/control
```

**`lspci -v`** — verifying PCIe tunnel is active:

```bash
# PCIe tunnel devices appear as PCI devices on a virtual root port
lspci -tv | grep -A 5 "Thunderbolt\|USB4"

# The NVMe in a TB dock shows up as a real PCIe device:
lspci -v | grep -A 10 "Non-Volatile"
# If the device appears here, the PCIe tunnel is active and the dock is authorised
```

**`xrandr --listproviders`** — confirming the display is driven by the GPU (not DisplayLink):

```bash
xrandr --listproviders
# If only "Provider 0: id: 0x... cap: 0xf, 9 Sources, 0 Sinks" → GPU only (correct)
# If a second provider appears → DisplayLink or software renderer in use

# List all outputs including tunnel-connected displays
xrandr --query | grep -E "^[A-Z]|connected|disconnected"
# DP-1 connected 2560x1440+0+0 (normal left inverted right) 597mm x 336mm
# DP-2 connected 1920x1080+2560+0 ...
```

**Thunderbolt security levels and `boltctl enroll`**: the security level controls when PCIe tunneling is permitted:

- `none`: all devices auto-authorised at plug-in. Suitable for locked-down workstations where physical access implies trust.
- `user`: user must authorise each new device via `boltctl enroll` or the GNOME/KDE system settings Thunderbolt panel. Device UUID is stored in `/var/lib/boltd/`.
- `secure`: adds cryptographic challenge-response using the device's challenge key (a 32-byte value burned into the device's NVM). Protects against device cloning.
- `dponly`: the ICM only creates DP tunnels; PCIe tunneling is blocked at firmware level. Prevents DMA attacks from untrusted docks while still allowing display output.

Note: `dponly` security mode means external NVMe enclosures and eGPUs will not function. Only DP display output works through the Thunderbolt connection.

```bash
# Check current security level
cat /sys/bus/thunderbolt/devices/domain0/security

# Authorise a device stored by boltd (persistent across reboots)
boltctl enroll --policy auto <device-uuid>

# Or using the bolt DBus API (what GNOME Settings uses):
gdbus call --system --dest org.freedesktop.bolt \
    --object-path /org/freedesktop/bolt \
    --method org.freedesktop.bolt.Manager.EnrollDevice \
    "<uuid>" "auto" ""
```

**`thunderbolt-info`**: on some distributions, the `thunderbolt-tools` package provides a `thunderbolt-info` utility that dumps the full switch topology and tunnel state in a human-readable format. It reads directly from `/sys/bus/thunderbolt/devices/` and augments the `boltctl list` output with port-level connectivity information.

**Combined diagnostic workflow** for a display that fails to appear after docking:

```bash
# 1. Is the dock physically enumerated at all?
boltctl list
# If empty: check cable quality, try different TB4 cable, check BIOS TB settings

# 2. Is the device authorised?
boltctl list | grep -i "status:"
# "status: unauthorized" → run: boltctl enroll --policy auto <uuid>

# 3. Did the DP tunnel get created?
dmesg | grep -i "dp tunnel\|bandwidth"

# 4. Did DRM see the HPD?
dmesg | grep -i "HPD\|hotplug\|connector"

# 5. Is the KMS connector present?
ls /sys/class/drm/ | grep DP

# 6. What does the connector report?
cat /sys/class/drm/card0-DP-1/status
cat /sys/class/drm/card0-DP-1/enabled

# 7. Force re-probe (AMDGPU)
echo 1 > /sys/kernel/debug/dri/0/amdgpu_dm_trigger_hpd_mst

# 8. Dump MST topology (if dock is MST-capable)
cat /sys/kernel/debug/dri/0/amdgpu_mst_topology
# or on Intel:
cat /sys/kernel/debug/dri/0/i915_dp_mst_info
```

---

## 12. Integrations

**Chapter 2 — KMS (Kernel Mode Setting):** MST connectors are first-class KMS objects. `drm_dp_mst_topology_mgr` is a `drm_private_obj` participating in the atomic state machine. Atomic commits invoke `drm_dp_mst_atomic_check()` as part of the driver's `.atomic_check()` chain, and the two-phase payload add/remove integrates with `drm_atomic_helper_commit_hw_done()` sequencing.

**Chapter 3 — Advanced Display Features:** DSC (Display Stream Compression) is the primary mechanism for fitting high-resolution streams into MST bandwidth budgets. `drm_dp_mst_atomic_enable_dsc()` marks payloads for DSC, and the DSC helper (`drm_dp_dsc_helper.c`) computes slice geometry and bpp targets. HDCP 2.2 over MST is implemented via the `QUERY_STREAM_ENC_STATUS` sideband message (`drm_dp_send_query_stream_enc_status()`).

**Chapter 74 — HDR and Wide Color Gamut:** HDR metadata (SMPTE ST 2086, CTA-861-H) delivered to MST sinks uses the same KMS connector property mechanism as SST. The `drm_dp_mst_edid_read()` path reads HDR capability from the sink's EDID CEA extension, populated per-connector even in multi-monitor MST topologies.

**Chapter 75 — Explicit GPU Synchronisation:** Multi-display vblank coordination across MST outputs uses timeline fences. `drm_dp_mst_topology_state.commit_deps` holds CRTC commit objects that the MST helper uses in `drm_dp_mst_atomic_wait_for_dependencies()` to ensure payload changes on one CRTC complete before a dependent CRTC's commit proceeds.

**Chapter 101 — Colour Science and ICC Profiles:** Each MST connector has an independent `drm_connector.state.gamma_lut` / `ctm` / `degamma_lut` pipeline. Colour management daemons (colord, ICC.rs) assign per-display ICC profiles based on EDID primaries retrieved independently via `drm_dp_mst_edid_read()` for each MST port.

**Chapters 117 and 122 — DKMS and Out-of-Tree Drivers:** NVIDIA's proprietary driver ships its own MST implementation separate from the kernel's `drm_dp_mst_topology.c`. It manages payload allocation internally and exposes MST connectors through the NVKMS API rather than standard DRM KMS. The open-source NVK driver (Chapter 10) will eventually integrate with `drm_dp_mst_topology.c`, as Nouveau partially does today.

**Chapter 4 — DMA-BUF and Buffer Allocation:** In a multi-head MST configuration, each display CRTC scans out from an independent framebuffer. For zero-copy multi-display scenarios (e.g., a shared desktop spanning three MST monitors), DMA-BUF fences ensure the GPU finishes rendering before any scanout plane updates, coordinated with the per-CRTC vblank interrupts.

---

*Sources and upstream references:*

- *Linux kernel `drm_dp_mst_topology.c`*: [github.com/torvalds/linux](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/display/drm_dp_mst_topology.c)
- *Linux kernel `drm_dp_mst_helper.h`*: [github.com/torvalds/linux](https://github.com/torvalds/linux/blob/master/include/drm/display/drm_dp_mst_helper.h)
- *AMDGPU DM MST types*: [github.com/torvalds/linux](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_mst_types.c)
- *Intel DP Alt Mode 2.1 patch series*: [lkml.iu.edu](https://lkml.iu.edu/hypermail/linux/kernel/2308.3/07136.html)
- *i915 MST payload fix series (Imre Deak)*: [patchwork.kernel.org](https://patchwork.kernel.org/project/intel-gfx/cover/20230125114852.748337-1-imre.deak@intel.com/)
- *VESA DisplayPort 2.1 specification*: [vesa.org](https://www.vesa.org/vesa-standards/standards/displayport/)
- *AMD Xilinx DP TX payload BW management*: [docs.amd.com](https://docs.amd.com/r/en-US/pg199-displayport-tx-subsystem/Payload-Bandwidth-Management)
- *Linux kernel Thunderbolt driver*: [github.com/torvalds/linux/drivers/thunderbolt](https://github.com/torvalds/linux/tree/master/drivers/thunderbolt)
- *Linux kernel Thunderbolt documentation*: [kernel.org](https://www.kernel.org/doc/html/latest/driver-api/thunderbolt.html)
- *Intel DP tunnel driver (i915)*: [github.com/torvalds/linux](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_dp_tunnel.c)
- *USB4 Specification v1.0*: [usb.org](https://www.usb.org/sites/default/files/USB4%20Specification_0.zip)
- *USB4 Version 2.0 Specification*: [usb.org](https://www.usb.org/sites/default/files/USB4_Version_2.0_Specification_20221110.zip)
- *Thunderbolt 4 specification*: [thunderbolttechnology.net](https://thunderbolttechnology.net/sites/default/files/Thunderbolt4_Spec_R1.pdf)
- *Intel Thunderbolt 5 overview*: [intel.com](https://www.intel.com/content/www/us/en/architecture-and-technology/thunderbolt/thunderbolt-technology-general.html)
