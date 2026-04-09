![Category](https://img.shields.io/badge/Category-Log%20Analysis-blue)
![Platform](https://img.shields.io/badge/Platform-pfSense%20Plus-2ea043)
![Status](https://img.shields.io/badge/Status-Active-brightgreen)

# Log Analysis

Firewall log analysis is a foundational SOC skill. This directory documents the methodology used to parse, classify, and act on pfSense firewall logs in this lab — and contains real-world analysis reports generated from captured traffic.

---

## Overview

pfSense outputs structured firewall logs via the `filterlog` facility using RFC 5424 syslog format. These logs capture every rule match — both allowed and denied traffic — across all interfaces and VLANs. Analysing these logs reveals reconnaissance attempts, internal device noise, and opportunities to refine firewall policy.

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

1. **Parse the format** — confirm fields map correctly to the RFC 5424 schema.
2. **Identify high-volume noise** — find broadcast, multicast, and vendor discovery traffic generating log volume with no security value.
3. **Extract anomalies** — isolate inbound probes, unusual protocols, or unexpected internal traffic.
4. **Classify by threat type** — map anomalies to known threat patterns (see table below).
5. **Tune suppression rules** — create `SOC_SILENCE_*` rules for confirmed noise to reduce volume.
6. **Correlate patterns** — identify repeat offenders, targeted service clusters, and timing patterns.
7. **Document and report** — produce a structured report using the [standard template](./reports/report-template.md).

---

## Threat Pattern Categories

| Pattern | Indicator | Common Ports | Disposition |
| :--- | :--- | :--- | :--- |
| **Port Scan** | Rapid sequential SYN to many ports from one source | Any | Confirm block; flag source for threat intel |
| **Service Reconnaissance** | Repeated probes to specific sensitive services | 22, 23, 161, 5900, 5985 | Flag for threat intel enrichment |
| **Protocol Abuse** | Unexpected protocol on a standard or non-standard port | GRE, ICMP tunnelling | Investigate; alert if pattern repeats |
| **Database Exposure Scan** | SYN probes to database ports from external sources | 27017, 5432, 3306, 7474 | Confirm no exposure; log for awareness |
| **Internal Device Noise** | Broadcast/multicast from known devices on schedule | 5353, 15600, 20002 | Tune with `SOC_SILENCE_*` rule |
| **DNS Bypass Attempt** | Outbound DNS from client to non-Unbound destination | 53, 853 | Block rule already in place; alert if rule is missing |

---

## SOC_SILENCE Convention

Rules prefixed `SOC_SILENCE_*` are dedicated noise-suppression rules. Their naming convention serves two purposes:

1. **Analyst clarity** — when reading raw logs or building SIEM queries, `SOC_SILENCE_*` rules are immediately distinguishable from actual security policy enforcement. Analysts know these entries are confirmed noise, not events requiring investigation.
2. **SIEM filter efficiency** — SIEM ingest pipelines can exclude `SOC_SILENCE_*` rule matches from alert queues while still retaining them for audit purposes.

Examples in use:

| Rule Name | Suppresses |
| :--- | :--- |
| `SOC_SILENCE_QUIC_NOISE` | UDP 443 QUIC from CDNs and browsers |
| `SOC_SILENCE_SAMSUNG_DISCOVERY` | UDP 15600 Samsung TV broadcast |
| `SOC_SILENCE_TCP_GHOSTS` | TCP to ghost ports with no matching service |
| `SOC_SILENCE_DOT_ATTEMPTS` | Outbound TCP 853 DoT bypass attempts |
| `SOC_SILENCE_IOT_NOISE` | Chatty IoT device broadcast traffic |

---

## SIEM Readiness

pfSense `filterlog` logs are well-suited for SIEM ingestion:

- **Format:** RFC 5424 with ISO 8601 timestamps — natively parseable by Splunk, Elastic, Wazuh, and Microsoft Sentinel.
- **Structure:** Comma-separated fields with consistent ordering — no custom parsing required for most platforms.
- **Volume control:** `SOC_SILENCE_*` rules reduce log volume before ingestion, keeping storage costs and alert fatigue low.

**Planned SIEM enhancements:**
- GeoIP enrichment on source IPs.
- Automated threat intel correlation (AbuseIPDB / VirusTotal).
- Behavioural detection rules (e.g., >20 unique destination ports from one source in 5 minutes).
- Cross-log correlation with DNS query logs and authentication events.

---

## Analysis Reports

Individual analysis reports are stored in the [`reports/`](./reports/) directory. Each report follows the [standard template](./reports/report-template.md).

| Report | Period | Key Findings |
| :--- | :--- | :--- |
| [April 2026](./reports/2026-04-analysis.md) | 2026-04-08 ~5.5 hrs | ~2,850 entries; external recon on 7 service ports; Samsung TV noise; MongoDB/VNC/Telnet scans |
