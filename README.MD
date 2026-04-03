# HOME SOC Operations Lab [ DISCONTINUED ]
### SOC Analyst L1 - Technical Writeups & Daily Operations

## Overview
A fully operational Security Operations Center deployed on a home LAN,
simulating REAL-WORLD SOC L1 ANALYST WORKFLOW including alert monitoring,
threat triage, IOC investigation, and incident documentation.

## Architecture
| Component | Role | IP |
|---|---|---|
| Kali Linux VM | SOC Manager | 192.168.1.8 |
| Ubuntu VM | Wazuh Agent | 192.168.1.x |   
| Windows 11 | Wazuh Agent | 192.168.1.x |
```sidenote about IP addresses
Kali Linux VM SOC Manager is a static IP address
The rest of the devices are managed by the DHCP (will always change | The agent stays and is stable )
Ubuntu Agents Temporarily Decomissioned
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
| #001 | Suspicious Account Creation | Medium | True Positive |

## Status
Active — updated daily



```
4/3/2026
moving onto a new one
```

```
important notes: Case 001 is a simulated True Positive
```
