# Voice Network Design Reference

## Table of Contents

1. [Voice VLAN Design](#voice-vlan-design)
2. [QoS Implementation](#qos-implementation)
3. [Capacity Planning](#capacity-planning)
4. [High Availability](#high-availability)
5. [WAN Design for Voice](#wan-design)
6. [SBC Architecture](#sbc-architecture)
7. [WebRTC Integration](#webrtc)
8. [Monitoring and Alerting](#monitoring)

---

## Voice VLAN Design

### Why Separate VLANs

Voice traffic requires dedicated VLANs for:
- **QoS enforcement** — Can't prioritise voice if it's mixed with data at L2
- **Security isolation** — Phones shouldn't be accessible from the data network
- **Broadcast domain separation** — Data broadcast storms won't affect phones
- **Simplified ACLs** — Easier to apply voice-specific firewall rules

### Standard Design Pattern

```
Phone ──[Voice VLAN 100]──┐
  │                        │
  └─[Data VLAN 10]─ PC    Switch ── Router/Firewall ── PBX
                           │
                           └── Server VLAN 200 ── PBX Server
```

Phones typically have a built-in switch. The PC connects through the phone:
- Phone tags voice traffic with VLAN 100
- PC traffic passes through untagged (native VLAN / data VLAN)

### VLAN Assignment Methods

1. **LLDP-MED** (preferred) — Switch tells the phone which VLAN to use via LLDP-MED
   (Link Layer Discovery Protocol - Media Endpoint Discovery). Most modern phones and
   switches support this.

2. **CDP** (Cisco) — Cisco Discovery Protocol performs the same function on Cisco
   switches/phones.

3. **DHCP Option 132/133** — DHCP can assign VLAN information via vendor-specific options.

4. **Static** — Configure the VLAN ID directly on the phone. Works but doesn't scale.

5. **802.1x with VLAN assignment** — RADIUS assigns VLAN based on device authentication.
   Most secure but most complex.

### MikroTik Voice VLAN Example

```routeros
# Create voice VLAN
/interface vlan add interface=bridge1 name=vlan100-voice vlan-id=100

# Bridge VLAN filtering (if using bridge VLAN filtering)
/interface bridge vlan
add bridge=bridge1 tagged=bridge1,ether1 untagged="" vlan-ids=100

# DHCP for voice VLAN
/ip pool add name=voice-pool ranges=10.0.100.10-10.0.100.250
/ip dhcp-server add address-pool=voice-pool interface=vlan100-voice name=voice-dhcp
/ip dhcp-server network add address=10.0.100.0/24 gateway=10.0.100.1 \
  dns-server=10.0.100.1 \
  dhcp-option=option66,s'http://provisioning.example.com/'

# QoS marking for voice VLAN traffic
/ip firewall mangle add chain=forward src-address=10.0.100.0/24 \
  protocol=udp dst-port=5060 action=set-priority new-priority=6 \
  comment="SIP signaling priority"
/ip firewall mangle add chain=forward src-address=10.0.100.0/24 \
  protocol=udp dst-port=10000-20000 action=set-priority new-priority=7 \
  comment="RTP voice priority"

# Firewall — isolate voice from data
/ip firewall filter add chain=forward src-address=10.0.10.0/24 \
  dst-address=10.0.100.0/24 action=drop comment="Block data to voice"
/ip firewall filter add chain=forward src-address=10.0.100.0/24 \
  dst-address=10.0.10.0/24 action=drop comment="Block voice to data"
# Allow voice to PBX
/ip firewall filter add chain=forward src-address=10.0.100.0/24 \
  dst-address=10.0.200.10/32 action=accept comment="Phones to PBX"
```

### Cisco Switch Voice VLAN Example

```
! Enable voice VLAN on access port
interface GigabitEthernet0/1
 switchport mode access
 switchport access vlan 10          ! Data VLAN
 switchport voice vlan 100          ! Voice VLAN
 spanning-tree portfast             ! Fast convergence for phones
 mls qos trust cos                  ! Trust QoS from phone

! Or with LLDP-MED (for non-Cisco phones)
lldp run
interface GigabitEthernet0/1
 lldp med-tlv-select network-policy
 network-policy profile 1
  voice vlan 100 cos 5
```

---

## QoS Implementation

### QoS Strategy

Voice QoS requires three components:
1. **Classification** — Identify voice traffic (by VLAN, port, DSCP)
2. **Marking** — Apply DSCP values (EF for RTP, CS3/AF31 for SIP)
3. **Queuing/Scheduling** — Prioritise marked traffic at congestion points

### DSCP Marking

Always mark at the closest point to the source (trust the phone's marking, or mark at the
access switch). Re-mark at WAN boundaries if needed.

| Traffic | DSCP | Decimal | PHB | Priority |
|---------|------|---------|-----|----------|
| Voice RTP | EF | 46 | Expedited Forwarding | Highest |
| Voice SIP | AF31 or CS3 | 26 or 24 | Assured Forwarding | High |
| Video RTP | AF41 | 34 | Assured Forwarding | High |
| Business Critical | AF21 | 18 | Assured Forwarding | Medium |
| Best Effort | BE | 0 | Default | Low |
| Scavenger | CS1 | 8 | Scavenger | Lowest |

### MikroTik QoS for Voice

```routeros
# Simple queue tree approach
# Mark voice packets
/ip firewall mangle
add chain=prerouting src-address=10.0.100.0/24 protocol=udp \
  dst-port=10000-20000 action=mark-packet new-packet-mark=voice-rtp
add chain=prerouting src-address=10.0.100.0/24 protocol=udp \
  dst-port=5060 action=mark-packet new-packet-mark=voice-sip

# Queue tree with priority
/queue tree
add name=voice-rtp parent=global packet-mark=voice-rtp priority=1 \
  max-limit=10M
add name=voice-sip parent=global packet-mark=voice-sip priority=2 \
  max-limit=1M
add name=all-other parent=global packet-mark=no-mark priority=8 \
  max-limit=90M

# Set DSCP on outbound
/ip firewall mangle
add chain=postrouting packet-mark=voice-rtp action=change-dscp \
  new-dscp=46 comment="EF for voice RTP"
add chain=postrouting packet-mark=voice-sip action=change-dscp \
  new-dscp=26 comment="AF31 for voice signaling"
```

### WAN QoS Sizing

Reserve bandwidth for voice on WAN links:

```
Voice reservation = max_concurrent_calls × bandwidth_per_call × 1.2 (overhead)

Example: 20 concurrent G.711 calls on a 100Mbps WAN:
  20 × 175kbps × 1.2 = 4.2 Mbps reservation
  This is ~4.2% of the link — easily achievable
  
Example: 20 concurrent G.711 calls on a 10Mbps WAN:
  Same 4.2 Mbps = 42% of the link — needs careful QoS management
```

Rule of thumb: If voice would use >33% of the WAN link, either upgrade the link or use
G.729 to reduce bandwidth.

---

## Capacity Planning

### Erlang Calculations

The Erlang is the unit of telecommunications traffic intensity.
1 Erlang = 1 circuit in continuous use for 1 hour.

**Erlang B formula** — calculates the probability that a call is blocked (no available circuits).
Used for trunk sizing.

```
Example:
- Office with 100 employees
- Average 6 calls per hour per employee
- Average call duration: 3 minutes
- Target blocking probability: 1% (P.01)

Traffic = (100 × 6 × 3/60) = 30 Erlangs

Using Erlang B table for 30 Erlangs at P.01:
Required trunks ≈ 42 concurrent channels

With SIP, "trunks" = maximum concurrent calls on the SIP trunk.
```

### Quick Erlang Reference (P.01 = 1% blocking)

| Traffic (Erlang) | Trunks Needed | Typical Use Case |
|-------------------|--------------|-------------------|
| 1 | 4 | Very small office (5 staff) |
| 5 | 11 | Small office (20 staff) |
| 10 | 18 | Medium office (40 staff) |
| 20 | 30 | Large office (80 staff) |
| 50 | 64 | Call centre (200 agents) |
| 100 | 120 | Large call centre |

### Server Sizing

**Asterisk:**
- 1 CPU core can handle ~150-200 concurrent G.711 calls (without transcoding)
- With G.729 transcoding: ~30-50 concurrent calls per core
- Recording adds ~10-20% CPU overhead
- RAM: 1GB base + ~2MB per concurrent call
- Disk I/O matters for recording and CDR writes

**FreeSWITCH:**
- Generally handles more concurrent calls than Asterisk for the same hardware
- ~500-800 concurrent calls per core (G.711, pass-through)
- With transcoding: similar to Asterisk per core

**Kamailio:**
- 10,000+ concurrent registrations on modest hardware
- 1,000+ call setups per second
- Memory-focused — needs RAM for user location table

---

## High Availability

### Active-Passive PBX Failover

```
                    ┌─── PBX Primary (active)
Phones ── Switch ──|
                    └─── PBX Secondary (standby)
                              ↕ (database replication)
```

Options:
- **Pacemaker/Corosync** — Linux HA cluster. Virtual IP floats between nodes.
- **DRBD** — Block-level replication for configuration and voicemail files.
- **Database replication** — MySQL/PostgreSQL replication for CDR, user data.

### Active-Active with Kamailio

```
                           ┌── Asterisk 1
Phones ── Kamailio (LB) ──┤
                           └── Asterisk 2
```

Kamailio dispatcher module load-balances across multiple media servers.
Registration data stored in shared database (MySQL/PostgreSQL).

### SIP Trunk Failover

```ini
# Asterisk PJSIP — multiple contacts with priority
[my-trunk]
type=aor
contact=sip:primary.provider.com
contact=sip:secondary.provider.com
qualify_frequency=30    ; Check health every 30s

# Or use DNS SRV for automatic failover
# _sip._udp.provider.com SRV 10 0 5060 primary.provider.com
# _sip._udp.provider.com SRV 20 0 5060 secondary.provider.com
```

### Geographic Redundancy

For carrier-grade voice:
```
Brisbane DC ── WAN ── Sydney DC
    │                     │
    PBX 1                PBX 2
    │                     │
    └──── Shared DB ──────┘
           (async replication)
```

Considerations:
- SIP registrations need to follow the user (use a distributed registrar like Kamailio)
- CDR reconciliation between sites
- DID routing — update SIP trunk provider to route to the active site
- WAN latency between sites affects media if calls span both sites

---

## WAN Design for Voice

### Requirements

| Metric | Target | Notes |
|--------|--------|-------|
| Latency | <150ms one-way | AU domestic typically <30ms on NBN |
| Jitter | <30ms | |
| Packet loss | <1% | |
| Bandwidth | Plan per above calculations | |

### WAN Technologies for Voice in Australia

| Technology | Typical Latency | Typical Jitter | Voice Suitability |
|-----------|----------------|-----------------|-------------------|
| NBN FTTP | 5-15ms | <5ms | Excellent |
| NBN FTTC/FTTN | 10-30ms | 5-15ms | Good (FTTN can vary) |
| NBN HFC | 10-25ms | 5-20ms | Good (watch for congestion) |
| NBN Fixed Wireless | 15-40ms | 10-30ms | Fair (weather-dependent) |
| NBN Satellite | 600ms+ | Variable | Poor (use G.729, expect quality issues) |
| Business-grade fibre | 2-10ms | <3ms | Excellent |
| 4G/5G | 20-50ms | 10-40ms | Fair (variable, use adaptive codecs) |
| MPLS | 5-15ms | <5ms | Excellent (with QoS) |

### VPN and Voice

Running voice over VPN adds overhead and potential issues:

- **IPSec overhead**: ~50-60 bytes per packet (significant for small voice packets)
- **Encryption latency**: Minimal on modern hardware
- **MTU issues**: Encapsulation can cause fragmentation. Set MTU appropriately.
- **UDP over IPSec**: Works, but some VPN implementations handle UDP poorly
- **Best practice**: Use TLS for SIP + SRTP for media instead of VPN for voice.
  This is more efficient and purpose-built for voice security.

---

## SBC Architecture

### What an SBC Does

A Session Border Controller sits at the network edge and provides:

1. **Topology hiding** — Masks internal network structure from external parties
2. **NAT traversal** — Handles SIP/SDP rewriting for NAT
3. **Protocol normalization** — Translates between different SIP dialects/versions
4. **Media handling** — RTP relay, transcoding, recording
5. **Security** — Rate limiting, ACLs, TLS termination, toll fraud prevention
6. **Interoperability** — Adapts SIP headers/behavior for different providers
7. **Regulatory compliance** — CLI manipulation, emergency call handling

### Open Source SBC Stack

```
                    ┌── Kamailio (SIP proxy)
Internet ──────────|
                    └── rtpengine (media proxy)
                              │
                         Internal network
                              │
                    PBX (Asterisk/FreeSWITCH)
```

**Kamailio** handles SIP signaling:
- Registration, authentication, routing
- NAT detection and fix (`nathelper` module)
- Rate limiting (`pike` module)
- Load balancing (`dispatcher` module)

**rtpengine** handles media:
- RTP relay and NAT traversal
- Codec transcoding
- SRTP ↔ RTP bridging
- Call recording (via forking)
- ICE/STUN/TURN for WebRTC

### rtpengine Configuration

```bash
# rtpengine startup options
rtpengine --interface=external/203.0.113.1 \
          --interface=internal/10.0.0.1 \
          --listen-ng=127.0.0.1:2223 \
          --port-min=30000 \
          --port-max=40000 \
          --log-level=5

# In Kamailio, control rtpengine:
rtpengine_manage("replace-origin replace-session-connection");
# This rewrites SDP c= and o= lines to use rtpengine's IP
```

### Commercial SBC Options

| Product | Notes |
|---------|-------|
| AudioCodes Mediant | Hardware and VM options, widely used in AU enterprise |
| Oracle SBC (Acme Packet) | Carrier-grade, expensive |
| Ribbon (GENBAND) SBC | Carrier-grade |
| Sangoma SBC | Integrated with FreePBX |
| Obbec | Popular with smaller carriers |

---

## WebRTC Integration

### Architecture

```
Browser ──[WSS + SRTP]──→ WebRTC Gateway ──[SIP + RTP]──→ PBX
```

WebRTC requires:
- **WSS (WebSocket Secure)** for SIP signaling (not UDP/TCP SIP)
- **DTLS-SRTP** for media encryption (mandatory in WebRTC)
- **ICE/STUN/TURN** for NAT traversal
- **Opus** codec (mandatory in WebRTC, usually supported by PBX)

### Platform Support

- **Asterisk**: Built-in WebRTC support via `res_http_websocket` and PJSIP WSS transport.
  Requires certificates for TLS/DTLS.

- **FreeSWITCH**: Built-in WebRTC support via `mod_verto` (FreeSWITCH-specific) or
  standard SIP-over-WSS.

- **Kamailio + rtpengine**: Kamailio handles WSS, rtpengine handles DTLS-SRTP ↔ RTP
  bridging. Most flexible for carrier-grade.

### Asterisk WebRTC Configuration

```ini
# http.conf
[general]
enabled=yes
bindaddr=0.0.0.0
bindport=8088
tlsenable=yes
tlsbindaddr=0.0.0.0:8089
tlscertfile=/etc/asterisk/certs/asterisk.pem
tlsprivatekey=/etc/asterisk/certs/asterisk.key

# pjsip.conf
[transport-wss]
type=transport
protocol=wss
bind=0.0.0.0:8089
cert_file=/etc/asterisk/certs/asterisk.crt
priv_key_file=/etc/asterisk/certs/asterisk.key

[webrtc-endpoint]
type=endpoint
transport=transport-wss
context=internal
disallow=all
allow=opus         ; WebRTC requires Opus
allow=alaw         ; Fallback
dtls_auto_generate_cert=yes
webrtc=yes         ; Shorthand for enabling all WebRTC requirements
; This sets: media_encryption=dtls, dtls_verify=fingerprint,
;   dtls_setup=actpass, ice_support=yes, rtcp_mux=yes,
;   use_avpf=yes
```

---

## Monitoring and Alerting

### What to Monitor

| Metric | Tool | Alert Threshold |
|--------|------|----------------|
| Registration count | PBX CLI / SNMP | Drop >10% from baseline |
| Failed registrations | PBX logs / fail2ban | >5 per minute from same IP |
| Concurrent calls | PBX CLI / AMI | >80% of capacity |
| Trunk status | PJSIP qualify / OPTIONS | Any trunk marked unreachable |
| Call quality (MOS) | RTCP-XR / VoIPmonitor | MOS <3.5 |
| Packet loss | RTCP reports | >1% |
| Jitter | RTCP reports | >30ms |
| CPU usage | OS monitoring | >70% sustained |
| Disk usage | OS monitoring | >80% (especially if recording) |
| SIP error rate | CDR analysis | >5% of calls failing |

### LibreNMS Integration

LibreNMS can monitor VoIP infrastructure via SNMP:

- Asterisk: `res_snmp` module exposes call counts, channel stats
- MikroTik: SNMP for interface stats, queue stats
- FreeSWITCH: `mod_snmp` or custom scripts

### Graylog / Syslog for SIP Logging

Forward PBX logs to Graylog for centralised analysis:

```bash
# Asterisk syslog output (logger.conf)
[logfiles]
syslog.local0 => notice,warning,error

# rsyslog forwarding to Graylog
# /etc/rsyslog.d/asterisk.conf
local0.* @graylog.server:514
```

Create Graylog extractors for SIP fields (Call-ID, From, To, response codes) to enable
searching and dashboarding.

### Prometheus + Grafana

Modern monitoring stack:
- **Asterisk exporter**: https://github.com/mhein-sa/asterisk_exporter
- **FreeSWITCH exporter**: Via mod_prometheus or custom ESL scripts
- **Kamailio**: `statsd` module exports metrics natively

Key Grafana dashboards:
- Active calls over time
- Registration count trend
- Call success/failure ratio
- Trunk utilisation
- Average MOS score
