# Chapter 214: Bluetooth Audio on Linux: BlueZ, A2DP, HFP, and LE Audio

> **Part**: Part XXVIII — Linux Multimedia
> **Audiences**: Systems developers working on Bluetooth audio integration; audio application developers targeting wireless headsets and speakers.

## Table of Contents

- [1. Bluetooth Stack Overview](#1-bluetooth-stack-overview)
  - [1.3 What is BlueZ?](#13-what-is-bluez)
  - [1.4 What is A2DP?](#14-what-is-a2dp)
  - [1.5 What is HFP?](#15-what-is-hfp)
  - [1.6 What is LE Audio?](#16-what-is-le-audio)
- [2. A2DP: Advanced Audio Distribution Profile](#2-a2dp-advanced-audio-distribution-profile)
- [3. HFP and HSP: Hands-Free and Headset Profiles](#3-hfp-and-hsp-hands-free-and-headset-profiles)
- [4. AVRCP: Remote Control and Media Browsing](#4-avrcp-remote-control-and-media-browsing)
- [5. PipeWire as Bluetooth Audio Backend](#5-pipewire-as-bluetooth-audio-backend)
- [6. LE Audio: BLE-Based Audio](#6-le-audio-ble-based-audio)
- [7. BlueZ LE Audio Support](#7-bluez-le-audio-support)
- [8. LC3 Codec Deep Dive](#8-lc3-codec-deep-dive)
- [9. Audio Policy and Routing](#9-audio-policy-and-routing)
- [10. Latency Analysis](#10-latency-analysis)
- [11. oFono Integration](#11-ofono-integration)
- [12. Debugging](#12-debugging)
- [Integrations](#integrations)

---

## 1. Bluetooth Stack Overview

BlueZ is the official Linux Bluetooth stack, spanning both kernel and userspace. The kernel side lives in two trees: `net/bluetooth/` implements the core protocol suite — HCI core, L2CAP, SCO, BNEP, RFCOMM, and the ISO layer added for LE Audio — while `drivers/bluetooth/` contains vendor-specific hardware adaptors. The USB subsystem driver `btusb` covers the vast majority of USB dongles and built-in controllers, with sub-drivers (`intel`, `broadcom`, `realtek`, `mediatek`) handling firmware loading. UART-connected controllers on embedded systems use `hci_uart.c`, which implements the N_HCI line discipline. [Source](https://naehrdine.blogspot.com/2021/03/bluez-linux-bluetooth-stack-overview.html)

### 1.1 Kernel Protocol Layers

The classic Bluetooth radio stack from bottom to top is:

```
HCI (USB/UART/SDIO transport)
  └─ L2CAP (logical link control and adaptation)
       ├─ RFCOMM  (serial emulation over L2CAP PSM 3)
       ├─ AVDTP   (audio/video distribution transport, PSM 25)
       └─ AVCTP   (audio/video control transport, PSM 23)
```

L2CAP operates with three channel modes relevant to audio: Basic (connection-oriented, unreliable), ERTM (Enhanced Retransmission Mode, reliable), and Streaming (unreliable but in-order, used for A2DP media). PSM (Protocol/Service Multiplexer) assignments for classic Bluetooth audio: AVDTP signaling uses PSM `0x0019` (25), AVCTP control uses PSM `0x0017` (23), AVCTP browsing uses `0x001B` (27), and RFCOMM multiplexes over PSM `0x0003`. [Source](https://man.archlinux.org/man/extra/bluez-utils/l2cap.7.en)

Userspace sees these as `AF_BLUETOOTH` sockets:

```c
/* L2CAP socket for AVDTP (profiles/audio/avdtp.c) */
int fd = socket(PF_BLUETOOTH, SOCK_SEQPACKET, BTPROTO_L2CAP);

struct sockaddr_l2 addr = {
    .l2_family = AF_BLUETOOTH,
    .l2_psm    = htobs(0x0019),  /* AVDTP PSM = 25 */
    .l2_bdaddr = remote_bdaddr,
};
connect(fd, (struct sockaddr *)&addr, sizeof(addr));

/* SCO socket for HFP voice */
int sco_fd = socket(PF_BLUETOOTH, SOCK_SEQPACKET, BTPROTO_SCO);
struct sockaddr_sco sco_addr = {
    .sco_family = AF_BLUETOOTH,
    .sco_bdaddr = remote_bdaddr,
};

/* ISO socket for LE Audio — requires kernel 6.4+ */
int iso_fd = socket(PF_BLUETOOTH, SOCK_SEQPACKET, BTPROTO_ISO);
```

The `BTPROTO_ISO` socket type was added around kernel 6.0 and reached stable form in 6.4; it is the foundation for LE Audio CIS and BIS connections.

### 1.2 BlueZ Daemon Architecture

The `bluetoothd` userspace daemon runs as root and communicates with the kernel through two mechanisms: the **Management API** (`MGMT` socket, documented in `doc/mgmt-api.txt` in the BlueZ tree) is the recommended abstract interface for all controller configuration and device management; raw HCI sockets are reserved for diagnostic tools such as `btmon`. [Source](https://linuxvox.com/blog/bluez-architecture-explain-this-architecture/)

`bluetoothd` loads profile plugins at runtime. Each profile plugin registers a handler for its profile UUID and receives device-level connect/disconnect callbacks. Persistent device state — bonding keys, profile state, cached attributes — is stored in `/var/lib/bluetooth/<adapter_addr>/<device_addr>/`.

The MGMT API abstracts away controller-specific HCI command variations. Key MGMT commands include `MGMT_OP_READ_INFO` (adapter capabilities), `MGMT_OP_SET_POWERED`, `MGMT_OP_SET_CONNECTABLE`, `MGMT_OP_SET_DISCOVERABLE`, `MGMT_OP_PAIR_DEVICE`, and `MGMT_OP_SET_EXP_FEATURE` (for enabling experimental features like the ISO socket). `bluetoothd` opens a single `AF_BLUETOOTH / BTPROTO_HCI` socket in management mode — `hci_sock_bind()` with `HCI_CHANNEL_CONTROL` — and all adapter lifecycle management passes through it. Raw HCI sockets with `HCI_CHANNEL_RAW` are what `btmon` and legacy tools use. [Source](https://naehrdine.blogspot.com/2021/03/bluez-linux-bluetooth-stack-overview.html)

The public API to the rest of the system is **D-Bus** under the `org.bluez` service name. Key interfaces under `/org/bluez`:

| Interface | Purpose |
|---|---|
| `org.bluez.Adapter1` | Controller properties, StartDiscovery/StopDiscovery |
| `org.bluez.Device1` | Device properties (Connected, Paired), Connect/Disconnect |
| `org.bluez.MediaEndpoint1` | Codec endpoint negotiation |
| `org.bluez.MediaTransport1` | Acquire transport file descriptor |
| `org.bluez.MediaPlayer1` | Playback control and browsing |
| `org.bluez.MediaControl1` | Basic AVRCP pass-through |

The `MediaEndpoint1` interface is the primary extension point through which audio servers (PipeWire, bluealsa) register codec endpoints and receive codec configuration from BlueZ. [Source](https://man.archlinux.org/man/extra/bluez-utils/org.bluez.MediaEndpoint.5.en)

### 1.3 What is BlueZ?

BlueZ is the official Bluetooth protocol stack for Linux, consisting of both a kernel-space component and a userspace daemon. The kernel portion, located under `net/bluetooth/` in the Linux source tree, implements the full Bluetooth protocol suite: HCI (Host Controller Interface), L2CAP (Logical Link Control and Adaptation Protocol), SCO (Synchronous Connection-Oriented links for voice), RFCOMM (serial port emulation), BNEP (Bluetooth Network Encapsulation Protocol), and the ISO layer introduced for LE Audio. Hardware drivers in `drivers/bluetooth/` bind specific USB dongles, UART modules, and SDIO adapters to the HCI core layer.

The userspace component is `bluetoothd`, a D-Bus service that exposes the `org.bluez` interface hierarchy to applications. `bluetoothd` interacts with the kernel exclusively through the Management API (MGMT socket), which abstracts away controller-specific HCI command variations. Audio applications and servers interact with `bluetoothd` through D-Bus interfaces such as `org.bluez.MediaEndpoint1` and `org.bluez.MediaTransport1` to register codec endpoints and acquire streaming file descriptors. BlueZ is the foundation on which all Bluetooth audio profiles on Linux — A2DP, HFP, HSP, AVRCP, and LE Audio — are built. Persistent state for bonded devices lives under `/var/lib/bluetooth/<adapter_addr>/<device_addr>/`, including link keys, profile activation records, and cached attribute values.

### 1.4 What is A2DP?

A2DP (Advanced Audio Distribution Profile) is the Bluetooth profile that defines one-way stereo audio streaming from a Source device (such as a phone or computer) to a Sink device (such as wireless headphones or a speaker). It operates over AVDTP (Audio/Video Distribution Transport Protocol), which runs over L2CAP using PSM 25 for signaling and negotiates separate media transport L2CAP channels for the actual audio stream.

A2DP requires support for SBC (Sub-Band Codec) as the mandatory baseline codec. Source and Sink endpoints — called Stream Endpoints (SEPs) — negotiate codec parameters using AVDTP signaling before streaming begins. Optional codecs such as aptX, AAC, LDAC, Opus, and LC3 are layered on top of the same AVDTP negotiation mechanism and require out-of-band codec capability advertisement. On Linux, BlueZ manages AVDTP signaling and SEP negotiation; the audio server (PipeWire or bluealsa) registers its supported codec endpoints through the `org.bluez.MediaEndpoint1` D-Bus interface, acquires a raw socket file descriptor via `org.bluez.MediaTransport1.Acquire()`, and encodes PCM audio into the negotiated codec format for transmission over the Bluetooth link.

### 1.5 What is HFP?

HFP (Hands-Free Profile) is the Bluetooth profile used for bidirectional voice communication between an Audio Gateway (AG) — typically a phone or laptop — and a Hands-Free Unit (HFU) such as a headset or car kit. It provides full call control through AT command signaling: call origination and answering, status indicators, volume adjustment, three-way calling, and codec negotiation. The predecessor HSP (Headset Profile) covers only basic microphone and speaker functionality without call control.

HFP uses two separate logical connections: an RFCOMM channel for AT command signaling, and a synchronous SCO or eSCO link for raw PCM voice audio. The codec used on the SCO link is negotiated during Service Level Connection (SLC) setup: CVSD (Continuous Variable Slope Delta modulation) provides 8 kHz narrowband audio, while mSBC (modified Sub-Band Codec) provides 16 kHz wideband audio when both endpoints support HFP 1.6 or later. Qualcomm's aptX Voice provides 32 kHz super-wideband audio in HFP 1.9. On Linux, BlueZ handles HFP and HSP signaling through its profile plugins, and audio servers such as PipeWire obtain SCO audio file descriptors from BlueZ over D-Bus and route voice streams through the standard audio graph.

### 1.6 What is LE Audio?

LE Audio is the Bluetooth audio framework built on Bluetooth Low Energy (BLE) radio technology rather than the classic Bluetooth radio used by A2DP and HFP. It replaces SCO links and AVDTP streams with ISO (Isochronous) channel types: CIS (Connected Isochronous Streams) for unicast audio to a paired headset, and BIS (Broadcast Isochronous Streams) for one-to-many audio broadcasting under the Auracast brand. LE Audio mandates LC3 (Low Complexity Communication Codec) as its baseline codec, which achieves better audio quality than SBC at lower bitrates and targets end-to-end latency below 20 ms in low-latency mode.

The LE Audio profile stack includes BAP (Basic Audio Profile) for stream setup, PACS (Published Audio Capabilities Service) for codec capability advertisement, ASCS (Audio Stream Control Service) for stream control, and TMAP and HAP for telephony and hearing aid use cases. On Linux, LE Audio support is built on `BTPROTO_ISO` sockets stabilized in kernel 6.4, which give userspace direct access to CIS and BIS Isochronous links. BlueZ 5.66 and later expose LE Audio profiles behind experimental feature flags. PipeWire's bluez5 plugin integrates LC3 codec support to complete the path from the Bluetooth ISO transport layer to the Linux audio graph.

---

## 2. A2DP: Advanced Audio Distribution Profile

A2DP defines a one-way stereo audio stream from a Source to a Sink using the AVDTP transport protocol. All A2DP devices must support SBC; all other codecs are optional extensions.

### 2.1 AVDTP State Machine

AVDTP (Audio/Video Distribution Transport Protocol) manages Stream Endpoint (SEP) negotiation and media transport over L2CAP. The state machine in `profiles/audio/avdtp.c` defines six states: [Source](https://github.com/RadiusNetworks/bluez/blob/master/profiles/audio/avdtp.c)

```c
/* profiles/audio/avdtp.c — AVDTP stream endpoint states */
typedef enum {
    AVDTP_STATE_IDLE       = 0,
    AVDTP_STATE_CONFIGURED = 1,
    AVDTP_STATE_OPEN       = 2,
    AVDTP_STATE_STREAMING  = 3,
    AVDTP_STATE_CLOSING    = 4,
    AVDTP_STATE_ABORTING   = 5,
} avdtp_state_t;

/* Signal IDs used in AVDTP signaling channel */
#define AVDTP_DISCOVER          0x01
#define AVDTP_GET_CAPABILITIES  0x02
#define AVDTP_SET_CONFIGURATION 0x03
#define AVDTP_OPEN              0x06
#define AVDTP_START             0x07
#define AVDTP_CLOSE             0x08
#define AVDTP_SUSPEND           0x09
#define AVDTP_ABORT             0x0A

static int send_request(struct avdtp *session, gboolean priority,
    struct avdtp_stream *stream, uint8_t signal_id,
    void *buffer, size_t size);

static void avdtp_sep_set_state(struct avdtp *session,
    struct avdtp_local_sep *sep, avdtp_state_t state);
```

### 2.2 A2DP Streaming Data Flow

The end-to-end A2DP connection sequence proceeds as follows:

1. BlueZ discovers remote SEPs by sending `AVDTP_DISCOVER` on the signaling L2CAP channel (PSM 25).
2. `AVDTP_GET_CAPABILITIES` queries each SEP's codec capabilities (codec type, supported bitpools, channel modes).
3. The source selects a configuration and sends `AVDTP_SET_CONFIGURATION`; both endpoints transition `IDLE → CONFIGURED`.
4. `AVDTP_OPEN` creates the media transport L2CAP channel; state advances to `OPEN`.
5. `AVDTP_START` begins streaming; state reaches `STREAMING`.
6. The audio server calls `org.bluez.MediaTransport1.Acquire()` over D-Bus to obtain a raw socket file descriptor plus `read_mtu`/`write_mtu` values.
7. The server encodes PCM to the negotiated codec and writes RTP-packetized frames to the fd.
8. `Release()` or `AVDTP_CLOSE`/`AVDTP_SUSPEND` reverse the flow.

[Source](https://deepwiki.com/bluez/bluez/4.2-a2dp)

### 2.3 SBC Codec Parameters

SBC (Sub-Band Codec) is specified by the A2DP specification Appendix B. Its configuration bitmap carried in the AVDTP `SET_CONFIGURATION` payload covers four parameter groups:

- **Sampling frequency**: 16/32/44.1/48 kHz
- **Channel mode**: Mono, Dual Channel, Stereo, Joint Stereo (JS exploits inter-channel correlation for better compression)
- **Block length**: 4, 8, 12, or 16 blocks per frame
- **Subbands**: 4 or 8 subbands
- **Allocation method**: Loudness (perceptual weighting) or SNR (flat bit allocation)
- **Bitpool**: 2–250 (source-selectable, higher = better quality)

Higher bitpool values produce higher bitrate and quality. At 44.1 kHz with 16 blocks, 8 subbands, and the "high quality" bitpool of 53, the stream runs at approximately 328 kbps. SBC-XQ uses a bitpool of 76 for Joint Stereo at 44.1 kHz, reaching approximately 453 kbps at near-transparent quality. [Source](https://fossies.org/linux/pipewire/spa/plugins/bluez5/README-SBC-XQ.md)

### 2.4 Codec Landscape

SBC is the mandatory baseline codec, using sub-band coding at up to 320 kbps. The optional codec ecosystem has expanded substantially:

| Codec | Bitrate | Latency | Notes |
|---|---|---|---|
| SBC | 127–345 kbps | ~150 ms | Mandatory; SBC-XQ uses high bitpool ≈ transparent |
| aptX | 352 kbps | ~80 ms | Qualcomm proprietary |
| aptX HD | 576 kbps | ~130 ms | 24-bit audio |
| aptX LL | 352 kbps | <40 ms | Gaming-oriented low-latency variant |
| AAC | Variable | ~150 ms | Via fdk-aac; quality depends on encoder |
| LDAC | 330/660/990 kbps | ~200 ms | Sony; three quality modes |
| FastStream | ~212 kbps | 50–70 ms | Low-latency bidirectional SBC |
| Opus (PW) | 32–320 kbps | Variable | PipeWire-specific multichannel extension |
| LC3Plus H3 | Variable | Variable | LE Audio codec over A2DP (transitional) |

SBC-XQ, the high-bitpool SBC variant that approaches transparency, is documented in the PipeWire bluez5 README. [Source](https://fossies.org/linux/pipewire/spa/plugins/bluez5/README-SBC-XQ.md)

---

## 3. HFP and HSP: Hands-Free and Headset Profiles

HFP (Hands-Free Profile) provides full call control including status indicators, voice dialing, and noise reduction. HSP (Headset Profile) is a simpler predecessor. Both use a RFCOMM channel for AT command signaling and a separate SCO/eSCO link for raw voice audio.

### 3.1 Service Level Connection Establishment

The SLC (Service Level Connection) setup over RFCOMM follows this AT command sequence: [Source](https://circuitlabs.net/bluetooth-hfp-implementation/)

```
/* HFP SLC establishment — RFCOMM channel, HF side initiates */

/* Step 1: Feature exchange */
AT+BRSF=<hf_features>   /* bitmask: bit0=3way, bit3=wideband(mSBC), etc. */
/* AG responds: */
+BRSF:<ag_features>\r\nOK

/* Step 2: Codec negotiation (if both advertise wideband, bit3 in BRSF) */
AT+BAC=1,2               /* HF supports CVSD(1) and mSBC(2) */
OK

/* Steps 3-4: Indicator negotiation */
AT+CIND=?                /* query indicator support */
AT+CIND?                 /* query current indicator values */

/* Step 5: Enable indicator updates */
AT+CMER=3,0,0,1

/* Step 6: Three-way call support (optional) */
AT+CHLD=?
```

The SCO/eSCO audio link is established separately after the SLC:

```
/* AG initiates wideband codec connection */
+BCS:2                   /* AG proposes mSBC (codec_id=2) */
AT+BCS=2                 /* HF confirms mSBC */
/* eSCO link opens with mSBC, 16 kHz sampling */

/* Volume control */
AT+VGS=7                 /* Set speaker gain 0–15 */
AT+VGM=7                 /* Set microphone gain 0–15 */

/* Call control */
ATD<number>;             /* Originate call */
ATA                      /* Answer incoming call */
AT+CHUP                  /* Hang up */
```

### 3.2 mSBC Wideband Speech

HFP 1.6 introduced wideband speech using mSBC (modified SBC) at 16 kHz, mandatory in HFP 1.8. CVSD (8 kHz) is always the mandatory fallback. The codec is negotiated via `AT+BAC` (supported codecs list) and the `+BCS`/`AT+BCS` exchange. mSBC runs over eSCO with packet type `EV3` (2-EV3, 3-EV3 for error correction). Recent kernel patches in the 6.x series addressed reliability issues with Intel and MediaTek USB controllers on SCO/eSCO. [Source](https://github.com/bluez/bluez/issues/1490)

The `BTPROTO_SCO` socket carries raw codec-encoded voice frames. For mSBC, each eSCO packet contains 60 bytes of compressed audio representing 7.5 ms at 16 kHz.

---

## 4. AVRCP: Remote Control and Media Browsing

AVRCP (Audio/Video Remote Control Profile) sits over AVCTP (Audio/Video Control Transport Protocol) on L2CAP PSM 23 (control) and PSM 27 (browsing). BlueZ implements AVRCP version 1.6 with both Target (TG) and Controller (CT) roles. [Source](https://deepwiki.com/bluez/bluez/4.3-avrcp)

### 4.1 Target and Controller Roles

The **Target** (TG) is the device being controlled — typically the audio source (smartphone, computer). It registers an SDP record advertising supported feature categories (Category 1: Player/Recorder, Category 2: Monitor/Amplifier, Category 3: Tuner, Category 4: Menu). The **Controller** (CT) is the device issuing commands — typically the headset or speaker.

BlueZ AVRCP is implemented across two files: `profiles/audio/avrcp.c` (PDU handling, metadata, player browsing) and `profiles/audio/avctp.c` (transport and pass-through framing).

### 4.2 AVC Pass-Through and PDU Handlers

Pass-through commands map remote buttons to Linux `uinput` events via BlueZ's input plugin. `AVC opcode 0x44` (PLAY) maps to `KEY_PLAYCD`; similar mappings exist for PAUSE, STOP, NEXT, PREVIOUS, VOLUME_UP, VOLUME_DOWN.

Key AVRCP PDU handlers:

| PDU ID | Name | Description |
|---|---|---|
| `0x10` | `GET_CAPABILITIES` | Supported events list |
| `0x20` | `GET_ELEMENT_ATTRIBUTES` | Track metadata (title, artist, album, duration) |
| `0x30` | `GET_PLAY_STATUS` | Playback state and position |
| `0x31` | `REGISTER_NOTIFICATION` | Subscribe to event notifications |
| `0x71` | `GET_FOLDER_ITEMS` | Browsing: list items in a folder |
| `0x72` | `GET_ITEM_ATTRIBUTES` | Per-item metadata during browse |
| `0x74` | `CHANGE_PATH` | Navigate folder hierarchy |

Registered notifications include: track change, playback status change, playback position, volume change, and available players. BlueZ exposes the aggregated state to applications via `org.bluez.MediaPlayer1` (play/pause/stop/next/previous methods, Shuffle/Repeat properties) and `org.bluez.MediaControl1` for basic pass-through control.

### 4.3 Volume Synchronization

AVRCP 1.4+ supports absolute volume control. The TG and CT negotiate `EVENT_VOLUME_CHANGED` notification; volume changes on either side propagate bidirectionally. PipeWire/WirePlumber hooks into this to keep system volume synchronized with the remote device's reported level.

---

## 5. PipeWire as Bluetooth Audio Backend

PipeWire replaced both PulseAudio's `module-bluetooth` and the standalone `bluealsa` daemon as the primary Bluetooth audio backend on modern Linux desktops. The integration is provided by the SPA (Simple Plugin API) plugin at `spa/plugins/bluez5/`. [Source](https://github.com/PipeWire/pipewire/blob/master/spa/plugins/bluez5/bluez5-dbus.c)

### 5.1 The spa_bt_monitor and D-Bus Integration

The central data structure in the bluez5 SPA plugin is `spa_bt_monitor`, which maintains complete state of all BlueZ objects:

```c
/* spa/plugins/bluez5/bluez5-dbus.c */
struct spa_bt_monitor {
    struct spa_handle handle;
    struct spa_device device;
    struct spa_log    *log;
    struct spa_loop   *main_loop;
    struct spa_dbus   *dbus;
    DBusConnection    *conn;

    struct spa_list adapter_list;        /* org.bluez.Adapter1 objects */
    struct spa_list device_list;         /* org.bluez.Device1 objects */
    struct spa_list remote_endpoint_list;/* org.bluez.MediaEndpoint1 on remote */
    struct spa_list transport_list;      /* org.bluez.MediaTransport1 objects */

    const struct media_codec * const *media_codecs; /* registered codec array */
    enum spa_bt_profile enabled_profiles;           /* bitmask of active roles */
};
```

The plugin subscribes to D-Bus signals: `InterfacesAdded` and `InterfacesRemoved` on `org.freedesktop.DBus.ObjectManager` to track BlueZ's object tree. When a new `MediaTransport1` appears (a device connects with A2DP or BAP), the plugin creates a corresponding PipeWire node.

### 5.2 Codec Plugin Interface

Each codec is a plugin implementing the `media_codec` interface: [Source](https://github.com/PipeWire/pipewire/blob/0908588d0c1ef3c050103c3d6f8a2f57cf091fb9/spa/plugins/bluez5/a2dp-codecs.h)

```c
/* spa/plugins/bluez5/a2dp-codecs.h */
struct media_codec {
    uint8_t      codec_id;
    a2dp_vendor_codec_t vendor;
    const char  *name;
    const char  *description;
    const struct spa_dict *info;
    size_t       send_buf_size;

    /* Capability advertisement to BlueZ endpoint */
    int (*fill_caps)(const struct media_codec *codec, uint32_t flags,
        uint8_t caps[A2DP_MAX_CAPS_SIZE]);

    /* Select codec config from remote capabilities */
    int (*select_config)(const struct media_codec *codec, uint32_t flags,
        const void *caps, size_t caps_size,
        const struct media_codec_audio_info *info,
        const struct spa_dict *settings,
        uint8_t config[A2DP_MAX_CAPS_SIZE]);

    /* Validate configuration blob */
    int (*validate_config)(const struct media_codec *codec, uint32_t flags,
        const void *caps, size_t caps_size, struct spa_audio_info *info);

    /* Codec instance lifecycle */
    void *(*init)(const struct media_codec *codec, uint32_t flags,
        void *config, size_t config_size,
        const struct spa_audio_info *info, void *props, size_t mtu);
    void  (*deinit)(void *data);

    /* Encoding path */
    int (*start_encode)(void *data, void *dst, size_t *dst_size,
        uint16_t *seqnum, uint32_t *timestamp);
    int (*encode)(void *data, const void *src, size_t src_size,
        void *dst, size_t dst_size, size_t *dst_out, int *need_flush);

    /* Adaptive Bitrate control */
    int (*abr_process)(void *data, size_t unsent);
    int (*reduce_bitpool)(void *data);
    int (*increase_bitpool)(void *data);
};
```

`abr_process()` is called by the transport loop to adapt bitrate based on send-queue depth, implementing the Adaptive Bitrate (ABR) mechanism that LDAC and SBC-XQ use to maintain quality under varying RF conditions.

### 5.3 BlueZ D-Bus Media Endpoint Protocol

When PipeWire registers a codec endpoint with BlueZ, it implements `org.bluez.MediaEndpoint1`. BlueZ calls back into this interface during connection: [Source](https://man.archlinux.org/man/extra/bluez-utils/org.bluez.MediaEndpoint.5.en)

```
/* org.bluez.MediaEndpoint1 — implemented by audio server (PipeWire) */
SetConfiguration(object transport, dict properties)
    /* Called when remote device selects this endpoint */
SelectConfiguration(array{byte} capabilities) -> array{byte}
    /* BlueZ asks server to select config from remote caps */
SelectProperties(dict capabilities) -> dict {Capabilities, Metadata, QoS, ...}
    /* BAP/ISO endpoints only — returns QoS parameters */
ClearConfiguration(object transport)
    /* Transport being torn down */
Reconfigure(dict properties)
    /* BAP unicast reconfigure */
Release()
    /* Endpoint being unregistered */
```

`org.bluez.MediaTransport1.Acquire()` returns a tuple `(fd, uint16 imtu, uint16 omtu)` — the raw socket and MTUs for reading and writing encoded audio frames.

### 5.3b Transport Lifecycle and Deferred Release

The `spa_bt_monitor` plugin maintains a **1-second deferred release timer** on media transports. When a transport moves from `active` to `idle` (for example, due to application disconnect), the plugin does not immediately call `org.bluez.MediaTransport1.Release()`. Instead, it schedules a 1-second timer; if the transport is re-acquired within that window (common during profile switches or application restarts), the timer is cancelled and no BlueZ roundtrip occurs. This avoids spurious reconnect failures during rapid profile switches. The timer fires via `spa_loop_utils_add_timer()` on the main loop. [Source](https://github.com/PipeWire/pipewire/blob/master/spa/plugins/bluez5/bluez5-dbus.c)

When `Acquire()` is called, BlueZ returns the raw socket fd and `imtu`/`omtu` values. The transport io thread calls `write()` to push RTP-packetized codec frames, sized to `omtu` or smaller; the MTU depends on the negotiated L2CAP channel and controller capabilities. The `abr_process()` callback is invoked each processing quantum if unsent bytes in the kernel socket buffer exceed a threshold, triggering `reduce_bitpool()` to lower SBC bitrate and ease congestion under RF stress.

### 5.4 WirePlumber Configuration

WirePlumber is the session manager that applies policy and routing on top of PipeWire nodes. Bluetooth-specific defaults live in `monitor.bluez.properties`: [Source](https://pipewire.pages.freedesktop.org/wireplumber/daemon/configuration/bluetooth.html)

```lua
-- /etc/wireplumber/bluetooth.conf (or wireplumber.conf.d)
monitor.bluez.properties = {
  bluez5.roles = [ a2dp_sink a2dp_source bap_sink bap_source
                   hfp_hf hfp_ag ],
  bluez5.codecs = [ sbc sbc_xq aac ldac aptx aptx_hd aptx_ll
                    faststream opus_05 lc3plus_h3 ],
  bluez5.enable-msbc         = true,
  bluez5.enable-sbc-xq       = true,
  bluez5.hfphsp-backend      = native,  -- or: ofono, hsphfpd
  bluez5.a2dp.ldac.quality   = auto,    -- or: hq, mq, sq, abr
  bluez5.default.rate        = 48000,
}
```

`bluez5.hfphsp-backend = native` means PipeWire handles HFP AT commands directly without requiring oFono. Setting it to `ofono` delegates call control to the oFono telephony daemon.

When a Bluetooth device connects with A2DP, PipeWire creates a sink node named `bluez_output.<addr>.a2dp-sink` (e.g., `bluez_output.02_11_45_A0_B3_27.a2dp-sink`). WirePlumber matches this against `monitor.bluez.rules` to apply per-device settings and links it into the audio graph.

---

## 6. LE Audio: BLE-Based Audio

LE Audio is a fundamental redesign of Bluetooth audio on top of Bluetooth Low Energy (BLE), introduced in Bluetooth 5.2. It replaces the classic BR/EDR SCO link with **isochronous channels** over BLE, and mandates the LC3 codec. [Source](https://www.freecodecamp.org/news/the-bluetooth-le-audio-handbook/)

### 6.1 Isochronous Channel Types

Two channel types carry audio:

**CIS (Connected Isochronous Stream)**: Point-to-point unicast audio between two paired devices. CIS replaces SCO/eSCO for headset audio. A CIG (Connected Isochronous Group) can contain multiple CIS for synchronized stereo or left/right earbud coordination.

**BIS (Broadcast Isochronous Stream)**: One-to-many broadcast audio. A BIG (Broadcast Isochronous Group) containing one or more BIS is received by any number of Sync Receivers without pairing. This is the mechanism underlying **Auracast** (the Bluetooth SIG branding for public broadcast audio).

### 6.2 Auracast Transmission Chain

An Auracast transmitter executes:

1. **Extended Advertising**: broadcasts the Broadcast Audio Announcement service UUID `0x1852` and Public Broadcast Announcement UUID `0x1856` in `ADV_EXT_IND` packets.
2. **Periodic Advertising**: carries the Basic Audio Announcement service UUID `0x1851` with the **BASE** (Broadcast Audio Source Endpoint) structure describing codec configuration and BIG parameters.
3. **BIG creation**: the controller creates the Broadcast Isochronous Group; each BIS carries LC3-encoded audio. Encryption is optional; when enabled, a 16-byte `Broadcast_Code` must be shared out-of-band.

[Source](https://www.collabora.com/news-and-blog/blog/2026/05/05/bluez-powered-auracast-broadcasting-on-genio-700/)

### 6.3 CIG and BIG Kernel Structures

The ISO socket API exposes CIG/BIS configuration through `setsockopt()` on `BTPROTO_ISO` sockets using `SO_BLUETOOTH` / `BT_ISO_QOS`. For CIS unicast, the caller sets ISO interval, SDU size, PHY, retransmission number, and maximum transport latency before calling `connect()`. For BIS broadcast, the caller configures BIG parameters including the optional 16-byte broadcast code before `bind()` and `listen()`.

> **Note: needs verification.** The exact `bt_iso_qos` structure layout (whether SDU/RTN/latency fields live directly under `.ucast` or inside nested `in`/`out` sub-structs of type `bt_iso_io_qos`) changed between kernel 6.0 and 6.4. Consult `include/net/bluetooth/bluetooth.h` in your target kernel before writing ISO socket code directly; the BlueZ library wrappers provide a stable abstraction layer across kernel versions. [Source](https://www.collabora.com/news-and-blog/blog/2025/11/24/implementing-bluetooth-le-audio-and-auracast-on-linux-systems/)

### 6.4 LE Audio Protocol Stack

```
Application (PipeWire BAP nodes)
    │
BAP (Basic Audio Profile)
    │
ASCS / PACS / BASS  (GATT services)
    │
ISO socket (BTPROTO_ISO, kernel 6.4+)
    │
BLE Controller (CIS/BIS, kernel HCI ISO layer)
```

The LE Audio profile stack builds upward from BAP: TMAP (Telephony and Media Audio Profile for earbuds), HAP (Hearing Access Profile), GMAP (Gaming Audio Profile, targeting <30 ms end-to-end), and CAP (Coordinated Audio Profile for synchronized sets) all rely on BAP as their foundation. [Source](https://novelbits.io/bluetooth-le-audio-auracast-profiles/)

### 6.5 QoS Presets

BAP defines standard QoS preset codes that encode sample rate, frame duration, retransmission count, and target latency:

| Preset | Rate | Frame | RTN | Transport Latency |
|---|---|---|---|---|
| `8_1_1` | 8 kHz | 7.5 ms | 1 | ~8 ms |
| `16_2_2` | 16 kHz | 10 ms | 2 | ~40 ms |
| `48_2_2` | 48 kHz | 10 ms | 2 | ~120 ms (high quality music) |
| `48_1_1` | 48 kHz | 7.5 ms | 1 | ~15 ms (gaming low-latency) |

GMAP targets end-to-end latency below 30 ms using short frames and minimal retransmissions.

---

## 7. BlueZ LE Audio Support

BlueZ 5.86 (tested with kernel 6.18.5, PipeWire 1.6.0, WirePlumber 0.5.13) implements: BAP, BASS, VCP (Volume Control Profile), MCP (Media Control Profile), MICP, CCP, CSIP, and TMAP. [Source](https://www.collabora.com/news-and-blog/blog/2025/11/24/implementing-bluetooth-le-audio-and-auracast-on-linux-systems/)

### 7.1 Enabling LE Audio

LE Audio requires explicit opt-in through two mechanisms:

```ini
# /etc/bluetooth/main.conf
[General]
Experimental = true
KernelExperimental = 6fbaf188-05e0-496a-9885-d6ddfdb4e03e
```

The UUID `6fbaf188-05e0-496a-9885-d6ddfdb4e03e` identifies the ISO socket kernel experimental feature. Without it, the BTPROTO_ISO socket returns `EPROTONOSUPPORT`.

Controller capabilities must be verified in `bluetoothctl`:

```
# bluetoothctl
[bluetooth]# show
...
Features:
    cis-central       <- required for unicast CIS (headset audio)
    cis-peripheral    <- required for peripheral CIS role
    iso-broadcaster   <- required for Auracast transmit
    sync-receiver     <- required for Auracast receive
```

Controller support is the primary limiting factor: the Realtek RTL8852BE returns `ENOSYS` (errno 38, "Function not implemented") on ISO connection attempts, while Intel AX210 and MediaTek MT7921 work reliably.

### 7.2 GATT Service Architecture

BlueZ exposes LE Audio capabilities through three core GATT services:

**PACS (Published Audio Capabilities Service)**: Each device publishes its supported codecs, supported sampling rates, frame durations, and channel counts as PAC (Published Audio Capabilities) records over BLE GATT. BlueZ reads these on connection and populates the `SelectProperties` response for `MediaEndpoint1`.

**ASCS (Audio Stream Endpoint Control Service)**: Manages ASE (Audio Stream Endpoint) state machines on the remote device. ASE states parallel AVDTP: Idle → Codec Configured → QoS Configured → Enabling → Streaming → Disabling → Releasing.

**BASS (Broadcast Audio Scan Service)**: Implements the Scan Delegator/Broadcast Assistant pattern. A Scan Delegator (e.g., hearing aid) offloads the work of scanning for Auracast transmitters to a Broadcast Assistant (e.g., smartphone or Linux computer), which writes `Broadcast_Receive_State` characteristics to instruct the delegator.

### 7.2b ASE State Machine

The ASE (Audio Stream Endpoint) state machine on the remote device parallels AVDTP. BlueZ drives state transitions by writing to ASCS control point characteristics over GATT:

| ASE State | ASCS Operation | Description |
|---|---|---|
| Idle | — | No codec configured |
| Codec Configured | Config Codec | Codec + frame parameters agreed |
| QoS Configured | Config QoS | ISO interval, SDU, RTN, latency set |
| Enabling | Enable | CIS being established |
| Streaming | — | ISO data flowing |
| Disabling | Disable | Stopping stream |
| Releasing | Release | Returning to Idle |

PipeWire drives this state machine through `SelectProperties` / `SetConfiguration` callbacks on the BAP `MediaEndpoint1` D-Bus interface. The QoS dictionary returned by `SelectProperties` maps directly onto the ASCS QoS configuration. [Source](https://novelbits.io/bluetooth-le-audio-auracast-profiles/)

### 7.3 BAP D-Bus Integration

For ISO/BAP endpoints, `org.bluez.MediaEndpoint1.SelectProperties` returns a richer dictionary than the classic A2DP `SelectConfiguration`:

```
SelectProperties(dict capabilities) -> dict:
  {
    "Capabilities": array{byte},   /* LC3 codec config */
    "Metadata":     array{byte},   /* LTV-encoded metadata */
    "QoS": dict {
        "Framing":  uint8,    /* 0=unframed, 1=framed */
        "PHY":      uint8,    /* 0x02 = 2M PHY */
        "SDU":      uint16,   /* max SDU size in bytes */
        "RTN":      uint8,    /* retransmission number */
        "Latency":  uint16,   /* max transport latency ms */
        "Delay":    uint32,   /* presentation delay µs */
    }
  }
```

PipeWire creates BAP sink and source nodes alongside A2DP nodes when the remote device advertises PACS. WirePlumber selects between BAP and A2DP profiles based on device capabilities and session policy.

BlueZ SIG qualification for the BlueZ + PipeWire LE Audio stack was expected to complete in 2026. [Source](https://osseu2025.sched.com/event/25Vs2)

---

## 8. LC3 Codec Deep Dive

LC3 (Low Complexity Communication Codec) is specified in ISO 11073-20702 (bluetooth profile numbering) and defined in the Bluetooth SIG LC3 specification. It is mandatory for all LE Audio devices. The reference open-source implementation is `google/liblc3`. [Source](https://en.wikipedia.org/wiki/LC3_(codec))

### 8.1 Algorithmic Pipeline

LC3 processes audio in frames of 7.5 ms or 10 ms. At 48 kHz with 10 ms frames, each frame contains 480 samples. The encoder pipeline has six stages:

1. **Preprocessing and MDCT**: A low-delay MDCT variant (MDCT-LD) with 50% overlap windowing produces N/2 spectral lines. The "low delay" variant reduces look-ahead compared to standard transform coding, contributing to LC3's latency advantage over AAC.

2. **Spectral Noise Shaping (SNS)**: The spectrum is divided into 64 bands. A psychoacoustic model analyzes masking thresholds and applies per-band gain shaping. The gain shape is quantized using a 38-bit vector quantizer and transmitted in the bitstream, allowing the decoder to reproduce the shaped noise floor.

3. **Long-Term Predictor Filter (LTPF)**: Pitch analysis on the time-domain input estimates the fundamental frequency of voiced speech. A pitch lag and gain are encoded; the decoder applies an adaptive predictor to reconstruct harmonic structure, improving quality for speech at low bitrates.

4. **Temporal Noise Shaping (TNS)**: An FIR filter applied in the spectral domain shapes quantization noise within subframes, improving handling of transient signals and pre-echo artifacts.

5. **Spectral Quantization**: Bit allocation across MDCT lines uses a combination of global gain and per-band fine gain. Scalar quantization followed by arithmetic (range) coding achieves entropy-efficient bitstream packing.

6. **Bitstream Assembly**: Header, SNS parameters, LTPF parameters, TNS coefficients, and quantized spectral data are packed into the output frame.

[Source](https://headphoneguidepro.com/bluetooth-le-audio-and-the-lc3-codec-a-technical-deep-dive/)

### 8.2 Supported Configurations

LC3 supports: sample rates 8/16/24/32/44.1/48 kHz; PCM formats 16/24/32-bit; frame durations 7.5 ms and 10 ms. The LC3+ extension (LC3Plus) adds 2.5 ms and 5 ms frames and a high-resolution mode at 96 kHz. Bitrate ranges:

| Sample Rate | Min Bitrate | Quality Notes |
|---|---|---|
| 8 kHz | 16 kbps | Narrowband voice |
| 16 kHz | 32 kbps | Wideband speech, acceptable |
| 24 kHz | 64 kbps | Super-wideband, high quality |
| 48 kHz | 96–160 kbps | Fullband, near-transparent |
| 48 kHz | 320 kbps | Maximum rate |

### 8.3 liblc3 API

The `google/liblc3` library provides the reference encoder and decoder: [Source](https://github.com/google/liblc3/blob/main/include/lc3.h)

```c
/* include/lc3.h — liblc3 reference implementation */

/* Query memory requirement — allocate this before setup */
unsigned lc3_encoder_size(int dt_us, int sr_hz);
/* dt_us: 7500 or 10000 (LC3); also 2500, 5000 for LC3+ */
/* sr_hz: 8000, 16000, 24000, 32000, 44100, 48000 */

/* Initialize encoder into caller-allocated buffer */
lc3_encoder_t lc3_setup_encoder(
    int dt_us,      /* frame duration in microseconds */
    int sr_hz,      /* encoded output sample rate */
    int sr_pcm_hz,  /* PCM input rate; resampling applied if different */
    void *mem);     /* caller buffer of lc3_encoder_size() bytes */

/* Encode one frame */
int lc3_encode(
    lc3_encoder_t encoder,
    enum lc3_pcm_format fmt,  /* LC3_PCM_FORMAT_S16/S24/S24_3LE/FLOAT */
    const void *pcm,
    int stride,               /* samples between channels (1 = interleaved mono) */
    int nbytes,               /* output frame size in bytes — controls bitrate */
    void *out);               /* output buffer */
/* Returns 0 on success */

/* LC3+ high-resolution mode (adds hrmode parameter) */
lc3_encoder_t lc3_hr_setup_encoder(
    bool hrmode,    /* true enables 96 kHz support */
    int dt_us, int sr_hz, int sr_pcm_hz, void *mem);

/* Decoder API */
unsigned lc3_decoder_size(int dt_us, int sr_hz);
lc3_decoder_t lc3_setup_decoder(int dt_us, int sr_hz, int sr_pcm_hz, void *mem);

int lc3_decode(
    lc3_decoder_t decoder,
    const void *in,           /* NULL triggers packet loss concealment */
    int nbytes,
    enum lc3_pcm_format fmt,
    void *pcm,
    int stride);
/* Returns: 0=ok, 1=PLC active (packet loss), -1=error */
```

Memory footprint is modest: `lc3_encoder_size(7500, 48000)` returns approximately 2 KB per channel, making LC3 viable on microcontrollers.

### 8.4 LC3 vs. SBC Comparison

LC3 achieves near-SBC-XQ quality at one-third the bitrate, with lower encoding complexity than AAC. Its fixed-frame-duration design yields predictable latency — 7.5 ms or 10 ms per frame — without the variable look-ahead of AAC's psychoacoustic model. The standardized royalty-free status (required for Bluetooth SIG certification) distinguishes it from aptX and LDAC. [Source](https://github.com/google/liblc3/blob/main/README.md)

---

## 9. Audio Policy and Routing

### 9.1 WirePlumber Lua Policy Engine

WirePlumber 0.5.x uses a Lua-based event dispatcher to implement session policy. The event system replaced the older flat Lua script hooks: policy rules subscribe to specific node/device events and react with linking or property-change actions.

Bluetooth device and node matching in `monitor.bluez.rules`:

```lua
-- Example: force LDAC high-quality for a specific device
-- (from WirePlumber documentation)
monitor.bluez.rules = [
  {
    matches = [
      { "device.name" = "~bluez_card.*" }
    ]
    actions = {
      update-props = {
        "bluez5.a2dp.ldac.quality" = "hq"
      }
    }
  }
  {
    matches = [
      { "node.name" = "~bluez_output.*" }
    ]
    actions = {
      update-props = {
        "session.suspend-timeout-seconds" = 0
      }
    }
  }
]
```

[Source](https://github.com/PipeWire/wireplumber/blob/master/src/config/wireplumber.conf.d.examples/bluetooth.conf)

### 9.2 Automatic A2DP-to-HFP Profile Switching

When a capture stream (microphone) opens on an A2DP headset, the headset's microphone is unavailable on the A2DP profile — A2DP is audio-only, unidirectional. WirePlumber detects a new `Audio/Source` node linked to a Bluetooth device currently using `a2dp-sink` and automatically switches the device's profile to `headset-head-unit` (HFP or HSP), which provides both playback and capture via SCO.

This behavior is controlled by the wireplumber setting:

```lua
wireplumber.settings = {
  "bluetooth.autoswitch-to-headset-profile" = true
}
```

After the capture stream closes, WirePlumber reverts the profile to A2DP, restoring higher audio quality. The switch introduces an audible dropout of 1–3 seconds while the new SCO link negotiates.

### 9.3 Per-Device Codec Preferences

Individual devices can have per-codec quality settings overridden via `monitor.bluez.rules` matching on `device.name`. The `bluez5.a2dp.ldac.quality` property accepts `hq` (990 kbps), `mq` (660 kbps), `sq` (330 kbps), and `auto` (ABR-controlled). The `bluez5.enable-sbc-xq` boolean enables the high-bitpool SBC extension when both ends support it.

---

## 10. Latency Analysis

### 10.1 A2DP End-to-End Latency

A2DP latency accumulates across three stages:

| Stage | Duration | Source |
|---|---|---|
| PCM capture and encoder buffer | 10–20 ms | Codec frame size |
| Bluetooth transmission + retransmissions | 30–50 ms | Radio + ACL scheduling |
| Decoder buffer at sink device | 50–100 ms | Sink-side jitter buffer |
| **Total** | **100–200 ms** | Typical measured |

This is tolerable for music but causes visible lip-sync issues in video (humans detect A/V desync above ~80 ms) and makes gaming impractical. Display servers compensate with `PA_PROP_MEDIA_LATENCY` or PipeWire's `clock.min-quantum` adjustments; mpv uses an audio delay correction based on measured Bluetooth latency.

### 10.2 Low-Latency Codec Options

**aptX LL** targets <40 ms end-to-end by reducing the encoder frame size and limiting jitter buffer depth on the receiving headset. It requires hardware support on both the transmitting controller and the headset DAC.

**FastStream** is a low-latency bidirectional SBC profile (supporting both speaker output and microphone input simultaneously) targeting 50–70 ms. It avoids the A2DP→HFP profile switch by providing simultaneous full-duplex audio, at the cost of using SBC at reduced quality for both directions.

### 10.3 LE Audio Latency

LE Audio CIS unicast with 7.5 ms frames and `RTN=1` achieves transport latency of 15–20 ms. The GMAP (Gaming Audio Profile) specification targets end-to-end latency below 30 ms, comparable to wired headphones. This requires controller support for the `48_1_1` QoS preset and a capable audio path on the application side. [Source](https://osseu2025.sched.com/event/25Vs2)

### 10.3b Presentation Delay and Synchronization

LE Audio introduces a **presentation delay** parameter distinct from transport latency. The receiver holds audio in a jitter buffer for exactly `presentation_delay` microseconds before rendering, allowing synchronization across multiple receivers in a coordinated set (e.g., left and right earbuds). The presentation delay is negotiated during QoS configuration via the `Delay` field in the BAP QoS dict. For music, typical values are 20,000–40,000 µs (20–40 ms); for gaming, 5,000–10,000 µs. All devices in a CSIP (Coordinated Set Identification Profile) set use the same presentation delay, achieving sample-accurate lip sync across speakers. [Source](https://www.freecodecamp.org/news/the-bluetooth-le-audio-handbook/)

### 10.4 Latency Measurement

```bash
# Measure PipeWire graph latency including Bluetooth transport
pipewire-latency

# Observe WirePlumber node quantum and rate
wpctl status | grep -A5 'bluez'

# Enable verbose PipeWire timing logs
PIPEWIRE_DEBUG=4 pipewire 2>&1 | grep -i latency

# btmon statistical analysis (requires gnuplot)
btmon -w /tmp/hci.btsnoop -i 0
btmon -a /tmp/hci.btsnoop      # generates per-connection latency stats
```

The `pipewire-latency` tool reports the configured quantum size (default 1024 samples at 48 kHz = ~21 ms) plus any additional Bluetooth transport delay reported by the transport node.

---

## 11. oFono Integration

oFono is a separate telephony daemon that implements the full HFP Audio Gateway (AG) role — the phone side of the HFP connection. It manages modem hardware, call state, and call routing independently of PipeWire. [Source](https://github.com/arkq/bluez-alsa/issues/29)

### 11.1 oFono Architecture

oFono registers with BlueZ via `org.bluez.Telephony` D-Bus interface as an HFP AG handler. It implements the AT command responder for calls originated or received on ModemManager-managed cellular modems. The chain for a VoIP or cellular call through a Bluetooth headset:

```
Application (linphone/Jami)
    │
oFono D-Bus API (org.ofono.VoiceCall)
    │
oFono HFP AG plugin → RFCOMM AT commands → headset
    │
oFono HFP audio → BlueZ SCO socket
    │
PipeWire (bluez5.hfphsp-backend=ofono) → SCO PCM node
    │
Application RTP audio path
```

ModemManager handles mobile broadband modems (LTE, 5G) and exposes them to oFono; oFono bridges the modem's voice call to the Bluetooth HFP link. For pure VoIP, `linphone` (or any PJSIP-based stack) can interface with oFono via its `org.ofono.VoiceCall` interface to route audio through the Bluetooth headset without needing cellular hardware.

### 11.2 WirePlumber Backend Selection

WirePlumber's `bluez5.hfphsp-backend` property controls which component manages HFP AT command exchange:

| Value | HFP role | Use case |
|---|---|---|
| `native` | HF only | Headset-side audio, no call control |
| `ofono` | AG | Full telephony with oFono |
| `hsphfpd` | HF | Alternative HF daemon |
| `none` | Disabled | HFP/HSP off |

For desktop use without telephony hardware, `native` is sufficient: WirePlumber handles the AT command handshake in-process, enabling headset microphone via SCO while PipeWire exposes the audio to applications.

---

## 12. Debugging

### 12.1 btmon — HCI Packet Capture

`btmon` captures and decodes all HCI traffic between the host stack and the Bluetooth controller, replacing the deprecated `hcidump`. [Source](https://man.archlinux.org/man/extra/bluez-utils/btmon.1.en)

```bash
# Capture all HCI traffic to btsnoop file
btmon -w /tmp/hci.btsnoop

# Replay with wall-clock timestamps
btmon -r /tmp/hci.btsnoop -t

# Statistical latency analysis (requires gnuplot)
btmon -a /tmp/hci.btsnoop

# Capture hci0 only, show raw SCO audio hex
btmon -i 0 -S

# Capture hci0, show raw A2DP frames
btmon -i 0 -A

# Capture hci0, show LE ISO/LE Audio frames
btmon -i 0 -I

# Management channel events only (adapter state changes)
btmon -M
```

**btmon output prefix legend**:

| Prefix | Meaning |
|---|---|
| `<` | HCI command/data: host → controller |
| `>` | HCI event/data: controller → host |
| `@` | MGMT socket traffic |
| `=` | Kernel annotations / system messages |

The btsnoop file format is compatible with Wireshark, enabling graphical AVDTP/AVCTP/L2CAP dissection. btmon also tracks per-connection statistics including connection type (BR-ACL, LE-ACL, BR-SCO, BR-ESCO, LE-ISO) and per-L2CAP-channel packet count, latency distribution, and frame sizes.

### 12.2 bluetoothctl

`bluetoothctl` is the primary interactive interface for pairing, connecting, and inspecting devices:

```bash
bluetoothctl
[bluetooth]# show                     # adapter info and supported features
[bluetooth]# devices                  # list known devices
[bluetooth]# info AA:BB:CC:DD:EE:FF   # device details, connected profiles
[bluetooth]# connect AA:BB:CC:DD:EE:FF
[bluetooth]# disconnect AA:BB:CC:DD:EE:FF
[bluetooth]# -- select-attribute /org/bluez/hci0/dev_XX/sep1/fd0
```

For LE Audio controller feature verification:

```bash
# Non-interactive — list adapter info
bluetoothctl show | grep -A20 "Features"
# Look for: cis-central, cis-peripheral, iso-broadcaster, sync-receiver
```

### 12.3 PipeWire Inspection

```bash
# Session graph: all active nodes, sinks, sources, and routes
wpctl status

# List all PipeWire objects matching bluez
pw-cli list-objects | grep -A10 bluez

# Real-time PipeWire node events (useful during connect/disconnect)
pw-mon

# PipeWire card profiles (equivalent to pactl on PulseAudio)
pactl list cards | grep -A40 'bluez_card'
pactl set-card-profile bluez_card.AA_BB_CC_DD_EE_FF a2dp-sink
pactl set-card-profile bluez_card.AA_BB_CC_DD_EE_FF headset-head-unit
```

### 12.4 Log Inspection

```bash
# PipeWire and WirePlumber logs together
journalctl -u pipewire -u wireplumber -f

# Last 5 minutes with priority filter
journalctl -u pipewire --since -5m -p debug

# Enable maximum PipeWire debug verbosity (run as replacement)
PIPEWIRE_DEBUG=4 pipewire

# WirePlumber with verbose Lua policy tracing
WIREPLUMBER_DEBUG=4 wireplumber
```

### 12.5 Deprecated Tools

`hcitool` is deprecated; use `bluetoothctl` for device management and `btmgmt` for low-level management commands:

```bash
# btmgmt replaces hcitool for controller management
btmgmt info                 # controller info
btmgmt power on             # power adapter
btmgmt find                 # start LE scan
btmgmt find-service         # BR/EDR device inquiry
```

`hcitool scan` and `hcitool dev` still work on most systems but are not maintained and may be removed in future distributions.

---

## Integrations

**Ch38 — PipeWire Audio Graph**: Bluetooth A2DP and BAP devices appear as PipeWire sink/source nodes (`bluez_output.*`, `bluez_input.*`) within the same graph as ALSA PCM devices, USB audio, and virtual nodes. WirePlumber applies the same linking and policy mechanisms to Bluetooth nodes as to any other PipeWire device. The `spa_bt_monitor` SPA plugin registers as a device provider alongside `spa-alsa` and `spa-v4l2` within the PipeWire daemon process.

**Ch38b — ALSA: Legacy bluealsa Path**: Before PipeWire, `bluealsa` (from `arkq/bluez-alsa`) provided ALSA PCM devices backed by BlueZ A2DP and HFP transports. Systems not running PipeWire can still use this path; `bluealsa` registers an ALSA plugin that presents a virtual PCM (`bluealsa:DEV=XX:XX:XX:XX:XX:XX`) to ALSA applications. The `bluez-alsa` issue tracker is a useful source of HFP compatibility notes across controller firmware versions.

**Ch213 — VoIP/Softphone Headset Use**: HFP enables Bluetooth headsets for VoIP clients. The integration chain — linphone or Jami → oFono → BlueZ HFP AG → SCO audio → PipeWire — is covered end-to-end in Ch213, which addresses SIP call setup, RTP media routing, and headset button event handling via AVRCP/uinput.

**Ch216 — Speech Synthesis to Bluetooth Speaker**: Text-to-speech engines (espeak-ng, flite, Piper) write synthesized audio to PipeWire audio sinks. When the default sink is a Bluetooth A2DP device, audio flows through the same codec negotiation and transport path described in this chapter, with WirePlumber managing automatic sink selection and A2DP codec ABR under varying RF conditions.
