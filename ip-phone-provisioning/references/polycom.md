# Polycom / Poly VVX Configuration & Provisioning Reference

## Table of Contents

1. [Provisioning Architecture](#provisioning-architecture)
2. [Master Configuration File](#master-config)
3. [Per-Phone Configuration](#per-phone-config)
4. [SIP Account Configuration](#sip-config)
5. [Codec Configuration](#codec-config)
6. [Network & VLAN](#network-config)
7. [Line Keys & BLF](#line-keys)
8. [Features](#features)
9. [Firmware Management](#firmware)
10. [Poly ZTP (Zero Touch Provisioning)](#ztp)
11. [Common Models](#models)
12. [Troubleshooting](#troubleshooting)

---

## Provisioning Architecture

Polycom uses a master config file that references other config files. The phone downloads
the master config first, then each referenced file.

```
Phone boots
  ↓
Obtains provisioning URL (DHCP Opt 66/160, ZTP, or manual)
  ↓
Downloads master config: 000000000000.cfg
  ↓
Master config lists other config files to load (APPLICATION directive)
  ↓
Phone downloads each referenced file in order
  ↓
First occurrence of a parameter wins (NOT last — opposite to Yealink!)
  ↓
Downloads {MAC}-phone.cfg for per-phone overrides
  ↓
Phone applies config
```

**CRITICAL DIFFERENCE FROM YEALINK**: In Polycom, the FIRST occurrence of a parameter takes
precedence. Files listed earlier in the master config override files listed later. This means
per-phone configs should be listed FIRST, and global defaults LAST.

### Default Credentials

| Access | Username | Password |
|--------|----------|----------|
| Web UI (admin) | admin | 456 (older) or Polycom456 (newer) |
| Web UI (user) | Polycom | 456 |
| Phone menu (advanced) | — | 456 |

**Change these immediately** during provisioning.

---

## Master Configuration File

### `000000000000.cfg`

This is the master config file that all Polycom phones look for by default:

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<!-- Master Configuration File -->
<!-- Files listed here are downloaded in order. FIRST match for any -->
<!-- parameter takes precedence — list specific files BEFORE general ones. -->

<APPLICATION
  APP_FILE_PATH="sip.ld"
  CONFIG_FILES="{MAC}-phone.cfg, site.cfg, features.cfg, sip-interop.cfg"
  MISC_FILES=""
  LOG_FILE_DIRECTORY=""
  OVERRIDES_DIRECTORY=""
  CONTACTS_DIRECTORY=""
  LICENSE_DIRECTORY=""
/>
```

### Key Attributes

| Attribute | Purpose |
|-----------|---------|
| `APP_FILE_PATH` | Firmware executable (default `sip.ld`) |
| `CONFIG_FILES` | Comma-separated list of config files to download |
| `MISC_FILES` | Additional files (wallpapers, ringtones) |
| `LOG_FILE_DIRECTORY` | Where to upload logs (if configured) |
| `OVERRIDES_DIRECTORY` | Directory for phone-generated override files |
| `CONTACTS_DIRECTORY` | Directory for contact/directory files |

### `{MAC}` Placeholder

Polycom replaces `{MAC}` with the phone's MAC address in UPPERCASE with no separators.
Example: `0004F2ABCDEF-phone.cfg`

---

## Per-Phone Configuration

### `{MAC}-phone.cfg`

Per-phone settings. This file is listed FIRST in CONFIG_FILES so its parameters take priority.

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<PHONE_CONFIG>
  <!-- SIP Registration -->
  <reg
    reg.1.displayName="Josh Finlay"
    reg.1.label="1001"
    reg.1.address="1001"
    reg.1.auth.userId="1001"
    reg.1.auth.password="SipP@ssw0rd!"
    reg.1.server.1.address="pbx.athena.net.au"
    reg.1.server.1.port="5060"
    reg.1.server.1.transport="UDPOnly"
  />
</PHONE_CONFIG>
```

### Site-Wide Config: `site.cfg`

Settings common to all phones at a site:

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<PHONE_CONFIG>
  
  <!-- SIP Server (default for all phones — per-phone overrides this) -->
  <reg
    reg.1.server.1.address="pbx.athena.net.au"
    reg.1.server.1.port="5060"
    reg.1.server.1.transport="UDPOnly"
    reg.1.server.1.register="1"
    reg.1.server.1.expires="3600"
  />

  <!-- Backup SIP server -->
  <reg
    reg.1.server.2.address="pbx-backup.athena.net.au"
    reg.1.server.2.port="5060"
    reg.1.server.2.transport="UDPOnly"
    reg.1.server.2.register="1"
  />

  <!-- Admin password (CHANGE THIS) -->
  <device
    device.set="1"
    device.auth.localAdminPassword="Y0urStr0ngP@ss!"
    device.auth.localUserPassword="UserP@ss123"
  />

  <!-- Time and locale -->
  <lcl
    lcl.datetime.time.24HourClock="1"
    lcl.datetime.date.format="DD/MM/YYYY"
  />

  <tcpIpApp
    tcpIpApp.sntp.address="au.pool.ntp.org"
    tcpIpApp.sntp.gmtOffset="36000"
  />
  <!-- gmtOffset is in seconds: 36000 = +10 hours (AEST) -->

  <!-- Tone scheme -->
  <se
    se.rt.ringPattern.default="ringer2"
  />

  <!-- Regional tone set -->
  <tone
    tone.chord.ringer.1.freq.1="400"
    tone.chord.ringer.1.freq.2="450"
  />

</PHONE_CONFIG>
```

---

## SIP Account Configuration

### Full Account Parameters

```xml
<reg
  <!-- Account 1 enable -->
  reg.1.server.1.register="1"
  
  <!-- Identity -->
  reg.1.displayName="Josh Finlay"
  reg.1.label="1001"
  reg.1.address="1001"
  reg.1.type="private"
  
  <!-- Authentication -->
  reg.1.auth.userId="1001"
  reg.1.auth.password="SipP@ssw0rd!"
  reg.1.auth.domain=""
  
  <!-- SIP Server -->
  reg.1.server.1.address="pbx.athena.net.au"
  reg.1.server.1.port="5060"
  reg.1.server.1.transport="UDPOnly"
  reg.1.server.1.register="1"
  reg.1.server.1.expires="3600"
  reg.1.server.1.retryMaxCount="3"
  
  <!-- Outbound proxy -->
  reg.1.outboundProxy.address=""
  reg.1.outboundProxy.port="5060"
  reg.1.outboundProxy.transport="UDPOnly"
  
  <!-- Caller ID -->
  reg.1.ringType="ringer2"
  reg.1.lineKeys="1"
/>
```

### Transport Options

| Value | Protocol |
|-------|----------|
| `UDPOnly` | SIP over UDP |
| `TCPOnly` | SIP over TCP |
| `TLS` | SIP over TLS |
| `DNSnaptr` | Use DNS NAPTR/SRV for discovery |

### Multiple Lines

```xml
<!-- Account 2 -->
<reg
  reg.2.server.1.register="1"
  reg.2.displayName="Sales Line"
  reg.2.label="Sales"
  reg.2.address="1050"
  reg.2.auth.userId="1050"
  reg.2.auth.password="SalesP@ss!"
  reg.2.server.1.address="pbx.athena.net.au"
  reg.2.server.1.port="5060"
  reg.2.server.1.transport="UDPOnly"
/>
```

---

## Codec Configuration

```xml
<voice>
  <!-- Codec preference (1 = highest priority) -->
  voice.codecPref.G711_A="1"
  voice.codecPref.G722="2"
  voice.codecPref.G711_Mu="3"
  voice.codecPref.G729_AB="4"
  
  <!-- Available codecs:
       G711_A     (PCMA / G.711 A-law — use for AU)
       G711_Mu    (PCMU / G.711 μ-law)
       G722       (HD Voice)
       G729_AB    (G.729)
       Lin16      (Linear 16-bit PCM)
       Opus       (Opus — newer firmware)
  -->
</voice>

<!-- DTMF -->
<tone
  tone.dtmf.level="-13"
  tone.dtmf.viaRtp.rfc2833Payload="101"
  tone.dtmf.chassis.masking="1"
/>

<!-- DTMF method -->
<voIpProt
  voIpProt.SIP.dtmfViaSignaling.rfc2833="1"
/>
```

---

## Network & VLAN

### VLAN Configuration

```xml
<device>
  <!-- Voice VLAN -->
  device.net.vlanId="100"
  
  <!-- LLDP for dynamic VLAN -->
  device.net.lldpEnabled="1"
  
  <!-- CDP (for Cisco switches) -->
  device.net.cdpEnabled="1"
</device>

<QOS>
  <!-- DSCP values -->
  QOS.Ethernet.rtp.DiffServ="46"
  QOS.Ethernet.callControl.DiffServ="26"
  
  <!-- 802.1p priority -->
  QOS.Ethernet.rtp.UserPriority="5"
  QOS.Ethernet.callControl.UserPriority="3"
</QOS>
```

### PC Port VLAN

```xml
<!-- Pass-through VLAN for PC connected to phone -->
<device
  device.net.vlanId.pc="0"
/>
```

---

## Line Keys & BLF

### Soft Key Configuration

VVX phones have both hard line keys and soft keys. BLF and speed dials are typically
configured as "attendant" or "enhanced feature keys":

```xml
<!-- BLF for extension 1002 on line key position -->
<attendant
  attendant.resourceList.1.address="1002"
  attendant.resourceList.1.label="Jane Smith"
  attendant.resourceList.1.type="normal"
/>

<!-- Speed dial -->
<speed-dial
  speed-dial.1.contact="0412345678"
  speed-dial.1.label="Mobile"
  speed-dial.1.protocol="SIP"
/>
```

### Enhanced Feature Keys (EFK)

For VVX 150/250/350/450 and newer models:

```xml
<efk
  efk.efkparam.1.status="1"
  efk.efkparam.1.mtype="blf"
  efk.efkparam.1.value="1002"
  efk.efkparam.1.label="Jane Smith"
  efk.efkparam.1.line="1"
  
  efk.efkparam.2.status="1"
  efk.efkparam.2.mtype="speedDial"
  efk.efkparam.2.value="0412345678"
  efk.efkparam.2.label="Mobile"
  efk.efkparam.2.line="1"
  
  efk.efkparam.3.status="1"
  efk.efkparam.3.mtype="park"
  efk.efkparam.3.value="701"
  efk.efkparam.3.label="Park 701"
  efk.efkparam.3.line="1"
/>
```

### BLF Key Types

| `mtype` Value | Function |
|---------------|----------|
| `blf` | Busy Lamp Field |
| `speedDial` | Speed dial |
| `park` | Call park |
| `line` | Line appearance |
| `sharedLine` | Shared line appearance |
| `url` | HTTP URL launch |

### Expansion Module (Sidecar)

```xml
<!-- VVX Expansion Module keys -->
<attendant
  attendant.reg="1"
  attendant.behaviors.display.spontaneousCallAppearances.normal="1"
  
  attendant.resourceList.1.address="1005"
  attendant.resourceList.1.label="Warehouse"
  attendant.resourceList.1.type="normal"
/>
```

---

## Features

### Call Forwarding

```xml
<divert
  divert.1.contact=""
  divert.1.autoOnSpecificCaller="0"
  divert.fwd.1.enabled="0"
  divert.busy.1.enabled="1"
  divert.busy.1.contact="*991001"
  divert.noanswer.1.enabled="1"
  divert.noanswer.1.contact="*991001"
  divert.noanswer.1.timeout="20"
/>
```

### Do Not Disturb

```xml
<feature
  feature.doNotDisturb.enable="0"
/>
```

### Call Waiting

```xml
<call
  call.callWaiting.enable="1"
  call.callWaiting.ring="playTone"
/>
```

### Voicemail

```xml
<msg
  msg.mwi.1.subscribe="1"
  msg.mwi.1.callBack="*97"
  msg.mwi.1.callBackMode="contact"
/>
```

### Directory

```xml
<!-- Corporate directory via LDAP -->
<dir
  dir.corp.address="ldap.athena.net.au"
  dir.corp.port="389"
  dir.corp.transport="TCP"
  dir.corp.baseDN="ou=people,dc=athena,dc=net,dc=au"
  dir.corp.filterPrefix=""
  dir.corp.scope="sub"
  dir.corp.attribute.1.name="cn"
  dir.corp.attribute.1.label="Name"
  dir.corp.attribute.1.type="first_name"
  dir.corp.attribute.2.name="telephoneNumber"
  dir.corp.attribute.2.label="Phone"
  dir.corp.attribute.2.type="phone_number"
/>
```

---

## Firmware Management

### Firmware Structure

Polycom firmware comes in "split" format — multiple files that go in the provisioning directory:

```
/provisioning/polycom/
├── 000000000000.cfg        # Master config
├── sip.ld                  # Main application (auto-detected)
├── 3111-40000-001.sip.ld   # Model-specific firmware (VVX 150)
├── 3111-48830-001.sip.ld   # Model-specific firmware (VVX 350)
├── 3111-48840-001.sip.ld   # Model-specific firmware (VVX 450)
└── ... etc
```

The phone automatically searches for `sip.ld` and its model-specific firmware file.

### Firmware Version Control

```xml
<!-- In master config, specify firmware path -->
<APPLICATION
  APP_FILE_PATH="firmware/6.4.3/sip.ld"
  CONFIG_FILES="{MAC}-phone.cfg, site.cfg"
/>
```

### Staged Firmware Rollout

Create separate master configs per firmware version and point different phone groups to them:

```xml
<!-- 000000000000-prod.cfg — production firmware -->
<APPLICATION APP_FILE_PATH="firmware/6.4.2/sip.ld" CONFIG_FILES="..." />

<!-- 000000000000-test.cfg — testing firmware -->
<APPLICATION APP_FILE_PATH="firmware/6.4.3/sip.ld" CONFIG_FILES="..." />
```

---

## Poly ZTP (Zero Touch Provisioning)

### How It Works

```
Phone boots → Contacts Poly ZTP cloud → ZTP returns provisioning URL → Phone downloads config
```

### Setup

1. Access Poly Lens (lens.poly.com) or the legacy ZTP portal
2. Add devices by MAC address or import from spreadsheet
3. Assign a provisioning server URL per device or group
4. Factory reset the phone — on boot, it contacts ZTP

### DHCP Fallback

If ZTP is not used, the phone falls back to:
1. DHCP Option 160 (provisioning URL)
2. DHCP Option 66 (TFTP/HTTP server)
3. Manual configuration

---

## Common Models

| Model | Lines | Keys | Screen | PoE | Bluetooth | WiFi |
|-------|-------|------|--------|-----|-----------|------|
| VVX 150 | 2 | 2 | 2.5" BW | Yes | No | No |
| VVX 250 | 4 | 4 | 2.8" Color | Yes | Yes | Yes |
| VVX 350 | 6 | 6 | 3.5" Color | Yes | Yes | Yes |
| VVX 450 | 12 | 12 | 4.3" Color | Yes | Yes | Yes |
| VVX 501 | 12 | 12 | 3.5" Touch | Yes | Yes | No |
| VVX 601 | 16 | 16 | 4.3" Touch | Yes | Yes | No |

**Note**: Poly was acquired by HP in 2023. Newer models may be branded HP Poly.
The VVX x50 series (150/250/350/450) are the current generation.

---

## Troubleshooting

### Phone Not Finding Config

1. Check the provisioning server type on the phone:
   Settings → Advanced (password: 456) → Admin Settings → Network Config → Provisioning Server
2. Verify server type matches URL scheme (HTTPS requires Server Type = HTTPS)
3. Check `000000000000.cfg` is at the root of the provisioning URL
4. Verify file permissions on the server

### Config File Parsing Errors

1. Polycom is very strict about XML formatting — validate with an XML parser
2. Use a tool like XML Notepad or `xmllint` to check syntax
3. Check phone status: Home → Settings → 4 → 1 → 3 shows provisioning status, file counts, errors

### Phone Won't Upgrade Firmware

1. Verify `sip.ld` is in the correct path specified by `APP_FILE_PATH`
2. Polycom won't downgrade firmware unless `prov.upgradeServer.force.enabled="1"` is set
3. Check free space on phone — firmware files can be large

### Parameter Not Taking Effect

1. Remember: FIRST occurrence wins in Polycom. If the parameter exists in an earlier file,
   it won't be overridden by a later file.
2. List per-phone config FIRST in CONFIG_FILES
3. Check for typos in parameter names — Polycom silently ignores unknown parameters
