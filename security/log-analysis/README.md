![Category](https://img.shields.io/badge/Category-Log%20Analysis-blue)
![Platform](https://img.shields.io/badge/Platform-pfSense%20Plus-2ea043)
![Status](https://img.shields.io/badge/Status-Active-brightgreen)

# Log Analysis

Firewall log analysis is a foundational SOC skill. This directory documents the methodology used to parse, classify, and act on pfSense firewall logs in this lab, and contains real-world analysis reports generated from captured traffic.

---

## Overview

pfSense outputs structured firewall logs via the `filterlog` facility using RFC 5424 syslog format. These logs capture every rule match across all interfaces and VLANs. Analysing them reveals active reconnaissance, internal device noise, and opportunities to refine firewall policy and SIEM alerting.

> **SOC Relevance:** Firewall logs are a primary data source in any SOC environment. The ability to parse log format, distinguish signal from noise, and tune rules for SIEM ingestion directly mirrors Tier 1 analyst responsibilities.

---

## Log Format Explained

pfSense `filterlog` entries use the RFC 5424 syslog envelope with a comma-separated payload:

```
<priority>version timestamp hostname appname procid msgid - payload
```

**Key fields in the `filterlog` payload:**

| Field | Description | Example |
| :--- | :--- | :--- |
| Rule Number | ID of the firewall rule that matched | `1000000103` |
| Interface | Interface where traffic was observed | `ix1` (WAN), `ix0.50` (VLAN50) |
| Action | Firewall decision | `block`, `pass` |
| Direction | Traffic flow | `in`, `out` |
| IP Version | Address family | `4` (IPv4), `6` (IPv6) |
| Protocol | Layer 4 protocol | `tcp`, `udp`, `gre`, `icmp` |
| Source IP | Originating address | External or internal IP |
| Destination IP | Target address | WAN IP or internal subnet |
| Source Port | Originating port (TCP/UDP) | `56339` |
| Destination Port | Target port (TCP/UDP) | `23` |
| TCP Flags | Connection state indicators | `S` (SYN), `SA` (SYN-ACK) |

---

## Analysis Methodology

The approach used for each analysis cycle:

1. **Parse and validate** - confirm fields map correctly to the RFC 5424 schema and timestamps are well-formed.
2. **Identify high-volume noise** - find broadcast, multicast, and vendor discovery traffic generating log volume with no security value.
3. **Extract anomalies** - isolate inbound probes, unusual protocols, or unexpected internal traffic.
4. **Classify by threat type** - map anomalies to known threat patterns (see table below).
5. **Attribute intent** - differentiate malicious scanning from operational vendor traffic (e.g., Microsoft STUN) and academic/research scanning (e.g., Palo Alto Unit42, SURFnet).
6. **Geo and threat intel enrichment** - look up source ASNs via ip-api.com/ipinfo.io and run top offenders through VirusTotal API v3.
7. **Map to MITRE ATT&CK** - assign observed techniques to ATT&CK Enterprise technique IDs to contextualise findings within a standard framework.
8. **Tune suppression rules** - create `SOC_SILENCE_*` rules for confirmed noise to reduce volume without losing the audit trail.
9. **Correlate patterns** - identify repeat offender netblocks, targeted service clusters, and timing anomalies.
10. **Document and report** - produce a structured report using the [standard template](./reports/report-template.md).

---

## Threat Pattern Categories

| Pattern | Indicator | Common Ports | Disposition |
| :--- | :--- | :--- | :--- |
| **Distributed Port Scan** | Multiple IPs from same /24 each hitting a few ports | Any | Flag netblock; correlate to ASN for cluster attribution |
| **Sequential Port Sweep** | Single source hitting many ports in order | Any | High-priority alert; cloud instance likely abused |
| **RDP Campaign** | Coordinated SYN to 3389 variants from /24 cluster | 3389, 3390, 3391 | Flag for threat intel; consider geo-block |
| **Service Reconnaissance** | Repeated probes to specific sensitive services | 22, 23, 161, 5900, 5985 | Flag for threat intel enrichment |
| **Unauthenticated API Scan** | Probes to services with no auth by default | 2375 (Docker), 9200 (ES), 27017 (Mongo) | High-severity alert; confirm no exposure |
| **IoT/Mobile Targeting** | Probes to device management ports | 5555 (ADB), 8728 (MikroTik) | Confirm block; monitor for volume increase |
| **VoIP Enumeration** | Large SIP payload to port 5060 | 5060 | Confirm block; toll fraud risk |
| **Protocol Abuse** | Unexpected protocol (GRE, ICMP tunnelling) | Protocol 47, ICMP | Investigate; alert if pattern repeats |
| **Database Exposure Scan** | SYN probes to database ports from external sources | 27017, 5432, 3306, 7474, 9200 | Confirm no exposure; log for awareness |
| **Internal Device Noise** | Broadcast/multicast from known devices on schedule | 5353, 15600, 20002 | Tune with `SOC_SILENCE_*` rule |
| **DNS Bypass Attempt** | Outbound DNS from client to non-Unbound destination | 53, 853 | Block rule already in place |
| **Operational False Positive** | Vendor infrastructure (e.g., Microsoft STUN) | 3478 | Confirm via VT; suppress with `SOC_SILENCE_*` |

---

## Intent Classification

Each threat pattern entry in a report includes an **Intent** label. This distinguishes sources that look like scanning but have a benign or research-driven purpose.

| Intent | Meaning | Examples |
| :--- | :--- | :--- |
| **Malicious** | Confirmed hostile scanning or exploitation attempts | RDP clusters, Docker API hunts, botnet recruitment |
| **Operational** | Legitimate vendor infrastructure triggering block rules | Microsoft Teams STUN (UDP 3478), CDN QUIC |
| **Academic/Research** | Internet measurement or threat intel collection | SURFnet, USC, Palo Alto Unit42, Driftnet |
| **Unknown** | Internet-wide noise with no confirmed attribution | General HTTP/HTTPS/SSH probes |

---

## MITRE ATT&CK Mapping

Reports include a MITRE ATT&CK technique mapping table to contextualise observed reconnaissance within the Enterprise framework (v15). This maps firewall-level observations to attacker objectives even when no exploitation occurred.

Common techniques observed in this lab's WAN traffic:

| Technique ID | Name |
| :--- | :--- |
| T1595.001 | Active Scanning: Scanning IP Blocks |
| T1595.002 | Active Scanning: Vulnerability Scanning |
| T1046 | Network Service Discovery |
| T1133 | External Remote Services |
| T1110 | Brute Force |
| T1190 | Exploit Public-Facing Application |
| T1219 | Remote Access Software |
| T1071.005 | Application Layer Protocol: VoIP |
| T1572 | Protocol Tunneling |
| T1590.002 | Gather Victim Network Info: Network Topology |

---

## SOC_SILENCE Convention

Rules prefixed `SOC_SILENCE_*` are dedicated noise-suppression rules. Their naming convention serves two purposes:

1. **Analyst clarity** - `SOC_SILENCE_*` rules are immediately distinguishable from actual security policy. Analysts know these entries are confirmed noise, not events requiring investigation.
2. **SIEM filter efficiency** - SIEM ingest pipelines can exclude `SOC_SILENCE_*` rule matches from alert queues while still retaining them for audit purposes.

Rules in use:

| Rule Name | Suppresses |
| :--- | :--- |
| `SOC_SILENCE_AZURE_STUN` | UDP 3478 from `52.115.0.0/16` (Microsoft Teams NAT traversal) |
| `SOC_SILENCE_SAMSUNG_DISCOVERY` | UDP 15600 Samsung TV broadcast on VLAN50 |
| `SOC_SILENCE_DOT_ATTEMPTS` | Outbound TCP 853 DoT bypass attempts from IoT VLAN |
| `SOC_SILENCE_DECO_DISCOVERY` | UDP 20002 TP-Link Deco mesh discovery beacons |
| `SOC_SILENCE_QUIC_NOISE` | UDP 443 QUIC from CDNs and browsers |
| `SOC_SILENCE_TCP_GHOSTS` | TCP to ghost ports with no matching service |
| `SOC_SILENCE_IOT_NOISE` | Chatty IoT device broadcast traffic |

---

## IP Redaction Policy

All reports follow a consistent redaction standard to protect operational security while retaining threat intelligence value.

| Location | Rule |
| :--- | :--- |
| Log entry samples (source IP) | Fully redacted as `[REDACTED_IP]` |
| Log entry samples (WAN/dest IP) | Fully redacted as `[REDACTED_WAN_IP]` |
| Prose and table references (individual IPs) | Last octet redacted: `1.2.3.xxx` |
| Repeat Offender table (netblocks) | Shown as-is: `x.x.x.0/24` (public threat intel) |
| Internal RFC 1918 addresses | Shown as-is |

---

## Report Structure

Each report produced from the [standard template](./reports/report-template.md) includes the following sections:

| Section | Purpose |
| :--- | :--- |
| Key Metrics Banner | Headline stats at a glance (entries, unique IPs, block rate, posture) |
| Report Metadata | Capture window, interfaces, analyst |
| Executive Summary + Risk Assessment | Summary of findings and validated security posture statement |
| Log Format Reference | Link to this README |
| Sample Log Entries | Redacted real entries with analysis of threat behaviour |
| Traffic Analysis Summary | Categorised volume breakdown |
| Reconnaissance / Threat Patterns | Port-level table with Intent classification |
| MITRE ATT&CK Mapping | Technique IDs mapped to observed behaviour |
| VirusTotal Scan Results | VT API v3 scores for top offending IPs |
| Repeat Offender Netblocks | ASN/geo attribution for top attacking clusters |
| Internal Noise Identified | IoT and vendor traffic classified as benign |
| Rule Tuning Actions Taken | SOC_SILENCE rules created and their outcome |
| SIEM Readiness Notes | Correlation rules, enrichment opportunities |
| Recommendations | Action items derived from the analysis |

---

## Report Naming Convention

Reports follow the format `FW-LAR-YYYY-MM-DD.md`:

- `FW` = asset type (Firewall)
- `LAR` = report type (Log Analysis Report)
- `YYYY-MM-DD` = date the analysis was completed

---

## SIEM Readiness

pfSense `filterlog` logs are well-suited for SIEM ingestion:

- **Format:** RFC 5424 with ISO 8601 timestamps, natively parseable by Splunk, Elastic, Wazuh, and Microsoft Sentinel.
- **Structure:** Comma-separated fields with consistent ordering, no custom parsing required for most platforms.
- **Volume control:** `SOC_SILENCE_*` rules reduce log volume before ingestion, keeping storage costs and alert fatigue low.
- **Threat intel enrichment:** Top offending IPs are submitted to VirusTotal API v3 (rate limited to 4 requests/min on the free plan). GeoIP and ASN attribution via ip-api.com and ipinfo.io.

**Planned SIEM enhancements:**
- GeoIP tagging on all inbound WAN block events.
- Automated VirusTotal lookup for any source IP exceeding 50 blocked events per hour.
- Correlation rule: more than 10 unique IPs from the same /24 targeting port 3389 within 60 minutes fires a medium-severity RDP cluster alert.
- Correlation rule: single source IP hitting more than 50 unique destination ports within 30 minutes fires a high-severity port sweep alert.
- Cross-log correlation with Unbound DNS query logs and authentication events.

---

## Analysis Reports

Reports are stored in the [`reports/`](./reports/) directory and named using the `FW-LAR-YYYY-MM-DD.md` convention.

| Report | Period | Entries | Key Findings |
| :--- | :--- | :--- | :--- |
| [FW-LAR-2026-04-09](./reports/FW-LAR-2026-04-09.md) | 2026-04-08 17:41 – 2026-04-09 18:48 (~25 hrs) | 16,417 | RDP cluster campaign (BG/RO), AWS port sweep (154 ports), Docker API hunting, ADB scan, Azure STUN false positive identified, 10 IPs VT-confirmed malicious |
