# Juniper JunOS Reference

## Platform Overview

JunOS runs on all Juniper platforms: MX (core/edge routers), QFX (data centre switches), EX (enterprise switches), SRX (firewalls), PTX (core transport), ACX (access). JunOS uses a hierarchical, XML-backed configuration with a commit model.

**Key strengths**: Commit/rollback model (up to 50 rollbacks), candidate config editing, commit confirmed, commit synchronize for dual-RE, powerful policy framework.

## CLI Syntax Patterns

### Two Modes
```
user@router> (operational mode — show commands, ping, traceroute)
user@router# (configuration mode — edit config)
```

### Configuration Styles

**Hierarchical (standard)**:
```junos
set interfaces ge-0/0/0 unit 0 family inet address 10.0.0.1/30
```

**Curly-brace (show config output)**:
```junos
interfaces {
    ge-0/0/0 {
        unit 0 {
            family inet {
                address 10.0.0.1/30;
            }
        }
    }
}
```

### Common Operations
```junos
# Enter config mode
configure

# Set a value
set interfaces ge-0/0/0 unit 0 family inet address 10.0.0.1/30

# Delete a value
delete interfaces ge-0/0/0 unit 0 family inet address 10.0.0.1/30

# Navigate hierarchy
edit interfaces ge-0/0/0 unit 0
set family inet address 10.0.0.1/30
top  # back to root
up   # one level up

# Review changes
show | compare

# Commit
commit check        # validate only
commit confirmed 5  # auto-rollback in 5 minutes unless confirmed
commit              # apply
commit and-quit     # apply and exit config mode
commit synchronize  # apply to both REs in dual-RE

# Rollback
rollback 1          # restore previous config
commit              # apply the rollback

# Show config
show configuration interfaces ge-0/0/0
show configuration | display set  # show in set format
```

## Routing

### BGP
```junos
set routing-options router-id 10.0.0.1
set routing-options autonomous-system 64512

# eBGP transit peer
set protocols bgp group TRANSIT type external
set protocols bgp group TRANSIT neighbor 203.0.113.1 peer-as 64500
set protocols bgp group TRANSIT neighbor 203.0.113.1 description "Transit Provider A"
set protocols bgp group TRANSIT neighbor 203.0.113.1 authentication-key "<BGP_PASSWORD>"
set protocols bgp group TRANSIT neighbor 203.0.113.1 hold-time 30
set protocols bgp group TRANSIT neighbor 203.0.113.1 import TRANSIT-IN
set protocols bgp group TRANSIT neighbor 203.0.113.1 export TRANSIT-OUT
set protocols bgp group TRANSIT neighbor 203.0.113.1 family inet unicast

# iBGP with route reflection
set protocols bgp group IBGP type internal
set protocols bgp group IBGP local-address 10.0.0.1
set protocols bgp group IBGP cluster 10.0.0.1
set protocols bgp group IBGP neighbor 10.0.0.2 description "PE2"
set protocols bgp group IBGP family inet unicast
set protocols bgp group IBGP family inet-vpn unicast
set protocols bgp group IBGP family l2vpn signaling
set protocols bgp group IBGP export NHS

# Next-hop-self for iBGP
set policy-options policy-statement NHS term 1 then next-hop self

# Prefix lists
set policy-options prefix-list BOGONS 0.0.0.0/0
set policy-options prefix-list BOGONS 10.0.0.0/8 orlonger
set policy-options prefix-list BOGONS 172.16.0.0/12 orlonger
set policy-options prefix-list BOGONS 192.168.0.0/16 orlonger
set policy-options prefix-list BOGONS 100.64.0.0/10 orlonger

set policy-options prefix-list OUR-PREFIXES 192.0.2.0/24

# Policies
set policy-options policy-statement TRANSIT-IN term REJECT-BOGONS from prefix-list BOGONS
set policy-options policy-statement TRANSIT-IN term REJECT-BOGONS then reject
set policy-options policy-statement TRANSIT-IN term ACCEPT then local-preference 200
set policy-options policy-statement TRANSIT-IN term ACCEPT then accept

set policy-options policy-statement TRANSIT-OUT term OUR-ROUTES from prefix-list OUR-PREFIXES
set policy-options policy-statement TRANSIT-OUT term OUR-ROUTES then accept
set policy-options policy-statement TRANSIT-OUT term DENY-ALL then reject

# Announce our prefix
set routing-options static route 192.0.2.0/24 discard
set protocols bgp group TRANSIT export TRANSIT-OUT
```

### OSPF
```junos
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 metric 10
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 bfd-liveness-detection minimum-interval 300
set protocols ospf area 0.0.0.0 interface lo0.0 passive
set protocols ospf reference-bandwidth 100g
set protocols ospf traffic-engineering
```

### IS-IS
```junos
set protocols isis interface ge-0/0/0.0 point-to-point
set protocols isis interface ge-0/0/0.0 level 2 metric 10
set protocols isis interface lo0.0 passive
set protocols isis level 1 disable
set protocols isis level 2 wide-metrics-only
set interfaces lo0 unit 0 family iso address 49.0001.0100.0000.0001.00
```

### Static Routes
```junos
set routing-options static route 0.0.0.0/0 next-hop 203.0.113.1
set routing-options static route 0.0.0.0/0 qualified-next-hop 203.0.113.5 preference 10
set routing-options static route 10.10.0.0/16 next-hop 10.0.0.1
set routing-options static route 192.0.2.0/24 discard  # null route for BGP announcement
```

## Switching & VLANs (EX / QFX)

```junos
# VLANs
set vlans DATA vlan-id 100
set vlans VOICE vlan-id 200

# Access port
set interfaces ge-0/0/1 unit 0 family ethernet-switching interface-mode access
set interfaces ge-0/0/1 unit 0 family ethernet-switching vlan members DATA

# Trunk port
set interfaces ge-0/0/24 unit 0 family ethernet-switching interface-mode trunk
set interfaces ge-0/0/24 unit 0 family ethernet-switching vlan members [DATA VOICE]

# IRB (L3 interface for VLAN)
set vlans DATA l3-interface irb.100
set interfaces irb unit 100 family inet address 10.100.0.1/24
```

### MX VLAN Handling (Router Style)
```junos
# On MX, VLANs are configured on sub-interfaces
set interfaces ge-0/0/0 vlan-tagging
set interfaces ge-0/0/0 unit 100 vlan-id 100
set interfaces ge-0/0/0 unit 100 family inet address 10.100.0.1/24
set interfaces ge-0/0/0 unit 200 vlan-id 200
set interfaces ge-0/0/0 unit 200 family inet address 10.200.0.1/24
```

## MPLS / L2VPN / L3VPN

### MPLS Core
```junos
# LDP
set protocols ldp interface ge-0/0/0.0
set protocols ldp interface ge-0/0/1.0
set protocols ldp interface lo0.0

# or RSVP-TE
set protocols rsvp interface ge-0/0/0.0
set protocols mpls interface ge-0/0/0.0
set protocols mpls label-switched-path TO-PE2 to 10.0.0.2

# Enable MPLS on interfaces
set interfaces ge-0/0/0 unit 0 family mpls
```

### L3VPN
```junos
# VRF (routing-instance)
set routing-instances CUSTOMER-A instance-type vrf
set routing-instances CUSTOMER-A interface ge-0/0/1.0
set routing-instances CUSTOMER-A route-distinguisher 64512:100
set routing-instances CUSTOMER-A vrf-target target:64512:100
set routing-instances CUSTOMER-A vrf-table-label

# Customer-facing interface
set interfaces ge-0/0/1 unit 0 family inet address 10.1.0.1/30

# VPNv4 in BGP
set protocols bgp group IBGP family inet-vpn unicast
```

### L2VPN / VPLS
```junos
# VPLS instance
set routing-instances VPLS-100 instance-type vpls
set routing-instances VPLS-100 interface ge-0/0/2.0
set routing-instances VPLS-100 route-distinguisher 64512:200
set routing-instances VPLS-100 vrf-target target:64512:200
set routing-instances VPLS-100 protocols vpls site-range 10
set routing-instances VPLS-100 protocols vpls site CE1 site-identifier 1

# L2circuit (pseudowire)
set protocols l2circuit neighbor 10.0.0.2 interface ge-0/0/3.0 virtual-circuit-id 300
```

## PPPoE (MX as BNG)

```junos
# Dynamic PPPoE subscriber management
set interfaces ge-0/0/0 flexible-vlan-tagging
set interfaces ge-0/0/0 encapsulation flexible-ethernet-services
set interfaces ge-0/0/0 unit 100 vlan-id 100
set interfaces ge-0/0/0 unit 100 encapsulation ppp-over-ether
set interfaces ge-0/0/0 unit 100 family pppoe dynamic-profile PPP-SUBSCRIBER

set dynamic-profiles PPP-SUBSCRIBER interfaces pp0 unit "$junos-interface-unit"
set dynamic-profiles PPP-SUBSCRIBER interfaces pp0 unit "$junos-interface-unit" ppp-options chap
set dynamic-profiles PPP-SUBSCRIBER interfaces pp0 unit "$junos-interface-unit" ppp-options pap
set dynamic-profiles PPP-SUBSCRIBER interfaces pp0 unit "$junos-interface-unit" pppoe-options underlying-interface "$junos-underlying-interface"
set dynamic-profiles PPP-SUBSCRIBER interfaces pp0 unit "$junos-interface-unit" family inet unnumbered-address lo0.0

set access profile RADIUS-PPP authentication-order radius
set access radius-server 10.0.0.10 secret "<RADIUS_SECRET>"
set access radius-server 10.0.0.10 source-address 10.0.0.1
```

## Firewall Filters

```junos
# Protect the RE (control plane)
set firewall family inet filter PROTECT-RE term ALLOW-BGP from protocol tcp
set firewall family inet filter PROTECT-RE term ALLOW-BGP from port bgp
set firewall family inet filter PROTECT-RE term ALLOW-BGP then accept

set firewall family inet filter PROTECT-RE term ALLOW-OSPF from protocol ospf
set firewall family inet filter PROTECT-RE term ALLOW-OSPF then accept

set firewall family inet filter PROTECT-RE term ALLOW-LDP from protocol tcp
set firewall family inet filter PROTECT-RE term ALLOW-LDP from port 646
set firewall family inet filter PROTECT-RE term ALLOW-LDP then accept
set firewall family inet filter PROTECT-RE term ALLOW-LDP-UDP from protocol udp
set firewall family inet filter PROTECT-RE term ALLOW-LDP-UDP from port 646
set firewall family inet filter PROTECT-RE term ALLOW-LDP-UDP then accept

set firewall family inet filter PROTECT-RE term ALLOW-ICMP from protocol icmp
set firewall family inet filter PROTECT-RE term ALLOW-ICMP then policer ICMP-POLICER
set firewall family inet filter PROTECT-RE term ALLOW-ICMP then accept

set firewall family inet filter PROTECT-RE term ALLOW-SSH from source-prefix-list MGMT-NETS
set firewall family inet filter PROTECT-RE term ALLOW-SSH from protocol tcp
set firewall family inet filter PROTECT-RE term ALLOW-SSH from destination-port ssh
set firewall family inet filter PROTECT-RE term ALLOW-SSH then accept

set firewall family inet filter PROTECT-RE term ALLOW-SNMP from source-prefix-list MGMT-NETS
set firewall family inet filter PROTECT-RE term ALLOW-SNMP from protocol udp
set firewall family inet filter PROTECT-RE term ALLOW-SNMP from destination-port snmp
set firewall family inet filter PROTECT-RE term ALLOW-SNMP then accept

set firewall family inet filter PROTECT-RE term DENY-ALL then log
set firewall family inet filter PROTECT-RE term DENY-ALL then discard

# Apply to loopback
set interfaces lo0 unit 0 family inet filter input PROTECT-RE

# Prefix list for management
set policy-options prefix-list MGMT-NETS 10.0.10.0/24
```

## VRRP

```junos
set interfaces irb unit 100 family inet address 10.100.0.2/24 vrrp-group 10 virtual-address 10.100.0.1
set interfaces irb unit 100 family inet address 10.100.0.2/24 vrrp-group 10 priority 150
set interfaces irb unit 100 family inet address 10.100.0.2/24 vrrp-group 10 preempt
set interfaces irb unit 100 family inet address 10.100.0.2/24 vrrp-group 10 accept-data
```

## SNMP

```junos
set snmp location "Brisbane POP"
set snmp contact "noc@athena.net.au"
set snmp community <COMMUNITY> authorization read-only
set snmp community <COMMUNITY> clients 10.0.10.0/24
set snmp trap-group TRAPS categories bgp
set snmp trap-group TRAPS categories routing
set snmp trap-group TRAPS targets 10.0.10.5
```

## Troubleshooting Commands

```junos
# BGP
show bgp summary
show bgp neighbor 203.0.113.1
show route advertising-protocol bgp 203.0.113.1
show route receive-protocol bgp 203.0.113.1
show route table inet.0 protocol bgp

# OSPF
show ospf neighbor
show ospf interface
show ospf database

# MPLS
show mpls interface
show ldp neighbor
show ldp database
show route table mpls.0
show l2vpn connections
show vpls connections

# General
show route
show route 192.0.2.0/24 detail
show interfaces terse
show interfaces ge-0/0/0 extensive
show chassis alarms
show system alarms
show log messages | match BGP

# Config
show configuration | display set | match bgp
show configuration | compare rollback 1

# Commit history
show system commit
```

## Common Gotchas

- **Don't forget `commit`**: Changes don't apply until committed
- **`commit confirmed`**: Always use for remote changes — auto-reverts if you lose connectivity
- **Interface naming**: `ge-` (gigabit), `xe-` (10G), `et-` (40/100G), format is `type-fpc/pic/port`
- **Unit numbers**: Junos always requires `unit X` — even for untagged interfaces (unit 0)
- **Policy default action**: If no term matches in a policy, default is reject. Always end with an explicit accept or reject term
- **Family statements**: Must explicitly enable protocol families on interfaces (`family inet`, `family mpls`, `family iso`)
- **Dual RE**: Use `commit synchronize` on dual-RE systems (MX with redundant routing engines)
- **Rescue config**: Set with `request system configuration rescue save` — provides a known-good fallback
