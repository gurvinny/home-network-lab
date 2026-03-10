![Status](https://img.shields.io/badge/Status-Active-brightgreen)
![Security](https://img.shields.io/badge/Security-Zero%20Trust-blue)

# 🛡️ Security Infrastructure

This directory contains the security configurations, firewall rulesets, and traffic analysis tools that enforce isolation and threat detection within the Home Network Security Lab.

---

## 📌 Security Philosophy

The core of this lab's security model is based on **VLAN segmentation** and the **principle of least privilege**. Rather than relying on a flat network where a compromised device has unrestricted access to the entire environment, devices are isolated into specific zones (e.g., Trusted, Untrusted, Isolated) based on their function and risk level.

> **Security Rationale:** By strictly enforcing boundaries between VLANs, the blast radius of a potential breach is minimized. An attacker compromising an IoT device is contained within the Untrusted zone, preventing lateral movement to personal computers or infrastructure management interfaces.

---

## 📊 Security Zones Architecture

The following diagram illustrates the logical separation of the network into distinct security zones, enforced by the pfSense firewall.

```mermaid
flowchart TB
    %% Cyber Sec Grey/Blue Theme Styling
    classDef firewall fill:#1a1b26,stroke:#00d2ff,stroke-width:2px,color:#ffffff;
    classDef zone fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#ffffff,stroke-dasharray: 5 5;
    classDef trusted fill:#1e293b,stroke:#10b981,stroke-width:2px,color:#ffffff;
    classDef untrusted fill:#1e293b,stroke:#ef4444,stroke-width:2px,color:#ffffff;
    classDef isolated fill:#1e293b,stroke:#f59e0b,stroke-width:2px,color:#ffffff;

    FW["pfSense Edge Firewall"]:::firewall

    subgraph Security_Zones ["Defined Security Zones"]
        direction TB

        subgraph Trusted_Zone ["Trusted Zone (Allow Outbound)"]
            direction LR
            VLAN10["VLAN 10: Main"]:::trusted
            VLAN40["VLAN 40: Servers"]:::trusted
            VLAN20["VLAN 20: Management"]:::trusted
        end

        subgraph Untrusted_Zone ["Untrusted Zone (Strict Containment)"]
            direction LR
            VLAN50["VLAN 50: IoT"]:::untrusted
            VLAN60["VLAN 60: Guest"]:::untrusted
        end

        subgraph Isolated_Zone ["Isolated Zone (Lab / Detonation)"]
            direction LR
            VLAN30["VLAN 30: Lab"]:::isolated
        end
    end
    class Security_Zones zone;

    FW -->|"Stateful Rules"| Trusted_Zone
    FW -->|"Drop by Default"| Untrusted_Zone
    FW -->|"Air-Gapped Logic"| Isolated_Zone
```

---

## 📑 Table of Contents

| Component | Directory | Description |
| :--- | :--- | :--- |
| **Firewall Rules** | [`/firewall-rules`](./firewall-rules/) | Core pfSense rulesets enforcing segmentation, access control lists (ACLs), and inter-VLAN routing policies. |
| **Intrusion Detection/Prevention** | [`/ids-ips`](./ids-ips/) | Suricata/Snort configurations for threat detection, malicious traffic blocking, and alerts. |
| **Adblockers & DNS Filtering** | [`/adblockers`](./adblockers/) | pfBlockerNG and Pi-hole settings for network-wide ad blocking, malicious domain filtering, and DNS sinkholing. |

---

## 🔒 Access Control Policies

The firewall strictly enforces the following default traffic behaviors:

-   **Intra-VLAN Traffic:** Handled by the MokerLink switch (Layer 2).
-   **Inter-VLAN Traffic:** Routed through the pfSense firewall (Layer 3) and subject to explicit ALLOW rules. All other traffic is silently DROPPED.
-   **Management Plane:** Access to VLAN 20 is completely restricted from all other subnets. Only designated administrative endpoints can access management interfaces.
