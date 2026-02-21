# MikroTik RouterOS Reference

## Platform Overview

MikroTik RouterOS runs on MikroTik hardware (CCR, CRS, hAP, etc.) and x86/CHR. The CLI uses a hierarchical path-based syntax. Configuration is applied immediately (no commit model). RouterOS 7 is current; RouterOS 6 is legacy but still widely deployed.

**Key difference from other vendors**: No commit/rollback. Changes are live immediately. Use `/system backup save` and `/export` before major changes.

## CLI Syntax Patterns

### Hierarchy Navigation
```routeros
# Navigate to a section
/ip address
# Or use full path inline
/ip address add address=10.0.0.1/30 interface=ether1
```

### Common Operations
```routeros
# Add
/ip address add address=10.0.0.1/30 interface=ether1 comment="WAN uplink"
# Set (modify existing)
/ip address set [find where interface=ether1] address=10.0.0.2/30
# Remove
/ip address remove [find where interface=ether1]
# Print / show
/ip address print
/ip address print detail
/ip address print where interface=ether1
# Export (show config)
/export
/ip address export
```

### Variable Patterns
```routeros
# Find by property
[find where name=ether1]
[find where comment~"WAN"]
# Global variables
:global myVar "value"
# Scripts
/system script add name=myScript source={
  :local iface "ether1"
  /ip address print where interface=$iface
}
```

## Routing

### BGP (RouterOS 7)
RouterOS 7 restructured BGP significantly from v6. Key concepts:

```routeros
# BGP connection (replaces v6 "peer" concept)
/routing/bgp/connection
add name=transit-provider \
    remote.address=203.0.113.1 \
    remote.as=64500 \
    local.role=customer \
    local.address=203.0.113.2 \
    address-families=ip \
    routing-table=main \
    as=64512 \
    output.filter-chain=bgp-out \
    input.filter-chain=bgp-in \
    multihop=no \
    hold-time=30s \
    keepalive-time=10s

# BGP template (apply common settings to multiple connections)
/routing/bgp/template
add name=ibgp-template \
    as=64512 \
    address-families=ip \
    routing-table=main \
    hold-time=30s

/routing/bgp/connection
add name=ibgp-rr \
    template=ibgp-template \
    remote.address=10.0.0.1 \
    local.address=10.0.0.2

# Route filters (v7 uses filter rules, not prefix-lists directly)
/routing/filter/rule
add chain=bgp-in rule="if (dst==0.0.0.0/0) { accept }"
add chain=bgp-in rule="reject"

add chain=bgp-out rule="if (dst in 192.168.0.0/16) { reject }"
add chain=bgp-out rule="if (dst in 10.0.0.0/8) { reject }"
add chain=bgp-out rule="if (dst in 172.16.0.0/12) { reject }"
add chain=bgp-out rule="accept"

# Viewing BGP state
/routing/bgp/session print
/routing/bgp/advertisements print
```

### BGP (RouterOS 6 — Legacy)
```routeros
/routing bgp instance set default as=64512 router-id=10.0.0.1
/routing bgp peer
add name=transit remote-address=203.0.113.1 remote-as=64500 \
    in-filter=bgp-in out-filter=bgp-out ttl=default

/routing filter
add chain=bgp-in action=accept prefix=0.0.0.0/0
add chain=bgp-in action=discard
```

### OSPF
```routeros
# RouterOS 7
/routing/ospf/instance
add name=default router-id=10.0.0.1

/routing/ospf/area
add name=backbone area-id=0.0.0.0 instance=default

/routing/ospf/interface-template
add area=backbone interfaces=ether1,ether2 type=ptp cost=10
add area=backbone interfaces=lo0 type=ptp passive

# RouterOS 6
/routing ospf instance set default router-id=10.0.0.1
/routing ospf interface add interface=ether1 cost=10 network-type=point-to-point
/routing ospf network add network=10.0.0.0/24 area=backbone
```

### Static Routes
```routeros
/ip route add dst-address=0.0.0.0/0 gateway=203.0.113.1 distance=1 comment="Default via transit"
/ip route add dst-address=10.10.0.0/16 gateway=10.0.0.1 distance=10
# Blackhole
/ip route add dst-address=192.168.0.0/16 type=blackhole
```

## Switching & VLANs

### Bridge VLAN Filtering (Recommended, v6.41+/v7)
```routeros
/interface bridge
add name=bridge1 vlan-filtering=no  comment="Enable filtering AFTER config"

/interface bridge port
add bridge=bridge1 interface=ether2 pvid=100
add bridge=bridge1 interface=ether3 pvid=200
add bridge=bridge1 interface=sfp-sfpplus1  # trunk

/interface bridge vlan
add bridge=bridge1 tagged=sfp-sfpplus1,bridge1 untagged=ether2 vlan-ids=100
add bridge=bridge1 tagged=sfp-sfpplus1,bridge1 untagged=ether3 vlan-ids=200

# Management VLAN IP
/interface vlan add interface=bridge1 vlan-id=100 name=vlan100-mgmt
/ip address add address=10.100.0.1/24 interface=vlan100-mgmt

# CRITICAL: Enable VLAN filtering LAST to avoid lockout
/interface bridge set bridge1 vlan-filtering=yes
```

**Warning**: Enabling `vlan-filtering=yes` before configuring VLANs will lock you out if your management traffic traverses the bridge. Always configure VLANs first, then enable filtering.

### VLAN Interfaces (Router-on-a-Stick)
```routeros
/interface vlan
add interface=ether1 vlan-id=100 name=vlan100
add interface=ether1 vlan-id=200 name=vlan200
/ip address add address=10.100.0.1/24 interface=vlan100
/ip address add address=10.200.0.1/24 interface=vlan200
```

## MPLS

```routeros
# Enable MPLS on interfaces
/mpls interface add interface=ether1
/mpls interface add interface=ether2

# LDP
/mpls ldp set enabled=yes lsr-id=10.0.0.1 transport-address=10.0.0.1
/mpls ldp interface add interface=ether1
/mpls ldp interface add interface=ether2

# VPLS tunnel
/interface vpls
add name=vpls-service100 vpls-id=100 \
    remote-peer=10.0.0.2 \
    cisco-style=no \
    cisco-style-id=0

# Verification
/mpls forwarding-table print
/mpls ldp neighbor print
/mpls ldp interface print
```

## PPPoE Server

```routeros
# PPPoE server for subscriber termination
/ip pool add name=pppoe-pool ranges=100.64.0.2-100.64.255.254

/ppp profile
add name=pppoe-profile \
    local-address=100.64.0.1 \
    remote-address=pppoe-pool \
    dns-server=8.8.8.8,8.8.4.4 \
    rate-limit=100M/100M \
    only-one=yes

/interface pppoe-server server
add service-name=ISP-Service \
    interface=ether2 \
    default-profile=pppoe-profile \
    authentication=pap,chap,mschap2 \
    one-session-per-host=yes \
    max-mtu=1480 \
    max-mru=1480

# RADIUS integration
/radius add address=10.0.0.10 secret=<RADIUS_SECRET> service=ppp
/ppp aaa set use-radius=yes accounting=yes interim-update=5m
```

## Firewall

```routeros
# Input chain (protect the router)
/ip firewall filter
add chain=input action=accept connection-state=established,related comment="Accept established"
add chain=input action=drop connection-state=invalid comment="Drop invalid"
add chain=input action=accept protocol=icmp comment="Accept ICMP"
add chain=input action=accept src-address=10.0.0.0/8 comment="Accept from management"
add chain=input action=accept protocol=tcp dst-port=179 comment="Accept BGP"
add chain=input action=accept protocol=udp dst-port=646 comment="Accept LDP"
add chain=input action=accept protocol=ospf comment="Accept OSPF"
add chain=input action=drop log=yes log-prefix="INPUT-DROP" comment="Drop all else"

# Forward chain
/ip firewall filter
add chain=forward action=accept connection-state=established,related
add chain=forward action=drop connection-state=invalid
add chain=forward action=accept in-interface=bridge-lan out-interface=ether1-wan comment="LAN to WAN"
add chain=forward action=drop log=yes log-prefix="FORWARD-DROP"

# NAT
/ip firewall nat
add chain=srcnat out-interface=ether1-wan action=masquerade
```

## VRRP

```routeros
/interface vrrp
add name=vrrp1 interface=vlan100 vrid=10 priority=150 \
    preemption-mode=yes interval=1s version=3

/ip address add address=10.100.0.2/24 interface=vlan100
/ip address add address=10.100.0.1/32 interface=vrrp1  # VIP
```

## Queues / QoS

```routeros
# Simple queue (per-subscriber)
/queue simple
add name=customer-001 target=100.64.0.50/32 \
    max-limit=50M/50M burst-limit=100M/100M \
    burst-threshold=40M/40M burst-time=10s/10s

# Queue tree (class-based)
/ip firewall mangle
add chain=forward protocol=udp dst-port=5060 action=mark-packet new-packet-mark=voip passthrough=no
add chain=forward action=mark-packet new-packet-mark=default passthrough=no

/queue type add name=voip-pfifo kind=pfifo pfifo-limit=64

/queue tree
add name=wan-out parent=ether1-wan max-limit=1G
add name=voip parent=wan-out packet-mark=voip priority=1 queue=voip-pfifo max-limit=10M
add name=default-traffic parent=wan-out packet-mark=default priority=8 max-limit=990M
```

## Monitoring / SNMP

```routeros
/snmp
set enabled=yes contact="noc@athena.net.au" location="Brisbane POP"
/snmp community
set public addresses=10.0.0.0/8 read-access=yes write-access=no
add name=monitoring addresses=10.0.10.0/24 read-access=yes security=none
```

## Useful Troubleshooting Commands

```routeros
# Interface status
/interface print detail
/interface monitor-traffic ether1 once

# Routing
/ip route print where active
/routing bgp session print
/routing bgp advertisements print peer=<peer-name>
/routing ospf neighbor print

# ARP / MAC
/ip arp print
/interface bridge host print

# Logs
/log print where topics~"bgp"
/log print where topics~"ospf"

# Packet sniffer
/tool sniffer quick interface=ether1 filter-address=203.0.113.1

# Torch (live traffic view)
/tool torch interface=ether1 src-address=0.0.0.0/0 dst-address=0.0.0.0/0

# Ping / traceroute
/ping 203.0.113.1 count=5 size=1500 do-not-fragment
/tool traceroute 203.0.113.1
```

## Common Gotchas

- **VLAN filtering lockout**: Always configure VLANs before setting `vlan-filtering=yes`
- **RouterOS 6 vs 7 syntax**: BGP, OSPF, and MPLS configs changed significantly in v7
- **No commit model**: Changes are immediate. Use Safe Mode (`Ctrl+X` in terminal) for a pseudo-rollback — if your session drops, changes revert
- **Bridge vs routing**: Interfaces in a bridge are switched, not routed. Remove from bridge before adding IP addresses directly
- **Firewall order**: Rules are processed top-to-bottom, first match wins
- **CHR licensing**: CHR has bandwidth limits per license tier (free = 1Mbps)
- **Default config**: New devices ship with a default config that may include DHCP client, firewall, NAT. Use `/system reset-configuration no-defaults=yes` for a clean start on carrier gear
