# Cisco Reference (IOS / IOS-XE / NX-OS / IOS-XR)

## Platform Overview

Cisco runs multiple operating systems across platforms:

| OS | Platforms | Config Model | Notes |
|----|-----------|-------------|-------|
| IOS | Legacy routers/switches (ISR G1/G2, Catalyst 2960/3560) | Immediate (write mem) | End of life for most |
| IOS-XE | ISR 1000/4000, ASR 1000, Catalyst 9000, CSR1000v | Immediate (write mem) | Linux-based, current mainstream |
| NX-OS | Nexus 3000/5000/7000/9000 | Immediate (copy run start) | Data centre focused |
| IOS-XR | ASR 9000, NCS 540/560/5500, 8000 series | Commit-based | Carrier-grade, structured config |

**Key difference**: IOS-XR uses a commit model like Junos. IOS/IOS-XE/NX-OS apply changes immediately.

## CLI Syntax Patterns

### IOS / IOS-XE
```ios
! Enter config mode
configure terminal

! Interface config
interface GigabitEthernet0/0/0
 ip address 10.0.0.1 255.255.255.252
 no shutdown

! Save config
end
write memory
! or: copy running-config startup-config
```

### NX-OS
```ios
! Similar to IOS but with differences
configure terminal
feature bgp
feature ospf
feature interface-vlan
feature lacp
feature vpc

interface Ethernet1/1
  no switchport
  ip address 10.0.0.1/30
  no shutdown
```

### IOS-XR
```ios
! Commit-based config
configure
interface GigabitEthernet0/0/0/0
 ipv4 address 10.0.0.1 255.255.255.252
 no shutdown
!
commit
! or: commit confirmed 300 (auto-rollback in 300 seconds if not confirmed)
end
```

## Routing

### BGP — IOS/IOS-XE
```ios
router bgp 64512
 bgp router-id 10.0.0.1
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 !
 ! eBGP transit peer
 neighbor 203.0.113.1 remote-as 64500
 neighbor 203.0.113.1 description Transit-Provider-A
 neighbor 203.0.113.1 password <BGP_PASSWORD>
 neighbor 203.0.113.1 timers 10 30
 !
 ! iBGP route reflector client
 neighbor 10.0.0.2 remote-as 64512
 neighbor 10.0.0.2 update-source Loopback0
 neighbor 10.0.0.2 description RR-Client-PE2
 !
 address-family ipv4 unicast
  neighbor 203.0.113.1 activate
  neighbor 203.0.113.1 route-map TRANSIT-IN in
  neighbor 203.0.113.1 route-map TRANSIT-OUT out
  neighbor 203.0.113.1 prefix-list BOGONS in
  !
  neighbor 10.0.0.2 activate
  neighbor 10.0.0.2 route-reflector-client
  neighbor 10.0.0.2 next-hop-self
  !
  network 192.0.2.0 mask 255.255.255.0
 exit-address-family
!
! Prefix lists
ip prefix-list BOGONS seq 5 deny 0.0.0.0/0
ip prefix-list BOGONS seq 10 deny 10.0.0.0/8 le 32
ip prefix-list BOGONS seq 15 deny 172.16.0.0/12 le 32
ip prefix-list BOGONS seq 20 deny 192.168.0.0/16 le 32
ip prefix-list BOGONS seq 25 deny 100.64.0.0/10 le 32
ip prefix-list BOGONS seq 100 permit 0.0.0.0/0 le 24
!
! Route maps
route-map TRANSIT-IN permit 10
 set local-preference 200
!
route-map TRANSIT-OUT permit 10
 match ip address prefix-list OUR-PREFIXES
```

### BGP — NX-OS
```ios
feature bgp

router bgp 64512
  router-id 10.0.0.1
  log-neighbor-changes
  address-family ipv4 unicast
    network 192.0.2.0/24
  neighbor 203.0.113.1
    remote-as 64500
    description Transit-Provider
    timers 10 30
    address-family ipv4 unicast
      route-map TRANSIT-IN in
      route-map TRANSIT-OUT out
      prefix-list BOGONS in
```

### BGP — IOS-XR
```ios
router bgp 64512
 bgp router-id 10.0.0.1
 bgp log neighbor changes detail
 address-family ipv4 unicast
  network 192.0.2.0/24
 !
 neighbor 203.0.113.1
  remote-as 64500
  description Transit-Provider
  timers 10 30
  address-family ipv4 unicast
   route-policy TRANSIT-IN in
   route-policy TRANSIT-OUT out
  !
 !
 neighbor 10.0.0.2
  remote-as 64512
  update-source Loopback0
  address-family ipv4 unicast
   route-reflector-client
   next-hop-self
  !
 !
!
! IOS-XR uses route-policy (not route-maps)
route-policy TRANSIT-IN
  set local-preference 200
  done
end-policy
!
route-policy TRANSIT-OUT
  if destination in OUR-PREFIXES then
    pass
  else
    drop
  endif
end-policy
!
prefix-set OUR-PREFIXES
  192.0.2.0/24
end-set
```

### OSPF
```ios
! IOS/IOS-XE
router ospf 1
 router-id 10.0.0.1
 auto-cost reference-bandwidth 100000
 passive-interface default
 no passive-interface GigabitEthernet0/0/0
 no passive-interface GigabitEthernet0/0/1
 network 10.0.0.0 0.0.0.3 area 0
 network 10.0.1.0 0.0.0.255 area 0

! Per-interface (preferred)
interface GigabitEthernet0/0/0
 ip ospf 1 area 0
 ip ospf network point-to-point
 ip ospf cost 10
 ip ospf bfd
```

### Static Routes
```ios
! IOS/IOS-XE
ip route 0.0.0.0 0.0.0.0 203.0.113.1 name Default-Transit
ip route 10.10.0.0 255.255.0.0 10.0.0.1 200 name Backup-Route

! NX-OS
ip route 0.0.0.0/0 203.0.113.1 name Default-Transit

! IOS-XR
router static
 address-family ipv4 unicast
  0.0.0.0/0 203.0.113.1 description Default-Transit
```

## Switching & VLANs

### IOS/IOS-XE (Catalyst)
```ios
! Create VLANs
vlan 100
 name Data
vlan 200
 name Voice

! Access port
interface GigabitEthernet1/0/1
 switchport mode access
 switchport access vlan 100
 spanning-tree portfast

! Trunk port
interface GigabitEthernet1/0/24
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 100,200
 switchport trunk native vlan 999

! SVI
interface Vlan100
 ip address 10.100.0.1 255.255.255.0
 no shutdown
```

### NX-OS
```ios
feature interface-vlan

vlan 100
  name Data

interface Ethernet1/1
  switchport
  switchport mode trunk
  switchport trunk allowed vlan 100,200

interface Vlan100
  no shutdown
  ip address 10.100.0.1/24
```

## MPLS / L3VPN

### IOS/IOS-XE
```ios
! Enable MPLS
mpls ip
mpls ldp router-id Loopback0

interface GigabitEthernet0/0/0
 mpls ip

! L3VPN (VRF)
ip vrf CUSTOMER-A
 rd 64512:100
 route-target export 64512:100
 route-target import 64512:100

interface GigabitEthernet0/0/1
 ip vrf forwarding CUSTOMER-A
 ip address 10.1.0.1 255.255.255.252

router bgp 64512
 address-family vpnv4
  neighbor 10.0.0.2 activate
  neighbor 10.0.0.2 send-community extended
 !
 address-family ipv4 vrf CUSTOMER-A
  redistribute connected
  redistribute static
```

### IOS-XR
```ios
mpls ldp
 router-id 10.0.0.1
 interface GigabitEthernet0/0/0/0
 !
!
vrf CUSTOMER-A
 address-family ipv4 unicast
  import route-target 64512:100
  export route-target 64512:100
 !
!
interface GigabitEthernet0/0/0/1
 vrf CUSTOMER-A
 ipv4 address 10.1.0.1 255.255.255.252
!
router bgp 64512
 vrf CUSTOMER-A
  rd 64512:100
  address-family ipv4 unicast
   redistribute connected
  !
 !
```

## HSRP / VRRP

```ios
! HSRP (Cisco proprietary, IOS/IOS-XE)
interface Vlan100
 ip address 10.100.0.2 255.255.255.0
 standby version 2
 standby 10 ip 10.100.0.1
 standby 10 priority 150
 standby 10 preempt
 standby 10 timers 1 3

! VRRP (standards-based)
interface Vlan100
 ip address 10.100.0.2 255.255.255.0
 vrrp 10 ip 10.100.0.1
 vrrp 10 priority 150
 vrrp 10 preempt
```

## QoS

```ios
! Class maps
class-map match-any VOICE
 match dscp ef
class-map match-any VIDEO
 match dscp af41

! Policy map
policy-map WAN-QOS
 class VOICE
  priority percent 20
 class VIDEO
  bandwidth percent 30
 class class-default
  fair-queue

! Apply to interface
interface GigabitEthernet0/0/0
 service-policy output WAN-QOS
```

## ACLs / Security

```ios
! Extended ACL
ip access-list extended PROTECT-MGMT
 permit tcp 10.0.0.0 0.0.255.255 any eq 22
 permit tcp 10.0.0.0 0.0.255.255 any eq 23
 permit udp 10.0.0.0 0.0.255.255 any eq 161
 deny   ip any any log

! Control Plane Policing
ip access-list extended CoPP-BGP
 permit tcp any any eq 179
 permit tcp any eq 179 any

class-map match-all CoPP-BGP
 match access-group name CoPP-BGP

policy-map CoPP-POLICY
 class CoPP-BGP
  police 1000000 conform-action transmit exceed-action drop
 class class-default
  police 500000 conform-action transmit exceed-action drop

control-plane
 service-policy input CoPP-POLICY
```

## SNMP / Monitoring

```ios
snmp-server community <COMMUNITY> RO SNMP-ACL
snmp-server location "Brisbane POP"
snmp-server contact "noc@athena.net.au"
snmp-server host 10.0.10.5 version 2c <COMMUNITY>
snmp-server enable traps bgp
snmp-server enable traps ospf

ip access-list standard SNMP-ACL
 permit 10.0.10.0 0.0.0.255
```

## Troubleshooting Commands

```ios
! BGP
show ip bgp summary
show ip bgp neighbors 203.0.113.1
show ip bgp neighbors 203.0.113.1 advertised-routes
show ip bgp neighbors 203.0.113.1 received-routes
show ip bgp 192.0.2.0/24

! OSPF
show ip ospf neighbor
show ip ospf interface brief
show ip ospf database

! MPLS
show mpls forwarding-table
show mpls ldp neighbor
show mpls ldp bindings

! General
show ip route
show ip interface brief
show interface GigabitEthernet0/0/0
show running-config | section router bgp
show logging | include BGP

! NX-OS specific
show ip bgp summary vrf all
show vpc
show port-channel summary
show interface status

! IOS-XR specific
show bgp summary
show route
show mpls forwarding
show commit changes diff
```

## Common Gotchas

- **`no bgp default ipv4-unicast`**: Best practice on IOS/IOS-XE — forces explicit activation per address family
- **VTP**: Can wipe your VLAN database if VTP mode is server and a higher revision number comes in. Set `vtp mode transparent` on carrier gear
- **IOS vs IOS-XE**: Mostly compatible syntax, but some features differ. Always check platform support
- **IOS-XR commit model**: Don't forget `commit` — config won't apply without it. Use `commit confirmed` for safety
- **NX-OS features**: Many features must be explicitly enabled with `feature <name>` before use
- **Subnet masks**: IOS uses dotted-decimal (255.255.255.0), IOS-XR and NX-OS use prefix length (/24)
- **Wildcard masks**: ACLs and OSPF network statements use wildcard (0.0.0.255), not subnet masks
