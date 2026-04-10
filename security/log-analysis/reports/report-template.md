![Category](https://img.shields.io/badge/Category-Log%20Analysis%20Report-blue)
![Platform](https://img.shields.io/badge/Platform-pfSense%20Plus-2ea043)
![Status](https://img.shields.io/badge/Status-Template-lightgrey)

# Log Analysis Report: [MONTH YEAR]

| Total Log Entries | Unique Source IPs | Blocked | VT-Confirmed Malicious Hosts | Security Posture |
|:---:|:---:|:---:|:---:|:---:|
| **~X,XXX** | **~X,XXX** | **~X%** | **~X** | **[Posture]** |

> **Instructions:** Copy this file to `reports/FW-LAR-YYYY-MM-DD.md` (date = report completion date). Fill in each section. In log entry samples, redact all external source IPs with `[REDACTED_IP]` and the WAN IP with `[REDACTED_WAN_IP]`. In prose and tables, redact the last octet of individual IPs (e.g., `1.2.3.xxx`). Netblock notation (`x.x.x.0/24`) is acceptable. Internal RFC 1918 addresses may be shown as-is. Remove this instruction block before publishing.

---

## Report Metadata

| Field | Value |
| :--- | :--- |
| **Report Date** | YYYY-MM-DD |
| **Capture Window** | HH:MM – HH:MM UTC±X (~X hours) |
| **Interfaces Analysed** | e.g., WAN (ix1), VLAN50 (ix0.50) |
| **Total Log Entries** | ~X,XXX |
| **Unique External Source IPs** | ~X,XXX |
| **Analyst** | Gurvin Singh |

---

## Executive Summary

_2–3 sentences summarising the capture window, key findings, and any actions taken._

### Risk Assessment

_One sentence on security posture and one on the most significant validated finding. Example: "The security posture remains Hardened. All external probes were blocked with no lateral movement indicators. The most significant finding was [X], which validates the necessity of [Y]."_

---

## Log Format Reference

Logs follow RFC 5424 syslog format via pfSense `filterlog`. See the [Log Analysis README](../README.md#log-format-explained) for full field definitions.

---

## Sample Log Entries

> All external source IPs are redacted with `[REDACTED_IP]`. WAN IP replaced with `[REDACTED_WAN_IP]`. Internal RFC 1918 addresses shown as-is.

### [Event Type, e.g., Reconnaissance: Port Scan]

```text
[Paste redacted log entry here]
```

**Analysis:** _What does this log entry show? What threat behaviour does it represent?_

---

### [Event Type, e.g., Internal Noise: Device Broadcast]

```text
[Paste redacted log entry here]
```

**Analysis:** _What device generated this? Is it a security concern or benign noise?_

---

_(Add additional entries as needed.)_

---

## Traffic Analysis Summary

| Category | Count | % of Total | Notes |
| :--- | :--- | :--- | :--- |
| Inbound WAN blocks | X | X% | External reconnaissance, scanners |
| Internal noise (suppressed) | X | X% | IoT broadcasts, discovery protocols |
| Legitimate passes | X | X% | Internal DNS, NTP, internet traffic |
| **Total** | **X,XXX** | **100%** | |

---

## Reconnaissance / Threat Patterns Identified

| Port | Protocol | Service | Classification | Intent | Volume | Disposition |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 23 | TCP | Telnet | Botnet recruitment scan | Malicious | High | Blocked |
| _[port]_ | _[proto]_ | _[service]_ | _[classification]_ | _[intent]_ | _[volume]_ | _[action]_ |

> Intent key: **Malicious** = confirmed hostile scanning. **Operational** = legitimate vendor infrastructure (false positive). **Academic/Research** = internet measurement or threat intel collection. **Unknown** = internet-wide noise with no confirmed attribution.

---

## MITRE ATT&CK Technique Mapping

> Mapped to MITRE ATT&CK Enterprise framework v15. All techniques represent the reconnaissance phase only. Default-deny WAN policy prevented any technique from progressing beyond initial contact.

| Technique ID | Technique Name | Observed Behaviour | Source Example |
| :--- | :--- | :--- | :--- |
| T1595.001 | Active Scanning: Scanning IP Blocks | _[describe]_ | _[netblock]_ |
| T1595.002 | Active Scanning: Vulnerability Scanning | _[describe]_ | _[netblock or redacted IP]_ |
| T1046 | Network Service Discovery | _[describe]_ | _[ports observed]_ |
| _[T-ID]_ | _[name]_ | _[behaviour]_ | _[source]_ |

---

## VirusTotal Scan Results

> Top offenders scanned via VT API v3. Individual IPs have last octet redacted. All source IPs are active internet scanners blocked at the WAN.

| IP (Redacted) | VT Malicious | VT Suspicious | Reputation | Country | ASN Owner | Last VT Scan |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `x.x.x.xxx` | X | X | X | [Country] | [ASN Owner] | YYYY-MM-DD |

---

## Repeat Offender Netblocks

> Netblock notation is acceptable for threat intel purposes. Individual IPs in prose are last-octet redacted. ASN and geolocation sourced from ip-api.com, ipinfo.io, and VirusTotal.

| Netblock | Hits | ASN | Country | Org | Behaviour |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `x.x.x.0/24` | X | ASxxxxx | [Country] | [Org] | [Describe behaviour] |

---

## Internal Noise Identified

| Source Device / Type | Traffic Type | Port | VLAN | Disposition |
| :--- | :--- | :--- | :--- | :--- |
| [Device type] | [Broadcast / Discovery / etc.] | [Port] | [VLAN ID] | [Suppressed / Allowed / Investigate] |

---

## Rule Tuning Actions Taken

| Rule Name | Rule Type | Trigger | Outcome |
| :--- | :--- | :--- | :--- |
| `SOC_SILENCE_[NAME]` | Block | [What traffic prompted this] | [Noise suppressed; log volume reduced by X%] |

---

## SIEM Readiness Notes

- RFC 5424 format confirmed, compatible with Splunk, Elastic/ELK, Wazuh, and Microsoft Sentinel.
- Timestamp format: ISO 8601 with timezone offset.
- Suggested enrichments: [GeoIP, threat intel lookup, etc.]
- `SOC_SILENCE_*` rule prefix enables SIEM filter rules to exclude noise at ingest without losing the audit trail.

---

## Recommendations

1. _[Action item derived from this analysis]_
2. _[Action item]_
3. _[Action item]_
