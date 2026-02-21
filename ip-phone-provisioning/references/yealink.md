# Yealink Configuration & Provisioning Reference

## Table of Contents

1. [Provisioning Architecture](#provisioning-architecture)
2. [Boot Files](#boot-files)
3. [Configuration File Hierarchy](#config-hierarchy)
4. [Core Configuration Parameters](#core-parameters)
5. [Account / Line Configuration](#account-config)
6. [Codec Configuration](#codec-config)
7. [Network & VLAN Configuration](#network-config)
8. [Line Keys / DSS Keys / BLF](#line-keys)
9. [Features Configuration](#features-config)
10. [Firmware Management](#firmware)
11. [Yealink RPS (Redirect & Provisioning Service)](#rps)
12. [Common Models Reference](#models)
13. [Troubleshooting](#troubleshooting)

---

## Provisioning Architecture

Yealink uses a boot file → config file chain:

```
Phone boots
  ↓
Obtains provisioning URL (DHCP Opt 66, PnP, RPS, or manual)
  ↓
Downloads boot file (y000000000000.boot or {mac}.boot)
  ↓
Boot file lists config files to download (in order)
  ↓
Downloads each config file in sequence
  ↓
Parameters merge: later files override earlier files for same parameter
  ↓
Phone applies config, registers to PBX
```

### File Naming Convention

| File | Naming | Scope |
|------|--------|-------|
| Common boot | `y000000000000.boot` | All Yealink phones |
| MAC boot | `{mac}.boot` (lowercase) | Specific phone |
| Common config | `y000000000000.cfg` | All Yealink phones |
| Model config | `y0000000000XX.cfg` | All phones of model XX |
| MAC config | `{mac}.cfg` (lowercase) | Specific phone |

**Model codes** (the XX in `y0000000000XX.cfg`):

| Code | Model(s) |
|------|----------|
| 68 | T46S/T46G, T48S/T48G |
| 69 | T42S/T42G |
| 70 | T41S/T41P |
| 85 | T54W, T57W |
| 95 | T53W, T53 |
| 96 | T43U |
| 97 | T46U |
| 108 | T54W (newer firmware) |
| 110 | T57W (newer firmware) |
| 123 | T34W |
| 159 | T58W |
| 65 | CP920 |

To find the exact code for your model, check the Yealink admin guide or look at the
`static.auto_provision.url_wildcard.pn` parameter in the phone's web UI.

---

## Boot Files

### Common Boot File: `y000000000000.boot`

```ini
#!version:1.0.0.1

## The first line MUST be the version line — do not edit or delete it.
## Lines starting with # are comments.
## Files download in listed order; later parameters override earlier ones.

## Global config (all phones, all models)
include:config "y000000000000.cfg"

## Model-specific config (use $MODEL_CODE wildcard or explicit filename)
include:config "y0000000000$MODEL_CODE.cfg"

## Per-phone config (MAC address)
include:config "$MAC.cfg"

## Optional: custom per-phone directory
## include:config "$MAC-directory.xml"
```

### MAC-Specific Boot File: `{mac}.boot`

If a MAC-specific boot file exists, it takes precedence over the common boot file.
Use this for phones that need a completely different config chain.

```ini
#!version:1.0.0.1

## Custom boot for a specific phone
include:config "y000000000000.cfg"
include:config "special-config.cfg"
include:config "$MAC.cfg"
```

### Boot File Variables

| Variable | Resolves To | Example |
|----------|------------|---------|
| `$MAC` | MAC address (lowercase, no colons) | `001565abcdef` |
| `$MODEL_CODE` | Model numeric code | `85` |

---

## Config Hierarchy

Configuration files are XML-formatted with a `.cfg` extension. Yealink uses an XML-like
syntax where parameter names are tag names:

```xml
<!-- y000000000000.cfg — Global config for all Yealink phones -->

## Overwrite mode: 0 = don't overwrite user changes, 1 = always overwrite
## Use 1 for managed deployments, 0 if users are allowed to customise
overwrite_mode = 1

## If a specific static. prefix is used, the parameter is ALWAYS applied
## regardless of overwrite mode. Use for critical settings.
## Example: static.auto_provision.server.url = http://prov.example.com/yealink/
```

### Overwrite Modes

| Mode | Behavior |
|------|----------|
| `overwrite_mode = 0` | Only apply parameters that haven't been changed by the user |
| `overwrite_mode = 1` | Force all parameters, overwriting any user changes |
| `static.` prefix | Always applied, regardless of overwrite mode |

**Best practice**: Use `overwrite_mode = 1` for centrally managed deployments. Use the
`static.` prefix for absolutely critical settings (provisioning URL, firmware URL, admin
password) that should never be overridden.

---

## Core Configuration Parameters

### Auto Provisioning Settings

```ini
## Provisioning server URL
static.auto_provision.server.url = https://prov.athena.net.au/yealink/

## Authentication for provisioning server
static.auto_provision.server.username = prov_user
static.auto_provision.server.password = prov_password

## Provisioning mode: 0=Off, 1=Power On, 2=Repeatedly, 3=Weekly
auto_provision.mode = 2

## Repeat interval in minutes (default 1440 = 24 hours)
auto_provision.repeat.minutes = 1440

## PnP (Plug and Play SIP multicast)
auto_provision.pnp_enable = 1

## DHCP option for provisioning: 0=Disabled, 1=Option 66, 2=Option 43,
## 3=Option 66 and 43, 4=Custom option
auto_provision.dhcp_option.enable = 1

## AES encryption key for config files (if using encrypted configs)
## auto_provision.aes_key_in_file = MySecretAESKey123
```

### Time & Locale (Australia)

```ini
## NTP
local_time.ntp_server1 = au.pool.ntp.org
local_time.ntp_server2 = pool.ntp.org

## Timezone: +10 for AEST (Brisbane/Sydney), +10:30 for ACST
local_time.time_zone = +10
local_time.time_zone_name = Australia/Brisbane

## DST: 0=Off (for QLD), 2=Auto
## For states with DST (NSW, VIC, SA, TAS, ACT):
## local_time.summer_time = 2
local_time.summer_time = 0

## Date/time display
local_time.date_format = 2    ## 0=WWW MMM DD, 1=DD-MMM-YY, 2=DD/MM/YYYY, etc
local_time.time_format = 1    ## 0=12hr, 1=24hr

## Language
lang.gui = English

## Tone scheme
voice.tone_country = Australia
```

### Admin & Security

```ini
## Web UI admin password (CHANGE THIS)
static.security.user_password = admin
static.security.user_name.admin = admin
static.security.user_password.admin = Y0urStr0ngP@ssw0rd!

## User-level password (for basic phone user access)
static.security.user_password.user = UserP@ss123

## Disable HTTP, force HTTPS for web UI
wui.http_enable = 0
wui.https_enable = 1

## Web UI port
wui.https_port = 443

## Disable Telnet
features.telnet_enable = 0

## SSH (disabled by default, enable for management if needed)
## static.network.ssh_enable = 1
```

---

## Account / Line Configuration

### Single Account (Line 1)

```ini
## Enable account 1
account.1.enable = 1

## Display name (caller ID name)
account.1.display_name = Josh Finlay

## SIP credentials
account.1.auth_name = 1001
account.1.user_name = 1001
account.1.password = SipP@ssw0rd!

## Label shown on the phone's line key
account.1.label = 1001

## SIP server
account.1.sip_server.1.address = pbx.athena.net.au
account.1.sip_server.1.port = 5060
account.1.sip_server.1.transport_type = 0   ## 0=UDP, 1=TCP, 2=TLS, 3=DNS-NAPTR

## Backup SIP server (failover)
account.1.sip_server.2.address = pbx-backup.athena.net.au
account.1.sip_server.2.port = 5060
account.1.sip_server.2.failover_mode = 1  ## 0=newcall only, 1=register failover
account.1.sip_server.2.transport_type = 0

## Outbound proxy (if required)
## account.1.outbound_proxy.1.address = proxy.athena.net.au
## account.1.outbound_proxy.1.port = 5060

## Registration expiry (seconds)
account.1.sip_server.1.expires = 3600

## NAT traversal
account.1.nat.nat_traversal = 0  ## 0=Disabled, 1=STUN, 2=Keep-alive
account.1.nat.udp_update_enable = 1
account.1.nat.udp_update_time = 30

## Caller ID
account.1.cid_source = 0  ## 0=FROM, 1=PAI, 2=RPID
account.1.outgoing_caller_id_name = Athena Networks
account.1.outgoing_caller_id_number = 0712345678
```

### Multiple Accounts

Yealink phones support multiple SIP accounts. Index starts at 1.

```ini
## Account 2 (second line)
account.2.enable = 1
account.2.label = Sales
account.2.display_name = Sales Line
account.2.auth_name = 1050
account.2.user_name = 1050
account.2.password = SalesP@ss!
account.2.sip_server.1.address = pbx.athena.net.au
account.2.sip_server.1.port = 5060
```

### SIP TLS Configuration

```ini
## Use TLS transport
account.1.sip_server.1.transport_type = 2  ## TLS

## TLS port (usually 5061)
account.1.sip_server.1.port = 5061

## SRTP: 0=Disabled, 1=Optional, 2=Compulsory
account.1.srtp_encryption = 1
```

---

## Codec Configuration

```ini
## Disable all codecs first, then enable in preference order
account.1.codec.1.enable = 1
account.1.codec.1.payload_type = PCMA    ## G.711 A-law — AU standard
account.1.codec.1.priority = 0           ## 0 = highest priority

account.1.codec.2.enable = 1
account.1.codec.2.payload_type = G722    ## HD Voice (wideband)
account.1.codec.2.priority = 1

account.1.codec.3.enable = 1
account.1.codec.3.payload_type = PCMU    ## G.711 μ-law — fallback
account.1.codec.3.priority = 2

account.1.codec.4.enable = 1
account.1.codec.4.payload_type = G729    ## Low bandwidth
account.1.codec.4.priority = 3

account.1.codec.5.enable = 0
account.1.codec.5.payload_type = opus    ## Opus — for WebRTC interop
account.1.codec.5.priority = 4

## Available payload types:
## PCMU, PCMA, G722, G729, G726-16, G726-24, G726-32, G726-40,
## iLBC, opus, AMR, AMR-WB
```

### DTMF Configuration

```ini
## DTMF mode per account: 0=Inband, 1=RFC2833, 2=SIP INFO, 3=RFC2833+SIP INFO
account.1.dtmf.type = 1            ## RFC2833 (recommended)
account.1.dtmf.dtmf_payload = 101  ## RTP payload type for RFC2833
```

---

## Network & VLAN Configuration

### Voice VLAN

```ini
## VLAN for the phone's voice traffic (PC port passes data VLAN through)
network.vlan.internet_port_enable = 1
network.vlan.internet_port_vid = 100       ## VLAN ID for voice
network.vlan.internet_port_priority = 5    ## 802.1p priority (0-7)

## PC port VLAN (pass data traffic from PC through phone)
network.vlan.pc_port_enable = 0            ## 0=untagged passthrough
## network.vlan.pc_port_vid = 10           ## If you need to tag PC traffic

## LLDP (for dynamic VLAN assignment from switch)
network.lldp.enable = 1

## CDP (Cisco Discovery Protocol)
network.cdp.enable = 1
```

### QoS / DSCP

```ini
## DSCP marking (decimal values)
network.qos.rtptos = 46            ## EF (Expedited Forwarding) for RTP
network.qos.signaltos = 26         ## AF31 for SIP signaling
## Alternative: signaltos = 24     ## CS3
```

### 802.1x

```ini
## 802.1x authentication
network.802_1x.mode = 0    ## 0=Disabled, 1=EAP-MD5, 2=EAP-TLS,
                            ## 3=PEAP-MSCHAPv2, 5=EAP-FAST, 8=EAP-TTLS-MSCHAPv2
## network.802_1x.identity = phone_user
## network.802_1x.md5_password = secret
```

---

## Line Keys / DSS Keys / BLF

Line keys (also called DSS keys) are configured per-index. The number of available keys
depends on the phone model.

### Key Types

| Type Value | Function | Notes |
|-----------|----------|-------|
| 0 | N/A (disabled) | |
| 1 | Conference | |
| 2 | Forward | |
| 3 | Transfer | |
| 4 | Hold | |
| 5 | DND | |
| 7 | Call Return | |
| 8 | SMS | |
| 9 | Directed Pickup | |
| 10 | Paging | |
| 13 | Speed Dial | Value = number to dial |
| 15 | Line | Value = account index |
| 16 | BLF | Value = extension to monitor |
| 17 | URL | Value = URL to open |
| 20 | Phone Lock | |
| 22 | Group Pickup | Value = pickup code |
| 23 | Multicast Paging | |
| 27 | BLF List | For BLF list from PBX |
| 34 | Hot Desking | |
| 35 | URL Record | |
| 38 | Directory | |
| 39 | Call Park (Park) | Value = park orbit |
| 40 | Call Park (Retrieve) | Value = park orbit |
| 45 | DTMF | Value = DTMF digits to send |
| 57 | Prefix | Value = prefix digits |

### Example: Line Keys

```ini
## Line key 1: Line 1 (account 1)
linekey.1.type = 15
linekey.1.line = 1
linekey.1.value =
linekey.1.label = Line 1

## Line key 2: BLF for extension 1002
linekey.2.type = 16
linekey.2.line = 1
linekey.2.value = 1002
linekey.2.label = Jane Smith
linekey.2.extension = 1002     ## Auto-dial on pickup

## Line key 3: Speed dial
linekey.3.type = 13
linekey.3.line = 1
linekey.3.value = 0412345678
linekey.3.label = Mobile

## Line key 4: BLF for extension 1003
linekey.4.type = 16
linekey.4.line = 1
linekey.4.value = 1003
linekey.4.label = Reception

## Line key 5: Call Park (park to orbit 701)
linekey.5.type = 39
linekey.5.line = 1
linekey.5.value = 701
linekey.5.label = Park 701

## Line key 6: Directed pickup
linekey.6.type = 9
linekey.6.line = 1
linekey.6.value = 1004
linekey.6.label = Pickup 1004
```

### Expansion Module (Sidecar) Keys

For phones with expansion modules (EXP40, EXP43, EXP50):

```ini
## Expansion module 1, key 1
expansion_module.1.key.1.type = 16
expansion_module.1.key.1.line = 1
expansion_module.1.key.1.value = 1005
expansion_module.1.key.1.label = Warehouse
```

---

## Features Configuration

### Call Waiting

```ini
call_waiting.enable = 1
call_waiting.tone = 1        ## 0=Off, 1=On (beep during active call)
```

### Call Forward

```ini
## Always forward
account.1.always_fwd.enable = 0
account.1.always_fwd.target =

## Busy forward
account.1.busy_fwd.enable = 1
account.1.busy_fwd.target = *991001   ## Voicemail or number

## No answer forward
account.1.timeout_fwd.enable = 1
account.1.timeout_fwd.target = *991001
account.1.timeout_fwd.timeout = 20    ## Seconds before forwarding
```

### Do Not Disturb

```ini
features.dnd.enable = 0
features.dnd.mode = 0    ## 0=Phone, 1=Custom (per-account)
```

### Voicemail

```ini
## Voicemail number (MWI subscribe)
voice_mail.number.1 = *97    ## Or your PBX voicemail access number
account.1.subscribe_mwi = 1
account.1.subscribe_mwi_to_vm = 1
```

### Action URL (Webhook Integration)

```ini
## Notify external system on call events
action_url.setup_completed = http://crm.example.com/api/call/start?caller=$caller_number
action_url.call_terminated = http://crm.example.com/api/call/end?caller=$caller_number&duration=$call_duration
action_url.incoming_call = http://crm.example.com/api/call/incoming?caller=$caller_number
```

### Phone Directory

```ini
## Remote phonebook URL
remote_phonebook.data.1.url = https://prov.athena.net.au/directory/company.xml
remote_phonebook.data.1.name = Company Directory

## LDAP directory
ldap.enable = 1
ldap.host = ldap.athena.net.au
ldap.port = 389
ldap.base = ou=people,dc=athena,dc=net,dc=au
ldap.name_attr = cn
ldap.number_attr = telephoneNumber
```

### Multicast Paging

```ini
## Listen to multicast paging
multicast.listen_address.1.ip_address = 224.0.1.75:10000
multicast.listen_address.1.label = All Page

## Paging key (assign to a linekey)
## linekey.X.type = 23
## linekey.X.value = 224.0.1.75:10000
## linekey.X.label = Page All
```

---

## Firmware Management

```ini
## Auto-update firmware from provisioning server
static.firmware.url = https://prov.athena.net.au/yealink/firmware/$PN/$PN.rom

## Or specify exact firmware file
## static.firmware.url = https://prov.athena.net.au/yealink/firmware/T54W/T54W-96.86.0.130.rom

## Firmware upgrade mode
## 0=Disabled, 1=Check on boot, 2=Check periodically, 3=Check on boot+periodically
over_provision.firmware_upgrade.mode = 1
```

### Firmware File Naming

Yealink firmware files are `.rom` format. Place them on the provisioning server:

```
/provisioning/yealink/firmware/
├── T54W/
│   └── T54W-96.86.0.130.rom
├── T46S/
│   └── T46S-66.86.0.130.rom
└── T53W/
    └── T53W-95.86.0.130.rom
```

**Important**: Yealink V87+ firmware requires a mandatory password change on first boot.
Pre-provision the admin password to avoid this blocking deployment:
`static.security.user_password.admin = Y0urP@ss!`

---

## Yealink RPS (Redirect & Provisioning Service)

Yealink RPS is a cloud service that redirects phones to your provisioning server on first boot.

### How It Works

```
Phone boots → Contacts Yealink RPS → RPS returns your provisioning URL → Phone downloads config
```

### Setup

1. Register at `https://rps.yealink.com` (or via your Yealink distributor)
2. Add your devices by MAC address (or import CSV)
3. Set the provisioning server URL for each device or group
4. Factory reset the phone — on boot, it contacts RPS and gets redirected

### RPS URL Format

In the RPS portal, set the server URL to:
`https://prov.athena.net.au/yealink/`

The phone will then download its boot file from that URL.

---

## Common Models Reference

| Model | Lines | Keys | Screen | PoE | Bluetooth | WiFi | Notes |
|-------|-------|------|--------|-----|-----------|------|-------|
| T31G | 2 | 2 | 2.4" BW | Yes | No | No | Budget |
| T33G | 4 | 4 | 2.4" Color | Yes | No | No | Entry |
| T34W | 4 | 4 | 2.4" Color | Yes | Yes | Yes | Entry wireless |
| T43U | 12 | 3 | 3.7" BW | Yes | No | No | Mid-range |
| T46U | 16 | 6 | 4.3" Color | Yes | Yes | No | Upper mid |
| T48U | 16 | 0 | 7" Touch | Yes | Yes | No | Touch screen |
| T53W | 12 | 3 | 3.7" BW | Yes | Yes | Yes | Mid wireless |
| T54W | 16 | 4 | 4.3" Color | Yes | Yes | Yes | Popular choice |
| T57W | 16 | 0 | 7" Touch | Yes | Yes | Yes | Executive |
| T58W | 16 | 0 | 7" Touch | Yes | Yes | Yes | Video capable |
| CP920 | 1 | - | Touch | Yes | Yes | Yes | Conference phone |
| CP960 | 1 | - | Touch | Yes | Yes | Yes | Premium conference |

---

## Troubleshooting

### Phone Not Downloading Config

1. Check provisioning URL: Web UI → Settings → Auto Provision → Server URL
2. Verify the URL is reachable: `curl -v http://prov.example.com/yealink/y000000000000.boot`
3. Check web server logs for 404s or 403s
4. Verify MAC address in filename matches (lowercase, no colons/dashes)
5. Check DHCP Option 66 is being served correctly

### Config Not Applying

1. Check `overwrite_mode` — is it set to 0 and the user has changed the setting?
2. Use `static.` prefix for settings that must always apply
3. Check boot file references the correct config files
4. Check XML syntax — malformed XML causes silent failures
5. Check provisioning logs on the phone: Web UI → Settings → Configuration

### Phone Stuck in Reboot Loop

1. Config file has a firmware URL that doesn't exist → phone keeps trying to upgrade
2. Fix the firmware URL or remove it from the config
3. Factory reset: hold OK button for 10 seconds during boot, or use the phone menu
4. Some models: press and hold the volume down + speaker button during boot

### BLF Not Working

1. Verify the PBX supports BLF (Asterisk: `hint` in dialplan + `subscribe_context`)
2. Check `linekey.X.line` matches the correct account index
3. Verify the phone can subscribe to the BLF extension (check PBX logs for SUBSCRIBE)
4. Some PBXes need `account.1.blf.blf_list_uri` configured for BLF lists
