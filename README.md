# Claude Skills: VoIP & IP Telephony Reference Library

A collection of Claude AI skills providing comprehensive engineering reference guides for VoIP infrastructure, IP phone provisioning, and PBX platform configuration.

## Overview

This repository contains two specialized skill modules designed to assist telecommunications engineers and system administrators with voice infrastructure implementation, configuration, and troubleshooting:

| Skill | Description |
|-------|-------------|
| **IP Phone Provisioning** | End-to-end configuration and deployment guides for major IP phone vendors |
| **VoIP & PBX Engineering** | Comprehensive VoIP architecture, protocol, and platform reference |

## Skills

### IP Phone Provisioning

Vendor-specific configuration guides covering:

- **Yealink** — T-series, CP-series, W-series DECT phones
- **Polycom/Poly** — VVX series
- **Snom** — D-series
- **Fanvil** — X/XU series

Topics include:
- SIP account setup and registration
- Boot files and configuration file hierarchy
- MAC-based, model-based, and global config layers
- BLF/DSS key programming
- Firmware management and zero-touch provisioning (ZTP)
- VLAN assignment via LLDP-MED/CDP
- Provisioning server setup (HTTP/HTTPS/TFTP/FTP)
- DHCP Option 66/67 configuration
- Phone security hardening

### VoIP & PBX Engineering

Core protocol and architecture reference:

- **SIP Protocol** — Message structure, headers, call flows, authentication
- **RTP/RTCP** — Media handling and quality metrics
- **Codecs** — G.711, G.729, G.722, Opus, bandwidth calculations
- **NAT Traversal** — STUN, TURN, ALG considerations
- **Security** — SRTP, TLS, certificate management

Platform-specific configuration guides:

- Asterisk (PJSIP and legacy chan_sip)
- FreeSWITCH
- Kamailio
- 3CX
- FusionPBX
- VitalPBX

Additional topics:
- Voice quality metrics (MOS/R-factor)
- DSCP/QoS marking and enforcement
- Jitter buffer configuration
- Troubleshooting workflows

### Australian Compliance

Dedicated guidance for Australian deployments:

- ACMA regulatory framework
- Number management and porting (LNP)
- Emergency services (000/106) configuration
- CLI and Caller ID rules
- Do Not Call Register compliance
- NBN voice integration

## Repository Structure

```
claude-skills/
├── ip-phone-provisioning/
│   ├── SKILL.md                    # Skill definition
│   └── references/
│       ├── provisioning-server.md  # Server setup guide
│       ├── yealink.md              # Yealink configuration
│       ├── polycom.md              # Polycom/Poly configuration
│       ├── snom.md                 # Snom configuration
│       └── fanvil.md               # Fanvil configuration
│
└── voip-pbx/
    ├── SKILL.md                    # Skill definition
    └── references/
        ├── sip-protocol.md         # SIP fundamentals
        ├── codecs-media.md         # Codec reference
        ├── pbx-platforms.md        # Platform configurations
        ├── troubleshooting.md      # Diagnostic workflows
        ├── network-design.md       # Voice network architecture
        └── australian-voice.md     # ACMA compliance
```

## Usage with Claude

These skills are designed to integrate with Claude AI. When activated, Claude can assist with:

**IP Phone Provisioning triggers:**
- "How do I configure a Yealink phone?"
- "Set up zero-touch provisioning"
- "Configure BLF keys on Polycom"
- "Troubleshoot phone registration issues"

**VoIP & PBX triggers:**
- "Explain SIP INVITE flow"
- "Configure Asterisk PJSIP trunk"
- "Troubleshoot one-way audio"
- "What codecs should I use for low bandwidth?"
- "Set up Australian emergency services"

## Content Statistics

| Metric | Value |
|--------|-------|
| Total documentation | ~6,000 lines |
| Reference documents | 11 |
| Vendor guides | 4 |
| PBX platform guides | 6 |

## Target Audience

- VoIP engineers and system administrators
- Telecommunications professionals
- IT teams deploying IP phone systems
- MSPs managing voice infrastructure

## License

This project is proprietary documentation. All rights reserved.
