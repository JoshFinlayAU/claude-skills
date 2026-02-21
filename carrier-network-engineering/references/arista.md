# Arista EOS Reference

## Platform Overview

Arista EOS (Extensible Operating System) runs on Arista switches (7000, 7050, 7060, 7260, 7280, 7300, 7500, 7800 series). Built on Linux with a multi-process state-sharing architecture (Sysdb). CLI is very similar to Cisco IOS, making it easy to transition from Cisco.

**Key strengths**: Linux-based (full bash access), multi-process architecture (individual process restart without reboot), MLAG for multi-chassis LAG, VXLAN/EVPN data centre fabric, eAPI (JSON-RPC management), streaming telemetry, CloudVision integration.

## CLI Syntax Patterns

```eos
! Very similar to Cisco IOS
configure terminal

! Interface config
interface Ethernet1
   no switchport
   ip address 10.0.0.1/30
   no shutdown

! Save config
write memory
! or: copy running-config startup-config

! Access Linux shell
bash

! Config sessions (like Junos candidate config)
configure session my-changes
! make changes...
commit
! or: abort
```

### Config Sessions (Recommended for Production)
```eos
! Create a session (changes don't apply until commit)
configure session CHANGE-123
   interface Ethernet1
      description "Uplink to PE2"
   exit

! Review diff
show session-config diffs

! Commit
commit

! Or abort
abort

! Commit with timer (auto-rollback)
commit timer 00:05:00
! Confirm before timer expires
configure session CHANGE-123
commit
```

## Routing

### BGP
```eos
router bgp 64512
   router-id 10.0.0.1
   no bgp default ipv4-unicast
   maximum-paths 4
   !
   ! eBGP transit
   neighbor TRANSIT peer group
   neighbor TRANSIT remote-as 64500
   neighbor TRANSIT description "Transit Provider"
   neighbor TRANSIT send-community standard extended
   neighbor TRANSIT maximum-routes 500000 warning-only
   neighbor 203.0.113.1 peer group TRANSIT
   neighbor 203.0.113.1 password <BGP_PASSWORD>
   !
   ! iBGP
   neighbor IBGP peer group
   neighbor IBGP remote-as 64512
   neighbor IBGP update-source Loopback0
   neighbor IBGP send-community standard extended
   neighbor 10.0.0.2 peer group IBGP
   neighbor 10.0.0.2 description "PE2"
   !
   address-family ipv4
      neighbor TRANSIT activate
      neighbor TRANSIT route-map TRANSIT-IN in
      neighbor TRANSIT route-map TRANSIT-OUT out
      neighbor TRANSIT prefix-list BOGONS in
      neighbor IBGP activate
      neighbor IBGP route-reflector-client
      neighbor IBGP next-hop-self
      network 192.0.2.0/24
   !
   address-family evpn
      neighbor IBGP activate
!
! Prefix lists
ip prefix-list BOGONS seq 10 deny 0.0.0.0/0
ip prefix-list BOGONS seq 20 deny 10.0.0.0/8 le 32
ip prefix-list BOGONS seq 30 deny 172.16.0.0/12 le 32
ip prefix-list BOGONS seq 40 deny 192.168.0.0/16 le 32
ip prefix-list BOGONS seq 50 deny 100.64.0.0/10 le 32
ip prefix-list BOGONS seq 100 permit 0.0.0.0/0 le 24
!
ip prefix-list OUR-PREFIXES seq 10 permit 192.0.2.0/24
!
! Route maps
route-map TRANSIT-IN permit 10
   set local-preference 200
!
route-map TRANSIT-OUT permit 10
   match ip address prefix-list OUR-PREFIXES
```

### OSPF
```eos
router ospf 1
   router-id 10.0.0.1
   auto-cost reference-bandwidth 100000
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   network 10.0.0.0/30 area 0.0.0.0
   max-lsa 12000

interface Ethernet1
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
   ip ospf cost 10
   ip ospf bfd
```

### Static Routes
```eos
ip route 0.0.0.0/0 203.0.113.1 name Default-Transit
ip route 10.10.0.0/16 10.0.0.1 200 name Backup-Route
ip route 192.0.2.0/24 Null0 name BGP-Anchor
```

## Switching & VLANs

```eos
! Create VLANs
vlan 100
   name Data
vlan 200
   name Voice

! Access port
interface Ethernet10
   switchport mode access
   switchport access vlan 100
   spanning-tree portfast

! Trunk port
interface Ethernet48
   switchport mode trunk
   switchport trunk allowed vlan 100,200
   switchport trunk native vlan 999

! SVI
interface Vlan100
   ip address 10.100.0.1/24
   no shutdown
```

## MLAG (Multi-Chassis LAG)

```eos
! MLAG peer link (both switches)
vlan 4094
   name MLAG-PEER
   trunk group MLAG-PEER

interface Ethernet49
   channel-group 999 mode active
interface Ethernet50
   channel-group 999 mode active

interface Port-Channel999
   switchport mode trunk
   switchport trunk group MLAG-PEER

! MLAG config (Switch 1)
mlag configuration
   domain-id MLAG-DOMAIN
   local-interface Vlan4094
   peer-address 10.255.0.2
   peer-link Port-Channel999
   reload-delay mlag 300
   reload-delay non-mlag 330

interface Vlan4094
   ip address 10.255.0.1/30
   no autostate

! MLAG member (both switches, same config)
interface Ethernet1
   channel-group 1 mode active
   mlag 1

interface Port-Channel1
   switchport mode trunk
   switchport trunk allowed vlan 100,200
   mlag 1
```

## VXLAN / EVPN

```eos
! VXLAN tunnel interface
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 100 vni 10100
   vxlan vlan 200 vni 10200
   vxlan vrf TENANT-A vni 50001

! EVPN
router bgp 64512
   address-family evpn
      neighbor SPINE activate
   !
   vlan 100
      rd auto
      route-target both 100:10100
      redistribute learned
   !
   vrf TENANT-A
      rd 64512:50001
      route-target import evpn 50001:50001
      route-target export evpn 50001:50001
      redistribute connected

! VRF definition
vrf instance TENANT-A
ip routing vrf TENANT-A

interface Vlan100
   vrf TENANT-A
   ip address virtual 10.100.0.1/24
```

## VRRP

```eos
interface Vlan100
   ip address 10.100.0.2/24
   vrrp 10 ipv4 10.100.0.1
   vrrp 10 priority 150
   vrrp 10 preempt
   vrrp 10 timers advertise 1
```

## QoS

```eos
! Class maps
class-map type qos match-any VOICE
   match dscp ef
class-map type qos match-any VIDEO
   match dscp af41

! Policy map
policy-map type qos WAN-QOS
   class VOICE
      set cos 5
      police rate 20 percent
   class VIDEO
      set cos 4
      bandwidth percent 30
   class class-default

! Apply
interface Ethernet1
   service-policy type qos input WAN-QOS
```

## ACLs / Security

```eos
! Extended ACL
ip access-list PROTECT-MGMT
   10 permit tcp 10.0.0.0/16 any eq ssh
   20 permit udp 10.0.0.0/16 any eq snmp
   30 permit tcp any any eq bgp
   40 permit ospf any any
   1000 deny ip any any log

! Apply to management
management ssh
   ip access-group PROTECT-MGMT in

! Control plane ACL
system control-plane
   ip access-group PROTECT-MGMT in
```

## SNMP

```eos
snmp-server community <COMMUNITY> ro
snmp-server host 10.0.10.5 version 2c <COMMUNITY>
snmp-server location "Brisbane POP"
snmp-server contact "noc@athena.net.au"
snmp-server enable traps bgp
```

## Troubleshooting Commands

```eos
! BGP
show ip bgp summary
show ip bgp neighbors 203.0.113.1
show ip bgp neighbors 203.0.113.1 advertised-routes
show ip bgp neighbors 203.0.113.1 received-routes

! OSPF
show ip ospf neighbor
show ip ospf interface
show ip ospf database

! MLAG
show mlag
show mlag detail
show mlag interfaces

! VXLAN / EVPN
show vxlan vtep
show vxlan address-table
show bgp evpn summary
show bgp evpn route-type mac-ip

! General
show ip route
show interfaces status
show interfaces Ethernet1
show running-config section router bgp
show logging last 50

! Linux shell access
bash sudo ip route show
bash sudo tcpdump -i et1 -n
```

## Common Gotchas

- **Very Cisco-like**: Intentionally similar to IOS. Most IOS knowledge transfers directly
- **Config sessions**: Use them for production changes — provides diff preview and atomic commits
- **No `switchport trunk encapsulation`**: Arista only supports 802.1Q, no ISL. Don't include the encapsulation command
- **MLAG peer link**: Must be its own VLAN with trunk group — don't allow normal traffic on the peer link VLAN
- **eAPI**: REST API is always available — useful for automation (`management api http-commands`)
- **Multi-process**: Individual daemons can be restarted without full reload (`agent <daemon> restart`)
- **Bash access**: Full Linux shell available. Useful for tcpdump, ip route, and scripting directly on the switch
- **VXLAN MTU**: Ensure fabric MTU supports VXLAN overhead (50 bytes). Set interface MTU to at least 9214
