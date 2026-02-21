---
name: carrier-network-engineering
description: >
  Carrier-grade network engineering skill covering configuration generation, troubleshooting,
  protocol design, and operational guidance for all major network vendors including MikroTik,
  Cisco (IOS/IOS-XE/NX-OS/IOS-XR), Juniper (JunOS), Nokia (SR OS), and Arista (EOS).
  Use this skill whenever the user asks about router/switch configurations, BGP/OSPF/MPLS/IS-IS
  routing, VLAN/VXLAN/VPLS/L2VPN/L3VPN design, VRRP/HSRP/GLBP redundancy, PPPoE/IPoE subscriber
  management, GPON/FTTH deployments, fixed wireless networks, carrier peering, firewall/ACL rules,
  QoS policies, SNMP/monitoring setup, network migrations, capacity planning, or any network
  infrastructure question — even if the user doesn't specify a vendor. Also trigger for
  troubleshooting commands, show/display/print command references, and config auditing.
  This is a core skill for a telecommunications consulting company.
---

# Carrier Network Engineering

Expert-level network engineering assistance for carrier and enterprise networks.

## Workflow

### 1. Identify the Context

When a network engineering request comes in, determine:

- **Vendor(s)**: Which platform(s) are involved? If unspecified, ask or infer from context.
- **Scope**: Config generation, troubleshooting, design review, migration, or documentation?
- **Protocol domain**: Routing (BGP/OSPF/IS-IS), switching (VLAN/VPLS/VXLAN), access (PPPoE/IPoE/GPON), or a combination?
- **Environment**: Carrier/ISP, enterprise, or data centre?

### 2. Load Vendor Reference

Read the appropriate vendor reference file(s) before generating any configuration:

| Vendor | Reference File |
|--------|---------------|
| MikroTik | `references/mikrotik.md` |
| Cisco (IOS/IOS-XE/NX-OS/IOS-XR) | `references/cisco.md` |
| Juniper (JunOS) | `references/juniper.md` |
| Nokia (SR OS) | `references/nokia.md` |
| Arista (EOS) | `references/arista.md` |

For multi-vendor scenarios (e.g. migration, interop, peering), read all relevant files.

Also read `references/protocols.md` for protocol-specific best practices that apply across vendors.

### 3. Generate Output

Depending on the task:

**Configuration generation:**
- Always produce complete, paste-ready config blocks — not fragments
- Include comments explaining each section
- Use the vendor's native CLI syntax exactly (no pseudo-config)
- Flag any values that need to be customised (use `<PLACEHOLDER>` format)
- If the config interacts with other devices, note what the peer side needs

**Troubleshooting:**
- Start with verification commands (show/display/print) to confirm current state
- Provide a logical diagnostic sequence, not a dump of every possible command
- Explain what each command output tells you and what to look for
- Suggest fixes with rollback-safe approaches where possible

**Design review / migration:**
- Identify risks before suggesting changes
- Provide a change plan with verification steps at each stage
- Include rollback procedures
- Consider traffic impact and maintenance windows

### 4. Output Format

Configs should be in fenced code blocks with the vendor name as the language tag:

```routeros
# MikroTik configs use routeros
```

```ios
! Cisco IOS/IOS-XE configs use ios
```

```junos
# Juniper configs use junos
```

```sros
# Nokia SR OS configs use sros
```

```eos
! Arista EOS configs use eos
```

When producing config files, save them with appropriate extensions:
- MikroTik: `.rsc`
- Cisco: `.cfg` or `.conf`
- Juniper: `.conf` or `.set` (for set-style)
- Nokia: `.cfg`
- Arista: `.cfg`

## Key Principles

**Safety first**: Never generate configs that could cause an outage without explicit warnings and rollback steps. Carrier networks carry live traffic — always think about impact.

**Vendor accuracy**: Each vendor has specific syntax, hierarchy, and commit models. Don't mix vendor syntaxes. If unsure about exact syntax for a specific software version, say so and provide what you're confident about.

**Complete configs**: Provide complete stanzas, not fragments. If configuring BGP, include the full router bgp block, not just the neighbor line. Context matters — the user needs to know where in the config hierarchy each block goes.

**Operational awareness**: Consider that configs need to be applied to live networks. Prefer approaches that minimise disruption — soft resets over hard resets, graceful shutdown before maintenance, commit confirmed where available.

**Australian context**: This skill primarily supports Australian carrier networks. Consider Australian-specific factors like NBN integration, ACMA regulations, numbering plans for VoIP, and common Australian carrier interconnect practices when relevant.

## Protocol Quick Reference

When the request involves specific protocols, also read `references/protocols.md` for cross-vendor best practices on:

- **BGP**: iBGP/eBGP, route reflectors, communities, prefix filtering, RPKI
- **OSPF/IS-IS**: Area design, stub types, route summarisation, BFD
- **MPLS**: LDP, RSVP-TE, L2VPN/L3VPN, VPLS, segment routing
- **Subscriber management**: PPPoE, IPoE, RADIUS integration, CGNAT
- **High availability**: VRRP, HSRP, GLBP, BFD, graceful restart
- **QoS**: Traffic classification, policing, shaping, queuing strategies
- **Security**: ACLs, firewall filters, CoPP/control plane protection, uRPF
- **Monitoring**: SNMP, NetFlow/sFlow/IPFIX, streaming telemetry

## Common Carrier Scenarios

These are the kinds of tasks this skill is built for:

- Configuring eBGP peering with upstream transit providers
- Setting up iBGP mesh or route reflector hierarchy
- MPLS L2VPN/L3VPN service provisioning
- PPPoE/IPoE subscriber termination and RADIUS integration
- VRRP/HSRP gateway redundancy
- VLAN trunking and service delivery over carrier ethernet
- GPON OLT/ONT provisioning
- Fixed wireless backhaul and CPE configuration
- Inter-carrier peering and traffic engineering
- Network migration between vendors (e.g. Cisco to Juniper)
- QoS policy design for voice/video/data prioritisation
- CGNAT deployment and port allocation
- Monitoring and alerting setup (LibreNMS, SNMP, syslog)
- Firewall rule design and control plane protection
- IPv6 deployment alongside existing IPv4 infrastructure
