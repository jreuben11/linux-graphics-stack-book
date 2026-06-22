# Chapter 187: HDMI CEC and the Linux CEC Subsystem

**Target audiences:** Embedded Linux developers implementing CEC for TV boxes and set-top devices; home theater automation engineers integrating Linux systems with HDMI networks; Raspberry Pi and Amlogic/Rockchip platform developers; kernel CEC subsystem contributors writing hardware drivers; Kodi and media center application developers needing CEC control of displays and AV receivers.

---

## Table of Contents

1. [CEC Protocol Fundamentals](#1-cec-protocol-fundamentals)
2. [Key CEC Message Categories](#2-key-cec-message-categories)
3. [Linux Kernel CEC Subsystem Architecture](#3-linux-kernel-cec-subsystem-architecture)
4. [Hardware CEC Driver Implementations](#4-hardware-cec-driver-implementations)
5. [The CEC Notifier Mechanism](#5-the-cec-notifier-mechanism)
6. [Userspace API — /dev/cecN](#6-userspace-api--devcecn)
7. [cec-ctl and v4l-utils Tooling](#7-cec-ctl-and-v4l-utils-tooling)
8. [libcec — Userspace Abstraction](#8-libcec--userspace-abstraction)
9. [CEC in DRM — Pin Framework and DP CEC Tunnelling](#9-cec-in-drm--pin-framework-and-dp-cec-tunnelling)
10. [Practical Automation and Troubleshooting](#10-practical-automation-and-troubleshooting)
11. [Integrations](#11-integrations)

---

## 1. CEC Protocol Fundamentals

Consumer Electronics Control (CEC) is the HDMI feature that allows interconnected devices — televisions, disc players, AV receivers, set-top boxes — to discover one another and exchange control commands over a single dedicated wire. Despite modern firmware stacks, network protocols, and IR blasters coexisting in home theater systems, CEC remains the only standardised, topology-aware, wire-native control bus on every HDMI cable. Understanding it from first principles is essential before examining the Linux kernel driver model.

### 1.1 Physical Layer

CEC occupies **pin 13** on all Type A, Type C, and Type E HDMI connectors. It is a single-wire, open-drain serial bus pulled to 3.3 V through a line that all devices share. Any device that drives the line low asserts a "dominant" state; releasing the line (allowing it to float to 3.3 V through a pull-up) produces a "recessive" state. This open-drain topology permits multiple initiators to compete for the bus — a form of wired-AND arbitration analogous to classic I²C — but at far lower speeds.

The effective bus speed is approximately **400 baud** (roughly 36 bytes per second peak). That slowness is a deliberate design choice: the bus must reliably traverse up to 10 m of passive HDMI cable, potentially branched through switches, without active repeaters.

[Source: HDMI Specification 1.3, CEC Electrical Specification; see also Linux kernel cec-gpio documentation](https://www.kernel.org/doc/Documentation/devicetree/bindings/media/cec-gpio.txt)

### 1.2 Bit Encoding

CEC uses pulse-width-modulated bit encoding — both logical 0 and logical 1 occupy the same total nominal duration but differ in how long the line is held low.

| Symbol | Low duration | High duration | Total |
|---|---|---|---|
| Start bit | 3.7 ms ± 0.2 ms | 0.8 ms ± 0.2 ms | 4.5 ms |
| Logical 0 | 1.5 ms ± 0.2 ms | 0.9 ms ± 0.2 ms | 2.4 ms |
| Logical 1 | 0.6 ms ± 0.2 ms | 1.8 ms ± 0.2 ms | 2.4 ms |

A receiver samples the line at 1.05 ms after the falling edge to distinguish a 0 from a 1: at that sample point, a logical 0 is still low while a logical 1 has already released to high. The start bit's markedly longer low pulse (3.7 ms) allows all bus participants to identify the beginning of a new frame regardless of what they were doing previously.

[Source: Analog Devices CEC Application Note AN-1049; HDMI 1.3 specification section 6.3]

### 1.3 Frame Structure

A CEC frame consists of:

1. **Start bit** — one start pulse as described above.
2. **Header block** — a single byte whose upper nibble (bits 7:4) is the **initiator** logical address and lower nibble (bits 3:0) is the **destination** logical address.
3. **Data blocks** — zero to fifteen additional bytes (opcode + parameters).
4. **EOM and ACK bits** — each byte is followed by an **End-of-Message (EOM)** bit (1 = last byte, 0 = more bytes follow) and an **ACK** bit. For directed messages, the destination device acknowledges receipt by pulling the line low during the ACK bit window; for broadcast messages (destination = 0xF), the initiator expects no pull-down.

The maximum frame length is 16 bytes (header + 15 data bytes), giving a maximum opcode payload of 14 bytes. An address-only "poll" message — header block with EOM=1 and no data bytes — is used during logical address allocation.

[Source: Linux kernel CEC documentation](https://docs.kernel.org/admin-guide/media/cec.html)

### 1.4 Physical Addresses

The **physical address** encodes the device's topological position in the HDMI tree. It is represented as four hexadecimal nibbles, written in dotted notation: `A.B.C.D`. The root of the tree (the television itself) always holds `0.0.0.0`. A device directly connected to HDMI input port 2 of the TV receives physical address `2.0.0.0`. If that device is an HDMI switch and a player is connected to its port 1, the player receives `2.1.0.0`, and so on up to four levels of nesting.

Physical addresses are **not configured manually by devices** — they are extracted from the **EDID** provided by the upstream display. Specifically, the CTA-861 extension block contains an HDMI Vendor Specific Data Block (HDMI VSDB) identified by the 24-bit HDMI OUI `0x000C03`. Bytes 4 and 5 of that VSDB carry the 16-bit CEC physical address of the source. When a device reads the EDID of the TV (or switch) it is connected to, it learns its own physical address.

An address of `0xFFFF` (written `f.f.f.f` by Linux tools) means the physical address is **unknown** — typically because no HPD (Hot-Plug Detect) signal is present or the EDID cannot be read.

### 1.5 Logical Addresses

The **logical address** is a 4-bit value (0–15) identifying the device type and role. Unlike physical addresses, logical addresses are claimed dynamically by each device at startup through a polling procedure.

| Logical address | Device type |
|---|---|
| 0 | TV |
| 1 | Recording Device 1 |
| 2 | Recording Device 2 |
| 3 | Tuner 1 |
| 4 | Playback Device 1 |
| 5 | Audio System |
| 6 | Tuner 2 |
| 7 | Tuner 3 |
| 8 | Playback Device 2 |
| 9 | Recording Device 3 |
| 10 | Tuner 4 |
| 11 | Playback Device 3 |
| 12 | Reserved |
| 13 | Reserved |
| 14 | Free Use |
| 15 | Broadcast (unregistered) |

**Allocation procedure:** A new device wishing to claim logical address 4 (Playback Device 1) sends a poll message with both source and destination set to 4. If no response is received, the address is free and the device claims it. If another device acknowledges, address 4 is already in use; the device attempts the next address in its type's list. If all candidates are taken, the device falls back to the Unregistered address (15), which cannot be used as a destination.

[Source: grokipedia CEC logical address description; Linux kernel cec-core documentation](https://docs.kernel.org/driver-api/media/cec-core.html)

### 1.6 CEC 1.3a vs CEC 2.0

**CEC 1.3a** (bundled with HDMI 1.3, first publicly published version) defines the fundamental message set: one-touch play, standby, routing control, remote control pass-through, OSD name, and system audio. It is the baseline that virtually all consumer HDMI devices implement.

**CEC 2.0** (introduced with HDMI 2.0, 2013) extends the protocol with dynamic auto lip-sync, audio return channel coordination, and an expanded range of device control commands. It increases the maximum data block count and introduces enhanced device discovery. Linux's CEC subsystem supports the full CEC 2.0 command set while remaining backward-compatible with 1.3a devices.

[Source: HDMI 2.0 specification overview, electronics-notes.com](https://www.electronics-notes.com/articles/audio-video/hdmi/hdmi-versions.php)

---

## 2. Key CEC Message Categories

CEC defines its messages by opcode bytes in the range 0x00–0xFF. The following categories cover the commands most relevant to Linux system integration.

### 2.1 One Touch Play

The One Touch Play feature allows a source device to wake the TV and switch its active input in a single sequence:

- **0x04 — Image View On** (sent to TV, logical address 0): Requests the TV to power up and display a picture. This message **must be the first CEC message sent** when waking a sleeping TV; sending Active Source before Image View On may not wake the display.
- **0x0D — Text View On**: Similar to Image View On but signals that a text overlay (OSD) will follow.
- **0x82 — Active Source** (broadcast, destination 0xF): Announces the physical address of the active source. The TV and any routing switches update their internal state to route that source to the display output. Payload: 2-byte physical address (e.g., `0x20 0x00` for `2.0.0.0`).

[Source: Linux CEC documentation active source example](https://docs.kernel.org/admin-guide/media/cec.html)

### 2.2 System Standby

- **0x36 — Standby** (directed or broadcast): Instructs the destination device to enter standby mode. When broadcast, all CEC devices on the bus should enter standby.

### 2.3 Routing Control

- **0x80 — Routing Change** (broadcast): Emitted by a switch when it changes the active input port. Payload: old physical address (2 bytes) + new physical address (2 bytes).
- **0x86 — Set Stream Path** (broadcast): Directs all devices to route signal from the specified physical address to the TV output. Used by the TV to request a specific source.
- **0x85 — Request Active Source** (broadcast): A newly powered-on device asks the current active source to re-announce itself.

### 2.4 Remote Control Pass-Through

This feature allows a TV remote to control a downstream playback device without requiring the device's own remote:

- **0x44 — User Control Pressed**: Payload is a 1-byte UI command code from the CEC user control code table. Common codes:
  - `0x41` — Volume Up
  - `0x42` — Volume Down
  - `0x43` — Mute
  - `0x44` — Play
  - `0x45` — Stop
  - `0x46` — Pause
  - `0x00` — Select (OK)
  - `0x01` — Up, `0x02` — Down, `0x03` — Left, `0x04` — Right
- **0x45 — User Control Released**: Signals button release (required to terminate key repeat).

### 2.5 System Audio

- **0x70 — System Audio Mode Request**: Sent by a source to the Audio System (AV receiver) to request that it handle volume control instead of the TV.
- **0x72 — Set System Audio Mode**: Broadcast by the AV receiver to inform all devices whether system audio mode is active.
- **0x71 — Give Audio Status**: Poll to AV receiver requesting current volume and mute state.

### 2.6 OSD and Device Discovery

- **0x64 — Set OSD String**: Sends a text string (up to 13 bytes) to be displayed on the TV's OSD.
- **0x46 — Give OSD Name** / **0x47 — Set OSD Name**: Query/response for a device's human-readable name (up to 14 bytes).
- **0x8F — Give Device Power Status** / **0x90 — Report Power Status**: Query and response for power state (0=On, 1=Standby, 2=In transition to standby, 3=In transition to on).
- **0x83 — Give Physical Address** / **0x84 — Report Physical Address** (broadcast): Discovery message; the response carries the physical address and device type.
- **0x9F — Get CEC Version** / **0x9E — CEC Version**: Protocol version negotiation.

---

## 3. Linux Kernel CEC Subsystem Architecture

The Linux kernel CEC subsystem lives under `drivers/media/cec/` and was introduced by Hans Verkuil, reaching mainline in Linux 4.8 (2016). It follows the standard Linux subsystem pattern: a core framework that hardware drivers plug into, with a character device interface exposed to userspace.

[Source: Linux kernel CEC source tree](https://github.com/torvalds/linux/tree/master/drivers/media/cec)

### 3.1 Source File Organisation

```
drivers/media/cec/
├── core/
│   ├── cec-adap.c        # Adapter framework, message queuing, state machine, worker thread
│   ├── cec-api.c         # /dev/cecN character device, ioctl dispatch, file operations
│   ├── cec-core.c        # Adapter allocation, registration, sysfs/debugfs
│   ├── cec-notifier.c    # HDMI→CEC driver binding, physical address notifications
│   └── cec-pin.c         # Software bit-bang CEC using high-resolution timers
└── platform/
    ├── meson/
    │   └── ao-cec.c      # Amlogic Meson AO CEC hardware driver
    ├── rockchip/
    │   └── rk3288-hdmi-cec.c  # Rockchip RK3288 CEC driver
    ├── samsung/
    │   └── s5p_cec.c     # Samsung Exynos CEC driver
    └── ...               # Additional SoC-specific drivers

drivers/gpu/drm/vc4/
└── vc4_hdmi_cec.c        # Raspberry Pi VC4 CEC driver (in DRM tree)

drivers/gpu/drm/display/
└── drm_dp_cec.c          # DisplayPort CEC tunnelling over AUX
```

### 3.2 struct cec_adapter

The central data structure is `struct cec_adapter`, defined in `include/media/cec.h`. A driver allocates one instance per CEC-capable HDMI output. Its most significant fields are:

```c
struct cec_adapter {
    struct module           *owner;
    char                     name[32];         /* human-readable adapter name */
    struct cec_devnode       devnode;           /* /dev/cecN device node */
    struct mutex             lock;             /* protects all state below */
    struct rc_dev           *rc;              /* optional remote-control device */

    /* State */
    u16                      phys_addr;        /* current physical address (0xffff = invalid) */
    u32                      capabilities;     /* CEC_CAP_* flags from hw */
    u8                       available_log_addrs; /* max simultaneous logical addresses */

    struct cec_log_addrs     log_addrs;        /* currently configured logical addresses */

    /* Queues */
    struct list_head         transmit_queue;   /* messages awaiting transmission */
    struct list_head         wait_queue;       /* messages awaiting reply */

    const struct cec_adap_ops *ops;            /* driver callbacks */
    void                    *priv;             /* driver private data */
};
```

[Source: Linux kernel cec-core documentation](https://docs.kernel.org/driver-api/media/cec-core.html)

### 3.3 struct cec_adap_ops Callbacks

Drivers implement the `cec_adap_ops` structure to interface their hardware with the framework:

```c
struct cec_adap_ops {
    /* Low-level hardware callbacks — called with adap->lock held */
    int  (*adap_enable)(struct cec_adapter *adap, bool enable);
    int  (*adap_monitor_all_enable)(struct cec_adapter *adap, bool enable);
    int  (*adap_monitor_pin_enable)(struct cec_adapter *adap, bool enable);
    int  (*adap_log_addr)(struct cec_adapter *adap, u8 logical_addr);
    void (*adap_unconfigured)(struct cec_adapter *adap);
    int  (*adap_transmit)(struct cec_adapter *adap, u8 attempts,
                          u32 signal_free_time, struct cec_msg *msg);
    void (*adap_nb_transmit_canceled)(struct cec_adapter *adap,
                                      const struct cec_msg *msg);
    void (*adap_status)(struct cec_adapter *adap, struct seq_file *file);
    void (*adap_free)(struct cec_adapter *adap);

    /* Error injection for compliance testing */
    int  (*error_inj_show)(struct cec_adapter *adap, struct seq_file *sf);
    bool (*error_inj_parse_line)(struct cec_adapter *adap, char *line);

    /* High-level callbacks — called without adap->lock */
    void (*configured)(struct cec_adapter *adap);
    int  (*received)(struct cec_adapter *adap, struct cec_msg *msg);
};
```

The separation between low-level and high-level callbacks is critical: low-level callbacks are invoked while the adapter lock is held (they must not sleep or call back into the CEC core), while `configured()` and `received()` are called from a worker thread context with the lock released.

### 3.4 Core API for Driver Authors

```c
/* Allocate an adapter — called from driver probe() */
struct cec_adapter *cec_allocate_adapter(const struct cec_adap_ops *ops,
                                          void *priv,
                                          const char *name,
                                          u32 caps,
                                          u8 available_las);

/* Register /dev/cecN — called after hardware is ready */
int cec_register_adapter(struct cec_adapter *adap, struct device *parent);

/* Unregister and free (use in driver remove()) */
void cec_unregister_adapter(struct cec_adapter *adap);
void cec_delete_adapter(struct cec_adapter *adap);

/* Signal received message from ISR or threaded IRQ handler */
void cec_received_msg(struct cec_adapter *adap, struct cec_msg *msg);

/* Signal transmit completion from ISR */
void cec_transmit_done(struct cec_adapter *adap, u8 status,
                        u8 arb_lost_cnt, u8 nack_cnt,
                        u8 low_drive_cnt, u8 error_cnt);

/* Update physical address (e.g., on EDID change) */
void cec_s_phys_addr(struct cec_adapter *adap, u16 phys_addr, bool block);
void cec_s_phys_addr_from_edid(struct cec_adapter *adap,
                                 const struct edid *edid);
```

The `caps` argument to `cec_allocate_adapter()` carries `CEC_CAP_*` flags:
- `CEC_CAP_PHYS_ADDR` — driver or userspace manages physical address
- `CEC_CAP_LOG_ADDRS` — userspace must configure logical addresses
- `CEC_CAP_TRANSMIT` — hardware can transmit (most adapters)
- `CEC_CAP_MONITOR_ALL` — hardware can receive all messages (promiscuous)
- `CEC_CAP_RC` — can drive a Linux remote control input device

### 3.5 Message Flow

**Reception path:** Hardware interrupt fires on start bit detection → ISR or threaded IRQ reconstructs frame byte-by-byte → calls `cec_received_msg()` → core framework dispatches to:
  - monitor-mode file handles (all traffic)
  - follower-mode file handles (directed/broadcast to this device)
  - the `received()` high-level callback (for kernel-side CEC handling)

**Transmission path:** Userspace calls `CEC_TRANSMIT` ioctl → core enqueues `cec_msg` on `transmit_queue` → worker thread `cec_thread_func()` dequeues and calls `adap_transmit()` → hardware transmits → ISR calls `cec_transmit_done()` → core handles retry, arbitration loss, NACK.

The worker thread also enforces **signal-free time** requirements: CEC mandates a minimum idle period on the bus between frames — 5 bit periods for a new initiator, 7 for a retry, 5 for a different initiator.

---

## 4. Hardware CEC Driver Implementations

### 4.1 Raspberry Pi VC4 — vc4_hdmi_cec.c

The VC4 DRM driver for BCM2835/BCM2711 (Raspberry Pi 3 and 4) implements CEC in `drivers/gpu/drm/vc4/vc4_hdmi_cec.c`. CEC support for the Pi 4 landed in Linux 5.13 (mid-2021) after patches from Maxime Ripard.

The BCM2711 HDMI hardware has a dedicated CEC controller block accessible through MMIO registers. The driver maps the HDMI register space, installs an IRQ handler for start-bit detection and receive-complete events, and implements the `cec_adap_ops` callbacks:

- `adap_enable()`: enables/disables the HDMI CEC hardware block clock and interrupt.
- `adap_log_addr()`: programs the hardware address filter register so the hardware only asserts ACK for frames addressed to the configured logical addresses.
- `adap_transmit()`: writes the outgoing frame to a transmit FIFO register and triggers transmission.

The driver registers a `cec_notifier` to receive physical address updates from the HDMI connector (see Section 5).

**Note:** CEC is available only when using the full KMS overlay (`vc4-kms-v3d`), not the firmware-based fake KMS overlay (`vc4-fkms-v3d`). With fake KMS, the firmware owns the CEC hardware.

[Source: Phoronix, Linux 5.13 CEC for Raspberry Pi 4](https://www.phoronix.com/news/Linux-5.13-Raspberry-Pi-4-CEC); [Raspberry Pi Linux issue #5254](https://github.com/raspberrypi/linux/issues/5254)

### 4.2 Amlogic Meson — ao-cec.c

Amlogic Meson SoCs (used in TV boxes such as ODROID-C2, Beelink, WeTek) place CEC inside the **Always-On (AO) power domain** — the same island that remains powered in standby mode, enabling wake-from-standby via CEC without powering up the main SoC.

The driver `drivers/media/cec/platform/meson/ao-cec.c` implements:

```c
static irqreturn_t meson_ao_cec_irq(int irq, void *data)
{
    struct meson_ao_cec_device *ao_cec = data;
    /* Quick check: clear interrupt, wake threaded handler */
    return IRQ_WAKE_THREAD;
}

static irqreturn_t meson_ao_cec_irq_thread(int irq, void *data)
{
    struct meson_ao_cec_device *ao_cec = data;
    u32 stat = readl(ao_cec->base + CEC_RX_MSG_STATUS);

    if (stat & CEC_RX_MSG_DONE) {
        /* Read received frame from RX FIFO registers */
        meson_ao_cec_read_rx_msg(ao_cec);
        /* Hand to framework */
        cec_received_msg(ao_cec->adap, &ao_cec->rx_msg);
    }
    if (stat & CEC_TX_MSG_DONE) {
        cec_transmit_done(ao_cec->adap, CEC_TX_STATUS_OK,
                          0, 0, 0, 0);
    }
    return IRQ_HANDLED;
}
```

The CEC hardware block has separate TX FIFO and RX FIFO registers, a clock divider register to achieve the ~400 baud CEC timing from the AO oscillator clock, and status bits for frame-complete, NACK, and arbitration loss. The driver programs `CEC_GEN_CNTL_REG` to set up gated clock modes for power efficiency.

[Source: Linux Kernel Driver DataBase, CONFIG_CEC_MESON_AO](https://cateee.net/lkddb/web-lkddb/CEC_MESON_AO.html); [ao-cec.c source browser](https://sbexr.rabexc.org/latest/sources//c7/5162af729c4cd2.html)

### 4.3 Rockchip — rk3288-hdmi-cec.c

The Rockchip RK3288 driver (`drivers/media/cec/platform/rockchip/rk3288-hdmi-cec.c`) integrates with the Designware HDMI IP (dw-hdmi) controller found on Rockchip and other SoCs. The CEC hardware in the Designware block implements the full CEC state machine in hardware, including start-bit detection, bit sampling, ACK generation, and arbitration. The driver's `adap_transmit()` writes message bytes into the HDMI TX CEC data registers and sets a transmit-start bit; the hardware handles timing autonomously and fires a completion interrupt.

### 4.4 Common Platform Driver Pattern

Regardless of SoC, the driver probe sequence follows this pattern:

```c
static int my_cec_probe(struct platform_device *pdev)
{
    struct my_cec_dev *dev;
    struct cec_adapter *adap;
    int ret;

    dev = devm_kzalloc(&pdev->dev, sizeof(*dev), GFP_KERNEL);

    /* Map hardware registers */
    dev->base = devm_platform_ioremap_resource(pdev, 0);

    /* Install ISR */
    dev->irq = platform_get_irq(pdev, 0);
    devm_request_threaded_irq(&pdev->dev, dev->irq,
                               my_cec_irq, my_cec_irq_thread,
                               IRQF_SHARED, "my-cec", dev);

    /* Allocate CEC adapter */
    adap = cec_allocate_adapter(&my_cec_ops, dev, "my-cec",
                                 CEC_CAP_DEFAULTS, CEC_MAX_LOG_ADDRS);

    /* Register notifier for physical address from HDMI driver */
    dev->notifier = cec_notifier_cec_adap_register(dev->hdmi_dev,
                                                     NULL, adap);

    /* Register /dev/cecN */
    ret = cec_register_adapter(adap, &pdev->dev);
    dev->adap = adap;
    platform_set_drvdata(pdev, dev);
    return ret;
}
```

The `CEC_CAP_DEFAULTS` macro expands to `CEC_CAP_LOG_ADDRS | CEC_CAP_TRANSMIT | CEC_CAP_PASSTHROUGH | CEC_CAP_RC`, which is appropriate for most hardware adapters where userspace manages logical addresses and transmits messages.

---

## 5. The CEC Notifier Mechanism

### 5.1 The Subsystem Boundary Problem

Linux's HDMI stack crosses a subsystem boundary: the **DRM subsystem** owns the HDMI connector and reads the EDID (which contains the CEC physical address in the HDMI VSDB). The **media subsystem** owns the CEC adapter. These subsystems have independent driver lifecycles: the HDMI DRM driver and the CEC platform driver probe in indeterminate order, may be in separate kernel modules, and on some SoCs live in entirely different device tree nodes.

The `cec_notifier` mechanism provides a loosely-coupled asynchronous channel so that physical address updates — which may happen at any time when a monitor is connected or disconnected — can flow from the HDMI driver to the CEC adapter without tight coupling.

[Source: Linux kernel cec-notifier.c](https://github.com/torvalds/linux/blob/master/drivers/media/cec/core/cec-notifier.c)

### 5.2 Notifier API

```c
/* Called by the HDMI connector driver to register as a notifier source */
struct cec_notifier *cec_notifier_conn_register(struct device *hdmi_dev,
                                                  const char *port_name,
                                                  const struct cec_connector_info *conn_info);
void cec_notifier_conn_unregister(struct cec_notifier *n);

/* Called by the CEC adapter driver to register as a listener */
struct cec_notifier *cec_notifier_cec_adap_register(struct device *hdmi_dev,
                                                      const char *port_name,
                                                      struct cec_adapter *adap);
void cec_notifier_cec_adap_unregister(struct cec_notifier *n,
                                       struct cec_adapter *adap);

/* Called by the HDMI driver when EDID changes */
void cec_notifier_set_phys_addr(struct cec_notifier *n, u16 phys_addr);
void cec_notifier_set_phys_addr_from_edid(struct cec_notifier *n,
                                            const struct edid *edid);
```

Internally, `cec_notifier_set_phys_addr_from_edid()` calls `cec_get_edid_phys_addr()` which scans the EDID extensions for a CTA-861 block, locates the HDMI VSDB (OUI `0x000C03`), and reads bytes 4–5 as the 16-bit physical address. If no valid VSDB is found, the address is set to `CEC_PHYS_ADDR_INVALID` (0xFFFF).

### 5.3 Lifecycle Example

```
Time →
  HDMI DRM driver probes:
    cec_notifier_conn_register(hdmi_dev, "HDMI-1", &conn_info)
    → creates notifier entry in global list, phys_addr = 0xFFFF

  CEC platform driver probes:
    cec_notifier_cec_adap_register(hdmi_dev, NULL, adap)
    → finds existing notifier, links adap, replays current phys_addr

  Monitor plugged in → EDID read:
    cec_notifier_set_phys_addr_from_edid(n, edid)
    → extracts 0x2000 (2.0.0.0) from HDMI VSDB
    → calls cec_s_phys_addr(adap, 0x2000, false)
    → adap broadcasts Report Physical Address (0x84) on CEC bus

  Monitor unplugged:
    cec_notifier_set_phys_addr(n, CEC_PHYS_ADDR_INVALID)
    → adap goes unconfigured, CEC disabled
```

---

## 6. Userspace API — /dev/cecN

Each CEC adapter registered with the kernel appears as `/dev/cec0`, `/dev/cec1`, etc. The character device interface is defined through a set of ioctls operating on `struct cec_msg` and supporting structures.

[Source: Linux CEC userspace API documentation](https://www.kernel.org/doc/html/latest/userspace-api/media/cec/cec-api.html)

### 6.1 struct cec_msg

```c
struct cec_msg {
    __u64 tx_ts;           /* timestamp of transmission (ns, CLOCK_MONOTONIC) */
    __u64 rx_ts;           /* timestamp of reception */
    __u32 len;             /* message length (1–16 bytes) */
    __u32 timeout;         /* ms to wait for reply (CEC_RECEIVE) */
    __u32 sequence;        /* set by kernel, unique per transmission */
    __u32 flags;           /* CEC_MSG_FL_REPLY_TO_FOLLOWERS etc. */
    __u8  msg[CEC_MAX_MSG_SIZE];  /* 16 bytes max */
    __u8  reply;           /* expected reply opcode (0 = no reply expected) */
    __u8  rx_status;       /* CEC_RX_STATUS_* flags */
    __u8  tx_status;       /* CEC_TX_STATUS_* flags */
    __u8  tx_arb_lost_cnt;
    __u8  tx_nack_cnt;
    __u8  tx_low_drive_cnt;
    __u8  tx_error_cnt;
};
```

`msg[0]` is always the header byte (initiator nibble | destination nibble). `msg[1]` is the opcode, and `msg[2]` onward are parameters.

### 6.2 Key ioctls

| ioctl | Purpose |
|---|---|
| `CEC_ADAP_G_CAPS` | Query adapter name, driver name, capabilities, available logical addresses |
| `CEC_ADAP_G_PHYS_ADDR` / `CEC_ADAP_S_PHYS_ADDR` | Get/set physical address |
| `CEC_ADAP_G_LOG_ADDRS` / `CEC_ADAP_S_LOG_ADDRS` | Get/set logical address configuration |
| `CEC_TRANSMIT` | Transmit a CEC message (blocking or non-blocking) |
| `CEC_RECEIVE` | Receive a CEC message with timeout |
| `CEC_DQEVENT` | Dequeue an event (physical address change, logical address lost, etc.) |
| `CEC_G_MODE` / `CEC_S_MODE` | Get/set operational mode |

### 6.3 Mode Flags

```c
CEC_MODE_NO_INITIATOR    /* cannot transmit */
CEC_MODE_INITIATOR       /* can transmit CEC messages */
CEC_MODE_EXCL_INITIATOR  /* exclusive initiator (no other process may transmit) */

CEC_MODE_NO_FOLLOWER     /* do not receive directed/broadcast messages */
CEC_MODE_FOLLOWER        /* receive messages directed to configured logical addresses */
CEC_MODE_EXCL_FOLLOWER   /* exclusive follower */
CEC_MODE_MONITOR         /* receive all messages (requires CAP_NET_ADMIN) */
CEC_MODE_MONITOR_ALL     /* receive all messages including those the hw filters out */
```

### 6.4 Complete C Example: Wake TV and Switch Input

The following example opens `/dev/cec0`, configures the adapter as a Playback Device, then sends the One Touch Play sequence to wake and switch the TV:

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <linux/cec.h>

int main(void)
{
    int fd = open("/dev/cec0", O_RDWR);
    if (fd < 0) { perror("open"); return 1; }

    /* Step 1: Query capabilities */
    struct cec_caps caps = {};
    ioctl(fd, CEC_ADAP_G_CAPS, &caps);
    printf("Adapter: %s/%s, caps=0x%x\n",
           caps.driver, caps.name, caps.capabilities);

    /* Step 2: Set mode — we will initiate transmissions */
    unsigned int mode = CEC_MODE_INITIATOR | CEC_MODE_FOLLOWER;
    ioctl(fd, CEC_S_MODE, &mode);

    /* Step 3: Configure as Playback Device */
    struct cec_log_addrs log_addrs = {};
    log_addrs.num_log_addrs = 1;
    log_addrs.log_addr_type[0] = CEC_LOG_ADDR_TYPE_PLAYBACK;
    log_addrs.primary_device_type[0] = CEC_OP_PRIM_DEVTYPE_PLAYBACK;
    log_addrs.all_device_types[0] = CEC_OP_ALL_DEVTYPE_PLAYBACK;
    snprintf((char *)log_addrs.osd_name, sizeof(log_addrs.osd_name),
             "LinuxPC");
    log_addrs.flags = CEC_LOG_ADDRS_FL_ALLOW_UNREG_FALLBACK;
    ioctl(fd, CEC_ADAP_S_LOG_ADDRS, &log_addrs);

    /* Re-read to get assigned logical address */
    ioctl(fd, CEC_ADAP_G_LOG_ADDRS, &log_addrs);
    uint8_t my_la = log_addrs.log_addr[0];  /* e.g., 4 = Playback Device 1 */
    printf("Assigned logical address: %d\n", my_la);

    /* Step 4: Get physical address */
    uint16_t phys_addr;
    ioctl(fd, CEC_ADAP_G_PHYS_ADDR, &phys_addr);
    printf("Physical address: %d.%d.%d.%d\n",
           (phys_addr >> 12) & 0xf, (phys_addr >> 8) & 0xf,
           (phys_addr >> 4) & 0xf, phys_addr & 0xf);

    /* Step 5: Send Image View On to TV (logical address 0) */
    struct cec_msg msg = {};
    cec_msg_init(&msg, my_la, 0);   /* src=my_la, dst=TV(0) */
    cec_msg_image_view_on(&msg);    /* opcode 0x04 */
    ioctl(fd, CEC_TRANSMIT, &msg);

    usleep(100000);  /* give TV time to wake */

    /* Step 6: Broadcast Active Source */
    cec_msg_init(&msg, my_la, 0xf);  /* broadcast */
    cec_msg_active_source(&msg, phys_addr);  /* opcode 0x82 */
    ioctl(fd, CEC_TRANSMIT, &msg);

    printf("Done — TV should now display this device's input.\n");
    close(fd);
    return 0;
}
```

This code uses the helper macros from `<linux/cec.h>` (`cec_msg_init`, `cec_msg_image_view_on`, `cec_msg_active_source`) which pack the correct opcode and parameter bytes.

---

## 7. cec-ctl and v4l-utils Tooling

The `v4l-utils` package provides three CEC-specific utilities: `cec-ctl` for sending commands and monitoring traffic, `cec-compliance` for protocol compliance testing, and `cec-follower` for simulating a CEC device.

[Source: v4l-utils CEC utilities](https://git.linuxtv.org/v4l-utils.git/tree/utils/cec-ctl); [cec-ctl man page](https://man.archlinux.org/man/cec-ctl.1.en)

### 7.1 cec-ctl Reference

**Device configuration:**
```bash
# Configure adapter as a Playback Device with OSD name
cec-ctl -d /dev/cec0 --playback --osd-name="LinuxPC"

# Configure as TV
cec-ctl -d /dev/cec0 -p 0.0.0.0 --tv

# Show CEC network topology
cec-ctl -d /dev/cec0 -S
```

**One Touch Play sequence:**
```bash
# Wake the TV (must be first)
cec-ctl -d /dev/cec0 --to 0 --image-view-on

# Announce active source at physical address 2.0.0.0
cec-ctl -d /dev/cec0 --to 15 --active-source phys-addr=2.0.0.0
```

**Standby and volume control:**
```bash
# Send standby to TV
cec-ctl -d /dev/cec0 --to 0 --standby

# Volume up via user control pass-through
cec-ctl -d /dev/cec0 --to 0 --user-control-pressed ui-cmd=volume-up
cec-ctl -d /dev/cec0 --to 0 --user-control-released
```

**Monitoring:**
```bash
# Monitor all CEC traffic (requires root for CEC_MODE_MONITOR_ALL)
sudo cec-ctl -d /dev/cec0 --monitor-all

# Low-level pin monitoring (bit-bang adapters only)
sudo cec-ctl -d /dev/cec0 --monitor-pin
```

### 7.2 Monitoring Output Interpretation

`cec-ctl --monitor-all` prints each frame in the form:

```
14:00 -> 0f:82:20:00    (Active Source, phys-addr=2.0.0.0)
00:04 -> 0f:82:00:00    (Active Source, phys-addr=0.0.0.0)
04:00 -> 46             (Give OSD Name)
00:04 -> 47:4c:69:6e:75:78:50:43  (Set OSD Name, name="LinuxPC")
```

The format is `SRC:DST -> BYTES  (decoded message)`. A source of `0f` means broadcast from the TV; `04` is Playback Device 1.

### 7.3 Compliance Testing

`cec-compliance` sends a structured sequence of CEC messages to test an adapter's conformance with the CEC specification. It checks correct opcode handling, reply timing, and logical address behaviour:

```bash
# Run compliance test against TV (logical address 0)
cec-compliance -d /dev/cec0 --to 0
```

`cec-follower` simulates a CEC device on the bus, responding to incoming messages according to the specification. It is useful for testing CEC initiator behaviour in software before real hardware is available.

### 7.4 Systemd Unit for Wake-on-Resume

The following unit file triggers One Touch Play whenever the system resumes from suspend, automatically waking and switching the connected TV:

```ini
# /etc/systemd/system/cec-wake-tv.service
[Unit]
Description=Send CEC One Touch Play on system resume
After=suspend.target

[Service]
Type=oneshot
ExecStart=/bin/bash -c '\
  cec-ctl -d /dev/cec0 --to 0 --image-view-on; \
  sleep 0.2; \
  cec-ctl -d /dev/cec0 --to 15 \
    --active-source phys-addr=$(cec-ctl -d /dev/cec0 --phys-addr)'

[Install]
WantedBy=suspend.target
```

Activate with `systemctl enable --now cec-wake-tv.service`.

---

## 8. libcec — Userspace Abstraction

### 8.1 Purpose and Supported Platforms

`libcec` is a cross-platform CEC abstraction library maintained by Pulse-Eight ([https://github.com/Pulse-Eight/libcec](https://github.com/Pulse-Eight/libcec)). It provides a unified C++ API (`ICECAdapter`) over several underlying transport mechanisms:

| Platform adapter | Underlying transport |
|---|---|
| Linux `/dev/cecN` | Kernel CEC character device |
| Raspberry Pi | Direct VideoCore GPU firmware CEC |
| Pulse-Eight USB dongle | `/dev/ttyACM0` via `inputattach` |
| NXP TDA995x | I²C CEC controller |
| Exynos SoCs | Kernel CEC device |

### 8.2 C++ API Overview

The primary interface is `ICECAdapter`, obtained from `LibCecInitialise()`:

```cpp
#include <libcec/cec.h>
using namespace CEC;

ICECAdapter *adapter = LibCecInitialise(&config);

// Open first available CEC adapter
cec_adapter_descriptor devices[10];
int count = adapter->DetectAdapters(devices, 10);
adapter->Open(devices[0].strComName);  // e.g., "/dev/cec0"

// Send One Touch Play
cec_command cmd;
cec_command::Format(cmd, CECDEVICE_BROADCAST,
                    CEC_OPCODE_ACTIVE_SOURCE);
cmd.parameters.PushBack((uint8_t)(phys_addr >> 8));
cmd.parameters.PushBack((uint8_t)(phys_addr & 0xff));
adapter->Transmit(cmd);

// Register callback for incoming commands
adapter->SetCallbacks(&myCallbackImpl, nullptr);
```

Key `ICECAdapter` methods:
- `Open(port, timeout)` — open the CEC adapter by path
- `Close()` — disconnect
- `Transmit(command)` — send a CEC command
- `SendKeypress(logicalAddress, key, wait)` — send user control pressed
- `PowerOnDevices(logicalAddress)` — send Image View On + Active Source
- `StandbyDevices(logicalAddress)` — send Standby
- `SetConfiguration(config)` — update device configuration at runtime

### 8.3 Kodi CEC Integration

Kodi uses libcec through its **CEC peripheral plugin** (`xbmc/peripherals/devices/PeripheralCecAdapter.cpp`). The plugin:

1. Detects available CEC adapters at startup via `DetectAdapters()`.
2. Opens the adapter and configures it as a Playback Device with OSD name "Kodi".
3. Sends **One Touch Play** (Image View On + Active Source) when playback begins.
4. Sends **Standby** when Kodi exits or the user requests system shutdown.
5. Sends **Inactive Source** (opcode `0x9D`) when Kodi stops being the active source.
6. Translates received **User Control Pressed** (0x44) messages into Kodi remote control events (play, pause, navigation).

User-configurable settings in Kodi's CEC peripheral:
- **Wake TV on start** — enables the Image View On sequence.
- **Power off TV on exit** — enables Standby on Kodi shutdown.
- **Send inactive source when switching input** — sends 0x9D to let the TV revert to its previous source.
- **CEC client adapter** — selects the physical adapter (useful when multiple CEC devices are present).

[Source: Kodi CEC wiki](https://kodi.wiki/view/CEC); [Pulse-Eight libcec README](https://github.com/Pulse-Eight/libcec)

---

## 9. CEC in DRM — Pin Framework and DP CEC Tunnelling

### 9.1 The cec-gpio / cec-pin Software Bit-Bang Framework

Some GPU drivers or embedded boards cannot provide a hardware CEC controller but do expose the raw CEC pin state. For these devices, Linux provides the **CEC pin framework** in `drivers/media/cec/core/cec-pin.c`.

The pin framework implements the complete CEC bit-banging protocol in software using Linux high-resolution timers (`hrtimer`). It requires a driver to implement `struct cec_pin_ops`:

```c
struct cec_pin_ops {
    bool (*read)(struct cec_adapter *adap);  /* sample pin level */
    void (*low)(struct cec_adapter *adap);   /* drive pin low */
    void (*high)(struct cec_adapter *adap);  /* release to high (open drain) */
    bool (*enable_irq)(struct cec_adapter *adap); /* enable edge IRQ */
    void (*disable_irq)(struct cec_adapter *adap);/* disable edge IRQ */
    void (*free)(struct cec_adapter *adap);  /* release resources */
    void (*status)(struct cec_adapter *adap, struct seq_file *file);
};
```

The `cec-gpio` platform driver (`drivers/media/cec/platform/cec-gpio.c`) wraps a GPIO descriptor with these operations, enabling CEC over any GPIO line specified in Device Tree:

```dts
/* Device Tree snippet for GPIO CEC */
cec {
    compatible = "cec-gpio";
    cec-gpios = <&gpio 42 (GPIO_ACTIVE_HIGH | GPIO_OPEN_DRAIN)>;
    hdp-gpios = <&gpio 43 GPIO_ACTIVE_HIGH>;  /* optional HPD */
};
```

The hrtimer fires at each expected bit boundary (~0.6 ms, ~1.5 ms, ~3.7 ms intervals). Because software timer latency can exceed 300 µs, pin-based CEC is not 100% reliable under high system load. The kernel documentation recommends disabling NTP clock slewing (`maxslewrate 40000` in `chrony.conf`) to reduce jitter on timer-based CEC implementations.

[Source: cec-gpio Device Tree binding](https://www.kernel.org/doc/Documentation/devicetree/bindings/media/cec-gpio.txt); [CEC pin framework documentation](https://docs.kernel.org/driver-api/media/cec-core.html)

### 9.2 DRM Connector CEC_PIN_IS_HIGH Property

On GPUs where the DRM driver can read the current CEC pin level but cannot implement a full hardware CEC controller, the driver can expose a DRM connector boolean property named `CEC_PIN_IS_HIGH`. The cec-pin framework polls or interrupt-monitors this property to implement software CEC.

This allows GPUs that expose HDMI through a DRM connector (where the CEC pin is accessible as a register read) to participate in CEC without dedicated CEC hardware. The DRM driver calls `drm_cec_notifier_set_phys_addr_from_edid()` when the EDID changes to pass the physical address to the CEC adapter.

### 9.3 DisplayPort CEC Tunnelling — drm_dp_cec.c

Active DisplayPort-to-HDMI adapters that comply with the VESA DisplayPort CEC-Tunneling-over-AUX standard can transparently relay CEC messages between a DP source and HDMI sink. The Linux implementation lives in `drivers/gpu/drm/display/drm_dp_cec.c`.

[Source: drm_dp_cec.c kernel source](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/display/drm_dp_cec.c)

The implementation uses DPCD (DisplayPort Configuration Data) registers in the range `0x3000`–`0x301F`:

| DPCD Register | Purpose |
|---|---|
| `DP_CEC_TUNNELING_CONTROL` | Enable CEC tunnelling and snooping |
| `DP_CEC_RX_MESSAGE_BUFFER` | Received CEC frame data |
| `DP_CEC_RX_MESSAGE_INFO` | RX frame length and status |
| `DP_CEC_TX_MESSAGE_BUFFER` | Frame to transmit |
| `DP_CEC_TX_MESSAGE_INFO` | TX frame length, trigger transmission |
| `DP_CEC_LOGICAL_ADDRESS_MASK` | Logical address filter for ACK generation |
| `DP_CEC_TUNNELING_IRQ_FLAGS` | Interrupt status: TX done, RX done, etc. |

The primary entry point for DRM GPU drivers is:

```c
/* Called when EDID becomes available on a DP connector */
void drm_dp_cec_set_edid(struct drm_dp_aux *aux,
                          const struct edid *edid);

/* Called when the DP connection is lost */
void drm_dp_cec_unset_edid(struct drm_dp_aux *aux);
```

`drm_dp_cec_set_edid()` extracts the CEC physical address from the EDID's HDMI VSDB and calls `drm_dp_cec_attach()` to register a CEC adapter. The GPU drivers for Intel (i915), Nouveau, and AMDGPU call these functions when a DP-to-HDMI adapter is detected.

**Limitation:** The DP CEC tunnelling specification only supports a **single HDMI device** behind the adapter. Multi-port DP-to-HDMI hubs cannot tunnel CEC for all connected devices simultaneously.

---

## 10. Practical Automation and Troubleshooting

### 10.1 Home Automation Integration

**Home Assistant** supports CEC through the `pycec` integration, which wraps `/dev/cec0` via Python bindings. The integration exposes each CEC-discovered device as a Home Assistant `media_player` entity, enabling automations like:

```yaml
# Example Home Assistant automation: turn on TV when arriving home
automation:
  trigger:
    platform: state
    entity_id: person.owner
    to: home
  action:
    service: media_player.turn_on
    target:
      entity_id: media_player.samsung_tv_cec
```

Under the hood, `pycec` sends `Image View On` followed by `Active Source` via the `/dev/cecN` ioctl interface.

### 10.2 CEC Bus Arbitration and Collision Recovery

When two devices attempt to transmit simultaneously, CEC resolves the conflict through **bit-level arbitration**: each transmitter monitors the bus while sending. If it drives a logical 1 (releases the line high) but reads back a 0 (dominant), it has lost arbitration to another device. The losing transmitter:

1. Immediately stops driving and defers transmission.
2. Backs off for a random number of signal-free periods.
3. Retries when the bus is idle.

The Linux framework tracks arbitration loss counts in `cec_msg.tx_arb_lost_cnt` and automatically retries up to the configured retry limit.

### 10.3 Physical Address Failures — 0xFFFF

The single most common CEC failure mode is `phys_addr = 0xFFFF` (invalid). This occurs when:

1. **No HDMI VSDB in EDID**: Older monitors, DVI-to-HDMI adapters, and some projectors do not include the HDMI Vendor Specific Data Block in their EDID. Without it, there is no CEC physical address to extract. **Diagnosis:** Run `edid-decode /sys/class/drm/card0-HDMI-A-1/edid | grep "Source Physical"`.

2. **EDID not yet available**: The CEC adapter was registered before the display's EDID was read. The physical address will update asynchronously when the EDID is read; check `dmesg | grep cec` for "set_phys_addr" messages.

3. **HPD not asserted**: If the hotplug-detect pin is not driven by the downstream device, no EDID handshake occurs. Check `/sys/class/drm/card0-HDMI-A-1/status`.

4. **CEC notifier not wired up**: The DRM driver and CEC driver are not connected via the notifier framework. On some SoC platforms, the connection is via Device Tree phandle (`hdmi-phandle = <&hdmi_node>`).

```bash
# Check current physical address
cec-ctl -d /dev/cec0 --phys-addr

# Expected output for a valid address:
# Physical Address: 2.0.0.0

# Expected output when invalid:
# Physical Address: f.f.f.f
```

### 10.4 HDMI Switch CEC Compatibility

Consumer HDMI switches vary widely in CEC support. A well-behaved switch:
- Rewrites the physical address in the EDID it presents to each downstream device, reflecting the correct depth (e.g., a device on port 1 of a switch at `2.0.0.0` receives `2.1.0.0`).
- Forwards routing change messages and updates its internal routing state.
- Passes through all CEC messages transparently.

Cheap switches may:
- Present the TV's EDID unmodified, causing all downstream devices to believe they are at `0.0.0.0`.
- Silently drop CEC messages, breaking routing control.
- Block broadcast messages.

If multiple devices show the same physical address, the HDMI switch is not rewriting EDID correctly. In this case, CEC routing commands will not reliably switch the switch's input.

### 10.5 CEC vs ARC/eARC

CEC and ARC (Audio Return Channel) both use the same HDMI connector but on **different pins**: CEC on pin 13, ARC on pin 37 (Type A). They are independent and coexist without interference. CEC is used to send control commands (volume, standby, source selection), while ARC/eARC carries high-quality audio back from the TV to an AV receiver. CEC's System Audio Mode commands (`0x70`, `0x72`) are used to negotiate which device handles volume control when ARC/eARC is active.

[Source: Understanding HDMI cable features, fycables.com](https://fycables.com/understanding-hdmi-cable-features-arc-earc-cec-and-more/)

### 10.6 Debugging with dmesg

Enable verbose CEC kernel messages:

```bash
# Enable dynamic debug for the CEC core
echo 'module cec +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/media/cec/* +p' > /sys/kernel/debug/dynamic_debug/control

# Check for CEC adapter registration and physical address updates
dmesg | grep -i cec
```

Typical success sequence in `dmesg`:
```
[    5.231] cec-gpio cec-gpio: Registered CEC adapter
[   12.047] cec0: physical address: 2.0.0.0
[   12.049] cec0: logical address: 4 (Playback Device 1)
[   12.052] cec0: report physical address: 2.0.0.0 (Playback Device)
```

---

## 11. Integrations

- **Ch2 (KMS and DRM Connector Model):** The `drm_connector` struct exposes the `CEC_PIN_IS_HIGH` boolean connector property when a GPU driver supports software CEC. The DRM KMS modeset pipeline also triggers physical address updates through the CEC notifier when EDID is read or cleared during hotplug events.

- **Ch140 (HDMI Audio — ARC and eARC):** ARC operates on a separate HDMI pin (pin 37, Type A) from CEC (pin 13) but the two interact functionally: CEC System Audio Mode messages (opcodes `0x70`, `0x72`) negotiate which device handles volume control when an eARC link is active. Both features are coordinated by the HDMI connector driver that manages the full pin set.

- **Ch142 (V4L2 and the Media Subsystem):** The CEC subsystem lives physically within `drivers/media/`, following the V4L2 subsystem's device node (`/dev/cecN`) and ioctl conventions. The `cec-compliance` and `cec-follower` tools from `v4l-utils` share infrastructure with V4L2 compliance testing tools. The media controller framework is not directly used by CEC, but both subsystems share the `media_devnode` infrastructure.

- **Ch183 (EDID and DisplayID — Capability Discovery):** The CEC physical address is carried in the HDMI Vendor Specific Data Block within the CTA-861 extension block of the EDID, at bytes 4–5 of the block with OUI `0x000C03`. The `cec_get_edid_phys_addr()` kernel function and the `edid-decode` tool both extract this field. When no HDMI VSDB is present, the CEC physical address is `0xFFFF` and CEC cannot function correctly.

- **Ch182 (HDMI Connector Physical Layer):** HDMI pin 13 is the dedicated CEC bus conductor shared by all devices on an HDMI network. The HDMI specification section 6.3 defines the electrical characteristics: open-drain, 3.3 V pull-up, maximum 10 m cable, 400 baud. The other functionally related pin is pin 18 (5V power, used for EDID power) and pin 19 (HPD, which triggers EDID reads that provide the CEC physical address).

## Roadmap

### Near-term (6–12 months)
- The CEC subsystem is gaining improved error injection infrastructure via debugfs, allowing kernel CI frameworks to automate CEC compliance testing against simulated bus faults — patches from Hans Verkuil are under active review on linux-media.
- HDMI 2.1 brings an expanded CEC 2.0 command set including HEC (HDMI Ethernet Channel) coordination opcodes; kernel support for the remaining unimplemented CEC 2.0 messages is tracked in the v4l-utils compliance test suite and is expected to be filled incrementally.
- The cec-pin software bit-bang framework is being hardened against high-resolution timer jitter through adaptive sampling windows, reducing CEC failures on heavily loaded embedded systems (Raspberry Pi in particular).
- Upstream work on improving the `drm_dp_cec` tunnelling driver aims to extend DisplayPort CEC tunnelling to cover more Intel, AMD, and Nouveau GPU paths where DP-to-HDMI adapters are common.

### Medium-term (1–3 years)
- As eARC displaces ARC, CEC System Audio Mode negotiation will need tighter kernel-level coordination between the CEC subsystem and ALSA/ASoC audio routing drivers; a proposed `cec-audio` bridge API may unify volume and source selection across both control planes.
- SoC vendors (Amlogic, Rockchip, MediaTek) are moving CEC hardware into always-on power islands that also contain PMU and RTC logic; future drivers will integrate CEC wake sources with the Linux `wakeup_source` and PM-domain frameworks to enable reliable standby-to-on transitions via CEC without main SoC power.
- The libcec project is expected to consolidate its Linux backend exclusively around the `/dev/cecN` kernel interface, deprecating the legacy Raspberry Pi VideoCore firmware path in favour of the mainline vc4 DRM CEC driver which now supports Pi 4 and Pi 5.
- Home automation platforms (Home Assistant, openHAB) are driving demand for a higher-level D-Bus CEC service daemon (similar to BlueZ for Bluetooth) that would expose CEC device discovery and control as standardised D-Bus interfaces, abstracting away `/dev/cecN` ioctls for application developers.

### Long-term
- As HDMI 2.2 and Ultra High Speed cabling mature, CEC may be extended or replaced by a higher-bandwidth in-band control channel (analogous to DisplayPort's sideband messaging) to support the growing command vocabulary needed by HDR metadata negotiation and adaptive sync coordination — the Linux stack would require a new subsystem layer bridging the existing CEC API to any successor protocol.
- Industry convergence on Matter (formerly CHIP) as the dominant home automation protocol may eventually see CEC bridges standardised at the firmware level, with Linux SoC platforms expected to expose CEC-discovered device state as Matter endpoints via kernel-to-userspace bridge daemons.
- Long-term kernel maintainer succession planning for the CEC subsystem (currently maintained by a small number of individuals at Cisco/LinuxTV) may lead to subsystem reorganisation, folding CEC more tightly into the DRM connector model rather than the media subsystem to reflect its role as a display control bus.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
