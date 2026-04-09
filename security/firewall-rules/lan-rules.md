![Category](https://img.shields.io/badge/Category-Firewall%20Rules-blue)
![Platform](https://img.shields.io/badge/Platform-pfSense%20Plus-2ea043)
![Status](https://img.shields.io/badge/Status-Active-brightgreen)

# LAN Firewall Rules

## Overview

This document details the firewall rules applied to the LAN interface (VLAN 10 — Main), evaluated top-to-bottom. Rules enforce DNS enforcement, management plane lockdown, and inter-VLAN access control.

---

## Rule Set

| # | Action | Protocol | Source | Destination | Port | Description |
| :- | :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | Pass | IPv4 TCP/UDP | `Grv_Trusted_Devices` | LAN address | Any | Admin access — only designated hosts can reach the management portal |
| 2 | Block | IPv4 TCP | Any | `! LAN address` | `TCP_GHOST_PORTS` | `SOC_SILENCE_TCP_GHOSTS` — suppress phantom port noise |
| 3 | Block | IPv6 Any | Any | Any | Any | `SOC_SILENCE_LAN_IPV6_LEAK` — drop IPv6 leakage |
| 4 | Block | IPv4 UDP | `Deco_Nodes` | Any | 20002 | Suppress mesh AP discovery broadcasts |
| 5 | Block | IPv4 TCP | Any | LAN address | 853 | `SOC_SILENCE_DOT_ATTEMPTS` — block DNS-over-TLS bypass |
| 6 | Pass | IPv6 UDP | Any | Any | 5353 | `SOC_SILENCE_MDNS_IPV6` — permit IPv6 mDNS for service discovery |
| 7 | Pass | IPv4 ICMP | LAN subnets | LAN address | Any | `SOC_ALLOW_ICMP_TO_GATEWAY` — permit gateway reachability checks |
| 8 | Pass | IPv4 TCP/UDP | LAN subnets | LAN address | 53 | Allow DNS queries to Unbound on the firewall |
| 9 | Block | IPv4 TCP | LAN subnets | This Firewall | `Management_Ports` | Block non-admin hosts from management interfaces |
| 10 | Pass | IPv4 TCP/UDP | LAN subnets | Any | `Common_Discovery_Ports` | Allow service discovery (Spotify, NetBIOS, mDNS) |
| 11 | Pass | IPv4 Any | LAN subnets | `Internal_Resources` | Any | Allow Main to all internal VLANs |
| 12 | Pass | IPv4 Any | LAN subnets | `! Internal_Networks` | Any | Allow LAN internet access — blocks remaining internal lateral movement |

---

## Key Takeaways

**DNS enforcement.** Only port 53 to the firewall is allowed. DNS-over-TLS (port 853) is explicitly blocked. This means clients cannot bypass the local Unbound resolver regardless of their configured DNS settings — whether hardcoded in an application or configured by a user.

**Management lockdown by named alias.** The `Grv_Trusted_Devices` alias controls administrative access using specific device IPs — not subnet-level trust. This means a device on VLAN 10 that is not in the alias group cannot reach the pfSense web UI even though it is on the "trusted" VLAN.

**Rule ordering is critical.** pfSense evaluates rules top-down, first-match-wins. The admin allow rule (Rule 1) must precede the management block rule (Rule 9) — otherwise the allow never fires.

**The `! Internal_Networks` pattern.** Rule 12 is the "internet-only escape valve" — it allows any traffic to non-RFC1918 destinations, blocking any remaining path to internal VLANs that was not explicitly permitted in earlier rules.

> **Rationale:** Restricting management access to named trusted devices reduces the internal attack surface. Forcing DNS to the local resolver prevents data exfiltration and evasion via external DoT servers. The combination of alias-based trust and rule ordering mimics enterprise ACL design patterns used in production SOC environments.
