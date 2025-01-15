# LynxGuard

> **Vulnerability Assessment and Security Monitoring for Web Application**  
> Graduation Thesis (IAP491) · FPT University, Hoa Lac Campus · Sep–Dec 2024  
> Team of 4 · Thesis honored by faculty council

---

## Problem Statement

FPT Education's web applications (fap.fpt.edu.vn, fu-edunext.fpt.edu.vn, career.fpt.edu.vn) lacked continuous security monitoring. The existing setup relied on manual checks and isolated tools — no unified pipeline for detecting, correlating, and alerting on web attacks in real-time. Attacks like SQLi, XSS, brute force, and DoS could go undetected for extended periods.

LynxGuard addresses this by integrating multiple open-source security tools into a cohesive SOC pipeline: from network-level detection (Snort) to application-layer filtering (ModSecurity), centralized log aggregation and correlation (Wazuh SIEM), infrastructure monitoring (Zabbix), and real-time alerting (Telegram bot).

---

## Architecture

```
                        Internet Traffic
                              │
                              ▼
               ┌──────────────────────────┐
               │      pfSense Firewall     │
               │  NAT · Routing · Zones    │
               └──────────┬───────────────┘
                           │
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
     ┌─────────────┐ ┌──────────┐ ┌──────────────┐
     │    Snort    │ │  Apache  │ │    Zabbix    │
     │  IDS/IPS   │ │    +     │ │  Monitoring  │
     │  (Network) │ │ModSecurity│ │  (Infra)    │
     └──────┬──────┘ │  (WAF)  │ └──────┬───────┘
            │        └────┬────┘        │
            │   Logs/Events│             │
            └──────────────┼────────────┘
                           │  syslog / agents
                           ▼
              ┌────────────────────────┐
              │      Wazuh Manager     │
              │  Log aggregation       │
              │  Correlation rules     │
              │  Threat detection      │
              │  Alert generation      │
              └────────────┬───────────┘
                           │
              ┌────────────┼──────────────┐
              ▼            ▼              ▼
      ┌──────────────┐ ┌────────┐ ┌─────────────┐
      │Wazuh Dashboard│ │ Active │ │Telegram Bot │
      │(Kibana-based) │ │Response│ │  (Python)   │
      └──────────────┘ └────────┘ └─────────────┘
```

**Environment:** Fully virtualized on VMware Workstation. All components run as separate VMs on a single host, with virtual network segments simulating a real enterprise environment.

| VM | OS | Role |
|----|-----|------|
| Gateway | pfSense 2.7 | Firewall, routing, NAT |
| Web Server | Ubuntu 22.04 | Apache + ModSecurity (WAF) |
| IDS | Ubuntu 22.04 | Snort 3 (network IDS) |
| SIEM | Ubuntu 22.04 | Wazuh Manager + Dashboard |
| Monitoring | Ubuntu 22.04 | Zabbix Server |
| Client | Windows 10 | Attacker simulation + testing |

---

## Stack

| Layer | Tool | Version | Role |
|-------|------|---------|------|
| Firewall | pfSense | 2.7 | Perimeter control, traffic routing between segments |
| IDS/IPS | Snort | 3.x | Signature-based network intrusion detection |
| WAF | ModSecurity | 3.x | OWASP Core Rule Set, application-layer filtering |
| SIEM | Wazuh | 4.x | Centralized log aggregation, correlation, threat detection |
| Monitoring | Zabbix | 6.x | Infrastructure health, threshold-based alerting |
| Vuln Assessment | Acunetix | — | Automated web vulnerability scanning |
| Pentesting | BurpSuite | Community | Manual web application testing |
| Alerting | Telegram Bot | Python | Real-time push notifications for security events |

---

## Data Flow

```
Attack detected
    │
    ├─▶ Snort (network layer)
    │       └─▶ Generates alert → syslog → Wazuh agent
    │
    ├─▶ ModSecurity (application layer)
    │       └─▶ Logs to Apache error log → Wazuh agent
    │
    ├─▶ pfSense (firewall)
    │       └─▶ syslog forwarded to Wazuh Manager
    │
    └─▶ Zabbix (infra monitoring)
            └─▶ Trigger action → notification → Telegram

All sources ──▶ Wazuh Manager
                    ├─▶ Correlate events across sources
                    ├─▶ Apply detection rules
                    ├─▶ Generate unified alerts
                    └─▶ Telegram Bot (high-severity events)
```

---

## Detection Coverage

| Attack Type | Detected By | Method |
|------------|-------------|--------|
| SQL Injection | ModSecurity + Snort | OWASP CRS rules + Snort signatures |
| XSS | ModSecurity | OWASP CRS rules |
| Port Scanning | Snort | Threshold-based signature |
| Brute Force | Wazuh | Log correlation — failed auth frequency |
| DoS / High traffic | Zabbix + pfSense | Bandwidth thresholds + packet rate |
| Directory Traversal | ModSecurity | OWASP CRS rules |
| Command Injection | ModSecurity | OWASP CRS rules |
| Protocol anomalies | Snort | Anomaly-based rules |

---

## Testing Methodology

Testing was conducted in three phases:

**Phase 1 — Automated vulnerability scanning**  
Acunetix scanned the target web application environment to establish a baseline of known vulnerabilities before the monitoring stack was fully deployed.

**Phase 2 — Simulated attacks against monitored environment**  
BurpSuite and manual testing were used to execute attacks (SQLi, XSS, brute force, directory traversal) against the live monitoring stack to verify detection and alerting.

**Phase 3 — Log analysis and tuning**  
Analyzed Wazuh alerts and Snort/ModSecurity logs from Phase 2 to identify false positives, tune rules, and validate end-to-end detection from attack execution to Telegram notification.

---

## My Contribution

Working as part of a 4-person team:

- **VMware environment setup:** Provisioned and configured all VMs — OS installation, network adapter assignment, snapshot management for repeatable testing
- **Network design:** Designed virtual network topology with isolated segments (WAN, LAN, DMZ zones) to simulate real enterprise traffic flow
- **pfSense configuration:** Firewall rules, NAT policies, inter-zone routing, syslog forwarding to Wazuh
- **Telegram bot (Python):** Wrote the alerting bot that subscribes to Zabbix and Wazuh alert webhooks and pushes formatted notifications to the team Telegram group; handles alert severity filtering and message formatting

---

## Key Findings

- **WAF effectiveness:** ModSecurity with OWASP CRS blocked 100% of automated SQLi/XSS attempts from Acunetix scanner
- **IDS false positive rate:** Snort generated high false-positive volume on normal HTTPS traffic — required significant rule tuning before practical use
- **Alert correlation:** Wazuh successfully correlated events across ModSecurity, Snort, and pfSense logs for multi-stage attack detection; without correlation, the same attack appeared as 3+ separate unrelated events in individual tool dashboards
- **End-to-end latency:** Average time from attack execution to Telegram notification: ~8 seconds

---

## Limitations

- Fully virtualized environment — real-world performance characteristics (hardware resource contention, physical network latency) were not captured
- Wazuh rule set used default rules + minimal custom tuning; production deployment would require significant rule customization for the specific application stack
- Testing scope limited to simulated attacks against a controlled environment — real attacker behavior (evasion, slow-and-low techniques) was not extensively tested
- Snort false positive rate in HTTPS-heavy environments requires long-term tuning and is not addressed in this project scope

---

## References

- [OWASP ModSecurity Core Rule Set](https://owasp.org/www-project-modsecurity-core-rule-set/)
- [Snort 3 Documentation](https://www.snort.org/documents)
- [Wazuh Documentation](https://documentation.wazuh.com/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
