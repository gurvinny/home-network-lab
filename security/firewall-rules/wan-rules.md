![Category](https://img.shields.io/badge/Category-Firewall_Rules-blue?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-pfSense_Plus-2ea043?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Active-brightgreen?style=for-the-badge)

# WAN Firewall Rules

## Overview

This document details the firewall rules applied to the WAN interface (`ix1`), evaluated top-to-bottom. The WAN policy is **default-deny** — the only allowed inbound traffic is the VPN listener. All other inbound connections are blocked before reaching the firewall or any internal service.

---

## Rule Set

| # | Action | Protocol | Source | Destination | Port | Description |
| :- | :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | Block | Any | RFC 1918 networks | Any | Any | Block spoofed private-range sources |
| 2 | Block | Any | Reserved / unassigned IANA | Any | Any | Block bogon networks |
| 3 | Block | IPv6 ICMP | Any | `ff02::1` | Any | Suppress IPv6 WAN multicast noise |
| 4 | Block | IPv4 UDP | Any | WAN address | 443 | `SOC_SILENCE_QUIC_NOISE` — suppress QUIC log pollution |
| 5 | Block | IPv4 Any | `192.168.1.0/24` | Any | Any | Suppress modem management subnet leak |
| 6 | Pass | IPv4 UDP | Any | WAN address | VPN listener | Allow WireGuard tunnel establishment |
| 7 | Block | IPv4 TCP/UDP | Any | Any | Any | Implicit default deny — everything else |

Two game-server port forwards previously existed for direct external play. Both are **disabled**; the workload is now reached through the VPN instead of an open port.

---

## Key Takeaways

**The VPN listener is the only inbound allow.** No SSH, no web UI, no management interface, and no game-server port is reachable from the internet. The single permitted endpoint is the WireGuard listener, which is silent to unauthenticated traffic — a packet without a valid key gets no response at all, so the port does not answer scanners the way a TCP service would. A Tailscale mesh runs alongside it and needs no inbound rule; see [`../vpn-access/README.md`](../vpn-access/README.md) for both paths and why the lab runs each.

**Disabled forwards stay disabled and documented.** The game-server forwards are retained in the configuration in a disabled state rather than deleted, so the decision remains visible during review instead of silently disappearing from the ruleset.

**RFC 1918 and bogon blocks** prevent spoofed private-range sources from being processed by any firewall rule lower in the stack. This is a foundational antispoofing control.

**QUIC noise suppression (`SOC_SILENCE_QUIC_NOISE`).** Modern browsers and CDNs generate constant UDP 443 (QUIC/HTTP3) traffic. Blocking this explicitly at the top of the WAN ruleset eliminates a major source of log pollution, keeping the analyst's focus on actual reconnaissance and attack traffic.

**Modem management subnet leak.** The upstream ISP modem uses `192.168.1.0/24` internally. Without this rule, management traffic from the modem can bleed into pfSense logs and occasionally the routing table — an often-overlooked lateral path that is suppressed here.

> **Rationale:** Explicitly blocking and silencing expected background internet noise at the top of the WAN ruleset ensures the default-deny log captures only targeted reconnaissance and actual attack attempts — not CDN traffic or modem management chatter.
