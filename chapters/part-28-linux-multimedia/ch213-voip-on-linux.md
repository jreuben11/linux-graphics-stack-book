# Chapter 213: VoIP on Linux: SIP, WebRTC, and Real-Time Communication

> **Part**: Part XXVIII — Linux Multimedia
> **Audience**: Systems developers and multimedia application developers building or integrating voice and video communication on Linux, including softphone implementers, VoIP infrastructure engineers, and browser platform engineers working with real-time media.

## Table of Contents

- [1. SIP Protocol Fundamentals](#1-sip-protocol-fundamentals)
  - [1.1 What is VoIP?](#11-what-is-voip)
  - [1.2 What is SIP?](#12-what-is-sip)
  - [1.3 What is RTP?](#13-what-is-rtp)
- [2. PJSIP Stack](#2-pjsip-stack)
- [3. liblinphone](#3-liblinphone)
- [4. Jami: Fully Decentralised P2P Calls](#4-jami-fully-decentralised-p2p-calls)
- [5. Matrix/Element and MatrixRTC](#5-matrixelement-and-matrixrtc)
- [6. WebRTC on Linux](#6-webrtc-on-linux)
- [7. PipeWire Role in VoIP](#7-pipewire-role-in-voip)
- [8. Codec Landscape](#8-codec-landscape)
- [9. QoS and Real-Time Scheduling](#9-qos-and-real-time-scheduling)
- [10. NAT Traversal](#10-nat-traversal)
- [11. Video Calling](#11-video-calling)
- [12. Security Model](#12-security-model)
- [13. Testing and Debugging](#13-testing-and-debugging)
- [Integrations](#integrations)

---

## 1. SIP Protocol Fundamentals

SIP (Session Initiation Protocol, [RFC 3261](https://www.rfc-editor.org/rfc/rfc3261.html)) is a text-based request/response protocol layered over UDP, TCP, or TLS on port 5060 (or 5061 for SIPS/TLS). Every real-time communication stack on Linux — whether PJSIP, liblinphone, or Asterisk — builds on the same two orthogonal state machines defined in that specification.

### Transaction State Machine

Transactions are short-lived and correspond to a single request plus its responses. The INVITE client transaction traverses five states:

```
Calling → Proceeding (on 1xx provisional) → Completed (on 3xx–6xx) → Terminated
                                           ↘ Terminated (on 2xx via dialog layer)
```

Timer B fires after 64 × T1 (default T1 = 500 ms, so 32 s) if no response arrives, causing the transaction to time out. On unreliable transports, Timer A retransmits the INVITE exponentially until Timer B fires. Critically, on reception of a 2xx response the ACK is **not** handled by the INVITE client transaction — it is sent directly by the dialog layer, bypassing the transaction FSM.

The **branch parameter** of the Via header carries a globally unique transaction identifier. RFC 3261 mandates the magic cookie prefix `z9hG4bK` followed by a hash of call-ID, CSeq, From-tag, To-tag, Request-URI, and topmost Via host. Proxies match incoming requests to transactions using this branch value.

### Dialog State Machine

A dialog is long-lived and spans the entire call. It is created upon receipt of a 2xx response to INVITE and identified by the triplet (Call-ID, local-tag, remote-tag). The dialog tracks:

- Local and remote sequence numbers (`CSeq`)
- Local and remote URI (stored from `Contact` headers)
- Route set (ordered list of `Record-Route` headers, reversed for in-dialog requests)

The `Contact` header is mandatory in INVITE requests and 2xx responses; it carries the direct SIP URI used for subsequent BYE and re-INVITE routing, bypassing the original proxy chain. A BYE request terminates the dialog. A re-INVITE within the dialog updates session parameters — most commonly changing the SDP to add video, modify codec, or implement call hold.

### SDP Offer/Answer

SDP exchange follows [RFC 3264](https://www.rfc-editor.org/rfc/rfc3264.html). The INVITE body is the **offer** — it contains one or more `m=` media descriptions with a list of supported RTP payload types, for example:

```sdp
v=0
o=alice 2890844526 2890844526 IN IP4 192.168.1.10
s=-
c=IN IP4 192.168.1.10
t=0 0
m=audio 49170 RTP/AVP 0 8 97
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:97 iLBC/8000
m=video 51372 RTP/AVP 31 32
a=rtpmap:31 H261/90000
a=rtpmap:32 MPV/90000
```

The 200 OK body is the **answer** — it restricts each `m=` line to the subset of codecs the answerer supports, selecting a single codec per stream. If a media type cannot be accepted, the answerer sets the port to 0.

PJMEDIA enforces a five-state negotiator for offer/answer: `NULL → LOCAL_OFFER → REMOTE_OFFER → WAIT_NEGO → DONE`, exposed through [`pjmedia_sdp_neg_create_w_local_offer()`](https://docs.pjsip.org/en/latest/api/generated/pjmedia/group/group__PJMEDIA__SDP__NEG.html), `pjmedia_sdp_neg_negotiate()`, and `pjmedia_sdp_neg_get_active_local/remote()`.

### 1.1 What is VoIP?

Voice over Internet Protocol (VoIP) is the family of technologies that digitise audio (and, by extension, video) and deliver it as packets over an IP network rather than over a dedicated circuit-switched connection. A VoIP system must solve three distinct problems: signaling (negotiating who calls whom, on what terms, and how the session ends), media transport (moving compressed audio/video samples across the network with low latency), and quality control (compensating for packet loss, jitter, and reordering).

On Linux these responsibilities map to distinct layers of the software stack. Signaling is handled by protocol libraries such as PJSIP or liblinphone, which implement SIP or proprietary protocols. Media transport relies on RTP running over UDP, with RTCP carrying statistics. Quality is maintained through jitter buffers, adaptive codecs, and echo cancellation integrated into media engines like PJMEDIA or Mediastreamer2. PipeWire provides the kernel-adjacent audio capture and playback plumbing that feeds these engines, replacing the earlier PulseAudio and ALSA direct-access approaches.

A complete Linux VoIP deployment therefore spans user-space libraries (PJSIP, liblinphone, libwebrtc), system audio services (PipeWire, WirePlumber), kernel audio drivers (ALSA HDA, USB Audio), and network subsystems (netfilter for NAT, tc for QoS shaping). This chapter traces those layers from SIP signaling through to real-time scheduling and codec selection.

### 1.2 What is SIP?

SIP (Session Initiation Protocol, RFC 3261) is a text-based application-layer protocol for establishing, modifying, and terminating multimedia sessions. It borrows its syntax and request/response model from HTTP and its addressing scheme from SMTP, using SIP URIs of the form `sip:user@domain` to identify endpoints. SIP itself carries no media; its sole purpose is signaling — it locates the remote party, negotiates session parameters via an embedded SDP body, and tears the session down with a BYE request when the call ends.

The protocol operates over UDP port 5060 (or TCP/TLS for reliability and encryption). Registration, the mechanism by which a SIP phone announces its current IP address to a registrar, uses the REGISTER method. Presence and instant messaging extensions (RFC 3856, RFC 3428) layer on the same core via SUBSCRIBE, NOTIFY, and MESSAGE methods.

On Linux, every major VoIP library — PJSIP, liblinphone, Twinkle, and Asterisk — builds its signaling core on RFC 3261 transaction and dialog state machines. The SDP negotiated inside SIP messages specifies the RTP port numbers, codecs, and SRTP keying material that the media layer then uses independently of SIP. Understanding SIP's separation of signaling from media is essential for diagnosing VoIP failures, because a call can succeed at the SIP level while the audio path fails entirely due to NAT, firewall, or codec mismatch.

### 1.3 What is RTP?

RTP (Real-time Transport Protocol, RFC 3550) is the standard transport for audio and video in VoIP and multimedia streaming. Unlike TCP, it makes no delivery guarantees; it runs over UDP and accepts that some packets will arrive late or not at all. RTP compensates through sequence numbers (allowing reordering detection), timestamps (driving playout scheduling in the jitter buffer), and SSRC identifiers (distinguishing multiple media streams within a session). RTCP, the companion control protocol, carries per-sender and per-receiver statistics — packet loss fraction, interarrival jitter, and round-trip time — that codecs and congestion controllers use to adapt their behaviour.

Each SDP `m=` line negotiates a separate RTP session. An audio-video call typically runs two RTP sessions on distinct UDP port pairs, each with a corresponding RTCP port at `RTP_port + 1` by convention (or via `a=rtcp:` attribute). Payload type numbers in the RTP header map to codec descriptions registered in the SDP `a=rtpmap:` lines.

On Linux, RTP is implemented entirely in user space. PJMEDIA's RTP session (`pjmedia_rtp_session`), Mediastreamer2's `RtpSession`, and libwebrtc's `RtpTransport` all manage RTP framing, RTCP scheduling, and jitter buffer interaction. Encryption is handled via SRTP (RFC 3711), which wraps RTP with AES-CM or AES-GCM authenticated encryption using keys exchanged either through SDP `a=crypto:` lines (SDES) or through DTLS-SRTP handshake as required by WebRTC.

---

## 2. PJSIP Stack

[PJSIP 2.17](https://github.com/pjsip/pjproject) (released April 2026, GPL-2.0) is the current release of the most widely deployed open-source SIP C library. Its layered architecture is shown below:

```
┌─────────────────────────────────────────────────────────────┐
│  PJSUA2 (C++ API with SWIG bindings: Java / Python / C#)   │
│  Endpoint │ Account │ Call │ Buddy │ Media                   │
├─────────────────────────────────────────────────────────────┤
│  PJSUA-LIB (C integration API)                              │
│  Integrates SIP + media + NAT traversal into one API        │
├──────────────┬──────────────────┬───────────────────────────┤
│  PJSIP       │  PJMEDIA         │  PJNATH                   │
│  SIP stack   │  Codecs/RTP/JB   │  ICE / STUN / TURN        │
├──────────────┴──────────────────┴───────────────────────────┤
│  PJLIB-UTIL (utilities)  │  PJLIB (OS portability layer)    │
└─────────────────────────────────────────────────────────────┘
```

### PJSUA2 High-Level C++ API

The [PJSUA2 API](https://docs.pjsip.org/en/latest/pjsua2/intro.html) exposes five primary classes:

- **`Endpoint`** — singleton that initialises the stack; must be created first.
- **`Account`** — SIP identity (URI, registrar, credentials); at least one required.
- **`Call`** — INVITE session with methods `makeCall()`, `answer()`, `setHold()`, `reinvite()`, `xfer()`, `getAudioMedia()`, `getVideoMedia()`.
- **`Buddy`** — presence subscription (SUBSCRIBE/NOTIFY).
- **`Media`** — abstract base; `AudioMedia`, `AudioMediaPlayer`, `AudioMediaRecorder`, `VideoMedia`.

```cpp
// pjsua2/account.hpp — minimal registration example
// Adapted from docs.pjsip.org/en/latest/pjsua2/using/call.html
#include <pjsua2.hpp>

class MyCall : public pj::Call {
public:
    MyCall(pj::Account &acc, int call_id = PJSUA_INVALID_ID)
        : pj::Call(acc, call_id) {}

    void onCallState(pj::OnCallStateParam &prm) override {
        pj::CallInfo ci = getInfo();
        if (ci.state == PJSIP_INV_STATE_CONFIRMED) {
            // Media is active — connect audio to conference bridge
            pj::AudioMedia &captureMedia =
                pj::Endpoint::instance().audDevManager().getCaptureDevMedia();
            pj::AudioMedia am = getAudioMedia(-1); // first audio stream
            captureMedia.startTransmit(am);
            am.startTransmit(
                pj::Endpoint::instance().audDevManager().getPlaybackDevMedia());
        }
    }
};
```

Threading: PJSUA2 creates internal worker threads that poll the SIP stack. For Python or Java SWIG bindings, set `threadCnt=0` in `EpConfig.uaConfig` and drive `Endpoint::libHandleEvents()` from the application event loop to avoid GIL conflicts. Errors surface as `pj::Error` exceptions with `status`, `reason`, and `srcFile` fields.

### PJSUA-LIB C API

The lower-level [PJSUA-LIB C API](https://docs.pjsip.org/en/latest/get-started/guidelines-api.html) provides fine-grained call control:

```c
// Minimal UAC in PJSUA-LIB
// Source: pjproject/pjsip-apps/src/pjsua/pjsua_app.c (simplified)
pjsua_config     ua_cfg;
pjsua_logging_config log_cfg;
pjsua_media_config   media_cfg;

pjsua_config_default(&ua_cfg);
pjsua_logging_config_default(&log_cfg);
pjsua_media_config_default(&media_cfg);

pjsua_create();
pjsua_init(&ua_cfg, &log_cfg, &media_cfg);
pjsua_transport_create(PJSIP_TRANSPORT_UDP, NULL, &transport_id);
pjsua_start();

// Account registration
pjsua_acc_config acc_cfg;
pjsua_acc_config_default(&acc_cfg);
acc_cfg.id = pj_str("sip:alice@example.com");
acc_cfg.reg_uri = pj_str("sip:example.com");
pjsua_acc_add(&acc_cfg, PJ_TRUE, &acc_id);

// Make call
pj_str_t dst = pj_str("sip:bob@example.com");
pjsua_call_make_call(acc_id, &dst, NULL, NULL, NULL, &call_id);
```

Key call-control functions: `pjsua_call_answer()`, `pjsua_call_hangup()`, `pjsua_call_set_hold()`, `pjsua_call_xfer()`, `pjsua_call_dial_dtmf()`, `pjsua_conf_connect()`.

### PJMEDIA Audio Dataflow

The [PJMEDIA audio pipeline](https://docs.pjsip.org/en/latest/specific-guides/media/audio_flow.html) is timer-driven. A speaker timer calls `pjmedia_port_get_frame()` on the conference bridge at regular intervals; the bridge mixes frames from all connected ports and delivers PCM samples to the sound device. On the receive path, IOQueue worker threads call `on_rx_rtp()` when UDP datagrams arrive; packets are unpacked via the internal RTP session and inserted into the jitter buffer.

The **Adaptive Delay Buffer** continuously measures actual device jitter and adjusts the playout delay to compensate without introducing audible pitch shifts. The default `PJMEDIA_SOUND_BUFFER_COUNT` targets approximately 150 ms of buffering. The conference bridge (`pjmedia_conf_create2()`) operates in software mix mode by default; it can be connected to hardware mixing via the AEC port.

### PJNATH ICE Transport

[PJNATH](https://www.pjsip.org/pjnath/docs/html/) encapsulates ICE, STUN, and TURN. Each `pj_ice_strans` holds N socket pairs (typically two: RTP + RTCP). The lifecycle follows:

```c
// Source: docs.pjsip.org/en/latest/specific-guides/network_nat/standalone_ice.html
pj_ice_strans_cfg ice_cfg;
pj_ice_strans_cfg_default(&ice_cfg);
ice_cfg.stun_cfg.max_pkt_size = 4096;
// Configure STUN server
ice_cfg.stun.server = pj_str("stun.example.com");
ice_cfg.stun.port   = 3478;
// Configure TURN server
ice_cfg.turn.server = pj_str("turn.example.com");
ice_cfg.turn.port   = 3478;
ice_cfg.turn.auth_cred.type = PJ_STUN_AUTH_CRED_STATIC;

pj_ice_strans *ice_st;
pj_ice_strans_create("ice", &ice_cfg, 2 /*comp*/, NULL, &icecb, &ice_st);
pj_ice_strans_init_ice(ice_st, PJ_ICE_SESS_ROLE_CONTROLLING, NULL, NULL);
// ... enumerate candidates via pj_ice_strans_enum_cands()
// ... exchange candidates with remote peer via signaling
pj_ice_strans_start_ice(ice_st, &rem_ufrag, &rem_passwd, cand_cnt, rem_cands);
// on_ice_complete callback fires with PJ_ICE_STRANS_OP_NEGOTIATION result
```

ICE negotiation typically completes in 10–100 ms for direct paths; failure detection requires 7–8 s. STUN/TURN keep-alive runs every 15 s. Trickle ICE is supported, allowing candidates to be exchanged incrementally during gathering.

---

## 3. liblinphone

[liblinphone](https://github.com/BelledonneCommunications/liblinphone) (AGPL v3 with proprietary dual license) is the multimedia SDK behind Linphone and Flexisip. It wraps a cohesive set of C++ libraries:

```
linphone-sdk
├── liblinphone        (LinphoneCore / linphone::Core C++ API)
│   ├── BelleSIP       (SIP stack replacing Sofia-SIP)
│   ├── Mediastreamer2 (audio/video filter graph)
│   │   └── Bzrtp      (ZRTP key agreement, RFC 6189)
│   ├── Belr           (ABNF grammar parser for SDP/SIP)
│   ├── Belcard        (vCard 4.0)
│   └── BcToolbox      (portability, threading primitives)
└── LIME               (e2e messaging: Double Ratchet + Crystals-Kyber PQC)
```

### LinphoneCore Lifecycle

```cpp
// Source: wiki.linphone.org/xwiki/wiki/public/view/Lib/
// Simplified C++ API usage
linphone::Factory *factory = linphone::Factory::get();
shared_ptr<linphone::Core> core =
    factory->createCore("config.rc", "factory-config.rc", nullptr);

// Set audio devices
core->setInputAudioDevice(core->getDefaultInputAudioDevice());
core->setOutputAudioDevice(core->getDefaultOutputAudioDevice());

// Register SIP account
auto accountParams = core->createAccountParams();
accountParams->setIdentityAddress(
    factory->createAddress("sip:alice@example.com"));
accountParams->setServerAddress(
    factory->createAddress("sip:example.com;transport=tls"));
accountParams->setRegisterEnabled(true);
auto account = core->createAccount(accountParams);
core->addAccount(account);

core->start();
// Application drives event loop: core->iterate() in a timer callback
```

### Call States and ZRTP Verification

The `linphone::Call` object traverses [22 states](https://download.linphone.org/snapshots/docs/liblinphone/5.2/c++/classlinphone_1_1Call.html), including:

- `OutgoingInit` → `OutgoingProgress` → `OutgoingRinging` → `StreamsRunning`
- `StreamsRunning` → `Paused` / `PausedByRemote` (hold)
- `StreamsRunning` → `UpdatedByRemote` (re-INVITE from remote)
- `End` → `Released`

Call control: `accept()`, `pause()`, `resume()`, `terminate()`, `decline(Reason)`, `transferTo()`, `update(CallParams)`. Audio device routing is per-call: `setInputAudioDevice()`, `setOutputAudioDevice()`.

ZRTP verification proceeds out-of-band. After media streams start, both peers receive a four-character Short Authentication String (SAS):

```cpp
// ZRTP SAS verification — both users read aloud and compare
string sas = call->getAuthenticationToken();  // e.g. "r3dn"
bool verified = call->getAuthenticationTokenVerified();
if (!verified) {
    // User verbally confirms SAS matches remote peer
    call->setAuthenticationTokenVerified(true);
    // Result cached in ZRTP DB; future calls to same peer auto-verified
}
```

The LIME layer adds Double Ratchet forward-secrecy for instant messaging and experimentally applies Crystals-Kyber post-quantum key encapsulation for encrypted group call key distribution.

---

## 4. Jami: Fully Decentralised P2P Calls

[Jami](https://jami.net/) (formerly Ring) eliminates the SIP server entirely. Peer discovery and call signaling travel through [OpenDHT](https://github.com/savoirfairelinux/opendht), a C++17 Kademlia distributed hash table (MIT license). Every Jami account IS an OpenDHT node; there is no central registration server.

### OpenDHT Architecture

```
Jami Client App
      │
  libjami (ring-daemon)
      │
  ┌───┴────────────────────────────────────┐
  │ OpenDHT (Kademlia DHT, C++17)          │
  │   dht::DhtRunner node                  │
  │   Bootstrap: bootstrap.jami.net:4222   │
  │   Key: SHA-1(accountPublicKey)         │
  └─────────────────────────────────────-──┘
      │
  ICE/STUN/TURN (pjnath)
      │
  Direct P2P media (SRTP, end-to-end encrypted)
```

```cpp
// Source: github.com/savoirfairelinux/opendht/blob/master/README.md
#include <opendht.h>

dht::DhtRunner node;
// Start DHT node on port 4222, generate identity
node.run(4222, dht::crypto::generateIdentity("mynode"), true);
// Bootstrap from known node
node.bootstrap("bootstrap.jami.net", "4222");

// Publish value under a key
auto key = dht::InfoHash::get("alice@jami.net");
node.put(key, dht::Value("hello"), [](bool success){
    if (success) printf("Value published to DHT\n");
});

// Listen for values under a key
node.listen(key, [](const std::vector<std::shared_ptr<dht::Value>>& values) {
    for (auto& v : values) {
        // Process incoming call offer (encrypted ICE SDP)
    }
    return true; // keep listening
});
```

Dependencies: `msgpack-c >= 1.2`, `GnuTLS >= 3.3`, `Nettle >= 2.4`, `{fmt} >= 9.0`.

### Call Establishment Flow

Call establishment without SIP: the caller encrypts an ICE SDP offer under the callee's public key and publishes it to the DHT at the callee's key hash. The callee listens continuously at its key hash, retrieves the encrypted offer, decrypts it with its private key, and publishes an ICE answer back under the caller's key hash. ICE negotiation then proceeds directly between endpoints (PJNATH provides the ICE engine). Media streams are SRTP-protected with keys derived during the ICE + DTLS handshake.

For human-readable addressing, [JamiNS](https://docs.jami.net/en_US/user/jami-distributed-network.html) resolves names (e.g., `alice.jami.net`) via an Ethereum blockchain contract queried over HTTP REST to `ns.jami.net`. Registration requires signing a blockchain transaction with the account private key. The DHT continues to function on local networks without internet access; name resolution degrades gracefully to raw key-hash addressing.

Every participating Jami node contributes storage to the DHT, making the network more resilient as it grows. End-to-end encryption with forward secrecy applies to all streams; there is no key escrow or server-side plaintext.

---

## 5. Matrix/Element and MatrixRTC

Matrix provides federated, end-to-end-encrypted messaging with a VoIP layer built on top of its room-event model.

### MatrixRTC (MSC4143)

[MSC4143 — MatrixRTC](https://github.com/matrix-org/matrix-spec-proposals/pull/4143) is the current group VoIP specification (part of Matrix 2.0). It supersedes the earlier [MSC3401](https://github.com/matrix-org/matrix-spec-proposals/pull/3401) native group VoIP signaling. MatrixRTC defines:

- **Session state** represented as Matrix room state events (`m.call.member`), not ephemeral to-device messages
- **Pluggable media transport backends** — the spec itself is transport-agnostic
- **LiveKit backend** (MSC4195) — the only currently usable backend, using LiveKit SFUs deployed per homeserver

MatrixRTC sessions can be distributed across multiple SFU instances for horizontal scaling (a 2025 capability). The spec is blocked pending MSC4354 (Sticky Events) and MSC4140 (Cancellable Delayed Events), which enable robust membership management.

### End-to-End Encryption

Matrix uses [libolm](https://gitlab.matrix.org/matrix-org/olm) for the Megolm session-based encryption applied to room events. For VoIP, the media-layer key distribution for per-call encryption travels through encrypted Matrix room events, so even SFU operators cannot decrypt participant audio/video. The Element client implements this as a "Per-Participant Keys" model where joining participants receive symmetric media keys encrypted to their Matrix device keys.

### Prior-Generation 1:1 Calls

The earlier per-room to-device event approach (MSC3401 and its predecessors) provided 1:1 calls with ICE candidates exchanged as Matrix events. While this mechanism still works in Element, MatrixRTC is the forward-looking architecture for both 1:1 and group calls.

---

## 6. WebRTC on Linux

### libwebrtc PeerConnection Architecture

The [Chromium libwebrtc PeerConnection](https://webrtc.googlesource.com/src/+/HEAD/pc/g3doc/peer_connection.md) is created through:

```cpp
// webrtc/api/peer_connection_interface.h
RTCConfiguration config;
config.sdp_semantics = webrtc::SdpSemantics::kUnifiedPlan;
config.servers = { {"stun:stun.example.com:3478"} };

auto result = factory->CreatePeerConnectionOrError(config, std::move(deps));
rtc::scoped_refptr<webrtc::PeerConnectionInterface> pc = result.MoveValue();
```

The PeerConnection is a "God object" composed of specialised sub-controllers:

```
PeerConnection
├── SdpOfferAnswerHandler     (offer/answer state machine, owns SDP)
├── RtpTransmissionManager    (RtpSender / RtpReceiver / RtpTransceiver)
├── DataChannelController     (SCTP-over-DTLS data channels)
├── JsepTransportController   (creates/configures DTLS+ICE transports per m= line)
├── Call                      (overall call, ReceiveStream/SendStream lifecycle)
└── RtcStatsCollector         (RTCStats report aggregation)

Transport stack per m= line:
  JsepTransport
  └── DtlsSrtpTransport   (exports SRTP keys after DTLS completes)
        ├── SrtpTransport (wraps libsrtp: ProtectRtp/UnprotectRtp)
        └── DtlsTransport (wraps OpenSSL/BoringSSL)
              └── P2PTransportChannel (ICE)
                    └── BasicPortAllocator
                          ├── UDPPort
                          ├── StunPort   (srflx candidates)
                          ├── TcpPort
                          └── TurnPort   (relay candidates)
```

Three thread types segregate concerns: `signaling_thread` (SDP and API calls), `worker_thread` (codec and RTP streams), `network_thread` (socket I/O). All are `rtc::Thread` instances; cross-thread calls use `PostTask`/`BlockingCall`.

### ICE Gathering in libwebrtc

[ICE gathering](https://webrtc.googlesource.com/src/+/HEAD/p2p/g3doc/ice.md) proceeds in five steps:

1. `SetLocalDescription()` triggers `MaybeStartGathering()` in `P2PTransportChannel`, which calls `BasicPortAllocator` to create port objects for each network interface.
2. `SignalCandidateGathered` fires as each candidate is discovered (host, srflx, relay); the application forwards candidates to the remote peer via its signaling channel.
3. `AddRemoteCandidate()` pairs each incoming remote candidate with local candidates to form candidate pairs.
4. STUN Binding requests run on pairs; success triggers `SignalCandidatePairChanged` and the active connection is selected via `IceControllerInterface`.
5. `SignalIceTransportStateChanged` transitions to `kConnected` and triggers the DTLS handshake.

### GStreamer webrtcbin

[GStreamer webrtcbin](https://gstreamer.freedesktop.org/documentation/webrtc/index.html) (`GstBin`) implements the W3C PeerConnection API as a GStreamer element. It uses libnice for ICE, libsrtp2 for SRTP, and GStreamer OpenSSL elements for DTLS.

```bash
# Sender pipeline — source: github.com/centricular/gstwebrtc-demos
gst-launch-1.0 \
  v4l2src ! queue ! videoconvert ! vp8enc ! rtpvp8pay ! \
  "application/x-rtp,media=video,payload=96,encoding-name=VP8" ! \
  webrtcbin name=send bundle-policy=max-bundle stun-server=stun://stun.l.google.com:19302
```

Key [webrtcbin signals](https://walterfan.github.io/gstreamer-cookbook/4.plugin/webrtcbin.html):

| Signal | Purpose |
|--------|---------|
| `on-negotiation-needed` | Pipeline entered PLAYING; trigger offer creation |
| `create-offer` / `create-answer` | Async SDP generation |
| `set-local-description` / `set-remote-description` | SDP installation |
| `on-ice-candidate` | Emit local ICE candidates to signaling |
| `add-ice-candidate` | Install remote ICE candidates |
| `pad-added` | New remote media stream available (src pad) |
| `on-data-channel` | Incoming SCTP data channel |

Bundle modes: `max-bundle` (single transport for all media, preferred) or `max-compat` (separate TransportStream per `m=` line). The `latency` property (default 200 ms) configures the internal jitter buffer.

---

## 7. PipeWire Role in VoIP

### Audio Routing for Softphones

VoIP softphones on Linux access audio through PipeWire via one of several compatibility layers:

- **PipeWire native API** — `pw_stream` with `PW_DIRECTION_INPUT` (microphone) and `PW_DIRECTION_OUTPUT` (speaker)
- **PulseAudio protocol** — via `pipewire-pulse` wire-protocol server (no library change needed)
- **ALSA PCM plugin** — `libasound_module_pcm_pipewire.so` for legacy ALSA-only softphones
- **JACK compatibility** — `pw-jack` LD_LIBRARY_PATH shim translating JACK API to PipeWire nodes

[WirePlumber](https://pipewire.pages.freedesktop.org/wireplumber/) as session manager enforces routing policy: it automatically links a SIP softphone's capture stream to the default microphone device and its playback stream to the default speaker, respecting user-configured device priorities.

### Real-Time Scheduling

[`page_module_rt`](https://docs.pipewire.org/page_module_rt.html) in PipeWire raises thread priority for the audio graph cycle:

```ini
# /etc/pipewire/pipewire.conf.d/rt.conf
context.modules = [
  { name = libpipewire-module-rt
    args = {
      rt.prio        = 88        # SCHED_FIFO priority
      rt.time.soft   = 200000    # 200 ms soft CPU limit per cycle
      rt.time.hard   = 400000    # 400 ms hard CPU limit (kernel kills above this)
      uclamp.min     = 0         # scheduler utilisation floor
      uclamp.max     = 100       # scheduler utilisation ceiling
    }
  }
]
```

The module first attempts `RLIMIT_RTPRIO` (direct `sched_setscheduler` if granted), then falls back to the XDG Portal Realtime D-Bus API, and finally RTKit. RTKit caps CPU monopoly to 20% of a single CPU core to prevent runaway RT threads from hanging the system.

### pw-loopback for VoIP Mixing

[`libpipewire-module-loopback`](https://docs.pipewire.org/page_module_loopback.html) links a capture node to a playback node in the PipeWire graph without touching the audio data. VoIP use cases:

```bash
# Create a virtual microphone that software SIP phone can select,
# mixing hardware mic with a music playback stream
pw-loopback \
  --capture-props='media.class=Audio/Source node.name=virtual-mic' \
  --playback-props='media.class=Stream/Output/Audio node.target=alsa_output.pci.stereo'
```

For call monitoring or recording, `pw-loopback` can bridge a sink (output) back to a source (input), making the speaker output available as a capture source for call recording without kernel-level loopback modules.

---

## 8. Codec Landscape

### Technical Comparison

The following codecs dominate Linux VoIP deployments:

| Codec | Bitrate | Bandwidth | Sampling | FEC | PLC | VBR | License |
|-------|---------|-----------|----------|-----|-----|-----|---------|
| G.711 | 64 kbps CBR | Narrowband | 8 kHz | No | No | No | ITU (free) |
| G.722 | 64 kbps CBR | Wideband | 16 kHz | No | Basic | No | ITU (free) |
| G.729 | 8 kbps CBR | Narrowband | 8 kHz | No | Limited | No | Patents expired 2017; commercial PBX still charge per-channel |
| Opus | 6–510 kbps | NB to fullband | 8–48 kHz | Yes | Yes | Yes | RFC 6716, royalty-free |

[Source: sipsymposium.com codec comparison](https://sipsymposium.com/guides/voip-codec-comparison-opus-g711-g729-g722); [RFC 6716](https://www.rfc-editor.org/rfc/rfc6716).

### Opus Internal Architecture

Opus ([RFC 6716](https://www.rfc-editor.org/rfc/rfc6716)) selects between two internal modes adaptively:

- **SILK** mode — LP-based codec optimised for narrowband/wideband speech at low bitrates (8–40 kbps). Provides better intelligibility for conversational speech.
- **CELT** mode — transform codec covering fullband audio at higher bitrates (40–510 kbps). Used for music, wideband conferencing, and low-latency applications.
- **Hybrid** mode — SILK + CELT in parallel for superwideband speech.

In-band FEC embeds a redundant low-bitrate encoding of the previous frame inside the current packet. When the decoder detects a lost packet (gap in sequence numbers), it uses the FEC data from the next received packet rather than pure PLC.

### NetEQ Adaptive Jitter Buffer

[NetEQ](https://chromium.googlesource.com/external/webrtc/+/master/modules/audio_coding/neteq/g3doc/index.md) in libwebrtc is the state-of-the-art adaptive jitter buffer used in Chrome/WebRTC. Two entry points: `InsertPacket()` (called on packet arrival) and `GetAudio()` (called every 10 ms on the playout thread).

```
GetAudio() decision tree (every 10 ms):
  ┌─ buffer level > target? ──▶ Acceleration
  │    time-compress audio (drop samples via WSOLA), reduce buffer
  ├─ buffer level < target? ──▶ Preemptive Expand
  │    time-stretch audio (insert samples via WSOLA), increase buffer
  ├─ packet on time?        ──▶ Normal decode
  ├─ packet lost?           ──▶ Expand (PLC: decoder extrapolation or CNG)
  ├─ post-loss, packet arrives ▶ Merge (WSOLA stitch at boundary)
  └─ DTX SID frame?         ──▶ Comfort Noise Generation
```

[Source: webrtchacks.com NetEQ explainer](https://webrtchacks.com/how-webrtcs-neteq-jitter-buffer-provides-smooth-audio/). Interarrival timing statistics continuously update the target buffer level, compensating for network jitter without degrading to a fixed 200 ms worst-case delay. NACK tracking generates RTCP NACK requests for retransmittable losses.

PJMEDIA's jitter buffer uses a simpler Adaptive Delay Buffer: it tracks actual playback jitter of the sound device (not just network jitter) and adjusts buffer depth to match. This ensures the combined network + device jitter is always covered.

---

## 9. QoS and Real-Time Scheduling

### DSCP Marking

Expedited Forwarding (EF) DSCP value 46 (`0x2E`) is the standard per-hop behaviour for voice RTP on managed networks. Setting it requires `CAP_NET_ADMIN` or a permissive `sysctl net.ipv4.ip_tos`:

```c
// Source: net/ipv4/route.c ip_tos2prio[] mapping
// Set DSCP/TOS on UDP socket for RTP traffic
int dscp_ef = 46;              // EF PHB
int tos = dscp_ef << 2;        // TOS = DSCP << 2 = 0xB8
setsockopt(rtp_sock, IPPROTO_IP, IP_TOS, &tos, sizeof(tos));

// Also set kernel socket priority for local qdisc scheduling
int prio = 6;                  // 0=lowest, 6=highest for non-root
setsockopt(rtp_sock, SOL_SOCKET, SO_PRIORITY, &prio, sizeof(prio));
```

The kernel maps the TOS byte to internal `sk_priority` via `ip_tos2prio[]` in `net/ipv4/route.c`. Traffic control enforces prioritisation at the egress qdisc:

```bash
# HTB qdisc with EF priority class — from DSCP/QoS for VoIP research
tc qdisc add dev eth0 root handle 1: htb default 30
tc class add dev eth0 parent 1: classid 1:10 htb rate 1mbit ceil 2mbit prio 1
tc class add dev eth0 parent 1: classid 1:30 htb rate 10mbit ceil 100mbit prio 3
# Mark EF-DSCP packets into priority class
tc filter add dev eth0 parent 1: protocol ip u32 \
    match ip tos 0xb8 0xff flowid 1:10
# Alternative: use iptables mangle
iptables -t mangle -A OUTPUT -p udp --dport 5004 -j DSCP --set-dscp 46
```

### Audio Thread Real-Time Priority

PJSIP and PipeWire both require elevated thread priority for audio processing. The hierarchy of mechanisms:

```
1. RLIMIT_RTPRIO (set by PAM / systemd rlimits)
   → sched_setscheduler(SCHED_FIFO, priority=88)
   → No CPU cap from kernel; RLIMIT_RTTIME provides watchdog

2. RTKit D-Bus API (org.freedesktop.RealtimeKit1)
   → MakeThreadRealtime(tid, priority)
   → Enforces 200 µs/s CPU budget (20% of one core)
   → pj_thread_set_priority() uses this as fallback

3. PipeWire page_module_rt
   → rt.time.soft/hard: CPU time limits per RT cycle
   → Kernel kills thread if it exceeds rt.time.hard
```

PJSIP audio threads should target SCHED_FIFO priority around 10–20 (below PipeWire's 88) to avoid preempting the audio graph driver thread.

---

## 10. NAT Traversal

### ICE RFC 8445

[ICE (Interactive Connectivity Establishment)](https://www.rfc-editor.org/rfc/rfc8445.html) solves NAT traversal by gathering multiple candidate addresses and systematically testing pairs. Candidate types:

- **Host** — local interface IP:port (gathered immediately)
- **Server-reflexive (srflx)** — external IP:port observed by STUN server ([RFC 5389](https://www.rfc-editor.org/rfc/rfc5389.html))
- **Peer-reflexive (prflx)** — external IP:port observed by the remote peer during connectivity checks
- **Relay** — TURN-allocated IP:port on the relay server ([RFC 5766](https://www.rfc-editor.org/rfc/rfc5766.html))

Candidate priority formula:

```
priority = type_preference × 2^24 + local_preference × 2^8 + (256 − component_ID)

type_preference: host=126, prflx=110, srflx=100, relay=0
```

Candidate pairs are sorted by descending `min(G,D) × 2^32 + max(G,D) × 2 + (G>D ? 1 : 0)` where G and D are controlling and controlled agent priorities for that pair. The controlling agent (INVITE offerer by default) selects the nominated pair.

### Symmetric NAT Problem

Symmetric NAT creates a unique external `(IP, port)` tuple per `(source, destination)` pair. STUN reflexive candidates are useless because the external port observed by the STUN server differs from the port seen by the peer. The mandatory fallback is a TURN relay allocation.

```
Full-cone NAT        → host or srflx pair succeeds
Port-restricted NAT  → srflx pair may succeed after simultaneous open
Symmetric NAT        → relay pair required (TURN)
```

### coturn TURN Server

[coturn](https://github.com/coturn/coturn) is the dominant open-source TURN/STUN server on Linux. It implements RFC 3489/5389/5780 (STUN), RFC 5766/6062/6156 (TURN), and RFC 8016 (TURN for WebRTC). Transport: UDP, TCP, TLS 1.3, DTLS 1.2. Ports: 3478 (STUN/TURN), 5349 (TLS/DTLS).

```bash
# Minimal turnserver.conf
listening-port=3478
tls-listening-port=5349
realm=example.com
# Long-term credential
user=alice:secretpassword
# Or TURN REST API time-limited secret (recommended for WebRTC)
use-auth-secret
static-auth-secret=my-hmac-secret

# Start
turnserver -c /etc/turnserver.conf -o
```

TURN REST API authentication: the client requests time-limited credentials from a web server, which computes `HMAC-SHA1(secret, timestamp:username)` as the password. The TURN server independently verifies the HMAC. This allows credential rotation without restarting the TURN server and avoids long-term password storage on clients.

---

## 11. Video Calling

### RTP Video Payload Formats

Video codecs in VoIP travel over RTP with codec-specific packetisation modes:

| Codec | RTP Payload | Packetisation spec | Default Profile |
|-------|------------|-------------------|-----------------|
| H.264 | dynamic (96+) | [RFC 6184](https://www.rfc-editor.org/rfc/rfc6184) | Constrained Baseline |
| VP8 | dynamic | [RFC 7741](https://www.rfc-editor.org/rfc/rfc7741) | — |
| VP9 | dynamic | [draft-ietf-payload-vp9](https://datatracker.ietf.org/doc/draft-ietf-payload-vp9/) | Profile 0 |
| AV1 | dynamic | [AOM AV1 RTP spec](https://aomediacodec.github.io/av1-rtp-spec/) | Seq 0 |

> **Note**: The VP9 RTP payload specification was a long-running IETF draft (`draft-ietf-payload-vp9`) that stalled for several years before eventually progressing; implementations reference the draft. AV1 RTP packetisation is specified by the Alliance for Open Media, not as an IETF RFC.

H.264 is the most widely supported for interop with SIP endpoints (PJSIP, liblinphone, hardware phones). VP8 is the baseline WebRTC codec. VP9 and AV1 provide better compression but have less universal hardware decode support.

### GStreamer V4L2 Pipeline with VA-API Hardware Encode

For low-latency video calls, software encode (x264, libvpx) introduces 50–100 ms of additional latency due to lookahead and buffering. Hardware VA-API encode reduces this to under 10 ms for single-frame encoding:

```bash
# V4L2 camera → hardware H.264 encode → RTP → webrtcbin
# vah264enc: new VA-API plugin merged into GStreamer main repo
gst-launch-1.0 \
  v4l2src device=/dev/video0 ! \
  video/x-raw,width=1280,height=720,framerate=30/1 ! \
  videoconvert ! \
  vah264enc rate-control=cbr bitrate=1500 ! \
  rtph264pay config-interval=1 pt=96 ! \
  "application/x-rtp,media=video,encoding-name=H264,payload=96" ! \
  webrtcbin name=wrtc
```

The `vah264enc` element (new VA plugin) and `vaapih264enc` (legacy, merged into main GStreamer) both call `vaBeginPicture()`, `vaRenderPicture()`, `vaEndPicture()` through libva. Rate control modes: CBR for video calls (predictable bitrate for DSCP marking), VBR for recording. For embedded and Raspberry Pi targets, `v4l2h264enc` uses the V4L2 M2M stateless encoder interface instead.

### Bandwidth Adaptation

RTCP feedback drives encoder bitrate adaptation ([RFC 4585](https://www.rfc-editor.org/rfc/rfc4585) defines the AVPF profile that carries these messages):

- **REMB** (Receiver Estimated Maximum Bitrate) — Google proprietary RTCP extension (`draft-alvestrand-rmcat-remb`); receiver estimates available bandwidth and signals it to the sender via a RTCP APP packet.
- **Transport-CC** (Transport-wide Congestion Control) — receiver reports per-packet arrival timestamps for all packets in a single RTCP feedback message; sender runs the GCC (Google Congestion Control) algorithm. The RTCP feedback message format is specified in `draft-holmer-rmcat-transport-wide-cc-extensions`.
- **PLI** (Picture Loss Indication, [RFC 4585](https://www.rfc-editor.org/rfc/rfc4585) §6.3.1) — RTCP PSFB message requesting an immediate IDR keyframe upon loss detection.
- **FIR** (Full Intra Request, [RFC 5104](https://www.rfc-editor.org/rfc/rfc5104)) — similar to PLI but used in multipoint conferences to synchronise all receivers.

Both libwebrtc and GStreamer webrtcbin implement REMB and Transport-CC. PJSIP's video subsystem supports PLI and FIR for keyframe recovery.

---

## 12. Security Model

### SIPS/TLS Signaling Security

Standard SIP runs on port 5060 over UDP or TCP without encryption — caller identity, routing headers, and SDP contents travel in plaintext. SIPS ([RFC 5630](https://www.rfc-editor.org/rfc/rfc5630.html)) mandates TLS on port 5061 for the entire signaling path to the next hop. Correct SIPS deployment requires TLS-capable proxies throughout the route set, `sips:` URIs in `Record-Route` headers, and mutual TLS (mTLS) to authenticate both client and server certificates.

SIPS secures **signaling only** — caller ID, routing, SDP parameters. Media requires independent protection via SRTP or ZRTP.

### SRTP (RFC 3711)

[SRTP](https://www.rfc-editor.org/rfc/rfc3711.html) encrypts and authenticates RTP payload with AES-128-CM (counter mode) and HMAC-SHA1. Key distribution uses one of three methods: SDES (deprecated, SDP `a=crypto:` lines, keys in plaintext SDP — must not be used without SIPS), DTLS-SRTP (RFC 5764, used in WebRTC), or ZRTP (RFC 6189, media-path DH).

libwebrtc cipher suite selection (`webrtc::CryptoOptions`):

```cpp
// webrtc/api/crypto/crypto_options.h
CryptoOptions options;
// Enabled by default:
options.srtp.enable_aes128_sha1_80 = true;   // SRTP_AES128_CM_HMAC_SHA1_80
// GCM suites (preferred when both sides support):
options.srtp.enable_gcm_crypto_suites = true; // SRTP_AEAD_AES_128_GCM, _256_GCM
// Audio-only shortened auth tag (not default):
options.srtp.enable_aes128_sha1_32 = false;  // SRTP_AES128_CM_HMAC_SHA1_32
```

`SrtpSession` wraps libsrtp: `ProtectRtp()`, `ProtectRtcp()`, `UnprotectRtp()`, `UnprotectRtcp()`. [Source: webrtc.googlesource.com SRTP documentation](https://webrtc.googlesource.com/src/+/refs/heads/lkgr/pc/g3doc/srtp.md).

### DTLS-SRTP Key Exchange (RFC 5764)

The DTLS handshake runs on the ICE-established UDP path. Fingerprints are exchanged in SDP to authenticate the DTLS certificate:

```
Offerer (a=setup:actpass)            Answerer (a=setup:active)
  │                                       │
  │◀════ DTLS ClientHello ════════════════│  (answerer initiates as DTLS client)
  │═════ ServerHello + Certificate ══════▶│
  │◀════ Certificate + CertVerify ════════│  verify fingerprint vs SDP a=fingerprint
  │═════ Finished ════════════════════════▶│
  │                                       │
  Both export keying material:
  label = "EXTRACTOR-dtls_srtp" (RFC 5705)
  → client_write_key | server_write_key | client_salt | server_salt
  Configure libsrtp send/recv sessions
  │                                       │
  │══════════════ SRTP ══════════════════▶│
```

`DtlsSrtpTransport::ExtractParams()` calls `SSL_export_keying_material()` (OpenSSL/BoringSSL) with the RFC 5705 label and configures `SrtpTransport` with the extracted keys. SDES is explicitly unsupported in modern WebRTC implementations — `a=crypto:` lines in SDP are rejected.

### ZRTP (RFC 6189)

[ZRTP](https://datatracker.ietf.org/doc/rfc6189/) performs Diffie-Hellman key agreement over the same UDP port as RTP, requiring no signaling protocol changes. The DH exchange is multiplexed as special ZRTP packets distinguishable by magic number in the first byte. Key advantages:

- **No signaling trust required** — keys never touch the SIP proxy
- **Short Authentication String** — four-character SAS allows users to detect man-in-the-middle by verbally comparing
- **Key continuity** — ZRTP caches verified peer keys; subsequent calls auto-verify without SAS comparison

liblinphone implements ZRTP via the [Bzrtp library](https://github.com/BelledonneCommunications/bzrtp). PJSIP also includes a ZRTP module. The SAS verification pattern:

```
Call established → ZRTP DH completes → SAS generated
Alice reads: "r3dn"   Bob reads: "r3dn"   ✓ match → no MiTM
Alice reads: "r3dn"   Bob reads: "x7qp"   ✗ mismatch → attack detected
```

### Key Exchange Trade-offs

| Method | Signaling dependency | Server trust | Forward secrecy | Deployment complexity |
|--------|---------------------|--------------|-----------------|----------------------|
| SDES | SDP (SIPS required) | Proxy sees keys | No | Low (legacy PBX) |
| DTLS-SRTP | SDP fingerprint | None beyond cert verification | Per-session | Medium (WebRTC standard) |
| ZRTP | None | None | Yes (DH + key continuity) | Low for endpoints; no server changes |

---

## 13. Testing and Debugging

### SIPp Load Tester

[SIPp](https://sipp.readthedocs.io/en/latest/scenarios/ownscenarios.html) is a Linux command-line SIP load generator. Scenarios are XML files defining the call flow:

```xml
<!-- Basic UAC scenario: INVITE → 100/180 → 200 → ACK → BYE -->
<!-- Source: sipp.readthedocs.io/en/latest/scenarios/ownscenarios.html -->
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE scenario SYSTEM "sipp.dtd">
<scenario name="Basic UAC">
  <send retrans="500">
    <![CDATA[
INVITE sip:[service]@[remote_ip]:[remote_port] SIP/2.0
Via: SIP/2.0/[transport] [local_ip]:[local_port];branch=[branch]
From: sipp <sip:sipp@[local_ip]:[local_port]>;tag=[call_number]
To: sut <sip:[service]@[remote_ip]:[remote_port]>
Call-ID: [call_id]
CSeq: 1 INVITE
Contact: sip:sipp@[local_ip]:[local_port]
Max-Forwards: 70
Content-Type: application/sdp
Content-Length: [len]

v=0
o=user1 53655765 2353687637 IN IP4 [local_ip]
s=-
c=IN IP4 [local_ip]
t=0 0
m=audio [media_port] RTP/AVP 0
a=rtpmap:0 PCMU/8000
    ]]>
  </send>
  <recv response="100" optional="true" />
  <recv response="180" optional="true" />
  <recv response="200" />
  <send><![CDATA[ACK ...]]></send>
  <pause milliseconds="5000"/>
  <send><![CDATA[BYE ...]]></send>
  <recv response="200" />
</scenario>
```

Run: `sipp -sn uac -r 10 -d 5000 192.168.1.100:5060` (10 calls/sec, 5 s pause). The `-sfrx` option enables alternate recv scenarios for mixed UAC/UAS testing. SIPp can generate RTP streams alongside SIP for end-to-end media testing.

### Wireshark SIP/RTP Analysis

Wireshark dissects SIP out of the box. Key workflows:

```bash
# Capture SIP and RTP on interface eth0
tshark -i eth0 -w voip-capture.pcap -f "port 5060 or portrange 16384-32767"

# Display filter for SIP only
# In Wireshark UI: display filter = "sip"
# Statistics → VoIP Calls → call flow graph (ladder diagram)

# RTP stream analysis
# Filter: rtp
# Statistics → RTP → RTP Streams → Analyze: jitter, loss, gaps

# SIP/TLS (port 5061) — requires exporting TLS session keys
# Set SSLKEYLOGFILE=/tmp/keys.log before starting softphone
```

The [Wireshark SIP wiki](https://wiki.wireshark.org/SIP) documents TCP segment reassembly for large SDP bodies, SIP/TLS decryption with session key export, and the VoIP call flow diagram generator that builds ladder diagrams from captures.

### rtptools

[rtptools 1.22](https://github.com/irt/rtptools) (Columbia University) provides four utilities:

```bash
# Capture RTP stream to binary dump
rtpdump -F dump -o capture.rtp 0.0.0.0/5004

# Replay at original timing
rtpplay -T -f capture.rtp 192.168.1.100/5004

# Generate RTP from text description
rtpsend -s 192.168.1.100/5004 << 'EOF'
0 len=172 ssrc=0x1 ts=0 m=1 pt=0 seq=1
10 len=172 ssrc=0x1 ts=160 m=0 pt=0 seq=2
EOF

# Unicast ↔ multicast bridge
rtptrans 224.0.1.1/5004 192.168.1.100/5004
```

rtpdump output formats: `-F dump` (binary, rtpplay-compatible), `-F ascii` (human-readable), `-F payload` (raw payload bytes only), `-F rtcp` (RTCP only).

### pjsua CLI Smoke Tests

PJSIP ships `pjsua` as a reference CLI softphone. Basic smoke test sequence:

```bash
# Register and wait for INVITE
pjsua \
  --id sip:alice@example.com \
  --registrar sip:example.com \
  --realm example.com \
  --username alice \
  --password secret \
  --no-tcp \
  --log-level 4

# Interactive CLI commands:
# m → make call (prompts for destination URI)
# h → hangup active call
# l → list active calls
# cc → conference connect two calls
# q → quit
```

For automated testing, combine with SIPp in UAS mode: SIPp answers calls and plays back audio; pjsua places calls and verifies audio round-trip via `--auto-answer 200 --duration 10 --null-audio`.

---

## Integrations

**Ch38 — PipeWire and the Video Session Layer**: PipeWire is the default audio backend for all Linux VoIP softphones. SIP softphones using PulseAudio API (PJSIP, liblinphone) transparently route through `pipewire-pulse`. Native PipeWire streams achieve lower latency (configurable quantum as low as 64 frames = 1.3 ms at 48 kHz) compared to PulseAudio. `pw-loopback` enables virtual microphone creation and call monitoring without kernel modules.

**Ch38b — ALSA and the Linux Audio Subsystem**: Legacy VoIP applications (asterisk-based IVR, embedded softphones) access audio directly via ALSA PCM ioctls. PipeWire's ALSA plugin (`libasound_module_pcm_pipewire.so`) transparently routes these applications through PipeWire's session management, but the ALSA plugin path adds one buffer copy and one quantum of latency compared to native PipeWire.

**Ch214 — Bluetooth Audio and HFP**: Bluetooth headsets used for VoIP calls rely on HFP (Hands-Free Profile) for bidirectional audio, which PipeWire exposes as a capture/playback device pair via `bluez-monitor`. VoIP applications select the headset device through the same audio device enumeration API used for wired devices; PipeWire's `bluez5` SPA plugin negotiates HFP codec (CVSD or mSBC at 16 kHz wideband) with the headset.

**Ch60 — V4L2 and the Camera Pipeline**: Video calls capture frames from V4L2 devices (`/dev/video0`). PipeWire's `spa-v4l2` plugin exposes V4L2 cameras as PipeWire video sources. For direct GStreamer pipelines (`v4l2src`), VA-API hardware encode (`vah264enc`) replaces software x264 for low-latency H.264 video, reducing encode latency from ~50 ms to under 10 ms on Intel and AMD integrated GPUs.

**Ch218 — SRT/NDI and Broadcast Streaming**: SRT (Secure Reliable Transport) targets one-way broadcast streaming with ARQ-based reliability, optimised for 200–500 ms one-way latency. WebRTC/SIP VoIP targets interactive bidirectional calls at 50–150 ms round-trip latency using ICE/DTLS rather than ARQ. The two protocols serve fundamentally different latency/reliability trade-offs; hybrid architectures (SRT for broadcast distribution, WebRTC for bidirectional Q&A) are emerging in live event production.
