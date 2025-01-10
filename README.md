# LynxGuard

> **Vulnerability Assessment and Security Monitoring for Web Application**  
> Graduation Thesis · FPT University (IAP491) · 2024  
> Team: 4 members

---

## Overview

LynxGuard is an integrated SOC platform designed to provide continuous security monitoring and vulnerability assessment for FPT Education's web applications. The system detects and responds to web attacks in real-time by combining multiple open-source security tools into a unified monitoring pipeline.

**Target systems:** FPT Education web applications (fap.fpt.edu.vn, fu-edunext.fpt.edu.vn, career.fpt.edu.vn)

---

## Architecture

```
                    ┌─────────────────────────────────┐
  Internet ──────▶  │  pfSense Firewall + Routing      │
                    └────────────┬────────────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                   ▼
       ┌─────────┐        ┌──────────┐        ┌──────────┐
       │  Snort  │        │  ModSec  │        │  Zabbix  │
       │  (IDS)  │        │  (WAF)   │        │(Monitor) │
       └────┬────┘        └────┬─────┘        └────┬─────┘
            │                  │                    │
            └──────────────────┼────────────────────┘
                               │  Logs & Alerts
                               ▼
                    ┌─────────────────┐
                    │  Telegram Bot   │ ◀── Real-time alerts
                    │    (Python)     │
                    └─────────────────┘
```

**Environment:** Fully virtualized on VMware — all components (firewall, servers, endpoints) run as VMs on a single host.

---

## Stack

| Component | Tool | Role |
|-----------|------|------|
| Firewall & Routing | pfSense | Network perimeter, traffic control |
| IDS/IPS | Snort | Signature-based intrusion detection |
| WAF | ModSecurity | OWASP rule-based request filtering |
| Monitoring | Zabbix | Infrastructure monitoring, alert triggers |
| Vuln Assessment | Acunetix + BurpSuite | Penetration testing, web vuln scanning |
| Alerting | Telegram Bot (Python) | Real-time notifications on detected threats |

---

## My Contribution

Working as part of a 4-person team, I was responsible for:

- **Infrastructure setup:** Configured the entire VMware environment — network topology, VM provisioning, routing between segments
- **Network design:** Designed the virtual network topology with isolated segments for attack simulation and monitoring
- **pfSense configuration:** Set up firewall rules, NAT, traffic routing between zones
- **Telegram bot:** Wrote the Python bot that receives Zabbix alerts and pushes real-time notifications to the team channel

---

## Attack Simulation & Testing

Tested against OWASP Top 10 categories including:

- SQL Injection
- Cross-Site Scripting (XSS)
- Brute Force / Credential Stuffing
- DoS/DDoS patterns

Penetration testing was conducted using Acunetix (automated scanning) and BurpSuite (manual testing) against the simulated FPT Education web application environment.

---

## Results

- Snort successfully detected signature-based attacks (SQLi, port scan, brute force)
- ModSecurity blocked OWASP Top 10 attack patterns at the WAF layer
- Zabbix triggered alerts within seconds of anomaly detection
- Telegram bot delivered real-time notifications to the team

**Note:** This repository documents the pre-Wazuh SIEM integration phase. A subsequent version added Wazuh for centralized log aggregation and correlation across all components.

---

## Thesis Context

- **Course:** IAP491 — Capstone Project, Information Assurance Department
- **University:** FPT University, Hoa Lac Campus
- **Supervisor:** Mr. Nguyen Manh Cuong
- **Period:** September – December 2024
- **Thesis honored** by the faculty council and featured on FPT University's official page
