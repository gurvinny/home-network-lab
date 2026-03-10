![Status](https://img.shields.io/badge/Status-Active-brightgreen)
![Network](https://img.shields.io/badge/Network-Core-blue)

# 🌐 Core Network Infrastructure

This directory documents the foundational physical and logical network layout of the Home Network Security Lab, detailing switch configurations, VLAN assignments, and routing topologies.

---

## 📌 Network Architecture Overview

The core network is designed for high-speed throughput (10GbE) combined with strict logical separation (VLAN tagging). The topology operates in a traditional "router-on-a-stick" methodology, with the pfSense firewall acting as the edge gateway and inter-VLAN router, while the MokerLink switch handles Layer 2 distribution.

> **Security Rationale:** Centralizing inter-VLAN routing at the pfSense firewall ensures all traffic passing between security boundaries is subject to stateful inspection and access control lists (ACLs). This prevents unauthorized communication between trusted zones and IoT/Guest environments.

---

## 🧰 Physical Hardware Summary

| Component | Model | Role |
| :--- | :--- | :--- |
| **Edge Gateway** | OEM Server Appliance (Atom C3758) | pfSense firewall; provides edge security, DHCP, DNS, and inter-VLAN routing. |
| **Core Switch** | MokerLink 10G0800GTM | 10GbE Managed Switch; handles Layer 2 traffic and 802.1Q VLAN tagging. |
| **Uplinks** | TP-Link TL-SM5310-T | 10GBase-T RJ45 SFP+ modules connecting the firewall to the switch and critical servers at 10Gbps. |

---

## 🌐 Logical VLAN Structure

The logical network relies on strict VLAN segmentation mapping specific subnets to device classifications.

| VLAN ID | Subnet | Name | Security Profile |
| :--- | :--- | :--- | :--- |
| **10** | `192.168.10.0/24` | Main | **Trusted** - General use devices (PCs, laptops). |
| **20** | `192.168.20.0/24` | Management | **Restricted** - Infrastructure access only. |
| **30** | `192.168.30.0/24` | Lab | **Isolated** - Malware testing and simulations. |
| **40** | `192.168.40.0/24` | Servers | **Controlled** - NAS and hosted services. |
| **50** | `192.168.50.0/24` | IoT | **Contained** - Smart home devices and appliances. |
| **60** | `192.168.60.0/24` | Guest | **Internet-only** - Visiting device access. |

---

## 📑 Table of Contents

| Configuration | Directory | Description |
| :--- | :--- | :--- |
| **Switch Configuration** | [`/switch-config`](./switch-config/) | Details port mappings, VLAN tagging (Trunk/Access), and management settings for the MokerLink 10G switch. |

---

## 🚀 Future Additions

The core network is planned for further enhancements to improve redundancy, traffic optimization, and routing capabilities.

-   [ ] **Quality of Service (QoS):** Implement traffic shaping on pfSense and the MokerLink switch to prioritize VoIP and gaming traffic over bulk downloads.
-   [ ] **Advanced Routing:** Explore deploying dynamic routing protocols (e.g., OSPF or BGP) internally to support redundant gateways or downstream lab routers.
-   [ ] **Link Aggregation (LACP/LAG):** Configure port bonding between the pfSense firewall and the core switch to increase inter-VLAN bandwidth and provide link redundancy.
-   [ ] **High Availability (HA):** Plan for a CARP setup with a secondary pfSense appliance to ensure seamless failover in the event of hardware failure.
