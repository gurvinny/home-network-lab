![Category](https://img.shields.io/badge/Category-Log%20Analysis%20Report-blue)
![Platform](https://img.shields.io/badge/Platform-pfSense%20Plus-2ea043)
![Status](https://img.shields.io/badge/Status-Template-lightgrey)

# Log Analysis Report — [MONTH YEAR]

> **Instructions:** Copy this file to `reports/YYYY-MM-analysis.md`. Fill in each section. Redact all external source IPs (replace with `[REDACTED_IP]` or `x.x.x.x`). Redact WAN IP (replace with `[REDACTED_WAN_IP]`). Internal RFC 1918 addresses may be shown. Remove this instruction block before publishing.

---

## Report Metadata

| Field | Value |
| :--- | :--- |
| **Report Date** | YYYY-MM-DD |
| **Capture Window** | HH:MM – HH:MM UTC±X (~X hours) |
| **Interfaces Analysed** | e.g., WAN (ix1), VLAN50 (ix0.50) |
| **Total Log Entries** | ~X,XXX |
| **Analyst** | Gurvin Singh |

---

## Executive Summary

_2–3 sentences summarising the capture window, key findings, and any actions taken. Example: "During a 5.5-hour capture window, approximately 2,850 firewall log entries were recorded. The majority of inbound traffic on the WAN interface consisted of automated internet-wide scanning targeting common service ports. Internal noise from IoT devices on VLAN50 was identified and tuned via new SOC_SILENCE rules."_

---

## Log Format Reference

Logs follow RFC 5424 syslog format via pfSense `filterlog`. See the [Log Analysis README](../README.md#log-format-explained) for full field definitions.

---

## Sample Log Entries

> All external source IPs redacted. WAN IP replaced with `[REDACTED_WAN_IP]`.

### [Event Type — e.g., Reconnaissance: Port Scan]

```text
[Paste redacted log entry here]
```

**Analysis:** _What does this log entry show? What threat behaviour does it represent?_

---

### [Event Type — e.g., Internal Noise: Device Broadcast]

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

| Port | Protocol | Service | Classification | Volume | Disposition |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 23 | TCP | Telnet | Automated scan | High | Blocked (default deny) |
| _[port]_ | _[proto]_ | _[service]_ | _[classification]_ | _[volume]_ | _[action]_ |

---

## Repeat Offender Netblocks

> External IPs redacted. ASN/geolocation enrichment listed where available.

| Netblock (Redacted) | ASN | Country | Behaviour |
| :--- | :--- | :--- | :--- |
| `x.x.x.x/24` | ASxxxxx | [Country] | [Describe: sequential port scan, repeated SYN, etc.] |

_In a production SOC, these would be cross-referenced against AbuseIPDB, VirusTotal, and OTX AlienVault._

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

_Observations about log quality, parsing compatibility, or enrichment opportunities specific to this capture window._

- RFC 5424 format confirmed — compatible with [Splunk / Elastic / Wazuh / Sentinel].
- Timestamp format: ISO 8601 with timezone offset.
- Suggested enrichments: [GeoIP, threat intel lookup, etc.]

---

## Recommendations

1. _[Action item derived from this analysis — e.g., "Add GeoIP enrichment to inbound block events"]_
2. _[Action item — e.g., "Investigate repeat offender netblock x.x.x.x/24 against AbuseIPDB"]_
3. _[Action item — e.g., "Create SIEM correlation rule: >20 unique dest ports from single source in 5 min"]_
