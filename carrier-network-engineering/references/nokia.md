# Nokia SR OS Reference

## Platform Overview

Nokia SR OS (Service Router Operating System) runs on Nokia 7750 SR, 7450 ESS, 7210 SAS, and 7250 IXR platforms. It is a carrier-grade OS with a strong service-oriented architecture. SR OS uses a hierarchical CLI with both classic CLI and MD-CLI (model-driven CLI) modes.

**Key strengths**: Service-oriented architecture (SAP/SDP model), robust MPLS/VPN implementation, carrier-grade reliability, comprehensive OAM, model-driven management.

## CLI Modes

### Classic CLI vs MD-CLI
SR OS supports two CLI modes. MD-CLI is the modern approach (YANG-modelled), Classic CLI is the traditional interface. Most production deployments still use Classic CLI.

```sros
# Classic CLI prompt
A:router# configure
A:router>config#

# MD-CLI prompt
[/]
A:router# configure private
(ex)[/]
A:router#
```

This reference uses **Classic CLI** as it remains the most common in production.

### Common Operations
```sros
# Enter config mode
configure

# Navigate
configure router bgp
configure service vpls 100

# Info (show config in context)
info
info detail

# Admin commands
admin save           # save config
admin rollback save  # save rollback checkpoint
admin rollback revert latest-rb
admin rollback compare latest-rb to active-cfg

# Show commands (from operational mode)
show router bgp summary
show router interface
show service id 100 base
```

## Routing

### BGP
```sros
configure
    router
        autonomous-system 64512
        router-id 10.0.0.1
        bgp
            group "TRANSIT"
                type external
                peer-as 64500
                hold-time 30
                import "TRANSIT-IN"
                export "TRANSIT-OUT"
                neighbor 203.0.113.1
                    authentication-key "<BGP_PASSWORD>"
                    description "Transit Provider A"
                exit
            exit
            group "IBGP"
                type internal
                local-address 10.0.0.1
                cluster 10.0.0.1
                family vpn-ipv4 vpn-ipv6
                neighbor 10.0.0.2
                    description "PE2"
                exit
            exit
        exit
    exit
exit

# Policy
configure
    router
        policy-options
            begin
            prefix-list "BOGONS"
                prefix 10.0.0.0/8 longer
                prefix 172.16.0.0/12 longer
                prefix 192.168.0.0/16 longer
                prefix 100.64.0.0/10 longer
            exit
            prefix-list "OUR-PREFIXES"
                prefix 192.0.2.0/24 exact
            exit
            policy-statement "TRANSIT-IN"
                entry 10
                    from
                        prefix-list "BOGONS"
                    exit
                    action reject
                    exit
                exit
                default-action accept
                    local-preference 200
                exit
            exit
            policy-statement "TRANSIT-OUT"
                entry 10
                    from
                        prefix-list "OUR-PREFIXES"
                    exit
                    action accept
                    exit
                exit
                default-action reject
                exit
            exit
            commit
        exit
    exit
exit
```

### OSPF
```sros
configure
    router
        ospf
            area 0.0.0.0
                interface "system"
                    passive
                exit
                interface "to-PE2"
                    interface-type point-to-point
                    metric 10
                    bfd-enable
                exit
            exit
            reference-bandwidth 100000000
            traffic-engineering
        exit
    exit
exit
```

### IS-IS
```sros
configure
    router
        isis
            level-capability level-2
            area-id 49.0001
            interface "system"
                passive
            exit
            interface "to-PE2"
                interface-type point-to-point
                level 2
                    metric 10
                exit
                bfd-enable
            exit
        exit
    exit
exit
```

### Static Routes
```sros
configure
    router
        static-route 0.0.0.0/0 next-hop 203.0.113.1
            description "Default via transit"
        exit
        static-route 192.0.2.0/24 black-hole
            description "BGP announcement anchor"
        exit
    exit
exit
```

## Service Architecture (SAP/SDP)

Nokia uses a service-oriented model:
- **SAP** (Service Access Point): Customer-facing interface (ingress/egress to a service)
- **SDP** (Service Distribution Point): Tunnel between PEs (MPLS LSP, GRE, etc.)
- **Service**: Logical construct binding SAPs and SDPs together

```
Customer --- [SAP] --- Service --- [SDP/tunnel] --- Service --- [SAP] --- Customer
```

### SDP (Tunnel)
```sros
configure
    service
        sdp 100 mpls create
            far-end 10.0.0.2
            ldp
            keep-alive
                shutdown
            exit
            no shutdown
        exit
    exit
exit
```

## L3VPN (VPRN)

```sros
configure
    service
        vprn 100 customer 1 create
            description "Customer A L3VPN"
            route-distinguisher 64512:100
            vrf-target target:64512:100
            interface "to-CE" create
                address 10.1.0.1/30
                sap 1/1/1:100 create
                    description "Customer A CE link"
                exit
            exit
            bgp
                no shutdown
            exit
            no shutdown
        exit
    exit
exit
```

## L2VPN (VPLS / Epipe)

### VPLS
```sros
configure
    service
        vpls 200 customer 1 create
            description "Customer B L2 service"
            stp
                shutdown
            exit
            sap 1/1/2:200 create
                description "CE access"
            exit
            spoke-sdp 100:200 create
                description "To PE2"
            exit
            no shutdown
        exit
    exit
exit
```

### Epipe (Point-to-Point L2VPN)
```sros
configure
    service
        epipe 300 customer 1 create
            description "Point-to-point circuit"
            sap 1/1/3:300 create
            exit
            spoke-sdp 100:300 create
            exit
            no shutdown
        exit
    exit
exit
```

## Interfaces

```sros
configure
    port 1/1/1
        description "Uplink to PE2"
        ethernet
            mode network
            encap-type dot1q
            mtu 9212
        exit
        no shutdown
    exit

    router
        interface "system"
            address 10.0.0.1/32
        exit
        interface "to-PE2"
            address 10.0.1.1/30
            port 1/1/1:0
            bfd 300 receive 300 multiplier 3
        exit
    exit
exit
```

## MPLS

```sros
configure
    router
        mpls
            interface "to-PE2"
            exit
            no shutdown
        exit
        ldp
            interface-parameters
                interface "to-PE2"
                exit
            exit
            no shutdown
        exit
        rsvp
            interface "to-PE2"
            exit
            no shutdown
        exit
    exit
exit
```

## QoS

```sros
configure
    qos
        sap-ingress 100 create
            description "Customer ingress QoS"
            queue 1 create
                rate max cir 0
            exit
            queue 11 multipoint create
                rate max cir 0
            exit
            fc "ef" create
                queue 1
                priority high
            exit
            fc "be" create
                queue 1
                priority low
            exit
        exit
        sap-egress 100 create
            queue 1 create
                rate 100000 cir 50000
            exit
        exit
    exit
exit
```

## Security / Filters

```sros
configure
    filter
        ip-filter 10 create
            description "Protect management"
            entry 10 create
                match protocol tcp
                    dst-port eq 22
                    src-ip 10.0.10.0/24
                exit
                action forward
            exit
            entry 20 create
                match protocol udp
                    dst-port eq 161
                    src-ip 10.0.10.0/24
                exit
                action forward
            exit
            entry 30 create
                match protocol tcp
                    dst-port eq 179
                exit
                action forward
            exit
            entry 1000 create
                action drop
            exit
        exit
    exit
exit

# Apply to management
configure
    router
        interface "system"
            ingress
                filter ip 10
            exit
        exit
    exit
exit
```

## SNMP

```sros
configure
    system
        snmp
            community "<COMMUNITY>" hash2 r version v2c
            packet-size 9216
        exit
        location "Brisbane POP"
        contact "noc@athena.net.au"
    exit
exit
```

## Troubleshooting Commands

```sros
# BGP
show router bgp summary
show router bgp neighbor 203.0.113.1
show router bgp routes
show router bgp routes 192.0.2.0/24 detail
show router bgp group "TRANSIT"

# OSPF
show router ospf neighbor
show router ospf interface
show router ospf database

# MPLS
show router mpls interface
show router ldp session
show router ldp bindings active
show router mpls lsp

# Services
show service id 100 base
show service id 100 sap
show service id 100 sdp
show service sdp
show service service-using

# General
show router route-table
show router interface
show port 1/1/1
show port 1/1/1 detail
show system information
show log log-id 99

# Debug
debug router bgp neighbor 203.0.113.1
tools dump router bgp routes
```

## Common Gotchas

- **Service model**: Everything is a service in SR OS. Even basic L3 routing uses the "Base" router context, which is itself a service. Understanding SAP/SDP is essential
- **Policy commit**: Route policies require `commit` within the `policy-options` context — separate from the general config save
- **Port modes**: Ports must be set to `network` or `access` mode before use in services
- **Encap types**: Must match between port config and SAP — `dot1q`, `qinq`, `null`
- **Customer objects**: Services require a customer ID. Create customers under `configure service customer`
- **No shutdown**: Many objects are created in a shutdown state by default. Don't forget `no shutdown`
- **Admin save**: Config isn't persistent until `admin save`. Changes to running config survive reboot only after save
- **SDP binding**: SDPs must be operationally up before services using them will come up. Check far-end reachability
