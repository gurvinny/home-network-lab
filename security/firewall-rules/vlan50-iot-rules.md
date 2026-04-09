![Category](https://img.shields.io/badge/Category-Firewall%20Rules-blue)
![Enforcement](https://img.shields.io/badge/Enforcement-pfSense%20Firewall-orange)
![Status](https://img.shields.io/badge/Status-Active-brightgreen)

# 📡 VLAN50 IoT Firewall Rules

## 📌 Overview

This document details the firewall rules applied to the VLAN50_IOT interface, evaluated from top to bottom.

---

## 🛡️ Rule Set

| # | Action | Protocol | Source | Destination | Port | Description |
|---|--------|----------|--------|-------------|------|-------------|
| 1 | Block | IPv4 TCP | * | ! VLAN50_IOT address | TCP_GHOST_PORTS | SOC_SILENCE_TCP_GHOSTS |
| 2 | Block | IPv4+6 UDP | * | * | CHATTY_IOT_PORTS | SOC_SILENCE_IOT_NOISE |
| 3 | Block | IPv4 UDP | Smart_TVs | * | 15600 | SOC_SILENCE_SAMSUNG_DISCOVERY |
| 4 | Pass | IPv4 TCP/UDP | * | 127.0.0.1 | 53 (DNS) | NAT Force IOT DNS to Local Cache |
| 5 | Pass | IPv4 * | VLAN50_IOT subnets | VLAN50_IOT address | * | Allow IoT to Gateway |
| 6 | Pass | IPv4 UDP | VLAN50_IOT subnets | This Firewall (self) | 123 (NTP) | Allow NTP |
| 7 | Pass | IPv4 TCP/UDP | 192.168.50.10 | LAN subnets | 631-9100 | Allow Printer to LAN for AirPrint |
| 8 | Block | IPv4 TCP | VLAN50_IOT subnets | This Firewall (self) | Management_Ports | Block management access to pfSense |
| 9 | Block | IPv4 * | VLAN50_IOT subnets | Internal_Networks | * | Block IoT to internal networks |
| 10 | Pass | IPv4 * | VLAN50_IOT subnets | ! Internal_Networks | * | Allow IoT internet |

---

## 💡 Key Takeaways

- **DNS NAT to 127.0.0.1:** Forces all IoT DNS queries through the local cache regardless of hardcoded DNS settings in device firmware. This effectively prevents DNS-based Command and Control (C2) channels that bypass DHCP-assigned resolvers.
- **Samsung Discovery on Port 15600:** This rule was identified from real log data as a constant broadcast source and was subsequently suppressed to reduce noise. (See the Log Analysis section for details).
- **Printer Exception:** Scoped tightly to a single IP (`192.168.50.10`) and a specific port range (`631-9100`), ensuring the printer cannot broadly access the LAN.
- **Block-Internal-Then-Allow-Internet Pattern:** The default posture isolated IoT completely. Even if new internal VLANs are added later, IoT devices remain isolated by default and can only reach the Internet.
- **Device-Specific Noise Tuning:** The `CHATTY_IOT_PORTS` and `Smart_TVs` aliases demonstrate how specific device noise can be finely tuned to keep logs clean and relevant.

> **Security Rationale:** IoT devices are highly prone to compromise due to poor vendor security practices. Enforcing strict network isolation ensures that a compromised IoT device cannot act as a pivot point to attack other internal networks or critical infrastructure.
