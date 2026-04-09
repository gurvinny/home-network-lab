![Category](https://img.shields.io/badge/Category-Firewall%20Rules-blue)
![Platform](https://img.shields.io/badge/Platform-pfSense%20Plus-2ea043)
![Status](https://img.shields.io/badge/Status-Active-brightgreen)

# WAN Firewall Rules

## Overview

This document details the firewall rules applied to the WAN interface (`ix1`), evaluated top-to-bottom. The WAN policy is **default-deny** — the only allowed inbound traffic is Tailscale. All other inbound connections are blocked before reaching the firewall or any internal service.

---

## Rule Set

| # | Action | Protocol | Source | Destination | Port | Description |
| :- | :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | Block | Any | RFC 1918 networks | Any | Any | Block spoofed private-range sources |
| 2 | Block | Any | Reserved / unassigned IANA | Any | Any | Block bogon networks |
| 3 | Block | IPv6 ICMP | Any | `ff02::1` | Any | Suppress IPv6 WAN multicast noise |
| 4 | Block | IPv4 UDP | Any | WAN address | 443 | `SOC_SILENCE_QUIC_NOISE` — suppress QUIC log pollution |
| 5 | Block | IPv4 Any | `192.168.1.0/24` | Any | Any | Suppress modem management subnet leak |
| 6 | Pass | IPv4 UDP | Any | WAN address | 41641 | Allow Tailscale direct connect (WireGuard) |

---

## Key Takeaways

**Tailscale is the only inbound allow.** No SSH, no web UI, no traditional VPN port is exposed to the internet. The WAN is effectively invisible to all scanners except Tailscale's WireGuard handshake port (`UDP 41641`). Tailscale acts as both subnet router and exit node — see [`../vpn-access/README.md`](../vpn-access/README.md) for full configuration details.

**RFC 1918 and bogon blocks** prevent spoofed private-range sources from being processed by any firewall rule lower in the stack. This is a foundational antispoofing control.

**QUIC noise suppression (`SOC_SILENCE_QUIC_NOISE`).** Modern browsers and CDNs generate constant UDP 443 (QUIC/HTTP3) traffic. Blocking this explicitly at the top of the WAN ruleset eliminates a major source of log pollution, keeping the analyst's focus on actual reconnaissance and attack traffic.

**Modem management subnet leak.** The upstream ISP modem uses `192.168.1.0/24` internally. Without this rule, management traffic from the modem can bleed into pfSense logs and occasionally the routing table — an often-overlooked lateral path that is suppressed here.

> **Rationale:** Explicitly blocking and silencing expected background internet noise at the top of the WAN ruleset ensures the default-deny log captures only targeted reconnaissance and actual attack attempts — not CDN traffic or modem management chatter.
