![Category](https://img.shields.io/badge/Category-Security%20Policy-red)
![Platform](https://img.shields.io/badge/Platform-pfSense%20Plus-2ea043)
![Status](https://img.shields.io/badge/Status-Active-brightgreen)

# Network Segmentation Policy

## Overview

This document defines the **network segmentation** and **inter-VLAN access policy** implemented in the Home Network Security Lab. The goal is to eliminate lateral movement paths, isolate untrusted devices, and simulate enterprise-grade network security controls.

All segmentation is enforced by the edge firewall. VLANs handle Layer 2 isolation; the firewall handles Layer 3 routing with a default-deny posture between all zones.

---

## Segmentation Goals

1. **Prevent Lateral Movement** — Stop attackers from pivoting from a compromised device to critical assets.
2. **Isolate Untrusted Networks** — Strictly contain IoT and Guest traffic to internet-only access.
3. **Protect the Management Plane** — Restrict infrastructure access to designated administrative endpoints only.
4. **Enable Lab Experimentation** — Allow malware detonation and attack simulation without exposing the home network.
5. **Maintain Usability** — Permit specific pinholes for shared services (printing, media discovery).

---

## VLAN Roles

| VLAN | Network | Purpose | Trust Level |
| :--- | :--- | :--- | :--- |
| **VLAN 10** | `192.168.10.0/24` | Trusted user devices — workstations, phones | **High** |
| **VLAN 20** | `192.168.20.0/24` | Network management and infrastructure | **Critical** |
| **VLAN 30** | `192.168.30.0/24` | Security lab and testing systems | **Low (Isolated)** |
| **VLAN 40** | `192.168.40.0/24` | Servers and internal services | **Medium** |
| **VLAN 50** | `192.168.50.0/24` | IoT and smart home devices | **Untrusted** |
| **VLAN 60** | `192.168.60.0/24` | Guest network access | **Untrusted** |

---

## Traffic Policy Matrix

Default behaviour for inter-VLAN traffic. Anything not explicitly listed is blocked.

| Source | Destination | Policy | Rationale | Implementation |
| :--- | :--- | :--- | :--- | :--- |
| **Main** | **Internet** | Allow | Standard access | Allow any (default) |
| **Main** | **IoT** | Allow | Users initiate control of smart devices | Allow source MAIN dest IoT |
| **Main** | **Servers** | Allow | Access to file shares and hosted services | Allow source MAIN dest SERVERS |
| **Main** | **Management** | Restricted | Admin access only | Allow specific IPs and ports |
| **IoT** | **Main** | Block | Prevent compromised devices reaching workstations | Block dest RFC 1918 |
| **IoT** | **Management** | Block | Protect infrastructure absolutely | Block dest RFC 1918 |
| **IoT** | **Internet** | Allow | Cloud connectivity for device function | Allow dest !RFC 1918 |
| **Lab** | **Main** | Block | Contain malware and attack tooling | Block dest RFC 1918 |
| **Guest** | **Internal** | Block | Internet-only — no internal visibility | Block dest RFC 1918 |

> **Rationale:** "Blocked" typically means an explicit deny to private IP ranges (`RFC 1918`) with an allow for internet traffic. This pattern ensures new internal VLANs added in the future remain isolated from untrusted zones by default.

---

## Management Network Protection

**VLAN 20 (Management)** is the highest-trust network in the lab. A compromise here means full control of all network infrastructure.

**Access Rules:**
- Only designated administrative endpoints (specific IPs within VLAN 10) can initiate connections to VLAN 20.
- All other sources are explicitly blocked from reaching management interfaces.

**Protected Services:**
- pfSense Web Configurator (HTTPS 443)
- Switch management interface (SSH / HTTP)
- Access point controllers
- Hypervisor management (if applicable)

> **Rationale:** Restricting management access to specific trusted IPs — not entire subnets — prevents internal lateral movement from escalating to infrastructure takeover.

---

## IoT Containment Strategy

IoT devices are treated as **untrusted by default** due to poor vendor security practices and frequent firmware vulnerabilities.

**Policies:**
1. **Internet-only** — IoT devices can reach cloud services but have no path to internal VLANs.
2. **Trusted-initiated only** — Trusted devices (VLAN 10) can initiate connections to IoT (e.g., casting), but IoT devices cannot initiate connections back to trusted zones (stateful firewall).
3. **AP client isolation** — Devices on the IoT AP cannot communicate directly with each other at Layer 2.

---

## Guest Network Isolation

| Policy | Implementation |
| :--- | :--- |
| Internet access | Unrestricted HTTP/HTTPS |
| Internal access | Hard block to `192.168.0.0/16`, `10.0.0.0/8`, `172.16.0.0/12` |
| DNS | Forced to Unbound — no external resolver access |
| Bandwidth | Capped to prevent abuse |

---

## DNS Enforcement Policy

All VLANs are subject to DNS enforcement via firewall NAT and block rules. No client can bypass the local Unbound resolver.

| Rule | Scope | Implementation |
| :--- | :--- | :--- |
| Force DNS to Unbound | All VLANs | NAT: outbound port 53 (UDP/TCP) → Unbound |
| Block DoT bypass | All VLANs | Block: outbound TCP port 853 |
| IoT DNS NAT override | VLAN 50 | NAT: all DNS → `127.0.0.1:53` regardless of destination |

See [`../dns/README.md`](../dns/README.md) for full DNS enforcement documentation.

---

## Remote Access Policy

Remote access is provided exclusively via Tailscale. No traditional VPN port is exposed.

| Rule | Implementation |
| :--- | :--- |
| WAN inbound allow | UDP 41641 — Tailscale direct connect only |
| All other inbound | Blocked — implicit default-deny WAN policy |
| Remote session scope | Inherits standard VLAN firewall rules via subnet routing |

See [`../vpn-access/README.md`](../vpn-access/README.md) for full remote access documentation.

---

## Service Exceptions (Pinholes)

Specific firewall rules allow cross-VLAN services while maintaining isolation:

| Service | Rule | Direction |
| :--- | :--- | :--- |
| **AirPrint** | Allow VLAN10 → Printer IP (ports 631, 9100) | One-way — printer cannot scan VLAN10 |
| **mDNS Reflection** | Avahi reflects multicast port 5353 across VLAN boundaries | Service discovery only — no subnet access |

---

## Security Benefits

- **Reduced blast radius** — Compromise in one VLAN is contained; no lateral path to other zones.
- **Defensive depth** — Multiple layers: VLAN isolation at Layer 2, firewall ACLs at Layer 3, DNS enforcement at Layer 7.
- **Full visibility** — Firewall logs every inter-VLAN traffic attempt. Every blocked probe is captured.
- **Compliance alignment** — Architecture mirrors PCI-DSS and ISO 27001 network segmentation requirements.

---

## Detailed Rule Documentation

| Interface | Document | Key Highlights |
| :--- | :--- | :--- |
| **WAN** | [`wan-rules.md`](./wan-rules.md) | Default-deny, bogon/RFC1918 blocks, QUIC suppression, Tailscale-only inbound |
| **LAN** | [`lan-rules.md`](./lan-rules.md) | Trust-tiered admin access, DNS enforcement, DoT blocking, management lockdown |
| **VLAN50 IoT** | [`vlan50-iot-rules.md`](./vlan50-iot-rules.md) | Full IoT isolation, DNS NAT override, noise tuning, scoped printer exception |

---

## Roadmap

- [x] SOC log tuning — `SOC_SILENCE_*` noise suppression framework implemented
- [x] DNS enforcement — DoT blocking + forced Unbound on all VLANs
- [x] IoT DNS NAT override — firmware-hardcoded DNS bypass prevented
- [x] Tailscale remote access — no-open-port VPN with subnet routing
- [ ] IDS/IPS — Suricata on VLAN40/50 gateways
- [ ] GeoIP blocking — deny high-risk countries at the WAN
- [ ] DNS sinkhole — pfBlockerNG for malicious domain filtering
