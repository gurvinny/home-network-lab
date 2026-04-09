![Category](https://img.shields.io/badge/Category-Firewall%20Rules-blue)
![Platform](https://img.shields.io/badge/Platform-pfSense%20Plus-2ea043)
![Status](https://img.shields.io/badge/Status-Active-brightgreen)

# VLAN50 IoT Firewall Rules

## Overview

This document details the firewall rules applied to the VLAN50 IoT interface, evaluated top-to-bottom. IoT devices are treated as **untrusted by default** — the ruleset enforces internet-only access, full internal network isolation, and DNS enforcement including NAT override for devices with firmware-hardcoded resolvers.

---

## Rule Set

| # | Action | Protocol | Source | Destination | Port | Description |
| :- | :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | Block | IPv4 TCP | Any | `! VLAN50_IOT address` | `TCP_GHOST_PORTS` | `SOC_SILENCE_TCP_GHOSTS` — suppress ghost port noise |
| 2 | Block | IPv4+6 UDP | Any | Any | `CHATTY_IOT_PORTS` | `SOC_SILENCE_IOT_NOISE` — suppress chatty IoT broadcasts |
| 3 | Block | IPv4 UDP | `Smart_TVs` | Any | 15600 | `SOC_SILENCE_SAMSUNG_DISCOVERY` — suppress Samsung discovery broadcast |
| 4 | Pass | IPv4 TCP/UDP | Any | `127.0.0.1` | 53 | NAT force IoT DNS to local Unbound cache |
| 5 | Pass | IPv4 Any | VLAN50 subnets | VLAN50 address | Any | Allow IoT to gateway |
| 6 | Pass | IPv4 UDP | VLAN50 subnets | This Firewall | 123 | Allow NTP — time synchronisation |
| 7 | Pass | IPv4 TCP/UDP | `192.168.50.10` | LAN subnets | 631–9100 | Allow printer to LAN for AirPrint (scoped to single IP) |
| 8 | Block | IPv4 TCP | VLAN50 subnets | This Firewall | `Management_Ports` | Block IoT devices from firewall management access |
| 9 | Block | IPv4 Any | VLAN50 subnets | `Internal_Networks` | Any | Block IoT to all internal VLANs |
| 10 | Pass | IPv4 Any | VLAN50 subnets | `! Internal_Networks` | Any | Allow IoT internet access only |

---

## Key Takeaways

**DNS NAT to `127.0.0.1`.** Rule 4 redirects all DNS queries to the local Unbound resolver via NAT, regardless of the destination IP the device uses. IoT device firmware frequently hardcodes resolvers like `8.8.8.8` or `1.1.1.1` at the application layer, bypassing DHCP-assigned DNS. This NAT override defeats that. It also prevents DNS-based Command and Control (C2) channels from reaching external resolvers.

**Samsung discovery suppression (port 15600).** This rule was created directly from the log analysis conducted in April 2026. Samsung TVs broadcast on UDP 15600 to the subnet broadcast address every ~5 seconds — this generated hundreds of log entries per hour with zero security value. The `SOC_SILENCE_SAMSUNG_DISCOVERY` rule suppresses this specific noise pattern identified from real traffic.

**Printer exception scoped to a single IP.** Rule 7 allows exactly one IP address (`192.168.50.10` — the printer) to reach LAN subnets on ports 631–9100 (IPP and JetDirect). This is the minimal access required for AirPrint to function. The printer cannot initiate any other connection to the LAN, and no other IoT device can reach the LAN on any port.

**Block-internal-then-allow-internet pattern.** Rule 9 blocks all IoT traffic to `Internal_Networks` (RFC 1918). Rule 10 then allows everything else — which is internet traffic. This pattern means new internal VLANs added in the future are automatically isolated from IoT devices without requiring additional rules.

**Device-specific noise tuning.** The `CHATTY_IOT_PORTS` and `Smart_TVs` named aliases demonstrate how observed per-device traffic patterns are encoded into reusable alias groups. This approach scales to many devices without adding individual rules for each one.

> **Rationale:** IoT devices are prime targets for compromise due to poor vendor security practices and infrequent firmware updates. Treating the entire VLAN as hostile by default ensures that a compromised IoT device — regardless of which device — has no path to reach workstations, servers, management interfaces, or the firewall itself.
