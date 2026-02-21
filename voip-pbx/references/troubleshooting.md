# VoIP Troubleshooting Reference

## Table of Contents

1. [Diagnostic Methodology](#diagnostic-methodology)
2. [Packet Capture Tools](#packet-capture-tools)
3. [Common Issues and Solutions](#common-issues)
4. [SIP Trace Analysis](#sip-trace-analysis)
5. [RTP/Media Analysis](#rtp-media-analysis)
6. [Quality Diagnostics](#quality-diagnostics)
7. [Network-Level Diagnostics](#network-level)
8. [Security Issues](#security-issues)

---

## Diagnostic Methodology

Always follow this order when troubleshooting VoIP issues:

1. **Reproduce** — Can you reproduce the issue? Is it constant or intermittent?
2. **Isolate** — Signaling problem or media problem? One direction or both?
3. **Capture** — Get a packet capture. SIP traces from the PBX are second best.
4. **Analyze** — Read the SIP flow, check SDP, verify RTP streams.
5. **Fix** — Apply the fix, verify with another capture.

### Quick Diagnostic Decision Tree

```
Problem reported
├── No audio at all?
│   ├── Check SDP c= lines for private IPs → NAT issue
│   ├── Check firewall for RTP port range → Ports blocked
│   └── Check codec negotiation → 488 / codec mismatch
├── One-way audio?
│   ├── Audio works A→B but not B→A → B's SDP has wrong IP (NAT)
│   ├── Check for SIP ALG → Disable it
│   └── Check directmedia/reinvite settings → Media path changed
├── Calls drop after X seconds?
│   ├── 30-32 seconds → SIP Timer B / INVITE timeout / NAT binding
│   ├── Exactly 15 or 30 minutes → Session timer expiry
│   └── Random times → Network instability / qualify timeout
├── Poor audio quality?
│   ├── Choppy/robotic → Packet loss or high jitter
│   ├── Echo → Echo cancellation issue / impedance mismatch
│   ├── Delay/latency → Network path too long / jitter buffer too large
│   └── Static/noise → Codec issue or electrical interference (analog)
├── Registration failing?
│   ├── 401/407 → Wrong credentials or realm mismatch
│   ├── 403 → IP not authorized / ACL blocking
│   └── Timeout → DNS resolution failure / firewall blocking
└── DTMF not working?
    ├── Check DTMF method match (RFC4733 vs INFO vs inband)
    ├── Check telephone-event negotiated in SDP
    └── Check for transcoding stripping events
```

---

## Packet Capture Tools

### tcpdump

The essential first step. Capture on the PBX/SBC interface.

```bash
# Capture all SIP traffic
tcpdump -i eth0 -n -s 0 -w /tmp/sip_capture.pcap port 5060

# Capture SIP + RTP (specific port range)
tcpdump -i eth0 -n -s 0 -w /tmp/voip_capture.pcap \
  '(port 5060 or port 5061 or (udp portrange 10000-20000))'

# Capture traffic to/from a specific host
tcpdump -i eth0 -n -s 0 -w /tmp/specific.pcap host 203.0.113.1

# Capture traffic for a specific phone (by IP)
tcpdump -i eth0 -n -s 0 -w /tmp/phone.pcap host 10.0.1.100

# Live SIP message viewing (quick and dirty)
tcpdump -i eth0 -n -s 0 -A port 5060 | grep -E "^(SIP|INVITE|REGISTER|BYE|CANCEL|ACK|OPTIONS|Via|From|To|Call-ID|CSeq|Contact)"

# Rotate captures (10 files of 100MB each)
tcpdump -i eth0 -n -s 0 -w /tmp/voip_%Y%m%d_%H%M%S.pcap \
  -G 3600 -W 10 port 5060
```

### sngrep

Purpose-built SIP message viewer. Interactive, shows call flows visually.

```bash
# Install
apt install sngrep  # Debian/Ubuntu
yum install sngrep  # CentOS/RHEL

# Live capture
sngrep -d eth0

# Read from pcap
sngrep -I /tmp/sip_capture.pcap

# Filter by call-id
sngrep -c "a84b4c76e66710"

# Filter by method
sngrep -M INVITE

# Filter by from/to
sngrep -f "sip:1001@"
```

sngrep provides:
- Call flow diagrams (arrow-style, like the ones in the SIP protocol reference)
- SDP inspection
- Call duration/status
- Filtering by method, caller, callee, Call-ID

### Wireshark

For deep analysis after capture. Key Wireshark features for VoIP:

```
# Display filters
sip                          # All SIP traffic
sip.Method == "INVITE"       # Only INVITEs
sip.Status-Code == 486       # Busy responses
sip.Call-ID == "abc123"      # Specific call
rtp                          # All RTP
rtcp                         # All RTCP

# VoIP analysis
Telephony → VoIP Calls       # Lists all detected calls
Telephony → RTP → RTP Streams  # Shows RTP stream stats
Telephony → RTP → Stream Analysis  # Jitter/loss graphs

# Play audio from capture
Telephony → VoIP Calls → select call → Play Streams
```

### ngrep

Network grep — search packet payloads for patterns.

```bash
# Find all SIP INVITEs to a specific number
ngrep -d eth0 -W byline "INVITE sip:0412345678" port 5060

# Find a specific Call-ID
ngrep -d eth0 -W byline "Call-ID: a84b4c76e66710" port 5060

# Find registration attempts from a user
ngrep -d eth0 -W byline "REGISTER" port 5060 | grep -A5 "From.*1001"
```

### Platform-Specific SIP Traces

**Asterisk:**
```bash
# PJSIP logger (writes to Asterisk console/log)
asterisk -rx "pjsip set logger on"
# ... reproduce issue ...
asterisk -rx "pjsip set logger off"

# Full debug to file
asterisk -rx "core set verbose 5"
asterisk -rx "core set debug 5"
# Logs go to /var/log/asterisk/full (check logger.conf)
```

**FreeSWITCH:**
```bash
# SIP trace
fs_cli -x "sofia global siptrace on"
# ... reproduce issue ...
fs_cli -x "sofia global siptrace off"

# Per-profile trace
fs_cli -x "sofia profile internal siptrace on"

# Save pcap from FreeSWITCH
fs_cli -x "sofia profile internal capture on"
# Creates /tmp/dump_*.pcap
```

**Kamailio:**
```bash
# Enable SIP message logging (add to kamailio.cfg or use kamctl)
kamctl trap      # Toggle SIP debug logging

# sipdump module for pcap output
loadmodule "sipdump.so"
modparam("sipdump", "enable", 1)
```

---

## Common Issues and Solutions

### No Audio (Both Directions)

| Check | What to Look For | Fix |
|-------|-----------------|-----|
| SDP c= line | Private IP (192.168.x, 10.x) | Enable NAT traversal on PBX |
| Firewall/RTP ports | UDP 10000-20000 blocked | Open RTP port range |
| Codec mismatch | 488 response / no common codec | Align codec configs |
| RTP reaching PBX? | tcpdump shows no inbound RTP | Check firewall, routing |
| directmedia/reinvite | PBX not in media path | Set directmedia=no |

### One-Way Audio

Almost always a NAT issue. Check:

1. **SDP in both directions** — does the c= line contain a routable IP?
2. **SIP ALG** — is the router rewriting SDP incorrectly? Disable SIP ALG.
3. **Symmetric RTP** — is `rtp_symmetric=yes` set? (Asterisk) / comedia enabled?
4. **Direct media** — did a re-INVITE change the media path? Set `directmedia=no`.
5. **Firewall state tracking** — is the firewall tracking UDP connections? Some firewalls
   drop return RTP because they don't associate it with the outbound stream.

### Calls Drop After 30 Seconds

- SIP Timer B (INVITE timeout) = 32 seconds
- The INVITE is being sent but no response is coming back
- Check: Is the INVITE reaching the remote end? (packet capture on both sides)
- Common cause: Firewall blocking responses, DNS resolution failure, routing issue

### Calls Drop After 15-30 Minutes

- SIP Session Timer (RFC 4028) — periodic re-INVITE or UPDATE to keep the session alive
- If the re-INVITE fails (e.g., Contact IP changed due to NAT), the call drops
- Fix: Ensure session timers are supported and Contact headers are correct
- Asterisk: `session_timers=accept` or `session_timers=refuse` (if you don't need them)

### Registration Expires / Drops

- NAT binding timeout — the NAT router forgets the port mapping
- Fix: Shorter registration interval (60-120s) or use OPTIONS keepalives
- Asterisk PJSIP: `qualify_frequency=30` sends OPTIONS every 30s
- Some firewalls have UDP timeout of 30-60 seconds — check firewall settings

### Echo

| Type | Cause | Fix |
|------|-------|-----|
| Acoustic echo | Speaker audio picked up by microphone | Enable echo cancellation on phone |
| Electrical echo | Impedance mismatch at FXO/FXS interface | Adjust echo canceller on gateway |
| Network echo | High latency causing perceptible delay | Reduce latency; not really "echo" |

Echo cancellation settings:
- Asterisk: `echocancel=yes` and `echotrain=800` on DAHDI channels
- Most IP phones have built-in echo cancellation — ensure it's enabled
- FXO gateways (Grandstream, Cisco ATA): adjust echo cancellation in web UI

### Toll Fraud

Signs of toll fraud:
- Spike in international calls, especially to premium rate numbers
- Calls at unusual hours
- Calls from unused extensions

Prevention:
```ini
# Asterisk - restrict international dialing per extension
[restricted-context]
; Only allow AU numbers, block 0011 prefix
exten => _0[2-9]XXXXXXXX,1,Dial(PJSIP/${EXTEN}@trunk)
exten => _04XXXXXXXX,1,Dial(PJSIP/${EXTEN}@trunk)
; No international pattern = blocked by default
```

```bash
# fail2ban for Asterisk (essential on internet-facing systems)
# /etc/fail2ban/jail.d/asterisk.conf
[asterisk]
enabled  = true
filter   = asterisk
logpath  = /var/log/asterisk/messages
maxretry = 3
bantime  = 86400
findtime = 600
action   = iptables-allports[name=ASTERISK, protocol=all]
```

```ini
# Asterisk - strong password requirements
# In pjsip.conf, always use complex passwords:
[1001-auth]
type=auth
auth_type=userpass
username=1001
password=xK9#mP2$vL8nQ    ; 15+ chars, mixed case, symbols, numbers
```

---

## SIP Trace Analysis

### Reading a SIP Trace — What to Look For

When you have a SIP trace (from sngrep, Wireshark, or PBX logs), check these in order:

1. **INVITE sent?** — Is the INVITE leaving the PBX toward the trunk/endpoint?
2. **100 Trying received?** — If not, the INVITE isn't reaching the remote end.
3. **Response code** — What's the final response? (See response code table in SKILL.md)
4. **SDP in INVITE** — Check codecs offered, c= line IP, RTP port.
5. **SDP in 200 OK** — Check codec selected, c= line IP, RTP port.
6. **ACK sent?** — If ACK doesn't reach, the 200 OK will be retransmitted and call may fail.
7. **RTP flowing?** — After ACK, are RTP packets actually being sent/received?

### Common Error Responses and What They Mean

| Response | Meaning | Common Cause | Fix |
|----------|---------|-------------|-----|
| 401/407 | Auth required | Missing or wrong credentials | Check username/password |
| 403 | Forbidden | IP not in ACL, or CLI not authorized | Check provider ACL settings |
| 404 | Not Found | Number doesn't exist | Check dial plan, number format |
| 408 | Request Timeout | Remote didn't respond in time | Network issue, check connectivity |
| 480 | Temporarily Unavailable | Endpoint offline | Check phone registration |
| 486 | Busy Here | Called party is on a call | Normal behavior; check call waiting |
| 487 | Request Terminated | CANCEL was sent (caller hung up) | Normal behavior |
| 488 | Not Acceptable Here | No common codec | Align codec settings |
| 500 | Internal Error | PBX/proxy error | Check PBX logs |
| 502 | Bad Gateway | Upstream returned invalid response | Check trunk connectivity |
| 503 | Service Unavailable | Trunk down or overloaded | Check trunk registration/status |

---

## RTP/Media Analysis

### Using Wireshark for RTP Analysis

1. Open capture in Wireshark
2. Go to Telephony → RTP → RTP Streams
3. For each stream, check:
   - **Packets**: Total count (expected = duration_seconds × 50 for 20ms ptime)
   - **Lost**: Packet loss count and percentage
   - **Max Delta**: Maximum inter-packet delay (jitter indicator)
   - **Max Jitter**: Calculated jitter value
   - **Mean Jitter**: Average jitter

4. Select a stream → Analyze → Shows per-packet timing graph

### Interpreting RTP Stats

| Metric | Good | Marginal | Bad |
|--------|------|----------|-----|
| Packet Loss | <0.5% | 0.5-2% | >2% |
| Jitter | <10ms | 10-30ms | >30ms |
| Max Delta | <30ms | 30-60ms | >60ms |
| Sequence errors | 0 | 1-5 | >5 |

### RTP Debug Commands

```bash
# Asterisk - RTP debug
asterisk -rx "rtp set debug on"
asterisk -rx "rtp set debug ip 10.0.1.100"  # Filter by IP

# FreeSWITCH - RTP debug
fs_cli -x "sofia global siptrace on"
# RTP stats are in the BYE/RTCP reports

# Check if RTP packets are arriving
tcpdump -i eth0 -n udp portrange 10000-20000 -c 100
# If you see packets, RTP is flowing. If not, firewall/NAT issue.
```

---

## Quality Diagnostics

### Testing Tools

```bash
# Ping with jitter analysis
ping -c 100 -i 0.02 remote_host | tail -1
# Shows min/avg/max/mdev — mdev is a jitter indicator

# mtr (traceroute with stats)
mtr -n -c 100 --report remote_host
# Shows per-hop loss and latency

# iperf3 (bandwidth and jitter test)
# Server: iperf3 -s
# Client:
iperf3 -c remote_host -u -b 100K -l 160 -t 60
# -u = UDP, -b 100K = 100kbps (simulates voice), -l 160 = G.711 payload size

# SIPVicious (SIP security scanner — for YOUR OWN systems only)
svmap 10.0.0.0/24            # Discover SIP devices
svwar -m INVITE 10.0.0.1     # Enumerate extensions
svcrack -u 1001 10.0.0.1     # Brute force test (YOUR systems only)

# SIPp (SIP load testing)
sipp -sf uac_pcap.xml -s 1001 10.0.0.1:5060 -r 10 -rp 1000 -m 100
# -r 10 = 10 calls/sec, -m 100 = 100 total calls
```

### VoIP Quality Monitoring

For ongoing quality monitoring, consider:

- **VoIPmonitor** — Open source, captures and analyzes SIP/RTP in real-time
- **Homer/SIPCAPTURE** — SIP capture and analysis platform (uses HEP protocol)
- **Asterisk CDR + RTCP stats** — Built-in, export to database
- **RTCP-XR** — If supported by endpoints, provides per-call MOS/R-factor

### Homer/SIPCAPTURE Setup (Brief)

Homer captures SIP messages via the HEP (Homer Encapsulation Protocol) protocol. Many SIP
proxies and PBXes can send HEP natively:

- Kamailio: `siptrace` module with HEP output
- FreeSWITCH: `mod_hep` module
- Asterisk: `res_hep` module (available in newer versions)

Homer provides a web interface for searching, filtering, and visualizing SIP call flows.

---

## Network-Level Diagnostics

### Voice VLAN Verification

```bash
# Check VLAN tagging on MikroTik
/interface vlan print
/interface bridge vlan print

# Check LLDP-MED on switch (Cisco example)
show lldp neighbors detail
# Should show phone's voice VLAN assignment

# Verify QoS/DSCP marking
tcpdump -i eth0 -n -v udp portrange 10000-20000 -c 5
# Look for "tos 0xb8" (DSCP EF/46) in the output
# 0xb8 = 10111000 = DSCP 46 (EF) with ECN bits 00

# Check for QoS on MikroTik
/queue tree print
/ip firewall mangle print where action=mark-packet
```

### Firewall Rules for VoIP

```bash
# iptables example for a VoIP server
# SIP
iptables -A INPUT -p udp --dport 5060 -j ACCEPT
iptables -A INPUT -p tcp --dport 5060 -j ACCEPT
iptables -A INPUT -p tcp --dport 5061 -j ACCEPT  # TLS

# RTP
iptables -A INPUT -p udp --dport 10000:20000 -j ACCEPT

# Restrict SIP to known IPs (recommended)
iptables -A INPUT -p udp --dport 5060 -s 203.0.113.0/24 -j ACCEPT
iptables -A INPUT -p udp --dport 5060 -j DROP

# MikroTik firewall for SIP trunk
/ip firewall filter add chain=input protocol=udp dst-port=5060 \
  src-address=203.0.113.0/24 action=accept comment="SIP from provider"
/ip firewall filter add chain=input protocol=udp dst-port=10000-20000 \
  action=accept comment="RTP media"

# IMPORTANT: Disable SIP ALG (MikroTik)
/ip firewall service-port set sip disabled=yes

# Disable SIP ALG (Linux/netfilter)
modprobe -r nf_conntrack_sip
echo "blacklist nf_conntrack_sip" >> /etc/modprobe.d/blacklist.conf
```

---

## Security Issues

### SIP Scanning / Brute Force

Symptoms:
- Lots of 401/407 responses in logs
- REGISTER/INVITE from unknown IPs
- User-Agent strings like "friendly-scanner", "sipvicious", "sipcli"

Mitigation:
```bash
# fail2ban (essential)
# Asterisk filter: /etc/fail2ban/filter.d/asterisk.conf
[Definition]
failregex = NOTICE.* .*: Registration from '.*' failed for '<HOST>:.*' - Wrong password
            NOTICE.* .*: Registration from '.*' failed for '<HOST>:.*' - No matching peer found
            NOTICE.* .*: Registration from '.*' failed for '<HOST>:.*' - Username/auth name mismatch
            NOTICE.* <HOST> failed to authenticate as '.*'

# Kamailio - pike module (in kamailio.cfg)
if (!pike_check_req()) {
    xlog("L_WARN", "Blocked $si - too many requests\n");
    exit;
}
```

### TLS Configuration Verification

```bash
# Test SIP TLS connection
openssl s_client -connect pbx.example.com:5061

# Check certificate
openssl x509 -in /etc/asterisk/certs/asterisk.pem -text -noout

# Generate self-signed cert for testing
openssl req -x509 -newkey rsa:4096 -keyout asterisk.key -out asterisk.crt \
  -days 365 -nodes -subj "/CN=pbx.athena.net.au"
cat asterisk.crt asterisk.key > asterisk.pem
```
