![Category](https://img.shields.io/badge/Category-Security-blue?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Active-brightgreen?style=for-the-badge)
![Policy](https://img.shields.io/badge/Compliance-CIS_Level_2-red?style=for-the-badge)

# 🛡️ Wazuh SIEM Deployment & CIS Hardening

This document outlines the deployment, storage partitioning, and system hardening of the Wazuh Manager, which serves as the primary SIEM and log aggregation platform for the lab environment.

---

## 📈 System Hardening (CIS Level 2)

The Wazuh Manager is hosted on an **Ubuntu Server Pro** virtual machine within the Management VLAN (VLAN 20). By utilizing Ubuntu Pro, the server benefits from **Canonical Livepatch** (for zero-downtime kernel security updates) and the **Ubuntu Security Guide (USG)**. The operating system has been rigorously hardened using USG, achieving a **92% CIS (Center for Internet Security) Level 2 Server score**.

> **Security Rationale:** SIEM infrastructure is a high-value target as it contains sensitive log data, alerting logic, and administrative configurations. Hardening the underlying OS to CIS Level 2 standards significantly reduces the attack surface, disabling unnecessary services and enforcing strict access controls.

---

## 🗄️ Security-Based Partitioning

To comply with CIS recommendations and prevent denial-of-service (DoS) conditions caused by disk exhaustion (log overflow), the Ubuntu Server Pro instance utilizes a strict partition scheme.

A dedicated partition is allocated for `/var/ossec` to isolate Wazuh's data processing overhead from the core operating system.

| Mount Point | Purpose | CIS Rationale / Protection |
| :--- | :--- | :--- |
| `/var` | System variables and data | Prevents system data from filling the root (`/`) partition. |
| `/var/log` | System logs | Isolates generic system logs to prevent service failure on overflow. |
| `/var/log/audit` | Audit daemon logs | Ensures audit trails are preserved without impacting other log services. |
| `/home` | User directories | Prevents users from consuming critical system disk space. |
| `/var/ossec` | Wazuh Manager data | **Custom Requirement:** Isolates Wazuh agent logs, vulnerability data, and decoders from system logs, ensuring SIEM overflow does not crash the host OS. |

---

## 🚀 Future Roadmap & Integrations

The following enhancements are planned for the Wazuh SIEM deployment to increase visibility and threat detection across the lab:

- [ ] **pfSense Syslog Ingestion:** Configure pfSense to forward firewall logs (syslog) to the Wazuh Manager.
- [ ] **Custom Decoders & Rules:** Develop custom decoders and alerting rules specifically tailored to pfSense firewall events.
- [ ] **Honeypot Deployment:** Deploy a honeypot server in an isolated VLAN and integrate its logs with Wazuh for proactive threat monitoring.
- [ ] **Minecraft Log Integration:** Feed Minecraft server logs into Wazuh to monitor for abuse, unauthorized access attempts, or application-level attacks.
- [ ] **Dashboard Optimization:** Build and optimize custom Wazuh dashboards specifically for visualizing pfSense traffic and Game Server metrics.