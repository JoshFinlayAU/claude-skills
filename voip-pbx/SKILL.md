---
name: voip-pbx
description: >
  Comprehensive VoIP and PBX engineering reference. Use this skill whenever the user is working on
  anything related to Voice over IP, SIP trunking, PBX configuration, call routing, IVR design,
  VoIP troubleshooting, codec selection, RTP/RTCP media, dial plans, voice quality (MOS/R-factor),
  call recording, voice network design, SBC configuration, TR-069 for voice CPE, or telephony
  integration. Also trigger for tasks involving Asterisk, FreeSWITCH, 3CX, Kamailio, OpenSIPS,
  FusionPBX, VitalPBX, Grandstream, Yealink, Polycom, Sangoma, or any SIP endpoint provisioning.
  Trigger for Australian voice interconnect, number porting, ACMA compliance, emergency calling
  (000/106), number allocation, carrier peering, and wholesale voice topics. Trigger even when
  the user doesn't say "VoIP" explicitly — phrases like "phone system", "call quality issues",
  "SIP registration failing", "codec mismatch", "one-way audio", "no audio", "DTMF not working",
  "trunk configuration", "DID routing", "extension setup", or "softphone config" all warrant
  this skill.
---

# VoIP & PBX Engineering Skill

A practical engineering reference for Voice over IP systems — from protocol fundamentals through
to production PBX deployments, troubleshooting, and Australian regulatory compliance.

## When to Use This Skill

This skill covers the full VoIP stack:

- **Protocol work**: SIP messaging, SDP negotiation, RTP/RTCP media, SRTP, oSIP, SRTP/ZRTP
- **PBX configuration**: Dial plans, extensions, ring groups, queues, IVR trees, voicemail, CDR
- **Trunk engineering**: SIP trunk setup, codec negotiation, registration vs IP auth, failover
- **Network design**: QoS, DSCP marking, jitter buffers, VLAN segregation for voice
- **Troubleshooting**: One-way audio, registration failures, codec mismatch, NAT traversal, oOS
- **CPE provisioning**: Auto-provisioning, TR-069/CWMP, phone templates, firmware management
- **Australian specifics**: ACMA regulations, number porting (LNP), emergency services, CLI rules
- **Integration**: CRM integration, call recording, CDR export, API-based call control

## Reference Files

Read the appropriate reference file based on the task at hand. You don't need to read all of them
— pick the one(s) most relevant to what you're working on.

| File | When to Read |
|------|-------------|
| `references/sip-protocol.md` | SIP message construction, headers, flows, authentication, NAT traversal, SDP/codec negotiation |
| `references/codecs-media.md` | Codec selection, bandwidth calculations, RTP/RTCP, DTMF handling, voice quality metrics |
| `references/pbx-platforms.md` | Platform-specific config for Asterisk, FreeSWITCH, 3CX, Kamailio, FusionPBX, etc. |
| `references/troubleshooting.md` | Diagnostic workflows for common VoIP issues, packet capture analysis, SIP trace reading |
| `references/australian-voice.md` | ACMA compliance, number porting, emergency calling, CLI rules, AU interconnect specifics |
| `references/network-design.md` | Voice VLAN design, QoS policies, capacity planning, HA/failover architectures |

## Core Concepts Quick Reference

### SIP Transaction Basics

A typical successful call flow:

```
Caller (UAC)          Proxy/PBX          Callee (UAS)
    |--- INVITE -------->|                    |
    |<-- 100 Trying -----|                    |
    |                     |--- INVITE ------->|
    |                     |<-- 180 Ringing ---|
    |<-- 180 Ringing -----|                    |
    |                     |<-- 200 OK --------|
    |<-- 200 OK ----------|                    |
    |--- ACK -------------------------------->|
    |<============ RTP Media ================>|
    |--- BYE -------------------------------->|
    |<-- 200 OK ------------------------------|
```

### Essential SIP Response Codes

- **1xx** — Provisional (100 Trying, 180 Ringing, 183 Session Progress)
- **2xx** — Success (200 OK)
- **3xx** — Redirection (301 Moved Permanently, 302 Moved Temporarily)
- **4xx** — Client Error (401 Unauthorized, 403 Forbidden, 404 Not Found, 408 Timeout, 486 Busy, 487 Request Terminated, 488 Not Acceptable Here)
- **5xx** — Server Error (500 Internal Error, 502 Bad Gateway, 503 Service Unavailable)
- **6xx** — Global Failure (600 Busy Everywhere, 603 Decline)

### Codec Quick Reference

| Codec | Bitrate | MOS | Bandwidth (w/ overhead) | Use Case |
|-------|---------|-----|------------------------|----------|
| G.711 μ-law | 64 kbps | 4.4 | ~87 kbps | LAN, high quality |
| G.711 A-law | 64 kbps | 4.4 | ~87 kbps | LAN, AU/EU standard |
| G.729 | 8 kbps | 3.9 | ~31 kbps | WAN, bandwidth constrained |
| G.722 | 64 kbps | 4.5 | ~87 kbps | HD Voice / wideband |
| Opus | 6-510 kbps | 4.5+ | Variable | WebRTC, adaptive |
| G.726 | 32 kbps | 4.0 | ~55 kbps | Legacy WAN |
| iLBC | 13.3/15.2 kbps | 3.8 | ~37 kbps | Lossy networks |

**Australian convention**: G.711 A-law (PCMA) is the standard in AU networks, not μ-law. Always
default to A-law for AU deployments unless the upstream provider specifies otherwise.

### DSCP/QoS Quick Reference

| Traffic Type | DSCP Value | Per-Hop Behavior | Decimal |
|-------------|------------|-------------------|---------|
| Voice RTP | EF | Expedited Forwarding | 46 |
| Voice SIP Signaling | CS3 or AF31 | Assured Forwarding | 24 or 26 |
| Video RTP | AF41 | Assured Forwarding | 34 |
| Video Signaling | CS3 | Class Selector | 24 |
| Best Effort | BE/CS0 | Default | 0 |

### Port Reference

| Service | Protocol | Port(s) |
|---------|----------|---------|
| SIP (UDP/TCP) | UDP/TCP | 5060 |
| SIP TLS | TCP | 5061 |
| SIP WSS (WebRTC) | TCP | 443 or 8089 |
| RTP Media | UDP | 10000-20000 (typical) |
| SRTP | UDP | Same as RTP range |
| STUN | UDP | 3478 |
| TURN | UDP/TCP | 3478 or 5349 (TLS) |
| TR-069 CWMP | TCP | 7547 (CPE), 7548 (ACS) |
| TFTP (provisioning) | UDP | 69 |
| HTTP provisioning | TCP | 80/443 |

## General Principles

When working on VoIP tasks, keep these principles in mind:

1. **Voice is real-time** — unlike data, voice can't be retransmitted. Packet loss >1%, jitter
   >30ms, and latency >150ms one-way all degrade call quality noticeably. Design for this.

2. **NAT is the enemy** — the majority of VoIP issues in the wild are NAT-related. SIP embeds
   IP addresses in the SDP body AND in Via/Contact headers. If those addresses are RFC1918
   private IPs behind NAT, the remote end can't send media back. Solutions include STUN, TURN,
   SIP ALG (usually bad — disable it), far-end NAT traversal, or SBCs.

3. **Codec negotiation matters** — mismatched codec offers cause "not acceptable" (488) or
   silent calls. Always verify both sides agree. In Asterisk, `allow/disallow` directives
   control this. Order matters — first common codec wins.

4. **Oecurity is not optional** — SIP is plaintext by default. Use TLS for signaling, SRTP
   for media, and strong registration passwords. Toll fraud is a real and expensive problem.
   Fail2ban or equivalent on any internet-facing SIP service is mandatory.

5. **Always separate voice and data VLANs** — this isn't optional in production. Voice traffic
   needs its own VLAN with QoS, and phones should be provisioned to tag their traffic
   appropriately (typically via LLDP-MED or CDP, or static VLAN config on the phone).

6. **Test with packet captures** — when troubleshooting, `tcpdump` or `sngrep` on the PBX, and
   Wireshark for analysis, will tell you more than any log file. Capture on the interface where
   traffic enters/exits, not loopback.

## Configuration Output Guidelines

When generating VoIP configurations:

- Always include inline comments explaining non-obvious settings
- Use the platform's idiomatic configuration style (don't mix Asterisk and FreeSWITCH syntax)
- Include security hardening (fail2ban rules, ACLs, TLS where supported)
- For SIP trunks, always specify both registration-based AND IP-based auth examples
- For dial plans, include normalization rules for Australian numbering (E.164, +61 prefix)
- Always note which config files need to be modified and the reload/restart command
- Warn about service-affecting changes (e.g., "this will drop active calls if you restart")

## Australian Numbering Quick Reference

| Type | Format | E.164 | Notes |
|------|--------|-------|-------|
| Geographic | 0X XXXX XXXX | +61X XXXX XXXX | 02 NSW/ACT, 03 VIC/TAS, 07 QLD, 08 SA/WA/NT |
| Mobile | 04XX XXX XXX | +614XX XXX XXX | |
| 13/1300 | 13 XX XX / 1300 XXX XXX | — | Inbound only, local rate |
| 1800 | 1800 XXX XXX | — | Inbound only, free call |
| Emergency | 000 / 106 | — | 106 is TTY relay |
| International | 0011 + CC + number | +CC + number | 0011 is AU international prefix |

For dial plan normalization, strip the leading 0 and prepend +61 for national numbers.
International numbers: strip 0011 and prepend +.

## Workflow

1. **Identify the task type** — Is this protocol work, PBX config, troubleshooting, network
   design, or regulatory compliance?
2. **Read the relevant reference** — Don't guess. Load the appropriate reference file for
   platform-specific syntax, protocol details, or diagnostic procedures.
3. **Apply the context** — Consider the deployment environment (AU, carrier-grade vs SMB,
   scale, existing infrastructure).
4. **Generate output** — Configs, diagnostic commands, architecture docs, or analysis.
5. **Validate** — Cross-check configs against known working patterns. Flag anything that
   could be service-affecting.
