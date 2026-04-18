![Category](https://img.shields.io/badge/Category-Infrastructure-blue?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Active-brightgreen?style=for-the-badge)
![Policy](https://img.shields.io/badge/Security-Zero_Trust-red?style=for-the-badge)

# 🖥️ Proxmox Virtual Infrastructure

This document details the hardware and virtual networking configuration for the primary virtualization host (Lenovo M70q), specifically focusing on segmentation, VLAN tagging, and the integration of core services like Wazuh and Authentik.

---

## 💻 Hardware Specifications

The virtualization host is a micro form-factor system optimized for high core count and memory capacity to support multiple isolated virtual environments.

* **Model:** Lenovo M70q
* **CPU:** Intel Core i7-12th Gen T-Series
* **Memory:** 64GB DDR4 (3200MHz)
* **Hypervisor:** Proxmox VE

> **Security Rationale:** Utilizing an energy-efficient micro node allows for continuous operation with reduced thermal footprint while providing sufficient computational resources to segment services securely via virtualization rather than relying on containerized isolation on a single host.

---

## 🔌 Physical & Virtual Networking

The Proxmox host is connected to **Port 2** on the MokerLink switch. To support both secure management access and segmented virtual machines, this port is configured as a trunk.

### Switch Port 2 Configuration
* **Native (Untagged):** VLAN 20 (Mgmt) - `192.168.20.0/24`
* **Tagged:** VLAN 40 (Servers) - `192.168.40.0/24`
* **Tagged:** VLAN 70 (Game Servers) - `192.168.70.0/24`

> **Security Rationale:** Running the management interface untagged on VLAN 20 ensures that the Proxmox host is isolated strictly to the Management VLAN. Tagging the guest network VLANs (40 and 70) ensures that VMs are segmented at the hypervisor bridge level, preventing inter-VM communication without passing through the pfSense firewall's inspection engine.

### VM Placement & Segmentation

| Virtual Machine | Primary Function | Assigned VLAN | Subnet |
| :--- | :--- | :--- | :--- |
| **Proxmox Host** | Hypervisor / Management | 20 (Mgmt) | `192.168.20.0/24` |
| **Wazuh Manager** | SIEM / Log Aggregation | 20 (Mgmt) | `192.168.20.0/24` |
| **Authentik** | Identity Provider (IdP) | 20 (Mgmt) | `192.168.20.0/24` |
| **Dev Server** | Local App Testing | 70 (Game Servers) | `192.168.70.0/24` |
| **Game Server** | Minecraft Server | 70 (Game Servers) | `192.168.70.0/24` |

---

## 🗺️ Topology Diagram

```mermaid
graph TD
    %% Styling for dark hacker theme
    classDef firewall fill:#2b2b2b,stroke:#00a8ff,stroke-width:2px,color:#ffffff
    classDef switch fill:#2b2b2b,stroke:#00a8ff,stroke-width:2px,color:#ffffff
    classDef proxmox fill:#1a1a1a,stroke:#00a8ff,stroke-width:2px,color:#ffffff
    classDef vmMgmt fill:#2b2b2b,stroke:#ff3333,stroke-width:2px,color:#ffffff
    classDef vmGame fill:#2b2b2b,stroke:#00cc66,stroke-width:2px,color:#ffffff

    FW["pfSense Firewall"]:::firewall -->|"10G SFP+"| SW["MokerLink Switch"]:::switch

    subgraph "Lenovo M70q Host"
        SW -->|"Port 2 Trunk"| PVE["Proxmox VE"]:::proxmox

        PVE -->|"Untagged (VLAN 20)"| MGMT_BR["vmbr0 (Mgmt)"]:::proxmox
        PVE -->|"Tagged (VLAN 40/70)"| VLAN_BR["vmbr0.X (Trunk)"]:::proxmox

        subgraph "VLAN 20 (Mgmt)"
            MGMT_BR --> WAZUH["Wazuh Manager"]:::vmMgmt
            MGMT_BR --> AUTH["Authentik IdP"]:::vmMgmt
        end

        subgraph "VLAN 70 (Game Servers)"
            VLAN_BR --> GAME["Minecraft Server"]:::vmGame
            VLAN_BR --> DEV["Dev Server"]:::vmGame
        end
    end
```
