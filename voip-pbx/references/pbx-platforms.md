# PBX Platforms Reference

## Table of Contents

1. [Asterisk](#asterisk)
2. [FreeSWITCH](#freeswitch)
3. [Kamailio](#kamailio)
4. [3CX](#3cx)
5. [FusionPBX](#fusionpbx)
6. [VitalPBX](#vitalpbx)
7. [Platform Comparison](#platform-comparison)
8. [Endpoint Provisioning](#endpoint-provisioning)
9. [TR-069 / CWMP](#tr-069)

---

## Asterisk

### Overview

Open-source PBX framework. The most widely deployed VoIP platform globally. Two SIP channel
drivers: `chan_sip` (legacy, deprecated) and `chan_pjsip` (current, recommended).

**Always use PJSIP for new deployments.** chan_sip is unmaintained and lacks modern features.

### Key Configuration Files

| File | Purpose |
|------|---------|
| `pjsip.conf` | SIP endpoints, trunks, transports, auth |
| `extensions.conf` | Dialplan (traditional format) |
| `extensions.ael` | Dialplan (AEL format — more readable for complex logic) |
| `queues.conf` | Call queue definitions |
| `voicemail.conf` | Voicemail boxes |
| `musiconhold.conf` | Hold music |
| `cdr.conf` / `cdr_adaptive_odbc.conf` | CDR output |
| `rtp.conf` | RTP port range |
| `modules.conf` | Module loading |
| `manager.conf` | AMI (Asterisk Manager Interface) access |
| `ari.conf` | ARI (Asterisk REST Interface) access |
| `http.conf` | HTTP server for ARI/WebSocket |

### PJSIP Configuration Patterns

#### Transport
```ini
[transport-udp]
type=transport
protocol=udp
bind=0.0.0.0:5060

[transport-tls]
type=transport
protocol=tls
bind=0.0.0.0:5061
cert_file=/etc/asterisk/certs/asterisk.pem
priv_key_file=/etc/asterisk/certs/asterisk.key
method=tlsv1_2
```

#### SIP Trunk (Registration-Based)
```ini
; Auth credentials
[my-provider-auth]
type=auth
auth_type=userpass
username=my_account_id
password=my_secret_password

; Registration to the provider
[my-provider-reg]
type=registration
transport=transport-udp
outbound_auth=my-provider-auth
server_uri=sip:sip.provider.com.au
client_uri=sip:my_account_id@sip.provider.com.au
retry_interval=60
expiration=3600

; Endpoint definition
[my-provider]
type=endpoint
transport=transport-udp
context=from-trunk
disallow=all
allow=alaw        ; A-law for AU
allow=ulaw        ; Fallback
dtmf_mode=rfc4733
outbound_auth=my-provider-auth
aors=my-provider
from_user=my_account_id
from_domain=sip.provider.com.au
; NAT settings if needed:
; rtp_symmetric=yes
; force_rport=yes
; rewrite_contact=yes

[my-provider]
type=aor
contact=sip:sip.provider.com.au
qualify_frequency=60

; Identify incoming by IP (if provider doesn't register to us)
[my-provider-identify]
type=identify
endpoint=my-provider
match=203.0.113.0/24
```

#### SIP Trunk (IP-Based Auth)
```ini
[carrier-trunk]
type=endpoint
transport=transport-udp
context=from-trunk
disallow=all
allow=alaw
dtmf_mode=rfc4733
direct_media=no

[carrier-trunk]
type=aor
contact=sip:203.0.113.1

[carrier-trunk-identify]
type=identify
endpoint=carrier-trunk
match=203.0.113.1
```

#### Extension/Phone
```ini
[1001]
type=endpoint
transport=transport-udp
context=internal
disallow=all
allow=alaw
allow=g722         ; HD voice for internal calls
auth=1001-auth
aors=1001
callerid="Josh Finlay" <1001>
mailboxes=1001@default
dtmf_mode=rfc4733
direct_media=no    ; Keep media flowing through Asterisk
rtp_symmetric=yes
force_rport=yes
rewrite_contact=yes

[1001-auth]
type=auth
auth_type=userpass
username=1001
password=SecureP@ssw0rd!

[1001]
type=aor
max_contacts=3     ; Allow 3 devices per extension
remove_existing=no ; Don't kick existing registrations
qualify_frequency=30
```

### Dialplan Patterns (extensions.conf)

```ini
[internal]
; Internal extension dialing
exten => _1XXX,1,NoOp(Dialing internal extension ${EXTEN})
 same => n,Dial(PJSIP/${EXTEN},30,tTrR)
 same => n,VoiceMail(${EXTEN}@default,u)
 same => n,Hangup()

; Australian local/national calls (strip leading 0, add +61)
exten => _0[2-9]XXXXXXXX,1,NoOp(AU geographic: ${EXTEN})
 same => n,Set(CALLERID(num)=0712345678)  ; Set outbound CLI
 same => n,Set(OUTNUM=+61${EXTEN:1})       ; Strip 0, prepend +61
 same => n,Dial(PJSIP/${OUTNUM}@my-provider,60)
 same => n,Hangup()

; Australian mobile
exten => _04XXXXXXXX,1,NoOp(AU mobile: ${EXTEN})
 same => n,Set(OUTNUM=+61${EXTEN:1})
 same => n,Dial(PJSIP/${OUTNUM}@my-provider,60)
 same => n,Hangup()

; 13/1300 numbers
exten => _13XXXX,1,Dial(PJSIP/${EXTEN}@my-provider,60)
exten => _1300XXXXXX,1,Dial(PJSIP/${EXTEN}@my-provider,60)

; 1800 free call
exten => _1800XXXXXX,1,Dial(PJSIP/${EXTEN}@my-provider,60)

; Emergency - ALWAYS route and NEVER block
exten => 000,1,NoOp(EMERGENCY CALL)
 same => n,Dial(PJSIP/000@my-provider,60)
 same => n,Hangup()

; International (0011 prefix)
exten => _0011.,1,NoOp(International: ${EXTEN})
 same => n,Set(OUTNUM=+${EXTEN:4})  ; Strip 0011, prepend +
 same => n,Dial(PJSIP/${OUTNUM}@my-provider,60)
 same => n,Hangup()

[from-trunk]
; Inbound DID routing
exten => _+61XXXXXXXXX,1,NoOp(Inbound DID: ${EXTEN})
 same => n,Goto(inbound-routes,${EXTEN},1)

exten => _0XXXXXXXXX,1,NoOp(Inbound DID: ${EXTEN})
 same => n,Set(NORMALIZED=+61${EXTEN:1})
 same => n,Goto(inbound-routes,${NORMALIZED},1)

[inbound-routes]
; Route DIDs to extensions/queues/IVRs
exten => +61712345678,1,Dial(PJSIP/1001,30)
 same => n,VoiceMail(1001@default,u)

exten => +61712345679,1,Goto(ivr-main,s,1)

[ivr-main]
exten => s,1,Answer()
 same => n,Playback(custom/welcome)
 same => n,Background(custom/ivr-menu)
 same => n,WaitExten(5)

exten => 1,1,Goto(sales-queue,s,1)
exten => 2,1,Goto(support-queue,s,1)
exten => i,1,Playback(invalid)
 same => n,Goto(ivr-main,s,1)
exten => t,1,Goto(ivr-main,s,1)
```

### Asterisk CLI Commands

```bash
# Reload configs
asterisk -rx "pjsip reload"
asterisk -rx "dialplan reload"
asterisk -rx "core reload"

# Show registrations
asterisk -rx "pjsip show registrations"

# Show endpoints and their status
asterisk -rx "pjsip show endpoints"
asterisk -rx "pjsip show endpoint my-provider"

# Active channels/calls
asterisk -rx "core show channels"
asterisk -rx "core show channel PJSIP/1001-00000001"

# SIP debug (VERY verbose — use sparingly)
asterisk -rx "pjsip set logger on"
asterisk -rx "pjsip set logger off"

# Dialplan debug
asterisk -rx "dialplan show internal"
asterisk -rx "core set verbose 5"

# Queue status
asterisk -rx "queue show"

# CDR
asterisk -rx "cdr show status"

# Generate a test call from CLI
asterisk -rx "channel originate PJSIP/1001 extension 1002@internal"
```

### AMI (Asterisk Manager Interface)

TCP interface (default port 5038) for external control:

```
Action: Login
Username: admin
Secret: password

Action: Originate
Channel: PJSIP/1001
Exten: 1002
Context: internal
Priority: 1
CallerID: Test <1001>

Action: Command
Command: pjsip show endpoints
```

### ARI (Asterisk REST Interface)

HTTP/WebSocket API for building applications. More modern than AMI.

```bash
# List channels
curl -u user:pass http://localhost:8088/ari/channels

# Originate a call
curl -X POST -u user:pass \
  "http://localhost:8088/ari/channels?endpoint=PJSIP/1001&extension=1002&context=internal"

# Answer a channel
curl -X POST -u user:pass \
  "http://localhost:8088/ari/channels/{channel_id}/answer"
```

ARI with Stasis applications lets you build dynamic call control in any language (Python,
Node.js, PHP, etc.). The `ari-py` and `ari-js` libraries simplify this.

---

## FreeSWITCH

### Overview

High-performance, modular telephony platform. Better suited than Asterisk for large-scale
or carrier-grade deployments. Core is written in C, uses XML for configuration, and Lua/Python/
JavaScript for scripting.

### Key Directories

```
/etc/freeswitch/
├── freeswitch.xml          # Master config (includes everything)
├── vars.xml                # Global variables
├── sip_profiles/
│   ├── internal.xml        # Internal SIP profile (phones, port 5060)
│   └── external.xml        # External SIP profile (trunks, port 5080)
├── directory/
│   └── default/            # User/extension definitions
├── dialplan/
│   ├── default.xml         # Internal dialplan
│   └── public.xml          # Inbound (external) dialplan
├── autoload_configs/       # Module configurations
│   ├── switch.conf.xml
│   ├── sofia.conf.xml      # SIP module config
│   └── ...
└── gateways/               # SIP trunk definitions (or in sip_profiles)
```

### Gateway (SIP Trunk) Configuration
```xml
<!-- /etc/freeswitch/sip_profiles/external/my-provider.xml -->
<include>
  <gateway name="my-provider">
    <param name="username" value="my_account_id"/>
    <param name="password" value="my_secret_password"/>
    <param name="realm" value="sip.provider.com.au"/>
    <param name="proxy" value="sip.provider.com.au"/>
    <param name="register" value="true"/>
    <param name="expire-seconds" value="3600"/>
    <param name="retry-seconds" value="30"/>
    <param name="caller-id-in-from" value="false"/>
    <param name="codec-prefs" value="PCMA,PCMU,G722"/>
  </gateway>
</include>
```

### User/Extension Definition
```xml
<!-- /etc/freeswitch/directory/default/1001.xml -->
<include>
  <user id="1001">
    <params>
      <param name="password" value="SecureP@ssw0rd!"/>
      <param name="vm-password" value="1234"/>
    </params>
    <variables>
      <variable name="toll_allow" value="domestic,international,local"/>
      <variable name="accountcode" value="1001"/>
      <variable name="user_context" value="default"/>
      <variable name="effective_caller_id_name" value="Josh Finlay"/>
      <variable name="effective_caller_id_number" value="1001"/>
      <variable name="outbound_caller_id_name" value="Athena Networks"/>
      <variable name="outbound_caller_id_number" value="0712345678"/>
    </variables>
  </user>
</include>
```

### FreeSWITCH CLI Commands (fs_cli)

```bash
# Reload config
fs_cli -x "reloadxml"
fs_cli -x "sofia profile external restart"  # WARNING: drops calls on this profile

# Show registrations
fs_cli -x "sofia status profile internal reg"
fs_cli -x "sofia status gateway my-provider"

# Active calls
fs_cli -x "show calls"
fs_cli -x "show channels"

# SIP trace
fs_cli -x "sofia global siptrace on"
fs_cli -x "sofia global siptrace off"

# Originate test call
fs_cli -x "originate sofia/internal/1001%10.0.0.1 &echo()"
```

---

## Kamailio

### Overview

High-performance SIP proxy and registrar. NOT a PBX — it doesn't handle media. Used as:
- SIP proxy / load balancer in front of Asterisk/FreeSWITCH
- Registrar for large-scale deployments (100k+ endpoints)
- SBC (with rtpproxy/rtpengine for media relay)
- Carrier-grade SIP routing engine

### When to Use Kamailio

- You have >1000 concurrent registrations
- You need to load-balance across multiple media servers
- You need carrier-grade SIP routing logic
- You need a SIP firewall / anti-fraud layer
- You're building a SIP trunking platform

### Architecture Pattern

```
Internet → Kamailio (SBC/proxy) → rtpengine (media relay)
                ↓                         ↓
           Asterisk/FreeSWITCH      RTP direct to endpoints
           (media processing)
```

### Key Config File: kamailio.cfg

Kamailio uses a C-like scripting language in its config:

```c
/* kamailio.cfg - simplified example */
#!KAMAILIO

/* Global parameters */
listen=udp:0.0.0.0:5060
listen=tcp:0.0.0.0:5060
listen=tls:0.0.0.0:5061

/* Module loading */
loadmodule "sl.so"          /* Stateless replies */
loadmodule "tm.so"          /* Transaction management */
loadmodule "rr.so"          /* Record-Route */
loadmodule "registrar.so"   /* SIP registrar */
loadmodule "usrloc.so"      /* User location DB */
loadmodule "nathelper.so"   /* NAT traversal */
loadmodule "rtpengine.so"   /* RTP proxy control */
loadmodule "dispatcher.so"  /* Load balancing */
loadmodule "pike.so"        /* Flood protection */
loadmodule "htable.so"      /* Hash tables */

/* Module parameters */
modparam("registrar", "max_expires", 3600)
modparam("usrloc", "db_url", "mysql://kamailio:pass@localhost/kamailio")
modparam("rtpengine", "rtpengine_sock", "udp:127.0.0.1:2223")
modparam("dispatcher", "list_file", "/etc/kamailio/dispatcher.list")

/* Main request routing */
request_route {
    /* Anti-flood */
    if (!pike_check_req()) {
        xlog("L_WARN", "PIKE: blocked $si\n");
        exit;
    }

    /* NAT detection */
    if (nat_uac_test("19")) {
        if (is_method("REGISTER")) {
            fix_nated_register();
        } else {
            fix_nated_contact();
        }
        force_rport();
        setflag(FLT_NATS);
    }

    /* REGISTER handling */
    if (is_method("REGISTER")) {
        if (!www_authorize("", "subscriber")) {
            www_challenge("", "1");
            exit;
        }
        save("location");
        exit;
    }

    /* Record route for in-dialog requests */
    if (!is_method("REGISTER")) {
        record_route();
    }

    /* Relay to media servers via dispatcher */
    if (is_method("INVITE")) {
        /* RTP engine for NAT traversal */
        if (isflagset(FLT_NATS)) {
            rtpengine_manage("replace-origin replace-session-connection");
        }

        /* Load balance across media servers */
        if (!ds_select_dst("1", "4")) {  /* group 1, round-robin */
            send_reply("503", "Service Unavailable");
            exit;
        }
    }

    t_relay();
}
```

### dispatcher.list (load balancing)
```
# Group 1: Asterisk media servers
1 sip:10.0.0.11:5060 0 0 weight=50
1 sip:10.0.0.12:5060 0 0 weight=50

# Group 2: FreeSWITCH conference servers
2 sip:10.0.0.21:5060 0 0
```

---

## 3CX

### Overview

Commercial PBX with web-based management. Popular in the SMB market. Runs on Linux (Debian)
or as a hosted solution. Less flexible than Asterisk/FreeSWITCH but much easier to manage.

### Key Points

- Configuration is done via web UI, not config files
- SIP trunk setup via web wizard
- Built-in provisioning for supported phones
- Supports hot desking, BLF, presence
- Licensing based on simultaneous calls, not extensions
- Has a built-in SBC component for remote workers
- REST API available for integration
- Backup/restore is straightforward

### 3CX CLI / API

```bash
# Useful 3CX commands (Debian-based install)
/usr/sbin/3CXPhoneSystem  # Main service
systemctl status 3cxpbx

# PostgreSQL database (3CX uses PG internally)
sudo -u phonesystem psql -d database_single

# Logs
/var/lib/3cxpbx/Instance1/Data/Logs/

# API endpoints (v18+)
GET  /api/SystemStatus
GET  /api/ExtensionList
POST /api/callcontrol/{ext}/makecall
```

---

## FusionPBX

### Overview

Web interface for FreeSWITCH. Provides a GUI for managing FreeSWITCH configurations. Multi-
tenant capable — good for hosting PBX services.

### Key Features

- Multi-tenant (multiple domains/companies on one server)
- Web-based administration
- Ring groups, queues, IVR, time conditions
- Built-in provisioning server
- FAX server
- Conference bridges
- CDR with web viewer
- REST API

### Common Tasks

```bash
# Restart FreeSWITCH (underlying engine)
systemctl restart freeswitch

# Database (FusionPBX uses PostgreSQL or SQLite)
sudo -u postgres psql fusionpbx

# Provisioning files location
/var/www/fusionpbx/app/provision/

# Logs
/var/log/freeswitch/
```

---

## VitalPBX

### Overview

Free/commercial PBX based on Asterisk (with FreePBX-like interface but independently
developed). Gaining traction as an alternative to FreePBX since Sangoma's licensing changes.

### Key Features

- Multi-tenant
- Web administration
- Add-on modules (some free, some paid)
- Built on CentOS/AlmaLinux with Asterisk
- Phone provisioning
- Sonata Suite for call centre features

---

## Platform Comparison

| Feature | Asterisk | FreeSWITCH | Kamailio | 3CX |
|---------|----------|------------|----------|-----|
| Type | PBX/media | PBX/media | SIP proxy | PBX |
| Scale | 100s of calls | 1000s of calls | 10000s+ reg | 100s of calls |
| Config | Text files | XML | C-like script | Web UI |
| Media handling | Yes | Yes | No (needs rtpengine) | Yes |
| Multi-tenant | Limited | Yes (FusionPBX) | Yes | Yes (Enterprise) |
| Learning curve | Medium | High | High | Low |
| Flexibility | Very high | Very high | Very high | Low-medium |
| WebRTC | Via module | Built-in | Via rtpengine | Built-in |
| Best for | SMB PBX | Carrier/hosted | SIP routing/SBC | SMB easy deploy |

---

## Endpoint Provisioning

### Auto-Provisioning Methods

1. **DHCP Option 66/67** — DHCP server tells phone where to fetch config.
   Option 66 = TFTP/HTTP server URL, Option 67 = config filename.

2. **DNS SRV / NAPTR** — Phone looks up `_sip._udp.domain.com` or provisioning URL via DNS.

3. **Redirect server** — Manufacturer's cloud redirect (Yealink RPS, Grandstream GDMS,
   Polycom ZTP). Phone contacts manufacturer on boot, gets redirected to your provisioning
   server.

4. **TR-069 / CWMP** — See section below.

5. **Manual** — Phone's web UI. Last resort.

### Common Phone Brands and Config Formats

| Brand | Config Format | Provisioning Protocol | Notes |
|-------|-------------|----------------------|-------|
| Yealink | XML (.cfg) | HTTP/HTTPS/TFTP | RPS cloud redirect available |
| Grandstream | XML or text | HTTP/HTTPS/TFTP | GDMS cloud management |
| Polycom/Poly | XML (.cfg) | HTTP/HTTPS/FTP | ZTP zero-touch provisioning |
| Cisco SPA | XML | HTTP/HTTPS/TFTP | Legacy but still common |
| Fanvil | XML | HTTP/HTTPS/TFTP | |
| Snom | XML | HTTP/HTTPS/TFTP | |

### Yealink Provisioning Template Example

```xml
<!-- y000000000000.cfg (MAC-based config) -->
<y0000000000xx>
  <!-- Account 1 -->
  <account.1.enable>1</account.1.enable>
  <account.1.label>1001</account.1.label>
  <account.1.display_name>Josh Finlay</account.1.display_name>
  <account.1.auth_name>1001</account.1.auth_name>
  <account.1.user_name>1001</account.1.user_name>
  <account.1.password>SecureP@ssw0rd!</account.1.password>
  <account.1.sip_server.1.address>pbx.athena.net.au</account.1.sip_server.1.address>
  <account.1.sip_server.1.port>5060</account.1.sip_server.1.port>
  <account.1.sip_server.1.transport_type>0</account.1.sip_server.1.transport_type>

  <!-- Codec preference -->
  <account.1.codec.1.enable>1</account.1.codec.1.enable>
  <account.1.codec.1.payload_type>PCMA</account.1.codec.1.payload_type>
  <account.1.codec.1.priority>0</account.1.codec.1.priority>

  <account.1.codec.2.enable>1</account.1.codec.2.enable>
  <account.1.codec.2.payload_type>G722</account.1.codec.2.payload_type>
  <account.1.codec.2.priority>1</account.1.codec.2.priority>

  <!-- DTMF -->
  <account.1.dtmf.type>1</account.1.dtmf.type>  <!-- 1=RFC2833 -->

  <!-- Network -->
  <network.vlan.internet_port_enable>1</network.vlan.internet_port_enable>
  <network.vlan.internet_port_vid>100</network.vlan.internet_port_vid>
  <network.vlan.pc_port_enable>0</network.vlan.pc_port_enable>

  <!-- QoS -->
  <network.qos.rtptos>46</network.qos.rtptos>     <!-- EF -->
  <network.qos.signaltos>26</network.qos.signaltos> <!-- AF31 -->
</y0000000000xx>
```

---

## TR-069 / CWMP

### Overview

TR-069 (CWMP — CPE WAN Management Protocol) is a remote management protocol for CPE devices
including IP phones, ATAs, routers, and ONTs. The ACS (Auto Configuration Server) pushes
configuration to and retrieves data from CPE devices.

### Architecture

```
ACS Server ←→ (HTTPS) ←→ CPE Device (phone/ATA/router)
Port 7547 (CPE listening)
Port 7548 (ACS listening for Connection Requests)
```

### Key Concepts

- **Inform** — CPE contacts the ACS periodically (or on events like boot, value change)
- **GetParameterValues** — ACS reads device settings
- **SetParameterValues** — ACS pushes settings to device
- **AddObject / DeleteObject** — Create/remove configuration objects (e.g., SIP accounts)
- **Download** — Push firmware or config files to the device
- **Reboot** — Remotely restart the device
- **Connection Request** — ACS tells CPE to contact it immediately (push model)

### Data Model

TR-069 uses a hierarchical parameter tree. For VoIP devices, the relevant data model is
defined in TR-104 (VoIP CPE) and TR-181 (Device:2 data model).

Common VoIP parameters:
```
Device.Services.VoiceService.1.VoiceProfile.1.SIP.ProxyServer
Device.Services.VoiceService.1.VoiceProfile.1.SIP.RegistrarServer
Device.Services.VoiceService.1.VoiceProfile.1.SIP.UserAgentDomain
Device.Services.VoiceService.1.VoiceProfile.1.Line.1.SIP.AuthUserName
Device.Services.VoiceService.1.VoiceProfile.1.Line.1.SIP.AuthPassword
Device.Services.VoiceService.1.VoiceProfile.1.Line.1.SIP.URI
Device.Services.VoiceService.1.VoiceProfile.1.Line.1.CallingFeatures.CallerIDName
Device.Services.VoiceService.1.VoiceProfile.1.Line.1.Enable
```

### ACS Software Options

| Software | Type | Notes |
|----------|------|-------|
| GenieACS | Open source | Node.js, MongoDB. Good for dev/small-medium scale |
| OpenACS | Open source | Java-based |
| AVSystem | Commercial | Enterprise-grade, used by large ISPs |
| Axiros | Commercial | Carrier-grade |
| FreeACS | Open source | Java, MySQL. Older but functional |

### GenieACS Quick Reference

```bash
# Install (Ubuntu/Debian)
npm install -g genieacs

# Services
genieacs-cwmp    # CWMP server (listens for CPE informs) - port 7547
genieacs-nbi     # Northbound API (REST) - port 7557
genieacs-fs      # File server (firmware, configs) - port 7567
genieacs-ui      # Web UI - port 3000

# API examples
# Get all devices
curl http://localhost:7557/devices

# Set parameter on a device
curl -X POST http://localhost:7557/devices/DEVICE_ID/tasks \
  -H "Content-Type: application/json" \
  -d '{"name": "setParameterValues", "parameterValues": [
    ["Device.Services.VoiceService.1.VoiceProfile.1.SIP.ProxyServer", "pbx.example.com", "xsd:string"]
  ]}'

# Reboot a device
curl -X POST http://localhost:7557/devices/DEVICE_ID/tasks \
  -H "Content-Type: application/json" \
  -d '{"name": "reboot"}'

# Push firmware
curl -X POST http://localhost:7557/devices/DEVICE_ID/tasks \
  -H "Content-Type: application/json" \
  -d '{"name": "download", "file": "firmware_v2.bin"}'
```

### TR-069 Security Considerations

- Always use HTTPS between ACS and CPE
- Use strong ACS credentials per device (not shared)
- Connection Request authentication prevents unauthorized reboots
- Certificate validation on CPE is often poorly implemented — consider mutual TLS
- Rate-limit informs to prevent DDoS on the ACS from misbehaving CPE
