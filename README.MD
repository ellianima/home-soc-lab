# HOME SOC Operations Lab [ DISCONTINUED ]
### SOC Analyst L1 - Technical Writeups & Daily Operations

## Overview
A fully operational Security Operations Center deployed on a home LAN,
simulating REAL-WORLD SOC L1 ANALYST WORKFLOW including alert monitoring,
threat triage, IOC investigation, and incident documentation.

## Architecture
| Component | Role | IP |
|---|---|---|
| Ubuntu WSL | SOC Manager | 172.23.201.127 (Or localhost) |
| Windows 11 | Wazuh Agent | 172.23.192.1 |
```sidenote about IP addresses
Not sure yet if windows or ubuntu would reset through DHCP
```

## Tools
| Tool | Purpose |
|---|---|
| Wazuh 4.14.4 | SIEM + EDR |
| Zeek 8.0.5 | Network traffic analysis |
| TheHive 5.2 | Incident case management |
| Kali Linux | SOC workstation |

## Daily Workflow
- Monitor Wazuh alerts across active agents
- Capture and analyze network traffic with Zeek
- Investigate IOCs using VirusTotal, urlscan.io, crt.sh
- Document incidents in TheHive and this repository

## Incidents
| Case | Title | Severity | Verdict |
|---|---|---|---|


## Status
Active — updated daily



```
4/3/2026
New repo
```

```
important notes: Hi
```
