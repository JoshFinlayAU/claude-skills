# Protocol Best Practices (Cross-Vendor)

This reference covers protocol design principles and best practices that apply regardless of vendor. Read this alongside the relevant vendor reference for implementation specifics.

## BGP

### Design Principles

**iBGP**: All routers in an AS must have full iBGP mesh OR use route reflectors (RR). For carrier networks, route reflectors are standard. Place RRs on core routers or dedicated RR appliances. Use redundant RRs (minimum 2) in separate clusters for resilience.

**eBGP**: Each transit/peering connection should have:
- Prefix filtering (inbound and outbound)
- Bogon filtering
- Maximum prefix limits
- MD5 authentication
- Explicit route-maps/policies

### Prefix Filtering (Mandatory)

Always filter inbound on all eBGP sessions. At minimum, reject:
- RFC1918 (10/8, 172.16/12, 192.168/16)
- CGNAT space (100.64/10) unless you're using it internally
- Loopback (127/8)
- Link-local (169.254/16)
- IETF test ranges (192.0.2/24, 198.51.100/24, 203.0.113/24)
- Class D/E (224/3)
- Default route (0/0) unless explicitly wanted
- Prefixes longer than /24 (IPv4) or /48 (IPv6)
- Your own prefixes (prevent loops)

For outbound, only advertise your own aggregates and customer prefixes. Never leak transit routes to peers.

### RPKI / ROV

Where supported, enable RPKI Route Origin Validation. This validates that the AS originating a prefix is authorised to do so.

### Communities

Use BGP communities to tag routes with metadata:
- Origin tagging (which peer/transit a route came from)
- Action communities (blackhole, no-export, prepend)
- Customer-defined communities for traffic engineering

Standard well-known communities:
- `no-export` (65535:65281) — don't advertise to eBGP peers
- `no-advertise` (65535:65282) — don't advertise to any peer
- `no-peer` (65535:65284) — don't advertise to bilateral peers

### Timers

| Scenario | Hold Time | Keepalive | Notes |
|----------|-----------|-----------|-------|
| eBGP transit | 30-90s | 10-30s | Balanced stability vs convergence |
| iBGP | 90s (default) | 30s | Stable core, use BFD for fast detection |
| eBGP peering (IXP) | 30s | 10s | Faster convergence at peering points |

Use BFD (Bidirectional Forwarding Detection) alongside BGP for sub-second failure detection without aggressive BGP timers.

### Graceful Restart

Enable BGP graceful restart on all sessions where supported. This maintains forwarding during a BGP process restart, preventing unnecessary route withdrawals.

## OSPF / IS-IS

### Which to Use?

| Factor | OSPF | IS-IS |
|--------|------|-------|
| Common in | Enterprise, smaller carriers | Large carriers, ISPs |
| IPv6 | OSPFv3 (separate process) | Native multi-topology |
| Scalability | Good with proper area design | Better at very large scale |
| Vendor support | Universal | Universal (carrier platforms) |

For most Australian carriers, either works. IS-IS is preferred for MPLS networks due to better integration and simpler multi-topology support.

### OSPF Design

- **Area 0 (backbone)**: All core routers. Keep it tight and well-connected
- **Stub areas**: Use for leaf sites that don't need full external routing
- **Totally stubby**: When the stub area only needs a default route
- **NSSA**: When the stub area needs to inject external routes (e.g. customer redistribution)
- **Reference bandwidth**: Set to match your fastest link (e.g. 100Gbps = `reference-bandwidth 100000`)
- **Point-to-point**: Use on all point-to-point links to avoid DR/BDR election overhead
- **Passive interfaces**: All interfaces that don't need OSPF adjacencies (loopbacks, management, customer-facing)
- **BFD**: Enable on all OSPF inter-router links for fast failure detection

### IS-IS Design

- **Level 2 only**: For flat carrier backbones, Level 2 only is simplest
- **Level 1/Level 2**: Use when you have a hierarchy (core = L2, access = L1)
- **Wide metrics**: Always enable. Narrow metrics (6-bit) are too limiting for modern networks
- **NET format**: `49.AREA.SYSTEM-ID.00` — e.g. `49.0001.0100.0000.0001.00` for router-id 10.0.0.1

## MPLS

### Label Distribution

- **LDP**: Simple, auto-discovers, works well for basic L2/L3VPN. Use for most carrier deployments
- **RSVP-TE**: When you need traffic engineering (explicit paths, bandwidth reservations). More complex
- **Segment Routing**: Modern replacement for LDP/RSVP-TE. Uses source routing with segment IDs. Preferred for new deployments where vendor supports it

### VPN Types

| Type | Use Case | Encapsulation |
|------|----------|---------------|
| L3VPN (VRF) | Customer routed services, internet VRF separation | MPLS labels, VPNv4 BGP |
| VPLS | Multi-point L2 bridging across MPLS core | MPLS pseudowires |
| L2VPN/Epipe | Point-to-point L2 circuits | MPLS pseudowires |
| EVPN | Modern L2/L3 overlay, replaces VPLS | VXLAN or MPLS data plane |

### MTU Considerations

MPLS adds label overhead. Ensure all core-facing interfaces support:
- Standard MPLS: 1504+ bytes (1 label = 4 bytes)
- Stacked labels (VPN): 1508-1520+ bytes
- Recommendation: Set core MTU to 9000+ (jumbo frames) to avoid fragmentation issues

## Subscriber Management (PPPoE / IPoE)

### PPPoE

- **MTU**: PPPoE overhead is 8 bytes. Set PPPoE interface MTU to 1492 (for 1500 byte upstream) or adjust upstream MTU
- **MRU**: Match MTU setting. Some CPE negotiate differently
- **Authentication**: PAP is simple but sends credentials in clear text. CHAP is preferred. MSCHAP2 for Windows compatibility
- **Session limits**: Always set per-interface and per-user session limits
- **Keepalives**: Enable LCP keepalives to detect dead sessions
- **RADIUS**: Use RADIUS for authentication, authorisation, and accounting. Send interim updates (5-minute intervals typical)
- **CoA (Change of Authorization)**: Enable RADIUS CoA for dynamic policy changes (speed changes, disconnects)
- **IP pools**: Use RFC6598 (100.64.0.0/10) for subscriber addressing when using CGNAT

### IPoE (DHCP-based)

- **DHCP relay/server**: Use Option 82 (relay agent information) for subscriber identification
- **Lease times**: Balance between address efficiency and DHCP server load. 1-24 hours typical
- **Address pools**: Size appropriately for subscriber count plus churn

### RADIUS Integration

Essential RADIUS attributes for ISP:
- `Framed-IP-Address` — subscriber IP
- `Framed-IP-Netmask` — subscriber mask
- `Framed-Pool` — IP pool name
- `Session-Timeout` — max session duration
- `Idle-Timeout` — inactivity timeout
- `Acct-Interim-Interval` — accounting update frequency
- Vendor-specific attributes for rate limiting (varies by vendor)

## High Availability

### VRRP/HSRP/GLBP

| Protocol | Standard | Notes |
|----------|----------|-------|
| VRRP | RFC 5798 (v3) | Standards-based, multi-vendor |
| HSRP | Cisco proprietary | v2 supports >255 groups, IPv6 |
| GLBP | Cisco proprietary | Load balancing across gateways |

**Best practice**: Use VRRP for multi-vendor environments. HSRP only in all-Cisco networks.

Key settings:
- **Preemption**: Enable if you want the higher-priority router to always be master
- **Timers**: Default VRRP is 1s advertisement. Sub-second possible but increases CPU
- **Authentication**: Use where supported (VRRP v2 supports simple auth)
- **Interface tracking**: Track uplink interfaces and decrement priority on failure

### BFD (Bidirectional Forwarding Detection)

Enable BFD on all inter-router links. Typical settings:
- Minimum interval: 300ms (carrier), 100ms (data centre)
- Multiplier: 3 (detect failure in ~1 second at 300ms)
- Integrate with BGP, OSPF, IS-IS, and static routes

### Graceful Restart / NSR

- **Graceful restart**: Maintains forwarding during control plane restart
- **Non-stop routing (NSR)**: Keeps routing state across RE switchover (dual-RE platforms)
- Enable both where available for maximum resilience

## QoS

### DSCP Markings (Standard)

| Class | DSCP | PHB | Typical Use |
|-------|------|-----|-------------|
| EF | 46 | Expedited Forwarding | Voice |
| AF41 | 34 | Assured Forwarding | Video |
| AF31 | 26 | Assured Forwarding | Signalling |
| AF21 | 18 | Assured Forwarding | Business data |
| CS6 | 48 | Network control | Routing protocols |
| CS7 | 56 | Network control | Network management |
| BE | 0 | Best Effort | Default |

### Design Principles

- **Trust boundaries**: Mark traffic at the network edge, trust markings in the core
- **Priority queuing**: Reserve for voice/real-time traffic only. Strict priority can starve other classes
- **Bandwidth allocation**: Ensure best-effort always gets minimum bandwidth
- **Policing vs shaping**: Police at ingress, shape at egress where possible
- **Core simplicity**: Keep QoS simple in the core (trust DSCP, apply queuing). Complex classification at the edge

## Security

### Control Plane Protection

Every carrier router should have:
1. **Management ACL/filter**: Restrict SSH/SNMP/HTTP to management networks only
2. **Routing protocol protection**: Only accept BGP/OSPF/IS-IS/LDP from known peers
3. **ICMP rate limiting**: Allow ICMP but rate-limit to prevent abuse
4. **CoPP/RE filter**: Apply to the control plane/loopback to protect the routing engine
5. **uRPF**: Enable unicast reverse path forwarding on customer-facing interfaces (loose mode for transit)

### Infrastructure ACLs

Protect your infrastructure addressing:
- Don't allow customer traffic destined to your loopback range
- Don't allow customer traffic destined to your P2P link addresses
- Only allow management protocols from designated management networks

### SNMP Security

- Use SNMPv3 with authentication and encryption where possible
- If using SNMPv2c, restrict by source IP (ACL)
- Use read-only communities. Never use read-write unless absolutely necessary
- Separate monitoring community from management community

## Monitoring

### Essentials

Every carrier device should export:
- **SNMP**: Interface counters, CPU, memory, routing table size, BGP session state
- **Syslog**: Centralised logging for events, alarms, config changes
- **NetFlow/sFlow/IPFIX**: Traffic flow data for capacity planning and anomaly detection
- **Streaming telemetry**: Where supported (gNMI/gRPC), preferred over SNMP polling for real-time data

### Key Metrics to Monitor

- Interface utilisation (aim for <80% sustained)
- BGP session state and prefix count changes
- OSPF/IS-IS adjacency state
- CPU and memory utilisation
- CRC errors, input/output errors, drops
- Routing table size and churn
- MPLS LSP state
- Power supply and fan status
