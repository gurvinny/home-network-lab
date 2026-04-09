![Category](https://img.shields.io/badge/Category-Log%20Analysis%20Report-blue)
![Platform](https://img.shields.io/badge/Platform-pfSense%20Plus-2ea043)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen)

# Log Analysis Report — April 2026

---

## Report Metadata

| Field | Value |
| :--- | :--- |
| **Report Date** | 2026-04-08 |
| **Capture Window** | 17:41 – 23:22 UTC-4 (~5.5 hours) |
| **Interfaces Analysed** | WAN (`ix1`), VLAN50 IoT (`ix0.50`) |
| **Total Log Entries** | ~2,850 |
| **Analyst** | Gurvin Singh |

---

## Executive Summary

During a 5.5-hour capture window on 2026-04-08, approximately 2,850 firewall log entries were recorded across the WAN and IoT VLAN interfaces. The dominant external traffic consisted of automated internet-wide scanning targeting high-value service ports — including Telnet, SNMP, VNC, MongoDB, and WinRM — all blocked by the default-deny WAN policy. A GRE protocol probe was identified as an unusual non-TCP/UDP scan. Internally, Samsung TV discovery broadcasts on VLAN50 were identified as a significant log polluter and prompted the creation of a `SOC_SILENCE_SAMSUNG_DISCOVERY` suppression rule. All external probes were blocked. No lateral movement or internal compromise indicators were observed.

---

## Log Format Reference

Logs follow RFC 5424 syslog format via pfSense `filterlog`. See the [Log Analysis README](../README.md#log-format-explained) for full field definitions.

---

## Sample Log Entries

> All external source IPs are shown as captured — these are inbound scan sources from the public internet, blocked by the firewall. The local WAN IP has been redacted and replaced with `[REDACTED_WAN_IP]`.

### Reconnaissance: Inbound Port Scan — VNC (TCP 5900)

```text
<134>1 2026-04-08T17:41:33-04:00 edgefw.core.home.arpa filterlog - - -
  1,6,,1000000103,ix1,match,block,in,4,0x0,,114,24337,0,none,6,tcp,48,
  165.245.162.235,[REDACTED_WAN_IP],5949,5900,0,S,1484833334,,65535,,mss;nop;nop;sackOK
```

**Analysis:** TCP SYN packet from an external host to destination port 5900 (VNC). The `S` flag confirms this is a connection initiation probe — not an established session. VNC is a common target for attackers seeking unpatched remote desktop access with weak credentials. Blocked by default-deny WAN policy.

---

### Reconnaissance: Telnet Scan — Duplicate SYN (TCP 23)

```text
<134>1 2026-04-08T17:42:54-04:00 edgefw.core.home.arpa filterlog - - -
  1,6,,1000000103,ix1,match,block,in,4,0x0,,40,10980,0,none,6,tcp,40,
  180.163.62.126,[REDACTED_WAN_IP],56339,23,0,S,1165667900,,65535,,

<134>1 2026-04-08T17:42:54-04:00 edgefw.core.home.arpa filterlog - - -
  1,6,,1000000103,ix1,match,block,in,4,0x0,,40,10980,0,none,6,tcp,40,
  180.163.62.126,[REDACTED_WAN_IP],56339,23,0,S,1165667900,,65535,,
```

**Analysis:** Duplicate TCP SYN packets to port 23 (Telnet) from the same source within the same second. Identical sequence numbers indicate a retransmission from an automated scanner. Telnet is a primary target for IoT botnet recruitment — devices with default credentials are compromised and added to botnets like Mirai. Both packets blocked.

---

### Reconnaissance: SNMP Enumeration (UDP 161)

```text
<134>1 2026-04-08T17:42:34-04:00 edgefw.core.home.arpa filterlog - - -
  1,6,,1000000103,ix1,match,block,in,4,0x0,,53,9386,0,DF,17,udp,71,
  193.163.125.127,[REDACTED_WAN_IP],23052,161,51
```

**Analysis:** Inbound UDP probe to port 161 (SNMP). SNMP enumeration is used by attackers to extract network topology information, device configurations, and running interface lists via weak community strings (e.g., `public`). The `DF` (Don't Fragment) flag suggests this is a direct targeted probe, not a fragmented scan. Blocked.

---

### Suspicious: GRE Tunnel Probe (Protocol 47)

```text
<134>1 2026-04-08T17:43:10-04:00 edgefw.core.home.arpa filterlog - - -
  1,6,,1000000103,ix1,match,block,in,4,0x0,,54,5245,0,DF,47,gre,564,
  84.54.149.97,[REDACTED_WAN_IP],datalength=544
```

**Analysis:** Inbound GRE (Generic Routing Encapsulation, IP protocol 47) packet. GRE is not a service this network exposes. External GRE probes may indicate: (1) an attempt to establish a covert tunnel, (2) a misconfigured service scanning for GRE endpoints, or (3) internet-wide protocol diversity scanning. The data length of 544 bytes suggests this is not an empty probe. Blocked. Flagged as worth monitoring for recurrence.

---

### Reconnaissance: MongoDB Database Scan (TCP 27017)

```text
<134>1 2026-04-08T23:21:16-04:00 edgefw.core.home.arpa filterlog - - -
  1,6,,1000000103,ix1,match,block,in,4,0x0,,239,54359,0,none,6,tcp,44,
  43.228.157.45,[REDACTED_WAN_IP],44878,27017,0,S,4194725836,,1025,,mss
```

**Analysis:** TCP SYN to port 27017 (MongoDB). MongoDB instances have historically been deployed without authentication and exposed to the internet, leading to mass database exfiltration events. This probe is characteristic of automated MongoDB enumeration tooling. Blocked.

---

### Internal Noise: Samsung TV Discovery Broadcast (UDP 15600)

```text
<134>1 2026-04-08T23:20:10-04:00 edgefw.core.home.arpa filterlog - - -
  1,176,,1770416543,ix0.50,match,block,in,4,0x0,,64,63329,0,DF,17,udp,63,
  192.168.50.101,192.168.50.255,34728,15600,43
```

**Analysis:** UDP broadcast from an IoT device (Samsung TV) to the VLAN50 broadcast address on port 15600 — Samsung's proprietary device discovery protocol. This traffic occurs every ~5 seconds across all Samsung TVs on the segment. While benign, it generates significant log volume with zero security value. This entry directly prompted the creation of the `SOC_SILENCE_SAMSUNG_DISCOVERY` firewall rule to suppress this traffic class. The rule is logged separately from the default block to maintain auditability.

---

### Normal Traffic: Internal DNS Pass (UDP 53)

```text
<134>1 2026-04-08T17:41:16-04:00 edgefw.core.home.arpa filterlog - - -
  1,168,,1775611791,ix0.50,match,pass,in,4,0x0,,64,33907,0,DF,17,udp,76,
  192.168.50.32,127.0.0.1,56603,53,56
```

**Analysis:** An IoT device on VLAN50 (`192.168.50.32`) resolving DNS to `127.0.0.1:53` — the result of the DNS NAT override rule redirecting all port 53 traffic to the local Unbound resolver. This is expected, normal behaviour confirming that DNS enforcement is functioning correctly. The device's DNS query is transparently intercepted and resolved by Unbound regardless of the device's configured resolver.

---

## Traffic Analysis Summary

| Category | Estimated Count | % of Total | Notes |
| :--- | :--- | :--- | :--- |
| Inbound WAN blocks | ~2,100 | ~74% | External reconnaissance — all blocked |
| Internal VLAN50 noise | ~550 | ~19% | Samsung TV, IoT broadcasts |
| Legitimate passes (DNS, NTP, etc.) | ~200 | ~7% | Normal internal traffic |
| **Total** | **~2,850** | **100%** | |

---

## Reconnaissance / Threat Patterns Identified

| Port | Protocol | Service | Classification | Disposition |
| :--- | :--- | :--- | :--- | :--- |
| 23 | TCP | Telnet | Botnet recruitment scan | Blocked (default deny) |
| 161 | UDP | SNMP | Network enumeration | Blocked (default deny) |
| 5900 | TCP | VNC | Remote desktop access scan | Blocked (default deny) |
| 5985 | TCP | WinRM | Windows remote management | Blocked (default deny) |
| 8728 | TCP | MikroTik API | Router infrastructure targeting | Blocked (default deny) |
| 27017 | TCP | MongoDB | Unauthenticated database exfiltration | Blocked (default deny) |
| 7474 | TCP | Neo4j | Graph database exposure scan | Blocked (default deny) |
| N/A | GRE (47) | Tunnel probe | Protocol diversity scan / covert tunnel attempt | Blocked (default deny) |

> All probes were blocked by the WAN default-deny policy. This network exposes no services on any of these ports.

---

## Repeat Offender Netblocks

> External IPs are shown as captured from public internet traffic blocked at the WAN.

| Netblock | Behaviour |
| :--- | :--- |
| `85.217.149.x` | Repeated sequential SYN probes to random high ports — consistent with distributed scanning cluster |
| `193.46.255.x`, `193.163.125.x` | Multiple probes to SNMP, Telnet, and VNC ports |
| `198.235.24.x`, `147.185.132.x`, `35.203.210.x` | Persistent SYN scanning across multiple sessions |

_In a production SOC, these netblocks would be submitted to AbuseIPDB and cross-referenced against threat intelligence feeds (OTX AlienVault, VirusTotal, Shodan)._

---

## Internal Noise Identified

| Source Device / Type | Traffic | Port | VLAN | Disposition |
| :--- | :--- | :--- | :--- | :--- |
| Samsung Smart TVs | Discovery broadcast every ~5 sec | UDP 15600 | VLAN 50 | Suppressed — `SOC_SILENCE_SAMSUNG_DISCOVERY` |
| TP-Link Mesh Nodes | Discovery beacons | UDP 20002 | VLAN 10/20 | Suppressed — `SOC_SILENCE_DECO_DISCOVERY` |
| IoT devices (various) | DNS-over-TLS bypass attempts | TCP 853 | VLAN 50 | Blocked — `SOC_SILENCE_DOT_ATTEMPTS` |
| IPv6 multicast | General IPv6 chatter / mDNS | Various | All | Partially suppressed — IPv6 block rules |

---

## Rule Tuning Actions Taken

| Rule Name | Trigger | Result |
| :--- | :--- | :--- |
| `SOC_SILENCE_SAMSUNG_DISCOVERY` | Samsung TV UDP 15600 broadcast every 5 sec | ~550 log entries/hour eliminated |
| `SOC_SILENCE_DOT_ATTEMPTS` | IoT firmware attempting TCP 853 DoT | Noise suppressed; DoT bypass confirmed blocked |

---

## SIEM Readiness Notes

- RFC 5424 format confirmed — logs are natively compatible with Splunk, Elastic/ELK, Wazuh, and Microsoft Sentinel.
- ISO 8601 timestamps with timezone offset present — no normalisation required for most SIEM platforms.
- Comma-separated `filterlog` payload maps cleanly to field-level parsing rules.
- `SOC_SILENCE_*` rule prefix enables SIEM filter rules to exclude noise at ingest without losing audit trail.

---

## Recommendations

1. **GeoIP enrichment** — Add country-of-origin to all inbound WAN block events. High-volume scan sources from high-risk regions may warrant additional visibility.
2. **Threat intel correlation** — Submit repeat offender netblocks (`85.217.149.x`, `193.163.125.x`) to AbuseIPDB and flag for automated reputation scoring.
3. **SIEM scan detection rule** — Create correlation rule: if a single source IP generates SYN packets to >20 unique destination ports within 5 minutes, fire a medium-severity alert.
4. **GRE monitoring** — Add a specific alert for inbound GRE (protocol 47) from any source. This traffic has no legitimate use case against this network and any recurrence warrants investigation.
5. **Expand log capture** — Include DNS query logs from Unbound in the next analysis window to correlate DNS behaviour with firewall events.
