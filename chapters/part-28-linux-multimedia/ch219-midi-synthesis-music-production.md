# Chapter 219: MIDI, Synthesis, and Music Production on Linux: FluidSynth, LV2, SuperCollider, and DAW Integration

> **Part**: Part XXVIII — Linux Multimedia
> **Audiences**: Audio application developers, electronic music programmers, and systems engineers building music production infrastructure on Linux.
> **Status**: First draft — 2026-07-22

---

## Table of Contents

- [Overview](#overview)
- [1. MIDI Fundamentals: Byte-Level Protocol and MIDI 2.0](#1-midi-fundamentals-byte-level-protocol-and-midi-20)
- [2. ALSA Sequencer: Client/Port Model and Routing](#2-alsa-sequencer-clientport-model-and-routing)
- [3. PipeWire MIDI: Graph Routing and UMP Nodes](#3-pipewire-midi-graph-routing-and-ump-nodes)
- [4. USB MIDI: Class-Compliant Driver and MIDI 2.0 UMP over USB](#4-usb-midi-class-compliant-driver-and-midi-20-ump-over-usb)
- [5. FluidSynth: SoundFont Renderer and Embedding API](#5-fluidsynth-soundfont-renderer-and-embedding-api)
- [6. SoundFont Ecosystem: SF2 Internals and Free Banks](#6-soundfont-ecosystem-sf2-internals-and-free-banks)
- [7. LV2 Plugin Standard: Turtle Manifest and Core Ontology](#7-lv2-plugin-standard-turtle-manifest-and-core-ontology)
- [8. LV2 Synthesizer Programming](#8-lv2-synthesizer-programming)
- [9. LADSPA, VST3, and Plugin Bridging](#9-ladspa-vst3-and-plugin-bridging)
- [10. Ardour DAW Internals](#10-ardour-daw-internals)
- [11. SuperCollider: scsynth, UGen Graph, and OSC Control](#11-supercollider-scsynth-ugen-graph-and-osc-control)
- [12. Csound: Orchestra/Score Language and Embedding](#12-csound-orchestrascore-language-and-embedding)
- [13. Surge XT Synthesizer: Open-Source Hybrid Synth](#13-surge-xt-synthesizer-open-source-hybrid-synth)
- [14. CLAP Plugin Standard](#14-clap-plugin-standard)
- [15. Sequencing and Notation: MuseScore, LilyPond, and MIDI Pipelines](#15-sequencing-and-notation-musescore-lilypond-and-midi-pipelines)
- [16. Real-Time Safety: Lock-Free Buffers and RT Scheduling](#16-real-time-safety-lock-free-buffers-and-rt-scheduling)
- [Integrations](#integrations)

---

## Overview

Linux provides a full-stack music production environment spanning the MIDI wire protocol, synthesis engines, plugin standards, and professional DAW infrastructure. This chapter covers the entire signal chain for audio application developers: from raw MIDI byte encoding through the ALSA sequencer's client/port routing model and PipeWire's UMP graph, into SoundFont synthesis with FluidSynth, plugin authorship under LV2 and CLAP, and high-level production tools including Ardour, SuperCollider, and Csound. Systems engineers will find detailed coverage of USB MIDI class-compliance, kernel UMP support introduced in Linux 6.5, and real-time scheduling constraints that govern every audio callback.

For the kernel-level ALSA PCM engine and ASoC framework see Ch38b; for PipeWire's session management and audio graph internals see Ch38.

---

## 1. MIDI Fundamentals: Byte-Level Protocol and MIDI 2.0

### 1.1 MIDI 1.0 Byte Encoding

MIDI 1.0 is a serial byte stream operating at 31.25 kbaud. Every message consists of a **status byte** (MSB=1, values 0x80–0xFF) followed by zero or more **data bytes** (MSB=0, values 0x00–0x7F). Channel messages encode the message type in the upper nibble and the target channel (0–15) in the lower nibble: [Source](https://midi.org/about-midi-part-3midi-messages)

| Status (hex) | Message | Data Bytes |
|---|---|---|
| `0x8n` | Note Off | key, velocity |
| `0x9n` | Note On | key, velocity |
| `0xAn` | Poly Aftertouch | key, pressure |
| `0xBn` | Control Change (CC) | controller, value |
| `0xCn` | Program Change | program |
| `0xDn` | Channel Aftertouch | pressure |
| `0xEn` | Pitch Bend | LSB, MSB (14-bit, center=0x2000) |

SysEx frames begin with `0xF0`, carry a manufacturer ID and arbitrary payload, and terminate with `0xF7`. Running status allows the status byte to be omitted when consecutive messages share the same status, reducing bandwidth in dense MIDI streams.

### 1.2 MIDI 2.0 and Universal MIDI Packet

The MIDI 2.0 specification (published February 20, 2020) introduces the **Universal MIDI Packet** (UMP) as the new wire format. UMP messages are 32-bit aligned: [Source](https://midi.org/universal-midi-packet-ump-and-midi-2-0-protocol-specification)

- **32-bit**: utility messages, system messages, and MIDI 1.0 channel voice messages
- **64-bit**: MIDI 2.0 channel voice messages carrying 32-bit velocity and controller values
- **128-bit**: SysEx 8 and mixed data sets

UMP organises addresses into 16 **Groups** × 16 channels = 256 addressable channels (vs. 16 in MIDI 1.0). **MIDI-CI** (Capability Inquiry) uses the legacy MIDI 1.0 transport as a negotiation handshake before switching to the MIDI 2.0 protocol, making it backward compatible.

Linux kernel MIDI 2.0 support (initial UMP framework) merged in Linux 6.5 (2023) via `CONFIG_SND_UMP`. UMP rawmidi devices appear at `/dev/snd/umpC*D*`, distinct from legacy `/dev/snd/midiC*D*`. [Source](https://docs.kernel.org/sound/designs/midi-2.0.html)

---

## 2. ALSA Sequencer: Client/Port Model and Routing

The ALSA sequencer (`snd_seq`) is a kernel-level MIDI event router with timing and subscription infrastructure. For the kernel driver internals and ASoC integration see Ch38b; this section focuses on the application-facing API in `alsa-lib`. [Source](https://www.alsa-project.org/alsa-doc/alsa-lib/seq.html)

### 2.1 Clients, Ports, and Subscriptions

Each ALSA sequencer **client** is an integer ID identifying a process or kernel entity. Clients own one or more **ports** typed by capability flags:

- `SND_SEQ_PORT_CAP_READ` / `SND_SEQ_PORT_CAP_WRITE` — local read/write
- `SND_SEQ_PORT_CAP_SUBS_READ` / `SUBS_WRITE` — allows external subscriptions

A **subscription** wires a sender port to a destination port; MIDI events flow along subscriptions. The special address `SND_SEQ_ADDRESS_SUBSCRIBERS` broadcasts to all subscribed destinations; `SND_SEQ_QUEUE_DIRECT` dispatches events immediately without timing.

```c
/* alsa-lib sequencer client setup */
/* From: alsa-project.org/alsa-doc/alsa-lib/seq.html */
snd_seq_t *seq;
snd_seq_open(&seq, "default", SND_SEQ_OPEN_DUPLEX, 0);
snd_seq_set_client_name(seq, "My MIDI App");

int port = snd_seq_create_simple_port(seq, "MIDI In",
    SND_SEQ_PORT_CAP_WRITE | SND_SEQ_PORT_CAP_SUBS_WRITE,
    SND_SEQ_PORT_TYPE_MIDI_GENERIC);

/* Subscription: wire another port as sender to this port */
snd_seq_port_subscribe_t *sub;
snd_seq_port_subscribe_alloca(&sub);
snd_seq_port_subscribe_set_sender(sub, &src_addr);  /* snd_seq_addr_t */
snd_seq_port_subscribe_set_dest(sub, &dst_addr);
snd_seq_port_subscribe_set_time_update(sub, 1);
snd_seq_subscribe_port(seq, sub);
```

The `snd_seq_event_t` structure carries: type, flags, time (tick or real-time clock), queue ID, source and destination `snd_seq_addr_t` (client+port pair), and a 12-byte fixed data payload. For MIDI 2.0 UMP events the `snd_seq_ump_event_t` extends this to a 16-byte payload when the `SNDRV_SEQ_EVENT_UMP` flag is set. An application switches a client to UMP mode with:

```c
snd_seq_set_client_midi_version(seq, SND_SEQ_CLIENT_UMP_MIDI_2_0);
/* ALSA automatically translates between UMP and legacy event formats */
```

### 2.2 Queue Timing Model

The sequencer's **queue** object provides tempo-based event scheduling. Events can carry a timestamp in two modes: absolute ticks (musical time) or real-time clock (nanoseconds from queue start). An application creates a queue with `snd_seq_alloc_named_queue()`, sets tempo via `snd_seq_change_queue_tempo()`, and starts it with `snd_seq_start_queue()`. The `SND_SEQ_QUEUE_DIRECT` special value bypasses queuing entirely, dispatching the event immediately — necessary for latency-sensitive applications such as live MIDI keyboard input.

Event types of interest to application developers:

| Type constant | Meaning |
|---|---|
| `SND_SEQ_EVENT_NOTEON` | Note On |
| `SND_SEQ_EVENT_NOTEOFF` | Note Off |
| `SND_SEQ_EVENT_CONTROLLER` | Control Change |
| `SND_SEQ_EVENT_PITCHBEND` | Pitch Bend |
| `SND_SEQ_EVENT_PGMCHANGE` | Program Change |
| `SND_SEQ_EVENT_SYSEX` | System Exclusive (variable-length) |
| `SND_SEQ_EVENT_TEMPO` | Tempo change (SMF meta event equivalent) |
| `SND_SEQ_EVENT_TIMESIG` | Time signature |
| `SND_SEQ_EVENT_CLOCK` | MIDI clock tick |

SysEx events use a variable-length `ext` union field rather than the fixed 12-byte `data` payload.

### 2.3 CLI Tools

| Tool | Function |
|---|---|
| `aconnect -l` | List all ALSA seq clients and ports |
| `aconnect 20:0 128:0` | Subscribe port 20:0 (sender) to 128:0 (destination) |
| `aplaymidi -p 128:0 piece.mid` | Play an SMF file via seq, sending to port 128:0 |
| `amidi -p hw:1,0 -s note_on.bin` | Send raw MIDI bytes to rawmidi device |
| `aseqdump -p 20:0` | Monitor and display events arriving on port 20:0 |

---

## 3. PipeWire MIDI: Graph Routing and UMP Nodes

PipeWire (v1.0+) integrates MIDI as first-class nodes in its media graph. For PipeWire session management and graph architecture see Ch38; this section covers MIDI-specific behaviour. [Source](https://docs.pipewire.org/page_midi.html)

### 3.1 SPA MIDI Data Format

PipeWire MIDI nodes use the SPA media type `application/control`. Events are carried in an `spa_pod_sequence` with `SPA_CONTROL_UMP` controls — each control encodes a 32-bit UMP word and a sample-accurate timestamp offset within the current buffer period. The port format is declared as `"32 bit raw UMP"` via `PW_KEY_FORMAT_DSP`.

```c
/* PipeWire MIDI source node: pw-docs/midi-src_8c-example.html */
data.filter = pw_filter_new_simple(
    pw_main_loop_get_loop(data.loop), "midi-src",
    pw_properties_new(
        PW_KEY_MEDIA_TYPE,     "Midi",
        PW_KEY_MEDIA_CATEGORY, "Playback",
        PW_KEY_MEDIA_CLASS,    "Midi/Source",
        NULL),
    &filter_events, &data);

/* Build MIDI event buffer in the process callback */
spa_pod_builder_push_sequence(&builder, &frame, 0);
spa_pod_builder_control(&builder, sample_offset, SPA_CONTROL_UMP);
spa_pod_builder_bytes(&builder, &ump_event, sizeof(ump_event));
spa_pod_builder_pop(&builder, &frame);
/* d->chunk->stride = 1; d->chunk->offset = 0; */
```

[Source](https://docs.pipewire.org/midi-src_8c-example.html)

### 3.2 Session Manager Policy and MIDI 1.0 Preference

WirePlumber creates MIDI nodes by bridging ALSA sequencer ports via the SPA ALSA Sequencer Bridge. PipeWire v1.7 reverted its default preference back to MIDI 1.0 because the UMP ↔ MIDI 1.0 conversion is not fully transparent in all scenarios — controllers expecting MIDI 1.0 semantics on the other side of a UMP node may lose resolution or running-status context.

Manual routing uses `pw-link` on the command line, or `qpwgraph` for a GUI. PipeWire MIDI bridges the legacy JACK-MIDI and ALSA-seq ecosystems: JACK MIDI is mapped 1:1 with no information loss; ALSA seq events are translated through the bridge with automatic format conversion.

---

## 4. USB MIDI: Class-Compliant Driver and MIDI 2.0 UMP over USB

The USB Audio Class 2.0 specification defines a MIDI streaming interface that Linux implements in `sound/usb/midi.c` within the `snd-usb-audio` driver. Class-compliant devices require no vendor driver — the kernel enumerates the USB descriptor and exposes rawmidi and sequencer ports automatically.

### 4.1 Linux USB MIDI 2.0 Support

MIDI 2.0 over USB uses the **UMP Endpoint** interface (introduced in USB Audio Class 3.0 MIDI 2.0 spec). The `snd-usb-audio` driver (Linux 6.5+) defaults to `altset 1` (MIDI 2.0 interface) when present, falling back to `altset 0` (MIDI 1.0) for devices that lack it. Module parameter `midi2_enable=0` forces MIDI 1.0 operation. The driver probes UMP Endpoint and Group Terminal Block information (UMP v1.1+) to build the topology seen by userspace. [Source](https://docs.kernel.org/sound/designs/midi-2.0.html)

New ioctls expose UMP topology to userspace:

```bash
# List UMP devices (distinct from legacy /dev/snd/midiC*D*)
ls /dev/snd/umpC*D*

# Query UMP endpoint info (SNDRV_UMP_IOCTL_ENDPOINT_INFO)
# Query Group Terminal Block info (SNDRV_UMP_IOCTL_BLOCK_INFO)
```

Kernel build options: `CONFIG_SND_UMP`, `CONFIG_SND_UMP_LEGACY_RAWMIDI`, `CONFIG_SND_SEQ_UMP`, `CONFIG_SND_SEQ_UMP_CLIENT`, `CONFIG_SND_USB_AUDIO_MIDI_V2`. The `midi_version` field in UMP ioctls distinguishes: 0 = legacy MIDI, 1 = MIDI 1.0 UMP, 2 = MIDI 2.0 UMP.

### 4.2 Class-Compliant vs. Proprietary Firmware

Most modern USB keyboards, controllers, and interfaces are class-compliant and work without extra software. Proprietary devices (some digital audio workstations, certain MIDI 2.0 beta hardware) may require vendor kernel modules or rely on firmware-specific USB descriptors that differ from the USB MIDI specification. The `usb-audio.quirks` table in the driver handles known deviations.

---

## 5. FluidSynth: SoundFont Renderer and Embedding API

FluidSynth is the reference SoundFont 2/SF3 software synthesizer, usable as a standalone daemon or as an embeddable library via `libfluidsynth`. Version 2.5.4 is current as of 2026. [Source](https://www.fluidsynth.org/api/)

### 5.1 Embedding API

The FluidSynth embedding lifecycle follows a settings → synth → audio driver pattern:

```c
/* From: fluidsynth.org/api/fluidsynth_simple_8c-example.html */
/* Build: gcc -g -O -o prog prog.c -lfluidsynth */
#include <fluidsynth.h>

fluid_settings_t *settings = new_fluid_settings();
fluid_synth_t    *synth    = new_fluid_synth(settings);
fluid_audio_driver_t *adrv = new_fluid_audio_driver(settings, synth);

/* Load a SoundFont; 1 = update MIDI bank presets immediately */
int sfid = fluid_synth_sfload(synth, "/usr/share/sounds/sf2/FluidR3_GM.sf2", 1);
if (sfid == FLUID_FAILED)
    fprintf(stderr, "%s\n", fluid_synth_error(synth));

/* Send MIDI: channel 0, middle C (key 60), velocity 100 */
fluid_synth_noteon(synth, 0, 60, 100);
/* ... at note end: */
fluid_synth_noteoff(synth, 0, 60);

/* Cleanup */
delete_fluid_audio_driver(adrv);
delete_fluid_synth(synth);
delete_fluid_settings(settings);
```

Key API calls: [Source](https://www.fluidsynth.org/api/group__synth.html)
- `fluid_synth_cc(synth, channel, controller, value)` — send Control Change
- `fluid_synth_pitch_bend(synth, channel, val)` — 14-bit pitch bend (0–16383)
- `fluid_synth_program_change(synth, channel, program)` — switch preset
- `fluid_synth_get_cpu_load(synth)` — returns percentage CPU used by synthesis

### 5.2 Synthesis Engine Internals

FluidSynth's synthesis engine: the polyphonic voice allocator instantiates a voice per Note On; each voice processes:
1. ADSR volume envelope (attack, decay, sustain, release) from SF2 `igen` generators
2. ADSR filter envelope driving a biquad lowpass filter (cutoff, resonance from SF2 generators)
3. Sample playback with selectable interpolation: linear, cubic, or 7th-order sinc
4. Per-voice pitch modulation (LFO, pitch envelope)
5. Reverb send (Schroeder–Moorer algorithm) and chorus send (multi-voice delay)

Real-time MIDI CC messages directly modulate synthesis parameters: CC 7 = volume, CC 10 = pan, CC 64 = sustain pedal hold, CC 91 = reverb depth, CC 93 = chorus depth. Settings `synth.audio-driver` selects the output backend (`alsa`, `pipewire`, `jack`, `pulseaudio`).

### 5.3 FluidSynth Shell and Settings

The standalone `fluidsynth` binary exposes an interactive shell for testing and live performance:

```bash
# Launch FluidSynth with PipeWire audio and ALSA MIDI input
fluidsynth -a pipewire -m alsa_seq /usr/share/sounds/sf2/FluidR3_GM.sf2

# Inside the fluidsynth shell:
> noteon 0 60 100      # channel 0, middle C, velocity 100
> cc 0 7 64            # channel 0, CC 7 (volume) = 64
> prog 0 40            # channel 0, program 40 (violin)
> gain 1.5             # master output gain
> reverb on
> chorus off
```

Key `fluid_settings_t` string keys for embedding: `audio.driver` (`pipewire`, `alsa`, `jack`), `midi.driver` (`alsa_seq`, `pipewire`, `jack`), `synth.polyphony` (default 256), `synth.sample-rate` (default 44100), `synth.interpolation` (`0`=none, `1`=linear, `4`=cubic, `7`=sinc). [Source](https://www.fluidsynth.org/api/LoadingSoundfonts.html)

---

## 6. SoundFont Ecosystem: SF2 Internals and Free Banks

SF2 is a RIFF-based binary format with a three-tier hierarchy: **Presets** (bank/program number → instrument mapping) → **Instruments** (key/velocity zone mapping → sample layers) → **Samples** (PCM audio stored in the `smpl` chunk as 16-bit signed integers). [Source](https://docs.fileformat.com/audio/sf2/)

### 6.1 Key Chunks and Generators

Critical RIFF chunks:

| Chunk | Content |
|---|---|
| `smpl` | 16-bit PCM sample data (SF3: OGG Vorbis compressed) |
| `phdr` | Preset headers: name, bank, program number, ibag index |
| `inst` | Instrument headers: name, ibag index |
| `igen` | Instrument generators: 58 types (keyRange, velRange, sampleID, loop start/end, attack/decay/sustain/release envelope times, filter cutoff, resonance, pan, coarse/fine tune) |
| `shdr` | Sample headers: name, start/end frames, loop start/end, sample rate, root pitch |

Velocity splits are implemented by giving an instrument multiple zones with overlapping or adjacent `velRange` generators, each pointing to a different sample (e.g., soft, medium, hard piano strikes). Loop points (`loopStart`, `loopEnd`) in `shdr` enable sustain loops; a second `sampleModes` generator selects loop mode.

### 6.2 SF3 and Free Banks

**SF3** replaces the `smpl` chunk with OGG Vorbis-compressed samples, documented at the FluidSynth wiki. FluidSynth 2.x natively decompresses SF3 at load time.

Free General MIDI banks: **GeneralUser GS** (128 GM melodic instruments + percussion, widely used in education software) and **Fluid R3 GM SoundFont** (the default on many Linux distributions, available at `/usr/share/sounds/sf2/FluidR3_GM.sf2`).

---

## 7. LV2 Plugin Standard: Turtle Manifest and Core Ontology

LV2 (LADSPA Version 2) is the open plugin standard for Linux audio, defined by an RDF/Turtle ontology and a C descriptor API. Core version 18.6 is current. [Source](https://lv2plug.in/ns/lv2core)

### 7.1 Manifest and Plugin Discovery

Each LV2 plugin is a bundle directory containing a `manifest.ttl` file and one or more `.so` shared libraries. The Turtle manifest declares the plugin URI, binary, and a link to its full description: [Source](https://lv2plug.in/book/)

```turtle
# manifest.ttl
@prefix lv2:  <http://lv2plug.in/ns/lv2core#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .

<http://example.org/plugins/mysynth>
    a lv2:Plugin, lv2:InstrumentPlugin ;
    lv2:binary <mysynth.so> ;
    rdfs:seeAlso <mysynth.ttl> .
```

The full plugin Turtle (`mysynth.ttl`) declares ports with types: `lv2:AudioPort`, `lv2:ControlPort`, and `atom:AtomPort` (with `atom:bufferType atom:Sequence` and `atom:supports midi:MidiEvent`) for MIDI input.

### 7.2 LV2_Descriptor and Feature Negotiation

The plugin binary exports a single `lv2_descriptor()` entry point returning a `LV2_Descriptor`:

```c
static const LV2_Descriptor descriptor = {
    "http://example.org/plugins/mysynth",  /* URI */
    instantiate,    /* (Descriptor*, rate, bundle_path, features[]) -> Handle */
    connect_port,   /* (Handle, port_index, data_location) */
    activate,
    run,            /* (Handle, sample_count) -- must be RT-safe */
    deactivate,
    cleanup,
    extension_data  /* (uri) -> const void* -- for extensions */
};

LV2_SYMBOL_EXPORT const LV2_Descriptor*
lv2_descriptor(uint32_t index) {
    return (index == 0) ? &descriptor : NULL;
}
```

Features the host optionally provides at `instantiate` time (e.g., `LV2_URID__map` for URI-to-integer mapping, `LV2_Worker__schedule` for the worker extension) are passed as a NULL-terminated array of `LV2_Feature*`. A plugin declares `lv2:requiredFeature` or `lv2:optionalFeature` in its Turtle to indicate what it needs. The `lv2:hardRTCapable` feature declares that `run()` is real-time safe.

The **lilv** host library discovers and loads LV2 plugins: `lilv_world_new()` creates a world; `lilv_world_load_all()` scans `LV2_PATH` directories and parses all `manifest.ttl` files; `lilv_plugin_instantiate()` creates an instance. [Source](https://drobilla.net/docs/lilv/)

---

## 8. LV2 Synthesizer Programming

### 8.1 MIDI Processing in run()

An LV2 instrument plugin receives MIDI via an `atom:AtomPort` input. The `run()` callback iterates events in the atom sequence and processes interleaved audio and MIDI: [Source](https://lv2plug.in/book/)

```c
/* libs/lv2/book: EG-amp.lv2 pattern adapted for MIDI synth */
static void
run(LV2_Handle instance, uint32_t sample_count)
{
    MySynth *self   = (MySynth *)instance;
    uint32_t offset = 0;

    LV2_ATOM_SEQUENCE_FOREACH(self->control_in, ev) {
        if (ev->body.type == self->uris.midi_MidiEvent) {
            /* Process audio samples up to this event's sample offset */
            synth_render(self, offset, (uint32_t)ev->time.frames - offset);
            offset = (uint32_t)ev->time.frames;

            const uint8_t *msg = (const uint8_t *)(ev + 1);
            switch (lv2_midi_message_type(msg)) {  /* lv2/midi/midi.h */
            case LV2_MIDI_MSG_NOTE_ON:
                note_on(self, msg[1], msg[2]);  break;
            case LV2_MIDI_MSG_NOTE_OFF:
                note_off(self, msg[1]);          break;
            case LV2_MIDI_MSG_CONTROLLER:
                handle_cc(self, msg[1], msg[2]); break;
            }
        }
    }
    synth_render(self, offset, sample_count - offset); /* remaining samples */
}
```

The `ev->time.frames` field gives a sample-accurate offset within the current buffer, enabling tight MIDI-to-audio synchronisation with no audible jitter.

### 8.2 Worker Extension and State Serialisation

For non-realtime operations (loading a sample file, SoundFont, or patch), the LV2 Worker extension bridges between threads:

```c
/* work() runs in a non-RT thread — file I/O is safe here */
static LV2_Worker_Status
work(LV2_Handle instance, LV2_Worker_Respond_Function respond,
     LV2_Worker_Respond_Handle handle, uint32_t size, const void *data)
{
    Sample *s = load_sample_file((const char *)data);  /* blocking OK */
    respond(handle, sizeof(Sample *), &s);
    return LV2_WORKER_SUCCESS;
}

/* work_response() called in the RT audio thread after respond() */
static LV2_Worker_Status
work_response(LV2_Handle instance, uint32_t size, const void *data)
{
    MySynth *self = (MySynth *)instance;
    self->sample  = *(Sample *const *)data;  /* atomic swap */
    return LV2_WORKER_SUCCESS;
}
```

State serialisation uses the `LV2_State_Interface` with `save()` / `restore()` callbacks, marking data as `LV2_STATE_IS_POD` for portability across hosts. URID mapping (URI → `uint32_t` integer) makes atom type comparisons real-time safe — the map is populated at `instantiate` time.

### 8.3 Compiling an LV2 Plugin

An LV2 plugin is compiled as a shared library against the `lv2` header package (headers only — no link-time LV2 library):

```bash
# Install: apt install lv2-dev
# Compile: plugin.so has no link-time dependency on an lv2 library
gcc -shared -fPIC -O2 -o mysynth.so mysynth.c \
    $(pkg-config --cflags lv2) \
    -DPLUGIN_URI=\"http://example.org/plugins/mysynth\"

# Install into user LV2 path:
mkdir -p ~/.lv2/mysynth.lv2
cp mysynth.so manifest.ttl mysynth.ttl ~/.lv2/mysynth.lv2/

# Discover and inspect via lilv-utils (from the lilv package):
lv2ls | grep mysynth
lv2info http://example.org/plugins/mysynth
```

The `LV2_PATH` environment variable overrides default search directories (`~/.lv2`, `/usr/lib/lv2`, `/usr/local/lib/lv2`). Hosts call `lilv_world_load_all()` to parse all `manifest.ttl` files found on the path. [Source](https://github.com/lv2/lilv)

---

## 9. LADSPA, VST3, and Plugin Bridging

### 9.1 LADSPA: Legacy Foundation

LADSPA (Linux Audio Developer's Simple Plugin API) defines the `LADSPA_Descriptor` struct with `instantiate`, `connect_port`, `activate`, `run`, `deactivate`, and `cleanup` callbacks. [Source](https://ladspa.org/)

LADSPA processes only audio and control data — it has no MIDI port mechanism and cannot implement instruments or handle program changes. It remains widely installed because thousands of LADSPA effect plugins exist in distribution repositories. LV2 is the modern successor that supersedes it for new development.

**DSSI** (Disposable Soft Synth Interface) extended LADSPA with a MIDI input mechanism, enabling software instruments. DSSI plugins are still found in distribution repositories (e.g., `hexter` Yamaha DX-7 emulator, `WhySynth`) but share LADSPA's lack of GUI embedding, custom UIs, and state serialisation — all features that LV2 handles natively.

### 9.2 VST3 on Linux

Steinberg's VST3 SDK (`steinbergmedia/vst3sdk`) supports Linux as a first-class platform. VST3 plugins are shared libraries under a `.vst3` bundle directory, loaded via `GetPluginFactory()`. Commercial DAWs (Reaper, Bitwig) support VST3 on Linux natively; Ardour 8+ added VST3 support.

**yabridge** bridges Windows VST2, VST3, and CLAP plugins to Linux via Wine with a host-side proxy and a Wine-side stub communicating over shared memory: [Source](https://github.com/robbert-vdh/yabridge)

```bash
# Install a Windows VST3 under Wine, then create a yabridge link:
yabridgectl add "$HOME/.wine/drive_c/Program Files/Steinberg/VstPlugins"
yabridgectl sync
# Linux DAW sees the plugin as a native .so with Wine handling the Windows binary
```

yabridge 5.x supports 32/64-bit bridging (bitbridge), plugin groups for VST2 inter-plugin communication, and CLAP bridging.

### 9.3 Carla Plugin Host

**Carla** is a plugin host and patchbay that loads LADSPA, DSSI, LV2, VST2, VST3, and AU plugins within a single process. It exposes them as JACK/ALSA/PipeWire endpoints and supports plugin chaining, parameter automation, and preset management — useful as a universal plugin rack for DAWs that do not natively support all formats. [Source](https://kx.studio/Applications:Carla)

---

## 10. Ardour DAW Internals

Ardour is a professional DAW with approximately 1 million lines of C++ (210k UI via gtkmm, 160k engine in libardour). [Source](https://ardour.org/development.html)

### 10.1 Session Engine and Object Hierarchy

The `Session` object owns: an `AudioEngine` (backend abstraction over ALSA, JACK, or PipeWire), `Track` objects (`AudioTrack` and `MidiTrack`), `Route` signal paths (source → insert chain → master bus), `Source` objects (`AudioSource` / `MidiSource` backed by files on disk), and the `TempoMap`.

**BBT time**: Ardour represents MIDI positions internally as `BBT_Position` (Bars:Beats:Ticks, where 1 beat = 1920 ticks). The `TempoMap` maps between audio sample frames and BBT positions, handling tempo changes and time signature events. MIDI export preserves BBT timing accurately in Standard MIDI File (SMF) format.

### 10.2 MIDI Pipeline

```
MidiSource (SMF .mid file)
  → MidiDiskstream (real-time disk read ahead, rt-safe ring buffer)
  → MidiTrack
  → LV2 instrument plugin (via libs/ardour/lv2_plugin.cc)
  → audio bus / master
```

The LV2 plugin graph is wired by Ardour's `lv2_plugin.cc` which calls `lilv_plugin_instantiate()`, maps all ports, and calls `run()` from the process callback. MIDI data flows as `atom:Sequence` events with sample-accurate frame timestamps derived from the BBT grid. [Source](https://github.com/Ardour/ardour/blob/master/libs/ardour/lv2_plugin.cc)

Non-realtime export renders the full session to a file via `libsndfile`, supporting FLAC, WAV, AIFF, and Opus output formats.

### 10.3 Asynchronous Signal/Callback Architecture

Ardour decouples its C++ backend from the UI via an asynchronous signal and callback system (documented on the Ardour development page). The audio engine fires typed signals that UI observers subscribe to; these are queued and dispatched on the GUI thread, preventing blocking in the audio callback. [Source](https://ardour.org/development.html) Plugin parameter changes from the GUI travel through a lock-free command queue (similar to the ring buffer pattern in §16) into the process callback, maintaining RT safety in the audio thread while providing responsive parameter control from the UI.

---

## 11. SuperCollider: scsynth, UGen Graph, and OSC Control

SuperCollider 3.14.1 (released November 24, 2025) is a four-component system: **scsynth** (real-time audio server), **supernova** (parallel DSP alternative), **sclang** (interpreted programming language), and **scide** (IDE). [Source](https://github.com/supercollider/supercollider)

### 11.1 Client–Server Architecture

`scsynth` accepts OSC (Open Sound Control) messages over UDP or TCP (default port 57110). The server maintains a **node tree** of Synths and Groups. `sclang` compiles `SynthDef` objects to a binary wire format and sends them to the server:

```supercollider
// sclang client: doc.sccode.org/Classes/SynthDef.html
SynthDef.new(\mySine, { |freq = 440, amp = 0.5, out = 0|
    var sig = SinOsc.ar(freq, 0, amp);
    Out.ar(out, sig)
}).add;   // compiles and transmits /d_recv OSC message to scsynth

// Instantiate a synth node (sends /s_new):
x = Synth.new(\mySine, [\freq, 880, \amp, 0.3]);

// Modify a running synth's parameters (sends /n_set):
x.set(\freq, 220);

// Free the node (sends /n_free):
x.free;
```

[Source](https://doc.sccode.org/Classes/SynthDef.html)

Argument name prefixes carry semantic meaning: `a_` = audio-rate bus mapping (`n_mapa`), `i_` = initial-rate (evaluated once), `t_` = trigger (generates an impulse on any change).

### 11.2 UGen Graph and RT Memory

A **UGen** (Unit Generator) is the base DSP class; hundreds of built-ins cover oscillators, filters, envelopes, noise generators, convolution, FFT analysis, and spatial panning. The UGen graph is a topologically sorted list; scsynth executes each UGen's `next()` method per audio block.

**RTAlloc**: scsynth pre-allocates a fixed memory pool at startup for UGen allocations. No `malloc()` calls occur inside the audio callback — this is enforced by the pool allocator. `supernova` extends this with a thread-pool parallel DSP engine for multi-core systems.

### 11.3 Linux Audio Backend

On Linux, scsynth uses JACK by default. PipeWire is accessed via the `pw-jack` compatibility shim (`pw-jack scsynth ...`), which presents PipeWire streams as JACK ports. Direct ALSA output is also supported. [Source](https://doc.sccode.org/Reference/AudioDeviceSelection.html)

---

## 12. Csound: Orchestra/Score Language and Embedding

Csound 6.18 is a programmable audio synthesis engine with approximately 1,700 built-in opcodes. Csound 7.0 is in development as of 2026. [Source](https://csound.com/docs/api/index.html)

### 12.1 Orchestra/Score Model

A Csound **orchestra** (`.orc`) defines instruments as `instr`/`endin` blocks containing chains of opcodes. A **score** (`.sco`) schedules note events with `i` statements (instrument number, start time, duration, and p-fields). The **CSD** format wraps both in an XML-like container for single-file distribution.

Time granularity: **i-rate** opcodes initialise once per note; **k-rate** opcodes run at the control rate (`sr/ksmps`, typically 48000/64 = 750 Hz); **a-rate** opcodes run at audio rate.

### 12.2 C Embedding API

```c
/* From: flossmanual.csound.com/csound-and-other-programming-languages/the-csound-api */
/* Link: -lcsound64 */
#include <csound/csound.h>

CSOUND *cs = csoundCreate(NULL);
csoundSetOption(cs, "-odac");           /* output to DAC */
csoundCompileOrc(cs,
    "instr 1\n"
    "  a1 oscil 0.5, p4\n"             /* p4 = frequency from score */
    "  out a1\n"
    "endin\n");
csoundReadScore(cs, "i 1 0 3 440\n");  /* instrument 1, t=0, dur=3s, freq=440 */
csoundStart(cs);
while (csoundPerformKsmps(cs) == 0) {  /* advance one k-block per call */
    /* host may modify parameters here between k-cycles */
}
csoundCleanup(cs); csoundDestroy(cs);
```

Plugin opcodes implement the `CSOUND_PLUGIN_INIT` macro and the `OpcodeBase.hpp` C++ template (from `csdl.h`). 

### 12.3 FAUST Integration

**FAUST** (Functional Audio Stream) is a domain-specific language for real-time DSP that compiles to Csound opcodes via the `faust2csound` tool. The toolchain generates C++ opcode source from a `.dsp` FAUST file, compiled as a Csound plugin `.so`. This allows rapid development of audio algorithms in FAUST's functional style while deploying inside a Csound orchestra. The FAUST-to-Csound tooling moved to a separate repository from the Csound main tree; consult the current FAUST project (`faust.grame.fr`) for the active target.

### 12.4 WebAssembly Deployment

Csound's core library compiles to WebAssembly via Emscripten (the Csound repository includes a `wasm/` build directory), enabling score playback and interactive synthesis in the browser without a server-side process. This deployment path is used by online educational tools and score-sharing sites. The API surface in the WebAssembly build mirrors the C API's `csoundCreate`/`csoundCompileOrc`/`csoundPerformKsmps` lifecycle. Consult the Csound repository's `wasm/` directory for current package names and build instructions.

### 12.5 Experimental GPU Opcodes

Csound 6 includes experimental GPU opcodes that offload table generation and signal processing to CUDA or OpenCL compute kernels. These are not part of the default build and require CUDA or OpenCL runtime libraries. Practical use is limited to specialised research scenarios. **Note: needs verification** — the GPU opcode API names and stability status should be confirmed against the current Csound repository's `Opcodes/` directory before use in production.

---

## 13. Surge XT Synthesizer: Open-Source Hybrid Synth

Surge XT is an open-source hybrid synthesizer combining virtual-analogue, wavetable, FM, and physical modelling synthesis in a single instrument. [Source](https://surge-synthesizer.github.io/) [Source](https://github.com/surge-synthesizer/surge)

### 13.1 Architecture and Oscillator Algorithms

Two **scenes** per patch each provide three oscillators. Twelve oscillator algorithms are available: Classic (Polyblep VA), Modern, Wavetable, Window, Sine, FM2, FM3, String (Karplus-Strong), Twist, Alias, S&H Noise, and Audio Input. Each scene has a filter section (24 filter types), two amplifier stages, and twelve LFOs — six per-voice (rate locks to voice pitch) and six global — including MSEG (multi-segment envelope generator) and Lua formula modulators.

The modulation matrix is near-universal: almost every continuous parameter can be a modulation destination. Sources include LFOs, envelopes, MIDI CCs, pitch bend, aftertouch, and macro controls.

### 13.2 Plugin Format Builds

Surge XT builds on JUCE 7 and supports all major Linux plugin formats via CMake:

```bash
cmake -B build -DSURGE_BUILD_LV2=TRUE -DSURGE_BUILD_VST3=TRUE \
               -DSURGE_BUILD_CLAP=TRUE -DSURGE_BUILD_STANDALONE=TRUE
cmake --build build --parallel
```

The LV2 build (`-DSURGE_BUILD_LV2=TRUE`) requires JUCE 7's LV2 support. CLAP and VST3 outputs are also generated. The standalone build uses JUCE's built-in ALSA or JACK audio backend on Linux.

---

## 14. CLAP Plugin Standard

CLAP (CLever Audio Plug-in) is an MIT-licensed open plugin standard co-developed by u-he and Bitwig, version 1.2.6 (March 11, 2025). [Source](https://github.com/free-audio/clap)

### 14.1 Entry Point and Extension System

A CLAP plugin is a shared library exporting `clap_entry` (a `clap_plugin_entry_t`):

```c
/* From: github.com/free-audio/clap */
#include <clap/clap.h>

/* Entry point exported from .so */
extern const clap_plugin_entry_t clap_entry;

/* Factory pattern: host discovers available plugins */
/* clap_plugin_factory_t: get_plugin_count, get_plugin_descriptor, create_plugin */

/* Extension query from plugin instance: */
const clap_plugin_params_t *params =
    (const clap_plugin_params_t *)plugin->get_extension(plugin, CLAP_EXT_PARAMS);
const clap_plugin_note_ports_t *note_ports =
    (const clap_plugin_note_ports_t *)plugin->get_extension(plugin, CLAP_EXT_NOTE_PORTS);
const clap_plugin_voice_info_t *voice_info =
    (const clap_plugin_voice_info_t *)plugin->get_extension(plugin, CLAP_EXT_VOICE_INFO);
```

### 14.2 Key Features and Adoption

CLAP's core design properties:

- **Thread-safe parameter automation**: the `params` extension defines a thread-safe protocol for host↔plugin parameter updates, avoiding the race conditions common in VST2.
- **Polyphonic per-voice modulation**: the `note-ports` extension carries note events with per-voice parameter modulation attached, enabling Bitwig Studio's MPE-style modulation without hacks.
- **Voice management**: `CLAP_EXT_VOICE_INFO` lets the host know how many voices are active, enabling CPU-aware scheduling.
- **ABI stability**: CLAP 1.x plugins work under any CLAP 1.y host without recompilation.

Adopted synthesizers and hosts: u-he (Diva, Hive, Repro), Bitwig Studio, Surge XT, Reaper, Carla. yabridge 5.x supports CLAP bridging from Windows to Linux. The `clap-helpers` C++ header-only library provides convenience wrappers for plugin authors.

---

## 15. Sequencing and Notation: MuseScore, LilyPond, and MIDI Pipelines

### 15.1 MuseScore 4

MuseScore 4's audio engine defines an `msynth` abstract synthesis interface above **FluidSynth** (SF2 rendering) and **Aeolus** (additive pipe organ synthesis). The core notation data model is **libmscore**, compiled to WebAssembly for browser-based score rendering at musescore.com. [Source](https://musescore.org/en/handbook/developers-handbook/musescore-4-developer-resources)

MIDI import/export is a primary workflow: MuseScore reads SMF `.mid` files with quantisation to its BBT grid, and exports rendered audio by routing libmscore's MIDI events through the FluidSynth `msynth` backend into an audio file via libsndfile.

**MusicXML** is the primary notation interchange format — MuseScore imports and exports MusicXML; Sibelius, Finale, Dorico, and Lilypond all read it. MuseScore does not export LilyPond directly; MusicXML is the bridge. The score font used by both MuseScore and LilyPond is **Emmentaler**, sharing a common visual vocabulary.

### 15.2 LilyPond

LilyPond is a text-based notation language that compiles to print-quality PDF/SVG scores. [Source](https://lilypond.org/) A complete pipeline from MIDI to engraved notation to rendered audio:

```bash
# 1. Convert SMF MIDI to MusicXML
musicxml2ly --output=piece.ly piece.musicxml   # via musicxml2ly (ships with LilyPond)
# 2. Engrave to PDF
lilypond piece.ly
# 3. Re-export to MIDI from LilyPond output, then render to audio
aplaymidi -p 128:0 piece.midi   # using ALSA seq → FluidSynth on port 128:0
```

MIDI import to LilyPond is via `midi2ly` (approximate; rhythm quantisation is lossy) or the MusicXML route.

### 15.3 MIDI Export and Fluid+Ardour Rendering Pipeline

A complete "score to audio" pipeline integrating the tools covered in this chapter:

```bash
# 1. Compose in MuseScore 4, export to SMF MIDI
musescore4 -o composition.mid composition.mscz

# 2. Start FluidSynth as a MIDI-to-audio renderer on ALSA seq port 128:0
fluidsynth -a pipewire -m alsa_seq -g 1.0 \
           /usr/share/sounds/sf2/FluidR3_GM.sf2 &

# 3. Play the MIDI file, captured by PipeWire to a WAV
pw-record --channels 2 --rate 48000 output.wav &
aplaymidi -p 128:0 composition.mid

# 4. Alternatively: import .mid into Ardour 8, assign LV2 instrument tracks,
#    render via Session > Export > Stem Export with libsndfile FLAC backend
```

For film/game scoring workflows, Ardour's BBT grid and MIDI pipeline (§10) replaces `aplaymidi`, giving per-track plugin mixing, MIDI editing, automation, and mastered export. MuseScore's internal FluidSynth backend (`msynth`) provides "quick listen" audio without exporting, while Ardour provides the production-quality render path.

---

## 16. Real-Time Safety: Lock-Free Buffers and RT Scheduling

Audio callbacks run at `SCHED_FIFO` priority with hard deadlines. For a buffer of 256 frames at 48 kHz, the deadline is 256/48000 ≈ 5.3 ms. Any blocking in the callback causes a **xrun** (underrun) — an audible glitch. [Source](https://lwn.net/Articles/339316/)

### 16.1 Forbidden Operations in the Audio Thread

| Forbidden | Reason |
|---|---|
| `malloc` / `free` | May invoke the kernel allocator, which can block |
| File I/O (`open`, `read`, `write`) | Kernel may block on disk or page fault |
| Mutex lock with contention | Priority inversion: low-priority thread holds lock |
| `printf` / `syslog` | May acquire internal libc locks |
| Any blocking syscall | Violates the RT deadline |

### 16.2 Lock-Free Data Structure Options

Two lock-free ring buffer implementations are widely used in Linux audio:

**jack_ringbuffer** (from `libjack-jackd2-dev`) is a battle-tested single-producer/single-consumer queue. It uses memory barriers but no locks, making it safe to use from two threads simultaneously without synchronisation. [Source](https://jackaudio.org/api/ringbuffer_8h.html)

**LCRQ** is a multi-producer/multi-consumer lock-free queue that admits more than two threads, using CAS (compare-and-swap) operations. It is suitable for routing events between multiple audio threads in a parallel DSP engine such as supernova.

For most plugin and DAW use cases, `jack_ringbuffer` is the right choice: it is simpler, has lower overhead, and the single-producer/single-consumer constraint matches the GUI thread ↔ audio thread communication model exactly.

### 16.3 jack_ringbuffer: Lock-Free Communication

`jack_ringbuffer` provides a single-producer / single-consumer lock-free ring buffer, safe for exactly two threads without any synchronisation primitives: [Source](https://jackaudio.org/api/ringbuffer_8h.html)

```c
/* jack/ringbuffer.h */
jack_ringbuffer_t *rb = jack_ringbuffer_create(4096);

/* In the non-RT GUI/control thread (producer): */
if (jack_ringbuffer_write_space(rb) >= sizeof(midi_ev))
    jack_ringbuffer_write(rb, (const char *)&midi_ev, sizeof(midi_ev));

/* In the RT audio callback (consumer): */
while (jack_ringbuffer_read_space(rb) >= sizeof(midi_ev)) {
    jack_ringbuffer_read(rb, (char *)&ev, sizeof(midi_ev));
    process_midi_event(&ev);
}
/* No locks. Safe because exactly one reader and one writer thread. */
```

### 16.4 RTKit, RLIMIT_RTTIME, and Portal Realtime

**RTKit** (RealtimeKit) is a D-Bus system service at `org.freedesktop.RealtimeKit1` that grants `SCHED_FIFO` scheduling to requesting audio processes without requiring `CAP_SYS_NICE`. PipeWire connects to RTKit at startup to elevate its process thread priority. RTKit's watchdog demotes any RT thread that consumes excessive CPU, preventing system stalls.

**RLIMIT_RTTIME** is a kernel-enforced limit (in microseconds) on the total CPU time a `SCHED_FIFO` thread may accumulate without blocking. It prevents a runaway audio plugin from monopolising a core. Set via `setrlimit(RLIMIT_RTTIME, ...)` or via PAM limits.

PipeWire 2026 also supports **Portal Realtime** (the XDG portal `org.freedesktop.portal.Realtime`) as an alternative to RTKit, removing the D-Bus dependency for sandboxed applications such as Flatpaks. [Source](https://www.linuxdj.com/notes/pipewire-real-time-scheduling-in-2026-rtkit-portal-and-rlimits/)

### 16.5 Memory Pre-Allocation Strategy

Audio plugins and servers pre-allocate all working memory at initialisation:

```c
/* Allocate voice pool at plugin instantiate() — never in run() */
self->voices = calloc(MAX_VOICES, sizeof(Voice));

/* Allocate audio output buffer at connect_port() time */
/* LV2 host provides the buffer; plugin writes into it directly */
```

SuperCollider's RTAlloc, FluidSynth's voice allocator, and scsynth's node tree all follow this pattern. C11 `stdatomic` operations (`atomic_load`, `atomic_store`, `atomic_compare_exchange`) are safe in the audio thread when used on appropriately aligned types — they compile to hardware atomics without kernel involvement.

---

## Integrations

- **Ch38 — PipeWire and the Video Session Layer**: PipeWire MIDI graph internals, session manager (WirePlumber) MIDI policy, ALSA-seq bridge, pw-link routing, and the `spa_pod_sequence` buffer format underpinning §3 of this chapter.

- **Ch38b — ALSA: The Linux Audio Subsystem**: Kernel-level ALSA sequencer subsystem (`snd_seq` core, queue engine, timer), rawmidi character device interface, and MIDI 2.0 UMP kernel infrastructure referenced in §2 and §4.

- **Ch54 — The Linux Input Stack**: MIDI controller HID enumeration, USB HID-to-ALSA mapping, the `snd-usb-audio` driver's class-compliant MIDI handling, and the input event pipeline that delivers controller data to userspace before it reaches the ALSA sequencer client in §2 and §4.

- **Ch213 — MIDI over IP: RTP-MIDI and rtpmidid**: Extends the sequencer model from §2 across the network via RTP-MIDI (RFC 6295); rtpmidid bridges RTP-MIDI sessions to ALSA sequencer ports, making remote instruments appear as local ALSA clients.

- **Ch216 — Speech Synthesis**: Speech synthesis engines (eSpeak NG, Festival) may be integrated as LV2 plugins (§7–§8) or as Csound opcodes (§12), feeding synthesised phoneme audio into the same DAW signal chain described in §10.
