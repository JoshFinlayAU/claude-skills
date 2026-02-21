# Australian Voice & Regulatory Reference

## Table of Contents

1. [ACMA Regulatory Framework](#acma-regulatory-framework)
2. [Carrier Licensing](#carrier-licensing)
3. [Number Management](#number-management)
4. [Number Porting (LNP)](#number-porting)
5. [Emergency Services (000/106)](#emergency-services)
6. [CLI and Caller ID Rules](#cli-rules)
7. [Do Not Call Register](#do-not-call)
8. [Wholesale Voice Interconnect](#wholesale-interconnect)
9. [NBN Voice](#nbn-voice)
10. [Key Industry Bodies and References](#key-references)

---

## ACMA Regulatory Framework

The Australian Communications and Media Authority (ACMA) is the regulator for
telecommunications in Australia. Key legislation:

- **Telecommunications Act 1997** — Core regulatory framework
- **Telecommunications (Consumer Protection and Service Standards) Act 1999**
- **Telecommunications Numbering Plan 2015** — Rules for number allocation and use
- **Telecommunications (Interception and Access) Act 1979** — Lawful interception requirements

### Carrier vs Carriage Service Provider (CSP)

- **Carrier** — Owns network infrastructure (cables, towers, exchanges). Must hold a carrier
  licence from ACMA. Subject to carrier obligations (interception, emergency services, etc.)
- **Carriage Service Provider (CSP)** — Provides services USING carrier infrastructure. No
  licence required, but must comply with service provider rules (TCP Code, number portability,
  etc.)
- **Content Service Provider** — Provides content (not relevant to voice).

Most VoIP providers operate as CSPs using carrier infrastructure. If you own and operate your
own network infrastructure, you likely need a carrier licence.

---

## Carrier Licensing

A carrier licence is required if you:
- Own or operate telecommunications network units (cables, towers, exchanges) in Australia
- This includes fibre networks, fixed wireless infrastructure, etc.

Carrier obligations include:
- Emergency services access (000)
- Lawful interception capability
- National numbering plan compliance
- Contribution to Telecommunications Industry Ombudsman (TIO)
- Annual carrier reporting to ACMA

CSP obligations (no licence needed):
- Comply with the Telecommunications Consumer Protections (TCP) Code
- Participate in number portability
- Handle complaints through internal process + TIO
- Contribute to industry levies (USO, TIO, etc.)

---

## Number Management

### Number Allocation

Numbers in Australia are managed by ACMA through the Integrated Public Number Database (IPND)
and the Australian Number Administration (ANA).

- **Geographic numbers (0X XXXX XXXX)** — Allocated to carriers/CSPs by ACMA in blocks
  (typically 10,000-number blocks). Must be used in the correct geographic area (charging zone).
- **Mobile numbers (04XX)** — Allocated to mobile carriers.
- **13/1300 numbers** — Inbound-only, local-rate. Allocated via ACMA auction or available
  through providers. Can be "smartnumbered" (vanity numbers).
- **1800 numbers** — Inbound-only, free-call. Same allocation as 13/1300.
- **0550 numbers** — Location-independent (VoIP numbers). Allocated to VoIP providers.
  Not commonly used — most VoIP providers use geographic numbers instead.

### IPND (Integrated Public Number Database)

The IPND is the national database of telephone numbers and their associated service addresses.

- **Mandatory**: All carriers and CSPs must provide customer records to the IPND
- Used by emergency services to determine caller location
- Used for directory assistance (1223, White Pages)
- Privacy flag available — customers can choose to be unlisted
- Updates should be made within 5 business days of service activation/change/disconnection
- IPND data manager: Currently Telstra (as designated by ACMA)

### Number Allocation for VoIP Providers

If you're a CSP providing VoIP:
1. You can obtain geographic numbers from an upstream carrier/wholesaler
2. You can apply directly to ACMA for number allocation (if you have sufficient scale)
3. You must register these numbers in the IPND with customer address details
4. Numbers must be assigned to the correct charging zone

---

## Number Porting (LNP)

Local Number Portability allows customers to keep their number when changing providers.

### How Porting Works in Australia

The porting process is managed through the Number Portability Administrator (currently AEMO).

1. **Customer requests port** — Customer authorises the new provider (Gaining CSP) to port
   their number.
2. **Porting request submitted** — Gaining CSP submits a porting request via the industry
   porting system.
3. **Validation** — Losing CSP validates the request (customer details must match).
4. **Porting window** — Simple ports: 1-2 business days. Complex ports: up to 10 business
   days.
5. **Cut-over** — At the scheduled time, the number routes to the Gaining CSP's network.
   The Losing CSP redirects calls via the porting network.

### Porting Categories

- **Simple port** — Single number, standard residential/business line. 1 business day.
- **Complex port** — Multiple numbers, ISDN, special services. Up to 10 business days.
- **Emergency return** — If porting goes wrong, an emergency return can be requested to
  route the number back to the Losing CSP within hours.

### Technical Implementation

When a number is ported, call routing uses one of two methods:

1. **All Call Query (ACQ)** — Every call to a ported number triggers a database lookup to
   find the current serving network. Used by most carriers.
2. **Onward Routing (OR)** — The original carrier receives the call and forwards it to the
   porting network, which then routes to the gaining carrier. Less efficient.

For a VoIP provider, you'll typically receive ported numbers via your upstream carrier's
SIP trunk. The carrier handles the porting network integration.

### Common Porting Issues

- **Account name mismatch** — Customer name on the account must exactly match what the
  Losing CSP has on record. Any discrepancy will cause a rejection.
- **Service qualification** — Some services (e.g., alarm lines, EFTPOS) may have
  restrictions on porting.
- **Partial porting** — Porting some numbers from a multi-number service (e.g., porting 2
  of 5 DDI numbers) can be complex.

---

## Emergency Services

### 000 / 106 Requirements

All carriers and CSPs providing voice services MUST provide access to emergency services.

- **000** — Voice emergency number (police, fire, ambulance)
- **106** — TTY/text relay emergency number (hearing/speech impaired)
- **112** — Also works on mobile (GSM standard) but routes to 000

### Technical Requirements

1. **Routing**: Emergency calls MUST be routed to the Emergency Call Person (ECP), which is
   currently Telstra. The ECP then connects the call to the appropriate PSAP (Public Safety
   Answering Point).

2. **CLI delivery**: The caller's number MUST be delivered to the ECP. Blocking caller ID on
   emergency calls is not permitted.

3. **Location information**: IPND data is used to determine caller location for fixed-line
   services. For VoIP, the registered service address in the IPND is used.

4. **VoIP-specific issues**:
   - VoIP users may be at a different physical location than the service address in IPND
   - ACMA requires VoIP providers to inform customers about the limitations of 000 over VoIP
   - Some VoIP providers require customers to update their registered address if they move
   - Power outages will affect VoIP (unlike traditional PSTN lines with battery backup)

5. **Call handling**: Emergency calls must NEVER be blocked, even if the customer has
   overdue payments or service restrictions.

### Implementation

```ini
# Asterisk dialplan — ALWAYS route 000 and 112
# These patterns should exist in EVERY context that allows outbound calls
exten => 000,1,NoOp(=== EMERGENCY CALL FROM ${CALLERID(num)} ===)
 same => n,Set(CALLERID(num)=${MAIN_DID})  ; Ensure valid CLI
 same => n,Dial(PJSIP/000@emergency-trunk,120)
 same => n,Dial(PJSIP/000@backup-trunk,120)  ; Failover
 same => n,Hangup()

exten => 112,1,Goto(000,1)  ; Route 112 same as 000
```

**IMPORTANT**: Test emergency call routing during deployment. Coordinate with your upstream
carrier for test procedures — DO NOT make test calls to actual 000 unless authorised.

---

## CLI Rules

### Calling Line Identification (CLI) Requirements

ACMA and the Communications Alliance have rules about CLI presentation:

1. **Valid CLI**: Outbound calls MUST present a valid, dialable Australian number that the
   calling party is authorised to use.

2. **No spoofing**: Presenting a CLI you're not authorised to use is prohibited. This includes
   using another organisation's number or a non-existent number.

3. **Withheld CLI**: Callers can choose to withhold their CLI (present anonymous). However,
   the actual CLI must still be delivered to the network for emergency services and law
   enforcement purposes (via PAI header).

4. **1300/1800 numbers**: These CANNOT be used as outbound CLI. They are inbound-only.

5. **International CLI**: For calls originating in Australia, the CLI should be an Australian
   number in E.164 format (+614XXXXXXXX, +61XXXXXXXX).

### Technical Implementation

```ini
# Asterisk — set CLI for outbound calls
# In the extension definition:
[1001]
type=endpoint
...
callerid="Josh Finlay" <0712345678>  ; Internal caller ID

# In the dialplan, override for trunk:
exten => _0[2-9]XXXXXXXX,1,Set(CALLERID(num)=0712345678)
 same => n,Set(PJSIP_HEADER(add,P-Asserted-Identity)=<sip:+6171234567@provider.com>)
 same => n,Dial(PJSIP/${EXTEN}@trunk)
```

For carrier-grade: Use P-Asserted-Identity (PAI) headers for the "real" CLI between trusted
networks, and From header for display purposes.

---

## Do Not Call Register

The Australian Do Not Call Register is maintained by ACMA.

- Businesses making telemarketing calls or sending marketing faxes must check numbers against
  the DNCR before contact.
- Exemptions exist for charities, political parties, educational institutions, and certain
  research organisations.
- Fines for non-compliance can be significant.
- The DNCR API allows automated checking (subscription-based).
- Not directly relevant to PBX configuration, but relevant if the voice system is used for
  outbound campaigns.

---

## Wholesale Voice Interconnect

### SIP Trunking Providers in Australia

Major wholesale voice providers that VoIP operators typically interconnect with:

| Provider | Notes |
|----------|-------|
| Telstra Wholesale | Largest carrier, direct 000 routing, comprehensive geographic coverage |
| Optus Wholesale | Second largest, strong mobile coverage |
| TPG/iiNet Wholesale | Competitive pricing, good data network |
| Symbio Networks | VoIP-focused wholesale, API-driven, popular with small carriers |
| VoiceHost | Wholesale SIP, popular with MSPs |
| Nexgen Networks | Enterprise-focused |
| Superloop | Data-centre and interconnect focused |

### Interconnect Considerations

1. **Codec**: Most AU wholesale providers prefer G.711 A-law (PCMA) on interconnects.
   G.729 may be accepted but check with the provider.

2. **Number format**: E.164 is standard (+61XXXXXXXXX). Some providers accept national format
   (0XXXXXXXXX). Clarify with provider — mismatched format is a common issue.

3. **Capacity**: SIP trunks may have a concurrent call limit. Plan for burst capacity
   (typically 2-3x average concurrent calls).

4. **Failover**: Configure at least two SIP trunk providers, or use the provider's redundant
   SBC addresses. DNS SRV records are the preferred method for failover.

5. **CDR reconciliation**: Match your CDR records against the provider's billing CDRs monthly.
   Discrepancies happen, especially with call duration rounding.

6. **CLI pass-through**: Verify that your authorised CLIs are whitelisted with the provider.
   Some providers reject calls with unrecognised CLIs.

---

## NBN Voice

### Voice on the NBN

The NBN (National Broadband Network) migration has moved voice services from traditional copper
PSTN to IP-based delivery.

- **NBN FTTP/FTTC/HFC**: Voice is delivered as a VoIP service over the NBN data connection.
  The RSP (Retail Service Provider) provides the VoIP service, not NBN Co.

- **NBN FTTN/FTTB**: Some services may still use the existing copper for voice (UNI-V port),
  but this is being phased out in favour of VoIP.

- **UNI-V (Voice) port**: On FTTP NTDs, there is a dedicated voice port (RJ11) that provides
  a traditional phone interface with battery backup. This is managed by NBN Co and delivers
  voice as a separate service. However, NBN Co is moving away from UNI-V.

- **Battery backup**: UNI-V has battery backup for emergency calls during power outages. VoIP
  over the data port (UNI-D) does NOT have battery backup — this is a compliance consideration.

### For VoIP Providers on NBN

If you're providing voice over NBN:
1. Your service runs over the RSP's data connection (UNI-D)
2. You need to ensure sufficient bandwidth and QoS for voice
3. You need to handle 000 routing through your network
4. You should inform customers about power outage limitations
5. Consider recommending a UPS for the NBN NTD + router for critical voice services

---

## Key References

| Resource | URL | Purpose |
|----------|-----|---------|
| ACMA | acma.gov.au | Regulatory body |
| Communications Alliance | commsalliance.com.au | Industry standards and codes |
| TIO | tio.com.au | Telecommunications Industry Ombudsman |
| ACMA Numbering Plan | legislation.gov.au | Telecommunications Numbering Plan 2015 |
| TCP Code | commsalliance.com.au/Activities/tcp | Consumer protection code |
| IPND | acma.gov.au/ipnd | Integrated Public Number Database |
| Do Not Call | donotcall.gov.au | DNCR registration and API |
| AEMO (Porting) | aemo.com.au | Number portability administration |

### Industry Standards (Communications Alliance)

- **C559** — Requirements for Customer Equipment for connection to a telecommunications network
- **C524** — Number Portability industry code
- **C525** — Handling of Life Threatening and Unwelcome Communications
- **C628** — Telecommunications Consumer Protections Code (TCP Code)
- **G616** — Integrated Public Number Database (IPND) — Industry Code
- **G654** — Emergency Call Services Requirements (draft, check for current version)
