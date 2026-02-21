# SIP Protocol Reference

## Table of Contents

1. [SIP Message Structure](#sip-message-structure)
2. [Key Headers](#key-headers)
3. [Authentication](#authentication)
4. [SDP and Codec Negotiation](#sdp-and-codec-negotiation)
5. [NAT Traversal](#nat-traversal)
6. [SIP Over TLS/WSS](#sip-over-tls)
7. [Common Call Flows](#common-call-flows)
8. [SIP Timers](#sip-timers)
9. [SIP Extensions and RFCs](#key-rfcs)

---

## SIP Message Structure

SIP messages are text-based (like HTTP). Two types: Requests and Responses.

### Request Format

```
METHOD sip:user@domain SIP/2.0
Via: SIP/2.0/UDP 10.0.0.1:5060;branch=z9hG4bK776asdhds
Max-Forwards: 70
To: <sip:user@domain>
From: <sip:caller@domain>;tag=1928301774
Call-ID: a84b4c76e66710@10.0.0.1
CSeq: 314159 METHOD
Contact: <sip:caller@10.0.0.1:5060>
Content-Type: application/sdp
Content-Length: [length]

[SDP Body]
```

### SIP Methods

| Method | Purpose | RFC |
|--------|---------|-----|
| INVITE | Initiate a session (call) | 3261 |
| ACK | Confirm INVITE was received | 3261 |
| BYE | Terminate a session | 3261 |
| CANCEL | Cancel a pending INVITE | 3261 |
| REGISTER | Register location with registrar | 3261 |
| OPTIONS | Query capabilities (often used as keepalive) | 3261 |
| INFO | Mid-dialog info (DTMF, etc.) | 6086 |
| REFER | Transfer a call | 3515 |
| NOTIFY | Event notification | 3265 |
| SUBSCRIBE | Subscribe to events | 3265 |
| UPDATE | Modify session without re-INVITE | 3311 |
| MESSAGE | Instant messaging | 3428 |
| PRACK | Provisional response ACK | 3262 |
| PUBLISH | Publish event state | 3903 |

---

## Key Headers

### Mandatory Headers (every request)

- **Via** — Records the transport path. Each proxy adds a Via. The `branch` parameter is the
  transaction ID (must start with `z9hG4bK` per RFC 3261). `rport` parameter requests the
  server to send responses back to the source port (critical for NAT).
- **To** — The logical recipient. Gets a `tag` parameter in responses (identifies the dialog).
- **From** — The logical sender. Always has a `tag` parameter.
- **Call-ID** — Unique identifier for the dialog. Same for all messages in a dialog.
- **CSeq** — Sequence number + method. Increments for new transactions within a dialog.
- **Max-Forwards** — Hop counter (starts at 70, decremented by each proxy).
- **Contact** — Where to send subsequent requests directly (bypassing proxies). This is where
  NAT problems often surface — if the Contact contains a private IP, the remote end can't
  reach it.

### Important Optional Headers

- **Record-Route** — Proxy inserts this to stay in the signaling path for the whole dialog.
- **Route** — Pre-loaded route set (from Record-Route of initial transaction).
- **Allow** — Lists methods the UA supports.
- **Supported** — Lists SIP extensions supported (e.g., `timer`, `replaces`, `100rel`).
- **Require** — Lists extensions that MUST be supported; causes 420 if not.
- **P-Asserted-Identity (PAI)** — Trusted identity (carrier-grade caller ID).
- **Remote-Party-ID (RPID)** — Caller ID, mostly legacy but still common in Asterisk.
- **Privacy** — Controls identity privacy (values: `none`, `id`, `header`, `session`).
- **Diversion** — Indicates call forwarding history.
- **User-Agent** — Identifies the SIP stack/device (useful for debugging).
- **X-headers** — Custom headers (e.g., `X-Asterisk-Account-Code`). Non-standard but widely used.

### Caller ID Headers — Priority and Usage

In carrier interconnects, caller ID handling can be confusing. Here's the priority order most
systems use:

1. **P-Asserted-Identity (PAI)** — Trusted network identity. Only meaningful in a trusted
   domain (between carriers/proxies). This is the "real" caller ID.
2. **Remote-Party-ID (RPID)** — Older mechanism, same purpose. Asterisk uses this by default
   unless configured otherwise (`sendrpid=pai`).
3. **From header** — The user-asserted identity. Can be spoofed. Should NOT be trusted for
   billing or routing decisions in a carrier context.

For Australian carrier interconnects, PAI is the standard. Configure your PBX to send PAI
with the allocated CLI, not the extension's caller ID.

---

## Authentication

### Digest Authentication Flow

```
UAC                        Registrar/Proxy
 |--- REGISTER/INVITE --------->|
 |<-- 401/407 (with nonce) -----|     401 = from registrar
 |--- REGISTER/INVITE --------->|     407 = from proxy
 |   (with Authorization)       |
 |<-- 200 OK -------------------|
```

The `WWW-Authenticate` (401) or `Proxy-Authenticate` (407) header contains:
- `realm` — Authentication domain
- `nonce` — Server-generated one-time value
- `algorithm` — Usually MD5
- `qop` — Quality of protection (auth, auth-int)

The response includes an `Authorization` or `Proxy-Authorization` header with:
- `username` — Auth username (often different from SIP URI user)
- `realm` — Must match
- `nonce` — Must match
- `uri` — The request URI
- `response` — MD5 hash of username:realm:password:nonce:etc.

### IP-Based Authentication

No registration needed. The SIP provider whitelists your public IP. Simpler but less flexible.
Used commonly for SIP trunks where the PBX has a static IP.

**Pros**: No registration state to maintain, no registration keepalives, simpler config.
**Cons**: Requires static IP, less granular, can't use from dynamic IP or behind NAT easily.

---

## SDP and Codec Negotiation

SDP (Session Description Protocol) is carried in the body of INVITE and 200 OK messages.
It describes the media session parameters.

### SDP Structure

```
v=0
o=- 20518 0 IN IP4 203.0.113.1
s=-
c=IN IP4 203.0.113.1
t=0 0
m=audio 49170 RTP/AVP 0 8 18 101
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:18 G729/8000
a=rtpmap:101 telephone-event/8000
a=fmtp:101 0-16
a=ptime:20
a=sendrecv
```

### Key SDP Fields

- **o=** — Origin. Contains the session ID and the IP address of the sender.
- **c=** — Connection data. The IP where media should be sent. THIS IS WHERE NAT BREAKS THINGS
  — if this contains a private IP (10.x, 172.16-31.x, 192.168.x), the remote end can't send
  media to it.
- **m=** — Media line. Format: `m=audio [port] RTP/AVP [payload types]`. The port is where
  this endpoint will receive RTP. Payload types reference the codec list below.
- **a=rtpmap:** — Maps payload type numbers to codec names.
- **a=fmtp:** — Format parameters. For telephone-event (DTMF), `0-16` means DTMF digits 0-9,
  *, #, and events A-D.
- **a=ptime:** — Packetization time in ms. 20ms is standard. Higher = more latency but less
  overhead.
- **a=sendrecv/sendonly/recvonly/inactive** — Media direction.

### Codec Payload Types

| PT | Codec | Clock Rate | Notes |
|----|-------|-----------|-------|
| 0 | PCMU (G.711 μ-law) | 8000 | North America standard |
| 8 | PCMA (G.711 A-law) | 8000 | AU/EU standard — USE THIS FOR AU |
| 9 | G.722 | 8000 | HD Voice (actually 16kHz wideband, clock rate is a known bug in RFC) |
| 18 | G.729 | 8000 | Low bandwidth, may need license |
| 101-127 | Dynamic | Varies | telephone-event (DTMF), Opus, etc. |

### Offer/Answer Model

1. **Caller** sends INVITE with SDP offer (their supported codecs, in preference order)
2. **Callee** responds with 200 OK containing SDP answer (selected codec from the offer)
3. The first mutually supported codec wins (callee picks from caller's offer)

If no common codec exists, callee responds `488 Not Acceptable Here`.

### Late vs Early Media

- **Early media** — Media flows before the call is answered (183 Session Progress with SDP).
  Used for ringback tones, announcements, IVR prompts before answer. Common in carrier
  interconnects.
- **Late negotiation** — INVITE sent without SDP; callee's 200 OK contains the offer, caller's
  ACK contains the answer. Less common but some providers use this.

---

## NAT Traversal

NAT is the #1 cause of VoIP problems. Here's why and what to do about it.

### The Problem

SIP puts IP addresses in two places:
1. **SIP headers** — Via, Contact (used for signaling path)
2. **SDP body** — c= line and m= port (used for media path)

When a phone behind NAT sends a REGISTER or INVITE, these fields contain the phone's private
IP (e.g., 192.168.1.100). The SIP server can't send responses or media to a private IP.

### Solutions (in order of preference)

1. **SIP-aware SBC/proxy at the edge** — Best solution for carrier-grade. The SBC terminates
   SIP on both sides and rewrites headers/SDP. Kamailio with `nathelper` and `rtpproxy/rtpengine`
   does this.

2. **STUN** — Phone discovers its public IP via STUN server, puts that in SIP/SDP. Works for
   most consumer NAT. Fails with symmetric NAT (common on carrier-grade NAT/CGNAT).

3. **TURN** — Relay server for media. Works through any NAT type but adds latency and requires
   relay infrastructure. Used by WebRTC as fallback.

4. **rport (RFC 3581)** — Server sends SIP responses back to the source IP:port it received
   the request from, regardless of what Via says. Most SIP servers support this. Solves
   signaling NAT, but NOT media NAT.

5. **Far-end NAT traversal** — The SIP server/PBX detects NAT (compares Via with source IP)
   and rewrites Contact/SDP to use the observed public IP. Asterisk: `nat=force_rport,comedia`.
   FreeSWITCH: similar with auto-nat detection.

6. **SIP ALG** — Application Layer Gateway in the NAT router. Theoretically rewrites SIP
   headers on the fly. In practice, **almost always breaks things**. DISABLE SIP ALG on every
   router. This is the first thing to check when debugging NAT issues.

### Detecting NAT

Compare the source IP of the packet with the IP in the Via header. If they differ, the UA is
behind NAT. Most SIP proxies/PBXes can do this automatically.

In Asterisk, the `nat=` option controls behavior:
- `nat=no` — Trust the addresses in SIP messages (use only on LAN)
- `nat=force_rport,comedia` — Always use the observed source IP for signaling and media
- `nat=auto_force_rport,auto_comedia` — Detect and apply automatically

### Common NAT Symptoms

| Symptom | Likely Cause |
|---------|-------------|
| One-way audio | SDP contains private IP; remote end sends media to unreachable address |
| No audio both ways | Both sides behind NAT, neither can reach the other |
| Registration works, calls fail | REGISTER uses small packets that traverse NAT; INVITE/media uses different ports |
| Calls drop after 30s | NAT binding timeout; SIP keepalives not configured |
| Calls work outbound but not inbound | Firewall not forwarding SIP/RTP ports to PBX |

---

## SIP Over TLS

### Transport Options

| Transport | Port | Encryption | Use Case |
|-----------|------|-----------|----------|
| UDP | 5060 | None | LAN, legacy |
| TCP | 5060 | None | Large messages, reliable transport |
| TLS | 5061 | Signaling only | Standard secure signaling |
| WSS | 443/8089 | Full | WebRTC softphones |

TLS encrypts SIP signaling but NOT media. For encrypted media, you also need SRTP.

### SRTP (Secure RTP)

- Encrypts the RTP media stream using AES
- Key exchange via SDP (`a=crypto:` line) or DTLS-SRTP (WebRTC standard)
- In SDP, the media line changes from `RTP/AVP` to `RTP/SAVP` (or `RTP/SAVPF` for WebRTC)
- Both sides must support it — if one side doesn't, the call either falls back to RTP or fails

---

## Common Call Flows

### Registration

```
Phone                    Registrar
  |--- REGISTER ----------->|
  |<-- 401 Unauthorized ----|  (with WWW-Authenticate nonce)
  |--- REGISTER ----------->|  (with Authorization digest)
  |<-- 200 OK --------------|  (with Expires: 3600)
  |                          |
  | ... (re-register before expiry) ...
  |--- REGISTER ----------->|
  |<-- 200 OK --------------|
```

Registration expiry defaults to 3600s (1 hour). For NAT, shorter intervals (60-120s) keep
the NAT binding alive, or use OPTIONS keepalives instead.

### Attended Transfer

```
A calls B (active call)
A puts B on hold (re-INVITE with sendonly)
A calls C (new call)
A transfers B to C:
  A --- REFER (Refer-To: C) ---> B
  B --- INVITE (Replaces: A-C dialog) ---> C
  B <-- 200 OK --- C
  A receives NOTIFY (200 OK, transfer succeeded)
  A hangs up
```

### Blind Transfer

```
A calls B (active call)
A sends REFER to B:
  A --- REFER (Refer-To: C) ---> B
  B --- INVITE ---> C
  A receives NOTIFY (transfer status)
  A hangs up
  B <-- 200 OK --- C
```

---

## SIP Timers

| Timer | Default | Purpose |
|-------|---------|---------|
| T1 | 500ms | RTT estimate; base for retransmissions |
| T2 | 4s | Max retransmit interval for non-INVITE |
| Timer A | T1 | INVITE retransmit (doubles each time up to T2) |
| Timer B | 64*T1 (32s) | INVITE transaction timeout |
| Timer C | >3min | Proxy INVITE transaction timeout |
| Timer D | >32s (UDP), 0s (TCP) | Wait time after response to absorb retransmits |
| Timer F | 64*T1 (32s) | Non-INVITE transaction timeout |
| Session-Timer | 1800s | Session keep-alive (re-INVITE or UPDATE) |

If calls drop after exactly 32 seconds, Timer B is expiring — the INVITE never got a response.
Check connectivity to the remote end.

---

## Key RFCs

| RFC | Subject |
|-----|---------|
| 3261 | SIP: Session Initiation Protocol (core spec) |
| 3262 | Reliability of Provisional Responses (PRACK) |
| 3263 | Locating SIP Servers (DNS SRV, NAPTR) |
| 3264 | Offer/Answer Model with SDP |
| 3265 | SIP Events (SUBSCRIBE/NOTIFY) |
| 3311 | UPDATE Method |
| 3323 | Privacy Mechanism (P-Asserted-Identity) |
| 3515 | REFER Method (call transfer) |
| 3550 | SRTP |
| 3581 | rport extension (NAT help) |
| 4566 | SDP: Session Description Protocol |
| 4733 | RTP Payload for DTMF (telephone-event) |
| 5765 | SRTP Security Descriptions (a=crypto) |
| 6026 | Correct Transaction Handling for 2xx to INVITE |
| 7118 | SIP over WebSocket |
| 8location | SRTP with DTLS |
