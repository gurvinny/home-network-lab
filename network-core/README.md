![Status](https://img.shields.io/badge/Status-Active-brightgreen)
![Network](https://img.shields.io/badge/Network-Core-blue)
![Platform](https://img.shields.io/badge/Platform-pfSense%20Plus-2ea043)

# Core Network Infrastructure

This directory documents the physical and logical network layout of the Home Network Security Lab — switch configuration, VLAN assignments, and routing topology.

---

## Network Architecture Overview

The core network uses a **router-on-a-stick** topology: a single 10Gb SFP+ trunk carries all VLAN-tagged traffic between the managed switch and the edge firewall. The firewall performs all Layer 3 inter-VLAN routing and stateful inspection. The switch handles Layer 2 distribution and 802.1Q VLAN tagging.

> **Rationale:** Centralising inter-VLAN routing at the edge firewall ensures all traffic crossing security zone boundaries is subject to stateful inspection and ACLs. This prevents any device from communicating across VLAN boundaries without passing through explicit firewall policy.

---

## Physical Hardware

| Component | Model | Role |
| :--- | :--- | :--- |
| **Edge Gateway** | OEM Server Appliance (Intel Atom C3758) | pfSense Plus — edge security, DHCP, Unbound DNS, inter-VLAN routing |
| **Core Switch** | MokerLink 10G0800GTM | 10GbE managed switching — Layer 2 distribution and 802.1Q VLAN tagging |
| **SFP+ Uplinks** | TP-Link TL-SM5310-T | 10GBase-T RJ45 SFP+ — copper uplinks between firewall, switch, and high-speed servers |

### Firewall Platform

The firewall appliance runs **pfSense Plus** (upgraded from Community Edition). Key capabilities unlocked by Plus:

- **QAT (Intel QuickAssist Technology)** — hardware acceleration for AES and RSA cryptographic operations. Offloads VPN and TLS handshake processing from CPU cores, maintaining throughput under Tailscale load.
- **ZFS Filesystem** — replaces UFS for the pfSense installation. Provides checksumming, silent corruption detection, and snapshot support for configuration backups.

---

## Logical VLAN Structure

| VLAN ID | Subnet | Name | Security Profile |
| :--- | :--- | :--- | :--- |
| **10** | `192.168.10.0/24` | Main | **Trusted** — general-use workstations and personal devices |
| **20** | `192.168.20.0/24` | Management | **Restricted** — infrastructure interfaces only |
| **30** | `192.168.30.0/24` | Lab | **Isolated** — malware testing and attack simulation |
| **40** | `192.168.40.0/24` | Servers | **Controlled** — NAS and hosted services |
| **50** | `192.168.50.0/24` | IoT | **Contained** — smart home devices and appliances |
| **60** | `192.168.60.0/24` | Guest | **Internet-only** — visiting device access |

---

## Contents

| Configuration | Directory | Description |
| :--- | :--- | :--- |
| **Switch Configuration** | [`/switch-config`](./switch-config/) | Port mappings, VLAN tagging (trunk/access), and management settings for the managed switch. |

---

## Roadmap

- [ ] **Quality of Service (QoS)** — Traffic shaping to prioritise latency-sensitive traffic over bulk downloads.
- [ ] **Link Aggregation (LACP/LAG)** — Port bonding between the firewall and core switch for increased bandwidth and link redundancy.
- [ ] **High Availability (HA)** — CARP setup with a secondary firewall appliance for seamless failover.
- [ ] **Advanced Routing** — Explore OSPF internally for redundant gateway and downstream lab router support.
