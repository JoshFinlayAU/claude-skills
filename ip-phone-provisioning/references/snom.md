# Snom D-Series Configuration & Provisioning Reference

## Table of Contents

1. [Provisioning Architecture](#provisioning-architecture)
2. [Configuration File Format](#config-format)
3. [Permission Flags](#permission-flags)
4. [SIP Account Configuration](#sip-config)
5. [Codec Configuration](#codec-config)
6. [Network & VLAN](#network-config)
7. [Function Keys](#function-keys)
8. [Features](#features)
9. [Firmware Management](#firmware)
10. [Snom SRAPS (Redirect Service)](#sraps)
11. [Common Models](#models)
12. [Troubleshooting](#troubleshooting)

---

## Provisioning Architecture

Snom uses a `setting_server` parameter to define where the phone fetches its config.
The provisioning order is configurable:

```
Phone boots
  ↓
Tries provisioning sources in order (configurable via provisioning_order):
  1. SRAPS (Snom Redirect and Provisioning Service) — cloud redirect
  2. SIP PnP (Plug and Play multicast)
  3. DHCP (Option 66/67)
  4. TR-069
  ↓
Downloads setting file from the resolved URL
  ↓
Setting file may reference additional files via <setting-files> / <file> tags
  ↓
Phone applies settings
```

### URL Variables

Snom supports variables in the setting server URL:

| Variable | Resolves To | Example |
|----------|------------|---------|
| `{mac}` | MAC address (lowercase, no colons) | `000413920a74` |
| `{model}` | Phone model | `D785` |
| `{fw_version}` | Current firmware version | `10.1.140.15` |
| `{ip}` | Phone's IP address | `10.0.100.42` |

**Example setting_server URLs**:
- Per-phone: `http://prov.example.com/snom/{mac}.xml`
- Model-based: `http://prov.example.com/snom/{model}/{mac}.xml`
- Combined: `http://prov.example.com/snom/{model}/general.xml`

### Default Credentials

| Access | Username | Password |
|--------|----------|----------|
| Web UI | admin | admin (or blank on older firmware) |
| Phone menu (advanced) | — | 0000 |

---

## Configuration File Format

Snom uses XML configuration files with a specific structure:

### Basic Structure

```xml
<?xml version="1.0" encoding="utf-8"?>
<settings>
  <phone-settings>
    <!-- Core phone parameters go here -->
    <language perm="RW">English</language>
    <timezone perm="R">AUS-10</timezone>
    <tone_scheme perm="R">Australia</tone_scheme>
    <ntp_server perm="">au.pool.ntp.org</ntp_server>
  </phone-settings>

  <functionKeys>
    <!-- Function key definitions -->
  </functionKeys>

  <tbook>
    <!-- Phone book entries -->
  </tbook>

  <dialplan>
    <!-- Dial plan rules -->
  </dialplan>

  <setting-files>
    <!-- References to additional config files -->
    <file url="http://prov.example.com/snom/firmware.xml" />
  </setting-files>
</settings>
```

### Modular Structure with Setting Files

For larger deployments, split configs into multiple files:

```xml
<?xml version="1.0" encoding="utf-8"?>
<settings>
  <phone-settings>
    <!-- Minimal settings here; bulk in referenced files -->
    <setting_server perm="">http://prov.example.com/snom/{mac}.xml</setting_server>
  </phone-settings>

  <setting-files>
    <!-- Global settings -->
    <file url="http://prov.example.com/snom/global.xml" />
    <!-- Model-specific -->
    <file url="http://prov.example.com/snom/{model}/model.xml" />
    <!-- Firmware -->
    <file url="http://prov.example.com/snom/{model}/firmware.xml" />
  </setting-files>
</settings>
```

### Indexed Parameters

Some parameters (SIP accounts, function keys) use an `idx` attribute:

```xml
<phone-settings>
  <!-- Account 1 -->
  <user_active idx="1" perm="">true</user_active>
  <user_name idx="1" perm="">1001</user_name>
  <user_host idx="1" perm="">pbx.athena.net.au</user_host>
  <user_pass idx="1" perm="">SipP@ssw0rd!</user_pass>

  <!-- Account 2 -->
  <user_active idx="2" perm="">true</user_active>
  <user_name idx="2" perm="">1050</user_name>
  <user_host idx="2" perm="">pbx.athena.net.au</user_host>
  <user_pass idx="2" perm="">SalesP@ss!</user_pass>

  <!-- Wildcard: apply to ALL indexes -->
  <user_dtmf_info idx="*" perm="">off</user_dtmf_info>
</phone-settings>
```

---

## Permission Flags

Snom's killer feature for managed deployments. The `perm` attribute controls what the user
can change:

| Flag | Meaning |
|------|---------|
| `""` (empty) | Writable by admin and provisioning, not shown to user |
| `"R"` | Read-only — user can see but not change |
| `"RW"` | Read-write — user can see and change |
| `"!"` | Hidden — not visible in web UI at all |
| `"&"` | Protected — only changeable via provisioning |

**Best practice**: Use `perm=""` or `perm="!"` for security-critical settings (SIP passwords,
provisioning URLs, admin passwords). Use `perm="RW"` for user-customisable settings (ringtone,
volume, display preferences).

```xml
<phone-settings>
  <!-- Users cannot see or change the SIP password -->
  <user_pass idx="1" perm="!">SipP@ssw0rd!</user_pass>
  
  <!-- Users can see the server but not change it -->
  <user_host idx="1" perm="R">pbx.athena.net.au</user_host>
  
  <!-- Users can change their ringtone -->
  <user_ringer idx="1" perm="RW">Ringer1</user_ringer>
  
  <!-- Provisioning URL is protected — only provisioning can change it -->
  <setting_server perm="&">http://prov.example.com/snom/{mac}.xml</setting_server>
</phone-settings>
```

---

## SIP Account Configuration

### Full Account Parameters

```xml
<phone-settings>
  <!-- Enable account -->
  <user_active idx="1" perm="">true</user_active>
  
  <!-- Identity -->
  <user_realname idx="1" perm="">Josh Finlay</user_realname>
  <user_name idx="1" perm="">1001</user_name>
  <user_phone idx="1" perm="">1001</user_phone>
  
  <!-- Authentication -->
  <user_pass idx="1" perm="!">SipP@ssw0rd!</user_pass>
  
  <!-- SIP server -->
  <user_host idx="1" perm="">pbx.athena.net.au</user_host>
  <user_outbound idx="1" perm="">pbx.athena.net.au</user_outbound>
  
  <!-- SIP port -->
  <user_srtp idx="1" perm="">off</user_srtp>
  
  <!-- Registration expiry -->
  <user_expiry idx="1" perm="">3600</user_expiry>
  
  <!-- Transport: udp, tcp, tls -->
  <user_sipusername_as_line idx="1" perm="">on</user_sipusername_as_line>
  
  <!-- NAT keepalive -->
  <user_server_type idx="1" perm="">default</user_server_type>
  
  <!-- Caller ID -->
  <user_dp_str idx="1" perm="">1001</user_dp_str>
  
  <!-- DTMF: off = RFC2833, on = SIP INFO -->
  <user_dtmf_info idx="1" perm="">off</user_dtmf_info>
  
  <!-- Subscribe to MWI (voicemail indicator) -->
  <user_subscribe_mwi idx="1" perm="">on</user_subscribe_mwi>
  <user_mailbox idx="1" perm="">*97</user_mailbox>
</phone-settings>
```

### SRTP

```xml
<phone-settings>
  <!-- SRTP: off, on (mandatory), optional -->
  <user_srtp idx="1" perm="">optional</user_srtp>
</phone-settings>
```

---

## Codec Configuration

```xml
<phone-settings>
  <!-- Codec priority per account (comma-separated, order = priority) -->
  <!-- Available: pcma, pcmu, g722, g729, g726-32, gsm, ilbc, opus -->
  <codec_priority_list idx="1" perm="">pcma,g722,pcmu,g729</codec_priority_list>
  
  <!-- Or configure individually: -->
  <codec_pcma idx="1" perm="">1</codec_pcma>    <!-- 1=enabled, 0=disabled -->
  <codec_pcmu idx="1" perm="">2</codec_pcmu>
  <codec_g722 idx="1" perm="">3</codec_g722>
  <codec_g729 idx="1" perm="">4</codec_g729>
  
  <!-- DTMF RTP payload type -->
  <dtmf_payload_type perm="">101</dtmf_payload_type>
</phone-settings>
```

---

## Network & VLAN

```xml
<phone-settings>
  <!-- Voice VLAN -->
  <vlan_id perm="">100</vlan_id>
  <vlan_qos perm="">5</vlan_qos>          <!-- 802.1p priority -->
  
  <!-- PC port VLAN -->
  <vlan_pc_id perm="">0</vlan_pc_id>       <!-- 0 = untagged -->
  
  <!-- LLDP -->
  <lldp_enable perm="">on</lldp_enable>
  
  <!-- CDP -->
  <cdp_enable perm="">on</cdp_enable>
  
  <!-- QoS/DSCP -->
  <tos_rtp perm="">184</tos_rtp>            <!-- 184 decimal = 0xB8 = DSCP 46 (EF) -->
  <tos_sip perm="">104</tos_sip>            <!-- 104 decimal = 0x68 = DSCP 26 (AF31) -->
  
  <!-- 802.1x -->
  <dot1x_enable perm="">off</dot1x_enable>
  <!-- dot1x_identity, dot1x_password for EAP-MD5 -->
  
  <!-- STUN -->
  <stun_enabled perm="">off</stun_enabled>
  <stun_server perm="">stun.example.com</stun_server>
</phone-settings>
```

**Note on TOS values**: Snom uses the full TOS byte value (8 bits), not just the DSCP value
(6 bits). DSCP 46 = binary 101110 → shifted left by 2 = 10111000 = 184 decimal.

---

## Function Keys

Snom function keys use the `<functionKeys>` XML section with numbered `fkey` elements:

```xml
<functionKeys>
  <!-- Function key 0: Line key (account 1) -->
  <fkey idx="0" context="active" perm="">line</fkey>
  <fkey_label idx="0" perm="">Line 1</fkey_label>
  
  <!-- Function key 1: BLF for extension 1002 -->
  <fkey idx="1" context="active" perm="">blf sip:1002@pbx.athena.net.au</fkey>
  <fkey_label idx="1" perm="">Jane Smith</fkey_label>
  
  <!-- Function key 2: Speed dial -->
  <fkey idx="2" context="active" perm="">speed 0412345678</fkey>
  <fkey_label idx="2" perm="">Mobile</fkey_label>
  
  <!-- Function key 3: BLF for extension 1003 -->
  <fkey idx="3" context="active" perm="">blf sip:1003@pbx.athena.net.au</fkey>
  <fkey_label idx="3" perm="">Reception</fkey_label>
  
  <!-- Function key 4: Call park -->
  <fkey idx="4" context="active" perm="">orbit 701</fkey>
  <fkey_label idx="4" perm="">Park 701</fkey_label>
  
  <!-- Function key 5: DTMF sequence -->
  <fkey idx="5" context="active" perm="">dtmf *21001#</fkey>
  <fkey_label idx="5" perm="">Fwd to 1001</fkey_label>
  
  <!-- Function key 6: URL action -->
  <fkey idx="6" context="active" perm="">url http://crm.example.com/lookup</fkey>
  <fkey_label idx="6" perm="">CRM Lookup</fkey_label>
</functionKeys>
```

### Function Key Types

| Type | Syntax | Purpose |
|------|--------|---------|
| `line` | `line` | Line appearance for an account |
| `blf` | `blf sip:ext@server` | Busy Lamp Field |
| `speed` | `speed number` | Speed dial |
| `orbit` | `orbit code` | Call park orbit |
| `dtmf` | `dtmf digits` | Send DTMF sequence |
| `url` | `url http://...` | Open URL / action URL |
| `button` | `button forward` | Toggle feature (forward, DND, etc.) |
| `xml` | `xml http://...` | XML browser application |
| `multicast` | `multicast 224.0.1.75:10000` | Multicast paging |
| `transfer` | `transfer number` | Blind transfer to number |
| `redirect` | `redirect number` | Redirect incoming call |
| `icom` | `icom` | Intercom call |
| `park+orbit` | `park+orbit 701` | Park call on orbit |
| `pickup` | `pickup ext` | Directed call pickup |

### Key Index Ranges by Model

| Model | Built-in Keys | Expansion Keys |
|-------|--------------|----------------|
| D715 | 0-4 | N/A |
| D735 | 0-31 | 0-17 per module |
| D765 | 0-15 | 0-17 per module |
| D785 | 0-23 | 0-17 per module |
| D862 | 0-5 | 0-17 per module |
| D865 | 0-11 | 0-17 per module |

---

## Features

### Call Forward

```xml
<phone-settings>
  <!-- Always forward -->
  <user_divert_all_enabled idx="1" perm="RW">off</user_divert_all_enabled>
  <user_divert_all idx="1" perm="RW"></user_divert_all>
  
  <!-- Busy forward -->
  <user_divert_busy_enabled idx="1" perm="RW">on</user_divert_busy_enabled>
  <user_divert_busy idx="1" perm="RW">*991001</user_divert_busy>
  
  <!-- No answer forward -->
  <user_divert_noanswer_enabled idx="1" perm="RW">on</user_divert_noanswer_enabled>
  <user_divert_noanswer idx="1" perm="RW">*991001</user_divert_noanswer>
  <user_divert_noanswer_timeout idx="1" perm="RW">20</user_divert_noanswer_timeout>
</phone-settings>
```

### Do Not Disturb

```xml
<phone-settings>
  <dnd_mode perm="RW">off</dnd_mode>
</phone-settings>
```

### Phone Directory (Address Book)

```xml
<tbook>
  <item context="active" type="sip" fav="false">
    <name>Josh Finlay</name>
    <number>1001</number>
    <ringtone>system</ringtone>
  </item>
  <item context="active" type="sip" fav="false">
    <name>Jane Smith</name>
    <number>1002</number>
  </item>
</tbook>
```

---

## Firmware Management

### Firmware via Provisioning

```xml
<phone-settings>
  <!-- Auto-update policy -->
  <update_policy perm="">auto_update</update_policy>
  
  <!-- Firmware URL (direct binary) -->
  <firmware perm="">http://prov.example.com/snom/firmware/snomD785-10.1.140.15.bin</firmware>
  
  <!-- Or: firmware status file (separate XML with firmware URL) -->
  <firmware_status perm="">http://prov.example.com/snom/D785/firmware.xml</firmware_status>
  
  <!-- Check interval (minutes) -->
  <firmware_interval perm="">2880</firmware_interval>
</phone-settings>
```

### Firmware Status File

```xml
<?xml version="1.0" encoding="utf-8"?>
<firmware-settings>
  <firmware perm="">http://prov.example.com/snom/firmware/snomD785-10.1.140.15.bin</firmware>
</firmware-settings>
```

---

## Snom SRAPS (Redirect Service)

Snom phones contact `https://secure-provisioning.snom.com` on first boot by default.

### Setup

1. Register at `https://sraps.snom.com`
2. Add devices by MAC address
3. Set the redirect URL to your provisioning server
4. On factory reset + boot, the phone contacts SRAPS, gets your URL, fetches config

### Disabling SRAPS

To skip SRAPS and go straight to DHCP or manual provisioning:

```xml
<phone-settings>
  <!-- Change provisioning order to skip SRAPS -->
  <provisioning_order perm="">dhcp sippnp</provisioning_order>
  <!-- Default: redirection dhcp sippnp tr069 -->
</phone-settings>
```

Or set `setting_server` directly:

```xml
<phone-settings>
  <setting_server perm="">http://prov.athena.net.au/snom/{mac}.xml</setting_server>
</phone-settings>
```

---

## Common Models

| Model | Lines | Keys | Screen | PoE | Bluetooth | WiFi | Notes |
|-------|-------|------|--------|-----|-----------|------|-------|
| D715 | 4 | 5 | 3.2" BW | Yes | No | No | Budget |
| D717 | 6 | 6 | 3.2" BW | Yes | No | No | Entry |
| D735 | 12 | 32 | 2.7" Color | Yes | Yes | No | Mid-range |
| D765 | 12 | 16 | 3.5" Color | Yes | Yes | No | Upper mid |
| D785 | 12 | 24 | 4.3" Color | Yes | Yes | No | Executive |
| D862 | 4 | 6 | 2.8" Color | Yes | Yes | Yes | Current entry |
| D865 | 12 | 12 | 4.3" Color | Yes | Yes | Yes | Current exec |

---

## Troubleshooting

### Finding Provisioning Logs

Snom provides detailed provisioning logs via the web UI:
- Web UI → System Information → Provisioning (shows last provisioning attempt and results)
- Web UI → System Information → Debug shows more detailed logs

Or via URL: `http://{phone_ip}/info.htm` — shows setting_server, provisioning status

### Config Not Being Applied

1. Check the `perm` flag — if a parameter was provisioned with `perm="R"` previously,
   only provisioning can change it again
2. Check `setting_server` is pointing to the correct URL
3. Check `provisioning_order` — is the phone trying SRAPS first and getting stuck?
4. Verify XML syntax — Snom is strict about well-formed XML
5. Look at provisioning logs for HTTP error codes

### Phone Trying SRAPS and Timing Out

If phones are slow to boot because they're trying SRAPS first:

```xml
<!-- In DHCP-provisioned initial config, change the order -->
<phone-settings>
  <provisioning_order perm="">dhcp sippnp</provisioning_order>
</phone-settings>
```

Or in SRAPS portal, set `provisioning_order` to skip SRAPS on subsequent boots.

### Factory Reset

- Via phone menu: Settings → Maintenance → Reset → Factory Reset (password: 0000)
- Via web UI: Advanced → Maintenance → Factory Reset
- Via URL: `http://{phone_ip}/reset.htm`
- Hard reset: Unplug, hold # key while powering on, wait for "Rescue" menu
