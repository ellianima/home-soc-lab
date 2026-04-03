# Home SOC Operations Lab
### SOC Analyst L1 — Technical Writeups & Daily Operations

## Overview
A fully operational Security Operations Center deployed on a home LAN,
simulating real-world SOC L1 analyst workflow including alert monitoring,
threat triage, IOC investigation, and incident documentation.

## Architecture
| Component | Role | IP |
|---|---|---|
| Ubuntu 22.04 (WSL2) | Wazuh Manager + SIEM | 172.23.201.127 |
| Windows 11 | Wazuh Agent + Monitored Endpoint | 172.23.192.1 |

> WSL2 IPs may reset on reboot. Always verify with `hostname -I` before operations.

## Tools
| Tool | Version | Purpose | Status |
|---|---|---|---|
| Wazuh | 4.14.4 | SIEM + EDR + XDR |  Active |
| Logstash | 7.17.29 | Alert pipeline to OpenSearch |  Active |
| Zeek | 8.0.5 | Network traffic analysis |  Planned |
| TheHive | 5.2 | Incident case management |  Planned |

## Daily Workflow
- Monitor Wazuh alerts across active agents
- Investigate IOCs using VirusTotal, AbuseIPDB, urlscan.io
- Document incidents in this repository

## Incidents
| Case | Title | Severity | Verdict |
|---|---|---|---|

## Status
Active — updated regularly
Last updated: April 3, 2026

```
4/3/2026
New repo
```

```
important notes: Hi
```
