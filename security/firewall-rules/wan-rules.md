![Category](https://img.shields.io/badge/Category-Firewall%20Rules-blue)
![Enforcement](https://img.shields.io/badge/Enforcement-pfSense%20Firewall-orange)
![Status](https://img.shields.io/badge/Status-Active-brightgreen)

# 🌐 WAN Firewall Rules

## 📌 Overview

This document details the firewall rules applied to the WAN interface (`ix1`), evaluated from top to bottom.

---

## 🛡️ Rule Set

| # | Action | Protocol | Source | Destination | Port | Description |
|---|--------|----------|--------|-------------|------|-------------|
| 1 | Block | * | RFC 1918 networks | * | * | Block private networks |
| 2 | Block | * | Reserved/Not assigned by IANA | * | * | Block bogon networks |
| 3 | Block | IPv6 ICMP any | * | ff02::1 | * | Silently block IPv6 WAN Multicast Noise |
| 4 | Block | IPv4 UDP | * | WAN address | 443 (HTTPS) | SOC_SILENCE_QUIC_NOISE |
| 5 | Block | IPv4 * | 192.168.1.0/24 | * | * | Silence Modem Management Leak |
| 6 | Pass | IPv4 UDP | * | WAN address | 41641 | Allow Tailscale Direct Connect (UDP 41641) |

---

## 💡 Key Takeaways

- **Inbound Allow:** The only inbound allow is Tailscale (WireGuard-based mesh VPN), avoiding exposed SSH/VPN ports.
- **QUIC Noise:** QUIC noise (UDP 443) is a major log polluter from CDNs and browsers. Blocking it explicitly suppresses logs for this standard noise.
- **Modem Management Leak:** The modem management leak rule blocks the upstream modem's `192.168.1.0/24` subnet from reaching pfSense — an often-overlooked lateral path.
- **SOC Silence Prefix:** All rules prefixed `SOC_SILENCE_*` exist specifically to reduce SIEM/log noise.

> **Security Rationale:** Explicitly blocking and silencing expected background internet noise at the very top of the ruleset ensures that the final default-deny log captures only actual targeted reconnaissance and attacks.
