![Category](https://img.shields.io/badge/Category-Firewall%20Rules-blue)
![Enforcement](https://img.shields.io/badge/Enforcement-pfSense%20Firewall-orange)
![Status](https://img.shields.io/badge/Status-Active-brightgreen)

# 💻 LAN Firewall Rules

## 📌 Overview

This document details the firewall rules applied to the LAN interface, evaluated from top to bottom.

---

## 🛡️ Rule Set

| # | Action | Protocol | Source | Destination | Port | Description |
|---|--------|----------|--------|-------------|------|-------------|
| 1 | Pass | IPv4 TCP/UDP | Grv_Trusted_Devices | LAN address | * | Admin Access: Only Grv Trusted Devices can see the login page |
| 2 | Block | IPv4 TCP | * | ! LAN address | TCP_GHOST_PORTS | SOC_SILENCE_TCP_GHOSTS |
| 3 | Block | IPv6 * | * | * | * | SOC_SILENCE_LAN_IPV6_LEAK |
| 4 | Block | IPv4 UDP | Deco_Nodes | * | 20002 | Silently block Deco discovery broadcasts |
| 5 | Block | IPv4 TCP | * | LAN address | 853 (DNS over TLS) | SOC_SILENCE_DOT_ATTEMPTS |
| 6 | Pass | IPv6 UDP | * | * | 5353 | SOC_SILENCE_MDNS_IPV6 |
| 7 | Pass | IPv4 ICMP | LAN subnets | LAN address | * | SOC_ALLOW_ICMP_TO_GATEWAY |
| 8 | Pass | IPv4 TCP/UDP | LAN subnets | LAN address | 53 (DNS) | Allow DNS to Firewall |
| 9 | Block | IPv4 TCP | LAN subnets | This Firewall (self) | Management_Ports | Block Others: Everyone else gets a "Connection Refused." |
| 10 | Pass | IPv4 TCP/UDP | LAN subnets | * | Common_Discovery_Ports | Allow App Discovery (Spotify, NetBIOS, mDNS) |
| 11 | Pass | IPv4 * | LAN subnets | Internal_Resources | * | Allow MAIN to all Internal VLANs |
| 12 | Pass | IPv4 * | LAN subnets | ! Internal_Networks | * | Allow LAN to Internet ONLY |

---

## 💡 Key Takeaways

- **DNS Enforcement:** Only port 53 to the firewall is allowed and DoT (853) is blocked, meaning devices cannot bypass the local DNS resolver.
- **Management Lockdown:** Uses a named alias group (`Grv_Trusted_Devices`), not subnet-based trust, enabling granular per-device administrative control.
- **Rule Ordering Matters:** pfSense evaluates top-down, first-match-wins. The administrative allow rule must come before the management block rule.
- **The `! Internal_Networks` Pattern:** The final allow rule permits internet access but strictly blocks any remaining internal lateral movement.

> **Security Rationale:** Restricting management interfaces to specific trusted devices reduces the internal attack surface. Forcing DNS to the local firewall enables network-wide DNS filtering and prevents data exfiltration or evasion via unauthorized DoT servers.
