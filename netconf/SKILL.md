---
name: netconf
description: Expert NETCONF (RFC 6241) skill for managing carrier-grade routing platforms via YANG-modelled XML-over-SSH. Covers protocol mechanics, datastore semantics, transactional configuration, capability discovery, subtree/XPath filtering, and the vendor-specific behaviour that actually matters in production - Nokia SR OS (7750 SR / 7250 IXR), IP Infusion OcNOS, Huawei VRP, Cisco IOS-XR/XE, and Juniper Junos. Includes tooling (ncclient, netopeer2, yangcli, pyang), YANG model family selection (IETF, OpenConfig, native), and worked examples for BGP, OSPF, IS-IS, MPLS, Eline/Epipe and VPLS provisioning.
risk: medium
source: kinetix-networks
date_added: '2026-04-30'
---

# NETCONF for Carrier-Grade Routing

## Use this skill when

- Building or extending automation that reads or writes router configuration via NETCONF (RFC 6241)
- Working with YANG models (IETF, OpenConfig, or vendor-native) on carrier devices
- Provisioning BGP, OSPF, IS-IS, MPLS, Eline/Epipe, VPLS, or L3VPN services programmatically
- Debugging `<rpc-error>` responses, capability mismatches, or namespace problems
- Choosing between `:candidate`, `:writable-running`, and `:confirmed-commit` workflows
- Integrating NETCONF data into a source-of-truth like Infrahub, NetBox, or a custom inventory
- Selecting between subtree filters and XPath filters for `<get>` / `<get-config>`
- Comparing OpenConfig vs vendor-native YANG for a specific feature on a specific platform

## Do not use this skill when

- The device exposes only legacy CLI or SNMP and has no NETCONF/YANG agent (use screen-scraping libraries like Netmiko/NAPALM instead)
- The work is purely gNMI/gRPC-based - those share YANG semantics but use a different RPC layer and encoding (Protobuf, not XML)
- The task is RESTCONF (RFC 8040) - the data models overlap but the wire protocol is HTTP+JSON and operation semantics differ slightly
- Reading SNMP MIBs or writing CLI templates - those are different management planes

## Instructions

1. **Confirm the management plane is enabled and the device is in the right mode.** Some platforms (Nokia SR OS, Junos with `system services netconf`) require explicit enablement. SR OS additionally requires `model-driven` configuration mode; the `classic` default rejects model-driven writes.
2. **Always inspect `<hello>` capabilities first.** The set of advertised capability URIs determines which datastores, operations, and YANG modules are usable. Never assume `:candidate`, `:confirmed-commit`, `:validate`, or `:writable-running` are present.
3. **Pick the YANG family deliberately.** OpenConfig gives portability across vendors but with feature gaps and deviations. Native modules (e.g. `nokia-conf`, `Cisco-IOS-XR-*`, `huawei-*`) give complete feature coverage at the cost of vendor lock-in. Mixing both within one transaction is supported but increases blast radius.
4. **Use the candidate datastore and `<commit>` whenever the platform supports it.** Direct writes to `:running` are atomic per RPC but cannot batch related changes; a half-applied configuration is much harder to roll back than an uncommitted candidate.
5. **Validate before commit, and prefer `:confirmed-commit` for risky changes.** A confirmed commit auto-rolls back if no follow-up commit arrives inside the timeout, surviving a session loss caused by the very change being made.
6. **Filter aggressively on `<get>`.** Carrier routers can return multi-megabyte state responses; subtree filters or XPath filters cut payload size and processing time by orders of magnitude.
7. **Treat XML namespaces as load-bearing.** A right-shaped payload in the wrong namespace is silently ignored or rejected with `unknown-element`. Always namespace the top-level element of every fragment.
8. **Log the raw RPC and reply on first run of any new operation.** The XML on the wire is the contract; pretty-printed Python objects hide bugs.
9. **Stop and ask the operator for confirmation before any write to `:running` on a production device** if the task brief is ambiguous about scope, peers, or commit policy.

## Purpose

NETCONF is an IETF standardised configuration management protocol (RFC 6241, June 2011) that exchanges XML-encoded RPCs over a secure transport - usually SSH on TCP/830 (RFC 6242), occasionally TLS on TCP/6513 (RFC 7589) or Call Home (RFC 8071). It is the dominant model-driven management plane for carrier routers from Nokia, Cisco, Juniper, Huawei, and IP Infusion, and is the substrate on which Infrahub, NSO, Crosswork, and most in-house automation is built.

NETCONF's value over CLI scraping is structural: every operation has a defined success/failure semantic, every datastore has explicit read and write rules, and every leaf is described by a YANG schema (RFC 7950) you can introspect. Its value over SNMP is that it was designed for configuration: it has transactions, locking, validation, and rollback as first-class primitives.

## Protocol Architecture

NETCONF is a four-layer protocol (RFC 6241 §1.2):

| Layer | Purpose | Examples |
|-------|---------|----------|
| Content | The data being exchanged | `<interface>`, `<bgp>`, `<network-instance>` |
| Operations | What to do with the content | `<get-config>`, `<edit-config>`, `<commit>` |
| Messages | RPC framing | `<rpc>`, `<rpc-reply>`, `<notification>` |
| Transport | Secure, connection-oriented byte stream | SSH, TLS, BEEP (deprecated) |

Every message is a well-formed XML document. RPCs carry a `message-id` that the reply echoes, allowing pipelining (a client may send N RPCs without waiting for replies and correlate them on receipt).

## Transport

### NETCONF over SSH (RFC 6242) - the default and most common

- IANA port **830/TCP** (some vendors also expose port 22 with the `netconf` SSH subsystem name)
- Authentication uses normal SSH mechanisms: password, public key, keyboard-interactive, certificate, or RADIUS/TACACS+ proxied through SSH
- The client requests the SSH subsystem `netconf` after connection - any modern NETCONF library does this automatically
- Framing: NETCONF 1.0 uses the `]]>]]>` end-of-message marker (don't put that string in your payload); NETCONF 1.1 uses chunked framing (`\n#<chunk-size>\n` ... `\n##\n`). The framing version is negotiated during `<hello>`

### NETCONF over TLS (RFC 7589)

- IANA port **6513/TCP**
- Mutual TLS - both sides present X.509 certificates
- Useful where SSH is policy-restricted; rare in service provider networks today

### NETCONF Call Home (RFC 8071)

- The device initiates the TCP connection back to a registered manager
- Default ports **4334/TCP** (SSH Call Home) and **4335/TCP** (TLS Call Home)
- Useful for devices behind NAT or zero-touch provisioning workflows

## Capabilities Exchange

Both peers send a `<hello>` immediately on session open advertising their supported capabilities as URIs. The intersection determines what is legal in that session.

```xml
<hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <capabilities>
    <capability>urn:ietf:params:netconf:base:1.1</capability>
    <capability>urn:ietf:params:netconf:capability:candidate:1.0</capability>
    <capability>urn:ietf:params:netconf:capability:confirmed-commit:1.1</capability>
    <capability>urn:ietf:params:netconf:capability:validate:1.1</capability>
    <capability>urn:ietf:params:netconf:capability:rollback-on-error:1.0</capability>
    <capability>urn:ietf:params:netconf:capability:xpath:1.0</capability>
    <capability>urn:ietf:params:netconf:capability:notification:1.0</capability>
    <capability>urn:ietf:params:netconf:capability:with-defaults:1.0?basic-mode=explicit</capability>
    <capability>urn:nokia.com:nc:sros:major-release-23:minor-release-7</capability>
    <!-- ...one capability per supported YANG module... -->
  </capabilities>
  <session-id>4271</session-id>
</hello>
```

### Capabilities you should look for

| URI suffix | What it means | Why you care |
|---|---|---|
| `base:1.0` / `base:1.1` | Protocol version (framing) | Determines wire framing; always negotiate the highest both sides support |
| `candidate:1.0` | A `<candidate>` datastore exists | Enables transactional configuration |
| `confirmed-commit:1.1` | Confirmed commit with timeout/persist | Lets you safely change management-plane reachability |
| `writable-running:1.0` | `<edit-config>` may target `<running/>` directly | Often *disabled* on transactional platforms |
| `validate:1.1` | `<validate>` operation supported | Lets you syntax/semantic-check before commit |
| `rollback-on-error:1.0` | `error-option=rollback-on-error` honoured | Atomic edits where the platform supports it |
| `startup:1.0` | A separate `<startup>` datastore exists | Required if you want changes to survive reboot independently of running |
| `xpath:1.0` | XPath filters in `<get>`/`<get-config>` | Much more expressive than subtree filtering |
| `url:1.0` | `<url>` may substitute for inline content | Useful for very large config payloads |
| `notification:1.0` | Subscriptions per RFC 5277 | Streaming events, syslog over NETCONF |
| `yang-library:1.0` | RFC 8525 YANG library available | Programmatic schema discovery |
| `with-defaults:1.0` | `<with-defaults>` parameter (RFC 6243) | Controls whether defaults are returned |

## Datastores

Datastores are the heart of NETCONF semantics. Get them wrong and everything else falls apart.

### Classic datastores (RFC 6241)

- **`<running/>`** - the configuration currently driving the device. Always present. Direct writes require `:writable-running`.
- **`<candidate/>`** - a scratchpad copy you edit and then `<commit>` to running. Requires `:candidate`. Multiple `<edit-config>` operations against `<candidate/>` accumulate; `<commit>` applies them as one transaction; `<discard-changes>` throws them away.
- **`<startup/>`** - the configuration loaded at next boot. Requires `:startup`. On many platforms (Nokia SR OS, Junos) startup is implicitly written by `commit` or by an explicit `<copy-config>`.

### NMDA datastores (RFC 8342, RFC 8526) - newer architecture

- **`<intended>`** - the configuration the operator *intends* to be active, after template/group expansion but before any platform-side rejection (read-only)
- **`<operational>`** - the actual operational state, including data not derived from configuration (link state, BGP session state, learned routes)
- New operations: `<get-data>` and `<edit-data>` with a `datastore` parameter that names the target by URI

NMDA matters because pre-NMDA `<get>` mixed configuration and state in a way that made it hard to reason about. On NMDA-aware platforms, prefer `<get-data>` with `datastore="ds:operational"` for state and `datastore="ds:running"` for configuration.

### Vendor specifics worth knowing

- **Nokia SR OS** advertises `:candidate`, `:startup`, and `:writable-running` *but* SR OS strongly recommends disabling `writable-running` (`/configure system management-interface netconf capabilities writable-running false`) to force transactional behaviour. The `<startup>` datastore on SR OS only supports `<copy-config>` and `<delete-config>` - you cannot `<edit-config>` it directly. SR OS also exposes `<intended>` (read-only post-expansion) and supports the NMDA `<get-data>` / `<edit-data>` operations.
- **Cisco IOS-XR** has `:candidate` and full transactional semantics; IOS-XE historically only has `:writable-running` but newer releases also expose `:candidate`.
- **Junos** uses its own private candidate model - every session that enters configuration mode gets its own candidate; `commit` merges to shared running.
- **OcNOS** uses an implicit transaction model: each `<edit-config>` opens a transaction that is closed by `<commit>` or `<discard-changes>`.
- **Huawei VRP** advertises `:candidate` and `:writable-running`; behaviour matches the IETF baseline closely.

## Protocol Operations

### Read operations

#### `<get>` - configuration *and* operational state

```xml
<rpc message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <get>
    <filter type="subtree">
      <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces"/>
    </filter>
  </get>
</rpc>
```

#### `<get-config>` - configuration only, from a named datastore

```xml
<rpc message-id="102" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <get-config>
    <source><running/></source>
    <filter type="xpath" select="/oc-netinst:network-instances/oc-netinst:network-instance[oc-netinst:name='default']/oc-netinst:protocols"
            xmlns:oc-netinst="http://openconfig.net/yang/network-instance"/>
  </get-config>
</rpc>
```

#### `<get-data>` - NMDA-aware read

```xml
<rpc message-id="103" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <get-data xmlns="urn:ietf:params:xml:ns:yang:ietf-netconf-nmda">
    <datastore xmlns:ds="urn:ietf:params:xml:ns:yang:ietf-datastores">ds:operational</datastore>
    <subtree-filter>
      <bgp xmlns="http://openconfig.net/yang/bgp"/>
    </subtree-filter>
    <with-defaults xmlns="urn:ietf:params:xml:ns:yang:ietf-netconf-with-defaults">report-all</with-defaults>
  </get-data>
</rpc>
```

### Write operations

#### `<edit-config>` - the workhorse

Per-element `nc:operation` attribute controls behaviour for each subtree:

| Operation | Semantics |
|-----------|-----------|
| `merge` (default) | Add/update; existing siblings untouched |
| `replace` | Replace this subtree exactly; missing children deleted |
| `create` | Like merge, but error if the target already exists |
| `delete` | Remove the target; error if it doesn't exist |
| `remove` | Remove the target; succeed silently if it doesn't exist |

The `<default-operation>` element on the RPC sets the fallback (`merge`, `replace`, or `none`). `none` is the most defensive: edits do *nothing* unless they have an explicit `nc:operation`.

The `<error-option>` element controls atomicity:
- `stop-on-error` (default) - stop processing on first error; partially-applied edits remain
- `continue-on-error` - apply what you can, report what you can't
- `rollback-on-error` - undo everything if anything fails. **Requires `:rollback-on-error` capability.** This is what you almost always want.

```xml
<rpc message-id="201" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <edit-config>
    <target><candidate/></target>
    <default-operation>none</default-operation>
    <error-option>rollback-on-error</error-option>
    <config>
      <network-instances xmlns="http://openconfig.net/yang/network-instance">
        <network-instance>
          <name>default</name>
          <protocols>
            <protocol>
              <identifier xmlns:oc-pol-types="http://openconfig.net/yang/policy-types">oc-pol-types:BGP</identifier>
              <name>bgp</name>
              <bgp>
                <neighbors>
                  <neighbor xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" nc:operation="merge">
                    <neighbor-address>192.0.2.10</neighbor-address>
                    <config>
                      <neighbor-address>192.0.2.10</neighbor-address>
                      <peer-as>65001</peer-as>
                      <description>upstream-A</description>
                    </config>
                  </neighbor>
                </neighbors>
              </bgp>
            </protocol>
          </protocols>
        </network-instance>
      </network-instances>
    </config>
  </edit-config>
</rpc>
```

#### `<copy-config>` - replace an entire datastore

Useful for backup/restore: `<copy-config><source><running/></source><target><url>file:///tmp/today.xml</url></target></copy-config>`. Note SR OS restricts `<copy-config>` URL combinations - source-and-target both as URLs is not supported, and URL→running is not supported.

#### `<delete-config>` - empty a datastore

Cannot target `<running/>`. Practically only useful for `<startup/>`.

### Transaction operations

#### `<lock>` and `<unlock>`

```xml
<rpc message-id="301" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <lock><target><candidate/></target></lock>
</rpc>
```

Locks the named datastore against other sessions. Always unlock in a `finally` block. SR OS additionally maintains an *implicit* lock on `<running/>` once you start editing `<candidate/>` - this also blocks classic CLI users from making changes until you `<commit>` or `<discard-changes>`.

#### `<commit>`

```xml
<rpc message-id="401" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <commit>
    <confirmed/>
    <confirm-timeout>120</confirm-timeout>
    <persist>commit-id-001</persist>
  </commit>
</rpc>
```

`<confirmed/>` plus `<confirm-timeout>` makes the commit revert automatically unless a follow-up `<commit>` lands inside the timeout. `<persist>` makes the confirmed commit survive a session drop - the follow-up confirming commit can come from a *different* session that supplies the same `<persist-id>`.

#### `<discard-changes>`

Wipes the candidate. Releases SR OS's implicit running lock. No effect on running.

#### `<validate>`

```xml
<rpc message-id="501" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <validate><source><candidate/></source></validate>
</rpc>
```

Runs schema and semantic checks (must/when expressions, leafrefs, identityrefs) without applying. Cheap insurance before commit.

### Session operations

- `<close-session>` - graceful close. Releases all locks held by this session.
- `<kill-session>` - forcibly terminate another session by `session-id`. Requires elevated privilege. Useful when a colleague's session has wedged a lock.

## Filtering

### Subtree filtering (mandatory in all servers)

A skeletal XML structure where present elements act as containment filters and empty elements act as leaf selectors. Powerful but verbose for cross-tree selection.

```xml
<filter type="subtree">
  <network-instances xmlns="http://openconfig.net/yang/network-instance">
    <network-instance>
      <name>default</name>
      <protocols>
        <protocol>
          <identifier>oc-pol-types:BGP</identifier>
          <name>bgp</name>
          <!-- empty - selects everything under this protocol -->
        </protocol>
      </protocols>
    </network-instance>
  </network-instances>
</filter>
```

### XPath filtering (requires `:xpath:1.0`)

```xml
<filter type="xpath"
        xmlns:oc-ni="http://openconfig.net/yang/network-instance"
        xmlns:oc-bgp="http://openconfig.net/yang/bgp"
        select="/oc-ni:network-instances/oc-ni:network-instance/oc-ni:protocols/oc-ni:protocol/oc-bgp:bgp/oc-bgp:neighbors/oc-bgp:neighbor[oc-bgp:state/oc-bgp:session-state='ESTABLISHED']"/>
```

XPath shines for "find all BGP sessions that are established", "find every interface in a specific VRF", and similar predicate-driven selection. Subtree is mandatory and ubiquitous; XPath is optional and not all servers implement it well.

## YANG Essentials

YANG (RFC 7950, version 1.1) describes the schema of the content layer. You don't need to write YANG to use NETCONF, but you do need to read it.

The fragments you will care about:

- **module** - top-level container, has a namespace and a prefix
- **container** - a non-leaf node grouping related data (e.g. `<bgp>`, `<global>`)
- **list** - keyed collection (e.g. `neighbor`, keyed by `neighbor-address`)
- **leaf** - terminal value (e.g. `peer-as`)
- **leaf-list** - ordered or unordered list of values
- **augment** - adds nodes from an external module to an existing path
- **deviation** - declares that a server *deviates* from the published model (a leaf is read-only, a feature is unsupported, a range is narrower)
- **identityref / identity** - extensible enumerations (BGP, ISIS, OSPF as identities under the BASE identity)

To explore a model offline use `pyang`:

```bash
pip install pyang
pyang -f tree openconfig-bgp.yang openconfig-bgp-common.yang openconfig-bgp-types.yang
pyang -f jstree --tree-print-no-expand-uses openconfig-network-instance.yang  > netinst.html
```

To pull schemas straight from the device (requires `:netconf-monitoring` per RFC 6022):

```xml
<rpc message-id="601" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <get-schema xmlns="urn:ietf:params:xml:ns:yang:ietf-netconf-monitoring">
    <identifier>openconfig-bgp</identifier>
    <version>2021-06-16</version>
    <format>yang</format>
  </get-schema>
</rpc>
```

## YANG Model Family Selection

Three families coexist on every modern carrier router. Choose deliberately.

### IETF models

Standardised, modest feature scope, mostly suitable for inventory and lightweight reads. Examples: `ietf-interfaces`, `ietf-ip`, `ietf-routing`, `ietf-system`, `ietf-keystore`. Useful for portable interface and IP-address operations; usually inadequate for full BGP/MPLS configuration on a service provider router.

### OpenConfig models

Vendor-neutral, designed by an operator-led working group, much richer than IETF models for routing protocols. The core modules are:

- `openconfig-interfaces` (+ `openconfig-if-ethernet`, `openconfig-if-aggregate`, `openconfig-vlan`)
- `openconfig-network-instance` - the umbrella for everything VRF/L3VPN/L2VPN, and the parent of routing-protocol instances
- `openconfig-bgp` (+ `openconfig-bgp-policy`)
- `openconfig-isis`, `openconfig-ospfv2`
- `openconfig-mpls` (+ `openconfig-mpls-rsvp`, `openconfig-mpls-ldp`, `openconfig-mpls-sr`)
- `openconfig-segment-routing`
- `openconfig-routing-policy`
- `openconfig-system`
- `openconfig-platform`

Vendors publish `*-deviations.yang` files alongside their releases (e.g. `nokia-sr-openconfig-bgp-deviations.yang`) declaring exactly which OpenConfig leaves they don't implement, implement read-only, or have a narrower range for. **Read the deviations before you commit to OpenConfig.** A leaf that compiles fine against the upstream model may simply be ignored by the platform.

### Vendor-native models

Always the most complete option for that vendor. Names follow predictable patterns:

- **Nokia SR OS**: `nokia-conf` (top-level config), `nokia-state` (top-level state), `nokia-types-*` for type definitions, with submodules per feature area (`nokia-conf-router`, `nokia-conf-service`, `nokia-conf-system`). The XML namespace is `urn:nokia.com:sros:ns:yang:sr:conf`. There is also an older Alcatel-Lucent Base-R13 model (`alu-conf-*-r13`, namespace `urn:alcatel-lucent.com:sros:ns:yang:conf-*-r13`) - it is not transactional, only works on `<running/>`, and Nokia advises against new use.
- **IP Infusion OcNOS**: a top-level `ZebOS` module includes submodules per feature area (`bgp`, `ospf`, `isis`, `vrf`, `interface`, `mpls`, etc.). Native models are required for advanced features; OpenConfig coverage exists but is partial.
- **Huawei VRP**: prefixed `huawei-*` (e.g. `huawei-bgp`, `huawei-ifm`, `huawei-mpls`).
- **Cisco IOS-XR**: prefixed `Cisco-IOS-XR-*` (e.g. `Cisco-IOS-XR-um-router-bgp-cfg`).
- **Cisco IOS-XE**: a unified `Cisco-IOS-XE-native` model (everything under `/native/...`) plus newer per-feature `Cisco-IOS-XE-*-cfg` modules.
- **Junos**: a single huge `junos-conf-root` plus per-feature submodules.

### Selection heuristic

- Reading inventory and operational state across mixed vendors → OpenConfig + IETF
- Reading or writing a fully-featured BGP/MPLS/L2VPN service on a single vendor → native
- Building a multivendor service abstraction (e.g. an Eline you provision on Nokia, OcNOS, and Huawei alike) → start from OpenConfig, fall back to native per-vendor where deviations bite

## Vendor-Specific Behaviour

### Nokia SR OS (7250 IXR, 7750 SR, 7950 XRS, VSR)

Enable model-driven mode and NETCONF (one-time, classic CLI):

```
/configure system management-interface configuration-mode model-driven
/configure system management-interface netconf admin-state enable
/configure system management-interface netconf listen admin-state enable
/configure system management-interface netconf capabilities writable-running false
/configure system management-interface yang-modules nokia-modules true
/configure system management-interface yang-modules openconfig-modules true
/configure system management-interface yang-modules base-r13-modules false
```

Per-user authorisation:

```
/configure system security aaa local-profiles profile "netconf-admin" netconf base-op-authorization lock true
/configure system security aaa local-profiles profile "netconf-admin" netconf base-op-authorization commit true
/configure system security aaa local-profiles profile "netconf-admin" netconf base-op-authorization kill-session true
```

User must have `access netconf` set on the local user record. NETCONF runs over SSH and supports RADIUS/TACACS+ authentication.

Counters and visibility from MD-CLI:

```
show system management-interface netconf statistics
show system management-interface netconf sessions
```

Watch out for:

- The implicit running-lock semantics described above - blocks classic CLI users while a `<candidate>` edit is open
- `<edit-config><error-option>` is fixed at `stop-on-error` regardless of what you ask for
- The `<candidate>` datastore only accepts the XML content layer; the older Nokia "CLI content layer" only works against `<running>`
- Each `<rpc>` may carry only one operation
- `<startup>` and `<url>` only support `<copy-config>` and `<delete-config>` - not `<edit-config>` / `<get-config>` / `<get>` / `<validate>`
- The Nokia `<commit>` is global - confirmed-commit semantics fully supported
- For Lawful Intercept config, only LI-authorised users see LI data via `<get>` regardless of NETCONF privileges

### IP Infusion OcNOS (carrier and data centre lines)

Native NETCONF server runs on TCP/830, supports IPv4 and IPv6. Default `max-sessions` is 1024; the 1024th attempt evicts session 1.

The native YANG hierarchy is rooted at `ZebOS`, which `include`s submodules. Always load `ZebOS` first if your client requires explicit module loading (yangcli does):

```
yangcli ocnos@10.0.0.1> mgrload ZebOS
```

Edit-config takes XML files referenced with `@`:

```
yangcli ocnos@10.0.0.1> edit-config target=candidate config=@/tmp/bgp.xml
yangcli ocnos@10.0.0.1> commit
```

OcNOS-specific behaviour:

- **Hitless merge** - re-applying configuration that already exists is silently accepted instead of raising an error. This makes idempotent automation easy but masks bugs where you wrote the *wrong* identical configuration.
- **Implicit transactions** - successive `<edit-config>` calls accumulate into one transaction until `<commit>` or `<discard-changes>`.
- **Force-unlock** - admin role can `force-unlock target=running after=60`; the holder is notified, the lock is released after 60 s.
- **URL capability** - supports `http`, `ftp`, and `file` schemes, but `<copy-config>` URL→running and URL→startup are unsupported.
- **Get caching** - large `<get>` responses are cached for performance; freshness has a per-path `lastgettime` timestamp under `/netconf-state`.

OcNOS also ships OpenConfig models on the SP-PLUS and IPADV-CE images.

### Huawei VRP (NE40E, CloudEngine, NetEngine)

NETCONF over SSHv2, default port 830. Enable with:

```
[~HUAWEI] netconf
[~HUAWEI-netconf] protocol inbound ssh port 830
[~HUAWEI-netconf] commit
```

Capability set is close to the IETF baseline. Key VRP-specific items:

- Both Schema-based and YANG-based models are exposed; YANG modules use the `huawei-*` prefix
- Supports `:candidate`, `:writable-running`, `:rollback-on-error`, `:confirmed-commit`, `:notification`
- The `commit` command in classic VRP CLI (V8) and the NETCONF `<commit>` use the same underlying transaction subsystem - mixing the two within one window is asking for pain; pick one management plane per change window
- `xml-format` extension for legacy non-YANG content; prefer the YANG-modelled namespace `urn:huawei:yang:huawei-*` for new work

### Cisco IOS-XR

Most carrier-mature Cisco platform for NETCONF. Enable:

```
ssh server v2
ssh server netconf vrf default
netconf-yang agent ssh
```

IOS-XR exposes `:candidate`, `:rollback-on-error`, `:validate:1.1`, `:confirmed-commit:1.1`, full OpenConfig coverage, and rich native models under `Cisco-IOS-XR-um-*` (unified model) and `Cisco-IOS-XR-*-cfg`. Operational state under `Cisco-IOS-XR-*-oper`.

Quirks:

- IOS-XR's `<commit>` integrates with the same commit history the CLI uses - `show configuration commit list` reflects NETCONF commits
- Confirmed-commit uses Cisco's existing commit-confirmed semantics; the `<persist>` token maps to a labelled commit
- Many models require explicit `feature` enablement before the corresponding subtree is writeable

### Cisco IOS-XE

Enable:

```
netconf-yang
```

Default exposes both NETCONF (TCP/830) and the underlying ConfD agent. Pre-16.8 had a polling-based operational data manager; 16.8+ treats operational data as a first-class NETCONF target.

The native model `Cisco-IOS-XE-native` mirrors the running-config tree (`/native/router/bgp/...`). OpenConfig models are also present but historically less complete than on IOS-XR.

### Juniper Junos

Enable:

```
set system services netconf ssh
```

Junos NETCONF predates the IETF standard - the original work was Juniper's contribution - so its semantics are slightly idiosyncratic:

- Each session enters `configure private` automatically; the candidate is per-session
- `<commit>`, `<rollback>`, `<load-configuration>` are first-class RPCs in addition to standard `<edit-config>` / `<commit>`
- Supports both XML and "text" (CLI-format) content via the `format` attribute on `<load-configuration>`
- The `junos:commit-options` extension supports `<synchronize/>` for routing-engine pairs

### Mikrotik RouterOS 7

NETCONF support on RouterOS is essentially absent; the platform's primary management APIs are the legacy proprietary API (TCP/8728, 8729 with TLS) and the newer REST API. If a workflow demands NETCONF on RouterOS, the realistic path is fronting it with a NETCONF-to-API gateway rather than expecting native server support.

## Carrier Routing Configuration Patterns

The patterns below show OpenConfig where viable and switch to native where OpenConfig deviations make it impractical.

### BGP neighbour (OpenConfig)

```xml
<network-instances xmlns="http://openconfig.net/yang/network-instance">
  <network-instance>
    <name>default</name>
    <protocols>
      <protocol>
        <identifier xmlns:p="http://openconfig.net/yang/policy-types">p:BGP</identifier>
        <name>bgp</name>
        <bgp>
          <global>
            <config>
              <as>65000</as>
              <router-id>192.0.2.1</router-id>
            </config>
          </global>
          <neighbors>
            <neighbor>
              <neighbor-address>198.51.100.2</neighbor-address>
              <config>
                <neighbor-address>198.51.100.2</neighbor-address>
                <peer-as>65001</peer-as>
                <peer-group>UPSTREAM</peer-group>
                <description>Transit-A</description>
                <enabled>true</enabled>
              </config>
              <afi-safis>
                <afi-safi>
                  <afi-safi-name xmlns:bt="http://openconfig.net/yang/bgp-types">bt:IPV4_UNICAST</afi-safi-name>
                  <config>
                    <afi-safi-name xmlns:bt="http://openconfig.net/yang/bgp-types">bt:IPV4_UNICAST</afi-safi-name>
                    <enabled>true</enabled>
                  </config>
                </afi-safi>
              </afi-safis>
            </neighbor>
          </neighbors>
        </bgp>
      </protocol>
    </protocols>
  </network-instance>
</network-instances>
```

### IS-IS L2 interface (OpenConfig)

```xml
<network-instances xmlns="http://openconfig.net/yang/network-instance">
  <network-instance>
    <name>default</name>
    <protocols>
      <protocol>
        <identifier xmlns:p="http://openconfig.net/yang/policy-types">p:ISIS</identifier>
        <name>isis-1</name>
        <isis>
          <global>
            <config>
              <instance>isis-1</instance>
              <level-capability xmlns:isis="http://openconfig.net/yang/isis-types">isis:LEVEL_2</level-capability>
            </config>
          </global>
          <interfaces>
            <interface>
              <interface-id>1/1/1</interface-id>
              <config>
                <interface-id>1/1/1</interface-id>
                <enabled>true</enabled>
                <circuit-type xmlns:isis="http://openconfig.net/yang/isis-types">isis:POINT_TO_POINT</circuit-type>
              </config>
            </interface>
          </interfaces>
        </isis>
      </protocol>
    </protocols>
  </network-instance>
</network-instances>
```

### Eline / Epipe (Nokia SR OS native)

OpenConfig L2VPN coverage is patchy on Nokia. Use native:

```xml
<configure xmlns="urn:nokia.com:sros:ns:yang:sr:conf">
  <service>
    <epipe>
      <service-name>EP-CUST-001</service-name>
      <admin-state>enable</admin-state>
      <service-id>1001</service-id>
      <customer>1</customer>
      <sap>
        <sap-id>1/1/1:100</sap-id>
        <admin-state>enable</admin-state>
      </sap>
      <spoke-sdp>
        <sdp-bind-id>10:1001</sdp-bind-id>
        <admin-state>enable</admin-state>
      </spoke-sdp>
    </epipe>
  </service>
</configure>
```

### VPLS service (Nokia SR OS native)

```xml
<configure xmlns="urn:nokia.com:sros:ns:yang:sr:conf">
  <service>
    <vpls>
      <service-name>VPLS-CUST-002</service-name>
      <admin-state>enable</admin-state>
      <service-id>2002</service-id>
      <customer>2</customer>
      <sap>
        <sap-id>1/1/2:200</sap-id>
        <admin-state>enable</admin-state>
      </sap>
      <mesh-sdp>
        <sdp-bind-id>10:2002</sdp-bind-id>
        <admin-state>enable</admin-state>
      </mesh-sdp>
    </vpls>
  </service>
</configure>
```

### MPLS LDP enable on an interface (OpenConfig)

```xml
<network-instances xmlns="http://openconfig.net/yang/network-instance">
  <network-instance>
    <name>default</name>
    <mpls>
      <signaling-protocols>
        <ldp xmlns="http://openconfig.net/yang/mpls-ldp">
          <global>
            <config>
              <lsr-id>192.0.2.1</lsr-id>
            </config>
          </global>
          <interface-attributes>
            <interface>
              <interface-id>1/1/1.0</interface-id>
              <config>
                <interface-id>1/1/1.0</interface-id>
                <hello-holdtime>15</hello-holdtime>
                <hello-interval>5</hello-interval>
              </config>
            </interface>
          </interface-attributes>
        </ldp>
      </signaling-protocols>
    </mpls>
  </network-instance>
</network-instances>
```

## Tooling

### `ncclient` (Python) - the de-facto NETCONF client library

```bash
pip install ncclient            # paramiko-based SSH
pip install 'ncclient[libssh]'  # libssh-based SSH (faster, fewer crypto bugs)
```

Minimum viable connect-and-read against a Nokia 7750:

```python
from ncclient import manager

with manager.connect(
    host="10.0.0.1",
    port=830,
    username="netconf-svc",
    key_filename="/home/josh/.ssh/id_ed25519",
    hostkey_verify=True,        # verify in production
    device_params={"name": "alu"},   # 'alu' device handler covers Nokia/Alcatel-Lucent SR OS
    timeout=30,
) as m:
    # Discover capabilities
    for c in m.server_capabilities:
        print(c)

    # Pull running config, OpenConfig BGP only
    bgp_filter = """
      <network-instances xmlns="http://openconfig.net/yang/network-instance">
        <network-instance>
          <name>default</name>
          <protocols>
            <protocol>
              <name>bgp</name>
              <bgp/>
            </protocol>
          </protocols>
        </network-instance>
      </network-instances>"""
    reply = m.get_config(source="running", filter=("subtree", bgp_filter))
    print(reply.data_xml)
```

Transactional edit with confirmed-commit:

```python
from ncclient import manager
from ncclient.xml_ import to_ele

CONFIG = """<config>
  <network-instances xmlns="http://openconfig.net/yang/network-instance">
    <network-instance>
      <name>default</name>
      <protocols>
        <protocol>
          <identifier xmlns:p="http://openconfig.net/yang/policy-types">p:BGP</identifier>
          <name>bgp</name>
          <bgp>
            <neighbors>
              <neighbor xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" nc:operation="merge">
                <neighbor-address>198.51.100.7</neighbor-address>
                <config>
                  <neighbor-address>198.51.100.7</neighbor-address>
                  <peer-as>65010</peer-as>
                  <description>peer-XYZ</description>
                </config>
              </neighbor>
            </neighbors>
          </bgp>
        </protocol>
      </protocols>
    </network-instance>
  </network-instances>
</config>"""

with manager.connect(host="10.0.0.1", port=830, username="netconf-svc",
                     key_filename="/home/josh/.ssh/id_ed25519",
                     hostkey_verify=True, device_params={"name": "alu"}) as m:
    with m.locked(target="candidate"):
        m.discard_changes()
        m.edit_config(target="candidate", config=CONFIG,
                      default_operation="none", error_option="rollback-on-error")
        m.validate(source="candidate")
        m.commit(confirmed=True, timeout=120, persist="kx-deploy-001")
        # ... operator out-of-band sanity check ...
        m.commit(persist_id="kx-deploy-001")  # confirms
```

Per-vendor `device_params['name']` values:

| Value | Vendor / OS |
|-------|-------------|
| `default` | Generic IETF behaviour |
| `alu` | Nokia / Alcatel-Lucent SR OS |
| `csr` | Cisco IOS-XE |
| `iosxr` | Cisco IOS-XR |
| `nexus` | Cisco NX-OS |
| `junos` | Juniper Junos |
| `huawei` | Huawei VRP |
| `huaweiyang` | Huawei VRP YANG-only mode |
| `h3c` | H3C Comware |

### `netopeer2-cli` (C reference client, sysrepo-based)

Useful for low-level investigation, schema retrieval, and notification subscriptions. Ships with sysrepo and libnetconf2.

### `yangcli` (OpenYuma)

The official OcNOS CLI client. Loads YANG modules locally and provides tab completion against the schema:

```
$ yangcli server=10.0.0.1 user=ocnos password=...
yangcli ocnos@10.0.0.1> mgrload ZebOS
yangcli ocnos@10.0.0.1> sget-config /vr/bgp source=running
yangcli ocnos@10.0.0.1> edit-config target=candidate config=@bgp.xml
yangcli ocnos@10.0.0.1> commit
```

### `pyang` - YANG compiler and validator

```bash
pyang -f tree --tree-depth 3 ocnos-bgp.yang
pyang --check-update-from old-bgp.yang new-bgp.yang        # compatibility check
pyang -f sample-xml-skeleton openconfig-network-instance.yang > skeleton.xml
```

### Ansible NETCONF modules

`ansible.netcommon.netconf_get`, `ansible.netcommon.netconf_config`, `ansible.netcommon.netconf_rpc`. Useful for orchestration but the abstraction is leaky - non-trivial work usually drops to raw XML inside Jinja2 templates.

### `pyangbind` - generate Python classes from YANG

Converts a YANG module into a Python class hierarchy whose attributes match the schema. Lets you build payloads with autocomplete and type checking:

```python
from pyangbind.lib.serialise import pybindIETFXMLEncoder
import oc_netinst                                # generated

ni = oc_netinst.openconfig_network_instance()
inst = ni.network_instances.network_instance.add("default")
bgp = inst.protocols.protocol.add('p:BGP', 'bgp').bgp
bgp.global_.config.as_ = 65000
xml = pybindIETFXMLEncoder.serialise(ni)
```

Investment cost is non-trivial; pays back at scale.

## Best Practices

1. **Use `default-operation=none` and explicit `nc:operation` attributes.** Default `merge` makes it too easy to accidentally retain stale state.
2. **Prefer `error-option=rollback-on-error` everywhere it's supported.** A half-applied edit is harder to recover from than no edit at all.
3. **Always lock before editing, always unlock in a `finally`.** Use `m.locked()` context managers in ncclient.
4. **Validate then commit, on platforms that support `:validate:1.1`.** Validation is cheap; mistakes in production are not.
5. **Use `:confirmed-commit` for any change that touches management plane reachability.** A change that disables your own SSH session would otherwise be unrecoverable without console access.
6. **Make every edit idempotent.** Re-running the same playbook should produce zero changes after the first apply. OcNOS hitless-merge gives this for free; on Nokia and Cisco you need to design it in.
7. **Keep one transaction per logical change.** Don't bundle "add neighbour X" with "delete neighbour Y" unless they truly belong together - it makes rollback harder to reason about.
8. **Source-of-truth diffs, not state diffs.** Compare the *intended* configuration in your IPAM/Infrahub to the rendered NETCONF payload, not the result of `<get-config>` (which may include normalisation the platform applied silently).
9. **Cache schemas locally.** `<get-schema>` is slow; pull modules once into a git repo and reference from there.
10. **Treat `<rpc-error>` carefully.** Servers report errors with `<error-tag>`, `<error-severity>`, `<error-app-tag>`, `<error-path>`, and `<error-message>`. Surface all five fields - the path is usually the giveaway for namespace mistakes.

## Common Pitfalls

- **Wrong namespace, identical structure.** A payload pasted from one vendor to another that matches structurally will fail with `unknown-element`. Always set the correct namespace on the top-level wrapper.
- **`identityref` with wrong prefix.** Values like `oc-pol-types:BGP` require the prefix to be declared on an ancestor element. Many libraries don't auto-handle this.
- **Whitespace in chunked framing.** NETCONF 1.1 chunk headers (`\n#<n>\n`) are sensitive to surrounding whitespace; let the library handle framing rather than hand-rolling it.
- **Treating `<get>` reply as ground truth for the configuration.** It includes operational state by default. Use `<get-config>` or NMDA `<get-data datastore="ds:running">` for pure config.
- **Forgetting `with-defaults`.** Different `basic-mode` values (`report-all`, `trim`, `explicit`, `report-all-tagged`) change whether implicit defaults appear in the reply. Reproducible diffs require pinning this.
- **Holding a lock through a long-running script.** Sessions that take 30 minutes to compute their next step block every other operator on the box. Lock late, unlock early.
- **Confusing capability negotiation with feature support.** A capability says the *server* understands an operation; it doesn't promise a *specific YANG model* is supported. Cross-check against the YANG library (RFC 8525) or `<hello>` capability URIs.
- **Assuming `delete` semantics match CLI `no`.** Native models often track admin-state and presence separately; `delete` removes the node entirely while CLI `no` may merely return it to default. Inspect the YANG.
- **`writable-running` writes that touch SR OS.** SR OS in model-driven mode rejects them by default - and Nokia recommends keeping it that way. Always use `<candidate>`.
- **Mixing classic CLI and NETCONF inside one change window.** SR OS's implicit lock will interfere; on VRP and IOS-XR, the commit history will be confusing. Pick one plane per change.

## Troubleshooting

### Logging the wire

ncclient logs everything at DEBUG to the `ncclient.transport.ssh` and `ncclient.operations` loggers:

```python
import logging
logging.basicConfig(level=logging.DEBUG)
logging.getLogger("ncclient.transport.ssh").setLevel(logging.DEBUG)
```

### `rpc-error` decoding cheat sheet

| `error-tag` | Common cause |
|-------------|--------------|
| `unknown-element` | Wrong namespace, or missing module not advertised in `<hello>` |
| `bad-element` | Right element, wrong content |
| `missing-attribute` | A required attribute (e.g. `nc:operation`) is absent |
| `bad-attribute` | Attribute value is wrong (e.g. operation name typo) |
| `unknown-namespace` | Server doesn't support the namespace at all |
| `access-denied` | NACM (RFC 8341) or vendor AAA blocked the operation |
| `lock-denied` | Another session holds the datastore lock |
| `data-exists` | Used `nc:operation="create"` on an existing node |
| `data-missing` | Used `nc:operation="delete"` on a missing node |
| `operation-not-supported` | Capability not advertised for what was attempted |
| `operation-failed` + app-tag | Vendor-specific - read `<error-message>` and `<error-info>` |

### NETCONF state introspection (RFC 6022)

```xml
<rpc message-id="800" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <get>
    <filter type="subtree">
      <netconf-state xmlns="urn:ietf:params:xml:ns:yang:ietf-netconf-monitoring"/>
    </filter>
  </get>
</rpc>
```

This returns active sessions, capability set, supported schemas, datastores, locks, and per-operation counters. Indispensable when diagnosing "why won't this work".

## Security

- Treat NETCONF the same as any other configuration plane: SSH keys with strong passphrases, RADIUS/TACACS+ for centralised authn/authz, NACM for fine-grained authorisation
- NACM (RFC 8341) lets you grant a service account permission to write only to a specific subtree (e.g. only neighbour configuration under one VRF) - well worth deploying
- Never disable `hostkey_verify` outside controlled lab environments
- Rotate service-account credentials and inspect the `netconf-state` session list to detect orphaned sessions

## Notifications & Telemetry

- **RFC 5277 / RFC 8639** - subscribe with `<create-subscription>`; receive `<notification>` messages until the subscription ends. Default `NETCONF` stream emits configuration-change events.
- **YANG-Push (RFC 8641)** - periodic or on-change pushes of YANG-modelled state. Useful as a low-rate telemetry channel where gNMI isn't available.
- For high-rate streaming telemetry (interface counters, BGP per-prefix updates), prefer gNMI - the protocol overhead of XML is uncomfortable beyond a few subscriptions per second.

## Reference RFCs

| RFC | Topic |
|-----|-------|
| 6020 / 7950 | YANG (1.0 / 1.1) |
| 6022 | NETCONF Monitoring |
| 6241 | NETCONF base protocol |
| 6242 | NETCONF over SSH |
| 6243 | with-defaults |
| 6470 | Base notifications |
| 6991 | Common YANG types |
| 7589 | NETCONF over TLS |
| 8040 | RESTCONF |
| 8071 | NETCONF / RESTCONF Call Home |
| 8072 | YANG Patch |
| 8341 | NACM |
| 8342 | NMDA |
| 8525 | YANG Library |
| 8526 | NETCONF NMDA extensions |
| 8639 | Subscribed notifications |
| 8640 / 8641 | NETCONF notifications / YANG-Push |

## Response Approach

1. **Determine which platform(s) are in scope** and confirm NETCONF is enabled. If state is unknown, perform a read-only capability inspection first.
2. **Map the requested change to a YANG path.** Choose IETF / OpenConfig / native deliberately based on coverage.
3. **Check vendor deviations** for the chosen model on the chosen release before committing to it.
4. **Render the XML payload** with explicit namespaces and explicit `nc:operation` attributes.
5. **Execute the operation pipeline**: lock → discard → edit → validate → commit (confirmed where appropriate) → confirm → unlock.
6. **Verify** with a targeted `<get>` or `<get-data datastore="ds:operational">` against the same subtree.
7. **Report** the diff between intended and observed, the `<rpc-error>`s if any, and the `<commit-id>` (where the platform exposes one) for audit.
8. **On failure, roll back deterministically** - either via the confirmed-commit timeout, an explicit `<discard-changes>` if the candidate hasn't been committed, or a corrective `<edit-config>` against the running datastore.

## Example Interactions

- "Pull the BGP neighbours and their session states across our 7750s and emit a CSV."
- "Provision an Eline service on a 7250 IXR using NETCONF; the SAP is 1/1/3:512 and the spoke-SDP is 100:50001."
- "Compare OpenConfig BGP coverage between Nokia SR OS 23.7 and OcNOS 6.5 - which leaves do I lose if I write portable code?"
- "Subscribe to configuration-change notifications across the core and write them to a Kafka topic."
- "An `<edit-config>` is failing with `unknown-element` on the second `<neighbor>` only - diagnose."
- "Generate a pyangbind-based Python class for the OpenConfig network-instance tree and wire it into our Infrahub renderer."
- "Build a confirmed-commit wrapper that handles `<persist>` so the confirming commit can come from a separate session after operator approval."
- "Translate this MD-CLI fragment from a Nokia commit log into the equivalent NETCONF `<edit-config>` payload."

## Limitations

- Use this skill only when the task involves NETCONF, YANG-modelled configuration, or YANG model selection. For pure CLI scraping, gNMI streaming telemetry, RESTCONF, or SNMP, reach for the appropriate domain.
- Vendor behaviour drifts release-to-release - capability sets, deviation files, and even default operation semantics can change between minor releases. When the version is unknown or unverified, surface that uncertainty before applying changes.
- Production write operations against a customer-bearing router require human approval. The skill should never silently commit to a production `<running>` datastore on its own - confirmed-commit + operator confirmation is the floor.
- The skill assumes you can read XML and YANG. It does not replace platform documentation for non-trivial native-model work; cite or link the vendor docs where the operator will need to follow up.
