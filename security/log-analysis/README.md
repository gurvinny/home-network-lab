![Category](https://img.shields.io/badge/Category-Log%20Analysis-blue)
![Enforcement](https://img.shields.io/badge/Enforcement-pfSense%20Firewall-orange)
![Status](https://img.shields.io/badge/Status-Active-brightgreen)

# 📈 Firewall Log Analysis

## 📌 Overview

This document presents a real-world analysis of pfSense firewall logs collected within the Home Network Security Lab environment. It demonstrates how logs can be parsed, analyzed, and used to identify reconnaissance attempts, internal noise, and refine firewall rulesets.

---

## 📜 Log Format Explanation

pfSense outputs its firewall logs using the **RFC 5424 syslog format** via the `filterlog` facility. The payload of the log message is a comma-separated list of fields that detail the network traffic event.

A typical `filterlog` entry contains the following key fields (among others):
1.  **Rule Number:** The ID of the firewall rule that triggered the log.
2.  **Interface:** The interface where the traffic was seen (e.g., `ix1` for WAN, `ix0.50` for VLAN50).
3.  **Action:** Whether the traffic was `pass` or `block`.
4.  **Direction:** `in` or `out`.
5.  **IP Version:** `4` for IPv4, `6` for IPv6.
6.  **Protocol:** e.g., `tcp`, `udp`, `gre`, `icmp`.
7.  **Source IP:** The originating IP address.
8.  **Destination IP:** The target IP address.
9.  **Source Port:** The originating port (for TCP/UDP).
10. **Destination Port:** The target port (for TCP/UDP).
11. **TCP Flags:** If applicable (e.g., `S` for SYN).

---

## 🕵️‍♂️ Redacted Sample Log Data

> **Note:** The public WAN IP has been redacted and replaced with `[REDACTED_WAN_IP]` in all log samples below. Internal RFC 1918 addresses and reserved local domains (like `.home.arpa`) are displayed as they are inherently non-routable.

### Reconnaissance: Inbound Port Scan (TCP SYN to VNC port 5900)
```text
<134>1 2026-04-08T17:41:33-04:00 edgefw.core.home.arpa filterlog - - -
  1,6,,1000000103,ix1,match,block,in,4,0x0,,114,24337,0,none,6,tcp,48,
  165.245.162.235,[REDACTED_WAN_IP],5949,5900,0,S,1484833334,,65535,,mss;nop;nop;sackOK
```

### Reconnaissance: Telnet Scan (port 23) — duplicate SYN packets from same source
```text
<134>1 2026-04-08T17:42:54-04:00 edgefw.core.home.arpa filterlog - - -
  1,6,,1000000103,ix1,match,block,in,4,0x0,,40,10980,0,none,6,tcp,40,
  180.163.62.126,[REDACTED_WAN_IP],56339,23,0,S,1165667900,,65535,,

<134>1 2026-04-08T17:42:54-04:00 edgefw.core.home.arpa filterlog - - -
  1,6,,1000000103,ix1,match,block,in,4,0x0,,40,10980,0,none,6,tcp,40,
  180.163.62.126,[REDACTED_WAN_IP],56339,23,0,S,1165667900,,65535,,
```

### Reconnaissance: SNMP Enumeration Attempt (UDP port 161)
```text
<134>1 2026-04-08T17:42:34-04:00 edgefw.core.home.arpa filterlog - - -
  1,6,,1000000103,ix1,match,block,in,4,0x0,,53,9386,0,DF,17,udp,71,
  193.163.125.127,[REDACTED_WAN_IP],23052,161,51
```

### Suspicious: GRE Tunnel Probe (Unusual Protocol Scan)
```text
<134>1 2026-04-08T17:43:10-04:00 edgefw.core.home.arpa filterlog - - -
  1,6,,1000000103,ix1,match,block,in,4,0x0,,54,5245,0,DF,47,gre,564,
  84.54.149.97,[REDACTED_WAN_IP],datalength=544
```

### Reconnaissance: MongoDB Scan (Targeting Exposed Databases)
```text
<134>1 2026-04-08T23:21:16-04:00 edgefw.core.home.arpa filterlog - - -
  1,6,,1000000103,ix1,match,block,in,4,0x0,,239,54359,0,none,6,tcp,44,
  43.228.157.45,[REDACTED_WAN_IP],44878,27017,0,S,4194725836,,1025,,mss
```

### Internal Noise: Samsung TV Discovery Broadcast (UDP 15600)
*This is the exact traffic that prompted the creation of the `SOC_SILENCE_SAMSUNG_DISCOVERY` rule.*
```text
<134>1 2026-04-08T23:20:10-04:00 edgefw.core.home.arpa filterlog - - -
  1,176,,1770416543,ix0.50,match,block,in,4,0x0,,64,63329,0,DF,17,udp,63,
  192.168.50.101,192.168.50.255,34728,15600,43
```

### Normal Traffic: Internal DNS Pass
```text
<134>1 2026-04-08T17:41:16-04:00 edgefw.core.home.arpa filterlog - - -
  1,168,,1775611791,ix0.50,match,pass,in,4,0x0,,64,33907,0,DF,17,udp,76,
  192.168.50.32,127.0.0.1,56603,53,56
```

---

## 📊 Traffic Analysis Summary

Over a ~5.5-hour capture window (17:41–23:22 UTC-4), approximately 2,850 log entries were recorded. The following insights were derived from this dataset:

### Reconnaissance Patterns Observed
- All blocked TCP entries carry the `S` (SYN) flag, indicating connection initiation attempts rather than established traffic.
- Targeted ports frequently included: `23` (Telnet), `161` (SNMP), `5900` (VNC), `5985` (WinRM), `8443`, `8728` (MikroTik API), `27017` (MongoDB), and `7474` (Neo4j).
- These patterns represent automated internet-wide scanners and botnets relentlessly probing for exposed, vulnerable services.

### Repeat Offender Netblocks
- `85.217.149.x` — Appeared repeatedly hitting random high ports (likely a scanning cluster).
- `193.46.255.x`, `193.163.125.x` — Multiple hits on common service ports.
- `198.235.24.x`, `147.185.132.x`, `35.203.210.x` — Persistent repeat SYN scanners.
- *In a production SOC, these IPs would be enriched and cross-referenced with threat intelligence feeds such as AbuseIPDB, VirusTotal, or OTX AlienVault.*

### Interesting Targeted Ports Table

| Port | Service | Why Attackers Target It |
|------|---------|------------------------|
| 23 | Telnet | Unencrypted remote shell, often has default credentials on IoT devices. |
| 161 | SNMP | Network enumeration and configuration extraction via weak community strings. |
| 5900 | VNC | Remote desktop access, frequently left unpatched or poorly secured. |
| 5985 | WinRM | Windows remote management protocol, heavily used for lateral movement. |
| 8728 | MikroTik API | Targeting vulnerable router infrastructure for complete network takeover. |
| 27017 | MongoDB | Database exfiltration; frequently deployed with no authentication. |
| 7474 | Neo4j | Graph database platform, often exposed to the internet with default credentials. |

### Internal Noise Identified and Silenced
- **Samsung Smart TVs:** Broadcasting consistently on UDP/15600 to the VLAN50 broadcast address every ~5 seconds.
- **TP-Link Deco Mesh:** Nodes sending constant discovery beacons on UDP/20002.
- **DNS Evasion:** Devices attempting DNS-over-TLS (port 853) to bypass local DNS filters.
- **IPv6 Chatter:** Constant IPv6 multicast and mDNS activity.

---

## 🔇 SOC Noise Tuning

The `SOC_SILENCE_*` naming convention was deliberately chosen for specific firewall rules. This ensures that when an analyst is reading logs or when building SIEM detection rules, noise-suppression rules are immediately distinguishable from actual security policy enforcements.

The tuning process took approximately 2 days of observing raw log output. The goal was to identify high-volume, low-value traffic (like broadcast storms and predictable vendor discovery protocols) and build explicit suppression rules. This drastically reduces the overall log volume, keeping the analyst's focus on potentially actionable security events.

---

## 🛠️ SIEM Readiness Assessment

The current logging configuration is in an excellent state for SIEM ingestion:

- **Standardized Format:** Logs are already in RFC 5424 syslog format with ISO 8601 timestamps.
- **Compatibility:** They are easily parseable by major SIEM platforms like Splunk, Elastic/ELK, Microsoft Sentinel, and Wazuh using either native or lightweight pfSense parsers.
- **Future Improvements:** To further elevate the security posture, the logs would benefit from:
  - GeoIP enrichment to identify the source country of attacks.
  - Automated threat intel correlation.
  - Behavioral alert rules (e.g., ">20 unique destination ports targeted from a single IP within 5 minutes").
  - Pairing these edge firewall logs with internal DNS query logs, endpoint EDR logs, and authentication logs for a comprehensive operational view.
