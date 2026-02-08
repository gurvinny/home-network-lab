![Category](https://img.shields.io/badge/Category-Security%20Policy-red)
![Enforcement](https://img.shields.io/badge/Enforcement-pfSense%20Firewall-orange)
![Status](https://img.shields.io/badge/Status-Active-brightgreen)

# 🛡️ Network Segmentation Policy

## 📌 Overview

This document defines the **network segmentation** and **inter-VLAN access policy** implemented in the home lab environment. The goal is to reduce lateral movement risk, isolate vulnerable devices, and simulate enterprise network segmentation practices.

The network is segmented using **VLANs** with **firewall-enforced routing** and access control implemented on pfSense.

---

## 🎯 Segmentation Goals

1.  **Prevent Lateral Movement:** Stop attackers from pivoting from a compromised device (e.g., Smart Fridge) to critical assets (e.g., NAS).
2.  **Isolate Untrusted Networks:** Strictly contain IoT and Guest traffic.
3.  **Protect Management Plane:** Ensure infrastructure interfaces are only accessible from secure admins.
4.  **Enable Lab Experimentation:** Allow malware detonation or attack simulation without risking the home network.
5.  **Maintain Usability:** Allow specific "pinholes" for shared services like printing and casting.

---

## 🌐 VLAN Roles

| VLAN   | Network            | Purpose                               | Trust Level |
| :----- | :----------------- | :------------------------------------ | :---------- |
| **VLAN10** | `192.168.10.0/24`  | Trusted user devices (PC, Phone)      | **High**    |
| **VLAN20** | `192.168.20.0/24`  | Network management & infrastructure   | **Critical**|
| **VLAN30** | `192.168.30.0/24`  | Security lab & testing systems        | **Low**     |
| **VLAN40** | `192.168.40.0/24`  | Servers and internal services         | **Medium**  |
| **VLAN50** | `192.168.50.0/24`  | IoT and smart home devices            | **Untrusted**|
| **VLAN60** | `192.168.60.0/24`  | Guest network access                  | **Untrusted**|

---

## 🚦 Traffic Policy Matrix

This matrix defines the *default* behavior for inter-VLAN traffic.

| Source      | Destination | Policy     | Rationale | Implementation |
| :---------- | :---------- | :--------- | :-------- | :------------- |
| **MAIN**    | **Internet**| ✅ Allowed | Standard access. | Allow Any (Default) |
| **MAIN**    | **IoT**     | ✅ Allowed | Users control smart devices. | Allow Source MAIN Dest IoT |
| **MAIN**    | **SERVERS** | ✅ Allowed | Access to file shares/Plex. | Allow Source MAIN Dest SERVERS |
| **MAIN**    | **MGMT**    | ⚠️ Restricted| Admin access only. | Allow specific IPs/Ports only |
| **IoT**     | **MAIN**    | ❌ Blocked | Prevent hacking of PCs. | Block Dest RFC1918 |
| **IoT**     | **MGMT**    | ❌ Blocked | Protect infrastructure. | Block Dest RFC1918 |
| **IoT**     | **Internet**| ✅ Allowed | Cloud connectivity. | Allow Dest !RFC1918 |
| **LAB**     | **MAIN**    | ❌ Blocked | Contain malware. | Block Dest RFC1918 |
| **GUEST**   | **Internal**| ❌ Blocked | Privacy & Security. | Block Dest RFC1918 |

> **Note:** "Blocked" usually means explicitly blocking traffic to private IP ranges (RFC1918) while allowing everything else (Internet).

---

## 🔐 Management Network Protection

The **Management VLAN (20)** is the "Crown Jewels" of the network.

**Access Rules:**
-   **Only** authorized admin workstations (e.g., `192.168.10.10`) can initiate connections to VLAN 20.
-   **Services Protected:**
    -   pfSense Web Configurator (HTTPS 443)
    -   Switch Management (SSH/HTTP)
    -   AP Controllers
    -   ESXi/Proxmox Management

> **Why?** If an attacker gains access to this VLAN, they own the entire network infrastructure.

---

## 🤖 IoT Containment Strategy

IoT devices are notoriously insecure. They are treated as **hostile** by default.

**Policies:**
1.  **Internet Only:** They can talk to the cloud (AWS/Google) but not to local servers.
2.  **Inbound Initiated:** Trusted devices (VLAN 10) can talk *to* them (e.g., to cast a video), but they cannot reply unless it's part of that established connection (Stateful Firewalling).
3.  **Isolation:** Client Isolation is enabled on the AP to prevent IoT devices from attacking *each other*.

---

## 👥 Guest Network Isolation

**Guest Policies:**
-   **Internet Access:** Unrestricted HTTP/HTTPS.
-   **Bandwidth Limit:** Capped at 50/10 Mbps.
-   **Internal Access:** Hard block to `192.168.0.0/16`, `10.0.0.0/8`, `172.16.0.0/12`.
-   **DNS:** Forced to use Public DNS (e.g., 1.1.1.1) to prevent internal hostname resolution.

---

## 🔓 Service Exceptions (Pinholes)

To maintain usability, specific exceptions are made to the "Deny All" rules.

1.  **Printing:**
    -   **Rule:** Allow `VLAN10` to `Printer_IP` on Ports `9100/515`.
    -   **Direction:** One-way. Printer cannot scan VLAN10.

2.  **mDNS Reflection (Avahi):**
    -   Allows "Service Discovery" packets (Multicast 5353) to cross VLAN boundaries.
    -   Enables your phone (VLAN 10) to "see" the Chromecast (VLAN 50).

---

## 🛡️ Security Benefits

This segmentation model provides:

-   **Reduced Blast Radius:** A compromise in one VLAN is contained.
-   **Defensive Depth:** Multiple layers of controls (Firewall + VLANs).
-   **Visibility:** Firewall logs show exactly which device is trying to scan the network.
-   **Compliance:** Mimics PCI-DSS / ISO27001 segmentation standards.

---

## 🚀 Future Improvements

-   [ ] **IDS/IPS:** Deploy Suricata on VLAN 40/50 gateways.
-   [ ] **GeoIP Blocking:** Block high-risk countries on the WAN.
-   [ ] **Time-Based Rules:** Disable Kids' IoT access at night.
-   [ ] **DNS Sinkhole:** Use pfBlockerNG to stop ads/malware domains.

---

## 📝 Conclusion

Segmentation is a foundational security control. This lab demonstrates practical implementation of **enterprise-style network isolation** and **firewall policy enforcement** within a home environment, balancing strict security with daily usability.
