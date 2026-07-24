# Chapter 216: Speech Synthesis and ASR on Linux: espeak-ng, Piper, and Whisper

**Target audiences**: Application developers and accessibility engineers integrating text-to-speech or speech recognition into Linux applications; systems engineers deploying on-device speech pipelines on constrained or privacy-sensitive Linux hardware.

This chapter covers the complete on-device speech stack available on Linux as of mid-2026: the speech-dispatcher abstraction daemon, the espeak-ng formant synthesiser, the Piper VITS-based neural TTS engine, the legacy Festival/Flite lineage, whisper.cpp for offline automatic speech recognition (ASR), GPU acceleration options for inference, the Vosk/Kaldi-based streaming ASR library, low-latency PipeWire capture pipelines, accessibility stack integration through AT-SPI2, wake word detection, and guidance for choosing among the available options.

---

## Table of Contents

1. [speech-dispatcher: Unified TTS Abstraction](#speech-dispatcher-unified-tts-abstraction)
   - [1.1 What is Text-to-Speech (TTS) Synthesis?](#11-what-is-text-to-speech-tts-synthesis)
   - [1.2 What is Automatic Speech Recognition (ASR)?](#12-what-is-automatic-speech-recognition-asr)
   - [1.3 What is speech-dispatcher?](#13-what-is-speech-dispatcher)
2. [espeak-ng: Formant Synthesis Engine](#espeak-ng-formant-synthesis-engine)
3. [Piper TTS: Neural Synthesis with VITS and ONNX](#piper-tts-neural-synthesis-with-vits-and-onnx)
4. [Festival and Flite: Legacy TTS](#festival-and-flite-legacy-tts)
5. [Whisper.cpp: Offline ASR with GGML](#whispercpp-offline-asr-with-ggml)
6. [GPU Acceleration for ASR](#gpu-acceleration-for-asr)
7. [Vosk: Kaldi-Based Streaming ASR](#vosk-kaldi-based-streaming-asr)
8. [ALSA and PipeWire Capture Pipeline](#alsa-and-pipewire-capture-pipeline)
9. [Accessibility Stack Integration](#accessibility-stack-integration)
10. [Wake Word Detection](#wake-word-detection)
11. [Benchmarking](#benchmarking)
12. [On-Device vs. Cloud Trade-offs](#on-device-vs-cloud-trade-offs)
13. [Integrations](#integrations)

---

## speech-dispatcher: Unified TTS Abstraction

speech-dispatcher (`speechd`) is the canonical TTS broker on Linux desktop systems. It arbitrates requests from multiple concurrent clients — screen readers, shell scripts, application dialogs — routing each to the configured synthesis back-end, managing priority queues and audio output. [Source](https://github.com/brailcom/speechd)

### SSIP Protocol and Connection Model

Clients connect over the Speech Dispatcher Internet Protocol (SSIP), a line-oriented ASCII protocol carried over either a Unix domain socket at `$XDG_RUNTIME_DIR/speech-dispatcher/speechd.sock` or TCP port 6560. The `libspeechd` C library wraps this protocol; `SPDConnection` tracks the socket file descriptor, a threading mode (`SPD_MODE_SINGLE` or `SPD_MODE_THREADED`), a send/receive buffer, a mutex, and optional event callbacks.

SSIP connection flow:

1. Client connects, sends `SET CLIENT_NAME myapp:component:user_data`.
2. Client configures engine: `SET SELF OUTPUT_MODULE espeak-ng`, `SET SELF LANGUAGE en`, `SET SELF VOICE_TYPE male1`, `SET SELF RATE 20`, `SET SELF PITCH 0`, `SET SELF VOLUME 100`.
3. Client sends `SPEAK`, transmits text terminated with `\r\n.\r\n`, receives a numeric reply containing the message ID.
4. Control commands: `STOP` (stop current message for this connection), `CANCEL` (cancel all queued messages), `PAUSE`, `RESUME`.
5. SSML mode: `SET SELF SSML_MODE ON` before `SPEAK` causes the text to be treated as SSML and forwarded to SSML-capable back-ends.

Rate is `[-100, +100]` relative to the module default; pitch and volume follow the same convention. [Source](https://github.com/brailcom/speechd/blob/master/src/api/c/libspeechd.h)

### libspeechd Client API

The `libspeechd` C API hides the protocol details:

```c
/* libspeechd client API - connect and speak
 * Header: src/api/c/libspeechd.h
 */
#include <libspeechd.h>

SPDConnection *conn = spd_open("myapp", "component", NULL, SPD_MODE_SINGLE);
if (!conn) { fprintf(stderr, "Failed to open speech-dispatcher\n"); exit(1); }

spd_set_output_module(conn, "espeak-ng");
spd_set_language(conn, "en");
spd_set_voice_type(conn, SPD_MALE1);
spd_set_rate(conn, 20);   /* -100 to +100 */
spd_set_pitch(conn, 0);
spd_set_volume(conn, 100);

/* Priority levels: SPD_PROGRESS, SPD_MESSAGE, SPD_TEXT,
   SPD_NOTIFICATION, SPD_ALARM */
int msg_id = spd_say(conn, SPD_MESSAGE, "Hello from speech-dispatcher");
spd_say(conn, SPD_ALARM, "Alert: disk full");

/* Index marks for position tracking */
conn->callback_im = my_index_mark_callback;  /* SPDCallbackIM */
spd_say(conn, SPD_MESSAGE, "First part<mark name='mid'/> second part");

spd_stop(conn);   /* stop current message */
spd_cancel(conn); /* cancel all messages */
spd_close(conn);
```

[Source](https://github.com/brailcom/speechd/blob/master/src/api/c/libspeechd.c)

### Module Architecture

Each back-end is a separate executable launched by speechd. Modules communicate with speechd over bidirectional pipes: a `TModuleDoublePipe` struct holds `pc[2]` (parent-to-child) and `cp[2]` (child-to-parent) pipe arrays. Modules receive SSIP sub-commands and return PCM audio samples or event notifications. [Source](https://github.com/brailcom/speechd/blob/master/src/server/output.h)

Supported modules include: `espeak-ng`, `pico` (SVOX Pico), `flite`, `festival`, `piper`, `mary`, `espeak-ng-mbrola`, and `generic` (a shell-command wrapper for arbitrary engines). The module infrastructure (`module_utils.h`) provides `module_strip_ssml()` to remove markup before sending text to non-SSML-aware back-ends, `module_strip_silence()`, `module_tts_output()` for PCM forwarding, and index-mark support using the `__spd_` prefix convention. [Source](https://github.com/brailcom/speechd/blob/master/src/modules/module_utils.h)

### Priority Queue Scheduling

Messages arrive with a priority level that determines scheduling. From lowest to highest: `PROGRESS`, `TEXT`, `MESSAGE`, `NOTIFICATION`, `ALARM`. A higher-priority message interrupts lower-priority synthesis. The daemon maintains per-client queues and a global scheduling loop that selects the highest-priority pending message across all clients.

### 1.1 What is Text-to-Speech (TTS) Synthesis?

Text-to-speech (TTS) synthesis is the computational process of converting written text into audible speech signals. On Linux, TTS drives accessibility tools such as screen readers, interactive voice response systems, embedded navigation devices, and application feedback mechanisms. The fundamental task is mapping the discrete symbols of written language — graphemes — through intermediate phonetic representations to continuous acoustic waveforms that a listener recognises as speech.

Two broad synthesis strategies appear in the Linux ecosystem. Formant synthesis, exemplified by espeak-ng, models the vocal tract as a filter bank driven by a periodic excitation source; it requires no recorded speech data, can produce voice in any language given phoneme rules and pronunciation dictionaries, and runs on minimal hardware. Neural synthesis, exemplified by Piper and its VITS architecture, uses deep neural networks trained on many hours of recorded speech; it produces substantially more natural output at the cost of higher compute requirements during inference. Both strategies produce PCM audio at a fixed sample rate — typically 16 kHz or 22.05 kHz — that feeds directly into ALSA or PipeWire for playback.

Linux applications do not invoke synthesis engines directly in a well-structured deployment. Instead they route requests through speech-dispatcher, which decouples client priority, language selection, and engine selection from application code, allowing system-wide policies to determine which synthesis back-end is active. The full TTS stack covered in this chapter spans this abstraction daemon, the two major open-source synthesis engines (espeak-ng and Piper), and the legacy Festival and Flite lineage.

### 1.2 What is Automatic Speech Recognition (ASR)?

Automatic speech recognition (ASR) converts continuous acoustic input — captured from a microphone as a PCM sample stream — into text transcripts. Modern ASR systems are neural sequence-to-sequence models trained on large corpora of labelled audio. The dominant paradigm on Linux as of mid-2026 is the Whisper encoder-decoder architecture and its efficient C++ reimplementation, whisper.cpp, which quantises model weights to run on CPU or GPU without a Python runtime. A complementary streaming-capable approach is provided by Vosk, which wraps a Kaldi-derived acoustic model and graph decoder with a compact runtime suitable for embedded and server deployments.

On Linux, ASR involves three pipeline stages: audio capture through ALSA or PipeWire, which delivers PCM frames into a ring buffer; feature extraction, typically log-mel spectrograms computed over 25 ms windows at a 10 ms stride; and model inference on CPU or GPU. GPU acceleration through CUDA, ROCm, or Vulkan compute can reduce inference latency from several seconds per utterance on a CPU to near real-time. The ASR subsystem produces plain-text transcripts or word-timed JSON output that downstream applications — voice command handlers, captioning tools, dictation frontends — consume directly.

Unlike TTS, Linux ASR does not route through speech-dispatcher. Each ASR engine exposes its own library API: `whisper_full()` and associated context functions for whisper.cpp, and `vosk_recognizer_accept_waveform()` with its JSON result interface for Vosk. Chapter 8 covers the PipeWire capture pipeline that feeds both engines.

### 1.3 What is speech-dispatcher?

speech-dispatcher (daemon binary `speechd`, package `speech-dispatcher`) is the standard TTS broker on Linux desktop and embedded systems. It provides a single client-facing abstraction over multiple synthesis back-ends, allowing applications, screen readers, and shell scripts to request speech output without knowing which engine is installed, where it is located, or how audio routing is configured.

The daemon accepts connections over the Speech Dispatcher Internet Protocol (SSIP), a line-oriented ASCII protocol carried on a Unix domain socket at `$XDG_RUNTIME_DIR/speech-dispatcher/speechd.sock` or on TCP port 6560. Clients authenticate, set parameters — output module, language, voice type, rate, pitch, volume — then submit text or SSML for synthesis using the `SPEAK` command. The daemon maintains per-client queues with five priority levels (`PROGRESS`, `TEXT`, `MESSAGE`, `NOTIFICATION`, `ALARM`) and a global scheduling loop that interrupts lower-priority synthesis when a higher-priority message arrives.

Each synthesis back-end runs as a separate executable module communicating with `speechd` over bidirectional pipes. Supported modules include `espeak-ng`, `piper`, `flite`, `festival`, and `generic` (a shell-command wrapper). The `libspeechd` C library exposes this to applications through functions such as `spd_open()`, `spd_say()`, `spd_stop()`, and `spd_close()`, hiding SSIP details entirely. speech-dispatcher is the standard integration point for AT-SPI2-based screen readers such as Orca and for desktop notification systems. Applications targeting accessibility on Linux should use the `libspeechd` API rather than calling synthesis engines directly.

---

## espeak-ng: Formant Synthesis Engine

espeak-ng is the most widely deployed open-source TTS engine on Linux, covering roughly 100 languages with a compact data footprint (typically 30–50 MB total). It uses formant synthesis rather than recorded speech, which produces a characteristic robotic tone but enables voice synthesis on minimal hardware. [Source](https://github.com/espeak-ng/espeak-ng)

### Synthesis Pipeline

Text passes through a multi-stage pipeline, all implemented in `src/libespeak-ng/`:

1. **readclause.c** — Sentence and clause segmentation; SSML `<speak>`, `<break>`, `<phoneme>`, and `<say-as>` tag parsing.
2. **translate.c** — Dictionary lookup in compiled `.dict` binary files (produced by `espeak_ng_CompileDictionary()`), then phoneme rule application for out-of-vocabulary words.
3. **MakePhonemeList()** in `synthesize.c` — Prosody and intonation assignment; stress rules; tone for tonal languages.
4. **synthesize.c** — Generates wave commands: F0 contour values and phoneme timing in milliseconds.
5. **wavegen.c** — Formant synthesis: models the vocal tract as a resonating filter bank generating samples at 22050 Hz. Alternatively, **klatt.c** (Klatt 1980 cascade-parallel synthesiser) or **synth_mbrola.c** (MBROLA diphone path) may be selected by voice configuration.

Phoneme tables are compiled into `espeak-ng-data/phontab` binary from source files in `phsource/`. Language voices live in `espeak-ng-data/voices/`. [Source](https://deepwiki.com/espeak-ng/espeak-ng)

### Language Coverage

espeak-ng ships phoneme databases and pronunciation rules for approximately 100 languages, organized by language family (Germanic, Romance, Slavic, Semitic, East Asian, etc.). Each language has: base phoneme definitions, language-specific adaptations, pronunciation rule files (`.rules`), and compiled pronunciation dictionaries (`.dict` binary files, compiled from `_rules` and `_list` source files via `espeak_ng_CompileDictionary()`).

### Modern libespeak-ng API

espeak-ng provides two API layers. The legacy `speak_lib.h` header (API revision 12, functions `espeak_Initialize`, `espeak_Synth`, `espeak_SetParameter`) remains functional. The modern `espeak_ng.h` is preferred for new code, with typed status codes (`espeak_ng_STATUS`) including `ENS_OK`, `ENS_NOT_INITIALIZED`, `ENS_VOICE_NOT_FOUND`, `ENS_MBROLA_NOT_FOUND`, `ENS_AUDIO_ERROR`, and others.

```c
/* espeak-ng modern API initialization and synthesis
 * Header: src/include/espeak-ng/espeak_ng.h
 */
#include <espeak-ng/espeak_ng.h>
#include <espeak-ng/speak_lib.h>

espeak_ng_ERROR_CONTEXT context = NULL;
espeak_ng_InitializePath(NULL);  /* use default data path */
espeak_ng_STATUS status = espeak_ng_Initialize(&context);
if (status != ENS_OK) {
    espeak_ng_PrintStatusCodeMessage(status, stderr, context);
    return 1;
}
espeak_ng_InitializeOutput(ENOUTPUT_MODE_SPEAK_AUDIO, 0, NULL);

espeak_ng_SetVoiceByName("en");
espeak_ng_SetParameter(espeakRATE, 175, 0);    /* 80-450 wpm, default 175 */
espeak_ng_SetParameter(espeakPITCH, 50, 0);   /* 50% of range */
espeak_ng_SetParameter(espeakVOLUME, 100, 0);

unsigned int uid;
espeak_ng_Synthesize("<speak>Hello <break time='500ms'/> world</speak>",
                     strlen(text), 0, POS_CHARACTER, 0,
                     espeakSSML | espeakCHARS_UTF8, &uid, NULL);
espeak_ng_Synchronize();
espeak_ng_Terminate();
```

[Source](https://github.com/espeak-ng/espeak-ng/blob/master/src/include/espeak-ng/espeak_ng.h)

### Phoneme Event Callbacks

Applications that need word-level or phoneme-level timing — for karaoke highlighting, lip-sync animation, or accessibility feedback — install a synthesis callback via the legacy API:

```c
/* espeak-ng phoneme event callback for word/phoneme timing
 * Header: src/include/espeak-ng/speak_lib.h
 */
static int synth_callback(short *wav, int numsamples,
                           espeak_EVENT *events) {
    while (events->type != espeakEVENT_LIST_TERMINATED) {
        switch (events->type) {
        case espeakEVENT_WORD:
            printf("Word at sample %d, text pos %d, length %d\n",
                   events->audio_position,
                   events->text_position,
                   events->length);
            break;
        case espeakEVENT_PHONEME:
            printf("Phoneme: %s\n", events->id.string);
            break;
        case espeakEVENT_MARK:
            printf("Index mark: %s\n", events->id.name);
            break;
        case espeakEVENT_END:
            printf("End of message %d\n", events->unique_identifier);
            break;
        }
        events++;
    }
    if (wav) process_audio(wav, numsamples);
    return 0;  /* non-zero aborts synthesis */
}

espeak_SetSynthCallback(synth_callback);
espeak_Initialize(AUDIO_OUTPUT_RETRIEVAL, 2000, NULL, 0);
```

[Source](https://github.com/espeak-ng/espeak-ng/blob/master/src/libespeak-ng/espeak_api.c)

### MBROLA Integration

espeak-ng can serve as a text-to-phoneme front-end for the MBROLA diphone synthesiser, which produces noticeably more natural speech by using recorded diphone units. The integration works by having espeak-ng generate MBROLA `.pho` phoneme files that are piped to the `mbrola` binary.

MBROLA voices are named `mb-xxN` where `xxN` is the MBROLA voice identifier (e.g., `mb-en1`, `mb-de4`). Phoneme translation files in `phsource/mbrola/xxN` map espeak-ng phonemes to MBROLA equivalents with duration-splitting percentages. Cross-accent voices such as `mb-de4-en` speak English text with a German accent.

```bash
# Generate MBROLA phoneme output for en1 voice
espeak-ng -v mb-en1 -q --pho 'Hello world'
# Output lines: <phoneme> <duration_ms> [pitch_targets...]
# e.g.:  h  80  0 95  50 120
#        E  120 0 110 50 115
#        l  60
#        @U 150 0 100 50 90
```

[Source](https://github.com/espeak-ng/espeak-ng/blob/master/docs/mbrola.md)

---

## Piper TTS: Neural Synthesis with VITS and ONNX

Piper is a neural TTS engine that uses the VITS (Variational Inference with Adversarial Learning for End-to-End Text-to-Speech) acoustic model architecture. Maintained as `OHF-Voice/piper1-gpl` under the Open Home Foundation (v1.4.2, April 2026), Piper produces substantially more natural-sounding speech than espeak-ng while remaining capable of CPU-only inference. [Source](https://github.com/OHF-Voice/piper1-gpl)

### Voice File Format

Each Piper voice consists of exactly two files: a `.onnx` model file containing both the VITS acoustic model and the baked-in HiFi-GAN vocoder, and a `.onnx.json` configuration file specifying phoneme mappings and inference parameters.

```json
{
  "audio": {
    "sample_rate": 22050,
    "quality": "medium"
  },
  "espeak": {
    "voice": "en-us"
  },
  "inference": {
    "noise_scale": 0.667,
    "length_scale": 1.0,
    "noise_w": 0.8
  },
  "phoneme_type": "espeak",
  "piper_version": "1.0.0",
  "num_symbols": 256,
  "num_speakers": 1,
  "phoneme_id_map": {
    "_": [0], "^": [1], "$": [2], " ": [3],
    "!": [4], "p": [46], "b": [47]
  }
}
```

Key inference parameters: `length_scale` (default 1.0; values below 1.0 accelerate speech, above 1.0 slow it down); `noise_scale` (0.667; controls prosody variation); `noise_w` (0.8; controls phoneme duration variation). [Source](https://github.com/OHF-Voice/piper1-gpl/blob/main/src/piper/config.py)

### Inference Pipeline

Text → espeak-ng phonemizer (language determined by `espeak.voice`) → IPA phoneme sequence → `phoneme_id_map` lookup → padded integer tensor → ONNX Runtime `session.run()` with inputs `phoneme_ids` (int64 `[1, T]`), `input_lengths` (int64 `[1]`), `scales` (float32 `[3]`: noise_scale, length_scale, noise_w_scale), optional `speaker_id` (int64 `[1]`) → VITS decoder produces mel-spectrogram → HiFi-GAN vocoder → raw float32 audio at `sample_rate` Hz → convert to int16 for output. The `hop_length` (default 256) determines the mel frame stride. [Source](https://github.com/OHF-Voice/piper1-gpl/blob/main/src/piper/voice.py)

### Python Streaming API

```python
# Piper TTS Python API - streaming synthesis
# File: src/piper/voice.py
from piper import PiperVoice
from piper.config import SynthesisConfig
import pyaudio

voice = PiperVoice.load("/models/en_US-lessac-medium.onnx")
# For GPU inference: PiperVoice.load(model_path, use_cuda=True)

print(f"Sample rate: {voice.config.sample_rate}")
print(f"eSpeak voice: {voice.config.espeak_voice}")

synth_config = SynthesisConfig(
    length_scale=1.0,   # < 1.0 faster, > 1.0 slower
    noise_scale=0.667,
    noise_w_scale=0.8,
    volume=1.0,
    normalize_audio=True,
)

# Streaming synthesis - yields AudioChunk per sentence
pa = pyaudio.PyAudio()
stream = pa.open(format=pyaudio.paInt16, channels=1,
                 rate=voice.config.sample_rate, output=True)
for chunk in voice.synthesize("Hello world. This is sentence two.",
                               synth_config=synth_config):
    # AudioChunk has: audio_float_array, audio_int16_bytes,
    # sample_rate, sample_width, channels
    stream.write(chunk.audio_int16_bytes)
stream.stop_stream(); stream.close()
```

[Source](https://github.com/OHF-Voice/piper1-gpl/blob/main/docs/API_PYTHON.md)

`voice.synthesize()` yields one `AudioChunk` per sentence, enabling the calling code to begin playback while subsequent sentences are still being synthesised — relevant for long text where latency to first audio matters.

### Voice Training

Custom voices are trained with PyTorch Lightning using the VITS architecture. The training dataset is a pipe-delimited CSV: `audio_filename|text|optional_speaker_id`. Preprocessing includes espeak-ng phonemization and audio caching in `--data.cache_dir`. Only `medium`-quality checkpoints are recommended for transfer learning without adjusting other hyperparameters. The vocoder warm-start option (`--model.vocoder_warmstart_ckpt`) copies vocoder parameters from a pretrained checkpoint while preserving phoneme embedding flexibility. [Source](https://github.com/OHF-Voice/piper1-gpl/blob/main/docs/TRAINING.md)

ONNX Runtime provider options: CPU (default) or CUDA (`use_cuda=True`, requires `onnxruntime-gpu`). The package does not currently expose a Vulkan compute provider; CUDA is the only GPU path.

---

## Festival and Flite: Legacy TTS

Festival is a general-purpose speech synthesis framework developed at the Centre for Speech Technology Research (CSTR), Edinburgh. It operates as a server accepting Scheme expressions; clients connect via socket and issue `(SayText "hello world")`. Festival supports diphone voices, HMM-based statistical parametric voices, and MBROLA integration. [Source](https://github.com/festvox/festvox)

Flite (Festival Lite) is a compact, standalone fork optimised for embedded use. Flite 4 ships four base voices: `awb`, `rms`, `slt` (clustergen statistical parametric), and `kal` (diphone database). CMU ARCTIC voices are studio-recorded at 16 kHz with approximately one hour of material per speaker. These voices are included in the `flite` package in most distributions. [Source](https://github.com/festvox/flite)

### Quality Comparison

The quality gap between Festival/Flite and neural TTS is substantial. VITS-based systems trained on 24-hour LJ Speech or similar single-speaker datasets achieve mean opinion scores (MOS) in the 4.0–4.4 range, approaching natural speech. Festival diphone voices score roughly 2.8–3.2. Flite scores lower still due to limited training data and the diphone concatenation artefacts. Both Festival and Flite remain relevant for constrained deployments where model size is critical: Flite with `kal` voice is under 3 MB, whereas even a Piper `low`-quality model is approximately 30 MB.

The Festvox 3.0 roadmap targets VITS and diffusion-based neural extensions, but as of 2026 the flagship on-Linux neural TTS recommendation is Piper rather than Festival-derived systems.

---

## Whisper.cpp: Offline ASR with GGML

whisper.cpp is a C/C++ reimplementation of OpenAI Whisper using the GGML tensor library. It supports the full Whisper model series (tiny through large-v3) in multiple quantised formats and exposes a C API (`include/whisper.h`) for embedding. [Source](https://github.com/ggml-org/whisper.cpp)

### Audio Parameters and Model Architecture

Fixed audio constants (from `whisper.h`):

- `WHISPER_SAMPLE_RATE` = 16000 Hz
- `WHISPER_N_FFT` = 400 samples
- `WHISPER_HOP_LENGTH` = 160 samples (10 ms hop)
- `WHISPER_CHUNK_SIZE` = 30 seconds

Each audio chunk is processed as a 80-bin log-mel spectrogram. The Whisper transformer uses 12 or 24 encoder layers and 12 or 24 decoder layers depending on model size. [Source](https://snailtext.app/blog/how-whisper-cpp-works/)

### Quantised Model Formats

| Format | Bits/weight | Notes |
|--------|-------------|-------|
| F16    | 16          | Full half-precision baseline |
| Q8_0   | ~8.5        | Minimal quality loss |
| Q5_1   | ~5.5        | Good quality/size balance |
| Q5_0   | ~5.0        | Slightly lower quality |
| Q4_0   | ~4.5        | Smallest, modest WER increase |

Memory footprints (F16): tiny=273 MB, base=388 MB, small=852 MB, medium=2.1 GB, large-v3=3.9 GB. Q5_0 large-v3 reduces this to approximately 1.1 GB. WER impact of quantisation is small: Whisper small F16 achieves 3.4% WER on LibriSpeech test-clean; Q5_1 reaches 3.5–3.7% — a negligible difference in practice. Large-v3 WER: 2.7% on test-clean, 5.2% on test-other. [Source](https://github.com/ggml-org/whisper.cpp/blob/master/README.md)

### C API

```c
/* whisper.cpp C API - full inference pipeline
 * Header: include/whisper.h
 */
#include "whisper.h"

/* Load quantized model - use_gpu enables CUDA/Vulkan/Metal */
struct whisper_context_params cparams = whisper_context_default_params();
cparams.use_gpu = true;
struct whisper_context *ctx =
    whisper_init_from_file_with_params("ggml-large-v3-q5_0.bin", cparams);

struct whisper_full_params params =
    whisper_full_default_params(WHISPER_SAMPLING_GREEDY);
params.language     = "en";
params.translate    = false;
params.n_threads    = 4;
params.print_realtime  = true;
params.print_timestamps = true;

/* Optional segment callback for streaming output */
params.new_segment_callback = my_segment_callback;
params.new_segment_callback_user_data = user_data;

/* pcm_f32: float32 audio at 16 kHz */
if (whisper_full(ctx, params, pcm_f32, n_samples) != 0) {
    fprintf(stderr, "Inference failed\n");
}

const int n_segments = whisper_full_n_segments(ctx);
for (int i = 0; i < n_segments; i++) {
    const char *text = whisper_full_get_segment_text(ctx, i);
    int64_t t0 = whisper_full_get_segment_t0(ctx, i); /* centiseconds */
    int64_t t1 = whisper_full_get_segment_t1(ctx, i);
    printf("[%ld --> %ld ms] %s\n", t0 * 10, t1 * 10, text);
}

/* Per-token confidence and DTW timestamps */
const int n_tokens = whisper_full_n_tokens(ctx, 0);
for (int j = 0; j < n_tokens; j++) {
    struct whisper_token_data td = whisper_full_get_token_data(ctx, 0, j);
    printf("token %d: p=%.3f t0=%ld\n", td.id, td.p, td.t0);
}
whisper_free(ctx);
```

[Source](https://github.com/ggml-org/whisper.cpp/blob/master/include/whisper.h)

### Streaming Mode

The `examples/stream/stream.cpp` example implements always-on transcription. An `audio_async` class abstracts PipeWire/ALSA capture at `WHISPER_SAMPLE_RATE`. Two operating modes:

- **Fixed-window (non-VAD)**: processes fixed-size windows with `n_samples_keep` overlap, so context carries across chunk boundaries.
- **VAD mode**: `vad_simple()` triggers on energy above `vad_thold=0.6` after a high-pass filter at `freq_thold=100.0 Hz`. Results are emitted immediately: `printf("%s", text); fflush(stdout);`.

The last complete segment's tokens are added as a prompt for the next chunk, preserving linguistic context across the chunk boundary. [Source](https://github.com/ggml-org/whisper.cpp/blob/master/examples/stream/stream.cpp)

---

## GPU Acceleration for ASR

### Build Configuration

whisper.cpp supports four GPU back-ends selectable at CMake configure time:

```bash
# CUDA (NVIDIA)
cmake -B build -DGGML_CUDA=1 && cmake --build build -j

# Vulkan (cross-vendor: NVIDIA, AMD, Intel Arc, Intel iGPU, ARM Mali)
cmake -B build -DGGML_VULKAN=1 && cmake --build build -j

# OpenCL (legacy AMD/Intel path via clblast)
cmake -B build -DGGML_OPENCL=1 && cmake --build build -j

# Metal (macOS only - not applicable to Linux)
```

[Source](https://github.com/ggml-org/whisper.cpp/blob/master/README.md)

### Vulkan Compute Backend

The Vulkan back-end (`ggml/src/ggml-vulkan/`) was added in 2024 and matured through 2025–2026. It manages Vulkan instance, device selection, command buffer pools, and pipeline caches. Matrix multiplications — the dominant operation in transformer attention — are dispatched via `vkCmdDispatch` with workgroup dimensions derived from specialisation constants.

The matrix multiply shader (`ggml/src/ggml-vulkan/vulkan-shaders/mul_mm.comp`) uses a tiled algorithm with compile-time specialisation:

```glsl
/* whisper.cpp Vulkan matmul shader (excerpt)
 * File: ggml/src/ggml-vulkan/vulkan-shaders/mul_mm.comp
 * Specialisation constants define tile dimensions at pipeline compile time
 */
layout(local_size_x_id = 0, local_size_y = 1, local_size_z = 1) in;

// Block tile: BM x BN (default 64x64)
layout(constant_id = 1) const int BM = 64;
layout(constant_id = 2) const int BN = 64;
// Warp tile: WM x WN (default 32x32)
layout(constant_id = 4) const int WM = 32;
layout(constant_id = 5) const int WN = 32;
// Thread tile: TM x TN (default 4x2)
layout(constant_id = 7) const int TM = 4;
layout(constant_id = 8) const int TN = 2;
// BK (K-dimension tile): 32 for F32/F16, 16 for quantized types
```

Two shader paths exist: `mul_mm.comp` (standard tiled multiply) and `mul_mm_cm2.comp` (NVIDIA cooperative matrix 2 via `GL_NV_cooperative_matrix2`, higher throughput on Turing+). On AMD and Intel hardware, the standard `GL_KHR_cooperative_matrix` path is used when available. Q4_0 dequantisation (unpacking 4-bit nibbles with per-block delta scale) is performed within the shader, avoiding a separate dequant pass. [Source](https://deepwiki.com/ggml-org/whisper.cpp/2.4.1-vulkan-backend)

A flash attention shader (`flash_attn.comp`) fuses the QK^T matmul and softmax operations for reduced memory bandwidth at long context lengths.

### Performance Benchmarks

| Platform | Backend | RTF (large-v3) | Notes |
|----------|---------|----------------|-------|
| Intel i9-12900K | CPU | ~2.5 | 2.5s decode per 1s audio |
| RTX 4070 | CUDA | ~0.12 | 8x real-time |
| Ryzen 7 6800H (Radeon 680M iGPU) | Vulkan | ~0.7 | 3–4x faster than CPU |
| AMD RX 7900 | Vulkan | ~0.15 | near-CUDA parity |

RTF = real-time factor: values below 1.0 mean faster than real-time. whisper.cpp 1.8.3 delivered a 12× speedup for integrated graphics via improved Vulkan dispatch. [Source](https://www.phoronix.com/news/Whisper-cpp-1.8.3-12x-Perf)

---

## Vosk: Kaldi-Based Streaming ASR

Vosk provides offline streaming ASR built on the Kaldi WFST decoder. Unlike whisper.cpp, Vosk produces incremental partial results and supports grammar-constrained recognition, making it well suited for command-and-control applications. [Source](https://github.com/alphacep/vosk-api)

### Architecture

Vosk wraps Kaldi's `online2` decoder: `OnlineIvectorExtractorAdaptationState` for speaker adaptation, `LatticeFasterDecoder` for WFST lattice search. `VoskModel` loads an acoustic model (TDNN-F architecture), a language model decoding graph (`HCLG.fst`), an i-vector extractor, and CMVN normalisation statistics. `VoskModel` is reference-counted and safe to share across threads; each thread requires its own `VoskRecognizer`.

Audio enters as PCM 16-bit mono at the sample rate specified to `vosk_recognizer_new()`. Results are JSON: `vosk_recognizer_result()` for complete utterances, `vosk_recognizer_partial_result()` for incremental output, `vosk_recognizer_final_result()` to flush on end-of-stream.

### C API

```c
/* vosk-api C usage - streaming recognition
 * Header: src/vosk_api.h
 */
#include <vosk_api.h>

vosk_set_log_level(-1); /* suppress Kaldi stdout */
VoskModel *model = vosk_model_new("/path/to/vosk-model-small-en-us-0.15");

VoskRecognizer *rec = vosk_recognizer_new(model, 16000.0f);
vosk_recognizer_set_words(rec, 1);         /* include word timing */
vosk_recognizer_set_partial_words(rec, 1); /* enable partial results */

/* Grammar-constrained recognition for command vocabulary */
VoskRecognizer *cmd_rec = vosk_recognizer_new_grm(
    model, 16000.0f,
    "[\"turn left\", \"turn right\", \"stop\", \"[unk]\"]");

/* Process audio chunks - PCM S16 mono 16 kHz */
int16_t buf[4000];
int n;
while ((n = read(audio_fd, buf, sizeof(buf))) > 0) {
    if (vosk_recognizer_accept_waveform_s(rec, buf, n / 2)) {
        /* Utterance complete */
        /* Returns JSON: {"text":"hello","result":[{"conf":0.98,
                          "start":0.0,"end":0.5,"word":"hello"}]} */
        printf("%s\n", vosk_recognizer_result(rec));
    } else {
        printf("Partial: %s\n", vosk_recognizer_partial_result(rec));
    }
}
printf("Final: %s\n", vosk_recognizer_final_result(rec));
vosk_recognizer_free(rec);
vosk_model_free(model);
```

[Source](https://github.com/alphacep/vosk-api/blob/master/src/vosk_api.h)

### Model Tiers and WER

| Language | Model size | WER (%) |
|----------|-----------|---------|
| English small | 40 MB | 9.85 |
| English large | 1.8 GB | 5.69 |
| Russian small | 45 MB | 22.71 |
| Russian large | 1.8 GB | 4.5 |
| Chinese small | 42 MB | 23.54 |
| Chinese large | 1.3 GB | 13.98 |
| French small | 41 MB | 23.95 |
| French large | 1.4 GB | 14.72 |

Small models require approximately 300 MB RAM at runtime; large models up to 16 GB. [Source](https://alphacephei.com/vosk/models)

### WebSocket Server

Vosk ships a WebSocket ASR server (`vosk-server`) built with Python asyncio:

```python
# vosk-server/websocket/asr_server.py (simplified)
import asyncio, websockets
from vosk import KaldiRecognizer, Model
from concurrent.futures import ThreadPoolExecutor

model = Model("/path/to/vosk-model")
executor = ThreadPoolExecutor(max_workers=4)

async def recognize(websocket):
    rec = KaldiRecognizer(model, 16000)
    async for data in websocket:
        loop = asyncio.get_event_loop()
        if isinstance(data, bytes):
            result = await loop.run_in_executor(
                executor, rec.AcceptWaveform, data)
            if result:
                await websocket.send(rec.Result())
            else:
                await websocket.send(rec.PartialResult())

asyncio.run(websockets.serve(recognize, "0.0.0.0", 2700))
```

[Source](https://github.com/alphacep/vosk-server/blob/master/websocket/asr_server.py)

The server also supports gRPC, MQTT, and WebRTC transports. For multi-threaded GPU decoding, `vosk_gpu_init()` initialises CUDA globally and `vosk_gpu_thread_init()` sets up per-thread CUDA contexts. Vosk's batch API (`VoskBatchModel` / `VoskBatchRecognizer`) parallelises decode across multiple audio streams via a work queue.

---

## ALSA and PipeWire Capture Pipeline

### Buffer Sizing

Low-latency ASR capture requires careful buffer and period configuration. At 48 kHz (PipeWire's typical default):

| PipeWire quantum (samples) | Approximate latency |
|---------------------------|---------------------|
| 64 | 1.3 ms |
| 128 | 2.7 ms |
| 256 | 5.3 ms |
| 512 | 10.7 ms |
| 1024 | 21.3 ms |
| 2048 | 42.7 ms |

For ASR applications, a quantum of 512–1024 samples offers a good balance between capture latency and CPU overhead. Very small quanta require rtkit real-time scheduling (`SCHED_FIFO`) to avoid dropouts; PipeWire communicates with rtkit-daemon automatically for registered clients. [Source](https://docs.pipewire.org/page_module_loopback.html)

ALSA FIFO period size for the ALSA plugin is configurable in `~/.config/pipewire/media-session.d/alsa-monitor.conf` via `api.alsa.period-size`. PipeWire 1.4.8 improved low-latency handling for ALSA FireWire devices via IRQ-mode enforcement in the Pro-Audio profile.

### Resampling with libswresample

whisper.cpp and Vosk require 16 kHz mono PCM. PipeWire typically captures at 48 kHz stereo. Resampling:

```c
/* Resample PipeWire 48 kHz stereo S16 to 16 kHz mono S16
 * using libswresample (part of FFmpeg)
 */
#include <libswresample/swresample.h>
#include <libavutil/channel_layout.h>
#include <libavutil/samplefmt.h>

SwrContext *swr = NULL;
AVChannelLayout in_layout  = AV_CHANNEL_LAYOUT_STEREO;
AVChannelLayout out_layout = AV_CHANNEL_LAYOUT_MONO;

swr_alloc_set_opts2(&swr,
    &out_layout, AV_SAMPLE_FMT_S16, 16000,
    &in_layout,  AV_SAMPLE_FMT_S16, 48000,
    0, NULL);
swr_init(swr);

/* per-callback: in_samples = quantum_size, out_samples scaled 16k/48k */
int out_samples = av_rescale_rnd(in_samples, 16000, 48000, AV_ROUND_UP);
swr_convert(swr,
            (uint8_t **)&out_buf, out_samples,
            (const uint8_t **)&in_buf, in_samples);

swr_free(&swr);
```

Alternatively, request 16 kHz mono directly from PipeWire using `spa_format_audio_raw_build()` with the desired rate and channel count — PipeWire's internal SPA resampler handles the conversion, offloading the work from the application.

### PipeWire Virtual Monitor Source

For always-on capture that does not interfere with other applications, create a virtual loopback source:

```ini
# ~/.config/pipewire/pipewire.conf.d/10-loopback.conf
# Creates a virtual Audio/Source fed from the hardware microphone
context.modules = [
    { name = libpipewire-module-loopback
      args = {
        node.description = "Microphone Monitor"
        capture.props = {
            node.name      = "capture.mic_monitor"
            audio.position = [ FL ]
            target.object  = "alsa_input.pci-0000_00_1f.3.analog-stereo"
            stream.dont-remix = true
        }
        playback.props = {
            node.name         = "playback.mic_monitor"
            media.class       = "Audio/Source"
            audio.position    = [ MONO ]
            node.description  = "Virtual Microphone Source"
        }
      }
    }
]
```

ASR applications connect to `playback.mic_monitor` by name. Multiple consumers (wake word detector, streaming ASR, audio logger) can all read from the same virtual source simultaneously without monopolising the hardware capture node. [Source](https://docs.pipewire.org/page_module_loopback.html)

---

## Accessibility Stack Integration

### AT-SPI2 and Data Flow

AT-SPI2 (Assistive Technology Service Provider Interface) is the standard accessibility bus on Linux, using D-Bus (`org.a11y.Bus`) to bridge GTK4 and Qt applications to assistive technologies. Version 2.56.4 (August 2025) is the current stable release. [Source](https://en.wikipedia.org/wiki/Assistive_Technology_Service_Provider_Interface)

The full data flow for a GTK4 application feeding GNOME Orca:

```
GTK4/Qt application
  → ATK/Qt Accessibility API
  → AT-SPI2 D-Bus bridge
    (publishes accessible objects: focus-changed, name-changed, text-changed events)
  → Orca screen reader (Python/GObject, subscribes to AT-SPI2 events)
    (formats speech output with language/voice metadata)
  → libspeechd client (Unix socket SSIP, spd_say())
  → speech-dispatcher server
    (priority queue, module routing)
  → espeak-ng or piper module
    (libespeak-ng / Piper ONNX inference)
  → PCM audio via PipeWire/ALSA
```

Orca uses speech-dispatcher as its default TTS back-end. When configured with a Piper module, the latency from screen event to first audio is typically 50–150 ms on modern hardware — well within the 200 ms perceptible threshold for interactive feedback.

### IBus Integration

IBus (Intelligent Input Bus) serves CJK and other complex input method engines. When TTS feedback is desired for character pronunciation hints — common accessibility practice in East Asian locales — IBus can invoke speech-dispatcher via the same SSIP interface, using `spd_say(conn, SPD_NOTIFICATION, phonetic_hint)`. This gives pronunciation guidance without interrupting the primary screen reader flow, since `SPD_NOTIFICATION` is preemptible by higher-priority levels.

---

## Wake Word Detection

### openWakeWord

openWakeWord implements a three-stage pipeline: (1) ONNX mel-spectrogram of 80 ms frames of 16 kHz 16-bit PCM; (2) a shared embedding backbone (TFHub-based architecture) that extracts speech features shared across all loaded models; (3) a lightweight per-wake-word classifier (fully connected or 2-layer RNN). Multiple wake word models run simultaneously in a single `predict()` call. [Source](https://github.com/dscripka/openWakeWord)

```python
# openWakeWord integration with PipeWire capture
# File: integration pattern using openWakeWord and pyaudio
import openwakeword, numpy as np
from openwakeword.model import Model
import pyaudio

openwakeword.utils.download_models()

model = Model(
    wakeword_models=["alexa", "hey_mycroft"],
    inference_framework="onnx"   # or 'tflite'
)

# PipeWire capture at 16 kHz mono (resampled internally by PW)
pa = pyaudio.PyAudio()
audio = pa.open(rate=16000, channels=1, format=pyaudio.paInt16,
                input=True, frames_per_buffer=1280)  # 80 ms at 16 kHz

while True:
    frame = audio.read(1280, exception_on_overflow=False)
    frame_int16 = np.frombuffer(frame, dtype=np.int16)

    predictions = model.predict(frame_int16)

    for wake_word, score in predictions.items():
        if score > 0.5:
            print(f"Wake word: {wake_word} (score={score:.3f})")
            start_asr_session()
```

Performance target: false-reject rate below 5%, false-accept below 0.5 per hour. All openWakeWord models are trained exclusively on synthetic speech, eliminating the need for manual data collection. The system can run 15–20 models simultaneously on a Raspberry Pi 3 CPU core. [Source](https://github.com/dscripka/openWakeWord)

### Porcupine (PicoVoice)

Porcupine is a proprietary ANSI-C wake word engine with a demonstrated latency advantage on constrained hardware. Its API:

```c
/* Porcupine C API
 * Header: include/pv_porcupine.h (PicoVoice SDK)
 */
pv_porcupine_t *handle = NULL;
pv_status_t status = pv_porcupine_init(
    access_key,     /* PicoVoice cloud access key */
    model_path,     /* .pv model file path */
    PV_DEVICE_CPU,
    1,              /* number of keywords */
    &keyword_path,  /* .ppn keyword file array */
    &sensitivity,   /* float [0,1] per keyword */
    &handle);

/* Audio: 16-bit PCM mono at engine sample rate,
   in frames of pv_porcupine_frame_length() samples */
int32_t keyword_index = -1;
pv_porcupine_process(handle, pcm_frame, &keyword_index);
/* keyword_index >= 0 means detection */

pv_porcupine_delete(handle);
```

[Source](https://github.com/Picovoice/porcupine)

Porcupine requires a commercial licence for production deployments. Sensitivity range is `[0, 1]`; higher values reduce false-rejects at the cost of more false-accepts.

### Always-On Pipeline Integration

The complete always-on pattern using PipeWire:

1. Create virtual loopback source pointing at the hardware microphone (see §8 above).
2. Open a PipeWire capture stream at 16 kHz, 16-bit, mono with quantum = 256–512 samples.
3. A dedicated real-time thread (`SCHED_FIFO`, registered with rtkit) feeds 80 ms frames to the wake word model.
4. On activation, wake the ASR thread and begin piping audio to whisper.cpp or Vosk.
5. After end-of-utterance (VAD silence), return to wake word monitoring.

The PipeWire quantum should be 256–1024 samples (5–21 ms at 48 kHz) for responsive capture. [Source](https://docs.pipewire.org/page_module_loopback.html)

---

## Benchmarking

### Word Error Rate (WER)

Standard ASR benchmarks use LibriSpeech (clean and other conditions) and Mozilla Common Voice. Key results:

| System | Model | LibriSpeech test-clean WER | Notes |
|--------|-------|---------------------------|-------|
| whisper.cpp | large-v3 F16 | 2.7% | GPU recommended |
| whisper.cpp | small Q5_1 | 3.5–3.7% | CPU-viable |
| Vosk | English large | 5.69% | Streaming, partial results |
| Vosk | English small | 9.85% | 40 MB, embedded-viable |

Common Voice results vary significantly by language and recording conditions; Vosk small models show 20–25% WER on non-English languages with limited training data.

[Sources](https://alphacephei.com/vosk/models), [whisper.cpp README](https://github.com/ggml-org/whisper.cpp/blob/master/README.md)

### Real-Time Factor

RTF (real-time factor): decode time divided by audio duration. RTF < 1.0 means faster than real-time.

| System | Hardware | RTF |
|--------|----------|-----|
| whisper.cpp large-v3 | Intel i9-12900K (CPU) | ~2.5 |
| whisper.cpp large-v3 | RTX 4070 (CUDA) | ~0.12 |
| whisper.cpp large-v3 | Radeon 680M iGPU (Vulkan) | ~0.7 |
| whisper.cpp small Q5_0 | Intel i9-12900K (CPU) | ~0.3 |
| Vosk English small | Raspberry Pi 4 | ~0.6 |

[Source](https://gigagpu.com/whisper-large-v3-rtf-by-gpu/)

### Latency to First Word (Streaming)

whisper.cpp does not emit partial results — it waits for a complete segment before output, typically 3–10 seconds of audio. In streaming mode with `stream.cpp`, the fixed-window approach emits results every N seconds configured by `step_ms`.

Vosk produces partial results after approximately 300–600 ms of silence-delimited speech, with partial updates during speech. For command-and-control use cases where total latency matters, Vosk with grammar-constrained mode is faster in practice than whisper.cpp streaming.

Cloud ASR (Google, AWS Transcribe): 300–800 ms round-trip latency for first word; whisper.cpp CUDA streaming: 50–300 ms per chunk. [Source](https://picovoice.ai/docs/benchmark/stt-whisper-cpp-streaming-speech-to-text/)

### Memory Footprint

| Engine | Model | RAM at inference |
|--------|-------|-----------------|
| espeak-ng | Built-in | ~15 MB |
| Piper | `low` (22 MB model) | ~80 MB |
| Piper | `medium` (65 MB model) | ~200 MB |
| Vosk | English small (40 MB) | ~300 MB |
| Vosk | English large (1.8 GB) | up to 16 GB |
| whisper.cpp | tiny Q5_0 | ~200 MB |
| whisper.cpp | large-v3 Q5_0 | ~1.2 GB |

---

## On-Device vs. Cloud Trade-offs

### TTS Comparison

| Engine | Latency (first audio) | Model size | Naturalness (MOS) | GPU required |
|--------|----------------------|-----------|-------------------|-------------|
| espeak-ng | <5 ms | ~30 MB | ~2.5 | No |
| Piper (low) | 10–30 ms | 22–65 MB | ~3.8 | No (CUDA optional) |
| Piper (medium) | 20–80 ms | 65–200 MB | ~4.1 | No (CUDA optional) |
| XTTS v2 (Coqui) | 200–500 ms | ~2 GB VRAM | ~4.4 | Yes for real-time |
| Cloud (Google TTS) | 300–600 ms | N/A | ~4.5 | N/A |

Piper (VITS ONNX) offers the best on-device balance: low latency (10–50 ms on modern CPUs), moderate footprint (30–200 MB), full privacy, runs offline. XTTS v2 (Coqui) adds voice cloning from a 3-second reference clip and higher naturalness, but requires 2 GB+ VRAM for real-time GPU inference; CPU inference is too slow for interactive use. [Source](https://github.com/OHF-Voice/piper1-gpl/blob/main/README.md)

### ASR Comparison

| System | Use case | Streaming | Model size | WER (en) |
|--------|---------|-----------|-----------|---------|
| Vosk small | Embedded, commands | Yes (partial) | 40 MB | 9.85% |
| Vosk large | Desktop offline | Yes (partial) | 1.8 GB | 5.69% |
| whisper.cpp small (CPU) | Transcription | Fixed-window | 852 MB (F16) | 3.4% |
| whisper.cpp large (GPU) | High-accuracy | Fixed-window | 3.9 GB (F16) | 2.7% |
| Cloud (AWS/Google) | Interactive | Yes | N/A | ~3–5% |

Decision heuristics:
- **Command-and-control (small vocabulary, low latency)**: Vosk with `vosk_recognizer_new_grm()` + grammar JSON. Sub-100-word vocabulary, near-zero latency partial results, 40 MB footprint.
- **General transcription, privacy required, GPU available**: whisper.cpp large-v3 Q5_0 with CUDA/Vulkan.
- **General transcription, CPU only**: whisper.cpp small or Vosk large.
- **Embedded / Raspberry Pi**: Vosk small, whisper.cpp tiny Q4_0.
- **Highest accuracy, cost insensitive**: Cloud ASR with local Vosk fallback.

### Privacy Considerations

All engines covered in this chapter operate entirely on-device: no audio leaves the local system. This is the primary advantage over cloud ASR for healthcare, legal, and financial transcription use cases. The trade-off is accuracy, especially for accented speech, domain-specific vocabulary, and low-resource languages where cloud providers have much larger training sets.

For applications requiring both privacy and high accuracy, a hybrid approach — local Vosk for wake-word-to-command recognition and whisper.cpp large for occasional full-sentence transcription — balances the constraints.

---

## Integrations

**Ch38 (PipeWire Audio Graph)**: TTS output from espeak-ng, Piper, or Festival is routed to PipeWire audio sinks via the speech-dispatcher module PCM output path or directly via `pw_stream` with `PW_DIRECTION_OUTPUT`. ASR capture streams open a `pw_stream` with `PW_DIRECTION_INPUT`. The virtual loopback source (`libpipewire-module-loopback`) described in §8 enables simultaneous capture by multiple ASR consumers without hardware conflicts.

**Ch38b (ALSA PCM Interface)**: Applications without PipeWire support use ALSA directly for capture (`snd_pcm_open()` with `SND_PCM_STREAM_CAPTURE`) and for TTS playback. The period and buffer sizing constraints described in §8 apply equally to the ALSA API; `snd_pcm_hw_params_set_period_size()` sets the interrupt granularity equivalent to the PipeWire quantum.

**Ch213 (VoIP Integration)**: ASR is used for transcription of VoIP calls (Vosk or whisper.cpp receiving the decoded audio stream from the SIP/WebRTC media path), and TTS provides IVR (Interactive Voice Response) prompts synthesised by Piper or espeak-ng. The PipeWire VoIP integration described in Ch213 (echo cancellation via libspeexdsp, `pw_stream` source/sink for SIP clients) feeds directly into the ASR capture pipeline.

**Ch214 (Bluetooth HFP Microphone)**: Bluetooth headsets using HFP (Hands-Free Profile) present as ALSA/PipeWire capture nodes after the BlueZ/PipeWire Bluetooth module creates SCO transport nodes. The 8 kHz or 16 kHz (mSBC wideband) audio from HFP maps directly to Vosk's required input format (16 kHz) or can be upsampled via libswresample before whisper.cpp inference.
