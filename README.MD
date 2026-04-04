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

> WSL2 IP is statically assigned via `/etc/wsl.conf` boot command. Persistent across reboots.

## Tools
| Tool | Version | Purpose | Status |
|---|---|---|---|
| Wazuh | 4.14.4 | SIEM + EDR + XDR |  Active |
| Logstash | 7.17.29 | Alert pipeline to OpenSearch |  Active |
| Zeek | 8.0.5 | Network traffic analysis |  Active |
| TheHive | 5.2 | Incident case management |  Planned |
| Wireshark | Latest | Packet analysis |  Active |
| VirusTotal | — | IOC enrichment |  Active (manual) |
| AbuseIPDB | — | IP reputation lookup |  Active (manual) |
| OTX AlienVault | — | Threat intelligence feeds |  Active (manual) |
| CyberChef | — | Log/IOC decoding |  Active (manual) |
| Nmap | — | Recon detection testing |  Planned |

## Daily Workflow
- Monitor Wazuh alerts across active agents
- Investigate IOCs using VirusTotal, AbuseIPDB, urlscan.io
- Document incidents in this repository

## Incidents
| Case | Title | Severity | Verdict |
|---|---|---|---|

## Status
Active — updated regularly
Last updated: April 4, 2026

```
4/4/2026
tralalero tralala
```

```
important notes: Helloooooooooo
```
