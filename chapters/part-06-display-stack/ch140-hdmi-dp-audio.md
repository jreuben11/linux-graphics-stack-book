# Chapter 140: HDMI and DisplayPort Audio on Linux

**Target audiences**: Kernel audio and display driver developers working on the DRM–ALSA bridge, system integrators building Linux HTPC or A/V systems, and users troubleshooting HDMI/DisplayPort audio issues on Linux desktop environments.

---

## Table of Contents

1. [Introduction](#introduction)
2. [HDMI Audio Protocol](#hdmi-audio-protocol)
3. [EDID Audio Capabilities and ELD](#edid-audio-capabilities-and-eld)
4. [drm_audio_component: The DRM–ALSA Bridge](#drm_audio_component-the-drm-alsa-bridge)
5. [ALSA HDA Intel Driver](#alsa-hda-intel-driver)
6. [PipeWire and HDMI Audio](#pipewire-and-hdmi-audio)
7. [HDMI Hotplug and Audio Lifecycle](#hdmi-hotplug-and-audio-lifecycle)
8. [DisplayPort Audio](#displayport-audio)
9. [HDMI CEC](#hdmi-cec)
10. [Integrations](#integrations)

---

## Introduction

HDMI and DisplayPort carry audio alongside video in the same cable. From a Linux kernel perspective, the display subsystem (DRM) owns the physical connection and the EDID-based audio capability discovery, while the audio subsystem (ALSA) owns the audio streaming. A specialised bridge API — `drm_audio_component` — connects them.

When you plug an HDMI cable into a TV or AV receiver, the following must happen: the GPU driver reads the TV's EDID to discover supported audio formats (PCM channels, compressed audio, sample rates), creates a compact summary called the ELD (EDID-Like Data), notifies the ALSA HDA codec driver, which creates PCM device entries in `/dev/snd/`. The user can then play audio through the TV via ALSA, PulseAudio, or PipeWire.

This chapter covers the complete chain: HDMI audio packet protocol, EDID Short Audio Descriptors, the ELD format and how DRM generates it, the `drm_audio_component` vtable that bridges DRM and ALSA, the `snd_hda_codec_hdmi` driver, PipeWire routing, hotplug handling, DisplayPort audio specificities, and HDMI CEC for device control.

[Linux kernel audio documentation](https://www.kernel.org/doc/html/latest/sound/hd-audio/index.html)

---

## HDMI Audio Protocol

### Audio in the TMDS Stream

HDMI carries audio as **Audio Sample Packets** embedded in the TMDS (Transition Minimised Differential Signalling) stream during Video Data Period blanking intervals. The audio data is interleaved with the video pixel data using the Auxiliary Video Information (AVI) InfoFrame and Audio InfoFrame.

Key audio formats supported by HDMI:

| Format | Max Channels | Max Sample Rate | Notes |
|---|---|---|---|
| LPCM (PCM) | 8 channels | 192 kHz / 24-bit | Always supported per HDMI spec |
| AC-3 (Dolby Digital) | 5.1 | 48 kHz | IEC 61937 compressed |
| DTS | 5.1 | 48 kHz | IEC 61937 compressed |
| DTS-HD Master Audio | 7.1 | 192 kHz | High-bandwidth compressed |
| Dolby TrueHD / Atmos | up to 24 objects | 192 kHz | HDMI 2.0+ HBR stream |
| DTS:X | up to 11.1 | 192 kHz | DTS-HD variant |

### Audio Clock Regeneration (ACR)

HDMI's ACR mechanism synchronises the audio clock with the TMDS clock:

```
f_audio = 128 × f_TMDS × (N / CTS)
```

Where:
- `f_TMDS` — TMDS clock frequency (e.g. 148.5 MHz for 1080p60)
- `N` — N value transmitted in ACR packets (e.g. 6144 for 48 kHz)
- `CTS` — Cycle Time Stamp, measured by the sink

For 48 kHz audio on 1080p60:
```
48000 = 128 × 148500000 × (6144 / CTS)
CTS = 128 × 148500000 × 6144 / 48000 = 148500 × 128 / 1000 ≈ 148500
```

This deterministic relationship is why HDMI audio is synchronised to the video frame rate and why audio glitches can occur on mode changes.

### HDMI 2.1 eARC

HDMI 2.1 introduces **eARC** (Enhanced Audio Return Channel) on the ARC pin. Unlike the original ARC (which was limited to 2-channel PCM or 5.1 IEC 61937), eARC supports:
- Full-bandwidth audio: Dolby TrueHD, DTS-HD, Dolby Atmos object-based
- Up to 192 kHz / 24-bit / 32-channel LPCM
- Channel: bidirectional on a dedicated differential pair (pins 13/14 on HDMI 2.1)

Linux eARC support is in development; as of 2025, ALSA supports eARC detection but full object-based audio pass-through depends on the AV receiver and source.

---

## EDID Audio Capabilities and ELD

### EDID CEA-861 Extension

HDMI sinks (TVs, monitors) advertise audio capabilities in the EDID CEA-861 extension block. The `CEA Extension Block` contains **Short Audio Descriptors (SADs)**, each 3 bytes:

```
Byte 0: [7:5] Audio Format Code, [4:3] Reserved, [2:0] Max Channels - 1
Byte 1: Sample rate bits: [6]=192kHz, [5]=176kHz, [4]=96kHz, [3]=88kHz,
                           [2]=48kHz, [1]=44.1kHz, [0]=32kHz
Byte 2: For LPCM: [2]=24-bit, [1]=20-bit, [0]=16-bit
        For compressed: max bitrate in 8 kbps units
```

Audio Format Codes (HDMI 1.4b):
- 1: LPCM
- 2: AC-3
- 3: MPEG-1
- 4: MP3
- 5: MPEG2
- 6: AAC LC
- 7: DTS
- 8: ATRAC
- 9: One Bit Audio
- 10: Enhanced AC-3 (Dolby Atmos)
- 11: DTS-HD
- 12: MAT (Dolby TrueHD)

### ELD (EDID-Like Data)

The kernel condenses EDID audio capabilities into an 84-byte **ELD** structure that the ALSA HDA driver can access efficiently:

```c
/* From include/drm/drm_edid.h */
#define ELD_MAX_SIZE    96

/* ELD byte layout (subset): */
/* [0] ELD version (0x02 = CEA-861-D) */
/* [1] ELD baseline length in DWORDs */
/* [4] Conn_Type[7:6] + S_AI[4] + HDCP[3] + Audio_Sync_Delay[2:0] */
/* [5] Speaker allocation bitmap */
/* [6-7] Port ID (from EDID) */
/* [8-9] Manufacturer ID */
/* [10-11] Product code */
/* [12-..] Monitor name string (null-terminated) */
/* [..] SAD array (3 bytes × sad_count) */
```

```c
/* DRM helper that generates ELD from parsed EDID: */
/* drivers/gpu/drm/drm_edid.c */
void drm_edid_to_eld(struct drm_connector *connector,
                     const struct drm_edid *drm_edid);
```

Inspecting the ELD on a running system:

```bash
# Show ELD for all HDMI/DP audio devices:
for f in /proc/asound/card*/eld*; do
    echo "=== $f ==="; cat "$f"
done

# Example output:
# monitor_present         1
# eld_valid               1
# monitor_name            LG ULTRAFINE
# connection_type         DisplayPort
# eld_version             [0x2] CEA-861D or below
# edid_version            [0x3] CEA-861-B, C or D
# manufacture_id          0x1e6d
# product_id              0x7736
# port_id                 0x0000000000000000
# support_hdcp            0
# support_ai              0
# audio_sync_delay        0
# speakers                [0x00000037] FL/FR LFE FC RL/RR
# sad_count               3
# sad0_coding_type        [0x1] PCM
# sad0_channels           2
# sad0_rates              [0xe0] 192000 96000 48000
# sad0_bits               [0x7] 24 20 16
```

---

## drm_audio_component: The DRM–ALSA Bridge

### Architecture

The `drm_audio_component` API (`include/drm/drm_audio_component.h`) allows GPU display drivers and ALSA HDA codec drivers to communicate without a direct dependency. The GPU driver registers a set of callbacks; the ALSA driver calls them when it needs audio information.

```c
/* include/drm/drm_audio_component.h */
struct drm_audio_component_ops {
    struct module *owner;

    /* Called by ALSA to get GPU power */
    void (*get_power)(struct device *dev, unsigned int port);
    void (*put_power)(struct device *dev, unsigned int port);

    /* Allow codec to override GPU clock wake */
    void (*codec_wake_override)(struct device *dev, bool enable);

    /* Get current CD clock frequency (needed for ACR) */
    int  (*get_cdclk_freq)(struct device *dev);

    /* Synchronise audio sample rate with display rate */
    int  (*sync_audio_rate)(struct device *dev, int port, int pipe, int rate);

    /* Read ELD for the given port/pipe */
    int  (*get_eld)(struct device *dev, int port, int pipe,
                    bool *enabled, unsigned char *buf, int max_bytes);
};

struct drm_audio_component {
    struct device *dev;
    const struct drm_audio_component_ops *ops;
    /* GPU-side audio notifier */
    struct drm_audio_component_audio_ops *audio_ops;
};
```

### Intel i915 Implementation

The i915 DRM driver implements `drm_audio_component` in `drivers/gpu/drm/i915/display/intel_audio.c` [Source](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/i915/display/intel_audio.c):

```c
static const struct drm_audio_component_ops i915_audio_component_ops = {
    .owner          = THIS_MODULE,
    .get_power      = i915_audio_component_get_power,
    .put_power      = i915_audio_component_put_power,
    .codec_wake_override = i915_audio_component_codec_wake_override,
    .get_cdclk_freq = i915_audio_component_get_cdclk_freq,
    .sync_audio_rate = i915_audio_component_sync_audio_rate,
    .get_eld        = i915_audio_component_get_eld,
};
```

When a display is connected or disconnected, i915 notifies the ALSA driver:

```c
/* Called when ELD changes (hotplug, mode change): */
void intel_audio_codec_enable(struct intel_encoder *encoder,
    const struct intel_crtc_state *crtc_state,
    const struct drm_connector_state *conn_state)
{
    /* ... program HDMI audio registers ... */

    /* Notify ALSA via audio component: */
    if (acomp && acomp->audio_ops && acomp->audio_ops->pin_eld_notify)
        acomp->audio_ops->pin_eld_notify(acomp->audio_ops->audio_ptr,
            (int)port, (int)pipe);
}
```

### AMD amdgpu Implementation

AMD's display driver implements the bridge in `drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_audio.c` [Source](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/amd/display/amdgpu_dm/amdgpu_dm_audio.c):

```c
static const struct drm_audio_component_ops amdgpu_dm_audio_component_ops = {
    .get_eld = amdgpu_dm_audio_component_get_eld,
    /* AMD doesn't need power gating callbacks for audio */
};
```

### Component Bind Mechanism

The DRM and ALSA drivers communicate through the Linux component framework (`drivers/base/component.c`):

```c
/* GPU driver side: */
component_add(&pdev->dev, &i915_audio_component_bind_ops);

/* ALSA HDA driver side: */
component_bind_all(&snd_dev, &snd_hdmi_lpe_audio_component_ops);
```

The component framework calls the `bind` callback when both the GPU driver and the ALSA driver are loaded and match. This avoids compile-time or load-time ordering dependencies.

---

## ALSA HDA Intel Driver

### snd_hda_codec_hdmi

HDMI/DP audio on Intel, AMD, and NVIDIA GPUs uses the Intel HDA (High Definition Audio) controller embedded in the GPU. The ALSA driver `snd_hda_intel` (`sound/pci/hda/`) manages the HDA controller, while `snd_hda_codec_hdmi` (`sound/pci/hda/patch_hdmi.c`) [Source](https://elixir.bootlin.com/linux/latest/source/sound/pci/hda/patch_hdmi.c) is the codec patch for HDMI/DP.

```bash
# Check loaded audio modules:
lsmod | grep snd_hda
# snd_hda_codec_hdmi    77824  1
# snd_hda_intel         53248  0
# snd_hda_codec        155648  2 snd_hda_codec_hdmi,snd_hda_intel

# List available PCM devices (HDMI sinks):
aplay -l
# card 0: PCH [HDA Intel PCH], device 3: HDMI 0 [HDMI 0]
#   Subdevices: 1/1
#   Subdevice #0: subdevice #0
# card 0: PCH [HDA Intel PCH], device 7: HDMI 1 [HDMI 1]
```

### PCM Stream per Port

`snd_hda_codec_hdmi` creates one PCM stream per HDMI/DP port:

```c
/* patch_hdmi.c: each port gets its own PCM */
struct hdmi_spec {
    int num_pins;
    struct hdmi_spec_per_pin pins[MAX_HDMI_PINS];
    struct hdmi_spec_per_cvt cvts[MAX_HDMI_CVTS];
    /* One pcm_rec per port: */
    struct hda_pcm pcm_rec[MAX_HDMI_PINS];
};
```

### HDA Verbs for Audio Setup

The HDA codec communicates with the audio DSP via **verbs** — 32-bit commands:

```c
/* Set pin widget control (enable audio output): */
snd_hda_codec_write(codec, pin_nid, 0,
    AC_VERB_SET_PIN_WIDGET_CONTROL, PIN_OUT);

/* Set stream format (48 kHz, 16-bit, 2-channel): */
snd_hda_codec_write(codec, cvt_nid, 0,
    AC_VERB_SET_STREAM_FORMAT,
    0x4011); /* 0x4 = 48kHz, 0x0 = 16-bit, 0x1 = 2ch */

/* Set channel ID for multi-channel (7.1): */
snd_hda_codec_write(codec, pin_nid, 0,
    AC_VERB_SET_CHANNEL_STREAMID,
    (stream_tag << 4) | channel_id);
```

### Testing HDMI Audio Playback

```bash
# Play a test tone to HDMI output:
aplay -D plughw:CARD=NVidia,DEV=0 -f cd /dev/urandom

# Or with a WAV file:
aplay -D hw:0,3 /usr/share/sounds/alsa/Front_Left.wav

# Check HDMI mixer controls:
amixer -c 0 contents | grep -i hdmi

# Enable HDMI audio output (if muted):
amixer -c 0 sset 'IEC958',0 on
```

### AMD HDMI Audio

AMD uses `snd_hda_codec_atihdmi` (in practice, the same `patch_hdmi.c` handles AMD too):

```bash
# AMD: list audio devices
aplay -l | grep AMD
# card 1: Generic [HD-Audio Generic], device 3: HDMI 0 [HDMI 0]
```

### NVIDIA HDMI Audio

NVIDIA's proprietary driver includes `patch_nvhdmi.c`. The open-source Nouveau driver also supports HDMI audio via a similar path.

```bash
# NVIDIA: check PCM devices
aplay -l | grep NV
```

---

## PipeWire and HDMI Audio

### HDMI Detection in PipeWire

PipeWire's ALSA monitor plugin (`spa/plugins/alsa/alsa-monitor.c`) discovers HDMI PCM devices from `/dev/snd/pcm*` and creates PipeWire sinks. WirePlumber watches udev events for card changes and updates the device graph.

```bash
# List all PipeWire audio sinks (including HDMI):
pw-cli list-objects | grep -A5 "type:PipeWire:Interface:Node" | grep -i hdmi
# or:
wpctl status | grep -i hdmi
# Output:
#  *  45. HDMI / DisplayPort [vol: 1.00]
#  *  46. HDMI / DisplayPort 2 [vol: 1.00]

# Check PipeWire's view of HDMI devices:
pw-dump | jq '.[] | select(.props."media.class" == "Audio/Sink") |
    {id: .id, name: .props."node.name"}'
```

### Setting HDMI as Default

```bash
# Set HDMI sink as default:
wpctl set-default 45  # use the HDMI sink ID from wpctl status

# Or with pactl (PulseAudio-compatible layer):
pactl list sinks | grep -A1 HDMI
pactl set-default-sink alsa_output.pci-0000_00_1f.3.hdmi-stereo-extra1
```

### WirePlumber Hotplug

When an HDMI cable is connected/disconnected:

1. HPD interrupt → DRM hotplug → i915 reads EDID → `drm_edid_to_eld`
2. `i915_audio_component_pin_eld_notify` called → ALSA HDA reads new ELD
3. ALSA creates or destroys PCM device → udev event
4. WirePlumber receives udev event via `spa-alsa-udev` monitor
5. WirePlumber creates/removes PipeWire sink for the HDMI port

```bash
# Monitor hotplug events:
udevadm monitor --kernel --subsystem-match=sound
# [kernel] add /devices/pci0000:00/.../sound/card0/pcmC0D3p (sound)
# [kernel] remove /devices/pci0000:00/.../sound/card0/pcmC0D3p (sound)
```

### HDMI Audio Passthrough (IEC 60958 / IEC 61937)

For Dolby Atmos or DTS-HD passthrough from PipeWire to an AV receiver:

```bash
# Check if HDMI output supports IEC 61937:
cat /proc/asound/card0/eld#0.3 | grep sad

# Configure PipeWire for passthrough (requires AES/EBU IEC958 format):
pactl load-module module-alsa-sink device=hw:0,3 \
    format=s16le rate=48000 channels=2 \
    sink_properties=device.description="HDMI-Passthrough"

# mpv with passthrough:
mpv --audio-device=alsa/hw:0,3 --audio-spdif=ac3,dts,dts-hd,truehd movie.mkv
```

### Debug Logging

```bash
# PipeWire debug:
PIPEWIRE_DEBUG=3 pipewire 2>&1 | grep -i hdmi

# WirePlumber debug:
WIREPLUMBER_DEBUG=D wireplumber 2>&1 | grep -i hdmi
```

---

## HDMI Hotplug and Audio Lifecycle

### Hot-Plug Detect (HPD)

HDMI pin 19 (HPD) carries a 5 V signal that the sink asserts when connected. The GPU polls or uses an interrupt on this pin:

```
HPD asserted
 → GPU ISR: drm_helper_hpd_irq_event()
 → drm_kms_helper_hotplug_event()
 → connector->detect() → reads EDID
 → drm_edid_parse() → drm_edid_to_eld()
 → intel_audio_codec_enable()
 → i915_audio_component_pin_eld_notify()
 → snd_hdac_i915_pindpcd_changed()
 → ALSA creates new PCM nodes
```

### Audio Loss on Suspend/Resume

Suspend/resume is a common source of HDMI audio breakage. The sequence:

1. System suspends → GPU runtime PM removes power → HDMI audio codec loses state
2. System resumes → GPU re-initialises → i915 replays display configuration
3. **Problem**: ALSA may not be notified of the new ELD after resume

Fix in userspace:
```bash
# Force ELD re-read after resume:
# Add to /etc/pm/sleep.d/99-hdmi-audio.sh:
#!/bin/sh
case "$1" in
    thaw|post)
        # Reload codec after resume
        echo 1 > /sys/bus/pci/drivers/snd_hda_intel/*/power/control
        ;;
esac
```

### Diagnosing ELD Issues

```bash
# Check if ELD is valid:
cat /proc/asound/card0/eld#0.3 | grep eld_valid
# eld_valid     1   (1 = valid, 0 = monitor not present or ELD unread)

# Low-level HDA verb inspection:
hda-verb /dev/snd/hwC0D0 0x6 GET_PIN_SENSE 0
# 0x80000000 = plugged (bit 31)
# 0x40000000 = HDCP present (bit 30)

# Check audio presence sense:
hdajacksensetest -a  # from alsa-tools package
```

---

## DisplayPort Audio

### DP Audio over Main Link

DisplayPort carries audio on the Main Link as **Audio Stream Packets** during idle periods between video line blanking. Unlike HDMI, DP audio is synchronous with the link clock (no ACR mechanism needed).

```c
/* DPCD Audio Capabilities register: */
/* DP_AUDIO_SINK_CAPS = 0x022 */

uint8_t caps;
drm_dp_dpcd_read(aux, DP_AUDIO_SINK_CAPS, &caps, 1);
/* Bits: [0] PCM, [1] AC-3, [2] MPEG1_L1_L2, [3] MPEG2, [4] AAC_LC,
         [5] DTS, [6] ATRAC */
```

### DP Audio Formats

DP 1.2+ supports:
- 8-channel LPCM at up to 192 kHz / 32-bit
- High-Bit Rate (HBR) audio: up to 32 channels
- IEC 61937 compressed audio (same codecs as HDMI)

DP audio typically outperforms HDMI ARC in bandwidth but is less common in AV receiver setups (most receivers use HDMI).

### MST Audio

With DisplayPort Multi-Stream Transport (MST), each MST sink can have independent audio:

```c
/* ELD per MST port: */
struct drm_dp_mst_port {
    /* ... */
    uint8_t cached_edid;
    /* eld stored per-port, not per-connector */
};
```

Each MST display connected via a daisy-chain or hub receives its own ELD and has its own ALSA PCM device on Linux.

### ARC vs eARC (Return Channel)

| Feature | ARC | eARC |
|---|---|---|
| HDMI version | 1.4 | 2.1 |
| Max audio | 5.1 IEC 61937 | Full TrueHD/DTS-HD |
| Bandwidth | ~1 Mbps | ~37 Mbps |
| Protocol | TMDS carrier detect | Differential pair |

Linux support for ARC is present (the GPU-side ELD handles capabilities), but eARC requires HDMI 2.1 hardware and driver support still being developed in the kernel as of 2025.

### ELD Connection Type

```c
/* drm_edid.h — connection type in ELD byte 4: */
#define DRM_ELD_CONN_TYPE_HDMI  (0 << 2)
#define DRM_ELD_CONN_TYPE_DP    (1 << 2)

int conn_type = drm_eld_get_conn_type(connector->eld);
```

This allows the ALSA driver to distinguish HDMI from DP audio even when both use the same HDA controller.

---

## HDMI CEC

### Consumer Electronics Control

CEC (Consumer Electronics Control) is a single-wire bidirectional bus running on HDMI pin 13. It allows devices in an AV system to command each other: a Linux PC can put the TV in standby, switch it to the PC's input, or respond to TV remote control buttons.

### Linux CEC Subsystem

The CEC subsystem lives in `drivers/media/cec/` [Source](https://elixir.bootlin.com/linux/latest/source/drivers/media/cec):

```bash
# List CEC adapters:
ls /dev/cec*
# /dev/cec0

# Get CEC adapter capabilities:
cec-ctl -d /dev/cec0 --info
# Driver          : i915
# Adapter         : 0000:00:02.0 i915
# Capabilities    : 0x00000037
#   Logical Addresses (0x00000001)
#   Transmit (0x00000002)
#   Passthrough (0x00000004)
#   RC (0x00000010)
#   Monitor All (0x00000020)
#   Error Reporting (0x00000100)
```

### CEC Commands

```bash
# Activate CEC on the adapter (claim logical address):
cec-ctl -d /dev/cec0 --playback

# Send "Image View On" to wake the TV:
cec-ctl -d /dev/cec0 --to 0 --image-view-on

# Send standby to all devices:
cec-ctl -d /dev/cec0 --broadcast --standby

# Request TV's audio status:
cec-ctl -d /dev/cec0 --to 0 --give-audio-status

# Monitor CEC traffic:
cec-follower -d /dev/cec0 -v

# One-touch play (switch TV to this source and play):
cec-ctl -d /dev/cec0 --active-source phys-addr=1.0.0.0
```

### DRM CEC Integration

GPU drivers expose CEC via `drm_connector`:

```c
/* Intel i915 CEC initialisation in intel_hdmi.c: */
void intel_hdmi_cec_adap_init(struct intel_digital_port *dig_port)
{
    struct cec_adapter *adap =
        cec_allocate_adapter(&intel_hdmi_cec_adap_ops, dig_port,
                             "i915", CEC_CAP_DEFAULTS, 1);
    drm_connector_set_cec_adap(&dig_port->hdmi.attached_connector->base,
                                adap);
}
```

### CEC over DisplayPort

DP 1.3+ defines a CEC-over-AUX tunnel (`drm_dp_cec.c` in the kernel). This allows DP→HDMI active adapters to pass CEC through the DP link:

```c
/* drivers/gpu/drm/drm_dp_cec.c */
struct cec_adapter *drm_dp_cec_register_connector(
    struct drm_dp_aux *aux,
    struct drm_connector *connector);
```

### libcec and Applications

[libcec](https://github.com/Pulse-Eight/libcec) provides a platform-independent CEC API. Kodi uses libcec to:
- Turn on the TV and switch to the Kodi input when Kodi starts
- Put the TV in standby when Kodi exits
- Map TV remote CEC keypresses to Kodi navigation actions

```bash
# Install and test:
sudo apt install libcec6 cec-utils
echo "scan" | cec-client -s -d 1 /dev/cec0
```

---

## Roadmap

### Near-term (6–12 months)

- **HDMI 2.1 FRL audio path**: AMD engineer Harry Wentland posted the first HDMI 2.1 FRL (Fixed Rate Link) patches to the Linux kernel mailing list in May 2026, targeting the Linux 7.2 merge window. FRL unlocks 4K 240Hz and 8K 120Hz video modes, and the associated high-bandwidth audio streams (Dolby Atmos, DTS:X at full fidelity) depend on this physical-layer work landing first. [Source](https://www.gamingonlinux.com/2026/05/further-expanded-amd-hdmi-2-1-support-is-coming-to-linux-now-with-frl-and-dsc/)
- **Generic HDMI CEC helpers for DRM bridges**: A patchset (v5 posted to dri-devel in 2025) introduces standardised CEC helpers that integrate with `drm_bridge_connector`, removing the current impossibility of implementing CEC for bridge-based display chains and standardising physical-address handling across drivers. [Source](https://www.mail-archive.com/dri-devel@lists.freedesktop.org/msg539234.html)
- **DRM HDMI audio helpers reused for DisplayPort bridges**: Linux 6.16 includes work to share the DRM HDMI audio helper infrastructure with DisplayPort bridge drivers, reducing duplicated ELD-generation and audio-component plumbing between HDMI and DP paths. Note: needs verification of final merge status.
- **ALSA 1.2.15+ stabilisation**: The ALSA userspace library stable release (1.2.15.3 as of January 2026) continues hardening of the HDA HDMI codec plugin, including improved IEC 61937 pass-through framing for compressed audio formats. [Source](https://embeddedprep.com/complete-guide-of-alsa-linux-audio/)
- **VRR + FRL audio clock recovery**: Until Variable Refresh Rate is reconciled with HDMI 2.1 FRL clock generation, FRL is disabled by default to avoid regressions; near-term patches are expected to resolve the VRR/FRL interaction. Note: needs verification.

### Medium-term (1–3 years)

- **Full eARC object-based audio pass-through**: Linux currently detects eARC-capable connectors but does not expose a full bidirectional eARC channel usable for Dolby Atmos object-based streaming from a Linux media player to an AV receiver. Full kernel eARC support — including the dedicated differential-pair signalling on HDMI 2.1 pins 13/14 — is an open design discussion on the alsa-devel mailing list. Note: needs verification of specific RFC.
- **PipeWire passthrough for compressed HDMI audio (IEC 61937)**: WirePlumber and PipeWire's ALSA sink currently manage PCM routing well, but IEC 61937 compressed pass-through (AC-3, DTS, TrueHD) requires an "S/PDIF RAW" path that bypasses PipeWire's sample-rate converter. Upstream discussions are ongoing about a dedicated passthrough node type. Note: needs verification.
- **DisplayPort 2.1 UHBR audio**: DP 2.1 Ultra-High Bit Rate (UHBR) links (20 Gbps+) require updated ACR (Audio Clock Recovery) tables and audio SDP (Secondary Data Packet) extensions. Kernel DRM DP helpers will need updates once AMD and Intel ship UHBR-capable hardware broadly on Linux. [Source](https://kernelnewbies.org/Linux_6.16)
- **Unified DRM audio topology API**: There is a long-standing desire to expose the full DRM audio connector topology (connectors, mux paths, ELD per sink) as a proper ALSA topology blob, enabling sound servers to enumerate multi-sink setups (e.g., MST with per-port audio) without heuristics. Note: needs verification of RFC status.

### Long-term

- **Kernel-managed eARC CEC co-ordination**: eARC uses a subset of CEC signalling on the ARC channel to negotiate audio format support. Long-term, the kernel CEC and ALSA subsystems may need a new shared API layer to coordinate eARC negotiation without requiring userspace (e.g., libcec) to arbitrate between the two subsystems. Note: speculative.
- **Spatial audio metadata in the Linux stack**: Dolby Atmos and DTS:X carry object-based metadata alongside audio. A future Linux spatial-audio API (analogous to Apple's `kAudioFormatFlagIsBigEndian` object path or Windows Spatial Sound) would require new ALSA PCM extensions, PipeWire node properties, and DRM HDMI audio-component changes to propagate rendering metadata end-to-end. Note: speculative.
- **USB4/Thunderbolt DP audio integration**: As Thunderbolt 4 / USB4 docks increasingly use DP-over-USB4 tunnels, audio over those tunnels (via the DP AUX audio path) must be routed through the USB4 connection manager and the DRM DP audio helpers. Long-term unification of USB4 and native DP audio paths is expected. Note: speculative.

---

## Integrations

- **Ch2 (KMS)** — DRM connector holds the EDID and ELD; `drm_edid_to_eld` runs after connector detection
- **Ch3 (Advanced Display / HDR)** — HDMI 2.1 ARC/eARC for HDR displays; High Frame Rate audio considerations
- **Ch38 (PipeWire)** — PipeWire ALSA monitor discovers HDMI PCM devices; WirePlumber manages hotplug routing
- **Ch128 (DisplayPort MST)** — Each MST sink has its own ELD and ALSA PCM; `drm_dp_mst_port` per-port audio
- **Ch139 (Hardware Planes)** — Video overlay plane + audio timing: hardware planes change display mode, triggering audio ACR recalculation
