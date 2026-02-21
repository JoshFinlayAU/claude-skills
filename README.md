# Claude Skills: Telecommunications & Network Engineering Reference Library

A collection of Claude AI skills providing comprehensive engineering reference guides for carrier network infrastructure, VoIP systems, IP phone provisioning, and PBX platform configuration.

## Overview

This repository contains specialized skill modules designed to assist telecommunications engineers, network engineers, and system administrators with network infrastructure design, voice systems implementation, and troubleshooting:

| Skill | Description |
|-------|-------------|
| **Carrier Network Engineering** | Carrier-grade network config generation, troubleshooting, and design for MikroTik, Cisco, Juniper, Nokia, and Arista |
| **IP Phone Provisioning** | End-to-end configuration and deployment guides for major IP phone vendors |
| **VoIP & PBX Engineering** | Comprehensive VoIP architecture, protocol, and platform reference |

## Skills

### Carrier Network Engineering

Expert-level network engineering assistance for carrier and enterprise networks, covering all major vendors:

- **MikroTik** — RouterOS configuration
- **Cisco** — IOS, IOS-XE, NX-OS, IOS-XR
- **Juniper** — JunOS
- **Nokia** — SR OS
- **Arista** — EOS

Capabilities include:

- Configuration generation with complete, paste-ready config blocks
- BGP/OSPF/IS-IS/MPLS routing design and implementation
- VLAN/VXLAN/VPLS/L2VPN/L3VPN service provisioning
- PPPoE/IPoE subscriber management and RADIUS integration
- GPON/FTTH deployment and fixed wireless networks
- VRRP/HSRP/GLBP high-availability design
- QoS policy design for voice/video/data prioritisation
- Firewall/ACL rules and control plane protection
- SNMP/NetFlow/sFlow monitoring setup
- Network migrations between vendors
- Carrier peering and traffic engineering
- CGNAT deployment and IPv6 rollout

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
├── carrier-network-engineering/
│   ├── SKILL.md                    # Skill definition
│   └── references/
│       ├── mikrotik.md             # MikroTik RouterOS reference
│       ├── cisco.md                # Cisco IOS/IOS-XE/NX-OS/IOS-XR reference
│       ├── juniper.md              # Juniper JunOS reference
│       ├── nokia.md                # Nokia SR OS reference
│       ├── arista.md               # Arista EOS reference
│       └── protocols.md            # Cross-vendor protocol best practices
│
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

**Carrier Network Engineering triggers:**
- "Configure eBGP peering with our upstream provider"
- "Set up MPLS L3VPN on Juniper"
- "Troubleshoot OSPF adjacency issues on Cisco"
- "Design a VRRP failover for MikroTik"
- "Migrate this Cisco config to Arista"
- "Set up PPPoE subscriber termination"

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
| Total documentation | ~6,000+ lines |
| Reference documents | 17 |
| Network vendor guides | 5 |
| Phone vendor guides | 4 |
| PBX platform guides | 6 |

## Target Audience

- Network engineers and architects
- VoIP engineers and system administrators
- Telecommunications / ISP / carrier professionals
- IT teams deploying IP phone systems
- MSPs managing voice and network infrastructure

## License

This project is proprietary documentation. All rights reserved.
