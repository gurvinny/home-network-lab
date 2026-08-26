![Category](https://img.shields.io/badge/Category-Security_Policy-red?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-pfSense_Plus-2ea043?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Active-brightgreen?style=for-the-badge)

# Network Segmentation Policy

## Overview

This document defines the **network segmentation** and **inter-VLAN access policy** implemented in the Home Network Security Lab. The goal is to eliminate lateral movement paths, isolate untrusted devices, and simulate enterprise-grade network security controls.

All segmentation is enforced by the edge firewall. VLANs handle Layer 2 isolation; the firewall handles Layer 3 routing with a default-deny posture between all zones.

---

## Segmentation Goals

1. **Prevent Lateral Movement** — Stop attackers from pivoting from a compromised device to critical assets.
2. **Isolate Untrusted Networks** — Strictly contain IoT and game-server traffic to internet-only access.
3. **Protect the Management Plane** — Restrict infrastructure access to designated administrative endpoints only.
4. **Contain Internet-Facing Workloads** — Keep externally reachable services in a segment with no path back into the estate.
5. **Maintain Usability** — Permit specific pinholes for shared services (printing, media discovery).

---

## VLAN Roles

| VLAN | Network | Purpose | Trust Level |
| :--- | :--- | :--- | :--- |
| **VLAN 10** | `192.168.10.0/24` | Trusted user devices — workstations, phones | **High** |
| **VLAN 20** | `192.168.20.0/24` | Network management, hypervisor, SIEM, identity provider | **Critical** |
| **VLAN 40** | `192.168.40.0/24` | Servers, self-hosted applications, development hosting | **Medium** |
| **VLAN 50** | `192.168.50.0/24` | IoT and smart home devices | **Untrusted** |
| **VLAN 70** | `192.168.70.0/24` | Game servers — internet-facing workloads | **Isolated** |

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
| **Servers** | **Management** | Block | Application compromise cannot pivot to the hypervisor or SIEM | Block dest `Internal_Networks` |
| **Servers** | **Game Servers** | Restricted | Control plane reaches its isolated node on named ports only | Allow specific host and ports |
| **Game Servers** | **Internal** | Block | Internet-facing workload has no path into the estate | Block dest `Internal_Networks` |
| **Game Servers** | **Management** | Restricted | SIEM agent traffic only | Allow dest SIEM agent ports |
| **Game Servers** | **Internet** | Allow | Updates and player traffic | Allow dest `! Internal_Networks` |

> **Rationale:** "Blocked" typically means an explicit deny to private IP ranges (`RFC 1918`) with an allow for internet traffic. This pattern ensures new internal VLANs added in the future remain isolated from untrusted zones by default.

---

## Standard Rule Order

Every segment's ruleset is built in the same six-stage order. Consistency is the point: an analyst reading any interface's rules knows where to look, and a rule in the wrong stage is visible as an anomaly rather than hiding in a long flat list.

| Stage | Purpose |
| :--- | :--- |
| **1. Explicit allows** | The few flows this segment genuinely needs, named and justified |
| **2. Forced DNS / NTP** | Redirect time and name resolution to the firewall |
| **3. `SOC_SILENCE_*`** | Silently drop expected-but-unwanted noise before it reaches the logs |
| **4. Block management ports** | Deny the segment access to the firewall's own administrative interfaces |
| **5. Block inter-VLAN** | Deny everything to `Internal_Networks` |
| **6. Allow egress** | Permit `! Internal_Networks` — internet only |

The net effect is **default-deny between segments, permit to internet**, with the firewall as the only shared service. Noise suppression sits above the deny rules deliberately: dropping known-benign chatter before the catch-all block keeps the default-deny log meaningful.

---

## Management Network Protection

**VLAN 20 (Management)** is the highest-trust network in the lab. A compromise here means full control of all network infrastructure.

**Access Rules:**
- Only a named alias of administrative endpoints can initiate connections to VLAN 20. Membership is explicit and enumerable, not a subnet.
- All other sources are explicitly blocked from reaching management interfaces.

**Protected Services:**
- pfSense Web Configurator (HTTPS 443)
- Switch management interface (SSH / HTTP)
- Access point controllers
- Hypervisor management interface — protected additionally by OIDC single sign-on with passkey enforcement, see [`../iam/authentik-passkey.md`](../iam/authentik-passkey.md)

> **Rationale:** Restricting management access to specific trusted IPs — not entire subnets — prevents internal lateral movement from escalating to infrastructure takeover.

---

## IoT Containment Strategy

IoT devices are treated as **untrusted by default** due to poor vendor security practices and frequent firmware vulnerabilities.

**Policies:**
1. **Internet-only** — IoT devices can reach cloud services but have no path to internal VLANs.
2. **Trusted-initiated only** — Trusted devices (VLAN 10) can initiate connections to IoT (e.g., casting), but IoT devices cannot initiate connections back to trusted zones (stateful firewall).
3. **AP client isolation** — Devices on the IoT AP cannot communicate directly with each other at Layer 2.

---

## Game Server Isolation

VLAN 70 hosts the only workloads in the lab that are reachable by people outside it. It is treated as the segment most likely to be compromised.

| Policy | Implementation |
| :--- | :--- |
| Internet access | Outbound HTTP/HTTPS for updates, plus player traffic |
| Internal access | Hard block to `Internal_Networks` — no path into any other segment |
| Control plane | The panel on VLAN 40 initiates to the node; the node never initiates back except for its heartbeat |
| SIEM agent | A single explicit allow to the log collector's agent port on VLAN 20 |
| DNS | NAT-forced to Unbound — no external resolver access |

See [`game-server-ztna.md`](./game-server-ztna.md) for the full ruleset.

---

## DNS Enforcement Policy

DNS enforcement is applied where it is needed rather than uniformly. Untrusted segments are **compelled** to use the local resolver by NAT redirection; trusted segments are **permitted** to use it and blocked from encrypted alternatives.

| Rule | Scope | Implementation |
| :--- | :--- | :--- |
| Force DNS to Unbound | VLAN 50 (IoT), VLAN 70 (Game Servers) | NAT: all DNS → `127.0.0.1:53` regardless of destination |
| Permit DNS to Unbound | VLAN 10 (Main), VLAN 40 (Servers) | Allow: port 53 (UDP/TCP) to the segment gateway |
| Force NTP to the firewall | VLAN 10, 20, 50, 70 | NAT: outbound UDP 123 → `127.0.0.1` |
| Block DoT bypass | All VLANs | Block: outbound TCP port 853 |

> **Why forced only on untrusted segments:** NAT-redirecting DNS breaks any client that validates the resolver it reached, and it hides misconfiguration. IoT firmware routinely hardcodes external resolvers, so the redirect is the only way to control it — the trusted segments do not need the same coercion, and leaving them un-NATed means a trusted host attempting to bypass the resolver shows up as a block event instead of being silently rewritten.

> **Why client DoT is blocked:** the resolver itself forwards upstream over DNS-over-TLS. Allowing clients to run their own DoT would move resolution outside the firewall's visibility while gaining nothing — the encryption to the upstream provider already exists.

See [`../dns/README.md`](../dns/README.md) for full DNS enforcement documentation.

---

## Remote Access Policy

Remote access runs over two paths: WireGuard tunnels terminated on the firewall, and a Tailscale mesh overlay.

| Rule | Implementation |
| :--- | :--- |
| WAN inbound allow | WireGuard listeners only |
| All other inbound | Blocked — implicit default-deny WAN policy; legacy game-server forwards disabled |
| Mesh path | Requires no inbound rule — the firewall establishes the session outbound |
| Remote session scope | Inherits standard VLAN firewall rules via subnet routing — a tunnel grants a path onto the network, not permission to cross segments |

See [`../vpn-access/README.md`](../vpn-access/README.md) for full remote access documentation.

---

## Service Exceptions (Pinholes)

Specific firewall rules allow cross-VLAN services while maintaining isolation:

| Service | Rule | Direction |
| :--- | :--- | :--- |
| **AirPrint** | Allow VLAN10 → the single reserved printer address (ports 631, 9100) | One-way — printer cannot scan VLAN10 |
| **mDNS Reflection** | Avahi reflects multicast port 5353 across VLAN boundaries | Service discovery only — no subnet access |
| **Home Automation** | Allow VLAN40 controller → VLAN50 devices | One-way — IoT cannot initiate back |
| **Game Server Control** | Allow VLAN40 panel → VLAN70 node on application and file-transfer ports | Plus a single node → panel heartbeat |
| **SIEM Agent** | Allow monitored hosts → the log collector's agent port on VLAN 20 | Agent traffic only — no management access |
| **Administrative Alias** | Allow the named admin device alias → any VLAN | The one deliberate exception to default-deny |

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
| **WAN** | [`wan-rules.md`](./wan-rules.md) | Default-deny, bogon/RFC1918 blocks, QUIC suppression, VPN-only inbound |
| **LAN** | [`lan-rules.md`](./lan-rules.md) | Trust-tiered admin access, DNS enforcement, DoT blocking, management lockdown |
| **VLAN50 IoT** | [`vlan50-iot-rules.md`](./vlan50-iot-rules.md) | Full IoT isolation, DNS NAT override, noise tuning, scoped printer exception |
| **VLAN70 Game Servers** | [`game-server-ztna.md`](./game-server-ztna.md) | Zero-trust ruleset for the internet-facing node, split control plane, SIEM agent path |

---

## Roadmap

- [x] SOC log tuning — `SOC_SILENCE_*` noise suppression framework implemented
- [x] DNS enforcement — DoT blocking everywhere, forced Unbound on untrusted segments
- [x] IoT DNS NAT override — firmware-hardcoded DNS bypass prevented
- [x] Remote access — WireGuard tunnels plus a Tailscale mesh with subnet routing
- [x] Game server ZTNA — isolated VLAN 70 node with a segmented control plane
- [ ] IDS/IPS — Suricata on the VLAN40/50 gateways
- [ ] GeoIP blocking — deny high-risk countries at the WAN
- [ ] DNS sinkhole — pfBlockerNG for malicious domain filtering
