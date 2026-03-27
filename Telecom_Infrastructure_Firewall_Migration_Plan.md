# Telecom Infrastructure Firewall Migration Plan

## ASA 5525-X Deployment – Network Segmentation Project

## Overview

This document outlines the plan to migrate telecom infrastructure systems behind a dedicated firewall (Cisco ASA 5525-X) to isolate critical telecom infrastructure from the enterprise LAN while allowing controlled access for SIP, management, and internet connectivity.

The goal is to protect infrastructure such as PBX servers, voice gateways, and channel banks without disrupting IP phones or transient client devices.

## Objectives

- Isolate telecom infrastructure from enterprise LAN.
- Preserve existing static IP addressing for infrastructure.
- Allow SIP trunking and RTP media through firewall.
- Allow PBX systems internet access for SIP registration and updates.
- Maintain management access from enterprise network.
- Leave IP phones and transient devices outside the firewall.
- Create a clean security boundary around critical infrastructure.
- Prepare for future VPN / zero-trust management access.

## Current State

| Network | Subnet | Notes |
|---|---|---|
| Enterprise LAN | 10.2.0.0/16 | Core network |
| VLAN 103 | 10.2.3.0/24 | Mixed devices, phones, infrastructure |
| UDMP | Default gateway for VLAN 103 | Currently routing telecom subnet |

**Problem:** Telecom infrastructure and transient devices are mixed in the same subnet with no firewall boundary.

## Target State (After Migration)

### Network Segmentation Plan

| Network | Subnet | Location |
|---|---|---|
| Enterprise / Management | 10.2.0.0/24 | Outside ASA |
| Telecom Infrastructure | 10.2.3.0/24 | Behind ASA |
| Phones / Transient Clients | 10.2.30.0/24 | UDMP VLAN 103 |

### Final Topology

```text
                Enterprise LAN / Core / UDMP
                         10.2.0.0/24
                                |
                        ASA 5525-X (outside)
                          10.2.0.130
                                |
                        ASA 5525-X (inside)
                           10.2.3.1
                                |
                    Telecom Infrastructure Switch
                                |
        ------------------------------------------------
        |        |        |        |        |          |
      Asterisk   PBX    Gateways  Routers  Channel Banks etc
```

Phones and transient devices remain outside the ASA:

- UDMP VLAN 103
- 10.2.30.0/24
- Phones / transient clients

## IP Addressing Plan

### ASA Interfaces

| Interface | Name | IP |
|---|---|---|
| Outside | enterprise | 10.2.0.130/24 |
| Inside | telecom | 10.2.3.1/24 |

### Routing

| Device | Route |
|---|---|
| ASA | Default route → 10.2.0.1 |
| UDMP / Core | Route 10.2.3.0/24 → 10.2.0.130 |

## Traffic Flow

- **Enterprise → Telecom Infrastructure**: Enterprise Host → Core → ASA Outside → ASA Inside → Telecom Device
- **Telecom Infrastructure → Internet**: Telecom Device → ASA → Core/UDMP → Internet
- **SIP Trunks**: SIP Provider ↔ Enterprise Network ↔ ASA ↔ PBX/Asterisk

## Firewall Policy Strategy (Initial vs Final)

### Initial Cutover Policy

During migration/testing:

- Allow all traffic between enterprise and telecom networks.
- Allow all outbound internet traffic from telecom network.

This is temporary.

### Final Intended Policy

Traffic will be restricted to:

- **From Enterprise → Telecom**
  - SSH
  - HTTPS
  - SNMP
  - Syslog
  - NTP
  - TACACS/RADIUS
  - Management ports only
- **From Telecom → Enterprise**
  - DNS
  - NTP
  - Syslog
  - Monitoring
  - Management services
  - SIP signaling
  - RTP media
  - Required application traffic only
- **From Telecom → Internet**
  - SIP provider IPs
  - RTP media ranges
  - DNS
  - NTP
  - HTTPS
  - Package updates
  - Monitoring endpoints

## SIP / RTP Considerations

Expected firewall allowances later:

| Protocol | Ports |
|---|---|
| SIP | 5060 / 5061 |
| RTP | Provider defined (often 10000–20000) |
| DNS | 53 |
| NTP | 123 |
| HTTPS | 443 |
| SSH | 22 |

Provider IP ranges should be explicitly allowed where possible.

## Migration Steps

### Phase 1 – Network Preparation

- Change UDMP VLAN 103 from `10.2.3.0/24` to `10.2.30.0/24`.
- Phones and transient devices obtain new DHCP addresses.
- Verify phones continue operating normally.

### Phase 2 – ASA Deployment

- Install ASA 5525-X in data center.
- Configure interfaces:
  - Outside → 10.2.0.130
  - Inside → 10.2.3.1
- Configure default route to 10.2.0.1.
- Add upstream route: `10.2.3.0/24 → 10.2.0.130`.
- Enable temporary allow-all policy.

### Phase 3 – Infrastructure Migration

- Physically move telecom infrastructure connections behind ASA.
- Verify devices can:
  - Reach enterprise network.
  - Reach internet.
  - Register SIP trunks.
  - Pass RTP audio.
- Monitor logs.

### Phase 4 – Security Hardening

- Replace allow-all rules with explicit ACLs.
- Restrict management access to admin hosts.
- Restrict SIP/RTP to provider networks.
- Enable logging and monitoring.
- Implement VPN / zero-trust management.

## Port-Channel / Uplink Design

ASA will connect to core switches using an EtherChannel for redundancy.

Recommended design:

| ASA | Core Switch |
|---|---|
| Gi0/0 | Port-channel |
| Gi0/1 | Port-channel |

- Port-channel carries enterprise VLAN / management subnet.
- Inside interface connects to dedicated telecom infrastructure switch or VLAN.

## Security Boundary Summary

| Network | Behind Firewall |
|---|---|
| Telecom infrastructure | Yes |
| Switch management | Yes |
| PBX / Asterisk | Yes |
| Voice routers | Yes |
| Channel banks | Yes |
| IP phones | No |
| User devices | No |

This ensures only critical infrastructure is protected while minimizing migration complexity.

## Future Enhancements

Planned future improvements:

- Site-to-site VPN.
- Remote admin VPN (zero trust management).
- Management-only interface.
- Logging to syslog server.
- SNMP monitoring.
- ACL hardening.
- Separate SBC / SIP DMZ network.
- High availability ASA pair.
- Dedicated transit subnet instead of management VLAN.
- Network object groups for SIP providers.
- QoS for RTP traffic.
- NetFlow / traffic monitoring.

## Summary

This migration will:

- Preserve existing infrastructure IP addresses.
- Move telecom infrastructure behind a firewall.
- Isolate critical systems from enterprise LAN.
- Maintain SIP trunk and internet connectivity.
- Allow future security hardening and VPN access.
- Provide a clean long-term network architecture for telecom systems.

This is the correct long-term architecture for protecting telecom infrastructure while maintaining operational flexibility.
