# Codecs & Media Reference

## Table of Contents

1. [Codec Deep Dive](#codec-deep-dive)
2. [Bandwidth Calculations](#bandwidth-calculations)
3. [RTP Protocol](#rtp-protocol)
4. [RTCP](#rtcp)
5. [DTMF Handling](#dtmf-handling)
6. [Voice Quality Metrics](#voice-quality-metrics)
7. [Transcoding](#transcoding)
8. [Call Recording](#call-recording)

---

## Codec Deep Dive

### G.711 (PCMU / PCMA)

The workhorse codec. Two variants:
- **μ-law (PCMU, PT 0)** — North American standard
- **A-law (PCMA, PT 8)** — Australian/European standard. **Always use A-law for AU deployments.**

Properties:
- Uncompressed PCM, 8kHz sampling, 8 bits/sample = 64 kbps
- No compression delay — lowest latency of any codec
- Excellent quality (MOS ~4.4) but high bandwidth
- No licensing requirements
- Supports transcoding without quality loss between μ-law and A-law

When to use: LAN calls, trunk connections with sufficient bandwidth, carrier interconnects
where quality is paramount.

### G.729 (and G.729a/G.729ab)

Low-bandwidth codec, widely used for WAN links.

Properties:
- 8 kbps bitrate, 10ms frame size
- MOS ~3.9 — perceptibly lower quality than G.711 but acceptable
- G.729a is a simplified (lower complexity) version, slightly lower quality
- G.729ab adds Voice Activity Detection (VAD) / Comfort Noise Generation (CNG)
- **Licensing**: Historically required patent licenses. Most patents expired 2017. Some
  implementations (especially hardware DSPs) may still require licenses. Asterisk's open-source
  G.729 codec (codec_g729 from Digium) was licensed per-channel. Check your platform.
- Each transcoding operation degrades quality noticeably — avoid tandem transcoding

When to use: WAN links with limited bandwidth, mobile workers over poor connections.

### G.722

HD Voice / Wideband codec.

Properties:
- 64 kbps bitrate, same as G.711
- 16 kHz sampling (wideband) — captures 50-7000 Hz vs G.711's 300-3400 Hz
- Noticeably clearer, more natural voice quality (MOS ~4.5)
- **Clock rate bug**: SDP specifies 8000 Hz for G.722 due to a historical error in RFC 3551.
  The actual sampling is 16 kHz. Every implementation works around this — just be aware.
- No licensing requirements
- PT 9 in SDP

When to use: Internal calls, executive lines, any call where both endpoints support it and
bandwidth isn't constrained. Many modern IP phones support G.722 by default.

### Opus

Modern, highly flexible codec. The gold standard for WebRTC.

Properties:
- Variable bitrate: 6 kbps to 510 kbps
- Supports narrowband (8 kHz) through fullband (48 kHz)
- Built-in forward error correction (FEC) — handles packet loss gracefully
- Dynamic bitrate adaptation based on network conditions
- Open source, royalty-free
- Dynamic payload type (typically 111 or 116 in SDP)
- Frame sizes: 2.5ms to 60ms
- Stereo support

When to use: WebRTC applications, softphones, any modern deployment where both ends support it.
Not widely supported on traditional hardware phones (Yealink/Grandstream support is growing).

### iLBC (Internet Low Bitrate Codec)

Properties:
- Two modes: 13.33 kbps (30ms frame) or 15.2 kbps (20ms frame)
- Specifically designed for packet loss resilience
- MOS ~3.8 — acceptable quality
- Each frame is independently decodable (no inter-frame dependencies)
- Open source

When to use: Networks with known packet loss problems where G.729 isn't available.

### AMR / AMR-WB (EVS)

Mobile network codecs, occasionally seen in carrier interconnects.

- AMR: 4.75-12.2 kbps, narrowband, GSM standard
- AMR-WB (G.722.2): 6.6-23.85 kbps, wideband, used in VoLTE
- EVS: Next-gen mobile codec, 5.9-128 kbps, super-wideband

Rarely used in PBX contexts but may appear in SIP trunks from mobile carriers.

---

## Bandwidth Calculations

### Formula

```
Total bandwidth per call = (codec bitrate + IP/UDP/RTP overhead) / (ptime / frame_size)
```

Simplified for 20ms ptime with Ethernet:

```
Overhead per packet:
  - IP header:  20 bytes
  - UDP header:  8 bytes
  - RTP header: 12 bytes
  - Ethernet:   18 bytes (14 header + 4 FCS)
  Total overhead: 58 bytes per packet

G.711 @ 20ms ptime:
  Payload: 160 bytes (64000 bps * 0.020s / 8)
  Total packet: 218 bytes = 1744 bits
  Packets per second: 50 (1000ms / 20ms)
  Total: 87.2 kbps per direction

G.729 @ 20ms ptime:
  Payload: 20 bytes (8000 bps * 0.020s / 8)
  Total packet: 78 bytes = 624 bits
  Packets per second: 50
  Total: 31.2 kbps per direction
```

### Capacity Planning

For concurrent call capacity, multiply by 2 (bidirectional) and add headroom:

| Codec | Per Call (both dirs) | 10 Calls | 50 Calls | 100 Calls |
|-------|---------------------|----------|----------|-----------|
| G.711 | ~175 kbps | 1.75 Mbps | 8.75 Mbps | 17.5 Mbps |
| G.729 | ~63 kbps | 630 kbps | 3.15 Mbps | 6.3 Mbps |
| G.722 | ~175 kbps | 1.75 Mbps | 8.75 Mbps | 17.5 Mbps |

**Rule of thumb**: Add 20% overhead for signaling, RTCP, ARP, and protocol overhead. For
carrier-grade planning, use 100 kbps per G.711 call as a safe estimate.

---

## RTP Protocol

### RTP Header Format

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|V=2|P|X|  CC   |M|     PT      |       sequence number         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           timestamp                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             SSRC                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

Key fields:
- **PT (Payload Type)** — Identifies the codec (0=PCMU, 8=PCMA, etc.)
- **Sequence Number** — Increments per packet; used to detect loss and reorder
- **Timestamp** — Media clock; used for playout timing and jitter calculation
- **SSRC** — Synchronization Source; random ID for each stream
- **Marker bit (M)** — Set on the first packet after silence (when VAD is used)

### RTP Port Allocation

- Typically use an even port for RTP, next odd port for RTCP (e.g., RTP=10000, RTCP=10001)
- Common ranges: 10000-20000 (Asterisk default), 16384-32767 (FreeSWITCH)
- For firewall rules, you need the FULL range open for UDP
- Reduce the range if you know your max concurrent calls:
  `ports_needed = max_concurrent_calls * 4` (2 per call direction, RTP+RTCP each)

### Symmetric RTP / Comedia

Normally, each side sends RTP to the IP:port specified in the other side's SDP. With symmetric
RTP (comedia), the endpoint sends RTP to wherever it receives RTP from. This solves NAT issues
for media because it uses the observed source address rather than the SDP address.

Asterisk: `directmedia=no` + `nat=force_rport,comedia`
FreeSWITCH: Enabled by default in most cases

---

## RTCP

RTCP runs alongside RTP to provide quality statistics.

### Key RTCP Reports

- **Sender Report (SR)** — Sent by the media sender. Contains NTP timestamp, packet count,
  octet count.
- **Receiver Report (RR)** — Sent by the media receiver. Contains:
  - Fraction lost (since last RR)
  - Cumulative packets lost
  - Extended highest sequence number received
  - Interarrival jitter
  - Last SR timestamp and delay since last SR

### RTCP-XR (Extended Reports)

RFC 3611 — Provides more detailed quality metrics:
- VoIP Metrics Block: MOS scores, R-factor, jitter buffer stats, burst/gap loss
- Supported by many modern phones and PBXes
- Useful for proactive quality monitoring

---

## DTMF Handling

Three methods exist; mixing them up is a common source of "DTMF not working" issues.

### Method 1: RFC 2833 / RFC 4733 (telephone-event) — PREFERRED

DTMF digits sent as named RTP events using payload type 101 (by convention).

SDP negotiation:
```
a=rtpmap:101 telephone-event/8000
a=fmtp:101 0-16
```

The `0-16` means events 0-9 (digits), 10 (*), 11 (#), 12-15 (A-D), 16 (flash).

Each DTMF event is sent as 3+ RTP packets:
1. Start packet (marker bit set, duration starts at 0)
2. Continuation packets (duration increases)
3. End packet (End flag set, sent 3 times for redundancy)

**This is the standard for nearly all modern VoIP systems.** Use this unless you have a
specific reason not to.

### Method 2: SIP INFO

DTMF sent as SIP INFO messages with `application/dtmf-relay` content type.

```
INFO sip:user@host SIP/2.0
Content-Type: application/dtmf-relay

Signal=5
Duration=160
```

Pros: Works through transcoders that strip RTP events. Out of band.
Cons: Can be delayed by SIP proxy processing. Not real-time.

### Method 3: In-band (Inband Audio)

DTMF tones encoded directly in the audio stream as actual audio frequencies.

Pros: Works with any codec... theoretically.
Cons: Compressed codecs (G.729, Opus) mangle the tone frequencies, making detection unreliable.
**Never use in-band with compressed codecs.**

### Troubleshooting DTMF

| Symptom | Likely Cause |
|---------|-------------|
| DTMF not detected at all | Method mismatch — one side sends RFC4733, other expects SIP INFO |
| DTMF detected but wrong digits | Codec/transcoder mangling in-band DTMF |
| DTMF sometimes works | Packet loss dropping RFC4733 events; or firewall blocking INFO |
| Double digits | End event not received; sender resends, receiver processes twice |
| IVR doesn't respond to DTMF | IVR expecting different DTMF method than what's being sent |

---

## Voice Quality Metrics

### MOS (Mean Opinion Score)

Scale of 1-5, where:
- 4.3-4.5: Excellent (toll quality, G.711/G.722)
- 4.0-4.3: Good (most users satisfied)
- 3.6-4.0: Fair (some users unsatisfied)
- 3.1-3.6: Poor (many users unsatisfied)
- 1.0-3.1: Bad (nearly all users unsatisfied)

### R-Factor (E-Model)

ITU-T G.107. Scale of 0-100, maps to MOS:
- R > 90: Excellent (MOS > 4.3)
- R 80-90: Good (MOS 4.0-4.3)
- R 70-80: Fair (MOS 3.6-4.0)
- R 60-70: Poor (MOS 3.1-3.6)
- R < 60: Bad

### Impairment Factors

| Factor | Acceptable | Degraded | Unacceptable |
|--------|-----------|----------|-------------|
| Latency (one-way) | <150ms | 150-300ms | >300ms |
| Jitter | <30ms | 30-50ms | >50ms |
| Packet Loss | <1% | 1-3% | >3% |

### Jitter Buffer

The jitter buffer at the receiver smooths out variable packet arrival times.

- **Static jitter buffer** — Fixed size (e.g., 40ms). Simple but may underperform.
- **Adaptive jitter buffer** — Dynamically adjusts based on observed jitter. Preferred.
- Larger buffer = less packet loss but more latency
- Typical: 20-60ms for LAN, 40-120ms for WAN

---

## Transcoding

Transcoding converts between codecs when two endpoints don't share a common codec.

### Performance Impact

- Each transcode operation adds ~2-5ms delay
- CPU intensive — a single-core system can typically handle 30-100 concurrent G.711↔G.729
  transcodes depending on hardware
- Quality degrades with each transcode step — avoid tandem transcoding
- Hardware DSP cards (Sangoma, Digium) offload transcoding from CPU

### Avoiding Transcoding

1. **Negotiate the same codec end-to-end** — Configure trunks and endpoints to prefer the
   same codec
2. **Direct media / reinvite** — Let endpoints send RTP directly to each other, bypassing
   the PBX media path entirely. Asterisk: `directmedia=yes`. Only works when both endpoints
   can reach each other directly (same LAN, or both on public IPs).
3. **Pass-through mode** — Some SBCs can pass media without touching it. Only works if both
   sides support a common codec.

---

## Call Recording

### Methods

1. **In-PBX recording** — The PBX mixes and records the audio. Simplest approach.
   - Asterisk: `MixMonitor()` application in the dialplan
   - FreeSWITCH: `record_session` or `uuid_record`

2. **Port mirroring / SPAN** — Mirror the switch port carrying RTP traffic to a recording
   server. Records without touching the PBX. Requires a tool to reassemble RTP into audio
   (e.g., `oreka`, `VoIPmonitor`).

3. **SIP SIPREC (RFC 7866)** — SIP-based recording. The PBX forks media to a recording server
   via a separate SIP session. Enterprise-grade, supported by some SBCs and PBXes.

### Storage Calculations

```
G.711 uncompressed: 64 kbps = 8 KB/s = 480 KB/min = 28.8 MB/hour per channel
Stereo (both legs separate): double the above
Compressed (e.g., WAV with GSM compression): ~5-10x smaller
MP3/Opus compressed: ~10-20x smaller than raw PCM
```

For a call centre with 50 concurrent agents recording 6 hours/day:
- G.711 WAV: 50 × 6h × 28.8 MB/h × 2 (stereo) = ~17 GB/day
- Opus compressed: ~1-2 GB/day

### Australian Legal Requirements for Recording

- **Federal law** (Telecommunications Interception and Access Act 1979) requires at least one
  party to consent in most cases. Some states (e.g., QLD) require all parties to consent.
- **Best practice**: Always announce recording ("this call may be recorded for quality and
  training purposes") regardless of jurisdiction.
- PCI-DSS compliance: Pause recording during credit card number entry. Most modern recording
  solutions support pause/resume via API or DTMF.
