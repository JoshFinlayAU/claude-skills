# Fanvil X/XU Series Configuration & Provisioning Reference

## Table of Contents

1. [Provisioning Architecture](#provisioning-architecture)
2. [Configuration File Format](#config-format)
3. [SIP Account Configuration](#sip-config)
4. [Codec Configuration](#codec-config)
5. [Network & VLAN](#network-config)
6. [DSS Keys / BLF](#dss-keys)
7. [Features](#features)
8. [Firmware Management](#firmware)
9. [Fanvil FDPS (Redirect Service)](#fdps)
10. [Common Models](#models)
11. [Troubleshooting](#troubleshooting)

---

## Provisioning Architecture

Fanvil phones use a config file download approach with priority-based source selection:

```
Phone boots
  ↓
Tries provisioning sources by priority:
  1. MDNS (multicast DNS discovery)
  2. FDPS (Fanvil Device Provisioning Service — cloud redirect)
  3. DHCP (Option 66)
  4. TR-069
  5. SIP PnP
  6. Phone Flash (stored URL from previous provisioning)
  ↓
Downloads config file(s) from the resolved URL
  ↓
Phone applies config and registers
```

### Config File Types

| File Type | Naming | Scope |
|-----------|--------|-------|
| MAC config | `{mac}.cfg` (lowercase) | Specific phone |
| Model common | `{model_name}.cfg` | All phones of that model |
| Custom | Any name (set in Phone Flash) | As configured |

### Common Config File Names by Model

| Model | Common Config Filename |
|-------|----------------------|
| X3U | `fanvil_x3u.cfg` |
| X4U | `fanvil_x4u.cfg` |
| X5U | `fanvil_x5u.cfg` |
| X6U | `fanvil_x6u.cfg` |
| X7C | `fanvil_x7c.cfg` |
| X210 | `fanvil_x210.cfg` |
| X301W | `fanvil_x301w.cfg` |
| X303W | `fanvil_x303w.cfg` |

### Default Credentials

| Access | Username | Password |
|--------|----------|----------|
| Web UI | admin | admin |
| Phone menu (advanced) | — | admin or 123 |

---

## Configuration File Format

Fanvil supports three config file formats: `.cfg` (text-based), `.txt`, and `.xml`.
The text-based `.cfg` format is most common and uses module headers.

### CFG Format Structure

```ini
##File Version: 2.0002
##

<<GLOBAL CONFIG MODULE>>
Host Name        :Athena-1001

<<NETWORK CONFIG MODULE>>
WANType          :DHCP
VLAN Enable      :1
VLAN ID          :100
VLAN Priority    :5

<<SIP CONFIG MODULE>>
SIP1 Phone Number        :1001
SIP1 Display Name        :Josh Finlay
SIP1 Register Addr       :pbx.athena.net.au
SIP1 Register Port       :5060
SIP1 Register User       :1001
SIP1 Register Passwd     :SipP@ssw0rd!
SIP1 Register TTL        :3600
SIP1 Enable              :1
SIP1 Transport           :0

<<AUDIO CONFIG MODULE>>
Primary Codec            :PCMA
Secondary Codec          :G722
Third Codec              :PCMU
Fourth Codec             :G729

<<DSS KEY CONFIG MODULE>>
Dss Key 1 Type           :1
Dss Key 1 Value          :1001
Dss Key 1 Label          :Line 1
Dss Key 1 Line           :1
```

### Key Rules

1. **Version header**: The first line `##File Version: X.XXXX` is mandatory. If the phone
   has already provisioned with this version, it won't re-apply. Increment the version to
   force re-provisioning.
2. **Module headers**: Each section starts with `<<MODULE NAME>>`. A parameter will only
   be applied if its module header is present in the config file.
3. **Spacing**: The colon `:` separates key from value. Spaces around the colon are required.
4. **Case sensitive**: Parameter names are case-sensitive.

### Config Encryption

Fanvil supports AES-encrypted config files:

1. Set "Config Encryption Key" in the phone's web UI (Maintenance → Auto Provision)
2. Encrypt the config file using the same key
3. The phone will decrypt on download

---

## SIP Account Configuration

### Full Account Parameters

```ini
<<SIP CONFIG MODULE>>
## Account 1
SIP1 Phone Number        :1001
SIP1 Display Name        :Josh Finlay
SIP1 Register Addr       :pbx.athena.net.au
SIP1 Register Port       :5060
SIP1 Register User       :1001
SIP1 Register Passwd     :SipP@ssw0rd!
SIP1 Register TTL        :3600
SIP1 Enable              :1

## Transport: 0=UDP, 1=TCP, 2=TLS
SIP1 Transport           :0

## Outbound proxy
SIP1 Proxy Addr          :
SIP1 Proxy Port          :5060

## Backup SIP server
SIP1 Backup Addr         :pbx-backup.athena.net.au
SIP1 Backup Port         :5060
SIP1 Backup Transport    :0

## NAT
SIP1 NAT Enable          :0
SIP1 STUN Server         :
SIP1 STUN Port           :3478

## Caller ID
SIP1 CID Source          :0
SIP1 Anonymous Enable    :0

## Subscribe
SIP1 MWI Subscribe       :1
SIP1 MWI Num             :*97
SIP1 BLF List URI        :
SIP1 Subscribe Period    :3600

## DTMF: 0=Inband, 1=RFC2833, 2=SIP INFO, 3=RFC2833+SIP INFO
SIP1 DTMF Mode           :1
SIP1 DTMF Payload        :101

## SRTP: 0=Disabled, 1=Optional, 2=Compulsory
SIP1 SRTP Enable         :0
```

### Multiple Accounts

Increment the number for each account (SIP1, SIP2, SIP3, etc.):

```ini
<<SIP CONFIG MODULE>>
## Account 2
SIP2 Phone Number        :1050
SIP2 Display Name        :Sales Line
SIP2 Register Addr       :pbx.athena.net.au
SIP2 Register Port       :5060
SIP2 Register User       :1050
SIP2 Register Passwd     :SalesP@ss!
SIP2 Register TTL        :3600
SIP2 Enable              :1
SIP2 Transport           :0
SIP2 DTMF Mode           :1
```

---

## Codec Configuration

```ini
<<AUDIO CONFIG MODULE>>
## Codec priority (first = highest priority)
Primary Codec            :PCMA
Secondary Codec          :G722
Third Codec              :PCMU
Fourth Codec             :G729

## Available codecs:
## PCMU, PCMA, G722, G729, G726-16, G726-24, G726-32, G726-40,
## iLBC_13K, iLBC_15K, AMR, AMR-WB, Opus

## Packetization time (ms)
SIP1 Ptime               :20

## Voice Activity Detection (silence suppression)
VAD Enable               :0

## Jitter buffer
Jitter Buffer Minimum    :30
Jitter Buffer Maximum    :120
Jitter Buffer Type       :1    ## 0=Fixed, 1=Adaptive
```

---

## Network & VLAN

```ini
<<NETWORK CONFIG MODULE>>
## IP configuration
WANType                  :DHCP      ## DHCP, Static, PPPoE

## Voice VLAN
VLAN Enable              :1
VLAN ID                  :100
VLAN Priority            :5

## PC port VLAN
PC VLAN Enable           :0
PC VLAN ID               :10
PC VLAN Priority         :0

## LLDP
LLDP Enable              :1

## CDP
CDP Enable               :1

## QoS/DSCP (decimal TOS byte values)
SIP QOS                  :104       ## 0x68 = DSCP 26 (AF31)
RTP QOS                  :184       ## 0xB8 = DSCP 46 (EF)

## NTP
SNTP Server              :au.pool.ntp.org
Time Zone                :+10:00    ## AEST
Daylight Saving          :0         ## 0=Off (QLD), 1=On
Time Format              :1         ## 0=12hr, 1=24hr
Date Format              :2         ## 0=YYYY/MM/DD, 1=MM/DD/YYYY, 2=DD/MM/YYYY

## HTTP port for web UI
HTTP Port                :80
HTTPS Port               :443
```

### Web UI Security

```ini
<<MANAGEMENT CONFIG MODULE>>
## Change admin password
Web Admin Password       :Y0urStr0ngP@ss!
Web Admin User           :admin

## Disable HTTP (force HTTPS)
HTTP Enable              :0
HTTPS Enable             :1
```

---

## DSS Keys / BLF

Fanvil DSS (Direct Station Selection) keys are configured in the DSS KEY CONFIG MODULE:

### DSS Key Types

| Type Value | Function |
|-----------|----------|
| 0 | None (disabled) |
| 1 | Line |
| 2 | Speed Dial |
| 3 | BLF (Busy Lamp Field) |
| 4 | Voice Mail |
| 5 | Directed Pickup |
| 6 | Call Forward |
| 7 | DND |
| 8 | SMS |
| 9 | Recall |
| 10 | Multicast Paging |
| 13 | Prefix |
| 14 | DTMF |
| 15 | Call Park |
| 16 | Intercom |
| 19 | Phone Lock |
| 20 | XML Group |
| 23 | Group Pickup |
| 27 | Hot Desking |
| 40 | URL |

### Example: DSS Key Configuration

```ini
<<DSS KEY CONFIG MODULE>>
## Key 1: Line 1
Dss Key 1 Type           :1
Dss Key 1 Value          :1001
Dss Key 1 Label          :Line 1
Dss Key 1 Line           :1

## Key 2: BLF for extension 1002
Dss Key 2 Type           :3
Dss Key 2 Value          :1002
Dss Key 2 Label          :Jane Smith
Dss Key 2 Line           :1

## Key 3: Speed dial
Dss Key 3 Type           :2
Dss Key 3 Value          :0412345678
Dss Key 3 Label          :Mobile
Dss Key 3 Line           :1

## Key 4: BLF for extension 1003
Dss Key 4 Type           :3
Dss Key 4 Value          :1003
Dss Key 4 Label          :Reception
Dss Key 4 Line           :1

## Key 5: Call Park
Dss Key 5 Type           :15
Dss Key 5 Value          :701
Dss Key 5 Label          :Park 701
Dss Key 5 Line           :1

## Key 6: Directed Pickup
Dss Key 6 Type           :5
Dss Key 6 Value          :1004
Dss Key 6 Label          :Pickup 1004
Dss Key 6 Line           :1

## Key 7: Multicast Paging
Dss Key 7 Type           :10
Dss Key 7 Value          :224.0.1.75:10000
Dss Key 7 Label          :Page All
Dss Key 7 Line           :0
```

### Expansion Module (Sidecar) Keys

For phones with expansion modules:

```ini
<<EXT KEY CONFIG MODULE>>
## Expansion module 1, key 1
Ext Key 1 Type           :3
Ext Key 1 Value          :1005
Ext Key 1 Label          :Warehouse
Ext Key 1 Line           :1
```

---

## Features

### Call Forward

```ini
<<SIP CONFIG MODULE>>
## Always forward
SIP1 Always Forward Enable   :0
SIP1 Always Forward Target   :

## Busy forward
SIP1 Busy Forward Enable     :1
SIP1 Busy Forward Target     :*991001

## No answer forward
SIP1 Noanswer Forward Enable :1
SIP1 Noanswer Forward Target :*991001
SIP1 Noanswer Forward Timeout:20
```

### DND

```ini
<<PHONE CONFIG MODULE>>
DND Mode                 :0        ## 0=Off, 1=On
```

### Call Waiting

```ini
<<PHONE CONFIG MODULE>>
Call Waiting Enable      :1
Call Waiting Tone        :1
```

### Action URL

```ini
<<ACTION URL MODULE>>
## Webhook on call events
Incoming Call            :http://crm.example.com/api/call/incoming?caller=$caller_number
Call Established         :http://crm.example.com/api/call/start?caller=$caller_number
Call Terminated          :http://crm.example.com/api/call/end?caller=$caller_number
```

### Auto Provision Settings

```ini
<<AUTO PROVISION MODULE>>
## Provisioning server URL
Config Server Path       :https://prov.athena.net.au/fanvil/
## Protocol: 0=Disabled, 1=TFTP, 2=FTP, 3=HTTP, 4=HTTPS
Config Server Protocol   :4
## Provisioning mode
Power On                 :1        ## Provision on boot
Repeat Enable            :1        ## Provision periodically
Repeat Interval          :1440     ## Minutes (24 hours)
## PnP
PNP Enable               :1
## DHCP option
DHCP Option Enable       :1
## Config encryption key (if using encrypted configs)
Config Encryption Key    :
```

### Phone Directory

```ini
<<PHONEBOOK CONFIG MODULE>>
## Remote phonebook
Remote Phonebook 1 URL   :https://prov.athena.net.au/directory/company.xml
Remote Phonebook 1 Name  :Company Directory
```

---

## Firmware Management

```ini
<<AUTO PROVISION MODULE>>
## Firmware URL
Firmware Server Path     :https://prov.athena.net.au/fanvil/firmware/
Firmware File Name       :X4U_2.12.16.3.z
## Protocol (same as config server)
Firmware Server Protocol :4

## Auto upgrade: 0=Disabled, 1=Enabled
Firmware Upgrade Enable  :1
```

### Firmware File Naming

Fanvil firmware files use `.z` (compressed) format:

```
/provisioning/fanvil/firmware/
├── X3U_2.12.16.3.z
├── X4U_2.12.16.3.z
├── X5U_2.12.16.3.z
└── X6U_2.12.16.3.z
```

---

## Fanvil FDPS (Device Provisioning Service)

FDPS is Fanvil's cloud redirect service, similar to Yealink RPS.

### How It Works

```
Phone boots → Contacts FDPS (fdps.fanvil.com) → FDPS returns your URL → Phone downloads config
```

### Setup

1. Register at `https://fdps.fanvil.com`
2. Create a group with your provisioning server URL
3. Add devices by MAC address to the group
4. Configure the "Configuration File Name" in the group settings:
   - Older models (X4, X5): `prov/fanvil.txt`
   - Newer models (X4U, X5U): `prov/fanvil-sysconf.xml`
5. Factory reset the phone — on boot, it contacts FDPS

### FDPS Credentials in PBX

Some PBX platforms (3CX, Vodia) can integrate with FDPS directly. Configure the FDPS
credentials in the PBX admin panel.

---

## Common Models

| Model | Lines | DSS Keys | Screen | PoE | Bluetooth | WiFi | Notes |
|-------|-------|----------|--------|-----|-----------|------|-------|
| X1SP | 2 | 0 | N/A | Yes | No | No | Budget, no screen |
| X3U | 6 | 2 | 2.4" Color | Yes | No | No | Entry level |
| X4U | 12 | 3+12 | 2.8"+2.4" Color | Yes | Yes | No | Popular mid-range |
| X5U | 16 | 4+8 | 3.5"+2.4" Color | Yes | Yes | No | Executive |
| X6U | 20 | 5+30 | 4.3"+2×2.4" Color | Yes | Yes | No | Receptionist |
| X7C | 20 | 0 | 5" Touch | Yes | Yes | No | Video-capable |
| X210 | 20 | 10+96 | 4.3"+2×3.5" Color | Yes | Yes | No | Super-receptionist |
| X301W | 4 | 2 | 2.4" Color | Yes | No | Yes | WiFi entry |
| X303W | 6 | 6 | 2.8" Color | Yes | No | Yes | WiFi mid |
| V62 | 6 | 6 | 2.4" Color | Yes | No | No | Router phone |
| V64 | 12 | 12 | 3.5" Color | Yes | Yes | No | Router phone |

**Router phones** (V62, V64, V65): These include a built-in SBC/router, useful for remote
workers — SIP traffic is tunneled over a single TCP port.

---

## Troubleshooting

### Phone Not Downloading Config

1. Check auto provision settings: Web UI → Maintenance → Auto Provision
2. Verify Config Server Path and Protocol are correct
3. Test URL accessibility: `curl -v https://prov.example.com/fanvil/{mac}.cfg`
4. Check DHCP Option 66 if using DHCP provisioning
5. Check the version header — if it matches the last provisioned version, the phone won't
   re-download. Increment the version number.

### Config Not Applying

1. **Module header missing**: Every parameter MUST have its parent module header present
   in the config file. If `<<SIP CONFIG MODULE>>` is missing, no SIP parameters will apply.
2. **Version check**: The `##File Version: X.XXXX` must be different from the last applied
   version. Increment it.
3. **Syntax**: Ensure exact spacing around `:` separators. Parameter names are case-sensitive.

### Phone Stuck in Provisioning Loop

1. Config file has a firmware URL pointing to a non-existent file
2. Config file has a syntax error causing partial application and reboot
3. Remove the firmware URL from config, or fix the firmware path
4. Factory reset: Press OK button for 10+ seconds, or Web UI → Maintenance → Factory Reset

### BLF Not Working

1. Verify DSS key type is set to 3 (BLF), not 2 (Speed Dial)
2. Check the Line parameter matches the SIP account that has BLF subscription capability
3. Verify PBX supports BLF subscribe for the target extension
4. Check `SIP1 BLF List URI` if using a BLF list from the PBX

### FDPS Not Redirecting

1. Verify MAC address is registered in FDPS portal
2. Check FDPS group has the correct provisioning URL
3. Ensure the phone has internet access to reach fdps.fanvil.com
4. Some older models need multiple factory resets during initial FDPS provisioning
5. Verify the "Configuration File Name" in FDPS matches what your server serves
