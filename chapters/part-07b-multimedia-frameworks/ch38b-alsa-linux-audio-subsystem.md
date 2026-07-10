# Chapter 38b: ALSA — The Linux Audio Subsystem

> **Part**: Part VII-B — Multimedia Frameworks and Desktop Integration
> **Audiences**: Systems developers implementing ALSA drivers (ASoC machine/codec drivers, HDA patch functions); application developers using `libasound` directly (embedded systems, low-latency audio tools, legacy apps not yet ported to PipeWire); audio engineers debugging Linux audio issues.
> **Status**: First draft — 2026-06-21

---

## Table of Contents

- [Overview](#overview)
- [1. Introduction and Architecture](#1-introduction-and-architecture)
- [2. ALSA Kernel Architecture](#2-alsa-kernel-architecture)
- [3. libasound PCM Programming](#3-libasound-pcm-programming)
- [4. Hardware Parameters in Depth](#4-hardware-parameters-in-depth)
- [5. ALSA Mixer API](#5-alsa-mixer-api)
- [6. ALSA PCM Plugins and asound.conf](#6-alsa-pcm-plugins-and-asoundconf)
- [7. UCM2: Use Case Manager](#7-ucm2-use-case-manager)
- [8. ASoC: Audio System on Chip Framework](#8-asoc-audio-system-on-chip-framework)
- [9. HDA: High Definition Audio](#9-hda-high-definition-audio)
- [10. ALSA Sequencer and MIDI](#10-alsa-sequencer-and-midi)
- [11. Debugging and Diagnostics](#11-debugging-and-diagnostics)
- [12. Integrations](#12-integrations)

---

## Overview

ALSA (Advanced Linux Sound Architecture) is the Linux kernel's primary audio subsystem, present in the mainline kernel since 2.6.6 (2004) where it replaced the older OSS (Open Sound System) interface. ALSA is not a single monolithic driver but a layered framework: the kernel provides the PCM engine, mixer controls, sequencer, and hardware abstraction; userspace applications interact through `libasound` (the ALSA library) or through character devices under `/dev/snd/`.

On modern desktop Linux systems, PipeWire owns the ALSA hardware devices and applications connect through PipeWire's compatibility layers. Direct `libasound` use remains prevalent for embedded and headless systems, for professional low-latency applications that pre-date PipeWire, and for legacy applications that access `hw:` devices directly. The `pipewire-alsa` PCM plugin makes this transparent: when installed, `snd_pcm_open("default", ...)` routes through PipeWire without application changes. See Ch38 for PipeWire internals.

---

## 1. Introduction and Architecture

### 1.1 The Layered View

ALSA organises the audio path into four tiers:

**Hardware**: Physical audio hardware — PCIe HDA controllers, SoC I2S/PCM buses, USB audio class devices, virtual drivers.

**Kernel drivers**: Located under `sound/` in the kernel tree. Key subtrees include `sound/core/` (PCM engine, control interface, timer, sequencer core), `sound/pci/` (PCI audio drivers: HDA, Envy24, EMU10k1), `sound/soc/` (the ASoC framework for SoC embedded audio), `sound/usb/` (USB audio and MIDI), and `sound/drivers/` (virtual and platform-independent devices).

**Character devices**: The kernel registers devices under `/dev/snd/` for each card and device:

| Node | Purpose |
|------|---------|
| `/dev/snd/controlC{N}` | Mixer controls for card N |
| `/dev/snd/pcmC{N}D{D}p` | PCM playback, card N device D |
| `/dev/snd/pcmC{N}D{D}c` | PCM capture, card N device D |
| `/dev/snd/seq` | MIDI sequencer (global) |
| `/dev/snd/timer` | Timer interface (global) |
| `/dev/snd/hwC{N}D{D}` | HDA hwdep for codec access |

**Userspace**: `libasound` (`libasound.so.2`, from the `alsa-lib` package) wraps all character device ioctls into a typed C API. Applications call `snd_pcm_open()`, `snd_pcm_hw_params()`, `snd_pcm_writei()` rather than `open(2)` + `ioctl(2)` directly. [Source](https://www.alsa-project.org/alsa-doc/alsa-lib/)

### 1.2 Post-2021 Deployment Reality

On most Linux desktops since Fedora 34 (2021), PipeWire holds exclusive access to `/dev/snd/pcmC*` hardware devices. Applications calling `snd_pcm_open("default", ...)` are transparently redirected through the `type pipewire` ALSA PCM plugin to a PipeWire stream. Direct `hw:` access still works for:

- Embedded or headless systems without a PipeWire session.
- Professional audio applications using the MMAP zero-copy path directly.
- Diagnostic tools (`aplay`, `speaker-test`, `hda-verb`) that need to bypass the session layer.
- Kernel driver development where testing against the raw PCM device is required.

---

## 2. ALSA Kernel Architecture

### 2.1 The Sound Card: `snd_card`

Every ALSA driver begins by allocating a sound card with `snd_card_new()` and registering it with `snd_card_register()`. The `snd_card` struct (defined in `include/sound/core.h`) is the root object: [Source](https://raw.githubusercontent.com/torvalds/linux/master/include/sound/core.h)

```c
struct snd_card {
    int     number;           /* card slot index (0–7 by default) */
    char    id[16];           /* sysfs/proc identifier */
    char    shortname[32];    /* brief name, e.g. "HDA Intel PCH" */
    char    longname[80];     /* full name with bus position */
    char    components[128];  /* space-delimited component string */
    char    driver[16];       /* driver name */
    struct  list_head devices;
    void   *private_data;
    void  (*private_free)(struct snd_card *card);
    /* ... internal fields ... */
};
```

`snd_card_new()` allocates the card struct plus `extra_size` bytes of driver-private space appended to it:

```c
int snd_card_new(struct device *parent, int idx, const char *xid,
                 struct module *module, int extra_size,
                 struct snd_card **card_ret);
```

`snd_card_register()` calls `device_add()` for the card device, populates the global `snd_cards[]` array, and makes the card visible to userspace. [Source](https://raw.githubusercontent.com/torvalds/linux/master/sound/core/init.c)

### 2.2 PCM Subsystem: `snd_pcm` and `snd_pcm_ops`

A PCM device is created with `snd_pcm_new()` after the card is initialised:

```c
int snd_pcm_new(struct snd_card *card, const char *id, int device,
                int playback_count, int capture_count,
                struct snd_pcm **rpcm);
```

`playback_count` and `capture_count` control how many substreams (independent hardware paths) each direction exposes. The function creates device nodes named `pcmC%iD%i%c` where the suffix is `'p'` for playback and `'c'` for capture. [Source](https://raw.githubusercontent.com/torvalds/linux/master/sound/core/pcm.c)

Each direction's operations are registered via:

```c
void snd_pcm_set_ops(struct snd_pcm *pcm, int direction,
                     const struct snd_pcm_ops *ops);
```

The `snd_pcm_ops` vtable (`include/sound/pcm.h`) defines the driver interface: [Source](https://raw.githubusercontent.com/torvalds/linux/master/include/sound/pcm.h)

```c
struct snd_pcm_ops {
    int  (*open)(struct snd_pcm_substream *substream);
    int  (*close)(struct snd_pcm_substream *substream);
    int  (*ioctl)(struct snd_pcm_substream *substream,
                  unsigned int cmd, void *arg);
    int  (*hw_params)(struct snd_pcm_substream *substream,
                      struct snd_pcm_hw_params *params);
    int  (*hw_free)(struct snd_pcm_substream *substream);
    int  (*prepare)(struct snd_pcm_substream *substream);
    int  (*trigger)(struct snd_pcm_substream *substream, int cmd);
    snd_pcm_uframes_t (*pointer)(struct snd_pcm_substream *substream);
    /* get_time_info, fill_silence, copy, mmap, ack ... */
};
```

The `trigger` callback receives one of these commands, defined in `include/sound/pcm.h` (not the UAPI header):

```c
#define SNDRV_PCM_TRIGGER_STOP          0
#define SNDRV_PCM_TRIGGER_START         1
#define SNDRV_PCM_TRIGGER_PAUSE_PUSH    2
#define SNDRV_PCM_TRIGGER_PAUSE_RELEASE 3
#define SNDRV_PCM_TRIGGER_SUSPEND       4
#define SNDRV_PCM_TRIGGER_RESUME        5
#define SNDRV_PCM_TRIGGER_DRAIN         6
```

The trigger callback must be minimal and non-sleeping — it is called from atomic context.

The `pointer` callback returns the current hardware DMA position in frames. The ALSA core calls it to update the ring buffer write pointer and to detect xruns.

### 2.3 The Ring Buffer: `snd_pcm_runtime`

When a PCM substream is opened, the ALSA core allocates `snd_pcm_runtime` and attaches it to `substream->runtime`. The key fields for DMA and pointer management are:

```c
struct snd_pcm_runtime {
    /* DMA buffer allocation */
    unsigned char      *dma_area;   /* kernel virtual address */
    dma_addr_t          dma_addr;   /* bus/physical address */
    size_t              dma_bytes;  /* total allocation size */

    /* Ring buffer geometry (set by hw_params) */
    snd_pcm_uframes_t   buffer_size;   /* total ring size in frames */
    snd_pcm_uframes_t   period_size;   /* interrupt granularity in frames */
    unsigned int        periods;       /* = buffer_size / period_size */
    snd_pcm_uframes_t   boundary;      /* wrap boundary for pointer arithmetic */

    /* Format (set by hw_params) */
    snd_pcm_format_t    format;
    unsigned int        rate;
    unsigned int        channels;

    /* Mmap-shared pages (accessible from userspace without syscall) */
    struct snd_pcm_mmap_status  *status;   /* contains hw_ptr */
    struct snd_pcm_mmap_control *control;  /* contains appl_ptr */
    /* ... */
};
```

**An important architectural detail**: `hw_ptr` and `appl_ptr` are not direct fields of `snd_pcm_runtime`. They live in mmap-shared pages:

- `runtime->status->hw_ptr` — in `struct __snd_pcm_mmap_status` (`include/uapi/sound/asound.h`). The kernel advances this when DMA completes.
- `runtime->control->appl_ptr` — in `struct __snd_pcm_mmap_control` (`include/uapi/sound/asound.h`). The ALSA library updates this as the application writes data.

Both pages are mapped into userspace via `mmap(2)` on the PCM file descriptor, so `snd_pcm_avail_update()` can read `hw_ptr` without a syscall. [Source](https://raw.githubusercontent.com/torvalds/linux/master/include/uapi/sound/asound.h)

### 2.4 The Period Interrupt

The DMA period is the fundamental timing unit. The hardware fires an interrupt after each `period_size` frames of audio are transferred. The driver's DMA completion ISR calls:

```c
void snd_pcm_period_elapsed(struct snd_pcm_substream *substream);
```

This function (in `sound/core/pcm_lib.c`) acquires the stream lock, advances `runtime->status->hw_ptr`, checks for xrun conditions, and wakes any process blocked in `snd_pcm_writei()` or `snd_pcm_readi()`. The caller must **not** hold the stream lock before calling `snd_pcm_period_elapsed()` — the function acquires it internally. [Source](https://raw.githubusercontent.com/torvalds/linux/master/sound/core/pcm_lib.c)

### 2.5 Mixer Controls: `snd_kcontrol`

The ALSA control interface exposes mixer elements through `snd_kcontrol` objects. Drivers create them from a template:

```c
struct snd_kcontrol_new {
    snd_ctl_elem_iface_t  iface;    /* SNDRV_CTL_ELEM_IFACE_MIXER (2) for mixers */
    const char           *name;     /* e.g. "Master Playback Volume" */
    unsigned int          index;
    snd_kcontrol_info_t  *info;     /* reports type, count, range */
    snd_kcontrol_get_t   *get;      /* read current value */
    snd_kcontrol_put_t   *put;      /* write new value, return 1 if changed */
    unsigned long         private_value;  /* driver-defined context */
};
```

Control element types (`include/uapi/sound/asound.h`):

| Constant | Value | Usage |
|----------|-------|-------|
| `SNDRV_CTL_ELEM_TYPE_BOOLEAN` | 1 | On/off switch |
| `SNDRV_CTL_ELEM_TYPE_INTEGER` | 2 | Volume in hardware units |
| `SNDRV_CTL_ELEM_TYPE_ENUMERATED` | 3 | Input source selection |
| `SNDRV_CTL_ELEM_TYPE_BYTES` | 4 | Raw byte arrays (ELD, firmware) |
| `SNDRV_CTL_ELEM_TYPE_IEC958` | 5 | S/PDIF status |

Registration:

```c
struct snd_kcontrol *kctl = snd_ctl_new1(&template, private_data);
int ret = snd_ctl_add(card, kctl);
```

`snd_ctl_add()` assigns a unique `numid`, links the control into `card->controls`, and sends an `SNDRV_CTL_EVENT_MASK_ADD` notification so that open control file descriptors see the new element. [Source](https://raw.githubusercontent.com/torvalds/linux/master/include/sound/control.h)

---

## 3. libasound PCM Programming

This section covers the complete `libasound` PCM workflow: open, hardware parameter negotiation, software parameter setup, write/recover loop, and clean shutdown.

### 3.1 Device Names

The `name` argument to `snd_pcm_open()` is resolved through `/usr/share/alsa/alsa.conf` and `~/.asoundrc`:

| Name | Description |
|------|-------------|
| `"default"` | Resolves to PipeWire (if installed) or card 0 with conversion plugins |
| `"hw:0,0"` | Direct kernel device: card 0, device 0, no conversion |
| `"plughw:0,0"` | Auto-format/rate/channel conversion wrapping `hw:0,0` |
| `"pipewire"` | Explicit PipeWire target via the ALSA PCM plugin |
| `"null"` | Discards all data |

Direct `hw:` access fails if the requested format or rate is not natively supported by the hardware. `plughw:` inserts software conversion; `default` on modern desktops routes through PipeWire (with its own high-quality resampler). [Source](https://www.alsa-project.org/alsa-doc/alsa-lib/group___p_c_m.html)

### 3.2 Complete Example: Sine Wave Playback

```c
/* compile: gcc -o sine_wave sine_wave.c -lasound -lm */
#include <alsa/asoundlib.h>
#include <math.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>

#define SAMPLE_RATE   48000
#define CHANNELS      2
#define PERIOD_FRAMES 512
#define BUFFER_FRAMES (PERIOD_FRAMES * 4)
#define FREQ_HZ       440.0

static int xrun_recovery(snd_pcm_t *pcm, int err) {
    if (err == -EPIPE) {
        /* Underrun: reinitialise the stream */
        err = snd_pcm_prepare(pcm);
        if (err < 0)
            fprintf(stderr, "prepare after underrun failed: %s\n",
                    snd_strerror(err));
    } else if (err == -ESTRPIPE) {
        /* Suspended: wait for resume */
        while ((err = snd_pcm_resume(pcm)) == -EAGAIN)
            sleep(1);
        if (err < 0)
            err = snd_pcm_prepare(pcm);
    }
    return err;
}

int main(void) {
    snd_pcm_t            *pcm;
    snd_pcm_hw_params_t  *hw;
    snd_pcm_sw_params_t  *sw;
    int                   err;

    /* --- Open --- */
    err = snd_pcm_open(&pcm, "default", SND_PCM_STREAM_PLAYBACK, 0);
    if (err < 0) { fprintf(stderr, "open: %s\n", snd_strerror(err)); return 1; }

    /* --- Hardware parameters --- */
    snd_pcm_hw_params_alloca(&hw);
    snd_pcm_hw_params_any(pcm, hw);

    snd_pcm_hw_params_set_access(pcm, hw, SND_PCM_ACCESS_RW_INTERLEAVED);
    snd_pcm_hw_params_set_format(pcm, hw, SND_PCM_FORMAT_S16_LE);
    snd_pcm_hw_params_set_channels(pcm, hw, CHANNELS);

    unsigned int rate = SAMPLE_RATE;
    snd_pcm_hw_params_set_rate_near(pcm, hw, &rate, NULL);

    snd_pcm_uframes_t period = PERIOD_FRAMES;
    snd_pcm_hw_params_set_period_size_near(pcm, hw, &period, NULL);

    snd_pcm_uframes_t bufsize = BUFFER_FRAMES;
    snd_pcm_hw_params_set_buffer_size_near(pcm, hw, &bufsize);

    err = snd_pcm_hw_params(pcm, hw);
    if (err < 0) { fprintf(stderr, "hw_params: %s\n", snd_strerror(err)); return 1; }

    /* Read back negotiated values */
    snd_pcm_hw_params_get_period_size(hw, &period, NULL);
    snd_pcm_hw_params_get_buffer_size(hw, &bufsize);
    fprintf(stderr, "period=%lu frames  buffer=%lu frames  rate=%u Hz\n",
            (unsigned long)period, (unsigned long)bufsize, rate);
    fprintf(stderr, "period latency = %.2f ms\n",
            1000.0 * period / rate);

    /* --- Software parameters --- */
    snd_pcm_sw_params_alloca(&sw);
    snd_pcm_sw_params_current(pcm, sw);
    /* Start playback automatically when ring buffer has one period of data */
    snd_pcm_sw_params_set_start_threshold(pcm, sw, period);
    /* Wake the application when at least one period of space is available */
    snd_pcm_sw_params_set_avail_min(pcm, sw, period);
    snd_pcm_sw_params(pcm, sw);

    /* --- Generate and write audio --- */
    int16_t *buf = malloc(period * CHANNELS * sizeof(int16_t));
    double   phase = 0.0;
    double   phase_inc = 2.0 * M_PI * FREQ_HZ / rate;
    int      running = 1;
    long     total_frames = rate * 3;  /* play 3 seconds */

    while (running && total_frames > 0) {
        snd_pcm_uframes_t n = period;
        if ((long)n > total_frames) n = total_frames;

        /* Fill buffer with interleaved stereo sine */
        for (snd_pcm_uframes_t i = 0; i < n; i++) {
            int16_t sample = (int16_t)(0.5 * 32767.0 * sin(phase));
            buf[i * CHANNELS]     = sample;   /* left */
            buf[i * CHANNELS + 1] = sample;   /* right */
            phase += phase_inc;
        }

        snd_pcm_sframes_t written = snd_pcm_writei(pcm, buf, n);
        if (written < 0) {
            err = xrun_recovery(pcm, (int)written);
            if (err < 0) { fprintf(stderr, "writei failed: %s\n", snd_strerror(err)); break; }
        } else {
            total_frames -= written;
        }
    }

    /* --- Drain, close, clean up --- */
    snd_pcm_drain(pcm);
    snd_pcm_close(pcm);
    free(buf);
    return 0;
}
```

### 3.3 Non-blocking Poll Loop

For event-driven applications, open with `SND_PCM_NONBLOCK` and use `poll(2)`:

```c
snd_pcm_t *pcm;
snd_pcm_open(&pcm, "default", SND_PCM_STREAM_PLAYBACK, SND_PCM_NONBLOCK);
/* ... hw/sw params setup ... */

int nfds = snd_pcm_poll_descriptors_count(pcm);
struct pollfd *pfds = malloc(nfds * sizeof(*pfds));
snd_pcm_poll_descriptors(pcm, pfds, nfds);

for (;;) {
    poll(pfds, nfds, -1);
    unsigned short revents;
    snd_pcm_poll_descriptors_revents(pcm, pfds, nfds, &revents);
    if (revents & POLLERR) {
        /* handle xrun via snd_pcm_recover() */
    }
    if (revents & POLLOUT) {
        /* space available: write a period of audio */
    }
}
```

Do not read `pfds[i].revents` directly — `snd_pcm_poll_descriptors_revents()` performs the correct translation from ALSA's internal encoding to standard `POLLIN`/`POLLOUT` flags. [Source](https://www.alsa-project.org/alsa-doc/alsa-lib/group___p_c_m.html)

---

## 4. Hardware Parameters in Depth

### 4.1 Period vs. Buffer

The ring buffer is divided into `periods` equal segments. On each DMA completion interrupt, the hardware advances by exactly `period_size` frames. The application must write new audio at least one period ahead of the hardware pointer to avoid an underrun (`EPIPE`). The headroom available is `buffer_size - period_size` frames; with `periods = 4` this gives three periods of headroom while the hardware plays the first.

**Latency relationships**:

```
period_latency  = period_size / sample_rate   [seconds]
buffer_latency  = buffer_size / sample_rate   [seconds]
minimum_latency = 2 × period_size / sample_rate
```

Example: `period_size=512`, `rate=48000` → `period_latency = 10.67 ms`, `buffer_latency = 42.67 ms` with four periods.

`snd_pcm_delay()` returns the instantaneous pipeline latency in frames (number of frames in the ring buffer not yet played), which accounts for hardware FIFO depth beyond the ring buffer itself. [Source](https://www.alsa-project.org/alsa-doc/alsa-lib/group___p_c_m.html)

### 4.2 `snd_pcm_avail_update()`

Before writing to the ring buffer, call `snd_pcm_avail_update()`:

```c
snd_pcm_sframes_t avail = snd_pcm_avail_update(pcm);
```

This function reads `hw_ptr` from the mmap-shared status page (no syscall on most architectures), computes available frames, and returns the count. The library's cached view of `hw_ptr` can be stale after a period interrupt; `snd_pcm_avail_update()` synchronises it. Always call before deciding how many frames to write.

### 4.3 Format Handling

Key format constants (`SND_PCM_FORMAT_*` from `include/sound/pcm.h`):

| Constant | Bits | Bytes | Notes |
|----------|------|-------|-------|
| `SND_PCM_FORMAT_S16_LE` | 16 | 2 | Most widely supported |
| `SND_PCM_FORMAT_S24_3LE` | 24 | 3 | Packed 24-bit, no padding |
| `SND_PCM_FORMAT_S24_LE` | 24 | 4 | 24-bit in 32-bit container |
| `SND_PCM_FORMAT_S32_LE` | 32 | 4 | Used for DSP/mixing |
| `SND_PCM_FORMAT_FLOAT_LE` | 32 | 4 | IEEE 754 float, ±1.0 range |

Non-interleaved access uses `snd_pcm_writen()` with separate per-channel arrays and `SND_PCM_ACCESS_RW_NONINTERLEAVED`.

### 4.4 MMAP Zero-Copy Access

For the lowest latency, use MMAP access to write directly into the DMA buffer:

```c
/* Open with MMAP access */
snd_pcm_hw_params_set_access(pcm, hw, SND_PCM_ACCESS_MMAP_INTERLEAVED);

/* In the write loop: */
snd_pcm_avail_update(pcm);

const snd_pcm_channel_area_t *areas;
snd_pcm_uframes_t offset, frames = period_size;

snd_pcm_mmap_begin(pcm, &areas, &offset, &frames);

/* areas[ch].addr is the DMA buffer base; write sample at frame f, channel ch: */
/* ptr = (uint8_t*)areas[ch].addr + areas[ch].first/8 + offset * areas[ch].step/8 */
int16_t *dst = (int16_t *)((uint8_t *)areas[0].addr
               + (areas[0].first / 8)
               + offset * (areas[0].step / 8));
fill_audio(dst, frames);

snd_pcm_mmap_commit(pcm, offset, frames);
```

`snd_pcm_channel_area_t` describes the memory layout per channel: `addr` (buffer base), `first` (bit offset to first sample), `step` (bit distance between consecutive samples). For interleaved stereo S16_LE: `step = 32` bits (two channels × 16 bits), `first = 0` for left and `16` for right. [Source](https://www.alsa-project.org/alsa-doc/alsa-lib/group___p_c_m___direct.html)

PipeWire's `spa-alsa` plugin uses this MMAP path for zero-copy DMA from the kernel ring buffer into the PipeWire graph. See Ch38 for graph scheduling details.

---

## 5. ALSA Mixer API

### 5.1 Opening and Traversing

The simple element API (`snd_mixer_selem_*`) abstracts hardware controls into named elements:

```c
snd_mixer_t    *mixer;
snd_mixer_selem_id_t *sid;
snd_mixer_elem_t    *elem;

/* Open and attach to the card */
snd_mixer_open(&mixer, 0);
snd_mixer_attach(mixer, "hw:0");       /* attach to card 0 */
snd_mixer_selem_register(mixer, NULL, NULL);
snd_mixer_load(mixer);

/* Find the "Master" element */
snd_mixer_selem_id_alloca(&sid);
snd_mixer_selem_id_set_index(sid, 0);
snd_mixer_selem_id_set_name(sid, "Master");
elem = snd_mixer_find_selem(mixer, sid);
```

### 5.2 Volume and Mute

```c
/* Read volume range */
long vol_min, vol_max;
snd_mixer_selem_get_playback_volume_range(elem, &vol_min, &vol_max);

/* Get current volume (left channel = SND_MIXER_SCHN_FRONT_LEFT) */
long vol;
snd_mixer_selem_get_playback_volume(elem, SND_MIXER_SCHN_FRONT_LEFT, &vol);

/* Set volume on all channels */
long new_vol = (vol_min + vol_max) / 2;   /* 50% */
snd_mixer_selem_set_playback_volume_all(elem, new_vol);

/* Get/set mute switch (0 = muted, 1 = unmuted) */
int sw;
snd_mixer_selem_get_playback_switch(elem, SND_MIXER_SCHN_FRONT_LEFT, &sw);
snd_mixer_selem_set_playback_switch_all(elem, 0);  /* mute */
```

### 5.3 dB Volumes and Enumerations

Hardware volume ranges are in raw hardware units; convert to dB with:

```c
long db_min, db_max;
snd_mixer_selem_get_playback_dB_range(elem, &db_min, &db_max);
/* values are in 0.01 dB units: -4000 = -40.00 dB */
```

For input source selection (HDMI output choice, microphone input type), elements are enumerated:

```c
if (snd_mixer_selem_is_enumerated(elem)) {
    unsigned int items = snd_mixer_selem_get_enum_items(elem);
    for (unsigned int i = 0; i < items; i++) {
        char name[64];
        snd_mixer_selem_get_enum_item_name(elem, i, sizeof(name), name);
        printf("  [%u] %s\n", i, name);
    }
}
```

[Source](https://www.alsa-project.org/alsa-doc/alsa-lib/group___mixer.html)

CLI equivalents: `amixer get Master` reads current state; `amixer set Master 70%` sets volume; `alsamixer` provides an ncurses TUI with per-element fine control.

---

## 6. ALSA PCM Plugins and asound.conf

### 6.1 Plugin Architecture

When an application opens `"default"`, `"plughw:0"`, or any named PCM that is not a raw `hw:` device, `libasound` constructs a *plugin chain*: each plugin wraps the next, transforming the audio stream. The hardware PCM device sits at the bottom; conversions are applied from top to bottom. Common plugins:

- **`plug`**: inserts SRC (sample rate conversion), format conversion, and channel remapping as needed. `plughw:0,0` is shorthand for a `plug` plugin over `hw:0,0`.
- **`dmix`**: software mixing. Opens `hw:0,0` exclusively as an internal stream, then mixes multiple application streams in shared memory using a 32-bit accumulation buffer. The first opening application becomes the "server"; subsequent applications attach to the shared memory segment.
- **`dsnoop`**: multi-app capture sharing — the inverse of `dmix`.
- **`pipewire`**: routes `libasound` API calls to a PipeWire stream. Provided by the `pipewire-alsa` package.

Each plugin in the chain adds buffering equal to at least one period, so plugin chains increase latency. A direct `hw:` device has only kernel→DMA latency; `default` through PipeWire typically adds 5–50 ms depending on the PipeWire quantum setting.

### 6.2 PipeWire ALSA Plugin

The `pipewire-alsa` package installs two components:

1. ALSA config files (typically dropped into `/usr/share/alsa/alsa.conf.d/` and symlinked from `/etc/alsa/conf.d/`):

```ini
# /etc/alsa/conf.d/99-pipewire-default.conf
pcm.!default {
    type pipewire
    playback_node "-1"
    capture_node  "-1"
    hint {
        show on
        description "Default ALSA Output (currently PipeWire Media Server)"
    }
}

ctl.!default {
    type pipewire
}
```

2. The ALSA external plugin library `libasound_module_pcm_pipewire.so`, implementing `snd_pcm_ioplug_t`. When `snd_pcm_open("default", ...)` is called, libasound loads this library, which creates a `pw_stream` connected to the PipeWire daemon. The application sees standard ALSA hw_params negotiation; internally the plugin is exercising the PipeWire wire protocol. [Source](https://github.com/PipeWire/pipewire/blob/master/pipewire-alsa/conf/99-pipewire-default.conf)

### 6.3 Manual asound.conf with Fallback

For embedded systems without PipeWire, a manual `~/.asoundrc` mixing setup:

```ini
# Software mixing via dmix, rate conversion via plug
pcm.!default {
    type plug
    slave.pcm {
        type dmix
        ipc_key 1234
        ipc_key_add_uid true
        slave {
            pcm "hw:0,0"
            format S16_LE
            rate 48000
            channels 2
            period_time 125000   # 125 ms in microseconds
            buffer_time 500000   # 500 ms total
        }
    }
}
```

---

## 7. UCM2: Use Case Manager

### 7.1 Purpose and Config Location

UCM2 (Use Case Manager version 2) provides a declarative, board-level routing configuration that abstracts codec internals from applications. Instead of an application issuing raw `amixer set` commands to enable a speaker path, it calls `snd_use_case_set(ucm, "_verb", "HiFi")` and UCM2 applies the correct sequence of ALSA control writes.

Config files live under `/usr/share/alsa/ucm2/`. The modern Syntax 4+ layout uses driver-based subdirectories:

```
/usr/share/alsa/ucm2/conf.d/{CardDriver}/{CardLongName}.conf
/usr/share/alsa/ucm2/conf.virt.d/{VirtualName}.conf    # virtual cards
```

The standalone `alsa-ucm-conf` package (separate from `alsa-lib` since version 1.2.1) provides configs for thousands of boards. [Source](https://github.com/alsa-project/alsa-ucm-conf)

### 7.2 File Structure

A UCM config file declares one or more *verbs* (top-level use cases), each of which contains devices and optional modifiers:

```ini
# CardName.conf (abbreviated example)

SectionUseCase."HiFi" {
    File "/MyBoard/HiFi.conf"
    Comment "HiFi audio playback and capture"
}
```

In `HiFi.conf`:

```ini
SectionVerb {
    EnableSequence [
        cset "name='Headphone Switch' 0"
        cset "name='Speaker Switch' 0"
        disdevall ""
    ]
    DisableSequence [
        cset "name='Power Save' on"
    ]
    Value {
        TQ HiFi
    }
}

SectionDevice."Speaker" {
    EnableSequence [
        cset "name='Speaker Switch' 1"
    ]
    DisableSequence [
        cset "name='Speaker Switch' 0"
    ]
    Value {
        PlaybackPriority 100
        PlaybackPCM "hw:${CardId},0"
        PlaybackVolume "name='Speaker Playback Volume'"
    }
}

SectionDevice."Headphones" {
    ConflictingDevice [ "Speaker" ]
    EnableSequence [
        cset "name='Headphone Switch' 1"
        cset "name='Speaker Switch' 0"
    ]
    DisableSequence [
        cset "name='Headphone Switch' 0"
    ]
    Value {
        PlaybackPriority 300
        PlaybackPCM "hw:${CardId},0"
        JackControl "name='Headphone Jack'"
        JackHWMute "Speaker"
    }
}

SectionDevice."Mic" {
    ConflictingDevice [ "Headset" ]
    EnableSequence [
        cset "name='Internal Mic Switch' 1"
    ]
    DisableSequence [
        cset "name='Internal Mic Switch' 0"
    ]
    Value {
        CapturePriority 100
        CapturePCM "hw:${CardId},0"
    }
}

SectionModifier."Capture Voice" {
    SupportedDevice [ "Mic" "Headset" ]
    EnableSequence [
        cset "name='VX_REC Path' on"
    ]
    DisableSequence [
        cset "name='VX_REC Path' off"
    ]
    Value {
        TQ Voice
        CapturePCM "hw:${CardId},1"
    }
}
```

[Source](https://github.com/alsa-project/alsa-ucm-conf/blob/master/ucm2/Intel/sof-essx8336/HiFi.conf)

### 7.3 cset Command

The `cset` command is the UCM workhorse — it maps directly to an `amixer cset` call:

```
cset "name='<control-name>'[,index=<N>] <value>[,<value>...]"
```

Additional sequence commands include `msleep <N>` (delay in milliseconds), `disdevall ""` (disable all active devices), `exec "/path args"` (run an external program), and `sysw "/sys/class/...=value"` (write a sysfs node). [Source](https://www.alsa-project.org/alsa-doc/alsa-lib/group__ucm__conf.html)

### 7.4 alsaucm CLI

```bash
# List cards with UCM configs
alsaucm listcards

# Activate a verb and enable a device (batch mode required for shared context)
alsaucm -c "sof-essx8336" -b - <<'EOF'
reset
set _verb HiFi
set _enadev Speaker
EOF

# Query available verbs and devices
alsaucm -c "sof-essx8336" -b - <<'EOF'
set _verb HiFi
list _devices
EOF
```

`list _devices` only works after `set _verb` within the same execution context — separate `alsaucm` invocations do not share state. [Source](https://man.archlinux.org/man/extra/alsa-utils/alsaucm.1.en)

### 7.5 Programmatic Access

```c
snd_use_case_mgr_t *ucm;
snd_use_case_mgr_open(&ucm, "sof-essx8336");
snd_use_case_set(ucm, "_verb", "HiFi");
snd_use_case_set(ucm, "_enadev", "Speaker");

/* Query current verb */
const char *verb;
snd_use_case_get(ucm, "_verb", &verb);
printf("Active verb: %s\n", verb);
free((void *)verb);

snd_use_case_mgr_close(ucm);
```

[Source](https://www.alsa-project.org/alsa-doc/alsa-lib/group__ucm.html)

Note: UCM3 does not exist as an official ALSA specification. The ALSA project continues evolving UCM2 through syntax versioning (currently Syntax 9, which adds `Repeat` blocks and integer conditions in `If`/`Condition` clauses). The latest config package is `alsa-ucm-conf 1.2.16.1` (June 2026).

---

## 8. ASoC: Audio System on Chip Framework

### 8.1 Motivation: The Three-Driver Model

Legacy `snd_pcm_ops` drivers bundle SoC DMA, I2C/SPI codec communication, and board-level routing into a single file. For embedded SoCs where the I2S controller (platform), codec chip (e.g., WM8994 on a Samsung board), and board routing (jack detection, amplifier GPIOs) are all separate components, this creates unresolvable coupling.

ASoC decomposes each embedded audio design into three independently reusable drivers: [Source](https://www.kernel.org/doc/html/latest/sound/soc/overview.html)

- **Machine driver**: Board-specific glue. Describes `snd_soc_dai_link` connections between platform DAIs and codec DAIs; handles clock routing, amplifier GPIOs, and jack detection. Creates the ALSA sound card via `devm_snd_soc_register_card()`.
- **Codec driver**: Codec-chip-specific. Contains `snd_soc_component_driver` with ALSA controls, DAPM widgets, DAPM routes, and register I/O (typically via regmap). No board-specific code.
- **Platform driver**: SoC DMA engine. Provides `snd_pcm_ops` for DMA setup, an `snd_soc_component_driver` for buffer allocation, and SoC DAI drivers for I2S/PCM clocking.

### 8.2 `snd_soc_card` and `snd_soc_dai_link`

The machine driver declares an `snd_soc_card` struct:

```c
static struct snd_soc_card my_board_card = {
    .name         = "my-board",
    .owner        = THIS_MODULE,
    .dai_link     = my_board_dai_links,
    .num_links    = ARRAY_SIZE(my_board_dai_links),
    .dapm_widgets = my_board_widgets,
    .num_dapm_widgets = ARRAY_SIZE(my_board_widgets),
    .dapm_routes  = my_board_routes,
    .num_dapm_routes = ARRAY_SIZE(my_board_routes),
};
```

In the current kernel, `snd_soc_dai_link` no longer uses the legacy scalar string fields `cpu_dai_name`, `codec_dai_name`, `codec_name`, and `platform_name`. These have been replaced by arrays of `snd_soc_dai_link_component`. The `SND_SOC_DAILINK_DEFS` and `SND_SOC_DAILINK_REG` macros populate these arrays at compile time: [Source](https://raw.githubusercontent.com/torvalds/linux/master/include/sound/soc.h)

```c
/* Declare component arrays */
SND_SOC_DAILINK_DEFS(hifi_link,
    DAILINK_COMP_ARRAY(COMP_CPU("samsung-i2s.0")),
    DAILINK_COMP_ARRAY(COMP_CODEC("wm8994-codec", "wm8994-aif1")),
    DAILINK_COMP_ARRAY(COMP_PLATFORM("samsung-i2s.0")));

static struct snd_soc_dai_link my_board_dai_links[] = {
    {
        .name        = "WM8994 AIF1",
        .stream_name = "Primary",
        .dai_fmt     = SND_SOC_DAIFMT_I2S
                     | SND_SOC_DAIFMT_NB_NF
                     | SND_SOC_DAIFMT_CBP_CFP,
        .init        = my_board_init,
        .ops         = &my_board_ops,
        SND_SOC_DAILINK_REG(hifi_link),   /* fills .cpus, .codecs, .platforms */
    },
};
```

Registration:

```c
static int my_board_probe(struct platform_device *pdev) {
    my_board_card.dev = &pdev->dev;
    return devm_snd_soc_register_card(&pdev->dev, &my_board_card);
}
```

`devm_snd_soc_register_card()` probes all component drivers referenced by `dai_link[]` and instantiates the ALSA `snd_card`. [Source](https://raw.githubusercontent.com/torvalds/linux/master/include/sound/soc.h)

### 8.3 Codec Driver and DAPM

A codec driver registers a `snd_soc_component_driver` alongside one or more `snd_soc_dai_driver` instances:

```c
static const struct snd_soc_component_driver wm8731_component = {
    .name             = "wm8731",
    .controls         = wm8731_snd_controls,
    .num_controls     = ARRAY_SIZE(wm8731_snd_controls),
    .dapm_widgets     = wm8731_dapm_widgets,
    .num_dapm_widgets = ARRAY_SIZE(wm8731_dapm_widgets),
    .dapm_routes      = wm8731_intercon,
    .num_dapm_routes  = ARRAY_SIZE(wm8731_intercon),
};
```

DAPM widgets describe the audio signal graph inside the codec. The macros (`SND_SOC_DAPM_DAC`, `SND_SOC_DAPM_MIXER`, etc.) populate `snd_soc_dapm_widget` structs. Routes describe signal paths — the struct field order is `{ sink, control, source }` (note: sink is declared first even though data flows source → sink): [Source](https://raw.githubusercontent.com/torvalds/linux/master/include/sound/soc-dapm.h)

```c
static const struct snd_soc_dapm_route wm8731_intercon[] = {
    /* { sink,           control,                source } */
    { "DAC",             NULL,                   "ACTIVE" },
    { "Output Mixer",    "Line Bypass Switch",   "Line Input" },
    { "Output Mixer",    "HiFi Playback Switch", "DAC" },
    { "LHPOUT",          NULL,                   "Output Mixer" },
    { "LOUT",            NULL,                   "Output Mixer" },
    { "ADC",             NULL,                   "Input Mux" },
    { "Input Mux",       "Line In",              "Line Input" },
    { "Input Mux",       "Mic",                  "Mic Bias" },
    { "Mic Bias",        NULL,                   "MICIN" },
};
```

Routes embedded in `snd_soc_component_driver` are applied automatically during component probe. Additional routes (board-level signal paths) can be added with `snd_soc_dapm_add_routes()` from the machine driver's init callback.

After changing routes or widget states via ALSA controls, the DAPM engine must re-evaluate the graph with `snd_soc_dapm_sync()`, which traverses the dirty-widget list and powers widgets on or off based on whether they lie on an active signal path from an input endpoint to an output endpoint. [Source](https://raw.githubusercontent.com/torvalds/linux/master/sound/soc/soc-dapm.c)

### 8.4 DAPM Power Management

DAPM maintains four bias levels (`enum snd_soc_bias_level` in `include/sound/soc-dapm.h`):

| Level | Description |
|-------|-------------|
| `SND_SOC_BIAS_OFF` (0) | No power — cold boot or hot removal |
| `SND_SOC_BIAS_STANDBY` (1) | VREF/VMID powered, codec ready |
| `SND_SOC_BIAS_PREPARE` (2) | Transitional — clocks ramping |
| `SND_SOC_BIAS_ON` (3) | Full operational power, stream active |

Power-down transitions execute before power-up transitions to minimise audio artefacts. Widgets that are not on any active path between a PCM source/sink and a physical pin (INPUT/OUTPUT/HP/SPK/MIC) are powered off, saving power on mobile SoCs. [Source](https://www.kernel.org/doc/html/latest/sound/soc/dapm.html)

---

## 9. HDA: High Definition Audio

### 9.1 Serial Link and Controller Architecture

Intel's HD Audio ("Azalia") specification defines a synchronous 24 MHz serial link between a controller and up to 15 codec slots. The controller communicates with codec widgets (audio converters, mixers, pin complexes, power widgets) by sending 32-bit *verbs* and receiving 64-bit *responses*. [Source](https://www.kernel.org/doc/html/latest/sound/hd-audio/notes.html)

**CORB (Command Outbound Ring Buffer)**: Controller DMA-writes verb commands; codecs read them. The ring buffer is 256 entries of 32-bit words, allocated in a kernel DMA page.

**RIRB (Response Inbound Ring Buffer)**: Codecs DMA-write 64-bit responses (32-bit data + 32-bit extension with codec address and unsolicited-event flag). The controller reads responses on interrupt or by polling.

**BDL (Buffer Descriptor List)**: For each stream, an array of up to 256 16-byte entries, each pointing to a chunk of the DMA ring buffer. Each entry has lower and upper 32-bit physical address fields, a byte count, and an IOC bit to trigger a completion interrupt. [Source](https://raw.githubusercontent.com/torvalds/linux/master/sound/hda/hdac_controller.c)

CORB/RIRB initialisation (`snd_hdac_bus_init_cmd_io()` in `sound/hda/hdac_controller.c`) writes the DMA base addresses to CORBLBASE/CORBUBASE and RIRBLBASE/RIRBUBASE, sets both sizes to 256 entries (register value `0x02`), resets write and read pointers, and enables DMA and interrupt flags in CORBCTL/RIRBCTL.

### 9.2 Controller Driver: `hda_intel.c`

`sound/pci/hda/hda_intel.c` is the PCI driver for Intel HDA controllers (PCI class `0x0403`). It handles PCI probe, MSI setup, Intel-specific power management quirks, and calls into the `sound/hda/` library layer. The CORB/RIRB communication and stream DMA programming are implemented in:

- `sound/hda/hdac_controller.c` — `snd_hdac_bus_send_cmd()`, `snd_hdac_bus_update_rirb()`, `snd_hdac_bus_reset_link()`
- `sound/hda/hdac_stream.c` — `snd_hdac_stream_setup()`, `snd_hdac_stream_setup_periods()`, `setup_bdle()`

Codec slots are enumerated by reading the `STATESTS` register after releasing the controller reset line — each set bit indicates a codec at that slot address. [Source](https://raw.githubusercontent.com/torvalds/linux/master/sound/hda/hdac_controller.c)

### 9.3 HDA Verbs

A 32-bit CORB command encodes: codec address (bits 31:28), NID — node ID within the codec (bits 26:20), verb code (bits 19:8), and payload (bits 7:0). Key verbs from `include/sound/hda_verbs.h`: [Source](https://raw.githubusercontent.com/torvalds/linux/master/include/sound/hda_verbs.h)

| Macro | Hex | Purpose |
|-------|-----|---------|
| `AC_VERB_GET_STREAM_FORMAT` | `0x0A00` | Read stream format from converter |
| `AC_VERB_SET_STREAM_FORMAT` | `0x0200` | Write stream format (rate/depth/channels) |
| `AC_VERB_SET_CHANNEL_STREAMID` | `0x0706` | Bind converter to DMA stream tag |
| `AC_VERB_GET_PIN_SENSE` | `0x0F09` | Read jack presence / impedance |
| `AC_VERB_SET_PIN_WIDGET_CONTROL` | `0x0707` | Enable pin output/input, VREF level |
| `AC_VERB_GET_AMP_GAIN_MUTE` | `0x0B00` | Read amplifier gain/mute |
| `AC_VERB_SET_AMP_GAIN_MUTE` | `0x0300` | Write amplifier gain/mute |
| `AC_VERB_SET_POWER_STATE` | `0x0705` | Set D0–D3 power state |
| `AC_VERB_PARAMETERS` | `0x0F00` | Read widget parameter |

The `hda-verb` tool (from `alsa-tools`) sends raw verbs via the hwdep interface:

```bash
# Read codec vendor ID (requires CONFIG_SND_HDA_HWDEP=y)
hda-verb /dev/snd/hwC0D0 0x0 PARAMETERS VENDOR_ID

# Enable speaker pin (node 0x14, output enable bit = 0x40)
hda-verb /dev/snd/hwC0D0 0x14 SET_PIN_WIDGET_CONTROL 0x40

# Read pin jack-presence sense
hda-verb /dev/snd/hwC0D0 0x14 GET_PIN_SENSE 0
```

### 9.4 Codec Patch Functions

Each codec is identified by a 32-bit vendor+device ID. Codec drivers register device tables:

- `sound/pci/hda/patch_realtek.c` — Realtek ALC* family. All recent laptop codecs use `patch_alc269()`, which calls into `snd_hda_gen_parse_auto_config()` to auto-discover DAC/ADC paths from BIOS pin default configuration verbs. Laptop-specific quirks are keyed on PCI subsystem IDs via `SND_PCI_QUIRK` tables.
- `sound/pci/hda/patch_hdmi.c` — Intel, AMD, and NVIDIA HDMI/DisplayPort audio codecs.
- `sound/pci/hda/patch_conexant.c` — Conexant CX* codecs (CX5066, CX20952, etc.).

[Source](https://raw.githubusercontent.com/torvalds/linux/master/sound/pci/hda/patch_realtek.c)

### 9.5 HDMI Audio and ELD

HDMI audio in the HDA model is handled by `patch_hdmi.c`. Each HDMI output pin has an associated ELD (EDID-Like Data) buffer inside the codec widget. ELD is a compact descriptor extracted from the display's CEA-861 EDID extension block and describes supported audio formats, sample rates, and speaker allocation.

The DRM/KMS display driver calls `drm_edid_to_eld()` (`drivers/gpu/drm/drm_edid.c`) when a display is connected, writing ELD bytes into the HDA codec's per-pin ELD buffer. This bridges the graphics stack — which detects HDMI hotplug and parses EDID — to the HDA audio path. The ALSA driver then reads ELD via vendor-specific verbs and exposes it via `/proc/asound/card*/eld#*`. See Ch140 for details of the HDMI audio signal path.

On hotplug (detected via `AC_VERB_GET_PIN_SENSE`), `patch_hdmi.c` creates or binds a PCM device to the pin. On unplug, the PCM device is released. IEC 61937 passthrough (AC3, DTS, TrueHD, DTS-HD) requires the `AC_PINCAP_HBR` pin capability and sets the `AES0` non-audio bit (`0x02`) in the digital stream control verb.

To direct audio to an HDMI output:

```bash
# List PCM devices (HDMI appears as additional device numbers)
aplay -l

# Play to HDMI (card 0, device 3 in this example)
aplay -D hw:0,3 /usr/share/sounds/alsa/Front_Left.wav
```

### 9.6 Debugging HDA

```bash
# Full codec widget graph (widget type, caps, amp vals, pin config, connections)
cat /proc/asound/card0/codec#0

# Send raw verbs (requires alsa-tools hda-verb)
hda-verb /dev/snd/hwC0D0 0x14 GET_PIN_SENSE 0
```

The `/proc/asound/card0/codec#0` dump shows each NID, its widget type, capability word, amplifier limits, PCM formats, pin capabilities, BIOS default pin config, current pin control byte, power state, and connection list. This output is essential for diagnosing missing audio paths on laptops where the BIOS pin configuration disagrees with what `patch_realtek.c` expects. [Source](https://www.kernel.org/doc/html/latest/sound/hd-audio/notes.html)

---

## 10. ALSA Sequencer and MIDI

### 10.1 `/dev/snd/seq` Architecture

The ALSA sequencer (`/dev/snd/seq`) is a kernel MIDI event router. Applications create *clients* and *ports*; the sequencer dispatches `snd_seq_event_t` records between ports, either immediately (direct mode) or via a timed queue (scheduled mode with tick or real-time timestamps).

Key event types (from `include/alsa/seq_event.h`):

| Constant | Value | Description |
|----------|-------|-------------|
| `SND_SEQ_EVENT_NOTEON` | 6 | MIDI Note On |
| `SND_SEQ_EVENT_NOTEOFF` | 7 | MIDI Note Off |
| `SND_SEQ_EVENT_CONTROLLER` | 10 | Control change (CC) |
| `SND_SEQ_EVENT_PGMCHANGE` | 11 | Program change |
| `SND_SEQ_EVENT_PITCHBEND` | 13 | Pitch wheel |
| `SND_SEQ_EVENT_SYSEX` | 130 | System exclusive |

[Source](https://github.com/alsa-project/alsa-lib/blob/master/include/seq_event.h)

### 10.2 Client and Port API

```c
snd_seq_t *seq;
int client_id, port_id;

/* Open sequencer for duplex I/O */
snd_seq_open(&seq, "default", SND_SEQ_OPEN_DUPLEX, 0);
snd_seq_set_client_name(seq, "my_app");
client_id = snd_seq_client_id(seq);

/* Create a port capable of reading/writing and accepting subscriptions */
port_id = snd_seq_create_simple_port(seq, "input port",
    SND_SEQ_PORT_CAP_READ | SND_SEQ_PORT_CAP_SUBS_READ |
    SND_SEQ_PORT_CAP_WRITE | SND_SEQ_PORT_CAP_SUBS_WRITE,
    SND_SEQ_PORT_TYPE_MIDI_GENERIC | SND_SEQ_PORT_TYPE_APPLICATION);

/* Connect to port 0 of client 20 (a USB MIDI controller) */
snd_seq_connect_from(seq, port_id, 20, 0);
```

[Source](https://www.alsa-project.org/alsa-doc/alsa-lib/seq.html)

### 10.3 Virtual MIDI and Raw MIDI

**`snd_virmidi`** (kernel module `snd-virmidi`, `CONFIG_SND_VIRMIDI`) creates a virtual soundcard that bridges rawmidi devices (`/dev/snd/midiC*D*`) to ALSA sequencer ports. This allows applications expecting raw MIDI byte streams to interact with sequencer-native software synthesisers.

**Raw MIDI** (`/dev/snd/midiC0D0`) provides byte-stream MIDI I/O without sequencer event parsing:

```c
snd_rawmidi_t *rmidi;
snd_rawmidi_open(&rmidi, NULL, "hw:0,0", SND_RAWMIDI_SYNC);
unsigned char msg[3] = { 0x90, 0x3C, 0x7F };  /* Note On, C4, velocity 127 */
snd_rawmidi_write(rmidi, msg, 3);
```

MIDI 2.0 (UMP) devices appear at `/dev/snd/umpC*D*` with 32-bit-aligned 4-byte UMP word reads. Kernel support was added in v6.5 (`CONFIG_SND_USB_AUDIO_MIDI_V2`). [Source](https://docs.kernel.org/sound/designs/midi-2.0.html)

### 10.4 USB MIDI

The `sound/usb/midi.c` driver implements USB Class MIDI 1.0. It decodes 4-byte USB MIDI Event Packets (cable number + 3 MIDI bytes) from bulk endpoints into `snd_rawmidi` streams. USB MIDI devices appear as standard ALSA sequencer ports; `lsusb` shows the USB MIDI class, while `aconnect -l` shows the sequencer port. [Source](https://github.com/torvalds/linux/blob/master/sound/usb/midi.c)

### 10.5 CLI Tools

```bash
# List all ALSA sequencer clients and ports with connection status
aconnect -l

# Connect MIDI controller (client 20) to software synth (client 128)
aconnect 20:0 128:0

# Play a MIDI file to a sequencer port
aplaymidi -p 128:0 song.mid

# Record MIDI input to a file
arecordmidi -p 20:0 output.mid

# Interactive ncurses MIDI patchbay
aconnectgui
```

PipeWire 0.3.65+ added MIDI sequencer port support via the `api.alsa.seq.source` / `api.alsa.seq.sink` SPA factories, making ALSA MIDI ports available in the PipeWire graph alongside audio streams. `pw-midiplay` and `pw-midirecord` are the PipeWire equivalents of `aplaymidi`/`arecordmidi`.

---

## 11. Debugging and Diagnostics

### 11.1 Listing Devices

```bash
# List all PCM playback and capture devices
aplay -l
arecord -l

# List ALSA cards with driver names
cat /proc/asound/cards

# PCM device info (state, format, sample rate, buffer/period sizes)
cat /proc/asound/card0/pcm0p/info
cat /proc/asound/card0/pcm0p/sub0/status

# HDA codec widget graph dump
cat /proc/asound/card0/codec#0

# HDMI ELD (check populated after display hotplug)
cat /proc/asound/card*/eld#*
```

### 11.2 Test Tones and Playback

```bash
# Generate stereo sine test tone at 1 kHz
speaker-test -t sine -f 1000 -c 2

# Play directly to hardware device (bypassing PipeWire)
aplay -D hw:0,0 /usr/share/sounds/alsa/Front_Left.wav

# Play to HDMI output device
aplay -D hw:0,3 /usr/share/sounds/alsa/Front_Left.wav
```

### 11.3 Mixer State

```bash
# Interactive mixer TUI
alsamixer

# Command-line mixer (amixer)
amixer get Master
amixer set Master 70%
amixer set Capture 80% cap     # enable capture and set level

# Persist mixer state (survives reboot via /etc/asound.state)
alsactl store
alsactl restore
```

### 11.4 Comprehensive Diagnostic

`alsa-info.sh` (from the `alsa-utils` package) collects kernel version, ALSA version, `aplay -l` output, card info, codec dumps, UCM state, PCI IDs, and relevant `dmesg` lines into a single file suitable for bug reports at alsa-project.org:

```bash
alsa-info.sh --stdout 2>&1 | tee /tmp/alsa-info.txt
```

### 11.5 Latency Measurement

```bash
# Round-trip audio latency via loopback (connect line-out to line-in)
latency -P hw:0,0 -C hw:0,0 -p 64 -n 3
```

The `latency` tool from `alsa-utils` measures the minimum achievable round-trip latency at a given period size.

### 11.6 Common Failure Modes

**HDMI audio produces no output**: ELD not populated — check `cat /proc/asound/card*/eld#*`. If empty, the DRM driver did not call `drm_edid_to_eld()`, or the display connection is not yet active. Try cycling display power or reloading the HDA module with `rmmod snd_hda_codec_hdmi && modprobe snd_hda_codec_hdmi`.

**Microphone not detected on a laptop**: The UCM `SectionDevice` for the microphone may not have been activated. Check `alsaucm -c <cardname> -b - <<< $'set _verb HiFi\nget _enadev'` and verify the mic device was enabled. Also check `alsamixer` for muted capture channels.

**Audio crackling or drop-outs**: The application is hitting period underruns. Increase `period_size` and `buffer_size` in the ALSA configuration, or ensure the audio thread runs at `SCHED_FIFO` priority. For real-time threads use `pthread_setschedparam()` with `SCHED_FIFO` and `sched_priority = 90` (below kernel IRQ threads at 99).

**`hw:` device busy**: PipeWire or PulseAudio holds the device. To test the raw device, stop PipeWire (`systemctl --user stop pipewire pipewire-pulse wireplumber`) first, or use `pw-cli` to temporarily suspend the ALSA sink node.

---

## 12. Integrations

**Ch38 (PipeWire and the Video Session Layer)**
PipeWire's `spa-alsa` plugin uses the MMAP zero-copy path described in §4.4 of this chapter to read/write the ALSA DMA ring buffer without an extra copy. The `api.alsa.enum.udev` WirePlumber monitor (described in Ch38 §2) detects ALSA card hotplug events and instantiates `pw_node` objects. The `pipewire-alsa` PCM plugin described in §6.2 of this chapter is the reverse bridge: it routes `libasound` API calls into PipeWire streams, providing transparent compatibility for legacy applications. PipeWire's graph cycle timing is driven by the ALSA DMA period interrupt via the `api.alsa.pcm.sink` driver node.

**Ch140 (HDMI and DisplayPort Audio)**
The `patch_hdmi.c` codec driver (§9.5) integrates with the DRM/KMS display stack. `drm_edid_to_eld()` in `drivers/gpu/drm/drm_edid.c` converts EDID CEA-861 data into the HDA ELD byte format and injects it into the codec's per-pin ELD buffer on hotplug. IEC 61937 compressed passthrough for AC3/DTS/TrueHD is configured at the HDA verb level by the HDMI codec driver. Ch140 covers the complete HDMI audio signal path from the display connector through the codec graph.

**Ch58 (GStreamer)**
GStreamer's `alsasrc` and `alsasink` elements use `libasound` under the hood, exercising the hw_params negotiation API described in §3. `alsasrc` is commonly used in embedded pipeline designs where direct ALSA access is preferred over PipeWire routing. Ch58 covers the GStreamer pipeline model and how ALSA elements integrate with video pipelines.

**Ch57 (FFmpeg)**
`ffmpeg -f alsa -i hw:0,0 output.wav` captures audio directly from an ALSA PCM device using the ALSA demuxer, which calls `snd_pcm_open()`, negotiates hw_params, and calls `snd_pcm_readi()` in a loop. For embedded recording pipelines without PipeWire, this is the minimal capture path. Ch57 covers FFmpeg's full multimedia pipeline model.

**Ch60c (WebRTC Server Infrastructure)**
The GStreamer `alsasrc` element provides direct ALSA capture for embedded WHIP pipelines. ALSA latency tuning (small `period_size`, `SCHED_FIFO` threads) is critical for VoIP audio capture where jitter directly degrades call quality. The ALSA sequencer's timing model described in §10 is relevant for synchronising audio capture with network packet transmission.

**Ch189 (VLC)**
VLC's ALSA output plugin (`aout_alsa`) uses the `libasound` API described in §3 for direct hardware output, bypassing PipeWire when neither PipeWire nor PulseAudio is available. This is the fallback path for headless VLC deployments on embedded Linux.

---

*Copyright © 2026 jreuben11. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).*
