---
name: ip-phone-provisioning
description: >
  Comprehensive IP phone configuration and auto-provisioning reference for Yealink, Polycom/Poly,
  Snom, and Fanvil desk phones. Use this skill whenever the user is working on phone provisioning,
  endpoint configuration, phone templates, MAC-based config files, DHCP Option 66/67 provisioning,
  zero-touch provisioning (ZTP), RPS/SRAPS redirect services, phone firmware management, BLF/DSS
  key programming, line key configuration, phone VLAN assignment via LLDP-MED or CDP, phone
  security hardening, SIP account setup on handsets, codec configuration on phones, or bulk phone
  deployment. Also trigger for provisioning server setup (HTTP/HTTPS/TFTP/FTP), phone directory
  management, phone XML applications, expansion module (sidecar) configuration, DECT handset
  provisioning, and any brand-specific phone configuration — including Yealink T-series/CP-series,
  Polycom/Poly VVX, Snom D-series, and Fanvil X/XU-series. Trigger even when the user just says
  "phone config", "set up a phone", "provision phones", "phone template", "BLF keys", "line keys",
  "phone won't register", "phone firmware update", "DHCP option 66", "phone keeps rebooting",
  "phone factory reset", or mentions specific model numbers like T54W, VVX450, D785, X4U, etc.
---

# IP Phone Configuration & Auto-Provisioning Skill

A practical engineering reference for configuring and mass-deploying Yealink, Polycom/Poly,
Snom, and Fanvil IP phones — from single-phone setup through to carrier-scale provisioning.

## When to Use This Skill

- **Phone configuration**: SIP account setup, codec preferences, DTMF mode, NAT settings,
  line keys, BLF, speed dials, call park, paging groups
- **Auto-provisioning**: Provisioning server setup, DHCP options, boot files, MAC-based configs,
  model-based configs, config file hierarchy and override logic
- **Bulk deployment**: Zero-touch provisioning, RPS/redirect services, PIN-based provisioning,
  PnP (Plug and Play), TR-069 for phones
- **Firmware management**: Version control, staged rollouts, auto-update policies
- **Phone network config**: Voice VLAN, LLDP-MED, QoS/DSCP on phones, 802.1x
- **Troubleshooting**: Registration failures, provisioning loops, config not applying, firmware
  stuck, BLF not working, audio issues specific to phone config

## Reference Files

Read the appropriate reference file for the vendor you're working with. The provisioning
server reference covers the server-side setup that applies to all vendors.

| File | When to Read |
|------|-------------|
| `references/provisioning-server.md` | Setting up the provisioning server (HTTP/TFTP/FTP), DHCP options, directory structure, security, multi-vendor hosting |
| `references/yealink.md` | Yealink T-series, CP-series, W-series DECT — config parameters, boot files, provisioning hierarchy, RPS, templates |
| `references/polycom.md` | Polycom/Poly VVX series — master config, per-phone config, config merge logic, ZTP, firmware structure |
| `references/snom.md` | Snom D-series — XML config format, permission flags, SRAPS redirect, provisioning order, function keys |
| `references/fanvil.md` | Fanvil X/XU series — config file format, FDPS redirect, module headers, DSS key programming |

## Quick Comparison: Vendor Provisioning

| Feature | Yealink | Polycom/Poly | Snom | Fanvil |
|---------|---------|-------------|------|--------|
| Config format | XML (`.cfg`) | XML (`.cfg`) | XML | cfg/txt/xml |
| Boot file | `y000000000000.boot` | `000000000000.cfg` | N/A (setting_server) | N/A (direct URL) |
| MAC config naming | `{mac}.cfg` | `{MAC}-phone.cfg` | `{mac}.xml` | `{mac}.cfg` |
| Common config | `y000000000000.cfg` | `000000000000.cfg` | Model-based URL | Model common file |
| Model config | `y0000000000XX.cfg` | N/A (in master cfg) | Model in URL path | Model common name |
| Cloud redirect | Yealink RPS | Poly ZTP | Snom SRAPS | Fanvil FDPS |
| Default web password | admin/admin | admin/456 | admin/admin | admin/admin |
| Provisioning protocols | HTTP, HTTPS, TFTP, FTP | HTTP, HTTPS, TFTP, FTP | HTTP, HTTPS, TFTP | HTTP, HTTPS, TFTP, FTP |
| DHCP options | 66, 67, 160 | 66, 160 | 66, 67 | 66 |
| PnP (SIP multicast) | Yes | Yes | Yes | Yes |

## Config File Hierarchy (General Pattern)

All vendors follow a similar layering principle — more specific configs override less specific:

```
┌─────────────────────────────┐
│  MAC-specific config        │  ← Highest priority (per-phone)
├─────────────────────────────┤
│  Model-specific config      │  ← Per phone model
├─────────────────────────────┤
│  Common/Global config       │  ← All phones of this vendor
├─────────────────────────────┤
│  Factory defaults           │  ← Lowest priority
└─────────────────────────────┘
```

Parameters set in higher-priority files override the same parameter in lower-priority files.
This lets you set defaults globally and override per-model or per-phone as needed.

## Essential SIP Parameters (All Vendors)

Every phone needs these core parameters configured. The parameter names vary by vendor but
the concepts are universal:

| Setting | Purpose | Notes |
|---------|---------|-------|
| SIP Server / Registrar | PBX address | IP or FQDN |
| SIP Port | PBX SIP port | Usually 5060 (UDP) or 5061 (TLS) |
| Auth Username | SIP auth username | Often the extension number |
| Auth Password | SIP registration password | Strong, unique per phone |
| Display Name | Caller ID name | Shown on the phone display |
| SIP Username / User ID | SIP URI user part | Often same as auth username |
| Transport | UDP / TCP / TLS | TLS preferred for security |
| Outbound Proxy | Proxy address | If required by network design |
| Codec Priority | Codec preference list | PCMA first for AU deployments |
| DTMF Mode | RFC4733 / SIP INFO / Inband | RFC4733 for nearly all cases |
| Voice VLAN | VLAN ID for voice traffic | Set via LLDP or static config |
| DSCP / QoS | DSCP values for SIP and RTP | EF (46) for RTP, AF31 (26) for SIP |

## Provisioning URL Wildcards

Most vendors support wildcards/variables in the provisioning URL. These are resolved by the
phone at boot time:

| Wildcard | Meaning | Example |
|----------|---------|---------|
| `$MAC` or `{mac}` | Phone MAC address (lowercase) | `001565abcdef` |
| `$PN` | Product name / model | `T54W`, `SIP-T46S` |
| `$FIRMWARE` | Current firmware version | `96.86.0.100` |
| `$SN` | Serial number | Varies |

**Example**: `http://prov.example.com/$PN/{mac}.cfg`
For a Yealink T54W with MAC 001565abcdef, resolves to:
`http://prov.example.com/T54W/001565abcdef.cfg`

## DHCP Options for Provisioning

| Option | Purpose | Format | Example |
|--------|---------|--------|---------|
| 66 | TFTP/provisioning server URL | String | `http://prov.example.com/` |
| 67 | Boot file name | String | `$PN.cfg` |
| 160 | Provisioning URL (vendor-specific) | String | `https://prov.example.com/` |
| 132 | VLAN ID (Cisco-style) | Integer | `100` |
| 133 | VLAN priority | Integer | `5` |

### MikroTik DHCP Option 66 Example

```routeros
# Set DHCP Option 66 for voice VLAN
/ip dhcp-server option
add code=66 name=option66 value="'http://prov.athena.net.au/phones/'"

/ip dhcp-server network
set [find address=10.0.100.0/24] dhcp-option=option66
```

## Provisioning Server Directory Structure

A clean multi-vendor provisioning server layout:

```
/var/www/provisioning/
├── yealink/
│   ├── firmware/
│   │   ├── T54W/
│   │   └── T46S/
│   ├── y000000000000.boot          # Common boot file
│   ├── y000000000000.cfg           # Global config (all Yealink)
│   ├── y0000000000XX.cfg           # Model-specific (XX = model code)
│   ├── {mac}.boot                  # Per-phone boot (optional)
│   └── {mac}.cfg                   # Per-phone config
├── polycom/
│   ├── firmware/
│   │   └── 6.4.3/                  # UCS firmware split files
│   ├── 000000000000.cfg            # Master config
│   ├── site.cfg                    # Site-wide settings
│   ├── {MAC}-phone.cfg             # Per-phone config
│   └── {MAC}-directory.xml         # Per-phone directory
├── snom/
│   ├── firmware/
│   ├── general.xml                 # Common settings
│   ├── {model}-firmware.xml        # Firmware per model
│   └── {mac}.xml                   # Per-phone config
└── fanvil/
    ├── firmware/
    ├── common.cfg                  # Common config
    ├── {model}.cfg                 # Model-specific
    └── {mac}.cfg                   # Per-phone config
```

## Australian-Specific Phone Settings

When deploying phones in Australia, always set:

| Setting | Value | Notes |
|---------|-------|-------|
| Timezone | `Australia/Brisbane` (+10) or relevant city | No DST in QLD |
| Tone scheme | `Australia` | Dial tone, busy tone, ringback |
| Date format | `DD/MM/YYYY` | Australian standard |
| Time format | 12h or 24h | User preference |
| Language | English | |
| Preferred codec | PCMA (G.711 A-law) | AU standard, NOT μ-law |
| Emergency number | 000 | Must be dialable from any phone |
| NTP server | `au.pool.ntp.org` or local | Time sync |

## Security Checklist

Before deploying phones to production:

- [ ] Change default web UI password on every phone (or provision a strong password)
- [ ] Use HTTPS for provisioning server (phones will download passwords over this connection)
- [ ] Enable config file encryption if the vendor supports it
- [ ] Restrict phone web UI access to management VLAN only
- [ ] Disable unused protocols (Telnet, HTTP if HTTPS is available)
- [ ] Set `perm` flags (Snom) or equivalent to prevent users from changing critical settings
- [ ] Use unique SIP registration passwords per phone (not shared)
- [ ] Enable 802.1x if your switch infrastructure supports it
- [ ] Disable SIP ALG on all routers between phones and PBX

## Workflow

1. **Identify the vendor and model** — check the reference for that specific vendor
2. **Read the vendor reference** — provisioning format, parameter names, and quirks differ
3. **Set up the provisioning server** — see `references/provisioning-server.md`
4. **Build config templates** — global → model → per-phone layering
5. **Test with one phone** — verify registration, audio, BLF, DTMF before bulk deployment
6. **Roll out** — use DHCP Option 66 or cloud redirect (RPS/ZTP/SRAPS/FDPS) for zero-touch
7. **Monitor** — check PBX registrations, test calls, verify firmware versions
